---
title: HashMap（JDK1.8）的扩容时机与阈值的探讨
date: 2020/02/17 15:51:49
tags: 集合
categories: 集合
---



最近又开始复习一些基础知识，发现网上很多关于HashMap的总结有一些问题，尤其是扩容的时机和阈值的确定仍然有一些疑问，因此果断还是自己看看源码。

本文只指出部分关于扩容的问题，其他详细分析可以参考其他文章。

## 究竟什么时候扩容

*  当put时发现table未初始化时，进行初始化扩容
*  当put加入节点后，发现size（键值对数量）>threshold时，进行扩容

在下列源码中分别对应注释

```
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;    //初始化扩容
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
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
            resize();    //超过阈值扩容
        afterNodeInsertion(evict);
        return null;
    }
```

## 阈值是如何确定的
其实最关键的就是阈值的确定，网上大部分文章只说了阈值是 `load factor*current capacity`，这个结论是正确的，但整个过程是根据使用的构造函数而有所不同的。
接下来我们只分析阈值和容量的关系。
下面是resize的部分代码
```
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
       
        /*** 这里省略table表的变化过程***/
    }
```

### 无参构造函数

```
public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```
可以看到这里的无参构造函数只初始化了负载因子为默认值0.75。
== 注意，此时hashmap的table,threshold都还没用初始化，即此时的容量为0，阈值也为0。==
而在第一次put时，检查`table==null`,进行第一次resize
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200217152914993.png)
此时在resize中会进入下面的分支
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200217153215517.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwOTk1MzM1,size_16,color_FFFFFF,t_70)
所以容量cap为默认值16，阈值threshold为 16*0.75=12。
而在之后的每次扩容中，容量和阈值都变为原来的两倍。
即两者仍然保持着0.75的比例。

### 带参数的构造函数

```
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
```

```
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
以上代码除了参数的合法判断以外，主要设置了负载因子和阈值，而阈值由tableSizeFor()函数确定。
```
    /**
     * Returns a power of two size for the given target capacity.
     */
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

可以看到注释，阈值threshold将被设置为最接近给定参数的2^n^的值。
==注意，此时只设置了阈值，table还没初始化，即容量cap还为0。==

同样是在put时，首次进行扩容，而扩容的分支如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200217154502207.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwOTk1MzM1,size_16,color_FFFFFF,t_70)
可以看到，容量cap被设置为了刚刚计算出的阈值2^n^，而阈值被重新设置为cap*loadFactor。虽然两者仍然保持着loadFactor的比例，但是先设置了阈值，然后赋值给了容量，再根据容量和负载因子重新计算阈值。这个过程中，阈值担当了一个中间变量的角色。

而在后续的扩容过程中，容量和阈值仍然保持着负载因子的比例关系，并同时变为原来的两倍。

