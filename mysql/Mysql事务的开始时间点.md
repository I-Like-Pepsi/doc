# Mysql事务的开始时间点

## 简介

在很长一段时间里我都认为在Mysql中事务的开始时间点就是我们执行的**begin/start transaction**执行后，直到有一天在聊天群里发现一个人提出的疑惑。大致意思如下：

|     | sessionA               | sessionB                                                                           |
| --- | ---------------------- | ---------------------------------------------------------------------------------- |
| t1  | start transaction;     |                                                                                    |
| t2  |                        | start transaction;                                                                 |
| t3  |                        | INSERT INTO t_score (id, user_name, score, kemu) VALUES ('1', 'mac', '100', '语文'); |
| t4  |                        | commit;                                                                            |
| t5  | select * from t_score; |                                                                                    |

> 注意Mysql的隔离级别为可重复读

现在的问题是sessionA能读到sessionB插入的数据吗？因为Mysql的隔离级别是可重复读，所以我想都没想这必定是查不到的。如果能查到，那岂不是和可重复读自相矛盾了。

但是我在Mysql上试验完之后发现，sessionB插入的数据sessionA是能查到的，对于这个结果我简直不敢相信。

## 再次试验

对于上面的结果我无法理解，于是又尝试下面的执行顺序：

|     | sessionA               | sessionB                                                                           |
| --- | ---------------------- | ---------------------------------------------------------------------------------- |
| t1  | start transaction;     |                                                                                    |
| t2  |                        | start transaction;                                                                 |
| t3  | select * from t_score; |                                                                                    |
| t4  |                        | INSERT INTO t_score (id, user_name, score, kemu) VALUES ('1', 'mac', '100', '语文'); |
| t5  |                        | commit;                                                                            |
| t6  | select * from t_score; |                                                                                    |

对于sessionA的第一次查询结果没什么好奇怪的肯定查不到，因为此时sessionB都还没有开始插入数据。sessionA的第二次查询按照可重复读的逻辑来说，应该也是查不到的。事实证明的确跟我们想的一样是无法查到的。

## 原因

一般情况下我们都认为begin/start transaction是事务开始的时间点，也就是一旦我们执行了start transaction事务就开始了，其实这是错误的。事务开始的真正时间点是start transaction之后执行的第一条语句，不管是什么语句，不管成功与否。

如果我们想要实现在执行start transaction后立马开启事务，需要加上```with consistent snapshot```才行。关于这点可以参考Mysql官方文档中的描述：

```http
https://dev.mysql.com/doc/refman/5.7/en/commit.html
```

而这条语句的真正含义是：执行start transaction同时建立本事务一致性读的快照，而不是等到执行第一条语句是才开始事务并建立一致性读的快照。

对于文中最初的例子，我们修改```start transaction```为```start transaction with consistent snapshot```，然后按照之前的顺序再次执行，最后sessionA是无法读取到sessionB插入的数据的。
