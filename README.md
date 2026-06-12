# SMPC-FL-Edge-IIoTset

Secure Multiparty Computation (SMPC)-enabled Federated Learning for IoT intrusion detection using the TON_IOT dataset.
This repository contains the full experimental code, configuration files, and result artifacts accompanying our paper.

## Overview

Traditional Federated Learning aggregates model updates on a central server, exposing individual client gradients.
This work integrates **Secure Multiparty Computation** into the aggregation step so the server never sees any client's raw weights.
Each client masks its update with pairwise noise before sending; the noise cancels during aggregation, leaving only the correctly weighted average.

Key properties of the framework:

- **Privacy-preserving aggregation** — pairwise noise masks cancel at the server; no raw weights are revealed.
- **Validation-aware client selection** — clients whose local models fail a shared validation check are excluded before aggregation.
- **Constrained Dirichlet non-IID partitioning** — every client receives a minimum per-class sample count; the remainder follows a Dirichlet distribution, producing realistic but bounded heterogeneity.
- **Robustness experiments** — a subset of clients can be assigned shuffled labels to simulate label-noise or Byzantine attacks.
- **Best-round selection on validation only** — the global model checkpoint with the highest validation macro-F1 is kept; the test set is evaluated exactly once after selection.

## Dataset

All experiments use the **TON_IOT** network intrusion dataset.

| Property | Value |
| --- | --- |
| Total records (raw) | ~211 000 |
| Feature columns (after preprocessing) | 113 |
| Target task | 10-class multiclass classification |
| Classes | `backdoor`, `ddos`, `dos`, `injection`, `mitm`, `normal`, `password`, `ransomware`, `scanning`, `xss` |
| Class-balance strategy | Resample each training class to 15 000 examples; validation and test splits kept at original distribution |
| Train / Val / Test split | 70 / 15 / 15 (stratified) |

The dataset is available from the University of New South Wales:
[https://research.unsw.edu.au/projects/toniot-datasets](https://research.unsw.edu.au/projects/toniot-datasets)

Place the file as `TON_IOT/train_test_network.csv` (or `TON_IOT/data.csv`) before running the notebooks.

## Model Architecture

```text
Input (113)
  → Dense(256) + BatchNorm + ReLU + Dropout(0.20)
  → Dense(128) + BatchNorm + ReLU + Dropout(0.15)
  → Dense(64)  + ReLU
  → Dense(10)  + Softmax
```

Optimizer: Adam (lr = 5 × 10⁻⁴, gradient clip norm = 1.0)  
Regularization: L2 weight decay (1 × 10⁻⁴) on all Dense layers  
Total parameters: 72 522 (71 754 trainable)

## Experiments

### Notebook Descriptions

| Notebook | Clients | Data dist. | Corrupted clients | Selection |
| --- | --- | --- | --- | --- |
| `TON_IOT/notebook.ipynb` | 5 or 10 | Non-IID (Dirichlet α=0.7) | Configurable | Validation-aware |
| `MPC-DNN-IID-5c-urpc.ipynb` | 5 | IID | 0 (1 unresponsive) | None |
| `MPC-DNN-IID-wp-3cc-ws.ipynb` | 10 | IID | 3 | Yes |
| `MPC-DNN-NonIID-wp-4cc-wos.ipynb` | 10 | Non-IID (warm-up) | 4 | No |
| `MPC-DNN-NonIID-wp-4cc-ws.ipynb` | 10 | Non-IID (warm-up) | 4 | Yes |
| `MPC-DNN-Noniid-test1.ipynb` | 10 | Non-IID (pure) | 0 | No |
| `central2.ipynb` | — | Centralized | — | — |

### Configuration Highlights

| Parameter | Value |
| --- | --- |
| Federated rounds | 15 |
| Local epochs per round | 4 |
| Batch size | 512 |
| Dirichlet α | 0.7 |
| Min samples per class per client | 500 |
| Noise scale (SMPC mask) | 1 × 10⁻⁴ |
| Validation accuracy threshold | 0.25 (multiclass) |

## Results

Results are stored under `TON_IOT/results_<clients>clients<config>/`.  
Each folder contains:

```text
<experiment>/
├── experiment_config.json          # Full hyperparameter record
├── validation_round_metrics.csv    # Per-round validation metrics
├── client_validation_metrics.csv   # Per-round per-client acceptance log
├── final_classification_report.csv # Test-set classification report
├── figures/
│   └── final_confusion_matrix.png
└── global_models/
    ├── global_model_round_XX.keras
    └── best_global_model.keras
```

### Sample Results (TON_IOT, 5 clients, α=0.7)

#### Clean setting (no corrupted clients)

| Class | Precision | Recall | F1 |
| --- | --- | --- | --- |
| backdoor | 0.865 | 0.999 | 0.928 |
| ddos | 0.947 | 0.892 | 0.919 |
| dos | 0.991 | 0.875 | 0.929 |
| injection | 0.671 | 0.672 | 0.671 |
| normal | 0.974 | 0.905 | 0.938 |
| password | 0.726 | 0.558 | 0.631 |
| ransomware | 0.765 | 0.853 | 0.807 |
| scanning | 0.818 | 0.866 | 0.841 |
| xss | 0.828 | 0.784 | 0.805 |
| **Overall accuracy** | | | **83.5%** |
| **Weighted F1** | | | **0.842** |

#### Robust setting (2 corrupted clients, label noise)

| Metric | Value |
| --- | --- |
| Overall accuracy | **84.9%** |
| Macro F1 | 0.783 |
| Weighted F1 | 0.856 |

## Requirements

```text
python >= 3.8
tensorflow >= 2.10
scikit-learn
imbalanced-learn
numpy
pandas
matplotlib
seaborn
joblib
```

Install with:

```bash
pip install tensorflow scikit-learn imbalanced-learn numpy pandas matplotlib seaborn joblib
```

GPU training was used for all reported results (TensorFlow 2.10, CUDA 11).
CPU execution is supported but will be significantly slower.

## Reproducing the Experiments

1. **Download the dataset** and place it at `TON_IOT/train_test_network.csv`.

2. **Open the main notebook** `TON_IOT/notebook.ipynb` in Jupyter.

3. **Set the configuration** at the top of the notebook:

   ```python
   EXPERIMENT_MODE = "clean"   # or "robust"
   NUM_CLIENTS    = 5          # or 10
   DIRICHLET_ALPHA = 0.7
   FAST_RUN       = False      # True for a quick smoke-test only
   ```

4. **Run all cells.** Artifacts are written to `TON_IOT/<run_name>/`.

5. For the IID and warm-up non-IID variants, open the corresponding notebook from the table above and run all cells.

6. For the centralized baseline, open `central2.ipynb` and run all cells.

> **Reporting note:** Run at least three random seeds and report mean ± std.
> Compare against standard FedAvg under the identical partition.
> Do not tune any hyperparameter using test-set performance.

## Repository Structure

```text
SMPC-FL-Edge-IIoTset/
├── TON_IOT/
│   ├── notebook.ipynb                        # Main optimized experiment
│   ├── results_5clients-alpha0.7/
│   │   ├── clean/mpc_dnn/                    # 5-client clean results
│   │   └── robust/mpc_dnn/                   # 5-client robust results
│   └── results_10clients_alpha0.7/
│       └── seed_*/clean|robust/mpc_dnn/       # 10-client results (multiple seeds)
├── MPC-DNN-IID-5c-urpc.ipynb
├── MPC-DNN-IID-wp-3cc-ws.ipynb
├── MPC-DNN-NonIID-wp-4cc-wos.ipynb
├── MPC-DNN-NonIID-wp-4cc-ws.ipynb
├── MPC-DNN-Noniid-test1.ipynb
├── central2.ipynb
└── README.md
```

## Citation

<!-- TODO: Replace with your full citation once the paper is published -->
```bibtex
@article{yourkey,
  title   = {Your Paper Title},
  author  = {Author One and Author Two and Author Three},
  journal = {Journal / Conference Name},
  year    = {2025},
}
```

## License

This code is released for academic research use.
If you use this work, please cite the paper above.
