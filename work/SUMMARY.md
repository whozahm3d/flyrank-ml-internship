# Capstone Summary — AI Referral Opportunity Scoring

*Working reference for ML-11 (Ship the Paper) and ML-12 (Tell the Story). Not a submitted deliverable itself — pulls together findings scattered across the capstone notebook's 7 sections into one place.*

## Question

Among pages with strong organic search visibility, which ones resemble pages that already receive AI-referral sessions, and can they be ranked to beat a naive impressions-only rule? Framed as ranking/scoring throughout — never classification — since AI-referral sessions are consistently too sparse (0.06% daily → 1.15–2.20% monthly/3-month → 6.43% at 90 days in the starter sample) to support honest classifier evaluation.

## Data

- **Release:** `FlyRank/internship-warehouse` (Hugging Face), snapshot 2026-07-03.
- **Window:** `fact_content_daily_performance`, months `2026-01` to `2026-03` (25,087,303 rows), strictly before the sealed June `_sample` file to avoid leaking a future outcome window into development.
- **Eligible article set after filtering** (excluding deleted content, no-14-day-history articles): 266,175 articles, base rate 2.20% (5,851 with `sessions_ai > 0`).
- **Representativeness:** only 39 of 104 total clients (37.5%) reach the eligible set — mostly `gsc_and_ga4` clients whose data window overlaps Jan–Mar; findings describe this subset, not FlyRank's full client base.

## Methodology

- **Features (GSC-derived only, causally separate from the label):** `gsc_impressions_sum`, `gsc_avg_position_mean`, `word_count` (with `has_word_count` flag, no `fillna(0)`, per the ML-01 finding that missingness correlates with content type).
- **Label:** `sessions_ai_window > 0`.
- **Score:** within-client normalized composite (z-scored impressions + word count, computed *within* each client) — chosen over the raw cross-client composite after testing showed raw scoring lets high-volume clients dominate and actually underperforms the naive baseline.
- **Validation split:** client-grouped (not random-row) to prevent row-level leakage across train/test, further stratified by client label rate to reduce (not eliminate) a large train/test label-rate gap caused by one outlier client holding ~half of all dataset-wide positives.
- **Leakage check:** injecting `sessions_ai` itself as a feature produced 66.47x lift vs. the honest 8.76x (7.6x inflation) — confirms the evaluation pipeline correctly detects an obvious leak.

## Results

| K | Score | Precision@K | Lift |
|---|---|---|---|
| 500 | naive baseline | 10.0% | 7.18x |
| 500 | raw composite | 8.8% | 6.32x |
| 500 | **within-client** | **12.2%** | **8.76x** |

- Within-client score leads consistently across K=100–2000 and passes the leakage check.
- **However:** a paired bootstrap (client-level resampling, the correct unit of independence with only 8 test clients) gives a 95% CI on the win margin of **[-4.60pp, +3.20pp]** — includes zero. Only 58.5% of resamples showed within-client beating naive. Removing the single dominant client (46.0% of the top-500) drops precision below the naive baseline.
- **Honest conclusion: the within-client score is directionally promising and mechanistically sound (fixes a real concentration problem), but its edge over naive ranking is not statistically confirmed at this test-set size.**

## Limitations

- Base rate (2.20%, 3-month window) undershoots the 90-day reference (6.43%) — expected direction, not directly comparable in magnitude.
- Train/test label-rate gap only partially resolved (2.34%/1.39% after stratification) — structural ceiling from having only 39 clients, one of which holds ~half of all positives.
- Result not statistically distinguishable from naive baseline (see Results).
- **Two independently-discovered client-metadata staleness issues** (both distinct from ML-04's `access_profile` anomaly): 6 of 39 clients (42,734 articles) have zero GSC signal despite passing eligibility filters; one of those six has a `gsc_data_start` value that contradicts its complete absence of GSC data.
- 12 articles have real `sessions_ai` activity despite zero GSC impressions (4 also missing `word_count`) — direct evidence the two GSC-derived features cannot explain all AI-referral activity.
- No causal claims possible anywhere in this work — associations only.
- Single non-seasonal 3-month window; no trend or seasonality claims possible.

## Ranked Recommendations

- 335 `high_opportunity` candidates across 35 of 39 eligible clients (capped at 10/client to prevent concentration from dominating output), exported to `work/outputs/capstone_ranked_recommendations.csv`.
- Reason codes: `high_opportunity` (12,126 articles) / `established_ai_source` (5,839) / `insufficient_data` (42,734 — the 6 zero-GSC clients) / `low_signal` (205,476).
- Framed explicitly as candidates for human review, not validated predictions, given the Results section's significance finding.

## Presentation additions (not new analysis)
 
- **Chart-read captions** added under all figures in Results, Limitations, and Ranked recommendations — one line each, explaining what to look for.
- **Finding tags** (NUANCED / NEW FINDING) applied to the two most important qualifying results: the within-client score's unconfirmed win, and the two metadata-staleness discoveries.
- **References subsection** added to Section 8 (Artifacts/Reproducibility), linking ML-01 through ML-05 notebooks as the direct lineage this capstone builds on.
- These were deliberately kept as style/structure only — no new datasets, features, or claims were added to match the scope of the external FlyRank portfolio report reviewed for inspiration.


## ML-12 Status
 
- **Demo outline + shareable cuts:** done, committed as closing markdown cells in `capstone.ipynb`.
- **Case-study framing in Abstract:** done — added a sentence explicitly naming the real FlyRank problem (no existing tool for AI-referral content prioritization) directly in Section 1 (Abstract), not left implicit in Section 2 only.
- **Live deployment:** blocked on ML-11 — ML-12's deliverable requires the paper's case-study framing to be *live at the deployed URL*, so full submission waits until the paper is shipped.

## Key numbers for paper text (keep consistent)

- Eligible articles: 266,175 | Base rate: 2.20% (5,851 positive)
- Within-client lift@500: 8.76x | Precision@500: 12.2%
- Bootstrap CI on win margin: [-4.60pp, +3.20pp] | 58.5% of resamples favored within-client
- Representativeness: 39/104 clients (37.5%)
- insufficient_data: 42,734 articles / 6 clients
- Final recommendations: 335 candidates / 35 clients
