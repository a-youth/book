---
Title: mysql索引
Keywords: 索引,index,联合索引，覆盖索引
Description: 普通索引和唯一索引应该怎么选择？字符串如何加索引才能更高效并且节省空间？
Author: douyacun
Cover: 
Label: 索引
Date: 2019-11-26 10:24:07
LastEditTime: 2019-11-26 23:07:12
---

innodb是索引存储表，表数据按主键顺序存放

[toc]

# 基础概念

聚簇索引、辅助索引、联合索引

## 聚簇索引

聚集索引按照每张表的主键构造一棵B+树，叶子节点存放的是整张表的行记录数据，叶子节点被称为数据页。

每张表只能按照一颗B+树排序，每张表只能有一个聚集索引

聚集索引对于主键的排序查找和范围查找速度非常快

没有主键或唯一索引，innodb默认会生成一个行号(不可见)，作为聚集索引的主键

## 辅助索引

辅助索引叶子节点不会包含行记录的所有数据。叶子节点除了包含键值以外还会包含主键，来查找叶子节点以外的其它数据。

辅助索引通过主键id来关联记录

## 联合索引

对表上的多个列进行索引

联合索引的技巧

1. 覆盖索引：从索引即可查询到所需字段，不需要查询聚集索引中的记录，减少io操作。

2. 最左前缀：对a,b创建索引

   - 可以用到索引
     - where a = 1;
     - where a = 1 and b = 2;
     - where a = 1 order by b;

   - 用不到索引
     - where b = 2;

3. 索引下推：a like 'hello%’ and b >10，检索出来的数据会先过滤掉b <= 10 的数据，然后在进行回表查询，减少回表查

## MRR(multi-range read)

目的： 减少磁盘的随机访问，将随机访问变为顺序的数据访问。

MRR执行过程：

- 优化器将二级索引查询到的记录放到一块缓存中
- 如果二级索引扫描到文件的末尾或者缓冲区已满，使用快排对缓冲区中的内容按照住建进行排序
- 用户线程调用MRR接口取cluster index, 然后根据cluster index取行数据
- 当根据缓冲区中的cluster index取完数据，继续调用2、3过程，直至扫面结束 

MRR好处：

- 使数据访问变得较为顺序，在查询辅助索引时，首先根据得到的结果，按照主键排序，
- 减少缓冲池中页被替换的次数
- 批量处理对键值的查询

explain 可以看到extra列 using MRR

- [MySQL · 特性分析 · 优化器 MRR & BKA](http://mysql.taobao.org/monthly/2016/01/04/)

## ICP

mysql 5.6开始支持，将where的过滤部分放到存储层

```sql
select * from salaries
where (from_date between "2019-01-10" and "2019-02-10") and (salaries between 3800 and 4000)
```

不启用ICP，则数据需要先取出from_date between 2019-01-10 and 2019-02-10的数据，然后再过滤 3800-4000的数据

启用ICP，则会在索引取出时就会进行where过滤，前提是where条件被索引覆盖

explain 可以看到extra列 using index condition

- [MySQL · 特性分析 · Index Condition Pushdown (ICP)](http://mysql.taobao.org/monthly/2015/12/08/)

# 疑难杂症

## 普通索引和唯一索引应该怎么选择？

例如身份证号，身份证号具有唯一性，由于身份证号比较大，不建议把身份证号作为主键，要么给身份证号加一个唯一索引，要么加一个普通索引。如果业务代码已经保证不会写入重复的身份证号，那么是选择普通索引还是唯一索引比较好？

查询过程：

- 对于普通索引，查找到第一个满足条件的索引后，需要继续查找下一个记录，知道碰到第一个不满足条件的记录
- 对于唯一索引，由于索引定义了唯一性，查找到第一个满足条件的记录后，会停止继续检索

这点查询性能影响很小，可以忽略，所以唯一索引和普通索引查询性能相差不多(别钻牛角尖，如果是字段重复度高，这种情况不符合我们讨论的场景)。

更新过程：

​	对于插入一条(4,500)

- 这个记录要更新的目标页在内存中
  - 唯一索引，找到3和5之间的位置，判断没有冲突值，插入这个值，语句结束；
  - 普通索引，找到3和5之间的位置，插入这个值，语句结束
- 这个记录要更新的目标页没有在内存中
  - 唯一索引，需要将目标页读入到内存中，判断没有冲突，插入这个值，语句结束
  - 普通索引，将更新记录在change buffer中，语句执行结束

数据从磁盘读入到内存涉及到随机I/O的访问，是数据库成本最高的操作之一。change buffer减少了随机磁盘访问。对于提升更新性能很明显

## change buffer

**什么条件下可以使用change buffer？**

对于唯一索引，所有的更新操作都要先判断这个操作是否违反唯一约束，要判断值是否就必须先把数据页读入到内存中才能判断，如果已经读入到内存中，那直接更新内存会更快，没必要使用change buffer了

唯一索引更新就不能使用change buffer了，实际上只有普通索引可以使用

**change buffer的使用场景？**

- change buffer对于更新过程有加速作用
- merge是数据真正更新的时刻，在一个数据库做merge之前，change buffer记录的越多，收益越大。写多读少的场景change buffer使用效果最好。
- 如果一个业务的更新模式写入之后马上做查询，将更新记录在了change buffer中，但之后由于马上要访问这个数据页，马上会触发merge，反而增加change buffer的维护成本。

**change buffer和redo log**

change buffer主要是在更新时，不用将数据读入到内存中，节省的是随机读磁盘的I/O消耗。

redo log主要节省的是随机写磁盘的I/O消耗（顺序写）

## 如何重建索引和主键索引？

索引可能因为删除，或者页分裂等原因，导致数据页有空洞，造成空间浪费

```sql
alter table T engine=InnoDB;
```

## 字符串如何创建索引?

**前缀索引**

定义好长度，可以做到既节省空间，又不用额外增加太多的查询成本。

例如邮箱，对于每个记录都是只取前6个字节

```sql
alter table user add index index2(email(6));
```

- 占用的空间更小
- 增加额外的扫描次数
- 用不上覆盖索引对查询性能的优化，需要回主键索引查一次

```sql
select id, email from SUser where email='douyacun@gmail.com';
```

当要给字符串创建前缀索引时，有什么方法能够确定我应该使用多长的前缀呢？

```sql
select 
  count(distinct left(email,4)）as L4,
  count(distinct left(email,5)）as L5,
  count(distinct left(email,6)）as L6,
  count(distinct left(email,7)）as L7,
from user;
```

**倒序存储**

例如：身份证号前面的几位是地址码重复度高，这个索引的区分度就非常低了。按照前缀索引来创建的索引需要创建12位以上，才能满足区分度要求。

存储身份证号的时候把它倒过来存，每次查询的时候，可以这么写：

```sql
select field_list from t where id_card = reverse(:id_card);
```

**hash字段**

使用crc32函数得到校验码使用整型存储，只在这个列上加索引，索引空间就会小很多。但是CRC32函数很容易碰撞，需要加上原条件判断

```sql
select field_list from t where id_card_crc = crc32(:id_card) and id_card = :id_card;
```