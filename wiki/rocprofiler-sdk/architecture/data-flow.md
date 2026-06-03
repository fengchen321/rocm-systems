# ROCprofiler-SDK 数据流

本文档描述 ROCprofiler-SDK 中数据从用户 API 调用到最终输出的完整路径，包括 Buffer Tracing 和 Callback Tracing 两条主要数据流路径。

> 验证记录：2026-06-03 使用 codegraph 查询 `rocprofiler_create_context`、`rocprofiler_create_buffer`、`rocprofiler_configure_buffer_tracing_service`、`rocprofiler_configure_callback_tracing_service`、`rocprofiler_get_timestamp`，并核对 `buffer.cpp`、`buffer_tracing.cpp`、`callback_tracing.cpp`、`context.cpp`、`rocprofiler.cpp`。 <!-- verified: 2026-06-03 -->

---

## 整体数据流概览

```
+=====================================================================+
|                        用户应用程序                                   |
|  (调用 HIP/HSA/Marker 等运行时 API)                                  |
+=====================================================================+
         |                                    |
         v                                    v
+-------------------+              +-------------------+
| 运行时 API 拦截   |              | 运行时 API 拦截   |
| (HSA wrapper)     |              | (HIP wrapper)     |
+-------------------+              +-------------------+
         |                                    |
         v                                    v
+=====================================================================+
|                    rocprofiler-sdk 内部处理                          |
|  +-------------------+  +-------------------+  +------------------+ |
|  | 关联 ID 生成      |  | 时间戳采集        |  | Context 检查     | |
|  | (correlation_id)  |  | (timestamp_ns)    |  | (active contexts)| |
|  +-------------------+  +-------------------+  +------------------+ |
+=====================================================================+
         |                                    |
         v                                    v
+-------------------+              +-------------------+
| Callback Tracing  |              | Buffer Tracing    |
| (同步回调)        |              | (异步缓冲)        |
+-------------------+              +-------------------+
         |                                    |
         v                                    v
+-------------------+              +-------------------+
| 工具回调函数      |              | 内部缓冲区        |
| (tool_callback)   |              | (record_header_   |
|                   |              |  buffer)          |
+-------------------+              +-------------------+
                                              |
                                              v
                                   +-------------------+
                                   | 缓冲区刷新        |
                                   | (flush/callback)  |
                                   +-------------------+
                                              |
                                              v
                                   +-------------------+
                                   | 工具数据处理      |
                                   | (tool_process)    |
                                   +-------------------+
                                              |
                                              v
                                   +-------------------+
                                   | 数据输出          |
                                   | (CSV/Perfetto/    |
                                   |  OTF2/JSON)       |
                                   +-------------------+
```

---

## 完整生命周期数据流

### 阶段 1: 工具注册与初始化

```
工具代码                          rocprofiler-sdk
   |                                    |
   |  rocprofiler_configure()           |
   |----------------------------------->|
   |  返回 rocprofiler_tool_configure_  |
   |  result_t {initialize, finalize}   |
   |<-----------------------------------|
   |                                    |
   |  [SDK 调用 initialize 回调]        |
   |<-----------------------------------|
   |                                    |
   |  rocprofiler_create_context()      |
   |----------------------------------->|
   |  返回 context_id                   |
   |<-----------------------------------|
   |                                    |
   |  rocprofiler_create_buffer()       |
   |----------------------------------->|
   |  返回 buffer_id                    |
   |<-----------------------------------|
   |                                    |
   |  rocprofiler_configure_buffer_     |
   |  tracing_service() 或              |
   |  rocprofiler_configure_callback_   |
   |  tracing_service()                 |
   |----------------------------------->|
   |  服务配置绑定到 Context            |
   |<-----------------------------------|
   |                                    |
   |  rocprofiler_start_context()       |
   |----------------------------------->|
   |  Context 激活，开始数据采集        |
   |<-----------------------------------|
```

**关键实现** (`source/lib/rocprofiler-sdk/registration.cpp`):
- `rocprofiler_configure` 是弱符号，工具实现此函数
- SDK 在初始化时查找并调用此函数
- 初始化完成后，配置被锁定（`get_init_status() > -1`）

**关键实现** (`source/lib/rocprofiler-sdk/context.cpp`):
- `rocprofiler_create_context` 调用 `rocprofiler::context::allocate_context()`
- `rocprofiler_start_context` 调用 `rocprofiler::context::start_context()`

---

### 阶段 2: 运行时 API 拦截

```
应用程序代码                     运行时库              rocprofiler-sdk
   |                              |                        |
   |  hipLaunchKernel()           |                        |
   |----------------------------->|                        |
   |                              |  HIP 拦截表            |
   |                              |  (tool_wrapper)        |
   |                              |----------------------->|
   |                              |                        |
   |                              |  rocprofiler 内部处理  |
   |                              |  - 生成关联 ID         |
   |                              |  - 采集时间戳          |
   |                              |  - 检查活跃 Context    |
   |                              |                        |
   |                              |  [Callback Tracing]    |
   |                              |  ENTER 阶段回调        |
   |                              |<-----------------------|
   |                              |                        |
   |                              |  调用真实实现           |
   |                              |  (real_hipLaunchKernel)|
   |                              |                        |
   |                              |  [Callback Tracing]    |
   |                              |  EXIT 阶段回调         |
   |                              |<-----------------------|
   |                              |                        |
   |                              |  [Buffer Tracing]      |
   |                              |  写入缓冲区记录        |
   |                              |                        |
   |  返回                         |                        |
   |<-----------------------------|                        |
```

**拦截机制** (`source/lib/rocprofiler-sdk/intercept_table.cpp`):
- 运行时库加载时，SDK 通过拦截表 hook API 函数
- 调用链: `应用 -> 运行时函数 -> 工具包装器 -> rocprofiler 包装器 -> 真实实现`

---

## Buffer Tracing 数据流

### 详细流程图

```
+------------------------------------------------------------------+
|                    Buffer Tracing 数据流                          |
+------------------------------------------------------------------+
|                                                                  |
|  1. API 调用触发                                                  |
|     应用程序调用运行时 API (如 hsa_init, hipLaunchKernel)         |
|                           |                                      |
|                           v                                      |
|  2. 拦截器捕获                                                   |
|     rocprofiler 包装器捕获 API 调用                               |
|                           |                                      |
|                           v                                      |
|  3. 关联 ID 生成                                                  |
|     生成内部关联 ID (internal correlation ID)                     |
|     查询外部关联 ID (external correlation ID)                     |
|     记录祖先关联 ID (ancestor correlation ID)                     |
|                           |                                      |
|                           v                                      |
|  4. 时间戳采集                                                    |
|     调用 common::timestamp_ns() 采集开始时间戳                    |
|                           |                                      |
|                           v                                      |
|  5. Context 检查                                                  |
|     遍历活跃 Context 数组                                         |
|     检查 Context 是否配置了对应的 Buffer Tracing 域               |
|                           |                                      |
|                           v                                      |
|  6. 记录构建                                                      |
|     构建 rocprofiler_record_header_t                              |
|     - category: ROCPROFILER_BUFFER_CATEGORY_TRACING              |
|     - kind: 对应的 buffer_tracing_kind                           |
|     - payload: 指向具体记录结构                                   |
|                           |                                      |
|                           v                                      |
|  7. 缓冲区写入                                                    |
|     调用 buffer::instance::emplace() 写入缓冲区                   |
|     使用双缓冲机制，原子操作保证线程安全                           |
|                           |                                      |
|                           v                                      |
|  8. 水位线检查                                                    |
|     检查缓冲区是否达到水位线 (watermark)                           |
|                           |                                      |
|              +-------------------+                               |
|              | 达到水位线?       |                               |
|              +-------------------+                               |
|              | 是              | 否                              |
|              v                 v                                  |
|  9a. 触发刷新               9b. 继续累积                         |
|      调用 flush_buffer()     等待下次触发或手动刷新               |
|              |                                                    |
|              v                                                    |
|  10. 后台线程处理                                                 |
|      内部线程池处理缓冲区数据                                     |
|      调用工具注册的 callback 函数                                  |
|              |                                                    |
|              v                                                    |
|  11. 工具数据处理                                                 |
|      工具解析 rocprofiler_record_header_t                         |
|      根据 category + kind 转换为具体记录类型                      |
|      输出到文件 (CSV/Perfetto/OTF2/JSON)                         |
+------------------------------------------------------------------+
```

### 关键实现文件

| 步骤 | 实现文件 | 关键函数 |
|------|----------|----------|
| 服务配置 | `buffer_tracing.cpp` | `rocprofiler_configure_buffer_tracing_service()` |
| 记录构建 | `context/context.hpp` | `buffer_tracing_service` 结构 |
| 缓冲区管理 | `buffer.hpp`, `buffer.cpp` | `buffer::instance::emplace()` |
| 关联 ID | `context/correlation_id.cpp` | 关联 ID 生成和管理 |
| 时间戳 | `rocprofiler.cpp` / `common` | `rocprofiler_get_timestamp()` / `common::timestamp_ns()` |

### 数据记录结构

```c
// 通用记录头
typedef struct rocprofiler_record_header_t {
    union {
        struct {
            uint32_t category;  // ROCPROFILER_BUFFER_CATEGORY_TRACING
            uint32_t kind;      // ROCPROFILER_BUFFER_TRACING_HSA_CORE_API 等
        };
        uint64_t hash;
    };
    void* payload;  // 指向具体记录结构
} rocprofiler_record_header_t;

// 示例: HSA API 记录
typedef struct rocprofiler_buffer_tracing_hsa_api_record_t {
    uint64_t                          size;
    rocprofiler_buffer_tracing_kind_t kind;
    rocprofiler_tracing_operation_t   operation;
    rocprofiler_correlation_id_t      correlation_id;
    rocprofiler_timestamp_t           start_timestamp;
    rocprofiler_timestamp_t           end_timestamp;
    rocprofiler_thread_id_t           thread_id;
} rocprofiler_buffer_tracing_hsa_api_record_t;
```

---

## Callback Tracing 数据流

### 详细流程图

```
+------------------------------------------------------------------+
|                   Callback Tracing 数据流                         |
+------------------------------------------------------------------+
|                                                                  |
|  1. API 调用触发                                                  |
|     应用程序调用运行时 API                                        |
|                           |                                      |
|                           v                                      |
|  2. 拦截器捕获                                                   |
|     rocprofiler 包装器捕获 API 调用                               |
|                           |                                      |
|                           v                                      |
|  3. 关联 ID 生成                                                  |
|     生成内部关联 ID                                               |
|     查询外部关联 ID                                               |
|                           |                                      |
|                           v                                      |
|  4. 时间戳采集                                                    |
|     采集开始时间戳                                                |
|                           |                                      |
|                           v                                      |
|  5. Context 检查                                                  |
|     遍历活跃 Context 数组                                         |
|     检查 Context 是否配置了对应的 Callback Tracing 域             |
|                           |                                      |
|                           v                                      |
|  6. ENTER 阶段回调                                                |
|     构建 rocprofiler_callback_tracing_record_t                    |
|     - phase = ROCPROFILER_CALLBACK_PHASE_ENTER                   |
|     在调用线程上同步调用工具回调函数                               |
|                           |                                      |
|                           v                                      |
|  7. 执行真实 API                                                  |
|     调用运行时 API 的真实实现                                     |
|                           |                                      |
|                           v                                      |
|  8. 时间戳采集                                                    |
|     采集结束时间戳                                                |
|                           |                                      |
|                           v                                      |
|  9. EXIT 阶段回调                                                 |
|     构建 rocprofiler_callback_tracing_record_t                    |
|     - phase = ROCPROFILER_CALLBACK_PHASE_EXIT                    |
|     - payload 包含返回值                                          |
|     在调用线程上同步调用工具回调函数                               |
|                           |                                      |
|                           v                                      |
|  10. 工具处理                                                     |
|      工具在回调中处理数据                                         |
|      可以实时修改行为或记录数据                                   |
+------------------------------------------------------------------+
```

### 关键实现文件

| 步骤 | 实现文件 | 关键函数 |
|------|----------|----------|
| 服务配置 | `callback_tracing.cpp` | `rocprofiler_configure_callback_tracing_service()` |
| 回调结构 | `context/context.hpp` | `callback_tracing_service` 结构 |
| 回调执行 | 各拦截器实现 | 如 `hsa/hsa.cpp`, `hip/hip.cpp` |

### 数据记录结构

```c
// 回调追踪记录
typedef struct rocprofiler_callback_tracing_record_t {
    rocprofiler_context_id_t            context_id;
    rocprofiler_thread_id_t             thread_id;
    rocprofiler_correlation_id_t        correlation_id;
    rocprofiler_callback_tracing_kind_t kind;
    rocprofiler_tracing_operation_t     operation;
    rocprofiler_callback_phase_t        phase;     // ENTER 或 EXIT
    void*                               payload;   // 指向域特定数据
} rocprofiler_callback_tracing_record_t;

// 示例: HSA API 回调数据
typedef struct rocprofiler_callback_tracing_hsa_api_data_t {
    uint64_t                     size;
    rocprofiler_hsa_api_args_t   args;      // API 参数
    rocprofiler_hsa_api_retval_t retval;    // 返回值 (仅 EXIT 阶段)
} rocprofiler_callback_tracing_hsa_api_data_t;
```

---

## 两种追踪模式对比

```
                    Callback Tracing              Buffer Tracing
                    ================              ==============

触发时机:           API 调用时立即触发            数据写入缓冲区，批量处理

线程模型:           在调用线程上同步执行          后台线程异步处理

数据内容:           ENTER: 参数                   完整记录 (开始/结束时间戳)
                    EXIT: 参数 + 返回值

性能开销:           较高 (每次调用都回调)         较低 (批量写入)

适用场景:           实时响应、行为修改            高性能数据采集、事后分析

API 配置:           rocprofiler_configure_        rocprofiler_configure_
                    callback_tracing_service()    buffer_tracing_service()

数据接收:           工具回调函数                  Buffer 回调函数
                    (rocprofiler_callback_        (rocprofiler_buffer_
                    tracing_cb_t)                 tracing_cb_t)
```

---

## Kernel Dispatch 数据流示例

以内核派发为例，展示完整的数据流路径：

```
应用程序                              rocprofiler-sdk                    GPU 驱动
   |                                       |                               |
   |  hipLaunchKernel(kernel, grid, block) |                               |
   |-------------------------------------->|                               |
   |                                       |                               |
   |  [HIP 拦截器]                         |                               |
   |  1. 生成关联 ID (internal=1001)       |                               |
   |  2. 采集开始时间戳 (T1)               |                               |
   |  3. 检查活跃 Context                  |                               |
   |                                       |                               |
   |  [Callback Tracing - ENTER]           |                               |
   |  4. 调用工具回调:                     |                               |
   |     record.phase = ENTER              |                               |
   |     record.payload = {args}           |                               |
   |                                       |                               |
   |  [Buffer Tracing - ENQUEUE]           |                               |
   |  5. 写入缓冲区:                       |                               |
   |     kind = KERNEL_DISPATCH_ENQUEUE    |                               |
   |     start_timestamp = T1              |                               |
   |                                       |                               |
   |  [执行真实 HIP API]                   |                               |
   |  6. 调用 hipLaunchKernel 真实实现     |----------------------------->|
   |                                       |                               |
   |                                       |  [HSA 拦截器]                 |
   |                                       |  7. 捕获 hsa_queue_add_       |
   |                                       |     packet 函数调用           |
   |                                       |  8. 生成新关联 ID (1002)      |
   |                                       |     ancestor = 1001           |
   |                                       |  9. 配置 AQL 包               |
   |                                       |     - 关联 ID                 |
   |                                       |     - 活跃 Context 快照       |
   |                                       |                               |
   |                                       |  10. 提交到 GPU 队列          |
   |                                       |----------------------------->|
   |                                       |                               |
   |  [HIP 拦截器]                         |                               |
   |  11. 采集结束时间戳 (T2)              |                               |
   |                                       |                               |
   |  [Callback Tracing - EXIT]            |                               |
   |  12. 调用工具回调:                    |                               |
   |      record.phase = EXIT              |                               |
   |      record.payload = {retval}        |                               |
   |                                       |                               |
   |  [Buffer Tracing - ENQUEUE 完成]      |                               |
   |  13. 更新缓冲区记录:                  |                               |
   |      end_timestamp = T2               |                               |
   |                                       |                               |
   |  返回                                 |                               |
   |<--------------------------------------|                               |
   |                                       |                               |
   |                                       |  [GPU 执行完成]               |
   |                                       |<-----------------------------|
   |                                       |                               |
   |                                       |  [信号处理]                   |
   |                                       |  14. 检测到内核完成           |
   |                                       |  15. 采集完成时间戳 (T3)      |
   |                                       |                               |
   |                                       |  [Callback Tracing - COMPLETE]|
   |                                       |  16. 调用工具回调:            |
   |                                       |      kind = KERNEL_DISPATCH   |
   |                                       |      operation = COMPLETE     |
   |                                       |      payload = {dispatch_info}|
   |                                       |                               |
   |                                       |  [Buffer Tracing - COMPLETE]  |
   |                                       |  17. 写入完成记录到缓冲区:    |
   |                                       |      kind = KERNEL_DISPATCH   |
   |                                       |      operation = COMPLETE     |
   |                                       |      timestamps = {T1, T3}    |
   |                                       |                               |
   |                                       |  [缓冲区刷新]                 |
   |                                       |  18. 达到水位线，触发回调     |
   |                                       |  19. 工具处理记录             |
   |                                       |  20. 输出到文件               |
```

---

## 关联 ID 追踪链

```
hipLaunchKernel()                    [关联 ID: 1001, ancestor: 0]
    |
    +---> hsa_queue_add_packet()     [关联 ID: 1002, ancestor: 1001]
    |         |
    |         +---> GPU 执行         [关联 ID: 1002, 从 AQL 包获取]
    |
    +---> hipMemcpy()                [关联 ID: 1003, ancestor: 0]
              |
              +---> hsa_memory_      [关联 ID: 1004, ancestor: 1003]
                    copy()
```

关联 ID 的 `ancestor` 字段允许工具重建完整的调用链层次结构。

---

## 缓冲区管理机制

### 双缓冲设计

```
+-------------------+     +-------------------+
|   Buffer 0        |     |   Buffer 1        |
|   (当前写入)      |     |   (等待刷新)      |
|                   |     |                   |
|  [record 1]       |     |  [record N+1]     |
|  [record 2]       |     |  [record N+2]     |
|  ...              |     |  ...              |
|  [record N]       |     |  [record N+M]     |
|                   |     |                   |
|  syncer[0] = true |     |  syncer[1] = false|
+-------------------+     +-------------------+
         |                          |
         v                          v
    写入操作                   刷新/读取操作
    (原子 CAS 锁)             (回调处理)
```

### 缓冲区策略

| 策略 | 枚举值 | 行为 |
|------|--------|------|
| DISCARD | `ROCPROFILER_BUFFER_POLICY_DISCARD` | 缓冲区满时丢弃新记录，增加 drop_count |
| LOSSLESS | `ROCPROFILER_BUFFER_POLICY_LOSSLESS` | 缓冲区满时阻塞写入，等待刷新完成 |

---

## 数据输出格式

工具通过 Buffer 回调接收数据后，可输出到多种格式：

| 格式 | 说明 | 文件扩展名 |
|------|------|-----------|
| CSV | 逗号分隔值，易于文本处理 | `.csv` |
| Perfetto | 二进制 protobuf 格式，用于 Perfetto UI | `.pftrace` |
| OTF2 | Open Trace Format 2，适合大规模追踪 | `.otf2` |
| JSON | 自定义 JSON schema | `.json` |
