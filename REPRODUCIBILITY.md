# ViPA-Fi: Reproducibility and Artifact Notes

This document describes the released **source snapshot**, not a completed independently reproduced benchmark. It is designed to help readers inspect the implementation, rerun MM-Fi experiments, and check the scope of downstream decision claims.

## 1. Snapshot contents and omissions

Available: `train.py`, `finetune.py`, `model/`, `feeder/`, MM-Fi YAML configs, ablation automation, evaluation/visualization scripts, and working reports.

Not included: MM-Fi dataset, model checkpoints/weights, executable environment lockfile, complete per-seed run logs, and independently validated end-to-end results. The working reports should **not** be treated as a release of verified ViPA-Fi performance.

## 2. Environment capture

The original README reports Python 3.10.16, PyTorch 2.7.0, CUDA 12.8. Save the following alongside any reproduced experiment:

```bash
python --version
python -m pip freeze > environment-freeze.txt
python -c "import torch, torchvision; print('torch', torch.__version__, 'torchvision', torchvision.__version__, 'cuda', torch.version.cuda, 'available', torch.cuda.is_available())"
```

`requirements.txt` omits PyTorch and torchvision, which must be installed separately. The optional dataset download helper can require `gdown`.

## 3. Exact dataset and data split

| Protocol | Config | `split_to_use` |
| --- | --- | --- |
| P1-S1 | `config/mmfi/pose_config_p1s1.yaml` | `random_split` (80/20; seed 0) |
| P1-S2 | `config/mmfi/pose_config_p1s2.yaml` | `cross_subject_split` |
| P1-S3 | `config/mmfi/pose_config_p1s3.yaml` | `cross_scene_split` |

Record the original MM-Fi release/version, modality (WiFi CSI), subject/scene allocation, any excluded files, preprocessing steps, and whether reported samples are frames or sequences. In this snapshot, the code calls the evaluation partition `val_dataset` even when run-time prints use the word `test`; distinguish validation from an untouched test set explicitly in any paper.

## 4. Training protocol

A baseline reproducibility run can be started with:

```bash
python train.py --config_file config/mmfi/pose_config_p1s2.yaml \
  --dataset_root /absolute/path/to/MMFi_Dataset \
  --experiment_name vipafi_s2 --total_epoch 50 --seed 42
```

The CLI exposes optimizer/training defaults including learning rate `3e-4`, weight decay `0.02`, `--total_epoch 50`, effective batch size `64`, joint aggregation `joint_attn`, uncertainty loss `0.10`, visibility loss `0.02`, extremity loss `0.35`, coarse loss `0.25`, and bone consistency loss `0.20`. Preserve `args.json`, the YAML config, the best checkpoint(s), TensorBoard event files, and `best_metrics.json` for each run. Report seed variability rather than presenting a single run as robust evidence.

## 5. Matched ablations

```bash
python run_ablation.py --table 2,3 \
  --config_file config/mmfi/pose_config_p1s2.yaml \
  --dataset_root /absolute/path/to/MMFi_Dataset \
  --backup_dir ./artifacts/ablation --total_epoch 30
```

The included ablation manager defines component and loss variants such as A5 (full model), B1--B5 (aggregation/layer-count controls), and C1--C5 (loss-weight controls). The runner's default is 30 epochs, not 50. For formal ablation tables, ensure equal splits, seeds, epochs, optimizer settings, checkpoint selection criteria, and evaluation code. Do not mix results from differently budgeted runs.

## 6. Metric definitions and units

- **MPJPE:** mean Euclidean 3D joint-coordinate error; code reporting should be checked for consistent conversion to millimeters.
- **PA-MPJPE:** MPJPE after the Procrustes alignment implemented by the evaluation utilities.
- **PCK@threshold:** proportion of joints under the specified spatial error threshold; the threshold and units must be stated.
- **Extremity MPJPE:** mean error over the configured elbow/hand subset in `train.py`; do not silently change the subset.
- **Predicted uncertainty:** obtained from the model's learned log-variance; verify all scale factors used for training/evaluation.
- **Visibility score:** supervised using model-prediction error-derived targets; not an external annotated visibility ground truth.

## 7. Downstream selective decision analysis

`run_analysis.py` calls `ReplayDecisionSimulator.simulate_zone_replay` and evaluates three predefined joint groups: torso/head, lower limbs, and upper extremities. The tested retention fractions are **1.0, 0.9, 0.8, 0.7, 0.6, 0.5**. At each fraction, ViPA-Fi retains the lowest-uncertainty samples and compares their mean joint error with a uniformly random subset (fixed RNG seed 42).

This is a **selective-perception/abstention proxy**. It does not simulate a time-continuous industrial robot, obstacles, safety-critical outcomes, or costs of an actual pause. Therefore, **false continuation**, **unnecessary pause**, and a calibrated risk-budget guarantee cannot be derived from the currently included script without an explicitly specified safety-event definition and additional evaluation code.

**Checkpoint compatibility warning:** `run_analysis.py` currently creates `ViPAFiNet(..., use_symmetric_edges=True)` regardless of the training flag. Confirm training/checkpoint compatibility before drawing conclusions.

**Uncertainty scaling warning:** the training script exposes `--uncertainty_scale` (default 100.0), while the analysis CLI exposes a distinct `--uncertainty_scale` (default 1.0). These defaults must not automatically be assumed equivalent; verify coordinate scaling and NLL interpretation end to end.

## 8. Suggested minimal artifacts for a camera-ready research release

- [ ] Versioned source commit and final environment file
- [ ] Official MM-Fi acquisition and exact split manifest / seed
- [ ] Config + exact shell command for every paper table entry
- [ ] Checkpoint and `best_metrics.json` for each reported run
- [ ] Evaluation script and fixed threshold/units for all metrics
- [ ] At least several independent seeds and an uncertainty summary
- [ ] Explicit distinction between validation selection and final testing
- [ ] A separately specified safety label and evaluation protocol before making action-safety claims
- [ ] Citation metadata and a rights-holder-selected software license

## 9. Double-blind publication caution

AAMAS 2027 requires anonymous manuscripts and anonymous supplementary materials. Do not submit this GitHub URL or a ZIP containing `.git`, Git user metadata, identifying documents, internal reports, or author names as a blind-review artifact. A suitably anonymized source-only supplementary ZIP is separate from the public GitHub release. See the [official AAMAS 2027 instructions](https://warwick.ac.uk/fac/sci/dcs/aamas2027/guidelines-and-policies/instructions/).