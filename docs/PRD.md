# PRD: Drug Candidate Selection Using QAOA and IBM Qiskit

## 1. Product Summary

Build a simple Jupyter notebook that demonstrates how classical machine learning and the Quantum Approximate Optimization Algorithm (QAOA) can be combined for a pharma-inspired drug candidate selection problem using a real public molecular dataset.

The project uses the Delaney ESOL dataset from MoleculeNet/DeepChem. Each molecule has a SMILES string, measured log aqueous solubility, and molecular descriptors. The notebook trains classical ML models to predict solubility, creates a small out-of-sample candidate pool, formulates a binary optimization problem using predicted solubility-derived scores, solves it with QAOA using IBM Qiskit, and compares the result against a brute-force classical optimizer.

## 2. Problem Statement

Early-stage drug discovery often requires prioritizing a small number of candidate molecules under limited experimental capacity. Real workflows consider many signals, including docking score, ADMET properties, toxicity risk, novelty, synthesis cost, and intellectual property constraints.

This project creates a tiny educational version of that decision problem:

> Given several real molecules from ESOL, first predict solubility with classical ML, then select exactly three candidates that balance predicted solubility with simple developability-style descriptor preferences.

## 3. Goals

- Demonstrate the correct split between ML prediction and QAOA optimization.
- Train simple classical ML baselines for solubility prediction.
- Demonstrate QAOA on a clear pharma-style selection problem.
- Use a real public dataset while keeping the QAOA candidate pool small enough to run on a laptop simulator.
- Show the full workflow from problem definition to Qiskit implementation.
- Compare the QAOA result against a brute-force classical optimizer.
- Produce simple visual output that highlights the selected candidates.

## 4. Non-Goals

- Real drug discovery, real molecular simulation, or clinical prediction.
- Real docking, QSAR, ADMET, or clinical prediction.
- Use of proprietary pharma data.
- Running on IBM Quantum hardware in the first version.
- Building a web app or production service.

## 5. Target User

Primary user:

- A beginner/intermediate learner who wants a simple quantum computing project with a pharma story.

Secondary user:

- A student or portfolio builder who wants a small Qiskit notebook that is easy to explain.

## 6. User Story

As a learner, I want to predict molecule solubility with classical ML and then prioritize molecules with QAOA, so that I can understand how prediction and quantum optimization play different roles in pharma workflows.

## 7. Functional Requirements

The notebook must:

- Load the Delaney ESOL dataset from `data/delaney-processed.csv`.
- Show dataset shape, source, and core columns.
- Split the dataset into training and test sets.
- Train classical ML models to predict measured log solubility.
- Report MAE, RMSE, and R2 for each ML model.
- Select the best ML model by test RMSE.
- Create a reproducible candidate pool of eight molecules from the test set.
- Compute a single candidate value score from predicted solubility and descriptor desirability.
- Formulate a binary optimization problem with one decision variable per molecule.
- Add a constraint that exactly three molecules must be selected.
- Solve the problem with a brute-force classical baseline.
- Convert the optimization problem to QUBO form.
- Solve the QUBO using QAOA from Qiskit.
- Decode the selected molecules from the QAOA result.
- Compare QAOA and brute-force results.
- Visualize candidate scores and highlight the selected QAOA molecules.

## 8. Success Criteria

- The notebook runs top-to-bottom in a fresh Python environment after installing `requirements.txt`.
- The notebook produces a selected subset of three molecules.
- The ML baseline metrics, brute-force optimizer output, and QAOA output are displayed clearly.
- At least one chart is generated.
- The notebook text explains that this is a toy educational example.

## 9. Dataset

Dataset:

- Name: Delaney ESOL
- Source: MoleculeNet / DeepChem
- Size: 1,128 molecules
- Key fields:
  - `Compound ID`
  - `smiles`
  - `measured log solubility in mols per litre`
  - `Molecular Weight`
  - `Number of H-Bond Donors`
  - `Number of Rings`
  - `Number of Rotatable Bonds`
  - `Polar Surface Area`

Primary reference:

- Delaney, John S. "ESOL: estimating aqueous solubility directly from molecular structure." Journal of Chemical Information and Computer Sciences 44.3 (2004): 1000-1005.

Dataset documentation:

- https://deepchem.readthedocs.io/en/latest/api_reference/moleculenet.html

Candidate pool filtering:

```text
150 <= Molecular Weight <= 500
Number of H-Bond Donors <= 5
Number of Rotatable Bonds <= 8
Number of Rings >= 1
```

Toy value score:

```text
value = 5 * normalized_predicted_solubility
      + 2 * molecular_weight_desirability
      + 2 * polar_surface_area_desirability
      + 1 * rotatable_bond_desirability
```

Optimization objective:

```text
maximize sum(value_i * x_i)
subject to sum(x_i) = 3
x_i in {0, 1}
```

## 10. Technical Approach

- Language: Python
- Interface: Jupyter notebook
- Quantum SDK: IBM Qiskit
- Main packages:
  - `qiskit`
  - `qiskit-aer`
  - `qiskit-algorithms`
  - `qiskit-optimization`
  - `pandas`
  - `numpy`
  - `matplotlib`
  - `scikit-learn`

Implementation steps:

1. Load the ESOL dataset in pandas.
2. Train and evaluate classical ML baselines.
3. Pick the best ML model by RMSE.
4. Sample a reproducible out-of-sample candidate pool.
5. Compute per-candidate selection scores from ML predictions.
6. Build a `QuadraticProgram`.
7. Add binary variables and selection constraint.
8. Solve exact optimizer baseline by enumerating all combinations.
9. Convert the problem to QUBO.
10. Run QAOA with a small number of reps.
11. Interpret and visualize the result.

## 11. Risks and Constraints

- QAOA is probabilistic, so results may vary slightly by random seed and optimizer settings.
- Qiskit APIs can change across versions, so dependencies are pinned by minimum compatible versions.
- The full dataset is real, but the QAOA candidate pool and scoring function are intentionally simplified.
- QAOA does not predict solubility; it only optimizes selection after ML prediction.
- The solution should be viewed as a learning demo, not an assessment of real molecule quality.

## 12. Future Enhancements

- Add IBM Quantum Runtime execution.
- Add larger candidate sets.
- Include additional constraints such as maximum toxicity or total budget.
- Replace synthetic scores with public docking or QSAR-style sample data.
- Compare QAOA against simulated annealing or classical integer programming.
- Add sampled bitstring probability visualization.
