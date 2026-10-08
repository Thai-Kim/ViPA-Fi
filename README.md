# ViPA-Fi: Visibility-Aware Probabilistic Skeleton Estimation from WiFi CSI

**Research code for WiFi-based 3D human pose estimation, joint-level uncertainty analysis, and offline selective-action replay.**

> **Project status:** Research implementation. This repository is not an official AAMAS release, and no acceptance or peer-reviewed performance claim is implied. Numerical results should be cited only after they have been independently reproduced using the specified evaluation protocol.

## Overview

Estimating 3D human pose from WiFi channel state information (CSI) is challenging because different joints have different degrees of observability. ViPA-Fi augments a graph-based pose estimator with joint-level probabilistic outputs and confidence-informed refinement. It is intended to support both pose estimation and downstream analysis of decisions made under uncertain perception.

The implementation includes:

- **CSI-to-joint feature aggregation:** shared CSI feature extraction and joint-specific evidence aggregation.
- **Graph-based 3D pose decoding:** a GraFormer-based coarse pose predictor for 17 body joints.
- **Probabilistic estimation:** per-joint coordinate uncertainty and an auxiliary visibility/observability estimate.
- **Hierarchical refinement:** visibility- and uncertainty-conditioned residual correction of limbs and extremities.
- **Controlled experiments:** configurable loss terms, alternative aggregation mechanisms, optional CSI consistency regularization, and ablation automation.
- **Offline decision analysis:** uncertainty-based selective retention (interpretable as *continue* versus *pause*) over predefined joint groups.

The offline decision analysis is **not** a validated physical-robot safety controller. In the current implementation, it compares uncertainty-ranked retention with a random-retention baseline and reports retained-sample pose error; it does not establish real-world collision avoidance or false-continuation rates.

## Method at a glance

```text
WiFi CSI
   | 
Shared feature encoder
   |
Skeleton evidence aggregation
   |
GraFormer coarse 3D pose
   |----> Joint-level log-variance and visibility scores
   |                    |
   +----> Visibility/uncertainty-guided limb refinement
                               |
                 Extremity refinement
                               |
                     Final 3D pose
                               |
                Uncertainty evaluation
                               |
                 Offline selective replay
```

The central implementation is in [`model/model.py`](model/model.py) (`ViPAFiNet`, `SkeletonEvidenceAggregator`, `JointUncertaintyHead`, and `VisibilityAwareExtremityRefinement`). The training objectives and their weights are defined in [`train.py`](train.py).

## Repository structure

| Path | Purpose |
| --- | --- |
| `model/` | CSI feature processing, graph decoder, uncertainty head, and pose refinement |
| `feeder/mmfi.py` | MM-Fi dataset loading and protocol-specific partitions |
| `config/mmfi/` | Random, cross-subject, and cross-environment experiment settings |
| `train.py` | Training and checkpoint/metric logging |
| `finetune.py` | Optional fine-tuning from an existing checkpoint |
| `run_ablation.py`, `ablation_manager.py`, `ablation_reporter.py` | Ablation orchestration and report generation |
| `run_analysis.py`, `analysis/` | Calibration, offline selective replay, and visual analysis |
| `generate_tables.py` | Result-table utilities |
| `reports/` | Working reports; **not** a verified release of ViPA-Fi numerical results |
| [`REPRODUCIBILITY.md`](REPRODUCIBILITY.md) | Protocols, evaluation caveats, and artifact checklist |

## Environment

The original project documentation specifies **Python 3.10.16, PyTorch 2.7.0, and CUDA 12.8**. Hardware and framework configurations other than these have not been verified as part of this documentation update.

```bash
git clone https://github.com/tiepaitvnuis/ViPA-Fi.git
cd ViPA-Fi
python -m venv .venv
# Linux/macOS: source .venv/bin/activate
# Windows PowerShell: .venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
# Install torch and torchvision for your operating system/CUDA from https://pytorch.org/get-started/locally/
python -m pip install -r requirements.txt
```

**Dependency note:** `torch` and `torchvision` are required by the source but are not pinned in `requirements.txt`; install them separately as shown above. The optional dataset downloader may also require `gdown`. For fully reproducible comparisons, record exact package versions, GPU, operating system, and CUDA runtime.

## Dataset and evaluation protocols

The project uses the [MM-Fi multimodal human-pose dataset](https://ntu-aiot-lab.github.io/mm-fi). The dataset itself is **not redistributed** in this repository. Obtain it from the original authors and comply with their terms of use.

The YAML configurations specify the following MM-Fi Protocol 1 settings:

| Setting | Configuration | Split configured in the file |
| --- | --- | --- |
| S1 | `config/mmfi/pose_config_p1s1.yaml` | `random_split` (80/20, random seed 0) |
| S2 | `config/mmfi/pose_config_p1s2.yaml` | `cross_subject_split` |
| S3 | `config/mmfi/pose_config_p1s3.yaml` | `cross_scene_split` / cross-environment |

An additional S2 configuration with a batch size of 64 is provided at `config/mmfi/pose_config_p1s2_vipafi.yaml`. Use the exact same dataset partition and preprocessing for every comparison. The evaluation loader is called `val_dataset` by the code; that name alone does not establish a separately held-out test set.

The YAML files contain a machine-specific Windows `dataset_root` value. **Always override it** using `--dataset_root` (or edit a local configuration) rather than assuming the included path exists on a new machine.

## Quick start

### Train on MM-Fi Protocol 1, Setting 1 (random split)

```bash
python train.py \
  --config_file config/mmfi/pose_config_p1s1.yaml \
  --dataset_root /absolute/path/to/MMFi_Dataset \
  --experiment_name vipafi_s1 \
  --total_epoch 50 \
  --seed 42
```

For cross-subject generalization, change the configuration to `config/mmfi/pose_config_p1s2.yaml`; for cross-environment generalization, use `config/mmfi/pose_config_p1s3.yaml`.

### Explicit model configuration

```bash
python train.py \
  --config_file config/mmfi/pose_config_p1s2.yaml \
  --dataset_root /absolute/path/to/MMFi_Dataset \
  --experiment_name vipafi_s2 \
  --agg_mode joint_attn \
  --ext_loss_weight 0.35 \
  --coarse_loss_weight 0.25 \
  --nll_loss_weight 0.10 \
  --visibility_loss_weight 0.02 \
  --consistency_weight 0 \
  --total_epoch 50 \
  --seed 42
```

`--consistency_weight 0` disables the optional extra-forward-pass CSI perturbation objective. The training script exposes additional flags through `python train.py --help`. **Record the exact command**, because some additional objectives (e.g., bone-length consistency) have nonzero defaults.

### Run ablations

```bash
python run_ablation.py \
  --table 2,3 \
  --config_file config/mmfi/pose_config_p1s2.yaml \
  --dataset_root /absolute/path/to/MMFi_Dataset \
  --backup_dir ./artifacts/ablation \
  --total_epoch 30
```

The ablation runner also supports `--table 1`, `--table all`, and `--runs A5,B1,B2,C1` to choose specific configurations. Its **30-epoch default differs from the 50-epoch training default**. Do not compare runs trained for different budgets as if they were matched experiments.

### Uncertainty and offline decision analysis

```bash
python run_analysis.py \
  --checkpoint /absolute/path/to/pose_mpjpe.pt \
  --config_file config/mmfi/pose_config_p1s2.yaml \
  --dataset_root /absolute/path/to/MMFi_Dataset \
  --output_dir ./reports/analysis_results
```

The script produces calibration plots, zone-wise selective-retention curves, pose grids, and uncertainty/visibility visualizations. **Verify that the model configuration and uncertainty scaling match the checkpoint before interpreting these outputs.** The analysis loader currently constructs the model with symmetric graph edges enabled; a checkpoint trained with a different graph setting may not be comparable. See [`REPRODUCIBILITY.md`](REPRODUCIBILITY.md).

## Metrics and outputs

Training logs contain **MPJPE**, **PA-MPJPE**, **PCK** at configured thresholds, per-joint/extremity error, and predicted uncertainty/visibility summaries. Checkpoints are written below `pose_weights/`; TensorBoard logs and `best_metrics.json` are written below `logs/`.

**No new ViPA-Fi quantitative claims are made by this README.** In particular, the GraphPose-Fi figures in `reports/comparison_tables.md` are labeled baseline/reference material, while the ViPA-Fi entries are pending. Before reporting a headline number, publish the associated data split, seed(s), training budget, checkpoint, metric units, and evaluation script or logs.

## Reproducibility and limitations

Read [`REPRODUCIBILITY.md`](REPRODUCIBILITY.md) before reproducing or citing numerical results. In particular:

1. The provided scripts use the selected MM-Fi partition's validation split for periodic model selection and reporting; a distinct untouched final test evaluation must be documented if claimed.
2. The visibility score is learned from an error-derived supervisory signal; it should not be interpreted as an independently annotated physical visibility label.
3. The selective-action experiment is an **offline replay**. Report its retention rates and retained-sample MPJPE, not unimplemented safety or robot-execution metrics.
4. Numerical figures in the working reports are not substitutes for versioned run logs and an independent reproducibility check.
5. No MM-Fi data, trained model weights, or final benchmark result files are bundled with this source snapshot.

## Citation and licensing

A citable paper reference (authors, venue, year, and DOI) should be added only once bibliographic details are confirmed. Until then, cite the specific repository commit and access date when referring to this software. **No open-source license has been supplied in this archive**; the rights holder should select and add a `LICENSE` file before broad reuse is encouraged.

## AAMAS 2027 submission note

AAMAS 2027 uses **double-blind peer review**. Do not link an author-identifiable GitHub repository from an anonymized manuscript or anonymized supplementary archive. Instead, submit anonymized supporting code/data according to the [official supplementary-material instructions](https://warwick.ac.uk/fac/sci/dcs/aamas2027/guidelines-and-policies/instructions/). The AAMAS supplementary upload is a **single ZIP of at most 25 MB**, and the official instructions require that it not reveal author identity. Public archival access can be provided at publication time where applicable.

---

*This README documents the code in the supplied research snapshot. It is not a claim of paper acceptance, deployed robot safety, or fully reproduced benchmark scores.*