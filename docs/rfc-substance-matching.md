# RFC-02 — Substance Matching (sub-design of RFC-01)

**Status:** Draft (open for review) · **Author:** Aya-Dilara · **Related:** `rfc-anti-doping-check.md` (RFC-01, the full-feature design this specialises), `prd-antidoping.md`, `architecture-decisions.md` (ADR-005)

The PRD specifies *what* the check does and defers one problem explicitly to this RFC: how an entered medication is matched to the prohibited list. The PRD calls it "the main question for the RFC" and "decides whether v1 works at all." This document scopes only that problem — not the whole feature — because it is the single point on which the check's correctness turns.

---

## 1. The problem, stated precisely

A doctor prescribes a *product* (a brand, a formulation, sometimes a combination). The prohibited list is defined over *substances* (active ingredients and their classes). The check must decide whether the thing prescribed contains or is a prohibited substance. Three properties make this hard:

- **Brand ≠ ingredient.** "Sudafed" is not on the list; pseudoephedrine (its ingredient) is. A name-to-name match misses it.
- **Combination products.** One product can carry several ingredients, any one of which may be prohibited.
- **The list is open.** The WADA Prohibited List bans named examples *and any substance with a similar chemical structure or biological effect* — so an exact-name match against the list is a lower bound on what a complete check requires, not the whole of it. ([International Testing Agency — Prohibited List](https://ita.sport/athlete-hub/the-prohibited-list/); [Treating the Elite Athlete, PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC6170056/))

**The failure mode to avoid:** a matcher that under-matches misses a real prohibited substance (a false clearance — the catastrophic failure); one that over-matches false-flags permitted substances (tripping the <5% guardrail and driving alert fatigue). The PRD's own words: free-text drug-name matching "would both miss real prohibited substances and false-flag permitted ones — failing the primary metric and tripping the guardrail at once."

---

## 2. Design goals and constraints

- **Correctness first:** err toward flagging (a false flag is recoverable via justification; a false clearance is not). This mirrors the fail-loud posture of ADR-002.
- **Deterministic and verifiable:** the matcher must be testable against the full list without users (the RTM's `Deterministic` requirements depend on this).
- **No free-text matching in v1:** the PRD forecloses it. v1 must constrain input so the match is exact and auditable.
- **Fits the existing architecture:** service-layer logic over a CSV-backed repository (ADR-003), no new layer in v1.
- **Honest about scope:** v1 solves the tractable subset; the open-list and combination-product problems are named and deferred, not hand-waved.

---

## 3. Options considered

**Option A — Free-text string match (doctor types a drug name).**
Rejected by the PRD. A typed string matched against list names fails on brands, spellings, and combinations. Both error directions fire. Not viable.

**Option B — Controlled substance vocabulary (doctor selects from a fixed list of active substances).**
The doctor picks the *active substance(s)* from a controlled, searchable set rather than typing a brand. Prescribing is already a structured action; adding a substance field is a modest extension. Matching is then exact against the prohibited list because both sides use the same vocabulary.
- *Strength:* deterministic, verifiable, no NLP, no brand database. Closes the brand-≠-ingredient and spelling problems by construction.
- *Cost:* the doctor must select active ingredients, not a brand — a UX cost (feeds the time-added guardrail), and it assumes the clinic's formulary is expressed in active substances.

**Option C — Product-to-ingredient mapping (doctor picks a brand; system resolves to ingredients).**
A drug database (e.g. a national medicines registry) maps each product to its active ingredients; the system then matches ingredients to the list. This is what a production system eventually needs.
- *Strength:* matches real prescribing behaviour (doctors think in brands); handles combination products.
- *Cost:* requires a licensed, maintained product database and an integration; introduces its own matching and currency problems. This is substantial, and exactly the kind of unresolved integration ADR-005 keeps out of v1.

---

## 4. Decision

**v1 uses Option B — a controlled substance vocabulary.** The doctor selects one or more active substances from a controlled set; the check matches those selections exactly against the prohibited-substance list, which is expressed in the same vocabulary. Free-text entry is not accepted for the checked field.

**v2 adds Option C — product-to-ingredient mapping** — so doctors can prescribe by brand and the system resolves to ingredients, handling combination products. This is sequenced to v2 in the PRD and depends on selecting and integrating a medicines database.

The open-list problem (similar-structure/effect substances) is **not** solved by exact matching in either version and is named as a standing limitation (section 7).

---

## 5. Data model and flow (v1)

- **Prohibited list** — `prohibited_substances.csv`, loaded at startup by a new repository on the existing CSV pattern. Fields: `substance` (canonical active-substance name), `category` (`always` / `in_competition`), `list_version` (year). Keyed in memory for O(1) lookup by canonical substance.
- **Controlled substance set** — the vocabulary the doctor selects from. For exact matching to hold, the prescribing substance field and the list must draw canonical names from the *same* source; reconciling them is a setup task (section 6).
- **Match step** — inside `ClinicService.prescribeMedication()` (ADR-003), after substance selection and before persist: for each selected substance, look up canonical name in the list map; on a hit, raise the flag with category; on miss, proceed. If the list cannot be read, fail loud (ADR-002) rather than returning "no match."

A match is therefore a set-membership test over a shared vocabulary — which is why it is deterministic and exhaustively testable.

---

## 6. The hard sub-problem v1 does not escape: canonicalisation

Option B removes brand and spelling variance *only if* the controlled substance set and the prohibited list agree on canonical names. If the clinic's substance set says "salbutamol" and the list says "albuterol" (the same drug, two names), an exact match misses it. So v1's correctness depends on a **canonicalisation step**: both sides must map to one naming authority (e.g. INN — International Nonproprietary Names), including known synonyms. This is the real work of v1's matcher, and the spike (A6) must resolve which naming authority is used and how synonyms are captured. Exact matching is only as good as the vocabulary it matches over.

---

## 7. Known limitations (named, not solved)

- **Open-list substances.** Exact matching cannot catch a novel substance prohibited by similarity of structure or effect. No deterministic list-match can; this needs expert review or a structural-similarity capability far beyond v1. Stated as a ceiling on what the check guarantees: it catches *listed* substances, not the full intent of an open list.
- **Combination products (v1).** With Option B the doctor must select each active substance; a product selected as a single unit whose components aren't enumerated could hide a prohibited ingredient. Option C (v2) addresses this.
- **Canonicalisation gaps.** A synonym missing from the mapping is a silent miss. Mitigation: err toward flagging on ambiguous names, and treat the synonym set as a maintained dependency like the list itself.

---

## 8. Verification

- **Exhaustive list test (deterministic):** feed every substance in the prohibited list through the matcher; assert each flags with the correct category. This proves no listed substance is missed — the v1 correctness guarantee, runnable with no users.
- **Synonym test:** for each known synonym pair, assert both canonicalise to the same entry and flag.
- **Negative test:** feed a set of known-permitted substances; assert zero flags (feeds the false-flag guardrail's deterministic lower bound).
- **Fail-loud test:** make the list unreadable; assert the matcher withholds rather than returns "no match."

What verification cannot establish pre-deployment: that the controlled vocabulary covers what doctors actually need to prescribe (a formulary-completeness question) — a field concern, consistent with the PRD's verification-vs-validation boundary.

---

## 9. Open questions for the spike (A6)

- Which naming authority is canonical (INN or another), and where does the synonym set come from?
- Is the clinic formulary expressible in active substances (Option B's assumption), or is brand-level entry unavoidable sooner — pulling Option C forward?
- What is the list's machine-readable source for v3, and does it supply canonical names compatible with the chosen authority?