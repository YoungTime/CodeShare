# ArrayList 源码分析

  ArrayList 是 Java 中比较常用的集合之一。它实现了 List<E>, RandomAccess, Cloneable, java.io.Serializable 这四个接口，List<E> 接口是为了让 ArrayList 去实现 List 的各种方法，实现 RandomAccess 是为了让 ArrayList 支持快速快速随机访问，实现 Cloneable 是为了让 ArrayList 支持 clone() 方法，实现 Serializable 是为了让 ArrayList 支持序列化。

  上面实现的 List 和 Serializable 接口可能大家都没什么疑惑，但是 RandomAccess  和 Cloneable 呢？我们看 RandomAccess  和 Cloneable  的源码，发现这两个接口都是空的。

![arraylist_1_1](../image/arraylist_1_1.png)

![arraylist_1_2](../image/arraylist_1_2.png)

其实这两个空的接口都是标记型接口，什么意思呢？就是来标记一个类是否属于使用快速随机访问个 clone() 方法的标记。

比如标记型接口是怎么标记是否可以使用快速随机访问的呢？在集合类 Collections 中我们可以看到：

![arraylist_1_3](../image/arraylist_1_3.png)

是否实现了 RandomAccess  接口使用的搜索方法是不一样的。那么 Cloneable 的标记呢？我们知道 clone() 方法是 Object 类的，所以直接来看 Object 的源码：

![arraylist_1_4](../image/arraylist_1_4.png)

我们看到了，如果没有实现 Cloneable 接口而使用 clone() 方法的话是会抛出 CloneNotSupportedException。

