# `rmcdhf90_mpi` 轨道优化证据汇总

## 1. 结论

当前正确能序的主要经验证原因是：**不继承已被错误单侧优化污染的 AS 基线，并让同一
$(n,l)$ 的 `nl-/nl` 都进入 varied list，从而消除原始正号 `nl` 单侧松弛造成的 $J$
依赖差分能量失衡。**

“与 NIST 差异不大”必须限定为：balanced 路径把错误单侧结果的数千至上万
`cm^-1` 偏差恢复到几十或数百 `cm^-1` 的合理量级。现有证据没有证明 balanced 本身
普遍提高 NIST 精度；Ni I Job 645 和 Cl I balanced 结果仍略劣于各自 fixed-TF/
no-varied 基线。

截至 2026-09-16，partner transaction 已在当前未提交工作树中实现并通过程序级回归：
伙伴候选从同一快照生成、使用共同阻尼、共同通过 post-`ORTHY` 检查后原子提交，失败时
整个组恢复并在同一快照上重试。该结论只补齐了算法语义证据；尚未替代逐 AS 物理验收。

## 2. B1/B2/B3：伙伴选择的单因素证据

B1 只选择正号 `nl`，B2 只选择 `nl-`，B3 同时选择两侧；同一行内其余输入保持一致。

| 体系/变体 | SCF 轮数 | 最小候选 overlap | 最大半径因子 | 最终顺序或间隔 |
|---|---:|---:|---:|---|
| Ni I NV | 1 | -- | -- | `J4<J3<J2` |
| Ni I B1 | 20 | 0.02920 | 8.610 | `J2<J3<J4` |
| Ni I B2 | 12 | 0.03570 | 8.416 | `J4<J3<J2` |
| Ni I B3 | 14 | 0.02902 | 8.630 | `J4<J3<J2` |
| Ni/Ca-like NV | 1 | -- | -- | `J2<J3<J4` |
| Ni/Ca-like B1 | 6 | 0.19329 | 2.781 | `J2<J4<J3` |
| Ni/Ca-like B2 | 7 | 0.29918 | 2.784 | `J2<J3<J4` |
| Ni/Ca-like B3 | 8 | 0.18857 | 2.798 | `J2<J3<J4` |
| Cl I B0/no-varied | 1 | -- | -- | `J3/2<J1/2`，`+923.8116 cm^-1` |
| Cl I B1 | 8 | 0.616734 | -- | 倒序，`-2479.5079 cm^-1` |
| Cl I B2 | 8 | 0.616734 | -- | `+3788.6034 cm^-1` |
| Cl I B3 | 12 | 0.558525 | -- | `+932.1882 cm^-1` |

能支持的结论：原始正号单侧选择在 Ni I、Ni/Ca-like、Cl I 中因果性地改变能序；B3
恢复预期顺序。B2 也可正确，所以不能推广成“所有单侧都必错”。Fe I 的一条单侧路径也
保持 `^5D_J` 顺序，进一步限制了结论强度。

B3 并未消除 Ni I 的低 overlap、大半径变化和节点变化，说明 varied-list 完整性影响
精细结构平衡，却不等于候选轨道已经稳定。该组历史 B3 数据产生于 partner transaction
实现之前，不能反向作为原子事务证据；新事务证据见第 8 节。

权威精简表位于
[`rmcdhf_test/test/rmcdhf_orbopt/RESULTS.md`](../../rmcdhf_test/test/rmcdhf_orbopt/RESULTS.md)。

## 3. Cl I：差分松弛如何翻转能序

旧单侧 AS1--AS5 中，轨道优化对 $J=1/2$ 的总能降低比 $J=3/2$ 多
`0.0121--0.0155 Eh`。AS5 的差值约 `0.0121238 Eh = 2660.9 cm^-1`；将 no-varied
的 `+921.13 cm^-1` 加上这项不平衡差分后，得到约 `-1739.74 cm^-1`，与观察到的
倒序一致。

这证明问题不是“总能有没有下降”：变分优化可以降低各态总能，同时把态间差分做坏。
详细原始表见 [Cl I 报告](./cal-report-analysis-cl-i-lbl-orbital-optimization-2026-08-18.md)。

## 4. Job 635/636：AS 基线与当前层贡献

固定同一 AS2 CSF 后的 RCI 对照为：

| 体系/波函数 | 顺序 | 第一间隔 (cm^-1) | 第二间隔 (cm^-1) |
|---|---|---:|---:|
| Ni I，`_NV AS1 + TF AS2` | `J4<J3<J2` | 1305.08 | 2168.02 |
| Ni I，优化 AS1 + TF AS2 | `J2<J3<J4` | -1411.04 | -2730.88 |
| Ni I，优化 AS1 + 优化 AS2 | `J2<J3<J4` | -4262.78 | -8606.07 |
| Ni/Ca-like，`_NV AS1 + TF AS2` | `J2<J3<J4` | 1833.33 | 4031.32 |
| Ni/Ca-like，优化 AS1 + TF AS2 | `J2<J3<J4` | 893.62 | 1953.59 |
| Ni/Ca-like，优化 AS1 + 优化 AS2 | `J4<J3<J2` | -102.85 | -369.73 |

Ni I 在错误优化 AS1 接上 TF AS2 时已经倒序，AS2 优化继续放大；Ni/Ca-like 的优化
AS1 先压缩间隔但保留顺序，AS2 全量优化才触发反转。因此每一 AS 都要单独验收，不能
只检查最后一层，也不能用当前层 TF 覆盖错误历史。

13 个定向轨道变体中未观察到非恒等 CI 根匹配。Ni/Ca-like `all_new` 发生能序改变时
根匹配仍恒等；有效证据支持“轨道更新改变不同 $J$ 的相对能量”，不支持把 root swap
当作主因。

原始汇总位于
[`decomposition_summary.csv`](../../data/rmcdhf_test_data/results/matched-tf-rci-636/decomposition_summary.csv)。
仓库内旧安全说明曾称 Job 636 不存在；现存结果目录和 `status.csv` 与该说法冲突，
本汇总以现存结果文件为准。

## 5. Job 645：正确、严格收敛，但不优于 TF

Job 645 使用正确 balanced AS1、balanced AS2、reduced ASF、固定阻尼、节点进度检查、
round guard 和 strict SCF。结果：

- Slurm、runner、RMCDHF 和严格检查均正常完成；
- 62 轮后轨道、加权能量和身份连续两轮通过；
- 目标根始终映射自身，最低相邻轮 CI 系数 overlap 为约 0.9927；
- 62 个 round 全部接受，无 root swap、无整轮 rollback；
- 最终 `^3F_4<^3F_3<^3F_2`，累计间隔 `1374.81/2290.63 cm^-1`。

| Ni I 目标 | NIST (cm^-1) | Job 645 | Job 645 误差 | fixed-TF/RCI 误差 |
|---|---:|---:|---:|---:|
| `^3F_3` | 1332.164 | 1374.811 | +42.647 | -27.102 |
| `^3F_2` | 2216.550 | 2290.635 | +74.085 | -48.536 |

Job 645 证明这条 balanced 路径可严格、稳定地保持正确顺序；它没有证明数值优于 TF。
按“优化后 NIST 绝对误差不得大于同层 fixed-TF/RCI”规则，两项目标都应拒绝。

Job 645 与较早停止的 Job 602 只差约 `0.03/0.21 cm^-1`；相同正确 AS1 上 full ASF
Job 601 与 reduced ASF 只差约 `1.1/1.6 cm^-1`。因此 strict SCF、状态集缩减和 round
guard 不是正确能序或剩余 NIST 误差的主要原因。

原始能级见
[`e1_vv2as2_rmcdhf.csv`](../../data/rmcdhf_test_data/results/ni-target-baseline-guard-645/e1_vv2as2_rmcdhf.csv)，
TF 对照见
[`pair_summary.csv`](../../data/rmcdhf_test_data/results/fixed-orbital-rci-pair-635/pair_summary.csv)。

## 6. NIST 精度的跨体系边界

| 体系 | fixed-TF/no-varied | balanced 优化 | 结论 |
|---|---|---|---|
| Cl I | B0 误差约 `+41.46 cm^-1` | B3/B4 误差约 `+49.83 cm^-1` | 顺序修复，但精度略差 |
| Ni I | Job 635 绝对误差 `27.10/48.54` | Job 645 `42.65/74.09` | 顺序正确，但不优于 TF |
| Ni IX | no-varied 约 `+140.9/+401.7` | balanced 阻尼约 `+10/+119` | 顺序与精度均明显改善 |

所以“balanced 后接近 NIST”的原因尚不能统一归结为一个程序开关。合理但未被单因素
证明的解释是：完整伙伴选择减少 $J$ 依赖不平衡，而 TF/受控松弛在截断 CSF 中保留了
不同程度的误差抵消。剩余偏差还可能包含有限相关空间、Breit/QED 以及下一阶段的核极化
贡献，必须逐项加入，不能由事后调阈值代替。

## 7. 已排除的主线与仍有效用途

| 因素 | 证据 | 当前定位 |
|---|---|---|
| 更强固定阻尼 | 三档最终间隔只小量变化 | 稳定接受轨道，不负责修正谱 |
| raw-candidate guard | 可连续拒绝而不能恢复候选 | 安全停止，不是优化器 |
| strict `METHOD=3` | 与 B3 相同且无 fallback | fallback 不是已测主因 |
| EOL 等权/统计权 | Cl I 只差 `0.687 cm^-1` | 不是大倒序原因 |
| deferred `ORTHY` | 第 2 轮非有限 | 错误方向，停止 |
| 节点 `THRESH` 调参 | 不能统一区分尾部振荡与物理节点 | 只作诊断 |
| CI round/root guard | Job 645 无 rollback/root swap | 防传播，不产生正确候选 |
| QDIF / Gatherv | 单因素物理结果未改善 | 保留正确性修复，不作因果解释 |
| 稀疏列全局偏移修改 | Job 609 谱项严重损坏，后续已纠正为 rank-local 偏移 | 错误实验不得作物理证据 |

## 8. Partner transaction 程序级证据（2026-09-16）

当前未提交工作树以 `GRASP_PAIR_TRANSACTION=1` 显式开启事务，默认关闭时仍执行已修正的
legacy 逐轨道路径。

| 验证 | 结果 | 结果目录 |
|---|---|---|
| CTest | 4/4 通过 | 构建树内测试 |
| Ni I AS1，1 rank | 51 个 schedule 最终全部 commit；1 次节点失败整组 rollback 后以 0.5 重试成功 | `pair-smoke-final-ni-20260916000500` |
| Ni/Ca-like AS1，1 rank | 28 个 schedule 全部 commit | `pair-smoke-final-nica-20260916000500` |
| Cl I AS1，1 rank | 36 个 schedule 全部 commit | `pair-smoke-final-20260915235000` |
| Cl I，1/2/4 rank | 决定序列完全相同；最终两级能量按 CSV 精度最大差 0.0 | `pair-smoke-final-20260915235000`、`pair-final-rank2-flush-20260916004000`、`pair-final-rank4-flush-20260916004000` |
| Cl I，46-rank production | runner/求解器均为 0；与 1/2/4 rank 的 36 个决定及最终能量完全相同 | `pair-final-prod46-cl-20260916010000` |
| post-`ORTHY` 故障 | 同一 schedule/snapshot rollback，再以 0.5 commit；`rmcdhf.exitcode=0` | `pair-final-post-orthy-20260916002000` |
| 持续故障/上限 | 同一 schedule/snapshot 记录 0.5、0.7 两次 rollback，随后 `MPI_Abort(92)` | `pair-final-retry-flush-20260916003000` |
| 默认关闭 | `rmcdhf.exitcode=0`，无 pair event，capability 未置真 | `pair-default-final-20260915231500` |
| `NSIC/MAXARR` | 测试专用 override 使额外 schedule 包含所选轨道的完整 group | `pair-nsic-cl-20260915232500` |

所有成功运行的 `orbopt_trace.csv` 都是固定 93 列且无溢出字段。Ni I 的自然
`nodes_expected` 失败和 post-`ORTHY` 注入共同证明：事务会整组恢复后在原 snapshot
上重新生成两个候选，不会只接受一侧。

`run_data_case.sh` 在部分运行的后处理末端返回 1，是因为当前二进制为 `NNNP=2990`，
归档参考为 590，比较器按设计拒绝未声明的径向网格比较。这里以结果目录中的
`rmcdhf.exitcode=0`、完整 trace 和 level CSV 判定求解器成功。

## 9. 尚缺的物理证据

程序级事务已具备，但尚无证据证明三体系的**逐 AS 直接优化候选**全部满足最终目标。
下一轮必须从正确 anchor 逐层执行，并在每层同时验证能序、目标身份、严格收敛、重复性、
运行前冻结的 NIST 容差和“不劣于同层 fixed-TF/RCI”。fallback 结果不计成功。

## 10. 复现记录的最小字段

每次保留：`isodata`、实际 `rcsf.inp`、上一 AS 与 TF `rwfn`、完整 stdin、状态选择和
权重、源码提交、可执行文件 SHA-256、`NNNP`、编译器/MPI/BLAS、rank/thread 绑定、全部
`GRASP_*`、stdout/stderr、退出码、逐轮 trace、最终 `rwf/mix/sum`、固定 RCI 结果、
NIST 来源与运行前冻结的容差。`_raw.c` 与计算实际读取的 `.c` 必须分别计数和校验。
