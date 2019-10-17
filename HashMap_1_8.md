# HashMap在JDK1.8中的实现

## HashMap源码分析

### 静态变量

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;    //16,默认初始容量必须是2的整数幂
static final int MAXIMUM_CAPACITY = 1 << 30;    //最大容量
static final float DEFAULT_LOAD_FACTOR = 0.75f;    //默认装载因子

//以下为1.8新增变量
static final int TREEIFY_THRESHOLD = 8;    //当一个槽中的链表长度大于这个值时，将链表转化为树
static final int UNTREEIFY_THRESHOLD = 6;    //当一个槽中树的节点数数小于这个值时，转化为链表
static final int MIN_TREEIFY_CAPACITY = 64;    //
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

