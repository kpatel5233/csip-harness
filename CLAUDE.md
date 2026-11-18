# csip-harness — Project Rules (read by every agent)

## 1. What we are building
`csip-harness` is an IEEE 2030.5 / CSIP (California Rule 21) **conformance test harness**,
similar in purpose to the QualityLogic test tool, running as a service on a Raspberry Pi or
any Linux server.

- The **DUT** (device under test) is a **gateway acting as a 2030.5 DER client**
  (or aggregator client). Utility-server testing is **out of scope**.
- The harness plays the **2030.5 server**. The DUT connects to *us* and polls.
- A user starts a test through a **REST Control API**, e.g. `POST /runs {test: "BASIC-001", dut: {...}}`,
  then points the DUT at the harness URL, and reads the PASS/FAIL report when the run ends.
- Test cases are transferred from **SunSpec CSIP Conformance Test Procedures v1.3** (baseline).
  Every test and criterion keeps its SunSpec ID so we can diff against v1.4 later.
- Architectural reference: BSGIP **CACTUS** (CSIP-AUS) — YAML test definitions + per-run
  test server + runner. We follow the pattern, not the code.

## 2. Architecture
```
User/CI ──► Control API (REST) ──► Run Orchestrator ──► loads YAML test definition
                                          │
                              2030.5 Test Server (TLS 1.2) ◄── Recorder
                                          │
                                      Evaluator ──► Results DB (SQLite) + Report
                              PKI (test CA)          ▲
                                                     │ HTTPS / 2030.5
                                   DUT gateway  or  Reference Client
```
Run lifecycle states: `CREATED → PROVISIONED → WAITING_FOR_CLIENT → RUNNING → EVALUATING → PASS | FAIL | TIMEOUT | ABORTED | ERROR`

v1 constraint: **one active run at a time**. Control API and 2030.5 server listen on separate ports.

## 3. Repository layout and ownership
| Path | Owner agent | Content |
|---|---|---|
| `definitions/` | spec-analyst | YAML test procedures, YAML schema, requirement matrix |
| `docs/` | spec-analyst (+ lead) | architecture, decision records (`docs/adr/`), reviews (`docs/reviews/` by reviewer) |
| `harness/` | harness-engineer | 2030.5 server, sep.xsd model, identity (LFDI/SFDI/PIN), TLS, recorder, evaluator |
| `pki/` | harness-engineer | test CA tooling, cert issuing/validation |
| `api/` | platform-engineer | REST Control API, run orchestrator, DB |
| `deploy/` | platform-engineer | Dockerfile, docker-compose, Pi setup |
| `refclient/` | refclient-engineer | basic reference DER-client gateway simulator |
| `tests/e2e/` | reviewer | end-to-end tests (API → run → refclient → report) |

**An agent only writes inside the paths it owns.** If it needs a change elsewhere, it writes
a request in its final report and the lead routes it.

## 4. Workflow per test case
1. **spec-analyst** writes `definitions/<ID>.yaml` + updates requirement matrix.
2. **GATE 1 — human (Keyur) approves the YAML.** No implementation before approval.
3. **harness-engineer / platform-engineer / refclient-engineer** implement in parallel.
4. **reviewer** reviews against YAML + SunSpec and runs e2e (positive and negative case).
5. **GATE 2 — human approves** the review report → merge.

The approved YAML is the **single source of truth**. Code that disagrees with it is wrong;
a YAML that disagrees with SunSpec goes back to spec-analyst.

## 5. Independence rule
- `refclient-engineer` works **from the YAML definitions and public standards only**.
  It must NOT read `harness/`, `api/` or `pki/` source. This prevents the harness and the
  reference client from sharing the same misunderstanding.
- `reviewer` does not fix code. It reports findings; the owner fixes them.

## 6. Protocol facts (verify against the standard, do not guess)
- IEEE 2030.5-2018, XML namespace `urn:ieee:std:2030.5:ns`, media type `application/sep+xml`.
- TLS 1.2, mutual auth, cipher `TLS_ECDHE_ECDSA_WITH_AES_128_CCM_8`, curve secp256r1.
  (Must be explicitly enabled in Python `ssl` / OpenSSL.)
- Certificate chain shape: SERCA → MICA → device cert. We generate a **lab PKI** with the same shape.
- **LFDI** = first 160 bits of SHA-256 over the DER-encoded device certificate (40 hex digits).
- **SFDI** = left-most 36 bits of the LFDI as decimal + 1 check digit (digit sum ≡ 0 mod 10).
- **PIN** = 6 digits, last digit is a check digit.
- Note: SunSpec v1.3 BASIC-001 step 4 calls LFDI "16-bit" — this is a typo; LFDI is 160-bit.

## 7. Engineering conventions
- Python **3.11** (Raspberry Pi OS Bookworm baseline), package manager `uv`.
- `ruff` (lint + format), `mypy --strict` on `harness/` and `api/`, `pytest` for unit tests.
- Type hints everywhere; dataclasses / pydantic for models.
- XML model generated from `sep.xsd` (xsdata); every outgoing and incoming payload is XSD-validated.
- Logging: structured (JSON), every 2030.5 exchange recorded with timestamp, method, path, status, body.
- No hidden network calls; no secrets or private keys committed (`pki/out/` is git-ignored).
- Must run on ARM64 (Pi) — avoid dependencies without ARM wheels.

## 8. Definition of Done (per test case)
- Approved YAML exists with SunSpec ID, steps, criteria, requirement IDs.
- Harness executes it; evaluator produces a verdict **per criterion**.
- Reference client passes it (positive e2e) **and** a deliberately broken client fails it on the
  expected criterion (negative e2e).
- Unit tests pass, ruff + mypy clean, reviewer report has no open blocking findings.

## 9. Copyright
SunSpec and IEEE documents are copyrighted. In YAML and docs: reference the section and
test/step ID and **paraphrase** — do not paste procedure text verbatim. Never commit the PDFs.

## 10. Agent final report format
Every agent ends its task with:
- **Done:** what was produced (paths)
- **Decisions:** anything decided that others must know
- **Requests:** changes needed in paths owned by other agents
- **Open questions:** for the lead / human
