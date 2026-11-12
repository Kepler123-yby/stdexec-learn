# 任务 03：同时请求三份数据

**场景：** 展示一个用户页面前，需要用户资料、积分和最近订单三份互不依赖的数据。对比逐个等待与并发汇总。

**产物：** `chapter03/task03.cpp`。使用 `exec::static_thread_pool`。

## 固定输入

| 分支 | 模拟耗时 | 成功结果 |
|---|---|---|
| profile | 300 ms | `std::string{"Ada"}` |
| points | 500 ms | `120` |
| orders | 200 ms | `7` |

实现三个返回 sender 的工厂函数，函数名自定。每个分支用 `starts_on` 或 `schedule` 将工作交给线程池，在回调中记录开始、结束时间并模拟耗时。

## 需要完成的工作

1. **顺序基线：** 在主线程逐个构造并 `sync_wait` 三个分支，记录总耗时。这是本章允许连续三次同步等待的对照实验。
2. **并发版本：** 用 `when_all(profile, points, orders)` 组合新建的三个 sender，只调用一次 `sync_wait`。
3. 在 `when_all` 下游接一个 `then`，接收三份结果，生成 `Ada: points=120, orders=7`。
4. 用 3 个 worker 运行并发版本，再用 1 个 worker 重跑，比较任务时间区间。
5. 移除分支的调度步骤，只保留 `just | then` 的耗时工作，再观察 `when_all` 是否仍有重叠。

## 约束

- `when_all` 的参数顺序固定为 profile、points、orders，不按完成顺序排序。
- 不为每个任务单独创建线程，不使用 `std::async`。
- 必须同时保留各分支的时间区间和总耗时，不能只凭“打印顺序看起来混在一起”判断并发。
- `sleep_for` 在本题中占用 worker；只用于放大时间差。

## 验收

- [ ] 三种执行方式得到相同业务结果。
- [ ] 普通空闲环境中，顺序基线约为 1000 ms，3-worker 并发版趋近最长分支的 500 ms 加调度开销；这些是观察值，不是严格时限。
- [ ] 3-worker 版本日志显示多个分支的执行区间重叠；单 worker 或无调度版没有这些阻塞回调的执行重叠。
- [ ] 即使 orders 通常先完成，结果依然按 profile、points、orders 的输入顺序交给下游。
- [ ] 三个分支分别返回 string、int、int，仍可使用普通 `sync_wait`。

## 写下你的解释

1. `when_all` 提供的是任务组合还是线程资源？
2. 三个分支类型不同，为什么不需要 `sync_wait_with_variant`？
3. 若三个分支都在 `start` 中同步阻塞完成，组合器为什么无法凭空创造并发？

参考：[组合适配器](../chapter01/6%20compositionAdaptors.md)。完成后进入[任务 04](04-dependent-work.md)。
