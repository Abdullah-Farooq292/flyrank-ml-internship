# Capstone Report — Content Opportunity Scoring

- **Author:** Abdullah Farooq
- **Lane:** Lane 2 — Refresh / Content Opportunity Scoring
- **Repo:** github.com/Abdullah-Farooq292/flyrank-ml-internship
- **Date:** [fill in submission date]

---

## 1. Problem framing

A content team cannot manually check every page in a large portfolio for signs of decline. This work supports the decision of which pages a reviewer should look at first, out of hundreds or thousands, when their time is limited.

**Unit of analysis:** one row = one content page, for one client, on one day.

**Output:** a ranked score per page (decline_score), with a plain-language reason code explaining why it was flagged.

**Action a human takes:** a content strategist reviews the top of the ranked queue and decides whether to refresh, expand, or leave a page alone.

**Cost of a wrong call:** flagging a healthy page wastes reviewer time on a page that didn't need attention. Missing a genuinely declining page means a lost opportunity to catch and fix a problem before it costs more traffic or visibility.

**Why ML helps here:** a fixed rule (e.g. "stale AND still getting traffic") only combines a couple of signals in a hand-chosen way. A trained model can weigh several signals together and find combinations a human wouldn't think to check — while still being interpretable enough to explain each flag.

---

## 2. Data safety

**Data used:** FlyRank's anonymized content-performance export (`data/raw/content_refresh_anonymized.csv`), ~30,000 active content pages. Features used: `content_age_days`, `days_since_last_update`, `avg_position`, `ctr`, `word_count`.

**Deliberately excluded:**
- `impressions_90d` — evaluated as a candidate feature, but excluded after a leakage audit found its 90-day window overlaps the label's own most-recent 30 days. Correlation with the label was near-zero (-0.018), and removing it did not hurt held-out precision.
- `trend_direction` and `trend_pct` — these are the source of the label itself (`is_declining_label`), not features. Using them as inputs would be direct label leakage.
- Any internal FlyRank product decision flags (health score, priority score, action type) — not present in this export, and would count as leakage if reconstructed, since they represent a decision already made downstream.

**Pseudonymous IDs:** `client_id` is used only to define the train/test split boundary (grouping), never passed to the model as a feature.

**Confirmation:** no client names, domains, URLs, or private queries appear anywhere in `work/` — verified via manual review of all notebook markdown and printed outputs before each commit.

---

## 3. Baseline

**Rule:** `stale_and_visible_score` — pages with `days_since_last_update >= 180` AND `impressions_90d >= 500`, scored proportional to traffic.

**Why it's a fair comparison:** it's evaluated on the identical held-out, client-grouped test set as the model, using the same metric (Precision@K).

**Validated on its own first:** among pages matching "stale + visible," 94.1% were genuinely declining, vs. 54.2% among all other pages — confirming the rule captures a real signal before using it as a baseline.

**Baseline numbers (honest, client-grouped split):** Precision@20: 0.500, Precision@50: 0.620.

---

## 4. Model / analysis

**Method:** Decision Tree classifier (`max_depth=4`, `class_weight="balanced"`). Chosen for interpretability — a decision-support tool needs a traceable "why" a page was flagged, not a black-box score.

**Feature list (final):** `content_age_days`, `days_since_last_update`, `avg_position`, `ctr`, `word_count`.

**Left out on purpose:** `impressions_90d` (leakage risk, see Data safety); all engagement/session features (GA4 data covers only ~4% of the full warehouse, too sparse for reliable use).

**Target / proxy, in one sentence:** `is_declining_label = 1` when a page's `trend_direction` reads "down" — a 30-day-vs-prior-30-day impressions trend bucket, used as a proxy for "worth reviewing now" rather than a confirmed future outcome.

---

## 5. Evaluation

**Split:** client-grouped (`GroupShuffleSplit` on `client_id`, 75/25 train/test), not random and not time-aware. Client-grouping was chosen because pages from the same client share structural similarities (site structure, content strategy); a random split risks the model memorizing client identity rather than learning a generalizable pattern. Zero client overlap was confirmed between train and test sets.

**Base rate:** 54.2% of pages outside the baseline's "stale + visible" bucket are still labeled declining — meaning the task's overall base rate is high (well over half of all pages are labeled "down" portfolio-wide: 16,262 of ~30,000). This matters for interpreting precision: a naive "flag everything" strategy would already score close to the base rate, so precision gains need to be read against that, not against zero.

**Metrics, model vs. baseline, same split:**

| Metric | Baseline | Model |
|---|---|---|
| Precision@20 | 0.500 | **0.650** |
| Precision@50 | **0.620** | 0.600 |

**Error analysis:** the model and baseline disagree in an informative way. There are 2,896 held-out pages where the model is confident (probability > 0.7) but the baseline scores zero (fails its stale+visible threshold) — of those, 58.6% are genuinely declining. This shows the model catches a real pattern the single-threshold baseline structurally cannot see, since the baseline only fires on one specific combination of signals (staleness + high traffic). Conversely, the baseline's edge at Precision@50 suggests that once the net widens, the simple "stale + visible" heuristic remains a strong, low-complexity signal on its own — the model's added flexibility doesn't uniformly help at every K.

A separate check (random split vs. grouped split) showed the random split overstated Precision@20 by up to 25 points relative to the honest grouped split — direct evidence that ungrouped validation would have produced a misleadingly optimistic result here.

---

## 6. Interpretation

**What the model found:** the tree's decision paths lean most heavily on `avg_position` first, then combinations of `content_age_days`, `ctr`, and `days_since_last_update`. In plain words: pages ranking poorly, combined with either being old or having weak click-through, are the strongest predictors of decline in this dataset — more so than staleness alone, which is what the baseline relies on exclusively.

**Surprises / negative results:** the baseline beating the model at Precision@50 was not the expected outcome (a trained model is often assumed to strictly beat a hand-written rule) — this is reported as a genuine, well-understood negative result rather than smoothed over: neither method dominates, and that's an honest finding, not a failure.

Because the tree has only `max_depth=4`, many pages land in the same leaf and receive an identical score — a real limitation, not a bug, and it's why the action queue also uses a secondary tiebreak plus per-page decision-path detail in its reason codes.

---

## 7. Recommendation

**Ranked actions a FlyRank editor would use tomorrow:**
- **High decline score + stale:** refresh candidate — update facts/examples, resubmit for recrawl.
- **High decline score + weak position, recently updated:** likely a relevance issue, not freshness — review content-market fit before refreshing again.
- **High decline score + thin content:** expansion candidate — add missing subtopics/depth.
- **Low decline score, strong position:** leave alone; monitor only.

**How to use it:** an editor pulls the top 20-50 rows of the ranked queue, checks each page's reason code, confirms against live data, and decides an action — never automating a change directly from the score.

**Confidence and limits, stated explicitly:** this is a decision-support shortlist with moderate precision (Precision@20: 0.650, meaning roughly 1 in 3 flagged pages in the top 20 will not actually be declining). It is validated on this portfolio only, on a single historical snapshot, and makes no causal or algorithmic claims. See the full Limitations section in the deployed paper for the complete list.

---

## 8. Reproducibility

**To re-run from a fresh clone:**
```bash
git clone https://github.com/Abdullah-Farooq292/flyrank-ml-internship
cd flyrank-ml-internship
pip install pandas scikit-learn numpy matplotlib
```
Then open and run (in order): `work/notebooks/w06_validation_audit.ipynb`, `work/notebooks/w07_action_playbook.ipynb`, `work/notebooks/capstone.ipynb`.

**Random seed:** `random_state=42` used consistently across all splits and model training, for reproducible results.

**Environment:** Google Colab default runtime (Python 3.12), standard `pandas`/`scikit-learn`/`numpy`/`matplotlib` — no non-default package versions required.

---

## Claims checklist before submitting
- [x] Language throughout uses observed / measured / directional / decision-support — no causal claims without an experiment, no claims about predicting Google's algorithm.
- [x] Metrics reported alongside base rate (Section 5) — not precision in isolation.
- [x] No client-identifying details anywhere in `work/`.
- [ ] Numbers in this report match a fresh re-run — confirm by re-running Section 5's cell before final submission.
