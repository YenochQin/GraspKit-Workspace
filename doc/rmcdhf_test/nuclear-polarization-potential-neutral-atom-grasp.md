---
title: "中性原子 GRASP/RMCDHF 计算中的核极化势：关联、现状与实现方式"
slug: "nuclear-polarization-potential-neutral-atom-grasp"
query: "分析核极化势在计算中性原子的过程中有没有关联，以及在 GRASP 的 rmcdhf 计算中的使用情况与方式"
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

# 中性原子 GRASP/RMCDHF 计算中的核极化势：关联、现状与实现方式

> **本项目中的任务定位（2026-09-15）**：核极化势是伙伴原子更新完成后的下一项
> Hamiltonian 物理任务，本文档保留。它尚未在当前 `rmcdhf_mpi` 中实现，也不是现有
> 单侧 `nl` 松弛导致能级错序的既有解释。实施时按 NP0（无势基线）→ NP1（固定 ASF
> 期望值）→ NP2（固定轨道 RCI）→ NP3（RMCDHF 自洽势）逐级比较，一次只增加一个
> 物理因素。

## 结论

核极化势与中性原子计算存在明确关联，尤其适用于中、重元素的精密能级、同位素位移和 King plot 计算；但当前 wiki 没有证据表明标准 GRASP2018 的 `rmcdhf` 已经原生实现了该势。现有文献给出了可加入核库仑势的局域模型和参数化公式，真正用于 GRASP 时通常需要修改 `rmcdhf` 或 `rci` 中的一电子势或一电子矩阵元代码。[[flambaum_2021_Nuclear]] [[effective-nuclear-polarization-potential]] [[grasp2018]]

## 1. 与中性原子计算的物理关联

核极化不是原子电子的普通芯极化，而是轨道电子通过虚光子激发原子核，随后核响应反馈到电子能级。其标量部分可用局域一电子势表示：

$$
V_L(r)=-\frac{e^2}{2}\frac{\alpha_0^{EL}}{r^{2L+2}+b^{2L+2}},
$$

其中

$$
\alpha_0^{EL}=\frac{8\pi}{2L+1}\frac{B(EL;L\rightarrow0)}{E_L}.
$$

主要包括 E1 巨偶极共振势

$$
V_1(r)=-\frac{e^2}{2}\frac{\alpha_0^{E1}}{r^4+b^4},
$$

以及形变核低能转动 E2 势

$$
V_2(r)=-\frac{e^2}{2}\frac{\bar{\alpha}_0^{E2}}{r^6+\tilde b^6}.
$$

势的形式及其与核极化率之间的关系见 [[effective-nuclear-polarization-potential]] 和 [[nuclear-polarizability]]。

### 1.1 中性原子中的作用特点

- 中性原子中的绝对核极化能移通常较小，但会随 $Z$ 增大而增强。现有参数化主要面向 $Z\geq20$ 的中重原子。[[flambaum_2021_Nuclear]]
- 直接贡献主要来自靠近原子核的 $s_{1/2}$ 轨道；$p_{1/2}$ 次之，通常低一至两个数量级；$p_{3/2}$、$d$、$f$ 等轨道的直接贡献很小。[[nuclear-polarization]]
- 即使高角动量轨道在核附近密度很小，它们仍可能通过电子芯轨道的自洽松弛受到间接影响。核极化势先改变 $s$ 电子，再通过 Hartree–Fock 平均场影响其他电子。[[flambaum_2021_Nuclear]]
- 对跃迁频率而言，封闭芯层的共同能移会部分相消，实际关心的是

$$
\Delta E_{\mathrm{NP}}^{u}-\Delta E_{\mathrm{NP}}^{l}.
$$

因此，上下能级的电子密度、组态混合或 $s/p_{1/2}$ 成分差异越大，跃迁核极化修正越可能留下可观测贡献。

- 在同位素位移中更值得注意，因为 $\alpha_0^{EL}$ 和截断参数 $b$ 都依赖质量数 $A$、核半径 $R$ 和形变参数 $\beta_2$。E2 项随 $\beta_2$ 非单调变化，可能产生 King plot 非线性。[[isotope-shift]] [[isotope-shift-king-plot]]

因此，核极化势不是中性原子常规能谱计算的首要修正，但对重原子高精度同位素位移、King plot、核半径提取和接触型观测量具有直接关联。

## 2. GRASP2018 中目前的使用状态

标准 GRASP 的 RMCDHF 哈密顿量为

$$
H_{\mathrm{DC}}=
\sum_i\left[
c\boldsymbol{\alpha}_i\cdot\boldsymbol{p}_i
+c^2(\beta_i-I)
+V_{\mathrm{nuc}}(r_i)
\right]
+\sum_{i<j}\frac{1}{r_{ij}}.
$$

`rnucleus` 根据两参数 Fermi 核电荷分布生成 $V_{\mathrm{nuc}}(r)$；`rmcdhf` 使用它进行轨道与 CSF 系数的自洽优化，随后 `rci` 在固定轨道基础上加入 Breit、QED 等修正并重新对角化。[[jonsson_2023_Introduction]] [[mcdhf]]

当前 wiki 支持以下判断：

1. [[flambaum_2021_Nuclear]] 明确提出将 $V_L(r)$ 加到核库仑势，然后求解自洽 relativistic Hartree–Fock 方程。
2. 该论文同时明确说明，完整多电子数值计算不在论文范围内，并留待后续工作。
3. [[nuclear-polarization]] 把“在特定多电子体系中进行完整自洽 MCDHF/RCI 计算”列为开放问题。
4. wiki 中没有检索到标准 `rmcdhf` 输入选项、GRASP 官方模块或已有中性原子算例可以直接启用 nuclear-polarization potential。
5. [[koziol_2026_Rciq]] 实现的是 Flambaum–Ginges 自能辐射势和真空极化势，不是核极化势。它只能作为“如何把局域势接入 GRASP `rci`”的代码架构范例。相关 claim [[rci-flambaum-ginges-potential-improves-qed]] 的状态是 `supported`、置信度为 $0.80$，但其证据不支持“核极化势已经在 GRASP 中实现”。

因此，核极化势与 GRASP 的物理接口是明确的，现成用户接口尚未在 wiki 中得到证实。

## 3. 在 `rmcdhf` 中自洽使用的形式

如果希望自洽考虑核极化，RMCDHF 哈密顿量应改为

$$
H_{\mathrm{DC+NP}}=H_{\mathrm{DC}}+\sum_i\left[V_1(r_i)+V_2(r_i)\right].
$$

等价地，在 `rmcdhf` 的径向 MCDHF 方程中执行

$$
V_{\mathrm{nuc}}(r)\longrightarrow V_{\mathrm{nuc}}(r)+V_1(r)+V_2(r).
$$

因为 $V_1$、$V_2$ 都是球对称标量一电子势，其径向矩阵元为

$$
\langle a|V_{\mathrm{NP}}|b\rangle
=\delta_{\kappa_a\kappa_b}
\int_0^\infty
\left[P_a(r)P_b(r)+Q_a(r)Q_b(r)\right]
V_{\mathrm{NP}}(r)\,dr.
$$

这种加入方式不会改变 CSF 的角系数结构，只增加同一 $\kappa$ 轨道间的一电子径向积分。该式是基于 GRASP 一电子局域势结构与 [[koziol_2026_Rciq]] 中局域辐射势积分形式得到的实现推导，尚不是 wiki 中已有的核极化势 GRASP 实现记录。

### 3.1 实现注意事项

- 不能简单通过修改 Fermi 核半径来模拟核极化势。有限核尺寸势和核极化势的径向结构不同：后者在远处分别按 $r^{-4}$ 和 $r^{-6}$ 衰减。
- $b$、$\tilde b$ 是同位素相关参数，应从 [[flambaum_2021_Nuclear]] 的拟合式及相应核参数得到。
- 文献中的 $b$ 通常以 $\mathrm{fm}$ 给出，写入 GRASP 径向网格前必须换算为 $a_0$。
- 势的主要变化区域约在几十到一百 $\mathrm{fm}$，需要检查 GRASP 指数径向网格在核附近的积分收敛性。
- 如果 `rmcdhf` 中加入了该势，最终 `rci` 构造哈密顿量时也应保留同一势，否则 SCF 轨道和最终 CI 哈密顿量不一致。
- 同位素之间必须使用各自的 $A$、$R$、$\beta_2$ 和极化率参数。

## 4. 三种可行计算路线

| 路线 | 做法 | 能否包含轨道松弛 | 适用目的 |
|---|---|---:|---|
| 后处理期望值 | 用现有 ASF 密度计算 $\langle\Psi|V_{\mathrm{NP}}|\Psi\rangle$ | 否 | 快速量级估计 |
| RCI 阶段加入 | 固定 `rmcdhf` 轨道，在 `rci` 哈密顿量中加入一电子矩阵元并重新对角化 | 部分：包含 CSF 重混合，不包含轨道松弛 | 灵敏度分析、较小修正 |
| RMCDHF 自洽加入 | 在 `rmcdhf` 径向势中加入 $V_{\mathrm{NP}}$，重新优化轨道与 CSF 系数，再做一致的 RCI | 是 | 推荐的最终高精度方案 |

第一种方法可直接计算

$$
\Delta E_{\Gamma J}^{\mathrm{NP}}
=\sum_{ab}\rho_{ab}^{\Gamma J}\langle a|V_{\mathrm{NP}}|b\rangle,
$$

其中 $\rho_{ab}^{\Gamma J}$ 是 ASF 的一体约化密度矩阵。它适合判断效应是否值得进入完整自洽计算。

对于重中性原子，推荐最终采用第三种路线，因为文献特别指出，高角动量电子所受到的核极化修正可能主要来自 $s$ 芯层变化引起的自洽平均场响应；纯后处理会漏掉这一部分。[[flambaum_2021_Nuclear]]

## 5. 建议的 GRASP 验证流程

1. 运行标准 `rmcdhf + rci`，得到无核极化势基准。
2. 分别实现 E1 和 E2 势，初期不要合并，以便定位数值问题。
3. 先在类氢体系上检验径向积分，复现文献中的 $1s$、$2s$ 或 $2p_{1/2}$ 能移。
4. 再以固定轨道方式计算中性原子的一阶修正，确认符号和数量级。
5. 最后执行势开启与关闭的两套完全相同的 `rmcdhf + rci`，并计算

$$
\delta E_{\mathrm{relax}}
=\Delta E_{\mathrm{self-consistent}}
-\Delta E_{\mathrm{fixed-orbital}}.
$$

6. 分别报告 E1、E2、直接项、轨道松弛项以及总修正。
7. 对同位素位移计算

$$
\delta\nu_{\mathrm{NP}}^{A,A'}
=\frac{
[\Delta E_u^{\mathrm{NP}}(A')-\Delta E_l^{\mathrm{NP}}(A')]
-[\Delta E_u^{\mathrm{NP}}(A)-\Delta E_l^{\mathrm{NP}}(A)]
}{h}.
$$

8. 检查活动空间、核附近径向网格、$b$ 参数和 $\beta_2$ 不确定度的收敛性。

## 6. 最终判断

核极化势适合被看作 GRASP 哈密顿量中的一个短程、同位素相关、球对称一电子修正。对于中性重原子：

- 做能级数量级估计：后处理期望值通常足够；
- 做精密跃迁能或同位素位移：至少应在 `rci` 中加入并重新对角化；
- 要包含文献强调的电子芯轨道响应：必须在 `rmcdhf` 的 SCF 循环中自洽加入。

## 7. 知识缺口与证据边界

- wiki 中尚无已经验证的“核极化势版 GRASP `rmcdhf`”实现。
- wiki 中尚无特定中性多电子原子的核极化势收敛研究或实验基准。
- [[flambaum_2021_Nuclear]] 支持“将局域势加入核库仑势并自洽求解”的方法方向，但不提供 GRASP 源码实现。
- [[koziol_2026_Rciq]] 只证明局域辐射势能够接入 GRASP `rci`；将其架构类比到核极化势属于实现建议，不是核极化势的直接验证。
- 完整的标准模型误差预算仍需联合处理有限核尺寸、质量位移、QED、核形变与核极化，避免重复计数。

因此，哈密顿量层面的接入方式具有明确依据，但具体 Fortran 修改位置、径向网格稳定性和中性原子数值准确度仍需结合 GRASP2018 源码与基准算例进一步验证。
