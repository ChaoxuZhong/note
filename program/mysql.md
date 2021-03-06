mysql

##### mysql结构

连接器——缓存（高版本已经取消）——分析器（词义分析，语法分析）——优化器——执行器——存储引擎



##### sql 执行流程

redo log： innoDB特有，记录在数据页上做的修改，使用循环写入（双指针）

binlog ：mysql service自带，记录语句原始逻辑，追加写

调用引擎后，先写redolog，然后service层写binlog，写成功commit，两阶段提交



##### redolog准确恢复数据库，binlog不可以

redolog写入磁盘后就擦除，redolog中的都是还没写入磁盘的

binlog不知道消息是否写入磁盘，没有标记

恢复数据库的几种情况：

>1.redolog 第一阶段提交了  binlog失败  发现binlog失败redolog不生效
>
>2.redolog 第一阶段提交了 binlog 成功 redolog未commit 也提交
>
>3.redolog commit了 那就已经一致了



问题：多个SOL组成的事务如何处理，如何回滚binlog？  —— MVCC 保证，怎么保证？

redolog写满后会擦除，会导致数据丢失吗，如果binlog的备份频率不高？ —— 这是另一个维度的问题，binlog记录好了不能丢。



##### Explain And Index

主键索引为聚簇索引(clustered index)

非主键索引为二级索引(secondary index)

commit work and chain 可以省begin 操作 



##### 事务隔离级别有哪些，如何实现的

> read uncommited  读未提交 ——直接读最新值
>
> read commited 读已提交——在SQL执行时创建
>
> repeatable read 可重复读——事务启动时创建视图，所以可以重复读
>
> serializable 串行化——直接使用加锁避免并行访问



##### 两阶段锁

行锁是在需要锁的时候加上的，在事务commit时解锁。

因此：在事务中需要锁多个行的时候，把最可能影响并发的锁网后面放。



##### 死锁检测

事务中可能发生死锁，可通过等待超时解决问题，但时间50s一般太长，太短又容易误伤，提前解锁

使用死锁检测，但是死锁检测会降低并发度，大量使用cpu。



##### 事务启动具体时机

事务不是在begin时启动，是在第一次操作innoDB引擎时启动。

想在begin时启动，使用start transaction with consistent snapshot;



##### read view

每个事务启动时都会创建一个视图，即创建一个事务数组，记录创建了未提交的事务，下边界为低水位，上边界为高水位

数据是否可见：  

>1.如果事务id高于高水位，不可见
>
>2.如果事务id低于低水位，可见
>
>3.如果事务id大于低水位，小于高水位
>
>>a.在事务数组中，不可见，因为创建的时候还没提交嘛
>>
>>b.不在事务数组中，可见，因为创建的时候已经提交了

所有的不可见都通过redolog回滚的得到可见的数据版本

翻译一下数据是否可见

>事务未提交，不可见
>
>事务已提交，在视图创建后提交，不可见
>
>事务已提交，在视图创建前提交，可见



##### 当前、快照读

快照读是在事务隔离级别下，读取的readview。

当前读是每次更新前update语句自动读最新的数据。



##### change buffer 思想

写的时候先不落盘，有查询的时候才merge落盘，减少随机IO操作，适合写多读少的场景。



##### 加索引的方式

>1.直接全字段加
>
>2.部分加、如果区分度不高比如身份证号可以倒序部分加
>
>3.hash加，但是也需要equal一下（回表）



##### 脏页和干净页

内存数据与磁盘中数据不同为脏页，相同为干净页



##### 数据库刷脏页的时机及带来的问题

>1.粉板 redolog 写满了需要刷——日志写满，所有更新堵住，不可接受
>
>2.内存不足淘汰一些页面，干净页直接淘汰太，脏页刷磁盘——【脏页控制策略】
>
>3.系统不繁忙的时候，抽空刷粉版——无需关心
>
>4.数据库正常退出时——无需关心

【脏页控制策略】

>1.首先需要主机IO能力，告诉innoDB，可以设置为磁盘的IOPS

刷了盘一定就是正确的数据吗，不一定commit了啊,读出来的数据确实不一定commit了，不知道怎么处理。