# Current Code Review of `rmcdhf90_mpi`

## 1. Scope and conclusion

This review covers the current
[`rmcdhf_test/src/appl/rmcdhf90_mpi`](../../rmcdhf_test/src/appl/rmcdhf90_mpi)
and the shared routines it actually links, with a static comparison against the
upstream copy at `/home/workstation2/AppFiles/grasp_raw/src`. The review date is
2026-09-16; partner-transaction conclusions refer to the then-current working tree.

The working tree implements an independent atomic `nl-/nl` transaction in
addition to the legacy sequential per-orbital path.
`GRASP_REQUIRE_BALANCED_PAIR` still means only input completeness.
`GRASP_PAIR_TRANSACTION=1` enables shared pre-state, common damping, joint
acceptance, and group rollback. Control output and traces keep these
capabilities distinct.

Section 9 independently rechecks two edits. `ED1=PED(J)` in `improvmpi.f90`
always reads zero on the legacy commit path because MPI `DAMPCK` never writes
`PED`; its formula also differs from serial `DAMPCK`. Removing the one-time
latch in `defcor.f90` has no observable effect in the current tree. Neither
finding changes the transaction invariants in Sections 4 and 8, but both refine
the interpretation of the corresponding rows in Section 3.1.

## 2. Current call paths

After parsing the varied list, `GETOLDmpi` calls:

```text
APPLY_FIXED_ORBITAL_DAMPING
CHECK_RELATIVISTIC_PAIRS
TRACE_ORBITAL_SELECTION
```

`CHECK_RELATIVISTIC_PAIRS` locates $\kappa=l$ and $\kappa=-(l+1)$ for a common
$(n,l)$ and compares `LFIX`. In strict mode, a missing or one-sided varied
partner stops execution. It does not itself build the group table used by SCF.

With transactions disabled, `SCFmpi` retains the legacy path:

```text
SETLAGmpi
for J in IORDER:
    if varied(J): IMPROVmpi(J)
repeat NSIC times:
    J = MAXARR()
    IMPROVmpi(J)
ORTHSC (when required)
MATRIXmpi + NEWCOmpi
```

One `IMPROVmpi(J)` performs `COFPOTmpi -> DEFCOR -> SOLVE -> CONSIS ->
normalize -> DAMPCK/DAMPOR -> ORTHY`. `DAMPOR` writes the candidate to `PF/QF`
before the next partner begins, and `ORTHY` may rebuild the complete
equal-$\kappa$ subspace. The second partner therefore does not see the original
pre-group state.

With transactions enabled, rank 0 constructs and broadcasts a deterministic
group table. The same complete snapshot is restored before preparing every
member. Only after all raw candidates exist does the code apply common damping,
provisionally write all members, and run post-damp and post-`ORTHY` checks. Any
failure restores the complete snapshot and immediately retries the same
snapshot/schedule. An orbital selected by `NSIC/MAXARR` is mapped back to its
complete group.

## 3. Effective differences from upstream

### 3.1 Unconditional numerical and parallel-correctness changes

| Location | Current change | Effect | Relation to fine-structure evidence |
|---|---|---|---|
| `setlagmpi.f90` | Restore QDIF/`OBQDIF` for unequal generalized occupations | Matches serial and memory-MPI formulas | Single-factor regression does not remove the residual error |
| `improvmpi.f90` | `ED1=PED(J)` | Intended to restore previous energy; Section 9.1 shows legacy MPI never updates `PED`, so this reads zero | Required initialization, not a unique cause; it does not have the intended legacy effect |
| `improvmpi.f90` | Use actual per-rank `NDCOF` in `MPI_Gatherv`; set `NDCOF=i_last` | Avoids uninitialized tails and truncation of the unique union | Single-factor target intervals are unchanged |
| `defcor.f90` | Remove `DATA FIRST/.TRUE./`; clear `DP(:2)/DQ(:2)` every call | Section 9.2 proves only this file writes those entries | Equivalent simplification, not a reproduced defect fix |
| `iniestmpi.f90` | Advance sparse offset over every local column | Matches rank-local packed storage | MPI correctness fix |
| `spicmvmpi.f90` | Advance `IBEG` over local packed columns | Avoids treating a global prior-column endpoint as a local offset | Restores consistent 46-rank initial CI |
| `parameter_def_M.f90` | `NNNP 590 -> 2990` | Enlarges radial-grid arrays | Build variant; not directly comparable with a 590-grid baseline |

`count.f90` and `coun_C.f90` add environment-controlled node-diagnostic
context only; they do not change the default node algorithm.

### 3.2 Optional diagnostic and safety paths

New modules include:

- `orbopt_control.f90`: switches, thresholds, targets, and rejection counters;
- `orbopt_metrics.f90`: norm, old/new overlap, mean radius, and node metrics;
- `orbopt_trace.f90`: selection, candidate, `ORTHY`, SCF, MPI, and target CSV;
- `orbopt_round_state.f90`: whole-round orbital/CI/scalar snapshots, CI
  coefficient assignment, rollback, and `.mix` rewriting.

Optional behavior includes fixed `ODAMP`, candidate rejection, post-damping
checks, node progress, strict `METHOD=3`, strict SCF, whole-round identity/order
rollback, and fixed-reference coefficient proxies. Most are disabled by
default; a nonfinite weighted energy always stops execution.

These mechanisms observe or reject bad candidates. Every Job 645 round was
accepted, so they do not explain its correct order. AS-level fallback also does
not count as successful optimization.

## 4. Current partner-transaction invariants

| Required invariant | Status |
|---|---|
| Deterministic partner groups drive SCF | Satisfied: rank 0 builds and broadcasts them in first-appearance `IORDER` order |
| Both candidates use the same `PF/QF/E` pre-state | Satisfied: the complete snapshot is restored before each member |
| Candidate generation is separate from commit | Satisfied: `IMPROVmpi(PREPARE_ONLY)` returns a raw candidate only |
| Both sides use common damping | Satisfied: use the more conservative retention; retry floors are 0.5/0.7/0.9 |
| Both sides accept or reject together | Satisfied: any raw, post-damp, or post-`ORTHY` failure rolls back the group |
| Post-`ORTHY` failure restores the group | Satisfied: complete orbital, working-array, and scalar state is restored and asserted |
| `NSIC/MAXARR` updates a group | Satisfied: selected `J` is mapped to its group |
| Trace proves atomic commit | Satisfied: fixed 93-column group/snapshot/schedule/phase/retry/decision records |

Historical B3 and Job 645 remain sequential per-orbital runs. Only a new run
with the independent transaction capability enabled is a partner-group atomic run.

## 5. Implemented code boundary

`orbopt_pair_types.f90` defines candidate records and MPI-independent grouping;
`orbopt_pair_transaction.f90` owns groups, the complete snapshot, candidate
buffers, and transaction state. The principal structures are:

```text
type orbital_group_t
    integer :: id, size, member(2), retry_count
    real(DOUBLE) :: shared_damping
end type

type orbital_candidate_t
    integer :: j, mtp, method, inv, jp, nodes
    real(DOUBLE) :: energy, scnsty, pz
    real(DOUBLE) :: norm, overlap, radius_ratio
    logical :: solve_failed, fallback_used, quality_failed
    real(DOUBLE), allocatable :: p(:), q(:)
end type
```

Rank 0 groups by `NP/NAK/LFIX/IORDER` and broadcasts the result. $s_{1/2}$ is a
singleton; for $l>0$, a missing, duplicate, or fixed partner fails before SCF.
Grouping does not depend on string labels.

`PREPARE_PAIR_CANDIDATE` restores the group snapshot, invokes prepare-only
`IMPROVmpi`, and copies the candidate. `IMPROVE_ORBITAL_GROUP` applies common
damping, provisionally writes all members, checks post-damping and post-`ORTHY`
state, and either commits once or fully restores and retries. Because `SOLVE`,
`DAMPCK/DAMPOR`, and `ORTHY` modify broad shared state, the first implementation
snapshots complete orbital and working arrays. The snapshot has not been
narrowed based on performance measurements.

The initial SCF pass walks the broadcast group table. The `NSIC` path maps the
orbital returned by `MAXARR` to a group and calls `IMPROVE_ORBITAL_GROUP`.
Common damping contributes to `DAMPMX`; retry count is group state.

## 6. Numerical and MPI decision rules

1. Restore one group snapshot before computing each candidate; `SETLAGmpi`
   remains unchanged within the group.
2. Select and trace the more conservative damping proposal.
3. Failure by any member at raw, post-damp, or post-`ORTHY` phase fails the group.
4. Rollback restores `PF/QF/E/MF/PZ/SCNSTY/ODAMP/METHOD/NSIC` and all orbitals
   that `ORTHY` may affect.
5. Reduce failures across ranks, have rank 0 broadcast the unique decision, and
   compare orbital-state digests after commit.
6. Exceeding the group retry limit uses `MPI_ABORT` or the common MPI shutdown;
   no subset of ranks may continue.

## 7. Guardrails that must remain, with correct interpretation

- Keep the fixed-TF/RCI anchor, identity/order/radial checks, `.rwf/.mix`
  rollback, and next-AS hash validation.
- CI-coefficient overlap across changing orbital bases is a same-CSF proxy, not
  a rigorous many-electron ASF overlap.
- Whole-round rollback protects one SCF round; it is not partner-group atomicity.
- Keep `GRASP_REQUIRE_BALANCED_PAIR` as the input gate and expose a separate
  `pair_transaction_enabled` capability.
- Rejected or fallback state may be used for recovery, but must be reported as
  optimization failure.

## 8. Validation status and completion criteria

The target builds and all four CTests pass. Transaction-on AS1 regressions for
Ni I, Ni/Ca-like, and Cl I exit with `rmcdhf.exitcode=0`. Cl I has identical
decision sequences and CSV-precision final energies at 1/2/4/46 ranks. Trace
evidence covers post-`ORTHY` fault recovery, the 0.5/0.7 retry limit,
`MPI_Abort(92)`, disabled-by-default behavior, and `NSIC/MAXARR` group mapping.
See Section 8 of the [test results](./rmcdhf90_mpi_orbital_optimization_test_results_en.md).

All Section 4 invariants have source and trace evidence. Existing implicit-MPI
cross-call type warnings remain. Physical completion is defined separately in
the [implementation plan](./rmcdhf90_mpi_orbital_optimization_debugging_plan_en.md):
each directly optimized AS must pass NIST and no-worse-than-fixed-TF acceptance.
Fallback does not count.

## 9. Independent static review (2026-09-16)

This review compared each file against `grasp_2990_NNNP`, the unmodified
GRASP2018 copy with only the `NNNP` patch, and manually traced calls. It did not
rebuild or rerun the MPI physical regressions. It did rebuild and pass the
MPI-independent `rmcdhf_pair_transaction_logic` test, independently confirming
the observed behavior of `BUILD_ORBITAL_GROUP_TABLE`,
`SUGGEST_ORBITAL_DAMPING`, and `RETRY_PAIR_DAMPING`.

The following claims were confirmed:

- grouping validates the `IORDER` permutation and unique $(n,\kappa)$ values,
  accepts $s_{1/2}$ singletons, and rejects missing/fixed partners;
- `SHARED_DAMPING=MAX(...)` uses the larger, more conservative proposal;
- transaction mode preserves Gauss-Seidel ordering **between groups**; only
  members **within a group** share a pre-state;
- `TRACE_FIELD_COUNT=93`; field 88 contains `MAX_PAIR_RETRIES` only on the
  one-time `event='control'` row and the actual retry count on pair rows;
- build-file dependency order is correct; changes in `solve.f90`, `orthy.f90`,
  and `newcompi.f90` are trace-only;
- `GETOLDmpi` call order matches Section 2; `DEFER_ORTHOGONALIZATION` and
  `STRICT_METHOD3` are off by default and are documented as excluded experiments;
- changes are confined to `rmcdhf90_mpi`, `lib9290/count.f90`,
  `libmod/coun_C.f90`, and `mpi90/{iniestmpi,spicmvmpi}.f90`. Serial,
  memory-only, and memory-MPI RMCDHF variants are untouched.

### 9.1 `PED` damping history

`improvmpi.f90` adds `ED1=PED(J)` to mirror the serial line, but serial and MPI
`DAMPCK` implement different algorithms:

| | Serial `dampck.f90` | MPI `dampck.f90` |
|---|---|---|
| Energy difference | Relative: `ED2=(ED2-E(J))/ED2` | Absolute, only when `IPR==J`: `ED2=ED2-E(J)` |
| Test | Always `ED1*ED2 < -1.0D-4` | Only for `IPR==J`: `ED1*ED2 > 0.0D0` |
| Coefficients | Reversal: `0.10+0.90*ODAMP`; otherwise `0.50*ODAMP` | Reversal: `0.25+0.75*ODAMP`; otherwise `0.75*ODAMP` |
| `IPR` | Unused compatibility parameter | Gates the same-orbital branch |
| `PED(J)` | Written unconditionally at the end | Never read or written in either MPI variant |

Both files match upstream byte-for-byte; this difference predates the current
work. `getscdmpi.f90` initializes `PED(:NW)=0`, and the legacy path never
updates it. Consequently `ED1=PED(J)` always assigns zero. Before this edit,
`ED1` was a saved local scalar carrying state across `DAMPCK` calls. The only
observable case is consecutive updates of the same orbital (`IPR==J`), such as
repeated `NSIC/MAXARR` selection or a single varied orbital. Previously that
case compared the real saved trend; now `0*ED2` always fails `>0`, selecting
`ODAMP=0.25+0.75*ODAMP` unconditionally.

The line therefore does not restore serial-equivalent energy history. It should
either be removed or documented accurately as a zero-valued placeholder.

The pair transaction is different. `SUGGEST_ORBITAL_DAMPING` reproduces MPI
coefficients and criteria, while `PED(J)=CANDIDATE%PED_PROPOSED` persists
history per orbital without the `IPR` gate. Transaction-on and legacy damping
proposals are therefore methodologically different, not merely atomic and
non-atomic wrappers around the same algorithm. Transaction-on/off trajectory
comparisons must record that distinction.

### 9.2 The one-time latch in `defcor.f90`

The edit replaces a one-time `DATA FIRST/.TRUE./` initialization of
`DP(:2)=DQ(:2)=0` with unconditional clearing on every call. A complete search
shows that only `defcor.f90` writes these entries; `setxuv.f90` only reads them.
The upstream one-time version is therefore already safe during a current run,
and this edit is an equivalent simplification rather than a reproduced bug fix.

Keeping the simplification is still sensible. The transaction snapshots
`DP/DQ` for every candidate; a process-lifetime hidden `SAVE` flag is fragile
next to a fully rollbackable transaction if another writer is added later.

### 9.3 Static-review limit

This section did not execute `rmcdhf_mpi` or audit every hidden `SAVE` local or
every module field outside the explicit snapshot list. Section 10 addresses the
build/test reproduction and the principal snapshot write surface.

## 10. Independent dynamic verification supplement (2026-09-16)

### 10.1 Build and CTest reproduction

After rerunning `cmake .`, all four CTest targets registered. Builds of
`test.rmcdhf_pair_transaction_logic`, `test.mpi90_local_sparse`, and
`rmcdhf_mpi` succeeded; `ctest` reported `100% tests passed, 0 tests failed out
of 4`. The final binary linked with warnings only. Warnings at
`orbopt_pair_transaction.f90:667,671` are the known implicit MPI cross-call type
warnings: `mpi_C` lacks type-specific interfaces for these generic calls, and
existing files already emit the same class of warning.

### 10.2 Remaining unconditional changes

The QDIF/`OBQDIF` branch in `setlagmpi.f90` matches both serial
`setlag.f90:226-238` and memory-MPI `setlagmpi.f90:263-274`; unlike `PED`, all
three variants share this multiplier formula.

The old fixed-size `MPI_GATHER` sent `ndcof_max` elements from every rank,
reading uninitialized tails where local `ndcof` was smaller. It then truncated
the merged result to `ndcof_max`, potentially dropping valid unique
coefficients when rank-local label sets were largely disjoint. `MPI_Gatherv`
now sends actual local counts, uses prefix displacements, and sets
`NDCOF=i_last`, the true size of the unique union. This is a genuine correctness
fix independent of whether one regression changes a target interval.

### 10.3 Snapshot completeness

The dynamic supplement read all routines directly reached by candidate
preparation, shared damping, and `ORTHY`, and compared module writes with
`SAVE_PAIR_SNAPSHOT`/`RESTORE_PAIR_SNAPSHOT`:

| Routine | Module state written | Covered |
|---|---|---|
| `COFPOTmpi` | `YP,XP,XQ` | Yes |
| `DEFCOR` | `DP,DQ` | Yes |
| `CONSIS` | `SCNSTY(J)` | Yes |
| `SOLVE` and potential helpers | `E,EPSMIN,EPSMAX,EMAX,P0,P,Q,MTP0,MTP,TA,TF,TG,XU,XV` | Yes |
| `DAMPOR` | `PZ,MTP0,MF,PF,QF,P,Q` | Yes |
| `RINT` | `MTP,TA` | Yes |
| `QUAD` | `TA`, including its three-element tail | Yes |
| `ORTHY` | `PZ,PF,QF,MF` for every nonfixed equal-`NAK` orbital | Yes, snapshots span all orbitals |

No directly visible module write lacked snapshot coverage. The conclusion
covers visible `SOLVE` writes for `METHOD<=2`; it does not independently
rederive internal write surfaces of unchanged helpers such as
`IN/OUT/START/SETXUV/SETXZ/SETXV/ESTIM/NEWE`.

### 10.4 Change surface and integration

Directory comparison confirms that `rmcdhf90`, `rmcdhf90_mem`,
`rmcdhf90_mem_mpi`, and `libdvd90` are unchanged. Transaction mode maps both
the main sweep and `NSIC/MAXARR` updates through `GROUP_FOR_ORBITAL`; mapping
failure is `ERROR STOP`, never silent single-orbital fallback.
`INITIALIZE_PAIR_TRANSACTION` runs once before the `NIT` loop, consistent with
`LFIX/IORDER/NP/NAK` remaining fixed during one SCF call. Finally,
`PAIR_TRANSACTION_ENABLED` forces `REQUIRE_BALANCED_PAIR`, and rank 0 reads and
broadcasts every added control, including `MAX_PAIR_RETRIES` and fault-injection
flags. No missing broadcast field was found.
