# 发送者消费者

**定义：**

> *Consumers are how a pipeline actually runs. They take a sender, connect it to a built-in receiver, and start the resulting operation. Until a consumer is called, a sender does nothing — it is just a description of work.*

消费者是启动 sender 所描述操作的入口。例如，`sync_wait` 接收一个 sender，将它连接到内置 receiver，得到操作状态（operation state），再启动该操作状态并等待完成。

上面的英文是对常见惰性 sender 管线的概括。严格来说，不一定要调用消费者才能执行：也可以手动调用 `connect` 和 `start`；某些 eager 操作还会提前启动工作。

**几种常见的消费者：**

| Consumer 消费者 | Returns 返回 | Use when 使用时机 | Eager or lazy? |
|---|---|---|---|
| `stdexec::sync_wait` | `std::optional<std::tuple<...>>` | Top-level synchronous wait; single value-completion shape.<br>顶层同步等待；单一值完成形态，可一次传递多个值。 | 调用时启动，并同步等待完成 |
| `stdexec::sync_wait_with_variant` | `std::optional<std::variant<std::tuple<...>...>>` | Same, but the sender has multiple value-completion shapes.<br>同样用于同步等待，但支持多种值完成形态。 | 调用时启动，并同步等待完成 |
| `exec::start_detached` | `void` | Top-level fire-and-forget; no owning scope. stdexec extension.<br>高层级的自动点火装置；无需固定作用范围。使用 stdexec 扩展函数。 | eager |
| `stdexec::spawn` | `void` | Fire-and-forget into an async scope that will be joined later.<br>在异步上下文中执行操作，该上下文会在之后被加入进来。 | eager |
| `stdexec::spawn_future` | sender 发件人 | Spawn into a scope and observe the result without blocking.<br>进入瞄准镜的视野中，观察结果，同时不要进行遮挡操作。 | eager |


## stdexec::sync_wait - 启动操作并同步等待完成

`stdexec::sync_wait` 是一个同步等待的消费者：它接收 sender，启动其描述的操作，直到操作完成才返回或抛出异常。这里的“完成”包括成功（value）、错误（error）和停止（stopped），不只是成功得到结果。

以本仓库 stdexec 的默认实现为例，其流程如下：

1. 创建内置 receiver 和用于保存完成结果的状态。
2. 调用 `connect(sender, receiver)`，得到操作状态 `op`。receiver 接收最终的完成信号及数据；不是把值“绑定成 receiver”，`connect` 本身也不启动操作。
3. 调用 `start(op)` 启动操作。`start` 返回 `void`，操作可能在这次调用中同步完成，也可能稍后异步完成。
4. 在调用线程上运行内部 `run_loop`，直到收到完成信号。因此，同步等待不等于忙等；调用线程可以执行投递到这个内部循环的工作，无工作时等待。它不会自动处理其他事件循环的任务。
5. 根据完成信号返回结果或抛出异常。返回 `optional` 的是 `sync_wait`，不是 `start`。

调用 `sync_wait` 就会启动并等待操作，因此将它标成 `lazy` 容易造成误解。sender 的惰性描述与消费者的同步等待是不同概念；`sync_wait` 也不会自动创建后台线程，操作在哪个线程执行取决于 sender 管线及其调度器。

**完成信号与返回语义：**

| 完成信号 | `sync_wait` 的行为 |
|---|---|
| `set_value(vs...)` | 返回有值的 `std::optional<std::tuple<std::decay_t<Ts>...>>`，tuple 保存传来的各个值。 |
| `set_error(error)` | 抛出异常；本仓库实现对 `std::exception_ptr` 重抛原异常，对 `std::error_code` 抛出 `std::system_error`，其他错误值作为异常抛出。 |
| `set_stopped()` | 返回空 `optional`（`std::nullopt`），表示停止。 |

这里假设启用了 C++ 异常支持。“单一值完成形态”指只有一种成功完成的参数类型组合，不是只能返回一个值。例如 `just(1, 2.0)` 对应 `tuple<int, double>`；`just()` 成功完成时对应有值的 `optional<tuple<>>`，与 stopped 返回的空 `optional` 不同。多种成功完成形态应使用 `sync_wait_with_variant`。

**使用案例：**

```cpp
auto [v] = stdexec::sync_wait(stdexec::just(42)).value();
// v == 42
```

这个例子正确：`just(42)` 会成功完成，因此可以直接调用 `.value()`。对于可能停止的 sender，应先检查 `optional`：

```cpp
auto result = stdexec::sync_wait(stdexec::just(42));
if (result) {
    auto [v] = *result;
    // 使用 v
} else {
    // 操作以 stopped 完成
}
```

若空 `optional` 上调用 `.value()`，会抛出 `std::bad_optional_access`；它与 sender 的 error 完成是两回事。

参考：[本仓库默认实现](../stdexec/include/stdexec/__detail/__sync_wait.hpp)、[NVIDIA stdexec 源码](https://github.com/NVIDIA/stdexec/blob/main/include/stdexec/__detail/__sync_wait.hpp)。


## stdexec::sync_wait_with_variant - 同步等待多种成功完成形态

两者都会启动操作并同步等待完成，区别在于支持的成功结果类型：

| 消费者 | 成功完成形态 | 返回类型 |
|---|---|---|
| `sync_wait` | 一种参数类型组合 | `optional<tuple<Ts...>>` |
| `sync_wait_with_variant` | 支持多种参数类型组合，也支持一种 | `optional<variant<tuple<As...>, tuple<Bs...>, ...>>` |

**对比示例：固定类型的结果与运行时选择的不同类型结果。**

第一部分使用 `just(42, 3.14)`，始终以 `set_value(int, double)` 成功完成。虽然一次传递两个值，但只有一种类型组合，因此可以使用 `sync_wait`。

第二部分的 `make_sender(ok)` 在 `ok` 为真时产生整数 `42`；为假时抛出异常，再由 `let_error` 将错误恢复为字符串 `"fallback"`。所以整个 sender 有两种成功完成形态：`set_value(int)` 和 `set_value(std::string)`，需要用 `sync_wait_with_variant` 接收。

```cpp
#include <stdexec/execution.hpp>

#include <exception>
#include <iostream>
#include <stdexcept>
#include <string>
#include <tuple>
#include <variant>

// 成功时产生 int；失败时，通过 let_error 恢复成 string 成功结果。
auto make_sender(bool ok) {
    return stdexec::just(ok)
         | stdexec::then([](bool ok) -> int {
               if (!ok) {
                   throw std::runtime_error("request failed");
               }
               return 42;
           })
         | stdexec::let_error([](std::exception_ptr) {
               return stdexec::just(std::string{"fallback"});
           });
}

int main() {
    // 一种成功形态：set_value(int, double)。
    auto fixed = stdexec::sync_wait(stdexec::just(42, 3.14));
    // fixed 的类型是 optional<tuple<int, double>>。
    if (fixed) {
        auto [number, ratio] = *fixed;
        std::cout << number << ", " << ratio << '\n';
    }

    for (bool ok : {true, false}) {
        // 两种成功形态：set_value(int) 或 set_value(string)。
        auto result = stdexec::sync_wait_with_variant(make_sender(ok));
        // result 是 optional，内部 variant 的分支为
        // tuple<int> 和 tuple<string>（不依赖分支顺序）。
        if (result) {
            std::visit([](const auto& values) {
                const auto& [value] = values;
                std::cout << value << '\n';
            }, *result);
        }

        // 改为以下调用会编译失败：sender 有两种成功完成形态。
        // auto invalid = stdexec::sync_wait(make_sender(ok));
    }
}
```

输出：

```text
42, 3.14
42
fallback
```

这里 `let_error` 返回的字符串是**恢复后的成功值**，不是 `sync_wait_with_variant` 将异常存进了 variant。未被恢复的 error 仍会导致它抛出异常；stopped 仍返回空 `optional`，这两点与 `sync_wait` 相同。

返回值的每一层各有用途：`optional` 区分成功与停止；`variant` 保存本次选择的成功类型组合；`tuple` 保存这次成功传递的各个值。每次操作仍只发送一次最终完成信号，variant 只持有其中一种结果。

