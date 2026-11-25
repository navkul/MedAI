# ATI Modeling Experiments — Boosting and Linear Ensembles

This track compares selected-feature XGBoost with ridge logistic regression,
shrinkage linear discriminant analysis (LDA), and partial least squares (PLS)
followed by logistic regression. See the [project README](../../README.md) for
the task, dataset, recorded results, and validation limitations.

## Setup

From the repository root, use Python 3.12 and an activated virtual environment:

```bash
python -m pip install -r pipelines/panel_ensemble/requirements.txt
```

`requirements-local.txt` remains a compatibility alias to the same dependencies.
Default data and artifact paths resolve relative to the scripts.

## Models

| CLI model name | Protein panel | Approach |
| --- | ---: | --- |
| `Starter XGBoost` | All proteins | Original boosted-tree baseline |
| `Selected XGBoost` | Top 90 | Shallow regularized boosted trees |
| `Selected Proteomics-Only XGBoost` | Top 90 | Stable protein selection, without clinical covariates |
| `Selected Ridge LR` | Top 80 | Scaled, L2-regularized logistic regression |
| `Selected Shrinkage LDA` | Top 120 | LDA with covariance shrinkage |
| `Selected PLS LR` | Top 260 | 14 supervised latent components + logistic regression |

The selected XGBoost branches use stable ANOVA selection over five stratified
subsamples within each training fold. The other branches use standard ANOVA
ranking. Clinical covariates are retained except in the proteomics-only branch.
Parameters and feature counts live in `model.py`.
Repeated stratified CV produces OOF predictions for comparing individual models
and cross-fitted convex blends. OOF log loss selects the final recipe.

## Evaluate, train, predict

Run from the repository root:

```bash
python pipelines/panel_ensemble/evaluate.py --data data/train.csv \
  --out pipelines/panel_ensemble/results/current
python pipelines/panel_ensemble/train.py --data data/train.csv \
  --recipe pipelines/panel_ensemble/results/current/final_recipe.json \
  --out pipelines/panel_ensemble/weights
python pipelines/panel_ensemble/predict.py --data /path/to/new_samples.csv \
  --model-dir pipelines/panel_ensemble/weights --out predictions_panel.csv
```

Evaluation defaults to five folds and three repeats. `--models` accepts quoted
model names; `--blend-models` controls eligibility for blending.
`--xgb-model-seeds` enables seed averaging. For a shorter run:

```bash
python pipelines/panel_ensemble/evaluate.py --data data/train.csv \
  --out pipelines/panel_ensemble/results/quick \
  --folds 3 --repeats 1 --xgb-n-jobs 1 \
  --models "Selected XGBoost" "Selected Ridge LR"
```

Evaluation saves `cv_results.csv`, `base_model_summary.csv`, OOF tables,
`ensemble_summary.csv`, and `final_recipe.json`. Training saves the required
models, feature lists, `ensemble_config.json`, and a training summary.
Selected XGBoost is always retained for starter-interface compatibility.

When a recipe is absent, training uses the historical fixed XGBoost/ridge/PLS
blend (0.41 / 0.12 / 0.32 / 0.15) and logs the fallback. Inference loads
`ensemble_config.json` when present, otherwise the single XGBoost artifact
in `--model-dir`.

```bash
bash pipelines/panel_ensemble/predict.sh /path/to/new_samples.csv predictions_panel.csv
python pipelines/panel_ensemble/evaluate.py --help
python pipelines/panel_ensemble/train.py --help
python pipelines/panel_ensemble/predict.py --help
```

The wrapper uses the caller's active environment; `MEDAI_PYTHON` can override
the interpreter. Output includes `sample_id`, `prob_ati`, and `pred_label`,
component probabilities for ensembles, and `true_label` for labeled inputs.

## Experiment history

- [results/selected_models/](results/selected_models/): saved five-fold comparison.

These records predate the current recipe-driven evaluator. Re-run evaluation to
assess changes; archived metrics do not guarantee performance on new cohorts.
