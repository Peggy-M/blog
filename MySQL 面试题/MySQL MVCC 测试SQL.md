# MySQL MVCC 测试SQL

以下是测试MySQL MVCC的SQL代码，您可以直接在MySQL中执行这些语句来测试MVCC机制：

```sql
-- 创建测试数据库和表
CREATE DATABASE IF NOT EXISTS mvcc_test;
USE mvcc_test;

-- 创建测试表
CREATE TABLE IF NOT EXISTS products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    quantity INT NOT NULL,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- 插入初始数据
TRUNCATE TABLE products;
INSERT INTO products (name, price, quantity) VALUES
('Product A', 19.99, 100),
('Product B', 29.99, 50),
('Product C', 9.99, 200);

-- 查看初始数据
SELECT * FROM products;

-- 开始第一个事务（事务1）
START TRANSACTION;

-- 在事务1中更新数据
UPDATE products SET price = 24.99 WHERE name = 'Product A';

-- 开始第二个事务（事务2）
-- 注意：需要在另一个MySQL连接/会话中执行
START TRANSACTION;

-- 在事务2中查询（应该看到旧价格，因为事务1尚未提交）
SELECT * FROM products WHERE name = 'Product A';

-- 回到事务1提交更改
COMMIT;

-- 在事务2中再次查询（仍然看到旧价格，因为MVCC）
SELECT * FROM products WHERE name = 'Product A';

-- 提交或回滚事务2
COMMIT;

-- 现在应该看到新价格
SELECT * FROM products WHERE name = 'Product A';

-- 测试读已提交隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;

-- 在另一个会话中更新数据
UPDATE products SET quantity = 75 WHERE name = 'Product B';

-- 回到当前会话查询（应该立即看到更改）
SELECT * FROM products WHERE name = 'Product B';

-- 清理
COMMIT;
DROP DATABASE IF EXISTS mvcc_test;
```

## 执行步骤说明

1. 首先执行创建数据库和表的语句
2. 打开两个MySQL客户端连接（会话）
3. 在第一个连接中执行事务1的操作
4. 在第二个连接中执行事务2的操作
5. 观察MVCC如何确保事务隔离性

## MVCC测试要点

- 在默认的REPEATABLE READ隔离级别下，每个事务看到的是数据库在事务开始时的快照
- 即使其他事务提交了更改，当前事务仍然看到一致的数据视图
- 使用READ COMMITTED隔离级别时，事务可以看到其他事务已提交的更改

您可以通过在不同MySQL会话中执行这些SQL语句来测试MVCC的行为。