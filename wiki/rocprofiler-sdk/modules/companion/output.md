# 输出格式化器 (Output Formatters)

## 概述

输出格式化器模块 (`source/lib/output/`) 是 rocprofiler-sdk 的数据输出层，负责将收集到的性能分析数据转换为多种输出格式。该模块位于整体架构的最下游，接收来自缓冲区追踪和回调追踪模块的原始数据记录，通过统计聚合和格式化处理，输出为 CSV、JSON、Perfetto (PFTrace)、OTF2 和 ROCPD (SQLite3) 等格式。它是 `rocprofv3` 工具的核心输出引擎。

## 关键文件

| 文件 | 路径 | 职责 |
|------|------|------|
| `generator.hpp` | `source/lib/output/generator.hpp` | 定义 `generator<Tp>` 和 `file_generator<Tp>` 模板，提供数据迭代器抽象 |
| `buffered_output.hpp` | `source/lib/output/buffered_output.hpp` | 定义 `buffered_output<Tp, DomainT>` 模板，管理缓冲区数据的刷新和读取 |
| `output_config.hpp` | `source/lib/output/output_config.hpp` | 定义 `output_config` 结构体，存储所有输出配置选项 |
| `output_stream.hpp` | `source/lib/output/output_stream.hpp` | 定义 `output_stream` 结构体，封装输出流的生命周期管理 |
| `metadata.hpp` | `source/lib/output/metadata.hpp` | 定义 `metadata` 结构体，聚合所有运行时元数据（代理、内核符号、代码对象等） |
| `statistics.hpp` | `source/lib/output/statistics.hpp` | 定义 `statistics<Tp, Fp>` 模板类，提供统计累积功能（计数、求和、最小/最大、方差等） |
| `generateCSV.hpp` | `source/lib/output/generateCSV.hpp` | CSV 格式生成器接口 |
| `generateJSON.hpp` | `source/lib/output/generateJSON.hpp` | JSON 格式生成器接口 |
| `generatePerfetto.hpp` | `source/lib/output/generatePerfetto.hpp` | Perfetto 格式生成器接口 |
| `generateOTF2.hpp` | `source/lib/output/generateOTF2.hpp` | OTF2 格式生成器接口 |
| `generateRocpd.hpp` | `source/lib/output/generateRocpd.hpp` | ROCPD (SQLite3) 格式生成器接口 |
| `generateStats.hpp` | `source/lib/output/generateStats.hpp` | 统计数据生成器接口 |
| `domain_type.hpp` | `source/lib/output/domain_type.hpp` | 定义 `domain_type` 枚举，标识追踪域类型 |
| `format_path.hpp` | `source/lib/output/format_path.hpp` | 路径格式化工具，支持 MPI rank/size 等占位符替换 |
| `tmp_file_buffer.hpp` | `source/lib/output/tmp_file_buffer.hpp` | 临时文件缓冲区管理 |
| `csv_output_file.hpp` | `source/lib/output/csv_output_file.hpp` | CSV 输出文件封装 |
| `agent_info.hpp` | `source/lib/output/agent_info.hpp` | GPU/CPU 代理信息数据结构 |
| `kernel_symbol_info.hpp` | `source/lib/output/kernel_symbol_info.hpp` | 内核符号信息数据结构 |
| `counter_info.hpp` | `source/lib/output/counter_info.hpp` | 性能计数器信息数据结构 |

## 核心数据结构

- **`generator<Tp>`** (`generator.hpp`): 抽象基类，提供数据记录的迭代接口。通过 `get(size_t idx)` 方法按索引获取数据批次。`file_generator<Tp>` 是其派生类，从临时文件中按位置读取数据。<!-- verified: 2026-05-27 -->

- **`buffered_output<Tp, DomainT>`** (`buffered_output.hpp`): 模板类，将数据记录与特定追踪域关联。提供 `flush()`（刷新到临时文件）、`read()`（读取临时文件）、`get_generator()`（获取数据生成器）等方法。每个追踪域（如 HIP API、HSA API、内核调度等）都有独立的 `buffered_output` 实例。<!-- verified: 2026-05-27 -->

- **`output_config`** (`output_config.hpp`): 输出配置结构体，包含：
  - 格式开关：`csv_output`、`json_output`、`pftrace_output`、`otf2_output`、`rocpd_output`
  - 路径配置：`output_path`、`output_file`、`tmp_directory`
  - 统计选项：`stats`、`stats_summary`、`stats_summary_per_domain`
  - Perfetto 配置：`perfetto_buffer_size`、`perfetto_shmem_size_hint`、`perfetto_backend`
  - 从环境变量加载：`load_from_env()` 静态方法<!-- verified: 2026-05-27 -->

- **`metadata`** (`metadata.hpp`): 元数据聚合结构体，包含：
  - 进程信息：`process_id`、`parent_process_id`、`process_start_ns`
  - 代理信息：`agents`（`agent_info_vec_t`）、`agents_map`
  - 符号信息：`kernel_symbols`、`code_objects`、`host_functions`
  - 标记消息：`marker_messages`
  - 计数器信息：`agent_counter_info`
  - 提供 `get_kernel_name()`、`get_kind_name()`、`get_operation_name()` 等查询方法<!-- verified: 2026-05-27 -->

- **`statistics<Tp, Fp>`** (`statistics.hpp`): 统计累积模板类，维护计数 (`m_cnt`)、求和 (`m_sum`)、平方和 (`m_sqr`)、最小值 (`m_min`)、最大值 (`m_max`)，提供 `get_mean()`、`get_variance()`、`get_stddev()`、`get_percent()` 等计算方法。<!-- verified: 2026-05-27 -->

- **`domain_type`** (`domain_type.hpp`): 枚举类型，定义追踪域：`HSA`、`HIP`、`MARKER`、`KERNEL_DISPATCH`、`MEMORY_COPY`、`SCRATCH_MEMORY`、`COUNTER_COLLECTION`、`RCCL`、`MEMORY_ALLOCATION`、`PC_SAMPLING_HOST_TRAP`、`ROCDECODE`、`ROCJPEG`、`PC_SAMPLING_STOCHASTIC`、`KFD`。<!-- verified: 2026-05-27 -->

- **`output_stream`** (`output_stream.hpp`): 输出流封装结构体，管理 `std::ostream*` 指针和析构函数，支持文件流和标准流。<!-- verified: 2026-05-27 -->

## 关键函数

- **`generate_csv()`** (`generateCSV.hpp`): 多重重载函数，将各种追踪数据记录格式化为 CSV 输出。每种追踪域（HIP API、HSA API、内核调度、内存拷贝、标记、计数器等）都有独立的重载。接收 `output_config`、`metadata`、`generator<Tp>` 和 `stats_entry_t` 参数。<!-- verified: 2026-05-27 -->

- **`write_json()`** (`generateJSON.hpp`): 将所有追踪数据写入 JSON 格式。接收 `json_output` 归档对象和所有追踪域的生成器。使用 cereal 序列化库的 `MinimalJSONOutputArchive`。<!-- verified: 2026-05-27 -->

- **`write_perfetto()`** (`generatePerfetto.hpp`): 将追踪数据写入 Perfetto 格式。接收代理数据和各追踪域的生成器。<!-- verified: 2026-05-27 -->

- **`write_otf2()`** (`generateOTF2.hpp`): 将追踪数据写入 OTF2 格式。接收代理数据和各追踪域的 deque 指针。<!-- verified: 2026-05-27 -->

- **`write_rocpd()`** (`generateRocpd.hpp`): 将追踪数据写入 ROCPD (SQLite3) 格式。接收代理数据和各追踪域的生成器。<!-- verified: 2026-05-27 -->

- **`generate_stats()`** (`generateStats.hpp`): 多重重载函数，对各种追踪数据进行统计聚合，返回 `stats_entry_t`。<!-- verified: 2026-05-27 -->

- **`format_path()`** (`format_path.hpp`): 路径格式化函数，替换 `%hostname%`、`%pid%`、`%rank%` 等占位符。<!-- verified: 2026-05-27 -->

- **`get_output_stream()`** (`output_stream.hpp`): 根据配置创建输出流，支持文件和标准输出。<!-- verified: 2026-05-27 -->

- **`get_buffer_elements()`** (`generator.hpp`): 从环形缓冲区容器中提取所有元素到标准容器。<!-- verified: 2026-05-27 -->

## 调用关系

- **上游**：
  - `rocprofv3` 工具（`source/bin/rocprofv3/`）在 profiling 会话结束时调用各 `generate_*` / `write_*` 函数输出数据
  - `buffered_output` 被回调追踪和缓冲区追踪模块填充数据

- **下游**：
  - `rocprofiler-sdk-rocpd` 库：ROCPD 输出格式依赖 rocpd 库的 SQL schema 和类型定义
  - `cereal` 序列化库：JSON 输出使用 `cereal::MinimalJSONOutputArchive`
  - `perfetto` SDK：Perfetto 输出使用 Perfetto trace SDK
  - `OTF2` 库：OTF2 输出使用 OTF2 API
  - `SQLite3` 库：ROCPD 输出使用 SQLite3 C API
  - `fmt` 库：CSV 和统计输出使用 fmt 格式化
  - `rocprofiler-sdk-aqlprofile`：通过 CMakeLists.txt 中的链接依赖

## 数据流

1. **数据收集阶段**：回调追踪和缓冲区追踪模块将数据记录写入环形缓冲区 (`ring_buffer`)
2. **缓冲区刷新**：`buffered_output::flush()` 将环形缓冲区数据写入临时文件
3. **数据读取**：`buffered_output::read()` 从临时文件读取数据
4. **生成器创建**：`get_generator()` 创建 `file_generator` 实例，提供数据迭代接口
5. **格式化输出**：各 `generate_*` / `write_*` 函数遍历生成器，将数据记录转换为目标格式
6. **统计聚合**：`generate_stats()` 对数据记录进行统计计算，生成 `stats_entry_t`
7. **输出写入**：通过 `output_stream` 将格式化数据写入文件或标准输出

## 已知限制与边界情况

- **临时文件依赖**：大量数据通过临时文件中转，需要足够的磁盘空间。临时目录可通过 `tmp_directory` 配置。
- **内存使用**：`get_buffer_elements()` 将所有数据加载到内存，对于非常大的追踪数据集可能导致内存压力。
- **格式互斥**：虽然支持同时输出多种格式，但每种格式的生成是独立的，没有共享中间表示。
- **MPI 支持**：路径格式化支持 MPI rank/size 占位符，但需要正确的环境变量设置 (`ROCPROF_MPI_RANK_VAR`、`ROCPROF_MPI_SIZE_VAR`)。
- **Perfetto 缓冲区大小**：默认 Perfetto 缓冲区大小为 1 GiB，可通过 `perfetto_buffer_size` 配置。过小的缓冲区可能导致数据丢失。
- **错误处理**：`file_generator::get()` 在索引越界时通过 `ROCP_ERROR` 日志报告错误并返回空数据。
- **线程安全**：`file_generator` 在构造时获取文件互斥锁，确保文件位置集合的一致性。

## 与官方文档的关系

- [使用 rocpd 输出格式](source/docs/how-to/using-rocpd-output-format.rst) - ROCPD 输出格式的使用指南
- [rocprofv3 I/O 控制选项](source/docs/how-to/rocprofv3-io-options.rst) - 输出路径和格式配置选项
- [使用 rocprofv3](source/docs/how-to/using-rocprofv3.rst) - rocprofv3 工具的完整使用指南
- [高级 rocprofv3 选项](source/docs/how-to/advanced-rocprofv3-options.rst) - 高级输出配置选项
