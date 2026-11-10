
# Sheepdog 项目简介
## 1. Sheepdog 是什么？

Sheepdog 是一个**分布式块存储系统**，主要用于为虚拟机提供虚拟磁盘。

它可以把多台存储服务器组织成一个存储集群，对外提供统一的存储服务。虚拟机使用虚拟磁盘时，可以像使用普通硬盘一样读写数据，而底层的数据由  
Sheepdog 集群负责管理和保存。

简单理解：

> Sheepdog  
> 将多台服务器的存储资源组织起来，为虚拟机提供虚拟磁盘，并负责管理这些虚拟磁盘的数据。

## 2. 核心概念

### 2.1 VDI（Virtual Disk Image）

VDI 是 Sheepdog 管理的虚拟磁盘。每个 VDI  
代表一块独立的逻辑磁盘，具有自己的 ID、容量和元数据。

例如：

- VDI A：80 GB
- VDI B：100 GB
- VDI C：200 GB

这些容量是虚拟磁盘对外呈现的逻辑容量，不代表数据一定连续存放在某一台物理服务器上。

### 2.2 Object

Sheepdog 会把 VDI 的数据空间划分成多个 Object，再对这些 Object  
进行管理和存储。

在本次分析的 Sheepdog 版本中，默认 Data Object 大小为 4 MiB。

例如：

```
VDI
 ├── Object 0
 ├── Object 1
 ├── Object 2
 ├── Object 3
 └── ...
```

因此，虚拟磁盘的一次写入请求，可以根据它在 VDI 中的位置，定位到对应的  
Object 以及 Object 内部的偏移。

### 2.3 Sheep Node

Sheep Node 是运行 Sheepdog 服务端进程 `sheep` 的存储节点。

多个 Sheep Node  
组成存储集群，共同提供存储服务。节点负责接收请求，并执行相应的 Object  
管理和存储操作。

## 3. 项目主要组成

目录 主要作用

---

`sheep/` 核心服务端，处理集群请求、VDI 和 Object 相关操作  
`include/` 定义项目使用的协议、结构体和公共接口  
`lib/` 提供公共功能，例如网络通信和基础数据处理  
`dog/` 集群管理命令行工具  
`sheepfs/` 提供基于 FUSE 的文件系统接口  
`sbd/` 提供块设备相关功能

## 4. Sheepdog 与虚拟机的关系

在虚拟机使用 Sheepdog 虚拟磁盘时，整体关系可以简化为：

```
虚拟机 Guest
    |
    | 对虚拟磁盘进行读写
    v
宿主机 QEMU
    |
    | QEMU Sheepdog Driver
    | 将虚拟磁盘 I/O 转换为 Sheepdog 请求
    v
Sheepdog 存储集群
    |
    | 接收并处理 Object 请求
    v
各个 Sheep Node
    |
    v
物理存储设备
```

这里需要区分项目边界：

- Guest 内部的文件系统负责管理虚拟磁盘内部的文件。
- QEMU 的 Sheepdog Driver 负责把虚拟磁盘 I/O 转换成 Sheepdog  
    能处理的请求。
- Sheepdog 服务端负责处理集群请求、管理  
    Object，并将数据保存到存储节点。

其中，QEMU 的 `block/sheepdog.c` 属于 QEMU 项目，不属于 Sheepdog  
仓库。Sheepdog 仓库主要包含存储集群服务端及其相关组件。

## 5. 本次学习关注的内容

本次主要研究的是：**虚拟机的一次磁盘写入，如何经过 QEMU 的 Sheepdog  
Driver 转换为 Sheepdog 的 Object 写请求，并由远端存储集群处理。**

建议重点关注以下内容：

1. VDI 的元数据和管理方式。
2. VDI 如何映射到多个 Object。
3. QEMU Sheepdog Driver 如何生成 Object 请求。
4. Sheepdog 服务端如何接收和分发请求。
5. Object 数据如何进入实际的存储流程。

暂时不展开 Guest 内部 Linux 文件系统、块设备驱动以及 QEMU  
的底层虚拟化实现。