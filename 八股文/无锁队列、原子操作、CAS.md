# 无锁队列、原子操作、CAS

## 1. 常见原子操作

| 原子操作 | 作用 | 示例 |
| ---- | ---- | ---- |
| `load()` | 原子读取 | `a.load()` |
| `store()` | 原子写入 | `a.store(10)` |
| `exchange()` | 原子交换，返回旧值 | `a.exchange(20)` |
| `compare_exchange_weak()` | CAS 比较并交换，允许伪失败 | `a.compare_exchange_weak(old, 20)` |
| `compare_exchange_strong()` | CAS 比较并交换，无伪失败 | `a.compare_exchange_strong(old, 20)` |
| `fetch_add()` / `fetch_sub()` | 原子加减，返回旧值 | `a.fetch_add(1)` |

## 2. 内存序

| 内存序 | 核心作用 | 什么时候使用 | 常见操作 |
| ---- | ---- | ---- | ---- |
| `memory_order_relaxed` | 只保证原子性 | 只需安全修改变量，不需要同步其他数据 | `load`、`fetch_add` |
| `memory_order_release` | 发布数据 | 生产者写完数据，通知消费者 | `store` |
| `memory_order_acquire` | 获取数据 | 消费者收到通知，需要读取生产者写入的数据 | `load` |
| `memory_order_acq_rel` | 既获取又发布 | 一个原子操作既需要获取其他线程发布的数据，又需要发布自己的数据 | `exchange`、CAS、`fetch_sub` |
| `memory_order_seq_cst` | 保证统一的全局顺序 | 多个原子操作需要统一顺序，或暂时无法证明较弱内存序正确 | `load`、`store`、CAS |

### （1）relaxed：只保证原子性

```cpp
std::atomic<int> count{0};

count.fetch_add(1, std::memory_order_relaxed);
```

多个线程同时加一，不会丢失更新，但不负责同步其他普通变量。

### （2）release + acquire：无锁队列

以单生产者、单消费者的队列为例：

```cpp
int buffer[8];
std::atomic<int> tail{0};

// 生产者
void push(int value) {
    buffer[0] = value;  // ① 写入数据

    tail.store(1, std::memory_order_release); // ② 发布
}

// 消费者
void pop() {
    int t = tail.load(std::memory_order_acquire);

    if (t == 1) {
        int value = buffer[0]; // ③ 读取数据
    }
}
```

这里是简化的一次性入队示例，不包含完整的循环队列和槽位复用逻辑。

### （3）acq_rel：多线程接力，有读取也有发布

假设三个 RPC 工作线程分别计算结果，最后一个线程负责汇总：

```cpp
int results[3];
std::atomic<int> remaining{3};

void worker(int id) {
    results[id] = (id + 1) * 100;

    int old = remaining.fetch_sub(
        1,
        std::memory_order_acq_rel
    );

    if (old == 1) {
        // 最后一个线程
        // 可以安全读取三个线程写入的 results
        process(results);
    }
}
```

### （4）seq_cst：统一全局顺序

假设有两个线程：

```cpp
std::atomic<int> a{0}, b{0};

// 线程 A
a.store(1, std::memory_order_seq_cst);
int x = b.load(std::memory_order_seq_cst);

// 线程 B
b.store(1, std::memory_order_seq_cst);
int y = a.load(std::memory_order_seq_cst);
```

线程 A：先写 `a=1`，再读 `b`。

线程 B：先写 `b=1`，再读 `a`。

如果换成 `store(release)` 和 `load(acquire)`，两个都读到 0 则是可能的，因为它们不提供这样的全局顺序保证。

---

# CAS

CAS（Compare And Swap，比较并交换）是一种原子操作，常用于多线程并发、无锁队列和无锁数据结构。

核心思想：先比较内存中的值是否等于预期值，如果相等就修改，否则修改失败。

## 1. CAS 的三个参数

- V（Value）：内存中的当前值。
- A（Expected）：预期值。
- B（Desired）：希望修改成的新值。

## 2. 为什么无锁队列需要 CAS？

假设两个生产者同时往队列中添加元素。

```text
初始 tail = 5

生产者 A：读取 tail = 5
生产者 B：读取 tail = 5
```

如果直接执行：

```cpp
tail = tail + 1;
```

两个线程可能都认为自己应该使用下标 5，发生竞争。

使用 CAS：

```cpp
size_t expected = 5;

bool ok = tail.compare_exchange_weak(
    expected, 6
);
```

执行过程：

```text
初始 tail = 5

生产者 A：CAS(5, 6) → 成功
          tail = 6

生产者 B：CAS(5, 6) → 失败
          expected 更新为 6

生产者 B：重新尝试 CAS(6, 7) → 成功
          tail = 7
```

这样，两个生产者就不会成功抢占同一个 `tail` 值。

---

## 相关笔记

- [[学习|C++ 学习地图]]
- [[八股文/八股文索引|八股文问题索引]]
- [[互斥锁、拷贝构造、内存泄漏]]
- [[CV条件变量]]
