---
title: Collection集合之HashMap源码学习记录（JDK8）
date: 2024-06-24 12:03:27
tags:
---

@[TOC](Collection集合之HashMap源码学习记录（JDK8）)
# 1.HashMap的UML图：
![Alt](https://img-blog.csdnimg.cn/direct/3fd97c7cc7894963ad7b8a61ac2334f2.png)​​​​
## 1.1. 关于AbstractMap
AbstractMap提供了Map接口的基本实现，最大限度的减少了实现Map接口所需的工作量。

 - 要实现一个不可修改的Map时，仅需继承AbstractMap并实现抽象方法entrySet().对于不可修改的Map该方法返回的Set应该不支持add/remove方法，set的迭代器也不应该支持remove方法。
```java 
public abstract Set<Entry<K,V>> entrySet();
```
- 实现可修改的Map时，需要实现entrySet()方法的同时，实现put(key,val)方法，缺少实现时调用put方法将抛出UnsupportedOperationException异常。与不可修改的Map相反，entrySet()放回的set需要支持add/remove方法，且该set的迭代器需要支持remove方法。
```java
    /**
     * {@inheritDoc}
     *
     * @implSpec
     * This implementation always throws an
     * <tt>UnsupportedOperationException</tt>.
     *
     * @throws UnsupportedOperationException {@inheritDoc}
     * @throws ClassCastException            {@inheritDoc}
     * @throws NullPointerException          {@inheritDoc}
     * @throws IllegalArgumentException      {@inheritDoc}
     */
    public V put(K key, V value) {
        throw new UnsupportedOperationException();
    }
```
# 2.HashMap
HashMap是Map接口基于Hash table的实现类。实现了Map接口的所有可选操作，允许null作为Map的key或value。HashMap不保证数据的顺序性，且不保证数据的顺序在一段时间内保持不变（当HashMap数据达到扩容限制时会重新hash，数据顺序发生改变）。

HashMap使用了数组+链表+红黑树的实现
![ALT](https://img-blog.csdnimg.cn/direct/4ef2eefdcb0948b7bf263281dea11e26.png)


## 2.1. 数据存储
HashMap的数据存放在transient Node<K,V>[] table中，即上述数组+链表+红黑树中的数组，table中的每个table[i]则是一个桶(bucket)，对应数组+链表+红黑树中的链表，当链表中的数据量大于阈值时，链表结构将重构成为数组+链表+红黑树中的红黑树。
Node<K,V>本质上是对Map.Entry<K,V>的实现，其中存放了hash值，key值，value值和next节点，其中hash值标记了数据所属的桶(bucket)。
![ALT](https://img-blog.csdnimg.cn/direct/9c7d93480d054e2faca171c19edc2c94.png)
当桶的数据结构由链表转换成红黑树时，Node<K,V>结构也对应的转换为TreeNode<K,V>。
![ALT](https://img-blog.csdnimg.cn/direct/c094ae14905544ef90fa8d83fb9d4681.png)


## 2.2. 几个关键静态变量
```java
    /**
     * The default initial capacity - MUST be a power of two.
     * 默认初始化容量为16（必须是2的幂）
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     * 最大容量，如果构造方法中指定了更大的容量，则使用此容量代替。
     * 最大容量不大于2的30次方。
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * The load factor used when none specified in constructor.
     * 负载系数，构造方法中未指定时使用默认值0.75
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     * 桶中的链表结构转换为树结构的阈值，当向已经有TREEIFY_THRESHOLD这么多节点的桶中添加值时
     * 量表结构将转换为树结构。TREEIFY_THRESHOLD的值必须大于2，且至少应该为8，以满足删除节点
     * 后树收缩而转回bin的链表结构的假定。
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection under removal.
     * resize时桶中树结构退化成链表结构的计数阈值，必须小于TREEIFY_THRESHOLD阈值，
     * 且最大为6以满足删除操作中的收缩检测。
     */
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
     * The smallest table capacity for which bins may be treeified.
     * (Otherwise the table is resized if too many nodes in a bin.)
     * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
     * between resizing and treeification thresholds.
     * 桶中链表结构树化的最小容量（当容量小于MIN_TREEIFY_CAPACITY，而桶中节点过
     * 多时，优先扩展表容量而不是进行树化操作）MIN_TREEIFY_CAPACITY至少应该为
     * TREEIFY_THRESHOLD的4倍，以避免resize和树化的阈值发生冲突。
     */
    static final int MIN_TREEIFY_CAPACITY = 64;
```
DEFAULT_INITIAL_CAPACITY和MAXIMUM_CAPACITY分别指定了数组+链表+红黑树中的数组的默认容量和最大容量。

DEFAULT_LOAD_FACTOR指定了数组的负载系数，当数组中已使用的下表数达到threshold（其容量*负载系数）时，HashMap就会进行resize操作。

TREEIFY_THRESHOLD指定了HashMap桶中的链表结构树化的阈值。

UNTREEIFY_THRESHOLD指定了HashMap桶中红黑树退化为链表的阈值。

MIN_TREEIFY_CAPACITY指定了HashMap进行树化的最小数组容量。

## 2.3. 添加值
首先通过hash(K)方法计算键的hash值，并通过与操作定位到桶，然后将key-value插入到桶中链表的尾端。

```java
    /**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    /**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; 
        Node<K,V> p; 
        int n, i;
        // 当table为空时，HashMap处于初始化状态，使用resize()初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        // 当桶i（table[i]）为null时，说明其内还未存放元素，将table[i]初始化为新的节点
        // 此时p为key对应桶的链表的头节点
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            // 若p的hash值与新插入的key的hash值相同，且p的key与新插入的key相等
            // 则新插入的key已存在，将其赋值给e。
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 如果p是TreeNode节点类似，则调用红黑树的putTreeVal方法插入节点
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
            	// 遍历桶中的链表
                for (int binCount = 0; ; ++binCount) {
                	// 遍历至尾节点
                    if ((e = p.next) == null) {
                    	// 将新的key-val插入到尾部
                        p.next = newNode(hash, key, value, null);
                        // 当桶大小达到TREEIFY_THRESHOLD时，
                        // 调用treeifyBin对链表做树化处理
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
					// 当链表中的元素的key与新插入的key相同时停止遍历
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            // e!=null说明HashMap中key已经存在
            if (e != null) { // existing mapping for key
            	// 取原key-val的value值
                V oldValue = e.value;
                // 原value替换为新的value值
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                // 返回旧值
                return oldValue;
            }
        }
        ++modCount;
        // HashMap的size大于threshold（容量*loadfactor）时调用resize
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
## 2.4. 树化
考虑一种极端情况，所有put的新key-value键值对的key对应的hash值都相同时，所有的键值对都会插入到同一个桶的末尾，当插入到第九个键值对时超过了TREEIFY_THRESHOLD的阈值8，此时将进入treeifyBin方法尝试树化链表。
![Alt](https://img-blog.csdnimg.cn/direct/53008a7808154ce78a33cf7bcd985fc3.png)
但此时HashMap的table大小为默认值16，小于最小树化阈值64，所以会优先进行resize操作扩容。

![Alt](https://img-blog.csdnimg.cn/direct/32c91a2b1f1648a6a5952790dc05786f.png)

在resize方法中对table大小做2倍扩容

![Alt](https://img-blog.csdnimg.cn/direct/e3e1f7049afc4a8aa491e8195308e6a1.png)
继续添加hash值相同的键值对，再次resize后table的大小达到64，此时再次添加相同hash值的键值对，进入treeifyBin时就会进入树化逻辑，如下图：
![Alt](https://img-blog.csdnimg.cn/direct/924a594440ee436d84b16f350a01607c.png)


## 2.5. 扩容
通常，当使用put添加键值对时，若HashMap的size达到threshold时，调用resize()方法对table进行扩容。
![Alt](https://img-blog.csdnimg.cn/direct/e96db752b18344179a5426efe9e2e227.png)
扩容时table大小变为原来的两倍，threshold=table容量*loadFactor同样变为原来的两倍。
![Alt](https://img-blog.csdnimg.cn/direct/1a24644113274b058175079b847942ee.png)
因为使用的是2倍扩容的方式，扩容后每个桶的索引要么保持原样，要么以2的幂的偏移量偏移。

1. 如对于hash值为1000的元素，在容量为16的table中的索引为1000&(16-1) = 8，即001111101000 & 1111 = 1000
2. 当table扩容一次变为32时，还是hash值为1000的元素，在table中的索引为1000&(32-1) = 8 ，即001111101000 & 00011111 = 1000 仍然为8，该桶的索引没有发生变化
3. 当table再次扩容，容量达到64时，hash值为1000的元素在table中的索引为1000&(64-1) = 40，即001111101000 & 00111111 = 00101000 此时，该桶的索引有了32(00100000)的偏移
4. 当table再次扩容，容量达到128时，hash值为1000的元素，在table中的索引为1000&(128-1) = 104，即001111101000 & 01111111 = 01101000 此时，该桶的索引有了64(01000000)的偏移

仔细观察会发现这个扩容后的索引即hash值与最高位1相与的值与原索引的和。当hash值与最高位1相与为0时，索引不发生变化，而当与运算结果为1时，则偏移最高位对应的值，如上述第2步，hash值与最高位与运算结果位0，桶的索引不变；而第3步及第4步，hash值与最高位与运算结果位1，最高位分别对应00100000=32及01000000=64，桶的索引对应偏移32和64。

# 3. HashMap几个常用方法
## 3.1. hash(Object key)
```java
    /**
     * Computes key.hashCode() and spreads (XORs) higher bits of hash
     * to lower.  Because the table uses power-of-two masking, sets of
     * hashes that vary only in bits above the current mask will
     * always collide. (Among known examples are sets of Float keys
     * holding consecutive whole numbers in small tables.)  So we
     * apply a transform that spreads the impact of higher bits
     * downward. There is a tradeoff between speed, utility, and
     * quality of bit-spreading. Because many common sets of hashes
     * are already reasonably distributed (so don't benefit from
     * spreading), and because we use trees to handle large sets of
     * collisions in bins, we just XOR some shifted bits in the
     * cheapest possible way to reduce systematic lossage, as well as
     * to incorporate impact of the highest bits that would otherwise
     * never be used in index calculations because of table bounds.
     */
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
对原key的hash值做了处理，将原hash值的高十六位无符号右移16位至低位后与原hash值做异或运算得到新的hash值。例如
原hash值为 |1111 0000 1010 0101 0101 1010 1111 0000
-------- | ------------
**右移16位得到**	  |**0000 0000 0000 0000 1111 0000 1010 0101**
**异或运算后得到** |	**1111 0000 1010 0101 1010 1010 0101 0101**

这么做的目的是将hash值的高位的影响带入到hash过程中去（这种方法并不能完全解决hash冲突的问题）。

HashMap的table大小总是2的幂，当hash值仅在table大小以上的位变化时，总是会出现hash冲突，如table大小为16，则索引的计算为hash&(16-1)，即hash&0000 0000 0000 0000 0000 0000 0000 1111，当hash值的变化在15以上的地方发生时，hash总是会出现冲突。

已知的如在一个小的table中存放连续的float值作为key时就会出现这种情况，我们尝试输出几个接近浮点数的hash值：

  浮点数       |             hash值
  ----|-----
39999990f     |  1001100000110001001011001111110
39999991.1f  |  1001100000110001001011001111110
39999991.2f  |  1001100000110001001011001111110
39999992f      | 1001100000110001001011001111110
39999992.1f   | 1001100000110001001011001111110
39999992.2f  |  1001100000110001001011001111110
39999993.1f   | 1001100000110001001011001111110
39999994f     |  1001100000110001001011001111110
39999995f    |   1001100000110001001011001111111

可以看到除了39999995f外其他几个数的hash值完全相同，即便将高位的影响带入到低位的hash计算仍会出现冲突，个人建议是重写Float的hash算法或者避免使用连续或相近的float作为Hashmap的键值。

## 3.2. get(Object key)和containsKey(Object key)
```java
    /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code (key==null ? k==null :
     * key.equals(k))}, then this method returns {@code v}; otherwise
     * it returns {@code null}.  (There can be at most one such mapping.)
     *
     * <p>A return value of {@code null} does not <i>necessarily</i>
     * indicate that the map contains no mapping for the key; it's also
     * possible that the map explicitly maps the key to {@code null}.
     * The {@link #containsKey containsKey} operation may be used to
     * distinguish these two cases.
     *
     * @see #put(Object, Object)
     */
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }


    /**
     * Returns <tt>true</tt> if this map contains a mapping for the
     * specified key.
     *
     * @param   key   The key whose presence in this map is to be tested
     * @return <tt>true</tt> if this map contains a mapping for the specified
     * key.
     */
    public boolean containsKey(Object key) {
        return getNode(hash(key), key) != null;
    }
```
get和containsKey内部的实现都是去查找key对应的节点，调用了getNode(hash,key)方法，先通过hash值找到key所在的桶，然后遍历桶中的链表或树，来查找key对应的节点。

## 3.3. getOrDefault(key, defaultValue)
```java
 @Override
public V getOrDefault(Object key, V defaultValue) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
}
```
在HashMap中查找key对应的value，若key不存在，则返回defaultValue。
## 3.4. put(key,value)
具体参考2.3. 添加值

## 3.5. putIfAbsent(key, value)
```java
@Override
public V putIfAbsent(K key, V value) {
    return putVal(hash(key), key, value, true, true);
}
```
仅当key不存在时做put操作。

## 3.6. remove(key)
```java
    /**
     * Removes the mapping for the specified key from this map if present.
     *
     * @param  key key whose mapping is to be removed from the map
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */
    public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }
```
remove操作先通过key的hash值找到对应的桶，然后遍历链表或树找到key，并将key从链表或树中删除

## 3.7. containsValue(value)
```java
    /**
     * Returns <tt>true</tt> if this map maps one or more keys to the
     * specified value.
     *
     * @param value value whose presence in this map is to be tested
     * @return <tt>true</tt> if this map maps one or more keys to the
     *         specified value
     */
    public boolean containsValue(Object value) {
        Node<K,V>[] tab; V v;
        if ((tab = table) != null && size > 0) {
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                    if ((v = e.value) == value ||
                        (value != null && value.equals(v)))
                        return true;
                }
            }
        }
        return false;
    }
```
containsValue会遍历整个hashMap的table

## 3.8.replace(key, oldValue, newValue)
```java
@Override
public boolean replace(K key, V oldValue, V newValue) {
    Node<K,V> e; V v;
    if ((e = getNode(hash(key), key)) != null &&
        ((v = e.value) == oldValue || (v != null && v.equals(oldValue)))) {
        e.value = newValue;
        afterNodeAccess(e);
        return true;
    }
    return false;
}
```
当HashMap中key对应的原值为oldValue时，使用newValue做替换操作，返回true；否则当key不存在，或key对应原值不为oldValue时，返回false。