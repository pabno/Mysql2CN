# Index Condition Pushdown Optimization


Index Condition Pushdown (ICP) 是一项适用于Mysql使用索引从表中检索行的优化。没有启用ICP，存储引擎
将遍历索引以定位基础表中的行并将他们返回给Mysql Server用于Where条件的求值。启用了ICP，如果Where条件的部分可以只使用索引列求值，Mysql Server会把这部分Where条件下推到Mysql 存储引擎层。存储引擎使用索引条目来计算下推的索引条件的值，且只有这些条件都满足后才会从表中读取这行记录。ICP可以减少存储引擎必须访问基础表的次数和Mysql Server必须访问存储引擎的次数。


适用于ICP优化的场景需要满足以下条件：
* ICP 用于 `range`, `ref`, `eq_ref`和`ref_or_null`访问方法需要访问一整行时的场景

* ICP可以被用于`InnoDB`和`MyISAM`表，包括他们的分区表

* 对于`InnoDB`表，ICP只能用于二级索引。ICP的目标是用于减少全行读取并因此减少I/O操作。对于`InnoDb`的聚簇索引，完整的行已经被读取进`InnoDb`缓冲区，这种情况下使用ICP并不能减少I/O

* ICP不支持二级索引建立在virtual generated columns的情况，`InnoDB`支持在virtual generated columns上建立二级索引
> virtual generated columns的值由其他列计算得到

* 相关子查询不可以使用ICP

* 涉及存储函数的条件不能使用ICP，存储引擎不能执行存储函数

*  触发条件不能使用ICP. (For information about triggered conditions, see [Section 8.2.2.4, “Optimizing Subqueries with the EXISTS Strategy”](https://dev.mysql.com/doc/refman/5.7/en/subquery-optimization-with-exists.html "8.2.2.4 Optimizing Subqueries with the EXISTS Strategy").)

为了了解这项优化的工作方式，首先要考虑的就是在不是用ICP时索引扫描是怎么处理的：

1. 获取下一行时，首先读取索引元组，然后使用这个索引元组定位并读取一整行

2. 测试这行数据是否满足where条件。基于这个测试接收或者拒绝这一行


使用ICP时，扫描过程变成：
1. 获取下一行的索引元组（不读取一整行）

2. 只使用索引列测试这行数据是否满足where条件。如果条件不满足则使用索引元组测试下一行

3. 如果条件满足，使用索引元组定位并读取一整行数据。

4. 测试where条件剩下的部分，基于测试结果判断是接收该行还是拒绝该行。


当使用ICP时，`EXPLAIN`的输出在`Extra`列显示`Using index condition`，且不会显示`Using Idex`。因为在需要读取全行时这并不适用。

假设有一张表保存了人物和他们地址的信息，表中定义了索引`INDEX (zipcode, lastname, firstname)`。如果我们知道了人物的`zipcode`值但不确定他的last name，我们可以如此查询

```
SELECT * FROM people
  WHERE zipcode='95054'
  AND lastname LIKE '%etrunia%'
  AND address LIKE '%Main Street%';
```

Mysql 可以使用索引查询任务通过`zipcode='95054'`条件，第二部分（`lastname LIKE '%etrunia%'`）不能用于限制必须扫描的行数，如果不使用ICP，这个查询必须检索所有满足`zipcode='95054'`条件的人物的数据行。
使用了ICP，Mysql 在读取整个数据行前检查`lastname LIKE '%etrunia%'`部分。这避免了读取满足`zipcode`条件但不满足`lastname`条件的全行数据


ICP默认开启，可以对`optimizer_switch`系统变量设置`index_condition_pushdown`标识来控制开关：


```
SET optimizer_switch = 'index_condition_pushdown=off';
SET optimizer_switch = 'index_condition_pushdown=on';
```

See [Section 8.9.3, “Switchable Optimizations”](https://dev.mysql.com/doc/refman/5.7/en/switchable-optimizations.html "8.9.3 Switchable Optimizations").