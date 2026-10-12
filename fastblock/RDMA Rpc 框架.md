
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


