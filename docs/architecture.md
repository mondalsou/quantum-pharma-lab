# Architecture: Quantum Pharma Lab

## Overview

This project combines classical machine learning with two quantum algorithms — QAOA and VQE — for pharma-inspired problems. The two algorithms solve different problems and operate at different layers.

```
┌─────────────────────────────────────────────────────────────┐
│                   ESOL Dataset (1,128 molecules)            │
└─────────────────────────┬───────────────────────────────────┘
                          │
              ┌───────────▼────────────┐
              │   Classical ML Layer   │
              │  (solubility scoring)  │
              └───────────┬────────────┘
                          │
           ┌──────────────┴──────────────┐
           │                             │
  ┌────────▼────────┐          ┌─────────▼────────┐
  │   QAOA Layer    │          │    VQE Layer      │
  │ (select which   │          │ (estimate ground  │
  │  molecules)     │          │  state energy)    │
  └─────────────────┘          └──────────────────┘
```

---

## QAOA Architecture

### What QAOA Does Here

QAOA does **not** predict molecular properties. It solves a **binary selection problem** after ML scoring is done:

> Given N scored molecules, select exactly K that maximize total score.

### Problem Formulation

```
maximize  Σ score_i · x_i
subject to  Σ x_i = K        (select exactly K)
            x_i ∈ {0, 1}     (binary: select or reject)
```

Each binary variable `x_i` represents one molecule candidate.

### QAOA Pipeline

```
ML scores
    │
    ▼
┌─────────────────────────────────┐
│ 1. QuadraticProgram             │  qiskit_optimization
│    - binary variables per mol   │
│    - linear objective: scores   │
│    - equality constraint: Σ=K   │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│ 2. QUBO Conversion              │  QuadraticProgramToQubo
│    - equality constraint →      │
│      penalty term in objective  │
│    - result: pure quadratic     │
│      binary problem             │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│ 3. QAOA Circuit                 │  qiskit_algorithms.QAOA
│    - p layers (reps)            │
│    - cost layer: encodes QUBO   │
│    - mixer layer: H gates       │
│    - parameters: γ (cost)       │
│                  β (mixer)      │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│ 4. Classical Optimizer          │  COBYLA (maxiter=200)
│    - minimizes ⟨ψ(γ,β)|H|ψ(γ,β)⟩│
│    - updates γ, β each iter     │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│ 5. Sampler + Measurement        │  Qiskit Sampler (4096 shots)
│    - sample bitstrings from     │
│      final circuit state        │
│    - most-probable bitstring    │
│      = selected molecules       │
└────────────────┬────────────────┘
                 │
                 ▼
          Selected portfolio
```

### QAOA Circuit Structure (p=3)

```
|0...0⟩ → H⊗n → [Cost(γ₁) → Mixer(β₁)] × 3 → Measure
                  └─────────────────────┘
                       one QAOA layer
```

- **Initial state**: uniform superposition over all 2^n bitstrings
- **Cost layer**: `exp(-i·γ·H_cost)` — rotates based on objective value
- **Mixer layer**: `exp(-i·β·H_mixer)` — X-rotations, allows transitions between bitstrings
- **p (reps)**: more layers = more expressive, deeper circuit

### QAOA Comparison: Brute Force vs SA vs QAOA

| Method | Exact? | Scales? | Role |
|---|---|---|---|
| Brute force | Yes | No (exponential) | Reference oracle |
| Simulated annealing | No | Yes | Scalable classical baseline |
| QAOA | No | Future hardware | Quantum heuristic |

### Large-Scale Variant (large_scale notebook)

For a 10,000-molecule library the full problem is too large for direct QAOA. The hybrid approach:

```
10,000 molecules
      │
      ▼ ML scoring + hard filters (MW, toxicity, cost)
      │
      ▼ Scaffold-based diversity reduction
      │
      ▼ 8-molecule shortlist
      │
      ▼ QAOA (select 3 with similarity penalty)
```

The quadratic objective adds pairwise similarity penalties to discourage redundant selections:

```
maximize  Σ score_i · x_i  −  λ · Σ sim(i,j) · x_i · x_j
```

---

## VQE Architecture

### What VQE Does Here

VQE estimates the **ground-state (lowest) energy** of a molecule. This is quantum chemistry, not optimization.

> Given a molecule's Hamiltonian, find the minimum eigenvalue.

### VQE Pipeline

```
Molecule geometry (atom coordinates)
    │
    ▼
┌─────────────────────────────────┐
│ 1. Electronic Structure Driver  │  PySCFDriver
│    - SCF calculation (STO-3G)   │
│    - produces full many-body    │
│      Hamiltonian                │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│ 2. Active Space Reduction       │  ActiveSpaceTransformer
│    - freeze core electrons      │
│    - keep 2 electrons,          │
│      2 spatial orbitals         │
│    - makes qubit count small    │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│ 3. Fermion → Qubit Mapping      │  ParityMapper
│    - fermionic operators →      │
│      qubit Pauli operators      │
│    - parity mapping reduces     │
│      qubit count by 2           │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│ 4. UCCSD Ansatz                 │  UCCSD + HartreeFock init
│    - starts at Hartree-Fock     │
│      reference state            │
│    - adds single + double       │
│      excitation operators       │
│    - parameterized by θ         │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│ 5. VQE Optimization             │  SLSQP (maxiter=100)
│    - minimize ⟨ψ(θ)|H|ψ(θ)⟩    │
│    - StatevectorEstimator        │
│      (exact simulation)         │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│ 6. Classical Reference          │  NumPyMinimumEigensolver
│    - exact diagonalization      │
│    - used to verify VQE error   │
└────────────────┬────────────────┘
                 │
                 ▼
   VQE energy vs exact energy
   (absolute error in Hartree)
```

### VQE vs QAOA Contrast

| Aspect | QAOA | VQE |
|---|---|---|
| Problem type | Combinatorial optimization | Quantum chemistry |
| Input | Scored molecule list | Atom coordinates |
| Output | Selected molecule indices | Ground-state energy (Hartree) |
| Hamiltonian | QUBO/Ising (classical) | Electronic structure (fermionic) |
| Ansatz | Fixed p-layer QAOA circuit | UCCSD (chemistry-inspired) |
| Optimizer | COBYLA | SLSQP |
| Pharma use | Portfolio selection | Molecular stability / reactivity |

---

## Execution Backend

Both algorithms run on **statevector simulation** (laptop-runnable):

| Component | Backend |
|---|---|
| QAOA sampling | `qiskit.primitives.Sampler` (statevector) |
| VQE expectation values | `qiskit.primitives.StatevectorEstimator` |
| Classical reference | `NumPyMinimumEigensolver` (exact diag) |

No IBM Quantum hardware required. Simulation is exact but limited to small qubit counts.

---

## Qubit Count Summary

| Notebook | N molecules | Qubits (QAOA) | Note |
|---|---|---|---|
| qaoa_drug_candidate_selection | 8 | ~10–12 | QUBO slack adds qubits |
| large_scale_drug_portfolio_qaoa | 8 (shortlisted) | ~10–12 | 10k filtered to 8 classically |
| vqe_esol_active_space_demo | 3 molecules | 2 per molecule | Active space: 2e, 2 orbitals |

---

## Data Flow

```
delaney-processed.csv
        │
        ├─► ML training split → solubility model
        │
        └─► test split → candidate pool → QAOA selection scores
                                      └─► 3 small molecules → VQE geometries
```
