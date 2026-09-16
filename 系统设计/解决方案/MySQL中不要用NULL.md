
```toc
```


## NULL 含义

NULL 不是空字符串，也不是 0。在 MySQL 中，NULL 表示 **未知的值**， 它是一种状态，而不是一个具体的值。它与空字符串 `''`、数字 `0` 完全不同。

**最大的区别在于：任何值与 NULL 进行运算的结果都是 NULL，比较结果既不是 TRUE 也不是 FALSE，而是 NULL。**

```sql
SELECT NULL = 0; -- 结果 NULL 
SELECT NULL = ''; -- 结果 NULL 
SELECT NULL != NULL; -- 结果 NULL
```

## 三值逻辑

在普通布尔逻辑中，只有 `TRUE` 和 `FALSE` 两种结果。而 MySQL 引入 NULL 后，出现了第三种结果：`UNKNOWN`。

当查询条件中出现 NULL 时，`WHERE` 子句只返回条件为 `TRUE` 的行，而 `FALSE` 和 `UNKNOWN` 都会被过滤掉。**这就是为什么 `NOT IN` 子查询中一旦有 NULL，结果集就会变为空的原因！**

**示例：`NOT IN` 陷阱**

```sql
-- 表结构 
CREATE TABLE users (id INT, name VARCHAR(20)); 
INSERT INTO users VALUES (1, 'Alice'), (2, 'Bob'), (3, NULL); -- 注意有个NULL 

-- 查询ID不在另一个子查询中的用户 
SELECT * FROM users WHERE id NOT IN (SELECT id FROM users WHERE name = 'Alice'); 
-- 预期结果：只有Bob，但实际返回空！
```

原因分析：子查询 `(SELECT id FROM users WHERE name = 'Alice')` 返回 `1`，没问题。但 `NOT IN (1)` 会与表中每一行比较，当遇到 `NULL` 时，`NOT IN` 的逻辑变成 `id NOT IN (1)`。

由于子查询中一旦包含 NULL，整个 `NOT IN` 条件会变成 `UNKNOWN`（三值逻辑），最终导致所有行被过滤。

**解决方案**：子查询中使用 `WHERE column IS NOT NULL` 排除 NULL，或改用 `NOT EXISTS`。

```sql
SELECT * FROM users u WHERE NOT EXISTS (SELECT 1 FROM users WHERE id = u.id AND name = 'Alice');
```


## NULL 如何让索引失效

### 索引不存储 NULL 值

在 InnoDB 的 B+Tree 索引中，**NULL 值不会被存储在二级索引中**（唯一索引除外）。这意味着，使用 `IS NULL` 查询时，MySQL 无法利用二级索引快速定位，只能扫描全表。

**实际测试**：一张 100 万行的表，`status` 列有 90%是 NULL，10%非 NULL。
查询 `WHERE status = 'ACTIVE'` 可以用到索引（因为非 NULL 部分值很多）。但查询 `WHERE status IS NULL` 会走全表扫描，性能极差。


### 复合索引中的 NULL 问题

复合索引 `(a, b)` 中，如果 a 为 NULL，则这一行不会出现在索引中。因此，`WHERE a = 1 AND b IS NULL` 可能只能用到索引的前半部分，但无法通过索引过滤 b 的 NULL。

## 聚合函数的陷阱

```sql
-- 表数据: amount 列有 (100, 200, NULL, 400) 
SELECT AVG(amount) FROM orders; -- 结果 (100+200+400)/3 = 233.33，不是 (700/4)=175 
SELECT COUNT(amount) FROM orders; -- 结果 3，NULL被忽略 
SELECT SUM(amount) FROM orders; -- 结果 700，忽略NULL
```

`count(*)` 和 `count(1)` 会统计所有数据，但是 `count(列)` 则会忽略 NULl 值的列。








