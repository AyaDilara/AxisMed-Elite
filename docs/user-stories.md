# User Stories & Acceptance Criteria — AxisMed Elite

This document specifies user stories with acceptance criteria for AxisMed Elite's core flows. It covers four flows: the anti-doping prescribing safeguard (proposed, specified in full in `prd-antidoping.md`), and three flows reconstructed from the current system — patient registration and records, appointment and clearance scheduling, and authentication and access control.

**Acceptance-criteria format.** Flow A (the anti-doping safeguard) is specified in scenario form (Given / When / Then), because its value lives in behaviour under different conditions — list available, list unavailable, substance clean, substance flagged. Flows B–D are specified as rule-based checklists, because their behaviour is a set of fixed invariants (a duplicate is rejected, a conflict is rejected, a role is enforced) with no branching flow to model. The format follows the story, not the reverse.

**Status legend.**
- `Reconstructed` — specifies behaviour already implemented in the current codebase. Acceptance criteria are anchored to verified service-layer behaviour.
- `Proposed` — specifies net-new behaviour not yet built. Written before implementation as forward specification.

## Story index

| ID | Story | Flow | Status |
|----|-------|------|--------|
| A1 | Prescribe medication | Prescribing + anti-doping | Reconstructed |
| A2a | Warn on always-prohibited substance | Prescribing + anti-doping | Proposed (v1) |
| A2b | Warn on in-competition-only substance | Prescribing + anti-doping | Proposed (v2) |
| A3 | Fail loud when the list can't be checked | Prescribing + anti-doping | Proposed |
| A4 | Record justification for a flagged substance | Prescribing + anti-doping | Proposed |
| A5 | Audit the justified prescription | Prescribing + anti-doping | Proposed |
| A6 | Spike — prohibited-substances data source | Prescribing + anti-doping | Proposed (investigation) |
| B1 | Register patient with type | Registration & records | Reconstructed |
| B2 | Prevent duplicate registration | Registration & records | Reconstructed |
| B3 | View global medical record | Registration & records | Reconstructed |
| B4 | Team-delegate privacy restriction | Registration & records | Reconstructed |
| C1 | Schedule appointment | Appointment & clearance | Reconstructed |
| C2 | Detect scheduling conflict | Appointment & clearance | Reconstructed |
| C3 | Record clearance test | Appointment & clearance | Reconstructed |
| C4 | Delegate views clearance status | Appointment & clearance | Reconstructed |
| D1 | Role-based login | Auth & access control | Reconstructed |
| D2 | Enforce permissions | Auth & access control | Reconstructed |
| D3 | Audit every action | Auth & access control | Reconstructed |

---

## Flow A — Prescribing + Anti-Doping Safeguard

Full behavioural specification for this flow — problem, users, goals, scope, technical design — is in `prd-antidoping.md`. The stories below are the delivery-level breakdown of that PRD. The safeguard is split by business rule so that the safety-critical case (the list cannot be checked) is independently testable rather than folded into the main warning story. The warning itself is split again along the release boundary defined in the PRD's sequencing section: always-prohibited substances (A2a, v1, exact list match) are separated from in-competition-only substances (A2b, v2), which require event-timing logic the current prescribing flow does not yet have. The v1/v2 decision lives in the PRD; the stories reference it rather than restate it.

### A1 — Prescribe medication `Reconstructed`

As a **doctor**, I want to prescribe a medication or supplement to a patient, so that the patient's record reflects the pharmacological support in their treatment.

**Given** an authenticated doctor and a valid patient
**When** the doctor submits a medication with name, dosage, frequency, and type
**Then** the medication is added to the patient's record
**And** the action is written to the audit log as `PRESCRIBE_MEDICATION` with timestamp and user

**Given** a user whose role is not doctor
**When** they attempt to prescribe
**Then** the prescription is rejected with an unauthorized-action error

### A2a — Warn on always-prohibited substance `Proposed (v1)`

As a **doctor**, I want to be warned when a substance I am prescribing is prohibited at all times, so that the athlete is not later ruled ineligible because of a drug we prescribed.

**Given** a prescribed substance that is prohibited at all times
**When** the doctor submits the prescription
**Then** an interruptive warning naming the substance and its category is displayed
**And** the prescription cannot be saved until the substance is removed or a justification is recorded

**Given** a prescribed substance that is not on the list
**When** the doctor submits the prescription
**Then** no anti-doping warning is shown
**And** prescribing proceeds normally

### A2b — Warn on in-competition-only substance `Proposed (v2)`

As a **doctor**, I want to be warned when a substance is prohibited in-competition and the athlete has an event in the restricted window, so that I am warned when it matters without being blocked for substances permitted outside competition.

Deferred to v2 per the PRD sequencing section: this story depends on wiring the prescribing flow to `SportingEvent` dates, which v1 does not include. Specified here so the deferred behaviour is visible, not lost.

**Given** an in-competition-only substance and an athlete linked to an event within the restricted window
**When** the doctor submits the prescription
**Then** the warning states the restriction is in-competition only
**And** one acknowledgement is required before saving

**Given** an in-competition-only substance and an athlete with no event in the restricted window
**When** the doctor submits the prescription
**Then** the substance is treated as permitted at this time
**And** prescribing proceeds normally

### A3 — Fail loud when the list can't be checked `Proposed`

As a **doctor**, I want the system to withhold the prescription and warn me when the prohibited-substances list cannot be checked, so that a missing or stale list is never mistaken for a clean result.

**Given** the prohibited-substances list is missing or unreadable
**When** the doctor submits any prescription
**Then** an interruptive warning states the substance cannot be verified
**And** the prescription is withheld until the doctor records an explicit acknowledgement
**And** the withheld attempt is written to the audit log

**Given** the loaded list is older than its current annual version by more than the freshness threshold *(threshold value: TBD — resolved by spike A6)*
**When** the doctor submits any prescription
**Then** the system flags that the list may be out of date
**And** a "no warning" result is not presented as a guarantee

**Design note.** This story fails closed: when substance status is unknown, the system withholds rather than proceeds. Absence of a warning must never read as a clearance. The freshness threshold is a parameter this specification references; the value is supplied by the A6 spike and does not block writing or reviewing these criteria.

### A4 — Record justification for a flagged substance `Proposed`

As a **doctor**, I want to record why I am prescribing a flagged substance, so that the decision is defensible in an anti-doping review.

**Given** a substance has been flagged
**When** the doctor chooses to proceed
**Then** a justification category is required before saving — for example Therapeutic Use Exemption on file, out-of-competition use, or clinical override
**And** a proceed attempt with no justification is rejected

**Given** a justification is recorded
**When** the prescription is saved
**Then** the substance, category, justification text, timestamp, and prescribing doctor are persisted with the prescription

### A5 — Audit the justified prescription `Proposed`

As a **compliance reviewer** *(proposed role — see note)*, I want every justified prohibited-substance prescription recorded, so that overrides can be reviewed after the fact by a party independent of the prescriber.

**Given** a doctor saves a prescription with a recorded justification
**When** the save completes
**Then** an audit entry records substance, justification category, justification text, doctor, patient, and timestamp
**And** the entry is append-only and cannot be edited

**Given** a team delegate later views clearance status for the same athlete
**When** a justified prohibited-substance prescription exists on record
**Then** the delegate sees clearance status only, never the justification — the existing privacy boundary holds

> **Proposed-role note.** *Compliance reviewer* is not a role in the current system, which has four roles: admin, doctor, patient, team delegate. This story assumes a fifth, read-only oversight role whose sole purpose is to consume the audit trail. Its rationale and modelling are specified in `gaps-and-recommendations.md`. Until that role exists, the audit entry is produced and retained, but its intended independent consumer does not yet exist.

### A6 — Spike: prohibited-substances data source `Proposed (investigation)`

As the delivery team, we need to determine how the prohibited-substances list is sourced, structured, matched, and aged, so that stories A2 and A3 become estimable and their parameters can be fixed.

This is a timeboxed investigation, not a shippable story. It resolves:
- Source and format of the list (static version-stamped file for v1; official machine-readable source deferred to the RFC).
- Substance matching — how an entered substance maps to a listed one across brands, spellings, and combination products. This is the primary open risk named in the PRD.
- The freshness threshold at which a loaded list is treated as stale (the value referenced by A3).

Stories A2 and A3 are not estimable until this investigation closes.

---

## Flow B — Patient Registration & Records

### B1 — Register patient with type `Reconstructed`

As an **admin**, I want to register a patient with a type classification, so that the system applies the correct handling for each patient category.

- A patient can be registered with a type of LOCAL, VISITING_INDIVIDUAL, VISITING_TEAM, or CHRONIC.
- On successful registration the patient is created with a global medical record.
- The registration is written to the audit log.

### B2 — Prevent duplicate registration `Reconstructed`

As an **admin**, I want a registration with an already-registered email rejected, so that one patient never holds two conflicting records.

- Registration with an email already present in the system is rejected with a duplicate-record error.
- No patient record is created when the email already exists.

### B3 — View global medical record `Reconstructed`

As a **doctor**, I want to view a patient's complete history regardless of the city that created each entry, so that I decide with the full record.

- A patient's diagnoses, treatment plans, functional tests, and appointments are returned as one unified record.
- Records created at any clinic location appear in the same unified history.

### B4 — Team-delegate privacy restriction `Reconstructed`

As a **team delegate**, I want to see only an athlete's clearance status and never their medical detail, so that athlete medical privacy is preserved.

- A team-delegate request for a patient's full history is rejected with an unauthorized-action error.
- The restriction is enforced at the service layer, not in the model.

---

## Flow C — Appointment & Clearance

### C1 — Schedule appointment `Reconstructed`

As an **admin**, I want to schedule an appointment for a patient with a staff member at a clinic, so that the patient is seen at a defined time.

- A scheduled appointment is created with a patient, staff member, clinic, date/time, and type.
- A newly scheduled appointment is set to SCHEDULED status.
- The scheduling action is written to the audit log.

### C2 — Detect scheduling conflict `Reconstructed`

As an **admin**, I want a scheduling attempt that collides with an existing appointment rejected, so that no double-booking occurs.

- A scheduling attempt for a time slot already taken is rejected with a scheduling-conflict error.
- No appointment is created when a conflict is detected.

### C3 — Record clearance test `Reconstructed`

As a **doctor**, I want to record a medical or doping clearance with a CLEARED or NOT_CLEARED result, so that an athlete's eligibility is officially on record.

- A CLEARED result requires a valid-until date; a CLEARED result without one is rejected with an invalid-clearance error.
- A NOT_CLEARED result is recorded without a valid-until date.
- A doping-screen clearance is recorded distinctly from a medical clearance.

### C4 — Delegate views clearance status `Reconstructed`

As a **team delegate**, I want to view whether a linked athlete is cleared, so that I know their availability without accessing medical detail.

- A linked athlete's clearance result and valid-until date are returned.
- No diagnosis, treatment, or other medical detail is included in the delegate's view.

---

## Flow D — Authentication & Access Control

### D1 — Role-based login `Reconstructed`

As **any user**, I want to log in and receive the menu for my role, so that I only see operations I am permitted to perform.

- Valid credentials establish a session for the matching role: admin, doctor, patient, or team delegate.
- The role-specific menu is presented on successful login.
- A successful login is written to the audit log.
- Invalid credentials do not establish a session.

### D2 — Enforce permissions `Reconstructed`

As a **patient** *(or any non-privileged role)*, I want operations my role cannot perform to be rejected, so that permissions cannot be bypassed.

- A non-doctor attempting to record a diagnosis is rejected with an unauthorized-action error.
- Sensitive clinical operations are checked at two independent levels: the session role and the acting staff member's own capability.
- Passing only one of the two checks does not authorize the operation.

### D3 — Audit every action `Reconstructed`

As an **administrator**, I want every service action recorded with action, timestamp, and user, so that the system holds a complete history.

- Every service-layer operation appends an audit entry with action name, ISO timestamp, and user email.
- The audit log is append-only.

*The proposed compliance-reviewer role (see A5 and `gaps-and-recommendations.md`) is the intended independent consumer of this log.*
