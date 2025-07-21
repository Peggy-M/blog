## 说说 Java 中得 HashMap 的原理

首先 `HashMap` 的底层是基于 **哈希表** 实现的，由于实现的特性决定了其本身在查询、插入、以及删除等方面的操作具有其较高的执行效率。

- **基本数据结构**

`HashMap` 的底层是由 **数组+链表/红黑树** 的结构实现的，以数组作为存放头节点的桶（bucket），采用链表/红黑树的方式来解决在添加元素的时候键值出现的 Hash 冲突问题。

- **put 方法插入元素**

假设插入一个元素`map.put("abc"，123);` 

1. 首先会对该插入元素的 key 值 "abc" 进行一次 hash 扰动运算,减少发生碰撞的概率；
2. 在 `tab[i = (n - 1) & hash])` 确定数组索引槽位既桶的下标之后，判断该桶是否为 null,如果为 null 则直接插入元素
3. 否则产生 hash 碰撞
   - 如果键值相同则直接覆盖旧值
   - 如果键不相等,则使用链表的形式进行尾插法
   - 如果链表的长度超过 8 并且数组长度超过 64 则转为 红黑树 

- **get 方法获取元素**

假设获取一个元素 `map.get("abc");` 

1. 首先计算 `tab[(n - 1) & hash]` 计算定位到数组索引槽的位置
2. 检查首个元素是否是要插值的值
3. 遍历链表/红黑树进行查找，如果找到返回，否则最终吧，遍历结束未找到返回 NULL

- **resize 扩容机制**

当前的元素数量如果超过阈值 `负载因子 loadFactor * 数组长度` 时， HashMap 会进行扩容, loadFactor 的默认值是 0.75

1. 扩容之后的桶大小为之前的 2 倍
2. 在扩容之后会变量重新计算每一个 key 的 hash 值，因此在着期间可能会出现原有元素的位置变动迁移
3. 在 1.8 之后，扩容时链表的节点位置要么保持原有的位置不变动，要么便宜旧容量大小而不需要重新 hash。

- **线程安全方面**

  `HashMap` 由于底层的实现机制，所有其本身是 **非线程安全的**，因此在多线程并发的环境下更多的是倾向于使用线程安全的 `ConcurrentHashMap`

~~~ java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; 
    Node<K,V> p; 
    int n, i;
    //判断当前的 table 是否为 NULL 或长度为 0
    if ((tab = table) == null || (n = tab.length) == 0)
    //如果为 NULL 或 长度为 0 ,调用 resize 进行扩容
        n = (tab = resize()).length;
        //当前的桶长度-1并进行 hash 得到数组的下标
    if ((p = tab[i = (n - 1) & hash]) == null)
        //如果当前的数组下标是空的直接新节点作为头节点
        tab[i] = newNode(hash, key, value, null);
    else {
        //不为空-解决hash冲突的问题
        Node<K,V> e; K k;
        //情况1-当前桶中第一个节点就是目标 key
        //检查当前 p.hash(当前链表或树节点的哈希值) 与 hash（要插入的新键的 hash 值）是否相等
        //这里有一个细节，优先判断 hash 值是否相同如果 hash 值不相同，则后面的判断就没有必要继续 
        //[这里列举一个特别简单的 hash 算法，比如对一个数进行取余，该余数作为 hash 值，余数相同也就是说hash值相同，但并不一定代表着取余的两个数是相同的，但若是两个数的余数不相同，那一定表示这两个数一定不相同]
        if (p.hash == hash &&
        //将当前节点的 key 值赋值给变量 k ,并与插入元素的 key 进行比较是否相同
            ((k = p.key) == key || (key != null && key.equals(k))))
            //表示两个 key 值是同一个对象
            e = p;
        //情况2-节点是一个树节点
        else if (p instanceof TreeNode)
        //否则判断该节点是否是一个红黑树节点，如果是一个树节点则调用树的插入方法
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        //情况3-链表结构
        else {
            //表里链表
            for (int binCount = 0; ; ++binCount) {
                //如果直到遍历到链表的尾部,仍然未找到对应的 key
                if ((e = p.next) == null) {
                    //表示为新元素，则创建一个新的节点并追加到链表尾部
                    p.next = newNode(hash, key, value, null);
                    //如果链表的长度达到了树化的阈值 8 则调用 treeifyBin 方法转为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                        //追加元素完成跳出循环
                    break;
                }
                //与情况1 当中查询 key 值逻辑一样
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    //找到对应的 key 值跳出循环
                    break;
                p = e;
            }
        }
        //在前面的操作当中，如果找到相同的 key 此处的 e 一定是不为 NULL 的
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // onlyIfAbsent 默认为 false key 相同的 value 值会被覆盖
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount; //当前新插入元素的修改次数+1
    if (++size > threshold) //如果超出阈值则进行扩容
        resize();
    afterNodeInsertion(evict);
    return null;
}
~~~



~~~ java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; 
    Node<K,V> first, e; 
    int n; K k;
    //与 put 方法的前置代码一样,先获取到数组索引槽的位置
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        //判断当前槽的首个元素的 key 是否相同,如果相同则直接返回
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        //表示首个元素不是获取首个元素的下一个元素
        if ((e = first.next) != null) {
                  //否则判断是否是树节点
            if (first instanceof TreeNode)
                //调用 getTreeNode 方法查找树节点元素
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            //节点为链表循环链表遍历查找
            do {

                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    //如果以上都没有查找到,则直接返回 NULL
    return null;
}
~~~

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/1462/1708924487010/698cb1a0cb694799ac76e69c1bc1552c.png)

## 为什么要在 1.8 之后调整插入元素的策略

在 1.8 之后将元素的头插法修改为了尾插法，其目的主要是为了避免在多线程的情况下头插法会导致链表插入出现环路，最终引发死循环。但其实 HashMap 本身就并非是线程安全的，所以在并发业务中应该使用其他线程安全的字典容器代替。

其主要是在 `resize()` 扩容方法 在并发环境下内部在调用 `threshold()` 方法进行数据迁移的时候，可能会引发死循环的问题，其具体代码如下：

~~~ java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {  // 遍历旧数组
        while(null != e) {         // 遍历链表
            Entry<K,V> next = e.next;  // 记录下一个节点
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);  // 计算新位置
            e.next = newTable[i];  // 头插法关键操作
            newTable[i] = e;      // 将e放到新位置
            e = next;             // 处理下一个节点
        }
    }
}
~~~

**死循环产生的具体过程:**

假设HashMap中有一个桶的链表结构：

~~~ java
Entry1 -> Entry2 -> null
~~~

- **阶段1：线程A开始执行但被挂起**

~~~ java
//线程A开始执行transfer方法：
while(null != e) {         // e = Entry1
    Entry<K,V> next = e.next;  // next = Entry2 (记住这个值)
    // 线程A在这里被挂起，还没执行下面的代码
    // 此时线程A的局部变量：
    // e = Entry1
    // next = Entry2
~~~

- **阶段2：线程B完整执行transfer**

  - 处理Entry1：
    - next = Entry2
    - 将Entry1插入新表：`newTable[i] = Entry1`，且`Entry1.next = null` (因为是第一个插入的)

  - 处理Entry2：
    - next = null
    - 将Entry2插入新表：`newTable[i] = Entry2`，且`Entry2.next = Entry1` (头插法)

  - 最终新表中的链表变为

    ~~~ java
    Entry2 -> Entry1 -> null
    ~~~

- **阶段3：线程A恢复执行**

  1. 线程A恢复执行，它仍然持有：

     - e = Entry1
     - next = Entry2 (这是在挂起前保存的值)

  2. 线程A继续执行：

     ~~~ java
     e.next = newTable[i];  // newTable[i]当前是Entry2(线程B插入的)
                           // 所以 Entry1.next = Entry2
     newTable[i] = e;      // 将Entry1放到链表头部
                          // 现在 newTable[i] = Entry1
     e = next;            // e = Entry2
     ~~~

     此时链表状态：

     ~~~ java
     newTable[i]: Entry1 -> Entry2 -> Entry1 -> Entry2 ->... (实际上已经成环)
     ~~~

     但让我们继续看下一轮循环如何完成这个环。

- **阶段4：线程A下一轮循环**

  1. 线程A进入下一轮while循环(e = Entry2)：

     ```java
     Entry<K,V> next = e.next;  // e = Entry2
                               // 由于线程B的操作，Entry2.next = Entry1
                               // 所以 next = Entry1
     ```

  2. 继续执行：

     ```java
     e.next = newTable[i];    // newTable[i]当前是Entry1
                             // 所以 Entry2.next = Entry1
     newTable[i] = e;        // 将Entry2放到链表头部
                            // 现在 newTable[i] = Entry2
     e = next;              // e = Entry1
     ```

     现在的关系：

     - Entry2.next = Entry1
     - Entry1.next = Entry2 (这是前面设置的)
       这样就形成了完整的环：

     ```java
     Entry1 <-> Entry2
     ```

- **阶段5：无限循环**

1. 线程A继续下一轮循环(e = Entry1)：

   ```java
   Entry<K,V> next = e.next;  // e = Entry1, Entry1.next = Entry2
                             // 所以 next = Entry2
   ```

2. 执行：

   ```java
   e.next = newTable[i];    // newTable[i]是Entry2
                           // 所以 Entry1.next = Entry2 (没变化)
   newTable[i] = e;        // newTable[i] = Entry1
   e = next;              // e = Entry2
   ```

3. 下一轮循环(e = Entry2)：

   ```java
   Entry<K,V> next = e.next;  // e = Entry2, Entry2.next = Entry1
                             // 所以 next = Entry1
   ```

## 为什么 JDK 1.8 会引入红黑树

在不引入红黑树的情况下，当发生 hash 冲突之后采用的拉链法，如果说 hash 分布并不理想（在一般的情况下出现多次 hash 冲突这种概率是非常小的，只有在特殊情况下，恶意攻击者可会构造大量的哈希值相同的键值进行工具，导致链表会变得特别长，造成服务器卡顿），那这条链就会变得非常长，因此在每一次对元素的操作当中就会出现先定位hash桶下表，然后遍历链表，时间复杂度为 O(n) 随着链路越长每一次操作查找定位元素的效率就会越低。而在1.8当链表转为红黑树之后时间复杂度降为O(long n)，比如说查找一个元素，长度为 100 的链表平均为 50 次比较，而红黑树最多只需要 7 次。

**为什么不选择其他的数据结构呢？**

- 不用 AVL 树

  - AVL树更严格平衡，查询稍快但插入/删除更慢

  - 红黑树的统计性能更好，适合频繁修改的场景

- 不用跳表（Skip List）

  - 跳表需要更多指针，内存开销更大

  - 红黑树在Java中实现更成熟

- 不用完成平衡树

  - 完全平衡维护成本过高

  - 红黑树的"近似平衡"已经足够好

**为什么一开始就不直接，使用红黑树而是达到阈值，通过链表转化为红黑树呢**

首先红黑树是比较复杂的在插入和删除元素的时候，它需要通过左旋、右旋的操作来保持平衡，而对于单链表而言是完成可以满足查询的效率的，但如果链表一旦过长为了提高查询效率就必须转为红黑树。如果说在一开始就直接使用红黑树反而会在新增元素时会降低效率，是一种性能的浪费。

**为什么默认的加载因子为 0.75    **

默认加载因此之所以为  0.75 是因为 Java 设计者 **空间成本** 与 **时间成本** 之间的权衡后的结果，该值是基于统计学原理经过数学分析和验证的。其主要的原因是为了保持**空间与时间的平衡**，当 **较低值(如0.5)**，更早的扩容，哈希冲的可能性降低，查询的效率更好，但这样会造成空间的使用率降低，内存存在较多的浪费。而 **较高值(如0.9)**，扩容更晚，空间利用率虽然提高了，但与之同时哈希冲突会增加，导致查询的性能会降低，因此 **0.75** 在两者之间取到了很好的平衡，空间的利用率保持在75%左右，而哈希冲突有又在合理的水平 ，并且保证桶的命中符合 **松柏分布**。

**为什么要在 hash 方法当中进行hash时右移动16位**

~~~ java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
~~~

因为在 HashMap 进行计算桶的位置的时候使用的是 `index = (n - 1) & hash` ，其中 `n` 是桶的数量也就是数组的长度，但问题就是如果 `n` 的值比较小比如默认的初始化为 16 ，在 `(n - 1) & hash` 只会用到  `hash` 的 **低4位** ，如果 `hashcode()` 计算的值高 16 位变化很大，但低 16 位置变化不大，就会导致哈希冲突，那么为了解决这个问题，就是尽可能的让低位与高位都能参与到 hash 运算，所以将 `hashCode()` 计算的结果向右偏移 16 位置，让高的信息可以影响到低位，使得低 16 位也包含高 16 位得特征，从而减少 hash 冲突而导致得桶下标重复命中的几率。

~~~  
h              = 00010010 00110100 01010110 01111000 (0x12345678)
h >>> 16       = 00000000 00000000 00010010 00110100 (0x00001234)
-------------------------------------------------- XOR
result         = 00010010 00110100 01000100 01001100 (0x1234444C) 
//在向右偏移之后再进行异或,这样就保证了低位当中也是包含了高位的值的，在桶数量较小的时候进行 (n - 1) & hash 就会有较好的 hash 分布
~~~

**为什么 Hash 值要与 length-1 相与**

~~~ java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
              .......  ....
        if ((p = tab[i = (n - 1) & hash]) == null)  //i = (n - 1) & hash
            tab[i] = newNode(hash, key, value, null);
        else {
                .......  ....
            }
        return null;
    }
~~~

其实这样做是主要是出于 **性能优化** 与 **hash一致性考虑**，首先位运算一定是比取余运算要快的，通过可能快几十个时钟周期，又由于 HashMap 当中的桶容量 `n` 总是 2 的幂比如 （16、32、64），即便是初始化给定的容量是一个奇数，也会被自动调整成 2 的幂，比如 17 会被自动调整成 32，因此从运算的角度来看 `( n - 1 ) & hash`  的结果值与 ` hash % n` 始终是一样的。这样就保证了每一次的 hash 都是一致的，同时又使用了运算效率更高的位运算代替了取余运算。

## 什么是 Hash 碰撞，怎么解决 Hash 碰撞

前面已经在很多方面重复提到过 Hash 碰撞，这里其实都不需要再次说明，关于 Hash 碰撞解决的办法，在 HashMap 方法当中当发生碰撞的时候，最简单的办法拉链法+二次扰动。

如果只是单纯的对与 Hash 碰撞的解决方案而言，比如要 **开发寻址法**，其实就是所以的元素都是存在哈希表数组的，发生碰撞时，按照某种探测方法寻找下一个空槽，比如**线性探测**(按照顺序查找)、**二次探测**(按平方数跳跃)、**双重哈希**(使用第二个哈希函数计算迁移步长)，还比如使用 **再次哈希**，多准备几个 Hash 函数，当第一个发生碰撞，尝试第二个，第三个等；还比如 **优质哈希函数动态扰动**、**动态扩容** 等。

## Java 的 CopyOnWriteArrayList 和 Collections.synchronizedList 有什么区别？分别有什么优缺点？

**CopyOnWriteArrayList**

 









