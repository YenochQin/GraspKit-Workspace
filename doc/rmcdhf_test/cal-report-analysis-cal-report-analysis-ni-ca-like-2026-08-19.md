---
title: "Calculation Report Analysis: Ni Ca-like e1_cv — NV versus optimized LBL"
slug: "cal-report-analysis-cal-report-analysis-ni-ca-like-2026-08-19"
artifact_type: cal_report_analysis
date_created: 2026-08-19
data_root: "data/rmcdhf_test_data"
data_dir: "inputs/Ni_Ca-like"
controlled_results: "results/orbital-iso-nica-632"
source_files:
  - "data/rmcdhf_test_data/inputs/Ni_Ca-like/e1_cv_NV"
  - "data/rmcdhf_test_data/inputs/Ni_Ca-like/even1_cv"
  - "data/rmcdhf_test_data/results/orbital-iso-nica-632"
  - "data/rmcdhf_test_data/results/matched-tf-rci-636"
---

# Calculation Report Analysis: Ni Ca-like `e1_cv` — NV versus optimized LBL

> **Evidence scope:** this retained report records the original NV versus
> optimized LBL observation. Its orbital-balance interpretation is a supported
> hypothesis, not a unique causal proof; current controlled conclusions are in
> [the evidence summary](./rmcdhf90_mpi_orbital_optimization_test_results_en.md).

## Scope

- Historical source identifier: `raw/cal_data/Ni_Ca-like/e1_cv` (the original
  report workspace is not part of this clone).
- Current reproducible inputs:
  [`data/rmcdhf_test_data/inputs/Ni_Ca-like`](../../data/rmcdhf_test_data/inputs/Ni_Ca-like).
- Current controlled evidence is indexed by
  [`rmcdhf_test/test/rmcdhf_orbopt/RESULTS.md`](../../rmcdhf_test/test/rmcdhf_orbopt/RESULTS.md);
  the original AS1--AS5 tables below remain a historical observation, not a
  replacement for those controlled runs.
- Runs analyzed: one directory-level run, with AS1--AS5 paired between `_NV` and ordinary LBL.
- Source files checked: all 20 CSV files in the directory: ten `rmcdhf` level tables and ten `rwfn` radial-orbital tables.
- Experimental reference: [NIST ASD v5.12, Ni IX levels](https://physics.nist.gov/cgi-bin/ASD/energy1.pl?de=0&spectrum=Ni+IX&submit=Retrieve+Data&units=0&format=0&output=0&page_size=15&multiplet_ordered=0&conf_out=on&term_out=on&level_out=on&unc_out=1&j_out=on&lande_out=on&perc_out=on&biblio=on&temp=), accessed 2026-08-19.

## Executive Summary

The `_NV` runs reproduce the experimentally constrained Ni IX term-internal energy separations much better than the ordinary optimized LBL runs. Using four comparisons that are independent of the NIST unknown offset $x$,

$$
E(^3F_3),\quad E(^3F_4),\quad E(^3P_2)-E(^1D_2),\quad E(^1G_4)-E(^1D_2),
$$

the AS5 mean absolute error is $399.82\ \mathrm{cm}^{-1}$ for `_NV` and $2135.30\ \mathrm{cm}^{-1}$ for ordinary LBL. Thus `_NV` lowers this robust spacing MAE by $81.3\%$ at AS5. The same ordering holds at every active-space layer: `_NV` MAE stays between $327.53$ and $399.82\ \mathrm{cm}^{-1}$, while optimized LBL remains between $1822.83$ and $2341.41\ \mathrm{cm}^{-1}$.

This is not a variational total-energy advantage. Ordinary LBL gives lower ground-state total energies by $0.0080$--$0.0137$ Hartree, as expected after orbital optimization, but its excitation-energy pattern is substantially worse. The result therefore indicates an orbital-balance problem: optimizing each newly added layer improves the state-averaged total energy while distorting differential correlation and fine-structure contributions across the target levels. Leaving the new orbitals at the Thomas--Fermi `rwfnestimate` shapes appears to preserve a more balanced representation for this model. This mechanism is a data-supported interpretation, not a uniquely established causal proof.

## Key Results

NIST gives $E(^3F_2)=0$, $E(^3F_3)=1880\ \mathrm{cm}^{-1}$, and $E(^3F_4)=4070\ \mathrm{cm}^{-1}$. It reports $E(^1D_2)=21900+x$, $E(^3P_2)=27160+x$, and $E(^1G_4)=35898+x\ \mathrm{cm}^{-1}$. NIST explains that $+x$ marks a level system whose internal relative positions are experimentally constrained but whose connection to the other level system is not known. Therefore the two safe spacings within that system are $5260$ and $13998\ \mathrm{cm}^{-1}$.

| AS | `_NV` MAE ($\mathrm{cm}^{-1}$) | LBL MAE ($\mathrm{cm}^{-1}$) | `_NV` RMSE ($\mathrm{cm}^{-1}$) | LBL RMSE ($\mathrm{cm}^{-1}$) | MAE reduction from `_NV` |
|---:|---:|---:|---:|---:|---:|
| 1 | 327.53 | 1822.83 | 351.97 | 1924.58 | 82.0% |
| 2 | 372.59 | 2341.41 | 413.61 | 2536.41 | 84.1% |
| 3 | 388.11 | 2257.63 | 437.02 | 2433.77 | 82.8% |
| 4 | 395.57 | 2184.70 | 448.82 | 2343.97 | 81.9% |
| 5 | 399.82 | 2135.30 | 455.76 | 2283.52 | 81.3% |

At AS5, the four offset-independent comparisons are:

| Quantity | NIST | `_NV` AS5 | `_NV` error | LBL AS5 | LBL error |
|---|---:|---:|---:|---:|---:|
| $E(^3F_3)$ | 1880 | 2020.52 | +140.52 | 272.25 | -1607.75 |
| $E(^3F_4)$ | 4070 | 4470.64 | +400.64 | 544.88 | -3525.12 |
| $E(^3P_2)-E(^1D_2)$ | 5260 | 4944.01 | -315.99 | 3417.63 | -1842.37 |
| $E(^1G_4)-E(^1D_2)$ | 13998 | 13255.87 | -742.13 | 12432.05 | -1565.95 |

All energies and errors in the table are in $\mathrm{cm}^{-1}$. `_NV` is closer to NIST in all four comparisons.

## Run-by-Run Notes

### `_NV`: unoptimized newly added layer

- AS1 gives the best robust aggregate result: MAE $327.53\ \mathrm{cm}^{-1}$.
- From AS1 to AS5, $E(^3F_3)$ is nearly stationary, changing only from $2021.63$ to $2020.52\ \mathrm{cm}^{-1}$; $E(^3F_4)$ changes from $4473.59$ to $4470.64\ \mathrm{cm}^{-1}$.
- The main drift occurs in higher term separations. $E(^1G_4)-E(^1D_2)$ decreases from $13515.40$ to $13255.87\ \mathrm{cm}^{-1}$, moving away from the NIST value $13998\ \mathrm{cm}^{-1}$ as the active space grows.
- The result is stable rather than monotonically improving: adding more unoptimized layers slightly worsens the four-spacing MAE after AS1.

### Ordinary optimized LBL

- AS1 already underestimates the $^3F$ fine-structure intervals severely: $589.98$ and $1287.21\ \mathrm{cm}^{-1}$ instead of $1880$ and $4070\ \mathrm{cm}^{-1}$.
- AS2 is the clearest failure point. The $^3F_4$ and $^3F_3$ levels appear at $62.36$ and $68.08\ \mathrm{cm}^{-1}$, respectively, reversing the experimental order and nearly collapsing the multiplet.
- AS3 restores the experimental order, and AS3--AS5 gradually enlarge the two intervals, but AS5 remains far too compressed at $272.25$ and $544.88\ \mathrm{cm}^{-1}$.
- The group-internal spacings involving $^1D_2$, $^3P_2$, and $^1G_4$ are also systematically too small. At AS5 their errors remain $-1842.37$ and $-1565.95\ \mathrm{cm}^{-1}$.

## Cross-Run Comparison

The total-energy and excitation-energy criteria point in opposite directions. The ordinary LBL ground-state total energy is below `_NV` by $0.007997$, $0.013403$, $0.013656$, $0.013468$, and $0.013260$ Hartree for AS1--AS5, respectively. The optimized orbitals therefore do lower the variational objective, but that lowering is not balanced across the nine target states and does not translate into better relative energies.

The `rwfn` files confirm that orbital optimization is not a small perturbation. Using the normalized radial-spinor overlap

$$
S_{ab}=\frac{\int \left(P_aP_b+Q_aQ_b\right)\,dr}{\sqrt{\int(P_a^2+Q_a^2)\,dr\int(P_b^2+Q_b^2)\,dr}},
$$

several newly added orbital pairs have small $|S_{ab}|$: at AS2, the new $5s$, $5p$, $5d$, $5f$, and $5g$ pairs have absolute overlaps approximately $0.276$, $0.313$, $0.309$, $0.117$, and $0.259$; by AS5, the corresponding new $n=8$ overlaps are approximately $0.039$--$0.065$. Some relativistic partner columns remain numerically identical, so the overlap evidence is orbital-specific rather than uniform across every column. These large shape differences are consistent with the observed spectral sensitivity to whether the new layer is optimized.

## Anomalies and Failure Signals

- The AS2 ordinary-LBL $^3F$ inversion and near collapse is the strongest failure signal.
- Neither workflow is converging monotonically toward NIST under the robust metric. `_NV` is best at AS1 and drifts moderately upward in error; ordinary LBL is worst at AS2 and then recovers slowly without approaching `_NV` accuracy by AS5.
- A naive comparison to the printed NIST values $21900+x$, $27160+x$, and $35898+x$ as absolute energies would be invalid because $x$ is unknown. Only differences within that marked system remove $x$.
- NIST provides no experimental energies for $^3P_0$, $^3P_1$, or $^1S_0$ on the retrieved Ni IX page, so their absolute accuracy cannot be assessed here.

## Interpretation

The evidence supports the user's observation: for this `e1_cv` model, retaining the Thomas--Fermi initial orbitals for the newly expanded layer gives markedly more realistic relative energies than optimizing that layer in the ordinary LBL procedure.

A plausible explanation is unbalanced orbital relaxation. The ordinary LBL optimization is free to lower the weighted total energy by changing diffuse/correlation orbitals substantially. If different terms exploit this flexibility unequally, common-mode variational improvement does not cancel cleanly in excitation energies. The resulting differential error is especially visible in the $^3F_J$ fine structure. `_NV` restricts that relaxation and may therefore preserve cancellation of errors between levels, even though its absolute total energy is higher.

The current files do not isolate whether the dominant source is the extended-optimal-level weighting, the selected optimization states, orbital orthogonality constraints, convergence to a local stationary point, or excessive coupling of the new layer to particular terms. The data demonstrate association and identify where the spectrum deteriorates, but they do not by themselves select one unique mechanism.

## Limits and Unknowns

- The run directory contains final `rmcdhf` level tables and radial orbitals but no optimization logs, EOL weights, convergence thresholds, CSF counts, or state-selection metadata.
- NIST's $+x$ system prevents absolute placement of $^1D_2$, $^3P_2$, and $^1G_4$ relative to the ground multiplet. Only their internal spacings are used in the primary metric.
- The `rmcdhf` outputs alone do not separate valence, core--valence, Breit, QED, or later RCI corrections.
- Radial overlaps describe shape changes but do not measure each orbital's contribution to a particular energy interval.

## Recommended Next Checks

1. Re-run AS2 with saved optimization logs and per-iteration energies, because AS2 is where the ordinary LBL $^3F$ multiplet collapses and reverses order.
2. Test selective optimization by angular symmetry: optimize one of $ns$, $np$, $nd$, $nf$, or $ng$ at a time while leaving the others at the Thomas--Fermi guess, then monitor the same four offset-free NIST spacings.
3. Vary the EOL state weights and optimization-state set. Report both total-energy lowering and the four-spacing MAE so variational improvement cannot mask spectral degradation.
4. Add a frozen-orbital control in which the new orbitals are orthogonalized but not variationally relaxed, separating the effect of orthogonality from radial optimization.
5. Carry both orbital sets into an otherwise identical RCI calculation. If the `_NV` advantage survives, the imbalance is likely embedded in the orbital basis; if it disappears, later correlation/Breit/QED treatment is compensating for it.
