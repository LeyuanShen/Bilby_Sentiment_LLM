# Alibaba News Sentiment and Stock Return Analysis

## Project Overview

This project studies whether Alibaba-related news sentiment can help explain or predict Alibaba's future stock returns.

The workflow combines news data cleaning, sentiment processing, stock return calculation, abnormal return construction, BERT-based sentiment classification, and return prediction analysis.

The main stock analyzed is Alibaba Group Holding Limited using the Hong Kong ticker `9988.HK`. The market benchmark is `3032.HK`, used as a Hang Seng TECH ETF proxy.

---

## Research Question

Can news sentiment related to Alibaba provide useful information for predicting Alibaba's future stock returns?

The project compares three main approaches:

1. Human-labeled sentiment score vs. 21-day abnormal return
2. BERT-predicted sentiment vs. 21-day abnormal return
3. End-to-end BERT regression from news text to 21-day abnormal return

---

## Data Inputs

The project uses two raw news datasets:

```text
sample1_200_en.xlsx
alibaba_sentiment_merged.xlsx
```

The two datasets are standardized into two sample periods:

```text
period_1_2025
period_2_2026
```

Each dataset contains news titles, news bodies, sentiment scores, sentiment explanations, dates, sources, and related metadata.

---

## Project Structure

```text
.
├── sample1_200_en.xlsx
├── alibaba_sentiment_merged.xlsx
├── step1_cleaned_outputs/
├── step2_step3_price_return_outputs/
├── step4_final_dataset_outputs/
├── step5_bert_sentiment_outputs_clean/
├── step7_sentiment_return_outputs/
├── step7B_bert_return_outputs/
├── step8_end_to_end_return_outputs/
└── step10_final_comparison_outputs/
```

---

## Workflow

### Step 1: News Cleaning and Standardization

This step standardizes and cleans the two raw news datasets.

Main operations include:

- Standardizing column names
- Parsing date columns
- Cleaning text fields
- Converting `sentiment_score` into numeric format
- Creating `full_text` by combining title and body
- Generating unique `article_id`
- Adding `source_period`
- Creating a rule-based Alibaba relevance flag, `relevance_flag`
- Removing duplicate articles based on title and body
- Generating quality-check reports

Main outputs:

```text
step1_cleaned_outputs/
├── period_1_2025_cleaned.csv
├── period_2_2026_cleaned.csv
├── step1_quality_report.csv
├── step1_score_distribution.csv
└── step1_cleaned_news_outputs.xlsx
```

---

### Step 2 and Step 3: Stock Price, Forward Return, and Abnormal Return

This step downloads daily stock price data using `yfinance` and calculates forward returns.

Tickers used:

```text
Alibaba stock: 9988.HK
Market benchmark: 3032.HK
```

Forward return windows:

```text
1 trading day
5 trading days
21 trading days
```

For each news article, the publication date is mapped to the next available trading day. Then the following return variables are calculated:

```text
ret_1d
ret_5d
ret_21d
market_ret_1d
market_ret_5d
market_ret_21d
abret_1d
abret_5d
abret_21d
```

Abnormal return is calculated as:

```text
abnormal return = Alibaba return - market benchmark return
```

Main outputs:

```text
step2_step3_price_return_outputs/
├── 9988_HK_price_with_forward_returns.csv
├── 3032_HK_HSTECH_ETF_PROXY_price_with_forward_returns.csv
├── period_1_2025_with_returns.csv
├── period_2_2026_with_returns.csv
├── step2_step3_return_quality_report.csv
└── step2_step3_price_return_outputs.xlsx
```

---

### Step 4: Final Dataset Construction

This step merges the two news periods into one final dataset and prepares return-ready samples.

Main datasets:

```text
final_all
final_return_ready
final_all_relevant
final_return_ready_relevant
```

Sample summary:

```text
Total merged samples: 376
Return-ready samples: 370
Alibaba-relevant samples: 154
Alibaba-relevant and return-ready samples: 150
```

Main outputs:

```text
step4_final_dataset_outputs/
├── final_alibaba_news_dataset.csv
├── final_alibaba_news_dataset_return_ready.csv
├── final_alibaba_news_dataset_relevant_only.csv
├── final_alibaba_news_dataset_return_ready_relevant_only.csv
└── step4_final_alibaba_news_datasets.xlsx
```

---

### Step 5: BERT Sentiment Classifier

This step trains a BERT-based sentiment classifier using `full_text` as the input text.

The original 1-5 sentiment score is converted into a 3-class label:

```text
1 or 2 -> negative
3      -> neutral
4 or 5 -> positive
```

Model used:

```text
distilbert-base-uncased
```

Main outputs:

```text
step5_bert_sentiment_outputs_clean/
├── bert_3class_metrics_summary.csv
├── bert_3class_confusion_matrix.csv
├── bert_3class_test_predictions_clean.csv
├── step5_bert_sentiment_outputs_clean.xlsx
└── best_distilbert_3class_model/
```

The trained model is later used to predict sentiment for the full return-ready dataset.

---

### Step 7A: Human Sentiment vs. Returns

This step tests whether the manually assigned sentiment score is related to Alibaba's future stock returns.

The analysis includes:

- Pearson correlation
- Spearman correlation
- Group mean comparison
- T-tests
- OLS regression

Main dependent variables:

```text
ret_1d
ret_5d
ret_21d
abret_1d
abret_5d
abret_21d
```

Main output folder:

```text
step7_sentiment_return_outputs/
```

---

### Step 7B: BERT-Predicted Sentiment vs. Returns

This step applies the trained BERT sentiment classifier to all return-ready news articles.

The key predicted sentiment variable is:

```text
bert_pred_sentiment
```

The same return tests from Step 7A are repeated using BERT-predicted sentiment.

Main output folder:

```text
step7B_bert_return_outputs/
```

Key result:

```text
BERT-predicted sentiment has a statistically significant positive relationship with 21-day abnormal return.
```

---

### Step 8: End-to-End BERT Return Prediction

This step directly predicts future abnormal return from news text.

Input:

```text
full_text
```

Target:

```text
abret_21d
```

A time-based train, validation, and test split is used because the task is financial return prediction.

Main output folder:

```text
step8_end_to_end_return_outputs/
```

The end-to-end model is compared with a simple mean-return baseline.

---

### Step 10: Final Model Comparison

The final step compares the three modeling approaches.

| Method | Input | Target |
|---|---|---|
| Human sentiment score | Manual sentiment score | `abret_21d` |
| BERT-predicted sentiment | BERT sentiment prediction | `abret_21d` |
| End-to-end BERT regression | News text | `abret_21d` |

Main outputs:

```text
step10_final_comparison_outputs/
├── step10_core_comparison.csv
├── step10_report_ready_summary.csv
├── step10_end_to_end_vs_baseline.csv
├── step10_final_interpretation.txt
└── step10_final_comparison.xlsx
```

---

## Key Findings

### Human Sentiment Score

Human sentiment has a weak positive relationship with 21-day abnormal return, but the relationship is not statistically significant.

```text
Pearson r = 0.0866
Pearson p-value = 0.1097
OLS beta = 0.0066
OLS p-value = 0.1097
```

### BERT-Predicted Sentiment

BERT-predicted sentiment shows a stronger and statistically significant relationship with 21-day abnormal return.

```text
Pearson r = 0.1520
Pearson p-value = 0.0048
OLS beta = 0.0154
OLS p-value = 0.0048
```

### End-to-End BERT Return Regression

The end-to-end BERT return prediction model is less stable and does not clearly outperform the mean-return baseline.

```text
Pearson r = 0.2213
Pearson p-value = 0.0677
RMSE = 0.0864
MAE = 0.0835
Directional accuracy = 0.1159
```

Baseline comparison:

```text
BERT RMSE = 0.0864
Baseline RMSE = 0.0790

BERT MAE = 0.0835
Baseline MAE = 0.0762
```

---

## Overall Conclusion

The results suggest that Alibaba-related news text contains some investment-relevant information. However, this information is better captured through a sentiment-based intermediate representation than through direct end-to-end return prediction.

Among the tested approaches, BERT-predicted sentiment provides the strongest evidence of return relevance for 21-day abnormal returns. Human sentiment shows a weaker positive relationship, while direct end-to-end BERT return regression does not clearly outperform a simple baseline.

---

## Installation

Install the required Python packages:

```bash
pip install pandas numpy openpyxl yfinance scikit-learn scipy statsmodels torch transformers datasets accelerate
```

Optional, if running in a Jupyter Notebook environment:

```bash
pip install notebook ipykernel
```

---

## How to Run

Run the notebook or scripts in the following order:

```text
Step 1  - Clean and standardize news data
Step 2  - Download stock and benchmark price data
Step 3  - Calculate forward returns and abnormal returns
Step 4  - Build final return-ready dataset
Step 5  - Train BERT sentiment classifier
Step 7A - Analyze human sentiment vs. returns
Step 7B - Analyze BERT-predicted sentiment vs. returns
Step 8  - Train end-to-end BERT return prediction model
Step 10 - Compare all methods
```

Make sure the raw Excel files are placed in the project root before running Step 1.

---

## Important Columns

| Column | Description |
|---|---|
| `article_id` | Unique article identifier |
| `date` | Original news publication date |
| `event_date` | Matched trading date |
| `title_en` | English news title |
| `body_en` | English news body |
| `full_text` | Combined title and body |
| `sentiment_score` | Original human sentiment score |
| `sentiment_3class` | Converted 3-class sentiment label |
| `bert_pred_sentiment` | BERT-predicted sentiment label |
| `relevance_flag` | Rule-based Alibaba relevance flag |
| `ret_1d` | 1-day Alibaba forward return |
| `ret_5d` | 5-day Alibaba forward return |
| `ret_21d` | 21-day Alibaba forward return |
| `abret_1d` | 1-day abnormal return |
| `abret_5d` | 5-day abnormal return |
| `abret_21d` | 21-day abnormal return |

---

## Notes

- `3032.HK` is used as a Hang Seng TECH ETF proxy because direct index data may be incomplete in `yfinance`.
- The relevance flag is rule-based and should be treated as a screening variable rather than a final manual judgment.
- The analysis is intended for academic research and should not be interpreted as investment advice.
- Financial return prediction is noisy, and statistical significance should be interpreted carefully.
