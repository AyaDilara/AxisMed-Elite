# AxisMed Elite — Anti-Doping Prescribing Safeguard

**A business-analysis case study: identifying a safety gap in a real codebase, specifying the feature that closes it, and carrying it from problem to prioritised, measurable, release-ready specification.**

Built on top of [AxisMed Elite](../README.md), a working Java console system for a sports-medicine clinic. The clinic's whole purpose is issuing doping clearance — yet its prescribing flow never checks what is prescribed against anti-doping rules. A doctor can prescribe a banned substance to a competing athlete and end the eligibility the clinic exists to certify. This case study is the analysis and specification that closes that gap.

Every claim here is grounded in the actual source: the gap, the hook point, and the data model were verified by code inspection, not assumed.

---

## Read in this order

**1. [Gaps & Recommendations](gaps-and-recommendations.md)** — *start here.*
The analysis that found the gap. Six findings from inspecting the codebase, severity-ordered, each with the evidence it rests on — the anti-doping gap first, plus the access-control and data-model issues surfaced along the way. This is the "why this work exists."

**2. [PRD — Anti-Doping Prescribing Check](prd-antidoping.md)** — *the core artifact.*
The product requirements: problem, users, success metrics (a primary outcome plus two guardrails and a counter-metric), scope and explicit non-goals, technical design, risks, and the v1 definition of done. The feature, specified end to end.

**3. [User Stories, Acceptance Criteria & Backlog](user-stories.md)** — *the delivery breakdown.*
The PRD decomposed into stories with testable acceptance criteria (Given/When/Then for the safeguard, rule checklists for reconstructed flows), a definition of ready, and the dependency-ordered build backlog.

**4. [Process Maps](process-maps.md)** — *the change, drawn.*
AS-IS and TO-BE prescribing flows in BPMN, the gap analysis that turns the difference between them into the backlog, and the swimlane/handoff analysis that locates where the safeguard succeeds or fails.

**5. [Prioritization-Method Divergence](prioritization-divergence.md)** — *further analysis.*
Four prioritisation methods applied to the backlog, showing where they agree (robust priorities) and where they diverge (and what each disagreement reveals about the method's bias). A reusable decision sheet for which method to trust for which kind of item.

---

## Technical depth

*The artifacts above are the business-analysis spine. The ones below go into the engineering "how" — included because the case study is meant to stand on both sides of the product/engineering seam, not only the product side. A reader focused on the analysis can stop at the five above.*

- **[Requirements Traceability Matrix](traceability-matrix.md)** — requirement → story → acceptance criterion → verification, in one grid, each requirement marked by whether it's provable now or only after deployment.
- **Architecture Decision Records** — the standing design decisions (gate on a boolean not the sport taxonomy; fail-loud default; service-layer hook; oversight role modelled without clinical capability; static list for v1) as decision records. *(planned)*
- **RFC — Technical Design** — the "how" the PRD defers to: architecture, data model, substance-matching, instrumentation, and list-source integration. *(planned)*
- **Metrics / KPI dashboard** — the PRD's metric system visualised (with clearly-labelled synthetic data, since the system has no live traffic). *(planned)*

---

## Competitive & market context

- **[Competitive analysis](competitive-analysis.md)** — how anti-doping medication checking is handled today (manual, athlete- and physician-facing), and where AxisMed's integrated prescriber-side check is genuinely novel.

---

## What this case study is — and is not

- It is a **specification and analysis** of a feature, grounded in a real codebase. The gap is real and code-verified; the feature is specified, not yet built.
- The safeguard is **verifiable** now (its correctness can be tested deterministically against the full prohibited list) but **not validated** (effectiveness needs a real clinic at a sample size that can detect it). The case study is careful to claim only the first.
- Numbers are not invented. Where a metric would need real traffic or a survey the system does not have, it is specified and the data gap is named, not faked.

---

## About the base system

AxisMed Elite — the system this case study analyses — is a Java 17 console application: a three-layer architecture (UI → service → repository), an abstract `Staff` hierarchy with polymorphic permission checks, role-based access across four roles, and a global patient record. See the [main README](../README.md) for the architecture, design decisions, and how to run it.