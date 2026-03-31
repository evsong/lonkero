## Context

Lonkero v3.7.3 is a Rust web security scanner with 253K lines of code. Its license enforcement has three layers:

1. **Runtime auth** (`license/mod.rs`): Functions like `verify_scan_authorized()` and `is_feature_available()` called at the top of 38 scanners. They connect to `https://lonkero.bountyy.fi/api/v1` for token validation.
2. **Anti-tamper** (`license/anti_tamper.rs`): Obfuscated integrity checks using function pointer hashing, SHA-256/512 checksums, and magic constants. Detects binary patching.
3. **Signing** (`signing/mod.rs`): `is_scan_authorized()` checks called in 6 scanners, validates scan tokens from the server.

Additionally, `oob_detector.rs` uses `oob.lonkero.bountyy.fi` for OOB callbacks, and `ml/federated.rs` uploads training data to Bountyy's server.

**Constraint**: Scanner files (133 `.rs` files, 157K lines) must remain untouched to minimize merge conflicts with upstream.

## Goals / Non-Goals

**Goals:**
- Compile a fully functional scanner with `cargo build --release --features no_license`
- All 133 scanners run without network dependency on Bountyy servers
- Original build (`cargo build --release`) remains identical to upstream
- Minimal diff — only files in `license/`, `signing/`, `Cargo.toml`, and 2 infra files changed

**Non-Goals:**
- Cracking or reverse-engineering the license protocol
- Modifying any scanner detection logic
- Adding new features beyond the bypass
- Supporting runtime toggle (compile-time only is sufficient)

## Decisions

### D1: Compile-time feature flag over runtime flag
**Choice**: `#[cfg(feature = "no_license")]` conditional compilation
**Over**: Runtime `--no-license` CLI flag or environment variable
**Rationale**: Compile-time means bypass code is completely absent from the original binary. No performance overhead, no accidental activation, cleaner separation. The anti-tamper system checks function pointers at runtime — a compile-time approach avoids triggering those checks entirely because the checking code itself is compiled out.

### D2: Bypass at function definition, not call site
**Choice**: Provide alternate function implementations in `license/mod.rs` gated by `#[cfg]`
**Over**: Adding `#[cfg]` guards at each of the 104 call sites in 44 files
**Rationale**: Changing 3 files vs 44 files. The public API (`verify_scan_authorized()`, `is_feature_available()`, etc.) stays identical — callers don't know or care which implementation they're calling. This is the key insight that makes the A+C approach work.

### D3: Stub the entire anti_tamper module
**Choice**: When `no_license`, replace the entire `anti_tamper.rs` public API with trivial stubs
**Over**: Trying to make the obfuscated code work without a license
**Rationale**: `anti_tamper.rs` is intentionally obfuscated (single-letter function names, magic constants, function pointer hashing). Understanding it is unnecessary — we only need its public functions to return safe values.

### D4: OOB fallback to interactsh
**Choice**: When `no_license`, `oob_detector.rs` defaults to `interact.sh` instead of `oob.lonkero.bountyy.fi`
**Over**: Disabling OOB detection entirely
**Rationale**: Blind SSRF/XXE detection requires OOB callbacks. Interactsh is open-source and widely used. A single `#[cfg]` on the default domain constant preserves full blind detection capability.

## Risks / Trade-offs

- **[Risk] Upstream update adds new auth checks** → Mitigation: Since we only bypass at the function definition level, new call sites automatically get the bypass. Only risk is if upstream adds NEW auth functions — monitor `license/mod.rs` exports on merge.
- **[Risk] Anti-tamper has hidden checks outside license/]** → Mitigation: Grepped the entire codebase — all 104 auth-related calls trace back to functions in `license/` and `signing/`. No hidden checks found in scanner code.
- **[Risk] OOB interactsh service unavailable** → Mitigation: OOB domain is configurable via config file, user can self-host.
- **[Trade-off] No federated ML** → Acceptable: ML auto-learning still works locally, just doesn't upload to Bountyy's server.
