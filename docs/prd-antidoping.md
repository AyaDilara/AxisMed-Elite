# PRD — Anti-Doping Prescribing Check (AxisMed Elite)

**Status:** Proposed

Specifies a check that screens prescribed medications and supplements against anti-doping rules at the point of prescribing. Clinical and anti-doping terms are glossed in parentheses for non-specialist readers.

## Problem

AxisMed Elite serves elite athletes and issues doping clearance (an official ruling that an athlete may compete), yet its prescribing flow records a medication or supplement without checking what is being prescribed. A doctor can prescribe a substance on the WADA Prohibited List (World Anti-Doping Agency — the body defining substances athletes may not use), causing the athlete to fail the very clearance the clinic certifies. The cost is direct: a wrongly prescribed substance can end an athlete's eligibility and expose the clinic to liability. It matters now because anti-doping is the clinic's defining constraint, and the prescribing flow is currently blind to it.

Assumption: the prescribing flow performs no substance-level checking today.

## Users

- Doctor — prescribes medications and supplements; needs to be warned before saving, not after the athlete is compromised.
- Team delegate — certifies the athlete to a team; relies on clearance being trustworthy, and sees clearance status only.
- Compliance reviewer (proposed role) — reviews the audit trail, including prohibited-substance justifications; read-only, prescribes and certifies nothing. Not present in the current four-role system; see below.

## Goals & Success Metric

- Primary: prohibited substances prescribed to an athlete without a recorded justification move to zero, measured per quarter from the audit log.
- Guardrail: false-flag rate (permitted substances wrongly flagged) stays under 5%, so doctors do not begin ignoring warnings.

## Scope

In (v1):
- Screen each prescribed substance against a loaded prohibited-substance list at the point of prescribing.
- Warn on a match, and require a justification before the prescription can be saved.

Cut / Later:
- Allergy, condition, and duplicate-drug checking — ordinary patient safety, a different problem; later.
- Product-name to active-ingredient mapping — v1 matches on a selected substance, not a free-text brand name.
- In-competition timing — some substances are prohibited only around events; v2 can compute this from existing `SportingEvent` dates.
- Automated annual list updates — v1 loads a static, version-stamped list.

## User Stories

1. As a doctor, I want to be warned when a medication or supplement I am prescribing is on the prohibited list, so that the athlete is not later ruled ineligible because of a drug we prescribed.
2. As a doctor, I want to record why I am prescribing a flagged substance, so that the decision is defensible in an anti-doping review and visible to whoever certifies clearance.

## Acceptance Criteria

Story 1:
- Given a substance prohibited at all times, when the doctor prescribes it, then AxisMed shows an interruptive warning naming the substance and category, and the prescription cannot be saved until the doctor removes it or records a justification.
- Given a substance prohibited in-competition only, when the doctor prescribes it, then AxisMed warns, states the restriction is in-competition only, and requires one acknowledgement before saving.
- Given a substance not on the list, when the doctor prescribes it, then no anti-doping warning appears and prescribing proceeds normally.
- Given the loaded list is older than its current annual version, when any prescription is made, then AxisMed flags that the list may be out of date, so a "no warning" result is not mistaken for a guarantee.

Story 2:
- Given a flagged substance, when the doctor proceeds, then AxisMed requires a justification category — for example TUE on file (Therapeutic Use Exemption — official permission for an athlete to use a prohibited drug for a genuine medical need), out-of-competition use, or clinical override — before saving.
- Given a justification is recorded, when the prescription is saved, then substance, category, justification, timestamp, and prescribing doctor are written to the audit log.
- Given a team delegate later views clearance status, when a justified prohibited-substance prescription exists, then the delegate still sees status only, never the justification — the existing privacy boundary holds.

## Technical Design

Data & dependencies:
- A prohibited-substance list stored as `prohibited_substances.csv`, loaded at startup by a new repository following the existing CSV-repository pattern; fields: substance name, category (always / in-competition), list version-year.
- The list is reissued annually by WADA, so its currency is a maintained dependency and needs an assigned owner; without one it silently ages and the check stops being trustworthy.
- Build vs buy: v1 builds a small static list to demonstrate the flow; a production version sources an official machine-readable list — an integration decision deferred to the RFC.

System impact:
- Hook point: `ClinicService.prescribeMedication()` — the check runs after the doctor enters the substance and before the `Medication` is written to the medical record and before `AuditService` logs the action.
- Data model: `Medication` gains an optional justification reference; the justification (category, note, timestamp, doctor) is written through the existing `AuditService`, avoiding a new persistence path.
- The change lives in the service layer plus one new repository, consistent with the existing three-layer design (UI → service → repository); no new architectural layer.
- Consumer: the justification audit entry is intended for a proposed read-only Compliance Reviewer role. It is modelled as a `UserRole` with no `Staff` object — deliberately not mirroring the current admin, which links to a `Doctor` staff record and therefore holds clinical capability an auditor must not have.

Non-functional requirements:
- The check runs synchronously and blocks save until resolved; a warning that can be bypassed by ignoring it is not a control.
- The list loads once at startup, held in memory and keyed for fast lookup.
- If the list file is missing or unreadable, the system warns on every prescription rather than proceeding silently — fail loud, not open.

Technical risks & open questions:
- Substance matching is the hard part: mapping what the doctor entered (a product or brand) to a listed substance (an active ingredient) is non-trivial across brands, spellings, and combination products. v1 sidesteps this with a controlled substance field; robust product-to-ingredient matching is the main question for the RFC.
- In-competition checking depends on the athlete's event dates, held in `SportingEvent`; wiring prescribing to event timing is a v2 design question.
- The external list's format and refresh mechanism are unresolved and belong in the RFC.

Sequencing:
- Spike (precondition for v1): resolve list source, substance-matching approach, and the staleness threshold before A2/A3 estimation. The threshold value is referenced by the fail-loud criterion but supplied by this spike, not by the criterion itself.
- v1 — static version-stamped list, exact substance match, warn + justify + audit.
- v2 — product-to-ingredient mapping; in-competition timing computed from `SportingEvent` dates.
- v3 — maintained or automated list updates from an official source.

## Definition of Done (v1)

v1 of the anti-doping check is done when all of the following hold — not when the code merely runs:

- Acceptance criteria for A2a (always-prohibited warning), A3 (fail-loud when unresolved), A4 (justification required), and A5 (override audited) are verified met.
- The fail-loud path is tested directly: an unavailable or unreadable list blocks the save, not only the happy path.
- An overridden prescription produces an append-only audit entry carrying substance, justification, doctor, and timestamp.
- The guardrail is measured, not merely instrumented: the false-flag rate on a representative substance set is confirmed under 5% before release. If it is not measured, v1 is not done.

This is a feature-level exit gate for v1, distinct from the per-story acceptance criteria: a story can meet its own criteria while the feature as a whole is not yet safe to ship.

## Biggest open risk

Substance matching. If v1 relies on free-text drug names instead of a controlled substance field, the check will both miss real prohibited substances and false-flag permitted ones — failing the primary metric and tripping the guardrail at once. The substance-selection design decides whether v1 works at all.
