# Python 绑定

## 概述

Python 绑定模块为 rocprofiler-sdk 提供了 Python 语言接口，包含三个子包：`rocpd`（ROCPD 数据库操作）、`rocprofv3`（可用计数器查询）和 `roctx`（ROCTx 标记 API）。这些绑定通过 pybind11 将 C++ 实现暴露为 Python 模块，使用户能够在 Python 脚本中直接操作性能分析数据、查询可用计数器和使用 ROCTx 标记功能。所有三个子包都通过 CMake 构建系统安装到 Python site-packages 目录，并在构建时注入版本信息。

## 关键文件

### rocpd 子包
| 文件 | 路径 | 职责 |
|------|------|------|
| __init__.py | source/lib/python/rocpd/__init__.py | 包初始化，暴露公共 API（connect, execute, read_*, write_*） |
| __main__.py | source/lib/python/rocpd/__main__.py | CLI 入口，定义 convert/merge/package/query/summary/time-window 子命令 |
| libpyrocpd.cpp | source/lib/python/rocpd/libpyrocpd.cpp | C++ pybind11 绑定实现（约 700 行） |
| libpyrocpd.hpp | source/lib/python/rocpd/libpyrocpd.hpp | C++ 绑定头文件，引用 output/metadata、output/output_config 等内部模块 |
| importer.py | source/lib/python/rocpd/importer.py | RocpdImportData 类，数据库连接和多文件合并管理 |
| schema.py | source/lib/python/rocpd/schema.py | RocpdSchema 类，数据库 schema 管理 |
| query.py | source/lib/python/rocpd/query.py | SQL 查询执行和多格式导出 |
| merge.py | source/lib/python/rocpd/merge.py | SQLite 数据库合并 |
| package.py | source/lib/python/rocpd/package.py | .rpdb 文件夹打包和管理 |
| csv.py | source/lib/python/rocpd/csv.py | CSV 输出生成 |
| pftrace.py | source/lib/python/rocpd/pftrace.py | Perfetto trace 输出生成 |
| otf2.py | source/lib/python/rocpd/otf2.py | OTF2 格式输出生成 |
| summary.py | source/lib/python/rocpd/summary.py | 性能数据摘要生成 |
| time_window.py | source/lib/python/rocpd/time_window.py | 时间窗口裁剪 |
| output_config.py | source/lib/python/rocpd/output_config.py | 输出配置管理 |
| source/ | source/lib/python/rocpd/source/ | C++ 辅助源码（common.hpp, functions.hpp, interop.hpp, perfetto.hpp, sql_generator.hpp, types.hpp） |

<!-- verified: 2026-05-27 -->

### rocprofv3 子包
| 文件 | 路径 | 职责 |
|------|------|------|
| __init__.py | source/lib/python/rocprofv3/__init__.py | 包初始化（仅导入 absolute_import） |
| avail.py | source/lib/python/rocprofv3/avail.py | 可用计数器和 PC 采样配置查询 |
| CMakeLists.txt | source/lib/python/rocprofv3/CMakeLists.txt | 构建配置 |

### roctx 子包
| 文件 | 路径 | 职责 |
|------|------|------|
| __init__.py | source/lib/python/roctx/__init__.py | 包初始化，暴露 ROCTx API（mark, rangePush, rangePop 等） |
| libpyroctx.cpp | source/lib/python/roctx/libpyroctx.cpp | C++ pybind11 绑定实现 |
| libpyroctx.hpp | source/lib/python/roctx/libpyroctx.hpp | C++ 绑定头文件 |
| context_decorators.py | source/lib/python/roctx/context_decorators.py | 装饰器和上下文管理器（RoctxRange, RoctxProfiler） |
| CMakeLists.txt | source/lib/python/roctx/CMakeLists.txt | 构建配置 |

<!-- verified: 2026-05-27 -->

## 核心数据结构

### RocpdImportData（rocpd 子包）
- **定义位置**: `source/lib/python/rocpd/importer.py:57`
- **职责**: ROCPD 数据库连接管理器，继承自 C++ 绑定的 `libpyrocpd.RocpdImportData`
- **关键特性**:
  - 支持从单个或多个数据库文件创建连接
  - 自动 ATTACH 多个数据库并创建 TEMPORARY VIEW 实现跨库查询
  - 支持上下文管理器（`with` 语句）
  - 属性委托：未找到的属性自动委托给底层 `sqlite3.Connection`
- **关键属性**:
  - `connection`: 底层 SQLite 连接
  - `table_info`: 临时视图信息字典
- **构造参数**:
  - `input`: 数据库文件路径（str、list[str] 或 RocpdImportData）
  - `skip_auto_merge` (bool): 跳过自动合并
  - `automerge_limit` (int): 合并上限（默认 IDEAL_NUMBER_OF_DATABASE_FILES=1）
  - `dbname` (str): 输出数据库名（默认 `:memory:`）

<!-- verified: 2026-05-27 -->

### RocpdSchema（rocpd 子包）
- **定义位置**: `source/lib/python/rocpd/schema.py:37`
- **职责**: 管理 ROCPD 数据库的 DDL schema
- **关键属性**:
  - `tables`: 表创建脚本（包含 `PRAGMA foreign_keys`）
  - `indexes`: 索引创建脚本
  - `views`: 视图创建脚本（rocpd/data/summary/marker 四类视图）
- **Schema 来源**: 通过 `libpyrocpd.load_schema()` 从 C++ 端加载，支持 Jinja2 模板变量（uuid, guid）

### output_config（rocpd 子包）
- **定义位置**: `source/lib/python/rocpd/output_config.py:53`
- **职责**: 输出配置管理，继承自 `libpyrocpd.output_config`
- **关键属性**: output_path, output_file, agent_index_value, kernel_rename 等
- **agent_index_value 特殊处理**: `"absolute"` 映射到 `libpyrocpd.agent_indexing.node`，`"type-relative"` 映射到 `logical_node_type`

### counter（rocprofv3 子包）
- **定义位置**: `source/lib/python/rocprofv3/avail.py:66`
- **职责**: 表示一个性能计数器
- **关键属性**: name, counter_handle, description, dimensions, is_hw_constant

### dimension（rocprofv3 子包）
- **定义位置**: `source/lib/python/rocprofv3/avail.py:47`
- **职责**: 表示计数器的一个维度
- **关键属性**: id, name, instances

### RoctxRange（roctx 子包）
- **定义位置**: `source/lib/python/roctx/context_decorators.py:30`
- **职责**: 提供 ROCTx 范围标记的装饰器和上下文管理器
- **用法**:
  - 装饰器: `@RoctxRange("my_region")`
  - 上下文管理器: `with RoctxRange("my_region"):`

### RoctxProfiler（roctx 子包）
- **定义位置**: `source/lib/python/roctx/context_decorators.py:68`
- **职责**: 提供 ROCTx profiler pause/resume 的装饰器和上下文管理器
- **用法**:
  - 装饰器: `@RoctxProfiler(tid=0)`
  - 上下文管理器: `with RoctxProfiler(tid=0):`

<!-- verified: 2026-05-27 -->

## 关键函数

### rocpd 子包公共 API

#### connect(input, *args, **kwargs)
- **定义位置**: `source/lib/python/rocpd/__init__.py:80`
- **职责**: 创建 RocpdImportData 连接实例
- **等价于**: `RocpdImportData(input, *args, **kwargs)`

#### execute(data, *args, **kwargs)
- **定义位置**: `source/lib/python/rocpd/__init__.py:84`
- **职责**: 在数据连接上执行 SQL 查询

#### read_agents / read_nodes / read_processes / read_threads
- **定义位置**: `source/lib/python/rocpd/__init__.py:88-101`
- **职责**: 读取 agent/节点/进程/线程信息
- **底层实现**: 委托给 `libpyrocpd.read_*()` C++ 函数

#### write_perfetto / write_csv / write_otf2
- **定义位置**: `source/lib/python/rocpd/__init__.py:104-173`
- **职责**: 将数据写入 Perfetto/CSV/OTF2 格式
- **参数**: connection (RocpdImportData), config (output_config), **kwargs

#### execute_statement(conn, statement, is_script=False)
- **定义位置**: `source/lib/python/rocpd/importer.py:117`
- **职责**: 执行 SQL 语句，支持单条语句和脚本模式

<!-- verified: 2026-05-27 -->

### rocprofv3 子包

#### fatal_error(msg, exit_code=1)
- **定义位置**: `source/lib/python/rocprofv3/avail.py:31`
- **职责**: 输出错误信息到 stderr 并退出

#### build_counter_string(obj)
- **定义位置**: `source/lib/python/rocprofv3/avail.py:37`
- **职责**: 将计数器对象格式化为可读字符串（包含维度信息）

<!-- verified: 2026-05-27 -->

### roctx 子包公共 API

#### mark(msg)
- **定义位置**: `source/lib/python/roctx/__init__.py:60`
- **职责**: 在任意附加的 profiler 中标记一个事件
- **底层调用**: `libpyroctx.roctxMark(msg)` -> `roctxMarkA()`

#### rangePush(msg) / rangePop()
- **定义位置**: `source/lib/python/roctx/__init__.py:76-79`
- **职责**: 开始/结束一个嵌套范围标记
- **底层调用**: `roctxRangePushA()` / `roctxRangePop()`

#### rangeStart(msg) / rangeStop(id)
- **定义位置**: `source/lib/python/roctx/__init__.py:84-89`
- **职责**: 开始/结束一个进程范围标记
- **底层调用**: `roctxRangeStartA()` / `roctxRangeStop()`

#### profilerPause(tid=0) / profilerResume(tid=0)
- **定义位置**: `source/lib/python/roctx/__init__.py:64-69`
- **职责**: 暂停/恢复 profiler 数据收集
- **底层调用**: `roctxProfilerPause()` / `roctxProfilerResume()`

#### getThreadId()
- **定义位置**: `source/lib/python/roctx/__init__.py:72`
- **职责**: 获取当前线程 ID
- **底层调用**: `roctxGetThreadId()`

#### nameOsThread(name)
- **定义位置**: `source/lib/python/roctx/__init__.py:92`
- **职责**: 为当前 CPU OS 线程命名

#### nameHipDevice(name, device_id=0)
- **定义位置**: `source/lib/python/roctx/__init__.py:100`
- **职责**: 为 HIP 设备命名

<!-- verified: 2026-05-27 -->

## 调用关系

### 上游（谁调用了本模块）

#### rocpd 子包
- **rocpd CLI** (`source/bin/rocpd.py`): 通过 `python3 -m rocpd` 调用
- **rocprofv3.py**: 生成 ROCPD 数据库后，用户通过 rocpd 模块进行后处理
- **用户脚本**: `import rocpd; data = rocpd.connect("file.db"); rocpd.write_perfetto(data)`

#### rocprofv3 子包
- **rocprofv3-avail.py** (`source/bin/rocprofv3-avail.py`): 调用 avail 模块查询可用计数器

#### roctx 子包
- **用户应用程序**: `import roctx; roctx.rangePush("my_region")`
- **rocprofv3.py**: 通过 `LD_PRELOAD` 注入 `librocprofiler-sdk-roctx.so` 时使用

<!-- verified: 2026-05-27 -->

### 下游（本模块调用了谁）

#### rocpd 子包
- **libpyrocpd (C++ 绑定)**: 所有数据库底层操作
  - `libpyrocpd.connect()`: 创建数据库连接
  - `libpyrocpd.read_*()`: 读取数据（通过 cereal SQLite3InputArchive 反序列化）
  - `libpyrocpd.write_perfetto/csv()`: 写入输出格式
  - `libpyrocpd.load_schema()`: 加载 SQL schema
  - `libpyrocpd.format_path()`: 路径格式化
  - `libpyrocpd.output_config`: 输出配置类
  - `libpyrocpd.RocpdImportData`: C++ 端导入数据类
- **rocprofiler-sdk-rocpd** (C 库): `rocpd.h`, `sql.h` 提供底层 SQL schema 和数据库操作
- **SQLite3**: 通过 C++ 绑定间接使用

#### rocprofv3 子包
- **rocprofiler-sdk C++ 库**: 通过 ctypes 加载 `librocprofv3-list-avail.so`
- **rocprofiler-sdk/agent.h**: 查询 agent 信息

#### roctx 子包
- **libpyroctx (C++ 绑定)**: 所有 ROCTx API 调用
- **rocprofiler-sdk-roctx** (C 库): `roctx.h` 提供底层 ROCTx 功能
  - `roctxMarkA()`: 标记事件
  - `roctxRangePushA()` / `roctxRangePop()`: 嵌套范围
  - `roctxRangeStartA()` / `roctxRangeStop()`: 进程范围
  - `roctxProfilerPause()` / `roctxProfilerResume()`: 控制采集
  - `roctxGetThreadId()`: 获取线程 ID
  - `roctxNameOsThread()` / `roctxNameHipDevice()`: 命名

<!-- verified: 2026-05-27 -->

## 数据流

### rocpd 数据流
```
rocprofv3 输出 (.db 文件)
    |
    v
rocpd.connect(input)
    |
    |-- package.flatten_rocpd_yaml_input_file() 解析输入
    |-- libpyrocpd.connect() 创建 C++ RocpdImportData
    |-- ATTACH 多个数据库文件
    |-- _create_temp_views() 创建跨库统一视图
    |-- _create_meta_views() 创建元数据视图
    |
    v
RocpdImportData 实例
    |
    |-- execute() / read_*() 查询数据
    |-- write_perfetto/csv/otf2() 导出数据
    |-- summary.generate_all_summaries() 生成摘要
    |
    v
输出文件 (pftrace/csv/otf2/db)
```

### roctx 数据流
```
用户 Python 脚本
    |
    |-- roctx.rangePush("name") / roctx.rangePop()
    |-- roctx.mark("event")
    |-- roctx.profilerPause() / roctx.profilerResume()
    |
    v
libpyroctx (C++ pybind11 绑定)
    |
    v
rocprofiler-sdk-roctx (C 库)
    |
    v
rocprofiler-sdk 运行时
    |
    v
rocprofv3 工具收集标记数据
```

<!-- verified: 2026-05-27 -->

## 已知限制与边界情况

### rocpd 子包
1. **SQLite 符号冲突**: `__init__.py` 使用 `ctypes.CDLL("libsqlite3.so", mode=ctypes.RTLD_GLOBAL)` 预加载 SQLite 以避免混合库符号冲突
2. **Python 版本限制**: 仅支持 CMake 构建时指定的 Python 版本
3. **自动合并阈值**: 当输入数据库数量超过 `IDEAL_NUMBER_OF_DATABASE_FILES`（默认 1）时自动合并，最大支持 8 个
4. **UUID 表名匹配**: 多数据库合并时通过 UUID 后缀匹配表名，UUID 不匹配的表会被静默跳过
5. **输入类型检查**: `RocpdImportData` 不接受现有的 `sqlite3.Connection` 对象，仅接受文件路径

### rocprofv3 子包
1. **ctypes 加载**: 使用 `ctypes.CDLL` 加载共享库，需要库文件在 `LD_LIBRARY_PATH` 中
2. **计数器维度**: 计数器可能有多个维度，每个维度有不同的实例数

### roctx 子包
1. **LD_PRELOAD 依赖**: ROCTx 功能需要 `librocprofiler-sdk-roctx.so` 被 `LD_PRELOAD` 注入才能工作
2. **线程安全**: ROCTx API 是线程安全的，但 Python GIL 可能影响多线程场景下的性能
3. **已注释功能**: `roctxNameHsaAgent` 和 `roctxNameHipStream` 在 C++ 绑定中已注释，Python 端未暴露
4. **range ID 管理**: `rangeStart()` 返回的 ID 必须传递给 `rangeStop()`，否则范围不会被关闭

<!-- verified: 2026-05-27 -->

## 与官方文档的关系

| 文档 | 路径 | 说明 |
|------|------|------|
| 使用 rocpd 输出格式 | [using-rocpd-output-format.rst](../../../../projects/rocprofiler-sdk/source/docs/how-to/using-rocpd-output-format.rst) | rocpd 数据库使用指南 |
| 使用 ROCTx | [using-rocprofiler-sdk-roctx.rst](../../../../projects/rocprofiler-sdk/source/docs/how-to/using-rocprofiler-sdk-roctx.rst) | ROCTx 标记 API 使用指南 |
| rocprofv3-avail | [using-rocprofv3-avail.rst](../../../../projects/rocprofiler-sdk/source/docs/how-to/using-rocprofv3-avail.rst) | 可用计数器查询（rocprofv3 子包） |
| rocpd README | [README.md](../../../../projects/rocprofiler-sdk/source/lib/python/rocpd/README.md) | rocpd Python 模块详细文档 |
| API 参考 - 工具库 | [tool_library.rst](../../../../projects/rocprofiler-sdk/source/docs/api-reference/tool_library.rst) | 工具库 API 参考 |
