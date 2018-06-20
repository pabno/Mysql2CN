
# Index Merge Optimization

索引合并访问方法使用多个range扫描来检索行并且把检索得到的结果合并成一个。这种访问方法只会合并单个表的扫描结果，而不会合并多个表的扫描结果。这种合并会产生基础扫描结果的交集，并集或者并集的交集

可能使用索引合并的例子：

```
SELECT * FROM tbl_name WHERE key1 = 10 OR key2 = 20;

SELECT * FROM tbl_name
  WHERE (key1 = 10 OR key2 = 20) AND non_key = 30;

SELECT * FROM t1, t2
  WHERE (t1.key1 IN (1,2) OR t1.key2 LIKE 'value%')
  AND t2.key1 = t1.some_col;

SELECT * FROM t1, t2
  WHERE t1.key1 = 1
  AND (t2.key1 = t1.some_col OR t2.key2 = t1.some_col2);
```

在`EXPLAIN`的输出中，使用索引合并方法时`type`列会出现`index_merge`。在这个例子中`key`列列出了使用了哪些索引，`key_len`列出了这些索引使用的长度



说明

索引合并优化算法有以下已知的缺陷： 

- 如果查询的`WHERE`子句有较深的`AND`/`OR`嵌套，Mysql则不会选择最优的计划，可以尝试以下的同义转换：

  ```
  (x AND y) OR z => (x OR z) AND (y OR z)
  (x OR y) AND z => (x AND z) OR (y AND z)
  ```

- 索引合并不适用于full-text索引



索引合并访问方法有多种算法，在`EXPALIN`的输出中，他们会显示在`Extra`列中：

- `Using intersect(...)`
- `Using union(...)`
- `Using sort_union(...)`



下面的部分描述了这些算法更多的细节。优化器会基于可选的成本预估，在索引合并算法和其他访问方法之间选择



是否使用索引合并取决于`optimizer_switch`系统变量中`index_merge`, `index_merge_intersection`, `index_merge_union`, `index_merge_sort_union`标识的值。默认情况下这些表示都是`on`。为了只开启特定的算法，可以设置`index_merge`为`off`，同时启用需要的算法

- [The Index Merge Intersection Access Algorithm](https://dev.mysql.com/doc/refman/5.7/en/index-merge-optimization.html#index-merge-intersection)
- [The Index Merge Union Access Algorithm](https://dev.mysql.com/doc/refman/5.7/en/index-merge-optimization.html#index-merge-union)
- [The Index Merge Sort-Union Access Algorithm](https://dev.mysql.com/doc/refman/5.7/en/index-merge-optimization.html#index-merge-sort-union)

## The Index Merge Intersection Access Algorithm

这种访问算法适用于`WHERE`子句可以转化成多个使用`AND`组合的范围条件，且这些范围条件满足一下任意一条：

- 索引的所有列都会被使用到（无论是单列索引还是多列索引）

  ```
  key_part1 = const1 AND key_part2 = const2 ... AND key_partN = constN
  ```

- 对于`InnoDB`来说，使用主键的任意范围条件

Examples:

```
SELECT * FROM innodb_table
  WHERE primary_key < 10 AND key_col1 = 20;

SELECT * FROM tbl_name
  WHERE (key1_part1 = 1 AND key1_part2 = 2) AND key2 = 2;
```

Index Merge intersection算法会同时扫描所有使用到的索引，并产生从合并索引扫描得到的行序列的交集

如果查询中的所有列都建立了索引，优化器将不会全表扫描。（在这个例子中，`EXPLAIN`输出中的`Extra`字段会包含`Using index`）。下面是一个例子：

```
SELECT COUNT(*) FROM t1 WHERE key1 = 1 AND key2 = 1;
```

如果查询中使用到的索引没有覆盖所有的列，当所有使用到的索引都满足范围扫描时则会进行全表扫描

在`InnoDB`里，如果其中一个merged condition使用了主键，则不会检索行，而是用来过滤使用其他条件检索的行



## The Index Merge Union Access Algorithm

这种算法与Index Merge intersection算法相似。该算法适用于`WHERE`子句可以转化成多个使用`OR`组合的范围条件，且这些范围条件满足一下任意一条：

- 索引的所有列都会被使用到（无论是单列索引还是多列索引）

  ```
  key_part1 = const1 AND key_part2 = const2 ... AND key_partN = constN
  ```

- 对于`InnoDB`来说，使用主键的任意范围条件

- 一个满足Index Merge intersection算法的条件

Examples:

```
SELECT * FROM t1
  WHERE key1 = 1 OR key2 = 2 OR key3 = 3;

SELECT * FROM innodb_table
  WHERE (key1 = 1 AND key2 = 2)
     OR (key3 = 'foo' AND key4 = 'bar') AND key5 = 5;
```

## The Index Merge Sort-Union Access Algorithm

该算法适用于`WHERE`子句可以转化成多个使用`OR`组合的范围条件，但是Index Merge union算法不适用的情况

Examples:

```
SELECT * FROM tbl_name
  WHERE key_col1 < 10 OR key_col2 < 20;

SELECT * FROM tbl_name
  WHERE (key_col1 > 10 OR key_col2 = 20) AND nonkey_col = 30;
```

 sort-union算法与union 算法不同的地方在于，sort-union需要先获取所有行的ID，然后在返回任意行前进行一次排序

 