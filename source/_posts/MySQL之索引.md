---
title: MySQL之索引
categories:
  - MySQL
  - 数据库
tags:
  - MySQL
date: 2018-09-23 22:43:49
---

索引是存储引擎快速找到记录的一种数据结构，索引对于良好的性能非常关键，尤其是当表中的数据越来越大时。<!-- more -->索引优化应该是对查询性能优化最有效的手段，创建一个最优的索引有时会将查询性能提高几个数量级。

## 索引类型

索引有很多类型，可以为不同的场景提供更好的性能，在MySQL中索引在存储引擎层而不是服务器层实现，所以并没有统一的索引标准，每个引擎实现的索引类型不完全相同，同样的索引实现方式也有可能不同。

### B-Tree 索引

当说到索引时，如果没有特别指明，那么通常是指 B-Tree 索引，它是最常用的索引类型，使用 B-Tree 结构（实际上很多引擎使用的 B+Tree 结构）来存储数据，大多数引擎都支持这种索引。

B-Tree 索引通常意味着所有的值都是按顺序存储的，并且每个叶子页到根的距离相同。B-Tree 索引都够加快访问数据的速度，因为存储引擎不再需要进行全表扫描来获取需要的数据，而是从索引的根节点开始进行搜索。

### 哈希索引

哈希索引基于哈希表实现，只有精确匹配索引所有列的查询才有效，对于每行数据，存储引擎会对所有的索引列计算一个哈希码，哈希索引将所有的哈希码存储在索引中，同时在哈希表中保存指向每个数据行的指针。

在MySQL中，只有 Memory 引擎显示支持哈希索引，但是其他引擎可以在程序层面上模拟哈希索引，比如在数据库表多加一列存储哈希码，哈希码的由应用程序计算，然后在哈希码列加 B-Tree 索引。

### 全文索引

全文索引是一种特殊类型的索引，它查找的是文本中的关键词，而不是直接比较索引中的值。全文索引更类似搜索引擎做的事情，适用于match against操作，而不是普通的where条件操作。

----
**注意：以下内容主要针对的是 InnoDB 存储引擎的 B+Tree 索引数据结构**

## 创建索引


### 主键索引

```
ALTER TABLE'table_name' ADD PRIMARY KEY'index_name' ('column');
```
### 唯一索引

```
ALTER TABLE'table_name' ADD UNIQUE 'index_name' ('column');
```

### 普通索引

```
ALTER TABLE'table_name' ADD INDEX'index_name' ('column');
```

### 全文索引

```
ALTER TABLE 'table_name' ADD FULLTEXT 'index_name' ('column');
```

### 组合索引

```
ALTER TABLE 'table_name' ADD INDEX 'index_name' ('column1', 'column2', ...);
```

## 使用索引

创建一个测试的用户表

```
DROP TABLE IF EXISTS user_test;
CREATE TABLE user_test(	
  id int AUTO_INCREMENT PRIMARY KEY,
  user_name varchar(30) NOT NULL,
  sex bit(1) NOT NULL DEFAULT b'1',
  city varchar(50) NOT NULL,
  age int NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

创建一个组合索引：

```
ALTER TABLE user_test ADD INDEX idx_user(user_name , city , age);
```

### 索引有效的查询

#### 全值匹配

全值匹配指的是和索引中的所有列进行匹配，注意全值匹配与where后查询条件的顺序无关。

```
SELECT * FROM user_test WHERE user_name = 'feinik' AND age = 26 AND city = '广州';
```

#### 匹配最左前缀

匹配最左前缀是指优先匹配最左索引列，如：上面创建的索引可用于查询条件为：（user_name ）、（user_name, city）、（user_name , city , age），满足最左前缀查询条件的顺序与索引列的顺序无关，如：（city, user_name）、（age, city, user_name）。

#### 匹配列前缀

指匹配列值的开头部分，如：查询用户名以feinik开头的所有用户。

```
SELECT * FROM user_test WHERE user_name LIKE 'feinik%';
```

#### 匹配范围值

如：查询用户名以feinik开头的所有用户，这里使用了索引的第一列

```
SELECT * FROM user_test WHERE user_name LIKE 'feinik%';
```

#### 精确匹配某一列并范围匹配另一列

```
SELECT * FROM user_test WHERE user_name = 'feinik' AND city = '广%';
```

### 索引无效的查询

#### 不使用最左列

where查询条件中不包含索引列中的最左索引列，则无法使用到索引查询，如：

```
SELECT * FROM user_test WHERE city = '广州';
```

或

```
SELECT * FROM user_test WHERE age= 26;
```

或

```
SELECT * FROM user_test WHERE city ='广州' AND age = '26';
```

#### 使用列后缀

即使where的查询条件是最左索引列，也无法使用索引查询用户名以feinik结尾的用户。

```
SELECT * FROM user_test WHERE user_name like '%feinik';
```

#### 跳过索引中的列

下面的例子中跳过了 city，所以只能使用索引的第一列 user_name。

```
SELECT * FROM user_test WHERE user_name = 'feinik' AND age = 26;
```

#### 范围查询右边的列无法使用索引

如果where查询条件中有某个列的范围查询，则其右边的所有列都无法使用索引优化查询，比如下面的例子中，age 无法使用索引：

```
SELECT * FROM user_test WHERE user_name ='feinik' AND city LIKE'广州%' AND age = 26;
```

#### 不独立的列

索引列不能是表达式的一部分，也不能作为函数的参数，否则无法使用索引查询。

```
SELECT * FROM user_test WHERE user_name = concat(user_name, 'fei');
```

## 高效索引的策略

### 前缀索引

有时候需要索引很长的字符列，这会增加索引的存储空间以及降低索引的效率，一种策略是可以使用哈希索引，还有一种就是可以使用前缀索引，前缀索引是选择字符列的前n个字符作为索引，这样可以大大节约索引空间，从而提高索引效率。

#### 前缀索引的选择性

前缀索引要选择足够长的前缀以保证高的选择性，同时又不能太长，我们可以通过以下方式来计算出合适的前缀索引的选择长度值：

第一步，计算原始选择性，index_column代表要添加前缀索引的列

```
SELECT COUNT(DISTINCT index_column)/COUNT(*) FROM table_name;
```

第二步，计算不同前缀的选择性

```
SELECT
  COUNT(DISTINCT LEFT(index_column,1))/COUNT(*) AS sel1,
  COUNT(DISTINCT LEFT(index_column,2))/COUNT(*) AS sel2,
  COUNT(DISTINCT LEFT(index_column,3))/COUNT(*) AS sel3,
  ...
FROM table_name;
```

通过以上语句逐步找到最接近于第一步中的前缀索引的选择性比值，那么就可以使用对应的字符截取长度来做前缀索引了。

#### 前缀索引的创建

```
ALTER TABLE table_name ADD INDEX index_name (index_column(length));
```

#### 使用前缀索引的注意点

前缀索引是一种能使索引更小，更快的有效办法，但是MySql无法使用前缀索引做ORDER BY 和 GROUP BY以及使用前缀索引做覆盖扫描。

### 选择合适的索引列顺序

在组合索引的创建中索引列的顺序非常重要，正确的索引顺序依赖于使用该索引的查询方式.

对于组合索引的索引顺序可以通过经验法则来帮助我们完成：将选择性最高的列放到索引最前列，该法则与前缀索引的选择性方法一致，但并不是说所有的组合索引的顺序都使用该法则就能确定，还需要根据具体的查询场景来确定具体的索引顺序。

### 聚簇索引与非聚簇索引

#### 聚簇索引

聚簇索引决定数据在物理磁盘上的物理排序，一个表只能有一个聚簇索引，如果定义了主键，那么 InnoDB 会通过主键来聚簇数据，如果没有定义主键，InnoDB 会选择一个唯一的非空索引代替，如果没有唯一的非空索引，InnoDB 会隐式定义一个主键来作为聚簇索引。

聚簇索引可以很大程度的提高访问速度，因为聚簇索引将索引和行数据保存在了同一个 B-Tree 中，所以找到了索引也就相应的找到了对应的行数据，但在使用聚簇索引的时候需注意避免随机的聚簇索引（一般指主键值不连续，且分布范围不均匀）。

如使用 UUID 来作为聚簇索引性能会很差，因为 UUID 值的不连续会导致增加很多的索引碎片和随机I/O，最终导致查询的性能急剧下降。

#### 非聚簇索引（二级索引）

与聚簇索引不同的是非聚簇索引并不决定数据在磁盘上的物理排序，非聚簇索引只是索引主键列，所以如果主键很大的话，非聚簇索引也会很大。另外非聚簇索引需要两次索引查找，而不是一次。

比如，在上面的例子中 id 是主键，是聚簇索引，（user_name，city, age）上建立的索引就是非聚簇索引，那么在InnoDB里实际上包含2个索引(即2个B+树)：。

```
id -> (user_name, sex, city, age); // 主索引，每个InnoDB的表必然包含一个主索引，主索引必然包含所有的列

(user_name，city, age) -> id; // 非聚簇索引只是索引主键列
```

如果你只查询主键的话，只要查(user_name，city, age)索引就可以了，不会再查主索引的：

```
SELECT id FROM user_test WHERE user_name = ? AND city = ? AND age = ?;
```

但如果你还要查询其他字段，就要先查(user_name，city, age)索引查到id后，再会查主索引查到sex，所以会比刚才多查一次：

```
SELECT id, sex FROM user_test WHERE user_name = ? AND city = ? AND age = ?;
```

### 覆盖索引

如果一个索引（如：组合索引）中包含所有要查询的字段的值，那么就称之为覆盖索引，如：

```
SELECT user_name, city, age FROM user_test WHERE user_name ='feinik' AND age > 25;
```

因为要查询的字段（user_name, city, age）都包含在组合索引的索引列中，所以就使用了覆盖索引查询，查看是否使用了覆盖索引可以通过执行计划中的Extra中的值为Using index则证明使用了覆盖索引，覆盖索引可以极大的提高访问性能。

### 使用索引来排序

在排序操作中如果能使用到索引来排序，那么可以极大的提高排序的速度，要使用索引来排序需要满足以下两点即可:

- ORDER BY 子句后的列顺序要与组合索引的列顺序一致，且所有排序列的排序方向（正序/倒序）需一致；
- 所查询的字段值需要包含在索引列中，及满足覆盖索引。
