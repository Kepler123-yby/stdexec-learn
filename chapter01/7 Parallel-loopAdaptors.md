# 并行循环适配器

## 1. bulk 的定义

bulk 在上游成功完成后，对索引范围 `[0, count)` 中的每个索引调用一次函数，并把上游的值继续传给下游。它返回组合后的 sender，构造时不执行循环。

```cpp
// 两种形式等价。
auto s1 = stdexec::bulk(sender, policy, count, f);
auto s2 = sender | stdexec::bulk(policy, count, f);
```

| 参数 | 作用 |
|---|---|
| sender | 上游操作，完成时提供零个或多个值 |
| policy | 执行策略，例如 seq、par；不传给 f |
| count | 非负整数，指定迭代次数；不直接传给 f |
| f | 每次接收索引 i，以及上游值的左值引用 |

假设上游产生 `values...`，执行效果可用下面的顺序伪代码理解，实际调度可能并行：

```text
收到上游 set_value(values...)
    对每个 i ∈ [0, count)：调用 f(i, values...)
    所有调用成功结束后：set_value(下游 receiver, values...)
```

索引类型与 count 的整数类型一致。**f 的返回值被忽略；bulk 不是收集每次返回值的 map，而是让函数通过引用修改数据，再继续传递上游值。** count 为 0 时不调用 f，直接传递上游成功值。

## 2. 上游不传值：通过捕获访问缓冲区

```cpp
// 片段：假设已定义 compute，且 namespace ex = stdexec。
std::vector<int> buf(1024, 0);

auto work = ex::just()
          | ex::bulk(ex::par, buf.size(), [&](std::size_t i) {
                buf[i] = compute(i);
            });
ex::sync_wait(std::move(work)).value();
```

just() 不产生值，所以 f 只有索引参数。buf 由 `[&]` 捕获，不经过 sender 传递。调用方保证它在执行期间有效；本例的 sync_wait 在离开 buf 的作用域前等待完成。

结果写入外部 buf，sync_wait 返回的是有值的 `optional<tuple<>>`，不是 vector。

## 3. 上游持有 vector：通过参数修改并传向下游

下面是可编译的完整例子：

```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <cstddef>
#include <iostream>
#include <utility>
#include <vector>

namespace ex = stdexec;

int compute(std::size_t i) {
    return static_cast<int>(i * i);
}

int main() {
    exec::static_thread_pool pool(4);
    std::vector<int> buf(1024, 0);
    const auto count = buf.size(); // 移动前记录迭代次数

    auto work = ex::starts_on(
                    pool.get_scheduler(), ex::just(std::move(buf)))
              | ex::bulk(ex::par, count,
                         [](std::size_t i, std::vector<int>& data) {
                             data[i] = compute(i);
                         });

    auto [result] = ex::sync_wait(std::move(work)).value();
    std::cout << result.size() << ' '
              << result.front() << ' ' << result.back() << '\n';
    // 1024 0 1046529
}
```

数据流为：

```text
just 移动持有 vector
  → bulk 各次调用接收 i 和同一个 vector 的引用
  → 写入各自索引对应的元素
  → 所有调用成功结束，向下游发送修改后的 vector
  → sync_wait 返回 optional<tuple<vector<int>>>
```

`just(buf)` 复制 vector；`just(std::move(buf))` 将其移动进 sender。移动后不要再用原 buf.size() 决定这次操作的迭代次数，应像示例一样提前保存。

保存完整例子为 `/tmp/bulk-demo.cpp` 后，在仓库根目录运行：

```bash
g++-14 -std=c++20 -pthread \
  -D_GLIBCXX_USE_TBB_PAR_BACKEND=0 \
  -I stdexec/include /tmp/bulk-demo.cpp -o /tmp/bulk-demo
/tmp/bulk-demo
```

## 4. 策略与 scheduler 的分工

| 策略 | 含义 |
|---|---|
| stdexec::seq | 顺序执行各次调用 |
| stdexec::par | 允许各次调用并行，不保证一定并行 |
| stdexec::unseq | 允许非顺序执行，例如向量化 |
| stdexec::par_unseq | 允许并行和非顺序执行 |

策略描述允许的执行方式，scheduler 及其 domain 提供具体执行资源和实现。**par 不是 scheduler，也不会自行创建一个线程池。**

直接使用 `just() | bulk(par, ...)` 不能据此认定计算会多线程执行；本仓库默认通用实现可以顺序运行。第 3 节显式使用线程池 scheduler，使 bulk 可以使用该资源的并行实现。

并行调用共享同一组上游值，不会给每次调用复制一份 vector。对普通 vector<int> 的不同元素分别写入可以并行进行；不要无同步地写同一元素，或并发调用 push_back、resize 等修改容器结构的操作。unseq/par_unseq 还有非顺序执行的约束，不能随意在回调中使用锁等同步操作。

## 5. bulk_chunked 与 bulk_unchunked

两者的参数形式与 bulk 相同，区别在于调用粒度：

| 接口 | 回调形式 | 使用场景 |
|---|---|---|
| bulk | f(i, values&...)，允许内部分块优化 | 普通批量计算，优先使用 |
| bulk_chunked | f(begin, end, values&...) | 自己处理区间，复用每块的临时缓冲或准备工作 |
| bulk_unchunked | f(i, values&...)，不授予分块执行许可 | 确实需要严格逐索引执行语义时使用 |

第 3 节的 bulk 管道段可以替换为以下任意一种，其他代码不变：

```cpp
| ex::bulk_chunked(ex::par, count,
                  [](std::size_t begin, std::size_t end, std::vector<int>& data) {
                      // 每块共用的准备工作可放在循环外。
                      for (auto i = begin; i < end; ++i) data[i] = compute(i);
                  })
```

```cpp
| ex::bulk_unchunked(ex::par, count,
                    [](std::size_t i, std::vector<int>& data) {
                        data[i] = compute(i);
                    })
```

chunked 的区间是 `[begin, end)`，块数和大小由实现决定；成功执行时，每个索引恰好被一个块覆盖。它也可以只产生一个覆盖整个范围的块。

unchunked 不意味着一个索引创建一个线程，也不保证所有迭代同时执行，不能据此让各次调用阻塞等待彼此。三者都忽略回调返回值，成功完成后继续传递上游值。

## 6. 完成与源码

bulk 要等所有迭代成功完成后才向下游发送成功值；它不会持续发送 count 次结果。上游错误或停止时，不执行循环，直接转发完成信号。回调异常会按相应执行实现的异常处理路径处理，不能把异常当作某个迭代的普通返回值继续收集。

与 [when_all](<6 compositionAdaptors.md>) 对比：when_all 组合多个 sender 的完成结果，bulk 则在一个上游 sender 提供的数据上批量调用同一个函数。

- [bulk 实现](../stdexec/include/stdexec/__detail/__bulk.hpp)：回调参数、执行策略，以及向 bulk_chunked 的转换。
- [线程池实现](../stdexec/include/exec/static_thread_pool.hpp)：执行资源及 bulk 调度定制。
- [调度适配器](<5 SchedulingAdaptors.md>)：如何指定启动资源和转移完成上下文。
