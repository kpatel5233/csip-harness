# Cross-agent request log

Requests that need changes in a path owned by another agent (CLAUDE.md §3, §10).
The lead records them here and routes them; the owner closes them.

Status: `open` = recorded, not yet routed · `routed` = sent to owner · `done`.

| ID | For | Requested by | Status | Request |
|---|---|---|---|---|
| REQ-001 | platform-engineer | Keyur (2026-11-18) | open | **Dev-mode timeout override in `POST /runs`.** Allow a run started in development mode to override the definition's `timeouts` (e.g. `wait_for_client_s`, `run_s`, `default_step_s`, step `timeout_s`, `settle_s`, `attestation_s`) so reference-client e2e runs finish fast. Must be rejected outside dev mode; the report must record that overrides were used. |
| REQ-002 | harness-engineer | spec-analyst (format 1.2) | open | Definition loader: enforce `format_version` 1.2, rules R1–R14 and `x-v1-subset`; every rejection names file, feature and JSON pointer. |
| REQ-003 | harness-engineer | spec-analyst (format 1.2) | open | Generic resource-tree server (paging `s`/`l`/`a`, list `all`/`results`, 404/405), always serve `application/sep+xml` regardless of Accept, strict XSD on `urn:ieee:std:2030.5:ns`, integer SFDI comparison. |
| REQ-004 | harness-engineer | spec-analyst (format 1.2) | open | Recorder capturing TLS metadata and client certificate chain; expression evaluator; the v1-subset check types. |
| REQ-005 | platform-engineer | spec-analyst (format 1.2) | open | `POST /runs` validates `dut` against `dut_inputs` (types, check digits, `derive_check`) → 422; PIN required, no default; attestation endpoint; wait up to `attestation_s` in EVALUATING; run-state precedence ABORTED > ERROR > FAIL > TIMEOUT > PASS. |
| REQ-006 | refclient-engineer | spec-analyst (BASIC-001 draft) | open | Fault flags for BASIC-001 negative cases: `skip_registration_get`, `stop_after_dcap`, `skip_fsa_traversal`, `use_other_enddevice`. Route only after Gate 1 approval of BASIC-001. |
| REQ-007 | reviewer | spec-analyst (format 1.2) | open | Positive e2e must answer the BASIC-001 C6 operator attestation via the API, otherwise the run ends TIMEOUT. |

Implementation requests (REQ-002 … REQ-007) are routed only after BASIC-001 passes Gate 1 (CLAUDE.md §4).
