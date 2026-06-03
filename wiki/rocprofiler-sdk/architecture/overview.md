# ROCprofiler-SDK 架构概览

> 验证记录：2026-06-03 使用 codegraph 查询 `rocprofiler_create_context`、`rocprofiler_create_buffer`、`rocprofiler_configure_callback_tracing_service`、`rocprofiler_configure_buffer_tracing_service`，并核对 `projects/rocprofiler-sdk/source/lib/rocprofiler-sdk/` 目录结构。 <!-- verified: 2026-06-03 -->

## 整体架构分层图

```
+=========================================================================+
|                          用户工具层 (Tool Layer)                         |
|  (rocprofv3, PAPI, Omnitrace, 自定义工具等)                              |
+=========================================================================+
                              |
                              v
+=========================================================================+
|                       公共 API 层 (Public API)                           |
|  rocprofiler.h | context.h | buffer.h | agent.h | callback_tracing.h    |
|  buffer_tracing.h | counters.h | registration.h | intercept_table.h     |
+=========================================================================+
                              |
                              v
+=========================================================================+
|                        核心 SDK 层 (Core SDK)                            |
|  +------------------+  +------------------+  +------------------------+  |
|  | Context 管理     |  | Buffer 管理      |  | 注册与初始化           |  |
|  | (context/)       |  | (buffer.cpp)     |  | (registration/)        |  |
|  +------------------+  +------------------+  +------------------------+  |
|  +------------------+  +------------------+  +------------------------+  |
|  | Agent 管理       |  | Correlation ID   |  | 内部线程管理           |  |
|  | (agent.cpp)      |  | (context/)       |  | (internal_threading)   |  |
|  +------------------+  +------------------+  +------------------------+  |
+=========================================================================+
                              |
                              v
+=========================================================================+
|                    运行时拦截层 (Runtime Interceptors)                    |
|  +-----------+  +-----------+  +----------+  +--------+  +-----------+  |
|  | HSA 拦截  |  | HIP 拦截  |  | Marker   |  | RCCL   |  | rocDecode |  |
|  | (hsa/)    |  | (hip/)    |  | (marker/)|  | (rccl/)|  | (rocdecode)| |
|  +-----------+  +-----------+  +----------+  +--------+  +-----------+  |
|  +------------------+  +------------------+  +------------------------+  |
|  | Code Object      |  | Kernel Dispatch  |  | OMPT 拦截              |  |
|  | (code_object/)   |  | (kernel_dispatch)|  | (ompt/)                |  |
|  +------------------+  +------------------+  +------------------------+  |
+=========================================================================+
                              |
                              v
+=========================================================================+
|                      GPU 驱动层 (GPU Driver)                             |
|  +------------------+  +------------------+  +------------------------+  |
|  | HSA Runtime      |  | KFD (amdgpu)     |  | AQL Profile            |  |
|  | (hsa lib)        |  | (hsakmt)         |  | (aqlprofile)           |  |
|  +------------------+  +------------------+  +------------------------+  |
|  +------------------+  +------------------+                             |
|  | AMD COMGR        |  | DRM              |                             |
|  | (code object)    |  | (GPU device)     |                             |
|  +------------------+  +------------------+                             |
+=========================================================================+
```

## 各层职责说明

### 1. 用户工具层 (Tool Layer)

用户工具层位于架构的最顶层，包含直接使用 ROCprofiler-SDK 的各类工具：

- **rocprofv3**: 官方命令行性能分析工具，是 `rocprof` / `rocprofv2` 的替代品
- **第三方工具**: 如 PAPI、Omnitrace 等通过 SDK API 集成的性能分析工具
- **自定义工具**: 用户编写的专用性能分析工具

工具通过 `rocprofiler_tool_initialize_t` 回调函数进行初始化，创建所需的 Context 并配置服务。

### 2. 公共 API 层 (Public API)

公共 API 层是 SDK 的用户接口，提供 C 语言 API（兼容 C++），主要头文件包括：

| 头文件 | 职责 |
|--------|------|
| `rocprofiler.h` | 核心 API：版本查询、初始化/终止、通用数据类型定义 |
| `context.h` | Context 生命周期管理：创建、启动、停止、查询 |
| `buffer.h` | Buffer 管理：创建、销毁、刷新、数据读取 |
| `agent.h` | GPU/CPU Agent 信息查询 |
| `callback_tracing.h` | 同步回调追踪服务配置 |
| `buffer_tracing.h` | 异步缓冲追踪服务配置 |
| `counters.h` | 硬件性能计数器配置与查询 |
| `registration.h` | 工具注册机制 |
| `intercept_table.h` | 运行时 API 拦截表管理 |
| `external_correlation.h` | 外部关联 ID 管理 |
| `fwd.h` | 前向声明：所有枚举、结构体、类型别名的定义 |

### 3. 核心 SDK 层 (Core SDK)

核心 SDK 层实现公共 API 的内部逻辑，主要模块包括：

- **Context 管理** (`source/lib/rocprofiler-sdk/context/`): 维护 Context 的生命周期，管理服务配置、Domain 和关联 ID
- **Buffer 管理** (`source/lib/rocprofiler-sdk/buffer.cpp`, `buffer.hpp`): 实现数据缓冲区的分配、写入、刷新和回调通知
- **Agent 管理** (`source/lib/rocprofiler-sdk/agent.cpp`, `agent.hpp`): 枚举和管理 GPU/CPU Agent 信息
- **注册与初始化** (`source/lib/rocprofiler-sdk/registration/`): 管理工具注册流程，协调多客户端初始化
- **内部线程管理** (`source/lib/rocprofiler-sdk/internal_threading.cpp`): 创建和管理 SDK 内部工作线程
- **关联 ID 管理** (`source/lib/rocprofiler-sdk/context/correlation_id.cpp`): 生成和追踪内部/外部关联 ID

### 4. 运行时拦截层 (Runtime Interceptors)

运行时拦截层通过拦截表（Intercept Table）机制 hook 各种运行时 API：

| 拦截模块 | 源码目录 | 拦截目标 |
|----------|----------|----------|
| HSA 拦截 | `source/lib/rocprofiler-sdk/hsa/` | HSA Core API、AMD Extension API、Image API、Finalizer API |
| HIP 拦截 | `source/lib/rocprofiler-sdk/hip/` | HIP Runtime API、HIP Compiler API |
| Marker 拦截 | `source/lib/rocprofiler-sdk/marker/` | ROCTx Marker API (Core、Control、Name) |
| RCCL 拦截 | `source/lib/rocprofiler-sdk/rccl/` | RCCL 通信库 API |
| rocDecode 拦截 | `source/lib/rocprofiler-sdk/rocdecode/` | rocDecode 视频解码 API |
| rocJPEG 拦截 | `source/lib/rocprofiler-sdk/rocjpeg/` | rocJPEG 图像处理 API |
| Code Object | `source/lib/rocprofiler-sdk/code_object/` | 代码对象加载/卸载事件 |
| Kernel Dispatch | `source/lib/rocprofiler-sdk/kernel_dispatch/` | 内核派发事件 |
| KFD | `source/lib/rocprofiler-sdk/kfd/` | KFD 内核事件（页面迁移、缺页等） |
| OMPT | `source/lib/rocprofiler-sdk/ompt/` | OpenMP Tools API |

拦截机制的工作原理：当应用程序调用运行时 API（如 `hsa_init`）时，调用链为：
```
应用程序 -> 运行时库函数 -> 工具包装器 -> rocprofiler 包装器 -> 真实实现
```

### 5. GPU 驱动层 (GPU Driver)

GPU 驱动层与底层 AMD GPU 驱动和运行时交互：

- **HSA Runtime**: 异构系统架构运行时，提供 GPU 内核调度、内存管理等基础功能
- **KFD (amdgpu)**: 内核态驱动，提供 GPU 设备管理、内存映射等底层功能
- **AQL Profile**: 硬件性能计数器采集的底层接口
- **AMD COMGR**: 代码对象管理库，用于加载和解析 GPU 代码对象
- **DRM**: 直接渲染管理器，用于 GPU 设备访问

## 关键设计决策

### 1. Context 为中心的服务模型

**决策**: 引入 Context 概念，将服务配置绑定到 Context 上。

**原因**: 旧版 ROCProfiler/ROCTracer 工具在初始化时不需要指定要使用的服务，导致库必须随时准备处理任何服务请求，引入了不必要的开销并使线程安全管理变得困难。

**优势**:
- 避免为从未使用的服务做准备（如未请求 HSA API 追踪则不生成包装器）
- 在服务配置阶段进行更严格的检查，及早发现问题
- 允许多个工具同时使用某些服务
- 在不引入并行瓶颈的情况下提高线程安全性

### 2. 双追踪模式：Callback Tracing 与 Buffer Tracing

**决策**: 提供两种追踪模式供工具选择。

- **Callback Tracing (同步回调)**: 在调用线程上立即触发回调，适合需要实时响应的场景
- **Buffer Tracing (异步缓冲)**: 数据写入内部缓冲区，由后台线程批量处理，适合高性能数据采集

### 3. 多客户端支持

**决策**: 允许多个工具同时注册并使用 SDK 服务。

**实现**: 通过 `rocprofiler_client_id_t` 标识每个客户端，每个客户端可以独立创建和管理自己的 Context。

### 4. C ABI 兼容性

**决策**: 使用纯 C API 接口，确保 ABI 向后兼容。

**实现**:
- 使用 `ROCPROFILER_EXTERN_C_INIT` / `ROCPROFILER_EXTERN_C_FINI` 宏包裹 C 接口
- 使用不透明句柄（如 `rocprofiler_context_id_t`）隐藏内部实现
- 版本化符号（`ROCPROFILER_SDK_VERSION_0_0`）

### 5. 可扩展的记录格式

**决策**: 使用 `rocprofiler_record_header_t` 作为通用记录头，通过 category + kind 两级标识区分记录类型。

**优势**: 新增追踪类型无需修改核心缓冲机制，只需定义新的 category/kind 组合和对应的 payload 结构。

## 源码目录结构

```
projects/rocprofiler-sdk/
├── source/
│   ├── include/rocprofiler-sdk/    # 公共 API 头文件
│   │   ├── rocprofiler.h           # 核心 API
│   │   ├── context.h               # Context 管理
│   │   ├── buffer.h                # Buffer 管理
│   │   ├── agent.h                 # Agent 信息
│   │   ├── callback_tracing.h      # 同步追踪
│   │   ├── buffer_tracing.h        # 异步追踪
│   │   ├── counters.h              # 性能计数器
│   │   ├── registration.h          # 工具注册
│   │   ├── intercept_table.h       # 拦截表
│   │   ├── fwd.h                   # 前向声明
│   │   ├── defines.h               # 宏定义
│   │   └── hsa/, hip/, marker/...  # 运行时特定头文件
│   └── lib/rocprofiler-sdk/        # SDK 实现
│       ├── context/                # Context 实现
│       ├── tracing/                # 追踪基础设施
│       ├── hsa/                    # HSA 拦截实现
│       ├── hip/                    # HIP 拦截实现
│       ├── marker/                 # Marker 拦截实现
│       ├── code_object/            # Code Object 追踪
│       ├── kernel_dispatch/        # 内核派发追踪
│       ├── counters/               # 计数器实现
│       ├── aql/                    # AQL Profile 集成
│       ├── pc_sampling/            # PC 采样
│       ├── kfd/                    # KFD 事件追踪
│       └── registration/           # 注册机制
├── external/                       # 外部依赖
├── samples/                        # 示例代码
├── tests/                          # 测试代码
└── cmake/                          # CMake 构建配置
```
