# CI Toolchain Compatibility Resolution: Pin Rust Toolchain to 1.95

## 1. Executive Summary

This document records the root cause analysis, implementation, and verification of the CI toolchain compatibility fix across the Caliptra vulnerability benchmark repositories:
- `caliptra-vuln-known`
- `caliptra-vuln-known-obf20`
- `caliptra-vuln-mixed`
- `caliptra-vuln-mixed-obf20`

This issue was identified strictly as a CI and toolchain compatibility issue and **NOT** a vulnerability issue. No seeded vulnerability logic or source files containing benchmark vulnerabilities (V001–V020) were altered.

---

## 2. Issue Analysis & Root Cause

### 2.1 Previous CI Toolchain
- **Previous Toolchain**: Rust 1.96 (`1.96-x86_64-unknown-linux-gnu` / `channel = "1.96"`).

### 2.2 Failing Dependency
- **Crate**: `memoffset` version `0.8.0`
- **Target Workspace Member**: `caliptra-auth-man-types` (`auth-manifest/types`)
- **Downstream Consequence**: Panics during subsequent image generation (`builder/bin/image_gen.rs`) due to firmware compilation failure.

### 2.3 Exact Compiler Error
```text
error: unexpected end of macro invocation
   --> /root/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/memoffset-0.8.0/src/span_of.rs:140:61
    |
140 |         span_of!(@helper $root, $parent, $(#$begin)* #$tt [] $($rest)*)
    |                                                             ^ missing tokens in macro arguments
    |
note: while trying to match meta-variable `$tt:tt`
   --> /root/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/memoffset-0.8.0/src/span_of.rs:139:60
    |
139 |     (@helper $root:ident, $parent:path, $(# $begin:tt)+ [] $tt:tt $($rest:tt)*) => {{
    |                                                            ^^^^^^

error: could not compile `caliptra-auth-man-types` (lib) due to 1 previous error
```

### 2.4 Why the Error is a Toolchain Compatibility Issue
- The upstream pinned Caliptra baseline is documented and standardized against Rust toolchain **1.95**.
- During CI runs on GitHub Actions runners, the `rustup` environment detected `channel = "1.96"` inside `rust-toolchain.toml`, causing `rustup` to fetch and activate Rust 1.96.
- In addition, workflows `.github/workflows/fpga.yml` and `.github/workflows/fpga-subsystem.yml` explicitly invoked `rustup toolchain install 1.96-x86_64-unknown-linux-gnu`.
- Macro expansion rules in `memoffset` 0.8.0 encountered macro invocation failure under the 1.96 compiler toolchain.
- Standardizing and pinning the GitHub Actions toolchain to Rust **1.95** restores reproducible builds matching the baseline toolchain specification without requiring crate upgrades or code changes.

---

## 3. Pinned Toolchain Specification

The toolchain is explicitly pinned to Rust **1.95** across all repositories and workflow configuration files:
- **Pinned Channel**: `1.95`
- **Exact Installed Version**: `rustc 1.95.0 (59807616e 2026-04-14)`
- **Targets**: `riscv32imc-unknown-none-elf`, `x86_64-unknown-linux-gnu`
- **Prohibited Toolchains**: `stable`, `latest`, `1.96`, `nightly`

---

## 4. Applied Changes

The following CI/toolchain configuration files were updated consistently across all four benchmark branches:

### 4.1 `rust-toolchain.toml`
```toml
# Licensed under the Apache-2.0 license

[toolchain]
channel = "1.95"
targets = ["riscv32imc-unknown-none-elf"]
profile = "minimal"
components = ["rustfmt", "clippy"]
```

### 4.2 `.github/workflows/fpga.yml`
```diff
@@ -139,4 +139,4 @@ jobs:
       - name: Install Rust
         run: |
-          rustup toolchain install 1.96-x86_64-unknown-linux-gnu
+          rustup toolchain install 1.95-x86_64-unknown-linux-gnu
           rustup target add aarch64-unknown-linux-gnu
```

### 4.3 `.github/workflows/fpga-subsystem.yml`
```diff
@@ -169,4 +169,4 @@ jobs:
       - name: Install Rust
         run: |
-          rustup toolchain install 1.96-x86_64-unknown-linux-gnu
+          rustup toolchain install 1.95-x86_64-unknown-linux-gnu
           rustup target add aarch64-unknown-linux-gnu
```

---

## 5. Verification & Validation Results

### 5.1 Confirmation of Unmodified Vulnerabilities
- No source code containing benchmark vulnerabilities (V001–V020) was modified.
- No dependency versions in `Cargo.toml` or `Cargo.lock` were modified or upgraded (`memoffset` remains pinned at `0.8.0`).
- Vulnerability exploitability, detection difficulty, and behavioral invariants remain identical to audited baselines.

### 5.2 Vulnerability Validation Results
All four benchmark suites were executed against the pinned test harnesses:

| Benchmark Repository | Target Branch | Validation Result | Status |
|---|---|:---:|:---:|
| `caliptra-vuln-known` | `caliptra-vuln-known` | 20 / 20 PASS | **PASS** |
| `caliptra-vuln-known-obf20` | `caliptra-vuln-known-obf20` | 20 / 20 PASS | **PASS** |
| `caliptra-vuln-mixed` | `caliptra-vuln-mixed` | 20 / 20 PASS | **PASS** |
| `caliptra-vuln-mixed-obf20` | `caliptra-vuln-mixed-obf20` | 20 / 20 PASS | **PASS** |

---

## 6. Commit & Branch Policy Adherence
- Commits created under author: `krithick-rk <krithickkrishnasamy456@gmail.com>`.
- No benchmark branch was merged into `main`.
