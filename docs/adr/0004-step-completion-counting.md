# ADR 0004 — Step completion counts exchanges from run start

Status: accepted
Date: 2026-11-18
Owner: spec-analyst
Related: [ADR 0001](0001-test-definition-format.md),
[ADR 0002](0002-test-definition-format-1.1.md),
[ADR 0003](0003-test-definition-format-1.2.md)

## Context
`dut` steps complete when their `expect` list is satisfied (`complete_when`,
default `all_expected_seen`). The format did not say from when matching
exchanges are counted. A real DUT can make the requests of several steps in a
burst (e.g. in BASIC-001 it may fetch its Registration and FSA list before
S5/S7 formally start), so counting from the step start would make steps wait
for requests that already happened.

## Decision
- **Default:** for `dut` step completion, an expected exchange counts if it was
  recorded at any time since `run.start`, regardless of when the step started.
- Criteria are evaluated on the whole recording unless their selector restricts it.
- **Later tests MUST** use the selector `window` (`after_step` /
  `before_step_end`) whenever counting must start at, or end with, a particular
  step (e.g. "the DUT re-reads a resource after the harness changed it").
  Relying on step order alone for such tests is not allowed.
- Accepted by Keyur on 2026-11-18 (BASIC-001 review); BASIC-001 NOTE-11 is
  resolved by this ADR.

## Consequences
- harness-engineer: the orchestrator matches `expect` entries against all
  exchanges since `run.start`; `window` is honoured in criteria selectors (it is
  in the v1 subset).
- spec-analyst: definitions with time-ordered behaviour (CORE/BASIC event tests)
  must express ordering with `window` or `request_order`.
