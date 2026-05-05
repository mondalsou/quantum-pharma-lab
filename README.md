# Drug Candidate Selection with QAOA

A pharma-flavored quantum computing project using IBM Qiskit.

The repository has two notebook tracks built on the same public cheminformatics dataset:

- `notebooks/qaoa_drug_candidate_selection.ipynb` explores a small ML + QAOA workflow for selecting a few molecules from an 8-candidate pool
- `notebooks/large_scale_drug_portfolio_qaoa_concept.ipynb` shows a larger portfolio-style concept with a 10,000-like screening library, classical shortlisting, similarity penalties, and QAOA on a reduced problem
- both notebooks compare QAOA against classical baselines so the quantum step is easy to interpret

The notebooks cover two related workflows:

- a small ML + QAOA candidate-selection notebook that chooses a limited number of molecules for follow-up testing
- a larger portfolio-style QAOA concept notebook that builds a 10,000-like screening library, shortlists candidates classically, adds similarity and developability constraints, and then optimizes a reduced selection problem
- both notebooks use classical ML or scoring heuristics to rank molecules before QAOA
- both compare the quantum result with classical baselines so the optimization step is easy to interpret

## How QAOA Solves This Problem

We have **8 ESOL candidate molecules**. The goal is to **select exactly 3** that maximise a combined score (predicted solubility + descriptor desirability).

Each possible selection is a binary string of length 8 — e.g. `|00010011⟩` means "select molecules 3, 4 and 8". There are 2⁸ = 256 such bitstrings, but only C(8,3) = 56 are valid.

**`|ψ(γ,β)⟩` is the quantum state** the circuit builds — a superposition over all 256 (2⁸)bitstrings at once, each carrying a complex amplitude. It is not the answer. It is the vehicle the quantum computer uses to find the answer.

```
|ψ(γ,β)⟩  →  measure  →  bitstring  →  decode  →  selected molecules
 (quantum)               (classical)              (your answer)
```


### The Full Chain

**Step 1 — Classical objective.**
Score each of the 8 molecules. Define the problem:

```
maximize  Σ score_i · x_i
subject to  Σ x_i = 3,  x_i ∈ {0, 1}
```

**Step 2 — Encode into H_cost.**
`H_cost` is a diagonal operator. For each basis state (bitstring) `|x⟩`, it returns the score of that selection:

```
H_cost |x⟩ = C(x) · |x⟩
```

**Step 3 — QUBO formulation.**
The equality constraint `Σ x_i = 3` is absorbed into the objective as a penalty term `P · (Σ xᵢ − 3)²`. The problem becomes a single unconstrained quadratic: the QUBO. Qiskit's `QuadraticProgramToQubo` does this automatically.

```
C(x)  =  Σ score_i · x_i  −  P · (Σ x_i − 3)²
```

**Step 4 — Make it a gate: `e^{−iγ H_cost}`.**
The gate rotates each bitstring's phase by its score. Amplitudes stay identical — only the angles change

```
e^{−iγ H_cost} |x⟩ = e^{−iγ C(x)} |x⟩
```

Since `|e^{−iγ C(x)}| = 1` always, amplitudes are unchanged — only phases shift. This is a valid unitary gate.

**Step 5 — Superposition.**
Initialise all 8 qubits i.e. a uniform superposition over all 256 bitstrings, each with amplitude `1/√256`. Every bitstring gets equal weight.

**Step 6 — Cost layer.**
Apply `e^{−iγ H_cost}`. Each of the 256 amplitudes gets a phase kick proportional to its score. High-scoring bitstrings rotate faster. **Amplitudes are still all equal** — no bitstring has been favoured yet. Only phases differ.

```
e^{−iγ H_cost} |ψ⟩  =  Σₓ  e^{−iγ C(x)} · αₓ · |x⟩

High score C(x)  →  big phase rotation
Low score C(x)   →  small phase rotation
|amplitude|      →  unchanged, still 1/16 for all

γ controls how much separation you create between phases.
```

**Step 7 — Mixer layer.**
Apply `e^{−iβ ΣXᵢ}` (X-rotations on all qubits). This is where interference happens. Bitstrings that were phase-aligned after the cost layer constructively interfere and grow in amplitude. Bitstrings that were out of phase destructively interfere and shrink. For the first time, probabilities are no longer equal.

**Step 8 — Repeat for p layers.**
Steps 6 and 7 form one QAOA layer. Repeat `p` times with independent `(γₖ, βₖ)` parameters per layer. More layers → deeper interference → better approximation, at the cost of a deeper circuit.


**Steps 5 to 8** are shown in **[here](https://mondalsou.github.io/quantum-pharma-lab/qaoa_wave_explainer.html)**

**Step 9 — COBYLA tunes the angles.**
A classical optimizer (COBYLA) evaluates `⟨ψ(γ,β)| H_cost |ψ(γ,β)⟩` after each circuit run. It adjusts `γ` and `β` to minimise this expectation value (equivalently, maximise the score). This outer loop runs for up to 200 iterations.

**Step 10 — Measure.**
The bitstring that appears most often is the one the interference pattern amplified most. That is the QAOA answer.

**Step 11 — Decode.**
Map the bitstring back to molecule indices. E.g. `|00010011⟩` → molecules 3, 4, 8 are selected. Compare against brute-force and simulated annealing to assess quality.


## Project Files

- `docs/architecture.md` - QAOA and VQE architecture: pipelines, circuit structure, data flow
- `docs/qaoa_wave_explainer.html` - interactive wave interference explainer
- `notebooks/qaoa_drug_candidate_selection.ipynb` - main implementation notebook
- `notebooks/large_scale_drug_portfolio_qaoa_concept.ipynb` - large-scale portfolio selection concept notebook
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

This is a toy project. The goal is to show:

- Classical ML predicts a molecular property.
- Classical and quantum optimizers decide which molecules to select under constraints.

## Dataset

The project uses the Delaney ESOL dataset from MoleculeNet/DeepChem. It contains 1,128 small molecules with SMILES strings, measured log aqueous solubility, and molecular descriptors.

Source:

- DeepChem MoleculeNet Delaney documentation: https://deepchem.readthedocs.io/en/latest/api_reference/moleculenet.html
- CSV mirror used by this project: https://raw.githubusercontent.com/deepchem/deepchem/master/datasets/delaney-processed.csv
