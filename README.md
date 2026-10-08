# De Novo HIV Drug Design: Full Pipeline

An end-to-end, single-notebook pipeline that goes from the MoleculeNet **HIV** dataset to a ranked list of candidate molecules for HIV-1 protease and reverse transcriptase.

**Generation** (Conditional Graph VAE, WGAN-GP with multi-objective RL, latent diffusion) -> **Docking** (AutoDock Vina on 1HSG and 1VRT) -> **Active learning** (closed surrogate/RL/docking loop) -> **Hit selection** (tiered funnel + Pareto ranking).

> **Honesty note.** This is a research scaffold. A docking score is a binding estimate, not evidence of binding, and the generated molecules are computational hypotheses. Each part of the notebook ends with a statement of what it does and does not show.

## Repository contents

| File | Description |
|---|---|
| `de_novo_full_pipeline_HIV.ipynb` | The complete pipeline (Parts A to E) |

## Pipeline overview

| Part | What it does | Output |
|---|---|---|
| **A. Generation** | Conditional Graph VAE, then WGAN-GP with Tchebycheff RL, then a latent DDPM | `generated_candidates.csv` |
| **B. Docking** | Prepares 1HSG (protease) and 1VRT (RT), validates by redocking, screens candidates | `dock_1HSG.csv`, `dock_1VRT.csv` |
| **C. Active learning** | Surrogate, RL, dock a sample, retrain the surrogate (closed loop) | `al_rounds.csv`, `al_model.pt` |
| **D. Hit selection** | Tiered funnel calibrated on known actives, plus Pareto ranking | `hits.csv` |
| **E. Replication** | Seed replication, fixed external test set, consolidated report | printed report |

All stages live in one notebook on purpose. Part D derives its thresholds from the distribution of known actives docked in Part B, and splitting the stages across notebooks would move that link into hand-carried CSV files where it can break silently.

## Key design points

- **Lossless graph representation.** Atom token = (symbol, formal charge, explicit H count); aromatic bonds are their own class. A SMILES -> graph -> SMILES round-trip test verifies this.
- **Ring-first decoding.** Valence repair, a bond budget and a ring size constraint turn decoder output into chemically realistic molecules (aromatic fraction and ring statistics are compared against the training set).
- **Five property objectives** (all in [0, 1], larger is better): affinity proxy (Random Forest on Morgan fingerprints, scaffold-split evaluated), QED, structural-alert safety (PAINS, Brenk, NIH), synthesizability (SA score) and a LogP window.
- **Augmented Tchebycheff scalarisation** with Dirichlet weights, so the generator is pushed toward several regions of the Pareto front rather than a single compromise.
- **Structure-based reward.** Vina scores are mapped onto [0, 1] and added as objectives; a docking surrogate model makes RL tractable.
- **Validation before interpretation.** Redocking RMSD (< 2 A) must pass before any docking score is read; ligand efficiency is used to remove the molecular size effect.
- **Over-optimisation guards** in the active-learning loop (limited RL steps per round, surrogate bias tracking, rollback of collapsed rounds).
- **Data-calibrated hit thresholds** taken from known actives docked against the same receptor, instead of hand-picked cutoffs.
- **Resumable.** Every expensive stage writes its output to disk and is skipped if the output exists. Set `FORCE_RECOMPUTE` to override.

## Requirements

- Python 3.10 or newer (the notebook handles Python 3.13, see below)
- Packages: `rdkit`, `meeko`, `gemmi`, `openbabel-wheel`, `scikit-learn`, `pandas`, `matplotlib`, `scipy`, `torch`, `vina`
- A GPU is recommended for the full budget; `QUICK_RUN = True` runs on CPU (Part A training takes roughly 35 minutes)

The notebook installs its packages in section 0.2. The `vina` package on PyPI only ships wheels up to CPython 3.12, so on newer interpreters the notebook automatically falls back to the static AutoDock Vina 1.2.5 binary. The two backends agree to within about 0.15 kcal/mol.

## Input data

Place these files in the working directory (or in a Google Drive folder named `HIV` when using Colab):

| File | Source |
|---|---|
| `HIV.csv` | MoleculeNet HIV dataset |
| `1HSG.cif` or `.pdb` | RCSB PDB entry 1HSG (HIV-1 protease) |
| `1VRT.cif` or `.pdb` | RCSB PDB entry 1VRT (HIV-1 reverse transcriptase) |

A preflight cell checks for these files before any heavy computation starts.

## Usage

### Google Colab

1. Create a folder named `HIV` in the root of your Google Drive and upload the three input files.
2. Open `de_novo_full_pipeline_HIV.ipynb` in Colab and run all cells.
3. Outputs are written back to the same Drive folder, so a dropped session can resume where it stopped.

### Local

```bash
git clone https://github.com/McahitKutsal/HIV-de-novo-full-pipeline.git
cd HIV-de-novo-full-pipeline
# put HIV.csv, 1HSG.cif and 1VRT.cif here
jupyter lab de_novo_full_pipeline_HIV.ipynb
```

Outside Colab the Drive cell is a no-op and the current directory is used.

### Budget

Set `QUICK_RUN = True` in the configuration for a fast smoke test (reduced epochs and molecule counts), or `QUICK_RUN = False` for the full budget (GPU and several hours).

## Limitations

- Docking scores are weak rankers and do not replace binding free energy calculations or assays.
- At `QUICK_RUN = True` the generator is undertrained, so conditional control is clear for QED and synthesizability but weak or unstable for affinity, safety and LogP.
- Generated molecules are small (about 17 heavy atoms on average) compared with clinical inhibitors, so raw docking scores should be compared through ligand efficiency.
- Fragment-based or pocket-conditioned generation would be the natural next step for larger and more synthesizable candidates.

## Acknowledgements

Datasets and tools: MoleculeNet / DTP AIDS Antiviral Screen, RCSB PDB, RDKit, AutoDock Vina, Meeko, Open Babel, PyTorch.
