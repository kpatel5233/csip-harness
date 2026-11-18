# ADR 0002 — Test definition format 1.1

Status: proposed (Gate 1 together with the first test definition)
Date: 2026-11-18
Owner: spec-analyst
Amends: [ADR 0001 — Test definition format 1.0](0001-test-definition-format.md)

## Context
Revision request from Keyur (Gate-1 review of format 1.0):
`definitions/schema.yaml` did not parse as YAML; `sunspec_tag` could not express
combined tags; the v1 implementation scope was unbounded; user-input validation
had leaked into the criterion vocabulary.

## Decision
1. **YAML hygiene.** Every one-line `description` in `schema.yaml` is double-quoted;
   longer ones use folded block scalars. Flow mappings `{...}` no longer carry
   descriptions. Example texts that themselves contained YAML syntax
   (e.g. a query regex mapping) were reworded in prose.
2. **`sunspec_tag`** is now a regex pattern instead of an enum. It accepts
   `none`, any single tag (`[C]`, `[A]`, `[S]`, `[TS]`, `[T]`, `[TC]`, `[U]`) and
   combined tags such as `[C, A]` or `[C, S]`.
3. **`x-v1-subset`** added to `schema.yaml`: the only check types, step roles,
   selector fields, expression constructs, enum values and optional sections the
   v1 loader accepts — exactly what BASIC-001 needs. Check types in v1:
   `request_count`, `request_order`, `request_header`, `xsd_valid`, `body_field`,
   `traversal`, `tls_property`, `operator_attestation`, `all_of`, `any_of`,
   `for_each`. Not in v1: `response_status`, `timing_window`, `poll_interval`,
   `not`, `harness` steps/actions, `preconditions.clock`, `direction:
   harness_to_dut`, duration arithmetic. The loader MUST reject anything outside
   the subset with an error naming the definition file, the feature and its JSON
   pointer (new semantic rule R13). Extending the subset needs an ADR.
4. **`dut_input` check type removed.** Its asserts (`pin_check_digit_valid`,
   `sfdi_check_digit_valid`, `sfdi_matches_lfdi`, `lfdi_matches_certificate`)
   only validated user input; that is done at `POST /runs` by `dut_inputs[].type`
   and `derive_check` (HTTP 422). The remaining two asserts
   (`certificate_is_p256_ecdsa`, `chain_leads_to_lab_serca`) also judged only the
   supplied input, not DUT behaviour, so the whole type was removed; the DUT's
   actually-presented certificate is judged via `tls_property`
   (`client_cert_lfdi`, `client_chain_length`). `evidence: dut_input` was removed
   accordingly.
5. **`format_version`** is now `const: "1.1"` (was a free MAJOR.MINOR pattern);
   `$id` updated to `urn:csip-harness:definitions:schema:1.1`. The README example
   uses 1.1 and only v1-subset features (its `clock` section was removed).

Everything else in ADR 0001 stands (verdict precedence, roles, ID stability,
requirement-ID namespaces).

## Consequences
- harness-engineer: loader enforces `const` version, R1–R13 and `x-v1-subset`
  with file/feature/pointer in the error; no `dut_input` check to implement.
- platform-engineer: `POST /runs` owns all input validation (types, check digits,
  `derive_check`).
- A curve/algorithm check on the DUT certificate, if needed later, must be added
  as a new `tls_property` property via a further ADR.
