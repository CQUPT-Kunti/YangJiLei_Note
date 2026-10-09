# C++ `overload` 模板知识点简记

```cpp
template<typename ... Ts>
struct overload : Ts ... {
    using Ts::operator() ...;
};

template<class... Ts>
overload(Ts...) -> overload<Ts...>;
```

## 1. 可变参数模板

```cpp
template<typename ... Ts>
```

表示：

```text
Ts...
= 一组类型
```

例如传入 3 个 lambda：

```text
Ts...
=
lambda1 类型
lambda2 类型
lambda3 类型
```

---

## 2. 参数包展开

```cpp
struct overload : Ts ...
```

表示：

> 同时继承 `Ts...` 中的所有类型。

例如：

```text
Ts... = A, B, C
```

相当于：

```cpp
struct overload : A, B, C
```

---

## 3. Lambda 本质

Lambda 可以简单理解成一个编译器生成的匿名类，里面有：

```cpp
operator()
```

例如：

```cpp
[](int x) {}
```

可以粗略理解成：

```cpp
struct 某个匿名类 {
    void operator()(int x) {
    }
};
```

---

## 4. `using Ts::operator() ...`

```cpp
using Ts::operator() ...;
```

表示：

> 把所有父类 lambda 的 `operator()` 都引入进来。

最后一个 `overload` 对象就可以同时拥有：

```cpp
operator()(int)
operator()(std::string)
operator()(double)
```

形成函数重载。

---

## 5. 类模板参数推导

```cpp
template<class... Ts>
overload(Ts...) -> overload<Ts...>;
```

作用：

> 根据传入的 lambda，自动推导 `Ts...` 的具体类型。

所以可以直接写：

```cpp
auto handler = utils::overload {
    [](int) {},
    [](std::string) {}
};
```

不需要手动写每个 lambda 的类型。

---

## 6. `std::visit`

FastBlock 中：

```cpp
std::visit(cb_handler, req_stk->resp_cb);
```

作用：

> 根据 `resp_cb` 当前实际保存的类型，自动调用 `cb_handler` 里匹配的 lambda。

例如：

```text
write_object_callback
→ write lambda

read_object_callback
→ read lambda

delete_callback
→ delete lambda
```

---

## 相关笔记

- [[学习|C++ 学习地图]]
- [[CV条件变量]]
- [[强制转换]]
