Range Access Method使用单个索引来检索表中的行，这些行被包含在一个或者多个索引值的区间中。这可以被用在单列索引或者多列索引中。下面的部分会给出优化器使用Range Access Method的条件的相关描述。

> eg:  where index_clonum >  index_value1 and index_clonum < index_value2

##### The Range Access Method for Single-Part Indexes

对于单列索引，索引值区间可以被表示为相应的where子句中的范围条件


对于单列索引范围条件可以定义为如下：

- 对于`BTREE` and `HASH`来说，当索引列使用`=`, `<=>`, `IN()`, `IS NULL`, `IS NOT NULL`操作符与一个常量比较。


- 对于 `BTREE`索引来说，当索引列使用 `>`, `<`, `>=`, `<=`, `BETWEEN`, `!=`, or `<>`操作符与常量比较，或者使用 `LIKE`操作符与一个字符串常量比较，但是字符串常量不以通配符开头

- 对于所有类型的索引，多个范围范围条件使用`OR`或者`AND`组合成的一个范围条件

上面说的常量，满足下面条件之一则属于常量：

- 查询字符串中的常量（eg: where index_colume = 4）

- 使用join连接的`const`表或者`system`表中的列

  
> const表：表中最多只有一个匹配行，在查询开始时就会被读取。由于只有一行，行中的列值会被优化器认为是常量。`const`表由于只会被读取一次，所以会非常快。
> system表：表中只有一行数据。属于const表的特殊情况，const表是join完后只有一列匹配

- 无相关子查询的结果
- 上面3种情况组合成表达式

下面是一些在`WHERE`子句中包含范围条件的查询:

```
SELECT * FROM t1
  WHERE key_col > 1
  AND key_col < 10;

SELECT * FROM t1
  WHERE key_col = 1
  OR key_col IN (15,18,20);

SELECT * FROM t1
  WHERE key_col LIKE 'ab%'
  OR key_col BETWEEN 'bar' AND 'foo';
```

一些非常量值在优化器常量传播阶段可能会转化成常量

MySQL试图从每个可能的索引的WHERE子句中提取范围条件。在提取过程中，去掉了不能用于构建范围条件的条件，合并了产生重叠范围的条件，去掉了产生空范围的条件

考虑下面的语句， 当`key1`是一个索引列，`nonkey`不是索引列：

```
SELECT * FROM t1 WHERE
  (key1 < 'abc' AND (key1 LIKE 'abcde%' OR key1 LIKE '%b')) OR
  (key1 < 'bar' AND nonkey = 4) OR
  (key1 < 'uux' AND key1 > 'z');
```

对于`key1`的提取过程如下

1. 从原始的 `WHERE` 子句开始:

   ```
   (key1 < 'abc' AND (key1 LIKE 'abcde%' OR key1 LIKE '%b')) OR
   (key1 < 'bar' AND nonkey = 4) OR
   (key1 < 'uux' AND key1 > 'z')
   ```

2. 移除`nonkey = 4` 和 `key1 LIKE '%b'`，因为他们不能用于范围扫描. 正确的移除方式是将他们替换成 `TRUE`, 这样就不会在范围扫描时错过任何匹配的行. 使用`TRUE` 替换后:

   ```
   (key1 < 'abc' AND (key1 LIKE 'abcde%' OR TRUE)) OR
   (key1 < 'bar' AND TRUE) OR
   (key1 < 'uux' AND key1 > 'z')
   ```

3. 去除总是为 true 或者 false的表达式:

   *   `(key1 LIKE 'abcde%' OR TRUE)` 总是为true

   *   `(key1 < 'uux' AND key1 > 'z')` 总是为false

   将这些表达是替换成常量:

   ```
   (key1 < 'abc' AND TRUE) OR (key1 < 'bar' AND TRUE) OR (FALSE)
   ```

   去除不必要的 `TRUE` 和`FALSE` 常量表达式：

   ```
   (key1 < 'abc') OR (key1 < 'bar')
   ```

4. 将重叠的区间合并成一个，可以得到用于范围扫描的最终条件： 

   ```
   (key1 < 'bar')
   ```

通常来说，使用范围扫描的条件是比`WHERE`子句限制少的。Mysql使用另外的检查来过滤掉那些满足范围条件但不满足 `WHERE`子句的行

范围条件提取算法可以处理任意深度的由 `AND`/`OR`嵌套的条件，其输出不依赖于`WHERE`子句中条件出现的顺序。 



对于空间索引，Mysql不支持在范围访问方法中合并多个范围。为了突破这个限制，你可以使用`UNION`联合多个相同的`SELECT` 语句，除非你把每个空间谓词放在一个不同的`SELECT`中。 

> 空间索引没使用过





##### The Range Access Method for Multiple-Part Indexes

多列索引中的范围条件是单列索引的扩展。多列索引范围条件限制索引行位于一个或多个关键字元组区间内。关键字元组区间是在一组关键字元组中定义的，使用来自索引的顺序。 

> 关键字元组可以理解为按照(key_part1, key_part2, key_part3)



举例来说，存在一个多列索引`key1(key_part1, key_part2, key_part3)`。存在以下元组集合列表

```
key_part1  key_part2  key_part3
  NULL       1          'abc'
  NULL       1          'xyz'
  NULL       2          'foo'
   1         1          'abc'
   1         1          'xyz'
   1         2          'abc'
   2         1          'aaa'
```

条件 `key_part1 = 1` 定义了这个区间:

```
(1,-inf,-inf) <= (key_part1,key_part2,key_part3) < (1,+inf,+inf)
```

在上面的数据集中，这个区间覆盖了第4，5，6个元组，且可以使用范围访问方法



相比之下， `key_part3 = 'abc'`没有定义单个区间，所以不能使用范围访问方法

> 左前缀匹配



下面的描述详细说明了多部分索引的范围条件是如何工作的。

* 对于`HASH` 索引，包含相同值的区间才可以使用范围条件。这意味着只有在以下条件才能使用

  ```
      key_part1 cmp const1
  AND key_part2 cmp const2
  AND ...
  AND key_partN cmp constN;
  ```

  这里，_`const1`_, _`const2`_, …是常量，_`cmp`_ 是 `=`, `<=>` or `IS NULL`表达式中的一个，而且条件需要覆盖所有索引列。举个例子，下面是一个有3列的hash索引，且使用了范围条件：

  `key_part1 = 1 AND key_part2 IS NULL AND key_part3 = 'foo'`

  

*   对于`BTREE`索引，区间可以使用`AND`组合条件确定，每个条件使用 `=`,`<=>`, `IS NULL`, `>`, `<`, `>=`, `<=`, `!=`, `<>`, `BETWEEN`, 或者`LIKE 'pattern`'（`pattern`不能用通配符开始）。



只要使用的比较操作符是 `=`, `<=>`, `IS NULL`, 优化器会使用尽可能长的关键字来确定区间。如果操作符是`>`, `<`,`>=` `<=`, `!=`, `<>`, `BETWEEN`, `LIKE`。优化器会使用这些条件，但是不会再使用更多的列。对于下面的表达式，优化器在第一个比较`=`中使用了索引.同时在第二个比较`>=`中使用了索引，但不会再使用更多的列。也不会使用第三个比较条件来确定范围区间了。

> 左前缀匹配。在自己实践中，如果第一个索引列用的是`>=`，那么第二个索引列还是可以用到的(通过key_len来确定是否有使用)。可能除此之外还有其他的优化规则存在



```
key_part1 = 'foo' AND key_part2 >= 10 AND key_part3 > 10
```

确定的区间是:

```
('foo',10,-inf) < (key_part1,key_part2,key_part3) < ('foo',+inf,+inf)
```

确定的范围区间很有可能会比最初的区间大.举个例子，上面确定的区间包含了`('foo', 11, 0)`，但这个元组却不满足原始的`WHERE`条件.

* 如果确定区间的条件是用`OR`组合，他们会形成一个所有确定区间并集的条件

  ```
  (key_part1 = 1 AND key_part2 < 2) OR (key_part1 > 5)
  ```

  确定区间是：

  ```
  (1,-inf) < (key_part1,key_part2) < (1,2)
  (5,-inf) < (key_part1,key_part2)
  ```

  

  在这个例子中，第一行确定的区间使用了第一个索引列确定左边界，使用第二个索引列确定右边界。第二行确定的区间只使用了一个索引列。`EXPLAIN`的`key_len`列指出了使用的索引长度

  

  在一些情况下，`key_len`列只是会指出使用了哪些索引列，但是这可能会与预期的不一样。假设`key_part1`和`key_part2`可以为`NULL`。在下列条件中，`key_len`列指出了使用了两个索引列的长度

  ```
  key_part1 >= 1 AND key_part2 < 2
  ```

  但事实上，这个条件会被转换成这样：

  ```
  key_part1 >= 1 AND key_part2 IS NOT NULL
  ```

  > 定义字段时尽量定义为not null

**The Range Access Method for Single-Part Indexes**部分描述了优化器在单列索引中如何组合和去除范围条件确定的区间。类似的步骤在多列索引中同样适用



##### Equality Range Optimization of Many-Valued Comparisons



考虑这个例子，当`col_name`是一个索引列时：

```
col_name IN(val1, ..., valN)
col_name = val1 OR ... OR col_name = valN
```

上面两个表达式如果`col_name`与其中一个值相等则返回true。这些比较等同于范围比较（这里的范围比较指的是col_name = val1 这样的单值比较，上述的表达式存在多个范围比较）。优化器是根据以下规则判断按照范围比较条件读取满足条件的行的代价：



* 如果`col_name`是唯一索引，由于每个给定的值最多只有一行，所以每个范围比较的花费是1。

  > n个值则是n

  

* 如果`col_name`不是唯一索引，优化器会使用索引深入（index dive）或者索引统计信息（index statistics）来确定每个范围的行数。

  > index dive:暂时翻译成索引深入。 查询索引项个数得到精确的行数，速度较慢，但评估的行数精确
  >
  > index statistics：使用索引的统计信息进行估算，速度快但没有index dive精确



对于索引深入（index dives），优化器会在每个范围比较前后做一次dive，使用范围内（索引值对应的索引项）的行数作为评估。举个例子，表达式`col_name IN (10, 20, 30)`有3个等值范围比较。针对每个范围，优化器会做两次dive用于生成行数评估。每一对dive都可以估计出有给定值的行数。 



索引深入提供精确的行数评估，但是随着表达式中比较值的数量增加，优化器需要更长的时间来生成行数估计。使用索引统计信息评估没有索引深入那么精确 ，但在比较值数量较大时，会表现的比较快



使用`eq_range_index_dive_limit`系统变量可以设置行评估策略切换时的表达式式值个数。假如设置为`N`，那么当表达式后的值个数不大于N时，则使用index dives；如果设置为0，则禁用index dieves

可以使用`ANALYZE TABLE`来更新索引统计数据



即使在索引潜水可能被使用的情况下，它们也会被跳过以满足所有这些条件的查询：

*   当单个索引使用`FORCE INDEX`时。其思想是当强制使用索引时，那么就获取不到索引深入的额外开销了
*   当索引不是唯一索引且不是`FULLTEXT`索引。
*   没有子查询存在
*   没有`DISTINCT`, `GROUP BY`, `ORDER BY`存在

当满足上述的跳过深入条件时，只有在单表查询时才会跳过索引深入，在多表查询中将不会生效。

> 这段可能翻译的有点不准确，查了很久资料也理解这么做的原因



##### Limiting Memory Use for Range Optimization

可以使用`range_optimizer_max_mem_size`系统变量来控制范围优化使用的内存

* 设置为 0 表示没有限制

* 当值大于0时，优化器会追踪范围访问方法消耗的内存。如果使用的内存超过了限定的值，那么范围访问方法将会被禁止，而是考虑使用其他的访问方法（包括全表扫描）。这可能不是最优的方案。如果发生这种情况，会出现下面的警告

  ```
  Warning    3170    Memory capacity of N bytes for
                     'range_optimizer_max_mem_size' exceeded. Range
                     optimization was not done for this query.
  ```



当单个查询使用的范围优化内存超过设置的值，优化器会返回到不太理想的计划中。增加`range_optimizer_max_mem_size`的值可以提高性能



为了评估执行范围条件时所需要的内存，可以使用以下准则：



* 对于像下面的简单查询，只有一个候选键存在的情况下，每个`or`大约需要使用到230 bytes

  ```
  SELECT COUNT(*) FROM t
  WHERE a=1 OR a=2 OR a=3 OR .. . a=N;
  ```

* 对于像下面的查询，每个`AND`使用大约 125 bytes

  ```
  SELECT COUNT(*) FROM t
  WHERE a=1 AND b=1 AND c=1 ... N;
  ```

* 对于使用`IN()`的查询：

  ```
  SELECT COUNT(*) FROM t
  WHERE a IN (1,2, ..., M) AND b IN (1,2, ..., N);
  ```

  对于每个`IN()`里面的值，可以视为使用`OR`组合起来。如果有多个 `IN()`列表，那么两个`IN()`列表中的任意两个值之间都视为使用`OR`组合起来。因为两个`IN()`列表则有 _`M`_ × _`N`_个OR

在5.7.11之前，每个`OR`组合消耗的字节数更搞，大约700 bytes.



##### Range Optimization of Row Constructor Expressions

当查询满足这种形式时，优化器会开启范围扫描访问方法：

```
SELECT ... FROM t1 WHERE ( col_1, col_2 ) IN (( 'a', 'b' ), ( 'c', 'd' ));
```

在以前的版本中，如果要开启范围扫描，查询必须要写成这样

```
SELECT ... FROM t1 WHERE ( col_1 = 'a' AND col_2 = 'b' )
OR ( col_1 = 'c' AND col_2 = 'd' );
```

当优化器使用范围扫描时，查询必须满足这些条件

*   只能使用`IN()`，不能使用`NOT IN()`
*   在`IN()`的左边，只能使用表的列引用，不能使用常量或者子查询
*   在`IN()`的右边，行构造器只能使用运行时常量，他们要么是字面量，要么是本地列引用。他们在执行期间绑
*   在`IN()`的右边，有超过一个行构造器

关于优化器和行构造器的信息, 请查看 [Section 8.2.1.18, “Row Constructor Expression Optimization”](https://dev.mysql.com/doc/refman/5.7/en/row-constructor-optimization.html "8.2.1.18 Row Constructor Expression Optimization")