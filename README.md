# AxisMed Elite
> A console-based Java system for an international sports-medicine clinic — managing patients, staff, appointments, diagnoses, treatment plans, and team clearances across six cities, with role-based access and a global patient record.

**Stack:** Java 17 · CSV persistence · **Status:** Functional prototype (console)

## Problem

Most introductory OOP projects choose a deliberately simple domain — a library, a to-do list — because the goal is to practise syntax, not to stress-test a design. AxisMed Elite takes the opposite approach: a domain complicated enough that the object model has to be load-bearing rather than decorative.

The clinic is fictional, but its constraints are taken seriously: doctors split across distinct specialties; staff with non-overlapping permissions; patients ranging from local chronic-condition cases to athletes flying in for a single pre-match clearance; and a privacy boundary where a team delegate can confirm an athlete is cleared to play without ever seeing why a teammate was not.

The architecture is a consequence of those constraints, not an imposition on a simple problem. An abstract `Staff` type, polymorphic permission checks, and a record that travels with the patient across locations are what the domain demands once it is complicated enough to be realistic.

## Overview

The clinic operates across Bucharest, London, Dubai, Istanbul, Madrid, and Doha, serving elite athletes, visiting teams, and chronic-condition patients. Patient records are global — one unified record per patient, accessible from any location. Four roles each see a different, permission-enforced slice of the system.

| Role | Capabilities |
| --- | --- |
| `ADMIN` | Full access — staff, patients, clinics, sporting events |
| `DOCTOR` | Diagnoses, treatment plans, prescriptions, functional and clearance tests |
| `PATIENT` | Own appointments, record, diagnoses, treatment plans, prescriptions, results |
| `TEAM_DELEGATE` | Registers athletes, links them to events, views clearance status only |

Clinical operations: patient registration by type (local, visiting individual, visiting team, chronic); appointment scheduling with conflict detection; diagnosis recording; treatment plans with rehabilitation protocols; functional tests (VO2 max, FMS, ECG, blood panels); medical and doping clearance with CLEARED / NOT_CLEARED results; prescribing; global medical record; performance reports; sporting-event management.

## Method

Three-layer architecture with dependency injection at the composition root.

```
ConsoleMenu (UI)  →  ClinicService (business logic)  →  Repositories (CSV persistence)
```

The UI layer never touches the data layer directly; repositories are injected into `ClinicService` via constructor. The model is 18 classes across six packages, built on an abstract `Staff` hierarchy and three behavioural interfaces (`Diagnosable`, `Schedulable`, `Recordable`).

Design decisions worth naming:

- **Polymorphic permissions over `instanceof`.** `ClinicService` never checks `if (staff instanceof Doctor)`. It calls `staff.canDiagnose()` — each subclass returns its own answer. Adding a staff type requires no change to the service layer.
- **Abstract class for `Staff`, not just an interface.** `Staff` must both define a contract (`canDiagnose()`, `getTitle()`) and hold shared concrete state and logic (scheduling, `getFullName()`, clinic assignment). An interface alone cannot carry instance fields or a constructor.
- **Collections chosen for access pattern.** `HashMap` for O(1) repository lookups by ID; `TreeMap` for medical-record history so appointments are stored and retrieved in chronological order without manual sorting; `LinkedList` for sequentially-appended lists; `ArrayList` where index access dominates.
- **Double permission check for sensitive actions.** Recording a diagnosis validates both the session role (`UserRole`) and the object capability (`staff.canDiagnose()`) — enforcement at session and object level independently.
- **Privacy rule lives in the service layer.** `TeamDelegate` holds no restriction logic itself; the "clearance status only" boundary is enforced in `ClinicService.viewFullPatientHistory()`. Business rules belong in the service layer, not the model.

Supporting mechanics: 13 enums remove invalid states at compile time; six checked exceptions carry domain failures through the service layer; generic singleton `CSVReaderService<T>` / `CSVWriterService<T>` handle all file I/O with RFC 4180 escaping; an `AuditService` logs every action with actor and timestamp before returning.

## Results

Representative console output.

```
=== AXISMED ELITE | DOCTOR ===
Select: 3

-- Record Diagnosis --
Appointment ID : APT001
Diagnosis name : ACL Grade II Sprain
ICD code       : S83.5
Diagnosis recorded. ID: DGN-9E1AE979C079
```

```
   AXISMED ELITE — PERFORMANCE REPORT
Patient : Marcus Johnson   ID: PAT001   Age: 36   Sport: FOOTBALL
--- Diagnoses (2) ---
  [2026-06-10] Medial Meniscus Tear (M23.2)
--- Treatment Plans (2) ---
  [PLN001] Status: ACTIVE | Recommendations: Avoid high-impact activities; ice 2x daily
--- Functional Tests (1) ---
  [2026-06-12] NUMERIC: 54.2 ml/kg/min
--- Medications (1) ---
  Ibuprofen — 400mg, 3x daily [pharmaceutical]
```

```
-- Audit Log --
  LOGIN                  2026-06-12T18:46:01   patient@axismed.com
  RECORD_DIAGNOSIS       2026-06-12T19:07:03   doctor@axismed.com
  PRESCRIBE_MEDICATION   2026-06-12T19:10:54   doctor@axismed.com
  RECORD_CLEARANCE_TEST  2026-06-12T19:12:39   doctor@axismed.com
```

## Reproduce

```bash
git clone https://github.com/ayadilara10/AxisMed-Elite.git
cd AxisMed-Elite
javac -d out -sourcepath src src/axismed/Main.java
java -cp out axismed.Main
```
Requires Java 17+, no external dependencies. Test accounts load at startup from `data/users.csv`:

| Email | Password | Role |
| --- | --- | --- |
| admin@axismed.com | admin123 | ADMIN |
| doctor@axismed.com | doctor123 | DOCTOR |
| patient@axismed.com | patient123 | PATIENT |
| delegate@axismed.com | delegate123 | TEAM_DELEGATE |

## Known Limitation

The prescribing flow and the doping-clearance flow do not cross-check each other. A doctor can prescribe a medication or supplement to an athlete under active clearance without the system verifying that the substance is not on a prohibited-substance list (e.g. WADA). In a real sports-medicine setting this is a safety gap: the absence of a warning is not the same as the substance being permitted.

This is called out deliberately rather than hidden — identifying it is part of the design analysis, and closing it is the first roadmap item.

The specification for closing this gap lives in [`docs/prd-antidoping.md`](docs/prd-antidoping.md): a PRD scoping an anti-doping prescribing check — problem, users, acceptance criteria, and technical design. The scope call is the point: v1 ships an exact substance match against a version-stamped prohibited list, warning and requiring a logged justification before a flagged prescription can be saved. Brand-name-to-ingredient matching and in-competition timing are deferred to v2 — narrowing v1 to what can be built correctly, because a matching layer that mis-flags substances would fail the check's own metric. The check hooks into `ClinicService.prescribeMedication()` and fails loud rather than open: unresolved substance status blocks the save.

## Roadmap

Each item states the outcome it unlocks, not just the change.

**Now — Prescriptions are safe by default for athletes.** No athlete under active clearance can be prescribed a prohibited substance without an explicit, logged override.
- Add a prohibited-substance check between the prescribing flow and the athlete's clearance status.
- Fail-safe: when substance status cannot be resolved, block and require confirmation rather than allowing silently.
**Now — Prescribing and clearance decisions are independently auditable.** Someone who neither prescribes nor certifies can review what was overridden and why.
- Add a read-only Compliance Reviewer role that consumes the audit log, including the prohibited-substance justifications the anti-doping check records.
- Separation of duties: the reviewer holds no clinical capability and cannot alter records.

**Next — Changes can be made without silent regressions.** The service layer can evolve with confidence.
- JUnit suite covering happy paths and every exception path in `ClinicService`, replacing manual console testing.

**Later — The clinic logic is reachable beyond the console.** Non-technical staff and other frontends can use the system.
- Migrate CSV persistence to a relational database (relationships maintained by foreign keys, not hand-wired at startup).
- Expose `ClinicService` as a REST API — the three-layer architecture was designed for exactly this migration.

## Structure

```
AxisMed-Elite/
├── src/axismed/
│   ├── Main.java              composition root — assembles dependencies
│   ├── enums/                 13 enums — remove invalid states at compile time
│   ├── exception/             6 checked domain exceptions
│   ├── model/
│   │   ├── staff/             Staff (abstract) + Doctor/Physio/Nutritionist/Trainer
│   │   ├── patient/           Patient, TeamDelegate
│   │   ├── appointment/       Appointment, ClearanceTest
│   │   ├── medical/           MedicalRecord, Diagnosis, TreatmentPlan, ...
│   │   ├── event/             SportingEvent
│   │   └── clinic/            Clinic
│   ├── service/               ClinicService, AuthService, AuditService
│   ├── repository/            generic CSV services + per-entity repositories
│   └── util/                  ConsoleMenu, DateUtils
├── data/                      CSV store (patients, staff, appointments, ..., audit_log)
└── README.md
```

## Conventions

- Docstrings: Javadoc on public service and repository methods (`@param`, `@return`, `@throws`).
- Comments: intent-comments on non-obvious logic (why, not what).
- Enums over strings for any field with a fixed value set.
- Business rules in the service layer; models hold state, not policy.
