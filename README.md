# AtomiQ Quantum Hackathon

# Hybrid quantum-classical term-deposit campaign

author: Sergey Grigorovich

A three-stage pipeline predicts subscription probabilities, selects six customers from the ten highest-ranked candidates, and estimates the probability of at least three subscriptions. The exported selection matches the exact classical optimum: **24.30 expected net-value units**, with an exact campaign success probability of **27.59%**.

## Files and execution

| File | Purpose |
| --- | --- |
| `01_stage1_prediction.ipynb` | Read the workbook, compare models, fit the frozen blend and rank 100 test customers. |
| `02_stage2_selection.ipynb` | Select six from the top ten using QAOA p=1, 2, 3; compare with exact enumeration. |
| `03_stage3_simulation.ipynb` | Encode the six probabilities and compare quantum sampling with exact and Monte Carlo results. |
| `stage1_results.csv` | 100 customer IDs, predicted probabilities, ranks and top-10 flags. |
| `stage2_results.json` | Candidate inputs, exported p=3 selection, constraints, objective, all QAOA runs and exact optimum. |
| `stage3_results.json` | Selected IDs, qubit order, probabilities, angles, measurement counts, estimates and errors. |
| `stage1_comparison.csv` | Cross-validation metrics for all eight model configurations. |
| `stage2_comparison.csv` | Objective gaps, sampling probabilities, counts, optimizer effort, runtimes and circuit sizes. |
| `stage3_comparison.csv` | Exact, statevector, quantum-sampling and Monte Carlo estimates with their errors. |
| `requirements.txt` | Pinned dependencies for all notebooks, JupyterLab and circuit drawing. |

Input: the corrected participant workbook, including test customer IDs, at `data/bank_campaign_participant_release.xlsx`.

Use Python 3.12. From the project folder, install dependencies and open JupyterLab:

```bash
python -m pip install -r requirements.txt
jupyter lab
```

Select the Python kernel from that environment. Run all cells in notebooks **01 → 02 → 03**, in order. Each notebook writes its result file and comparison table in the project folder. Stage 2 reads Stage 1 predictions; Stage 3 reads those predictions and the exported Stage 2 selection. Keep the three primary result files at the submission root.

Numbers below are rounded for readability; exported files retain full precision. Bold marks the best comparable metrics, including ties. Timings and optimizer evaluation counts are from the supplied comparison tables.

## Stage 1: subscription prediction

Training uses 300 labelled customers, including 35 subscribers; test labels for the 100 prediction customers are hidden. Classical references use `sklearn.linear_model.LogisticRegression` and `sklearn.ensemble.GradientBoostingClassifier`. Logistic regression uses C=1, standardized numeric inputs and one-hot categorical inputs. Boosting uses 100 trees, depth 2, learning rate 0.05 and minimum leaf size 5. The notebook also compares both models with `campaign` omitted.

The quantum model omits `campaign`, applies simple log transforms to skewed inputs, preserves the never-contacted `pdays` indicator, then uses scaling, one-hot encoding and four PCA components. `FidelityStatevectorKernel` and `QSVC` use a ZZ feature map with two linear repetitions, alpha=2, C=1 and balanced class weights. Three-fold sigmoid calibration wraps the whole pipeline. PCA components are mapped to `0 .. s*pi/2`, comparing s=1, 0.5 and 0.25. This bandwidth comparison is motivated by Shaydulin and Wild [2]; it is not a replication of their experiments.

Validation uses `StratifiedKFold`: three folds repeated for seeds 42–46. Encoders, scalers, PCA and calibration fit only within training partitions. Probability metrics use pooled out-of-fold predictions for each seed, then average across seeds. P@10 and R@10 are averaged over the 100-customer validation folds. **AP** means `average_precision_score`; a separate trapezoidal PR-AUC is not computed.

| Model | ROC-AUC ↑ | AP ↑ | P@10 ↑ | R@10 ↑ | Brier ↓ | Log loss ↓ |
| --- | --- | --- | --- | --- | --- | --- |
| Logistic, all variables | 0.555 | 0.174 | 0.153 | 0.132 | 0.110 | 0.395 |
| Boosting, all variables | 0.610 | 0.235 | 0.260 | 0.225 | 0.103 | 0.365 |
| Logistic, no campaign | 0.559 | 0.177 | 0.153 | 0.132 | 0.110 | 0.393 |
| Boosting, no campaign | 0.614 | 0.233 | **0.287** | **0.248** | 0.103 | 0.365 |
| Quantum SVM, scale 1 | 0.568 | 0.149 | 0.133 | 0.114 | 0.103 | 0.360 |
| Quantum SVM, scale 0.5 | 0.603 | 0.178 | 0.193 | 0.168 | 0.102 | 0.355 |
| Quantum SVM, scale 0.25 | 0.632 | 0.199 | 0.193 | 0.166 | 0.101 | 0.351 |
| 50:50 blend (submitted) | **0.645** | **0.255** | 0.273 | 0.237 | **0.099** | **0.345** |

The frozen submission is the **50:50 probability average of full-input boosting and the scale-0.25 quantum SVM**. It improves the reported ROC-AUC, AP, Brier score and log loss over boosting alone; boosting without `campaign` has the highest P@10 and R@10. After refitting on all training rows, test predictions are sorted by decreasing probability, breaking ties by smaller customer ID. The first ten become Stage 2 candidates.

## Stage 2: constrained customer selection

For each candidate, expected net value is `v_i = p_i * expected_deposit_value_i - contact_cost_i`. Select exactly six, maximizing their total value, with cost ≤25 and capacity ≤22. Exact enumeration checks all **210** six-customer subsets.

For this candidate pool, the six largest costs sum to 17 and the six largest capacities sum to 18. Both inequalities are therefore redundant, and all 210 subsets are feasible. They are still checked when decoding measured states.

The quantum formulation uses ten decision qubits and no auxiliary qubits. With `u_i = v_i / max(abs(v))`, minimize:

`E(x) = -sum(u_i*x_i) + 1.1*(sum(x_i)-6)^2`

Since each absolute scaled value is at most one, correcting a cardinality error of r bits changes the linear term by at most r but removes penalty 1.1*r². Thus an incorrect-cardinality state cannot be the global minimum. This follows the squared cardinality-penalty pattern in the Qiskit portfolio tutorial [3].

`qaoa_ansatz` supplies Hadamard initialization and the global X mixer. COBYLA optimizes exact statevector expectations with **three starts and a cap of 1,000 function evaluations per start at every p**. For higher p, the first start extends the previous best angles with a zero layer. The lowest-energy run is sampled with the unchanged `StatevectorSampler` default of **1,024 shots**. We export the highest-value feasible measured bitstring; invalid samples remain in the saved counts.

| Method | Best net value ↑ | Gap ↓ | P(feasible) ↑ | P(optimum) ↑ |
| --- | --- | --- | --- | --- |
| Exact classical | **24.30** | **0.00** | — | — |
| QAOA p=1 | **24.30** | **0.00** | 53.74% | 0.318% |
| QAOA p=2 | **24.30** | **0.00** | 60.83% | 0.455% |
| QAOA p=3 (exported) | **24.30** | **0.00** | **61.86**% | **0.850**% |

P(feasible) and P(optimum) are exact probabilities from each optimized statevector. The classical row is an exact search, so sampling probabilities do not apply.

| Method | Total evaluations ↓ | Runtime, s ↓ | Feasible shots / 1,024 ↑ | Optimal shots / 1,024 ↑ |
| --- | --- | --- | --- | --- |
| Exact classical | — | **0.006** | — | — |
| QAOA p=1 | **134** | 1.83 | 544 | 5 |
| QAOA p=2 | 1967 | 14.60 | 641 | 2 |
| QAOA p=3 (exported) | 2732 | 29.48 | **642** | **9** |

Evaluation totals sum all three starts; the 1,000 cap applies separately to each start. Every winning run reports convergence. All depths found the same exact optimum. **p=3 is the exported run**: it has the highest feasible and optimal-state probabilities, at greater runtime and circuit cost. Individual shot counts can fluctuate even when exact probabilities improve. Exact enumeration is much faster for this small instance.

The exported bitstring is `0001011111`, displayed as q9...q0; candidate i, in Stage 1 rank order, maps to qubit i. It selects six customers with **cost 12/25**, **capacity 14/22**, **net value 24.30** and **zero optimality gap**. Their IDs and Stage 3 mapping appear below.

## Stage 3: probability of at least three subscriptions

Use one qubit per selected customer, starting at zero. Apply `Ry(theta_i)` with `theta_i = 2*arcsin(sqrt(p_i))`, so measuring 1 has probability p_i. Separate rotations encode independent responses; no entanglement or auxiliary qubits are needed.

| Stage 3 qubit | Customer ID | Subscription probability |
| --- | --- | --- |
| q0 | C0081 | 46.52% |
| q1 | C0031 | 45.10% |
| q2 | C0079 | 29.81% |
| q3 | C0170 | 24.34% |
| q4 | C0327 | 24.14% |
| q5 | C0256 | 18.14% |

Qiskit displays outcomes as q5...q0. Count a success when a bitstring contains at least three ones. The classical reference is `scipy.stats.poisson_binom.sf(2, p)`; the second exact check sums the matching statevector probabilities. Quantum sampling and classical Monte Carlo each use 1,024 draws.

| Method | P(at least 3) | Shots / trials | Absolute error, percentage points ↓ |
| --- | --- | --- | --- |
| Exact classical | 27.593% | — | Reference |
| Exact statevector | 27.593% | — | **<0.001** |
| Quantum sampling | 27.734% | 1,024 | **0.142** |
| Classical Monte Carlo | 30.469% | 1,024 | 2.876 |

The statevector result agrees with the exact classical calculation to floating-point precision. Among the two sampled estimates, quantum sampling has the smaller error in this run: **0.142 percentage points**. It records **284 successes out of 1,024**, versus 312 for Monte Carlo. Both sampled methods have theoretical standard error **1.40 percentage points**. A closer estimate for one fixed seed is sampling variation, not evidence of quantum advantage.

**Business recommendation:** contact the six customers listed above to maximize expected net value within the challenge constraints. The model implies **1.88 expected subscribers** and a **27.59%** chance of reaching three subscriptions. That target is therefore unlikely under the model. The selection objective is expected net value, not the probability of reaching the subscription target.

## Execution, resources and limitations

All quantum computations use ideal Qiskit CPU statevector simulation. Stage 1 computes exact kernels without shot sampling. Stages 2 and 3 include exact and noiseless finite-shot results. No physical backend, device noise model or error mitigation is used; **physical qubit count: N/A (simulation only)**. Hardware noise robustness has not been evaluated.

| Circuit | Logical qubits | Depth, original → transpiled | 1-qubit gates, original → transpiled | 2-qubit gates, original → transpiled | SWAPs, both |
| --- | --- | --- | --- | --- | --- |
| Stage 1 ZZ feature map | 4 | 19 → 23 | 22 → 38 | 12 → 12 | 0 |
| QAOA p=1 | 10 | 20 → 59 | 30 → 125 | 45 → 90 | 0 |
| QAOA p=2 | 10 | 32 → 94 | 50 → 220 | 90 → 180 | 0 |
| QAOA p=3 | 10 | 44 → 129 | 70 → 315 | 135 → 270 | 0 |
| Stage 3 | 6 | 1 → 4 | 6 → 24 | 0 → 0 | 0 |

Counts exclude measurements. Transpilation uses basis `rz`, `sx`, `x`, `cx`, optimization level 1 and seed 42, with no hardware target or coupling map. Stage 1 counts describe the feature-map circuit. QAOA counts describe the bound optimized circuits; their transpiled two-qubit gates are CX gates.

The fixed seed is **42** for stochastic model settings, calibration folds, optimizer initialization, samplers, transpilation and Monte Carlo. Outer cross-validation uses **42, 43, 44, 45, 46**. Package versions are recorded in `requirements.txt` and notebook outputs. Floating-point and platform differences can change optimizer paths and runtimes despite fixed seeds.

Model choices reuse a small, imbalanced training set and its validation folds, so these are development metrics rather than an untouched performance estimate. Hidden-test metrics are unavailable. Stage 1 probability errors propagate into customer rankings, the Stage 2 objective and the Stage 3 outcome probability. Zero optimization gap proves optimality only for the supplied predicted probabilities; exact Stage 3 agreement validates encoding and arithmetic, not calibration or the independence assumption. The sampling standard error excludes this model uncertainty. The simulated quantum components demonstrate the required workflow; no computational quantum advantage is claimed.

## Sources

Submission follows the organizer rulebook.

1. [Qiskit Machine Learning: Quantum Kernel Machine Learning](https://stfc.github.io/qiskit-machine-learning/tutorials/03_quantum_kernel.html) — fidelity kernels and QSVC.
2. Shaydulin and Wild (2022), [Importance of kernel bandwidth in quantum machine learning](https://doi.org/10.1103/PhysRevA.106.042407) — motivation for the input-range comparison.
3. [Qiskit Finance: Portfolio Optimization](https://qiskit-community.github.io/qiskit-finance/tutorials/01_portfolio_optimization.html) — squared cardinality penalties.
4. [Qiskit qaoa_ansatz](https://quantum.cloud.ibm.com/docs/en/api/qiskit/qiskit.circuit.library.qaoa_ansatz), [StatevectorSampler](https://quantum.cloud.ibm.com/docs/en/api/qiskit/qiskit.primitives.StatevectorSampler), and [SciPy COBYLA](https://docs.scipy.org/doc/scipy-1.17.0/reference/optimize.minimize-cobyla.html) — Stage 2 circuit, sampling and optimization.
5. [Qiskit Ry gate](https://quantum.cloud.ibm.com/docs/en/api/qiskit/qiskit.circuit.library.RYGate) and [SciPy poisson_binom](https://docs.scipy.org/doc/scipy-1.17.0/reference/generated/scipy.stats.poisson_binom.html) — Stage 3 encoding and exact reference.
