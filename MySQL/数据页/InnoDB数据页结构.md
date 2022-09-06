# InnoDB数据页结构

# 1. 数据页的结构

前面我们已经说过了，一个数据页占用 `16KB` 大小的存储空间，它可以划分为以下几个部分。

![Untitled](InnoDB%E6%95%B0%E6%8D%AE%E9%A1%B5%E7%BB%93%E6%9E%84%20e9b4a0e3652a4c46ba71819a8556541b/Untitled.png)

| 名称 | 中文名 | 占用空间大小 | 简单描述 |
| --- | --- | --- | --- |
| File Header | 文件头部 | 38字节 | 页的一些通用信息 |
| Page Header | 页面头部 | 56字节 | 数据页专用的一些信息 |
| Infimum+Supremum | 页面中的最小记录和最大记录 | 26字节 | 两个虚拟的记录 |
| User Records | 用户记录 | 不确定 | 用户存储的记录内容 |
| Free Space | 空闲空间 | 不确定 | 页中尚未使用的空间 |
| Page Directory | 页目录 | 不确定 | 也中某些记录的相应位置 |
| File Trailer | 文件尾部 | 8字节 | 校验页是否完整 |

# 2. 记录

## 示例数据

首先创建一个示例表，并插入4条数据

```sql
CREATE TABLE page_demo (
    c1 int(11) NOT NULL,
    c2 int(11) DEFAULT NULL,
    c3 varchar(1000) DEFAULT NULL,
    PRIMARY KEY (c1)
) CHARSET=ascii ROW_FORMAT=COMPACT;
-- 插入4条数据
INSERT INTO page_demo VALUES(1,100,'aaaa'), (2,200,'bbbb'), (3,300,'cccc'),(4,400,'dddd');
```

## 记录头信息

上面插入的4条记录对应部分行信息如下图（这里只画出了记录的部分头信息和部分记录的真实数据）：

![Untitled](InnoDB%E6%95%B0%E6%8D%AE%E9%A1%B5%E7%BB%93%E6%9E%84%20e9b4a0e3652a4c46ba71819a8556541b/Untitled%201.png)

我们详细说一下上图记录的头信息中每个数据的含义

- `delete_flag` ：用来表示当前记录是**否被删除**，占用1个比特位。1表示记录被删除了；0表示未被删除。被删除的记录不会从磁盘中移除（因为移除记录会导致其他记录需要重新排序）。InnoDB会将被删除的放入到垃圾链表中（从原有的next_record链表移除），在该链表上的空间是是**可重用**的（之后插入记录可以重用这部分空间）。
- `min_rec_flag` ：表示是否是B+数每层非叶子节点中的最小目录项。上图中我们插入的记录的 `min_rec_flag` 都是 0，他们都不是B+树中非叶子节点中最小的目录项记录。
- `heap_no` ：表示记录在 `User Record` 中的相对位置，记录在 `User Record` 中是紧密排列着的。每次申请一条记录的空间，该记录比物理位置在它前面的那条记录的 `heap_no`大`1` （heap_no 最小的是 Infimum 和 Supremum）。*PS：heap_no分配后就不会改变了，删除堆中的某条记录，heap_no也不会改变。*
    
    ![Untitled](InnoDB%E6%95%B0%E6%8D%AE%E9%A1%B5%E7%BB%93%E6%9E%84%20e9b4a0e3652a4c46ba71819a8556541b/Untitled%202.png)
    
- `record_type` ：表示记录的类型。
    - `2` ：表示 `Infimum` 记录（一个页面中最小记录）
    - `3` ：表示 `Supremu` 记录（一个页面中最大记录）
    - `0` ：表示普通记录
    - `1` ：表示目录项记录
- `next_record` ：表示当前记录的真实数据到下一条记录的真实数据的距离，同时将页中的记录连接起来。如果是正值，说明当前记录的下一条记录在当前该记录的后面；如果是负值，说明当前记录的下一条记录在当前记录的前面。并且从上图可以看出 `next_record` 指针指向的是下一条记录的真实数据部分的开始位置，这样可以方便向左读取**记录的额外信息**，向右读取**记录的真实数据部分**。
- `n_owned` ：页中的记录会分为很多个组，后面会说。

## 删除记录的变化

当我们执行下面的SQL删除第二天用户记录时

```sql
DELETE FROM page_demo WHERE c1 = 2;
```

![Untitled](InnoDB%E6%95%B0%E6%8D%AE%E9%A1%B5%E7%BB%93%E6%9E%84%20e9b4a0e3652a4c46ba71819a8556541b/Untitled%203.png)

删除一条记录会进行以下修改：

- 第2条记录并没有从存储空间中删除，而是把该记录的 `delete_flag` 改为`0`。
- 然后再将第2条记录的 `next_record` 改为`0`，意味着它没有下一条记录了。
- 将第1条记录的 `next_record`  指向第3条记录。
- 将第二组的组长 `supremum` 的 `n_owned` 减1，之前改组有5条记录，删除了一条，还有4条， `n_owned` 改为`4` 。

当我们将第2条记录重新插入记录时，不会分配信息的存储空间，会直接复用被删的记录2的空间。

# 3. 页目录

前面我们知道，页中的记录是会根据主键的大小，通过 `next_record` 连接成一个单向链表。当我们想找到一条目标记录时，只能通过变量单向链表。如果页中记录过多时，这个变量的过程就会很耗时。所以就有了页目录（**Page Directory**）。

**页目录**中存的是每组记录中**组长在页面中的偏移量**，这些偏移量称为**槽**（**Slot**），每个槽占用2字节。那么问题来了组是什么呢？

**组**就是将页面中相邻的几条记录分为一个组，然后将组中**最后一条记录**作为**组长**。并在组长的 `n_owned` 中记录当前组中的记录数量。那么如何进行分组呢？InnoDB的分组需要符合以下要求：

- 规定 `Infimum` 记录所在的组只能有1条记录，这条记录就是 `Infimum` 自己，并且他是组长。
- `Supremum` 记录所在的分组拥有的记录数只能在 `1~8`之间。
- 其他分组中的记录数只能是 `4~8` 条之间。

继续向表中插入更多的数据

```sql
INSERT INTO page_demo VALUES(5,500,'eeee'),(6,600,'ffff'),(7,700,'gggg'),(8,800,'hhhh'),(9,900,'iiii'),(10,1000,'jjjj'),(11,1100,'kkkk'),(12,1200,'llll'),(13,1300,'mmmm'),(14,1400,'nnnn'),(15,1500,'oooo'),(16,1600,'pppp');
```

从下图我们可以看到，页目录中共有5个槽，说明页中共有5个组。

![Untitled](InnoDB%E6%95%B0%E6%8D%AE%E9%A1%B5%E7%BB%93%E6%9E%84%20e9b4a0e3652a4c46ba71819a8556541b/Untitled%204.png)

我们以一条SQL为例，看它是如何通过页目录找到目标记录的

```sql
SELECT * FROM page_demo WHERE c1 = 7;
```

步骤如下（二分法）：

1. 先找到页目录中的中间那个槽， (0+4)/2=2 所以先跟 Slot2对应的组长记录比较，Slot2对应的记录的 c1=8，因为 8>7(目标记录c1=3)，去 Slot0 到 Slot2中找
2. (0+2)/2 = 1 ，用Slot1记录的c1=4 进行比较，发现 4<7。此时发现不能继续二分了，就从Slot1对应的记录开始，通过 next_record去找对应的目标记录，变量3条记录后，可以找到目录记录c1=7。

# 4. Page Header

存储数据页中的记录的状态信息

| 名称 | 占用空间大小 | 描述 |
| --- | --- | --- |
| PAGE_N_DIR_SLOTS | 2字节 | 目录页中槽的数量 |
| PAGE_HEAP_TOP | 2字节 | 未使用的空间最小地址，从该地址开始之后就是Free Space |
| PAGE_N_HEAP | 2字节 | 第1位表示本记录是否为紧凑型的记录，剩余的15位表示本页的堆中记录的数量（包括infimum、supremum和被删除的记录） |
| PAGE_FREE | 2字节 | 表示垃圾链表的表头结点在页面中的偏移位置 |
| PAGE_GARBAGE | 2字节 | 已删除的记录占用的字节数 |
| PAGE_LAST_INSERT | 2字节 | 最后插入记录的位置 |
| PAGE_DIRECTION | 2字节 | 记录插入的方向 |
| PAGE_N_DIRECTION | 2字节 | 一个方向连续插入的记录数量 |
| PAGE_N_RECS | 2字节 | 该页中用户记录的数量（不包括infimum和supremum记录以及被删除的记录） |
| PAGE_MAX_TRX_ID | 8字节 | 当前页中的最大事务id |
| PAGE_LEVEL | 2字节 | 当前页中B+树所处的层级 |
| PAGE_INDEX_ID | 8字节 | 索引ID，表示当前页属于那个索引 |
| PAGE_BTR_SEG_LEAF | 10字节 | B+树叶子结点段的头部信息，仅在B+树的根页面中定义 |
| PAGE_BTR_SEG_TOP | 10字节 | B+树非叶子结点段的头部信息，仅在B+树的根页面中定义 |
- `PAGE_DIRECTION` ：假如插入的一条记录的主键比上一条记录的主键值大，我们说这条记录插入的方向是右边，反之则是左边。
- `PAGE_N_DIRECTION` ：如果连续插入新纪录的方向是一致的，会把同一个方向插入记录的条数记录下来。方向发生改变会清零重新统计。

# 5. File Header

描述了一些通用于各种页的信息。

| 名称 | 占用空间大小 | 描述 |
| --- | --- | --- |
| FIL_PAGE_SPACE_OR_CHECKSUM | 4字节 | 表示页的校验和（checksum) |
| FIL_PAGE_OFFSET | 4字节 | 页号 |
| FIL_PAGE_PREV | 4字节 | 上一页的页号 |
| FIL_PAGE_NEXT | 4字节 | 下一个页的页号 |
| FIL_PAGE_LSN | 8字节 | 仅在系统表的第一个页中定义，代表文件至少被刷新到了对应的LSN值 |
| FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID | 4字节 | 页属于哪个表空间 |
- `FIL_PAGE_PREV` 和 `FIL_PAGE_NEXT` 是双向链表用于链接两个页。
- `FIL_PAGE_TYPE` 表示当前页的类型，如下是其他类型的页

## File Page Type

| 名称 | 十六进制 | 描述 |
| --- | --- | --- |
| FILE_PAGE_TYPE_ALLOCATED | 0x0000 | 最新分配，还未使用 |
| FIL_PAGE_UNDO_LOG | 0x0002 | undo 日志页 |
| FIL_PAGE_INODE | 0x0003 | 存储段的信息 |
| FIL_PAGE_IBUF_FREE_LIST | 0x0004 | Change Buffer 空闲列表 |
| FIL_PAGE_IBUF_BITMAP | 0x0005 | Change Buffer 的一些属性 |
| FIL_PAGE_TYPE_SYS | 0x0006 | 存储一些系统数据 |
| FIL_PAGE_TYPE_TRX_SYS | 0x0007 | 事务系统数据 |
| FIL_PAGE_TYPE_FSP_HDR | 0x0008 | 表空间头部信息 |
| FIL_PAGE_TYPE_BLOB | 0x000A | 溢出页 |
| FIL_PAGE_INDEX | 0x45BF | 索引页，也就是我们所说的数据页 |

# 6. File Trailer

为了检测在将页刷到磁盘的过程，是否出现了只刷了一半到（未刷写完整），由两部分组成

- 前4字节：代表页的校验和
- 后4字节：代表页面被最后修改时对应的LSN的最后4字节，正常与 `File Header` 部分的 `FIL_PAGE_LSN` 的后4字节相同。

# 参考资料

---

1. 《MySQL是怎么运行的》— 小孩子4919