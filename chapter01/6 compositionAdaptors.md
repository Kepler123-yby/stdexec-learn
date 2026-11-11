# 组合适配器

这三个接口分别解决两件事：**汇总多个操作**，以及**将一个操作的多种成功输出形状统一成 variant**。本文对应本仓库使用的 stdexec 版本。

## 1. when_all：启动多个操作，汇总完成结果

when_all 接收零个或多个 sender，返回组合 sender。连接并启动后，它启动各个子操作，等所有子操作完成，再向下游发送一次最终完成通知。

所有分支成功时，将各分支的成功值按**输入 sender 的排列顺序**拼接为 set_value 参数，不按完成先后排序：

```cpp
// 以下片段假设 namespace ex = stdexec，且已包含所需头文件。
auto work = ex::when_all(
    ex::just(1),
    ex::just(2.5),
    ex::just(std::string{"x"})
);
auto [i, d, s] = ex::sync_wait(std::move(work)).value();
// i == 1，d == 2.5，s == "x"
```

这里三个分支的输出类型本来就不同，普通 when_all 完全支持。

**when_all 构造时返回 sender，不是结果元组。** 成功完成时它发出 `set_value(receiver, 1, 2.5, string{"x"})`；外面的 sync_wait 才把这些值存入 tuple，并返回 optional<tuple<int, double, string>>。

单个分支也可以一次发送多个值，例如 `when_all(just(1, 2), just(3))` 成功时发送三个值 1、2、3。零个输入的 when_all() 则成功发送零个值。

### 懒执行与并发

构造组合 sender 不启动子操作，要等 connect 后调用 start。

when_all 不会依次等待每个异步任务完成后再启动下一个，但**不保证同时执行，不自行创建线程，也不保证具体启动顺序**。纯 just 分支会在调用 start 的线程中同步完成；异步分支能否重叠、是否真正并行，取决于分支自己的操作和执行资源。

例如，让两个 CPU 任务使用同一个多线程池即可具备并行执行条件，不需要为每个分支创建独立 scheduler：

```cpp
exec::static_thread_pool pool(4);
auto cpu = pool.get_scheduler();

auto work = ex::when_all(
    ex::starts_on(cpu, ex::just(10) | ex::then([](int x) { return x * 2; })),
    ex::starts_on(cpu, ex::just(5)  | ex::then([](int x) { return x + 1; }))
);
auto [a, b] = ex::sync_wait(std::move(work)).value(); // 20、6
```

starts_on 是安排 CPU 工作的一种方式，不是并发组合的必需条件；异步 I/O sender 本身也可以提供异步执行能力。

### 错误与停止

- 全部分支成功，才向下游发送拼接后的成功值。
- 某个分支错误或停止时，会请求其余分支停止；停止是协作请求，不会强制杀死线程。
- 即使出现错误或停止，仍等待所有子操作完成，才向下游交付最终通知。
- 最终错误到达 sync_wait 时抛出异常；最终停止时返回 nullopt，不返回部分成功结果。

when_all 不是“谁先完成就返回谁”的竞速操作。

## 2. 先区分两种“不同类型”

| 情况 | 示例 | 需要 variant 吗 |
|---|---|---|
| 不同 sender 各有固定形状 | A 发 int，B 发 string | 不需要，when_all 可直接组合 |
| 同一 sender 有多种可能的成功形状 | A 根据运行时情况发 int 或 string | 要统一形状时使用 into_variant |
| 一次完成携带多个值 | A 发 (int, double) | 仍是一种形状，不等于多种候选形状 |

所谓“多种成功形状”，指一个 sender 的类型允许多种成功完成签名，例如：

```cpp
// 完成签名示意，不是两个都要发送：
set_value_t(int)
set_value_t(std::string)
```

每次执行只会完成一次。本例可能发送 int，也可能发送 string，不是持续多次发送，也不是只有参数名字不同。

普通 when_all 不接受某个输入具有多种成功形状。若要组合上述多形状输入，可使用下面两个接口。

## 3. into_variant：统一单个 sender 的成功形状

```cpp
auto wrapped = ex::into_variant(source);
// 或：
auto wrapped = source | ex::into_variant();
```

假设 source 可能成功发送 int 或 string，转换后只有一种成功形状：

```cpp
set_value_t(std::variant<std::tuple<int>, std::tuple<std::string>>)
```

每一种候选形状用 tuple 保存其参数包，variant 选择本次实际发生的形状。一次完成携带 (int, double) 时，对应的候选类型就是 tuple<int, double>。

```cpp
// 假设 source 具有上面的两种成功形状。
auto [value] = ex::sync_wait(source | ex::into_variant()).value();
std::visit([](const auto& values) {
    std::cout << std::get<0>(values) << '\n';
}, value);
```

into_variant 可以单独用于统一下游回调的参数，或者使多形状 sender 可交给普通 sync_wait，不需要存在多个待汇总任务。

它只转换成功通道，原有错误、停止仍向下游转发；构造 variant 时抛出的异常会转为错误完成。它不额外创建线程、不切换 scheduler。

## 4. when_all_with_variant：先统一各分支，再汇总

其组合关系是：

```cpp
ex::when_all_with_variant(a, b)
// 等价于：
ex::when_all(ex::into_variant(a), ex::into_variant(b))
```

若 A 可能发送 int 或 string，B 固定发送 double，成功完成后交给下游的是两个 variant：

```text
A 的结果：variant<tuple<int>, tuple<string>>
B 的结果：variant<tuple<double>>
```

即使 B 只有一种形状，也会被包装。sync_wait 返回的结果存储相应为：

```text
optional<tuple<variant_A, variant_B>>
```

它不是将所有分支合成一个大 variant，也不会只返回先完成的分支；并发、错误和停止行为沿用 when_all。

## 5. 完整例子：一个 sender 的两种成功形状

下面先执行可能抛异常的计算，再用 let_error 将错误恢复成 string。于是 make_source 返回的 sender 可能成功发送 int，也可能成功发送 string。这里的 string 是恢复后的成功值，不再是错误通知。

```cpp
#include <stdexec/execution.hpp>
#include <exception>
#include <iostream>
#include <stdexcept>
#include <string>
#include <tuple>
#include <utility>
#include <variant>

namespace ex = stdexec;

auto make_source(bool fail) {
    return ex::just(fail)
         | ex::then([](bool fail) -> int {
               if (fail) throw std::runtime_error("calculation failed");
               return 42;
           })
         | ex::let_error([](std::exception_ptr) {
               return ex::just(std::string{"fallback"});
           });
}

void print_value(const auto& value) {
    std::visit([](const auto& values) {
        std::cout << std::get<0>(values) << '\n';
    }, value);
}

int main() {
    // 单个 sender 转换形状，无需 when_all。
    auto one = make_source(true) | ex::into_variant();
    auto [value] = ex::sync_wait(std::move(one)).value();
    print_value(value); // fallback

    // 多个 sender：分别转换形状，再汇总。
    auto all = ex::when_all_with_variant(make_source(false), ex::just(2.5));
    auto [a, b] = ex::sync_wait(std::move(all)).value();
    print_value(a); // 42
    print_value(b); // 2.5
}
```

保存为 `/tmp/composition-demo.cpp` 后，在仓库根目录编译运行：

```bash
g++-14 -std=c++20 -pthread \
  -D_GLIBCXX_USE_TBB_PAR_BACKEND=0 \
  -I stdexec/include /tmp/composition-demo.cpp -o /tmp/composition-demo
/tmp/composition-demo
```

不能简单写一个返回类型推导的 lambda，在两个运行时分支中分别 return just(42) 和 just(string{"x"})：这两个表达式的 C++ 类型不同，lambda 的返回类型推导不会自动创建多形状 sender。

## 6. 选择与源码

| 需求 | 接口 |
|---|---|
| 汇总多个 sender，各自没有多种成功形状 | when_all |
| 统一单个 sender 的多种成功形状 | into_variant |
| 汇总多个 sender，输入可能有多种成功形状 | when_all_with_variant |

into_variant 是形状转换工具，when_all_with_variant 是将该工具与 when_all 组合后的便捷接口。

- [when_all 与 when_all_with_variant](../stdexec/include/stdexec/__detail/__when_all.hpp)：子操作启动、完成计数、错误停止处理，以及 into_variant 的组合展开。
- [into_variant](../stdexec/include/stdexec/__detail/__into_variant.hpp)：成功值打包为 variant<tuple<...>, ...>，其他完成通道转发。
- [执行流程](<4 senderExecutionFlow.md>)与[调度适配器](<5 SchedulingAdaptors.md>)：连接、启动和执行资源如何配合。
