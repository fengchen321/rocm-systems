# Rocprofiler-SDK 源码阅读 Wiki 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 创建 `wiki/rocprofiler-sdk/` 目录结构和所有文档，形成完整的源码阅读笔记体系。

**Architecture:** 分层组织——先架构概览，再模块深入。所有代码内容通过 `codegraph` 验证。纯 Markdown 文件，无 git commit。

**Tech Stack:** Markdown, codegraph CLI

**设计文档:** `docs/superpowers/specs/2026-05-27-rocprofiler-sdk-wiki-design.md`

**约束:** wiki 写作过程中不允许 git commit，所有文件作为工作区变更保留。

## 模块笔记模板（所有 Task 8-24 共用）

每个模块笔记必须包含以下 8 个章节：

```markdown
# 模块名称

## 概述
一段话说明该模块的职责和在整体架构中的位置。

## 关键文件
| 文件 | 路径 | 职责 |
|------|------|------|

## 核心数据结构
- 结构体/类的说明、关键字段

## 关键函数
- 函数签名、职责、调用时机

## 调用关系
- 上游：谁调用了本模块
- 下游：本模块调用了谁

## 数据流
该模块的数据如何流入、处理、流出。

## 已知限制与边界情况
- 该模块的已知限制、错误处理机制、边界情况

## 与官方文档的关系
链接到 source/docs/ 中对应的 rst 文档。
```

---

### Task 1: 创建目录结构

**Files:**
- Create: `wiki/rocprofiler-sdk/` (目录)

- [ ] **Step 1: 创建所有子目录**

```bash
mkdir -p wiki/rocprofiler-sdk/architecture
mkdir -p wiki/rocprofiler-sdk/modules/core
mkdir -p wiki/rocprofiler-sdk/modules/interceptors
mkdir -p wiki/rocprofiler-sdk/modules/companion
mkdir -p wiki/rocprofiler-sdk/modules/tools
mkdir -p wiki/rocprofiler-sdk/modules/bindings
```

- [ ] **Step 2: 验证目录结构**

```bash
find wiki/rocprofiler-sdk -type d | sort
```

Expected:
```
wiki/rocprofiler-sdk
wiki/rocprofiler-sdk/architecture
wiki/rocprofiler-sdk/modules
wiki/rocprofiler-sdk/modules/bindings
wiki/rocprofiler-sdk/modules/companion
wiki/rocprofiler-sdk/modules/core
wiki/rocprofiler-sdk/modules/interceptors
wiki/rocprofiler-sdk/modules/tools
```

---

### Task 2: 编写 README.md 入口文件

**Files:**
- Create: `wiki/rocprofiler-sdk/README.md`

- [ ] **Step 1: 阅读项目 README 获取基本信息**

读取 `projects/rocprofiler-sdk/README.md`，提取项目简介、版本、核心功能。

- [ ] **Step 2: 编写 README.md**

内容包含：

1. **项目简介**（一段话）
2. **学习路线**（四阶段，含链接）：
   - 第一阶段：架构理解 → overview.md, key-abstractions.md, data-flow.md, dependencies.md
   - 第二阶段：核心模块 → agent.md, registration.md, context.md, buffer.md, callback-tracing.md, counters.md, pc-sampling.md
   - 第三阶段：拦截与输出 → hip.md, hsa.md, kfd.md, output.md, rocpd.md
   - 第四阶段：工具与扩展 → rocprofv3.md, python.md
3. **术语表链接** → glossary.md
4. **使用说明**（如何阅读、如何验证）

---

### Task 3: 编写 glossary.md 术语表

**Files:**
- Create: `wiki/rocprofiler-sdk/glossary.md`

- [ ] **Step 1: 编写术语表**

从设计文档的术语表章节提取所有术语，按字母排序。格式：

```markdown
# 术语表

| 术语 | 全称 | 说明 |
|------|------|------|
| AQL | ... | ... |
```

---

### Task 4: 编写 architecture/overview.md

**Files:**
- Create: `wiki/rocprofiler-sdk/architecture/overview.md`

- [ ] **Step 1: 用 codegraph 分析项目整体结构**

```bash
codegraph files
```

获取源码目录的整体布局。

- [ ] **Step 2: 阅读项目 README 和 docs/conceptual/ 获取架构信息**

读取 `projects/rocprofiler-sdk/source/docs/conceptual/comparing-with-legacy-tools.rst` 了解架构演进。

- [ ] **Step 3: 编写 overview.md**

内容：
- 整体架构分层图（文字版）：Public API → Core SDK → Runtime Interceptors → GPU Driver
- 各层职责说明
- 关键设计决策

---

### Task 5: 编写 architecture/key-abstractions.md

**Files:**
- Create: `wiki/rocprofiler-sdk/architecture/key-abstractions.md`

- [ ] **Step 1: 用 codegraph 查找核心抽象的定义**

```bash
codegraph query agent
codegraph query context
codegraph query buffer
codegraph query callback
```

确认这些核心概念对应的源文件和数据结构。

- [ ] **Step 2: 阅读公共 API 头文件**

读取 `projects/rocprofiler-sdk/source/include/rocprofiler-sdk/` 下的核心头文件（agent.h, context.h, buffer.h 等）。

- [ ] **Step 3: 编写 key-abstractions.md**

对每个核心抽象：
- 一句话定义
- 对应的头文件和实现文件
- 关键数据结构
- 在整体架构中的角色

---

### Task 6: 编写 architecture/data-flow.md

**Files:**
- Create: `wiki/rocprofiler-sdk/architecture/data-flow.md`

- [ ] **Step 1: 用 codegraph 追踪核心数据流路径**

```bash
codegraph callees source/lib/rocprofiler-sdk/buffer.cpp:rocprofiler_configure_buffer_service
codegraph callees source/lib/rocprofiler-sdk/callback_tracing.cpp:rocprofiler_configure_callback_service
```

- [ ] **Step 2: 编写 data-flow.md**

内容：
- 用户 API 调用 → SDK 内部处理 → 数据输出的完整路径
- buffer tracing 和 callback tracing 两条主要数据流路径
- 数据流图（文字版）

---

### Task 7: 编写 architecture/dependencies.md

**Files:**
- Create: `wiki/rocprofiler-sdk/architecture/dependencies.md`

- [ ] **Step 1: 分析 CMakeLists.txt 获取模块依赖**

读取 `projects/rocprofiler-sdk/source/lib/rocprofiler-sdk/CMakeLists.txt` 和根目录的 `CMakeLists.txt`。

- [ ] **Step 2: 列出外部依赖**

读取 `projects/rocprofiler-sdk/external/` 目录结构。

- [ ] **Step 3: 编写 dependencies.md**

内容：
- 内部模块间依赖关系
- 外部依赖列表及用途
- 依赖关系图（文字版）

---

### Task 8: 编写 modules/core/agent.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/core/agent.md`

- [ ] **Step 1: 用 codegraph 分析 agent 模块**

```bash
# 查找 agent 相关符号
codegraph query agent
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers agent.cpp:<function_name>
codegraph callees agent.cpp:<function_name>
```

- [ ] **Step 2: 阅读源文件**

读取 `projects/rocprofiler-sdk/source/lib/rocprofiler-sdk/agent.cpp` 和对应的头文件。

- [ ] **Step 3: 按模板编写 agent.md**

填充概述、关键文件、核心数据结构、关键函数、调用关系、数据流、已知限制、官方文档链接。

---

### Task 9: 编写 modules/core/registration.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/core/registration.md`

- [ ] **Step 1: 用 codegraph 分析 registration 模块**

```bash
# 查找 registration 相关符号
codegraph query registration
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers registration.cpp:<function_name>
codegraph callees registration.cpp:<function_name>
```

- [ ] **Step 2: 阅读源文件**

读取 `projects/rocprofiler-sdk/source/lib/rocprofiler-sdk/registration.cpp`（约54K，最大的文件）。

- [ ] **Step 3: 按模板编写 registration.md**

---

### Task 10: 编写 modules/core/context.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/core/context.md`

- [ ] **Step 1: 用 codegraph 分析 context 模块**

```bash
# 查找 context 相关符号
codegraph query context
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers context.cpp:<function_name>
codegraph callees context.cpp:<function_name>
```

- [ ] **Step 2: 阅读源文件**

读取 `projects/rocprofiler-sdk/source/lib/rocprofiler-sdk/context/` 目录下的文件。

- [ ] **Step 3: 按模板编写 context.md**

---

### Task 11: 编写 modules/core/buffer.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/core/buffer.md`

- [ ] **Step 1: 用 codegraph 分析 buffer 模块**

```bash
# 查找 buffer 相关符号
codegraph query buffer
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers buffer.cpp:<function_name>
codegraph callees buffer.cpp:<function_name>
```

- [ ] **Step 2: 阅读源文件**

读取 `projects/rocprofiler-sdk/source/lib/rocprofiler-sdk/buffer.cpp` 和 `buffer_tracing.cpp`。

- [ ] **Step 3: 按模板编写 buffer.md**

---

### Task 12: 编写 modules/core/callback-tracing.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/core/callback-tracing.md`

- [ ] **Step 1: 用 codegraph 分析 callback_tracing 模块**

```bash
# 查找 callback_tracing 相关符号
codegraph query callback_tracing
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers callback_tracing.cpp:<function_name>
codegraph callees callback_tracing.cpp:<function_name>
```

- [ ] **Step 2: 阅读源文件**

读取 `projects/rocprofiler-sdk/source/lib/rocprofiler-sdk/callback_tracing.cpp`。

- [ ] **Step 3: 按模板编写 callback-tracing.md**

---

### Task 13: 编写 modules/core/counters.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/core/counters.md`

- [ ] **Step 1: 用 codegraph 分析 counters 模块**

```bash
# 查找 counters 相关符号
codegraph query counters
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers counters/<file>.cpp:<function_name>
codegraph callees counters/<file>.cpp:<function_name>
```

- [ ] **Step 2: 阅读源文件**

读取 `projects/rocprofiler-sdk/source/lib/rocprofiler-sdk/counters/` 目录。

- [ ] **Step 3: 按模板编写 counters.md**

---

### Task 14: 编写 modules/core/pc-sampling.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/core/pc-sampling.md`

- [ ] **Step 1: 用 codegraph 分析 pc_sampling 模块**

```bash
# 查找 pc_sampling 相关符号
codegraph query pc_sampling
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers pc_sampling/<file>.cpp:<function_name>
codegraph callees pc_sampling/<file>.cpp:<function_name>
```

- [ ] **Step 2: 阅读源文件**

读取 `projects/rocprofiler-sdk/source/lib/rocprofiler-sdk/pc_sampling/` 目录。

- [ ] **Step 3: 按模板编写 pc-sampling.md**

---

### Task 15: 编写 modules/interceptors/hip.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/interceptors/hip.md`

- [ ] **Step 1: 用 codegraph 分析 HIP 拦截模块**

```bash
# 查找 HIP 拦截相关符号
codegraph query hip
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers hip/<file>.cpp:<function_name>
codegraph callees hip/<file>.cpp:<function_name>
```

- [ ] **Step 2: 阅读源文件**

读取 `projects/rocprofiler-sdk/source/lib/rocprofiler-sdk/hip/` 目录。

- [ ] **Step 3: 按模板编写 hip.md**

---

### Task 16: 编写 modules/interceptors/hsa.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/interceptors/hsa.md`

- [ ] **Step 1: 用 codegraph 分析 HSA 拦截模块**

```bash
# 查找 HSA 拦截相关符号
codegraph query hsa
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers hsa/<file>.cpp:<function_name>
codegraph callees hsa/<file>.cpp:<function_name>
```

- [ ] **Step 2: 阅读源文件**

读取 `projects/rocprofiler-sdk/source/lib/rocprofiler-sdk/hsa/` 目录。

- [ ] **Step 3: 按模板编写 hsa.md**

---

### Task 17: 编写 modules/interceptors/kfd.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/interceptors/kfd.md`

- [ ] **Step 1: 用 codegraph 分析 KFD 拦截模块**

```bash
# 查找 KFD 拦截相关符号
codegraph query kfd
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers kfd/<file>.cpp:<function_name>
codegraph callees kfd/<file>.cpp:<function_name>
```

- [ ] **Step 2: 阅读源文件**

读取 `projects/rocprofiler-sdk/source/lib/rocprofiler-sdk/kfd/` 目录。

- [ ] **Step 3: 按模板编写 kfd.md**

---

### Task 18: 编写 modules/companion/output.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/companion/output.md`

- [ ] **Step 1: 用 codegraph 分析 output 模块**

```bash
# 查找 output 相关符号
codegraph query output
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers output/<file>.cpp:<function_name>
codegraph callees output/<file>.cpp:<function_name>
```

- [ ] **Step 2: 阅读源文件**

读取 `projects/rocprofiler-sdk/source/lib/output/` 目录。

- [ ] **Step 3: 按模板编写 output.md**

---

### Task 19: 编写 modules/companion/rocpd.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/companion/rocpd.md`

- [ ] **Step 1: 用 codegraph 分析 rocpd 模块**

```bash
# 查找 rocpd 相关符号
codegraph query rocpd
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers rocprofiler-sdk-rocpd/<file>.cpp:<function_name>
codegraph callees rocprofiler-sdk-rocpd/<file>.cpp:<function_name>
```

- [ ] **Step 2: 阅读源文件**

读取 `projects/rocprofiler-sdk/source/lib/rocprofiler-sdk-rocpd/` 目录。

- [ ] **Step 3: 按模板编写 rocpd.md**

---

### Task 20: 编写 modules/companion/aqlprofile.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/companion/aqlprofile.md`

- [ ] **Step 1: 用 codegraph 分析 aqlprofile 模块**

```bash
# 查找 aqlprofile 相关符号
codegraph query aqlprofile
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers aqlprofile/<file>.cpp:<function_name>
codegraph callees aqlprofile/<file>.cpp:<function_name>
```

- [ ] **Step 2: 阅读源文件**

读取 `projects/rocprofiler-sdk/source/lib/aqlprofile/` 目录。

- [ ] **Step 3: 按模板编写 aqlprofile.md**

---

### Task 21: 编写 modules/companion/roctx.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/companion/roctx.md`

- [ ] **Step 1: 用 codegraph 分析 roctx 模块**

```bash
# 查找 roctx 相关符号
codegraph query roctx
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers rocprofiler-sdk-roctx/<file>.cpp:<function_name>
codegraph callees rocprofiler-sdk-roctx/<file>.cpp:<function_name>
```

- [ ] **Step 2: 阅读源文件**

读取 `projects/rocprofiler-sdk/source/lib/rocprofiler-sdk-roctx/` 目录。

- [ ] **Step 3: 按模板编写 roctx.md**

---

### Task 22: 编写 modules/tools/rocprofv3.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/tools/rocprofv3.md`

- [ ] **Step 1: 用 codegraph 分析 rocprofv3 相关代码**

```bash
# 查找 rocprofv3 相关符号
codegraph query rocprofv3
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers rocprofv3.py:<function_name>
codegraph callees rocprofv3.py:<function_name>
```

- [ ] **Step 2: 阅读 rocprofv3 源码**

读取 `projects/rocprofiler-sdk/source/libexec/rocprofv3.py`（约84K）。

- [ ] **Step 3: 阅读官方 how-to 文档**

读取 `projects/rocprofiler-sdk/source/docs/how-to/` 下关于 rocprofv3 的文档。

- [ ] **Step 4: 按模板编写 rocprofv3.md**

---

### Task 23: 编写 modules/tools/rocpd-cli.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/tools/rocpd-cli.md`

- [ ] **Step 1: 用 codegraph 分析 rocpd CLI 相关代码**

```bash
# 查找 rocpd CLI 相关符号
codegraph query rocpd
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers rocpd.py:<function_name>
codegraph callees rocpd.py:<function_name>
```

- [ ] **Step 2: 阅读 rocpd CLI 源码**

读取 `projects/rocprofiler-sdk/source/libexec/rocpd.py`。

- [ ] **Step 3: 按模板编写 rocpd-cli.md**

---

### Task 24: 编写 modules/bindings/python.md

**Files:**
- Create: `wiki/rocprofiler-sdk/modules/bindings/python.md`

- [ ] **Step 1: 用 codegraph 分析 Python 绑定**

```bash
# 查找 Python 绑定相关符号
codegraph query python
# 用 query 结果中的具体符号做 callers/callees 分析
codegraph callers python/<file>.py:<function_name>
codegraph callees python/<file>.py:<function_name>
```

- [ ] **Step 2: 阅读源文件**

读取 `projects/rocprofiler-sdk/source/lib/python/` 目录下的三个子目录（rocpd, rocprofv3, roctx）。

- [ ] **Step 3: 按模板编写 python.md**

---

### Task 25: 最终审查

**Files:**
- 所有已创建的 wiki 文件

- [ ] **Step 1: 验证所有内部链接**

检查 README.md 中的所有链接是否指向存在的文件。

- [ ] **Step 2: 验证 codegraph 标注**

检查各模块笔记中的 `<!-- verified: YYYY-MM-DD -->` 标注是否存在。

- [ ] **Step 3: 检查模板一致性**

确认所有模块笔记都遵循统一模板格式。
