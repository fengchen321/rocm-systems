# HSA 运行时拦截器

## 概述

HSA 运行时拦截器是 rocprofiler-sdk 中负责拦截和追踪 HSA（Heterogeneous System Architecture）运行时 API 的核心模块。HSA 是 AMD GPU 编程的底层运行时接口，该模块通过替换 `HsaApiTable` 中的函数指针来实现 API 拦截，覆盖 Core API、AMD 扩展 API、Finalizer 扩展 API、Image 扩展 API、Tools API 以及可选的 PC Sampling 扩展 API 六个子表。与 HIP 拦截器类似，该模块使用模板元编程和编译期序列化技术为每个 HSA API 生成类型安全的包装函数。该模块位于 `source/lib/rocprofiler-sdk/hsa/` 目录。

## 关键文件

| 文件 | 路径 | 职责 |
|------|------|------|
| hsa.hpp | `source/lib/rocprofiler-sdk/hsa/hsa.hpp` | 核心头文件，定义 `hsa_api_table_t`、`hsa_api_info`、`hsa_api_impl` 等模板及公共接口 |
| hsa.cpp | `source/lib/rocprofiler-sdk/hsa/hsa.cpp` | 核心实现，包含 functor 模板、copy_table/update_table/dlsym_table 实现、HSA 引用计数管理 |
| hsa.def.cpp | `source/lib/rocprofiler-sdk/hsa/hsa.def.cpp` | 宏定义文件，为每个 HSA API 生成模板特化 |
| defines.hpp | `source/lib/rocprofiler-sdk/hsa/defines.hpp` | 定义 `HSA_API_INFO_DEFINITION` 等核心宏 |
| queue.hpp | `source/lib/rocprofiler-sdk/hsa/queue.hpp` | HSA 队列包装类，拦截队列创建/销毁操作 |
| queue.cpp | `source/lib/rocprofiler-sdk/hsa/queue.cpp` | 队列拦截实现 |
| queue_controller.hpp | `source/lib/rocprofiler-sdk/hsa/queue_controller.hpp` | 队列控制器，管理所有被追踪的 HSA 队列 |
| queue_controller.cpp | `source/lib/rocprofiler-sdk/hsa/queue_controller.cpp` | 队列控制器实现 |
| agent_cache.hpp | `source/lib/rocprofiler-sdk/hsa/agent_cache.hpp` | Agent 缓存，存储每个 GPU agent 的 HSA 信息（池、最近 CPU 等） |
| agent_cache.cpp | `source/lib/rocprofiler-sdk/hsa/agent_cache.cpp` | Agent 缓存实现 |
| memory_allocation.hpp/cpp | `source/lib/rocprofiler-sdk/hsa/memory_allocation.{hpp,cpp}` | 内存分配追踪，拦截 HSA 内存分配/释放操作 |
| async_copy.hpp/cpp | `source/lib/rocprofiler-sdk/hsa/async_copy.{hpp,cpp}` | 异步拷贝追踪 |
| hsa_barrier.hpp/cpp | `source/lib/rocprofiler-sdk/hsa/hsa_barrier.{hpp,cpp}` | HSA 屏障（barrier）追踪 |
| signal.hpp | `source/lib/rocprofiler-sdk/hsa/signal.hpp` | HSA 信号追踪辅助 |
| scratch_memory.hpp/cpp | `source/lib/rocprofiler-sdk/hsa/scratch_memory.{hpp,cpp}` | Scratch 内存分配追踪，拦截 Tools API 中的 scratch 内存操作 |
| pc_sampling.hpp/cpp | `source/lib/rocprofiler-sdk/hsa/pc_sampling.{hpp,cpp}` | PC 采样扩展拦截 |
| aql_packet.hpp/cpp | `source/lib/rocprofiler-sdk/hsa/aql_packet.{hpp,cpp}` | AQL 包处理辅助 |
| profile_serializer.hpp/cpp | `source/lib/rocprofiler-sdk/hsa/profile_serializer.{hpp,cpp}` | 性能剖析数据序列化 |
| utils.hpp | `source/lib/rocprofiler-sdk/hsa/utils.hpp` | 工具函数 |
| abi.cpp | `source/lib/rocprofiler-sdk/hsa/abi.cpp` | ABI 兼容性检查 |

## 核心数据结构

- **`hsa_api_table_t`（`::HsaApiTable`）**：HSA 顶层 API 表，包含 `core_`、`amd_ext_`、`finalizer_ext_`、`image_ext_`、`tools_`、`pc_sampling_ext_` 等子表指针和版本信息。 <!-- verified: 2026-05-27 -->

- **`hsa_api_info<TableIdx, OpIdx>`**：模板特化的 API 信息结构体（由 `hsa.def.cpp` 中的宏生成），每个 HSA API 函数对应一个特化实例。关键成员与 HIP 的 `hip_api_info` 类似。 <!-- verified: 2026-05-27 -->

- **`hsa_api_impl<TableIdx, OpIdx>`**：API 包装实现，提供 `set_data_args`、`exec`、`functor` 三个核心模板方法，逻辑与 HIP 的 `hip_api_impl` 类似。 <!-- verified: 2026-05-27 -->

- **`hsa_domain_info<TableIdx>`**：域信息模板，定义每个子表（Core/AmdExt/FinalizeExt/ImageExt）的回调/缓冲域索引。

- **`hsa_table_lookup<Idx>` / `hsa_table_id_lookup<Tp>`**：表查找模板，实现表索引与表类型之间的双向映射。

- **`hsa_api_func<FuncT>`**：函数特征提取模板，从函数指针类型中提取返回类型和参数类型元组。

- **`tracing_table` / `internal_table`**：空标记类型，用于区分追踪用副本表和内部保存表。

- **`QueueController`**：队列控制器类，管理所有被追踪的 HSA 队列。关键方法：`init()`、`add_queue()`、`remove_queue()`。 <!-- verified: 2026-05-27 -->

- **`AgentCache`**：Agent 缓存类，存储 GPU agent 的 HSA 代理句柄、内存池信息、最近 CPU 代理等。 <!-- verified: 2026-05-27 -->

## 关键函数

- **`hsa_api_impl::functor(Args... args)`**：核心包装函数模板。职责：拦截 HSA API 调用，执行回调追踪和缓冲追踪。与 HIP functor 类似，但增加了对 HSA 引用计数的特殊处理和 `hsa_status_t` 返回值的错误检查。调用时机：用户调用任何被追踪的 HSA API 时。 <!-- verified: 2026-05-27 -->

- **`copy_table<TableT>(TableT* _orig, uint64_t _tbl_instance)`**：从 HSA 运行时提供的原始函数表复制函数指针。特殊处理 `hsa_init` 和 `hsa_shut_down` 以实现引用计数管理。 <!-- verified: 2026-05-27 -->

- **`update_table<TableT>(TableT* _orig, uint64_t _tbl_instance)`**：将原始函数表中的函数指针替换为包装函数。先复制到追踪表，再根据已注册上下文决定替换哪些函数指针。 <!-- verified: 2026-05-27 -->

- **`dlsym_table<TableT>(TableT* _orig)`**：通过 `dlsym(RTLD_DEFAULT, ...)` 动态查找 HSA API 符号并填充函数表。用于处理运行时才可用的扩展 API。 <!-- verified: 2026-05-27 -->

- **`hsa_init_refcnt_impl()` / `hsa_shut_down_refcnt_impl()`**：HSA 初始化/关闭的引用计数包装。`hsa_init` 每次调用增加引用计数，`hsa_shut_down` 每次调用减少引用计数，仅在最后一个引用关闭时真正调用 `hsa_shut_down_fn`。 <!-- verified: 2026-05-27 -->

- **`get_hsa_timestamp_period()`**：获取 HSA 时间戳周期（纳秒），通过查询 `HSA_SYSTEM_INFO_TIMESTAMP_FREQUENCY` 计算。

- **`get_hsa_status_string(hsa_status_t)`**：将 HSA 状态码转换为可读字符串。

- **`get_table()` / `get_core_table()` / `get_amd_ext_table()` 等**：获取各子表的访问函数，使用 `common::static_object` 管理表的生命周期。

- **`get_tracing_core_table()` / `get_tracing_amd_ext_table()` 等**：获取追踪用副本表。

## 调用关系

- **上游**：
  - `registration` 模块：在工具库加载时调用 `copy_table` 保存原始函数表
  - `context` 模块：在上下文配置变更时调用 `update_table` 安装/卸载拦截器
  - HSA 运行时：用户代码调用 HSA API 时触发 functor
  - `hip::stream` 模块：流追踪依赖 HSA 队列拦截（`hsa::enable_queue_intercept()`）

- **下游**：
  - `tracing` 模块：functor 调用 `tracing::populate_contexts()`、`tracing::execute_phase_enter_callbacks()`、`tracing::execute_phase_exit_callbacks()`、`tracing::execute_buffer_record_emplace()`
  - `context` 模块：调用 `context::pop_latest_correlation_id()`
  - `buffer` 模块：通过缓冲记录写入
  - `thread_trace` 模块：在最后一个 HSA 引用关闭时调用 `thread_trace::flush_and_stop()`
  - `pc_sampling` 模块：PC 采样扩展表的 copy_table 委托给 `pc_sampling::copy_table()`
  - `scratch_memory` 模块：Tools API 表的 copy_table/update_table 委托给 `scratch_memory` 模块

## 数据流

1. **初始化阶段**：HSA 运行时提供 `HsaApiTable` -> `copy_table` 保存原始函数指针到 `internal_table` -> 同时复制到 `tracing_table` -> `update_table` 将需要追踪的函数指针替换为 functor -> 特殊处理 `hsa_init`/`hsa_shut_down` 实现引用计数
2. **运行时拦截**：用户调用 HSA API -> 进入 functor -> 检查是否有活跃的追踪上下文 -> 若无则直接调用原始函数 -> 若有则执行与 HIP 类似的回调/缓冲追踪流程
3. **函数指针回退**：对于某些扩展 API，如果原始表中函数指针为空，`dlsym_table` 会尝试通过 `dlsym` 查找符号并填充

## 已知限制与边界情况

- **HSA 引用计数**：rocprofiler-sdk 对 `hsa_init`/`hsa_shut_down` 进行引用计数管理。当引用计数降为 0 时，会先调用 `thread_trace::flush_and_stop()` 停止线程追踪，再调用真正的 `hsa_shut_down`。这确保了在 ROCR Runtime 卸载前正确清理资源。 <!-- verified: 2026-05-27 -->

- **表版本兼容性**：每个子表（Core/AMDExt 等）都有独立的版本号（major_version + step_version），`copy_table` 通过 `_orig->version.minor_id`（即表大小）判断函数指针是否存在于当前版本。 <!-- verified: 2026-05-27 -->

- **PC Sampling 扩展**：PC Sampling 扩展表（`PcSamplingExtTable`）仅在 `ROCPROFILER_SDK_HSA_PC_SAMPLING > 0` 时编译，其 copy_table 委托给专门的 `pc_sampling` 模块。 <!-- verified: 2026-05-27 -->

- **scratch_memory 特殊处理**：Tools API 表（`hsa_amd_tool_table_t`）的 copy_table/update_table 不走通用路径，而是委托给 `scratch_memory` 模块的专用实现。 <!-- verified: 2026-05-27 -->

- **终止状态处理**：与 HIP 类似，当 finalize 状态激活时，functor 跳过追踪逻辑直接调用原始函数。

- **关联 ID 为 nullptr**：在 finalize 期间，`tracing::correlation_service::construct()` 可能返回 nullptr，此时 functor 会跳过追踪直接调用原始函数。 <!-- verified: 2026-05-27 -->

- **HSA 时间戳精度**：`get_hsa_timestamp_period()` 通过 `1000000000 / sysclock_hz` 计算，可能存在整数除法精度损失。

## 与官方文档的关系

- API 参考 - 拦截表服务：[intercept table](../../../../projects/rocprofiler-sdk/source/docs/api-reference/rocprofiler-sdk_api/modules/intercept_table.rst)
- API 参考 - 回调追踪服务：[callback tracing](../../../../projects/rocprofiler-sdk/source/docs/api-reference/rocprofiler-sdk_api/modules/callback_tracing.rst)
- API 参考 - 缓冲追踪服务：[buffer tracing](../../../../projects/rocprofiler-sdk/source/docs/api-reference/rocprofiler-sdk_api/modules/buffer_tracing.rst)
- API 参考 - PC 采样服务：[PC sampling service](../../../../projects/rocprofiler-sdk/source/docs/api-reference/rocprofiler-sdk_api/modules/pc_sampling_service.rst)
- 上述 API 参考页是 Doxygen group 生成入口；直接阅读主要头文件更有信息量：[intercept_table.h](../../../../projects/rocprofiler-sdk/source/include/rocprofiler-sdk/intercept_table.h)、[callback_tracing.h](../../../../projects/rocprofiler-sdk/source/include/rocprofiler-sdk/callback_tracing.h)、[buffer_tracing.h](../../../../projects/rocprofiler-sdk/source/include/rocprofiler-sdk/buffer_tracing.h)、[pc_sampling.h](../../../../projects/rocprofiler-sdk/source/include/rocprofiler-sdk/pc_sampling.h)
