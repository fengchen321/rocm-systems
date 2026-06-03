# 缓冲区追踪服务模块

## 概述

缓冲区（Buffer）模块提供了 rocprofiler-sdk 的缓冲式追踪基础设施。它管理追踪数据的存储、水位触发的自动刷新、以及双缓冲机制支持的无损（lossless）模式。该模块是缓冲式追踪服务（buffer tracing service）的核心组件，负责将各子系统（HSA、HIP、marker 等）产生的追踪记录高效地存储到缓冲区中，并在适当的时机通过回调函数将数据传递给工具。缓冲区模块还与内部线程池集成，确保刷新操作不会阻塞追踪路径。

## 关键文件

| 文件 | 路径 | 职责 |
|------|------|------|
| buffer.cpp | source/lib/rocprofiler-sdk/buffer.cpp | 缓冲区管理的核心实现，包含创建、刷新、销毁操作 |
| buffer.hpp | source/lib/rocprofiler-sdk/buffer.hpp | 内部头文件，定义 buffer::instance 结构体和模板方法 |
| buffer.h | source/include/rocprofiler-sdk/buffer.h | 公共 API 头文件 |
| buffer_tracing.cpp | source/lib/rocprofiler-sdk/buffer_tracing.cpp | 缓冲追踪服务的配置和查询实现 |

<!-- verified: 2026-05-27 -->

## 核心数据结构

### buffer::instance（内部结构）
- **定义位置**: `source/lib/rocprofiler-sdk/buffer.hpp:45`
- **职责**: 存储单个缓冲区实例的完整状态
- **关键字段**:
  - `buffers` (array<buffer_t, 2>): 双缓冲数组，用于无损模式下的交替写入和刷新
  - `syncer` (array<atomic_flag, 2>): 每个缓冲区的同步标志，防止写入和刷新并发冲突
  - `buffer_idx` (atomic<uint32_t>): 当前活跃缓冲区的索引
  - `drop_count` (atomic<uint64_t>): 丢弃的记录计数（在非无损模式下）
  - `watermark` (uint64_t): 触发自动刷新的记录数量阈值
  - `context_id` (uint64_t): 关联的上下文 ID
  - `buffer_id` (uint64_t): 缓冲区 ID
  - `task_group_id` (uint64_t): 线程池任务组 ID
  - `callback` (rocprofiler_buffer_tracing_cb_t): 数据刷新回调函数
  - `callback_data` (void*): 回调用户数据
  - `policy` (rocprofiler_buffer_policy_t): 缓冲区策略（NONE 或 LOSSLESS）

<!-- verified: 2026-05-27 -->

### unique_buffer_vec_t
- **定义**: `common::container::stable_vector<std::unique_ptr<instance>, 4>`
- **职责**: 全局缓冲区实例存储，支持稳定的迭代和并发访问

### record_header_buffer
- **定义位置**: `lib/common/container/record_header_buffer.hpp`
- **职责**: 底层记录存储，管理原始字节的序列化和反序列化

<!-- verified: 2026-05-27 -->

## 关键函数

### 公共 API

- **`rocprofiler_create_buffer`** (`buffer.cpp:273`)
  - 签名: `rocprofiler_status_t rocprofiler_create_buffer(rocprofiler_context_id_t context, size_t size, size_t watermark, rocprofiler_buffer_policy_t action, rocprofiler_buffer_tracing_cb_t callback, void* callback_data, rocprofiler_buffer_id_t* buffer_id)`
  - 职责: 创建新的追踪缓冲区并关联到指定上下文
  - 参数:
    - `context`: 关联的上下文 ID
    - `size`: 缓冲区大小（字节）
    - `watermark`: 触发刷新的记录数量
    - `action`: 缓冲区策略（ROCPROFILER_BUFFER_POLICY_NONE 或 ROCPROFILER_BUFFER_POLICY_LOSSLESS）
    - `callback`: 数据可用时的回调函数
    - `callback_data`: 回调用户数据
    - `buffer_id` [out]: 返回的缓冲区 ID
  - 调用时机: 工具在配置阶段调用，必须在初始化完成之前

- **`rocprofiler_flush_buffer`** (`buffer.cpp:317`)
  - 签名: `rocprofiler_status_t rocprofiler_flush_buffer(rocprofiler_buffer_id_t buffer_id)`
  - 职责: 手动触发缓冲区刷新
  - 特殊处理: 如果启用了 PC 采样，会先刷新内部 PC 采样缓冲区

- **`rocprofiler_destroy_buffer`** (`buffer.cpp:329`)
  - 签名: `rocprofiler_status_t rocprofiler_destroy_buffer(rocprofiler_buffer_id_t buffer_id)`
  - 职责: 销毁缓冲区
  - 前置条件: 缓冲区不能正在被刷新

<!-- verified: 2026-05-27 -->

### 内部函数

- **`allocate_buffer`** (`buffer.cpp:102`)
  - 签名: `std::optional<rocprofiler_buffer_id_t> allocate_buffer()`
  - 职责: 分配新的缓冲区实例
  - 副作用: 首次调用时初始化内部线程池

- **`get_buffer`** (`buffer.cpp:88`)
  - 签名: `instance* get_buffer(rocprofiler_buffer_id_t buffer_id)`
  - 职责: 根据缓冲区 ID 获取缓冲区实例指针
  - 实现: 使用直接索引（O(1)）而非线性搜索

- **`is_valid_buffer_id`** (`buffer.cpp:71`)
  - 职责: 验证缓冲区 ID 是否有效

- **`flush`** (`buffer.cpp:133`)
  - 签名: `rocprofiler_status_t flush(rocprofiler_buffer_id_t buffer_id, bool wait)`
  - 职责: 执行缓冲区刷新操作
  - 实现:
    1. 获取任务组
    2. 等待当前刷新完成（如果需要）
    3. 创建刷新任务，调用 `process_record_headers` 处理记录
    4. 在刷新任务中调用用户的回调函数
    5. 提交任务到线程池

### 模板方法

- **`instance::emplace`** (`buffer.hpp:122`)
  - 签名: `template <typename Tp> bool emplace(uint32_t category, uint32_t kind, Tp& value)`
  - 职责: 将追踪记录写入缓冲区
  - 行为:
    - 获取当前缓冲区索引
    - 等待刷新完成（如果正在刷新）
    - 尝试写入记录
    - 如果缓冲区满：
      - 无损模式：触发刷新并重试
      - 非无损模式：增加丢弃计数
    - 如果记录数达到水位，触发异步刷新

<!-- verified: 2026-05-27 -->

## 调用关系

### 上游（谁调用了本模块）
- **context 模块**: 关联缓冲区到上下文
- **buffer_tracing 模块**: 配置缓冲追踪服务时指定缓冲区
- **hsa 模块**: 写入 HSA API 追踪记录
- **hip 模块**: 写入 HIP API 追踪记录
- **marker 模块**: 写入 ROCTx 追踪记录
- **kernel_dispatch 模块**: 写入内核调度记录
- **hsa/async_copy 模块**: 写入内存复制记录
- **hsa/memory_allocation 模块**: 写入内存分配记录
- **kfd 模块**: 写入 KFD 事件记录
- **pc_sampling 模块**: 写入 PC 采样记录
- **counters 模块**: 写入计数器记录

### 下游（本模块调用了谁）
- **internal_threading 模块**: 使用线程池执行刷新任务
- **common/container/record_header_buffer**: 底层记录存储
- **pc_sampling 模块**: 在刷新时调用 `flush_internal_agent_buffers`

<!-- verified: 2026-05-27 -->

## 数据流

### 记录写入流程
1. **调用**: 子系统调用 `buffer->emplace(category, kind, value)`
2. **同步检查**: `local_sync` 确保缓冲区未在刷新
3. **写入**: 调用 `record_header_buffer::emplace` 序列化记录
4. **容量检查**: 如果写入失败（缓冲区满）：
   - 无损模式：触发同步刷新，切换到另一个缓冲区，重试写入
   - 非无损模式：增加丢弃计数
5. **水位检查**: 如果记录数 >= 水位，触发异步刷新

### 缓冲区刷新流程
1. **触发**: 水位达到或手动调用 `rocprofiler_flush_buffer`
2. **任务创建**: 创建 lambda 任务，捕获缓冲区 ID 和索引
3. **同步**: 使用 `atomic_flag` 确保同一缓冲区不会并发刷新
4. **记录处理**: 调用 `process_record_headers` 遍历所有记录头
5. **回调调用**: 将记录头数组传递给用户的回调函数
6. **清理**: 清除缓冲区内容和同步标志

### 缓冲区创建流程
1. **验证**: 检查初始化状态和缓冲区 ID 是否已存在
2. **分配**: 调用 `allocate_buffer` 创建新实例
3. **配置**: 设置水位、策略、回调函数等参数
4. **内存分配**: 为缓冲区分配指定大小的内存
   - 无损模式：分配两个缓冲区（双缓冲）

<!-- verified: 2026-05-27 -->

## 已知限制与边界情况

- **配置锁定**: 缓冲区只能在初始化完成之前创建（`get_init_status() > -1` 时返回错误）
- **缓冲区大小**: 如果记录大小超过缓冲区容量，会记录错误并返回 false
- **并发刷新**: 使用 `atomic_flag` 防止同一缓冲区的并发刷新
- **等待策略**: 刷新等待使用 `yield + sleep_for(10us)` 的忙等待策略
- **终结期间刷新**: 如果在终结期间触发刷新，会强制等待（wait=true）
- **双缓冲开销**: 无损模式需要两倍的内存
- **丢弃计数**: 非无损模式下，缓冲区满时记录会被丢弃，丢弃计数通过回调的 `drop_count` 参数传递
- **PC 采样集成**: 手动刷新会先刷新 PC 采样的内部缓冲区
- **任务组销毁**: 如果在任务组销毁后尝试刷新，会触发 FATAL 错误

<!-- verified: 2026-05-27 -->

## 与官方文档的关系

- API 参考: [rocprofiler-sdk API reference](../../../../projects/rocprofiler-sdk/source/docs/api-reference/rocprofiler-sdk_api_reference.rst)
- 缓冲服务: [buffered services](../../../../projects/rocprofiler-sdk/source/docs/api-reference/buffered_services.rst)
- 头文件 Doxygen 注释: [buffer.h](../../../../projects/rocprofiler-sdk/source/include/rocprofiler-sdk/buffer.h)、[buffer_tracing.h](../../../../projects/rocprofiler-sdk/source/include/rocprofiler-sdk/buffer_tracing.h) 中包含详细的 API 文档
