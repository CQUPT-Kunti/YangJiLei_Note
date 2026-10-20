## 1. std::variant 和 std::visit 是什么？

- `std::variant`：C++17 引入的类型安全联合体，同一时间只能存储一种类型。
- `std::visit`：根据 `variant` 当前存储的类型，执行对应的处理逻辑。
### 简单示例

```
#include <iostream>
#include <variant>
#include <string>

int main() {
    std::variant<int, std::string> data;

    data = 100;

    std::visit([](const auto& value) {
        std::cout << value << '\n';
    }, data);

    data = std::string("hello");

    std::visit([](const auto& value) {
        std::cout << value << '\n';
    }, data);
}
```

输出：

```
100
hello
```

`variant` 内部记录当前存储的是 `int` 还是 `string`，`visit` 会根据实际类型调用对应逻辑。

注意：切换类型时，variant 会自动管理旧对象的析构和新对象的构造，而不是完全不需要构造、析构。

## 2. 为什么不用 union？

|对比|union|variant|
|---|---|---|
|存储多种类型|支持|支持|
|自动记录当前类型|不支持|支持|
|类型安全检查|需要自己保证|自动检查|
|string 等复杂对象|生命周期管理麻烦|自动构造、析构|
|配合访问器分发|手动判断类型|`std::visit`|

variant 的主要优势是类型安全和自动管理对象生命周期。

## 3. 工业开发中什么场景使用？

|设计场景|为什么使用 variant|
|---|---|
|RPC 消息类型|同一个消息对象可以表示请求、响应、错误|
|事件驱动框架|统一存储连接、读写、超时等不同事件|
|状态机|使用不同类型表示不同状态|
|配置系统|配置值可能是 `int`、`bool`、`string`|
|编译器 AST|不同类型的语法节点使用统一的数据结构|

### 例如：RPC 框架中的消息处理

```
struct Request {
    std::string method;
};

struct Response {
    int result;
};

struct Error {
    std::string message;
};

using Message = std::variant<Request, Response, Error>;
```

处理消息：

```
Message msg = Request{"add"};

std::visit([](const auto& m) {
    using T = std::decay_t<decltype(m)>;

    if constexpr (std::is_same_v<T, Request>) {
        std::cout << "RPC Request: " << m.method;
    }
    else if constexpr (std::is_same_v<T, Response>) {
        std::cout << "RPC Response: " << m.result;
    }
    else {
        std::cout << "RPC Error: " << m.message;
    }
}, msg);
```

这样一个 `Message` 就能表示三种消息，避免手动使用 `union + type` 管理消息类型。

这种设计尤其适合候选类型固定、但运行时需要根据类型执行不同操作的场景。

面试总结：

`std::variant` 是类型安全的联合体，`std::visit` 根据当前有效类型进行访问。相比 `union`，它能自动管理 `std::string` 等复杂对象的构造和析构，并避免错误访问无效成员。

常见于 RPC 消息分发、事件系统、状态机和配置管理等场景