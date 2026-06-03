# 硬件计数器收集模块

## 概述

硬件计数器（Counters）模块负责收集 AMD GPU 的硬件性能计数器数据。该模块支持两种收集模式：调度级计数器收集（dispatch counting）和设备级计数器收集（device counting）。调度级计数器在每次内核调度时收集指定的硬件计数器，而设备级计数器在设备运行期间持续收集计数器数据。该模块通过 AQL（Architected Query Language）包注入机制与 GPU 硬件交互，支持派生指标的 AST（抽象语法树）评估，并提供了丰富的指标查询和配置 API。

## 关键文件

| 文件 | 路径 | 职责 |
|------|------|------|
| controller.hpp | source/lib/rocprofiler-sdk/counters/controller.hpp | CounterController 类定义，管理计数器配置和调度 |
| controller.cpp | source/lib/rocprofiler-sdk/counters/controller.cpp | CounterController 实现 |
| core.hpp | source/lib/rocprofiler-sdk/counters/core.hpp | 计数器核心类型和函数声明 |
| device_counting.hpp | source/lib/rocprofiler-sdk/counters/device_counting.hpp | 设备级计数器收集接口 |
| device_counting.cpp | source/lib/rocprofiler-sdk/counters/device_counting.cpp | 设备级计数器收集实现 |
| dispatch_handlers.hpp | source/lib/rocprofiler-sdk/counters/dispatch_handlers.hpp | 调度级计数器处理器 |
| dispatch_handlers.cpp | source/lib/rocprofiler-sdk/counters/dispatch_handlers.cpp | 调度级计数器处理器实现 |
| metrics.hpp | source/lib/rocprofiler-sdk/counters/metrics.hpp | 指标定义和管理 |
| evaluate_ast.hpp | source/lib/rocprofiler-sdk/counters/evaluate_ast.hpp | AST 评估引擎 |
| hsa_adapter.hpp | source/lib/rocprofiler-sdk/counters/hsa_adapter.hpp | HSA 适配器，处理硬件计数器数据 |
| counters.h | source/include/rocprofiler-sdk/counters.h | 公共 API 头文件 |
| dispatch_counting_service.h | source/include/rocprofiler-sdk/dispatch_counting_service.h | 调度计数服务 API |
| device_counting_service.h | source/include/rocprofiler-sdk/device_counting_service.h | 设备计数服务 API |

<!-- verified: 2026-05-27 -->

## 核心数据结构

### counter_config
- **定义位置**: `source/lib/rocprofiler-sdk/counters/controller.hpp:45`
- **职责**: 存储计数器收集配置，可在多个上下文间共享
- **关键字段**:
  - `agent` (const rocprofiler_agent_t*): 目标 agent
  - `metrics` (vector<Metric>): 用户请求的指标列表
  - `reqired_hw_counters` (set<Metric>): 所需的硬件计数器（派生指标会被分解）
  - `required_special_counters` (set<Metric>): 非硬件计数器（如 agent 属性）
  - `asts` (vector<EvaluateAST>): 评估 AST 列表
  - `id` (rocprofiler_counter_config_id_t): 配置 ID
  - `pkt_generator` (unique_ptr<CounterPacketConstruct>): AQL 包生成器
  - `packets` (Synchronized<vector<unique_ptr<AQLPacket>>>): AQL 包缓存

<!-- verified: 2026-05-27 -->

### counter_callback_info
- **定义位置**: `source/lib/rocprofiler-sdk/counters/core.hpp:46`
- **职责**: 存储调度级计数器收集的状态和回调信息
- **关键字段**:
  - `user_cb` (rocprofiler_dispatch_counting_service_cb_t): 用户回调函数
  - `callback_args` (void*): 用户回调数据
  - `context` (rocprofiler_context_id_t): 关联的上下文
  - `internal_context` (const context::context*): 内部上下文指针
  - `queue_id` (hsa::ClientID): HSA 队列客户端 ID
  - `buffer` (optional<rocprofiler_buffer_id_t>): 可选的缓冲区
  - `record_callback` (rocprofiler_dispatch_counting_record_cb_t): 记录回调
  - `packet_return_map` (Synchronized<unordered_map>): AQL 包返回映射

<!-- verified: 2026-05-27 -->

### agent_callback_data
- **定义位置**: `source/lib/rocprofiler-sdk/counters/device_counting.hpp:44`
- **职责**: 存储设备级计数器收集的状态
- **关键字段**:
  - `context_idx` (uint64_t): 上下文 ID
  - `queue` (hsa_queue_t*): HSA 队列
  - `packet` (shared_ptr<CounterAQLPacket>): AQL 计数器包
  - `completion` (hsa_signal_t): 三态信号（1: 允许开始, 0: 进行中, -1: 完成）
  - `start_signal` (hsa_signal_t): 启动信号
  - `user_data` / `callback_data` (rocprofiler_user_data_t): 用户数据
  - `profile` (shared_ptr<counter_config>): 计数器配置
  - `agent_id` (rocprofiler_agent_id_t): agent ID
  - `cb` (rocprofiler_device_counting_service_cb_t): 用户回调
  - `buffer` (rocprofiler_buffer_id_t): 缓冲区 ID

<!-- verified: 2026-05-27 -->

### CounterController
- **定义位置**: `source/lib/rocprofiler-sdk/counters/controller.hpp:66`
- **职责**: 全局计数器控制器，管理所有计数器配置
- **方法**:
  - `add_profile`: 添加计数器配置到全局缓存
  - `destroy_profile`: 从全局缓存移除配置
  - `configure_dispatch`: 配置调度级计数器收集
  - `configure_agent_collection`: 配置设备级计数器收集
  - `get_profile_cfg`: 获取指定 ID 的配置

<!-- verified: 2026-05-27 -->

## 关键函数

### 公共 API

- **`rocprofiler_create_counter_config`** (counters.h)
  - 职责: 创建计数器配置，指定 agent 和要收集的指标

- **`rocprofiler_destroy_counter_config`** (counters.h)
  - 职责: 销毁计数器配置

- **`rocprofiler_iterate_agent_supported_counters`** (counters.h)
  - 职责: 迭代指定 agent 支持的所有计数器

- **`rocprofiler_iterate_counter_dimensions`** (counters.h)
  - 职责: 迭代指定计数器的维度信息

- **`rocprofiler_query_counter_info`** (counters.h)
  - 职责: 查询计数器的详细信息

<!-- verified: 2026-05-27 -->

### 内部函数

#### 调度级计数器

- **`configure_buffered_dispatch`** (`core.hpp:82`)
  - 签名: `rocprofiler_status_t configure_buffered_dispatch(rocprofiler_context_id_t context_id, rocprofiler_buffer_id_t buffer, rocprofiler_dispatch_counting_service_cb_t callback, void* callback_args)`
  - 职责: 配置基于缓冲区的调度计数器收集

- **`configure_callback_dispatch`** (`core.hpp:88`)
  - 签名: `rocprofiler_status_t configure_callback_dispatch(rocprofiler_context_id_t context_id, rocprofiler_dispatch_counting_service_cb_t callback, void* callback_data_args, rocprofiler_dispatch_counting_record_cb_t record_callback, void* record_callback_args)`
  - 职责: 配置基于回调的调度计数器收集

- **`start_context` / `stop_context`** (`core.hpp:102/105`)
  - 职责: 启动/停止调度级计数器收集上下文

#### 设备级计数器

- **`configure_agent_collection`** (`core.hpp:95`)
  - 签名: `rocprofiler_status_t configure_agent_collection(rocprofiler_context_id_t context_id, rocprofiler_buffer_id_t buffer_id, rocprofiler_agent_id_t agent_id, rocprofiler_device_counting_service_cb_t cb, void* user_data)`
  - 职责: 配置设备级计数器收集

- **`start_agent_ctx` / `stop_agent_ctx`** (`device_counting.hpp:106/113`)
  - 职责: 启动/停止设备级计数器收集
  - 实现: 发送 AQL 启动/停止包到 GPU 队列

- **`read_agent_ctx`** (`device_counting.hpp:120`)
  - 签名: `rocprofiler_status_t read_agent_ctx(const context::context* ctx, rocprofiler_user_data_t user_data, rocprofiler_counter_flag_t flags, std::vector<rocprofiler_counter_record_t>* out_counters)`
  - 职责: 读取设备级计数器数据
  - 支持同步和异步模式

- **`device_counting_service_finalize`** (`device_counting.hpp:93`)
  - 职责: 终结所有设备级计数器服务

- **`device_counting_service_hsa_registration`** (`device_counting.hpp:99`)
  - 职责: 在 HSA 初始化后启动之前配置的上下文

<!-- verified: 2026-05-27 -->

## 调用关系

### 上游（谁调用了本模块）
- **context 模块**: 在 `start_context`/`stop_context` 中调用 `counters::start_context`/`counters::stop_context` 和 `counters::start_agent_ctx`/`counters::stop_agent_ctx`
- **registration 模块**: 调用 `device_counting_service_finalize` 和 `device_counting_service_hsa_registration`
- **工具库**: 通过公共 API 创建计数器配置和配置收集服务

### 下游（本模块调用了谁）
- **hsa 模块**: 使用 AQL 包注入机制与 GPU 交互
  - `hsa::AQLPacket`: AQL 包抽象
  - `hsa::CounterAQLPacket`: 计数器专用 AQL 包
  - `hsa::ClientID`: 队列拦截客户端 ID
- **hsa/agent_cache**: 获取 agent 的硬件信息
- **aql 模块**: 使用 `CounterPacketConstruct` 生成 AQL 包
- **context/domain 模块**: 管理追踪域配置
- **ioctl 模块**: 通过 ioctl 接口与 KFD 驱动交互
- **metrics 模块**: 指标定义和查询
- **evaluate_ast 模块**: 评估派生指标的 AST

<!-- verified: 2026-05-27 -->

## 数据流

### 调度级计数器收集流程
1. **配置**: 工具创建 `counter_config` 并调用 `configure_dispatch`
2. **队列拦截**: 注册 HSA 队列拦截回调
3. **包注入**: 在内核调度前注入计数器启动 AQL 包
4. **内核执行**: GPU 执行内核并收集计数器数据
5. **包注入**: 在内核调度后注入计数器停止 AQL 包
6. **数据读取**: 从 GPU 内存读取计数器数据
7. **AST 评估**: 使用 AST 将原始硬件计数器转换为派生指标
8. **回调/缓冲**: 将结果通过回调或缓冲区传递给工具

### 设备级计数器收集流程
1. **配置**: 工具调用 `configure_agent_collection` 指定 agent 和指标
2. **启动**: `start_agent_ctx` 发送 AQL 启动包到 GPU 队列
3. **收集**: GPU 持续收集计数器数据
4. **读取**: `read_agent_ctx` 从 GPU 内存读取计数器数据
5. **停止**: `stop_agent_ctx` 发送 AQL 停止包到 GPU 队列

### 指标查询流程
1. **枚举 agent**: 通过 `iterate_agent_supported_counters` 获取 agent 支持的计数器
2. **查询维度**: 通过 `iterate_counter_dimensions` 获取计数器的维度信息
3. **创建配置**: 通过 `create_counter_config` 创建包含所需指标的配置
4. **配置收集**: 将配置关联到上下文

<!-- verified: 2026-05-27 -->

## 已知限制与边界情况

- **上下文互斥**: 同一上下文不能同时使用调度级和设备级计数器收集
- **设备级收集状态机**: 使用三态信号（1/0/-1）管理收集状态，防止并发操作
- **AQL 包缓存**: 使用包缓存减少频繁的内存分配
- **异步读取**: 设备级计数器支持异步读取模式，但不支持重叠的异步读取
- **HSA 初始化依赖**: 设备级计数器需要在 HSA 初始化后才能启动
- **硬件限制**: 某些计数器可能受固件限制，需要特殊处理
- **AST 评估**: 派生指标通过 AST 评估，评估失败时返回错误
- **计数器 ID 解码**: 硬件计数器 ID 包含编码信息（如 XCC、SE、实例），需要正确解码
- **ioctl 依赖**: 部分功能通过 ioctl 与 KFD 驱动交互，需要适当的权限

<!-- verified: 2026-05-27 -->

## 与官方文档的关系

- API 参考: [rocprofiler-sdk API reference](../../../../projects/rocprofiler-sdk/source/docs/api-reference/rocprofiler-sdk_api_reference.rst)
- 计数器收集服务: [counter collection services](../../../../projects/rocprofiler-sdk/source/docs/api-reference/counter_collection_services.rst)
- 头文件 Doxygen 注释:
  - [counters.h](../../../../projects/rocprofiler-sdk/source/include/rocprofiler-sdk/counters.h)
  - [dispatch_counting_service.h](../../../../projects/rocprofiler-sdk/source/include/rocprofiler-sdk/dispatch_counting_service.h)
  - [device_counting_service.h](../../../../projects/rocprofiler-sdk/source/include/rocprofiler-sdk/device_counting_service.h)
