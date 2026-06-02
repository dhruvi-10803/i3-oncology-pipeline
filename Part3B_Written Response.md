# Part 3B — Written Response

---

## Results Summary

The pipeline analysed 389 oncology trials with definitive outcomes 
(271 successful, 118 failed) yielding an overall proxy success rate 
of 69.7%. Key findings:

- Higher phase trials complete at higher rates — Phase 4 (92.9%) 
  consistently outperforms Phase 2 (75.0%), reflecting that later-phase 
  trials are better resourced and have survived earlier de-risking stages.

- Enrollment is the strongest predictor of success — median enrollment 
  for successful trials (57 patients) is 3x that of failed trials 
  (18 patients). A Random Forest classifier (AUC 0.782) confirmed 
  enrollment as the top feature (importance: 0.309), ahead of phase 
  (0.107). Operational factors matter more than scientific phase.

- Success rates are declining over time — 1990s trials achieved 82.4% 
  vs 62.2% for 2020s trials, suggesting earlier research addressed 
  well-validated targets while modern trials tackle harder biology.

- Target class drives meaningful differences — EGFR (82.4%) and TYMS 
  (85.7%) consistently outperform Proteasome inhibitors (37.5%) and 
  VEGF-A antibodies (42.9%).

- Lymphoma Phase 2 trials show 100% completion across 12 trials while 
  Multiple Myeloma Phase 2 trials show only 50% — despite both being 
  heavily researched blood cancers with distinct biology and treatment 
  landscapes.

---

## Limitations of the Current Success Metric

The proxy defines success as trial completion at Phase 2 or above with 
confirmed enrollment. This measures operational completion, not 
therapeutic success. A trial can complete while its primary endpoint 
fails entirely — making a completed but efficacy-failed trial 
indistinguishable from a genuinely successful one.

Several biological factors make this proxy imprecise:

**Enrollment thresholds are cancer-dependent**
The enrollment > 0 threshold was chosen as a minimal sanity check. 
Biologically appropriate thresholds vary significantly by cancer type. 
A rare cancer like Peritoneal Mesothelioma or Uveal Melanoma may have 
only 500-2000 new cases globally per year — a trial with 15 patients 
represents a meaningful fraction of the available population and may 
be well-powered. The same enrollment for a common cancer like breast 
cancer or lung cancer would be statistically meaningless. A production 
pipeline would incorporate disease-specific power calculations based 
on incidence rates and expected effect sizes.

**Trial duration adequacy is also cancer-dependent**
Fast-moving cancers like Acute Myeloid Leukemia or Glioblastoma allow 
meaningful survival endpoints to be captured within 12-18 months. 
Slow-moving cancers like Prostate Cancer or Follicular Lymphoma require 
5-10 years of follow-up to observe overall survival differences. A short 
completed trial for AML may be perfectly adequate; the same duration 
for prostate cancer would be scientifically insufficient. Trial duration 
without cancer context is therefore a weak signal.

**Phase labels are inconsistent across trial designs**
In oncology, Phase 1 trials often use cancer patients who have exhausted 
other options and frequently report early efficacy signals. Some landmark 
approvals — including early pembrolizumab indications — were based partly 
on Phase 1 data. Excluding all Phase 1 completions loses real signal. 
Additionally, basket trials (one drug, multiple cancer types) and adaptive 
designs blur phase boundaries further, making phase-based exclusions 
imprecise.

**Line of therapy confounds comparisons**
First-line trials in treatment-naive patients have higher expected response 
rates than third-line trials in heavily pre-treated populations. Comparing 
success rates across indications without controlling for line of therapy 
inflates apparent differences between cancer types.

---

## Additional Data Needed

To redefine success closer to therapeutic value, the following data 
would be most impactful:

**Primary endpoint outcome**
A binary flag or numeric result for the prespecified primary endpoint — 
objective response rate, progression-free survival, or overall survival. 
This is the single most important missing field. ClinicalTrials.gov 
mandates result submission for completed trials; linking these records 
would immediately upgrade the proxy from operational to scientific success.

**Reason for termination**
TERMINATED trials are currently all labelled failures. Termination 
reasons vary significantly — lack of efficacy, safety signal, funding 
withdrawal, company merger, or investigator decision. A safety 
termination is a harder failure than a funding lapse. This field would 
enable finer-grained outcome labelling.

**Regulatory outcome**
FDA or EMA approval status, Breakthrough Therapy designation, or 
NDA/BLA submission status. These are publicly available via FDA 
databases and would allow a hard regulatory success label that directly 
reflects clinical and commercial value.

**Line of therapy**
Whether a drug is tested as first-line, second-line, or later treatment 
significantly affects expected success rates and is currently absent 
from the dataset.

**Cancer-specific enrollment benchmarks**
Statistical power calculations per indication and phase, drawing on 
FDA guidance documents, would replace the current blanket enrollment > 0 
threshold with biologically defensible minimums.

---

## How the Schema Would Evolve

The current schema is a single flat analytical table with 29 columns. 
With richer data it would normalise into related tables:

**trials** — core table retaining identifiers, phase, dates, enrollment, 
and derived fields. Adds: primary_endpoint_met, termination_reason, 
regulatory_outcome, line_of_therapy, result_submission_date.

**trial_results** — one row per result submission, linked via nct_id. 
Contains endpoint_type, endpoint_value, result_date, publication_doi, 
positive_result_flag. Handles multiple reported endpoints per trial.

**interventions** — one row per drug-trial pair. Contains drug_id, 
drug_name_canonical, technology_class, is_primary_intervention. 
Eliminates string parsing and distinguishes investigational drugs 
from supportive care medications.

**indications** — one row per indication-trial pair. Contains 
indication_raw, oncotree_code, mesh_id, broad_category. Using 
OncoTree or MeSH codes as canonical identifiers resolves the 
free-text synonym problem — the 437 unique raw indication values 
would reduce to a controlled standardised set. The current pipeline 
used the OncoTree API from Memorial Sloan Kettering Cancer Center 
for partial standardisation; a production version would query the 
NCI Thesaurus API — the same vocabulary used by ClinicalTrials.gov 
itself — at ingestion for complete coverage.

**drugs** — lookup table keyed on drug ID. Contains drug_name_canonical, 
drugbank_id, pubchem_cid, mechanism_class, approval_status. Integration 
with the DrugBank API or PubChem API would resolve salt form synonyms 
automatically at ingestion — eliminating the manual cleaning currently 
required for entries like vincristine sulfate → Vincristine.

**targets** — lookup table containing target_abbreviation, gene_symbol, 
uniprot_id, target_class. Integration with UniProt or ChEMBL would 
standardise the encoding-corrupted target abbreviations present in 
the current dataset and enable proper target class grouping.

---

## What a Production Pipeline Would Do Differently

**Real-time status updates**
The 121 UNKNOWN trials and 29 COMPLETED trials with ESTIMATED enrollment 
represent stale registry data. A production pipeline would schedule 
periodic ClinicalTrials.gov API pulls to refresh trial status, enrollment 
actuals, and result submissions automatically.

**Survival analysis for right-censored trials**
259 ongoing trials were excluded from success rate calculations. A 
production pipeline would implement Kaplan-Meier survival analysis 
to properly incorporate these right-censored observations — treating 
ongoing trials as censored rather than excluded. This recovers 259 
currently excluded trials and produces completion probability estimates 
with confidence intervals, stratified by phase, cancer type, and 
technology class.

**Vocabulary standardisation via APIs**
Indication standardisation currently uses a manually curated dictionary 
validated against OncoTree. A production pipeline would query the NCI 
Thesaurus API at ingestion to map every incoming indication to a 
standardised code automatically. Drug name resolution would use the 
DrugBank or PubChem API to canonicalise every drug name at ingestion. 
Biological target standardisation would use UniProt or ChEMBL APIs.

**Success metric versioning**
As richer data becomes available the success definition will evolve. 
A production pipeline would version the success metric explicitly — 
allowing historical comparisons across definition versions and 
transparent audit trails of how the metric changed over time.