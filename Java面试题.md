# 容器

### Collection和Map类图
![[Pasted image 20230822202106.png]]

### ArrayList 和 Vector 的区别？
同步性不同
Vector 是线程安全的，它的方法之间都加了 synchronized 关键字修饰，而 ArrayList 是非线程安全的。因此在不考虑线程安全情况下使用 ArrayList 的效率更高。

数据增长不同
ArrayList 和 Vector 都有一个初始的容量大小，并在必要时对数组进行扩容。但 Vector 每次增长两倍，而 ArrayList 增长 1.5 倍。


### LinkedList 为什么不能实现 RandomAccess 接口？

`RandomAccess` 是一个标记接口，用来表明实现该接口的类支持随机访问（即可以通过索引快速访问元素）。由于 `LinkedList` 底层数据结构是链表，内存地址不连续，只能通过指针来定位，不支持随机快速访问，所以不能实现 `RandomAccess` 接口。

### ArrayList 与 LinkedList 区别?

- **是否保证线程安全：** `ArrayList` 和 `LinkedList` 都是不同步的，也就是不保证线程安全；
- **底层数据结构：** `ArrayList` 底层使用的是 **`Object` 数组**；`LinkedList` 底层使用的是 **双向链表** 数据结构（JDK1.6 之前为循环链表，JDK1.7 取消了循环。注意双向链表和双向循环链表的区别，下面有介绍到！）
- **插入和删除是否受元素位置的影响：**
    - `ArrayList` 采用数组存储，所以插入和删除元素的时间复杂度受元素位置的影响。 比如：执行`add(E e)`方法的时候， `ArrayList` 会默认在将指定的元素追加到此列表的末尾，这种情况时间复杂度就是 O(1)。但是如果要在指定位置 i 插入和删除元素的话（`add(int index, E element)`），时间复杂度就为 O(n)。因为在进行上述操作的时候集合中第 i 和第 i 个元素之后的(n-i)个元素都要执行向后位/向前移一位的操作。
    - `LinkedList` 采用链表存储，所以在头尾插入或者删除元素不受元素位置的影响（`add(E e)`、`addFirst(E e)`、`addLast(E e)`、`removeFirst()`、 `removeLast()`），时间复杂度为 O(1)，如果是要在指定位置 `i` 插入和删除元素的话（`add(int index, E element)`，`remove(Object o)`,`remove(int index)`）， 时间复杂度为 O(n) ，因为需要先移动到指定位置再插入和删除。
- **是否支持快速随机访问：** `LinkedList` 不支持高效的随机元素访问，而 `ArrayList`（实现了 `RandomAccess` 接口） 支持。快速随机访问就是通过元素的序号快速获取元素对象(对应于`get(int index)`方法)。
- **内存空间占用：** `ArrayList` 的空间浪费主要体现在在 list 列表的结尾会预留一定的容量空间，而 LinkedList 的空间花费则体现在它的每一个元素都需要消耗比 ArrayList 更多的空间（因为要存放直接后继和直接前驱以及数据）。

我们在项目中一般是不会使用到 `LinkedList` 的，需要用到 `LinkedList` 的场景几乎都可以使用 `ArrayList` 来代替，并且，性能通常会更好！就连 `LinkedList` 的作者约书亚 · 布洛克（Josh Bloch）自己都说从来不会使用 `LinkedList` 。

![](https://oss.javaguide.cn/github/javaguide/redisimage-20220412110853807.png)

另外，不要下意识地认为 `LinkedList` 作为链表就最适合元素增删的场景。我在上面也说了，`LinkedList` 仅仅在头尾插入或者删除元素的时候时间复杂度近似 O(1)，其他情况增删元素的平均时间复杂度都是 O(n) 。


# 常见
### volatile 和transient

`volatile` 和 `transient` 都是 Java 中用来修饰字段（成员变量）的关键字，用于不同的用途。

1. **volatile**: `volatile` 关键字用于声明一个字段为“易变的”（volatile），这意味着对该字段的读写操作都会直接在主内存中进行，而不会被缓存在线程的本地内存中。这样可以确保多个线程能够正确地看到该字段的最新值，从而避免了由于线程间的不同步而引发的问题，比如可见性问题和指令重排序问题。通常用于标识共享变量，以确保正确的多线程访问。
	 **禁止指令重排序（Preventing Instruction Reordering）**：编译器和处理器在优化时会进行指令重排序，这可能导致线程间操作的顺序不同于代码的原始顺序。使用 `volatile` 关键字可以防止编译器和处理器对被标记变量操作的重排序，从而确保操作按照代码的顺序执行。
```java
public class VolatileExample {
    private volatile int count = 0;
    
    public void increment() {
        count++;
    }
    
    public int getCount() {
        return count;
    }
}

```
2. **transient**: `transient` 关键字用于声明一个字段不会被默认的序列化过程所包括。当一个对象被序列化（比如写入到文件或通过网络传输），只有标记为 `transient` 的字段不会被序列化，而其他非 `transient` 字段会被保存和恢复。这在一些场景下很有用，比如对象中包含了敏感信息或者不需要序列化的临时计算结果。
```java
public class Employee implements Serializable {
    private String name;
    private transient String confidentialInfo;
    
    // ...
}
```

要注意的是，`volatile` 和 `transient` 解决的问题领域不同。`volatile` 解决的是多线程间共享变量的可见性和有序性问题，而 `transient` 解决的是对象序列化过程中的字段选择问题。虽然它们在关键字上有一些相似之处，但用途不同，不能互相替代。

