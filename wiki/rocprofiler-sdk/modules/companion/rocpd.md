# ROCPD 输出格式 (ROCm Profiling Data)

## 概述

ROCPD 模块 (`source/lib/rocprofiler-sdk-rocpd/`) 是 rocprofiler-sdk 的 SQLite3 数据库存储层，提供 ROCm Profiling Data (ROCPD) 格式的实现。该模块作为 `rocprofv3` 的默认输出格式，将所有性能分析数据（执行追踪、性能计数器、硬件指标和上下文元数据）存储在符合 ACID 标准的 SQLite3 数据库中。它定义了 SQL schema 加载机制和状态管理接口，是输出格式化器模块 (`source/lib/output/`) 的底层依赖。

## 关键文件

| 文件 | 路径 | 职责 |
|------|------|------|
| `rocpd.cpp` | `source/lib/rocprofiler-sdk-rocpd/rocpd.cpp` | ROCPD 核心实现：版本查询、状态字符串管理 |
| `sql.cpp` | `source/lib/rocprofiler-sdk-rocpd/sql.cpp` | SQL schema 加载和 Jinja 变量替换实现 |
| `CMakeLists.txt` | `source/lib/rocprofiler-sdk-rocpd/CMakeLists.txt` | 构建配置，生成 `librocprofiler-sdk-rocpd.so` 共享库 |

## 核心数据结构

- **`rocpd_status_t`** (`rocpd.h`): 状态码枚举，定义所有 ROCPD 操作的返回状态：
  - `ROCPD_STATUS_SUCCESS` (0): 操作成功
  - `ROCPD_STATUS_ERROR`: 通用错误
  - `ROCPD_STATUS_ERROR_INVALID_ARGUMENT`: 无效参数
  - `ROCPD_STATUS_ERROR_SQL_ERROR`: 通用 SQL 错误
  - `ROCPD_STATUS_ERROR_SQL_INVALID_ENGINE`: 无效 SQL 引擎
  - `ROCPD_STATUS_ERROR_SQL_INVALID_SCHEMA_KIND`: 无效 schema 类型
  - `ROCPD_STATUS_ERROR_SQL_SCHEMA_NOT_FOUND`: schema 文件未找到
  - `ROCPD_STATUS_ERROR_SQL_SCHEMA_PERMISSION_DENIED`: schema 文件无法读取<!-- verified: 2026-05-27 -->

- **`status_string<Idx>`** (`rocpd.cpp`): 模板结构体，通过 `ROCPD_STATUS_STRING` 宏为每个状态码定义名称和描述字符串。使用编译期模板特化实现状态码到字符串的映射。<!-- verified: 2026-05-27 -->

- **`rocpd_sql_engine_t`** (`sql.h`): SQL 引擎枚举：
  - `ROCPD_SQL_ENGINE_SQLITE3`: SQLite3 引擎（目前唯一支持的引擎）
  - `ROCPD_SQL_ENGINE_NONE`: 无引擎
  - `ROCPD_SQL_ENGINE_LAST`: 哨兵值<!-- verified: 2026-05-27 -->

- **`rocpd_sql_schema_kind_t`** (`sql.h`): Schema 类型枚举：
  - `ROCPD_SQL_SCHEMA_ROCPD_TABLES`: 表定义 (`rocpd_tables.sql`)
  - `ROCPD_SQL_SCHEMA_ROCPD_INDEXES`: 索引定义 (`rocpd_indexes.sql`)
  - `ROCPD_SQL_SCHEMA_ROCPD_VIEWS`: 视图定义 (`rocpd_views.sql`)
  - `ROCPD_SQL_SCHEMA_ROCPD_DATA_VIEWS`: 数据视图 (`data_views.sql`)
  - `ROCPD_SQL_SCHEMA_ROCPD_SUMMARY_VIEWS`: 摘要视图 (`summary_views.sql`)
  - `ROCPD_SQL_SCHEMA_ROCPD_MARKER_VIEWS`: 标记视图 (`marker_views.sql`)<!-- verified: 2026-05-27 -->

- **`rocpd_sql_schema_jinja_variables_t`** (`sql.h`): Jinja 模板变量结构体，用于 schema 文件中的变量替换：
  - `uuid`: 用于表名中的唯一标识符（自动添加下划线前缀，连字符替换为下划线）
  - `guid`: 全局唯一标识符<!-- verified: 2026-05-27 -->

- **`rocpd_sql_options_t`** (`sql.h`): SQL 选项标志：
  - `ROCPD_SQL_OPTIONS_SQLITE3_PRAGMA_FOREIGN_KEYS`: 启用外键约束，在 schema 前插入 `PRAGMA foreign_keys = ON;`<!-- verified: 2026-05-27 -->

- **`rocpd_version_triplet_t`** (`rocpd.h`): 版本三元组结构体，包含 `major`、`minor`、`patch` 字段。<!-- verified: 2026-05-27 -->

## 关键函数

- **`rocpd_get_version(uint32_t* major, uint32_t* minor, uint32_t* patch)`** (`rocpd.cpp`): 查询 ROCPD 库版本号。返回 `ROCPD_STATUS_SUCCESS`。<!-- verified: 2026-05-27 -->

- **`rocpd_get_version_triplet(rocpd_version_triplet_t* info)`** (`rocpd.cpp`): 查询版本三元组。<!-- verified: 2026-05-27 -->

- **`rocpd_get_status_name(rocpd_status_t status)`** (`rocpd.cpp`): 获取状态码的名称字符串（如 `"ROCPD_STATUS_SUCCESS"`）。使用编译期索引序列递归查找。<!-- verified: 2026-05-27 -->

- **`rocpd_get_status_string(rocpd_status_t status)`** (`rocpd.cpp`): 获取状态码的描述字符串（如 `"Success"`）。<!-- verified: 2026-05-27 -->

- **`rocpd_sql_load_schema(engine, kind, options, variables, callback, schema_path_hints, num_schema_path_hints, user_data)`** (`sql.cpp`): 核心 schema 加载函数。执行以下步骤：
  1. 验证 SQL 引擎类型（目前仅支持 SQLite3）
  2. 根据 `kind` 确定 schema 文件名
  3. 按优先级搜索 schema 文件：用户路径 > 环境变量 `ROCPD_SCHEMA_PATH` > 库安装路径
  4. 读取 schema 文件内容
  5. 执行 Jinja 变量替换（`{{uuid}}`、`{{guid}}`）
  6. 可选添加 SQLite3 PRAGMA 语句
  7. 通过回调函数返回 schema 路径和内容<!-- verified: 2026-05-27 -->

- **`rocpd::sql::get_install_path()`** (`sql.cpp`): 内部函数，通过 `dladdr` 定位 `rocpd_sql_load_schema` 符号的库路径，计算 `share/rocprofiler-sdk-rocpd` 目录作为 schema 安装路径。<!-- verified: 2026-05-27 -->

## 调用关系

- **上游**：
  - 输出格式化器模块 (`source/lib/output/generateRocpd.cpp`)：调用 ROCPD API 将追踪数据写入 SQLite3 数据库
  - Python 绑定 (`source/lib/python/rocpd/libpyrocpd.cpp`)：提供 `RocpdImportData` 函数用于 Python 接口
  - `rocpd` 命令行工具：使用 `rocpd_sql_load_schema` 加载 schema

- **下游**：
  - SQLite3 C API：通过 SQL 语句操作数据库
  - 文件系统：读取 `.sql` schema 文件
  - `rocprofiler::common` 库：使用日志、环境变量、文件系统等基础设施
  - `rocprofiler-register`：通过 `dlsym` 和 `dladdr` 进行符号查找

## 数据流

1. **Schema 加载**：`rocpd_sql_load_schema()` 被调用时，按优先级搜索 schema 文件
2. **文件读取**：找到 schema 文件后，读取其内容到字符串
3. **变量替换**：对 schema 内容执行 Jinja 模板变量替换（`{{uuid}}` → `_value`，`{{guid}}` → `value`）
4. **PRAGMA 注入**：如果启用了外键选项，在 schema 前插入 `PRAGMA foreign_keys = ON;`
5. **回调返回**：通过 `rocpd_sql_load_schema_cb_t` 回调函数将处理后的 schema 路径和内容返回给调用者
6. **数据库初始化**：调用者使用返回的 schema 内容初始化 SQLite3 数据库表结构
7. **数据写入**：输出格式化器将追踪数据通过 SQL INSERT 语句写入数据库

## 已知限制与边界情况

- **仅支持 SQLite3**：目前 `ROCPD_SQL_ENGINE_SQLITE3` 是唯一支持的 SQL 引擎，其他引擎返回 `ROCPD_STATUS_ERROR_SQL_INVALID_ENGINE`。
- **Schema 文件搜索**：搜索路径按用户路径 > 环境变量 > 库安装路径的优先级。如果所有路径都找不到 schema 文件，返回 `ROCPD_STATUS_ERROR_SQL_SCHEMA_NOT_FOUND`。
- **UUID 特殊处理**：`{{uuid}}` 变量有特殊处理逻辑：非空字符串自动添加下划线前缀，连字符 (`-`) 替换为下划线 (`_`)，因为 UUID 用于表名。
- **空内容检查**：如果 schema 文件读取后内容为空，返回 `ROCPD_STATUS_ERROR_SQL_SCHEMA_PERMISSION_DENIED`。
- **日志初始化**：库加载时通过 `_rocpd_init_logging` 全局变量强制初始化日志系统。
- **字符串持久化**：使用 `rocprofiler::common::get_string_entry()` 确保回调返回的字符串指针在回调返回后仍然有效。
- **ABI 稳定性**：通过 `ROCPD_EXTERN_C_INIT` / `ROCPD_EXTERN_C_FINI` 宏确保 C 接口的 ABI 稳定性。

## 与官方文档的关系

- [使用 rocpd 输出格式](source/docs/how-to/using-rocpd-output-format.rst) - ROCPD 输出格式的完整使用指南，包括生成、转换和查询
- [rocprofv3 I/O 控制选项](source/docs/how-to/rocprofv3-io-options.rst) - 输出格式和路径配置选项
- [使用 rocprofv3](source/docs/how-to/using-rocprofv3.rst) - rocprofv3 工具的使用指南
