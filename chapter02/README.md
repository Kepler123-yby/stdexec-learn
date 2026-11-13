# 第三章：stdexec 任务驱动实战

目标：亲手把普通函数组合成异步任务，观察值、完成信号和执行线程如何沿管线流动，并能解释任务何时启动、何时完成、由谁等待和管理生命周期。

本章按 **CPU + C++20 + 本仓库 stdexec** 设计。题目给出场景、约束和验收条件，解题代码由你编写。耗时操作先用本地模拟，不需要服务器或 GPU。接口核对基于 stdexec 子模块提交 `1d58877`；扩展接口使用 `exec::`，具体命名以本地头文件为准。

## 学习路线

| 顺序 | 任务 | 经典模式 | 核心接口 | 建议时间 |
|---|---|---|---|---|
| 01 | [先看清值怎么流动](01-value-flow.md) | 惰性管线、同步消费 | `just`、`then`、`sync_wait` | 30–45 分钟 |
| 02 | [把计算交给工作线程](02-scheduling.md) | 后台执行、线程切换与返回 | `schedule`、`starts_on`、`continues_on`、`on` | 45–60 分钟 |
| 03 | [同时请求三份数据](03-fan-out-fan-in.md) | 并发分发与汇总 | `when_all`、线程池 | 45–60 分钟 |
| 04 | [先查用户，再查订单](04-dependent-work.md) | 有依赖的异步链 | `let_value`、`when_all` | 45–60 分钟 |
| 05 | [请求失败后降级](05-error-recovery.md) | 异常传播、同步与异步恢复 | `upon_error`、`let_error`、`sync_wait_with_variant` | 45–60 分钟 |
| 06 | [给请求设置超时](06-timeout-and-stop.md) | 竞速、定时器、协作取消 | `exec::when_any`、`exec::schedule_after` | 60–90 分钟 |
| 07 | [后台任务统一收尾](07-scoped-work.md) | 提前启动、作用域管理、延后取值 | `counting_scope`、`spawn`、`spawn_future` | 60–90 分钟 |
| 08 | [批量数据处理流水线](08-batch-pipeline.md) | 数据并行与综合任务图 | `bulk`、调度切换、`when_all` | 90–120 分钟 |

任务 01 是语法热身；02 开始使用真实工作线程；06 使用可取消定时器，观察与阻塞 `sleep_for` 的区别。建议按顺序完成，每次只实现一题。

## 文件与构建约定

每题编写一个独立程序，例如 `chapter03/task01.cpp`。每个程序有自己的 `main()`，不要将多个练习一起链接为一个可执行文件。在 `chapter03/notes.md` 中记录每题的实际日志、结论和未解决问题。

以下命令在**仓库根目录**执行，写好对应源文件后即可单独编译，不需要修改根 CMake：

```bash
mkdir -p /tmp/stdexec-learn-ch03
g++-14 -std=c++20 -O0 -g -Wall -Wextra -pthread \
  -D_GLIBCXX_USE_TBB_PAR_BACKEND=0 \
  -I stdexec/include chapter03/task01.cpp \
  -o /tmp/stdexec-learn-ch03/task01
/tmp/stdexec-learn-ch03/task01
```

后续替换文件名和可执行文件名即可。这里的宏用于关闭本机 libstdc++ 的 TBB 并行算法后端；本章使用的 `exec::static_thread_pool` 仍正常工作。

常用头文件按需要包含：

| 功能 | 头文件 |
|---|---|
| sender 算法、counting scope、spawn | `<stdexec/execution.hpp>` |
| 固定大小线程池 | `<exec/static_thread_pool.hpp>` |
| 单线程上下文 | `<exec/single_thread_context.hpp>` |
| 定时器上下文和 scheduler | `<exec/timed_thread_scheduler.hpp>` |
| 竞速组合 | `<exec/when_any.hpp>` |

标准库类型也应包含相应头文件，如 `<chrono>`、`<thread>`、`<tuple>`、`<utility>`、`<vector>`、`<syncstream>`。

## 每题统一要求

1. 先写下预期的数据类型、执行线程和完成信号，再运行验证。
2. 日志至少包含：任务名、阶段、相对起点的毫秒数、线程 ID。使用 `steady_clock` 计时，使用 `osyncstream` 或互斥保护完整日志行。
3. 除题目明确要求的对照实验外，只在 `main()` 的同步边界调用 `sync_wait`，不要在工作线程的 `then` 中阻塞等待另一个 sender。
4. 为每次独立运行重新构造 sender；消费需要移动的 sender 时使用 `std::move`，不要重复启动同一个 operation state。
5. pool、context 和被引用的数据必须活到相关工作完成之后。通过值传递或移动保存异步操作需要的数据，解释每一个跨异步阶段的引用捕获为何有效。
6. 数值和事件依赖关系用于验收；耗时用于辅助观察。操作系统调度可能推迟任务，不要求精确毫秒数、固定 worker ID 或固定的并发日志顺序。
7. 任务 03–05 中的 `sleep_for` 只是占用工作线程的耗时模拟，不是真正的异步 I/O；任务 06 改用异步定时器。

## 每题交付

- 一个可独立编译运行的 `taskNN.cpp`。
- 题目列出的输入和边界情况的运行结果。
- 一张文字任务图，标出数据类型与调度位置。
- 对题末问题的回答，尤其说明“构造 sender”和“启动操作”的区别。

可在自己的 `notes.md` 中按下表记录，不要只保存最终数字：

| 项目 | 记录内容 |
|---|---|
| 运行命令 | 编译和运行命令 |
| 预测 | 哪些任务能重叠，结果类型是什么 |
| 观察 | 日志、数值、线程和耗时 |
| 解释 | 观察与预测一致吗，差异由什么导致 |

## 查阅入口

遇到语法问题先查 [chapter01](../chapter01/)，再查看本仓库的 [官方示例](../stdexec/examples/) 和 [测试](../stdexec/test/)。外部资料可参考 [NVIDIA stdexec 用户指南](https://nvidia.github.io/stdexec/user/index.html)，接口差异以本仓库源码为准。
