# FastBlock 结构体与缩写对照表

> 用途：读 `route*.md` 时快速查“这个结构体/缩写是什么、在哪一层、负责什么”。

相关：[[系统架构]]、[[FastBlock 第一阶段复习：上层 IO → PG]]、[[SPDK]]、[[NVME]]、[[函数名称]]

## 一句话主线

```text
Application / VM
↓
Client
↓
Object / PG / Leader OSD
↓
RPC / RDMA
↓
OSD service
↓
osd_stm
↓
Raft
↓
LocalStore
↓
SPDK Blobstore
↓
NVMe SSD
```

---

## Client 侧结构体

| 名称 | 位置 | 属于哪层 | 作用 |
|---|---|---|---|
| `bdev_fastblock` | [bdev_fastblock.cc](../../fastblock/src/bdev/bdev_fastblock.cc#L38) | SPDK bdev / Client 入口 | 一个 FastBlock bdev 的上下文，保存 `pool_id`、`image_name`、`block_size` 等信息。 |
| `libblk_client` | [libfblock.h](../../fastblock/src/include/fastblock/client/libfblock.h) | Client | 上层块 IO 的入口，把 image offset 拆成 object 写。 |
| `fblock_client` | [fb_client.h](../../fastblock/src/include/fastblock/client/fb_client.h) | Client | 真正负责 PG 映射、Leader 缓存、请求队列、RPC stub、response 处理。 |
| `request_stack_type` | [fb_client.h](../../fastblock/src/include/fastblock/client/fb_client.h#L79) | Client 异步请求 | 一个请求的“现场保存包”：保存 request、response、callback、ctx、stub、leader key。 |
| `leader_request_stack_type` | [fb_client.h](../../fastblock/src/include/fastblock/client/fb_client.h#L67) | Client Leader 查询 | 查询某个 PG Leader 时使用的上下文，保存 get_leader 请求、响应、目标 OSD。 |
| `leader_osd_info` | [fb_client.h](../../fastblock/src/include/fastblock/client/fb_client.h#L116) | Client Leader cache | 缓存某个 PG 当前 Leader 的 `leader_id / addr / port / is_valid / is_onflight`。 |
| `write_source` | [libfblock.cc](../../fastblock/src/client/libfblock.cc#L114) | Client completion | 一个上层 IO 可能拆成多个 Object 写，`write_source` 用来聚合所有 Object 的完成。 |
| `rpc_service_osd_Stub` | [osd_msg.pb.h](../../fastblock/src/include/fastblock/rpc/osd_msg.pb.h#L5579) | Protobuf RPC | Client 侧调用 OSD 服务的本地代理，内部把调用转给 `RpcChannel::CallMethod()`。 |

---

## OSD / PG 结构体

| 名称 | 位置 | 属于哪层 | 作用 |
|---|---|---|---|
| `osd_service` | [osd_service.h](../../fastblock/src/osd/osd_service.h#L29) | OSD RPC service | OSD 端业务 RPC 入口，处理 `process_write/read/delete/get_leader` 等请求。 |
| `partition_manager` | [partition_manager.h](../../fastblock/src/osd/partition_manager.h) | OSD PG 管理 | 管 PG 到 shard、Raft 实例、状态机的映射。 |
| `_shard_table` | [partition_manager.h](../../fastblock/src/osd/partition_manager.h#L154) | OSD PG 管理 | `PG name -> shard_id`，判断一个 PG 在哪个 core 上。 |
| `_pgs` | [partition_manager.h](../../fastblock/src/osd/partition_manager.h#L152) | OSD PG 管理 | 保存 PG 对应的 `raft_server_t`。 |
| `_sm_table` | [partition_manager.h](../../fastblock/src/osd/partition_manager.h#L157) | OSD PG 管理 | 保存 PG 对应的 `osd_stm`。 |
| `osd_stm` | [osd_stm.h](../../fastblock/src/osd/osd_stm.h#L218) | OSD State Machine | 一个 PG 的状态机，把 write 请求封装成 Raft entry，commit 后 apply 到 object_store。 |
| `osd_service_complete` | [osd_stm.cc](../../fastblock/src/osd/osd_stm.cc#L108) | OSD completion | OSD write 的完成回调，负责填 response、`done->Run()`、释放对象锁。 |
| `write_request` | [osd_msg.proto](../../fastblock/proto/osd_msg.proto#L16) | OSD RPC message | Client 发给 OSD 的写请求：`pool_id / pg_id / object_name / offset / data`。 |
| `write_cmd` | [osd_msg.proto](../../fastblock/proto/osd_msg.proto#L58) | Raft log meta | 写入 Raft entry 的元信息：`object_name / offset`。 |

---

## Raft 结构体

| 名称 | 位置 | 属于哪层 | 作用 |
|---|---|---|---|
| `raft_server_t` | [raft.h](../../fastblock/src/raft/raft.h) | Raft | 一个 PG 在某个 OSD 上的 Raft 实例，保存 term、commit index、leader/follower 状态等。 |
| `raft_node` | [raft_node.h](../../fastblock/src/raft/raft_node.h#L26) | Raft peer 状态 | Leader 眼里的一个 follower，保存 `next_idx / match_idx / lease`。 |
| `raft_entry_t` | [raft_msg.proto](../../fastblock/proto/raft_msg.proto#L22) | Raft log entry | 一条 Raft 日志，字段是 `term / idx / type / meta / data`。 |
| `raft_log` | [raft_log.h](../../fastblock/src/raft/raft_log.h#L26) | Raft log 管理 | 管内存 entry cache、`_next_idx`，并桥接到 `disk_log` 落盘。 |
| `entry_cache` | [raft_cache.h](../../fastblock/src/raft/raft_cache.h#L24) | Raft log cache | 内存日志缓存，同时保存 entry 对应的 complete callback。 |
| `msg_appendentries_t` | [raft_msg.proto](../../fastblock/proto/raft_msg.proto#L39) | Raft RPC message | Leader 发给 Follower 的 AppendEntries 请求。 |
| `msg_appendentries_response_t` | [raft_msg.proto](../../fastblock/proto/raft_msg.proto#L69) | Raft RPC message | Follower 回给 Leader 的 AppendEntries 响应，带 `success / current_idx / term` 等。 |
| `state_machine` | [state_machine.h](../../fastblock/src/raft/state_machine.h#L23) | Raft apply | Raft 状态机基类，commit 后由 apply poller 调 `apply()`。 |
| `apply_complete` | [state_machine.cc](../../fastblock/src/raft/state_machine.cc#L38) | Raft apply completion | apply 完成后推进 `last_applied_idx`，并触发原始请求完成。 |
| `pg_group_t` | [pg_group.h](../../fastblock/src/raft/pg_group.h) | PG / Raft 管理 | 管一个 OSD 上多个 PG 对应的 Raft 实例。 |

---

## LocalStore 结构体

| 名称 | 位置 | 属于哪层 | 作用 |
|---|---|---|---|
| `storage_manager` | [storage_manager.h](../../fastblock/src/localstore/storage_manager.h) | LocalStore 总入口 | 每个 core 一个，持有 kvstore、各 PG 的 disk_log 和 object_store。 |
| `disk_log` | [disk_log.h](../../fastblock/src/localstore/disk_log.h#L72) | Raft log 持久化 | 一个 PG 一个，负责把 Raft log 写入环形 log blob。 |
| `rolling_blob` | [rolling_blob.h](../../fastblock/src/localstore/rolling_blob.h) | LocalStore / SPDK blob | 环形 blob 封装，支持 append、read、trim，底层调用 `spdk_blob_io_writev/readv`。 |
| `log_entry_t` | [log_entry.h](../../fastblock/src/localstore/log_entry.h#L21) | 磁盘日志格式 | disk_log 真正落盘的日志格式：header + data。 |
| `object_store` | [object_store.h](../../fastblock/src/localstore/object_store.h#L30) | Object 数据持久化 | 一个 PG 一个，负责 object 到 blob 的映射、读写、RMW。 |
| `object_store::table` | [object_store.h](../../fastblock/src/localstore/object_store.h#L146) | Object 映射表 | 内存表：`object_name -> object/blob`，重启时靠 blob xattr 重建。 |
| `fb_blob` | [types.h](../../fastblock/src/localstore/types.h#L27) | SPDK blob 句柄 | 保存 `spdk_blob*` 和 `spdk_blob_id`。 |
| `blob_tree` | [blob_manager.h](../../fastblock/src/localstore/blob_manager.h#L34) | Blob 管理 | 每个 core 的 blob 目录：kv blob、log blobs、object blobs。 |
| `kvstore` | [kv_store.h](../../fastblock/src/localstore/kv_store.h#L71) | 元数据 KV | 保存 Raft/PG 元数据，如 term、vote_for、node_cfg、last_apply_index。 |
| `blob_rw_ctx` | [object_store.cc](../../fastblock/src/localstore/object_store.cc#L615) | Object IO context | object_store 异步读写时保存现场，SPDK 回调时靠它恢复请求。 |

---

## RPC / RDMA 结构体

| 名称 | 位置 | 属于哪层 | 作用 |
|---|---|---|---|
| `msg::rdma::client` | [client.h](../../fastblock/src/include/fastblock/msg/rdma/client.h) | RDMA client | 管 RDMA 连接、CQ polling、memory pool、请求发送。 |
| `msg::rdma::client::connection` | [client.h](../../fastblock/src/include/fastblock/msg/rdma/client.h#L152) | RDMA connection / RpcChannel | 一条 RDMA 连接，同时实现 protobuf `RpcChannel`。 |
| `msg::rdma::server` | [server.h](../../fastblock/src/include/fastblock/msg/rdma/server.h) | RDMA server | 监听连接、收包、反序列化、按 service/method 分发。 |
| `rpc_request` | [client.h](../../fastblock/src/include/fastblock/msg/rdma/client.h#L156) | RPC 请求现场 | 保存 req_key、method、controller、request、response、closure、metadata。 |
| `request_meta` | [types.h](../../fastblock/src/include/fastblock/msg/rdma/types.h#L82) | RPC header | RPC 头：`service_name / method_name / data_size`。 |
| `transport_data` | [transport_data.h](../../fastblock/src/include/fastblock/msg/rdma/transport_data.h) | RDMA payload 管理 | 负责序列化 protobuf、组织 metadata、SGE、WR，大消息还会记录远端读写信息。 |
| `memory_pool` | [memory_pool.h](../../fastblock/src/include/fastblock/msg/rdma/memory_pool.h#L31) | RDMA 内存池 | 启动/建连时分配并注册 MR，请求时借 buffer，用完归还。 |
| `net_context` | [memory_pool.h](../../fastblock/src/include/fastblock/msg/rdma/memory_pool.h#L35) | RDMA buffer 单元 | 一个预注册 buffer，带 `addr / mr / sge / wr`。 |
| `socket` | [socket.h](../../fastblock/src/include/fastblock/msg/rdma/socket.h) | RDMA verbs 封装 | 单条连接的 QP 管理，真正调用 `ibv_post_send/recv`。 |
| `completion_queue` | [cq.h](../../fastblock/src/include/fastblock/msg/rdma/cq.h#L34) | RDMA CQ | 封装 `ibv_create_cq` 和 `ibv_poll_cq`。 |
| `rpc_controller` | [rpc_controller.h](../../fastblock/src/include/fastblock/msg/rpc_controller.h#L22) | Protobuf RPC 状态 | 保存一次 RPC 的失败状态、错误文本、PD/peer 信息等。 |
| `connect_cache` | [connect_cache.h](../../fastblock/src/include/fastblock/rpc/connect_cache.h) | OSD 间连接缓存 | OSD 侧缓存到其他 OSD 的 RDMA connection / raft stub。 |
| `ring_write_context` | [fb_client.h](../../fastblock/src/include/fastblock/client/fb_client.h#L87) | Write Ring | Client write ring 路径的临时上下文，析构时 dereg/free 临时 DMA buffer。 |
| `write_ring_slot` | [osd_service.cc](../../fastblock/src/osd/osd_service.cc#L61) | Write Ring | OSD 侧预注册的远端写入 slot，Client 用 RDMA WRITE 把数据写进来。 |

---

## SPDK / NVMe 相关名词

| 名称          | 全称                                        | 作用                                                          |
| ----------- | ----------------------------------------- | ----------------------------------------------------------- |
| SPDK        | Storage Performance Development Kit       | 用户态高性能存储框架，FastBlock 用它做 bdev、blobstore、NVMe IO、poller。     |
| bdev        | Block Device                              | SPDK 的块设备抽象层，统一 NVMe、文件、virtio 等后端。                         |
| Blob        | -                                         | SPDK Blobstore 里的逻辑大文件，不等于物理连续空间。                           |
| Blobstore   | -                                         | SPDK 的 blob 管理层，负责 blob 创建、删除、xattr、逻辑 offset 到底层 bdev 的映射。 |
| xattr       | extended attribute                        | blob 上的扩展属性，FastBlock 用它记录 object name、PG、blob type。        |
| DMA         | Direct Memory Access                      | 设备直接访问内存，不经过 CPU 拷贝数据。                                      |
| DMA buffer  | -                                         | 设备可访问、pinned、对齐的内存，FastBlock 常用 `spdk_zmalloc()` 分配。        |
| Reactor     | -                                         | SPDK 每个 core 上的事件循环，反复执行 poller。                            |
| SPDK Thread | -                                         | SPDK 的轻量事件线程，不是 pthread，用 `spdk_thread_send_msg` 投递任务。      |
| Poller      | -                                         | 注册到 SPDK thread 上的回调，周期 0 表示每轮 reactor 都执行。                 |
| IO Channel  | -                                         | SPDK per-thread 访问设备的上下文，减少跨核锁。                             |
| NVMe        | Non-Volatile Memory Express               | 主机和 SSD 通信的协议。                                              |
| SQ          | Submission Queue                          | NVMe 提交队列，主机把命令放进去。                                         |
| CQ          | Completion Queue                          | NVMe 完成队列，SSD 完成 IO 后写 CQE。                                 |
| CQE         | Completion Queue Entry                    | 完成队列里的一条完成记录。                                               |
| PRP         | Physical Region Page                      | NVMe 描述主机内存页的方式。                                            |
| SGL         | Scatter-Gather List                       | NVMe/RDMA 中描述多段内存的方式。                                       |
| PCIe        | Peripheral Component Interconnect Express | CPU/内存和 NVMe/RNIC 之间的高速总线，DMA 走这里。                          |
| RNIC        | RDMA Network Interface Card               | RDMA 网卡，能直接 DMA 读写注册内存。                                     |

---

## RDMA 缩写

| 缩写         | 全称                          | 在 FastBlock 里的作用                                                  |
| ---------- | --------------------------- | ----------------------------------------------------------------- |
| RDMA       | Remote Direct Memory Access | 远端直接内存访问，FastBlock 的 Client-OSD、OSD-OSD 网络通信底层。                   |
| QP         | Queue Pair                  | 一条 RDMA connection 的发送/接收队列对。                                     |
| SQ         | Send Queue                  | QP 里的发送队列，`ibv_post_send` 把 WR 放进去。                               |
| RQ         | Receive Queue               | QP 里的接收队列，需要提前 `ibv_post_recv`。                                   |
| CQ         | Completion Queue            | RDMA 完成队列，send/recv/read/write 完成后产生 CQE。                         |
| CQE        | Completion Queue Entry      | RDMA 完成事件，FastBlock poll CQ 后按 opcode 分发。                         |
| WR         | Work Request                | 交给 RNIC 的任务描述，如 SEND、RECV、RDMA_READ、RDMA_WRITE。                   |
| SGE        | Scatter-Gather Element      | WR 里描述一段本地内存：`addr / length / lkey`。                              |
| MR         | Memory Region               | 注册给 RNIC 的内存区域，注册后才有 `lkey/rkey`。                                 |
| PD         | Protection Domain           | RDMA 保护域，MR、QP 等资源属于某个 PD。                                        |
| lkey       | Local Key                   | 本地访问 MR 的 key，SGE 里用。                                             |
| rkey       | Remote Key                  | 远端访问 MR 的 key，RDMA READ/WRITE 需要。                                 |
| SEND       | -                           | RDMA send，需要对端提前 post receive。FastBlock 普通 RPC 的 metadata 走 SEND。 |
| RECV       | Receive                     | 接收 SEND 的 WR，提前挂到 RQ 上。                                           |
| RDMA READ  | -                           | 主动从远端 MR 拉数据。FastBlock 普通大 RPC payload 会用它。                       |
| RDMA WRITE | -                           | 主动写远端 MR。FastBlock write ring 用它把数据写到 OSD slot。                   |

---

## 容易混的几组

| 名称 | 区别 |
|---|---|
| Object vs Blob | Object 是 FastBlock 的逻辑对象；Blob 是 SPDK Blobstore 的逻辑存储单元。当前 Object 数据基本是一对象一 blob。 |
| PG vs Raft Group | FastBlock 里一个 PG 对应一个 Raft Group；同一个 PG 在多个 OSD 上各有一个 `raft_server_t`。 |
| `disk_log` vs `object_store` | `disk_log` 保存 Raft 操作日志；`object_store` 保存 commit/apply 后真正的对象数据。 |
| Append vs Commit vs Apply | Append 是日志进入本节点；Commit 是多数派确认；Apply 是状态机真正执行。 |
| Stub vs RpcChannel vs Connection | Stub 决定调用哪个远端方法；RpcChannel 是传输抽象；connection 是 RDMA 连接并实现 RpcChannel。 |
| MR vs SGE vs WR | MR 是注册内存；SGE 是一段内存描述；WR 是交给网卡执行的任务。 |
| CQ vs Poller | CQ 是硬件/verbs 完成队列；poller 是软件循环里负责轮询 CQ 的函数。 |
| SPDK Thread vs pthread | SPDK Thread 是 reactor 内的事件调度对象，不是 OS 线程。 |
| SQ/CQ in NVMe vs SQ/CQ in RDMA | 名字类似，都是提交/完成队列；NVMe 属于 SSD 协议，RDMA 属于网卡 verbs。 |

---

## 路线对应

| 文件                                                       | 主要查什么                                             |
| -------------------------------------------------------- | ------------------------------------------------- |
| [route.md](../../fastblock/route.md)                     | Client 第一段：Image → Object → PG → `send_request()` |
| [route2.md](../../fastblock/route2.md)                   | PG → Leader OSD、Leader cache                      |
| [route3.md](../../fastblock/route3.md)                   | 请求队列、poller、`get_stub()`                          |
| [route4.md](../../fastblock/route4.md)                   | Client RPC/RDMA 发送与 response/completion           |
| [route5.md](../../fastblock/route5.md)                   | OSD 收 write、找 PG、进 `osd_stm`、交给 Raft              |
| [plan/5_src_raft.md](../../fastblock/plan/5_src_raft.md) | Raft propose、replicate、commit、apply               |
| [route6.md](../../fastblock/route6.md)                   | LocalStore：disk_log、object_store、kv_store         |
| [route7.md](../../fastblock/route7.md)                   | SPDK / NVMe / DMA / SQ / CQ                       |
| [route8.md](../../fastblock/route8.md)                   | Protobuf RPC + RDMA 网络层、MR/SGE/WR/QP/CQ           |
