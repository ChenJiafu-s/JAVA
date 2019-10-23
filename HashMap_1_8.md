

[TOC]



# HashMap在JDK1.8中的实现

参考博文：https://www.jianshu.com/p/ee0de4c99f87

## HashMap源码分析

### 静态变量

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;    //16,默认初始容量必须是2的整数幂
static final int MAXIMUM_CAPACITY = 1 << 30;    //1073741824，最大容量
static final float DEFAULT_LOAD_FACTOR = 0.75f;    //默认装载因子

//以下为1.8新增变量
static final int TREEIFY_THRESHOLD = 8;    //当一个槽中的链表长度大于这个值时，将链表转化为红黑树
static final int UNTREEIFY_THRESHOLD = 6;    //当一个槽中树的节点数数小于这个值时，转化为链表
static final int MIN_TREEIFY_CAPACITY = 64;    //
```

### 成员变量

```java
transient Node<K,V>[] table;	//实际存储key、value的数组，被封装成了Node
transient Set<Map.Entry<K,V>> entrySet;
transient int size;		//当前HashMap中Node的数量
transient int modCount;
int threshold;
final float loadFactor;
```



### 构造函数

#### HashMap()

```java
// 无参构造函数
// 构造一个空的HashMap，初始容量为16，负载因子为0.75
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR;    //其他变量使用默认值
}
```

#### HashMap(int initialCapacity)

```java
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```

#### HashMap(int initialCapacity, float loadFactor)

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
```

当初始容量小于0时，抛出**IllegalArgumentException**异常；当初始容量大于最大容量时，就让初始容量等于最大容量。

当负载因子小于0或者不是数字时，抛出**IllegalArgumentException**异常。

实际上**threshold**的值就等于capacity * loadFactor，当HashMap的size到了**threshold**时，就要进行resize，也就是扩容。

tableSizeFor()的功能就是返回一个比给定整数大的最小的2的幂次方整数，例如给出10，则返回2的4次方，也就是16。

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

**note：**HashMap要求容量必须是2的幂

首先，`int n = cap - 1`是为了防止cap已经是2的幂时，执行完后面的几条转化后，返回的capacity时当前cap的2倍。然后一系列位操作，操作的结果就是将一个32位的整数，从高位往低位寻找第一个1，并将这个1以后的所有低位都变成1。最后返回结果，并赋值给threshold。

#### HashMap(Map<? extends K, ? extends V> m)

```java
//构造一个和指定Map有相同mappings的HashMap，初始容量能重组的容下指定的Map，负载因子位0.75
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

putMapEntries()函数源码如下：

```java
/**
 * 将m中的所有元素存入当前HashMap实例中
 */
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        //判断table是否已经初始化，若未初始化，则先初始化一些变量。
        if (table == null) { // pre-size
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        else if (s > threshold)
            resize();	//若待插入的map的size大于threshold，则进行resize()扩容
        //最后，遍历map，将所有的键值对插入到当前Hashmap实例中
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

### 重要方法

#### hash(key)

```java
/**
 * 将key的hashCode的高16位与低16位进行异或运算，间接的在低16位保留了高16位的信息，用于散列
 */
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

#### resize()

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;	// 保存当前table
    int oldCap = (oldTab == null) ? 0 : oldTab.length;	// 保存当前table的容量
    int oldThr = threshold;	// 保存当前table的阈值
    int newCap, newThr = 0;	// 初始化新的table的容量与阈值
    // 当原table非空时
    if (oldCap > 0) {
        // 若旧table容量已超过最大容量，更新阈值为Integer.MAX_VALUE（最大整形值），以后就不会自动扩容了。
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 容量变为两倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            //阈值变为两倍
            newThr = oldThr << 1; // double threshold
    }
    // 当原table为空时，但oldThr大于0，代表用户创建了一个HashMap，使用了后三个带参构造函数，但从未插入元素，导致oldTab为null，oldCap为0，oldThr为用户指定的HashMap的初始容量。
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    // 用户调用第一个无参狗赞函数创建的，此时所有值均采用默认值，oldTab数组为空，oldCap为0，oldThr为0
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 当新的阈值为0的时候
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    // 初始化新的table
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 当原table不为空时，将原来的节点，重新放到新的table中去
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 若节点为单个节点，直接在新table中进行定位
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 若节点为TreeNode节点，则进行红黑树的rehash操作
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 若时链表，则进行链表的rehash操作
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    // 将同一桶中的元素根据(e.hash & oldCap)是否为0进行分割
                    do {
                        next = e.next;
                        // 根据算法　e.hash & oldCap 判断节点位置rehash　后是否发生改变
                        // 最高位==0，这是索引不变的链表。
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        //最高位==1，这是索引发生改变的链表。
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {	// 原bucket位置的尾指针不为空(即还有node)  
                        loTail.next = null;	// 链表最后得有个null
                        newTab[j] = loHead;	// 链表头指针放在新桶的相同下标(j)处
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        // rehash后，节点新的位置一定为原来基础上加oldCap
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

resize()使用的是2次幂的扩展，每次扩展后的空间为原空间的两倍，所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。我们在扩充HashMap的时候，只需要看原来的hash值新增的哪个位是1还是0就好了，是0的话索引不变，是1的话，索引变成原索引+源容量的新索引。

**什么时候扩容：**在put操作时，当前容器中元素的个数达到了阈值，就会扩容。

**扩容(resize)：**其实就是重新计算容量(扩大一倍原容量)，并将原来容器中的元素放入其中。

#### putVal(int hash, K key, V value, boolean onlyIfAbsent,                   boolean evict)

```java
 //实现put和相关方法。
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //如果table为空或者长度为0，则resize()
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //确定插入table的位置，算法是(n - 1) & hash，在n为2的幂时，相当于取摸操作。
    ////找到key值对应的槽并且是第一个，直接加入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    //在table的i位置发生碰撞，有两种情况，1、key值是一样的，替换value值，
    //2、key值不一样的有两种处理方式：2.1、存储在i位置的链表；2.2、存储在红黑树中
    else {
        Node<K,V> e; K k;
        //第一个node的hash值即为要加入元素的hash
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //2.2
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        //2.1
        else {
            //不是TreeNode,即为链表,遍历链表
            for (int binCount = 0; ; ++binCount) {
                ///链表的尾端也没有找到key值相同的节点，则生成一个新的Node,
                //并且判断链表的节点个数是不是到达转换成红黑树的上界达到，则转换成红黑树。
                if ((e = p.next) == null) {
                    // 创建链表节点并插入尾部
                    p.next = newNode(hash, key, value, null);
                    ////超过了链表的设置长度8就转换成红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //如果e不为空就替换旧的oldValue值
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

hash冲突的集中情况：

1. 两节点key值相同(hash值一定相同)，导致冲突。

2. 两节点key值不同，但hash出来的hash值相同，导致冲突。

3. 两节点key值不同，hash值也不同，但hash值对数组长度取模后相同，导致冲突。

---

## 1.7和1.8的HashMap的不同点

1. JDK1.7用的是头插法，而JDK1.8及之后使用的是尾插法。因为JDK1.7是用单链表进行纵向延伸，使用头插法能够提高插入效率，但是也会容易出现逆序且唤醒链表死循环问题(多线程resize时出现逆序，之后get陷入死循环)。而JDK1.8之后是因为加入了红黑树使用尾插法，能够避免出现逆序且链表死循环的问题。
2. 扩容后数据存储位置的计算方式也不一样：
   - 在JDK1.7中是直接用hash值和需要扩容的二进制数进行&。
   - 在JDK1.8中是直接用了JDK1.7的时候计算的规律，也就是扩容前的原始位置+扩容的大小=JDK1.8的计算方式。这种方式只需要判断hash值新增参与运算的位是0还是1就直接迅速计算出了扩容后的存储方式。
3. JDK1.7的时候使用的数组+单链表的数据结构。但是在JDK1.8及之后时，使用的是数组+链表+红黑树的数据结构(当链表长度达到8时，转化红黑树；当红黑树的节点数小于等于6时，转化为链表)。

---

## HashMap为什么是线程不安全的？

1. put的时候导致的多线程数据不一致：比如有两个线程A和B，首先A希望插入一个key-value对到HashMap中，首先计算记录所要落到hash桶的索引坐标，然后获取到该桶里面的链表头节点，此时线程A的时间片用完了，而此时线程B被调度得以执行，和线程A一样执行，只不过线程B成功将记录茶道了桶里面，假设线程A插入的计算计算出来的hash桶索引和线程B要插入的记录计算出来的hash桶索引是一样的，那么当线程B成功插入后，线程A再次被调度运行时，它依然持有过期的链表头，但是它并不知道链表头已经更新了，以至于当线程A插入记录后，就覆盖掉了线程B所插入的记录，造成了数据不一致的行为，因此，线程不安全。
2. resize引起的死循环：这种情况发成在HashMap自动扩容时，当2个线程同时检测到元素个数超过数组大小✖负载因子。此时2个线程会在put()方法中调用resize()，两个线程同时修改一个链表结构会产生一个循环链表(JDK1.7中，会出现resize前后元素循序导致的情况)。接下来再向通过get()获取到某一个元素时，就会出现死循环。

---

## HashMap和HashTable的区别

HashMap和HashTable都实现了Map接口，但决定用哪一个之前要先弄清楚它们之间的区别。主要的区别有：线程安全性，同步(synchronization)，以及速度。

1. HashMap可以接受null作为key和value。HashTable不接受null作为key或者value。
2. HashMap是非synchronized的，线程不安全。HashTable是synchronized的，线程安全。
3. 由于第二点，在单线程中，HashMap的性能比HashTable要好。
4. HashMap的迭代器(Iterator)是fail-fast迭代器，而HashTable的迭代器(enumerator)不是fail-fast的。所以当其他贤臣给改变了HashMap的结构(增加或者移除元素)，将会抛出ConcurrentModificationException，但迭代器本身的remove()方法一处元素则不会抛出该异常。但这并不是一个一定发生的行为，要看JVM。
5. HashMap不能保证随着时间的推移Map中的元素次序是不变的。

---

参考博文：https://www.jianshu.com/p/ee0de4c99f87