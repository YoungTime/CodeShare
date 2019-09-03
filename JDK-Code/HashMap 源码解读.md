# HashMap 源码解读

  说到 HashMap，大家一定都不会陌生，不管是我们平时使用，或者是面试的时候，都会遇到它，了解其源码还是相当重要的。

  HashMap 其实维护的的数据结构是 Node<K,V>  的数组加链表（下面会说到为什么），也是说的维护的 Hash 桶，什么意思呢？按照笔者个人理解就是说 HashMap 中保存数据的是一个叫 table 的数组，HashMap 有新数据时就会插入到特定生成的数组下标（如何生成下面会说到）中，如果当前下标没有数据就直接放入，只有一个数据的话，数组下标中还是会放入旧的 Node，并且将旧的 Node 作为头指针，将新的 Node 作为旧的 Node 的子节点以链表的方式保存。按照笔者个人理解，每一个数组下标可以看做是一个 Hash 桶，来放一个或多个 Node 数据。

  那么 Node 是什么呢？Node 可以理解为 HashMap 中实际保存每一组 key-value 的映射项，它是实现的 Map.Entry<K,V>，下面我们来看看 Node 的源码。

```Java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
    // 这里有一个对下一个节点的引用，可以看出使用了链表
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }
    
        ...
}
```

  Node 的源码还是一目了然了，就不用了多做解释了，再正式解读 HashMap 的主要方法之前，先看一下一些重要的参数

// 默认的容量大小 为16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

// 最大容量 2的30次方
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认装载因子 为 0.75f
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// HashMap 保存数据使用的数组
transient Node<K,V>[] table;

// HashMap 中实际保存的键值对的数量
transient int size;

// 扩容值，当 size 达到这个值时，HashMap 就要扩容
int threshold; 为 capacity * load factor

## 构造方法
  HashMap 有四个构造方法，我们来看一下这四个方法：

```java
/**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and load factor.
     *
     * @param  initialCapacity the initial capacity 传入的初始化容量
     * @param  loadFactor      the load factor 传入的装载因子
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
public HashMap(int initialCapacity, float loadFactor) {
    // 容量不能为空
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
    // 如果传入的容量已经大于规定的最大容量，就为最大容量
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
    // 装载因子不能小于等于 0
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
    // 设置装载因子和确定扩容值
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

/**
     * @param  initialCapacity the initial capacity. 只传入初始容量
     * @throws IllegalArgumentException if the initial capacity is negative.
     */
public HashMap(int initialCapacity) {
    // 装载因子为默认装载因子 0.75f
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

// 这个应该是我们最常用的构造方法，它所有的字段都是默认字段
public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

// 这个构造方法会传入一个 map
public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
    // 将传的 map put 到我们的 hashMap 中
        putMapEntries(m, false);
    }
```
  上面我们看到使用了 putMapEntries 方法将原 map 放入新的 HashMap，那么它到底干了啥呢？

  ```java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        // 拿到传入的 map 的 size
        int s = m.size();
        if (s > 0) {
            // 如果当前保存 Node 的数组为空
            if (table == null) { // pre-size
                // 使用 size / loadFactor(装载因子) 拿到一个临时值
                float ft = ((float)s / loadFactor) + 1.0F;
                // 如果已经大于规定的最大值就使用最大值
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                        (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    // 确定扩容值，这个方法下面有说明
                    threshold = tableSizeFor(t);
            }
            else if (s > threshold)
                // 如果 Node 数组不为空，并且 size 已经大于扩容值，就需要扩容了
                // 扩容方法下面会讲到
                resize();
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                // 这个方法是真正往 HashMap 存值的方法，下面也会说明
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
  ```

  上面的构造方法我们看到了会传入初始化的容量，其实这个容量并不是真正用来作为数组的容量的，而是为了来确定扩容值的，我们上面看到了的确认扩容值的方法是 tableSizeFor(initialCapacity)，那么这个方法是干嘛的呢？

```java
/**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

​    这个方法是返回一个大于 cap 并且最接近的一个 2的n次幂的一个值来作为扩容值。

  看完了构造方法，我们来看一下 HashMap 的最常用的方法，get 和 put，我们可以看到，其实 put 方法也是调用的 putVal 方法，和上面 putMapEntries 方法实际调用的是同一个方法。

```java
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```

  所以说 put 方法就是使用的 putVal 方法：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        //这里第四个元素，作为临时变量
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // 如果当前 HashMap 为空或者长度为 0
        if ((tab = table) == null || (n = tab.length) == 0)
            // 直接扩容，n 为扩容后数组的长度
            n = (tab = resize()).length;
        // (n-1) & hash 是为了找到当前数据放在数组中的位置
        // (n-1) & hash 其实就是当 n 为 2 的整数次幂时，等于 hash % (n-1)
        // 想知道为啥的话，大家可以看看为啥，或者等笔者下一篇文章
        if ((p = tab[i = (n - 1) & hash]) == null)
            // 直接放入一个新的 Node
            tab[i] = newNode(hash, key, value, null);
        else {
            // 当这个桶里面有数据了，那就意味着要放进链表了
            // 下面讲到
        }
        // modCount 记录了操作 HashMap 的次数
        ++modCount;
        // size 到达了扩容值就会扩容
        if (++size > threshold)
            resize();
        // 也是空方法
        afterNodeInsertion(evict);
        return null;
    }
```

那么是怎么往链表中放的呢？

```java
// 当这个桶里面有数据了，那就意味着要放进链表了
            Node<K,V> e; K k;
            // 上面可以看出，p 为数组当前位置存在的 Node
            if (p.hash == hash &&
                    ((k = p.key) == key || (key != null && key.equals(k))))
                // key 相同直接赋值
                e = p;
            else if (p instanceof TreeNode)
                // 如果使用的是 TreeNode 就用 putTreeVal 方法
                // 感兴趣大家可以去看看 HashMap 使用红黑树
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    // for 循环直到找到子节点为空的时候
                    if ((e = p.next) == null) {
                        // 子节点为空时则放入
                        p.next = newNode(hash, key, value, null);
                        // 这里当链表过长时会转为红黑树
                        // TREEIFY_THRESHOLD 在 HashMap 中值为 8
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    // 或者链表中有 key 相同的直接赋值
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                // onlyIfAbsent 的意思是为 true 就不改变已有的值
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                // 这是个空方法
                afterNodeAccess(e);
                return oldValue;
            }
```

  HashMap 的 put 方法就是这样的，那么 get 方法呢？在 HashMap 中的 get 方法使用的是 getNode 方法：

```java
public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
```

那我们来看看 getNode：

```java
final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
                (first = tab[(n - 1) & hash]) != null) {
            // 当数组不为空并且长度大于0，并且 (n - 1) & hash 处的 Node 不为空
            if (first.hash == hash && // always check first node
                    ((k = first.key) == key || (key != null && key.equals(k))))
                // 如果该位置 key 相同就直接返回
                return first;
            if ((e = first.next) != null) {
                // 如果该节点的子节点不为空
                if (first instanceof TreeNode)
                    // 使用的树节点就按照树节点找
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    // 子节点不为空时就一直循环查找子节点
                    if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                        // 直到找到对应 key 所在的 value
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

  我们看到了，上面 put 数据时会进行扩容操作，那么按道理来说 HashMap 是将 Node 数据放入到了数组下标为 (n - 1) & hash 的 Hash 桶中，当 hash 值一样时便会放入数组下标中 Node 的子节点使用链表保存，那么为什么还需要扩容呢？其实就是因为链表的长度过长时会影响我们 HashMap 的效率，所以说 HashMap 扩容就是将 HashMap 中 Hash 桶中的链表变短，而实际扩容也是这么操作的：

```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            // 旧容量大于 0
            if (oldCap >= MAXIMUM_CAPACITY) {
                // 不能超过最大值，不能再扩容
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 新容量为旧容量 *2 并且小于最大值
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
                // 如果容量小于最大值大于默认值
                //  新的扩容值为旧的 *2
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            // 旧容量为0，但是旧扩容值大于0
            // 新容量就为旧的扩容值
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            // 旧容量和扩容值都为 0
            // 新容量就为默认容量，新扩容值就为默认容量*默认装载因子(cap*0.75f)
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            // 如果新的扩容值为0，只要容量*装载因子的值和容量都小于规定的最大值
            // 扩容值就为容量*装载因子，否则为 Integer 的最大值
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
        }
        // 赋值新的扩容值
        threshold = newThr;
        // 接下来开始扩容的数据迁移
        // 下面马上说
        }
        return newTab;
    }
```

  我们看到了怎么更新容量和扩容值的，那么扩容完不做数据迁移，扩容又有什么意义呢？下面我们就来看看怎么做的数据迁移：

```java
@SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        // 确认新的数组
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                // 遍历旧数组
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    // 旧数组当前节点不为空，先把新数组当前节点置为空
                    oldTab[j] = null;
                    if (e.next == null)
                        // 如果当前节点没有子节点，意思是当前桶中只有一个 Node 数据
                        // 直接将旧节点数据放入节点下标为 e.hash & (newCap - 1) 中
                        // e.hash & (newCap - 1) 的作用上面已经说过
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        // 红黑树还是不涉及
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        // 如果节点的子节点不为空
                        // 这里我们可以理解为定义了一个 lo 链表
                        // loHead 为头指针，loTail 为尾指针
                        Node<K,V> loHead = null, loTail = null;
                        // hiHead 为 hi 链表的头指针，hiTail 为尾指针
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            // 一直循环到子节点为空
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                // 这里我们看到当 (e.hash & oldCap) 为 0时
                                if (loTail == null)
                                    // 判断 lo 的尾指针是否为空
                                    // 为空则插入头指针
                                    loHead = e;
                                else
                                    // 不为空则将新的 Node 插入的 lo 下一个节点
                                    loTail.next = e;
                                loTail = e;
                                // 其实这里总结起来就是当 e.hash & oldCap) == 0 
                                // 将节点 e 插入 lo 链表
                            }
                            else {
                                // 当 (e.hash & oldCap) 不为 0时
                                if (hiTail == null)
                                    // 判断 hi 的尾指针是否为空
                                    // 为空则插入头指针
                                    hiHead = e;
                                else
                                    // 不为空则将新的 Node 插入的 hi 下一个节点
                                    hiTail.next = e;
                                hiTail = e;
                                // 其实这里总结起来就是当 e.hash & oldCap) == 0 
                                // 将节点 e 插入 hi 链表
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            // 如果 lo 链表不为空
                            // 则将 lo 链表放入数组的当前位置 j
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            // 如果 hi 链表不为空
                            // 则将 hi 链表放入数组的新位置 j + oldCap
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
```

  为什么 hi 链表要放入数组的下标为 j + oldCap 中呢？这个就不得不说写这个源码的大佬的强大了，这里正好解释了上面为什么用要使用 e.hash & oldCap 来判断是插入到 lo 链表还是 hi 链表了。

  我们知道，put 时是将数据放入到下标为 **(n - 1) & hash**  中，那么 (n - 1) & hash 是干了啥呢？我们上面说了 (n - 1) & hash 就等于 hash %  (n - 1) ，其实在二进制中，取余就是取低位，这个是什么意思呢？因为 n 是2的整数次幂，这里我们整数为 m：

![hashmap_1_1](/image/hashmap_1_1.png)

那么 n - 1 呢

![hashmap_1_2](/image/hashmap_1_2.png)

  都知道 0 & 任何数为0，1 & 任何数为任何数，所以  (n - 1) & hash 就是拿到了 hash 值的低 m 位，那么扩容后呢？上面说过了扩容是新的容量为旧容量*2，如果新容量为 n1，那么 n1 = 2*n = 2^(m+1)，那么(n1 - 1) & hash 就是取 hash 值的低 m+1 位，那么新位置与旧位置差了多大呢？

  通过上面的代码可以看到，距离差为 j + oldCap - j = oldCap，距离差就为 oldCap，为什么是这个呢？按照上面说的距离差应该是 (n1 - 1) & hash - (n - 1) & hash，那么就是 hash 值的后 m+1 位减去 hash 值的后 m 位，那么就等于 hash 的二进制只有第 m+1 位不确定，其它为都为 0 的值。二进制中就只有 0 和 1，如果为 0 ，那么距离差就为 0，那么就是 j 的位置，那如果 第 m+1 为 1 呢？记得上面的图吗？n 的 1 后面有 m 个 0，那么第 m+1 位为 1 那不就是 n，也就是 oldCap 吗？所以新的位置就在 j + oldCap。

  那么我们怎么拿到第 m+1 位是 0 还是 1 呢？你们看 oldCap 也就是 n，只有第 m+1 位为 1 ，1 & 任何数为任何数，0 & 任何数为 0，那么第 m+1 位的值不就等于  e.hash & n，也就是 e.hash & oldCap 吗？

  所以我们源码中才会判断当 e.hash & oldCap 为 0 时将节点插入到 lo 链表，不为 0 时插入到 hi 链表，我们的扩容就将一个链表巧妙的分成了两个链表。

  好了，HashMap 的源码主要部分就先说到这里，如果想要一起学习话，可以持续关注这个项目，或者关注我的个人微信公众号。

![hashmap_1_3](/image/hashmap_1_3.png)

  觉得喜欢的话请给我一个 Star，谢谢！

 



