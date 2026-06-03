# 回调式追踪模块

## 概述

回调式追踪（Callback Tracing）模块提供了 rocprofiler-sdk 的同步追踪机制。与缓冲式追踪不同，回调式追踪在追踪事件发生时立即调用用户注册的回调函数，使工具能够在事件的进入（enter）和退出（exit）阶段执行自定义逻辑。该模块支持多种追踪域（如 HSA API、HIP API、marker、内核调度等），每个域可以独立配置操作过滤器。回调式追踪适用于需要实时响应追踪事件的场景，如自定义指标计算、条件断点、或与其他追踪系统的集成。

## 关键文件

| 文件 | 路径 | 职责 |
|------|------|------|
| callback_tracing.cpp | source/lib/rocprofiler-sdk/callback_tracing.cpp | 回调追踪服务的配置和查询实现 |
| callback_tracing.h | source/include/rocprofiler-sdk/callback_tracing.h | 公共 API 头文件，定义回调追踪的类型和接口 |

<!-- verified: 2026-05-27 -->

## 核心数据结构

### callback_tracing_service（定义在 context.hpp）
- **定义位置**: `source/lib/rocprofiler-sdk/context/context.hpp:54`
- **职责**: 存储上下文的回调追踪配置
- **关键字段**:
  - `domains` (domain_context<rocprofiler_callback_tracing_kind_t>): 追踪域配置，记录哪些域和操作被启用
  - `callback_data` (callback_array_t): 回调数据数组，每个追踪域对应一个回调函数和用户数据

### callback_data（callback_tracing_service 内部）
- **定义位置**: `source/lib/rocprofiler-sdk/context/context.hpp:56`
- **关键字段**:
  - `callback` (rocprofiler_callback_tracing_cb_t): 用户回调函数指针
  - `data` (void*): 用户数据指针

### rocprofiler_callback_tracing_kind_t（枚举）
- **定义位置**: `source/include/rocprofiler-sdk/callback_tracing.h`
- **取值**:
  - `ROCPROFILER_CALLBACK_TRACING_NONE`
  - `ROCPROFILER_CALLBACK_TRACING_HSA_CORE_API`
  - `ROCPROFILER_CALLBACK_TRACING_HSA_AMD_EXT_API`
  - `ROCPROFILER_CALLBACK_TRACING_HSA_IMAGE_EXT_API`
  - `ROCPROFILER_CALLBACK_TRACING_HSA_FINALIZE_EXT_API`
  - `ROCPROFILER_CALLBACK_TRACING_HIP_RUNTIME_API`
  - `ROCPROFILER_CALLBACK_TRACING_HIP_COMPILER_API`
  - `ROCPROFILER_CALLBACK_TRACING_MARKER_CORE_API`
  - `ROCPROFILER_CALLBACK_TRACING_MARKER_CONTROL_API`
  - `ROCPROFILER_CALLBACK_TRACING_MARKER_NAME_API`
  - `ROCPROFILER_CALLBACK_TRACING_CODE_OBJECT`
  - `ROCPROFILER_CALLBACK_TRACING_SCRATCH_MEMORY`
  - `ROCPROFILER_CALLBACK_TRACING_KERNEL_DISPATCH`
  - `ROCPROFILER_CALLBACK_TRACING_MEMORY_COPY`
  - `ROCPROFILER_CALLBACK_TRACING_MEMORY_ALLOCATION`
  - `ROCPROFILER_CALLBACK_TRACING_RCCL_API`
  - `ROCPROFILER_CALLBACK_TRACING_OMPT`
  - `ROCPROFILER_CALLBACK_TRACING_RUNTIME_INITIALIZATION`
  - `ROCPROFILER_CALLBACK_TRACING_ROCDECODE_API`
  - `ROCPROFILER_CALLBACK_TRACING_ROCJPEG_API`
  - `ROCPROFILER_CALLBACK_TRACING_HIP_STREAM`
  - `ROCPROFILER_CALLBACK_TRACING_MARKER_CORE_RANGE_API`

<!-- verified: 2026-05-27 -->

## 关键函数

### 公共 API

- **`rocprofiler_configure_callback_tracing_service`** (`callback_tracing.cpp:128`)
  - 签名: `rocprofiler_status_t rocprofiler_configure_callback_tracing_service(rocprofiler_context_id_t context_id, rocprofiler_callback_tracing_kind_t kind, const rocprofiler_tracing_operation_t* operations, size_t operations_count, rocprofiler_callback_tracing_cb_t callback, void* callback_args)`
  - 职责: 为指定上下文配置回调追踪服务
  - 参数:
    - `context_id`: 目标上下文 ID
    - `kind`: 追踪域类型
    - `operations`: 要追踪的操作数组（NULL 表示所有操作）
    - `operations_count`: 操作数组大小（0 表示所有操作）
    - `callback`: 回调函数
    - `callback_args`: 回调用户数据
  - 返回值: 成功或错误状态
  - 调用时机: 工具在配置阶段调用，必须在初始化完成之前

- **`rocprofiler_query_callback_tracing_kind_name`** (`callback_tracing.cpp:165`)
  - 签名: `rocprofiler_status_t rocprofiler_query_callback_tracing_kind_name(rocprofiler_callback_tracing_kind_t kind, const char** name, uint64_t* name_len)`
  - 职责: 查询追踪域的名称字符串
  - 实现: 使用编译期模板特化实现 O(1) 查找

- **`rocprofiler_query_callback_tracing_kind_operation_name`** (`callback_tracing.cpp:179`)
  - 签名: `rocprofiler_status_t rocprofiler_query_callback_tracing_kind_operation_name(rocprofiler_callback_tracing_kind_t kind, rocprofiler_tracing_operation_t operation, const char** name, uint64_t* name_len)`
  - 职责: 查询指定域中操作的名称字符串
  - 实现: 使用 switch-case 分发到各子系统的 `name_by_id` 函数

- **`rocprofiler_iterate_callback_tracing_kinds`** (`callback_tracing.cpp:320`)
  - 签名: `rocprofiler_status_t rocprofiler_iterate_callback_tracing_kinds(rocprofiler_callback_tracing_kind_cb_t callback, void* data)`
  - 职责: 迭代所有可用的回调追踪域

- **`rocprofiler_iterate_callback_tracing_kind_operations`** (`callback_tracing.cpp:334`)
  - 签名: `rocprofiler_status_t rocprofiler_iterate_callback_tracing_kind_operations(rocprofiler_callback_tracing_kind_t kind, rocprofiler_callback_tracing_kind_operation_cb_t callback, void* data)`
  - 职责: 迭代指定域中所有可用的操作

- **`rocprofiler_iterate_callback_tracing_kind_operation_args`** (`callback_tracing.cpp:463`)
  - 签名: `rocprofiler_status_t rocprofiler_iterate_callback_tracing_kind_operation_args(rocprofiler_callback_tracing_record_t record, rocprofiler_callback_tracing_operation_args_cb_t callback, int32_t max_deref, void* user_data)`
  - 职责: 迭代追踪记录中的操作参数
  - 注意: 在 ENTER 阶段使用 max_deref > 1 可能导致段错误

<!-- verified: 2026-05-27 -->

## 调用关系

### 上游（谁调用了本模块）
- **工具库**: 通过 `rocprofiler_configure_callback_tracing_service` 配置回调追踪
- **context 模块**: 在启动/停止上下文时检查回调追踪配置
- **各子系统**: 在 API 调用的 enter/exit 阶段触发回调

### 下游（本模块调用了谁）
- **context/domain 模块**: 调用 `add_domain` 和 `add_domain_op` 管理追踪域配置
- **各子系统的 name_by_id/get_ids 函数**: 用于查询操作名称和 ID
  - `hsa::name_by_id` / `hsa::get_ids`
  - `hip::name_by_id` / `hip::get_ids`
  - `marker::name_by_id` / `marker::get_ids`
  - `rccl::name_by_id` / `rccl::get_ids`
  - `code_object::name_by_id` / `code_object::get_ids`
  - `kernel_dispatch::name_by_id` / `kernel_dispatch::get_ids`
  - `hsa::async_copy::name_by_id` / `hsa::async_copy::get_ids`
  - `hsa::memory_allocation::name_by_id` / `hsa::memory_allocation::get_ids`
  - `hsa::scratch_memory::name_by_id` / `hsa::scratch_memory::get_ids`
  - `ompt::name_by_id` / `ompt::get_ids`
  - `runtime_init::name_by_id` / `runtime_init::get_ids`
  - `rocdecode::name_by_id` / `rocdecode::get_ids`
  - `rocjpeg::name_by_id` / `rocjpeg::get_ids`
  - `hip::stream::name_by_id` / `hip::stream::get_ids`

<!-- verified: 2026-05-27 -->

## 数据流

### 配置流程
1. **工具调用**: 工具调用 `rocprofiler_configure_callback_tracing_service` 指定要追踪的域和操作
2. **上下文查找**: 通过 `get_mutable_registered_context` 获取上下文指针
3. **服务创建**: 如果上下文尚未有回调追踪服务，创建新的 `callback_tracing_service`
4. **域注册**: 调用 `add_domain` 注册追踪域
5. **回调存储**: 将回调函数和用户数据存储在 `callback_data` 数组中
6. **操作注册**: 对每个指定的操作调用 `add_domain_op` 注册

### 回调触发流程
1. **API 拦截**: 子系统的 API 包装器在函数进入时被调用
2. **上下文检查**: 检查是否有活跃的上下文配置了该追踪域和操作
3. **ENTER 回调**: 调用用户的回调函数，phase = ROCPROFILER_CALLBACK_PHASE_ENTER
4. **原始函数调用**: 调用原始的 API 函数
5. **EXIT 回调**: 调用用户的回调函数，phase = ROCPROFILER_CALLBACK_PHASE_EXIT

### 名称查询流程
1. **域名称**: 使用编译期模板特化，通过 `index_sequence` 实现 O(1) 查找
2. **操作名称**: 使用 switch-case 分发到各子系统的 `name_by_id` 函数
3. **操作迭代**: 使用各子系统的 `get_ids` 函数获取所有可用操作 ID

<!-- verified: 2026-05-27 -->

## 已知限制与边界情况

- **配置锁定**: 回调追踪只能在初始化完成之前配置（`get_init_status() > -1` 时返回 ROCPROFILER_STATUS_ERROR_CONFIGURATION_LOCKED）
- **重复配置**: 同一上下文的同一追踪域不能重复配置（返回 ROCPROFILER_STATUS_ERROR_SERVICE_ALREADY_CONFIGURED）
- **不支持的域**: 某些追踪域可能被标记为不支持（返回 ROCPROFILER_STATUS_ERROR_NOT_IMPLEMENTED）
- **参数迭代限制**: 在 ENTER 阶段使用 max_deref > 1 可能导致段错误，因为参数指针在 enter 阶段可能尚未初始化
- **回调执行上下文**: 回调在 API 调用的线程上下文中同步执行，长时间运行的回调会影响应用性能
- **操作过滤**: 如果 operations 为 NULL 且 operations_count 为 0，则追踪该域的所有操作
- **类型安全**: 使用 `static_cast` 将 `record.payload` 转换为具体的记录类型，类型不匹配会导致未定义行为

<!-- verified: 2026-05-27 -->

## 与官方文档的关系

- API 参考: [rocprofiler-sdk API reference](../../../../projects/rocprofiler-sdk/source/docs/api-reference/rocprofiler-sdk_api_reference.rst)
- 回调服务: [callback services](../../../../projects/rocprofiler-sdk/source/docs/api-reference/callback_services.rst)
- 头文件 Doxygen 注释: [callback_tracing.h](../../../../projects/rocprofiler-sdk/source/include/rocprofiler-sdk/callback_tracing.h) 中包含详细的 API 文档
