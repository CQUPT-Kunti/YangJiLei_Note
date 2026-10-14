# enable_shared_from_this & weak_ptr

在异步的情况下，外部 `this` 指针析构的时候，如果想要异步任务继续执行，就使用 `enable_shared_from_this` 来解决重复释放；如果不需要执行，直接 reject 的情况下使用 `weak_ptr`。他在把捕获 `this` 进行转换为 `shared_ptr` 的时候会因为外部释放而实例为 `nullptr`，不能进行任务。

## 1. enable_shared_from_this

它主要是为了防止两类问题：

- 重复释放内存（Double Free）。
- 异步回调中对象提前析构，导致悬空指针（Use After Free）。

### （1）避免重复释放内存

```cpp
class A {
public:
    std::shared_ptr<A> getPtr() {
        return std::shared_ptr<A>(this);
    }
};

auto p1 = std::make_shared<A>();
auto p2 = p1->getPtr();
```

虽然 `p1` 和 `p2` 指向同一个对象，但是它们拥有两个独立的引用计数控制块。

当 `p1` 和 `p2` 分别析构时，都认为自己需要销毁这个对象，导致重复释放，产生未定义行为。

### （2）用 shared_from_this 让 Lambda 持有共享所有权

只要 Lambda 持有它，对象就不会因为外部 `shared_ptr` 释放而析构。

## 2. weak_ptr

### （1）解决 shared_ptr 循环引用导致的内存泄漏

```cpp
class B;

class A {
public:
    std::shared_ptr<B> b;
};

class B {
public:
    std::shared_ptr<A> a;
};

auto a = std::make_shared<A>();
auto b = std::make_shared<B>();

a->b = b;
b->a = a;
```

结果：A 和 B 都无法析构，造成内存泄漏。

解决方法：将其中一个 `shared_ptr` 改成 `weak_ptr`。

### （2）解决异步回调中的对象生命周期问题

```cpp
async_call([this = weak_from_this()]() {
    handleRead();
});
```

- 对象还活着：`lock()` 成功，安全执行。
- 对象已销毁：`lock()` 失败，不执行。

---

# mutable

作用：

## 1. 允许在 const 成员函数中修改被 mutable 修饰的成员变量

```cpp
class A {
private:
    mutable int count = 0;
    int value = 10;

public:
    void func() const {
        count++;  // 正确：mutable 允许修改
        // value++;  // 错误：const 函数不能修改普通成员
    }
};
```

## 2. 修饰 Lambda 表达式，允许修改按值捕获的变量副本

正常捕获的时候 `operator()` 默认是 `const` 的。

```cpp
int x = 10;

auto func = [x]() mutable {
    x++;
    std::cout << x << '\n';
};

func();  // 11

std::cout << x;  // 10
```

---

# 汇编和编译

```text
预处理（.i） → 编译（.s） → 汇编（.o） → 链接（可执行文件）
```

- 预处理：展开头文件、宏替换、处理条件编译。输入 `.cpp` → 输出 `.i`
- 编译：语法分析、类型检查、代码优化、生成汇编代码。输入 `.i` → 输出 `.s`
- 汇编：将汇编代码转换成机器码。输入 `.s` → 输出 `.o`
- 链接：合并目标文件和库，进行符号解析、地址重定位。输入 `.o` + 库文件 → 输出可执行文件（Linux 下通常无后缀）

---

## 相关笔记

- [[学习|C++ 学习地图]]
- [[八股文/八股文索引|八股文问题索引]]
- [[互斥锁、拷贝构造、内存泄漏]]
