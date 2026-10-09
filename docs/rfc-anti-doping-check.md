# RFC-01 — Anti-Doping Prescribing Check: Technical Design

**Status:** Draft (open for review) · **Author:** Aya-Dilara · **Related:** `prd-antidoping.md` (the what/why this implements), `architecture-decisions.md` (ADR-001–005), `rfc-substance-matching.md` (RFC-02, the matching sub-design)

The PRD specifies what the anti-doping check does and why; it defers the *how* to this RFC. This document is that how: the data model, the service-layer mechanics, the fail-loud behaviour, the instrumentation, and the external-list integration path. One problem — matching an entered medication to the prohibited list — is deep enough to have its own document (RFC-02); this RFC treats it as a dependency and points there rather than re-deriving it.

Scope: the v1 feature as a whole, in the existing three-layer architecture (UI → service → repository). Where a decision is already recorded as an ADR, this RFC implements it rather than re-arguing it.

---

## 1. Components and change surface

The feature touches four places and adds one. Nothing new architecturally (ADR-003).

- **New — `ProhibitedSubstanceRepository`** — loads `prohibited_substances.csv` at startup, following the existing generic `CSVReaderService<T>` pattern; holds the list in memory keyed by canonical substance name for O(1) lookup.
- **Changed — `ClinicService.prescribeMedication()`** — the hook point. The check runs after substance entry and before the `Medication` is persisted and audited.
- **Changed — `Patient`** — gains the `subjectToAntiDoping` field (the gate, ADR-001) and `Sport.NONE`/`OTHER` for data-model completeness (off the safety path).
- **Changed — `Medication`** — gains an optional justification reference (category, note, timestamp, doctor).
- **Changed — registration flow (`registerPatient`)** — captures `subjectToAntiDoping` (story A7).

The existing `AuditService` is reused for the justification record — no new persistence path.

---

## 2. Data model

**Prohibited list** (`prohibited_substances.csv`):
- `substance` — canonical active-substance name (naming authority resolved by the A6 spike; see RFC-02 §6).
- `category` — `always` | `in_competition`.
- `list_version` — the annual edition year, used for the staleness check.

**Patient — new field:**
- `subjectToAntiDoping` — boolean gate. Modelled as a slowly-changing attribute: the *current* value is stored on the patient, and changes are recorded as dated events (§5) so a metric computed over a past period can use the value that was true *then*, not the value now. Without the dated history, a retrospective guardrail reading would filter historical prescriptions by a patient's present status — a known correctness trap. For v1's small population, this is simple; the event log is what makes it correct at scale.

**Medication — new field:**
- `justification` — optional `{ category, note, timestamp, doctorId }`, present only when a flagged substance was overridden (ADR of record: written through `AuditService`).

**Substance matching** — the controlled-vocabulary approach and its canonicalisation sub-problem are RFC-02; this RFC assumes a `match(substance) → {hit, category}` capability exists and is deterministic.

---

## 3. Control flow at the hook

Inside `prescribeMedication()`, in order:

1. **Permission** — existing `requireRole(DOCTOR)` + capability check (unchanged).
2. **Gate (ADR-001)** — if `patient.subjectToAntiDoping` is false, skip the check entirely; proceed to persist. Non-regulated patients never reach the matcher — this is the false-flag guardrail, enforced structurally.
3. **Match** — for each selected substance, call the matcher (RFC-02).
   - *List unreadable or stale past threshold* → **fail loud** (ADR-002): withhold the save, warn with cause + "not a clearance" + required manual action, audit the withheld attempt. Do not return "no match."
   - *Hit* → raise an interruptive warning naming substance + category; block save.
   - *No hit* → proceed.
4. **Justification (on a flagged proceed)** — require a justification category; reject a proceed with none; attach the justification to the `Medication` and write it to the audit log.
5. **Persist + audit** — unchanged tail: create `Medication`, add to record, `AuditService` logs the action (now carrying justification when present).

The check is synchronous and blocks the save (NFR, ADR-002): a warning that can be bypassed by ignoring it is not a control.

---

## 4. Fail-loud mechanism (ADR-002, in detail)

The matcher returns one of three outcomes, never a silent pass: `CLEAR`, `FLAGGED(category)`, `UNVERIFIABLE(reason)`. The third is the safety-critical one. It is produced when:
- the list file is missing or unreadable at the time of the check, or
- the loaded `list_version` is older than the current edition by more than the staleness threshold (threshold value from the A6 spike).

`UNVERIFIABLE` routes to the withhold-and-warn path, not the proceed path. This is a positive return value the matcher must emit, not an exception swallowed into a default — the design makes "we could not check" a first-class result so it can never collapse into "no warning." The staleness check requires the `list_version` field (§2) to be present and compared on every check, not only at load.

---

## 5. Instrumentation design

The PRD's metrics are only real if the events they are computed from are emitted and named consistently. This section specifies that layer — the bridge from "metrics defined" to "metrics collectible."

**Events** (object_action, past tense, snake_case; one behaviour each):
- `prescription_submitted` — a prescription reaches the hook.
- `prohibited_flag_raised` — the matcher returned `FLAGGED`.
- `list_check_failed` — the matcher returned `UNVERIFIABLE`.
- `override_recorded` — a doctor proceeded on a flag with a justification.
- `override_reviewed` — the compliance reviewer rendered a verdict.
- `antidoping_status_changed` — a patient's `subjectToAntiDoping` value changed (the dated event backing §2's slowly-changing attribute).

**Properties** (context on an event, never values baked into the event name):
- on `prescription_submitted`: `patient_id`, `doctor_id`, `subject_to_antidoping` (the as-of value at prescription time), `time_added_ms`.
- on `prohibited_flag_raised`: `substance`, `category`, `was_overridden`, `list_version`.
- on `list_check_failed`: `reason` (`missing` | `stale`), `list_version`.
- on `override_recorded`: `substance`, `justification_category`, `doctor_id`.
- on `override_reviewed`: `verdict` (`justified` | `unjustified`), `reviewer_id`.

**User traits** (persistent attributes of the patient/user):
- `subjectToAntiDoping` (patient) — slowly-changing, versioned via `antidoping_status_changed`.
- `role` (user), `primarySport` (patient).

**How each metric composes from this layer** (this is the proof the metrics are grounded, not words):
- *Primary (catches / violations):* `prohibited_flag_raised` where `was_overridden` with a legitimate justification = a prevention event (the leading proxy); the lagging violation count is its field confirmation.
- *Unwanted-flag guardrail:* `prohibited_flag_raised` filtered to permitted-substance or non-regulated cases ÷ `prescription_submitted` where `subject_to_antidoping` true.
- *Counter-metric:* `override_reviewed` where `verdict = unjustified` ÷ `override_reviewed`, reported with review coverage.
- *UX guardrail:* median `time_added_ms` on `prescription_submitted` where `subject_to_antidoping` true.

Every metric decomposes cleanly into events + properties + traits — which is why they are computable rather than aspirational.

---

## 6. External list integration (v3 path)

v1 loads a static version-stamped CSV (ADR-005). v3 sources an official, maintained list. The design keeps that change contained: the matcher depends on the `ProhibitedSubstanceRepository` interface, not on CSV specifically, so swapping the backing source (a scheduled import, or a live API) is a repository-level change, not a service-layer one. When the source becomes a live external service it becomes a separate participant (the process-maps' v3 pool): its availability and latency become first-class, and the `UNVERIFIABLE` path (§4) already models "cannot reach the list" — so the fail-loud behaviour extends to a network failure without redesign. The canonical-naming compatibility between the external source and the controlled vocabulary is an open question for RFC-02 §9.

---

## 7. Non-functional requirements

- **Synchronous, blocking:** the check completes before the save returns (ADR-002).
- **In-memory list:** loaded once at startup, keyed for O(1) lookup; re-load is a restart concern in v1, a refresh concern in v3.
- **Fail loud, not open:** unreadable/stale list → withhold (§4).
- **Performance budget:** the check sits on the prescribing path and feeds the UX time guardrail; its added latency must stay within the (TBD) time-added bound. In-memory lookup makes the match itself negligible; the budget is really about the justification UX, not the computation.

---

## 8. Testing (how this RFC is verified)

- **Unit — control flow:** gate skips non-regulated; flag blocks; no-hit proceeds; unverifiable withholds. (Covers RTM R1, R2, R3, R5.)
- **Unit — fail loud:** list missing and list stale both yield `UNVERIFIABLE` → withhold + audit. (RTM R5, R6.)
- **Unit — justification:** proceed without justification rejected; with justification persisted and audited. (RTM R7, R8.)
- **Integration — privacy:** delegate view never exposes justification. (RTM R10.)
- **Integration — instrumentation:** each event fires with its properties; a metric query over the events returns the expected figure on a fixtured dataset.
- **Deferred to RFC-02:** the exhaustive match-against-full-list correctness test.

Verification establishes correctness; effectiveness remains a field claim (PRD's verification-vs-validation boundary).

---

## 9. Open questions

- Staleness threshold value and canonical naming authority — A6 spike (shared with RFC-02).
- Whether `antidoping_status_changed` history is built in v1 or deferred — v1's tiny population makes the current-value-only shortcut temporarily safe, but the event is cheap and prevents a later backfill; recommend building it from the start.
- Re-load/refresh strategy for the in-memory list between v1 (restart) and v3 (live) — sequenced, but the trigger mechanism is unspecified.
- `canPrescribe()` vs `canDiagnose()` (gaps finding 5): the hook currently rides on `canDiagnose()`; whether to split before or after this feature ships is a sequencing call.