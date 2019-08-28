# HashMap 源码解读

  说到 HashMap，大家一定都不会陌生，不管是我们平时使用，或者是面试的时候，都会遇到它，了解其源码还是相当重要的。
  HashMap 其实维护的的数据结构是数组加链表，

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