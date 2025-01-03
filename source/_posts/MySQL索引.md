---
title: MySQL索引
category: MySql
tag: 索引
date: 2024/10/10 20:46:25
---

# MySQL索引



## 索引类型

### 物理存储角度

聚集索引和非聚集索引。

聚集索引其实就是主键索引，该结构把key和data都存放在同一结构中

非聚集索引结构住存放了key值，data还需要通过回表过程来查找



### 逻辑角度

普通索引

唯一索引

主键索引

组合索引

前缀索引

空间索引



### 数据结构角度

B+Tree索引，hash索引，FULLTEXT索引，R-Tree索引





## B-Tree和B+Tree的区别

![循环依赖流程图 (3)](/images/Btree和B+tree简化图.png)



- B+树中，所有的叶子结点存放的是索引+数据，而B树中，内结点也存放了数据。 所有B+树的内结点存放的索引数据更多，也就降低了树的高度，减少IO次数，提升了查询的效率，另外B+树的所有查询都要落到叶子结点，检索的时间比较平均
- B+树中的叶子结点是双向链表的关系，好处是方便范围查找