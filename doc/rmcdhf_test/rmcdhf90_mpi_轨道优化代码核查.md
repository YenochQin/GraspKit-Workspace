# `rmcdhf90_mpi` 当前代码核查

## 1. 范围与结论

本页核查当前
[`rmcdhf_test/src/appl/rmcdhf90_mpi`](../../rmcdhf_test/src/appl/rmcdhf90_mpi)
及其实际链接的公共例程，并与 `/home/workstation2/AppFiles/grasp_raw/src` 原版作静态
对照。核查日期更新为 2026-09-16；partner transaction 结论对应当前未提交工作树。

结论：当前工作树已在 legacy 顺序逐轨道路径之外实现独立的 `nl-/nl` 原子更新事务。
`GRASP_REQUIRE_BALANCED_PAIR` 仍只表示输入完整性；只有
`GRASP_PAIR_TRANSACTION=1` 才启用共同前态、共同阻尼、共同验收和组级 rollback。
这两个 capability 在控制输出和 trace 中保持独立。

第 9 节是对本次改动的独立静态复核：`improvmpi.f90` 新增的 `ED1=PED(J)`
在 legacy 提交路径下恒读到 0（MPI 版 `DAMPCK` 从未写回 `PED`，与串行版是两套不同
公式），`defcor.f90` 移除的一次性锁存经全树检索确认对当前代码库无可观测影响。两项
均不影响第 4、8 节的事务不变量结论，但会改变对 3.1 节两行改动"作用"的理解。

## 2. 当前调用路径

`GETOLDmpi` 解析 varied list 后执行：

```text
APPLY_FIXED_ORBITAL_DAMPING
CHECK_RELATIVISTIC_PAIRS
TRACE_ORBITAL_SELECTION
```

`CHECK_RELATIVISTIC_PAIRS` 按同一 $n,l$ 查找 $\kappa=l$ 与 $\kappa=-(l+1)$，比较两侧
`LFIX`；严格开关开启时，缺失伙伴或只变一侧会终止。这里没有建立供 SCF 调度的 group。

`SCFmpi` 默认关闭事务时仍执行：

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

`IMPROVmpi(J)` 在一次调用内完成 `COFPOTmpi -> DEFCOR -> SOLVE -> CONSIS -> normalize
-> DAMPCK/DAMPOR -> ORTHY`。`DAMPOR` 已把候选写回 `PF/QF` 后，下一个伙伴才开始计算；
`ORTHY` 还可能重建当前轨道的整个同 $\kappa$ 子空间。因此第二个伙伴看到的已不是组前
状态。

事务开启时改为按 rank 0 广播的确定性 group 表调度。每个成员前恢复同一完整快照，
调用 prepare-only `IMPROVmpi` 后复制 raw candidate；全部候选准备完成才施加一个共同
阻尼、临时共同写回、执行 post-damp 和 post-`ORTHY` 检查。任一失败则完整恢复并在
同一个 snapshot/schedule 上立即重试。`NSIC/MAXARR` 选中单轨道后也映射回完整 group。

## 3. 相对原版的有效差异

### 3.1 无条件数值/并行正确性修改

| 位置 | 当前修改 | 作用 | 与精细结构问题的证据关系 |
|---|---|---|---|
| `setlagmpi.f90` | 恢复不同 generalized occupation 时的 QDIF/`OBQDIF` 分支 | 与串行和 mem-MPI 公式一致 | 单因素回归未消除剩余偏差 |
| `improvmpi.f90` | `ED1=PED(J)` | 意图恢复前态能量；见 9.1——MPI 版 `DAMPCK` 从不写回 `PED`，legacy 路径下该行恒读 0 | 必要初始化，不是唯一根因；9.1 表明该行本身在 legacy 路径下不产生预期效果 |
| `improvmpi.f90` | `MPI_Gatherv` 按各 rank 实际 `NDCOF` 收集，并令 `NDCOF=i_last` | 避免未初始化尾部与 unique union 截断 | 单因素结果未改变目标间隔 |
| `defcor.f90` | 移除 `DATA FIRST/.TRUE./` 锁存，`DP(:2)/DQ(:2)` 改为逐次清零 | 见 9.2——全树检索确认 `DP/DQ` 只由本文件写入，锁存对现有代码库无可观测影响 | 静态验证为等价简化，非缺陷修复 |
| `iniestmpi.f90` | 稀疏 offset 沿本 rank 所有列推进 | 匹配 rank-local packed storage | MPI 正确性修复 |
| `spicmvmpi.f90` | `IBEG` 沿本 rank packed columns 推进 | 避免把全局前一列端点当本地 offset | 46-rank 初始 CI 恢复一致 |
| `parameter_def_M.f90` | `NNNP 590 -> 2990` | 扩大径向网格数组 | 构建变体，不能与 590-grid 基线混比 |

`count.f90` 与 `coun_C.f90` 只增加按环境开关的节点诊断上下文，不改变默认节点算法。

### 3.2 可选诊断和安全路径

新增模块：

- `orbopt_control.f90`：运行时开关、阈值、目标态和拒绝计数；
- `orbopt_metrics.f90`：范数、旧/新轨道 overlap、平均半径和节点指标；
- `orbopt_trace.f90`：选择、候选、`ORTHY`、SCF、MPI 摘要和目标态 CSV；
- `orbopt_round_state.f90`：整轮 `PF/QF`、CI 向量和标量快照、CI 系数 assignment、
  回退及 `.mix` 重写。

主要可选行为包括 fixed `ODAMP`、候选轨道拒绝、post-damping 检查、节点进度、strict
`METHOD=3`、strict SCF、整轮身份/能序回退和固定参考系数代理。大部分默认关闭；
non-finite weighted energy 会无条件停止。

这些路径能观测或拒绝坏候选。Job 645 的全部整轮都被接受，所以它们不能解释该作业的
正确能序；AS 级 fallback 也不能作为优化成功。

## 4. 伙伴事务不变量的当前状态

| 必要不变量 | 当前状态 |
|---|---|
| 确定性 partner-group 表供 SCF 调度 | 已满足；rank 0 构造并广播，顺序由 `IORDER` 首次出现决定 |
| 两个候选从相同 `PF/QF/E` 前态生成 | 已满足；每个 member 前恢复同一完整快照 |
| 候选生成与正式提交分离 | 已满足；`IMPROVmpi(PREPARE_ONLY)` 只返回 raw candidate |
| 双侧共用阻尼 | 已满足；取双方建议中更保守的旧轨道保留比例，retry 下限为 0.5/0.7/0.9 |
| 双侧同时接受或同时拒绝 | 已满足；raw、post-damp、post-`ORTHY` 任一失败即整组 rollback |
| post-`ORTHY` 失败恢复整个伙伴组 | 已满足；完整 `PF/QF` 和相关工作/标量状态恢复并断言 |
| `NSIC/MAXARR` 按组补充更新 | 已满足；选中 `J` 后通过映射调度整个 group |
| trace 证明 group 原子提交 | 已满足；固定 93 列记录 group/snapshot/schedule/phase/retry/decision |

历史 B3 和 Job 645 仍只能描述为顺序逐轨道更新；只有开启独立 transaction capability
的新运行可以称为 partner-group 原子提交。

## 5. 已实现的代码边界

### 5.1 group 描述与候选记录

`orbopt_pair_types.f90` 提供独立 candidate 类型和无 MPI 依赖的 group 构造逻辑；
`orbopt_pair_transaction.f90` 持有 group 表、完整快照、候选缓冲和事务状态。核心记录为：

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

rank 0 从 `NP/NAK/LFIX/IORDER` 构造 group 后广播。$s_{1/2}$ 是单成员组；$l>0$
缺失、重复或 fixed partner 在 SCF 前报错。group 表不依赖字符串标签。

### 5.2 将 `IMPROVmpi` 分成两阶段

- `PREPARE_PAIR_CANDIDATE`：恢复组快照，调用 prepare-only `IMPROVmpi`，复制候选；
- `IMPROVE_ORBITAL_GROUP`：施加共同阻尼，临时写回所有成员，完成 post-damping 与
  post-`ORTHY` 检查，然后一次 commit 或完整 restore/retry。

`SOLVE` 会修改 `E(J)`，`DAMPCK/DAMPOR` 会交换 `P/Q` 与 `PF/QF`，`ORTHY` 会修改同
$\kappa$ 的多条轨道。当前第一版采用完整轨道和工作数组快照，优先保证恢复语义；尚未
按性能测量缩小 snapshot 范围。

### 5.3 修改 `SCFmpi` 调度

首轮按已广播 group 表遍历；`NSIC` 路径将 `MAXARR` 选出的轨道映射到 group 后调用
`IMPROVE_ORBITAL_GROUP`。共同阻尼并入 `DAMPMX`，失败重试计数保存在 group 层。

## 6. 数值和 MPI 决策规则

1. 两个候选均从组快照恢复后的同一状态计算；本轮 `SETLAGmpi` 在组内保持不变。
2. 共同阻尼采用更保守一侧的建议，明确记录从建议值到最终值的规则。
3. raw、post-damp、post-`ORTHY` 三个阶段任一成员失败，group 即失败。
4. group 回退恢复 `PF/QF/E/MF/PZ/SCNSTY/ODAMP/METHOD/NSIC` 及 `ORTHY` 可能影响的轨道。
5. 所有 rank 对失败标志归约，rank 0 广播唯一决定；提交后比较 rank 摘要。
6. 超过组级重试上限后使用 `MPI_ABORT` 或现有统一 MPI 终止路径，禁止部分 rank 继续。

## 7. 必须保留但不能误解的护栏

- fixed-TF/RCI anchor、身份/能序/径向检查、`.rwf/.mix` 回退和下一 AS hash 验证保留；
- CI 系数 overlap 在轨道基组改变后只是 same-CSF proxy，不是严格多电子 ASF overlap；
- round rollback 保护一次 SCF round，不能替代 partner-group 原子提交；
- `GRASP_REQUIRE_BALANCED_PAIR` 继续作为输入闸门，并新增独立
  `pair_transaction_enabled` capability；
- rejected/fallback 状态可作恢复输入，但报告中必须标为优化失败。

## 8. 验证状态与完成条件

目标构建和 CTest 4/4 已通过；Ni I、Ni/Ca-like、Cl I AS1 的 transaction-on 回归均有
`rmcdhf.exitcode=0`。Cl I 1/2/4/46 rank 的决定序列相同且最终能量按 CSV 精度一致；
post-`ORTHY` 故障恢复、0.5/0.7 重试上限、`MPI_Abort(92)`、默认关闭和
`NSIC/MAXARR` group 映射均已有 trace 证据。详见
[测试结果](./rmcdhf90_mpi_轨道优化测试结果.md) 第 8 节。

代码层第 4 节当前均有源码与 trace 证据。仍需继续处理构建中遗留的隐式 MPI 跨调用
类型告警。物理层完成要求另见
[实施方案](./rmcdhf90_mpi_轨道优化调试方案.md)：每个 AS 必须由直接优化候选通过 NIST
和 fixed-TF 非劣性验收；护栏回退不计成功。

## 9. 独立复核（2026-09-16，以静态阅读为主，未重新执行物理回归）

本节是对第 3–8 节结论的独立复核，方法为逐文件对照 `grasp_2990_NNNP`（未修改的
GRASP2018 上游副本，仅带 `NNNP` 补丁）与当前 `rmcdhf_test` 工作树，并手工跟踪调用链；
未重新编译或运行 MPI 二进制，也未重新执行第 8 节列出的回归目录。唯一重新执行的是
无 MPI 依赖的单元测试：在既有 `build/` 树中执行
`cmake --build . --target test.rmcdhf_pair_transaction_logic` 后
`ctest -R rmcdhf_pair_transaction_logic` 显示 `Passed`（"passed pair-transaction
logic tests"），与第 8 节"CTest 4/4"的说法一致，并独立确认了下文对
`BUILD_ORBITAL_GROUP_TABLE`/`SUGGEST_ORBITAL_DAMPING`/`RETRY_PAIR_DAMPING` 的手工
跟踪没有偏离其实际运行行为。除 9.1、9.2 外，
以下条目复核为准确，与文档原述一致：`BUILD_ORBITAL_GROUP_TABLE` 的 `IORDER` 置换
校验、`(n,κ)` 唯一性检查、$s_{1/2}$ 单成员规则与缺失/fixed 伙伴拒绝，与
`test/rmcdhf_orbopt/pair_transaction_logic.f90` 逐条对照一致；
`orbopt_pair_transaction.f90` 中 `SHARED_DAMPING = MAX(SHARED_DAMPING,
CANDIDATES(MEMBER_POSITION)%DAMPING_SUGGESTED)` 确认按"更大阻尼即更保守"取双方
建议中较大者；`scfmpi.f90` 里 `PAIR_TRANSACTION_ENABLED` 分支按 `PAIR_GROUP_COUNT`
遍历 group，与 legacy 逐轨道遍历在**组间**保持相同的顺序式（Gauss-Seidel）语义——
组之间仍是前一组已提交状态影响后一组的快照，只有**组内**两个成员改为共享前态；
`orbopt_trace.f90` 的 `TRACE_FIELD_COUNT=93` 与第 8 节"固定 93 列"的说法一致
（其中字段 88 在 `event='control'` 的一次性设置行里复用为 `MAX_PAIR_RETRIES`，
在 `pair_member`/`pair_decision` 行里是真实 `pair_retry_count`——按 `event` 过滤后
不冲突，非缺陷）；新增源文件在 `CMakeLists.txt`/`Makefile`/`test/CMakeLists.txt`
中的依赖顺序和链接目标均正确；`solve.f90`、`orthy.f90`、`newcompi.f90` 相对上游
的差异均为纯 trace 埋点（`TRACE_ORTHY_ORDER/PROJECTION`、`TRACE_LEVEL_ENERGY`、
`COUNT_CONTEXT` 赋值），未改变各自原有算法；`getoldmpi.f90` 中
`APPLY_FIXED_ORBITAL_DAMPING`/`CHECK_RELATIVISTIC_PAIRS`/`TRACE_ORBITAL_SELECTION`
的调用顺序与第 2 节描述逐字匹配；`rscfmpivu.f90` 新增的 `DEFER_ORTHOGONALIZATION`
（置 `ORTHST=.FALSE.`）与 `STRICT_METHOD3`（强制 `METHOD=3`）两个开关默认关闭，且
均已在测试结果文档第 7 节记为已排除的实验方向，未发现与既有结论冲突之处。对
`rmcdhf_test/src` 与 `grasp_2990_NNNP/src` 的全量文件级 diff 确认：本次工作树相对
上游的改动范围仅限于 `rmcdhf90_mpi` 全部文件、`lib9290/count.f90`、
`libmod/coun_C.f90`、`mpi90/{iniestmpi,spicmvmpi}.f90`；`rmcdhf90`、`rmcdhf90_mem`、
`rmcdhf90_mem_mpi` 三个变体未被触碰。这份全量文件级差异之外，没有发现落在第 3.1、
3.2 节描述范围之外的改动来源——`getscdmpi.f90`（新增 `PED` 初始化）、
`improvmpi_I.f90`（接口签名随 `improvmpi.f90` 同步）以及两个 `CMakeLists.txt`/
`Makefile` 的改动都只是已核查修改的必然配套，不构成独立的数值路径变化；表 3.1
本身仍只收录"无条件数值/并行正确性修改"这一子集，不等同于全部被修改文件的清单。

### 9.1 `PED` 阻尼历史：MPI 与串行是两套不同公式，`ED1=PED(J)` 在 legacy 路径下恒空

`improvmpi.f90:142` 新增 `ED1 = PED(J)`，意图与串行 `rmcdhf90/improv.f90:73` 的
同一行取得一致行为。但两者调用的 `DAMPCK` 并非同一算法：

| | 串行 `rmcdhf90/dampck.f90` | MPI `rmcdhf90_mpi/dampck.f90` |
|---|---|---|
| 能量差定义 | 相对值：`ED2=(ED2-E(J))/ED2` | 绝对值：只在 `IPR==J` 分支内 `ED2=ED2-E(J)` |
| 判据 | 无条件比较 `ED1*ED2 < -1.0D-4` | 只有 `IPR==J` 时才比较 `ED1*ED2 > 0.0D0` |
| 系数 | 振荡时 `0.10+0.90*ODAMP`；否则 `0.50*ODAMP` | 反转时 `0.25+0.75*ODAMP`；否则 `0.75*ODAMP` |
| `IPR` 的作用 | 声明为参数，但函数体内从未读写，纯接口遗留 | 决定走哪个分支的门控条件（是否与上次调用同一轨道） |
| `PED(J)` | 每次调用末尾无条件写回 `PED(J)=ED2`（`dampck.f90:52`） | 从未读写；`rmcdhf90_mpi`、`rmcdhf90_mem_mpi` 两个 MPI 变体都没有一处写 `PED` |

两个文件均与 `grasp_2990_NNNP` 上游逐字节一致（`diff` 确认无差异），即这一分歧是
GRASP2018 上游串行与 MPI 可执行文件长期独立演化的结果，不是本仓库本次改动引入的
问题。

由于 `PED(:NW)` 只在 `getscdmpi.f90:197` 初始化为 `0.0D0`，此后在 legacy（非
`GRASP_PAIR_TRANSACTION`）提交路径下没有任何代码写回它，`ED1 = PED(J)` 在这条路径
下**恒等于读到 0.0**。这一行真正改变的是：修改前，`ED1` 是 `DATA ED1/0.D0/` 声明的
本地 SAVE 标量，跨调用保留上一次 `DAMPCK` 调用结束时写入的值；修改后，这份跨调用
记忆在每次调用开始时都被强制清零。唯一可观察差异出现在 `IPR==J` 分支——即同一轨道
被连续两次 `IMPROVmpi`，中间没有任何其它轨道调用（典型场景：`NSIC/MAXARR` 连续
选中同一个最不自洽轨道，或变分轨道列表只有一个轨道）。修改前，该分支比较的是上一次
调用真实保留下来的 `ED1*ED2`；修改后，`0.0` 乘任何值都不满足 `>0.0`，因此**无条件
落入"检测到反转"分支** `ODAMP(J)=0.25+0.75*ODAMP(J)`，而不是保留原有的趋势比较。

结论：`ED1=PED(J)` 没有达到"恢复前态能量"的目的——两套 `DAMPCK` 公式本身不兼容，
单行修改无法弥合；它在唯一会产生效果的窄场景下，把行为从"比较真实趋势"改成
"无条件假定反转"，是否更优并不确定。这与本文档 3.1 节"必要初始化，不是唯一根因"
以及测试结果文档"单因素回归未消除剩余偏差"的经验观察一致——该行改动本来就只有很小
的可观测影响。建议二选一：删除该行（它不产生注释所声称的效果），或保留但把注释改为
准确描述（"MPI 路径下 `PED` 恒为占位符，此行当前无实际效果"），避免以后有人按字面
理解为"已恢复与串行等价的阻尼历史"。

作为对照，pair transaction 路径（`orbopt_pair_types.f90` 的
`SUGGEST_ORBITAL_DAMPING` + `orbopt_pair_transaction.f90:506` 的
`PED(J) = CANDIDATE%PED_PROPOSED`）复刻的是**MPI 版**系数（`0.75`／
`0.25+0.75`、绝对差、`>0.0` 判据——与单元测试中 "adaptive same direction" 和
"adaptive direction reversal" 两个用例核对一致），但改为真正逐轨道持久化 `PED`，
不依赖 `IPR` 门控。这是唯一让该比较分支在 MPI 家族中真正生效的路径，但也意味着
开启 `GRASP_PAIR_TRANSACTION=1` 后，每个轨道每轮都会依据自身历史在"继续衰减阻尼"
和"提高阻尼"之间选择；而 legacy 路径由于 `IPR` 几乎总不等于当前 `J`
（多轨道轮流处理），实际几乎总是走单纯衰减分支。两条路径提出的阻尼建议因此不是
同一算法在"事务化前后"的等价形式，比较 transaction-on/off 数值轨迹时应把这一点
当作已知的方法学差异记录，而不是纯粹的原子性包装。

### 9.2 `defcor.f90` 一次性锁存：改动本身安全，但原写法在当前仓库中并无可观测影响

`defcor.f90` 移除了 `DATA FIRST/.TRUE./` 门控的一次性 `DP(:2)=DQ(:2)=0`
初始化，改为每次调用无条件执行（现第 43–44 行）。对
`rmcdhf_test/src/appl/rmcdhf90_mpi/*.f90` 全量检索 `DP(`/`DQ(` 赋值，确认整个目录
下只有 `defcor.f90` 本身写这两个数组（`setxuv.f90` 只读取 `DP(2:)`/`DQ(2:)` 作为
输入）。因此上游"只清零一次"的写法在单次运行内其实是安全的：这两个格点在第一次
调用后被置零，此后没有任何代码再写它们，不会被后续调用观察到"脏"值。这项改动是
**等价简化**，不是可复现缺陷的修复——静态分析没有找到会触发不同行为的路径。它仍
值得保留：`DP`/`DQ` 现在整体被 `orbopt_pair_transaction.f90` 的
`SAVE_PAIR_SNAPSHOT`/`RESTORE_PAIR_SNAPSHOT` 纳入逐候选快照，一个只在进程生命周期
内触发一次的隐藏 `SAVE` 标志，与"每个候选都可完整回滚"的事务模型放在一起是脆弱但
目前未爆发的耦合；提前移除比日后在别处新增一次对 `DP(1:2)/DQ(1:2)` 的写入时才发现
更安全。

### 9.3 复核范围

本节基于源码静态阅读、`grasp_2990_NNNP` 差异对照与仓库内单元测试交叉核对；未编译或
执行 `rmcdhf_mpi`，未重新运行第 8 节列出的任何回归目录，也未对 `PF`/`QF` 之外的
全部模块级状态（例如 `SOLVE`/`COFPOTmpi`/`ORTHY` 调用链内可能存在的隐藏 `SAVE`
局部变量、`orb_C`/`wave_C`/`scf_C`/`pote_C` 中未列入快照清单的字段）做完整的写入面
复核，这部分留待专项的快照完整性检查补充。

## 10. 独立动态验证补充（2026-09-16，构建/测试复现 + 剩余调用链核对）

本节与第 9 节并行独立完成，补充其明确留空的部分：9.3 未编译或运行 `rmcdhf_mpi`，
本节改为实际重新配置、编译并运行测试；9.3 未做的 `SOLVE`/`COFPOTmpi`/`ORTHY`
调用链快照完整性核对，本节逐一补齐；另外核对第 3.1 节中第 9 节未涉及的两行改动
（`setlagmpi.f90`、`MPI_Gatherv`）。与第 9 节的 `ED1=PED(J)`/`defcor.f90` 结论
（见 9.1、9.2）没有冲突，不重复展开。

### 10.1 构建与 CTest 复现

`rmcdhf_test/build/` 此前只配置到 `test.lib9290_quad`（早于本次 pair transaction
测试写入 `test/CMakeLists.txt`）。重新执行 `cmake .` 后 4 个 CTest 目标全部注册；
分别构建 `test.rmcdhf_pair_transaction_logic`、`test.mpi90_local_sparse`、
`rmcdhf_mpi` 均成功，`ctest` 结果为 `100% tests passed, 0 tests failed out of 4`，
与第 8 节"CTest 4/4"一致。`rmcdhf_mpi` 完整链接成功，只有告警、没有错误；确认
`orbopt_pair_transaction.f90:667`、`:671`（`CONSENSUS_BAD` 中的
`MPI_Allreduce`/`MPI_Bcast`）属于第 8 节已提及的"隐式 MPI 跨调用类型告警"：
`mpi_C` 未对这些泛型调用提供按实参类型区分的显式接口，`improvmpi.f90`、
`matrixmpi.f90`、`setcslmpi.f90`、`getoldmpi.f90` 等既有文件早已有同类告警。新文件
延续而非修复这一既存缺口，构建输出未见新增错误级问题。

### 10.2 `setlagmpi.f90` 与 `MPI_Gatherv` 两行的上游对照

第 9 节复核了 `ED1=PED(J)` 与 `defcor.f90`，第 3.1 节其余两行改动核对如下：

- `setlagmpi.f90` 的 QDIF/`OBQDIF` 分支：对照
  `grasp_2990_NNNP/src/appl/rmcdhf90/setlag.f90:226-238` 与
  `.../rmcdhf90_mem_mpi/setlagmpi.f90:263-274`，两者均已有该分支；原
  `rmcdhf90_mpi/setlagmpi.f90` 之前只有 `OBQSUM` 分支。"与串行和 mem-MPI 公式一致"
  的表述属实——这一行与 `ED1=PED(J)` 不同：三个变体共用同一 `SETLAG`/`SETLAGmpi`
  乘子公式，不存在 9.1 所述"两套不同公式"的问题。
- `improvmpi.f90` 的 `MPI_Gatherv` 改动：原 `MPI_GATHER` 以各 rank 最大值
  `ndcof_max` 为定长发送计数，当某 rank 实际 `ndcof<ndcof_max` 时会读出该 rank
  本地 `nda`/`da` 数组中未初始化的尾部；随后又用 `NDCOF=ndcof_max` 截断合并结果，
  当不同 rank 持有大致不相交的角系数标签子集、其并集长度超过任一 rank 本地长度时，
  会丢弃并集中超出 `ndcof_max` 的有效系数。改为 `MPI_Gatherv` 按各 rank 真实
  `ndcof` 发送、用前缀和 `ndcof_disp` 定位、`NDCOF=i_last`（实际合并的唯一系数数）
  后，逐行重新推导确认三处改动逻辑自洽，是与"某次回归是否观察到目标间隔变化"无关
  的真实正确性修复。

### 10.3 快照完整性：补齐 9.3 留空的调用链核对

逐一阅读 `PREPARE_PAIR_CANDIDATE`/`APPLY_SHARED_DAMPING`/`ORTHY` 实际触达的全部
子例程全文——`COFPOTmpi`、`DEFCOR`、`SOLVE`（368 行）、`CONSIS`、`DAMPOR`、
`RINT`、`QUAD`、`ORTHY`（235 行）——列出每一处对模块级状态的写入，并与
`SAVE_PAIR_SNAPSHOT`/`RESTORE_PAIR_SNAPSHOT` 覆盖的字段比对：

| 子例程 | 写入的模块级字段 | 已被快照覆盖 |
|---|---|---|
| `COFPOTmpi` | `YP,XP,XQ`（`pote_C`） | 是 |
| `DEFCOR` | `DP,DQ`（`def_C`） | 是 |
| `CONSIS` | `SCNSTY(J)`（`scf_C`）；其本地 `MTP` 与模块级 `tatb_C::MTP` 同名但非同一变量 | 是 |
| `SOLVE` | `E,EPSMIN,EPSMAX,EMAX,P0,P,Q,MTP0,MTP,TA`；经 `SETPOT`/`SETXUV`/`SETXZ`/`SETXV` 写入 `TF,TG,XU,XV`（按变量分工与注释确认，这些被调用子例程本次未改动） | 是 |
| `DAMPOR` | `PZ,MTP0,MF,PF,QF,P,Q` | 是 |
| `RINT`（`DAMPOR`/`ORTHY` 均调用） | `MTP,TA` | 是 |
| `QUAD` | `TA`（含把 `TA(MTP+1:MTP+3)` 尾部清零；`TA_SNAPSHOT` 按 `SIZE(TA)` 声明，覆盖该越界余量） | 是 |
| `ORTHY` | 对同一 `NAK` 的**全部**非固定轨道（不只两个伙伴成员）写入 `PZ,PF,QF,MF`；快照按全部轨道维度保存这些字段（如 `PF_SNAPSHOT(NNNP,NW)`），天然覆盖。本地 `MTP0/MTP` 与模块级同名变量不是同一变量 | 是 |

未发现被写入但未被快照覆盖的模块级字段；`SAVE_PAIR_SNAPSHOT`/
`RESTORE_PAIR_SNAPSHOT` 对 5.2 节所述"完整轨道和工作数组快照"的实现是完备的。
此结论仅覆盖 `METHOD<=2` 时 `SOLVE` 主体直接可见的写入目标，未逐条重新推导
`IN/OUT/START/SETXUV/SETXZ/SETXV/ESTIM/NEWE` 内部的写入面（这些子例程签名与上游
完全一致、本次未改动）。

### 10.4 变更面与集成路径核实

`diff -rq` 逐目录对照 `rmcdhf_test/src` 与 `grasp_2990_NNNP/src`，确认 `rmcdhf90`、
`rmcdhf90_mem`、`rmcdhf90_mem_mpi`、`libdvd90` 完全未改动；改动只出现在
`rmcdhf90_mpi` 全目录，以及 `lib9290/count.f90`、`libmod/coun_C.f90`、
`mpi90/{iniestmpi,spicmvmpi}.f90`，与第 1 节声明的核查范围一致，未发现范围外的
未记录改动。

`scfmpi.f90` 中 `PAIR_TRANSACTION_ENABLED` 分支下，首轮遍历与 `NSIC/MAXARR` 补充轮
都通过 `GROUP_FOR_ORBITAL` 把选中轨道映射回整组，映射失败 `ERROR STOP`，不会静默
退回单轨道更新；`INITIALIZE_PAIR_TRANSACTION` 只在进入 `NIT` 循环前调用一次，
group 表在同一次 SCF 调用内不再重建，与 `LFIX/IORDER/NP/NAK` 同一次调用内不变的
前提相符。`orbopt_control.f90` 新增 `PAIR_TRANSACTION_ENABLED → REQUIRE_BALANCED_PAIR`
单向强制、以及全部新增开关（`MAX_PAIR_RETRIES`、`PAIR_FAULT_*`）的 rank 0 读取与
`MPI_Bcast` 广播逐项核对，未发现遗漏广播的字段。
