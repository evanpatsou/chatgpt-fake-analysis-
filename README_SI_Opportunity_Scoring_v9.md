# SI Opportunity Scoring — README (v9)

This package contains a stakeholder-friendly notebook that builds an **SI opportunity ranking** for clients.

- **Goal:** identify **IDs with `si_offering = 0`** (not currently in an SI offering) who are **most likely to be interested** in an SI offering.
- **Champion approach (recommended for production):** **Weighted rule scorecard** (constrained, explainable, data-driven).
- **Challenger approach:** **Calibrated Logistic Regression** (run in shadow mode until true labels exist).

**Notebook:** `si_opportunity_scoring_v9_recommended_modeling.ipynb`

---

## 1) What the notebook produces

### A) Stakeholder evidence
On a held-out validation set, it compares three methods:

1. **Fixed rule**: `alpha=0.80` and fixed confirmation weights  
2. **Weighted rule (champion)**: learns confirmation weights + learns `alpha` within bounds  
3. **ML (challenger)**: Calibrated Logistic Regression with MiFID interaction features

It reports:
- **Precision@Top 10% / 20%** and **Lift@Top 10% / 20%** (primary operational metrics)
- **Lift-by-decile curve**
- **Calibration curve**
- AUC / Average Precision / Brier score (supporting metrics)

### B) Operational output
A **Top 20 table** of `si_offering=0` IDs ranked by the champion model (`p_weighted`) including:
- percentile + bucket (`Low / Average / High`)
- “why” columns (contribution breakdown): `whyW_si`, `whyW_sfdr`, `whyW_pai`, `whyW_tax`

---

## 2) Data requirements

### Input file
The notebook expects a CSV called `data.csv` (you can change the path in the notebook).

### Required columns
- `ID`
- `IO_TYPE`, `LIFE_CYCLE`
- `OFFERING_NAME`
- `SI_CONSIDERATION_CD` (S1/S2/S3)
- `MIFID` (Yes/No)
- `SFDR_PREF`, `SFDR_ACTUAL` (F1/F2/F3)
- `PAI_PREF` (e.g., "PAI Selected" or empty)
- `TAXONOMYPREF` (A1/A2/A3)
- ESG topic flags: `GHG`, `Biodiversity`, `Water`, `Waste`, `Social` (Yes/No/--)

> If `data.csv` is not present, the notebook generates a **synthetic demo dataset** so the flow remains runnable.

---

## 3) Business logic implemented

### Branch by MiFID
- If **`MIFID = 0`** → score uses **only** `SI_CONSIDERATION`
- If **`MIFID = 1`** → score blends SI + confirmations:


  **Score (probability) = alpha · SI + (1−alpha) · Confirmations**

### Signals
- **SI anchor:** `si_norm` from `SI_CONSIDERATION_CD`
  - S1 → 0.0, S2 → 0.5, S3 → 1.0
- **SFDR opportunity:**
  - `sfdr_gap = clip(SFDR_PREF - SFDR_ACTUAL, -2, 2)`
  - `sfdr_norm = max(sfdr_gap, 0) / 2` (0..1; only when pref > actual)
- **PAI block:** `pai_block` (0..1)
  - 0 if no PAI, else `0.5 + 0.5*topics_norm`
- **Taxonomy:** `tax_norm` (0..1) from A1/A2/A3 → 0/0.5/1

### Why alpha matters
- High alpha → safer (anchored on explicit SI), but may miss confirmation signal.
- Low alpha → more inference (risk of irrelevant outreach).
The champion model **learns alpha within a conservative bound** (default 0.60–0.90).

---

## 4) Notebook structure (what stakeholders see)

1. **Load + shuffle** (removes ordering artifacts)
2. **Cleaning & filtering (ONLY)**
   - removes `IO_TYPE=zombie`
   - keeps `LIFE_CYCLE=open`
   - standardizes placeholder missing values
3. **Proxy label + ID-level aggregation**
   - derives `si_offering` from `OFFERING_NAME` containing token “SI”
   - aggregates to one row per ID (unit of action)
4. **Feature mappings table**
5. **Encoding & feature engineering**
6. **Worked example (~60%)**
7. **Train/validation split**
8. **Evaluation helpers (precision/lift + calibration)**
9. **Fixed rule**
10. **Weighted rule (champion)**
11. **ML calibrated LR (challenger)**
12. **Validation comparison**
13. **Operational output (Top 20 + buckets + why columns)**
14. **Risks & governance**
15. **Pilot plan**

---

## 5) How to run

1. Place your dataset at `data.csv` (or update `DATA_PATH` in the notebook).
2. Run all cells top-to-bottom.

Recommended environment:
- Python 3.10+
- pandas, numpy, scikit-learn, matplotlib

---

## 6) How to customize safely

### A) Adjust alpha bounds (stakeholder conservatism)
In the weighted rule section:
- `alpha_grid = np.arange(0.60, 0.91, 0.05)`

If stakeholders want SI to dominate, narrow it to e.g. **0.75–0.90**.

### B) Change the KPI used to pick alpha
Currently alpha is chosen by **CV Average Precision** on training data.
You can swap to:
- maximize **Lift@Top 10%**
- maximize **Precision@Top 20%**

(Those are often closer to operational ROI.)

### C) Change bucketing thresholds
Current buckets:
- Low: 0–50th percentile
- Average: 50–80th
- High: 80–100th

Adjust based on outreach capacity (e.g., “we can contact top 15% weekly”).

---

## 7) Risks & governance (what to communicate)

- **Proxy label bias:** `si_offering` is membership, not true interest.
- **Leakage:** never use `OFFERING_NAME` as a feature (the notebook does not).
- **Missing data:** defaults to lowest tier can under-score.
- **Drift:** preferences/process/product naming change.

Controls:
- monitor missingness + score distribution monthly
- monitor top-bucket hit rate monthly
- refit weights + alpha quarterly (or after process changes)
- ML stays in shadow mode unless it remains calibrated and stable

---

## 8) Pilot plan (to make it truly optimal)

Run an experiment to collect **true outcomes** on `si_offering=0`:
- Treatment: top bucket by champion score
- Control: randomized eligible sample
- Outcomes: response, meeting booked, adoption/mandate change, pipeline created

After pilot:
- retrain using true outcomes (not proxy)
- consider **uplift modeling** (incremental ROI), which is the real endgame objective.

---

## Files
- `si_opportunity_scoring_v9_recommended_modeling.ipynb` — main notebook (recommended approach)
