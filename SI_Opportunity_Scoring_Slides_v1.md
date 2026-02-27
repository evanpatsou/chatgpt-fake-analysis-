<!--
Slide-ready Markdown (works well with reveal.js / marp / most markdown-to-slides tools)
Each slide is separated by a horizontal rule: ---
-->

# SI Opportunity Scoring
### Finding clients with **si_offering = 0** who are likely interested in SI
**Champion:** Weighted rule scorecard (constrained & explainable)  
**Challenger:** Calibrated Logistic Regression (shadow mode)  
**Date:** 2026-02-27

---

# The business question
**Who should we contact next about SI offerings?**

We want to **prioritize outreach** to clients who:
- are **not currently** in an SI offering (**si_offering = 0**)
- show **high likelihood of SI interest**

**Success looks like:**
- higher hit rate (responses / meetings / adoption) in the **top bucket** vs baseline

---

# What makes this hard
We do not have perfect “interest” labels yet.

Current benchmark label is a **proxy**:
- `si_offering = 1` if **OFFERING_NAME contains “SI”**

**Implication:**  
This can reflect sales/process decisions, not pure interest → models can overfit the proxy.

**Modeling strategy:**  
Start with constrained, explainable scoring → run a pilot → learn from true outcomes.

---

# Data unit of action
We act on **client IDs**, not row-level records.

**Step:**
- derive row-level `si_offering_row` from OFFERING_NAME
- aggregate to **1 record per ID**
  - `si_offering = max(si_offering_row)` per ID
  - preferences taken by mode / first non-null (transparent rule)

**Output:** a clean, consistent **ID-level** modeling table.

---

# Cleaning rules & impact
**Cleaning is hygiene only (no feature encoding yet).**

We:
- remove `IO_TYPE = zombie`
- keep `LIFE_CYCLE = open`
- normalize placeholders (`"--"`, empty) → missing

**Evidence (shown in notebook):**
- rows removed per rule (bar chart)
- missingness snapshot after cleaning
- proxy label rate shift after filtering

**Decision:** proceed only if filtering doesn’t distort the population unreasonably.

---

# Feature mapping (what each signal means)
**We keep features small & explainable.**

| Business concept | Raw fields | Engineered signal |
|---|---|---|
| Explicit SI interest | SI_CONSIDERATION | `si_norm` (S1=0, S2=0.5, S3=1) |
| MiFID present | MIFID | `MIFID` (Yes=1, else 0) |
| SFDR “current alignment” | SFDR_ACTUAL | `sfdr_actual_norm` (F1=0, F2=0.5, F3=1) |
| SFDR “upgrade opportunity” | SFDR_PREF vs SFDR_ACTUAL | `sfdr_opp_norm = max(pref-actual,0)/2` |
| PAI + topic intensity | PAI_PREF + topics | `pai_block` (0..1) |
| Taxonomy preference | TAXONOMYPREF | `tax_norm` (A1=0, A2=0.5, A3=1) |

---

# Why we changed SFDR
Earlier: `sfdr_norm = max(pref-actual, 0)` captured only **upgrade opportunity**.

**Problem:**  
If `SFDR_ACTUAL = F3`, the client is already aligned with sustainable products, but:
- `pref-actual` may be 0 → the signal disappears

**Fix (v11):**
- keep **alignment**: `sfdr_actual_norm`
- keep **opportunity**: `sfdr_opp_norm`

**Interpretation:**
- Actual = “what they accept today”
- Gap = “where they want to go”

---

# Business logic (the scoring structure)
### Branching by MiFID
- If **MIFID = 0** → score uses **only** SI consideration
- If **MIFID = 1** → score blends:


  **Score = alpha · SI + (1−alpha) · Confirmations**

**Default:** `alpha = 0.80` (SI is the anchor)  
**Better:** learn alpha from data within bounds (e.g. 0.60–0.90)

---

# Worked example (≈60%)
Stakeholder-friendly example under the fixed rule:

- MIFID = 1  
- SI = S2 → `si_norm = 0.5`  
- confirmations strong → `confirm = 1.0`  
- alpha = 0.80  

**Score = 100 × (0.8×0.5 + 0.2×1.0) = 60%**

This makes the score intuitive: “SI matters most; confirmations add evidence.”

---

# Method 1: Fixed rule (baseline)
**Purpose:** establish a transparent benchmark.

- alpha fixed (e.g., 0.80)
- confirmation weights fixed (stakeholder-approved)

Confirmations include:
- SFDR alignment (`sfdr_actual_norm`)
- SFDR opportunity (`sfdr_opp_norm`)
- PAI intensity (`pai_block`)
- Taxonomy (`tax_norm`)

**What stakeholders get:**
- clear formula
- “why” contributions per client

---

# Method 2: Weighted rule (recommended champion)
**Purpose:** keep the same scorecard structure, but learn what matters from data.

We learn from training data:
1) confirmation weights (SFDR / PAI / Tax / missing flags)
2) alpha split (bounded) to balance SI vs confirmations

**Constraints (governable):**
- weights constrained to be non-negative (“supportive evidence”)
- weights normalized to sum to 1
- alpha searched only within conservative range (e.g., 0.60–0.90)

**Why it works:** explainable like a rule, but empirically justified.

---

# Method 3: ML challenger (Calibrated Logistic Regression)
**Purpose:** controlled ML uplift without losing explainability.

- use **Calibrated Logistic Regression** (isotonic calibration)
- add MiFID interaction features so confirmations contribute mainly when **MIFID=1**

**Why LR (not kNN/SVM):**
- interpretability (coefficients)
- robust tabular baseline
- calibrated probabilities for buckets/thresholds

**Deployment recommendation:** shadow mode until pilot labels exist.

---

# Evaluation: what “good” means operationally
We prioritize **top-bucket ROI**, so we emphasize:

- **Precision@Top 10% / 20%**
- **Lift@Top 10% / 20%** (top rate ÷ baseline rate)
- **Lift-by-decile curve** (should increase toward top decile)
- **Calibration curve** (predicted probability matches observed)

Supporting metrics:
- AUC, Average Precision (AP), Brier score

---

# Validation comparison (Fixed vs Weighted vs ML)
**One held-out validation split** for all methods.

**What we look for:**
- Weighted rule improves lift vs fixed rule
- ML improves further **only if** lift improves materially and calibration stays good

**Slide visual (from notebook):**
- table of AUC / AP / Brier
- bar charts of metrics
- lift curves for each method

**Decision:** select champion based on top-bucket lift + stability.

---

# Operational output: the outreach list
Deliverable: Top ranked **si_offering = 0** IDs with explanations.

For each ID we output:
- predicted score / probability
- percentile + bucket (Low / Average / High)
- “why columns” (component contributions), e.g.
  - why_SI, why_SFDR_alignment, why_SFDR_opportunity, why_PAI, why_Taxonomy

**Why it matters:**  
Front office can explain *why* an ID is prioritized → improves adoption.

---

# Governance & pitfalls
**Key pitfalls**
- Proxy label bias (membership ≠ interest)
- Missingness and encoding mismatches can hide signal
- Drift: questionnaire completion/process/product naming changes
- Leakage: never use OFFERING_NAME as a feature (only for proxy label creation)

**Controls**
- monthly monitoring: missingness + score distributions + top-bucket hit rate
- quarterly refit for weights/alpha (or after process change)
- human review for top-ranked edge cases

---

# Pilot plan (to create true labels)
To make the model truly optimal, run a controlled pilot:

- Population: `si_offering = 0`
- Treatment: top bucket by champion score
- Control: randomized eligible sample
- Track outcomes:
  - response
  - meeting booked
  - adoption / mandate change
  - pipeline created

**Then:** retrain on true outcomes (and consider uplift modeling).

---

# Recommendation & next steps
**Ship now:**
- Weighted rule scorecard as **champion**
- Calibrated LR as **challenger** (shadow)

**Next (4–8 weeks depending on outreach cycle):**
- run pilot → collect true labels
- re-evaluate models on true outcomes
- lock operational thresholds & monitoring

**Outcome:** a scoring system that is explainable, measurable, and improvable.
