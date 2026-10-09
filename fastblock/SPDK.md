
# SPDK

相关：[[系统架构]]、[[FastBlock 结构体与缩写对照表]]、[[NVME]]

# 格式

1. SPDK 是 C 写的，所以你经常看到：

```
static callback + void* ctx
```

	因为普通成员函数会偷偷带一个 `this` 参数，C 函数指针不认识这个。
	
	加 `static` 后：

```
普通成员函数
= 有 this
= 不能直接传给 C callback

static 成员函数
= 没有 this
= 就像普通函数
= 可以传给 C callback
```

2. spdk_blob_io_write(blob, channel, buf, start_lba, num_lba, ...) 把 `buf` 里的数据，写到这个 Blob 的 `start_lba` 开始的 `num_lba` 个逻辑块里。
	虽然Blob 里的逻辑 LBA 是连续的，但是只是逻辑上，FastBlock在对修改的部分进行映射到磁盘的时候，也就是物理块上他不一定是真正连续的。

---

## 相关笔记

- [[系统架构]]
- [[FastBlock 第一阶段复习：上层 IO → PG]]
- [[FastBlock 结构体与缩写对照表]]
- [[NVME]]
- [[块存储/什么是块存储|什么是块存储]]
