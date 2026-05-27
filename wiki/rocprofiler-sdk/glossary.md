# 术语表

| 术语 | 全称 | 说明 |
|------|------|------|
| Agent | GPU Agent | 一个可被管理的 GPU 设备实例 |
| AQL | Architected Queue Language | AMD GPU 队列语言，用于提交 kernel dispatch |
| ATT | Advanced Thread Trace | 硬件级线程追踪 |
| Buffer | Tracing Buffer | 异步数据收集缓冲区，用于批量存储追踪事件 |
| Callback | Tracing Callback | 同步事件通知，事件发生时立即调用用户回调函数 |
| Context | Profiling Context | 一次 profiling 会话的配置和状态 |
| Dispatch | Kernel Dispatch | 一次 GPU kernel 启动 |
| HIP | Heterogeneous-compute Interface for Portability | AMD GPU 编程模型（类似 CUDA） |
| HSA | Heterogeneous System Architecture | 异构系统架构，GPU 运行时底层接口 |
| KFD | Kernel Fusion Driver | AMD GPU 内核驱动 |
| PC Sampling | Program Counter Sampling | 程序计数器采样，用于热点分析 |
| PM4 | Packet Manager 4 | GPU 命令包格式 |
| ROCPD | ROCm Profiling Data | 基于 SQLite 的结构化输出格式 |
| ROCPD CLI | ROCPD Command Line | rocpd.py 命令行工具，用于查询和转换 ROCPD 数据库 |
| ROCTx | ROCm Tracing Extensions | 用户自定义标记/注解 API |
| SPM | System Performance Monitor | 系统性能监控 |
| Tool | Profiler Tool | 用户编写的 profiling 工具，通过注册系统接入 SDK |
