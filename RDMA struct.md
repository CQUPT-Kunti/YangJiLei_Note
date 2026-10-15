# rpc_request

定义位置：`src/include/fastblock/msg/rdma/client.h`

|字段|作用|
|---|---|
|`request_key`|本次 RPC 的关联 ID，用来匹配请求和响应|
|`method`|要调用的 protobuf 方法描述符|
|`ctrlr`|RPC 控制器，保存错误、连接等控制信息|
|`request`|protobuf 请求对象|
|`response`|protobuf 响应对象|
|`closure`|RPC 完成后的回调函数|
|`meta`|请求的 metadata 缓冲区，保存服务名、方法名等信息|
|`request_data`|请求传输数据，负责组织请求的 meta/data buffer 和 WR|
|`reply_data`|响应传输数据，负责接收和解析服务端响应|
|`start_at`|请求开始时间，用于超时判断和耗时统计|

整体关系：

```
rpc_request
├── request_key       → 请求/响应匹配
├── method            → 调用哪个 RPC 方法
├── ctrlr             → 控制信息
├── request           → 请求 protobuf
├── response          → 响应 protobuf
├── closure           → 完成回调
├── meta              → RPC 元数据
├── request_data      → 请求传输内容
├── reply_data        → 响应传输内容
└── start_at          → 超时/耗时
```
