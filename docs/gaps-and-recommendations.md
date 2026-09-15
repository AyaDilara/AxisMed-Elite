# Gaps & Recommendations — AxisMed Elite

Gaps identified by analysis of the current system, each with the evidence it rests on, its impact, and a recommendation. Findings are grouped by theme, most consequential first: the anti-doping design gap and its dependencies, then access-control consistency gaps surfaced during the same analysis. Every finding is verified against the source — the evidence line says how, so a reader can check it against the repository.

Entry 1 is the flagship design gap the anti-doping specification addresses. Entries 2–3 are its dependencies. Entries 4–6 are access-control gaps surfaced while analysing the prescribing and clearance flows; they are independent of the anti-doping work and each is a small, self-contained fix.

---

## 1. Prescribing does not cross-check anti-doping status `Critical` `being addressed`

**Finding.** A doctor can prescribe a substance on the WADA Prohibited List to a competing athlete with no signal from the system, causing the athlete to fail the very clearance the clinic certifies.

**Evidence (code inspection).** A prohibited-substance check exists nowhere in `src/`. Its only architecturally valid location, `ClinicService.prescribeMedication()`, records and audits the medication with no reference to a prohibited-substance list, the athlete's `SportingEvent` participation, or clearance status. `Medication` is a plain data class with no validation. The prescribing and clearance flows never cross-reference.

**Impact.** Compliance and safety. The harm is low-or-unknown frequency but catastrophic and irreversible — a wrongly prescribed substance can end an athlete's eligibility and expose the clinic to liability — and the system currently offers no barrier against it. This is the clinic's defining constraint.

**Recommendation.** Close per `prd-antidoping.md`: an interruptive, fail-loud check at the prescribing hook, gated on the patient's anti-doping status, with a logged justification (Therapeutic Use Exemption) path. Specified and sequenced; v1 in progress.

---

## 2. No way to distinguish a regulated patient from a non-regulated one `High`

**Finding.** The system cannot tell an athlete subject to anti-doping rules from a hobbyist, chronic-care, or retired patient. Without that distinction the anti-doping check would either run on everyone (over-flagging, tripping its own guardrail) or not at all.

**Evidence (code inspection).** `Patient` carries `primarySport` (a `Sport`), `patientType`, and `membershipType`, but none reliably answers "is this person subject to anti-doping rules." `primarySport` records that someone does a sport, not that they compete under WADA — a weekend swimmer and a retired professional both carry one. No `subjectToAntiDoping` field exists.

**Impact.** The anti-doping check has no correct gate. Gating on `primarySport` over-includes hobbyists and the retired; at this clinic's patient mix that over-flagging is large enough to breach the <5% false-flag guardrail on volume alone.

**Recommendation.** Add a dedicated `subjectToAntiDoping` boolean, set at registration (story A7), and gate the check on it — decoupled from the `Sport` taxonomy so the safeguard never depends on classifying sports correctly. The accepted v1 error direction is over-inclusion (flag a retired athlete) over under-inclusion (miss an active one), since missing a regulated athlete is the catastrophic failure. Separately, add `Sport.NONE` and `Sport.OTHER` for data-model completeness — these are tidiness, not on the safety path. The flag is a maintained field: stale at scale unless captured progressively at each patient's next touchpoint.

---

## 3. No independent oversight of prescribing and clearance decisions `Medium-High`

**Finding.** No role reviews the audit trail — in particular the prohibited-substance justifications the anti-doping check will produce — independently of the staff who prescribe and certify.

**Evidence (code inspection).** The four roles are ADMIN, DOCTOR, PATIENT, TEAM_DELEGATE. The audit log (`AuditService`) is written on every action but has no dedicated read-only consumer. Worse, the admin account is not a clean auditor: `admin@axismed.com` links to `STAFF001`, stored in `staff.csv` with `role=DOCTOR`, and the `StaffRepository` factory has no ADMIN case — it defaults to constructing a `Doctor`. So the admin's `Staff` object holds `canDiagnose() == true`; admin is prevented from clinical actions only by the session-level role check, not by its object.

**Impact.** Separation of duties is absent. The parties who prescribe and clear athletes are the same parties who would audit those decisions — the audit is only as trustworthy as the people it audits.

**Recommendation.** Add a read-only **Compliance Reviewer** role (proposed fifth role) that consumes the audit log and the justification records, holds no clinical capability, and cannot alter records. Model it as a `UserRole` with no `Staff` object — deliberately not mirroring the current admin, whose `Doctor` stub carries clinical capability an auditor must not have. This is the oversight half of Entry 1; it is the intended consumer of stories A5 and D3.

---

## 4. Clearance recording is missing its service-layer permission gate `Low` `becomes real under the planned API`

**Finding.** `recordClearanceTest` is protected at the menu layer only, not at the service layer — the single sensitive operation gated at one level instead of two.

**Evidence (code inspection).** The clearance option appears only in the doctor menu (`ConsoleMenu.doctorRecordClearanceTest`, `case 8` inside `showDoctorMenu`; menus are dispatched by role), so a patient or delegate never reaches it through the console. But every *other* sensitive operation also enforces a service-layer check — `registerPatient` (ADMIN), `scheduleAppointment` (ADMIN/DOCTOR), `recordDiagnosis` (DOCTOR), `prescribeMedication` (DOCTOR), `registerStaff` (ADMIN) all call `requireRole`/`requireAnyRole`. `recordClearanceTest` calls neither. It relies entirely on the menu as its gate.

**Impact.** None today: the menu is a genuine gate and no non-doctor path reaches the service. The exposure is latent — the roadmap exposes `ClinicService` as a REST API, and the moment a second entry point exists, the menu gate no longer applies and clearance recording becomes the only sensitive write with no service-layer protection. Since clearance is what the anti-doping work exists to protect, that inconsistency is worth closing before the API lands.

**Recommendation.** Add `requireRole(UserRole.DOCTOR, "recordClearanceTest")` and the object-level capability check, matching `recordDiagnosis` and `prescribeMedication`, so clearance is gated at the same two levels as every other sensitive operation. Low urgency, near-zero effort — a one-line consistency fix that removes the latent exposure.

---

## 5. Prescribing authority rides on the diagnosis capability `Medium`

**Finding.** The right to prescribe is not modelled as its own capability; it is inferred from the ability to diagnose.

**Evidence (code inspection).** `prescribeMedication` gates on `staff.canDiagnose()` (line 408) — the same predicate `recordDiagnosis` uses. There is no `canPrescribe()`. Any staff type that can diagnose can prescribe, and the two authorities cannot be separated.

**Impact.** Low today (only doctors diagnose), but it couples two distinct clinical authorities. Any future staff type permitted to diagnose but not to prescribe (or vice versa) cannot be expressed, and the model misstates what the permission actually represents.

**Recommendation.** Introduce a distinct `canPrescribe()` capability and gate `prescribeMedication` on it. Low urgency, low effort; worth doing before new staff types are added.

---

## 6. Admin is modelled as a Doctor `Medium`

**Finding.** The administrative account is backed by a clinical `Staff` object, so at the object level it holds clinical capability it should never exercise.

**Evidence (code inspection).** Detailed under Entry 3: `admin@axismed.com` → `STAFF001` → `role=DOCTOR` in `staff.csv`; the `StaffRepository` factory defaults unknown roles to `Doctor`. Admin's `canDiagnose()` returns true; only the session role check stops clinical actions.

**Impact.** A single missed session-level check would let admin perform clinical operations. The model expresses "admin is a doctor," which is not the intended access design, and it blocks a clean Compliance Reviewer modelling (Entry 3) if that role were built by mirroring admin.

**Recommendation.** Model administrative and oversight roles as `UserRole`s without a clinical `Staff` object, or introduce a non-clinical `Staff` subtype whose capability predicates all return false. Resolve alongside the Compliance Reviewer role, since both turn on the same "role without clinical capability" modelling.
