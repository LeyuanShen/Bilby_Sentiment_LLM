# Alibaba News Sentiment Analysis — Development Summary

## Purpose of This File

This file is a short summary of the project development process.

It explains:

- what problem we worked on
- how the project evolved
- what datasets and methods were used
- what results were achieved
- how the final delivered package was formed

This file is included for presentation and review. It is not required to run the code, but it helps readers understand the full workflow behind the final deliverables.

---

## 1. Project Goal

The goal of this project is to build a practical workflow for scoring the sentiment of Alibaba-related financial news.

We use a 5-point scale:

- `1` = strongly negative
- `2` = negative
- `3` = neutral
- `4` = positive
- `5` = strongly positive

The final objective is not only to label news articles, but also to test whether the sentiment signal contains useful information for short-term market reaction.

---

## 2. Project Development Process

### Stage 1 · Article Filtering and Early Exploration

We first identified Alibaba-related articles from a larger news collection and explored simple rule-based sentiment ideas.

This stage helped us understand the data and confirmed that keyword or dictionary methods were not strong enough for this task, because financial news sentiment depends heavily on context.

### Stage 2 · Human Annotation

We manually labeled two Alibaba-related datasets.

The first dataset was an independent 200-article gold-standard set. It was used as an external evaluation dataset.

The second dataset was `alibaba_sentiment_merged.xlsx`, which contains 176 manually labeled Alibaba-related articles. This became the main dataset for LLM prompt development and evaluation.

### Stage 3 · LLM Prompt Engineering

We designed and tested multiple prompt variants using Google Gemini.

The final prompt, `VFS_Analyst_v6`, treats the model as a sell-side financial analyst covering Alibaba and uses calibrated rules to better distinguish neutral, negative, and positive news.

### Stage 4 · External Validation and Signal Testing

After selecting the final prompt, we evaluated it in two ways:

- classification performance against human labels
- relationship between sentiment scores and Alibaba stock excess returns

We also built a baseline comparison using text embeddings and Ridge regression.

---

## 3. Main Datasets Used

### Main Development Dataset

File: `data/alibaba_sentiment_merged.xlsx`

- 176 manually labeled Alibaba-related articles
- used for prompt development and in-sample evaluation
- split into train / validation / test sets

### External Validation Dataset

File: `data/200_gold_standard.csv`

- 200 manually labeled articles
- used only for external validation
- not used during prompt development

---

## 4. Main Method

The main method is an LLM few-shot sentiment scoring pipeline.

Core setup:

- model: Google Gemini 2.5 Flash Lite
- input: article title and article body
- output: sentiment score, confidence, and reasoning
- prompt style: few-shot prompt with analyst persona and score calibration rules

Main notebook:

- `LLM_Sentiment_Rating_Fixed.ipynb`

External validation notebook:

- `LLM_200_Gold_Evaluation.ipynb`

Comparison baseline notebook:

- `Approach3_Embedding_Regression.ipynb`

---

## 5. Final Results

### Results on `alibaba_sentiment_merged.xlsx`

This dataset was used as the main dataset for LLM development.

| Split | Exact Match | ±1 Accuracy | Spearman ρ | n |
|------|-------------|-------------|------------|---|
| Train | 68.6% | 98.1% | 0.820 | 105 |
| Validation | **82.9%** | 100% | 0.905 | 35 |
| Test | **86.1%** | 100% | 0.904 | 36 |

### Results on `200_gold_standard.csv`

This dataset was used as an external test after prompt development.

| Metric | Result |
|------|--------|
| Exact Match | **69.0%** |
| ±1 Accuracy | **100.0%** |
| MAE | 0.310 |
| Spearman ρ | 0.540 |
| n | 200 |

Interpretation:

- the final prompt performs strongly on the main labeled dataset
- it also generalizes reasonably well to a separate gold-standard dataset
- most prediction errors remain close to the correct score

### Market Signal Result

For Alibaba next-day excess return, the LLM-predicted sentiment score achieved:

- Spearman rho = `0.228`
- p-value = `0.023`

This suggests that the LLM sentiment score contains useful short-term market information.

### Baseline Comparison Result

The embedding + Ridge regression baseline achieved:

- daily aggregated T+1 Spearman rho = `0.029`
- p-value = `0.67`

This result was not statistically significant, so the LLM pipeline outperformed the simpler baseline.

---

## 6. Final Delivered Package

The final submitted package contains:

- cleaned notebooks
- input data needed for final evaluation
- output CSV and JSON results
- visualization files
- prompt documentation
- project README
- this development summary
- presentation slides and speech script

This package is designed to be:

- readable for reviewers
- reproducible for demonstration
- practical for company-facing delivery

---

## 7. Key Achievements

By the final stage of the project, we completed the following:

- built an end-to-end Alibaba financial news sentiment scoring workflow
- manually labeled a 176-article Alibaba dataset for model development
- manually labeled and evaluated on a separate 200-article gold-standard dataset
- developed the final LLM prompt `VFS_Analyst_v6`
- achieved **86.1%** exact-match accuracy on the held-out test split of the 176-article dataset
- achieved **69.0%** exact-match accuracy on the external 200-article gold-standard dataset
- found a significant T+1 excess-return signal from the LLM sentiment score
- packaged the final code, results, README, and presentation materials into one handoff folder

---

## 8. Recommended Reading Order

For a quick overview:

1. `README.md`
2. `Work.md`

For final code and results:

1. `LLM_Sentiment_Rating_Fixed.ipynb`
2. `LLM_200_Gold_Evaluation.ipynb`
3. `output/`

For presentation materials:

1. `Alibaba_News_Sentiment_Presentation.pptx`
2. `speech_script_2min.txt`