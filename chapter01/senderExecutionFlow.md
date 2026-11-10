# just → then → sync_wait 的内部执行流程

本文以本仓库中的 stdexec 实现为依据，解释值如何经过 sender 执行链，到达最终消费者。

最重要的区分是：**构造 sender 只是描述操作；connect 建立执行状态；start 启动操作；完成信号将结果交给 receiver。**

## 1. 从一个完整例子开始

```cpp
#include <stdexec/execution.hpp>

#include <iostream>
#include <utility>

namespace ex = stdexec;

int main() {
    auto source = ex::just(21);
    std::cout << "1. just sender 已构造\n";

    auto work = std::move(source)
              | ex::then([](int x) {
                    std::cout << "3. then 收到 " << x << '\n';
                    return x * 2;
                });
    std::cout << "2. then sender 已构造，回调尚未执行\n";

    auto result = ex::sync_wait(std::move(work));
    auto [value] = result.value();
    std::cout << "4. sync_wait 返回 " << value << '\n';
}
```

输出：

```text
1. just sender 已构造
2. then sender 已构造，回调尚未执行
3. then 收到 21
4. sync_wait 返回 42
```

在仓库根目录可以编译：

```bash
g++-14 -std=c++20 -pthread \
  -D_GLIBCXX_USE_TBB_PAR_BACKEND=0 \
  -I stdexec/include /tmp/sender-flow.cpp -o /tmp/sender-flow
/tmp/sender-flow
```

上面的命令假设已将完整例子保存到 `/tmp/sender-flow.cpp`。

## 2. 三个角色：sender、receiver、operation state

| 角色 | 在本例中的职责 |
|---|---|
| sender | 描述“产生 21，再调用函数计算两倍” |
| receiver | 接收操作发出的成功值、错误或停止信号 |
| operation state | 保存连接后的执行状态，供 `start` 启动，并在操作未完成时保持有效 |

二者之间的接口关系是：

```cpp
// 接口示意，不是一个独立的完整程序。
auto operation = ex::connect(sender, receiver);
ex::start(operation);
```

`connect` 的返回值是 operation state，不是计算结果。`start` 启动 operation state，也不会直接返回计算结果。

结果通过 receiver 的完成接口传递：

```cpp
ex::set_value(receiver, values...); // 成功，可以携带零个或多个值
ex::set_error(receiver, error);     // 失败，携带错误
ex::set_stopped(receiver);          // 停止，不携带值
```

这些是统一调用入口，具体 receiver 提供相应处理。下面讨论 `just(21)` 的成功路径，最后补充错误与停止路径。

## 3. 构造阶段：保存值和回调，尚未传值

### 3.1 just 保存输入

```cpp
auto source = ex::just(21);
```

它创建一个保存 `21` 的 sender，描述：

> 当连接并启动后，将保存的值通过 set_value 发送给 receiver。

`just` 按衰变后的值类型保存参数。本例保存的是一个 `int`，不是马上向某个接收者发送 `21`。

### 3.2 then 将上游和回调组合起来

```cpp
auto work = std::move(source) | ex::then(f);
```

这里 `ex::then(f)` 是适配器闭包，`|` 将它应用到上游 sender，产生新的 sender。对于本例，可以理解成：

```text
then sender
├── 上游 sender：just(21)
└── 回调 f：接收 int，返回 x * 2
```

此时 `f` 没有被调用，`21` 也没有被发送。

**then 是适配器；then 与上游组合后产生的是 sender。实际接收上游完成信号的是连接阶段建立的内部 receiver。**

## 4. 连接阶段：消费者建立接收链

### 4.1 sync_wait 准备最终接收者

调用 `sync_wait(std::move(work))` 后，它为本次操作准备：

- 最终结果存储，本例类型为 `std::optional<std::tuple<int>>`。
- 错误存储，用于保留异常。
- 一个用于驱动等待过程的 `run_loop`。
- 一个能访问这些存储位置的最终 receiver。

最终 receiver 的成功处理可以简化理解为下面的伪代码；实际实现支持多值、转发和异常处理：

```text
final_receiver.set_value(value):
    将 value 保存到结果存储
    调用等待 loop 的 finish()
```

### 4.2 connect 将 then 的内部 receiver 接到 just 后面

`sync_wait` 内部调用：

```cpp
auto operation = ex::connect(sender, final_receiver);
```

对于本例，可以把逻辑连接过程理解成：

1. 外层 then 操作保存回调 `f` 和最终 receiver。
2. 为上游准备一个内部 receiver，负责调用 `f` 并向最终 receiver 转发结果。
3. 将 just sender 与这个内部 receiver 连接，产生上游的 operation state。
4. 返回能够启动这条链的外层 operation state。

实际 stdexec 使用通用模板组合这些状态，不一定按以上名称创建独立对象；这里描述的是各部分的逻辑职责。

建立的完成信号路径是：

```mermaid
flowchart LR
    J[just 的操作] -->|set_value：21| T[then 的内部 receiver]
    T -->|调用 f：得到 42| F[sync_wait 的最终 receiver]
    F --> R[保存结果并结束等待]
```

图中 then 到最终 receiver 的交接同样通过 `set_value` 完成。

**本例的 connect 只建立状态和连接关系，不调用业务回调，不发送 21。**

### 4.3 组合后的 sender 到底是什么对象

可以先把模板和内部标签去掉，用下面的结构想象这个例子的对象。以下代码只说明结构，不是完整、符合 stdexec 概念的 sender 实现：

```cpp
struct MultiplyByTwo {
    int operator()(int x) const { return x * 2; }
};

struct JustSender {
    int value;
};

struct ThenSender {
    JustSender upstream;
    MultiplyByTwo function;
};

// 对应 just(21) | then(MultiplyByTwo{})：
ThenSender work{JustSender{21}, MultiplyByTwo{}};
```

**组合就是对象嵌套：外层对象保存上游对象和函数对象，不是启动一个后台服务，也不是把值存进 future。**

再加一层 then，结构类似 `ThenSender<ThenSender<JustSender, F>, G>`。实际类型使用模板，回调可以是任意满足要求的函数对象。

本版本源码中，`then_t::__sender<F, S>` 继承自 `__sexpr<then_t, F, S>`，后者基于 `__tuple<Tag, Data, Child...>` 保存标签、数据和子 sender。因此实际存储形式对应：

```text
then_t::__sender<回调类型, 上游类型>
└── __sexpr
    ├── 标签：then_t
    ├── 数据：回调对象
    └── 子 sender：just sender，内部保存 21
```

### 4.4 receiver 从哪里来，长什么样

首先，`sync_wait` 创建最终 receiver。它持有结果存储和等待状态的指针，收到完成通知后写入结果并结束等待。

然后，`connect(work, final_receiver)` 创建 then 的操作状态，并用一个内部 receiver 连接上游 just。用伪代码描述其形状：

```text
ThenState:
    function                  // 保存回调对象
    downstream_receiver       // 保存 sync_wait 提供的最终 receiver

ThenReceiver:
    state 的引用或指针

    set_value(x):
        y = state.function(x)
        set_value(state.downstream_receiver, y)

JustOperation:
    value = 21
    receiver = ThenReceiver{指向 ThenState}

    start():
        set_value(receiver, value)

ThenOperation:
    state = ThenState{回调, 最终 receiver}
    upstream_operation = JustOperation{21, 指向 state 的内部 receiver}

    start():
        start(upstream_operation)
```

伪代码省略了错误、停止、环境查询和转发。实际通用内部 receiver 是 `__rcvr<Tag, State, Index>`，持有操作状态的引用，并将完成调用分派到 then 的处理逻辑。

连接前后可以对应为：

```text
连接前的 sender 对象：
ThenSender { upstream = JustSender{21}, function = f }

连接后的 operation state：
ThenOperation
├── state
│   ├── function = f
│   └── downstream_receiver = sync_wait 的最终 receiver
└── upstream_operation
    ├── value = 21
    └── receiver = ThenReceiver{引用上面的 state}
```

sender 描述本身不绑定最终消费者；连接时传入哪个 receiver，产生的 operation state 就负责把结果交给它。内部 receiver 和它引用的状态必须在操作未完成期间保持有效。

### 4.5 receiver 与 future 的区别

| 对象 | 作用 | 典型接口 |
|---|---|---|
| receiver | 操作完成时被调用的处理对象 | `set_value`、`set_error`、`set_stopped` |
| `std::promise` | 将结果写入共享状态 | `set_value`、`set_exception` |
| `std::future` | 从共享状态等待并取得结果 | `wait`、`get` |

receiver 是完成通知的接收端，可以理解成一组带状态的回调。它不要求提供结果存储，也没有统一的 `get()` 接口。

从职责看，最终 receiver 写入结果的动作更接近 promise 一侧；但 receiver 还可以立即调用下一步，并不必须经过 promise/future 的共享状态。then 的内部 receiver 就是在收到值后直接继续计算。

本例中，等待和取值是 sync_wait 的职责，receiver 只负责接收通知、保存结果并唤醒等待。因此不能把 receiver 本身理解成 future。

## 5. 启动阶段：从外层操作启动上游

接下来，`sync_wait` 调用：

```cpp
ex::start(operation);
```

外层 then 操作启动其上游 operation state，最终进入 just 操作的 `start()`。

just 操作已经保存了值和 receiver，所以它可以立即发送成功完成信号。用伪代码表示：

```text
just_operation.start():
    set_value(then_receiver, 21)
```

因此，**just 真正“传值”的时刻是 start 阶段，传递手段是调用 receiver 的 set_value。**

这不是 `just(21)` 函数直接返回一个 `int`，也不是 `start()` 返回一个 `int`。

## 6. 完成阶段：then 收到值并继续发送结果

then 的内部 receiver 收到 `set_value(..., 21)` 后，执行如下逻辑：

```text
then_receiver.set_value(21):
    调用 f(21)，得到 42
    set_value(final_receiver, 42)
```

这里有两种不同的结果传递方式：

| 位置 | 传递方式 |
|---|---|
| 回调 `f` 返回 42 | 普通 C++ 函数的 `return` |
| then 把 42 交给下游 | 调用 `set_value(final_receiver, 42)` |

所以，**回调返回值之后，还需要由 then 将它包装成成功完成信号，继续传给下游。**

如果回调返回 sender 对象，then 也会把该对象作为普通值传递，不会连接和启动它。需要接续内部 sender 时使用 `let_value`。

## 7. sync_wait 保存结果，结束等待并返回

最终 receiver 收到 `42`，将其保存到 `optional<tuple<int>>` 中，并调用 `loop.finish()`。

`sync_wait` 的主干逻辑可以简化为以下伪代码，省略具体类型与 receiver 的构造细节：

```text
准备 state、result 和 final_receiver
operation = connect(sender, final_receiver)
start(operation)
state.loop.run()
如果保存了异常，则重新抛出
返回 result
```

本例中，just 和 then 都同步完成，所以 **start(operation) 返回之前，42 就已经保存好了，finish() 也已经被调用。** 随后的 `loop.run()` 不需要继续等待结果。

对于真正异步完成的 sender，`start()` 可能先返回；`sync_wait` 随后在自己的 loop 中等待或执行被调度到该 loop 的工作，直到收到完成通知。这个 loop 不会代替所有外部 I/O 或 GPU 上下文驱动其事件循环。

最终 `sync_wait` 返回 `optional<tuple<int>>`。用户再通过 `.value()` 和结构化绑定取出普通的 `int`。

## 8. 用一条调用链串起来

```text
构造 just sender：保存 21
构造 then sender：保存上游和回调
调用 sync_wait
  ├─ 准备结果存储、run_loop、最终 receiver
  ├─ connect：建立操作状态和接收关系
  ├─ start：启动上游
  │    └─ just_operation.start()
  │         └─ set_value(then_receiver, 21)
  │              ├─ f(21) → 42
  │              └─ set_value(final_receiver, 42)
  │                   ├─ 保存 42
  │                   └─ loop.finish()
  ├─ loop.run()：本例已完成，无须继续等待
  └─ 返回包含 42 的 optional<tuple<int>>
```

如果再接一个 then，就在接收链中增加一层：

```text
just → then(f) 的内部 receiver → then(g) 的内部 receiver → 最终 receiver
 21              f(21)                 g(f(21))
```

不是消费者依次主动“取出”各层结果，而是完成信号沿着接收链向下游传递。

## 9. 这条链在哪个线程上执行

本例没有调度切换：just 在调用它的 operation state 的 `start()` 的线程上同步完成；then 在收到上游成功完成信号时直接调用回调。

如果主线程调用 `sync_wait(work)`，本例中 just 发送值、then 调用回调、最终 receiver 保存结果，都发生在主线程上。

**没有显式 scheduler 不等于行为未定义，也不等于存在一个默认线程池。** 不能把 just 的原地完成行为推广到所有 sender：异步 I/O 等操作有各自的完成机制。

希望显式指定启动资源时，可以包装整条链：

```cpp
// 假设 pool 是仍然存活的 exec::static_thread_pool。
auto result = ex::sync_wait(
    ex::starts_on(pool.get_scheduler(), std::move(work))
);
```

对于本文这条只包含 just 和 then 的链，回调会在线程池 worker 上执行，调用 sync_wait 的线程等待最终完成。若链内部再切换执行资源，完成位置还会随之改变。

## 10. 错误、停止与特殊返回值

| 情况 | then 的行为 | sync_wait 的结果 |
|---|---|---|
| 上游成功，回调返回普通值 | 将返回值通过 `set_value` 发送给下游 | 返回包含值元组的 optional |
| 回调返回 `void` | 回调结束后调用无参数的 `set_value` | 返回有值的 `optional<tuple<>>` |
| 回调抛异常 | 通过 `set_error` 向下游传递异常 | 在启用异常的配置下重新抛出 |
| 上游发送错误 | 不调用回调，转发错误 | 在启用异常的配置下抛出相应异常 |
| 上游发送停止信号 | 不调用回调，转发停止信号 | 返回 `std::nullopt` |

`std::nullopt` 表示停止完成，不表示操作仍未完成。

## 11. 对应源码入口

以下链接对应本仓库中的实现，内部名称和组织方式可能随 stdexec 版本变化：

- [just：保存数据、connect 和 operation state 的 start](../stdexec/include/stdexec/__detail/__just.hpp)：重点看 `__just::__opstate::start` 与 `__impl::__connect`。
- [then：处理上游完成信号](../stdexec/include/stdexec/__detail/__then.hpp)：重点看 `__complete_fn`，成功时调用 `__set_value_from`，其他完成信号直接转发。
- [组合对象与内部 receiver](../stdexec/include/stdexec/__detail/__basic_sender.hpp)：重点看 `__sexpr`、`__rcvr` 和 `__opstate` 的成员与构造过程。
- [回调返回值转为完成信号](../stdexec/include/stdexec/__detail/__receivers.hpp)：重点看 `__set_value_from`，包括 void 返回值和异常处理。
- [sync_wait：最终 receiver 与消费流程](../stdexec/include/stdexec/__detail/__sync_wait.hpp)：重点看 `__sync_wait::__receiver` 和包含 `connect → start → loop.run` 的实现。

相关概念可结合 [核心概念](coreConcepts.md)、[发送方工厂](senderFactory.md) 和 [发送方适配器](senderAdaptors.md) 阅读。
