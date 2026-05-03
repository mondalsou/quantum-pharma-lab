# Drug Candidate Selection with QAOA

A very small pharma-flavored quantum computing project using IBM Qiskit.

The notebook models drug candidate prioritization as a two-step ML plus quantum optimization problem using a real public cheminformatics dataset:

- choose a limited number of molecules for follow-up testing
- train classical ML models to predict measured aqueous solubility
- use the best ML model's predictions as candidate scores
- reward simple developability-style descriptor targets
- solve the same toy selection problem with simulated annealing
- solve the toy problem with QAOA
- compare simulated annealing and QAOA selection against a brute-force classical optimizer

## Project Files

- `docs/PRD.md` - product requirements document
- `docs/architecture.md` - QAOA and VQE architecture: pipelines, circuit structure, data flow
- `notebooks/qaoa_drug_candidate_selection.ipynb` - main implementation notebook
- `notebooks/large_scale_drug_portfolio_qaoa_concept.ipynb` - large-scale portfolio selection concept notebook
- `notebooks/vqe_esol_active_space_demo.ipynb` - VQE molecular energy demo using three small ESOL candidates
- `data/delaney-processed.csv` - MoleculeNet/Delaney ESOL dataset
- `requirements.txt` - Python dependencies

## Setup

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
jupyter lab
```

Open:

```text
notebooks/qaoa_drug_candidate_selection.ipynb
```

## Project Scope

This is a toy educational project. It does not perform real molecular docking or clinical prediction. The goal is to show the correct division of labor:

- Classical ML predicts a molecular property.
- Classical and quantum optimizers decide which molecules to select under constraints.

## Dataset

The project uses the Delaney ESOL dataset from MoleculeNet/DeepChem. It contains 1,128 small molecules with SMILES strings, measured log aqueous solubility, and molecular descriptors.

Source:

- DeepChem MoleculeNet Delaney documentation: https://deepchem.readthedocs.io/en/latest/api_reference/moleculenet.html
- CSV mirror used by this project: https://raw.githubusercontent.com/deepchem/deepchem/master/datasets/delaney-processed.csv
