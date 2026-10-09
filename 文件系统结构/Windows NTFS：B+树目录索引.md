# Windows NTFS

## 1. NTFS 的核心文件记录：MFT Record

NTFS 最核心的数据结构是：

```text
MFT
Master File Table
```

可以简单理解成：

```text
MFT

Record 100
Record 101
Record 102
...
Record 500
```

每个文件或目录都有对应的 MFT Record。

例如：

```text
D:\资料\项目A\你好.jpg
```

假设：

```text
资料      → MFT 100
项目A     → MFT 200
你好.jpg   → MFT 500
```

---

## 2. MFT Record 里面有什么？

NTFS 和 ext4 最大区别之一是：

> MFT Record 更像一个“属性容器”。

例如：

```text
MFT 500
│
├── Record Header
│
├── $STANDARD_INFORMATION
│     创建时间
│     修改时间
│     文件属性
│     ...
│
├── $FILE_NAME
│     Name = "你好.jpg"
│     Parent = MFT 200
│
└── $DATA
      文件数据
      或数据位置
```

常见属性：

```text
$STANDARD_INFORMATION
$FILE_NAME
$DATA
$ATTRIBUTE_LIST
$INDEX_ROOT
$INDEX_ALLOCATION
```

---

## 3. NTFS 怎么在目录里找文件？

假设系统已经知道：

```text
D:\资料\项目A
```

对应：

```text
MFT 200
```

现在要找：

```text
你好.jpg
```

项目A 是一个目录，因此有目录索引：

```text
MFT 200
   ↓
目录索引
   ↓
按文件名查找
   ↓
"你好.jpg" → MFT 500
```

NTFS 目录使用 B-tree / B+tree 风格的索引结构。

重点：

```text
这个树只负责当前目录。
```

不是：

```text
整个D盘的一棵文件名B+树
```

---

## 4. B+树怎么比较中文文件名？

B+树不要求 key 是数字。

它只要求：

```text
key 可以比较大小
```

例如：

```text
abc.txt
你好.jpg
世界.mp4
```

都是字符串 key。

NTFS 使用 Unicode 保存文件名，并根据规定的字符串排序规则进行比较。

所以：

```text
"你好.jpg"
    ↓
字符串比较
    ↓
决定B树往哪个分支走
```

---

## 5. 找到 MFT 500 后怎么办？

目录索引找到：

```text
你好.jpg → MFT 500
```

之后读取：

```text
MFT 500
```

例如：

```text
$FILE_NAME
    你好.jpg
    Parent = MFT 200

$DATA
    文件大小 = 5MB
    data runs = ...
```

如果是大文件：

```text
MFT 500
   ↓
$DATA
   ↓
data runs
   ↓
磁盘 cluster
   ↓
JPG数据
```

如果文件非常小，NTFS 甚至可能直接把数据放在 MFT Record 里。

---

## 6. 为什么 Windows 已知 MFT 后还能找完整路径？

这是 NTFS 很重要的特点。

例如：

```text
D:\资料\项目A\设计稿\首页.psd
```

假设：

```text
首页.psd → MFT 800
设计稿    → MFT 300
项目A     → MFT 200
资料      → MFT 100
```

MFT 800 的 `$FILE_NAME` 中可以有：

```text
Name   = 首页.psd
Parent = MFT 300
```

MFT 300：

```text
Name   = 设计稿
Parent = MFT 200
```

MFT 200：

```text
Name   = 项目A
Parent = MFT 100
```

于是可以向上：

```text
MFT 800 首页.psd
    ↑
MFT 300 设计稿
    ↑
MFT 200 项目A
    ↑
MFT 100 资料
    ↑
根目录
```

反过来拼：

```text
D:\资料\项目A\设计稿\首页.psd
```

所以：

```text
正常打开：
根目录 → 子目录 → 文件
```

而：

```text
已经知道MFT：
文件 → 父目录 → 父目录 → 根目录
```

---

## 7. 只输入“首页.psd”为什么有的软件也能马上找到？

这通常不是因为某个 NTFS 目录 B+树能全盘搜索。

更可能是：

```text
搜索软件自己的全局索引

首页.psd → MFT 800
```

拿到 MFT 800 后：

```text
MFT 800
↓
父 MFT
↓
父 MFT
↓
根目录
```

最终得到完整路径。

所以要区分：

```text
NTFS目录索引
```

和：

```text
Windows Search / 第三方搜索软件的全局索引
```

不是一个东西。

---

## 8. NTFS 可以怎么记？

```text
父目录 MFT
    ↓
目录 B-tree
    ↓
文件名
    ↓
目标 MFT
    ↓
$FILE_NAME
$STANDARD_INFORMATION
$DATA
    ↓
磁盘数据
```

核心特点：

```text
MFT 是中心
```

文件名、父目录关系、文件数据等都通过不同 Attribute 组织起来。

---

## 相关笔记

- [[学习|C++ 学习地图]]
- [[文件系统结构/文件系统|文件系统]]
- [[文件系统结构/Linux ext4：HTree 目录索引|Linux ext4：HTree 目录索引]]
- [[块存储/通过数据库理解块存储的使用方式|通过数据库理解块存储]]
