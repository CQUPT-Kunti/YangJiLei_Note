
# FastBlock 第一阶段复习：上层 IO → PG

相关：[[系统架构]]、[[FastBlock 结构体与缩写对照表]]、[[SPDK]]、[[块存储/什么是块存储|什么是块存储]]


这一阶段只看 Client 侧：

```text
Application / SPDK bdev
↓
bdev_fastblock_write()
↓
libblk_client::write()
↓
Image → Object
↓
calc_first_object_position()
↓
get_image_object_name()
↓
write_object()
↓
calc_target()
↓
得到 PG id
↓
send_request()
```

这一阶段解决的问题只有一个：

> **一次上层块 IO，怎么最终算出它对应哪个 PG。**

---

## 1. 上层 IO

上层先产生一个：

```text
spdk_bdev_io
```

可以把它理解成一张 IO 任务单：

```text
WRITE
写哪个 bdev
从哪个 block 开始
写多少 block
数据在哪里
```

---

## 2. bdev_fastblock_write()

作用：

> 把 SPDK 的块 IO 转换成 FastBlock 可以处理的 Image IO。

主要做：

```text
spdk_bdev_io
↓
找到 bdev_fastblock
↓
拿到：
pool_id
image_name
↓
block offset 转成 byte offset
↓
block length 转成 byte length
↓
找到当前 CPU shard 的 blk_client
↓
blk_cli->write()
```

核心理解：

```text
SPDK 的块请求
↓
FastBlock Image 写请求
```

---

## 3. libblk_client::write() 第一层

`spdk_bdev_io` 里的数据可能存在多个 `iovec` 中：

```text
iov[0]
iov[1]
iov[2]
```

内存地址可能不连续。

所以这里先：

```text
多个 iovec
↓
拼成一个连续 std::string buf
```

例如：

```text
2KB + 1KB + 1KB
↓
连续 4KB buf
```

注意：

> 这里说的是内存中的数据连续，不是磁盘物理位置连续。

---

## 4. libblk_client::write() 第二层

拿到连续的 `buf` 后：

> 按照 Object 边界，把一次 Image IO 拆成多个 Object IO。

例如：

```text
object_size = 4MB

写入：
offset = 6MB
length = 7MB
```

会拆成：

```text
Object1
offset = 2MB
写 2MB

Object2
offset = 0
写 4MB

Object3
offset = 0
写 1MB
```

所以这里完成：

```text
Image IO
↓
Object IO
```

---

## 5. calc_first_object_position()

作用：

> 算第一个 Object 怎么写。

返回三个信息：

```text
first_object_size
first_object_offset
object_seq
```

例如：

```text
offset = 6MB
object_size = 4MB
```

得到：

```text
object_seq = 1
object_offset = 2MB
first_object_size = 2MB
```

---

## 6. get_image_object_name()

作用：

> 根据 Image 和 Object 序号生成具体的 Object 名字。

大致：

```text
pool_id
+
image_name
+
object_seq
↓
object_name
```

例如：

```text
Object 序号 = 3
↓
生成对应的 object_name
```

之后系统主要通过 `object_name` 来处理这个 Object。

---

## 7. write_object()

作用：

> 把一个 Object 写操作正式包装成 `osd::write_request`。

写入的信息主要有：

```text
pool_id
pg_id
object_name
offset
data
```

但这里的 `pg_id` 还需要先通过：

```text
calc_target()
```

计算出来。

---

## 8. calc_target()

作用：

> Object → PG。

核心逻辑：

```text
object_name
↓
jenkins_hash()
↓
得到 hash 值
↓
获取这个 Pool 的 pg_num
↓
hash % pg_num
↓
得到 PG id
```

例如：

```text
Pool 1
pg_num = 8

Object Name
↓
hash = 12345

12345 % 8
↓
PG 1
```

所以：

```text
Object
↓
PG
```

就在这里完成。

---

## 9. send_request()

到这里第一阶段停止。

此时已经知道：

```text
我要写什么 Object
属于哪个 Pool
属于哪个 PG
写入 offset
写入 data
```

并且已经构造好了：

```text
osd::write_request
```

然后进入：

```text
send_request()
```

第二阶段才继续研究：

```text
PG
↓
Leader OSD
↓
RPC
```

---

# 第一阶段完整流程

```text
spdk_bdev_io
↓
bdev_fastblock_write()
↓
得到 pool_id / image_name
↓
block offset 转 byte offset
↓
找到 blk_client
↓
libblk_client::write()
↓
多个 iovec 拼成连续 buf
↓
按照 object_size 拆分
↓
Object1
Object2
Object3
↓
calc_first_object_position()
↓
确定第一个 Object 的位置和大小
↓
get_image_object_name()
↓
生成 Object 名
↓
write_object()
↓
构造 osd::write_request
↓
calc_target()
↓
Object Name → PG id
↓
send_request()
```

# 最终记忆

第一阶段只需要记住：

```text
SPDK IO
↓
Image
↓
Object
↓
PG
```

一句话：

> **第一阶段就是把 SPDK 的块 IO 转换成 FastBlock 的 Object IO，并最终计算出每个 Object 属于哪个 PG。**

到 `send_request()` 为止停止。

后面的：

```text
PG
↓
Leader OSD
↓
真正 RPC 发送
```

属于第二阶段。

---

## 相关笔记

- [[系统架构]]
- [[FastBlock 结构体与缩写对照表]]
- [[SPDK]]
- [[NVME]]
- [[块存储/什么是块存储|什么是块存储]]
