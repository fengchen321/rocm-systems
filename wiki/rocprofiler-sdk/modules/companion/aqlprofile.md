# AQL/PM4 Profiling 包生成 (AQLProfile)

## 概述

AQLProfile 模块 (`source/lib/aqlprofile/`) 是 rocprofiler-sdk 的 GPU 性能计数器采集核心，负责生成 AQL (Architected Queueing Language) 和 PM4 命令包以配置和读取 GPU 硬件性能计数器。该模块实现了与 AMD GPU 驱动的底层接口，支持多种 GPU 架构（GFX9、GFX10、GFX11、GFX12 等），提供 PMC (Performance Monitor Counter)、SPM (Streaming Performance Monitor) 和 SQTT (Shader Queue Thread Trace) 三种 profiling 模式。它是 rocprofiler-sdk 中最底层的硬件接口模块之一。

## 关键文件

| 文件 | 路径 | 职责 |
|------|------|------|
| `aqlprofile.hpp` | `source/lib/aqlprofile/aqlprofile.hpp` | AQLProfile 接口头文件，处理内部/外部 AQLProfile 切换 |
| `aql_profile_v2.h` | `source/lib/aqlprofile/aql_profile_v2.h` | AQLProfile v2 API 定义，包含句柄、事件、参数等类型 |
| `hsa_includes.h` | `source/lib/aqlprofile/hsa_includes.h` | HSA 运行时头文件包含 |
| `core/aql_profile.hpp` | `source/lib/aqlprofile/core/aql_profile.hpp` | 核心 AQLProfile 内部接口定义 |
| `core/aql_profile.cpp` | `source/lib/aqlprofile/core/aql_profile.cpp` | AQLProfile 核心实现：命令包生成、SPM 数据处理 |
| `core/pm4_factory.h` | `source/lib/aqlprofile/core/pm4_factory.h` | PM4 工厂类，根据 GPU 架构创建对应的 PM4 构建器 |
| `core/commandbuffermgr.hpp` | `source/lib/aqlprofile/core/commandbuffermgr.hpp` | 命令缓冲区分区管理器 |
| `core/memorymanager.hpp` | `source/lib/aqlprofile/core/memorymanager.hpp` | GPU 内存管理器 |
| `core/counters.cpp` | `source/lib/aqlprofile/core/counters.cpp` | 计数器数据处理 |
| `core/gfx9_factory.cpp` | `source/lib/aqlprofile/core/gfx9_factory.cpp` | GFX9 架构 PM4 工厂实现 |
| `core/gfx10_factory.cpp` | `source/lib/aqlprofile/core/gfx10_factory.cpp` | GFX10 架构 PM4 工厂实现 |
| `core/gfx11_factory.cpp` | `source/lib/aqlprofile/core/gfx11_factory.cpp` | GFX11 架构 PM4 工厂实现 |
| `core/gfx12_factory.cpp` | `source/lib/aqlprofile/core/gfx12_factory.cpp` | GFX12 架构 PM4 工厂实现 |
| `core/gfx1250_factory.cpp` | `source/lib/aqlprofile/core/gfx1250_factory.cpp` | GFX1250 (MI450) 架构 PM4 工厂实现 |
| `core/populate_aql.cpp` | `source/lib/aqlprofile/core/populate_aql.cpp` | AQL 包填充实现 |
| `core/spm_decode.cpp` | `source/lib/aqlprofile/core/spm_decode.cpp` | SPM 数据解码 |
| `core/spm_v2.cpp` | `source/lib/aqlprofile/core/spm_v2.cpp` | SPM v2 接口实现 |
| `core/threadtrace.cpp` | `source/lib/aqlprofile/core/threadtrace.cpp` | 线程追踪实现 |
| `core/last_error.cpp` | `source/lib/aqlprofile/core/last_error.cpp` | 最后错误信息管理 |
| `core/ip_offset_table_init.cpp` | `source/lib/aqlprofile/core/ip_offset_table_init.cpp` | IP 偏移表初始化 |
| `core/parse_ip_discovery.cpp` | `source/lib/aqlprofile/core/parse_ip_discovery.cpp` | IP 发现数据解析 |
| `pm4/cmd_builder.h` | `source/lib/aqlprofile/pm4/cmd_builder.h` | PM4 命令构建器基类 |
| `pm4/pmc_builder.h` | `source/lib/aqlprofile/pm4/pmc_builder.h` | PMC PM4 包构建器 |
| `pm4/spm_builder.h` | `source/lib/aqlprofile/pm4/spm_builder.h` | SPM PM4 包构建器 |
| `pm4/sqtt_builder.h` | `source/lib/aqlprofile/pm4/sqtt_builder.h` | SQTT PM4 包构建器 |
| `util/hsa_rsrc_factory.h` | `source/lib/aqlprofile/util/hsa_rsrc_factory.h` | HSA 资源工厂，管理 HSA API 表 |
| `def/gpu_block_info.h` | `source/lib/aqlprofile/def/gpu_block_info.h` | GPU 块信息定义 |

## 核心数据结构

- **`Pm4Factory`** (`core/pm4_factory.h`): PM4 工厂类，是 AQLProfile 的核心抽象。根据 GPU 架构创建对应的 PM4 构建器实例。关键成员：
  - `cmd_builder_`: PM4 命令构建器
  - `pmc_builder_`: PMC PM4 包构建器
  - `spm_builder_`: SPM PM4 包构建器
  - `sqtt_builder_`: SQTT PM4 包构建器
  - `agent_info_`: GPU 代理信息
  - `gpu_id_`: GPU 架构标识（`GFX9_GPU_ID`、`MI300_GPU_ID` 等）
  - `block_map_`: GPU 块信息映射表
  - 提供 `Create()` 静态工厂方法和 `Destroy()` 方法<!-- verified: 2026-05-27 -->

- **`gpu_id_t`** (`core/pm4_factory.h`): GPU 架构枚举：
  - `GFX9_GPU_ID`: 通用 GFX9
  - `MI100_GPU_ID`、`MI200_GPU_ID`、`MI300_GPU_ID`、`MI350_GPU_ID`、`MI450_GPU_ID`: 特定 MI 系列
  - `GFX10_GPU_ID`、`GFX11_GPU_ID`、`GFX115X_GPU_ID`、`GFX12_GPU_ID`: 其他架构<!-- verified: 2026-05-27 -->

- **`BlockInfoMap`** (`core/pm4_factory.h`): GPU 块信息映射类，提供 `Get(block_id)` 和 `Find(name)` 方法查询 GPU 块信息。<!-- verified: 2026-05-27 -->

- **`CommandBufferMgr`** (`core/commandbuffermgr.hpp`): 命令缓冲区管理器，将命令缓冲区分为 prefix、rdcmds、rd2cmds、precmds、postcmds 等区域。关键方法：
  - `AddPrefix()`: 添加前缀命令
  - `SetRdSize()` / `SetRd2Size()`: 设置读取命令大小
  - `SetPreSize()` / `SetPostSize()`: 设置前置/后置命令大小<!-- verified: 2026-05-27 -->

- **`profile_t`** (`aql_profile_v2.h`): Profile 配置结构体（`hsa_ven_amd_aqlprofile_profile_t`），包含：
  - `agent`: GPU 代理句柄
  - `events`: 事件数组
  - `event_count`: 事件数量
  - `parameters`: 参数数组
  - `command_buffer`: 命令缓冲区描述符
  - `output_buffer`: 输出缓冲区描述符<!-- verified: 2026-05-27 -->

- **`event_t`** (`aql_profile_v2.h`): 事件结构体（`hsa_ven_amd_aqlprofile_event_t`），包含：
  - `block_name`: GPU 块名称
  - `block_index`: 块索引
  - `counter_id`: 计数器 ID<!-- verified: 2026-05-27 -->

- **`pm4_agent_info`** (`core/pm4_factory.h`): GPU 代理信息结构体，包含：
  - `agent_gfxip`: GFXIP 名称
  - `cu_num`: 计算单元数量
  - `se_num`: Shader Engine 数量
  - `shader_arrays_per_se`: 每个 SE 的 Shader Array 数量
  - `xcc_num`: XCC 数量<!-- verified: 2026-05-27 -->

- **`GpuBlockInfo`** (`def/gpu_block_info.h`): GPU 块信息结构体，包含块名称、实例数量、计数器数量、属性标志等。<!-- verified: 2026-05-27 -->

## 关键函数

- **`Pm4Factory::Create(const hsa_agent_t agent, bool concurrent)`** (`core/pm4_factory.h`): 根据 HSA 代理创建 PM4 工厂。通过 `hsa_agent_get_info` 获取代理名称和芯片 ID，调用 `GetGpuId()` 确定 GPU 架构，然后创建对应的工厂实例。使用 `instances_` 静态映射缓存已创建的实例。<!-- verified: 2026-05-27 -->

- **`Pm4Factory::GetGpuId(std::string_view gfx_ip)`** (`core/pm4_factory.h`): 根据 GFXIP 名称字符串确定 GPU 架构 ID。使用前缀匹配策略，更具体的 ID 必须在更通用的 ID 之前。<!-- verified: 2026-05-27 -->

- **`Pm4Factory::Destroy()`** (`core/pm4_factory.h`): 销毁所有 PM4 工厂实例，释放内存。<!-- verified: 2026-05-27 -->

- **`PopulateAql(cmd_buffer, cmd_size, cmd_writer, aql_packet)`** (`core/aql_profile.hpp`): 将 PM4 命令缓冲区填充到 AQL 包中。<!-- verified: 2026-05-27 -->

- **`LegacyAqlAcquire(aql_packet, data)` / `LegacyAqlRelease(aql_packet, data)`** (`core/aql_profile.hpp`): 旧版 AQL 获取/释放接口。<!-- verified: 2026-05-27 -->

- **`spm_iterate_data(profile, callback, data)`** (`core/aql_profile.cpp`): 使用驱动 API 迭代 SPM 数据。<!-- verified: 2026-05-27 -->

- **`RegisterAgent(agent_info)`** (`core/pm4_factory.h`): 注册 GPU 代理信息，返回 `aqlprofile_agent_handle_t` 句柄。<!-- verified: 2026-05-27 -->

- **`GetAgentInfo(agent_id)`** (`core/pm4_factory.h`): 根据代理句柄获取 `AgentInfo` 指针。<!-- verified: 2026-05-27 -->

- **`hsa_rsrc_factory_init(hsa_api_table)`** (`aqlprofile.hpp`): 初始化 HSA 资源工厂，仅在内部 AQLProfile 模式下执行实际初始化。<!-- verified: 2026-05-27 -->

## 调用关系

- **上游**：
  - `rocprofiler-sdk` 核心库：通过 `aql_profile_v2.h` 定义的 C API 调用 AQLProfile 功能
  - `rocprofiler-sdk` PMC 模块：使用 `Pm4Factory` 生成性能计数器配置命令
  - `rocprofiler-sdk` SPM 模块：使用 SPM 构建器生成流式性能监控命令
  - `rocprofiler-sdk` 线程追踪模块：使用 SQTT 构建器生成线程追踪命令
  - `rocprofiler-sdk` 输出库：通过 CMakeLists.txt 链接依赖

- **下游**：
  - HSA 运行时：通过 `hsa_rsrc_factory.h` 中的 HSA API 表调用 HSA 运行时函数
  - GPU 驱动：PM4 命令包最终由 GPU 驱动执行
  - `abseil-cpp`：使用 Abseil 容器和工具
  - `fmt`：使用 fmt 格式化库

## 数据流

1. **代理注册**：`RegisterAgent()` 注册 GPU 代理信息（GFXIP、CU 数量等）
2. **工厂创建**：`Pm4Factory::Create()` 根据代理信息创建对应架构的工厂实例
3. **事件配置**：用户指定要监控的 GPU 性能事件（块名称、块索引、计数器 ID）
4. **命令生成**：PM4 构建器（`PmcBuilder`、`SpmBuilder`、`SqttBuilder`）生成 PM4 命令包
5. **AQL 填充**：`PopulateAql()` 将 PM4 命令填充到 AQL 包中
6. **命令提交**：AQL 包提交到 GPU 队列执行
7. **数据采集**：GPU 执行 PM4 命令，将性能计数器数据写入输出缓冲区
8. **数据读取**：`counters.cpp` 和 `spm_decode.cpp` 处理采集到的原始数据

## 已知限制与边界情况

- **GPU 架构支持**：仅支持 AMD GPU 架构（GFX9-GFX12、MI100-MI450）。不支持的架构返回 `INVAL_GPU_ID` 并抛出异常。
- **内部/外部模式**：通过 `ROCPROFILER_EXTERNAL_AQLPROFILE` 宏控制使用内部还是外部 AQLProfile 库。外部模式下不执行 HSA 资源工厂初始化。
- **并发模式**：`Pm4Factory::CheckConcurrent()` 检查 `HSA_VEN_AMD_AQLPROFILE_PARAMETER_NAME_K_CONCURRENT` 参数以确定是否使用并发模式。
- **SPM KFD 模式**：通过 `ROCP_SPM_KFD_MODE` 环境变量控制 SPM 数据获取方式（使用驱动公共 API）。
- **异常处理**：使用 `event_exception` 和 `aql_profile_exc_msg` 异常类报告错误。无效块名称、块索引越界等都会抛出异常。
- **实例缓存**：`Pm4Factory` 使用静态 `instances_` 映射缓存已创建的实例，避免重复创建。`Destroy()` 会清除所有缓存。
- **线程安全**：`Pm4Factory` 使用互斥锁 (`mutex_`) 保护实例创建和销毁操作。
- **命令缓冲区大小**：`CommandBufferMgr` 在缓冲区大小为零时抛出异常。

## 与官方文档的关系

- [使用 PC 采样](source/docs/how-to/using-pc-sampling.rst) - PC 采样功能使用指南
- [CDNA3/CDNA4 PC 采样](source/docs/how-to/cdna3-cdna4-pc-sampling.rst) - CDNA 架构特定的 PC 采样配置
- [使用线程追踪](source/docs/how-to/using-thread-trace.rst) - 线程追踪功能使用指南
- [使用 rocprofv3](source/docs/how-to/using-rocprofv3.rst) - rocprofv3 工具的完整使用指南
