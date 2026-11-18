# ADR 0005 — `no_requirement_reason`, test definition format 1.3

Status: accepted
Date: 2026-11-18
Owner: spec-analyst
Related: [ADR 0001](0001-test-definition-format.md),
[ADR 0002](0002-test-definition-format-1.1.md),
[ADR 0003](0003-test-definition-format-1.2.md),
[ADR 0004](0004-step-completion-counting.md)

## Context
Rule R9 required every non-self-check criterion to cite at least one
requirement ID. In the BASIC-001 review, Keyur limited citations to the IDs
SunSpec prints for the test (ADR 0003 decision 5, BASIC-001 NOTE-10). Some
measurable criteria are then covered by none of those IDs, which violated R9.

## Decision
Chosen by Keyur 2026-11-18 to keep the requirements matrix limited to IDs
SunSpec prints for the test:
- New optional criterion field `no_requirement_reason` (non-empty string):
  why the criterion cannot be traced to a printed requirement ID.
- R9 now reads: every criterion has at least one requirement ID unless
  `evidence: harness_self_check` is set or `no_requirement_reason` is given.
  Mutual exclusion: a criterion with a non-empty `requirements` list MUST NOT
  set `no_requirement_reason`.
- Criteria without requirement IDs produce no rows in
  `definitions/requirements_matrix.csv`.
- `no_requirement_reason` is part of the v1 subset (`criterion_fields`).
- `format_version` const, `$id`, header comment, README and README example
  bumped to **1.3**.

## First users
BASIC-001 criteria C1, C4, C5, C9 and C11.

## Consequences
- harness-engineer: loader accepts format 1.3 only and enforces the revised R9
  including the mutual exclusion; the report shows `no_requirement_reason` for
  untraced criteria.
- The requirements matrix remains a strict copy of SunSpec-printed IDs.
