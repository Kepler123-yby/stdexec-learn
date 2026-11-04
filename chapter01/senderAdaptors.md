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
auto s = stdexec::then(32) | stdexec::then([](int i) { return i * 2; });
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
// ↑ start：just 发出 set_value(21)，then 的 receiver 收到后才调用 f，这时才会打印
```

## let系列适配器

- let_value
- let_error
- let_stopped

这个let系列适配器和then系列适配器的区别在于then返回的值是sender，其他语法几乎一样。

**疑问：** 明明let是返回一个sender，但为什么取的最终返回值的操作和let的取值操作一样

let_value 会启动回调返回的 sender，并把它产生的值、错误或停止信号传给后续步骤。

例如：
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
上游 just(1) 产生 1。
let_value 调用回调，得到 just(2) 这个 sender。
let_value 连接并启动 just(2)。
just(2) 产生 2，传给后面的 then。
