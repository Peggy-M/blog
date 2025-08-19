# MySQL 执行计划测试数据SQL

以下是用于测试MySQL执行计划的SQL代码，包含多个表结构和关联查询：

```sql
-- 创建测试数据库
CREATE DATABASE IF NOT EXISTS execution_plan_test;
USE execution_plan_test;

-- 创建用户表
CREATE TABLE users (
    user_id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    registration_date DATE NOT NULL,
    country_code CHAR(2) NOT NULL,
    last_login DATETIME,
    INDEX idx_registration_date (registration_date),
    INDEX idx_country (country_code)
);

-- 创建订单表
CREATE TABLE orders (
    order_id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    order_date DATETIME NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL,
    status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled') NOT NULL,
    payment_method VARCHAR(20) NOT NULL,
    INDEX idx_user_id (user_id),
    INDEX idx_order_date (order_date),
    INDEX idx_status (status),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- 创建订单详情表
CREATE TABLE order_items (
    item_id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    INDEX idx_order_id (order_id),
    INDEX idx_product_id (product_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

-- 创建产品表
CREATE TABLE products (
    product_id INT PRIMARY KEY AUTO_INCREMENT,
    product_name VARCHAR(100) NOT NULL,
    category_id INT NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    stock_quantity INT NOT NULL,
    supplier_id INT NOT NULL,
    INDEX idx_category_id (category_id),
    INDEX idx_price (price)
);

-- 创建产品类别表
CREATE TABLE categories (
    category_id INT PRIMARY KEY AUTO_INCREMENT,
    category_name VARCHAR(50) NOT NULL,
    parent_category_id INT NULL,
    INDEX idx_parent_category (parent_category_id)
);

-- 创建国家表
CREATE TABLE countries (
    country_code CHAR(2) PRIMARY KEY,
    country_name VARCHAR(50) NOT NULL,
    region VARCHAR(50) NOT NULL
);

-- 插入国家数据
INSERT INTO countries (country_code, country_name, region) VALUES
('US', 'United States', 'North America'),
('UK', 'United Kingdom', 'Europe'),
('DE', 'Germany', 'Europe'),
('FR', 'France', 'Europe'),
('JP', 'Japan', 'Asia'),
('CN', 'China', 'Asia'),
('IN', 'India', 'Asia'),
('BR', 'Brazil', 'South America'),
('AU', 'Australia', 'Oceania'),
('CA', 'Canada', 'North America');

-- 插入用户数据
INSERT INTO users (username, email, registration_date, country_code, last_login) VALUES
('john_doe', 'john@example.com', '2022-01-15', 'US', '2023-05-10 09:30:25'),
('jane_smith', 'jane@example.com', '2022-02-20', 'UK', '2023-05-12 14:22:18'),
('robert_johnson', 'robert@example.com', '2022-03-05', 'DE', '2023-05-11 11:05:42'),
('sarah_williams', 'sarah@example.com', '2022-01-28', 'FR', '2023-05-09 16:45:30'),
('michael_brown', 'michael@example.com', '2022-04-10', 'JP', '2023-05-08 10:15:55'),
('emily_davis', 'emily@example.com', '2022-02-15', 'CN', '2023-05-13 08:20:10'),
('david_miller', 'david@example.com', '2022-03-25', 'IN', '2023-05-07 13:40:35'),
('lisa_wilson', 'lisa@example.com', '2022-04-05', 'BR', '2023-05-14 17:25:50'),
('james_taylor', 'james@example.com', '2022-01-10', 'AU', '2023-05-06 12:15:45'),
('susan_anderson', 'susan@example.com', '2022-02-28', 'CA', '2023-05-15 15:35:20'),
('william_thomas', 'william@example.com', '2022-03-15', 'US', '2023-05-05 09:50:30'),
('karen_jackson', 'karen@example.com', '2022-04-20', 'UK', '2023-05-16 14:10:15'),
('richard_white', 'richard@example.com', '2022-01-05', 'DE', '2023-05-04 11:30:40'),
('patricia_harris', 'patricia@example.com', '2022-02-10', 'FR', '2023-05-17 16:45:25'),
('charles_martin', 'charles@example.com', '2022-03-20', 'JP', '2023-05-03 13:05:50'),
('jennifer_thompson', 'jennifer@example.com', '2022-04-15', 'CN', '2023-05-18 08:20:35'),
('thomas_garcia', 'thomas@example.com', '2022-01-20', 'IN', '2023-05-02 10:40:10'),
('mary_martinez', 'mary@example.com', '2022-02-05', 'BR', '2023-05-19 15:55:45'),
('christopher_robinson', 'christopher@example.com', '2022-03-10', 'AU', '2023-05-01 12:15:20'),
('elizabeth_clark', 'elizabeth@example.com', '2022-04-25', 'CA', '2023-05-20 17:30:55');

-- 插入类别数据
INSERT INTO categories (category_name, parent_category_id) VALUES
('Electronics', NULL),
('Clothing', NULL),
('Books', NULL),
('Smartphones', 1),
('Laptops', 1),
('Tablets', 1),
('Men''s Clothing', 2),
('Women''s Clothing', 2),
('Fiction', 3),
('Non-Fiction', 3);

-- 插入产品数据
INSERT INTO products (product_name, category_id, price, stock_quantity, supplier_id) VALUES
('iPhone 14', 4, 999.99, 50, 1),
('Samsung Galaxy S23', 4, 899.99, 45, 2),
('MacBook Pro 16"', 5, 2399.99, 30, 1),
('Dell XPS 15', 5, 1899.99, 40, 3),
('iPad Pro', 6, 1099.99, 35, 1),
('Samsung Galaxy Tab', 6, 749.99, 25, 2),
('Men''s T-Shirt', 7, 29.99, 100, 4),
('Women''s Dress', 8, 59.99, 80, 5),
('Science Fiction Novel', 9, 14.99, 200, 6),
('Biography', 10, 19.99, 150, 7),
('Google Pixel 7', 4, 799.99, 40, 8),
('Lenovo ThinkPad', 5, 1499.99, 35, 9),
('Men''s Jeans', 7, 49.99, 120, 4),
('Women''s Blouse', 8, 39.99, 90, 5),
('Mystery Novel', 9, 12.99, 180, 6),
('History Book', 10, 24.99, 130, 7);

-- 插入订单数据
INSERT INTO orders (user_id, order_date, total_amount, status, payment_method) VALUES
(1, '2023-01-15 10:30:00', 999.99, 'delivered', 'credit_card'),
(2, '2023-01-20 14:45:00', 1899.99, 'processing', 'paypal'),
(3, '2023-02-05 09:15:00', 29.99, 'shipped', 'credit_card'),
(4, '2023-02-10 16:20:00', 59.99, 'delivered', 'debit_card'),
(5, '2023-02-15 11:30:00', 14.99, 'delivered', 'paypal'),
(6, '2023-03-01 13:45:00', 1099.99, 'processing', 'credit_card'),
(7, '2023-03-10 15:20:00', 49.99, 'shipped', 'debit_card'),
(8, '2023-03-15 10:10:00', 39.99, 'delivered', 'credit_card'),
(9, '2023-04-01 12:30:00', 24.99, 'delivered', 'paypal'),
(10, '2023-04-05 14:45:00', 799.99, 'processing', 'credit_card'),
(1, '2023-04-10 09:20:00', 1499.99, 'shipped', 'debit_card'),
(2, '2023-04-15 16:30:00', 12.99, 'delivered', 'credit_card'),
(3, '2023-05-01 11:45:00', 899.99, 'processing', 'paypal'),
(4, '2023-05-05 13:20:00', 749.99, 'shipped', 'credit_card'),
(5, '2023-05-10 15:30:00', 19.99, 'delivered', 'debit_card');

-- 插入订单详情数据
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
(1, 1, 1, 999.99),
(2, 4, 1, 1899.99),
(3, 7, 1, 29.99),
(4, 8, 1, 59.99),
(5, 9, 1, 14.99),
(6, 5, 1, 1099.99),
(7, 13, 1, 49.99),
(8, 14, 1, 39.99),
(9, 16, 1, 24.99),
(10, 11, 1, 799.99),
(11, 12, 1, 1499.99),
(12, 15, 1, 12.99),
(13, 2, 1, 899.99),
(14, 6, 1, 749.99),
(15, 10, 1, 19.99);

-- 示例查询1：基本连接查询
EXPLAIN 
SELECT u.username, u.email, o.order_date, o.total_amount, o.status
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE u.country_code = 'US'
ORDER BY o.order_date DESC;

-- 示例查询2：多表连接查询
EXPLAIN
SELECT u.username, u.country_code, o.order_date, oi.quantity, p.product_name, c.category_name
FROM users u
JOIN orders o ON u.user_id = o.user_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN categories c ON p.category_id = c.category_id
WHERE o.order_date >= '2023-03-01'
AND o.status = 'delivered'
ORDER BY o.order_date DESC;

-- 示例查询3：带聚合函数的查询
EXPLAIN
SELECT u.country_code, c.country_name, COUNT(o.order_id) as order_count, 
       SUM(o.total_amount) as total_revenue, AVG(o.total_amount) as avg_order_value
FROM users u
JOIN orders o ON u.user_id = o.user_id
JOIN countries c ON u.country_code = c.country_code
WHERE o.order_date BETWEEN '2023-01-01' AND '2023-06-30'
GROUP BY u.country_code, c.country_name
HAVING total_revenue > 1000
ORDER BY total_revenue DESC;

-- 示例查询4：子查询
EXPLAIN
SELECT u.username, u.email, u.registration_date,
       (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.user_id) as order_count,
       (SELECT MAX(order_date) FROM orders o WHERE o.user_id = u.user_id) as last_order_date
FROM users u
WHERE u.registration_date >= '2022-02-01'
ORDER BY order_count DESC;

-- 示例查询5：复杂条件查询
EXPLAIN
SELECT p.product_name, c.category_name, p.price, p.stock_quantity,
       COUNT(oi.order_id) as times_ordered,
       SUM(oi.quantity) as total_quantity_sold
FROM products p
JOIN categories c ON p.category_id = c.category_id
LEFT JOIN order_items oi ON p.product_id = oi.product_id
WHERE p.price > 500
AND c.parent_category_id = 1  -- Electronics category
GROUP BY p.product_id, p.product_name, c.category_name, p.price, p.stock_quantity
HAVING times_ordered > 0
ORDER BY total_quantity_sold DESC;

-- 查看表结构和索引信息
SHOW INDEX FROM users;
SHOW INDEX FROM orders;
SHOW INDEX FROM order_items;
SHOW INDEX FROM products;
SHOW INDEX FROM categories;
```

## 使用说明

1. 执行以上SQL代码创建测试数据库和表结构
2. 使用`EXPLAIN`关键字分析查询性能
3. 尝试添加/删除索引来观察执行计划的变化
4. 测试不同类型的查询（简单查询、连接查询、子查询、聚合查询等）

## 执行计划分析要点

- 查看`type`列：ALL（全表扫描）、index（索引扫描）、range（范围扫描）、ref（索引查找）、eq_ref（唯一索引查找）、const（常量查询）
- 查看`key`列：实际使用的索引
- 查看`rows`列：预估需要扫描的行数
- 查看`Extra`列：额外信息（Using where、Using index、Using temporary、Using filesort等）

您可以通过修改查询条件、添加索引或调整表结构来观察执行计划的变化，从而优化查询性能。



好的，这些是 MySQL `EXPLAIN` 输出中 `type` 列的不同值，它们代表了查询时查找数据的方式，**性能从最差 (ALL) 到最优 (const) 依次提高**。

下面我为你详细解释这些类型的区别，特别是你列出的 `ref` 和 `eq_ref`。

## 性能等级排序（从最佳到最差）

`const` > `eq_ref` > `ref` > `range` > `index` > `ALL`

---

### 详细解释

#### 1. `eq_ref` (唯一索引扫描)

*   **含义**：这是除了 `const` 之外最好的连接类型。它通常出现在使用 **`PRIMARY KEY` 或 `UNIQUE` 索引** 进行表连接时。
*   **工作原理**：对于来自前一个表的每一行组合，从当前表中**读取唯一的一行**。MySQL 知道最多只返回一条符合条件的记录。
*   **示例**：
    ```sql
    SELECT * 
    FROM users 
    INPLICIT JOIN orders ON users.id = orders.user_id; 
    -- 假设 orders.user_id 是 PRIMARY KEY 或 UNIQUE KEY
    ```
*   **特点**：**性能极佳**，因为数据库通过唯一键直接定位到单条记录。

#### 2. `ref` (非唯一索引扫描)

*   **含义**：使用**非唯一索引（普通索引）** 或者**唯一索引的非唯一性前缀**进行查找。
*   **工作原理**：对于来自前一个表的每一行组合，从当前表中**读取所有匹配索引值的行**（可能有多行）。
*   **示例**：
    ```sql
    SELECT * 
    FROM users 
    WHERE last_name = 'Smith'; 
    -- 假设 last_name 字段上有一个普通索引 INDEX(last_name)
    
    SELECT * 
    FROM orders 
    IMPLICIT JOIN users ON users.email = orders.customer_email;
    -- 假设 users.email 上有普通索引，但 email 可能不唯一
    ```
*   **特点**：**性能很好**，但不如 `eq_ref`，因为它可能需要处理多个匹配的行。

#### 3. `range` (索引范围扫描)

*   **含义**：使用索引检索**给定范围**的行。
*   **工作原理**：扫描索引的一部分，而不是全部。关键字段是 `key` 列显示了使用的索引，`rows` 列显示了预估要检查的行数。
*   **示例**：
    ```sql
    SELECT * 
    FROM users 
    WHERE age BETWEEN 20 AND 30; 
    -- 假设 age 字段有索引
    
    SELECT * 
    FROM orders 
    WHERE order_date > '2023-01-01'; 
    -- 假设 order_date 字段有索引
    ```
*   **特点**：性能优于全索引扫描 (`index`)，因为它只扫描索引的一个范围。

#### 4. `index` (全索引扫描)

*   **含义**：扫描整个索引树。
*   **工作原理**：与 `ALL` 类似，但优点是**只扫描索引文件**，通常比扫描数据文件（`ALL`）要小，所以更快。
*   **常见场景**：
    1.  查询的字段全部包含在某个索引中（覆盖索引）。
    2.  使用索引来执行 `ORDER BY` 操作。
*   **示例**：
    ```sql
    SELECT id FROM users; 
    -- 如果 id 是主键，这就是一个覆盖索引扫描
    
    SELECT last_name FROM users ORDER BY last_name; 
    -- 如果 last_name 有索引，可以直接按索引顺序读取
    ```
*   **特点**：避免了全表扫描，但如果索引很大，速度仍然较慢。

#### 5. `ALL` (全表扫描)

*   **含义**：**性能最差**的连接类型。MySQL 必须扫描整张表来找到匹配的行。
*   **工作原理**：从表的第一行开始，逐行检查，直到最后一行。
*   **何时发生**：通常是因为**没有索引可用**，或者查询优化器认为使用索引的成本比全表扫描更高（例如对小表进行查询）。
*   **示例**：
    ```sql
    SELECT * 
    FROM users 
    WHERE phone_number = '123-456-7890'; 
    -- 如果 phone_number 字段上没有索引
    ```
*   **特点**：**必须极力避免**的类型，尤其是对于大表。解决方案是为查询条件添加合适的索引。

#### 6. `const` (常量查询)

*   **含义**：**性能最佳**的访问类型。表示通过索引一次就能找到唯一一行。
*   **工作原理**：MySQL 优化器将查询的某部分优化为一个常量。它只会被读取一次。
*   **示例**：
    ```sql
    SELECT * 
    FROM users 
    WHERE id = 1; 
    -- 通过主键（PRIMARY KEY）查询
    
    SELECT * 
    FROM countries 
    WHERE country_code = 'US'; 
    -- 通过唯一键（UNIQUE KEY）查询
    ```
*   **特点**：速度极快，是理想的访问类型。

---

### 总结对比表

| 类型 (type)  | 含义                             | 使用场景                         | 性能     | 备注                                 |
| :----------- | :------------------------------- | :------------------------------- | :------- | :----------------------------------- |
| **`const`**  | 通过主键或唯一键找到**一条**记录 | `WHERE primary_key = 1`          | **最佳** | 最多返回一条记录                     |
| **`eq_ref`** | 连接时使用主键或唯一键           | `A JOIN B ON A.id = B.unique_id` | **极佳** | 对于前表的每一行，后表只有一条匹配   |
| **`ref`**    | 使用普通（非唯一）索引查找       | `WHERE indexed_column = value`   | **很好** | 对于前表的每一行，后表可能有多条匹配 |
| **`range`**  | 使用索引检索一个范围             | `BETWEEN`, `>`, `<`, `IN()`      | **好**   | 只扫描索引的一部分                   |
| **`index`**  | 全索引扫描                       | 覆盖索引，按索引排序             | **一般** | 扫描整个索引，但比 `ALL` 好          |
| **`ALL`**    | 全表扫描                         | 无可用索引或小表查询             | **最差** | **必须优化**                         |

### 如何优化？

1.  **目标是让 `type` 尽可能接近左边（`const`, `eq_ref`, `ref`）。**
2.  如果看到 `ALL`，检查 `WHERE` 子句和 `JOIN` 条件中的字段，并为它们创建索引。
3.  如果看到 `index`，考虑是否可以优化为使用 `WHERE` 条件的 `range` 扫描，或者是否使用了覆盖索引。
4.  使用 `EXPLAIN` 是优化查询性能最强大的工具之一。



您观察得非常准确！`const` 和 `eq_ref` 确实看起来很相似，但它们有**本质的区别**。

## const 与 eq_ref 的核心区别

**`const` 用于单表查询，`eq_ref` 用于多表连接查询。**

---

## 详细对比

### `const` (常量查询)

*   **场景**：**单表查询**
*   **条件**：使用 **主键 (PRIMARY KEY)** 或 **唯一索引 (UNIQUE INDEX)** 的等值查询
*   **工作原理**：MySQL 知道查询条件精确匹配**唯一一条**记录，直接将查询优化为常量
*   **示例**：
    
    ```sql
    SELECT * FROM users WHERE user_id = 5;
    -- user_id 是主键
    
    SELECT * FROM products WHERE sku = 'ABC123';
    -- sku 字段有唯一索引
    ```
*   **特点**：**最快的访问方式**，因为只需要一次索引查找

### `eq_ref` (等值引用)

*   **场景**：**多表连接查询** (JOIN)
*   **条件**：连接条件使用被驱动表的 **主键** 或 **唯一索引**
*   **工作原理**：对于驱动表的**每一行**，在被驱动表中通过主键/唯一索引查找**恰好一行**匹配记录
*   **示例**：
    ```sql
    SELECT * 
    FROM orders 
    JOIN users ON orders.user_id = users.user_id;
    -- users.user_id 是主键
    
    SELECT *
    FROM order_items 
    JOIN products ON order_items.product_id = products.product_id;
    -- products.product_id 是主键
    ```
*   **特点**：**最好的连接类型**，对驱动表的每行记录只做一次索引查找

---

## 本质区别总结

| 特性         | `const`      | `eq_ref`                    |
| ------------ | ------------ | --------------------------- |
| **查询类型** | 单表查询     | 多表连接                    |
| **查找次数** | 1次查找      | N次查找 (N=驱动表行数)      |
| **优化阶段** | 查询优化阶段 | 查询执行阶段                |
| **性能影响** | 绝对最优     | 相对最优 (取决于驱动表行数) |

## 执行计划示例

```sql
-- 单表查询 - 会出现 const
EXPLAIN SELECT * FROM users WHERE user_id = 1;

-- 多表连接 - 会出现 eq_ref
EXPLAIN 
SELECT * 
FROM orders 
JOIN users ON orders.user_id = users.user_id 
WHERE orders.order_id = 100;
```

在这个连接查询中：
- 对 `orders` 表的查询可能是 `const` (如果 `order_id` 是主键)
- 对 `users` 表的连接会是 `eq_ref` (因为通过主键 `user_id` 连接)

## 为什么这个区别很重要？

理解这个区别有助于：
1. **性能优化**：知道 `const` 是绝对最优，而 `eq_ref` 的性能**取决于驱动表**(所以使用小表驱动大表，保证获取的小表相互之间交集数据更少)的大小
2. **查询设计**：在设计连接时，确保连接条件使用主键或唯一索引
3. **问题诊断**：当连接性能不佳时，可以检查驱动表是否过大

简单来说：**`const` 是单表查询的极致优化，`eq_ref` 是多表连接的最佳方式。**