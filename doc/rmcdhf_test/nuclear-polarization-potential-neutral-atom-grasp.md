---
title: "Nuclear-Polarization Potentials in Neutral-Atom GRASP/RMCDHF Calculations: Relevance, Current Status, and Implementation"
slug: "nuclear-polarization-potential-neutral-atom-grasp"
query: "Analyze the relevance of nuclear-polarization potentials to neutral-atom calculations and how they are used in GRASP rmcdhf calculations"
source_pages:
  - flambaum_2021_Nuclear
  - effective-nuclear-polarization-potential
  - nuclear-polarization
  - nuclear-polarizability
  - mcdhf
  - grasp2018
  - jonsson_2023_Introduction
  - koziol_2026_Rciq
  - rci-flambaum-ginges-potential-improves-qed
  - isotope-shift
  - isotope-shift-king-plot
date_created: 2026-08-20
artifact_type: method-analysis
domain: atomic-structure
tags: [nuclear-polarization, effective-potential, neutral-atoms, GRASP2018, RMCDHF, MCDHF, RCI, isotope-shift]
---

# Nuclear-Polarization Potentials in Neutral-Atom GRASP/RMCDHF Calculations: Relevance, Current Status, and Implementation

> **Role in this project (2026-09-15):** The nuclear-polarization potential is the next
> Hamiltonian-physics task after completion of partner-orbital updates, so this document is retained.
> It has not yet been implemented in the current `rmcdhf_mpi`, nor does it explain the existing
> level-ordering error caused by one-sided `nl` relaxation. Implementation should compare NP0
> (no-potential baseline) → NP1 (fixed-ASF expectation value) → NP2 (fixed-orbital RCI) → NP3
> (self-consistent RMCDHF potential), adding only one physical effect at each stage.

## Conclusion

Nuclear-polarization potentials are clearly relevant to neutral-atom calculations, particularly precision energy-level, isotope-shift, and King-plot calculations for medium and heavy elements. However, the current wiki provides no evidence that standard GRASP2018 `rmcdhf` implements such a potential natively. The literature gives local models and parameterizations that can be added to the nuclear Coulomb potential; using them in GRASP generally requires modifying the one-electron potential or one-electron matrix-element code in `rmcdhf` or `rci`. [[flambaum_2021_Nuclear]] [[effective-nuclear-polarization-potential]] [[grasp2018]]

## 1. Physical Relevance to Neutral-Atom Calculations

Nuclear polarization is not ordinary electronic core polarization. Instead, an orbital electron excites the nucleus through a virtual photon, and the nuclear response feeds back into the electronic energy level. Its scalar part can be represented by a local one-electron potential:

$$
V_L(r)=-\frac{e^2}{2}\frac{\alpha_0^{EL}}{r^{2L+2}+b^{2L+2}},
$$

where

$$
\alpha_0^{EL}=\frac{8\pi}{2L+1}\frac{B(EL;L\rightarrow0)}{E_L}.
$$

The principal contributions are the E1 giant-dipole-resonance potential

$$
V_1(r)=-\frac{e^2}{2}\frac{\alpha_0^{E1}}{r^4+b^4},
$$

and the low-energy rotational E2 potential of a deformed nucleus

$$
V_2(r)=-\frac{e^2}{2}\frac{\bar{\alpha}_0^{E2}}{r^6+\tilde b^6}.
$$

See [[effective-nuclear-polarization-potential]] and [[nuclear-polarizability]] for the forms of these potentials and their relation to nuclear polarizability.

### 1.1 Characteristics in Neutral Atoms

- Absolute nuclear-polarization energy shifts in neutral atoms are generally small, but increase with $Z$. Existing parameterizations mainly target medium and heavy atoms with $Z\geq20$. [[flambaum_2021_Nuclear]]
- Direct contributions come primarily from $s_{1/2}$ orbitals near the nucleus. The $p_{1/2}$ contribution is next, typically one or two orders of magnitude smaller; direct contributions from $p_{3/2}$, $d$, and $f$ orbitals are very small. [[nuclear-polarization]]
- Even high-angular-momentum orbitals with little density near the nucleus can be affected indirectly through self-consistent relaxation of the electronic core. The nuclear-polarization potential first changes the $s$ electrons and then affects other electrons through the Hartree–Fock mean field. [[flambaum_2021_Nuclear]]
- For transition frequencies, common shifts of closed core shells partially cancel. The relevant quantity is

$$
\Delta E_{\mathrm{NP}}^{u}-\Delta E_{\mathrm{NP}}^{l}.
$$

Thus, the greater the difference in electron density, configuration mixing, or $s/p_{1/2}$ content between the upper and lower levels, the more likely the transition nuclear-polarization correction is to remain observable.

- The effect deserves more attention in isotope shifts because both $\alpha_0^{EL}$ and the cutoff parameter $b$ depend on the mass number $A$, nuclear radius $R$, and deformation parameter $\beta_2$. The E2 term varies nonmonotonically with $\beta_2$ and may produce King-plot nonlinearity. [[isotope-shift]] [[isotope-shift-king-plot]]

The nuclear-polarization potential is therefore not the primary correction in routine neutral-atom spectroscopy, but it is directly relevant to high-precision isotope shifts, King plots, nuclear-radius extraction, and contact observables in heavy atoms.

## 2. Current Status in GRASP2018

The standard GRASP RMCDHF Hamiltonian is

$$
H_{\mathrm{DC}}=
\sum_i\left[
c\boldsymbol{\alpha}_i\cdot\boldsymbol{p}_i
+c^2(\beta_i-I)
+V_{\mathrm{nuc}}(r_i)
\right]
+\sum_{i<j}\frac{1}{r_{ij}}.
$$

`rnucleus` generates $V_{\mathrm{nuc}}(r)$ from a two-parameter Fermi nuclear-charge distribution. `rmcdhf` uses it to optimize orbitals and CSF coefficients self-consistently, after which `rci` adds Breit, QED, and other corrections with fixed orbitals and rediagonalizes. [[jonsson_2023_Introduction]] [[mcdhf]]

The current wiki supports the following conclusions:

1. [[flambaum_2021_Nuclear]] explicitly proposes adding $V_L(r)$ to the nuclear Coulomb potential and then solving the self-consistent relativistic Hartree–Fock equations.
2. The paper also states explicitly that complete numerical calculations for many-electron systems are outside its scope and left for future work.
3. [[nuclear-polarization]] lists “complete self-consistent MCDHF/RCI calculations for specific many-electron systems” as an open problem.
4. The wiki contains no standard `rmcdhf` input option, official GRASP module, or existing neutral-atom example that directly enables a nuclear-polarization potential.
5. [[koziol_2026_Rciq]] implements the Flambaum–Ginges self-energy radiative potential and vacuum-polarization potential, not a nuclear-polarization potential. It serves only as an architectural example of how to integrate a local potential into GRASP `rci`. The related claim [[rci-flambaum-ginges-potential-improves-qed]] has status `supported` and confidence $0.80$, but its evidence does not support the assertion that a nuclear-polarization potential has been implemented in GRASP.

Thus, the physical interface between a nuclear-polarization potential and GRASP is clear, but the wiki does not establish the existence of a ready-to-use user interface.

## 3. Self-Consistent Form in `rmcdhf`

To include nuclear polarization self-consistently, the RMCDHF Hamiltonian should become

$$
H_{\mathrm{DC+NP}}=H_{\mathrm{DC}}+\sum_i\left[V_1(r_i)+V_2(r_i)\right].
$$

Equivalently, in the radial MCDHF equations of `rmcdhf`, make the replacement

$$
V_{\mathrm{nuc}}(r)\longrightarrow V_{\mathrm{nuc}}(r)+V_1(r)+V_2(r).
$$

Because $V_1$ and $V_2$ are both spherically symmetric scalar one-electron potentials, their radial matrix elements are

$$
\langle a|V_{\mathrm{NP}}|b\rangle
=\delta_{\kappa_a\kappa_b}
\int_0^\infty
\left[P_a(r)P_b(r)+Q_a(r)Q_b(r)\right]
V_{\mathrm{NP}}(r)\,dr.
$$

This approach does not change the angular-coefficient structure of the CSFs; it only adds one-electron radial integrals between orbitals with the same $\kappa$. This formula is an implementation deduction based on GRASP's local one-electron-potential structure and the local radiative-potential integral in [[koziol_2026_Rciq]], not a record of an existing GRASP nuclear-polarization implementation in the wiki.

### 3.1 Implementation Considerations

- A nuclear-polarization potential cannot be simulated simply by changing the Fermi nuclear radius. The radial structures of finite-nuclear-size and nuclear-polarization potentials differ: the latter decay as $r^{-4}$ and $r^{-6}$, respectively, at long range.
- $b$ and $\tilde b$ are isotope-dependent parameters and should be obtained from the fits in [[flambaum_2021_Nuclear]] and the corresponding nuclear parameters.
- The literature generally gives $b$ in $\mathrm{fm}$; it must be converted to $a_0$ before use on the GRASP radial grid.
- The potentials vary mainly over roughly tens to one hundred $\mathrm{fm}$, so integration convergence on the GRASP exponential radial grid near the nucleus must be checked.
- If the potential is added to `rmcdhf`, the same potential should remain when the final `rci` Hamiltonian is constructed; otherwise the SCF orbitals and final CI Hamiltonian are inconsistent.
- Each isotope must use its own $A$, $R$, $\beta_2$, and polarizability parameters.

## 4. Three Feasible Computational Routes

| Route | Method | Includes orbital relaxation? | Suitable purpose |
|---|---|---:|---|
| Postprocessed expectation value | Use the existing ASF density to calculate $\langle\Psi|V_{\mathrm{NP}}|\Psi\rangle$ | No | Rapid order-of-magnitude estimate |
| Add at the RCI stage | Keep `rmcdhf` orbitals fixed, add one-electron matrix elements to the `rci` Hamiltonian, and rediagonalize | Partial: includes CSF remixing, not orbital relaxation | Sensitivity analysis and small corrections |
| Add self-consistently in RMCDHF | Add $V_{\mathrm{NP}}$ to the radial potential in `rmcdhf`, reoptimize orbitals and CSF coefficients, then perform a consistent RCI calculation | Yes | Recommended final high-precision approach |

The first method directly calculates

$$
\Delta E_{\Gamma J}^{\mathrm{NP}}
=\sum_{ab}\rho_{ab}^{\Gamma J}\langle a|V_{\mathrm{NP}}|b\rangle,
$$

where $\rho_{ab}^{\Gamma J}$ is the ASF one-body reduced density matrix. This is suitable for deciding whether the effect warrants a complete self-consistent calculation.

For heavy neutral atoms, the third route is recommended for final calculations. The literature specifically notes that nuclear-polarization corrections to high-angular-momentum electrons may arise mainly from the self-consistent mean-field response caused by changes to the $s$ core; pure postprocessing misses this contribution. [[flambaum_2021_Nuclear]]

## 5. Recommended GRASP Validation Procedure

1. Run standard `rmcdhf + rci` to obtain a baseline without the nuclear-polarization potential.
2. Implement the E1 and E2 potentials separately at first, to make numerical problems easier to isolate.
3. Test the radial integrals on hydrogenlike systems and reproduce the published $1s$, $2s$, or $2p_{1/2}$ shifts.
4. Then calculate the first-order correction for a neutral atom with fixed orbitals, confirming its sign and order of magnitude.
5. Finally, perform otherwise identical `rmcdhf + rci` calculations with the potential enabled and disabled, and calculate

$$
\delta E_{\mathrm{relax}}
=\Delta E_{\mathrm{self-consistent}}
-\Delta E_{\mathrm{fixed-orbital}}.
$$

6. Report the E1, E2, direct, orbital-relaxation, and total corrections separately.
7. For isotope shifts, calculate

$$
\delta\nu_{\mathrm{NP}}^{A,A'}
=\frac{
[\Delta E_u^{\mathrm{NP}}(A')-\Delta E_l^{\mathrm{NP}}(A')]
-[\Delta E_u^{\mathrm{NP}}(A)-\Delta E_l^{\mathrm{NP}}(A)]
}{h}.
$$

8. Check convergence with respect to the active space and radial grid near the nucleus, as well as uncertainties in $b$ and $\beta_2$.

## 6. Final Assessment

The nuclear-polarization potential is best regarded as a short-range, isotope-dependent, spherically symmetric one-electron correction to the GRASP Hamiltonian. For heavy neutral atoms:

- For order-of-magnitude estimates of energy levels, a postprocessed expectation value is generally sufficient.
- For precision transition energies or isotope shifts, it should at least be added to `rci` followed by rediagonalization.
- To include the electronic-core response emphasized in the literature, it must be added self-consistently within the `rmcdhf` SCF loop.

## 7. Knowledge Gaps and Evidence Boundaries

- The wiki contains no validated implementation of a “nuclear-polarization-enabled GRASP `rmcdhf`.”
- The wiki contains no convergence study or experimental benchmark for a nuclear-polarization potential in a specific neutral many-electron atom.
- [[flambaum_2021_Nuclear]] supports the methodological direction of adding a local potential to the nuclear Coulomb potential and solving self-consistently, but provides no GRASP source implementation.
- [[koziol_2026_Rciq]] demonstrates only that a local radiative potential can be integrated into GRASP `rci`; extending that architecture by analogy to a nuclear-polarization potential is an implementation recommendation, not direct validation.
- A complete Standard Model error budget must still treat finite nuclear size, mass shift, QED, nuclear deformation, and nuclear polarization jointly while avoiding double counting.

The Hamiltonian-level integration approach therefore has a clear basis, but the exact Fortran modification points, radial-grid stability, and numerical accuracy for neutral atoms still require further validation against the GRASP2018 source and benchmark cases.
