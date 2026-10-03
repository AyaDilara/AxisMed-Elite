# Competitive Analysis — Anti-Doping Prescribing Check

Scope: a focused analysis of one question — **is prescriber-side, point-of-prescription anti-doping checking available today, integrated into the prescribing workflow?** This is deliberately not a broad EHR comparison. AxisMed is a focused safeguard, not an enterprise clinical system, so the useful question is not how it compares feature-for-feature against large EHRs, but whether the specific capability it adds — anti-doping checking at the moment of prescribing — exists anywhere today. The analysis stays on that axis and ends in a positioning decision.

---

## The landscape — who addresses this job today

**1. Athlete-facing lookup tools (the substitute that exists).**
Services such as Global DRO and national anti-doping apps let an athlete or support person search a specific medication and see its prohibited status against the current list. These are the real, widely-used tools for the job, and official guidance places the duty to use them squarely on the athlete: athletes must check all medications, prescription and over-the-counter, against the Prohibited List before use ([International Testing Agency — Medications](https://ita.sport/athlete-hub/medications/)).
- *What they do:* answer "is this substance prohibited?" on demand.
- *Where they sit:* separate, manual, **athlete-initiated** — outside any clinical system.
- *The structural gap:* the burden is on the athlete, under strict liability, *around* the prescription — not on the prescribing system *at the moment of prescribing.*

**2. The prescriber's duty is manual (the gap, in official guidance).**
Guidance for health professionals treating elite athletes places the responsibility on the physician to *manually* consult the list: physicians are advised to be aware of the athlete's competition status and to consult the appropriate Prohibited List before prescribing any medication, and to keep their own knowledge of the list current ([Treating the Elite Athlete: Anti-Doping Information for the Health Professional (PMC)](https://pmc.ncbi.nlm.nih.gov/articles/PMC6170056/)). No automated or integrated check is assumed or provided — the safeguard is the physician's memory and diligence.

**3. The manual duty is genuinely hard to discharge.**
Two properties of the list make the manual check error-prone, which is what gives an integrated check its value:
- It changes. The Prohibited List is reissued at least annually (in force each 1 January, published ~3 months ahead), and substances can be added mid-year in exceptional cases ([International Testing Agency — The Prohibited List](https://ita.sport/athlete-hub/the-prohibited-list/)). A physician relying on memory is relying on a moving target.
- It is an *open* list: it names examples but also prohibits substances with similar biological effects, not only those explicitly listed ([PMC, above](https://pmc.ncbi.nlm.nih.gov/articles/PMC6170056/)). Exact-name matching is therefore a lower bound on what a complete check requires — a limitation AxisMed's own design names (substance matching is its biggest open risk).

**4. Direct competitors (integrated prescriber-side anti-doping check).**
None identified in research. The categories above cover the field: anti-doping checking is manual and athlete-facing, and the prescriber's duty is explicitly a manual professional responsibility with no system check assumed. No product was found that screens a prescription against the prohibited list *inside the prescriber's workflow, at the point of prescribing.* This is an absence-of-evidence claim, not proof of absence — it is defensible as "none identified," and is worth re-checking against current sports-medicine software before any public, definitive wording.

---

## Where AxisMed is novel — and where it is not

**Novel (the defensible claim):** moving the check from a **manual, athlete-initiated, out-of-system lookup** — or the physician's unaided memory — to an **automated, prescriber-side safeguard at the point of prescription**: fail-loud, gated on anti-doping status, with a logged justification path. The *integration* is the contribution. The reference data it would integrate against (a maintained prohibited-substance source such as Global DRO) already exists; AxisMed's move is wiring it into the prescribing decision so the check does not depend on the physician remembering to perform it.

**Not novel (what it must not claim):**
- Not a new prohibited-substance database — those exist and are maintained by anti-doping organisations.
- Not better than enterprise EHRs at general clinical decision support — it is a focused prototype; that is not the axis.
- Not validated effectiveness — the novelty is of *approach and placement*, not of proven field outcomes.

---

## The decision

- **Compete on:** integration — anti-doping checking placed inside the prescribing workflow, which no current prescriber system was found to do. This is the wedge and the whole point of the feature.
- **Do not compete on:** database coverage (integrate the existing maintained source), or general EHR breadth (out of scope, and the wrong contest for a focused safeguard).
- **Honest positioning:** "anti-doping medication checking today is a manual duty — on the athlete to look up, and on the physician to remember and consult before prescribing — with no integrated system check. AxisMed's contribution is moving that check into the prescriber's workflow at the point of prescription, where the list's annual changes and open-list nature make unaided manual checking unreliable." Precise, sourced, and not over-claimed.