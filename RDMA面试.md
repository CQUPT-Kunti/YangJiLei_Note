# RDMA 高并发面试题与参考回答（对象存储 Demo 版）

> **使用方法**：先背“30 秒项目介绍”和 ★ 题；其余题按面试追问复习。回答刻意保持简短，但保留关键因果关系。
>
> **范围说明**：依据你提供的《RDMA》笔记和我们讨论的 CQUPT Object Storage RDMA **设计方案**。以下“本 Demo”指计划采用的架构，**不意味着所有功能已经完成或测出了性能提升**。涉及 SRQ、共享 CQ、批处理、流控等内容是**高并发延伸问题**，并非要求当前 Demo 全部实现。

## 先背：30 秒项目介绍

**问题：你这个 RDMA 对象存储项目是怎么设计的？**

**回答：**我的设计是在原有对象存储传输路径之外增加一个**异步 RDMA Fast Path**，默认优先使用 RDMA，连接等明确安全失败时回退到原来的 gRPC。控制消息仍用 Protobuf 描述，通过简单的 `RdmaConnection : google::protobuf::RpcChannel` 与 RDMA SEND/RECV 传输；大块 Chunk 数据支持两种方式：**Pull（Store RDMA READ Client 内存）**和 **Push（Client RDMA WRITE Store 分配给自己的 Slot）**。Client 和每个 Store 的 MVP 连接使用一个 RC QP；完成通知由简单的 CQ 处理线程驱动，最终仍调用原有 `ChunkStore::WriteChunk()` 并等待持久化 ACK。暂时不引入 SPDK、通用内存池和复杂 RPC Runtime。

**一句话重点：**这是一个**异步、可回退、保留原存储语义**的 RDMA Demo，而不是重新造一个对象存储或 FastBlock。

---

## 一、项目设计与架构（★ 优先复习）

**本节小结：**讲清楚“为什么加 RDMA、加在哪里、为什么不重写原有模块”。

### Q01 ★ 为什么要给对象存储加入 RDMA？
**答：**对象存储会频繁传输较大的 Chunk。RDMA 可让网卡直接访问已经注册的用户态内存，减少传统网络协议栈参与和部分内存复制，适合研究大块数据传输的 CPU 开销与延迟。不过最终速度还取决于磁盘、网络、注册内存开销和并发度，不能只凭用了 RDMA 就断言性能一定更高。

### Q02 ★ 整体架构如何分层？
**答：**分为**业务层**（`ObjectTransfer` / 当前 Storage transfer）、**传输选择层**（优先 RDMA、否则原 gRPC）、**RDMA 通信层**（Connection、QP/CQ/MR、异步完成），以及**存储适配层**（把收到的 Chunk 交给 `ChunkStore`）。RDMA 底层放在独立的 `modules/rdma/`，存储业务不直接操作 verbs。

### Q03 ★ 为什么 RDMA 只先优化数据面，不替换 Metadata/Raft 全部 RPC？
**答：**RDMA 最直接的价值在大量数据搬运。Metadata、心跳、Raft 等小消息如果全部替换，复杂度可能超过收益。Demo 保留原控制/一致性机制，把改动集中在 Chunk 读写路径，能更容易对比与回退。

### Q04 ★ 为什么选 `google::protobuf::RpcChannel`？它负责什么？
**答：**它是 Protobuf 旧 Generic Services 中 Stub 调用底层 RPC 实现的抽象接口，核心是 `CallMethod()`。我用它来学习“生成的 Stub → 序列化 → 自定义 RDMA 传输 → 服务端调用”的完整链路；它**不是** QP 或 RDMA 网卡连接本身。这个 API 已弃用，新版本 Protobuf 生成 Generic Service Stub 可能需要兼容选项，因此 Demo 会单独确认版本。

### Q05 ★ 为什么不直接照搬 FastBlock？
**答：**FastBlock 面向更完整的高性能存储 RPC Runtime，包含 SPDK 线程、复杂异步调度与资源池。我的目标是理解完整流程，所以只保留 `RpcChannel`、RDMA CM、verbs、异步 CQ、READ/WRITE 与必要生命周期；不使用 SPDK，也不复刻其 poller 框架。

### Q06 ★ 为什么 `modules/rdma/` 要独立？
**答：**`RdmaConnection`、CQ、MR 和 RDMA READ/WRITE 是通用通信能力，不应该依赖 `ChunkStore`。`StorageRdmaService` 则是存储适配器，负责校验和落盘。分离后，未来 Repair/Rebalance 也有机会复用 RDMA 通信层。

### Q07 ★ Demo 的“异步”具体体现在哪里？
**答：**`CallMethod()` 或 `AsyncWrite()` 提交 WR 后**不等待最终 RPC 响应**就返回；CQ 处理线程在完成事件到来后根据 `wr_id` / `request_id` 找到对应上下文，再执行回调。**异步不等于不调用 `ibv_poll_cq()`**，而是不让业务线程阻塞等待完成。

---

## 二、RDMA 基础、连接和内存（★ 必问题）

**本节小结：**能讲清楚 Context/PD/QP/CQ/MR 的职责，以及 READ、WRITE、SEND 的根本区别。

### Q08 ★ Context、PD、QP、CQ、MR 分别是什么？
**答：**`ibv_context` 是设备访问入口；PD 是资源保护域；QP 包含 SQ（发送队列）和 RQ（接收队列）；CQ 保存完成项；MR 是注册过的内存区域，带有访问权限、`lkey/rkey`。CQ **不属于 QP 内部**，只是被 QP 的发送/接收方向引用。

### Q09 ★ 一个 RC QP 是两个队列还是一条网络连接？
**答：**本地一个 QP 由 SQ + RQ 组成；一条 RC 通信关系通常是**本地 QP 与远端 QP 一对一**。不能把“两条本地 QP”当成一条 RC 连接，也不能把 CQ 当成 QP 的子队列。

### Q10 ★ 为什么使用 RC，而不是 UC/UD？
**答：**RC 提供可靠、有序的连接语义，适合需要明确完成结果的 Chunk 传输和 RPC 控制消息。UC/UD 的可靠性、消息边界和操作支持有所不同，对学习型 Demo 来说会增加额外设计成本。

### Q11 ★ `librdmacm` 和 `libibverbs` 分工是什么？
**答：**`librdmacm` 负责地址/路由解析、连接请求、接受连接等生命周期；`libibverbs` 负责 PD、CQ、QP、MR，以及提交 SEND/RECV/READ/WRITE 和获取 CQE。采用 RDMA CM 后不必自己再用 TCP 手工交换 QPN/PSN/GID 来建立 RC QP。

### Q12 ★ QPN、PSN、GID 各自是什么？
**答：**QPN 是 QP 编号；PSN 是 RC 包序列号，辅助可靠传输；GID 用于标识 RDMA 网络端点（RoCE 场景中与网络接口和地址相关）。如果手工连接，需要正确查询和交换；在本 Demo 中连接建立主要交给 RDMA CM。

### Q13 ★ `malloc()` 和 `ibv_reg_mr()` 有什么区别？
**答：**`malloc()` 只是申请进程内存；`ibv_reg_mr()` 把指定内存注册给 RDMA 设备，设置访问权限并返回 MR、`lkey`、`rkey`。`ibv_dereg_mr()` 不会自动 `free()`，因此 Demo 用简单 RAII 对象封装这两个步骤。

### Q14 ★ `lkey` 与 `rkey` 的区别是什么？
**答：**`lkey` 用于本地 WR 的 SGE，证明该 QP 能合法访问这块本地内存；`rkey` 连同远端地址提供给**对端**进行授权的 RDMA READ/WRITE。`rkey` 不是内存地址，也不该当成可永久使用的凭据。

### Q15 ★ RDMA SEND、READ、WRITE 最大差别是什么？
**答：**SEND/RECV 是双边消息操作，接收方要**提前 post RECV**；READ 是读取者从远端注册内存主动拉数据；WRITE 是写入者把本地数据直接写到远端注册内存。普通 READ/WRITE **不要求远端为这一次操作 post RECV**，但仍需要交换地址、rkey 和长度。

### Q16 ★ `ibv_post_send()` 返回 0，是数据传完了吗？
**答：**不是。返回 0 只表示 WR 成功提交。还需等对应 CQE，检查 `wc.status == IBV_WC_SUCCESS`；即使 RDMA 完成，也不能证明 Storage 业务完成或磁盘已经持久化。

### Q17 ★ 为什么 READ 和 WRITE 要携带远端地址和 rkey？
**答：**因为 one-sided 操作直接访问远端内存，没有普通 RPC 接收方帮你查找目标 buffer。网卡需要用 `remote_addr + rkey + length` 检查访问的范围与权限；远端地址不能拿到本地直接当指针解引用。

---

## 三、异步 RPC、CQ 和多请求关联（★ 高频追问）

**本节小结：**异步的关键不是某个线程名，而是**提交后返回、完成后关联、资源直到完成才释放**。

### Q18 ★ `RdmaConnection::CallMethod()` 为什么要保存 pending map？
**答：**异步时可能连续发出多个 RPC，响应返回顺序不一定与请求一致。`pending[request_id]` 保存响应对象、Controller 和 `done` 回调，收到响应后根据 `request_id` 找回正确的一次调用。

### Q19 ★ `request_id` 和 `wr_id` 有什么区别？
**答：**`request_id` 是**业务层一整次 RPC**的关联标识；`wr_id` 是**某个具体 RDMA WR**的本地完成标识。一次 RPC 可能包含 SEND、WRITE 等多个 WR，因此不应把两者简单视为同一个概念。

### Q20 ★ 什么是 CQE？由谁处理？
**答：**CQE（WC）是已完成 WR 的记录，包含 `wr_id`、`status`、`opcode` 等。Demo 用一个简单 CQ 完成处理线程接收事件并调用 `ibv_poll_cq()` 取出 WC，然后按类型触发逻辑，不需要 FastBlock 的多级 poller 框架。

### Q21 ★ Completion channel 与 CQ 有什么区别？
**答：**CQ **存放完成项**，completion channel **通知有完成事件**。`ibv_get_cq_event()` 不会替你取走 WC，仍需调用 `ibv_poll_cq()`；还要正确注册 CQ 通知并对事件做 `ibv_ack_cq_events()`。

### Q22 ★ 为什么必须预先 post RECV？
**答：**因为 SEND 到达时接收端要有可用的 RECV WR 和注册 buffer，否则可能遇到 RNR（Receiver Not Ready）及重试失败。收到一条控制消息后，需要在合适时机补充 RECV；处理消息前要注意接收缓冲区不能被过早覆盖。

### Q23 ★ `done->Run()` 什么时候调用？
**答：**在 RPC **真正完成**（得到响应或明确失败）后调用，而不是发出 SEND 的时候。服务端异步 Pull 还会先提交 RDMA READ，等 READ CQE、校验并写盘之后，才通过 `done` 生成最终响应。

### Q24 ★ 为什么不能在异步操作提交后立即 free buffer？
**答：**RDMA 网卡可能仍要读写那块内存。发送 buffer 通常要等对应本地 SEND/WRITE 完成才能释放；Pull 里 Client 暴露给 Store 的 MR 还必须活到 Store 的 READ 已结束。否则可能访问失效内存。

### Q25 ★ 高并发时 `pending` map 如何保证线程安全？
**答：**提交线程和 CQ 回调线程可能同时读写 map，可用一把 `std::mutex` 保护插入、查找、删除。拿到并移出待完成对象后先释放锁，再执行用户回调，避免回调重入 `RdmaConnection` 导致死锁。

### Q26 ★ 回调里能不能直接执行磁盘 `fsync`？
**答：**Demo 可以在低并发下先跑通，但高并发时会阻塞 CQ 处理，导致 CQE 积压。扩展时应让 CQ 线程只做完成分发，把可能阻塞的校验、写盘、`fsync` 交给存储模块已有的工作线程或简单任务队列；这不需要 SPDK。

---

## 四、高并发：资源组织与性能瓶颈（面试核心）

**本节小结：****一条连接一个 QP**是容易理解的起点，但真正的高并发取决于 QP/CQ/MR 的资源成本、队列深度、完成处理速度和业务背压。

### Q27 ★ 一张 RDMA 网卡能有多少个 QP？
**答：**通常一个设备 Context/PD 下可以创建多个 QP；具体上限不是固定常数，要看设备、固件、驱动、队列参数和资源限制。高并发设计一般避免为每个 Chunk 新建 QP，而是复用 Client 与 Store 之间的连接。

### Q28 ★ 为什么高并发情况下不建议“一请求一个 QP”？
**答：**每个 QP 都有设备资源和连接建立成本；请求越多，管理、状态切换和内存消耗越高。更合适的方式通常是**一条连接复用 QP，允许多个 WR in-flight**，通过完成项区分它们。

### Q29 ★ Context、PD、CQ 都应该每个连接一个吗？
**答：**不必。一个进程可复用 Context，一个 PD 可以关联多个 QP/MR，多个 QP 也能把完成项送到共享 CQ。Demo 可以每连接一组资源以简化实现；高并发再考虑共享资源减少开销。

### Q30 ★ 共享 CQ 与每连接独立 CQ，怎么选？
**答：**独立 CQ 代码直观、隔离好，但连接很多时 CQ 和线程数量可能过大；共享 CQ 降低数量，却需要根据 `wr_id/qp_num` 分发并注意竞争。Demo 采用简单方案，高并发优化时按工作线程分配 CQ 往往更有扩展性。

### Q31 ★ SRQ 是什么？什么时候需要？
**答：**SRQ（Shared Receive Queue）让多个 QP 共享 Receive WR 池，减少大量连接各自预留 RECV buffer 的开销。它主要帮助 SEND/RECV 场景，并**不会合并多个 RC QP 为一条连接**。Demo 不必实现。

### Q32 ★ QP 的 `max_send_wr`、`max_recv_wr` 为什么重要？
**答：**它们限制 SQ/RQ 能同时容纳多少未完成 WR。业务提交速度超过完成处理或接收补充速度时，就会遇到队列满、RNR 或错误。高并发要用 in-flight 限额和背压，不能无限 post WR。

### Q33 ★ 什么是背压（Backpressure）？
**答：**当空闲 Slot、RECV buffer、SQ 深度或磁盘处理能力不足时，系统必须限制新请求，而不是无限提交。Demo 可以简单返回 `BUSY` 或在**尚未开始远端数据操作时**回退 gRPC；生产系统才考虑等待队列、credits 等精细流控。

### Q34 ★ 为什么 MR 不应该高频注册/注销？
**答：**内存注册涉及驱动/设备维护映射和访问权限，通常比复用已注册 buffer 开销大。我的 Demo 为了看清流程，临时 Pull/SEND buffer 可以每次申请注册、完成后注销；若追求高 QPS，才考虑 MR pool、预注册等优化。

### Q35 ★ 高并发下你这个“每 Client 一个 SlotPool”有什么利弊？
**答：**优点是归属明确，Client 只操作自己的 Slot，避免不同 Client 争抢同一 Slot；并且写前可直接从本地 `RemoteSlotMap` 找空位。缺点是 Store 内存占用随 Client 数和 Slot 数近似线性增长，空闲 Client 也可能占着注册内存。Demo 先选简单隔离，生产环境再加配额和按需分配。

### Q36 ★ SlotPool 解决了所有并发竞争吗？
**答：**没有。它解决的是**跨 Client 共用 Slot 的竞争**；一个 Client 内仍可能有多个上传线程竞争自己的 Slot，本地 `RemoteSlotMap` 需要锁或原子状态。Store 也必须保证同一 Slot 还在写盘时不能被再次覆盖。

### Q37 ★ 高并发时吞吐上不去，最可能卡在哪里？
**答：**至少要分别检查：**网络带宽、QP/WR 深度、CQE 处理速度、MR 注册、内存复制、磁盘写入与 fsync**。RDMA 只是传输环节，若磁盘先达到瓶颈，换 RDMA 也不一定提升整体对象写入吞吐。

### Q38 ★ 你会测哪些性能指标？
**答：**分开测**网络传输**和**端到端对象写入**，分别看吞吐、CPU、P50/P99 延迟、在途 WR、CQE 处理速率、MR 注册次数、Slot 使用率及磁盘 fsync 时间。对比 gRPC 时保持相同 Chunk 大小、并发数、硬件和持久化语义，才有可比性。

### Q39 ★ 如果从 1 个 Client 扩展到 1000 个 Client，你会先改什么？
**答：**先处理**每连接线程/CQ 与 per-client SlotPool 的线性开销**，不优先改业务协议。可以改为有限的 CQ 工作线程与共享资源、控制 Slot 配额、增加真正的背压；但这些是高并发扩展计划，不是当前 Demo 已实现能力。

---

## 五、Pull/Push 与对象存储语义（★ 设计追问）

**本节小结：**Pull/Push 的最大区别是**谁暴露内存、谁主动发起 one-sided 操作**；存储成功最终必须经过原来的落盘路径。

### Q40 ★ Pull 模式怎么工作？为什么选择它？
**答：**Client 注册包含 Chunk 的 MR，把 `addr/rkey/length` 通过控制 RPC 告诉 Store；Store 发起 `IBV_WR_RDMA_READ`，读入本地 buffer 后交给 `ChunkStore`。它能让我直接理解远程读、MR 授权和异步 READ completion。

### Q41 ★ Push 模式怎么工作？为什么维护 RemoteSlotMap？
**答：**Store 为**每个 Client**单独创建 SlotPool 并提供 Slot 的地址、rkey 与容量；Client 只拉取自己的 Slot 列表，写入时从本地 Map 选空位并发起 RDMA WRITE。Map 目的是减少每个 Chunk 申请写入地址的控制面往返；Store 仍拥有真实内存和最终状态。

### Q42 ★ 为什么 Push 写完后还需要 `PushReady`？
**答：**普通 RDMA WRITE 不会像 SEND 一样在 Store 产生一条“应用层接收消息”。因此 Client 必须通过控制消息告诉 Store 哪个 Slot、哪个 Chunk、有效长度和 checksum 已准备好，Store 才知道何时可以校验和落盘。

### Q43 ★ 为什么 WRITE 和 READY 使用同一个 RC QP？
**答：**MVP 将控制 SEND 和数据 WRITE 都放在一个 RC QP，避免跨连接协调。为了更易理解和验证，Demo 可以**先收到本地 WRITE 成功 CQE，再提交 READY SEND**，这样明确先写数据、再通知 Store。不能把 READY 的成功当成磁盘持久化。

### Q44 ★ RDMA WRITE Completion、READY、Durable ACK 有何区别？
**答：**WRITE Completion 是 RDMA 数据操作完成；READY 是 Client 向 Store 发出的处理通知；Durable ACK 是 Store 按现有 ChunkStore 语义完成写入后返回的业务结果。只有最后一个才可以作为对象上传“该副本写成功”的依据。

### Q45 ★ 为什么不说这个实现是完全零拷贝？
**答：**RDMA 能减少网络传输链路上的 CPU 参与和部分复制，但数据通常先进入 DRAM，再由 CPU/文件系统写到磁盘。当前 Demo 可能还会临时复制到注册 buffer，因此只能说“采用 RDMA one-sided 搬运”，不能夸大为 RNIC 直接零拷贝写盘。

### Q46 ★ 一个 Client 拿到 Store 的 `addr/rkey` 后，可以一直写吗？
**答：**不行。它只对对应已注册 MR、连接/授权关系和有效生命周期有意义。Store 释放 Slot MR、重建 Pool 或重启后，旧 descriptor 必须失效，Client 要重新获得信息。Demo 可以简化重连，但不能继续使用已注销的 rkey。

### Q47 ★ 为什么暂时不做 MemoryPool，Push 却要长期持有 Slot MR？
**答：**通用 MemoryPool 是为了重复利用内存、降低注册成本，性能 Demo 可以暂时省略。但 Push Slot 的远端地址已经暴露给 Client，在 Client 仍使用它的期间 MR 必须保持注册，这属于**正确性要求**而不是性能优化。

---

## 六、故障、安全与回退（★ 容易被追问）

**本节小结：**最重要的原则是：**网络失败不一定表示远端没有成功写入**，所以 fallback 要按失败阶段判断。

### Q48 ★ RDMA 默认优先，失败就直接 gRPC 重试吗？
**答：**不能一概而论。如果连接建立失败、尚未开始任何远端数据操作，可以安全回退；但若 READ/WRITE 已经提交或 ACK 丢失，远端状态可能不确定，直接重传可能造成重复操作。Demo 对后一类先返回错误，不做复杂恢复。

### Q49 ★ `ibv_post_send()` 成功但后来收到错误 CQE，怎么处理？
**答：**先按 `wr_id` 找到对应 WR，检查 `wc.status` 并标记该操作失败。还要结合它属于连接前、数据传输中或等待 ACK 阶段来决定能否回退；不应把所有 CQE 错误都当成安全重试。

### Q50 ★ RNR 是什么？
**答：**Receiver Not Ready，典型原因是对方 SEND 到来时没有足够的 Receive WR。修正方式是预先 `post_recv` 并及时补充，或在高并发时考虑更大的 RECV 深度/SRQ；增加网络带宽不能解决 RNR 本质问题。

### Q51 ★ 为什么 remote access error 常与 rkey/MR 有关？
**答：**one-sided READ/WRITE 必须访问合法的远端注册区域，地址范围、rkey 和访问权限都需匹配。使用过期 rkey、越界地址，或者没给 `REMOTE_READ/REMOTE_WRITE` 权限都可能导致远端访问错误。

### Q52 ★ 为什么“一个 Client 一个 SlotPool”仍需要访问隔离？
**答：**Pool 归属只是软件逻辑；RDMA 实际上按注册内存、rkey、权限和连接资源执行访问检查。Demo 至少确保不会把 Client A 的 Slot descriptor 发给 B，不授予无关权限；生产环境还需要认证与更严格的授权策略。

### Q53 ★ CQ 满了会怎样？
**答：**如果应用处理 CQE 太慢，可能发生 CQ overrun，导致 CQ 进入错误状态。高并发下要保证 CQ 及时排空、CQ 容量合理并避免 completion 线程被写盘等长任务阻塞。

### Q54 ★ 断连后如何避免异步请求永远不结束？
**答：**即使 Demo 不做自动重连，也应在连接明确失效时把 pending RPC 标记失败并调用对应回调，避免业务一直等。注销 MR 前还要确保相关 WR 不再访问它；已经无法判断是否写到远端的操作不要盲目 gRPC 重试。

---

## 七、面试官可能继续追问的“设计取舍”

**本节小结：**回答时优先说**为什么现在简化、将来在什么压力下才扩展**，不要把 Demo 包装成工业级系统。

### Q55 ★ 为什么现在不用共享 CQ？
**答：**Demo 主要验证异步操作和业务路径，每连接独立 CQ 容易理解与调试。高并发大量连接时每连接 CQ/线程成本可能太高，再考虑按工作线程共享 CQ，并用 `wr_id/qp_num` 分发完成项。

### Q56 ★ 为什么现在不用 MR Pool？
**答：**当前目的是观察 `malloc → ibv_reg_mr → post WR → completion → ibv_dereg_mr → free` 的完整生命周期。这个路径在高并发下可能很慢，但能帮助理解正确性；若性能测试发现注册成本明显，再引入预注册 Pool。

### Q57 ★ 为什么现在不做 SPDK 或复杂 poller？
**答：**项目原有存储栈不使用 SPDK，也没有必要为了 RDMA 改造整个线程模型。简单 `std::thread` + CQ completion channel 已经能展示异步；后续需要高性能时可以优化 CQ 分发，但不必依赖 SPDK。

### Q58 ★ 如果要支持真正的大并发，你觉得第一步是什么？
**答：**先量化瓶颈，而不是立刻增加复杂抽象。若瓶颈是注册内存，做 MR 复用；是 CQ 线程，分离阻塞业务工作并合理共享 CQ；是 Slot 内存，限制 per-client 配额；是磁盘，就优化 ChunkStore 端的 IO 并发。

### Q59 ★ 你的 Demo 最值得强调的技术点是什么？
**答：**我认为是三个：① 通过 RpcChannel 把 Protobuf RPC 和传输解耦；② 通过 `request_id + CQE + callback` 实现真正异步；③ 同时跑通 Client Pull 和 per-client Slot Push，并保持原 ChunkStore 的 durability/fallback 边界。

---

## 八、最后速背：12 个一句话答案

1. **QP** = SQ + RQ；**CQ** = 完成项所在队列，不在 QP 里面。
2. **PD** 管资源保护关系；**MR** 是注册内存；**Context** 是本地设备入口。
3. **RC QP** 通常与对端 RC QP 一对一；一个设备可以有多个 QP。
4. **SEND** 要求远端提前 post RECV；普通 **READ/WRITE** 不需要。
5. **`ibv_post_send()==0`** 只表示成功提交，不能当成完成。
6. **`wc.status==IBV_WC_SUCCESS`** 表示对应 WR 完成，**不表示写盘完成**。
7. **`wr_id`** 对应 WR；**`request_id`** 对应整次 RPC。
8. **异步 RPC** = 提交即返回；CQE 到来后找 pending 并回调。
9. **Pull** = Client 暴露 MR，Store 发 RDMA READ。
10. **Push** = Store 为每个 Client 分 SlotPool，Client 发 RDMA WRITE，再 SEND READY。
11. **Fallback** = 数据尚未开始时可回退；远端状态不确定时不能盲目重试。
12. **高并发瓶颈** = QP/CQ/WR 深度、MR 注册、Slot 容量、CPU、网卡与磁盘共同决定。

---

## 资料与适用性说明

**直接来自你的 RDMA 笔记的基础知识**：Context/PD/QP/CQ/MR、RC QP、QPN/PSN/GID、`ibv_reg_mr` 与 `malloc`、READ/WRITE/SEND 差异、CQE 语义、共享 CQ/SRQ 等。笔记中有**手工交换 QPN/PSN/GID**的示例；**本 Demo 采用 `librdmacm` 建连**，不要把两套连接代码混写。

**来自我们讨论的 Demo 方案**：`modules/rdma/`、`RdmaConnection : RpcChannel`、简单异步 CQ 线程、Pull/Push、Store 为每个 Client 独立分 SlotPool、RDMA 优先 + 安全回退，以及存储仍走原 `ChunkStore::WriteChunk()`。这是**架构目标**，并非已验证的代码事实。

**为回答“高并发”面试题而增加的延伸知识**：共享 CQ、SRQ、背压、请求/线程/内存规模、P99、按瓶颈扩展资源等，不要求当前 Demo 全部实现。

参考资料：

- [rdma-core / libibverbs 与 librdmacm](https://github.com/linux-rdma/rdma-core)
- [`ibv_poll_cq(3)`：CQE 字段及 CQ overrun](https://man7.org/linux/man-pages/man3/ibv_poll_cq.3.html)
- [`ibv_get_cq_event(3)`：完成事件、ACK、重新注册通知](https://man7.org/linux/man-pages/man3/ibv_get_cq_event.3.html)
- [`ibv_post_send(3)`：WR 提交、SGE、远端地址与 rkey](https://man7.org/linux/man-pages/man3/ibv_post_send.3.html)
- [`ibv_reg_mr(3)`：MR 注册和访问权限](https://man7.org/linux/man-pages/man3/ibv_reg_mr.3.html)
- [Protobuf `RpcChannel` / Generic Services 官方说明](https://protobuf.dev/reference/cpp/api-docs/google.protobuf.service/)
- [Protobuf 2026 年 Generic Services 生成兼容性公告](https://protobuf.dev/news/2026-09-22/)

