# C++ condition_variable 等待函数

`std::condition_variable` 常用的等待函数有三个：

1. `wait()`
2. `wait_for()`
3. `wait_until()`

---

## 1. wait()

### 作用

一直等待，直到其他线程调用 `notify_one()` 或 `notify_all()` 唤醒。

### 使用方法

```cpp
std::mutex mutex_;
std::condition_variable cv;

bool ready = false;


void waitTask()
{
    std::unique_lock<std::mutex> lock(mutex_);

    cv.wait(lock, [] {
        return ready;
    });

    // ready == true 后继续执行
}
```

含义：

```text
一直等待

直到：
ready == true
```

---

## 2. wait_for()

### 作用

等待指定时间。

如果条件提前满足，会立即返回。

如果超过等待时间，也会返回。

### 使用方法

例如：

等待最多 1 分钟，除非存储大小超过限制。

```cpp
std::mutex mutex_;
std::condition_variable cv;

uint64_t totalBytes = 0;
uint64_t limit = 1024;


bool waitFlush()
{
    std::unique_lock<std::mutex> lock(mutex_);

    return cv.wait_for(
        lock,
        std::chrono::minutes(1),
        [] {
            return totalBytes >= limit;
        }
    );
}
```

含义：

```text
等待：

totalBytes >= limit

或者：

等待超过 1 分钟

满足任意一个就返回
```

---

## 3. wait_until()

### 作用

等待到指定时间点。

### 使用方法

```cpp
std::mutex mutex_;
std::condition_variable cv;


void waitTask()
{
    std::unique_lock<std::mutex> lock(mutex_);

    auto timeout =
        std::chrono::steady_clock::now()
        + std::chrono::minutes(1);


    cv.wait_until(
        lock,
        timeout
    );

    // 到时间后继续执行
}
```

含义：

```text
等待到：

当前时间 + 1分钟

然后返回
```

---

# 三者区别

| 函数 | 作用 |
| ---- | ---- |
| wait() | 一直等待，直到被唤醒 |
| wait_for() | 等待一段时间 |
| wait_until() | 等待到某个时间点 |

---

# 简单记忆

```text
wait
    等别人叫醒

wait_for
    等多久

wait_until
    等到什么时候
```

---

## 相关笔记

- [[学习|C++ 学习地图]]
- [[强制转换]]
- [[文件操作]]
