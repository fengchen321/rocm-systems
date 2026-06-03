# HIP 运行时拦截器

## 概述

HIP 运行时拦截器是 rocprofiler-sdk 的核心模块之一，负责拦截和追踪 AMD HIP 运行时 API 调用。该模块通过函数表（dispatch table）替换机制，将 HIP 运行时和编译器 API 的函数指针替换为带有追踪逻辑的包装函数（functor），从而在不修改用户代码的情况下实现 API 调用的回调追踪和缓冲追踪。该模块位于 `source/lib/rocprofiler-sdk/hip/` 目录，是连接用户 HIP 应用与 rocprofiler-sdk 追踪基础设施的桥梁。

## 关键文件

| 文件 | 路径 | 职责 |
|------|------|------|
| hip.hpp | `source/lib/rocprofiler-sdk/hip/hip.hpp` | 核心头文件，定义 `HipApiTable`、`hip_api_info`、`hip_api_impl` 等模板结构体及 `copy_table`/`update_table` 等公共接口声明 |
| hip.cpp | `source/lib/rocprofiler-sdk/hip/hip.cpp` | 核心实现，包含 functor 模板（API 包装逻辑）、copy_table/update_table 实现、名称/ID 查询、参数迭代等 |
| hip.def.cpp | `source/lib/rocprofiler-sdk/hip/hip.def.cpp` | 宏定义文件，通过 `HIP_API_INFO_DEFINITION_V` 等宏为每个 HIP API 生成模板特化 |
| defines.hpp | `source/lib/rocprofiler-sdk/hip/defines.hpp` | 定义 `HIP_API_INFO_DEFINITION_0`、`HIP_API_INFO_DEFINITION_V`、`HIP_API_TABLE_LOOKUP_DEFINITION` 等核心宏 |
| stream.hpp | `source/lib/rocprofiler-sdk/hip/stream.hpp` | HIP 流（stream）追踪子模块头文件 |
| stream.cpp | `source/lib/rocprofiler-sdk/hip/stream.cpp` | HIP 流追踪实现，拦截 `hipStreamCreate`/`hipStreamDestroy`/`hipStreamSet` 等操作 |
| utils.hpp | `source/lib/rocprofiler-sdk/hip/utils.hpp` | 工具函数，提供参数字符串化（stringize）功能 |
| abi.cpp | `source/lib/rocprofiler-sdk/hip/abi.cpp` | ABI 兼容性检查，通过 `ROCP_SDK_ENFORCE_ABI` 宏确保函数表中函数指针的顺序不被重排 |
| details/format.hpp | `source/lib/rocprofiler-sdk/hip/details/format.hpp` | 格式化辅助，用于 HIP API 参数的格式化输出 |
| details/ostream.hpp | `source/lib/rocprofiler-sdk/hip/details/ostream.hpp` | 流输出辅助，定义 HIP 数据类型的 ostream 操作符 |

## 核心数据结构

- **`HipApiTable`**：聚合 HIP 编译器和运行时两个子表的顶层结构体，包含 `compiler`（`HipCompilerDispatchTable*`）和 `runtime`（`HipDispatchTable*`）两个指针。 <!-- verified: 2026-05-27 -->

- **`hip_api_info<TableIdx, OpIdx>`**：模板特化的 API 信息结构体（由 `hip.def.cpp` 中的宏生成），每个 HIP API 函数对应一个特化实例。关键字段/方法：
  - `table_idx` / `operation_idx`：表索引和操作索引
  - `name`：API 函数名字符串
  - `callback_domain_idx` / `buffered_domain_idx` / `buffered_ext_domain_idx`：追踪域索引
  - `get_table()` / `get_table_func()`：获取函数表和函数指针
  - `get_functor()`：返回包装后的函数指针
  - `as_arg_addr()` / `as_arg_list()`：参数地址和字符串化列表 <!-- verified: 2026-05-27 -->

- **`hip_api_impl<TableIdx, OpIdx>`**：API 包装实现基类，提供核心模板方法：
  - `set_data_args()`：将实际参数写入追踪数据结构
  - `exec()`：调用原始函数指针
  - `functor()`：完整的包装函数，包含进入/退出回调、时间戳记录、缓冲区写入 <!-- verified: 2026-05-27 -->

- **`hip_domain_info<TableIdx>`**：域信息模板，定义每个表（Runtime/Compiler）的回调/缓冲域索引、参数类型等。

- **`hip_table_lookup<TableIdx>` / `hip_table_id_lookup<Tp>`**：表查找模板，实现表索引与表类型之间的双向映射。 <!-- verified: 2026-05-27 -->

- **`stream_map_t`**（`std::unordered_map<hipStream_t, rocprofiler_stream_id_t>`）：HIP 流到 rocprofiler 流 ID 的映射表，用于流追踪子模块。

## 关键函数

- **`hip_api_impl::functor(Args... args)`**：核心包装函数模板。职责：拦截 HIP API 调用，执行进入回调、记录开始时间戳、调用原始函数、记录结束时间戳、执行退出回调、写入缓冲记录。调用时机：用户调用任何被追踪的 HIP API 时。 <!-- verified: 2026-05-27 -->

- **`copy_table(TableT* _orig, uint64_t _tbl_instance)`**：从 HIP 运行时提供的原始函数表复制函数指针到内部保存表。职责：保存原始函数指针以便后续恢复。调用时机：HIP 函数表初始化时。 <!-- verified: 2026-05-27 -->

- **`update_table(TableT* _orig)`**：将原始函数表中的函数指针替换为包装函数。职责：根据已注册的上下文决定哪些 API 需要被拦截，然后替换对应的函数指针。调用时机：上下文配置变更时。 <!-- verified: 2026-05-27 -->

- **`should_wrap_functor()`**：判断指定操作是否需要被包装。职责：遍历所有已注册上下文，检查是否有回调或缓冲追踪器启用了该域和操作。 <!-- verified: 2026-05-27 -->

- **`name_by_id<TableIdx>(uint32_t id)` / `id_by_name<TableIdx>(const char* name)`**：API 名称与 ID 的双向查询，内部编译为 switch 语句。 <!-- verified: 2026-05-27 -->

- **`iterate_args<TableIdx>(...)`**：遍历指定 API 调用的参数，通过回调函数逐个报告参数信息。 <!-- verified: 2026-05-27 -->

- **`stream::create_write_functor()` / `create_destroy_functor()` / `create_read_functor()`**：流追踪专用包装函数，分别处理流创建、销毁和设置操作。 <!-- verified: 2026-05-27 -->

## 调用关系

- **上游**：
  - `registration` 模块：在工具库加载时调用 `copy_table` 保存原始函数表
  - `context` 模块：在上下文配置变更时调用 `update_table` 安装/卸载拦截器
  - HIP 运行时：用户代码调用 HIP API 时触发 functor

- **下游**：
  - `tracing` 模块：functor 调用 `tracing::populate_contexts()`、`tracing::execute_phase_enter_callbacks()`、`tracing::execute_phase_exit_callbacks()`、`tracing::execute_buffer_record_emplace()` 等
  - `context` 模块：调用 `context::pop_latest_correlation_id()` 管理关联 ID
  - `buffer` 模块：通过 `execute_buffer_record_emplace` 写入缓冲区
  - `common` 模块：使用 `common::timestamp_ns()` 获取时间戳

## 数据流

1. **初始化阶段**：HIP 运行时提供函数表 -> `copy_table` 保存原始函数指针到内部表 -> `update_table` 将需要追踪的函数指针替换为 functor
2. **运行时拦截**：用户调用 HIP API -> 进入 functor -> 检查是否有活跃的追踪上下文 -> 若无则直接调用原始函数 -> 若有则：
   - 获取线程 ID 和关联 ID
   - 填充回调上下文和缓冲上下文
   - 执行进入阶段回调（`execute_phase_enter_callbacks`）
   - 记录开始时间戳
   - 调用原始 HIP 函数
   - 记录结束时间戳
   - 执行退出阶段回调（`execute_phase_exit_callbacks`）
   - 写入缓冲记录（`execute_buffer_record_emplace`）
3. **参数追踪**：在回调中，通过 `set_data_args` 将实际参数转换并存储到 `rocprofiler_hip_api_args_t` 联合体中，支持通过 `iterate_args` 遍历参数

## 已知限制与边界情况

- **ABI 版本依赖**：HIP 函数表的布局由 HIP 运行时版本决定，`abi.cpp` 中通过 `ROCP_SDK_ENFORCE_ABI` 宏在编译期检查函数指针顺序。不同 HIP 版本（通过 `HIP_RUNTIME_API_TABLE_STEP_VERSION` 控制）支持不同数量的 API。若 HIP 运行时更新导致函数表布局变化，需要同步更新 ABI 检查。 <!-- verified: 2026-05-27 -->

- **终止状态处理**：当 `registration::get_fini_status() != 0` 时（即正在 finalize），functor 会跳过所有追踪逻辑直接调用原始函数，避免在清理阶段访问已释放的资源。 <!-- verified: 2026-05-27 -->

- **nullptr 函数指针**：`exec` 方法在函数指针为 nullptr 时返回默认错误值（如 `hipErrorUnknown`），并通过 `ROCP_ERROR` 记录日志。 <!-- verified: 2026-05-27 -->

- **dim3 类型转换**：HIP 的 `dim3` 类型在参数记录时会被转换为 `rocprofiler_dim3_t`，通过 `convert_arg_type` 辅助函数实现。 <!-- verified: 2026-05-27 -->

- **流追踪的特殊处理**：`hipStreamLegacy`（0x01）会被映射为 nullptr（null stream），`hipStreamPerThread`（0x02）使用线程局部存储的流 ID。流追踪仅支持回调模式，不支持缓冲模式。 <!-- verified: 2026-05-27 -->

- **多流参数限制**：流追踪要求每个 HIP API 最多只能有一个 `hipStream_t` 参数，有多个流参数的函数（如 `hipStreamCopyAttributes`）会被显式禁用。 <!-- verified: 2026-05-27 -->

- **关联 ID 引用计数**：使用 `ref_count = 2` 的引用计数机制管理关联 ID 的生命周期，在函数调用前后各递减一次。

## 与官方文档的关系

- API 参考 - 拦截表服务：[intercept table](../../../../projects/rocprofiler-sdk/source/docs/api-reference/rocprofiler-sdk_api/modules/intercept_table.rst)
- API 参考 - 回调追踪服务：[callback tracing](../../../../projects/rocprofiler-sdk/source/docs/api-reference/rocprofiler-sdk_api/modules/callback_tracing.rst)
- API 参考 - 缓冲追踪服务：[buffer tracing](../../../../projects/rocprofiler-sdk/source/docs/api-reference/rocprofiler-sdk_api/modules/buffer_tracing.rst)
- API 参考 - 外部关联 ID：[external correlation](../../../../projects/rocprofiler-sdk/source/docs/api-reference/rocprofiler-sdk_api/modules/external_correalation.rst)
- 上述 API 参考页是 Doxygen group 生成入口；直接阅读主要头文件更有信息量：[intercept_table.h](../../../../projects/rocprofiler-sdk/source/include/rocprofiler-sdk/intercept_table.h)、[callback_tracing.h](../../../../projects/rocprofiler-sdk/source/include/rocprofiler-sdk/callback_tracing.h)、[buffer_tracing.h](../../../../projects/rocprofiler-sdk/source/include/rocprofiler-sdk/buffer_tracing.h)、[external_correlation.h](../../../../projects/rocprofiler-sdk/source/include/rocprofiler-sdk/external_correlation.h)
