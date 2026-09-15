# `rmcdhf90_mpi` 当前代码核查

## 1. 范围与结论

本页核查当前
[`rmcdhf_test/src/appl/rmcdhf90_mpi`](../../rmcdhf_test/src/appl/rmcdhf90_mpi)
及其实际链接的公共例程，并与 `/home/workstation2/AppFiles/grasp_raw/src` 原版作静态
对照。核查日期为 2026-09-15。

结论：当前分支已包含若干 MPI/数值正确性修复、诊断与生产安全机制，但核心更新仍是
Fischer Algorithm 5.1 的**顺序逐轨道更新**。`GRASP_REQUIRE_BALANCED_PAIR` 只验证输入；
没有 `nl-/nl` 原子更新组。文档和 capability 不得把“伙伴完整”表述为“伙伴联合求解”。

## 2. 当前调用路径

`GETOLDmpi` 解析 varied list 后执行：

```text
APPLY_FIXED_ORBITAL_DAMPING
CHECK_RELATIVISTIC_PAIRS
TRACE_ORBITAL_SELECTION
```

`CHECK_RELATIVISTIC_PAIRS` 按同一 $n,l$ 查找 $\kappa=l$ 与 $\kappa=-(l+1)$，比较两侧
`LFIX`；严格开关开启时，缺失伙伴或只变一侧会终止。这里没有建立供 SCF 调度的 group。

`SCFmpi` 每轮仍执行：

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

## 3. 相对原版的有效差异

### 3.1 无条件数值/并行正确性修改

| 位置 | 当前修改 | 作用 | 与精细结构问题的证据关系 |
|---|---|---|---|
| `setlagmpi.f90` | 恢复不同 generalized occupation 时的 QDIF/`OBQDIF` 分支 | 与串行和 mem-MPI 公式一致 | 单因素回归未消除剩余偏差 |
| `improvmpi.f90` | `ED1=PED(J)` | 恢复阻尼/重试需要的前态能量 | 必要初始化，不是唯一根因 |
| `improvmpi.f90` | `MPI_Gatherv` 按各 rank 实际 `NDCOF` 收集，并令 `NDCOF=i_last` | 避免未初始化尾部与 unique union 截断 | 单因素结果未改变目标间隔 |
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

## 4. 伙伴事务缺口

| 必要不变量 | 当前状态 |
|---|---|
| 确定性 partner-group 表供 SCF 调度 | 无；只有启动检查 |
| 两个候选从相同 `PF/QF/E` 前态生成 | 否；第一个先写回 |
| 候选生成与正式提交分离 | 否；`IMPROVmpi` 一次完成 |
| 双侧共用阻尼 | 否；各自 `ODAMP(J)` / `DAMPCK` |
| 双侧同时接受或同时拒绝 | 否；单轨道 guard 独立返回 |
| post-`ORTHY` 失败恢复整个伙伴组 | 否；只有单轨道/整 SCF round 两种粒度 |
| `NSIC/MAXARR` 按组补充更新 | 否；仍选择单个 `J` |
| trace 证明 group 原子提交 | 无相应 schema/capability |

所以 B3 和 Job 645 的准确描述是“两个伙伴都列入 varied list 后，按 `IORDER` 依次
更新”；不能写成“联合阻尼”或“成对求解”。

## 5. 建议代码边界

### 5.1 group 描述与候选记录

新增独立模块（名称可调整）：

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

rank 0 从 `NP/NAK/LFIX/IORDER` 构造 group 后广播。$s_{1/2}$ 是单成员组；其他缺失伙伴
在生产模式报错。group 表构造不应依赖字符串标签。

### 5.2 将 `IMPROVmpi` 分成两阶段

- `PREPARE_ORBITAL_CANDIDATE(J, SNAPSHOT, CANDIDATE)`：生成候选并填充记录，但不写
  正式 `PF/QF`、不清空组级拒绝计数、不调用 `ORTHY`；
- `COMMIT_ORBITAL_GROUP(GROUP, CANDIDATES)`：施加共同阻尼，临时写回所有成员，完成
  post-damping 与 post-`ORTHY` 检查，然后一次 commit 或完整 restore。

`SOLVE` 会修改 `E(J)`，`DAMPCK/DAMPOR` 会交换 `P/Q` 与 `PF/QF`，`ORTHY` 会修改同
$\kappa$ 的多条轨道。第一版应采用完整轨道快照，先保证语义正确，再按测量结果缩小
snapshot 范围。

### 5.3 修改 `SCFmpi` 调度

首轮 `IORDER` 遍历与 `NSIC` 路径都调用 `IMPROVE_ORBITAL_GROUP`。`MAXARR` 选出一个
轨道后必须映射到其 group，并将整组作为一次补充更新。`DAMPMX`、拒绝次数和收敛状态
也按 group 汇总，避免一侧已经收敛就绕开另一侧。

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

## 8. 验证与完成条件

修改后至少运行目标级构建、Fortran/状态逻辑测试、partner transaction 测试和 Ni I、
Ni/Ca-like、Cl I 数据回归。MPI/Fortran 环境需先加载 `mpi/openmpi-x86_64`；Python
分析使用项目现有环境。

代码层完成要求：第 4 节全部由源码与 trace 证明“已满足”，失败注入测试通过，关闭
开关保持当前已修正 legacy 结果。物理层完成要求另见
[实施方案](./rmcdhf90_mpi_轨道优化调试方案.md)：每个 AS 必须由直接优化候选通过 NIST
和 fixed-TF 非劣性验收；护栏回退不计成功。
