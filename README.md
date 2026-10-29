# Predicting ER status in breast cancer from gene expression

A small, leakage-free machine-learning project: train a classifier on TCGA-BRCA RNA-seq to predict estrogen receptor (ER) status, then test it on two independent public microarray cohorts from different studies and different chips.

The point of the project is the evaluation design, not a new biomarker. ER status is close to a one-gene readout (ESR1), so a high AUC is expected.

## Data

| Role | Dataset | Platform | Samples used | ER+ fraction |
|---|---|---|---|---|
| Training | TCGA-BRCA (UCSC Xena, legacy HiSeqV2, log2(RSEM+1)) | RNA-seq | 780 primary tumors with a known ER label | 0.77 |
| External test 1 | GEO GSE7390 | Affymetrix HG-U133A | 198 | 0.68 |
| External test 2 | GEO GSE20711 | Affymetrix HG-U133 Plus 2 | 87 | 0.48 |

All data are public and downloaded by the notebook at run time. No data files are stored in this repository.

## Methods

- Shared genes only between training and test cohorts; probes mapped to gene symbols, with multiple probes per gene averaged.
- Per-sample rank transform to reduce differences between RNA-seq and microarray scales.
- Nested cross-validation (5 inner folds for hyperparameters, 5 outer folds for the performance estimate). Gene selection (top-k by F-score), scaling, and the classifier all sit inside one pipeline and are re-fit within each training fold, so nothing leaks.
- Models compared: L2 logistic regression and random forest. The model was chosen by nested-CV AUC on TCGA only.
- External cohorts were used for testing only. GSE20711 was used once, with hyperparameters frozen from the first run.
- Controls: shuffled-label training (repeated), and gene ablations (removing known ER-pathway genes, then removing every gene the model used).

## Results

**Internal (nested CV on TCGA):** logistic regression AUC 0.957 +/- 0.015, random forest AUC 0.954 +/- 0.018.

**External cohorts** (model trained on TCGA only; the second analysis adds per-cohort gene z-scoring):

| Cohort | n | AUC (rank only) | Balanced acc. at 0.5 (rank only) | AUC (z-scored) | Balanced acc. at 0.5 (z-scored) |
|---|---|---|---|---|---|
| GSE7390 | 198 | 0.926 (95% CI 0.882 to 0.963) | 0.567 | 0.931 | 0.869 |
| GSE20711 | 87 | 0.850 (95% CI 0.758 to 0.928) | 0.573 | 0.869 (95% CI 0.783 to 0.940) | 0.808 |

**Controls**
- Shuffled labels: mean external AUC 0.499 (30 repeats, GSE7390) and 0.484 (20 repeats, GSE20711), as expected for no real signal.
- Ablation (GSE7390): removing 10 ER-pathway genes left AUC unchanged (0.926); removing all 200 genes the model used still gave AUC 0.898, so the model relies on a broad ER-associated program rather than a single gene.

## What the results show

- The ranking ability transfers across studies and platforms (AUC about 0.85 to 0.93), with a drop from the internal estimate as expected.
- Raw model scores do not transfer: at the default 0.5 threshold the rank-only model calls most ER+ tumors negative in both external cohorts, because scores are shifted. Per-cohort gene z-scoring largely fixes thresholding, but it does not meaningfully change AUC.

## Limitations

- ER status is nearly a one-gene readout, so this is a methods exercise, not a new biomarker.
- Labels differ in origin (IHC in TCGA, each study's own call in GEO), so some disagreement is label noise.
- Per-cohort z-scoring was chosen after seeing the GSE7390 result, so it is reported as a secondary analysis. It uses the test cohort's own gene means and SDs, so it works for a batch of samples from one study, not a single new patient. It also assumes similar class balance between cohorts, which may contribute to over-calling ER+ in GSE20711 (untested).
- Two external cohorts is a small number; GSE20711 has only 87 samples, so its confidence interval is wide.
- Individual model coefficients should not be interpreted as biology (correlated features in a regularized model).

## How to run

Open `er_status_classifier.ipynb` in Google Colab and choose Runtime > Run all. The first run downloads roughly 100 to 200 MB. Python dependencies are listed in `requirements.txt` (Colab already includes most of them; `GEOparse` is installed by the first cell).

## Data sources

- UCSC Xena TCGA hub (TCGA-BRCA)
- NCBI GEO: GSE7390, GSE20711
