# ROCprofiler-SDK 核心抽象

本文档描述 ROCprofiler-SDK 中的核心抽象概念，包括每个抽象的定义、对应的头文件和实现文件、关键数据结构以及在整体架构中的角色。

---

## 1. Context（上下文）

### 一句话定义

Context 是服务配置的容器，将一组追踪/计数服务绑定在一起，可通过启动/停止来控制数据采集的生命周期。

### 对应文件

| 类型 | 文件路径 |
|------|----------|
| 公共头文件 | `source/include/rocprofiler-sdk/context.h` |
| 前向声明 | `source/include/rocprofiler-sdk/fwd.h` (第 605 行) |
| 内部实现头文件 | `source/lib/rocprofiler-sdk/context/context.hpp` |
| 核心实现 | `source/lib/rocprofiler-sdk/context/context.cpp` |
| 关联 ID 管理 | `source/lib/rocprofiler-sdk/context/correlation_id.cpp` |
| Domain 管理 | `source/lib/rocprofiler-sdk/context/domain.cpp` |

### 关键数据结构

```c
// 公共 API 层 - 不透明句柄
typedef struct rocprofiler_context_id_t {
    uint64_t handle;
} rocprofiler_context_id_t;
```

```cpp
// 内部实现层 - 完整结构 (context.hpp)
struct context {
    size_t                                    size               = 0;
    uint64_t                                  context_idx        = 0;  // context id
    uint32_t                                  client_idx         = 0;  // tool id
    correlation_tracing_service               correlation_tracer = {};
    std::unique_ptr<callback_tracing_service> callback_tracer    = {};
    std::unique_ptr<buffer_tracing_service>   buffered_tracer    = {};
    std::unique_ptr<dispatch_counter_collection_service> dispatch_counter_collection = {};
    std::unique_ptr<device_counting_service>             device_counter_collection   = {};
    std::unique_ptr<pc_sampling_service>                 pc_sampler                  = {};
    std::unique_ptr<thread_trace::DispatchThreadTracer>  dispatch_thread_trace       = {};
    std::unique_ptr<thread_trace::DeviceThreadTracer>    device_thread_trace         = {};
    std::unique_ptr<spm_dispatch_counter_collection_service> dispatch_spm = {};
};
```

### 在架构中的角色

Context 是 ROCprofiler-SDK 的核心组织单元。与旧版工具不同，SDK 要求工具在初始化时显式创建 Context 并配置所需的服务。这种设计使得 SDK 能够：
- 仅准备被请求的服务，避免不必要的开销
- 支持多工具同时使用（每个工具有独立的 Context）
- 在配置阶段进行验证，及早发现冲突

### 公共 API

| 函数 | 说明 |
|------|------|
| `rocprofiler_create_context()` | 创建新的 Context |
| `rocprofiler_start_context()` | 启动 Context，开始数据采集 |
| `rocprofiler_stop_context()` | 停止 Context，停止数据采集 |
| `rocprofiler_context_is_active()` | 查询 Context 是否处于活跃状态 |

---

## 2. Buffer（缓冲区）

### 一句话定义

Buffer 是用于存储异步追踪数据的内存区域，支持双缓冲机制，当数据达到水位线时通过回调通知工具。

### 对应文件

| 类型 | 文件路径 |
|------|----------|
| 公共头文件 | `source/include/rocprofiler-sdk/buffer.h` |
| 前向声明 | `source/include/rocprofiler-sdk/fwd.h` (第 674 行) |
| 内部实现头文件 | `source/lib/rocprofiler-sdk/buffer.hpp` |
| 核心实现 | `source/lib/rocprofiler-sdk/buffer.cpp` |
| Buffer Tracing 实现 | `source/lib/rocprofiler-sdk/buffer_tracing.cpp` |

### 关键数据结构

```c
// 公共 API 层 - 不透明句柄
typedef struct rocprofiler_buffer_id_t {
    uint64_t handle;
} rocprofiler_buffer_id_t;

// Buffer 回调函数类型
typedef void (*rocprofiler_buffer_tracing_cb_t)(
    rocprofiler_context_id_t      context,
    rocprofiler_buffer_id_t       buffer_id,
    rocprofiler_record_header_t** headers,
    size_t                        num_headers,
    void*                         data,
    uint64_t                      drop_count
);
```

```cpp
// 内部实现层 - Buffer 实例 (buffer.hpp)
struct instance {
    using buffer_t = common::container::record_header_buffer;
    static constexpr auto size = 2;  // 双缓冲

    std::array<buffer_t, size>         buffers       = {};
    std::array<std::atomic_flag, size> syncer        = {};  // 读写锁
    std::atomic<uint32_t>              buffer_idx    = {};  // 当前缓冲区索引
    std::atomic<uint64_t>              drop_count    = {};
    uint64_t                           watermark     = 0;
    uint64_t                           context_id    = 0;
    uint64_t                           buffer_id     = 0;
    uint64_t                           task_group_id = 0;  // 线程池分配
    rocprofiler_buffer_tracing_cb_t    callback      = nullptr;
    void*                              callback_data = nullptr;
    rocprofiler_buffer_policy_t        policy        = ROCPROFILER_BUFFER_POLICY_NONE;
};
```

### 在架构中的角色

Buffer 是异步数据采集模式的核心组件。当工具配置 Buffer Tracing 服务时，SDK 将追踪数据写入 Buffer 而非立即回调。Buffer 支持两种策略：
- **DISCARD**: 缓冲区满时丢弃新记录
- **LOSSLESS**: 缓冲区满时阻塞写入，等待刷新

双缓冲机制允许在一个缓冲区被读取/处理的同时，另一个缓冲区继续接收数据。

### 公共 API

| 函数 | 说明 |
|------|------|
| `rocprofiler_create_buffer()` | 创建缓冲区并绑定到 Context |
| `rocprofiler_destroy_buffer()` | 销毁缓冲区 |
| `rocprofiler_flush_buffer()` | 手动刷新缓冲区，触发回调 |
| `rocprofiler_query_buffer_bytes()` | 查询缓冲区已使用字节数 |
| `rocprofiler_buffer_packet_count()` | 查询缓冲区中的记录数量 |

---

## 3. Agent（代理）

### 一句话定义

Agent 代表系统中的一个计算设备（GPU 或 CPU），提供设备属性查询和设备标识功能。

### 对应文件

| 类型 | 文件路径 |
|------|----------|
| 公共头文件 | `source/include/rocprofiler-sdk/agent.h` |
| 前向声明 | `source/include/rocprofiler-sdk/fwd.h` (第 682 行) |
| 内部实现头文件 | `source/lib/rocprofiler-sdk/agent.hpp` |
| 核心实现 | `source/lib/rocprofiler-sdk/agent.cpp` |
| HSA Agent 缓存 | `source/lib/rocprofiler-sdk/hsa/agent_cache.hpp` |

### 关键数据结构

```c
// 公共 API 层 - Agent 标识
typedef struct rocprofiler_agent_id_t {
    uint64_t handle;
} rocprofiler_agent_id_t;

// Agent 类型枚举
typedef enum rocprofiler_agent_type_t {
    ROCPROFILER_AGENT_TYPE_NONE = 0,
    ROCPROFILER_AGENT_TYPE_CPU,
    ROCPROFILER_AGENT_TYPE_GPU,
} rocprofiler_agent_type_t;

// Agent 信息结构 (v0)
typedef struct rocprofiler_agent_v0_t {
    // GPU/CPU 设备属性...
    rocprofiler_agent_id_t   id;
    rocprofiler_agent_type_t type;
    // ... 更多属性
} rocprofiler_agent_v0_t;
```

```cpp
// 内部实现层 - UUID 视图
struct uuid_view_t {
    union {
        uint8_t  bytes[16];
        uint64_t value64[2];
    };
};
```

### 在架构中的角色

Agent 是 SDK 与硬件设备交互的抽象层。SDK 在初始化时枚举系统中的所有 Agent，并缓存其属性信息。其他服务（如计数器采集、PC 采样、内核追踪）都需要通过 Agent ID 来指定目标设备。

### 公共 API

| 函数 | 说明 |
|------|------|
| `rocprofiler_query_agents()` | 查询系统中所有 Agent 信息 |
| `rocprofiler_agent_get_timestamp()` | 获取 Agent 时间戳 |
| `rocprofiler_agent_get_timestamp_frequency()` | 获取时间戳频率 |

---

## 4. Callback Tracing（回调追踪）

### 一句话定义

Callback Tracing 是一种同步追踪模式，在运行时 API 调用发生时立即在调用线程上触发回调函数。

### 对应文件

| 类型 | 文件路径 |
|------|----------|
| 公共头文件 | `source/include/rocprofiler-sdk/callback_tracing.h` |
| 前向声明 | `source/include/rocprofiler-sdk/fwd.h` (第 157 行, 第 718 行) |
| 核心实现 | `source/lib/rocprofiler-sdk/callback_tracing.cpp` |
| 内部服务结构 | `source/lib/rocprofiler-sdk/context/context.hpp` |

### 关键数据结构

```c
// 回调追踪记录
typedef struct rocprofiler_callback_tracing_record_t {
    rocprofiler_context_id_t            context_id;
    rocprofiler_thread_id_t             thread_id;
    rocprofiler_correlation_id_t        correlation_id;
    rocprofiler_callback_tracing_kind_t kind;
    rocprofiler_tracing_operation_t     operation;
    rocprofiler_callback_phase_t        phase;
    void*                               payload;
} rocprofiler_callback_tracing_record_t;

// 回调函数类型
typedef void (*rocprofiler_callback_tracing_cb_t)(
    rocprofiler_context_id_t              context,
    rocprofiler_callback_tracing_record_t record,
    void*                                 user_data
);

// 回调阶段
typedef enum rocprofiler_callback_phase_t {
    ROCPROFILER_CALLBACK_PHASE_NONE  = 0,
    ROCPROFILER_CALLBACK_PHASE_ENTER,  // API 调用前
    ROCPROFILER_CALLBACK_PHASE_EXIT,   // API 调用后
} rocprofiler_callback_phase_t;
```

```cpp
// 内部服务结构
struct callback_tracing_service {
    struct callback_data {
        rocprofiler_callback_tracing_cb_t callback = nullptr;
        void*                             data     = nullptr;
    };
    using domain_t         = rocprofiler_callback_tracing_kind_t;
    using callback_array_t = std::array<callback_data, domain_info<domain_t>::last>;
    domain_context<domain_t> domains       = {};
    callback_array_t         callback_data = {};
};
```

### 在架构中的角色

Callback Tracing 适合需要实时响应的场景，例如：
- 在 API 调用前后修改行为
- 实时统计 API 调用次数
- 与外部追踪系统集成

回调在 ENTER 和 EXIT 两个阶段触发，允许工具在 API 调用前后分别执行操作。

---

## 5. Buffer Tracing（缓冲追踪）

### 一句话定义

Buffer Tracing 是一种异步追踪模式，将追踪数据写入内部缓冲区，由后台线程批量处理并回调通知工具。

### 对应文件

| 类型 | 文件路径 |
|------|----------|
| 公共头文件 | `source/include/rocprofiler-sdk/buffer_tracing.h` |
| 前向声明 | `source/include/rocprofiler-sdk/fwd.h` (第 192 行) |
| 核心实现 | `source/lib/rocprofiler-sdk/buffer_tracing.cpp` |
| 内部服务结构 | `source/lib/rocprofiler-sdk/context/context.hpp` |

### 关键数据结构

```c
// Buffer 追踪记录示例 (HSA API)
typedef struct rocprofiler_buffer_tracing_hsa_api_record_t {
    uint64_t                          size;
    rocprofiler_buffer_tracing_kind_t kind;
    rocprofiler_tracing_operation_t   operation;
    rocprofiler_correlation_id_t      correlation_id;
    rocprofiler_timestamp_t           start_timestamp;
    rocprofiler_timestamp_t           end_timestamp;
    rocprofiler_thread_id_t           thread_id;
} rocprofiler_buffer_tracing_hsa_api_record_t;

// 通用记录头
typedef struct rocprofiler_record_header_t {
    union {
        struct {
            uint32_t category;  // rocprofiler_buffer_category_t
            uint32_t kind;      // domain-specific kind
        };
        uint64_t hash;
    };
    void* payload;
} rocprofiler_record_header_t;
```

```cpp
// 内部服务结构
struct buffer_tracing_service {
    using domain_t       = rocprofiler_buffer_tracing_kind_t;
    using buffer_array_t = std::array<rocprofiler_buffer_id_t, domain_info<domain_t>::last>;
    domain_context<domain_t> domains     = {};
    buffer_array_t           buffer_data = {};
};
```

### 在架构中的角色

Buffer Tracing 适合高性能数据采集场景，相比 Callback Tracing 有更低的运行时开销。数据记录包含完整的时序信息（开始/结束时间戳），适合事后分析。

---

## 6. Correlation ID（关联 ID）

### 一句话定义

Correlation ID 是用于关联同一操作在不同层级（API 调用、内核派发、内存拷贝等）产生的追踪记录的唯一标识符。

### 对应文件

| 类型 | 文件路径 |
|------|----------|
| 前向声明 | `source/include/rocprofiler-sdk/fwd.h` (第 629 行) |
| 外部关联 API | `source/include/rocprofiler-sdk/external_correlation.h` |
| 核心实现 | `source/lib/rocprofiler-sdk/context/correlation_id.cpp` |
| 外部关联实现 | `source/lib/rocprofiler-sdk/external_correlation.cpp` |

### 关键数据结构

```c
// 关联 ID 结构
typedef struct rocprofiler_correlation_id_t {
    uint64_t                internal;   // SDK 内部生成的唯一 ID
    rocprofiler_user_data_t external;   // 工具指定的外部 ID
    uint64_t                ancestor;   // 产生此操作的父操作的 internal ID
} rocprofiler_correlation_id_t;

// 用户数据联合体
typedef union rocprofiler_user_data_t {
    uint64_t value;
    void*    ptr;
} rocprofiler_user_data_t;
```

### 在架构中的角色

Correlation ID 是连接不同追踪记录的桥梁。例如，一个 HIP API 调用可能触发 HSA API 调用和内核派发，这些不同层级的事件通过 Correlation ID 关联起来，形成完整的调用链。`ancestor` 字段支持多级嵌套追踪。

---

## 7. Registration（注册）

### 一句话定义

Registration 是工具注册机制，允许工具通过 `rocprofiler_configure` 函数注册初始化和终止回调，实现工具的自动生命周期管理。

### 对应文件

| 类型 | 文件路径 |
|------|----------|
| 公共头文件 | `source/include/rocprofiler-sdk/registration.h` |
| 内部实现头文件 | `source/lib/rocprofiler-sdk/registration.hpp` |
| 核心实现 | `source/lib/rocprofiler-sdk/registration/registration.cpp` |

### 关键数据结构

```c
// 客户端标识
typedef struct rocprofiler_client_id_t {
    size_t         size;
    const char*    name;      // 客户端名称（用于调试）
    const uint32_t handle;    // SDK 分配的唯一句柄
} rocprofiler_client_id_t;

// 工具初始化回调
typedef int (*rocprofiler_tool_initialize_t)(
    rocprofiler_client_finalize_t finalize_func,
    void*                         tool_data
);

// 工具终止回调
typedef void (*rocprofiler_tool_finalize_t)(void* tool_data);

// 配置结果结构
typedef struct rocprofiler_tool_configure_result_t {
    size_t                         size;
    rocprofiler_tool_initialize_t  initialize;
    rocprofiler_tool_finalize_t    finalize;
    void*                          tool_data;
} rocprofiler_tool_configure_result_t;
```

### 在架构中的角色

Registration 机制是工具与 SDK 交互的入口点。工具通过实现 `rocprofiler_configure` 函数返回配置结果，SDK 在适当的时机调用初始化和终止回调。这种设计支持：
- 多工具同时注册
- 自动生命周期管理（通过 atexit 处理器）
- 延迟初始化（在运行时库加载后）

---

## 8. Intercept Table（拦截表）

### 一句话定义

Intercept Table 是运行时 API 函数指针表，允许工具在运行时 API 调用前后插入自定义逻辑，实现 API 包装和链式调用。

### 对应文件

| 类型 | 文件路径 |
|------|----------|
| 公共头文件 | `source/include/rocprofiler-sdk/intercept_table.h` |
| 内部实现头文件 | `source/lib/rocprofiler-sdk/intercept_table.hpp` |
| 核心实现 | `source/lib/rocprofiler-sdk/intercept_table.cpp` |

### 关键数据结构

```c
// 拦截表类型
typedef enum rocprofiler_intercept_table_t {
    ROCPROFILER_HSA_TABLE            = (1 << 0),
    ROCPROFILER_HIP_RUNTIME_TABLE    = (1 << 1),
    ROCPROFILER_HIP_COMPILER_TABLE   = (1 << 2),
    ROCPROFILER_MARKER_CORE_TABLE    = (1 << 3),
    ROCPROFILER_MARKER_CONTROL_TABLE = (1 << 4),
    ROCPROFILER_MARKER_NAME_TABLE    = (1 << 5),
    ROCPROFILER_RCCL_TABLE           = (1 << 6),
    ROCPROFILER_ROCDECODE_TABLE      = (1 << 7),
    ROCPROFILER_ROCJPEG_TABLE        = (1 << 8),
} rocprofiler_intercept_table_t;

// 拦截表注册回调
typedef void (*rocprofiler_intercept_library_cb_t)(
    rocprofiler_intercept_table_t type,
    uint64_t                      lib_version,
    uint64_t                      lib_instance,
    void**                        tables,
    uint64_t                      num_tables,
    void*                         user_data
);
```

### 在架构中的角色

Intercept Table 是运行时拦截层的核心机制。当运行时库（如 HSA、HIP）加载时，SDK 通过拦截表 hook API 函数。调用链为：
```
应用程序 -> 运行时函数 -> 工具包装器 -> rocprofiler 包装器 -> 真实实现
```

工具可以通过 `rocprofiler_at_intercept_table_registration` 注册回调，在新的运行时库加载时获取拦截表。

---

## 9. Domain（域）

### 一句话定义

Domain 是追踪操作的分类单元，将相关的 API 函数或事件归为一组，用于配置追踪范围和组织追踪数据。

### 对应文件

| 类型 | 文件路径 |
|------|----------|
| 前向声明 | `source/include/rocprofiler-sdk/fwd.h` (第 157-248 行) |
| 内部实现 | `source/lib/rocprofiler-sdk/context/domain.hpp` |
| Domain 实现 | `source/lib/rocprofiler-sdk/context/domain.cpp` |

### 关键数据结构

```c
// Callback Tracing 域
typedef enum rocprofiler_callback_tracing_kind_t {
    ROCPROFILER_CALLBACK_TRACING_NONE = 0,
    ROCPROFILER_CALLBACK_TRACING_HSA_CORE_API,
    ROCPROFILER_CALLBACK_TRACING_HSA_AMD_EXT_API,
    ROCPROFILER_CALLBACK_TRACING_HIP_RUNTIME_API,
    ROCPROFILER_CALLBACK_TRACING_HIP_COMPILER_API,
    ROCPROFILER_CALLBACK_TRACING_MARKER_CORE_API,
    ROCPROFILER_CALLBACK_TRACING_CODE_OBJECT,
    ROCPROFILER_CALLBACK_TRACING_KERNEL_DISPATCH,
    ROCPROFILER_CALLBACK_TRACING_MEMORY_COPY,
    ROCPROFILER_CALLBACK_TRACING_MEMORY_ALLOCATION,
    // ... 更多域
} rocprofiler_callback_tracing_kind_t;

// Buffer Tracing 域
typedef enum rocprofiler_buffer_tracing_kind_t {
    ROCPROFILER_BUFFER_TRACING_NONE = 0,
    ROCPROFILER_BUFFER_TRACING_HSA_CORE_API,
    ROCPROFILER_BUFFER_TRACING_HIP_RUNTIME_API,
    ROCPROFILER_BUFFER_TRACING_KERNEL_DISPATCH,
    ROCPROFILER_BUFFER_TRACING_MEMORY_COPY,
    // ... 更多域
} rocprofiler_buffer_tracing_kind_t;
```

### 在架构中的角色

Domain 是服务配置的基本单位。工具在配置追踪服务时，需要指定要追踪的 Domain。每个 Domain 对应一组相关的 API 函数或事件类型。Domain 机制使得工具可以精确控制追踪范围，避免不必要的开销。

---

## 10. Counter Config（计数器配置）

### 一句话定义

Counter Config 是硬件性能计数器的配置集合，定义了要在特定 Agent 上采集的计数器列表。

### 对应文件

| 类型 | 文件路径 |
|------|----------|
| 公共头文件 | `source/include/rocprofiler-sdk/counters.h` |
| 前向声明 | `source/include/rocprofiler-sdk/fwd.h` (第 699 行) |
| 计数器核心 | `source/lib/rocprofiler-sdk/counters/core.hpp` |
| 计数器实现 | `source/lib/rocprofiler-sdk/counters/counters.cpp` |
| 计数器配置实现 | `source/lib/rocprofiler-sdk/counter_config.cpp` |
| 设备计数 | `source/lib/rocprofiler-sdk/counters/device_counting.hpp` |

### 关键数据结构

```c
// 计数器配置 ID
typedef struct rocprofiler_counter_config_id_t {
    uint64_t handle;
} rocprofiler_counter_config_id_t;

// 计数器 ID
typedef struct rocprofiler_counter_id_t {
    uint64_t handle;
} rocprofiler_counter_id_t;

// 计数器记录
typedef struct rocprofiler_counter_record_t {
    rocprofiler_counter_instance_id_t id;
    double                            counter_value;
    rocprofiler_dispatch_id_t         dispatch_id;
    rocprofiler_user_data_t           user_data;
    rocprofiler_agent_id_t            agent_id;
} rocprofiler_counter_record_t;
```

### 在架构中的角色

Counter Config 允许工具定义要采集的硬件性能计数器集合。工具首先查询可用的计数器，然后创建配置，最后将配置绑定到 Context 的计数器采集服务。支持两种模式：
- **Dispatch Counting**: 每次内核派发后采集计数器值
- **Device Counting**: 持续采集设备级计数器

---

## 抽象关系图

```
                    +-------------------+
                    |   Registration    |
                    | (工具注册入口)     |
                    +-------------------+
                             |
                             v
                    +-------------------+
                    |     Client        |
                    | (工具客户端标识)   |
                    +-------------------+
                             |
                             v
+------------------+  +-------------------+  +------------------+
|     Agent        |  |     Context       |  |  Intercept Table |
| (设备抽象)       |  | (服务容器)        |  | (API 拦截)       |
+------------------+  +-------------------+  +------------------+
                             |
              +--------------+--------------+
              |              |              |
              v              v              v
     +---------------+ +------------+ +----------------+
     |Callback Tracing| |Buffer Tracing| |Counter Config |
     |(同步回调追踪)  | |(异步缓冲追踪)| |(计数器配置)    |
     +---------------+ +------------+ +----------------+
              |              |              |
              v              v              v
     +------------------------------------------------+
     |              Correlation ID                     |
     |         (跨层级记录关联)                         |
     +------------------------------------------------+
              |
              v
     +------------------------------------------------+
     |                   Domain                        |
     |          (追踪操作分类)                          |
     +------------------------------------------------+
```
