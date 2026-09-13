# `rmcdhf90_mpi` 轨道优化测试结果

## 1. 测试目标与证据范围

本文档汇总当前已完成的 RMCDHF MPI 轨道优化诊断实验，重点回答：

1. 只优化 `nl` 而不优化 `nl-` 是否与错误的精细结构顺序相关；
2. 成对优化相对论伙伴能否恢复预期 J 顺序；
3. 固定负阻尼能否减少实际接受轨道的过大径向变化；
4. 候选轨道护栏能否拒绝异常更新并避免假收敛。

数值结果来自 `rmcdhf_test/test/rmcdhf_orbopt/RESULTS.md` 中已归档的摘要、
2026-08-25 完成的 B4 阻尼矩阵和 48 核三轮重复性汇总，以及
2026-08-26 至 2026-08-28 完成的 Cl I B7 单例和已选 Ni/Ca-like MPI
复现结果。仓库只保存小型汇总；未跟踪或仅位于外部实验目录的
完整 trace 不自动视为已归档证据。

自 2026-08-31 起，所有 runner 结果保存到
`data/rmcdhf_test_data/results/` 下的新建结果目录；输入文件始终从
`data/rmcdhf_test_data/inputs/` 复制后再计算。从 `/tmp` 找回的历史结果已保存在
`data/rmcdhf_test_data/results/recovered_20260831/tmp/`。

## 2. 测试对象与变体

已测试三个体系：

- Cl I AS1–AS5；
- Ni I；
- Ni/Ca-like。

第 3 节的 B0–B3 统一表只汇总 Ni I 和 Ni/Ca-like 两组 AS2；
Cl I 的 B0–B8 对照在后续小节单独记录。

使用的轨道选择变体为：

| 标识 | 轨道处理 | 目的 |
| --- | --- | --- |
| NV / B0 | 新轨道全部固定 | 未优化基线 |
| B1 | 仅优化原输入中的正号成员 `nl` | 重现不平衡优化 |
| B2 | 只优化负号成员 `nl-` | 伙伴单侧反事实 |
| B3 | `nl-` 与 `nl` 成对优化 | 检验伙伴平衡假设 |
| B4 | B3 加固定 `ODAMP=-0.5` | 检验阻尼对径向过冲的影响 |
| Guard | B3/B4 加候选轨道质量护栏 | 检验异常拒绝和失败终止 |

runner 使用归档 AS2 CSF 输入和前一 AS 波函数，调用当前 `rangular_mpi`、`rwfnestimate` 和 `rmcdhf_mpi`。
当 `GRASP_OMP_THREADS>1` 时，runner 使用 `--map-by slot:PE=N --bind-to core`
为每个 MPI rank 分配互不重叠的物理核，并设置 `OMP_PLACES=cores`、
`OMP_PROC_BIND=true`。每个结果目录保存 `mpi_launcher.txt` 供核查。

## 3. B0–B3 轨道选择结果

| 算例 | SCF 轮数 | 最小原始候选重叠 | 最大对称半径因子 | 节点变化次数 | 最终 J 顺序 |
| --- | ---: | ---: | ---: | ---: | --- |
| Ni I NV | 1 | 不适用 | 不适用 | 0 | J4 < J3 < J2 |
| Ni I B1 | 20 | 0.02920 | 8.610 | 2 | J2 < J3 < J4 |
| Ni I B2 | 12 | 0.03570 | 8.416 | 1 | J4 < J3 < J2 |
| Ni I B3 | 14 | 0.02902 | 8.630 | 3 | J4 < J3 < J2 |
| Ni/Ca-like NV | 1 | 不适用 | 不适用 | 0 | J2 < J3 < J4 |
| Ni/Ca-like B1 | 6 | 0.19329 | 2.781 | 0 | J2 < J4 < J3 |
| Ni/Ca-like B2 | 7 | 0.29918 | 2.784 | 0 | J2 < J3 < J4 |
| Ni/Ca-like B3 | 8 | 0.18857 | 2.798 | 0 | J2 < J3 < J4 |

### 3.1 对 J 顺序的观测

- Ni I B1 的 J 顺序与 NV 相反；B2 和 B3 均恢复了与 NV 一致的 `J4 < J3 < J2`。
- Ni/Ca-like B1 产生 `J2 < J4 < J3` 的中间两级错序；B2 和 B3 均恢复与 NV 一致的 `J2 < J3 < J4`。
- B3 消除了 varied list 不平衡警告，并在两组 AS2 实验中恢复预期 J 顺序。

这些结果支持“相对论伙伴选择不完整会导致不平衡轨道松弛，并影响精细结构顺序”的优先假设。

### 3.2 对轨道稳定性的观测

成对选择未完全解决轨道稳定性问题：

- Ni I B3 的最小重叠仍仅为 `0.02902`，明显低于方案中的 `0.1` 异常关注线；
- Ni I B3 的最大平均半径变化因子为 `8.630`，同时记录到 3 次节点变化；
- Ni/Ca-like B3 没有节点变化，但最小重叠 `0.18857` 和半径因子 `2.798` 仍表明更新不是微小扰动。

因此，B3 证明伙伴完整性影响能级顺序，但不能单独保证候选径向轨道稳定。

## 4. B4 固定阻尼结果

`GRASP_ORBITAL_DAMPING=-0.5` 不改变 `SOLVE` 生成的原始候选，但会改变 `DAMPOR` 后实际接受的轨道。

| 算例 | SCF 轮数 | 原始重叠 / 半径因子 | 接受后重叠 / 半径因子 | 接受后节点变化 | J 顺序 |
| --- | ---: | --- | --- | ---: | --- |
| Ni I B3+B4 | 29 | 0.02847 / 8.588 | 0.71710 / 2.359 | 3 | J4 < J3 < J2 |
| Ni/Ca-like B3+B4+guard | 17 | 0.19559 / 2.789 | 0.77317 / 1.558 | 0 | J2 < J3 < J4 |

结果表明：

- 原始候选质量没有因固定阻尼而改善；
- 实际接受轨道的最小重叠提高到约 `0.72–0.77`；
- 接受后最大平均半径因子降到约 `1.56–2.36`；
- Ni/Ca-like 接受后未观测到节点变化，Ni I 仍有 3 次节点变化；
- 预期 J 顺序保持不变。

这是目前支持“负阻尼能抑制实际接受更新过冲”的最强实验证据。`-0.5`
已经 Cl I AS1–AS5 的链式实验、`-0.2/-0.5/-0.8` 三体系矩阵
及每个变体三次重复。重复性验收已通过，但 Ni I 节点稳定性仍未通过，
因此仍不应固化为生产默认值。

### 4.1 2026-08-25 阻尼全矩阵

现已使用同一构建完成 Cl I AS1、
Ni/Ca-like AS2 和 Ni I AS2 的 `ODAMP=-0.2/-0.5/-0.8` 九算例矩阵。
九项均通过 legacy 收敛语义检查，退出码为 0，且没有 rank 摘要差异。

| 算例 | `ODAMP` | 轮数 | 接受后最小重叠 | 接受后最大半径因子 | 接受后节点变化 |
| --- | ---: | ---: | ---: | ---: | ---: |
| Cl I AS1 | -0.2 | 15 | 0.70448 | 2.48192 | 0 |
| Cl I AS1 | -0.5 | 21 | 0.89490 | 1.56066 | 0 |
| Cl I AS1 | -0.8 | 47 | 0.98577 | 1.13917 | 0 |
| Ni/Ca-like AS2 | -0.2 | 10 | 0.40966 | 2.43846 | 0 |
| Ni/Ca-like AS2 | -0.5 | 17 | 0.77317 | 1.55820 | 0 |
| Ni/Ca-like AS2 | -0.8 | 40 | 0.97383 | 1.16298 | 0 |
| Ni I AS2 | -0.2 | 18 | 0.26869 | 5.93090 | 3 |
| Ni I AS2 | -0.5 | 29 | 0.71710 | 2.35940 | 3 |
| Ni I AS2 | -0.8 | 47 | 0.97056 | 1.32233 | 2 |

三档阻尼下的最终谱序全部正确，间隔只有小量变化。增大阻尼能显著改善
实际接受轨道的重叠和半径稳定性，但会将收敛轮数增至 40–47。Ni I 的
原始候选最小重叠在三档中仍约为 `0.028`，最大半径因子仍约为 `8.6`；
因此阻尼稳定的是接受路径，不是 `SOLVE` 生成的原始候选。

在已测参数中，`-0.5` 仍是稳定性与收敛成本较均衡的诊断选择；
`-0.8` 适合强稳定性对照，但不支持设为默认。Ni I 节点稳定性仍未完成。

### 4.2 48 核三轮重复性矩阵

主机有 48 个物理核（2 路 × 每路 24 核，无 SMT）。亲和性核查发现，
原来虽设置了 4 MPI ranks × 12 OpenMP threads，OpenMPI 默认却将
每个 rank 限制在单个核上，实际只能使用 4 个核。修正后的启动器使用：

```text
mpirun --map-by slot:PE=12 --bind-to core -n 4
```

四个 rank 分别绑定到互不重叠的 `0–11`、`12–23`、`24–35`、
`36–47` 核集。在这一相同 48 核配置下，重新运行了三轮完整矩阵：

- 27/27 个 RMCDHF 运行退出码为 0；
- 三个九算例矩阵均为 `complete`；
- 第 2、3 轮相对第 1 轮的 18/18 项比较全部为 `match`；
- 浮点汇总的最大绝对差为 `0.00000000e+00`；
- 轮数、节点变化数、fallback 数和谱序全部一致；
- 27/27 个启动记录均包含 `slot:PE=12 --bind-to core`，无 rank 摘要差异文件。

三轮结果与上表已归档数值完全一致。精简证据保存在
`test/rmcdhf_orbopt/results/damping_repeatability_20260825/`，完整 trace 位于
`/tmp/rmcdhf-damping-repeats-pe12-20260825`。“每个阻尼变体至少三次”现已完成。

这里的 `4 ranks × 12 threads` 只证明当时测试配置的数值重复性，不代表推荐的
生产性能配置。后续监测发现 RMCDHF 主体并不会持续利用每 rank 的 12 个
OpenMP/BLAS 线程，部分阶段总 CPU 占用仅约 8%，等价于主要只有 4 个 MPI rank
工作。因此，从后续作业起，生产规模计算改用 `48 MPI ranks × 1 thread`；
1/2/4 ranks 仅保留用于 MPI 进程数重复性检查。历史 Job 528、529、532、537
的科学结论仍有效，但其 4×12 配置不能作为性能基准。

### 4.3 与本地 NIST ASD CSV 的对比

阻尼矩阵最初的自动汇总只比较计算结果之间的谱序和间隔，没有生成
“计算−NIST”列。本次使用 `data/nist_levels` 中 2026-08-24 下载的
NIST ASD CSV，按相同组态、项、宇称和 $J$ 做了补充匹配：

- Cl I：`3s2.3p5 2P*`，以 $J=3/2$ 为零点；
- Ni I：`3d8.(3F).4s2 3F`，以 $J=4$ 为零点；
- Ni IX（Ni/Ca-like）：`3p6.3d2 3F`，以 $J=2$ 为零点。

| 体系/能级 | NIST 间隔 (cm⁻¹) | `ODAMP=-0.2` 误差 | `-0.5` 误差 | `-0.8` 误差 |
| --- | ---: | ---: | ---: | ---: |
| Cl I $^2P_{1/2}$ | 882.3515 | +49.8363 | +49.8325 | +49.8240 |
| Ni I $^3F_2$ | 2216.550 | -201.117 | -201.071 | -198.984 |
| Ni I $^3F_3$ | 1332.164 | -121.322 | -121.294 | -120.021 |
| Ni IX $^3F_3$ | 1880 | +10.345 | +10.398 | +10.668 |
| Ni IX $^3F_4$ | 4070 | +118.661 | +118.773 | +119.335 |

三档阻尼的 NIST 误差很接近，说明增强阻尼的主要作用是稳定轨道接受路径，
而不是显著修正最终谱间隔。与已归档的 no-varied 基线相比：

- Ni IX 改善最明显：no-varied AS2 的 $J=3/4$ 误差约为
  `+140.9/+401.7 cm⁻¹`，成对阻尼后约为 `+10/+119 cm⁻¹`；
- Ni I no-varied AS2 的 $J=2/3$ 误差约为 `+87.3/+53.4 cm⁻¹`，
  绝对值小于成对阻尼后的约 `-200/-121 cm⁻¹`；
- Cl I no-varied AS1 间隔 `923.811587 cm⁻¹`，对 NIST 误差约
  `+41.4601 cm⁻¹`，也略优于阻尼成对优化的约 `+49.83 cm⁻¹`。

因此，“恢复正确谱序”与“比 no-varied 更接近 NIST”不是同一验收条件。
当前成对阻尼方案对 Ni IX 同时改善两者，但对 Ni I 和 Cl I 主要解决谱序/轨道稳定问题，
没有超过 no-varied 基线的 NIST 间隔精度。

## 5. 候选轨道护栏结果

### 5.1 正常通过路径

Ni/Ca-like B3+B4+guard 使用默认最小重叠阈值 `0.1`，没有触发拒绝，运行正常完成。

### 5.2 连续拒绝和明确终止

Ni I guard 运行中，`6s`、`5d`、`4f-` 和 `4f` 原始候选被拒绝。当单轨道拒绝限制设为 2 时，第三次拒绝 `6s` 后程序明确终止，没有接受该轨道，也没有将“未更新”误报为 SCF 收敛。trace 记录的下一步建议阻尼强度为 `0.5`、`0.7`、`0.9`。

### 5.3 恢复尝试的限制

在最小重叠阈值改为 `0.03` 的恢复实验中，被拒绝的 `4f` 原始候选在 6 次尝试中没有恢复，重叠反而从 `0.02847` 降到 `0.02452`。

原因是当前护栏在 `DAMPOR` 之前拒绝原始候选，虽然提高了 `ODAMP` 建议值，但没有对同一原始候选实际尝试“阻尼后再验收”。这可能拦住原本经阻尼后是稳定的更新。

因此，当前 guard 的有效交付是“检测并阻止静默接受”，而不是“自动恢复异常轨道”。guard 必须继续默认关闭。

## 6. 求解器 fallback 和 MPI 观测

### 6.x Job 557–559：AS1 命名波函数继承与初始节点诊断

为定位 Ni I AS2 节点异常是否在首次 `SOLVE` 前已经存在，进行了 AS1→AS2
链式测试。AS2 使用 AS1 生成的命名波函数（`e1_vv2as1.w`，复制为
`previous.w`），而不是 `rwfn.out`。

Job 557 首次加入了初始轨道输出，但误把 `MF` 当作节点数；`MF` 实际是该轨道
使用的径向网格长度（约 365–425），因此该批诊断无效。Job 558 的作业脚本已
完成，但因修正版二进制编译失败，实际仍调用旧二进制，结论同样无效。

Job 559 修正为在 `GETSCDmpi` 返回后直接调用 GRASP 的 `COUNT`，统计读入
波函数 `PF(:,J)` 的实际节点数。结果目录为：

```text
data/rmcdhf_test_data/results/ni-as2-initial-nodecount-559/
```

AS2 初始波函数的关键结果为：

| 轨道 | 理论 `NNODEP` | 读入波函数节点数 |
|---|---:|---:|
| `5d-` | 2 | 3 |
| `5d` | 2 | 3 |
| `6s` | 5 | 6 |

其余已输出轨道的节点数与理论值一致。AS2 第一次 `SOLVE` 随后仍报告
`6s expected=5 counted=3`，并触发 `5d-`、`5d`、`6s` 的护栏拒绝。

该结果证明：Ni I AS2 的异常节点在读入 AS1 波函数、第一次求解之前已经存在，
不是 AS2 的首次 `SOLVE`、阻尼或 MPI 归约新产生的。下一步应检查 AS1 最终
`.w` 文件中这三个轨道的实际节点，以及 AS1→AS2 波函数轨道映射；在此之前
不应放宽节点护栏判据，也不应把 `NNODEP` 替换为“候选节点必须等于理论值”。

诊断实现由以下提交构成：`cbd30c1`（初始状态输出）、`a698d4e`（修正
继承节点计数）。可复现实验脚本为
`test/rmcdhf_orbopt/run_ni_as2_initial_nodecount.sbatch`，并已同步复制到
`data/rmcdhf_test_data/inputs/rmcdhf_orbopt/`。

### 6.y Job 564–565：新增轨道初始估计方法 A/B

在确认 AS1 已有轨道继承节点正确后，Job 564 和 565 只改变 AS2 新增轨道
的 `rwfnestimate` 方法。Job 564 使用 Screened Hydrogenic（方法 3），Job
565 使用 Screened Hydrogenic [custom Z]（方法 4，程序给出 `Z_eff=22`）。

| 方法 | `5p-/5p` | `5d-/5d` | `6s` |
|---|---|---|---|
| Thomas–Fermi（基线） | 4/4 | 3/3 | 6 |
| Screened Hydrogenic | 4/4 | 2/2 | 6 |
| custom Z (`Z_eff=22`) | 4/4 | 3/3 | 6 |
| 理论 `NNODEP` | 3/3 | 2/2 | 5 |

结果目录分别为 `ni-as2-screened-initial-v2-564/` 和
`ni-as2-customz-initial-565/`。两项 AS2 均因节点护栏拒绝而退出；AS1 均
正常完成。

该 A/B 结果表明，改变 `Z_eff` 不能消除 `5p` 和 `6s` 的多一个节点，且会
使 `5d` 的结果在方法间变化。问题不像是简单的波函数继承错误或有效电荷
选择错误，更可能与新轨道在径向网格上的外层尾部、截断边界或低幅值数值
振荡有关：`COUNT` 当前对完整 `MF` 网格计数，可能把尾部符号翻转识别为
物理节点。下一项实验应在 `MTP`、`MTP-10`、`MTP-20` 等截断位置重复
`COUNT`，先确认是否存在外层伪节点，再决定是否需要调整节点判据。

### 6.0 Job 528：护栏并行复测

为确认护栏逻辑在生产并行配置下仍成立，提交 Slurm Job 528：1 节点、48
个 CPU、4 个 MPI ranks，每个 rank 12 个 OpenMP 线程。两个 AS2 算例均在同一
作业中独立执行，结果归档于
`data/rmcdhf_test_data/results/guard-tests-528/`。

- Ni I：连续候选拒绝后按预期终止；`GRASP_EXPECT_RMCDHF_FAILURE=1` 正确捕获
  非零的 `rmcdhf_mpi` 状态，作业包装器记录 `ni_i_as2_guard,0`；
- Ni/Ca-like：护栏未触发拒绝，17 轮 legacy SCF 收敛，接受后最小重叠
  `0.9999997`、最大半径因子 `1.0008062`、节点变化 `0`，并生成完整
  `e1_cvas2_rmcdhf.csv`；包装器记录 `ni_ca_like_as2_guard,0`；
- Slurm 作业和 batch step 均为 `COMPLETED`，退出码 `0:0`，资源记账为
  `cpu=48`。因此，单核 Job 527 得到的护栏结论已在 4-rank/48-CPU 配置下复现。

### 6.1 2026-08-28 Slurm 三轮 MPI 复现

为验证结果不依赖 MPI 进程数，使用 `rmcdhf_test/test/rmcdhf_orbopt/run_repro_sbatch.sh`
通过 Slurm 作业 491 完成 Ni/Ca-like AS2 的 B0、B1、B3 和 B8 三轮复现。作业按
`Cl_I/mcdhfmpi.sh` 的规范申请 1 节点、48 个 task、`batch` 分区；三轮分别使用
MPI 1/2/4，并将 OpenMP 线程设为 12/6/3。作业耗时 1:23:49，Slurm 状态为
`COMPLETED`，退出码为 0。

原结果保存在 `test/data/results/repro_sbatch_20260828/`；删除事故后该目录未能原样找回。12 个运行全部
`rmcdhf.exitcode=0`，并且每个变体三次生成的 `rmcdhf.sum` 文件逐字节一致：

| 变体 | MPI 1/2/4 | 三次结果 |
| --- | --- | --- |
| B0 | 1/2/4 | MD5 `ec3e70f89e44e55597512519ab8b75d0` |
| B1 | 1/2/4 | MD5 `651da357426fe1d77f2f738435f09e8c` |
| B3 | 1/2/4 | MD5 `3db25dde7a5e19bb7108ba66295df509` |
| B8 | 1/2/4 | MD5 `eac46afb5f4cf7407f6ae9bd516588ce` |

所有 12 份 `orbopt_trace.csv` 均通过 legacy 收敛语义检查，证明 B1 的异常谱序、
B3 的成对优化结果和 B8 的等权重结果在这一 Ni/Ca-like AS2
输入上可于 MPI 1/2/4 重复。该结果补齐了所列 B0/B1/B3/B8 的三次
重复和 MPI 1/2/4 一致性证据，不代表 B2、B5、B6、strict 或全部体系已完成
同等覆盖。

同时修复 `check_strict_scf.py`：新版 trace 缺少派生字段时，根据
`convg_orbital`、`convg_energy`、`convg_final` 和 `wtaev0` 自动推导
`convg_legacy`、`convg_strict`、`energy_valid`、`strict_streak`，旧版 trace
仍按原字段直接验证。由此避免将分析器版本不匹配误报为计算失败。

已记录的 B2/B3/B4 运行均未发生 `SOLVE` 失败后的 `METHOD=2` fallback。当前证据不支持把 fallback 作为这两组 AS2 结果的首要原因。

原先的性能记录未显式为每个 rank 分配 12 个 PE，因此不能用于
证明 4×12 是最快配置。本次已证明修正后的 4×12 可完整使用 48 个物理核
并保持三轮数值完全一致；若要比较最佳性能，仍需在同样显式绑定下重做计时基准。

这些是生产 `_mpi` 路径的重复性观测；串行与 MPI 逐轮对照不属于本方案的必需验收。

## 7. 构建与工具检查

2026-08-25 使用当前 `rmcdhf_test` 源码进行了新的干净构建和 smoke
数值回归。实施版本为子仓库提交 `71954ca`
(`rmcdhf_mpi: preserve summary after rsave`)。

构建核查包括：

- 在新的 `/tmp` out-of-source 目录中配置 CMake；
- 使用 GNU Fortran 15.2.1、OpenMPI 和 FlexiBLAS；
- 成功编译、链接 `rmcdhf_mpi`；
- `bash -n` 通过 `run_data_case.sh` 和 `run_matrix.sh`；
- `git diff --check` 通过；
- CTest `lib9290_quad` 1/1 通过。

仓库内原有 `build-debug/CMakeCache.txt` 绑定旧源路径
`/home/workstation2/AppFiles/grasp_test`，因此本次使用
`/tmp/rmcdhf-build-debug` 作为新的 out-of-source Debug 构建目录。

### 7.1 2026-08-25 smoke 矩阵

执行命令为：

```shell
GRASP_BINDIR=/tmp/rmcdhf-build-debug/bin \
  bash test/rmcdhf_orbopt/run_matrix.sh \
  /tmp/rmcdhf-smoke-20260825-fix smoke
```

此矩阵使用 Ni/Ca-like AS1 小型数据，覆盖 B0–B6 和 strict SCF：

| 运行 | 配置 | `rmcdhf_mpi` 退出码 | 结果 |
| --- | --- | ---: | --- |
| B0 | 新轨道全部固定 | 0 | 成功 |
| B1 | 仅正号伙伴 | 0 | 成功 |
| B2 | 仅负号伙伴 | 0 | 成功 |
| B3 | 成对优化 | 0 | 成功 |
| B4 | B3 + `ODAMP=-0.5` | 0 | 成功 |
| B5 | 延迟 `ORTHY` | 1 | 按设计预期失败 |
| B6 | 固定 `METHOD=3` | 0 | 成功 |
| strict | B3 + strict SCF | 0 | 成功 |

对每个成功算例的最低正宇称 $J=2,3,4$ 能级做统一比较，以 $J=2$
为零点得到：

| 运行 | 顺序 | $E_{J=3}$ (cm⁻¹) | $E_{J=4}$ (cm⁻¹) |
| --- | --- | ---: | ---: |
| B0 | J2 < J3 < J4 | 2021.629114 | 4473.586527 |
| B1 | J2 < J3 < J4 | 589.982486 | 1287.212568 |
| B2 | J2 < J3 < J4 | 3599.094687 | 7700.599946 |
| B3 | J2 < J3 < J4 | 2021.701541 | 4473.364858 |
| B4 | J2 < J3 < J4 | 2021.694517 | 4473.349934 |
| B6 | J2 < J3 < J4 | 2021.701541 | 4473.364858 |
| strict | J2 < J3 < J4 | 2021.703296 | 4473.368369 |

B3 与 B6 结果相同，再次说明本算例中 `METHOD` fallback 不是结果差异的原因。
B4 相对 B3 只有小量间隔变化，strict 模式在增加收敛轮数后也保持同一能级顺序。

B5 由 `GRASP_EXPECT_RMCDHF_FAILURE=1` 明确标记为预期失败，因此整个
smoke profile 最终状态为 `complete`。收敛语义检查证实：

- legacy 模式在第 7 轮停止；
- strict 模式在第 13 轮停止，最后两轮连续满足双门槛。

### 7.2 测试驱动器缺陷与修复

第一次 smoke 运行中，B0 数值计算和标准 GRASP 后处理均已成功，
但 `compare_sum.py` 报告 `rmcdhf.sum` 不存在。根因是 `rsave`
将通用文件重命名为算例前缀的 `${result_name}.sum`，而后续比较器和
`run_matrix.sh` 仍以 `rmcdhf.sum` 作为稳定接口。

`run_data_case.sh` 现在会在 `rsave` 后从 `${result_name}.sum` 恢复一份
`rmcdhf.sum`。修复后重跑整个 smoke 矩阵，所有预期成功/失败路径均符合方案，
同时 `archive_comparison.csv` 和矩阵级汇总可正常生成。

## 8. 未完成的测试项

### 8.1 Job 529：Cl I AS2--AS5 链式节点稳定性

使用 48 CPU、4 MPI ranks（每 rank 12 个 OpenMP 线程）提交连续活动空间测试。
AS2 的输出波函数逐阶段传递给 AS3、AS4 和 AS5，固定 `ODAMP=-0.5`。作业
529 完成，四个阶段退出码均为 0，均通过 legacy 收敛检查：

| 阶段 | SCF 轮数 | 最小接受重叠 | 最大接受半径因子 | 节点变化 |
| --- | ---: | ---: | ---: | ---: |
| AS2 | 20 | 0.9999999 | 1.0003965 | 0 |
| AS3 | 21 | 0.9999828 | 1.0010473 | 0 |
| AS4 | 27 | 0.9995258 | 1.0025099 | 0 |
| AS5 | 29 | 0.9999336 | 1.0046493 | 0 |

结果目录为 `data/rmcdhf_test_data/results/cl-as-chain-529/`。在该 Cl I 链式
算例中，扩大活动空间没有引入节点跳变；AS4/AS5 的半径变化仍处于百分之一量级，
支持将该配置作为稳定基线。该结论不能外推到 Ni I（其 AS2 仍有节点变化）。

### 8.2 Job 530：Ni I 节点验收 A/B

Job 530 使用 48 CPU、4 MPI ranks × 12 OpenMP threads，对 Ni I AS2 仅切换
`GRASP_REJECT_NODE_CHANGE`，并放宽重叠/半径阈值以隔离节点因素。`node_check_off`
在 29 轮后正常完成；`node_check_on` 在第 3 轮对 `6s` 连续第 3 次节点拒绝，
日志明确记录 `ORBOPT rejection limit exceeded for 6s`，4 个 RMCDHF 进程随后
退出。但 MPI 运行时 `prterun` 未完成回收，Slurm 作业在计算已停止后仍保持
RUNNING、CPU 使用率接近 0%，最终由管理员取消（总挂起约 17 小时）。

该结果确认节点验收确实能阻止异常候选，但也暴露出“预期失败路径 + MPI 回收”
的作业包装问题；不能把 Slurm 的长时间 RUNNING 误判为仍在计算。后续应在脚本中
为预期失败路径设置超时/进程组清理，并单独测试拒绝后的阻尼恢复。

### 8.3 Job 532：预期失败路径进程组清理复测

Job 532 使用修正后的 runner，在检测到 `ERROR STOP ORBOPT` 或拒绝上限标记后
立即终止 timeout/MPI launcher，并保留完整退出码。48 CPU、4 MPI ranks 的
`node_check_off` 在 29 轮后正常完成；`node_check_on` 在第 3 轮对 `6s`
触发节点拒绝并以 `rmcdhf.exitcode=1` 结束，包装器将其作为 expected failure
记录。整个作业 10 分 40 秒完成，Slurm 退出码为 `0:0`，未留下 `prterun`、
`mpirun` 或 `rmcdhf_mpi` 残留进程。结果位于
`data/rmcdhf_test_data/results/ni-node-acceptance-532/`。

这证明 530 暴露的是 MPI 失败回收/脚本问题，而非 RMCDHF 在节点拒绝后持续计算。

### 8.4 Job 533：阻尼后节点恢复证据

Job 533 从 Job 525 的 Ni I AS2 trace 逐更新配对 `orbital_metrics`（原始
`SOLVE` 候选）与 `accepted_metrics`（`DAMPOR` 后实际接受轨道），统计原始
节点变化能否被阻尼修复：

| `ODAMP` | 原始节点变化 | 阻尼后恢复 | 阻尼后仍变化 | 新引入变化 |
| ---: | ---: | ---: | ---: | ---: |
| -0.2 | 4 | 1 | 3 | 0 |
| -0.5 | 9 | 6 | 3 | 0 |
| -0.8 | 28 | 26 | 2 | 0 |

结果位于 `data/rmcdhf_test_data/results/node-recovery-533/`。证据表明在原始
候选阶段直接拒绝会拦截大量本可由阻尼恢复的更新；下一步可以实现默认关闭的
“先阻尼、再按节点/重叠/半径验收”实验路径。仍未恢复的候选必须回退，不能仅因
高阻尼总体恢复率较高而接受。

### 8.5 Job 537：阻尼后验收矩阵

Job 537 使用 48 CPU、4 MPI ranks 完成四算例矩阵，Slurm 状态
`COMPLETED`、退出码 `0:0`，耗时 21 分 41 秒，且没有残留 MPI 进程。

| 算例 | `rmcdhf_mpi` 状态 | 终止轮次 | 接受后节点变化 | 结论 |
| --- | ---: | ---: | ---: | --- |
| Ni I，阻尼前，-0.5 | 1（预期失败） | 3 | 0 | `6s` 连续拒绝 |
| Ni I，阻尼后，-0.5 | 1（预期失败） | 13 | 0 | `6s` 最终仍连续拒绝 |
| Ni I，阻尼后，-0.8 | 1（预期失败） | 15 | 0 | `5d` 最终仍连续拒绝 |
| Ni/Ca-like，阻尼后，-0.5 | 0 | 17 | 0 | 正常收敛 |

阻尼后验收将 Ni I 的有效推进从第 3 轮延长到第 13/15 轮，并保证所有实际
接受更新都没有节点变化，说明新路径能恢复多数候选且不会静默接受错误节点。
但 Ni I 仍不能完成 SCF：-0.5 最终卡在 `6s`，-0.8 最终卡在 `5d`。
因此该实验开关继续默认关闭，不能宣称已经解决 Ni I 稳定性问题。

### 8.6 Jobs 538/541：默认关闭回归与 48-rank 试运行

Job 538 验证新开关默认关闭时的兼容性。Ni I 与 Ni/Ca-like AS2 均正常完成，
相对 Job 525 对应 `ODAMP=-0.5` 基线的 CSF 数、径向网格、能级集合完全一致，
最大能量差均为 `0.0 Hartree`。因此默认关闭回归通过。

Job 541 尝试以 `48 MPI ranks × 1 thread` 重跑阻尼后验收。Ni I 的 -0.5/-0.8
算例仍分别以真实退出码 1 预期终止。Ni/Ca-like 已推进到第 17 轮，但在该轮
最后一个轨道更新附近没有完成 SCF 收尾，达到 30 分钟 runner 超时后以 124
结束；Slurm 作业因此为 `FAILED (124:0)`。作业总耗时 52 分 03 秒，累计
CPU 时间约 41 小时，说明大部分时间确有计算，并非全程 0% CPU 的 launcher
假挂起。

该结果表明“所有算例直接使用 48 ranks”不能仅凭核数假定更高效；当前

### 8.7 Jobs 546--549：纯 MPI rank 扩展性诊断

同一 Ni/Ca-like AS2 在 12、24、48 MPI ranks（每 rank 1 thread）下均完成 17
轮收敛，结果完全一致。用时分别为 987 s、1958 s、1893 s。该结果说明 12
ranks 已接近本算例的最佳规模，24/48 ranks 的通信和负载粒度开销使其更慢。
后续生产应依据实测扩展性选择 ranks（当前该算例先用 12 ranks），并保持
`OMP_NUM_THREADS=OPENBLAS_NUM_THREADS=1`。
Ni/Ca-like AS2 在 48 ranks 下还存在负载粒度、同步或退出阶段问题。Job 541
不能作为 48-rank 验收通过证据，后续应先做 48-rank 卡点诊断，再决定生产
rank 数；同时保持每 rank 1 thread，不恢复低利用率的 4×12 混合配置。

下列内容在原调试方案中已定义，但尚没有完整结果：

- B7 独立 CI/RCI 的 Ni I、Ni/Ca-like 多体系矩阵；
- B8 不同状态集合对照（等权/统计权重对照已完成）；
- 除已验证的 Cl I 对照外，所有算例的 `_mpi` 1/2/4 重复性覆盖；不再要求串行与 MPI 逐轮数值一致；
- 每轮 `rwfn/rmix`、可执行文件校验值、编译器和 commit 的完整实验归档；
- AS2 以后已观测节点数波动的稳定化与验收。

## 9. 当前证据结论

| 假设 | 当前证据 | 结论 |
| --- | --- | --- |
| varied list 遗漏 `nl-` 会造成不平衡松弛 | B1 错序，B2/B3 恢复预期 J 顺序 | 得到支持 |
| 只要成对优化就能保证轨道稳定 | Ni I B3 仍有低重叠、大半径变化和节点变化 | 不成立 |
| 固定负阻尼能抑制实际接受轨道的过冲 | B4 接受后重叠显著提高，半径变化和节点变化降低 | 得到支持 |
| 增强阻尼会显著改善 NIST 间隔 | 三档阻尼的间隔误差只有小量变化 | 当前不支持 |
| `METHOD=2` fallback 是已测 AS2 异常的主因 | 已记录运行中无 fallback | 当前不支持 |
| 在原始候选阶段直接拒绝可以自动恢复 | 放宽阈值后 `4f` 仍连续恶化 | 当前不成立 |
| 已满足最终生产验收 | 三轮阻尼重复性已通过，但 B7 只完成 Cl I 单例，完整串行/MPI 矩阵和节点稳定性未完成 | 不成立 |

## 10. B7 独立 CI/RCI 对照（2026-08-26）

已加入 `test/rmcdhf_orbopt/run_b7_ci.sh`。该脚本严格按
`test/data/Cl_I/rci.sh` 的原始 GRASP 工作流执行：默认调用模块提供的串行
`rci`，随后运行 `jj2lsj` 和 `rlevels`；设置 `GRASP_B7_MPI=1` 时才调用模块
提供的 `rci_mpi`。脚本不会使用 `rmcdhf_test/build-debug/bin/rci*`。

在 Cl I AS1 成对阻尼（B3+B4）最终轨道上完成了一次固定轨道 RCI：138 个 CSF、
两个负宇称块（$J=1/2,3/2$），生成 `b7.cm`、`b7.csum`、`b7.clog` 和
`b7.level`。最低 ${}^{2}P$ 间隔为 `897.43 cm⁻¹`，而同一轨道的 RMCDHF
间隔为 `932.18 cm⁻¹`；两者均保持 $J=3/2$ 低于 $J=1/2$。该差异是固定轨道
CI（含 RCI 的重混合、Breit/QED 选项）相对 RMCDHF EOL 结果的独立对照，不能
解释为再次进行了轨道优化。

当前 B7 交付证明了固定轨道 CI 路径可复现并能与 RMCDHF 输出分离比较；尚未
完成 Ni I、Ni/Ca-like 的多体系 B7 矩阵，也尚未把 CI 差异归因到具体 Breit/QED
子项。

## 11. 测试阶段结论

## 12. AS2 节点稳定性专项（Job 525）

2026-09-01 通过 Slurm 作业 525 完成 Ni I 与 Ni/Ca-like AS2 的成对优化阻尼对照，覆盖 `ODAMP=-0.2/-0.5/-0.8`。六个算例均以退出码 0 完成，未发生 `METHOD` fallback。

| 体系 | 阻尼 | 迭代数 | 最小接受重叠 | 最大半径因子 | 接受后节点变化 |
|---|---:|---:|---:|---:|---:|
| Ni/Ca-like AS2 | -0.2 | 11 | 0.409663 | 2.438464 | 0 |
| Ni/Ca-like AS2 | -0.5 | 18 | 0.773172 | 1.558197 | 0 |
| Ni/Ca-like AS2 | -0.8 | 41 | 0.973825 | 1.162980 | 0 |
| Ni I AS2 | -0.2 | 19 | 0.268685 | 5.930900 | 3 |
| Ni I AS2 | -0.5 | 30 | 0.717101 | 2.359403 | 3 |
| Ni I AS2 | -0.8 | 48 | 0.970563 | 1.322330 | 2 |

增强负阻尼可以显著提高接受轨道重叠并降低半径突变，但会增加收敛轮数。Ni/Ca-like AS2 三档阻尼均无接受后节点变化；Ni I AS2 即使在 `-0.8` 下仍有少量节点变化，因此 Ni I 节点稳定性尚未通过。`-0.8` 只作为强稳定性对照，不设为生产默认值。逐轮摘要保存在 `data/rmcdhf_test_data/results/node-stability-525/`。

已有测试支持以下阶段性结论：

1. 对可比较的相对论伙伴，生产输入应至少进行完整性检查；
2. B3 在已测 Ni I 和 Ni/Ca-like AS2 中恢复了预期 J 顺序；
3. 对 Ni I，成对选择后仍必须配合阻尼或更深的求解/正交化调查；
4. `ODAMP=-0.5` 对实际接受更新显示出明确稳定作用，但还不足以设为全局默认值；
5. 当前 guard 适合做失败检测和保护性终止，不适合宣称具有自动恢复能力；
6. Cl I、strict SCF、B5/B6、阻尼全矩阵、三轮重复性和已选 MPI 对照已补齐，
   但在完成 B7、完整串行/MPI 矩阵与节点稳定性验收之前，本实施仍是诊断/实验版，
   不是已验收的最终生产修复。
### Job 567：COUNT 的 `THRESH` 敏感性

Job 567 测试了 `GRASP_COUNT_THRESH=0.01、0.05、0.20`，并在 `MTP`、
`MTP-10`、`MTP-20` 截断位置重复计数。结果为：

| `THRESH` | `5p-/5p` | `5d-/5d` | `6s` |
|---:|---|---|---:|
| 0.01 | 4/4 | 3/3 | 6 |
| 0.05（默认） | 4/4 | 3/3 | 6 |
| 0.20 | 2/2 | 3/3 | 4 |
| 理论值 | 3/3 | 2/2 | 5 |

三种截断位置的结果完全相同，排除了外层网格边界伪节点。`THRESH=0.20`
会滤掉真实节点，不能用于生产；默认 `0.05` 暂不修改。结果目录为
`data/rmcdhf_test_data/results/ni-count-threshold-matrix-567/`。
### Job 574：高 `THRESH` 边界测试

Job 574 测试了 `THRESH=0.25、0.30、0.40、0.50`。`THRESH=0.30` 可将
`5d-/5d` 调整为理论的 2 个节点，但同时使 `5p` 和 `6s` 严重欠计数；
更高阈值继续删除真实节点。结果证明不存在可同时修正所有新增轨道的统一
全局 `THRESH`，不能通过调高默认阈值解决 Ni I AS2 问题。

结果目录：`data/rmcdhf_test_data/results/ni-count-threshold-matrix-574/`。
后续应转向轨道相关的节点判定，或修正新增轨道的初始径向函数生成过程。
### Job 575：恢复 `ED1=PED(J)` 的回归验证

串行 `IMPROV` 在每次轨道优化开始时执行 `ED1=PED(J)`，MPI 版此前遗漏。
已补回该行并重新编译，在 Job 575 中运行 Ni I AS1→AS2 回归。AS1 正常
完成；AS2 仍因 `5d-`、`5d`、`6s` 节点候选连续被拒绝而达到拒绝上限，
`rmcdhf.exitcode=1`（按预期护栏失败）。

结果目录：`data/rmcdhf_test_data/results/ni-ed1-fix-regression-575/`。
该修复纠正了 MPI 与串行版的能量状态初始化差异，是必要的兼容性修正，
但没有消除 Ni I AS2 的节点失稳，因此不是当前能级顺序异常的唯一根因。

### Jobs 579–582：`NDCOF` 合并计数与节点护栏诊断

Job 579 使用提交 `5dbd12f`（MPI `DA/NDA` 合并后令 `NDCOF=i_last`）在
Ni/Ca-like AS2 上运行 46 个 MPI rank。作业退出码为 0，17 轮收敛，无
fallback、节点变化或轨道拒绝。前三个 $^3F_J$ 能级保持
$J=2<J=3<J=4$，间隔为 1890.40 和 2298.38 cm$^{-1}$；NIST 基准为
1880 和 2200 cm$^{-1}$。本次结果证明该修复在高 rank 下可稳定运行，但
尚不能单独证明它消除了所有物理偏差。

Job 580 使用同一修复和 46 个 MPI rank 完成 Cl I AS2–AS5 成对优化。四个
阶段退出码均为 0，分别经过 20、21、27、29 轮收敛；无 fallback 和节点
变化。所有阶段均保持 $^2P_{3/2}<{}^2P_{1/2}$，间隔为 930.99、931.12、
931.15、931.09 cm$^{-1}$，而 NIST 为 882.3515 cm$^{-1}$。因此轨道优化
后的顺序稳定，但精细结构间隔仍有约 49 cm$^{-1}$ 的系统偏差。

Job 581 在 Ni I AS1→AS2 上恢复节点护栏（46 个 MPI rank）。AS1 正常
完成；AS2 在第 12 轮因 `5d-` 候选节点由 3 变为 2，达到拒绝上限并以
退出码 1 终止。阻尼后的已接受轨道节点数未变化，首次异常出现在候选
轨道阶段。

Job 582 只关闭节点拒绝，其他参数与 Job 581 相同。AS1 和 AS2 均成功
收敛（29、32 轮），最终保持 $^3F_4<{}^3F_3<{}^3F_2$；间隔为 1375.88
和 916.11 cm$^{-1}$，NIST 为 1332.164 和 884.39 cm$^{-1}$。trace 明确
记录了接受后的节点变化：第 3–4 轮 `5p-`、`5p`、`6s`、`5d-`、`5d`
分别出现 3→2、3→2、6→5、3→2、3→2。该对照证明 AS2 失败来自节点
护栏对候选轨道的拒绝，而非求解器不能收敛；候选轨道生成和初始径向
函数仍是后续主线。

### Jobs 586–588：重复性与 `ORTHY` 排除实验

Jobs 586、587 分别重复 Jobs 581、582 的节点护栏开/关实验。两次 AS2
`orbopt_trace.csv` 与对应原实验逐字节相同（SHA-256 分别为
`105819ba...f504a9` 和 `1b05593c...478d2`）。这证明护栏失败和关闭护栏后
的收敛结果在 46-rank LBL MPI 流程中可重复。

对 Job 587 最终 `e1_vv2as2.w` 使用 `graspkit-tools` 独立转换为径向 CSV，
再按 GRASP `COUNT` 的 0.05 幅值阈值计数，得到 `5p-/5p=3/3`、
`5d-/5d=2/2`、`6s=5`，全部等于 `NNODEP`。因此第 3–4 轮记录的节点变化
不是最终轨道丢失物理节点，而是优化把带多余节点的初始估计修正到要求值。
当前护栏要求 `nodes_old == nodes_candidate`，会错误拒绝这种合法修正。

Job 588 设置 `GRASP_DEFER_ORTHY=1`，在 Ni I AS1 第 3 轮即失败。`4d`
候选重叠降至约 `1.6e-51`，平均半径从约 5.76 缩至 `7.5e-9 a0`，连续触发
半径拒绝。结合 Job 587 的节点变化在 `DAMPOR` 后、`ORTHY` 前已经出现，
逐轨道 `ORTHY` 不是节点变化的首发位置；延迟正交化会破坏 SCF 稳定性，
不能作为修复。

### Job 589：理论节点判据与阻尼后检查的边界

Job 589 使用提交 `b47ac87` 的“候选节点必须等于 `NNODEP`”判据，46 个
MPI rank，`ODAMP=-0.5`，逐轨道 `ORTHY`，并开启阻尼后检查。AS1 正常经过
29 轮；AS2 在第 3 个宏迭代处理 `6s` 时以 `rmcdhf.exitcode=137` 停止。
标准输出同时记录了 3 次
`ERROR STOP ORBOPT orbital rejection limit exceeded`，随后 runner 主动
终止 `srun` 进程组，因此 137 是预期失败路径的 launcher 清理码，不是
Slurm 的 OOM 证据。

该 trace 将失败阶段分开了：

| 阶段 | `6s` 节点数 | 与理论 `NNODEP=5` 的关系 |
| --- | ---: | --- |
| AS2 初始旧轨道 | 6 | 多 1 个节点 |
| 第 1/2 轮原始 `SOLVE` 候选 | 3 | 少 2 个节点，原始检查拒绝 |
| Job 587 同一输入的 `DAMPOR` 后候选 | 6 | 回到旧轨道节点数，距离理论值没有变坏 |

因此“阻尼后候选必须立即等于理论值”仍然过严：它会拒绝从错误初始节点
向理论节点逐步靠近的合法更新。更细的代码审计还发现，`DAMPOR` 之后
`P/Q` 保存旧轨道、`PF/QF` 保存阻尼后的候选，但 Job 589 的检查直接把
`CALCULATE_ORBITAL_METRICS` 返回的 `NODES_OLD/NODES_CANDIDATE` 作为原方向
传入；阻尼后检查实际检查了反向的“旧/新”对象。该方向错误必须与节点
判据修正分开记录。

下一项实验通过 `GRASP_NODE_GUARD_PROGRESS=1` 只改变这两处节点护栏逻辑：
原始候选不再要求直接等于 `NNODEP`，阻尼后按 `abs(nodes- NNODEP)` 相对旧
轨道不增加来验收，并修正阻尼后质量指标的参数方向；阻尼因子、正交化顺序、
EOL 权重、MPI rank 和输入保持不变。若该实验能完成 AS2，再检查最终波函数是否回到 `5p-/5p=3/3`、
`5d-/5d=2/2`、`6s=5` 以及 $^3F_4<{}^3F_3<{}^3F_2$，不能只依据作业
退出码判定修复成立。

### Job 590：节点单向进度护栏

Job 590 在同一 Ni I AS1→AS2 输入上开启 `GRASP_NODE_GUARD_PROGRESS=1`，使用
46 个 MPI rank、`ODAMP=-0.5`、逐轨道 `ORTHY` 和
`/home/workstation2/caltmp/mpi_tmp`。AS1、AS2 分别经过 29、33 轮并以退出码
0 完成；`status.csv` 为 `status,complete`，没有 `METHOD=2` fallback。

该开关只允许节点数向理论 `NNODEP` 靠近，原始 `SOLVE` 候选不再因尚未立即
达到理论值而拒绝，且阻尼后检查使用了正确的新旧轨道方向。trace 显示：

| 轨道 | 初始节点 | 最终节点 | 理论 `NNODEP` | 诊断行为 |
| --- | ---: | ---: | ---: | --- |
| `6s` | 6 | 5 | 5 | 前 3 轮阻尼后保持 6，第 4 轮恢复到 5 |
| `5p-` | 3 | 3 | 3 | 第 3 轮原始候选 3→2，被 `nodes_progress` 拒绝一次 |
| `5p` | 3 | 3 | 3 | 无最终节点变化 |
| `5d-` | 3 | 2 | 2 | 第 4 轮恢复到理论值 |
| `5d` | 3 | 2 | 2 | 第 4 轮恢复到理论值 |

AS2 的最终 $^3F_J$ 顺序仍为 $^3F_4<{}^3F_3<{}^3F_2$，间隔为
1375.88 和 916.11 cm$^{-1}$；相对 NIST 的 1332.164 和 884.39 cm$^{-1}$，
误差分别为约 +43.72 和 +31.72 cm$^{-1}$。因此该实验验证了节点进度护栏、
阻尼后检查方向修正和 46-rank LBL 流程的稳定性，但没有消除 Ni I 精细结构
间隔的物理偏差，也不能单独作为最终生产修复。

与 Job 587 比较，AS1 结果完全一致；AS2 最大能量差为
`4.39×10^-7 Hartree`（约 `0.096 cm^-1`）。差异来自 `5p-` 的一次受控节点
拒绝，最终节点计数和能级顺序没有改变。

结果目录：`data/rmcdhf_test_data/results/ni-as1-as2-chain-590/`；日志：
`data/rmcdhf_test_data/log/590_rmcdhf-ni-node-progress.log`。

### Jobs 591–604：Ni/Ca-like、Cl I、Ni I 复测以及 B7/B8 状态选择

本批 RMCDHF 作业（591–597、601–604）使用 46 个 MPI rank，B7 固定轨道
RCI 作业（598–600）使用 1 个 rank。作业已经结束，但退出状态不能简单等同于
“测试通过”，需要按计算本体和 wrapper 后处理分别判断。

| 作业 | 输入/开关 | 计算结果 | 结论 |
|---|---|---|---|
| 591 | Ni/Ca-like，default-off | 第 11 轮附近达到旧脚本的 30 min runner 超时，`rmcdhf.exitcode=124` | 不是节点护栏失败；预算不足 |
| 592 | Cl I，default-off，AS1→AS5 | AS1/AS2 分别 21/20 轮、退出码 0；AS3 exact-node guard 退出 137 | AS1/AS2 可重复，AS3 为预期护栏失败，AS4/AS5 未运行 |
| 593 | Ni I，legacy exact-node guard | AS1 29 轮、退出码 0；AS2 连续拒绝 `6s`，退出码 1 | `status.csv` 标为 `expected_failure`，说明旧判据仍会拒绝合法节点修正 |
| 594 | Ni/Ca-like，`GRASP_NODE_GUARD_PROGRESS=1` | 与 591 一样在第 11 轮附近以 124 结束 | 进度护栏尚未运行到收敛，不能据此评价护栏 |
| 595 | Cl I，node-progress，AS1→AS5 | AS1/AS2 与 592 相同；AS3 连续 `nodes_progress` 拒绝，退出码 1 | 放宽为单向进度后仍不能完成 AS3，AS4/AS5 未运行 |
| 596 | Ni I，node-progress 重复 | AS1/AS2 为 29/33 轮，退出码均为 0，`status=complete` | 与 Job 590 的 trace 逐字节一致，46-rank 流程具有重复性 |
| 597 | Ni I，保存每轮 `rwfn.out` | AS1/AS2 为 29/33 轮，退出码均为 0，`status=complete` | 可用于定位 `SOLVE→DAMPOR→ORTHY` 的第一次状态差异 |
| 598 | Ni I B7 固定轨道 RCI | 已生成 `b7_rci.csv` | `$^3F_3`、`$^3F_2` 间隔为 1297.15、2160.19 cm⁻¹，顺序正确 |
| 599 | Ni/Ca-like B7 固定轨道 RCI | 已生成 `b7_rci.csv` | `$^3F_3`、`$^3F_4` 间隔为 1702.72、3746.95 cm⁻¹，低于 NIST 1880、4070 cm⁻¹ |
| 600 | Cl I B7 固定轨道 RCI | 已生成 `b7_rci.csv` | $^2P_{1/2}-{}^2P_{3/2}=896.13$ cm⁻¹，接近 NIST 882.3515 cm⁻¹ |
| 601 | Ni I B8 full ASF | 33 轮，退出码 0，`status=complete` | 能级与 Job 590 相同，$^3F_4<{}^3F_3<{}^3F_2$；full ASF 选择没有改变结果 |
| 602 | Ni I B8 target ASF | RMCDHF 18 轮、退出码 0，顺序正确 | 间隔 1374.78、915.63 cm⁻¹；wrapper 只因 target 与 full 的 level 集合不同而比较失败 |
| 603 | Ni/Ca-like B8 full ASF | 第 11 轮附近以 124 结束 | 仍是时间预算问题，不能评价 full ASF |
| 604 | Ni/Ca-like B8 target ASF | 第 11 轮附近以 124 结束 | 仍是时间预算问题，不能评价 target ASF |

`sacct` 在 2026-09-08 核对到 Jobs 589–604 均已进入 Slurm 终态；其中
终态为 `COMPLETED` 的作业才表示脚本和计算本体正常返回。591、594、603、604
的 `rmcdhf_mpi` 均被 30 分钟 runner 超时终止（退出码 124），592、595 和 593
分别是链式护栏/节点进度护栏的预期失败路径，598–600 的 RCI 已写出 CSV，但
后处理脚本因没有 `b7.h`/`b7.ch` 而返回 1。因而“作业都结束”不能写成“所有
算例都完成”；可用于数值结论的本批 RMCDHF 结果是 596、597、601、602，
以及 591/594/603/604 截止超时前的诊断 trace。

Ni/Ca-like 早先的完整 46-rank 记录位于
`data/rmcdhf_test_data/results/ndcof-ilast-20260905/np46/`，耗时约 2687 s
（44 min 47 s）。这解释了 591、594、603、604 在 30 min 限制下的 124；它们
的 trace 在前 11 轮基本一致，没有节点变化或 fallback。相关脚本已经改为 60 min，
重跑后才能对 Ni/Ca-like 的进度护栏和 B8 状态选择作出结论。

B7 的原始 wrapper 错误地给 `read_level_to_csv.py` 传入 `-gj`，而 RCI 只生成
了 `b7.level`，没有 `b7.h`/`b7.ch`。去掉该选项后已生成三个 `b7_rci.csv`，上表
数值可归档；对应修正后的 sbatch 脚本仍可重新提交以取得完整的自动化日志。
Ni I B8 target 的后处理应使用 `GRASP_ALLOW_LEVEL_DIFFERENCES=1`，不能把 level
集合差异误判为 RMCDHF 失败。

Job 597 的波函数交叉检查显示，AS1 的 overlap/radius 相关系数为
0.9731/0.99899，AS2 降为 0.8944/0.97848，且每轮波函数均已保留。结合 Job
596 的逐字节重复性，当前主线应审计 AS2 新增轨道进入 MPI 优化后的状态传递；
不能再把剩余的 Ni I 或 Ni/Ca-like 间隔偏差全部归因于节点护栏。B7 结果也表明
固定轨道 RCI 对间隔的修正具有体系依赖性。Fe I 的外部优化/未优化输入和 NIST
基准现在已经补齐，但尚未运行 QDIF 回归，因此仍不能声称四个体系的修复验证
已经完成。

## 结论分层与后续主线

前期测试证据需要按定位价值区分。最重要的结果是：不优化轨道时，无论
活性空间大小，能级顺序和谱项均正确；AS1 已有轨道的 `.w` 继承也正确；
逐个优化 AS2 新增轨道仍会出错。因此根因应位于新增轨道进入
`IMPROVmpi` 后的通用优化更新流程，而不是 RCI、Thomas–Fermi 初猜或某个
轨道之间的特殊耦合。

已确认的代码问题是 MPI 版 `IMPROVmpi` 遗漏 `ED1=PED(J)`，已在 Job 575
前修复。该修复是必要的状态初始化修正，但 Job 575 仍出现 `5d-`、`5d`、
`6s` 节点拒绝，说明它不是唯一根因。后续实验聚焦 LBL 的 MPI 优化路径，
逐步核对阻尼、重试、MPI `DA/NDA` 汇总、轨道数组交换和正交化调用。

节点相关测试的有效作用是排除错误方向：`MTP/MTP-10/MTP-20` 一致，排除
外层截断；`THRESH` 矩阵证明统一全局阈值不可行；极值和 `COUNT_TRACE`
说明 `COUNT` 是幅值筛选诊断，不等价于物理节点。它们不能解释能级顺序
异常，因此不再作为主修复方向。初始轨道估计方法 A/B 也未改变“不优化
正确、优化后错误”的事实，不能作为根因修复。

后续主线改为逐步审计 `SETLAGmpi → SOLVE → IMPROVmpi → DAMPCK/DAMPOR →
ORTHY → MATRIXmpi/NEWCOmpi` 的状态传递，记录 LBL MPI 路径中第一次出现
异常的轨道、能量和径向数组状态。护栏只保留为诊断辅助，不能用放宽节点
阈值替代优化流程修复。

### QDIF 单因素回归（Ni I，Job 605；结果见本文末节）

在前述作业结束后，静态对照发现 `rmcdhf90_mpi/setlagmpi.f90` 的两个同时
变分轨道分支始终使用 `OBQSUM`，遗漏了串行版和 `rmcdhf90_mem_mpi` 保留的
`QDIF > 0.1` 分支。Job 597 的 AS2 trace 中，`5d-/3d-`、`5d/3d`、
`5p-/2p-` 和 `5p/2p` 等配对的广义占据数差异远大于 0.1，因此该差异确实
会进入 LBL 拉格朗日乘子计算。该分支已恢复。为保持单因素顺序，已用只包含
该 QDIF 修复的源码构建专用 `build-qdif/bin/rmcdhf_mpi`（SHA-256
`0bcc19dc6670dc6f02285ec8b61ed27274904c2f4a8a88f57ced3b91ae94f9ff`）；此前
的 Jobs 590、596、597 均早于这次构建，不能作为 QDIF 修复后的数值结果。

新增单独脚本
`rmcdhf_test/test/rmcdhf_orbopt/slurm/run_qdif_fix_ni_46.sbatch`，保持
Ni I balanced、`ODAMP=-0.5`、node-progress 护栏、46 MPI rank 和
`/home/workstation2/caltmp` 不变，只重跑 AS1→AS2。结果目录为
`data/rmcdhf_test_data/results/qdif-fix-ni-<jobid>/`；提交后应比较
`orbopt_trace.csv`、每轮 `rwfn.out.iter*`、拉格朗日乘子日志、最终节点、
$J$ 顺序和 NIST 间隔，再决定是否把同一修改扩展到 Ni/Ca-like、Cl I 和 Fe I。

该脚本已经固定使用 `build-qdif/bin`，不会混入后续 `MPI_Gatherv` 或稀疏矩阵
索引修复。包含当前全部源码修改的正式构建位于 `build/bin/rmcdhf_mpi`，但尚未
用于数值回归；因此下一步仍只能先提交 Ni I QDIF 作业。

为保证后续单因素作业无需重新改动源码，已另外构建
`build-gatherv/bin/rmcdhf_mpi`（只含 QDIF 与 `MPI_Gatherv`，SHA-256
`8f3a65f72210972100ddc440e69baa4accb9d930746552be6e7e33c70c553456`），并新增
`run_gatherv_fix_ni_46.sbatch`。`run_sparse_index_fix_ni_46.sbatch` 使用当前
`build/bin`，其顺序固定为 QDIF → `MPI_Gatherv` → 稀疏列索引；在前一作业完成
并分析前，不提交后一个作业。

为避免复制 Fe I 外部验证目录中数 GB 的 CSF 文件，测试 runner 已增加可选的
`GRASP_TEST_DATA_ROOT`。默认工作区输入路径保持不变；Fe I 专用脚本从
`/home/workstation2/nvmesdd2T/rmcdhf_test_cal` 读取 `e1_vv3` 与 `e1_vv3_NV`
数据，输出仍位于本工作区的结果目录。Fe I 的 QDIF AS1→AS2 结果尚未产生，
因此当前仍不能宣称四个体系的修复回归已经完成。统一基准目录现在也已补齐
官方 NIST ASD 的 `data/nist_levels/Fe_I.csv`（847 条记录）；Fe I 导出中
`Lande` 字段的尾随 `?` 已由 `nist_data` 按不确定数值读取。

### `IMPROVmpi` 系数汇总的待验证问题

在 QDIF 回归尚未提交期间，进一步静态审计发现 `IMPROVmpi` 的
`NDA/DA` 汇总使用 `MPI_GATHER(..., ndcof_max, ...)`，而每个 rank 的实际
系数数量是 `NDCOF`。这会发送短列表之后的未初始化元素，不能保证
`DACON` 得到的合并系数只来自有效局部结果。工作区已经准备了使用
`MPI_Gatherv`、按各 rank 实际计数和位移汇总的独立修改，并已编译进当前
`build/bin/rmcdhf_mpi`，但尚未运行回归；因此当前 Ni I、Ni/Ca-like、Cl I
和 Fe I 的数值结果仍不能归因于或排除这一问题。它必须在 QDIF 单因素回归
之后单独验证。

### MPI 稀疏矩阵索引审计（待单因素回归）

在同一轮源码审计中发现，MPI Davidson 路径的两个稀疏矩阵例程存在相同的
跨步列偏移错误。`spicmvmpi.f90` 处理 `MYID+1, MYID+1+NPROCS, ...` 列时，
旧代码把 `IBEG` 延续到上一个本 rank 的列；`iniestmpi.f90` 则把
`JOFFSPAR` 延续到上一个本 rank 的列。两者都应按当前列的累计端点重新计算：
`IENDC(ICOL-1)+1` 或 `JCOL(J-1)+1`。

用独立 8×8 稀疏矩阵复算后，旧 `SPICMVmpi` 在 2/3/4 rank 的最大误差约为
`1.95/3.81/4.79`，修正后为机器精度；`INIESTmpi` 的 packed 矩阵在 2/3 rank
也有错误非零元，修正后与串行构造一致。该错误在 1 rank 时不会出现，因此
不能用旧的 1-rank 结果排除它。

当前已修改源码并已重建 `build/bin/rmcdhf_mpi`，但尚未运行数值回归：

- `rmcdhf_test/src/lib/mpi90/spicmvmpi.f90`；
- `rmcdhf_test/src/lib/mpi90/iniestmpi.f90`。

这组修改属于独立的“MPI 稀疏矩阵索引”因素，必须在 QDIF 和 `MPI_Gatherv`
回归后，用同一 Ni I AS1→AS2 夹具单独验证；在此之前，现有结果只能说明旧
二进制的行为，不能作为修正后的四体系证据。

### NVMe 成对验证集的输入审计

已检查 `/home/workstation2/nvmesdd2T/rmcdhf_test_cal` 中四个体系的优化与
不优化目录，并将 runner 的外部数据映射补齐：Ni/Ca-like 使用 `e1_cv`，Cl I
使用 `o1_cc1`，Fe I 使用 `e1_vv3`；相应的 `_NV` 目录作为不优化输入。外部
优化日志中的 varied list 也已核对，Cl I 的 AS1--AS5 为
`4s,4p,4d,4f`、`5s,5p,5d,5f,5g`、`6s,6p,6d,6f,6g`、
`7s,7p,7d,7f,7g`、`8s,8p,8d,8f,8g`，与工作区旧 Cl I 夹具的列表不同；
显式设置 `GRASP_TEST_DATA_ROOT` 时 runner 现在选择外部列表，默认工作区
测试保持原样。

外部归档 CSV 已先作为物理基准审查，显示需要由 MPI 重新计算解释的现象：

| 体系/阶段 | 优化归档谱序 | 不优化归档谱序 | 观察 |
|---|---|---|---|
| Ni I AS1 | `J2 < J3 < J4` | `J4 < J3 < J2` | 优化后倒序 |
| Ni I AS2 | `J2 < J3 < J4` | `J4 < J3 < J2` | 倒序持续 |
| Ni/Ca-like AS1 | `J2 < J3 < J4` | `J2 < J3 < J4` | AS1 未倒序 |
| Ni/Ca-like AS2 | `J2 < J4 < J3` | `J2 < J3 < J4` | 中间两级错序 |
| Cl I AS1 | `J1/2 < J3/2` | `J3/2 < J1/2` | 优化后倒序 |
| Cl I AS2--AS5 | `J1/2 < J3/2` | `J3/2 < J1/2` | 倒序持续 |

Fe I 外部优化目录目前没有对应的 `_rmcdhf.csv` 归档，而不优化目录有完整
AS1--AS5 CSV；因此 Fe I 必须通过同一 runner 重新生成优化结果，不能用文件
缺失当作计算失败。上述审计只说明外部数据与旧工作区基线不同，尚未替代
QDIF、`MPI_Gatherv` 和稀疏索引的单因素回归。

四个体系的外部 AS1→AS2 优化/不优化脚本已经分别保存于
`rmcdhf_test/test/rmcdhf_orbopt/slurm/`，名称分别为
`run_external_{ni,nica,cl,fe}_{optimized,nv}_46.sbatch`。它们固定使用正式
`build/bin`，因此必须在 Ni I 的三项单因素回归通过后再提交。


## 14. Job 605：QDIF 单因素 Ni I 回归（2026-09-08）

Job 605 使用 `test/rmcdhf_orbopt/slurm/run_qdif_fix_ni_46.sbatch`，固定 46 个
MPI rank、`ODAMP=-0.5`、node-progress 护栏、`mpi/openmpi-x86_64`、
`grasp/grasp_2990_NNNP` 和 `/home/workstation2/caltmp/mpi_tmp`。它调用只包含
QDIF 修复的 `build-qdif/bin/rmcdhf_mpi`，二进制 SHA-256 为
`0bcc19dc6670dc6f02285ec8b61ed27274904c2f4a8a88f57ced3b91ae94f9ff`。原始日志和
结果分别为 `data/rmcdhf_test_data/log/605_rmcdhf-ni-qdif.log` 与
`data/rmcdhf_test_data/results/qdif-fix-ni-605/`。

AS1 有完整的退出文件和 CSV：`as1/rmcdhf.exitcode` 为 `0`，并生成了
`e1_vv2as1_rmcdhf.csv`、`orbopt_trace.csv`、`orbopt_summary.csv`、
`rwfn_metrics.csv` 以及 29 轮 `rwfn.out.iter*`。最终三条 Ni I 精细结构能级为：

| 能级 | 计算累计间隔 (cm⁻¹) | NIST 累计间隔 (cm⁻¹) | 计算−NIST (cm⁻¹) |
| --- | ---: | ---: | ---: |
| $^3F_3-{}^3F_4$ | 1358.20 | 1332.164 | +26.036 |
| $^3F_2-{}^3F_4$ | 2276.86 | 2216.550 | +60.310 |

相邻间隔为 `1358.20` 和 `918.66 cm⁻¹`，谱序仍是
$^3F_4 < {}^3F_3 < {}^3F_2$。这些数值与 Job 590/596 的 AS1 结果一致，
QDIF 修复没有在 AS1 产生可见差异。

AS2 没有完成 wrapper 的正常收尾。文件系统最后只保存到第 23 轮的中间输出：
`orbopt_trace.csv` 有第 23 轮的 level-energy 快照，`rwfn.out.iter023` 和
`rwfn.out` 已写出，但没有 `as2/rmcdhf.exitcode`、`status.csv`、最终
`*_rmcdhf.csv` 或非空 `rmcdhf.sum`，stdout 也没有 MPI execution-complete
标记。第 23 轮快照换算得到的临时间隔约为 `1375.90` 和 `2291.95 cm⁻¹`
（相邻第二间隔约 `916.05 cm⁻¹`）；它不是最终结果，不能用于修复验收。AS2
在第 3 轮对 `5p-` 的一次 `nodes_progress` 拒绝，以及截至中断前的能量和
轨道更新，与 Job 596 的对应 trace 相同；目前没有证据表明 QDIF 已改变
实际 LBL 更新路径。由于本次会话的 Slurm socket 不可用，无法从 `sacct` 读取
终态码，因此将 Job 605 归类为“AS1 完成、AS2 未完成”，而不是算法通过或算法
失败。

为避免分区默认时限再次截断 AS2，QDIF、`MPI_Gatherv`、稀疏索引以及外部
NVMe 回归脚本均已增加显式 `#SBATCH --time=02:00:00`。下一次必须先重新提交
QDIF 脚本并确认 AS1、AS2 都存在退出码、`status.csv` 和最终 CSV；在此之前
不得提交 `run_gatherv_fix_ni_46.sbatch`。

## 15. Job 607：QDIF 单因素 Ni I 完整回归（2026-09-09）

Job 607 使用 `run_qdif_fix_ni_46.sbatch`、46 MPI ranks、02:00:00 时限和
`build-qdif/bin/rmcdhf_mpi`。AS1、AS2 均完整收尾，`rmcdhf.exitcode=0`，最终
CSV、`rmcdhf.sum`、`status.csv`、逐轮 `rwfn.out.iter*`、trace/summary 和
metrics 均存在；总状态为 `complete`。AS2 共 33 轮，最后节点计数为 3、2、5，
未触发 fallback。

AS1 顺序为 `³F4 < ³F3 < ³F2`，累计间隔 `1358.20/2276.86 cm⁻¹`，相对 NIST
`1332.164/2216.550` 偏差 `+26.036/+60.310 cm⁻¹`；相邻间隔为
`1358.20/918.66 cm⁻¹`。AS2 顺序同样为 `³F4 < ³F3 < ³F2`，累计间隔
`1375.88/2291.99 cm⁻¹`，偏差 `+43.716/+75.440 cm⁻¹`；相邻间隔
`1375.88/916.11 cm⁻¹`，第二相邻间隔相对 NIST `884.386` 偏差 `+31.724 cm⁻¹`。
与 Jobs 590、596、597 的 SOLVE→DAMPOR→ORTHY 路径和早期节点/半径行为一致，
未见 QDIF 导致的首次可辨差异；该单因素未修复精细结构偏差。现已满足完整结果
闸门，可以提交 MPI_Gatherv 单因素回归。

## 16. Job 608：MPI_Gatherv 单因素 Ni I 回归（2026-09-09）

Job 608 的 AS1、AS2 均完整收尾（exitcode 0，status complete，最终 CSV/summary/trace/metrics 齐全）。AS2 最终顺序 `³F4 < ³F3 < ³F2`，累计间隔 `1375.88/2291.99 cm⁻¹`，相对 NIST 偏差 `+43.716/+75.440 cm⁻¹`，与 Job 607 完全一致；未观察到 Gatherv 修复引起的首次轨迹或物理结果差异。因此该单因素未修复偏差，现进入稀疏列索引单因素。

## 17. Job 609：稀疏列索引单因素 Ni I 回归（2026-09-09）

Job 609 AS1/AS2 均 wrapper 完整收尾（exitcode 0、status complete），但物理结果明显损坏：AS1 首三能级为 `³P2, ³F2, ⁵G2`（累计 0/34664.59/195552.43 cm⁻¹），AS2 首三能级为 `³F2, ³P2, ³P2`（累计 0/28314.65/108684.39 cm⁻¹），不再是目标 `³F4,³F3,³F2`。因此稀疏列索引修复引入严重谱项/排序错误；这是算法失败而非超时或 wrapper 失败。四体系外部成对作业已提交（Jobs 610–617）。


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

## 2026-09-11：Jobs 618–621 完成

四个修正后的外部优化作业均 `COMPLETED 0:0`，每个均完成 AS1→AS2，两个阶段
`rmcdhf.exitcode=0`、`status,complete`、最终 CSV 和 archive comparison 齐全。
均使用 Job 608 的 `build-gatherv/bin/rmcdhf_mpi`；不优化作业未重跑。

- **Job 618 Ni I**：AS1 首三项 `³F2 < ³F3 < ³F4`，间隔 1314.83、累计
  2734.45 cm⁻¹；AS2 为同序，间隔 4243.88、累计 8440.35 cm⁻¹。
- **Job 619 Ni/Ca-like**：AS1 `³F2 < ³F3 < ³F4`，累计 1287.21 cm⁻¹；
  AS2 为 `³F2 < ³F4 < ³F3`，累计 62.36、68.08 cm⁻¹，出现中间两级倒序/塌缩。
- **Job 620 Cl I**：AS1 `²P1/2 < ²P3/2`，间隔 989.39 cm⁻¹；AS2 同序，
  间隔 1521.60 cm⁻¹。此次不再是 100 轮未收敛；两阶段均完整收尾。
- **Job 621 Fe I**：AS1 `⁵D4 < ⁵D3 < ⁵D2 < ⁵D1 < ⁵D0`，累计到 J=0 为
  1006.11 cm⁻¹；AS2 同序，累计 1137.60 cm⁻¹。

四组 archive comparison 均显示 CSF 数和能级标签匹配，能量差为零；2990 点径向
网格差异按 NNNP 规则放行。Ni/Ca-like AS2 的错序仍是物理结果，不能因脚本成功
或归档匹配而判为修复通过。

## 2026-09-11：Jobs 631/632 定向新增轨道隔离矩阵完成

定向 AS2 隔离测试已经完成。Job 631（Ni I）和 Job 632（Ni/Ca-like）均为
`COMPLETED 0:0`，13 个变体的 `rmcdhf.exitcode` 全部为 `0`。结果目录为
`data/rmcdhf_test_data/results/orbital-iso-nii-631/` 和
`data/rmcdhf_test_data/results/orbital-iso-nica-632/`。所有变体的
`energy_converged=true`，而 `orbital_converged=false`；这是当前 legacy 能量
停止判据的结果，不能解释为新增轨道已经物理收敛。

`freeze_new` 只固定 AS2 新增轨道，同时保留 AS1 的 varied list，因此不等同于
已有的 `_NV` 不优化计算。每个变体均复用同一 AS1 波函数，仅改变 AS2 的新增轨道
选择。汇总器现在将 Ni I NIST 参考固定为 `J=4`，先把计算结果归一到计算的
`J=4`，再计算 NIST 误差；Ni/Ca-like 以 `J=2` 为零点。

### Ni I（Job 631）

表中的 `E3-E4` 和 `E2-E4` 是以计算的 `J=4` 为零点的有符号间隔，误差依次为
相对 NIST `1332.164` 和 `2216.550 cm⁻¹` 的计算值减参考值。

| 变体 | 最终顺序 | E3-E4 | E2-E4 | 误差(J3/J2) cm⁻¹ | 首次低重叠 | 接受后节点变化 |
|---|---|---:|---:|---|---|---:|
| `all_new` | J2 < J3 < J4 | -4196.47 | -8440.35 | -5528.63 / -10656.90 | 6s, 0.04646 | 2 |
| `freeze_new` | J2 < J3 < J4 | -1310.35 | -2523.12 | -2642.51 / -4739.67 | — | 0 |
| `only-4f` | J2 < J3 < J4 | -3795.65 | -7565.98 | -5127.81 / -9782.53 | 4f, 0.02814 | 0 |
| `only-5d` | J2 < J3 < J4 | -1507.03 | -2917.87 | -2839.19 / -5134.42 | — | 1 |
| `only-5p` | J2 < J3 < J4 | -1453.59 | -2797.19 | -2785.75 / -5013.74 | — | 0 |
| `only-6s` | J2 < J3 < J4 | -1332.43 | -2572.34 | -2664.59 / -4788.89 | 6s, 0.04646 | 1 |

Ni I 的所有变体都保持错误的 `J2 < J3 < J4`，因此只优化一个新增轨道也足以
进入错误分支；问题不能归结为只有全部新增轨道同时更新才发生。`all_new` 和
`only-4f` 的首轮候选重叠已经低于 `0.1`，`5d`、`6s` 还出现接受后节点变化。

### Ni/Ca-like（Job 632）

这里的间隔以 `J=2` 为零点，NIST 参考为 `E3-E2=1880` 和 `E4-E2=4070 cm⁻¹`。

| 变体 | 最终顺序 | E3-E2 | E4-E2 | 误差(J3/J4) cm⁻¹ | 首次顺序变化 | 接受后节点变化 |
|---|---|---:|---:|---|---|---:|
| `all_new` | J2 < J4 < J3 | 68.08 | 62.36 | -1811.92 / -4007.64 | iter=3 | 0 |
| `freeze_new` | J2 < J3 < J4 | 1072.87 | 2387.90 | -807.13 / -1682.10 | — | 0 |
| `only-5d` | J2 < J3 < J4 | 992.93 | 2202.55 | -887.07 / -1867.45 | — | 0 |
| `only-5f` | J2 < J3 < J4 | 923.01 | 2033.31 | -956.99 / -2036.69 | — | 0 |
| `only-5g` | J2 < J3 < J4 | 409.52 | 881.40 | -1470.48 / -3188.60 | — | 0 |
| `only-5p` | J2 < J3 < J4 | 1047.06 | 2330.78 | -832.94 / -1739.22 | — | 0 |
| `only-5s` | J2 < J3 < J4 | 1068.84 | 2378.99 | -811.16 / -1691.01 | — | 0 |

Ni/Ca-like 中每个单轨道变体都保持正确顺序，只有 `all_new` 在第 3 轮发生顺序
变化并塌缩到几十 `cm⁻¹`。这里没有低重叠和节点变化，优先级应放在多轨道 EOL
权重、CI 根耦合和根身份跟踪，而不是继续调单轨道节点护栏。

### 隔离矩阵结论

这批结果完成了最小判别实验，但没有完成物理修复。Ni I 证明新增轨道优化路径
本身即可触发错误谱序；Ni/Ca-like 证明多轨道联合更新还会产生单轨道测试看不到的
根/能级错序。当前 `orbopt_trace.csv` 没有 CI 向量重叠匹配，最终 `LSJ` 也不能
单独证明谱项身份，因此下一步只需实现 CI 根跟踪、整轮回退和轨道/能量/目标态
身份三重收敛门槛；不再重复 Jobs 631/632 的输入或变体。

## 21. 现有 `rscf92.dbg` 的离线 CI 根匹配（2026-09-11）

没有重新提交作业。新增分析脚本
`rmcdhf_test/test/rmcdhf_orbopt/analyze_ci_root_trace.py` 从已有的
`rscf92.dbg` 重建每轮 CI 向量，用同一 $J\pi$ 块内的最大绝对向量重叠做一一匹配，
并与 `orbopt_trace.csv` 的逐轮能量合并。每个变体的明细位于对应结果目录下的
`ci_root_analysis/ci_root_matches.csv`、`ci_root_summary.csv`、
`ci_energy_order.csv` 和 `ci_root_report.md`。

13 个已有变体全部解析成功，均未出现非恒等 CI 根匹配：

| 体系/变体 | CI 周期 | 最低匹配重叠 | 非恒等匹配 | 全局能级顺序变化 |
|---|---:|---:|---:|---|
| Ni I `all_new` | 21 | 0.854815 | 0 | 无 |
| Ni I `freeze_new` | 8 | 0.999700 | 0 | 无 |
| Ni I `only-4f` | 37 | 0.977067 | 0 | 无 |
| Ni I `only-5d` | 19 | 0.708187 | 0 | 无 |
| Ni I `only-5p` | 24 | 0.998448 | 0 | 无 |
| Ni I `only-6s` | 11 | 0.998959 | 0 | 无 |
| Ni/Ca-like `all_new` | 7 | 0.970475 | 0 | 第 1、3 轮 |
| Ni/Ca-like `freeze_new` | 4 | 1.000000 | 0 | 无 |
| Ni/Ca-like `only-5d` | 9 | 0.999791 | 0 | 无 |
| Ni/Ca-like `only-5f` | 13 | 0.999227 | 0 | 无 |
| Ni/Ca-like `only-5g` | 9 | 0.989907 | 0 | 无 |
| Ni/Ca-like `only-5p` | 9 | 0.999924 | 0 | 无 |
| Ni/Ca-like `only-5s` | 7 | 0.999961 | 0 | 无 |

Ni I 的最低重叠出现在第 0→1 轮的 block 1（$J=0$）第二根；Ni I
`only-5d` 的该重叠降至 0.708187，但最大重叠匹配仍保持原根编号，没有发生根
置换。这个异常不在目标 $^3F_{2,3,4}$ 的 block 3 内。重叠阈值 0.90 仅作为检查
标记，不能直接作为拒绝门槛。

Ni/Ca-like `all_new` 在第 1 轮和第 3 轮发生全局能量顺序变化，但每个 $J\pi$
块内的 CI 根匹配仍为恒等匹配，且最低向量重叠为 0.970475。这说明目前没有证据
表明错序来自未跟踪的 CI 根交换；更符合轨道更新改变不同 $J$ 态能量、EOL 权重或
近简并能级相对位置的机制。下一步应保留该离线根跟踪诊断，优先做固定轨道 CI/RCI
对照和轨道更新贡献分析，暂不实现只针对根交换的自动回退。

## 22. Jobs 631/632 的固定轨道 CI/RCI 成对对照（Job 635）

第 2 步使用
`test/rmcdhf_orbopt/slurm/run_fixed_orbital_rci_pair.sbatch` 完成。对每个体系，
先用对应 `_NV` AS1 波函数和 AS2 CSF 运行 `rangular_mpi → rwfnestimate` 生成
TF AS2 波函数，再对 TF 波函数和 Job 631/632 `all_new` 最终波函数运行相同的
固定轨道 RCI。四个 `b7_rci.csv` 均已生成；Job 635 的调度脚本最后只因收尾哈希
路径错误返回 1，四个 RCI 本身均成功，结果目录中的 `status.csv` 和
`all_outputs.sha256` 已补齐。CSF 哈希与输入一致：Ni I 为
`4630ea1bf4ec59863b4971868fd8c12d35fe4e218efac41729dc52e32520fc38`，
Ni/Ca-like 为 `5c82085ad4ba31358322c771b724c79356b8a0eff3c23b132307d18cf295ec33`。

| 体系/轨道 | 三个目标态的能量顺序 | 第一间隔（cm⁻¹） | 第二间隔（cm⁻¹） | 相对 NIST 误差（cm⁻¹） |
|---|---|---:|---:|---:|
| Ni I，TF | `J4 < J3 < J2` | `E3-E4=1305.08` | `E2-E4=2168.02` | `-27.08 / -48.53` |
| Ni I，优化后 | `J2 < J3 < J4` | `E3-E4=-4262.78` | `E2-E4=-8606.07` | `-5594.94 / -10822.62` |
| Ni/Ca-like，TF | `J2 < J3 < J4` | `E3-E2=1833.33` | `E4-E2=4031.32` | `-46.67 / -38.68` |
| Ni/Ca-like，优化后 | `J4 < J3 < J2` | `E3-E2=-102.85` | `E4-E2=-369.73` | `-1982.85 / -4439.73` |

这批 TF 使用 `_NV` AS1，而 631/632 的优化波函数继承的是优化 AS1，因此它是跨
基线对照，不能单独把差异归因于 AS2 新轨道。它仍然说明两条完整计算链的固定
RCI 结果差异很大；Ni I 的固定 RCI 与 RMCDHF `all_new` 同样为 `J2 < J3 < J4`。
Ni/Ca-like 的固定 RCI 为 `J4 < J3 < J2`，RMCDHF 为 `J2 < J4 < J3`，表示 EOL
自洽松弛还会继续改变各 $J$ 态的相对移动。

四个固定 RCI 的前三个态仍为同一 `^3F_J` 谱项，主导组态权重约为 `0.90–0.99`；
本批没有观察到目标态直接变成另一组态，也没有新的 CI 根交换证据。严格归因见
下一节。

## 23. 同一 AS1 的严格 TF/优化 AS2 对照（Job 636）

Job 636 复用 Jobs 631/632 的 `previous.w`，只用 `rwfnestimate` 生成 AS2 TF
轨道，再运行同一套固定 RCI；优化轨道 RCI 直接复用 Job 635 的 CSV。结果目录为
`data/rmcdhf_test_data/results/matched-tf-rci-636/`，包括
`decomposition_summary.csv`、AS1 对照 `radial_as1_nii.csv`/
`radial_as1_nica.csv`，以及 AS2 对照 `radial_nii.csv`/`radial_nica.csv`。

| 体系/波函数 | 目标态顺序 | 第一间隔（cm⁻¹） | 第二间隔（cm⁻¹） |
|---|---|---:|---:|
| Ni I，`_NV` AS1 + TF AS2（Job 635） | `J4 < J3 < J2` | `1305.08` | `2168.02` |
| Ni I，优化 AS1 + TF AS2（Job 636） | `J2 < J3 < J4` | `E3-E4=-1411.04` | `E2-E4=-2730.88` |
| Ni I，优化 AS1 + 优化 AS2 | `J2 < J3 < J4` | `E3-E4=-4262.78` | `E2-E4=-8606.07` |
| Ni/Ca-like，`_NV` AS1 + TF AS2（Job 635） | `J2 < J3 < J4` | `1833.33` | `4031.32` |
| Ni/Ca-like，优化 AS1 + TF AS2（Job 636） | `J2 < J3 < J4` | `893.62` | `1953.59` |
| Ni/Ca-like，优化 AS1 + 优化 AS2 | `J4 < J3 < J2` | `E3-E2=-102.85` | `E4-E2=-369.73` |

这一步把两个贡献分开了。Ni I 的错误顺序在优化 AS1、AS2 仍为 TF 时已经出现；
AS2 优化继续扩大错误间隔。Ni/Ca-like 的优化 AS1 先压缩间隔但保留顺序，AS2
优化后才反转。TF 到优化 AS2 的径向变化集中在新增轨道：Ni I 的 `4f/5d/5p/6s`
重叠约为 `0.008/0.205/0.345/-0.082`；Ni/Ca-like 的 `5s–5g` 重叠约为
`0.12–0.32`，平均半径缩小约 2.8–3.2 倍，部分节点改变。相对于 `_NV` AS1，
Ni I 的 AS1 `4d/5s` 重叠约为 `0.397/-0.670`，Ni/Ca-like 的 `4s–4f` 重叠约为
`0.865–0.957`。这说明下一步应先固定经过验证的 AS1/TF 基线，再分别测试 AS1
优化和 AS2 分组优化；不能把所有异常都归咎于 AS2 新轨道，也不能直接恢复全量
轨道优化。

## 24. 631/632 轨迹与 636 分解的离线合并（2026-09-11）

本步没有提交新作业。新增 `test/rmcdhf_orbopt/analyze_as1_as2_contributions.py`，
读取已有的 `orbopt_trace.csv`、`ci_root_analysis/` 和
`matched-tf-rci-636/decomposition_summary.csv`，输出
`matched-tf-rci-636/as1_as2_contribution.csv` 与 `as1_as2_contribution.md`。
能级首次变化按相对第 0 轮超过 `1 cm⁻¹` 统计；半径比为候选半径/旧半径。

| 体系 | 变体 | 首次间隔变化 | 最低接受重叠 | 最大半径因子 | 接受后节点变化 | 最终顺序 | CI 全局能量顺序变化 |
|---|---|---:|---:|---:|---:|---|---:|
| Ni I | `only-4f` | iter=1 | 0.0281 | 8.55 | 0 | `J2<J3<J4` | 0 |
| Ni I | `only-5d` | iter=1 | 0.1970 | 4.40 | 1 | `J2<J3<J4` | 0 |
| Ni I | `only-5p` | iter=1 | 0.4655 | 2.47 | 0 | `J2<J3<J4` | 0 |
| Ni I | `only-6s` | iter=1 | 0.0465 | 3.15 | 1 | `J2<J3<J4` | 0 |
| Ni/Ca-like | `only-5s` | iter=1 | 0.4021 | 1.84 | 0 | `J2<J3<J4` | 0 |
| Ni/Ca-like | `only-5p` | iter=1 | 0.4319 | 1.92 | 0 | `J2<J3<J4` | 0 |
| Ni/Ca-like | `only-5d` | iter=1 | 0.3955 | 2.25 | 0 | `J2<J3<J4` | 0 |
| Ni/Ca-like | `only-5f` | iter=1 | 0.2034 | 2.33 | 0 | `J2<J3<J4` | 0 |
| Ni/Ca-like | `only-5g` | iter=1 | 0.3000 | 2.78 | 0 | `J2<J3<J4` | 0 |

Ni I 的每个单轨道变体在第 1 轮就改变相对间隔，但 CI 向量匹配没有非恒等分配；
这支持“轨道更新改变不同 $J$ 态的相对能量”，不支持纯粹的 CI 根编号交换。
Ni/Ca-like 的单轨道变体均保持顺序，只有 `all_new` 在第 3 轮变为 `J2<J4<J3`；
该变体的 CI 最低匹配重叠为 `0.9705`，全局能量排序在第 1、3 轮改变，而根匹配仍为恒等。

结合 Job 636 的严格固定 RCI 对照，Ni I 的倒序在优化 AS1、AS2 仍为 TF 时已经存在，
AS2 优化继续放大错误间隔；Ni/Ca-like 则由优化 AS1 先压缩间隔，AS2 全量优化再触发反转。
因此目前不再重复 Jobs 631/632/635/636，也不把固定新增轨道误称为物理修复；下一步应在
现有轨道优化代码中实现整轮回退、目标态 CI 身份跟踪和轨道/能量/身份三重收敛门槛。

## 25. 整轮回退实现后的本地验证（2026-09-11）

已完成实现但没有提交新作业。`orbopt_round_state.f90` 保存每个 SCF 轮次的轨道、
CI 向量/能量、块平均能量、权重和 CI 根标签；`scfmpi.f90` 在
`MATRIXmpi → NEWCOmpi` 后做整轮候选检查，失败时恢复快照、增加负阻尼、清除收敛
状态并覆盖写回恢复的波函数，达到回退上限则停止。`round_decision` 事件已经加入
`orbopt_trace.csv`，可审计接受/回退、最低向量重叠、能量顺序变化和根匹配原因。

新增环境变量均默认关闭：
`GRASP_ROUND_ROLLBACK_GUARD`、`GRASP_ROUND_REJECT_ENERGY_ORDER`、
`GRASP_MIN_STATE_OVERLAP`、`GRASP_MAX_ROUND_ROLLBACKS`。启用护栏时严格收敛还要求
本轮 CI 身份稳定；未启用时保留原有收敛语义和输入交互。

本地验证命令为加载 `mpi/openmpi-x86_64` 后执行
`cmake --build build -j2 --target rmcdhf_mpi`，结果为成功；`git diff --check` 也已
通过。本次没有重新运行既有作业，因此这里不能宣称 Ni I、Ni/Ca-like 或 Cl I 的物理
顺序已经修复，后续只需用三个缩小夹具验证新开关的回退和三重收敛日志。

## 26. Job 638：整轮回退护栏验证

结果目录：`data/rmcdhf_test_data/results/round-guard-minimal-638/`。三组算例均使用
46 个 MPI rank、严格收敛和整轮回退护栏（最低 CI 重叠阈值 0.85，最多 3 次回退）。

| 算例 | RMCDHF 退出码 | 回退/接受轮次 | 最低重叠 | 结果判定 |
|---|---:|---:|---:|---|
| Ni I AS2 | 1 | 4 / 0 | 0.154087 | 保护性停止；第 4 次回退超过上限，无最终 CSV |
| Ni/Ca-like AS2 | 1 | 4 / 3 | 0.071677 | 第 1–3 轮回退，第 4–6 轮接受，第 7 轮再回退并停止，无最终 CSV |
| Cl I AS1 | 0 | 0 / 100 | 0.999323 | RMCDHF 正常完成但到 100 轮上限，严格收敛检查未通过；有最终 CSV |

Ni I 的四个 `round_decision` 均为 `state_overlap+root_assignment`，重叠为
0.159913、0.181285、0.154087、0.157074。Ni/Ca-like 前三轮重叠为 0.071677、
0.105908、0.391092，随后接受轮为 0.981732、0.881384、0.952346，第 7 轮降至
0.618559。Cl I 的 100 轮全部为 `accepted`，重叠范围 0.999323–0.999837，未见
根匹配变化或能级顺序变化；但末轮 `convg_orbital=false`、`convg_energy=false`。

旧版 `status.csv` 中 Cl I 的 `runner_exit=1` 只代表严格后处理检查失败，不代表
RMCDHF 异常。为避免混淆，`run_data_case.sh` 现在额外写出
`convergence_check.exitcode`，最小护栏脚本的状态表同时报告
`rmcdhf_exit` 和 `convergence_exit`，并将结果分类为
`rmcdhf_failed`、`rmcdhf_success_strict_not_converged`、`strict_converged` 或
后处理失败。Job 638 的数值结果没有因该报告修正而改变。

同时，`orbopt_trace.csv` 已增加 `round_accepted`、`round_order_changed`、
`round_nonidentity`、`round_rollback_count`、`round_min_overlap` 和对应控制参数
的专用列；Job 638 的旧 trace 仍使用原有列解释，后续作业不再依赖通用收敛列承载
回退语义。

本作业证明了护栏能够阻止低重叠/非恒等匹配候选生成最终物理文件，也能在
Ni/Ca-like 的候选恢复稳定后暂时接受更新；它没有证明能级顺序已经恢复。Ni I 和
Ni/Ca-like 都以保护性停止结束，下一步仍需修正按目标 $J\pi$ 块进行的能量比较、
全局一一 CI 根匹配和专用 trace 字段，然后再决定是否运行新的候选修复测试。

## 27. 目标态范围修正与离线验收（2026-09-12）

已完成目标态范围修正。`GRASP_ROUND_TARGET_STATES` 使用 `NEWCOmpi` 的 1-based
全局 state index；配置后，最低 CI 重叠门槛只作用于目标根，辅助根的 assignment
仍完整记录但不会单独触发回退。目标态能量先按 CI overlap assignment 映射回旧根，
再比较相对顺序，因此目标态位于不同 $J\pi$ block 时仍能保持身份一致。未配置
目标态时保留全根重叠检查，并自动关闭能级顺序拒绝。

最小护栏脚本默认将 Ni I、Ni/Ca-like 的目标态设为 `3,4,7`，Cl I 设为 `1,2`；
可用 `GRASP_ROUND_TARGET_STATES_NI_I`、`GRASP_ROUND_TARGET_STATES_NICA` 和
`GRASP_ROUND_TARGET_STATES_CL_I` 覆盖。trace 控制行记录目标态字符串；新增的
`round_identity_stable` 列让严格收敛检查可以验证目标态身份门槛，同时保留旧 trace
兼容性。

离线脚本 `test/rmcdhf_orbopt/test_round_state_logic.py` 已通过五项检查：恒等匹配、
合法 root 交换、Kuhn--Munkres 全局最优匹配、跨 block 目标态能量顺序，以及非目标
辅助根低重叠不触发门槛。更新后的 `rmcdhf_mpi` 已重新构建成功；本节没有重复既有
作业，下一步再用一次最小护栏作业验证真实 trace。

## 28. Job 639 复核：测试输入与 trace 输出缺陷（2026-09-12）

Job 639 不作为有效物理验证。Ni I 与 Ni/Ca-like 的默认目标态误设为 `4,5,6`，
它们都在 $J=3$ block；目标 $^3F_J$ 的正确全局索引是 `3,4,7`，Cl I 的 `1,2`
保持正确。另一个问题是目标态字符串中的逗号未在 trace 中引用，导致所有算例的
`orbopt_trace.csv` 控制行 schema 错位，`compare_rmcdhf.py` 报
`inconsistent CSV schema`；Cl I 的 RMCDHF 虽退出 0，后处理仍因该错误中断。

在错误目标集合下，Ni I 4 轮均因 `energy_order+state_overlap` 回退；Ni/Ca-like
前三轮回退、随后 32 轮接受，最终超时退出 124；Cl I 100 轮均接受并达到迭代上限。
这些数字只能记录故障表现，不能证明护栏已修复排序。

现已将默认目标态改为 `3,4,7`，为含逗号的 trace 字段增加 CSV 引号，并把状态表
中的 `convergence_exit=not_run` 归类为后处理失败。修正后的本地构建、脚本语法和
五项离线 assignment 测试均通过；Job 639 的结果不再复用。
