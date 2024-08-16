
## 一、ConcurrentHashMap

&nbsp;&nbsp;&nbsp;&nbsp;ConcurrentHashMap是HashMap的线程安全版，下面为了描述方便将ConcurrentHashMap简称为hashmap

### 1.1 类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/fdf7ed01ed0e7beafe7c34b2e196f172.png)

Map：定义基本的增删除改查操作
ConcurrentMap：主要是定义了一些default的方法，比如compute，forEach，定义了接口方法putIfAbsent
AbstractMap：模板方法，实现了一些基本方法

### 1.2 类成员变量

```java
//2的30次方，表示map对应的数组的最大容量，最高位为符号位
private static final int MAXIMUM_CAPACITY = 1 << 30;

//默认大小，是2次幂的大小
private static final int DEFAULT_CAPACITY = 16;

//加载因子，可以通过 n - n >>> 2进行计算出需要扩容的阈值
private static final float LOAD_FACTOR = 0.75f;

//转换成红黑树的阈值，但并不是说达到这个8就一定会转换成红黑树
static final int TREEIFY_THRESHOLD = 8;

//将红黑树退化成链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;

//转换成红黑树的容量
static final int MIN_TREEIFY_CAPACITY = 64;

//扩容时，最小步长
private static final int MIN_TRANSFER_STRIDE = 16;

//扩容时用于校验的数据位
private static int RESIZE_STAMP_BITS = 16;

//RESIZE_STAMP_BITS为16，那么通过这里计算出来的值，低16位全是1
//用于控制参与扩容的最大线程数
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

//校验位偏移位数，默认16
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

//ForwardingNode节点的hash值，扩容时，如果某个hash桶（数组中每个存放元素的位置称为桶）的元素被转移，那么会被ForwardingNode节点占用
static final int MOVED     = -1; // hash for forwarding nodes

//红黑树的根节点的hash值，它的类型为TreeBin
static final int TREEBIN   = -2; // hash for roots of trees

//ReservationNode节点的hash值，这是一个占位符节点，用于computeIfAbsent 和 compute方法
//用于可以传入函数处理，函数的处理多长时间是未知的，为了避免过多线程处理完用户的数据都堆在一起去put值到hash表中，作者在开始调用用户的函数前先通过cas
//竞争，然后插入ReservationNode占位符，下次程序处理完数据后可以直接替换自己已经占好的位置即可
static final int RESERVED  = -3; // hash for transient reservations

//掩码，除了最高位，其他位都是1，所以它和任何32的值位与，都会是一个整数
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

/** Number of CPUS, to place bounds on some sizings */
//获取系统的CPU个数
static final int NCPU = Runtime.getRuntime().availableProcessors();

```

### 1.3 构造方法

```java
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    //MAXIMUM_CAPACITY为2的30次幂，如果初始化容量大小超过最大容量的一半，那么设置初始化容量为
    //最大容量，其他将容量设置为初始化容量 大于并且最接近 3 * initialCapacity / 2 + 1 值的2的2次幂的值
    //比如我们设置的初始化容量是100，那么最接近 3 * 100 / 2 + 1的2的2次幂的是256
    //为什么扩容1.5倍，实际上我们的元素个数达到容量的0.75之后就会扩容了
    //所以这里出入的容量按道理应该乘以4/3，但是为了更好的使用位运算这里
    //直接扩容了1.5倍
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}
```
将一个容量扩容为最接近于用户设置容量的2次幂值的方法如下：

```java
private static final int tableSizeFor(int c) {
    int n = c - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
为什么通过以上各种移位，然后或一下就能得到最接近的2次幂呢？<br />
举个例子，比如我们有一个值是3，那么通过上面的公式调整的大小为（3 * 3 / 2 + 1） = 5，也就是扩容了1.5倍（因为实际容量达到0.75的阈值之后就会发生扩容，按道理需要将我们传入的容量扩容1.333...，但是为了使用更高效的位运算，此处变成扩容1.5倍）加1（把tableSizeFor方法当成一个独立的方法来看，我不知道别人会给我传入什么样的参数，我会认为tableSizeFor方法传入的参数c可能是已经是一个2的幂次方的值了，所以要减去1，相应的，如果我要找一个比自己大的2次幂的值，那就可以加1）得到的值为5
那么与5最接近的2次幂的值为8，也就是调用tableSizeFor(5)的结果为8
由于tableSizeFor内部又减去了1（为什么减一？计算出来的容量可能就是2次幂的值，减去1，那么最近的那个2次幂就是本身） <br />

那么参数移位计算的值为4

int类型4的二进制为
000...0100


```java
(1) 执行n |= n >>> 1
n >>> 1 ======> n为000...0010
     或     000...0100
            000...0010
======> n = 000...0110

(2) 执行n |= n >>> 2
 n >>> 2 ======> n为000...0001
    或      000...0110
            000...0001
======> n = 000...0111

(3) 执行n |= n >>> 4
 n >>> 2 ======> n为000...0000
    或      000...0111
            000...0000
======> n = 000...0111
            .
            .
            .
```


将最终得出的值加1变成了000...1000，也就是8

从上面的例子中我们可以发现，它将最高位第一个为1以下的bit位设置成了1，我们知道一个2次幂的数，在二进制中只有其中一位为1，其余的都是零，那么我们只要通过以上方
式将最高位第一个为1以下的bit位都变成1然后加1那么就得到了最接近的2次幂的值。

比如，原始值000...0100，最高位为1的在第三位，我们最终目标就是把这个值变成000...0111，有以下几个步骤 <br />
1. 我们要把第二位变成1，要怎么做？那就是向右移动一位，然后进行位或就得到了000...0110，也就是上面的 n |= n >>> 1 <br />
2. 我们得到了000...0110之后，我们要把第1位也变成1，也就是000...0111，怎么办？ <br />
   有人说了，我可以继续向右移一位然后位或啊，的确，似乎达到了目的，但是明明我在上面的计算中得到了2个1，我可以移动2个1加快置1的速度
   为什么我只移动1个1呢？ <br />
   所以hashmap有更好的办法，那就是每次移位都是上一次的2倍，为什么是2倍？？？<br />
   我们先忽略某个数值上的其他位置的1，比如010...1010，十进制是2的30次方加10，我们就认准最高位那个1，其他位置的1忽略,
   也就是相当于010...0000（2的30次方），我们第一次移位后变成了011...0000，这个数有两个1了，那么下次我移动2位再进行位或我就能产生4个一，再下次我移动4
   位我就能得到8个1。。。。。。与其说每次移动前一次的两倍，还不如说是因为要让数值上的1能够最大化利用而导致的2倍移动
   由于int类型数值是32位的，并且最高位为符号位，所以我们总共只需要移动31次即可，也就是可以移动5次（1 + 2 + 4 + 8 + 16）


那么问题又来了，为啥要千方百计的弄成2次幂的值？有什么特殊的深意吗？ <br />
在后面的代码中我们可以看到ConcurrentHashMap进行取模的计算公式为 (cap - 1) & hash而不是hash % cap，计算机底层都是通过位运算的，如果省去%符号的解析与转换直接
使用计算机认识的位运算，那将大大提高效率，节省资源的消耗。

下面我们来分析下为什么在cap为2次幂的值的时候 hash % cap = (cap - 1) & hash ？<br />
首先我们假设cap为16
hash / 16 算出来的是除以16的商，由于16为2的4次方，那么也可以通过 hash >> 4 计算出它的商，而余数就是被移除掉的4位
也就是说我们只要把hash的后四位拎出来就是余数了，要想把后四位拎出来，那么就要
hash & 000...1111 16是2的4次方，它的第5位为1（000...10000）,那么我们只要减去1就变成了000...1111
所以得出公式 hash & (16 - 1) = hash % cap

### 1.4 添加元素

#### 1.4.1 基础说明

在分析map的添加元素方法之前，我们先来学习下它是怎么计算hash值的，并且是怎么获取数组下标的
计算hash值的算法如下：

```java
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```
参数h为key值的hash值，也就是直接调用key.hashCode()方法得到的值，那么为啥还要对这个hash值进行 h ^ (h >>> 16)这样的操作呢？
> 在1.3小节说到hashMap通过这样的操作提高了hashmap定位的效率，但是又出现了一个问题，那就是均匀分布的问题，我们依然假设容量为16，那么任何与它进行求余
hash的余数都是该hash值的后四位，如果是一个比容量大的值，它的变化总是发生在高5位上，低四位不变，那么就会出现他们计算的余数都是一样的，所以hashmap需要通过
某种方式将hash进行扰动，从公式h ^ (h >>> 16)来看，它将hash值的高16位与低16进行了异或处理，这样高16的变化也将体现到低16位上。

计算出被扰动的值之后又与上HASH_BITS的值，这是啥意思？
> 在进行高16位与低16位异或的过程中，可能会导致数值变成一个负数，HASH_BITS的值为0x7FFFFFFF，最高位为0，这样就保证值为正数

下面看到hashmap的添加元素的方法

```java
//key：键，value：值，onlyIfAbsent：为true表示在当前键值不存在时添加，存在时不添加
final V putVal(K key, V value, boolean onlyIfAbsent) {
    //键值不能为空
    if (key == null || value == null) throw new NullPointerException();
    //这个方法在上面讲过，就是用来扰动hash值的
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //判断是否需要初始化
        if (tab == null || (n = tab.length) == 0)
            //初始化hash表
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //如果当前下标位置不存在数据，那么通过cas原子替换操作设置值，添加成功直接break，如果失败，继续循环
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //如果当前节点的hash值为MOVED（-1）,表示hash表正在扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            //直接使用synchronized进行锁定
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                //如果onlyIfAbsent为false，替换旧值，否则不做处理
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            //如果下一个节点为空，创建下一个节点
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    //如果节点类型是红黑树结构的，那么通过红黑树的方式插入值
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        //红黑树的方式插入数据
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            //onlyIfAbsent为false时替换数据
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                //如果某个链表所持有的节点个数大于等于转换成树结构的阈值，默认是8，那么就要将当前链表转换成红黑树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    //返回旧值
                    return oldVal;
                break;
            }
        }
    }
    //计数
    addCount(1L, binCount);
    return null;
}
```
上面这段方法主要做了以下操作
- 判断是否已经初始化hash表，如果没有进行初始化
- 检查对应算出的下标的是否有node存在，如果不存在的，直接创建一个node原子操作存放在对应的数组下标中作为根node
- 如果存在节点并且这个节点的hash为-1，表示正在扩容操作，然后帮助扩容
- 同步添加元素（可能是链表操作，也可能是红黑树操作，通过hash值来判断）
- 检查当前数组下标上的链表长度是否达到转换成红黑树的阈值8
- 计数并检查是否需要扩容

#### 1.4.2 初始化容量

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        //如果sizeCtl为负数，表示当前hash表正在初始化操作
        if ((sc = sizeCtl) < 0)
            //其他线程让出cpu
            Thread.yield(); // lost initialization race; just spin
            //原子操作，将sizeCtl字段的值设置为-1
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                //双重检查是否已经初始化完成，因为在当前线程处理之前，有可能被其他线程处理了
                if ((tab = table) == null || tab.length == 0) {
                    //如果我们没有指定hashmap的容量，那么将会使用默认容量
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    //创建一个Node数组
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 3n/4，也就是负载因子为0.75f，这是触发扩容的容量阈值
                    sc = n - (n >>> 2);
                }
            } finally {
                //记录扩容阈值
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```
可以看到，在初始化时作者Doug Lea使用cas原子操作sizeCtl，将其值置为-1，竞争成功的线程进入初始化流程，其他竞争失败的线程进行自旋，为了避免cpu的浪费，
使用Thread.yield()让出cpu时间片，初始化好后，以容量0.75的大小做为下一个扩容阈值

#### 1.4.3 计数并判断是否需要扩容

```java
private final void addCount(long x, int check) {
    //CounterCell是一个嵌套类，用于维护计数
    //CounterCell从源码注释说明了，它的容量大小也是2的幂次方
    //CounterCell数组，它用于存储每个线程的计算，所以它的计数方式是通过独立的线程相互隔离的
    CounterCell[] as; long b, s;
    //如果counterCells不为空，说明已经发生了竞争，如果为空，那么可能没有竞争，通过原子操作的方式使用baseCount进行计数
    //一旦原子操作计数失败，那么说明发生了竞争
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        //竞争标志，用于标识CounterCell数组某个下标位置是否存在竞争
        boolean uncontended = true;
        //如果counterCells没有初始化，或者已经初始化了，但是对应线程所在的单元值为空
        //或者出现多个线程竞争用一个计数单元，那么就需要进行扩容
        //ThreadLocalRandom.getProbe()的值在调用了ThreadLocalRandom.localInit()时便会初始化，内部使用了AtomicLong进行递增操作
        //所以不同线程的probe的值是不同的
        if (as == null || (m = as.length - 1) < 0 ||
            //由于CounterCell的容量为2的幂次方，所以这里的[ThreadLocalRandom.getProbe() & m就是在求余
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            //如果cas操作某个计数单元失败，那么说明这个计数单元发生竞争，有多个线程的发生了hash碰撞，uncontended变为false
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            //进行计数单元的初始化或者扩容，自旋cas设置值
            fullAddCount(x, uncontended);
            return;
        }
        //这个值是putVal方法中的binCount值
        if (check <= 1)
            return;
        //循环counterCells，计算总个数
        s = sumCount();
    }
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        //检查节点数是否已经超过了阈值，如果超过了，那么将需要进行扩容
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            //计算公式Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1))
            //Integer.numberOfLeadingZeros(n)是用来计算前导零的个数，从高位起一直到低位的第一个不为零为止，最大值也就是32
            //1 << (RESIZE_STAMP_BITS - 1)  RESIZE_STAMP_BITS的值为16，将1向左移动15位，也就是第16位上的值为1
            //然后两者进行位或
            int rs = resizeStamp(n);
            //如果sc小于零，那么表示正在扩容
            if (sc < 0) {
                //向右移动16位（在扩容时，rs的值会向左移动16位，然后赋值给sc，此处向右移动16位相当于恢复原来的值）
                //然后和原来的rs值进行对比，如果不相等，那么认为扩容已经结束
                //sc == rs + 1 表示扩容基本已完成，还剩一个线程在做最后收尾操作
                //sc == rs + MAX_RESIZERS相等表示参与扩容的线程数已经达到最大了
                //(nt = nextTable) == null 表示扩容完成，扩容后的hash数组已经复制给tab，nextTable会被置null
                //transferIndex <= 0 表示任务范围都已经划分出去了
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                //将参与扩容的线程数加1，然后一起帮助扩容操作
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            //sc在扩容开始前是一个正值，也就是扩容阈值，在开始扩容时将通过cas原子操作进行赋值
            //rs << RESIZE_STAMP_SHIFT 将上面得到rs向左移动16位，我们知道rs的第16位上的值为1，如果直接向左移动16位的话，那么它会是一个负值
            //向左移动16位将rs分割成高16位与低16位，低16位用于计算有多少线程正在参与扩容，高16位用于进行校验（就是上面sc < 0里面的校验是否已经结束了扩容）
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                //进行扩容操作
                transfer(tab, null);
            //计算当前节点个数，用于循环判断扩容
            s = sumCount();
        }
    }
}
```
首先判断是否存在计算单元，没有的，通过原子递增baseCount, 如果存在计算单元，用当前线程的Probe值计算索引下标进行原子操作
对于cas操作baseCount或者cas操作计算单元失败的将会进入计算单元的初始化，自旋cas，直到更新成功

我们单独把 Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1)) 拎出来举个例子，假设我们当前map的hash桶大小为16
那么16的二进制为000...10000，第5位为1，那么这个1前面有27个零，对应的二进制为000...011011
Integer.numberOfLeadingZeros的方法如下：

```java
public static int numberOfLeadingZeros(int i) {
    // HD, Figure 5-6
    //如果值为零，那不用说，对于一个int类型的值来说它的前导领导的个数就是32个
    if (i == 0)
        return 32;
    int n = 1;
    //向右移位16位，如果为零，那么说明高16位全是零，并将i的低16位移动到高16位
    if (i >>> 16 == 0) { n += 16; i <<= 16; }
    //然后向右移动24位就是为了检查低16位的高8位是不是零。。。。。。
    if (i >>> 24 == 0) { n +=  8; i <<=  8; }
    if (i >>> 28 == 0) { n +=  4; i <<=  4; }
    if (i >>> 30 == 0) { n +=  2; i <<=  2; }
    n -= i >>> 31;
    return n;
}
```
对于上面这段代码，我们通过举例子的方式进行说明，比较好理解，其原理是这样的，比如我们有以下二进制<br />
0000000000000000 0000000000011011 <br />
当我们想要找到前导零的时候，我们可以向右移动16位，如果移动16位的值为0，那么说明前导零已经至少16个，那么我们可以忽略高16位了，直接圈定低16位，把低16移动到
高16位，将已经计算过的高16前导零直接左移舍弃掉，那么这个值就变成 <br />
0000000000011011 0000000000000000 <br />
为了方便理解，我们直接把不参与计算的bit位去掉（低16位），也就变成了 <br />
0000000000011011  <br />
此时我们向右移动24位，其实就是把00000000 00011011分成两半，高8位为零的，那么前导零的个数加8，加上前面的16个，已经有24个前导零了,然后向左移动8位丢弃
高8位的零，然后我们继续切分剩下的00011011，分成两半就是0001 1011，此时这里的高4经过右移后不再等于零，那么这里的高4位不能丢弃了（左移丢弃），
那么我们就切分这个高4位00 01 发现高两位为零，加上前面的24就有26个前导零了，继续切分01，有一个0，那么总共就是27个零，JDK采用二分法计算一个二进制的前导零个数
只不过这个二分是左边构成的值与一个零做比较而已。

计算好前导零的个数后，与上(1 << (RESIZE_STAMP_BITS - 1)，RESIZE_STAMP_BITS的值为16，将1左移15也就是2的15次方，第16位为1。
在进行扩容时，通过 rs = Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1)) 计算出来的值会向左移动RESIZE_STAMP_SHIFT位，
这个值为 32 - RESIZE_STAMP_BITS，我们知道RESIZE_STAMP_BITS为16，那么 RESIZE_STAMP_SHIFT 也位16，前面说到计算出的rs值的第16为1，那么向左移动16位，意味这将
会产生一个负数，最后通过原子操作赋值给了sizeCtl，低16位用于记录当前一同进行扩容的线程数，高16位做为校验数据，用于判断扩容是否结束

下面这段代码就是进行计算单元的cas操作

```java
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    //如果当前线程的probe为零，那么需要进行初始化
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        //内部使用AtomicLong获取值，所以每个线程对应的probe是不一样的
        ThreadLocalRandom.localInit();      // force initialization
        h = ThreadLocalRandom.getProbe();
        //对于这种线程没有进行随机初始化的，不认为是一次竞争失败
        wasUncontended = true;
    }
    //碰撞标记
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
        //计数单元数组中已经有其他线程创建值时进入分支，否则需要进行创建
        if ((as = counterCells) != null && (n = as.length) > 0) {
            //由于n为2的n次幂值，所以这是一个求余操作
            //对应索引下标的的值为空，那么就需要进行赋值
            if ((a = as[(n - 1) & h]) == null) {
                //cellsBusy用于标记CounterCell是否正在扩容或者添加元素操作
                if (cellsBusy == 0) {            // Try to attach new Cell
                    CounterCell r = new CounterCell(x); // Optimistic create
                    //通过cas进行操作cellsBusy，将其值设置为1，当为1的时候表示技术单元数组正在扩容或者添加元素
                    if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            //双重检查，因为在当前线程处理之前，有可能被其他线程处理了
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        //创建成功的，直接跳出循环，创建不成功，那么说明在当前线程处理之前，有可能被其他线程处理了
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            //wasUncontended在addCount方法，可能会因为多个线程同时cas同一个索引下的计数单元失败而置为false
            else if (!wasUncontended)       // CAS already known to fail
                //将其置为true，进入下次循环
                wasUncontended = true;      // Continue after rehash
                //cas操作计数单元的值，如果成功，那么直接break，不成功的，继续下一轮
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
                //counterCells != as在发生counterCells创建或者扩容时会为true
                //当前n扩容到大于系统cpu个数时，那么collide一直会被设置为false，也就是counterCells数组将不再被扩容
                //这主要为了性能考虑，如果太大，循环统计数据的时间复杂度将变成O(N)
            else if (counterCells != as || n >= NCPU)
                collide = false;            // At max size or stale
                //通常情况下，如果一个线程竞争了两次都失败了，那么counterCells数组的碰撞比较严重，需要进行扩容
            else if (!collide)
                collide = true;
                //cas操作
            else if (cellsBusy == 0 &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    //双重检查，因为在当前线程处理之前，有可能被其他线程处理了
                    if (counterCells == as) {// Expand table unless stale
                        //扩容为原来长度的2倍
                        CounterCell[] rs = new CounterCell[n << 1];
                        for (int i = 0; i < n; ++i)
                            //迁移值
                            rs[i] = as[i];
                        //重新赋值
                        counterCells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            //重新计算线程的probe值，尽量避免碰撞
            h = ThreadLocalRandom.advanceProbe(h);
        }
        //cas竞争，对计数单元数组进行初始化
        else if (cellsBusy == 0 && counterCells == as &&
                 U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
            boolean init = false;
            try {
                // 双重检查，因为在当前线程处理之前，有可能被其他线程处理了
                // Initialize table
                if (counterCells == as) {
                    //默认大小为2
                    CounterCell[] rs = new CounterCell[2];
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            //初始化完成，那么当前进行初始化的线程使命结束
            if (init)
                break;
        }
        //初始化竞争的失败的，尝试继续cas baseCount，成功的话，break
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
```

多个线程去更新自己索引上的计数单元，由于可能发生hash碰撞，所以他们都是使用的cas原子操作，操作失败的进行自旋，用于cas和无限循环构成了一个自旋锁
与全部线程都去cas更新baseCount对比，明显fullAddCount方法的效率会更高。

下面是统计元素个数的代码

```java
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```
很简单，就是循环计数单元数组，进行叠加，可能有人会问，这种方式计算出来的个数准确吗？它没加锁哎，要是有一个线程又添加了一个计数单元那不是不准了吗？
的确，这么计算的值是不准确，因为这是并发操作，在你获取元素个数的时候，别的线程就是会改变map的元素个数，不像线程不安全的HashMap，我们通常都是一个线程去
put值，然后一个线程去获取个数。

#### 1.4.4 扩容

第一步：hash表的扩容
```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    //n：原数组长度，stride：步长
    int n = tab.length, stride;
    //长度缩小8倍后对CPU个数取整，小于MIN_TRANSFER_STRIDE（16），那么步长被设置为16
    //所以最小步长为16，假设您的电脑是4核心的，那么这个n至少为512，才不会小于MIN_TRANSFER_STRIDE（16）
    //为什么最小的步长只能是16？如果切分的太细，线程一次性处理的范围太小，会影响到性能
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            //将hash表扩展为原来的2倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    //。。。。。。省略划分任务与迁移节点代码
}
```
上面一段代码有这么几个步骤：
- 确定步长，所谓步长是用来给线程划分任务范围的，为什么会有步长这种东西？因为作者在扩容的时候并不是直接一把锁锁住，他不仅不锁，还邀请其他线程一同去帮助扩容hash数组，所以定义出了步长来划分任务
- 将hash数组扩容为原来的两倍

第二步：给线程划分任务范围

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    
    //。。。。。。省略hash表扩容代码

    //扩容后的hash表长度
    int nextn = nextTab.length;
    //创建ForwardingNode节点，这个节点内部指向了新扩容出来的hash表
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    //用于标识是否需要继续给当前线程自己划分任务
    boolean advance = true;
    //当前线程的扩容操作是否完成
    boolean finishing = false; // to ensure sweep before committing nextTab
    //这里的i和bound将被用于圈定当前线程的任务范围，右边界为i，左边界为bound
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            //nextIndex：表示下一个任务范围的右边界，nextBound：表示下一个任务的左边界
            //(bound, i)
            int nextIndex, nextBound;
            //当前线程的任务范围右边界减去1并检查当前线程的右边界是否大于等于左边界（如果右边界已经小于左边界了，那么说这个范围的数据已经迁移完成了）
            if (--i >= bound || finishing)
                advance = false;
                //任务都已经划分出去了，没有任务可领了
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            //原子操作，更新可划分任务的右边界
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        //如果当前线程的扩容任务完成，并且没有可划分的任务了，那么进入此分支
        //i >= n || i + n >= nextn 这个判断没明白为啥要这么干，而且看代码不可能会出现这样的条件
        //更何况 i >= n 与 i + n >= nextn 是等价的
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                nextTable = null;
                //赋值为扩容后的hash表
                table = nextTab;
                //扩容后的0.75     2n - 2n >> 2 = 2n - (2n / 4) = n << 1 - n >> 1
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            //原子操作，将正在扩容的线程个数减去1，低16位表示正在参与扩容的线程个数
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // 迁移完毕之后再检查一遍
            }
        }
        //如果对应需要迁移的节点下标是null，那么无需迁移，直接cas设置为ForwardingNode，表示该位置的数据已被迁移并且也表示当前hash表正在扩容迁移
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        //如果当前节点就是ForwardingNode节点，那么说明正在扩容并且这个位置的节点已被迁移，标记advance为true，告诉当前线程去处理其他节点或者竞争新的任务范围
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
        //。。。。。。省略节点迁移代码
    
}
```


假设系统CPU个数为4核心，原始hash数组的长度为2048，那么其步长为256，将会划分以下范围的任务<br />

```java
[2048 - 256 = 1992, 2048)
[1992 - 256 = 1736, 1992)
[1736 - 256 = 1480, 1736)
            .
            .
            .
[256 - 256 = 0, 256)         
```
2048总共可划分成8个子任务，这8个子任务交由多线程去竞争获取。竞争到任务的线程以[bound, i)作为任务区间，倒序迁移节点，也就是--i迁移每个节点直到i &lt; bound，
做完自己的任务后，如果还有余下的任务可分配的将继续通过cas竞争任务范围，没有任务可分配了，那么更新正在扩容线程数量，并返回，如果当前线程为最后一个扩容线程了，那么
会做些额外的操作，比如重新赋值扩容阈值，用扩容后的hash表替换原来的hash表

第三步：迁移内容

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {

    。。。。。。省略hash表扩容与划分任务代码

    synchronized (f) {
        //双重检查当前节点是否已经更新，因为在当前线程处理之前，有可能被其他线程处理了
        if (tabAt(tab, i) == f) {
            //假设hash表扩容前的长度为n，那么ln就表示下标落在0-n的节点（也就是runBit = 0的），hn表示下标落在n-2n的节点（也就是runBit = 1的）
            Node<K,V> ln, hn;
            //是否为链表，某下标中头节点的hash值如果大于零为链表，小于零为红黑树（红黑树头节点类型为TreeBin）
            if (fh >= 0) {
                //将节点hash值与原始长度相与，我们知道n是一个2次幂的值，也就是说它的二进制bit位上只有一个1
                //前面我们进行求余计算时使用的是 hash & (n - 1)，假设我们长度为16，二进制为000...010000，减一后得到000...001111，这个值与任何hash值位与
                //它都是将这个hash值的低四位原封不动的给截取出来，当我们进行扩容后，hash表长度变为32，减一后得到000...011111，比原来多了一个1
                //任何hash值与扩容后的长度求余，都是截取低5位，所以如果一个hash值的第5位为1，那么它通过新容量求余出来的数将会比原容量求余算出来的值大n
                //如果这个hash值的第5位为零，那么通过新容量求余出来的值和原来容量求余出来的值是一样的
                int runBit = fh & n;
                Node<K,V> lastRun = f;
                //循环链表，找到最后一段连续相同索引的节点，最后这一段节点可以直接复用，无需new出新的Node对象
                for (Node<K,V> p = f.next; p != null; p = p.next) {
                    int b = p.hash & n;
                    if (b != runBit) {
                        runBit = b;
                        lastRun = p;
                    }
                }
                //runBit为0的节点用ln来维护（ln是索引未0-n的节点）
                //runBit为1的节点用hn来维护
                if (runBit == 0) {
                    ln = lastRun;
                    hn = null;
                }
                else {
                    hn = lastRun;
                    ln = null;
                }
                //循环一直到lastRun为止，lastRun后面的节点时一段连续相同索引位置的节点，可以直接复用，lastRun放在哪，后面的节点就会跟着放在哪
                for (Node<K,V> p = f; p != lastRun; p = p.next) {
                    int ph = p.hash; K pk = p.key; V pv = p.val;
                    if ((ph & n) == 0)
                        //使用头插法
                        ln = new Node<K,V>(ph, pk, pv, ln);
                    else
                        hn = new Node<K,V>(ph, pk, pv, hn);
                }
                //cas设置0-n索引位置的值
                setTabAt(nextTab, i, ln);
                //cas设置n-2n索引位置的值
                setTabAt(nextTab, i + n, hn);
                //将原hash表对应迁移走的下标设置为ForwardingNode节点，以便告诉下个操作到这个位置的线程，我正在扩容，你要不要一起
                setTabAt(tab, i, fwd);
                advance = true;
            }
            //如果是红黑树
            else if (f instanceof TreeBin) {
                TreeBin<K,V> t = (TreeBin<K,V>)f;
                TreeNode<K,V> lo = null, loTail = null;
                TreeNode<K,V> hi = null, hiTail = null;
                int lc = 0, hc = 0;
                //下面的操作和操作链表是一样的套路
                for (Node<K,V> e = t.first; e != null; e = e.next) {
                    int h = e.hash;
                    TreeNode<K,V> p = new TreeNode<K,V>
                        (h, e.key, e.val, null, null);
                    if ((h & n) == 0) {
                        if ((p.prev = loTail) == null)
                            lo = p;
                        else
                            loTail.next = p;
                        loTail = p;
                        ++lc;
                    }
                    else {
                        if ((p.prev = hiTail) == null)
                            hi = p;
                        else
                            hiTail.next = p;
                        hiTail = p;
                        ++hc;
                    }
                }
                //如果当前下标对应的节点个数已经小于退化红黑树的阈值，那么退化为链表
                ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                    (hc != 0) ? new TreeBin<K,V>(lo) : t;
                //如果当前下标对应的节点个数已经小于退化红黑树的阈值，那么退化为链表
                hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                    (lc != 0) ? new TreeBin<K,V>(hi) : t;
                //cas设置0-n索引位置的值
                setTabAt(nextTab, i, ln);
                //cas设置n-2n索引位置的值
                setTabAt(nextTab, i + n, hn);
                //将原hash表对应迁移走的下标设置为ForwardingNode节点，以便告诉下个操作到这个位置的线程，我正在扩容，你要不要一起
                setTabAt(tab, i, fwd);
                advance = true;
            }
        }
    }
}
```
作者迁移节点的时候没有再次通过 hash & (2n -1) 的方式计算下标，而是直接将hash值与原来的数组长度进行位与，等于零的新索引与旧索引一样，等于1的，新索引 = 旧索引 + 旧长度
那么其原理是什么呢？

假设原容量为16，某个hash值对其求余的方式为 hash & (n - 1) <br />
扩容前
```java
     xxxxxxxxxxx
&    0000..01111
-----------------
=    0000..0xxxx
```
扩容后

```java
     xxxxxxxxxxx
&    0000..11111
-----------------
=    0000..xxxxx
```
如果某个hash值的第5位为1，那么扩容前与扩容后的值分别为

```java
扩容前 0000..0xxxx
扩容后 0000..1xxxx

```
0000..1xxxx - 0000..0xxxx = 16

如果某个hash值的第5位为0，那么扩容前与扩容后的值分别为
```java
扩容前 0000..0xxxx
扩容后 0000..0xxxx

```
扩容前与扩容后相等

#### 1.4.5 帮助扩容

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    //如果发现当前要put的节点被ForwardingNode占用，那么说明hash表正在扩容
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        //计算扩容校验位（stamp，邮戳）
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
               //是否已经扩容完成或者已达到扩容参与线程数上限或者已无任务范围可划分
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
                //cas递增参与扩容线程数
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                //扩容
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```
帮助扩容的代码逻辑和1.4.3小节中的计数方法逻辑里的扩容是一样的，这里就不过多的说明了

#### 1.4.6 数据结构变换触发条件

```java
if (binCount != 0) {
    //某数组下标对应的节点个数是否达到了红黑树阈值
    if (binCount >= TREEIFY_THRESHOLD)
        //达到后进入treeifyBin方法，可能进行红黑树转换
        treeifyBin(tab, i);
        //其他情况，如果只是替换了旧值，那么直接返回旧值，不做计数和扩容判断
    if (oldVal != null)
        return oldVal;
    break;
}

private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        //除了判断某个数组下标的节点数达到了红黑树阈值之外，还会判断当前hash数组的长度是否已经超过了64
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            //没有超过64，会尝试通过扩容来解决，将节点重新hash
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            //节点锁，锁粒度已经小到某个hash数组元素了
            synchronized (b) {
                //双重检查，因为在当前线程处理之前，有可能被其他线程处理了
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        //创建一个树节点，在构造器中创建一颗红黑树
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        //虽然创建了红黑树，但是为了扩容的读取方便，依然会构建一个链表
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    //将对应的下标位置设置为TreeBin，TreeBin的hash值为-2，引用的firstTreeNode为头节点
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```
虽然某个下标位置的节点个数已经超过红黑树的阈值，但是并不是就意味就要进行树化，如果hash数组的长度小于64，那么会尝试进行扩容，如果超过64，那么就进行树化

创建红黑树

```java
TreeBin(TreeNode<K,V> b) {
    super(TREEBIN, null, null, null);
    //设置根节点
    this.first = b;
    TreeNode<K,V> r = null;
    for (TreeNode<K,V> x = b, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        //将传递进来的节点设置为根节点
        if (r == null) {
            x.parent = null;
            //根据红黑树的特性，根节点是黑色的
            x.red = false;
            r = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            //用于维护实现了Comparable节点class类型
            Class<?> kc = null;
            for (TreeNode<K,V> p = r;;) {
                //dir表示方向：-1表示放到左节点，1表示放在右节点，ph：表示父hash值
                int dir, ph;
                //父key
                K pk = p.key;
                //首先通过hash来比较，小于父hash放在父节点的left
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                            //获取实现了Comparable的class类型，如果没有实现Comparable并且对应的比较的泛型类型Comparable<T>的T不相匹配，那么会
                            //返回null
                          (kc = comparableClassFor(k)) == null) ||
                          //调用Comparable的compareTo进行比较
                         (dir = compareComparables(kc, k, pk)) == 0)
                    //处理没有实现Comparable接口或者对比后相等的元素
                    //比较其类名，如果类名也相等的，那么通过System.identityHashCode方法比较
                    dir = tieBreakOrder(k, pk);
                    TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    //设置父节点
                    x.parent = xp;
                    if (dir <= 0)
                        //左节点
                        xp.left = x;
                    else
                        //右节点
                        xp.right = x;
                    //进行红黑树的左旋右旋，保持平衡
                    r = balanceInsertion(r, x);
                    break;
                }
            }
        }
    }
    //设置根节点
    this.root = r;
    assert checkInvariants(root);
}
```
关于红黑树的内容将会在TreeMap分析中讲解。

### 1.5 总结

- 构造
    - 当我们构建一个ConcurrentHashMap的时候，如果我们传入了一个初始容量a，那么首先需要对这个初始容量做些调整，如果是大于2的29次方，那么返回最大容量2的30次方，
      如果小于2的29次方，那么先扩容1.5倍，然后再调整为最接近于 1.5a 的二次幂的值。为什么这么做呢？当ConcurrentHashMap中的节点个数达到容量的0.75之后需要进行扩容操作，
      所以实际上可以使用的容量（用户肯定希望自己设置的容量就是实际可用的容量）是某个更大容量的0.75。按道理我们也是要扩容4/3倍才对，为什么这里扩容1.5呢？扩容1.5倍
      可以使用位运算提高计算机的运算速度，a = a + a >> 1。扩容后，需要将容量转换成最近接1.5a的一个2次幂的值，
      其转换原理就是将某个值从最高位为1那个bit位开始往下所有
      bit位都变成1，最后加1就得到最接近1.5a的一个2次幂的值，但由于不知道这个最高位为1的bit位具体在某个位置，所以就按第31为1来位移，总共移位31次。

- 添加元素
    - 插入元素时，会判断是否已经初始化，如果未初始化会将sizeCtl cas为-1，表示正在初始化，其他线程看到ConcurrentHashMap未初始化并且sizeCtl为负数时进行自旋，并调用
      Thread.yield()让出cpu
    - 用计算出的下标，判断当前数组元素是否已被占用，如果没有，创建节点cas替换，如果有值并且hash值不小于0，将使用synchronized进行锁定，如果是链表结构的，循环链表
      ，查找是否是已经存在的key值，是就判断是否替换，不是就创建新的节点追加，如果是红黑树结构的，查找是否是已经存在的key值，是就判断是否替换，不是就创建新的节点插
      入（如
      果影响了红黑树的平衡将会进行左旋右旋）。接下来会判断链表节点个数是否大于等于8个，如果成立，那么再判断当前ConcurrentHashMap的容量是否大于等于64，如果大于这个
      值就会将链表进行树化。最后还会计算ConcurrentHashMap节点个数。
    - 用计算出的下标，判断当前数组元素是否已被占用，如果有并且hash值为-1，那么表示ConcurrentHashMap正在扩容，当前线程需要判断是否需要帮助扩容。在参与扩容线程数
      没有超过最大限制（扩容时sizeCtl的低16位表示参与扩容线程数）或者扩容还未结束，任务未完全划分出去，那么当前线程需要参与扩容。当前线程进入扩容逻辑，首先需要
      根据步长去竞争还未划分出去的任务，竞争成功后，将指定范围的节点迁移，迁移完后继续竞争下一个任务，如果没有任务了，那么递减sizeCtl上表示正在参与扩容的线程数
- 计数
    - 首先使用baseCount（它是AtomicLong类型）去计数，如果计数失败，说明此时出现了竞争，那么会初始化一个
      CounterCell数组，这个CounterCell数组的容量也是2次幂的值，我们获取每个线程的探针值（这个探针值是通过AtomicLong进行递增得到的，不会重复），然后对数组长度求余
      得到一个下标，在这个下标上创建CounterCell元素进行cas计数，如果连续两次计数失败，那么说明hash冲突比较严重，需要扩容，最大扩容的容量不超过CPU的个数。最后我
      ConcurrentHashMap的节点个数就是baseCount加上CounterCell数组数组里面每个CounterCell的计数值。计算出节点个数之后，会判断当前节点个数是否已经超过了容量的0.75，
      如果超过了，那么将会进行扩容操作。
- 扩容
    - 计算旧容量的前导零的个数（容量为2次幂的值，不同的容量其前导零的个数一定不同）然后与上 1 << 15，作为校验值，左移16位，低16位用于记录正在参与扩容线程的个数。
      创建一个容量为原来2倍的数组，按照步长（根据cpu进行计算 (NCPU > 1) ? (n >>> 3) / NCPU : n) < 16，最小步骤为16，步长太小，对效率产生影响）将旧数组从后往前
      划分任务，然后让线程去竞争任务（将每个任务块看成一个元素的话，就像是从一个同步栈取元素）。竞争到任务之后，线程需要迁移任务范围内的节点，迁移的时候也是一门学
      问，作者并未重新用hash值去计算下标，而是用节点的hash值与原始容量进行位运算，如果得到的值为1，那么其下标就是原来下标值加上旧数组长度，如果为0，那么下标不变。
- 取值
    - 计算下标，到对应节点中获取数据，如果发现对应下标的节点hash小于零，那么调用当前节点的find方法，如果是正在扩容使用的ForwardingNode，那么其内部维护了一个扩容
      后的数组，元素值将会从这个新数组中查找，所以这也就解释了为什么取值时不需要加锁的原因。