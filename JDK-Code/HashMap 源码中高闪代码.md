# HashMap 源码中高闪代码

  HashMap 源码中有很多的东西是值得我们去认真琢磨的，这里笔者就分享其中的一些超级棒的地方，当然还有更多的地方值得大家去努力探寻。

## 2 的整数次幂与 (n-1) & hash 

  在笔者的 [HashMap 源码解析](https://github.com/YoungTime/CodeShare/blob/master/JDK-Code/HashMap%20%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB.md) 中说到了，HashMap 中规定了，数组 table 的容量以及扩容值都必须为 2 的整数次幂。在 HashMap 中，元素 Node 在数组中的位置是 hash 对数组容量取余，也就是 hash % n，而且也说到了 (n-1) & hash 相当于 hash % n，这样的好处我们当然一眼就能看出来，<font color=red>位运算肯定是比取余要快</font>，那为什么当 n 为 2 的整数次幂时，(n-1) & n 就等于 hash % n？我们来看一下。



  要明白为什么这两个作用相等，其实主要是明白取余在二进制上意味着什么，如果是 a % b（HashMap 中 a 与 b 都为 int 整数），那么就是说整数 a 可以分为多少个整数 b 后，并且还能剩余的整数。如果是一个随机的整数，换算成 2 进制时，完全不知道会有多少个 1，多少个 1，怎么可能在二进制中非常明了的取余是什么呢？比如我们取 5 % 3：

   ![hashmap_2_1](../image/hashmap_2_1.png)

  如果大家只看二进制，能够看出 5 % 3 等于多少吗？当然是很不容易的。当然大家不要想着 10 进制里面的 5 % 3 = 2，要不然你就是在为难我胖虎。

![hashmap_2_2](../image/hashmap_2_2.jpg)

  那么怎么样才能在二进制快速找到一个数的余数呢？只要我们的 b 的二进制表示只有一个 1 不就行了？看图：

![hashmap_2_3](../image/hashmap_2_3.png)

  这样看 a % b 是不是一下子就明了，知道 a / b 和 a % b 的值时什么了。啥？还是不知道？

![hashmap_2_4](../image/hashmap_2_4.png)

a 的前面 x 位等于有多少个 b，a 的后 y 位表示 a % b 为多少。我们分别将 a 的前 x 位置为 0，和后 y 位置为 0：

![hashmap_2_5](../image/hashmap_2_5.png)

![hashmap_2_6](../image/hashmap_2_6.png)

