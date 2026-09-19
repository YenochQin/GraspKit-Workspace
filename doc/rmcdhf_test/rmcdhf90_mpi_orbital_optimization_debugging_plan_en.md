# `rmcdhf90_mpi` LBL Orbital-Optimization Implementation Plan

## 1. Final objective and current status

The objective is to change the `rmcdhf_mpi` numerical-update path so that, for
each new active space (AS) initialized with Thomas-Fermi (TF) orbitals, the
balanced relativistic partners `nl-` and `nl` form one atomic update group for
candidate generation, common damping, and common acceptance. A result accepted
directly by the optimizer must:

1. preserve the configuration, term, parity, and $J$ identity of each target;
2. produce the expected level order;
3. agree with NIST within tolerances frozen before execution and perform no
   worse than fixed-TF/RCI in the same AS and CSF space;
4. converge strictly, be reproducible, and be safe to inherit into the next AS.

Falling back to TF, the previous AS, or withholding a release is necessary for
production safety, but must be recorded as optimization failure for that AS. A
correct fallback is not a successful optimization. NIST is an acceptance
reference only: it must not enter the Hamiltonian, force an order, or reorder
eigenroots.

As of 2026-09-16, input-completeness checks, candidate validation, strict
convergence, whole-round rollback, and AS publication transactions exist. The
partner-group atomic transaction was implemented in `rmcdhf_test` commit
`eefa521` and passed 1/2/4/46-rank tests, fault recovery, retry-limit tests, and
AS1 program-level regression for all three systems. `GRASP_REQUIRE_BALANCED_PAIR`
still checks only varied-list completeness; shared pre-state, common damping,
and shared accept/reject behavior are enabled independently by
`GRASP_PAIR_TRANSACTION=1`. **The final numerical objective is not complete**
because the per-AS physical acceptance in Section 5 remains outstanding.

### 2026-09-16 orchestration fix

The production entry point in Section 5,
`graspkit-tools/scripts/grasp_regular_cal/run_orbopt_stage.py`, is invoked once
per AS by the generated `mcdhfmpi.sh`. It predates the partner transaction.
Before this fix, `rmcdhf_environment()` did not set `GRASP_PAIR_TRANSACTION`
and `REQUIRED_RMCDHF_CAPABILITIES` did not require it. Production submissions
therefore silently used the old sequential per-orbital path even after the
Fortran transaction passed regression.

The runner now sets `GRASP_PAIR_TRANSACTION="1"` and
`GRASP_MAX_PAIR_RETRIES="3"`, and requires the
`GRASP_PAIR_TRANSACTION` capability. A binary that does not declare it is
rejected before launch instead of silently falling back. Assertions were added
to `graspkit-tools/tests/test_run_orbopt_stage.py`. Offline verification ran:

```text
graspkit-tools/.venv/bin/python -m pytest \
  tests/test_run_orbopt_stage.py \
  tests/test_orbopt_stage_guard.py \
  tests/test_grasp_regular_orbopt_generator.py
```

All 15 tests passed; no cluster job was submitted.

The `test/rmcdhf_orbopt/run_data_case.sh` path with
`GRASP_STAGE_TRANSACTION=1` is explicitly an archived-data regression adapter
in `STAGE_GUARD_IMPLEMENTATION.md`, not the production entry point for a new AS.
Its caller exports each `GRASP_*` variable explicitly. Equivalent runs must add
`GRASP_PAIR_TRANSACTION=1` to their `env VAR=...` list, as in the Section 8
pair-smoke runs of the [test results](./rmcdhf90_mpi_orbital_optimization_test_results_en.md).
No committed `.sbatch` file was changed for that adapter.

## 2. Why the current path restores the correct order

### 2.1 Strongest controlled evidence

The strongest supported conclusion is: **a correct AS baseline not damaged by
one-sided optimization, combined with a complete `nl-/nl` varied list, avoids
the $J$-dependent differential-energy imbalance caused by positive-sign-only
`nl` relaxation.**

The B1/B2/B3 fixtures differ only in the varied list:

| System | B1: `nl` only | B2: `nl-` only | B3: both | Conclusion |
|---|---|---|---|---|
| Ni I | `J2<J3<J4`, reversed | `J4<J3<J2` | `J4<J3<J2` | Positive-only variation is the directional trigger |
| Ni/Ca-like | `J2<J4<J3` | `J2<J3<J4` | `J2<J3<J4` | Complete selection restores the expected order |
| Cl I | `-2479.51 cm^-1`, reversed | `+3788.60 cm^-1` | `+932.19 cm^-1` | Varying both restores the sign and a reasonable scale |

B2 can also preserve the expected order. The evidence therefore does not say
that every one-sided choice must fail; it says that the original positive-only
path creates unbalanced relaxation in these three fixtures, while balanced
selection is the robust input constraint.

The inherited AS baseline matters as much as the current layer:

| Ni I wavefunction source | Target order | Intervals above the lowest level (`cm^-1`) |
|---|---|---:|
| `_NV AS1 + TF AS2` | `J4<J3<J2` | `1305.08 / 2168.02` |
| Incorrectly optimized AS1 + TF AS2 | `J2<J3<J4` | `-1411.04 / -2730.88` |
| Same incorrect AS1 + optimized AS2 | `J2<J3<J4` | `-4262.78 / -8606.07` |

Regenerating TF orbitals only for the current AS cannot repair a contaminated
previous AS. The chain must start from a correct anchor and remain balanced at
every layer.

### 2.2 Physical interpretation and evidential limit

EOL minimizes the weighted total energy of selected states, not their
fine-structure intervals. In a truncated CSF space, orbital rotations cannot be
compensated as they would be by full CI. Relaxing only one relativistic partner
can therefore give different correlation-energy reductions to different $J$
blocks. In Cl I, the one-sided optimization lowers $J=1/2$ by about
`0.0121--0.0155 Eh` more than $J=3/2`; the AS5 difference is about
`2661 cm^-1`, enough to change a positive interval into a negative one.

TF orbitals act as a common-central-field constraint in the finite model and
may preserve favorable error cancellation by preventing excessive asymmetric
state-averaged relaxation. This explanation is consistent with the data, but
is not an isolated theorem. Further comparisons must hold the CSF space, EOL
weights, and AS fixed.

### 2.3 Guardrails do not explain NIST agreement

Ni I Job 645 gives `J4<J3<J2` and intervals
`1374.81/2290.63 cm^-1`, compared with NIST
`1332.164/2216.550 cm^-1`. Its errors of
`+42.647/+74.085 cm^-1` are small relative to the catastrophic one-sided
errors, but worse than the fixed-TF/RCI anchor's absolute errors of
`27.102/48.536 cm^-1`. The optimized result must therefore be rejected under
the current no-worse-than-TF criterion.

Balanced selection is proven to restore ordering and avoid catastrophic error,
not to improve NIST accuracy universally: Ni IX improves substantially, while
Ni I and Cl I remain slightly worse than fixed TF/no-varied. Nor do the current
data identify the following as primary causes of the correct order:

- all 62 Job 645 rounds were accepted, so the round guard did not alter its path;
- strict versus earlier SCF stopping changes only about `0.03/0.21 cm^-1`;
- full versus reduced ASF changes only about `1--2 cm^-1`;
- three `ODAMP` settings improve smoothness but change final intervals little;
- no effective run shows a CI root swap;
- `ED1`, QDIF, `MPI_Gatherv`, and sparse-column offsets are correctness issues,
  but their single-factor regressions do not explain the residual NIST error.

## 3. Implemented core: atomic partner updates

"Joint solution" does not combine two different-$\kappa$ Dirac equations into
one equation. It means that both candidates are derived from one accepted
pre-state and then committed as an indivisible transaction.

### 3.1 Deterministic grouping

1. Rank 0 constructs and broadcasts groups by $(n,l)`. Group order is the first
   occurrence in the original `IORDER`; order within a group is fixed.
2. An $s_{1/2}$ orbital is a valid singleton. For $l>0$, a group must contain
   both $\kappa=l$ and $\kappa=-(l+1)$; production rejects missing partners.
3. Input-completeness and numerical-transaction behavior have separate
   capability and trace flags. `balanced-pair` must not imply joint updates.

### 3.2 Separate candidate generation from commit

1. Snapshot all recoverable state before a group starts. Because `ORTHY` can
   modify other equal-$\kappa$ orbitals, save complete `PF/QF` plus relevant
   `E/MF/PZ/SCNSTY/ODAMP/METHOD/NSIC`, not just the two partners.
2. For each partner, run `COFPOTmpi -> DEFCOR -> SOLVE -> CONSIS -> normalize`
   into an independent buffer containing `P/Q/MTP0/E`, metrics, and failures.
   Restore the group snapshot before generating the next partner. The first
   candidate must never enter the second candidate's potential.
3. Both use the accepted state represented by the current `SETLAGmpi`. Any
   `METHOD` fallback is a group failure or an explicit shared policy, never a
   unilateral change.
4. Derive one conservative damping coefficient, for example the larger
   old-orbital retention proposed by either side, and apply it to both.
5. Check finiteness, norm, node progress, radius ratio, and overlap against each
   old orbital after damping. Failure on either side rejects both.
6. Only after these checks, provisionally write both candidates and execute
   affected `ORTHY` operations in deterministic order. Any post-`ORTHY` failure
   restores the complete snapshot.
7. Commit atomically only after all checks pass. On failure, increment a group
   retry counter and raise common damping; stop explicitly at a fixed limit.
8. Both the main `SCFmpi` sweep and `MAXARR/NSIC` supplementary updates schedule
   groups, never reverting to a single-orbital update.

### 3.3 MPI agreement and auditability

- Reduce candidate-failure flags consistently across all ranks; rank 0 emits
  the single group decision and broadcasts it.
- Trace the group ID, both labels, shared pre-state ID/hash, both raw,
  post-damp, and post-`ORTHY` metrics, common damping, fallback, retry number,
  and final commit/rollback decision.
- After commit, all ranks must have the same orbital-state digest. Any mismatch
  terminates immediately through `MPI_ABORT`.

## 4. Program-level validation

As of 2026-09-16, the following pass: deterministic grouping and invalid-input
unit tests; shared-snapshot candidates; group rollback for raw, post-damp, and
post-`ORTHY` failures; 0.5/0.7 retries and limit termination; group scheduling
for `NSIC/MAXARR`; disabled-by-default behavior; Cl I agreement at 1/2/4/46
ranks; and transaction-on AS1 regression for Ni I, Ni/Ca-like, and Cl I. See
Section 8 of the [test results](./rmcdhf90_mpi_orbital_optimization_test_results_en.md).

The minimum suite must continue to cover:

1. group construction, $s$ singletons, and rejection of duplicate, unknown, or
   missing partners;
2. byte-for-byte restoration of both partners and `ORTHY`-affected orbitals
   when the first candidate passes and the second fails;
3. whole-group restoration after a post-`ORTHY` failure;
4. common damping escalation `0.5 -> 0.7 -> 0.9`, or a freeze policy, with no
   possibility of unilateral acceptance;
5. group semantics for `METHOD` fallback, nonfinite candidates, and retry limit;
6. group execution of supplementary `NSIC` updates;
7. equivalent decisions and acceptable numerical agreement at 1/2/4 and
   production rank counts;
8. agreement with the corrected legacy path when the new switch is disabled;
9. replay of every group decision from the trace alone.

## 5. Per-AS physical acceptance for LBL

For every AS, hold the CSF space, state set, and EOL weights fixed:

1. Generate the new TF layer from the previous accepted AS. Run fixed-orbital
   CI/RCI first, match NIST by configuration, term, parity, and $J$, and freeze
   the baseline, target order, and absolute tolerances.
2. Run the pair transaction. Only a directly optimized candidate can pass;
   TF/previous-AS fallback is production recovery only.
3. Require orbital, weighted-energy, and target-identity criteria for two
   consecutive rounds, with no unexplained fallback or nonfinite values and
   acceptable leading target configurations and weights.
4. Every target interval must have the correct sign/order, meet the predeclared
   NIST absolute tolerance, and have absolute error no worse than same-layer
   fixed-TF/RCI. Do not relax thresholds after inspecting results.
5. Advance Ni I, Ni/Ca-like, and Cl I layer by layer from correct AS1 anchors.
   Repeat each at least twice and retain inputs, executable SHA-256, source
   commit, compiler/MPI/BLAS, rank/thread binding, and all `GRASP_*` variables.
6. Stop the optimization chain when a layer fails. Never propagate a rejected
   or fallback wavefunction as a successful result.

## 6. Nuclear-polarization potential: next physical task

Nuclear polarization is a later task, not the established explanation for the
current one-sided partner reversal. Implement it as an independent Hamiltonian
factor only after the no-NP pair-transaction baseline is stable:

- **NP0:** Standard `rmcdhf + rci`; freeze the no-NP baseline.
- **NP1:** Evaluate $\langle\Psi|V_{NP}|\Psi\rangle$ on fixed ASFs to validate
  sign and magnitude.
- **NP2:** Add E1/E2 one-electron radial matrix elements in fixed-orbital RCI,
  allowing CI remixing without orbital relaxation.
- **NP3:** Add $V_{NP}$ self-consistently to the `rmcdhf` radial potential and
  retain the same potential in final `rci`.

Test E1 and E2 separately. Validate radial integrals first on hydrogenic
`1s/2s/2p_{1/2}` before neutral many-electron systems. Record direct shifts,
CI remixing, orbital relaxation

$$\delta E_{relax}=\Delta E_{NP3}-\Delta E_{NP2},$$

nuclear parameters, unit conversions, the near-nuclear grid, and active-space
convergence. See the [nuclear-polarization note](./nuclear-polarization-potential-neutral-atom-grasp.md).

## 7. Authoritative material and definition of done

- [Test results](./rmcdhf90_mpi_orbital_optimization_test_results_en.md) contain
  reproducible evidence and negative results.
- [Code review](./rmcdhf90_mpi_orbital_optimization_code_review_en.md) records
  current implementation differences and remaining gaps.
- [Serial workflow](./rmcdhf90_en.md) explains the original per-orbital
  algorithm; it is not MPI acceptance evidence.
- The [Cl I report](./cal-report-analysis-cl-i-lbl-orbital-optimization-2026-08-18.md)
  and [Ni/Ca-like report](./cal-report-analysis-cal-report-analysis-ni-ca-like-2026-08-19.md)
  preserve the original observations but do not replace controlled causality tests.

The objective is complete only when the partner transaction is implemented,
program-level regression passes, and the **directly optimized candidate at
every AS** passes Section 5 for all three systems. A working production fallback,
one correct ordering, or an inability to find another defect is insufficient.
