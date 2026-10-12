
# 实现RdmaConnection

通过public google::protobuf::RpcChannel 实现一个 RdmaConnection
重要实现CallMethod。（把一次 Protobuf RPC 调用，转换成 FastBlock 自己能处理的 `rpc_request`，并交给后面的 RDMA 发送流程。）

# 实现RdmaController

fastblocks 有controller，但 FastBlock 没有自己重新定义一套 Controller，而是使用的protobuf自带的，


# PendingRpc 请求管理
发送请求后，客户端怎么记住这个请求？
需要一个 `PendingRpc` 和一个请求管理表，来记录所有已经发送、但还没有收到响应的 RPC。

- 需要什么？ 一个 `PendingRpc` 保存单次 RPC 的上下文，一个 Map 保存所有未完成的 RPC。
    
- 用来做什么？ 给每个 RPC 分配唯一的 `request_id`，把请求与对应的 Controller、Response、回调关联起来。
    
- 保证什么？ 多个 RPC 可以同时执行，并且每个请求都有独立的状态，不会相互混淆。发送失败或连接关闭时，也能找到并结束未完成的请求。
    
- 不这样做会怎么样？ 客户端可能同时发出多个 RPC，但收到响应后不知道它属于哪个请求，也无法可靠清理失败的请求。

# 完成异步 RPC 请求

需要一个响应处理机制，来接收服务端返回的数据，并找到第三阶段保存的对应请求。

- 需要什么？ 一个与 RDMA RECV Completion 对接的 RPC 响应处理机制。
    
- 用来做什么？ 收到 Response 后，通过 `request_id` 找到对应的 `PendingRpc`，把结果交给调用方，并通知 RPC 已经完成。
    
- 保证什么？ 响应不会匹配到错误请求；每个 RPC 只完成一次；即使响应乱序、重复到达或与连接关闭同时发生，也不会重复回调或访问失效对象。
    
- 不这样做会怎么样？ 虽然 RDMA 已经收到响应，但上层 Protobuf RPC 不知道请求完成了，调用方也无法可靠获得结果。
    

第四阶段的本质：建立一个机制，把收到的 Response 正确交还给发起 RPC 的调用方。

# 服务端：接收 RPC，找到方法，执行服务，发送响应

