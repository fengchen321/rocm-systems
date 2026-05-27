# ROCprofiler-SDK 依赖关系

本文档描述 ROCprofiler-SDK 项目的内部模块间依赖关系和外部依赖列表。

---

## 内部模块间依赖关系

### 核心库依赖图

```
+=========================================================================+
|                    rocprofiler-sdk-shared-library                        |
|                    (最终输出: librocprofiler-sdk.so)                     |
+=========================================================================+
         |
         | 链接
         v
+=========================================================================+
|                 rocprofiler-sdk-object-library                           |
|                 (核心 SDK 目标库)                                         |
+=========================================================================+
         |
         | 链接 (PRIVATE)
         v
+------------------+  +------------------+  +------------------+  +-------+
| common-library   |  | amd-comgr        |  | aqlprofile       |  | drm   |
| (通用工具库)     |  | (代码对象管理)   |  | (AQL 性能计数器) |  | (GPU) |
+------------------+  +------------------+  +------------------+  +-------+
         |                    |                    |                   |
         v                    v                    v                   v
+------------------+  +------------------+  +------------------+  +-------+
| abseil           |  | ELFIO            |  | HSA Runtime      |  | libdrm|
| (日志/检查)      |  | (ELF 解析)       |  | (hsakmt)         |  |       |
+------------------+  +------------------+  +------------------+  +-------+

+------------------+  +------------------+
| dw (libdw)       |  | headers          |
| (DWARF 调试信息) |  | (公共头文件)     |
+------------------+  +------------------+
```

### SDK 内部子模块依赖

```
rocprofiler-sdk-object-library
    |
    +-- source/lib/rocprofiler-sdk/ (核心源文件)
    |   ├── agent.cpp              -> Agent 管理
    |   ├── buffer.cpp             -> Buffer 管理
    |   ├── buffer_tracing.cpp     -> Buffer Tracing 服务
    |   ├── callback_tracing.cpp   -> Callback Tracing 服务
    |   ├── context.cpp            -> Context 管理
    |   ├── counter_config.cpp     -> 计数器配置
    |   ├── counters.cpp           -> 计数器核心
    |   ├── device_counting_service.cpp -> 设备计数
    |   ├── dispatch_counting_service.cpp -> 派发计数
    |   ├── external_correlation.cpp -> 外部关联 ID
    |   ├── intercept_table.cpp    -> 拦截表管理
    |   ├── internal_threading.cpp -> 内部线程
    |   ├── ompt.cpp               -> OMPT 支持
    |   ├── pc_sampling.cpp        -> PC 采样
    |   ├── registration.cpp       -> 工具注册
    |   ├── rocprofiler.cpp        -> 主入口
    |   └── runtime_initialization.cpp -> 运行时初始化
    |
    +-- source/lib/rocprofiler-sdk/context/ (Context 子模块)
    |   ├── context.cpp            -> Context 实现
    |   ├── correlation_id.cpp     -> 关联 ID 管理
    |   └── domain.cpp             -> Domain 管理
    |
    +-- source/lib/rocprofiler-sdk/hsa/ (HSA 拦截子模块)
    |   ├── hsa.cpp                -> HSA API 拦截
    |   ├── agent_cache.cpp        -> Agent 缓存
    |   ├── async_copy.cpp         -> 异步拷贝追踪
    |   ├── memory_allocation.cpp  -> 内存分配追踪
    |   ├── queue.cpp              -> 队列管理
    |   ├── queue_controller.cpp   -> 队列控制器
    |   └── scratch_memory.cpp     -> Scratch 内存追踪
    |
    +-- source/lib/rocprofiler-sdk/hip/ (HIP 拦截子模块)
    |   ├── hip.cpp                -> HIP API 拦截
    |   └── stream.cpp             -> HIP Stream 追踪
    |
    +-- source/lib/rocprofiler-sdk/code_object/ (Code Object 子模块)
    |   └── code_object.cpp        -> 代码对象加载追踪
    |
    +-- source/lib/rocprofiler-sdk/kernel_dispatch/ (内核派发子模块)
    |   └── kernel_dispatch.cpp    -> 内核派发追踪
    |
    +-- source/lib/rocprofiler-sdk/counters/ (计数器子模块)
    |   ├── counters.cpp           -> 计数器核心
    |   ├── device_counting.cpp    -> 设备计数
    |   └── dispatch_counting.cpp  -> 派发计数
    |
    +-- source/lib/rocprofiler-sdk/aql/ (AQL Profile 子模块)
    |   └── aqlprofile.cpp         -> AQL Profile 集成
    |
    +-- source/lib/rocprofiler-sdk/pc_sampling/ (PC 采样子模块)
    |   ├── pc_sampling.cpp        -> PC 采样核心
    |   ├── service.cpp            -> PC 采样服务
    |   └── code_object.cpp        -> PC 采样代码对象
    |
    +-- source/lib/rocprofiler-sdk/marker/ (Marker 子模块)
    |   └── marker.cpp             -> ROCTx Marker 拦截
    |
    +-- source/lib/rocprofiler-sdk/kfd/ (KFD 子模块)
    |   └── kfd.cpp                -> KFD 事件追踪
    |
    +-- source/lib/rocprofiler-sdk/rccl/ (RCCL 子模块)
    |   └── rccl.cpp               -> RCCL 拦截
    |
    +-- source/lib/rocprofiler-sdk/rocdecode/ (rocDecode 子模块)
    |   └── rocdecode.cpp          -> rocDecode 拦截
    |
    +-- source/lib/rocprofiler-sdk/rocjpeg/ (rocJPEG 子模块)
    |   └── rocjpeg.cpp            -> rocJPEG 拦截
    |
    +-- source/lib/rocprofiler-sdk/tracing/ (追踪基础设施)
    |   ├── tracing.cpp            -> 追踪核心
    |   └── profiling_time.cpp     -> 时间戳管理
    |
    +-- source/lib/rocprofiler-sdk/details/ (内部细节)
    |
    +-- source/lib/rocprofiler-sdk/ompt/ (OMPT 子模块)
    |   └── ompt.cpp               -> OMPT 拦截
    |
    +-- source/lib/rocprofiler-sdk/registration/ (注册子模块)
    |   ├── registration.cpp       -> 注册核心
    |   ├── iterate.cpp            -> 迭代器
    |   └── late.cpp               -> 延迟启动
    |
    +-- source/lib/rocprofiler-sdk/spm/ (SPM 子模块)
    |   └── spm.cpp                -> SPM 计数器
    |
    +-- source/lib/rocprofiler-sdk/thread_trace/ (线程追踪子模块)
    |   └── thread_trace.cpp       -> 线程追踪
```

### 共享库额外依赖

```
rocprofiler-sdk-shared-library
    |
    +-- rocprofiler-sdk-object-library (所有核心功能)
    |
    +-- rocprofiler-sdk-cxx-filesystem (文件系统库)
    |
    +-- rocprofiler-sdk-dl (动态链接库)
    |
    +-- shared_library.cpp (共享库入口点)
```

---

## 外部依赖列表

### 必需依赖

| 依赖库 | 目录/包名 | 用途 | 链接目标 |
|--------|-----------|------|----------|
| **Abseil** | `external/abseil-cpp` | 日志、检查、信号处理 | `rocprofiler-sdk-abseil` |
| **fmt** | `external/fmt` | 格式化输出 | `rocprofiler-sdk-fmt` |
| **PTL** | `external/ptl` | 线程池 | `rocprofiler-sdk-ptl` |
| **cereal** | `external/cereal` | 序列化 | `rocprofiler-sdk-cereal` |
| **yaml-cpp** | `external/yaml-cpp` | YAML 解析 | `rocprofiler-sdk-yaml-cpp` |
| **nlohmann/json** | `external/json` | JSON 解析 | `rocprofiler-sdk-json` |
| **ELFIO** | `external/elfio` | ELF 文件解析 | `rocprofiler-sdk-elfio` |
| **Perfetto** | `external/perfetto` | Perfetto 追踪格式输出 | `rocprofiler-sdk-perfetto` |
| **OTF2** | `external/otf2` | OTF2 追踪格式输出 | `rocprofiler-sdk-otf2` |
| **GOTCHA** | `external/gotcha` | 函数包装/拦截 | `rocprofiler-sdk-gotcha` |
| **SQLite3** | `external/sqlite` | 数据库输出 | `rocprofiler-sdk-sqlite3` |
| **libdw** | 系统库 | DWARF 调试信息解析 | `rocprofiler-sdk-dw` |
| **libdrm** | 系统库 | GPU 设备访问 | `rocprofiler-sdk-drm` |
| **AMD COMGR** | 系统库 | AMD 代码对象管理 | `rocprofiler-sdk-amd-comgr` |
| **AQL Profile** | 系统库 | 硬件性能计数器 | `rocprofiler-sdk-aqlprofile` |

### 可选依赖

| 依赖库 | 目录/包名 | 用途 | 条件 |
|--------|-----------|------|------|
| **GoogleTest** | `external/googletest` | 单元测试 | `ROCPROFILER_BUILD_TESTS=ON` |
| **pybind11** | `external/pybind11` | Python 绑定 | `ROCPROFILER_BUILD_PYBIND11=ON` |
| **doxygen-awesome-css** | `external/doxygen-awesome-css` | 文档样式 | `ROCPROFILER_BUILD_DOCS=ON` |
| **GHC Filesystem** | `external/filesystem` | C++17 文件系统回退 | `ROCPROFILER_BUILD_GHC_FS=ON` |

---

## 外部依赖详细说明

### 1. Abseil (abseil-cpp)

- **仓库**: `https://github.com/abseil/abseil-cpp.git`
- **版本**: `lts_2026_01_07`
- **用途**:
  - `absl::log` / `absl::log_initialize`: 日志系统
  - `absl::check`: 断言检查
  - `absl::log_globals`: 日志全局配置
  - `absl::vlog_is_on`: 详细日志控制
  - `absl::failure_signal_handler`: 信号处理

### 2. fmt

- **仓库**: `https://github.com/fmtlib/fmt.git`
- **版本**: `master`
- **用途**: 字符串格式化，替代 `sprintf` 等不安全函数

### 3. PTL (Parallel Task Library)

- **仓库**: `https://github.com/jrmadsen/PTL.git`
- **版本**: `rocprofiler` 分支
- **用途**: 线程池实现，用于后台数据处理
- **配置**: 禁用 TBB，禁用 GPU，启用锁

### 4. cereal

- **仓库**: `https://github.com/jrmadsen/cereal.git`
- **版本**: `rocprofiler` 分支
- **用途**: 数据序列化，用于配置文件和数据持久化
- **配置**: 启用线程安全 (`CEREAL_THREAD_SAFE=1`)

### 5. yaml-cpp

- **仓库**: `https://github.com/jbeder/yaml-cpp.git`
- **版本**: `master`
- **用途**: YAML 配置文件解析，用于计数器配置等
- **最低版本**: 0.8.0

### 6. nlohmann/json

- **仓库**: `https://github.com/nlohmann/json.git`
- **版本**: `develop`
- **用途**: JSON 数据处理，用于配置和输出

### 7. ELFIO

- **仓库**: `https://github.com/serge1/ELFIO.git`
- **版本**: `Release_3.12`
- **用途**: ELF 文件解析，用于代码对象加载追踪

### 8. Perfetto

- **仓库**: `https://github.com/google/perfetto`
- **版本**: `v44.0`
- **用途**: 生成 Perfetto 格式的追踪数据 (.pftrace)
- **构建**: 编译为静态库 `rocprofiler-sdk-perfetto`

### 9. OTF2

- **目录**: `external/otf2`
- **用途**: 生成 OTF2 格式的追踪数据，适合大规模追踪可视化

### 10. GOTCHA

- **仓库**: `https://github.com/jrmadsen/GOTCHA.git`
- **版本**: `rocprofiler` 分支
- **用途**: 函数包装/拦截框架，用于 hook 运行时 API

### 11. SQLite3

- **仓库**: `https://github.com/sqlite/sqlite`
- **版本**: `version-3.47.0`
- **用途**: 数据库输出格式，用于结构化数据存储

### 12. GoogleTest (可选)

- **仓库**: `https://github.com/google/googletest.git`
- **版本**: `main`
- **用途**: 单元测试框架
- **配置**: 启用 GMock

### 13. pybind11 (可选)

- **仓库**: `https://github.com/pybind/pybind11.git`
- **版本**: `v2.9.2`
- **用途**: Python 绑定，用于 Python 工具集成

---

## 系统依赖

| 依赖 | 用途 | 查找方式 |
|------|------|----------|
| **HSA Runtime** | GPU 内核调度、内存管理 | 系统安装 |
| **hsakmt** | KFD 内核接口 | 系统安装 |
| **AMD COMGR** | 代码对象管理 | `find_package(AMD_COMGR)` |
| **AQL Profile** | 硬件性能计数器采集 | 系统安装 |
| **libdrm** | GPU 设备访问 | 系统安装 |
| **libdw** | DWARF 调试信息 | `Findlibdw.cmake` |
| **libelf** | ELF 文件读取 | `Findlibelf.cmake` |
| **libdl** | 动态链接 | 系统标准库 |
| **rocDecode** (可选) | 视频解码库 | `FindrocDecode.cmake` |
| **rocJPEG** (可选) | 图像处理库 | `FindrocJPEG.cmake` |

---

## 依赖关系图（文字版）

```
                           用户工具
                              |
                              v
                    +-------------------+
                    | rocprofiler-sdk   |
                    | (共享库)          |
                    +-------------------+
                              |
         +--------------------+--------------------+
         |                    |                    |
         v                    v                    v
+----------------+  +----------------+  +----------------+
| 核心 SDK       |  | 公共头文件     |  | Python 绑定    |
| (object lib)   |  | (headers)      |  | (pybind11)     |
+----------------+  +----------------+  +----------------+
         |
    +----+----+----+----+----+----+----+----+----+----+
    |    |    |    |    |    |    |    |    |    |    |
    v    v    v    v    v    v    v    v    v    v    v
+------+ +--+ +--+ +--+ +--+ +--+ +--+ +--+ +--+ +--+
|common| |ab| |fm| |PT| |ce| |ya| |js| |EL| |pe| |OT|
|lib   | |se| |t | |L | |re| |ml| |on| |FI| |rf| |F2|
|      | |il| |  | |  | |al| |-c| |  | |O | |et| |  |
+------+ +--+ +--+ +--+ +--+ |pp| +--+ +--+ |to| +--+
                              +--+           +--+
    |    |    |    |    |    |    |    |    |    |
    v    v    v    v    v    v    v    v    v    v
+------+ +--+ +--+ +--+ +--+ +--+ +--+ +--+ +--+ +--+
|libdw | |li| |AM| |AQ| |GO| |SQ| |rc| |rc| |HS| |hs|
|      | |bd| |D | |L | |TC| |Li| |oD| |oJ| |A | |ak|
|      | |rm| |CO| |Pr| |HA| |te| |ec| |PE| |Ru| |mt|
|      | |  | |MG| |of| |  | |3 | |od| |G | |nt| |  |
|      | |  | |R | |il| |  | |  | |e | |  | |im| |  |
|      | |  | |  | |e | |  | |  | |  | |  | |e | |  |
+------+ +--+ +--+ +--+ +--+ +--+ +--+ +--+ +--+ +--+
```

---

## CMake 目标依赖总结

### PUBLIC 依赖 (传递给使用者)

- `rocprofiler-sdk::rocprofiler-sdk-headers`: 公共头文件

### PRIVATE 依赖 (仅内部使用)

- `rocprofiler-sdk::rocprofiler-sdk-build-flags`: 编译标志
- `rocprofiler-sdk::rocprofiler-sdk-memcheck`: 内存检查
- `rocprofiler-sdk::rocprofiler-sdk-common-library`: 通用工具库
- `rocprofiler-sdk::rocprofiler-sdk-amd-comgr`: AMD COMGR
- `rocprofiler-sdk::rocprofiler-sdk-aqlprofile`: AQL Profile
- `rocprofiler-sdk::rocprofiler-sdk-drm`: DRM
- `rocprofiler-sdk::rocprofiler-sdk-dw`: libdw
- `rocprofiler-sdk::rocprofiler-sdk-cxx-filesystem`: 文件系统 (仅共享库)
- `rocprofiler-sdk::rocprofiler-sdk-dl`: 动态链接 (仅共享库)

---

## 构建选项对依赖的影响

| CMake 选项 | 影响的依赖 | 默认值 |
|------------|-----------|--------|
| `ROCPROFILER_BUILD_TESTS` | GoogleTest | OFF |
| `ROCPROFILER_BUILD_SAMPLES` | 无额外依赖 | OFF |
| `ROCPROFILER_BUILD_DOCS` | doxygen-awesome-css | OFF |
| `ROCPROFILER_BUILD_PYBIND11` | pybind11 | ON |
| `ROCPROFILER_BUILD_GHC_FS` | GHC Filesystem | OFF |
| `ROCPROFILER_BUILD_ABSEIL` | 从源码构建 Abseil | ON |
| `ROCPROFILER_BUILD_FMT` | 从源码构建 fmt | ON |
| `ROCPROFILER_BUILD_YAML_CPP` | 从源码构建 yaml-cpp | ON |
| `ROCPROFILER_BUILD_SQLITE3` | 从源码构建 SQLite3 | ON |
| `ROCPROFILER_BUILD_GOTCHA` | 从源码构建 GOTCHA | ON |
| `ROCPROFILER_BUILD_AQLPROFILE` | 启用 AQL Profile 支持 | ON |
