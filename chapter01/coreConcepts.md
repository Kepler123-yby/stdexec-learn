# 核心概念

## scheduler 调度器

**原文定义：**

> *A scheduler is an object that provides a way to schedule work. They are lightweight handles to what is often a heavy-weight and immovable execution context*

调度器是**提供一种调度工作的方法**的对象。对于**执行上下文**来说，它是一种轻便的**句柄**。

### 执行上下文

**原文定义：**

> *Execution contexts are where work actually happens, and they can be anything that can execute code. Examples of execution contexts:*
> - thread pools
> - Event loops
> - GPUs
> - An I/O subsystem
> - any other execution model

简单来说就是执行上下文就是执行操作实际发生的地方，比如一些执行上下文的例子如上

#### scheduler的一些创建方式

| 执行资源 | 创建或获取方式 | 执行特点 |
|---|---|---|
| 默认系统并行资源 | `stdexec::get_parallel_scheduler()` | 使用系统默认的并行执行资源 |
| 自己管理的线程池 | `exec::static_thread_pool pool{4};` → `pool.get_scheduler()` | 可以指定工作线程数量 |
| 独立的单线程上下文 | `exec::single_thread_context ctx;` → `ctx.get_scheduler()` | 在专门的一条线程上执行 |
| 手动驱动的事件循环 | `stdexec::run_loop loop;` → `loop.get_scheduler()` | 由调用 `loop.run()` 的线程执行 |
| 原地执行 | `stdexec::inline_scheduler{}` | 在启动操作的线程上直接执行 |
| Linux I/O 上下文 | `exec::io_uring_context ctx;` → `ctx.get_scheduler()` | 支持基于 `io_uring` 的异步 I/O |
| NVIDIA GPU 上下文 | `nvexec::stream_context ctx;` → `ctx.get_scheduler()` | 在 GPU 执行上下文中调度 |

## sender 发送者

**原文定义：**

> *A sender is an object that describes an asynchronous computation that may happen later. It can do nothing on its own, but when connected to a receiver, it returns an operation state that can start the work described by the sender.*

简单来说，sender 是对“某个将来才会发生的异步计算”的描述。它自己什么都不会做；只有当它与一个 receiver 连接（`connect`）后，才会得到一个操作状态（operation state），再调用 `start` 才真正开始执行。

**发送者特性：**

- Produces values (or errors) asynchronously.
- Can be requested to stop early.
- Supports composition.
- Is lazy (does nothing until connected and started).

**创建sender:**

```cpp
auto sender = stdexec::schedule(sched);
```

## sender algorithm 发送方方法

**原文定义：**

>  *stdexec provides a set of generic sender algorithms that can be used to chain operations together. There are also algorithms, like stdexec::sync_wait, that can be used to launch the sender. The sender algorithms take care of connecting the sender to a receiver and managing the lifetime the operation state.*

简单来说就是，将发送者连接到接收者并管理操作状态生命周期的算法。

一般分为以下几类：

| 分类 | 输入 → 输出 | 典型算法 |
|---|---|---|
| **Sender factory：发送方工厂** | 普通参数 → sender | `just`、`just_error`、`just_stopped`、`schedule`、`read_env` |
| **Sender adaptor：发送方适配器** | 一个或多个 sender → 新 sender | `then`、`let_value`、`starts_on`、`continues_on`、`on`、`when_all`、`upon_error`、`upon_stopped` |
| **Sender consumer：发送方消费者** | sender → 启动执行、取得结果等 | `sync_wait` |

