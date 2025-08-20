# Redis 的秒杀库存减单防止超卖场景

### 1. 为什么 Redis + Lua 可以防止超卖

- Redis 的 **单线程** + **Lua 脚本原子执行** 特性：
   一段 Lua 脚本执行时，中间不会被别的命令打断。
- 这样你就能在 **一个脚本里完成多步操作**：
  - 判断库存是否 > 0
  - 扣减库存
  - 记录用户（防止一人多单）
  - 写入异步队列
- 因为是一个事务性的整体操作，所以不会出现“两个线程同时看到库存够 → 都减库存 → 超卖”的情况。

####  **未使用 Lua 脚本**

- Redis 是单线程的，每一条命令（比如 `GET`、`DECR`）本身是**原子的**。
- 但是业务逻辑是多条命令拼出来的（`GET → 判断 → DECR`）。
- 这几条命令之间存在**空隙**，在空隙里别的线程可以插进来执行。
  - 所以会出现：A 和 B 都读到 `stock=1`，然后都去 `DECR`，最终库存变成 -1。

------

####  **使用 Lua 脚本**

- Redis 的执行模型是：**一次只能执行一个命令**（包括 `EVAL`/`EVALSHA` 执行 Lua 脚本）。
- 当你把“判断 + 扣减库存 + 记录用户”都写在一个 Lua 脚本里执行时：
  - 线程 A 调用 `EVALSHA` → Redis 把整段脚本一次性执行完（脚本里所有命令是一个整体）。
  - 在 A 的脚本没执行完之前，Redis 不会去切换执行 B 的脚本。
  - 所以 B 必须等 A 执行结束后才能开始执行。
- 这样就不会出现“两个线程同时读到同一个库存”的情况。

------

#### 类比理解

- **未用 Lua**：你把几步业务逻辑拆成了几次去银行窗口办事，A 和 B 有可能交叉办理，导致混乱。
- **用了 Lua**：你把整个业务打包成一份材料，交给银行窗口“一次性办完”，下一个人只能等前一个人办完再轮到。

------

#### 再确认你的总结

> “如果使用了 Lua 脚本，在同一时刻只能有一个线程执行，比如线程 A 执行，而线程 B 就无法执行，必须等到线程 A 执行完毕后，线程 B 才可以执行”

✅ **完全正确**。这就是 Redis 单线程 + Lua 原子性的本质。

------

### 2. 没有用 Lua 的反例（会超卖）

假设只用普通的 Redis 命令：

```pseudo
# 伪代码（没有 Lua）
stock = GET seckill:stock:1001
if stock > 0:
    DECR seckill:stock:1001
    SADD seckill:users:1001 userA
    return SUCCESS
else:
    return FAIL
```

#### 并发场景

- **线程 A** 和 **线程 B** 几乎同时执行：
  - A 执行 `GET` → 读到 stock = 1
  - B 执行 `GET` → 也读到 stock = 1
- 两边判断都满足 `> 0`
- A 执行 `DECR` → 库存变成 0
- B 也执行 `DECR` → 库存变成 -1 ❌（超卖）

> 原因：Redis 单条命令是原子，但“多条命令组合”之间是有空隙的。不同客户端可以在空隙里插进来，导致并发逻辑错误。

------

### 3. 用 Lua 改造后的写法（不会超卖）

```lua
local stockKey = KEYS[1]
local userSetKey = KEYS[2]
local userId = ARGV[1]

local stock = tonumber(redis.call('GET', stockKey) or "0")
if stock <= 0 then
    return 1  -- 库存不足
end

if redis.call('SISMEMBER', userSetKey, userId) == 1 then
    return 2  -- 已买过
end

redis.call('DECR', stockKey)
redis.call('SADD', userSetKey, userId)
return 0  -- 成功
```

- 这段脚本执行时，Redis 不会让别的命令插进来。
- 所以不可能出现两个线程同时读到 stock=1 的情况。
- 结果就是：要么线程 A 成功，要么线程 B 成功，永远不会超卖。

------

✅ **总结**：

- **Redis 单条命令原子**，但“多条命令组合”不原子 → 可能超卖。
- **Redis + Lua 脚本原子**，把所有逻辑塞进一个脚本里执行，就能彻底避免超卖。

------

好 👌 我帮你写一个最小可运行的 **Java Demo**，展示两种情况：

------

### 4. Maven 依赖

确保你的项目引入了 Jedis（最简单直观）：

```xml
<dependency>
  <groupId>redis.clients</groupId>
  <artifactId>jedis</artifactId>
  <version>4.3.2</version>
</dependency>
```

------

### 5. Java Demo 代码

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;

import java.util.Collections;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class SeckillDemo {

    private static final String STOCK_KEY = "seckill:stock";
    private static final int THREADS = 50;  // 模拟并发用户数
    private static final int INIT_STOCK = 10;

    public static void main(String[] args) throws InterruptedException {
        JedisPool jedisPool = new JedisPool("127.0.0.1", 6379);

        // 先初始化库存
        try (Jedis jedis = jedisPool.getResource()) {
            jedis.set(STOCK_KEY, String.valueOf(INIT_STOCK));
        }

        System.out.println("====== 测试普通命令，会超卖 ======");
        testNormal(jedisPool);

        // 重置库存
        try (Jedis jedis = jedisPool.getResource()) {
            jedis.set(STOCK_KEY, String.valueOf(INIT_STOCK));
        }

        System.out.println("\n====== 测试Lua脚本，不会超卖 ======");
        testLua(jedisPool);

        jedisPool.close();
    }

    // 普通命令：GET → 判断 → DECR
    private static void testNormal(JedisPool jedisPool) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(THREADS);
        ExecutorService executor = Executors.newFixedThreadPool(THREADS);

        for (int i = 0; i < THREADS; i++) {
            executor.submit(() -> {
                try (Jedis jedis = jedisPool.getResource()) {
                    String stockStr = jedis.get(STOCK_KEY);
                    int stock = stockStr == null ? 0 : Integer.parseInt(stockStr);

                    if (stock > 0) {
                        jedis.decr(STOCK_KEY);
                        System.out.println(Thread.currentThread().getName() + " 抢购成功");
                    } else {
                        System.out.println(Thread.currentThread().getName() + " 库存不足");
                    }
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await();
        try (Jedis jedis = jedisPool.getResource()) {
            System.out.println("最终库存 = " + jedis.get(STOCK_KEY));
        }
        executor.shutdown();
    }

    // Lua脚本：判断 + 扣减原子执行
    private static void testLua(JedisPool jedisPool) throws InterruptedException {
        String luaScript =
                "local stock = tonumber(redis.call('GET', KEYS[1]) or '0') " +
                "if stock <= 0 then " +
                "   return 1 " + // 库存不足
                "end " +
                "redis.call('DECR', KEYS[1]) " +
                "return 0 ";

        try (Jedis jedis = jedisPool.getResource()) {
            String sha1 = jedis.scriptLoad(luaScript);

            CountDownLatch latch = new CountDownLatch(THREADS);
            ExecutorService executor = Executors.newFixedThreadPool(THREADS);

            for (int i = 0; i < THREADS; i++) {
                executor.submit(() -> {
                    try (Jedis j = jedisPool.getResource()) {
                        Object result = j.evalsha(sha1, Collections.singletonList(STOCK_KEY), Collections.emptyList());
                        if (Integer.valueOf(result.toString()) == 0) {
                            System.out.println(Thread.currentThread().getName() + " 抢购成功");
                        } else {
                            System.out.println(Thread.currentThread().getName() + " 库存不足");
                        }
                    } finally {
                        latch.countDown();
                    }
                });
            }

            latch.await();
            try (Jedis j = jedisPool.getResource()) {
                System.out.println("最终库存 = " + j.get(STOCK_KEY));
            }
            executor.shutdown();
        }
    }
}
```

------

### 6. 运行结果示例

假设初始库存 `10`，并发用户 `50`：

- **普通命令版**：

  ```
  ... 抢购成功 x10
  ... 抢购成功 x几个
  最终库存 = -5   <-- ❌ 出现超卖
  ```

- **Lua 脚本版**：

  ```
  ... 抢购成功 x10
  ... 库存不足 x40
  最终库存 = 0   <-- ✅ 不会超卖
  ```

------

这样你跑一次就能直观看到：
 **Lua 脚本原子执行**，防止超卖；而普通命令分步执行，在并发下会出问题。