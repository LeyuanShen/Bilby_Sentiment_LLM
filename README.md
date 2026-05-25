# Alibaba News Sentiment Analysis — LLM Pipeline

## Project Overview

This project implements an automated **financial news sentiment scoring system** for Alibaba Group (9988.HK) using Google Gemini LLM with few-shot prompting. The system rates news articles on a 1–5 scale and evaluates whether sentiment scores predict next-day stock excess returns.

`Work.md` is included as a companion document that summarizes the project development process, dataset construction, and how the final delivered package was formed.

### Key Results

| Metric | Value |
|--------|-------|
| Model | Google Gemini 2.5 Flash Lite |
| Test Accuracy (exact match) | **86.1%** |
| Test ±1 Accuracy | 100% |
| T+1 Excess Return Spearman ρ | 0.228 (p=0.023) |
| External Validation (200 articles) | 69.0% exact, 100% ±1 |

---

## File Structure

```
Alibaba_Sentiment_LLM/
├── README.md                          ← This file
├── Work.md                            ← Development summary and workflow overview
├── LLM_Sentiment_Rating_Fixed.ipynb   ← Main pipeline (train/test + signal analysis)
├── LLM_200_Gold_Evaluation.ipynb      ← External validation on 200-article dataset
├── Approach3_Embedding_Regression.ipynb ← Comparison: text embedding approach
├── prompt.md                          ← Full prompt design documentation
├── data/
│   ├── alibaba_sentiment_merged.xlsx  ← 176 human-annotated articles (train/val/test)
│   └── 200_gold_standard.csv         ← 200-article independent validation set
└── output/
    ├── fs_train_VFS_Analyst_v6.csv    ← Training set predictions (n=105)
    ├── fs_val_VFS_Analyst_v6.csv      ← Validation set predictions (n=35)
    ├── fs_test_VFS_Analyst_v6.csv     ← Test set predictions (n=36)
    ├── 200_gold_VFS_Analyst_v6.csv    ← External validation predictions (n=200)
    ├── prompt_registry.json           ← Prompt version history & metrics
    ├── return_signal_summary.json     ← Signal analysis results (Spearman ρ, OLS β)
    ├── sentiment_vs_returns.png       ← Bar chart: mean excess return by score
    ├── sentiment_vs_returns_scatter.png ← Scatter: article-level score vs return
    └── approach3_results.json         ← Embedding regression baseline results
```

---

## Notebooks

### 1. `LLM_Sentiment_Rating_Fixed.ipynb` — Main Pipeline

**Purpose**: End-to-end LLM sentiment scoring + stock return signal evaluation.

**Steps**:
1. Import libraries & configure Gemini API parameters
2. Define `GeminiLLMClient` class with retry logic
3. Set API key (user must provide their own)
4. Run VFS_Analyst_v6 prompt on train/test sets (with caching)
5. Fetch 9988.HK & ^HSI prices, compute T+1/T+3/T+5 excess returns
6. Signal analysis: Spearman ρ, OLS regression, score-bucket returns
7. Visualization: bar charts and scatter plots

**Caching**: Results are saved to CSV. Re-running will load from cache (no API calls needed).

---

### 2. `LLM_200_Gold_Evaluation.ipynb` — External Validation

**Purpose**: Test the same prompt on a completely independent 200-article dataset.

The 200 articles were **never seen** during prompt development. This validates generalization:

| Metric | Original Test Set | 200 Gold Standard |
|--------|------------------|-------------------|
| Exact match | 86.1% | 69.0% |
| ±1 accuracy | 100% | 100% |
| Spearman ρ | 0.904 | 0.540 |

The lower exact match on the 200-article set is primarily due to the model's conservative bias on Score 4 articles (predicting 3 instead of 4).

---

### 3. `Approach3_Embedding_Regression.ipynb` — Baseline Comparison

**Purpose**: Compare LLM prompting against a traditional ML approach (text embeddings + Ridge regression).

**Method**:
- Encode articles using `all-MiniLM-L6-v2` (384-dim sentence embeddings)
- Ridge regression with time-series cross-validation to predict T+1 excess returns
- PCA dimensionality reduction

**Result**: Daily-aggregated Spearman ρ = +0.029 (p=0.67, not significant), confirming that the LLM approach (ρ=0.228) significantly outperforms raw embedding regression.

---

## How to Run

### Prerequisites

```bash
pip install google-genai pandas numpy openpyxl scipy scikit-learn yfinance statsmodels matplotlib
# For Approach 3 only:
pip install sentence-transformers torch
```

### Quick Start

1. Open `LLM_Sentiment_Rating_Fixed.ipynb`
2. In Step 3, replace `"YOUR_API_KEY_HERE"` with your Gemini API key
   - Get one at: https://aistudio.google.com/apikey
3. Run all cells in order

**Note**: If output CSV files exist in `output/`, results will load from cache (no API calls). Delete the CSV files to force re-evaluation.

### VPN Requirement

Google Gemini API is not available in all regions. If you see `FAILED_PRECONDITION` errors, connect a VPN to a supported region (US, Japan, etc.).

---

## Methodology

### Sentiment Scale

| Score | Meaning | Example |
|-------|---------|---------|
| 1 | Strongly Negative | Major crisis directly involving Alibaba |
| 2 | Negative | Alibaba-specific penalty, platform-targeting regulation |
| 3 | Neutral | General market news, indirect mention, mixed signals |
| 4 | Positive | Direct Alibaba win, AI infrastructure benefiting Alibaba Cloud |
| 5 | Strongly Positive | Exceptional market-leadership achievement |

### Prompt Design (VFS_Analyst_v6)

- **Persona**: Sell-side financial analyst covering Alibaba
- **Few-shot**: 2 examples per score class (10 total) from training set
- **Calibration rules**: Explicit boundary conditions for scores 2, 4, 5
- **Scoring process**: 5-step checklist to ensure consistent classification
- **Output**: JSON with score, confidence, key_factors, reasoning

### Signal Evaluation

- Excess return = Alibaba daily return − Hang Seng Index daily return
- Post-market adjustment: articles after 15:00 HKT use T+2 as base
- Evaluation horizons: T+1, T+3, T+5 trading days

---

## Dataset Description

### `alibaba_sentiment_merged.xlsx` (176 articles)

- Source: Financial news from 2026-02 to 2026-04
- Columns: `date`, `original_title_en`, `original_body_en`, `sentiment_score`, `sentiment_reason`
- Score distribution: 1(2), 2(50), 3(69), 4(49), 5(6)
- Split: 60% train / 20% val / 20% test (stratified, random_state=42)

### `200_gold_standard.csv` (200 articles)

- Independent validation set (not used in prompt development)
- Columns: `inserted_at`, `title_en`, `body_en`, `newspaper`, `sentiment_score`, `sentiment_reason`
- Score distribution: 2(1), 3(113), 4(83), 5(3)

---

## API Cost Estimate

| Task | Articles | Estimated Cost |
|------|----------|---------------|
| Full train+val+test | 176 | ~$0.05 (Gemini Flash Lite) |
| 200 gold standard | 200 | ~$0.06 |
| Single article | 1 | < $0.001 |

Gemini 2.5 Flash Lite is extremely cost-efficient for classification tasks.

---

## Contact

HKU Capstone Project — Quant China Data Team
