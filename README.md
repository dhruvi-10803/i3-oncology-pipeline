# Oncology Trial Success Rate Pipeline
### i3 Digital Health — Technical Assignment

---

## Problem Statement

Given a raw extract of 1000 oncology clinical trials, design and implement 
a reproducible pipeline that transforms messy, entry-focused data into 
actionable insights on trial success rates.

---

## Repository Structure

```
├── data/
│   └── SampleDateExtract.xlsx        
├── part1a_data_quality.ipynb         
├── part1b_cleaning_schema.ipynb      
├── part2_analysis.ipynb              
├── part3B_written_response.md        
└── README.md                         
```

---

## How to Run

```bash
pip install pandas numpy matplotlib seaborn scikit-learn rapidfuzz openpyxl requests
```

Run notebooks in order:
1. `part1a_data_quality.ipynb`
2. `part1b_cleaning_schema.ipynb`
3. `part2_analysis.ipynb`

Place `SampleDateExtract.xlsx` inside a `data/` folder in the same 
directory as the notebooks before running.

---

## Data Overview

- 1000 interventional oncology clinical trials
- 18 raw columns — trial IDs, phase, recruitment status, dates, 
  enrollment, indications, drugs, technologies, biological targets
- Sourced from ClinicalTrials.gov registry
- Date fields stored as Excel serial numbers — parsed automatically by pandas
- Multi-value fields stored as stringified Python lists — parsed 
  using ast.literal_eval

---

## Methodology

### Part 1A — Data Quality Report

Full profiling of raw data before any transformation. Covers:
- Field completeness and null distribution with severity flags (OK, MINOR, WARNING, CRITICAL)
- Cardinality analysis across all 18 columns
- Dirty values: encoding corruption (102 rows in target abbreviations), 
  free-text synonyms in indications, December 31st date clustering (13.9%)
- Structural anomalies: duplicate ID check, nct_id format validation, 
  list length misalignment (365 rows — confirmed as expected behaviour 
  not data error)
- Distribution plots: phase, recruitment status, enrollment, 
  trials over time, top 15 indications

Key findings:
- No duplicate trial IDs, all 1000 nct_ids follow correct NCT + 8 digit format
- 40 trials have no phase recorded — identified as non-standard trials 
  (surgical procedures, anaesthesia studies, nutritional interventions)
- 365 rows show drug-technology list misalignment — not an error, reflects 
  that supportive care drugs and procedures have no biological targets recorded
- 121 trials have UNKNOWN status — 12% of dataset with no outcome information

---

### Part 1B — Cleaning and Schema Design

- Parsed all 7 list columns from stringified text to real Python lists
- Derived fields: phase_int (decimal), phase_numeric (whole number), 
  trial_type (standard vs non-standard), trial_duration_days, start_year
- Logical validation with 4 checks and flag columns:
  - 1 trial with primary_completion_date after completion_date
  - 18 trials with enrollment value but missing type
  - 29 COMPLETED trials with ESTIMATED enrollment
- Intervention categorisation: each intervention tagged as drug, imaging, 
  surgery, radiation, placebo, diagnostic, transplant, or supportive
- Drug name synonym cleaning: salt forms mapped to base names 
  (vincristine sulfate → Vincristine), typos corrected, placebo variants unified.
  Note: full resolution would require DrugBank or PubChem API integration.
- Indication standardisation using three-layer approach:
  1. Manual mapping dictionary for top cancer types
  2. OncoTree database (Memorial Sloan Kettering Cancer Center) — 897 cancer types
  3. Fuzzy string matching as fallback
- 77.4% of trials mapped to 27 distinct cancer categories
- Output: df_clean.csv — 1000 rows, 29 columns

---

### Part 2 — Success Definition and Analysis

**The core problem:** No success flag exists in the dataset. A defensible 
proxy was constructed from available fields.

**Proxy definition:**

| Label | Criteria |
|---|---|
| Success | COMPLETED + phase >= 2 + enrollment > 0 + has drug + standard trial |
| Failure | TERMINATED |
| Excluded — never started | WITHDRAWN (36 trials) |
| Excluded — right censored | RECRUITING, ACTIVE_NOT_RECRUITING, NOT_YET_RECRUITING, ENROLLING_BY_INVITATION (259 trials) |
| Excluded — ambiguous | UNKNOWN, SUSPENDED (125 trials) |
| Excluded — non-standard | Missing phase or no drug intervention (64 trials) |

**Working dataset: 389 trials — 271 success, 118 failure**

**Overall proxy success rate: 69.7%**

**Critical caveat:** This proxy measures operational trial completion, 
not therapeutic success. A completed trial may have failed its primary 
endpoint entirely. Rates should be interpreted as completion rates for 
trials that reached Phase 2 or above, not as evidence of drug efficacy.

---

### Stratified Success Rates

Success rates computed across four dimensions:

**1. By Phase**
Clear positive relationship — Phase 4 (92.9%) > Phase 3 (78.8%) > 
Phase 2 (75.0%) > Phase 1/2 combo (72.1%). Higher phase trials are 
better resourced and have survived earlier de-risking stages.

**2. By Technology Type**
Antibody (74.5%) outperforms Small Molecule (70.4%). Other Protein 
Therapy leads at 86.7% but only 15 trials — interpret with caution.

**3. By Target Class**
EGFR (82.4%) and TYMS (85.7%) consistently high. Proteasome inhibitors 
(37.5%) and VEGF-A antibodies (42.9%) notably below average — both 
heavily used in difficult refractory settings.

**4. By Cancer Type × Phase**
Lymphoma Phase 2 — 100% (12 trials). Multiple Myeloma Phase 2 — 50% 
despite being one of the most actively researched blood cancers. 
Lung Cancer improves from Phase 2 (56.2%) to Phase 3 (80.0%).

---

### Additional Analyses

**Success trend over time**
Clear declining trend — 1990s: 82.4%, 2000s: 73.2%, 2010s: 67.2%, 
2020s: 62.2%. Likely reflects that well-validated targets were 
addressed first; modern trials tackle harder biology.

**Enrollment distribution by outcome**
Median enrollment for successful trials (57 patients) is 3x that of 
failed trials (18 patients). Strong visual separation between 
distributions confirms enrollment as a key differentiator.

**Most tested drugs**
Vincristine (90%), Prednisone (90%), Etoposide (85.7%) lead. 
Bortezomib (33.3%) and Pemetrexed (45.5%) underperform — both 
used heavily in difficult refractory settings.

**Phase transition analysis**
Phase 4 has highest completion (56%) and lowest termination (4%). 
Phase 1/2 combo trials have lower completion than pure Phase 2 — 
suggesting combo phase trials are more experimental and higher risk.

**Combination vs single drug**
Three or more drug combinations (73.3%) outperform single drugs (70.4%). 
Two-drug combinations underperform at 65.0% — possibly reflecting 
experimental pairings that do not pan out.

**Target × Technology heatmap**
EGFR success is technology-independent — antibody (81.8%) and small 
molecule (83.3%) perform similarly. Target biology matters more than 
delivery mechanism for this class.

---

### Machine Learning — Random Forest Feature Importance

A Random Forest classifier was trained on 375 trials with definitive 
outcomes to identify which features best predict trial success.

**Cross-validated AUC: 0.782**

| Feature | Importance |
|---|---|
| Enrollment | 0.309 |
| Trial duration (days) | 0.196 |
| Cancer category | 0.118 |
| Start year | 0.117 |
| Primary target | 0.108 |
| Phase | 0.107 |
| Technology type | 0.045 |

**Key insight:** Contrary to expectation, phase is not the strongest 
predictor of success. Enrollment size and trial duration dominate — 
suggesting operational factors (funding, patient recruitment, trial 
management) matter more than scientific phase in determining whether 
a trial reaches completion.

---

## Limitations

- Success proxy captures operational completion, not therapeutic efficacy
- 121 UNKNOWN trials excluded — potential selection bias if these 
  systematically differ from trials with known outcomes
- Indication standardisation covers 77.4% of trials; 22.6% remain 
  in broad catch-all categories
- Drug name synonyms partially resolved; full resolution requires 
  DrugBank or PubChem API integration
- Enrollment adequacy thresholds are cancer and phase dependent — 
  a fixed enrollment > 0 threshold is a simplification
- Small strata (n < 10) should be interpreted with caution
- Right-censored ongoing trials excluded from rate calculations

---

## Tools and Libraries

Python 3.x — pandas, numpy, matplotlib, seaborn, scikit-learn, 
rapidfuzz, openpyxl, requests

External database — OncoTree API (Memorial Sloan Kettering Cancer Center)