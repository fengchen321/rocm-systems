# GPU Agent 管理模块

## 概述

Agent 模块负责发现和管理系统中的所有计算代理（CPU 和 GPU）。它是 rocprofiler-sdk 的基础模块，为其他所有追踪和分析功能提供设备拓扑信息。该模块通过读取 KFD（Kernel Fusion Driver）的 sysfs 拓扑节点来发现系统中的所有 agent，并将其映射到 HSA（Heterogeneous System Architecture）运行时的 agent 表示。每个 agent 包含详细的硬件属性信息，如 SIMD 数量、计算单元数、内存库、缓存层次和 IO 链路等。

## 关键文件

| 文件 | 路径 | 职责 |
|------|------|------|
| agent.cpp | source/lib/rocprofiler-sdk/agent.cpp | Agent 发现的核心实现，包括从 sysfs 读取拓扑、解析属性、建立 HSA 映射 |
| agent.hpp | source/lib/rocprofiler-sdk/agent.hpp | 内部头文件，声明 agent 管理的内部接口 |
| agent.h | source/include/rocprofiler-sdk/agent.h | 公共 API 头文件，定义 rocprofiler_agent_v0_t 结构体和查询接口 |
| agent_cache.hpp | source/lib/rocprofiler-sdk/hsa/agent_cache.hpp | HSA agent 缓存，存储 agent 的 HSA 运行时信息 |

<!-- verified: 2026-05-27 -->

## 核心数据结构

### rocprofiler_agent_v0_t（公共 API）
- **定义位置**: `source/include/rocprofiler-sdk/agent.h:130`
- **职责**: 存储 agent 的完整属性信息
- **关键字段**:
  - `id` (rocprofiler_agent_id_t): 内部不透明标识符
  - `type` (rocprofiler_agent_type_t): agent 类型枚举（CPU、GPU）
  - `node_id` (uint32_t): KFD 拓扑节点 ID，等同于 HSA_AMD_AGENT_INFO_DRIVER_NODE_ID
  - `logical_node_id` (int32_t): 逻辑序列号，范围 [0..N)
  - `logical_node_type_id` (int32_t): 同类型 agent 的逻辑序列号
  - `cpu_cores_count` / `simd_count`: CPU 核心数和 SIMD 数量
  - `cu_count` (uint32_t): 计算单元数量
  - `gfx_target_version` (uint32_t): GPU 架构版本号
  - `name` / `product_name` / `vendor_name` (const char*): agent 名称信息
  - `uuid` (rocprofiler_uuid_t): GPU 唯一标识符
  - `runtime_visibility`: 运行时可见性位域（HSA、HIP、RCCL、rocdecode）
  - `mem_banks` / `caches` / `io_links`: 内存库、缓存和 IO 链路数组指针

<!-- verified: 2026-05-27 -->

### cpu_info（内部结构）
- **定义位置**: `source/lib/rocprofiler-sdk/agent.cpp:80`
- **职责**: 存储从 /proc/cpuinfo 解析的 CPU 信息
- **关键字段**: processor, family, model, physical_id, core_id, apicid, vendor_id, model_name

### bdf_info（内部结构）
- **定义位置**: `source/lib/rocprofiler-sdk/agent.cpp:98`
- **职责**: 存储 PCI Bus/Device/Function 信息
- **关键字段**: domain, bus, device, function

### agent_pair（内部结构）
- **定义位置**: `source/lib/rocprofiler-sdk/agent.cpp:1030`
- **职责**: 存储 rocprofiler agent 和 HSA agent 的映射关系
- **关键字段**: rocp_agent (const rocprofiler_agent_t*), hsa_agent (hsa_agent_t)

<!-- verified: 2026-05-27 -->

## 关键函数

### 公共 API

- **`rocprofiler_query_available_agents`** (`agent.h:293`)
  - 签名: `rocprofiler_status_t rocprofiler_query_available_agents(rocprofiler_agent_version_t version, rocprofiler_query_available_agents_cb_t callback, size_t agent_size, void* user_data)`
  - 职责: 通过回调函数向调用者提供所有可用 agent 的列表
  - 调用时机: 工具初始化后，需要查询系统 GPU/CPU 信息时

<!-- verified: 2026-05-27 -->

### 内部函数

- **`read_topology`** (`agent.cpp:619`)
  - 职责: 从 KFD sysfs 拓扑节点读取所有 agent 信息
  - 实现: 遍历 `/sys/class/kfd/kfd/topology/nodes` 目录下的节点，解析 properties 文件

- **`get_agents`** (`agent.cpp:1127`)
  - 签名: `std::vector<const rocprofiler_agent_t*> get_agents()`
  - 职责: 返回所有已发现 agent 的指针列表

- **`get_agent`** (`agent.cpp:1140`)
  - 签名: `const rocprofiler_agent_t* get_agent(rocprofiler_agent_id_t id)`
  - 职责: 根据 agent ID 查找并返回对应的 agent

- **`construct_agent_cache`** (`agent.cpp:1166`)
  - 签名: `void construct_agent_cache(::HsaApiTable* table)`
  - 职责: 建立 rocprofiler agent 与 HSA agent 的映射关系，创建 AgentCache
  - 调用时机: HSA API 表注册时（在 registration.cpp 中调用）

- **`get_hsa_agent`** (`agent.cpp:1395`)
  - 签名: `std::optional<hsa_agent_t> get_hsa_agent(const rocprofiler_agent_t* agent)`
  - 职责: 根据 rocprofiler agent 获取对应的 HSA agent

- **`get_rocprofiler_agent`** (`agent.cpp:1413`)
  - 签名: `const rocprofiler_agent_t* get_rocprofiler_agent(hsa_agent_t agent)`
  - 职责: 根据 HSA agent 获取对应的 rocprofiler agent

- **`get_agent_cache`** (`agent.cpp:1424`)
  - 签名: `const hsa::AgentCache* get_agent_cache(const rocprofiler_agent_t* agent)`
  - 职责: 获取 agent 的缓存信息

- **`update_agent_runtime_visibility`** (`agent.cpp:406`)
  - 职责: 根据环境变量（ROCR_VISIBLE_DEVICES、HIP_VISIBLE_DEVICES 等）更新 agent 的运行时可见性

- **`internal_refresh_topology`** (`agent.cpp:1454`)
  - 职责: 刷新 agent 拓扑（仅用于内部测试）

<!-- verified: 2026-05-27 -->

## 调用关系

### 上游（谁调用了本模块）
- **registration.cpp**: 在 `rocprofiler_set_api_table` 中调用 `construct_agent_cache` 建立 HSA agent 映射
- **context 模块**: 通过 `get_agents()` 获取 agent 列表用于计数器配置
- **counters 模块**: 使用 agent 信息配置硬件计数器收集
- **hsa/queue_controller**: 使用 agent 信息进行队列管理
- **pc_sampling 模块**: 使用 agent 信息配置 PC 采样

### 下游（本模块调用了谁）
- **lib/common/static_object**: 用于管理全局静态对象的生命周期
- **lib/common/string_entry**: 用于字符串的内部化存储
- **lib/common/environment**: 用于读取环境变量
- **hsa/agent_cache**: 用于缓存 HSA agent 信息
- **aqlprofile**: 用于注册 agent 到 AQL profile 系统
- **libdrm/amdgpu**: 用于获取 GPU 的营销名称和硬件信息

<!-- verified: 2026-05-27 -->

## 数据流

1. **发现阶段**: `read_topology()` 从 KFD sysfs 路径（`/sys/class/kfd/kfd/topology/nodes`）读取所有节点的 properties 文件
2. **解析阶段**: 对每个节点解析其属性（CPU 核心数、SIMD 数量等），判断 agent 类型（CPU/GPU）
3. **补充阶段**: 对 GPU agent，通过 libdrm/amdgpu 获取营销名称和家族 ID；对 CPU agent，从 /proc/cpuinfo 获取名称
4. **可见性更新**: 根据环境变量（ROCR_VISIBLE_DEVICES、HIP_VISIBLE_DEVICES 等）设置 agent 的运行时可见性
5. **HSA 映射**: `construct_agent_cache()` 通过 HSA API 枚举所有 HSA agent，并与已发现的 rocprofiler agent 建立映射
6. **查询阶段**: 通过 `rocprofiler_query_available_agents` API 向外部工具提供 agent 信息

<!-- verified: 2026-05-27 -->

## 已知限制与边界情况

- **ABI 版本检查**: `rocprofiler_query_available_agents` 会检查调用者使用的 agent 结构体大小是否与 SDK 兼容，不兼容时返回 `ROCPROFILER_STATUS_ERROR_INCOMPATIBLE_ABI`
- **Agent 类型判断**: 当 cpu_cores_count > 0 且 simd_count > 0 时（APU），将其标记为 GPU 类型
- **拓扑路径优先级**: 依次检查环境变量 ROCPROFILER_KFD_TOPOLOGY、AMD_KFD_TOPOLOGY、HSA_MODEL_TOPOLOGY，最后使用默认的 sysfs 路径
- **随机偏移量**: agent ID 使用随机偏移量以避免与其他实例冲突
- **内存管理**: mem_banks、caches、io_links 数组使用 `new[]` 分配，在 unique_ptr 的自定义删除器中释放
- **HSA agent 数量不匹配**: 如果 rocprofiler agent 数量与 HSA agent 数量不一致，会触发 FATAL 错误

<!-- verified: 2026-05-27 -->

## 与官方文档的关系

- API 参考: `source/docs/api-reference/rocprofiler-sdk_api_reference.rst`
- 相关概念文档: `source/docs/conceptual/comparing-with-legacy-tools.rst`
- 头文件 Doxygen 注释: `source/include/rocprofiler-sdk/agent.h` 中包含详细的 API 文档
