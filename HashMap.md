[TOC]



# HashMap

参考博文：https://blog.csdn.net/zhangerqing/article/details/8193118

## HashMap的内部存储结构

HashMap的内部存储结构：哈希表。一般实现哈希表的方法采用“拉链法”。

![拉链法](拉链法.png)

从上图中，可以发现哈希表是由数组+链表组成的，一个长度为16的数组中，每个元素存储的是一个链表的头节点。它的内部其实是用一个Entity数组来实现的，属性有key、value、next。

### 初始化

首先来看三个常量：

```java
static final int DEFAULT_INITIAL_CAPACITY = 16;    //初始容量：16
static final int MAXIMUM_CAPACITY = 1 << 30;    //最大容量：2的30次方：1073741824
static final float DEFAULT_LOAD_FACTOR = 0.75f;    //装载因子
```

再来看无参构造方法，也是最常用的构造方法：

```java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    threshold = (int)(DEFAULT_INITIAL_CAPACITY * DEFAULT_LOAD_FACTOR);
    table = new Entry[DEFAULT_INITIAL_CAPACITY];
    init();
}
```

其中loadFactor、threshold用于扩容。table说明默认开辟16个大小的空间。再来看另一个重要的构造方法。

```java
public HashMap(int initialCapacity, float loadFactor) {
    if(initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " + initialCpacity);
    if(initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if(loadFactor <=0 || Float.isNan(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
    
    //find a power of 2 >= initialCapacity
    init capacity = 1;
    while(capacity < initialCapacity)
        capacity <<= 1;
    
    this.loadFactor = loadFactor;
    threshold = (int)(capacity * loadFactor);
    table = new Entry[capacity];
    init();
}
```

重点在于

```java
while(capacity < initialCapacity)
        capacity <<= 1;
...
table = new Entry[capacity];
```

所以，这种构造方法最终构造出来的表大小取决于capacity的大小。

### put(Object key, Object value)操作

当调用put操作时：

```java
public V put(K key, V value) {
    if(key == null)
        return putForNullKey(value);
    int hash = hash(key.hashCode());
    int i = indexFor(hash, table.length);
    for(Entry<K, V> e = table[i]; e != null; e = e.next) {
        Object k;
        if(e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    addEntry(hash, key, value, i);
    return null;
}
```

#### key为null时

调用putForNullKey(V value)，源码如下：

```java
private V putForNullKey(V value) {
    for(Entry<K, V> e = table[0]; e != null; e = e.next) {
        if(e.key == null) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    addEntry(0, null, value, 0);
    return null;
}
```

从Entry数组的table[0]开始遍历，知道找到第一个key为null的Entry，将其value设置为新的value。

如果没有找到key为null的元素，则用addEntry(0, null, value, 0);增加一个新的entry，代码如下：

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K, V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<K, V>(hash, key, value, e);
    if(size++ >= threshold)
        resize(2 * table.length);
}
```

#### key不为null时

hash函数的实现如下：

```java
static int hash(int h) {
    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position hava a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

总结一下：前两行检查key是否为空，然后取得key的哈希索引值，再之后的for循环时检查索引为i的这条链上有没有key重复的，有则替换并返回原值，无则不处理。最后插入新的Entry。

### get(Object key)操作

get(Object key)操作是根据键来获取值，源码如下：

```java
public V get(Object key) {
    if (key == null) {
        return getForNullKey();
    }
    int hash = hash(key.hashCode());
    for (Entry<K, V> e = table[indexFor(hash, table.length)]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
            return e.value;
    }
    return null;
}
```

#### key为null时

调用getForNullKey()，源码如下：

```java
private V getForNullKey() {
    for (Entry<K, V> e = table[0]; e != null; e = e.next) {
        if (e.key == null)
            return e.value;
    }
    return null;
}
```

#### key不为null时

先用hash函数得到hash值，再用indexFor()得到索引i的值，然后循环遍历链表，如果存在key值相等，则返回其value，否则返回空值。

### 总结HashMap的put和get操作

```java
//存储时
int hash = key.hashCode();
int i = hash % Entry[].length;
Entry[i] = value;

//取值时
int hash = key.hashCode();
int i = hash % Entry[].length;
return Entry[i];
```

---

## HashTable的内部存储结构

HashTable和HashMap采用相同的存储机制，二者的实现基本一致，不同的是：

1、HashMap是非线程安全的，HashTable是线程安全的，其内部方法基本都是synchronized。

2.HashTable不允许有null值的 存在。

3.在HashTable中调用put方法时，如果key为null，直接抛出NullPointerException.

---

## HashTable和ConcurrentHashMap的比较

ConcurrentHashMap是线程安全的HashMap的实现。

不同于HashTable用synchronized关键字加锁，ConcurrentHashMap是基于lock操作的，这样在保证同步的时候，不会锁住整个对象。

### 构造方法

ConcurrentHashMap是基于Segment数组的，一种类似与Entry的数据结构：

```java
public ConcurrentHashMap() {
    this(16, 0.75F, 16);
}
```

默认传入值16，调用如下方法：

```java
public ConcurrentHashMap(int paramInt1, float paramFloat, int paramInt2) {
    if ((paramFloat <= 0F) || (paramInt1 < 0) || (paramInt2 <= 0))
        throw new IllegalArgumentException();
    
    if(paramInt2 > 65536) {
        paramInt2 = 65536;
    }
    
    int i = 0;
    int j = 1;
    while (j < paramInt2) {
        ++i;
        j <<= 1;
    }
    this.segmentShift = (32 - i);
    this.segmentMask = (j - 1);
    this.segments = Segment.newArray(j);
    
    if(paramInt1 > 1073741824)
        paramInt1 = 1073741825;
    int k = paramInt1 / j;
    if(k * j < paramInt1)
        ++k;
    int l = 1;
    while (l < k)
        l << = 1;
    
    for(int i1 = 0; i1 < this.segments.length; ++i1)
        this.segments[i1] = new Segment(l, paramFloat);
}
```

这三个参数，paramInt1就是initialCapacity，数组的初始化大小，paramFloat为loadFactor装载因子，而多出来的paramInt2则是concurrentLevel。

### put操作

```java
public V put(K paramK, V paramV) {
    if (paramV == null) 
        throw new NullPointerException();
    int i = hash(paramK.hashCode());
    return segmentFor(i).put(paramK, i, paramV, false);
}
```

当key为null时，直接抛出NullPointer异常，并且此处hash函数也与HashMap中不一样：

```java
private static int hash(int paramInt) {
    paramInt += (paramInt << 15 ^ 0xFFFFCD7D);
    paramInt ^= paramInt >>> 10;
    paramInt += (paramInt << 3);
    paramInt ^= paramInt >>> 6;
    paramInt += (paramInt << 2) + (paramInt << 14);
    return (paramInt ^ paramInt >>> 16);
}
```

segmentFor()源码如下：

```java
final Segment<K, V> segmentFor(int paramInt) {
    return this.segments[(paramInt >>> this.segmentShift & this.segmentMask)];
}
```

根据上述代码找到Segment对象后，调用put操作：

```java
V put(K paramK, int paramInt, V paramV, boolean paramBoolean) {
    lock();
    try {
        Object localObject1;
        Object localObject2;
        int i = this.count;
        if (i++ > this.threshold)
            rehash();
        ConcurrentHashMap.HashEntry[] arrayOfHashEntry = this.table;
        int j = paramInt & arrayOfHashEntry.length - 1;
        ConcurrentHashMap.HashEntry localHashEntry1 = arrayOfHashEntry[j];
        ConcurrentHashMap.HashEntry localHashEntry2 = localHashEntry1;
        while ((localHashEntry2 != null) && (((localHashEntry2.hash != paramInt) || (!(paramK.equals(localHshEntry2.key)))))) {
            localHashEntry2 = localHashEntry2.next;
        }
        
        if (localHashEntry2 != null) {
            localObject1 = localHashEntry2.value;
            if (!(paramBoolean))
                localHashEntry2.value = paramV;
        }
        else {
            localObject1 = null;
            this.modCount += 1;
            arrayOfHashEntry[j] = new ConcurrentHashMap.HashEntry(paramK, paramInt, localHashEntry1, paramV);
            this.count = i;
        }
        return localObject1;
    } finally {
        unlock();
    }
}
```

lock()时ReentrantLock类的一个方法，用当前存储的个数+1来和threshold比较，如果大于threshold，则进行rehash，并将容量扩大2倍，重新进行hash。之后对hash的值和数组大小-1按行按位与操作后，得到当前的key需要放入的位置，之后与HashMap一样。

**ConcurrentHashMap基于concurrentLevel划分出了多个Segment来对key-value进行存储，从而避免每次锁定整个数组，在默认的情况下允许16个线程并发无阻塞的操作集合对象，尽可能地减少并发时的阻塞现象。**

------

参考博文：https://blog.csdn.net/zhangerqing/article/details/8193118

