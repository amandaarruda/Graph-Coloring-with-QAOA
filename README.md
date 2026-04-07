# 🎨 QAOA for Graph Coloring

A hybrid quantum-classical approach to solving the Graph Coloring Problem using the Quantum Approximate Optimization Algorithm (QAOA).

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue?logo=python)](https://www.python.org/)
[![PennyLane](https://img.shields.io/badge/PennyLane-0.44-purple)](https://pennylane.ai/)

---


## 📋 Table of Contents


- [Overview](#overview)
- [The Problem](#the-problem)
- [Algorithm](#algorithm)
- [Implementation](#implementation)
- [Experiments](#experiments)
- [Results](#results)
- [Limitations & Future Work](#limitations--future-work)
- [Getting Started](#getting-started)
- [Repository Structure](#repository-structure)
- [Team](#team)
- [References](#references)


---


## Overview


This project was developed as part of the **Brazil Quantum CAMP** competition by **Team Q-Trust AI**. We implement QAOA - a hybrid quantum-classical algorithm - to solve the **Graph Coloring Problem**, which consists of assigning colors to graph vertices such that no two adjacent vertices share the same color.


We validate the approach across three experiments of increasing complexity:


| Experiment | Graph | Vertices | Colors | Qubits | χ (chromatic number) |
|:---:|:---:|:---:|:---:|:---:|:---:|
| 1 | Path P₃ | 3 | 2 | 6 | 2 |
| 2 | Cycle C₅ | 5 | 3 | 15 | 3 |
| 3 | Complete K₄ | 4 | 4 | 16 | 4 |


---


## The Problem


In the **Graph Coloring Problem**, given a graph G = (V, E), we seek a function that assigns a color c ∈ {0, ..., k−1} to each vertex such that for every edge (u, v) ∈ E:


```text
color(u) ≠ color(v)
```


The minimum number of colors needed is called the **chromatic number χ(G)**.


### One-Hot Encoding


To represent this on a quantum computer, we use **one-hot encoding**: each vertex `v` is represented by `k` qubits `x_{v,0}, ..., x_{v,k−1}`, where exactly one qubit equals 1, indicating the assigned color. The qubit index is:


```text
q(v, c) = v · k + c
```


### Cost Hamiltonian


The QUBO (Quadratic Unconstrained Binary Optimization) formulation is converted to an Ising Hamiltonian:


```text
H_C = Σ_{(u,v)∈E} Σ_c x_{u,c} · x_{v,c}   <- conflict term
    + A · Σ_v (Σ_c x_{v,c} − 1)²            <- one-hot constraint
```


The penalty coefficient `A = 5` was used empirically to balance constraint enforcement and conflict minimization.


---


## Algorithm


**QAOA** (Quantum Approximate Optimization Algorithm) is a hybrid quantum-classical algorithm that iteratively refines a parameterized quantum circuit to minimize an expectation value ⟨H_C⟩.


### How It Works


```text
1. Initialize: uniform superposition |+⟩^⊗n via Hadamard gates
2. Repeat p layers:
   a. Cost layer:  U_C(γ) = exp(-iγ H_C)  - biases qubits toward low-energy states
   b. Mixer layer: U_M(β) = exp(-iβ H_M)  - prevents trapping in local minima
3. Measure: sample bitstrings and decode the best valid coloring
4. Optimize: classical optimizer updates γ and β parameters
```


The circuit depth (number of layers `p`) controls the trade-off between solution quality and computational cost.


---


## Implementation


**Framework:** [PennyLane](https://pennylane.ai/) `v0.44`


### Key Components


- **`indice_qubit(v, c, k)`** - maps vertex-color pair to qubit index
- **`construir_hamiltoniano(n, k, edges, A)`** - builds the Ising Hamiltonian (h, J, cst)
- **`circuito_qaoa(params, coefs, obs, n_qubits, p)`** - variational QAOA circuit
- **`otimizar(n_qubits, coefs, obs, p, n_iter, n_restarts)`** - classical optimization loop
- **`amostrar(params, ...)`** - samples bitstrings from the optimized circuit
- **`decodificar(bitstring, n, k)`** - decodes measurement outcome to color assignment


### Classical Optimizers Compared


| Optimizer | Strategy | Best for |
|:---:|:---:|:---:|
| **COBYLA** | Gradient-free, constraint-aware | Noisy landscapes |
| **Nelder-Mead** | Simplex-based, gradient-free | Smooth landscapes |
| **L-BFGS-B** | Quasi-Newton (Hessian approx.) | Differentiable objectives |


All optimizers used **10 random restarts** to mitigate local minima.


---


## Experiments


### Experiment 1 - Path Graph P₃ (Validation)


- 3 vertices, 2 colors, 6 qubits
- Known optimal: 0 conflicts (2-colorable)
- Used to validate the Hamiltonian construction and circuit


### Experiment 2 - Cycle Graph C₅


- 5 vertices, 3 colors, 15 qubits
- χ(C₅) = 3 - requires an odd number of colors
- Tests scalability to mid-size instances


### Experiment 3 - Complete Graph K₄


- 4 vertices, 4 colors, 16 qubits
- χ(K₄) = 4 - every vertex is adjacent to all others
- Most constrained instance; tests maximum density


---


## Results


### Effect of Circuit Depth p


| Graph | p=1 | p=2 | p=3 |
|:---:|:---:|:---:|:---:|
| P₃ | Higher cost | **Best trade-off** | Marginal gain |
| C₅ | Higher cost | **Best trade-off** | Diminishing returns |
| K₄ | Higher cost | **Best trade-off** | Diminishing returns |


Increasing `p` from 1→2 yielded the largest improvement; gains from 2→3 were minimal (**diminishing returns**).


### Optimizer Comparison (p=2)


All three optimizers converged to valid colorings with 0 conflicts. L-BFGS-B achieved the best final cost value, though differences were small. The algorithm was **robust to optimizer choice** when multiple restarts were used.


### Summary Table


| Experiment | Graph | Qubits | Final Cost | Conflicts |
|:---:|:---:|:---:|:---:|:---:|
| 1 | P₃ | 6 | ~−7.25 | **0** |
| 2 | C₅ | 15 | closer to theoretical | **0** |
| 3 | K₄ | 16 | further from theoretical | **0** |


Despite not reaching theoretical optima in larger instances, QAOA found **valid colorings with zero conflicts** in all three experiments.


---


## Limitations & Future Work


### Current Limitations


- **Low-depth circuits** are outperformed by classical algorithms (e.g., QAOA p=1 achieves approximation ratio 0.6924 on MaxCut vs. 0.878 for Goemans-Williamson)
- **Standard mixer H_M = Σ X_i** does not preserve the one-hot subspace, allowing the circuit to explore invalid states
- **Barren plateaus** (near-zero gradients) can hinder convergence at large p
- **Decoherence** limits circuit depth on real NISQ hardware


### Planned Improvements


1. **XY Mixer** - replace the standard mixer with one that preserves the one-hot subspace, restricting evolution to valid states only
2. **Warm-start initialization** - use `greedy_color` solution to initialize circuit parameters, reducing the number of restarts needed
3. **Progressive depth growth** - transfer optimized parameters from depth p to p+1 instead of re-initializing
4. **Systematic penalty tuning** - vary A systematically rather than fixing A=5 empirically


---


## Getting Started


### Prerequisites


```bash
python >= 3.10
pip install pennylane networkx scipy matplotlib
```


### Run the Notebook


**Option 1 - Google Colab (recommended):**


[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/11fJt-MmM1lq-Vgz1TY42h8j-mev065zN?usp=sharing)


**Option 2 - Local:**


```bash
git clone https://github.com/<your-username>/qaoa-graph-coloring.git
cd qaoa-graph-coloring
pip install -r requirements.txt
jupyter notebook notebooks/QAOA_Graph_Coloring.ipynb
```


---


## Repository Structure

´´´
qaoa-graph-coloring/
│
├── notebooks/
│ └── QAOA_Graph_Coloring.ipynb # Main experiment notebook
│
├── docs/
│ └── report.pdf # Technical report (Brazil Quantum CHMP)
│
├── assets/
│ └── (figures generated by the notebook)
│
├── requirements.txt # Python dependencies
├── LICENSE # MIT License
└── README.md # This file
´´´


---


## Team


**Team Q-Trust AI** - Brazil Quantum CHMP, 2026


| Name |
|:---|
| Amanda Arruda |
| Caio Silva |
| Diogo Lacerda |
| Eduarda Mendes |
| Igor Oliveira |
| Mateus Granha |
| Paulo Aquino |
| Rebeca Vitória Tenório |
| Vinícius Leal |


---


## References


1. Farhi, E., Goldstone, J., & Gutmann, S. (2014). *A Quantum Approximate Optimization Algorithm*. MIT. [arXiv:1411.4028](https://arxiv.org/abs/1411.4028)
2. Ceroni, J. *Intro to QAOA*. PennyLane Demos. [pennylane.ai](https://pennylane.ai/qml/demos/tutorial_qaoa_intro)
3. Baker, J. S., & Radha, S. K. (2022). *Wasserstein solution quality and the QAOA: a portfolio optimization case study*. [arXiv:2202.06782](https://doi.org/10.48550/arXiv.2202.06782)
4. Hodson, M. et al. (2019). *Portfolio rebalancing experiments using the quantum alternating operator ansatz*. [arXiv:1911.05296](https://doi.org/10.48550/arXiv.1911.05296)
5. Chandarana, P. et al. (2022). *Digitized-counterdiabatic quantum algorithm for protein folding*.
6. Li, J. et al. (2020). *Hierarchical improvement of QAOA for object detection*. ISQED 2020. [DOI](https://doi.org/10.1109/ISQED48828.2020.9136973)


---


<div align="center">
  <sub>Developed at the Brazil Quantum CAMP · 2026</sub>
</div>
