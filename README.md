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

## How QAOA Solves This Problem

We have **8 ESOL candidate molecules**. The goal is to **select exactly 3** that maximise a combined score (predicted solubility + descriptor desirability).

Each possible selection is a binary string of length 8 — e.g. `|00010011⟩` means "select molecules 3, 4 and 8". There are 2⁸ = 256 such bitstrings, but only C(8,3) = 56 are valid.

**`|ψ(γ,β)⟩` is the quantum state** the circuit builds — a superposition over all 256 bitstrings at once, each carrying a complex amplitude. It is not the answer. It is the vehicle the quantum computer uses to find the answer.

```
|ψ(γ,β)⟩  →  measure  →  bitstring  →  decode  →  selected molecules
 (quantum)               (classical)              (your answer)
```

- The optimization problem is classical: maximize `Σ score_i · x_i` subject to `Σ x_i = 3`
- QAOA encodes that into `H_cost`: each bitstring's amplitude gets phase-shifted by its score
- `|ψ(γ,β)⟩` gets shaped by alternating cost and mixer layers so the best bitstring accumulates the most amplitude
- You measure and the superposition collapses — the bitstring that comes out is your molecule selection
- The classical optimizer (COBYLA) tunes `γ` and `β` across many circuit runs to maximise the chance of measuring the right answer

**`|ψ⟩` is the vehicle. The bitstring is the destination.**

### Interactive Explainer

**[How QAOA Works Through Waves →](docs/qaoa_wave_explainer.html)**
See the five steps — superposition, phase shift, mixer, p layers, measurement — as live animated waves. Each wave is one basis state. Watch how `|ψ⟩` gets shaped until the right bitstring dominates. Open in any browser.

## Project Files

- `docs/PRD.md` - product requirements document
- `docs/architecture.md` - QAOA and VQE architecture: pipelines, circuit structure, data flow
- `docs/qaoa_wave_explainer.html` - interactive wave interference explainer
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
