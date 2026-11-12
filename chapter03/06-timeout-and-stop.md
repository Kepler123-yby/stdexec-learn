# 任务 06：给请求设置超时

**场景：** 请求和超时定时器竞速，先完成的一方决定结果，其余工作收到停止请求。体会“请求取消”与“强制终止线程”的区别。

**产物：** `chapter03/task06.cpp`。使用 `exec::timed_thread_context`、`exec::schedule_after` 和 `exec::when_any`。

## 数据与接口提示

用同一种业务结果类型区分请求成功与超时，例如：

```cpp
enum class Kind { response, timeout };
struct Outcome {
    Kind kind;
    int value; // response 时为 42，timeout 时为 0
};
```

`exec::schedule_after(timer_scheduler, duration)` 返回定时 sender；成功到期时发送零个值，可在后面的 `then` 中产生 `Outcome`。上下文头文件是 `<exec/timed_thread_scheduler.hpp>`，不是 `timed_thread_context.hpp`。

## 需要完成的工作

1. 请求分支：用定时 sender 模拟等待，完成时产生 `Outcome{Kind::response, 42}`。
2. 超时分支：用另一个定时 sender 等待截止时长，完成时产生 `Outcome{Kind::timeout, 0}`。
3. 用 `exec::when_any` 组合两者，然后用 `sync_wait` 获取最终结果。
4. 分别运行请求 100 ms / 超时 600 ms，以及请求 600 ms / 超时 100 ms。
5. 在两个分支上记录 stopped：可用 `let_stopped` 打印后返回 `just_stopped()`，保持停止信号继续传播。
6. 单独做显式停止实验：创建 `stdexec::inplace_stop_source`，先调用它的 `request_stop()`，再把它的 token 通过 `stdexec::write_env(stdexec::prop{stdexec::get_stop_token, source.get_token()})` 注入一个定时 sender，等待后确认得到空 optional。停止源必须活到操作完成，后续正常请求不要复用这个已停止的 token。

## 对照实验：停止请求无法打断普通阻塞回调

把请求分支改成在线程池 worker 上执行 `sleep_for(600ms)` 后返回 `Outcome`；超时分支仍使用独立定时上下文，期限为 100 ms。

为保证对照有效，让超时定时器等待 worker 发出的“已进入阻塞回调”信号之后再开始计时。对照实验使用至少 2 个 worker：请求回调用 `std::latch::count_down()` 发信号，超时分支先调度到另一个 worker，在 `let_value` 回调中 `wait()` 后才构造并返回定时 sender。不要让定时线程承担这个等待。

记录 winner、worker 结束时间，以及外层 `sync_wait` 返回时间。worker 必须真正进入阻塞回调，才能证明正在执行的 `sleep_for` 没有被强行中断。

## 验收

- [ ] 空闲环境下，第一组返回 response，第二组返回 timeout；两者都使用有值的 `optional<tuple<Outcome>>`。
- [ ] 业务超时是本题定义的普通 value 结果，不能与 stopped 的空 optional 混淆。
- [ ] 可取消定时器版本通常在较短期限附近结束；被取消分支出现 stopped 记录，成功回调不执行。
- [ ] 注入已停止 token 的实验中，`sync_wait` 返回空 optional。
- [ ] 阻塞对照实验中，即使 timeout 已获胜，也要等已进入的 600 ms 回调结束，外层等待才完成。
- [ ] context、pool 和同步对象一直有效，直到组合操作完成。

## 写下你的解释

1. 本仓库的 `when_any` 记录首个完成信号、请求其他分支停止，为什么还需要等待所有分支结束后才向外完成？
2. 为什么不能把 `sleep_for` 当成可取消的异步定时器？
3. 若最先到达的是 error，`when_any` 会继续等后面的成功吗？结合源码解释它与“首个成功”的区别。

时间只作观察，繁忙机器可能有明显调度延迟。参考：[定时器测试](../stdexec/test/exec/test_timed_thread_scheduler.cpp)、[when_any 实现](../stdexec/include/exec/when_any.hpp)。完成后进入[任务 07](07-scoped-work.md)。
