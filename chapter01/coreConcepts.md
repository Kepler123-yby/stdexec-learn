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

### connect 与 start：从描述到执行

严格来说，通常写的是 `stdexec::connect(sender, receiver)` 和 `stdexec::start(op)`：它们是**定制点对象（CPO）**，提供统一的调用入口，再分派到具体类型的实现。`connect` 面向 sender，`start` 面向它返回的 operation state，而不是直接对 sender 调用 `start`。

先区分三个角色：

| 角色 | 作用 |
|---|---|
| sender（发送者） | 描述要做什么，例如产生 `42`、执行函数或调度到线程池 |
| receiver（接收者） | 定义完成后如何接收值、错误或停止信号 |
| operation state（操作状态） | 保存这一次执行需要的状态，把计算描述与结果接收方绑定起来 |

#### 1. connect：构造一次执行所需的状态

```cpp
auto op = stdexec::connect(std::move(sender), std::move(receiver));
```

`connect` 根据 sender 和 receiver 的具体类型，构造对应的操作状态。它通常保存 receiver、sender 携带的数据，以及执行中需要的资源或子操作状态。比如 `just(42)` 的操作状态需要保存待发送的 `42` 和接收它的 receiver。

**这一步建立连接，不启动所描述的计算。** 不过，构造状态本身仍然会执行代码，例如移动对象、准备资源，也可能抛出异常。因此，“惰性”并不表示 `connect` 完全没有开销。

对于组合 sender，连接过程通常还会为上游构造内部 receiver。例如 `just(21) | then(f)` 中，内部 receiver 收到 `21` 后执行 `f`，再把结果交给下游 receiver；具体实现可以合并或优化这些状态。

#### 2. start：启动操作状态

```cpp
stdexec::start(op);
```

`start` 分派到操作状态的启动实现。它返回 `void`，且不能向外抛异常；**计算结果通过 receiver 交付，不通过 `start` 的返回值交付**。

启动不等于创建线程，也不等于等待完成：

- `just(42)` 可以在调用 `start` 的线程上立即交付 `42`，在 `start` 返回前就完成。
- 线程池调度器产生的 `schedule(sched)` 通常在启动时把任务入队，之后由工作线程发送完成信号。

因此，执行发生在哪条线程上、何时完成，由具体 sender 和执行上下文决定，不能仅凭 `start` 判断。

#### 3. 完成：通过 receiver 回传结果

一次操作最终通过以下三种完成通道之一通知 receiver：

| 完成调用 | 含义 |
|---|---|
| `stdexec::set_value(std::move(receiver), values...)` | 成功完成，传递零个或多个值 |
| `stdexec::set_error(std::move(receiver), error)` | 以错误完成，传递错误对象 |
| `stdexec::set_stopped(std::move(receiver))` | 以停止状态完成，不携带值 |

同一次操作只发送一次完成信号。具体 sender 支持哪些通道，由它的完成签名决定，例如 `just(42)` 只需要值通道。

下面用同步完成的 `just` 展示整个流程：

```cpp
#include <stdexec/execution.hpp>
#include <cstdio>
#include <utility>

struct print_receiver {
    using receiver_concept = stdexec::receiver_tag;

    auto get_env() const noexcept -> stdexec::env<> { return {}; }

    void set_value(int value) && noexcept {
        std::printf("收到：%d\n", value);
    }
};

int main() {
    auto sender = stdexec::just(42);  // 描述计算，尚未交付值
    auto op = stdexec::connect(std::move(sender), print_receiver{});
    // op 保存本次执行的状态，此时仍未打印
    stdexec::start(op);              // 同步调用 receiver，打印“收到：42”
}
```

流程可以记为：

```text
sender + receiver
        │ connect：绑定并构造状态
        ▼
  operation state
        │ start：启动
        ▼
  执行计算 / 提交任务
        │ 完成（可能立即，也可能稍后）
        ▼
receiver 的 set_value / set_error / set_stopped
```

**生命周期约束：** 对同一个操作状态只能调用一次 `start`，并以左值 `op` 传入；操作状态必须保持稳定，不能在执行中移动或提前销毁，至少要存活到完成通知。`start` 返回并不代表异步操作已经完成。上例能在 `main` 结束时安全销毁 `op`，是因为 `just` 已经同步完成；换成线程池操作时，就需要等待机制来保证生命周期。sender 若引用了外部对象，这些对象的生命周期也需要单独保证。

日常使用 `stdexec::sync_wait(sender)` 时，连接内部 receiver、保存操作状态、调用 `start` 并等待完成等步骤由消费者负责，因此通常不需要手写这些底层调用。

参考：[官方开发者指南](https://nvidia.github.io/stdexec/developer/index.html)与 [connect / start 参考文档](https://nvidia.github.io/stdexec/reference/index.html)。

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

