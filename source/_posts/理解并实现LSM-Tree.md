---
title: 理解并实现LSM Tree
date: 2022-07-26 23:37:49
categories: 存储
tags: [分布式, LSM Tree, 存储]
---

# 1 背景

对于存储介质为磁盘或SSD的数据库，长期以来主流使用B+树这种索引结构来实现快速数据查找。当数据量不太大时，B+树读写性能表现非常好。但是在海量数据情况下，B+树越来越高，由于B+树更新和删除数据时需要沿着B+树逐层进行页分裂和页合并，严重影响数据写入性能。为了应对这种情况，google在论文`《Bigtable: A Distributed Storage System for Structured Data》`中介绍了一种新的数据组织结构`LSM Tree(Log-Structured Merge Tree)`，随后，`Bigtable`主要作者`Jeffrey Dean`和 `Sanjay Ghemawat`开源了一款基于LSM Tree实现的数据库LevelDB，让大家对LSM Tree的思想和实现理解得更为透彻、深入。当前，比较流行的NoSQL数据库，如Cassandra、RocksDB、HBase、LevelDB等，newSQL数据库，如TiDB，均是使用LSM Tree来组织磁盘数据的。

<!-- more -->
<!-- markdownlint-disable MD041 MD002--> 

# 2 基本原理

LSM Tree(Log-Structured Merge Tre)中文名为：日志结构合并树。其实这并不是一种具体的数据结构，更多是一种数据结构的设计思想，是一个分层、有序、针对块存储设备（机械硬盘和SSD）特点而设计的数据存储结构。它的核心理论基础还是磁盘的顺序写速度比随机写速度快非常多，即使是SSD，由于块擦除和垃圾回收的影响，顺序写速度还是比随机写速度快很多。如下图所示：

<img src="%E7%90%86%E8%A7%A3%E5%B9%B6%E5%AE%9E%E7%8E%B0LSM-Tree/image-20220726235834566.png" alt="image-20220726235834566" style="zoom:50%;" />

LSM Tree将要存储的数据格式化为多个**SSTable（Sorted String Table）**，一个`SSTable`内的数据是有序的任意字节组（arbitrary byte string），而且，SSTable写入磁盘后是不可修改的（Log-Structured）。如有更新数据的需求时，LSM Tree不修改旧数据，直接将新的数据写入新的SSTable中。若有删除需求时，而是将删除的标记写入SSTable中，等到后面合并的时候直接将该数据删除。所以LSM Tree的磁盘写入单位为SSTable，即对磁盘的操作都是顺序的块写入，没有随机写入操作。从而将磁盘的写性能尽可能发挥出来。

`LSM Tree`这种独特的写入方式，导致在查找数据时，`LSM Tree`就不能像`B+树`那样在一个统一的索引表中进行查找，而是从最新的`SSTable`到老的`SSTable`依次进行查找。如果在新`SSTable`中找到了需查找的数据或相应的删除标记，则直接返回查找结果；如果没有找到，再到老的`SSTable`中进行查找，直到最老的`SSTable`查找完。为了提高查找效率，`LSM Tree`对`SSTable`进行分层、有序组织，也就是说把`SSTable`组织成多层，同一层可以有多个`SSTable`，同一个数据在同一层的多个`SSTable`中可以不重复，而且数据可以做到在同一层中是有序的，即每一个`SSTable`内的数据是有序的，前一个`SSTable`的最大数据值小于后一个`SSTable`的最小数据值。这样可以加快在同一层`SSTable`中的数据查询速度。同时，`LSM Tree`会将多个`SSTable`合并（`Compact`）为一个新的`SSTable`，这样可以减少`SSTable`的数量，同时把修改前的数据或删除的数据真正从`SSTable`中删除，减小了`SSTable`的大小（Merge Tree），以提高查找性能。

# 3 整体框架

<img src="%E7%90%86%E8%A7%A3%E5%B9%B6%E5%AE%9E%E7%8E%B0LSM-Tree/image-20220727210426468.png" alt="image-20220727210426468" style="zoom: 50%;" />

WAL（Write Ahead Log）实际上为数据库的日志结构文件，以Append-Only方式追加记录到日志文件，并不属于LSM Tree。在实际的系统实践中，WAL为系统不可或缺的部分，将WAL囊括进来，才能从整体上更准确地理解LSM Tree。

如上图所示，`LSM Tree`的数据由两部分组成：内存部分和持久到磁盘中的部分。内存部分由一个`MemTable`和一个或多个`Immutable MemTable`组成。磁盘中的部分由分布在多个level的`SSTable`组成。level级数越小（`level 0`）表示处于该level的`SSTable`越新，level级数越大（`level 1...level N`）表示处于该level的`SSTable`越老，最大级数（`level N`）大小由系统设定。在本图中，磁盘中最小的级数为`level 0`，也有的系统把内存中的`Immutable MemTable`定义为`level 0`，而磁盘中的数据从`Level 1`开始，这只是level定义的不同，并不影响系统的工作流程和对系统的理解。

其中WAL可以保证当系统崩溃重启后，内存中MemTable和Immutable MemTable中未持久化到磁盘中SSTable的数据不会丢失。

内存中MemTable一般为有序数据结构，实现方式有很多种，如有序数组、红黑树、二叉搜索树，比较常见的方式为跳表，跳表支持高效的动态插入数据，对数据进行排序，也能高效地对数据进行精确查询与范围查询。

`SSTable`一般由一组数据block和一组元数据block组成。元数据block存储了`SSTable`数据block的描述信息，如索引、BloomFilter、压缩、统计等信息。因为`SSTable`是不可更改的，且是有序的，索引往往采用二分法数组结构就可以了。为了节省存储空间和提升数据块block的读写效率，可以对数据block进行压缩。

## 数据写入过程

<img src="%E7%90%86%E8%A7%A3%E5%B9%B6%E5%AE%9E%E7%8E%B0LSM-Tree/image-20220727212404732.png" alt="image-20220727212404732" style="zoom:50%;" />

`LSM Tree`写入数据时，先写一条记录到`WAL`中，然后将数据写入内存中的`MemTable`中，这样写入操作就完成了。（写入WAL与MemTable可以以事务的形式实现，两者成功才算成功）

写`MemTable`时，写入新的数据，与修改现有数据的部分字段以及修改现有数据的所有字段，写入操作过程几乎是一样的，都是把传入的数据（合并）写入到`MemTable`中。删除数据时，则是在`MemTable`中写入一条删除标记。当`MemTable`的大小达到设定的大小（经典值是64KB）时，`LSM Tree`会把当前`MemTable`设置为一个不可修改的`Immutable MemTable`，然后创建一个新的`MemTable`供新的数据写入。同时，`LSM Tree`一般会有一些与写入线程（或进程）相独立的背景线程（或进程）负责将`Immutable MemTable` flush到磁盘中，将数据持久化。完成持久化的Immutable MemTable对应的`WAL`就可以从磁盘中删除了。而内存中`Immutable MemTable`数量的多少取决于`Immutable MemTable` flush的速度与`Immutable MemTable`生成的速度（数据写入速度）的差值。

从上面介绍的写入过程可以知道，LSM Tree的写入过程非常简单，只需要写WAL与MemTable即可。其中WAL是Appand-Only的方式在日志结构文件末尾追加的记录（顺序写），无需其余复杂的操作，写入性能非常好。而MemTable虽然需要如跳表的插入和排序，但是都是在内存中的操作，并且MemTable的大小也有限（64K），所以MemTable的写入性能也非常好。

在LSM Tree的结构中，数据的写入性能与系统中的数据总量和数据更新类型（修改还是删除数据）都无关，写入速度比较稳定。

### 空间放大

如果同一个数据被多次更新，那这个数据可能存储在多个SSTable当中，甚至统一数据的不同部分的最新数据内容存储在不同的SSTable中（数据部分更新的场景），因而同一数据在磁盘当中有多份副本，且老的副本已经过时，这导致了数据实际占用的存储空间大于数据的实际大小。这一现象叫做空间放大（space amplification）

## 数据查找过程

<img src="%E7%90%86%E8%A7%A3%E5%B9%B6%E5%AE%9E%E7%8E%B0LSM-Tree/image-20220727233111681.png" alt="image-20220727233111681" style="zoom:50%;" />

因为[空间放大](#空间放大)的问题，所以LSM Tree的数据查找过程如上图所示：先在内存`MemTable`中查找，然后在内存中的`Immutable MemTable`中查找，然后在`level 0 SSTable`中查找，最后在`level N SSTable`中查找。其中SSTable的查找顺序也为从新到老，直到在某个（或某些个）`SSTable`中查找到了所需的数据，或者最老的`SSTable`查找完也没有找到需要的数据。

查找某个具体的SSTable时，先把SSTable的元数据block读到内存中，根据BloomFilter可以快速确定数据在当前SSTable中是否存在，如果存在，则采用二分法确定数据在哪个数据block，然后将相应数据block读到内存中进行精确查找。若不存在，直接跳过，查找下一个更早的SSTable，重复以上步骤。

### 读放大

由于多层SSTable结果的存在，查询数据时需要遍历多个可能不包含数据的SSTable（最坏的情况需要读取所有的SSTable）。这种现象叫做读放大（read amplification）

### 读放大优化

读放大问题严重影响了LSM Tree的数据查找性能，特别是当数据不存在或者在非常老的SSTable中。在著名的Google三驾马车之一的论文[BigTable](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf)提到了几种提升数据查找性能的方法：

1. 压缩（**Compression**）：将SSTable的block进行压缩，降低block的磁盘空间占用，也能减少磁盘的读写时间，但是需要消耗CPU的性能来进行block的压缩和解压缩。
2. 布隆过滤器（**BloomFilter**）：BloomFilter可以快速确定数据不在SSTable中，而不用读取数据block内容。
3. 缓存（**Cache**）：SSTable是不可变的，非常适合缓存。可以将热点数据缓存到到内存中，尽量避免访问磁盘。
4. SSTable合并（**Compaction**）：将多个SSTable合并为一个SSTable，删除旧数据或物理删除已经被删除的数据，降低空间放大；同时减少SSTable数量，降低读放大。

## SSTable合并

### 写放大

SSTable合是将多SSTable合并成一个新的SSTable写入下一个Level，合并过程中，会删除过期重叠数据，并将标记为删除的数据从磁盘中删除，减少了SSTable的数据，可以有效缓解读放大和空间放大问题。这个过程需要将所有涉及合并的SSTable都读取到内存当中，并将合并的新SSTable写入到磁盘，这样会消耗大量的 CPU 时间和磁盘 IO。这种现象叫写放大（write amplification）

SSTable合并分为`minor compaction`和`major compaction`

- Minor Compaction：是指将内存中的`Immutable MemTable`内容flush到磁盘中形成SSTable时进行的数据处理过程
- Major Compaction:是指相邻的两级level将数字小level（如level 0）的SSTable合并入数字大level（如level 1）的SSTable中的过程

SSTable合并其实就是在空间放大、写放大、读放大几个相互制约的因素间寻求平衡，不同的应用场景需要重点优化解决某个问题。

### Leveled Compaction

![image-20220728233102326](%E7%90%86%E8%A7%A3%E5%B9%B6%E5%AE%9E%E7%8E%B0LSM-Tree/image-20220728233102326.png)

leveled compaction的策略为：对每层的SSTable的数据总量设置一个阈值，level数据越大，阈值也越大，如level 0的阈值为10MB，level 1的阈值为100MB，level 2的阈值为1000MB。当其中某层的SSTable数据总量达到阈值后，选取一个SSTabel合并到下一个level的一个或多个SSTable中。合并后会删除重复的数据，也就说，同一层的SSTable中，没有重复的数据。

leveled compaction能保证同一层的SSTable没有重复数据，相对来说对空间放大有最好的优化效果。但是，合并时需要读取多个SSTable，然后写入多个更新后的SSTable，导致更大的写放大和读放大。但是在数据查找过程中，由于减少了需要查找SSTable的数量，降低了数据查找时的读放大，提升了数据查找的性能。所以，leveled compaction适合比较关注数据查询速度和控制磁盘空间占用的场景。**比如，数据写入一次，但是会被频繁反复查询；数据经常被修改，但是需要控制磁盘空间大小和保证数据查询速度。**

> Cassandra的LCS（`Leveled Compaction Strategy`）和RocksDB的`Classic Leveled Compaction`采用的就是leveled compaction策略。

### Tiered Compaction

![image-20220728233443623](%E7%90%86%E8%A7%A3%E5%B9%B6%E5%AE%9E%E7%8E%B0LSM-Tree/image-20220728233443623.png)

tiered compaction的策略为：当某level层的SSTable数量达到设定的阈值时，则将该层的多个SSTable合并为一个新的SSTable，并放入高一层level中。

tiered compaction比较偏重于控制写放大在一定程度的前提下降低空间放大和提升数据查询性能。但是在同一层的SSTable中还是存在数据重叠，数据查询性能不如leveled compaction。尤其是随着level数越来越大，单个SSTable数据量也越来越大，合并触发条件也越来越难，这些巨型SSTable中的重叠数据和被删除的数据占用的空间也就越来越难被释放掉。tiered compaction比较适合数据修改频率不高，且最近写入的数据查询频率比较高的场景。

> Cassandra的STCS（`Size Tiered Compaction Strategy`）和RocksDB的`universal Compaction`使用的是tiered compaction策略。

### leveled-tiered mixed compaction

是指在一个系统中，结合上述两者合并策略的优势，在部分level之间采用leveled compaction策略，另一部分level之间采用tiered compaction策略。

>  RocksDB的`leveled Compaction`采用的是`leveled-tiered mixed compaction`策略。数据从`Immutable MemTable` flush到level 0时采用的是`tiered compaction`，也就是说level0中是存在重叠数据的。level0到levelN之间采用`leveld compaction`策略。

### FIFO compaction

![image-20220729000754155](%E7%90%86%E8%A7%A3%E5%B9%B6%E5%AE%9E%E7%8E%B0LSM-Tree/image-20220729000754155.png)

`FIFO compaction`在磁盘中只有一个level层级，SSTable按生成的时间顺序排列，一旦生成数据就不会再修改，删除过早生成的SSTable。适合时间序列数据。

> Cassandra的TWCS（`Time Window Compaction Strategy`）、DTCS（`Date Tiered Compaction Strategy`）和RocksDB的`FIFO compaction`使用的是FIFO类型的合并策略。

# 4 基于LSM Tree实现简单KV存储

## 设计思路

### WAL

WAL是保证系统崩溃重启后恢复内存的数据，即，**能够还原所有对内存表的写操作，重新顺序执行这些操作，使得内存表恢复到上一次的状态**。

当Immutable MemTable的数据flush到磁盘后，则可以删除WAL文件，重新创建一个文件。

> 关于 WAL 部分的实现，有不同的做法，有的全局只有唯一一个 WAL 文件，有的则使用多个 WAL 文件，具体的实现会根据场景而变化。

恢复内存表数据时，需要从 WAL 文件中，读取每一次操作信息，重新作用于内存表，即重新执行各种写入操作。因此，直接对内存表进行写操作，和从 WAL 恢复数据重新对内存表进行写操作，都是一样的。恢复的内存表为实现的数据结构（二叉搜索树）

![image-20220729003944686](%E7%90%86%E8%A7%A3%E5%B9%B6%E5%AE%9E%E7%8E%B0LSM-Tree/image-20220729003944686.png)

#### 目标

需要将操作信息写到文件中，并且能够从 WAL 文件恢复内存表

### MemTable & Immutable MemTable

MemTable与Immutable MemTable使用相同的数据结构（二叉搜索树）

系统初始化时，MemTable为空，即没有任何数据，Immutable MemTable为null，即没有分配任何内存。其中MemTable的操作为**插入入KV和删除Key**。

一般当MemTable中的Key数量达到阈值时，MemTable 就会变成Immutable MemTable ，然后创建一个新的MemTable， Immutable MemTable会在合适的时机，转换为SSTable，flush到磁盘文件中。所以Immutable MemTable为一个临时对象。

#### 目标

要实现增删查改、遍历

### SSTable

<img src="%E7%90%86%E8%A7%A3%E5%B9%B6%E5%AE%9E%E7%8E%B0LSM-Tree/image-20220730214859305.png" alt="image-20220730214859305" style="zoom:50%;" />

SSTable结构可区分为多个部分：数据区、KV索引区、元数据区

<img src="%E7%90%86%E8%A7%A3%E5%B9%B6%E5%AE%9E%E7%8E%B0LSM-Tree/image-20220730215725108.png" alt="image-20220730215725108" style="zoom:50%;" />

**数据区**：Immutable MemTable被flush到disk的SSTable中时，顺序将每个KV数据都压缩成二进制数据，存储在数据区；
**KV索引区**：为增加数据区的查找性能，对数据区的每个KV都创建索引，并将索引数据压缩成二进制数据，存储在KV索引区；
**元数据区**：占用固定的数据长度（64K），记录SSTable的数据结构，当查询SSTable的时候，可以先读取该长度内容获取数据；

#### 目标

能够加载文件信息，从中查找对应的数据

### SSTable Tree

随着系统的运行，SSTable会不断增加，则需要对这些SSTable进行管理。如当同一level的SSTable数据大小或者SSTable的数量达到一定阈值后，SSTable进行合并等操作，查询的时候可以增加布隆过滤器快速判断SSTable中是否有Key等。

#### 目标

负责管理所有 SSTable，进行文件合并等

### GIT 代码

> https://github.com/MarkHe1222/lsm

# 总结

1. 写数据时尽量批量操作。LSM Tree数据写入性能已经很高了，但是批量操作时可以节省网络传输RTT时间。
2. 将数据进行分片。这样多个分片可以并行写，如果数据路由处理得当，也可以提升数据查询速度。但是增加了维护多个分片数据读写的复杂度。
3. 选择合适的主键。两个比较流行的选择是：1）采用递增的数字作为主键；2）采用业务本身标识符作为主键。数字作为主键可以减少写入和查询时进行比较、排序等操作的时间，还能提升索引缓存的效率；递增的数字往往保证是顺序写入的，可以减少排序时间。但是递增的数字往往不具备业务语义，业务实际查询时需要先查二级索引，然后进行主键查找。业务标识符往往是业务实际的身份区分符号，业务也往往通过业务标识符进行数据查询。但是业务标识符往往是一个字符串，可能会比较长，这样比较、排序、缓存效率方面不如数字。一般情况下，本人建议LSM Tree数据库采用业务标识符作为主键。因为为业务标识符建立索引以及维护索引的成本是免不了的，与其建立二级索引，不如直接建立主键索引。
4. 设计合理的二级索引，不建立不需要的二级索引。二级索引可以提升相应数据的查询速度，每增加一个二级索引，就需要额外维护相应的二级索引文件，严重影响写入数据性能。
5. 根据具体场景使用合适的SSTable合并策略。单次写，频繁读场景选择leveled compaction策略。
6. 在允许情况下关闭自动SSTable合并，在业务量低的时间段强制执行SSTable合并。
7. 数据更新时合理选择全量更新（覆盖写）方式还是部分更新（增量写）方式。全量更新方式增加了传输和写入的数据量，但是可以提升数据查询速度。部分更新方式会使数据分布在多个SSTable中，需要查询和合并多个SSTable中的数据才能得到完整的数据，会降低数据查询速度。如果数据修改比较频繁，且需要较高查询速度，建议采用全量更新方式。

# 引用

- [LevelDB-GitHub](https://github.com/google/leveldb)
- [磁盘I/O那些事](https://tech.meituan.com/2017/05/19/about-desk-io.html)
- [理解 LSM Tree：一种高效读写的存储引擎](https://mp.weixin.qq.com/s/7kdg7VQMxa4TsYqPfF8Yug)
- [使用 LSM Tree 思想实现一个 KV 数据库](https://blog.csdn.net/sD7O95O/article/details/125038808?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-125038808-blog-105377370.pc_relevant_sortByAnswer&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-125038808-blog-105377370.pc_relevant_sortByAnswer&utm_relevant_index=1)
- [LSM Tree-Based存储引擎的compaction策略](https://www.jianshu.com/p/e89cd503c9ae?utm_campaign=hugo&u_atoken=494fdad0-a3ba-4a73-9b84-abdc66566c50&u_asession=01TB3FROgxhlcU337Vj5OhDfW-5uMmbZWhej68-VdZDB7hG4Th4dX6Ri12I8oRSa5rX0KNBwm7Lovlpxjd_P_q4JsKWYrT3W_NKPr8w6oU7K8vPGlHUZOAPp0aZqGb04UmPpcarp92QKzyJKyYjREPlmBkFo3NEHBv0PZUm6pbxQU&u_asig=05g3629DOBq16YsuugxDy6-_PgNsCIk1KzlHLxZMwpQuyM-sEhvkyH52g9myoV99Pf0yUnhYd5Lwx8te7vhKEvO2hgC15vp56N7ZL0cnxqILMi3M1Y69u4lT5J76ILZDYyMRxkMsMt2_RPWXlQL6bz_PpGTky1w9ZyYAQrYHyw4nn9JS7q8ZD7Xtz2Ly-b0kmuyAKRFSVJkkdwVUnyHAIJzZydNbTMws_2ej6_jSbTNa0zrURdqDnre3mKfjk5KqOiWPRPQyB_SKrj-61LB_f61u3h9VXwMyh6PgyDIVSG1W94f1HPDhDW61g-vj2fa5Kq3IQNwfjy1LpNxIdkfKC0bMSjsbsPHULZ941MPP6uvuLnioLscH950VoSx3U92RzpmWspDxyAEEo4kbsryBKb9Q&u_aref=797dMnj5gcoMAnfTFwfKK88d%2BCo%3D)
- [LSM Tree原理详解](https://www.jianshu.com/p/b43b856e09bb?u_atoken=223e125a-9ceb-44f3-aca4-40de44b26b96&u_asession=01l0ZXqC0_2UjRL4dYjBVf8DGDPymPwntVaK9_HubLJx_XO-1IAAlyNXZTW6g6WxS7X0KNBwm7Lovlpxjd_P_q4JsKWYrT3W_NKPr8w6oU7K800Z4SvggDiBaE0u3VZgT4Ppcarp92QKzyJKyYjREPlmBkFo3NEHBv0PZUm6pbxQU&u_asig=05_UDHsNjsfU-Q4Sf8CfJcSNOEnQSmonOcsFKx1FeS2zZn_lWBMvwS3KMTKrLDn4hqSGgJVy5kDLRz-Eb02U3Y8e1EIqdi1UzMXN1XaoBW-Dqlt8K-4YJ3upWVIsjBp_ahKd-p2N4z8GBNObYI5tQvrmRkWVA1Hy5RChT83Ff2ut_9JS7q8ZD7Xtz2Ly-b0kmuyAKRFSVJkkdwVUnyHAIJzQwIsqIyyzeuCvmumnEayvSX_smdbLpBZ1MvNL2ZBL7oWPRPQyB_SKrj-61LB_f61u3h9VXwMyh6PgyDIVSG1W-ICVx0Jk-rBFAe5nBXiZ7LzZkRHp6MiWHt6djiloXwCg-BwDnGmhS4JOFiRsa3ooHbCGkeDOqqAHBOzRZIOj1hmWspDxyAEEo4kbsryBKb9Q&u_aref=OI25968K%2FBc4pOWTI6RcrH7J15w%3D)
- [LSM树由来、设计思想以及应用到HBase的索引](https://www.cnblogs.com/yanghuahui/p/3483754.html)
- [LSM调研](http://chengfeng96.com/blog/2018/08/04/LSM%E8%B0%83%E7%A0%94%E7%AC%94%E8%AE%B0/)
- [KV实现](https://github.com/burhanxz/Distributed-KV)

