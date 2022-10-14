# redis benchmark 分析

## 简介
- redis-benchmark 是用于测试 redis 各种指令的性能。
- 他每次性能测试一个指令（get、set、ping。。。），并且 redis-benchmark 的实现并没有使用 hiredis 中 redisCommand 系列 api 来发送操作数据库的指令，而是借助了 aeEventLoop 来收发指令。
- redis-benchmark 最终会统计出两大类指标，**吞吐量**和**指令请求的时延**。但是这两者并不是简单的倒数关系，因为它们的统计过程不同。
- redis-benchmark 并不能用来模拟日常使用场景的性能。

## redis-benchmark 在测试一次 benchmark 的流程

  1. 将 string 型指令通过 formatCommand 转化成 Resp 协议格式的 string
  2. createClient 创建 client 连接
  3. 注册写事件 writeHandler（向 server 发送指令）
  4. **开始计时，记录这次测试的开始时间（totalLatency）**
  5. 进入 aeMain 中的 EventLoop 等待该 client 的事件通知
  6. 执行 writeHandler，**开始计时、记录这次指令收发的开始时间（c->latency）**
  7. 发送完指令，注册读事件 readHandler（接受处理来自 server 的回复）
  8. 进入 eventLoop，等待该 client 的事件通知
  9. 执行 readHandler，
  
     - 在**刚进入 readHandler 时就记录这次指令收发的结束时间（c->latency）**。
     - 接受处理回复
     - 进入 clientDone, 统计总请求个数+1（request_finished）
     - 若没达到指定的请求个数，恢复 client 的读写事件状态到第 3 步（注册 writeHandle），接着等待下一个 eventLoop 来发送指令。
     - 若达到指定的请求个数，freeClient
  10. 完成这次 benchmark，统计这次 benchmark 的结束事件（totalLatency）

- 由上述流程我们可以看到，它的性能统计过程有几点特殊的地方：
  
  1.  相比直接使用 redisCommand 发送命令来测试性能，redis-benchmark 测试性能的流程中将 formatCommand 这一步排除在统计时延的过程之外。
  2. redis-benchmark 统计单次指令收发的时延时，它仅统计从 client 刚刚开始发送（刚进入 writeHandler），到刚收到来自 server 端回复的事件（刚进入 readHandler），这个过程中的时延。而将读取处理来自 server 端的回复过程排除在外

## 分析结论

- 由以上两点我们可以看出，对于 redis-benchmark 来说，单条指令的收发时延（latency）意味着由以下几部分组成：

  - client 到 server 之间 socket 收发，读写到 redis buffer 中的耗时 * 2（来回）
  - server eventLoop 获取到该 client 的 event 事件的耗时
  - server 端解析 client 指令的耗时
  - server 端执行 client 指令的耗时
  - server 端将 reply 格式化成 resp 协议的耗时

- 而对于吞吐量指标来说，相比起用 redisCommand 发送相同数量的请求，它们之间的耗时组成差异，可大致理解为：

    - redisCommand 总耗时 = benchmark 总耗时 + 请求数量 * 该请求格式化成 resp 协议的耗时 - benchmark aeEventLoop 部分的总耗时。

## 总结

- redis-benchmark 不是为了模拟测试实际使用过程中的性能指标。因为它自己实现了套客户端处理读写的方式，而且一次性能测试只是测试一种指令。
- 它的目的看起来更像是为了测试：

    - redis server 端本身架构的性能（异步事件机制、请求处理）。而对于 client 端来说，因为它的实现借用的 server 侧的实现（aeEventLoop），因此也可以把它看作是一个 server 端，可以测试 ae 模块的性能。
    - Client server 之间网络连接的性能。