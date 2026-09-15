---
artifact_type: code_analysis
title: "GRASP2018 rmcdhf90 — 多组态 Dirac–Hartree–Fock 自洽场流水线"
target: rmcdhf_test/src/appl/rmcdhf90
date: 2026-08-16
analysis_objective: 重建 MCDHF 自洽场端到端路径——轨道改进与组态对角化交替迭代，从 rcsf.inp + rwfn.inp + mcp.* 到 rwfn.out 及 EOL 模式的 rmix.out
target_language: fortran90
related_papers: []
lang: zh
---

# 代码过程分析：GRASP2018 `rmcdhf90`

> **用途边界**：本文是原版串行 `rmcdhf90` 的算法参考，用于理解
> `GETOLD → SETLAG → IMPROV → DAMPOR/ORTHY → MATRIX/NEWCO`。它不是当前
> `rmcdhf90_mpi` 修改状态或物理验收的证据；两者有差异时以当前 MPI 源码、
> [代码核查](./rmcdhf90_mpi_轨道优化代码核查.md)和受控测试为准。

## 范围与目标

- **目标代码**：当前工作区的 `rmcdhf_test/src/appl/rmcdhf90`（113 个文件、110 个
  `.f90`：54 个接口/实现对，加 `mpi_s.f90` 模块与 `rscfvu.f90` 主程序；源码修订史为
  1992–2017，是 GRASP92 RSCF92 的分块移植 + Gaigalas 2017 改版）。
- **分析目标**：重建主运行路径——多组态 Dirac–Hartree–Fock 自洽场（MCDHF）迭代：装载 CSF 列表、角系数文件与初始轨道，在 EOL 模式下先对角化一次，然后逐轮执行轨道改进、正交化、轨道写出以及再次组态对角化。停止条件是轨道自洽或加权能量变化满足任一判据，而非要求两者同时满足。
- **范围选择**：单一入口 `PROGRAM RSCFVU`；本目录是**串行版**（`MYID=0, NPROCS=1` 硬编码，`rscfvu.f90:85-86`），MPI 变体在 `rmcdhf90_mpi`。编译产物 `${GRASP}/bin/rmcdhf`，链接 `libdvd90 -l9290 -lmod`（`Makefile:1-2`）；`libdvd90` 只在大于 4000 维的块上通过 `GDVD` 参与求解，小块由 LAPACK `DSPEVX` 直接处理。
- 核心算法出处：SCF 迭代是 C. Froese Fischer, Comput. Phys. Rep. **3** (1986) 290 的 Algorithm 5.1（`scf.f90:5-7`），轨道求解是其 Algorithm 5.2/5.3，页码为 295（`solve.f90:5-6`）。验证仅为静态分析（未编译执行）。

## 入口与端到端概要

```
isodata, rcsf.inp, rwfn.inp, mcp.30..32+KMAXF
  ├─ SETMCP  ← 读回 rangular 写的全局头 (NCORE,NBLOCK,KMAXF,NCFBLK,IDBLK,DIAG,ICCUT,LFORDR)
  ├─ SETCSL  ← 全部 CSF 占据数 IQA + 分块对称性 JPGG
  └─ GETSCD  ← isodata/网格/ACCY; SETRWFA 载入初始轨道;
       EOL? → GETOLD (LODSTATE 选能级, GETOLDWT 权重, LFIX/IORDER 变分轨道) 或 GETALD
  EOL 初始状态 → MATRIX + NEWCO
  SCF 主循环 (Fischer Alg. 5.1):
     SETLAG → {IMPROV(每个变分轨道) + [MAXARR→IMPROV(最不自洽轨道)]×NSIC}
     → 收敛判定 → ORTHSC → ORBOUT(rwfn.out) → [MATRIX + NEWCO]
  输出: rwfn.out, rmcdhf.sum, rmcdhf.log; EOL 时另有 rmix.out
```

数学模型：MCDHF 是对加权平均能量在轨道正交归一约束下的变分优化。设能级权重 $w_\alpha$（$\sum_\alpha w_\alpha = 1$），

$$
E[\{P_a,Q_a\},\{c\}] = \sum_{\alpha} w_\alpha E_\alpha, \qquad E_\alpha = E_{\text{av}}^{(b(\alpha))} + \lambda_\alpha,
$$

对 $P_a,Q_a$ 变分给出带 Lagrange 乘子的径向 Dirac 方程组，对展开系数 $c_{r\alpha}$ 变分给出 H(DC) 本征值问题；EOL 路径先求一次 CI 状态，再交替更新轨道与 CI 状态：

$$
\bigl(\{P,Q\}^{(k+1)}, c^{(k+1)}\bigr) = G\bigl(\{P,Q\}^{(k)}, c^{(k)}\bigr), \qquad \text{迭代映射 } G = \text{CI 对角化} \circ \text{轨道改进}.
$$

## 全局处理模型

| 阶段 | 实现 | 变换 |
|---|---|---|
| $T_1$ 输入装载 | `SETMCP`/`SETCSL`/`GETSCD` | 文件 → 轨道表、CSF 块、mcp 句柄、初始 $(P,Q)$、权重/变分轨道选择 |
| $T_2$ 组态对角化 | `MATRIX`→`SETHAM`+`MANEIG` | 角系数×径向积分 → 稀疏 H → 小块 `DSPEVX` / 大块 `GDVD` → $(\lambda_\alpha,c_\alpha)$ |
| $T_3$ 能级综合 | `NEWCO` | 本征对 → 权重 $w_\alpha$、广义占据数 $\mathrm{UCF}(a)$、加权平均能量 |
| $T_4$ 轨道改进 | `SETLAG` + `IMPROV`→`COFPOT`/`DEFCOR`/`SOLVE`/`DAMPOR` | 当前势场 → 改进的单个 $(P_a,Q_a)$ |
| $T_5$ 收敛与输出 | `SCF` 循环尾部 + `ORBOUT`/`ENDSUM` | → `rwfn.out`, `rmcdhf.sum`；EOL 时更新 `rmix.out` |

## 逐阶段分析

### 阶段 1：输入装载（`SETMCP` → `SETCSL` → `GETSCD`）

- **代码**：`rscfvu.f90:132-152`；`setmcp.f90:78-137`；`setcsl.f90`；`getscd.f90:3-495`；`getold.f90:60-140`。
- **输入**：`isodata`、`rcsf.inp`、`rwfn.inp`、`mcp.*`、stdin。
- **处理**：
  1. `SETMCP` 读回 [[rangular90]] 写入的 `mcp.30` 全局头：`(NCORE,NBLOCK,KMAXF)`、`NCFBLK`、`IDBLK`，并从每个文件读 `(DIAG,ICCUT,LFORDR)`（`setmcp.f90:78-137`）。当前代码不比较各 MCP 文件中的三个末尾值，而是让后读文件覆盖模块状态。EOL/AL 分支实际只由 `DIAG` 决定：`DIAG=.TRUE.` 进入 AL，否则无论 `LFORDR` 为何都进入 EOL；`ICCUT` 在本目录中只用于 `STRSUM` 摘要。
  2. `SETCSL` 调 `LODCSH2GG` 装载全部 CSF 的打包占据数 `IQA`，并记录各块带符号的 $2J+1$ 值 `JPGG`。2017 版已经删除全局 `JQSA/JCUPA` 数组，不能再把它们列为本阶段输出。
  3. `GETSCD`（与 [[rwfnestimate90]] 的 `GETINF` 同构）：核模型/网格默认（点核 $r_0=e^{-65/16}/Z$, $h=2^{-4}$；有限核 $2\times10^{-6}/Z$, $0.05$）、$\mathrm{ACCY}=h^6$（可改）、`SETRWFA` 载入初始轨道、节点数表 $\mathrm{NNODEP}=n-l-1$（`getscd.f90:181-185`）。
  4. **EOL/AL 分支**：`DIAG` → AL（不对角化），否则进入 EOL（`getscd.f90:187-204`，原交互询问已被硬编码为 `.TRUE.`）。EOL 时 `GETOLD`：`LODSTATE` 选取每块能级（`NEVBLK`、`ICCMIN`、总 `NCMIN`），`GETOLDWT` 读权重，并交互确定 `LFIX/IORDER` 以及光谱/关联轨道。初始设置为 `NSCF=24`、`NSOLV=3`、`ORTHST=.TRUE.`、`METHOD=3`、`ODAMP=0`，但光谱轨道随后改为 `METHOD=1`，而且 `GETSCD` 在两种模式下都会无条件读取一个新的 `NSCF` 并覆盖 24 或 AL 的 12（`getscd.f90:206-211`）。
- **输出**：全部状态——轨道、CSF、块结构、能级选择、权重、变分开关。
- **公式**：$T_1: (\text{files}, \text{stdin}) \mapsto \bigl(\{P^{(0)},Q^{(0)}\},\ \{J^\pi_b, N_b\},\ w_{1..N_{\min}},\ \text{LFIX}\bigr)$。

#### LBL 扩展中 spectroscopic orbitals 的选择

`GETOLD` 先询问 `Enter orbitals to be varied (Updating order)`，随后询问 `Which of these are spectroscopic orbitals?`（`getold.f90:88-135`）。第二个问题不是要求再次列出全部可变轨道，而是要在刚才选择的可变轨道中区分**光谱轨道（spectroscopic orbitals）**和**关联轨道（correlation orbitals）**。

- **光谱轨道**：构成目标原子态原始参考组态或多参考（MR）组态的物理壳层轨道，包括实际使用的内壳层、价层和目标激发态轨道。它们具有明确的组态占据意义。
- **关联轨道**：为描述电子关联而在活性空间中附加的轨道。它们可以带有普通的 $n\ell j$ 标签，但若只通过从 MR 出发的单、双等激发进入扩展 CSF，就不因此成为光谱轨道。

设 $\mathcal V_k$ 是第 $k$ 轮输入的可变轨道集合，$\mathrm{MR}$ 是生成关联 CSF 之前由用户定义的原始参考组态集合，$q_a(\Phi)$ 是轨道 $a$ 在参考组态 $\Phi$ 中的占据数，则

$$
\mathcal S_{\mathrm{spec}}
=
\left\{a\;\middle|\;\exists\,\Phi\in\mathrm{MR},\ q_a(\Phi)>0\right\},
\qquad
\mathcal S_{\mathrm{input},k}=\mathcal V_k\cap\mathcal S_{\mathrm{spec}}.
$$

判定时必须使用原始参考/MR 组态，不能使用激发后生成的完整 CSF 列表；否则所有活性轨道都可能在某些关联 CSF 中具有非零占据，从而被错误地标为光谱轨道。同一个标签也取决于计算目标：例如 `3s` 在含 $\cdots 2p\,3s$ 的目标激发态参考组态中是光谱轨道，但在只研究基态、仅为关联展开添加 $3s$ 时是关联轨道。

在标准 layer-by-layer（LBL）活性空间扩展中，旧的 core/valence 光谱轨道通常已经优化并被冻结，每一轮只变分新增加的关联层 $\mathcal A_k$。此时

$$
\mathcal V_k=\mathcal A_k,
\qquad
\mathcal A_k\cap\mathcal S_{\mathrm{spec}}=\varnothing,
$$

所以第二个问题应直接输入**空行**，不能输入 `*`。例如新增 $n=4$ 关联层时：

```text
Enter orbitals to be varied (Updating order)
4s,4p-,4p,4d-,4d,4f-,4f
Which of these are spectroscopic orbitals?
<直接按 Enter，不输入轨道>
```

只有当本轮同时重新优化参考组态中的物理轨道时，才填写二者的交集。例如碳样体系以 $1s^2 2s^2 2p^2$ 为参考，同时变分价轨道和新 $n=3$ 关联层时：

```text
Enter orbitals to be varied (Updating order)
2s,2p-,2p,3s,3p-,3p,3d-,3d
Which of these are spectroscopic orbitals?
2s,2p-,2p
```

其中 `2p-`/$2p` 分别表示 $2p_{1/2}$/$2p_{3/2}$，`3s,3p-,3p,3d-,3d` 仅为关联轨道。如果所有待优化轨道都是参考组态的物理轨道，例如初始 DHF/MCDHF 优化，则可输入 `*`；若某个光谱轨道已经固定，即使在第二个问题中写出，源码也会因 `LFIX(LOC)=.TRUE.` 而忽略它。`GETRSL` 支持空格、逗号和 `*` 通配符（`lib/lib9290/getrsl.f90:3-169`）。

程序先令所有轨道 `LCORRE=.TRUE.`，再对第二个问题中选出的非固定轨道执行（`getold.f90:113-126`）

$$
\mathrm{METHOD}(a)=1,
\qquad
\mathrm{NOINVT}(a)=\mathrm{false},
\qquad
\mathrm{ODAMP}(a)=0,
\qquad
\mathrm{LCORRE}(a)=\mathrm{false}.
$$

因此，把纯 LBL 新层误填为 `*` 不只是名称错误：它会改变积分方法、节点符号控制、阻尼状态和同 $\kappa$ 轨道的正交化顺序。若存在任何可变关联轨道，`GETOLD` 还会将补充改进次数 `NSIC` 置零；`ORTHY` 则按“固定轨道 → 光谱变分轨道 → 关联变分轨道”的顺序重建同 $\kappa$ 组（`getold.f90:128-135`；`orthy.f90:78-178`）。

### 阶段 2：组态对角化（`MATRIX` → `SETHAM` + `MANEIG`，核心阶段之一）

- **代码**：`matrix.f90:3-318`；`setham.f90:3-352`；`maneig.f90`（Davidson 驱动）→ `lib/libdvd90/dvdson.f90`。
- **输入**：当前轨道 $(P,Q)$、`mcp.30`（稀疏结构 `IENDC/IROW`）、`mcp.31`/`mcp.32+k`（角系数表）。
- **处理**（逐块 `JBLOCK`）：
  1. **对角闭壳层项**（不需要 mcp 系数，直接从占据数表生成）：对每行 $r$
$$H_{rr} \mathrel{+}= q_{a,r}\, I(aa) + \mathrm{FCO}(0)\, F^0(aa) + \sum_k \mathrm{FCO}(k)\, F^k(aa) + \mathrm{FCO}(0)\, F^0(ab) + \sum_k \mathrm{GCO}(k)\, G^k(ab)$$
  其中 `FCO/GCO` 是闭壳层角系数的即时计算，$I, F^k, G^k$ 由 `RINTI/SLATER`（底层 `YZK` 径向积分）给出（`setham.f90:83-209`）。
  2. **非对角项**（消费 mcp 系数表）：读 `mcp.31` 每个 $I(ab)$ 标签组、`mcp.32+k` 每个 $R^k(abcd)$ 标签组，对标签计算一次径向积分，然后
$$\mathrm{EMT}(\mathrm{INDX}_i) \mathrel{+}= I(ab)\cdot \mathrm{COEFF}_i \quad\text{或}\quad R^k(abcd)\cdot \mathrm{COEFF}_i,\qquad i=1..\mathrm{NCONTR}$$
  （`setham.f90:223-342`）——正是 [[rangular90]] 排序输出的消费端：**每个积分只算一次径向值，与整组角系数做缩并**。
  3. **平均能量与平移**：$E_{\text{av}} = \frac{1}{N_{\text{CF}}}\sum_r H_{rr}$，对角元减去 $E_{\text{av}}$ 降低条件数（`matrix.f90:178-203`）。
  4. **分尺寸本征求解**：`NCF=1` 时直接取 $(\lambda,\mathbf c)=(0,1)$；`dvdfirst .OR. (NCF<=4000)` 为真时，`INIEST2` 将整个（或左上 $\min(NCF,4000)\times\min(NCF,4000)$）稀疏块展开为 packed 上三角矩阵并用 LAPACK `DSPEVX` 直接求所需本征对（`maneig.f90:119-120`）；仅当 $NCF>4000$ **且该块非首次对角化**（`dvdfirst=.FALSE.`）时，才跳过 `DSPEVX`，直接复用上一轮已收敛的本征向量/本征值作为 `GDVD` Davidson 的初始子空间求解完整稀疏块（`maneig.f90:121-128,134-171`；`lib/lib9290/iniest2.f90:44-82`）。也就是说，`DSPEVX` 子块初始化只发生在该块首次对角化时；$NCF>4000$ 并非直接触发该初始化的条件。
  5. **本征向量阻尼与正交化**：与上轮向量混合 $\mathbf{c} \leftarrow (1-\mu)\mathbf{c}_{\text{new}} + \mu\,\mathbf{c}_{\text{old}}$，重归一化（主分量取正），同块内向量化简正交（`matrix.f90:216-267`）。
  6. **写 `rmix.out`**（仅 EOL，unit 25）：每块依次写 `(JBLOCK,NCF,NEVBLK,2J+1,\pi)`、能级指标、`EAV` 与相对本征值数组 `EVAL`、本征向量（`matrix.f90:291-297`）。文件不直接存储 $E_{\text{av}}+\lambda_\alpha$；消费者必须组合二者。
- **公式**：
$$\sum_s H_{rs}\,c_{s\alpha}=(E_{\text{av}}^{(b)}+\lambda_\alpha)c_{r\alpha},\qquad H_{rs}=\sum_{ab}T_{rs}(ab)I(ab)+\sum_k\sum_{(ab)(cd)}V^k_{rs}(abcd)R^k(abcd)$$
- **公式-代码对应**：$H$ 装配即 `SETHAM`；$E_{\text{av}}$ 即 `matrix.f90:178-203`；本征问题由 `MANEIG` 选择 `DSPEVX/GDVD`；`matrix.f90:294` 分别写出 $E_{\text{av}}$ 与 $\lambda_\alpha$，而 `NEWCO` 在内存中以二者之和构造物理能量。

### 阶段 3：能级综合（`NEWCO`）

- **代码**：`newco.f90:3-131`；`dsubrs.f90`；`csfwgt.f90`。
- **输入**：本征对 $(\lambda_\alpha, \mathbf{c}_\alpha)$、权重 `WEIGHT`。
- **处理**：权重归一化 $w_\alpha = W_\alpha / \sum_\beta W_\beta$（$W_\alpha \in \{1,\ 2J_\alpha{+}1,\ \text{用户值}\}$，`newco.f90:61-77`）；广义占据数
$$\mathrm{UCF}(a) = \sum_{b}\sum_{r \in b} d_{rr}^{(b)}\, q_{a,r}, \qquad d_{rs}^{(b)} = \sum_{\alpha \in b} w_\alpha\, c_{r\alpha} c_{s\alpha}\ (\mathrm{DSUBRS})$$
即组态混合对角密度乘占据（`newco.f90:84-92`）；加权平均能量 $\bar E = \sum_\alpha w_\alpha (E_{\text{av}}^{(b(\alpha))} + \lambda_\alpha)$（`:97-110`）；`CSFWGT` 打印每个 ASF 的前五个 CSF 贡献。
- **输出**：$w_\alpha$、$\mathrm{UCF}(a)$、$\bar E$（SCF 收敛判据用）。$\mathrm{UCF}$ 是轨道方程中占据数的来源——EOL 特有：占据数变成分数的"广义"值。
- **公式-代码对应**：$d_{rr}$ 即 `DSUBRS(EOL,I,I,NB)`，$q_{a,r}$ 即 `IQ(J, I+NCFPAST(NB))`（`newco.f90:88`）。

### 阶段 4：轨道改进（`SETLAG` + `IMPROV`，核心阶段之二）

- **代码**：`scf.f90:160-170`；`setlag.f90`；`improv.f90:3-187`；势场装配 `cofpot.f90`（`SETCOF+YPOT+XPOT+LAGCON+DACON`）；`defcor.f90`；`solve.f90`；`dampck/dampor.f90`；`consis.f90`。
- **输入**：当前全部轨道、$\mathrm{UCF}$、Lagrange 乘子表。
- **处理**：
  1. **SETLAG**（每轮 SCF 一次）：确定乘子对列表——同 $\kappa$、至少一侧变分、且排除"双侧变分且双侧满壳"的对（`setlag.f90:85-100`），随后用当前轨道更新各乘子值（正交性约束的离殻项）。
  2. **IMPROV(J)**（对每个变分轨道）：
     - `COFPOT` 装配径向方程系数：`YPOT` 直接势（Pyper, Comput. Phys. Commun. 21 (1980) 211, Eq.(14)）、`XPOT` 交换势（Grant–Mayers–Pyper）、`LAGCON` Lagrange 乘子项、`DACON` 非对角 $I(ab)$ 贡献（各文件头注释）。
     - `DEFCOR` 差分格式的延迟修正。
     - `SOLVE`（Fischer Alg. 5.2/5.3）：打靶法解微扰 Dirac 径向方程——节点计数 + 能量修正迭代（与 [[rwfnestimate90]] 的 `SOLVH` 同构骨架，但势场是自洽的），失败时降级 `METHOD=2` 重试（`improv.f90:96-115`）。
     - `CONSIS` 在候选轨道归一化和阻尼之前，以旧轨道为参照计算（`consis.f90:34-40`）
$$
\mathrm{SCNSTY}(a)=\sqrt{\mathrm{UCF}(a)}\max_{1\le i\le M_a}\left(|P_a^{\mathrm{cand}}(r_i)-P_a^{\mathrm{old}}(r_i)|+|Q_a^{\mathrm{cand}}(r_i)-Q_a^{\mathrm{old}}(r_i)|\right).
$$
候选轨道随后按 $\|(P,Q)\|=1$ 归一化（`improv.f90:117-134`）。
     - 阻尼：未收敛（$\mathrm{SCNSTY}>\mathrm{ACCY}$）时令 $\mu_a=|\mathrm{ODAMP}(a)|$，`DAMPOR` 执行
$$
(P_a,Q_a)_{\mathrm{store}}=(1-\mu_a)(P_a,Q_a)_{\mathrm{cand}}+\mu_a(P_a,Q_a)_{\mathrm{old}},
$$
再归一化；否则令 $\mu_a=0$，整体采用候选轨道。负 `ODAMP` 表示固定 $|\mathrm{ODAMP}|$，非负值由 `DAMPCK` 根据相邻能量变化自适应调整（`dampck.f90:35-52`；`dampor.f90:55-117`）。
     - `ORTHST` 为真时，`ORTHY` 不只处理当前轨道，而是重建其整个同 $\kappa$ 组：固定轨道在前，随后为光谱变分轨道和关联变分轨道；若 `LSORT` 为真，前者按 $\mathrm{SCNSTY}/E^2$、后者按 `SCNSTY` 排序，再对全部非固定轨道做 Schmidt 正交化（`orthy.f90:78-226`）。
- **公式**（每轨道的 Euler–Lagrange 方程，结构性表述）：
$$\mathcal{F}_a\begin{pmatrix}P_a\\ Q_a\end{pmatrix} = \varepsilon_a \begin{pmatrix}P_a\\ Q_a\end{pmatrix} + \sum_{b\,:\,\kappa_b=\kappa_a} \varepsilon_{ab}\begin{pmatrix}P_b\\ Q_b\end{pmatrix}, \qquad \mathcal{F}_a = \mathcal{F}_a\bigl[\mathrm{UCF},\ V_{\text{dir}}(\{P,Q\}),\ V_{\text{exc}}(\{P,Q\})\bigr]$$
- **公式-代码对应**：$\varepsilon_{ab}$ 即 `lagr_C` 的乘子（`SETLAG/LAGCON`）；$\mathcal{F}_a$ 的三项即 `YPOT/XPOT/DACON`；解算即 `SOLVE`；阻尼混合在 `DAMPOR`。

### 阶段 5：收敛判定与输出（`SCF` 尾部）

- **代码**：`scf.f90:143-249`。
- **处理**：EOL 在循环前先执行一次 `MATRIX+NEWCO`。每轮 SCF 再执行 `SETLAG` → 依 `IORDER` 逐轨道 `IMPROV` → 最多 `NSIC` 次“`MAXARR`（取 $\mathrm{SCNSTY}$ 最大者 $K$）+ `IMPROV(K)`”。轨道判据为
$$
C_{\mathrm{orb}}=\left[\max_{a\in\mathcal V}\mathrm{SCNSTY}(a)\le\mathrm{ACCY}\right],
$$
随后非 `ORTHST` 时调用 `ORTHSC`，`ORBOUT` 写 `rwfn.out`；EOL 再执行 `MATRIX+NEWCO` 并检查
$$
C_E=\left[\left|\frac{\bar E_k-\bar E_{k-1}}{\bar E_k}\right|<10^{-3}\mathrm{ACCY}\right].
$$
源码以逻辑或停止：$C_{\mathrm{stop}}=C_{\mathrm{orb}}\lor C_E$（`scf.f90:176-228`），并不要求两个条件同时成立。达到条件或耗尽用户输入的 `NSCF` 后，`ENDSUM` 完成 `rmcdhf.sum`。
- **输出**：所有正常进入循环的路径写 `rwfn.out`（G92RWF 轨道）和 `rmcdhf.sum`；只有 EOL 路径创建并逐轮重写 `rmix.out`，其中能量保存为分离的 $E_{\mathrm{av}}$ 与相对值 $\lambda_\alpha$。`rmcdhf.log` 总是打开；非默认模式若启用调试还会创建默认名 `rscf92.dbg` 或用户指定的调试文件。

## 分支、迭代与状态

- **双层迭代结构**：外层 SCF 轮（$\le$ `NSCF`）× 内层轨道遍历 + 补充改进（`NSIC` 次）；EOL 时每轮 SCF 嵌套一次全块 CI 对角化。这正是 Fischer Alg. 5.1 的"improve each orbital in turn, then the least self-consistent ones"策略。
- **EOL/AL 分支**：`DIAG=.TRUE.` 时进入 AL，其余情况进入 EOL；`ICCUT` 不参与选择。EOL 执行阶段 2/3，AL 试图通过 `GETALD` 从 CSF 权重直接构造占据数，但当前 AL 路径存在后述未初始化状态，不能视为与 EOL 同等完整。
- **方法/阻尼状态**：`METHOD(a)`（积分方法 1/2/3，失败自动降级）、`ODAMP/CDAMP`（轨道/向量阻尼）、`NSIC` 可在迭代中自减（`improv.f90:147`）。
- **跨轮状态**：`EVEC`（上轮本征向量，`MATRIX` 里复制到 `CMVL` 用于阻尼）、`SCNSTY`、`IORDER`、Lagrange 乘子数组 `IECC/ECV`。
- **本征求解分支**：$NCF\le4000$ 的非平凡块，以及 $NCF>4000$ 但该块首次对角化（`dvdfirst=.TRUE.`）的块，均由 `INIEST2`（内部用 `DSPEVX`）直接求解或构造初始子空间；只有 $NCF>4000$ 且非首次对角化的块才跳过 `DSPEVX`、复用上一轮本征向量调用 `GDVD`。
- **每轮持久化**：`ORBOUT` 每轮写 `rwfn.out`；EOL 的 `MATRIX` 同时从头更新 `rmix.out`。这些文件可作为外部后续计算或重启输入，但消费者关系不由本程序内部保证。
- **调试状态**：非默认模式选择生成调试输出后，`SETDBG` 虽逐项询问，却在末尾把 `LDBPG(1:5)`、`LDBPR(1:30)`、`LDBPA(1:3)` 全部强制设为 `.TRUE.`；实际行为是启用这些范围内的全部调试输出。

## 最终输出构造

端到端复合：

$$
\bigl(\{P^{(0)},Q^{(0)}\},\ \mathbf{c}^{(0)}\bigr)\xrightarrow{G^k}\bigl(\{P^{(k)},Q^{(k)}\},\mathbf{c}^{(k)}\bigr)\to\texttt{rwfn.out}+\begin{cases}\texttt{rmix.out},&\mathrm{EOL},\\ \varnothing,&\mathrm{AL},\end{cases}
$$
$$
G = \underbrace{\mathrm{NEWCO} \circ \mathrm{MATRIX}}_{\text{CI: } H(\{P,Q\})\mathbf{c}_\alpha = E_\alpha\mathbf{c}_\alpha} \;\circ\; \underbrace{\prod_{a \in \text{varied}} \mathrm{IMPROV}_a}_{\mathcal{F}_a(\{P,Q\},\mathrm{UCF})\text{ 逐轨道解}},
$$

EOL 的循环在轨道判据或能量判据任一成立时停止。`rwfn.out` 保存当轮轨道；`rmix.out` 仅在 EOL 模式保存块结构、$E_{\mathrm{av}}$、相对本征值与本征向量。rci/hfs/rtransition 等程序对这些文件的使用属于外部 GRASP 工作流语义。

## 不确定性与未解析路径

1. **AL 路径初始化不完整**：`GETALD` 不像 `GETOLD` 那样初始化 `LFIX`、`NFIX`、`IORDER`，但 `SCF` 随后用这些量选择和索引待更新轨道；`GETALDWT` 的用户权重分支又以未初始化的局部变量 `NCMIN` 控制读入长度；此外 AL 不调用 `NEWCO`，但 `SCF` 仍无条件使用未赋值的 `WTAEV` 做能量收敛测试。故 AL 的实际结果在当前静态路径下未定义，不能仅以设计意图补全。
2. **SETLAG 乘子更新公式未全部化简**：已确认乘子对的选取规则及 `setlag.f90:147-277` 的四个更新分支，但未把其积分表达式化简为独立理论推导。
3. **`SOLVE/IN/OUT/NEWE` 内部**：已确认齐次、非齐次和变分方程的拼接、节点判断以及 `NEWE` 更新分支；底层差分方程系数未逐项重写。
4. **`FCO/GCO` 的角系数公式**：只确认其在即时对角项和 `SETCOF` 势系数中的调用、求和与归一化位置，未展开内部角动量代数。
5. **串行版边界**：MPI 变体具有行分片、归约和通信差异；本页只覆盖 `MYID=0,NPROCS=1` 的执行语义，`mpi_s.f90` 在此只是占位模块。

## Research Wiki 映射

本页是按用户要求存放在 `codes/grasp/` 下的代码分析工件（非标准 `outputs/` 位置）。与同目录另四页构成 GRASP2018 完整流水线链条：

- [[rcsfgenerate90]] 生成（或 [[rcsfinteract90]] 以 MR 精简）分块 `rcsf.inp`；
- [[rwfnestimate90]] 生成初始轨道 `rwfn.inp`；
- [[rangular90]] 生成角系数 `mcp.30..32+KMAXF`（其 LFORDR/ICCUT 头在此被读回）；
- 本页：rmcdhf 消费三者，MCDHF 自洽场迭代总是写 `rwfn.out`，并仅在 EOL 模式写 `rmix.out`；
- 典型外部工作流中，rci/rci_hf/rhfs/rtransition 等可消费收敛轨道或混合系数；该消费者关系不是由 `rmcdhf90` 自身实现的。
