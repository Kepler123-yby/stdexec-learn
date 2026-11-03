# 发送方工厂

相关内容可以参考：[官方说法入口](https://nvidia.github.io/stdexec/user/index.html)

## just - 注入常量值

**原文定义:**

> *stdexec::just is the simplest sender factory. You give it zero or more values; you get back a sender that, when started, immediately delivers those values to its receiver as a value completion. No context transition, no asynchrony — just a synchronous handoff of values into the sender world.*

简单来说，`stdexec::just` 是最简单的发送方工厂，它接受零个或多个值，返回一个发送方，当启动时，立即将这些值传递给接收者作为值完成。

**使用案例：**

通常搭配`lambda`和`std::exec::then`使用：

```cpp
auto sender = stdexec::just() | 
              stdexec::then([](int x) { 
                  return x * 2; 
              });
```

## just_error - 引入一个明显的错误

**原文定义：**

> *stdexec::just_error is to the error channel what just is to the value channel: it produces a sender that immediately completes with the given error.*

简单来说就是和`stdexec::just`类似，搭配的另一个工具是`stdexec::upon_error`，只不过它是在错误通道上使用的。

## just_stopped - 执行字面量消除操作

**原文定义：**

> *stdexec::just_stopped produces a sender that immediately completes via set_stopped (no datums — the stopped channel carries none).*

这个的定义的作用是执行字面量消除操作，将一个值转换为一个立即停止的 sender。

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
        | stdexec::then([](int value) {
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

查询一个source的状态，根据其反映的状态来决定下一步的操作。

**原理剖析：**

光看这一个函数的定义，很难理解它的作用。接下来将会从底层描述其原理：

在`stdexec`中，有一个停止请求源的类型：`stdexec::inplace_stop_source`，这个类型提供两个核心操作：`get_token()`和`request_stop()`。

此时的`stdexec::read_env`的作用是通过`stdexec::get_stop_token`读取`token`的状态，从而停止整个循环的工作，案例如下：

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
