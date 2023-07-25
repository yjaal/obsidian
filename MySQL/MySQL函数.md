```toc

```


## 字符串函数

### 字符串截取-substring_index

```sql
-- substring_index('待截取字符串', '分隔符', '要截取位置-整数')
-- SUBSTRING_INDEX(s, delimiter, number)
-- 返回从字符串 s 的第 number 个出现的分隔符 delimiter 之后的子串。
-- 如果 number 是正数，返回第 number 个字符左边的字符串。
-- 如果 number 是负数，返回第(number 的绝对值(从右边数))个字符右边的字符串。
SELECT SUBSTRING_INDEX('a*b','*',1) from dual;-- a
SELECT SUBSTRING_INDEX('a*b','*',-1)  from dual;    -- b
SELECT SUBSTRING_INDEX(SUBSTRING_INDEX('a*b*c*d*e','*',3),'*',-1) from dual;  -- c
```

### 字符串截取-substring

```sql
-- substring(s, start, len)
-- sebstr(s, start, len)
-- 从字符串 s 的 start 位置截取长度为 len 的子字符串，从1开始数
SELECT SUBSTRING("RUNOOB", 2, 3) from dual; -- UNO
```


### 字符串长度

```sql
select char_length('aa') from dual; -- 2
select length('aa') from dual; -- 2
```

### 字符串合并

```sql
-- concat(s1, s2, ..., sn)
-- concat_wx('分隔符', s1, s2, ..., sn)
select concat('a', 'b', 'c') from dual; -- abc
select concat('#', 'a', 'b', 'c') from dual; -- a#b#c
```

### 字符串位置

```sql
-- field (s1, s2, ..., sn) 返回 s1 在整个字符串数组中的位置
-- find_in_set (s1, s2) 返回字符串 s2 中与 s1 匹配的字符串的位置
select field ('a', 'b', 'c') from dual; -- 0
select find_in_set ('a', 'abc') from dual; -- 0
```

### 字符串替换

```sql
-- insert (s1, x, len, s2) 字符串 s1 中从 x 位置开始长度为 len 的字符串替换为 s2
-- replace (s, s1, s2) 用 s2 替换 s 中 s1 的字符串
select insert ('google. com', 1, 6, 'runoob') from dual; -- runoob. com
select replace ('abc', 'a', 'x') from dual; -- xbc
```

## 数字函数

### 平均值

```sql
-- avg (expression)
select avg (price) from x_table;
```

### 返回>=x 的最小整数

```sql
-- ceil (x)
select ceil (1.5) from dual; -- 2
```

### 返回<=x 的最大整数

```sql
-- floor (x)
select floor (1.5) from dual; -- 1
```


### 最大最小值

```sql
-- greatest (expr1, expr2, ...) 返回最大值
-- least (expr1, expr2, ...) 返回最小值
-- max (expression) 返回最大值
-- min (expression) 返回最小值
```


## 日期函数

### 天数增加或减少

```sql
-- adddate (date, day)
select adddate ('2022-03-12', interval 1 day) from dual; -- 2022-03-13
select adddate ('2022-03-12', interval -1 day) from dual; -- 2022-03-11
```


### 当前日期，时间

```sql
select curdate () from dual; -- 当前日期
select current_time () from dual; -- 当前时间
select current_timestamp () from dual; -- 当前日期时间
```


### 日期格式化

```sql
-- 年 (%Y-4 位, %y-2 位)，月 (%m: 00-12)，日 (%d)，时 (%H: 00-23, %h: 00-12)，分 (%i)，秒 (%s)
-- date_format (d, f)
select date_format ('2022-11-11 10:10:10 ', '%Y%m%d %r') from dual;-- 20221111 10:10:10 AM
select date_format ('2022-11-11 10:10:10 ', '%Y%m%d') from dual;-- 20221111
```

### 获取日期中的指定的值，如年，月，日

```sql
-- extract (type from date)
SELECT EXTRACT (MINUTE FROM '2011-11-11 11:11:11 ')  -- 11
```

### 日期相减 (d1-d2)

```sql
select datediff (d1, d2) from dual;
```

### 时间相减

```sql
select timediff ('2019-06-03 12:30:00 ', '2019-06-03 12:29:30 ')    -- 00:00:30
等价于
select timediff (' 12:30:00 ', ' 12:29:30 ')    -- 00:00:30
```

### Unix 时间转换

```sql
select JOB_NAME, from_unixtime(PREV_FIRE_TIME/1000, '%Y%m%d %H:%m:%s'), from_unixtime(NEXT_FIRE_TIME/1000, '%Y%m%d %H:%m:%s') from QRTZ_TRIGGERS where TRIGGER_NAME = 'FixFunctionStatisticsDetailDataTaskTrigger';

select UNIX_TIMESTAMP(now()) from dual;
select from_unixtime(1689757300000/1000, '%Y%m%d %H:%m:%s') from dual
```

## 高级表达式

### @i:=@i+1

```sql
-- 这是一种伪计数
-- Oracle 中有一个伪列 rownum，可以在生成查询结果表的时候生成一组递增的序列号。MySQL 中没有这个伪列，但是有时候要用，可以用如下方法模拟生成一列自增序号。
-- 这里初始值为 0，自增间隔为 1
select @i:=@i+1 from x_table, (select @i:=0) as t limit 10;
```

### 判断

```sql
-- if (expr, v1, v2) 如果表达式 expr 成立，返回结果 v1；否则，返回结果 v2。
SELECT IF (1 > 0,'正确','错误') from dual;   -- 正确
```

### 空判断

```sql
-- ifnull (v1, v2)
-- 如果 v1 的值不为 NULL，则返回 v1，否则返回 v2。
SELECT IFNULL (null,'Hello Word') from dual; -- Hello Word
```

### 从 json 中解析出具体字段

```sql
json_extract 从 json 中解析出具体字段，json_unquote 去除引号
select json_unquote (json_extract ('json_info', '$. name')) as name from t_table;
```
