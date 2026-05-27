# KFD 内核驱动拦截器

## 概述

KFD（Kernel Fusion Driver）内核驱动拦截器是 rocprofiler-sdk 中负责监控 AMD GPU 内核驱动事件的模块。与 HIP 和 HSA 拦截器通过函数表替换实现 API 拦截不同，KFD 拦截器通过 Linux ioctl 接口打开 `/dev/kfd` 设备文件，注册感兴趣的内核事件（如页面迁移、页面错误、队列驱逐/恢复、GPU 解映射），然后在后台线程中使用 `poll()` 系统调用异步读取事件数据。该模块还实现了事件配对逻辑，将瞬时事件（如 page_migrate_start/end）合并为范围记录（range record）。该模块位于 `source/lib/rocprofiler-sdk/kfd/` 目录。

## 关键文件

| 文件 | 路径 | 职责 |
|------|------|------|
| kfd.hpp | `source/lib/rocprofiler-sdk/kfd/kfd.hpp` | 公共头文件，声明 `init()`、`finalize()`、`name_by_id()`、`get_ids()` 接口 |
| kfd.cpp | `source/lib/rocprofiler-sdk/kfd/kfd.cpp` | 核心实现，包含事件解析、KFD ioctl 交互、后台轮询线程、事件配对逻辑 |
| kfd.def.cpp | `source/lib/rocprofiler-sdk/kfd/kfd.def.cpp` | 宏定义文件，通过 `SPECIALIZE_KFD_EVENT_INFO`、`SPECIALIZE_KFD_KIND_INFO` 等宏为每个 KFD 事件类型生成模板特化 |
| defines.hpp | `source/lib/rocprofiler-sdk/kfd/defines.hpp` | 定义 `SPECIALIZE_KFD_EVENT_INFO`、`SPECIALIZE_KFD_KIND_INFO`、`SPECIALIZE_KFD_IOC_IOCTL` 等核心宏及 `ASSERT_SAME_AND_COPY` 辅助宏 |
| utils.hpp | `source/lib/rocprofiler-sdk/kfd/utils.hpp` | 工具头文件，定义 `kfd_event_record` 联合体、`kfd_event_info`/`kfd_kind_info`/`kfd_operation_info` 模板、位掩码计算函数 |

## 核心数据结构

- **`kfd_event_record`**：KFD 事件记录联合体，包含 `kind`（追踪类型）、`operation`（操作类型）和一个匿名联合体 `data`，可存储以下五种事件记录之一：
  - `page_migrate_event`（`rocprofiler_buffer_tracing_kfd_event_page_migrate_record_t`）
  - `page_fault_event`（`rocprofiler_buffer_tracing_kfd_event_page_fault_record_t`）
  - `queue_event`（`rocprofiler_buffer_tracing_kfd_event_queue_record_t`）
  - `unmap_event`（`rocprofiler_buffer_tracing_kfd_event_unmap_from_gpu_record_t`）
  - `dropped_event`（`rocprofiler_buffer_tracing_kfd_event_dropped_events_record_t`）
  - 以及三种范围记录：`page_migrate_record`、`page_fault_record`、`queue_record` <!-- verified: 2026-05-27 -->

- **`rocprofiler_kfd_event_id` 枚举**：定义 KFD 事件类型枚举，包括：
  - `KFD_EVENT_PAGE_MIGRATE_START/END`
  - `KFD_EVENT_PAGE_FAULT_START/END`
  - `KFD_EVENT_QUEUE_EVICTION/RESTORE`
  - `KFD_EVENT_UNMAP_FROM_GPU`
  - `KFD_EVENT_DROPPED_EVENT` <!-- verified: 2026-05-27 -->

- **`kfd_event_info<KFD_EVENT_XXX>`**：模板特化的事件信息结构体（由 `defines.hpp` 中的宏生成），包含：
  - `kind`：事件 ID
  - `buffer_kind`：缓冲追踪类型
  - `kfd_id`：KFD SMI 事件 ID
  - `kfd_bitmask`：KFD 事件位掩码
  - `format_str`：`sscanf` 解析格式字符串 <!-- verified: 2026-05-27 -->

- **`poll_kfd_t`**：KFD 轮询管理结构体，包含：
  - `kfd_fd`：KFD 设备文件描述符
  - `thread_notify`：用于通知后台线程退出的管道
  - `bg_thread`：后台轮询线程
  - 构造函数中打开 KFD 设备、为每个 GPU 获取事件 fd、写入事件掩码、启动后台线程 <!-- verified: 2026-05-27 -->

- **`config`**：全局配置单例，管理 `poll_kfd_t` 的生命周期，支持 fork 后重置。 <!-- verified: 2026-05-27 -->

- **`agent_id_map_t`（`std::unordered_map<uint64_t, rocprofiler_agent_id_t>`）**：KFD GPU 节点 ID 到 `rocprofiler_agent_id_t` 的映射表。

- **`kfd_kind_info<KIND>` / `kfd_operation_info<KIND, OP>`**：追踪类型和操作类型的信息模板。

## 关键函数

- **`init()`**：KFD 模块初始化函数。职责：检查 KFD 版本（要求 >= 1.11），检查是否有活跃的 KFD 追踪上下文，创建 `config` 单例启动后台轮询线程。注册 `pthread_atfork` 处理函数以支持 fork。返回 `ROCPROFILER_STATUS_ERROR_INCOMPATIBLE_KERNEL` 如果 KFD 版本过低。调用时机：上下文激活时。 <!-- verified: 2026-05-27 -->

- **`finalize()`**：KFD 模块终止函数。职责：通过 `config::reset()` 销毁轮询管理器，停止后台线程。 <!-- verified: 2026-05-27 -->

- **`poll_events(small_vector<pollfd>)`**：后台轮询线程主函数。职责：使用 `poll()` 系统调用等待 KFD 事件 fd 上的数据，读取事件字符串并调用 `handle_reporting` 处理。线程名为 `bg:poll-kfd`。 <!-- verified: 2026-05-27 -->

- **`handle_reporting(std::string_view event_data)`**：事件处理入口。职责：解析事件 ID -> 调用 `parse_event` 解析事件数据 -> 获取匹配的追踪上下文 -> 对每个上下文调用 `check_paired_events` 和 `emplace_event_buffer_record`。 <!-- verified: 2026-05-27 -->

- **`parse_event<KFD_EVENT_XXX>(const agent_id_map_t&, std::string_view)`**：模板特化的事件解析函数。职责：使用 `sscanf` 按照 `kfd_event_info` 中定义的格式字符串解析原始事件文本，填充 `kfd_event_record`。每种事件类型（page_migrate_start/end、page_fault_start/end、queue_eviction/restore、unmap、dropped）都有独立的特化实现。 <!-- verified: 2026-05-27 -->

- **`check_paired_events(const context_t*, const kfd_event_record&)`**：事件配对逻辑。职责：将瞬时事件（start/end）配对为范围记录：
  - page_migrate_start + page_migrate_end -> page_migrate_record
  - page_fault_start + page_fault_end -> page_fault_record
  - queue_eviction + queue_restore -> queue_record
  - 使用 `thread_local` 的 `unordered_set` 缓存未配对的 start 事件
  - 配对成功后写入缓冲区并从缓存中移除 <!-- verified: 2026-05-27 -->

- **`emplace_event_buffer_record(const context_t*, const kfd_event_record&)`**：将事件记录写入追踪缓冲区。职责：根据事件 kind 分发到对应的缓冲区。 <!-- verified: 2026-05-27 -->

- **`kfd_readlines(std::string_view, handler)`**：按行分割事件数据字符串。KFD 可能在一次 read 中返回多个事件（以换行符分隔），此函数逐行调用处理函数。

- **`name_by_id(uint32_t kind, uint32_t id)` / `get_ids(uint32_t kind)`**：根据追踪类型和操作 ID 查询名称/获取 ID 列表。

## 调用关系

- **上游**：
  - `context` 模块：在上下文激活时调用 `kfd::init()` 初始化 KFD 监控
  - `context` 模块：在上下文销毁/finalize 时调用 `kfd::finalize()` 停止监控

- **下游**：
  - Linux 内核 KFD 驱动：通过 `ioctl(AMDKFD_IOC_GET_VERSION)` 获取版本，通过 `ioctl(AMDKFD_IOC_SMI_EVENTS)` 获取每个 GPU 的事件 fd
  - `agent` 模块：通过 `agent::get_agents()` 获取 GPU agent 列表，构建节点 ID 到 agent ID 的映射
  - `buffer` 模块：通过 `buffer::get_buffer()` 获取追踪缓冲区，调用 `buffer->emplace()` 写入记录
  - `context` 模块：通过 `context::get_active_contexts()` 获取活跃的追踪上下文
  - `internal_threading` 模块：通过 `notify_pre/post_internal_thread_create()` 管理内部线程生命周期

## 数据流

1. **初始化阶段**：`init()` 被调用 -> 检查 KFD 版本 >= 1.11 -> 检查是否有活跃的 KFD 追踪上下文 -> 创建 `config` 单例 -> `poll_kfd_t` 构造：打开 `/dev/kfd` -> 为每个 GPU 通过 `AMDKFD_IOC_SMI_EVENTS` 获取事件 fd -> 写入事件掩码启用所需事件 -> 启动后台轮询线程
2. **事件轮询**：后台线程 `poll_events` 使用 `poll()` 等待事件 -> 事件到达时 `read()` 读取原始字符串 -> `kfd_readlines` 按行分割 -> 对每行调用 `handle_reporting`
3. **事件解析与分发**：`handle_reporting` -> `to_rocprofiler_kfd_event_id` 从字符串解析 KFD 事件 ID -> `parse_event` 模板特化解析具体事件数据 -> `get_contexts` 获取匹配的追踪上下文 -> 对每个上下文：
   - 调用 `check_paired_events` 尝试配对（start 事件缓存，end 事件匹配后写入范围记录）
   - 调用 `emplace_event_buffer_record` 写入瞬时事件记录
4. **线程退出**：`finalize()` -> `config::reset()` -> `poll_kfd_t` 析构 -> 向 `thread_notify` fd 写入 "E" -> 后台线程检测到退出信号 -> 关闭所有 fd 并退出

## 已知限制与边界情况

- **KFD 版本要求**：要求 KFD ioctl 版本 >= 1.10（`kfd_ioctl_version >= 1010`），运行时检查要求 >= 1.11。版本过低时返回 `ROCPROFILER_STATUS_ERROR_INCOMPATIBLE_KERNEL`。 <!-- verified: 2026-05-27 -->

- **事件解析健壮性**：`parse_event` 使用 `sscanf` 解析，通过检查返回的 scan count 验证解析完整性。page_migrate_end 事件兼容旧版 KFD（可能缺少 error_code 字段，scan_count == 8 时默认 error_code = 0）。 <!-- verified: 2026-05-27 -->

- **事件配对的线程安全性**：配对使用的 `unordered_set` 是 `thread_local` 的，这意味着每个线程独立维护配对状态。如果 start 和 end 事件在不同线程到达，将无法配对。

- **fork 支持**：通过 `pthread_atfork` 注册处理函数，在子进程中重置 `config` 单例并重新初始化，避免子进程共享父进程的线程和 fd。 <!-- verified: 2026-05-27 -->

- **事件丢失**：KFD 驱动可能因缓冲区溢出而丢失事件，此时会报告 `KFD_EVENT_DROPPED_EVENT` 类型的事件，包含丢失事件的计数。

- **页面大小转换**：KFD 报告的地址和大小以页（4KB）为单位，解析时通过 `page_to_bytes()`（左移 12 位）转换为字节。 <!-- verified: 2026-05-27 -->

- **队列恢复重调度**：`ROCPROFILER_KFD_EVENT_QUEUE_RESTORE_RESCHEDULED` 是一个瞬时事件，不参与配对逻辑，直接通过 `emplace_event_buffer_record` 写入缓冲区。 <!-- verified: 2026-05-27 -->

- **未映射事件**：`KFD_EVENT_UNMAP_FROM_GPU` 和 `KFD_EVENT_DROPPED_EVENT` 是瞬时事件，不参与 start/end 配对。

- **上下文过滤**：仅当上下文启用了 `ROCPROFILER_BUFFER_TRACING_KFD_EVENT_*` 或 `ROCPROFILER_BUFFER_TRACING_KFD_*`（范围记录）域时，才会被包含在事件分发中。

- **KFD 设备路径**：硬编码为 `/dev/kfd`。

## 与官方文档的关系

- API 参考 - 缓冲追踪服务：`source/docs/api-reference/rocprofiler-sdk_api/modules/buffer_tracing.rst`
- API 参考 - 上下文管理：`source/docs/api-reference/rocprofiler-sdk_api/modules/context_management.rst`
