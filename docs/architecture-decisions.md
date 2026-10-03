# Architecture Decision Records — Anti-Doping Prescribing Check

Standing design decisions for the anti-doping safeguard, each as a self-contained record: the context that forced the decision, the decision itself, the alternatives rejected, and the consequences accepted. These are the decisions a future engineer must not silently reverse without understanding why they were made.

Format follows Nygard-style ADRs (Context / Decision / Consequences). Consolidated into one file because the set is fixed and read together; in a long-lived project these would be separate append-only records.

---

## ADR-001 — Gate the check on a dedicated `subjectToAntiDoping` flag, not the sport taxonomy

**Status:** Accepted (v1)

**Context.** The check must run for athletes subject to anti-doping rules and not for other patients (hobbyists, chronic-care, retired), or it over-flags and trips its own false-flag guardrail on volume. The data model already carries `primarySport`, `patientType`, and `membershipType` — any of which could seem to answer "is this an athlete?" But none does reliably: `primarySport` records that someone does a sport, not that they compete under anti-doping rules — a weekend swimmer and a retired professional both carry one.

**Decision.** Add a dedicated boolean `subjectToAntiDoping`, set at registration, and gate the check solely on it. Do not infer regulated status from `primarySport`, `patientType`, or `membershipType`.

**Alternatives rejected.** Gating on `primarySport != NONE` — over-includes hobbyists and the retired. Gating on `patientType` — those categories describe why the patient is at the clinic, not their anti-doping status. Both couple a safety-critical gate to a taxonomy built for another purpose.

**Consequences.** A new field must be captured at registration and kept current (a maintained dependency; stale at scale without progressive re-capture). In exchange, the safeguard never depends on classifying sports correctly — correctness of the gate is independent of correctness of the sport data. The accepted error direction is over-inclusion (flag a retired athlete) over under-inclusion (miss an active one), since missing a regulated athlete is the catastrophic failure.

---

## ADR-002 — Fail loud: an unresolvable check withholds the prescription, never silently clears it

**Status:** Accepted (v1)

**Context.** The prohibited-substance list can be unavailable (missing, unreadable, or stale past its version). The system must do something defined when it cannot verify a substance. The danger is specific: a silent system does not read as "unverified" — the absence of a warning reads as "cleared." A fail-open design would let a prohibited substance through while the doctor believes the system vetted it — worse than no system, because it manufactures false confidence.

**Decision.** When substance status cannot be resolved, block the save and warn, stating the cause, that this is not a clearance, and the manual action required. Never proceed silently.

**Alternatives rejected.** Fail open (proceed when the check can't run) — converts an outage into a silent false clearance. Warn-but-allow-bypass — a warning that can be ignored is not a control.

**Consequences.** During a list outage, prescribing for regulated patients is blocked until acknowledged — an operational cost accepted deliberately, because the alternative is an undetected violation. A sustained outage is handled as an incident (notify, brief doctors, suspend-with-oversight), distinct from the per-attempt block.

---

## ADR-003 — Hook the check into the service layer at `prescribeMedication()`, in-process for v1

**Status:** Accepted (v1)

**Context.** The check needs the patient (for the gate), the substance, the list, and the audit trail at one point. The only location with access to all of these is the prescribing path in the service layer. The system is a three-layer architecture (UI → service → repository); business rules already live in the service layer by convention.

**Decision.** Run the check inside `ClinicService.prescribeMedication()`, after the substance is entered and before the `Medication` is written and audited. Load the list via a new repository following the existing CSV-repository pattern. No new architectural layer.

**Alternatives rejected.** Validation inside the `Medication` model — a value object cannot reach the list or the patient's status, and business rules do not belong in the model by project convention. A check in the UI/menu layer — would not be enforced if the service were ever called from another entry point (the planned REST API).

**Consequences.** The check is enforced wherever the service is called, surviving the future API migration. It adds one synchronous step to the prescribing path (the UX time guardrail governs this). In v1 the list is in-process; v3's external source will make the list a separate participant — a known future change, not a v1 concern.

---

## ADR-004 — Model the Compliance Reviewer as a role without clinical capability, not by mirroring admin

**Status:** Accepted (proposed role)

**Context.** The override audit trail needs an independent, read-only consumer (separation of duties). A natural shortcut is to model the new role like the existing admin. But admin is not a clean template: `admin@axismed.com` links to a `Staff` record stored as `role=DOCTOR`, and the `StaffRepository` factory defaults unrecognised roles to `Doctor` — so admin's object returns `canDiagnose() == true` and is held back from clinical actions only by the session-level role check, not by its object.

**Decision.** Model the Compliance Reviewer as a `UserRole` with no clinical `Staff` object — read-only, holding no clinical capability and unable to alter records. Do not mirror admin's structure.

**Alternatives rejected.** Mirroring admin — would give the auditor a `Doctor` stub with latent clinical capability, exactly the property an auditor must not have, and would inherit admin's own modelling flaw (ADR-005).

**Consequences.** A new `UserRole` and a read-only menu/query path are required. The auditor cannot, even by a missed check, perform clinical actions, because its object holds no clinical capability. This role is also the ground truth for the illegitimate-override counter-metric, so it is load-bearing for both accountability and measurement.

---

## ADR-005 — v1 loads a static, version-stamped list; external sourcing is deferred

**Status:** Accepted (v1), revisited at v3

**Context.** A production check should source an official, maintained prohibited-substance list, and the list changes at least annually. But robust product-name-to-active-ingredient matching and the external integration (format, refresh, availability) are substantial unknowns — the feature's biggest open risk. Shipping them in v1 would couple the core safeguard to the hardest unsolved problems.

**Decision.** v1 loads a small, static, version-stamped list from CSV and matches on a controlled substance field (not free-text brand names). External sourcing and automated updates are deferred to v3; product-to-ingredient matching and in-competition timing to v2.

**Alternatives rejected.** Building the external integration in v1 — blocks the safeguard on an unresolved integration and a hard matching problem. Free-text drug-name matching in v1 — would both miss real prohibited substances and false-flag permitted ones, failing the primary metric and tripping the guardrail at once.

**Consequences.** v1 demonstrates the full flow correctly on a controlled vocabulary, and its list currency is a manual maintained dependency needing an assigned owner until automation lands. The version stamp makes staleness detectable (ADR-002's fail-loud staleness check depends on it). The deferral is explicit and sequenced, not an omission.