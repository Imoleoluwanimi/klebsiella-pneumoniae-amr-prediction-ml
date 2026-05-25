# Predicting Antibiotic Resistance in Bacteria Using Machine Learning
### A Genomic Data Science Case Study *Klebsiella pneumoniae* Meropenem Resistance

**Author:** Delight Abioye | Bio-AI Researcher | Nigeria  
**Contact:** abioyedelight2017@gmail.com | [LinkedIn](https://linkedin.com/in/delight-abioye-66533723b) | [GitHub](https://github.com/Imoleoluwanimi)

---

## What This Project Does

This project builds a complete machine learning pipeline that reads the DNA sequence of a *Klebsiella pneumoniae* bacterium and predicts whether it will resist meropenem a last-resort antibiotic used in Nigerian and global hospitals when almost everything else has failed.

Instead of waiting 48–72 hours for a laboratory culture result, a genomic resistance prediction tool could give clinicians actionable information in minutes, potentially saving lives in resource-limited settings where treatment delays are common.


## Why This Problem Matters in Nigeria

*Klebsiella pneumoniae* is one of the leading causes of hospital-acquired infections in Nigerian hospitals responsible for pneumonia, bloodstream infections, and surgical wound infections. Carbapenem-resistant K. pneumoniae (CRKP) is a growing crisis because carbapenems like meropenem are last-resort antibiotics. When resistance develops, treatment options become extremely limited.

In a critically ill patient, every hour on the wrong antibiotic matters. A fast, DNA-based resistance prediction tool could directly improve clinical outcomes in settings where slow laboratory infrastructure is a real barrier to timely care.


## The Most Important Finding - Phylogenetic Data Leakage

> **This project's most valuable contribution is not its model performance, it is what we discovered when we investigated why the model performed so well.**

An initial train-test split produced **100% classification accuracy** an impossible result that signalled something was wrong. After implementing cross-validation, AUC remained at 94.41% still suspiciously high.

We ran a **pairwise cosine similarity analysis** on k-mer profiles across 200 randomly sampled genomes and discovered:

- **98% of genome pairs** showed similarity greater than 0.99
- 200 sampled genomes collapsed to just **2 distinct biological clusters** at 95% similarity
- Multiple genome pairs showed **1.0 similarity**, identical DNA profiles

**The biological explanation:** K. pneumoniae resistant to meropenem spreads through hospital outbreaks. A single resistant strain infects dozens of patients in the same ward. Each patient's isolate gets sequenced and submitted to the database separately but they are all essentially the same bacterium. Our 726 genomes likely represent only 17–50 truly distinct strains.

**The implication:** Our models were memorizing strain fingerprints, not learning generalizable resistance mechanisms. This is called **phylogenetic data leakage**  one of the most common but least discussed problems in microbial genomics ML. Standard cross-validation does not catch it. We did.

This finding directly informs the methodology of the follow-up MDR-TB research project, where pairwise similarity analysis is applied first to validate genuine dataset diversity.


## Pipeline Overview


BV-BRC Database (global genomic AMR data)
        ↓
API Query → K. pneumoniae + Meropenem labels
        ↓
Data Cleaning (remove unlabeled, Intermediate, duplicates)
        ↓
Genome Sequence Download (batch FASTA retrieval)
        ↓
k-mer Feature Extraction (k=6, 18,016 features)
        ↓
Sparse Matrix Storage (scipy.sparse - memory efficient)
        ↓
Phylogenetic Leakage Investigation (cosine similarity + clustering)
        ↓
Multi-Model Training with SMOTE (LR, SVM, RF, XGBoost)
        ↓
5-Fold Stratified Cross-Validation
        ↓
Comprehensive Evaluation (AUC, Recall, Precision, Specificity, F1, MCC)
        ↓
Results Interpretation with Clinical Context



## Dataset

| Property | Value |
|---|---|
| Source | BV-BRC (Bacterial and Viral Bioinformatics Resource Center) |
| Organism | *Klebsiella pneumoniae* |
| Antibiotic | Meropenem (carbapenem class - last resort) |
| Total genomes | 726 (after cleaning) |
| Susceptible | 514 (70.8%) |
| Resistant | 212 (29.2%) |
| Feature space | 18,016 unique 6-mers per genome |

---

## What Are K-mers?

A k-mer is a short DNA substring of fixed length k. By sliding a window of length k across a genome and counting the frequency of every unique substring, we create a numerical fingerprint for that genome.

**Example k=3 on sequence ATGCTT:**
```
ATG → TGC → GCT → CTT
```

Antibiotic resistance genes have characteristic DNA sequences that produce distinctive k-mer patterns. Without telling the model which genes to look for, k-mer frequencies allow it to discover these patterns from labeled data.

We used **k=6** producing 18,016 unique hexanucleotide features per genome, stored as a sparse matrix.



## Results

### Multi-Model Comparison k=6 Features with SMOTE

| Model | AUC-ROC | Recall | Precision | Specificity | F1 | MCC | Recall Std |
|---|---|---|---|---|---|---|---|
| Logistic Regression | 94.16% | 82.98% | 78.13% | 90.26% | 80.32% | 0.72 | 5.16% |
| **Linear SVM** | **96.24%** | **88.65%** | **90.25%** | **95.91%** | **89.36%** | **0.85** | **2.86%** |
| Random Forest | 94.63% | 79.71% | 79.13% | 91.06% | 79.07% | 0.71 | 7.02% |
| XGBoost | 96.11% | 89.08% | 80.11% | 90.66% | 84.13% | 0.78 | 7.03% |

**Linear SVM is the best model** wins on AUC, Precision, Specificity, F1, MCC, and stability (lowest Recall Std).

### Why Linear SVM Wins

Linear SVM is architecturally designed for high-dimensional sparse data exactly the structure of k-mer matrices. It outperforms tree-based models (Random Forest, XGBoost) because it finds optimal decision boundaries in high-dimensional spaces more efficiently, and is less prone to overfitting on clonal data.

### Clinical Context

| Metric | Value | What It Means |
|---|---|---|
| Recall (Sensitivity) | 88.65% | Of 100 resistant patients - 89 correctly flagged |
| Specificity | 95.91% | Of 100 susceptible patients - 96 correctly cleared |
| Missed resistant cases | ~11% | These patients receive ineffective meropenem |
| False alarms | ~4% | These patients unnecessarily escalated to alternatives |

> **Important caveat:** These metrics are influenced by the clonal dataset structure. Performance on genuinely diverse or novel strains would likely be lower. Clinical deployment requires validation on a phylogenetically independent dataset.


## Honest Limitations

1. **Phylogenetic clonal bias** - 98% of genome pairs share >0.99 DNA similarity. Models memorize strain fingerprints rather than generalizable resistance mechanisms. Phylogeny-aware train-test splitting is required for unbiased evaluation.

2. **Global dataset** - BV-BRC data is dominated by European and Asian isolates. Performance on Nigerian clinical K. pneumoniae strains is unknown.

3. **Single antibiotic focus** - This pipeline addresses meropenem resistance only. Clinical utility requires multi-drug profiles.

4. **SHAP interpretation not yet completed** - Identifying which specific k-mers and resistance genes drive model predictions is planned as future work.



## Future Work

- [ ] **k=8 comparison** - Evaluate whether longer k-mers improve classification under clonal conditions
- [ ] **SHAP biological interpretation** - Map predictive k-mers to known resistance genes via BLAST
- [ ] **Phylogeny-aware splitting** - Implement genome clustering before train-test splitting for unbiased metrics
- [ ] **African isolate validation** - Filter for African-origin isolates and assess geographic performance differences
- [ ] **Cross-Pathogen Transferability** - Apply this pipeline with diversity validation to my ongoing research, poster accepted at Deep Learning Indaba 2026

---

## Repository Structure

```
klebsiella-pneumoniae-amr-prediction-ml/
│
├── notebooks/
│   ├── 01_Data_Extraction_KPN_AMR.ipynb     # API query, download, cleaning
│   └── 02_AMR_Feature_Engineering.ipynb     # k-mers, leakage investigation, modeling
│
├── report/
│   └── KPN_AMR_Project_Report.pdf           # Full project report
│
├── figures/
│   ├── class_distribution.png               # Class balance visualization
│   ├── genome_length_distribution.png       # Sequence length EDA
│   ├── similarity_heatmap.png               # Phylogenetic similarity matrix
│   ├── confusion_matrix_linear_svm.png      # Best model confusion matrix
│   └── roc_curves_all_models.png            # ROC curves — all four models
│
├── README.md
└── requirements.txt
```

---

## Technical Stack

| Component | Tool |
|---|---|
| Data acquisition | BV-BRC REST API, Python requests |
| Sequence handling | Biopython |
| Feature extraction | scikit-learn CountVectorizer |
| Sparse storage | scipy.sparse |
| Class imbalance | imbalanced-learn SMOTE |
| Models | scikit-learn (LR, LinearSVC), XGBoost |
| Evaluation | scikit-learn metrics |
| Similarity analysis | scikit-learn cosine_similarity |
| Clustering | scikit-learn AgglomerativeClustering |
| Visualization | matplotlib, seaborn |
| Environment | Google Colab, Google Drive |

---

## How to Run This Project

### Requirements
```bash
pip install biopython scikit-learn xgboost imbalanced-learn scipy pandas numpy matplotlib seaborn requests
```

### Steps
1. Open `notebooks/01_Data_Extraction_KPN_AMR.ipynb` in Google Colab
2. Run all cells to download data and genome sequences from BV-BRC
3. Files are saved to Google Drive automatically
4. Open `notebooks/02_AMR_Feature_Engineering.ipynb`
5. Run reload cell at top, loads all data without re-downloading
6. Run remaining cells in order - feature extraction, leakage investigation, modeling

> **Note:** Genome download takes 30–60 minutes depending on connection speed. All intermediate files are saved to Drive so you never need to re-download.

---

## About This Project

This project is part of a larger body of work applying AI and machine learning to infectious disease problems in Nigeria. It serves as:

1. **A learning foundation** building skills in genomic ML pipelines, biological feature engineering, and clinical evaluation frameworks

2. **A methodological contribution** the phylogenetic leakage finding is a genuine research finding that informs best practices in microbial genomics ML

3. **A bridge to research** The methodology validated in this project provides the foundational framework for my upcoming poster presentation on localized genomic pathogen modeling at the Deep Learning Indaba 2026.

---

## Author

**Delight Abioye** is a Nigerian microbiologist and independent Bio-AI researcher applying machine learning to infectious disease problems indigenous to Nigeria. She holds a B.Tech in Microbiology from the Federal University of Technology Minna and independently built expertise in AI/ML through free programs and self-directed learning starting with no laptop and no institutional support.

Her work sits at the intersection of microbiology, genomics, and machine learning, with a focus on building computational tools specifically validated for Nigerian and African clinical contexts.

-  abioyedelight2017@gmail.com
-  [LinkedIn](https://linkedin.com/in/delight-abioye-66533723b)
-  [GitHub](https://github.com/Imoleoluwanimi)


