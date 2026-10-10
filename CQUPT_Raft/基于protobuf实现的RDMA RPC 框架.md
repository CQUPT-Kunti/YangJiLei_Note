
# 实现RdmaConnection

通过public google::protobuf::RpcChannel 实现一个 RdmaConnection
重要实现CallMethod。（把一次 Protobuf RPC 调用，转换成 FastBlock 自己能处理的 `rpc_request`，并交给后面的 RDMA 发送流程。）

# 实现RdmaController

fastblocks 有controller，但 FastBlock 没有自己重新定义一套 Controller，而是使用的protobuf自带的，
