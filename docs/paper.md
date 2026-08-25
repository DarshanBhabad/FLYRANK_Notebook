# Ranking Refresh Opportunities — Capstone Paper

## Abstract

This project builds a practical ranked review engine that helps editors prioritize which content pages to review for refresh or metadata fixes. Using the FlyRank internship starter dataset (30,000 anonymized content rows) I frame the decision, build a transparent baseline score, and compare simple learned models to the rule baseline. The best models and the rule are validated with client-aware splits and top-K metrics (precision@K) so the output maps directly to an editor's capacity. The repo contains reproducible notebooks and the ranked action queue; a public-safe playbook turns model outputs into human review actions.

## Introduction / Problem statement

Decision supported: which pages should a content editor review first (refresh, title/meta rewrite, monitor, or prune) given limited review capacity. The actor is the content/editor team; the action is a human review and targeted update. A wrong recommendation wastes editorial time (false positives) or misses high-impact pages (false negatives). The goal is a ranked queue that maximizes precision@K for the top pages the team can realistically review.

## Data

- Starter slice used here: `data/raw/content_refresh_anonymized.csv` (30,000 rows × 44 columns). This public starter snapshot is included in the repository. The full warehouse release is `FlyRank/internship-warehouse` on Hugging Face (gated). See docs/data-dictionary.md and skills/flyrank/flyrank-data/SKILL.md for column definitions and gotchas.\n
- Date windows and tables used: starter CSV is a snapshot with trailing 90-day aggregates; full-work (not required for publication) runs on `fact_content_daily_performance` partitioned by month (e.g. `month=2026-03`) for production-scale validation.\n
- Exclusions: I deliberately exclude product decision flags and any derived pipeline outputs like `priority_score` or `health_score`. I do not use `trend_direction` or `trend_pct` as features because they are label-derived and cause leakage. This repo follows the public-safe rule: no client names, domains, raw URLs, or private queries appear in the write-up.

## Methodology

Lane chosen: Refresh / Content Opportunity Scoring — a ranked action queue with reason codes.\n
Features (examples used in the notebooks): impressions_90d (visibility), ctr, avg_position, days_since_last_update (freshness), word_count, ai_sessions_90d (sparse). Each feature is justified as "knowable at decision time" (they are historical aggregates).\n
Label / proxy: the starter target in the teaching pipeline is `is_declining_label` (derived from `trend_direction == 'down'`), which the starter notebooks use to demonstrate the pipeline. For honest future-window prediction work you must build a prior-window -> future-window label and enforce leakage control; this paper uses the starter workflow and emphasizes the leakage controls and why `trend_*` fields are excluded from training.\n
Baseline: a transparent rule-based baseline score (described in work/notebooks/w04_baseline_score.ipynb) that combines visibility, freshness risk, and a simple position opportunity term into a single score and emits reason codes such as `stale_visible_page` and `low_ctr_visible_page`. The baseline writes `work/outputs/baseline_action_score.csv` on each run.\n
Validation: client-holdout splits and precision@K for a realistic editor capacity (top-50, top-100) are used. The repo's notebooks demonstrate client-level grouping for train/test splits and top-K reviews of the ranked queue. Leakage checks remove `trend_pct`/`trend_direction` from the feature set and demonstrate the leaky AUC jump in a controlled demo (w03 feature-leakage-check).

## Results (summary)

- Starter dataset: 30,000 rows; is_declining_label prevalence ≈ 54% (16,262 rows in the prepared starter feature vector).\n
- The teaching materials and starter run show baseline and model numbers on this slice (reference numbers from the starter outputs): logistic regression ROC AUC ≈ 0.70, random forest ROC AUC ≈ 0.75, precision@50 improvements over the baseline rule were observed in the starter pipeline (see docs and outputs/model_report.md in the repo).\n
- In this capstone, the rule baseline is computed in `work/notebooks/w04_baseline_score.ipynb` and a ranked CSV is written for reviewer inspection. The top-K review and manual checks are recorded in the notebook (what would make each pick wrong). The honest leakage demo is shown in `work/notebooks/w03_feature_leakage_check.ipynb`.

## Limitations & honest framing

- Observational only: this analysis is decision-support and observational. It cannot prove that an editorial refresh will cause traffic recovery — proving causation requires an experiment.\n
- Data limits: the warehouse is an unbalanced panel (clients have different history depth). GA4 availability differs per client and early history may be GSC-only. These constraints require per-client windows and volume filters.\n
- Sparse signals: AI referral signals are sparse across the warehouse, and any analysis on them should be treated as EDA and ranking only.

## Ranked recommendations (action playbook)

- Output format: a ranked CSV with `content_id`, `client_id`, `baseline_score`, `reason_code`, and `action`. Reason codes include `stale_visible_page` (suggest: review_refresh), `low_ctr_visible_page` (suggest: review_ctr), and `monitor`.\n
- Use case: editors act on the top-K list each review cycle; precision@K is the operational metric to optimize. The notebook shows how to compute precision@K and select a K that matches reviewer capacity.

## Reproducibility

All notebooks used to generate numbers, run baselines, and test leakage are in `work/notebooks/` in this repo. Key notebooks: \n
- work/notebooks/w01_research_question.ipynb\n- work/notebooks/w02_ml_task_framing.ipynb\n- work/notebooks/w03_data_contract.ipynb\n- work/notebooks/w03_feature_leakage_check.ipynb\n- work/notebooks/w04_baseline_score.ipynb\n\nRun `python scripts/run_all.py` for the starter pipeline (the repo includes scripts that reproduce the basic pipeline and outputs). The full-warehouse experiments require gating to the Hugging Face release and a READ token (HF_TOKEN).

## Acknowledgments & data credit

Built on the FlyRank ML Internship dataset — https://flyrank.ai

---

This paper and all code are public-safe: no client names, domains, raw URLs, private queries, or credentials are included.
