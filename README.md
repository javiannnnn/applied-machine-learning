Network Intrusion Detection with Machine Learning

An applied machine learning project that detects malicious network traffic for the fictional cybersecurity organisation **XYZ Cybersecurity**.

The project develops and evaluates:

- A **binary classifier** that predicts `BENIGN` or `ATTACK`
- A **multiclass classifier** that identifies the specific network-traffic category
- Cost-sensitive decision thresholds for operational deployment
- A production-monitoring and responsible-AI plan

The complete analysis, model training and evaluation workflow is available in [`AML.ipynb`](./AML.ipynb).

## Project Overview

Network-intrusion datasets are highly imbalanced: normal traffic and common denial-of-service attacks greatly outnumber rare attack types. This project therefore uses metrics that remain informative under class imbalance instead of relying only on accuracy.

The workflow covers:

1. Dataset loading and consolidation
2. Data cleaning and exploratory analysis
3. Binary and multiclass feature engineering
4. Class-imbalance handling with SMOTE and class weights
5. Baseline model comparison
6. Optuna hyperparameter optimisation
7. Soft-voting and stacking ensembles
8. Confusion-matrix and 3D t-SNE analysis
9. Cost-sensitive threshold selection
10. Production drift monitoring
11. Ethical and responsible-deployment considerations

## Dataset

Three labelled network-flow CSV files are combined into one dataframe.

| Stage | Number of flows |
|---|---:|
| Raw data | 1,054,102 |
| Cleaned data | 944,552 |
| Held-out test set | 188,911 |

The cleaning process:

- Strips whitespace from column names
- Repairs malformed label text
- Removes exact duplicate rows
- Replaces infinite values with missing values
- Drops rows containing invalid values
- Removes constant columns
- Removes the duplicated `Fwd Header Length.1` feature

The multiclass target contains:

- `BENIGN`
- `Bot`
- `DoS GoldenEye`
- `DoS Hulk`
- `DoS Slowhttptest`
- `DoS slowloris`
- `Heartbleed`
- `Web Attack - Brute Force`
- `Web Attack - Sql Injection`
- `Web Attack - XSS`

> The dataset files are not included in this repository. To reproduce the notebook, provide `Dataset1.csv`, `Dataset2.csv` and `Dataset3.csv`, then update the file paths where necessary.

## Models Evaluated

Four individual classification algorithms were evaluated:

- Logistic Regression
- Decision Tree
- Random Forest
- LightGBM

Two multiclass ensembles were also tested:

- Equal-weight soft voting
- Three-fold probability stacking

Hyperparameters were optimised with **Optuna's Tree-structured Parzen Estimator sampler**. Matthews Correlation Coefficient was used as the cross-validation objective because it gives a balanced assessment under severe class imbalance.

## Handling Class Imbalance

### Binary classification

The original attack labels are combined into a single `ATTACK` class.

- An 80/20 stratified train-test split is used.
- SMOTE raises the minority `ATTACK` class to half the size of `BENIGN`.
- SMOTE is fitted only on training data.
- During tuning, SMOTE is performed inside each cross-validation fold to prevent data leakage.

### Multiclass classification

A layered strategy is used:

- SMOTE is applied to selected minority classes with enough real examples.
- Balanced class weights support classes that are not oversampled.
- Ultra-rare classes are not synthetically expanded from only a handful of samples.
- Every ultra-rare training example is retained in the tuning subset.

## Main Results

### Binary classification

The tuned **LightGBM** classifier was the strongest binary model.

| Metric | Tuned LightGBM |
|---|---:|
| Accuracy | 0.9997 |
| Macro F1 | 0.9995 |
| Weighted F1 | 0.9997 |
| ATTACK recall | 0.9998 |
| ATTACK F1 | 0.9993 |
| ATTACK F2 | 0.9996 |
| Matthews Correlation Coefficient | 0.9991 |

The model detected almost every attack while producing very few false alarms on the held-out test set.

### Multiclass classification

The tuned **LightGBM** classifier also achieved the strongest overall multiclass macro F1 score.

| Model | Accuracy | Macro F1 | Macro F2 | MCC |
|---|---:|---:|---:|---:|
| **LightGBM** | **0.9989** | **0.8526** | **0.8693** | **0.9967** |
| Random Forest | 0.9986 | 0.8328 | 0.8242 | 0.9960 |
| Decision Tree | 0.9986 | 0.8283 | 0.8308 | 0.9958 |
| Stacking Ensemble | 0.9972 | 0.8033 | 0.8464 | 0.9919 |
| Soft Voting Ensemble | 0.9989 | 0.7947 | 0.8075 | 0.9968 |
| Logistic Regression | 0.9846 | 0.5437 | 0.5432 | 0.9547 |

Although overall accuracy was close to 100%, the macro metrics show that rare attack categories remain more difficult to classify. Results for `Heartbleed` and `Web Attack - Sql Injection` should be interpreted cautiously because their test-set support is extremely small.

## 3D t-SNE Visualisations

The notebook uses PCA followed by three-dimensional t-SNE to visualise local structure in the high-dimensional network-flow data. The exported figures are stored in the [`t-SNE Plots`](./t-SNE%20Plots) folder.

### Binary actual classes

![Binary 3D t-SNE showing actual classes](./t-SNE%20Plots/Binary%203D%20t-SNE%20Actual%20Classes%20Plot.png)

The embedding shows broad separation between benign and attack traffic, with some overlap where malicious flows have characteristics similar to legitimate activity.

### Binary prediction outcomes

![Binary 3D t-SNE showing prediction outcomes](./t-SNE%20Plots/Binary%203D%20t-SNE%20Prediction%20Outcomes%20Plot.png)

Most observations are classified correctly. Missed attacks are concentrated near overlapping class boundaries rather than being distributed randomly.

### Multiclass actual traffic classes

![Multiclass 3D t-SNE showing actual traffic classes](./t-SNE%20Plots/Multiclass%203D%20t-SNE%20Actual%20Traffic%20Classes.png)

Several traffic categories form distinct local structures, while related attack categories show greater overlap.

### Multiclass correctness

![Multiclass 3D t-SNE showing correct and misclassified observations](./t-SNE%20Plots/Multiclass%203D%20t-SNE%20Correct%20Vs%20Misclassified%20Plot.png)

Misclassifications occur in localised regions, primarily where minority attack classes overlap in the reduced feature space.

> t-SNE is used for exploratory visualisation. It is not used as an input to the classifiers, and distances in the embedding should not be interpreted as exact global relationships.

## Cost-Sensitive Threshold Analysis

In cybersecurity, missing a genuine attack can be far more expensive than investigating a false alarm. The notebook therefore evaluates decision thresholds using an illustrative cost matrix.

| Actual / Predicted | BENIGN | ATTACK |
|---|---:|---:|
| BENIGN | $0 | $125 |
| ATTACK | $4,880,000 | $0 |

Under these assumptions, the calculated cost-minimising threshold was `0.001`.

| Threshold strategy | Threshold | False negatives | False positives | Attack recall | Attack precision |
|---|---:|---:|---:|---:|---:|
| Default | 0.500 | 9 | 50 | 0.9998 | 0.9987 |
| F1-maximising | 0.590 | 9 | 46 | 0.9998 | 0.9988 |
| F2-maximising | 0.280 | 5 | 58 | 0.9999 | 0.9985 |
| Cost-minimising | 0.001 | 0 | 1,231 | 1.0000 | 0.9698 |

This threshold is not presented as a universal deployment value. It depends on the assumed incident cost, analyst capacity, alert budget, probability calibration and operational environment.

## Exploratory Analysis

The notebook includes:

- Class-distribution analysis
- Data-cleaning retention analysis
- Spearman correlation clustermaps
- Benign-versus-attack feature distributions
- Correlation heatmaps by traffic type
- Normalised feature profiles by attack category
- Mutual-information analysis
- Classification reports
- Baseline-versus-tuned comparisons
- Confusion matrices
- Optuna parameter-importance plots
- Interactive Plotly t-SNE figures

Features identified as particularly informative include:

- Packet Length Standard Deviation
- Packet Length Mean
- Backward Packet Length Maximum
- Flow Duration
- Flow Packets/s

## Production Monitoring

The proposed monitoring plan tracks:

- Per-feature Population Stability Index
- Kolmogorov-Smirnov statistics
- Prediction-score drift
- Attack-flag rate
- Recall on analyst-confirmed samples

Potential retraining triggers include severe feature drift, severe prediction drift, sustained abnormal alert rates or a meaningful decline in attack recall.

## Responsible Deployment

The model is intended as a **decision-support system**, not a fully autonomous security authority.

Predictions should be reviewed alongside other evidence before actions such as:

- Permanently blocking a customer
- Isolating a production server
- Taking disciplinary action
- Reporting an individual or organisation

Important risks include:

- Bias caused by unequal class representation
- Poor reliability estimates for ultra-rare attacks
- False positives that increase analyst workload
- False negatives that leave attacks undetected
- Concept drift as network behaviour evolves
- Overconfidence caused by near-perfect aggregate accuracy

## Repository Structure

```text
applied-machine-learning/
├── AML.ipynb
├── README.md
└── t-SNE Plots/
    ├── Binary 3D t-SNE Actual Classes Plot.png
    ├── Binary 3D t-SNE Prediction Outcomes Plot.png
    ├── Multiclass 3D t-SNE Actual Traffic Classes.png
    └── Multiclass 3D t-SNE Correct Vs Misclassified Plot.png
