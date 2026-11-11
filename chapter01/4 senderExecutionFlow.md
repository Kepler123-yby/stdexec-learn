# just → then / let_value → sync_wait 的内部执行流程

本文以本仓库的 stdexec 实现为依据。核心流程是：**构造 sender 描述操作，connect 建立执行状态，start 启动操作，receiver 接收完成通知，消费者处理最终结果。**

## 1. 同一个结果，两种组合

```cpp
#include <stdexec/execution.hpp>
#include <iostream>
#include <utility>

namespace ex = stdexec;

int main() {
    auto a = ex::just(21)
           | ex::then([](int x) {
                 std::cout << "then 收到 " << x << '\n';
                 return x * 2;          // 回调返回普通值
             });
    auto [first] = ex::sync_wait(std::move(a)).value();
    std::cout << "then 结果：" << first << '\n';

    auto b = ex::just(21)
           | ex::let_value([](int& x) {
                 std::cout << "let_value 收到 " << x << '\n';
                 return ex::just(x * 2); // 回调返回内部 sender
             });
    auto [second] = ex::sync_wait(std::move(b)).value();
    std::cout << "let_value 结果：" << second << '\n';
}
```

构造 a、b 时回调尚未执行；分别交给 sync_wait 后才输出：

```text
then 收到 21
then 结果：42
let_value 收到 21
let_value 结果：42
```

保存为 `/tmp/sender-flow.cpp` 后，在仓库根目录编译运行：

```bash
g++-14 -std=c++20 -pthread \
  -D_GLIBCXX_USE_TBB_PAR_BACKEND=0 \
  -I stdexec/include /tmp/sender-flow.cpp -o /tmp/sender-flow
/tmp/sender-flow
```

## 2. 构造后的 sender 是什么对象

组合就是对象嵌套。以下只说明成员关系，省略 sender 协议接口：

```cpp
struct JustSender {
    int value;
};

template<class Upstream, class Function>
struct ThenSender {
    Upstream upstream;
    Function function;
};

template<class Upstream, class Factory>
struct LetValueSender {
    Upstream upstream;
    Factory factory;
};
```

`just(21) | then(f)` 相当于保存 `ThenSender{JustSender{21}, f}`；再接 then(g)，就形成 `ThenSender<ThenSender<JustSender, F>, G>`。

let_value 同样保存上游和回调。**构造阶段不调用 factory；本例中的内部 just sender 在回调执行时才创建。**

本版本中，then 的实际类型 `then_t::__sender<F, S>` 基于 `__sexpr<then_t, F, S>`；let_value 使用 `__sexpr<let_value_t, Factory, Upstream>`。这些表达式保存标签、回调和子 sender，不保存最终消费者的 receiver。

## 3. connect：receiver 从哪里来

### 三个角色

| 角色 | 职责 |
|---|---|
| sender | 描述操作及其组合，保存输入和回调 |
| receiver | 接收 set_value、set_error 或 set_stopped 完成通知 |
| operation state | connect 的产物，保存执行所需状态，由 start 启动 |

```cpp
// 接口示意。
auto operation = ex::connect(sender, receiver);
ex::start(operation);
```

connect 返回执行状态，start 返回 void；结果通过完成通知传递，而不是从这两个调用直接返回。

### 最终 receiver 与内部 receiver

sync_wait 先准备结果存储、错误存储和等待用的 run_loop，再创建指向这些存储的最终 receiver。连接外层 sender 时，各适配器创建自己的内部 receiver，用它连接上游。

then 的连接结果可以理解成：

```text
ThenOperation
├── state
│   ├── function：f
│   └── downstream_receiver：sync_wait 的最终 receiver
└── upstream_operation：连接后的 just 操作
    ├── value：21
    └── receiver：ThenReceiver，引用上面的 state
```

内部 receiver 收到值后的逻辑是：

```text
ThenReceiver.set_value(x):
    y = state.function(x)
    set_value(state.downstream_receiver, y)
```

实际通用内部 receiver 是 `__rcvr<Tag, State, Index>`，通过状态引用分派完成处理。以上省略了错误、停止和环境查询；connect 本身不调用本例中的业务回调。

### receiver 不是 future

| 对象 | 接口与职责 |
|---|---|
| receiver | 通过 set_value、set_error、set_stopped 被通知完成，可以立即调用下一步 |
| std::promise | 通过 set_value、set_exception 将结果写入共享状态 |
| std::future | 通过 wait、get 等待并取出结果 |

receiver 更像一组带状态的完成回调，不要求存储结果，也没有统一的 get()。then 的内部 receiver 继续计算，sync_wait 的最终 receiver 才负责保存结果、结束等待。**最后向调用者返回结果的是 sync_wait。**

## 4. start 与完成通知：两条执行流

### then：回调返回值直接发给下游

sync_wait 建立连接后调用外层 start，外层操作启动上游：

```text
start(ThenOperation)
  └─ start(上游 just 操作)
       └─ set_value(ThenReceiver, 21)
            ├─ f(21) → 42
            └─ set_value(最终 receiver, 42)
                 ├─ 保存 42
                 └─ loop.finish()
```

just 在 start() 内同步发送保存的值。then 调用 f 后，将其普通 return 值作为 set_value 的参数传给下游。若继续连接 then(g)，下游就是 g 对应的内部 receiver，完成信号继续沿链传递。

### let_value：回调返回 sender，再接续执行

外层 connect 时，let_value 准备的状态类似：

```text
LetValueOperation
├── factory：回调对象
├── downstream_receiver：最终 receiver
├── saved_arguments：保存上游值的空间，初始为空
└── operation_storage：分阶段保存操作状态
    ├── 上游阶段：连接后的上游操作，内部 receiver 指向外层状态
    └── 内部阶段：回调返回的 sender 所连接出的操作状态
```

本版本用 variant 复用两个阶段的操作存储，并非始终保留两份独立状态。内部 sender 只能等上游成功完成、factory 被调用后再连接。

```text
start(LetValueOperation)
  └─ start(上游 just 操作)
       └─ set_value(let_value 的内部 receiver, 21)
            ├─ 保存上游值 21
            ├─ factory(保存值的引用) → just(42)
            ├─ connect 内部 sender，保存内部操作状态
            └─ start(内部操作)
                 └─ set_value(内部 receiver 包装, 42)
                      └─ set_value(最终 receiver, 42)
                           ├─ 保存 42
                           └─ loop.finish()
```

内部 receiver 包装负责提供相应环境并转发完成信号。源码通过 submit_result 封装内部提交：默认路径是构造时 connect、submit 时 start，定制 sender 也可能走合并提交路径。

### 关键差异

| 阶段 | then | let_value |
|---|---|---|
| 构造 | 保存上游和回调 | 相同 |
| 外层 connect、start | 连接并启动上游 | 相同，另准备参数和内部操作存储 |
| 上游成功完成 | 用上游值调用回调 | 先保存值，再以保存值的左值引用调用回调 |
| 回调返回后 | 将返回值发送给下游 | 连接并启动回调返回的 sender |
| 下游完成 | 由回调结果驱动 | 由内部 sender 的完成信号驱动 |

**let_value 多接续的是回调返回的 sender，不是把上游再执行一次。** then 的回调也可以返回 sender，但它只会把该对象当作普通值发送给下游，不会启动它。

## 5. 时序、生命周期与等待

### 同步与异步完成

上面的两条调用链缩进在 start 下，是因为本例所有操作同步完成：start 返回前结果已经保存。若上游异步完成，外层 start 可以先返回，之后收到成功通知时才调用 factory、连接并启动内部 sender。

内部 sender 也可以异步完成。let_value 不在回调中阻塞等待，下游在内部操作完成时才收到通知。

### 为什么 let_value 保存上游值

内部操作可能引用上游值，因此保存值和内部操作状态都需要覆盖其执行过程。调用方须保持外层 operation state 在操作未完成期间有效。

保存值以左值引用传入 factory，所以例子使用 int&；按值接收也可以，只接收 int&& 的回调则不能匹配。

### sync_wait 如何返回结果

其主干逻辑为：

```text
准备结果存储、等待 loop 和最终 receiver
operation = connect(sender, final_receiver)
start(operation)
loop.run()
如果保存了异常，则重新抛出
返回结果存储
```

成功时最终 receiver 保存值并调用 loop.finish()。本例在 start 返回前已完成，loop.run() 不需要继续等待。异步情况下，它等待或执行被安排到该 loop 的工作，直到完成；它不会代替所有外部 I/O、GPU 上下文驱动事件循环。

本例返回 `optional<tuple<int>>`，其中包含 42。结果通过 `.value()` 和结构化绑定取出。

## 6. 不能直接用 then 替代的例子：同一工作线程上的异步接续

场景：回调在一个专用工作线程上执行，它创建的下一项操作也必须排进同一线程的队列。下游需要取得下一项操作产生的 int，并且不能阻塞工作线程。

```cpp
#include <stdexec/execution.hpp>
#include <exec/single_thread_context.hpp>
#include <iostream>
#include <string_view>
#include <utility>

namespace ex = stdexec;

int main(int argc, char** argv) {
    exec::single_thread_context context;
    auto scheduler = context.get_scheduler();

    // 根据上游值创建新操作；该操作仍需由同一个 worker 执行。
    auto async_double = [scheduler](int x) {
        return ex::schedule(scheduler)
             | ex::then([x] { return x * 2; });
    };

    auto request = ex::schedule(scheduler)
                 | ex::then([] { return 21; });

    if (argc > 1 && std::string_view(argv[1]) == "--blocking") {
        auto work = std::move(request)
                  | ex::then([&](int x) {
                        std::cout << "worker 等待自己队列中的任务" << std::endl;
                        auto [value] = ex::sync_wait(async_double(x)).value();
                        return value;
                    });
        ex::sync_wait(std::move(work)); // 死锁，无法正常返回
    } else {
        auto work = std::move(request)
                  | ex::let_value(async_double)
                  | ex::then([](int value) { return value + 1; });
        auto [value] = ex::sync_wait(std::move(work)).value();
        std::cout << value << '\n'; // 43
    }
}
```

保存为 `/tmp/let-value-required.cpp`，使用第 1 节编译参数生成 `/tmp/let-value-required`。默认运行正常返回 43；阻塞分支可以用下面的命令观察，2 秒后由 timeout 终止：

```bash
/tmp/let-value-required
timeout 2s /tmp/let-value-required --blocking
```

两种写法的差异：

```text
let_value：worker 创建并启动下一项操作 → 操作入队 → 当前回调返回
          → worker 继续取任务 → 产生 42 → 下游得到 43

then + sync_wait：下一项操作入队 → worker 阻塞等结果
                 → 队列中的操作等 worker 执行 → 互相等待
```

如果直接写 then(async_double)，它只把内部 sender 当普通对象传下去，后面的 `[](int value)` 无法接收；如果在 then 中用 sync_wait 强行取值，就会出现上面的死锁。

**这里无法直接替代的是“根据上游值创建并非阻塞地接续内部操作”这一职责。** 可以使用其他接续机制重新实现它，但普通 then 本身只负责调用回调、转交返回值。

## 7. 错误、停止与执行上下文

| 情况 | 行为 |
|---|---|
| 上游错误或停止 | then、let_value 均不调用回调，直接转发 |
| then 回调返回 void | 下游收到无参数 set_value，sync_wait 返回有值的 optional<tuple<>> |
| let_value 回调返回普通值或 void | 不满足回调必须返回 sender 的要求，无法形成合法连接 |
| 调用回调，或 let_value 保存参数、连接内部 sender 时抛异常 | 通过 set_error 向下游传递异常 |
| 内部 sender 错误或停止 | let_value 转发相应完成信号 |
| 错误最终到达 sync_wait | 启用异常时抛出相应异常 |
| 停止最终到达 sync_wait | 返回 std::nullopt，表示已停止完成 |

第 1 节没有调度切换：主线程调用 sync_wait 时，两条链的计算都在主线程执行。第 6 节则显式调度到了专用工作线程。just 同步完成不代表存在默认 scheduler，也不能推广到所有 sender。

需要指定启动资源时，可以用 `starts_on(scheduler, sender)`。then、let_value 自身不指定新 scheduler；内部操作可以改变完成上下文。详见 [调度适配器](<5 SchedulingAdaptors.md>)。

## 8. 源码入口

| 源码 | 重点 |
|---|---|
| [__just.hpp](../stdexec/include/stdexec/__detail/__just.hpp) | __opstate::start 发送保存的值 |
| [__then.hpp](../stdexec/include/stdexec/__detail/__then.hpp) | __complete_fn 调用回调并转发结果 |
| [__let.hpp](../stdexec/include/stdexec/__detail/__let.hpp) | __impl 保存参数，__start_next_fn 接续内部 sender |
| [__submit.hpp](../stdexec/include/stdexec/__detail/__submit.hpp) | submit_result 封装内部连接与启动 |
| [__basic_sender.hpp](../stdexec/include/stdexec/__detail/__basic_sender.hpp) | __sexpr、__rcvr、__opstate 的对象结构 |
| [__receivers.hpp](../stdexec/include/stdexec/__detail/__receivers.hpp) | __set_value_from 处理回调返回值与异常 |
| [__sync_wait.hpp](../stdexec/include/stdexec/__detail/__sync_wait.hpp) | 最终 receiver 与 connect → start → loop.run |

相关概念：[核心概念](<1 coreConcepts.md>)、[发送方工厂](<2 senderFactory.md>)、[发送方适配器](<3 senderAdaptors.md>)。
