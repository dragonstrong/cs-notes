[toc]



# 第16章_多版本并发控制

## 1. 什么是MVCC

MVCC （Multiversion Concurrency Control），多版本并发控制。顾名思义，MVCC 是通过数据行的多个版本管理来实现数据库的 <font color=orange>**并发控制**</font> 。这项技术使得在InnoDB的事务隔离级别下执行 <font color=orange>**一致性读**</font>操作有了保证。换言之，就是为了查询一些正在被另一个事务更新的行，并且可以看到它们被更新之前的值，这样 在做查询的时候就不用等待另一个事务释放锁。

MVCC没有正式的标准，在不同的DBMS中MVCC的实现方式可能是不同的，也不是普遍使用的（大家可以参考相关的DBMS文档）。这里讲解InnoDB中MVCC的实现机制（MySQL其他的存储引擎并不支持它）。

## 2. 快照读与当前读

MVCC在MySQL InnoDB中的实现主要是为了提高数据库并发性能，用更好的方式去处理 <font color=orange>**读-写冲突**</font>，做到 即使有读写冲突时，也能做到 <font color=orange>**不加锁 ， 非阻塞并发读**</font> ，而这个读指的就是 `快照读` , 而非 `当前读` 。当前 读实际上是一种加锁的操作，是悲观锁的实现。而MVCC本质是采用乐观锁思想的一种方式。

### 2.1 快照读

快照读又叫一致性读，读取的是快照数据。**<font color=orange>不加锁的简单的 SELECT 都属于快照读</font>**，即不加锁的非阻塞 读；比如这样：

```mysql
SELECT * FROM player WHERE ...
```

之所以出现快照读的情况，是基于提高并发性能的考虑，快照读的实现是基于MVCC，它在很多情况下， 避免了加锁操作，降低了开销。

既然是基于多版本，那么快照读可能读到的并不一定是数据的最新版本，而有可能是之前的历史版本。 

快照读的前提是隔离级别不是串行级别，串行级别下的快照读会退化成当前读。

### 2.2 当前读

<font color=orange>**当前读读取的是记录的最新版本**</font>（最新数据，而不是历史版本的数据），读取时还要保证其他并发事务 不能修改当前记录，会对读取的记录进行加锁。加锁的 SELECT，或者对数据进行增删改都会进行当前 读。比如：

```mysql
SELECT * FROM student LOCK IN SHARE MODE; # 共享锁
SELECT * FROM student FOR UPDATE; # 排他锁
INSERT INTO student values ... # 排他锁
DELETE FROM student WHERE ... # 排他锁
UPDATE student SET ... # 排他锁
```

## 3. 复习

### 3.1 再谈隔离级别

我们知道事务有 4 个隔离级别，可能存在三种并发问题：

![image-20220714140441064](assets/image-20220714140441064.png)

在MySQL中，默认的隔离级别是可重复读，可以解决脏读和不可重复读的问题，如果仅从定义的角度来看，它并 不能解决幻读问题。如果想要解决幻读问题，就需要采用串行化的方式，也就是将隔离级别提升到最高，但这样会大幅降低数据库的事务并发能力。MVCC可以不采用锁机制，而是通过乐观锁的方式来解决不可重复读和幻读问题！它可以在大多数情况下替代行级锁，降低系统的开销。

**注意：**

- <font color=red size=5>**MySQL 可重复读隔离级别 使用"MVCC+临键锁" 可以避免幻读**</font>
- <font color=red size=5>**但它和串行化又有区别，因为串行化是全部使用加锁机制来避免的。**</font>
- <font color=red size=5>**MVCC仅用于读已提交、可重复读这两种隔离级别**</font>

![image-20220714140541555](assets/image-20220714140541555.png)

### 3.2 隐藏字段、Undo Log版本链

回顾一下undo日志的版本链，对于使用 InnoDB 存储引擎的表来说，它的聚簇索引记录中都包含两个必要的隐藏列。

* <font color=orange>**trx_id**</font> ：每次一个事务对某条聚簇索引记录进行改动时，都会把该事务的 <font color=orange>**事务id**</font> 赋值给 `trx_id` 隐藏列。 
* <font color=orange>**roll_pointer**</font> ：每次对某条聚簇索引记录进行改动时，都会把旧的版本写入到 <font color=orange>**undo日志**</font>中，然 后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息。

<img src="assets/image-20220714140716427.png" alt="image-20220714140716427" style="float:left;" />

假设插入该记录的`事务id`为`8`，那么此刻该条记录的示意图如下所示：

![image-20220714140801595](assets/image-20220714140801595.png)

> insert undo只在事务回滚时起作用，当事务提交后，该类型的undo日志就没用了，它占用的Undo Log Segment也会被系统回收（也就是该undo日志占用的Undo页面链表要么被重用，要么被释放）。

假设之后两个事务id分别为 `10` 、 `20` 的事务对这条记录进行` UPDATE` 操作，操作流程如下：

![image-20220714140846658](assets/image-20220714140846658.png)

能不能在两个事务中交叉更新同一条记录呢？不能！这不就是一个事务修改了另一个未提交事务修改过的数据，<font color=orange>**脏写**</font>。InnoDB使用锁来保证不会有脏写情况的发生，也就是在<font color=orange>**第一个事务更新了某条记录后，就会给这条记录加锁**</font>，另一个事务再次更新时就需要等待第一个事务提交了，把锁释放之后才可以继续更新.



每次对记录进行改动，都会记录一条undo日志，每条undo日志也都有一个 `roll_pointer` 属性 （ `INSERT` 操作对应的undo日志没有该属性，因为该记录并没有更早的版本），可以将这些 `undo日志` 都连起来，串成一个链表：

![image-20220714141012874](assets/image-20220714141012874.png)

对该记录每次更新后，都会将旧值放到一条 `undo日志` 中，就算是该记录的一个旧版本，随着更新次数 的增多，所有的版本都会被 `roll_pointer` 属性连接成一个链表，我们把这个链表称之为 `版本链` ，版 本链的头节点就是当前记录最新的值。

每个版本中还包含生成该版本时对应的` 事务id `。

## 4. MVCC实现原理之ReadView

MVCC 的实现依赖于：<font color=orange>**隐藏字段、Undo Log（多版本）、Read View(想让会话看到哪个版本)**</font>。

### 4.1 什么是ReadView

在MVCC机制中，多个事务对同一个行记录进行更新会产生多个历史快照，这些历史快照保存在Undo Log里。如 果一个事务想要查询这个行记录，<font color=orange>**需要读取哪个版本的行记录**</font>呢？这时就需要用到ReadView 了，它帮我们解决 了行的可见性问题。

<font color=orange>**ReadView**</font>就是事务在使用MVCC机制进行快照读操作时产生的<font color=orange>**读视图**</font>。<font color=orange>**当事务启动时，会生成数据库系统当前的一个快照**</font>，InnoDB为每个事务构造了一个数组，用来记录并维护系统当前<font color=orange>**活跃事务**</font>的ID （"活跃" 指的就是**<font color=orange>启动 了但还没提交，已提交的事务不会出现在里面</font>**）。

### 4.2 设计思路

使用 `READ UNCOMMITTED` 隔离级别的事务，由于可以读到未提交事务修改过的记录，所以直接读取记录的最新版本就好了。

使用 `SERIALIZABLE` 隔离级别的事务，InnoDB规定使用加锁的方式来访问记录。

<font color=red size=5>**MVCC仅用于读已提交、可重复读这两种隔离级别，其他用不着**</font>

使用 <font color=orange>**READ COMMITTED 和 REPEATABLE READ**</font>隔离级别的事务，都必须保证读到 `已经提交了的` 事务修改过的记录。假如另一个事务已经修改了记录但是尚未提交，是不能直接读取最新版本的记录的，核心问题就是需要判断一下版本链中的哪个版本是当前事务可见的，这是ReadView要解决的主要问题。

这个ReadView中主要包含4个比较重要的内容，分别如下：

1. <font color=orange>**creator_trx_id**</font>，创建这个 Read View 的事务 ID。

   > 说明：只有在对表中的记录做改动时（执行INSERT、DELETE、UPDATE这些语句时）才会为 事务分配事务id，否则在一个只读事务中的事务id值都默认为0。

2. <font color=orange>**trx_ids**</font> ，表示在生成ReadView时当前系统中活跃的读写事务的 <font color=orange>**事务id列表**</font> 。

3. <font color=orange>**up_limit_id**</font> ，活跃的事务中最小的事务 ID。

4. <font color=orange>**low_limit_id**</font> ，表示生成ReadView时系统中应该分配给下一个事务的 id 值。low_limit_id 是系 统最大的事务id值，这里要注意是系统中的事务id，需要区别于正在活跃的事务ID。

> 注意：low_limit_id并不是trx_ids中的最大值，事务id是递增分配的。比如，现在有id为1， 2，3这三个事务，之后id为3的事务提交了。那么一个新的读事务在生成ReadView时， trx_ids就包括1和2，up_limit_id的值就是1，low_limit_id的值就是4。

**举例：** 

trx_ids为trx2、trx3、trx5和trx8的集合，系统的最大事务ID （ low_limit_id） 为trx8+1 （ 如果之前没有其他的新增事务），活跃的最小事务ID （ up_limit_id） 为 trx2。

![image-20240717155754553](assets/image-20240717155754553.png)

### 4.3 ReadView的规则

有了这个ReadView，这样在访问某条记录时，只需要按照下边的步骤判断记录的某个版本是否可见。比如现在select * from student where id=1; 想判断到底读取哪一个版本？

* 如果被访问版本的trx_id属性值与ReadView中的 <font color=orange>**creator_trx_id**</font> 值相同，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问（自己修改，自己想查）。 

  下面都是想查其他事务修改的（<font color=orange>**creator_trx_id**</font> 值不同）

* 如果被访问版本的trx_id属性值小于ReadView中的 up_limit_id 值，表明生成该版本的事务在当前事务生成ReadView前已经提交，所以该版本可以被当前事务访问（事务id都是递增分配，比最小值都小，一定是已经提交过，已提交的版本可以访问）。 

* 如果被访问版本的trx_id属性值大于或等于ReadView中的 low_limit_id 值，表明生成该版本的事务在当前事务生成ReadView后才开启，所以该版本不可以被当前事务访问。 

* 如果被访问版本的trx_id属性值在ReadView的 up_limit_id 和 low_limit_id 之间，那就需要判断一下trx_id属性值是不是在 trx_ids 列表中。

  * 如果在，说明创建ReadView时生成该版本的事务还是活跃的，该版本不可以被访问。 
  * 如果不在，说明创建ReadView时生成该版本的事务已经被提交，该版本可以被访问。

### 4.4 MVCC整体操作流程

了解了这些概念之后，我们来看下当查询一条记录的时候，系统如何通过MVCC找到它：

1. 首先获取事务自己的版本号，也就是事务 ID； 
2. 获取 ReadView； 
3. 查询得到的数据，然后与 ReadView 中的事务版本号进行比较；
4. 如果不符合 ReadView 规则，就需要从 Undo Log 中获取历史快照； 
5. 最后返回符合规则的数据。

如果某个版本的数据对当前事务不可见的话，那就顺着版本链找到下一个版本的数据，继续按照上边的步骤判断 可见性，依此类推，直到版本链中的最后一个版本。如果最后一个版本也不可见的话，那么就意味着该条记录对 该事务完全不可见，查询结果就不包含该记录。

> [!NOTE]
>
> InnoDB中，MVCC是通过Undo Log + Read View进行数据读取，Undo Log保存了历史快照，而Read View规则帮我们判断当前版本的数据对该事务是否可见.



在隔离级别为读已提交（<font color=orange>**Read Committed**</font>）时，一个事务中的<font color=red>**每一次 SELECT 查询都会重新获取一次 Read View**</font>。

如表所示：

![image-20220715130843147](assets/image-20220715130843147.png)

> 注意，此时同样的查询语句都会重新获取一次 Read View，这时<font color=orange>**如果 Read View 不同，就可能产生不可重复读或者幻读**</font>的情况。

当隔离级别为<font color=orange>**可重复读**</font>的时候，就避免了不可重复读，这是因为一个事务<font color=red>**只在第一次 SELECT 的时候会获取一次 Read View**</font>，而后面所有的 SELECT 都会复用这个 Read View（所以不会出现不可重复读），如下表所示：

![image-20220715130916437](assets/image-20220715130916437.png)

## 5. 举例说明

假设现在student表中只有一条由事务id为8的事务插入的一条记录：

```mysql
mysql> select * from student;
+----+--------+--------+
| id | name   | class  |
+----+--------+--------+
|  1 | 张三   | 一班   |
+----+--------+--------+
1 row in set (0.00 sec)
```

MVCC只能在READ COMMITTED和REPEATABLE READ两个隔离级别下工作。接下来看一下READ COMMITTED和 REPEATABLE READ所谓的生成ReadView的时机不同到底不同在哪里。

### 5.1 READ COMMITTED隔离级别下

**<font color=red>READ COMMITTED ：每次读取数据前都生成一个ReadView</font>。**

现在有两个 `事务id` 分别为 `10` 、 `20` 的事务在执行:

```mysql
# Transaction 10
BEGIN;
UPDATE student SET name="李四" WHERE id=1;
UPDATE student SET name="王五" WHERE id=1;
# Transaction 20
BEGIN;
# 更新了一些别的表的记录
...
```

> 说明：事务执行过程中，只有在第一次真正修改记录时（比如使用INSERT、DELETE、UPDATE语句），才会被分配一个单独的事务id，这个事务id是递增的。所以我们才在事务2中更新一些别的表的记录，目的是让它分配事务id。

此刻，表student 中 id 为 1 的记录得到的版本链表如下所示：

![image-20220715133640655](assets/image-20220715133640655.png)

假设现在有一个使用 `READ COMMITTED` 隔离级别的事务开始执行：

```mysql
# 使用READ COMMITTED隔离级别的事务
BEGIN;

# SELECT1：Transaction 10、20未提交
SELECT * FROM student WHERE id = 1; # 得到的列name的值为'张三'
```

这个<font color=orange>**SELECT1**</font>的执行过程如下： 

步骤1：在执行SELECT语句时会先生成一个<font color=orange>**ReadView**</font>，ReadView的<font color=orange>**trx_ids**</font>列表的内容就是<font color=orange>**[10，20]**</font>， **<font color=orange>up_limit_id</font>** 为 <font color=orange>**10** </font>， <font color=orange>**low_limit_id**</font> 为<font color=orange> **21**</font>， <font color=orange>**creator_trx_id**</font> 为<font color=orange> **0**</font>。 

步骤2：从版本链中挑选可见的记录，从图中看出，最新版本的列<font color=orange>**name**</font>是<font color=orange>**'王五'**</font>，该版本的trx_id值为 <font color=orange>**10**</font>，<font color=orange>**在trx_ids列表内**</font>，不符合可见性要求，根据roll_pointer跳到下一个版本。

步骤3：下一个版本的列name是<font color=orange>**'李四'**</font>，该版本的trx_id值也为10 ，也<font color=orange>**在trx_ids列表内**</font>，不符合要求，继续跳到下一个版本。

步骤4：下一个版本的列name是<font color=orange>**'张三'**</font>，该版本的trx_id值为8 ，小于ReadView中的up_limit_id 值10，所以这个版本是<font color=orange>**符合要求**</font>的，最后返回给用户的版本就是这条列name为'张三' 的记录。



之后，我们把 `事务id` 为 `10` 的事务提交一下：

```mysql
# Transaction 10
BEGIN;
UPDATE student SET name="李四" WHERE id=1;
UPDATE student SET name="王五" WHERE id=1;
COMMIT;
```

然后再到 `事务id` 为 `20` 的事务中更新一下表 `student` 中 `id` 为 `1` 的记录：

```mysql
# Transaction 20
BEGIN;
# 更新了一些别的表的记录
...
UPDATE student SET name="钱七" WHERE id=1;
UPDATE student SET name="宋八" WHERE id=1;
```

此刻，表student中 `id` 为 `1` 的记录的版本链就长这样：

![image-20220715134839081](assets/image-20220715134839081.png)

然后再到刚才使用 `READ COMMITTED` 隔离级别的事务中继续查找这个 id 为 1 的记录，如下：

```mysql
# 使用READ COMMITTED隔离级别的事务
BEGIN;

# SELECT1：Transaction 10、20均未提交
SELECT * FROM student WHERE id = 1; # 得到的列name的值为'张三'

# SELECT2：Transaction 10提交，Transaction 20未提交
SELECT * FROM student WHERE id = 1; # 得到的列name的值为'王五'
```

这个<font color=orange>**SELECT2**</font>的执行过程如下： 

步骤1：在执行SELECT语句时<font color=red>**又会生成**</font>一个<font color=red>**ReadView**</font>，ReadView的<font color=orange>**trx_ids**</font>列表的内容就是<font color=red>**[20]**</font>， **<font color=orange>up_limit_id</font>** 为 <font color=orange>**20** </font>， <font color=orange>**low_limit_id**</font> 为<font color=orange> **21**</font>， <font color=orange>**creator_trx_id**</font> 为<font color=orange> **0**</font>。 

步骤2：从版本链中挑选可见的记录，从图中看出，最新版本的列<font color=orange>**name**</font>是<font color=orange>**'宋八'**</font>，该版本的trx_id值为 <font color=orange>**20**</font>，<font color=orange>**在trx_ids列表内**</font>，不符合可见性要求，根据roll_pointer跳到下一个版本。

步骤3：下一个版本的列<font color=orange>**name**</font>是<font color=orange>**'钱七'**</font>，该版本的trx_id值为 <font color=orange>**20**</font>，<font color=orange>**在trx_ids列表内**</font>，不符合可见性要求，根据roll_pointer跳到下一个版本。

步骤4：下一个版本的列<font color=orange>**name**</font>是<font color=orange>**'王五'**</font>，该版本的trx_id值为 <font color=orange>**10**</font>，小于ReadView中的up_limit_id 值20，所以这个版本是<font color=orange>**符合要求**</font>的，最后返回给用户的版本就是这条列name为'王五' 的记录。

以此类推，如果之后事务id为20的记录也提交了，再次在使用READ COMMITTED隔离级别的事务中查询表 student中id值为1的记录时，得到的结果就是'宋八'了，具体流程就不分析了。

> [!NOTE]
>
> 强调：<font color=red>**使用READ COMMITTED隔离级别的事务在每次查询开始时都会生成一个独立的ReadView**</font>。 



### 5.2 REPEATABLE READ隔离级别下

使用 `REPEATABLE READ` 隔离级别的事务来说，<font color=red>**只会在第一次执行查询语句时生成一个 ReadView ，之后的查询就不会重复生成了**</font>。

比如，系统里有两个 `事务id` 分别为 `10` 、 `20` 的事务在执行：

```mysql
# Transaction 10
BEGIN;
UPDATE student SET name="李四" WHERE id=1;
UPDATE student SET name="王五" WHERE id=1;
# Transaction 20
BEGIN;
# 更新了一些别的表的记录
...
```

此刻，表student 中 id 为 1 的记录得到的版本链表如下所示：

![image-20220715140006061](assets/image-20220715140006061.png)

假设现在有一个使用 `REPEATABLE READ` 隔离级别的事务开始执行：

```mysql
# 使用REPEATABLE READ隔离级别的事务
BEGIN;

# SELECT1：Transaction 10、20未提交
SELECT * FROM student WHERE id = 1; # 得到的列name的值为'张三'
```

这个<font color=orange>**SELECT1**</font>的执行过程如下： 

步骤1：在执行SELECT语句时会先生成一个<font color=orange>**ReadView**</font>，ReadView的<font color=orange>**trx_ids**</font>列表的内容就是<font color=orange>**[10，20]**</font>， **<font color=orange>up_limit_id</font>** 为 <font color=orange>**10** </font>， <font color=orange>**low_limit_id**</font> 为<font color=orange> **21**</font>， <font color=orange>**creator_trx_id**</font> 为<font color=orange> **0**</font>。 

步骤2：从版本链中挑选可见的记录，从图中看出，最新版本的列<font color=orange>**name**</font>是<font color=orange>**'王五'**</font>，该版本的trx_id值为 <font color=orange>**10**</font>，<font color=orange>**在trx_ids列表内**</font>，不符合可见性要求，根据roll_pointer跳到下一个版本。

步骤3：下一个版本的列name是<font color=orange>**'李四'**</font>，该版本的trx_id值也为10 ，也<font color=orange>**在trx_ids列表内**</font>，不符合要求，继续跳到下一个版本。

步骤4：下一个版本的列name是<font color=orange>**'张三'**</font>，该版本的trx_id值为8 ，小于ReadView中的up_limit_id 值10，所以这个版本是<font color=orange>**符合要求**</font>的，最后返回给用户的版本就是这条列name为'张三' 的记录。

之后，我们把 `事务id` 为 `10` 的事务提交一下：

```mysql
# Transaction 10
BEGIN;

UPDATE student SET name="李四" WHERE id=1;
UPDATE student SET name="王五" WHERE id=1;

COMMIT;
```

然后再到 `事务id` 为 `20` 的事务中更新一下表 `student` 中 `id` 为 `1` 的记录：

```mysql
# Transaction 20
BEGIN;
# 更新了一些别的表的记录
...
UPDATE student SET name="钱七" WHERE id=1;
UPDATE student SET name="宋八" WHERE id=1;
```

此刻，表student 中 `id` 为 `1` 的记录的版本链长这样：

![image-20220715140354217](assets/image-20220715140354217.png)

然后再到刚才使用 `REPEATABLE READ` 隔离级别的事务中继续查找这个 `id` 为 `1` 的记录，如下：

```mysql
# 使用REPEATABLE READ隔离级别的事务
BEGIN;
# SELECT1：Transaction 10、20均未提交
SELECT * FROM student WHERE id = 1; # 得到的列name的值为'张三'
# SELECT2：Transaction 10提交，Transaction 20未提交
SELECT * FROM student WHERE id = 1; # 得到的列name的值仍为'张三'
```

<font color=orange>**SELECT2**</font>的执行过程如下： 

步骤1：因为当前事务隔离级别为REPEATABLE READ，之前在执行SELECT1时已经生成过ReadView，所以此时直接<font color=red>**复用之前的ReadView**</font>，<font color=orange>**trx_ids**</font>列表的内容就是<font color=red>**[10，20]**</font>， **<font color=orange>up_limit_id</font>** 为 <font color=orange>**10** </font>， <font color=orange>**low_limit_id**</font> 为<font color=orange> **21**</font>， <font color=orange>**creator_trx_id**</font> 为<font color=orange> **0**</font>。 

步骤2：从版本链中挑选可见的记录，从图中看出，最新版本的列<font color=orange>**name**</font>是<font color=orange>**'宋八'**</font>，该版本的trx_id值为 <font color=orange>**20**</font>，<font color=orange>**在trx_ids列表内**</font>，不符合可见性要求，根据roll_pointer跳到下一个版本。

步骤3：下一个版本的列<font color=orange>**name**</font>是<font color=orange>**'钱七'**</font>，该版本的trx_id值为 <font color=orange>**20**</font>，<font color=orange>**在trx_ids列表内**</font>，不符合可见性要求，根据roll_pointer跳到下一个版本。

步骤4：下一个版本的列<font color=orange>**name**</font>是<font color=orange>**'王五'**</font>，该版本的trx_id值为 <font color=orange>**10**</font>，<font color=orange>**在trx_ids列表内**</font>，不符合可见性要求；同理下列<font color=orange>**name**</font>是<font color=orange>**'李四'**</font>  也不符合要求，继续跳到下一个版本。

步骤4：下一个版本的列<font color=orange>**name**</font>是<font color=orange>**'张三'**</font>，该版本的trx_id值为 <font color=orange>**8**</font>，小于ReadView中的up_limit_id 值10，所以这个版本是<font color=orange>**符合要求**</font>的，最后返回给用户的版本就是这条列name为<font color=orange>**'张三'** </font>的记录。

这次`SELECT`查询得到的结果是重复的，记录的列`c`值都是`张三`，这就是`可重复读`的含义。如果我们之后再把`事务id`为`20`的记录提交了，然后再到刚才使用`REPEATABLE READ`隔离级别的事务中继续查找这个`id`为`1`的记录，得到的结果还是`张三`，具体执行过程大家可以自己分析一下。

### 5.3 如何解决幻读

在可重复读隔离级别下，说明InnoDB 是如何解决幻读的。使用 `REPEATABLE READ` 隔离级别的事务，<font color=red>**只会在第一次执行查询语句时生成一个 ReadView ，之后的查询就不会重复生成了**</font>。

假设现在表 student 中只有一条数据，数据内容中，主键 id=1，隐藏的 trx_id=10，它的 undo log 如下图所示。

<img src="assets/image-20220715141002035.png" alt="image-20220715141002035" style="zoom:80%;" />

假设现在有事务 A 和事务 B 并发执行，<font color=orange>**事务 A**</font> 的事务 id 为 <font color=orange>**20**</font> ， <font color=orange>**事务 B**</font> 的事务 id 为 <font color=orange>**30**</font> 。

步骤1：事务 A 开始第一次查询数据，查询的 SQL 语句如下。

```mysql
select * from student where id >= 1;
```

在开始查询之前，MySQL 会为事务 A 产生一个 ReadView，此时 ReadView 的内容如下： **<font color=orange>trx_ids= [20,30] ， up_limit_id=20 ， low_limit_id=31 ， creator_trx_id=20</font>** 。

由于此时表 student 中只有一条数据，且符合 where id>=1 条件，因此会查询出来。然后根据 ReadView 机制，发现该行数据的trx_id=10，小于事务 A 的 ReadView 里 up_limit_id，这表示这条数据是事务 A 开启之前，其他事务就已经提交了的数据，因此事务 A 可以读取到。

结论：事务 A 的第一次查询，能读取到一条数据，id=1。

步骤2：接着事务 B(trx_id=30)，往表 student 中新插入两条数据，并提交事务。

```mysql
insert into student(id,name) values(2,'李四');
insert into student(id,name) values(3,'王五');
commit;
```

此时表student 中就有三条数据了，对应的 undo 如下图所示：

![image-20220715141208667](assets/image-20220715141208667.png)

步骤3：接着事务 A 开启第二次查询，根据可重复读隔离级别的规则，<font color=orange>**此时事务 A 并不会再重新生成 ReadView**</font>。此时表 student 中的 3 条数据都满足 where id>=1 的条件，因此会先查出来。然后根据 ReadView 机制，判断每条数据是不是都可以被事务 A 看到。

1）首先 id=1 的这条数据，前面已经说过了，可以被事务 A 看到。 

2）然后是 id=2 的数据，它的 trx_id=30，此时事务 A 发现，这个值处于 up_limit_id 和 low_limit_id 之 间，因此还需要再判断 30 是否处于 trx_ids 数组内。由于事务 A 的 trx_ids=[20,30]，因此在数组内，这表 示 id=2 的这条数据是与事务 A 在同一时刻启动的其他事务提交的，所以这条数据不能让事务 A 看到。

3）同理，id=3 的这条数据，trx_id 也为 30，因此也不能被事务 A 看见。

![image-20220715141243993](assets/image-20220715141243993.png)

结论：最终事务 A 的第二次查询，只能查询出 id=1 的这条数据。这和事务 A 的第一次查询的结果是一样 的，因此没有出现幻读现象，所以说在 <font color=orange>**MySQL 的可重复读隔离级别下，不存在幻读问题**</font>。

## 6. 总结

这里介绍了 **<font color=orange>MVCC</font>** 在 <font color=orange>**READ COMMITTD 、 REPEATABLE READ**</font> 这两种隔离级别的事务在执行快照读操作时 访问记录的版本链的过程。这样使不同事务的 <font color=orange>**读-写、 写-读**</font> 操作并发执行，从而提升系统性能。

核心点在于 ReadView 的原理， `READ COMMITTD` 、 `REPEATABLE READ` 这两个隔离级别的一个很大不同 就是生成ReadView的时机不同：

* <font color=orange>**READ COMMITTD**</font> 在每一次进行普通SELECT操作前<font color=orange>**都会生成**</font>一个ReadView 
* <font color=orange>**REPEATABLE READ只在第一次**</font>进行普通SELECT操作前<font color=orange>**生成**</font>一个ReadView，之后的查询操作都重复 使用这个ReadView就好了。

> [!NOTE]
>
> 说明：之前说执行DELETE语句或者更新主键的UPDATE语句并不会立即把对应的记录完全从页面中删除， 而是执行所谓的delete mark操作，相当于只是对记录打上了一个删除标志位，这主要就是为MVCC服务 的。

MVCC可以解决： 

1 . <font color=orange>**读写之间阻塞的问题**</font>。通过MVCC可以让读写互相不阻塞，即读不阻塞写，写不阻塞读，从而提升事务并发处理能力。

2 . <font color=orange>**降低死锁的概率**</font>。因为MVCC采用了乐观锁的方式，读取数据时并不需要加锁，对于写操作，也只锁定必要的行。

3 . <font color=orange>**解决快照读的问题**</font>。当查询数据库在某个时间点的快照时，只能看到这个时间点之前事务提交更新的结果，而不能看到这个时间点之后事务提交的更新结果。

通过MVCC我们可以解决：

