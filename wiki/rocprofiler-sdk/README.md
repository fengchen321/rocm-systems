# ROCprofiler-SDK Wiki

## 项目简介

ROCprofiler-SDK 是 AMD 新一代 GPU profiling/tracing SDK，提供硬件计数器采集、PC 采样、内核追踪等能力。它是 ROCm 性能分析工具链的核心基础设施，替代了旧版 ROCTracer、ROCprofiler (v1/v2) 和 rocprof/rocprofv2，通过统一的 API 为开发者提供对 GPU 计算应用的低层级性能分析接口。SDK 支持 HIP/HSA/KFD API 追踪、dispatch/device 级别计数器采集、PC Sampling（Host Trap）、线程追踪（SQTT/ATT），以及通过 ROCTx API 实现的用户自定义标记/注解功能。

## 学习路线

### 第一阶段：架构理解

从整体架构入手，了解 SDK 的分层设计、核心抽象和数据流转方式。

- [架构总览](architecture/overview.md)
- [核心抽象](architecture/key-abstractions.md)
- [数据流](architecture/data-flow.md)
- [依赖关系](architecture/dependencies.md)

### 第二阶段：核心模块

深入各个核心模块，理解 Agent 管理、注册系统、Context、Buffer、Callback Tracing、计数器和 PC 采样的实现细节。

- [Agent](modules/core/agent.md)
- [Registration](modules/core/registration.md)
- [Context](modules/core/context.md)
- [Buffer](modules/core/buffer.md)
- [Callback Tracing](modules/core/callback-tracing.md)
- [Counters](modules/core/counters.md)
- [PC Sampling](modules/core/pc-sampling.md)

### 第三阶段：拦截与输出

了解各运行时拦截层（HIP/HSA/KFD）的工作机制，以及结构化数据输出（ROCPD）的设计。

- [HIP 拦截器](modules/interceptors/hip.md)
- [HSA 拦截器](modules/interceptors/hsa.md)
- [KFD 拦截器](modules/interceptors/kfd.md)
- [Companion 输出](modules/companion/output.md)
- [ROCPD](modules/companion/rocpd.md)

### 第四阶段：工具与扩展

掌握命令行工具 rocprofv3 的使用方式，以及 Python 绑定等扩展能力。

- [rocprofv3](modules/tools/rocprofv3.md)
- [rocprofv3 指令触发流程](modules/tools/rocprofv3-flows.md)
- [Python 绑定](modules/bindings/python.md)

## 术语表

查阅项目中涉及的专业术语及其解释，请参见 [术语表](glossary.md)。

## 使用说明

本 wiki 是源码阅读笔记，内容基于对 `projects/rocprofiler-sdk` 源码的分析和整理。代码相关内容（如函数调用关系、数据结构定义、模块依赖等）均通过 codegraph 工具进行交叉验证，以确保准确性。文档中的分析和结论可能随源码演进而变化，建议结合实际代码版本阅读。
