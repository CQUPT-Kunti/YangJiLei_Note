
# memory_pool.h 
普通内存池

# ring_write_pool.h
高速内存池

# class event_channel
封装 RDMA CM 的事件通道，用来监听连接建立、连接断开等连接管理事件。

主要功能：

```
1. 创建 RDMA event channel
2. 保存 event channel 的 fd
3. 用 poll() 检查是否有 CM 事件
4. 用 rdma_get_cm_event() 取出具体事件
5. 销毁 event channel
```

核心流程：

```
rdma_create_event_channel()
        ↓
得到 channel->fd
        ↓
poll(fd)
        ↓
rdma_get_cm_event()
        ↓
返回 rdma_cm_event
```

监听的事件包括：

```
地址解析完成
路由解析完成
收到连接请求
连接建立完成
连接断开
连接失败
```

字段
```
::rdma_event_channel* _channel{};
::pollfd _fd{};
```

# socket
FastBlock 对一条 RDMA 连接的封装。

它主要负责：

```
1. 保存连接配置 endpoint
2. 创建 rdma_cm_id
3. 解析地址和路由
4. 创建并管理 QP
5. 发起连接或接受连接
6. 提交 SEND/RECV/RDMA WR
7. 处理连接状态变化
8. 关闭并释放连接资源
```

核心关系：

```
endpoint
    ↓
socket
    ├── rdma_cm_id
    ├── QP
    ├── PD
    └── provider
```

它主要负责：

```
1. 保存连接配置 endpoint
2. 创建 rdma_cm_id
3. 解析地址和路由
4. 创建并管理 QP
5. 发起连接或接受连接
6. 提交 SEND/RECV/RDMA WR
7. 处理连接状态变化
8. 关闭并释放连接资源
```

核心关系：

```
endpoint
    ↓
socket
    ├── rdma_cm_id
    ├── QP
    ├── PD
    └── provider
```

发送和接收：


