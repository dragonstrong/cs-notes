[toc]





# 第14章_MySQL事务日志

事务有4种特性：原子性、一致性、隔离性和持久性。那么事务的四种特性到底是基于什么机制实现呢？

* 事务的 <font color=orange>**隔离性**</font>由 <font color=orange>**锁机制**</font>实现。
* 而事务的 <font color=orange>**原子性、一致性和持久性**</font>由事务的<font color=orange>**redo 日志**</font>和<font color=orange>**undo 日志**</font>来保证。
  + REDO LOG 称为 <font color=orange>**重做日志**</font>，提供再写入操作，恢复提交事务修改的页操作(在从内存写入磁盘的过程中宕机了，重启后能从redo日志中继续剩余写入)，用来保证事务的<font color=orange>**持久性**</font>。
  + UNDO LOG 称为 <font color=orange>**回滚日志**</font> ，回滚行记录到某个特定版本，用来保证事务的<font color=orange>**原子性、一致性**</font>。

有的DBA或许会认为 UNDO 是 REDO 的逆过程，其实不然。REDO 和 UNDO都可以视为是一种 `恢复操作`，但是：

* redo log: 是存储引擎层 (innodb) 生成的日志，记录的是<font color=orange>**物理级别**</font>上的页修改操作，比如页号xxx，偏移量yyy写入了'zzz'数据。主要为了保证数据的可靠性。
* undo log: 是存储引擎层 (innodb) 生成的日志，记录的是 <font color=orange>**逻辑操作**</font>日志，比如对某一行数据进行了INSERT语句操作，那么undo log就记录一条<font color=orange>**与之相反**</font>的DELETE操作。主要用于 `事务的回滚` (undo log 记录的是每个修改操作的 `逆操作`) 和 `一致性非锁定读` (undo log 回滚行记录到某种特定的版本——MVCC，即多版本并发控制)。

## 1. redo日志

InnoDB存储引擎是<font color=orange>**以页为单位**</font>来管理存储空间的。在真正访问页面之前，需要把在<font color=orange>**磁盘上的页**</font>缓存到内存中的<font color=orange>**Buffer Pool**</font>之后才可以访问。所有的变更都必须<font color=orange>**先更新缓冲池**</font>中的数据，然后缓冲池中的<font color=orange>**脏页**</font>会<font color=orange>**以一定的频率被刷入磁盘** </font>(<font color=orange>**checkPoint**</font>机制)，通过缓冲池来优化CPU和磁盘之间的鸿沟，这样就可以保证整体的性能不会下降太快。

### 1.1 为什么需要REDO日志

一方面，缓冲池可以帮助我们消除CPU和磁盘之间的鸿沟，checkpoint机制可以保证数据的最终落盘，然 而由于checkpoint <font color=orange>并不是每次变更的时候就触发</font>的，而是master线程隔一段时间去处理的。所以最坏的情 况就是事务提交后，刚写完缓冲池，数据库宕机了，那么这段数据就是丢失的，无法恢复。

另一方面，事务包含 `持久性` 的特性，就是说对于一个已经提交的事务，在事务提交后即使系统发生了崩溃，这个事务对数据库中所做的更改也不能丢失。

那么如何保证这个持久性呢？ `一个简单的做法` ：在事务提交完成之前把该事务所修改的所有页面都刷新 到磁盘，但是这个简单粗暴的做法有些问题:

* **<font color=orange>修改量与刷新磁盘工作量严重不成比例</font>**

  有时候我们仅仅修改了某个页面中的一个字节，但是我们知道在InnoDB中是以页为单位来进行磁盘IO的，也就是说我们在该事务提交时不得不将一个完整的页面从内存中刷新到磁盘，我们又知道一个默认页面时16KB大小，只修改一个字节就要刷新16KB的数据到磁盘上显然是<font color=orange>小题大做</font>了。

* **<font color=orange>随机IO刷新较慢</font>**

  一个事务可能包含很多语句，即使是一条语句也可能修改许多页面，假如该事务修改的这些页面可能并不相邻，这就意味着在将某个事务修改的Buffer Pool中的页面`刷新到磁盘`时，需要进行很多的`随机IO`，随机IO比顺序IO要慢，尤其对于传统的机械硬盘来说。

<font color=orange>**另一个解决的思路**</font> ：我们只是想让已经提交了的事务对数据库中数据所做的修改永久生效，即使后来系 统崩溃，在重启后也能把这种修改恢复出来。所以我们其实没有必要在每次事务提交时就把该事务在内 存中修改过的全部页面刷新到磁盘，只需要把 **<font color=orange>修改</font>** 了哪些东西 **<font color=orange>记录一下</font>** 就好。比如，某个事务将系统 表空间中 第10号 页面中偏移量为 100 处的那个字节的值 1 改成 2 。我们只需要记录一下：将第0号表 空间的10号页面的偏移量为100处的值更新为 2 

InnoDB引擎的事务采用了WAL技术 (<font color=orange>**Write-Ahead Logging**</font>)，这种技术的思想就是<font color=orange>**先写日志，再写磁盘，只有日志写入成功，才算事务提交成功**</font>，这里的日志就是<font color=orange>**redo log**</font>。当发生宕机且数据未刷到磁盘的时候，可以通过redo log来恢复，保证ACID中的D，这就是redo log的作用。

![image-20220710202517977](assets/image-20220710202517977.png)

### 1.2 REDO日志的好处、特点

#### 1. 好处

* <font color=orange>**redo日志降低了刷盘频率** </font>
* **<font color=orange>redo日志占用的空间非常小</font>**

存储表空间ID、页号、偏移量以及需要更新的值，所需的存储空间是很小的，刷盘快。

#### 2. 特点

* **<font color=orange>redo日志是顺序写入磁盘的</font>**

  在执行事务的过程中，每执行一条语句，就可能产生若干条redo日志，这些日志是按照<font color=orange>**产生的顺序写入磁盘的**</font>，也就是使用顺序ID，效率比随机IO快。

* **<font color=orange>事务执行过程中，redo log不断记录</font>**

  redo log跟bin log的区别，redo log是<font color=orange>**存储引擎层**</font>产生的，而bin log是<font color=orange>**数据库层**</font>产生的。假设一个事务，对表做10万行的记录插入，在这个过程中，一直不断的往redo log顺序记录，而bin log不会记录，直到这个事务提交，才会一次写入到bin log文件中。

### 1.3 redo的组成

Redo log可以简单分为以下两个部分：

* <font color=orange>**重做日志的缓冲 (redo log buffer)**</font> ，保存在内存中，是<font color=orange>**易失**</font>的。

在服务器启动时就会向操作系统申请了一大片称之为 redo log buffer 的 <font color=orange>**连续内存**</font> 空间，翻译成中文就是redo日志缓冲区。这片内存空间被划分为若干个连续的<font color=orange>**redo log block**</font>。一个redo log block占用<font color=orange>**512字节**</font>大小。

![image-20220710204114543](assets/image-20220710204114543.png)

**<font color=orange>参数设置：innodb_log_buffer_size：</font>**

redo log buffer 大小，默认 <font color=orange>**16M**</font> ，最大值是4096M，最小值为1M。

```mysql
mysql> show variables like '%innodb_log_buffer_size%';
+------------------------+----------+
| Variable_name          | Value    |
+------------------------+----------+
| innodb_log_buffer_size | 16777216 |
+------------------------+----------+
```

* <font color=orange>**重做日志文件 (redo log file)** </font>，保存在硬盘中，是<font color=orange>**持久**</font>的。

REDO日志文件如图所示，其中`ib_logfile0`和`ib_logfile1`即为REDO日志。

![image-20240716125746194](assets/image-20240716125746194.png)

说明：<font color=orange>**mysql 8.0.30之前的版本是这样，之后的版本redo log变了**</font>

![image-20240716125916856](assets/image-20240716125916856.png)

详情参见官方文档：https://dev.mysql.com/doc/refman/8.0/en/innodb-redo-log.html

![image-20240716130018248](assets/image-20240716130018248.png)



![image-20240716130046193](assets/image-20240716130046193.png)

### 1.4 redo的整体流程

以一个更新事务为例，redo log 流转过程，如下图所示：

![image-20240716110149765](assets/image-20240716110149765.png)

```
第1步：先将原始数据从磁盘中读入内存中来，修改数据的内存拷贝
第2步：生成一条重做日志并写入redo log buffer，记录的是数据被修改后的值
第3步：当事务commit时，将redo log buffer中的内容刷新到 redo log file，对 redo log file采用追加写的方式
第4步：定期将内存中修改的数据刷新到磁盘中
```

> 体会： Write-Ahead Log(预先日志持久化)：在持久化一个数据页之前，先将内存中相应的日志页持久化。

### 1.5 redo log的刷盘策略

redo log的写入并不是直接写入磁盘的，InnoDB引擎会在写redo log的时候先写redo log buffer，之后以` 一 定的频率 `刷入到真正的redo log file 中。这里的一定频率怎么看待呢？这就是我们要说的刷盘策略。

![image-20220710205015302](assets/image-20220710205015302.png)

注意，redo log buffer刷盘到redo log file的过程并不是真正的刷到磁盘中去，只是刷入到 <font color=orange>**文件系统缓存** </font>（page cache）中去（这是现代操作系统为了提高文件写入效率做的一个优化），真正的写入会交给系统自己来决定（比如page cache足够大了）。那么对于InnoDB来说就存在一个问题，如果交给系统来同 步，同样如果系统宕机，那么数据也丢失了（虽然整个系统宕机的概率还是比较小的）。

针对这种情况，InnoDB给出 **<font color=orange>innodb_flush_log_at_trx_commit</font>** 参数，该参数控制 commit提交事务 时，如何将 redo log buffer 中的日志刷新到 redo log file 中。它支持三种策略：

* **<font color=orange>设置为0</font>** ：表示每次事务提交时不进行刷盘操作。（系统默认master thread每隔1s进行一次重做日 志的同步） 第1步：先将原始数据从磁盘中读入内存中来，修改数据的内存拷贝 第2步：生成一条重做日志并写入redo log buffer，记录的是数据被修改后的值 第3步：当事务commit时，将redo log buffer中的内容刷新到 redo log file，对 redo log file采用追加 写的方式 第4步：定期将内存中修改的数据刷新到磁盘中 
* <font color=orange>**设置为1**</font> ：表示每次事务提交时都将进行同步，刷盘操作（ <font color=orange>**默认值**</font> ） 
* **<font color=orange>设置为2</font>** ：表示每次事务提交时都只把 redo log buffer 内容写入 page cache，不进行同步。由os自 己决定什么时候同步到磁盘文件。

![image-20240716140011929](assets/image-20240716140011929.png)

另外，InnoDB存储引擎有一个<font color=orange>**后台线程**</font>，每隔<font color=orange>**1秒**</font>，就会把`redo log buffer`中的内容写到文件系统缓存(`page cache`)，然后调用刷盘操作(<font color=orange>**innodb_flush_log_at_trx_commit=0时只能交给后台线程刷盘**</font>)。

![image-20220710210339724](assets/image-20220710210339724.png)

也就是说，一个没有提交事务的`redo log`记录，也可能会刷盘。因为在事务执行过程 redo log 记录是会写入 `redo log buffer`中，这些redo log 记录会被`后台线程`刷盘。

![image-20220710210532805](assets/image-20220710210532805.png)

除了后台线程每秒`1次`的轮询操作，还有一种情况，当<font color=orange>**redo log buffer占用的空间即将达到innodb_log_buffer_size（这个参数默认是16M）的一半**</font>的时候，后台线程会<font color=orange>**主动刷盘**</font>。

### 1.6 不同刷盘策略演示

#### 1. 流程图

<img src="assets/image-20220710210751414.png" alt="image-20220710210751414" style="float:left;" />

**小结**:   innodb_flush_log_at_trx_commit=1

只要事务提交成功，<font color=orange>**redo log**</font>记录就一定在硬盘里，不会有任何数据丢失。如果事务执行期间MySQL挂了或宕机，但是<font color=orange>**事务没有提交，所以日志丢了也不会有损失**</font>，可以保证ACID的D,数据绝对不会丢失，但是<font color=orange>**效率最差**</font>的。 建议使用默认值，虽然操作系统宕机的概率理论小于数据库宕机的概率，但是既然使用了事务，那么数据的安全相对来说更重要些。



<img src="assets/image-20220710211335379.png" alt="image-20220710211335379" style="float:left;" />

**小结**： innodb_flush_log_at_trx_commit=2 

只要事务提交成功，<font color=orange>**redo log buffer**</font>中的内容只写入<font color=orange>**文件系统缓存**</font>(page cache)。如果仅仅只是<font color=orange>**MySQL挂了不会有任何数据丢失**</font>，但是<font color=orange>**操作系统宕机可能会有1秒数据的丢失**</font>，这种情况下无 法满足ACID中的D，但是数值2肯定是效率最高的。



<img src="assets/image-20220710211831675.png" alt="image-20220710211831675" style="float:left;" />

**小结**： innodb_flush_log_at_trx_commit=0

master thread中每隔1秒进行一次刷盘操作，因此实例crash<font color=orange>**最多丢失1秒**</font>内的事务。 （ master thread是负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性）。数值0的IO效率高于1，但有丢失数据的风险，无法保证D。

#### 2. 举例

比较innodb_flush_log_at_trx_commit对事务的影响。

```mysql
CREATE TABLE test_load(
a INT,
b CHAR(80)
)ENGINE=INNODB;
```

```mysql
DELIMITER//
CREATE PROCEDURE p_load(COUNT INT UNSIGNED)
BEGIN
DECLARE s INT UNSIGNED DEFAULT 1;
DECLARE c CHAR(80) DEFAULT REPEAT('a',80);
WHILE s<=COUNT DO
INSERT INTO test_load SELECT NULL, c;
COMMIT;
SET s=s+1;
END WHILE;
END //
DELIMITER;
```

存储过程代码中，每插入一条数据就进行一次显式COMMIT操作。在默认的设置下，即参数 innodb_flush_log_at_trx_commit=1，InnoDB存储引擎会将重做日志缓冲中的日志写入文件，并调用一 次fsync操作。 执行命令CALL  p_load (30000)，向表中插入3万行的记录，并执行3万次的fsync操作。在默认情况下所需的时间:

```mysql
mysql> CALL p_load(30000);
Query OK, 0 rows affected(1 min 23 sec)
```

`1 min 23 sec`的时间显然是不能接受的。而造成时间比较长的原因就在于fsync操作所需要的时间。

修改参数innodb_flush_log_at_trx_commit，设置为0：

```mysql
mysql> set global innodb_flush_log_at_trx_commit = 0;
```

```mysql
mysql> truncate table test_load; #清空表
mysql> CALL p_load(30000); #重新插入数据
Query OK, 0 rows affected(38 sec)
```

修改参数innodb_flush_log_at_trx_commit，设置为2：

```mysql
mysql> set global innodb_flush_log_at_trx_commit = 2;
```

```mysql
mysql> CALL p_load(30000);
Query OK, 0 rows affected(46 sec)
```

形成这个现象的主要原因：后者大大减少了fsync的次数，提高了数据库执行的性能。下表显示了在innodb_flush_log_at_trx_commit不同设置下，调用存储过程p_load插入3万行记录所需的时间：

| innodb_flush_log_at_trx_commit | 执行耗时 |
| :----------------------------: | :------: |
|               0                |   38s    |
|               1                | 1min23s  |
|               2                |   46s    |

针对上述存储过程，为提高事务的提交性能，应该在将3万行记录插入表后进行一次COMMIT操作，而不是每插入一条记录后进行一次COMMIT。这样做的好处是可以使事务方法在rollback时回滚到事务最开始的确定状态。

> [!NOTE]
>
> 虽然用户可以通过设置参数innodb_flush_log_at_trx_commit为0或2提高了事务提交的性能，但丧失了ACID特性。

### 1.7 写入redo log buffer 过程

#### 1. 补充概念：Mini-Transaction

MySQL把对底层页面中的一次原子访问过程称之为一个<font color=orange>Mini-Transaction</font>，简称`mtr`，比如，向某个索引对应的B+树中插入一条记录的过程就是一个`Mini-Transaction`。一个所谓的`mtr`可以包含一组redo日志，在进行崩溃恢复时这一组`redo`日志可以作为一个不可分割的整体。

一个事务可以包含若干条语句，每一条语句其实是由若干个 `mtr` 组成，每一个 `mtr` 又可以包含若干条 redo日志，画个图表示它们的关系就是这样：

![image-20220710220653131](assets/image-20220710220653131.png)

#### 2. redo 日志写入log buffer

<img src="assets/image-20220710220838744.png" alt="image-20220710220838744" style="float:left;" />

![image-20220710220919271](assets/image-20220710220919271.png)

<img src="assets/image-20220710221221981.png" alt="image-20220710221221981" style="float:left;" />

![image-20220710221318271](assets/image-20220710221318271.png)

不同的事务可能是 `并发` 执行的，所以 T1 、 T2 之间的 mtr 可能是 `交替执行` 的。没当一个mtr执行完成时，伴随该mtr生成的一组redo日志就需要被复制到log buffer中，也就是说不同事务的mtr可能是交替写入log buffer的，我们画个示意图（为了美观，我们把一个mtr中产生的所有redo日志当做一个整体来画）：

![image-20220710221620291](assets/image-20220710221620291.png)

有的mtr产生的redo日志量非常大，比如`mtr_t1_2`产生的redo日志占用空间比较大，占用了3个block来存储。

#### 3. redo log block的结构图

一个redo log block是由`日志头、日志体、日志尾`组成。日志头占用12字节，日志尾占用8字节，所以一个block真正能存储的数据是512-12-8=492字节。

<img src="assets/image-20220710223117420.png" alt="image-20220710223117420" style="float:left;" />

![image-20220710223135374](assets/image-20220710223135374.png)

真正的redo日志都是存储到占用`496`字节大小的`log block body`中，图中的`log block header`和`log block trailer`存储的是一些管理信息。我们来看看这些所谓`管理信息`都有什么。

![image-20220711144546439](assets/image-20220711144546439.png)

<img src="assets/image-20220711144608223.png" alt="image-20220711144608223" style="float:left;" />

### 1.8 redo log file

#### 1. 相关参数设置

* `innodb_log_group_home_dir` ：指定 redo log 文件组所在的路径，默认值为 `./` ，表示在数据库 的数据目录下。MySQL的默认数据目录（ `var/lib/mysql`）下默认有两个名为 `ib_logfile0` 和 `ib_logfile1` 的文件，log buffer中的日志默认情况下就是刷新到这两个磁盘文件中。此redo日志 文件位置还可以修改。

* `innodb_log_files_in_group`：指明redo log file的个数，命名方式如：ib_logfile0，iblogfile1... iblogfilen。默认2个，最大100个。

  ```mysql
  mysql> show variables like 'innodb_log_files_in_group';
  +---------------------------+-------+
  | Variable_name             | Value |
  +---------------------------+-------+
  | innodb_log_files_in_group | 2     |
  +---------------------------+-------+
  #ib_logfile0
  #ib_logfile1
  ```

* `innodb_flush_log_at_trx_commit`：控制 redo log 刷新到磁盘的策略，默认为1。

* `innodb_log_file_size`：单个 redo log 文件设置大小，默认值为 `48M` 。最大值为512G，注意最大值 指的是整个 redo log 系列文件之和，即（innodb_log_files_in_group * innodb_log_file_size ）不能大 于最大值512G。

  ```mysql
  mysql> show variables like 'innodb_log_file_size';
  +----------------------+----------+
  | Variable_name        | Value    |
  +----------------------+----------+
  | innodb_log_file_size | 50331648 |
  +----------------------+----------+
  ```

根据业务修改其大小，以便容纳较大的事务。编辑my.cnf文件并重启数据库生效，如下所示

```mysql
[root@localhost ~]# vim /etc/my.cnf
innodb_log_file_size=200M
```

> 在数据库实例更新比较频繁的情况下，可以适当加大 redo log 数组和大小。但也不推荐 redo log 设置过大，在MySQL崩溃时会重新执行REDO日志中的记录。

#### 2. 日志文件组

<img src="assets/image-20220711152137012.png" alt="image-20220711152137012" style="float:left;" />

![image-20220711152242300](assets/image-20220711152242300.png)

总共的redo日志文件大小其实就是： `innodb_log_file_size × innodb_log_files_in_group` 。

采用循环使用的方式向redo日志文件组里写数据的话，会导致后写入的redo日志覆盖掉前边写的redo日志？当然！所以InnoDB的设计者提出了checkpoint的概念。

#### 3. checkpoint

在整个日志文件组中还有两个重要的属性，分别是 write pos、checkpoint

* `write pos`是当前记录的位置，一边写一边后移
* `checkpoint`是当前要擦除的位置，也是往后推移

每次刷盘 redo log 记录到日志文件组中，write pos 位置就会后移更新。每次MySQL加载日志文件组恢复数据时，会清空加载过的 redo log 记录，并把check point后移更新。write pos 和 checkpoint 之间的还空着的部分可以用来写入新的 redo log 记录。

<img src="assets/image-20220711152631108.png" alt="image-20220711152631108" style="zoom:80%;" />

如果 write pos 追上 checkpoint ，表示`日志文件组`满了，这时候不能再写入新的 redo log记录，MySQL 得 停下来，清空一些记录，把 checkpoint 推进一下。

<img src="assets/image-20220711152802294.png" alt="image-20220711152802294" style="zoom:80%;" />

### 1.9 redo log 小结

<img src="assets/image-20220711152930911.png" alt="image-20220711152930911" style="float:left;" />

## 2. Undo日志

redo log是事务持久性的保证，undo log是事务原子性的保证。在事务中 **<font color=orange>更新数据</font>** 的 <font color=orange>**前置操作**</font> 其实是要先写入一个 `undo log` 。

### 2.1 如何理解Undo日志

事务需要保证 <font color=orange>**原子性** </font>，也就是事务中的操作要么全部完成，要么什么也不做。但有时候事务执行到一半会出现一些情况，比如：

* 情况一：事务执行过程中可能遇到各种错误，比如<font color=orange>**服务器本身的错误 ， 操作系统错误** </font>，甚至是突然 <font color=orange>断电</font>导致的错误。
* 情况二：程序员可以在事务执行过程中手动输入 <font color=orange>**ROLLBACK**</font> 语句结束当前事务的执行。

以上情况出现，我们需要把数据改回原先的样子，这个过程称之为 <font color=orange>**回滚**</font> ，这样就可以造成一个假象：这 个事务看起来什么都没做，所以符合 `原子性` 要求。



当对一条记录做改动时（<font color=orange>**INSERT、DELETE , UPDATE**</font>），都要备份应对回滚：

- <font color=orange>**插入一条记录**</font>：至少要把这条记录的主键值记下来，回滚时只需把这个主键值对应的记录删掉即可
- <font color=orange>**删除一条记录**</font>：要把这条记录的完整内容都记下来，回滚时再将其插入
- <font color=orange>**修改一条记录**</font>：至少要把修改这条记录前的旧值记录下来，回滚时再把这条记录改回旧值。

 MySQL把这些为了回滚而记录的这些内容称之为<font color=orange>**撤销日志**</font>或者 <font color=orange>**回滚日志**</font>（即<font color=orange>**undo log**</font>）。注意，由于<font color=orange>**查询操作 （SELECT）不会修改任何用户记录，所以不需要记入相应的undo日志**</font>。

### 2.2 Undo日志的作用

* **作用1：<font color=orange>回滚数据</font>**

undo用于将数据库物理地恢复到执行语句或事务之前的样子。但事实并非如此。undo是<font color=orange>**逻辑日志**</font>，只是将数据库逻辑地恢复到原来的样子。所有修改都被逻辑地取消了，但是<font color=orange>**数据结构和页本身在回滚前后可能有所区别**</font>。这是因为在多用户并发系统中，可能会有数十、数百甚至数千个并发事务。数据库的主要任务就是协调对数据记录的并发访问。比如，一个事务插入了几条记录，导致新建数据页，其他事务也在新建页中插入数据，后面前一个事务回滚，新建的页不可能撤销，因为其他事务插入的事务还要用。

* **作用2：<font color=orange>MVCC</font>**

undo的另一个作用是MVCC，即在InnoDB存储引擎中<font color=orange>**MVCC的实现是通过undo来完成**</font>。当用户读取一行记录时，若该记录以及被其他事务占用，当前事务可以通过undo读取之前的行版本信息，以此实现非锁定读取。

### 2.3 undo的存储结构

#### 1. 回滚段与undo页

InnoDB对undo log的管理采用段的方式，也就是 <font color=orange>**回滚段（rollback segment）**</font> 。每个回滚段记录了 <font color=orange>**1024** </font>个 <font color=orange>**undo log segment**</font> ，而在每个undo log segment段中进行 <font color=orange>**undo页**</font>的申请。

* 在<font color=orange>**InnoDB1.1版本之前**</font> （不包括1.1版本），只有一个rollback segment，因此支持同时在线的事务限制为 <font color=orange>**1024**</font>。虽然对绝大多数的应用来说都已经够用。 
* 从<font color=orange>**1.1版本开始**</font>InnoDB支持最大 `128个rollback segment` ，故其支持同时在线的<font color=orange>**事务限制提高到了 128*1024**</font>。

```mysql
mysql> show variables like 'innodb_undo_logs';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| innodb_undo_logs | 128   |
+------------------+-------+
```

虽然InnoDB1.1版本支持128个rollback segment，但这些rollback segmen都存储于<font color=orange>**共享表空间ibdata**</font>中。从 lnnoDB1.2 版本开始，可通过参数对rollback segment做进一步设置，这些参数包括：

- <font color=orange>**innodb_undo_directory**</font> ：设置rollback segment文件所在路径，意味着rollback segment可以存放在共享表空间以外的位置（独立表空间）。该参数的默认值为"./"  （ /var/lib/mysql ）。

- <font color=orange>**innodb_undo_logs**</font>：设置rollback segment个数，默认值为128，在lnnoDB1.2版本中，该参数用来替换之前版本的innodb_rollback_segments参数 

-  <font color=orange>**innodb_undo_tablespaces**</font> ：设置构成rollback segment文件的数量，使rollback segment可以较为平均地分布在多个文件中。设置该参数后，会在路径innodb_undo_directory 看到前缀为undo的文件，该文件就代表 rollback segment文件。

  

undo log相关参数一般很少改动。

**<font color=orange>undo页的重用</font>**

当开启一个事务需要写undo log时，就得先去undo log segment中找一个空闲的位置，有空位就去申请undo页，在这个申清到的undo页中进行undo log的写入。

mysql页大小默认是16k，为每一个事务分配一个页非常浪费，假设应用的TPS （每秒处理的事务数目）为1000，那么1s就需要1000个页（16M），1分钟大概需要1G的存储。这样下去除非MySQL清理频率很高，否则随着时间的推移，磁盘空间会增长的非常快，而且很多空间都是浪费的。于是undo页就被设计的可以<font color=orange>**重用**</font>，当事务提交时，并不会立刻删除undo页。因为重用，所以这个undo页可能混杂其他事务的undo log。undo log在commit后，会被放在一个<font color=orange>**链表**</font>中，然后判断undo页的使用空间是否小于<font color=orange>**3/4**</font>，是则表示当前的undo页可以被重用，不会被回收，其他事务的undo log可以记录在当前undo页的后面。由于undo log是<font color=orange>**离散**</font>的，所以清理对应的磁盘空间时，效率不高。

#### 2. 回滚段与事务

1. <font color=orange>**每个事务只会使用一个回滚段，一个回滚段在同一时刻可能会服务于多个事务**</font>。

2. 当一个事务开始的时候，会制定一个回滚段，在事务进行的过程中，当数据被修改时，原始的数据会被复制到回滚段。

3. 在回滚段中，事务会不断填充盘区，直到事务结束或所有的空间被用完。如果当前的盘区不够 用，事务会在段中请求扩展下一个盘区，如果所有已分配的盘区都被用完，事务会覆盖最初的盘 区或者在回滚段允许的情况下扩展新的盘区来使用。

4. 回滚段存在于undo表空间中，在数据库中可以存在多个undo表空间，但同一时刻只能使用一个 undo表空间。

   ```mysql
   mysql> show variables like 'innodb_undo_tablespaces';
   +-------------------------+-------+
   | Variable_name           | Value |
   +-------------------------+-------+
   | innodb_undo_tablespaces | 2     |
   +-------------------------+-------+
   # undo log的数量，最少为2. undo log的truncate操作有purge协调线程发起。在truncate某个undo log表空间的过程中，保证有一个可用的undo log可用。
   ```

5. 当事务提交时，InnoDB存储引擎会做以下两件事情：

   + 将undo log放入列表中，以供之后的purge操作 
   + 判断undo log所在的页是否可以重用，若可以分配给下个事务使用

#### 3. 回滚段中的数据分类

1. <font color=orange>**未提交的回滚数据(uncommitted undo information)**</font>：该数据所关联的事务并未提交，用于实现读一致性，所以该数据不能被其他事务的数据覆盖。
2. <font color=orange>**已经提交但未过期的回滚数据(committed undo information)**</font>：该数据关联的事务已经提交，但是仍受到undo retention参数的保持时间的影响。
3. <font color=orange>**事务已经提交并过期的数据(expired undo information)**</font>：事务已经提交，而且数据保存时间已经超过 undo retention参数指定的时间，属于已经过期的数据。当回滚段满了之后，就优先覆盖“事务已经提交并过期的数据"。

事务提交后不能马上删除undo log及undo log所在的页。这是因为可能还有其他事务需要通过undo log来得到行记录之前的版本。故事务提交时将undo log放入一个链表中，<font color=orange>**是否可以最终删除undo log以undo log所在页由purge线程来判断**</font>。

### 2.4 undo的类型

在InnoDB存储引擎中，undo log分为：

* insert undo log

  insert undo log是指insert操作中产生的undo log。因为insert操作的记录，只对事务本身可见，对其他事务不可见（这是事务隔离性的要求），故该undo log可以在事务提交后直接删除。不需要进行purge操作。

* update undo log

  update undo log记录的是对delete和update操作产生的undo log。该undo log可能需要提供MVCC机制，因此不能在事务提交时就进行删除。提交时放入undo log链表，等待purge线程进行最后的删除。

### 2.5 undo log的生命周期

#### 1. 简要生成过程

以下是undo+redo事务的简化过程

假设有两个数值，分别为A=1和B=2，然后将A修改为3，B修改为4

```mysql
1. start transaction;
2. id录 A=1 到undo log;
3. update A = 3;
4. 记录 A=3 到redo log;
5. 记录 B=2 到undo log;
6. update B = 4;
7. 记录B = 4 到redo log;
8. 将redo log刷新到磁盘
9. commit
```

- 在1-8步骤的任意一步系统宕机，事务未提交（只在内存层面更改），该事务就不会对磁盘上的数据做任何影响。
- 如果在8-9之间宕机，恢复之后可以选择回滚，也可以选择继续完成事务提交，因为此时redo log已经持久化
- 若在9之后系统宕机，内存映射中变更的数据还来不及刷回磁盘，那么系统恢复之后，可以根据redo log把数据刷回磁盘。 



**只有Buffer Pool的流程：**

![image-20220711162505008](assets/image-20220711162505008.png)

**有了Redo Log和Undo Log之后：**

![image-20220711162642305](assets/image-20220711162642305.png)

在更新Buffer Pool中的数据之前，我们需要先将该数据事务开始之前的状态写入Undo Log中。假设更新到一半出错了，我们就可以通过Undo Log来回滚到事务开始前。

#### 2. 详细生成过程

对于InnoDB引擎来说，每个行记录除了记录本身的数据外，还有几个<font color=orange>**隐藏列**</font>: 

- <font color=orange>**DB_ROW_ID**</font> ：如果没有为表显式定义主键，并且表中也没有定义唯一索引，那么InnoDB会自动为表添加一 个row_id的隐藏列作为主键。 
- <font color=orange>**DB_TRX_ID**</font> ：每个事务都会分配一个<font color=orange>**事务ID**</font>，当对某条记录发生变更时，就会将这个事务的事务ID写入trx_id 中。 
- <font color=orange>**DB_ROLL_PTR**</font> ：<font color=orange>**回滚指针**</font>，指向undo log。

![image-20240716165731835](assets/image-20240716165731835.png)

**当执行INSERT时：**

```mysql
begin;
INSERT INTO user (name) VALUES ("tom");
```

插入的数据都会生成一条insert undo log，并且数据的回滚指针会指向它。undo log会记录undo log的序号、插入主键的列和值...，那么在进行rollback的时候，通过主键直接把对应的数据删除即可。

![image-20220711163725129](assets/image-20220711163725129.png)

**当我们执行UPDATE时：**

对应更新的操作会产生update undo log，并且会分更新主键和不更新主键的，假设现在执行：

```mysql
UPDATE user SET name="Sun" WHERE id=1;
```

![image-20220711164138414](assets/image-20220711164138414.png)

这时会把老的记录写入新的undo log，让回滚指针指向新的undo log，它的undo no是1，并且新的undo log会指向老的undo log（undo no=0）。

假设现在执行：

```mysql
UPDATE user SET id=2 WHERE id=1;
```

![image-20220711164421494](assets/image-20220711164421494.png)

对于更新主键的操作，会先把原来的数据deletemark标识打开，这时并没有真正的删除数据，真正的删除会交给清理线程去判断，然后在后面插入一条新的数据，新的数据也会产生undo log，并且undo log的序号会递增。

可以发现每次对数据的变更都会产生一个undo log，当一条记录被变更多次时，那么就会产生多条undo log，undo log记录的是变更前的日志，并且每个undo log的序号是递增的，那么当要回滚的时候，按照序号`依次向前推`，就可以找到我们的原始数据了。

#### 3. undo log是如何回滚的

以上面的例子来说，假设执行rollback，那么对应的流程应该是这样：

1. 通过undo no=3的日志把id=2的数据删除 
2. 通过undo no=2的日志把id=1的数据的deletemark还原成0 
3. 通过undo no=1的日志把id=1的数据的name还原成Tom 
4. 通过undo no=0的日志把id=1的数据删除

#### 4. undo log的删除

* 针对于insert undo log

  因为<font color=orange>**insert**</font>操作的记录，只对事务本身可见，对其他事务不可见。故该undo log可以在事务提交后<font color=orange>**直接删除**</font>，不需要进行purge操作。

* 针对于update undo log

  该<font color=orange>**undo log可能需要提供MVCC机制，因此不能在事务提交时就进行删除**</font>。提交时放入undo log链表，等待purge线程进行最后的删除。

> 补充：
>
> purge线程两个主要作用是：<font color=orange>**清理undo页和清理page里面带有Delete_Bit标识的数据行**</font>。在InnoDB中，事务中的Delete操作实际上并不是真正的删除掉数据行，而是一种Delete Mark操作，在记录上标识Delete_Bit，而不删除记录。是一种“假删除”，只是做了个标记，真正的删除工作需要后台purge线程去完成。

### 2.6 小结

![image-20220711165612956](assets/image-20220711165612956.png)

undo log是逻辑日志，对事务回滚时，只是将数据库逻辑地恢复到原来的样子。

redo log是物理日志，记录的是数据页的物理变化，undo log不是redo log的逆过程。

