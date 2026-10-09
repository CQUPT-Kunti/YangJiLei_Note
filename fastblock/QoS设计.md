
状态：允许，IOPS不够，带宽不够，


成熟存储 QoS 的典型设计不是“IOPS 通过后拆 Object，再逐 Object 扣带宽”，而是：**一个 Logical IO 同时经过多个 limiter；任何一个 limiter 不允许，就把整个 IO 排队；所有 limiter 都允许才进入后端。大 IO 用欠账/时间片机制处理，而不是等完整带宽额度攒够。**
时间片最多10ms。


volume接口设置，

struct volume_qos {
    uint64_t iops_limit;      // 正常 IOPS 上限
    uint64_t bw_limit;        // 正常带宽上限，bytes/s

    uint64_t burst_iops;      // 允许短时突发的 IOPS
    uint64_t burst_bw;        // 允许短时突发的带宽
    uint32_t burst_seconds;   // burst 最多持续多久

    uint32_t timeslice_us;    // QoS 调度周期，比如 5ms

    double iops_tokens;       // 当前剩余 IOPS 额度
    double bw_tokens;         // 当前剩余带宽额度

    uint64_t last_refill_us;  // 上次补额度的时间

    std::deque<spdk_bdev_io *> pending; // 暂时不能发的 Logical IO
};


```
WRITE:
    iops_cost = 1;
    bw_cost   = io_size;

READ:
    iops_cost = 1;
    bw_cost   = io_size;

DELETE:
    iops_cost = 1;
    bw_cost   = 0;
```



# FastBlock Volume QoS 设计交接说明

## 1. 目标

在 FastBlock 的 **Volume / SPDK bdev Logical IO 入口**增加 QoS。

```text
Logical IO
    ↓
Volume QoS
    ↓
允许放行
    ↓
FastBlock 原 READ / WRITE 路径
    ↓
Object split
    ↓
Client / RPC / PG / OSD
```

QoS 只控制 Logical IO 什么时候进入原数据路径。

QoS 不负责：

```text
Object retry
RPC retry
Raft
错误处理
IO completion
parent/child Object 生命周期
```

---

## 2. QoS 控制项

每个 Volume 支持：

```text
IOPS Limit
Bandwidth Limit

Burst IOPS
Burst Bandwidth
Burst Duration

Timeslice

IOPS Token
Bandwidth Token

Pending Logical IO Queue
```

IO Cost：

```text
READ:
    IOPS cost = 1
    BW cost   = IO bytes

WRITE:
    IOPS cost = 1
    BW cost   = IO bytes

DELETE / DISCARD:
    IOPS cost = 1
    BW cost   = 0
```

---

## 3. Volume QoS 结构体

建议：

```text
src/bdev/volume_qos.h
```

```cpp
struct volume_qos {
    // -------------------------
    // Configuration
    // -------------------------

    // 0 = unlimited
    uint64_t iops_limit = 0;       // IO/s
    uint64_t bw_limit = 0;         // bytes/s

    uint64_t burst_iops = 0;       // burst IO/s
    uint64_t burst_bw = 0;         // burst bytes/s
    uint32_t burst_seconds = 1;

    // QoS 调度粒度
    uint32_t timeslice_us = 5000;  // 5ms


    // -------------------------
    // Runtime Token
    // -------------------------

    // 允许为负数，负数表示已经透支未来额度
    double iops_tokens = 0;
    double bw_tokens = 0;

    // 上一次 refill 时间
    uint64_t last_refill_us = 0;


    // -------------------------
    // Waiting Logical IO
    // -------------------------

    std::deque<spdk_bdev_io *> pending;


    // -------------------------
    // SPDK Poller
    // -------------------------

    struct spdk_poller *poller = nullptr;
};
```

QoS 挂到：

```cpp
struct bdev_fastblock {
    ...
    volume_qos qos;
};
```

一个：

```text
bdev_fastblock
```

对应一个 Volume/Image。

---

## 4. Token 核心语义：允许欠费

这是当前设计的重要规则。

Token **不要求大于当前 IO 的完整 cost**。

只要当前对应 Token：

```text
> 0
```

当前 IO 就允许放行。

放行之后直接扣完整 cost，因此 Token 可以变成负数。

例如：

```text
bw_tokens = 64KiB

来了：
WRITE 1MiB
```

因为：

```text
64KiB > 0
```

所以允许发送。

扣费：

```text
64KiB - 1MiB
= -960KiB
```

得到：

```text
bw_tokens = -960KiB
```

这表示：

```text
已经提前消费了未来 960KiB 的带宽额度。
```

后续 READ/WRITE 到来时：

```text
bw_tokens <= 0
```

不允许继续发送，进入：

```text
pending
```

Poller 后续不断 refill：

```text
-960KiB
-940KiB
-920KiB
...
0
+20KiB
```

当：

```text
bw_tokens > 0
```

才允许下一笔需要 BW 的 IO 放行。

IOPS 完全相同。

---

## 5. Admission 规则

READ / WRITE：

```text
iops_tokens > 0
AND
bw_tokens > 0
```

才能放行。

DELETE / DISCARD：

```text
iops_tokens > 0
```

即可，因为：

```text
bw_cost = 0
```

不要因为：

```text
bw_tokens <= 0
```

阻塞 DELETE。

逻辑：

```cpp
bool volume_qos_can_admit(
    volume_qos *qos,
    spdk_bdev_io *io)
{
    auto cost = volume_qos_get_cost(io);

    if (qos->iops_limit != 0 &&
        qos->iops_tokens <= 0) {
        return false;
    }

    if (cost.bytes > 0 &&
        qos->bw_limit != 0 &&
        qos->bw_tokens <= 0) {
        return false;
    }

    return true;
}
```

注意：

这里不是：

```text
iops_tokens >= iops_cost
bw_tokens   >= bw_cost
```

而是：

```text
当前余额 > 0
    ↓
允许当前IO消费
    ↓
扣完整cost
    ↓
允许变负
```

---

## 6. 扣费

建议：

```cpp
void volume_qos_charge(
    volume_qos *qos,
    spdk_bdev_io *io);
```

例如：

```cpp
void volume_qos_charge(
    volume_qos *qos,
    spdk_bdev_io *io)
{
    auto cost = volume_qos_get_cost(io);

    if (qos->iops_limit != 0) {
        qos->iops_tokens -= cost.iops;
    }

    if (qos->bw_limit != 0 && cost.bytes > 0) {
        qos->bw_tokens -= cost.bytes;
    }
}
```

例如：

```text
iops_tokens = 0.4

当前IO cost = 1

放行后：

0.4 - 1
= -0.6
```

下一笔 IO：

```text
iops_tokens <= 0
→ pending
```

---

## 7. Refill

建议：

```cpp
void volume_qos_refill(volume_qos *qos);
```

不要简单固定：

```text
每5ms固定加X
```

应该按真实 elapsed：

```cpp
elapsed = now - last_refill_us;

iops_tokens += iops_limit * elapsed_seconds;
bw_tokens   += bw_limit   * elapsed_seconds;

last_refill_us = now;
```

例如：

```text
IOPS = 1000
elapsed = 5ms

增加：
5 IO token
```

带宽：

```text
BW = 4MiB/s
elapsed = 5ms

增加：
约20.48KiB
```

如果之前：

```text
bw_tokens = -100KiB
```

则 refill：

```text
-100KiB
→ -79.52KiB
→ -59.04KiB
→ ...
→ > 0
```

然后 pending IO 才重新有资格放行。

---

## 8. Burst

配置：

```cpp
uint64_t burst_iops;
uint64_t burst_bw;
uint32_t burst_seconds;
```

含义：

```text
iops_limit / bw_limit
= 长期稳定速度

burst_iops / burst_bw
= 短时间峰值速度

burst_seconds
= Burst 时间窗口
```

Burst 主要限制 Token 正方向能够积累多少。

即：

```text
Token 最大正余额
```

不能因为 Volume 长时间空闲就无限积累：

```text
1000000 IO token
```

需要按照 burst 配置设置上限。

负方向则代表：

```text
Debt / 欠费
```

---

## 9. Pending Queue

直接使用 FastBlock 当前真实 Logical IO：

```cpp
std::deque<spdk_bdev_io *> pending;
```

不额外创建：

```cpp
pending_io
```

因为：

```cpp
spdk_bdev_io *
```

已经包含：

```text
IO type
iovs
iovcnt
offset_blocks
num_blocks
bdev
```

pending 中只能保存：

```text
尚未被 QoS 放行的 Logical IO
```

不能保存 Object/RPC 请求。

---

## 10. Poller

建议：

```cpp
int volume_qos_poller(void *arg);
```

Poller 做：

```text
1. 获取当前时间

2. 根据 elapsed refill

3. 更新 last_refill_us

4. 检查 pending.front()

5. Token > 0：
      charge
      pop
      resume

6. Token <= 0：
      停止本轮 dispatch
```

伪代码：

```cpp
int volume_qos_poller(void *arg)
{
    auto *qos = static_cast<volume_qos *>(arg);

    volume_qos_refill(qos);

    while (!qos->pending.empty()) {
        auto *io = qos->pending.front();

        if (!volume_qos_can_admit(qos, io)) {
            break;
        }

        volume_qos_charge(qos, io);

        qos->pending.pop_front();

        volume_qos_resume(io);
    }

    return SPDK_POLLER_BUSY;
}
```

---

## 11. IO 到达时

建议入口：

```cpp
bool volume_qos_try_admit(
    volume_qos *qos,
    spdk_bdev_io *io);
```

流程：

```text
Logical IO
    ↓
refill
    ↓
检查 Token
    │
    ├── 对应 Token > 0
    │       ↓
    │     charge
    │       ↓
    │     Token允许变负
    │       ↓
    │     RELEASE
    │
    └── Token <= 0
            ↓
        pending.push_back(io)
```

---

## 12. 函数命名

建议：

```cpp
void volume_qos_init(volume_qos *qos);

void volume_qos_fini(volume_qos *qos);

void volume_qos_refill(volume_qos *qos);

qos_cost volume_qos_get_cost(
    spdk_bdev_io *io);

bool volume_qos_can_admit(
    volume_qos *qos,
    spdk_bdev_io *io);

void volume_qos_charge(
    volume_qos *qos,
    spdk_bdev_io *io);

bool volume_qos_try_admit(
    volume_qos *qos,
    spdk_bdev_io *io);

void volume_qos_enqueue(
    volume_qos *qos,
    spdk_bdev_io *io);

void volume_qos_resume(
    spdk_bdev_io *io);

int volume_qos_poller(void *arg);
```

---

## 13. 代码接入位置

主要：

```text
src/bdev/bdev_fastblock.cc
src/bdev/volume_qos.h
src/bdev/volume_qos.cc
```

WRITE：

```text
_bdev_fastblock_submit_request
        ↓
bdev_fastblock_write
        ↓
Volume QoS
        ↓
ALLOW
        ↓
原 blk_cli->write
        ↓
Object split
```

READ：

```text
bdev_fastblock_get_buf_cb
        ↓
Volume QoS
        ↓
ALLOW
        ↓
原 blk_cli->read
```

QoS 必须位于：

```text
Logical IO
→ Object split
```

之前。

---

## 14. 多 Volume

每个 Volume 拥有独立：

```text
iops_limit
bw_limit

burst配置

iops_tokens
bw_tokens

last_refill_us

pending
poller
```

例如：

```text
Volume A
 └── volume_qos A

Volume B
 └── volume_qos B
```

它们默认独立 accounting。

---

## 15. Thread / Shard 约束

`spdk_bdev_io *` 不能随意换 SPDK thread 恢复。

原则：

```text
哪个 SPDK thread/shard 提交 Logical IO
就应该在哪个 thread/shard 恢复
```

如果同一个 Volume 被多个 shard 使用，需要保证各 shard 的 pending/resume 不造成跨线程调用 client。

---

## 16. 最终扣费模型

最终 QoS 语义：

```text
READ
    ↓
需要：
1 IOPS
+
IO bytes BW

如果：
IOPS token > 0
BW token > 0
    ↓
允许
    ↓
一次性扣完整 cost
    ↓
Token允许变负


WRITE
    ↓
同 READ


DELETE
    ↓
只检查：
IOPS token > 0
    ↓
扣1 IOPS
    ↓
不扣BW
```

负 Token：

```text
Token > 0
    = 当前可以放一笔新的IO

Token <= 0
    = 已经欠费
    = 新IO进入pending

Poller refill
    ↓
Token重新 > 0
    ↓
继续放行
```

因此整个模型最终是：

```text
                   Volume QoS
                       │
             ┌─────────┴─────────┐
             │                   │
         IOPS Token           BW Token
             │                   │
             └─────────┬─────────┘
                       │
                   Logical IO
                       │
              applicable token > 0 ?
                 /             \
               YES              NO
                │                │
             charge           pending
                │                │
         token可变成负数          │
                │                │
             RELEASE          poller
                │                │
                │          refill debt
                │                │
                └──────────◄─────┘
                       │
                FastBlock原路径
                       │
                  Object split
```

核心原则：

```text
余额为正即可放当前IO。

当前IO允许把Token扣成负数。

Token小于等于0后，
后续IO不能继续发送，进入pending。

Poller按实际elapsed持续补额度。

Token重新大于0以后，
再从pending继续放行。
```


- **Volume 配置接口还没真正做完。** 也就是怎么在创建/修改 Volume 时设置 `iops_limit`、`bw_limit`、`burst_iops`、`burst_bw`、`burst_seconds`、`timeslice_us`，以及运行中动态修改。
- **多 SPDK thread/channel 并发问题还没处理。** 当前一个 Volume 共享 `iops_tokens / bw_tokens / io_pending / poller`，如果多个线程同时 submit，会有并发风险。
- **Burst 语义还比较粗。** 现在实现本质上是用 `burst_iops * burst_seconds`、`burst_bw * burst_seconds` 作为 token 正余额上限。 如果你最终要求的是严格“峰值速率最多持续 N 秒”，后面还需要继续完善。
- **UNMAP/Delete 真正的数据路径还没实现。** QoS 已经能给它算 `1 IOPS + 0 BW`，但你前面确认了 FastBlock bdev 当前 UNMAP 最终还是直接失败，所以 QoS 支持不等于 delete 功能完整。
- **测试还没正式做。** 至少还要验证 IOPS、Bandwidth、负 token、pending 恢复、burst、FLUSH、多个 Volume、长时间 fio，以及 QoS disabled 时性能/行为是否和原来一致。
- **正式构建环境还有 Ninja/Makefiles 冲突。** 目前单文件和临时链接通过，但正式 build 还没真正跑通。