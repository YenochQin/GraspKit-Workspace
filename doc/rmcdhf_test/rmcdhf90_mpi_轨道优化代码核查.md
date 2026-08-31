# `rmcdhf90_mpi` 轨道优化代码核查

本文档记录对 `rmcdhf_test` 中 `rmcdhf90_mpi` 改造代码的一次独立核查结果，
对照基准是 [轨道优化调试方案](./rmcdhf90_mpi_轨道优化调试方案.md) 与
[轨道优化已完成实施](./rmcdhf90_mpi_轨道优化已完成实施.md)。

核查日期 2026-08-31。核查对象为工作树分支 `main`，本地 HEAD `5c8a848`
(`Merge pull request #1 from YenochQin/0.1.0-test1`)。上游基线取
`grasp_2990_NNNP/src/appl/rmcdhf90_mpi`。

本文档只记录核查发现，不代表已实施修复；发现项与
[后续实施需求](./rmcdhf90_mpi_后续实施需求.md) 的 P2 顺序配合使用。

## 1. 核查范围与方法

改动范围通过目录级差异确定：

```sh
diff -rq grasp_2990_NNNP/src/appl/rmcdhf90_mpi \
         rmcdhf_test/src/appl/rmcdhf90_mpi
```

改动仅限 `rmcdhf90_mpi` 一个目录：

- 新增 `orbopt_control.f90`（212 行）、`orbopt_trace.f90`（490 行）、
  `orbopt_metrics.f90`（99 行）；
- 修改 `getoldmpi.f90`、`improvmpi.f90`、`scfmpi.f90`、`newcompi.f90`、
  `orthy.f90`、`setlagmpi.f90`、`rscfmpivu.f90`；
- 修改 `Makefile` 与 `CMakeLists.txt` 的目标文件列表。

`rmcdhf90`、`rmcdhf90_mem`、`rmcdhf90_mem_mpi` 与 `src/lib/` 全部未改动，
因此本次改造不影响串行版与 `_mem` 变体的数值路径。

`main` 与 `origin/0.1.0-test1` 的 `src/` 逐字节相同（`git diff --stat HEAD
origin/0.1.0-test1 -- src/` 为空），所以下述源码结论对两个分支同时有效。

除逐行阅读外，还对三个新模块做了实际编译验证：

```sh
mpif90 -c -I<lib/mod> -fallow-argument-mismatch -Wall orbopt_*.f90
```

`orbopt_metrics.f90` 与 `orbopt_trace.f90` 在 `-Wall` 下无告警通过；
`orbopt_control.f90` 必须带 `-fallow-argument-mismatch` 才能通过（见缺陷 J）。

## 2. 实现正确的部分

以下各项经核对确认与方案一致，可作为后续修改的稳定基础。

1. **模块依赖与构建顺序正确。** `Makefile` 的 `OBJS` 和
   `CMakeLists.txt` 都把 control → trace → metrics 置于所有消费者之前，
   无循环依赖。`ORBOPT_TRACE_C` 依赖 `ORBOPT_CONTROL_C`，顺序满足。
2. **相对论伙伴配对判据正确**（`orbopt_trace.f90:429-488`）。
   由 `NAK=κ` 推 $l$、构造伙伴 $\kappa$、对 $\kappa=-1$ 即 $s_{1/2}$ 正确
   `CYCLE`，并用 `I>J` 保证每对只报告一次。两级 warn/abort 行为符合方案 §6.4。
3. **收敛判据拆分与方案 §6.7 完全对应**（`scfmpi.f90:270-289`）。
   `CONVG_LEGACY = CONVG_ORBITAL .OR. CONVG_ENERGY`、
   `CONVG_STRICT = CONVG_ORBITAL .AND. CONVG_ENERGY .AND. ENERGY_VALID`、
   连续两轮退出，且 `ENERGY_VALID = EOL .AND. NIT>1 .AND. WTAEV/=0` 明确
   排除首轮 `WTAEV0=0` 的伪触发——这正是方案点名要求的一条。
4. **积分约定与 GRASP 一致。** `orbopt_metrics.f90:31-55` 复用
   `TA/RP/QUAD`，重叠积分取 `MTP = MIN(MTP0, MF(J))`，与
   `lib/lib9290/rint.f90:53-64` 中 `RINT(...,0)` 的写法逐项相同；
   `COUNT` 的调用形状与 `DAMPOR`、`SOLVE` 一致。指标在归一化之后计算，
   满足方案 §6.5「先用未改变轨道验证 $S_J\approx1$」的可验证性要求。
5. **`DAMPOR` 前后两次指标调用的参数交换是正确的。** `DAMPOR` 把上一轮
   轨道换入 `P/Q` 并令 `MTP0=MTPO`，新接受轨道留在 `PF/QF`；
   `improvmpi.f90:361-364` 将实参顺序整体对调，语义正好复原，
   `accepted_metrics` 行的 candidate 列确实对应新轨道。写法脆弱但结论正确。
6. **护栏插入位置符合方案 §6.6** 步骤 1–4：在候选归一化之后、
   `DAMPCK/DAMPOR/ORTHY` 之前判定，`PF/QF` 因未进入 `DAMPOR` 而自然保持旧值。
7. **确实防住了「反复拒绝被误判为收敛」。** 拒绝时把 `SCNSTY(J)` 还原为
   旧的大值，`MAXARR` 仍会返回该轨道，`CONVG_ORBITAL` 保持 `.FALSE.`。
8. **trace 的 MPI 写入策略符合方案 §6.3。** 主 CSV 仅由 rank 0 写入，
   逐 rank 文件只在摘要不一致时产生，不存在多进程竞争同一文件。
9. **`tatb_C` 的 `TA/MTP` 被指标模块改写不影响数值路径。** 调用点之后的
   `DEL1/DEL2` 使用局部量，`DAMPCK` 不用 `TA`，`DAMPOR` 经 `RINT` 自行重设
   `MTP`，`ORTHY` 自行赋值 `MTP0`。已逐一确认。

## 3. 确认的缺陷

按建议处理顺序排列，编号在第 5 节复用。

### A. 诊断打印点过早，记录的不是 SCF 实际使用的状态

`APPLY_FIXED_ORBITAL_DAMPING`、`CHECK_RELATIVISTIC_PAIRS`、
`TRACE_ORBITAL_SELECTION` 都位于 `getoldmpi.f90:190-192`，但
`GETSCDmpi` 在其后仍会改写状态：

- `getscdmpi.f90:236-517` 的「Modify other defaults?」分支可重写
  `METHOD`、`NOINVT`、`ODAMP`、`THRESH`、`NSIC`、`NSOLV`、`ORTHST`，
  并在 521-530 广播；
- `rscfmpivu.f90:159-161` 之后再强制 `ORTHST=.FALSE.` 或 `METHOD=3`。

因此 CSV 的 `selection` 行可能与进入 `SCFmpi` 时的有效值不一致，
而「varied list 是否真的只含 `nl`」正是提交 2 要回答的问题
（方案 §6.3 验收）。应将上述调用移到 `GETSCDmpi` 返回之后，
或放在 `SCFmpi` 入口处。

### B. `GRASP_ORBITAL_DAMPING` 会挪动 stdin，也可能被静默覆盖

由 A 导致：`ODAMP` 在 `getscdmpi.f90:396` 的判据
`ODAMP(I)/=0.D0 .AND. .NOT.LFIX(I)` 之前就变为非零，于是
「Subshell accelerating parameters have been set. Revise these?」
这一提问在 `NDEF/=0` 且用户回答「Modify other defaults = y」的路径下
**会新出现一次**，历史输入脚本将错位；若回答 yes，环境变量设置的阻尼
被覆盖且无任何提示。

这与已完成实施文档「未向历史 stdin 增加新问题」的表述不符。默认
`NDEF=0` 路径提前 `RETURN`，不受影响，因此影响面窄，但缺陷真实存在。

### C. 固定阻尼作用范围超出方案

`getoldmpi.f90:169` 刚把变分光谱轨道的 `ODAMP` 显式置 0，
第 190 行的调用又对所有 `.NOT.LFIX` 轨道覆盖。方案 P1.2 的目标是
「新关联层」。结果 B4 的观测（最小已接受重叠由 `0.558525` 提升到
`0.894903`）混入了「光谱轨道也被加了固定阻尼」这一额外变量。
建议限制到 `LCORRE` 为真的轨道。

### D. `STRICT_METHOD3` 把两件事混在一起

`rscfmpivu.f90:160` 的 `WHERE (.NOT.LFIX(:NW)) METHOD(:NW) = 3` 会覆盖
`getoldmpi.f90:167` 给变分光谱轨道设置的 `METHOD=1`。而 `METHOD<=2` 才是
调用 `NEWE`、执行节点数与 `NNODEP` 校验的分支（`solve.f90:324-336`），
`METHOD=3` 直接跳过该校验。

又因 `METHOD(:NW)=3` 本来就是全局默认（`getscdmpi.f90:192`），
该开关的实际效果只有两条：撤销光谱轨道的 `METHOD=1`，以及 `FAIL` 时
`ERROR STOP`。

因此「B6 与 B3 完全一致」只在**没有变分光谱轨道**时成立；一旦存在，
B6 就不是干净的「隔离求解器分支」实验，而是顺带关闭了节点约束。
建议拆分为 `STRICT_NO_FALLBACK`（只堵 `FAIL → METHOD=2`）与
`FORCE_METHOD3` 两个独立开关。

### E. 护栏拒绝路径并非无副作用

`improvmpi.f90:317-342`：

- `MF(J)` 与 `PZ(J)` 从来不由 `SOLVE` 写入（只有 `DAMPOR`、`ORTHY` 写），
  所以这两个还原是空操作。说明还原集合没有按 `SOLVE` 的真实副作用推导。
- `NSIC` 在第 306 行（护栏判定之前）可能已被永久递减，被拒绝的候选
  照样缩小了 `NSIC`。
- `METHOD(J)` 可能已被第 302 行的 `1 → 2` 重试改掉，未还原。
- `ODAMP(J)` 被永久改写为固定负值（第 327 行），而
  `CLEAR_ORBITAL_REJECTIONS(J)` 只清计数、**不恢复 `ODAMP`**。
  由于 `DAMPCK` 的 `ADAPTV = ODAMP(J) >= 0`（`dampck.f90:38`），
  一次拒绝就把该轨道在整个后续运行中转为非自适应固定阻尼。
  该行为未在已完成实施文档中记录。
- `P/Q/P0/MTP0` 保留被拒候选。对数值无害（下一次 `SOLVE` 覆盖），
  但 `LDBPR(23/24/25)` 下 `PRWF` 会把被拒轨道当作已接受打印。

### F. 「逐次提高阻尼强度」基本不生效

拒绝后直接 `RETURN`，重试只能依赖 `SCFmpi` 的 `DO KOUNT = 1, NSIC` 循环，
而 `getoldmpi.f90:177-182` 在存在变分关联轨道时强制 `NSIC=0`——
正是本次研究的 LBL 场景。因此提高后的 `ODAMP` 只在下一个宏迭代生效，
`MAX_REJECTS_PER_ORBITAL=3` 计的是连续宏迭代次数，最多两次升挡
（`-0.5 / -0.7 / -0.9`）便终止。行为可用，但文档表述容易被读作
迭代内重试。

### G. 非有限 `WTAEV` 的终止是无开关的默认行为改变

`scfmpi.f90:255-259` 不受任何开关保护。基线遇到 NaN 时
`ABS((WTAEV-WTAEV0)/WTAEV) < 0.001*ACCY` 为假，会一直跑到 `NSCF`
并写出无意义的 `.sum`；现在直接 `ERROR STOP`、退出码 1。

改动本身合理，但违反方案 §6「未设置新开关时必须保持当前数值结果不变」
的字面要求；而且 B5 的「预期失败」实际由这条检查产生，不是延迟正交化
本身的性质。建议加开关，或在文档中明确登记为有意的默认变更。

### H. AL（非 EOL）路径的能量判据语义改变

基线无条件求 `ABS((WTAEV - WTAEV0)/WTAEV)`，而 `WTAEV` 只在
`IF (EOL) CALL NEWCOmpi(WTAEV)` 中赋值——AL 运行读的是未定义变量。
新代码用 `EOL .AND. WTAEV /= 0.D0` 保护并置 `CONVG_ENERGY=.FALSE.`。

这是修正，但同属默认路径改变，且两份文档都没有 AL 的 golden baseline。

### I. `ASSOCIATED(EVEC)` 作用于关联状态未定义的指针

`eigv_C` 将 `EVEC` 声明为普通 pointer，没有 `=> NULL()` 初值；
它只在 `IF (EOL)` 时于 `SCFmpi` 被 `ALLOC`。
`orbopt_trace.f90:392` 对它调用 `ASSOCIATED`，在 AL 运行加
`GRASP_TRACE_ORBOPT=1` 时属未定义行为。

修法：给 `eigv_C` 的 `EVEC` 加 `=> NULL()`，或显式传入 `EOL` 标志。

### J. MPI 数据类型常量绕过本仓库的 `-i8` 约定

`improvmpi.f90:104` 与 `scfmpi.f90:89` 特意使用
`I_MPI = MPI_INTEGER` / `L_MPI = MPI_LOGICAL` 间接层（注释中标注
`MPI_INTEGER8` / `MPI_LOGICAL8`）以支持 `-i8` 与
`-fdefault-integer-8`。`orbopt_control.f90:81-104` 直接使用裸
`MPI_LOGICAL` 与 `MPI_INTEGER`，此类构建下会静默传输错误长度的数据。

相关但独立的一点：`Makefile` 路径的
`FC_FLAGS ?= -O2 -fno-automatic`（`rmcdhf_test/Makefile:15`）
**不含** `-fallow-argument-mismatch`，只有 `CMakeLists.txt:110` 有。
同类型混用在既有文件中也存在（`getscdmpi.f90:157` 与 `165` 分别以
`MPI_DOUBLE_PRECISION` 与 `MPI_INTEGER` 调用同一 `MPI_Bcast`），
因此这不是本次引入的回归；但它意味着 `Makefile` 的 `OBJS` 改动
从未被任何已记录的构建验证过——已完成实施文档 §5 只做了
CMake Debug 构建。

### K. 任何异常终止路径都不会关闭 trace 文件

`CLOSE_ORBOPT_TRACE` 只在 `scfmpi.f90:318` 调用。四条新的致命路径
（伙伴检查、strict METHOD3、拒绝上限、非有限 `WTAEV`）全部使用
`ERROR STOP`，属错误终止，标准不要求 flush 或关闭已连接单元。
也就是说**最需要查看 trace 的运行最可能丢失缓冲行**。

建议每行后 `FLUSH`，或让致命退出统一走一个先关闭 trace 再调用
`MPI_ABORT` 的辅助过程。

### L. `ERROR STOP` 而非 `MPI_ABORT` / `stopmpi`

四条路径在所有 rank 上判据一致（输入均已广播或复制），实践中
`mpirun` 会整体拆掉作业；但仓库本身有 `MPI_ABORT`
（`lib/mpi90/cpath.f90:121`）与 `stopmpi` 约定，裸 `ERROR STOP`
没有 `MPI_Finalize`，在部分启动器下可能留下孤儿进程。
基线 `improvmpi.f90:258` 也用 `ERROR STOP`，属既有风格混杂。

### M. CSV schema 用同一列承载不同类型

`TRACE_FIELD_COUNT=58` 且表头固定，但 `control` 行把值写进了含义不同的列：

| 写入位置 | 表头列名 | 实际内容 |
| --- | --- | --- |
| `FIELDS(15)` | `odamp` | `FIXED_ORBITAL_DAMPING` |
| `FIELDS(38)` | `convg_final` | `ENABLE_ORBITAL_GUARD` |
| `FIELDS(39)` | `overlap` | `MIN_ORBITAL_OVERLAP` |
| `FIELDS(43)` | `radius_old` | `MAX_RADIUS_RATIO` |
| `FIELDS(46)` | `nodes_candidate` | `REJECT_NODE_CHANGE`（逻辑值进整数列） |
| `FIELDS(47)` | `mf_old` | `MAX_REJECTS_PER_ORBITAL` |

`mpi_summary_mismatch` 行又把 `overlap` 列当作摘要数值使用。
第 55–58 列本来就是给开关准备的，却只用于其中四个开关。
按列定型读取（例如 pandas）会得到 object dtype 或转换失败。
方案要求「稳定的 CSV 或 JSONL 字段」，建议增加专用 `control_*` 列，
或单独写 `orbopt_controls.csv`。

### N. 方案 P0.2 的输入校验只实现了四分之一

方案要求对「重复索引 / 固定轨道 / 未知轨道 / 同 $\kappa$ 伙伴缺失」
都给出警告，当前只实现了最后一项。

另外方案 §3.2 的 $\mathcal V_k \cap \mathcal S_k$ 只是逐轨道记录、
从未校验；伙伴之间 `LCORRE` 是否一致也没有检查——而
`getoldmpi.f90:167-170` 恰恰让 `LCORRE` 不同的伙伴拿到不同的
`METHOD`、`NOINVT` 与 `ODAMP`，这是与 `LFIX` 不平衡同等合理的
失衡来源。

## 4. 文档与仓库状态不一致

已完成实施文档称核查时 HEAD 为 `0.1.0-test1` 分支的 `8028fcd`。
该 commit 在本 clone 中不存在（`git cat-file -t 8028fcd` 失败）。
当前检出为 `main`（`5c8a848`），比 `origin/0.1.0-test1` 落后三个提交：
`71954ca`、`883ebcc`、`15a0b47`。

由此产生下列差异：

| 文档列为已交付 | 工作树 | `origin/0.1.0-test1` |
| --- | --- | --- |
| `check_damping_repeatability.py` | 缺 | 有 |
| `run_damping_repeats.sh` | 缺 | 有 |
| `summarize_damping.py` | 缺 | 有 |
| `run_repro_sbatch.sh`（§5.5 作业 491） | 缺 | **缺** |
| `test/data/`（runner 输入来源） | 缺 | **缺**（被 `.gitignore` 排除） |

需要单独说明两点：

1. §5.5 的 `run_repro_sbatch.sh` 在两个分支中都不存在，该条记录
   在仓库中没有任何支撑。
2. `test/data/` 既被 `.gitignore` 排除，也不在本地文件系统上存在，
   而 `run_data_case.sh` 按文档需要从它复制 `isodata`、CSF 与波函数。
   因此已完成实施文档 §5.2–5.6 的全部数值（Cl I 的
   `932.188187 cm⁻¹`、九算例阻尼矩阵、三轮重复性等）
   **无法从本 clone 复现或核对**。这一点与
   [后续实施需求](./rmcdhf90_mpi_后续实施需求.md) §1 的最小可复现
   数据集要求直接相关。

`src/` 在两分支间相同，所以第 2、3 节的源码结论不受此影响；
缺失的只是工具与结果层。

## 5. 建议的最小修复顺序

一次只改一个因素，每步之后重跑 B0/B1/B3 与默认开关的 golden baseline。

1. **K + I**：trace 的 flush 与关闭；`eigv_C` 的 `EVEC` 加 `=> NULL()`。
   纯诊断修复，不触及数值路径。
2. **A**：把 `TRACE_ORBITAL_SELECTION` 与
   `APPLY_FIXED_ORBITAL_DAMPING` 移到 `GETSCDmpi` 之后。该步同时消除 B。
3. **C + D**：把固定阻尼与 `METHOD=3` 限制到 `LCORRE` 轨道，
   并把 `STRICT_METHOD3` 拆为「禁 fallback」与「强制 METHOD=3」。
   B4 与 B6 的结论需要在此之后重跑才成立。
4. **E**：拒绝时恢复 `ODAMP(J)`、`METHOD(J)`、`NSIC`；
   或明确把「一次拒绝即永久固定阻尼」写入文档作为设计决定。
5. **G + H**：给这两处默认行为改变加开关，或在文档中显式登记为
   有意的默认变更，并补 AL 路径的 golden baseline。
6. **J + M + N**：`-i8` 数据类型间接层、CSV 列分离、
   补齐输入校验（重复 / 固定 / 未知索引，以及伙伴 `LCORRE` 一致性）。
7. **L**：致命退出统一改走 `MPI_ABORT` 或 `stopmpi`（可与第 1 步合并）。

## 6. 总体判断

这批改动的方向与关键实现细节是正确的。最容易出错的四处——
积分约定、`DAMPOR` 前后的 `P/Q` 与 `PF/QF` 语义、收敛判据拆分、
相对论伙伴配对判据——都正确，且所有会改变数值路径的开关默认关闭。

主要问题不在算法本身，而在两类：

- **诊断插入时机**（A、B）与**开关语义混杂**（C、D）。这两类会直接
  削弱 B4 与 B6 实验结论的可归因性，应在扩大 AS 之前处理。
- **护栏的残留副作用**（E）与**几处无开关的默认行为改变**（G、H）。

方案第 7 节的生产验收确实还不能宣称通过，这一点已完成实施文档
第 6、7 节的自我定位是准确的。当前交付物应继续按
「RMCDHF MPI 轨道优化诊断与实验性护栏版」表述。
