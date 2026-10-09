# OpenSSL EVP SHA256 使用方式

OpenSSL SHA256 基本流程：

```text
创建上下文
    ↓
初始化SHA256
    ↓
输入数据
    ↓
获取结果
    ↓
释放资源
```

---

# 一、普通指针管理

## 完整代码

```cpp
#include <openssl/evp.h>
#include <iostream>


void sha256()
{
    // 1. 创建上下文
    EVP_MD_CTX* ctx =
        EVP_MD_CTX_new();


    if(!ctx)
    {
        return;
    }


    // 2. 初始化SHA256
    EVP_DigestInit_ex(
        ctx,
        EVP_sha256(),
        nullptr
    );


    // 3. 输入数据
    const char* data = "hello";


    EVP_DigestUpdate(
        ctx,
        data,
        strlen(data)
    );


    // 4. 获取结果

    unsigned char hash[EVP_MAX_MD_SIZE];

    unsigned int hashLength = 0;


    EVP_DigestFinal_ex(
        ctx,
        hash,
        &hashLength
    );


    // 5. 释放资源

    EVP_MD_CTX_free(ctx);
}
```

---

## 流程说明

### 创建

```cpp
EVP_MD_CTX_new()
```

创建 SHA256 计算上下文。


---

### 初始化

```cpp
EVP_DigestInit_ex(
    ctx,
    EVP_sha256(),
    nullptr
);
```

选择 SHA256 算法。


---

### 输入数据

```cpp
EVP_DigestUpdate(
    ctx,
    data,
    size
);
```

把数据交给 SHA256 计算。

可以调用多次：

```cpp
Update()
Update()
Update()
```

适合大文件。


---

### 获取结果

```cpp
EVP_DigestFinal_ex(
    ctx,
    hash,
    &hashLength
);
```

生成最终 SHA256。


---

### 释放

```cpp
EVP_MD_CTX_free(ctx);
```

释放 OpenSSL 上下文。


---

# 二、unique_ptr 智能指针管理

因为：

```cpp
EVP_MD_CTX
```

不是 C++ 对象。

不能：

```cpp
delete ctx;
```

需要：

```cpp
EVP_MD_CTX_free(ctx);
```

所以给 unique_ptr 指定删除函数。


---

## 定义类型

```cpp
using ContextPtr =
    std::unique_ptr<
        EVP_MD_CTX,
        decltype(&EVP_MD_CTX_free)
    >;
```

含义：

```text
unique_ptr管理EVP_MD_CTX

释放时调用：

EVP_MD_CTX_free()
```

---

## 创建智能指针

```cpp
ContextPtr ctx(
    EVP_MD_CTX_new(),
    &EVP_MD_CTX_free
);
```

等价于：

```cpp
EVP_MD_CTX* ctx =
    EVP_MD_CTX_new();
```

但是会自动释放。


---

## 完整代码

```cpp
#include <openssl/evp.h>

#include <memory>


using ContextPtr =
    std::unique_ptr<
        EVP_MD_CTX,
        decltype(&EVP_MD_CTX_free)
    >;



void sha256()
{
    // 创建上下文
    ContextPtr ctx(
        EVP_MD_CTX_new(),
        &EVP_MD_CTX_free
    );


    if(!ctx)
    {
        return;
    }


    // 初始化
    EVP_DigestInit_ex(
        ctx.get(),
        EVP_sha256(),
        nullptr
    );


    const char* data = "hello";


    // 输入数据
    EVP_DigestUpdate(
        ctx.get(),
        data,
        strlen(data)
    );


    // 获取结果

    unsigned char hash[EVP_MAX_MD_SIZE];

    unsigned int hashLength = 0;


    EVP_DigestFinal_ex(
        ctx.get(),
        hash,
        &hashLength
    );


    // 不需要手动释放
}
```

---

# 两种方式区别

|方式|释放方式|
|-|-|
|普通指针|手动调用 `EVP_MD_CTX_free()`|
|unique_ptr|自动调用 `EVP_MD_CTX_free()`|

---

# 两个关键点

## 1. 为什么使用 ctx.get()

因为：

```cpp
ctx
```

类型：

```cpp
unique_ptr<EVP_MD_CTX>
```

而 OpenSSL 要：

```cpp
EVP_MD_CTX*
```

所以：

```cpp
ctx.get()
```

获取内部原始指针。


---

## 2. SHA256完整调用顺序

```cpp
EVP_MD_CTX_new()

        ↓

EVP_DigestInit_ex()

        ↓

EVP_DigestUpdate()

        ↓

EVP_DigestFinal_ex()

        ↓

EVP_MD_CTX_free()
```

---

## 相关笔记

- [[学习|C++ 学习地图]]
- [[强制转换]]
- [[文件操作]]

智能指针版本：

```cpp
EVP_MD_CTX_new()

        ↓

unique_ptr管理

        ↓

自动调用

EVP_MD_CTX_free()
```
