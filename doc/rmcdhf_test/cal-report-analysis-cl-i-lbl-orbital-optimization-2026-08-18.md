---
title: "Calculation Report Analysis: Cl I LBL orbital optimization"
slug: "cal-report-analysis-cl-i-lbl-orbital-optimization-2026-08-18"
artifact_type: cal_report_analysis
date_created: 2026-08-18
data_root: "data/rmcdhf_test_data"
data_dir: "inputs/Cl_I"
controlled_results: "results/rmcdhf-full-20260901d"
source_files:
  - "data/rmcdhf_test_data/inputs/Cl_I/o1_vv_no_varied"
  - "data/rmcdhf_test_data/inputs/Cl_I/o1_vv1"
  - "data/rmcdhf_test_data/results/rmcdhf-full-20260901d"
  - "data/rmcdhf_test_data/results/cl-as-chain-580"
---

# Calculation Report Analysis: Cl I LBL orbital optimization

> **Evidence scope:** this is the retained source report for the original
> no-varied versus one-sided-optimized observation. It establishes the energy
> and radial contrast, but does not by itself identify the unique cause. The
> controlled B1/B2/B3 conclusion is maintained in
> [the evidence summary](./rmcdhf90_mpi_轨道优化测试结果.md).

## Scope

- Historical source identifier: `raw/cal_data/Cl_I/` (the original report
  workspace is not part of this clone).
- Current reproducible inputs:
  [`data/rmcdhf_test_data/inputs/Cl_I`](../../data/rmcdhf_test_data/inputs/Cl_I).
- Current B0--B3 controlled outputs:
  [`rmcdhf-full-20260901d`](../../data/rmcdhf_test_data/results/rmcdhf-full-20260901d);
  current balanced AS1--AS5 chain:
  [`cl-as-chain-580`](../../data/rmcdhf_test_data/results/cl-as-chain-580).
- Runs analyzed: AS1--AS5, each with `_no_varied` and optimized variants
- Source files checked: all 10 RMCDHF level tables and all 10 RWFN radial-orbital tables in `Cl_I`
- User-provided run meaning: `_no_varied` means that orbitals newly added by the current LBL active-space expansion were not optimized; their `rwfnestimate` Thomas--Fermi initial guesses were retained as the final radial orbitals.

The signed fine-structure interval used below is

$$
\Delta E_{\mathrm{FS}} = E({}^{2}P_{1/2})-E({}^{2}P_{3/2}).
$$

Thus, $\Delta E_{\mathrm{FS}}>0$ means that $J=3/2$ is the ground level, while $\Delta E_{\mathrm{FS}}<0$ means that the two levels are inverted.

## Executive Summary

The data directly confirm the reported contrast. Every `_no_varied` calculation gives the same, correct $J$ order, ${}^{2}P_{3/2}<{}^{2}P_{1/2}$, and the interval converges smoothly from $923.81$ to $921.13\ \mathrm{cm}^{-1}$ over AS1--AS5. Every optimized calculation gives the opposite order, ${}^{2}P_{1/2}<{}^{2}P_{3/2}$; its signed interval improves from $-2479.51$ to $-1739.74\ \mathrm{cm}^{-1}$ but never approaches zero or recovers the correct sign.

The inversion is not caused by ambiguous level identification. The ASF labels remain ${}^{2}P_{J}$ and the Landé factors remain close to $g_{3/2}=4/3$ and $g_{1/2}=2/3$ in both sequences. It is an energy-ordering failure between identifiable fine-structure partners.

The optimized RWFN files expose a strong and systematic orbital imbalance. Spectroscopic orbitals through $3p$ are identical between the two sequences, but selected virtual orbitals contract drastically. More importantly, for a given $l>0$, the CSV orbital without a trailing `-` changes while its `-` partner remains numerically identical. In GRASP-style notation this corresponds to optimizing the $j=l+1/2$ component while leaving the $j=l-1/2$ component at its initial form. The $s$ virtuals are also optimized. This one-sided relativistic-orbital relaxation is the clearest diagnostic associated with the wrong $J$ order.

The available CSVs establish association, not the internal cause in the LBL selection code. They support the practical conclusion that retaining the Thomas--Fermi estimates is safer for this run than applying the present optimization route, and they motivate an explicit audit of the varied-orbital list and balanced optimization of both relativistic partners.

## Key Results

### Fine-structure convergence

| AS | `_no_varied` ground | $\Delta E_{\mathrm{FS}}$ (`_no_varied`) | optimized ground | $\Delta E_{\mathrm{FS}}$ (optimized) |
|---:|---|---:|---|---:|
| 1 | ${}^{2}P_{3/2}$ | $+923.81\ \mathrm{cm}^{-1}$ | ${}^{2}P_{1/2}$ | $-2479.51\ \mathrm{cm}^{-1}$ |
| 2 | ${}^{2}P_{3/2}$ | $+922.89\ \mathrm{cm}^{-1}$ | ${}^{2}P_{1/2}$ | $-1894.98\ \mathrm{cm}^{-1}$ |
| 3 | ${}^{2}P_{3/2}$ | $+921.34\ \mathrm{cm}^{-1}$ | ${}^{2}P_{1/2}$ | $-1830.21\ \mathrm{cm}^{-1}$ |
| 4 | ${}^{2}P_{3/2}$ | $+921.20\ \mathrm{cm}^{-1}$ | ${}^{2}P_{1/2}$ | $-1771.04\ \mathrm{cm}^{-1}$ |
| 5 | ${}^{2}P_{3/2}$ | $+921.13\ \mathrm{cm}^{-1}$ | ${}^{2}P_{1/2}$ | $-1739.74\ \mathrm{cm}^{-1}$ |

The `_no_varied` interval changes by only $2.68\ \mathrm{cm}^{-1}$ across the full active-space ladder and by $0.07\ \mathrm{cm}^{-1}$ from AS4 to AS5. This is internally smooth convergence with the same sign. The optimized interval changes by $739.77\ \mathrm{cm}^{-1}$ between AS1 and AS5, but remains inverted by a large margin.

### Absolute-energy lowering is strongly $J$ dependent

| AS | Lowering of $E({}^{2}P_{3/2})$ | Lowering of $E({}^{2}P_{1/2})$ | Excess lowering of $J=1/2$ |
|---:|---:|---:|---:|
| 1 | $0.0318349\ E_h$ | $0.0473416\ E_h$ | $0.0155067\ E_h$ |
| 2 | $0.0371827\ E_h$ | $0.0500218\ E_h$ | $0.0128391\ E_h$ |
| 3 | $0.0378205\ E_h$ | $0.0503575\ E_h$ | $0.0125370\ E_h$ |
| 4 | $0.0377215\ E_h$ | $0.0499882\ E_h$ | $0.0122667\ E_h$ |
| 5 | $0.0375222\ E_h$ | $0.0496460\ E_h$ | $0.0121238\ E_h$ |

Optimization lowers both total energies, as expected from added variational freedom, but lowers $J=1/2$ substantially more. At AS5 the differential lowering is $0.0121238\ E_h$, or about $2660.9\ \mathrm{cm}^{-1}$. This is exactly the scale required to turn the unoptimized $+921.13\ \mathrm{cm}^{-1}$ interval into the optimized $-1739.74\ \mathrm{cm}^{-1}$ interval. A lower total energy therefore does not validate the fine-structure balance: the decisive quantity is the differential correlation/relaxation energy between the two $J$ blocks.

### Radial-orbital changes

For each common orbital, the radial overlap was evaluated as

$$
S_{ab}=\frac{\int [P_a(r)P_b(r)+Q_a(r)Q_b(r)]\,dr}
{\sqrt{\int[P_a^2(r)+Q_a^2(r)]\,dr\int[P_b^2(r)+Q_b^2(r)]\,dr}},
$$

and the mean radius as

$$
\langle r\rangle=\frac{\int r[P^2(r)+Q^2(r)]\,dr}{\int[P^2(r)+Q^2(r)]\,dr}.
$$

Only $|S|$ is interpreted because an orbital's overall phase is conventional.

| Orbital | First visible at |         $ |        S |       $ | $\langle r\rangle$ `_no_varied`                    | $\langle r\rangle$ optimized | Diagnostic |
| ------- | ---------------: | --------: | -------: | ------: | -------------------------------------------------- | ---------------------------- | ---------- |
| $3d$    |              AS1 | $0.76874$ |  $3.591$ | $1.796$ | strong contraction; $3d-$ unchanged                |                              |            |
| $4s$    |              AS1 | $0.58546$ |  $4.629$ | $2.068$ | strong contraction                                 |                              |            |
| $4p$    |              AS1 | $0.55779$ |  $6.000$ | $2.394$ | strong contraction; $4p-$ unchanged                |                              |            |
| $4d$    |              AS2 | $0.04254$ | $12.127$ | $2.142$ | almost complete shape replacement; $4d-$ unchanged |                              |            |
| $4f$    |              AS2 | $0.01772$ | $16.565$ | $1.875$ | almost complete shape replacement; $4f-$ unchanged |                              |            |
| $5s$    |              AS2 | $0.13283$ | $11.208$ | $2.149$ | severe contraction                                 |                              |            |
| $5p$    |              AS2 | $0.13533$ | $14.047$ | $2.217$ | severe contraction; $5p-$ unchanged                |                              |            |
| $5d$    |              AS3 | $0.00425$ | $23.307$ | $2.126$ | near-orthogonal shapes; $5d-$ unchanged            |                              |            |
| $6g$    |              AS4 | $0.00038$ | $43.265$ | $1.924$ | near-orthogonal shapes; $6g-$ unchanged            |                              |            |
| $7d$    |              AS5 | $0.00006$ | $54.521$ | $0.386$ | extreme inward collapse; $7d-$ unchanged           |                              |            |
| $8p$    |              AS5 | $0.00137$ | $56.168$ | $2.070$ | near-orthogonal shapes; $8p-$ unchanged            |                              |            |

The occupied $1s$, $2s$, $2p-$, $2p$, $3s$, $3p-$, and $3p$ orbitals have $|S|=1$ and identical mean radii between the two sequences. The virtual `-` orbitals, including $3d-$, $4p-$, $4d-$, $4f-$, $5p-$, and all later `-` partners, are also numerically unchanged. The changed virtual orbitals are carried unchanged into later AS files after their first optimization, so the pattern is cumulative rather than a fresh unrelated fluctuation at each AS.

## Run-by-Run Notes

### AS1

The failure begins immediately. `_no_varied` gives ${}^{2}P_{3/2}$ as the ground level and $923.81\ \mathrm{cm}^{-1}$ for the interval, whereas optimization gives ${}^{2}P_{1/2}$ as the ground level with an inverted $2479.51\ \mathrm{cm}^{-1}$ separation. The first layer already changes $3d$, $4s$, and $4p$ strongly, while $3d-$ and $4p-$ stay fixed.

### AS2

The unoptimized interval remains stable at $922.89\ \mathrm{cm}^{-1}$. Optimization adds severe contraction of $4d$, $4f$, $5s$, and $5p$, with overlaps of only $0.018$--$0.135$ relative to their Thomas--Fermi forms. The optimized order remains inverted at $-1894.98\ \mathrm{cm}^{-1}$.

### AS3--AS5

The `_no_varied` sequence converges monotonically to $921.13\ \mathrm{cm}^{-1}$. In the optimized sequence the magnitude of the inversion decreases monotonically, but the sign remains wrong. Newly optimized high-$n$ orbitals commonly change from very diffuse Thomas--Fermi functions with $\langle r\rangle\sim20$--$65$ to compact functions with $\langle r\rangle\sim1$--$2$; $7d$ is the most extreme case, reaching $\langle r\rangle=0.386$.

## Cross-Run Comparison

Three facts move together across the paired runs:

1. Suppressing new-orbital optimization preserves a stable positive fine-structure interval.
2. Enabling the present optimization route produces large radial changes in only one member of many relativistic orbital pairs.
3. The same route gives $J=1/2$ an extra $0.012$--$0.016\ E_h$ of lowering relative to $J=3/2$, which reverses the level order.

This comparison rules out active-space size alone as the sufficient explanation: the same AS1--AS5 labels and final orbital inventories give opposite signs depending on whether the new orbital shapes are varied.

## Anomalies and Failure Signals

- **Immediate sign reversal:** the wrong order is already present at AS1 rather than emerging only in a large, noisy active space.
- **Relativistic-pair imbalance:** $nl$ changes while the corresponding $nl-$ stays exactly at the Thomas--Fermi form for many $p$, $d$, $f$, and $g$ pairs.
- **Extreme contraction:** several diffuse estimates are replaced by compact functions with nearly zero overlap with the estimates. The $7d$ mean radius contracts by a factor of about $141$.
- **Variational-energy trap:** optimized total energies are lower, but the energy gain is not balanced between the $J$ levels.
- **Persistent inversion:** adding layers reduces the magnitude of the error but does not restore the sign, so apparent numerical convergence of the optimized series would be misleading.

## Interpretation

The source files directly support the following diagnosis:

> The incorrect Cl I fine-structure order is associated with the LBL new-orbital optimization step, not with the mere inclusion of the corresponding active-space orbitals. The present optimization produces strongly contracted virtual orbitals and unbalanced relaxation between relativistic partners, which is accompanied by excessive stabilization of the ${}^{2}P_{1/2}$ level.

The most plausible mechanism is that the varied-orbital selection or optimization schedule is not treating both $j=l\pm1/2$ partners consistently. Such one-sided flexibility can change spin-dependent correlation differently in the two total-$J$ blocks. This is a mechanism-level interpretation of the observed RWFN asymmetry, not yet proof of the exact software-control-flow defect. Proving that defect requires the LBL input, optimization log, and the actual varied-orbital list for each iteration.

The result also explains why the Thomas--Fermi guesses can give a better $J$ order despite higher total energies: their radial shapes are less variationally optimized, but they preserve a more balanced common basis across the fine-structure partners. Correct ordering here is controlled by a small difference between two large total energies; balanced approximation can outperform unbalanced energy minimization.

## Limits and Unknowns

- The directory contains final RMCDHF and RWFN CSVs, but no LBL input deck, SCF iteration log, orbital-selection trace, state weights, convergence thresholds, or orbital Lagrange multipliers.
- The data do not show whether the one-sided `-`/non-`-` pattern is caused by an input choice, a wrapper selector, an index overshoot, or behavior inside the atomic-structure program.
- No fixed-orbital cross calculation is present that combines the `_no_varied` orbitals with the optimized calculation settings, or vice versa.
- Mean radius and overlap demonstrate orbital reorganization but do not decompose the Hamiltonian contribution responsible for the differential $J$ lowering.
- The report assesses internal consistency and the user-specified correct order. It does not use an external experimental reference value as an additional accuracy benchmark.

## Recommended Next Checks

1. **Audit the exact varied-orbital list.** Save the LBL-generated optimization input at every AS step. Verify that every intended pair $nl-$ and $nl$ appears together and that no orbital index has crossed the boundary of the newly appended layer.

2. **Run an AS1 pair-balanced matrix.** Starting from the same Thomas--Fermi orbitals, compare: all new orbitals frozen; $3d-/3d$ optimized together; $4p-/4p$ optimized together; $4s$ only; and all three balanced groups together. AS1 is sufficient because the inversion already appears there.

3. **Separate orbital-shape and CSF-space effects.** Freeze the `_no_varied` AS1 and AS5 bases and rerun the same CI/RCI spaces used by the optimized sequence. Then freeze the optimized bases and repeat identical diagonalizations. Keep the CSF lists and Hamiltonian options byte-identical across each pair.

4. **Use common EOL weights for both levels.** Optimize ${}^{2}P_{3/2}$ and ${}^{2}P_{1/2}$ with explicitly recorded, symmetric state weights. Report both total energies and the signed interval after every SCF macro-iteration.

5. **Add orbital safeguards.** For each newly varied orbital, record normalization, node count, $\langle r\rangle$, overlap with its initial guess, and overlap with lower-$n$ orbitals of the same symmetry. Flag changes such as $|S|<0.1$ or order-of-magnitude radius contraction for review rather than accepting convergence solely from total energy.

6. **Use the `_no_varied` route as the current baseline.** Until the pair-selection behavior is explained, the AS5 `_no_varied` result is the internally stable reference: correct $J$ order and a converged interval of $921.13\ \mathrm{cm}^{-1}$.
