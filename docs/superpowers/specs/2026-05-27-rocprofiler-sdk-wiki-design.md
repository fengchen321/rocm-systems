# Rocprofiler-SDK 源码阅读 Wiki 设计文档

## 概述

为 `projects/rocprofiler-sdk` 项目建立一个人类可读的源码阅读 wiki，采用分层组织方式：先建立架构级全局理解，再按功能域深入各模块。所有代码相关内容须通过 `codegraph` 工具验证。

## 约束

- **格式**：纯 Markdown 文件，Git 管理
- **验证**：涉及代码的内容必须用 `codegraph query/callers/callees` 验证
- **版本控制**：wiki 写作过程中不允许 git commit，所有文件作为工作区变更保留。写作完成后由用户决定是否提交。

## 目录结构

```
wiki/rocprofiler-sdk/
│
├── README.md                          # 入口：项目简介 + 学习路线图
├── glossary.md                        # 术语表
│
├── architecture/
│   ├── overview.md                    # 整体架构图 + 分层说明
│   ├── data-flow.md                   # 核心数据流
│   ├── key-abstractions.md            # 核心抽象：Agent, Context, Buffer, Callback
│   └── dependencies.md               # 模块间依赖 + 外部依赖
│
└── modules/
    ├── core/                          # 核心 SDK 库 (source/lib/rocprofiler-sdk/)
    │   ├── agent.md                   # GPU Agent 管理
    │   ├── buffer.md                  # 缓冲区追踪服务
    │   ├── callback-tracing.md        # 回调式追踪
    │   ├── counters.md                # 硬件计数器收集
    │   ├── pc-sampling.md             # PC 采样
    │   ├── registration.md            # 工具注册系统
    │   └── context.md                 # 分析上下文管理
    │
    ├── interceptors/                  # 运行时拦截层
    │   ├── hip.md                     # HIP 运行时拦截
    │   ├── hsa.md                     # HSA 运行时拦截
    │   └── kfd.md                     # KFD 内核驱动拦截
    │
    ├── companion/                     # 伴随库
    │   ├── aqlprofile.md              # AQL/PM4 profiling 包生成
    │   ├── roctx.md                   # ROCTx 标记库
    │   ├── rocpd.md                   # ROCPD 输出格式
    │   └── output.md                  # 输出格式化器
    │
    ├── tools/                         # CLI 工具
    │   ├── rocprofv3.md               # 主 profiling 工具
    │   └── rocpd-cli.md               # ROCPD 数据库查询工具
    │
    └── bindings/
        └── python.md                  # Python 绑定
```

## 笔记模板

每个模块笔记遵循统一格式：

```markdown
# 模块名称

## 概述
一段话说明该模块的职责和在整体架构中的位置。

## 关键文件
| 文件 | 路径 | 职责 |
|------|------|------|
| xxx.cpp | source/lib/rocprofiler-sdk/ | ... |

## 核心数据结构
- 结构体/类的说明、关键字段

## 关键函数
- 函数签名、职责、调用时机

## 调用关系
- 上游：谁调用了本模块
- 下游：本模块调用了谁
- （使用 codegraph callers/callees 验证）

## 数据流
该模块的数据如何流入、处理、流出。

## 已知限制与边界情况
- 该模块的已知限制、错误处理机制、边界情况

## 与官方文档的关系
链接到 source/docs/ 中对应的 rst 文档。
```

## codegraph 工具说明

`codegraph` 是已安装在本机的代码智能分析工具，项目已通过 `codegraph init` 和 `codegraph index` 完成初始化。

常用命令：

```bash
# 搜索符号定义
codegraph query <symbol_name>

# 查找谁调用了某个函数
codegraph callers <file>:<function>

# 查找某个函数调用了谁
codegraph callees <file>:<function>

# 查看项目文件结构
codegraph files
```

验证成功的判断标准：命令返回非空结果且包含预期的文件路径和符号名称。

## 验证规范

每个模块笔记中的代码相关内容须通过 codegraph 验证：

| 内容类型 | 验证命令 | 示例 |
|----------|----------|------|
| 符号存在性 | `codegraph query <symbol>` | `codegraph query rocprofiler_configure_buffer_service` |
| 调用关系 | `codegraph callers <symbol>` | `codegraph callers agent.cpp:get_agent` |
| 被调用关系 | `codegraph callees <symbol>` | `codegraph callees registration.cpp:registration_init` |

验证通过后在对应条目旁标注：`<!-- verified: YYYY-MM-DD -->`

## 学习路线

### 第一阶段：架构理解
1. [整体架构](architecture/overview.md)
2. [核心抽象](architecture/key-abstractions.md)
3. [数据流](architecture/data-flow.md)
4. [模块依赖](architecture/dependencies.md)

### 第二阶段：核心模块（按依赖顺序）
1. [Agent 管理](modules/core/agent.md) — GPU 设备发现与管理
2. [注册系统](modules/core/registration.md) — 工具如何接入 SDK
3. [上下文管理](modules/core/context.md) — profiling 会话管理
4. [缓冲区追踪](modules/core/buffer.md) — 异步数据收集
5. [回调追踪](modules/core/callback-tracing.md) — 同步事件通知
6. [硬件计数器](modules/core/counters.md) — 性能计数器采集
7. [PC 采样](modules/core/pc-sampling.md) — 程序计数器采样

### 第三阶段：拦截与输出
1. [HIP 拦截](modules/interceptors/hip.md)
2. [HSA 拦截](modules/interceptors/hsa.md)
3. [KFD 拦截](modules/interceptors/kfd.md)
4. [输出格式化](modules/companion/output.md)
5. [ROCPD](modules/companion/rocpd.md)

### 第四阶段：工具与扩展
1. [rocprofv3](modules/tools/rocprofv3.md)
2. [Python 绑定](modules/bindings/python.md)

## 术语表

| 术语 | 全称 | 说明 |
|------|------|------|
| AQL | Architected Queue Language | AMD GPU 队列语言，用于提交 kernel dispatch |
| HSA | Heterogeneous System Architecture | 异构系统架构，GPU 运行时底层接口 |
| HIP | Heterogeneous-compute Interface for Portability | AMD GPU 编程模型（类似 CUDA） |
| KFD | Kernel Fusion Driver | AMD GPU 内核驱动 |
| ROCTx | ROCm Tracing Extensions | 用户自定义标记/注解 API |
| ROCPD | ROCm Profiling Data | 基于 SQLite 的结构化输出格式 |
| ATT | Advanced Thread Trace | 硬件级线程追踪 |
| SPM | System Performance Monitor | 系统性能监控 |
| PC Sampling | Program Counter Sampling | 程序计数器采样，用于热点分析 |
| PM4 | Packet Manager 4 | GPU 命令包格式 |
| Dispatch | Kernel Dispatch | 一次 GPU kernel 启动 |
| Agent | GPU Agent | 一个可被管理的 GPU 设备实例 |
| Context | Profiling Context | 一次 profiling 会话的配置和状态 |
| Buffer | Tracing Buffer | 异步数据收集缓冲区，用于批量存储追踪事件 |
| Callback | Tracing Callback | 同步事件通知，事件发生时立即调用用户回调函数 |
| Tool | Profiler Tool | 用户编写的 profiling 工具，通过注册系统接入 SDK |
| ROCPD CLI | ROCPD Command Line | `rocpd.py` 命令行工具，用于查询和转换 ROCPD 数据库 |
