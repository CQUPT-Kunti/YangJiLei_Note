
1.首先需要一个PD 保护域，申请QP 和注册MR，和创建其他的RDMA资源都需要 PD。

2.整体流程
```
客户端 A                         服务端 B
    |                                |
    | 目标 IP = 192.168.1.20         |
    |                                |
    v                                |
rdma_resolve_addr()                  |
    |                                |
    +-- 选择本地 RDMA 设备           |
    +-- 确定本地源地址               |
    |                                |
    v                                |
rdma_resolve_route()                 |
    |                                |
    +-- 解析到对端的网络路径         |
    +-- 获得路径相关参数             |
    |                                |
    v                                |
ibv_alloc_pd()                       |
    |                                |
    +-- 创建本地保护域 PD            |
    |                                |
    v                                |
rdma_create_qp()                     |
    |                                |
    +-- 创建本地 QP                  |
    |                                |
    v                                |
rdma_connect()                       |
    |                                |
    +-------- 连接请求 ------------->|
    |<------- 连接响应 --------------|
    |                                |
    v                                v
           RDMA RC 连接建立
```

记住三个区别：

- Address：确定使用哪个本地 RDMA 设备和地址。
- Route：确定如何通过 RDMA 网络到达对端。
- PD：本地资源的保护域，和网络路径解析没有关系。


--------------------------------------------------------------

| 看到什么                      | 说明现在在做什么                 |
| ------------------------- | ------------------------ |
| `rdma_event_channel`      | 处理连接管理事件                 |
| `rdma_cm_id`              | 操作某个 RDMA 端点或连接          |
| `rdma_accept()`           | 接受连接                     |
| `rdma_create_qp()`        | 创建数据传输所需的 QP             |
| `ibv_post_recv()`         | 提前准备接收缓冲区                |
| `ibv_post_send()`         | 提交发送或 RDMA READ/WRITE WR |
| `ibv_poll_cq()`           | 获取网卡完成结果                 |
| `cqe->opcode`             | 判断哪个类型的 WR 完成            |
| `push_back()`             | 把任务交给某个后续处理阶段            |
| `front()` / `pop_front()` | 当前阶段取出任务处理               |

rdma_listen
rdma_accept
ibv_post_recv
receive