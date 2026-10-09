# FastBlock IO 主流程核心类全景

> 本文是对当前 FastBlock 源码（工作区处于 `1c367e5a spec-kit init`，不含 QoS 实验代码）一次只读审计的产物。
> 目标：从一个 IO 进入 FastBlock 开始，沿真实主流程追到 RPC/RDMA 发送、响应、retry、Logical IO 完成，把主流程中真正承担职责的 **class / struct / protobuf message / 全局对象** 逐个讲清楚。
> 证据格式：`文件::函数/字段 (file:line)`。无法从源码确认的写 `NOT PROVEN FROM CURRENT SOURCE`；SPDK 内部一律标 `BACKGROUND ONLY`。
> 本文不改动任何 FastBlock 源码。

**主流程示例（贯穿全文）**：`Volume A` 上一个 **10 MiB WRITE**，`object_size = 4 MiB`，consumer worker = `T1`。

---

## 1. 主流程类 / 结构体总表

| 类型 / 对象 | 类型类别 | 所属层 | 一句话职责 | Thread Ownership | 生命周期级别 |
|---|---|---|---|---|---|
| `spdk_bdev_io` | SPDK object (BACKGROUND ONLY) | SPDK bdev | 承载一次块 IO（iov/offset/len） | SPDK 框架 | Logical IO 级（SPDK 侧） |
| `bdev_fastblock` | C++ struct | FastBlock bdev | 一个 bdev/volume 的句柄（pool/image/size/object_size） | 非线程；可被多线程读 | Volume/bdev 级 |
| `bdev_fastblock_io_channel` | C++ struct | FastBlock bdev | per-thread channel 上下文（pfd/disk/group_ch） | **per-thread** | channel 级 |
| `bdev_fastblock_group_channel` | C++ struct | FastBlock bdev | 模块级 epoll 组（poller + epoll_fd） | 模块 io_device 级 | 进程级 |
| `spdk_bdev_module` / `spdk_bdev_fn_table` | SPDK 接口表 | FastBlock bdev | 把 FastBlock 函数挂进 SPDK bdev 框架 | 进程 | 进程级 |
| `core_sharded` | C++ class | Shard/线程 | shard 数组 + 每 shard 一个 SPDK thread + `invoke_on` | 全局单例 | 进程级 |
| `global::blk_clients` | global container | Shard/Client | 每 shard（+app thread）一个 `libblk_client` | 进程级共享 vector | 进程级 |
| `global::vhost_worker_threads` | global container | Shard/线程 | shard → worker thread 记录 | 进程级 | 进程级 |
| `utils::simple_poller` | C++ class | 事件驱动 | 把函数注册为 SPDK poller（跨线程安全投递） | 绑定目标 thread | 进程/对象级 |
| `monitor::client` (+`pg_map`/`osd_map`) | C++ class | 控制面 | 提供 PG 数、PG 成员、OSD 信息（路由元数据） | 绑定一个 thread | 进程级 |
| `libblk_client` | C++ class | Logical IO | **API wrapper**：Logical IO ↔ Object 拆分 + parent 聚合 | 对象，绑定 shard | per-thread 级 |
| `write_source` | C++ struct | Logical IO | 一个 Logical WRITE 的 parent completion context | 归属发起线程 | Logical IO 级 |
| `read_source` | C++ struct | Logical IO | 一个 Logical READ 的 parent completion context | 归属发起线程 | Logical IO 级 |
| `fblock_client` | C++ class | Object/Client | 管 Object→PG→Leader→RPC 生命周期与请求队列 | 绑定 `_current_thread` | per-thread 级 |
| `request_stack_type` | C++ struct | Object request | 一个 Object RPC 客户端的完整状态帧 | 归属 `_current_thread` | Object request 级 |
| `leader_key_type` | C++ struct | 路由 | `(pool_id, pg_id)` 打包成 leader key | — | 值类型 |
| `leader_osd_info` | C++ struct | 路由 | PG Leader 缓存（id/addr/port/valid/onflight） | 单写线程（client 线程） | per-thread 级缓存 |
| `write_ring_state` (+`write_ring_slot_info`) | C++ struct | Transport 优化 | write-ring 连接状态与 slot 表 | 单写线程 | connection 级 |
| `osd::write_request` | protobuf request | RPC payload | 发给 OSD 的写请求 5 元组 | — | Object request 级 |
| `osd::write_reply` | protobuf reply | RPC payload | OSD 写响应（state） | — | Object request 级 |
| `osd::read_request` / `read_reply` | protobuf | RPC payload | 读请求/响应（含 data） | — | Object request 级 |
| `rpc_service_osd_Stub` | protobuf 生成类 | RPC | 远端 OSD 服务的本地代理 | — | per-connection 复用 |
| `msg::rdma::rpc_controller` | C++ class | RPC | 传输成败与错误文本 | 随 stack | RPC attempt 级 |
| `msg::rdma::client` | C++ class | Transport | RDMA RPC 客户端（连接池/内存池/CQ poller） | 绑定 thread | 进程/线程级 |
| `msg::rdma::client::connection` | C++ class (`RpcChannel`) | Transport | 一条 RDMA 连接 = RpcChannel（发送/接收/匹配） | 绑定 thread | connection 级 |
| `connection::rpc_request` | C++ struct | Transport | 传输层的单个待发/在途 RPC | connection 线程 | RPC attempt 级 |
| `request_meta` | C++ struct | Transport | RPC 头（service/method/data_size） | — | 单次 RPC |
| `transport_data` | C++ class | Transport | RPC 消息的内存/SGE/WR 组织与序列化 | connection 线程 | RPC attempt 级 |
| `memory_pool` / `net_context` | C++ class/struct | Transport | 预注册 MR 的 DMA 内存池 | connection 线程 | 进程/线程级 |
| `completion_queue` | C++ class | Transport | RDMA CQ 封装（创建/poll） | 绑定设备 | 进程级 |
| `socket` | C++ class | Transport | 单条连接的 QP/verbs/cm 封装 | connection 线程 | connection 级 |
| `osd_service` | C++ class | 服务端入口 | OSD 端 RPC handler（process_write 等） | 绑定 shard thread | 进程级 |

---

## 2. 生命周期层级

```text
进程级
├── core_sharded（shard 数组 + per-shard SPDK thread）
├── global::blk_clients（vector，容量 = core 数 + 1）
├── global::vhost_worker_threads
├── monitor::client
├── completion_queue（RDMA 设备级）
├── bdev_fastblock_group_channel（模块 io_device 级）
└── spdk_bdev_module / fastblock_fn_table

Volume / bdev 级（一个 image 一个）
└── bdev_fastblock（disk + image/pool/size/object_size）

per-thread / channel 级
├── bdev_fastblock_io_channel
├── libblk_client（每 shard 一个，channel 创建时建立）
└── fblock_client（随 libblk_client 1:1）

connection / transport 级
├── msg::rdma::client（fblock_client 内一个）
├── msg::rdma::client::connection（按目标 OSD 建，可复用）
├── socket / write_ring_state
├── memory_pool / net_context（client 级共享池）
└── rpc_service_osd_Stub（连接就绪后建）

Logical IO 级（一次上层 IO）
├── write_source（WRITE）
└── read_source（READ）

Object request 级（每个 object fragment 一个）
├── request_stack_type
├── osd::write_request / osd::write_reply（或 read 版）
├── msg::rdma::rpc_controller
├── request_meta
├── connection::rpc_request（进入传输层后）
└── transport_data（借用 memory_pool 块）
```

---

## 3. Ownership 全景

```text
global::blk_clients : std::vector<std::shared_ptr<libblk_client>>
        │  shared_ptr
        v
libblk_client
        │  std::unique_ptr<fblock_client> _client
        v
fblock_client
        │
        ├── std::shared_ptr<msg::rdma::client> _rpc_client
        │        ├── connection（shared_ptr，按 OSD 缓存）
        │        └── memory_pool（shared_ptr：meta/data）
        │
        ├── std::list<std::unique_ptr<request_stack_type>> _requests
        ├── std::list<std::unique_ptr<request_stack_type>> _on_flight_requests
        ├── unordered_map<key, unique_ptr<leader_osd_info>> _leader_osd
        ├── unordered_map<key, unique_ptr<leader_request_stack_type>> _leader_requests / _on_flight_leader_requests
        └── unordered_map<conn_id, shared_ptr<write_ring_state>> _write_rings
                 └── shared_ptr<connection> conn

request_stack_type（被 _requests / _on_flight_requests / 局部 unique_ptr 持有）
        ├── std::unique_ptr<osd::write_request> req        （或 read_request）
        ├── std::unique_ptr<osd::write_reply>   resp       （或 read_reply）
        ├── std::unique_ptr<msg::rdma::rpc_controller> ctrlr
        ├── osd::rpc_service_osd_Stub* stub（裸指针，指向 _stubs 中的对象）
        ├── void* ctx              （指向 write_source / read_source）
        ├── resp_cb                （write_object_callback / read_object_callback）
        └── std::unique_ptr<ring_write_context> ring_ctx

write_source（裸指针，被每个 child 回调 ctx 引用）
        └── cb = write_callback（最终 bdev 回调）
```

关键点：

- `libblk_client` → `fblock_client` 是 **unique_ptr 独占**（`src/include/fastblock/client/libfblock.h:66`）。
- `request_stack_type` 在队列中由 **unique_ptr** 持有；出队后由局部 `unique_ptr` 持有（`process_response`），retry 时同一对象被移回 `_requests`。
- `stub` 是裸指针，真正的 owner 是 `fblock_client::_stubs`（`unordered_map<conn_id, unique_ptr<Stub>>`）。
- `write_source` 的 owner 是“所有 child 回调的 `ctx` 合集”，由 `write_source::invoke()` 在最后一个 child 完成时 `delete this`。

---

## 4. Thread / Core / Shard / Client 关系

```text
SPDK core list（spdk_env_get_first_core / next_core / count）
      │  下标
      v
shard index（utils::get_current_shard_id；core_sharded::this_shard_id）
      │
      v
SPDK thread（core_sharded 构造时 spdk_thread_create，单核 cpumask 绑定）
      │  channel 创建回调在该线程运行
      v
per-thread io_channel（bdev_fastblock_io_channel）
      │
      v
global::blk_clients[shard] → libblk_client → fblock_client(_current_thread)
```

必须记住：

- `fblock_client` 是**对象**，不是线程；它绑定 `_current_thread`。
- shard 是**下标**，不是 core id（core 不连续时数值不同）。
- `bdev_fastblock_io_channel` 不是 client；它是 bdev 的 per-thread 上下文（eventfd/epoll 用）。
- Volume 与 shard **无绑定关系**（`struct bdev_fastblock` 无 shard 字段）。
- 数据路径选 client 的唯一依据：`utils::get_current_shard_id()`（`src/bdev/bdev_fastblock.cc:327`、`:379`）。

---

## 5. 一个 IO 到底存在多少层“请求”

```text
Logical WRITE（10 MiB，1 个）
        │ libblk_client 拆
        v
Object child（3 个：4+4+2 MiB）
        │ fblock_client 封装
        v
request_stack_type（3 个）
        │ 内含 payload
        v
osd::write_request（3 个 protobuf message）
        │ Stub → CallMethod 封装
        v
connection::rpc_request（3 个，进入连接队列）
        │ transport_data 序列化
        v
ibv_send_wr / SGE（一段内存描述，post 到 QP）
        v
RDMA send
```

**这些都叫“request”，但不是同一个东西**：`request_stack_type` 是客户端生命周期帧；`osd::write_request` 是线上 protobuf payload；`connection::rpc_request` 是传输层在途任务。

---

## 6. 类型详解

### 6.1 `spdk_bdev_io`

1. **一句话定义**：SPDK blk-mq 的一次块 IO 对象，FastBlock 只是它的“处理者”。
2. **它不是什么**：不是 FastBlock 类型（`BACKGROUND ONLY`，定义在 SPDK）；不是 FastBlock request。
3. **所属层次**：SPDK bdev 层。
4. **为什么需要**：SPDK 用它携带 iov/offset/len，FastBlock 通过它拿到用户数据与完成句柄。
5. **谁创建**：SPDK blk-mq（外部）；FastBlock 只是接收。
6. **谁持有**：SPDK bdev 框架；FastBlock 保存裸指针用于回传完成。
7. **关键字段**：`u.bdev.iovs/iovcnt`、`u.bdev.offset_blocks/num_blocks`、`bdev->blocklen`、`bdev->ctxt`（指向 `bdev_fastblock`）。
8. **关键方法**：`spdk_bdev_io_complete`（最终完成）。
9. **Thread Ownership**：由提交线程使用；完成必须回到同一线程（`BACKGROUND ONLY`）。
10. **生命周期**：SPDK 创建 → FastBlock 读 iov/offset → callback → `spdk_bdev_io_complete`。
11. **上游**：consumer（vhost/nvmf/block_bench）。
12. **本层做什么**：无（FastBlock 不修改它，只读）。
13. **为下一层准备**：提供 offset/length 与 iov 数据，供 `bdev_fastblock_write` 转成 Logical IO。
14. **何时退出**：`bdev_fastblock_write_callback()` → `spdk_bdev_io_complete`（`src/bdev/bdev_fastblock.cc:290-300`）。
15. **易混淆**：v.s. `write_source`（后者是 FastBlock 的 parent 上下文，不完整 IO）。
16. **源码**：`src/bdev/bdev_fastblock.cc::bdev_fastblock_write` 读取字段（`:309`）。

### 6.2 `bdev_fastblock`

1. **一句话定义**：FastBlock 注册进 SPDK bdev 框架的一个“卷句柄”。
2. **它不是什么**：不是线程、不是 client、不是请求；它是 `spdk_bdev` 的宿主结构。
3. **所属层次**：FastBlock bdev 层（Volume 级）。
4. **为什么需要**：把 pool_id/image_name/size/object_size 绑定到一个 bdev 上，IO 回调时能找回。
5. **谁创建**：`bdev_fastblock_create()`（`:677`，`calloc`）。
6. **谁持有**：`fastblock->disk.ctxt = fastblock`（`:759` 附近），由 SPDK bdev 模块持有；`bdev_fastblock_free()` 释放。
7. **关键字段**：

| 字段 | 类型 | 含义 | 谁写 | 谁读 | 生命周期意义 |
|---|---|---|---|---|---|
| `disk` | `spdk_bdev` | 注册进 SPDK 的设备对象 | create | SPDK/回调 | 与 bdev 同生灭 |
| `image_name` | `char*` | 远端 image 名 | create | write/read 入口 | Logical IO 参数 |
| `pool_id` | `uint64_t` | pool | create | write/read 入口 | 路由参数 |
| `object_size` | `uint64_t` | 拆分粒度 | create | 拆分逻辑 | 决定 object 数 |
| `image_size` / `block_size` | `uint64_t`/`uint32_t` | 容量/块长 | create | SPDK blockcnt 计算 | bdev 规格 |
| `monitor_address` | `char*` | 监控地址（记录） | create | 日志 | 元数据 |
| `reset_timer`/`reset_bdev_io` | 指针 | reset 排它 | `bdev_fastblock_reset` | timer cb | 管理操作 |

8. **关键方法**：无 IO 方法（IO 入口是回调函数）。
9. **Thread Ownership**：非线程绑定；多线程可并发读字段。
10. **生命周期**：`bdev_fastblock_create` → 注册 bdev（`:769`）→ `bdev_fastblock_delete`（`:782`）→ `spdk_bdev_unregister` → `bdev_fastblock_destruct`（`:233`）→ `bdev_fastblock_free`（`:74`）。
11. **上游**：`bdev_fastblock_rpc.cc` 的 JSON-RPC 创建命令（`bdev_fastblock_create`）。
12. **本层做什么**：保存卷元数据 + 挂 bdev 接口表。
13. **为下一层准备**：`bdev_io->bdev->ctxt` 可反查本对象，从而拿到 pool/image/object_size。
14. **何时退出主流程**：bdev 删除/进程退出。
15. **易混淆**：v.s. `libblk_client`（后者是 IO 执行者，不持卷元数据）。
16. **源码**：`src/bdev/bdev_fastblock.cc:38`（定义）、`:677`（创建）、`:74`（释放）。

### 6.3 `bdev_fastblock_io_channel`

1. **一句话定义**：SPDK per-thread channel 的 FastBlock 上下文（eventfd/disk/group_ch）。
2. **它不是什么**：不是 client、不是请求；不是 per-volume 句柄。
3. **所属层次**：FastBlock bdev 层的 per-thread 资源。
4. **为什么需要**：SPDK channel 需要一个 ctx；FastBlock 用它保存每个线程自己的 epoll/eventfd 与磁盘指针。
5. **谁创建**：`bdev_fastblock_create_cb()`（`:484`），由 `spdk_io_device_register(fastblock, ..., sizeof(channel), ...)`（`:765-768`）在 channel 创建时回调。
6. **谁持有**：SPDK io_device/channel。
7. **关键字段**：`pfd`（eventfd）、`disk`（`bdev_fastblock*`）、`group_ch`（模块组）。
8. **关键方法**：`bdev_fastblock_free_channel()`（`:456`）。
9. **Thread Ownership**：per-thread（谁创建 channel 谁用）。
10. **生命周期**：channel 创建 → 使用（epoll 注册）→ channel 销毁（`bdev_fastblock_destroy_cb` `:559`）。
11. **上游**：consumer `spdk_bdev_get_io_channel`/FastBlock `bdev_fastblock_get_io_channel`（`:595`）。
12. **本层做什么**：准备该线程的 bdev 通道资源；**同时触发该线程的 `libblk_client` 创建**（`:548-550`）。
13. **为下一层准备**：让本线程的 shard 槽位拥有一个 client。
14. **何时退出**：channel 销毁回调。
15. **易混淆**：v.s. `libblk_client`（同一线程各一个，但职责完全不同：channel 管 SDKB 通道，client 管 IO）。
16. **源码**：`src/bdev/bdev_fastblock.cc:61`（定义）、`:484`（创建回调）、`:559`（销毁回调）。

### 6.4 `bdev_fastblock_group_channel`

1. **一句话定义**：模块级 epoll 组（一个 poller + 一个 epoll_fd）。
2. **它不是什么**：不是 per-thread channel；不参与 WRITE 数据路径。
3. **所属层次**：FastBlock bdev 模块资源。
4. **为什么需要**：把多个 channel 的 eventfd 聚到一个 epoll 上，由模块 poller 统一处理（reset/管理事件）。
5. **谁创建**：`spdk_io_device_register(&fastblock_if, bdev_fastblock_group_create_cb, ...)`（`:884-886`）；channel 通过 `spdk_io_channel_get_ctx(spdk_get_io_channel(&fastblock_if))` 取得（`:509`）。
6. **谁持有**：模块 io_device。
7. **关键字段**：`poller`、`epoll_fd`。
8. **关键方法**：`bdev_fastblock_group_poll`（`:833` 附近）。
9. **Thread Ownership**：模块级（无 per-thread 状态）。
10. **生命周期**：进程级。
11. **上游**：模块注册。
12. **本层做什么**：管理操作的事件聚合。
13. **为下一层准备**：无（WRITE 主路径不经它）。
14. **何时退出**：进程结束。
15. **易混淆**：v.s. `bdev_fastblock_io_channel`。
16. **源码**：`src/bdev/bdev_fastblock.cc:55`（定义）、`:884`（注册）、`:833`（poll）。

### 6.5 `spdk_bdev_module` / `fastblock_fn_table`

1. **一句话定义**：FastBlock 与 SPDK bdev 框架之间的“接口表”。
2. **它不是什么**：不是请求、不是线程。
3. **所属层次**：SPDK/FastBlock 边界。
4. **为什么需要**：SPDK 只认接口表；FastBlock 通过它挂上 `submit_request`/`get_io_channel`/`io_type_supported`。
5. **谁创建**：静态定义 `fastblock_if`（`:202`）、`fastblock_fn_table`（`:668`）；`SPDK_BDEV_MODULE_REGISTER`（`:209`）。
6. **谁持有**：SPDK 框架（`disk.module`/`disk.fn_table`）。
7. **关键字段**：`.submit_request`、`.get_io_channel`、`.io_type_supported`、`.dump_info_json`。
8. **关键方法**：`bdev_fastblock_submit_request()`（`:429`）→ `_bdev_fastblock_submit_request()`（`:395`）。
9. **Thread Ownership**：进程级。
10. **生命周期**：进程级。
11. **上游**：SPDK bdev 框架调用。
12. **本层做什么**：把 SPDK 的 IO 类型分派到 FastBlock 入口（READ/WRITE/FLUSH/RESET）。
13. **为下一层准备**：`bdev_fastblock_write`/`get_buf_cb`。
14. **何时退出**：进程结束。
15. **易混淆**：v.s. `bdev_fastblock`（fn_table 是方法表，bdev 是数据）。
16. **源码**：`src/bdev/bdev_fastblock.cc:202` / `:668` / `:429`。

### 6.6 `core_sharded`

1. **一句话定义**：FastBlock 的 shard 表——每 shard 一个 core、一个 SPDK thread，并提供 `invoke_on`。
2. **它不是什么**：不是线程本身；是“shard→core→thread”的索引结构。
3. **所属层次**：进程级基础设施。
4. **为什么需要**：FastBlock 所有 per-shard 资源需要统一编号与跨 shard 投递入口。
5. **谁创建**：`core_sharded::construct(...)`（`src/bdev/common.cc:126`）。
6. **谁持有**：文件级 `g_core_sharded`（`core_sharded.h:31`）。
7. **关键字段**：

| 字段 | 类型 | 含义 | 谁写 | 谁读 | 生命周期意义 |
|---|---|---|---|---|---|
| `_shard_cores` | `vector<uint32_t>` | shard → core id | 构造 | `this_shard_id`/`invoke_on` | 进程级 |
| `_threads` | `vector<spdk_thread*>` | shard → thread | 构造 | `get_thread`/`invoke_on` | 进程级 |

8. **关键方法**：`system::capacity()`（`:92`）、`get_thread()`（`:197`）、`this_shard_id()`（`:232`）、`invoke_on()`（`:216`）、`construct()`（`:187`）、`stop_all()`（`:191`）。
9. **Thread Ownership**：全局；`invoke_on` 会把任务投到目标 shard 的 thread（`spdk_thread_send_msg`，`:225`）。
10. **生命周期**：`construct` → 使用 → `stop_all`。
11. **上游**：应用初始化 `common.cc`。
12. **本层做什么**：建立 shard 与 thread 的一一对应（`spdk_thread_create` + 单核 cpumask，`:163-167`）。
13. **为下一层准备**：`global::blk_clients` 的索引语义与 OSD 侧 `invoke_on`。
14. **何时退出**：`common.cc::general_stop()` → `core_sharded::stop_all()`。
15. **易混淆**：shard ≠ core id（是下标）；shard ≠ thread（`_threads[shard]` 才是 thread）。
16. **源码**：`src/include/fastblock/base/core_sharded.h:67`（类）、`:156`（构造）、`:232`（this_shard_id）。

### 6.7 `global::blk_clients`

1. **一句话定义**：每 shard 一个 `libblk_client` 的全局表（容量 = core 数 + 1）。
2. **它不是什么**：不是线程池；不是连接池；元素是 client 对象。
3. **所属层次**：进程级全局资源。
4. **为什么需要**：数据路径按“当前 shard”O(1) 找到本线程的 client。
5. **谁创建**：`common.cc::fb_client_init` 中 `resize(capacity + 1)`（`:139-140`；`app_thread_shard_id = capacity`）。
6. **谁持有**：全局 vector（`src/bdev/global.cc:19`）。
7. **关键字段**：`shared_ptr<libblk_client>` 元素；空槽位表示该 shard 还没有 client。
8. **关键方法**：无（被 `at(index)` 直接访问）。
9. **Thread Ownership**：进程级共享；元素只在对应 shard 线程被正常使用（数据路径）。
10. **生命周期**：进程级；元素在 channel 创建时填充（`bdev_fastblock.cc:548-550`），channel 销毁时清空（`:585-590`）。
11. **上游**：`common.cc` 初始化 + channel create cb。
12. **本层做什么**：shard → client 的查表。
13. **为下一层准备**：`blk_cli->write/read`。
14. **何时退出**：进程结束（或 bdev 通道销毁置空）。
15. **易混淆**：`global::blk_client`（单数）在仓库中**只声明未使用**（`NOT PROVEN 其用途`）。
16. **源码**：`src/include/fastblock/bdev/global.h:22`、`src/bdev/global.cc:19`、`src/bdev/common.cc:139`。

### 6.8 `utils::simple_poller`

1. **一句话定义**：把回调注册成绑定目标 SPDK thread 的 poller 的小工具。
2. **它不是什么**：不是线程；不持有业务状态。
3. **所属层次**：事件驱动基础设施。
4. **为什么需要**：注册/注销 poller 必须在目标线程执行；它封装了 `spdk_thread_send_msg`。
5. **谁创建**：`fblock_client` 构造时创建三个成员（`fb_client.h:1357-1359`）。
6. **谁持有**：`fblock_client`（`_leader_poller`/`_request_poller`/`_response_poller`）。
7. **关键字段**：`_poller`、`_thread`。
8. **关键方法**：`register_poller()`（`simple_poller.h:100`）、`unregister_poller()`（`:83`）、`handle_register()`（`:70`）、`set_thread()`（`:105`）。
9. **Thread Ownership**：注册在目标 thread；注销在目标 thread。
10. **生命周期**：client 构造 → `handle_start` 注册（`fb_client.h:837-839`）→ client 停止注销。
11. **上游**：`fblock_client::start`。
12. **本层做什么**：把 `poll_leader/poll_request/poll_response` 挂到 `_current_thread`。
13. **为下一层准备**：三个 poller 驱动的请求流水线。
14. **何时退出**：`unregister_poller`（stop 流程）。
15. **易混淆**：v.s. `connection` 自己的 poller（`rpc_cli_conn`，在 `msg/rdma/client.h` 内注册）。
16. **源码**：`src/include/fastblock/utils/simple_poller.h:21`、`src/include/fastblock/client/fb_client.h:837`。

### 6.9 `monitor::client`（+ `pg_map` / `osd_map`）

1. **一句话定义**：Client 侧控制面对象，缓存集群地图（PG 数、PG 成员、OSD 地址/状态）。
2. **它不是什么**：不参与数据发送；不是线程。
3. **所属层次**：控制面（主流程的“路由元数据来源”）。
4. **为什么需要**：`calc_target` 需要 `pg_num`；找 Leader 需要“PG 成员里一个可用 OSD”；Leader 校验需要 OSD 信息。
5. **谁创建**：`common.cc::fb_client_init`（`:135`）。
6. **谁持有**：`global::mon_client`（unique_ptr）；`fblock_client` 持裸指针 `_mon_cli`。
7. **关键字段**：`pg_map::pool_pg_map`（`monclient/client.h:211`）、`osd_map::data`（`:124`）、`pool_version`（`:212`）。
8. **关键方法**：`get_pg_num()`（`monclient/client.cc:1057`）、`get_pg_first_available_osd_info()`（`:1029`）、`get_osd_info()`（`:1068`）、`get_pool_id()`（`monclient/client.h:545`）。
9. **Thread Ownership**：绑定一个 SPDK thread；地图由周期 poller 更新。
10. **生命周期**：进程级。
11. **上游**：Monitor（TCP 拉取）。
12. **本层做什么**：周期拉取并缓存集群地图。
13. **为下一层准备**：PG 数、PG 成员、OSD 信息供路由。
14. **何时退出**：进程结束。
15. **易混淆**：v.s. `osd/mon_client.cc` 的 `monitor_client`（OSD 侧子类，同名不同物）。
16. **源码**：`src/include/fastblock/monclient/client.h:38`、`src/monclient/client.cc:1057`。

### 6.10 `libblk_client`

1. **一句话定义**：一个 Logical IO 拆成 Object 的 **API wrapper**（本 shard 一个）。
2. **它不是什么**：不是线程、不是 SPDK 对象、不是请求；不管理 Leader/RPC。
3. **所属层次**：Logical IO 层。
4. **为什么需要**：把“块 IO 语义”（连续 offset/len）翻译成“对象语义”（object 名/内偏移/分片），并聚合 child 完成；让 `fblock_client` 专注 Object→PG→Leader→RPC。
5. **谁创建**：channel 创建回调 `bdev_fastblock_create_cb()`（`bdev_fastblock.cc:548`）。
6. **谁持有**：`global::blk_clients[index]`（`shared_ptr`）。
7. **关键字段**：

| 字段 | 类型 | 含义 | 谁写 | 谁读 | 生命周期意义 |
|---|---|---|---|---|---|
| `_client` | `unique_ptr<fblock_client>` | 真正的客户端实现（1:1） | ctor | write/read | 与 wrapper 同生灭 |
| `_mon_cli` | `monitor::client*` | 控制面指针（仅记录） | ctor | 无（透传） | 进程级 |

（`default_object_size` 是命名空间常量，`libfblock.h:28`。）

8. **关键方法**：

| 方法 | 作用 | 输入 | 输出/副作用 | 下一步 |
|---|---|---|---|---|
| `write()`（iov 版，`:83`） | iov → `std::string` | bdev iov/len | 拼出连续 buffer | 调 buffer 版 write |
| `write()`（buffer 版，`:177`） | 拆分 + 逐 Object 提交 | offset/len/buf | 建 `write_source`；循环 `write_object` | `fblock_client::write_object` |
| `read()`（`:296`） | 拆分 + 逐 Object 提交 | offset/len | 建 `read_source`；循环 `read_object` | `fblock_client::read_object` |
| `calc_image_object_prefix()`（`:332`） | 生成 object 名前缀 | pool/image | 前缀字符串 | 命名 |
| `calc_first_object_position()`（`:338`） | 首 object 定位 | offset/len/size | `(首片大小, 内偏移, seq)` | 拆分循环 |
| `get_image_object_name()`（`:352`） | object 名 = 前缀+seq | prefix/seq | 名字 | 提交 |

9. **Thread Ownership**：对象；在“创建它的线程/该 shard 线程”上使用（同一前提）。
10. **生命周期**：channel 创建 → 使用 → channel 销毁（`bdev_fastblock_destroy_cb` 中 `stop`）。
11. **上游**：`bdev_fastblock_write()`/`get_buf_cb()`。
12. **本层做什么**：拆分 + parent 聚合（10 MiB → 3 个 child）。
13. **为下一层准备**：每个 Object 的名字/内偏移/数据片，交给 `fblock_client` 建请求。
14. **何时退出**：bdev 通道销毁或进程结束。
15. **易混淆**：v.s. `fblock_client`（见第 7 节对比）。
16. **源码**：`src/include/fastblock/client/libfblock.h:56`、`src/client/libfblock.cc:83/177/296`。

### 6.11 `write_source`

1. **一句话定义**：一个 Logical WRITE 的 **parent completion context**。
2. **它不是什么**：不是 Object 请求；不是线程；一个 10 MiB WRITE 只有 **1 个** `write_source`。
3. **所属层次**：Logical IO 层。
4. **为什么需要**：一个 Logical WRITE 拆成 N 个 Object，不能让第一个 Object 完成就通知上层；必须等 N 个都完成并取首个错误。
5. **谁创建**：`libblk_client::write()`（buffer 版）`new write_source(cb, obj_num, bdev_io)`（`libfblock.cc:193`）。
6. **谁持有**：裸指针，被每个 child 的 `source` 参数持有；`obj_num` 归零时 `invoke()` 自毁。
7. **关键字段**：

| 字段 | 类型 | 含义 | 谁写 | 谁读 | 生命周期意义 |
|---|---|---|---|---|---|
| `cb` | `write_callback` | 最终 bdev 完成回调 | ctor | `invoke` | 终点 |
| `obj_num` | `uint32_t` | 剩余未完成 child 数 | `write_done` | `write_done` | **归零即完成** |
| `bdev_io` | `spdk_bdev_io*` | 原始 IO | ctor | `invoke` | 完成句柄 |
| `result` | `int32_t` | 首个错误（初始 success） | `write_done` | `invoke` | 最终结果 |
| `thread` | `spdk_thread*` | 发起线程（DEBUG 回切用） | ctor | `invoke` | 线程归属 |

8. **关键方法**：

| 方法 | 作用 | 输入 | 输出/副作用 | 下一步 |
|---|---|---|---|---|
| `write_done()`（`:151`） | child 完成 | `(src, state)` | 记错误、`obj_num--`；归零则 `invoke` | `invoke` |
| `invoke()`（`:138`） | 最终通知 | 无 | release 直接 `cb`+`delete this`；DEBUG 经 `thread_run` 回线程 | bdev callback |
| `thread_run()`（`:131`） | DEBUG 回线程执行 | `this` | `cb` 后 `delete` | — |

9. **Thread Ownership**：归属发起 IO 的线程；child 回调可能来自 client 线程，DEBUG 下切回。
10. **生命周期**：`new`（拆分前）→ 被 N 个 child 引用 → 最后一次 `write_done` → `invoke` → `cb` → `delete`。
11. **上游**：`libblk_client::write()`。
12. **本层做什么**：child 完成计数与首个错误保存。
13. **为下一层准备**：给 `bdev_fastblock_write_callback` 提供 `(bdev_io, result)`。
14. **何时退出**：`obj_num == 0` 时。
15. **易混淆**：v.s. `request_stack_type`（见第 7 节对比）。
16. **源码**：`src/client/libfblock.cc:114`（定义）、`:151`（write_done）、`:138`（invoke）。

### 6.12 `read_source`

1. **一句话定义**：一个 Logical READ 的 parent completion context（带数据聚合）。
2. **它不是什么**：不是 Object 请求；与 `write_source` 不同，它**持有 buffer**。
3. **所属层次**：Logical IO 层。
4. **为什么需要**：读的 N 个 Object 数据要按 object_idx 拼到一块连续 buffer，再一次性交给上层。
5. **谁创建**：`libblk_client::read()` `new read_source(...)`（`libfblock.cc:309`）。
6. **谁持有**：裸指针，被每个 child 的 `source` 持有；`obj_num` 归零时自毁。
7. **关键字段**：

| 字段 | 类型 | 含义 | 谁写 | 谁读 | 生命周期意义 |
|---|---|---|---|---|---|
| `cb` | `read_callback` | 最终读回调 | ctor | `thread_run` | 终点 |
| `obj_num` | `uint32_t` | 剩余 child 数 | `read_done` | `read_done` | 归零即完成 |
| `buf` | `char*` | 聚合读数据（`new char[len]`） | ctor | `read_done`/回调 | **拥有数据** |
| `len` | `uint64_t` | 总长 | ctor | 回调 | — |
| `first_object_size` | `uint64_t` | 首片大小（计算 object 内偏移） | ctor | `read_done` | 拼接位置 |
| `offset` | `uint64_t` | 原始逻辑偏移（日志） | ctor | 日志 | — |
| `result` / `thread` | — | 首个错误 / 发起线程 | `read_done`/ctor | `thread_run` | 同 write |

8. **关键方法**：`read_done()`（`:269`，按 `object_idx` 拷贝进 `buf` 并递减）；`invoke()`（`:256`）；`thread_run()`（`:250`）。析构 `delete[] buf`（`:247`）。
9. **Thread Ownership**：同 `write_source`。
10. **生命周期**：同 `write_source`（buffer 随对象析构释放）。
11. **上游**：`libblk_client::read()`。
12. **本层做什么**：按 object_idx 拼接多 Object 数据。
13. **为下一层准备**：完整读数据 + 结果给 `bdev_fastblock_read_callback`。
14. **何时退出**：`obj_num == 0`。
15. **易混淆**：v.s. `write_source`（不持 buffer、按计数聚合）。
16. **源码**：`src/client/libfblock.cc:218`（定义）、`:269`（read_done）。

### 6.13 `fblock_client`

1. **一句话定义**：Client 侧真正的“发货调度员”——管 Object→PG→Leader→RPC 生命周期与请求队列。
2. **它不是什么**：不是线程（是绑定 `_current_thread` 的对象）；不拆 Object；不直接读写数据。
3. **所属层次**：Object / Client 实现层；向上接 `libblk_client`，向下接 RPC/transport。
4. **为什么需要**：把 Object 请求变成可发送的 RPC：路由（PG/Leader）、排队、发送、响应、重试、回调。
5. **谁创建**：`libblk_client` 构造时 `make_unique<fblock_client>`（`libfblock.h:61`）。
6. **谁持有**：`libblk_client::_client`（unique_ptr）。
7. **关键字段**：

| 字段 | 类型 | 含义 | 谁写 | 谁读 | 生命周期意义 |
|---|---|---|---|---|---|
| `_rpc_client` | `shared_ptr<msg::rdma::client>` | RDMA RPC 客户端 | ctor | 连接 | 传输层入口 |
| `_mon_cli` | `monitor::client*` | 控制面 | ctor | pg_num/leader | 路由元数据 |
| `_current_thread` | `spdk_thread*` | 绑定线程 | ctor | send/start/stop | **线程归属** |
| `_requests` | `list<unique_ptr<request_stack_type>>` | 待发 FIFO | handle_send | process_request | queued |
| `_on_flight_requests` | `list<unique_ptr<request_stack_type>>` | 在途 FIFO | process_request | process_response | 等响应 |
| `_leader_osd` | `unordered_map<uint64, unique_ptr<leader_osd_info>>` | Leader 缓存 | on_leader_acquired/retry | process_request | 路由缓存 |
| `_leader_requests` / `_on_flight_leader_requests` | map | leader 查询请求 | enqueue/process | on_leader_acquired | 路由查询 |
| `_stubs` | `unordered_map<uint64, unique_ptr<Stub>>` | 每连接 stub 缓存 | on_connection_ready/get_stub | process_request | RPC 代理 |
| `_write_rings` | `unordered_map<uint64, shared_ptr<write_ring_state>>` | write-ring 状态 | ensure/acquire | process_request | 传输优化 |

8. **关键方法**：

| 方法 | 作用 | 输入 | 输出/副作用 | 下一步 |
|---|---|---|---|---|
| `write_object()`（`:1267`） | 建写请求 | object/offset/data/pool | `calc_target`→`send_request` | `send_request` |
| `read_object()`（`:1292`） | 建读请求 | object/offset/len/idx | 同上 | `send_request` |
| `delete_object()`（`:1319`） | 建删除请求 | object/pool | 同上（当前无 bdev 入口） | `send_request` |
| `send_request()`（`:709`） | 建 stack + 线程切换 | pool/pg/req/cb/ctx | `new request_stack_type`；投递/直呼 | `handle_send_request` |
| `handle_send_request()`（`:869`） | Leader 决策 + 入队 | stack | 查 `_leader_osd`；入 `_requests` | poller |
| `process_request()`（`:1063`） | 发送 | 队头 | `get_stub`；`stub->process_write`；移入 on-flight | RPC |
| `process_leader_request()`（`:1034`） | 查 Leader | leader 请求 | `stub->process_get_leader` | `on_leader_acquired` |
| `on_leader_acquired()`（`:1199`） | 缓存 Leader | response | 写 `leader_id/addr/port` | 后续 process_request |
| `on_response()`（`:1261`） | 标记响应 | stack | `is_responsed = true` | `process_response` |
| `process_response()`（`:932`） | 分发/重试 | on-flight 队头 | 成功调 cb / 失败 `retry_request` | 完成或重试 |
| `retry_request()`（`:326`） | 重试复用 | stack | 失效 leader/transport；`push_front _requests` | 重新发送 |
| `get_stub()`（`:588`） | 拿连接代理 | osd id/addr/port | 复用/建连；可能返回 nullptr | stub 调用 |

9. **Thread Ownership**：`_current_thread` 是唯一合法写线程（token/queue/leader 缓存）；`send_request` 对非本线程调用做 `spdk_thread_send_msg`。
10. **生命周期**：随 `libblk_client`；`start` 注册 poller（`:834-841`），`stop` 注销。
11. **上游**：`libblk_client` 的 Object 循环。
12. **本层做什么**：路由 + 排队 + 发送 + 响应 + 重试。
13. **为下一层准备**：把 payload 交给 Stub/Connection。
14. **何时退出**：client 停止（bdev 通道销毁/进程退出）。
15. **易混淆**：v.s. `libblk_client`（见第 7 节）。
16. **源码**：`src/include/fastblock/client/fb_client.h:46`、队列 `:1364-1365`、Leader 缓存 `:1368`。

### 6.14 `request_stack_type`

1. **一句话定义**：一个 Object RPC 的完整状态帧（“档案袋”）。
2. **它不是什么**：不是 protobuf payload；不是 Logical IO；不是线程。
3. **所属层次**：Object request 层。
4. **为什么需要**：异步 RPC 返回时函数栈早已销毁，必须把“发的是谁、回调谁、上下文、状态”装在一个对象里跨函数存活。
5. **谁创建**：`fblock_client::send_request()` `new request_stack_type{}`（`:713`）。
6. **谁持有**：`_requests` → `_on_flight_requests` → 局部 `unique_ptr`（`process_response`）；retry 时复用。
7. **关键字段**：

| 字段 | 类型 | 含义 | 谁写 | 谁读 | 生命周期意义 |
|---|---|---|---|---|---|
| `stub` | `Stub*` | 目标 OSD 代理 | process_request | process_write | 发送目标 |
| `ctrlr` | `unique_ptr<rpc_controller>` | 传输状态 | RPC 层 | process_response | 失败判断 |
| `req` | `variant<unique_ptr<write_request>, ...>` | payload | write_object | stub | 发送内容 |
| `resp` | `variant<unique_ptr<write_reply>, ...>` | 响应 | process_request 建壳；RPC 填充 | process_response | 结果 |
| `resp_cb` | `variant<write_object_callback, ...>` | child 回调 | send_request | process_response | 回调链 |
| `ctx` | `void*` | parent（`write_source*`） | send_request | 回调 | 聚合连接 |
| `obj_index` | `uint64_t` | 读时定位 | send_request | read 回调 | 读拼接 |
| `is_responsed` | `bool` | 响应到达标记 | on_response | process_response | poller 语义 |
| `leader_osd_key` | `uint64_t` | `(pool,pg)` | send_request | 查 leader 缓存 | 路由 key |
| `ring_ctx` | `unique_ptr<ring_write_context>` | write-ring 上下文 | post_ring_write | 完成/重试 | MR 生命周期 |

8. **关键方法**：无成员方法（纯状态 + `ring_write_context` 析构 `ibv_dereg_mr`/`spdk_free`，`:87-94`）。
9. **Thread Ownership**：只在 client 线程流转（`_requests`/`_on_flight_requests`/poller）；callback 可在连接线程触发 `on_response`（仅置位）。
10. **生命周期**：

```text
send_request: new
   ↓
handle_send_request: _requests 尾部
   ↓
process_request: 出队 → stub->process_write → _on_flight_requests
   ↓
on_response: is_responsed = true
   ↓
process_response: 出队到局部 unique_ptr
   ├── 成功/终态：cb(ctx,state) → 作用域结束析构
   └── 可重试：retry_request → push_front(_requests)（同一对象复用）
```

11. **上游**：`fblock_client::write_object/read_object`。
12. **本层做什么**：承载一次 Object RPC 的全部异步状态。
13. **为下一层准备**：`req`（payload）、`ctrlr`、`resp`、`closure` 交给 Stub/Connection。
14. **何时退出**：终态回调后析构（或 pool 不存在时被 `handle_send_request` 直接 delete，`:918`）。
15. **易混淆**：v.s. `osd::write_request`（payload）；v.s. `write_source`（Logical IO）。
16. **源码**：`src/include/fastblock/client/fb_client.h:79`（定义）、`:713`（创建）、`:1191`（入 on-flight）、`:346`（retry 复用）。

### 6.15 `leader_key_type` / `make_leader_key()`

1. **一句话定义**：把 `(pool_id, pg_id)` 打包成一个 `uint64_t` 的 map key。
2. **它不是什么**：不是 Leader 信息；只是 key。
3. **所属层次**：路由。
4. **为什么需要**：Leader 缓存/查询都用单键 map；PG 身份必须 pool+pg 联合。
5. **谁创建**：值类型，`make_leader_key()` 生成（`:611`）。
6. **谁持有**：`request_stack_type::leader_osd_key`。
7. **关键字段**：`int32_t pool_id; int32_t pg_id;`（`:111`）。
8. **关键方法**：`make_leader_key()`（`:611`）、`from_leader_key()`（`:620`）。
9. **Thread Ownership**：值类型，无。
10. **生命周期**：随请求帧。
11. **上游**：`send_request`。
12. **本层做什么**：打包/解包。
13. **为下一层准备**：`_leader_osd.find(key)`。
14. **何时退出**：随 stack。
15. **易混淆**：v.s. `leader_osd_info`（key vs value）。
16. **源码**：`src/include/fastblock/client/fb_client.h:111`。

### 6.16 `leader_osd_info`

1. **一句话定义**：一个 PG 的 Leader OSD 缓存值。
2. **它不是什么**：不是连接；不是 PG 成员表（成员表在 monclient）。
3. **所属层次**：路由缓存层。
4. **为什么需要**：避免每个 Object 都发 `get_leader` RPC。
5. **谁创建**：`handle_send_request` 缓存缺失时 `emplace`（`:889`）；`on_leader_acquired` 填值（`:1232-1235`）。
6. **谁持有**：`fblock_client::_leader_osd`（unique_ptr）。
7. **关键字段**：

| 字段 | 类型 | 含义 | 谁写 | 谁读 | 生命周期意义 |
|---|---|---|---|---|---|
| `leader_id` | `int32_t` | Leader OSD id | on_leader_acquired | process_request/get_stub | 目标 OSD |
| `addr` | `std::string` | 地址 | 同上 | 同上 | 连接参数 |
| `port` | `int32_t` | 端口 | 同上 | 同上 | 连接参数 |
| `is_valid` | `bool` | 缓存是否可信 | update_leader_state/retry | process_request | 是否重查 |
| `is_onflight` | `bool` | 是否正在查 | handle/enqueue/acquired | process_request | 防重复查询 |
| `epoch` | time_point | 获取时间 | acquired | update_leader_state | 版本兜底 |

8. **关键方法**：无（值结构）。
9. **Thread Ownership**：client 线程单写。
10. **生命周期**：首次请求 PG 时创建 → 缓存复用 → 失效重查（同一对象更新）→ 随 client 存活。
11. **上游**：`process_leader_request`/`on_leader_acquired`。
12. **本层做什么**：缓存 Leader。
13. **为下一层准备**：`get_stub(leader_id, addr, port)`。
14. **何时退出**：`_leader_osd` 清空（client 销毁）。
15. **易混淆**：v.s. `pg_info_type.osds`（成员列表，可由 Monitor 获得；Leader 必须问 OSD）。
16. **源码**：`src/include/fastblock/client/fb_client.h:116`。

### 6.17 `write_ring_state`（+ `write_ring_slot_info`）

1. **一句话定义**：一条到 OSD 的 write-ring 优化通道状态（queue id / lease / slots）。
2. **它不是什么**：不是连接；不是请求；不是 OSD 端 ring 本体。
3. **所属层次**：Transport 优化层（默认开启）。
4. **为什么需要**：大 payload 用 RDMA WRITE 直写 OSD 预注册 slot，再用小 commit RPC 通知，绕开普通 RPC 收包解析路径。
5. **谁创建**：`get_or_create_write_ring_state()`（`:269`）。
6. **谁持有**：`fblock_client::_write_rings`（shared_ptr）；`ring_write_context` 持 shared_ptr 引用。
7. **关键字段**：`conn`、`queue_id`、`lease_us`、`next_slot`、`is_ready/is_onflight`、`slots`（`slot_info`：`remote_addr/remote_key/slot_size/busy`，`:130-135`）。
8. **关键方法**：`acquire_write_ring_async()`（`:457`）、`post_ring_write()`（`:496`）、`on_write_ring_ready()`（`:411`）、`acquire_write_ring_slot()`（`:479`）。
9. **Thread Ownership**：client 线程单写。
10. **生命周期**：首次写该 OSD 时创建 → 复用/租约 → 连接失效或进程结束。
11. **上游**：`process_request` 的 write 分支（`should_use_write_ring()`，`:200`）。
12. **本层做什么**：RDMA WRITE 数据 + commit RPC。
13. **为下一层准备**：远端 slot 地址/rkey 与 commit 请求。
14. **何时退出**：租约过期/连接失效/client 销毁。
15. **易混淆**：v.s. `connection`（ring 复用 connection，但状态独立）。
16. **源码**：`src/include/fastblock/client/fb_client.h:137`（state）、`:130`（slot）、`:496`（post_ring_write）。

### 6.18 `osd::write_request` / `osd::write_reply`

1. **一句话定义**：真正发给 OSD 的写请求/响应 protobuf message。
2. **它不是什么**：不是客户端生命周期帧；不是 `request_stack_type`。
3. **所属层次**：RPC payload。
4. **为什么需要**：跨进程协议的数据载体（protobuf 生成）。
5. **谁创建**：`write_object()` 里 `make_unique<osd::write_request>()`（`:1276`）；reply 在 `process_request` 写分支 `make_unique<osd::write_reply>()`（`:1159`）。
6. **谁持有**：`request_stack_type::req` / `resp`（unique_ptr，variant 内）。
7. **关键字段**（`proto/osd_msg.proto:16` / `:25`）：

| 字段 | 类型 | 含义 |
|---|---|---|
| `pool_id` | uint64 | 目标 pool |
| `pg_id` | uint64 | 目标 PG（`calc_target` 计算） |
| `object_name` | bytes | 对象名 |
| `offset` | uint64 | object 内偏移 |
| `data` | bytes | 分片数据 |
| `write_reply.state` | int32 | OSD 业务状态 |

8. **关键方法**：protobuf 生成（`SerializeToArray`/`ParseFromArray`）。
9. **Thread Ownership**：被 client 线程构造；被 RPC 层序列化。
10. **生命周期**：随 `request_stack_type`；serialize 后仍在 stack 内存活到完成/重试。
11. **上游**：`fblock_client::write_object()`。
12. **本层做什么**：承载 5 元组 payload。
13. **为下一层准备**：交给 Stub 序列化经 RDMA 发送。
14. **何时退出**：随 stack 析构。
15. **易混淆**：v.s. `request_meta`（RPC 头，不是业务 payload）。
16. **源码**：`proto/osd_msg.proto:16/25`、生成类 `src/include/fastblock/rpc/osd_msg.pb.h`。

### 6.19 `osd::read_request` / `read_reply`

1. **一句话定义**：读请求/响应 protobuf（`read_reply` 携带 `data`）。
2. **它不是什么**：不是聚合 buffer（那是 `read_source::buf`）。
3. **所属层次**：RPC payload。
4. **为什么需要**：读的 payload 在响应里返回。
5. **谁创建**：`read_object()`（`:1307`）/ `process_request` 读分支（`make_unique<osd::read_reply>()`）。
6. **谁持有**：`request_stack_type::req` / `resp`。
7. **关键字段**：`pool_id/pg_id/object_name/offset/length`（proto `:30`）；reply `state/data`（`:39`）。
8. **关键方法**：protobuf 生成。
9. **Thread Ownership**：同 write。
10. **生命周期**：同 write。
11. **上游**：`fblock_client::read_object()`。
12. **本层做什么**：读 payload 载体。
13. **为下一层准备**：`data` 经 `process_response` → `read_object_callback` → `read_source::read_done` → 拼入 `buf`。
14. **何时退出**：随 stack 析构。
15. **易混淆**：v.s. `read_source::buf`（客户端聚合缓冲）。
16. **源码**：`proto/osd_msg.proto:30/39`。

### 6.20 `rpc_service_osd_Stub`

1. **一句话定义**：远端 OSD 服务（`rpc_service_osd`）的本地代理。
2. **它不是什么**：不是连接；不实现传输；protoc 生成。
3. **所属层次**：RPC 层。
4. **为什么需要**：把“调用一个远端方法”变成“把请求交给 RpcChannel”。
5. **谁创建**：`fblock_client::get_stub()`（`:588`）与连接就绪回调（`:382`），存 `_stubs[conn_id]`。
6. **谁持有**：`fblock_client::_stubs`（unique_ptr）；`request_stack_type::stub` 是裸指针借引用。
7. **关键字段**：`channel_`（`RpcChannel*`，即 `connection`；`osd_msg.pb.h:5639`）。
8. **关键方法**：`process_write()`（`osd_msg.pb.h:5593`）→ `channel_->CallMethod(...)`；`process_read/process_delete/process_get_leader/process_acquire_write_ring/process_commit_ring_write`。
9. **Thread Ownership**：随 client 线程。
10. **生命周期**：连接就绪创建 → 每次 RPC 调用 → 连接失效随 `_stubs` 清理。
11. **上游**：`process_request`（写 `/`:1160`）。
12. **本层做什么**：方法名 + 参数转交 RpcChannel。
13. **为下一层准备**：`connection::CallMethod`。
14. **何时退出**：`invalidate_connection`/client 销毁。
15. **易混淆**：v.s. `connection`（stub 是“调哪个方法”，connection 是“从哪条连接走”）。
16. **源码**：`src/include/fastblock/rpc/osd_msg.pb.h:5579`、`:5593`。

### 6.21 `msg::rdma::rpc_controller`

1. **一句话定义**：protobuf `RpcController` 实现，记录传输失败与错误文本。
2. **它不是什么**：不是业务状态；不参与路由。
3. **所属层次**：RPC 层。
4. **为什么需要**：区分“传输失败”（如 -ENOLINK）与“OSD 业务失败”（`reply.state`）。
5. **谁创建**：随 `request_stack_type`（`:98`）；OSD 服务端另有一份（带 PD/peer 地址）。
6. **谁持有**：`request_stack_type::ctrlr`（unique_ptr）。
7. **关键字段**：`_failed`、`_error_reason`、`_pd`、`_peer_address`（`rpc_controller.h:22-72`）。
8. **关键方法**：`Failed()`（`:37`）、`ErrorText()`（`:39`）、`SetFailed()`（`:43`）、`Reset()`、`attach_pd/peer_address`。
9. **Thread Ownership**：随 stack；RPC 层在响应时写，client 在 `process_response` 读。
10. **生命周期**：随 stack；retry 时 `Reset()`。
11. **上游**：`process_request` 传入 Stub。
12. **本层做什么**：承载传输成败。
13. **为下一层准备**：`process_response` 的 `ctrlr->Failed() ? -ENOLINK : resp->state()`。
14. **何时退出**：随 stack。
15. **易混淆**：v.s. `write_reply::state`（业务状态）。
16. **源码**：`src/include/fastblock/msg/rpc_controller.h:22`。

### 6.22 `msg::rdma::client`

1. **一句话定义**：RDMA RPC 传输客户端（连接管理 + 内存池 + CQ poller）。
2. **它不是什么**：不是业务 client；不是 `fblock_client`。
3. **所属层次**：Transport 层。
4. **为什么需要**：把 protobuf RPC 变成 RDMA 报文并管理连接生命周期。
5. **谁创建**：`fblock_client` 构造 `make_shared<msg::rdma::client>(name, thd, opts)`（`:735`）。
6. **谁持有**：`fblock_client::_rpc_client`（shared_ptr）。
7. **关键字段**：`_opts`（`options`，`:59`）、`_meta_pool`/`_data_pool`（`client.h:1298-1305`）、`_wc`/CQ poller、连接表。
8. **关键方法**：`start()`、`emplace_connection()`（`:1802`）、`remove_connection()`、`is_start()`（`:1820`）。
9. **Thread Ownership**：绑定构造时传入的 thread。
10. **生命周期**：随 `fblock_client`。
11. **上游**：`fblock_client::ensure_connection`。
12. **本层做什么**：建连/断连、连接池、DMA 池。
13. **为下一层准备**：`connection`（可作 RpcChannel）。
14. **何时退出**：client 停止。
15. **易混淆**：v.s. `monitor::client`（完全不同的类，同名“client”）。
16. **源码**：`src/include/fastblock/msg/rdma/client.h:55`。

### 6.23 `msg::rdma::client::connection`

1. **一句话定义**：一条到 OSD 的 RDMA 连接，同时实现 protobuf `RpcChannel`。
2. **它不是什么**：不是线程；不是 stub；不是请求。
3. **所属层次**：Transport 层。
4. **为什么需要**：protobuf 只给 RpcChannel 抽象，需要自己实现 RDMA 传输、请求匹配、响应分发。
5. **谁创建**：`msg::rdma::client`（连接建立流程）；`on_connection_ready` 中存入 `_stubs`。
6. **谁持有**：`msg::rdma::client`、`_stubs` 的构造参数、`write_ring_state::conn`（shared_ptr）。
7. **关键字段**：`_sock`（socket/QP）、`_onflight_requests` / `_priority_onflight_requests`、`_unresponsed_requests`、`_wait_read_requests`、`cqe_list`、`_dispatch_id`。
8. **关键方法**：

| 方法 | 作用 | 下一步 |
|---|---|---|
| `CallMethod()`（`:510`） | RpcChannel 入口：req_key+meta+入队 | `enqueue_request` |
| `enqueue_request()`（`:394`） | 建 `transport_data`，入连接队列 | 连接 poller |
| `process_request_once()`（`:330`） | 序列化 + post WR | `ibv_post_send` |
| `handle_poll()`（`:449`） | 连接主循环：发送/接收/完成 | `handle_cqe` |
| `post_send_wr()`（`:1139`）/`post_external_send_wr()`（`:1148`） | 直接 post（ring 用） | socket |

9. **Thread Ownership**：绑定 client thread；poller 在其上。
10. **生命周期**：按目标 OSD 建立/复用/失效（`invalidate_connection`）。
11. **上游**：`Stub`/`std::make_shared<Stub>(conn)`。
12. **本层做什么**：发送/接收/RPC 匹配/回调 closure。
13. **为下一层准备**：`socket::send`（QP）。
14. **何时退出**：连接关闭或 client 停止。
15. **易混淆**：v.s. `socket`（connection 管 RPC 语义，socket 管 QP/verbs）。
16. **源码**：`src/include/fastblock/msg/rdma/client.h:152`。

### 6.24 `connection::rpc_request`

1. **一句话定义**：传输层内部的单个 RPC 任务（在途表元素）。
2. **它不是什么**：不是 `request_stack_type`；不是 protobuf message。
3. **所属层次**：Transport 层。
4. **为什么需要**：序列化前需要把 `method/ctrlr/request/response/closure/meta` 绑定，并生成 `req_key` 用于响应匹配。
5. **谁创建**：`CallMethod()` 中 `make_unique<rpc_request>(...)`（`:557`）。
6. **谁持有**：连接队列 `_onflight_requests` / `_unresponsed_requests`（unique_ptr）。
7. **关键字段**：

| 字段 | 类型 | 含义 |
|---|---|---|
| `request_key` | `correlation_index_type` | 响应匹配键 |
| `method` | `MethodDescriptor*` | 方法描述 |
| `ctrlr` / `request` / `response` / `closure` | 指针 | 来自 `request_stack_type` |
| `meta` | `unique_ptr<char[]>` | `request_meta` 副本 |
| `request_data` / `reply_data` | `unique_ptr<transport_data>` | 发送/接收内存组织 |
| `start_at` | time_point | 超时判定 |

8. **关键方法**：无（传输层处理）。
9. **Thread Ownership**：connection 线程。
10. **生命周期**：CallMethod 创建 → `enqueue_request` → `process_request_once` 发出 → 进 `_unresponsed_requests` → 响应 `closure->Run()` → erase。
11. **上游**：`Stub::process_write` → `CallMethod`。
12. **本层做什么**：绑定 RPC 与回调，维持到响应。
13. **为下一层准备**：`transport_data` 序列化与 WR。
14. **何时退出**：响应完成或连接失败（`free_resources` 统一 Run closure）。
15. **易混淆**：v.s. `request_stack_type`（上层生命周期）。
16. **源码**：`src/include/fastblock/msg/rdma/client.h:156`。

### 6.25 `request_meta`

1. **一句话定义**：RPC 头：`service_name + method_name + data_size`（字符串编码，无数字 id）。
2. **它不是什么**：不是业务 payload；不是 protobuf message 本体。
3. **所属层次**：Transport 层。
4. **为什么需要**：OSD 端靠它分发到正确的 service/method。
5. **谁创建**：`make_request_meta()`（`types.h:95`），在 `CallMethod` 中调用（`:545`）。
6. **谁持有**：`rpc_request::meta`（`unique_ptr<char[]>`）。
7. **关键字段**：`service_name[]/service_name_size`、`method_name[]/method_name_size`、`data_size`。
8. **关键方法**：`make_request_meta()`（`:95`）、`load_request_meta()`（`:110`）、`store_request_meta()`（`:116`）。
9. **Thread Ownership**：随 RPC。
10. **生命周期**：与 `rpc_request` 同。
11. **上游**：Stub 生成的 `method` 描述符。
12. **本层做什么**：打包 RPC 头。
13. **为下一层准备**：序列化时放在 payload 前。
14. **何时退出**：响应后随 `rpc_request` 释放。
15. **易混淆**：v.s. `metadata`（`transport_data` 内部的多段元数据）——不同概念。
16. **源码**：`src/include/fastblock/msg/rdma/types.h:82`。

### 6.26 `transport_data`

1. **一句话定义**：RPC 消息在 RDMA 内存里的组织者（SGE/WR 链 + 序列化目标）。
2. **它不是什么**：不是业务对象；不拥有内存池（只是借块）。
3. **所属层次**：Transport 层。
4. **为什么需要**：数据必须先写进注册内存（MR）才能被网卡 DMA。
5. **谁创建**：`enqueue_request()`（`:396`）为发送；响应侧由 `connection`（reply_data）。
6. **谁持有**：`rpc_request::request_data/reply_data`（unique_ptr）。
7. **关键字段**：`_meta_pool/_data_pool`、`_metas/_datas`（借来的 `net_context*`）、`_transport_size/_serialized_size`、tag（完整性标记）。
8. **关键方法**：`serialize_data()`（`:283`，`SerializeToArray` 进 DMA 内存）、`make_send_request()`（`:425`，构造 WR/SGE）、`unserialize_data()`（`:333`，响应解析）、析构归还池（`:103`）。
9. **Thread Ownership**：connection 线程。
10. **生命周期**：入队时创建 → 发送/接收 → 完成后析构归还。
11. **上游**：`connection::enqueue_request`。
12. **本层做什么**：序列化 + 分块 + SGE/WR 组织。
13. **为下一层准备**：`socket::send` 可直接 post 的 WR 链。
14. **何时退出**：请求完成后析构。
15. **易混淆**：v.s. `memory_pool`（池拥有内存，transport_data 借用）。
16. **源码**：`src/include/fastblock/msg/rdma/transport_data.h:29`。

### 6.27 `memory_pool` / `net_context`

1. **一句话定义**：预注册 MR 的 DMA 内存池；每块一个 `net_context{mr, wr, sge}`。
2. **它不是什么**：不是业务 buffer；不由 QoS/客户端直接管理。
3. **所属层次**：Transport 层。
4. **为什么需要**：RNIC 只能访问注册内存；提前注册避免每请求 reg/dereg。
5. **谁创建**：`msg::rdma::client` 构造（`client.h:1298-1305`）。
6. **谁持有**：`msg::rdma::client`（shared_ptr）；`transport_data` 借块并归还。
7. **关键字段**（`memory_pool.h:35`）：`mr`、`wr`、`sge`；池：`_capacity/_element_size`、freelist。
8. **关键方法**：`get()`（`:119`）、`put()`（`:129`）、构造中逐块 `ibv_reg_mr`（`:81`）、`free()`。
9. **Thread Ownership**：client 线程。
10. **生命周期**：进程/线程级；块循环复用。
11. **上游**：`transport_data`。
12. **本层做什么**：注册内存的分配/回收。
13. **为下一层准备**：`sge.addr/lkey` 与 WR 模板。
14. **何时退出**：client 停止。
15. **易混淆**：v.s. OSD 侧 write-ring slot（另一套独立注册内存）。
16. **源码**：`src/include/fastblock/msg/rdma/memory_pool.h:31`。

### 6.28 `completion_queue`

1. **一句话定义**：RDMA CQ 的 C++ 封装（创建 + poll）。
2. **它不是什么**：不是 poller；不是请求。
3. **所属层次**：Transport 层。
4. **为什么需要**：完成事件是异步 IO 的核心；必须有对象管理 `ibv_cq`。
5. **谁创建**：`msg::rdma::client` 构造 `make_shared<completion_queue>(...)`（`client.h:1292`）。
6. **谁持有**：`msg::rdma::client`（shared_ptr）。
7. **关键字段**：`_cq`（`ibv_cq`）、`poll_size`。
8. **关键方法**：ctor `ibv_create_cq`（`cq.h:178`）、`poll()`（`:202`，`ibv_poll_cq`）。
9. **Thread Ownership**：client 线程 poll。
10. **生命周期**：进程/线程级。
11. **上游**：client core poller。
12. **本层做什么**：把 CQE 批量取出并分发到 connection。
13. **为下一层准备**：`connection::handle_cqe`。
14. **何时退出**：client 停止。
15. **易混淆**：v.s. `spdk_poller`（软件轮询器，不是硬件队列）。
16. **源码**：`src/include/fastblock/msg/rdma/cq.h:34`。

### 6.29 `socket`

1. **一句话定义**：一条 RDMA 连接的 QP/verbs/cm 封装。
2. **它不是什么**：不是 RPC；不管 service/method。
3. **所属层次**：Transport 层（最底层）。
4. **为什么需要**：把 `rdma_cm`、QP、`ibv_post_send/recv` 封装成对象。
5. **谁创建**：连接建立时（`msg::rdma::client`）。
6. **谁持有**：`connection::_sock`（unique_ptr）。
7. **关键字段**：`_id`（`rdma_cm_id`，含 QP）、`_pd`、`_provider`。
8. **关键方法**：`send()`（`:370`，`ibv_post_send` `:372`）、`receive()`（`:384`，`ibv_post_recv` `:386`）、`create_qp()`（`:786`）、cm 事件处理。
9. **Thread Ownership**：connection 线程。
10. **生命周期**：随 connection。
11. **上游**：`connection::process_request_once` / `post_send_wr`。
12. **本层做什么**：post WR 到 QP；post RECV。
13. **为下一层准备**：RNIC 执行 → CQ。
14. **何时退出**：连接关闭。
15. **易混淆**：v.s. `connection`（socket 是裸传输，connection 是 RPC 语义）。
16. **源码**：`src/include/fastblock/msg/rdma/socket.h:43`。

### 6.30 `osd_service`（服务端入口补充）

> 仅在主流程边界处补充：OSD 端如何进入 handler；不展开 Raft/LocalStore。

1. **一句话定义**：OSD 侧 `rpc_service_osd` 服务实现（handler 集合）。
2. **它不是什么**：不是客户端类型；不是 RDMA 层。
3. **所属层次**：服务端入口。
4. **为什么需要**：接收 `process_write/read/delete/get_leader` 并分派到 PG/raft。
5. **谁创建**：OSD 启动 `global_osd_service = make_unique<osd_service>(...)`（`src/osd/osd.cc:330`）。
6. **谁持有**：OSD 进程全局；注册进 RDMA server（`osd.cc:315-316`）。
7. **关键字段**：`_pm`（partition_manager）、`_monitor_client`、write-ring 队列表。
8. **关键方法**：`process_write()`（`osd_service.h:71`）→ 模板 `process<>`（`:215`）→ PG 查找/Leader 检查 → `osd_stm::write_and_wait`。
9. **Thread Ownership**：在 shard thread 上执行。
10. **生命周期**：进程级。
11. **上游**：RDMA server 按 `request_meta` 分发（`server.h:649`）。
12. **本层做什么**：找 PG、校验 Leader、转 Raft。
13. **为下一层准备**：交给 Raft（本文边界外）。
14. **何时退出**：OSD 停止。
15. **易混淆**：v.s. `fblock_client`（一个是服务端 handler，一个是客户端）。
16. **源码**：`src/osd/osd_service.h:29`、`:71`、`:215`。

---

## 7. 最容易混淆的类型对比

### 7.1 `libblk_client` vs `fblock_client`

| 维度 | `libblk_client` | `fblock_client` |
|---|---|---|
| 处理什么 | Logical IO（连续 offset/len） | Object 请求 |
| 拆分 Object | **是** | 否 |
| 持有请求队列 | 否 | **是**（`_requests`/`_on_flight_requests`） |
| PG/Leader | 否 | **是** |
| parent 聚合 | **是**（`write_source`/`read_source`） | 否（只做单 child 回调） |
| RPC | 否 | **是**（Stub/connection） |
| ownership | `global::blk_clients` | `libblk_client::_client` |

为什么分两层：把“块语义”和“对象/路由/RPC 语义”解耦；`fblock_client` 复用度高（工具/测试直接使用），`libblk_client` 则是 bdev 场景的适配层。

### 7.2 `write_source` vs `request_stack_type`

- `write_source` 代表 **Logical WRITE**；一个 10 MiB WRITE 有 **1 个**。
- `request_stack_type` 代表 **Object child**；同样场景有 **3 个**（4+4+2 MiB）。
- 关系：3 个 stack 的 `ctx` 都指向同一个 `write_source`；`write_done` 被调用 3 次，`obj_num` 3→0。

### 7.3 `request_stack_type` vs `osd::write_request`

- `request_stack_type`：客户端生命周期上下文（含 stub/ctrlr/resp/cb/ctx），**retry 时复用同一个**。
- `osd::write_request`：真正发给 OSD 的 protobuf payload；retry 时同样被复用（`stack->_req` 不重建）。

### 7.4 stub vs connection

- `stub`：方法代理（“调哪个方法”），构造时绑定一个 `RpcChannel*`。
- `connection`：RpcChannel 的实现（“从哪条连接走、怎么发/收”）。
- 调用关系：`stub->process_write()` → `channel_->CallMethod()`。

### 7.5 `rpc_controller` vs `write_reply`

- `rpc_controller`：**传输层**成败（超时、断连），失败时 `Failed()==true`。
- `write_reply::state`：**OSD 业务层**结果（如 `RAFT_ERR_NOT_LEADER`）。
- 两者都需要：`process_response` 先看 `ctrlr->Failed()`（→ -ENOLINK）再看 `resp->state()`。

### 7.6 shard vs core vs SPDK thread

- `shard`：core 列表中的**下标**（`utils::get_current_shard_id`）。
- `core`：SPDK 的核 id（可能不连续）。
- `SPDK thread`：`core_sharded` 为每个 shard core 创建的线程（单核 cpumask）。
- 关系：`shard → core → thread` 一一对应；client 对象挂在 shard 槽位。

---

## 8. 完整 10 MiB WRITE 实例

```text
Volume A，WRITE 10 MiB，object_size 4 MiB，consumer worker = T1
```

| 步骤 | 当前主角类 | 新创建对象 | 谁持有 | 为下一层准备 |
|---|---|---|---|---|
| 1 | `spdk_bdev_io` + `bdev_fastblock` | — | SPDK | offset/len/iov |
| 2 | `bdev_fastblock_io_channel` | （channel 已存在） | SPDK | 本线程 shard 上下文 |
| 3 | `global::blk_clients[T1.shard]` → `libblk_client` | — | 全局 vector | 找到本线程 client |
| 4 | `libblk_client::write()` | **`write_source` ×1**（obj_num=3） | child 回调的 ctx | 拆出 3 个 Object |
| 5 | `fblock_client::write_object()` ×3 | **`osd::write_request` ×3** | stack.req | 3 个 payload |
| 6 | `fblock_client::send_request()` ×3 | **`request_stack_type` ×3** + `rpc_controller` ×3 | `_requests` | 3 个待发帧 |
| 7 | `handle_send_request()` ×3 | `leader_osd_info`（首个 PG 查 Leader 时） | `_leader_osd` | 路由决策 |
| 8 | `process_leader_request()` | `leader_request_stack_type` | `_leader_requests` | get_leader |
| 9 | `on_leader_acquired()` | — | `_leader_osd` | leader 缓存 |
| 10 | `process_request()` ×3 | `osd::write_reply` ×3 | stack.resp | 发送 |
| 11 | `Stub::process_write` | `connection`（首次）+ `Stub` | `_stubs` | RpcChannel |
| 12 | `connection::CallMethod()` | `rpc_request` ×3 | 连接队列 | req_key/meta |
| 13 | `transport_data` | 借用 `memory_pool` 块 ×3 | rpc_request | WR/SGE |
| 14 | `socket::send()` | `ibv_send_wr` | — | RDMA 发送 |
| 15 | 响应：Object0 success | `on_response` 置位 | `_on_flight_requests` | callback |
| 16 | `process_response()` | — | — | `write_source::write_done`：3→2 |
| 17 | Object1 `RAFT_ERR_NOT_LEADER` | — | — | **retry_request**（同一 stack 回 `_requests`；leader 失效重查；obj_num 仍=2） |
| 18 | Object1 重发成功 | — | — | 2→1 |
| 19 | Object2 终态错误 | — | — | result=error；1→0 |
| 20 | `write_source::invoke()` | — | — | `bdev_fastblock_write_callback` |
| 21 | `spdk_bdev_io_complete` | — | — | 上层 IO 完成 |

对象数量小结：

```text
bdev_io:            1（SPDK）
write_source:       1
Object:             3
request_stack_type: 3
osd::write_request: 3
osd::write_reply:   3
rpc_controller:     3
connection:         1（首次建，之后复用）
```

---

## 9. 完整对象关系总图

```text
Application / VM
       |
       v
SPDK bdev consumer（worker thread T1）
       |
       v
spdk_bdev_io  ──────────────►  bdev_fastblock (ctxt)
       |                              ^
       v                              |
bdev_fastblock_write() / get_buf_cb() |
       |  按 current shard            |
       v                              |
global::blk_clients[shard] ──► libblk_client
                                   |  unique_ptr
                                   v
                              fblock_client (_current_thread)
                                   |
                    +--------------+---------------+
                    |                              |
              _leader_osd                    _requests / _on_flight_requests
                    |                              |
                    v                              v
             leader_osd_info                request_stack_type
                                                   |
                    +------------------------------+-------------------+
                    |              |               |                  |
                    v              v               v                  v
         osd::write_request  osd::write_reply  rpc_controller   ring_write_context(optional)
                    |
                    v
             rpc_service_osd_Stub  ──►  connection (RpcChannel)
                                            |
                                            v
                                    connection::rpc_request
                                            |
                                            v
                                      transport_data  ──借块──► memory_pool/net_context
                                            |
                                            v
                                         socket (QP)
                                            |
                                            v
                                     completion_queue (CQ)
                                            |
                                            v
                                          OSD
```

---

## 10. 创建关系图

```text
SPDK_BDEV_MODULE_REGISTER(fastblock_if)          # 模块注册（进程启动）
   ↓
bdev_fastblock_create()                          # JSON-RPC 创建 bdev
   ↓  spdk_io_device_register(..., bdev_fastblock_create_cb, ...)
channel create（consumer 某线程首次拿 channel）
   ↓
bdev_fastblock_create_cb()
   ├── bdev_fastblock_io_channel
   └── libblk_client  →  fblock_client  →  msg::rdma::client
                                                ├── memory_pool（meta/data）
                                                └── completion_queue

Logical WRITE（bdev_fastblock_write）
   ↓
libblk_client::write()
   ├── write_source
   └── per Object:
          fblock_client::write_object()
             ├── osd::write_request
             └── send_request()
                    └── request_stack_type (+ rpc_controller)
                           ↓ process_request
                           ├── osd::write_reply
                           ├── Stub（首次）
                           ├── connection（首次）
                           └── connection::rpc_request
                                  └── transport_data
```

---

## 11. 持有关系图（创建者 ≠ 持有者）

```text
创建者                        持有者（owner）
-----------------------------------------------
bdev_fastblock_create      →  SPDK bdev（disk.ctxt）
bdev_fastblock_create_cb   →  SPDK io_channel（ctx_buf）
common.cc                  →  global::blk_clients（shared_ptr）
bdev_fastblock_create_cb   →  global::blk_clients[shard]（shared_ptr）
libblk_client ctor         →  libblk_client::_client（unique_ptr）
send_request               →  _requests / _on_flight_requests / 局部 unique_ptr
write_object               →  request_stack_type::req（unique_ptr）
process_request            →  request_stack_type::resp（unique_ptr）
send_request               →  request_stack_type::ctrlr（unique_ptr）
connection::CallMethod     →  连接队列（unique_ptr）
transport_data             →  rpc_request（unique_ptr），内存归 memory_pool
get_stub / on_connection_ready →  fblock_client::_stubs（unique_ptr）
libblk_client::write       →  裸 write_source（靠回调集合“共同持有”）
```

关键：`write_source` 没有智能指针 owner；它的“所有权”是每个 child 回调的 `ctx`，最后一个回调负责 `delete`。

---

## 12. 线程归属图

```text
Thread T0（shard 0）                 Thread T1（shard 1）
  ├── io_channel0                      ├── io_channel1
  ├── libblk_client0                   ├── libblk_client1
  ├── fblock_client0                   ├── fblock_client1
  │     ├── _requests0                 │     ├── _requests1
  │     ├── _leader_osd0               │     ├── _leader_osd1
  │     └── _rpc_client0               │     └── _rpc_client1
  │           ├── connection(s)0       │           ├── connection(s)1
  │           └── memory_pool0         │           └── memory_pool1
  └── 处理 T0 提交的 Logical IO          └── 处理 T1 提交的 Logical IO
        （write_source0）                    （write_source1）

跨线程：send_request 发现当前线程 != _current_thread 时
        spdk_thread_send_msg(_current_thread, do_send_request, stack)
        —— 投递任务，不移动线程
```

---

## 13. 人话类比（仅辅助，后面必须看真实定义）

| 类型                   | 类比               | 真实定义                                  |
| -------------------- | ---------------- | ------------------------------------- |
| `libblk_client`      | 拆单员              | Logical IO → Object 拆分 + parent 聚合    |
| `fblock_client`      | 发货调度员            | Object→PG→Leader→RPC 生命周期与队列          |
| `write_source`       | Logical WRITE 总账 | parent completion context（obj_num 计数） |
| `read_source`        | 读的总账 + 拼图板       | parent context + 聚合 buffer            |
| `request_stack_type` | 一个 Object 的档案袋   | 异步请求的完整状态帧                            |
| `osd::write_request` | 真正寄出的货单          | 线上 protobuf payload                   |
| `rpc_controller`     | 运输过程状态单          | 传输成败 + 错误文本                           |
| `connection`         | 一条运输线路           | RDMA 连接 + RpcChannel                  |
| `stub`               | 线路上的发货窗口         | 远端服务方法代理                              |
| `memory_pool`        | 已备案的仓库           | 预注册 MR 的 DMA 内存池                      |
| `write_ring_state`   | 对方预留的收货位         | OSD 预注册 slot 的本地视图                    |

---

## 14. 类出现顺序速查（WRITE 主路径）

```text
01. spdk_bdev_io
02. bdev_fastblock
03. spdk_bdev_module / fastblock_fn_table
04. bdev_fastblock_io_channel
05. global::blk_clients
06. libblk_client
07. write_source
08. fblock_client
09. leader_key_type
10. leader_osd_info
11. osd::write_request
12. request_stack_type
13. msg::rdma::rpc_controller
14. osd::write_reply
15. rpc_service_osd_Stub
16. msg::rdma::client
17. connection
18. connection::rpc_request
19. request_meta
20. transport_data
21. memory_pool / net_context
22. socket
23. completion_queue
（READ 额外：read_source、osd::read_request/read_reply）
```

---

## 15. 如果只记一句话

| 类型 | 只记一句话 |
|---|---|
| `spdk_bdev_io` | SPDK 的一次块 IO（BACKGROUND） |
| `bdev_fastblock` | 一个 volume/bdev 的句柄（pool/image/object_size） |
| `bdev_fastblock_io_channel` | per-thread bdev 通道上下文，并触发该 shard 的 client 创建 |
| `bdev_fastblock_group_channel` | 模块级 epoll 组（管理事件，不在 WRITE 主路径） |
| `fastblock_fn_table` | FastBlock 挂进 SPDK 的方法表 |
| `core_sharded` | shard → core → SPDK thread 的索引 + `invoke_on` |
| `global::blk_clients` | 每 shard 一个 client 的全局表 |
| `simple_poller` | 把回调注册到目标线程的 poller 小工具 |
| `monitor::client` | PG/OSD 路由元数据来源 |
| `libblk_client` | Logical IO 拆 Object 的 wrapper |
| `write_source` | Logical WRITE 的 parent 完成聚合器 |
| `read_source` | Logical READ 的 parent 聚合器（带 buffer） |
| `fblock_client` | 管 Object→PG→Leader→RPC 生命周期与队列 |
| `request_stack_type` | 一个 Object RPC 的状态帧 |
| `leader_key_type` | `(pool, pg)` 打包的 key |
| `leader_osd_info` | PG Leader 缓存 |
| `write_ring_state` | write-ring 优化通道状态 |
| `osd::write_request` | 发给 OSD 的写 payload |
| `osd::write_reply` | OSD 写响应（业务 state） |
| `osd::read_request/read_reply` | 读 payload/响应（含 data） |
| `rpc_service_osd_Stub` | 远端 OSD 方法代理 |
| `rpc_controller` | 传输成败状态 |
| `msg::rdma::client` | RDMA RPC 客户端（连接+内存池+CQ） |
| `connection` | 一条 RDMA 连接（RpcChannel） |
| `connection::rpc_request` | 传输层在途 RPC 任务 |
| `request_meta` | RPC 头（service/method/size） |
| `transport_data` | 序列化 + SGE/WR 组织者 |
| `memory_pool/net_context` | 预注册 MR 的 DMA 块 |
| `completion_queue` | CQ 封装（poll 完成） |
| `socket` | QP/verbs 裸传输 |
| `osd_service` | 服务端 handler 入口 |

---

## 16. 审计覆盖检查

```text
Forward path checked: YES（bdev_fastblock_submit_request → … → ibv_post_send）
Response path checked: YES（CQE → handle_cqe → on_response → process_response）
Retry path checked: YES（process_response → retry_request → _requests → 重新 get_leader/发送）
Completion path checked: YES（child cb → write_source → bdev callback → spdk_bdev_io_complete）
Transport objects checked: YES（Stub/connection/rpc_request/request_meta/transport_data/memory_pool/CQ/socket）
Per-thread/client objects checked: YES（io_channel/libblk_client/fblock_client/core_sharded/blk_clients）
Server entry checked: YES（osd_service::process_write，仅边界）
```

覆盖方式：先沿 `bdev_fastblock_submit_request` 正向走一次并记录类型；再从 `connection::handle_cqe`/`on_response` 反向走一次；最后沿 retry 与 completion 各走一次，对照本文清单核对。

---

## 17. 未证实事项（NOT PROVEN FROM CURRENT SOURCE）

- SPDK channel 语义（“IO 一定运行在创建 channel 的线程上”）属 SPDK 行为，FastBlock 未显式断言（`BACKGROUND ONLY`）。
- `spdk_thread` 是否会迁移到其它 core。
- `global::blk_client`（单数）的用途（只声明未使用）。
- `bdev_fastblock_io_channel` 除 eventfd/epoll 之外是否影响 IO 线程选择（当前 WRITE 路径不读它）。
- write-ring 在租约/高压下的精确时序。
- OSD 端 `process_write` 之后的行为（Raft/LocalStore/Blobstore）不在本审计范围。
