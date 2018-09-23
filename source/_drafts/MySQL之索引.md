---
title: MySQL之索引
categories:
  - MySQL
  - 数据库
tags:
  - MySQL
date: 2018-09-23 22:43:49
---

简介

<!-- more -->

正文


我不说结论，就举个例子，假设表有4个列（比题主多1列，这样可以把问题说明白）：

id, name, age, sex
id是主键，(name, age)是索引，那么在InnoDB里实际上包含2个索引(即2个B+树)：

id -> (name, age, sex); // 主索引，每个InnoDB的表必然包含一个主索引，主索引必然包含所有的列
(name, age) -> id; // (name, age)这个索引
现在问题来了，如果你的查询是：

SELECT id FROM table WHERE name = ? AND age = ?
只要查(name, age)索引就可以了，不会再查主索引的；但如果查询是：

SELECT id, sex FROM table WHERE name = ? AND age = ?
先查(name, age)索引查到id后，再会查主索引查到sex，所以会比刚才多查一次。