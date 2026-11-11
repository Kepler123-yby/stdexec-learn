# 调度适配器：从哪里启动、在哪里继续、如何返回

本文对应本仓库的 stdexec 版本。建议先阅读 [sender 的内部执行流程](senderExecutionFlow.md)，再把调度切换加入 `connect → start → 完成通知` 的流程。

## 1. 启动上下文与完成上下文

| 概念 | 含义 |
|---|---|
| 启动上下文 | 调用 operation state 的 start() 时所在的执行上下文 |
| 完成上下文 | 调用 receiver 的 set_value、set_error 或 set_stopped 时所在的执行上下文 |
| scheduler | 执行资源的轻量句柄，可用于创建调度操作；资源可以是线程池、单线程事件循环等 |

启动和完成可以发生在不同的上下文。异步操作可以在主线程启动、在 I/O 线程完成；接在它后面的 then 由成功完成通知触发，不会自动回到主线程。

**指定 scheduler 不等于固定线程。** 线程池 scheduler 可以选择不同 worker；需要线程亲和性时，可以使用专用单线程上下文。

## 2. 未显式指定 scheduler 的 just

```cpp
auto work = stdexec::just(21)
          | stdexec::then([](int x) { return x * 2; });
```

构造时，just 保存值，then 保存上游 sender 和回调，没有执行计算。连接并启动后，just 在它的 start() 调用过程中同步发送值，then 收到成功信号后直接调用回调。

如果主线程直接调用 sync_wait，这条链中的计算就在主线程执行。

**没有显式 scheduler 不等于执行行为未定义，也不等于自动使用默认线程池。** 具体行为由 sender 决定，不能把 just 的同步完成行为推广到异步 I/O 等 sender。

应用可以在统一提交入口包装 `starts_on(default_scheduler, sender)`，确定默认启动资源；这仍不禁止内部操作在其他资源上完成。

## 3. schedule：创建调度 sender

```cpp
auto work = stdexec::schedule(scheduler)
          | stdexec::then([] {
                // 在 scheduler 的执行资源上调用。
                return 42;
            });
```

schedule 是 sender 工厂，构造时只产生调度描述，连接并启动对应操作后才安排执行。

在线程池中，通常是任务入队、worker 取出操作、发送无参数的成功完成信号，然后触发 then。inline scheduler 可以直接在启动线程完成，不一定存在队列。

## 4. starts_on：指定上游从哪里启动

```cpp
auto work = stdexec::starts_on(scheduler, sender);
```

starts_on 返回组合 sender。启动后先调度到目标 scheduler，再在那里连接并启动传入的 sender。本版本没有 `sender | starts_on(scheduler)` 的管道形式。

调度顺序可以近似理解为：

```cpp
stdexec::schedule(scheduler)
| stdexec::let_value([sender = std::move(sender)]() mutable {
      return std::move(sender);
  });
```

这是顺序示意，不是完整替代实现；真实 starts_on 还处理执行环境等信息。

**它指定启动资源，不保证整条链都在该资源完成。** 内部 sender 可以切换 scheduler，也可以由外部异步资源完成。starts_on 自身不添加返回原上下文的调度操作。

典型用途：将 just 与 then 组成的计算链交给 CPU 线程池启动。

## 5. continues_on：转交上游完成信号

```cpp
auto work = stdexec::continues_on(sender, scheduler);
// 或：
auto work = sender | stdexec::continues_on(scheduler);
```

准确名称是 **continues_on**，不是 continue_on。

它不把上游已经发生的计算搬到目标资源，而是在上游完成后保存完成数据，调度到目标资源，再向下游交付完成信号。

```text
上游按自身语义执行
  → 产生值、错误或停止信号
  → continues_on 保存完成数据并安排目标调度
  → 在目标资源上向下游发送完成通知
  → 后续 then 在成功通知到来时执行
```

例如 `starts_on(a, just(21) | then(f)) | continues_on(b) | then(g)`：f 在 A 上执行，f 完成后切换到 B，g 在 B 上执行。

后续操作若再次切换资源，上下文还会变化。continues_on 自身不自动切回来。

## 6. on：局部切换后恢复原调度上下文

### 6.1 整个 sender 形式：on(target, sender)

```cpp
auto work = stdexec::on(target, sender);
```

返回位置来自最终连接到的 receiver 的环境，通过 get_start_scheduler 查询。执行时先在 target 上启动 sender，完成后把完成信号转交回查询到的 scheduler。

本版本的核心组合逻辑是：

```text
old = 从 receiver 环境查询 get_start_scheduler
continues_on(starts_on(target, sender), old)
```

所以，sender 在哪条线程构造，不决定返回位置。

### 6.2 插入局部操作：sender | on(target, closure)

```cpp
auto work = sender | stdexec::on(target, stdexec::then(f));
```

上游按自身语义执行；完成后切换到 target，执行 closure 所组成的操作；最后恢复上游的成功完成 scheduler。该 scheduler 通过上游属性及执行环境查询获得。

本版本的核心组合逻辑是：

```text
old = 查询 get_completion_scheduler<set_value_t>，结合上游属性与执行环境解析
sender | continues_on(target) | closure | continues_on(old)
```

显式写出两个 scheduler 时，往返关系更清楚：

```cpp
auto work = stdexec::schedule(a)
          | stdexec::then([] { return 21; })             // A
          | stdexec::on(b, stdexec::then([](int x) {
                return x * 2;                           // B
            }))
          | stdexec::then([](int x) { return x + 1; });  // 恢复到 A
```

on 通过查询确定返回资源，不是记住线程 ID 再跳回那条线程。返回线程池 scheduler 不保证返回同一个 worker。

如果环境无法提供可恢复的 scheduler，不能假设 on 总能返回；本版本某些无法解析的组合会编译失败，根环境另有 inline scheduler 回退规则。

### 6.3 隐式上下文与 sync_wait

```cpp
auto work = stdexec::just(21)
          | stdexec::on(b, stdexec::then(f));
auto result = stdexec::sync_wait(std::move(work));
```

这个例子隐藏了返回位置的来源：本版本的 sync_wait 在环境中提供其内部 run_loop 的 scheduler，on 可以据此解析返回位置。这不表示 just 固定绑定了某个全局 scheduler。

无论链在哪个资源完成，**sync_wait 返回后的普通 C++ 代码都会由调用它的线程继续执行**。只检查最终数值或返回后的线程，无法体现 on 的恢复作用。应在 on 后、sync_wait 前再接一个 then，观察该下游回调的位置。

## 7. 完整例子：A → B 与 A → B → A

使用两个真实的单线程上下文，避免用同一个系统 scheduler 假装两个资源，也不把 CPU scheduler 标注成 GPU。

```cpp
#include <stdexec/execution.hpp>
#include <exec/single_thread_context.hpp>

#include <iostream>
#include <thread>
#include <utility>

namespace ex = stdexec;

void where(const char* name) {
    std::cout << name << ": " << std::this_thread::get_id() << '\n';
}

int main() {
    exec::single_thread_context context_a;
    exec::single_thread_context context_b;
    auto a = context_a.get_scheduler();
    auto b = context_b.get_scheduler();
    where("main");

    auto transfer = ex::starts_on(
        a, ex::just(21) | ex::then([](int x) {
            where("continues_on upstream: A");
            return x;
        }))
        | ex::continues_on(b)
        | ex::then([](int x) {
              where("continues_on calculation: B");
              return x * 2;
          })
        | ex::then([](int x) {
              where("continues_on downstream: still B");
              return x;
          });
    auto [first] = ex::sync_wait(std::move(transfer)).value();

    auto roundtrip = ex::schedule(a)
        | ex::then([] { where("on upstream: A"); return 21; })
        | ex::on(b, ex::then([](int x) {
              where("on calculation: B");
              return x * 2;
          }))
        | ex::then([](int x) {
              where("on downstream: back to A");
              return x;
          });
    auto [second] = ex::sync_wait(std::move(roundtrip)).value();

    // 整个 sender 形式：从 A 启动，on 从环境取得返回位置 A。
    auto whole = ex::starts_on(a, ex::on(
        b, ex::just(21) | ex::then([](int x) {
            where("whole on calculation: B");
            return x * 2;
        })))
        | ex::then([](int x) {
              where("whole on downstream: back to A");
              return x;
          });
    auto [third] = ex::sync_wait(std::move(whole)).value();

    where("after sync_wait: main");
    std::cout << "results: " << first << ' ' << second << ' ' << third << '\n';
}
```

保存完整例子到 `/tmp/scheduling-demo.cpp` 后，在仓库根目录运行：

```bash
g++-14 -std=c++20 -pthread \
  -D_GLIBCXX_USE_TBB_PAR_BACKEND=0 \
  -I stdexec/include /tmp/scheduling-demo.cpp -o /tmp/scheduling-demo
/tmp/scheduling-demo
```

观察标注 A 的回调使用同一条线程，标注 B 的回调使用另一条线程；三个结果均为 42。

## 8. 与推理任务队列的配合

| 执行资源 | 负责的步骤 | 调度方式 |
|---|---|---|
| CPU 线程池 | 图像预处理、后处理 | starts_on 指定入口，continues_on 切入 |
| Channel 专用线程 | 串行使用 TensorRT context，提交推理并等待 CUDA 完成 | continues_on 切入，或 on 临时借用 |
| 主线程驱动的事件循环 | 需要主线程执行的回调 | continues_on(main_scheduler)，主线程必须驱动该事件循环 |
| 主线程消费结果队列 | 获取任意已完成任务的输出 | worker 将结果入队，主线程无需作为 scheduler 参与计算链 |

你的 runOnChannel 返回普通 Output，并在返回前等待 CUDA 完成，所以在推理线程上使用 then(runOnChannel) 即可。若接口改为返回异步推理 sender，则用 let_value 接续它。

on 表示“临时借用目标资源，完成后恢复”，continues_on 表示“把后续阶段交给目标资源”。把 CPU 代码切到推理线程不等于直接安排 GPU kernel，GPU 工作仍由 CUDA/TensorRT 接口提交。

## 9. 速查与源码

| 接口 | 调度意图 | 是否主动恢复原 scheduler |
|---|---|---|
| schedule(sch) | 创建在 sch 上完成的调度操作 | 否 |
| starts_on(sch, s) | 在 sch 上启动 s | 否 |
| s \| continues_on(sch) | 将 s 的完成通知交到 sch | 否 |
| on(sch, s) | 在 sch 上启动 s，再恢复启动 scheduler | 是，返回位置从环境查询 |
| s \| on(sch, closure) | 在 sch 上执行局部操作，再恢复上游成功完成 scheduler | 是，返回位置从上游及环境查询 |

- [scheduler 接口](../stdexec/include/stdexec/__detail/__schedulers.hpp)。
- [starts_on](../stdexec/include/stdexec/__detail/__starts_on.hpp)：目标调度与启动环境。
- [continues_on](../stdexec/include/stdexec/__detail/__continues_on.hpp)：保存完成数据、调度与转发。
- [on](../stdexec/include/stdexec/__detail/__on.hpp)：两种形式的 __transform_sender 展示 scheduler 的查询和组合。
- [sync_wait](../stdexec/include/stdexec/__detail/__sync_wait.hpp)：__sync_wait::__env 提供内部 loop 的 scheduler。
