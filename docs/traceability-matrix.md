# Requirements Traceability Matrix — Anti-Doping Prescribing Check

Traces every requirement of the anti-doping safeguard from its source in `prd-antidoping.md` through the story that delivers it, the acceptance criterion that defines "done," and how it would be verified. Scope is the proposed anti-doping feature only; the reconstructed flows (B/C/D) are existing behaviour documented in `user-stories.md`, not feature requirements, and are out of scope here.

**Why this exists.** The matrix catches two failures a coverage check is meant to find: an **orphan requirement** (specified but with no story or test — dropped scope) and **gold-plating** (a story with no originating requirement — scope creep). Every row below closes left-to-right with no gap; the coverage summary at the end states what that proves and what it does not.

**Verification types.** `Deterministic` — testable now against the full prohibited list, no users needed (verification). `Field` — needs real deployment and traffic to confirm (validation, not available pre-launch). `Review` — confirmed by inspection of the artifact or audit record. This column is deliberately honest about which requirements can be proven now and which only after deployment.

---

## Matrix

| Req ID | Requirement (source: PRD) | Story | Acceptance criterion | Verification |
|---|---|---|---|---|
| R1 | A prescribed always-prohibited substance must be flagged before save, for regulated patients only | A2a | Interruptive warning naming substance + category; save blocked until removed or justified | Deterministic — feed every always-prohibited substance on the list, assert each flags |
| R2 | A non-regulated patient must never trigger the check | A2a, A7 | Patient not `subjectToAntiDoping` → no check runs, prescribing proceeds | Deterministic — prescribe to a non-flagged patient, assert no warning |
| R3 | A permitted (unlisted) substance must not be flagged | A2a | Substance not on list → no warning, proceeds normally | Deterministic — feed known permitted substances, assert zero flags (false-flag guardrail) |
| R4 | The check must run only for patients subject to anti-doping rules (the gate) | A7 | Patient stored with `subjectToAntiDoping`; value available to the check as its gate | Review — inspect the gate field and its read at the hook point |
| R5 | Anti-doping status must be captured at registration | A7 | Admin sets status on registration/edit; legacy patients default to subject-to-anti-doping until reviewed | Review — registration flow captures the field; migration default confirmed |
| R6 | When the list cannot be checked, the prescription must be withheld, not silently cleared (fail loud) | A3 | Missing/unreadable list → interruptive warning (cause + not-a-clearance + manual-verify action), save withheld until acknowledged, attempt audited | Deterministic — simulate list unavailable, assert withhold + audit entry |
| R7 | A stale list must not be mistaken for a clean result | A3 | List older than current version by the freshness threshold → flags possible staleness; "no warning" not presented as a guarantee | Deterministic (once threshold set by A6) — load an out-of-date list, assert staleness flag |
| R8 | A flagged substance may be prescribed only with a recorded justification | A4 | Proceed requires a justification category (TUE / out-of-competition / clinical override); no-justification proceed is rejected | Deterministic — attempt proceed without justification, assert rejection |
| R9 | A justified override must be recorded immutably for review | A4, A5 | Append-only audit entry: substance, category, justification, doctor, patient, timestamp | Review — inspect audit record is written and non-editable |
| R10 | Override records must be reviewable by a party independent of the prescriber | A5 | Compliance reviewer (proposed role) consumes the justification audit trail; read-only | Review — role consumes the record; Field — reviewer verdicts feed the counter-metric |
| R11 | The justification must not leak across the team-delegate privacy boundary | A5 | Delegate viewing clearance sees status only, never the justification | Deterministic — delegate view asserts no justification exposed |
| R12 | List source, substance-matching approach, and staleness threshold must be resolved before A2a/A3 are estimable | A6 (spike) | Investigation resolves source/format, matching, and the freshness-threshold value | Review — spike output closes the three open parameters |
| R13 | The primary outcome (prohibited prescriptions without justification → zero) must be measurable | A5 (audit source) | Measured per quarter from the audit log; leading proxy = catches heeded | Field — lagging outcome; confirmed only at a real clinic over time |
| R14 | Doctor trust must not be eroded by over-flagging (guardrail) | A2a, A3 | Unwanted-flag rate < 5% on regulated-patient prescriptions | Field — requires real traffic to measure; Deterministic lower bound via R3 |
| R15 | The check must not add intolerable friction (UX guardrail) | A2a | Median time added per prescription within bound (threshold TBD, usability testing) | Field — requires usability testing to set bound and measure |

---

## Coverage summary

- **15 requirements, all traced** left-to-right to a story, an acceptance criterion, and a verification method. No orphan requirement (every requirement has a story and a test); no gold-plating (every anti-doping story A2a–A7 appears as the delivery vehicle for at least one requirement).
- **Verifiable now (deterministic):** R1, R2, R3, R6, R8, R11, and R7 once A6 sets the threshold. These can be proven against the prohibited list and the code without any users — the feature's *correctness*.
- **Reviewable:** R4, R5, R9, R10, R12 — confirmed by inspecting the field, the flow, the audit record, or the spike output.
- **Field-only (validation, not available pre-launch):** R13, R14, R15, and the field half of R10. These depend on real deployment, traffic, or usability testing. They are specified and traced, but the matrix does not claim they are tested — consistent with the PRD's verification-vs-validation boundary.

**What this proves:** the specification is complete and internally consistent — every requirement reaches a testable definition of done, and the feature can be *verified* for correctness now. **What it does not prove:** effectiveness in practice, which is a field claim the matrix explicitly defers rather than fakes.