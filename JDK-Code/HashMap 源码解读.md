# HashMap 源码解读

  说到 HashMap，大家一定都不会陌生，不管是我们平时使用，或者是面试的时候，都会遇到它，了解其源码还是相当重要的。
  HashMap 其实维护的的数据结构是 Node<K,V>  的数组加链表（下面会说到为什么），那么 Node 是什么呢？
  Node 可以理解为 HashMap 中实际保存每一组 key-value 的映射项，它是实现的 Map.Entry<K,V>，下面我们来看看 Node 的源码。

```Java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
    // 这里有一个对下一个节点的运用，可以看出使用了指针
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

// 默认加载因素 为 0.75f
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// HashMap 保存数据使用的数组
transient Node<K,V>[] table;

// HashMap 中实际保存的键值对的数量
transient int size;

// 扩容值，当 size 达到这个值时，HashMap 就要扩容
int threshold;
