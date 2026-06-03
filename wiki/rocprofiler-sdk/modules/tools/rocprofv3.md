# rocprofv3 -- 主 Profiling CLI 工具

## 概述

rocprofv3 是 rocprofiler-sdk 的主要命令行性能分析工具，用于对 AMD GPU 应用程序进行追踪（tracing）和性能计数器（PMC）采集。它是一个 Python 脚本，通过设置环境变量和 LD_PRELOAD 机制将 `librocprofiler-sdk-tool.so` 和 `librocprofiler-sdk.so` 注入到目标应用程序中，从而在运行时拦截 HIP、HSA、KFD 等 API 调用并收集性能数据。rocprofv3 支持多种输出格式（rocpd/SQLite、csv、json、pftrace、otf2），支持 MPI 多进程环境、进程附加（attach）模式、PC 采样、高级线程追踪（ATT）以及多遍（multi-pass）计数器采集。

## 阅读入口

- 只想了解 CLI 脚本职责、关键函数和公共数据流，继续读本页。
- 想按某个 `rocprofv3` 选项追到环境变量、C++ config 字段和 SDK service，读 [rocprofv3 指令触发流程](rocprofv3-flows.md)。
- 想看底层模块内部实现，跳到 [HSA 拦截器](../interceptors/hsa.md)、[硬件计数器](../core/counters.md)、[PC 采样](../core/pc-sampling.md)、[AQLProfile](../companion/aqlprofile.md) 或 [输出格式化器](../companion/output.md)。

## 关键文件

| 文件 | 路径 | 职责 |
|------|------|------|
| rocprofv3.py | source/bin/rocprofv3.py | 主 CLI 脚本（约 2262 行），包含命令行解析、环境配置、应用启动逻辑 |
| rocprofv3-flows.md | wiki/rocprofiler-sdk/modules/tools/rocprofv3-flows.md | 按 CLI 指令整理的触发流程、环境变量映射和 C++ 服务配置链路 |
| rocpd.py | source/bin/rocpd.py | rocpd CLI 启动脚本，通过 `python3 -m` 调用 rocpd 模块 |
| rocprofv3-avail.py | source/bin/rocprofv3-avail.py | 可用计数器和 PC 采样配置查询工具 |
| rocprof-attach.py | source/bin/rocprof-attach.py | 进程附加模式的启动脚本 |
| tool.cpp | source/lib/rocprofiler-sdk-tool/tool.cpp | C++ 端工具库实现，包含 `rocprofv3_main`、`rocprofv3_signal` 等核心函数 |
| main.c | source/lib/rocprofiler-sdk-tool/main.c | 工具库入口，包含 `rocprofv3_libc_start_main` 拦截 |

<!-- verified: 2026-05-27 -->

## 核心数据结构

### dotdict
- **定义位置**: `source/bin/rocprofv3.py:46`
- **职责**: 继承自 `dict`，支持点号访问字典属性，用于将命令行参数和配置统一为可点号访问的对象
- **关键特性**: 递归将嵌套 dict 转换为 dotdict，支持序列化（`__getstate__`/`__setstate__`）

### CONST_VERSION_INFO
- **定义位置**: `source/bin/rocprofv3.py:33`
- **职责**: 存储版本信息的字典，在 CMake 构建时通过模板替换填充
- **关键字段**: version, git_revision, library_arch, system_name, system_processor, compiler_id, rocm_version

### booleanArgAction
- **定义位置**: `source/bin/rocprofv3.py:330`
- **职责**: 自定义 argparse Action，将字符串参数转换为布尔值，支持 `on/off/true/false/yes/no` 等形式

<!-- verified: 2026-05-27 -->

## 关键函数

### main(argv=None)
- **定义位置**: `source/bin/rocprofv3.py:2105`
- **职责**: 程序主入口点
- **调用时机**: 命令行执行 `rocprofv3` 时
- **实现逻辑**:
  1. 调用 `parse_arguments()` 解析命令行参数
  2. 处理 `--version` 和无参数情况
  3. 如果指定了 `-i/--input`，调用 `parse_input()` 解析输入文件
  4. 检测多遍（multi-pass）PMC 采集模式
  5. 单遍模式下调用 `run()` 执行一次应用；多遍模式下循环调用 `run()` 执行多次

<!-- verified: 2026-05-27 -->

### parse_arguments(args=None)
- **定义位置**: `source/bin/rocprofv3.py:335`
- **职责**: 使用 argparse 定义并解析所有命令行选项
- **返回**: `(parsed_args, app_args)` 元组
- **支持的选项组**:
  - I/O 选项：`-i`, `-o`, `-d`, `-f`, `--output-config`, `--log-level`, `-E`
  - 聚合追踪选项：`-r/--runtime-trace`, `-s/--sys-trace`
  - 基本追踪选项：`--hip-trace`, `--marker-trace`, `--kernel-trace`, `--memory-copy-trace`, `--memory-allocation-trace`, `--kfd-trace`, `--scratch-memory-trace`, `--hsa-trace`, `--rccl-trace`, `--kokkos-trace`, `--rocdecode-trace`, `--rocjpeg-trace`
  - 细粒度追踪选项：`--hip-runtime-trace`, `--hip-compiler-trace`, `--hsa-core-trace`, `--hsa-amd-trace` 等
  - 计数器采集选项：`--pmc`
  - PC 采样选项：`--pc-sampling-beta-enabled`, `--pc-sampling-unit`, `--pc-sampling-method`, `--pc-sampling-interval`
  - 后处理选项：`--stats`, `-S/--summary`, `-D/--summary-per-domain`
  - 内核命名选项：`-M/--mangled-kernels`, `-T/--truncate-kernels`, `--kernel-rename`
  - 过滤选项：`--profile-mpi-ranks`, `--kernel-include-regex`, `--kernel-exclude-regex`, `--collection-period`, `--selected-regions`
  - Perfetto 选项：`--perfetto-backend`, `--perfetto-buffer-size`, `--perfetto-buffer-fill-policy`
  - ATT 选项：`--advanced-thread-trace`, `--att-target-cu`, `--att-simd-select`, `--att-buffer-size` 等
  - 高级选项：`--preload`, `--rocm-root`, `--sdk-soversion`, `--pid/--attach`, `--attach-duration-msec`

<!-- verified: 2026-05-27 -->

### run(app_args, args, **kwargs)
- **定义位置**: `source/bin/rocprofv3.py:1300`
- **职责**: 配置环境变量并启动目标应用程序
- **参数**:
  - `app_args`: 目标应用程序的命令行参数列表
  - `args`: 解析后的 rocprofv3 参数（dotdict）
  - `use_execv` (bool): 是否使用 `os.execvpe` 替换进程（默认 True）
  - `pass_id` (int): 多遍模式下的遍次编号
- **实现逻辑**:
  1. 检测 MPI 环境（rank/size），设置 `ROCPROF_MPI_RANK_VAR` 和 `ROCPROF_MPI_SIZE_VAR`
  2. 根据 `--profile-mpi-ranks` 判断当前 rank 是否需要采集
  3. 解析并验证所有共享库路径（`librocprofiler-sdk-tool.so`, `librocprofiler-sdk.so`, `librocprofiler-sdk-roctx.so` 等）
  4. 设置 `LD_PRELOAD` 注入工具库
  5. 将所有追踪/计数器/过滤选项转换为 `ROCPROF_*` 环境变量
  6. 处理 `--list-avail`（调用 `rocprofv3-avail`）和 `--pid`（调用 `rocprof-attach`）模式
  7. 通过 `os.execvpe` 或 `subprocess.check_call` 启动目标应用

<!-- verified: 2026-05-27 -->

### get_mpi_rank_and_size(custom_rank_env, custom_size_env)
- **定义位置**: `source/bin/rocprofv3.py:127`
- **职责**: 从 MPI 环境变量检测当前进程的 rank 和 world size
- **支持的 MPI 实现**: PBS, SLURM, PMI, MVAPICH2, OpenMPI, Intel MPI
- **返回**: `(rank, size, rank_var, size_var)` 元组

### parse_rank_specification(rank_spec)
- **定义位置**: `source/bin/rocprofv3.py:172`
- **职责**: 解析 rank 范围字符串（如 `"0-3,8,10-15"`）为 rank 集合
- **返回**: `set` 类型的 rank 集合

### should_rank_provide_output(mpi_ranks_spec, custom_rank_env, custom_size_env)
- **定义位置**: `source/bin/rocprofv3.py:200`
- **职责**: 判断当前 MPI rank 是否应生成 profile/trace 输出
- **返回**: `bool`

### resolve_library_path(val, args, is_sdk_lib=True)
- **定义位置**: `source/bin/rocprofv3.py:241`
- **职责**: 解析共享库路径，处理符号链接、版本后缀（`.so.X` 和 `.so.X.Y.Z`）

### parse_input(input_file)
- **定义位置**: `source/bin/rocprofv3.py:1136`
- **职责**: 解析输入文件（支持 `.txt`, `.json`, `.yaml`, `.yml` 格式）
- **返回**: `list[dotdict]` 的作业配置列表

### get_args(cmd_args, inp_args, ...)
- **定义位置**: `source/bin/rocprofv3.py:1198`
- **职责**: 合并命令行参数和输入文件参数，处理冲突检测
- **参数**:
  - `filter`: 正则表达式列表，指定哪些参数冲突时应报错（而非警告）
  - `require_in_both`: 是否要求所有参数在两个来源中都存在

<!-- verified: 2026-05-27 -->

## 调用关系

### 上游（谁调用了本模块）
- **用户命令行**: `rocprofv3 [options] -- <application>` 直接调用
- **MPI 启动器**: `mpirun -n N rocprofv3 ...` 或 `srun rocprofv3 ...` 通过作业调度器调用
- **动态链接器 / glibc 启动入口**: `LD_PRELOAD` 注入 `librocprofiler-sdk-tool.so` 后，工具库的 `__libc_start_main` 包装函数进入 `rocprofv3_libc_start_main`，再把目标程序入口替换为 `rocprofv3_main`

### 下游（本模块调用了谁）
- **rocprofv3-avail** (rocprofv3-avail.py): 当 `--list-avail` 时调用，查询可用计数器和 PC 采样配置
- **rocprof-attach** (rocprof-attach.py): 当 `--pid` 时调用，执行进程附加
- **librocprofiler-sdk-tool.so**: 通过 `LD_PRELOAD` 注入，由 C++ 运行时加载
- **librocprofiler-sdk.so**: 核心 SDK 库，通过 `LD_PRELOAD` 注入
- **librocprofiler-sdk-roctx.so**: 当 `--marker-trace` 时通过 `LD_PRELOAD` 注入
- **os.execvpe / subprocess.check_call**: 启动目标应用程序

<!-- verified: 2026-05-27 -->

## 数据流

```
用户命令行
    |
    v
rocprofv3.py::main()
    |-- parse_arguments() 解析 CLI 参数
    |-- parse_input() 解析输入文件（可选）
    |-- get_args() 合并参数
    |
    v
rocprofv3.py::run()
    |-- get_mpi_rank_and_size() 检测 MPI 环境
    |-- should_rank_provide_output() 判断是否采集
    |-- resolve_library_path() 解析共享库路径
    |-- 设置 ROCPROF_* 环境变量（追踪选项、计数器、过滤器等）
    |-- LD_PRELOAD 注入 librocprofiler-sdk-tool.so + librocprofiler-sdk.so
    |
    v
os.execvpe() 启动目标应用
    |
    v
目标进程内（通过 LD_PRELOAD）:
    librocprofiler-sdk.so::rocprofiler_sdk_shlib_ctor()
        |-- registration::initialize()
        |-- 查找 ROCP_TOOL_LIBRARIES 中的 rocprofiler_configure()
        |-- 调用 tool_init()
    librocprofiler-sdk-tool.so::__libc_start_main()
        |-- rocprofv3_libc_start_main()
        |-- 保存真实应用 main
        |-- 用 rocprofv3_main 包装真实 main
    librocprofiler-sdk-tool.so::rocprofv3_main()
        |-- 确认/补齐 rocprofiler-sdk 初始化
        |-- 注册回调/缓冲区处理追踪事件
        |-- 收集 HIP/HSA/KFD/ROCTx API 调用数据
        |-- 收集内核调度、内存拷贝等事件
        |-- 收集 PMC 计数器数据（可选）
        |-- 收集 PC 采样数据（可选）
        |-- 收集 ATT 线程追踪数据（可选）
        |
        v
    输出文件生成:
        - rocpd (SQLite .db) 格式
        - csv 格式
        - json 格式
        - pftrace (Perfetto) 格式
        - otf2 格式
```

各 CLI 指令如何展开成环境变量、C++ config 字段和 SDK service，见 [rocprofv3 指令触发流程](rocprofv3-flows.md)。

<!-- verified: 2026-05-27 -->

## 已知限制与边界情况

1. **LD_PRELOAD 限制**: 依赖 `LD_PRELOAD` 机制注入工具库，不适用于静态链接的应用程序或不支持 `LD_PRELOAD` 的环境
2. **MPI rank 检测**: 自动检测支持常见 MPI 实现（OpenMPI, SLURM, PBS 等），但非标准 MPI 实现可能需要通过 `--mpi-world-rank-variable` 和 `--mpi-world-size-variable` 手动指定
3. **多遍 PMC 采集**: 多遍模式（多个 `--pmc` 标志）不兼容 `--pid`（附加模式）和 `--collection-period`
4. **ATT 与 PMC 互斥**: `--att-perfcounters` 和 `--att-activity` 不能与 `--pmc` 同时使用
5. **PC 采样 beta 状态**: PC 采样功能仍为 beta 版本，需要显式启用 `--pc-sampling-beta-enabled` 或设置 `ROCPROFILER_PC_SAMPLING_BETA_ENABLED=ON`
6. **YAML 依赖**: 解析 YAML 输入文件需要安装 `pyyaml` 包，否则会报错提示安装
7. **文本输入格式废弃**: `.txt` 格式的计数器输入文件已被废弃，将在未来版本中移除
8. **进程附加配置一致性**: 附加模式下，重新附加到同一 PID 时必须使用与首次附加相同的追踪选项
9. **信号处理**: 默认安装信号处理器，可通过 `--disable-signal-handlers` 禁用以使用应用自身的信号处理器

<!-- verified: 2026-05-27 -->

## 与官方文档的关系

| 文档 | 路径 | 说明 |
|------|------|------|
| 使用 rocprofv3 | [using-rocprofv3.rst](../../../../projects/rocprofiler-sdk/source/docs/how-to/using-rocprofv3.rst) | rocprofv3 基本使用指南 |
| rocprofv3 CLI 选项 | [rocprofv3-cli-options.rst](../../../../projects/rocprofiler-sdk/source/docs/quick-reference/rocprofv3-cli-options.rst) | CLI 选项快速参考 |
| 高级 rocprofv3 选项 | [advanced-rocprofv3-options.rst](../../../../projects/rocprofiler-sdk/source/docs/how-to/advanced-rocprofv3-options.rst) | 高级选项详细说明 |
| I/O 选项 | [rocprofv3-io-options.rst](../../../../projects/rocprofiler-sdk/source/docs/how-to/rocprofv3-io-options.rst) | 输入输出选项说明 |
| MPI 使用 | [using-rocprofv3-with-mpi.rst](../../../../projects/rocprofiler-sdk/source/docs/how-to/using-rocprofv3-with-mpi.rst) | MPI 环境使用指南 |
| 进程附加 | [using-rocprofv3-process-attachment.rst](../../../../projects/rocprofiler-sdk/source/docs/how-to/using-rocprofv3-process-attachment.rst) | 进程附加模式说明 |
| OpenMP 使用 | [using-rocprofv3-with-openmp.rst](../../../../projects/rocprofiler-sdk/source/docs/how-to/using-rocprofv3-with-openmp.rst) | OpenMP 环境使用指南 |
| 线程追踪 | [using-thread-trace.rst](../../../../projects/rocprofiler-sdk/source/docs/how-to/using-thread-trace.rst) | ATT 线程追踪使用指南 |
| PC 采样 | [using-pc-sampling.rst](../../../../projects/rocprofiler-sdk/source/docs/how-to/using-pc-sampling.rst) | PC 采样使用指南 |
| 内核命名与过滤 | [kernel-naming-filtering.rst](../../../../projects/rocprofiler-sdk/source/docs/how-to/kernel-naming-filtering.rst) | 内核命名和过滤选项说明 |
| rocprofv3-avail | [using-rocprofv3-avail.rst](../../../../projects/rocprofiler-sdk/source/docs/how-to/using-rocprofv3-avail.rst) | 可用计数器查询工具说明 |
| rocpd 输出格式 | [using-rocpd-output-format.rst](../../../../projects/rocprofiler-sdk/source/docs/how-to/using-rocpd-output-format.rst) | rocpd 输出格式说明 |
| 与旧版工具对比 | [comparing-with-legacy-tools.rst](../../../../projects/rocprofiler-sdk/source/docs/conceptual/comparing-with-legacy-tools.rst) | 与 rocprof v1/v2 的对比 |
