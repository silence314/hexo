---
title: Java集合
date: 2020-09-07 17:19:16
tags: 
 - Java 
 - 集合
typora-root-url: ../../themes/butterfly/source
description: Java集合相关总结
cover: /blogImg/Collection.jpeg
categories: Java基础
---

# java集合简介

Java集合大致可以分为Set、List、Queue和Map四种体系，其中Set代表无序、不可重复的集合；List代表有序、重复的集合；而Map则代表具有映射关系的集合，Java 5 又增加了Queue体系集合，代表一种队列集合实现。
 Java集合就像一种容器，可以把多个对象（实际上是对象的引用，但习惯上都称对象）“丢进”该容器中。从Java 5 增加了泛型以后，Java集合可以记住容器中对象的数据类型，使得编码更加简洁、健壮。

## java集合和数组的区别

1. 数组长度在初始化时指定，意味着只能保存定长的数据。而集合可以保存数量不确定的数据。同时可以保存具有映射关系的数据（即关联数组，键值对 key-value）。
2. 数组元素即可以是基本类型的值，也可以是对象。集合里只能保存对象（实际上只是保存对象的引用变量），基本数据类型的变量要转换成对应的包装类才能放入集合类中。

## 两大基类Collection与Map

![java集合](/blogImg/Collection.jpeg)

在集合框架的类继承体系中，最顶层有两个接口：

- `Collection`表示一组纯数据
- `Map`表示一组key-value对



主要有三个接口：

- `Set`表示不允许有重复元素的集合
- `List`表示允许有重复元素的集合
- `Queue` JDK1.5新增，与上面两个集合类主要是的区分在于`Queue`主要用于存储数据，而不是处理数据。



Map并不是一个真正意义上的集合，但是这个接口提供了三种“集合视角”，使得可以像操作集合一样操作它们，具体如下：

- 把map的内容看作key的集合
- 把map的内容看作value的集合
- 把map的内容看作key-value映射的集合

# ArrayList

`ArrayList`是`Vector`的翻版，只是去除了线程安全。`Vector`因为种种原因不推荐使用了，这里我们就不对其进行分析了。`ArrayList`是一个可以动态调整大小的`List`实现，其数据的顺序与插入顺序始终一致，其余特性与`List`中定义的一致。

## ArrayList的继承结构

可以看到，`ArrayList`是`AbstractList`的子类，同时实现了`List`接口。除此之外，它还实现了三个标识型接口，这几个接口都没有任何方法，仅作为标识表示实现类具备某项功能。`RandomAccess`表示实现类支持快速随机访问，`Cloneable`表示实现类支持克隆，具体表现为重写了`clone`方法，`java.io.Serializable`则表示支持序列化，如果需要对此过程自定义，可以重写`writeObject`与`readObject`方法。

`ArrayList`可以动态调整大小，所以我们才可以无感知的插入多条数据，其默认大小是10。而要想扩充数组的大小，只能通过复制。这样一来，默认大小以及如何动态调整大小会对使用性能产生非常大的影响。我们举个例子来说明此情形：

比如默认大小为10，我们向`ArrayList`中插入5条数据，并不会涉及到扩容。如果想插入100条数据，就需要将`ArrayList`大小调整到100再进行插入，这就涉及一次数组的复制。如果此时，还想再插入50条数据呢？那就得把大小再调整到150，把原有的100条数据复制过来，再插入新的50条数据。自此之后，我们每向其中插入一条数据，都要涉及一次数据拷贝，且数据量越大，需要拷贝的数据越多，性能也会迅速下降。

## 构造方法与初始化

`ArrayList`一共有三个构造方法，用到了两个成员变量。

```java
//这是一个用来标记存储容量的数组，也是存放实际数据的数组。
//当ArrayList扩容时，其capacity就是这个数组应有的长度。
//默认时为空，添加进第一个元素后，就会直接扩展到DEFAULT_CAPACITY，也就是10
//这里和size区别在于，ArrayList扩容并不是需要多少就扩展多少的
//ArrayList 实现了 Serializable 接口，这意味着 ArrayList 支持序列化。transient 的作用是说不希望 elementData 数组被序列化，重写了 writeObject实现,每次序列化时，先调用 defaultWriteObject() 方法序列化 ArrayList 中的非 transient 元素，然后遍历 elementData，只序列化已存入的元素，这样既加快了序列化的速度，又减小了序列化之后的文件大小。
transient Object[] elementData;

//这里就是实际存储的数据个数了
private int size;
```

下面我们看看构造函数

```java
//默认构造方法。文档说明其默认大小为10，但正如elementData定义所言，
//只有插入一条数据后才会扩展为10，而实际上默认是空的
 public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

//带初始大小的构造方法，一旦指定了大小，elementData就不再是原来的机制了。
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
    }
}

//从一个其他的Collection中构造一个具有初始化数据的ArrayList。
//这里可以看到size是表示存储数据的数量
//这也展示了Collection这种抽象的魅力，可以在不同的结构间转换
public ArrayList(Collection<? extends E> c) {
    //转换最主要的是toArray()，这在Collection中就定义了
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

## 重要方法

### add

```java
//在最后添加一个元素
public boolean add(E e) {
    //先确保elementData数组的长度足够
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}

public void add(int index, E element) {
    rangeCheckForAdd(index);

    //先确保elementData数组的长度足够
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //将数据向后移动一位，空出位置之后再插入
    System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
    elementData[index] = element;
    size++;
}
```

以上两种添加数据的方式都调用到了`ensureCapacityInternal`这个方法，我们看看它是如何完成工作的：

```java
//在定义elementData时就提过，插入第一个数据就直接将其扩充至10
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    
    //这里把工作又交了出去
    ensureExplicitCapacity(minCapacity);
}

//如果elementData的长度不能满足需求，就需要扩充了
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

//扩充
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    //可以看到这里是1.5倍扩充的
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    
    //扩充完之后，还是没满足，这时候就直接扩充到minCapacity
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    //防止溢出
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

至此，我们彻底明白了`ArrayList`的扩容机制了。首先创建一个空数组**elementData**，第一次插入数据时直接扩充至10，然后如果**elementData**的长度不足，就扩充1.5倍，如果扩充完还不够，就使用需要的长度作为**elementData**的长度。

这样的方式显然比我们例子中好一些，但是在遇到大量数据时还是会频繁的拷贝数据。那么如何缓解这种问题呢，`ArrayList`为我们提供了两种可行的方案：

- 使用`ArrayList(int initialCapacity)`这个有参构造，在创建时就声明一个较大的大小，这样解决了频繁拷贝问题，但是需要我们提前预知数据的数量级，也会一直占有较大的内存。
- 除了添加数据时可以自动扩容外，我们还可以在插入前先进行一次扩容。只要提前预知数据的数量级，就可以在需要时直接一次扩充到位，与`ArrayList(int initialCapacity)`相比的好处在于不必一直占有较大内存，同时数据拷贝的次数也大大减少了。这个方法就是ensureCapacity(int minCapacity)，其内部就是调用了`ensureCapacityInternal(int minCapacity)`。

# ArrayDeque

这个集合可能很少用到，不过他有一个和hashMap类似的对于容量的处理。

## 构造函数与重要成员变量

```java
//存放元素，长度和capacity一致，并且总是2的次幂
//这一点，我们放在后面解释
transient Object[] elements; 

//capacity最小值，也是2的次幂
private static final int MIN_INITIAL_CAPACITY = 8;

//标记队首元素所在的位置
transient int head;

//标记队尾元素所在的位置
transient int tail;
```

```java
//默认构造函数，将elements长度设为16，相当于最小capacity的两倍
public ArrayDeque() {
    elements = new Object[16];
}

//带初始大小的构造
public ArrayDeque(int numElements) {
    allocateElements(numElements);
}

//从其他集合类导入初始数据
public ArrayDeque(Collection<? extends E> c) {
    allocateElements(c.size());
    addAll(c);
}
```

这里看到有两个构造函数都用到了`allocateElements`方法，这是一个非常经典的方法，我们接下来就先重点研究它。

**寻找最近的两次幂**

在定义`elements`变量时说，其长度总是2的次幂，但用户传入的参数并不一定符合规则，所以就需要根据用户的输入，找到比它大的最近的2次幂。比如用户输入13，就把它调整为16，输入31，就调整为32，等等。考虑下，我们有什么方法可以实现呢？

来看下`ArrayDeque`是怎么做的吧：

```java
private void allocateElements(int numElements) {
    int initialCapacity = MIN_INITIAL_CAPACITY;
    // Find the best power of two to hold elements.
    // Tests "<=" because arrays aren't kept full.
    if (numElements >= initialCapacity) {
        initialCapacity = numElements;
        initialCapacity |= (initialCapacity >>>  1);
        initialCapacity |= (initialCapacity >>>  2);
        initialCapacity |= (initialCapacity >>>  4);
        initialCapacity |= (initialCapacity >>>  8);
        initialCapacity |= (initialCapacity >>> 16);
        initialCapacity++;

        if (initialCapacity < 0)   // Too many elements, must back off
            initialCapacity >>>= 1;// Good luck allocating 2 ^ 30 elements
    }
    elements = new Object[initialCapacity];
}
```

看到这段迷之代码了吗？在`HashMap`中也有一段类似的实现。但要读懂它，我们需要先掌握以下几个概念：

- 在java中，int的长度是32位，有符号int可以表示的值范围是 (-2)<sup>31</sup> 到 2<sup>31</sup>-1，其中最高位是符号位，0表示正数，1表示负数。
- `>>>`：无符号右移，忽略符号位，空位都以0补齐。
- `|`：位或运算，按位进行或操作，逢1为1。

我们知道，计算机存储任何数据都是采用二进制形式，所以一个int值为80的数在内存中可能是这样的：

> 0000 0000 0000 0000 0000 0000 0101 0000

比80大的最近的2次幂是128，其值是这样的：

> 0000 0000 0000 0000 0000 0000 1000 0000

我们多找几组数据就可以发现规律：

- 每个2的次幂用二进制表示时，只有一位为 1，其余位均为 0（不包含符合位）
- 要找到比一个数大的2的次幂（在正数范围内），只需要将其最高位左移一位（从左往右第一个 1 出现的位置），其余位置 0 即可。

但从实践上讲，没有可行的方法能够进行以上操作，即使通过`&`操作符可以将某一位置 0 或置 1，也无法确认最高位出现的位置，也就是基于最高位进行操作不可行。

但还有一个很整齐的数字可以被我们利用，那就是 2n-1，我们看下128-1=127的表示形式：

> 0000 0000 0000 0000 0000 0000 0111 1111

把它和80对比一下：

> 0000 0000 0000 0000 0000 0000 0101 0000 //80
> 0000 0000 0000 0000 0000 0000 0111 1111  //127

可以发现，我们只要把80从最高位起每一位全置为1，就可以得到离它最近且比它大的 2n-1，最后再执行一次+1操作即可。具体操作步骤为（为了演示，这里使用了很大的数字）：
 原值：

> 0011 0000 0000 0000 0000 0000 0000 0010

1. 无符号右移1位

> 0001 1000 0000 0000 0000 0000 0000 0001

1. 与原值`|`操作：

> 0011 1000 0000 0000 0000 0000 0000 0011

可以看到最高2位都是1了，也仅能保证前两位为1，这时就可以直接移动两位

1. 无符号右移2位

> 0000 1110 0000 0000 0000 0000 0000 0000

1. 与原值`|`操作：

> 0011 1110 0000 0000 0000 0000 0000 0011

此时就可以保证前4位为1了，下一步移动4位

1. 无符号右移4位

> 0000 0011 1110 0000 0000 0000 0000 0000

1. 与原值`|`操作：

> 0011 1111 1110 0000 0000 0000 0000 0011

此时就可以保证前8位为1了，下一步移动8位

1. 无符号右移8位

> 0000 0000 0011 1111 1110 0000 0000 0000

1. 与原值`|`操作：

> 0011 1111 1111 1111 1110 0000 0000 0011

此时前16位都是1，只需要再移位操作一次，即可把32位都置为1了。

1. 无符号右移16位

> 0000 0000 0000 0000 0011 1111 1111 1111

1. 与原值`|`操作：

> 0011 1111 1111 1111 1111 1111 1111 1111

1. 进行+1操作：

> 0100 0000 0000 0000 0000 0000 0000 0000

如此经过11步操作后，我们终于找到了合适的2次幂。写成代码就是：

```ruby
    initialCapacity |= (initialCapacity >>>  1);
    initialCapacity |= (initialCapacity >>>  2);
    initialCapacity |= (initialCapacity >>>  4);
    initialCapacity |= (initialCapacity >>>  8);
    initialCapacity |= (initialCapacity >>> 16);
    initialCapacity++;
```

不过为了防止溢出，导致出现负值（如果把符号位置为1，就为负值了）还需要一次校验：

```cpp
if (initialCapacity < 0)   // Too many elements, must back off
     initialCapacity >>>= 1;// Good luck allocating 2 ^ 30 elements
```

至此，初始化的过程就完毕了。

## 重要操作方法

### add方法

```java
//在队首添加一个元素，非空
public void addFirst(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[head = (head - 1) & (elements.length - 1)] = e;
    if (head == tail)
        doubleCapacity();
}

//在队尾添加一个元素，非空
public void addLast(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[tail] = e;
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)
        doubleCapacity();
}
```

这里，又有一段迷之代码需要我们认真研究了，这也是`ArrayDeque`值得我们研究的地方之一，通过位运算提升效率。

```java
elements[head = (head - 1) & (elements.length - 1)] = e;
```

很明显这是一个赋值操作，而且应该是给head之前的位置赋值，所以`head = (head - 1)`是合理的操作，那这个`& (elements.length - 1)`又表示什么呢？

在之前的定义与初始化中，`elements.length`要求为2的次幂，也就是 2n 形式，那这个`& (elements.length - 1)`也就是 2n-1 了，在内存中用二进制表示就是从最高位起每一位都是1。我们还以之前的127为例：

> 0000 0000 0000 0000 0000 0000 0111 1111

`&`就是按位与，全1才为1。那么任意一个正数和127进行按位与操作后，都只有最右侧7位被保留了下来，其他位全部置0（除符号位），而对一个负数而言，则会把它的符号位置为0，`&`操作后会变成正数。比如-1的值是1111 ... 1111（32个1），和127按位操作后结果就变成了127 。所以，对于正数它就是取模，对于负数，它就是把元素插入了数组的结尾。所以，这个数组并不是向前添加元素就向前扩展，向后添加就向后扩展，它是循环的，类似这样：

<img src="/blogImg/循环队列示意图.jpg" alt="循环队列示意图" style="zoom:50%;" />

循环队列示意图

初始时，head与tail都指向a[0]，这时候数组是空的。当执行`addFirst()`方法时，head指针移动一位，指向a[elements.length-1]，并赋值，也就是给a[elements.length-1]赋值。当执行`addLast()`操作时，先给a[0]赋值，再将tail指针移动一位，指向a[1]。所以执行完之后head指针位置是有值的，而tail位置是没有值的。

随着添加操作执行，数组总会占满，那么怎么判断它满了然后扩容呢？首先，如果head==tail，则说明数组是空的，所以在添加元素时必须保证head与tail不相等。假如现在只有一个位置可以添加元素了，类似下图：

<img src="/blogImg/循环队列即将填满.jpg" alt="循环队列即将填满" style="zoom:50%;" />

循环队列即将充满示意图

此时，tail指向了a[8]，head已经填充到a[9]了，只有a[8]是空闲的。很显然，不管是`addFirst`还是`addLast`，再添加一个元素后都会导致head == tail。这时候就不得不扩容了，因为head == tail是判断是否为空的条件。扩容就比较简单了，直接翻倍，我们看代码：

```java
private void doubleCapacity() {
    //只有head==tail时才可以扩容
    assert head == tail;
    int p = head;
    int n = elements.length;
    //在head之后，还有多少元素
    int r = n - p; // number of elements to the right of p
    //直接翻倍，因为capacity初始化时就已经是2的倍数了，这里无需再考虑
    int newCapacity = n << 1;
    if (newCapacity < 0)
        throw new IllegalStateException("Sorry, deque too big");
    Object[] a = new Object[newCapacity];
    //左侧数据拷贝
    System.arraycopy(elements, p, a, 0, r);
    //右侧数据拷贝
    System.arraycopy(elements, 0, a, r, p);
    elements = a;
    head = 0;
    tail = n;
}
```

## 总结

`ArrayDeque`通过循环数组的方式实现的循环队列，并通过位运算来提高效率，容量大小始终是2的次幂。当数据充满数组时，它的容量将翻倍。作为`Stack`，因为其非线程安全所以效率高于`java.util.Stack`，而作为队列，因为其不需要结点支持所以更快（LinkedList使用Node存储数据，这个对象频繁的new与clean，使得其效率略低于ArrayDeque）。但队列更多的用来处理多线程问题，所以我们更多的使用`BlockingQueue`，关于多线程的问题，以后再认真研究。

# HashMap

`HashMap`可能是我们使用最多的键值对型的集合类了，它的底层基于哈希表，采用数组存储数据，使用链表来解决哈希碰撞。在JDK1.8中还引入了红黑树来解决链表长度过长导致的查询速度下降问题。

JDK1.8主要解决或优化了一下问题：

1. resize 扩容优化
2. 引入了红黑树，目的是避免单条链表过长而影响查询效率
3. 解决了多线程死循环问题，但仍是非线程安全的，多线程时可能会造成数据丢失问题。

| 不同                     | JDK 1.7                                                      | JDK 1.8                                                      |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 存储结构                 | 数组 + 链表                                                  | 数组 + 链表 + 红黑树                                         |
| 初始化方式               | 单独函数：`inflateTable()`                                   | 直接集成到了扩容函数`resize()`中                             |
| hash值计算方式           | 扰动处理 = 9次扰动 = 4次位运算 + 5次异或运算                 | 扰动处理 = 2次扰动 = 1次位运算 + 1次异或运算                 |
| 存放数据的规则           | 无冲突时，存放数组；冲突时，存放链表                         | 无冲突时，存放数组；冲突 & 链表长度 < 8：存放单链表；冲突 & 链表长度 > 8：树化并存放红黑树 |
| 插入数据方式             | 头插法（先讲原位置的数据移到后1位，再插入数据到该位置）      | 尾插法（直接插入到链表尾部/红黑树）                          |
| 扩容后存储位置的计算方式 | 全部按照原来方法进行计算（即hashCode ->> 扰动函数 ->> (h&length-1)） | 按照扩容后的规律计算（即扩容后的位置=原位置 or 原位置 + 旧容量） |

## 构造函数与成员变量

在看构造函数和成员变量前，我们要先看下其数据单元，因为`HashMap`有普通的元素，还有红黑树的元素，所以其数据单元定义有两个：

```java
// 普通节点
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
    // ...
}

// 树节点，继承自LinkedHashMap.Entry
// 这是因为LinkedHashMap是HashMap的子类，也需要支持树化
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    // ...
}

// LinkedHashMap.Entry的实现
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

成员变量：

```java
// capacity初始值，为16，必须为2的次幂
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

// capacity的最大值，为2^30
static final int MAXIMUM_CAPACITY = 1 << 30;

// load factor，是指当容量被占满0.75时就需要rehash扩容
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 链表长度到8，就转为红黑树
static final int TREEIFY_THRESHOLD = 8;

// 树大小为6，就转回链表
static final int UNTREEIFY_THRESHOLD = 6;

// 至少容量到64后，才可以转为树
static final int MIN_TREEIFY_CAPACITY = 64;

// 保存所有元素的table表
transient Node<K,V>[] table;

// 通过entrySet变量，提供遍历的功能
transient Set<Map.Entry<K,V>> entrySet;

// 下一次扩容值
int threshold;

// load factor
final float loadFactor;
```

构造函数：

```java
 public HashMap(int initialCapacity, float loadFactor) {
    // ... 参数校验    
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
}

public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

这个构造函数和其他的也没什么差别，这里我们还要看下`tableSizeFor`这个方法：

```java
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

这个和上面的ArrayDeque的扩容方法是一样的。

## 重要方法

### put方法

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

这里用到的方法很简单，就是把key与其高16位异或。因为没有完美的哈希算法可以彻底避免碰撞，所以只能尽可能减少碰撞，在各方面权衡之后得到一个折中方案。

```java
// 参数onlyIfAbsent表示是否替换原值
// 参数evict我们可以忽略它，它主要用来区别通过put添加还是创建时初始化数据的
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 空表，需要初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        // resize()不仅用来调整大小，还用来进行初始化配置
        n = (tab = resize()).length;
    // (n - 1) & hash这种方式也熟悉了吧？都在分析ArrayDeque中有体现
    //这里就是看下在hash位置有没有元素，实际位置是hash % (length-1)
    if ((p = tab[i = (n - 1) & hash]) == null)
        // 将元素直接插进去
        tab[i] = newNode(hash, key, value, null);
    else {
        //这时就需要链表或红黑树了
        // e是用来查看是不是待插入的元素已经有了，有就替换
        Node<K,V> e; K k;
        // p是存储在当前位置的元素
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p; //要插入的元素就是p，这说明目的是修改值
        // p是一个树节点
        else if (p instanceof TreeNode)
            // 把节点添加到树中
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 这时候就是链表结构了，要把待插入元素挂在链尾
            for (int binCount = 0; ; ++binCount) {
                //向后循环
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 链表比较长，需要树化，
                    // 由于初始即为p.next，所以当插入第9个元素才会树化
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 找到了对应元素，就可以停止了
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                // 继续向后
                p = e;
            }
        }
        // e就是被替换出来的元素，这时候就是修改元素值
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 默认为空实现，允许我们修改完成后做一些操作
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // size太大，达到了capacity的0.75，需要扩容
    if (++size > threshold)
        resize();
    // 默认也是空实现，允许我们插入完成后做一些操作
    afterNodeInsertion(evict);
    return null;
}
```

![putVal大致流程](/blogImg/hashmapPutVal.png)

以上方法和我们开头看到的文档描述一致，在插入时可能会从链表变成红黑树。里面用到了`TreeNode.putTreeVal`方法向红黑树中插入元素，关于`TreeNode`的方法我们最后分析。除此之外，还有一个树化的方法是`treeifyBin`，我们现在看下其原理：

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //如果表是空表，或者长度还不到树化的最小值，就需要重新调整表了
    // 这样做是为了防止最初就进行树化
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        // while循环的目的是把链表的每个节点转为TreeNode
        do {
            // 根据当前元素，生成一个对应的TreeNode节点
            TreeNode<K,V> p = replacementTreeNode(e, null);
            //挂在红黑树的尾部，顺序和链表一致
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            // 这里也用到了TreeNode的方法，我们在最后一起分析
            // 通过头节点调节TreeNode
            // 链表数据的顺序是不符合红黑树的，所以需要调整
            hd.treeify(tab);
    }
}
```

无论是在`put`还是`treeify`时，都依赖于`resize`，它的重要性不言而喻。它不仅可以调整大小，还能调整树化和反树化（从树变为链表）所带来的影响。我们看看它具体做了哪些工作：

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 大小超过了2^30
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 扩容，扩充为原来的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // 原来的threshold设置了
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        // 全部设为默认值
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
     // 扩容完成，现在需要进行数据拷贝，从原表复制到新表
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    // 这是只有一个值的情况
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 重新规划树，如果树的size很小，默认为6，就退化为链表
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 处理链表的数据
                    // loXXX指的是在原表中出现的位置
                    Node<K,V> loHead = null, loTail = null;
                    // hiXXX指的是在原表中不包含的位置
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        //这里把hash值与oldCap按位与。
                        //oldCap是2的次幂，所以除了最高位为1以外其他位都是0
                        // 和它按位与的结果为0，说明hash比它小，原表有这个位置
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 挂在原表相应位置
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 挂在后边
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

当put时，如果发现目前的bucket占用程度已经超过了Load Factor所希望的比例，那么就会发生resize。在resize的过程，简单的说就是把bucket扩充为2倍，之后重新计算index，把节点再放到新的bucket中。resize的注释是这样描述的：

> Initializes or doubles table size. If null, allocates in accord with initial capacity target held in field threshold. Otherwise, because we are using power-of-two expansion, the elements from each bin must either **stay at same index**, or **move with a power of two offset** in the new table.

大致意思就是说，当超过限制的时候会resize，然而又因为我们使用的是2次幂的扩展(指长度扩为原来2倍)，所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。

怎么理解呢？例如我们从16扩展为32时，具体的变化如下所示：
![16扩展为32](/blogImg/16扩展为32.png)

因此元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：
![16rehash32](/blogImg/16rehash32.png)

因此，我们在扩充HashMap的时候，不需要重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”。可以看看下图为16扩充为32的resize示意图：
![](/blogImg/图示.png)

==头插法还是尾插法==：1.8之前是头插法，1.8之后是尾插法。

1.8之前因为resize的赋值方式，使用了**单链表的头插入方式，同一位置上新元素总会被放在链表的头部位置**，在旧数组中同一条Entry链上的元素，通过重新计算索引位置后，有可能被放到了新数组的不同位置上。**使用头插会改变链表的上的顺序，但是如果使用尾插，在扩容时会保持链表元素原本的顺序，就不会出现链表成环的问题了**



### 删除元素

```java
// matchValue是说只有value值相等时候才可以删除，我们是按照key删除的，所以可以忽略它。
// movable是指是否允许移动其他元素，这里是和TreeNode相关的
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        // 不同情况下获取待删除的node节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                            (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                                (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                // TreeNode删除
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                // 在队首，直接删除
                tab[index] = node.next;
            else
                // 链表中删除
                p.next = node.next;
            ++modCount;
            --size;
            // 默认空实现，允许我们删除节点后做些处理
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

