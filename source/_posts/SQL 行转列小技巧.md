---
title: "SQL 行转列小技巧"
catalog: true
date: 2020-12-09 23:51:24
subtitle: “学习笔记”
header-img: "/img/default.png"
tags:
- 后端
- Mysql
catagories:
---

好久没更新博客了，最近在网上看到一道SQL行列转换的问题，之前没遇到过类似的场景，完全想不到该咋搞，特此记录一下吧。

问题是这样的，如何通过SQL语句将如下表1展现成表2的样子？

##### 表1
|year|month|value|
|--|--|--|
|2018|1|1.2|
|2018|2|1.3|
|2018|3|1.2|
|2018|4|1.1|
|2019|1|2.2|
|2019|2|2.2|
|2019|3|2.9|
|2019|4|1.8|


##### 表2
|year|m1|m2|m3|m4|
|--|--|--|--|--|
|2018|1.2|1.3|1.2|1.1|
|2019|2.2|2.2|2.9|1.8|

答案：
```
SELECT 
    t.year,
    MAX(IF(t.month = 1, t.value, 0)) AS 'm1',
    MAX(IF(t.month = 2, t.value, 0)) AS 'm2',
    MAX(IF(t.month = 3, t.value, 0)) AS 'm3',
    MAX(IF(t.month = 4, t.value, 0)) AS 'm4'
FROM
    data_table t
GROUP BY year;
```

思考：
> 其实从表2的行上首先可以看出数据是GROUP BY year准没错啦，然后是要思考列上如何展示指定月份的对应的值。我理解为，GROUP BY 后数据被分成若干个Bucket，我们可以Bucket中的数据进行聚合，和Elasticsearch的聚合函数类似。因此，我们可以通过*max()* 或 *if()* 函数帮忙计算一下对应结果。
> 但是，反思了一下。这样的SQL对每一行每一列都会进行计算，当数据量巨大时这样的SQL会极大的消耗数据库CPU，降低吞吐量，不要随意使用。






