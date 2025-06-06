# 一、逻辑存储结构

InnoDB 的逻辑存储结构如下图所示：

![image1](assets/image1.jpeg)

1、表空间

表空间是 InnoDB 存储引擎逻辑结构的最高层， 如果用户启用了参数 innodb_file_per_table（在 8.0 版本中默认开启），则每张表都会有一个表空间（xxx.ibd），一个 MySQL 实例可以对应多个表空间，用于存储记录、索引等数据。

2、段

段分为数据段（Leaf node segment）、索引段（Non-leaf node segment）、回滚段（Rollback segment），InnoDB 是索引组织表，数据段就是 B+ 树的叶子节点，索引段即为 B+ 树的非叶子节点。段用来管理多个 Extent（区）。

3、区

区，表空间的单元结构，每个区的大小为 1M。默认情况下，InnoDB 存储引擎页大小为 16K，即一个区中一共有 64 个连续的页。

4、页

页，是 InnoDB 存储引擎磁盘管理的最小单元，每个页的大小默认为 16KB。为了保证页的连续性，InnoDB 存储引擎每次从磁盘申请 4~5 个区。

5、行

行，InnoDB 存储引擎数据是按行进行存放的。

在行中，默认有两个隐藏字段：

* Trx_id：最后一次操作事务的 id，每次对某条记录进行改动时，都会把对应的事务 id 赋值给 trx_id 隐藏列。
* Roll_pointer：每次对某条引记录进行改动时，都会把旧的版本写入到 undo 日志中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息。

# 二、架构

## 1、概述

从 MySQL 5.5 版本开始，默认使用 InnoDB 存储引擎，它擅长事务处理，具有崩溃恢复特性，在日常开发中使用非常广泛。下面是 InnoDB 架构图，左侧为内存结构，右侧为磁盘结构。

![image2](assets/image2.jpeg)

## 2、内存结构

![image3](assets/image3.jpeg)

在左侧的内存结构中，主要分为这么四大块儿：Buffer Pool、Change Buffer、Adaptive Hash Index、Log Buffer。接下来介绍一下这四个部分。

1、Buffer Pool

InnoDB 存储引擎基于磁盘文件存储，访问物理硬盘和在内存中进行访问，速度相差很大，为了尽可能弥补这两者之间的 I/O 效率的差值，就需要把经常使用的数据加载到缓冲池中，避免每次访问都进行磁盘 I/O。

在 InnoDB 的缓冲池中不仅缓存了索引页和数据页，还包含了 undo 页、插入缓存、自适应哈希索引以及 InnoDB 的锁信息等等。

缓冲池 Buffer Pool，是主内存中的一个区域，里面可以缓存磁盘上经常操作的真实数据，在执行增删改查操作时，先操作缓冲池中的数据（若缓冲池没有数据，则从磁盘加载并缓存），然后再以一定频率刷新到磁盘，从而减少磁盘 I/O，加快处理速度。

缓冲池以 Page 页为单位，底层采用链表数据结构管理 Page。根据状态，将 Page 分为三种类型：

* free page：空闲 page，未被使用。
* clean page：被使用 page，数据没有被修改过。
* dirty page：脏页，被使用 page，数据被修改过，其中数据与磁盘的数据产生了不一致。

在专用服务器上，通常将多达 80％ 的物理内存分配给缓冲池。

参数查看：`show variables like 'innodb_buffer_pool_size';`

![image4](assets/image4.jpeg)

2、Change Buffer

Change Buffer，更改缓冲区（针对于非唯一二级索引页），在执行 DML 语句时，如果这些数据 Page 没有在 Buffer Pool 中，不会直接操作磁盘，而会将数据变更存在更改缓冲区 Change Buffer 中，在未来数据被读取时，再将数据合并恢复到 Buffer Pool 中，再将合并后的数据刷新到磁盘中。

Change Buffer 的意义是什么呢？

先来看一幅图，这个是二级索引的结构图：

![image5](assets/image5.jpeg)

与聚集索引不同，二级索引通常是非唯一的，并且以相对随机的顺序插入二级索引。同样，删除和更新可能会影响索引树中不相邻的二级索引页，如果每一次都操作磁盘，会造成大量的磁盘 I/O。有了 ChangeBuffer 之后，我们可以在缓冲池中进行合并处理，减少磁盘 I/O。

3、Adaptive Hash Index

自适应 hash 索引，用于优化对 Buffer Pool 数据的查询。MySQL 的 InnoDB 引擎中虽然没有直接支持 hash 索引，但是给我们提供了一个功能就是这个自适应 hash 索引。因为前面我们讲到过，hash 索引在进行等值匹配时，一般性能是要高于 B+ 树的，因为 hash 索引一般只需要一次 I/O 即可，而 B+ 树可能需要几次匹配，所以 hash 索引的效率要高，但是 hash 索引又不适合做范围查询、模糊匹配等。

InnoDB 存储引擎会监控对表上各索引页的查询，如果观察到在特定的条件下 hash 索引可以提升速度，则建立 hash 索引，称之为自适应 hash 索引。

**自适应哈希索引，无需人工干预，是系统根据情况自动完成**。

参数：adaptive_hash_index

4、Log Buffer

Log Buffer：日志缓冲区，用来保存要写入到磁盘中的 log 日志数据（redo log、undo log），默认大小为 16MB，日志缓冲区的日志会定期刷新到磁盘中。如果需要更新、插入或删除许多行的事务，增加日志缓冲区的大小可以节省磁盘 I/O。

参数：

* innodb_log_buffer_size：缓冲区大小
* innodb_flush_log_at_trx_commit：日志刷新到磁盘时机，取值主要包含以下三个：
  * 1：日志在每次事务提交时写入并刷新到磁盘，默认值。
  * 0：每秒将日志写入并刷新到磁盘一次。
  * 2：日志在每次事务提交后写入，并每秒刷新到磁盘一次。

![image6](assets/image6.jpeg)

## 3、磁盘结构

接下来，再来看看 InnoDB 体系结构的右边部分，也就是磁盘结构：

![image7](assets/image7.jpeg)

1、System Tablespace

系统表空间是更改缓冲区的存储区域。如果表是在系统表空间而不是每个表文件或通用表空间中创建的，它也可能包含表和索引数据。（在 MySQL 5.x 版本中还包含 InnoDB 数据字典、undolog 等）

参数：`innodb_data_file_path`

![image8](assets/image8.jpeg)

系统表空间，默认的文件名叫 ibdata1。

2、File-Per-Table Tablespaces

独立表空间，如果开启了 `innodb_file_per_table` 开关，则每个表的文件表空间包含单个 InnoDB 表的数据和索引，并存储在文件系统上的单个数据文件中。

开关参数：`innodb_file_per_table`，该参数默认开启。

![image9](assets/image9.jpeg)

那也就是说，我们每创建一个表，都会产生一个表空间文件，如图：

![image10](assets/image10.jpeg)

3、General Tablespaces

通用表空间，需要通过 `CREATE TABLESPACE` 语法创建通用表空间，在创建表时，可以指定该表空间。

* 创建表空间

`CREATE TABLESPACE ts_name ADD DATAFILE 'file_name' ENGINE = engine_name;`

![image11](assets/image11.jpeg) 

* 创建表时指定表空间

`CREATE TABLE xxx ... TABLESPACE ts_name;`

![image12](assets/image12.jpeg)

4、Undo Tablespaces

撤销表空间，MySQL 实例在初始化时会自动创建两个默认的 undo 表空间（初始大小 16M），用于存储 undo log 日志。

5、Temporary Tablespaces

InnoDB 使用会话临时表空间和全局临时表空间。存储用户创建的临时表等数据。

6、Doublewrite Buffer Files

双写缓冲区，InnoDB 引擎将数据页从 Buffer Pool 刷新到磁盘前，先将数据页写入双写缓冲区文件中，便于系统异常时恢复数据。

![image13](assets/image13.jpeg)

7、Redo Log

重做日志，是用来实现事务的持久性。该日志文件由两部分组成：重做日志缓冲（redo log buffer）以及重做日志文件（redo log），前者是在内存中，后者在磁盘中。当事务提交之后会把所有修改信息都会存到该日志中，用于在刷新脏页到磁盘时，发生错误时，进行数据恢复使用。

以循环方式写入重做日志文件，涉及两个文件：

![image14](assets/image14.jpeg)

前面我们介绍了 InnoDB 的内存结构，以及磁盘结构，那么内存中我们所更新的数据，又是如何到磁盘中的呢？此时就涉及到一组后台线程，接下来，就来介绍一些 InnoDB 中涉及到的后台线程。

![image15](assets/image15.jpeg)

## 4、后台线程

![image16](assets/image16.png)

在 InnoDB 的后台线程中，分为 4 类，分别是：Master Thread、IO Thread、Purge Thread、Page Cleaner Thread。

1、Master Thread

核心后台线程，负责调度其他线程，还负责将缓冲池中的数据异步刷新到磁盘中，保持数据的一致性，还包括脏页的刷新、合并插入缓存、undo 页的回收。

2、IO Thread

在 InnoDB 存储引擎中大量使用了 AIO 异步非阻塞 I/O 来处理 I/O 请求，这样可以极大地提高数据库的性能，而 I/O Thread 主要负责这些 I/O 请求的回调。

| 线程类型             | 默认个数 | 职责                         |
| -------------------- | -------- | ---------------------------- |
| Read thread          | 4        | 负责读操作                   |
| Write thread         | 4        | 负责写操作                   |
| Log thread           | 1        | 负责将日志缓冲区刷新到磁盘   |
| Insert buffer thread | 1        | 负责将写缓冲区内容刷新到磁盘 |

我们可以通过以下的这条指令，查看到 InnoDB 的状态信息，其中就包含 IO Thread 信息。

`show engine innodb status \G;`

![image17](assets/image17.jpeg)

3、Purge Thread

主要用于回收事务已经提交了的 undo log，在事务提交之后，undo log 可能不用了，就用它来回收。

4、Page Cleaner Thread

协助 Master Thread 刷新脏页到磁盘的线程，它可以减轻 Master Thread 的工作压力，减少阻塞。

# 三、事务原理

## 1、事务基础

1、概念

事务 是一组操作的集合，它是一个不可分割的工作单位，事务会把所有的操作作为一个整体一起向系统提交或撤销操作请求，即这些操作要么同时成功，要么同时失败。

2、特性

* 原子性（Atomicity）：事务是不可分割的最小操作单元，要么全部成功，要么全部失败。
* 一致性（Consistency）：事务完成时，必须使所有的数据都保持一致状态。
* 隔离性（Isolation）：数据库系统提供的隔离机制，保证事务在不受外部并发操作影响的独立环境下运行。
* 持久性（Durability）：事务一旦提交或回滚，它对数据库中的数据的改变就是永久的。

那实际上，我们研究事务的原理，就是研究 MySQL 的 InnoDB 引擎是如何保证事务的这四大特性的。

![image18](assets/image18.jpeg)

而对于这四大特性，实际上分为两个部分。其中的原子性、一致性、持久化，实际上是由 InnoDB 中的两份日志来保证的，一份是 redo log 日志，一份是 undo log 日志。而持久性是通过数据库的锁，加上 MVCC 来保证的。

![image19](assets/image19.jpeg)

我们在讲解事务原理的时候，主要就是来研究一下 redolog，undolog 以及 MVCC。

## 2、redo log

重做日志，记录的是事务提交时数据页的物理修改，是用来实现事务的持久性。

该日志文件由两部分组成：重做日志缓冲（redo log buffer）以及重做日志文件（redo log file），前者是在内存中，后者在磁盘中。当事务提交之后会把所有修改信息都存到该日志文件中，用于在刷新脏页到磁盘，发生错误时，进行数据恢复使用。

如果没有 redolog，可能会存在什么问题呢？我们一起来分析一下。

我们知道，在 InnoDB 引擎中的内存结构中，主要的内存区域就是缓冲池，在缓冲池中缓存了很多的数据页。当我们在一个事务中，执行多个增删改的操作时，InnoDB 引擎会先操作缓冲池中的数据，如果缓冲区没有对应的数据，会通过后台线程将磁盘中的数据加载出来，存放在缓冲区中，然后将缓冲池中的数据修改，修改后的数据页我们称为脏页。 而脏页则会在一定的时机，通过后台线程刷新到磁盘中，从而保证缓冲区与磁盘的数据一致。 而缓冲区的脏页数据并不是实时刷新的，而是一段时间之后将缓冲区的数据刷新到磁盘中，假如刷新到磁盘的过程出错了，而提示给用户事务提交成功，而数据却没有持久化下来，这就出现问题了，没有保证事务的持久性。

![image20](assets/image20.jpeg)

那么，如何解决上述的问题呢？在 InnoDB 中提供了一份日志 redo log，接下来我们再来分析一下，通过 redolog 如何解决这个问题。

![image21](assets/image21.jpeg)

有了 redolog 之后，当对缓冲区的数据进行增删改之后，会首先将操作的数据页的变化，记录在 redo log buffer 中。在事务提交时，会将 redo log buffer 中的数据刷新到 redo log 磁盘文件中。过一段时间之后，如果刷新缓冲区的脏页到磁盘时，发生错误，此时就可以借助于 redo log 进行数据恢复，这样就保证了事务的持久性。而如果脏页成功刷新到磁盘或者涉及到的数据已经落盘，此时 redolog 就没有作用了，就可以删除了，所以存在的两个 redolog 文件是循环写的。

那为什么每一次提交事务，要刷新 redo log 到磁盘中呢，而不是直接将 buffer pool 中的脏页刷新到磁盘呢？

因为在业务操作中，我们操作数据一般都是随机读写磁盘的，而不是顺序读写磁盘。而 redo log 在往磁盘文件中写入数据，由于是日志文件，所以都是顺序写的。顺序写的效率，要远大于随机写。这种先写日志的方式，称之为 WAL（Write-Ahead Logging）。

## 3、undo log

回滚日志，用于记录数据被修改前的信息，作用包含两个：提供回滚（保证事务的原子性）和 MVCC（多版本并发控制）。

undo log 和 redo log 记录物理日志不一样，它是逻辑日志。可以认为当 delete 一条记录时，undo log 中会记录一条对应的 insert 记录，反之亦然，当 update 一条记录时，它记录一条对应相反的 update 记录。当执行 rollback 时，就可以从 undo log 中的逻辑记录读取到相应的内容并进行回滚。

* Undo log 销毁：undo log 在事务执行时产生，事务提交时，并不会立即删除 undo log，因为这些日志可能还用于 MVCC。

* Undo log 存储：undo log 采用段的方式进行管理和记录，存放在前面介绍的 rollback segment 回滚段中，内部包含 1024 个 undo log segment。

# 四、MVCC

## 1、基本概念

1、当前读

读取的是记录的最新版本，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加锁。对于我们日常的操作，如：`select ... lock in share mode`（共享锁），`select ... for update`、update、insert、delete（排他锁）都是一种当前读。

测试：

![image22](assets/image22.jpeg)

在测试中我们可以看到，即使是在默认的 RR 隔离级别下，事务 A 中依然可以读取到事务 B 最新提交的内容，因为在查询语句后面加上了 lock in share mode 共享锁，此时是当前读操作。当然，当我们加排他锁的时候，也是当前读操作。

2、快照读

简单的 select（不加锁）就是快照读，快照读，读取的是记录数据的可见版本，有可能是历史数据，不加锁，是非阻塞读。

* Read Committed：每次 select，都生成一个快照读。
* Repeatable Read：开启事务后第一个 select 语句才是快照读的地方。
* Serializable：快照读会退化为当前读。

测试：

![image23](assets/image23.jpeg)

在测试中，我们看到即使事务 B 提交了数据，事务 A 中也查询不到。原因就是因为普通的 select 是快照读，而在当前默认的 RR 隔离级别下，开启事务后第一个 select 语句才是快照读的地方，后面执行相同的 select 语句都是从快照中获取数据，可能不是当前的最新数据，这样也就保证了可重复读。

3、MVCC

全称 Multi-Version Concurrency Control，多版本并发控制。指维护一个数据的多个版本，使得读写操作没有冲突，快照读为 MySQL 实现 MVCC 提供了一个非阻塞读功能。MVCC 的具体实现，还需要依赖于数据库记录中的三个隐式字段、undo log 日志、ReadView。

接下来，我们再来介绍一下 InnoDB 引擎的表中涉及到的隐藏字段 、undolog 以及 ReadView，从而来介绍一下 MVCC 的原理。

## 2、隐藏字段

1、介绍

![image24](assets/image24.jpeg)

当我们创建了上面的这张表，我们在查看表结构的时候，就可以显式的看到这三个字段。实际上除了这三个字段以外，InnoDB 还会自动地给我们添加三个隐藏字段及其含义分别是：

| 隐藏字段    | 含义                                                         |
| ----------- | ------------------------------------------------------------ |
| DB_TRX_ID   | 最近修改事务 ID，记录插入这条记录或最后一次修改该记录的事务 ID。 |
| DB_ROLL_PTR | 回滚指针，指向这条记录的上一个版本，用于配合 undo log，指向上一个版本。 |
| DB_ROW_ID   | 隐藏主键，如果表结构没有指定主键，将会生成该隐藏字段。       |

而上述的前两个字段是肯定会添加的，是否添加最后一个字段 DB_ROW_ID，得看当前表有没有主键，如果有主键，则不会添加该隐藏字段。

2、测试

（1）查看有主键的表 stu

进入服务器中的 `/var/lib/mysql/itcast/`，查看 stu 的表结构信息，通过如下指令：

`ibd2sdi stu.ibd`

查看到的表结构信息中有一栏 columns，在其中我们会看到除了我们建表时指定的字段以外，还有额外的两个字段 分别是：DB_TRX_ID 、DB_ROLL_PTR，因为该表有主键，所以没有 DB_ROW_ID 隐藏字段。

![image25](assets/image25.png)



![image26](assets/image26.png)

（2）查看没有主键的表 employee

建表语句：

```mysql
create table employee(id int, name varchar(10));
```

此时，我们再通过以下指令来查看表结构及其其中的字段信息：

`ibd2sdi employee.ibd`

查看到的表结构信息中，有一栏 columns，在其中我们会看到处理我们建表时指定的字段以外，还有额外的三个字段，分别是：DB_TRX_ID、DB_ROLL_PTR、DB_ROW_ID，因为 employee 表是没有指定主键的。

![image27](assets/image27.png)



![image28](assets/image28.png)



![image29](assets/image29.png)

## 3、undolog

### 介绍

回滚日志，在 insert、update、delete 的时候产生的便于数据回滚的日志。

当 insert 的时候，产生的 undo log 日志只在回滚时需要，在事务提交后，可被立即删除。

而 update、delete 的时候，产生的 undo log 日志不仅在回滚时需要，在快照读时也需要，不会立即被删除。

### 版本链

有一张表原始数据为：

![image30](assets/image30.png)

* DB_TRX_ID：代表最近修改事务 ID，记录插入这条记录或最后一次修改该记录的事务 ID，是自增的
* DB_ROLL_PTR：由于这条数据是才插入的，没有被更新过，所以该字段值为 null。

然后，有四个并发事务同时在访问这张表。

（1）第一步

![image31](assets/image31.jpeg)

当事务 2 执行第一条修改语句时，会记录 undo log 日志，记录数据变更之前的样子。然后更新记录，并且记录本次操作的事务 ID，回滚指针，回滚指针用来指定如果发生回滚，回滚到哪一个版本。

![image32](assets/image32.png)

（2）第二步

![image33](assets/image33.jpeg)

当事务 3 执行第一条修改语句时，也会记录 undo log 日志，记录数据变更之前的样子。然后更新记录，并且记录本次操作的事务 ID，回滚指针，回滚指针用来指定如果发生回滚，回滚到哪一个版本。

![image34](assets/image34.jpeg)

（3）第三步

![image35](assets/image35.jpeg)

当事务 4 执行第一条修改语句时，也会记录 undo log 日志，记录数据变更之前的样子；然后更新记录，并且记录本次操作的事务 ID，回滚指针，回滚指针用来指定如果发生回滚，回滚到哪一个版本。

![image36](assets/image36.png)

> 最终我们发现，不同事务或相同事务对同一条记录进行修改，会导致该记录的 undolog 生成一条记录版本链表，链表的头部是最新的旧记录，链表尾部是最早的旧记录。

## 4、ReadView

ReadView（读视图）是 快照读 SQL 执行时 MVCC 提取数据的依据，记录并维护系统当前活跃的事务（未提交的）id。

ReadView 中包含了四个核心字段：

| 字段           | 含义                                                       |
| -------------- | ---------------------------------------------------------- |
| m_ids          | 当前活跃的事务 ID 集合                                     |
| min_trx_id     | 最小活跃事务 ID                                            |
| max_trx_id     | 预分配事务 ID，当前最大事务 ID + 1（因为事务 ID 是自增的） |
| creator_trx_id | ReadView 创建者的事务 ID                                   |

而在 ReadView 中就规定了版本链数据的访问规则：

trx_id 代表当前 undolog 版本链对应事务 ID。

| 条件                               | 是否可以访问                                  | 说明                                          |
| ---------------------------------- | --------------------------------------------- | --------------------------------------------- |
| trx_id == creator_trx_id           | 可以访问该版本                                | 成立，说明数据是当前这个事务更改的。          |
| trx_id < min_trx_id                | 可以访问该版本                                | 成立，说明数据已经提交了。                    |
| trx_id > max_trx_id                | 不可以访问该版本                              | 成立，说明该事务是在  ReadView 生成后才开启。 |
| min_trx_id <= trx_id <= max_trx_id | 如果 trx_id 不在 m_ids 中，是可以访问该版本的 | 成立，说明数据已经提交。                      |

不同的隔离级别，生成 ReadView 的时机不同：

* READ COMMITTED：在事务中每一次执行快照读时生成 ReadView。
* REPEATABLE READ：仅在事务中第一次执行快照读时生成 ReadView，后续复用该 ReadView。

## 5、原理分析

### RC 隔离级别

RC 隔离级别下，在事务中每一次执行快照读时生成 ReadView。

我们就来分析事务 5 中，两次快照读读取数据，是如何获取数据的？

在事务 5 中，查询了两次 id 为 30 的记录，由于隔离级别为 Read Committed，所以每一次进行快照读都会生成一个 ReadView，那么两次生成的 ReadView 如下。

![image37](assets/image37.jpeg)

那么这两次快照读在获取数据时，就需要根据所生成的 ReadView 以及 ReadView 的版本链访问规则，到 undolog 版本链中匹配数据，最终决定此次快照读返回的数据。

1、先来看第一次快照读具体的读取过程：

![image38](assets/image38.jpeg)

![image39](assets/image39.jpeg)

在进行匹配时，会从 undo log 的版本链，从上到下进行挨个匹配：

（1）先匹配第一条记录，这条记录对应的 trx_id 为 4，也就是将 4 带入右侧的匹配规则中。① ② ③ ④ 都不满足，则继续匹配 undo log 版本链的下一条。

![image40](assets/image40.jpeg)

（2）再匹配第二条记录，这条记录对应的 trx_id 为 3，也就是将 3 带入右侧的匹配规则中。① ② ③ ④ 都不满足，则继续匹配 undo log 版本链的下一条。

![image41](assets/image41.jpeg)

（3）再匹配第三条记录，这条记录对应的 trx_id 为 2，也就是将 2 带入右侧的匹配规则中。① 不满足，② 满足，终止匹配，此次快照读，返回的数据就是版本链中记录的这条数据。

![image42](assets/image42.jpeg)

2、再来看第二次快照读具体的读取过程：

![image43](assets/image43.jpeg)

![image44](assets/image44.jpeg)

在进行匹配时，会从 undo log 的版本链，从上到下进行挨个匹配：

（1）先匹配第一条记录，这条记录对应的 trx_id 为 4，也就是将 4 带入右侧的匹配规则中。① ② ③ ④ 都不满足，则继续匹配 undo log 版本链的下一条。

![image40](assets/image40.jpeg)

（2）再匹配第二条记录，这条记录对应的 trx_id 为 3，也就是将 3 带入右侧的匹配规则中。① 不满足，② 满足，终止匹配。此次快照读，返回的数据就是版本链中记录的这条数据。

![image41](assets/image41.jpeg)

### RR 隔离级别

RR 隔离级别下，仅在事务中第一次执行快照读时生成 ReadView，后续复用该 ReadView。而 RR 是可重复读，在一个事务中，执行两次相同的 select 语句，查询到的结果是一样的。

那 MySQL 是如何做到可重复读的呢？我们简单分析一下就知道了

![image45](assets/image45.jpeg)

我们看到，在 RR 隔离级别下，只是在事务中第一次快照读时生成 ReadView，后续都是复用该 ReadView，那么既然 ReadView 都一样，ReadView 的版本链匹配规则也一样，那么最终快照读返回的结果也是一样的。

> 所以，MVCC 的实现原理就是通过 InnoDB 表的隐藏字段、UndoLog 版本链、ReadView 来实现的。而 MVCC + 锁，则实现了事务的隔离性。而一致性则是由 redolog 与 undolog 保证。

![image46](assets/image46.jpeg)

