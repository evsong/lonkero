## Why

Lonkero is a source-available web security scanner with 133 scanners and 195,000+ payloads. The detection logic is fully readable, but runtime execution is gated by server-side license verification (`lonkero.bountyy.fi`), anti-tamper checks, and feature flags. We want to build a version that runs fully offline for personal security research and learning, without depending on Bountyy's license server.

## What Changes

- Add a Cargo feature flag `no_license` that bypasses all authorization checks at compile time
- When `no_license` is enabled:
  - All `verify_scan_authorized()` calls return `true` (44 scanner files, zero code changes in scanners)
  - All `is_feature_available(_)` calls return `true` (unlocks all 133 scanners including enterprise-only)
  - All `is_killswitch_active()` calls return `false` (disables remote killswitch)
  - All anti-tamper integrity checks pass unconditionally
  - All `is_scan_authorized()` signing checks return `true`
  - OOB detector falls back to interactsh/self-hosted instead of `oob.lonkero.bountyy.fi`
  - Federated ML upload to Bountyy server is disabled
- When `no_license` is NOT enabled: original behavior preserved, upstream merge-compatible
- **BREAKING**: `--features no_license` builds cannot use Bountyy's OOB callback infrastructure

## Capabilities

### New Capabilities
- `license-bypass`: Cargo feature flag `no_license` that provides compile-time bypass implementations for all license/auth/anti-tamper functions. Only modifies files in `src/license/`, `src/signing/`, and `Cargo.toml`.
- `offline-oob`: Replace Bountyy OOB callback domain with configurable self-hosted or interactsh fallback when `no_license` is enabled.

### Modified Capabilities
<!-- No existing openspec specs to modify -->

## Impact

- **Files changed**: `Cargo.toml`, `src/license/mod.rs`, `src/license/anti_tamper.rs`, `src/signing/mod.rs`, `src/oob_detector.rs`, `src/ml/federated.rs` (6 files total)
- **Files NOT changed**: All 133 scanner files, all payload files, all analysis/inference/ML modules, CLI, reporting — zero touch
- **Build**: Two build targets from same codebase (`cargo build --release` vs `cargo build --release --features no_license`)
- **Upstream sync**: Minimal merge conflict surface since scanner code is untouched
- **Dependencies**: No new dependencies added
