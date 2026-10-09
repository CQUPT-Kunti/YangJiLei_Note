# Linux ext4

## 1. ext4 的核心文件记录：inode

ext4 的 inode 更接近一个固定结构体。

例如：

```text
/home/tom/你好.jpg
```

假设：

```text
tom       → inode 200
你好.jpg   → inode 500
```

inode 500 可以简化理解成：

```text
inode 500
{
    i_mode
    i_uid
    i_gid

    i_size

    i_atime
    i_mtime
    i_ctime
    i_crtime

    i_links_count

    i_flags

    i_block
}
```

含义：

```text
i_mode
    文件类型 + 权限

i_uid / i_gid
    用户和组

i_size
    文件大小

i_links_count
    有多少硬链接

i_block
    extent tree / 数据块映射
```

---

## 2. inode 里面有没有文件名？

普通文件 inode 通常没有：

```text
name = "你好.jpg"
```

也没有：

```text
parent_inode = 200
```

文件名存在父目录中。

例如：

```text
inode 200
类型：directory
```

它的数据块里面：

```text
.          → inode 200
..         → inode 100
你好.jpg   → inode 500
test.txt   → inode 501
```

所以：

```text
父目录知道孩子是谁
```

而不是：

```text
普通文件inode知道自己的父亲是谁
```

---

## 3. `..` 到底是什么？

对于目录：

```text
.  → 当前目录
.. → 父目录
```

例如：

```text
/home/tom
```

假设：

```text
/home     inode 100
tom       inode 200
```

tom 的目录数据：

```text
.  → inode 200
.. → inode 100
```

所以：

```text
/home/tom/..
```

可以回到：

```text
/home
```

但是：

```text
inode 500（你好.jpg）
```

是普通文件。

它没有：

```text
..
```

也没有：

```text
parent_inode
```

---

## 4. 为什么普通文件 inode 不保存父 inode？

因为 Linux 支持硬链接。

例如：

```text
/home/tom/你好.jpg
/backup/photo.jpg
```

可以同时指向：

```text
inode 500
```

也就是：

```text
tom目录：
你好.jpg ─────┐
              ↓
           inode 500
              ↑
backup目录：  │
photo.jpg ────┘
```

那么 inode 500 到底应该写：

```text
parent = tom
```

还是：

```text
parent = backup
```

没有唯一答案。

甚至文件名也不是唯一：

```text
你好.jpg
photo.jpg
```

都可以是同一个 inode。

---

## 5. ext4 怎么通过文件名找到 inode？

假设已经知道：

```text
/home/tom
```

对应：

```text
inode 200
```

现在找：

```text
你好.jpg
```

如果目录很小，可以直接扫描 directory entry。

如果目录很大，ext4 可以使用：

```text
HTree
Hashed B-tree
```

过程：

```text
已知父目录 inode 200
        ↓
hash("你好.jpg")
        ↓
HTree
        ↓
找到对应目录块
        ↓
你好.jpg → inode 500
```

注意：

HTree 找到的最终还是 directory entry：

```text
名字 → inode
```

---

## 6. 找到 inode 500 后怎么办？

读取 inode 500：

```text
inode 500

类型：普通文件
大小：5MB

i_block
   ↓
extent tree
```

例如：

```text
逻辑 block 0~1279
        ↓
物理 block 80000~81279
```

所以：

```text
inode 500
   ↓
extent
   ↓
磁盘 block
   ↓
JPG数据
```

因此：

```text
HTree：
解决“这个名字对应哪个inode”
```

```text
inode：
解决“这个文件是什么”
```

```text
extent：
解决“这个文件的数据在哪里”
```

---

## 7. 如果我站在 `/`，只搜索一个很深的文件呢？

例如实际文件是：

```text
/a/b/c/d/e/你好.jpg
```

但是你只知道：

```text
你好.jpg
```

ext4 没有一个：

```text
整个文件系统：
你好.jpg → inode 500
```

这样的全局 HTree。

每一个目录有自己的索引：

```text
/
自己的HTree

/a
自己的HTree

/a/b
自己的HTree
```

所以类似：

```bash
find / -name "你好.jpg"
```

需要递归遍历目录。

大概：

```text
/
↓
扫描子目录
↓
进入 /a
↓
扫描
↓
进入 /a/b
↓
...
↓
找到 你好.jpg
```

---

## 8. 那为什么 Linux 搜索软件可以瞬间找到完整路径？

因为它可能自己建立了：

```text
全局搜索数据库
```

例如：

```text
你好.jpg → /a/b/c/d/e/你好.jpg
```

这不是 ext4 HTree。

要区分：

```text
ext4 HTree：
父目录 + 文件名 → inode
```

和：

```text
搜索软件索引：
文件名 → 全局路径
```

---

## 9. ext4 可以怎么记？

```text
父目录 inode
      ↓
HTree / directory entry
      ↓
"你好.jpg" → inode 500
      ↓
inode 500
      ↓
extent tree
      ↓
磁盘 block
```

最核心的一句话：

> ext4 中，目录项负责“名字 → inode”，inode 负责文件自身的元数据和数据位置。

而普通文件 inode 通常既不知道自己的文件名，也不知道唯一的父目录。

---

## 相关笔记

- [[学习|C++ 学习地图]]
- [[文件系统结构/文件系统|文件系统]]
- [[文件系统结构/Windows NTFS：B+树目录索引|Windows NTFS：B+树目录索引]]
- [[块存储/通过数据库理解块存储的使用方式|通过数据库理解块存储]]
