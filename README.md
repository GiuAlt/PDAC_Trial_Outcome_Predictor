# PDAC Trial Outcome Predictor

A machine learning project to extract **mechanistic insights** from pancreatic ductal adenocarcinoma (PDAC) clinical trial data. Using publicly available trial metadata, drug mechanisms, eligibility criteria, and outcome labels, this project identifies which biological and trial design features are associated with clinical trial success or failure in one of oncology's most challenging diseases.

---

## Biological motivation

PDAC has a 5-year survival rate below 12% and remains largely resistant to modern targeted therapies and immunotherapy. Despite decades of clinical trials, only a handful of treatment strategies have demonstrated meaningful survival benefit. This project asks: **can we learn from the pattern of failures?**

Rather than building a clinical prediction tool, the goal is mechanistic interpretation — using interpretable machine learning to surface which features of trial design and drug biology consistently distinguish successful from failed trials.

---

## Data source

All clinical trial data comes from the **AACT database** (Aggregate Analysis of ClinicalTrials.gov), maintained by Duke University's CTTI. AACT provides a clean, relational version of ClinicalTrials.gov as flat text files.

- Registration: [aact.ctti-clinicaltrials.org](https://aact.ctti-clinicaltrials.org)
- Export used: `20260413_export_ctgov.zip` (pipe-delimited flat files)
- The raw AACT files are not included in this repository (2.24 GB)

Publication abstracts were retrieved via the **PubMed Entrez API** (Biopython).

---

## Project structure

```
PDAC_Trial_Outcome_Predictor/
│
├── notebooks/
│   ├── 01_Read_Data.ipynb          # Data loading, PDAC filtering, exploration
│   ├── 02_Feature_Eng.ipynb        # Feature engineering (all phases)
│   └── 03_Modelling.ipynb          # Modeling, SHAP, UMAP
│
├── outputs/
│   ├── pdac_master_labeled_v3.csv  # 107 labeled PDAC trials
│   ├── pdac_features_X.csv         # Final feature matrix (107 × 63)
│   ├── pdac_labels_y.csv           # Binary outcome labels
│   ├── pdac_abstracts_labeled.csv  # Rule-based labeled abstracts
│   ├── pdac_trial_abstracts.csv    # Raw PubMed abstracts
│   ├── shap_summary.png            # SHAP beeswarm plot
│   ├── shap_importance_bar.png     # SHAP mean importance bar chart
│   ├── umap_outcome.png            # UMAP colored by outcome
│   └── umap_moa.png                # UMAP colored by mechanism of action
│
├── Data/                           # Raw AACT files — not tracked by git
├── .gitignore
├── requirements.txt
└── README.md
```

---

## Methods

### 1. Trial selection
Starting from the full AACT database (~500k trials), PDAC trials were identified via condition name filtering. After restricting to interventional Phase 2/3 trials with completed or terminated status, **1090 candidate trials** were identified.

### 2. Outcome labeling
Labels were assigned from three independent sources:

| Source | Method | Trials labeled |
|---|---|---|
| PubMed abstracts | Rule-based NLP + local LLM (Mistral via Ollama) | 83 |
| CT.gov `why_stopped` field | Rule-based classification of termination reason | 24 |
| **Total** | | **107 trials** |

Binary label: `1 = SUCCESS` (primary endpoint met), `0 = FAILURE` (primary endpoint not met or terminated for futility).

### 3. Feature engineering
Features were engineered across four categories:

- **Trial design**: phase, enrollment size (log-transformed), duration, trial era, sponsor type
- **Intervention**: number of drugs, combination flag, biological/radiation components
- **Mechanism of action**: 35 binary MoA categories mapped from drug names (DrugBank + manual annotation)
- **Eligibility criteria**: NLP extraction of disease stage, line of therapy, biomarker requirements, performance status thresholds

Final feature matrix: **107 trials × 63 features**

### 4. Modeling
Two models were evaluated using 5-fold stratified cross-validation:

| Model | Mean AUC | Std |
|---|---|---|
| Logistic Regression | 0.612 | ±0.100 |
| Random Forest | 0.653 | ±0.080 |

The Random Forest was selected for interpretation. SHAP (SHapley Additive exPlanations) values were computed to identify which features drive predictions. UMAP was used to visualize the trial feature space in 2D.

---

## Key findings

**Top predictive features (SHAP):**

1. `duration_days` — Longer trials associate with success; short trials often stopped early for failure
2. `elig_second_line` — Second-line trials consistently fail; first-line trials have better outcomes
3. `enrollment` — Larger, well-powered trials more likely to succeed
4. `elig_adjuvant` — Post-surgical adjuvant trials have historically succeeded in PDAC (CONKO-001, ESPAC-4)
5. `moa_chemo_nucleoside` — Gemcitabine backbone correlates with success — the most validated PDAC treatment
6. `moa_immuno_checkpoint` — Checkpoint inhibitors consistently associate with failure
7. `moa_targeted_vegf` — Anti-angiogenic strategies consistently associate with failure

**UMAP structure:**

Trials separate into two distinct clusters in feature space:
- A **chemotherapy-backbone cluster** (upper left) with mixed outcomes — some combinations work, most don't
- A **novel targeted/biological cluster** (lower right) that is overwhelmingly associated with failure

**Translational gap:**

Despite the well-established role of tumor stiffness, collagen deposition, cancer-associated fibroblasts, and extracellular matrix remodeling in PDAC pathophysiology, **none of these biophysical features appear in clinical trial protocol descriptions**. This represents a quantifiable disconnect between basic science and clinical trial design.

---

## Limitations

- **Sample size**: 107 labeled trials limits model complexity and generalizability
- **Publication bias**: labels derived from published abstracts skew toward positive trials
- **Label noise**: LLM-based labeling (Mistral) introduces classification uncertainty
- **Feature confounding**: trial era correlates with both treatment type and outcome, making causal interpretation difficult
- **PDAC heterogeneity**: BRCA-mutated, MSI-high, and KRAS-variant PDAC are biologically distinct but pooled here

---

## Reproducing this project

```bash
# Clone the repo
git clone https://github.com/GiuAlt/PDAC_Trial_Outcome_Predictor.git
cd PDAC_Trial_Outcome_Predictor

# Install dependencies
pip install -r requirements.txt

# Download AACT data
# Register at https://aact.ctti-clinicaltrials.org
# Download the flat text files export and place in Data/

# Run notebooks in order
# 01_Read_Data.ipynb
# 02_Feature_Eng.ipynb
# 03_Modelling.ipynb
```

You will also need **Ollama** with Mistral installed for the LLM labeling step:
```bash
ollama pull mistral
```


## License

MIT License
