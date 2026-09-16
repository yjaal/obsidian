
```toc
```

## 为什么慢

### 使用 explain 分析执行计划

执行 `EXPLAIN` 查看每条关联的访问类型。重点关注：

- `type` 列：出现 `ALL`（全表扫描）、`index`（全索引扫描）需要优化；理想是 `ref`、`eq_ref` 或 `const`。
- `rows` 列：估算扫描行数，明显偏大的表考虑加索引。
- `Extra` 列：出现 `Using temporary`（使用临时表）、`Using filesort`（文件排序）是性能大敌。

  
### 查看真实 SQL 执行时间分布

```sql
SET profiling = 1; 
-- 执行你的慢SQL 
SELECT ...; 
SHOW PROFILES; 
SHOW PROFILE FOR QUERY 1;
```

结果会显示每个阶段（sending data、creating sort index 等）的耗时，帮你判断是 IO 瓶颈还是 CPU/排序瓶颈。

### 检查数据库配置

- `join_buffer_size`：太小会导致多次扫描。
- `tmp_table_size` / `max_heap_table_size`：太大会导致磁盘临时表。
- `innodb_buffer_pool_size`：是否足够容纳热数据。


## 解决方案

下面从**低成本、易改动的 SQL 层**到**高成本、长效的架构层**，逐级给出具体解决方案

### 索引优化

确保每个 `ON` 和 `WHERE` 条件中的列都有索引。对于 `LEFT JOIN`，右表关联列必须索引。联合索引要遵循**最左前缀**原则。



### 调整 JOIN 顺序

让小表驱动大表。


### 拆分 JOIN + 应用层组装

当 10 张表关联只是为了展示一个列表，且数据量不是天文数字时，可以在 Java 代码中分批查询，再用 Stream 合并。

### 临时表或衍生表

将多次使用的 JOIN 中间结果物化，减少重复计算。


### 使用大数据计算

后台使用 hive 计算统计，或者使用 click house 或者 Doris














