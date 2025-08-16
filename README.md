# Application of PSO for Scheduling in Multi-Core Systems

Particle Swarm Optimization (PSO) for real-time task scheduling on multi-core CPUs.
We jointly optimize **task–to–core assignment**, **per-core frequency scaling**, and **per-job start-time decisions**, trading off energy vs. timing quality.

Authors: **Sina Beyrami**, **Mohamad Amin Karami**, **Mahyar Afshar**

---

## TL;DR

* **Baseline:** Worst-Fit-Decreasing (WFD) mapping + **EDF** per core.
* **Objective:** Minimize **energy** with a QoS penalty if we fall below baseline.
* **PSO decision vector:**

  1. task → core indices, 2) per-core frequency factors (∈\[0.5, 1.0]), 3) per-job start-time fractions (slack sharing ∈\[0,1]).
* **Metrics:** Energy, QoS (deadline-aware), Success Rate (on-time jobs).
* **Result (sample runs):** Energy ↓ **\~11–43%** vs. baseline, with QoS and success-rate trade-offs depending on swarm hyperparameters and utilization.

---

## Repository structure

```
Application-of-PSO-for-Scheduling-in-Multi-Core-Systems/
└── PSO_Scheduling_ES_proj_20.ipynb   # full implementation, experiments, and plots
```

---

## Problem setup

* Generate periodic tasks with **UUniFast** under a target hyper-period (LCM) using selectable divisors.
* Compute hyper-period and expand to jobs.
* **Baseline:**

  * Map tasks to cores via **Worst-Fit-Decreasing** (balances utilization).
  * Schedule jobs on each core with **EDF**.
  * Measure baseline **QoS, Energy, Success Rate**.
* **PSO search space:**

  * `N_tasks` integers (task→core),
  * `N_cores` reals (per-core frequency factors),
  * `Σ (hyper_period / period_i)` reals (per-job start-time decisions).
* **Objective/fitness:**

  ```
  fitness = Energy + 100000 * max(0, QoS_baseline - QoS_candidate)
  ```

  Energy ∝ Σ power · (freq³) · duration; QoS favors on-time completion with exponential lateness decay.

---

## Metrics

* **Energy Consumption**: sum over schedule reports using cubic DVFS model `energy ~ power × freq³ × time`.
* **QoS**: Σ priority × exp(−λ × delay), with delay = max(0, finish − deadline).
* **Success Rate**: fraction of jobs completed by their deadlines.

---

## Requirements

* Python 3.9+
* Packages: `numpy`, `matplotlib`, `tqdm`, `cupy` *(for GPU-accelerated PSO; optional)*

Install:

```bash
pip install numpy matplotlib tqdm cupy-cuda12x
# or a matching CuPy build for your CUDA version
```

> **No GPU?** You can run on CPU by aliasing CuPy to NumPy at the top of the notebook:
>
> ```python
> try:
>     import cupy as cp
> except Exception:
>     import numpy as cp  # fallback to CPU
> ```

---

## How to run

1. Open `PSO_Scheduling_ES_proj_20.ipynb` in Colab/Jupyter.
2. Adjust **global constants** near the top:

   * `TASK_NUMBERS` (e.g., 100), `MAX_HYPER_PERIOD` (e.g., 1000–1200),
   * PSO hyperparams: `W`, `C1`, `C2`, `SWARM_SIZE`, `ITERATIONS`,
   * QoS decay `LAMBDA_PARAM`.
3. Choose test cases (cores × utilization per core) in `TEST_CASES`.
4. Run all cells. The notebook prints **Baseline** vs **PSO** metrics and draws comparison plots for:

   * **Energy Consumption**
   * **QoS**
   * **Success Rate**

---

## Key implementation details

* **Task model**: `execution_time = round(utilization × period)`, per-task priority and power.
* **Baseline scheduler**: EDF with preemption, per-core ready queues.
* **Start-time decisions** (PSO): each job may be delayed within its slack `(period − execution_time)` using a decision ∈\[0,1]; jobs are then serialized (with resource contention) to form actual start/finish times.
* **Frequency scaling**: per-core factor ∈\[0.5, 1.0] impacts job duration and energy (cubic law).
* **Penalty** enforces **QoS ≥ baseline** in the objective (tunable via multiplier).

---

## Sample results (from the notebook)

**Energy reductions** for one configuration (`W=0.5, C1=1.0, C2=1.0`):

| Cores | Util. | Baseline Energy | PSO Energy |         Δ% |
| ----: | :---: | --------------: | ---------: | ---------: |
|     8 |  0.25 |         740,550 |    425,549 | **−42.6%** |
|     8 |  0.50 |         994,900 |    815,980 |     −18.0% |
|     8 |  0.70 |       1,496,950 |  1,325,929 |     −11.4% |
|    16 |  0.25 |       1,106,250 |    835,239 |     −24.5% |
|    16 |  0.50 |       1,926,600 |  1,682,445 |     −12.7% |
|    16 |  0.70 |       2,750,550 |  2,299,803 |     −16.4% |
|    32 |  0.25 |       2,120,500 |  1,586,213 |     −25.2% |
|    32 |  0.50 |       4,629,200 |  3,541,284 |     −23.5% |
|    32 |  0.70 |       5,518,950 |  4,114,233 |     −25.5% |

**Trade-offs:** In several runs, QoS is close to—but can be slightly below—baseline and success rate can drop at high utilizations. These are governed by `λ` and the penalty weight; increasing either pushes PSO toward higher QoS / success at the expense of smaller energy savings.

---

## Reproducing the figures

The notebook finishes with helper routines:

* `plot_comparison(...)` produces per-metric line charts (PSO vs Baseline) across utilizations for 8/16/32 cores.
* Multiple PSO settings are provided (four “runs” with different `W, C1, C2`) to study sensitivity.

---

## Tips & tuning

* **Faster runs**: reduce `TASK_NUMBERS`, `SWARM_SIZE`, or `ITERATIONS`.
* **Tighter timing**: increase `LAMBDA_PARAM` or the QoS penalty multiplier inside `evaluate_particle`.
* **Higher feasibility**: constrain start-time decisions or raise min frequency.
* **Exploration vs exploitation**: tune `W` (inertia), `C1` (cognitive), `C2` (social).

---

## License

MIT © Sina Beyrami

---

## Citation

If you use this code or ideas in academic work, please cite this repository and acknowledge the authors listed above.

---

*Questions or improvements? Feel free to open an issue or PR.*
