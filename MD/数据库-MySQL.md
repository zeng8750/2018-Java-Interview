
## MySQL
### 引擎对比
1. InnoDB支持事务
2. InnoDB支持外键
3. InnoDB有行级锁，MyISAM是表级锁

MyISAM相对简单所以在效率上要优于InnoDB。如果系统插入和查询操作多，不需要事务外键。选择MyISAM  
如果需要频繁的更新、删除操作，或者需要事务、外键、行级锁的时候。选择InnoDB。

### [数据库性能优化](https://www.zhihu.com/question/19719997)
1. 优化SQL语句和索引，在where/group by/order by中用到的字段建立索引，索引字段越小越好，复合索引建立的顺序
2. 加缓存，Memcached, Redis
3. 主从复制，读写分离
4. 垂直拆分，其实就是根据你模块的耦合度，将一个大的系统分为多个小的系统，也就是分布式系统
5. 水平切分，针对数据量大的表，这一步最麻烦，最能考验技术水平，要选择一个合理的sharding key,为了有好的查询效率，表结构也要改动，做一定的冗余，应用也要改，sql中尽量带sharding key，将数据定位到限定的表上去查，而不是扫描全部的表；

### [SQL优化](https://www.imooc.com/video/3711)：

* 对经常查询的列建立索引，但索引建多了当数据改变时修改索引会增大开销
* 使用精确列名查询而不是*，特别是当数据量大的时候
* 减少[子查询](http://www.cnblogs.com/zhengyun_ustc/p/slowquery3.html)，使用Join替代
* 不用NOT IN，因为会使用全表扫描而不是索引；不用IS NULL，NOT IS NULL，因为会使索引、索引统计和值更加复杂，并且需要额外一个字节的存储空间。

问：max(xxx)如何用索引优化？
答：在xxx列上建立索引，因为索引是B+树顺序排列的，锁在下次查询的时候就会使用索引来查询到最大的值是哪个

问：如何对分页进行优化？
答：`SELECT * FROM big_table order by xx LIMIT 1000000,20`，这条语句会查询出1000020条的所有数据然后丢弃掉前1000000条，为了避免全表扫描的操作，在order by的列上加**索引**就能通过扫描索引来查询。但是这条语句会查询还是会扫描1000020条，还能改进成`select id from big_table where id >= 1000000 order by xx LIMIT 0,20`，用ID作为**过滤**条件将不需要查询的数据直接去除。

#### 最左前缀匹配原则
在mysql建立联合索引时会遵循最左前缀匹配的原则，即最左优先，在检索数据时从联合索引的最左边开始匹配。mysql会一直向右匹配直到遇到范围查询（>/</between/like）就停止匹配，比如a=3 and b=4 and c>5 and d=6，如果建立（a,b,c,d）联合索引，d就用不到索引，如果建立（a,b,d,c）则都能用到索引。

问：有个表特别大，字段是姓名、年龄、班级，如果调用`select * from table where name = xxx and age = xxx`该如何通过建立索引的方式优化查询速度？  
答：由于mysql查询每次只能使用一个索引，如果在name、age两列上创建联合索引的话将带来更高的效率。如果我们创建了(name, age)的联合索引，那么其实相当于创建了(name, age)、(name)三个索引，这是联合索引的特性，与联合索引的实现原理有关，也是最佳左前缀特性。因此我们在创建复合索引时应该将最常用作限制条件的列放在最左边，依次递减。其次还要考虑该列的数据离散程度，如果有很多不同的值的话建议放在左边，name的离散程度也大于age。

## 事务隔离级别
1. 原子性（Atomicity）：事务作为一个整体被执行 ，要么全部执行，要么全部不执行；
2. 一致性（Consistency）：在事务开始之前和事务结束以后，数据库的完整性没有被破坏。这表示写入的资料必须完全符合所有的预设规则，这包含资料的精确度、串联性以及后续数据库可以自发性地完成预定的工作；
3. 隔离性（Isolation）：多个事务并发执行时，一个事务的执行不应影响其他事务的执行，然后你可以扯到隔离级别；
4. 持久性（Durability）：一个事务一旦提交，对数据库的修改应该永久保存。

[https://www.jianshu.com/p/4e3edbedb9a8](https://www.jianshu.com/p/4e3edbedb9a8)

## 锁表、锁行
[https://segmentfault.com/a/1190000012773157](https://segmentfault.com/a/1190000012773157)

### 悲观锁乐观锁、如何写对应的SQL
[https://www.jianshu.com/p/f5ff017db62a](https://www.jianshu.com/p/f5ff017db62a)

### 索引
#### 原理
我们拿出一本新华字典，它的目录实际上就是一种索引：非聚集索引。我们可以通过目录迅速定位我们要查的字。而字典的内容部分一般都是按照拼音排序的，这实际上又是一种索引：聚集索引。聚集索引这种实现方式使得按主键的搜索十分高效，但是辅助索引搜索需要检索两遍索引：首先检索辅助索引获得主键，然后用主键到主索引中检索获得记录。

使用B+树来构建索引，为什么不用二叉树？红黑树在磁盘上的查询性能远不如B+树。原因是红黑树最多只有两个子节点，所以高度会非常高，导致遍历查询的次数会多，又因为红黑树在数组中存储的方式，导致逻辑上很近的父节点与子节点可能在物理上很远，导致无法使用磁盘预读的局部性原理，需要很多次IO才能找到磁盘上的数据；但B+树一个节点中可以存储很多个索引的key，且将大小设置为一个页，一次磁盘IO就能读取很多个key，且叶子节点之间还加上了下个叶子节点的指针，遍历索引也会很快。

B+树的高度如何计算？在Linux里，每个页默认4KB，假设索引的是8B的long型数据，每个key后有个页号4B，还有6B的其他数据（参考《MySQL技术内幕：InnoDB存储引擎》P193的页面数据），那么每个页的扇出系数为4KB/(8B+4B+6B)=227，即每个页可以索引245个key。在高度h=3时，s=227^3=1100万。通常来说，索引树的高度在2~4。

#### 优化
如何选择合适的列建立索引？

1. WHERE / GROUP BY / ORDER BY / ON 的列
2. 离散度大（不同的数据多）的列使用索引才有查询效率提升
3. 索引字段越小越好，因为数据库按页存储的，如果每次查询IO读取的页越少查询效率越高

## 分区分库分表
分区：把一张表的数据分成N个区块，在逻辑上看最终只是一张表，但底层是由N个物理区块组成的，通过将不同数据按一定规则放到不同的区块中提升表的查询效率。

水平分表：为了解决单标数据量过大（数据量达到千万级别）问题。所以将固定的ID hash之后mod，取若0~N个值，然后将数据划分到不同表中，需要在写入与查询的时候进行ID的路由与统计

垂直分表：为了解决表的宽度问题，同时还能分别优化每张单表的处理能力。所以将表结构根据数据的活跃度拆分成多个表，把不常用的字段单独放到一个表、把大字段单独放到一个表、把经常使用的字段放到一个表

分库：面对高并发的读写访问，当数据库无法承载写操作压力时，不管如何扩展slave服务器，此时都没有意义了。因此数据库进行拆分，从而提高数据库写入能力，这就是分库。

问题：

1. 事务问题。在执行分库之后，由于数据存储到了不同的库上，数据库事务管理出现了困难。如果依赖数据库本身的分布式事务管理功能去执行事务，将付出高昂的性能代价；如果由应用程序去协助控制，形成程序逻辑上的事务，又会造成编程方面的负担。
2. 跨库跨表的join问题。在执行了分库分表之后，难以避免会将原本逻辑关联性很强的数据划分到不同的表、不同的库上，我们无法join位于不同分库的表，也无法join分表粒度不同的表，结果原本一次查询能够完成的业务，可能需要多次查询才能完成。
3. 额外的数据管理负担和数据运算压力。额外的数据管理负担，最显而易见的就是数据的定位问题和数据的增删改查的重复执行问题，这些都可以通过应用程序解决，但必然引起额外的逻辑运算。

欢迎光临[我的博客](http://www.wangtianyi.top/?utm_source=github&utm_medium=github)，发现更多技术资源~
