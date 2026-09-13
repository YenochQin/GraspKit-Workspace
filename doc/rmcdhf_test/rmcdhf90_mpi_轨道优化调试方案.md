# `rmcdhf90_mpi` 轨道优化调试与改进方案

## 1. 目标与当前判断

目标是解释并修复以下现象：在 Ni-like/Ca-like 与 Cl I 的两组计算中，新增 LBL 轨道不优化（`_no_varied`）时相对能量更接近实验值且 $J$ 顺序正确；启用轨道优化后，能量变差，甚至出现精细结构倒序。计划针对
`/home/workstation2/AppFiles/GraspKit-Workspace/rmcdhf_test/src/appl/rmcdhf90_mpi`
进行只读基线、最小改动、逐级验证；源码修改应在独立副本或 git 分支完成。

当前优先级最高的可检验假设如下：

1. LBL 输入传给 `GETOLDmpi` 的 varied-orbital 列表不完整，尤其只选择了 $nl$ 而漏掉 $nl-$；源码会把全部轨道先设为 `LCORRE=.TRUE.`，只有第二次选择的“spectroscopic orbitals”才改为 `METHOD=1, NOINVT=.FALSE., ODAMP=0, LCORRE=.FALSE.`（`getoldmpi.f90:150–168`）。
2. 只优化相对论伙伴的一侧，会导致不同 $J$ 块获得不平衡的轨道松弛；报告中多组 `nl` 发生大幅收缩而 `nl-` 保持 Thomas–Fermi 形式，这是最强的关联证据，但仍需由实际输入与运行日志确认。
3. `ORTHY` 按“固定 → 光谱 → 关联”顺序对同 $\kappa$ 轨道 Schmidt 正交化；若伙伴缺失、顺序不稳定或新轨道径向形状近乎塌缩，正交化可能把误差传播到其他轨道。
4. `SOLVE` 失败时 `IMPROVmpi` 自动把 `METHOD` 改为 2 并重试；`DAMPOR` 的阻尼只由单轨道 `SCNSTY` 和 `ODAMP` 控制，而 SCF/EOL 在 `SCFmpi` 中以轨道判据或加权能量相对变化即可停止。这可能接受“总能量收敛、精细结构未收敛”的状态。

不要以总能量降低作为成功标准。对一对精细结构能级定义

$$
\Delta E_J=E(J_1)-E(J_2),\qquad
\delta E_J^{\mathrm{opt}}=\Delta E_J^{\mathrm{opt}}-\Delta E_J^{\mathrm{ref}},
$$

其中 `ref` 优先取 `_no_varied` 和实验/NIST 的可比相对间隔。轨道优化的验收条件必须同时包括 $\Delta E_J$ 的符号、误差、轨道稳定性和可重复收敛。

本文档是目标方案，不是已完成清单。截至 2026-08-30，P0 诊断主体、
B0–B6、strict SCF、阻尼矩阵和已选 MPI 对照已有实施或结果；B7/B8、
全矩阵重复仍未完成；Ni/Ca-like AS2 节点稳定性已通过专项初验，Ni I AS2 仍有节点变化；串行/MPI 逐轮对照不作为本方案验收项。具体状态以
[轨道优化已完成实施](./rmcdhf90_mpi_轨道优化已完成实施.md) 和
[轨道优化测试结果](./rmcdhf90_mpi_轨道优化测试结果.md) 为准；已实施代码与
本方案的逐项符合性核查见
[轨道优化代码核查](./rmcdhf90_mpi_轨道优化代码核查.md)。

## 2. 已有证据与需要保留的基线

- `wiki/codes/grasp/rmcdhf90.md` 已重建串行版的 MCDHF 流程：EOL 中先 `MATRIX+NEWCO`，随后每轮 `SETLAG → IMPROV(各变分轨道) → MAXARR/NSIC → ORBOUT → MATRIX+NEWCO`；MPI 版应逐项对照，而不能假定二者完全相同。
- Cl I 报告显示：所有 `_no_varied` 的 ${}^{2}P_{3/2}$ 都低于 ${}^{2}P_{1/2}$，间隔约 $923.81\to921.13\ \mathrm{cm}^{-1}$；优化序列始终倒序，间隔约 $-2479.51\to-1739.74\ \mathrm{cm}^{-1}$。优化额外降低 $J=1/2$ 能量约 $0.012$–$0.016\ E_h$。
- Ni/Ca-like 报告显示普通 LBL 在 AS2 出现 $^3F_J$ 倒序/塌缩，未优化新层的序列更稳；新增轨道与初始估计的重叠可降至 $<0.1$，说明这不是微小数值扰动。
- 这些报告只能证明“优化路径与错误谱相关”，不能单独证明是 `GETOLDmpi`、`ORTHY`、`SOLVE` 或输入包装器中的哪一处导致。
- [[nuclear-polarization-potential-neutral-atom-grasp]] 表明核极化势可以作为短程、球对称的一电子势进入 RMCDHF/RCI，并可通过芯层松弛间接改变其他轨道；但当前 Wiki 没有标准 GRASP2018 已实现该势的证据。其主要直接影响是 $s_{1/2}$ 和 $p_{1/2}$，不支持把它作为当前虚轨道单侧收缩和大幅 $J$ 倒序的首要解释。

每次实验都保存：完整 stdin、`rcsf.inp`/`rwfn.inp`、`mcp.*`、stdout/stderr、`rmcdhf.log`、每轮 `rwfn`/`rmix`、编译器与 git commit；禁止只比较最终 `rmcdhf.sum`。

## 3. 代码路径审计

### 3.1 构建可复现的调用图

入口是 `rscfmpivu.f90:26` 的 `PROGRAM RSCFmpiVU`，输入阶段在 `getscdmpi.f90:218` 调 `GETOLDmpi`，SCF 主循环位于 `scfmpi.f90:176–249`，轨道更新位于 `improvmpi.f90`，链接由 `Makefile` 中的 `rmcdhf_mpi`、`-ldvd90 -lmpiu90 -l9290 -lmod` 决定。先记录实际编译命令、MPI 进程数和运行时环境。

### 3.2 逐项打印并校验轨道状态

在不改变数值逻辑的诊断版本中增加可关闭的 trace（建议 `LDBPR` 新增一个专用开关），每个 SCF 宏迭代打印：

- `NW, NFIX, NSIC, NSCF, ORTHST, LSORT`；
- `IORDER(i), NP(i), NH(i), NAK(i), LFIX(i), LCORRE(i), METHOD(i), NOINVT(i), ODAMP(i)`；
- `SETLAGmpi` 建立的每个同 $\kappa$ Lagrange 乘子对；
- `IMPROVmpi` 的 `ED1/ED2, DNORM, SCNSTY, INV, JP, NNP, METHOD` 变化及是否发生 fallback；
- `ORTHY` 实际正交化的同 $\kappa$ 轨道顺序与每次投影范数。

用一个小脚本把 trace 还原成表格，检查：

$$
\mathcal V_k=\{i:LFIX_i=\mathrm{false}\},\quad
\mathcal S_k=\{i:LCORRE_i=\mathrm{false}\},\quad
\mathcal V_k\cap\mathcal S_k
$$

是否与 LBL 意图一致；对每个 $l>0$ 强制报告 $nl-$ 与 $nl$ 是否同时出现在 $\mathcal V_k$。

### 3.3 生产 MPI 路径重复性审计

串行 `rmcdhf90` 主要用于目标电子组态自身生成的 CSF；活性空间扩展后 CSF 数量很大，生产流程使用 `rmcdhf90_mpi`，不要求二者逐轮数值一致。仅对生产 `_mpi` 路径在 1、2、4 个进程下做重复性检查；如出现进程数相关差异，再单独审计 `NDA/DA` gather、去重与归约。

本机后续生产规模测试统一采用 **48 个 OpenMPI ranks × 每 rank 1 个线程**，即
`mpirun -n 48 rmcdhf_mpi`，并固定 `OMP_NUM_THREADS=1`、
`OPENBLAS_NUM_THREADS=1` 和 `GRASP_OMP_THREADS=1`。RMCDHF 的主要并行工作由
MPI 分配；OpenMP 仅可能由 FlexiBLAS/OpenBLAS 在部分线性代数阶段使用。
实测 `4 MPI ranks × 12 OpenMP/BLAS threads` 在部分阶段总 CPU 占用约为 8%，
不能作为生产资源配置。1/2/4 ranks 只用于进程数重复性诊断，不要求跑满节点，
不得据此沿用混合并行配置执行后续生产计算。

## 4. 最小诊断实验矩阵

所有实验从同一 AS1（因为 Cl I 在 AS1 已倒序）开始，固定 CSF、EOL 状态、权重、网格、核模型、ACCY、NSCF、NSIC、积分方法和编译器。

| 编号 | 变体 | 目的 | 关键观测 |
|---|---|---|---|
| B0 | 新层全部固定（`_no_varied`） | 基线 | $\Delta E_J$、轨道与 CI 可复现 |
| B1 | 只优化实际 `nl` | 重现当前失败 | 是否复现单侧收缩和错误 $J$ 序 |
| B2 | 只优化 `nl-` | 伙伴单侧反事实 | 观察倒序是否转移/消失 |
| B3 | `nl-` 与 `nl` 成对优化 | 检验伙伴平衡假设 | 间隔符号、差分能量、重叠 |
| B4 | 成对优化但 `ODAMP=-0.2,-0.5,-0.8` | 检验过冲/局部极小 | 每轮 $\Delta E_J$ 与 SCNSTY 轨迹 |
| B5 | 成对优化，`ORTHST=.false.`，仅循环末正交化 | 分离正交化影响 | 若恢复正确顺序，重点审计 `ORTHY` |
| B6 | 成对优化，固定 `METHOD=3`，禁止失败 fallback | 分离求解器分支 | 是否仍出现塌缩/倒序 |
| B7 | 同一最终轨道做独立 CI/RCI | 分离轨道与 CI 影响 | 基组差异是否足以产生 $\Delta E_J$ 变化 |
| B8 | 对称 EOL 权重与不同状态集合 | 检验状态加权偏置 | $J$ 依赖的额外能量降低 |

每个变体至少做 3 个重复（不同 MPI 进程数或同配置重复），并画出

$$
\Delta E_J^{(k)},\quad
\max_i\mathrm{SCNSTY}_i^{(k)},\quad
|S_i^{(k)}|,\quad
\langle r\rangle_i^{(k)},\quad
E_{J_1}^{(k)}-E_{J_1}^{(0)},
$$

随 SCF 轮次 $k$ 的曲线。重点记录 $|S|<0.1$、节点数变化、$\langle r\rangle$ 一个数量级收缩、以及 $J$ 间隔变号。

### 4.1 核极化势作为独立的后续实验轴

核极化势不得加入 B0–B8 的主诊断矩阵。主矩阵的目标是定位现有 Hamiltonian 下的轨道选择、求解、阻尼和正交化问题；此时加入新的物理势会使“优化算法修复”和“Hamiltonian 改变”无法区分。

只有主线修复通过第 7 节验收后，再运行下列独立矩阵：

| 编号 | 核极化势处理 | 轨道处理 | 目的 |
|---|---|---|---|
| NP0 | 关闭 | 使用已验收轨道 | 无核极化基线 |
| NP1 | 后处理一阶期望值 | 固定 NP0 轨道 | 判断符号和数量级是否值得继续 |
| NP2 | RCI Hamiltonian 中加入 | 固定 NP0 轨道、允许 CSF 重混合 | 分离 CI 重混合贡献 |
| NP3 | RMCDHF 与 RCI 均加入 | 重新优化轨道 | 测量完整自洽松弛贡献 |

定义

$$
\delta E_{\mathrm{relax}}
=\Delta E_{\mathrm{NP3}}-\Delta E_{\mathrm{NP2}},
$$

其中 NP2 与 NP3 必须使用相同的核参数、CSF 空间、EOL 权重和最终 RCI 选项。若 NP1 已远小于目标误差预算，则停止，不进入 NP2/NP3。

## 5. 建议的代码改进顺序

### P0：只加诊断，不改物理

1. 添加 `TRACE_ORB_OPT` 编译/运行开关，记录上述状态和每轮轨道摘要。
2. 对输入 varied list 做合法性检查：重复索引、固定轨道、未知轨道、同 $\kappa$ 伙伴缺失都给出明确警告；不要静默修正用户输入。
3. 在 `GETOLDmpi` 返回前输出解析后的 $\mathcal V_k$、$\mathcal S_k$ 和 `IORDER`；在 `SCFmpi` 收敛时同时打印轨道判据与能量判据，明确是哪个条件触发停止。

### P1：安全护栏（默认关闭，先做 A/B）

1. **伙伴成对策略**：提供 `PAIR_ORBITALS` 选项，将同 $n,l$ 的 $nl-$/$nl$ 作为一个不可拆分优化组；若用户只给一侧，默认报错或要求显式 `ALLOW_UNBALANCED`。
2. **渐进阻尼**：先通过默认关闭的实验开关对新关联层采用固定负 `ODAMP`，例如 $-0.5$；每轮限制径向变化，并在 $|S|<0.1$ 或 $\langle r\rangle$ 突变时回退到上轮轨道。在多体系验收前不将其设为生产默认值。
3. **收敛双门槛**：程序内部只有同时满足轨道变化和加权能量变化，且连续两轮成立时才接受；目标精细结构间隔的稳定性由外部测试驱动检查。不能把现有 `OR` 语义悄悄改掉，应先用新开关比较。

### P2：算法层改进（仅在 P0/P1 定位后）

- 将新关联轨道和 spectroscopic 轨道分组，使用成组阻尼/正交化，避免 `ORTHY` 在同 $\kappa$ 组内以不稳定顺序重塑轨道。
- 对 `SOLVE` 的节点数、`INV/NOINVT`、`METHOD` fallback 增加失败原因与候选轨道质量检查；禁止“求解成功”但节点数错误的候选进入 `DAMPOR`。
- 研究 EOL 目标函数是否需要状态均衡或 state-specific/MCDHF 分阶段优化；这属于物理模型改变，必须与纯代码修复分开评估。

### P3：可选的核极化势扩展（不属于当前缺陷修复）

在 P0–P2 完成并稳定后，可按 [[nuclear-polarization-potential-neutral-atom-grasp]] 增加

$$
H_{\mathrm{DC+NP}}=H_{\mathrm{DC}}+\sum_i V_{\mathrm{NP}}(r_i),
$$

其中

$$
V_{\mathrm{NP}}(r)=V_1(r)+V_2(r),
\qquad
\langle a|V_{\mathrm{NP}}|b\rangle
=\delta_{\kappa_a\kappa_b}
\int [P_aP_b+Q_aQ_b]V_{\mathrm{NP}}(r)\,dr.
$$

该扩展必须有独立开关，默认关闭；核极化关闭时必须重现 P0–P2 的全部 golden baseline。不能通过修改 Fermi 核半径近似该势，也不能只在 RMCDHF 中加入而在最终 RCI 中遗漏。

## 6. 代码实施过程

### 6.4 Ni I AS1→AS2 初始波函数节点诊断（已完成）

Job 559 已验证：AS2 读入 AS1 命名波函数后、第一次 `SOLVE` 之前，
`5d-`、`5d` 和 `6s` 的实际节点数分别为 3、3、6，而理论 `NNODEP`
分别为 2、2、5。节点异常因此已经存在于继承的波函数或其轨道映射中。
诊断必须使用 `COUNT(PF(:,J),MF(J),...)`；`MF` 本身是径向网格长度，不能
作为节点数。Job 557/558 的早期输出因误用 `MF` 或调用旧二进制而无效。

后续定位顺序固定为：

1. 对 AS1 最终 `.w` 文件中的对应轨道重新计数；
2. 对比 AS1 写出、AS2 读入后的轨道顺序及 `NP/NAK` 映射；
3. 只有确认继承文件本身节点正确后，才继续审计 AS2 的 `SOLVE/ORTHY`。

在完成上述 A/B 诊断前，不修改节点护栏判据，不以理论 `NNODEP` 直接替代
“旧节点与候选节点比较”的现有逻辑。

### 6.5 新增轨道估计方法分析与下一步

Job 564（Screened Hydrogenic）和 Job 565（Screened Hydrogenic custom-Z）
显示：`5p` 始终为 4 个节点、`6s` 始终为 6 个节点；`5d` 则在不同估计
方法间为 2 或 3 个节点。理论值分别为 `3、5、2`。因此不能把问题归结为
单一 `Z_eff` 或 AS1 波函数继承。

当前优先假设是新增轨道在径向网格外层存在截断边界符号翻转或低幅值振荡，
而 `COUNT` 对完整 `MF` 区间计数，将其识别为额外节点。下一项诊断必须对
同一初始 `PF` 在 `MTP`、`MTP-10`、`MTP-20` 等截断位置分别计数，并记录
尾部振幅；只有确认截断敏感性后，才考虑修改节点判据或 `rwfnestimate`
输出，不能先放宽节点护栏，也不能修改 `SOLVE`/`ORTHY`。

实施必须按下面的提交顺序进行。每个提交只解决一个问题；前一阶段未通过回归时，不进入下一阶段。所有新增行为默认关闭，未设置新开关时必须保持当前 `rmcdhf_mpi` 的数值结果和交互输入顺序不变。

### 6.1 实施前准备：冻结基线

1. 在 `${grasp_code_path}` 中记录当前分支、commit、dirty 状态、编译器、MPI 实现和链接库版本。若该目录由 git 管理，创建独立分支，例如 `codex/rmcdhf-orbopt-diagnostics`；若不由 git 管理，先复制为一个独立开发树，不直接覆盖生产程序。
2. 使用现有构建流程编译，不先调整优化级别或数值库：

   ```shell
   make -C "${grasp_code_path}/appl/rmcdhf90_mpi"
   ```

3. 为 Cl I AS1 和 Ni/Ca-like AS2 各冻结一套最小输入。每套输入包含 `isodata`、`rcsf.inp`、`rwfn.inp`、`mcp.*`、完整 stdin 和预期输出摘要。
4. 分别运行 `_no_varied` 和当前 optimized 基线，保存可执行文件校验值、退出码、stdout、`rmcdhf.log`、`rmcdhf.sum`、`rwfn.out`、`rmix.out`。这些结果成为后续的 golden baseline。
5. 在任何物理改动前，先确认相同可执行文件重复运行结果一致；MPI 1、2、4 进程的差异另外记录，不能混入轨道算法改动。

建议在 `${grasp_code_path}/test/rmcdhf_orbopt/` 建立测试夹具，只存可公开且尺寸足够小的输入。大型计算数据继续放在外部实验目录，测试目录只保存路径清单与结果摘要。

### 6.2 提交 1：新增独立控制模块，不改变数值流程

新增 `${grasp_code_path}/appl/rmcdhf90_mpi/orbopt_control.f90`，定义模块 `ORBOPT_CONTROL_C`。第一版只包含：

```fortran
LOGICAL :: TRACE_ORBOPT = .FALSE.
LOGICAL :: WARN_UNBALANCED_PAIR = .TRUE.
LOGICAL :: REQUIRE_BALANCED_PAIR = .FALSE.
LOGICAL :: ENABLE_ORBITAL_GUARD = .FALSE.
LOGICAL :: STRICT_SCF_CONVERGENCE = .FALSE.
INTEGER :: ORBOPT_TRACE_UNIT
```

实现 `INIT_ORBOPT_CONTROL`：由 rank 0 读取环境变量或一个单独的可选配置文件，再通过 `MPI_Bcast` 同步到全部进程。建议的环境变量为 `GRASP_TRACE_ORBOPT`、`GRASP_REQUIRE_BALANCED_PAIR`、`GRASP_ORBITAL_GUARD` 和 `GRASP_STRICT_SCF`。不得向现有 stdin 中插入新问题，否则历史输入脚本会错位。

具体接线：

1. 在 `Makefile` 的 `OBJS` 中把 `orbopt_control.o` 放在使用它的目标文件之前。
2. 在 `rscfmpivu.f90` 中，于 `startmpi2` 完成、`myid/nprocs` 已确定后调用 `INIT_ORBOPT_CONTROL`。
3. 不复用 `LDBPR(1:30)`：这些编号已被 `solve.f90`、`scfmpi.f90`、`xpot.f90` 和 `ypot.f90` 使用。
4. 默认配置下重新运行两个 golden baseline。要求能量、轨道数组和输出文件在既定浮点容差内不变；否则撤销本提交并定位初始化副作用。

### 6.3 提交 2：记录真实轨道选择与优化状态

新增 `orbopt_trace.f90`，提供 `TRACE_ORBITAL_SELECTION`、`TRACE_SCF_BEGIN`、`TRACE_ORBITAL_UPDATE`、`TRACE_SCF_END` 四个过程。输出采用稳定的 CSV 或 JSONL 字段，不解析人类可读的格式化日志。

修改点如下：

| 文件 | 插入位置 | 记录内容 |
|---|---|---|
| `getoldmpi.f90` | 两次 `GETRSL` 结果广播并完成 `LFIX/LCORRE` 设置后 | 轨道标签、索引、`NP/NH/NAK`、`LFIX`、`LCORRE`、`METHOD`、`NOINVT`、`ODAMP`、`IORDER` |
| `scfmpi.f90` | 每个 `NIT` 开始处 | `NIT, NSCF, NSIC, ACCY, ORTHST, LSORT, WTAEV0` |
| `improvmpi.f90` | `SOLVE` 前、`SOLVE` 后、`DAMPOR` 后 | `J, E_old, E_candidate, METHOD`、fallback、`DNORM`、`SCNSTY`、`ODAMPJ, INV, JP, NNP` |
| `orthy.f90` | `KINDX` 排序完成后 | 当前 $\kappa$ 组的固定/光谱/关联分组与实际 Schmidt 顺序 |
| `scfmpi.f90` | 收敛判断处 | 轨道判据、能量判据、最终触发停止的判据 |

主 trace 只由 rank 0 写入 `orbopt_trace.csv`。为审计 MPI 一致性，每个 rank 另计算关键数组的摘要或校验值；只有摘要不一致时才写 `orbopt_trace.rankNNN.csv`，避免多个进程竞争同一文件。

本提交的验收是“打开 trace 只增加日志，不改变结果”。随后用日志回答两个事实问题：当前 varied list 是否真的只包含 `nl`；发生异常收缩的轨道是否经历了 `METHOD=2` fallback、异常节点计数或特定的 `ORTHY` 顺序。

### 6.4 提交 3：实现相对论伙伴完整性检查

在 `getoldmpi.f90` 完成 varied list 解析后调用新过程 `CHECK_RELATIVISTIC_PAIRS`。配对不能靠字符串是否带 `-` 判断，应使用量子数：对轨道 $i$ 由 `NAK(i)=\kappa_i` 得到

$$
l_i=\begin{cases}
\kappa_i, & \kappa_i>0,\\
-\kappa_i-1, & \kappa_i<0,
\end{cases}
$$

具有相同 $n$、相同 $l>0$ 且 $\kappa=l$ 与 $\kappa=-(l+1)$ 的两个轨道组成一对。检查的是

$$
V_i=\neg LFIX(i),\qquad V_i\oplus V_j,
$$

即一侧可变而另一侧固定的异或情形。

实施行为分两级：

- 默认只警告，输出缺失伙伴和解析后的 varied list，保证兼容旧任务；
- `GRASP_REQUIRE_BALANCED_PAIR=1` 时，在进入 `SCFmpi` 前终止并给出可修正的轨道列表。

这一提交只保证“选择完整”，不改变 `SCFmpi` 逐轨道更新的算法，也不能称为两个伙伴的同时联立优化。使用 B0–B3 验证：B1/B2 应产生警告或在严格模式失败，B3 应通过检查。

### 6.5 提交 4：增加候选轨道质量度量，仍不拒绝更新

新增 `orbopt_metrics.f90`。在 `improvmpi.f90` 中，候选轨道完成归一化后、调用 `DAMPOR` 前，比较候选数组 `P/Q` 与当前已存轨道 `PF(:,J)/QF(:,J)`，计算：

$$
S_J=\int\left[P_J^{\mathrm{cand}}P_J^{\mathrm{old}}+
Q_J^{\mathrm{cand}}Q_J^{\mathrm{old}}\right]dr,
$$

以及候选范数、旧/新节点数、`MF/MTP0`、能量变化、`SCNSTY` 和平均半径 $\langle r\rangle$。积分必须复用 GRASP 的网格权重与 `QUAD/RINT` 约定；先用一个未改变轨道验证 $S_J\approx1$，再与外部 `rwfn` 分析脚本交叉验证。没有完成该交叉验证前，不使用 $S_J$ 或 $\langle r\rangle$ 控制求解流程。

外部新增 `${grasp_code_path}/test/rmcdhf_orbopt/compare_rmcdhf.py`，负责从保存的每轮输出计算 $\Delta E_J$、$\langle r\rangle$、节点数和径向重叠。运行时 trace 与外部脚本必须对同一轨道给出一致趋势。

### 6.6 提交 5：加入可回退的轨道护栏

只有提交 4 证明“异常轨道指标能够稳定区分失败与正常更新”后，才实现 `GRASP_ORBITAL_GUARD=1`。第一版阈值不要写死为最终物理常数，而应由配置读取并写入日志，例如 `MIN_ORBITAL_OVERLAP`、`MAX_RADIUS_RATIO` 和 `MAX_REJECTS_PER_ORBITAL`。

在 `improvmpi.f90` 中的控制流程明确为：

1. `SOLVE` 前保存 `E_old=E(J)`、`MF_old`、`PZ_old`；当前已接受轨道仍保存在 `PF/QF`。
2. `SOLVE` 得到并归一化候选轨道后计算质量指标。
3. 指标正常：保持原路径，调用 `DAMPCK → DAMPOR → ORTHY`。
4. 指标异常：不调用 `DAMPOR`，不调用 `ORTHY`，恢复 `E(J)=E_old`、`MF/PZ` 等被 `SOLVE` 改写的标量；`PF/QF` 本来没有被接受，因此保持旧轨道。
5. 将该轨道标为未收敛，增加拒绝计数，并尝试更强固定阻尼或替代 `METHOD`。超过最大拒绝次数时明确停止，不能让 SCF 把“反复拒绝、轨道未更新”误判为收敛。

必须为“接受”“拒绝后成功”“连续拒绝后终止”各准备一个小测试。B4 用于选择阻尼，不允许仅凭一次 Cl I 结果确定默认值。

### 6.7 提交 6：拆分并收紧收敛判据

在 `scfmpi.f90` 中将当前混合变量 `CONVG` 拆成：

```fortran
CONVG_ORBITAL
CONVG_ENERGY
CONVG_LEGACY
CONVG_STRICT
```

对应关系为

$$
C_{\mathrm{legacy}}=C_{\mathrm{orb}}\lor C_E,
\qquad
C_{\mathrm{strict}}=C_{\mathrm{orb}}\land C_E.
$$

默认仍使用 legacy；`GRASP_STRICT_SCF=1` 时要求 strict 条件连续满足两轮才退出。程序内部暂不直接以特定 $\Delta E_J$ 收敛，因为目标能级对随计算任务变化；目标间隔稳定性由外部测试驱动检查。日志必须输出每个布尔量和连续满足次数。

回归要求：默认模式重现 golden baseline；strict 模式允许多做 SCF 轮次，但不得因为 `WTAEV0=0` 的首轮状态错误触发能量收敛。

### 6.8 提交 7：证据驱动的算法修改

前六个提交完成后，根据实验结果只选择一个分支实施：

| 证据 | 下一项改动 | 涉及文件 |
|---|---|---|
| varied list 确实漏掉伙伴，B3 恢复正确结果 | 将严格伙伴检查用于生产输入；暂不改求解器 | `getoldmpi.f90`, `orbopt_control.f90` |
| 选择完整但 `ORTHY` 后才出现异常 | 增加可选的稳定同 $\kappa$ 顺序，比较即时与轮末正交化 | `orthy.f90`, `improvmpi.f90`, `scfmpi.f90` |
| 异常总与 `METHOD=2` fallback/节点错误同现 | 增加 strict fallback 和节点验收；失败时保留旧轨道 | `solve.f90`, `improvmpi.f90` |
| 串行正确、MPI 多进程错误 | 单独修复 `NDA/DA` gather、去重或归约，先恢复进程数不变性 | `improvmpi.f90`, `cofpotmpi.f90` |
| 所有数值路径一致，但 EOL 权重改变结论 | 转入状态权重/目标函数实验，不把它包装成软件缺陷 | `getoldwt.f90`, `newcompi.f90` 及输入方案 |

真正的“伙伴联立更新”需要同时保存两条候选轨道、成组正交化并共同接受或回退，属于新的优化算法，不应与输入成对检查放在同一个提交。只有 B3 仍失败且证据明确指向逐轨道更新顺序时，才单独设计这一改动。

### 6.9 自动化回归与提交门槛

新增 `test/rmcdhf_orbopt/run_matrix.sh` 或等价驱动，按以下顺序运行：

1. 编译诊断版；
2. 运行 Cl I AS1 的 B0–B6；
3. 运行 Ni/Ca-like AS2 的关键子集 B0、B1、B3、B4；
4. 对通过的候选运行 MPI 1、2、4 进程；
5. 用 `compare_rmcdhf.py` 生成统一 CSV 和 Markdown 汇总；
6. 检查默认关闭所有开关时是否重现 golden baseline。

每个提交必须附带：修改文件、开关值、输入摘要、运行命令、退出码、能量差、$\Delta E_J$、轨道指标、MPI 一致性和结论。若某阶段失败，保留日志并停止；不要同时调整伙伴选择、阻尼、正交化和 EOL 权重，否则无法归因。

### 6.10 主线稳定后的核极化势实施

此阶段是独立功能分支，建议分支名为 `codex/grasp-nuclear-polarization`，不得与轨道优化修复提交混合。源码静态追踪给出的接入链为：

```text
getscdmpi.f90: CALL NUCPOT
  -> lib/lib9290/nucpot.f90: 生成 ZZ(r) = -r V_nuc(r)
  -> ypot.f90: YP(:N) = ZZ(:N) / NPROCS，进入径向 SCF 方程
  -> lib/lib9290/rinti.f90: MODE=0 时加入核势一电子积分
  -> setham.f90: RINTI(a,b,0) 进入 CI Hamiltonian
```

因此实现应分四个提交：

1. **NP 提交 1：参数与势函数。** 新增共享模块，例如 `NUCPOL_C`，保存 `ENABLE_NP`、E1/E2 开关、$\alpha_0^{EL}$、$b$、$\tilde b$ 和单位信息。输入参数必须显式记录同位素、质量数和来源；所有以 $\mathrm{fm}$ 给出的长度在进入 GRASP 网格前换算为 $a_0$。
2. **NP 提交 2：只生成势并验证。** 计算 $V_1(r)$、$V_2(r)$ 和

   $$
   ZZ_{\mathrm{NP}}(r)=-rV_{\mathrm{NP}}(r),
   \qquad
   ZZ_{\mathrm{total}}=ZZ_{\mathrm{nuc}}+ZZ_{\mathrm{NP}}.
   $$

   先只输出网格表，不进入 SCF。检查符号、$r\rightarrow0$ 有限性、远区 $r^{-4}/r^{-6}$ 渐近行为和核附近网格收敛。E1 与 E2 分开验证。
3. **NP 提交 3：固定轨道/RCI 路径。** 先实现同一 $\kappa$ 的一电子矩阵元或后处理期望值，完成 NP1/NP2。使用类氢或其他有参考数据的体系验证 $1s$、$2s$、$2p_{1/2}$ 能移；当前 Wiki 尚无已经验证的中性多电子 GRASP 算例，所以不能直接以 Cl I 的谱改善作为正确性证明。
4. **NP 提交 4：自洽 RMCDHF 路径。** 在 `CALL NUCPOT` 之后、任何 `YPOT/RINTI` 消费 `ZZ` 之前组合 `ZZ_{\mathrm{total}}`，再运行 NP3。最终 RCI 必须使用同一势和同一参数。分别报告直接固定轨道修正、CSF 重混合和轨道松弛：

   $$
   \Delta E_{\mathrm{NP}}^{\mathrm{total}}
   =\Delta E_{\mathrm{direct}}
   +\Delta E_{\mathrm{CI-mix}}
   +\Delta E_{\mathrm{relax}}.
   $$

不要直接永久修改全局 `lib/lib9290/nucpot.f90` 的默认行为。原型阶段应由独立开关在调用 `NUCPOT` 后叠加势；确认 `rmcdhf`、`rci` 和其他使用 `lib9290` 的程序均需要该能力后，再决定是否下沉为共享库接口。

核极化扩展的通过条件是：关闭开关时结果与基线一致；E1/E2 分项可复现；固定轨道和自洽结果差异可解释；核网格、活动空间和参数不确定度均有收敛记录。它是否改善实验能级只能作为物理结果，不能替代这些实现验证。

## 7. 验收标准与停止规则

修复候选只有在以下条件全部满足时才可进入更大 AS：

- Cl I AS1–AS5 保持正确的 ${}^{2}P_{3/2}<{}^{2}P_{1/2}$ 顺序，且不依赖 MPI 进程数；
- Ni/Ca-like AS2 不再出现 $^3F_J$ 倒序或异常塌缩；
- 变分能量仍下降，但 $J$ 间的额外降低不再达到足以翻转 $\Delta E_J$ 的量级；
- 新轨道与初始估计的重叠、节点数和 $\langle r\rangle$ 变化有可解释记录，异常会触发回退而非静默接受；
- 串行与 MPI 在数值容差内一致，运行可由保存的输入和 commit 完全复现。

若只能恢复 $J$ 顺序而不能改善实验误差，应保留该版本作为“稳定基线”，不要宣称轨道波函数更准确；后续应进入 CI/RCI、Breit/QED、CSF 平衡或 EOL 权重的物理验证。

## 8. 预期交付物

1. `data/rmcdhf_test_data/results/<run>/`：每个实验的输入快照、日志和轨道质量 CSV；
2. `test/rmcdhf_orbopt/compare_rmcdhf.py`：计算 $\Delta E_J$、能量差、径向重叠、节点数、$\langle r\rangle$ 并生成图表；
3. 一份“输入选择错误 / MPI 差异 / 正交化问题 / 求解器问题 / 物理模型偏置”的证据表；
4. 仅在证据支持时提交 P1/P2 代码补丁，并附逐项回归结果。

## 9. 相关资料

- Wiki 代码分析：`rmcdhf90.md`
- Ni/Ca-like 计算分析：`cal-report-analysis-cal-report-analysis-ni-ca-like-2026-08-19.md`
- Cl I LBL 分析：`cal-report-analysis-cl-i-lbl-orbital-optimization-2026-08-18.md`
- 核极化势与 GRASP：`nuclear-polarization-potential-neutral-atom-grasp`
- MPI 源码：`/home/workstation2/AppFiles/GraspKit-Workspace/rmcdhf_test/src/appl/rmcdhf90_mpi`
### Job 567：`COUNT` 阈值敏感性

`THRESH=0.01` 与默认 `0.05` 的节点计数一致，而 `THRESH=0.20` 已开始
滤掉真实节点。结合 `MTP/MTP-10/MTP-20` 计数完全一致，当前问题不是
外层网格截断。默认阈值保持不变，后续仅在 `0.05–0.20` 区间细分测试，
并记录节点零点位置和尾部振幅。
### Job 574：统一 `THRESH` 判据不可行

高阈值边界测试显示，`THRESH=0.30` 虽能修正 `5d`，却会把 `5p`、`6s`
的真实节点过滤掉；`THRESH≥0.40` 更会使多个轨道接近零节点。因此放弃
继续搜索统一全局阈值，默认 `THRESH=0.05` 保持不变。后续工作必须采用
轨道相关的极值/节点判定，或回到 `rwfnestimate` 修正初始径向函数；不得
以高阈值绕过节点护栏。
### Job 575：`ED1` 状态初始化修复后的结果

对比串行 `IMPROV` 发现 MPI 版缺少 `ED1=PED(J)`，已补回并完成回归。修复
后 AS2 仍在节点护栏处终止，说明该差异虽是必须修复的 MPI 兼容性问题，
却不能单独解释 Ni I AS2 的轨道失稳。后续分析仍需关注 `SOLVE` 候选、
阻尼接受路径以及 EOL 轨道耦合，不得因该修复通过编译就宣称根因已解决。
## 现有测试的取舍与主线调整

不优化轨道时，各活性空间均得到正确能级顺序和谱项；AS1 `.w` 继承正常；
逐个优化 AS2 新增轨道仍会复现异常。这三项证据把问题范围限定为新增
轨道进入 `IMPROVmpi` 后的通用更新流程。

节点计数、`MTP` 截断和 `THRESH` 矩阵已排除外层截断及统一阈值修复路径，
保留为诊断证据，不再继续扩展阈值搜索。Thomas–Fermi、Screened Hydrogenic
和 custom-Z 初猜比较也不作为主线，因为不优化路径本身已经正确。

已修复的 `ED1=PED(J)` 遗漏属于确定的 MPI 状态 bug，但修复后的 Job 575
仍失败，必须继续审计 LBL 的 MPI 优化状态传递。下一步按单变量顺序检查
`ED1/ED2/PED`、`DAMPCK/DAMPOR`、`P/Q` 与 `PF/QF` 交换、`SETLAGmpi`、
MPI `DA/NDA` 合并以及 `ORTHY` 调用，定位第一次异常后再实施修复。

### Jobs 579–582 后的诊断结论

`NDCOF=i_last` 修复已在 46-rank Ni/Ca-like AS2（Job 579）和 Cl I
AS2–AS5（Job 580）通过实际计算验证：作业均正常退出，J 顺序正确，且
没有 rank 间一致性异常。Ni I 的对照（Jobs 581、582）进一步将问题定位
到候选轨道：启用节点护栏时 `5d-` 候选由 3 个节点变为 2 个并达到拒绝
上限；仅关闭护栏后 AS2 可收敛，但 `5p-`、`5p`、`6s`、`5d-`、`5d`
均出现节点减少。下一轮实验应保持阻尼和 MPI 配置不变，审计
`SOLVE` 输出到 `DAMPOR` 输入之间的初始径向函数、节点和半径变化，不能
用放宽全局节点阈值作为修复。

### Jobs 586–588 后的节点判据修正

Jobs 586、587 与 Jobs 581、582 的 AS2 trace 逐字节一致。对 Job 587 最终
波函数独立计数后，`5p-/5p`、`5d-/5d`、`6s` 均回到理论 `NNODEP`。
因此下一项单因素代码实验不是关闭节点检查，而是把判据从“候选节点必须
等于旧轨道节点”改为“候选节点必须等于 `NNODEP`”。这允许多余初始节点
被修正，同时阻止正确的 `5p` 从 3 个节点变为 2 个。

该判据已由提交 `b47ac87` 实现并通过 Release 构建。对应 46-rank 独立作业
为 `test/rmcdhf_orbopt/slurm/run_ni_expected_nodes_46.sbatch`；验收时必须核对
AS1/AS2 退出码、最终节点摘要、$^3F_J$ 顺序及相对 NIST 的两个间隔。

Job 588 再次确认 `GRASP_DEFER_ORTHY=1` 会导致径向函数塌缩；该方向停止。
后续数值实验仅使用 LBL 的 MPI 程序，串行源码只作静态参考，不再安排
串行/MPI 数值对照。

### Job 589 后的阶段审计

Job 589 在 46-rank Ni I AS1→AS2 上验证 `b47ac87` 的理论节点判据。AS1
完成；AS2 的 `6s` 在第 1、2、3 次尝试中分别出现原始候选 3、3、4 个
节点，而旧轨道为 6、理论 `NNODEP` 为 5，连续拒绝后停止。Job 587 的
对应 trace 显示，同样的原始候选经 `DAMPOR` 后可以保持 6 个节点，随后
在第 4 轮阻尼后转为 5 个节点。这说明候选应允许“向理论节点单向修正”，
不能要求原始或阻尼后的每一次中间结果立即等于理论值。

Job 589 还揭示了一个独立的参数方向问题：`DAMPOR` 交换数组后，`P/Q`
是旧轨道、`PF/QF` 才是阻尼后的候选；当前阻尼后质量检查没有交换
`NODES_OLD/NODES_CANDIDATE`（以及对应半径），所以检查对象方向反了。
下一项单因素修改必须同时满足：

1. 原始 `SOLVE` 阶段只作重叠和半径的初筛；节点不要求立即等于 `NNODEP`。
2. 阻尼后检查使用正确的新旧方向。
3. 节点只允许不增加 `abs(nodes-NNODEP)`；从错误的多节点或少节点状态向
   理论值靠近可以接受，离理论值更远必须回退。
4. 其余 `ODAMP=-0.5`、逐轨道 `ORTHY`、EOL 权重、46 MPI rank 和输入不变。

该路径由 `GRASP_NODE_GUARD_PROGRESS=1` 显式开启，默认仍使用旧的精确
`NNODEP` 判据或完全关闭护栏，便于将本次实验与 Job 589 逐项比较。

Job 589 的 `rmcdhf.exitcode=137` 来自 runner 在检测到重复 `ERROR STOP`
后终止 `srun` 进程组，不作为 OOM 或算法数值结果解释。完成上述代码修改后，
先重跑 Ni I AS1→AS2，再按最终节点、谱项、$J$ 顺序和 NIST 间隔验收。

### Job 590 验收结果：节点进度护栏可运行但未解决物理偏差

Job 590 按上述单因素方案开启 `GRASP_NODE_GUARD_PROGRESS=1`，在 46-rank
LBL MPI 流程中完成 Ni I AS1→AS2。AS1、AS2 分别为 29、33 轮，退出码均为
0，且没有 `METHOD=2` fallback。AS2 的最终节点为 `5p-/5p=3/3`、
`5d-/5d=2/2`、`6s=5`，与理论 `NNODEP` 一致；仅在第 3 轮对 `5p-` 的
原始 3→2 候选执行过一次 `nodes_progress` 拒绝。`6s` 从初始 6 个节点在
阻尼阶段保持 6，随后恢复到理论 5 个节点，说明“节点偏离理论值不增加”
允许了合法的渐进修正。

最终顺序为 $^3F_4<{}^3F_3<{}^3F_2$，但两个间隔仍为 1375.88 和
916.11 cm$^{-1}$，相对 NIST 的误差约为 +43.72 和 +31.72 cm$^{-1}$。因此
该开关可以作为诊断和稳定基线保留，不能把它当作已经修复 Ni I 精细结构
偏差的生产补丁。下一步应继续审计 `SOLVE → DAMPOR → ORTHY → MATRIXmpi`
中的轨道/能量状态传递，并用 Ni/Ca-like、Cl I 和补齐输入后的 Fe I 做回归。

结果目录为 `data/rmcdhf_test_data/results/ni-as1-as2-chain-590/`，日志为
`data/rmcdhf_test_data/log/590_rmcdhf-ni-node-progress.log`。

### Jobs 591–604：完成状态分类与下一步

本批作业都已经结束；RMCDHF 作业使用 46 个 rank，B7 固定轨道 RCI 使用
1 个 rank。验收时必须区分四种情况：

1. **正常完成**：Job 596、597 的 Ni I AS1/AS2，以及 Job 601 的 Ni I B8
   full。它们均为退出码 0，节点和谱序可直接用于回归。
2. **预期护栏失败**：Job 593 的 legacy exact-node AS2、Job 592/595 的 Cl I
   AS3。失败发生在节点判据上，不能当作求解器崩溃；progress 判据并没有让
   Cl I 的 AS3 通过，因此暂时不能宣称 Cl I AS1–AS5 链完成。
3. **runner 时间超时**：Job 591、594、603、604 的 `rmcdhf.exitcode=124`。
   trace 在第 11 轮附近仍正常推进，旧脚本的 30 min 预算小于已有完整
   Ni/Ca-like 46-rank 记录的 2687 s。应先用已修改为 60 min 的脚本重跑，
   再评价 node-progress 或 B8 ASF 选择。
4. **后处理比较器失败**：Job 602 的 RMCDHF 本体 18 轮、退出码 0，目标态
   顺序正确；wrapper 因 full/target level 集合不同而失败。比较器应允许
   level 集合差异，不能据此回退代码。

Job 598–600 的 B7 固定轨道 RCI 已得到可读的 `b7_rci.csv`。Ni I 的两个
精细结构间隔为 1297.15、2160.19 cm⁻¹，Ni/Ca-like 为 1702.72、3746.95
cm⁻¹，Cl I 为 896.13 cm⁻¹。它们与 RMCDHF 的差异说明 CI/RCI 和状态选择对
间隔有体系依赖性；B7 不能用来替代 MPI LBL 轨道优化的回归验收。原 wrapper
的 `-gj` 参数错误已经从三个 sbatch 脚本中去掉。

当前后续顺序固定为：

1. 依次重跑 60 min 的 `default-off-nica`、`node-progress-nica`、`b8-nica-full`
   和 `b8-nica-target`，分别记录是否收敛、最终节点、$J$ 顺序和 NIST 间隔；
2. 可选地重跑三个修正后的 B7 脚本，确认自动化 CSV 生成与手动生成结果一致；
3. 使用 Job 597 的 `rwfn.out.iter*` 对 AS2 首次异常轮次做
   `SOLVE→DAMPOR→ORTHY` 数组对照，重点检查 `P/Q`、`PF/QF`、`ED1/ED2/PED`
   和 MPI `DA/NDA` 汇总；
4. 保留 Job 590/596 作为 46-rank node-progress 稳定基线，不安排串行/MPI
   数值对照；
5. 补齐 Fe I 输入后再做跨体系回归，当前不能以 Ni I、Ni/Ca-like、Cl I 的
   结果宣称四体系验证完成。

### QDIF 修复后的首个回归

静态比较确认 `rmcdhf90_mpi/setlagmpi.f90` 曾删除同时变分轨道的
`QDIF > 0.1` 分支，导致广义占据数相差很大的轨道对也使用 `OBQSUM` 公式。
该分支已经按串行版及 `rmcdhf90_mem_mpi` 恢复。为保持单因素顺序，已构建
不含后续 `MPI_Gatherv` 和稀疏索引修改的专用 `build-qdif/bin/rmcdhf_mpi`。由于
Jobs 590、596、597 在该构建之前运行，它们只能作为旧 MPI 基线。

首个单因素作业是
`test/rmcdhf_orbopt/slurm/run_qdif_fix_ni_46.sbatch`：Ni I balanced，
AS1→AS2，46 个 MPI rank，`ODAMP=-0.5`，node-progress 护栏和其余输入保持
不变。只有该作业完成并与 Job 590/596 对比后，才能判断 QDIF 修复是否改变
LBL 轨道优化；在此之前不应扩展到其他体系或宣称精细结构偏差已经修复。

后续两个 Ni I 单因素脚本已经分别绑定到独立二进制：

1. `run_gatherv_fix_ni_46.sbatch` 使用 `build-gatherv/bin`，只叠加
   `MPI_Gatherv` 系数汇总修复；
2. `run_sparse_index_fix_ni_46.sbatch` 使用当前 `build/bin`，在前两项基础上
   叠加 `SPICMVmpi/INIESTmpi` 稀疏列索引修复。

它们只能按上述顺序提交，且每一步都要与同一 Ni I AS1→AS2 输入的前一步比较
`rmcdhf.exitcode`、`orbopt_trace.csv`、每轮波函数、最终节点、谱序和 NIST 间隔。

为纳入外部验证目录中的 Fe I 成对数据，`run_data_case.sh` 现在支持可选的
`GRASP_TEST_DATA_ROOT`，默认仍读取工作区 `data/rmcdhf_test_data/inputs`。
新增的 `run_qdif_fix_fe_vv3_46.sbatch` 从
`/home/workstation2/nvmesdd2T/rmcdhf_test_cal/Fe_I/e1_vv3` 读取优化输入，并
从对应的 `_NV` 目录读取 `isodata`，不会复制 Fe I 的数 GB CSF 文件；计算结果
仍写入工作区 `data/rmcdhf_test_data/results`。该脚本只运行 Fe I AS1→AS2，
应在 Ni I QDIF 回归完成后提交，以保持单因素顺序。

### 下一项待验证：`IMPROVmpi` 的 `NDA/DA` 可变长度汇总

继续审计 `IMPROVmpi` 时发现，原 MPI 实现先用 `MPI_GATHER` 收集每个 rank
的 `NDCOF`，随后却把 `ndcof_max` 作为所有 rank 的 `sendcount`。当某个 rank
的实际 `NDCOF` 小于最大值时，这会从其 `NDA/DA` 数组的未初始化尾部发送数据；
如果该 rank 的分配容量也小于最大值，还可能越过有效分配边界。合并循环虽然只
消费每个 rank 报告的实际 `NDCOF` 项，但这些项的布局已经由固定步长发送决定，
不能把这种发送方式视为安全的 padding。

工作区已在
`rmcdhf_test/src/appl/rmcdhf90_mpi/improvmpi.f90` 准备用 `MPI_Gatherv`：以
各 rank 的实际 `NDCOF` 作为计数，以前缀和作为位移，再广播并合并唯一的
`NDA/DA` 项。该修改已编译进当前正式 `build/bin/rmcdhf_mpi`，但尚未运行，
不能与尚未提交的 QDIF 回归混用；
QDIF 结果完成后，应以同一 Ni I AS1→AS2 输入单独验证这一汇总修复，再扩展到
其他体系。

### 新发现：稀疏矩阵 MPI 分块的列偏移错误

继续对 `IMPROVmpi` 上游的 `MATRIXmpi → MANEIGmpi → SPICMVmpi` 路径做静态审计
时，发现一个比轨道护栏更早发生的 MPI 算法错误。`src/lib/mpi90/spicmvmpi.f90`
按 `ICOL=MYID+1,N,NPROCS` 处理跨步列，却把 `IBEG` 从上一个本 rank 的列延续到
下一列。由于中间列属于其他 rank，下一次乘法实际应使用
`IENDC(ICOL-1)+1:IENDC(ICOL)`；旧代码却从上一个本地列的尾部开始，读入被跳过
列的非零元。`src/lib/mpi90/iniestmpi.f90` 有同样的模式：`JOFFSPAR` 只在本地
列间更新，不能作为跨步列的全局稀疏偏移；每列必须使用
`JCOL(J-1)+1:JCOL(J)`。

用 8×8 对称稀疏矩阵的独立 Python 复算验证了这个静态结论：在 2、3、4 个
rank 下，旧 `SPICMVmpi` 的最大矩阵向量误差分别为约 `1.95、3.81、4.79`，
按每列端点修正后误差为机器精度；`INIESTmpi` 的 packed 矩阵在 2、3 个 rank
下也分别出现非零元值错误，修正后与串行构造完全一致。1 rank 时旧路径恰好
退化为正确的连续列，因此旧的单进程检查不能发现该问题。

工作区已准备两个只改索引的源码修正，并已编译进当前正式
`build/bin/rmcdhf_mpi`，但还没有数值回归：

- `spicmvmpi.f90` 每个本地列重新计算 `IBEG=IENDC(ICOL-1)+1`；
- `iniestmpi.f90` 每个本地列使用 `JCOL(J-1)+1` 作为稀疏起点。

这两个修正已经编译进当前 `build/bin/rmcdhf_mpi`，但没有与 QDIF 或
`MPI_Gatherv` 回归混用。它们仍应在 QDIF 和系数汇总两个单因素结果完成后，
作为“MPI 稀疏矩阵索引”因素用同一 Ni I AS1→AS2 输入验证，再扩展到四个体系。

### 外部 NVMe 成对数据的使用顺序

四个体系的主要验证输入位于
`/home/workstation2/nvmesdd2T/rmcdhf_test_cal`，不是工作区的小型夹具。显式
设置 `GRASP_TEST_DATA_ROOT` 后，`run_data_case.sh` 使用以下目录和命名：

| 体系 | 优化目录/前缀 | 不优化目录/前缀 |
|---|---|---|
| Ni I | `Ni_I/e1_vv2` / `e1_vv2` | `Ni_I/e1_vv2_NV` / `e1_vv2_NV` |
| Ni/Ca-like | `Ni_Ca-like/e1_cv` / `e1_cv` | `Ni_Ca-like/e1_cv_NV` / `e1_cv_NV` |
| Cl I | `Cl_I/o1_cc1` / `o1_cc1` | `Cl_I/o1_cc1_NV` / `o1_cc1_NV` |
| Fe I | `Fe_I/e1_vv3` / `e1_vv3` | `Fe_I/e1_vv3_NV` / `e1_vv3_NV` |

外部优化日志已核对其真实 varied list，尤其 Cl I 使用正号轨道列表，不能套用
工作区 balanced 夹具的负号伙伴列表。外部归档已显示 Ni I、Ni/Ca-like 和 Cl I
的优化谱序偏差，而 Fe I 优化 CSV 尚未归档；因此在 Ni I QDIF 单因素、
`MPI_Gatherv` 单因素和稀疏索引单因素依次完成后，应对每个体系分别提交优化与
不优化脚本，并用 `data/nist_levels/{Ni_I,Ni_Ca-like,Cl_I,Fe_I}.csv` 做同一套
项、宇称和 J 匹配。不能用工作区夹具的结果代替这四组外部成对验证。

对应的 AS1→AS2 外部脚本已经分别写出：

- `run_external_ni_optimized_46.sbatch` / `run_external_ni_nv_46.sbatch`；
- `run_external_nica_optimized_46.sbatch` / `run_external_nica_nv_46.sbatch`；
- `run_external_cl_optimized_46.sbatch` / `run_external_cl_nv_46.sbatch`；
- `run_external_fe_optimized_46.sbatch` / `run_external_fe_nv_46.sbatch`。

这些脚本固定使用 46 个 MPI rank、`/home/workstation2/caltmp` 和当前正式
`build/bin`，并关闭诊断护栏与额外阻尼以重现外部归档的优化/不优化输入语义。
它们必须在三项 Ni I 单因素回归完成后再提交；脚本已经通过 `zsh -n`，但尚未
提交 Slurm 作业。


### Job 605 复盘：QDIF 回归被截断，不能进入下一因素

2026-09-08 的 Job 605 使用 `build-qdif/bin/rmcdhf_mpi` 完成了 Ni I AS1，
但 AS2 只运行到第 23 轮中间输出。AS1 的 `rmcdhf.exitcode=0`，最终顺序为
$^3F_4 < {}^3F_3 < {}^3F_2$，累计间隔为 `1358.20` 和 `2276.86 cm⁻¹`，
相对 NIST `1332.164` 和 `2216.550 cm⁻¹` 的误差为 `+26.036` 和 `+60.310
cm⁻¹`。这与 Job 590/596 的 AS1 完全一致。

AS2 目录没有 `rmcdhf.exitcode`、`status.csv`、最终 `*_rmcdhf.csv` 或非空
`rmcdhf.sum`；`rmcdhf.stdout` 在最后一组 `ORBOPT SOLVE` 后结束。trace 的
第 23 轮临时间隔约为 `1375.90/2291.95 cm⁻¹`（相邻间隔约 `916.05 cm⁻¹`），
但这是未完成计算的快照，不能作为最终物理结果。与 Job 596 对照，前 23 轮
已有的 `SOLVE → DAMPOR → ORTHY` 记录、`5p-` 一次节点进度拒绝以及能量轨迹
没有观察到 QDIF 引起的差异；因此 QDIF 修复尚未被证实有效，也没有被证伪。

由于当前会话不能访问 Slurm accounting socket，不能把缺少退出文件准确标成某个
Slurm 终态；调试记录应使用“AS1 完成、AS2 未完成”分类。为防止再次被分区默认
时限截断，当前 QDIF、`MPI_Gatherv`、稀疏列索引和四体系外部脚本都显式设置了
`#SBATCH --time=02:00:00`。

后续闸门保持严格顺序：

1. 重新提交 `run_qdif_fix_ni_46.sbatch`，确认 AS1/AS2 均有退出码、`status.csv`、
   最终 CSV、逐轮波函数和 `orbopt_summary.csv`；与 Jobs 590、596、597 比较
   第一个不同的 Lagrange multiplier、候选轨道、节点、半径和最终间隔。
2. 只有 QDIF 作业完整后，提交 `run_gatherv_fix_ni_46.sbatch`；只增加
   `MPI_Gatherv`，其余输入、46 rank、模块和 NVMe 临时目录保持不变。
3. 分析 Gatherv 完整结果后，提交 `run_sparse_index_fix_ni_46.sbatch`；验收
   `SPICMVmpi/INIESTmpi` 修复对轨道、节点、谱序和 NIST 间隔的影响。
4. 三个 Ni I 单因素都完成并归档后，再按“优化→不优化”成对提交外部 NVMe
   脚本：Ni I、Ni/Ca-like、Cl I、Fe I。所有主验收仍只使用 MPI，统一以
   `data/nist_levels` 的组态、项、宇称和 $J$ 匹配，不安排串行/MPI 数值对照。

每一步都要把退出状态、是否完整收尾、首次差异轮次、最终节点、谱项与 J 顺序、
相邻及累计间隔、NIST 偏差写回测试结果文档；超时、护栏预期失败和 wrapper 后处理
失败必须分别标注，不能合并成物理算法结论。

### Job 607 结论

QDIF 回归已完整完成（AS1/AS2 exitcode 0，status complete，AS2 33 轮）。最终
Ni I AS2 的 `³F4 < ³F3 < ³F2` 顺序和 `1375.88/2291.99 cm⁻¹` 累计间隔仍偏离
NIST `1332.164/2216.550`（`+43.716/+75.440`）。与旧作业的优化轨迹未出现首次
差异，因此 QDIF 单因素不能解释或修复该物理偏差。下一步按闸门提交
`run_gatherv_fix_ni_46.sbatch`。

### Job 608 结论

MPI_Gatherv 单因素 AS1/AS2 完整成功，结果与 Job 607（QDIF）逐项一致：AS2 `³F4 < ³F3 < ³F2`，累计间隔 `1375.88/2291.99 cm⁻¹`，NIST 偏差 `+43.716/+75.440 cm⁻¹`。未发现首次差异轮次或轨道节点变化，Gatherv 修复未解决精细结构偏差。稀疏列索引修复回归 Job 609 已提交。

### Job 609 结论

稀疏列索引修复作业完整结束，但 AS1/AS2 能级映射严重异常（AS1 为 ³P2/³F2/⁵G2，AS2 为 ³F2/³P2/³P2），说明 `SPICMVmpi/INIESTmpi` 修改破坏了矩阵列索引语义。该结果归类为物理算法失败；外部四体系成对作业 Jobs 610–617 已提交，不能将 Job 609 视为通过。


## 2026-09-10：Jobs 610–617 最终状态核查

本节更新此前运行中/排队的记录。`sacct -X` 确认所有已提交作业均已结束：
607/608/609 为 COMPLETED（0:0）；610/611/612/613/614/616 为 FAILED（1:0）；
615/617 为用户取消（未运行）。没有 TIMEOUT。进程退出成功不能替代物理验收。

| Job | 体系/模式 | 耗时 | AS1 rmcdhf.exitcode | 失败位置与证据 |
|---|---|---|---|---|
| 610 | Ni I 优化 | 00:00:06 | 137 | 输入读取失败：stdout 为 `Not an ISOtope Data File;`，结果目录 isodata 首行是路径文本与 `Atomic number:` 拼接；无 trace、sum 为 0 字节。后处理又因缺少 trace 报错，掩盖了原始输入失败。 |
| 611 | Ni I 不优化 | 00:00:10 | 0 | AS1 单轮结束并生成 CSV，波函数交叉检查报 `fewer than two matching accepted orbital updates`；不优化模式与该检查不兼容。另有严重谱项/间隔异常，不能因修复 wrapper 就判为物理通过。 |
| 612 | Ni/Ca-like 优化 | 00:00:13 | 0 | AS1 第 7 轮按能量判据结束、CSV 已生成；归档比较报 `(1, 0, +)` 能量 -1504.453130741 vs -1497.916938943 Eh。基态被标为 ¹D₂，谱明显异常。 |
| 613 | Ni/Ca-like 不优化 | 00:00:10 | 0 | AS1 单轮结束并生成 CSV，同 611 的 accepted-updates 后处理失败；谱也异常。 |
| 614 | Cl I 优化 | 00:54:17 | 0 | AS1 达 100 轮上限，stdout 为 `Maximum iterations in SCF Exceeded.`，末轮 orbital/energy/final 均 false；检查报最后 scf_end 非 final。CSV 虽生成，间隔为 71110370.95 cm⁻¹，组态异常，不是收敛结果。 |
| 615 | Cl I 不优化 | 00:00:00 | — | 已取消，未运行。 |
| 616 | Fe I 优化 | 00:00:07 | 137 | 输入读取失败：实际有 7 个 J 块，在第 7 块出现 `unable to decode 4s;`，随后 GETOLDWT 第 51 行读取权重时 EOF；rank 0 退出 2，其余 ranks 被终止。无 trace、sum 为 0 字节，后处理缺 trace 是次生错误。 |
| 617 | Fe I 不优化 | 00:00:00 | — | 已取消，未运行。 |

六个已运行外部结果目录均只有 AS1、没有 AS2，也没有顶层 `status.csv`。
因此四体系外部 AS1→AS2 验收尚未完成，不能将现有 CSV 作为完整链通过证据。
原始证据保存在 `data/rmcdhf_test_data/log/61*.log` 和各 `results/external-*/as1/`
的 `rmcdhf.stdout`、`rmcdhf.exitcode`、`orbopt_trace.csv`（存在时）。

后续使用用户已有外部 `_NV` 归档作为不优化参照，不重新提交不优化作业；611/613
仅保留为本次失败诊断证据，不替代已有归档。本次只核查和记录，没有重新提交作业。
优先补齐 Job 608→609 的首次差异与稀疏存储语义审计，随后修正输入生成和失败状态
处理，再对候选修复进行 Ni I MPI 回归，通过后恢复外部优化验收。当前结果支持
“稀疏索引修改后的构建发生严重回归”，尚不足以独立证明具体索引表达式的根因；
此前直接断言具体代码语义已被证实破坏的措辞应以此处的证据边界为准。


## 2026-09-10：失败作业脚本修正与定向重提交

按用户要求仅重新提交 Ni I、Ni/Ca-like、Cl I、Fe I 的外部优化 AS1→AS2，
不重复单因素回归，不重新计算已有 `_NV` 参照。runner 修正 Fe I 的七个 J 块
选择为 `1-5 / 1-4 / 1-8 / 1-6 / 1-7 / 1-2 / 1-2`（34 个 ASF，AS1/AS2
与外部 sum 一致）；Ni I 使用同体系 `_NV/isodata` 并检查核数据首行。
外部脚本开启能量差报告模式，保留 CSF/能级集合和真实收敛检查；缺失 trace
不再掩盖 rmcdhf 原始退出码。不优化模式跳过不适用的轨道更新交叉检查。
每个作业记录二进制路径、SHA-256 和失败阶段，并在 caltmp 下使用 job 专属目录。

本次明确改用已有 `build-gatherv/bin/rmcdhf_mpi`（Job 608 使用过的 QDIF +
Gatherv 构建），避开 Job 609 中发生严重回归的稀疏索引构建。不改 Fortran 源码，
也不把这个构建宣称为已通过四体系物理验收。Cl I 的 100 轮未收敛不能通过关闭
检查来“修好”，此次仍保留未收敛判定，使用上述构建重新计算。

提交前完成 bash/zsh 语法检查、四体系八套真实输入准备和归档 ASF/权重核对；
准备目录为 `data/rmcdhf_test_data/results/script-preflight-20260910-165427/`，
未启动 MPI。用 Job 612 已有 sum 验证能量差报告保留数值并正常返回。

重提交编号：618 Ni I 优化、619 Ni/Ca-like 优化、620 Cl I 优化、621 Fe I 优化。均为 46 ranks / 46 CPU、02:00:00 时限；不优化作业未提交。

### Jobs 618–621 结论（2026-09-11）

输入和后处理修正后的四个外部优化作业均完整完成 AS1→AS2，退出码为 0。结果表明
脚本故障已排除，但原始物理现象仍存在：Ni I AS2 保持 `³F2<³F3<³F4` 但间隔扩大
到 `4243.88/8440.35 cm⁻¹`；Ni/Ca-like AS2 为 `³F2<³F4<³F3` 且两级仅
`62.36/68.08 cm⁻¹`；Cl I 为正确 `²P1/2<²P3/2` 但间隔 `1521.60 cm⁻¹`；
Fe I 保持 `⁵D` 顺序、AS2 累计间隔 `1137.60 cm⁻¹`。四项均与各自归档标签/CSF
一致，不能把 archive comparison 通过误认为 NIST 物理验收通过。下一步只需将这些
优化结果与已有 `_NV` 结果和 `data/nist_levels` 做最终量化对比，再决定代码修复；
不应重复运行已完成的输入/单因素回归。

## 18. 2026-09-11 方案复审：现方案不能单独解决根翻转和谱项跳变

### 18.1 结论

现方案适合作为“轨道候选数值稳定性 + MPI 数据路径”的诊断方案，不能作为
“新增活性轨道优化后始终保持目标能级顺序和谱项”的充分修复方案。它目前主要
约束的是轨道伙伴、节点、重叠、半径、阻尼和 SCF 停止条件，没有约束每个
$J\pi$ 块中的 CI 根身份（root identity）。因此它最多可以减少一部分由轨道
塌缩、节点错误或 MPI 数据损坏引起的错序，不能防止以下两种现象：

1. 相邻 CI 根在优化过程中交换能量顺序，但物理态仍由连续波函数重叠定义；
2. 强混合使某个态的主导组态/谱项改变，这可能是合法的物理混合，也可能是
   错误的根跟踪，不能仅凭最终 CSV 的第一组 `LSJ` 标签区分。

### 18.2 已有结果对方案能力的反证

- B3/B4 在小型 Ni I、Ni/Ca-like 算例中可恢复部分 J 顺序，但 Ni I 的原始
  候选重叠仍低至约 `0.028`、半径因子约 `8.6`，说明“伙伴成对 + 阻尼”并未
  使候选轨道稳定。
- Jobs 607/608 的 QDIF 与 MPI_Gatherv 单因素结果没有改变 Ni I 物理结果；
  Job 609 的稀疏列索引修改反而产生 `³P/³F/⁵G` 等错误谱项，说明 MPI 修复
  必须先有独立的稀疏矩阵单元测试，不能仅以作业 exitcode 为依据。
- Jobs 618–621 虽均完成 AS1→AS2，但 Ni I AS2 的 `³F` 间隔扩大到
  `4243.88/8440.35 cm⁻¹`，Ni/Ca-like AS2 变成 `³F2 < ³F4 < ³F3` 且两级
  只有 `62.36/68.08 cm⁻¹`。这些结果说明“CSV/CSF 标签与归档匹配”和“物理
  态身份及 NIST 间隔正确”是两件事。
- 当前生产 trace 中常见 `convg_energy=true`、`convg_orbital=false`、
  `convg_final=true`。这正是 `SCFmpi` legacy 的 OR 语义：加权总能量先收敛
  即可退出，不能证明新增轨道和目标态已经稳定。

### 18.3 现方案缺少的核心机制：CI 根跟踪

必须把“能级序号”与“物理态身份”分开。对每轮 `MATRIXmpi/NEWCO` 产生的每个
$J\pi$ 块，保存上轮目标 ASF 的 CI 向量，并计算当前与上轮的重叠矩阵；在同一
$J\pi$ 块内用最大重叠的一一匹配确定根，而不是按能量排序或按固定 ASF 序号
认定同一个态。每个目标态还要记录：

- CI 向量重叠及匹配后的根编号；
- 目标组态、项、宇称、$J$ 的 dominant-CFS/ASF 权重；
- 与 AS1/上一 SCF 轮的轨道重叠、节点和平均半径；
- 目标态的相邻及累计间隔。

若能量顺序交换但 CI/轨道重叠连续，应记录为真实或近真实的 root crossing，
并保持物理态标签；若重叠突然跌落、主导组态突变且没有可解释的近简并，则必须
拒绝整轮轨道更新并回退。root assignment 本身是诊断信息，只有伴随低重叠或目标态
身份不连续时才应触发回退；当前 trace 还需要记录这些身份指标才能完成这项判断。

### 18.4 需要替换的最小修复主线

1. **先做状态身份诊断，不先改阻尼参数。** 在 `NEWCOmpi` 后输出各 $J\pi$ 块
   的 eigenvalue、CI 向量重叠匹配、目标组态权重和根交换事件。最终 CSV 的
   `LSJ` 只能作摘要，不能作为根跟踪数据。
2. **把轨道接受从单轨道改为可回退的 SCF 轮次。** 伙伴轨道、同 $\kappa$ 正交
   化组和本轮所有新增活性轨道完成候选、阻尼、正交化后，再统一重建 CI；只要
   任一目标态的根重叠、组态权重、节点或半径验收失败，就恢复整轮轨道和标量状态。
   当前 `IMPROVmpi` 的单轨道回退无法阻止随后其他轨道改变 CI 根。
3. **生产收敛必须为三重门槛：** 轨道判据 AND 加权能量判据 AND 目标态身份在
   连续至少两轮稳定；不得用 legacy 的 OR 语义作为物理验收。`METHOD` fallback、
   节点进度拒绝和整轮回退都必须被记录为未收敛事件。
4. **区分数值修复和物理目标。** 若目标是“与 NIST 的间隔更接近”，需要固定的
   state-specific/EOL 状态集合与权重，或在独立的 CI/RCI 阶段优化目标态；伙伴
   检查和阻尼只能保证轨道更新较稳定，不能凭护栏强制产生 NIST 能级顺序。
5. **MPI 修复单独验收。** `MPI_Gatherv` 和稀疏列修复必须先用 2/3/4 个 rank
   的人工稀疏矩阵、packed 矩阵和非零元索引测试验证；这是库级正确性测试，不是
   用串行生产计算替代 MPI 主验收。Job 609 的谱项损坏在此之前不能进入生产构建。

### 18.5 缩小后的验证闸门

不再重复完整 B0–B8 矩阵。使用一个能稳定复现根跳变的 Ni/Ca-like AS2、一个
Ni I AS2 节点异常夹具和一个 Cl I AS1 倒序夹具，均走 46-rank MPI 生产路径：

- baseline：现有不优化或已归档优化输入，保存每轮 CI/ASF 身份；
- candidate：只加入根跟踪和整轮回退，其他输入、阻尼和 MPI 构建不变；
- acceptance：两次独立完整运行中，目标态匹配无未解释的低重叠/组态突变，最终
  轨道与三重收敛门槛通过；再报告 J 顺序和 NIST 偏差。

只有这三个夹具同时通过，才能判断问题已从“会发生未跟踪的根翻转/轨道塌缩”
降低为可解释的物理近简并。即使通过，也不能承诺所有原子和所有活性空间都保持
NIST 顺序；那需要单独的状态选择和物理模型验证。

### 18.6 当前方案的停止项

在根跟踪、整轮回退和三重收敛门槛实现前，以下方向不应继续作为修复主线：重复
外部优化/不优化作业、继续调 `ODAMP` 数值、放宽 `THRESH`、把最终 `LSJ` 排序
当作状态身份，或把 Job 609 的稀疏索引修改作为生产修复。现有 Jobs 618–621
只证明输入和 wrapper 已能完整运行，不能证明轨道优化缺陷已经解决。

## 19. 关键现象复审：为什么固定 Thomas–Fermi 新轨道反而更接近 NIST

### 19.1 这不是矛盾，而是变分目标与验收目标不同

`rwfnestimate` 生成的 Thomas–Fermi 轨道在“不优化”路径中只是固定的相关轨道基底；
它不进入新增轨道的 `SOLVE → DAMPOR → ORTHY` 更新。固定轨道仍可参与后续
CI/EOL 矩阵，但不会获得独立的轨道松弛自由度。

轨道优化所降低的是当前有限 CSF/EOL 空间中的总能量或加权总能量，并不保证每个
$J$ 态的相对能量、精细结构间隔或 NIST 偏差同时降低。某个 $J$ 块若从新增轨道
松弛中获益更多，就会出现总能量下降而精细结构变差的情况。固定 TF 轨道的误差
可能在各 $J$ 态之间较均衡，甚至产生误差抵消，所以相对间隔更接近 NIST；这不是
“初猜比收敛轨道更正确”的变分定理结论，而是有限空间和状态加权下的结果。

### 19.2 对当前四体系证据的解释

现有模式符合“新增轨道更新触发异常”的共同特征：不优化目录保持目标谱序且较
接近 NIST，优化目录在 Ni I、Ni/Ca-like 和早期 Cl I 数据中出现倒序或塌缩；
Fe I 虽保持顺序，间隔仍发生明显变化。Ni I AS2 初始 `5d/6s` 节点异常在
读入波函数时已经可见，但固定 TF 轨道仍能得到正确谱，这说明初始轨道异常本身
不是充分原因；危险步骤是其随后进入 `SOLVE/DAMPOR/ORTHY` 并改变共享轨道和 CI
矩阵。

因此需要同时考虑三种机制：

1. **不平衡轨道松弛：** 新关联轨道对不同 $J$ 态的相关能贡献不相等，EOL 总能量
   下降却扩大精细结构误差；
2. **径向分支或轨道塌缩：** `SOLVE` 可能从 TF 轨道跳到错误节点/半径分支，
   `ORTHY` 再把该变化传播到同 $\kappa$ 轨道；
3. **CI 根和组态混合变化：** 优化后同一能量序号可能对应另一根，或主导组态
   权重真正发生强混合。最终 `LSJ` 文本不能区分这三者。

这也解释了为什么单独的伙伴成对、QDIF、MPI_Gatherv 或负阻尼不能作为充分修复：
它们最多约束其中一部分轨道更新路径，不能保证目标态相对能量和 CI 根身份同时
保持。

### 19.3 最小判别实验（无需重复整套回归）

后续只需要同一 Ni I AS2 和 Ni/Ca-like AS2 各做以下三个固定 MPI 生产路径：

- **TF-freeze：** 保持现有 TF 新轨道，继续优化原有 spectroscopic 轨道；
- **one-new-orbital：** 每次只允许一个新增轨道进入 `IMPROV`，记录第一次改变
  能级间隔、节点、半径或 CI 根身份的轨道；
- **fixed-orbital CI/RCI：** 对 TF 轨道和优化后轨道分别使用完全相同的固定轨道
  CI/RCI 空间和状态集合，比较 CI 本身与 RMCDHF 自洽更新的差异。

每个运行只需保存目标态的 CI 向量重叠、主导 CSF 权重、每个 $J$ 的能量和总
EOL 能量。若 TF-freeze 正常而 one-new-orbital 在某个轨道立即失败，根因位于
该新增轨道的求解/阻尼/正交化路径；若 fixed-orbital CI 已经正确而 RMCDHF
更新后错误，根因位于轨道松弛或 EOL 状态权重；若 CI 本身也发生根交换，则还要
加入上一节的 root tracking。

### 19.4 实际生产策略

在根身份跟踪、整轮回退和状态收敛门槛完成前，**固定新增 TF 轨道是合理的生产
工作区方案**，因为它已经在现有验证集上提供了正确谱序和较好的相对间隔。它不能
被称为轨道优化修复，也不能外推为所有元素的高精度结果；它是避免错误新增轨道
松弛的保守运行模式。后续若要恢复新增轨道优化，应先通过上述最小判别实验，再
决定采用成组、state-specific 或分阶段的受控优化。

## 20. 定向轨道隔离测试（2026-09-11，Jobs 631/632）

定向隔离脚本已经实际完成，而不是停留在预检。Job 631（Ni I）和 Job 632
（Ni/Ca-like）均为 `COMPLETED 0:0`；每个变体的 `rmcdhf.exitcode=0`，结果目录
分别为 `data/rmcdhf_test_data/results/orbital-iso-nii-631/` 和
`data/rmcdhf_test_data/results/orbital-iso-nica-632/`。每个运行都满足
`energy_converged=true`，但 `orbital_converged=false`，所以这里的“完成”仍是
legacy 能量停止语义，不能当作轨道物理收敛通过。

`freeze_new` 保留 AS1 的 varied list，只固定 AS2 新增轨道，不等同于已有的
`_NV` 不优化基线。每个体系均使用同一 AS1 波函数，比较 `freeze_new`、`all_new`
和逐个 `only-*` 变体；汇总器已修正 Ni I 的 NIST 零点为 `J=4`，并先将计算能级
归一到计算的 `J=4` 再计算误差。

Ni I 的所有变体最终都是 `J2 < J3 < J4`。`all_new` 的相对 `J=4` 间隔为
`E3-E4=-4196.47`、`E2-E4=-8440.35 cm⁻¹`，相对 NIST 的误差为
`-5528.63/-10656.90 cm⁻¹`；`freeze_new` 仍为错误顺序，不能替代 `_NV` 基线。
`only-5d` 和 `only-6s` 分别出现 1 次接受后节点变化，`all_new` 首轮 `6s`
重叠为 `0.04646`，`only-4f` 首轮 `4f` 重叠为 `0.02814`。这说明 Ni I
的问题在单个新增轨道进入优化路径时就可以出现，不是只有多轨道同时更新才触发。

Ni/Ca-like 的 `freeze_new` 和所有 `only-*` 变体均保持 `J2 < J3 < J4`，但
相对 NIST 间隔偏小；`all_new` 在第 3 轮发生首次能量顺序变化，最终为
`J2 < J4 < J3`，相对 `J2` 的间隔仅为 `E4-E2=62.36`、`E3-E2=68.08 cm⁻¹`。
所有变体都没有低重叠或接受后节点变化。因此这里更符合多轨道 EOL/CI 根耦合或
根身份变化，而不是单个轨道节点塌缩。

这批结果完成了方案第 19.3 节所需的最小隔离判别：新增轨道更新是触发错误谱序
的必要诊断轴，但现有 trace 仍没有 CI 向量重叠匹配，不能区分真实近简并和未跟踪
的根交换。后续主线仍应是 CI 根跟踪、整轮轨道回退和轨道/能量/目标态身份三重
收敛门槛；不再重复这些已完成的变体，也不把 `freeze_new` 称为轨道优化修复。

## 22. 固定轨道 CI/RCI 成对对照（2026-09-11，Job 635）

第 2 步已经完成。新增脚本
`rmcdhf_test/test/rmcdhf_orbopt/slurm/run_fixed_orbital_rci_pair.sbatch`
对 Ni I 和 Ni/Ca-like 各生成一套 AS2 Thomas–Fermi 波函数，再把它与 Jobs
631/632 的 `all_new` 最终波函数分别送入完全相同的固定轨道 RCI。两套 RCI 使用
相同的 AS2 CSF、相同的状态选择和相同的后处理；CSF 哈希分别为
`4630ea1b...20fc38` 和 `5c82085a...95ec33`。结果目录为
`data/rmcdhf_test_data/results/fixed-orbital-rci-pair-635/`，成对汇总在
`pair_summary.csv`。需要保留一个输入边界：这批 TF 波函数来自 `_NV` AS1，
而 631/632 的优化波函数继承的是各自优化 AS1，因此它不是严格的同一 AS1 对照，
只能作为跨基线结果。

| 体系/轨道 | 目标态顺序 | 第一间隔（cm⁻¹） | 第二间隔（cm⁻¹） | 相对 NIST 误差（cm⁻¹） |
|---|---|---:|---:|---:|
| Ni I，TF | `J4 < J3 < J2` | `E3-E4=1305.08` | `E2-E4=2168.02` | `-27.08 / -48.53` |
| Ni I，优化后 | `J2 < J3 < J4` | `E3-E4=-4262.78` | `E2-E4=-8606.07` | `-5594.94 / -10822.62` |
| Ni/Ca-like，TF | `J2 < J3 < J4` | `E3-E2=1833.33` | `E4-E2=4031.32` | `-46.67 / -38.68` |
| Ni/Ca-like，优化后 | `J4 < J3 < J2` | `E3-E2=-102.85` | `E4-E2=-369.73` | `-1982.85 / -4439.73` |

这组跨基线结果说明 `_NV` AS1 上的 TF AS2 与优化链最终轨道之间存在巨大差异，
但不能把差异全部归因于 AS2 新轨道优化。Ni I 的固定 RCI 顺序与 RMCDHF `all_new`
一致；Ni/Ca-like 的固定 RCI 为 `J4 < J3 < J2`，而 RMCDHF 为 `J2 < J4 < J3`，
说明自洽 EOL 更新还会继续改变各 $J$ 态的相对移动。

四个 RCI 的前三个态仍被标记为同一 `^3F_J` 谱项，主导 CSF 权重也保持在约
`0.90–0.99`；这批数据没有证明目标态直接换成另一组态，也没有证明发生了 CI
根编号交换。它证明的是更早、更基本的一点：优化后的轨道基组本身已经足以破坏
能级顺序。严格的 AS2 归因需要保持 631/632 的 AS1 不变，再比较 TF AS2 与优化
AS2；这一步见下一节。

## 23. 同一 AS1 的严格 TF/优化 AS2 对照（Job 636）

Job 636 使用 Jobs 631/632 各自保存的 `previous.w` 作为共同 AS1，运行
`rangular_mpi → rwfnestimate` 只生成 AS2 TF 轨道，再运行与 Job 635 相同的固定
RCI。已有的优化轨道 RCI CSV 直接复用，没有重复优化 RCI。结果目录为
`data/rmcdhf_test_data/results/matched-tf-rci-636/`，能级分解在
`decomposition_summary.csv`，AS1 基线变化在 `radial_as1_nii.csv`/
`radial_as1_nica.csv`，AS2 变化在 `radial_nii.csv` 和 `radial_nica.csv`。

| 体系/波函数 | 目标态顺序 | 第一间隔（cm⁻¹） | 第二间隔（cm⁻¹） |
|---|---|---:|---:|
| Ni I，`_NV` AS1 + TF AS2（Job 635） | `J4 < J3 < J2` | `1305.08` | `2168.02` |
| Ni I，优化 AS1 + TF AS2（Job 636） | `J2 < J3 < J4` | `E3-E4=-1411.04` | `E2-E4=-2730.88` |
| Ni I，优化 AS1 + 优化 AS2 | `J2 < J3 < J4` | `E3-E4=-4262.78` | `E2-E4=-8606.07` |
| Ni/Ca-like，`_NV` AS1 + TF AS2（Job 635） | `J2 < J3 < J4` | `1833.33` | `4031.32` |
| Ni/Ca-like，优化 AS1 + TF AS2（Job 636） | `J2 < J3 < J4` | `893.62` | `1953.59` |
| Ni/Ca-like，优化 AS1 + 优化 AS2 | `J4 < J3 < J2` | `E3-E2=-102.85` | `E4-E2=-369.73` |

严格配对后的结论是：Ni I 的错误顺序在 AS1 优化后、AS2 新轨道仍保持 TF 时
已经出现；AS2 优化进一步把间隔推向错误分支。Ni/Ca-like 的 AS1 优化先保持
顺序但显著压缩间隔，AS2 优化再造成反转。径向指标也只在新增 AS2 轨道上出现
明显变化：Ni I 为 `4f/5d/5p/6s`，重叠分别约 `0.008/0.205/0.345/-0.082`；
Ni/Ca-like 为 `5s–5g`，重叠约 `0.12–0.32`，平均半径缩小约 2.8–3.2 倍，且
部分节点发生变化。相对于 `_NV` AS1，Ni I 的 AS1 `4d/5s` 重叠只有约
`0.397/-0.670`；Ni/Ca-like 的 `4s–4f` 重叠约为 `0.865–0.957`。现阶段不能
再把问题概括为“只要固定 AS2 新轨道就恢复正确谱序”；生产上应先固定经过验证
的 AS1/TF 基线，再单独设计 AS1 与 AS2 的受控优化实验。

## 24. 631/632 轨迹与 636 分解的离线合并（2026-09-11）

没有重新提交作业。新增
`rmcdhf_test/test/rmcdhf_orbopt/analyze_as1_as2_contributions.py`，直接读取
631/632 的 `orbopt_trace.csv`、已有的 `ci_root_analysis/` 和 636 的
`decomposition_summary.csv`，生成
`data/rmcdhf_test_data/results/matched-tf-rci-636/as1_as2_contribution.csv` 与
`as1_as2_contribution.md`。能级首次变化使用相对第 0 轮超过 `1 cm⁻¹` 的门槛；半径
比定义为 `radius_candidate/radius_old`，同时给出对称半径因子。

离线汇总得到三个关键结果：

1. **Ni I 的单轨道路径在第 1 轮就改变精细结构间隔。** `only-4f` 的最低接受
   重叠为 `0.0281`、最大半径因子为 `8.55`；`only-6s` 为 `0.0465/3.15`，并有
   1 次节点变化；`only-5d` 为 `0.197/4.40`，也有 1 次节点变化；`only-5p`
   的变化较小（`0.466/2.47`）。四个单轨道变体最后仍为 `J2<J3<J4`，因此
   “新增轨道一进入优化就改变能级”已被逐轨道复现，但不能把所有变化归因于同一个
   节点塌缩。
2. **Ni/Ca-like 的单轨道变体没有造成目标态顺序反转。** `only-5s/5p/5d/5f/5g`
   的最大半径因子约为 `1.84–2.78`，接受后节点变化均为零，最后都保持
   `J2<J3<J4`；`all_new` 才在第 3 轮变为 `J2<J4<J3`。它的 CI 根向量最低匹配
   重叠为 `0.9705`，第 1、3 轮出现全局能量排序变化，但 13 个变体均没有非恒等
   CI 向量匹配。因此目前没有“CI 根编号交换”证据，Ni/Ca-like 更符合多轨道 EOL
   能量重排或近简并耦合。
3. **严格 AS1/AS2 归因保持不变。** 636 的固定 RCI 已显示 Ni I 在“优化 AS1+
   TF AS2”时就已经倒序，AS2 优化只会继续放大错误间隔；Ni/Ca-like 则由优化
   AS1 先压缩间隔，AS2 全量优化再触发反转。因而当前可执行的生产策略仍是固定已
   验证的 AS1/TF 基线；恢复新增轨道优化前，必须实现整轮回退、目标态身份跟踪和
   三重收敛门槛，并继续保留本离线报告作为验收输入。

这一步已经足以决定下一步方向，不需要再重复 Jobs 631/632/635/636。后续代码工作应
在现有 trace 上实现“候选更新→整轮接受/回退”的状态机，并以目标 $^3F_J$ 的
CI 向量重叠和相对能量同时验收；若仅增加根编号重排检查，不能覆盖 Ni I 的 AS1
已倒序情形。

## 25. 整轮候选回退与身份门槛的代码实施（2026-09-11）

本步已完成代码接入，尚未启用新开关运行生产作业，也没有重复 Jobs
631/632/635/636。新增 `src/appl/rmcdhf90_mpi/orbopt_round_state.f90`，在每轮
轨道更新前保存 `PF/QF`、轨道标量、CI 能量/向量、块平均能量、权重以及
`ICCMIN/IATJPO/IASPAR`；`MATRIXmpi → NEWCOmpi` 后计算全局能量排序和每个
$J\pi$ 块内的绝对 CI 向量重叠匹配。目标态顺序变化或最低重叠低于门槛时，恢复
整轮状态、增加负阻尼、清除本轮收敛标志，并覆盖写回恢复后的
`.w` 文件；超过回退次数上限则停止，避免错误状态被报告为收敛。

`scfmpi.f90` 已将该状态机接在完整的轨道更新和重新对角化之后；严格收敛条件在
启用整轮护栏时额外要求 CI 重叠连续稳定，允许高重叠的合法 root crossing。`orbopt_trace.f90` 增加 `round_decision` 事件，
记录接受/回退、最低重叠、能量顺序变化、根匹配和回退次数。新增控制默认关闭，
可通过 `GRASP_ROUND_ROLLBACK_GUARD`、`GRASP_ROUND_REJECT_ENERGY_ORDER`、
`GRASP_MIN_STATE_OVERLAP` 和 `GRASP_MAX_ROUND_ROLLBACKS` 开启；默认关闭时不改变
历史输入顺序和数值路径。

验证只进行本地构建：加载 `mpi/openmpi-x86_64` 后，`cmake --build build -j2
--target rmcdhf_mpi` 成功，且 `git diff --check` 无空白错误。该构建结果证明模块
接口和链接完整，不等同于三个物理夹具已经通过；下一步应在不重复既有基线的前提下，
用 Ni I、Ni/Ca-like 和 Cl I 三个最小夹具各运行一次新开关，检查 `round_decision`
以及三重收敛日志，再决定是否扩大测试范围。

## 26. Job 638：整轮回退护栏的首轮生产路径验证

Job 638 使用 46 个 MPI rank、`GRASP_ROUND_ROLLBACK_GUARD=1`、
`GRASP_ROUND_REJECT_ENERGY_ORDER=1`、`GRASP_MIN_STATE_OVERLAP=0.85`、
`GRASP_MAX_ROUND_ROLLBACKS=3` 和严格收敛模式，结果目录为
`data/rmcdhf_test_data/results/round-guard-minimal-638/`。这次测试的目的只是确认
候选轨道状态不会在身份异常时继续写成最终结果，不能把保护性停止当作物理收敛。

| 算例 | round_decision | 最低重叠 | 终止证据 | 最终 CSV |
|---|---:|---:|---|---|
| Ni I AS2 | 4（全部回退） | 0.154087 | 第 4 次回退超过上限，`ERROR STOP SCFmpi: round rollback limit exceeded` | 无 |
| Ni/Ca-like AS2 | 4 回退、3 接受 | 0.071677 | 第 7 轮重叠 0.618559，再次回退后超过总上限 | 无 |
| Cl I AS1 | 100（全部接受） | 0.999323 | 达到 100 轮上限，轨道和能量严格条件未同时满足 | 有 |

Ni I 的四次回退均同时报告 `state_overlap+root_assignment`，说明旧候选与保存
状态的最低 CI 向量重叠约为 0.15–0.18，且贪心匹配得到非恒等根分配。护栏在生成
最终 `.w`/CSV 之前停止了计算。Ni/Ca-like 的前三轮最低重叠为 0.071677、
0.105908、0.391092，随后三轮重叠升至 0.981732、0.881384、0.952346 而被接受；
第 7 轮降至 0.618559 后又回退，说明状态机确实能接受恢复稳定的候选，也能阻止
后续恶化。Cl I 没有回退、非恒等根匹配或能级顺序变化，但 100 轮末仍为
`convg_orbital=false`、`convg_energy=false`，因此只是成功运行到上限。

这批结果证明了“检测并拒绝危险整轮候选”的控制路径，尚未证明它能找到正确的
物理解。Ni I 和 Ni/Ca-like 被安全停止而不是恢复收敛，下一步要在不放宽门槛的
前提下分析候选失败的具体轨道、目标 $J\pi$ 块和根身份；若要恢复计算，必须先把
能量顺序比较限定到目标态集合。CI 向量匹配现已改为每个 $J\pi$ 块内的
Kuhn–Munkres 全局一一匹配，但 Job 638 的旧 trace 仍是此前的逐步贪心结果；
全局跨 block 排序仍不能作为物理态顺序判据。

Job 638 还暴露出报告层面的歧义：旧版汇总只记录 `runner_exit`，所以 Cl I 的
`rmcdhf.exit=0`、已有最终 CSV 但严格检查失败被显示成 `runner_exit=1`。现已在
`run_data_case.sh` 写出 `convergence_check.exitcode`，并让
`run_round_guard_minimal_46.sbatch` 的状态表同时记录 RMCDHF 退出码和严格检查
退出码，区分 `rmcdhf_failed`、`rmcdhf_success_strict_not_converged`、
`strict_converged` 及后处理失败。该修改只改善状态分类，不改变数值路径。

`orbopt_trace.csv` 也已增加 `round_accepted`、`round_order_changed`、
`round_nonidentity`、`round_rollback_count`、`round_min_overlap` 以及对应控制
参数的专用列；旧的通用收敛列不再承载回退语义。这个改动只改善可审计性，Job 638
已有 trace 仍按旧列解释。全局块内能量判据和 CI 根身份跟踪完成前，不应扩大生产
回归，也不应重复 Job 638 或把其三个结果作为顺序修复证据。

## 27. 目标态范围修正与离线验收（2026-09-12）

已继续实施目标态范围修正。`GRASP_ROUND_TARGET_STATES` 现在按
`NEWCOmpi` 的 **1-based 全局 state index** 指定目标根；配置后，整轮最低 CI
重叠只在目标根对应的当前行上作为回退门槛，未跟踪的辅助根仍保留完整
Kuhn--Munkres assignment 诊断，但不会单独令整轮失败。目标态能量顺序仍通过
同一 assignment 映射回旧根，因此目标态跨不同 $J\pi$ block 时也能比较其物理
顺序。没有设置目标态时保持“所有根参与重叠门槛”的兼容行为；能级顺序拒绝在此
情况下自动关闭，避免把全局 state index 顺序误当作物理排序。

最小 46-rank 诊断脚本已给出当前夹具的默认范围：Ni I 和 Ni/Ca-like 使用
`3,4,7`，Cl I 使用 `1,2`；ASF 选择变化时可分别通过
`GRASP_ROUND_TARGET_STATES_NI_I`、`GRASP_ROUND_TARGET_STATES_NICA` 和
`GRASP_ROUND_TARGET_STATES_CL_I` 覆盖。trace 的控制行记录目标态原始字符串，
每个 `scf_end` 行新增 `round_identity_stable`，严格收敛检查据此验证三重门槛，
并兼容没有该字段的旧 trace。

新增 `test/rmcdhf_orbopt/test_round_state_logic.py`，离线覆盖恒等匹配、合法
root 交换、非贪心矩阵的全局最优 assignment、跨 block 目标态顺序以及辅助根低
重叠不触发目标门槛五个情形。执行 `python3 test/rmcdhf_orbopt/test_round_state_logic.py`
已通过；更新后的 `rmcdhf_mpi` 也已重新本地构建成功。下一次只需提交一批带默认
目标态范围的最小护栏作业，检查真实 trace 是否不再因非目标 block 的低重叠回退，
再决定是否调整阈值或扩大算例。

## 28. Job 639 复核：测试输入与 trace 输出缺陷（2026-09-12）

Job 639 不能作为目标态护栏的有效验证。复核发现两个测试层缺陷：

1. 脚本把 Ni I 和 Ni/Ca-like 的目标态误设为 `4,5,6`，这三个索引都属于
   $J=3$ block；目标 $^3F_J$ 的全局 state index 应为 `3,4,7`，分别对应
   $J=2,3,4$。Cl I 的 `1,2` 正确。
2. trace 控制行把逗号分隔的目标态字符串直接写入 CSV 未加引号，导致
   `compare_rmcdhf.py` 报 `inconsistent CSV schema`。因此 Cl I 虽然 RMCDHF
   本身以退出码 0 完成，后处理仍被损坏的 trace 阻断；Ni 两个算例的
   `round_decision` 可以读取，但不能作为完整验收证据。

Job 639 的可用诊断仍显示：错误目标集合下，Ni I 四轮均因
`energy_order+state_overlap` 回退；Ni/Ca-like 前三轮因低重叠回退，随后 32 轮接受，
最终因 90 分钟超时退出 124；Cl I 100 轮均接受并达到 SCF 迭代上限。由于目标集合
和 CSV 均有问题，这些现象不用于判断修复效果。

现已修正：默认目标集合改为 `3,4,7`，trace 输出对含逗号字段使用 CSV 引号，
状态表把 `convergence_check` 尚未生成正确分类为后处理失败，而不是严格收敛失败。
修正后重新构建和离线测试已通过；下一次测试必须重新生成有效 trace 后再评价目标态
护栏，不能复用 Job 639 的结果。
