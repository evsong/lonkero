## ADDED Requirements

### Requirement: Feature flag existence
The project SHALL define a Cargo feature `no_license` in `Cargo.toml` under `[features]` with no dependencies.

#### Scenario: Feature flag declared
- **WHEN** inspecting `Cargo.toml`
- **THEN** a `[features]` section exists containing `no_license = []`

### Requirement: License startup bypass
When compiled with `--features no_license`, `verify_license_for_scan()` SHALL return `Ok(LicenseStatus)` with Enterprise-level access, without contacting `lonkero.bountyy.fi`.

#### Scenario: Scan startup succeeds offline
- **WHEN** `cli/main.rs` calls `verify_license_for_scan()`
- **THEN** returns `Ok` with `license_type: "Enterprise"`, `max_targets: u32::MAX`, `killswitch_active: false`

#### Scenario: Original behavior preserved
- **WHEN** compiled WITHOUT `no_license` feature
- **THEN** `verify_license_for_scan()` validates against the Bountyy license server

### Requirement: Authorization bypass
When compiled with `--features no_license`, `verify_scan_authorized()` SHALL always return `true`.

#### Scenario: Scanner authorization passes
- **WHEN** any scanner calls `crate::license::verify_scan_authorized()`
- **THEN** the function returns `true` without network calls

### Requirement: Feature availability bypass
When compiled with `--features no_license`, both `is_feature_available()` AND `has_feature()` SHALL return `true` for any input.

#### Scenario: is_feature_available unlocks enterprise scanners
- **WHEN** a scanner calls `crate::license::is_feature_available("enterprise_cmd_injection")`
- **THEN** the function returns `true`

#### Scenario: has_feature unlocks CMS/framework scanners
- **WHEN** a framework scanner calls `crate::license::has_feature("laravel_security")`
- **THEN** the function returns `true` (unlocks 11 framework-specific scanners + browser_assist)

### Requirement: Killswitch disabled
When compiled with `--features no_license`, `is_killswitch_active()` SHALL return `false`.

#### Scenario: Killswitch check
- **WHEN** the system checks killswitch status
- **THEN** returns `false`

### Requirement: Binary integrity bypass
When compiled with `--features no_license`, `verify_binary_integrity()`, `verify_enforcement_integrity()`, and `verify_rt_state()` SHALL return `true`.

#### Scenario: All integrity checks pass
- **WHEN** any integrity verification function is called
- **THEN** returns `true`

### Requirement: License utility bypasses
When compiled with `--features no_license`, all remaining license utility functions SHALL return safe values.

#### Scenario: allows_commercial_use
- **WHEN** `allows_commercial_use()` is called
- **THEN** returns `true`

#### Scenario: get_max_targets
- **WHEN** `get_max_targets()` is called
- **THEN** returns `usize::MAX`

#### Scenario: get_license_signature
- **WHEN** `get_license_signature()` is called
- **THEN** returns a dummy string (e.g. `"no_license"`)

#### Scenario: is_validation_stale
- **WHEN** `is_validation_stale()` is called
- **THEN** returns `false`

#### Scenario: hours_since_validation
- **WHEN** `hours_since_validation()` is called
- **THEN** returns `0`

#### Scenario: get_integrity_marker
- **WHEN** `get_integrity_marker()` is called
- **THEN** returns a valid u64 value

#### Scenario: scan counters
- **WHEN** `increment_scan_counter()` or `get_scan_counter()` is called
- **THEN** returns `0`

#### Scenario: print_license_info
- **WHEN** `print_license_info()` is called
- **THEN** prints a minimal no-op message or nothing

### Requirement: Anti-tamper full module bypass
When compiled with `--features no_license`, the ENTIRE `anti_tamper.rs` module SHALL be replaced with safe stubs using a single `#[cfg]` module-level gate. This covers all 22+ pub functions including honeypot functions.

#### Scenario: Core checks pass
- **WHEN** `i_p()`, `v_s()`, `v_c()`, `v_f()`, `q_c()`, `r_c()` are called
- **THEN** all return `true`

#### Scenario: Tamper detection disabled
- **WHEN** `was_tampered()`, `d_d()` are called
- **THEN** return `false`

#### Scenario: Value authentication passthrough
- **WHEN** `v_a(n)` or `c_a(n, e)` are called
- **THEN** `v_a` returns `n`, `c_a` returns `true`

#### Scenario: Honeypot functions neutralized
- **WHEN** `b_l()`, `u_l()`, `g_f()`, `p_l()` are called
- **THEN** return `false` (never trigger tamper response)

#### Scenario: Honeypot no-ops
- **WHEN** `e_a()`, `d_v()`, `s_t()`, `t_r()` are called
- **THEN** execute as no-ops (no state poisoning)

#### Scenario: Pub use aliases work
- **WHEN** code calls `initialize_protection`, `was_tampered`, `full_integrity_check` etc. via pub use aliases
- **THEN** they resolve to the bypass stubs

### Requirement: Signing full bypass
When compiled with `--features no_license`, ALL signing functions SHALL work without Bountyy server access.

#### Scenario: is_scan_authorized passes
- **WHEN** `signing::is_scan_authorized()` is called
- **THEN** returns `true`

#### Scenario: authorize_scan succeeds
- **WHEN** `signing::authorize_scan(...)` is called
- **THEN** returns `Ok(ScanToken)` with a dummy token, no network call

#### Scenario: sign_results succeeds
- **WHEN** `signing::sign_results(...)` is called
- **THEN** returns `Ok(ReportSignature)` with a dummy signature

#### Scenario: get_scan_token returns dummy
- **WHEN** `signing::get_scan_token()` is called
- **THEN** returns `Some(dummy_token)`

### Requirement: Federated ML download disabled
When compiled with `--features no_license`, the federated ML client SHALL NOT contact `lonkero.bountyy.fi/api/federated/v1`. Note: this is a download client (model distribution), not an uploader.

#### Scenario: No model download attempted
- **WHEN** `FederatedClient::fetch_global_model()` or `fetch_categories()` is called
- **THEN** returns early with cached/default model, no network call

### Requirement: Scanner code untouched
No files under `src/scanners/` SHALL be modified by this change.

#### Scenario: Scanner file integrity
- **WHEN** comparing `src/scanners/` before and after the change
- **THEN** all files have identical content (zero diff)
