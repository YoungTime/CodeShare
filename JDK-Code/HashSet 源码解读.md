# HashSet 源码解读
  HashSet 也是 Java 集合中一个相对常用的，其内部实现比较简单，而且其内部是相当于维护了一个 HashMap，可以先看一下 [HashMap 源码解读](https://github.com/YoungTime/CodeShare/blob/master/JDK-Code/HashMap%20%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB.md)。与 HashMap 维护的 key-value 两个成员不同的是，HashSet 只相当于维护了一个成员 E，虽然内部使用的 HashMap，但是内部维护的 HashMap 的 value 是一个 final 的 Object，所以相当于 HashSet 是维护了 HashMap 的 key。

## HashSet 的内部元素

```java
// HashSet 内部维护的 HashMap，key 为泛型 E，value 为 Object
private transient HashMap<E,Object> map;
// 这个 final 的 Object 就是维护的 HashMap 保存的 value，不变
private static final Object PRESENT = new Object();
```

## HashSet 的构造方法

  HashSet 有 5 个构造方法，其中四个对应的是 HashMap 的四个构造方法，一个对应的是LinkedHashMap，所以如果想了解 HashSet 的源码，建议先看一下 [HashMap 的源码](https://github.com/YoungTime/CodeShare/blob/master/JDK-Code/HashMap%20%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB.md)。

  前四个构造方法：

```java
public HashSet() {
        map = new HashMap<>();
    }

public HashSet(Collection<? extends E> c) {
        // HashMap 中默认装载因子是 0.75
        // 所以用 size/0.75 + 1 与 HashMap 默认容量 16 比较，取较大值
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }

public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }

public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }
```

  我们可以看到，前四个构造方法，完全就是新建了一个 HashMap，使用了 HashMap 不同的构造方法，传入的容量或者装载因子。第五个构造方法呢？

```java
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
```

  这个构造方法也是初始化了一个 LinkHashMap，而其中的 dummy 参数并没有什么意义，只是为了让这个构造方法与上面第三个构造方法区分开。

## add 方法

  其实 HashSet 的 add 方法也是调用了 HashMap 的 put 方法：

```java
public boolean add(E e) {
        // HashSet 的 add，就是将元素放到 HashMap 的 key 的位置
        // value 就放的是我们上面说到的 final 的 Object 实例
        return map.put(e, PRESENT)==null;
    }
```

## 其它方法

  其实 HashSet 中的很多方法都是调用的 HashMap 的方法，如：iterator、size、contains、remove、clear 等等：

```java
public Iterator<E> iterator() {
        return map.keySet().iterator();
    }

public int size() {
        return map.size();
    }

public boolean isEmpty() {
        return map.isEmpty();
    }

public boolean contains(Object o) {
        return map.containsKey(o);
    }

public boolean remove(Object o) {
        return map.remove(o)==PRESENT;
    }

public void clear() {
        map.clear();
    }
```

## 总结

  这么看起来，其实 HashMap 就是一个只维护了 key 的 HashMap，所以 HashMap key 的属性就是 HashSet 元素的属性，也就是说因为 HashMap 的 key 可以为空但是不能重复的，所以 HashSet 的元素就是可以为空但是不能重复的。

  好了，关于 HashSet 的源码分享就是这些了，其实只要了解了 HashMap 的源码，HashSet 的源码内容就大致了解。如果想要一起学习的话，可以持续关注这个 [GitHub项目](https://github.com/YoungTime/CodeShare)，或者关注我的个人微信公众号。

![hashmap_1_3](../image/hashmap_1_3.png)

  觉得喜欢的话请给我这个 [GitHub 项目](https://github.com/YoungTime/CodeShare) 一个 Star，谢谢！ 