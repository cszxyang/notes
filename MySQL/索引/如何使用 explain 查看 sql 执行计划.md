MySQL 优化器会根据表、列、索引信息以及 WHERE 子句中的条件来考虑使用何种技术有效地执行 SQL 查询，比如，无需读取所有行即可执行对巨大表的查询； 无需比较行的每个组合即可执行涉及多个表的 join 等等。 优化器选择执行最有效查询的一组操作称为“查询执行计划”，也称为 EXPLAIN 计划。作为 MySQL 的使用者，掌握执行计划的各个方面的是很有必要，这将使你清楚自己的查询是否已被优化，并且如果发现一些低效的操作，则能给你带来新的思考，迫使你去学习 SQL 语法和索引技术以改进计划。

先交代一下实验环境

```mysql
mysql> select version();
+-------------------------+
| version()               |
+-------------------------+
| 5.7.29-0ubuntu0.18.04.1 |
+-------------------------+
```

建个测试表，同时使用存储过程造点数据

```mysql
create table test_explain (
	id int(11) not null,
    a int(11) not null,
    b int(11) not null,
    c int(11) not null,
    primary key(id),
    key idx_ab (a,b),
    key idx_b (b)
) engine = InnoDB default charset = utf8;

drop procedure idata;
delimiter ;;
create procedure idata()
    begin
        declare i int;
        set i=1;
        while(i <= 1000) do
        insert into test_explain values(i, RAND() * 100, RAND() * 100, i);
        set i=i+1;
        end while;
    end;;
delimiter ;
call idata();
```

查看下表行数

```mysql
mysql> select count(*) from test_explain;
+----------+
| count(*) |
+----------+
|     1000 |
+----------+
1 row in set (0.00 sec)
```

随便写几条查询进行执行计划分析

### join type

type 列中的 type 其实是 `join type`，它描述了表间是如何联合在一起的，在 Json 格式的 EXPLAIN 输出中使用`access_key`字段表示。看到这里我有个疑问，既然描述的是的表间关系，那对于一个只涉及单表的查询，这些规则会不会有什么差异呢？

下面是 type 列可能的所有取值，从好到坏依次解释。

### system

> The table has only one row (= system table). This is a special case of the `const` join type.

当使用索引查询时最多只有一行匹配时就会出现 system，这是 `const` 的一种特殊情况。

实验了很多次，最多也就出现 `const` ，难道是因为版本的问题？

### const

> The table has at most one matching row, which is read at the start of the query. Because there is only one row, values from the column in this row can be regarded as constants by the rest of the optimizer. `const` tables are very fast because they are read only once.
>
> `const` is used when you compare all parts of a `PRIMARY KEY` or `UNIQUE index` to constant values.

当使用全列主键或者唯一索引时，就会使用 `const` 类型，比如下面这两种查询

```mysql
# 单一主键
SELECT * FROM tbl_name WHERE primary_key=1;
# 联合主键
SELECT * FROM tbl_name WHERE primary_key_part1=1 AND primary_key_part2=2;
```

因为表中最多只有一个匹配行，所以优化器的其余部分可以将这一行中列的值视为常量。 `const` 表非常快，因为它们只读取一次。

### ref

```mysql
mysql> explain select b from test_explain where b = 1;
```

查看存在的行数，和上面 expain 显示的 rows 一样

```mysql
mysql> select count(*) from test_explain where b=1;
```

> All rows with matching index values are read from this table for each combination of rows from the previous tables. `ref` is used if the join uses only a leftmost prefix of the key or if the key is not a `PRIMARY KEY` or `UNIQUE` index (in other words, if the join cannot select a single row based on the key value). If the key that is used matches only a few rows, this is a good join type.

含有与索引值匹配的所有行将被从该表中读取出来，然后与前一个表进行 join，当仅用到非主键索引或非唯一索引的最左前缀进行查询时，就会出现 ref ，换言之，join 时使用索引值查询如果不能只定位到一行数据时将会出现 ref。如果通过索引查询得到的只有几行，那 join 的性能也是很好的。



参考

- [MySQL 5.7 官方手册 - Understanding the Query Execution Plan](https://dev.mysql.com/doc/refman/5.7/en/execution-plan-information.html)