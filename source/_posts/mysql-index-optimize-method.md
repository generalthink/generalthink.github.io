---
title: mysql-index-optimize-method
date: 2024-09-23 10:03:44
tags: [mysql, 索引优化, join, explain]
---

今天遇到的一个问题
SELECT
            id,
            name,
            service_line_id serviceLineId,
            company_id companyId,
            source,
            form_type formType,
            form_config formConfig,
            columns_config columnsConfig
        FROM
            sys_published_form_config
        WHERE
            del_flag = '0' AND status = '0' AND source IN
            <foreach collection="sources" item="source" open="(" separator="," close=")">
                #{source}
            </foreach>

source是bitInt, del_flag,status都是bit类型的数据

最开始尝试添加了索引idx_source_status_del_flag,但是同样的sql突然查询不出来数据了(mysql版本8.0.23).

但是如果把其中部分sql换成 del_flag = b'0' AND status = b'0' 或者 del_flag = 0 AND status = 0就可以查询出来数据了。

所以字段最好不要使用bit类型的 


SELECT * FROM your_table WHERE bit_column = 0;
SHOW WARNINGS;

SELECT * FROM your_table WHERE bit_column = '0';
SHOW WARNINGS;


原始sql:

```sql
 SELECT DISTINCT
            a.user_id userId,
            a.name userName
        FROM
            sc_request r
            LEFT JOIN sc_request_assign_auditor a ON a.request_id = r.id
        WHERE
            a.user_id IS NOT NULL AND a.type = #{auditorType} AND r.id IN(
                SELECT id FROM sc_request WHERE application_type = #{applicationTypeNew} AND status IN
                <foreach item="s" collection="newRequestStatus" open="(" separator="," close=")">
                    #{s}
                </foreach>
                UNION
                SELECT id FROM sc_request WHERE application_type = #{applicationTypeAmend} AND status IN
                <foreach item="s" collection="amendRequestStatus" open="(" separator="," close=")">
                    #{s}
                </foreach>
            )

```


优化后的sql

```sql
SELECT a.user_id userId, a.name userName FROM sc_request r JOIN sc_request_assign_auditor a ON a.request_id = r.id WHERE a.user_id IS NOT NULL AND a.type = 1 AND ( ( r.application_type = 1 AND r.status IN (1,2,3,4,5,6,7,8,9,10,11,12,13,14) ) or ( r.application_type = 2 AND r.status IN (2,3,4,5,6,7,8,9,10,11,12,13,14) ) ) group by a.user_id order by a.user_id

```

索引建议：
针对 sc_request_assign_auditor 表（别名 a）
联合索引: 创建 (type, user_id, request_id) 的联合索引，并覆盖 name 字段以实现覆盖索引：
  CREATE INDEX idx_a_type_userid_rid_name ON sc_request_assign_auditor(type, user_id, request_id, name);
理由:
type = 1 是等值条件，作为索引首列可快速过滤数据。
user_id IS NOT NULL 虽然无法直接利用索引，但联合索引能加速关联查询。
request_id 是 JOIN 条件的关键字段，存放于索引中可加速连接操作。
name 包含在索引中可避免回表查询。
针对 sc_request 表（别名 r）
联合索引: 为 (application_type, status) 创建索引：
  CREATE INDEX idx_r_apptype_status ON sc_request(application_type, status);
理由:
WHERE 条件中的 application_type 和 status 是联合条件，索引首列 application_type 可快速定位到特定类型的记录，status 用于范围过滤




如果我使用EXPLAIN ANALYZE YOUR_QUERY_HERE语句得到的结果是不是就是这个sql语句的执行顺序，执行的时候顺序一定是从最内层开始然后到最外层的？

注意几点：

执行顺序：
通常情况下，执行计划是从内向外、从上到下解读的。最内层（最缩进）的操作通常先执行，然后逐步向外。

并非严格的自下而上：
虽然大体上是从内向外执行，但 MySQL 的查询执行器可能会在某些情况下调整执行顺序，特别是在使用嵌套循环连接（Nested Loop Join）时。

并行执行：
某些操作可能会并行执行，特别是在 MySQL 8.0 及以后的版本中，引入了哈希连接等可能并行执行的操作。

管道执行：
某些操作可能会以流水线（pipeline）方式执行，而不是严格按照层级顺序。

优化器重排：
查询优化器可能会重新排列某些操作的顺序，以提高效率。

实际执行时间：
EXPLAIN ANALYZE 提供的 "actual time" 指标显示了每个操作的实际执行时间，这有助于理解查询中的瓶颈所在。

循环（Loops）：
注意输出中的 "loops" 值，它表示某个操作被执行的次数。

成本估算：
"cost" 值是优化器的成本估算，不一定完全反映实际执行时间。

解读示例：
以你之前提供的 EXPLAIN ANALYZE 输出为例：


-> Limit: 300 row(s)  (cost=22406.58 rows=300) (actual time=308.782..311.080 rows=300 loops=1)
    -> Nested loop left join  (cost=22406.58 rows=56214) (actual time=308.781..311.056 rows=300 loops=1)
        -> Nested loop left join  (cost=13992.46 rows=48673) (actual time=308.763..310.361 rows=300 loops=1)
            -> Sort: i.latest_submit_date DESC  (cost=6698.20 rows=48673) (actual time=308.738..308.820 rows=300 loops=1)
                -> Filter: ((i.`status` <> 'Draft') and (i.del_flag = '0'))  (cost=6698.20 rows=48673) (actual time=0.040..96.214 rows=48345 loops=1)
                    -> Table scan on i  (cost=6698.20 rows=48673) (actual time=0.034..73.519 rows=51397 loops=1)
执行顺序大致如下：

表扫描 (Table scan on i)
过滤 (Filter)
排序 (Sort)
嵌套循环左连接 (Nested loop left join)
限制结果集 (Limit)
但请注意，某些操作（如连接）可能会交错执行，而不是严格按照这个顺序。

总之，EXPLAIN ANALYZE 提供了非常有价值的信息来理解查询的执行过程，但解读时需要考虑 MySQL 查询执行的复杂性。它是一个强大的工具，用于识别性能瓶颈和优化机会。


在 EXPLAIN ANALYZE 的输出中，actual time 字段的格式为 "start_time..end_time"，其中的点号 (..) 表示一个范围。这两个数值的含义如下：

第一个数值 (start_time)：表示操作开始的时间点。
第二个数值 (end_time)：表示操作结束的时间点。
这两个时间值的单位是毫秒（ms）。

在你提供的例子中：


actual time=308.738..308.820
这表示：

操作开始于查询执行后的 308.738 毫秒
操作结束于查询执行后的 308.820 毫秒
你可以通过计算这两个值的差来得到该操作的持续时间：
308.820 - 308.738 = 0.082 毫秒

这种表示方法提供了两种重要信息：

操作的持续时间（通过计算差值）
操作在整个查询执行过程中的相对位置（通过观察开始时间）
几点需要注意的地方：

对于一些快速操作，开始时间和结束时间可能是相同的。

对于某些操作（如嵌套循环），第一个数字可能表示第一行结果的产生时间，第二个数字表示最后一行结果的产生时间。

这些时间值是累积的，所以你会看到内层操作的时间通常小于外层操作的时间。

时间单位是毫秒，所以小的差异（如 0.082 毫秒）在大多数情况下并不显著，除非是在高度优化的系统中或者这个操作被执行了很多次。

### 索引基数

要查看索引的基数（Cardinality），你可以使用以下几种方法：

使用 SHOW INDEX 命令：


SHOW INDEX FROM table_name;
这个命令会显示表的所有索引信息，包括每个索引的基数。例如：


SHOW INDEX FROM tc_app_info;
查询 INFORMATION_SCHEMA：


SELECT INDEX_NAME, COLUMN_NAME, CARDINALITY
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = 'your_database_name'
AND TABLE_NAME = 'your_table_name'
ORDER BY INDEX_NAME, SEQ_IN_INDEX;
将 'your_database_name' 和 'your_table_name' 替换为你的实际数据库和表名。


需要注意的是：

基数是一个估计值，不是精确值。
基数表示索引中唯一值的数量。基数越高，索引的选择性就越好。
对于复合索引，基数表示的是所有索引列组合的唯一值数量。
基数会随着数据的变化而变化，MySQL 会定期更新这个值。
解读基数：

对于单列索引，如果基数接近表中的行数，说明这个索引的选择性很好。
对于复合索引，基数应该等于或接近表中的行数，这表明索引的组合具有很好的选择性。
如果基数很低（相对于表的大小），可能意味着索引的选择性不佳，可能需要重新考虑索引策略。
通过查看和比较不同索引的基数，你可以更好地理解哪些索引可能更有效，从而帮助优化查询性能。


### 索引基数

对于复合索引，SHOW INDEX 命令会为索引中的每个列显示一行，每行都有一个基数值。这些基数值的含义如下：

第一个列的基数：表示该列的唯一值数量的估计值。

第二个列的基数：表示前两个列组合的唯一值数量的估计值。

第三个列的基数：表示前三个列组合的唯一值数量的估计值。

以此类推...

所以，你看到的不是"每增加一个字段，基数的变化情况"，而是"到这个字段为止，所有前面字段组合的唯一值估计数"。

举个例子，假设有一个复合索引 (A, B, C)：


+------------+------------+-----------+-------------+
| Column_name| Cardinality| Index_type| Column_index|
+------------+------------+-----------+-------------+
| A          |       1000 | BTREE     |           1 |
| B          |       5000 | BTREE     |           2 |
| C          |      10000 | BTREE     |           3 |
+------------+------------+-----------+-------------+
这表示：

列 A 大约有 1000 个唯一值
列 A 和 B 的组合大约有 5000 个唯一值
列 A、B 和 C 的组合大约有 10000 个唯一值
重要注意事项：

这些都是估计值，不是精确值。

基数通常在表发生重大变化时更新，或者通过 ANALYZE TABLE 命令手动更新。

对于复合索引，后面列的基数通常大于或等于前面列的基数。

基数值对于优化器选择索引非常重要。高基数通常意味着索引的选择性好。

在某些情况下，你可能会看到基数为 NULL，这通常意味着 MySQL 还没有为该索引收集统计信息。