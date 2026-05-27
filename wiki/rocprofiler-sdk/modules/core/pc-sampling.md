# PC 采样模块

## 概述

PC 采样（PC Sampling）模块负责收集 AMD GPU 的程序计数器（Program Counter）采样数据。该模块通过硬件采样机制，在 GPU 执行内核时定期捕获当前的 PC 值，从而提供内核执行的热点分析能力。PC 采样支持两种硬件方法：基于间隔的采样和基于事件的采样，采样单位可以是时钟周期或指令数。该模块集成了 HSA PC 采样扩展（hsa_ven_amd_pc_sampling）和 KFD ioctl 接口，并提供了采样数据的解析、关联 ID 管理和缓冲区管理功能。

## 关键文件

| 文件 | 路径 | 职责 |
|------|------|------|
| service.hpp | source/lib/rocprofiler-sdk/pc_sampling/service.hpp | PC 采样服务的核心接口声明 |
| service.cpp | source/lib/rocprofiler-sdk/pc_sampling/service.cpp | PC 采样服务的核心实现 |
| controller.hpp | source/lib/rocprofiler-sdk/pc_sampling/controller.hpp | PC 采样控制器 |
| controller.cpp | source/lib/rocprofiler-sdk/pc_sampling/controller.cpp | PC 采样控制器实现 |
| types.hpp | source/lib/rocprofiler-sdk/pc_sampling/types.hpp | PC 采样类型定义 |
| core.cpp | source/lib/rocprofiler-sdk/pc_sampling/core.cpp | PC 采样核心功能 |
| cid_manager.hpp | source/lib/rocprofiler-sdk/pc_sampling/cid_manager.hpp | 关联 ID 管理器 |
| cid_manager.cpp | source/lib/rocprofiler-sdk/pc_sampling/cid_manager.cpp | 关联 ID 管理器实现 |
| code_object.hpp | source/lib/rocprofiler-sdk/pc_sampling/code_object.hpp | 代码对象管理 |
| code_object.cpp | source/lib/rocprofiler-sdk/pc_sampling/code_object.cpp | 代码对象管理实现 |
| defines.hpp | source/lib/rocprofiler-sdk/pc_sampling/defines.hpp | PC 采样编译时定义 |
| pc_sampling.h | source/include/rocprofiler-sdk/pc_sampling.h | 公共 API 头文件 |

<!-- verified: 2026-05-27 -->

## 核心数据结构

### PCSAgentSession
- **定义位置**: `source/lib/rocprofiler-sdk/pc_sampling/types.hpp:48`
- **职责**: 存储单个 agent 的 PC 采样会话配置和状态
- **关键字段**:
  - `agent` (const rocprofiler_agent_t*): 目标 agent
  - `method` (rocprofiler_pc_sampling_method_t): 采样方法（NONE 或 INTERVAL）
  - `unit` (rocprofiler_pc_sampling_unit_t): 采样单位（NONE 或 CYCLES 或 INSTRUCTIONS）
  - `interval` (uint64_t): 采样间隔
  - `buffer_id` (rocprofiler_buffer_id_t): 关联的缓冲区 ID
  - `hsa_agent` (optional<hsa_agent_t>): 对应的 HSA agent
  - `hsa_pc_sampling` (hsa_ven_amd_pcs_t): HSA PC 采样扩展句柄
  - `intercept_cb_id` (hsa::ClientID): HSA 队列拦截回调 ID
  - `ioctl_pcs_id` (uint32_t): KFD ioctl PC 采样 ID
  - `parser` (unique_ptr<PCSamplingParserContext>): 采样数据解析器上下文
  - `cid_manager` (unique_ptr<PCSCIDManager>): 关联 ID 管理器
  - `context_id` (rocprofiler_context_id_t): 关联的上下文 ID
  - `client_idx` (uint32_t): 关联的客户端 ID

<!-- verified: 2026-05-27 -->

### pc_sampling_service（定义在 context.hpp）
- **定义位置**: `source/lib/rocprofiler-sdk/context/context.hpp:120`
- **职责**: 存储上下文的 PC 采样服务配置
- **关键字段**:
  - `agent_sessions` (unordered_map<rocprofiler_agent_id_t, shared_ptr<PCSAgentSession>>): 每个 agent 的采样会话映射
  - `enabled` (atomic<bool>): 服务启用状态

### global_pc_sampling_sessions_map_t
- **定义**: `std::unordered_map<rocprofiler_agent_id_t, std::shared_ptr<PCSAgentSession>>`
- **职责**: 全局 PC 采样会话映射，用于 O(1) 的 agent 所有权检查

### rocprofiler_pc_sampling_method_t（枚举）
- **定义位置**: `source/include/rocprofiler-sdk/pc_sampling.h`
- **取值**:
  - `ROCPROFILER_PC_SAMPLING_METHOD_NONE`: 无效方法
  - `ROCPROFILER_PC_SAMPLING_METHOD_INTERVAL`: 基于间隔的采样

### rocprofiler_pc_sampling_unit_t（枚举）
- **定义位置**: `source/include/rocprofiler-sdk/pc_sampling.h`
- **取值**:
  - `ROCPROFILER_PC_SAMPLING_UNIT_NONE`: 无效单位
  - `ROCPROFILER_PC_SAMPLING_UNIT_CYCLES`: 基于时钟周期
  - `ROCPROFILER_PC_SAMPLING_UNIT_INSTRUCTIONS`: 基于指令数

<!-- verified: 2026-05-27 -->

## 关键函数

### 公共 API

- **`rocprofiler_configure_pc_sampling_service`** (pc_sampling.h)
  - 职责: 为指定上下文配置 PC 采样服务
  - 参数: context_id, agent, method, unit, interval, buffer_id
  - 调用时机: 工具在配置阶段调用

<!-- verified: 2026-05-27 -->

### 内部函数

- **`configure_pc_sampling_service`** (`service.hpp:66`)
  - 签名: `rocprofiler_status_t configure_pc_sampling_service(context::context* ctx, const rocprofiler_agent_t* agent, rocprofiler_pc_sampling_method_t method, rocprofiler_pc_sampling_unit_t unit, uint64_t interval, rocprofiler_buffer_id_t buffer_id)`
  - 职责: 内部配置 PC 采样服务
  - 实现: 创建 PCSAgentSession，配置 HSA PC 采样扩展或 KFD ioctl

- **`start_service`** (`service.hpp:57`)
  - 签名: `rocprofiler_status_t start_service(const context::context* ctx)`
  - 职责: 启动指定上下文的 PC 采样服务
  - 调用时机: 上下文启动时由 context 模块调用

- **`stop_service`** (`service.hpp:60`)
  - 签名: `rocprofiler_status_t stop_service(const context::context* ctx)`
  - 职责: 停止指定上下文的 PC 采样服务
  - 调用时机: 上下文停止时由 context 模块调用

- **`post_hsa_init_start_active_service`** (`service.hpp:63`)
  - 签名: `void post_hsa_init_start_active_service()`
  - 职责: 在 HSA 初始化后启动之前配置的活跃服务
  - 调用时机: HSA API 表注册时

- **`is_pc_sample_service_configured`** (`service.hpp:74`)
  - 签名: `bool is_pc_sample_service_configured(rocprofiler_agent_id_t agent_id)`
  - 职责: 检查指定 agent 是否已配置 PC 采样

- **`get_agent_session`** (`service.hpp:77`)
  - 签名: `PCSAgentSession* get_agent_session(rocprofiler_agent_id_t agent_id)`
  - 职责: 获取指定 agent 的 PC 采样会话

- **`flush_internal_agent_buffers`** (`service.hpp:80`)
  - 签名: `rocprofiler_status_t flush_internal_agent_buffers(rocprofiler_buffer_id_t buffer_id)`
  - 职责: 刷新指定缓冲区关联的所有 agent 的内部 PC 采样缓冲区
  - 调用时机: 手动刷新缓冲区时

- **`service_sync`** (`service.hpp:83`)
  - 签名: `void service_sync(rocprofiler_client_id_t client_id)`
  - 职责: 同步指定客户端的所有 PC 采样操作
  - 调用时机: 客户端终结时

- **`service_fini`** (`service.hpp:86`)
  - 签名: `void service_fini()`
  - 职责: 终结所有 PC 采样服务
  - 调用时机: SDK 终结时

- **`get_global_pc_sampling_sessions`** (`service.hpp:53`)
  - 签名: `common::Synchronized<global_pc_sampling_sessions_map_t>& get_global_pc_sampling_sessions()`
  - 职责: 获取全局 PC 采样会话映射的线程安全引用

<!-- verified: 2026-05-27 -->

## 调用关系

### 上游（谁调用了本模块）
- **context 模块**: 在 `start_context`/`stop_context` 中调用 `pc_sampling::start_service`/`pc_sampling::stop_service`
- **registration 模块**:
  - 在 HSA 初始化时调用 `pc_sampling::code_object::initialize` 和 `pc_sampling::post_hsa_init_start_active_service`
  - 在终结时调用 `pc_sampling::code_object::finalize`、`pc_sampling::service_sync` 和 `pc_sampling::service_fini`
- **buffer 模块**: 在 `rocprofiler_flush_buffer` 中调用 `pc_sampling::flush_internal_agent_buffers`
- **工具库**: 通过 `rocprofiler_configure_pc_sampling_service` 配置 PC 采样

### 下游（本模块调用了谁）
- **hsa 模块**: 使用 HSA PC 采样扩展（hsa_ven_amd_pc_sampling）
- **hsa/queue 模块**: 使用队列拦截机制
- **context 模块**: 获取上下文信息
- **buffer 模块**: 写入采样数据到缓冲区
- **pc_sampling/parser**: 解析原始采样数据
- **pc_sampling/cid_manager**: 管理关联 ID
- **pc_sampling/code_object**: 管理代码对象信息
- **ioctl 模块**: 通过 KFD ioctl 与驱动交互

<!-- verified: 2026-05-27 -->

## 数据流

### PC 采样配置流程
1. **工具调用**: 工具调用 `rocprofiler_configure_pc_sampling_service` 指定 agent、方法、单位和间隔
2. **会话创建**: 创建 `PCSAgentSession` 对象
3. **HSA 初始化**: 如果 HSA 已初始化，配置 HSA PC 采样扩展；否则延迟到 HSA 初始化时
4. **全局注册**: 将会话注册到全局会话映射

### PC 采样启动流程
1. **上下文启动**: context 模块调用 `start_service`
2. **启用设置**: 设置 `pc_sampling_service::enabled` 为 true
3. **硬件启动**: 通过 HSA 扩展或 KFD ioctl 启动硬件采样

### PC 采样数据流
1. **硬件采样**: GPU 硬件按配置的间隔捕获 PC 值
2. **数据传输**: 采样数据从 GPU 内存传输到主机内存
3. **数据解析**: `PCSamplingParserContext` 将原始数据解析为结构化的采样记录
4. **关联 ID**: `PCSCIDManager` 管理采样记录与内核调度的关联
5. **缓冲写入**: 解析后的记录写入关联的缓冲区
6. **回调通知**: 如果配置了缓冲区回调，通知工具数据可用

### PC 采样停止流程
1. **上下文停止**: context 模块调用 `stop_service`
2. **硬件停止**: 通过 HSA 扩展或 KFD ioctl 停止硬件采样
3. **数据刷新**: 刷新剩余的采样数据
4. **启用清除**: 设置 `pc_sampling_service::enabled` 为 false

<!-- verified: 2026-05-27 -->

## 已知限制与边界情况

- **编译时条件**: PC 采样功能通过 `ROCPROFILER_SDK_HSA_PC_SAMPLING` 宏控制，需要 HSA PC 采样扩展支持
- **HSA 初始化依赖**: PC 采样需要在 HSA 初始化后才能启动
- **Agent 唯一性**: 每个 agent 同时只能有一个活跃的 PC 采样会话
- **采样方法限制**: 当前仅支持 `ROCPROFILER_PC_SAMPLING_METHOD_INTERVAL` 方法
- **采样单位**: 支持时钟周期和指令数两种单位
- **缓冲区关联**: PC 采样会话必须关联到一个缓冲区
- **关联 ID 管理**: `PCSCIDManager` 负责管理采样记录与内核调度的关联，确保正确的因果关系
- **代码对象追踪**: 需要代码对象信息来将 PC 值映射到源代码位置
- **KFD ioctl**: 部分功能通过 KFD ioctl 实现，需要适当的权限
- **全局会话映射**: 使用 `Synchronized` 包装确保线程安全访问
- **终结顺序**: `service_fini` 必须在 `code_object::finalize` 之后调用，但必须在 `queue_controller_fini` 之后

<!-- verified: 2026-05-27 -->

## 与官方文档的关系

- API 参考: `source/docs/api-reference/rocprofiler-sdk_api_reference.rst`
- PC 采样: `source/docs/api-reference/pc_sampling.rst`
- CDNA3/CDNA4 PC 采样: `source/docs/how-to/cdna3-cdna4-pc-sampling.rst`
- 使用 PC 采样: `source/docs/how-to/using-pc-sampling.rst`
- 头文件 Doxygen 注释: `source/include/rocprofiler-sdk/pc_sampling.h` 中包含详细的 API 文档
