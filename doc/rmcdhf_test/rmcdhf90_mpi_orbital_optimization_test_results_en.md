# `rmcdhf90_mpi` Orbital-Optimization Evidence Summary

## 1. Conclusion

The strongest empirical explanation for the correct level ordering is:
**do not inherit an AS baseline contaminated by incorrect one-sided
optimization, and include both `nl-` and `nl` partners of each $(n,l)$ pair in
the varied list.** This removes the $J$-dependent differential-energy imbalance
caused by relaxing only the positive-sign `nl` orbital.

"Close to NIST" has a narrow meaning here: the balanced path reduces errors of
several thousand to more than ten thousand `cm^-1` from the incorrect
one-sided path to tens or hundreds of `cm^-1`. The evidence does not show that
balanced optimization generally improves NIST accuracy. Ni I Job 645 and the
balanced Cl I result remain slightly worse than their fixed-TF/no-varied
baselines.

As of 2026-09-16, the partner transaction has passed program-level regression:
partner candidates are generated from one snapshot, use common damping, and
commit atomically only after shared post-`ORTHY` checks. A failed group is
restored and retried from the same snapshot. This establishes the intended
algorithmic semantics, but does not replace per-AS physical acceptance.

## 2. B1/B2/B3: single-factor evidence for partner selection

B1 varies only positive-sign `nl`, B2 only `nl-`, and B3 both. All other input
on a row is held constant.

| System/variant | SCF rounds | Minimum candidate overlap | Maximum radius factor | Final order or interval |
|---|---:|---:|---:|---|
| Ni I NV | 1 | -- | -- | `J4<J3<J2` |
| Ni I B1 | 20 | 0.02920 | 8.610 | `J2<J3<J4` |
| Ni I B2 | 12 | 0.03570 | 8.416 | `J4<J3<J2` |
| Ni I B3 | 14 | 0.02902 | 8.630 | `J4<J3<J2` |
| Ni/Ca-like NV | 1 | -- | -- | `J2<J3<J4` |
| Ni/Ca-like B1 | 6 | 0.19329 | 2.781 | `J2<J4<J3` |
| Ni/Ca-like B2 | 7 | 0.29918 | 2.784 | `J2<J3<J4` |
| Ni/Ca-like B3 | 8 | 0.18857 | 2.798 | `J2<J3<J4` |
| Cl I B0/no-varied | 1 | -- | -- | `J3/2<J1/2`, `+923.8116 cm^-1` |
| Cl I B1 | 8 | 0.616734 | -- | reversed, `-2479.5079 cm^-1` |
| Cl I B2 | 8 | 0.616734 | -- | `+3788.6034 cm^-1` |
| Cl I B3 | 12 | 0.558525 | -- | `+932.1882 cm^-1` |

The supported conclusion is that the original positive-sign-only selection
causally changes the ordering in Ni I, Ni/Ca-like, and Cl I, while B3 restores
the expected order. B2 can also be correct, so the evidence does not justify
the broader claim that every one-sided selection must fail. A one-sided Fe I
case also preserves the `^5D_J` order.

B3 does not eliminate Ni I's low overlaps, large radius changes, or node
changes. Varied-list completeness affects fine-structure balance but does not
guarantee stable candidates. These historical B3 results predate the partner
transaction and are not evidence for its atomicity; see Section 8 for that.
The authoritative compact table is
[`RESULTS.md`](../../rmcdhf_test/test/rmcdhf_orbopt/RESULTS.md).

## 3. Cl I: how differential relaxation reverses the order

In the old one-sided AS1-AS5 sequence, optimization lowers the $J=1/2$ total
energy by `0.0121--0.0155 Eh` more than $J=3/2$. At AS5, the difference is
about `0.0121238 Eh = 2660.9 cm^-1`. Applying that imbalance to the no-varied
interval of `+921.13 cm^-1` predicts about `-1739.74 cm^-1`, consistent with
the observed reversal.

The problem is therefore not whether optimization lowers total energies:
variational optimization can lower every state while degrading their
differences. See the [Cl I report](./cal-report-analysis-cl-i-lbl-orbital-optimization-2026-08-18.md).

## 4. Jobs 635/636: inherited AS baseline versus current-layer effect

Fixed-RCI comparisons using the same AS2 CSF space are:

| System/wavefunction | Order | First interval (`cm^-1`) | Second interval (`cm^-1`) |
|---|---|---:|---:|
| Ni I, `_NV AS1 + TF AS2` | `J4<J3<J2` | 1305.08 | 2168.02 |
| Ni I, optimized AS1 + TF AS2 | `J2<J3<J4` | -1411.04 | -2730.88 |
| Ni I, optimized AS1 + optimized AS2 | `J2<J3<J4` | -4262.78 | -8606.07 |
| Ni/Ca-like, `_NV AS1 + TF AS2` | `J2<J3<J4` | 1833.33 | 4031.32 |
| Ni/Ca-like, optimized AS1 + TF AS2 | `J2<J3<J4` | 893.62 | 1953.59 |
| Ni/Ca-like, optimized AS1 + optimized AS2 | `J4<J3<J2` | -102.85 | -369.73 |

Ni I is already reversed when TF AS2 is appended to the incorrectly optimized
AS1, and AS2 optimization amplifies the error. For Ni/Ca-like, optimized AS1
compresses the intervals without reversing them; full AS2 optimization causes
the reversal. Every AS must therefore be accepted independently. Checking only
the final layer, or replacing only the current layer with TF orbitals, cannot
repair contaminated history.

No nonidentity CI-root match was found among 13 targeted orbital variants. Even
when Ni/Ca-like `all_new` changes the level order, root matching remains the
identity. The evidence supports relative $J$-dependent energy shifts, not root
swapping, as the primary mechanism.

Raw results are in
[`decomposition_summary.csv`](../../data/rmcdhf_test_data/results/matched-tf-rci-636/decomposition_summary.csv).
An older safety note said Job 636 did not exist, but the retained result
directory and `status.csv` contradict that statement; this summary follows the
existing result files.

## 5. Job 645: correct and strictly converged, but not better than TF

Job 645 uses correct balanced AS1 and AS2, a reduced ASF set, fixed damping,
node-progress checks, a round guard, and strict SCF convergence. It completed
normally; orbital, weighted-energy, and identity criteria passed for two
consecutive rounds after 62 rounds. Every target root mapped to itself, the
minimum adjacent-round CI-coefficient overlap was about 0.9927, and all 62
rounds were accepted without root swap or whole-round rollback. The final order
was `^3F_4<^3F_3<^3F_2`, with cumulative intervals
`1374.81/2290.63 cm^-1`.

| Ni I target | NIST (`cm^-1`) | Job 645 | Job 645 error | fixed-TF/RCI error |
|---|---:|---:|---:|---:|
| `^3F_3` | 1332.164 | 1374.811 | +42.647 | -27.102 |
| `^3F_2` | 2216.550 | 2290.635 | +74.085 | -48.536 |

Job 645 proves that the balanced path can converge strictly and stably while
preserving the correct order. It does not prove better numerical accuracy than
TF. Under the rule that optimized NIST absolute error may not exceed the
same-layer fixed-TF/RCI error, both targets must be rejected.

Job 645 differs from the earlier-stopping Job 602 by only about
`0.03/0.21 cm^-1`. On the same correct AS1, full-ASF Job 601 differs from the
reduced-ASF result by only about `1.1/1.6 cm^-1`. Strict SCF, state-set
reduction, and the round guard are therefore not the principal explanation for
the correct order or the remaining NIST error.

See [`e1_vv2as2_rmcdhf.csv`](../../data/rmcdhf_test_data/results/ni-target-baseline-guard-645/e1_vv2as2_rmcdhf.csv)
and the TF [`pair_summary.csv`](../../data/rmcdhf_test_data/results/fixed-orbital-rci-pair-635/pair_summary.csv).

## 6. Limits of the NIST-accuracy claim across systems

| System | fixed-TF/no-varied | Balanced optimization | Conclusion |
|---|---|---|---|
| Cl I | B0 error about `+41.46 cm^-1` | B3/B4 error about `+49.83 cm^-1` | Order repaired, accuracy slightly worse |
| Ni I | Job 635 absolute errors `27.10/48.54` | Job 645 `42.65/74.09` | Correct order, not better than TF |
| Ni IX | no-varied about `+140.9/+401.7` | balanced damped about `+10/+119` | Order and accuracy both improve materially |

No single switch yet explains why a balanced result is close to NIST. A
plausible, but not isolated, explanation is that complete partner selection
reduces $J$-dependent imbalance while TF or controlled relaxation preserves
different degrees of error cancellation in truncated CSF spaces. Residual
errors may also contain finite-correlation-space, Breit/QED, and later nuclear-
polarization contributions. These must be introduced separately, not replaced
by post hoc threshold tuning.

## 7. Excluded primary explanations and their remaining uses

| Factor | Evidence | Current role |
|---|---|---|
| Stronger fixed damping | Three settings change final intervals only slightly | Stabilizes accepted orbitals; does not repair spectra |
| Raw-candidate guard | Can reject continuously without recovering a candidate | Safe stop, not an optimizer |
| Strict `METHOD=3` | Same outcome as B3 with no fallback | Fallback is not the tested primary cause |
| Equal/statistical EOL weights | Cl I differs by only `0.687 cm^-1` | Not the large reversal mechanism |
| Deferred `ORTHY` | Nonfinite values in round 2 | Invalid direction; stopped |
| Node `THRESH` tuning | Cannot uniformly distinguish tail oscillations from physical nodes | Diagnostic only |
| CI round/root guard | Job 645 has no rollback or root swap | Prevents propagation; does not create correct candidates |
| QDIF / `Gatherv` | Single-factor physical results do not improve | Retained correctness fixes, not causal explanations |
| Global sparse-column offset | Job 609 badly corrupts terms; corrected to rank-local offset | Invalid experiment, not physical evidence |

## 8. Program-level evidence for the partner transaction (2026-09-16)

`GRASP_PAIR_TRANSACTION=1` explicitly enables the transaction. With it unset,
the corrected legacy per-orbital path remains active.

| Validation | Result | Result directory |
|---|---|---|
| CTest | 4/4 passed | Build-tree tests |
| Ni I AS1, 1 rank | All 51 schedules eventually committed; one node failure rolled back the group and succeeded at 0.5 | `pair-smoke-final-ni-20260916000500` |
| Ni/Ca-like AS1, 1 rank | All 28 schedules committed | `pair-smoke-final-nica-20260916000500` |
| Cl I AS1, 1 rank | All 36 schedules committed | `pair-smoke-final-20260915235000` |
| Cl I, 1/2/4 ranks | Identical decisions; maximum final two-level energy difference at CSV precision is 0.0 | `pair-smoke-final-20260915235000`, `pair-final-rank2-flush-20260916004000`, `pair-final-rank4-flush-20260916004000` |
| Cl I, 46-rank production | Runner and solver exit 0; decisions and final energies match 1/2/4 ranks | `pair-final-prod46-cl-20260916010000` |
| Injected post-`ORTHY` fault | Rollback on the same schedule/snapshot, then commit at 0.5; `rmcdhf.exitcode=0` | `pair-final-post-orthy-20260916002000` |
| Persistent fault/limit | Rollback at 0.5 and 0.7 on one schedule/snapshot, then `MPI_Abort(92)` | `pair-final-retry-flush-20260916003000` |
| Disabled by default | `rmcdhf.exitcode=0`, no pair event, capability false | `pair-default-final-20260915231500` |
| `NSIC/MAXARR` | Test override makes the extra schedule include the selected orbital's complete group | `pair-nsic-cl-20260915232500` |

Every successful `orbopt_trace.csv` has exactly 93 columns and no overflow
fields. The natural Ni I `nodes_expected` failure and injected post-`ORTHY`
failure both show that the transaction restores the whole group and regenerates
both candidates from the original snapshot rather than accepting one side.

For some runs, `run_data_case.sh` returns 1 late in postprocessing because the
current binary uses `NNNP=2990` while the archived reference uses 590; the
comparator intentionally rejects an undeclared radial-grid comparison. Solver
success here is established by `rmcdhf.exitcode=0`, a complete trace, and the
level CSV in the result directory.

## 9. Missing physical evidence

Program-level transaction semantics are established, but no evidence yet shows
that the **directly optimized candidate at every AS** satisfies the final goal
for all three systems. The next campaign must start from correct anchors,
advance layer by layer, and check order, identity, strict convergence,
repeatability, a NIST tolerance frozen before execution, and performance no
worse than same-layer fixed-TF/RCI. Fallback output does not count as success.

### 9.1 Ni I AS2 transaction probe (Job 647)

The first post-transaction AS2 continuation was submitted through
[`run_pair_physical_as2_ni.sbatch`](../../rmcdhf_test/test/rmcdhf_orbopt/slurm/run_pair_physical_as2_ni.sbatch)
with four MPI ranks, the balanced AS1 wavefunction from the two-repeat AS1
campaign, `GRASP_PAIR_TRANSACTION=1`, strict SCF, fixed damping, and the node
progress guard enabled.  The job exited with `rmcdhf_mpi` status `92` in the
first SCF round.  The `6s` group failed the expected-node check (`expected=5`,
`counted=3`) at every attempt; transaction retries used damping `0.50`,
`0.70`, `0.90`, and `0.90`, then the pair retry limit was exceeded.  No
candidate wavefunction was accepted and no AS2 physical comparison was made.

The complete scheduler log is
[`647_rmcdhf-pair-physical-as2-ni.log`](../../data/rmcdhf_test_data/log/647_rmcdhf-pair-physical-as2-ni.log),
and the retained trace/result directory is
[`pair-physical-as2-ni-647`](../../data/rmcdhf_test_data/results/pair-physical-as2-ni-647).
This is a rejected AS2 attempt, not fallback success.  It establishes that
the current balanced AS1 anchor does not pass the enabled `nodes_progress`
gate for the AS2 `6s` candidate, so the chain cannot advance under that
production policy.

The follow-up diagnostic keeps the transaction and strict-convergence
controls but disables only `GRASP_NODE_GUARD_PROGRESS`; it is intended to
identify whether the failure is specific to that diagnostic gate.  Even if it
converges, it cannot satisfy the production node-stability requirement without
a separately justified threshold.

## 10. Minimum provenance for reproduction

Retain `isodata`, the actual `rcsf.inp`, previous-AS and TF `rwfn`, complete
stdin, state selection and weights, source commit, executable SHA-256, `NNNP`,
compiler/MPI/BLAS versions, rank/thread binding, every `GRASP_*` variable,
stdout/stderr, exit code, per-round trace, final `rwf/mix/sum`, fixed-RCI result,
NIST source, and tolerances frozen before execution. Count and hash `_raw.c` and
the `.c` actually read by the calculation separately.
