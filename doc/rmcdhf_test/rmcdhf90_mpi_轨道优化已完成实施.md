# `rmcdhf90_mpi` 轨道优化已完成实施

## 1. 文档范围

本文档记录 [rmcdhf90_mpi_轨道优化调试方案](./rmcdhf90_mpi_轨道优化调试方案.md) 中已经落地的代码、运行开关和诊断工具。实施源码位于：

```text
/home/workstation2/AppFiles/GraspKit-Workspace/rmcdhf_test
```

最新核查日期为 2026-08-30；核查时 `rmcdhf_test` 位于分支
`0.1.0-test1`，本地 HEAD 为 `8028fcd`
(`test: cover MPI reproducibility comparator`)。固定阻尼矩阵实施由
`883ebcc` (`rmcdhf_mpi: add fixed damping matrix`) 引入，主要轨道优化
诊断功能最初由 `b30a1e3`
(`Add RMCDHF orbital optimization safeguards`) 引入。核查时本地分支比
`origin/0.1.0-test1` 领先 8 个提交；因此这些后续提交在推送前只能作为
本地可追溯实施，不应表述为已在远程分支发布。未跟踪的 `results/`
目录不列入仓库内已归档证据。

## 2. 总体实施状态

当前已完成 P0 诊断能力的主体，完成了相对论伙伴检查、候选轨道指标、固定阻尼和实验性轨道护栏。所有会改变数值路径的功能默认关闭，未向历史 stdin 增加新问题。

本阶段完成的是“可诊断、可对照、可做 A/B 实验的原型”，不等于方案第 7 节的最终验收已经完成。

方案与当前实施的对应状态如下：

| 方案项 | 当前状态 | 说明 |
| --- | --- | --- |
| P0 诊断、trace、伙伴检查和轨道指标 | 已实施 | 默认不改变历史数值路径 |
| B0–B6 诊断矩阵 | 已实施并有阶段性结果 | B5 是有记录的预期失败，不是可用修复 |
| B7 独立 CI/RCI | 部分完成 | 只完成 Cl I AS1 单例 |
| B8 EOL 权重/状态集 | 部分完成 | 等权/统计权重已完成，不同状态集未完成 |
| 串行/MPI 和每变体三次重复 | 部分完成 | 已选变体通过，尚非 B0–B8 全矩阵 |
| 护栏拒绝后自动恢复 | 未完成 | 当前只能拒绝并明确终止 |
| 方案第 7 节生产验收 | 未通过 | 完整 MPI 矩阵、B7/B8 和节点稳定性仍缺失 |
| NP0–NP3 | 未开始 | 符合主线验收后再做的阶段顺序 |

原型的首次导入提交 `b30a1e3` 同时包含控制、trace、指标、护栏和
测试工具，没有遵守调试方案第 6 节“每个提交只解决一个问题”的
提交粒度要求。本文档按功能映射实施状态，不将历史提交序列追认为已符合该要求。

## 3. 已完成的代码实施

### 3.1 独立运行时控制模块

新增 `src/appl/rmcdhf90_mpi/orbopt_control.f90`，定义 `ORBOPT_CONTROL_C` 模块，已实现：

- rank 0 读取环境变量；
- 通过 `MPI_Bcast` 将控制值同步到所有 rank；
- 检查阻尼、重叠阈值、半径比和拒绝次数的有效范围；
- 记录每条轨道的连续拒绝次数；
- 保留默认历史行为。

已提供的环境变量如下。

| 环境变量 | 作用 | 默认值 |
| --- | --- | --- |
| `GRASP_TRACE_ORBOPT` | 启用轨道优化 trace | 关闭 |
| `GRASP_REQUIRE_BALANCED_PAIR` | 将不平衡伙伴从警告升级为终止 | 关闭 |
| `GRASP_ORBITAL_DAMPING` | 为可变轨道设置固定负 `ODAMP` | `0`，不覆盖 |
| `GRASP_ORBITAL_GUARD` | 启用候选轨道护栏 | 关闭 |
| `GRASP_MIN_ORBITAL_OVERLAP` | 最小候选重叠 | `0.1` |
| `GRASP_MAX_RADIUS_RATIO` | 最大对称平均半径变化因子 | `10` |
| `GRASP_REJECT_NODE_CHANGE` | 节点数改变时拒绝候选 | 开启 |
| `GRASP_MAX_REJECTS_PER_ORBITAL` | 单轨道最大连续拒绝次数 | `3` |
| `GRASP_STRICT_SCF` | 要求轨道与能量判据连续两轮同时满足 | 关闭 |
| `GRASP_TRACE_RWFN` | 保存逐轮 `rwfn.out.iterNNN` 供外部交叉验证 | 关闭 |
| `GRASP_DEFER_ORTHY` | B5：只在宏迭代末执行正交化 | 关闭 |
| `GRASP_STRICT_METHOD3` | B6：固定变分轨道 METHOD=3，禁止失败后降级 | 关闭 |

`rscfmpivu.f90` 在 `startmpi2` 完成后设置 trace 目录并调用 `INIT_ORBOPT_CONTROL`。`Makefile` 和 `CMakeLists.txt` 均已包含新模块。

### 3.2 结构化轨道优化 trace

新增 `src/appl/rmcdhf90_mpi/orbopt_trace.f90`，开启 `GRASP_TRACE_ORBOPT=1` 后，rank 0 向运行起始目录写入 `orbopt_trace.csv`。已记录：

- 解析后的轨道选择：`NW`、`NFIX`、`IORDER`、`NP/NH/NAK`、`LFIX`、`LCORRE`、`METHOD`、`NOINVT`、`ODAMP`；
- SCF 轮次开始状态：`NIT`、`NSCF`、`NSIC`、`ACCY`、`ORTHST`、`LSORT`、`WTAEV0`；
- `SETLAGmpi` 建立的 Lagrange 乘子轨道对；
- `SOLVE` 结果、候选归一化、`INV/JP/NNP`、`METHOD` fallback 和失败状态；
- 候选轨道与 `DAMPOR` 后实际接受轨道的质量指标；
- `ORTHY` 的分组、Schmidt 顺序和每次投影重叠；
- 每轮能级、加权能量和 SCF 结束判据；
- 关键数组的每 rank 摘要。仅在 rank 摘要不一致时写入 `orbopt_trace.rankNNN.csv`。

### 3.3 相对论伙伴完整性检查

`getoldmpi.f90` 在完成 varied list 和 spectroscopic list 解析后调用 `CHECK_RELATIVISTIC_PAIRS`。配对使用 `NP` 和 `NAK=κ`，不依赖轨道名字中是否带 `-`。

已实现两级行为：

1. 默认模式输出不平衡或缺失伙伴的明确警告，不修改用户输入；
2. `GRASP_REQUIRE_BALANCED_PAIR=1` 时在进入 SCF 前终止。

测试 runner 中已增加 `minus_only`（B2）和 `balanced`（B3）输入变体；`balanced` 会同时开启严格伙伴检查。

### 3.4 候选轨道质量指标

新增 `src/appl/rmcdhf90_mpi/orbopt_metrics.f90`。在 `SOLVE` 得到并归一化候选轨道后，复用 GRASP 的 `TA/RP/QUAD` 积分约定计算：

- 候选轨道与旧轨道的有符号径向重叠；
- 新旧轨道范数；
- 新旧平均半径；
- 新旧大分量节点数；
- `MF/MTP0`、轨道能量改变和相关求解器状态。

同样的指标会在 `DAMPOR` 后再计算一次，从而区分“原始候选轨道”和“实际接受的阻尼轨道”。

### 3.5 可配置的固定阻尼

`GRASP_ORBITAL_DAMPING` 允许把 `(-1,0)` 内的固定负阻尼应用到所有可变轨道。无效值会被拒绝并恢复为不覆盖历史 `ODAMP`。

该功能已用 B3+B4 实验评估；当前仍默认关闭，没有把 `-0.5` 硬编码为生产默认值。

### 3.6 实验性候选轨道护栏

`GRASP_ORBITAL_GUARD=1` 时，`IMPROVmpi` 在 `DAMPOR` 和 `ORTHY` 之前评估原始候选轨道。已实现：

- 按重叠、半径比和节点变化拒绝异常候选；
- 拒绝时不调用 `DAMPOR` 和 `ORTHY`；
- 恢复 `E(J)`、`MF(J)` 和 `PZ(J)`，并保留原 `PF/QF`；
- 将该轨道保持为未收敛；
- 逐次提高建议阻尼强度；
- 连续拒绝超过上限时明确终止，避免被 SCF 误判为收敛。

该护栏仍是诊断原型。已有实验表明，先拒绝原始候选可能阻止本来经 `DAMPOR` 后可以稳定的更新，因此该功能不应作为生产默认策略。

## 4. 已完成的运行和分析工具

`test/rmcdhf_orbopt/` 下已提供：

- `run_data_case.sh`：在新输出目录中运行 Ni I 或 Ni/Ca-like AS1/AS2，支持 `optimized`、`nv`、`minus_only`、`balanced`；
- `compare_sum.py`：比较 CSF 数、径向网格和 `rmcdhf.sum` 能级；
- `compare_rmcdhf.py`：从 trace 生成每轮能量间隔、最小重叠、最大半径因子、节点变化和 fallback 摘要；
- `compare_fine_structure.py`：比较各变体的 J=2、3、4 最低正宇称能级顺序和相对能量；
- `check_damping_repeatability.py`：对比多轮阻尼矩阵的离散字段和浮点摘要；
- `run_damping_repeats.sh`：在同一 CPU 亲和性下运行三次完整阻尼矩阵并自动比较；
- `README.md`：记录运行方法、参数和输出物；
- `RESULTS.md`：记录已完成的 AS2 B2/B3/B4 结果。

从 2026-08-26 起，所有测试 runner 都强制将生成结果写入
`rmcdhf_test/test/data/` 下的结果子目录；相对路径自动解析到该目录，目录外的
绝对路径会被拒绝。每次运行都会从 `test/data` 中的现有算例复制 `isodata`、CSF
和波函数输入，再在新的结果目录计算，避免结果与输入来源脱离。

runner 会保存 stdin、stdout、退出码、`orbopt_trace.csv`、`orbopt_summary.csv`、`rmcdhf.sum` 及与归档基线的比较。若输出目录已存在则拒绝覆盖。
多线程运行还会保存 `mpi_launcher.txt`；其中记录显式的
`--map-by slot:PE=N --bind-to core` 启动参数与 OpenMP 线程设置。

`rsave` 会将通用 `rmcdhf.sum` 重命名为算例前缀的 `.sum`文件。
提交 `71954ca` 在 `rsave` 后恢复通用名副本，保持
`compare_sum.py`、`compare_fine_structure.py` 和 `run_matrix.sh` 之间的稳定产物接口。

## 5. 已完成的构建核查

2026-08-24 使用以下环境进行了独立干净构建：

```text
Fortran: GNU Fortran 15.2.1
MPI: OpenMPI，通过 mpi/openmpi-x86_64 module 加载
BLAS/LAPACK: FlexiBLAS
CMake build type: Debug
```

`rmcdhf_mpi` 目标编译和链接成功。构建输出仍包含 GRASP 历史源码中的 MPI 参数类型/秩不匹配警告，但本次新增模块没有造成编译失败。

仓库内原有 `build-debug/CMakeCache.txt` 记录的旧源路径是 `/home/workstation2/AppFiles/grasp_test`，不能在当前路径下直接增量构建；使用新的 out-of-source 构建目录可正常完成构建。

### 5.1 2026-08-24 后续实施与验证

本轮补齐了方案 6.7 的严格收敛判据：`CONVG_LEGACY` 保持轨道或能量的历史 OR 语义，`CONVG_STRICT` 使用轨道与能量的 AND 语义；严格模式只在条件连续满足两轮后退出，并将首轮能量比较标为无效。Ni/Ca-like AS1、balanced、单 MPI rank 对照中，默认模式第 7 轮按能量条件退出，严格模式运行到第 13 轮，在第 12、13 轮连续满足双条件后退出。

新增了默认关闭的逐轮 RWFN 快照和独立解析工具：

- `compare_rwfn.py` 直接解析 G92RWF Fortran unformatted records，计算归一化重叠、范数、平均半径、节点趋势和轨道能量变化；
- `crosscheck_rwfn.py` 按迭代和 `n,κ` 将外部结果与内部 `accepted_metrics` 对齐；
- 同一 Ni/Ca-like AS1 对照匹配了 49 次轨道更新，重叠趋势相关系数为 `0.998322`，半径变化因子趋势相关系数为 `0.980128`；
- 开启逐轮快照与关闭时的最终 `rmcdhf.sum` 完全一致。

新增 `run_matrix.sh`：smoke 矩阵覆盖现有 Ni/Ca-like AS1 的 B0–B6 和 strict SCF；full 矩阵扩展到仓库现有的 Ni I/Ni-Ca-like AS1/AS2 以及 balanced AS2 的 MPI 1/2/4 对照。runner 会自动执行收敛语义检查；开启 `GRASP_TRACE_RWFN=1` 时还会执行外部波函数趋势交叉验证。

随后补齐 B5/B6 的默认关闭诊断开关并加入 smoke 矩阵。Ni/Ca-like AS1 的 B5 在第 3 轮产生非有限加权能量，证明该输入不能稳定使用轮末正交化；`SCFmpi` 现会在加权能量首次变为非有限值时明确终止，避免继续空转至 100 轮。B6 全程保持 METHOD=3、无 fallback，7 轮退出且最终能量与 B3 完全一致，因此当前数据不支持把 METHOD fallback 作为该算例的根因。

### 5.2 Cl I 数据实施与验证

`test/data/Cl_I` 加入后，runner 已支持 Cl I optimized/no-varied、AS1–AS5、半整数 J 解析和链式前一阶段波函数。小型、可追踪的 `fixtures/cl_isodata` 由归档 `rmcdhf.sum` 中的 Z=17、核质量和 Fermi 参数重建；用当前程序重跑 B0/B1 可逐能级复现归档结果。

Cl I AS1 结果为：B0 正确顺序、间隔 `923.811587 cm-1`；B1 单侧正伙伴优化倒序 `-2479.507866 cm-1`；B2 负伙伴单侧优化恢复顺序但扩大为 `3788.603393 cm-1`；B3 成对优化恢复正确顺序和 `932.188187 cm-1`；B4 固定 `-0.5` 阻尼为 `932.184017 cm-1`，最小已接受重叠由 B3 的 `0.558525` 提升到 `0.894903`。B5 在第 2 轮产生非有限能量并明确失败；B6 与 B3 完全一致且无 fallback；strict SCF 从 12 轮延长到 35 轮，间隔为 `932.190382 cm-1`。

B4 的 105 个已接受更新经外部 RWFN 工具交叉验证，重叠趋势相关系数 `0.996964`，半径变化因子相关系数 `0.989158`。串行 B3 与 MPI B3 的结果逐字节一致；MPI 1/2/4 rank 的 B4 结果也逐字节一致，且没有 rank 摘要差异。

B8 的同状态集合权重对照也已完成：统计权重 B4 为 `932.184017 cm-1`，对称等权为 `931.497237 cm-1`，两者均在 21 轮收敛且保持正确顺序。约 `-0.686780 cm-1` 的变化远小于 B1 倒序量级，因此当前 EOL 权重选择不能解释单侧优化缺陷；不同状态集合仍属于后续 B8 扩展。

使用每阶段新生成的成对阻尼波函数串联 AS1–AS5 后，全部保持正确的 `J3/2 < J1/2`，间隔依次为 `932.184017`、`930.990273`、`931.119697`、`931.148075`、`931.094852 cm-1`。全部阶段无 fallback、无 rank 摘要差异；但 AS2 起仍有节点计数波动，因此谱顺序验收通过，轨道形状最终稳定性尚不能宣称完全通过。

### 5.3 2026-08-25 干净构建与 smoke 回归

使用 `/tmp/rmcdhf-build-debug` 重新配置 Debug 构建，GNU Fortran 15.2.1、
OpenMPI 和 FlexiBLAS 环境下 `rmcdhf_mpi` 编译、链接成功。CTest
`lib9290_quad` 1/1 通过，测试驱动脚本通过 `bash -n` 和 `git diff --check`。

Ni/Ca-like AS1 smoke profile 完整运行 B0–B6 及 strict SCF。B0、B1、B2、
B3、B4、B6 和 strict 运行退出码均为 0；B5 退出码为 1，并被
runner 按已知的延迟正交化非有限能量路径识别为预期失败。最终
`matrix_status.csv` 记录 `profile=smoke,status=complete`。legacy 收敛在第 7 轮停止，
strict 收敛在第 13 轮停止。

此次回归首先发现一个非数值性的 runner 缺陷：B0 计算成功后，
`rsave` 已重命名 `rmcdhf.sum`，导致归档比较阶段因旧路径不存在而失败。
提交 `71954ca` 修复该问题后，重跑的整个 smoke 矩阵通过，
`archive_comparison.csv` 与矩阵级汇总均正常生成。该修复不改变 RMCDHF 数值路径。

### 5.4 B4 固定阻尼全矩阵

`run_matrix.sh` 新增独立 `damping` profile，自动运行 Cl I AS1、Ni/Ca-like AS2
和 Ni I AS2 在 `ODAMP=-0.2/-0.5/-0.8` 下的九个成对优化算例。
新增 `summarize_damping.py`，从每个 `orbopt_trace.csv` 和 `rmcdhf.sum`
统一汇总轮数、原始/接受轨道指标、节点变化、fallback、谱序和相对间隔。

2026-08-25 完成了该矩阵。所有算例退出码为 0、
通过 legacy 收敛检查、保持预期精细结构顺序，且没有 MPI rank 摘要差异。
`-0.2/-0.5/-0.8` 的轮数分别为：Cl I `15/21/47`，Ni/Ca-like `10/17/40`，
Ni I `18/29/47`。对应的接受后最小重叠为：Cl I `0.704/0.895/0.986`，
Ni/Ca-like `0.410/0.773/0.974`，Ni I `0.269/0.717/0.971`。

该矩阵表明强阻尼可以稳定实际接受轨道，但收敛成本显著增加。Ni I 原始候选的
最小重叠仍约 `0.028`，说明固定阻尼没有修复 `SOLVE` 候选本身。因此 `-0.5`
作为当前诊断平衡值，`-0.8` 只作强稳定性对照，两者均不设为默认。

阻尼矩阵完成后，又按组态、项、宇称和 $J$ 与
`data/nist_levels/Cl_I.csv`、`Ni_I.csv` 和 `Ni_Ca-like.csv` 进行了对比。
使用的 NIST 参考间隔为 Cl I $^2P_{1/2}$ `882.3515 cm⁻¹`，
Ni I $^3F_{2,3}$ `2216.550/1332.164 cm⁻¹`，Ni IX $^3F_{3,4}$
`1880/4070 cm⁻¹`。

`-0.2/-0.5/-0.8` 三档下，Cl I 误差均约 `+49.83 cm⁻¹`；
Ni I $J=2/3$ 误差约为 `-201/-121 cm⁻¹`；Ni IX $J=3/4$
误差约为 `+10/+119 cm⁻¹`。三档阻尼之间的 NIST 误差差异很小，
证明增强阻尼主要影响轨道更新稳定性，不会显著修正最终间隔。

### 5.5 三轮重复与收敛校验器兼容

2026-08-28 按服务器 `Cl_I/mcdhfmpi.sh` 的 Slurm 头部规范新增
`test/rmcdhf_orbopt/run_repro_sbatch.sh`，并以作业 491 运行 Ni/Ca-like AS2
的 B0、B1、B3、B8。每个变体分别用 MPI 1、2、4 完成三次计算；12 个运行
全部退出码为 0，四个变体在三种 MPI 配置下的 `rmcdhf.sum` 均逐字节一致。
完整输出归档于 `test/data/results/repro_sbatch_20260828/`。

同时更新 `check_strict_scf.py`，兼容新版只输出基础收敛字段的轨迹，并在检查时
推导 legacy/strict 收敛字段。现有复现轨迹均已通过 legacy 检查；这项修复只影响
诊断工具，不改变 `rmcdhf_mpi` 数值路径。

与 no-varied 基线比较，Ni IX 间隔精度明显改善；Ni I 和 Cl I 虽保持了
正确谱序，但 NIST 间隔误差仍大于各自的 no-varied 结果。因此实施验收中
必须分开报告谱序、轨道稳定性和 NIST 间隔误差，不能用其中一项代替另一项。

### 5.6 显式 48 核绑定与三轮重复性

亲和性检查发现，原来的“4 MPI ranks × 12 threads”只设置了
线程数，OpenMPI 默认将每个 rank 绑定到单核，实际只使用了 4 核。
`run_data_case.sh` 现在于线程数大于 1 时构造：

```text
mpirun --map-by slot:PE=<threads> --bind-to core -n <ranks>
```

同时设置 `OMP_PLACES=cores` 和 `OMP_PROC_BIND=true`。在 48 物理核
主机上，默认重复性配置为 4 ranks × 12 cores/rank，四个 rank
的核集互不重叠。

修正后在同一 48 核配置下运行三轮九算例阻尼矩阵。27/27
个 RMCDHF 运行成功，三个矩阵均完成，无 rank 摘要差异。第 2、3 轮
相对第 1 轮的 18 项比较全部匹配，最大浮点摘要差为 0，迭代次数、
节点/fallback 计数和谱序也完全一致。精简结果已保存在
`test/rmcdhf_orbopt/results/damping_repeatability_20260825/`。

因此，“每个阻尼变体至少三次”已列入完成项。原有的性能对比因
缺少显式 PE 绑定而不再作为“4×12 最快”的证据；若要选择最佳并行
配置，需要在同一显式绑定政策下重新计时。

## 6. 尚未列入“已完成”的项目

为防止将原型误写为最终修复，以下项目明确不属于已完成范围：

- B5/B6 及 B8 等权/统计权重对照已实施；B7 固定轨道 CI/RCI runner 与 Cl I 单例对照已实施，但 Ni I、Ni/Ca-like 多体系矩阵及 B8 不同状态集合尚未实施；
- 串行与 MPI 的 Cl I AS1 最终结果及 MPI 1/2/4 已一致，但所有算例的逐轮串行/MPI 全矩阵尚未完成；
- `run_matrix.sh` 已覆盖 Ni 与 Cl I B0–B6，但尚未作为长耗时 CTest 默认执行；
- B6 strict fallback 已实现；稳定 `ORTHY` 顺序、节点验收和真正的伙伴联立更新尚未实施；
- 核极化势 NP0–NP3 尚未开始，符合“主线验收后再做”的阶段顺序。

## 7. 当前可交付结论

已完成的代码足以回答 varied list 是否平衡、异常轨道在哪一轮产生、候选经阻尼后是否稳定、是否发生 fallback，以及不同 rank 的关键数组是否一致。

当前交付物应定义为“RMCDHF MPI 轨道优化诊断与实验性护栏版”，不应定义为“已通过 Cl I/Ni-Ca-like 全部验收的生产修复版”。
