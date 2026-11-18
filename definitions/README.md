# Test definitions

One YAML file per SunSpec CSIP conformance test, named `<SunSpec ID>.yaml`
(baseline: *SunSpec CSIP Conformance Test Procedures v1.3*). The format is
specified field-by-field in [`schema.yaml`](schema.yaml), which is also a JSON
Schema (draft 2020-12) the loader validates against. Format version: **1.2**
(history: `docs/adr/0001`, `0002`, `0003`; format approved by Keyur on
2026-11-18).

The approved YAML is the single source of truth (CLAUDE.md §4): the harness
executes it, the evaluator judges against it, and the reference client is built
from it.

## Structure at a glance

| Section | What it holds |
|---|---|
| `format_version`, `id`, `title`, `status`, `approval` | Format version, stable SunSpec ID, Gate-1 state |
| `sunspec_ref` | Document, version, section, pages, errata rows — cite, never quote |
| `applies_to` | `der_client` and/or `aggregator_client` (server testing is out of scope; v1 supports `der_client` only) |
| `dut_inputs` | What the user sends in `dut` of `POST /runs` (certificate, LFDI, PIN, managed devices…) |
| `preconditions` | TLS/server config and the 2030.5 resource tree the harness provisions, with concrete values |
| `steps` | Ordered procedure; each step has a role, a SunSpec citation, and (for DUT steps) the expected requests and harness answers |
| `criteria` | Measurable checks on recorded traffic, each with `<ID>.C<n>`, requirement IDs, linked steps, failure message |
| `timeouts` | Client-wait, run, per-step, settle and attestation limits |
| `negative_cases` | Broken-client behaviours (`<ID>.N<n>`) and which criteria they must fail |
| `excluded`, `notes` | Dropped server-only content; errata, interpretations, open questions |

## Mapping SunSpec roles to step roles

| SunSpec tag | `role` | Meaning in the harness |
|---|---|---|
| `[C]`, `[A]` | `dut` | Observation window: the DUT is expected to make the listed requests |
| `[S]`, `[TS]` | `harness` (or `preconditions`) | Setup becomes provisioning; in-procedure server actions become harness actions |
| `[T]` | `recorder` | Requirement on the recorder (always records everything) |
| — | `operator` | A human must observe something not visible on the wire |
| `[TC]`, pure `[S]` criteria | `excluded` | Server-under-test content, listed for audit only |

## Check vocabulary

Every criterion has exactly one `check` (combinators allowed). Checks only read
recorded exchanges, TLS metadata, the provisioned resource graph, or operator
attestations. Validation of user-supplied inputs (check digits, LFDI vs.
certificate, SFDI vs. LFDI) is **not** a criterion: the Control API performs it
at `POST /runs` via `dut_inputs[].type` and `derive_check` and rejects bad input
with HTTP 422.

| `type` | Passes when | In v1 subset |
|---|---|---|
| `request_count` | number of matching exchanges is within `[min, max]` (`max: 0` = forbidden) | yes |
| `request_order` | first occurrences of the selectors appear in the given order | yes |
| `request_header` | a request header satisfies `op`/`value` | yes |
| `response_status` | status codes are in `expected` (DUT's own responses when `direction: harness_to_dut`) | no |
| `xsd_valid` | request/response body validates against sep.xsd (optionally with a given root) | yes |
| `body_field` | an XPath (`sep:` prefix) on the body satisfies `op`/`value` | yes |
| `timing_window` | first matching exchange happens within `[min_s, max_s]` of an anchor | no |
| `poll_interval` | gaps between consecutive matching exchanges are within bounds | no |
| `traversal` | the DUT GET every href reachable from a root resource | yes |
| `tls_property` | TLS version / cipher / curve / client LFDI / chain length match | yes |
| `operator_attestation` | the operator answered an attestation with the expected value | yes |
| `all_of`, `any_of` | combinators | yes |
| `not` | inverts a sub-check | no |
| `for_each` | iterates a list input, e.g. aggregator managed devices | no |

A selector that matches nothing makes the check FAIL unless `allow_empty: true`.

## v1 subset

`schema.yaml` describes the full format, but the v1 loader accepts only the
features listed under `x-v1-subset` — exactly what BASIC-001 needs. Anything
else (e.g. `aggregator_client`, `for_each`, list inputs, `harness` steps,
`preconditions.clock`, `timing_window`, `direction: harness_to_dut`, duration
arithmetic in expressions) is rejected with an error naming the definition
file, the feature, and its location. Aggregator clients are out of scope for v1.

## Protocol conventions fixed by the format

- XML namespace is `urn:ieee:std:2030.5:ns` (IEEE 2030.5-2018) only; bodies in
  any legacy namespace fail XSD validation.
- The harness answers normally whatever the request's Accept header says.
  Strict HTTP checks belong to CORE tests; elsewhere (e.g. BASIC tests) the
  Accept-header criterion uses `severity: recommended`, so a violation is WARN.
- SFDI is 1–12 decimal digits (36-bit value + check digit), compared as an
  integer; no fixed length or zero padding.
- Identity inputs (LFDI, SFDI, PIN, certificate) are required and have no
  default. PIN 111115 (the SunSpec default) appears only as an `example`.

## Verdicts

Per criterion: `PASS`, `FAIL`, `WARN` (recommended-level failure),
`NOT_APPLICABLE`, `INCONCLUSIVE` (evidence missing), `ERROR`.
Run terminal state, first rule that applies: `ABORTED` (user abort) → `ERROR`
(any criterion ERROR, incl. failed `harness_self_check`) → `FAIL` (any FAIL) →
`TIMEOUT` (any INCONCLUSIVE) → `PASS`.

**Operator-observed criteria.** Some SunSpec criteria describe DUT-internal
behaviour (e.g. "client validated the PIN", "client applied the default
control"). These use `evidence: operator_attestation`: an `operator` step posts
a prompt through the Control API, the operator answers, and the answer is
recorded like any other evidence. A missing answer makes the criterion
INCONCLUSIVE (run → TIMEOUT), never PASS; there is no separate lifecycle
state for it. Prefer a traffic-observable proxy where one exists and document
the choice in `rationale`.

## Authoring rules

1. Paraphrase; cite section, step and pass/fail bullet numbers. No verbatim SunSpec/IEEE text.
2. Every criterion must be checkable from traffic, TLS data, or an attestation (input validation happens at `POST /runs`).
3. Every interpretation or erratum gets a `notes` entry; unresolved ones set `open_question: true` and block approval.
4. IDs (`<ID>.C<n>`, `<ID>.N<n>`, `S<n>`) are never renumbered or reused.
5. Requirement IDs are namespaced by the matrix they come from and copied exactly as printed in the SunSpec matrices / "Requirements Tested" tables — even when absent from the matrix body (e.g. `2030.5:BASE.074`; add a note). Examples: `2030.5:SEC.004`, `CSIP:P14`, `IEEE2030.5-2018:6.3.3`, `RFC5246:7.4`.
6. `definitions/requirements_matrix.csv` (columns: `test_id, criterion_id, requirement_id, sunspec_version`) is regenerated from the criteria; `EXAMPLE-*` files are never exported.

## Example (fictional)

`EXAMPLE-001` is invented to illustrate the format. It is **not** a SunSpec
test and is never offered for conformance runs. It uses only `x-v1-subset`
features.

```yaml
format_version: "1.2"
id: EXAMPLE-001
title: "FICTIONAL - client reads DeviceCapability, then Time"
status: draft
sunspec_ref:
  document: none
  version: "n/a"
  section: "n/a"
  pages: "n/a"
applies_to: [der_client]
purpose: >
  Invented fixture. The DUT connects, reads DeviceCapability, follows its
  TimeLink and reads Time, declaring the 2030.5 media type.

dut_inputs:
  - name: client_certificate
    type: pem_certificate
    required: true
    description: DUT device certificate (public part only); authorizes the TLS client.
    example: "-----BEGIN CERTIFICATE-----\nMIIB...\n-----END CERTIFICATE-----"
  - name: lfdi
    type: lfdi
    required: true
    description: LFDI the DUT reports for itself.
    derive_check: lfdi_matches_certificate
    derive_from: client_certificate
    example: "3E4F45AB31EDFE5B67E343E5E4562E31984E23E5"

preconditions:
  server:
    dcap_path: /dcap
    tls:
      version: "1.2"
      cipher_suites: [TLS_ECDHE_ECDSA_WITH_AES_128_CCM_8]
      curve: secp256r1
      client_auth: required
    authorized_clients: ["${dut.lfdi}"]
  resources:
    - key: dcap
      type: DeviceCapability
      href: /dcap
      poll_rate_s: 900
      links: {TimeLink: tm}
    - key: tm
      type: Time
      href: /tm
      fields: {tzOffset: 0, dstOffset: 0, dstStartTime: 0, dstEndTime: 0}
      description: "currentTime and quality are filled by the harness defaults (UTC, quality 7) at response time."

steps:
  - id: S1
    role: recorder
    sunspec_tag: "[T]"
    extension: true
    description: Start recording all exchanges and TLS metadata.
  - id: S2
    role: dut
    sunspec_tag: "[C]"
    extension: true
    description: DUT reads DeviceCapability.
    expect:
      - {method: GET, resource: dcap, response: {status: 200, body: dcap}}
    timeout_s: 120
  - id: S3
    role: dut
    sunspec_tag: "[C]"
    extension: true
    description: DUT follows TimeLink and reads Time.
    expect:
      - {method: GET, resource: tm, response: {status: 200, body: tm}}
    timeout_s: 120
  - id: S4
    role: operator
    sunspec_tag: none
    extension: true
    description: Operator checks the DUT clock now matches the harness clock.
    prompt:
      text: "Does the DUT's local UI/log show a time within 5 s of the harness clock?"
      attestation: dut_clock_synced
      choices: ["yes", "no"]

criteria:
  - id: EXAMPLE-001.C1
    title: DeviceCapability is read before Time
    sunspec_criterion: extension
    verifies_steps: [S2, S3]
    requirements: ["2030.5:GEN.004"]
    evidence: traffic
    severity: mandatory
    check:
      type: request_order
      sequence:
        - {method: GET, resource: dcap, status: 200}
        - {method: GET, resource: tm, status: 200}
    failure_message: "DUT did not GET /dcap and then the Time resource it links to."
  - id: EXAMPLE-001.C2
    title: Requests declare application/sep+xml in Accept
    sunspec_criterion: extension
    verifies_steps: [S2, S3]
    requirements: ["2030.5:GEN.056"]
    evidence: traffic
    severity: mandatory
    check:
      type: request_header
      select: {method: GET, window: {after_step: S2, before_step_end: S3}}
      header: Accept
      op: contains
      value: application/sep+xml
    failure_message: "At least one DUT request lacked 'application/sep+xml' in Accept."
  - id: EXAMPLE-001.C3
    title: Every DUT connection used the mandatory cipher suite
    sunspec_criterion: extension
    verifies_steps: [S2]
    requirements: ["2030.5:SEC.020"]
    evidence: tls
    severity: mandatory
    check:
      type: tls_property
      property: cipher_suite
      op: equals
      value: TLS_ECDHE_ECDSA_WITH_AES_128_CCM_8
    failure_message: "A DUT TLS connection negotiated a different cipher suite."
  - id: EXAMPLE-001.C4
    title: Operator confirms the DUT synchronised its clock
    sunspec_criterion: extension
    verifies_steps: [S3, S4]
    requirements: ["2030.5:TIME.001"]
    evidence: operator_attestation
    severity: recommended
    check: {type: operator_attestation, attestation: dut_clock_synced, expected: "yes"}
    failure_message: "Operator reported that the DUT clock was not synchronised."
    rationale: Clock adoption is internal to the DUT and not visible on the wire.
  - id: EXAMPLE-001.C5
    title: Harness served a schema-valid Time payload
    sunspec_criterion: extension
    verifies_steps: [S3]
    requirements: []
    evidence: harness_self_check
    severity: mandatory
    check:
      type: xsd_valid
      select: {method: GET, resource: tm}
      source: response_body
      root: Time
      allow_empty: true   # DUT may never ask (see N2); that is judged by C1, not here
    failure_message: "Harness bug: Time response was not XSD-valid."

timeouts:
  wait_for_client_s: 600
  run_s: 300
  default_step_s: 120
  settle_s: 10
  attestation_s: 300

negative_cases:
  - id: EXAMPLE-001.N1
    title: Client sends a non-2030.5 Accept header
    fault:
      kind: alter_request_header
      refclient_flag: accept_text_xml
      target: {method: GET}
      params: {header: Accept, value: "text/xml"}
      description: "Every GET carries the header Accept set to text/xml instead of application/sep+xml."
    expected:
      run_verdict: FAIL
      failing_criteria: [EXAMPLE-001.C2]
      passing_criteria: [EXAMPLE-001.C3]
  - id: EXAMPLE-001.N2
    title: Client never reads Time
    fault:
      kind: omit_request
      refclient_flag: skip_time_get
      target: {method: GET, resource: tm}
      description: Client reads /dcap but ignores TimeLink.
    expected:
      run_verdict: FAIL
      failing_criteria: [EXAMPLE-001.C1]

notes:
  - id: NOTE-1
    kind: scope
    text: Fictional fixture for format documentation and loader unit tests only.
    open_question: false
  - id: NOTE-2
    kind: interpretation
    text: "C2 is mandatory only to illustrate a negative case. In real BASIC tests the Accept-header criterion is severity recommended (WARN); strict HTTP checks belong to CORE tests."
    open_question: false
```
