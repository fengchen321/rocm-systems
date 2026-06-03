# rocpd -- ROCPD 数据库查询 CLI 工具

## 概述

rocpd 是 rocprofiler-sdk 的数据库查询和转换命令行工具，用于操作由 rocprofv3 生成的 ROCPD 格式（SQLite）性能数据文件。它提供了数据库合并（merge）、格式转换（convert）、打包（package）、SQL 查询（query）、时间窗口裁剪（time-window）和摘要生成（summary）等子命令。rocpd 的启动脚本 `source/bin/rocpd.py` 是一个薄包装器，通过 `python3 -m rocpd` 调用实际的 Python 模块 `source/lib/python/rocpd/__main__.py`。核心数据访问通过 C++ pybind11 绑定库 `libpyrocpd` 实现，提供高性能的 SQLite 数据库读写和格式转换能力。

## 关键文件

| 文件 | 路径 | 职责 |
|------|------|------|
| rocpd.py | source/bin/rocpd.py | CLI 启动脚本（薄包装器），通过 `python3 -m` 调用 rocpd 模块 |
| __main__.py | source/lib/python/rocpd/__main__.py | CLI 主入口，定义所有子命令（convert, merge, package, query, summary, time-window） |
| __init__.py | source/lib/python/rocpd/__init__.py | 包初始化，暴露公共 API（connect, execute, read_agents, write_perfetto 等） |
| importer.py | source/lib/python/rocpd/importer.py | RocpdImportData 类，管理数据库连接和多数据库合并视图 |
| libpyrocpd.cpp | source/lib/python/rocpd/libpyrocpd.cpp | C++ pybind11 绑定实现，提供高性能数据库操作 |
| libpyrocpd.hpp | source/lib/python/rocpd/libpyrocpd.hpp | C++ 绑定头文件 |
| schema.py | source/lib/python/rocpd/schema.py | RocpdSchema 类，管理 ROCPD 数据库 schema（表、索引、视图） |
| query.py | source/lib/python/rocpd/query.py | SQL 查询执行和结果导出（CSV, HTML, Markdown, PDF, Dashboard） |
| merge.py | source/lib/python/rocpd/merge.py | SQLite 数据库合并功能 |
| package.py | source/lib/python/rocpd/package.py | 数据库打包和 .rpdb 文件夹管理 |
| csv.py | source/lib/python/rocpd/csv.py | CSV 格式输出生成 |
| pftrace.py | source/lib/python/rocpd/pftrace.py | Perfetto trace 格式输出生成 |
| otf2.py | source/lib/python/rocpd/otf2.py | OTF2 格式输出生成 |
| summary.py | source/lib/python/rocpd/summary.py | 性能数据摘要/统计生成 |
| time_window.py | source/lib/python/rocpd/time_window.py | 基于时间窗口的数据裁剪 |
| output_config.py | source/lib/python/rocpd/output_config.py | 输出配置管理类 |

<!-- verified: 2026-05-27 -->

## 核心数据结构

### RocpdImportData
- **定义位置**: `source/lib/python/rocpd/importer.py:57`（Python 类），`source/lib/python/rocpd/libpyrocpd.cpp:133`（C++ 绑定）
- **职责**: 管理 ROCPD 数据库连接，支持从多个数据库文件创建统一视图
- **关键特性**:
  - 继承自 C++ 绑定的 `libpyrocpd.RocpdImportData`
  - 支持上下文管理器（`with` 语句）
  - 自动将多个数据库文件 ATTACH 并创建 TEMPORARY VIEW 实现跨库查询
  - 支持 `.rpdb` 文件夹、YAML 索引文件、通配符等多种输入格式
- **构造参数**:
  - `input`: 数据库文件路径（字符串、字符串列表或 RocpdImportData 实例）
  - `skip_auto_merge` (bool): 是否跳过自动合并
  - `automerge_limit` (int): 自动合并的数据库数量上限（默认 1，最大 8）
  - `dbname` (str): 输出数据库名称（默认 `:memory:`）

<!-- verified: 2026-05-27 -->

### RocpdSchema
- **定义位置**: `source/lib/python/rocpd/schema.py:37`
- **职责**: 管理 ROCPD 数据库的 schema 定义
- **关键属性**:
  - `tables`: 表创建 SQL 脚本
  - `indexes`: 索引创建 SQL 脚本
  - `views`: 视图创建 SQL 脚本（rocpd, data, summary, marker 视图）
- **关键方法**:
  - `write_schema(connection)`: 在给定连接上执行所有 schema 脚本
  - `load_schema(engine, kind, options, variables)`: 静态方法，从 C++ 端加载 schema 模板

<!-- verified: 2026-05-27 -->

### output_config
- **定义位置**: `source/lib/python/rocpd/output_config.py:53`
- **职责**: 输出配置管理，继承自 C++ 绑定的 `libpyrocpd.output_config`
- **关键属性**: output_path, output_file, agent_index_value, kernel_rename 等
- **关键方法**: `update(**kwargs)`: 批量更新配置属性

### rocpd_db（C++ 结构体）
- **定义位置**: `source/lib/output/generateRocpd.cpp:124`
- **职责**: C++ 端的 ROCPD 数据库生成器，负责将追踪数据写入 SQLite 数据库

<!-- verified: 2026-05-27 -->

## 关键函数

### CLI 子命令入口（__main__.py）

#### main(argv=None, config=None)
- **定义位置**: `source/lib/python/rocpd/__main__.py:33`
- **职责**: CLI 主入口，解析子命令并分发到对应处理函数
- **支持的子命令**:
  - `convert`: 数据库格式转换（csv, pftrace, otf2）
  - `merge`: 合并多个数据库文件
  - `package`: 打包数据库到 .rpdb 文件夹
  - `query`: 执行 SQL 查询并导出结果
  - `summary`: 生成性能数据摘要
  - `time-window`: 基于时间窗口裁剪数据

<!-- verified: 2026-05-27 -->

### 公共 API（__init__.py）

#### connect(input, *args, **kwargs)
- **定义位置**: `source/lib/python/rocpd/__init__.py:80`
- **职责**: 创建 RocpdImportData 连接实例
- **返回**: `RocpdImportData` 对象

#### execute(data, *args, **kwargs)
- **定义位置**: `source/lib/python/rocpd/__init__.py:84`
- **职责**: 在数据连接上执行 SQL 查询

#### read_agents(data, condition="")
- **定义位置**: `source/lib/python/rocpd/__init__.py:88`
- **职责**: 读取 agent 信息（委托给 `libpyrocpd.read_agents`）

#### read_nodes(data, condition="") / read_processes(data, condition="") / read_threads(data, condition="")
- **定义位置**: `source/lib/python/rocpd/__init__.py:92-101`
- **职责**: 读取节点/进程/线程信息

#### write_perfetto(connection, config=None, **kwargs)
- **定义位置**: `source/lib/python/rocpd/__init__.py:104`
- **职责**: 将数据写入 Perfetto pftrace 格式

#### write_csv(connection, config=None, **kwargs)
- **定义位置**: `source/lib/python/rocpd/__init__.py:128`
- **职责**: 将数据写入 CSV 格式

#### write_otf2(connection, config=None, **kwargs)
- **定义位置**: `source/lib/python/rocpd/__init__.py:152`
- **职责**: 将数据写入 OTF2 格式

<!-- verified: 2026-05-27 -->

### 内部函数

#### merge_sqlite_dbs(sources, dest_path, on_log=None)
- **定义位置**: `source/lib/python/rocpd/merge.py:34`
- **职责**: 将多个 SQLite 数据库合并到一个目标数据库
- **实现**: 逐个 ATTACH 源数据库，复制表数据到目标数据库

#### export_sqlite_query(conn, query, params, export_format, export_path, ...)
- **定义位置**: `source/lib/python/rocpd/query.py:46`
- **职责**: 执行 SQL 查询并导出结果
- **支持格式**: csv, html, md (markdown), pdf, dashboard, clipboard, console

#### flatten_rocpd_yaml_input_file(input, **kwargs)
- **定义位置**: `source/lib/python/rocpd/package.py:65`
- **职责**: 处理多种输入格式（YAML、.rpdb 文件夹、直接数据库文件、通配符），返回扁平化的数据库文件列表

#### prepare_output_folder(output_path, consolidate)
- **定义位置**: `source/lib/python/rocpd/package.py:39`
- **职责**: 准备输出文件夹路径，自动添加 `.rpdb` 扩展名

<!-- verified: 2026-05-27 -->

## 调用关系

### 上游（谁调用了本模块）
- **用户命令行**: `rocpd convert/merge/package/query/summary/time-window` 直接调用
- **rocprofv3.py**: 通过 `--output-format rocpd` 生成 ROCPD 数据库后，用户使用 rocpd 进行后续处理
- **Python 脚本**: `import rocpd; rocpd.connect(...)` 作为库使用

### 下游（本模块调用了谁）
- **libpyrocpd (C++ 绑定)**: 所有数据库底层操作（连接、查询、写入、schema 加载）
- **libsqlite3**: SQLite 数据库引擎（通过 C++ 绑定间接调用）
- **rocprofiler-sdk-rocpd**: C 端 ROCPD 库，提供 SQL schema 定义和数据库操作原语
- **pandas**: 查询导出时使用 pandas 读取 SQL 查询结果
- **otf2**: OTF2 格式输出时使用 Python OTF2 库
- **Jinja2**: Dashboard 模板渲染（可选依赖）

<!-- verified: 2026-05-27 -->

## 数据流

```
rocprofv3 输出 (.db 文件)
    |
    v
rocpd CLI (__main__.py)
    |
    |-- connect(input) --> RocpdImportData
    |       |
    |       |-- package.flatten_rocpd_yaml_input_file() 解析输入
    |       |-- libpyrocpd.connect() 创建 SQLite 连接
    |       |-- _create_temp_views() ATTACH 多个数据库并创建统一视图
    |       |-- _create_meta_views() 创建元数据视图
    |
    v
子命令处理:
    |
    |-- convert:
    |   |-- csv.write_csv() --> CSV 文件
    |   |-- pftrace.write_pftrace() --> .pftrace 文件（通过 libpyrocpd.write_perfetto）
    |   |-- otf2.write_otf2() --> OTF2 文件
    |
    |-- merge:
    |   |-- merge.merge_sqlite_dbs() --> 合并后的 .db 文件
    |
    |-- package:
    |   |-- package.prepare_output_folder() --> .rpdb 文件夹
    |   |-- 生成 index.yaml 索引文件
    |
    |-- query:
    |   |-- query.export_sqlite_query() --> CSV/HTML/MD/PDF/Dashboard
    |
    |-- summary:
    |   |-- summary.generate_all_summaries() --> 摘要输出
    |
    |-- time-window:
        |-- time_window.apply_time_window() --> 裁剪后的 .db 文件
```

<!-- verified: 2026-05-27 -->

## 已知限制与边界情况

1. **Python 版本限制**: rocpd 仅支持 CMake 构建时指定的 Python 版本，运行时会检查版本兼容性
2. **SQLite 版本兼容性**: `check_function_availability()` 包含对旧版 SQLite（如 RHEL 8）的回退处理
3. **自动合并限制**: 默认自动合并到 1 个数据库文件，最大支持 8 个（`IDEAL_NUMBER_OF_DATABASE_FILES = 1`, `MAX_LIMIT_OF_DATABASE_FILES = 8`）
4. **RTLD_GLOBAL 加载**: 为避免 SQLite 符号冲突，`__init__.py` 中使用 `ctypes.CDLL("libsqlite3.so", mode=ctypes.RTLD_GLOBAL)` 预加载 SQLite
5. **输入验证**: 通过检查文件头 16 字节（`SQLite format 3\x00`）验证输入是否为有效的 SQLite 数据库
6. **多数据库视图**: 多个数据库合并时通过 UUID 后缀匹配表名创建统一视图，UUID 不匹配的表会被跳过
7. **OTF2 依赖**: OTF2 输出格式需要安装 Python `otf2` 包
8. **Dashboard 依赖**: Dashboard 输出格式需要安装 `jinja2` 和 `pandas`

<!-- verified: 2026-05-27 -->

## 与官方文档的关系

| 文档 | 路径 | 说明 |
|------|------|------|
| 使用 rocpd 输出格式 | [using-rocpd-output-format.rst](../../../../projects/rocprofiler-sdk/source/docs/how-to/using-rocpd-output-format.rst) | rocpd 输出格式使用指南 |
| rocprofv3 I/O 选项 | [rocprofv3-io-options.rst](../../../../projects/rocprofiler-sdk/source/docs/how-to/rocprofv3-io-options.rst) | 包含 rocpd 输出选项说明 |
| rocpd README | [README.md](../../../../projects/rocprofiler-sdk/source/lib/python/rocpd/README.md) | rocpd Python 模块的详细使用文档 |
