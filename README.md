# Full Waveform Inversion & Numerical Optimisation

**Imperial College London — MSc Environmental Data Science & Machine Learning**
**Tools:** Python · Devito · NumPy · Matplotlib

---

## Overview

This project implements a **Full Waveform Inversion (FWI) pipeline** for recovering subsurface
velocity structure from seismic recordings, alongside a companion investigation into the
numerical optimisation methods that underpin it.

FWI is a physics-driven inverse problem: given a set of seismic measurements recorded at the
surface and a smooth initial guess of the subsurface velocity model, the goal is to iteratively
recover the true velocity structure by minimising the misfit between observed and
forward-modelled shot records. It sits at the intersection of PDE-constrained simulation,
gradient-based optimisation, and computational geophysics.

---

## Motivation

Directly measuring subsurface structure is physically impossible at scale. Seismic inversion
is the primary tool by which geophysicists infer what lies beneath — from hydrocarbon
exploration to earthquake hazard modelling to carbon storage site characterisation.

FWI is the state-of-the-art approach because it uses the **full information content** of the
recorded wavefield (amplitude, phase, and travel time) rather than just arrival times.
This makes it significantly more powerful than classical travel-time tomography, but also
substantially more computationally demanding — requiring repeated PDE solves and
adjoint-state gradient computations per iteration.

This project explores that computational challenge directly: how do numerical discretisation
choices (spatial finite-difference order, absorbing boundary thickness) affect inversion
fidelity, convergence stability, and cost?

---

## Repository Structure

```
.
├── Gradient-Optimisation.ipynb           # Gradient-based optimisation: Newton's method & gradient descent
├── fwi-inversion.ipynb           # Full FWI pipeline: subsampling, forward modelling, inversion, experiments
├── data/
│   ├── README.md         # Data format and acquisition geometry description
│   ├── smoothed_vp.npy   # Initial smoothed velocity model (401 × 401 grid, 6 m spacing)
│   ├── shot_0.npy        # Shot record — source position 0
│   ├── shot_1.npy        # Shot record — source position 1
│   ├── ...
│   ├── shot_8.npy        # Shot record — source position 8
│   └── time_values.npy   # Time axis for the original shot records (ms)
└── README.md
```

> **Note:** The `.npy` data files are not included in this public repository as they are
> provided under an academic license. The notebooks are fully documented and all outputs
> are embedded as cell results.

---

## Process

### Part 1 — Gradient-Based Optimisation

Before tackling FWI, the core optimisation machinery is built and analysed on a
closed-form objective function:

- **Analytic gradient and Hessian** derived and implemented from scratch in NumPy
- **Newton's method** implemented and run to convergence from the origin
- **Gradient descent** tested across a range of step sizes, with convergence behaviour
  compared against the theoretical bound derived from Hessian eigenvalues
- Convergence plots produced to visualise the effect of step size on stability and
  rate of descent

This establishes the mathematical foundation for understanding *why* FWI converges
the way it does under different configurations.

---

### Part 2 — Full Waveform Inversion Pipeline

#### Step 1 — Model Subsampling
The original 401 × 401 velocity grid (6 m spacing) is subsampled to 101 × 101
(24 m spacing) to reduce computational cost while preserving the large-scale
velocity structure relevant to low-frequency FWI.

#### Step 2 — Devito Model Construction
A Devito `Model` object is constructed from the subsampled grid, encoding the
velocity field, absorbing boundary layer (`nbl`), and finite-difference parameters
for the PDE solver.

#### Step 3 — Time Axis Generation & Shot Record Interpolation
A new time axis is generated from the subsampled model's CFL-stable `critical_dt`.
All 9 shot records are linearly interpolated onto this axis to align observed data
with the solver's internal time grid before computing residuals.

#### Step 4 — FWI Inversion Loop
The inversion minimises the $L_2$ misfit objective:

$$\phi(\mathbf{m}) = \frac{1}{2} \|\mathbf{d}_{\text{syn}}(\mathbf{m}) - \mathbf{d}_{\text{obs}}\|^2$$

Per iteration:
1. **Forward solve** — simulate wave propagation and record synthetic data
2. **Residual** — compute difference between observed and synthetic shot records
3. **Adjoint solve** — back-propagate residual to obtain the gradient via the adjoint-state method
4. **Update** — apply gradient step with box constraints ($v_p \in [1, 4]$ km/s)

#### Step 5 — Configuration Experiments

Two controlled experiments evaluate the sensitivity of FWI to key numerical parameters:

| Experiment | Parameter varied | Range tested | Fixed |
|---|---|---|---|
| 1 | `space_order` (FD stencil order) | 4, 6, 8, 10 | `nbl=10` |
| 2 | `nbl` (absorbing boundary width) | 20, 40, 60, 80, 100 | `space_order=4` |

Each experiment records three diagnostics: forward-modelled shot records,
recovered velocity models, and objective function convergence curves.

**Key findings:**
- Higher `space_order` reduces numerical dispersion and sharpens velocity recovery,
  with diminishing returns beyond order 4 for this grid and frequency content
- Insufficient `nbl` introduces spurious boundary reflections that contaminate
  gradients and produce corner artefacts in the recovered model; `nbl ≥ 60`
  was found to be the minimum effective value
- The two parameters are orthogonal — improving one does not compensate for a
  deficit in the other

---

## Key Concepts Demonstrated

- Physics-based simulation using Devito's finite-difference PDE framework
- Adjoint-state gradient computation for PDE-constrained optimisation
- Numerical stability analysis (CFL condition, absorbing boundaries)
- Systematic experimental design and fidelity vs. cost trade-off analysis
- Convergence diagnostics for iterative inverse problems
