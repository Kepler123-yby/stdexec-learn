# 任务 07：后台任务统一收尾

**场景：** 主线程提交多项后台任务后继续处理其他工作，退出前统一等它们完成；对其中需要结果的任务，稍后再取值。

**产物：** `chapter03/task07.cpp`。使用线程池和 `stdexec::counting_scope`。

## 本题采用的接口

```cpp
stdexec::counting_scope scope;
// stdexec::spawn(sender, scope.get_token());
// auto future = stdexec::spawn_future(sender, scope.get_token());
// stdexec::sync_wait(scope.join());
```

本题统一用上述接口。仓库另有扩展 `exec::async_scope`，其接口为 `scope.spawn(...)`、`scope.on_empty()`；不要把两套写法混在一个 scope 对象上。

## 需要完成的工作

1. 准备固定大小结果数组，提交 ID 为 `0..5` 的六个任务。每个任务在线程池执行，结果为 `id * id`，写入各自数组槽位；主线程提交后立即继续普通工作。
2. 使用 `spawn` 提交。让 ID 为 3 的任务抛出异常，其余任务成功。
3. 在提交前给每条管线加错误处理，把失败记入该 ID 的状态槽位，并恢复为正常完成；错误处理函数本身不抛异常。
4. 等待 `scope.join()`，随后检查每个状态和结果，最后再销毁 scope、线程池和数组。
5. 用另一个 scope 做 `spawn_future` 实验：提交一个返回 `42` 的任务，保存返回的 sender，在主线程做一段普通工作，之后消费 future 获取结果，最后 join。
6. 用同步信号验证 future 对应的工作在消费 future 前就已启动：worker 发布 started，主线程观察到 started 后再调用 `sync_wait(std::move(future))`。不要仅凭一条日志的偶然先后下结论。

## 约束与提示

- 每个任务按值保存自己的 ID，不引用提交循环的局部计数变量。
- 数组预先分配，每个任务只写自己的槽位，主线程在 join 后读；不要并发 `push_back`，不要使用位压缩的 `vector<bool>` 作为独立状态槽位。
- `spawn` 本身可以立即启动工作，但不会自动切换线程；真正的后台执行仍需显式调度。
- `spawn` 的未处理 error 可能导致终止，必须在管线中处理本题注入的错误。
- 不用主线程 `sleep_for` 猜测任务是否结束，不用 `start_detached` 代替 scope。
- 提交全部任务后再 join，join 完成后不再向这个 scope 提交；future 在正常路径消费后再 join，异常路径也须确保收尾。

## 验收

- [ ] ID 0、1、2、4、5 的结果分别为 `0、1、4、16、25`；ID 3 状态为失败，总计成功 5 项、失败 1 项。
- [ ] 主线程可以在提交后、join 前执行其他代码。
- [ ] join 返回时所有六个任务的状态已确定，之后没有后台任务继续访问数组。
- [ ] future 尚未被消费时，worker 已经发出 started 信号；最终获取结果 `42`。
- [ ] 不依赖 scope 析构函数隐式完成等待。

## 写下你的解释

1. `when_all` 与 scope 各适合管理什么样的任务集合？
2. `spawn_future` 返回 sender，为什么其工作仍可能已经启动？它与普通惰性 sender 有何不同？
3. 如果主线程提前退出，被引用数组或 pool 先销毁，会产生什么生命周期问题？

参考：[counting scope 测试](../stdexec/test/stdexec/types/test_counting_scopes.cpp)、[spawn_future 测试](../stdexec/test/stdexec/algos/adaptors/test_spawn_future.cpp)。完成后进入[任务 08](08-batch-pipeline.md)。
