# 分析上下文管理模块

## 概述

上下文（Context）模块是 rocprofiler-sdk 的核心管理模块，负责创建、配置、启动和停止分析上下文。每个上下文代表一个独立的分析会话，可以关联回调追踪服务、缓冲追踪服务、计数器收集服务、PC 采样服务和线程追踪服务等。上下文模块维护两个主要集合：已注册上下文（所有已创建的上下文）和活跃上下文（当前正在收集数据的上下文）。它使用无锁的原子操作实现高效的活跃上下文检查，确保在追踪路径上的最小性能开销。

## 关键文件

| 文件 | 路径 | 职责 |
|------|------|------|
| context.cpp | source/lib/rocprofiler-sdk/context/context.cpp | 上下文管理的核心实现 |
| context.hpp | source/lib/rocprofiler-sdk/context/context.hpp | 内部头文件，定义 context 结构体和所有服务结构体 |
| domain.cpp | source/lib/rocprofiler-sdk/context/domain.cpp | 追踪域（domain）管理实现 |
| domain.hpp | source/lib/rocprofiler-sdk/context/domain.hpp | 追踪域管理的内部头文件 |
| correlation_id.cpp | source/lib/rocprofiler-sdk/context/correlation_id.cpp | 关联 ID 管理实现 |
| correlation_id.hpp | source/lib/rocprofiler-sdk/context/correlation_id.hpp | 关联 ID 管理的内部头文件 |
| context.h | source/include/rocprofiler-sdk/context.h | 公共 API 头文件 |

<!-- verified: 2026-05-27 -->

## 核心数据结构

### context（内部结构）
- **定义位置**: `source/lib/rocprofiler-sdk/context/context.hpp:134`
- **职责**: 存储单个分析上下文的完整状态
- **关键字段**:
  - `size` (size_t): 结构体大小，用于版本兼容性检查
  - `context_idx` (uint64_t): 上下文 ID
  - `client_idx` (uint32_t): 关联的工具客户端 ID
  - `correlation_tracer` (correlation_tracing_service): 关联追踪服务
  - `callback_tracer` (unique_ptr<callback_tracing_service>): 回调追踪服务（可选）
  - `buffered_tracer` (unique_ptr<buffer_tracing_service>): 缓冲追踪服务（可选）
  - `dispatch_counter_collection` (unique_ptr<dispatch_counter_collection_service>): 调度计数器收集服务（可选）
  - `device_counter_collection` (unique_ptr<device_counting_service>): 设备计数器收集服务（可选）
  - `pc_sampler` (unique_ptr<pc_sampling_service>): PC 采样服务（可选）
  - `dispatch_thread_trace` (unique_ptr<thread_trace::DispatchThreadTracer>): 调度线程追踪（可选）
  - `device_thread_trace` (unique_ptr<thread_trace::DeviceThreadTracer>): 设备线程追踪（可选）
  - `dispatch_spm` (unique_ptr<spm_dispatch_counter_collection_service>): SPM 计数器收集（可选）

<!-- verified: 2026-05-27 -->

### callback_tracing_service
- **定义位置**: `source/lib/rocprofiler-sdk/context/context.hpp:54`
- **职责**: 管理回调式追踪的配置
- **关键字段**:
  - `domains` (domain_context<rocprofiler_callback_tracing_kind_t>): 追踪域配置
  - `callback_data` (callback_array_t): 每个域的回调函数和用户数据

### buffer_tracing_service
- **定义位置**: `source/lib/rocprofiler-sdk/context/context.hpp:69`
- **职责**: 管理缓冲式追踪的配置
- **关键字段**:
  - `domains` (domain_context<rocprofiler_buffer_tracing_kind_t>): 追踪域配置
  - `buffer_data` (buffer_array_t): 每个域关联的缓冲区 ID

### dispatch_counter_collection_service
- **定义位置**: `source/lib/rocprofiler-sdk/context/context.hpp:78`
- **职责**: 管理调度级别的计数器收集
- **关键字段**:
  - `callbacks` (vector<shared_ptr<counter_callback_info>>): 计数器收集实例列表
  - `enabled` (Synchronized<bool>): 启用状态标志

### device_counting_service
- **定义位置**: `source/lib/rocprofiler-sdk/context/context.hpp:103`
- **职责**: 管理设备级别的计数器收集
- **关键字段**:
  - `conf_agents` (unordered_set<uint64_t>): 已配置的 agent 集合
  - `agent_data` (vector<agent_callback_data>): agent 级别的收集数据
  - `status` (atomic<state>): 服务状态（DISABLED/LOCKED/ENABLED/EXIT）
  - `enabled` (Synchronized<bool>): 启用状态标志

### pc_sampling_service
- **定义位置**: `source/lib/rocprofiler-sdk/context/context.hpp:120`
- **职责**: 管理 PC 采样服务
- **关键字段**:
  - `agent_sessions` (unordered_map<rocprofiler_agent_id_t, shared_ptr<PCSAgentSession>>): 每个 agent 的采样会话
  - `enabled` (atomic<bool>): 启用状态标志

<!-- verified: 2026-05-27 -->

## 关键函数

### 公共 API

- **`rocprofiler_create_context`** (context.h)
  - 职责: 创建新的分析上下文并返回上下文 ID

- **`rocprofiler_start_context`** (context.h)
  - 职责: 启动指定的上下文，使其开始收集数据

- **`rocprofiler_stop_context`** (context.h)
  - 职责: 停止指定的上下文

<!-- verified: 2026-05-27 -->

### 内部函数

- **`allocate_context`** (`context.cpp:211`)
  - 签名: `std::optional<rocprofiler_context_id_t> allocate_context()`
  - 职责: 分配新的上下文结构，设置 context_idx 和 client_idx
  - 返回值: 新分配的上下文 ID

- **`start_context`** (`context.cpp:268`)
  - 签名: `rocprofiler_status_t start_context(rocprofiler_context_id_t context_id)`
  - 职责: 将上下文添加到活跃上下文数组，并启动所有关联的服务
  - 冲突检查: 检查是否与已活跃的上下文冲突（如两个调度计数器上下文）
  - 服务启动: 根据上下文配置启动对应的计数器/采样/追踪服务

- **`stop_context`** (`context.cpp:354`)
  - 签名: `rocprofiler_status_t stop_context(rocprofiler_context_id_t idx)`
  - 职责: 从活跃上下文数组中移除上下文，并停止所有关联的服务
  - 实现: 使用原子 compare_exchange_strong 将上下文指针设为 nullptr

- **`get_active_context`** (`context.cpp:175`)
  - 签名: `const context* get_active_context(rocprofiler_context_id_t id)`
  - 职责: 高效检查指定上下文是否处于活跃状态
  - 性能: 使用原子加载，无锁操作

- **`get_registered_context`** (`context.cpp:255`)
  - 签名: `const context* get_registered_context(rocprofiler_context_id_t id)`
  - 职责: 获取已注册上下文的指针

- **`get_mutable_registered_context`** (`context.cpp:242`)
  - 签名: `context* get_mutable_registered_context(rocprofiler_context_id_t id)`
  - 职责: 获取已注册上下文的可变指针（用于配置阶段）

- **`push_client` / `pop_client`** (`context.cpp:191/202`)
  - 职责: 设置/清除当前正在初始化的客户端 ID
  - 用途: 在客户端初始化期间，新创建的上下文会自动关联到该客户端

- **`get_client_contexts`** (`context.cpp:405`)
  - 签名: `context_id_array_t get_client_contexts(rocprofiler_client_id_t id)`
  - 职责: 获取指定客户端的所有上下文 ID

- **`stop_client_contexts`** (`context.cpp:422`)
  - 职责: 停止指定客户端的所有上下文

- **`deactivate_client_contexts`** (`context.cpp:441`)
  - 职责: 从活跃数组中移除指定客户端的所有上下文（不停止服务）

- **`deregister_client_contexts`** (`context.cpp:454`)
  - 职责: 注销指定客户端的所有上下文和关联的缓冲区

- **`validate_context`** (`context.cpp:262`)
  - 职责: 验证上下文配置的有效性

<!-- verified: 2026-05-27 -->

## 调用关系

### 上游（谁调用了本模块）
- **registration 模块**: 在客户端初始化期间调用 `push_client`/`pop_client`，在终结时调用 `stop_client_contexts`/`deactivate_client_contexts`/`deregister_client_contexts`
- **buffer 模块**: 在缓冲区分配时关联到上下文
- **buffer_tracing 模块**: 通过 `get_mutable_registered_context` 配置缓冲追踪服务
- **callback_tracing 模块**: 通过 `get_mutable_registered_context` 配置回调追踪服务
- **counters 模块**: 通过 `start_context`/`stop_context` 启动/停止计数器收集
- **pc_sampling 模块**: 通过 `start_context`/`stop_context` 启动/停止 PC 采样

### 下游（本模块调用了谁）
- **buffer 模块**: 在 `deregister_client_contexts` 中清理关联的缓冲区
- **counters 模块**: 调用 `start_context`/`stop_context`/`start_agent_ctx`/`stop_agent_ctx`
- **pc_sampling 模块**: 调用 `start_service`/`stop_service`
- **thread_trace 模块**: 调用 `start_context`/`stop_context`
- **spm 模块**: 调用 `start_context`/`stop_context`

<!-- verified: 2026-05-27 -->

## 数据流

### 上下文创建流程
1. **客户端设置**: `push_client(client_idx)` 设置当前客户端 ID
2. **分配**: `allocate_context()` 在已注册上下文数组中创建新条目
3. **配置**: 通过各服务的配置函数（如 `rocprofiler_configure_buffer_tracing_service`）设置上下文的服务
4. **客户端清除**: `pop_client(client_idx)` 清除当前客户端 ID

### 上下文启动流程
1. **查找**: 从已注册上下文中查找目标上下文
2. **冲突检查**: 检查是否与已活跃的上下文冲突
3. **原子插入**: 使用 compare_exchange_strong 将上下文指针插入活跃数组的空闲槽位
4. **服务启动**: 根据上下文配置启动对应的子服务（计数器、采样、追踪等）

### 上下文停止流程
1. **原子移除**: 使用 compare_exchange_strong 将活跃数组中的上下文指针设为 nullptr
2. **服务停止**: 调用各子服务的停止函数
3. **计数更新**: 减少活跃上下文计数

### 追踪检查流程（is_tracing）
1. **域检查**: 检查上下文是否配置了指定的追踪域
2. **操作检查**: 检查上下文是否配置了指定的追踪操作
3. **优化**: 使用模板特化和内联优化，确保追踪路径上的最小开销

<!-- verified: 2026-05-27 -->

## 已知限制与边界情况

- **上下文冲突**: 不能同时有两个活跃的调度计数器收集上下文（`dispatch_counter_collection` 互斥）
- **客户端隔离**: 每个上下文关联到一个客户端，客户端终结时其所有上下文会被停止和注销
- **随机偏移**: 上下文 ID 使用随机偏移量避免冲突
- **活跃上下文数组**: 使用 `stable_vector` 存储，支持并发读取和原子更新
- **空闲槽位复用**: 停止上下文后，其槽位可以被新启动的上下文复用
- **ABA 问题防护**: 使用原子指针和 compare_exchange_strong 防止并发启动/停止的竞态条件
- **线程安全的活跃上下文检查**: `get_active_contexts` 使用原子加载和计数检查，支持无锁读取
- **上下文数量限制**: 受 `stable_vector` 的 chunk_size 限制，但会自动扩展

<!-- verified: 2026-05-27 -->

## 与官方文档的关系

- API 参考: [rocprofiler-sdk API reference](../../../../projects/rocprofiler-sdk/source/docs/api-reference/rocprofiler-sdk_api_reference.rst)
- 回调服务: [callback services](../../../../projects/rocprofiler-sdk/source/docs/api-reference/callback_services.rst)
- 缓冲服务: [buffered services](../../../../projects/rocprofiler-sdk/source/docs/api-reference/buffered_services.rst)
- 计数器收集: [counter collection services](../../../../projects/rocprofiler-sdk/source/docs/api-reference/counter_collection_services.rst)
- PC 采样: [PC sampling](../../../../projects/rocprofiler-sdk/source/docs/api-reference/pc_sampling.rst)
- 线程追踪: [thread trace](../../../../projects/rocprofiler-sdk/source/docs/api-reference/thread_trace.rst)
- 头文件 Doxygen 注释: [context.h](../../../../projects/rocprofiler-sdk/source/include/rocprofiler-sdk/context.h) 中包含 Context 生命周期 API 文档
