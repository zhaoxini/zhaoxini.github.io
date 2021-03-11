---
title: mysql下的join buffer
date: 2020-07-10 22:07:48
tags: Mysql
---

我们在查询的时候，通过explain可以看到经常有使用join buffer（Using join buffer(Block Nested Loop)

![](https://user-gold-cdn.xitu.io/2020/7/22/17375660c0e7449d?w=2538&h=372&f=png&s=82383)

要想知道join buffer，就是先知道join运行的原理。

## 嵌套循环连接（Nested-Loop Join）

[详细请参考掘金小册《Mysql是怎样运行的》](https://juejin.im/book/6844733769996304392/section/6844733770055024654)

join的原理就是嵌套循环连接，驱动表作为第一层，被驱动表作为第二层，情况基本如下

```python
for 驱动表的每行记录 in 驱动表的记录
	for 被驱动表每行记录 in 被驱动表的记录
		if (on条件（驱动表的每行记录，被驱动表的每行记录）？return true : false
```

时间复杂度为 count(驱动表的记录) x count(被驱动表) 

比如：有一张课程表course

```java
course                                teacher
---------------------                 ------------------------
course_id| teacher_id                 teacher_id| teacher_name
---------------------                 ------------------------
1        | 1                          1         | leo
---------------------                 ------------------------
2        | 2                          2         | john
---------------------                 ------------------------
```

sql语句如下

```sql
 select * from course left join teacher 
	on course.teacher_id = teacher.teacher_id 
	where course_id > 0。
```

因为是左连接，cource为驱动表， teacher为被驱动表。

1. **第一步筛选出驱动表符合条件的记录**

   mysql 查询的时候先根据where条件进行对course的单表查询, 查询出的每一条记录都要在与teacher进行一次查询匹配。例如，where course_id > 0, 那么从course表中拿到第一条记录的时候发现是>0的，可以留下，然后该去teacher表中进行匹配了。

2. **通过连接条件on后的条件对被驱动表的数据筛选**

   第一条的teacher_id 为1，on 条件为teacher_id 相等，也就是我们在teacher表中执行如下的单表查询

   ```java
   select * from teacher where teacher_id = 1 
   ```

   那么这条记录如果没有加任何索引，执行中要查询的就是这个teacher表中的所有记录。

**3. 将查询的结果与驱动表进行连接并返回给客户端**

连接就要根据左连接还是右连接进行匹配了，没有的加null值，等等。

## 基于块的嵌套循环连接（Block Nested-Loop Join）

这样一条一条的查就像双重循环，效率低下，被驱动表如果数据量大会出现性能瓶颈。

我们可以思考几个问题：

- 每一条驱动表的记录去被驱动表进行条件匹配的时候，重复匹配了多次，我们是否加个缓存来解决？
- 缓存的目的是什么？
- 如果加缓存，这个缓存应该加在哪，驱动表还是被驱动表？

首先我们加缓存的目的是为了取一次数据尽量不被舍弃，可以多次复用。我们查询了很多次被驱动表，如果查询了一次就被缓存下来，在内存中进行数据的匹配操作，效率会高很多。但是会出现一个问题，被驱动表往往是数据量大的一方，被驱动表也可以是多个。所以缓存被驱动表到内存不现实。我们可以反向来想，缓存驱动表。将驱动表的筛选出的所有数据进行缓存，然后每次从被驱动表拿来的数据都与缓存的驱动表的所有记录进行匹配，符合条件的都留下来。这样被驱动表的每条记录只取一次就可以了。

所以join_buffer 就是这个缓冲区，join buffer就是执行连接查询前申请的一块固定大小的内存，先把若干条驱动表结果集中的记录装在这个join buffer中，然后开始扫描被驱动表，每一条被驱动表的记录一次性和join buffer中的多条驱动表记录做匹配。join_buffer 就是这个内存块。

![](https://user-gold-cdn.xitu.io/2020/7/22/1737563b590bb0c5?w=826&h=479&f=png&s=104174)

复杂度也从O(n^2) 变成了 O(n)

## MySQL如何使用联接缓冲区缓存

上面所有说的过程都是在被驱动表查询类型是ALL或者Index, 也就是进行全盘扫描，没有加索引，也不是等值连接这种情况下。

[官方文档定义](https://dev.mysql.com/doc/internals/en/join-buffer-size.html)：

Basic information about the join buffer cache:

- The size of each join buffer is determined by the value of the `join_buffer_size` system variable.
- ***This buffer is used only when the join is of type `ALL` or `index` (in other words, when no possible keys can be used).***
- A join buffer is never allocated for the first non-const table, even if it would be of type `ALL` or `index`.
- The buffer is allocated when we need to do a full join between two tables, and freed after the query is done.
- Accepted row combinations of tables before the `ALL`/`index` are stored in the cache and are used to compare against each read row in the `ALL` table.
- ***We only store the used columns in the join buffer, not the whole rows.***

上面说，仅仅是在type 为ALL 或者 index 条件下，为什么？

当然是因为使用其他计算方法效率更高，比如我们驱动表进行条件筛选的时候，where teacher_id =1 , 给teacher_id 加上索引就不需要筛选被驱动表中的所有数据，这样的效率更高。只有在Mysql优化器认定type为ALL的时候才**可能**会去用join_buffer

最后一句说：我们仅仅存储使用到的列，而不是整个行

join_buffer存储的是驱动表中用到的列的数据，比如course表，记录还是两条，但是每条记录里只有teacher_id而没有course_id。

所以要纠正一下上文中的说话，比如记录啊、每一行并不是真的指行里所有的数据，只是需要用到的列的数据。

在看下官方的定义的执行步骤：

假设您具有以下联接：

```sql
Table name      Type
t1              range
t2              ref
t3              ALL
```

然后按以下步骤完成连接：

```sql
- While rows in t1 matching range
 - Read through all rows in t2 according to reference key
  - Store used fields from t1, t2 in cache
  - If cache is full
    - Read through all rows in t3
      - Compare t3 row against all t1, t2 combinations in cache
        - If row satisfies join condition, send it to client
    - Empty cache

- Read through all rows in t3
 - Compare t3 row against all stored t1, t2 combinations in cache
   - If row satisfies join condition, send it to client
```

前面的描述意味着`t3`扫描表的次数 确定如下：

```sql
S = size-of-stored-row(t1,t2)
C = accepted-row-combinations(t1,t2)
scans = (S * C)/join_buffer_size + 1
```

t1 range ,t2 ref ，所以这部分没有用join_buffer，而是将t1和t2匹配的结果缓存在缓冲区

[]()

t3 为all, 读取的时候拿出t3的每一行与t1和t2缓存进行比较。

[]()

扫描的行数为

scans = (S * C)/join_buffer_size + 1

这个join_buffer 也是有一定大小的，如果驱动表> join_buffer，就需要分多次。所以上面要除以/join_buffer_size

## 一些结论

Some conclusions:

- The larger the value of `join_buffer_size`, the fewer the scans of `t3`. If `join_buffer_size` is already large enough to hold all previous row combinations, there is no speed to be gained by making it larger.
- If there are several tables of join type `ALL` or `index`, then we allocate one buffer of size `join_buffer_size` for each of them and use the same algorithm described above to handle it. (In other words, we store the same row combination several times into different buffers.)

- 值越大`join_buffer_size`，扫描的次数越少`t3`。如果 `join_buffer_size`已经足够大以容纳所有先前的行组合，则使其变大无法获得任何速度。
- 如果有多个联接类型为`ALL`或的表`index`，则我们`join_buffer_size`为每个表 分配一个大小相同的缓冲区， 并使用上述相同的算法进行处理。（换句话说，我们将同一行组合多次存储到不同的缓冲区中。）

**参考 《MySQL 是怎样运行的：从根儿上理解 MySQL》 作者:小孩子4919**