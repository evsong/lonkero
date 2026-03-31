## ADDED Requirements

### Requirement: OOB default unchanged
The OOB detector already defaults to `oast.pro` (Interactsh). When compiled with `--features no_license`, the `BountyyCallback` service type SHALL be disabled to prevent any fallback to `oob.lonkero.bountyy.fi`.

#### Scenario: Default OOB service is Interactsh
- **WHEN** `oob_detector` initializes without user config
- **THEN** uses `oast.pro` as callback domain (existing behavior, no change needed)

#### Scenario: BountyyCallback variant disabled
- **WHEN** compiled with `no_license` and code attempts to select `BountyyCallback` service
- **THEN** falls back to Interactsh (`oast.pro`) instead

#### Scenario: User override preserved
- **WHEN** user specifies a custom OOB domain via `LONKERO_OOB_DOMAIN` env var
- **THEN** the custom domain is used regardless of feature flag

### Requirement: Known limitation documented
OOB callback VERIFICATION via Interactsh is not fully implemented in the existing codebase (`check_interactsh_callback()` returns `false`). This is a pre-existing limitation, not a regression from `no_license`.

#### Scenario: OOB payloads still generated
- **WHEN** blind SSRF/XXE scanner generates OOB payloads
- **THEN** payloads use the `oast.pro` domain correctly

#### Scenario: Callback verification returns false
- **WHEN** system checks for OOB callback hits
- **THEN** returns `false` (pre-existing behavior, not a regression)
