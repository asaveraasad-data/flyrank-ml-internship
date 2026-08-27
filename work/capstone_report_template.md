# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Asavera
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/asaveraasad-data/flyrank-ml-internship
- **Date:** 25 August 2026

## 1. Problem framing

### Decision supported

This project supports the decision of **which content pages should be reviewed first** when editorial capacity is limited.

The unit of analysis is an individual content item/page. The system produces a ranked list based on the likelihood that a content item belongs to the declining class, together with a final refresh score, confidence level, suggested action, and reason codes.

The human action is not automatic content modification. An editor uses the ranked queue to decide which pages deserve attention first, then reviews the page and its underlying signals before deciding whether a refresh or another intervention is appropriate.

The cost of a wrong call is asymmetric. Prioritizing a page that does not need attention can waste editorial time, while failing to surface a genuinely declining page can delay a potentially useful review. For this reason, the model is treated as **decision support rather than an automatic refresh system**.

ML is useful here because the dataset contains multiple observable search-performance and freshness signals. A learned ranking approach can combine these signals and prioritize a smaller review queue instead of relying on a single manual rule.

### Research question

**Can observable search-performance and content-freshness signals be used to identify which content items are most likely to be declining, so that FlyRank editors can prioritize which pages to review first?**

---

## 2. Data safety

The analysis uses the anonymized FlyRank content-performance dataset:

`data/raw/content_refresh_anonymized.csv`

The supplied dataset contains **30,000 content items and 44 columns**.

The available data includes search-performance, engagement, content characteristics, freshness, and trend-related signals.

The final model uses the following five features:

- `impressions_90d`
- `content_age_days`
- `avg_position`
- `ctr`
- `days_since_last_update`

Additional fields were available in the dataset but were not included in the final model.

### Deliberately excluded fields

`trend_direction` and `trend_pct` were excluded because they are used to derive the target. Including them would create label leakage.

`content_id` and `client_id` were not used as predictive features. `client_id` was used for client-level grouping during validation.

The target was defined as:

`is_declining_label = (trend_direction == "down")`

### Public-safety check

No client names, domains, page URLs, private search queries, credentials, or other client-identifying information are included in the analysis or public report.

The supplied CSV does not expose an explicit release identifier, table name, or calendar date field. Therefore, this report does not claim a specific release or calendar date window that cannot be verified from the available material.

---

## 3. Baseline

A transparent baseline ranking was established before evaluating the learned model.

The baseline provides a simple reference point for determining whether the Random Forest adds useful ranking performance beyond a non-learned approach.

The baseline and Random Forest were evaluated on the same ranking task using the same primary evaluation split.

### Primary evaluation results

| Metric | Baseline | Random Forest |
|---|---:|---:|
| PR-AUC | 0.5237 | 0.6064 |
| Precision@25 | 0.76 | 0.84 |
| Precision@50 | 0.66 | 0.76 |
| Precision@100 | 0.61 | 0.73 |

On the primary evaluation split, Random Forest improved PR-AUC by **0.0827** and Precision@25 by **0.08** compared with the baseline.

The target base rate in the 30,000-row dataset was:

- Declining: **16,262 / 30,000 (54.21%)**
- Not declining: **13,738 / 30,000 (45.79%)**

This base rate provides context for interpreting the ranking metrics.

---

## 4. Model / analysis

The final learned model is a **Random Forest classifier**.

The model predicts the declining class using five observable search-performance and freshness signals:

1. `impressions_90d`
2. `content_age_days`
3. `avg_position`
4. `ctr`
5. `days_since_last_update`

The target is defined in one sentence as:

> A content item is labeled declining when `trend_direction == "down"`.

The label-derived fields `trend_direction` and `trend_pct` were excluded from the predictive feature set to avoid leakage.

Pseudonymous identifiers were also excluded from the predictive feature set.

The model output is used for ranking rather than as a claim that a page will definitely decline.

The Week 7 action playbook then converts the ranked output into editorial recommendations using the model score, supporting signals, confidence, and reason codes.

---

## 5. Evaluation

### Validation design

The analysis uses **client-level grouped validation**.

The purpose of grouping by client is to avoid simply mixing pages from the same client across training and evaluation data.

The primary evaluation compares the Random Forest with the transparent baseline on the same ranking task.

The main metrics are:

- PR-AUC
- Precision@25
- Precision@50
- Precision@100

### Primary evaluation

On the primary evaluation split:

- Random Forest PR-AUC: **0.6064**
- Baseline PR-AUC: **0.5237**
- Random Forest Precision@25: **0.84**
- Baseline Precision@25: **0.76**

The model therefore performed better than the baseline on the primary evaluation split.

### Validation variability

Client-level validation showed that performance was not stable across every split.

| Split | Random Forest Precision@25 | Baseline Precision@25 |
|---|---:|---:|
| 1 | 0.84 | 0.76 |
| 2 | 1.00 | 0.76 |
| 3 | 0.48 | 0.80 |
| 4 | 0.60 | 0.32 |
| 5 | 0.68 | 0.28 |

Random Forest Precision@25 ranged from **0.48 to 1.00**, while the baseline ranged from **0.28 to 0.80**.

This means the Random Forest was not consistently better than the baseline on every client-level split. The variation is important when interpreting the primary result.

### Error / uncertainty interpretation

The main observed uncertainty is variation across client-level validation splits. A model that performs strongly on one client partition can perform substantially differently on another.

Therefore, the model should not be treated as guaranteed to outperform the baseline for every client or future dataset.

---

## 6. Interpretation

The analysis found that a combination of search-performance and freshness signals can produce a useful ranking of content items for editorial review.

The final model focuses on:

- recent search visibility through `impressions_90d`
- content age through `content_age_days`
- search position through `avg_position`
- click capture through `ctr`
- freshness through `days_since_last_update`

The final recommendation layer also produces reason codes that make the ranking more interpretable. Examples of observed reason-code patterns include declining demand, low CTR on visible pages, moderate CTR weakness, model decline risk, and low engagement on visible pages.

The strongest practical interpretation is therefore not:

> "The model knows which pages need refreshing."

Instead, the supported interpretation is:

> "The model can help rank pages for human editorial review using observable search-performance and freshness signals."

A significant negative result is the variability across client-level validation splits. The model's primary-split improvement should therefore not be generalized into a guarantee of consistent future performance.

---

## 7. Recommendation

The final Week 7 action playbook produced a ranked queue of **200 pages**.

### Action distribution

| Suggested action | Pages | Share |
|---|---:|---:|
| Refresh + CTR Review | 130 | 65.0% |
| Refresh | 35 | 17.5% |
| Refresh + Engagement Review | 35 | 17.5% |
| **Total** | **200** | **100%** |

### Confidence distribution

| Confidence | Pages | Share |
|---|---:|---:|
| High | 161 | 80.5% |
| Medium | 39 | 19.5% |
| Low | 0 | 0.0% |
| **Total** | **200** | **100%** |

The average final refresh score in the queue was **77.949**, and the average model probability was **0.802**.

### Editorial workflow

A FlyRank editor could use the output as follows:

1. Start with the highest-ranked pages.
2. Review the model reason codes.
3. Inspect the underlying search-performance and freshness signals.
4. Manually evaluate search intent, content quality, factual accuracy, and whether the page actually requires a refresh.
5. Decide whether to refresh, review further, or leave the page unchanged.
6. Monitor the page after any editorial intervention.

The highest-confidence use of the system is **prioritization**, not automatic editorial decision-making.

---

## 8. Reproducibility

The project is organized as a sequence of notebooks covering the research question, task framing, data and leakage checks, baseline development, modeling, validation, and the final action playbook.

The main notebooks are:

- `work/notebooks/w01_research_question.ipynb`
- `work/notebooks/w02_ml_task_framing.ipynb`
- `work/notebooks/w03_data_contract.ipynb`
- `work/notebooks/w03_feature_leakage_check.ipynb`
- `work/notebooks/w04_baseline_score.ipynb`
- `work/notebooks/w04_signal_audit.ipynb`
- `work/notebooks/w05_model.ipynb`
- `work/notebooks/w06_validation_audit.ipynb`
- `work/notebooks/w07_action_playbook.ipynb`
- `work/notebooks/capstone.ipynb`

The capstone notebook brings the final research question, methodology, evaluation results, validation audit, and recommendation output together.

The source dataset is:

`data/raw/content_refresh_anonymized.csv`

The reported workflow includes the final feature set, target definition, baseline comparison, client-level validation results, and ranked recommendation output.

### Re-run note

The available capstone material does not document a single random seed or a pinned environment specification. Therefore, this report does not claim bit-for-bit deterministic reproduction.

The notebooks provide the inspectable analysis workflow and the reported evaluation outputs.

---

## Claims checklist

The analysis uses careful decision-support language throughout.

The results are described as:

- observed
- measured
- directional
- decision-support

The project does not claim:

- that the model predicts Google's ranking algorithm
- that a page will definitely decline
- that refreshing a page will cause traffic or ranking improvements
- that the model will outperform the baseline for every client
- that the model automatically determines what an editor should change

The ranked output should therefore be interpreted as a prioritization aid requiring human review.
