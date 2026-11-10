# 发送器适配器

## 适配器定义

**原文定义：**

> *Adaptors take an existing sender and produce a new sender with additional behavior. They sit in the middle of a pipeline.*

适应器是一个函数，它接受一个发送器并返回一个新的发送器，新的发送器具有额外的行为。它位于处理流程的中间环节

## then系列适配器

- `then`           - 转换一个值
- `upon_error`     - 处理错误
- `upon_stopped`   - 处理停止

在官方文档对这三者的定义是相当复杂的，但其实简单来说就是，接收一个发送器以及一个函数，将发送器的值转换为新的值，或者处理发送器的错误或停止信号。

### 执行逻辑

要知道使用一个适配器的执行逻辑，需要提前和发送者消费者的执行逻辑结合起来理解。

#### 构造期

构造期是适配器的执行逻辑的第一阶段，它在发送器被构造时执行。这个时期是使用发送器和发送器适配器构造发送器的过程。

```cpp
auto s = stdexec::just(21) | stdexec::then([](int i) { return i * 2; });
// ↑ 构造期：只是把 just、then 和 f 存进嵌套对象，f 没有被调用
```

接下来其实属于发送器消费者的一个时期，但是为了更细所以我拆成了两个时期：

#### 连接期

连接期是适配器的执行逻辑的第二阶段，它在发送器被连接时执行。这个时期是将发送器和接收器连接起来的过程。

```cpp
auto op = stdexec::connect(s, some_receiver);
// ↑ connect：搭好操作状态，f 仍然没有被调用
```

#### 开始期

执行期是适配器的执行逻辑的第三阶段，它在发送器被执行时执行。这个时期是将发送器的值发送给接收器的过程。

```cpp
stdexec::start(op);
// ↑ start：just 发出 set_value(21)，then 的内部 receiver 收到后调用回调，向下游发送 42
```

## let系列适配器

- let_value
- let_error
- let_stopped

then 和 let_value 与上游组合后，都返回 sender。区别在于如何处理回调的返回值：then 把返回值作为普通值传给下游；let_value 要求回调返回 sender，连接并启动它，再把它的完成结果传给下游。let_error 和 let_stopped 分别对错误、停止通道执行类似的接续。

为什么 then 和 let_value 最后都能通过 sync_wait 获得普通值？因为 sync_wait 接收的是整条链的完成结果。接续内部 sender 是 let_value 的职责，不是 sync_wait 递归执行返回的 sender。

let_value 会启动回调返回的 sender，并把它产生的值、错误或停止信号传给后续步骤。

例如（以下是执行链片段，构造后还需连接并启动，例如交给 sync_wait）：
```cpp
ex::just(1)
| ex::let_value([](int& x) {
    return ex::just(x + 1);
})
| ex::then([](int value) {
    std::cout << value; // 输出 2
});
```
这条链被启动后：

1. 上游 just(1) 产生 1。
2. let_value 调用回调，得到 just(2) 这个 sender。
3. let_value 连接并启动 just(2)。
4. just(2) 产生 2，传给后面的 then。

消费者会提供最终 receiver，再通过 connect 得到 operation state 并调用 start。receiver 接收完成信号，不提供 future.get() 这样的统一取值接口；对于 sync_wait，最终 receiver 保存结果并结束等待，最后由 sync_wait 返回结果。具体对象结构见 [内部执行流程](senderExecutionFlow.md)，执行资源切换见 [调度适配器](SchedulingAdaptors.md)。
