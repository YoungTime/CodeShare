# HashMap 源码解读

  说到 HashMap，大家一定都不会陌生，不管是我们平时使用，或者是面试的时候，都会遇到它，了解其源码还是相当重要的。

  HashMap 其实维护的的数据结构是 Node<K,V>  的数组加链表（下面会说到为什么），那么 Node 是什么呢？

  Node 可以理解为 HashMap 中实际保存每一组 key-value 的映射项，它是实现的 Map.Entry<K,V>，下面我们来看看 Node 的源码。

```Java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
    // 这里有一个对下一个节点的运用，可以看出使用了链表
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





