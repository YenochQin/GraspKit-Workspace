# `rmcdhf90_mpi` 轨道优化证据汇总

## 1. 结论

当前正确能序的主要经验证原因是：**不继承已被错误单侧优化污染的 AS 基线，并让同一
$(n,l)$ 的 `nl-/nl` 都进入 varied list，从而消除原始正号 `nl` 单侧松弛造成的 $J$
依赖差分能量失衡。**

“与 NIST 差异不大”必须限定为：balanced 路径把错误单侧结果的数千至上万
`cm^-1` 偏差恢复到几十或数百 `cm^-1` 的合理量级。现有证据没有证明 balanced 本身
普遍提高 NIST 精度；Ni I Job 645 和 Cl I balanced 结果仍略劣于各自 fixed-TF/
no-varied 基线。

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
精细结构平衡，却不等于候选轨道已经稳定，更不等于伙伴联合数值更新已经实现。

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

## 8. 尚缺的证据

当前程序没有 partner transaction，因此尚无运行能证明：两个伙伴从同一前态生成候选、
使用共同阻尼、post-`ORTHY` 后同时接受或同时恢复。B3/Job 645 只能证明“完整 varied
list + 顺序逐轨道更新”在这些夹具上的结果。

下一轮有效证据必须来自实现后的 pair transaction，并要求每个 AS 的直接优化候选同时
通过能序、身份、严格收敛、NIST 预设容差和“不劣于 TF”门槛。fallback 结果不计成功。

## 9. 复现记录的最小字段

每次保留：`isodata`、实际 `rcsf.inp`、上一 AS 与 TF `rwfn`、完整 stdin、状态选择和
权重、源码提交、可执行文件 SHA-256、`NNNP`、编译器/MPI/BLAS、rank/thread 绑定、全部
`GRASP_*`、stdout/stderr、退出码、逐轮 trace、最终 `rwf/mix/sum`、固定 RCI 结果、
NIST 来源与运行前冻结的容差。`_raw.c` 与计算实际读取的 `.c` 必须分别计数和校验。
