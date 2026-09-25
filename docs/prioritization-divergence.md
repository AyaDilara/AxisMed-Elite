# Prioritization-Method Divergence — AxisMed

Four prioritization methods applied to the AxisMed backlog, to show where they **agree** (robust priorities you can trust) and where they **diverge** (rankings that are an artifact of the method, not a real signal). The value is the divergence: when methods disagree, the disagreement localises which method's bias is distorting the ranking.

**This is a qualitative analysis, not a scored one.** AxisMed has 7 seed users and no volume, revenue, or survey data, so no method here produces real numbers — inventing RICE scores or survey results would be fabricated rigor. Each cell states how a method's *logic* classifies the item, and marks `N/A` with a reason where a method genuinely does not apply. Every method is evaluated; only the ones that fit AxisMed's nature are applied, and the rest are left out on the record with their reason. Knowing which method governs which item — and which to leave out — is the analysis, not scoring everything.

---

## The backlog

- **Core safeguard** — A2a (always-prohibited warning) + A3 (fail-loud) + A7 (`subjectToAntiDoping` gate). The feature.
- **A4** — justification / TUE override path (the *need*; met by a manual paper process in v1).
- **A5** — override audit entry.
- **A6** — spike: prohibited-substances data source.
- **Compliance reviewer** — proposed read-only oversight role (gaps finding 3).
- **Finding 4** — service-layer permission gate on clearance recording.
- **Findings 5 & 6** — `canPrescribe` capability; admin-modelled-as-Doctor.

---

## Divergence matrix

| Item | MoSCoW | Kano | RICE | Opportunity scoring | Cost of delay | Verdict |
|---|---|---|---|---|---|---|
| Core safeguard | Must | Must-be (clinic) / Excitement (market) | N/A — safety mandate, not a discretionary bet | Top: max importance, ~0 satisfaction | High risk-cost per week | **Robust-high** |
| A4 (TUE need) | Should → Must (depends on the question asked) | Must-be — blocking legitimate care is catastrophic | N/A — mandate | High — legitimate care denied | High — immediate care harm | **Divergent** |
| A5 (override audit) | Should | Must-be for governance; invisible to the doctor | N/A / low | Medium | Medium | Should (mild) |
| A6 (spike) | Must (unblocks the Musts) | N/A — not a feature | N/A — investigation | N/A | Enabler, sequence first | **Enabler** |
| Compliance reviewer | Should | Indifferent to doctor; important to governance | Low reach | Medium | Medium | **Divergent** |
| Finding 4 (clearance gate) | Could / Won't (v1) | Indifferent | Low | Low | Low now, **rising with the API** | **Divergent** |
| Findings 5 & 6 | Could / Won't | Indifferent | Low impact, low effort | N/A — not user outcomes | Low, flat | **Robust-low** |

---

## Robust items — every applicable method agrees, prioritise with confidence

**Core safeguard → robust-high.** Must (MoSCoW), top opportunity (importance × dissatisfaction), high cost of delay, and either must-be or excitement under Kano depending on frame. RICE abstains — but for a *principled* reason (you don't RICE a safety/compliance mandate), which is agreement, not divergence. When every applicable lens points the same way and the abstaining one abstains on principle, the priority is not a judgment call. Build first.

**Findings 5 & 6 → robust-low.** Low under every method: low RICE (little impact, though cheap), Kano-indifferent (users never see them), flat low cost of delay. No method inflates them. Defer without guilt.

Robust items need no deliberation. The methods are only worth arguing over where they diverge.

---

## Method artifacts — where the ranking depends on the method, and what the divergence reveals

**A4 — MoSCoW says Should, everything else says Must.**
The split traces to *which MoSCoW question you ask*. "Does the feature run without it?" → Should (the check functions). "Is the release safe and legitimate to ship without it?" → Must (without A4 the system hard-blocks medically necessary prohibited substances — a dangerous release). Kano, opportunity scoring, and cost of delay all see the harm and rank it Must. **The divergence exposes MoSCoW's bias: its Must-test defaults to functionality, and functionality is the wrong bar for a safety feature.** Resolution: A4-the-need is a Must, met by a manual paper-TUE process in v1; A4-the-feature is the deferred implementation.

**Compliance reviewer — Kano says indifferent, governance says important.**
Kano rates it low because the *doctor* (the user it measures) never sees it. But its value is separation of duties and being the counter-metric's ground truth — governance value, not user-experience value. **The divergence exposes Kano's bias: it is user-experience-centric and systematically undervalues non-user-facing governance and infrastructure work.** Trust MoSCoW/cost-of-delay here, not Kano.

**Finding 4 — every static method says low; only cost of delay says "do it before the API."**
MoSCoW, Kano, and RICE score it once and rank it low forever (the menu protects clearance today). Cost of delay sees its **accelerating profile**: near-zero cost now, but the day the roadmap's REST API ships, the menu gate no longer applies and this becomes the only unguarded sensitive write. **The divergence exposes the shared blind spot of all static methods: they cannot see an item whose priority is low today but rising.** This is the single clearest case for keeping cost of delay in the toolkit.

**A6 — an enabler, not a ranked item.**
No method scores it, because it is a spike: it exists to make A2a/A3 estimable. It is sequenced first as a precondition, not prioritised against features. Forcing it into a scoring method is a category error.

---

## Reusable sheet — which method to trust, and its bias

| Method | Answers | Blind to | Trust it for | Distrust it on |
|---|---|---|---|---|
| RICE | Value ÷ effort for discretionary bets | Time; mandates; needs volume data | A large discretionary backlog with real usage data | Safety/compliance mandates (abstain); anything without reach data |
| MoSCoW | What is necessary for *this* release | Why; ROI; time | Scoping a release boundary | Its Must-test — "runs" vs "safe to ship" changes the answer |
| Kano | What *kind* of satisfaction a feature gives | Governance/infra value; ROI | Deciding investment style (bulletproof a must-be, don't gold-plate it) | Non-user-facing work — it rates governance/infra as indifferent |
| Opportunity scoring | Which unmet need to pursue (desirability) | Feasibility; viability; landmines | Finding the wedge before choosing a feature | Top scores — they magnetise toward the hardest, most-avoided (landmine) problems |
| Cost of delay | What each week of waiting costs | Absolute value; needs a cost model | Sequencing by urgency; catching rising-cost items | Precision without a real cost model — keep it qualitative when you must |

**The operating rule.** When the methods agree, the priority is robust — act on it. When they diverge, the divergence is not noise: it names which method's bias is distorting the item, and you take the ranking from the method whose lens actually fits the item's nature — safety-viability over MoSCoW functionality, governance value over Kano's user-centricity, rising-cost over static scoring. No single method is trusted alone; the disagreement between them is the most useful signal they produce.