# redb 设计文档

--------------------------------------------------------------------------------------------------

redb 是一个简单、便携、高性能、ACID、嵌入式键值存储。

每个 redb 数据库包含一组表，支持单个写入者和多个并发读取者。redb 使用 MVCC 提供隔离，并提供单一隔离级别：可序列化，其中所有写入都是顺序应用的。

每个表都是一个键值映射，提供类似于 `BTreeMap` 的接口。

# 文件格式
从逻辑上讲，redb 数据库文件由一些元数据和几个 B-tree 组成：
* pending free tree：从事务 ID 到它们释放的页面列表的映射
* table tree：表名到表定义的映射
* data tree(s)（每个表一个）：表的键到值的映射

除了数据库元数据外，所有其他数据结构都是写时复制的。

所有多字节整数都以小端序存储。

数据库文件以数据库头开始，后面跟着一个或多个区域。每个区域包含一个头和一个数据部分，数据部分被分割成许多页面。这些区域允许数据库文件的高效、动态增长。

```
<-------------------------------------------- 8 bytes ------------------------------------------->
========================================== Super header ==========================================
---------------------------------------- Header (64 bytes) ---------------------------------------
| magic number                                                                                   |
| magic con.| god byte  | padding               | page size                                      |
| region header pages                           | region max data pages                          |
| number of full regions                        | data pages in trailing region                  |
| padding                                                                                        |
| padding                                                                                        |
| padding                                                                                        |
| padding                                                                                        |
------------------------------------ Commit slot 0 (128 bytes) -----------------------------------
| version   | user nn   | sys nn    | f-root nn | padding                                        |
| user root page number                                                                          |
| user root checksum                                                                             |
| user root checksum (cont.)                                                                     |
| user root length                                                                               |
| system root page number                                                                        |
| system root checksum                                                                           |
| system root checksum (cont.)                                                                   |
| system root length                                                                             |
| padding                                                                                        |
| padding                                                                                        |
| padding                                                                                        |
| padding                                                                                        |
| transaction id                                                                                 |
| slot checksum                                                                                  |
| slot checksum (cont.)                                                                          |
----------------------------------------- Commit slot 1 ------------------------------------------
|                                 Same layout as commit slot 0                                   |
---------------------------------- footer padding (192+ bytes) -----------------------------------
==================================================================================================
| Region header (unused as of v3)                                                                |
--------------------------------------------------------------------------------------------------
| Region data                                                                                    |
==================================================================================================
| ...more regions                                                                                |
==================================================================================================
```

## Database super-header

数据库 super-header 长 512 字节（向上舍入到最近的页面），由一个 header 和两个"commit slots"组成。

### Database header (64 bytes)
数据库 header 包含几个不可变字段，如数据库页面大小、区域大小和 magic number。
事务数据存储在双缓冲字段中，主副本通过更新单个字节来管理，该字节控制哪个事务指针是主副本。

* 9 bytes: magic number
* 1 byte: god byte
* 2 byte: padding
* 4 bytes: page size
* 4 bytes: region header pages
* 4 bytes: region max data pages
* 4 bytes: number of full regions
* 4 bytes: data pages in partial trailing region
* 32 bytes: padding to 64 bytes

`magic number` 必须设置为 ASCII 字母 'redb' 后跟 0x1A, 0x0A, 0xA9, 0x0D, 0x0A。这个序列是
受 PNG magic number 启发。

`god byte`，之所以这样命名是因为这个字节控制整个数据库的状态，是一个包含三个标志的位域：
* 第一位：`primary_bit` 标志，指示事务 slot 0 或事务 slot 1 包含最新的提交。
* 第二位：`recovery_required` 标志，如果设置则在打开数据库时必须运行恢复过程。这可以
  是完全修复，其中区域跟踪器和区域分配器状态——如下所述——通过从所有活动根遍历 btree 来重建，
  或快速修复，其中状态直接从分配器状态表加载。
* 第三位：`two_phase_commit` 标志，指示主槽中的事务是否使用两阶段提交写入。如果是这样，
  主槽保证有效，修复不会查看辅助槽。此标志始终与主位原子更新。

redb 依赖于这是一个单字节的事实来执行原子提交。

`page size` 是 redb 页面的大小（以字节为单位）

`region header pages` 是每个区域 header 中的页面数

`region max data pages` 是每个区域中的最大数据页面数

`number of full regions` 存储数据库中的完整区域数。这可以在事务期间随着数据库的增长或缩小而改变。
只有在数据库不需要恢复时此字段才有效。否则必须从文件长度重新计算并验证

`pages in partial trailing region` 除最后一个外的所有区域都必须是完整的。这存储了最后一个区域中的页面数。
只有在数据库不需要恢复时此字段才有效。否则必须从文件长度重新计算并验证

### Transaction slot 0 (128 bytes):
* 1 byte: 文件格式版本号
* 1 byte: 布尔值，指示用户根页面是否为非空
* 1 byte: 布尔值，指示系统根页面是否为非空
* 1 byte: 布尔值，指示释放的表根页面是否为非空
* 4 bytes: 填充到 64 位对齐
* 8 bytes: 用户根页面
* 16 bytes: 用户根校验和
* 8 bytes: 用户根长度
* 8 bytes: 系统根页面
* 16 bytes: 系统根校验和
* 8 bytes: 系统根长度
* 40 bytes: 填充
* 8 bytes: 最后提交的事务 ID
* 16 bytes: slot 校验和

`version` 数据库的文件格式版本。这存储在事务数据中，以便在升级期间可以原子地更改。

`user root page` 是用户表树根的页面号。

`user root checksum` 存储用户根页面的 XXH3_128bit 校验和，后者又存储其子页面的校验和。

`user root length` 是用户表树中的表数。此字段在文件格式 v2 中是新的。

`system root page` 是系统表树根的页面号。

`system root checksum` 存储系统根页面的 XXH3_128bit 校验和，后者又存储其子页面的校验和。

`system root length` 是系统表树中的表数。此字段在文件格式 v2 中是新的。

`slot checksum` 是事务槽中所有前面字段的 XXH3_128bit 校验和。

### Transaction slot 1 (128 bytes):
* 与 slot 0 相同的布局

### Region tracker
区域跟踪器是一个 `BtreeBitmap` 数组，用于跟踪每个区域中空闲的页面顺序。
它可以存储在两个不同的地方：关闭时，它被写入区域数据部分中的页面，
而在启用快速修复的提交时，它被写入分配器状态表中的条目。
前者仅在干净关闭后有效；后者即使在崩溃后也可使用。
```
<-------------------------------------------- 8 bytes ------------------------------------------->
==================================================================================================
| num_allocators                                | sub_allocator_len                              |
| BtreeBitmap data...                                                                            |
==================================================================================================
```

`BtreeBitmap` 是一个 64 路树，其中每个节点是单个值，指示任何后代是否空闲。
这些节点作为单个位存储，打包到 `u64` 值中。树的每一层都是完全填充的，除了叶层。

* 4 bytes: 树高度
* 4 bytes (重复): 层的结束偏移量。不包括根层
* n bytes: 树数据

```
<-------------------------------------------- 8 bytes ------------------------------------------->
==================================================================================================
| height                                        | end offset...                                  |
==================================================================================================
| Tree data                                                                                      |
==================================================================================================
```

## Region layout
* 1 byte: 区域格式版本
* 3 bytes: 填充
* 4 bytes: 分配器状态的长度（以字节为单位）
* n bytes: 分配器状态

每个区域由包含元数据的 header（主要是区域页面的分配状态）和包含页面的数据部分组成。
页面有一个基本大小，默认为 OS 页面大小，并且以基本大小的 2 次幂倍数可变大小。
页面格式在[以下部分](#b-tree-pages)中描述

```
<-------------------------------------------- 8 bytes ------------------------------------------->
==================================================================================================
| version  | padding                            | allocator state length                         |
--------------------------------------------------------------------------------------------------
| Regional allocator state                                                                       |
==================================================================================================
| Region pages                                                                                   |
==================================================================================================
```

### Regional allocator state
区域分配器是伙伴分配器，在区域内分配页面。其伙伴分配器实现依赖于上面描述的 `BtreeBitmap` 和 `U64GroupedBitmap`。
btree 映射用于跟踪空闲页面，分组位图用于跟踪偶数地址范围已分配的顺序
* 1 byte: 最大顺序
* 3 byte: 填充到 32 位对齐
* 4 bytes: 页面数
* 4 byte (重复): 顺序分配器空闲状态的结束偏移量
* 4 byte (重复): 顺序分配器已分配页面状态的结束偏移量
* n bytes: 空闲索引数据
* n bytes: 已分配数据

与区域跟踪器一样，区域分配器状态可以存储在两个不同的地方。
关闭时，它被写入上述区域 header，而在启用快速修复的提交时，
它被写入分配器状态表中的条目。前者仅在干净关闭后有效；
后者即使在崩溃后也可使用。

```
<-------------------------------------------- 8 bytes ------------------------------------------->
==================================================================================================
| max order | padding                           | number of pages                                |
--------------------------------------------------------------------------------------------------
| Order end offsets...                                                                           |
==================================================================================================
| Order allocator state...                                                                       |
==================================================================================================
```

## B-tree pages

分配的页面可能有两种类型：b-tree 分支页面或 b-tree 叶页面。下面描述每种格式：

### Branch page:
* 1 byte: 类型
* 1 byte: 填充到 16 位对齐
* 2 bytes: num_keys（键数）
* 4 byte: 填充到 64 位对齐
* 重复 (num_keys + 1 次):
* * 16 bytes: 子页面校验和
* 重复 (num_keys + 1 次):
* * 8 bytes: 页面号
* （可选）重复 (num_keys 次):
* * 4 bytes: 键结束。键的结束偏移量，不包括
* 重复 (num_keys 次):
* * n bytes: 键数据
```
<-------------------------------------------- 8 bytes ------------------------------------------->
==================================================================================================
| type     | padding   | number of keys        | padding                                         |
--------------------------------------------------------------------------------------------------
| child page checksum (repeated num_keys + 1 times)                                              |
|                                                                                                |
--------------------------------------------------------------------------------------------------
| child page number (repeated num_keys + 1 times)                                                |
--------------------------------------------------------------------------------------------------
| (optional) key end (repeated num_keys times) | alignment padding                               |
==================================================================================================
| Key data                                                                                       |
==================================================================================================
```
`type` 对于分支页面是 `2`

`num_keys` 指定页面中的键数

`child page checksum` 是子页面的校验和数组，在 `page number` 数组中

`page number` 是子页面号数组

`key_end` 是键的结束偏移量数组。它是可选的，对于固定宽度键类型绝对不能存储

`alignment padding` 可选填充，使键数据从键类型所需对齐的倍数开始

### Leaf page:
* 1 byte: 类型
* 1 byte: 保留（填充到 16 位对齐）
* 2 bytes: num_entries（对数）
* （可选）重复 (num_entries 次):
* * 4 bytes: key_end
* （可选）重复 (num_entries 次):
* * 4 bytes: value_end
* 重复 (num_entries 次):
* * n bytes: 键数据
* 重复 (num_entries 次):
* * n bytes: 值数据
```
<-------------------------------------------- 8 bytes ------------------------------------------->
==================================================================================================
| type     | padding   | number of entries      | (optional) key end (repeated entries times)    |
--------------------------------------------------------------------------------------------------
| (optional) value end (repeated entries times) | (optional) key alignment padding               |
==================================================================================================
| Key data                                      | (optional) value alignment padding             |
==================================================================================================
| Value data                                                                                     |
==================================================================================================
```

`type` 对于叶页面是 `1`

`num_entries` 指定叶中的键值对数

`key_end` 是键的结束偏移量数组。它是可选的，对于固定宽度键类型绝对不能存储

`value_end` 是值的结束偏移量数组。它是可选的，对于固定宽度值类型绝对不能存储

`key alignment padding` 可选填充，使键数据从键类型所需对齐的倍数开始

`value alignment padding` 可选填充，使值数据从值类型所需对齐的倍数开始

# Commit strategies

所有数据在写入时都使用非加密 Merkle 树 XXH3_128 进行校验和。这
允许检测部分提交的事务并在崩溃后回滚。

所有 redb 事务都是原子的，并使用以下提交策略之一。

## Non-durable commits
redb 支持"非持久化"提交，这意味着没有持久性保证。但是，在发生崩溃时
数据库仍保证一致，并将返回到最后一个非持久化提交或最后一个
完整提交。

非持久化提交通过内存标志实现，该标志指示读取器从辅助页面读取，
即使它尚未提升为主页面。
在发生崩溃时，数据库将简单地回滚到主页面，并且可以通过正常修复过程安全地重建分配器状态。
请注意，在非持久化提交期间不允许释放页面，因为它可能随时回滚。

## 1-phase + checksum durable commits (1PC+C)
默认情况下使用减少延迟的提交策略。提交使用单个 `fsync` 执行。
首先，写入所有数据和校验和以及单调递增的事务
id，然后翻转主页面并调用 `fsync`。如果发生崩溃，我们必须验证主页面具有更大的
事务 id 并且所有校验和都有效。如果此检查失败，则数据库将用辅助页面替换部分
更新的主页面。

下面我们给出这种一阶段提交方法的简要正确性分析，并考虑 fsync 期间可能发生的不同故障情况：

1. 如果事务的所有数据都被写入磁盘或都没有写入，则此策略显然有效
2. 如果只写入了一些数据而不是全部，则需要考虑几种情况：
   1. 如果未更新控制主页面的位，则事务没有效果，此方法是安全的
   2. 如果已更新，则必须验证事务是否完全写入，否则回滚：
      1. 如果未写入事务数据，则会检测到事务 id 较旧并回滚
      2. 如果只写入了一部分，则校验和验证将失败并回滚。

## 2-phase durable commits (2PC)
也可以使用两阶段提交策略，当处理恶意数据时以及攻击者对 redb 进程具有高度控制时缓解理论攻击（见下文）。
首先，数据被写入 btree 的新副本，其次执行 `fsync`，
最后翻转控制哪个 btree 副本是主副本的字节并执行第二次 `fsync`。

### 1PC+C 的安全性
鉴于 1PC+C 提交策略依赖于非加密校验和 (XXH3)，至少在理论上存在攻击它的方法。
需要接受恶意输入的用户被鼓励使用 2PC。

攻击者可以使部分提交的事务看起来像完全提交的事务。
场景可能是这样的：
```
table.insert(malicious_key, malicious_value);
table.insert(good_key, good_value);
txn.commit();
```
攻击者希望事务看起来像：
```
table.insert(malicious_key, malicious_value);
txn.commit();
```
为此，他们需要：
1) 控制页面刷新到磁盘的顺序，以便与 `good_key` 相关的页面永远不会被写入
2) 在提交的 fsync() 操作期间或紧接其前引入崩溃
3) 确保部分写入数据的校验和有效

在完全控制工作负载或读取数据库文件的情况下，(3) 是可能的，因为 XXH3 不具有抗冲突性。
但是，它要求攻击者了解数据库内容，因为校验和的输入包括
许多其他值（b-tree 根中的所有其他键及其子节点号）

# MVCC (multi-version concurrency control)

redb 使用 MVCC 将事务彼此隔离。这是在底层的写时复制
b-tree 数据结构之上实现的，该数据结构是 redb 的基础。读取事务制作 b-tree 根的私有副本，
并在数据库中注册，以便不会释放根引用的页面。
当写入事务释放页面时，它被推入队列，只有在所有读取事务
引用它的完成后才重新使用。

## Savepoints

保存点和回滚在相同的 MVCC 结构上实现。创建保存点时
注册为读取事务以保留数据库快照。此外，它保存页面分配器状态的副本
——假设 4kb 页面，每 1GB 值大约 64kB。要回滚
数据库到保存点，恢复 b-tree 的根，并将页面分配器的快照
与当前分配的页面进行差异比较，以确定自保存点以来分配了哪些页面
创建。这些页面然后排队释放，完成回滚。

保存点有两种类型：
1. 临时的。这些保存点在删除时立即释放
2. 持久的。这些保存点持久化在数据库文件中，因此跨重启。
   它们存储在系统表树中的表内。必须显式释放它们。

## 基于纪元的回收
如上所述，事务和保存点依赖于基于纪元的页面回收，以确保
只有在不再引用页面后才释放它们。必须小心确保
引用的页面永远不会被释放。下面描述了高级设计：

### 定义
* 事务状态：
  * 未提交。正在进行的写入事务
  * 已提交。已提交的写入事务
  * 已中止。已中止的写入事务
  * 孤立。不再被引用的读取或写入事务
  * 已回收。成为孤立的写入事务，其所有待释放页面都
    已回收
* 脏页面：已分配且分配它的事务尚未提交的页面
* 已提交页面：已分配且分配它的事务已提交的页面
* 待释放页面：已分配且不再从数据树或系统中可访问的页面
  树根，从关联的事务开始。这些存储在释放的树中。

### 不变量
必须维护以下不变量
* 已提交页面永远不会被原地修改
* 已提交页面只能通过转换到"待释放页面"状态来释放，并且
  关联的事务和所有先前的事务都已达到孤立状态后
* 页面将只包含指向相同或更早事务中的页面的页面指针。这遵循
  从已提交页面永远不会被修改以及事务 ID 单调递增的事实
* 释放的树必须只包含待释放页面状态的页面

### 关键操作
#### 写入事务期间的修改
写入事务通过非常简单的机制维护所需的不变量。所有修改
对已提交页面的写时复制。也就是说，分配新页面，
在新页面中进行修改，并构建引用新页面的新 b-tree。脏页面是
允许原地修改——通常通过释放它们来完成。
普通表、系统表和释放的树都使用这种方法。
当中止事务时，它分配的所有页面都会立即释放。

#### 保存点恢复
恢复保存点将所有普通表恢复到创建保存点时的状态。
保存点系统表不受影响，并且必须将释放的树置于
一致状态。
恢复普通表是微不足道的，因为保存点捕获数据根。

释放的表需要以两种方式更新：
1) 来自保存点的恢复数据根必须只引用已提交的页面，所以我们删除所有
   未来事务中的待释放页面，这些页面在数据释放的树中。
2) 通过恢复保存点数据根而变得不可访问的所有页面都必须添加到释放的
   表中。这是通过将数据树中的分配页面表中的所有页面添加到
   数据释放的树中来完成的。

#### 数据库修复
要在不干净关闭后修复数据库，我们必须：
1) 更新 super header 以引用最后完全提交的事务
2) 更新分配器状态，使其与上述事务中的所有数据库根一致

如果崩溃前的最后一次提交启用了快速修复，那么这两者都是微不足道的。主提交槽保证有效，
因为它使用两阶段提交写入，并且相应的分配器状态存储在分配器状态表中。

否则，我们需要执行完全修复：

对于 (1)，如果主提交槽无效，我们切换到辅助槽。

对于 (2)，我们通过遍历以下树并标记所有引用的页面为已分配来重建分配器状态：
* 数据树
* 系统树
* 释放的树，以及其中包含的所有待释放页面
保存点引用的所有页面都必须包含在上述内容中，因为它是：
a) 直接由数据、系统或释放的树引用——即它是已提交的页面
b) 未被引用，在这种情况下它处于待释放状态并包含在释放的树中

# 版本变更
## v1
初始文件格式

## v2
向 btrees 和表添加了长度字段。这允许常数时间 `len()`

## v3
* 删除了释放的树。相反，这些页面存储在两个系统表中，一个用于数据树
  一个用于系统树。
* 添加了"分配的页面"系统表，用于跟踪每个事务分配的页面当
  存在保存点时
* 删除了分配器状态。相反，使用"快速修复"代码路径。

# 关于底层媒体的假设
redb 的设计即使在断电或在行为不佳的媒体上也是安全的。
因此，我们只对底层文件系统提供的保证做出一些假设：
1. 单字节写入是原子的：即，每个字节将完全写入或根本不写入，
   即使在断电的情况下
2. `fsync` 操作后写入是持久的
3. ["powersafe overwrite"](https://www.sqlite.org/psow.html)：当应用程序写入
   文件中的字节范围时，该范围之外的字节不会改变，
   即使写入发生在崩溃或断电之前。sqlite 默认情况下在所有现代版本中都做出这个
   假设。

# 代码结构与数据结构对应关系

以下是在 redb 代码库中实现的与设计文档中描述的数据结构对应的具体代码位置：

1. **DatabaseHeader** - `src/tree_store/page_store/header.rs`
   - 包含数据库超级头、事务槽等结构
   - 对应设计文档中的"Database super-header"和"Transaction slot"

2. **BtreeHeader** - `src/tree_store/btree_base.rs`
   - 包含根页面号、校验和和长度信息
   - 对应设计文档中的B-tree头部信息

3. **PageNumber** - `src/tree_store/page_store/base.rs`
   - 页面编号结构，包含区域号、页面索引和页面顺序
   - 对应设计文档中的页面寻址机制

4. **Leaf Page Format** - `src/tree_store/btree_base.rs` (LeafAccessor, RawLeafBuilder)
   - 叶页面的访问器和构建器
   - 对应设计文档中的"Leaf page"格式

5. **Branch Page Format** - `src/tree_store/btree_base.rs` (BranchAccessor, RawBranchBuilder)
   - 分支页面的访问器和构建器
   - 对应设计文档中的"Branch page"格式

6. **RegionTracker 和 BuddyAllocator** - `src/tree_store/page_store/region.rs` 和 `src/tree_store/page_store/buddy_allocator.rs`
   - 区域跟踪器和伙伴分配器
   - 对应设计文档中的"Region tracker"和"Regional allocator state"

7. **BtreeBitmap 和 U64GroupedBitmap** - `src/tree_store/page_store/bitmap.rs`
   - 位图数据结构，用于页面分配跟踪
   - 对应设计文档中的BtreeBitmap结构

8. **DatabaseLayout 和 RegionLayout** - `src/tree_store/page_store/layout.rs`
   - 数据库和区域布局管理
   - 对应设计文档中的区域布局描述

9. **TransactionalMemory** - `src/tree_store/page_store/page_manager.rs`
   - 事务内存管理器，负责页面分配和事务管理
   - 对应设计文档中的页面管理机制

10. **Savepoint** - `src/tree_store/page_store/savepoint.rs`
    - 保存点实现
    - 对应设计文档中的"Savepoints"部分

所有结构都与设计文档中的描述一致，实现了文档中提到的核心功能，包括MVCC、B-tree存储结构、页面分配和回收机制、事务提交策略(1PC+C和2PC)、保存点和回滚功能以及数据库文件格式和布局。

# primary_copy 机制详解

在 redb 中，"primary_copy" 实际上是"primary slot"（主槽）的概念，它是 redb 数据库事务提交和恢复机制的核心部分。

## 概念解释

redb 使用双缓冲机制来管理事务提交，包含两个事务槽（transaction slots）：
- **Slot 0**: 事务槽 0
- **Slot 1**: 事务槽 1

其中一个槽被标记为"primary"（主槽），代表当前有效的数据库状态。"primary_copy" 本质就是指这个当前活跃的事务槽。

## 实现机制

### 1. God Byte 标志位
在数据库头部的 "god byte" 中，使用不同的位来控制：
- **PRIMARY_BIT (0x01)**: 控制哪个槽是主槽
- **TWO_PHASE_COMMIT (0x04)**: 指示是否使用 2PC 提交

### 2. Swap 操作
`swap_primary_slot()` 函数通过异或操作切换主槽：
```rust
pub(super) fn swap_primary_slot(&mut self) {
    self.primary_slot ^= 1;  // 在 0 和 1 之间切换
}
```

### 3. 两种提交策略

#### 1PC+C (1-phase + checksum commit)
- 默认的高性能提交策略
- 只需要一次 fsync
- 依赖校验和来检测部分提交
- 流程：
  1. 写入数据到 secondary slot
  2. 写入头部信息
  3. 翻转 primary bit（切换主槽）
  4. 执行 fsync

#### 2PC (2-phase commit)
- 更安全但较慢的提交策略
- 需要两次 fsync
- 流程：
  1. 写入数据到 secondary slot
  2. 第一次 fsync（确保数据持久化）
  3. 翻转 primary bit（切换主槽）
  4. 第二次 fsync（确保头部更新持久化）

## 代码实现位置

- `src/tree_store/page_store/header.rs`: DatabaseHeader 结构和 primary_bit 定义
  - `PRIMARY_BIT` 常量定义为 1
  - `DatabaseHeader::swap_primary_slot()` 方法实现槽切换
  - god byte 中的标志位管理

- `src/tree_store/page_store/page_manager.rs`: 提交逻辑实现
  - `TransactionalMemory::commit_inner()` 方法包含完整的提交流程
  - 写入 secondary slot、翻转主槽、fsync 操作

- `src/transactions.rs`: 事务控制接口
  - `WriteTransaction::set_two_phase_commit()` 方法允许用户控制提交策略

## 优势

这种设计的优势包括：
- **高性能**: 1PC+C 策略只需要一次 fsync，比传统 2PC 快约 2 倍
- **安全性**: 通过校验和和事务 ID 检测和处理部分提交
- **原子性**: 通过单字节切换保证事务的原子性

# 保存点间数据流转详解

在两个保存点之间，kv 数据从写入到落盘的流程以及不同模块中的状态流转如下：

## 1. 核心概念回顾

根据本文档，redb 使用 MVCC（多版本并发控制）实现事务隔离，所有写入都是顺序应用的。保存点和回滚在相同的 MVCC 结构上实现，创建保存点时会注册为读取事务以保留数据库快照，同时保存页面分配器状态的副本。

## 2. 写入到落盘的完整流程

### 写入阶段
1. **数据写入**：用户通过 `WriteTransaction` 打开表并进行数据写入（插入、更新、删除），这些操作会修改 B-tree 结构。
2. **页面分配**：在写入过程中，如果需要新页面，会通过 `TransactionalMemory::allocate()` 分配页面，这些页面被记录在 `allocated_since_commit` 集合中。
3. **非持久化提交**：如果使用 `Durability::None`，调用 `non_durable_commit()`，数据更新到 secondary slot，但不调用 `fsync`，页面可能被放入 `unpersisted` 集合。
4. **持久化提交**：如果使用 `Durability::Immediate`，调用 `durable_commit()`，执行以下步骤：
   - 处理需要释放的页面，将它们从 `DATA_FREED_TABLE` 和 `SYSTEM_FREED_TABLE` 中移除并实际释放。
   - 如果启用快速修复（quick-repair），会将分配器状态保存到 `ALLOCATOR_STATE_TABLE` 系统表中。
   - 调用 `TransactionalMemory::commit()`，将数据和系统根更新写入 secondary slot。
   - 如果启用两阶段提交（2PC），先调用 `fsync` 确保数据持久化，再更新主槽位并再次调用 `fsync`。
   - 如果使用单阶段提交（1PC+C），更新 secondary slot 后直接切换主槽位并调用 `fsync`。

### 落盘阶段
1. **文件系统写入**：通过 `PagedCachedFile` 将页面数据写入文件系统缓存。
2. **数据持久化**：调用 `fsync` 将文件系统缓存中的数据真正写入磁盘，确保数据持久化。

## 3. 不同模块中的状态流转

### WriteTransaction 模块
- **初始状态**：`dirty` 标志为 false，没有打开的表。
- **写入过程中**：打开表后 `dirty` 标志变为 true，表被记录在 `open_tables` 中。
- **创建保存点**：调用 `ephemeral_savepoint()` 或 `persistent_savepoint()` 时，会检查 `dirty` 标志，如果为 true 则返回错误。保存点会记录当前的数据根（`user_root`）和事务 ID。
- **提交过程中**：
  - `commit_inner()`：根据 `durability` 设置调用 `non_durable_commit()` 或 `durable_commit()`。
  - `durable_commit()`：
    - 调用 `process_freed_pages()` 处理需要释放的页面。
    - 如果启用快速修复，调用 `store_system_freed_pages()` 将系统表中释放的页面信息存储到 `SYSTEM_FREED_TABLE`。
    - 调用 `TransactionalMemory::commit()` 执行实际的提交操作。
    - 更新 `transaction_tracker` 中的非持久化提交状态。
  - `non_durable_commit()`：
    - 调用 `process_freed_pages_nondurable()` 处理未持久化的释放页面。
    - 调用 `store_system_freed_pages()` 将系统表中释放的页面信息存储到 `SYSTEM_FREED_TABLE`。
    - 调用 `TransactionalMemory::non_durable_commit()` 执行非持久化提交操作。
    - 注册非持久化提交到 `transaction_tracker`。
- **提交完成后**：清空 `allocated_since_commit` 和 `unpersisted` 集合，更新 `transaction_tracker` 中的保存点状态。

### TransactionalMemory 模块
- **页面分配**：`allocate()` 方法分配新页面，并将其添加到 `allocated_since_commit` 集合中。
- **页面释放**：
  - `free_if_uncommitted()`：如果页面在 `allocated_since_commit` 集合中，则将其释放并从集合中移除。
  - `free_if_unpersisted()`：如果页面在 `unpersisted` 集合中，则将其释放并从集合中移除。
- **提交过程中**：
  - `commit_inner()`：
    - 更新 secondary slot 的事务 ID、用户根和系统根。
    - 如果启用两阶段提交，调用 `fsync`。
    - 切换主槽位，更新 `two_phase_commit` 标志。
    - 写入新头部并调用 `fsync`。
    - 清空 `allocated_since_commit` 和 `unpersisted` 集合。
  - `non_durable_commit()`：
    - 更新 secondary slot 的事务 ID、用户根和系统根。
    - 将 `allocated_since_commit` 中的页面转移到 `unpersisted` 集合中。
    - 设置 `read_from_secondary` 标志为 true。
- **提交完成后**：`allocated_since_commit` 和 `unpersisted` 集合为空，`read_from_secondary` 标志根据提交类型设置。

### TransactionTracker 模块
- **保存点管理**：
  - `allocate_savepoint()`：分配新的保存点 ID，并将其与事务 ID 关联。
  - `deallocate_savepoint()`：释放保存点 ID 及其关联的事务 ID。
  - `is_valid_savepoint()`：检查保存点 ID 是否有效。
  - `invalidate_savepoints_after()`：使指定保存点之后的所有保存点失效。
- **非持久化提交管理**：
  - `register_non_durable_commit()`：注册非持久化提交，记录其与上次持久化提交的关联。
  - `clear_pending_non_durable_commits()`：清空待处理的非持久化提交。
  - `is_unprocessed_non_durable_commit()`：检查非持久化提交是否已处理。
  - `mark_unprocessed_non_durable_commit()`：标记非持久化提交为已处理。
  - `oldest_unprocessed_non_durable_commit()`：获取最旧的未处理非持久化提交。

### Savepoint 模块
- **创建**：`new_ephemeral()` 创建临时保存点，记录当前事务 ID 和数据根。
- **持久化**：`set_persistent()` 将临时保存点标记为持久化保存点。
- **序列化**：`from_savepoint()` 将保存点序列化为字节数据，`to_savepoint()` 将字节数据反序列化为保存点对象。

通过以上分析，我们可以看到在两个保存点之间，kv 数据从写入到落盘涉及多个模块的协同工作，每个模块都有明确的职责和状态流转过程，共同保证了数据的一致性和持久性。