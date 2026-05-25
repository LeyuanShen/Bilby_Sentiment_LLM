# LLM Sentiment Prompt Registry — Alibaba News

## Setup

| Item | Value |
|------|-------|
| Dataset | `alibaba_sentiment_merged.xlsx` — 176 articles, manually annotated |
| Split | 6:2:2 stratified (seed=42) → train=105, val=35, test=36 |
| Train eval | 30-article sample (few-shot examples excluded to prevent leakage) |
| Model | `gemini-2.5-flash-lite` (Google Gemini API) |
| Target | Val exact match ≥ 80% |

## Score Distribution (full dataset)

| Score | Label | Count |
|-------|-------|-------|
| 1 | Strongly Negative | 2 |
| 2 | Negative | 50 |
| 3 | Neutral | 69 |
| 4 | Positive | 49 |
| 5 | Strongly Positive | 6 |

---

## Results Summary

| # | Prompt | Strategy | Train | Val | Test | Status |
|---|--------|----------|-------|-----|------|--------|
| 1 | VFS_2per | 2 examples/class, basic scale def | 70.0% | — | — | Baseline |
| 2 | VFS_3per | 3 examples/class, basic scale def | 63.3% | — | — | More examples hurt |
| 3 | VFS_CoT | 2/class + 4-step CoT | 73.3% | 62.9% | — | First val run |
| 4 | VFS_Boundary | 2/class + verbose boundary rules + Q1–Q3 checklist | 66.7% | — | — | Verbose hurts |
| 5 | VFS_Select | Boundary-aware selection + focused 3/4 rule + CoT | 66.7% | — | — | Worse than CoT |
| 6 | VFS_SmartBody | Smart body extract (Alibaba section) + 2/class + CoT | 66.7% | — | — | Smart body didn't help |
| 7 | VFS_Flash | `gemini-2.5-flash` model instead of lite | 50.0% | — | — | All same pred; ρ=NaN |
| 8 | VFS_TwoStep | Two-stage: classify subject then score | 76.7% | 60.0% | — | "Primary subject" criterion wrong |
| 9 | VFS_Hybrid | TwoStep + CoT hybrid | 50.0% | — | — | Degenerate output |
| 10 | VFS_Refine34 | Score first, then refine 3/4 | 60.0% | 60.0% | — | Refinement not better than CoT |
| 11 | VFS_Analyst | **Financial analyst perspective** (investor framing) | 66.7% | 68.6% | — | Key insight: analyst framing works |
| 12 | VFS_Analyst_v2 | Analyst + low threshold for 2/4 | 53.3% | 65.7% | — | Score 3 accuracy collapsed (33%) |
| 13 | VFS_Analyst_v3 | Analyst + "dominant theme" language | 73.3% | 71.4% | — | Score 4=85% but score 3=58% |
| 14 | VFS_Analyst_v4 | v3 + strict "named Alibaba" for score 2 | 76.7% | 65.7% | — | Over-restricted score 2 (56%) |
| 15 | VFS_Analyst_v5 | Domain-specific 2/3/4 rules; roundup=3 | 70.0% | 71.4% | — | Score 3=83% but score 4=62% |
| **16** | **VFS_Analyst_v6** | **v5 score-2/3 + v3 score-4 + roundup exception** | **66.7%** | **82.9% ✅** | **86.1% ✅** | **TARGET REACHED** |

### Key Insight
Human annotators use a **financial analyst/investor perspective**:
- Score 4 covers both (A) direct Alibaba news AND (B) policy/market that creates favourable conditions for Alibaba's core segments
- Score 2 covers both (A) named Alibaba setbacks AND (B) regulations targeting the **e-commerce platform / online marketplace model** specifically (not general antitrust or supply chain)
- Score 3 is the default for general macro, broad antitrust, supply chain, and multi-company roundups where Alibaba is not the main subject

### VFS_Analyst_v6 Final Results
```
Val  (35 articles):  exact=82.9%  adj±1=100%  MAE=0.171  ρ=0.888
Test (36 articles):  exact=86.1%  adj±1=100%  MAE=0.139  ρ=0.904
```

**Test confusion matrix**:
```
pred  1   2  3  4
gold
1     1   0  0  0
2     0  13  3  0
3     0   0  9  1
4     0   0  1  8
```
Per-class (test): Score 1 → 100%, Score 2 → 81%, Score 3 → 90%, Score 4 → 89%

---

## Prompt Definitions

### 1. VFS_2per — Few-shot 2 per class

**Date**: 2026-05-14  
**Results**: Train 70.0%, adj±1 96.7%, MAE 0.333, ρ 0.705

**Strategy**: Simple few-shot with 2 randomly sampled examples per score class (10 total).
Basic one-line scale definition. No reasoning guidance.

**System prompt**:
```
You are an expert financial sentiment analyst specialising in Alibaba Group.
Rate news articles on a 1–5 sentiment scale:
1 = strongly negative  (major crisis, penalty, fraud, severe loss)
2 = negative           (competition pressure, disputes, margin squeeze, adverse development)
3 = neutral            (factual reporting, unclear direction, mixed signals)
4 = positive           (new product, market gain, improved outlook, favourable news)
5 = strongly positive  (exceptional growth, major strategic win, landmark breakthrough)

Below are EXAMPLES labeled by a human analyst. Follow the same annotation style and judgment criteria.

[10 few-shot examples — 2 per class, random selection from training set]

---
Return ONLY valid JSON: {"sentiment_score":<1-5>,"confidence":<0-1>,"key_factors":["..."],"reasoning":"..."}
```

---

### 2. VFS_3per — Few-shot 3 per class

**Date**: 2026-05-14  
**Results**: Train 63.3%, adj±1 96.7%, MAE 0.400, ρ 0.666

**Strategy**: Same as VFS_2per but 3 random examples per class (15 total). More examples hurt — likely
due to increased prompt length diluting attention on the scoring criteria.

**System prompt**: Identical to VFS_2per except 15 few-shot examples (3 per class).

---

### 3. VFS_CoT — Few-shot 2 per class + Chain-of-Thought *(Current best)*

**Date**: 2026-05-14  
**Results**: Train 73.3%, Val 62.9%, adj±1 97.1%, MAE 0.400, ρ 0.704

**Strategy**: 2 random examples per class + 4 explicit reasoning questions before scoring.
The CoT steps force the model to consider Alibaba's direct impact before assigning a score.

**System prompt**:
```
You are an expert financial sentiment analyst specialising in Alibaba Group.
Rate news articles on a 1–5 sentiment scale:
[same scale definition as VFS_2per]

Below are EXAMPLES labeled by a human analyst. Follow the same annotation style and judgment criteria.

[10 few-shot examples — 2 per class, random selection from training set]

---
Before assigning a score, reason through:
  1. What is the main event described?
  2. Does it directly affect Alibaba's revenue, market position, or reputation?
  3. Is the net impact positive, negative, or ambiguous?
  4. How would a financial investor react to this news?

Return ONLY valid JSON: {"sentiment_score":<1-5>,"confidence":<0-1>,"key_factors":["..."],"reasoning":"..."}
```

---

### 4. VFS_Boundary — Explicit boundary rules + Q1–Q3 checklist

**Date**: 2026-05-14  
**Results**: Train 66.7%, adj±1 96.7%, MAE 0.367, ρ 0.633

**Strategy**: 2 random examples + highly detailed scoring criteria with explicit ✗ NOT rules
and a 3-question decision checklist (Centrality / Specificity / Direction). Hypothesis: verbose
rules would help separate score 3 from 4. In practice the extra length hurt accuracy.

**System prompt**:
```
You are an expert financial sentiment analyst specialising in Alibaba Group.
Rate news articles on a 1–5 scale using the exact criteria below.

━━━ SCORING CRITERIA ━━━
1 = Strongly Negative
    • Direct punitive action against Alibaba: major regulatory fine/ban, criminal
      investigation, fraud disclosure, or catastrophic operational failure.

2 = Negative
    • A regulation or policy that DIRECTLY restricts Alibaba's specific business
      practices or demonstrably cuts its profit/revenue.
    • Alibaba loses significant market share to a named competitor.
    • A concrete, near-term cost or revenue hit is explicitly mentioned.
    ✗ NOT score 2 if Alibaba is merely one of many platforms mentioned with no
      specific Alibaba harm stated, or if impact is vague/speculative.

3 = Neutral
    • Alibaba is NOT the primary subject — mentioned only briefly or as one of
      several companies, with no specific Alibaba development described.
    • Industry/macro/policy trends that are only indirectly or long-term relevant
      to Alibaba, with no concrete near-term catalyst for Alibaba itself.
    • Mixed signals: both positive and negative elements roughly cancel out.
    • Regulatory news where Alibaba is implicitly affected but unnamed.
    ✗ NOT score 3 if a specific, named Alibaba development is described.

4 = Positive
    • Alibaba IS the primary subject (or a named Alibaba entity: Taobao, Tmall,
      Alibaba Cloud, Tongyi, etc.) with a specific, concrete positive catalyst:
      new product/service launch, AI or cloud milestone, earnings beat, analyst
      upgrade, strategic partnership, or measurable competitive gain.
    • Competitor weakness that DIRECTLY and specifically benefits Alibaba.
    ✗ NOT score 4 for vague 'long-term support' or industry tailwinds without
      a named, specific Alibaba catalyst.

5 = Strongly Positive
    • Exceptional, landmark achievement backed by quantifiable evidence:
      record-breaking revenue/growth figures, a landmark global partnership
      (e.g. IOC selection), or a viral consumer metric proving massive scale.
    • Earnings report with clearly above-expectations results AND specific numbers.
    ✗ NOT score 5 for generally good news — needs a clear 'landmark' or 'record' quality.

━━━ BOUNDARY DECISION CHECKLIST ━━━
Answer these 3 questions before scoring:
  Q1. Centrality: Is Alibaba the PRIMARY subject of this article, or just
      mentioned in passing / as one of many?
      → 'Mentioned in passing' → likely 3 (unless direct negative impact → 2)
  Q2. Specificity: Is there a SPECIFIC, NAMED development directly affecting
      Alibaba's revenue, profit, or market position?
      → 'No specific development' → likely 3
  Q3. Direction & Magnitude: Is the impact positive or negative, and how strong?
      → Specific positive → 4; landmark/record → 5
      → Specific negative → 2; catastrophic/punitive → 1

━━━ EXAMPLES (human-labeled) ━━━
[10 few-shot examples — 2 per class, random selection]

━━━ TASK ━━━
Step 1: Answer Q1-Q3 for the article.
Step 2: Assign the score that best matches the criteria above.
Step 3: Return ONLY valid JSON: {"sentiment_score":<1-5>,...}
```

---

### 5. VFS_Select — Boundary-aware selection + focused 3/4 rule + CoT

**Date**: 2026-05-15  
**Results**: Train 66.7%, adj±1 96.7%, MAE 0.367, ρ 0.582, Val —

**Strategy**: Two changes vs VFS_CoT:
1. **Boundary-aware few-shot selection**: instead of random sampling, deliberately pick
   - Score 3: articles that prominently mention Alibaba but are STILL 3 (indirect/macro) — teaches: "Alibaba mentioned ≠ score 4"
   - Score 4: articles with specific, named Alibaba catalysts — teaches: "concrete event = 4"
   - Score 2: articles where regulation directly targets Alibaba (not just the industry)
2. **One focused boundary rule** added to the prompt (concise, not verbose like VFS_Boundary)

**System prompt**:
```
You are an expert financial sentiment analyst specialising in Alibaba Group.
Rate news articles on a 1–5 sentiment scale:
[same scale definition as VFS_2per]

KEY RULE — Score 3 vs 4:
• Score 4 ONLY if Alibaba (or a named entity: Taobao, Tmall, Alibaba Cloud, Tongyi, Cainiao,
  Ele.me, Hema) is the PRIMARY subject AND there is a SPECIFIC, CONCRETE positive development
  (product launch, earnings beat, AI milestone, named partnership, measurable market gain).
• Score 3 if Alibaba is just one of many companies mentioned, or the positive impact is
  indirect / long-term / lacks a specific near-term catalyst for Alibaba itself.

EXAMPLES (human-labeled — score-3 examples are chosen to show cases where Alibaba appears
prominently but still scores 3 because the impact is indirect):

[10 boundary-aware few-shot examples — 2 per class, selected for boundary clarity]

---
Before assigning a score, reason through:
  1. Is Alibaba (or a named subsidiary) the PRIMARY focus of this article?
  2. Is there ONE specific, concrete event directly affecting Alibaba's revenue / market position?
  3. What score does this map to per the criteria above?

Return ONLY valid JSON: {"sentiment_score":<1-5>,"confidence":<0-1>,"key_factors":["..."],"reasoning":"..."}
```

---

### 6. VFS_SmartBody — Smart body extraction + 2/class + CoT

**Date**: 2026-05-15  
**Results**: Train 66.7%, adj±1 96.7%, MAE 0.400, ρ 0.629, Val —

**Root cause this addresses**: News roundup articles (财经早播/汇总) have titles mentioning Alibaba
but BODY text starting with completely unrelated content (e.g., Foreign Ministry statements about
Strait of Hormuz). With a plain 350-char body truncation, few-shot examples showed the LLM:
- Title: "[Broadcast] ... Alibaba's HappyHorse AI..."
- Body: "Foreign Ministry: Hopes relevant parties responsibly abide by ceasefire..."
- Score: 3, Reason: "Broader news mix dilutes catalyst effect"
→ The LLM cannot understand the reason because it can't see the news mix! This is actively
misleading few-shot supervision.

**Fix — `_smart_body()` function**:
```python
def _smart_body(text, intro_len=400, ali_window=250, max_len=700):
    intro = text[:intro_len]
    m = ALI_RE.search(text, intro_len)   # find Alibaba entity beyond intro
    if m:
        center = m.start()
        start  = max(intro_len, center - ali_window // 2)
        end    = min(len(text), start + ali_window)
        ali_snip = " [...] " + text[start:end]
    else:
        ali_snip = ""
    return (intro + ali_snip)[:max_len]
```

Applied at BOTH few-shot example construction AND inference time (consistent context).

**System prompt**:
```
You are an expert financial sentiment analyst specialising in Alibaba Group.
Rate news articles on a 1–5 sentiment scale:
[same scale definition]

KEY RULE — Score 3 vs 4:
• Score 4 ONLY if Alibaba (or named entity: Taobao, Tmall, Alibaba Cloud, Tongyi, Cainiao,
  Ele.me, Hema) is the PRIMARY subject AND there is a SPECIFIC, CONCRETE positive development
  (product launch, earnings beat, AI milestone, named partnership, measurable market gain or win).
• Score 3 if Alibaba is one of many companies in a news roundup, or the impact is indirect /
  long-term / lacks a specific near-term catalyst for Alibaba.

EXAMPLES (human-labeled — body excerpts show the Alibaba-relevant section
even when the article is a news roundup):

[10 smart-body few-shot examples — 2 per class, body shows Alibaba content]

---
Before assigning a score, reason through:
  1. Is Alibaba (or a named subsidiary) the PRIMARY focus of this article,
     or just one item in a broader news roundup?
  2. Is there ONE specific, concrete event directly affecting Alibaba's revenue
     or market position in the near term?
  3. What score does this map to per the criteria above?

Return ONLY valid JSON: {"sentiment_score":<1-5>,"confidence":<0-1>,"key_factors":["..."],"reasoning":"..."}
```

---

### 7. VFS_Flash — Swap to gemini-2.5-flash model

**Date**: 2026-05-15  
**Results**: Train 50.0%, Val —, ρ=NaN (degenerate — all predictions same value)

**Strategy**: Same system prompt as VFS_CoT but using `gemini-2.5-flash` instead of `gemini-2.5-flash-lite`. Hypothesis: larger model = better discrimination.  
**Outcome**: Total failure — ρ=NaN means all 30 training predictions were identical. The model collapsed to a single output value regardless of article content. Reverted to lite.

**System prompt**: Identical to VFS_CoT (same prompt text, different model passed to API).

---

### 8. VFS_TwoStep — Two-stage subject-then-score

**Date**: 2026-05-15  
**Results**: Train 76.7%, Val 60.0%, adj±1 97.1%, MAE 0.429, ρ=0.619

**Strategy**: First classify whether Alibaba is the "primary subject"; then apply different scoring criteria for primary vs. secondary articles.

**Why it failed**: The "primary subject" gate is too strict for the human annotation style. Human annotators rate policy/macro articles as score 4 even when Alibaba is not the primary subject — as long as Alibaba's business clearly benefits. The two-stage gate blocked too many legitimate score-4 articles.

**System prompt** (key excerpt):
```
You are a sell-side financial analyst covering Alibaba Group.
Rate news articles on a 1–5 sentiment scale.

STEP 1 — Is Alibaba the primary subject of this article?
  YES: Alibaba (or Taobao, Tmall, Alibaba Cloud, Tongyi, Cainiao, Ele.me, Hema) is the
       main focus — the article is primarily ABOUT an Alibaba development.
  NO:  Alibaba is mentioned in passing, as one of many companies, or the article
       is mainly about a macro/policy/industry trend.

STEP 2 — Assign sentiment score:
  If PRIMARY (YES):
    1/2 = negative/strongly negative direct impact on Alibaba
    3   = neutral — factual reporting, mixed signals, unclear direction
    4/5 = positive/strongly positive direct Alibaba development
  If SECONDARY (NO):
    Score 3 by default, UNLESS the macro/policy change has a clear, specific positive
    or negative effect on Alibaba's core business → then score 4 or 2

[10 few-shot examples — 2 per class, seed=42]

Return ONLY valid JSON: {"sentiment_score":<1-5>,...}
```

---

### 9. VFS_Hybrid — TwoStep + CoT merged

**Date**: 2026-05-15  
**Results**: Train 50.0%, Val —, ρ=0.561

**Strategy**: Attempt to combine the two-stage logic from VFS_TwoStep with CoT reasoning steps from VFS_CoT in a single prompt.  
**Outcome**: Degenerate — train accuracy dropped to 50%. The merged instruction structure confused the model, producing inconsistent outputs. Similar failure mode to VFS_Flash.

---

### 10. VFS_Refine34 — Score then refine 3/4 boundary

**Date**: 2026-05-15  
**Results**: Train 60.0%, Val 60.0%, adj±1 94.3%, MAE 0.457, ρ=0.666

**Strategy**: Two-pass scoring — assign initial score using CoT, then if score is 3 or 4, apply a targeted refinement asking whether it is "background context" (→ 3) or "actionable signal for Alibaba investors" (→ 4).

**Why it failed**: The refinement step still relies on "is Alibaba the dominant topic?" which misunderstands the human annotation style. adj±1 dropped to 94.3% (some errors are ±2), worse than other prompts.

**System prompt** (key excerpt):
```
SCORING PROCESS:
  Step 1: Assign initial score using 4-step CoT
  Step 2 (only if initial score is 3 or 4):
    Ask: Is this article background context for the broader market (→ score 3),
    or does it provide an actionable investment signal specifically for Alibaba (→ score 4)?
    Criteria for score 4: Alibaba IS the primary topic AND there is a concrete near-term catalyst.
    Criteria for score 3: General industry/macro/policy without Alibaba as primary focus.
```

---

### 11. VFS_Analyst — Financial analyst / investor perspective *(Key breakthrough)*

**Date**: 2026-05-15  
**Results**: Train 66.7%, Val 68.6%, adj±1 100%, MAE 0.314, ρ=0.754

**Key insight**: Reframed the task from "is Alibaba the primary subject?" to **"how would a sell-side analyst covering Alibaba rate this for investors?"**
- Score 4 explicitly covers TWO types: (Type A) direct Alibaba news AND (Type B) policy/market developments that create favourable conditions for Alibaba's core segments
- adj±1 jumped to 100% for the first time — the model understood rough direction perfectly

**Val per-class**: Score 2 → 67%, Score 3 → 75%, Score 4 → 69%, Score 5 → 0%

**System prompt**:
```
You are a sell-side financial analyst covering Alibaba Group.
Rate news articles on a 1–5 sentiment scale from an investor's perspective:
1 = strongly negative  (major crisis, severe penalty, catastrophic event directly involving Alibaba)
2 = negative           (development that A SELL-SIDE ANALYST would flag as a KEY RISK for Alibaba
                        investors: direct Alibaba setback, platform-specific regulation, named
                        competitor gain at Alibaba's expense)
3 = neutral            (background context, general industry trends, or news where Alibaba is one
                        of many — no specific near-term catalyst for Alibaba's outlook)
4 = positive           (development that A SELL-SIDE ANALYST would flag as a KEY POSITIVE for
                        Alibaba investors: direct Alibaba good news (Type A), OR policy/market
                        shift that CLEARLY and DIRECTLY benefits Alibaba's e-commerce, cloud, AI,
                        or logistics business (Type B indirect positive))
5 = strongly positive  (exceptional / landmark / record-breaking positive for Alibaba)

IMPORTANT: Score 4 does NOT require Alibaba to be the primary subject.
If a policy or market development creates a CLEAR FAVOURABLE CONDITION for Alibaba's
business, rate it 4 even if Alibaba is not named — as a savvy analyst would.

[10 few-shot examples — 2 per class, seed=42]

━━━ SCORING PROCESS ━━━
  1. What is the main development?
  2. Would a sell-side analyst put this in the KEY RISKS section? → likely score 2
  3. Would a sell-side analyst put this in the KEY POSITIVES section? → likely score 4
  4. If neither → score 3
  5. Is it exceptional/record-breaking? → score 5

Return ONLY valid JSON: {"sentiment_score":<1-5>,"confidence":<0-1>,"key_factors":["..."],"reasoning":"..."}
```

---

### 12. VFS_Analyst_v2 — Analyst + low detection threshold for 2/4

**Date**: 2026-05-15  
**Results**: Train 53.3%, Val 65.7%, adj±1 100%, MAE 0.343, ρ=0.766

**Strategy**: Lower the threshold for assigning scores 2 and 4 — include "maybe" cases.

**Val per-class**: Score 2 → 89%, Score 3 → **33%**, Score 4 → 85%  
**Why it failed**: Scores 2 and 4 individually improved, but **score 3 accuracy collapsed to 33%**. The lower threshold swept neutral articles into 2 or 4 whenever any minor positive or negative element was present. Fundamental trade-off between sensitivity and specificity for score 3.

---

### 13. VFS_Analyst_v3 — Analyst + "dominant theme" language

**Date**: 2026-05-15  
**Results**: Train 73.3%, Val 71.4%, adj±1 100%, MAE 0.286, ρ=0.808

**Strategy**: Require positive/negative signal to be the **DOMINANT THEME** before assigning score 4 or 2.

**Val per-class**: Score 2 → 78%, Score 3 → 58%, Score 4 → 85%  
**Remaining problem**: Score 3 accuracy dropped to 58% — 3 neutral articles called score 2 (broad criteria caught antitrust/supply-chain articles), 2 called score 4.

**Val confusion matrix**:
```
pred  2  3   4
gold
2     7  2   0
3     3  7   2
4     0  2  11
5     0  0   1
```

---

### 14. VFS_Analyst_v4 — v3 + strict "named Alibaba" for score 2

**Date**: 2026-05-15  
**Results**: Train 76.7%, Val 65.7%, adj±1 100%, MAE 0.343, ρ=0.752

**Strategy**: Fix 3→2 errors from v3 by requiring Alibaba to be specifically named, OR the rule to explicitly target e-commerce platform businesses.

**Val per-class**: Score 2 → **56%**, Score 3 → 58%, Score 4 → 85%  
**Why it failed**: Over-restricted score 2 — 4 true-negative articles now missed (called score 3). Many legitimate platform-regulation articles human annotators rate as 2 were incorrectly passed as 3. Score 2 accuracy crashed from 78% to 56%.

---

### 15. VFS_Analyst_v5 — Domain-specific calibration; roundup → 3

**Date**: 2026-05-15  
**Results**: Train 70.0%, Val 71.4%, adj±1 100%, MAE 0.286, ρ=0.786

**Strategy**: Domain-specific rules from error analysis of v3:
- Score 2: **platform/e-commerce regulation and cross-border e-commerce supervision** → 2 even without naming Alibaba
- Score 3: **broad antitrust**, supply chain, general logistics → 3, NOT 2
- Score 4: **AI infrastructure** (NVIDIA GTC) and **Chinese AI milestones** (domestic token volumes) → 4
- **Roundup articles** (Tech Weekly, AI Weekly) → score 3 unless Alibaba is primary subject

**Val per-class**: Score 2 → 78%, Score 3 → **83%**, Score 4 → **62%**  
**Why it partially failed**: The "roundup → always score 3" rule over-applied. 3 new 4→3 errors appeared as score-4 roundup articles (Tech Weekly Alibaba T-Head chip, AI Weekly Chinese token milestones, market briefing with Alibaba stock gains) were incorrectly downgraded to 3.

**Val confusion matrix**:
```
pred  2   3  4
gold
2     7   2  0
3     1  10  1
4     0   5  8
5     0   0  1
```

---

### 16. VFS_Analyst_v6 — v5 score-2/3 + v3 score-4 + roundup exception ✅ TARGET REACHED

**Date**: 2026-05-15  
**Results**: Train 66.7%, Val **82.9% ✅**, Test **86.1% ✅**, adj±1 100%, MAE(val)=0.171, ρ(val)=0.888

**Strategy**: Synthesised all lessons from 15 iterations:
1. **Score 2**: Keep v5's domain rule (platform/e-commerce regulation → 2, antitrust/supply-chain → 3)
2. **Score 4**: Keep v3's broad detection (AI infrastructure, Chinese AI ecosystem, both Type A + Type B)
3. **Roundup exception**: Roundup articles are NOT automatically score 3 — score 4 if a KEY ITEM is a meaningful Alibaba development (chip, AI model, investment, stock movement)
4. **Score 5**: Explicit guidance for Alibaba achieving market leadership in a major competitive event

**Val per-class**: Score 2 → 78%, Score 3 → 83%, Score 4 → 92%, Score 5 → 0%  
**Test per-class**: Score 1 → 100%, Score 2 → 81%, Score 3 → 90%, Score 4 → 89%

**Val confusion matrix**:
```
pred  2   3   4
gold
2     7   2   0
3     1  10   1
4     0   1  12
5     0   0   1
```

**Test confusion matrix**:
```
pred  1   2  3  4
gold
1     1   0  0  0
2     0  13  3  0
3     0   0  9  1
4     0   0  1  8
```

**System prompt** (full):
```
You are a sell-side financial analyst covering Alibaba Group.
Rate news articles on a 1–5 sentiment scale from an investor's perspective:
1 = strongly negative  (major crisis directly involving Alibaba)
2 = negative           (specific negative impact on Alibaba's business)
3 = neutral            (general context, mixed signals, or indirect connection)
4 = positive           (specific positive impact on Alibaba's business or outlook)
5 = strongly positive  (exceptional achievement or market-leadership win for Alibaba)

━━━ SCORE 2 CALIBRATION ━━━
Rate 2 (negative) when any of the following apply:
  a) Alibaba is NAMED as facing a penalty, investigation, revenue loss, or market setback
  b) NEW RULES specifically targeting e-commerce platforms, online marketplaces,
     cross-border e-commerce, or internet platform businesses — including
     NEW TAX/COMPLIANCE OBLIGATIONS on platform operators or merchants
  c) Competitor gains explicitly described as coming at Alibaba's expense
Rate 3 (NOT 2) for:
  • Broad antitrust enforcement affecting many industries (general fair-competition news)
  • Supply-chain security decrees, physical logistics rules, general trade regulations
  • Consumer protection or sector rules that do NOT target the platform/e-commerce model
  • Vague macro headwinds without direct platform-economy impact

━━━ SCORE 4 CALIBRATION ━━━
Rate 4 (positive) when any of the following apply:
  a) Direct Alibaba development: product launch, revenue growth, partnership, strategic win
  b) Policy explicitly supporting e-commerce consumption, cloud computing, AI infrastructure,
     or digital economy in a way that directly benefits Alibaba's revenue streams
  c) AI INFRASTRUCTURE: new GPU/chip generation announcements (e.g. NVIDIA GTC, new
     accelerator chips) — these directly expand Alibaba Cloud's GPU/AI service capacity
  d) CHINESE AI ECOSYSTEM news: domestic AI usage milestones (token call volumes,
     Chinese LLM rankings), Chinese tech companies (including Alibaba) investing heavily
     in AI infrastructure — Alibaba Tongyi/Qianwen is a TOP beneficiary
  e) Alibaba's senior leadership (Jack Ma, CEO) making a strategic move reported as
     significant standalone or headline news
  f) Multi-company tech roundup (Tech Weekly, AI Weekly, market briefing) where one of
     the KEY ITEMS is a MEANINGFUL Alibaba development (chip, AI model, investment,
     stock movement, subsidiary win)
Rate 3 (NOT 4) for:
  • Purely foreign AI company news (OpenAI, Google, Meta) with no Chinese AI angle
  • Broad economic growth or service-industry policies benefiting many sectors equally
  • Roundup articles where Alibaba receives only a BRIEF or PERIPHERAL mention
    alongside 5+ other companies with no significant Alibaba-specific item

━━━ SCORE 5 CALIBRATION ━━━
Rate 5 (strongly positive) when Alibaba achieves clear MARKET LEADERSHIP:
  • Alibaba's AI product dominates a major high-profile competitive event
    (e.g. winning AI red envelope war by volume, topping a national AI benchmark)
  • Record-breaking Alibaba financial results or landmark regulatory/competitive win

━━━ FEW-SHOT EXAMPLES ━━━
[10 few-shot examples — 2 per class, seed=42, body_len=350]

━━━ SCORING PROCESS ━━━
  1. What is the article's main development? (one sentence)
  2. Score-2 check: platform/e-commerce-specific rule, named Alibaba setback?
  3. Score-4 check: AI infrastructure, Chinese AI milestone, direct Alibaba positive,
     or roundup with meaningful Alibaba item?
  4. If neither → score 3; if one fires → 2 or 4
  5. If positive is EXCEPTIONAL (market leadership win) → score 5

Return ONLY valid JSON: {"sentiment_score":<1-5>,"confidence":<0-1>,
"key_factors":["..."],"reasoning":"..."}
```

---

## Workflow Reminder

**每次新 Prompt 实验后必须更新本文件（prompt.md）：**
1. 在 Results Summary 表中添加新行（序号、名称、策略、Train/Val/Test 结果）
2. 在 Prompt Definitions 末尾添加新章节，包含：
   - 日期、完整指标（exact/adj±1/MAE/ρ）
   - 策略描述（做了什么改变、为什么、结果如何）
   - 完整 System Prompt 文本
   - 混淆矩阵和 per-class 精度（如果有 val 结果）
3. 更新 Key Insight（如有新发现）
