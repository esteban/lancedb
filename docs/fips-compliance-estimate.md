# FIPS Compliance Estimate for LanceDB

**Related Issue:** [lancedb/lancedb#1884](https://github.com/lancedb/lancedb/issues/1884) — OpenSSL/FIPS failure on LanceDB 0.14.0+

## Problem Summary

On FIPS-enabled RHEL 9 systems, importing LanceDB (Python, versions > 0.13.0) causes:
```
crypto/fips/fips.c:154: OpenSSL internal error: FATAL FIPS SELFTEST FAILURE
Abort (core dumped)
```

This occurs because LanceDB's Rust dependencies statically link non-FIPS-validated
cryptographic libraries (`ring`, `aws-lc-rs`) that conflict with FIPS-mode OpenSSL
enforcement at the OS level.

## Current Cryptographic Dependency Chain

### TLS/HTTPS Stack

LanceDB uses **rustls** (pure Rust TLS) — NOT OpenSSL — for all TLS operations:

| Component | Version | Crypto Backend | Used By |
|---|---|---|---|
| `reqwest` | 0.12.24 | `rustls` 0.23.31 via `hyper-rustls` 0.27.7 | LanceDB remote client |
| `hyper-rustls` | 0.27.7 | `rustls` 0.23.31 → `ring` 0.17.14 + `aws-lc-rs` 1.13.0 | reqwest |
| `hyper-rustls` | 0.24.2 | `rustls` 0.21.12 → `ring` 0.17.14 | AWS SDK (`aws-smithy-http-client`) |
| `hf-hub` | 0.4.1 | `rustls-tls` feature | HuggingFace integration (optional) |

### Crypto Libraries Present

1. **`ring` 0.17.14** — Primary crypto provider for both rustls versions. Contains
   compiled C code (via `cc` crate) including its own AES, SHA, etc. implementations.
   **Not FIPS-validated.**

2. **`aws-lc-rs` 1.13.0 / `aws-lc-sys` 0.28.0** — AWS LibCrypto bindings. Currently
   pinned in `nodejs/Cargo.toml` as a build workaround. Available as an alternative
   crypto provider for rustls 0.23 but **not currently configured as the default**.
   AWS-LC *can* be built in FIPS mode but is not currently.

3. **`openssl-probe` 0.1.6** — Only probes for system certificate locations. Does NOT
   link against OpenSSL's crypto. Used by `rustls-native-certs`.

4. **Additional crypto crates** (via AWS SDK, object_store): `sha2`, `hmac`, `sha1`,
   `aes`, `pbkdf2`, `rsa`, `p256` — pure Rust implementations for request signing, not TLS.

### Root Cause of FIPS Failure

On FIPS-enabled systems, the OS enforces that all cryptographic operations go through
FIPS-validated modules. The `ring` crate bundles its own C crypto implementations
(BoringSSL-derived) which are loaded into the process. The FIPS enforcement detects these
non-validated crypto routines and triggers the fatal selftest failure at process startup.

## Proposed Solution: aws-lc-rs FIPS Mode

The recommended path to FIPS compliance is to replace `ring` with `aws-lc-rs` built in
FIPS mode. AWS-LC has a [FIPS 140-3 validated version](https://github.com/aws/aws-lc/blob/main/crypto/fipsmodule/FIPS.md).

### Required Changes

#### Phase 1: Consolidate on rustls 0.23 with aws-lc-rs (Estimated: 2-3 weeks)

1. **Remove `ring` dependency in favor of `aws-lc-rs`**

   In `rust/lancedb/Cargo.toml` and workspace `Cargo.toml`, configure reqwest and
   rustls to use `aws-lc-rs` as the sole crypto provider:

   ```toml
   # Workspace Cargo.toml - add feature to select crypto backend
   [features]
   fips = ["rustls/fips", "aws-lc-rs/fips"]
   ```

   Configure `reqwest` with explicit rustls + aws-lc-rs:
   ```toml
   reqwest = { version = "0.12", default-features = false, features = [
       "rustls-tls-manual-roots-no-provider",
       # ... other features
   ] }
   ```

   Then programmatically install the aws-lc-rs crypto provider:
   ```rust
   rustls::crypto::aws_lc_rs::default_provider()
       .install_default()
       .expect("Failed to install crypto provider");
   ```

2. **Eliminate dual rustls versions**

   The `aws-smithy-http-client` brings in `hyper-rustls 0.24.2` → `rustls 0.21.12`
   → `ring`. This needs resolution:
   - Update AWS SDK dependencies to versions using `hyper-rustls 0.27+` / `rustls 0.23+`
   - Or configure `aws-smithy-http-client` with `rustls-0_23-crt` feature if available

3. **Ensure `object_store` uses the same TLS stack**

   `object_store` (workspace dependency) also pulls in `ring` via `reqsign`. Verify it
   can be configured to use `aws-lc-rs` instead.

   **Files to modify:**
   - `/Cargo.toml` (workspace)
   - `/rust/lancedb/Cargo.toml`
   - `/python/Cargo.toml`
   - `/nodejs/Cargo.toml`

#### Phase 2: Add FIPS Feature Flag (Estimated: 1 week)

4. **Add `fips` feature flag to LanceDB**

   ```toml
   # rust/lancedb/Cargo.toml
   [features]
   fips = []  # Enables FIPS-validated crypto via aws-lc-rs FIPS module
   ```

   When the `fips` feature is enabled, `aws-lc-rs` should be built with its `fips`
   feature, which links against the FIPS-validated AWS-LC module instead of the
   standard one.

5. **Expose in Python/Node.js bindings**

   ```toml
   # python/Cargo.toml
   [features]
   fips = ["lancedb/fips"]

   # nodejs/Cargo.toml
   [features]
   fips = ["lancedb/fips"]
   ```

   Python wheels for FIPS would need to be built separately (e.g., `pip install lancedb[fips]`
   or a separate `lancedb-fips` package) since the FIPS build requires CMake and Go
   toolchains for aws-lc-fips-sys.

#### Phase 3: Build & CI Infrastructure (Estimated: 1-2 weeks)

6. **FIPS build requirements**

   Building `aws-lc-rs` with FIPS requires:
   - CMake 3.18+
   - Go 1.18+ (for BoringSSL's FIPS module build)
   - C/C++ compiler
   - NASM (on Windows)

   CI must be updated to install these tools and run tests with the `fips` feature.

7. **FIPS CI testing**

   - Add a CI job that builds with `--features fips`
   - Test on a FIPS-enabled container (e.g., RHEL 9 UBI with `fips-mode-setup --enable`)
   - Verify no `ring` symbols are linked in the final binary

8. **Python wheel builds**

   - Add a maturin build variant for FIPS-enabled wheels
   - Consider publishing as separate wheels (e.g., `lancedb-fips`) due to different
     build toolchain requirements

#### Phase 4: Testing & Validation (Estimated: 1 week)

9. **Validation steps**
   - Verify `ring` is completely removed from the dependency tree when FIPS is enabled
   - Run on FIPS-enabled RHEL 9 container
   - Verify TLS connections work (remote LanceDB Cloud, S3, GCS, Azure)
   - Run full test suite with `--features fips`
   - Check binary for non-FIPS crypto symbols using `nm` / `objdump`

### Alternative Approaches Considered

| Approach | Pros | Cons |
|---|---|---|
| **aws-lc-rs FIPS** (recommended) | Drop-in replacement for ring; FIPS 140-3 validated; maintained by AWS | Requires CMake + Go for FIPS build |
| **System OpenSSL via openssl crate** | Uses OS-provided FIPS module | Major refactor; different API; loses cross-platform consistency |
| **BoringSSL FIPS via boring crate** | Google's FIPS module | Less ecosystem support than aws-lc-rs; complex build |
| **Dynamically link system crypto** | Simplest FIPS path | Breaks hermetic builds; platform-dependent behavior |

## Effort Summary

| Phase | Scope | Estimated Effort |
|---|---|---|
| Phase 1: Consolidate crypto backend | Dependency management, remove ring | 2-3 weeks |
| Phase 2: FIPS feature flag | Feature flags across Rust/Python/Node.js | 1 week |
| Phase 3: Build infrastructure | CI, wheel builds, toolchain setup | 1-2 weeks |
| Phase 4: Testing & validation | FIPS container testing, integration tests | 1 week |
| **Total** | | **5-7 weeks** |

## Key Risks

1. **Transitive dependency conflicts**: Some dependencies may hard-depend on `ring`
   with no `aws-lc-rs` alternative. Audit with `cargo tree -i ring` to identify all
   consumers.

2. **AWS SDK compatibility**: The AWS SDK's TLS stack migration is in progress.
   Depending on the version, `aws-smithy-http-client` may or may not support
   `rustls 0.23` / `aws-lc-rs`. This is the most likely blocker.

3. **Build complexity**: FIPS builds require Go and CMake, which complicates CI and
   user builds. Pre-built wheels mitigate this for Python users.

4. **Performance**: `aws-lc-rs` performance is comparable to `ring` in most
   benchmarks, but should be validated for LanceDB workloads.

5. **Platform support**: `aws-lc-rs` FIPS is primarily validated on Linux x86_64.
   macOS/Windows/ARM support exists but may not be FIPS-certified.

## Immediate Workaround

For users who need FIPS compliance today, the workaround is to pin LanceDB to
version 0.13.0, which predates the introduction of the conflicting crypto libraries.
