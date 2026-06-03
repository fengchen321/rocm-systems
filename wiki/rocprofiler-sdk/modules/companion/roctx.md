# ROCTx 标记库 (ROCm Tools Extension)

## 概述

ROCTx 模块 (`source/lib/rocprofiler-sdk-roctx/`) 是 rocprofiler-sdk 的用户代码标注库，提供 ROCm Tools Extension (ROCTx) API。该模块允许用户在应用程序中插入标记 (markers) 和范围 (ranges) 来标注代码区域，以便在性能分析工具（如 `rocprofv3`）的输出中可视化这些标注。ROCTx 通过 `rocprofiler-register` 机制注册 API 表，支持工具拦截器对 API 调用的监控。它是连接用户应用程序和 rocprofiler-sdk 工具生态的桥梁。

## 关键文件

| 文件 | 路径 | 职责 |
|------|------|------|
| `roctx.cpp` | `source/lib/rocprofiler-sdk-roctx/roctx.cpp` | ROCTx 核心实现：API 函数、API 表注册、内部状态管理 |
| `abi.cpp` | `source/lib/rocprofiler-sdk-roctx/abi.cpp` | ABI 版本检查和函数指针顺序验证 |
| `CMakeLists.txt` | `source/lib/rocprofiler-sdk-roctx/CMakeLists.txt` | 构建配置，生成 `librocprofiler-sdk-roctx.so` 共享库 |

## 核心数据结构

- **`roctx_api_table`** (`roctx.cpp`): 内部 API 表结构体，聚合三个子表：
  - `core`: `roctxCoreApiTable_t` - 核心 API 函数指针表
  - `control`: `roctxControlApiTable_t` - 控制 API 函数指针表
  - `name`: `roctxNameApiTable_t` - 命名 API 函数指针表<!-- verified: 2026-05-27 -->

- **`roctxCoreApiTable_t`** (`roctx.h`): 核心 API 表，包含 6 个函数指针：
  - `roctxMarkA_fn` (索引 0): 插入标记
  - `roctxRangePushA_fn` (索引 1): 压入嵌套范围
  - `roctxRangePop_fn` (索引 2): 弹出嵌套范围
  - `roctxRangeStartA_fn` (索引 3): 启动独立范围
  - `roctxRangeStop_fn` (索引 4): 停止独立范围
  - `roctxGetThreadId_fn` (索引 5): 获取线程 ID<!-- verified: 2026-05-27 -->

- **`roctxControlApiTable_t`** (`roctx.h`): 控制 API 表，包含 2 个函数指针：
  - `roctxProfilerPause_fn` (索引 0): 暂停分析器
  - `roctxProfilerResume_fn` (索引 1): 恢复分析器<!-- verified: 2026-05-27 -->

- **`roctxNameApiTable_t`** (`roctx.h`): 命名 API 表，包含 4 个函数指针：
  - `roctxNameOsThread_fn` (索引 0): 命名 OS 线程
  - `roctxNameHsaAgent_fn` (索引 1): 命名 HSA 代理
  - `roctxNameHipDevice_fn` (索引 2): 命名 HIP 设备
  - `roctxNameHipStream_fn` (索引 3): 命名 HIP 流<!-- verified: 2026-05-27 -->

- **`nested_range_level_buffer`** (`roctx.cpp`): 线程局部存储，维护当前线程的嵌套范围层级计数器。使用 placement new 在静态缓冲区上构造。<!-- verified: 2026-05-27 -->

- **`start_stop_range_id_buffer`** (`roctx.cpp`): 全局原子计数器，为 `roctxRangeStartA` 分配唯一的范围 ID。<!-- verified: 2026-05-27 -->

## 关键函数

- **`roctxMarkA(const char* message)`** (`roctx.cpp`): 插入一个标记点。在内部实现中为空操作（无实际行为），但工具拦截器可以捕获此调用。<!-- verified: 2026-05-27 -->

- **`roctxRangePushA(const char* message)`** (`roctx.cpp`): 压入一个嵌套范围。返回当前嵌套层级（从 0 开始递增）。通过 `get_nested_range_level()` 访问线程局部计数器。<!-- verified: 2026-05-27 -->

- **`roctxRangePop()`** (`roctx.cpp`): 弹出当前嵌套范围。如果嵌套层级为 0（无匹配的 Push），返回 -1。否则返回递减后的层级。<!-- verified: 2026-05-27 -->

- **`roctxRangeStartA(const char* message)`** (`roctx.cpp`): 启动一个独立范围。通过原子计数器分配唯一范围 ID 并返回。<!-- verified: 2026-05-27 -->

- **`roctxRangeStop(roctx_range_id_t id)`** (`roctx.cpp`): 停止指定 ID 的独立范围。在内部实现中为空操作。<!-- verified: 2026-05-27 -->

- **`roctxGetThreadId(roctx_thread_id_t* tid)`** (`roctx.cpp`): 获取当前线程 ID，通过 `common::get_tid()` 实现。<!-- verified: 2026-05-27 -->

- **`roctxProfilerPause(roctx_thread_id_t tid)`** (`roctx.cpp`): 请求暂停分析器。返回 0。<!-- verified: 2026-05-27 -->

- **`roctxProfilerResume(roctx_thread_id_t tid)`** (`roctx.cpp`): 请求恢复分析器。返回 0。<!-- verified: 2026-05-27 -->

- **`roctxNameOsThread(const char* name)`** (`roctx.cpp`): 为当前 OS 线程命名。返回 0。<!-- verified: 2026-05-27 -->

- **`roctxNameHsaAgent(const char* name, const hsa_agent_s* agent)`** (`roctx.cpp`): 为 HSA 代理命名。返回 0。<!-- verified: 2026-05-27 -->

- **`roctxNameHipDevice(const char* name, int device_id)`** (`roctx.cpp`): 为 HIP 设备命名。返回 0。<!-- verified: 2026-05-27 -->

- **`roctxNameHipStream(const char* name, const ihipStream_t* stream)`** (`roctx.cpp`): 为 HIP 流命名。返回 0。<!-- verified: 2026-05-27 -->

- **`get_table_impl()`** (`roctx.cpp`): 内部函数，初始化 API 表并通过 `rocprofiler_register_library_api_table()` 注册到 rocprofiler-register 系统。返回 API 表的静态引用。<!-- verified: 2026-05-27 -->

- **`get_table()`** (`roctx.cpp`): 内部函数，返回已初始化的 API 表指针（延迟初始化）。<!-- verified: 2026-05-27 -->

## 调用关系

- **上游**：
  - 用户应用程序：直接调用 `roctxMarkA()`、`roctxRangePushA()` 等公开 API
  - `rocprofiler-sdk` 工具拦截器：通过 `rocprofiler-register` 机制拦截 ROCTx API 调用
  - Python 绑定：通过 `roctx` Python 模块调用 ROCTx API

- **下游**：
  - `rocprofiler-register`：通过 `rocprofiler_register_library_api_table()` 注册 API 表
  - `rocprofiler::common` 库：使用 `get_tid()`、`init_logging()`、`static_object` 等基础设施
  - `rocprofiler-register` 头文件：`rocprofiler-register.h` 提供注册 API

## 数据流

1. **库加载**：`librocprofiler-sdk-roctx.so` 被加载时，`get_table_impl()` 被调用
2. **API 表初始化**：创建 `roctx_api_table` 实例，填充三个子表的函数指针
3. **注册**：通过 `rocprofiler_register_library_api_table()` 将 API 表注册到 rocprofiler-register 系统
4. **工具拦截**：rocprofiler-register 将 API 表传递给已注册的工具，工具可以替换函数指针以拦截 API 调用
5. **用户调用**：应用程序调用 ROCTx API（如 `roctxRangePushA("my_region")`）
6. **函数分发**：公开 API 函数通过 `get_table()->core.roctxRangePushA_fn(message)` 调用实际实现
7. **状态更新**：内部实现更新线程局部状态（嵌套层级、范围 ID）
8. **数据记录**：工具拦截器记录 API 调用信息（时间戳、消息、线程 ID 等）

## 已知限制与边界情况

- **内部实现为空操作**：`roctxMarkA()`、`roctxRangeStop()`、`roctxProfilerPause()`、`roctxProfilerResume()` 等函数的内部实现为空操作，实际功能依赖工具拦截器。
- **嵌套范围层级限制**：`roctxRangePop()` 在嵌套层级为 0 时返回 -1，表示没有匹配的 Push 调用。
- **范围 ID 溢出**：`start_stop_range_id_buffer` 使用 `std::atomic<roctx_range_id_t>`，理论上可能溢出（取决于 `roctx_range_id_t` 的类型大小）。
- **ABI 版本控制**：`abi.cpp` 通过 `ROCP_SDK_ENFORCE_ABI_VERSIONING` 和 `ROCP_SDK_ENFORCE_ABI` 宏确保 API 表的 ABI 稳定性。如果函数指针顺序改变，编译时会报错。
- **线程安全**：嵌套范围层级使用 `thread_local` 存储，每个线程独立。范围 ID 使用原子操作，支持多线程并发分配。
- **延迟初始化**：API 表在首次调用时初始化（通过 `get_table()` 的静态局部变量）。
- **注册失败处理**：如果 `rocprofiler_register_library_api_table()` 返回非 `ROCP_REG_SUCCESS` 且非 `ROCP_REG_NO_TOOLS` 的状态，会记录警告日志。
- **placement new 使用**：使用 placement new 在静态缓冲区上构造对象，避免静态初始化顺序问题。

## 与官方文档的关系

- [使用 ROCTx](../../../../projects/rocprofiler-sdk/source/docs/how-to/using-rocprofiler-sdk-roctx.rst) - ROCTx 的完整使用指南，包括标记、范围、API 列表和示例代码
- [使用 rocprofv3](../../../../projects/rocprofiler-sdk/source/docs/how-to/using-rocprofv3.rst) - rocprofv3 工具的使用指南，包含标记追踪功能
- [rocprofv3 I/O 控制选项](../../../../projects/rocprofiler-sdk/source/docs/how-to/rocprofv3-io-options.rst) - 输出格式和路径配置选项
