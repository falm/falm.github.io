---
layout: post
title: mysql
category: 技术
tags: [mysql]
keywords: mysql, 性能优化
---

## MySQL几种随机查询的性能分析

这周遇到一个需求，是要根据若干条件，随机的选取一些数据，本来这个问题还是比较简单的，可以使用编程语言自带的随机函数，对查询出来的数据集再进行，随机选取，但是大家都知道，如果在数据表的数据量上来了后，这种方法显然就是非常不靠谱的，本文我就来聊一聊通过MySQL数据库查询随机行数据。

### 方法

首先让我们来列举一下，可能使用的若干方法。

#### Order by Rand() 

这个方法，几个方案中最慢的一个，但是我们还是把它列举了出来，没有比较就没有伤害，所以一定要有一个速度慢的衬托才行。它之所以慢，是因为 在order by 子句后的 rand() 函数会先为每一行数据生成一个 1~0之间的随机数，然后在根据这个数字，进行排序再选出最小的N行数据（N取决于limit N)。

```mysql
mysql> explain select * from users order by RAND() limit 1\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: users
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 8
        Extra: Using temporary; Using filesort
1 row in set (0.00 sec)
```

**优点:** 简单易记，可以任意选取若干数据行，可以使用条件限定结果集。

**缺点:** 该语句的执行速度取决于，数据量的多少，一般来说超过万行的数据就不推荐使用这种方式了。

#### Rand() 改进方法1

上面使用 order by rand() 方法，我们说了它的性能非常差，这个方法就是对它的改进，同样是使用rand() 函数不过这次我们把，它用在 where条件中。

```mysql
SELECT id FROM users, (SELECT ((1/COUNT(*))*100) as n FROM users) as x WHERE RAND()<=x.n LIMIT 1;
```

上面的方法，首先使用了一个子查询，计算出你想要随机出的记录所在总记录的百分比，然后再乘上100（防止比例过小）再使用这个小数，去和随机数比较，取出小于或等于这个小数的记录。举个例子 你想从一百万条记录中随机取10条记录，那么算式就是 10/1_000_000 * 100 = 0.001 查询语句就是：

```mysql
SELECT id FROM users WHERE RAND()<=0.001 LIMIT 10;
```

```mysql
mysql> explain SELECT id FROM users, (SELECT ((1/COUNT(*))*100) as n FROM users) as x WHERE RAND()<=x.n LIMIT 1\G
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: <derived2>
         type: system
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 1
        Extra: NULL
*************************** 2. row ***************************
           id: 1
  select_type: PRIMARY
        table: users
         type: index
possible_keys: NULL
          key: index_users_on_user_key
      key_len: 767
          ref: NULL
         rows: 210220
        Extra: Using where; Using index
*************************** 3. row ***************************
           id: 2
  select_type: DERIVED
        table: users
         type: index
possible_keys: NULL
          key: index_users_on_user_key
      key_len: 767
          ref: NULL
         rows: 210220
        Extra: Using index
```

**优点:** 速度尚可，可以用于主键非连续的表中，可以容易的使用Limit和Where语句限定随机结果集的大小和条件。

**缺点:** 需要子查询统计总记录数（数据量大可能比较耗时），随机性不好 末尾的记录的随机比例远低于其他记录，对于随机分布要求比较高的场景，就不太适合了。

#### Rand() 改进方法2

改进方法1中达到了快速数据的目的，但是它的随机性不好，那么改进方法2就是使用一定的性能去换取随机分布率。

```mysql
SELECT id FROM users, (SELECT ((1/COUNT(*))*100) as n FROM users) as x WHERE RAND()<=x.n ORDER BY RAND() LIMIT 1;
```

只需要再主查询语句中加入 order by rand()就可以达到随机分布率的提升。

**优点:** 和**改进方法1**的有点一样，并且随机分布更好。

**缺点:** 因为使用了order by rand() 所以该语句的执行速度取决于，数据量的多少。

#### Inner join 

```mysql
SELECT * FROM users as u JOIN (SELECT ROUND(RAND() * (SELECT MAX(id) FROM users)) AS id ) AS u2 WHERE u.id >= u2.id ORDER BY u.id DESC LIMIT 1;
```

该方法巧妙的使用了自增长的ID主键，取其最大值，然后再乘上随机函数 的到一个 随机的ID，这样你就可以根据想要得到的随机记录数，决定使用  >= 或是 = 运算符去筛选结果了( = 仅用于随机一条记录的情况)。

**优点:** 速度非常快。

**缺点:** 查询语句稍微有些复杂，被查询的表必须是连续自增的主键表，例如（1，2，3....N) 不能是 （1,3,8,22) 因为根据最大ID随机出来的不确定ID可能不存在对应的记录，并且无法使用条件去筛选随机结果集。

### 性能比较

我们使用下面的表结构，去分别创建（10K，25K，50K，100K，250K，500K，1000K）的数据，然后分别使用上面的几种方案进行查询，看看他们的性能如何。

```mysql
+------------+--------------+------+-----+---------+----------------+
| Field      | Type         | Null | Key | Default | Extra          |
+------------+--------------+------+-----+---------+----------------+
| id         | int(11)      | NO   | PRI | NULL    | auto_increment |
| nickname   | varchar(191) | YES  |     | NULL    |                |
| avatar_url | varchar(255) | YES  |     | NULL    |                |
| uid        | int(11)      | YES  |     | NULL    |                |
| user_key   | varchar(36)  | YES  | UNI | NULL    |                |
| channel_id | varchar(36)  | NO   | UNI | NULL    |                |
| created_at | datetime     | NO   |     | NULL    |                |
| updated_at | datetime     | NO   |     | NULL    |                |
| user_type  | smallint(6)  | YES  |     | 1       |                |
| cellphone  | varchar(50)  | YES  | MUL | NULL    |                |
+------------+--------------+------+-----+---------+----------------+
```

![benchmark](http://upload-images.jianshu.io/upload_images/1767848-e1c38de23f65db29.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以从上图看出第一种方法和其他三种有非常显著的性能差别，方法3和方法2也在一百万的数据量上开始拉开距离了，inner join 方法即使在一百万的数据量上，也是非常快速的。
### 总结

综上所述，除了第一种方法外，其他的三种方式都可以根据你，具体的使用场景，包括数据量，想要获取的记录数是否有限定条件，去决定使用哪种方式。