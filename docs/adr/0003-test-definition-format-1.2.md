# ADR 0003 — Test definition format 1.2 (resolved open questions)

Status: accepted
Date: 2026-11-18
Owner: spec-analyst
Amends: [ADR 0001 — format 1.0](0001-test-definition-format.md),
[ADR 0002 — format 1.1](0002-test-definition-format-1.1.md)

**Format approved by Keyur on 2026-11-18 after these changes.**

## Context
ADR 0001/0002 left open questions. Keyur decided them; this ADR records each
decision and the resulting changes to `definitions/schema.yaml` and
`definitions/README.md`. No test definition is written as part of this change.

## Decisions (resolved)
1. **Aggregator out of scope for v1.** BASIC-001 will declare
   `applies_to: [der_client]` only. `x-v1-subset` now lists `dut_types:
   [der_client]`; `for_each`, list-type DUT inputs, `dut.<name>[i]` indexing and
   `item.<field>` moved to "not in v1". `body_field` stays. The README example
   uses no `for_each`.
2. **Missing operator answer = INCONCLUSIVE → run TIMEOUT.** No new lifecycle
   state. Stated explicitly in the schema VERDICTS text and the README.
3. **"Client validated SFDI/LFDI/PIN" (BASIC-001).** v1 judges this with
   automatic traffic criteria on the DCAP → EndDeviceList → own EndDevice →
   Registration → FSA traversal, plus exactly ONE mandatory
   `operator_attestation` that the DUT confirmed SFDI/LFDI/PIN match.
   No format change needed (existing check types suffice).
   **Deferred:** harness-side mismatch faults (e.g. serving a wrong PIN/SFDI and
   requiring the DUT to stop) will come later as a separate negative variant;
   they are not added to the format now.
4. **SFDI = 1–12 decimal digits** (36-bit value + check digit), compared as an
   integer; no fixed length, no zero-padding requirement. Updated the `sfdi`
   input type, `sfdi_of` and the global comparison conventions.
5. **Requirement IDs copied exactly as printed** in the SunSpec matrices /
   "Requirements Tested" tables; the namespace is the matrix they come from,
   even when the ID is missing from the matrix body (e.g. `2030.5:BASE.074`,
   with a note in the definition). The pattern `^(...):.+$` already accepts any
   printed ID; only the description changed.
6. **XML namespace `urn:ieee:std:2030.5:ns` only** (IEEE 2030.5-2018). Legacy
   namespaces are not acceptable; bodies in another namespace fail `xsd_valid`
   and `body_field`. Stated in the schema conventions, `x-v1-subset`, and README.
7. **Wrong Accept header: harness responds normally.** In BASIC-001 the
   Accept-header criterion is `severity: recommended` → WARN. Strict HTTP checks
   belong to CORE tests. The format already maps recommended → WARN (non-blocking);
   the `severity` description now says so explicitly.
8. **PIN is a required DUT input with no default.** New semantic rule R14:
   identity inputs (lfdi, sfdi, pin, pem_certificate) are `required: true` with
   no `default`; 111115 may appear only as `example`.

Also: `format_version` const, `$id`, README and README example bumped to **1.2**.

## Consequences
- harness-engineer: loader enforces 1.2, R1–R14 and the narrowed `x-v1-subset`
  (reject `aggregator_client`, `for_each`, list inputs); SFDI integer comparison;
  namespace-strict XSD validation; serve `application/sep+xml` regardless of Accept.
- platform-engineer: `POST /runs` requires PIN (no default); SFDI accepted as
  1–12 digits with check-digit validation.
- spec-analyst: BASIC-001 may now be written against format 1.2 (not part of
  this change).
