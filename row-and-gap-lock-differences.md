# 行锁和间隙锁的区别

## 概述

在MySQL InnoDB存储引擎中，行锁（Row Lock）和间隙锁（Gap Lock）是两种重要的锁机制，用于实现事务的隔离性和防止幻读问题。

## 行锁（Row Lock）

### 定义
行锁是对数据库表中**已存在的某一行记录**加锁，防止其他事务对该行进行修改或删除。

### 特点
- **锁定对象**：实际存在的数据行
- **作用范围**：仅锁定具体的一行或多行记录
- **锁定时机**：当事务对某行记录进行UPDATE、DELETE或SELECT FOR UPDATE操作时
- **并发性**：粒度小，并发性能好

### 类型
1. **共享锁（S Lock）**：读锁，多个事务可以同时持有
2. **排他锁（X Lock）**：写锁，独占访问

### 示例
```sql
-- 事务A对id=10的行加排他锁
BEGIN;
SELECT * FROM users WHERE id = 10 FOR UPDATE;
-- 此时其他事务无法修改id=10这一行
UPDATE users SET name = 'New Name' WHERE id = 10;
COMMIT;
```

## 间隙锁（Gap Lock）

### 定义
间隙锁是对索引记录之间的"间隙"加锁，锁定的是一个范围，但**不包括记录本身**。它锁定的是两个索引记录之间的区域，或者第一个索引记录之前的区域，或者最后一个索引记录之后的区域。

### 特点
- **锁定对象**：索引记录之间的间隙（不存在的范围）
- **作用范围**：防止其他事务在这个间隙中插入新记录
- **锁定时机**：在RR（可重复读）隔离级别下，进行范围查询时
- **目的**：防止幻读（Phantom Read）

### 工作原理
假设表中有记录id = 1, 5, 10, 20：
- 间隙1：(-∞, 1)
- 间隙2：(1, 5)
- 间隙3：(5, 10)
- 间隙4：(10, 20)
- 间隙5：(20, +∞)

### 示例
```sql
-- 假设users表中有id = 5, 10, 15
BEGIN;
SELECT * FROM users WHERE id BETWEEN 8 AND 12 FOR UPDATE;
-- 此查询会对(5, 10), (10, 15)之间的间隙加锁
-- 其他事务无法插入id=8, 9, 11, 12的记录
COMMIT;
```

## 临键锁（Next-Key Lock）

### 定义
临键锁是**行锁 + 间隙锁**的组合，锁定一个范围，**并且锁定记录本身**。

### 示例
```sql
-- 使用上面的例子
SELECT * FROM users WHERE id BETWEEN 8 AND 12 FOR UPDATE;
-- 实际上会加临键锁：
-- - 间隙锁：(5, 10)
-- - 行锁：id=10
-- - 间隙锁：(10, 15)
-- - 行锁：id=15（如果扫描到）
```

## 主要区别对比

| 对比维度 | 行锁（Row Lock） | 间隙锁（Gap Lock） |
|---------|-----------------|-------------------|
| **锁定对象** | 已存在的具体数据行 | 索引记录之间的间隙 |
| **锁定范围** | 具体的一行或多行 | 一个不包含记录的范围 |
| **主要作用** | 防止其他事务修改/删除已有数据 | 防止其他事务插入新数据 |
| **触发条件** | 精确匹配查询或更新 | 范围查询 |
| **解决问题** | 并发修改冲突 | 幻读问题 |
| **隔离级别** | RC和RR级别都有 | 主要在RR级别 |
| **对INSERT的影响** | 不影响（如果插入的是新行） | 阻止在间隙内插入 |

## 实际场景示例

### 场景1：行锁示例

```sql
-- 初始数据
-- id: 1, 5, 10, 15, 20

-- 事务A
BEGIN;
UPDATE users SET name = 'A' WHERE id = 10;
-- 只锁定id=10这一行

-- 事务B（可以成功）
INSERT INTO users (id, name) VALUES (8, 'B');  -- ✓ 可以插入

-- 事务C（会等待）
UPDATE users SET name = 'C' WHERE id = 10;  -- ✗ 阻塞，等待行锁释放
```

### 场景2：间隙锁示例

```sql
-- 初始数据
-- id: 1, 5, 10, 15, 20

-- 事务A（RR隔离级别）
BEGIN;
SELECT * FROM users WHERE id > 5 AND id < 15 FOR UPDATE;
-- 锁定间隙(5, 10), (10, 15)
-- 锁定记录id=10

-- 事务B（会等待）
INSERT INTO users (id, name) VALUES (8, 'B');   -- ✗ 阻塞，在间隙(5,10)内
INSERT INTO users (id, name) VALUES (12, 'C');  -- ✗ 阻塞，在间隙(10,15)内

-- 事务C（可以成功）
INSERT INTO users (id, name) VALUES (3, 'D');   -- ✓ 不在锁定间隙内
INSERT INTO users (id, name) VALUES (16, 'E');  -- ✓ 不在锁定间隙内

-- 事务D（会等待）
UPDATE users SET name = 'F' WHERE id = 10;  -- ✗ 阻塞，行被锁定
```

### 场景3：避免幻读

```sql
-- 事务A（使用间隙锁防止幻读）
BEGIN;
-- 第一次查询
SELECT * FROM users WHERE age BETWEEN 20 AND 30;  -- 返回2条记录
-- 此时加了间隙锁

-- 事务B尝试插入
INSERT INTO users (id, age) VALUES (100, 25);  -- 阻塞！

-- 事务A再次查询
SELECT * FROM users WHERE age BETWEEN 20 AND 30;  -- 仍然是2条记录
-- 没有幻读！
COMMIT;
```

## 如何查看锁信息

```sql
-- 查看当前锁信息（MySQL 5.7+）
SELECT * FROM performance_schema.data_locks;

-- 查看锁等待信息
SELECT * FROM performance_schema.data_lock_waits;

-- 查看InnoDB状态（包含锁信息）
SHOW ENGINE INNODB STATUS;
```

## 性能影响和最佳实践

### 行锁
- **优点**：锁粒度小，并发性能好
- **缺点**：锁数量多时，管理开销增加
- **最佳实践**：
  - 尽量使用索引列作为查询条件
  - 缩短事务时间
  - 避免长时间持有锁

### 间隙锁
- **优点**：有效防止幻读
- **缺点**：可能降低并发性能，可能导致死锁
- **最佳实践**：
  - 如果不需要防止幻读，考虑使用RC隔离级别（不使用间隙锁）
  - 避免大范围的范围查询
  - 注意间隙锁导致的死锁问题

## 死锁示例

间隙锁容易导致死锁：

```sql
-- 初始数据：id = 5, 10, 15

-- 事务A
BEGIN;
SELECT * FROM users WHERE id = 7 FOR UPDATE;  -- 锁定间隙(5, 10)

-- 事务B
BEGIN;
SELECT * FROM users WHERE id = 8 FOR UPDATE;  -- 也想锁定间隙(5, 10)，等待

-- 事务A
INSERT INTO users (id, name) VALUES (8, 'A');  -- 需要插入意向锁，等待事务B
-- 死锁！
```

## 总结

1. **行锁**锁定**已存在的记录**，防止并发修改
2. **间隙锁**锁定**记录之间的间隙**，防止幻读
3. **临键锁**是两者的结合，提供完整的可重复读保证
4. 在**可重复读（RR）**隔离级别下，InnoDB默认使用临键锁
5. 在**读已提交（RC）**隔离级别下，不使用间隙锁，只使用行锁
6. 正确理解和使用这些锁机制，对于数据库性能优化和避免死锁至关重要

## 参考资料

- [MySQL官方文档 - InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- [MySQL事务隔离级别](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
