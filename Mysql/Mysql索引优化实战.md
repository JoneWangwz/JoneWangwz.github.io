# Mysql索引优化实战

## 目标

- 索引下推优化详解
- MySQL优化器索引选择探究
- 索引优化order by 与group by
- Using filesort文件排序详解
- 索引设计原则与实战

### 1.联合索引第一个字段用范围不走索引

```sql
EXPLAIN SELECT * FROM employees WHERE name > 'LiLei' AND age = 22 AND position ='manager';
```

![image-20211030185959739](../image/image-20211030185959739.png)

联合索引第一个字段就使用范围查找索引，MySQL可能觉得一个字段就适用范围查找，结果集应该非常大，因此选择走全表扫描。

### 2.强制走索引

```sql
EXPLAIN SELECT * FROM employees force index(idx_name_age_position) WHERE name > 'LiLei' AND age = 22 AND position ='manager';
```

![image-20211030190220675](../image/image-20211030190220675.png)

虽然使用了强制索引，扫描的行数看上去少了不少，但最终查询的效率也不一定比全表扫描的效率高，可能涉及的到的回表效率不高。

测试

```sql
关闭缓存
set global query_cache_size=0;
set global query_cache_type=0;

SELECT * FROM employees WHERE name > 'LiLei';
0.422s
SELECT * FROM employees force index(idx_name_age_position) WHERE name > 'LiLei';
0.656s
```

### 3.覆盖索引优化

```sql
EXPLAIN SELECT name,age,position FROM employees WHERE name > 'LiLei' AND age = 22 AND position ='manager';
```

![image-20211030191040477](../image/image-20211030191040477.png)

此处优化使用覆盖索引，查询字段使用索引中的列，这样就不需要回表进行查询，效率提升。

### 4.in 和or在数据量比较大的情况下会走索引，在数据量比较小的情况下会走全表扫描。

```sql
EXPLAIN SELECT * FROM employees WHERE name in ('LiLei','HanMeimei','Lucy') AND age = 22 AND position='manager';
```

![image-20211030191816266](../image/image-20211030191816266.png)

```sql
EXPLAIN SELECT * FROM employees WHERE (name = 'LiLei' or name = 'HanMeimei') AND age = 22 AND position='manager';
```

![image-20211030191855620](../image/image-20211030191855620.png)

创建一个表employees_copy和employees的表结构一样，但是数据量比较少。

```sql
EXPLAIN SELECT * FROM employees_copy WHERE name in ('LiLei','HanMeimei','Lucy') AND age = 22 AND position='manager';
```

![image-20211030192139486](../image/image-20211030192139486.png)

```sql
EXPLAIN SELECT * FROM employees_copy WHERE (name = 'LiLei' or name = 'HanMeimei') AND age = 22 AND position='manager';
```

![image-20211030192121502](../image/image-20211030192121502.png)

根据上面的对比可以看出，in和or检索的数据范围和整个表的数据也是MySQL是否选择走索引的一个重要原因。

### 5.like kk%一般都会走索引

```sql
EXPLAIN SELECT * FROM employees WHERE name like 'LiLei%' AND age = 22 AND position ='manager';
```

![image-20211030192417281](../image/image-20211030192417281.png)

```sql
EXPLAIN SELECT * FROM employees_copy WHERE name like 'LiLei%' AND age = 22 AND position ='manager';
```

![image-20211030192518892](../image/image-20211030192518892.png)

通过上面的例子可以看出来，employees和employees_copy两张表都走了索引。

## 索引下推

对于辅助的联合索引，正常情况下遵照最左前缀原则执行```SELECT * FROM employees WHERE name like 'LiLei%' AND age = 22 AND position ='manager';```，这种情况下只会走索引的第一列，而后面的数据都是无序的，无法很好的利用索引。

在MySQL5.6之前的版本，只会走索引列name是”LILEI“开头的数据，然后再拿这些索引所对应的主键逐个回表，根据主键索引找到对应的数据，在逐渐比较age和position列的数据。

在MySQL5.6引入索引下推优化，可以在索引遍历的过程中，对索引的所有字段都进行判断，筛选完成以后，在进行回表查询，这样可以有效的减少回表的次数。在使用索引下推以后，上面的查询在匹配到name是”LILEI“后，还会对age和position进行匹配，当发现符合条件以后，再根据主键id回表找到对应数据。

索引下推会减少回表次数，对InnoDB搜索引擎的表索引下推只针对二级索引，InnoDB的主键索引的叶子节点上面保存的全行数据，这时候使用索引下推并不会减少查询的全行数据。

**为什么范围查找没有使用索引下推进行优化？**

MySQL可能认为范围查找的范围比较大，像like kk% 在大多数情况下数据集还是比较小的，所以MySQL给like kk% 使用了索引下推，在实际情况中like kk% 并非全都会走索引下推。

## MySQL如何选择合适的索引

```sql
 EXPLAIN select * from employees where name > 'a';
```

![image-20211030200605530](../image/image-20211030200605530.png)

如果name索引需要遍历name字段联合索引树，然后还需要根据遍历出来的主键再回聚簇索引中寻找数据，成本对比全表扫描还高，可以用覆盖索引进行优化，这样只需要遍历name字段所在的索引树就可以了：

```sql
EXPLAIN select name,age,position from employees where name > 'a' ;
```

![image-20211030200919481](../image/image-20211030200919481.png)

```sql
 EXPLAIN select * from employees where name > 'zzz' ;
```

![image-20211030201045349](../image/image-20211030201045349.png)

针对上面的name > 'a' 和name > 'zzz'，MySQL执行了不一样的结果，MySQL可以使用trace工具进行查询，开启trace工具会影响MySQL的性能，所以只能作为临时分析sql语句使用，用完之后立即关闭。

```sql
set session optimizer_trace="enabled=on",end_markers_in_json=on; 
```

执行语句

```sql
select * from employees where name > 'a' order by position;
SELECT * FROM information_schema.OPTIMIZER_TRACE;
```

```json
trace语句
{
  "steps": [
    {
      "join_preparation": { ‐‐第一阶段：SQL准备阶段，格式化sql
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `employees`.`id` AS `id`,`employees`.`name` AS `name`,`employees`.`age` AS `age`,`employees`.`position` AS `position`,`employees`.`hire_time` AS `hire_time` from `employees` where (`employees`.`name` > 'a') order by `employees`.`position`"
          }
        ] /* steps */
      } /* join_preparation */
    },
    {
      "join_optimization": {‐‐第二阶段：SQL优化阶段
        "select#": 1,
        "steps": [
          {
            "condition_processing": {‐‐条件处理
              "condition": "WHERE",
              "original_condition": "(`employees`.`name` > 'a')",
              "steps": [
                {
                  "transformation": "equality_propagation",
                  "resulting_condition": "(`employees`.`name` > 'a')"
                },
                {
                  "transformation": "constant_propagation",
                  "resulting_condition": "(`employees`.`name` > 'a')"
                },
                {
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "(`employees`.`name` > 'a')"
                }
              ] /* steps */
            } /* condition_processing */
          },
          {
            "substitute_generated_columns": {
            } /* substitute_generated_columns */
          },
          {
            "table_dependencies": [‐‐表依赖详情
              {
                "table": "`employees`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ] /* depends_on_map_bits */
              }
            ] /* table_dependencies */
          },
          {
            "ref_optimizer_key_uses": [
            ] /* ref_optimizer_key_uses */
          },
          {
            "rows_estimation": [‐‐预估表的访问成本
              {
                "table": "`employees`",
                "range_analysis": {
                  "table_scan": { ‐‐全表扫描情况
                    "rows": 99926,‐‐扫描行数
                    "cost": 20340‐‐查询成本
                  } /* table_scan */,
                  "potential_range_indexes": [‐‐查询可能使用的索引
                    {
                      "index": "PRIMARY",‐‐主键索引
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "idx_name_age_position",‐‐辅助索引
                      "usable": true,
                      "key_parts": [
                        "name",
                        "age",
                        "position",
                        "id"
                      ] /* key_parts */
                    }
                  ] /* potential_range_indexes */,
                  "setup_range_conditions": [
                  ] /* setup_range_conditions */,
                  "group_index_range": {
                    "chosen": false,
                    "cause": "not_group_by_or_distinct"
                  } /* group_index_range */,
                  "analyzing_range_alternatives": {‐‐分析各个索引使用成本
                    "range_scan_alternatives": [
                      {
                        "index": "idx_name_age_position",
                        "ranges": [ ‐‐索引使用范围
                          "a < name"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false, ‐‐使用该索引获取的记录是否按照主键排序
                        "using_mrr": false,
                        "index_only": false,‐‐是否使用覆盖索引
                        "rows": 49963,‐‐索引扫描行数
                        "cost": 59957,‐‐索引使用成本
                        "chosen": false,‐‐是否选择该索引
                        "cause": "cost"
                      }
                    ] /* range_scan_alternatives */,
                    "analyzing_roworder_intersect": {
                      "usable": false,
                      "cause": "too_few_roworder_scans"
                    } /* analyzing_roworder_intersect */
                  } /* analyzing_range_alternatives */
                } /* range_analysis */
              }
            ] /* rows_estimation */
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ] /* plan_prefix */,
                "table": "`employees`",
                "best_access_path": { ‐‐最优访问路径
                  "considered_access_paths": [ ‐‐最优访问路径
                    {
                      "rows_to_scan": 99926,
                      "access_type": "scan", ‐‐访问类型：为scan，全表扫描
                      "resulting_rows": 99926,
                      "cost": 20338,
                      "chosen": true,‐‐确定选择
                      "use_tmp_table": true
                    }
                  ] /* considered_access_paths */
                } /* best_access_path */,
                "condition_filtering_pct": 100,
                "rows_for_plan": 99926,
                "cost_for_plan": 20338,
                "sort_cost": 99926,
                "new_cost_for_plan": 120264,
                "chosen": true
              }
            ] /* considered_execution_plans */
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": "(`employees`.`name` > 'a')",
              "attached_conditions_computation": [
              ] /* attached_conditions_computation */,
              "attached_conditions_summary": [
                {
                  "table": "`employees`",
                  "attached": "(`employees`.`name` > 'a')"
                }
              ] /* attached_conditions_summary */
            } /* attaching_conditions_to_tables */
          },
          {
            "clause_processing": {
              "clause": "ORDER BY",
              "original_clause": "`employees`.`position`",
              "items": [
                {
                  "item": "`employees`.`position`"
                }
              ] /* items */,
              "resulting_clause_is_simple": true,
              "resulting_clause": "`employees`.`position`"
            } /* clause_processing */
          },
          {
            "reconsidering_access_paths_for_index_ordering": {
              "clause": "ORDER BY",
              "steps": [
              ] /* steps */,
              "index_order_summary": {
                "table": "`employees`",
                "index_provides_order": false,
                "order_direction": "undefined",
                "index": "unknown",
                "plan_changed": false
              } /* index_order_summary */
            } /* reconsidering_access_paths_for_index_ordering */
          },
          {
            "refine_plan": [
              {
                "table": "`employees`"
              }
            ] /* refine_plan */
          }
        ] /* steps */
      } /* join_optimization */
    },
    {
      "join_execution": {‐‐第三阶段：SQL执行阶段
        "select#": 1,
        "steps": [
          {
            "filesort_information": [
              {
                "direction": "asc",
                "table": "`employees`",
                "field": "position"
              }
            ] /* filesort_information */,
            "filesort_priority_queue_optimization": {
              "usable": false,
              "cause": "not applicable (no LIMIT)"
            } /* filesort_priority_queue_optimization */,
            "filesort_execution": [
            ] /* filesort_execution */,
            "filesort_summary": {
              "rows": 100003,
              "examined_rows": 100003,
              "number_of_tmp_files": 30,
              "sort_buffer_size": 262056,
              "sort_mode": "<sort_key, packed_additional_fields>"
            } /* filesort_summary */
          }
        ] /* steps */
      } /* join_execution */
    }
  ] /* steps */
}
结论：全表扫描的成本低于索引扫描，所以mysql最终选择全表扫描
select * from employees where name > 'zzz' order by position;
SELECT * FROM information_schema.OPTIMIZER_TRACE;

查看name > 'zzz' 的trace可以看出，索引扫描的成本低于全表扫描，可能是因为name > 'zzz' 的数据并没有存在，因此不存在回表的可能。
```

## 常见SQL深入优化

order by和group by优化

Case1:

```sql
 EXPLAIN SELECT * FROM employees WHERE name = 'LiLei' and position ='dev' ORDER BY age;
```

![image-20211030205836009](../image/image-20211030205836009.png)

这里利用最左前缀法则，可以发现name使用了索引字段，其实age也使用了索引字段，当使用order by时，此时extra没有体现Using filesort，意味着这里使用的是索引排序而不是外部的排序，因此age走了索引；同时，key_len的值只展示了name的长度，这是因为key_len只统计的是where后面”=“的索引长度。这里的position字段是没有走索引的。

Case2:

```sql
EXPLAIN SELECT * FROM employees WHERE name = 'LiLei' ORDER BY position;
```

![image-20211030210549331](../image/image-20211030210549331.png)

根据最左前缀法则，只使用了name索引，由于使用了position进行排序，跳过了age，因此并没有使用position索引，使用order by，但没有使用position索引排序，因此就会使用Using filesort进行排序。

Case3:

```sql
  EXPLAIN SELECT * FROM employees WHERE name = 'LiLei' ORDER BY age,position;
```

![image-20211030211116485](../image/image-20211030211116485.png)

查找用到了name索引，但是排序使用的是age和position索引。extra列中没有出现Using filesort，可以看出age和postion都是使用的索引排序。

Case4:

```sql
  EXPLAIN SELECT * FROM employees WHERE name = 'LiLei' ORDER BY position,age;
```

![image-20211030211427047](../image/image-20211030211427047.png)

当order by的顺序发生变化时，就会发现只有在查找时，使用了name索引，而在排序的过程中，使用了Using filesort，可以看出，order by的顺序是不会发生改变的，这与创建索引的顺序不一致。

Case5:

```sql
  EXPLAIN SELECT * FROM employees WHERE name = 'LiLei' and age = 18 ORDER BY position,age;
```

![image-20211030211702450](../image/image-20211030211702450.png)

这个SQL语句在查找时，选择了name和age索引，排序的时候的顺序时position，age，但是由于age=18已经是常量了，对于age的排序已经没有任何意义，这部分已经被优化，所以此时的排序只有position。

Case6：

```sql
  EXPLAIN SELECT * FROM employees WHERE name = 'LiLei' ORDER BY age asc,position DESC;
```

![image-20211030212157482](../image/image-20211030212157482.png)

在查找时选择了name索引，在排序的时候age选择升序，符合索引的排序，position则是按照降序排列的，因此和索引排序不能保持一致，所以使用Using filesort。

Case7：

```sql
   EXPLAIN SELECT * FROM employees WHERE name in ('LiLei','zhuge') ORDER BY age ,position;
```

![image-20211030212502719](../image/image-20211030212502719.png)

对于排序而言，多个相等条件也是范围查询。

Case8：

```sql
EXPLAIN SELECT * FROM employees WHERE name >'a' ORDER BY name;
```

![image-20211030213112799](../image/image-20211030213112799.png)

这里使用的是Using filesort，进行了全表扫描，原因可能是数据量太大，MySQL优化器认为全表扫描更优。filesort是对聚簇索引进行的排序，而不是对二级索引。

优化：

```sql
EXPLAIN SELECT name,age,position FROM employees WHERE name >'a' ORDER BY name;
```

![image-20211030213449043](../image/image-20211030213449043.png)

这里直接缩小查询范围，仅仅将数据字段锁定在二级索引树中，使用覆盖索引进行优化。

## 优化总结：

1.MySQL支持的排序扫描的方式有两种index和filesort，**Using index是MySQL扫描索引本身进行的排序，Using filesort则是对聚簇索引进行的全表扫描排序**。因此，index效率高，而filesort则是效率比较低。

2.order by下面两种情况会使用Using index

1）order by语句索引使用最左前缀法则

2）使用where子句和order by子句条件列组合满足索引最左前列原则

3.尽量在索引列上完成排序，遵循索引建立(索引创建的顺序)时的最左前缀法则。

4.如果order by不在索引列上，就会产生filesort

5.能用覆盖索引尽量使用覆盖索引

6.group by和order by很类似，其实质先进行排序再进行分组，遵照索引的最左前缀法则。对于group by如果不需要排序 ，加上order by null禁止排序。注，where优先级高于having，能写在where中的限定条件就不要限定在having中。