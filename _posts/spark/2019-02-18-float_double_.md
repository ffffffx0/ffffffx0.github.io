---
title: float&double&decimal精度损失
tagline: "spark"
category : java
layout: post
tags : [Spark,scala,java]
---
起初是同事在用spark sql时候反馈的一个问题，sql如下

```
%hive
select 
10900*now_pocket_limit_rate as d
from 
dp_fk_tmp.dszx_88w_v2
where uid='41195' 
```
这个now_pocket_limit_rate在表里的值为0.7，返回结果为：
```
7629.999999999999
```
此时脑海居然浮现出之前decimal group by不准确的问题，好吧先让用cast as decimal(38,1)) 替换

```
select 
cast(10900*now_pocket_limit_rate  as decimal(38,1)) as d
from 
dp_fk_tmp.dszx_88w_v2
where uid='41195' 
```
结果肯定没问题

回想起来以为是spark sql的问题，搜索无果，然后用scala跑了下

```
scala> 10900*0.7
res0: Double = 7629.999999999999

scala> 10900*0.5
res1: Double = 5450.0
```

这个难道是scala的问题，直到我又用java跑了下

```
 System.out.println(10900 * 0.7);
```
输出也是7629.999999999999

卧槽，脑回路才反应过来，这不是浮点精度问题么,经典的还有1-0.9

```
scala> 1.0 - 0.9
res6: Double = 0.09999999999999998
```

shit  疑问了半天，是个精度问题。。。

这个mysql也是这样的吧

```
mysql> create table tt(num double);
Query OK, 0 rows affected (0.07 sec)

mysql> insert into tt(num)values(0.7);
Query OK, 1 row affected (0.00 sec)

mysql> select *  from tt;
+------+
| num  |
+------+
|  0.7 |
+------+
1 row in set (0.00 sec)

mysql> select num*10900 from tt;
+-------------------+
| num*10900         |
+-------------------+
| 7629.999999999999 |
+-------------------+
1 row in set (0.00 sec)

mysql> 
```
默认方式
```
mysql> CREATE TABLE tt(f FLOAT DEFAULT NULL,d DOUBLE DEFAULT NULL,de DECIMAL DEFAULT NULL);
Query OK, 0 rows affected (0.09 sec)
mysql> desc tt;
+-------+---------------+------+-----+---------+-------+
| Field | Type          | Null | Key | Default | Extra |
+-------+---------------+------+-----+---------+-------+
| f     | float         | YES  |     | NULL    |       |
| d     | double        | YES  |     | NULL    |       |
| de    | decimal(10,0) | YES  |     | NULL    |       |
+-------+---------------+------+-----+---------+-------+
3 rows in set (0.00 sec)
```
插入数据

```
mysql> INSERT INTO tt(f,d,de) VALUES(1.234,1.234,1.234);
Query OK, 1 row affected, 1 warning (0.06 sec)

mysql> select * from tt;
+-------+-------+------+
| f     | d     | de   |
+-------+-------+------+
| 1.234 | 1.234 |    1 |
+-------+-------+------+
1 row in set (0.00 sec)
```

这个是默认的精度方式插入的：float，double按照四舍五入，decimal默认是整形数据，没有保留小数点后的数据

接着修改下精度和标度
```
mysql> ALTER TABLE tt MODIFY f FLOAT(8,2);
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> ALTER TABLE tt MODIFY d DOUBLE(8,2);
Query OK, 0 rows affected (0.04 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> ALTER TABLE tt MODIFY de DECIMAL(8,2);
Query OK, 1 row affected (0.05 sec)
Records: 1  Duplicates: 0  Warnings: 0

mysql> desc tt;
+-------+--------------+------+-----+---------+-------+
| Field | Type         | Null | Key | Default | Extra |
+-------+--------------+------+-----+---------+-------+
| f     | float(8,2)   | YES  |     | NULL    |       |
| d     | double(8,2)  | YES  |     | NULL    |       |
| de    | decimal(8,2) | YES  |     | NULL    |       |
+-------+--------------+------+-----+---------+-------+
3 rows in set (0.00 sec)

mysql> select * from tt;
+------+------+------+
| f    | d    | de   |
+------+------+------+
| 1.23 | 1.23 | 1.00 |
+------+------+------+
1 row in set (0.00 sec)
```
sum求和有时也会有问题
```
mysql> SELECT SUM(f),SUM(d),SUM(de) FROM tt;
+--------+--------+---------+
| SUM(f) | SUM(d) | SUM(de) |
+--------+--------+---------+
|   1.23 |   1.23 |    1.00 |
+--------+--------+---------+
1 row in set (0.00 sec)

mysql> INSERT INTO tt(f,d,de) VALUES(0.001,0.001,0.001);
Query OK, 1 row affected, 1 warning (0.24 sec)

mysql> SELECT SUM(f),SUM(d),SUM(de) FROM tt;
+--------+--------+---------+
| SUM(f) | SUM(d) | SUM(de) |
+--------+--------+---------+
|   1.23 |   1.23 |    1.00 |
+--------+--------+---------+
1 row in set (0.00 sec)

mysql> INSERT INTO tt(f,d,de) VALUES(0.01,0.01,0.01);
Query OK, 1 row affected (0.00 sec)

mysql> SELECT SUM(f),SUM(d),SUM(de) FROM tt;
+--------+--------+---------+
| SUM(f) | SUM(d) | SUM(de) |
+--------+--------+---------+
|   1.24 |   1.24 |    1.01 |
+--------+--------+---------+
1 row in set (0.01 sec)

mysql> select * from tt;
+------+------+------+
| f    | d    | de   |
+------+------+------+
| 1.23 | 1.23 | 1.00 |
| 0.00 | 0.00 | 0.00 |
| 0.01 | 0.01 | 0.01 |
+------+------+------+
3 rows in set (0.00 sec)

mysql> ALTER TABLE tt MODIFY f FLOAT;
Query OK, 0 rows affected (0.08 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> ALTER TABLE tt MODIFY d DOUBLE;
Query OK, 0 rows affected (0.04 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> ALTER TABLE tt MODIFY de DECIMAL;
Query OK, 3 rows affected, 1 warning (0.06 sec)
Records: 3  Duplicates: 0  Warnings: 1

mysql> select * from tt;
+-------+-------+------+
| f     | d     | de   |
+-------+-------+------+
| 1.234 | 1.234 |    1 |
|     0 |     0 |    0 |
|  0.01 |  0.01 |    0 |
+-------+-------+------+
3 rows in set (0.00 sec)

mysql> INSERT INTO tt(f,d,de) VALUES(1.234,0.01,1.23);
Query OK, 1 row affected, 1 warning (0.00 sec)

mysql> select * from tt;
+-------+-------+------+
| f     | d     | de   |
+-------+-------+------+
| 1.234 | 1.234 |    1 |
|     0 |     0 |    0 |
|  0.01 |  0.01 |    0 |
| 1.234 |  0.01 |    1 |
+-------+-------+------+
4 rows in set (0.00 sec)

mysql> SELECT SUM(f),SUM(d),SUM(de) FROM tt;
+-------------------+--------+---------+
| SUM(f)            | SUM(d) | SUM(de) |
+-------------------+--------+---------+
| 2.477999934926629 |  1.254 |       2 |
+-------------------+--------+---------+
1 row in set (0.00 sec)

mysql>
```
