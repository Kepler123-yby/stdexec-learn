# 发送方工厂

相关内容可以参考：[官方说法入口](https://nvidia.github.io/stdexec/user/index.html)

## just - 注入常量值

**原文定义:**

> *stdexec::just is the simplest sender factory. You give it zero or more values; you get back a sender that, when started, immediately delivers those values to its receiver as a value completion. No context transition, no asynchrony — just a synchronous handoff of values into the sender world.*

简单来说，`stdexec::just` 是最简单的发送方工厂，它接受零个或多个值，返回一个发送方，当启动时，立即将这些值传递给接收者作为值完成。

**使用案例：**

通常搭配`lambda`和`stdexec::then`使用，`just`给出的值会作为参数传给后面的`lambda`（值的个数和类型要对得上）：

```cpp
auto sender = stdexec::just(21) | 
              stdexec::then([](int x) { 
                  return x * 2; 
              });
```

## just_error - 引入一个明显的错误

**原文定义：**

> *stdexec::just_error is to the error channel what just is to the value channel: it produces a sender that immediately completes with the given error.*

简单来说就是和`stdexec::just`类似，搭配的另一个工具是`stdexec::upon_error`，只不过它是在错误通道上使用的。

注意：错误值不会进入`then`，`then`只处理值通道，错误会原样向后传递，直到遇到`upon_error`（或其他处理错误的算法）。如果最后一直没人处理，`sync_wait`会把这个错误作为异常抛出。

## just_stopped - 引入一个“已取消”的结果

**原文定义：**

> *stdexec::just_stopped produces a sender that immediately completes via set_stopped (no datums — the stopped channel carries none).*

`stdexec::just_stopped`不接受任何参数，它返回一个 sender，启动后立即通过停止通道（`set_stopped`）完成，停止通道不携带任何数据。

它常用来人为制造“被取消”这条路径，方便测试后面的`upon_stopped`等算法。和`just_error`一样，`then`不会处理它，停止信号会原样向后传递。

**使用案例：**

```cpp
#include <stdexec/execution.hpp>

#include <iostream>
#include <string>
#include <utility>

// Sender 是模板参数，可以接收不同具体类型的 sender
template <typename Sender>
void run(const char* name, Sender source)
{
    std::cout << "=== " << name << " ===\n";

    auto work =
        std::move(source)
        // 注意 noexcept：不加的话 then 会额外多出 set_error(std::exception_ptr)，
        // 而下面 upon_error 的 lambda 只能接收 std::string，会导致编译失败
        | stdexec::then([](int value) noexcept {
              std::cout << "then：处理成功值 " << value << '\n';
              return value * 2;
          })
        | stdexec::upon_error([](const std::string& error) {
              std::cout << "upon_error：处理错误 " << error << '\n';
              return -1;  // 将错误恢复为成功值
          })
        | stdexec::upon_stopped([] {
              std::cout << "upon_stopped：处理停止\n";
              return -2;  // 将停止恢复为成功值
          });

    // 三条路径最终都会产生一个 int
    auto [result] = stdexec::sync_wait(std::move(work)).value();

    std::cout << "最终结果：" << result << "\n\n";
}

int main()
{
    run("just",
        stdexec::just(21));

    run("just_error",
        stdexec::just_error(std::string{"文件不存在"}));

    run("just_stopped",
        stdexec::just_stopped());
}
```

## read_env — 从接收者的环境中读取一个值

**原文定义：**

> *`stdexec::read_env` lets a pipeline inspect properties of its receiver — the receiver’s stop token, allocator, preferred scheduler, and anything else the environment exposes. You give it a query CPO (`stdexec::get_stop_token`, `stdexec::get_scheduler`, …) and get back a sender that, when started, evaluates that query against the connected receiver’s environment and delivers the result as a value.*

读取“接收者环境”里的信息（如停止令牌、调度器、分配器等），并把读到的值作为值完成发给后面的算法。注意，它读的是**下游**（最终连接的 receiver）提供的环境，而不是上游 sender 的状态。

**原理剖析：**

光看这一个函数的定义，很难理解它的作用。接下来将会从底层描述其原理：

在`stdexec`中，有一个停止请求源的类型：`stdexec::inplace_stop_source`，这个类型提供两个核心操作：`get_token()`和`request_stop()`。

此时的`stdexec::read_env`的作用是通过`stdexec::get_stop_token`从环境中取出`token`，后续的算法再根据`token`的状态来停止循环，案例如下：

```cpp
auto token = source.get_token();

// 方式一：明确传入这个 token
auto a = stdexec::just(token)
       | stdexec::then([](auto token) {
           while (true) 
           {
               if (token.stop_requested()) 
               {
                   break;  // 退出当前任务的循环
               }
               // 执行一轮工作
           }
         });

// 方式二：使用调用方放进环境的 token
auto b = stdexec::read_env(stdexec::get_stop_token)
       | stdexec::then([](auto token) {
           while (true) 
           {
               if (token.stop_requested()) 
               {
                   break;  // 退出当前任务的循环
               }
               // 执行一轮工作
           }
         });

// 方式二需要调用方真的把 token 放进环境，否则读到的是 never_stop_token，
// stop_requested() 永远为 false，循环永远不会退出。
// 直接用 sync_wait(b) 就属于这种情况：sync_wait 的环境不提供 stop token。
// 可以用 write_env 手动把 token 写进环境：
auto c = std::move(b)
       | stdexec::write_env(stdexec::prop{stdexec::get_stop_token, token});
```

## schedule — 在调度器上启动一个管道操作

**原文定义：**

> *`stdexec::schedule` is how you say “begin work on this execution context”. You give it a scheduler — a lightweight handle to a thread pool, GPU stream, event loop, etc. — and get back a sender that, when started, value-completes (with no datums) from the context of that scheduler. Anything you chain after it runs on that context.*

你需要提供一个调度器——一个用于管理线程池、GPU 流水线、事件循环等资源的轻量级接口——然后会得到一个发送器，当这个发送器启动时，它会从那个调度器的上下文中返回一个无数据的值。之后，你可以在这个发送器上继续执行任何操作。

**使用案例：**

```cpp
auto sched = stdexec::get_parallel_scheduler();

auto sndr =
  stdexec::schedule(sched)               // hop onto sched
  | stdexec::then([] { return 42; });    // ... and compute on it

auto [v] = stdexec::sync_wait(std::move(sndr)).value();
// v == 42, computed on the parallel scheduler
```

## | 左结合符号解析

**接下来用一个案例说明：**

```cpp
auto s1 = stdexec::then(sndr, f);
```
和

```cpp
auto s2 = sndr | stdexec::then(f);
```
相互等效，那我们可以推导，左结合是以有到右的方式传参数：

```cpp
sndr | stdexec::then(f) = stdexec::then(sndr, f)
// 同样，有

sndr | stdexec::then(f) | stdexec::then_error(g) | stdexec::then_stopped(s) = stdexec::then_stopped(stdexec::then_error(stdexec::then(sndr, f), g))
```

将这个传入`stdexec::sync_wait`，sync_wait会等待发送器完成并返回结果。
