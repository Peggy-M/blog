### MySQL 的死锁以及解决方案

死锁是指两个或更多的事务在执行过程中，因**争夺资源**而陷入的一种相互等待的状态，若无外力干预，这些事务都将无法继续执行。

一个经典的比喻是**十字路口堵车**：四辆车同时到达十字路口，彼此都占着路，同时都需要对方让路才能自己通过，结果谁都动不了。

---

### 死锁是如何发生的？（4个必要条件）

死锁的发生必须同时满足以下四个条件，缺一不可：

1.  **互斥条件**：一个资源每次只能被一个事务使用。（例如，一行数据被一个事务锁定了，其他事务就不能同时锁定它）。
2.  **请求与保持条件**：一个事务因请求资源而阻塞时，**对已获得的资源保持不放**。（事务A拿着锁X，同时又去申请锁Y）。
3.  **不剥夺条件**：事务已获得的资源，在未使用完之前，**不能被其他事务强行剥夺**。（不能把事务A持有的锁强行抢过来给事务B）。
4.  **循环等待条件**：若干事务之间形成一种**首尾相接的循环等待**关系。（事务A等待事务B释放锁，而事务B又在等待事务A释放锁）。

### 一个典型的死锁场景

假设有两个事务 T1 和 T2，以及两行数据 `Row1` 和 `Row2`。

| 时间 | 事务 T1                                                      | 事务 T2                                                      |
| :--- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 1    | `UPDATE table SET ... WHERE id = 1;` (锁定 Row1)             |                                                              |
| 2    |                                                              | `UPDATE table SET ... WHERE id = 2;` (锁定 Row2)             |
| 3    | `UPDATE table SET ... WHERE id = 2;` (**等待** T2 释放 Row2 的锁) |                                                              |
| 4    |                                                              | `UPDATE table SET ... WHERE id = 1;` (**等待** T1 释放 Row1 的锁) |
| 5    | **死锁发生！**                                               | **死锁发生！**                                               |

此时，T1 在等 T2，T2 在等 T1，形成了一个循环等待，死锁就发生了。

---

### 如何解决死锁？

数据库本身有一个**死锁检测机制**。一旦检测到死锁，它会自动介入解决，**不需要DBA手动去杀死进程**（除非特殊情况）。

数据库的解决策略通常是：**选择一个代价最小的事务作为“牺牲品”**，将其回滚并释放它持有的所有锁。这样其他事务就可以继续执行了。这个牺牲品的事务会收到一个错误（在MySQL中通常是 `ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction`）。

**应用程序收到这个错误后，正确的做法是：**
1.  捕获这个死锁异常。
2.  等待一段随机时间（避免同时重试再次冲突）。
3.  **重试整个事务**。

### **死锁处理方案**

#### **1. 自动处理机制**

* **死锁检测**
  数据库（如InnoDB）自动检测死锁，强制回滚代价较小的事务。

  ```
  SHOW ENGINE INNODB STATUS; -- 查看MySQL死锁日志
  ```

* **锁超时设置**
  设置事务等待锁的超时时间，超时后自动回滚。

  ```
  SET innodb_lock_wait_timeout = 30; -- MySQL设置锁超时时间（秒）
  ```

#### **2. 代码与设计优化**

* **统一资源访问顺序**
  确保所有事务按固定顺序访问资源（如按主键排序更新）。
* **短事务原则**
  减少事务执行时间，尽快提交或回滚，避免长事务占用锁。
* **优化SQL与索引**
  * 为查询条件添加索引，避免全表扫描。
  * 避免批量操作锁大量数据，分批处理。
  * 使用 `SELECT ... FOR UPDATE`时明确指定索引。
* **降低隔离级别**
  使用 `READ COMMITTED`代替 `REPEATABLE READ`，减少间隙锁的使用（需权衡数据一致性）。

#### **3. 程序容错机制**

* **重试策略**
  捕获死锁异常（如MySQL的 `1213`错误码），在代码层重试事务。

  ```
  import org.springframework.retry.annotation.Backoff;
  import org.springframework.retry.annotation.Retryable;
  import org.springframework.stereotype.Service;
  
  @Service
  public class AccountService {
  
      @Retryable(
          value = {DeadlockLoserDataAccessException.class}, 
          maxAttempts = 3, 
          backoff = @Backoff(delay = 1000, multiplier = 2)
      )
      @Transactional
      public void transferWithRetry(int fromId, int toId, BigDecimal amount) {
          // 转账业务逻辑
          jdbcTemplate.update("UPDATE account SET balance = balance - ? WHERE id = ?", amount, fromId);
          jdbcTemplate.update("UPDATE account SET balance = balance + ? WHERE id = ?", amount, toId);
      }
  }
  ```

* **乐观锁**
  通过版本号或时间戳避免显式加锁（适用于冲突较少的场景）。

  ```
  UPDATE table SET col = new_val, version = version + 1 
  WHERE id = 1 AND version = old_version;
  ```

---

### 如何预防和减少死锁？（重点）

虽然无法 100% 避免死锁，但遵循以下最佳实践可以极大地降低其发生概率：

#### 1. 保持事务简短且高效
   - **尽快提交**：事务运行的时间越短，持有锁的时间就越短，与其他事务冲突的窗口期就越小。
   - **不要在事务里做无关操作**：避免在数据库事务中执行网络请求、RPC调用、处理文件等耗时操作。

#### 2. 以固定的顺序访问数据
   - 这是**最重要也是最有效**的预防措施。如果所有事务都按相同的顺序（例如，先更新表A，再更新表B；先操作id小的记录，再操作id大的记录）来访问表和行，就可以打破“循环等待”条件。
   - **反面教材**：一个事务先插订单再扣库存，另一个事务先扣库存再插订单，极易死锁。

#### 3. 为操作创建合适的索引
   - 如果 `UPDATE` 或 `DELETE` 语句的 `WHERE` 子句没有用到索引，MySQL 会进行**表级锁**或锁定**大量不必要的行**，大大增加了死锁的概率。使用索引可以确保只锁定需要的行。

#### 4. 使用低隔离级别
   - 如果业务允许，使用 `READ COMMITTED` 隔离级别比 `REPEATABLE READ` 的锁开销更小，出现死锁的概率也更低。

#### 5. 避免大事务和批量操作
   - 一次性更新成千上万行的事务非常危险，它会长时间持有大量锁。尽量将大批量操作拆分成小批次执行。

#### 6. 使用 `SELECT ... FOR UPDATE` 和 `LOCK IN SHARE MODE` 时要格外谨慎
   - 这些显式加锁的语句会增加死锁的风险，确保你真的需要它们。

#### 7. 使用乐观锁
   - 对于一些并发更新冲突不剧烈的场景，可以考虑使用乐观锁（如版本号 `version` 字段），**在应用层**解决冲突，从而避免数据库层面的行锁竞争。本质上是“无锁”设计。

---

### 当死锁发生时，如何排查？

在MySQL中，可以使用以下命令来分析和监控死锁：

1.  **开启死锁日志**：在 `my.cnf` 中设置 `innodb_print_all_deadlocks = ON`，这样所有死锁信息都会写入错误日志，便于分析。
2.  **查看最近死锁信息**：
    ```sql
    SHOW ENGINE INNODB STATUS;
    ```
    查看输出结果中的 `LATEST DETECTED DEADLOCK` 部分，它会详细记录导致死锁的两个事务的SQL语句、持有的锁和等待的锁，是排查问题的黄金信息。

3. **监控工具**

* Prometheus + Grafana监控锁等待。
* 第三方工具（如Percona Toolkit、pt-deadlock-logger）。