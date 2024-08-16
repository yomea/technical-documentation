## 一、HashMap基础

### 1.1 简介

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/b7f1ab48b73fbfed8cd1777b0830b7ef.png)

- Map：定义基本的增删除改查操作
- AbstractMap：模板方法，实现了一些基本方法

HashMap是使用数组+链表+红黑树的方式构成的，它与ConcurrentHashMap的主要区别就是一个是线程不安全，一个是线程安全，在代码上可能会有些差别，但是在计算容量，定位
索引等原理上是基本差不多的，想要更多的了解ConcurrentHashMap，请转移到[ConcurrentHashMap博文](https://blog.csdn.net/qq_27785239/article/details/102713655)。

### 1.2 字段说明

```java
//HashMap的默认大小16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

//HashMap的最大容量，2的30次方，有符号的int类型
static final int MAXIMUM_CAPACITY = 1 << 30;

//负载因子，用于计算容量阈值，达到阈值后将发生扩容
static final float DEFAULT_LOAD_FACTOR = 0.75f;

//转换成红黑树的阈值，hash数组下标上的某个链表如果达到8个以上（含8个），可能会转换成红黑树
//为什么是可能？除了节点个数达到8个以上（含8个），还要看hash数组的大小是否大于等于64，如果不大于，HashMap将优先使用扩容
//将节点重新hash
static final int TREEIFY_THRESHOLD = 8;

//退化为链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;

//这个就是转换为红黑树的hash数组最小容量
static final int MIN_TREEIFY_CAPACITY = 64;

//修改计数，通常用于迭代器循环的时候进行并发检查
//如果使用迭代器迭代元素的时候，发生删除或者添加元素，将抛出ConcurrentModificationException异常
transient int modCount;
```

### 1.3 hash操作

HashMap的计算某个key值的hash操作和ConcurrentHashMap是一样的，以下是HashMap计算hash值和进行索引定位的代码

```java
//计算hash值
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

//索引定位
tab[i = (n - 1) & hash]
```


调用key自身的hashCode方法得到hash值，然后与hash值的高16进行异或。这么做是为了进行hash扰乱，将高16的变化能够得到体现。如果我们HashMap的数组的容量小于2的16次
方，而大部分的hash值都在高16位进行变化，那么他们进行 hash & (n - 1)得到的索引下标将不会变，导致严重的hash冲突，为了避免出现这样的情况，那么就需要将高16位发
生的变化也体现到低16位上。

> Q: 为什么是异或而不是什么位或，位与呢？

> A: 将一个hash值进行无符号右移之后，高16位为0，如果两个值进行相与，那么高16将全部变成零，丢失了高16，显然是不合适的，再者，如果低16位上某bit位上是零，那么
高16位上与之相对应的bit位即使发生了变化，它位与得到的值依然是0，无法体现出高16位的这种变化，如果是位或，高16位可以得到保留，但是低16位中只要有bit位为1的，那
么得到的bit位就是1，与之相对的高16对应位上的变化无法体现，举个例子

```java
低16位   1001101101001111                        1001101101001111
高16位   1001010010001000    --> 改变高位bit位   1001010010001101  --> 只改了第1位和第3位
位或     1001111111001111                        1001111111001111  --> 没能体现出高16的第1位和第3位的变化
异或     0000111111000111                        0000111111000010  --> 体现出了高16的第1位和第3位的变化

```
再来个更加残酷的例子

```java
低16位   1111111111111111                        1111111111111111
高16位   1001010010001000    --> 改变高位bit位   1111111111111111  --> 把高16位的bit位全部设置为1
       --------------------                    --------------------
位或     1111111111111111                        1111111111111111  --> 不管高16位怎么变，这个值毫无改变，无法体现出高16的任何变化
异或     0000111111000111                        0000000000000000  --> 体现出了高16的变化，如果此时高16任何一处的bit变成零，它都能捕捉到它的变化

```
另外，与一个全零的数进行异或等于本身

```java
   1001010010001000
 ^ 0000000000000000
--------------------
 = 1001010010001000
```
所以hash值与hash >>> 16进行异或不仅不会导致高16位丢失，还能把高16的变化体现出来。

> Q: 为什么 hash % n = (n - 1) & hash？

> A: 首先先声明，只有当n为2次幂的值的时候，hash % n = (n - 1) & hash，为什么呢？hash对一个2次幂的值进行求余的时候，我们知道 hash / 2 ^ n = hash >>> n
其实hash向右移出去的值就是余数，对于一个n为2次幂的整数值，它的所有bit位上只有一个1，比如 16 = 000...10000，它是2的4次方的值，hash / 2 ^ 4 = hash >>> 4,
如果此时我们要得到hash被移出去的4位bit位怎么办？ 那当然是 hash & 000...01111 = hash & (16 - 1) 把16替换成n就变成了hash & (n - 1)

### 1.4 扩容

```java
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
    //如果还没有初始化，并且有设置阈值，那么将这个阈值设置为新的容量
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        //如果没有初始化，设置为默认的容量16
        newCap = DEFAULT_INITIAL_CAPACITY;
        //计算容量阈值，容量乘以 0.75，在ConcurrentHashMap中是通过 DEFAULT_LOAD_FACTOR - (DEFAULT_LOAD_FACTOR >> 2) 类似的操作
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        //通过新容量计算新的阈值
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        。。。。。。
    }
    return newTab;
}
```
首先检查是否已经初始化过，如果初始化过，那么扩容为原来的2倍，如果原来的容量大于默认的容量，那么相应的阈值也直接扩容为原来的2倍
要是还没有初始化，那么设置默认的容量

### 1.5 迁移

```java
final Node<K,V>[] resize() {
    。。。。。。省略扩容代码
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                //将老的hash数组对应的数值置为null
                oldTab[j] = null;
                //如果hash数组对应的下标的元素只有一个节点，那么直接定位，将值移到新hash数组即可
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    //
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    //loHead：表示newIndex == oldIndex的节点链表中的头节点
                    //loTail：表示newIndex == oldIndex的节点链表中的尾节点
                    Node<K,V> loHead = null, loTail = null;
                    //hiHead：表示newIndex == oldIndex + oldCap的节点链表中的头节点
                    //hiTail：表示newIndex == oldIndex + oldCap的节点链表中的尾节点
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        //hash值与旧容量相与，这种hash操作我们在ConcurrentHashMap那章研究过
                        //如果是零，节点的下标和原来一样，如果是1，那么下标等于 oldIndex + oldCap
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
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
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

上面这段迁移代码除了没有锁，cas之外，原理和ConcurrentHashMap是一样的，感兴趣的可以移步到[博文ConcurrentHashMap](https://blog.csdn.net/qq_27785239/article/details/102713655)。HashMap在迁移链表和红黑树节点的时候，没有重
新计算节点的位置，而是通过 hash & oldCap 的方式去确定他们的位置，原理如下：

```java
oldCap = 000...0010000
newCap = 000...0100000
```
新容量与旧容量的二进制相比，就是左移了一位，当他们减去1的时候

```java
oldCap - 1 = 000...0001111
newCap - 1 = 000...0011111
```
当一个hash与上面两个值进行位与的时候，如果这个hash值的第5位的bit值为零，那么算出来的下标和原来是一样的，如果这个bit位为1，那么算出来的下标就是
oldIndex + oldCap

关于红黑树的部分可以移步到[博文TreeMap](https://blog.csdn.net/qq_27785239/article/details/102791617)

### 1.6 视图

#### 1.6.1 KeySet

##### 1.6.1.1 迭代

对于keySet我们主要研究下对应的迭代器KeyIterator

```java
public final Iterator<K> iterator() { 
    return new KeyIterator(); 
}
                |
                |
                V
// KeyIterator 继承自 HashIterator
HashIterator() {
    //modCount是记录HashMap的修改次数
    expectedModCount = modCount;
    Node<K,V>[] t = table;
    current = next = null;
    index = 0;
    if (t != null && size > 0) { // advance to first entry
        do {
            //跳过null值，直到找到下一个节点
        } while (index < t.length && (next = t[index++]) == null);
    }
}

final class KeyIterator extends HashIterator implements Iterator<K> {
    //next
    public final K next() { 
        return nextNode().key; 
    }
}
```
判断是否存在下一个节点

```java
public final boolean hasNext() {
    return next != null;
}
```

获取下一个节点

```java
final Node<K,V> nextNode() {
    Node<K,V>[] t;
    Node<K,V> e = next;
    //如果在循环的过程中，HashMap发生改变，那么抛出错误
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    if (e == null)
        throw new NoSuchElementException();
    //获取当前链表或者红黑树的下一个节点
    if ((next = (current = e).next) == null && (t = table) != null) {
        do {
            //跳过null值，直到找到下一个节点链表或者红黑树的头节点
        } while (index < t.length && (next = t[index++]) == null);
    }
    return e;
}
```

##### 1.6.1.2 删除

```java
public final void remove() {
    //获取迭代到的节点
    Node<K,V> p = current;
    if (p == null)
        throw new IllegalStateException();
    //如果发生并发修改抛出错误
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    current = null;
    K key = p.key;
    //调用HashMap的移除方法
    removeNode(hash(key), key, null, false, false);
    //更新新的修改计数值
    expectedModCount = modCount;
}
```

#### 1.6.2 EntrySet

下面是HashMap的entrySet方法

```java
public Set<Map.Entry<K,V>> entrySet() {
    Set<Map.Entry<K,V>> es;
    return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
}

```
EntrySet使用的迭代器为EntryIterator，它是HashMap的内部类，其实和KeyIterator在原理上是一样的

```java
public final Iterator<Map.Entry<K,V>> iterator() {
    return new EntryIterator();
}
```
迭代操作

```java
final Node<K,V> nextNode() {
    Node<K,V>[] t;
    Node<K,V> e = next;
    //如果在迭代的过程中发生修改，抛出错误
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    if (e == null)
        throw new NoSuchElementException();
        //获取当前链表或者红黑树的下一个节点
    if ((next = (current = e).next) == null && (t = table) != null) {
        do {
            //循环找到下一个链表或者红黑树的头节点
        } while (index < t.length && (next = t[index++]) == null);
    }
    return e;
}
```
可以看出对视图的操作最终都会反映到HashMap上

## 二、总结

&nbsp;&nbsp;&nbsp;&nbsp;除了线程安全，CAS操作，计数机制外，HashMap与ConcurrentHashMap的原理基本一样。除此之外，与之相似的一个集合类叫HashSet，这个HashSet
内部直接关联了一个HashMap，下面是它的构造器

```java
public HashSet() {
    map = new HashMap<>();
}
```
下面是它的add方法

```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```
PRESENT是一个固定的Object对象。

还有一个维护了节点添加顺序的类LinkedHashMap，它继承自HashMap，它除了将值put到HashMap中，它还用两个成员变量，head和tail用于维护一个单独的链表保持添加顺序，其主要逻辑在创建新节点的时候

```java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    //将新创建的节点添加到链表尾部
    linkNodeLast(p);
    return p;
}

private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
```
当我们调用keySet和entrySet方法的时候，其内部使用的迭代器都是通过LinkedHashMap单独维护的链表进行遍历的。