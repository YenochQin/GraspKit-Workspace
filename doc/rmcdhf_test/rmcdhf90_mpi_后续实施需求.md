# `rmcdhf90_mpi` 后续实施需求

本文档用于记录继续实施
[`rmcdhf90_mpi_轨道优化调试方案.md`](./rmcdhf90_mpi_轨道优化调试方案.md)
所需的输入、输出和验收材料。当前已有 Cl I 与 Ni/Ca-like 的诊断结果，已经足以
证明单侧伙伴优化与精细结构异常相关；若要继续修改 `ORTHY`、`SOLVE`、MPI 归约或
轨道接受流程，则需要可独立复现的原始问题数据。

## 1. 最小可复现数据集

请为每个需要继续分析的体系准备一个独立目录，至少包含：

```text
isodata
rcsf.inp 或实际使用的 *.c
rwfn.inp 或初始 *.w
完整 stdin
rmcdhf.stdout（以及 stderr，如单独保存）
rmcdhf.log
rmcdhf.sum
rwfn.out
rmix.out（如果程序生成）
```

如果计算经过 `rangular`、`rwfnestimate`、`rsave`、`jj2lsj` 或 `rhfs`，也请保留这些
步骤的输入、输出和日志。文件可以压缩存放，但必须保留原始文件名和目录层级。

不需要提交整个大型计算目录；优先提供能够在一台节点上独立重现异常的最小活动空间。
大型中间文件可以放在服务器实验目录，只需在需求记录中给出绝对路径、大小和校验值。

## 2. 程序与运行环境信息

### 2.1 当前计算的统一来源

目前已提供的计算结果均在同一台服务器上完成，并通过 Slurm `sbatch` 脚本提交。
结果目录应保留实际使用的 sbatch 脚本；脚本中的 module 加载、MPI 启动方式、任务数、
CPU 绑定和环境变量都是结果来源审计的一部分，不能只保留最终的 `rmcdhf.sum`。

当前服务器基础环境为：

```text
MPI module：mpi/openmpi-x86_64
Fortran：本机 gfortran
版本：gcc version 15.2.1 20260123 (Red Hat 15.2.1-7) (GCC)
BLAS：FlexiBLAS 管理的 OpenBLAS OpenMP 后端
动态库：libflexiblas_openblas-openmp.so
```

### 2.2 两个 GRASP module 的区别

计算使用的 GRASP 来自 `grasp/grasp_2990_NNNP` 或 `grasp/grasp_raw`，必须在结果记录中明确写出。`grasp_raw` 是从 GitHub 完整 clone 后直接编译的程序；`grasp_2990_NNNP` 则是在同一源码基础上，将 `rmcdhf_test/src/lib/libmod/parameter_def_M.f90` 中的 `NNNP` 从 590 改为 2990 后重新编译得到。`NNNP` 会影响径向网格上限和数组规模，因此两者不能混入同一个 golden baseline。

每个数据集必须同时记录：

```text
程序：rmcdhf 或 rmcdhf_mpi
源码仓库与 commit
可执行文件路径和 sha256
编译器版本
MPI 实现与版本
BLAS/LAPACK 或 FlexiBLAS 信息
节点名、MPI 进程数、OpenMP 线程数
所有 GRASP_* 环境变量
GRASP_MODULE 或 GRASP_PATH
```

建议直接保存以下命令的输出：

```sh
git log -1 --oneline
sha256sum path/to/rmcdhf_mpi
gfortran --version
mpirun --version
module list
env | sort | grep -E '^(GRASP|OMP|OPENBLAS|MPI)'
```

对于已提供的服务器计算结果，还必须记录实际 sbatch 脚本路径或脚本副本、所用 GRASP module、NNNP 值、`mpi/openmpi-x86_64`、gfortran 15.2.1 以及 `libflexiblas_openblas-openmp.so`。脚本中的 `srun --mpi=pmix`、`--cpu-bind`、`--ntasks-per-node` 和 `OMP_NUM_THREADS` 等参数必须原样保留，因为它们会影响 MPI 进程布局和数值可重复性。

Python 分析和校验应使用 `graspkit-tools` 目录中由 `uv` 创建的环境，不使用系统 Python
或单独创建的临时环境。推荐命令为：

```sh
cd graspkit-tools
uv run python -m pytest ...
uv run python ../rmcdhf_test/test/rmcdhf_orbopt/compare_mpi_reproducibility.py ...
```

如果已经激活该环境，也必须在运行记录中保留 `graspkit-tools/.venv/bin/python3` 的
路径和 `uv` 锁定的依赖版本。

### 2.3 CSF 文件命名与数量审计

原始数据计算过程中，使用 `rcsfgenerate` 为每个活性空间生成的 CSF 文件命名为：

```text
${conf}as${as}_raw.c
```

计算实际读取或归档保存的 CSF 文件命名为：

```text
${conf}as${as}.c
```

因此，`_raw.c` 是活性空间生成阶段的输入快照，`.c` 是后续计算阶段实际使用或保存的
CSF 文件。两者必须同时保留或在 manifest 中明确来源，不能仅根据 basename 判断它们
内容相同。

计算过程中可能调用 GRASP 的 `rcsfinteract` 工具。若调用过，`rcsfinteract` 可能筛选、
重排或扩展 CSF，使计算前后的 CSF 数量不一致。每个活性空间都应记录：

- `rcsfgenerate` 生成的 `_raw.c` CSF 数量；
- `rcsfinteract` 的输入、输出、完整 stdout/stderr 和调用参数；
- `rmcdhf`/`rci` 实际读取的 `.c` 文件及其 CSF 数量；
- 计算前后数量差异及产生差异的具体步骤。

在 `compare_sum.py` 或其他汇总脚本中，CSF 数量比较必须注明比较的是 `_raw.c`、
`rcsfinteract` 输出还是最终 `.c`，避免把正常的 CSF 筛选误判为数据损坏或程序不一致。

## 3. 问题描述

每个问题案例应明确写出：

- 预期结果与实际结果；
- 发生异常的 J、宇称和能级间隔；
- 从第几轮 SCF 开始异常；
- 实际 varied list，特别是 `nl` 与 `nl-` 是否成对；
- 异常轨道的重叠、平均半径和节点数变化；
- 是否出现 `SOLVE` 失败或 `METHOD=2` fallback；
- 是否出现 `ORTHY` 警告、轨道拒绝或非有限加权能量。

如果问题只在 MPI 版本出现，请明确标记为 MPI 专属问题，并提供对应的串行结果。

## 4. MPI 一致性对照

同一输入至少提供以下运行组合中的可用部分：

```text
serial rmcdhf90
rmcdhf90_mpi：1 rank
rmcdhf90_mpi：2 ranks
rmcdhf90_mpi：4 ranks
```

每次运行都应保存完整输出目录。对照内容不应只限于最终 `rmcdhf.sum`，还应包括每轮
`orbopt_trace.csv`、`rwfn`/`rmix` 快照、退出码和启动命令。若结果不一致，应优先
比较 `MATRIX/NEWCO` 能量、本征向量、`SCNSTY`、径向范数和候选轨道摘要。

## 5. 后续 P2 实施顺序

收到最小复现数据后，按以下顺序推进，一次只改变一个因素：

1. 先在当前 commit 上重现原始问题并冻结 golden baseline；
2. 若异常与 `ORTHY` 顺序相关，只增加可选稳定排序和对照日志；
3. 若异常与节点错误或 `SOLVE` fallback 相关，只增加候选验收和失败原因记录；
4. 若只有 MPI 多进程错误，单独修复 gather/归约和去重逻辑；
5. 只有在上述证据明确支持时，才实现真正的伙伴成组更新或拒绝后阻尼恢复。

所有新行为必须默认关闭，不改变历史 stdin 顺序；每项修改都必须重新运行 B0/B1/B3
以及至少一个原始问题案例，并比较能级、轨道指标、退出码和 MPI 一致性。

## 6. 交付与验收材料

后续每个 P2 提交至少附带：

- 修改文件和 commit；
- 完整运行命令与环境快照；
- 输入摘要和文件校验值；
- `rmcdhf.exitcode`、轨道 trace、能级摘要；
- 与 golden baseline 的能量和精细结构差异；
- 串行/MPI 1/2/4 对照结果；
- 对异常重叠、节点、半径和 fallback 的解释；
- 未通过项目及停止原因。

在这些材料齐全前，不将“谱序恢复”单独表述为轨道波函数或生产算法已经修复。
