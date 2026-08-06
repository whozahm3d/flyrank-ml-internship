# Capstone Summary — AI Referral Opportunity Scoring

*Working reference for ML-11 (Ship the Paper) and ML-12 (Tell the Story). Not a submitted deliverable itself — pulls together findings scattered across the capstone notebook's 9 sections into one place.*

**Notebook structure (post-renumbering):** 1. Abstract → 2. Question → 3. Data → 4. Methodology → 5. Results (vs. baseline) → 6. Limitations → 7. Ranked recommendations → 8. Artifacts / Reproducibility (includes References subsection) → 9. Acknowledgments & Data Credit. Closing cells below Section 9 hold the ML-12 deliverables (5-minute demo outline + shareable cuts).

## Question

Among pages with strong organic search visibility, which ones resemble pages that already receive AI-referral sessions, and can they be ranked to beat a naive impressions-only rule? Framed as ranking/scoring throughout — never classification — since AI-referral sessions are consistently too sparse (0.06% daily → 1.15–2.20% monthly/3-month → 6.43% at 90 days in the starter sample) to support honest classifier evaluation.

## Data

- **Release:** `FlyRank/internship-warehouse` (Hugging Face), snapshot 2026-07-03.
- **Window:** `fact_content_daily_performance`, months `2026-01` to `2026-03` (25,087,303 rows), strictly before the sealed June `_sample` file to avoid leaking a future outcome window into development.
- **Eligible article set after filtering** (excluding deleted content, no-14-day-history articles): 266,175 articles, base rate 2.20% (5,851 with `sessions_ai > 0`).
- **Representativeness:** only 39 of 104 total clients (37.5%) reach the eligible set — mostly `gsc_and_ga4` clients whose data window overlaps Jan–Mar; findings describe this subset, not FlyRank's full client base.
- **`word_count` coverage:** 69.25% of eligible articles — stable with the March-only figure (67.59%), confirming this is a structural gap in `dim_content`, not a month-specific artifact. `has_word_count` flag used rather than `fillna(0)`.
- **GA4 access exclusion:** 50.28% of raw rows have no GA4 access across the 3-month window (vs. 30.7% in March alone) — the eligible pool skews toward clients with earlier `ga4_data_start`, not a random sample.

## Methodology

- **Features (GSC-derived only, causally separate from the label):** `gsc_impressions_sum`, `gsc_avg_position_mean`, `word_count` (with `has_word_count` flag, no `fillna(0)`, per the ML-01 finding that missingness correlates with content type).
- **Label:** `sessions_ai_window > 0`.
- **Score:** within-client normalized composite (z-scored impressions + word count, computed *within* each client) — chosen over the raw cross-client composite after testing showed raw scoring lets high-volume clients dominate and actually underperforms the naive baseline.
- **Client concentration (ML-02's finding, re-tested at capstone scale):** raw composite score had one client holding 60.6% of train top-500 / 92.4% of test top-500. Within-client normalization dropped this to 20.8% (train) / 46.0% (test), with more unique clients represented in both — real mitigation, not a full fix (test set only has 8 clients total).
- **Feature correlation:** `z_word_count` vs. `z_impressions`, 0.127 — word count adds modest, not dominant, signal (417/500 overlap between impressions-only and composite-score top-500), matching ML-01's original "weak word count difference" finding.
- **Validation split:** client-grouped (not random-row) to prevent row-level leakage across train/test, further stratified by client label rate to reduce (not eliminate) a large train/test label-rate gap caused by one outlier client holding ~half of all dataset-wide positives.
- **Leakage check:** injecting `sessions_ai` itself as a feature produced 66.47x lift vs. the honest 8.76x (7.6x inflation) — confirms the evaluation pipeline correctly detects an obvious leak.

## Results

| K | Score | Precision@K | Lift |
|---|---|---|---|
| 500 | naive baseline | 10.0% | 7.18x |
| 500 | raw composite | 8.8% | 6.32x |
| 500 | **within-client** | **12.2%** | **8.76x** |

- **Evaluated on:** the test split only — 40,631 articles, 8 clients, 1.39% label rate. Train was used solely to build z-score normalization statistics.
- Within-client score leads consistently across K=100–2000 and passes the leakage check. Margin is largest at low K (12.2x at K=100 vs. naive's 7.9x) and narrows steadily, effectively tying by K=2000 (4.45x vs. 4.49x) — the expected shape for a genuinely informative score.
- **However:** a paired bootstrap (client-level resampling, the correct unit of independence with only 8 test clients) gives a 95% CI on the win margin of **[-4.60pp, +3.20pp]** — includes zero. Only 58.5% of resamples showed within-client beating naive. Removing the single dominant client (46.0% of the top-500) drops precision below the naive baseline.
- **Honest conclusion: the within-client score is directionally promising and mechanistically sound (fixes a real concentration problem), but its edge over naive ranking is not statistically confirmed at this test-set size.**

## Limitations

- Base rate (2.20%, 3-month window) undershoots the 90-day reference (6.43%) — expected direction, not directly comparable in magnitude.
- Train/test label-rate gap only partially resolved (2.34%/1.39% after stratification) — structural ceiling from having only 39 clients, one of which holds ~half of all positives.
- **Within-client score beats naive baseline — NUANCED.** Consistent directionally across K=100–2000, but not statistically confirmed: a paired bootstrap (client-level resampling) gives a 95% CI of [-4.60pp, +3.20pp] on the win margin — includes zero, only 58.5% of resamples favored the scored approach (see Results).
- **Two client-metadata staleness issues — NEW FINDING, not anticipated at lane-lock.** Distinct from ML-04's `access_profile` anomaly, found independently while diagnosing Section 7's (formerly Section 6's) `insufficient_data` clients: 6 of 39 eligible clients (42,734 articles) have zero GSC signal (`gsc_data_available = False`) across the entire window, despite passing Section 3's eligibility filters; one of those six also has a `gsc_data_start` value that contradicts the complete absence of GSC data.
- 12 articles have real `sessions_ai` activity despite zero GSC impressions (4 also missing `word_count`) — direct evidence the two GSC-derived features cannot explain all AI-referral activity.
- No causal claims possible anywhere in this work — associations only.
- Single non-seasonal 3-month window; no trend or seasonality claims possible.

## Ranked Recommendations

- 335 `high_opportunity` candidates across 35 of 39 eligible clients (capped at 10/client to prevent concentration from dominating output), exported to `work/outputs/capstone_ranked_recommendations.csv`.
- Reason codes: `high_opportunity` (12,126 articles) / `established_ai_source` (5,839) / `insufficient_data` (42,734 — the 6 zero-GSC clients) / `low_signal` (205,476).
- Framed explicitly as candidates for human review, not validated predictions, given the Results section's significance finding.
- **Correction made during build:** an earlier version of `high_opportunity` let 60 zero-impression articles through (15.8% of the initial 380-candidate list) — z-scored word count alone was enough to clear the threshold even with no search visibility, contradicting the lane's own premise (candidates must already have visibility). Fixed by requiring `gsc_impressions_sum` above the client's own median before qualifying.
- **"How to use this list" subsection** (Why/How/Expected/Measure format): (1) review `high_opportunity` candidates for restructuring — explicitly caveated as not a guaranteed lift, given the unconfirmed significance result; (2) treat `established_ai_source` pages as reference examples for that client, not action items.

## Presentation additions (not new analysis)

- **Chart-read captions** added under all figures in Results, Limitations, and Ranked recommendations — one line each, explaining what to look for.
- **Finding tags** (NUANCED / NEW FINDING) applied to the two most important qualifying results: the within-client score's unconfirmed win, and the two metadata-staleness discoveries.
- **References subsection** added to Section 8 (Artifacts/Reproducibility), linking ML-01 through ML-05 notebooks as the direct lineage this capstone builds on.
- These were deliberately kept as style/structure only — no new datasets, features, or claims were added to match the scope of the external FlyRank portfolio report reviewed for inspiration.

## ML-12 Status

- **Demo outline + shareable cuts:** done, committed as closing markdown cells in `capstone.ipynb`.
- **Case-study framing in Abstract:** done — added a sentence explicitly naming the real FlyRank problem (no existing tool for AI-referral content prioritization) directly in Section 1 (Abstract), not left implicit in Section 2 only.
- **Live deployment:** blocked on ML-11 — ML-12's deliverable requires the paper's case-study framing to be *live at the deployed URL*, so full submission waits until the paper is shipped.

## Open items before ML-11 (from notebook review)

- **Stale cross-references, 3 spots** — pre-renumbering section numbers leftover in prose: Methodology's closing cell says score "carried into Section 4" (self-referential; should say Section 5); Results' concentration check says "matching Section 3's concentration finding" (should be Section 4, where concentration was actually measured); Ranked Recommendations' final tally cites "Section 2 addendum" for the zero-GSC finding (should be Section 3 or Section 6).
- **Leftover draft cell** — an early, unheadered version of "How to use this list" (written before the visibility-floor fix, references "next quarter's data release") duplicates the corrected, headered version later in the same section. Delete the earlier one.
- **Figure attribution mismatch** — `capstone_precision_ci_comparison.png` is tagged "(Section 5)" in the Artifacts list, but the cell that generates it (and its chart-read) physically sits inside Section 6 Limitations. Decide: relabel the citation to Section 6, or move the cell up into Section 5.

## Key numbers for paper text (keep consistent)

- Eligible articles: 266,175 | Base rate: 2.20% (5,851 positive)
- Within-client lift@500: 8.76x | Precision@500: 12.2%
- Bootstrap CI on win margin: [-4.60pp, +3.20pp] | 58.5% of resamples favored within-client
- Representativeness: 39/104 clients (37.5%)
- insufficient_data: 42,734 articles / 6 clients
- Final recommendations: 335 candidates / 35 clients
- `word_count` coverage: 69.25% | Test split: 40,631 articles / 8 clients / 1.39% label rate
- Concentration: raw composite 92.4% (test top-500) vs. within-client 46.0% (test top-500)
