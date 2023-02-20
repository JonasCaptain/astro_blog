---
layout: '@/templates/BasePost.astro'
title: MySQL学习笔记
description: 主要摘抄至出版书籍
pubDate: 2023-03-20T00:00:00Z
imgSrc: '/assets/images/image-post.jpeg'
imgAlt: '认真、专注'
draft: true
---



## 索引

### 简介

索引有两种，一种是BTree索引，一种是Hash索引

#### 分类

1. 普通索引：不添加任何限制条件
2. 唯一性索引：使用Unique参数，确保该字段中所有的value具有唯一性
3. 全文索引：使用FullText参数，只能创建在Char、VarChar、Text类型的字段上，只有MyISAM存储引擎支持全文索引
4. 单列索引：在一个字段上建立的普通索引，唯一性索引或全文索引
5. 多列索引：在多个字段上建立的普通索引，唯一性索引或全文索引
6. 空间索引：使用Spatial参数，只有MyISAM存储引擎支持空间索引，必须建立在空间数据类型上，且必须非空

#### 索引的设计原则

1. 选择唯一性索引

2. 为经常需要排序、分组和联合操作的字段建立索引

   如：order by、group by、distinct、union等操作的字段，特别是排序

3. 为常作为查询条件的字段建立索引

4. 限制索引的数目（避免过多的浪费空间）

5. 尽量使用数据量少的索引

6. 尽量使用前缀来索引

   如：索引Text类型字段的前N个字符

7. 删除不再使用或者很少使用的索引

#### 创建索引

三种方式：

* 创建表时创建索引(create table xxx 附带)
* 已经存在的表上创建索引(create index)
* 使用alter table语句来创建索引(alter table add index)

#### 创建表时创建索引

```sql
## 普通索引
create table index1(id int,
                    name varchar(20),
                       sex boolean,
                       index(id));
## 唯一性索引
create table index2(id int unique,
                       name varchar(20));

create table index2(id int,
                       name varchar(20),
                    unique index index2_id(id ASC));

## 全文索引
create table index3(id int,
                   info varchar(20),
                   fulltext index index3_info(info))
                   engine=MyISAM;
## 创建单列索引
create table index4(id int,
                       subject varchar(30),
                       index index4_st(subject(10)));
                       ## 只索引subject前10个字符
## 创建多列索引
create table index5(id int,
                       name varchar(20),
                       sex char(4),
                       index index5_ns(name, sex));
explain select * from index5 where name = '123'\G;
explain select * from index5 where name = '123' and sex = 'N'\G;

## 创建空间索引
create table index6(id int,
                       space geometry not null,
                   spatial index index6_sp(space))
                   engine=MyISAM;
```

#### 在已经存在的表上创建索引

```sql
CREATE [UNIQUE|FULLTEXT|SPATIAL] INDEX 索引名 ON 表名(属性名[(长度)] [ASC|DESC]);

## 创建普通索引
create index index7_id on index7(id);
## 创建唯一性索引
create unique index8_id on index8(course_id);
## 创建全文索引
create fulltext index index9_info on index9(info);
## 创建单列索引
create index index10_addr on index10(address(4));
## 创建多列索引
create index index11_na on index11(name, address);
## 创建空间索引
create spatial index12_line on index12(line);
```

#### 用alter table语句创建索引

```sql
ALTER TABLE 表名 ADD [UNIQUE|FULLTEXT|SPATIAL] INDEX 索引名 (属性名[(长度)] [ASC|DESC]);

## 创建普通索引
alter table index13 add index index13_name(name(20));
## 创建唯一性索引
alter table index14 add unique index index14_id(course_id);
## 创建全文索引
alter table index15 add fulltext index index15_info(info);
## 创建单列索引
alter table index16 add index index16_addr(address(4));
## 创建多列索引
alter table index17 add index index17_na(name, address);
## 创建空间索引
alter table index18 add spatial index index18_line(line);
```

#### 删除索引

```sql
drop index 索引名 on 表名;
drop index id on index1;
```

#### B+Tree索引和Hash索引的区别

##### 特点

* Hash索引结构的特殊性，其检索效率非常高，索引的检索可以一次定位
* B+Tree索引需要从根节点到枝节点，最后才能访问到叶节点这样多次的**IO**访问

##### 为什么不都用Hash索引而使用B+Tree索引？

1. Hash索引仅仅能满足“=”、“IN”和“”查询，不能使用范围查询，因为经过相应的Hash算法处理之后的Hash值的大小关系，并不能保证和Hash运算前完全一样
2. Hash索引无法被用来避免数据的排序操作，因为Hash值的大小关系并不一定和Hash运算前的键值完全一样
3. Hash索引不能利用部分索引键查询，对于组合索引，Hash索引在计算Hash值的时候，是组合索引键合并后再一起计算Hash值，而不是单独的计算Hash值，所以通过组合索引的前面一个或几个索引键进行查询的时候，Hash索引也无法被利用
4. Hash索引在任何时候都不能避免表扫描，由于不同索引键存在相同Hash值，所以即使满足某个Hash键值数据的记录条数，也无法从Hash索引中直接完成查询，还是要回表查询数据
5. Hash索引遇到大量Hash值相等的情况后性能并不一定就会比B+Tree索引高

##### 补充

1. MySQL中，只有HEAP/MEMORY引擎才显式支持Hash索引
2. 常用的**InnoDB**引擎中默认使用的是B+Tree索引，它会实时监控表上索引的使用情况，如果认为建立Hash索引可以提高查询效率，则自动在内存中的“自适应Hash索引缓冲区”建立Hash索引(在InnoDB中默认开启自适应Hash索引)，通过观察搜索模式，MySQL会利用index key的前缀建立Hash索引，入股过一个表几乎大部分都在缓冲池中，那么建立一个Hash索引能够加快等值查询
3. 如果是等值查询，那么Hash索引明显有绝对优势，因为只需要经过一次算法即可找到相应的键值；当然了，这个前提是，键值都是唯一的。如果键值不是唯一的，就需要先找到该键所在的位置，然后再根据链表往后扫描，直到找到相应的数据
4. 如果是范围查询检索，这时候Hash索引就毫无用武之地了，因为原先是有序的键值，经过Hash算法后，有可能变成不连续的了，就没办法再利用索引完成范围查询检索；同理Hash索引没办法利用索引完成排序，以及 like 'xx%'这样的部分模糊查询(这种部分模糊查询，本质上也是范围查询)
5. Hash索引不支持多列联合索引的最左匹配规则
6. B+Tree索引的关键字检索效率比较平均，不像BTree那样波动幅度大，在有大量重复键值情况下，Hash索引的效率也是极低的，因为存在所谓的Hash碰撞问题
7. 在大多数场景下，都会有范围查询、排序、分组等查询特征，用B+Tree索引就可以了

#### BTree与B+Tree的区别

* BTree，每个节点存储Key和Data，所有节点组成这颗树，并且叶子节点指针为null，叶子节点不包含任何关键字信息
* B+Tree，所有的叶子节点中包含了全部关键字的信息，及指向含有这些关键字记录的指针，且叶子节点本身依关键字的大小自小而大的顺序链接，所有的非终端节点可以看成是索引部分，节点中仅含有其子树根节点中最（或最小）关键字。（而BTree的非终节点也包含需要查找的有效信息）

#### 为什么说B+Tree比BTree更适合实际应用中操作系统的文件索引和数据库索引？

* B+的磁盘读写代价更低
  
  B+的内部节点并没有指向关键字具体信息的指针。因此其内部节点相对BTree更小。如果把所有同一内部节点的关键字存放在同一块盘中，那么盘所能容纳的关键字数量也越多。一次性读入内存中的需要查找的关键字也就越多。相对来说IO读写次数也就降低了。

* B+Tree的查询效率更加稳定
  
  由于非终节点并不是最终指向文件内容的节点，而只是叶子节点中的关键字的索引。所以任何关键字的查找必须走一条从根节点到叶子节点的路。所有关键字查询的路径长度相同，导致每一个数据的查询效率相当。

#### 聚簇索引和非聚簇索引的区别

1. 聚簇索引(clustered index):

   聚簇索引表记录的排列顺序和索引的排列顺序一致，所以查询效率快，只要找到第一个索引值记录，其余就连续性的记录在物理也一样连续存放。聚簇索引的对应的缺点就是修改慢，因为为了保证表中记录的物理和索引顺序一致，在记录插入的时候，会对数据页重新排序。

   聚簇索引类似于新华字典总用拼音去查找汉字，拼音检索表的顺序都是按照a~z排列的，就想相同的逻辑顺序与物理顺序一样，当你需要查找a,ai两个读音的字，或是想一次寻找多个sha的同音字时，也许向后翻几页，或紧接着下一行就得到结果了。

2. 非聚簇索引(nonclustered index):

   非聚簇索引指定了表中记录的逻辑顺序，但是记录的物理和索引不一定一致，两种索引都采用B+Tree结构，非聚簇索引的叶子层并不和实际数据页相重叠，而采用叶子层包含一个指向表中的记录在数据页中的指针方式。非聚簇索引层次多，不会造成数据重排。

   非聚簇索引类似在新华字典上通过偏旁部首来查询汉字，检索表也是按照横、竖、撇来排列的，但是由于正文中是a~z的拼音顺序，所以就类似于逻辑地址与物理地址的不对应。同时适用的情况就在于分组，大数目的不同值，频繁更新的列中，这些情况即不适合聚簇索引。

**根本区别**：

聚簇索引和非聚簇索引的根本区别是表记录的排列顺序和与索引的排列顺序是否一致。

## 事务

### 1.什么是事务

事务是对数据库中一些操作进行统一的回滚或者提交的操作，主要用来保证数据的完整性和一致性。

### 2.事务四大特性(ACID)

**原子性-Atomicity：**

指事务包含的操作要么全部成功，要么全部失败回滚，因此事务的操作如果操作成功就必须要完全应用到数据库，如果操作失败则不能对数据库有任何影响。

**一致性-Consistency：**

事务开始前和结束后，数据库的完整性约束没有被破坏。比如A向B转账，不能A扣了钱，B却没收到。

**隔离性-Isolation：**

隔离性是当多个用户并发访问数据库时，比如操作同一张表时，数据库为每一个用户开启事务，不能被其他事务的操作所干扰，多个并发事务之间要相互隔离。同一时间，只允许一个事务请求同一数据，不同的事务之间彼此没有任何干扰。比如A正在从一张银行卡中取钱，在A取钱的过程结束前，B不能向这张卡转账。

**持久性-Durability：**

持久性是指一个事务一旦被提交了，那么对数据库总的数据的改变就是永久性的，即便在数据库系统遇到故障的情况下也不会丢失提交事务的操作。

### 3.事务的并发？事务隔离级别，每个级别会引发什么问题，MySQL默认是哪个级别？

理论上来说，事务应该彼此完全隔离，以避免并发事务所导致的问题。然后，那样会对性能产生机打的影响，因为事务必须按顺序运行，在实际开发中，为了提升性能，事务会以较低的隔离级别运行，事务的隔离级别可以通过隔离事务属性指定。

#### 事务并发带来的问题

1. **脏读**：事务A读取了事务B更新的数据，然后B回滚操作，那么A读取到的数据就是脏数据
2. **不可重复读**：事务A多次读取同一数据，事务B在事务A多次读取的过程中，对数据做了更新并提交，导致事务A多次读取同一数据时，结果因为事务B先后两次读取到的数据结果会不一致
3. **幻读**：幻读解决了不可重复读，保证了同一个事务里，查询的结果都是事务开始的状态（一致性）

**小结**：不可重复读和幻读很容易混淆，不可重复读侧重于修改，幻读侧重于新增或删除。解决不可重复读的问题只需要锁住满足条件的行，解决幻读需要锁表。

#### 事务的隔离级别

**读未提交：**另一个事务修改了数据，但尚未提交，而本事务中的 SELECT会读到这些未提交的数据。

**不可重复读：**事务A多次读取同一数据，事务B在事务A多次读取的过程中，对数据做了更新并提交，导致事务A多次读取同一数据时，结果因为事务B先后两次读取到的数据结果会不一致。

**可重复读：**在同一个事务里，SELECT的结果是事务开始时间点的状态，因此同样的SELECT操作读到的结果会是一致的。但是会有幻读现象。

**串行化：**最高的隔离级别，在这个隔离级别下，不会产生任何异常。并发的事务，就像事务是在一个个按照顺序执行一样。

**特别注意：**

MySQL默认的事务隔离级别为 repeatable-read(可重复读)

MySQL支持4种隔离级别

事务的隔离级别要得到底层数据库引擎的支持，而不是应用程序或者框架的支持。

Orale支持的2种隔离级别：read_commited，serializable

MySQL中默认的事务隔离级别并不会锁住读取到的行

| 事务隔离级别 | 操作          |
| ------ | ----------- |
| 未提交读   | 写数据只会锁住相应的行 |
| 可重复读   | 写数据会锁住整张表   |
| 串行化    | 读写数据都会锁住整张表 |

#### 4.事务传播行为

1. PROPAGATION_REQUIRED：如果当前没有事务，就创建一个新事务，如果当前存在事务，就加入该事务，该设置是最常用的设置。

2. PROPAGATION_SUPPORTS：支持当前事务，如果当前存在事务，就加入该事务，如果当前不存在事务，就以非事务执行。

3. PROPAGATION_MANDATORY：支持当前事务，如果当前存在事务，就加入该事务，如果当前不存在事务，就抛出异常。

4. PROPAGATION_REQUIRES_NEW：创建新事务，无论当前存不存在事务，都创建新事务。

5. PROPAGATION_NOT_SUPPORTED：以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。

6. PROPAGATION_NEVER：以非事务方式执行，如果当前存在事务，则抛出异常。

7. PROPAGATION_NESTED：如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作。

#### 5.嵌套事务

什么是嵌套事务？

嵌套是指子事务套在父事务中执行，子事务是父事务的一部分，在进入事务之前，父事务建立一个回滚点，叫save point，然后执行子事务，这个子事务的执行也算是父事务的一部分，然后子事务执行结束，父事务继续执行。重点就在于那个save point。

如果子事务回滚，会发生什么？

父事务会回滚到进入子事务前建立的save point，然后尝试其他的事务或者其他的业务逻辑，父事务之前的操作不会受到影响，更不会自动回滚。

如果父事务回滚，会发生什么？

父事务回滚，子事务也会跟着回滚！因为父事务结束之前，子事务是不会提交的。

事务的提交，是什么情况？

子事务先提交，父事务再提交，子事务是父事务的一部分， 由父事务统一提交。

## 存储引擎

### 1.MySQL常见的三种存储引擎(InnoDB、MyISAM，MEMORY)的区别？

1. InnoDB支持事务，MyISAM不支持。事务是一种高级的处理方式，如在一些列增删改中只要那个出错还可以回滚还原，而MyISAM就不行。
2. MyISAM适合查询以及插入为主的应用。
3. InnoDB适合频繁修改以及涉及到安全性较高的应用。
4. InnoDB支持外键，MyISAM不支持。
5. 从MySQL5.5之后，InnoDB是默认引擎。
6. InnoDB不支持FullText类型的索引。
7. InnoDB中不保存表的行数，如SELECT COUNT() from table时，InnoDB需要扫描一遍整个表来计算有多少行，但是MyISAM只要简单的读出保存好的行数即可。注意的是，当count()语句包含where条件时，MyISAM也需要扫描整个表。
8. 对于自增长的字段，InnoDB中必须包含只有该字段的索引，但是在MyISAM表中，可以和其他字段一起建立联合索引。
9. DELETE FROM table时，InnoDB不会重新建立表，而是一行一行的删除，效率非常慢，MyISAM则会重新建表。
10. InnoDB支持行锁(某些情况还是锁整表，如update table set a = 1 where user like '%lee%')。

### 2.MySQL存储引擎中的MyISAM与InnoDB如何选择？

MySQL实际上有很多引擎，每个引擎都有各自的优缺点，可以择优选择使用：MyISAM，InnoDB，MERGE、MEMORY（HEAP）、BDB（BerkeleyDB）、EXAMPLE、FEDERATED、ARCHIVE、CSV、BLACKHOLE。

虽然引擎种类很多，但常用的就是InnoDB和MyISAM

1. InnoDB会支持一些数据库高级功能，如事务功能和行级锁，MyISAM不支持。
2. MyISAM的性能更优，占用的存储空间少，所以选择何种引擎，视具体情况而定。

MEMORY是MySQL汇总一类特殊的存储引擎，它使用存储在内存中的内容来创建表，而且数据全部放在内存中。每个基于MEMORY存储引擎的表，实际对应一个磁盘文件。改文件的文件名与表名相同，类型为frm类型。该文件中只存储表的结构。而其数据文件，都是存储在内存中，这样有利于数据的快速处理，提高整个存储引擎的表的使用。如果不需要了，可以释放内存，甚至删除不需要的表。

### 3.MySQL的MyISAM与InnoDB在事务、锁级别各自的使用场景？

事务处理方面：

* MyISAM强调的是性能，每次查询具有原子性，其执行效率比InnoDB快，但不支持事务
* InnoDB提供事务支持，外键等高级数据库功能。具有事务提交(commit)、事务回滚(rollback)和崩溃修复能力(crash recovery capabilites)的事务安全型表

锁级别：

* MyISAM只支持表级锁，用户在操作MyISAM表时，SELECT、UPDATE、DELETE、INSERT语句都会给表自动加锁，如果加锁以后的表满足INSERT并发的情况下，可以在表的尾部插入新的数据。
* InnoDB支持事务后行级锁，是InnoDB最大的特色。行锁大幅度提高了多用户并发操作的性能。但是InnoDB的行数，只是在Where的主键是有效的前提下，非主键的Where都会锁全表的。

## 数据库锁

### 1.MySQL都有什么锁，死锁判定原理和具体场景，死锁怎么解决？

#### MySQL有三种锁

* 表级锁：开销小，加锁快；不会出现死锁；锁粒度最大，发生锁冲突的概率最大，并发度最低
* 行级锁：开销大，加锁慢；会出现死锁；锁粒度最小，发生冲突的概率最低，并发度也最高
* 页级锁：开销和加锁的时间介于表级锁和行级锁；会出现死锁；锁粒度介于表锁和行锁之间，并发度一般

#### 什么是死锁？

**死锁：**指两个或两个以上的进程在执行过程中，因争夺资源而造成的一项互相等待的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程称为死锁进程。

表级锁不会产生死锁，所以解决死锁主要还是针对于最常用的InnoDB。

**死锁的关键在于：**两个（或以上）Session加锁的顺序不一致。那么对应的解决死锁问题的关键就是，让不同的session加锁有次序。

#### 死锁的解决办法

1. 查出死锁的线程并kill掉

   ```mysql
   SELECT trx_MySQL_thread_id FROM information_schema.INNODB_TRX;
   ```

2. 设置锁的超时时间

   InnoDB行锁的等待时间，单位秒，可在会话级别设置，RDS实例该参数的默认值为50秒

   生产环境不推荐使用过大的 inoodb_lock_wait_timeout 参数值，该参数支持在会话级别修改，方便应用在会话级别单独设置某些特殊操作的行锁等待超时时间。如下：

   ```mysql
   set innodb_lock_wait_timeout = 1000;
   ```

3. 指定获取锁的顺序

#### 2. 有哪些锁，SELECT时怎么加排它锁？

##### 加锁机制

* 悲观锁（Pessimistic Lock）：
  
  特点：先获取锁，在进行业务操作。
  
  即“悲观”的认为获取锁是非常有可能失败的，因此要先确保获取锁成功在进行业务操作。通常所说的“一锁二查三更新”即指的是使用悲观锁。通常来讲在数据库上的悲观锁需要数据库本身提供自持，即通过常用的 select .... for update 操作来实现悲观锁。当数据库执行select .... for update时会获取被select 中的数据行的行锁，因此其他并发执行的select .... for update如果试图选择同一行则会发生排斥(需要等待行锁被释放)，因此达到锁的效果。select .... for update 获取的行锁会在当前事务结束时自动释放，因此必须在事务中使用。
  
  **补充**：不同的数据库对 select .... for update 的实现和支持都是有区别的。
  
  * Oracle支持 select .... for update no wait，表示如果拿不到锁立刻报错，而不是等待，MySQL就没有 no wait这个选项。
  * MySQL还有个问题是 select .... for update语句执行中所有扫描过的行都会被锁上，这一点很容易造成问题。因此如果在MySQL中用悲观锁务必要确定走了索引，而不是全表扫描。

* 乐观锁（Optimistic Lock）：
  
  1. 乐观锁，也叫乐观并发控制，它假设多用户并发的事务在处理时不会彼此互相影响，各事务能够在不产生锁的情况下处理各自影响的那部分数据。在提交数据更新之前，每个事务会先检查在该事务读取数据库后，有没有其他事务又修改了该数据。如果其他事务有更新的话，那么当前正在提交的事务会进行回滚。
  
  2. 乐观锁的特点是先进行业务操作，不得万不得已不去拿锁。即“乐观”的认为拿锁多半是会成功的，因此在进行完业务操作需要实际更新的最后一步再去拿锁就好。乐观锁在数据上的实现完全是逻辑的，不需要数据库提供特殊的支持。
  
  3. 一般的做法是在需要锁的数据上增加一个版本号，或者时间戳。

     实现方式举例如下：

     乐观锁（给表加一个版本号字段），这个并不是乐观锁的定义，给表加版本号，是数据实现乐观锁的一种方式。

     1. seelct data as old_data, version as old_version from ...;
     2. 根据获取的数据进行业务操作，得到new_data和new_version
     3. update table set data = new_data, version = new_version where version = old_version

     ```code
     if (updated row > 0 ) {
     
     ​    //乐观锁获取成功，操作完成
     
     } else {
     
     ​    //乐观锁获取失败，回滚并重试
     
     }
     ```

     **注意：**

     * 乐观锁在不发生取锁失败的情况下开销比乐观锁小，但是一旦发生失败回滚开销则比较大，因此适合用在取锁失败概率比较小的场景，可以提升系统并发性能
     * 乐观锁还适用于一些特殊的场景，例如在业务操作过程中无法和数据库保持连接等悲观锁无法使用的地方

#### **总结：**

悲观锁和乐观锁是数据库用来保证数据并发安全防止更新丢失的两种方法。悲观锁和乐观锁大部分场景下差异不大，一些独特的场景下有一些差异，一般可以从如下几个方面来判断：

* 响应速度：如果需要非常高的响应速度，建议采用乐观锁方案，成功就执行，不成功就失败，不需要等待其他并发去释放锁。
* 冲突频率：如果冲突频率非常公安，建议是采用悲观锁，保证成功率，如果冲突频率大，乐观锁会需要多次重试才能成功，代价比较大。
* 重试代价：如果重试代价大，建议采用悲观锁。

#### 扩展知识点

行级锁是MySQL中锁定粒度最细的一种锁，行级锁能大大减少数据库操作的冲突，行级锁分为共享锁和排它锁两种。

**共享锁(Share Lock)：**

共享锁又称读锁，是读取操作创建的锁。其他用户可以并发读取共享锁，不能加排它锁。获取共享锁的事务只能读取数据，不能修改数据。

用法：

```sql
select ... lock in share mode;
```

在查询语句后面增加lock in share mode，MySQL就会对查询结果中的每行都加共享锁，当没有其他线程对查询结果集中的任何一行使用排它锁时，可以成功申请共享锁，否则会被阻塞。其他线程也可以读取使用了共享锁的表，而且这些线程读取的是同一个版本的数据。

**排它锁(Exclusive Lock)：**

排它锁又称写锁、独占锁，如果事务T对数据A加上排它锁后，则其他事务不能再对A加任何类型的封锁。获取排它锁的事务既能读取数据，又能修改数据。

用法：

```sql
select ... for update;
```

在查询语句后面加上 for update，MySQL就会对查询结果集中的每一行都加排它锁，当没有其他线程对查询结果集中的任何一行使用排它锁时，可以成功申请排它锁，否则会被阻塞。

**意向锁(Intention Lock)：**

意向锁是表级锁，其设计目的主要是为了在一个事务中揭示下一行将要被请求锁的类型。

InnoDB中的两个表锁：

* 意向共享锁(IS)：表示事务准备给数据行加入共享锁，也就说一个数据行加共享锁前必须先取得该表的IS锁；
* 意向排它锁(IX)：类似上面，表示事务准备给数据行加入排它锁，说明事务在一个数据行加排它锁前必须先取得该表的IX锁；

意向锁是InnoDB自动加的，不需要用户干预。

对于INSERT、UPDATE和DELETE，InnoDB会自动给涉及的数据加排它锁；对于一般的SELECT语句，InnoDB不会加任何锁，事务可以通过以下语句显式加共享锁或排它锁。

```sql
## 共享锁
select ... lock in share mode;
## 排它锁
select ... for update;
```

#### 意向锁的引入解决了什么问题？

> 假设，事务A获取了某一行的排它锁，尚未提交，此时事务B想要获取表锁时，必须要确认表的每一行都不存在排它锁，很明显效率会很低，引入意向锁之后，效率就会大为改善。

1. 如果事务A获取了某一行的排它锁，实际此表存在两种锁，表中某一行的排它锁和表上的意向排它锁；
2. 如果事务B试图在该表级别上加锁时，则受到上一个意向锁的阻塞，它在锁定该表前不必检查各个页或行锁，而只需要检查表上的意向锁。

**宏观角度理解锁(图)**
![mysql锁](../../../public/assets/images/mysql/MySQL-锁.jpg)

从锁粒度角度来讲，InnoDB允许行级锁与表级锁共存，而意向锁是表锁；从锁模式角度看，意向锁是一种独立类型，辅助解决记录锁效率不及的问题；从兼容性角度，意向锁包含了共享/排它两种。

**总结下意向锁的几个特征：**

1. 意向锁是表锁。做兼容性比照时，一定要分清是表锁还是行锁，如下图所示：

   | 兼容性    | IS  | IX  | S   | X   |
   | ------ | --- | --- | --- | --- |
   | **IS** | 兼容  | 兼容  | 兼容  | 不兼容 |
   | **IX** | 兼容  | 兼容  | 不兼容 | 不兼容 |
   | **S**  | 兼容  | 不兼容 | 兼容  | 不兼容 |
   | **X**  | 不兼容 | 不兼容 | 不兼容 | 不兼容 |

   这里的排它（X）、共享（S）,说的是表锁。表级锁不会和行级锁做比对，这里特别容易混淆。意向锁不会与行级锁中的共享(S)、排它(X)互斥。

2. 用户无法操作意向锁，意向锁是由InnoDB自己维护的。说白了，意向锁是帮助InnoDB提高效率的一种手段。

举例：

表

| ID  | name      |
| --- | --------- |
| 1   | ROADHOG   |
| 2   | Reinhardt |
| 3   | Tracer    |
| 4   | Genji     |
| 5   | Hanzo     |
| 6   | Mccree    |

事务A先获取了某一行的排它锁，并未提交：

```sql
select * from users where id = 6 for update;
```

1. 事务A获取了users表上的**意向排它锁**。
2. 事务A获取了id为6的数据行上的**排它锁**。

之后事务B想要获取users表的共享锁：

```sql
## 这是共享锁，表级锁
lock tables users READ;
## 这是排它锁，表级锁
lock tables user WRITE;
```

1. 事务B检测到事务A持有users表的**意向排它锁**。
2. 事务B对users表的加锁请求被阻塞（排斥）。

最后事务C也想获取users表中某一行的排它锁：

```sql
select * from users where id = 5 for update;
```

1. 事务C申请users表的**意向排它锁**。
2. 事务C检测到事务A持有users表的**意向排它锁**。
3. 因为意向锁之间并不互斥，所以事务C获取到了users表的意向排它锁。
4. 因为id为5的数据行上不存在任何**排它锁**，最终事务C成功获取到了该数据行上的排它锁。

**总结：**

1. InnoDB支持多粒度锁，特定场景下，行级锁可以与表级锁共存。
2. 意向锁之间互不排斥，但除了IS与S兼容外，意向锁会与 共享 / 排它锁互斥。
3. IX，IS是表级锁，不会和行级的X，S发生冲突，只会和表级的X、S发生冲突。
4. 意向锁在保证并发行的前提下，实现了行锁和表锁共存且满足事务隔离性的要求。

#### 3.行锁的实现方式（Record Lock）

InnoDB行锁是通过给索引上的**`索引项`**（*通常是主键列或者唯一索引*）加锁来实现的，否则加的锁就会变成`Next-Key Lock`。同时查询语句必须为精准匹配（=），不能为 > 、 < 、 like等，否则也会退化成`Next-Key Lock`。这一点MySQL与Oracle不同，候着是通过在数据块中相对应数据行加锁来实现的。InnoDB这种行锁实现特点意味着：只要通过索引条件检索数据，InnoDB才会使用行级锁，否则InnoDB将使用表级锁。

> 实际应用中，需要特别注意InnoDB行锁这一特性，不然的话，可能会导致大量的锁冲突，从而影响性能。

1. 在不通过索引条件查询的时候，InnoDB确实使用的是表锁，而不是行锁。

在如下所示的例子中，开始tab_no_index表没有索引：

```sql
create table tab_no_index(id int,name varchar(10)) engine=innodb;  
## Query OK, 0 rows affected (0.15 sec)  
insert into tab_no_index values(1,'1'),(2,'2'),(3,'3'),(4,'4');  
## Query OK, 4 rows affected (0.00 sec)  
## Records: 4  Duplicates: 0  Warnings: 0  
```

| session_1                                                                                                                                                                                                                                              | session_2                                                                                                                                                                                                                                               |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| mysql> set autocommit=0;<br/>Query OK, 0 rows affected (0.00 sec)<br/>mysql> select * from tab_no_index where id = 1 ;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 1  \| 1  \|<br/>+------+------+<br>1 row in set (0.00 sec) | mysql> set autocommit=0;<br/>Query OK, 0 rows affected (0.00 sec)<br/>mysql> select * from tab_no_index where id = 2 ;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 2  \| 2  \|<br/>+------+------+<br/>1 row in set (0.00 sec) |
| mysql> select * from tab_no_index where id = 1 for update;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 1  \| 1  \|<br/>+------+------+<br/>1 row in set (0.00 sec)                                                            |                                                                                                                                                                                                                                                         |
|                                                                                                                                                                                                                                                        | mysql> select * from tab_no_index where id = 2 for update;<br/>等待                                                                                                                                                                                       |

在如上表所示的例子中，看起来session_1只给一行加了排他锁，但session_2在请求其他行的排他锁时，却出现了锁等待！原因就是在没有索引的情况下，InnoDB只能使用表锁。当我们给其增加一个索引后，InnoDB就只锁定了符合条件的行，如下表所示。

创建tab_with_index表，id字段有普通索引：

```sql
create table tab_with_index(id int,name varchar(10)) engine=innodb;  
## Query OK, 0 rows affected (0.15 sec)  
alter table tab_with_index add index id(id);  
## Query OK, 4 rows affected (0.24 sec)  
## Records: 4  Duplicates: 0  Warnings: 0  
```

 InnoDB存储引擎的表在使用索引时使用行锁例子

| session_1                                                                                                                                                                                                                                                 | session_2                                                                                                                                                                                                                                                 |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| mysql> set autocommit=0;<br/>Query OK, 0 rows affected (0.00 sec)<br/>mysql> select * from tab_with_index where id = 1 ;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 1  \| 1  \|<br/>+------+------+<br/>1 row in set (0.00 sec) | mysql> set autocommit=0;<br/>Query OK, 0 rows affected (0.00 sec)<br/>mysql> select * from tab_with_index where id = 2 ;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 2  \| 2  \|<br/>+------+------+<br/>1 row in set (0.00 sec) |
| mysql> select * from tab_with_index where id = 1 for update;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 1  \| 1  \|<br/>+------+------+<br/>1 row in set (0.00 sec)                                                             |                                                                                                                                                                                                                                                           |
|                                                                                                                                                                                                                                                           | mysql> select * from tab_with_index where id = 2 for update;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 2  \| 2  \|<br/>+------+------+<br/>1 row in set (0.00 sec)                                                             |

2. 由于MySQL的行锁是针对索引加的锁，不是针对记录加的锁，所以虽然是访问不同行的记录，但是如果是使用相同的索引键，是会出现锁冲突的。应用设计的时候要注意这一点。

在如下表所示的例子中，表tab_with_index的id字段有索引，name字段没有索引：

```sql
alter table tab_with_index drop index name;  
## Query OK, 4 rows affected (0.22 sec)  
## Records: 4  Duplicates: 0  Warnings: 0  
insert into tab_with_index  values(1,'4');  
## Query OK, 1 row affected (0.00 sec)  
select * from tab_with_index where id = 1;  
+------+------+  
| id   | name |  
+------+------+  
| 1    | 1    |  
| 1    | 4    |  
+------+------+  
2 rows in set (0.00 sec)  
```

 InnoDB存储引擎使用相同索引键的阻塞例子

| session_1                                                                                                                                                                                                    | session_2                                                                                                                                   |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| mysql> set autocommit=0;<br/>Query OK, 0 rows affected (0.00 sec)                                                                                                                                            | mysql> set autocommit=0;<br/>Query OK, 0 rows affected (0.00 sec)                                                                           |
| mysql> select * from tab_with_index where id = 1 and name = '1' for update;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 1  \| 1  \|<br/>+------+------+<br/>1 row in set (0.00 sec) |                                                                                                                                             |
|                                                                                                                                                                                                              | 虽然session_2访问的是和session_1不同的记录，但是因为使用了相同的索引,所以需要等待锁：<br/>mysql> select * from tab_with_index where id = 1 and name = '4' for update;<br/>等待 |

3. 当表有多个索引的时候，不同的事务可以使用不同的索引锁定不同的行，另外，不论是使用主键索引、唯一索引或普通索引，InnoDB都会使用行锁来对数据加锁。

在如下表所示的例子中，表tab_with_index的id字段有主键索引，name字段有普通索引：

```sql
alter table tab_with_index add index name(name);  
## Query OK, 5 rows affected (0.23 sec)  
## Records: 5  Duplicates: 0  Warnings: 0  
```

 InnoDB存储引擎的表使用不同索引的阻塞例子

| session_1                                                                                                                                                                                                         | session_2                                                                                                                                                                                                                                       |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| mysql> set autocommit=0;<br/>Query OK, 0 rows affected (0.00 sec)                                                                                                                                                 | mysql> set autocommit=0;<br/>Query OK, 0 rows affected (0.00 sec)                                                                                                                                                                               |
| mysql> select * from tab_with_index where id = 1 for update;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 1  \| 1  \|<br/>\| 1  \| 4  \|<br/>+------+------+<br/>2 rows in set (0.00 sec) |                                                                                                                                                                                                                                                 |
|                                                                                                                                                                                                                   | Session_2使用name的索引访问记录，因为记录没有被索引，所以可以获得锁：<br/>mysql> select * from tab_with_index where name = '2' for update;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 2  \| 2  \|<br/>+------+------+<br/>1 row in set (0.00 sec) |
|                                                                                                                                                                                                                   | 由于访问的记录已经被session_1锁定，所以等待获得锁。<br/>mysql> select * from tab_with_index where name = '4' for update;                                                                                                                                             |

4. 即便在条件中使用了索引字段，但是否使用索引来检索数据是由MySQL通过判断不同执行计划的代价来决定的，**如果MySQL认为全表扫描效率更高，比如对一些很小的表，它就不会使用索引，这种情况下InnoDB将使用表锁，而不是行锁**。因此，在分析锁冲突时，别忘了检查SQL的执行计划，以确认是否真正使用了索引。

在下面的例子中，检索值的数据类型与索引字段不同，虽然MySQL能够进行数据类型转换，但却不会使用索引，从而导致InnoDB使用表锁。通过用explain检查两条SQL的执行计划，我们可以清楚地看到了这一点。

例子中tab_with_index表的name字段有索引，但是name字段是varchar类型的，如果where条件中不是和varchar类型进行比较，则会对name进行类型转换，而执行的全表扫描。

```sql
alter table tab_no_index add index name(name);  
## Query OK, 4 rows affected (8.06 sec)  
## Records: 4  Duplicates: 0  Warnings: 0  
explain select * from tab_with_index where name = 1 \G  
*************************** 1. row ***************************  
           id: 1  
  select_type: SIMPLE  
        table: tab_with_index  
         type: ALL  
possible_keys: name  
          key: NULL  
      key_len: NULL  
          ref: NULL  
         rows: 4  
        Extra: Using where  
1 row in set (0.00 sec)  
mysql> explain select * from tab_with_index where name = '1' \G  
*************************** 1. row ***************************  
           id: 1  
  select_type: SIMPLE  
        table: tab_with_index  
         type: ref  
possible_keys: name  
          key: name  
      key_len: 23  
          ref: const  
         rows: 1  
        Extra: Using where  
1 row in set (0.00 sec)  
```

#### 4. 间隙锁(Gap Lock)与Next -Key锁

**间隙锁**基于`非唯一索引`，它`锁定一段范围内的索引记录`。**间隙锁**基于下面将会提到的`Next-Key Locking` 算法，请务必牢记：**使用间隙锁锁住的是一个区间，而不仅仅是这个区间中的每一条数据**。

```sql
SELECT * FROM table WHERE id BETWEN 1 AND 10 FOR UPDATE;
```

即所有在`（1，10）`区间内的记录行都会被锁住，所有id 为 2、3、4、5、6、7、8、9 的数据行的插入会被阻塞，但是 1 和 10 两条记录行并不会被锁住。
除了手动加锁外，在执行完某些 SQL 后，InnoDB 也会自动加**间隙锁**，这个我们在下面会提到。

**临键锁（Next-Key Locks）**
Next-Key 可以理解为一种特殊的**间隙锁**，也可以理解为一种特殊的**算法**。通过**临建锁**可以解决`幻读`的问题。 每个数据行上的`非唯一索引列`上都会存在一把**临键锁**，当某个事务持有该数据行的**临键锁**时，会锁住一段**左开右闭区间**的数据。需要强调的一点是，`InnoDB` 中`行级锁`是基于索引实现的，**临键锁**只与`非唯一索引列`有关，在`唯一索引列`（包括`主键列`）上不存在**临键锁**。

假设有如下表：
**MySql**，**InnoDB**，**Repeatable-Read**：table(id PK, age KEY, name)

| id  | age | name  |
| --- | --- | ----- |
| 1   | 10  | Lee   |
| 3   | 24  | sower |
| 5   | 32  | cp    |
| 7   | 45  | fansw |

该表中 `age` 列潜在的`临键锁`有：
(-∞, 10],
(10, 24],
(24, 32],
(32, 45],
(45, +∞],
在`事务 A` 中执行如下命令：

```sql
-- 根据非唯一索引列 UPDATE 某条记录 
UPDATE table SET name = Vladimir WHERE age = 24; 
-- 或根据非唯一索引列 锁住某条记录 
SELECT * FROM table WHERE age = 24 FOR UPDATE; 
```

不管执行了上述 SQL 中的哪一句，之后如果在`事务 B` 中执行以下命令，则该命令会被阻塞：

```sql
INSERT INTO table VALUES(100, 26, 'Ezreal'); 
```

很明显，`事务 A` 在对 `age` 为 24 的列进行 UPDATE 操作的同时，也获取了 `(24, 32]` 这个区间内的临键锁。
不仅如此，在执行以下 SQL 时，也会陷入阻塞等待：

```sql
INSERT INTO table VALUES(100, 30, 'Ezreal'); 
```

那最终我们就可以得知，在根据`非唯一索引` 对记录行进行 `UPDATE \ FOR UPDATE \ LOCK IN SHARE MODE` 操作时，InnoDB 会获取该记录行的 `临键锁` ，并同时获取该记录行下一个区间的`间隙锁`。
即`事务 A`在执行了上述的 SQL 后，最终被锁住的记录区间为 `(10, 32)`。
**总结:**

1. **InnoDB** 中的`行锁`的实现依赖于`索引`，一旦某个加锁操作没有使用到索引，那么该锁就会退化为`表锁`。
2. **记录锁**存在于包括`主键索引`在内的`唯一索引`中，锁定单条索引记录。
3. **间隙锁**存在于`非唯一索引`中，锁定`开区间`范围内的一段间隔，它是基于**临键锁**实现的。
4. **临键锁**存在于`非唯一索引`中，该类型的每条记录的索引上都存在这种锁，它是一种特殊的**间隙锁**，锁定一段`左开右闭`的索引区间。

------

另外一个网站的举例：

举例来说，假如emp表中只有101条记录，其empid的值分别是 1,2,...,100,101，下面的SQL：

```sql
Select * from  emp where empid > 100 for update;
```

是一个范围条件的检索，InnoDB不仅会对符合条件的empid值为101的记录加锁，也会对empid大于101（这些记录并不存在）的“间隙”加锁。

InnoDB使用间隙锁的目的，一方面是为了防止幻读，以满足相关隔离级别的要求，对于上面的例子，要是不使用间隙锁，如果其它事务插入了empid大于100的任何记录，那么本事务如果再次执行上述语句，就会发生幻读；另外一方面，是为了满足其恢复和复制的需要。

很显然，在使用范围条件检索并锁定记录时，InnoDB这种加锁机制会阻塞符合条件范围内键值的并发插入，这往往会造成严重的锁等待。因此，在实际应用开发中，尤其是并发插入比较多的应用，我们要尽量优化业务逻辑，尽量使用相等条件来访问更新数据，避免使用范围条件。

还要特别说明的是，InnoDB除了通过范围条件加锁时使用间隙锁外，如果使用相等条件请求给一个不存在的记录加锁，InnoDB也会使用间隙锁！

在如下表所示的例子中，假如emp表中只有101条记录，其empid的值分别是1,2,......,100,101。

​                                  *InnoDB存储引擎的间隙锁阻塞例子*

| session_1 |session_2 |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| mysql> select @@tx_isolation;<br/>+-----------------+<br/>\| @@tx_isolation \|<br/>+-----------------+<br/>\| REPEATABLE-READ \|<br/>+-----------------+<br/>1 row in set (0.00 sec)<br/>mysql> set autocommit = 0;<br/>Query OK, 0 rows affected (0.00 sec) | mysql> select @@tx_isolation;<br/>+-----------------+<br/>\| @@tx_isolation \|<br/>+-----------------+<br/>\| REPEATABLE-READ \|<br/>+-----------------+<br/>1 row in set (0.00 sec)<br/>mysql> set autocommit = 0;<br/>Query OK, 0 rows affected (0.00 sec) |
| 当前session对不存在的记录加for update的锁：<br/>mysql> select * from emp where empid = 102 for update;<br/>Empty set (0.00 sec)                                                                                                                                           |                                                                                                                                                                                                                                                              |
|                                                                                                                                                                                                                                                              | 这时，如果其他session插入empid为102的记录（注意：这条记录并不存在），也会出现锁等待：<br/>mysql>insert into emp(empid,...) values(102,...);<br/>阻塞等待                                                                                                                                            |
| Session_1 执行rollback：mysql> rollback;<br/>Query OK, 0 rows affected (13.04 sec)                                                                                                                                                                              |                                                                                                                                                                                                                                                              |
|                                                                                                                                                                                                                                                              | 由于其他session_1回退后释放了Next-Key锁，当前session可以获得锁并成功插入记录：<br/>mysql>insert into emp(empid,...) values(102,...);<br/>Query OK, 1 row affected (13.35 sec)                                                                                                           |

#### 5.插入意向锁

> 前言：
>
> 在讲解之前，先来思考一个问题——假设有用户表结构如下：
> **MySql**，**InnoDB**，**Repeatable-Read**：users(id PK, name, age KEY)

| id  | name | age |
| --- | ---- | --- |
| 1   | Mike | 10  |
| 2   | Jone | 20  |
| 3   | Tony | 30  |

首先事务A插入了一行数据，并且没有Commit

```sql
INSERT INTO users SELECT 4, 'Bill', 15;
```

随后事务B试图插入一行数据

```sql
INSERT INTO users SELECT 5, 'Louis', 16;
```

请问：

1. 使用了什么锁？
2. 事务B是否会被事务A阻塞？

插入意向锁（Insert Intention Locks）

`插入意向锁`是在插入一条记录行前，由 **INSERT** 操作产生的一种`间隙锁`。该锁用以表示插入**意向**，当多个事务在**同一区间**（gap）插入**位置不同**的多条数据时，事务之间**不需要互相等待**。假设存在两条值分别为 4 和 7 的记录，两个不同的事务分别试图插入值为 5 和 6 的两条记录，每个事务在获取插入行上独占的（排他）锁前，都会获取（4，7）之间的`间隙锁`，但是因为数据行之间并不冲突，所以两个事务之间并**不会产生冲突**（阻塞等待）。

总结来说，`插入意向锁`的特性可以分成两部分：

1. `插入意向锁`是一种特殊的`间隙锁` —— `间隙锁`可以锁定**开区间**内的部分记录。
2. `插入意向锁`之间互不排斥，所以即使多个事务在同一区间插入多条记录，只要记录本身（`主键`、`唯一索引`）不冲突，那么事务之间就不会出现**冲突等待**。

需要强调的是，虽然`插入意向锁`中含有`意向锁`三个字，但是它并不属于`意向锁`而属于`间隙锁`，因为`意向锁`是**表锁**而`插入意向锁`是**行锁**。
现在我们可以回答开头的问题了：

1. 使用`插入意向锁`与`记录锁`。
2. `事务 A` 不会阻塞`事务 B`。

为什么不用间隙锁
如果只是使用普通的`间隙锁`会怎么样呢？还是使用我们文章开头的数据表为例：

首先`事务 A` 插入了一行数据，并且没有 `commit`：

```sql
INSERT INTO users SELECT 4, 'Bill', 15; 
```

此时 `users` 表中存在**三把锁**：

1. id 为 4 的记录行的`记录锁`。
2. age 区间在（10，15）的`间隙锁`。
3. age 区间在（15，20）的`间隙锁`。

最终，`事务 A` 插入了该行数据，并锁住了（10，20）这个区间。
随后`事务 B` 试图插入一行数据：

```sql
INSERT INTO users SELECT 5, 'Louis', 16; 
```

因为 16 位于（15，20）区间内，而该区间内又存在一把`间隙锁`，所以`事务 B` 别说想申请自己的`间隙锁`了，它甚至不能获取该行的`记录锁`，自然只能乖乖的等待 `事务 A` 结束，才能执行插入操作。
很明显，这样做事务之间将会频发陷入**阻塞等待**，**插入的并发性**非常之差。这时如果我们再去回想我们刚刚讲过的`插入意向锁`，就不难发现它是如何优雅的解决了**并发插入**的问题。

**总结：**

1. **MySql InnoDB** 在 `Repeatable-Read` 的事务隔离级别下，使用`插入意向锁`来控制和解决并发插入。
2. `插入意向锁`是一种特殊的`间隙锁`。
3. `插入意向锁`在锁定区间相同但记录行本身不冲突的情况下互不排斥。

## 数据库查询分类

### 多表查询

* 内连接

> 用比较运算符根据每个表共有的列的值匹配两个表中的行(=或>,<)

```sql
select
        d.Good_ID ,
        d.Classify_ID,
        d.Good_Name
        from
        Commodity_list d
        inner join commodity_classification c
        on d.Classify_Description=c.Good_kinds_Name
```

得到的满足某一条件的是A，B内部的数据；正因为得到的是内部共有数据，所以连接方式称为内连接

![](../../../public/assets/images/mysql/inner_join.png)

* 外连接之左连接

```sql
select
        *
        from
        commodity_classification c
        left join commodity_list d
        on d.Classify_Description=c.Good_kinds_Name
```

![](../../../public/assets/images/mysql/left_join.png)

首先是left_join左表的数据全部罗列，然后有满足条件的右表数据都会全部罗列出来，不满足条件的右表记录以NULL值显示。

**左连接升级**：left join 或者 left outer join(等同于left join) + where B.column is null

```sql
select
        *
        from
        commodity_classification c
        left join commodity_list d
        on d.Classify_Description=c.Good_kinds_Name
        where d.Classify_Description is null
```

![](../../../public/assets/images/mysql/left_join_exclusive_null.png)

* 外连接之右连接

```sql
select
        *
        from
        commodity_classification c
        right join commodity_list d
        on d.Classify_Description=c.Good_kinds_Name
```

![](../../../public/assets/images/mysql/right_join.png)

与左连接相反，首先是右表数据全部罗列，然后又满足条件的左表数据都会全部罗列出来，不满足条件的左表记录以NULL值显示。

**右连接升级**：right join 或者 right outer join(等同于right join) + where A.column is null

![](../../../public/assets/images/mysql/right_join_exclusive_null.png)

* 外连接之全外连接

full join ，mysql不支持，但是可以用left join union right join代替

```sql
select
        *
        from
        commodity_classification c
        left join commodity_list d
        on d.Classify_Description=c.Good_kinds_Name
        union
select
        *
        from
        commodity_classification c
        right join commodity_list d
        on d.Classify_Description=c.Good_kinds_Name
```

这种场景下得到的是满足某一条件的公共记录，和独有记录，union是合并的意思，所以这种方式的查询结果就是A、B的全集

![](../../../public/assets/images/mysql/full_join.png)

**全外连接升级**：

```sql
select
        *
        from
        commodity_classification c
        left join commodity_list d
        on d.Classify_Description=c.Good_kinds_Name
        where d.Classify_Description is null
        union
select
        *
        from
        commodity_classification c
        right join commodity_list d
        on d.Classify_Description=c.Good_kinds_Name
         where c.Good_kinds_Name is null
```

这种场景下得到的是A、B中不满足某一条件记录之和

![](../../../public/assets/images/mysql/full_join_exclusive_null.png)

* 交叉连接

交叉连接返回左表中的所有行，左表中的每一行与右表中的所有行组合。交叉连接也称作笛卡尔积。

```sql
## 第一种方式，不指定where子句，得到的结果就是两表相乘
select
        *
        from
        commodity_classification c
       cross join commodity_list d
## 指定where子句，会先生成笛卡尔积，再从里面挑选符合where条件的记录
select
        *
        from
        commodity_classification c
       cross join commodity_list d 
       where c.Good_kinds_Name=d.Classify_Description

## 第二种方式，不显式指定cross join，同样得到的也是笛卡尔积
select
        *
        from
        commodity_classification c,
        commodity_list d 
       where c.Good_kinds_Name=d.Classify_Description
```

* union与union all

```sql
/*
UNION 用于合并两个或多个 SELECT 语句的结果集，并消去表中任何重复行。
UNION 内部的 SELECT 语句必须拥有相同数量的列，列也必须拥有相似的数据类型。
同时，每条 SELECT 语句中的列的顺序必须相同.
*/
SELECT column_name FROM table1
UNION
SELECT column_name FROM table2
/*
默认地，UNION 操作符选取不同的值。如果允许重复的值，请使用 UNION ALL。
当 ALL 随 UNION 一起使用时（即 UNION ALL），不消除重复行
*/
SELECT column_name FROM table1
UNION ALL
SELECT column_name FROM table2
```

## 《MySQL技术内幕 InnoDB存储引擎》第2版

### 第2章 InnoDB存储引擎

#### 1. InnoDB体系架构

![](../../../public/assets/images/mysql/Innodb存储引擎体系架构.png)

从图中可见，InnoDB存储引擎有多个内存块，可以认为这些内存块组成了一个大的内存池，负责如下工作：

* 维护所有 进程 / 线程 需要访问的多个内部数据结构。
* 缓存磁盘上的数据，方便快速地读取，同时在对磁盘文件的数据修改之前在这里缓存。
* 重做日志(redo log)缓冲。
* ……

##### ***后台线程***

InnoDB存储引擎是多线程的模型，因此其后台有多个不同的后台线程，负责处理不同的任务。

1. **Master Thread**

   Master Thread是一个非常核心的后台线程，主要负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、合并插入缓冲(INSERT BUFFER)、UNDO页的回收等。

2. **IO Thread**

   在InnoDB存储引擎中大量使用了AIO(Async IO)来处理写IO请求，这样可以极大提高数据库的性能。而IO Thread的工作主要是负责这些IO请求的回调(call back)处理。通过innodb_read_io_threads和innodb_write_io_threads参数进行设置。通过命令show engine innodb status来观察InnoDB中的IO Thread。

3. **Purge Thread**

   事务被提交后，其所使用的undo log可能不再需要，因此需要Purge Thread来回收已经使用并分配的undo页。可以在MySQL数据库的配置文件中添加如下命令来启用独立的Purge Thread：

   ```ini
   [mysqld]
   innodb_purge_threads=1
   ```

   在InnoDB 1.1版本中，即使将Innodb_purge_threads设为大于1，InnoDB存储引擎启动时也会将其设为1，并在错误文件中出现如下类似提示：

   ```log
   120529 22:54:16 [Warning] option 'innodb-purge-threads': unsigned value 4 adjusted to 1
   ```

   从InnoDB 1.2版本开始，InnoDB支持多个Purge Thread，这样做的目的是为了进一步加快undo 页的回收。同时由于Purge Thread需要离散地读取undo 页，这样也能更进一步利用磁盘的随机读取性能。

4. **Page Cleaner Thread**

   Page Cleaner Thread是在InnoDB 1.2.X版本引入的。其作用是将之前版本中脏页的刷新操作都放入到单独的线程中来完成。而其目的是为了减轻原Master Thread的工作及对于用户查询线程的阻塞，进一步提高InnoDB存储引擎的性能。

#### ***内存***

1. **缓冲池**

   InnoDB存储引擎是基于磁盘处处的，并将其中的记录按照页的方式进行管理。因此可将其视为基于磁盘的数据库系统(Disk-base Database)。在数据库系统中，由于CPU速度与磁盘速度之间的鸿沟，基于磁盘的数据库系统通常使用缓冲池技术来提高数据库的整体性能。

   大致原理如下：在数据库中进行读取页的操作，首先将从磁盘读到的页放在缓冲池中，这个过程称为将页“FIX”在缓冲池中。下一次读取相同页时，首先判断是否在缓冲池中。若在，称该页在缓冲池中被命中，直接读取该页。否则，读取磁盘上的页。对于数据库的修改操作，则首先修改在缓冲池中的页，然后再以一定的频率刷新到磁盘上。这里需要注意的是，页从缓冲池刷新回磁盘的操作并不是在每次页发生更新时触发，而是通过一种称为Checkpoint的机制刷新回磁盘。同样，这也是为了提高数据库的整体性能。

   对于InnoDB存储引擎而言，其缓冲池的配置通过参数innodb_buffer_pool_size来设置。

   具体来看，缓冲池中缓存的数据页类型有：索引页、数据页、undo 页、插入缓冲(insert buffer)、自适应哈希索引(adaptive hash index)、InnoDB存储的锁信息(lock info)、数据字典信息(data dictionary)等。不能简单的认为，缓冲池只是缓存索引页和数据页，它们只是占缓冲池很大的一部分而已。

   ![](../../../public/assets/images/mysql/InnoDB内存数据对象.png)

   从InnoDB 1.0.X版本开始，允许有多个缓冲池实例。每个页根据哈希值平均分配到不同的缓冲池实例中。这样做的好处是减少数据库内部的资源竞争，增加数据库的并发处理能力。可以通过参数innodb_buffer_pool_instances来进行配置，该值默认为1。在配置文件中奖innodb_buffer_pool_instances设置为大于1的值就可以得到多个缓冲池实例。再通过命令 show engine innodb status 可以观察到变化。

   从MySQL5.6版本开始，还可以通过information_schema架构下的表innodb_buffer_pool_stats来观察缓冲池的状态。

2. **LRU List、Free List 和 Flush List**

   通常来说，数据库中的缓冲池是通过LRU(Latest Recent Used，最近最少使用)算法来进行管理的。即最频繁使用的页在LRU列表的前端，而最少使用的页在LRU列表的尾端。当缓冲池不能存放新读取到的页时，将首先释放LRU列表中尾端的页。

   在InnoDB存储引擎中，缓冲池中页的大小默认为16KB，同样使用LRU算法对缓冲池进行管理。稍有不同的是InnoDB存储引擎对传统的LRU算法做了一些优化。在InnoDB存储引擎中，LRU列表中还加入了midpoint位置。新读取到的页，虽然是最新访问的页，但并不是直接放入到LRU列表的首部，而是放入LRU列表的midpoint位置。这个算法在InnoDB存储引擎下称为midpoint insertion strategy。在默认配置下，该位置在LRU列表长度的5/8处。midpoint位置可由参数innodb_old_blocks_pct控制。

   ```mysql
   mysql>show variables like 'innodb_old_blocks_pct'\G
   ************************** 1. row **************************
   Variable_name: innodb_old_blocks_pct
           Value: 37
   1 row in set (0.00 sec)
   ```

   从上面的例子可以看到，参数innodb_old_blocks_pct默认值为37，表示新读取的页插入到LRU列表尾端的37%的位置(差不多3/8的位置)。在InnoDB存储引擎中，把midpoint之后的列表称为old列表，之前的列表称为new列表。可以简单的理解为new列表中的页都是最为活跃的热点数据。

   这样做的原因是，因为直接将读取到的页放入到LRU的首部，那么某些SQL操作可能会是缓冲池中的页被刷新出，从而影响缓冲池的效率。常见的这类操作为索引或数据的扫描操作。这类操作需要访问表中的许多页，甚至是全部的页，而这些页通常来说又仅在这次查询操作中需要，并不是活跃的热点数据。如果页被放入LRU列表的首部，那么非常可能将所需要的热点数据页从LRU列表中移除，而在下一次需要读取该页时，InnoDB存储引擎需要再次访问磁盘。为了解决这个问题，InnoDB存储引擎引入了另一个参数来以进一步管理LRU列表，这个参数是innodb_old_blocks_time，用于表示页读取到mid位置后需要等待多久才会被加入到LRU列表的热端。因此当需要执行上述所说的SQL操作时，可以通过下面的方法尽可能是LRU列表中的热点数据不被刷出。

   ```mysql
   mysql> set global innodb_old_blocks_time=1000;
   Query OK, 0 rows affected (0.00 sec)
   
   ## data or index scan operation
   ## ...
   mysql> set global innodb_old_blocks_time=0;
   Query OK, 0 rows affected (0.00 sec)
   ```

   如果用户预估自己活跃的热点数据不止63%，那么在执行SQL语句前，还可以通过下面的语句来减少热点页不可能被刷出的概率。

   ```mysql
   mysql> set global innodb_old_blocks_pct=20;
   Query OK, 0 rows affected (0.00 sec)
   ```

   LRU列表用来管理已经读取的页，但当数据库刚启动时，LRU列表是空的，即没有任何页。这时页都存放在Free列表中。当需要从缓冲池中分页时，首先从Free列表中查找是否有可用的空闲页，若有则将该页从Free列表中删除，放入到LRU列表中。否则，根据LRU算法，淘汰LRU列表末尾的页，将该内存空间分配给新的页。当页从LRU列表的old部分加入到new部分时，称此时发生的操作为 **page made young**，而因为innodb_old_blocks_time的设置而导致页没有从old部分移动到new部分的操作称为 **page not made young**。可以通过命令 show engine innodb status来观察LRU列表及Free列表的使用情况和运行状态。

   ```mysql
   mysql> show engine innodb status \G
   =====================================
   2021-10-09 16:05:00 0x7feb9bb47700 INNODB MONITOR OUTPUT
   =====================================
   Per second averages calculated from the last 5 seconds
   -----------------
   BACKGROUND THREAD
   -----------------
   srv_master_thread loops: 1478477 srv_active, 0 srv_shutdown, 5524787 srv_idle
   srv_master_thread log flush and writes: 0
   ----------
   SEMAPHORES
   ----------
   OS WAIT ARRAY INFO: reservation count 4
   OS WAIT ARRAY INFO: signal count 4
   RW-shared spins 0, rounds 0, OS waits 0
   RW-excl spins 1, rounds 6, OS waits 0
   RW-sx spins 0, rounds 0, OS waits 0
   Spin rounds per wait: 0.00 RW-shared, 6.00 RW-excl, 0.00 RW-sx
   ------------
   TRANSACTIONS
   ------------
   Trx id counter 15483
   Purge done for trx's n:o < 15483 undo n:o < 0 state: running but idle
   History list length 0
   LIST OF TRANSACTIONS FOR EACH SESSION:
   ---TRANSACTION 422126106451512, not started
   0 lock struct(s), heap size 1136, 0 row lock(s)
   ---TRANSACTION 422126106450568, not started
   0 lock struct(s), heap size 1136, 0 row lock(s)
   ---TRANSACTION 422126106449624, not started
   0 lock struct(s), heap size 1136, 0 row lock(s)
   ---TRANSACTION 422126106447736, not started
   0 lock struct(s), heap size 1136, 0 row lock(s)
   ---TRANSACTION 422126106448680, not started
   0 lock struct(s), heap size 1136, 0 row lock(s)
   ---TRANSACTION 422126106446792, not started
   0 lock struct(s), heap size 1136, 0 row lock(s)
   --------
   FILE I/O
   --------
   I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
   I/O thread 1 state: waiting for completed aio requests (log thread)
   I/O thread 2 state: waiting for completed aio requests (read thread)
   I/O thread 3 state: waiting for completed aio requests (read thread)
   I/O thread 4 state: waiting for completed aio requests (read thread)
   I/O thread 5 state: waiting for completed aio requests (read thread)
   I/O thread 6 state: waiting for completed aio requests (read thread)
   I/O thread 7 state: waiting for completed aio requests (read thread)
   I/O thread 8 state: waiting for completed aio requests (read thread)
   I/O thread 9 state: waiting for completed aio requests (read thread)
   I/O thread 10 state: waiting for completed aio requests (read thread)
   I/O thread 11 state: waiting for completed aio requests (read thread)
   I/O thread 12 state: waiting for completed aio requests (read thread)
   I/O thread 13 state: waiting for completed aio requests (read thread)
   I/O thread 14 state: waiting for completed aio requests (write thread)
   I/O thread 15 state: waiting for completed aio requests (write thread)
   I/O thread 16 state: waiting for completed aio requests (write thread)
   I/O thread 17 state: waiting for completed aio requests (write thread)
   I/O thread 18 state: waiting for completed aio requests (write thread)
   I/O thread 19 state: waiting for completed aio requests (write thread)
   I/O thread 20 state: waiting for completed aio requests (write thread)
   I/O thread 21 state: waiting for completed aio requests (write thread)
   I/O thread 22 state: waiting for completed aio requests (write thread)
   I/O thread 23 state: waiting for completed aio requests (write thread)
   I/O thread 24 state: waiting for completed aio requests (write thread)
   I/O thread 25 state: waiting for completed aio requests (write thread)
   Pending normal aio reads: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0] , aio writes: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0] ,
    ibuf aio reads:, log i/o's:, sync i/o's:
   Pending flushes (fsync) log: 0; buffer pool: 1
   874 OS file reads, 4458742 OS file writes, 100385 OS fsyncs
   0.00 reads/s, 0 avg bytes/read, 0.60 writes/s, 0.00 fsyncs/s
   -------------------------------------
   INSERT BUFFER AND ADAPTIVE HASH INDEX
   -------------------------------------
   Ibuf: size 1, free list len 0, seg size 2, 0 merges
   merged operations:
    insert 0, delete mark 0, delete 0
   discarded operations:
    insert 0, delete mark 0, delete 0
   Hash table size 138401, node heap has 0 buffer(s)
   Hash table size 138401, node heap has 0 buffer(s)
   Hash table size 138401, node heap has 0 buffer(s)
   Hash table size 138401, node heap has 0 buffer(s)
   Hash table size 138401, node heap has 1 buffer(s)
   Hash table size 138401, node heap has 1 buffer(s)
   Hash table size 138401, node heap has 2 buffer(s)
   Hash table size 138401, node heap has 4 buffer(s)
   0.00 hash searches/s, 0.00 non-hash searches/s
   ---
   LOG
   ---
   Log sequence number          18182045
   Log buffer assigned up to    18182045
   Log buffer completed up to   18182045
   Log written up to            18182045
   Log flushed up to            18182045
   Added dirty pages up to      18182045
   Pages flushed up to          18182045
   Last checkpoint at           18182045
   227 log i/o's done, 0.00 log i/o's/second
   ----------------------
   BUFFER POOL AND MEMORY
   ----------------------
   Total large memory allocated 549715968
   Dictionary memory allocated 463343
   Buffer pool size   32768 ## 有32768个页，即32768*16K，共512MB的缓冲池
   Free buffers       31720 ## 表示当前Free列表中页的数量
   Database pages     1040 ## 表示LRU列表中页的数量
   Old database pages 364
   Modified db pages  0
   Pending reads      0
   Pending writes: LRU 0, flush list 0, single page 0
   Pages made young 6049711, not young 12706
   0.00 youngs/s, 0.00 non-youngs/s
   Pages read 851, created 300576, written 4358280
   0.00 reads/s, 0.00 creates/s, 0.00 writes/s
   Buffer pool hit rate 1000 / 1000, young-making rate 71 / 1000 not 0 / 1000
   Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
   LRU len: 1040, unzip_LRU len: 0
   I/O sum[37]:cur[0], unzip sum[0]:cur[0]
   --------------
   ROW OPERATIONS
   --------------
   0 queries inside InnoDB, 0 queries in queue
   0 read views open inside InnoDB
   Process ID=543626, Main thread ID=140649955288832 , state=sleeping
   Number of rows inserted 67955720, updated 0, deleted 0, read 68608883
   9.60 inserts/s, 0.00 updates/s, 0.00 deletes/s, 9.60 reads/s
   Number of system rows inserted 47, updated 335, deleted 2, read 301015
   0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
   ----------------------------
   END OF INNODB MONITOR OUTPUT
   ============================
   ```

   可能的情况是Free buffers与Database pages的数量之和不等于Buffer pool size。因为缓冲池中的页还可能会被分配给自适应哈希索引、Lock信息、Insert Buffer等页，而这部分页不需要LRU算法进行维护，因此不存在与LRU列表中。

   **pages made young** 显示了LRU列表中页移动到前端的次数，因为设置了innodb_old_blocks_time的值，所以 **not young** 当前值为 12706。youngs/s、non-youngs/s表示每秒这两类操作的次数。这里还有一个重要的观察变量——Buffer pool hit rate，表示缓冲池的命中率。通常该值不应该小于95%，若发生Buffer pool hit rate小于95%这种情况，用户需要观察是否由于全表扫描引起的LRU列表被污染的问题。

   > Note：执行命令show engine innodb status显示的不是当前的状态，而是过去某个时间范围内InnoDB存储引擎的状态。上面截取的信息可以看出，Per second averages calculated from the last 5 seconds，代表的信息为过去5秒内的数据库状态。

   从InnoDB 1.2版本开始，还可以通过表INNODB_BUFFER_POOL_STATS来观察缓冲池的运行状态

   ```mysql
   mysql> select POOL_ID, HIT_RATE,
   -> PAGES_MADE_YOUNG, PAGES_NOT_MADE_YOUNG
   -> FROM information_schema.INNODB_BUFFER_POOL_STATS\G
   **************************** 1. row ****************************
                POOL_ID: 0
                  HIT_RATE: 980
       PAGES_MADE_YOUNG: 450
   PAGES_NOT_MADE_YOUNG: 0
   ```

   此外，还可以通过表INNODB_BUFFER_PAGE_LRU来观察每个LRU列表中每个页的具体信息，例如通过下面的语句可以看到缓冲池LRU列表中SPACE为1的表的页类型：

   ```mysql
   mysql> select TABLE_NAME,SPACE,PAGE_NUMBER,PAGE_TYPE
       -> FROM INNODB_BUFFER_PAGE_LRU where SPACE = 1;
   +-------------------+-------+-------------+--------------------------+
   | TABLE_NAME        | SPACE | PAGE_NUMBER | PAGE_TYPE                |
   +-------------------+-------+-------------+--------------------------+
   | NULL              |     1 |           0 | FILE_SPACE_HEADER        |
   | NULL              |     1 |           1 | IBUF_BITMAP              |
   | NULL              |     1 |           2 | INODE                    |
   | test/t            |     1 |           3 | INDEX                    |
   +-------------------+-------+-------------+--------------------------+
   ```

   InnoDB存储引擎从1.0.X版本开始支持压缩页的功能，即将原本16KB的页压缩为1KB、2KB、4KB、8KB。由于页的大小发生了变化，LRU列表也有了些许改变。对于非16KB的页，是通过unzip_LRU列表进行管理的。通过命令show engine innodb status可以观察到如下内容：

   ```mysql
   Buffer pool hit rate 1000 / 1000, young-making rate 71 / 1000 not 0 / 1000
   Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
   LRU len: 1539, unzip_LRU len: 156
   I/O sum[37]:cur[0], unzip sum[0]:cur[0]
   ```

   可以看到LRU列表中一共有1539个页，而unzip_LRU列表中有156个页。这里需要注意的是，LRU中的页包含了unzip_LRU列表中的页。

   对于压缩页的表，每个表的压缩比率可能各不相同。可能存在有的表页大小为8KB，有的表页大小为2KB的情况。

   **unzip_LRU是怎样从缓冲池中分配内存的呢？**

   首先，在unzip_LRU列表中对不同压缩页大小页进行分别管理。其次，通过伙伴算法进行内存分配。例如对需要从缓冲池汇总申请页为4KB的大小，其过程如下：

   1. 检查4KB的unzip_LRU列表，检查是否有可用的空闲页；
   2. 若有，则直接使用；
   3. 否则，检查8KB的unzip_LRU列表；
   4. 若能够得到空闲页，将页分成2个4KB页，存放到4KB的unzip_LRU列表；
   5. 若不能得到空闲页，从LRU列表中申请一个16KB的页，将页分为1个8KB的页，2个4KB的页，分别存放到对应的unzip_LRU列表中。

   同样可以通过information_schema.INNODB_BUFFER_PAGE_LRU来观察unzip_LRU列表中的页，如：

   ```mysql
   mysql> select TABLE_NAME,SPACE,PAGE_NUMBER,COMPRESSED_SIZE
       -> from INNODB_BUFFER_PAGE_LRU
       -> where COMPRESSED_SIZE <> 0;
   +-------------------+-------+-------------+----------------------+
   | TABLE_NAME        | SPACE | PAGE_NUMBER | COMPRESSED_SIZE      |
   +-------------------+-------+-------------+----------------------+
   | sbtest/t          |     9 |         134 |                 8192 |
   | sbtest/t          |     9 |         135 |                 8192 |
   | sbtest/t          |     9 |          96 |                 8192 |
   | sbtest/t          |     9 |         136 |                 8192 |
   ....
   ```

   在LRU列表中的页被修改后，称该页未脏页(dirty page)，即缓冲池中的页和磁盘上的页的数据产生了不一致。这时数据库会通过checkpoint机制将脏页刷新回磁盘，而Flush列表中的页即为脏页列表。需要注意的是，脏页既存在与LRU列表中，也存在与Flush列表中。LRU列表用来管理缓冲池中页的可用性，Flush列表用来管理将页刷新回磁盘，二者互不影响。

   同LRU列表一样，Flush列表也可以通过show engine innodb status来查看，后面的数字表示了脏页的数量。

   ```mysql
   Modified db pages  0
   ```

   information_schema库下并没有类似INNODB_BUFFER_PAGE_LRU表来显示脏页的数量及脏页的类型，但正如前面描述的那样，脏页同样存在与LRU列表中，故用户可以通过元数据表INNODB_BUFFER_PAGE_LRU来查看，唯一不同的是需要加入OLDEST_MODIFICATION大于0的SQL查询条件，如：

   ```mysql
   mysql> select TABLE_NAME,SPACE,PAGE_NUMBER,PAGE_TYPE
       -> from INNODB_BUFFER_PAGE_LRU
       -> where OLDEST_MODIFICATION > 0;
   +-------------------+-------+-------------+-------------------+
   | TABLE_NAME        | SPACE | PAGE_NUMBER | PAGE_TYPE         |
   +-------------------+-------+-------------+-------------------+
   | NULL              |     0 |          56 | SYSTEM            |
   | NULL              |     0 |           0 | FILE_SPACE_HEADER |
   | test/t            |     1 |           3 | INDEX             |
   | NULL              |     0 |         320 | INODE             |
   | NULL              |     0 |         325 | UNDO_LOG          |
   +-------------------+-------+-------------+-------------------+
   ```

   可以看到当前共有5个脏页及它们对应的表和类型。TABLE_NAME为NULL表示该页属于系统表空间。

3. **重做日志缓冲**

   InnoDB存储引擎的内存区域除了有缓冲池外，还有重做日志缓冲(redo log buffer)。InnoDB存储引擎首先将重做日志信息先放入缓冲区，然后按一定频率将其刷新到重做日志文件。重做日志缓冲一般不需要设置的很大，因为一般情况下每一秒钟会将重做日志缓冲刷新到日志文件，因此用户只需要保证每秒产生的事务量在这个缓冲大小之内即可。该值可由配置参数innodb_log_buffer_size控制，默认为8MB。

   在通常情况下，8MB的重做日志缓冲池足以满足大部分的应用，因为重做日志在下列三种情况下会将重做日志缓冲中的内容刷新到外部磁盘的重做日志文件中。

   * Master Thread每一秒将重做日志缓冲刷新到重做日志文件；
   * 每个事务提交时会将重做日志缓冲刷新到重做日志文件；
   * 当重做日志缓冲池剩余空间小于1/2时，重做日志缓冲刷新到重做日志文件。

4. **额外的内存池**

   额外的内存池通常被DBA忽略，他们认为该值并不十分重要，事实恰恰相反，该值同样重要。在InnoDB存储引擎中，对内存的管理是通过一种称为内存堆(heap)的方式进行的。在对一些数据结构本身的内存进行分配时，需要从额外的内存池中进行申请，当该区域内存不够时，会从缓冲池中进行申请。例如，分配了缓冲池(innodb_buffer_pool)，但是每个缓冲池中的帧缓冲(frame buffer)还有对应的缓冲控制对象(buffer control block)，这些对象记录了一些诸如LRU、锁、等待等信息，而这个对象的内存需要从额外内存池中申请。因此，在申请了很大的InnoDB缓冲池时，也应考虑相应地增加这个值。

#### 2. Checkpoint技术

**背景**：缓冲池解决了CPU与磁盘速度鸿沟的问题，但是每次更新都是先在缓冲池中完成，再刷新到磁盘，假如刷新过程突然宕机，数据就会丢失。

为了避免发生数据丢失问题，当前事务数据库系统普遍采用了**Write Ahead Log**策略，即当事务提交时，先写重做日志(redo log)，在修改页(即缓冲池中对应的数据)。当发生宕机时，通过重做日志来完成数据的恢复。这也是事务ACID中D(Durability持久性)的要求。

**思考**：如果重做日志可以无限地增大，同时缓冲池也足够大，能够缓冲所有数据库的数据，那么是不需要将缓冲池中页的新版本刷新回磁盘。因为发生宕机时，完全可以通过重做日志来恢复，但是这需要2个条件：

* 缓冲池可以缓存数据库中的所有数据；
* 重做日志可以无限增大。

对于第一个条件，刚建好的数据库还可以胜任，但是随着业务的扩展，用户的增加，缓冲池是不可能缓存下所有的数据，这是可以预见的。第二个条件，理论和实际都可以实现，但是成本要求太高，同时不便于运维。因为重做日志什么时候接近磁盘可使用空间的阈值是未知的，且还需要让存储设备支持动态扩展技术。

假如，2个条件都具备，那么还有一个情况需要考虑：宕机后数据库的恢复时间。当数据库运行了几个月甚至几年时，重新应用重做日志的时间会非常久，此时恢复的代价也会非常大。

**解决**：Checkpoint(检查点)技术的目的就是解决以下几个问题。

* 缩短数据库的恢复时间；
* 缓冲池不够用时，将脏页刷新到磁盘；
* 重做日志不可用时，刷新脏页。

当数据库发生宕机时，数据库不需要重做所有的日志，因为Checkpoint之前的页都已经刷新回磁盘。故数据库只需对checkpoint后的重做日志进行恢复。此外，当缓冲池不够用时，根据LRU算法会移除最近最少使用的页，若此页为脏页，那么需要强制执行Checkpoint，将脏页也就是页的新版本刷新回磁盘。重做日志出现不可用的情况是因为当前事务数据库对重做日志的设计都是循环使用的，并不是让其无限增大。重做日志可以被重用的部分都是不再需要的部分，当发生宕机时这部分日志就可以被覆盖重用。若此时重做日志还需要使用，那么必须强制产生Checkpoint，将缓冲池中的页至少刷新到当前重做日志的位置。对于InnoDB存储引擎而言，其是通过LSN(Log Sequence Number)来标记版本的。而LSN是8字节的数字，其单位是字节。每个页有LSN，重做日志也有LSN，Checkpoint也有LSN。可以通过show engine innodb status来观察：

```mysql
---
LOG
---
Log sequence number          18182045
Log buffer assigned up to    18182045
Log buffer completed up to   18182045
Log written up to            18182045
Log flushed up to            18182045
Added dirty pages up to      18182045
Pages flushed up to          18182045
Last checkpoint at           18182045
227 log i/o's done, 0.00 log i/o's/second
```

在InnoDB存储引擎中，Checkpoint发生的时间、条件及脏页的选择等都非常复杂。而Checkpoint所做的事情无外乎是将缓冲池中的脏页刷回到磁盘。不同之处在于每次刷新多少页、每次从哪里取脏页，以及什么时间触发Checkpoint。在InnoDB存储引擎内部，有两种Checkpoint，分别为：

* Sharp Checkpoint
* Fuzzy Checkpoint

Sharp Checkpoint发生在数据库关闭时将所有的脏页都刷新回磁盘，这是默认的工作方式，即参数innodb_fast_shutdown=1。但是若数据库在运行时也使用Sharp Checkpoint，那么数据库的可用性就会受到很大影响。故在InnoDB存储引擎内部使用Fuzzy Checkpoint进行页的刷新，即只刷新一部分脏页，而不是刷新所有的脏页回磁盘。

在InnoDB存储引擎中可能发生如下几种情况的Fuzzy Checkpoint：

* Master Thread Checkpoint
* FLUSH_LRU_LIST Checkpoint
* Async/Sync Flush Checkpoint
* Dirty Page too much Checkpoint

Master Thread 发生的Checkpoint，差不多以每秒或每十秒的速度从缓冲池的脏页列表中的刷新一定比例的页回磁盘。这个过程是异步的，即此时InnoDB存储引擎可以进行其他的操作，用户查询线程不会阻塞。

FLUSH_LRU_LIST Checkpoint是因为InnoDB存储引擎需要保证LRU列表中需要有差不多100个空闲页可以使用。在InnoDB 1.1.X 版本之前，需要检查LRU列表中是否有足够的可用空间操作发生在用户查询线程中，显然这会阻塞用户的查询操作。倘若没有100个可用空闲页，那么InnoDB存储引擎会将LRU列表尾端的页移除。如果这些页中有脏页，那么需要进行Checkpoint，而这些页是来自LRU列表的，因此成为FLUSH_LRU_LIST Checkpoint。从MySQL5.6版本，也就是InnoDB 1.2.X开始，这个检查放在了一个单独的Page Cleaner线程中进行，并且用户可以通过参数innodb_lru_scan_depth控制LRU列表中可用页的数量，该值默认为1024，如：

```mysql
mysql> show variables like 'innodb_lru_scan_depth'\G
**************************** 1. row ****************************
Variable_name: innodb_lru_scan_depth
        Value: 1024
```

Async/Sync Flush Checkpoint指的是重做日志不可用的情况，这时需要强制将一些页刷新回磁盘，而此时脏页是从脏页列表中选择的。若将已经写入到重做日志的LSN记为 redo_lsn，将已经刷新回磁盘最新页的LSN记为checkpoint_lsn，则可定义：

checkpint_age = redo_lsn - checkpoint_lsn

`定义checkpoint存在的时长,redo_lsn是最新一次写入redo log的lsn，checkpoint_lsn是最近一次刷新回磁盘的lsn，两者做差得到一个时间差值，即距离上一次刷新回磁盘经过了多长时间`

再定义以下变量：

async_water_mark = 75% * total_redo_log_file_size

sync_water_mark = 90% * total_redo_log_file_size

若每个重做日志文件的大小为1GB，并且定义了两个重做日志文件，则重做日志文件的总大小为2GB。那么async_water_mark = 1.5GB，sync_water_mark = 1.8GB。则：

* 当checkpoint_age < async_water_mark时，不需要刷新任何脏页到磁盘；
* 当async_water_mark<checkpoint_age<sync_water_mar时，触发Async Flush，从Flush列表中刷新足够的脏页回磁盘，使得刷新后满足checkpoint_age<async_water_mark；
* checkpoint_age > sync_water_mark，这种情况一般很少发生，除非设置的重做日志文件太小，并且在进行类似Load Data的Bulk Insert操作。此时触发Sync Flush操作，从Flush列表中刷新足够的脏页回磁盘，使得刷新后满足checkpoint_age<async_water_mark。

在InnoDB 1.2.X 版本之前，Async Flush Checkpoint会阻塞发现问题的用户的查询线程，而Sync Flush Checkpoint会阻塞所有的用户查询线程，并且等待脏页刷新完成。从InnoDB 1.2.X版本开始，也就是MySQL5.6版本，这部分的操作同样放入到了单独的Page Cleaner Thread中，故不会阻塞用户查询线程。

MySQL官方版本并不能查看刷新页是从Flush列表中还是从LRU列表中进行Checkpoint的，也不知道因为重做日志而产生的Async/Sync Flush次数。但是提供了方法，通过命令show engine innodb status来观察。（MySQL5.7版本没有找到相关数据）

最后一种Checkpoint的情况是Dirty Page too much，脏页太多，导致InnoDB存储引擎强制进行Checkpoint。其目的总得来说还是为了保证缓冲池中有足够可用的页。其由参数innodb_max_dirty_pages_pct控制：

```mysql
mysql> show variables like 'innodb_max_dirty_pages_pct'\G
**************************** 1. row ****************************
Variable_name: innodb_max_dirty_pages_pct
        Value: 75
```

innodb_max_dirty_pages_pct值为75表示，当缓冲池中脏页的数量占据75%时，强制进行Checkpoint，刷新一部分的脏页到磁盘。

#### 3. InnoDB关键特性

InnoDB存储引擎的关键特性包括：

* 插入缓冲（Insert Buffer）
* 两次写（Double Write）
* 自适应哈希索引（Adaptive Hash Index）
* 异步IO（Async IO）
* 刷新邻接页（Flush Neighbor Page）

1. Insert Buffer(插入缓冲)

在InnoDB存储引擎中，主键是行唯一的标识符。通常应用程序中行记录的插入顺序是按照主键递增的顺序进行插入的。因此，插入聚集索引（Primary Key）一般是顺序的，不许需要磁盘的随机读取。例如：表中字段设置了auto_increment属性，值就会自动增长，同时页中的行记录按该字段的值进行顺序存放。一般情况下，不许需要随机读取另一个页中的记录，因此对于这类情况下的插入操作，速度是非常快的。

> Note:
>
> 并不是所有的主键插入都是顺序的。若主键类是UUID这样的类，那么插入和辅助索引一样，同样是随机的。即使主键是自增类型，但是插入的是指定的值，而不是NULL值，那么同样可能导致插入并非连续的情况。

不过并不是每张表上只有一个聚集索引，更多情况下，一张表上有多个非聚集的辅助索引（Secondary index）。比如，用户需要按照b这个字段进行查找，并且b这个字段不是唯一的。

```mysql
create table t (
    a int auto_increament,
    b varcahr(30),
    primary key(a),
    key(b)
)
```

在这样的情况下产生了一个非聚集索引且索引值不唯一。在进行插入操作时，数据页的存放还是按照主键a进行顺序存放，但是对于非聚集索引叶子节点的插入不再是顺序的了，这时就需要离散的访问非聚集索引页，由于随机读取的存在而导致了插入操作性能的下降。当然这并不是b字段上索引的错误，而是因为B+树的特性决定了非聚集索引插入的离散性。

InnoDB存储引擎设计了Insert Buffer，对于非聚集索引的插入或更新操作，不是每一次直接插入到索引页中，而是先判断插入的非聚集索引页是否在缓冲池中，若在，则直接插入；若不在，则先放到一个Insert Buffer对象中。数据库这个非聚集的索引已经插到叶子节点，而实际并没有，只是存放在另一个位置。然后再以一定频率和情况进行Insert Buffer和辅助索引页子节点的merge（合并）操作，这时通常能将多个插入合并到一个操作中（因为在一个索引页中），这就大大提高了对于非聚集索引插入的性能。然而Insert Buffer的使用需要同时满足以下两个条件：

* 索引是辅助索引（secondary index）；
* 索引不是唯一（unique）的。

########## 2. Change Buffer

InnoDB从1.0.X开始引入了Change Buffer，可将其视为Insert Buffer升级。从这个版本开始，InnoDB存储引擎可以对DML操作——Insert、Delete、Update都进行缓冲，他们分别是：Insert Buffer、Delete Buffer、Update Buffer。

和前面的Insert Buffer一样，Change Buffer使用的对象依然是非唯一的辅助索引。对一条记录进行update操作可能分为两个过程：

* 将记录标记为已删除；
* 真正将记录删除。

因此Delete Buffer对应update操作的第一个过程，即将记录标记为删除。Purge Buffer对应update操作的第二个过程，即将记录真正的删除。同时，InnoDB存储引擎提供了参数 innodb_change_buffering，用来开启各种Buffer的选项。该参数的可选值为：inserts、deletes、purges、changes、all、none。changes表示启用inserts和deletes，all表示启用所有，none表示都不启用。该参数默认值为all。

从1.2.X版本开始，可以通过参数 innodb_change_buffer_max_size 来控制Change Buffer最大使用内存的数量。

```mysql
mysql> show VARIABLES like 'innodb_change_buffer_max_size'
**************************** 1. row ****************************
Variable_name: innodb_max_dirty_pages_pct
        Value: 25
```

innodb_change_buffer_max_size默认值为25，表示最多使用1/4的缓冲池内存空间。而需要注意的是，该参数的最大有效值为50。

```mysql
## show engine innodb status
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
```

insert表示Insert Buffer；delete mark 表示Delete Buffer ；delete 表示 Purge Buffer；discarded operations 表示当Change Buffer发生merge时，表示已经删除，此时就无需再将记录合并（merge）到辅助索引中了。

########## 3. Insert Buffer 的内部实现

Insert Buffer的数据结构是一颗B+树。在MySQL4.1之前的版本中每张表有一颗Insert Buffer B+ 树。而在现在的版本中，全局只有一颗Insert Buffer B+树，负责对所有的表的辅助索引进行Insert Buffer。而这颗B+树存放在共享表空间中，默认也就是ibdata1中。因此，试图通过独立表空间ibd文件恢复表中数据时，往往会导致CHECK TABLE 失败。这是因为表的辅助索引中的数据可能还在Insert Buffer中，也就是共享空间中，所以通过ibd文件进行恢复后，还需要进行REPAIR TABLE操作来重建表上所有的辅助索引。

########## 4. Merge Insert Buffer

概括的说，Merge Insert Buffer 的操作可能发生在以下几种情况下：

* 辅助索引页被读取到缓冲池时；
* Insert Buffer Bitmap 页追踪到该辅助索引页已无可用空间时；
* Master Thread。

第一种情况为当辅助索引页被读取到缓冲池中时，例如这在执行正常的SELECT查询操作，这是需要检查Insert Buffer Bitmap页，然后确认该辅助索引页是否有记录存放于Insert Buffer B+ 树中。若有，则将Insert Buffer B+ 树中该页记录插入到该辅助索引页中。可以看到对该页多次的记录操作通过一次操作合并到了原有的辅助索引页中，因此性能会有大幅提高。

Insert Buffer Bitmap 页用来追踪每个辅助索引页的可用空间，并至少有1/32页的空间。若插入辅助索引记录时检测到插入记录后可用空间会小于1/32页，则会强制进行一个合并操作，即强制读取辅助索引页，将Insert Buffer B+数中该页的记录及待插入的记录插入到辅助索引中。这就是第二种情况。

在Master Thread线程中每秒或每10秒会进行一次 Merge Insert Buffer 操作，不同之处在于每次进行merge操作的页的数量不同。在Master Thread中，执行merge操作的不止是一个页，而是根据srv_innodb_io_capacity的百分比来决定真正要合并多少个辅助索引页。但InnoDB存储引擎又是根据怎样的算法来得知需要合并的辅助索引页呢？在Insert Buffer B+树中，辅助索引页根据（space，offset）都已排序好，故可以根据（space，offset）的排序进行页的选择。然而，对于Insert Buffer页的选择，InnoDB存储引擎并非采用这个方式，它随机地选择Insert Buffer B+树中的一个页，读取该页中的space及之后所需要数量的页。该算法在复杂情况下应有更好的公平性。同时，若进行merge时，要进行merge的表已经被删除，此时可以直接丢弃已经被Insert/Change Buffer的数据记录。

2. 两次写

`如果说Insert Buffer带给InnoDB存储引擎是性能上的提升，那么double write带给InnoDB存储引擎的是数据页的可靠性。`

当发生数据库宕机时，可能InnoDB存储引擎正在写入某个页到表中，而这个页只写了一部分，比如16KB的页，只写了前4KB，之后就发生了宕机，这种情况被称为**部分写失效（partial page write）**。

![](../../../public/assets/images/mysql/InnoDB_DoubleWrite架构.png)

**Double Write**由两部分组成，一部分是内存中的doublewrite buffer，大小为2MB，另一部分是物理磁盘上共享表空间中连续的128个页，即2个区（extent），大小同样为2MB。在对缓冲池的脏页进行刷新时，并不直接写入磁盘，而是会通过memory函数将脏页先复制到内存中的doublewrite buffer，之后通过doublewrite buffer再分两次，每次1MB顺序地写入共享表空间的物理磁盘上，然后马上调用fsync函数，同步磁盘，避免缓冲写带来的问题。在这个过程中，因为doublewrite页是连续的，因此这个过程是顺序写的，开销并不是很大。在完成doublewrite页的写入后，再将doublewrite buffer中的页写入哥哥表空间文件中此时的写入则是离散的。可以通过命令show global status like 'innodb_dblwr%'来观察。

```mysql
mysql> show global status like 'innodb_dblwr%';
**************************** 1. row ****************************
Variable_name: innodb_dblwr_pages_written
        Value: 6325194
**************************** 2. row ****************************
Variable_name: innodb_dblwr_writes
        Value: 100399
```

doublewrite一共写了6325194个页，但实际写入次数为100399，基本上符合64:1。如果发现系统在高峰时的Innodb_dblwr_pages_written:Innodb_dblwr_writes远小于64:1，那么可以说明系统写入压力并不是很高。

如果操作系统在将页写入磁盘的过程中发生了崩溃，在恢复过程中，InnoDB存储引擎可以从共享表空间中的doublewrite中找到该页的一个副本，将其复制到表空间文件，再应用重做日志。

若查看MySQL官方手册，会发现在命令show global status中Innodb_buffer_pool_pages_flushed变量表示当前从缓冲池中刷新磁盘页的数量。根据之前的介绍，在默认情况下所有页的刷新首先都需要放入到doublewrite中，因此该变量应该和Innodb_dblwr_pages_written一致。

参数skip_innodb_doublewrite可以禁止使用doublewrite功能，这时可能会发生前面提及的写失效问题。不过如果用户有多个从服务器(slave server)，需要提供较快的性能(如在slave server上做的是RAID 0)，也许启用这个参数是一个办法。不过对于需要提高数据高可靠性的主服务器(master server)，任何时候都应确保开启doublewrite功能。

> Note:
>
> 有些文件系统本身就提供了部分写失效的防范机制，如ZFS文件系统。在这种情况下，用户就不要启用doublewrite了。

######## 3. 自适应哈希索引

`哈希(hash)是一种非常快的查找方法，一般情况下的时间复杂度为O(1)。而B+树的查找次数，取决于B+树的高度，生产环境中，B+树的高度一般为3~4层，故需要3~4次的查询。`

InnoDB存储引擎会监控对表上各索引页的查询。如果观察到建立哈希索引可以带来速度提升，则建立哈希索引，称之为自适应哈希索引（Adaptive Hash Index，AHI）。AHI是通过缓冲池的B+树页构造而来，因此建立的速度很快，而且不需要对整张表构建哈希索引。InnoDB存储引擎会自动根据访问频率和模式来自动地为某些热点页建立哈希索引。

AHI有一个要求，即对这个页的连续访问模式必须是一样的。例如对于（a，b）这样的联合索引页，其访问模式可以是以下情况：

* WHERE a=XXX
* WHERE a=XXX and b=XXX

访问模式一样指的是查询条件一昂，若交替执行上述两种查询，那么InnoDB存储引擎不会对该页构造AHI。此外还有如下要求：

* 以该模式访问了100次
* 页通过该模式访问了N次，其中N=页中记录 * 1/16

根据InnoDB存储引擎官方文档显示，启用AHI后，读取和写入速度可以提高2倍，辅助索引的连续操作性能可以提高5倍。AHI是非常好的优化模式，其设计思想是数据库自优化(self-tuning)，即无需DBA对数据库进行人为调整。通过命令show engine innodb status观察使用情况。

```mysql
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 138401, node heap has 0 buffer(s)
Hash table size 138401, node heap has 0 buffer(s)
Hash table size 138401, node heap has 0 buffer(s)
Hash table size 138401, node heap has 0 buffer(s)
Hash table size 138401, node heap has 1 buffer(s)
Hash table size 138401, node heap has 1 buffer(s)
Hash table size 138401, node heap has 2 buffer(s)
Hash table size 138401, node heap has 4 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
```

哈希索引只能用来搜索等值的查询，如where inde_col = 'xxx'。而对于其他类型的查找，如范围查找，是不能使用哈希索引的，因此这里出现了non-hash searchs/s的情况。通过 hash searchs:non-hash searchs可以大概了解使用哈希索引后的效率。

由于AHI是有InnoDB存储引擎控制的，因此这里的信息仅供参考。不给过可以通过观察show engine innodb status的结果及参数 innodb_adaptive_hash_index来考虑是禁用或启用此特性，默认AHI为开启状态。

######## 4. 异步IO

为了提高磁盘操作性能，当前的数据库系统都采用异步IO(Asynchronous IO，AIO)的方式来处理磁盘操作。AIO的一个优势是可以进行IO Merge操作，也就是将多个IO合并为1个IO，这样可以提高IOPS的性能。

在InnoDB 1.1.X之前，AIO的实现通过InnoDB存储引擎中的代码来模拟实现。而从1.1.X开始，提供了内核级别的AIO支持，称为Native AIO。因此在编译或者运行该版本的MySQL时，需要libaio库的支持。如没有则会出现如下提示：

```shell
/usr/local/mysql/bin/mysqld: error while loading shared libraries: libaio.so.1: cannot open shared object file: No such file or directory
```

需要注意的是，Native AIO 需要操作系统提供支持。Windows和Linux都支持，而Mac OSX则未提供。参数innodb_use_native_aio用来控制是否启用Native AIO，在Linux操作系统下，默认值为ON。

######## 5. 刷新邻接页

工作原理：当刷新一个脏页时，InnoDB存储引擎会检测该页所在去（extent）的所有页，如果是脏页，那么一起进行刷新。这样做的好处显而易见，通过AIO可以将多个IO写入操作合并为一个IO操作，故该工作机制在传统机械磁盘下有着显著的优势。但是需要考虑下面两个问题：

* 是不是可能将不怎么脏的页进行了写入，而该页之后又会很快变成脏页？
* 固态硬盘有着较高的IOPS，是否还需要这个特性？

为此，InnoDB存储引擎从1.2.X开始提供了参数innodb_flush_neighbors，用来控制是否启用该特性。对于传统机械硬盘建议启用该特性，而对于固态硬盘有着超高IOPS性能的磁盘，则建议将该参数设置为0。

## 第3章 文件

### 1. 参数文件

> 默认情况下，MySQL实例会按照一定顺序在指定的位置进行读取配置文件，通过命令mysql --help|grep my.cnf 来寻找即可。

参数分为两种类型：

* 动态参数(dynamic)
* 静态参数(static)

动态参数在MySQL运行中可以进行修改，静态参数不能进行修改，类似于只读概念。

### 2. 日志文件

#### 2.1 错误日志(error log)

错误日志文件对MySQL的启动、运行、关闭过程进行了记录。遇到问题时应该首先查看该文件以便定位问题。该文件记录了所有的错误信息，也记录一些警告信息或正确的信息。用户可以通过命令show variables like 'log_error'来定位文件。

##### 2.2  慢查询日志(slow log)

MySQL启动时有一个参数long_query_time，默认值为10，代表10秒。当SQL执行时长超过此阈值时，将会记录SQL到慢查询日志中。因为执行时间过长的SQL往往都是有问题的SQL，需要优化。默认情况下，MySQL并不启动慢查询日志，需要手动将这个参数设置为ON。

```mysql
mysql> show variables like 'long_query_time'; ## 查询阈值
mysql> show variables like 'log_slow_queries'; ## 查询慢日志开关
```

需要注意的是，运行时间刚好等于long_query_time的语句，并不会被记录下，也就是说，在源代码中判断的是大于，并非大于等于。从MySQL5.1开始，long_query_time开始以微秒记录SQL的运行时间。另一个和慢查询日志有关的参数是log_queries_not_using_indexes，如果运行的SQL语句没有使用索引，则MySQL数据库同样会将这条SQL语句记录到慢查询日志文件。首先确认打开了log_queries_not_using_indexes:

```mysql
mysql> show variables like 'log_queries_not_using_indexes';
```

MySQL5.6.5版本开始新增了一个参数log_throttle_queries_not_using_indexes，用来表示每分钟允许记录到slow log的且未使用索引的SQL语句次数。该值默认为0，表示没有限制。在生产环境下，若没有使用索引，此类SQL语句会频繁被记录到slow log，从而导致slow log文件的大小不断增加。

随着MySQL运行时间的增加，可能会有越来越多的SQL被记录到慢查询日志中，此时要分析该文件就不那么简单直观了。可以使用数据库提供的mysqldumpslow命令，可以很好的解决该问题

```log
mysqldumpslow nh122-190-slow.log
如果希望得到执行时间最长的10条SQL语句
mysqldumpslow -s al -n 10 david.log
```

MySQL5.1开始可以将慢查询的日志记录放入一张表中，慢查询表在mysql库下，名为slow_log，其表结构定义如下：

```mysql
CREATE TABLE `slow_log` (
  `start_time` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  `user_host` mediumtext NOT NULL,
  `query_time` time(6) NOT NULL,
  `lock_time` time(6) NOT NULL,
  `rows_sent` int NOT NULL,
  `rows_examined` int NOT NULL,
  `db` varchar(512) NOT NULL,
  `last_insert_id` int NOT NULL,
  `insert_id` int NOT NULL,
  `server_id` int unsigned NOT NULL,
  `sql_text` mediumblob NOT NULL,
  `thread_id` bigint unsigned NOT NULL
) ENGINE=CSV DEFAULT CHARSET=utf8 COMMENT='Slow log'
```

参数log_output指定了慢查询输出的格式，默认为FILE，可以将它设为TABLE，然后就可以查询mysql.slow_log表了。参数log_output是全局动态的，因此用户可以在线修改。查看slow_log表的定义会发现该表使用的引擎是CSV，对大数据量下的查询效率可能不高。用户可以把slow_log表的引擎切换到MyISAM，并在start_time列上添加索引进一步的提高查询效率。但是，如果已经启动了慢查询，将会提示错误。

MySQL的slow log通过运行时间来对SQL语句进行捕获，但是当数据库容量较小时，可能因为数据库刚建立，此时非常大的可能是数据全部缓存在缓冲池汇总，SQL语句运行的时间可能都是非常短的。

#### 2.3 查询日志(query log)

查询日志记录了所有对MySQL数据库请求的信息，无论这些请求是否得到了正确的执行。默认文件名：主机名.log。

#### 2.4 二进制日志(binary log)

二进制日志记录了对MySQL数据库执行更改的所有操作，但是不包括SELECT和SHOW这类操作，因为操作本身没有对数据进行修改。然而，若操作本身并没有导致数据库发生变化，那么该操作可能也会写入二进制日志。

二进制日志主要有以下几种作用：

* 恢复（recovery）：某些数据的恢复需要二进制日志，例如，在一个数据库全备文件恢复后，用户可以通过二进制日志进行point-in-time的恢复。
* 复制（replication）：其原理与恢复类似，通过复制和执行二进制日志使一条远程的MySQL数据库(一般称为slave或standby)与一台MySQL数据库(一般称为master或primary)进行实时同步。
* 审计（audit）：用户可以通过二进制日志中的信息来进行审计，判断是否有对数据库进行注入的攻击。

二进制日志文件在默认情况下并没有启动，需要手动指定参数来启动。开启二进制日志，会使性能下降1%，但考虑到可以使用复制（replication）和point-in-time的恢复，这些性能损失绝对是可以且应该被接受的。

以下配置文件的参数影响这二进制日志记录的信息和行为：

* max_binlog_size
* binlog_cache_size
* sync_binlog
* binlog-do-db
* binlog-ignore-db
* log-slave-update
* binlog_format

参数**max_binlog_size**指定了单个二进制日志文件的最大值，如果超过该值，则产生新的二进制文件，后缀名+1，并记录到.index文件。当使用事务的表存储引擎（如InnoDB）时，所有未提交（uncommitted）的二进制日志会被记录到一个缓存中去，等该事务提交（committed）时直接将缓冲中的二进制日志写入二进制日志文件，而该缓冲的大小有**binlog_cache_size**决定，默认大小为32K。此外binlog_cache_size是基于会话（session）的，也就是说，当月一个线程开始一个事务时，MySQL会自动分配一个大小为binlog_cache_size的缓存，因此该值的设置需要相当小心，不能设置过大。当一个事务的记录大于设定的binlog_cache_size时，MySQL会把缓冲中的日志写入一个临时文件中，因此该值又不能设置的太小。通过show global status命令查看binlog_cache_use、binlog_cache_disk_use的状态，可以判断当前binlog_cache_size的设置是否合适。binlog_cache_use记录了使用缓冲写二进制日志的次数，binlog_cache_disk_use记录了使用临时文件写二进制日志的次数。

```mysql
mysql> show variables like 'binlog_cache_size';
+------------------+---------+
|Variable_name     |  Value  |
+------------------+---------+
|binlog_cache_size |  32768  |
+------------------+---------+
mysql> show global status like 'binlog_cache%';
+----------------------+---------+
|Variable_name         |  Value  |
+----------------------+---------+
|binlog_cache_disk_use |    0    |
|binlog_cache_use      |  33553  |
+----------------------+---------+
```

使用缓冲次数为33553，临时文件使用次数为0。

在默认情况下，二进制日志并不是在每次写的时候同步到磁盘（用户可以理解为缓冲写）。因此，当数据库所在操作系统发生宕机时，可能会有最后一部分数据没有写入二进制日志文件中，这会给恢复和复制带来问题。参数**sync_binlog**=[N]表示每写缓冲多少次就同步磁盘。如果将N设为1，即sync_binlog=1，表示采用同步写磁盘的方式来写二进制日志，这时写操作不使用操作系统的缓冲来写二进制日志。sync_binlog的默认值为0，如果使用InnoDB存储引擎进行复制，并且想得到最大的高可用性，建议将该值设为ON。不过该值为ON时，确实会对数据库的IO系统带来一定的影响。

参数**binlog-do-db**和**binlog-ignore-db**表示需要写入或忽略写入哪些库的日志。默认为空，表示需要同步所有库的日志到二进制日志。

如果当前数据库是复制中的slave角色，则它不会将从master取得并执行的二进制日志写入自己的二进制日志文件中去。如果需要写入，要设置**log-slave-update**。如果需要搭建master->slave->slave架构的复制，则必须设置该参数。

**binlog_format**参数十分重要，它影响了二进制日志的格式。该参数可设置的值有：STATEMENT、ROW、MIXED。

1. STATEMENT格式和之前的MySQL版本一样，二进制日志文件记录的是逻辑SQL语句。
2. 在ROW格式下，二进制日志记录的不再是简单的SQL语句，而是记录表的行更改情况。基于ROW格式的复制类似于Oracle的物理Standby（当然还是有区别）。同时，对上述提及的Statement格式下复制的问题予以解决。从MySQL5.1版本开始，如果设置了binlog_format为ROW，可以将InnoDB的事务隔离基本设为READ COMMITTED，以获得更好的并发性。
3. 在MIXED格式下，MySQL默认采用STATEMENT格式进行二进制日志文件的记录，但是在一些情况下会使用ROW格式，可能的情况有：
   1. 表的存储引擎为NDB，这是对表的DML操作都会以ROW格式记录。
   2. 使用了UUID()、USER()、CURRENT_USER()、FOUND_ROWS()、ROW_COUNT()等不确定函数。
   3. 使用了INSERT DELAY语句。
   4. 使用了用户定义的函数（UDF）
   5. 使用了临时表（temporary table）

此外，binlog_format参数还有对于存储引擎的限制

| 存储引擎      | Row格式 | Statement格式 |
| --------- | ----- | ----------- |
| InnoDB    | Yes   | Yes         |
| MyISAM    | Yes   | Yes         |
| HEAP      | Yes   | Yes         |
| MERGE     | Yes   | Yes         |
| NDB       | Yes   | No          |
| Archive   | Yes   | Yes         |
| CSV       | Yes   | Yes         |
| Federate  | Yes   | Yes         |
| Blockhole | No    | Yes         |

通常情况下，将参数binlog_format设置为ROW，可以为数据库的恢复和复制带来更好的可靠性。但是也会增加二进制文件的大小，有些语句下的ROW格式可能需要更大的容量。而且由于复制是采用传输二进制日志方式实现的，因此复制的网络开销也有所增加。

查看二进制日志文件的方式，通过mysqlbinlog命令完成。

#### 3. 套接字文件

在UNIX系统下本地连接MySQL可以采用UNIX域套接字方式，这种方式需要一个套接字(socket)文件。套接字文件可由参数socket控制。一般在/tmp/目录下，名为mysql.sock。

#### 4. pid文件

MySQL启动后，会将自己的进程ID写入一个文件中，这个文件就是pid文件。该文件可由参数pid_file控制，默认位于数据库目录下，文件名为主机名.pid。

#### 5. 表结构定义文件

因为MySQL插件式存储引擎体系结构的关系，MySQL数据的存储是根据表进行的，每个表都会有与之对应的文件。但不论表采用何种引擎，MySQL都有一个亿frm为后缀名的文件，这个文件记录了该表的表结构定义。frm还用来存放视图的定义，如用户创建了一个v_a视图，那么对应的会产生一个v_a.frm文件，用来记录视图的定义，该文件是文本文件，可以直接使用cat命令进行查看。

#### 6. InnoDB存储引擎文件

##### 6.1 表空间文件

InnoDB采用将存储的数据按表空间（tablespace）进行存放的设计。在默认配置下会有一个初始大小为10MB，名为ibdata1的文件。该文件就是默认的表空间文件（tablespace file），用户可以通过参数innodb_data_file_path对其进行设置。格式如下：

`innodb_data_file_path=datafile_spec1[;datafile_spec2]...`

用户可以通过多个文件组成一个表空间，同时制定文件的属性，如：

```ini
[mysqld]
innodb_data_file_path = /db/ibdata1:2000M;/dr2/db/ibdata2:2000M:autoextend
```

这里将/db/ibdata1和/dr2/db/ibdata2两个文件用来组成表空间。若这两个文件位于不同的磁盘上，磁盘的负载可能被平均，因此可以提高数据库的整体性能。同时，两个文件的文件名后都跟了属性，表示文件ibdata1的大小为2000MB，文件ibdata2的大小为2000MB，如果用完这2000MB，该文件可以自动增长(autoextend)。

设置innodb_data_file_path参数后，所有基于InnoDB存储引擎的表的数据都会记录到该共享表空间中。若设置了参数innodb_file_per_table，则用户可以将每个基于InnoDB存储引擎的表产生一个独立的表空间。独立表空间的命名规则为：表名.ibd。通过这样的方式，用户不用将所有的数据都存放与默认的表空间中。

需要注意的是，单独的表空间文件仅存储该表的数据、索引和插入缓冲BITMAP等信息，其余信息还是存放在默认的表空间中。

![](../../../public/assets/images/mysql/Innodb表存储引擎体文件.png)

######## 6.2 重做日志文件

在默认情况下，在InnoDB存储引擎的数据目录下会有两个名为ib_logfile0和ib_logfile1的文件。在MySQL官方手册中将其称为InnoDB存储引擎的日志文件，不过更准确的定义应该是重做日志文件（redo log file）。它们记录了对于INNODB存储引擎的事务日志。

当实例或介质失败（media failure）时，重做日志文件就能派上用场。例如，数据库由于所在主机掉电导致实例失败，InnoDB存储引擎会使用重做日志恢复到掉电前的时刻，以此来保证数据的完整性。

每个InnoDB存储引擎至少有1个重做日志文件组（group），每个文件组下至少有2个重做日志文件，如默认的ib_logfile0和ib_logfile1。为了得到更高的可靠性，用户可以设置多个的镜像日志组（mirrored log groups），将不同的文件组放在不同的磁盘上，以此提高重做日志的高可用性。在日志组中每个重做日志文件的大小一致，并以循环写入的方式运行。InnoDB存储引擎先写重做日志文件1，当达到文件的末尾时，会切换至重做日志文件2，再当重做日志文件2也被写满时，会切回到重做日志文件1中。

![](../../../public/assets/images/mysql/InnoDB重做日志文件组.png)

下列参数影响这重做日志文件的属性：

* innodb_log_file_size
* innodb_log_files_in_group
* innodb_mirrored_log_groups
* innodb_log_group_home_dir

参数**innodb_log_file_size**指定每个重做日志文件的大小。

参数**innodb_log_files_in_group**指定了日志文件组中重做日志文件的数量，默认为2。

参数**innodb_mirrored_log_groups**指定了日志镜像文件组的数量，默认为1，表示只有一个日志文件组，没有镜像。若磁盘本身已经做了高可用的方案，如磁盘这列，那么可以不开启重做日志镜像的功能。

参数**innodb_log_group_home_dir**指定了日志文件组所在路径，默认为./，表示在MySQL数据库的数据目下。

------

重做日志文件的大小设置对于InnoDB存储引擎的性能有着非常大的影响。重做日志文件不能设置的太大，太大在恢复时可能需要很长的时间；太小，容易导致一个事务的日志需要多次切换重做日志文件。此外，重做日志文件太小会导致频繁发生async checkpoint，导致性能抖动。

既然同样是记录事务日志，与之前的二进制日志有什么区别？

首先，二进制日志会记录所有与MySQL数据库有关的日志记录，包括InnoDB、MyISAM、HEAP等其他存储引擎的日志。而InnoDB存储引擎的重做日志只记录有关自己本身的事务日志。

其次，记录的内容不同，无论用户将二进制日志文件记录的格式设为STATEMENT还是ROW，又或者是MIXED，其记录的都是关于一个事务的具体操作内容，即该日志是逻辑日志。而InnoDB存储引擎的重做日志文件记录的是关于每个页(page)的更改的物理情况。

此外，写入的时间也不同，二进制日志文件仅在事务提交前进行提交，即只写磁盘一次，不论这时该事务多大。而在事务进行的过程中，却不断有重做日志条目(redo entry)被写入到重做日志文件中。

------

写入重做日志文件的操作不是直接写，而是先写入一个重做日志缓冲(redo log buffer)中，然后按照一定的条件顺序地写入日志文件。

![](../../../public/assets/images/mysql/重做日志写入过程.png)

从重做日志缓冲往磁盘写入时，是按512个字节，也就是一个扇区的大小进行写入。因为扇区是写入的最小单位，因此可以保证写入必定是成功的。因此在重做日志的写入过程中不需要有doublewrite。

**重做日志缓冲按照什么样的条件顺序写入日志文件？**

主线程(master thread)每秒会将重做日志缓冲写入磁盘的重做日志文件中，不论事务是否已经提交。另一个触发写磁盘的过程是有参数innodb_flush_log_at_trx_commit控制，表示在提交(commit)操作时，处理重做日志的方式。

参数**innodb_flush_log_at_trx_commit**的有效值为0、1、2。

* 0代表当事务提交时，并不将事务的重做日志写入磁盘上的日志文件，而是等待主线程每秒的刷新。
* 1表示在执行commit时将重做日志缓冲同步写到磁盘，即伴有fsync的调用。
* 2表示将重做日志异步写到磁盘，即写文件系统的缓存中。因此不能完全保证在执行commit时肯定会写入重做日志文件，只是有这个动作发生。

因此为了保证事务ACID中的持久性，必须将innodb_flush_log_at_trx_commit设置为1，也就是每当有事务提交时，就必须确保事务都已经写入重做日志文件。那么当数据库因为意外发生宕机时，可以通过重做日志文件恢复，并保证可以恢复已经提交的事务。而将重做日志文件设置为0或2，都有可能发生恢复时部分事务丢失。不同之处在于，设置为2时，当MySQL数据库发生宕机而操作系统及服务器并没有发生宕机时，由于此时未写入磁盘的事务日志保存在文件系统缓存中，当恢复时同样能保证数据不丢失。

## 第4章 表

### 1. 索引组织表

在InnoDB存储引擎中，表都是根据主键顺序组织存放的，这种存储方式的表称为索引组织表(index organized table)。在InnoDB存储引擎表中，每张表都有个主键(primary key)，如果在创建表时没有显式地定义主键，则InnoDB存储引擎会按如下方式选择或创建主键：

* 首先判断表汇总是否有非空的唯一索引(Unique Not Null)，如果有，则改了即为主键。
* 如果不符合上述条件，InnoDB存储引擎自动创建一个6字节大小的指针。

当表中有多个非空唯一索引时，InnoDB存储引擎将选择建表时第一个定义的非空唯一索引为主键。需要注意的是，主键的选择根据的是定义索引的顺序，而不是建表时列的顺序。

### 2. InnoDB逻辑存储结构

从InnoDB存储引擎的逻辑存储结构看，所有数据都被逻辑地存放在一个空间中，称为表空间(tablespace)。表空间又由段(segmetn)、区(extent)、页(page)组成。页在一些文档中有时也称为块(block)，InnoDB存储引擎的逻辑存储结构大致如图。

![](../../../public/assets/images/mysql/InnoDB存储引擎逻辑存储结构.png)

#### 2.1 表空间（Tablespace）

表空间可以看做是InnoDB存储引擎逻辑结构的最高层，所有的数据都放在表空间中。默认情况下，有一个共享表空间ibdata1，即所有数据都存放在这个表空间内。如果用户启用了参数innodb_file_per_table，则每张表内的数据可以单独放到一个表空间。

如果启用了innodb_file_per_table的参数，需要注意的是每张表的表空间内存放的只是数据、索引和插入缓冲bitmap页，其他类的数据，如回滚(undo)信息、插入缓冲索引页、系统事务信息、二次写缓冲(Double write buffer)等还是存放在原来的共享表空间内。即使启用了参数innodb_file_per_table之后，共享表空间还是会不断地增加其大小。

#### 2.2 段（Segment）

表空间是由段组成的，常见的段有数据段、索引段、回滚段等。因为InnoDB存储引擎表是索引组织的，因此数据即索引，索引即数据。那么数据段即为B+树的叶子节点（Leaf node segment），索引段即为B+树的非索引节点（Non-Leaf node segment）。

#### 2.3 区（extent）

区是由连续页组成的空间，在任何情况下每个区的大小都为1MB。为了保证区中页的连续性，InnoDB存储引擎一次从磁盘申请4~5个区。在默认情况下，InnoDB存储引擎页的大小为16KB，即一个区中一共有64个连续的页。InnoDB 1.0.X开始引入压缩页，每个页的大小可以通过参数key_block_size设置为2K、4K、8K，因此每个区对应页的数量应该为512、256、128。

InnoDB 1.2.X版本新增了参数innodb_page_size，通过该参数可以将默认页的大小设置为4K、8K，但是页中的数据库不是压缩。 这是区中页的数量同样也为256、128。总之，无论页的大小如何变化，区的大小总是1MB。

#### 2.4 页（page）

同大多数数据库一样，InnoDB有页的概念，页是InnoDB磁盘管理的最小单位。默认每个页大小为16KB，通过参数也可以设置为4K、8K、16K。

在InnoDB存储引擎中，常见的页类型：

* 数据页（B-tree Node）
* undo页（undo Log Page）
* 系统页（System Page）
* 事务数据页（Transaction system Page）
* 插入缓冲位图页（Insert Buffer Bitmap）
* 插入缓冲空闲列表页（Insert Buffer Free List）
* 未压缩的二进制大对象页（Uncompressed BLOB Page）
* 压缩的二进制大对象页（Compressed BLOB Page）

#### 2.5 行（row）

InnoDB存储引擎是面向列的(row-oriented)，也就是数据是按行进行存放的。每个页存放的行记录也是有硬性定义的，最多允许存放16KB/2-200行的记录，即7992行记录。既然有row-oriented，也就有column-oriented数据库。MySQL infobright存储引擎就是按列来存放数据的。

### 3. InnoDB行记录格式（略过）

### 4. InnoDB数据页结构（略过）

### 5. 约束

#### 5.1 数据完整性

一般来说，数据完整性有以下三种形式：

* 实体完整性保证表中有一个主键。
  
  在InnoDB存储引擎表中，用户可以通过定义Primary Key或Unique Key约束来保证实体的完整性。用户还可以通过编写一个触发器来保证数据的完整性。

* 域完整性保证数据每列的值满足特定的条件。
  
  在InnoDB存储引擎表中，域完整性可以通过以下几种途径来保证：
  
  * 选择合适的数据类型确保一个数据值满足特定的条件。
  * 外键约束（Foreign Key）。
  * 还可以考虑用default约束作为强制域完整性的一个方面。

* 参照完整性保证两张表之间的关系。
  
  InnoDB存储引擎支持外键，因此允许用户定义外键以强制参照完整性，也可以通过编写触发以强制执行。

对于InnoDB存储引擎本身而言，提供了以下几种约束：

* Primary Key
* Unique Key
* Foreign Key
* Default
* NOT NULL

######## 5.2 约束的创建和查找

约束创建可以采用以下两种方式：

* 表建立时就进行约束定义
* 利用ALTER TABLE命令来进行创建约束

#### 5.3 约束和索引的区别

约束是一个逻辑的概念，用来保证数据的完整性，而索引是一个数据结构，既有逻辑上的概念，在数据库中还代表着物理存储的方式。

#### 5.4 对错误数据的约束

在某些默认设置下，MySQL数据库允许非法的或不正确的数据的插入或更新，又或者可以在数据库内部将其转化为一个合法的值，如向NOT NULL的字段插入一个NULL值，MySQL数据库会将其更改为0再进行插入，因此数据库本身没有对数据的正确性进行约束。比如往date列插入了一个非法的日期'2009-02-30'。MySQL并没有报错，而是显示了警告(Waring)。如果用户想通过约束对数据库非法数据的插入或更更新，那么用户必须设置参数sql_mode，用来严格审核输入的参数。

#### 5.5 ENUM和SET约束

#### 5.6 触发器与约束

#### 5.7 外键约束

外键用来保证参照完整性，MySQL下的MyISAM存储引擎本身不支持外键，对于外键的定义只是起到一个注释作用。外键的定义如下：

```mysql
[CONSTRAINT] [symbol] FOREIGN KEY
[index_name] (index_col_name, ...)
REFERENCES tbl_name (index_col_name, ...)
[ON DELETE reference_option]
[ON UPDATE reference_option]
reference_option:
RESTRICT | CASCADE | SET NULL | NO ACTION
```

可以在CREATE TABLE时就添加外键，也可以在表创建后通过ALTER TABLE命令来添加。一般来说，被引用的表为父表，引用的表称为子表。外键定义时的ON DELETE和ON UPDATE表示在对父表进行DELETE和UPDATE操作时，对子表所做的操作，可以定义的子表操作有：

* CASCADE
* SET NULL
* NO ACTION
* RESTRICT

CASCADE表示当父表发生DELETE或UPDATE操作时，对相应的子表中的数据页进行DELETE或UPDATE操作。

SET NULL表示父表发生DELETE或UPDATE操作时，相应子表中的数据被更新为NULL值，但是子表中的列必须允许为NULL值。

NO ACTION表示父表发生DELETE或UPDATE操作时，抛出错误，不允许这类操作发生。

RESTRICT表示父表发生DELETE或UPDATE操作时，抛出错误，不允许这类操作发生。如果定义外键没有指定ON DELETE或ON UPDATE，RESTRICT就是默认的外键设置。

#### 6. 视图

在MySQL数据库中，视图（View）是一个命名的虚表，它由一个SQL查询来定义，可以当做表使用。与持久表（permanent table）不同的是，视图中的数据没有实际的物理存储。

##### 6.1 视图的作用

视图的主要用途之一是被用做一个抽象装置，特别是对于一些应用程序，程序本身不需要关心基表（base table）的结构，只需要按照视图定义来取数据或更新数据，因此视图同时在一定程度上起到一个安全层的作用。

```mysql
CREATE [OR REPLACE]
[ALGORITHM = {UNDEFINED | MERGE | TEMPTABLE}]
[DEFINER = {user | CURRENT_USER}]
[SQL SECURITY {DEFINER | INVOKER}]
VIEW view_name [(column_list)]
AS select_statement
[WITH [CASCADE | LOCAL] CHECK OPTION]
```

虽然视图是基于基表的一个虚拟表，但是用户可以对某些视图进行更新操作，其本质就是通过视图的定义来更新基本表。一般称可以进行更新操作的视图为可更新视图（updateable view）。视图定义中的 with check option就是针对于可更新的视图的，即更新的值是否需要检查。

##### 6.2 物化视图

Oracle支持物化视图——该视图不是基于基表的虚表，而是根据基表实际存在的实表，即物化视图的数据存储在非易失的存储设备上。物化视图可以用于预先计算并保存多表的链接（JOIN）或聚集（GROUP BY）等耗时较多的SQL操作结果。这样，在执行复杂查询时，就可以避免进行这些耗时的操作，从而快速得到结果。物化视图的好处是对于一些复杂的统计类查询能直接查出结果。

在Oracle中，物化是的创建方式包括以下两种：

* BUILD IMMEDIATE
* BUILD DEFERRED

BUILD IMMEDIDATE是默认的创建方式，在创建物化视图的时候就生成数据，而BUILD DEFERRED则在创建物化视图时不生成数据，以后根据需要再生成数据。查询重写是指当对物化视图的基表进行查询时，数据库会自动判断能否通过查询物化视图来直接得到最终的结果，如果可以，则避免了聚集或连接等这类较为复杂的SQL操作，直接从已经计算好的物化视图中得到所需的数据。物化视图的刷新是指当基表发生了DML操作，物化视图何时采用哪种方式和基表进行同步。刷新的的模式有两种：

* ON DEMAND
* ON COMMIT

ON DEMAND意味着物化视图在用户需要的时候进行刷新，ON COMMIT意味着物化视图在对基表的DML操作提交时的同事进行刷新。而刷新的方式有4种：

* FAST
* COMPLETE
* FORCE
* NEVER

FAST采用增量刷新，只刷新自上次刷新以后进行的修改。COMPLETE是对整个物化视图进行完全的刷新。FORCE，数据库在刷新时会去判断是否可以进行快速刷新，如果可以，则用FAST，否则采用COMPLETE。NEVER是物化视图不进行任何刷新。

MySQL本身并不支持物化视图，换句话说，MySQL数据库中的视图总是虚拟的。但是用户可以通过一些机制来实现物化视图的功能。例如要创建一个 ON DEMAND的物化视图还是比较简单的，用户只需定时把数据导入到另一张表。

### 7. 分区表

#### 7.1 分区概述

分区功能并不是在存储引擎层完成的，因此不是只有InnoDb才支持分区，常见的存储引擎MyISAM、NDB都支持。CSV、FEDORATED、MERGE等就不支持。

分区的过程是将一个表或索引分解为多个更小、更可管理的部分。就访问数据库的应用而言，从逻辑上讲，只要一个表或一个索引，但是在物理上这个表或索引可能由数十个物理分区组成。每个分区都是独立的对象，可以独自处理，也可以作为一个更大的对象的一部分进行处理。

MySQL数据库支持的分区类型为水平分区(将同一表中不同**行的记录**分配到不同的物理文件中)，并不支持垂直分区(将同一表中不同**列的记录**分配到不同物理文件中)。此外，MySQL数据库的分区是局部分区索引，一个分区中既存放了数据又存放了索引。而全局分区是指，数据存放在各个分区中，但是所有数据的索引放在一个对象中。可以通过以下命令查看当前数据库是否启用了分区功能：

```mysql
mysql> show variables like '%partition%';
Variable_name: have_partitioning
        Value: YES
```

当前MySQL数据库支持以下几种类型的分区：

* RANGE分区：行数基于属于一个给定连续区间的列值被放入分区
* LIST分区：和RANGE分区，只是LIST分区面向的是离散的值
* HASH分区：根据用户自定义的表达式的返回值来进行分区，返回值不能为负数
* KEY分区：根据MySQL数据库提供的哈希函数来进行分区

无论创建何种类型的分区，如果表中存在主键或唯一索引时，分区列必须是唯一索引的一个组成部分。

```mysql
create table t1(
    col1 int not null,
    col2 date not null,
    col3 int not null,
    col4 int not null,
    unique key(col1,col2)
)
partition by hash(col3)
partitions 4;
ERROR 1503: A PRIMARY KEY must include all columns in the table's partitioning function
```

唯一索引可以是允许NULL值的，并且分区列只要是唯一索引的一个组成部分，不需要整个唯一索引都是分区列。

```mysql
create table t1(
    col1 int null,
    col2 date null,
    col3 int null,
    col4 int null,
    unique key (col1, col2, col3, col4)
)
partition by hash(col3)
partitions 4;
Query OK, 0 rows affected
```

如果建表时没有指定主键，唯一索引，可以指定任何一个列作为分区列，因此下面两句SQL都是正确的。

```mysql
create table t1 (
    col1 int null,
    col2 date null,
    col3 int null,
    col4 int null
) engine=innodb
partition by hash(col3)
partitions 4;

create table t1 (
    col1 int null,
    col2 date null,
    col3 int null,
    col4 int null,
    key (col4)
) engine=innodb
partition by hash(col3)
partitions 4;
```

#### 7.2 分区类型(略)

------

## 第7章 事务

### 1. 事务的分类

从事务理论的角度来说，可以把事务分为以下几种类型：

* 扁平事务（Flat Transaction）
* 带有保存点的扁平事务（Flat Transaction with Savepoints）
* 链事务（Chained Transaction）
* 嵌套事务（Nested Transaction）
* 分布式事务（Distributed Transaction）

**扁平事务** 是事务类型中最简单的一种，但在实际生产环境中，这可能是使用最为频繁的事务。在扁平事务中，所有操作都处于同一层次，其由begin work开始，由commit work或rollback work结束，其间的操作都是原子的，因此 **扁平事务** 是应用程序成为原子操作的基本组成模块。

![](../../../public/assets/images/mysql/InnoDB扁平事务的三种情况.png)

**扁平事务** 的主要限制是不能提交或者回滚事务的某一部分，或分几个步骤提交。这里给出一个扁平事务不足以支持的例子。例如用户在旅行网站上进行自己的旅行度假计划，用户设想从杭州到意大利的佛罗伦萨，这两个城市之间没有直达的班机，需要用户预订并转乘航班，或者需要搭火车等待。用户预订旅行度假的事务为：

```mysql
Begin Work
## S1: 预订杭州到上海的高铁
## S2：上海浦东国际机场坐飞机，预订去米兰的航班
## S3：在米兰转火车前往佛罗伦萨，预订去佛罗伦萨的火车
```

但是当用户执行到S3时，发现由于飞机抵达米兰的时间太晚，已经没有当日的火车，于是决定在米兰住一晚，第二天出发。这时如果事务为扁平事务，则需要回滚S1、S2、S3的三个操作，这个代价就显得有点大。因为当再次进行该事务时，S1、S2的执行计划是不变的。也就是说，如果支持有计划的回滚操作，那么就不需要终止整个事务。因此就出现了 **带有保存点的扁平事务**。

**带有保存点的扁平事务**，除了支持扁平事务支持的操作外，允许在事务执行过程中回滚到同一事务中较早的一个状态。这是因为某些事务可能在执行过程中出现错误并不会导致所有的操作都无效，放弃整个事务不合乎要求，开销也太大。**保存点(Savepoint)** 用来通知系统应该记住事务当前的状态，以便当之后发生错误时，事务能回到保存点当时的状态。

对于扁平的事务来说，其隐式地设置了一个保存点。然而在整个事务中，只有这一个保存点，因此回滚只能回滚到事务开始时的状态。保存点用 save work 函数来建立，通知系统记录当前的处理状态。当出现问题时，保存点能用作内部的重启动点，根据应用逻辑，决定是回到最近一个保存点还是其他更早的保存点。

<img src="../../../public/assets/images/mysql/InnoDB在事务中使用保存点.png" style="zoom:67%;" />

灰色部分的操作表示由rollback work而导致部分回滚，实际并没有执行的操作。当begin work开启一个事务时，隐式地包含了一个保存点，当事务通过rollback work：2发出部分回滚命令时，事务回滚到保存点2，接着一次执行，并再次执行到rollback work：7，直到最后commit work操作，这时表示事务结束，除灰色阴影部分的操作外，其余操作都已经执行，并且提交。

另一点需要注意的是，保存点在事务内部是递增的，从图中可以看出。有人可能会想，返回保存点2以后，下一个保存点可以为3，因为之前的工作都终止了。然而新的保存点编号为5，这意味着rollback 不影响保存点的计数，并且单调递增的编号能保持事务执行的整个历史过程，包括在执行过程中想法的改变。

此外，当事务执行rollback work：2命令发出部分回滚命令时，要记住事务并没有完全被回滚，只是回滚到了保存点2而已。这代表当前事务还是活跃的，如果想要完全回滚事务，还需要执行命令rollback work。

**链事务** 可视为保存点模式的一个变种。带有保存点的扁平事务，当 **发生系统崩溃** 时，所有的保存点都将消失，因为其保存点是易失的（volatile），而非持久的（persistent）。这意味着当进行恢复时，事务需要从开始处重新执行，而不能从最近的一个保存点继续执行。

链事务的思想是：在提交一个事务时，释放不需要的数据对象，将必要的处理上下文隐式地传给下一个要开始的事务。注意，提交事务操作和开始下一个事务操作将合并为一个原子操作。这意味着下一个事务将看到上一个事务的结果，就好像在一个事务中进行的一样。

![](../../../public/assets/images/mysql/InnoDB链事务的工作方式.png)

链事务与带有保存点的扁平事务不同的是，带有保存点的扁平事务能回滚到任意正确的保存点。而链事务的回滚仅限于当前事务，即只能恢复到最近一个的保存点。对于锁的处理，两者也不相同。链事务在执行commit后即释放了当前事务所持有的锁，而带有保存点的扁平事务不影响迄今为止所持有的锁。

**嵌套事务** 是一个层次结构框架。由一个顶层事务（top-level transaction）控制着各个层次的事务。顶层事务之下嵌套的事务被称为子事务（subtransaction），其控制每一个局部的变换。

![](../../../public/assets/images/mysql/InnoDB嵌套事务的结构层次.png)

下面给出Moss对嵌套事务的定义：

1. 嵌套事务是由若干事务组成的一颗树，子树既可以是嵌套事务，也可以是扁平事务。
2. 处在叶节点的事务是扁平事务。但是每个子事务从根到叶节点的距离可以是不同的。
3. 位于根节点的事务称为顶层事务，其他事务称为子事务。事务的前驱称（predecessor）为父事务（parent），事务的下一层称为儿子事务（child）。
4. 子事务既可以提交也可以回滚。但是它的提交操作并不马上生效，除非其父事务已经提交。因此可以推论出，任何子事务都在顶层事务提交后才真正的提交。
5. 树中任意一个事务的回滚会引起它的所有子事务一起回滚，故子事务仅保留A、C、I特性，不具有D的特性。

在Moss的理论中，实际的工作是交由叶子节点来完成的，即只有叶子节点的事务才能访问数据库、发送消息、获取其他类型的资源。而高层的事务仅负责逻辑控制，决定何时调用相关的子事务。即使一个系统不支持嵌套事务，用户也可以通过保存点技术来模拟嵌套事务。

**分布式事务** 通常是一个在分布式环境下运行的扁平事务，因此需要根据数据所在位置访问网络中的不同节点。

对于InnoDB存储引擎来说，其支持扁平事务、带有保存点的事务、链事务、分布式事务。对于嵌套事务，其并不原生支持，因此对有并行事务需求的用户来说，MySQL的InnoDB存储引擎就显得无能为力，不过还是可以通过带有保存点的事务来模拟。

### 2. 事务的实现

事务隔离性由锁来实现。原子性、一致性、持久性通过数据库的 **redo log** 和 **undo log** 来完成。redo log称为重做日志，用来保证事务的原子性和持久性。undo log称为回滚日志，用来保证事务的一致性。

redo和undo的作用都可以视为是一种恢复操作，redo恢复提交事务修改的页操作，而undo回滚行记录到某个特定版本。因此两者记录的内容不同，redo通常是物理日志，记录的是页的物理修改操作。undo是逻辑日志，根据每行记录进行记录。

------

######## 2.1 redo

########## 1. 基本概念

重做日志用来实现事务的持久性，即事务ACID中的D。其由两部分组成，一是内存中的日志缓冲（redo log buffer），其是易失的；而是重做日志文件（redo log file），其是持久的。

InnoDB是事务的存储引擎，其通过 **Force Log at Commit** 机制实现事务的持久性，即当事务提交（commit）时，必须先将事务的所有日志写入到重做日志文件进行持久化，待事务的commit操作完成才算完成。这里的日志是指重做日志，在InnoDB存储引擎中，由两部分组成，即redo log 和 undo log。redo log用来保证事物的持久性，undo log用来帮助事务回滚及MVCC的功能。redo log基本上都是顺序写的，在数据库运行时不需要对redo log的文件进行读取操作。而undo log是需要进行随机读写的。

为了保证

每次日志都写入重做日志，在每次将重做日志缓冲写入重做日志文件后，InnoDB存储引擎都需要调用一次fsync操作。由于重做日志文件打开并没有使用O_DIRECT选项，因此重做日志缓冲先写入文件系统缓存。为了确保重做日志写入磁盘，必须进行一次fsync操作。由于fsync的效率取决于磁盘的性能，因此磁盘的性能决定了事务提交的性能，也就是数据库的性能。

InnoDB存储引擎中的参数 **innodb_flush_log_at_trx_commit** 用来控制重做日志刷新到磁盘的策略。该参数的默认值为：1，表示事务提交时必须调用一次fsync操作。还可以设置该参数的值为0和2。0表示事务提交时不进行写入重做日志操作，这个操作仅在master thread中完成，而在master thread中每1秒会进行一次重做日志的fsync操作。2表示事务提交时将重做日志写入到文件系统的缓存中，不进行fsync操作，即由文件系统决定什么时间进行刷新磁盘。在2这个设置下，数据库宕机而操作系统不宕机，并不会导致事务丢失。若操作系统宕机，且文件系统并没有刷盘，重启数据库会丢失未刷盘的那部分事务。

有一个实验数据如下：

| innodb_flush_log_at_trx_commit | 执行所用时间     |
|:------------------------------:|:----------:|
| 0                              | 13.90s     |
| 1                              | 1min53.11s |
| 2                              | 23.37s     |

虽然用户可以通过修改参数为0或2来提高事务的性能，但是这个方式丧失了事务的ACID特性。

########## 2. log block

在InnoDB存储引擎中，重做日志都是以512字节进行存储的。这意味着重做日志缓存、重做日志文件都是以块（block）的方式进行保存的，称之为重做日志块（redo log block），每块的大小为512字节。

若一个页中产生的重做日志数量大于512字节 ，那么需要分割为多个重做日志块进行存储。此外，由于重做日志块的大小和磁盘扇区大小一样，都是512字节，因此重做日志的写入可以保证原子性，不需要doublewrite技术。

重做日志块除了日志本身之外，还由日志块头（log block header）及日志块尾（log block tailer）两部分组成。重做日志头一共占用12字节，重做日志尾占用8字节。故每个重做日志块实际可以存储的大小为492字节。

![](../../../public/assets/images/mysql/InnoDB重做日志块缓存结构.png)

| 名        称                | 占用字节 |
|:-------------------------:|:----:|
| LOG_BLOCK_HDR_NO          | 4    |
| LOG_BLOCK_HDR_DATA_LEN    | 2    |
| LOG_BLOCK_FIRST_REC_GROUP | 2    |
| LOG_BLOCK_CHECKPOINT_NO   | 4    |

log buffer是有log block组成，在内部log buffer就好似一个数组，因此**LOG_BLOCK_HDR_NO**用来标记这个数组中的位置。其是递增并且循环使用的，占用4个字节，但是由于第一位用来判断是否是flush bit，所以最大值为2G。

**LOG_BLOCK_HDR_DATA_LEN**占用2字节，表示log block所占用的大小。当log block被写满时，该值为0x200，表示使用全部log block空间，即占用512字节。

**LOG_BLOCK_FIRST_REC_GROUP**占用2个字节，表示log back中第一个日志所在的偏移量。如果该值的大小和**LOG_BLOCK_HDR_DATA_LEN**相同，则表示当前log block不包含新的日志。如事务T1的重做日志1占用762字节，事务T2的重做日志占用100字节。由于每个log block实际只能保存492个字节，因此其在log buffer中的情况如图所示：

![](../../../public/assets/images/mysql/log buffer示例图.png)

从图中观察得到，由于事务T1的重做日志占用762字节，因此需要占用两个log block。左侧的log block中的LOG_BLOCK_FIRST_REC_GROUP为12，即log block中第一个日志的开始位置（`自己注：因为log block header占用大小为12字节，且LOG_BLOCK_FIRST_REC_GROUP表示的是偏移量，真正的log body就是需要从头偏移12字节开始写入`）。在第二个log block中，由于包含了之前事务T1的重做日志，事务T2的日志才是log block的第一个日志，因此该log block的LOG_BLOCK_FIRST_REC_GROUP为282（270+12）。

**LOG_BLOCK_CHECKPOINT_NO**占用4个字节，表示该log block最后写入时的检查点第4个字节的值。

log block tailer只由1个部分组成，其值和LOG_BLOCK_HDR_NO相同，并在函数log_block_init中被初始化。

| 名      称         | 大小（字节） |
|:----------------:|:------:|
| LOG_BLOCK_TRL_NO | 4      |

########## 3. log group

log group为重做日志组，其中有多个重做日志文件。虽然源码中已支持log group的镜像功能，但是在ha_innobase.cc文件中禁止了该功能。因此InnoDB存储引擎实际上只有一个log group。

log group是一个逻辑上的概念，并没有一个实际存储的物理文件来表示log group信息。log group由多个重做日志文件组成，每个log group中的日志文件大小是相同的。

重做日志文件中存储的就是之前在log buffer中保存的log block，因此其也是根据块的方式进行物理存储的管理，每个块的大小与log block一样，同样为512字节。在InnoDB存储引擎运行过程中，log buffer根据一定的规则将内存中的log block刷新到磁盘。这个规则具体是：

* 事务提交时
* 当log buffer中有一半的内存空间已经被使用时
* log checkpoint时

对于log block的写入追加（append）在redo log file的最后部分，当一个redo log file 写满时，会接着写入下一个redo log file，其使用方式为round-robin。

虽然log block总是在redo log file的最后部分进行写入，但并不是顺序写入的。redo log file 除了保存log buffer刷新到磁盘的log block，还保存了一些其他的信息。对于log group中的第一个redo log file，其前2KB的部分保存4个512字节大小的块，其中存放的内容如下表：

| 名        称      | 大小（字节） |
|:---------------:|:------:|
| log file header | 512    |
| checkpoint1     | 512    |
| 空               | 512    |
| checkpoint2     | 512    |

需要特别注意的是，上述信息仅在每个log group的第一个redo log file中进行存储。log group中的其余redo log file仅保留这些空间，但不保存上述信息。正因为保存了这些信息，对redo log 的写入并不是完全顺序的。因为其除了log block的写入操作，还需要更新前2KB部分的信息，这些信息对于InnoDB存储引擎的恢复操作来说非常关键和重要。

在log file header后面的部分为InnoDB存储引擎保存的checkpoint值，其设计是交替写入，这样的设计避免了因介质失败而导致无法找到可用的checkpoint的情况。

########## 4. 重做日志格式

不同的数据库操作会有对应的重做日志格式。此外，由于InnoDB存储引擎的存储管理是基于页的，故其重做日志格式也是基于页的。虽然有着不同的重做日志格式，但是他们有着通用的头部格式，如图：

![](../../../public/assets/images/mysql/InnoDB重做日志格式.png)

通用的头部格式由以下3部分组成：

* redo_log_type：重做日志的类型
* space：表空间的ID
* page_no：页的偏移量

之后的body部分，根据重做日志类型的不同，会有不同的存储内容，例如：

![](../../../public/assets/images/mysql/InnoDB插入和删除的重做日志格式.png)

########## 5. LSN

LSN是Log Sequence Number的缩写，其代表的是日志序列号。在InnoDB存储引擎中，LSN占用8字节，并且单调递增。LSN表示的含义有：

* 重做日志写入的总量
* checkpoint的位置
* 页的版本

LSN不仅记录在重做日志，还存在于每个页中。在每个页的头部，有一个值FILE_PAGE_LSN，记录了该页的LSN。在页中，LSN表示该页最后刷新时LSN的大小。因为重做日志记录的是每个页的日志，因此页中的LSN用来判断是否需要进行恢复操作。

########## 6. 恢复

InnoDB存储引擎在启动时不管上次数据库运行是否正常关闭，都会尝试进行恢复操作。因为重做日志记录的是物理日志，因此恢复的速度比逻辑日志，如二进制日志，要快得多。与此同时，InnoDB存储引擎自身也对恢复进行了一定程度的优化，如顺序读取及并行应用重做日志，这样可以进一步地提高数据库恢复速度。

######## 2.2 undo

########## 1. 基本概念

重做日志记录了事务的行为，可以很好地通过其对页进行“重做”操作。但是事务有时还需要进行回滚操作，这是就需要undo。因此在对数据库进行修改时，InnoDB存储引擎不但会产生redo，还会产生一定量的undo。这样如果用户执行的事务或语句由于某些原因失败了，又或者用户用一条rollback语句请求回滚，就可以利用这些undo信息将数据回滚到修改之前的样子。

redo存放在重做日志文件中，与redo不同，undo存放在数据库内部的一个特殊段（segment）中，这个段称为undo段（undo segment）。undo段位于共享表空间内。undo是逻辑日志，只是将数据库逻辑地恢复到原来的样子。这是因为在多用户并发系统中，可能会有数十、数百甚至数千个并发事务。数据库的主要任务就是协调对数据记录的并发访问。比如，一个事务在修改当前一个页中某几条记录，同时还有别的事务在同一个页中另几条记录进行修改。因此，不能将一个页回滚到事务开始的样子，因为这样会影响其他事务正在进行的工作。

例如，用户执行了一个INSERT 10W条记录的事务，这个事务会导致分一个新的段，即表空间会增大。在用户执行rollback时，会将插入的事务进行回滚，但是表空间的大小并不会因此而收缩。因此，当InnoDB存储引擎回滚时，它实际上做的是与先前相反的操作。对于每个INSERT，InnoDB存储引擎会完成一个DELETE；对于每个DELETE，InnoDB存储引擎会执行一个INSERT；对于每个UPDATE，InnoDB存储引擎会执行一个相反的UPDATE，将修改前的行改回去。

除了回滚操作，undo的另一个作用就是MVCC，即在InnoDB存储引擎中MVCC的实现是通过undo来完成的。当用户读取一行记录时，若该记录已经被其他事务占用，当前事务可以通过unod读取之前的行版本信息，一次实现非锁定读取。

最后也是最为重要的一点，undo log会产生redo log，也就是undo log的产生会伴随着redo log的产生，这是因为undo log也需要持久性的保护。

########## 2. undo存储管理

InnoDB存储引擎对undo的管理同样采用段的方式。但是这个段和之前介绍的段有所不同。首先InnoDB存储引擎有rollback segment，每个回滚段中记录了1024个undo log segment，而在每个undo log segment段中进行undo页的申请。共享表空间偏移量为5的页(0,5)记录了所有rollback segment header所在的页，这个页的类型为FIL_PAGE_TYPE_SYS。

可通过参数对rollback segment进一步的设置。参数有：

* innodb_undo_directory
* innodb_undo_logs
* innodb_undo_tablespaces

参数 **innodb_undo_directory** 用于设置rollback segment文件所在的路径。意味着rollback segment可以存放在共享表空间以外的位置，即可以设置为独立表空间。该参数的默认值为“.”，表示当前InnoDB存储引擎的目录。

参数 **innodb_undo_logs** 用来设置rollback segment的个数，默认值为128。

参数 **innodb_undo_tablespaces** 用来设置构成rollback segment文件的数量，这样rollback segment可以较为平均的分布在多个文件中。设置该参数后，会在路径innodb_undo_directory看到undo为前缀的文件，该文件就代表rollback segment文件。

需要特别注意的是，事务在undo log segment分配页并写入undo log的这个过程同样需要写入重做日志。当事务提交时，InnoDB存储引擎会做以下两件事情：

* 将undo log放入列表中，供之后的purge操作
* 判断undo log所在的页是否可以重用，若可以分配给下个事务使用

事务提交后并不能马上删除undo log 及 undo log所在的页。这是因为可能还有其他的事务需要通过undo log 来得到行记录之前的版本。故事务提交时将undo log 放入一个链表中，是否可以最终删除 undo log及undo log所在页由purge线程来判断。

########## 3. undo log 格式

在InnoDB存储引擎中，undo log分为：

* insert undo log
* update undo log

**insert undo log** 是指insert操作中产生的undo log。因为insert操作的记录，只对事务本身可见，对其他事务不可见（这是事务隔离性的要求），故该undo log可以在事务提交后直接删除。不需要进行purge操作。

**update undo log** 比insert undo log 记录的内容更多，所需占用的空间也更大。next、start、undo_no、table_id与之前介绍的insert undo log 部分相同。

########## 4. 查看undo信息

######## 2.3 purge

delete和update操作可能并不直接删除原有的数据，把真正的删除操作放在了purge环节中完成。

purge用于最终完成delete和update操作。这样设计是因为InnoDB存储引擎支持MVCC，所以记录不能在事务提交时立即处理。这时其他事务可能正在引用，是否可以删除该条记录通过purge来判断。若该行记录已不被任何其他事务引用，那么就可以进行真正的delete操作。

######## 2.4 group commit

若事务为非只读事务，则每次事务提交时需要进行一次fsync操作，以此保证重做日志都已经写入磁盘。当数据库发生宕机时，可以通过重做日志进行恢复。虽然固态硬盘的出现提高了磁盘的性能，然而磁盘你的fsync性能是有限的。为了提高磁盘fsync的效率，当前数据库都提供了group commit的功能，即一次fsync可以刷新确保多个事务日志被写入文件。

###### 3. 事务控制语句

在MySQL命令行默认的设置下，事务都是自动提交的（auto commit）的，即执行SQL语句后就会马上执行commit操作。因此要显式的开启一个事务需要使用命令begin、start transaction，或者执行 set auto_commit=0，禁用当前会话的自动提交。

* start transaction|begin：显式的开启一个事务
* commit：提交事务，使得已对数据库做的所有修改成为永久性的。
* savepoint identifier：savepoint允许在事务中创建一个保存点，一个事务中可以有多个savepoint。
* release savepoint identifier：删除一个事务的保存点，当没有一个保存点执行这条语句时，会抛出一个异常。
* rollback to [savepoint] identifier：这个语句与savepoint命令一起使用。可以把事务回滚到标记点，而不会滚在此标记之前的任何工作。
* set transaction：这个语句用来设置事务的隔离级别。

commit和commit work都是用来提交事务。不同之处在于commit work用来控制事务结束后的行为是chain还是release。如果是chain，那么事务就变成了了链事务。

用户可以通过参数completion_type来进行控制，该参数默认为0，表示没有任何操作。此时commit 和 commit work是等价的。参数设置为1时，commit work 等同于commit and chain，表示马上自动开启一个相同隔离级别的事务。参数设置为2时，commit work等同于commit and release。在事务提交后会自动断开与服务器的连接。

###### 4. 隐式提交的SQL语句

以下这些SQL语句会产生一个隐式的提交操作，即执行完这些语句后，会有一个隐式的commit操作。

* DDL：alter database...upgrade data directory name, alter event, alter procedure, alter table ,alter view, create database, create event,create index, create procedure, create table, create trigger, create view, drop database, drop event, drop index, drop procedure, drop table, drop trigger, drop view, rename table, truncate table.
* 用来隐式的修改MySQL架构的操作：create user, drop user, grant, rename user, revoke, set password.
* 管理语句：analyze table, cache index, check table, load index into cache, optimize table, repair table.

###### 5. 对于事务操作的统计

由于InnoDB存储引擎是支持事务的，因此InnoDB存储引擎的应用需要在考虑请求数（Question Per Second， QPS）的同时，应该关注每秒事务处理的能力（Transaction Per Second，TPS）。

计算TPS的方法是（com_commit+com_rollback）/ time。但是利用这种方法进行的计算前提是：所有的事务都必须是显式提交的，如果存在隐式提交和回滚（默认autocommit=1），不会计算到com_commit和com_rollback变量中。

###### 6. 事务的隔离级别

SQL标准定义的4个隔离级别为：

* read uncommitted
* read committed
* repeatable read
* serializable

#### 第8章 备份与恢复

###### 1. 备份与恢复概述

可以根据不同的类型来划分备份的方法。根据备份的方法不同可以划分为：

* Hot Backup（热备）
* Cold Backup（冷备）
* Warm Backup（温备）

**Hot Backup** 是指数据库运行中直接备份，对正在运行的数据库操作没有任何的影响。这种方式在MySQL官方手册中称为 Online Backup（在线备份）。**Cold Backup** 是指备份操作在数据库停止的情况下，这种备份最简单，一般只需要复制相关的数据库物理文件即可。这种方式在MySQL官方手册中称为 Offline Backup（离线备份）。**Warm Backup** 备份同样是在数据库运行中进行的，但是会对当前数据库的操作有所影响，如加一个全局读锁以保证备份数据的一致性。

按照备份后文件的内容，备份又可以划分为：

* 逻辑备份
* 裸文件备份

在MySQL数据库中，**逻辑备份** 是指备份出的文件内容是可读的，一般是文本文件。内容一般是由一条条SQL语句，或者是表内实际数据组成。这类方法的好处是可以观察导出文件的内容，一般适用于数据库升级、迁移等工作。但其缺点是恢复需要的时间往往较长。

**裸文件备份** 是指复制数据库的物理文件，既可以是在数据库运行中的复制（如ibbackup、xtrabackup这类工具），也可以是在数据库停止运行时直接的数据文件复制。这类备份的恢复时间往往较逻辑备份短很多。

按照备份数据库的内容来分，备份可以划分为：

* 完全备份
* 增量备份
* 日志备份

**完全备份** 是指对数据库进行一个完整的备份。**增量备份** 是指上次完全备份的基础上，对于更改的数据进行备份。**日志备份** 主要是指对MySQL数据库二进制日志的备份，通过对一个完全备份进行二进制日志的重做（replay）来完成数据库的point-in-time的恢复工作。MySQL数据库复制（replication）的原理就是异步实时的将二进制日志重做传送并应用到从（slave/standby）数据库。

###### 2. 冷备

对于InnoDB存储引擎的冷备非常简单，只需要备份MySQL数据库的 frm 文件，共享空间文件，独立表空间文件（*.ibd），重做日志文件。另外建议定期备份MySQL数据库的配置文件my.cnf，这样有利于恢复的操作。

冷备的优点：

* 备份简单，只要复制相关文件即可。
* 备份文件易于在不同操作系统，不同MySQL版本上进行恢复。
* 恢复相当简单，只需要把文件恢复到指定位置即可。
* 恢复速度快，不需要执行任何SQL语句，也不需要重建索引。

冷备的缺点：

* InnoDB存储引擎冷备的文件通常比逻辑文件大很多，因为表空间中存放这很多其他的数据，如undo段，插入缓冲等信息。
* 冷备也不总是可以轻易的跨平台。操作系统、MySQL版本、文件大小写敏感和浮点数格式都会成为问题。

###### 3. 逻辑备份

######## 3.1 mysqldump

mysqldump的参数选项有很多，可以通过mysqldump --help命令来查看所有的参数，有些参数有缩写形式，如 --lock-tables的缩写形式 -l。这里举例一些比较重要的参数。

* --single-transaction：在备份开始前，先执行start transaction命令，以此来获得备份的一致性，当前该参数只对InnoDB存储引擎有效。当启用该参数并进行备份时，确保米有其他任何的DDL语句执行，因为一致性读并不能隔离DDL操作。
* --lock-tables（-l）：在备份中，以此锁住每个Schema下的所有表。一般用于MyISAM存储引擎，当备份时只能对数据库进行读取操作，不过备份依然可以保证一致性。对于InnoDB存储引擎，不需要使用该参数，用--single-transaction即可。并且--lock-tables和--single-transaction是互斥的，不能同时使用。如果既有MyISAM表又有InnoDB表，就能选择--lock-tables了，--lock-tables选项是依次对每个Schema中的表上锁的，因此只能保证每个Schema下表备份的一致性，而不能保证所有架构下表的一致性。
* --lock-all-tables（-x）：在备份过程中，对所有Schema中的所有表上锁。这个可以避免之前说的--lock-tables参数不能同时锁住所有表的问题。
* --add-drop-database：在create database前先运行drop database。这个参数需要和--all-databases或者--databases选项一起使用。在默认情况下，导出的文本文件中并不会有create database。
* --master-data [=value]：通过该参数产生的备份转存文件主要用来建立一个replication。当value的值为1时，转存文件中记录change master语句。当value的值为2时，change master语句被写出SQL注释。在默认情况下，value的值为空。
* --events（-E）：备份事件调度器。
* --routines（-R）：备份存储过程和函数。
* --triggers：备份触发器。
* --hex-blob：将binary、varbinary、blog和bit列类型备份为16进制的格式。mysqldump导出的文件一般都是文本文件，如果导出的类型中有前面几种，在文本模式下有些字符不可见，若添加--hex-blob选项，结果会以16进制的方式显示。
* --tab=path（-T path）：产生TAB分隔的数据文件。对于每张表，mysqldump创建一个包含create table语句的table_name.sql文件，和包含数据的tbl_name.txt文件。可以使用--fields-terminated-by=...，--fields-enclosed-by=...，--fields-optionally-enclosed-by=...，--fields-escaped-by=...，--lines-terminated-by=...来改变默认的分隔符、换行符。
* --where='where_condition'（-w 'where_conditon'）：导出给定条件的数据。

######## 3.2 select ... into outfile

select...into语句也是一种逻辑备份的方法，更准确的说是导出一张表的数据

######## 3.3 逻辑备份的恢复

mysqldump的恢复操作比较简单，因为导出的备份文件就是SQL语句，只需要执行这个文件就行了。

```mysql
mysql -uroot -p < backup.sql
```

mysqldump不能导出视图，为了保证完全的恢复，在通过mysqldump导出存储过程、触发器、事件、数据后，还需要导出视图的定义或者备份视图定义的 frm 文件，并在恢复时进行导入。

######## 3.4 load data infile

若通过mysqldump-tab或者通过select into outfile导出的数据需要恢复，可以通过命令load data infile来进行导入。要对服务器文件使用load data infile，必须拥有FILE权限。

######## 3.5 mysqlimport

mysqlimport是MySQL数据库提供的一个命令行程序，本质上调用的是load data infile接口。

###### 4. 二进制日志备份与恢复

备份binlog，先通过flush logs生成一个新的binlog，然后再备份之前的binlog。

恢复binlog，通过mysqlbinlog

```mysql
mysqlbinlog binlog.0000001 | mysql -uroot -p
```

###### 5. 热备

######## 5.1 ibbackup

ibbackup是InnoDB存储引擎官方提供的热备工具，可以同时备份MyISAM和InnoDB表。对于InnoDB表其备份的工作原理如下：

1. 记录备份开始时，InnoDB存储引擎重做日志文件检查点的LSN。
2. 复制共享表空间文件以及独立表空间文件。
3. 记录复制完表空间文件后，InnoDB重做日志文件检查点的LSN。
4. 复制在备份时产生的重做日志。

ibbackup优点：

* 在线备份，不阻塞任何的SQL语句。
* 备份性能好，备份的实质是复制数据库文件和重做日志文件。
* 支持压缩备份，通过选项，可以支持不同级别的压缩。
* 跨平台支持，ibbackup可以运行在Linux、Windows以及主流的UNIX系统平台上。

ibbackup对InnoDB的恢复步骤为：

* 恢复表空间文件
* 应用重做日志文件

######## 5.2 XtraBackup

XtraBackup备份工具是由Percona公司开发的开源热备工具。

######## 5.3 XtraBackup实现增量备份

Mysql数据库本身提供的工作并不支持真正的增量备份，准确的说，二进制日志恢复应该是point-in-time的恢复而不是增量备份。而XtraBackup工具支持对于InnoDB存储引擎的增量备份，其工作原理如下：

1. 首先完成一个全备，并记录下此时检查点的LSN。
2. 在进行增量备份时，比较表空间中每个页的LSN是否大于上次备份的LSN，如果是，则备份该页，同时记录当前检查点的LSN。

###### 6. 快照备份

Mysql数据库本身不支持快照功能，快照备份是基于文件系统支持的快照功能对数据进行备份。备份的前提是将所有数据库文件放在同一文件分区中，然后对该分区进行快照操作。

###### 7. 复制

######## 7.1 复制的工作原理

复制（replication）是MySQL数据库提供的一种高可用高性能的解决方案，一般用来建立大型的应用。replication的工作原理分为以下3个步骤：

1. 主服务器（master）把数据更改记录到二进制日志（binlog）中。
2. 从服务器（slave）把主服务器的二进制日志复制到自己的中继日志（relay log）中。
3. 从服务器重做中继日志中的日志，把更改应用到自己的数据库上，以达到数据的最终一致性。

复制的工作原理并不复杂，其实就是一个完全备份加上二进制日志备份还原。不同是这个二进制日志的还原操作基本上实时在进行中，这里特别需要注意的是，复制不是实时的同步，而是异步实时。这中间存在主从服务器之间的执行延时，如果主服务器压力很大，则可能导致主从延时较大。

![](../../../public/assets/images/mysql/Mysql复制工作原理.png)

从服务器有2个线程，一个是I/O线程，负责读取主服务其的二进制日志，并将其保存为中继日志；另一个是SQL线程，复制执行中继日志。

show slave status的主要变量

| 变     量               | 说明                                                                                       |
| --------------------- | ---------------------------------------------------------------------------------------- |
| slave_io_state        | 显示当前IO线程的状态                                                                              |
| master_log_file       | 显示当前同步的主服务器的二进制日志                                                                        |
| read_master_log_pos   | 显示当前同步到主服务器上二进制日志的偏移量位置，单位是字节                                                            |
| relay_master_log_file | 当前中继日志同步的二进制日志                                                                           |
| relay_log_file        | 显示当前写入的中继日志                                                                              |
| relay_log_pos         | 显示当前执行到中继日志的偏移量位置                                                                        |
| slave_io_running      | 从服务器中IO线程的运行状态，YES表示正常                                                                   |
| slave_sql_running     | 从服务器中SQL线程的运行状态，YES表示正常                                                                  |
| exec_master_log_pos   | 表示同步到主服务器的二进制日志偏移量的位置。（Read_Master_Log_Pos - Exec_Master_Log_Pos）可以表示当前SQL线程运行的延时，单位是字节。 |

######## 7.2 快照+复制的备份架构

复制可以用来作为备份，但功能不仅限于备份，其主要功能如下：

* 数据分布。由于Mysql数据库提供的复制并不需要很大的带宽要求，因此可以在不同的数据中心之间实现数据的复制。
* 读取的负载平衡。通过建立多个从服务器，可将读取平均的分布到这些从服务器中，并且减少了主服务器的压力。一般通过DNS的Round-Robin和Linux的LVS功能都可以实现负载平衡。
* 数据库备份。复制对备份很有帮助，但是从服务器不是备份，不能完全替代备份。
* 高可用性和故障转移。通过复制建立的从服务器有助于故障转移，减少故障的停机时间和恢复时间。

可见，复制的设计不是简单用来备份的，并且只是用复制来进行备份是远远不够的。一个比较好的方法是通过对从服务器上的数据库所在的分区做快照，以此来避免误操作对复制造成的影响。当发生在主服务器上的误操作时，只需要将从服务器上的快照进行恢复，然后再根据二进制日志进行point-in-time的恢复即可。

![](../../../public/assets/images/mysql/快照+备份的复制架构.png)

还有一些其他的方法来调整复制，比如采用延时复制，即间歇的开启从服务器上的同步，保证大约1小时的延时。这的确也是一个办法，只是数据库在高峰和非高峰期间每小时产生的二进制日志量是不同的，用户很难精确的控制。另外，这种方法也不能完全起到对误操作的防范作用。

此外，建议在从服务器上启用read-only选项，这样能保证从服务器上的数据仅与主服务器进行同步，避免其他线程修改数据。在启用read-only选项后，如果操作从服务器的用户没有super权限，则对从服务器进行任何的修改会抛出一个错误。

------

## 《MySQL必知必会》

#### 子查询(杂记)

`子查询总是从内向外处理，即先执行子查询，将得到的结果作为外部查询的筛选条件`

#### 联结(连接)

###### 内部联结

等值联结(queijoin)，基于两个表之间的相等测试，这种联结也称为内部联结。

```sql
## 常用写法
select vend_name, prod_name, prod_price from vendors, products where vendors.vend_id = products.vend_id order by vend_name, prod_name;

## 标准写法
select vend_name, prod_name, prod_price from vendors inner join products on vendors.vend_id = products.vend_id order by vend_name, prod_name;
```

`Mark: ANSI SQL规范首选Inner Join语法。此外，尽管使用where子句定义联结的确比较简单，但是使用明确的联结语法能够确保不会忘记联结条件，有时候这样做也能影响性能。`

###### 联结多个表

SQL对一条select语句中可以联结的表的数目没有限制。创建联结的基本规则也相同。

```sql
select prod_name, vend_name, prod_price, quantity from orderitems, products, vendors where products.vend_id = vendors.vend_id and orderitems.prod_id = products.prod_id and order_num = 20005;

+----------------+-------------+------------+----------+
| prod_name      | vend_name   | prod_price | quantity |
+----------------+-------------+------------+----------+
| .5 ton anvil   | Anvils R Us |       5.99 |       10 |
| 1 ton anvil    | Anvils R Us |       9.99 |        3 |
| TNT (5 sticks) | ACME        |      10.00 |        5 |
| Bird seed      | ACME        |      10.00 |        1 |
+----------------+-------------+------------+----------+
4 rows in set (0.00 sec)
```

`Mark：Mysql在运行时关联指定的每个表以处理联结。这种处理可能是非常耗费资源的，因此应该仔细，不要联结不必要的表。联结的表越多，性能下降越厉害。`

有时候子查询并不总是执行复杂select操作的最有效的方法

```sql
select cust_name, cust_contact 
from customers 
where cust_id in (
    select cust_id 
    from orders 
    where order_num in (
        select order_num 
        from orderitems 
        where prod_id = 'TNT2'));
## ===>
select cust_name, cust_contatct
from customers, orders, orderitems
where customers.cust_id = order.cust_id
    and orderitems.order_num = orders.order_num
    and prod_id = 'TNT2';
```

###### 自联结

```sql
select prod_id, prod_name
from products
where vend_id = (select vend_id
                from products
                where prod_id = 'DTNTR');
## ===>
select p1.prod_id, p1.prod_name
from products as p1, products as p2
where p1.vend_id = p2.vend_id
and p2.prod_id = 'DTNTR';
```

`Mark：自联结通常作为外部语句用来替代从相同表中检索数据时使用的子查询语句。虽然最终的结果是相同的，但有时候处理联结远比处理子查询快得多。应该试一下两种方法，以确定哪一种的性能更好。`

###### 自然联结

无论何时对表进行联结，应该至少有一个列出现在不止一个表中(被联结的列)。标注你的联结返回所有数据，甚至相同的列多次出现。自然联结排除多次出现，使每个列只返回一次。自然联结是这样一种联结，其中你只能选择那些唯一的列。这一般是通过对表使用通配符(select *)，对所有其他表的列使用明确的子集来完成的。

```sql
select c.*, o.order_num, o.order_date,
       oi.prod_id, oi.quantity, OI.item_price
from customer as c, order as o, orderitems as oi
where c.cust_id = o.cust_id
and oi.order_num = o.order_num
and prod_id = 'FB';
```

在这个例子中，通配符只对第一个表使用。所有其他列明确列出，所以没有重复的列被检索出来。

事实上，迄今为止我们建立的每个内部联结都是自然联结，很可能我们永远都不会用到不是自然联结的内部联结。

###### 外部联结

许多联结将一个表中的行与另一个表中行相关联。但有时候会需要包含没有关联行的那些行，这种类型的联结称为外部联结。

```sql
## 内部联结，仅检索出匹配条件的记录
select customers.cust_id, orders.order_num
from customers inner join orders
on customers.cust_id = orders.cust_id;
+---------+-----------+
| cust_id | order_num |
+---------+-----------+
|   10001 |     20005 |
|   10001 |     20009 |
|   10003 |     20006 |
|   10004 |     20007 |
|   10005 |     20008 |
+---------+-----------+
5 rows in set (0.00 sec)

## 外部联结，可以检索出未匹配上的记录
select customers.cust_id, orders.order_num
from customers left outer join orders
on customers.cust_id = orders.cust_id;
+---------+-----------+
| cust_id | order_num |
+---------+-----------+
|   10001 |     20005 |
|   10001 |     20009 |
|   10002 |      NULL |
|   10003 |     20006 |
|   10004 |     20007 |
|   10005 |     20008 |
+---------+-----------+
6 rows in set (0.00 sec)
```

在使用外部联结**outer join**语法时，必须使用**right**或**left**关键字指定包括其所有行的表(right指出的是outer join 右边的表，而left指出的是outer join左边的表)

#### 组合查询

多数SQL查询都只包含从一个或多个表中返回数据的单条select语句。Mysql允许执行多个查询(多条select语句)，并将结果作为单个查询结果集返回。这些组合查询通常称为并(Union)或复合查询(compound query)。

有两种情况，其中需要使用组合查询：

* 在单个查询中从不同的表返回类似结构的数据；
* 对单个表执行多个查询，按单个查询返回数据。

举例：假如需要价格小于等于5的所有物品的一个列表，而且还想包括供应商1001和1002生产的所有物品(不考虑价格)。

```sql
## 单条SQL
select vend_id, prod_id, prod_price from products where prod_price <= 5;
+---------+---------+------------+
| vend_id | prod_id | prod_price |
+---------+---------+------------+
|    1003 | FC      |       2.50 |
|    1002 | FU1     |       3.42 |
|    1003 | SLING   |       4.49 |
|    1003 | TNT1    |       2.50 |
+---------+---------+------------+
4 rows in set (0.00 sec)
select vend_id, prod_id, prod_price from products where vend_id in (1001, 1002);
+---------+---------+------------+
| vend_id | prod_id | prod_price |
+---------+---------+------------+
|    1001 | ANV01   |       5.99 |
|    1001 | ANV02   |       9.99 |
|    1001 | ANV03   |      14.99 |
|    1002 | FU1     |       3.42 |
|    1002 | OL1     |       8.99 |
+---------+---------+------------+
5 rows in set (0.00 sec)

## 组合查询
select vend_id, prod_id, prod_price from products where prod_price <= 5
union
select vend_id, prod_id, prod_price from products where vend_id in (1001, 1002);
+---------+---------+------------+
| vend_id | prod_id | prod_price |
+---------+---------+------------+
|    1003 | FC      |       2.50 |
|    1002 | FU1     |       3.42 |
|    1003 | SLING   |       4.49 |
|    1003 | TNT1    |       2.50 |
|    1001 | ANV01   |       5.99 |
|    1001 | ANV02   |       9.99 |
|    1001 | ANV03   |      14.99 |
|    1002 | OL1     |       8.99 |
+---------+---------+------------+
8 rows in set (0.00 sec)

## 作为参考，列出使用多条where子句而不是使用union的相同查询
select vend_id, prod_id, prod_price 
from products
where prod_price < =5
or vend_id in (1001, 1002);
```

此例中，使用union可能比使用where子句更为复杂，但对于更复杂的过滤条件，或者从多个表(而不是单个表)中检索数据的情形，使用union可能会使处理更简单。

###### union规则

* union必须由两条或两条以上的select语句组成，语句之间用关键字union分隔
* union中的每个查询必须包含相同的列、表达式或聚集函数
* 列数据类型必须兼容：类型不必完全相同，但必须是DBMS可以隐含地转换的类型

从前面的例子中，可以看出来，第一条SQL查询结果是4条记录，第二条SQL查询结果是5条记录，而使用了union关键字之后，返回了8条记录而不是9条。union从查询结果集中自动去重了，这是union的默认行为，但是如果想要返回所有匹配的行，可使用union all而不是union

```sql
select vend_id, prod_id, prod_price from products where prod_price <= 5
union all
select vend_id, prod_id, prod_price from products where vend_id in (1001, 1002);
```

`Mark：union几乎总是完成与多个where条件相同的工作，但是union all完成了where完成不了的工作，如果确实需要需要每个条件的匹配行全部出现(包含重复)，则必须使用union all。`

#### 全文本检索(全文索引)

> 想使用全文本检索功能，表所使用的存储引擎必须是MyISAM

使用全文本索引的语法要求：

```sql
Syntax:
MATCH (col1,col2,...) AGAINST (expr [search_modifier])

select note_text from productnotes where Match(note_text) Against('rabbit');
+----------------------------------------------------------------------------------------------------------------------+
| note_text                                                                                                            |
+----------------------------------------------------------------------------------------------------------------------+
| Customer complaint: rabbit has been able to detect trap, food apparently less effective now.                         |
| Quantity varies, sold by the sack load.
All guaranteed to be bright and orange, and suitable for use as rabbit bait. |
+----------------------------------------------------------------------------------------------------------------------+
```

使用Match()和Against()执行全文本搜索，其中Match()指定被搜索的列，Against()指定要使用的搜索表达式。传递给Match()的值必须与Fulltext()定义中的相同，如果指定多个列，则必须列出它们(而且次序正确)。

上面的例子也可以用Like语句改写

```sql
select note_text
from productnotes
where note_text like '%rabbit%';
+----------------------------------------------------------------------------------------------------------------------+
| note_text                                                                                                            |
+----------------------------------------------------------------------------------------------------------------------+
| Quantity varies, sold by the sack load.
All guaranteed to be bright and orange, and suitable for use as rabbit bait. |
| Customer complaint: rabbit has been able to detect trap, food apparently less effective now.                         |
+----------------------------------------------------------------------------------------------------------------------+
2 rows in set (0.00 sec)
```

改写后的like子句，同样可以检索出两行，但次序不同(虽然并不总是出现这种情况)。

上述两条select语句都不包含order by子句。后者以不特别有用的顺序返回数据。前者(使用全文本搜索)返回以文本匹配的良好程度排序的数据。两个行都包含此词rabbit，但包含词rabbit作为第3个词的行的等级比作为第20个词的行高。这很重要。全文本搜索的一个重要部分就是对结果排序。具有较高等级的行先返回(因为这些行很可能是你真正想要的行)。

```sql
select note_text,
       Match(note_text) Against('rabbit') as rank
from productnotes;

+-----------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
| note_text                                                                                                                                                 | rank               |
+-----------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
| Customer complaint:
Sticks not individually wrapped, too easy to mistakenly detonate all at once.
Recommend individual wrapping.                          |                  0 |
| Can shipped full, refills not available.
Need to order new can if refill needed.                                                                          |                  0 |
| Safe is combination locked, combination not provided with safe.
This is rarely a problem as safes are typically blown up or dropped by customers.         |                  0 |
| Quantity varies, sold by the sack load.
All guaranteed to be bright and orange, and suitable for use as rabbit bait.                                      | 1.5905543565750122 |
| Included fuses are short and have been known to detonate too quickly for some customers.
Longer fuses are available (item FU1) and should be recommended. |                  0 |
| Matches not included, recommend purchase of matches or detonator (item DTNTR).                                                                            |                  0 |
| Please note that no returns will be accepted if safe opened using explosives.                                                                             |                  0 |
| Multiple customer returns, anvils failing to drop fast enough or falling backwards on purchaser. Recommend that customer considers using heavier anvils.  |                  0 |
| Item is extremely heavy. Designed for dropping, not recommended for use with slings, ropes, pulleys, or tightropes.                                       |                  0 |
| Customer complaint: rabbit has been able to detect trap, food apparently less effective now.                                                              | 1.6408053636550903 |
| Shipped unassembled, requires common tools (including oversized hammer).                                                                                  |                  0 |
| Customer complaint:
Circular hole in safe floor can apparently be easily cut with handsaw.                                                                |                  0 |
| Customer complaint:
Not heavy enough to generate flying stars around head of victim. If being purchased for dropping, recommend ANV02 or ANV03 instead.   |                  0 |
| Call from individual trapped in safe plummeting to the ground, suggests an escape hatch be added.
Comment forwarded to vendor.                            |                  0 |
+-----------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
14 rows in set (0.00 sec)
```

这里，在select而不是where子句中使用Match()和Against()。这使所有行都被返回(因为没有where子句)。Match()和Against()用来建立一个计算列(别名为rank)，此列包含全文本搜索计算出的等级值。等级由Mysql根据行中词的数目、唯一词的数目、整个索引中词的总数以及包含该词的行的数目计算出来。正如所见，不包含词rabbit的行等级为0。确实包含词rabbit的两个行每行都有一个等级值，文本中词靠前的行的等级值比词靠后的行的等级值高。

这个例子有助于说明全文本搜索如何排除行(排除那些等级为0的行)，如何排序结果(按等级以降序排序)。

`Mark：如果指定多个搜索项，则包含多数匹配词的那些行将具有比包含较少词(或仅有一个匹配)的那些行高的等级值`

通过对比执行计划，全文本搜索比Like的速度要快得多

```sql
mysql> explain select note_text from productnotes where note_text like '%rabb
it%'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: productnotes
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 14
     filtered: 11.11
        Extra: Using where
1 row in set, 1 warning (0.00 sec)

mysql> explain select note_text from productnotes where Match(note_text) Against('rabbit')\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: productnotes
   partitions: NULL
         type: fulltext
possible_keys: note_text
          key: note_text
      key_len: 0
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)

mysql>
```

全文本搜索仅检索了一行，就找到了需要的行，使用Like则是全表扫描，当然这没有可比性，一个是通过where顾虑，一个是走索引，但是这种涉及到文本检索的情况，足以证明全文索引能加快检索速度。

###### 查询扩展

查询扩展用来设法放宽所返回的全文本搜索结果的范围。考虑下面的情况。你想找出所有提到anvils的注释。只有一个注释包含anvils，但你还想找出可能与你搜索有关的其他行，即使它们不包含词anvils。

在使用查询扩展时，MySQL对数据和索引进行两边扫描来完成搜索：

* 首先，进行一个基本的全文本搜索，找出与搜索条件匹配的所有行；
* 其次，MySQL检查这些匹配行并选择所有有用的词；
* 而其次，MySQL再进行全文本搜索，这次不仅使用原来的条件，而且还使用所有有用的词。

```sql
select note_text from productnotes where match(note_text) against('anvils');
+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| note_text                                                                                                                                                |
+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| Multiple customer returns, anvils failing to drop fast enough or falling backwards on purchaser. Recommend that customer considers using heavier anvils. |
+----------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

select note_text from productnotes
where match(note_text) against('anvils' with query expansion);
+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| note_text                                                                                                                                                |
+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| Multiple customer returns, anvils failing to drop fast enough or falling backwards on purchaser. Recommend that customer considers using heavier anvils. |
| Customer complaint:
Sticks not individually wrapped, too easy to mistakenly detonate all at once.
Recommend individual wrapping.                         |
| Customer complaint:
Not heavy enough to generate flying stars around head of victim. If being purchased for dropping, recommend ANV02 or ANV03 instead.  |
| Please note that no returns will be accepted if safe opened using explosives.                                                                            |
| Customer complaint: rabbit has been able to detect trap, food apparently less effective now.                                                             |
| Customer complaint:
Circular hole in safe floor can apparently be easily cut with handsaw.                                                               |
| Matches not included, recommend purchase of matches or detonator (item DTNTR).                                                                           |
+----------------------------------------------------------------------------------------------------------------------------------------------------------+
7 rows in set (0.00 sec)
```

这次返回了7行。第一行包含词anvils，因此等级最高。第二行与anvils无关，但因为它包含第一行中的两个词（ customer和recommend），所以也被检索出来。第3行也包含这两个相同的词，但它们在文本中的位置更靠后且分开得更远，因此也包含这一行，但等级为第三。第三行确实也没有涉及anvils（按它们的产品名）。  

###### 布尔文本搜索

MySQL支持全文本搜索的另外一种形式，称为布尔方式(boolean mode)。以布尔方式，可以提供关于如下内容的细节：

* 要匹配的词；
* 要排斥的词(如果某行包含这个词，则不返回该行，即使它包含其他指定的词也是如此)；
* 排列提示(指定某些词比其他词重要，更要的的词等级更高)；
* 表达式分组；
* 另外一些内容。

`Mark： 布尔方式不同于迄今为止使用的全文本搜索语句的地方在于，即使没有定义fulltext索引，也可以使用它。但这是一种非常缓慢的操作(其性能将随着数据量的增加而降低)`

```sql
select note_text from productnotes
where match(note_text) against('heavy' in boolean mode);

+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| note_text                                                                                                                                               |
+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| Item is extremely heavy. Designed for dropping, not recommended for use with slings, ropes, pulleys, or tightropes.                                     |
| Customer complaint:
Not heavy enough to generate flying stars around head of victim. If being purchased for dropping, recommend ANV02 or ANV03 instead. |
+---------------------------------------------------------------------------------------------------------------------------------------------------------+
2 rows in set (0.00 sec)
```

此全文本搜索检索包含词heavy的所有行。其中使用了关键字in boolean mode，但实际上没有指定布尔操作符，因此其结果与没有指定布尔方式的结果相同。

`Mark：虽然这个例子的结果与没有in boolean mode的相同，但其行为有一个重要的差别`

为了匹配包含heavy但不包含以rope开始的词的行

```sql
select note_text from productnotest
where match(note_text) against('heavy -rope*' in boolean mode);

+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| note_text                                                                                                                                               |
+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| Customer complaint:
Not heavy enough to generate flying stars around head of victim. If being purchased for dropping, recommend ANV02 or ANV03 instead. |
+---------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

使用-rope\*明确地只是MySQL排除包含rope\*的行。

全文本布尔操作符

| 布尔操作符 | 说明                                    |
| ----- | ------------------------------------- |
| +     | 包含，词必须存在                              |
| -     | 排除，词必须不存在                             |
| >     | 包含，而且增加等级值                            |
| <     | 包含，且减少等级值                             |
| ()    | 把词组成子表达式(允许这些子表达式作为一个组被包含、排除、排列等)     |
| ~     | 取消一个词的排序值                             |
| *     | 词尾的通配符                                |
| ""    | 定义一个短语(与单个词的列表不一样，它匹配整个短语以便包含或排除这个短语) |

```sql
select note_text from productnotes
where match(note_text) against('+rabbit +bait' in boolean mode);
## 这个搜索匹配包含词rabbit和bait的行

select note_text from productnotes
where match(note_text) against('rabbit bait' in boolean mode);
## 没有指定操作符，这个搜索匹配包含rabbit和bait中的至少一个词的行

select note_text from productnotes
where match(note_text) against('"rabbit bait"' in boolean mode);
## 匹配短语rabbit abit而不是匹配两个词

select note_text from productnotes
where match(note_text) against('>rabbit <carrot' in boolean mode);
## 匹配rabbit和carrot，增加前者等级，降低后者等级

select note_text from productnotes
where match(note_text) against('+safe +(<combination)' in boolean mode);
## 匹配词safe和combination，降低后者的等级。
```

`Mark：在布尔方式中，不按等级值降序排序返回的行`

关于全文本搜搜的某些重要说明：

* 在索引全文本数据时，短词被忽略且从索引中排除。短词定义为那些具有3个或3个以下字符的词(如果需要，这个数值可以更改)。
* MySQL带有一个内建的非用词(stopword)列表，这些词在索引全文本数据时总是被忽略。如果需要，可以覆盖这个列表。
* 许多词出现的频率很高，搜索它们没有用处(返回太多结果)。因此，MySQL规定了一条50%规则，如果一个词出现在50%以上的行中，则将它作为一个非用词忽略。50%规则不用于In Boolean Mode。
* 如果表中的行数少于3行，则全文本搜索不返回结果(因为每个词或者不出现，或者至少出现在50%的行中)。
* 忽略词中的单引号。例如，don't 索引为 dont。
* 不具有词分隔符的语言不能恰当地返回全文本搜索结果。
* 如前所述，仅在MyISAM引擎中支持全文本搜索。

#### 插入数据

提高整体性能：

数据库经常被多个客户访问，对处理什么请求以及用什么次序处理进行管理是MySQL的任务。Insert操作可能很耗时(特别是有很多索引需要更新时)，而且它可能降低等待处理的select语句的性能。如果数据检索是最重要的，则可以通过在insert和into之间添加关键字low_priority，指示MySQL降低Insert语句的优先级，insert low_priority into，此方法也适用于update和delete。

单条Insert语句处理多个插入比使用多条Insert语句快。

#### 更新和删除数据

如果用Update语句更新多行时，并且在更新这些行中的一行或多行时出现一个错误，则整个Update操作被取消(错误发生前更新的所有行被恢复到它们原来的值)。为即使是发生错误，也继续进行更新，可以使用ignore关键字，update ignore 表名 set 列名=修改值 where 列名=条件

delete 删除表中所有记录的性能不如truncate，是因为truncate是直接将表删除然后重新创建一张空表，而delete是逐行删除记录。

#### 视图

视图本身不包含数据，查询视图返回的数据都是从其他表汇总检索出来的，如果表中的数据进行过修改，视图将返回改变过的数据。

**性能问题**：因为视图不包含数据，所以每次查询视图时，都必须处理查询所需要的任一个检索。如果用了多个联结和过滤创建了复杂的视图或者嵌套了视图，可能会发现性能下降得很厉害。

视图的规则和限制：

* 与表一样，必须唯一命名。
* 对于可以创建的视图数目没有限制。
* 为了创建视图，必须具有足够的访问权限。这些限制通常由DBA授予。
* 视图可以嵌套，即可以利用从其他视图中检索数据的查询来构造一个视图。
* order by可以用在视图中，但如果从该视图检索数据select中页含有order by，那么该视图中的order by将被覆盖。
* 视图不能索引，也不能有关联的触发器或默认值。
* 视图可以和表一起使用。

更新视图，即更新其基表，对视图的增删改都是对其基表进行增删改。但是，并非所有的视图都是可更新的。基本上可以说，如果MySQL不能正确地确定被更新的基数据，则不允许更新(包括插入和更新)。这实际上意味着，如果视图定义中有以下操作，则不能进行视图的更新：

* 分组(使用group by 和 having)
* 联结
* 子查询
* 并
* 聚集函数
* distinct
* 导出(计算)列

#### 存储过程

变量 (Variable) 内存中一个特定的位置，用来临时存储数据。

```sql
delimiter //
create procedure productpricing(
    OUT pl decimal(8,2),
    OUT ph decimal(8,2),
    OUT pa decimal(8,2)
)
begin
    select min(prod_price) into pl from products;
    select max(prod_price) into ph from products;
    select avg(prod_price) into pa from products;
end //
delimiter ;
```

此存储过程接受3个参数：pl存储产品最低价格，ph存储产品最高价格，pa存储产品平均价格。每个参数必须具有指定的类型，这里使用十进制。关键字 **OUT** 指出相应的参数用来从存储过程传出一个值(返回给调用者)。MySQL支持 **IN** (传递给存储过程)、**OUT** (从存储过程传出)、**INOUT** (对存储过程传入和传出) 类型的参数。存储过程的代码位于begin和end语句内。

为调用此存储过程，必须指定3个变量名

```sql
call productpricing(@pricelow,
                    @pricehigh,
                    @priceaverage);
## 所有的变量都必须以 @ 开始
## 执行完存储过程后，就可以直接使用变量
select @pricehigh, @pricelow, @priceaverage;
+------------+-----------+---------------+
| @pricehigh | @pricelow | @priceaverage |
+------------+-----------+---------------+
|      55.00 |      2.50 |         16.13 |
+------------+-----------+---------------+
1 row in set (0.00 sec)
```

下面再举一个例子

```sql
delimiter //
create procedure ordertotal(
    IN onumber int,
    OUT ototal decimal(8,2)
)
begin
    select sum(item_price*quantity)
    from orderitems
    where order_num = onumber
    into ototal;
end //
delimiter ;

call ordertotal(20005, @total);

select @ototal;
+--------+
| @total |
+--------+
| 149.87 |
+--------+
1 row in set (0.00 sec)
```

一个较为复杂的存储过程

```sql
delimiter //
-- Name: ordertotal
-- Parameters: onumber = order number
--                taxable = 0 if not taxable, 1 if taxable
--                ototal = order total variable
create procedure ordertotal(
    IN onumber INT,
    IN taxable BOOLEAN,
    OUT ototal DECIMAL(8,2)
) comment 'Obtain order total, optionally adding tax'
begin


    -- declare variable for total
    declare total decimal(8,2);
    -- declare tax percentage
    declare taxrate int default 6;

    -- get the order total
    select sum(item_price*quantity)
    from orderitems
    where order_num = onumber
    into total;

    -- is this taxable?
    if taxable then
        -- yes, so add taxrate to the total
        select total+(total/100*taxrate) into total;
    end if;

    -- and finally, save to out variable
    select total into ototal;
end //
delimiter ;

call ordertotal(20005, 0, @total);
Query OK, 1 row affected (0.00 sec)

mysql> select @total;
+--------+
| @total |
+--------+
| 149.87 |
+--------+
1 row in set (0.00 sec)

call ordertotal(20005, 1, @total);
Query OK, 1 row affected (0.00 sec)

mysql> select @total;
+--------+
| @total |
+--------+
| 158.86 |
+--------+
1 row in set (0.00 sec)
```

#### 游标

使用游标涉及几个明确的步骤。

* 在能够使用游标前，必须声明(定义)它。这个过程实际上没有检索数据，它只是定义要使用的select语句。
* 一旦声明后，必须打开游标以供使用。这个过程用前面定义的select语句把数据实际检索出来。
* 对于填有数据的游标，根据需要取出(检索)各行。
* 在结束游标使用时，必须关闭游标。

在声明游标后，可根据需要频繁地打开和关闭游标。在游标打开后，可根据需要频繁地执行取操作。

创建游标：用declare语句创建。declare命名游标，并定义相应的select语句，根据需要带where和其他子句。

```sql
create procedure processorders()
begin
    declare ordernumbers cursor
    for
    select order_num from orders;
end
```

这个存储过程用declare语句定义和命名游标，这里为ordernumbers。存储过程处理完成后，游标就消失（因为它局限于存储过程）。在定义游标之后，可以打开它。

打开游标：open cursor

```sql
open ordernumbers;
```

在处理open语句时执行查询，存储检索出的数据以供浏览和滚动。

游标使用完之后，使用如下语句关闭游标：

```sql
close cursor;
close ordernumbers;
```

close释放游标使用的所有内部内存和资源，因此在每个游标不再需要时都应该关闭。

在一个游标关闭后，如果没有重新打开，则不能使用它。但是，使用声明过的游标不需要再次声明，用open语句打开就可以了。

`Mark：如果你不明确关闭游标，MySQL将会在到达END语句时自动关闭它。`

修改前面一个例子：

```sql
create procedure processorders()
begin
    -- declare the cursor
    declare ordernumbers cursor
    for
    select order_num from orders;

    -- open the cursor
    open ordernumbers;

    -- close the cursor
    close ordernumbers;
end
```

使用游标数据：

```sql
create procedure processorders()
begin
    declare o int;
    declare ordernumbers cursor
    for
    select order_num from orders;
    open ordernumbers;
    fetch ordernumbers into o;
    close ordernumbers;
end
```

使用fetch用来检索当前行的order_num列（将自动从第一行开始）到一个名为o的局部声明变量中。对检索出的数据不做任何处理。

```sql
delimiter //
create procedure processorders()
begin
    declare done boolean default 0;
    declare o int;
    declare ordernumbers cursor
    for
    select order_num from orders;
    declare continue handler for sqlstate '02000' set done = 1;
    open ordernumbers;
    repeat
        fetch ordernumbers into o;
    until done end repeat;
    close ordernumbers;
end //
delimiter ;
```

为了进一步把这些内容组织起来，这里给出更详细的版本

```sql
delimiter //
create procedure processorders()
begin
    declare done boolean default 0;
    declare o int;
    declare t decimal(8,2);

    declare ordernumbers cursor
    for
    select order_num from orders;
    declare continue handler for sqlstate '02000' set done = 1;

    create table if not exists ordertotals(
        order_num int,
        total decimal(8,2)
    );

    open ordernumbers;
    repeat
        fetch ordernumbers into o;
        call ordertotal(o, 1, t);
        insert into ordertotals(order_num, total)
        values(o, t);
        until done end repeat;
    close ordernumbers;
end //
delimiter ;
```

这个例子中，增加了一个变量t，存储每个订单的合计。此外，还创建了一个新表，名为ordertotals。这个表将保存存储过程生成的结果。fetch像以前一样取每个order_num，然后用call执行另一个存储过程来计算每个订单的带税的合计。最后用insert保存每个订单的订单号和合计。

此存储过程不返回数据，但它能够创建和填充另一个表，可以用一条简单的select语句查看该表。

#### 触发器

###### 创建触发器

* 唯一的触发器名
* 触发器关联的表
* 触发器应该响应的活动(delete、insert或update)
* 触发器何时执行

```sql
create trigger newproduct after insert on products
for each row select 'product added';
```

在每次insert products表后显示一次'product added'；为什么是每一行都要显示，是因为使用了for each row。

`Mark：只有表才支持触发器，视图不支持（临时表也不支持）`

单一的触发器不能与多个事件或多个表关联，所以，如果你需要一个对insert和update操作执行的触发器，则应该定义两个触发器。

`Mark：如果before触发器失败，则MySQL将不执行请求的操作。此外，如果before触发器或语句本身失败，MySQL将不执行after触发器（如果有的话）`

###### 删除触发器

```sql
drop trigger newproduct;
```

触发器不能更新或覆盖。为了修改一个触发器，必须先删除它，然后再重新创建。

###### 使用触发器

######## Insert触发器

insert触发器在执行insert语句之前或之后执行。需要知道以下几点：

* 在insert触发器代码内，可引用一个名为new的虚拟表，访问被插入的行；
* 在before insert触发器中，new中的值也可以被更新（允许更改被插入的值）；
* 对于auto_increment列，new在insert执行之前包含0，在inset执行之后包含新的自动生成值。

```sql
create trigger neworder after insert on orders
for each row select new.order_num;
```

此代码创建一个名为neworder的触发器，按照after insert on orders 执行。触发器从new.order_num取得这个值并返回它。此触发器必须按照after insert 执行，因为在before insert语句执行之前，新order_num还没有生成。对于orders的每次插入使用这个触发器总是返回新的订单号。

`Mark： Before or After？ 通常，将before用于数据验证和净化（目的是保证插入表中的数据确实是需要的数据）`

######## delete触发器

delete触发器在delete语句执行之前或之后执行，需要知道以下两点：

* 在delete触发器代码内，你可以引用一个名为old的虚拟表，访问被删除的行；
* old中的值全都是只读的，不能更新。

```sql
delimiter //
create trigger deleteorder before delete on orders
for each row
begin
    insert into archive_orders(order_num, order_date, cust_id)
    values(old.order_num, old.order_date, old.cust_id);
end //
delimiter ;
```

在任意订单被删除前执行此触发器。使用一条insert语句将old中的值保存到一个名为archive_orders的存档表中。

使用before delete触发器的有点（相对于after delete触发器来说），如果由于某种原因，订单不能存档，delete本身将被放弃。

`Mark：触发器deleteorder使用begin和end语句标记触发体。在这个例子中不是必须的，不过也没有害处。使用begin end块的好处是触发器能容纳多条SQL语句`

######## update触发器

update触发器在update语句执行之前或之后执行。需要知道以下几点：

* 在update触发器代码中，可以引用一个名为old的虚拟表访问以前（update语句前）的值，引用一个名为new的虚拟表访问新更新的值；
* 在before update触发器中，new中的值可能也被更新（允许更改将要用于update语句中的值）；
* old中的值全都是只读，不能更新。

```sql
create trigger updateoder before update on orders
for each row set new.vend_state = Upper(new.vend_state);
```

在每次更新之前都对字段进行大写处理

#### 管理事务

事务处理是一种机制，用来管理必须成批执行的MySQL操作，以保证数据不包含不完整的操作结果。利用事务处理，可以保证一组操作不会中途停止，它们或者作为整体执行，或者完全不执行。如果没有错误发生，整租语句提交给数据库表。如果发生错误，则进行回退以恢复数据库到某个已知且安全的状态。

在使用事务和事务处理时，有几个需要知道术语：

* 事务（transaction）指一组SQL语句；
* 回退（rollback）撤销指定SQL语句的过程；
* 提交（commit）将未存储的SQL语句结果写入数据库表；
* 保留点（savepoint）事务处理中设置的临时占位符（placeholder），你可以对它发布回退（与回退整个事务处理不同）。

```sql
## 标记一个事务的开始
start transaction;

## 回退
rollback；

## 事务回退仅支持DML语句，即 insert update delete
## 但是DDL语句，是立即生效机制，所以不会回退

## 提交
commit;

## 保留点
savepoint delete1;
## 回退到保留点
rollback delete1;
```

简单的rollback和commit语句就可以写入或撤销整个事务处理。但是，只是对简单的事务处理才这样做，更复杂的事务可能需要部分提交或回退。

为了支持回退部分事务，必须在事务处理块中合适的位置放置占位符。这样，如果要回退，可以回退到某个占位符。这些占位符称为保留点。

`Mark：保留点越多越好？可以在MySQL代码中设置任意多的保留点，越多越好。因为保留点越多，你就越能按自己的意愿灵活进行回退`

`Mark：释放保留点   保留点在事务处理完成（执行一条rollback或commit）后自动释放。也可以使用release savepoint明确的释放保留点`

正常情况下，MySQL默认设置为自动提交模式，可以通过设置autocommit值来动态修改是否自动提交。1：开启自动提交，0：关闭自动提交。

#### 字符集

字符集的继承关系：

字段继承--->表继承--->库

简而言之，创建库的时候默认不设置字符集，则使用服务器配置文件中的默认值

创建表的时候不设置字符集，则使用数据库配置的字符集

创建表中的字段时，不单独设置字符集，则使用表的字符集

#### 权限

| 权限                      | 说明                                                                  |
| ----------------------- | ------------------------------------------------------------------- |
| all                     | 除grant option外的所有权限                                                 |
| alter                   | 使用alter table                                                       |
| alter routine           | 使用alter procedure和drop procedure                                    |
| create                  | 使用create table                                                      |
| create routine          | 使用create procedure                                                  |
| create temporary tables | 使用create temporary table                                            |
| create user             | 使用create user、drop user、rename user和revoke all privileges           |
| create view             | 使用create view                                                       |
| delete                  | 使用delete                                                            |
| drop                    | 使用drop table                                                        |
| execute                 | 使用call和存储过程                                                         |
| file                    | 使用select into outfile和load data infile                              |
| grant option            | 使用grant和revoke                                                      |
| index                   | 使用create index和drop index                                           |
| insert                  | 使用insert                                                            |
| lock tables             | 使用lock tables                                                       |
| process                 | 使用 show full processlist                                            |
| reload                  | 使用flush                                                             |
| replication client      | 服务位置的访问                                                             |
| replication slave       | 由复制从属使用                                                             |
| select                  | 使用select                                                            |
| show databases          | 使用show databases                                                    |
| show view               | 使用show create view                                                  |
| shutdown                | 使用mysqladmin shutdown(用来关闭MySQL)                                    |
| super                   | 使用change master、kill、logs、purge、master和set global。还允许mysqladmin调试登录 |
| update                  | 使用update                                                            |
| usage                   | 无访问权限                                                               |

`未来的授权：在使用grant和revoke时，用户账号必须存在，但对所涉及的对象没有这个要求。这允许管理在创建数据库和表之前设计和实现安全措施。这样做的副作用是，当某个数据库或表被删除时(用Drop语句)，相关的访问权限依然存在。而且，如果将来重建这些库表，这些访问权限依然生效`

#### 数据库维护

`首先刷新未写数据：为了保证所有数据被写到磁盘（包括索引数据），可能需要在进行备份前使用flush tables语句`

```sql
## analyze用来检查表键是否正确
analyze table orders;

## check table用来针对许多问题对表进行检查
check table orders, orderitems;

## 如果MyISAM表访问产生不正确和不一致的结果，需要用repair table来修复相应的表
repair table orders;

## 如果从一个表中删除大量数据，应该使用optimize table来回收所用的空间，从而优化表的性能
optimize table orders;
```

#### 数据类型整理

字符串

| 数据类型       | 说明                                                            |
| ---------- | ------------------------------------------------------------- |
| char       | 1~255个字符定长字符串。长度必须在创建时指定，否则MySQL假定为char(1)                    |
| enum       | 接受最多64k个字符串组成的一个预定义结合的某个字符串                                   |
| longtext   | 与text相同，但最大长度为4GB                                             |
| mediumtext | 与text相同，但最大长度为16K                                             |
| set        | 接受最多64个字符串组成的一个预定义集合的零个或多个字符串                                 |
| text       | 最大长度为64K的变长文本                                                 |
| tinytext   | 与text相同，但最大长度为255字节                                           |
| varchar    | 长度可变，最多不超过255字节。如果在创建时指定为varchar(n)，则可存储0~n个字符的变长字符串（其中n≤255） |

数值

| 数据类型         | 说明                                                                                  |
| ------------ | ----------------------------------------------------------------------------------- |
| bit          | 位字段，1~64位                                                                           |
| bigint       | 整数值，支持-8223372036854775808~9223372036854775807，如果是unsigned，为0~1844674407370955615的数 |
| boolean或bool | 布尔标志，或者为0或者为1，主要用于开/关(on/off)标志                                                     |
| decimal或dec  | 精度可变的浮点值                                                                            |
| double       | 双精度浮点值                                                                              |
| float        | 单精度浮点值                                                                              |
| int或integer  | 整数值，支持-2147483648~2147483647，如果是unsigned，为0~4294967295                              |
| mediumint    | 整数值，支持-83388608~8388607，如果是unsigned，为0~16777215                                     |
| real         | 4字节的浮点值                                                                             |
| smallint     | 整数值，支持-32768~32767，如果是unsigned，为0~65535                                             |
| tinyint      | 整数值，支持-128~127，如果是unsigned，为0~255                                                   |

日期和时间

| 数据类型      | 说明                                                  |
| --------- | --------------------------------------------------- |
| date      | 表示1000-01-01~9999-12-31的日期，格式为YYYY-MM-DD            |
| datetime  | date和time的组合                                        |
| timestamp | 功能和datetime相同（但范围较小）                                |
| time      | 格式为HH：MM：SS                                         |
| year      | 用2位数字表示，范围是70(1970年)~69(2069年)，用4位数字表示范围是1901~2155年 |

二进制

| 数据类型       | 说明             |
| ---------- | -------------- |
| blob       | Blob最大长度为64KB  |
| mediumblob | Blob最大长度为16MB  |
| longblob   | Blob最大长度为4GB   |
| tinyblob   | Blob最大长度为255字节 |

------

## 《高性能Mysql》

#### 优化特定类型的查询

###### 1.优化count()查询

作用：

1. 统计某个列值的数量。要求列值是非空的（不统计NULL值）。如果在count()的括号中指定了列或列的表达式，则统计的就是这个表达式有值的结果数。
2. 统计行数。当MySQL确认括号内的表达式值不可能为空时，实际上就是在统计行数。最简单的就是在使用count(*)时，这种情况下通配符\*并不会像我们猜想的那样扩展成所有的列，实际上，它会忽略所有的列而直接统计所有的行数。

简单的优化：

```sql
## 如何快速查找到所有ID大于5的城市
select count(*) from world.City where id > 5;
## 这样做的结果是该查询需要扫描4097行数据，如果将条件反转一下，先查找ID小于5的城市数
## 然后用总城市数一减就能得到同样的结果，却可以将扫描的行数减少到5行以内
select (select count(*) from world.City) - count(*) from world.City where ID <=5;
## 这样做可以大大减少需要扫描的行数，是因为在查询优化阶段会将其中的子查询直接当做一个常数来处理。
```

有时候会遇到这样的问题，在同一个查询中统计同一个列中不同值的数量，以减少查询的语句量。假如：假设可能需要同归一个查询返回各种不同颜色的商品数量，此时不能使用or语句。

```sql
select count(color='blue' or color='red') from items;
## 上面这样做就无法区分不同颜色的商品数量，也不能在where条件中指定颜色
select count(*) from items where color='blue' and color='red';
## 因为颜色条件是互斥的

## 下面查询可以在一定程度上解决这个问题
select sum(if(color='blue', 1, 0)) as blue, sum(if(color='red', 1, 0)) as red from items;
select sum(color='blue') as blue, sum(color='red') as red from items;
## 或者
select count(color='blue' or null) as blue, count(color='red' or null) as red from items;
```

使用近似值：

有时候某些业务并不要求完全精确的count值，此时可以用近似值来替代。explain出来的优化器估算的行数就是一个不错的近似值，执行explain并不需要真正地去执行查询，所以成本很低。很多时候，精确计算的成本非常高，而计算近似值则非常简单。

曾今有一个客户希望统计他的网站的当前活跃用户数是多少，这个活跃用户数保存在缓存中，过期时间为30分钟，所以每个30分钟需要重新计算并放入缓存。因此这个活跃用户数本身就不是精确值，所以使用近似值替代是可以接受的。另外，如果要精确统计在线人数，通常where条件是会很复杂，一方面需要剔除当前非活跃用户，另一方面还要剔除系统中某些特定ID的“默认”用户，去掉这些约束条件对总数的影响很小，但却可能很好地提升该查询的性能。更进一步的优化则可以尝试删除distinct这样的约束来避免文件排序。这样重写过的查询要比原来的精确统计的查询快很多，而返回的结果则几乎相同。

更复杂的优化：

通常来说，count()都需要扫描大量的行才能获得精确地结果，因此很难优化。除了之前的方法，在MySQL层面还能做的就只有索引覆盖扫描了。如果这不够，就需要考虑修改应用架构，可以增加汇总表或者增加类似memcached这样的外部缓存系统。

###### 2.优化关联查询

* 确保ON或者USING子句中的列上有索引。在创建索引的时候要考虑到关联的顺序。当表A和表B用列C关联时，如果优化器的关联顺序是B、A，那么就不需要在B表的对应列上建立索引。没有用到的索引只会带来额外的负担。一般来说，除非其他理由，否则只需要在关联顺序中的第二个表的相应列上创建索引。
* 确保任何的GROUP BY和ORDER BY中的表达式只涉及到一个表中的列，这样MySQL才有可能使用索引来优化这个过程。
* 当升级MySQL的时候需要注意：关联语法、运算符优先级等其他可能会发生变化的地方。因为以前是普通关联的地方可能会变成笛卡尔积，不同类型的关联可能会生成不同的结果等。

###### 3.优化子查询

关于子查询优化我们给出的最重要的优化建议就是尽可能使用关联查询代替，至少当前的MySQL版本需要这样。

###### 4.优化GROUP BY 和DISTINCT

在很多场景下，MySQL都使用同样的办法优化这两种查询，事实上，MySQL优化器会在内部处理的时候相互转换这两类查询。它们都可以使用索引来优化，这也是最有效的优化办法。

在MySQL中，当无法使用索引的时候，GROUP BY 使用两种策略来完成：使用临时表或者文件排序来做分组。对于任何查询语句，这两策略的性能都有可以提升的地方。可以通过使用提示SQL_BIG_RESULT 和 SQL_SMALL_RESULT 来让优化器按照你希望的方式运行。

如果需要对关联查询做分组(GROUP BY)，并且是按照查找表中的某个列进行分组，那么通常采用查找表的标识列分组的效率会比其他列更高。

```sql
## 例如下面的查询效率不会很好
select actor.first_name, actor.last_name, count(*)
from sakila.film_actor
    inner join sakila.actor using(actor_id)
group by acotr.first_name, actor.last_name;

## 假如按照下面的写法效率会更高
select actor.first_name, actor.last_name, count(*)
from sakila.film_actor
    inner join sakila.actor using(actor_id)
group by film_actor.actor_id;
```

这个查询利用了actor的id直接相关的特点，因此改写后的结果不受影响，但显然不是所哟的关联语句的分组查询都可以改写成在select中直接使用非分组列的形式的。甚至可能会在服务器上设置SQL_MODE来禁止这样的写法。如果是这样，也可以通过MIN()或者MAX()函数来绕过这种限制，但一定要清楚，select后面出现的非分组列一定是直接依赖分组列，并且在每个组内的值是唯一的，或者是业务上根本不在乎这个值具体是什么：

```sql
select min(actor.first_name), max(actor.last_name), ....;
```

**优化GROUP BY WITH ROLLUP**

分组查询的一个变种就是要求MySQL对返回的分组结果再做一次超级聚合。可以使用with rollup子句来实现这种逻辑，但可能会不够优化。可以通过explain来观察其执行计划，特别要注意分组是否是通过文件排序或者临时表实现的。然后再去掉with rollup子句看执行计划是否相同。

###### 5.优化LIMIT分页

在偏移量非常大的时候，例如可能是 limit 10000,20这样的查询，这是MySQL需要查询10020条记录然后只返回最后的20条，前面的10000条记录都将被抛弃。如果所有的页面被访问的频率都相同，那么这样的查询平均需要访问半个表的数据。要优化这种查询，要么是在页面中限制分页的数量，要么是优化大偏移量的性能。

优化此类分页查询的一个最简单的办法就是尽可能的使用索引覆盖扫描，而不是查询所有的列。然后根据需要做一次关联操作再返回所需的列。对于偏移量很大的时候，这样做的效率会提升非常大。

```sql
select film_id, description, from sakila.film order by title limit 50, 5;
## 如果sakila.film表非常大，这个查询最好改写成下面的样子
select film.film_id, film.description
from sakila.film
    inner join(
        select film_id from sakila.film
        order by title limit 50, 5
    ) as lim using(film_id);
```

这里的“延迟关联”将大大提升查询效率，它让MySQL扫描尽可能少的页面，获取需要访问的记录后再根据关联列会原表查询需要的所有列。这个技术也可以用于优化关联查询中的limit子句。

有时候也可以将limit查询转换为已知位置的查询，让MySQL通过范围扫描获得到对应的结果。例如，如果在一个位置列上有索引，并且预先计算出了边界值，上面的查询就可以改写为：

```sql
select film_id, description from sakila.film
where position between 50 and 54 order by position;
```

对数据进行排名的问题也与此类似，但往往还会同时和GROUP BY混合使用。在这种情况下通常都需要预先计算并存储排名信息。

limit和offset问题，其实是offset的问题，它会导致MySQL扫描大量不需要的行然后抛弃掉。如果可以使用书签记录上次取数据的位置，那么下次就可以直接从该书签记录的位置开始扫描，这样就可以避免使用offset。

```sql
select * from sakila.rental order by rental_id desc limit 20;
## 假设上面的查询返回的是主键为16049到16030的记录，那么下一页查询就可以从16030这个点开始
select * from sakila.rental 
where rental_id < 16030 order by rental_id desc limit 20;
```

此类改写的好处是无论翻页到多么靠后，其性能都会很好。
