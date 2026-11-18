# ADR 0001 — Test definition format 1.0

Status: proposed (awaiting Gate 1 together with the first test definition)
Date: 2026-11-18
Owner: spec-analyst

## Context
csip-harness needs one machine-readable description per SunSpec CSIP v1.3 test
that the harness executes, the evaluator judges, and the reference client is
built against, independently. Only DER-client / aggregator-client behaviour is in
scope; the harness is always the 2030.5 server.

## Decision
- One YAML file per SunSpec test ID; format specified in `definitions/schema.yaml`,
  which is a JSON Schema (draft 2020-12) in YAML plus loader-enforced semantic rules R1–R12.
- `format_version` "MAJOR.MINOR"; loader rejects foreign MAJOR.
- SunSpec setup [S] becomes `preconditions` (a declarative resource tree with concrete
  values); in-procedure [S] becomes `harness` steps; [C]/[A] become `dut` observation
  steps; [T] becomes `recorder` steps; pure server-under-test content goes to `excluded`.
- Steps orchestrate; only `criteria` produce verdicts. Criteria are evaluated after the
  run on recorded data using a closed vocabulary of 12 check types + 4 combinators.
- Behaviour not visible on the wire uses `evidence: operator_attestation`, collected
  through the Control API; missing answers yield INCONCLUSIVE → run TIMEOUT, never PASS.
- Harness self-checks (`evidence: harness_self_check`) map to ERROR, never to a DUT FAIL.
- Run state precedence: ABORTED > ERROR > FAIL > TIMEOUT > PASS; WARN (SHOULD-level) does
  not block PASS.
- Requirement IDs are namespaced (`2030.5:`, `CSIP:`, `IEEE2030.5-2018:`, `RFC<n>:`).
- IDs of tests, criteria (`.C<n>`), negative cases (`.N<n>`) and steps are never renumbered,
  to keep v1.3 → v1.4 diffs meaningful.

## Consequences
- harness-engineer implements: loader + semantic rules, resource-tree server, recorder,
  expression evaluator, check vocabulary.
- platform-engineer implements: `dut_inputs` validation at `POST /runs`, attestation
  endpoint, EVALUATING wait for attestations, verdict precedence.
- refclient-engineer implements each `negative_cases[].fault.refclient_flag`.
- Adding a check type is a MINOR format change and needs a new ADR.
