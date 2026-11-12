# 任务 01：先看清值怎么流动

**场景：** 输入数量和单价，计算订单金额。先把值传递、回调参数和消费者的关系弄清楚，再引入线程。

**产物：** `chapter03/task01.cpp`。本题只需要 `<stdexec/execution.hpp>` 和相应标准库头文件。

## 需要完成的工作

1. 用 `just(3, 20)` 表示数量为 3、单价为 20。金额统一用整数，单位为元。
2. 第一个 `then` 接收这两个参数，返回总价；第二个 `then` 接收总价，减去 5 元优惠。
3. 构造整条 sender 后打印 `pipeline built`；两个回调分别打印执行标记；随后在 `main()` 中调用一次 `sync_wait` 并打印结果。
4. 另建一条只包含 `just(3, 20)` 的管线，直接交给 `sync_wait`，用结构化绑定拿到两个值。
5. 另建 `just()` 管线并等待它，检查返回的 optional 是否有值。

## 约束与语法提示

- 不使用线程池、`std::thread` 或 `std::async`。本题就是同步完成的对照组。
- 第一条管线的两个回调都只做普通值转换。
- 用 `static_assert` 和 `std::is_same_v` 检查前两种结果的完整类型。
- 先检查 optional，再提取结果；观察“零个成功值”与“空 optional”是否相同。
- 在本题最后单独尝试让第一个回调只接收一个 `int`，阅读编译错误，再恢复正确版本。

## 验收

- [ ] 构造阶段没有两个计算回调的执行日志。
- [ ] 最终金额为 `55`，返回类型为 `std::optional<std::tuple<int>>`。
- [ ] 直接消费 `just(3, 20)` 得到 `3` 和 `20`，类型为 `std::optional<std::tuple<int, int>>`。
- [ ] `just()` 成功时 optional 有值，其内部是 `std::tuple<>`。
- [ ] 回调线程 ID 都与调用 `sync_wait` 的主线程相同。
- [ ] 能解释参数个数不匹配时为何编译失败，第二个值为何不会被自动忽略。

## 写下你的解释

1. `then(f)` 的参数由哪里决定？下一段收到的是原始输入还是 `f` 的返回值？
2. “一个 sender 的一种成功形态”与“成功时只传一个值”有何区别？
3. 构造 sender、`connect`、`start` 和 `sync_wait` 分别处在哪一层？

参考：[消费者](../chapter01/8%20SenderConsumers.md)、[适配器](../chapter01/3%20senderAdaptors.md)。完成后进入[任务 02](02-scheduling.md)。
