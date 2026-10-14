```
client / server
│
├── event_channel
│   ├── rdma_event_channel*
│   └── fd
│       └── 只负责监听 RDMA CM 连接事件
│
├── protection_domain
│   └── ibv_pd*
│
├── completion_queue
│   └── ibv_cq*
│
└── connection / connection_record
    │
    ├── socket
    │   ├── rdma_cm_id*
    │   │   └── channel → event_channel
    │   │
    │   ├── QP
    │   │   └── _id->qp
    │   │
    │   ├── PD
    │   └── endpoint
    │
    ├── connection_id
    ├── dispatch_id
    ├── recv buffer / memory_pool
    └── RPC task
        ├── server_id / task_id
        ├── correlation_index
        ├── service_name
        └── method_name

```


```
socket
    ├── 管理一条 RDMA 连接
    ├── 持有 rdma_cm_id
    ├── 持有 QP
    ├── 提交 Send/Recv WR
    └── 处理连接建立和关闭

client/server
    ├── 管理 CQ
    ├── 轮询完成事件
    ├── 处理收到的数据
    └── 执行 RPC 逻辑
```

# endpoint
`endpoint` 可以理解为建立一个 RDMA 连接所需要的地址和配置参数集合

可以按功能分成 5 个模块。

### 1. 地址与监听模块

|参数|意义|
|---|---|
|`addr`|IP 地址或监听地址|
|`port`|RDMA 连接端口|
|`passive`|是否作为服务端监听|
|`backlog`|服务端监听队列长度|

### 2. RDMA 连接建立模块

|参数|意义|
|---|---|
|`resolve_timeout_us`|地址、路由解析超时时间|
|`poll_cm_event_timeout_us`|RDMA CM 事件等待超时时间|

### 3. QP 队列与数据能力模块

|参数|意义|
|---|---|
|`max_send_wr`|发送队列最大 WR 数|
|`max_recv_wr`|接收队列最大 WR 数|
|`max_send_sge`|一个发送 WR 最大 SGE 数|
|`max_recv_sge`|一个接收 WR 最大 SGE 数|
|`max_inline_data`|允许直接 inline 发送的数据大小|

### 4. CQ 与完成通知模块

|参数|意义|
|---|---|
|`cq_num_entries`|CQ 容量|
|`qp_sig_all`|是否让所有 WR 产生完成事件|

### 5. RDMA 设备选择模块

|参数|意义|
|---|---|
|`device_name`|指定 RDMA 设备|
|`device_port`|指定 RDMA 设备端口|
|`gid_index`|指定 GID 索引|


# RDMA CM 连接管理
字段：

```
rdma_cm_id* _id;
rdma_addrinfo* _res;
```

作用：

```
保存 RDMA CM 的连接 ID 和地址解析结果。
```

#  Provider

字段：

```
std::unique_ptr<provider> _provider;
```

作用：

```
封装 QP 创建、连接建立和断开操作。
```

# QP 管理


字段：

```
ibv_qp_init_attr* _qp_init_attr;
ibv_qp_attr* _qp_attr;
```

作用：

```
保存 QP 创建参数和 QP 当前状态。
```

主要负责：

```
创建 QP
设置 Send/Recv 队列容量
设置 SGE 数量
设置 inline 大小
修改 QP 状态
```

# FastBlock：connect_task 的异步事件处理模型

FastBlock 客户端采用 **SPDK 单线程 Poller + RDMA CM 异步事件 + connect_task 状态机** 的连接处理模型。

## 1. 整体执行流程（ASCII 图）

```
                  FastBlock 客户端
                         |
                         v
                 创建 connection
                         |
                         v
                  创建 socket
                         |
                         | rdma_resolve_addr()
                         | 发起异步地址解析
                         v
                 创建 connect_task
                 conn_state =
                   wait_address_resolved
                         |
                         v
                 放入 _connect_tasks
                         |
                         v
================================================
              SPDK Poller 周期性执行
================================================
                         |
                         v
                 handle_cm_event()
                         |
                         v
               process_connect_retry()
                         |
                         | 正常任务直接跳过
                         v
                  _channel.poll()
                         |
                         | 检查 event_channel 的 fd
                         |
              +----------+----------+
              |                     |
           无事件                  有事件
              |                     |
              v                     v
         本轮直接返回       rdma_get_cm_event()
                                    |
                                    v
                              根据 evt->id
                             找到 connect_task
                                    |
                                    v
                           process_connect_task()
                                    |
                                    v
                          根据 conn_state 处理
```

## 2. connect_task 的三个状态

```
[wait_address_resolved]
          |
          | 收到 ADDR_RESOLVED
          | 调用 rdma_resolve_route()
          v
[wait_route_resolved]
          |
          | 收到 ROUTE_RESOLVED
          | 创建 QP
          | 调用 rdma_connect()
          v
[wait_connect_established]
          |
          | 收到 ESTABLISHED
          v
      连接建立成功
          |
          v
  从 _connect_tasks 移除
```

这三个 `wait_*` 状态是 **FastBlock 自己定义的状态**，并不是 RDMA 底层维护的状态。

## 3. 为什么不会出现状态还没更新，下一轮 Poller 就执行？

```
      SPDK 线程                   RDMA 底层
          |                           |
          | fd.resolve_route()        |
          |-------------------------->|
          |                           | 开始解析路由
          |                           |
          |                           | 解析完成
          |                           | 产生 ROUTE_RESOLVED
          |                           | fd 变为可读
          |                           |
          | conn_state =              |
          |   wait_route_resolved     |
          |                           |
          | break                     |
          |                           |
          | 当前回调返回              |
          |                           |
========== 下一轮 SPDK Poller ==========
          |                           |
          | _channel.poll()           |
          |                           |
          | 取出 ROUTE_RESOLVED       |
          |                           |
          | 进入 wait_route_resolved  |
```

**关键理解：**

- `rdma_resolve_route()` 发起异步操作，不会等待解析完成。
    
- RDMA 底层可能在 `conn_state` 更新之前就产生事件。
    
- fd 可读不代表立即执行 `_channel.poll()`。
    
- 同一个 SPDK 线程必须先执行完当前回调，才能进入下一轮 Poller。
    
- 因此，下一轮处理事件时，`conn_state` 已经更新完成。
    

## 4. 阻塞与非阻塞

`_channel.poll()` 使用：

```
poll(&_fd, 1, 0);
```

最后的 `0` 表示 **非阻塞检查**。

- 有事件：立即取出并处理。
    
- 没有事件：立即返回，下一轮再检查。
    
- 如果 Poller 回调内部执行了真正的阻塞操作，会阻塞整个 SPDK 线程。
    
- 这里的三个 `wait_*` 状态只是记录“正在等待什么”，**不会让线程阻塞等待**。
    

## 5. 哪些部分不需要深入？

```
FastBlock                      RDMA 底层
    |                              |
    | rdma_resolve_route()         |
    |----------------------------->|
    |                              | 解析路由
    |                              | 产生事件
    |                              | 使 fd 可读
    |                              |
    | _channel.poll()              |
    |<-----------------------------|
    |                              |
    | 修改 conn_state              |
```

RDMA 库和 Linux RDMA 子系统负责执行解析、产生事件、通知 fd 可读。这部分底层实现暂时不需要学习。

**总结：**`**connect_task**` **不会自己执行，也不是状态改变触发 Poller。SPDK Poller 周期性检查共享的 event_channel，获取事件后找到对应的 connect_task，再根据 FastBlock 自己定义的状态推进连接流程。**


# 6
