# rmcdhf90_mpi partner-group atomic transactions: implementation record for this iteration

Date: 2026-09-16

This document provides a persistent record of the implementation and validation results for Sections 3--4 of the
[`rmcdhf90_mpi` orbital-optimization debugging plan](./rmcdhf90_mpi_orbital_optimization_debugging_plan_en.md)
from this iteration. It records the current uncommitted working tree and local regression evidence; it does not mean that the "per-AS physical acceptance" in Section 5 of the plan has been completed.

## 1. Conclusions from this iteration

The partner-group atomic-update transaction for `rmcdhf_mpi` has been implemented and is enabled explicitly with the environment variable `GRASP_PAIR_TRANSACTION=1`; when it is disabled by default, the legacy per-orbital path is retained. When enabled:

1. rank 0 constructs `(n,l)` groups in order of first appearance in `IORDER`, with negative kappa followed by positive kappa within each group;
2. `s1/2` may form a single-member group, while a missing, duplicated, or fixed partner for `l>0` is rejected before the SCF calculation;
3. the raw candidate for every partner is generated from the same complete pre-group snapshot; the preceding candidate never becomes a potential input for the next candidate;
4. all partners use the same conservative damping, are written back together provisionally, and then undergo numerical and post-`ORTHY` checks together;
5. if any member, or any orbital affected by `ORTHY`, fails, `PF/QF` and the associated scalars and work arrays are restored in full;
6. a rollback is retried immediately against the same schedule/snapshot, with successive damping floors of `0.5 -> 0.7 -> 0.9`;
7. exceeding the limit terminates all processes consistently through `MPI_Abort`;
8. when `MAXARR/NSIC` selects a single orbital, it is mapped back to the entire group rather than falling back to a single-orbital commit;
9. every raw, post-damp, post-`ORTHY`, and commit/rollback event is written to a fixed 93-column CSV; the file is flushed before abnormal termination, allowing the decision chain to be replayed;
10. after every commit/rollback, an orbital-state digest is reduced; any disagreement between ranks causes an immediate `MPI_Abort`.

## 2. Principal code changes

- `rmcdhf_test/src/appl/rmcdhf90_mpi/orbopt_pair_types.f90`
  - candidate data structures;
  - group-table construction and input rejection;
  - a hidden-state-free damping recommendation equivalent to MPI `DAMPCK`;
  - retry damping values of 0.5/0.7/0.9.
- `rmcdhf_test/src/appl/rmcdhf90_mpi/orbopt_pair_transaction.f90`
  - complete snapshots, restoration, and element-by-element restoration assertions;
  - candidate generation from a shared pre-state;
  - shared damping, shared checks, `ORTHY`, and atomic commit;
  - immediate rollback/retry from the same snapshot;
  - MPI consensus and cross-rank state-digest checks.
- `rmcdhf_test/src/appl/rmcdhf90_mpi/improvmpi.f90`
  - added a prepare-only phase;
  - returns `SOLVE`'s `INV/JP/NNP` and fallback/failure state to the transaction;
  - the prepare phase returns before damping, `ORTHY`, or final writeback.
- `rmcdhf_test/src/appl/rmcdhf90_mpi/scfmpi.f90`
  - schedules the initial pass by group;
  - schedules supplementary `NSIC/MAXARR` updates by group.
- `rmcdhf_test/src/appl/rmcdhf90_mpi/orbopt_trace.f90`
  - expanded the trace from 81 to 93 columns;
  - added group, snapshot, schedule, shared-damping, retry, phase, and decision fields;
  - flushes every row so that rollback records reach disk before `MPI_Abort`.
- `rmcdhf_test/test/rmcdhf_orbopt/pair_transaction_logic.f90`
  - covers group construction, invalid tables, fixed/missing partners, damping, and the retry ladder.

Two determinism fixes were also made:

- initialize `PED`, preventing candidate damping history from coming from an uninitialized value;
- explicitly zero the first two points in `DEFCOR` on every invocation, eliminating candidate-generation order dependence.

## 3. Build and unit validation

The final binary was built with the OpenMPI module:

    source /usr/share/Modules/init/zsh
    module load mpi/openmpi-x86_64
    cmake --build rmcdhf_test/build --target rmcdhf_mpi \
      test.rmcdhf_pair_transaction_logic -j4
    ctest --test-dir rmcdhf_test/build --output-on-failure

Result: 4/4 passed:

- `lib9290_quad`
- `mpi90_local_sparse`
- `rmcdhf_pair_transaction_logic`
- `rmcdhf_orbopt_round_state_logic`

`git diff --check` passed. The build still emits pre-existing cross-call argument-type warnings from the old implicit MPI bindings; this iteration neither treats those warnings as a new regression nor claims to have eliminated them.

## 4. System regressions with the final binary

All directories below are under `data/rmcdhf_test_data/results/`. Every `rmcdhf.exitcode` is 0:

| System | Results directory | Group schedules | Rollbacks | Result |
|---|---|---:|---:|---|
| Ni I AS1 | `pair-smoke-final-ni-20260916000500` | 51 | 1 | Every schedule ultimately committed; after one `nodes_expected` failure, a retry from the same snapshot succeeded |
| Ni/Ca-like AS1 | `pair-smoke-final-nica-20260916000500` | 28 | 0 | All committed |
| Cl I AS1 | `pair-smoke-final-20260915235000` | 36 | 0 | All committed |

Every CSV has exactly 93 columns and no overflowing fields. The natural Ni I failure provides particularly useful validation of group-level semantics in a normal run: group 3 was rolled back in full because of `nodes_expected`, then retried against the same snapshot with shared damping of 0.5 and committed. Neither partner was accepted alone.

Note: the `run_data_case.sh` process still ultimately returns 1 because post-processing explicitly refuses to compare the current `NNNP=2990` wavefunction directly with the archived `NNNP=590` wavefunction:

    error: radial grid differs: 2990 vs 590;
    use --allow-radial-grid-difference only for a documented NNNP change

This occurs after `rmcdhf_mpi` exits successfully and after the CSV/level files have been generated. Run success is therefore determined by `rmcdhf.exitcode=0` in each results directory; this known post-processing gate is not misreported as a solver failure.

## 5. MPI rank consistency

Cl I results with the final binary:

- 1 rank: `pair-smoke-final-20260915235000`
- 2 ranks: `pair-final-rank2-flush-20260916004000`
- 4 ranks: `pair-final-rank4-flush-20260916004000`

All three runs contain the same 36 `(iteration, group_id, schedule_id, decision, retry_count)` decisions. The maximum difference between the final two level energies is 0.0 at the precision of the generated CSV. No cross-rank state-digest mismatch was triggered.

The production rank count has not yet been run with this iteration's final binary, so only 1/2/4-rank validation is claimed here.

## 6. Rollback and retry fault injection

### 6.1 Post-`ORTHY` rollback

Directory: `pair-final-post-orthy-20260916002000`

- group 2, schedule 2, snapshot 2 first rolled back because of `non_finite`;
- it retried and committed against the same schedule/snapshot with shared damping of 0.5;
- `rmcdhf.exitcode=0`;
- no `rollback did not restore snapshot` message occurred.

This covers full restoration of both partners and of orbitals modified indirectly by `ORTHY`.

### 6.2 Retry limit

Directory: `pair-final-retry-flush-20260916003000`

- two consecutive rollbacks were recorded for the same group 2, schedule 2, and snapshot 2;
- the shared damping values were 0.5 and 0.7 in sequence;
- the second rollback caused the retry count to exceed the configured limit of 1;
- all processes stopped consistently through `MPI_Abort(92)`, with `rmcdhf.exitcode=92`;
- both rollback rows were flushed to the CSV before the abort.

## 7. Default-off behavior and `NSIC` evidence

- Default-off regression: `pair-default-final-20260915231500`. `rmcdhf.exitcode=0`, there are no pair events, and `pair_transaction_enabled` is not true, demonstrating that the legacy schedule remains isolated behind an explicit switch.
- To cover the normally-zero Cl I `NSIC` branch, a test-only override was used to produce `pair-nsic-cl-20260915232500`: each iteration increased from the normal three group decisions to four, and the extra schedule's raw trace contains every member of the group. This proves that the single orbital selected by `MAXARR` is mapped to the complete group. The temporary override was not retained in the final source.

## 8. Work not completed in this iteration

This iteration completed the core implementation in Section 3 of the plan and the principal program-level regressions in Section 4. It must not be used to declare the overall orbital-optimization objective complete. The remaining work is to:

1. validate the production rank count with the final binary;
2. advance Ni I, Ni/Ca-like, and Cl I from the correct AS1 anchor, one AS at a time, through each system's target AS;
3. freeze a fixed-TF/RCI baseline with the same CSFs and EOL weights at every layer;
4. at every layer, verify target-state identity, energy ordering, strict convergence, reproducibility, the frozen NIST tolerance, and the requirement that results be no worse than TF;
5. stop an optimization chain when any layer fails; a fallback must not be counted as direct optimization success;
6. consider the nuclear-polarization potential in Section 6 of the plan only after completing these physical acceptance tests.

## 9. Working-tree state

The source changes and this report have not yet been committed. Modified and newly added files remain in `rmcdhf_test`; this report is a new file under `doc/rmcdhf_test` and is currently ignored by the parent repository's `.gitignore`, so it serves only as a persistent local record and does not by itself make the parent workspace dirty. No commit has been created, and no submodule gitlink has been updated.
