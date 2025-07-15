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

## 为什么要在 1.8 之后调整插入元素的策略

在 1.8 之后将元素的头插法修改为了尾插法，其目的是为了避免在多线程的情况下导致链表插入出现环路，最终引发死循环。但其实 HashMap 本身就并非是线程安全的，所以在并发业务中应该使用其他线程安全的字典容器代替。

