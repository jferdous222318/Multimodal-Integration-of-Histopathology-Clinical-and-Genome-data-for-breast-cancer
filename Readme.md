OVERVIEW

This project predicts five clinical outcomes for breast cancer patients by
combining three kinds of data:
  - Clinical records (age, stage, receptor status, etc.)
  - Genomic data (RNA-seq gene expression)
  - Histopathology slide images (H&E whole-slide images)
 
Main question: does combining the three data types predict outcomes better
than any single one?
 
Outcomes (each is a yes/no label):
  1. Cancer stage          late (III-IV) vs early (I-II)
  2. Lymph node spread     node positive vs not
  3. Survival              died vs alive at last check-up
  4. Recurrence            cancer came back / progressed / persisted
  5. Treatment response    complete response vs not
 
2. DATA

Source: TCGA-BRCA (public dataset, GDC portal).
 
  Clinical          1,095 patients (1,098 before matching)
  Genomic (RNA-seq) 1,095 patients
  Slide images      118 patients, 140 slides, 50,718 tiles (512x512)
 
Slides: 101 tumour, 38 normal, 1 metastatic. 93 patients have a tumour slide.
Mean 430 tiles per patient (range 58 to 2,432). Up to 100 tiles per patient
are embedded.
 
Label counts (positive / total):
  Task         Matched cohort (118)   Full cohort (1,095)
  Stage        32 / 108               246 / 994
  Node         57 / 109               496 / 985
  Survival     25 / 118               152 / 1,095
  Recurrence   21 / 118               120 / 1,095
  Treatment    10 / 118               124 / 1,095
 
Only 118 patients have all three data types. Treatment response has just
10 positives in that group, so its results are exploratory only.

3. NOTEBOOKS

  temp-data    Clinical and genomic preprocessing, label construction
  cleaning     Slide tiling, quality control, Macenko stain normalisation
  fydp-final   Embeddings, models, cross-validation, results, GradCAM
  histo-2nd    preprocess histopathoogy image
 
Run order on Kaggle (fydp-final): pip cell, setup cell, cohort cell,
embedding cell (about 12 min on GPU), encoder comparison, then the cell that
defines LateFusion / evaluate / run_cv / make_models.
 

4. PREPROCESSING

4.1 Clinical (temp-data)
  - Merge clinical, follow_up, pathology_detail, exposure TSVs by patient ID
  - Replace GDC missing marker with NaN; keep the latest follow-up per patient
  - 471 columns -> 119 engineered features:
      11 numeric   (median imputation, z-score)
      5 ordinal    (AJCC stage, T, N, M, grade; order kept)
      103 one-hot  (nominal columns)
  - Leakage columns (LEAK_COLS) removed in fydp-final: columns that encode
    the label (stage/T/N/M/grade for stage and node, survival time and death
    columns, recurrence source columns). Final: 103 clinical features.
 
4.2 Genomic (temp-data)
  - 4,472 RNA TSVs mapped to patients through the metadata JSON
  - One file per patient kept -> 1,095 patients
  - Value used: tpm_unstranded
  - 60,660 genes -> variance filter (0.1) -> 33,730
  - log1p -> top 2,000 genes by standard deviation -> z-score
  - PCA to 100 components (73.4% variance)
 
4.3 Labels
  Survival    vital_status = Dead
  Treatment   treatment outcome = complete response
  Stage/Node  from AJCC values
  Recurrence  the progression_or_recurrence column gave 0 positives, so the
              label was rebuilt from follow_up using four signals (any one
              = positive): recurrence/progression "Yes", days_to_recurrence
              > 0, days_to_progression > 0, disease response "With Tumor".
              Result: 120 positives.
 
4.4 Images (cleaning)
  - Slides cut into 512x512 tiles
  - Quality control: reject tiles more than 85% white, more than 50% dark,
    or blurry (Laplacian variance below 50)
  - Macenko stain normalisation against one reference tile (torchstain)
  - Manifest CSV: tile path, slide, patient ID
 

5. FEATURE EXTRACTION FROM IMAGES
Frozen pretrained encoders (no training, 118 patients are too few):
  ResNet50          pilot / baseline only
  EfficientNet-B4   1,792 dimensions, 380 px
  ViT-Small         384 dimensions, 224 px
  PLIP (vinid/plip) 768 dimensions, L2-normalised
 
Per patient: mean pooling of up to 100 tile embeddings, StandardScaler,
PCA to 50 components.
Encoder per task: ViT-Small for stage, survival, recurrence, treatment;
EfficientNet-B4 for node. This choice was made on the same 118 patients,
so it is slightly optimistic.
 

6. MODELS

  SVM                      good on small data
  XGBoost (max_depth 2)    non-linear; shallow to limit overfitting
  LightGBM                 tree model, different splitting strategy
  ElasticNet logistic reg. stable with many features, few patients
  LateFusion               three experts (SMOTE + Scaler + PCA10 +
                           ElasticNet), one logistic meta-learner
 
Tried and not used in final results:
  AttentionMIL   failed in a 64-patient pilot
  Swin, GAT, TabNet, DeepSurv   proposed, never implemented
 
7. EVALUATION

  - Leave-one-out CV on the 118 patients
  - 5-fold stratified CV on the 1,095 patients
  - SMOTE and scaling inside training folds only (no test leakage)
  - Metrics: AUC (main), F1, precision, recall, balanced accuracy, MCC
  - Reported: best-classifier AUC and mean over 4 classifiers
  - Seven experiments: Clinical, Genomic, Clinical+Genomic, Histo,
    Clinical+Histo, Genomic+Histo, All three
  - Same-118 baselines so clinical/genomic can be compared fairly with images
 

8. RESULTS (best-classifier AUC)

  Task        Best configuration          AUC
  Stage       Clinical + Genomic          0.748
  Node        Clinical + Genomic          0.826
  Survival    Clinical + Histo (LightGBM) 0.822  (95% CI 0.708-0.913)
  Recurrence  Clinical                    0.739
  Treatment   Histo only (XGBoost)        0.849  (only 10 positives)
 
Findings:
  - Clinical data is the strongest and most reliable source.
  - Genomic data alone is weak (about 0.57 to 0.64).
  - Fusion is task dependent; using all three is not always better.
  - PLIP did not beat the ImageNet encoders.
  - Survival and treatment findings depend on ViT-Small:
    with EfficientNet-B4 they drop to 0.687 and 0.424.
 
9. INTERPRETABILITY

  - GradCAM on frozen EfficientNet-B4 (label independent, grid artifacts)
  - PLIP attention rollout (attends to borders/background)
  Both are qualitative examples, not evidence.
 

10. LIMITATIONS

  - Only 118 patients with images; 10 treatment positives
  - Public data only, no external validation
  - Survival label ignores follow-up time (censoring)
  - Missing treatment/recurrence data treated as negative
  - Recurrence definition is broad (includes "With Tumor")
  - Encoder choice made on the same 118 patients (optimistic)
  - Bootstrap CI understates real uncertainty
  - Genomic scaler and PCA fitted on all patients before CV
  - One RNA file per patient kept; sample type (tumour 01 vs normal 11)
    not verified
  - Mean pooling can hide small tumour regions
  - Many blank tiles in slides; GradCAM/attention are qualitative
 
======================================================================
11. REQUIREMENTS
======================================================================
Python 3, numpy, pandas, scikit-learn, imbalanced-learn, xgboost, lightgbm,
torch, torchvision, timm, transformers (PLIP), torchstain, opencv-python,
matplotlib. A GPU is needed for embedding extraction.