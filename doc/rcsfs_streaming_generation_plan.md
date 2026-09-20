# rCSFs 大规模流式生成、磁盘去重与 CSF 恢复实施方案

日期：2026-09-21

状态：工程方案；Phase 1–3 已于 2026-09-21 实施并通过 Rust 全套测试，Phase 0 和 Phase
4–5 仍待实施。流式生成、磁盘去重目前仍是 crate 内路径，尚未替换 CLI 的多输出编排。本文取代：

- `doc/csf_descriptor_v2_ml_design.md` §9 中“生成和去重期间不得写盘”的旧约束；
- `doc/csf_generation_streaming_design.md` 的“直接同时写出 CSF 与描述符、无需最终 CSF
  去重”方案。

V2 描述符的数据合同、seniority/耦合含义及可逆性要求保持不变。旧文档保留为诊断和设计
演进记录；发生冲突时以本文为准。

核对基线：

| 仓库 | 提交 | 用途 |
| --- | --- | --- |
| `rCSFs` | `983772d6516f062960ed904e96988c3a991c6052` | 当前生成器、V2 编解码、CLI 和恢复路径 |
| 父工作区 | `54e0941` | 当前子模块指针与协调文档 |

## 1. 目标

为数千万至上亿条 CSF 提供内存有界的生成路径：生成器直接把完整整数记录编码为
CSFDescriptorV2，分批写入临时磁盘；在磁盘上完成精确去重、稳定排序和块计数；最终发布
描述符文件，并从描述符流式恢复规范 CSF 文本。

目标输入是：

```toml
[generate]
order = "*"
core = 1
references = [
    "2s(2,i)2p(6,i)3s(2,1)3p(6,i)3d(8,*)4s(2,*)",
    "2s(2,i)2p(6,i)3s(2,i)3p(6,5)3d(8,*)4s(2,*)",
]
active_orbitals = "9s,9p,9d,9f,7g"
j_min = 8
j_max = 8
excitations = 4
continue_lists = false

[output]
generate_descriptors = true
csf = "calculation.c"
parquet = "calculation.parquet"
descriptor_parquet = "calculation_descriptors.parquet"
normalize = false
```

已运行只到占据组态枚举结束的安全探针，结果为：

```text
unique_occupations=7031941
relativistic_entries=86032095
core_subshells=1
active_subshells=56
```

该探针没有进入完整 CSF 生成。GRASP 的参考结果为 77,362,152 条 CSF；该数量仍须在
服务器验收阶段由新实现复核。

## 2. 当前问题

### 2.1 当前数据流

当前 `generate_csfs_from_transcript` 执行：

```text
解析 transcript
  -> enumerate_occupations：保存全部占据组态
  -> 为每个组态克隆 GenerationRequest
  -> generate_csfs_parallel：收集 Vec<CompleteCsfFile>
  -> write_generated_csfs：此时才创建 CSF 文件
  -> Python CLI 再把 CSF 转成三行文本 Parquet
  -> 再从三行文本生成描述符 Parquet
  -> 最后发布所有目标文件
```

因此，描述符导出发生在峰值内存阶段之后。`generate_descriptors=true` 不会降低生成阶段的
内存；失败时最终目录也不会出现部分结果，因为 CLI 使用私有临时目录并在最后发布。

### 2.2 长期存活的数据

当前实现同时保留：

- 7,031,941 个 `EnumeratedConfiguration`。
- 从这些组态克隆出的 `GenerationRequest` 列表。
- 每个组态对应的 `CompleteCsfFile`。
- 每条 CSF 重复保存的 `OccupiedSubshell` 与 `IntermediateCoupling`。
- 每个组态结果重复的五行头、Peel 字符串、`Vec` 元数据和容量余量。
- 组态内部并行分支合并期间的新旧记录缓冲。

当前平台上基础结构大小为：

| 结构 | `size_of`，不含堆数据 |
| --- | ---: |
| `CsfRecord` | 24 B |
| `OccupiedSubshell` | 8 B |
| `IntermediateCoupling` | 4 B |
| `SymmetryBlock` | 24 B |

这些数字不能直接代表实际 RSS；字符串、Vec 容量、分配器元数据、并行临时结果和 Python
阶段都必须计入。

### 2.3 当前接口中的额外问题

- `rcsfs/cli.py::_generate_outputs` 仍显式设置 `descriptor_version = 1`，与公开接口已经默认
  V2 的行为不一致。新的大规模路径必须直接使用 V2，不能从 V1 恢复 CSF。
- `py_restore_csfs_from_descriptors` 当前把全部 Parquet 行收集为 `Vec<Vec<i32>>` 后才写出，
  会在恢复数千万行时再次形成全量内存峰值。
- `write_generated_csfs` 负责最终 J/宇称块排序；直接按工作线程完成顺序落盘会改变输出顺序。
- 当前占据组态阶段已经去除相同占据配置，但尚未证明多参考生成的完整 CSF 不会重复。

## 3. 范围与非目标

### 3.1 本方案包含

- 从生成器直接得到完整、可逆的整数记录。
- 逐条编码为 V2，不经过 CSF 文本反解析。
- 有界批次、磁盘暂存和受控并行。
- 基于完整 V2 身份的精确去重。
- 保留首次出现顺序和 GRASP 的 J/宇称块顺序。
- 流式恢复 CSF 文本。
- 可选生成现有三行文本 Parquet。
- 原子发布、失败清理、磁盘预算检查和阶段统计。
- 小规模结果与旧路径做逐字节/逐行差分。

### 3.2 本方案不包含

- 改变 CSF 的角动量耦合、态表或激发枚举物理规则。
- 改变 V2 四字段和两个全局字段的数据合同。
- V2 浮点归一化。
- 将行号、哈希、分桶号或生成序号作为 ML 特征。
- 第一阶段就实现跨进程断点续跑。断点续跑在正确性和清理语义稳定后实施。
- 第一阶段就把 7,031,941 个占据组态改为完全流式枚举。该优化独立列为后续阶段。

## 4. 核心设计决策

### 4.1 V2 是磁盘中的权威中间表示

固定 Peel 表下，一条 V2 记录包含：

```text
[N, 2J, seniority, 2K_printed] * M + [total_two_j, parity]
```

它足以恢复当前支持范围内的规范 CSF。临时数据和最终描述符均使用同一字段语义；临时文件
可以增加内部列，但不能改变 V2 字段：

```text
range_ordinal: u32     # 生成范围顺序，只用于稳定排序
local_ordinal: u64     # 范围内首次出现顺序
descriptor fields      # V2 权威记录
```

内部列在最终描述符中删除，也不进入 ANN/CNN。

### 4.2 精确重复的定义

生成结果中的两条记录仅在以下内容全部相同时才重复：

- 同一个有序 Peel 表。
- 全部 V2 子壳层字段。
- `total_two_j`。
- `parity`。

由于 V2 保留 seniority、显式耦合和缺失状态，不能再使用 V1 三字段作为去重键。哈希仅用于
选择磁盘桶；同桶内必须比较完整 V2 行，不能仅凭哈希删除记录。

导入既有 CSF 文件并做无损往返时仍保留重复行。磁盘去重只属于“从配置生成新 CSF”的
接口，不能静默应用于 `convert_csfs` 或普通 `restore-csfs`。

### 4.3 稳定顺序

最终顺序保持当前 `write_generated_csfs` 的合同：

1. 按 `(total_two_j, parity)` 的 `BTreeMap` 顺序排列块。
2. 同一块内按占据组态的原始枚举顺序排列。
3. 同一组态内按当前态表遍历和耦合遍历顺序排列。
4. 重复记录保留第一次出现的位置。

工作线程完成顺序不得影响输出。每条临时记录使用 `(range_ordinal, local_ordinal)` 表达确定性
顺序，不使用全局原子计数器分配顺序，因为原子计数会把线程调度写入输出合同。

### 4.4 内存预算

内存必须由配置和内部批量上限共同约束，不能依赖“通常不会积压”。实现至少报告：

- 组态枚举有效大小和容量。
- 当前生成范围的记录数。
- 待写批次数和字节数。
- Parquet/Arrow builder 容量。
- 去重桶峰值。
- 进程峰值 RSS（平台支持时）。

若 M=56，每条稠密 V2 行为 `4*56+2=226` 个 Int32，即 904 字节。77,362,152 行的
未压缩字段载荷约为 69.94 GB（65.13 GiB）。实现不能把整份稠密矩阵放入内存。

## 5. 深模块与 seam

### 5.1 外部深模块

调用者只使用一个外部接口：

```rust
pub fn generate_outputs(
    transcript: &str,
    outputs: &GenerationOutputs,
    options: &GenerationOptions,
) -> Result<GenerationStats>;
```

接口负责隐藏：任务分段、并行调度、V2 编码、临时目录、分桶、去重、最终合并、CSF 恢复、
可选 Parquet 和原子发布。Python CLI 不再按顺序拼接五个独立转换调用。

`GenerationOptions` 只公开用户需要决定的内容：

```rust
pub struct GenerationOptions {
    pub threads: Option<usize>,
    pub scratch_dir: Option<PathBuf>,
    pub memory_budget_bytes: Option<u64>,
    pub keep_scratch_on_error: bool,
}
```

row group 大小、生成范围大小、桶数和通道容量由深模块根据预算推导，先不暴露为公共参数。
统计信息可报告最终选择值，便于服务器诊断。

### 5.2 生成器内部 seam

`Generator::emit` 不再直接依赖 `CompleteCsfFile`。定义借用记录视图：

```rust
pub(crate) struct GeneratedRecordRef<'a> {
    pub occupied: &'a [OccupiedSubshell],
    pub couplings: &'a [IntermediateCoupling],
    pub total_two_j: u16,
    pub parity: Parity,
}

pub(crate) trait GeneratedRecordSink {
    fn push(&mut self, record: GeneratedRecordRef<'_>) -> Result<()>;
}
```

提供两个真实 adapter：

- `CompleteCsfSink`：构建当前 `CompleteCsfFile`，保持既有 Rust 接口和小规模测试。
- `DescriptorBatchSink`：调用共享 `write_feature_row`/V2 编码，把行加入固定上限批次并写入
  当前生成范围的暂存文件。

测试和生产通过同一 seam。物理递归只知道 `GeneratedRecordSink`，不知道 Parquet、路径、哈希
或 Python。

### 5.3 任务分段

不能为 7,031,941 个组态各创建一个文件，也不能让 writer 为等待较早任务而无限缓存较晚
任务。将连续组态切成确定性的 `GenerationRange`：

```rust
pub(crate) struct GenerationRange {
    pub ordinal: u32,
    pub start_configuration: usize,
    pub end_configuration: usize,
}
```

每个范围由一个工作任务按组态顺序处理，并为遇到的每个 `(2J, parity)` 写一组滚动 segment。
范围可以并行完成；最终按 `range.ordinal` 合并，因此不需要在内存中重排工作线程结果。

范围大小根据组态数量和内存预算推导。初始目标为总范围数约 512–4096 个，而不是七百多万
个。segment 达到大小上限时滚动，避免单文件过大。

## 6. 目标数据流

```text
配置/TOML
  -> 解析 ExcitationRequest
  -> 枚举并合并占据组态
  -> 预计算最终 Peel 表
  -> 划分有序 GenerationRange
  -> N 个 worker：
       生成完整整数记录
       -> validate_record
       -> encode_v2/write_feature_row
       -> 按 (2J, parity) 写 range segment
  -> 按对称性块进行哈希分桶
  -> 每桶完整键去重，标记首次出现 ordinal
  -> 按块顺序、range 顺序、local 顺序输出 survivor
  -> 得到最终 block_lengths 和 record_count
  -> 构建 header TOML 并计算 SHA-256
  -> 写最终 descriptor Parquet KV metadata
  -> 流式恢复 calculation.c
  -> 可选生成 calculation.parquet
  -> 原子发布完整输出集合
```

## 7. Peel 表的确定

当前 `write_generated_csfs` 从非空生成结果收集实际使用的相对子壳层。流式 writer 在第一条
记录之前就需要固定 V2 列，因此不能等所有 CSF 生成后再确定 Peel。

在占据组态枚举完成后扫描 `EnumeratedConfiguration`，收集所有非零相对子壳层，按当前
`(n, l, kappa)` 规则排序。这与从所有非空 `CompleteCsfFile.subshells` 求并集等价，但不需要
生成 CSF。增加差分测试，逐配置比较旧路径和预计算路径的 Peel 头。

若一个占据组态不可能产生目标 J 的 CSF，它可能带来最终未使用轨道。为保持旧文件头字节
一致，首版应先验证这种情况是否影响目标配置。若影响，采用两阶段位图：worker 在生成时记录
实际使用轨道的 56 位掩码；全部范围结束后求并集，再把临时局部轨道映射展开为最终 Peel。
不得未经测试就把完整 active orbital 列表写入 Peel。

## 8. 临时文件格式

### 8.1 目录布局

```text
<scratch>/rcsfs-<config-hash>-<random>/
  run.toml
  ranges/
    range-000000/
      j0008-even-000.arrow
      j0008-odd-000.arrow
    ...
  buckets/
    j0008-even-bucket-000.arrow
    ...
  survivors/
    j0008-even.bits
    j0008-odd.bits
  final/
    descriptors.parquet
    header.toml
    calculation.c
    calculation.parquet
```

临时 segment 优先使用 Arrow IPC stream/file：无需为每个小段建立复杂 Parquet metadata，写入
和顺序扫描开销较低。最终交换文件继续使用 Parquet。若基准证明 Parquet segment 更合适，可以
替换内部 adapter；外部接口不改变。

### 8.2 `run.toml`

保存：

- 格式版本和实现提交号。
- 输入配置规范化文本及 SHA-256。
- Peel 表、DescriptorLayout 和缺失值。
- 线程数、内存预算、范围大小、桶数。
- 每个阶段状态和每个 segment 的记录数/字节数。
- 完成的范围、块和桶。
- 错误摘要。

首版仅用于诊断和清理，不承诺恢复执行。实现 `--resume` 时必须校验配置哈希、格式版本和构建
兼容性，不能盲目复用旧临时文件。

## 9. 磁盘去重算法

### 9.1 分桶

对每个对称性块独立处理。计算完整 V2 行的稳定 128 位哈希，使用哈希低位选择 B 个桶。
写入桶的记录包括：

```text
global_block_ordinal
hash128
完整 V2 行
```

`global_block_ordinal` 在按 range/segment 顺序扫描时顺次分配，因此与线程完成顺序无关。

桶数根据以下公式向上取 2 的幂：

```text
estimated_bucket_bytes <= memory_budget_for_dedup / safety_factor
```

`safety_factor` 首版至少取 2，覆盖 HashMap、Vec 容量和 Arrow 解码开销。若实际桶超过预算，
递归增加一个哈希位再次分桶，不能继续加载直至 OOM。

### 9.2 完整键比较

每个桶按原始扫描顺序读取。HashMap 以 128 位哈希定位候选列表，再比较完整 V2 行：

- 首次出现：将其 `global_block_ordinal` 标为 survivor。
- 完整行重复：忽略后续记录。
- 哈希相同但完整行不同：保留两条，并记录碰撞统计。

survivor 使用每块一个磁盘 bitset；77,362,152 位约为 9.2 MiB，远小于保存全部 u64 索引。
bitset 可分段更新或内存映射。Windows 和 Unix 必须有一致的文件增长、flush 和错误语义。

### 9.3 最终稳定输出

按 `(2J, parity)` 块顺序重新扫描 range segment，并递增当前块 ordinal。bitset 为 1 时写入最终
描述符；否则计入 `duplicate_count`。这一遍天然保留首次出现顺序，无需再做全局排序。

统计必须满足：

```text
generated_count = unique_count + duplicate_count
sum(block_lengths) = unique_count
descriptor_rows = unique_count
restored_csf_records = unique_count
```

## 10. 最终描述符和元数据

最终 Parquet 只包含正式 V2 列，不包含临时排序列或哈希。KV metadata 至少包括现有字段及：

```text
descriptor_version=2
generation_pipeline_version=1
deduplicated=true
generated_record_count=<去重前>
record_count=<去重后>
duplicate_count=<差值>
block_lengths=<JSON>
source_config_sha256=<规范化配置哈希>
source_header_sha256=<最终 header TOML 哈希>
```

最终 header TOML 保存五行头、块长度、生成统计、Peel 表和配置哈希。先完成去重和块计数，再
构建 header 并计算哈希，最后创建正式 Parquet writer，避免事后修改 Parquet footer。

若 `generate_descriptors=false`，仍生成内部 V2 暂存和去重结果，但不发布最终 descriptor
Parquet。CSF 恢复结束后删除内部 V2 文件。

## 11. 流式恢复 CSF

新增 Rust 深模块：

```rust
pub fn restore_csf_stream(
    descriptor_path: &Path,
    header_path: &Path,
    output_path: &Path,
    selection: RestoreSelection<'_>,
) -> Result<RestoreStats>;
```

实现步骤：

1. 读取并验证 Parquet schema、列名、类型、null、版本、Peel 表和 header hash。
2. 读取 header 的 `block_lengths`，验证总和等于 Parquet 行数。
3. 创建原子临时输出和 `BufWriter`。
4. 写五行头。
5. 逐个 RecordBatch 读取；每行复用一个 `Vec<i32>` scratch。
6. `decode_v2_into` 后调用 `validate_record`。
7. 在块起点写 `" *"`，直接调用单记录格式化函数写三行。
8. flush、核对记录数和块数，再原子发布。

完整恢复不构建 `Vec<Vec<i32>>`、`CompleteCsfFile` 全集或完整字符串数组。

子集恢复的接口继续支持索引，但采用以下策略：

- 少量有序索引：顺序扫描 descriptor，并在命中时写出。
- 无序或重复索引：先验证、排序读取请求，同时保存请求位置；结果仍按调用者给出的索引顺序。
- 超大索引集合：使用磁盘索引或明确拒绝超出内存预算，不能悄悄全量加载。
- 子集块按输出记录的相邻 J/宇称重新构建，不直接套用原始 `block_lengths`。

## 12. CSF 三行 Parquet

现有 `calculation.parquet` 保存 CSF 三行文本。首版允许在恢复 `calculation.c` 后调用现有
流式 `convert_csfs`，因为它不形成全量内存峰值。后续可以给单记录格式化器增加双写 adapter，
同时输出文本和 Arrow batch，减少一次磁盘读取。

无论采用哪种实现，以下内容必须一致：

- `idx` 与最终去重后 CSF 顺序一致。
- header 的 `block_lengths` 与描述符、CSF 文本一致。
- `calculation.parquet` 的三行可重新构造相同规范记录。

## 13. 并行、背压与失败传播

### 13.1 并行模型

- 工作粒度为 `GenerationRange`，不是单个组态。
- 每个 range 内按组态顺序生成，避免内部再创建全量分支结果。
- worker 数由 `--threads` 控制；TOML 的 `core` 是 GRASP core selector，不是 CPU 数量。
- 每个 worker 最多持有一个 V2 batch 和有限数量 Arrow builder。
- range segment 直接由 worker 写入独立路径，避免共享 Parquet writer 锁。

### 13.2 错误传播

任何 worker、writer、分桶、去重、恢复或发布错误都必须：

1. 设置共享取消标记。
2. 停止派发新 range。
3. 让工作线程在安全检查点退出。
4. 关闭并 flush 能关闭的 writer。
5. 保留第一条根因，并附加其他线程错误作为上下文。
6. 删除尚未发布的最终输出。
7. 根据 `keep_scratch_on_error` 删除或保留 scratch，并打印其路径。

不得用错误 sentinel 填充描述符，不得报告 `success=true`，不得因有界通道两端退出顺序错误而
死锁。

## 14. 磁盘预算与预检

M=56、N=77,362,152 时，单份未压缩 V2 字段约 65.13 GiB。最坏情况下同时存在：

- range segment：约 1 份。
- hash bucket：约 1 份，另含 ordinal/hash。
- 最终 descriptor：约 1 份。
- 恢复后的 CSF 文本和三行 Parquet。
- 文件系统、Arrow 和压缩临时开销。

不能依赖压缩率做安全预检。目标服务器建议至少准备 250–350 GiB scratch；实际阈值由一个
1% 或固定百万行样本测得后更新。运行时每完成一个阶段重新检查剩余空间；预计无法完成时应
提前失败并报告已写字节、估算剩余字节和可用空间。

scratch 应位于高速本地 NVMe，避免共享网络文件系统上产生大量 segment 和随机桶写入。最终
发布目录可以不同，但跨文件系统发布不能假装成原子 rename；需要复制到目标目录内临时文件、
fsync 后再发布。

## 15. 输出集合发布

引入 `OutputTransaction` 深模块，集中管理：

- `calculation.c`
- `calculation.parquet`
- `calculation_header.toml`
- `calculation_descriptors.parquet`
- `calculation_descriptors.toml`

所有目标路径先做词法、canonical path、symlink 和 hardlink 冲突检查。每个文件先写入目标目录
旁的唯一临时路径。全部文件关闭并验证后才发布；若发布中途失败，删除本次已创建的目标，恢复
“全部存在或全部不存在”的可观察结果。发布 manifest 最后写入，可作为输出集合完整标记。

不覆盖已有文件，除非未来新增显式 `overwrite=true`；默认行为保持当前安全合同。

## 16. CLI 与配置

首版新增：

```text
rcsfs csfsgenerate --config gencsfs.toml \
  --generation-storage disk \
  --scratch-dir /local/nvme/rcsfs \
  --memory-budget-gib 64 \
  --threads 32
```

对应可选 TOML：

```toml
[generate]
storage = "disk"
scratch_dir = "/local/nvme/rcsfs"
memory_budget_gib = 64
```

命令行覆盖 TOML。首版保留 `storage="memory"` 用于差分测试和小任务；当磁盘路径完成服务器
验收后，将 `auto` 设为默认：根据占据组态数和保守上界选择内存或磁盘。不能在生成一半后才
从 memory 自动切换到 disk，因为这会复杂化顺序和资源所有权。

修正 `_generate_outputs`：

- 配置路径要求描述符时始终生成 V2。
- `generate_descriptors` 只决定是否发布最终描述符，不改变内部权威 V2 生成。
- Python 不再负责串联“生成文本 → 转 Parquet → 生成描述符”的核心事务。
- 输出统计增加 stage、scratch、generated/unique/duplicate、disk bytes 和峰值内存。

如果用户 TOML 中实际包含 `4s(2,\*)`，TOML basic string 会因非法转义失败；合法文本是
`4s(2,*)`。文档示例不得为了 Markdown 显示向真实 TOML 添加反斜杠。

## 17. 分阶段实施

### Phase 0：基线和可测量性

1. 固定目标配置的规范 transcript 测试夹具，但完整服务器输入不进入常规 CI。
2. 增加只枚举占据组态的 `plan-generation` 内部接口/CLI dry-run，输出组态数、Peel 数和粗略
   磁盘预算，不生成 CSF。
3. 增加阶段计时、记录数、分配容量和可选 RSS 采样。
4. 修正配置生成路径仍强制 V1 的问题。

门禁：dry-run 对目标配置稳定报告 7,031,941 个组态、86,032,095 个相对子壳层条目和
56 个活动相对子壳层。

### Phase 1：生成器 seam（已实施，待提交）

1. 新增 `GeneratedRecordRef` 与 `GeneratedRecordSink`。
2. 用 `CompleteCsfSink` 重现现有 `generate_csfs`。
3. 删除 `Generator` 对 `CompleteCsfFile` 的直接依赖。
4. 串行和组态内部并行分支都通过 sink，不再先生成 branch `CompleteCsfFile` 再复制。

门禁：现有全部 Rust 测试通过；注册样例的记录、块和规范文本逐字节不变。

实施记录：`GeneratedRecordRef`、`GeneratedRecordSink` 和 `CompleteCsfSink` 已加入
`rCSFs/src/csf_generation/mod.rs`。串行递归和组态内部并行分支都通过 sink；并行分支只暂存
无文件头的整数记录，再按原顺序交给最终 sink。新增回归测试比较记录 sink 与
`CompleteCsfSink` 在串行及并行分支下的结果；`cargo test` 通过。

### Phase 2：range segment 流式 V2（已实施，待提交）

1. 预计算 Peel 表。
2. 划分 `GenerationRange`。
3. 实现 `DescriptorBatchSink` 和 Arrow segment writer。
4. 实现受控线程池、取消和阶段统计。
5. 暂不去重，按块/range 顺序合并为最终 descriptor。

门禁：小/中型配置的 memory 与 disk 两条路径逐行 V2 相同；线程数 1、2、最大核心数输出相同；
故障注入不留下最终文件。

实施记录：新增 `rCSFs/src/csf_generation/streaming.rs`。它预计算 Peel 表、按固定大小生成
确定性 `GenerationRange`、以受控 Rayon 线程池并行处理 range，并将 `DescriptorBatchSink` 的
V2 行写入每个 `(2J, parity, range)` 的 Arrow IPC segment；segment 包含内部
`range_ordinal`/`local_ordinal` 排序列。合并阶段按块、range 和本地序号写出最终 V2 Parquet，
去除这两个内部列，并在任一 segment 读取或写入失败时不发布最终文件。新增测试验证：range
划分、Peel 表、线程数 1/2 的逐行等价，以及失败合并不留下输出。现有 CLI 仍走旧路径，待
Phase 4 的 `generate_outputs` 输出事务统一接线。

### Phase 3：磁盘精确去重（已实施，待提交）

1. 实现稳定 128 位哈希和按块分桶。
2. 实现超预算桶递归拆分。
3. 实现完整键比较和 survivor bitset。
4. 第二遍稳定输出并计算 `block_lengths`。

门禁：跨 range、跨 segment、哈希碰撞、seniority 不同、缺失与显式零不同等测试全部通过；
首次出现顺序保持不变。

实施记录：`streaming.rs` 现在为每个 `(2J, parity)` 块计算完整 V2 整数行的稳定 SHA-256
前 128 位摘要，并将 `{块内序号、摘要、完整行}` 写入磁盘 bucket。超过
`max_rows_per_bucket` 的 bucket 使用与主摘要独立的稳定摘要递归拆分；超过显式深度上限会返回
错误而不是继续扩大内存。叶 bucket 只将所有整数完全相等的后续行视为重复，首行写入块级
`survivors.bitset`。第二遍按原来的 block/range/local 顺序重扫 Arrow segment，只输出位图保留
的行，并将 generated、unique、duplicate 计数和 `block_lengths` 写入 Parquet KV metadata。测试
覆盖了强制摘要碰撞下 seniority 差异、缺失值与显式零的保留，以及递归分桶后跨 segment 的首行
顺序保持；`cargo test` 全部通过。CLI 接线和最终 CSF 恢复仍由 Phase 4 负责。

### Phase 4：流式恢复和输出事务

1. 把 `py_restore_csfs_from_descriptors` 改为 RecordBatch 流式解码。
2. 抽取单记录规范写出接口，不构建完整 `CompleteCsfFile`。
3. 实现 `OutputTransaction`。
4. CLI 切换到新的 `generate_outputs` 深模块。
5. 发布可选 CSF 三行 Parquet。

门禁：端到端结果与 memory 路径逐字节一致；恢复峰值内存不随 N 线性增长。

### Phase 5：占据组态和恢复能力增强

1. 消除 `EnumeratedConfiguration -> GenerationRequest` 全量克隆。
2. 评估多参考 merge 的移动语义或外部去重。
3. 增加 `--resume`。
4. 根据实测将 `storage=auto` 设为默认。

此阶段不阻塞当前 77,362,152 条目标输入的首次服务器验证，但决定更大输入的上限。

## 18. 逐文件修改清单

| 文件 | 修改 |
| --- | --- |
| `rCSFs/src/csf_generation/mod.rs` | 引入 sink seam；`Generator::emit` 推送借用记录；移除分支结果复制；保留 `CompleteCsfSink`。 |
| `rCSFs/src/csf_generation/occupations.rs` | dry-run 统计、Peel 预计算；Phase 5 消除全量 request 克隆和优化 merge。 |
| `rCSFs/src/csf_generation/pipeline.rs` | 新的深模块编排入口、range 划分、统计、取消、最终块顺序。旧入口作为 memory adapter。 |
| `rCSFs/src/descriptor_v2.rs` | 为 `GeneratedRecordRef` 提供零额外分配编码；保留严格 row 校验；新增批量/流式恢复辅助。 |
| `rCSFs/src/descriptor_schema.rs` | 临时与最终 schema 的权威列名、metadata 版本、去重 identity 定义。 |
| `rCSFs/src/complete_csf.rs` | 抽取规范单记录 writer；完整模型继续服务兼容路径，不参与流式全集恢复。 |
| `rCSFs/src/streaming_generation.rs`（新增） | range worker、segment writer、取消与内存预算。 |
| `rCSFs/src/external_dedup.rs`（新增） | 分桶、完整键比较、递归拆桶、survivor bitset 和统计。 |
| `rCSFs/src/output_transaction.rs`（可扩展 `atomic_output.rs`） | 多文件路径校验、暂存、验证、发布和失败回滚。 |
| `rCSFs/src/csfs_descriptor.rs` | 最终 Parquet writer 复用；恢复改为 RecordBatch 流式；严格 schema/null/metadata 校验。 |
| `rCSFs/src/lib.rs` | 注册新的 PyO3 生成/恢复入口与统计。 |
| `rCSFs/rcsfs/__init__.py` | Python 包装和公共类型；默认 V2；公开 storage/scratch/memory budget。 |
| `rCSFs/rcsfs/_rcsfs.pyi`、`_types.py` | 同步接口和阶段统计字段。 |
| `rCSFs/rcsfs/cli.py` | 删除 Python 级多阶段核心编排；解析新选项；修正强制 V1；打印资源统计。 |
| `rCSFs/tests/` | sink 差分、顺序、去重、恢复、故障、磁盘预算和端到端测试。 |

`graspkit-tools` 的训练读取接口无需因生成实现改变。最终 V2 schema 和行序保持现有合同；只需做
一次端到端回归，确认 CI 标签与 descriptor 行仍按同一索引对齐。

## 19. 测试矩阵

### 19.1 生成等价性

- 单组态、单 CSF。
- 多组态、多个 J/宇称块。
- seniority 相同 J 的不同态。
- 显式 J=0/K=0 与缺失值。
- trailing empty Peel 位置。
- 两参考列表存在重叠。
- 无可达目标 J 的组态。
- 1、2、8、最大线程数。

断言 memory/disk adapter 的：记录数、V2 行、块长度、CSF 字节完全一致。

### 19.2 去重

- 同 segment 重复。
- 不同 segment 重复。
- 不同 range 重复。
- 相同哈希、不同完整行的人工碰撞。
- occupation 相同但 seniority 不同。
- occupation/state 相同但中间耦合不同。
- 缺失值和显式零不同。
- 重复记录保留首次出现顺序。

### 19.3 失败与清理

- scratch 空间不足。
- segment writer 写失败。
- worker 返回物理校验错误。
- bucket 文件截断。
- descriptor schema/KV 被篡改。
- header hash/block_lengths 不匹配。
- 最终输出在发布前被其他进程创建。
- 用户中断和 worker panic。

断言没有半成品最终输出、没有通道死锁，错误包含阶段、range/row 和根因。

### 19.4 内存边界

CI 使用可控合成生成器产生大量重复记录，记录以下内部上界：

- 同时存活的 V2 batch 数不超过配置。
- 单 worker builder 容量不超过预算。
- 超大桶触发拆分而不是继续扩容。
- streaming restore 不保存历史 batch。

RSS 测试仅作为平台允许时的性能门禁，不能替代上述结构性断言。

## 20. 验收门禁

### 20.1 常规门禁

从 `rCSFs/` 并使用 `graspkit-tools/.venv`：

```bash
cargo test
cargo fmt --check
python -m pytest
ruff check .
ruff format --check .
basedpyright rcsfs
```

Rust 改动后按仓库说明同时刷新树内扩展和 Tools 共享环境，再运行 Tools 相关 pytest、Ruff 和
BasedPyright。不得用旧 `.so` 的测试结果验收新源码。

### 20.2 服务器分级验收

1. `excitations=2`：与旧 memory 路径逐字节比较。
2. `excitations=3`：记录每阶段时间、RSS、scratch 和压缩率。
3. 目标 `excitations=4`：使用本地 NVMe scratch 和显式内存预算。

目标输入最终必须满足：

- 枚举统计为 7,031,941 个唯一占据组态和 56 个活动相对子壳层。
- 最终 CSF 数与确认后的 GRASP 基准一致；若去重前后数量不同，同时报告两者和重复数。
- `sum(block_lengths) == descriptor rows == CSF records == CSF Parquet rows`。
- descriptor 随机抽样恢复和完整流式恢复均通过严格解析。
- 不同线程数得到相同内容哈希。
- 峰值 RSS 低于配置预算，并为操作系统、SSH 和文件缓存预留空间。
- 失败不会令服务器失去 SSH 响应；磁盘不足会提前终止。

若 rCSFs 的去重后数量与 GRASP 的 77,362,152 不同，先比较占据组态、每个 J/宇称块计数和
首个差异记录，不能为了匹配总数而放宽完整身份键。

## 21. 完成定义

只有以下条件全部满足，才能宣布大规模流式生成完成：

- 新路径不在任一阶段保存全部 CSF 或全部稠密 V2 行。
- V2 是生成到恢复之间唯一的权威记录语义。
- 去重使用完整键并保留首次出现顺序。
- 完整恢复逐批进行，内存不随总记录数线性增长。
- memory 与 disk adapter 在可比较输入上输出完全一致。
- CLI 配置路径不再强制 V1。
- 输出集合安全发布，错误路径无半成品。
- 目标服务器输入成功完成并留下可复核的阶段统计。
- `graspkit-tools` 的行号、标签和训练输入对齐回归通过。
