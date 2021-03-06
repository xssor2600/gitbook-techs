##### Mysql架构总结
![image](https://s3.ax1x.com/2020/12/26/r4a6aR.png)

- 连接器

  连接器负责跟客户端建立连接、获取权限、维持和管理连接。连接命令一般是这么写的：

  ```sql
  mysql -h$ip -P$port -u$user -p
  // 查看链接
  mysql> show processlist;
  +----+------+-----------------+------+---------+------+-------+-----------------
  -+
  | Id | User | Host            | db   | Command | Time | State | Info
   |
  +----+------+-----------------+------+---------+------+-------+-----------------
  -+
  |  4 | root | localhost:55722 | test | Query   |    0 | init  | show processlist
   |
  +----+------+-----------------+------+---------+------+-------+-----------------
  -+
  1 row in set (0.00 sec)
  ```

  若是Command列包含状态为Sleep字段就表示这个是一个空闲连接。客户端如果太长时间没动静，连接器就会自动将它断开。这个时间是由参数wait_timeout控制的，**默认值是8小时**。

  > 使用长连接为什么MySQL占用内存会涨得快？

  这是因为MySQL在执行过程中临时使用的内存是管理在连接对象里面的。这些资源会在连接断开的时候才释放。所以如果长连接累积下来，可能导致内存占用太大.

  解决方案：

  - 定期断开长连接：执行过一个占用内存的大查询后，断开连接，之后要查询再重连。
  - MySQL 5.7后的版本通过`mysql_reset_connection`初始化连接资源。

  

- 查询缓存

  但是大多数情况下建议不要使用查询缓存，为什么呢？因为查询缓存往往弊大于利。查询缓存的失效非常频繁，只要有对一个表的更新，这个表上所有的查询缓存都会被清空。

- 分析器

  分析器先会做“词法分析”。你输入的是由多个字符串和空格组成的一条SQL语句，MySQL需要识别出里面的字符串分别是什么，代表什么。就要做“语法分析”。根据词法分析的结果，语法分析器会根据语法规则，判断你输入的这个SQL语句是否满足MySQL语法。

  **常见SQL语法错误是否就是在此阶段处理返回的？**

- 优化器

  优化器是在表里面有多个索引的时候，决定使用哪个索引；或者在一个语句有多表关联（join）的时候，决定各个表的连接顺序。

- 执行器

  ```sql
  mysql> select * from T where ID=10;
  ```

  比如我们这个例子中的表T中，ID字段没有索引，那么执行器的执行流程是这样的：

  1. 调用InnoDB引擎接口取这个表的第一行，判断ID值是不是10，如果不是则跳过，如果是则将这行存在结果集中；
  2. 调用引擎接口取“下一行”，重复相同的判断逻辑，直到取到这个表的最后一行。
  3. 执行器将上述遍历过程中所有满足条件的行组成的记录集作为结果集返回给客户端。

  至此，这个语句就执行完成了。

![image](https://s3.ax1x.com/2020/12/26/r4g3E8.jpg)

- 连接层

  最上层的连接池是一些连接服务，包含本地sock通信和大多数基于C/S工具实现的类似于TCP/IP的通信。主要完成一些类似于连接处理、授权认证及相关的安全方案。在该层上引入了线程池的概念，为通过认证安全接入的客户端提供线程。同样在该层上可以实现基于SSL的安全连接。服务器也会为安全接入的每个客户端验证它所具有的操作权限。

- 服务层

  第二层架构主要完成大多数的核心服务功能，如SQL接口、缓存的查询、SQL的分析和优化、内置函数等。所有跨存储引擎的功能也在这一层实现，如过程、函数等。在该层，服务器会解析查询并创建相应的内部解析树，并对其完成相应的优化如确定查询表的顺序，是否利用索引等，最后生成相应的执行操作。如果是select语句，服务器还会查询内部的缓存，如果缓存空间足够大，这样在频繁读操作的环境中能够很好的提升系统的性能。

- 引擎层

  存储引擎真正的负责MySQL中数据的存储和提取，服务器通过API与存储引擎进行通信，不同的存储引擎具有的特性不同，我们可以根据实际需进行选取。

- 存储层

  数据存储层，主要是将数据存储在运行于裸设备的文件系统之上，并完成与存储引擎的交互。

  

> 在保证逻辑正确的前提下，尽量减少扫描的数据量，是数据库系统设计的通用法则之一。 如果内存够，就要多利用内存，尽量减少磁盘访问。



**Mysql进程线程设计？**

mysql是一个单进程多线程的数据库，在innodb中大概3种线程为：

- 主线程Master Thread

  这是主线程，非常核心，其用途主要是做一些`周期性的任务`，在不同的innodb版本其功能不同，这里就看最早期的版本。早起的innodb Master线程会有两种频率的任务，一种是每1秒一次的，还有每10秒一次的。

  **每秒工作任务**：

  1. 刷新日志
  2. 刷新最多100多个脏页
  3. 合并插入缓冲
  4. 如果空闲切换background

  **每10s工作任务:**

  1. 刷新日志
  2. 刷新脏页
  3. 删除undo日志
  4. 合并插入缓冲

  

- IO Thread线程，用于异步处理写请求

- purge Thread线程，用于删除undo日志。

  用于删除undo日志，这是后续的innodb版本，才将这个事情从Master线程中独立出来了





##### Mysql重要组成

- 定义数据库和实例

  物理操作系统的文件或其他形式文件类型的集合。在mysql数据库中，数据库文件可以是`frm、MYD、MYI、ibd`结尾的文件。

- 实例

  MySQL数据库由**后台线程以及一个共享内存组成**。共享内存可以被运行的后台线程所共享。数据库实例才是真正用于操作数据库文件的。

数据库是文件的集合，是依照某种数据模型组织起来并存放于二级存储器中的数据集合；数据库实例是程序，是位于用户与操作系统之间的一层数据管理软件。

**用户对数据库数据的 数据定义、数据查询、数据维护、数据库运行控制等都是在数据库实例下进行的，应用程序只有通过数据库实例才能和数据库打交道。**

> 数据库与数据库实例之间的关系？（mysql中）

在MySQL数据库中实例与数据库的关系是一一对应的一个数据库实例对应一个数据库，一个数据库对应一个实例。

Mysql重要路径：

| 路径              | 解释                    | 备注                          |
| ----------------- | ----------------------- | ----------------------------- |
| /var/lib/mysql    | mysql数据库文件存放路径 | /var/lib/mysql/test.cloud.pid |
| /usr/share/mysql  | 配置文件目录            | mysql.server命令以及配置文件  |
| /usr/bin          | 相关命令目录            | mysqladmin mysqldump等命令    |
| /etc/init.d/mysql | 启停相关命令            |                               |

![image](https://s3.ax1x.com/2020/12/26/r4gXrt.jpg)

在mysql保存数据库相关文件的data目录下，有几个重要的文件。

当每次创建一个数据库的时候，会在mysql的安装data路径下创建一个该数据库名字的文件夹。该数据库文件夹下存放的是该库内的所有表数据.

例如，进入test数据库目录：会发现该库下保存的表名有几种文件格式结尾的文件：

- ***.frm**：（表结构）

  保存每个表的元数据，包括表结构的定义。（与数据库引擎无关）

- ***.ibd**: （数据和索引）

  innoDB引擎开启了独立表空间（my.cnf中配置innodb_file_per_table=1)产生的存放该表的数据和索引文件。

  

> MYISAM与INNODB存储引擎区别?

| 对比项 | MYISAM                                                    | INNODB                                                       |
| ------ | --------------------------------------------------------- | ------------------------------------------------------------ |
| 事务   | 不支持                                                    | 支持                                                         |
| 行表锁 | `表锁`,即使操作一条记录也会锁住整个表，不适合高并发的操作 | `行锁`,操作时候只锁某一行，不对其他行有影响，适合并发操作    |
| 缓存   | 只缓存索引，不缓存真实数据（`非聚簇索引`)                 | 不仅缓存索引还要缓存真实数据，对内存要求高，而且内存大小对性能有决定性影响 |
| 表空间 | 小                                                        | 大                                                           |
| 关注点 | 性能                                                      | 事物                                                         |





##### Mysql Log

在mysql中存在两类日志：`binlog(归档日志)`和`redo log（重做日志)`

> 为什么会有两种log?

历史原因一开始有binlog，后续新的InnodDB存储引擎后开发设计了redo log.(redo log是InnoDB独有的，这两个log也是存在磁盘上的文件日志，具体文件内容是否字符可读还是二进制？）

有了redo log，InnoDB就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为`crash-safe`。

Server层，它主要做的是**MySQL功能层面**的事情，Server层也有自己的日志，称为`binlog（归档日志）`。



SQL语句执行与两类日志密切相关，使用SQL语句执行过程阐述日志作用。

```sql
ysql> create table T(ID int primary key, c int);
mysql> update T set c=c+1 where ID=2;
```

更新语句会把该表T上的查询缓存置为无效。执行流程与查询流程大致相同：**分析器→优化器→执行器→存储磁盘**

与查询不同的是，更新语句还会涉及到两个日志： `redo log, binlog`

**数据库系统都离不开物理的磁盘存储，所谓的所有库、表、索引等等概念，映射到物理终究都是“文件”。**

那么CPU与磁盘IO的处理级别差异，为了充分利用资源提供性能引出了 =⇒ 缓存。

但是最后更新的数据，**终究是要持久化的**。如何设计合理算法提高数据库性能、安全性、可用性是我们孜孜不倦追求的。



> 类似于长连接、连接池问题对性能带来的影响？

在MySQL里也有这个问题，如果每一次的更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录，然后再更新，整个过程IO成本、查找成本都很高。

如何解决？ MySQL里经常说到的WAL技术，**WAL**的全称是`Write-Ahead Logging`。

WAL的全称是Write-Ahead Logging，它的关键点就是**先写日志，再写磁盘**。

大概机制为：

```
具体来说，当有一条记录需要更新的时候，InnoDB引擎就会先把记录写到redo log里面，并更新内存，这个时候更新就算完成了。同时，InnoDB引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做。
```

**那么，在物理磁盘上如何查看mysql的wal机制产生的记录内容，保存在哪？**

默认情况下，mysql数据目录下会有两种类型文件：` ib_logfile`与`ibdata`.

- ` ib_logfile`

  其中的**redo log**就是ib_logfile0,ib_logfile1.... 是有固定大小的文件，**记录InnodDB的事务日志**(redo日志与innodb有关）。这些文件的**写入是顺序、循环写的**，logfile0写完从logfile1继续，logfile3写完则logfile0继续。

  ![image](https://s3.ax1x.com/2020/12/27/r5nPQP.png)

- `ibdata`

  ibdata中保存的数据包含：

  数据字典也就是InnoDB表的元数据

  改变缓冲区

  双写缓冲区

  撤消日志（UNDO LOG) 



**redo log与bin log有什么异同？**

1. redo log是InnoDB引擎特有的；binlog是MySQL的Server层实现的，所有引擎都可以使用。
2. redo log是物理日志，记录的是“在某个数据页上做了什么修改”；binlog是逻辑日志，记录的是这个语句的原始逻辑，比如“给ID=2这一行的c字段加1 ”。
3. redo log是循环写的，空间固定会用完；binlog是可以追加写入的。“追加写”是指binlog文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。



```sql
mysql> create table T(ID int primary key, c int);
mysql> update T set c=c+1 where ID=2;
```

1. `执行器先找引擎`取ID=2这一行。ID是主键，引擎直接用树搜索找到这一行。如果ID=2这一行所在的`数据页`本来就在内存中，就直接返回给执行器；否则，需要先从`磁盘读入内存`，然后再返回。
2. 执行器拿到引擎给的行数据，把这个值加上1，比如原来是N，现在就是N+1，得到新的一行数据，再调用引擎接口写入这行新数据。
3.  引擎将这行新数据`更新到内存`中，同时将这个更新操作记录到`redo log`里面，此时redo log处于**prepare**状态。然后告知执行器执行完成了，随时可以提交事务。
4. 执行器生成这个`操作的binlog`，并把binlog写入磁盘。
5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的redo log改成提交（**commit**）状态，更新完成。

redo log的状态是引擎负责处理和变更的。



###### 两阶段提交

将redo log的写入拆成了两个步骤：`prepare`和`commit`，这就是"**两阶段提交**"。

> 为什么必须有“两阶段提交”呢？

这是为了让两份日志之间的逻辑一致。提供mysql的高可用的一种机制，`crash-safe`.

**用反证法论证为什么需要redo log的二阶段提交:**

```
update T set c=c+1 where ID=2;
```

仍然用前面的update语句来做例子。假设当前ID=2的行，字段c的值是0，再假设执行update语句过程中在写完第**一个日志后**，**第二个日志**还没有写完期间发生了crash，会出现什么情况呢？

- **先写redo log后写binlog**

  假设在redo log写完，binlog还没有写完的时候，MySQL进程异常重启。由于我们前面说过的，**redo log写完之后，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行c的值是1**。但是**由于binlog没写完就crash了，这时候binlog里面就没有记录这个语句**。因此，之后备份日志的时候，存起来的binlog里面就没有这条语句。然后你会发现，如果需要用这个binlog来恢复临时库的话，由于这个语句的binlog丢失，这个临时库就会少了这一次更新，恢复出来的这一行c的值就是0，与原库的值不同。

- **先写binlog后写redo log。**

  如果在binlog写完之后crash，由于redo log还没写，崩溃恢复以后这个事务无效，所以这一行c的值是0。但是binlog里面已经记录了“把c从0改成1”这个日志。所以，在之后用binlog来恢复的时候就多了一个事务出来，恢复出来的这一行c的值就是1，与原库的值不同。

**可以看到，如果不使用“两阶段提交”，那么数据库的状态就有可能和用它的日志恢复出来的库的状态不一致。**



- `innodb_flush_log_at_trx_commit`与`sync_binlog`

redo log用于保证crash-safe能力。`innodb_flush_log_at_trx_commit`这个参数设置成1的时候，表示每次事务的redo log都直接持久化到磁盘。这个参数我建议你设置成1，这样可以保证MySQL异常重启之后数据不丢失。

`sync_binlog`这个参数设置成1的时候，表示每次事务的binlog都持久化到磁盘。这个参数我也建议你设置成1，这样可以保证MySQL异常重启之后binlog不丢失。



参考: [[详细分析MySQL事务日志(redo log和undo log)](https://www.cnblogs.com/DataArt/p/10209573.html)](https://www.cnblogs.com/DataArt/p/10209573.html)



> 通常对DBA有那些需求：怎样让数据库恢复到半个月内任意一秒的状态？

binlog会记录所有的逻辑操作，并且是采用“追加写”的形式。如果你的DBA承诺说半个月内可以恢复，那么备份系统中一定会保存最近半个月的所有binlog，同时系统会定期做整库备份。这里的“定期”取决于系统的重要性，可以是一天一备，也可以是一周一备。

当需要恢复到指定的某一秒时，比如某天下午两点发现中午十二点有一次误删表，需要找回数据，那你可以这么做：

- 首先，找到最近的一次全量备份，如果你运气好，可能就是昨天晚上的一个备份，从这个备份恢复到临时库；
- 然后，从备份的时间点开始，将备份的binlog依次取出来，重放到中午误删表之前的那个时刻。



##### 事务隔离级别

事务启动方式？

- 显式启动事务语句， begin 或 start transaction。配套的提交语句是commit，回滚语句是rollback。
- set autocommit=0，这个命令会将这个线程的自动提交关掉。意味着如果你只执行一个select语句，这个事务就启动了，而且并不会自动提交。这个事务持续存在直到你主动执行commit 或 rollback 语句，或者断开连接。

**长事务**

长事务(Long-Lived Transactions)，顾名思义，就是执行时间较长的事务。

长事务意味着系统里面会存在很老的事务视图。由于这些事务随时可能访问数据库里面的任何数据，所以这个事务提交之前，数据库里面它可能用到的回滚记录都必须保留，这就会导致`大量占用存储空间`。长事务还`占用锁`资源，也可能拖垮整个库。



如何避免长事务对事务的影响？

`general log` ： 将所有到达MySQL Server的SQL语句记录下来。

一般不会开启开功能，因为log的量会非常庞大。但个别情况下可能会临时的开一会儿general log以供排障使用。

```sql
mysql> show variables like 'general_log';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| general_log   | OFF   |
+---------------+-------+
1 row in set (0.00 sec)

mysql> show variables like 'log_output';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_output    | FILE  |
+---------------+-------+
1 row in set (0.00 sec)


// general_log文件保存位置
mysql> show variables like 'general_log_file';
+------------------+------------------------------------------------------------
-----+
| Variable_name    | Value
     |
+------------------+------------------------------------------------------------
-----+
| general_log_file | D:\software\database\mysql-5.6.44-winx64\data\yx20180324-PC
.log |
+------------------+------------------------------------------------------------
-----+
1 row in set (0.00 sec)
```

首先，从应用开发端来看：

- 确认是否使用了set autocommit=0。这个确认工作可以在测试环境中开展，把MySQL的general_log开起来，然后随便跑一个业务逻辑，通过general_log的日志来确认。一般框架如果会设置这个值，也就会提供参数来控制行为，你的目标就是把它改成1。（set autocommit=0 手动提交事务；=1自动提交事务
- 确认是否有不必要的只读事务。有些框架会习惯不管什么语句先用begin/commit框起来。我见过有些是业务并没有这个需要，但是也把好几个select语句放到了事务中。这种只读事务可以去掉。
- 业务连接数据库的时候，根据业务本身的预估，通过`SET MAX_EXECUTION_TIM`E命令，来控制每个语句执行的最长时间，避免单个语句意外执行太长时间。（**5.7.8版本新增**，max_execution_time参数的单位是ms）





**在MySQL中，事务支持是在引擎层实现的。为什么会出现隔离级别概念？**

因为当同一个数据库中有多个事务同时执行，可能会出现`脏读、不可重复读、幻读`问题。所以，就定义了隔离级别来针对不同情况来避免上述问题。

SQL标准的事务隔离级别包括：

- **读未提**交是指，一个事务还没提交时，它做的变更就能被别的事务看到。
- **读提交**是指，一个事务提交之后，它做的变更才会被其他事务看到。
- **可重复读**是指，一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。当然在可重复读隔离级别下，未提交变更对其他事务也是不可见的。
- **串行化**，顾名思义是对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行。

```sql
mysql> create table T(c int) engine=InnoDB;
insert into T(c) values(1);
```

| 事务A                   | 事务B           |
| ----------------------- | --------------- |
| 启动事务查询得到的值是1 | 启动事务        |
|                         | 查询得到的值是1 |
|                         | 将1改成2        |
| 查询得到值V1            |                 |
|                         | 提交事物B       |
| 查询得到的值V2          |                 |
| 提交事务A               |                 |
| 查询得到的值V3          |                 |

针对不同的隔离级别，并发事物不同时刻的值是不同的：

- 若是隔离级别是：读未提交。则V1=2，这时候事物B虽然没有提交，但是结果已经被事物A看到了，因此V2=V3=2.

- 若是隔离级别是：读提交。则V1=1，V2=2。事物B的更新在提交后才能被A看到，所以V3=2.

- 若隔离级别是：可重复读.则V1=V2=1，V3=2.之所以V2=1，遵循原则：事物在执行期间看到的数据前后必须是一致的。

- 若隔离级别是：串行化，则事物B执行“将1改成2”时候，会被锁住，知道事物A提交后，事物B才可以继续执行，从A的角度看，V1=V2=1，V3的值=2.

  

> 如何查询和设置当前会话的隔离级别？

```sql
//.查看当前会话隔离级别
mysql> select @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set (0.00 sec)

//查看系统当前隔离级别
mysql> select @@global.tx_isolation;
+-----------------------+
| @@global.tx_isolation |
+-----------------------+
| REPEATABLE-READ       |
+-----------------------+
1 row in set (0.00 sec)

//设置当前会话隔离级别
set session transaction isolatin level repeatable read;

//设置系统当前隔离级别
set global transaction isolation level repeatable read;
```



###### 事务隔离级别如何实现？

Mysql的事务隔离底层是通过视图隔离来解决的。

在“可重复读”隔离级别下，这个视图是在事务启动时创建的，整个事务存在期间都用这个视图。

在“读提交”隔离级别下，这个视图是在每个SQL语句开始执行的时候创建的。

读未提交”隔离级别下直接返回记录上的最新值，没有视图概念

“串行化”隔离级别下直接用加锁的方式来避免并行访问。

>MySQL中，实际上每条记录在更新的时候都会同时记录一条回滚操作。记录上的最新值，通过回滚操作，都可以得到前一个状态的值。（undo log   目前版本是存储在数据库data路径下的ibdata1文件中)

假设一个值从1被按顺序改成了2、3、4，在回滚日志里面就会有类似下面的记录:

![image](https://s3.ax1x.com/2020/12/27/r5jRj1.png)

视图A、B、C里面，这一个记录的值分别是1、2、4，同一条记录在系统中可以存在多个版本，就是数据库的**多版本并发控制**（`MVCC`）

对于read-view A，要得到1，就必须将当前值依次执行图中所有的回滚操作得到。

同时你会发现，即使现在有另外一个事务正在将4改成5，这个事务跟read-view A、B、C对应的事务是不会冲突的。

> 回滚日志总不能一直保留吧，什么时候删除呢？

答案是，在不需要的时候才删除。也就是说，系统会判断，当没有事务再需要用到这些回滚日志时，回滚日志会被删除。

什么时候才不需要了呢？就是当系统里没有比这个回滚日志更早的read-view的时候。





##### Mysql锁

加锁的范围，MySQL里面的锁大致可以分成`全局锁`、`表级锁`和`行锁`三类。

**锁思想是双向的：有加锁、必然要释放锁。[主动释放锁、自动释放锁]**

- 全局锁

  全局锁就是对整个数据库实例加锁。MySQL提供了一个加全局读锁的方法，命令是 `Flush tables with read lock (FTWRL)`。当你需要让整个库处于只读状态的时候，可以使用这个命令，之后其他线程的以下语句会被阻塞：数据更新语句（数据的增删改）、数据定义语句（包括建表、修改表结构等）和更新类事务的提交语句。

  使用场景：全局锁的典型使用场景是，**做全库逻辑备份**。也就是把整库每个表都select出来存成文本。

  

  **看来加全局锁不太好。但是细想一下，备份为什么要加锁呢？我们来看一下不加锁会有什么问题?**

  MVCC支持多个事务同时对同一个库表进行并发操作，若是不加锁： 事务A读取X表记录进行备份，同一时间段有事务B对该表X进行删除、新增。那么最后备份出来的数据就会产生不一致情况。（备份系统备份的得到的库不是一个逻辑时间点，这个视图是逻辑不一致的。)

  官方自带逻辑备份命令 `mysqldump`，当结合参数`–single-transaction`的时候，导数据之前就会启动一个事务，来确保拿到一致性视图。而由于MVCC的支持，这个过程中数据是可以正常更新的。(前提是支持引擎支持事务,single-transaction方法只适用于所有的表使用事务引擎的库。）

  >`–single-transaction`: 设置事务的隔离级别是RR，这样保证在一个事务中所有相同的查询读取到的数据都是一样的，也就大概保证了在dump期间，如果其他innodb引擎的线程修改了表的数据并提交，对改dump线程的数据并无影响。

  你一定在疑惑，有了这个功能，为什么还需要FTWRL呢?

  一致性读是好，但前提是引擎要支持这个隔离级别。比如，对于MyISAM这种不支持事务的引擎，如果备份过程中有更新，总是只能取到最新的数据，那么就破坏了备份的一致性。这时，我们就需要使用FTWRL命令了。

  

  既然要全库只读，为什么不使用`set global readonly=true`的方式呢？

  虽然两种方式能达到的效果是一样的，但是还是有不同:

  1. 在有些系统中，readonly的值会被用来做其他逻辑，比如用来判断一个库是主库还是备库。（配置命令外的其他作用）
  2. 在异常处理机制上有差异。FTWRL客户端异常断开会释放锁，readonly则会一直是只读状态。这样会导致整个库长时间处于不可写状态，风险较高。



- 表级锁

  MySQL里面表级别的锁有两种：一种是`表锁`，一种是`元数据锁`（meta data lock，**MDL**)。

  表锁的语法是 **lock tables … read/write**。可以主动释放锁和客户端断开自动释放锁。

  MDL（metadata lock)。MDL不需要显式使用，在访问一个表的时候会被自动加上。（version >5.5)

  **MDL的作用是，保证读写的正确性**:

  当对一个表做增删改查操作的时候，加MDL读锁；**当要对表做结构变更操作的时候，加MDL写锁**。

  1. 读锁之间不互斥，因此你可以有多个线程同时对一张表增删改查。
  2. 读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。

  小表字段的添加由于MDL和多session操作的问题可能会引起：一段时间内session过多线程阻塞挂掉。

  > MDL锁并不是在语句执行完成后就释放掉，而是事务完成提交才会释放MDL锁。

  

- 行锁

  MySQL的行锁是在引擎层由各个引擎自己实现的。（MYISAM引擎不支持事务、行锁）

  在InnoDB事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。这个就是两阶段锁协议。

  同一个事务内的**多条行锁先加锁、后加锁都会在事务提交后释放锁**，那么新的事务对旧事务内加行锁的记录操作都是需要上一个事务提交后才能处理，那么调整行锁位置有什么意义？

  在行锁之间的处理间隔内，还能尽可能的提高并发性。（往后放置那么在前面事务其他行锁处理的过程中，本身行锁并不会被加锁，进而提高并发性。）



**死锁**

有锁就会有产生死锁的风险。何为死锁，为什么会产生死锁？

**并发系统不同线程循环依赖资源，都在等待对方线程进行释放共享资源。**

| 事务A                                                        | 事务B                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| begin;<br /><br />update t set k = k+1 where id = 1;   (id=1记录被行锁，MDL读锁) | begin;                                                       |
|                                                              | update t set k=k+1 where id =2;（id=2记录行锁，MDL读锁)      |
| update t set k=k+1 where id =2;  (阻塞，等待事务B释放id=2行锁) |                                                              |
|                                                              | update t set k = k+1 where id = 1;（阻塞，等待事务A释放行锁1） |
|                                                              |                                                              |

事务A在等待事务B释放id=2的行锁，而事务B在等待事务A释放id=1的行锁。 事务A和事务B在互相等待对方的资源释放，就是进入了死锁状态。



**发现死锁如何处理？**

- 一种策略是，直接进入等待，直到超时。这个超时时间可以通过参数`innodb_lock_wait_timeout`来设置。（在InnoDB中，innodb_lock_wait_timeout的默认值是**50s**)
- 另一种策略是，发起死锁检测，发现死锁后，**主动回滚死锁链条中的某一个事务，让其他事务得以继续执行**。将参数`innodb_deadlock_detect`设置为on，表示开启这个逻辑。

正常情况下我们还是要采用第二种策略，即：主动死锁检测，而且innodb_deadlock_detect的默认值本身就是on。主动死锁检测在发生死锁的时候，是能够快速发现并进行处理的，但是它也是有额外负担的。

**死锁检测要耗费大量的CPU资源**。（所有事务都要更新同一行的场景，单个线程需要自测是否会导致死锁O(n))



