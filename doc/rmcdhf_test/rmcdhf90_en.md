---
artifact_type: code_analysis
title: "GRASP2018 rmcdhf90 - Multiconfiguration Dirac-Hartree-Fock Self-Consistent-Field Pipeline"
target: rmcdhf_test/src/appl/rmcdhf90
date: 2026-08-16
analysis_objective: Reconstruct the end-to-end MCDHF self-consistent-field path, from rcsf.inp, rwfn.inp, and mcp.* to rwfn.out and, in EOL mode, rmix.out
target_language: fortran90
related_papers: []
lang: en
---

# Code-Path Analysis: GRASP2018 `rmcdhf90`

> **Scope:** This document describes the algorithm in the original serial
> `rmcdhf90`: `GETOLD -> SETLAG -> IMPROV -> DAMPOR/ORTHY -> MATRIX/NEWCO`.
> It is not evidence of the current state or physical validity of
> `rmcdhf90_mpi`. Where the implementations differ, consult the current MPI
> source, the [code review](./rmcdhf90_mpi_orbital_optimization_code_review_en.md),
> and the controlled tests.

## Scope and objective

- **Target:** `rmcdhf_test/src/appl/rmcdhf90` (113 files, including 110
  `.f90` files: 54 interface/implementation pairs, `mpi_s.f90`, and the
  `rscfvu.f90` main program). Its revision history spans 1992-2017 and combines
  a modular port of GRASP92 RSCF92 with Gaigalas's 2017 revision.
- **Objective:** Reconstruct the main MCDHF self-consistent-field path: load
  the CSF list, angular-coefficient files, and initial orbitals; perform an
  initial diagonalization in EOL mode; then alternate orbital improvement,
  orthogonalization, orbital output, and configuration diagonalization. The
  calculation stops when either the orbital or weighted-energy criterion is
  satisfied; it does not require both.
- **Boundary:** The only entry point is `PROGRAM RSCFVU`. This directory is the
  serial implementation (`MYID=0, NPROCS=1` is hard-coded in
  `rscfvu.f90:85-86`); the MPI variant is in `rmcdhf90_mpi`. The executable is
  `${GRASP}/bin/rmcdhf` and links `libdvd90 -l9290 -lmod` (`Makefile:1-2`).
  `libdvd90` is used through `GDVD` only for blocks larger than 4,000 after the
  first diagonalization; smaller blocks use LAPACK `DSPEVX` directly.
- The SCF loop implements Algorithm 5.1 of C. Froese Fischer, Comput. Phys.
  Rep. **3** (1986) 290 (`scf.f90:5-7`). The orbital solver implements
  Algorithms 5.2 and 5.3 on page 295 (`solve.f90:5-6`). This document is based
  on static analysis; the program was not compiled or executed for this review.

## Entry point and end-to-end flow

```text
isodata, rcsf.inp, rwfn.inp, mcp.30..32+KMAXF
  |-- SETMCP  <- global header written by rangular
  |              (NCORE, NBLOCK, KMAXF, NCFBLK, IDBLK, DIAG, ICCUT, LFORDR)
  |-- SETCSL  <- packed CSF occupations IQA and block symmetries JPGG
  `-- GETSCD  <- isotope data, grid, ACCY; SETRWFA loads initial orbitals;
       EOL? -> GETOLD (levels, weights, varied orbitals) or GETALD
  Initial EOL state -> MATRIX + NEWCO
  Main SCF loop (Fischer Algorithm 5.1):
     SETLAG -> {IMPROV(each varied orbital)
                + [MAXARR -> IMPROV(least-consistent orbital)] x NSIC}
     -> convergence test -> ORTHSC -> ORBOUT(rwfn.out) -> [MATRIX + NEWCO]
  Output: rwfn.out, rmcdhf.sum, rmcdhf.log; EOL also writes rmix.out
```

MCDHF varies a weighted mean energy subject to orbital orthonormality. For
level weights $w_\alpha$, where $\sum_\alpha w_\alpha=1$,

$$
E[\{P_a,Q_a\},\{c\}] = \sum_\alpha w_\alpha E_\alpha,
\qquad E_\alpha=E_{\mathrm{av}}^{(b(\alpha))}+\lambda_\alpha.
$$

Variation of $P_a,Q_a$ gives coupled radial Dirac equations with Lagrange
multipliers; variation of $c_{r\alpha}$ gives the $H(DC)$ eigenproblem. The EOL
path first obtains a CI state and then alternates orbital and CI updates:

$$
(\{P,Q\}^{(k+1)},c^{(k+1)})=G(\{P,Q\}^{(k)},c^{(k)}),
\qquad G=\text{CI diagonalization}\circ\text{orbital improvement}.
$$

## Processing model

| Stage | Implementation | Transformation |
|---|---|---|
| $T_1$ Input | `SETMCP` / `SETCSL` / `GETSCD` | Files to orbital table, CSF blocks, MCP handles, initial $(P,Q)$, weights, and varied-orbital selection |
| $T_2$ CI diagonalization | `MATRIX` -> `SETHAM` + `MANEIG` | Angular coefficients and radial integrals to sparse $H$, then `DSPEVX` or `GDVD` to $(\lambda_\alpha,c_\alpha)$ |
| $T_3$ Level reduction | `NEWCO` | Eigenpairs to weights, generalized occupations `UCF(a)`, and weighted mean energy |
| $T_4$ Orbital update | `SETLAG` + `IMPROV` -> `COFPOT` / `DEFCOR` / `SOLVE` / `DAMPOR` | Current potential to one improved orbital $(P_a,Q_a)$ |
| $T_5$ Convergence/output | End of `SCF` + `ORBOUT` / `ENDSUM` | Write `rwfn.out` and `rmcdhf.sum`; EOL also updates `rmix.out` |

## Stage 1: input (`SETMCP` -> `SETCSL` -> `GETSCD`)

- **Code:** `rscfvu.f90:132-152`, `setmcp.f90:78-137`, `setcsl.f90`,
  `getscd.f90:3-495`, and `getold.f90:60-140`.
- `SETMCP` reads the global header produced by `rangular` from `mcp.30`:
  `(NCORE,NBLOCK,KMAXF)`, `NCFBLK`, and `IDBLK`, followed by
  `(DIAG,ICCUT,LFORDR)` from each file. The code does not compare the trailing
  values between MCP files; each later read overwrites module state. `DIAG`
  alone selects AL versus EOL: true selects AL, otherwise EOL. `ICCUT` is used
  here only in the `STRSUM` summary.
- `SETCSL` calls `LODCSH2GG` to load packed CSF occupations `IQA` and the signed
  $2J+1$ value `JPGG` for every block. The 2017 version removed the global
  `JQSA/JCUPA` arrays, so they are not outputs of this stage.
- `GETSCD`, analogous to `GETINF` in `rwfnestimate90`, establishes nuclear and
  radial-grid defaults. For a point nucleus, $r_0=e^{-65/16}/Z$ and $h=2^{-4}$;
  for a finite nucleus, $r_0=2\times10^{-6}/Z$ and $h=0.05$.
  It sets $\mathrm{ACCY}=h^6$ unless overridden, loads initial orbitals through
  `SETRWFA`, and sets $\mathrm{NNODEP}=n-l-1$ (`getscd.f90:181-185`).
- With `DIAG=.TRUE.`, the program follows AL without diagonalization; otherwise
  it follows EOL (`getscd.f90:187-204`). In EOL, `GETOLD` uses `LODSTATE` to
  select levels per block, `GETOLDWT` to read weights, and prompts for
  `LFIX/IORDER` and the spectroscopic/correlation classification. Initial
  defaults are `NSCF=24`, `NSOLV=3`, `ORTHST=.TRUE.`, `METHOD=3`, and
  `ODAMP=0`; spectroscopic orbitals are later changed to `METHOD=1`.
  `GETSCD` unconditionally reads a new `NSCF` in either mode, overriding 24 in
  EOL or 12 in AL (`getscd.f90:206-211`).

### Selecting spectroscopic orbitals in LBL expansions

After `Enter orbitals to be varied (Updating order)`, `GETOLD` asks
`Which of these are spectroscopic orbitals?` (`getold.f90:88-135`). The second
prompt classifies the already selected varied orbitals; it does not request the
whole varied list again.

- **Spectroscopic orbitals** are physical shells occupied in the original
  reference or multireference configurations, including the relevant core,
  valence, and target excited-state orbitals.
- **Correlation orbitals** are added to the active space to describe electron
  correlation. An ordinary $n\ell j$ label does not make one spectroscopic if
  it enters only through excitations from the multireference.

If $\mathcal V_k$ is the varied set at layer $k$, `MR` is the reference set
before correlation CSFs are generated, and $q_a(\Phi)$ is the occupation of
orbital $a$ in reference configuration $\Phi$, then

$$
\mathcal S_{\mathrm{spec}}=
\{a\mid\exists\Phi\in\mathrm{MR},q_a(\Phi)>0\},
\qquad
\mathcal S_{\mathrm{input},k}=\mathcal V_k\cap\mathcal S_{\mathrm{spec}}.
$$

Use the original reference/MR configurations, not the expanded CSF list. The
latter may occupy every active orbital somewhere and would misclassify all of
them as spectroscopic. Classification also depends on the target: `3s` is
spectroscopic for a target reference containing $\cdots2p\,3s$, but it is a
correlation orbital if added only to correlate a ground state.

In a standard layer-by-layer expansion, old core and valence spectroscopic
orbitals are normally optimized and frozen, while only the newly added
correlation layer $\mathcal A_k$ varies. Thus

$$
\mathcal V_k=\mathcal A_k,
\qquad \mathcal A_k\cap\mathcal S_{\mathrm{spec}}=\varnothing,
$$

so the second prompt should receive an empty line, not `*`:

```text
Enter orbitals to be varied (Updating order)
4s,4p-,4p,4d-,4d,4f-,4f
Which of these are spectroscopic orbitals?
<press Enter without entering an orbital>
```

When physical reference orbitals are reoptimized together with a new
correlation layer, enter their intersection. For a carbon-like reference
$1s^2 2s^2 2p^2$ with varied valence and new $n=3$ orbitals:

```text
Enter orbitals to be varied (Updating order)
2s,2p-,2p,3s,3p-,3p,3d-,3d
Which of these are spectroscopic orbitals?
2s,2p-,2p
```

Here `2p-` and `2p` denote $2p_{1/2}$ and $2p_{3/2}$; all listed $n=3$
orbitals are correlation orbitals. Enter `*` only when every varied orbital is
physical in the reference, as in an initial DHF/MCDHF optimization. A fixed
spectroscopic orbital is ignored even if entered because `LFIX(LOC)=.TRUE.`.
`GETRSL` accepts spaces, commas, and `*` (`lib/lib9290/getrsl.f90:3-169`).

The program initially sets `LCORRE=.TRUE.` for all orbitals. For every selected
nonfixed spectroscopic orbital it then sets (`getold.f90:113-126`)

$$
\mathrm{METHOD}(a)=1,\quad \mathrm{NOINVT}(a)=\mathrm{false},\quad
\mathrm{ODAMP}(a)=0,\quad \mathrm{LCORRE}(a)=\mathrm{false}.
$$

Misclassifying a pure LBL layer with `*` therefore changes the integration
method, node-sign control, damping state, and orthogonalization order within a
$\kappa$ group. If any varied correlation orbital exists, `GETOLD` also sets
the supplementary-improvement count `NSIC` to zero. `ORTHY` rebuilds each
$\kappa$ group in the order fixed, varied spectroscopic, then varied
correlation (`getold.f90:128-135`, `orthy.f90:78-178`).

## Stage 2: CI diagonalization (`MATRIX` -> `SETHAM` + `MANEIG`)

- **Code:** `matrix.f90:3-318`, `setham.f90:3-352`, `maneig.f90`, and
  `lib/libdvd90/dvdson.f90`.
- **Input:** Current $(P,Q)$, sparse structure `IENDC/IROW` from `mcp.30`, and
  angular coefficients from `mcp.31` and `mcp.32+k`.

For each `JBLOCK`, `SETHAM` builds closed-shell diagonal terms directly from
occupations. It evaluates the required $I$, $F^k$, and $G^k$ radial integrals
through `RINTI/SLATER` and `YZK`. For off-diagonal terms it reads each
$I(ab)$ group from `mcp.31` and each $R^k(abcd)$ group from `mcp.32+k`, evaluates
the radial integral once per label, and contracts it with all angular
coefficients in the group. This is the consumer side of `rangular`'s sorted
output.

The block mean

$$E_{\mathrm{av}}=\frac{1}{N_{\mathrm{CF}}}\sum_r H_{rr}$$

is subtracted from the diagonal to improve conditioning (`matrix.f90:178-203`).
For `NCF=1`, the solution is $(\lambda,\mathbf c)=(0,1)`. `INIEST2` expands a
packed upper triangle and calls LAPACK `DSPEVX` whenever `dvdfirst` is true or
`NCF<=4000`. Thus the first diagonalization of a block larger than 4,000 also
uses `DSPEVX` on the leading subblock to initialize the eigenspace. Only later
diagonalizations of such a block reuse the previous eigenvectors and call
`GDVD` directly (`maneig.f90:119-171`, `iniest2.f90:44-82`).

New eigenvectors are damped against the previous vectors, normalized with a
positive leading component, and mutually orthogonalized within each block.
In EOL mode, `MATRIX` writes `rmix.out`: block metadata, selected levels,
`EAV`, relative eigenvalues `EVAL`, and eigenvectors. It stores $E_{\mathrm{av}}$
and $\lambda_\alpha$ separately; consumers must combine them:

$$
\sum_s H_{rs}c_{s\alpha}=
(E_{\mathrm{av}}^{(b)}+\lambda_\alpha)c_{r\alpha}.
$$

## Stage 3: level reduction (`NEWCO`)

`NEWCO` normalizes requested weights,
$w_\alpha=W_\alpha/\sum_\beta W_\beta$, where $W$ is 1, $2J+1$, or a
user-supplied value. It forms the block density

$$
d_{rs}^{(b)}=\sum_{\alpha\in b}w_\alpha c_{r\alpha}c_{s\alpha}
$$

and generalized occupations

$$
\mathrm{UCF}(a)=\sum_b\sum_{r\in b}d_{rr}^{(b)}q_{a,r}.
$$

It also computes
$\bar E=\sum_\alpha w_\alpha(E_{\mathrm{av}}^{(b(\alpha))}+\lambda_\alpha)$,
which supplies the SCF energy criterion, and `CSFWGT` prints the five leading
CSF contributions to each ASF (`newco.f90:3-131`, `dsubrs.f90`, `csfwgt.f90`).

## Stage 4: orbital improvement (`SETLAG` + `IMPROV`)

Once per SCF iteration, `SETLAG` selects Lagrange-multiplier pairs with equal
$\kappa$, at least one varied member, and not both members simultaneously
varied and closed (`setlag.f90:85-100`). It then updates the multipliers from
the current orbitals.

For each varied orbital, `IMPROV(J)`:

1. calls `COFPOT` to assemble direct (`YPOT`), exchange (`XPOT`), Lagrange
   (`LAGCON`), and off-diagonal one-electron (`DACON`) terms;
2. applies the delayed finite-difference correction in `DEFCOR`;
3. calls `SOLVE` (Fischer Algorithms 5.2/5.3), which uses shooting, node
   counting, and energy correction to solve the perturbed radial Dirac
   equation, retrying with `METHOD=2` on failure;
4. evaluates the candidate against the old orbital before normalization and
   damping:

$$
\mathrm{SCNSTY}(a)=\sqrt{\mathrm{UCF}(a)}
\max_i\left(|P_a^{\mathrm{cand}}(r_i)-P_a^{\mathrm{old}}(r_i)|+
|Q_a^{\mathrm{cand}}(r_i)-Q_a^{\mathrm{old}}(r_i)|\right);
$$

5. normalizes the candidate and, if unconverged, applies

$$
(P_a,Q_a)_{\mathrm{store}}=(1-\mu_a)(P_a,Q_a)_{\mathrm{cand}}+
\mu_a(P_a,Q_a)_{\mathrm{old}},
$$

   followed by renormalization. Negative `ODAMP` specifies fixed
   $|\mathrm{ODAMP}|`; nonnegative values are adjusted by `DAMPCK` from recent
   energy changes (`dampck.f90:35-52`, `dampor.f90:55-117`).

When `ORTHST` is true, `ORTHY` rebuilds the whole equal-$\kappa$ group, not
only the current orbital. Fixed orbitals come first, followed by varied
spectroscopic and varied correlation orbitals. With `LSORT`, the first group is
ordered by $\mathrm{SCNSTY}/E^2$ and the second by `SCNSTY`; all nonfixed
orbitals are then Schmidt-orthogonalized (`orthy.f90:78-226`).

## Stage 5: convergence and output

EOL performs `MATRIX+NEWCO` before entering the loop. Each SCF iteration runs
`SETLAG`, improves every orbital in `IORDER`, and performs at most `NSIC`
supplementary updates of the orbital selected by `MAXARR`. The orbital test is

$$C_{\mathrm{orb}}=[\max_{a\in\mathcal V}\mathrm{SCNSTY}(a)\le\mathrm{ACCY}].$$

If `ORTHST` is false, `ORTHSC` follows. `ORBOUT` writes `rwfn.out`; EOL then
runs `MATRIX+NEWCO` again and checks

$$C_E=\left[\left|\frac{\bar E_k-\bar E_{k-1}}{\bar E_k}\right|
<10^{-3}\mathrm{ACCY}\right].$$

The source stops on the logical OR,
$C_{\mathrm{stop}}=C_{\mathrm{orb}}\lor C_E$ (`scf.f90:176-228`). Every normal
path writes `rwfn.out` and `rmcdhf.sum`; only EOL creates and rewrites
`rmix.out`. `rmcdhf.log` is always opened. Nondefault debug mode may also write
`rscf92.dbg` or a user-selected debug file.

## State and unresolved paths

- The outer SCF loop (at most `NSCF`) contains a varied-orbital sweep and up to
  `NSIC` supplementary updates. In EOL, each outer iteration also contains a
  full blockwise CI diagonalization.
- Cross-iteration state includes `EVEC`, `SCNSTY`, `IORDER`, and the Lagrange
  multiplier arrays `IECC/ECV`. `METHOD`, `ODAMP/CDAMP`, and `NSIC` can change
  during iteration.
- `ORBOUT` persists `rwfn.out` every iteration; EOL `MATRIX` rewrites
  `rmix.out` from the beginning.
- In nondefault debug mode, `SETDBG` ultimately forces `LDBPG(1:5)`,
  `LDBPR(1:30)`, and `LDBPA(1:3)` all true, regardless of the preceding
  item-by-item prompts.
- **Incomplete AL initialization:** Unlike `GETOLD`, `GETALD` does not
  initialize `LFIX`, `NFIX`, or `IORDER`, although `SCF` later uses them.
  `GETALDWT` also controls user-weight input with an uninitialized local
  `NCMIN`, and AL does not call `NEWCO` even though `SCF` uses `WTAEV` in its
  energy test. The behavior of the current AL path is therefore undefined by
  this static analysis.
- The multiplier expressions in `SETLAG`, the finite-difference coefficients
  inside `SOLVE/IN/OUT/NEWE`, and the angular algebra in `FCO/GCO` were traced
  to their call sites but not independently rederived.
- This analysis covers only serial semantics. The MPI version changes row
  distribution, reductions, and communication; `mpi_s.f90` is only a stub here.

## Workflow map

- `rcsfgenerate90`, or `rcsfinteract90` after MR reduction, produces blocked
  `rcsf.inp`.
- `rwfnestimate90` produces initial `rwfn.inp`.
- `rangular90` produces `mcp.30..32+KMAXF`.
- `rmcdhf90` consumes these inputs, always writes `rwfn.out`, and writes
  `rmix.out` only in EOL mode.
- Downstream GRASP tools such as `rci`, `rci_hf`, `rhfs`, and `rtransition` can
  consume the converged orbitals or mixing coefficients; those consumer
  relationships are external to `rmcdhf90` itself.
