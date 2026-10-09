# FastBlock Volume QoS 前置调研

相关：[[系统架构]]、[[FastBlock 结构体与缩写对照表]]、[[FastBlock 第一阶段复习：上层 IO → PG]]、[[SPDK]]、[[NVME]]

本文只分析当前项目代码，不实现 QoS，不修改项目源码。

## 结论

当前项目没有完整的传统 Volume QoS。

传统 QoS 指按卷配置 `max_iops`、`max_bandwidth`、read/write 独立限速、burst、min guarantee、公平调度、token bucket 等机制。当前代码里没有这些配置字段，也没有按时间补充额度的 token bucket / leaky bucket。

但 `kfastblock` 内核模块里已经有一个非常接近 QoS 插入点的 per-volume 调度机制：

```text
Volume
  -> kfastblock_scheduler_controller
  -> dispatch_window
  -> 每个 block request 采样一个 request dispatch window
  -> 限制该 request 同时下发的 object IO 数量
  -> object 完成后 refill，继续派发剩余 object
```

所以结论是：

| 问题 | 判断 |
|---|---|
| 项目是否已有 QoS | 疑似 / QoS-like |
| 是否已有传统卷级 QoS | 否 |
| 是否已有卷级调度状态 | 是 |
| 主要限制资源 | object dispatch 并发数，不是 IOPS / 带宽 |
| 当前算法 | dispatch window + completion refill；adaptive/pressure 策略近似 AIMD |
| 最适合插入 Volume QoS 的位置 | `kfastblock_queue_rq()` 之后、`kfastblock_transport_submit()` 之前，或 `kfastblock_transport_refill_dispatch_window()` 内 |

README 也把“实现卷QoS”列在 future works，说明项目语义上也认为它还没完成：`README.md:118-120`。

## 一、Volume 是什么

当前真正表示 Volume 的结构体是：

```text
kfastblock/include/kfastblock/volume.h
struct kfastblock_volume
```

源码位置：`../../fastblock/kfastblock/include/kfastblock/volume.h:163`

它是内核态 block device 的卷对象。每个 `kfastblock_volume` 基本对应一个 Linux block device：

| 成员 | 作用 |
|---|---|
| `dev_id` / `major` / `minor` | Linux block device 设备号相关信息 |
| `struct gendisk *disk` | Linux block layer 暴露出来的磁盘对象 |
| `char disk_name[DISK_NAME_LEN]` | 设备名，格式类似 `kfb0` |
| `struct blk_mq_tag_set tag_set` | blk-mq 队列的 tag set；每个 request 的 private data 是 `struct kfastblock_request` |
| `open_count` | 当前打开该 block device 的引用数 |
| `ready` | Volume 是否可处理 IO |
| `inflight_ios` | 该 Volume 当前 block IO 在途数量 |
| `spec` | attach 参数，包括 monitor 地址、pool/image 名、debug 参数 |
| `view` | 当前卷看到的 image 信息、OSD map、PG route、leader 等元数据 |
| `stats` / `pipeline_stats` | IO、object pipeline、错误等统计 |
| `object_buffer_pool` | object IO 临时 buffer 池，限制和 `dispatch_window` 相关 |
| `scheduler` | per-volume 的调度控制器 |
| `inflight_wq` | 等待 `inflight_ios` 归零，用于 flush/stop/drain，不是 QoS 等待队列 |
| `flush_in_progress` | flush 排它门闩，有 flush 时普通 IO 返回 resource |
| `state_lock` | 保护 volume view / dispatch window 等状态 |
| `refresh_work` | 定时刷新 image / cluster map |
| `queue_paused` | 队列暂停标志，元数据不健康或手动暂停时使用 |
| `manual_queue_pause` | 手动暂停标志 |
| `dispatch_window` | 默认 object dispatch 窗口，初始值 8 |
| `socket_cache` / `monitor_cache` | OSD / monitor TCP socket 缓存 |

### Volume 创建

入口：

```text
kfastblock_volume_attach()
```

源码位置：`../../fastblock/kfastblock/src/volume.c:4962`

创建流程：

```text
kfastblock_volume_attach(spec, major, bus, parent_dev)
  -> 检查同名 pool/image 是否已 attach
  -> kzalloc(struct kfastblock_volume)
  -> 初始化 ready/open_count/inflight_ios/queue_paused/dispatch_window
  -> kfastblock_buffer_pool_init()
  -> kfastblock_scheduler_init()
  -> INIT_DELAYED_WORK(refresh_work)
  -> kfastblock_volume_copy_attach_spec()
  -> kfastblock_meta_bootstrap_volume()
  -> kfastblock_volume_add_disk()
  -> list_add_tail(&vol->node, &g_kfastblock_volumes)
  -> atomic_set(&vol->ready, 1)
  -> kfastblock_volume_schedule_refresh()
```

关键点：

`kfastblock_volume_attach()` 初始化了 `vol->scheduler` 和 `vol->dispatch_window`。这说明现有 scheduler 是 per-volume，不是全局单例。

### Volume 注册到 Linux block layer

入口：

```text
kfastblock_volume_add_disk()
```

源码位置：`../../fastblock/kfastblock/src/volume.c:4888`

真实注册流程：

```text
vol->tag_set.ops = &kfastblock_mq_ops
vol->tag_set.queue_depth = KFASTBLOCK_DEFAULT_QUEUE_DEPTH
vol->tag_set.cmd_size = sizeof(struct kfastblock_request)
blk_mq_alloc_tag_set(&vol->tag_set)
blk_mq_alloc_disk(&vol->tag_set, vol)
vol->disk->private_data = vol
vol->disk->fops = &kfastblock_bd_ops
set_capacity(vol->disk, vol->view.image.size_bytes >> SECTOR_SHIFT)
kfastblock_volume_apply_queue_limits(vol)
vol->disk->queue->queuedata = vol
device_add_disk(&vol->dev, vol->disk, NULL)
```

几个重要事实：

| 问题 | 答案 |
|---|---|
| Volume 是否一一对应 block device | 是，`vol->disk` 是它的 `gendisk` |
| Volume 是否拥有 request queue | 是，`blk_mq_alloc_disk()` 创建 queue，`queuedata = vol` |
| Volume 是否拥有 scheduler/controller | 是，`vol->scheduler` |
| blk-mq queue depth | `KFASTBLOCK_DEFAULT_QUEUE_DEPTH = 128` |
| 每个 request 的 PDU | `struct kfastblock_request` |

### Volume 销毁

主要路径：

```text
kfastblock_volume_detach()
  -> kfastblock_volume_unregister()
  -> kfastblock_volume_begin_stop()
  -> blk_mq_quiesce_queue()
  -> kfastblock_volume_drain_io()
  -> del_gendisk()
  -> put_disk()
  -> device_unregister()
  -> kfastblock_volume_free()
```

源码位置：

```text
../../fastblock/kfastblock/src/volume.c:1914
../../fastblock/kfastblock/src/volume.c:4839
../../fastblock/kfastblock/src/volume.c:5072
```

销毁时会：

```text
ready = 0
cancel refresh_work
quiesce block queue
等待 inflight_ios 归零
删除 gendisk
释放 tag_set / buffer pool / socket cache / meta view
```

## 二、Volume 和后端存储关系

`kfastblock_volume` 自己不直接保存本地盘对象。它保存的是 image / pool / pg / osd 的路由视图：

```text
struct kfastblock_cluster_view {
    struct kfastblock_image_info image;
    osdmap_epoch;
    pgmap_epoch;
    leader_epoch;
    osds;
    routes;
}
```

源码位置：`../../fastblock/kfastblock/include/kfastblock/meta.h:64`

`image` 包含：

| 字段 | 作用 |
|---|---|
| `pool_name` | pool 名 |
| `image_name` | image 名，也就是当前卷对应的远端块设备名 |
| `size_bytes` | 卷大小 |
| `block_size` | Linux queue logical block size |
| `object_size` | 拆分 object 的大小，默认 4 MiB |
| `pool_id` | pool id |
| `pg_count` | PG 数量 |
| `read_only` | 是否只读 |

实际后端路径是：

```text
Volume
  -> image metadata
  -> object_name
  -> pg_id
  -> PG leader OSD
  -> TCP socket
  -> OSD 服务端
  -> 用户态后端 / Raft / LocalStore / SPDK Blobstore
```

也就是说：Volume 是 Linux block device 的前端视图；后端由 `view.routes` 和 leader 选择决定。

## 三、完整 Block IO 请求路径

### read/write/discard 主路径

真实调用链：

```text
Linux blk-mq
  -> kfastblock_queue_rq()
  -> kfastblock_request_init()
  -> kfastblock_scheduler_sample_window()
  -> kfastblock_request_split()
  -> kfastblock_volume_account_io_submit()
  -> kfastblock_transport_submit()
  -> kfastblock_transport_run_request_submit()
  -> kfastblock_transport_prepare_request_runtime()
  -> kfastblock_request_prepare_runtime()
  -> kfastblock_transport_prefetch_submit_leaders()
  -> kfastblock_request_pick_dispatch_batch()
  -> kfastblock_transport_queue_object_work()
  -> queue_work(g_kfastblock_transport_wq)
  -> kfastblock_transport_object_work()
  -> kfastblock_transport_submit_object()
  -> kfastblock_transport_write_object()
     或 kfastblock_transport_read_object()
     或 kfastblock_transport_delete_object()
  -> kfastblock_transport_finish_object_success()
  -> kfastblock_transport_complete_request()
  -> kfastblock_transport_refill_dispatch_window()
  -> blk_mq_end_request()
```

### `kfastblock_queue_rq()`

源码位置：`../../fastblock/kfastblock/src/volume.c:4541`

输入：

```text
struct blk_mq_hw_ctx *hctx
const struct blk_mq_queue_data *bd
```

关键逻辑：

```text
vol = hctx->queue->queuedata
rq = bd->rq
kf_req = blk_mq_rq_to_pdu(rq)
blk_mq_start_request(rq)

if !vol || !ready || queue_paused:
    blk_mq_end_request(BLK_STS_RESOURCE)

if passthrough:
    blk_mq_end_request(BLK_STS_IOERR)

if FLUSH:
    kfastblock_volume_handle_flush()

if read_only and write/discard:
    blk_mq_end_request(BLK_STS_IOERR)

down_read(&vol->state_lock)
if !meta_io_ready:
    kick_refresh()
    blk_mq_end_request(BLK_STS_RESOURCE)

if flush_in_progress:
    blk_mq_end_request(BLK_STS_RESOURCE)

kfastblock_volume_get_io(vol)
kfastblock_request_init(kf_req, vol, rq)
kfastblock_request_split(kf_req)
up_read(&vol->state_lock)
kfastblock_volume_account_io_submit()
kfastblock_transport_submit(kf_req)
```

它修改的关键状态：

| 状态 | 修改 |
|---|---|
| `vol->inflight_ios` | `kfastblock_volume_get_io()` 加一 |
| `kf_req` | 初始化为当前 block request 的私有请求 |
| `kf_req->dispatch_window` | 在 `request_init()` 中采样 |
| `kf_req->objects` | 在 `request_split()` 中生成 |

和 Volume 的关系：

`hctx->queue->queuedata` 就是 `vol`。所以每个 block request 从一开始就能拿到自己的 Volume。

### flush 路径

真实调用链：

```text
kfastblock_queue_rq()
  -> req_op(rq) == REQ_OP_FLUSH
  -> kfastblock_volume_handle_flush()
  -> kfastblock_volume_begin_flush()
  -> kfastblock_volume_drain_io()
  -> wait_event_timeout(vol->inflight_wq, inflight_ios == 0)
  -> blk_mq_end_request()
```

源码位置：`../../fastblock/kfastblock/src/volume.c:1839`

flush 不拆 object，也不走 `transport_submit()`。它等待当前 Volume 上已有的普通 IO 完成。`inflight_wq` 的唤醒点在：

```text
kfastblock_volume_put_io()
  -> atomic_dec_and_test(&vol->inflight_ios)
  -> wake_up_all(&vol->inflight_wq)
```

源码位置：`../../fastblock/kfastblock/src/volume.c:1804`

这个等待队列是 drain/flush 用的，不是 QoS pending queue。

## 四、Block Request 如何拆成 Object IO

核心结构体：

```text
struct kfastblock_request
struct kfastblock_object_extent
struct kfastblock_request_object_runtime
```

源码位置：`../../fastblock/kfastblock/include/kfastblock/request.h:39`、`../../fastblock/kfastblock/include/kfastblock/request.h:63`、`../../fastblock/kfastblock/include/kfastblock/request.h:99`

### object 数量如何计算

估算函数：

```text
kfastblock_request_estimate_extent_count()
```

源码位置：`../../fastblock/kfastblock/src/request.c:136`

逻辑：

```text
first_object = byte_offset / object_size
last_object = (byte_offset + byte_length - 1) / object_size
count = last_object - first_object + 1
```

如果超过 `KFASTBLOCK_MAX_OBJECT_EXTENTS = 128`，返回 `KFASTBLOCK_MAX_OBJECT_EXTENTS + 1`，后续分配阶段返回 `-E2BIG`。

### object size 是多少

默认：

```text
KFASTBLOCK_DEFAULT_OBJECT_SIZE = 4 MiB
KFASTBLOCK_MAX_IO_BYTES = 4 MiB
```

源码位置：`../../fastblock/kfastblock/include/kfastblock/common.h:13`

实际请求使用：

```text
kf_req->request_object_size = vol->view.image.object_size
```

源码位置：`../../fastblock/kfastblock/src/request.c:324`

### 跨 object 如何拆分

真实拆分函数：

```text
kfastblock_request_split()
```

源码位置：`../../fastblock/kfastblock/src/request.c:1387`

伪代码：

```text
current_offset = request byte offset
remaining = request byte length

while remaining > 0:
    object_seq = current_offset / object_size
    object_offset = current_offset % object_size
    object_len = min(remaining, object_size - object_offset)

    extent.object_seq = object_seq
    extent.request_offset = original_length - remaining
    extent.object_offset = object_offset
    extent.length = object_len
    extent.object_name = "{pool_id}__blk_data___{image_name}{object_seq}"
    extent.pg_id = calc_pg(object_name, pg_count)
    track_unique_pg(pg_id)

    remaining -= object_len
    current_offset += object_len
    nr_objects++

capture_pg_hints()
```

`object_extent` 保存的是一个 object 子请求的定位信息：

| 字段 | 作用 |
|---|---|
| `object_seq` | 第几个 object |
| `request_offset` | 这段 object 数据在原始 block request 中的偏移 |
| `object_offset` | 这段 IO 在 object 内的偏移 |
| `length` | 本 object 子请求长度 |
| `pg_id` | 该 object 映射到哪个 PG |
| `object_name` | 后端对象名 |

### object 状态机

状态定义：

```text
INIT
READY
QUEUED
IN_FLIGHT
REQUEUE
DONE
FAILED
CANCELLED
```

源码位置：`../../fastblock/kfastblock/include/kfastblock/request.h:21`

常规成功路径：

```text
INIT
  -> READY        kfastblock_request_prepare_runtime()
  -> QUEUED       kfastblock_request_pick_dispatch_batch()
  -> IN_FLIGHT    kfastblock_request_mark_object_inflight()
  -> DONE         kfastblock_request_mark_object_complete(ret == 0)
```

失败路径：

```text
IN_FLIGHT
  -> FAILED       kfastblock_request_mark_object_complete(ret != 0)
```

重试路径：

```text
QUEUED / IN_FLIGHT
  -> REQUEUE
  -> READY        kfastblock_request_requeue_object()
```

取消路径：

```text
READY / QUEUED
  -> CANCELLED    kfastblock_request_cancel_unqueued()
```

### request 如何知道完成

`kfastblock_transport_prepare_request_runtime()` 设置：

```text
atomic_set(&kf_req->pending_objects, kf_req->nr_objects)
```

源码位置：`../../fastblock/kfastblock/src/transport.c:3487`

每个 object 完成后：

```text
kfastblock_transport_complete_request()
  -> kfastblock_transport_update_request_status_after_object()
  -> kfastblock_transport_advance_request_pending()
  -> atomic_dec_return(&kf_req->pending_objects)
  -> remaining == 0 时 kfastblock_transport_finish_request_now()
  -> blk_mq_end_request()
```

源码位置：`../../fastblock/kfastblock/src/transport.c:3295`、`../../fastblock/kfastblock/src/transport.c:3315`、`../../fastblock/kfastblock/src/transport.c:3207`

## 五、现有 Scheduler / Backpressure

核心结构：

```text
struct kfastblock_scheduler_controller
```

源码位置：`../../fastblock/kfastblock/include/kfastblock/scheduler.h:25`

字段：

| 字段 | 作用 |
|---|---|
| `lock` | 保护 scheduler 状态 |
| `base_window` | 配置窗口，默认来自 `vol->dispatch_window` |
| `current_window` | 当前动态窗口 |
| `min_window` | 最小窗口，初始化为 1 |
| `max_window` | 最大窗口，初始化为 128 |
| `pressure_inflight_limit` | pressure 策略下，Volume 在途 IO 超过该值时缩小窗口 |
| `cooldown_ms` | retry/dispatch failure 后的冷却时间，默认 250ms |
| `consecutive_successes` | 连续成功计数，用于增长窗口 |
| `shrink_events` / `grow_events` | 窗口收缩/增长次数 |
| `retry_events` | retry 次数 |
| `dispatch_failures` | workqueue 派发失败次数 |
| `sample_events` | 采样次数 |
| `pressure_limited_samples` | 因 inflight pressure 被限制的采样次数 |
| `cooldown_limited_samples` | 因 cooldown 被限制的采样次数 |
| `cooldown_until_jiffies` | cooldown 截止时间 |
| `policy` | static/adaptive/pressure |
| `dynamic_enabled` | 是否启用动态窗口 |

### 初始化

`kfastblock_volume_attach()` 里：

```text
vol->dispatch_window = KFASTBLOCK_DEFAULT_OBJECT_DISPATCH_WINDOW
kfastblock_scheduler_init(&vol->scheduler, vol->dispatch_window, 1, KFASTBLOCK_MAX_OBJECT_EXTENTS)
```

源码位置：`../../fastblock/kfastblock/src/volume.c:4990`

`KFASTBLOCK_DEFAULT_OBJECT_DISPATCH_WINDOW = 8`。

### policy

定义：

```text
STATIC = 0
ADAPTIVE = 1
PRESSURE = 2
```

源码位置：`../../fastblock/kfastblock/include/kfastblock/scheduler.h:8`

#### STATIC

如果 `dynamic_enabled == false`，或者 policy 是 `STATIC`，采样使用固定窗口：

```text
window = base_window
```

不会因 success/retry/failure 动态调整。

#### ADAPTIVE

默认策略是 `ADAPTIVE`。

它的逻辑是：

```text
sample_window:
    effective_window = current_window

success:
    consecutive_successes++
    if consecutive_successes >= current_window:
        current_window = min(current_window + 1, base_window)

retry or dispatch failure:
    current_window = max(current_window / 2, min_window)
```

这是“加性增大、乘性减小”的 AIMD 形态。

#### PRESSURE

`PRESSURE` 在 `ADAPTIVE` 基础上，采样时会看当前 Volume 的 `inflight_ios` 和 cooldown：

```text
if pressure_inflight_limit && inflight_ios > pressure_inflight_limit:
    over = inflight_ios - pressure_inflight_limit
    limited = max(current_window - over, min_window)

if cooldown_active:
    limited = max(limited / 2, min_window)

effective_window = min(limited, request_objects)
```

源码位置：`../../fastblock/kfastblock/src/scheduler.c:72`、`../../fastblock/kfastblock/src/scheduler.c:242`

### scheduler 什么时候被采样

采样点在：

```text
kfastblock_request_init()
  -> kfastblock_scheduler_sample_window(&vol->scheduler, atomic_read(&vol->inflight_ios), kf_req->max_object_extents, NULL)
  -> kf_req->dispatch_window = sampled value
```

源码位置：`../../fastblock/kfastblock/src/request.c:336`

注意：采样发生在 block request 初始化阶段。采样结果复制到 `kf_req->dispatch_window`。后续同一个 request 的 refill 使用这个固定值，不会每次 refill 重新采样 `vol->scheduler`。

### scheduler 什么时候更新

成功：

```text
kfastblock_transport_finish_object_success()
  -> kfastblock_scheduler_note_success(&ctx->kf_req->vol->scheduler)
```

源码位置：`../../fastblock/kfastblock/src/transport.c:2317`

retry：

```text
kfastblock_recovery_apply_object_failure()
  -> kfastblock_scheduler_note_retry(&vol->scheduler)
```

源码位置：`../../fastblock/kfastblock/src/recovery.c:215`

dispatch failure：

```text
kfastblock_transport_queue_object_work()
  -> queue_work() 失败
  -> kfastblock_scheduler_note_dispatch_failure(&kf_req->vol->scheduler)
```

源码位置：`../../fastblock/kfastblock/src/transport.c:646`

### dispatch window 如何限流

`kfastblock_request_dispatch_credits()`：

```text
window = kf_req->dispatch_window ? kf_req->dispatch_window : 1
dispatched = kf_req->queued_objects + kf_req->inflight_objects
credits = dispatched >= window ? 0 : window - dispatched
```

源码位置：`../../fastblock/kfastblock/src/request.c:963`

这不是按秒限速，而是“同一个 block request 内，同时 queued + inflight 的 object 数不能超过 dispatch_window”。

### window 满了会怎样

不会 sleep。

不会阻塞线程。

不会把请求放入一个专门的 QoS wait queue。

不会返回限流错误。

真实行为是：

```text
READY object 暂时留在 kf_req->object_runtime
已派发 object 完成后
  -> kfastblock_transport_refill_dispatch_window()
  -> 根据 credits 再 pick 一批 READY object
  -> queue_work()
```

源码位置：`../../fastblock/kfastblock/src/transport.c:693`、`../../fastblock/kfastblock/src/transport.c:3295`

所以当前 backpressure 是 request-local 的窗口回填模型。

## 六、并发控制机制汇总

| 机制 | 位置 | 作用 | 是否 QoS |
|---|---|---|---|
| blk-mq queue depth | `volume.c:4896` | 每卷 blk-mq 深度 128 | 不是 QoS，是 Linux queue 并发上限 |
| `vol->inflight_ios` | `volume.h:174` | 统计当前 Volume 在途 block IO | 可作为 QoS 输入 |
| `flush_in_progress` | `volume.c:1813` | flush 排它，普通 IO 返回 resource | 不是 QoS |
| `queue_paused` | `volume.c:1691`、`volume.c:4549` | 元数据不健康/手动暂停时 quiesce queue | 不是 QoS |
| `state_lock` | `volume.h:196` | 保护 view 和部分卷状态 | 不是 QoS |
| `dispatch_window` | `volume.h:201` | 限制 request 内 object 并发派发 | QoS-like |
| `object_state_lock` | `request.h:116` | 保护 object 状态计数 | 不是 QoS |
| `dispatch_lock` | `request.h:117` | request dispatch 互斥 | 可复用为 QoS/refill 保护点 |
| `g_kfastblock_transport_wq` | `transport.c` | object workqueue，真正异步执行 object IO | 不是 QoS，但负责后续调度 |
| socket cache | `volume.h:204` | OSD/monitor 连接复用 | 不是 QoS |
| `cooldown_until_jiffies` | `scheduler.h:47` | retry/failure 后限制采样窗口 | QoS-like |

## 七、当前没有哪些 QoS 能力

搜索过的相关机制包括：

```text
qos / QoS
throttle / throttling
rate_limit / ratelimit / rate limiter
token_bucket / token bucket
limiter
iops
bandwidth / bw_limit
bps
burst
quota
scheduler
pending queue / wait queue / qos queue
acquire / consume / refill
max_iops / min_iops
max_bandwidth / min_bandwidth
```

实际发现：

| 能力 | 当前是否存在 |
|---|---|
| 最大 IOPS | 否 |
| 最小 IOPS 保证 | 否 |
| 最大带宽 bytes/s | 否 |
| 最小带宽保证 | 否 |
| 读 IOPS / 写 IOPS 分开限制 | 否 |
| 读带宽 / 写带宽分开限制 | 否 |
| burst credit | 否 |
| token bucket | 否 |
| leaky bucket | 否 |
| fixed time slice quota | 否 |
| priority queue | 否 |
| weighted fair queue | 否 |
| 多租户公平调度 | 否 |
| request 级延迟队列 | 否 |
| QoS timer/poller | 否 |

有些 `ratelimited` 是 Linux 日志限频，例如 `dev_info_ratelimited()`，不是 IO 限速。

## 八、当前架构是否适合实现 Volume QoS

适合。

原因：

1. `kfastblock_volume` 已经是 per-volume 对象。
2. Linux request queue 的 `queuedata` 直接指向 `vol`。
3. 每个 block request 初始化时都能拿到 `vol`、`rq`、op、bytes。
4. 现有 `vol->scheduler` 已经提供了 per-volume controller 的位置。
5. 现有 object dispatch/refill 已经提供了“延迟派发”的自然位置。
6. `inflight_ios`、`queued_objects`、`inflight_objects` 已经有现成统计。

但也有一个限制：

当前 `dispatch_window` 控的是“一个 request 内 object 并发”，不是“整个 Volume 每秒允许多少 IO/bytes”。如果要做真正 Volume QoS，不能只改 `dispatch_window` 字段；至少要新增 per-volume 的额度状态。

## 九、最适合插入 Volume QoS 的位置

### 位置 A：`kfastblock_queue_rq()` 中，在 `request_init()` 之后

位置：

```text
kfastblock_queue_rq()
  -> kfastblock_request_init()
  -> [Volume QoS check]
  -> kfastblock_request_split()
  -> kfastblock_transport_submit()
```

优点：

| 优点 | 说明 |
|---|---|
| 能看到原始 block request | 可按 `req_op(rq)` 区分 read/write/discard |
| 能看到 bytes | 可做 IOPS 和 bandwidth |
| 已经拿到 `vol` | 天然 per-volume |
| 提交前限制 | 不会占用后端 object / socket / workqueue |

缺点：

如果额度不足，需要一个 per-volume pending request queue 或者让 blk-mq later retry。否则只能返回 `BLK_STS_RESOURCE`，语义更像设备忙，不像精确 QoS。

### 位置 B：`kfastblock_transport_submit()` 内，runtime 准备后、initial dispatch 前

位置：

```text
kfastblock_transport_prepare_request_runtime()
  -> [Volume QoS consume/check]
  -> kfastblock_transport_prepare_initial_dispatch_batch()
```

优点：

| 优点 | 说明 |
|---|---|
| request 已经 split | 可以知道 `nr_objects`、PG 分布、object length |
| 可复用 dispatch/refill 模型 | 更容易把不足额度的 object 留在 READY |
| 不阻塞 blk-mq 入口线程 | 后续由 worker/refill 驱动 |

缺点：

如果要按 block request 粒度排队，这里已经进入 transport 语义，设计要更小心。

### 位置 C：`kfastblock_transport_refill_dispatch_window()` 内

位置：

```text
object complete
  -> kfastblock_transport_refill_dispatch_window()
  -> [Volume QoS credits]
  -> kfastblock_request_pick_dispatch_batch()
```

优点：

| 优点 | 说明 |
|---|---|
| 最贴合当前架构 | 当前已经是 completion-driven refill |
| 不需要阻塞 | 可让 READY object 等待下一次额度 |
| 容易和 dispatch_window 组合 | `credits = min(window_credits, qos_credits)` |

缺点：

真正 token bucket 需要 timer/work 来在没有 object 完成时重新唤醒，否则额度恢复但没有完成事件触发 refill。

### 建议

最小可行设计：

```text
per-volume qos controller 放在 struct kfastblock_volume

kfastblock_queue_rq():
    做 request 级 IOPS/bytes 统计或快速拒绝/排队判断

kfastblock_transport_refill_dispatch_window():
    做 object dispatch 级额度检查
    credits = min(dispatch_window_credits, qos_credits)

timer/delayed_work:
    周期性 refill token
    唤醒/继续 dispatch pending request/object
```

如果目标是传统“卷级 QoS”，核心对象应该绑定到 `struct kfastblock_volume`，不是绑定到 `struct kfastblock_request`。`request` 只保存本次请求消耗了多少额度、是否被延迟。

## 十、后续实现 Volume QoS 的设计草图

建议新增概念：

```text
struct kfastblock_qos_config {
    u64 max_read_iops;
    u64 max_write_iops;
    u64 max_read_bps;
    u64 max_write_bps;
    u64 read_burst;
    u64 write_burst;
    bool enabled;
};

struct kfastblock_qos_controller {
    spinlock_t lock;
    struct kfastblock_qos_config config;
    u64 read_iops_tokens;
    u64 write_iops_tokens;
    u64 read_bytes_tokens;
    u64 write_bytes_tokens;
    unsigned long last_refill_jiffies;
    struct delayed_work refill_work;
    struct list_head pending_requests;
};
```

放置位置：

```text
struct kfastblock_volume {
    ...
    struct kfastblock_scheduler_controller scheduler;
    struct kfastblock_qos_controller qos;
    ...
};
```

注意：这是后续设计建议，不是当前代码已有实现。

## 十一、最终判断

当前项目已经有：

```text
per-volume block device
per-volume request queue
per-volume inflight counter
per-volume dispatch scheduler
request -> object split
object dispatch window
completion-driven refill
retry/failure/success 反馈
```

当前项目没有：

```text
per-volume max_iops
per-volume max_bandwidth
read/write independent qos
token bucket
qos pending queue
qos timer wakeup
min guarantee
priority/fair scheduler
```

一句话结论：

当前 `kfastblock` 还没有传统卷级 QoS，但已经有一个非常好的落点：`struct kfastblock_volume` 里的 `scheduler/dispatch_window` 和 `transport_refill_dispatch_window()`。后续实现 Volume QoS 时，最稳的路线是在 Volume 上新增 QoS controller，把 `dispatch_window` 当作 object 并发控制，把 QoS token bucket 当作每卷 IOPS/带宽控制，两者取最小额度后再 dispatch。

