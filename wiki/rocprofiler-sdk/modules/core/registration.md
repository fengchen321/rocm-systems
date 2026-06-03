# 工具注册系统

## 概述

注册系统是 rocprofiler-sdk 的核心入口模块，负责管理整个 SDK 的生命周期，包括工具发现、配置、初始化和终结。该模块是连接外部工具库（通过 `rocprofiler_configure` 符号）与 SDK 内部子系统的桥梁。它处理运行时 API 表的注册（HSA、HIP、ROCTx、RCCL、rocdecode、rocjpeg 等），协调多个客户端工具的并发初始化和终结，并提供 attach/detach 机制支持进程附加场景。注册系统的设计确保了工具的发现和初始化是线程安全的，并且支持延迟加载和强制配置等高级场景。

## 关键文件

| 文件 | 路径 | 职责 |
|------|------|------|
| registration.cpp | source/lib/rocprofiler-sdk/registration.cpp | 注册系统的核心实现，包含工具发现、生命周期管理、API 表分发 |
| registration.hpp | source/lib/rocprofiler-sdk/registration.hpp | 内部头文件，声明注册系统的内部接口 |
| registration.h | source/include/rocprofiler-sdk/registration.h | 公共 API 头文件，定义注册相关的公共接口 |
| iterate.hpp | source/lib/rocprofiler-sdk/registration/iterate.hpp | 迭代运行时注册信息的辅助模块 |
| late.hpp | source/lib/rocprofiler-sdk/registration/late.hpp | 延迟注册传播机制 |

<!-- verified: 2026-05-27 -->

## 核心数据结构

### client_library（内部结构）
- **定义位置**: `source/lib/rocprofiler-sdk/registration.cpp:315`
- **职责**: 存储单个已注册工具客户端的完整信息
- **关键字段**:
  - `name` (std::string): 工具库的名称或路径
  - `dlhandle` (void*): 动态库句柄
  - `configure_func` (decltype(::rocprofiler_configure)*): 工具的配置函数指针
  - `configure_attach_func` (decltype(::rocprofiler_configure_attach)*): 工具的附加配置函数指针
  - `configure_result` (rocprofiler_tool_configure_result_t*): 配置结果，包含 initialize 和 finalize 回调
  - `configure_attach_result` (rocprofiler_tool_configure_attach_result_t*): 附加配置结果，包含 tool_attach 和 tool_detach 回调
  - `internal_client_id` (rocprofiler_client_id_t): 内部客户端标识符
  - `mutable_client_id` (rocprofiler_client_id_t): 可变客户端标识符（供工具使用）

<!-- verified: 2026-05-27 -->

### attach_status（内部结构）
- **定义位置**: `source/lib/rocprofiler-sdk/registration.cpp:264`
- **职责**: 跟踪 attach 功能的状态
- **关键字段**:
  - `has_attach_table` (bool): 是否已接收 attach API 表
  - `is_attached` (bool): 当前是否处于附加状态

### 状态管理
- **init_status**: 原子整数，值为 0 表示未初始化，-1 表示正在初始化，1 表示已初始化
- **fini_status**: 原子整数，值为 0 表示未终结，-1 表示正在终结，1 表示已终结

<!-- verified: 2026-05-27 -->

## 关键函数

### 公共 API

- **`rocprofiler_is_initialized`** (`registration.cpp:1027`)
  - 签名: `rocprofiler_status_t rocprofiler_is_initialized(int* status)`
  - 职责: 查询 SDK 是否已初始化

- **`rocprofiler_is_finalized`** (`registration.cpp:1034`)
  - 签名: `rocprofiler_status_t rocprofiler_is_finalized(int* status)`
  - 职责: 查询 SDK 是否已终结

- **`rocprofiler_force_configure`** (`registration.cpp:1041`)
  - 签名: `rocprofiler_status_t rocprofiler_force_configure(rocprofiler_configure_func_t configure_func)`
  - 职责: 强制使用指定的配置函数初始化 SDK（用于延迟加载场景）
  - 调用时机: 当工具无法通过正常机制被发现时

- **`rocprofiler_iterate_runtime_registration_info`** (`registration.cpp:1078`)
  - 签名: `rocprofiler_status_t rocprofiler_iterate_runtime_registration_info(rocprofiler_runtime_registration_info_cb_t callback, void* data)`
  - 职责: 迭代所有运行时注册信息

- **`rocprofiler_set_api_table`** (`registration.cpp:1096`)
  - 签名: `int rocprofiler_set_api_table(const char* name, uint64_t lib_version, uint64_t lib_instance, void** tables, uint64_t num_tables)`
  - 职责: 接收各运行时库的 API 表并进行分发处理
  - 调用时机: 各运行时库（HSA、HIP 等）初始化时通过 rocprofiler-register 调用

<!-- verified: 2026-05-27 -->

### 内部函数

- **`initialize`** (`registration.cpp:897`)
  - 签名: `void initialize()`
  - 职责: 执行 SDK 的完整初始化流程
  - 流程: 初始化日志 -> 设置注册库路径 -> 调用所有工具的 configure 函数 -> 调用所有工具的 initialize 函数 -> 初始化内部线程

- **`finalize`** (`registration.cpp:957`)
  - 签名: `void finalize()`
  - 职责: 执行 SDK 的完整终结流程
  - 流程: 同步异步复制 -> 终结设备计数 -> 终结队列控制器 -> 终结线程追踪 -> 终结 OMPT -> 终结 KFD -> 终结 PC 采样 -> 终结代码对象 -> 调用所有工具的 finalize 函数

- **`find_clients`** (`registration.cpp:343`)
  - 签名: `client_library_vec_t find_clients()`
  - 职责: 发现所有包含 `rocprofiler_configure` 符号的工具库
  - 搜索策略:
    1. 强制配置函数（如果有）
    2. ROCP_TOOL_LIBRARIES 环境变量指定的库
    3. 弱符号 `rocprofiler_configure`（如果存在）
    4. RTLD_DEFAULT 和 RTLD_NEXT 查找
    5. 遍历所有已加载的共享库

- **`invoke_client_configures`** (`registration.cpp:603`)
  - 职责: 调用所有已发现工具的 configure 函数
  - 返回值: 工具返回的 `rocprofiler_tool_configure_result_t` 包含 initialize 和 finalize 回调

- **`invoke_client_initializers`** (`registration.cpp:680`)
  - 职责: 调用所有工具的 initialize 回调
  - 特殊处理: 如果只有一个客户端，finalize 函数直接调用全局 finalize

- **`invoke_client_finalizers`** (`registration.cpp:711`)
  - 职责: 调用所有工具的 finalize 回调

- **`invoke_client_finalizer`** (`registration.cpp:808`)
  - 签名: `void invoke_client_finalizer(rocprofiler_client_id_t client_id)`
  - 职责: 终结指定客户端，包括停止其上下文、同步异步操作、调用 finalize 回调

- **`invoke_client_attaches`** (`registration.cpp:730`)
  - 职责: 调用所有已注册客户端的 tool_attach 函数

- **`invoke_client_detaches`** (`registration.cpp:768`)
  - 职责: 调用所有已注册客户端的 tool_detach 函数

- **`set_rocprofiler_register_library`** (`registration.cpp:200`)
  - 职责: 设置 ROCPROFILER_REGISTER_LIBRARY 环境变量，确保 rocprofiler-register 使用当前库实例

- **`get_this_library_path`** (`registration.cpp:158`)
  - 职责: 获取当前 rocprofiler-sdk 库的文件路径

<!-- verified: 2026-05-27 -->

### API 表分发函数（rocprofiler_set_api_table 内部）

- **HIP 处理** (`registration.cpp:1130-1158`):
  - 复制 HIP Runtime API 表
  - 安装 rocprofiler API 包装器
  - 通知运行时初始化
  - 安装 HIP stream 推断包装器
  - 通知拦截表注册

- **HSA 处理** (`registration.cpp:1185-1254`):
  - 复制所有 HSA API 表（core、amd_ext、image_ext、finalizer_ext、tools、pc_sampling_ext）
  - 初始化 aqlprofile 资源工厂
  - 构建 agent 缓存映射
  - 初始化线程追踪
  - 初始化队列控制器
  - 注册设备计数服务
  - 初始化异步复制和内存分配追踪
  - 初始化代码对象追踪
  - 安装所有 HSA API 包装器
  - 初始化 PC 采样服务

- **ROCTx 处理** (`registration.cpp:1256-1298`):
  - 复制 ROCTx 三组 API 表（core、ctrl、name）
  - 安装 rocprofiler API 包装器
  - 安装 range 包装器

- **RCCL 处理** (`registration.cpp:1299-1373`):
  - 验证 RCCL 分发表的 ABI 兼容性
  - 复制 RCCL API 表
  - 安装 rocprofiler API 包装器

- **rocdecode 处理** (`registration.cpp:1374-1397`):
  - 复制 rocdecode API 表
  - 安装 rocprofiler API 包装器

- **rocjpeg 处理** (`registration.cpp:1398-1420`):
  - 复制 rocjpeg API 表
  - 安装 rocprofiler API 包装器

- **rocattach 处理** (`registration.cpp:1421-1438`):
  - 转发 API 表到队列控制器和代码对象模块
  - 设置 attach 状态

<!-- verified: 2026-05-27 -->

## 调用关系

### 上游（谁调用了本模块）
- **rocprofiler-register**: 外部注册库，通过 `rocprofiler_set_api_table` 传递 API 表
- **工具库**: 通过 `rocprofiler_force_configure` 强制初始化
- **atexit 处理器**: 在程序退出时调用 `finalize`
- **context 模块**: 在初始化期间被调用以创建和管理上下文

### 下游（本模块调用了谁）
- **agent 模块**: 调用 `construct_agent_cache` 建立 HSA agent 映射
- **context 模块**: 调用 `push_client`/`pop_client`/`get_client_contexts`/`stop_client_contexts`/`deactivate_client_contexts`/`deregister_client_contexts`
- **hsa 模块**: 调用 `copy_table`/`update_table`/`queue_controller_init`/`async_copy_init`/`memory_allocation_init`
- **hip 模块**: 调用 `copy_table`/`update_table`/`stream::update_table`
- **marker 模块**: 调用 `copy_table`/`update_table`/`range::update_table`
- **rccl 模块**: 调用 `copy_table`/`update_table`
- **rocdecode 模块**: 调用 `copy_table`/`update_table`
- **rocjpeg 模块**: 调用 `copy_table`/`update_table`
- **intercept_table 模块**: 调用 `notify_intercept_table_registration`
- **code_object 模块**: 调用 `initialize`/`finalize`
- **counters 模块**: 调用 `device_counting_service_finalize`/`device_counting_service_hsa_registration`
- **pc_sampling 模块**: 调用 `code_object::initialize`/`service_sync`/`service_fini`/`post_hsa_init_start_active_service`
- **thread_trace 模块**: 调用 `initialize`/`finalize`
- **ompt 模块**: 调用 `finalize_ompt`
- **kfd 模块**: 调用 `finalize`
- **internal_threading 模块**: 调用 `initialize`/`finalize`
- **runtime_init 模块**: 调用 `initialize`
- **common 模块**: 调用 `init_logging`/`destroy_static_tl_objects`/`destroy_static_objects`

<!-- verified: 2026-05-27 -->

## 数据流

### 初始化流程
1. **库加载**: 当 `librocprofiler-sdk.so` 被加载时，`init_logging_at_load` 全局变量触发 `init_logging()`
2. **API 表注册**: rocprofiler-register 调用 `rocprofiler_set_api_table`，传入运行时库的 API 表
3. **首次调用触发初始化**: `rocprofiler_set_api_table` 内部通过 `std::call_once` 调用 `initialize()`
4. **工具发现**: `find_clients()` 通过多种策略发现包含 `rocprofiler_configure` 符号的工具库
5. **配置调用**: `invoke_client_configures()` 调用每个工具的 configure 函数，获取初始化/终结回调
6. **初始化调用**: `invoke_client_initializers()` 调用每个工具的 initialize 回调
7. **API 表处理**: 根据 API 表名称（hip/hsa/roctx/rccl 等）分发到对应的处理函数
8. **包装器安装**: 每个处理函数复制原始 API 表、安装追踪包装器、通知拦截表注册

### 终结流程
1. **触发**: `finalize()` 通过 `atexit` 或显式调用触发
2. **同步**: 同步所有异步操作（异步复制、队列控制器、PC 采样）
3. **子系统终结**: 按顺序终结各子系统（设备计数、线程追踪、OMPT、KFD、PC 采样、代码对象）
4. **客户端终结**: 调用所有客户端的 finalize 回调
5. **清理**: 销毁静态对象

### API 表分发流程
1. **接收**: `rocprofiler_set_api_table` 接收 API 表和库名称
2. **分发**: 根据名称（"hip"/"hsa"/"roctx" 等）路由到对应处理分支
3. **复制**: 调用 `xxx::copy_table` 保存原始函数指针（用于非追踪路径调用）
4. **安装**: 调用 `xxx::update_table` 替换 API 表中的函数指针为追踪包装器
5. **通知**: 调用 `intercept_table::notify_intercept_table_registration` 通知工具可以进一步修改 API 表

<!-- verified: 2026-05-27 -->

## 已知限制与边界情况

- **单次初始化**: 使用 `std::call_once` 确保初始化只执行一次，重复调用 `initialize()` 会被忽略
- **单次终结**: 使用 `std::atomic_flag` 确保终结只执行一次
- **状态检查**: 初始化和终结都有严格的状态检查，防止在错误的生命周期阶段执行操作
- **配置锁定**: 初始化完成后（init_status > -1），不允许再配置新的上下文或缓冲区
- **客户端去重**: `find_clients` 使用 `is_unique_configure_func` 确保同一配置函数不会被多次调用
- **随机客户端偏移**: 使用随机偏移量避免客户端 ID 冲突
- **HSA 工具兼容性**: 设置 `HSA_TOOLS_ROCPROFILER_V1_TOOLS=0` 以避免 HSA 运行时的重复注册 bug
- **RCCL ABI 验证**: 对 RCCL 分发表进行运行时 ABI 兼容性检查，不兼容时禁用追踪
- **线程安全**: 使用互斥锁保护客户端注册和终结操作
- **延迟加载支持**: `rocprofiler_force_configure` 支持在运行时强制初始化，并触发 API 表的重新传播
- **CODECOV 支持**: 在终结时调用 `__gcov_dump()` 收集代码覆盖率数据（如果启用）

<!-- verified: 2026-05-27 -->

## 与官方文档的关系

- API 参考: [rocprofiler-sdk API reference](../../../../projects/rocprofiler-sdk/source/docs/api-reference/rocprofiler-sdk_api_reference.rst)
- 工具库指南: [tool library](../../../../projects/rocprofiler-sdk/source/docs/api-reference/tool_library.rst)
- 进程附加: [process attachment](../../../../projects/rocprofiler-sdk/source/docs/api-reference/process_attachment.rst)
- 拦截表: [intercept table](../../../../projects/rocprofiler-sdk/source/docs/api-reference/intercept_table.rst)
- 头文件 Doxygen 注释: [registration.h](../../../../projects/rocprofiler-sdk/source/include/rocprofiler-sdk/registration.h) 中包含详细的 API 文档
