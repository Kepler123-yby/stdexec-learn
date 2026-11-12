# 任务 02：把计算交给工作线程

**场景：** 在后台计算订单金额，再交给另一条线程格式化结果，观察调度边界。

**产物：** `chapter03/task02.cpp`。使用两个 `exec::single_thread_context`，分别命名为 A、B；它们的生命周期覆盖所有等待。

## 需要完成的工作

分别构造并运行三条独立管线，每次输入 `21`，先乘 2，再加 1，结果都为 `43`：

| 管线 | 要求的执行位置 | 要练习的接口 |
|---|---|---|
| 基线 | 两次计算都在主线程 | `just`、`then` |
| 单向转交 | 乘 2 在 A，加 1 在 B，后续观察回调也在 B | `starts_on`、`continues_on` |
| 局部往返 | 从 A 出发，乘 2 在 B，返回 A 后加 1 | `schedule`、`on(target, closure)` |

每条管线都打印：构造时、每次计算时、最终观察回调中，以及 `sync_wait` 返回后的线程 ID。

## 约束与语法提示

- 本版本使用 `starts_on(scheduler, sender)`，不要写成 `sender | starts_on(scheduler)`。
- `continues_on` 可以写成 `sender | continues_on(scheduler)`，它影响交付到下游的完成信号。
- 往返实验先用 `schedule(A)` 建立明确的上游调度位置，再使用 `on(B, then(...))`。
- `schedule(A)` 成功发送零个值；若要生成 `21`，接一个无参数回调。
- 为观察恢复行为，必须在 `on` 之后、`sync_wait` 之前记录线程。
- 不使用 `sleep_for` 推测在哪条线程执行，以线程 ID 为证据。

## 验收

- [ ] 三条管线结果都为 `43`。
- [ ] 单向转交的回调顺序为 A → B → B。
- [ ] 局部往返的回调顺序为 A → B → A。
- [ ] 所有 `sync_wait` 返回后的普通代码都在主线程。
- [ ] 能把两个 context 改成线程池后，解释为何“回到 A scheduler”不再保证回到 A 的同一个 worker。

## 写下你的解释

1. `continues_on(B)` 会把前面已经执行的计算搬到 B 吗？
2. sender 在主线程构造，为什么回调能在工作线程执行？
3. `on` 返回的位置由什么决定？为什么只看 `sync_wait` 返回后的线程无法验证它？

参考：[调度适配器](../chapter01/5%20SchedulingAdaptors.md)。完成后进入[任务 03](03-fan-out-fan-in.md)。
