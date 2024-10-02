# 性能优化

## 索引

三板斧：1、尽量走索引，2、选择合理的索引类型， 3、减少不必要的索引

### 分析执行计划

```sql
EXPLAIN SELECT * FROM your_table WHERE your_column = 'value';
EXPLAIN ANALYZE SELECT * FROM your_table WHERE your_column = 'value';
```

执行计划包含多个步骤，通常以树状结构展示，最顶层步骤是查询的最终操作（如 `Seq Scan`、`Index Scan`、`Nested Loop` 等），子步骤是执行的中间过程。 

| node_en                              | node_cn                  | des                                                          | note                                                         |
| ------------------------------------ | ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Seq Scan                             | 顺序扫描                 | PostgreSQL 逐行扫描整个表。这通常发生在没有索引的情况下。    | 顺序扫描适合小表，若表较大且没有索引，会导致性能问题。       |
| Index Scan                           | 索引扫描                 | PostgreSQL 使用索引查找符合条件的行，适合查询特定行或范围的场景。 | 索引扫描一般比顺序扫描更高效，但如果查询返回大量行，性能可能不如顺序扫描。 |
| Index Only Scan                      | 仅索引扫描               | PostgreSQL 只使用索引扫描，而不需要访问表中的数据。适合索引包含查询所需的所有列的情况。 | 它依赖于索引的完整性和`VACUUM`清理操作，否则可能需要额外的访问表来获取数据。 |
| Bitmap Index Scan & Bitmap Heap Scan | 位图索引扫描和位图堆扫描 | Bitmap 扫描是一种折衷，它首先通过索引生成一个位图，然后通过位图定位到表中所需的行。 | 适合返回大量行但仍有索引的情况，能够在索引和表之间取得较好的平衡。 |
| Nested Loop                          | 嵌套循环                 | PostgreSQL 使用嵌套循环连接两张表。通常用于较小的数据集或当连接的外表非常小且索引良好时。 | 对于大表，如果没有有效的索引，这种方式可能非常慢。           |
| Hash Join                            | 哈希连接                 | 通过构建哈希表来连接两个数据集，适合大表连接。               | 哈希连接对于大规模数据集上的等值连接（例如`=`）非常高效。    |
| Merge Join                           | 归并连接                 | 两个排序后的数据集可以通过归并排序的方式连接。               | 适用于已经排序的数据集或有排序索引的场景。                   |
| Sort                                 | 排序                     | PostgreSQL 需要对查询结果进行排序时，可能会使用该节点。      | 如果查询结果集很大且没有适合的索引，排序操作会很慢，尤其是涉及磁盘 I/O 时。 |

每个**节点**通常会包含以下字段：

- **Node Type**：执行计划中的操作类型，比如 `Seq Scan`、`Index Scan`、`Nested Loop` 等。

- **Cost**：`EXPLAIN` 计划中有两种成本值：

  - **Startup Cost**：执行该节点的启动开销，即开始返回第一行的时间。
  - **Total Cost**：返回所有结果行的总开销，表示该节点执行完成的成本。

  成本单位是基于相对 CPU 和 I/O 的消耗，而不是实际的时间单位。

- **Rows**：估算该节点返回的行数。这是基于表统计信息的预估值。

- **Width**：返回的行的字节大小。对列选择操作的估算大小。

当使用 `EXPLAIN ANALYZE` 时，会有两个额外的字段：

- **Actual Time**：执行每个节点的实际时间，通常表示为两个值：
  - **实际启动时间**：从开始执行到返回第一行的时间。
  - **实际总时间**：执行该节点所有操作所用的总时间。
- **Actual Rows**：节点实际处理的行数，与预估的 `Rows` 值进行对比。如果偏差很大，可能表明表统计信息不准确，影响查询计划的选择。

### 常用查看索引信息的sql

```sql
-- 查看某张表的所有索引
select * from pg_indexes where tablename = 'abc'

-- 查看某个索引的详细信息
SELECT *
FROM pg_stat_user_indexes
WHERE indexrelname = 'your_index_name';

-- 查看表上每个索引的大小
-- pg_relation_size(indexrelid)：返回索引占用的字节数。
-- pg_size_pretty：将字节数转换为人类可读的格式（如 MB、GB）。
SELECT indexname,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_indexes
JOIN pg_class ON pg_class.relname = pg_indexes.indexname
WHERE tablename = 'your_table_name';

-- 查看所有索引的总大小
SELECT pg_size_pretty(SUM(pg_relation_size(indexrelid))) AS total_index_size
FROM pg_indexes
WHERE tablename = 'your_table_name';

-- 查看索引使用情况统计
-- idx_scan：索引被扫描的次数。
-- idx_tup_read：通过索引读取的元组数。
-- idx_tup_fetch：通过索引实际获取的元组数。
SELECT indexrelname AS index_name, idx_scan AS number_of_scans, idx_tup_read AS tuples_read, idx_tup_fetch AS tuples_fetched
FROM pg_stat_user_indexes
WHERE relname = 'your_table_name';

-- 查看索引字段及排序方式
-- indisunique：是否为唯一索引。
-- indisprimary：是否为主键索引。
-- amname：索引方法（如 B-tree）。
-- order：索引的排序方式。
SELECT
    i.indrelid::regclass AS table_name,
    a.attname AS column_name,
    i.indisunique AS is_unique,
    i.indisprimary AS is_primary,
    am.amname AS index_method,
    CASE i.indisasc WHEN true THEN 'ASC' ELSE 'DESC' END AS order
FROM pg_index i
JOIN pg_class c ON c.oid = i.indexrelid
JOIN pg_am am ON c.relam = am.oid
JOIN pg_attribute a ON a.attnum = ANY(i.indkey) AND a.attrelid = i.indrelid
WHERE i.indexrelid = 'your_index_name'::regclass;

```

在PostgreSQL中执⾏CREATE INDEX命令时，可以使⽤**`CONCURRENTLY`**参数并行创建索引，使用CONCURRENTLY参数不会锁表，创建索引过程中不会阻塞表的更新、插⼊、删除操作。

由于PostgreSQL的MVCC内部机制，当运行⼤量的更新操作后，会有“索引膨胀”的现象，这时候可以通过CREATE INDEX CONCURRENTLY在不阻塞查询和更新的情况下，在线重新创建索引，创建好新的索引之后，再删除原先有膨胀的索引，减小索引尺寸，提高查询速度。



## 改写sql

1. CTE减少嵌套 （Common Table Expressions，简称CTE）
2. 减少子查询 
3. 物化视图、临时表
4. exits使用
5. 拆分sql
6. 改写sql
7. 防止类型转换失真



## 数据库参数

1. 精确统计信息
2. 干涉执行计划
3. 调整性能参数
4. pg_hint_plan

### **postgresql.conf配置文件**

使用sql语句查看全局配置，如 `show shared_buffers`;

**`shared_buffers`** 参数用于指定 PostgreSQL 使用的共享内存缓冲区的大小，它决定了数据库系统在内存中缓存的数据量。一般pgsql默认128M，相对较小。

**`work_mem`**用来限制每个服务进程进⾏排序或hash时的内存分配。默认4MB。

- 当每个进程得到的work_mem不⾜以排序或hash使⽤时，排序会借助磁盘临时⽂件，使得排序和hash的性能严重下降。

- PostgreSQL在排序时有Top-N heapsort、Quick sort、External merge这⼏种排序⽅法，**如果在查询计划中发现使⽤了External merge**，说明需要适当增加work_mem的值

**`random_page_cost`**代表随机访问磁盘块的代价估计。默认值为4。

- 若使用机械硬盘，则4无影响；若使用固态磁盘，建议设置为1.5，比**`seq_page_cost`**稍大



## 硬件性能

影响数据库性能的主要硬件因素有磁盘IO、CPU、内存和网络

- 磁盘IO：读写磁盘；
- CPU：内存管理、进程管理、生成执行计划、聚合、join操作、连接管理等；
- 内存：读写内存

最先到达瓶颈的，通常是磁盘的I/O，在投入生产之前应该对磁盘的容量和吞吐量进⾏估算，较大的内存可以明显降低服务器的I/O压力。