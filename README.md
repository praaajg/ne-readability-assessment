# Nepali Textbook Readability Assessment

[![Tests](https://github.com/praaajg/ne-readability-assessment/actions/workflows/test.yml/badge.svg)](https://github.com/praaajg/ne-readability-assessment/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

There is no widely available readability tool for Nepali. This repo builds one —
not a single formula, but a **library of ten readability methods** for Nepali
school-textbook sentences (Grades 1–10), compared against grade level with
statistical discipline.

**Headline result:** gradient boosting over 30 script/morphology/coverage features
predicts grade at MAE 2.0888 (book-level 95% CI [1.73, 2.50]), numerically best but
statistically tied with NCI (2.10) and ridge (2.16); lasso (2.22) is placed among the weakest
formulas instead. Adjacent grades are statistically inseparable by *any* method
(best Cliff's δ = 0.21) — an honest limit, not a bug.

## Findings (test set; book-level CIs, 8 of 78 gaps significant)
| rank | candidate | MAE [95% CI] | Pearson r |
|---|---|---|---|
| 1 | gradient boosting | 2.09 [1.73, 2.50] | 0.32 |
| 2 | NCI (refit v1) | 2.10 [1.72, 2.54] | 0.26 |
| 3 | ARI analog | 2.13 [1.73, 2.59] | 0.22 |
| 4 | ridge | 2.16 [1.80, 2.56] | 0.25 |
| 5 | Dale–Chall analog | 2.15 [1.74, 2.62] | 0.16 |
| 12–13 | halant-ratio, matra-ease | 2.20–2.21 | ≈ 0 |

- Gradient boosting is numerically best and NCI ranks second; under book-level
  resampling only gaps against lasso and the tail stay significant, so most of
  the order is indicative; lasso trails into the formula tail instead.
- Methods form two near-orthogonal families: syllable/length vs word-coverage
  (Dale–coverage r = 0.9438 [0.9434, 0.9441], cross-family |r| ≤ 0.17) —
  combining them is why models win.
- Baseline-word coverage falls across grades (0.47 → 0.36, non-monotonic) while
  conjunct density rises ~80% (0.19 → 0.34); long-matra ratio is grade-flat, so the
  Hindi matra weight does not transfer to Nepali.
- Sentence-level grade prediction is intrinsically hard (every MAE ≈ 2.0–2.2, just
  0.12 above always predicting the mean grade);
  grade means separate cleanly while adjacent grades do not.
- Refit check (all weights re-estimated on textbook segments, no hand-graded data):
  nothing adopted — NRS/flesch/dale show no gain, fog/ari refits degrade, and the
  fresh NCI simplex argmax degrades too; shipped NCI v1 (held-out-confirmed) stays.

> **Ground truth stance.** No human-graded difficulty data exists for Nepali yet, so
> textbook-assigned grade is used everywhere as an explicitly labeled **working proxy**,
> never as claimed truth.

## Repo map
| Path | What lives there |
|---|---|
| `nepali_readability/` | Installable library: `script.py` (Devanagari analyzer), `halant.py` (virama-deletion family), `baseline_words.py` (baseline list + Hunspell morphology), `formulas.py` (10 methods, one signature), `scorer.py` (`score/compare/batch_score`), `metrics.py` (CIs, effect sizes, tests), `config.py` |
| `tests/` | Unit tests with hand-computed expectations (`pytest`) |
| `examples/` | Runnable usage demo with captured output |
| `data/` | `external/` baseline word list + ne_NP dictionary · `corpus/` merged-sentence snapshot · `processed/` result tables, frozen split and model files |

## The ten methods
Classic analogs (Flesch, Fog, ARI, Dale–Chall on a 1,653-word Grade 1–3 baseline list, train books only) ·
Nepali NRS/NGL (Hindi-calibrated weights verbatim as cited, uncalibrated v0) ·
NCI (Nepali-refit v1 weights, adopted) ·
Nepali-specific halant-ratio, baseline-coverage, matra-ease and conjunct-ease.
Each is implemented in `nepali_readability/formulas.py` with one shared signature;
per-grade summary stats are committed under `data/processed/`.
The trained models (ridge/lasso/gradboost) share the identical frozen split and metrics.

## Quick start
```python
from nepali_readability import ReadabilityScorer
rs = ReadabilityScorer()
r = rs.score("राम स्कुल जान्छ ।")
r["nrs"]["ease"]            # 53.5 — Nepali ease score (0–100, higher=easier)
r["ngl"]                   # {'grade': 4, 'band': 'Grade 3-5'} (Hindi-calibrated map)
r["hunspell_cov"]["ease"]  # baseline-word coverage as ease
rs.compare([text1, text2]) # easiest-first ranking
```
Full run with output: `examples/score_nepali.py` (see `examples/score_nepali_output.txt` —
easy sentence at NRS 94.3, conjunct-heavy at 3.1, publisher boilerplate at 0.0).
Full API: `nepali_readability/scorer.py`.

## The NRS formula

NRS, the Nepali Readability Score (0–100, higher = easier), adapts the Flesch
shape with two Devanagari-specific terms:

```
NRS = 206.0
      -  60.0 × (syllables per word)
      -   1.8 × (words per sentence)
      -  70.0 × (conjuncts per word)
      -   8.0 × (long matras / all matras)
```

| Coefficient | Term | Why |
|---|---|---|
| 206.0 | intercept | Flesch-family scale anchor |
| 60.0 | syllables/word | primary difficulty driver, as in English |
| 1.8 | words/sentence | secondary sentence-length effect |
| 70.0 | conjuncts/word | Devanagari-specific: virama-fused clusters (ज्ञ, क्ष, त्र) mark harder, Sanskrit-heavy vocabulary |
| 8.0 | long-matra ratio | heavy (*guru*) syllables read harder — though our data show this term carries no grade signal in Nepali, unlike Hindi |

v0 uses Hindi-calibrated weights verbatim (cited, uncalibrated for Nepali);
the Nepali refit found no gain for this form, so v0 ships. Companions:
NCI = 0.35·syllable + 0.35·sentence + 0.30·conjunct + 0.00·matra
(Nepali-refit v1, adopted; each 0–1; ease = 100·(1−NCI)), and
NGL = 10.083 − 0.1088·NRS mapping ease to grades 1–13 (CDC bands).

Worked example (real output):

```python
>>> r = rs.score("राम स्कुल जान्छ ।")
>>> r["nrs"], r["nci"], r["ngl"]
({'raw': 53.47, 'ease': 53.47}, {'raw': 0.32, 'ease': 68.08}, {'grade': 4, 'band': 'Grade 3-5'})
>>> r = rs.score("विद्यालयमा ज्ञान र विज्ञानको संरचना छ ।")
>>> r["nrs"], r["nci"], r["ngl"]
({'raw': 3.11, 'ease': 3.11}, {'raw': 0.39, 'ease': 60.98}, {'grade': 10, 'band': 'Grade 9-10'})
```

Interpretation: the plain three-word sentence lands mid-scale (NRS 53.5, grade 4);
the conjunct-dense sentence collapses to NRS 3.1 (grade 10) — conjunct density
doing exactly the work the 70.0 coefficient prices in. NCI agrees directionally
(68.1 vs 61.0) while compressing less, since it blends capped sub-scores
(the matra term refit to zero).

## Install and test
1. `python -m pip install -e .` (use `python -m pip` so the package lands in the same env as `python`)
2. Score text as in Quick start, or run `python examples/score_nepali.py`.
3. Run the suite: `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 python -m pytest tests/ -q` (33 tests).
4. Reproduce the analysis tables: `pip install pandas`, then `python rebuild.py`
   (~15–30 min; regenerates `features_v1/v2/v3` + all `scores_*`/`stats_*` from the
   committed corpus snapshot using only the installed library — the one documented
   deviation is empty `chapter_title`, which no live consumer reads). Model training
   needs scikit-learn (research environment); trained artifacts are committed.

The study behind this package merged 5,634 Hugging Face textbook segments
(`dineshkarki/nepali-textbooks-corpus`, Apache-2.0) with 42 raw OCR files into
288,115 deduplicated sentences (283,586 Grades 1–10 plus A-only 11–12); per-grade summary
stats, the frozen split and the trained model files are committed under `data/`.

## Relationship to `hindi-readability`
[`Erprabhat8423/hindi-readability`](https://github.com/Erprabhat8423/hindi-readability)
(PyPI `hindi-readability` 0.3.0) is a **structural reference only** — its package
layout and scorer API shape informed ours. No code is copied (source-similarity
audit: zero shared blocks ≥200 chars); its formula weights appear solely as cited,
uncalibrated starting points slated for refit. Full credits: `CREDITS.md`.

## Status, changelog & license
The study is complete; open review items (ground-truth scoping, anomaly
adjudications, calibration-adoption calls) await linguist/supervisor review.

### Changelog
- `v0.1.0` — initial release: 10-method library, frozen split + significance-backed
  ranking (gradboost MAE 2.09), 288k-sentence merged corpus, 33 tests passing, CI.

### Citation
```bibtex
@software{nepali_readability,
  author = {praaajg},
  title  = {ne-readability-assessment: Nepali Textbook Readability Assessment},
  year   = {2026},
  url    = {https://github.com/praaajg/ne-readability-assessment}
}
```

MIT License (see `LICENSE`; bundled ne_NP dictionary files carry
their own LGPL 2.1 terms).
