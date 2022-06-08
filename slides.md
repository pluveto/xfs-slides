---
theme: purplin
position: center
highlighter: shiki
lineNumbers: false
layout: intro

---


# 深入理解 XFS 文件系统

高度可扩展、高性能、现代化

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
    开始 <carbon:arrow-right class="inline"/>
  </span>
</div>

<div class="abs-br m-6 flex gap-2">
  <button @click="$slidev.nav.openInEditor()" title="Open in Editor" class="text-xl icon-btn opacity-50 !border-none !hover:text-white">
    <carbon:edit />
  </button>
  <a href="https://github.com/slidevjs/slidev" target="_blank" alt="GitHub"
    class="text-xl icon-btn opacity-50 !border-none !hover:text-white">
    <carbon-logo-github />
  </a>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->


---
layout: image-x
imageOrder: 2
image: https://raw.githubusercontent.com/pluveto/0images/master/2022/06/upgit_20220608_1654682585.png
---
# 为什么要研究 XFS？

+ 文件系统是数据可靠性软件层面的基石
+ 上古文件系统：FAT、EXT2、EXT3 结构简单但难以适应现代需要
+ 技术基础：并行读写、日志事务、延迟分配、空闲空间管理、反向索引
  + 我们能学习到计算机领域的核心工程技术
+ 面向未来：设计分布式文件系统、数据库存储引擎时的参考

XFS 目前已经是 CentOS 7 的默认文件系统。

---

# XFS 的整体架构

+ On-Disk / In-memory Structure（磁盘上/内存中结构）
+ Metadata Integrity（元数据完整性）
+ Delayed Logging（延迟日志）
+ Sharing Data Blocks（共享数据块）

---

# 我们的重点

+ XFS 的磁盘结构组织
  + 整体 On-Disk 布局
  + B+tree 的空闲空间管理机制
  + B+tree + Extend 的数据管理机制
  + inode 的实现、分配和管理机制

+ 日志、事务化文件系统的实现
  + 事务管理
  + journals/logs 的 in-core log buffer -> on-disk log

---

# 磁盘块抽象

“九层之台，起于累土”

在错综复杂的文件系统最底层，所有操作都可以归结为两个简单的物理块读写操作：

```rust
pub trait BlockDevice: Send + Sync {
    fn read_block(&self, block_id: usize, buf: &mut [u8]) -> bool;
    fn write_block(&self, block_id: usize, buf: &[u8]) -> bool;
```

如果需要**任意读取**，即从随机位置，读取随即长度，则需要计算出读取的块号，**完整读取**后选出想要的部分。
**任意写入**也是同理，需要计算出写入的块号，然后先读出块中内容到缓冲，**在缓冲完成合并**，然后完整写入磁盘块。因此，是否对齐，对齐的参数，对读写性能会产生重要的影响。

> 附注：几个基本概念
> 
> + 物理扇区：硬盘的最小存储单元，常见大小：512 字节、4096 字节
> + 逻辑扇区：即通常意义上的扇区。出于兼容性考虑，硬盘芯片会将物理扇区统一为 512 字节的逻辑扇区，这样对于 OS 而言，扇区大小> 好像是 512 字节，但实际上硬盘内部的存储单元是 4096 字节。当然，多数情况下物理扇区大小等于逻辑扇区大小。
> + 物理块：一般认为就是逻辑扇区，也称磁盘块。
> + 逻辑块：格式化磁盘时设置的就**文件系统而言**的分配单元，用户可以自由设置，是物理块的整数倍
> 
> 多数情况下，我们接触到的上述四个值都是 512B 为单位。

---

# AG (Allocation Group)

XFS 将一个分区（也可以是整块磁盘）分成多个 Group，对于不同 AG 发起的 IO 可以并行执行。

第一个 AG 称为 Primary AG.

![upgit_20220608_1654684291.png](https://raw.githubusercontent.com/pluveto/0images/master/2022/06/upgit_20220608_1654684291.png)

类似 AG 的抽象在传统文件系统中是没有的。这是磁盘容量的飞涨（数 TB 已成为常态）、磁盘实现（如固态硬盘）的演进的必然选择。除了更便于并发外，AG 的抽象使得 XFS 可以使用 AG 内相对位置进行地址表示，有 LHDR、SHDR 两种块头结构，进一步减小了元数据对空间的占用，也提高了文件系统的地址表示能力。

XFS 的单分区最大容量约 8EB（相当于 8 x 1024 x 1024 TB)


---
layout: image-x
image: https://raw.githubusercontent.com/pluveto/0images/master/2022/06/upgit_20220608_1654687979.png
---

## 单 AG 结构

<img src="" alt="">

+ SB - Superblock
+ AGF - AG free block info. 存放四个指针
  + BNO root
  + CNT root
  + RMAP root
  + REFC root
+ AGFL
+ AGI
+ BNO root block
+ CNT root block
+ INO root block
+ FINO root block
+ RMAP root block
+ NULL terminating block

---

## AG / Superblock （上）

Superblock（SB）占用一个扇区的大小。存储结构如下：

```rust
pub struct SuperBlock {
    pub magicnum: u32,  // 魔数
    pub blocksize: u32, // 逻辑块大小 通常是4096字节（4KB）
    pub dblocks: u32,   // 数据块数
    pub rblocks: u32,   // 实时块数 https://www.cnblogs.com/orange-CC/p/12711078.html
    pub rextents: u32,  // 实时扩展数
    pub uuid: UUID,     // 唯一标识符
    logstart: u32,      // 日志起始块号
    rootino: u32,       // 根目录Inode号
    rbmino: u32,        // 实时块位图Inode号
    rextsize: u32,      // 实时扩展块大小
    agblocks: u32,      // 块组大小 最后一个AG的实际大小可能会不一样
    agcount: u32,       // 块组数
    rbmblocks: u32,     // 实时块位图块数
    logblocks: u32,     // 日志块数
    version: u16,       // 版本号
    sectsize: u16,      // 扇区大小 bytes
    inodesize: u16,     // Inode大小 bytes
    inopblock: u16,     // Inode per block
```

---

## AG / Superblock （下）

可以看到，AG0 的 Superblock 包含了整个文件系统的一些关键信息。其中比较值得注意的是几个指针：

+ logstart：日志起始块号
+ rootino：根目录Inode号

> 注：
> + r 开头的属性属于实时设备，这是因为 XFS 兼容了实时模式的 Linux. 此处暂时略过。
> + 磁盘上的指针，都是指块号，不要与内存中的指针相混淆。

```rust
    fsname: [u8; 16],   // 文件系统名称
    blocksize_bits: u8, // 块大小位数，即以2为底的对数
    sectsize_bits: u8,  // 扇区大小位数，即以2为底的对数
    inodesize_bits: u8, // Inode大小位数，即以2为底的对数
    inpblock_bits: u8,  // Inode per block位数，即以2为底的对数
    agblocks_bits: u8,  // 块组大小位数，即以2为底的对数
    rextents_bits: u8,  // 实时扩展块大小位数，即以2为底的对数
    inprogress: u8,     // 是否正在进行格式化
    imax_pct: u8,       // 最大Inode百分比
    icount: u64,        // Inode总数
    ifree: u64,         // 空闲Inode总数
    fdblocks: u64,      // 空闲数据块总数
    frextents: u64,     // 空闲实时扩展总数
}
```

---
class: text-center
---

Superblock 是文件系统最重要的结构。

<hr> 

由于 XFS 的体系过于复杂，若是线性叙事，则没有几十个小时，是无法说清楚的。

**我们将从一个文件/文件夹的如何被创建的角度，解析 XFS 的结构。**
---

## 空闲 inode 管理

当创建一个新文件时，首先要分配一个空闲的 Inode，然后再分配一个空闲的数据块。

而要分配 inode，首先检查 AGI 的 agi_freecount，确定是否有空闲的 Inode。

如果有，则从 AGF 可以找到 FINO B+tree 的 root 块。（AGI 中，存有 root 字段，指向 finobt（空闲 inode 索引 b+tree））

root 是一个 B+ 树块. 其结点结构如下：

```rust
pub struct BtreeBlock {
    magicnum: u32, // B+树块的 Magic Number AB3B
    level: u32,   // B+树块的层级，如果level是0，表示当前的 block 存储的都是叶子节点，否则是索引节点。越靠近根部越大
    numrecs: u32, // 如果是叶子节点，表示当前 B+树块中的记录数。否则表示有多少子节点（AllocRec）。
    header: BtreeBlockUnion,
}
```


> 注：为什么会有空闲的 inode?
> 当分配 inode 时，会一次性分配整个 chunk 的 inode，数量为 64 个（取决于 XFS_INODES_PER_CHUNK 常量）。
> 如果只用到一个，则剩余 63 个 inode 会被索引至空闲表。


---

## B+tree blk 的通用结构

在介绍空闲 inode 的定位之前，有必要介绍一下 XFS 的 B+tree 是如何组织起来的。以 BtreeBlockShdr 为 header 的 B+ 树块为例，一个 B+tree 块可以分为以下部分：

![upgit_20220608_1654693240.png](https://raw.githubusercontent.com/pluveto/0images/master/2022/06/upgit_20220608_1654693240.png)

```rust
#[derive(Copy, Clone)]
struct BtreeBlockShdr {
    leftSibling: u32,  // 左兄弟节点的块号
    rightSibling: u32, // 右兄弟节点的块号

    blkno: u64, // 当前节点的块号。字节数/512 可得之
    lsn: u64,   // 最后写入节点的日志 SN（序列号）
    uuid: UUID, // 当前节点的UUID
    owner: u32, // 当前节点的所有者（即哪个 ag）
    crc: u32,   // 本块的 CRC 校验值
}
```

## 空闲 inode 的定位

从 B+tree 根节点开始，查询 recs，可按照 startino 定位到一个空闲的 inode，并且知道具体的空闲详情。这是因为 inode 的 Records （同时存在于叶节点和中间节点）结构如下：

<img src="https://raw.githubusercontent.com/pluveto/0images/master/2022/06/upgit_20220608_1654693473.png" style="max-height:240px;" >

```rust
pub struct InodeBtreeRecord {
    startino: u32, // 一个 inode chunk 里 inode num 最小的那 inode，也就是这个 chunk 的起始 inode
    holemask: u16, // 空洞掩码。稀疏 inode 允许以小于 chunk 的大小分配 inodes，从而 chunk 中有的位置需要跳过。
    // 16 位，每位代表 4 个连续 inode 空洞。
    count: u8, // 表示一共有多少已分配的 inode，当未启用 sparse 时为 64，但若启用了 sparse，则为 64 - 4 * n(holemask)
    freecount: u8, // 当前记录的空余 inode 数量（已分配但尚未使用的 inode 数量）
    free: u64, // finode 位图，64 位对应 inode chunk 里的 inode 的空闲情况。1 表示可用。
}
```


---

## （全新）inode 的分配

在小概率情况下，我们无法找到一个空闲的 inode。此时会进行 inode 分配。方法是通过空闲空间管理器，定位到一个空闲的逻辑块，然后将这个逻辑块作为 chunk，分配 64 个 inode 于其中，并利用其一。

使用后，将已用的，纳入 INO Btree 索引，将未用的，纳入 FINO Btree 索引。

那么，怎么就完成分配了呢？实际上，无论是分配 inode chunk，还是给刚创建的文件分配数据块，都需要用到**空闲空间管理器**。因此接下来详细介绍。

顺带一提，XFS 中 inode 的命名规则为：AGno + blockno + block offset

这样，我们无需对 inode 再进行一道索引，直接根据 inode 编号就能定位到数据位置。非常巧妙。

---

## 空闲空间管理概览

试想，我们如果要创建一个 100KB 的文件，那么最好的方式便是直接分配一个 100KB 的连续物理块。

但现实中，为了充分利用空间，我们不得不把一个文件的物理块分散到几个地方。

（当然，对于应用程序而言，他们看到的是逻辑线性的文件空间。）

因此，有必要设计一个结构。既要满足充分利用空间的要求，又要能够快速定位到空闲的物理块。

空闲空间同样通过 B+tree 来管理，但是它的记录结构略有不同。

> 注：结点头和 FINOB+tree 基本相同，因此略过。


---

## 用于空闲空间管理时的 Records (内部节点)

内部结点用于快速定位到目标结点。有两种索引方式，BNO 索引和 CNT 索引。

<img src="https://raw.githubusercontent.com/pluveto/0images/master/2022/06/upgit_20220608_1654694396.png" style="max-height:250px;" >

```rust
pub struct AllocRec {
    startblock: u32, // 当前记录的起始块号
    blockcount: u32, // 当前记录的块数
}
```
从宏观上看，相当于建立了两个映射，分别可以用来定位块号和定位空闲空间。
---


## 利用 CNT B+tree 定位空闲空间

第一步，AGF 的 roots 字段，分别存放了 bnoroot、cntroot、rmaproot（注：rmaproot 是对已用空间的倒排索引）

```rust
pub struct AGF {
    magicnum: u32,   // AG扇区的 Magic Number
    // 此处略去若干字段
    /**
     * 从下标 0 开始，依次是
     * bnoroot  表示“以块号为索引的 FS B+tree 根节点”的块号
     * cntroot  表示“以块数为索引的 FS B+tree 根节点”的块号
     * rmaproot 表示“表示对已用空间的倒排索引 B+tree 根节点”的块号
     */
    roots: [u32; XFS_BTNUM_AGF], // 用来管理空间所需的几个结构的头地址
    spare0: u32,     // 有意留空的字段
    levels: [u32; XFS_BTNUM_AGF], // B+树的层级或深度，与agf_roots一一对应。对于新创建的AG，三个 level 都是 1
    spare1: u32,                  // 占位

    flfirst: u32, // FL(空闲链表)块的开始位置
    fllast: u32,  // FL 块的结束位置。
    flcount: u32, // FL 中的块数量。
    // 此处略去若干字段
}
```

---

## 利用 CNT B+tree 定位空闲空间

第二步，根据 cntroot 的块号，可以找到 CNT B+tree 的根节点。从而找到第一级索引。

第三步，根据各级索引的 blockcount 字段，可以找到空闲空间的起始块号。

接下来就可以使用该空间空间。

---

## 插播：空间碎片化问题

由于 XFS 使用 FINOB+tree 来管理空间，虽然延迟分配使其碎片问题优于其它文件系统，但原理上决定了仍然存在空间碎片化的问题。startblock 和 blockcount 的记录对会增长。导致 b+tree 需要扩容。

为了满足扩容需求，mkfs 时每个 AG 预留出 4 个 block 的 Free List 区间，并将对应的 block number 记录到 internal Free List 集合 (每个 AG 的第四个 sector)。

internal Free List 内部是采用 ring buffer 的方式对 block 做管理，细节比较繁杂，简单来说，若 b+tree 需要扩容，则需要对叶子节点进行拆分，则可以从 freelist 中取出空闲的 block 进行使用。freelist 用满后，可执行扩容。

若碎片问题影响仍然存在，则可通过 xfs_fsr 进行碎片整理。

---

## inode 的初始化

inodesize 默认为 512 字节，由三部分构成。

<img src="https://raw.githubusercontent.com/pluveto/0images/master/2022/06/upgit_20220608_1654697411.png" style="max-height: 130px">

比较有趣的一点是 datafork 和 xattrfork 是动态大小的，通过 forkoff 给定的值分界。

下面介绍一下这三部分。

---

## inode core

下面是 inode core 的结构（D前缀是因为此结构存放于磁盘之中），由于此结构非常长，仅摘取最为关键的部分列出。文件和目录共享使用的 inode 结构并无特异之处。

```rust
pub struct Dinode {   
    mode: u16,         // st_mode 权限位。前3个8进制为表示文件的类型等，比如普通文件 S_IFREG
                       // 常见的有 S_IFBLK, S_IFCHR, S_IFDIR, S_IFIFO, S_IFLNK。后三位就是文件的权限如 rwxrwxrwx。
    format: u8,        // 格式，常见有 FMT_LOCAL, FMT_EXTENTS, FMT_BTREE. FMT_DEV 用于字符或块设备
    uid: u32,          // 文件所有人
    gid: u32,          // 文件所属组
    nlink: u32,        // 硬链接计数
    atime: timestamp,  // 最后访问时间
    mtime: timestamp,  // 最后修改时间
    ctime: timestamp,  // 最后 inode 状态修改时间
    size: u64, // Inode的大小，对于文件inode来说并不是其实际占用空间的大小，而是看EOF的位置。对于目录inode来说就是目录条目所占的空间。
    nblocks: u64, // 统计此 inode 占用的文件系统块数。
    forkoff: u8, // datafork 和 attrfork 的分界线。乘以8等到真正的偏移字节量
    flags: InodeFlags, // inode 标记
    /* di_next_unlinked is the only non-core field in the old dinode */
    next_unlinked: u32, // 之前提到 unlinked 哈希表，记录已经 unlink 但仍然被引用的 inode。此处是该哈希表的拉链。
    changecount: u64, // inode 的 i_version，每次修改 inode 时加 1
    /* fields only written to during inode creation */
    crtime: timestamp, // 创建时间
}

```

---

## data fork (普通文件)


存放文件的数据。xfs 有两种模式，一种是 extent 模式，一种是 b+tree 模式。本质而言，它们都充当 Bmap 的作用（注意：bmap 不是位图，而是块映射）

**extent 模式**:一个 extend 长 128B，由 flag, 逻辑偏移, 物理块号，物理块数几部分组成。**对于简单的文件，并不需要用 b+tree 去存储，只要用数个 extend 就够了**（这实际上就是我们初学操作系统时了解到的模式，直接在 inode 中记录文件的物理块位置）。这些 extend 在具体读写文件时都会被转换为物理块位置去读写。（和 inode 一样，一个 extend 也可能处于预分配未写入状态）

**B+tree 模式**：当 extents 不能满足需求，这种情况下，extents 会被销毁，用一棵树取而代之。原来 extents 的地方变成了一个 b+tree 树根。

内部节点和设计和之前所述大同小异。而叶节点实际上就是 extents.

**类似我们初学 OS 时了解的多级索引，但这种索引是通过 b+tree 实现的，而且可以任意级别。**

---

## data fork（目录）

> 目录块与FS块有所不同。大家只要知道目录有自己单独的 directory blocksize 就行了。

+ **短目录**：对简单的目录，直接在 datafork 存目录信息。包括目录头和目录项。不单独存储 `.` 和 `..`
+ **块目录**：当短目录不够用（datafork 满）则需要变成块目录。块目录信息用单独的 dir block，虽然是线性，但通过哈希表 leaf entry 存放。索引起来很快。
+ **叶目录**：块目录的微小改进，多个 data blocks 来存储 data entries，并将 leaf entries 转移到一个单独的 leaf block 中。
+ **节点目录**：在页目录的基础上，增加对叶块的索引（free index block）。
+ **B+tree 目录**：在节点目录的基础上，增加多层 node block。

---

## data fork（目录）


```rust
pub struct DirShortFormatHeader {
    count: u16,      // 此目录 项的个数
    i8count: u8,     // 表示有多少目录的条目是用于64位inode的。
    parent: [u8; 8], // 父目录的inode号
}
// （短）目录项：
pub struct DirShortFormatEntry {
    namelen: u8,     // 文件名长度
    offset: [u8; 2], // 偏移，用于辅助在readdir的时候迭代目录内容用的。
    name: [u8],      // 文件名，flexible array
}
```
> 文件名、目录名是通过弹性数组存放的。因此不同于一些早期文件系统，它们只能采用固定长度的目录名。而 XFS 的目录名几乎可以任意长度。

## datafork （符号链接）

同学们有没有好奇符号链接文件是怎么实现的？XFS 符号链接有两种：
+ **local symlink**：即直接在 datafork 进行连接。内容就是路径。
+ **extent symlink**：这是考虑到链接路径过长，datafork 无法容纳，则采用单独的 symlink block 来存放，原先 datafork 改存索引。

符号链接没有 b+tree 组织形式。我猜是考虑到不可能有那么长的目标文件名。

---

## attr fork

attr fork 的作用是存放或索引扩展属性。扩展属性相当于给文件加自定义字段。例如 acl 和 selinux 等都是基于该特性的。如果你用过 setxattr getxattr 之类的 syscall，你应该会很熟悉它。其内容简单而言就是 key-val 对。

为了支持变长，使用 namelen, valuelen 记录 key-val 对的长度。

和目录一样，支持 local, extent 和 btree 三种组织形式。大同小异，因此略过。

---

# 文件系统日志

Log-structured File System

LFS 提高读写性能的基本思想：在内存中设立一个缓冲队列，把写操作缓冲到队列中，每隔一段时间，被缓冲在内存中的所有未决的写操作都被存到磁盘中。

LFS 日志结构的基本思想：记录系统下一步将要做什么的日志。这样当系统在完成它们即将完成的任务前崩溃时，重新启动后，可以通过查看日志，获取崩溃前计划完成的任务，并完成它们。

---

# XFS 日志结构

XFS 日志被分为一系列日志记录（log record，LR）。每个日志记录都有一个日志记录头（log record header）

xlog_rec_header 摘录如下：

```c

typedef struct xlog_rec_header {
	__be32	  h_cycle; // 所属写周期
	__be32	  h_len;   // 字节长度，64 bit 对其
	__be64	  h_lsn;   // 此 LR 序列号
	__be64	  h_tail_lsn; // 第一个未提交的写 buffer LR 的序列号
	__be32	  h_prev_block; // 上一个 LR 的块号
	__be32	  h_num_logops; // 此 LR 的操作个数
	__be32	  h_cycle_data[XLOG_HEADER_CYCLE_SIZE / BBSIZE]; // 循环数据
  // ...
} xlog_rec_header_t;
```

---

而一个日志记录头之后，又包含多个日志操作(log op)头。每个 log op 后紧跟 data region.

```c
typedef struct xlog_op_header {
__be32 oh_tid; // 此操作的事务 ID
__be32 oh_len; // 数据域长度
__u8 oh_clientid; // 操作来源
__u8 oh_flags; // 状态位
__u16 oh_res2; // 填充
} xlog_op_header_t;
```

值得一提的是 oh_flags，包括：

+ `XLOG_START_TRANS` 开始一个新事务。则下一个操作是事务头描述。
+ `XLOG_COMMIT_TRANS` 提交事务
+ `XLOG_CONTINUE_TRANS` 中继此事务到另一个 log record
+ `XLOG_WAS_CONT_TRANS` 表示事务开始于前一个 log record
+ `XLOG_END_TRANS` 事务结束
+ `XLOG_UNMOUNT_TRANS` 卸载文件系统

---

# 事务头和日志项，以及日志原理

在操作头之后，可以跟多种类型的负载。这些负载通过 th_type 区分。

```c
typedef struct xfs_trans_header {
	uint		th_magic; uint		th_type;		// 事务类型
	int32_t		th_tid;		// 事务 id
	uint		th_num_items;		// 事务日志记录项目数
} xfs_trans_header_t;
```

每个 log item 有如下类型（仅摘录比较重要的）：

```
XFS_LI_EFI        Extent 释放（intent）
XFS_LI_EFD        Extent 释放（done）
XFS_LI_INODE      Inode 更新
XFS_LI_ICREATE    Inode 创建
XFS_LI_RUI        反向索引更新（intent）
XFS_LI_RUD        反向索引更新（done）
XFS_LI_CUI        引用计数更新（intent）
XFS_LI_CUD        引用计数更新（done）
XFS_LI_BUI        文件块映射更新（intent）
XFS_LI_BUD        文件块映射更新（done）
```
可以看到，基本上就是两阶段提交协议。
---

## 总结

XFS 通过 AG 组织分区，通过 SB 和 AGF、AGI、AGFL 架构起 AG 的结构。XFS 贯穿始终思路的就是索引。抛弃了位图方案，并重度使用 b+tree 索引。就全局而言：

+ BNO 空闲空间块号索引
+ CNT 空闲空间块数索引
+ INO 索引节点号索引
+ FINO 空闲索引节点号索引

就一个 inode 而言： datafork/attrfork 内使用混杂索引。其索引设计思想基本遵循这样的思路： local/extent/
leaf/b+tree

在日志结构化方面，通过 log - log_record - log_op - log_item 的层次，基于两阶段提交来实现多层次的日志。

在性能优化方面，利用日志化、并行 IO、延迟分配等，实现了高性能。实现了 Reflink，能够高速拷贝文件。

---

## 参考文献

*XFS Algorithms & Data Structures 3rd Edition*

*XFS in Linux source* https://github.com/torvalds/linux/tree/master/fs/xfs

*Related articles* https://lwn.net/
 