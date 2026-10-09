# RDMA

> **适用范围**：主要讨论 Linux `rdma-core` / `libibverbs`、RoCEv2 和 **RC（Reliable Connection）QP**。其他传输类型（UD、UC 等）的状态转换和通信语义并不完全相同。
>
> **示例环境**：主机 A `192.168.10.11`（提供数据），主机 B `192.168.10.12`（发起 RDMA Read）。示例 QPN、PSN、GID Index 仅用于解释，实际必须查询并交换，不能硬编码。

# 一、过程关键函数

下面按“**初始化 → 建立 QP 连接 → 注册内存 → 远程读取 → 回收资源**”组织。实际程序可以在 QP 状态转换前就分配并注册内存；此处是便于学习的一种顺序。

| 阶段 | 函数 / 操作 | 作用 |
| --- | --- | --- |
| 设备初始化 | `ibv_get_device_list()` | 枚举本机 RDMA 设备 |
| 设备初始化 | `ibv_open_device()` | 打开 RDMA 设备，获得 `ibv_context` |
| 设备初始化 | `ibv_alloc_pd()` | 在指定 Context 上创建保护域 PD |
| 设备初始化 | `ibv_create_cq()` | 创建完成队列 CQ |
| 设备初始化 | `ibv_create_qp()` | 创建 QP，普通 RC QP 初始处于 RESET |
| 查询与交换 | `ibv_query_port()` / `ibv_query_gid()` | 查询端口状态、MTU、选定 GID 等本地信息 |
| 查询与交换 | TCP / 其他控制通道 | **交换 QPN、初始 PSN、GID 等 QP 连接信息** |
| QP 连接 | `ibv_modify_qp()` | 分别执行 `RESET → INIT → RTR → RTS`，配置 RC 连接 |
| 内存准备 | `malloc()` / `memset()` 等 | 分配和初始化普通用户内存；**不是注册 MR** |
| 内存准备 | `ibv_reg_mr()` | 把内存注册到 PD，获得 LKey/RKey |
| 内存信息交换 | TCP / RDMA Send/Recv | **A 将 `addr + rkey + length` 告诉 B**（用于 RDMA Read） |
| 发起远程读取 | `ibv_post_send()` | B 向自己的 QP 的 SQ 提交 `IBV_WR_RDMA_READ` |
| 查询完成 | `ibv_poll_cq()` | B 获取完成项，并检查 `wc.status` |
| 资源回收 | `ibv_dereg_mr()` / `free()` | 先注销 MR，再释放普通内存；必须确保操作已结束 |
| 资源回收 | `ibv_destroy_qp()`、`ibv_destroy_cq()`、`ibv_dealloc_pd()`、`ibv_close_device()` | 清理 QP、CQ、PD、Context 等资源 |

**ASCII 流程图（紧跟关键函数表；A 提供数据，B 主动读）：**

```text
       Host A: 192.168.10.11             Host B: 192.168.10.12
       ---------------------             ---------------------
       ibv_open_device()                 ibv_open_device()
                |                                 |
       ibv_alloc_pd()                    ibv_alloc_pd()
                |                                 |
       ibv_create_cq()                   ibv_create_cq()
                |                                 |
       ibv_create_qp()                   ibv_create_qp()
          QP = RESET                        QP = RESET
                |                                 |
       query local QPN/PSN/GID           query local QPN/PSN/GID
                |                                 |
                +--- exchange QPN/PSN/GID ------->+
                +<-- exchange QPN/PSN/GID --------+
                |                                 |
       modify_qp(INIT)                    modify_qp(INIT)
                |                                 |
       modify_qp(RTR)                     modify_qp(RTR)
                |                                 |
       modify_qp(RTS)                     modify_qp(RTS)
                |                                 |
                +<--- peer-ready sync ----------->+
                |                                 |
       malloc(source_buffer)             malloc(dest_buffer)
                |                                 |
       fill data                         ibv_reg_mr(LOCAL_WRITE)
                |                                 |
       ibv_reg_mr(REMOTE_READ)                    |
                |                                 |
                +--- addr + rkey + length ------->+
                |                                 |
                |                       ibv_post_send(RDMA_READ)
                |                                 |
       A's RNIC handles Read <--------------------+
                |                                 |
       A's memory -- data via RNIC --------------> B's memory
                                                  |
                                         ibv_poll_cq()
                                         check wc.status
                                                  |
                                         use dest_buffer
                |                                 |
       stop access / synchronize both endpoints
                |                                 |
       ibv_dereg_mr()                    ibv_dereg_mr()
       free(source_buffer)               free(dest_buffer)
                |                                 |
       destroy QP/CQ/PD/context          destroy QP/CQ/PD/context
```

> **两个不同的“交换”不要混淆：**
> 1. **连接交换：** QPN、PSN、GID，用于配置 RC QP；不要求传 RKey。
> 2. **内存交换：** 远程地址、RKey、长度，用于执行 RDMA Read/Write；仅知道对端 IP 或 QPN 不足以访问其内存。
>
# 二、资源关系：Context、PD、QP、CQ、MR

## 2.1 各对象分别是什么

| 对象 | 含义 | 关键作用 |
| --- | --- | --- |
| **RDMA Device** | 本地 RDMA 设备（例如 `mlx5_0`、`rxe0`） | 提供硬件或软件 RDMA 能力；`mlx5_0` **不是所有设备的默认名** |
| **Context** (`ibv_context`) | 进程打开某个 RDMA 设备后获得的设备上下文 | 作为访问本地 RDMA 设备和创建资源的入口；**不是网络连接** |
| **PD** (Protection Domain) | 资源保护域 | QP、MR（以及 SRQ 等）在创建时关联 PD；限制不兼容资源的配合使用；**不是内存地址或内存分配器** |
| **QP** (Queue Pair) | 发送队列 **SQ** + 接收队列 **RQ** | 应用程序向 SQ 提交 Send / Read / Write WR；向 RQ 提交 Receive WR（若使用 SRQ 则接收队列可共享） |
| **CQ** (Completion Queue) | 完成队列 | 收集完成项 WC；**CQ 不在 QP 内部**，而是在创建 QP 时通过 `send_cq`、`recv_cq` 关联 |
| **MR** (Memory Region) | 已注册内存区域 | 描述注册内存的范围、权限，并提供 LKey/RKey |
| **SRQ** (Shared Receive Queue) | 共享接收队列（可选） | 多个 QP 可以共享接收 WR 池，适合大量 Send/Recv 连接 |

Context 不等于“整张网卡只能创建一个”：多个进程可分别打开同一个设备得到多个 Context；一个进程通常也只需一个 Context 即可管理大量连接。Context 主要承载设备访问、provider/驱动操作接口及相关资源上下文。

PD 也不是跨机器共享的“保护域编号”：主机 A 和主机 B 各有本地 PD，不要求彼此 PD 相同。

## 2.2 一对多 / 多对一关系

| 关系 | 设计约束 / 常见情况 |
| --- | --- |
| 一个 RDMA 设备 → Context | **一对多**，可有多个 Context；没有所有设备通用的固定最大数量 |
| 一个 Context → PD | **一对多**，可创建多个 PD |
| 一个 PD → QP | **一对多**，大量 QP 可绑定同一个 PD |
| 一个 PD → MR | **一对多**，多个 MR 可注册在一个 PD 中 |
| 一个 QP → PD | **只绑定一个 PD**（普通 `ibv_create_qp(pd, ...)`）；QP 不能同时属于两个 PD |
| 一个 QP ↔ MR | **多对多的使用关系**，但本地 QP 使用的 MR 必须满足 PD/权限约束；MR 并非“挂在某个 QP 里” |
| 一个 CQ → QP | **一对多**，多个 QP 可以将完成项汇入同一个 CQ |
| 一个 QP → CQ | SQ 关联一个 `send_cq`，RQ 关联一个 `recv_cq`；**两者可以指向同一个 CQ** |
| 一个 RC QP ↔ 对端 RC QP | **通常一对一连接**，两台主机各创建一个 QP；不是两个本地 QP 组成一条 RC 连接 |

**ASCII 资源关系图：**

```text
RDMA Device (mlx5_0)
  |
  +-- ibv_context (one process)
        |
        +-- PD_1
        |    +-- QP_1  -----+ 
        |    +-- QP_2  -----+----> CQ_1 (shared completion queue)
        |    +-- QP_3  -----+ 
        |    +-- MR_log    (lkey, rkey)
        |    +-- MR_ctrl   (lkey, rkey)
        |
        +-- PD_2
             +-- QP_4  ---------> CQ_2
             +-- MR_kv

Note: CQs are created from context, not owned by PD_1 or PD_2.
      Each QP binds to one PD, but several QPs can use one CQ.
```

创建一个 QP 并将它的发送/接收完成项关联到同一个 CQ：

```c
struct ibv_qp_init_attr attr = {0};
attr.qp_type = IBV_QPT_RC;
attr.send_cq = cq;      // SQ 的完成项放入 cq
attr.recv_cq = cq;      // RQ 的完成项也放入 cq
attr.cap.max_send_wr = 128;
attr.cap.max_recv_wr = 128;
attr.cap.max_send_sge = 1;
attr.cap.max_recv_sge = 1;

struct ibv_qp *qp = ibv_create_qp(pd, &attr);
// 实际程序必须检查 qp != NULL。
```

**纠正容易写错的点：**

- QP 包含的是 **SQ + RQ**，不是 “CQ + RQ”；`RQ` 是 *Receive Queue*（接收队列），不是“回复队列”。
- CQ 和 QP 的对应关系在创建 QP 时由指针指定，**不通过 QP 编号进行按位或运算来选择 CQ**。
- 应用程序轮询共享 CQ 时，可使用 `wc.wr_id`、`wc.qp_num`、`wc.opcode` 等识别完成事件；要区分 **请求提交成功** 与 **请求执行完成**。

# 三、RC QP 连接：RESET → INIT → RTR → RTS

QP 创建后通常处于 RESET；进入 RTS 才允许本地 SQ 正常主动发出 RDMA 请求。**这四个状态是配置通信能力的过程，不是“已经发生四次数据传输”。**

| 状态 | 全称 | 说明 |
| --- | --- | --- |
| **RESET** | Reset | QP 创建后的初始状态，尚未配置通信参数 |
| **INIT** | Initialized | 本地端口、P_Key、QP 访问权限等已配置；可以提交 Receive WR，但尚不能正常收发业务包 |
| **RTR** | Ready To Receive | 已设置对端 QPN、接收 PSN、路径信息等；具备处理对端数据包的能力 |
| **RTS** | Ready To Send | 已设置本地发送 PSN、超时与重试等；本地程序可以通过 SQ 提交 Send / Read / Write 请求 |

> 对于 RC：**进入 RTR 并不意味着必须调用 `ibv_post_recv()` 才能响应远程 RDMA Read/Write**。普通 RDMA Read/Write 不使用远端的 Receive WR；但 RDMA Send 需要接收端预先提交 Receive WR（或使用 SRQ）。

## 3.1 A、B 各用自己的还是对方的参数？

假设 A 的 QPN=100、PSN=1000、GID=GID_A、本地 GID Index=3；B 的 QPN=200、PSN=5000、GID=GID_B、本地 GID Index=4。**均为假设值。**

| 状态转换 | 属性 | A 填什么 | B 填什么 |
| --- | --- | --- | --- |
| RESET → INIT | `port_num`、`pkey_index`、`qp_access_flags` | **A 自己的**本地配置 | **B 自己的**本地配置 |
| INIT → RTR | `dest_qp_num` | **B 的 QPN=200** | **A 的 QPN=100** |
| INIT → RTR | `rq_psn` | **B 的发送 PSN=5000** | **A 的发送 PSN=1000** |
| INIT → RTR | `ah_attr.grh.dgid` | **B 的 GID_B** | **A 的 GID_A** |
| INIT → RTR | `ah_attr.grh.sgid_index` | **A 自己的索引=3** | **B 自己的索引=4** |
| RTR → RTS | `sq_psn` | **A 自己的 PSN=1000** | **B 自己的 PSN=5000** |

重要对应关系：

```text
A.sq_psn == B.rq_psn       # A 发，B 按此 PSN 接收
B.sq_psn == A.rq_psn       # B 发，A 按此 PSN 接收
```

自己的 QPN 在 `ibv_create_qp()` 时由设备分配，通过 `qp->qp_num` 获取；**不是在 RTR/RTS 时再给自己配置一个 QPN**。

## 3.2 三次转换时最重要的字段

```c
// RESET -> INIT（只涉及本地初始化）
attr.qp_state = IBV_QPS_INIT;
attr.port_num = 1;
attr.pkey_index = 0;
attr.qp_access_flags = IBV_ACCESS_REMOTE_READ |
                       IBV_ACCESS_REMOTE_WRITE;

// INIT -> RTR（关键是配置对端及接收路径）
attr.qp_state = IBV_QPS_RTR;
attr.dest_qp_num = remote_qpn;       // 对方的 QPN
attr.rq_psn = remote_psn;            // 对方的 SQ 起始 PSN
attr.ah_attr.grh.dgid = remote_gid;  // 对方的 GID
attr.ah_attr.grh.sgid_index = my_gid_index; // 自己的索引

// RTR -> RTS（关键是配置本地 SQ）
attr.qp_state = IBV_QPS_RTS;
attr.sq_psn = my_psn;                // 自己的发送 PSN
```

上面仅摘录核心字段，**不能当作可直接运行的完整 `ibv_modify_qp()` 调用**。RC 状态转换还需要 `path_mtu`、`max_dest_rd_atomic`、`min_rnr_timer`、`max_rd_atomic`、`timeout`、`retry_cnt`、`rnr_retry` 以及各阶段正确的 `attr_mask`；用 `ibv_modify_qp(qp, &attr, mask)` 执行，每次检查返回值。

完成状态转换后，A、B 可以通过 TCP 互发自己定义的 `QP_READY` 控制消息做同步；该消息**不是 RDMA 标准规定的强制握手**。`ibv_modify_qp()` 返回 0 表示**本地配置成功**，并不自动证明远端业务已准备好。

## 3.3 QPN、PSN、GID 的作用

- **QPN（Queue Pair Number）**：QP 编号，用于标识 RDMA 设备上的某个 QP。IP/GID 参与定位网络端点，QPN 用于确定目标 QP；QPN **不是 TCP 端口**。
- **PSN（Packet Sequence Number）**：RC 传输中使用的包序列号，类似 TCP 的 *sequence number*，**不是 ACK**。可靠传输还依靠 ACK/NAK、超时、重试等机制；不是“一失败就无限立即重传”。PSN 按数据包序列推进，和 TCP 按字节编号也不同。
- **GID（Global Identifier）**：128 位标识。在 RoCEv2 中与 IP 地址和网络接口相关，路径配置时 `dgid` 指对端 GID，`sgid_index` 指本地设备端口的 GID 表索引。

### 为什么一个端口有多个 GID Index？

常见原因是一个设备端口关联多个 IP / VLAN / IPv6 地址，或者同一 GID 值存在 RoCEv1、RoCEv2 等不同类型的条目。

**选择原则：** 主机 A 选自己 `192.168.10.11` 对应的 RoCEv2 GID 表项，B 选自己 `192.168.10.12` 对应的 RoCEv2 GID 表项；两台机器的 **GID Index 可以不同**。不要默认 index 一定等于 3，也不要默认设备名一定是 `mlx5_0`。

```bash
ibv_devices
ibv_devinfo
rdma link show

# 把设备名、端口号和 index 换成实际查询到的值
cat /sys/class/infiniband/mlx5_0/ports/1/gids/3
cat /sys/class/infiniband/mlx5_0/ports/1/gid_attrs/types/3
cat /sys/class/infiniband/mlx5_0/ports/1/gid_attrs/ndevs/3
```

# 四、MR、LKey/RKey，以及 RDMA Read / Write / Send

## 4.1 `malloc` 和 `ibv_reg_mr` 不是同一件事

```c
// A：分配内存、准备数据、允许 B 远程读取
char *source = malloc(1024);
// 检查 source != NULL，并在注册前写入业务数据
struct ibv_mr *source_mr = ibv_reg_mr(
    pd_a, source, 1024, IBV_ACCESS_REMOTE_READ);

// B：分配接收数据的本地内存，允许本地 RNIC 写入
char *dest = malloc(1024);
struct ibv_mr *dest_mr = ibv_reg_mr(
    pd_b, dest, 1024, IBV_ACCESS_LOCAL_WRITE);
```

- `malloc()` 申请的是进程内存；`ibv_reg_mr()` **注册已有内存**，不替代 `malloc()`。
- 普通 MR 注册使驱动/网卡建立访问和保护信息，通常涉及固定内存页；特殊 MR（例如 ODP）机制可能不同。
- **LKey**：本机 QP 的 SGE 使用，例如 `sge.lkey = dest_mr->lkey`。
- **RKey**：远端节点执行 RDMA Read/Write/Atomic 时提供，结合远程地址、权限、范围及连接保护进行检查；它不是地址，也不是加密密钥。
- A 若授权远端 **Remote Read**，设置 `IBV_ACCESS_REMOTE_READ`；B 的目标缓冲区需要 `IBV_ACCESS_LOCAL_WRITE`。
- 若某 MR 设置了 `IBV_ACCESS_REMOTE_WRITE` 或 `IBV_ACCESS_REMOTE_ATOMIC`，还必须设置 `IBV_ACCESS_LOCAL_WRITE`。

## 4.2 B 如何主动读取 A 的内存？

A 通过可信的控制通道告诉 B：

```text
remote_addr  = A's MR address
remote_rkey  = A's MR rkey
remote_len   = requested length within registered MR
```

B 提交 RDMA Read（以下展示核心字段）：

```c
struct ibv_sge sge = {0};
sge.addr = (uintptr_t)dest;
sge.length = 1024;
sge.lkey = dest_mr->lkey;          // B 自己的 LKey

struct ibv_send_wr wr = {0};
struct ibv_send_wr *bad_wr = NULL;
wr.wr_id = 1;
wr.opcode = IBV_WR_RDMA_READ;
wr.send_flags = IBV_SEND_SIGNALED;
wr.sg_list = &sge;
wr.num_sge = 1;
wr.wr.rdma.remote_addr = remote_addr; // A 的地址
wr.wr.rdma.rkey = remote_rkey;        // A 的 RKey

int rc = ibv_post_send(qp_b, &wr, &bad_wr);
if (rc != 0) { /* WR 提交失败：处理错误 */ }
```

B 提交后，**B 的 RNIC 发起请求，A 的 RNIC 在硬件层面响应并读取 A 的内存**，数据最终 DMA 到 B 的 `dest`。A 不需要为每次 RDMA Read 调用一次普通的 `recv()` 或 `memcpy()`。

**但 `ibv_post_send()` 返回 0 仅表示 WR 提交成功，不代表数据已经到达。** B 要继续轮询 CQ：

```c
struct ibv_wc wc;
int n = ibv_poll_cq(cq_b, 1, &wc);
if (n > 0 && wc.wr_id == 1 && wc.status == IBV_WC_SUCCESS) {
    // 该 RDMA Read 成功完成，此时可以使用 dest 数据
}
// n==0: 暂时没有完成项；n<0: CQ 轮询失败。
// 真实程序应持续轮询/等待、检查所有完成项并处理超时。
```

A 必须保证 B 读取期间其 MR 有效、地址范围合法、数据已准备好；若并发修改同一片数据，还需应用层同步。

## 4.3 Read、Write、Send 的区别

| 操作 | 谁主动提交 `ibv_post_send()` | 对端需要 `ibv_post_recv()` 吗？ | 需要从对端拿什么？ |
| --- | --- | --- | --- |
| **RDMA Read** | 读取者（本例 B） | 不需要 | A 的 `addr + rkey`，以及合法长度 |
| **RDMA Write** | 写入者（例如 A） | 普通 Write 不需要 | B 的 `addr + rkey`，以及合法长度 |
| **RDMA Send** | 发送者（例如 A） | **需要**：B 预先提交 Receive WR（或 SRQ 接收 WR） | 不需要对端接收缓冲区的 RKey |

`"Hello RDMA"` 只是 Demo 的测试数据，不是 RDMA 标准定义的握手。项目中可以发送自定义的 `QP_READY` 控制消息、Raft AppendEntries 消息或日志数据；**数据传输成功不等于业务已经处理或持久化成功**。

## 4.4 完成与返回值

| 调用/检查 | 成功的含义 |
| --- | --- |
| `ibv_modify_qp(...) == 0` | **本地** QP 属性/状态转换成功 |
| `ibv_post_send(...) == 0` | WR 已成功提交到 SQ |
| `ibv_poll_cq(...) > 0` | 已取出一个或多个完成项，**还要检查 WC 状态** |
| `wc.status == IBV_WC_SUCCESS` | 对应 WR 成功完成（需要匹配 `wr_id` 等） |

使用 `IBV_SEND_SIGNALED` 或配置 SQ 全部发信号时，成功操作才会在发送 CQ 中产生可轮询的完成项；不发信号的成功 WR 通常不单独产生成功 CQE。对 RC Send 而言，发送方的 WC 成功也**不代表接收方应用已经处理消息**，更不代表 Raft 日志已落盘。

# 五、高并发场景的资源设计

## 5.1 一张网卡上，Context / PD / QP / CQ 怎么分配？

假设一台服务器要维护 **10,000 条 RC 连接**，一种可选的资源组织方式是：

```text
NIC: mlx5_0
  |
  +-- Process: Raft / Storage Server
        |
        +-- Context x 1
              |
              +-- PD x 1
              |    +-- QP x 10000    (typically one RC QP per peer connection)
              |    +-- MR x several (registered buffer pools)
              |
              +-- CQ x 8          (e.g. one CQ per polling worker)
```

**这里的 10,000 只是架构假设，不是所有设备都能达到的上限。** 真正能创建多少 QP/CQ/MR/PD，要看设备规格、固件、资源占用和实际 `ibv_query_device()` 查询结果；队列深度、未完成 Read 数、CPU 轮询能力也影响并发。

- **Context**：多个独立进程使用同一网卡时，通常每个进程有自己的 Context；**单个进程不必为了每条连接再打开一个 Context**。
- **PD**：一个可信服务通常可以先用一个 PD。只有需要隔离不同资源集合、业务或租户时，再考虑多个 PD。**PD 数量不是吞吐量开关**。
- **QP**：普通 RC 设计中每条远端连接通常使用独立 QP；一个 QP 能同时排队多个 WR，但受队列深度及 RDMA Read/Atomic 资源限制。
- **CQ**：多个 QP 可以绑定同一个 CQ；高并发时常按工作线程将 QP 分组，分配给不同 CQ，避免 CQ 轮询竞争和溢出。
- **MR**：可以注册少量大的缓冲池，由同一 PD 下多个 QP 复用，避免每次请求重复注册/注销内存。
- **SRQ（可选）**：若大量 QP 主要用 Send/Recv，多个 QP 可以共享接收 WR 池；**SRQ 不意味着合并这些 QP 的 RC 连接**。

### 按工作线程分 CQ 的示例

```c
// 假设已经有同一个 pd，以及从同一个 ctx 创建的 cq0 / cq1
struct ibv_qp_init_attr attr = {0};
attr.qp_type = IBV_QPT_RC;
attr.cap.max_send_wr = 128;
attr.cap.max_recv_wr = 128;
attr.cap.max_send_sge = 1;
attr.cap.max_recv_sge = 1;

attr.send_cq = cq0;
attr.recv_cq = cq0;
struct ibv_qp *qp_for_worker0 = ibv_create_qp(pd, &attr);

attr.send_cq = cq1;
attr.recv_cq = cq1;
struct ibv_qp *qp_for_worker1 = ibv_create_qp(pd, &attr);
```

这两个 QP 属于**同一个 PD**，但它们的完成项进入**不同 CQ**。CQ 是通过指针直接指定的，**不是“用多个 QPN 做按位或，得到 CQ 编号”**。

## 5.2 什么时候需要多个 PD？

例如同一个设备 Context 下有两套需隔离的业务：

```text
Context
  +-- PD_Raft
  |     +-- QP_Raft_1, QP_Raft_2, ...
  |     +-- MR_Raft_Log
  |
  +-- PD_KV
        +-- QP_KV_1, QP_KV_2, ...
        +-- MR_KV_Data
```

每个普通 QP **只绑定一个 PD**；同一 PD 可以有大量 QP 和 MR。QP 不能把另一 PD 下 MR 的 LKey 直接拿来用作本地 SGE。PD 不是完整的跨机器鉴权系统，远程访问还必须正确控制 RKey、地址、权限和连接。



# 六、生命周期与常见错误

1. `malloc/free` 管理进程内存；`ibv_reg_mr/ibv_dereg_mr` 管理 RDMA 内存注册。成功 `ibv_dereg_mr()` **不会自动 `free()`**。
2. 注销 MR 前，应确保本地相关 WR 已完成或被安全终止，并协调对端不再使用旧 `addr + rkey`；MR 注销后旧 RKey 不应继续使用。
3. 不要把远端虚拟地址当成本地指针直接解引用；远端地址要放进 RDMA WR 中，由 RNIC 执行访问。
4. `QP=RTS` 只表示**可以主动提交**，不会自动发送一个 `"Hello RDMA"` 包；仍需应用程序调用 `ibv_post_send()`。
5. `ibv_post_send()==0` **不等于传输成功**；使用 CQ 检查与 WR 对应的完成项，处理失败、超时及重试上限。
6. 共享 CQ 时注意 CQ 容量和及时轮询；CQ overrun 可使 CQ 进入不可用的错误状态。
7. `mlx5_0`、GID Index、QPN、PSN、MTU 都不应当被默认写死。设备需枚举，路径参数需查询与交换。
8. 代码中的“位或”通常出现在 **访问权限、`attr_mask` 等 flags 的组合**，不是 CQ 分配或 QP 路由方法。

# 七、官方参考资料

- [rdma-core 项目（包含 libibverbs、librdmacm 等）](https://github.com/linux-rdma/rdma-core)
- [`ibv_create_qp(3)`：QP 与 PD、send_cq、recv_cq 的关联](https://man7.org/linux/man-pages/man3/ibv_create_qp.3.html)
- [`ibv_modify_qp(3)`：RC QP 状态转换所需字段](https://manpages.debian.org/trixie/libibverbs-dev/ibv_modify_qp.3.en.html)
- [`ibv_reg_mr(3)`：注册内存、访问权限、LKey/RKey](https://man7.org/linux/man-pages/man3/ibv_reg_mr.3.html)
- [`ibv_post_send(3)`：提交 WR 的语义](https://man7.org/linux/man-pages/man3/ibv_post_send.3.html)
- [`ibv_poll_cq(3)`：完成队列与完成状态](https://man7.org/linux/man-pages/man3/ibv_poll_cq.3.html)
- [NVIDIA RoCE 文档：GID 表、类型与 IP/VLAN 关系](https://networking-docs.nvidia.com/mlnxenswum/23070512/rdma-over-converged-ethernet-roce)
