## 一、基础

### 1.1 类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d7b941fa1370487b37dba61c6c644d75.png)

- Iterable：可迭代的，声明实现此接口的类具有迭代元素的能力，在JDK8之后增加了默认方法forEach与spliterator，forEach用于遍历元素，spliterator用于分割迭代器，通常
  用于并行流
- Collection：定义一些集合操作的基本方法，比如add元素，删除元素，包含，交集等等，还有JDK8的流
- AbstractCollection：模板类，实现了基本的方法，比如remove，contains等方法
- Queue：定义基本的队列操作方法，add，offer，poll，peek等
- AbstractQueue：模板类，实现了Queue接口的一些基本操作方法
- Itr：PriorityQueue的内部类，实现了Iterator接口，是专属于PriorityQueue的迭代器
- PriorityQueueSpliterator：用于优先队列的分割迭代器，优先队列的数据结构类似一颗二叉树，但是他们某一分支上的节点并不是一直都是大于前一个节点或者小于一个节点
  具体我们将在后面分析

### 1.2 字段说明

```java
//初始容量
private static final int DEFAULT_INITIAL_CAPACITY = 11;

//使用数组实现的队列
transient Object[] queue;

//队列元素的个数
private int size = 0;

//比较器，用于决定两个元素的优先级
private final Comparator<? super E> comparator;

//队列修改次数
transient int modCount = 0; 

//queue数组的最大容量，这里减去8是因为数组作为一个对象，它在JVM中存在对象头，对象头会消耗一部分容量
//对于数组的话，由于无法直接确定数组的大小，所以会额外使用4个字节来储存数组长度，说起来数组的大小为啥不能用long表示呢？也许这也是一个原因
//系统限制了
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

```

### 1.3 构造器

```java
public PriorityQueue(int initialCapacity,
                         Comparator<? super E> comparator) {
    // Note: This restriction of at least one is not actually needed,
    // but continues for 1.5 compatibility
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.queue = new Object[initialCapacity];
    this.comparator = comparator;
}
```

### 1.4 添加元素

```java
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    //修改次数加1
    modCount++;
    int i = size;
    //如果原始长度大于64，那么按照1.5倍增长
    //如果小于64的，2倍增长
    if (i >= queue.length)
        grow(i + 1);
    //元素个数加1
    size = i + 1;
    if (i == 0)
        //为1直接赋值
        queue[0] = e;
    else
        //进行堆排序
        siftUp(i, e);
    return true;
}
```

小顶堆（比较时，数值小的优先级高，如果以数值大优先级更高，那么这种被称为大顶堆）排序

```java
private void siftUpComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>) x;
    while (k > 0) {
        //计算父元素的下标
        int parent = (k - 1) >>> 1;
        Object e = queue[parent];
        //如果添加的元素值小于父元素的值，那么需要替换位置
        if (key.compareTo((E) e) >= 0)
            break;
        queue[k] = e;
        k = parent;
    }
    queue[k] = key;
}
```
刚看到这段代码的人，也许会感到一脸懵逼，为什么和我们自己预想的不一样，优先队列嘛，不就在添加元素的时候和前面的元素比较一下，谁优先级更高谁在前面不就完事了吗？如果我们以这种方式进行比较的话，注定这是一段耗费性能的烂代码

其实上面这段代码使用数组实现一个小顶堆（类似一颗二叉树），假设现在我们有一组数字 -》 9，11,10,133,144，20，其实现的小顶堆如下：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/3ddf05dd8ab92c2caf10ada886b3e23c.png)

对应的数组平铺的数据顺序为从上到下，从左到右 -》 9,11,10,133,144,20

此时如果在插入数 8，那么小顶堆演化的过程如下：

第一步：

当添加一个节点的时候，先找到它所属的父节点，这里的父节点就是10，然后将插入节点与父节点进行比较，如果插入节点比父节点小，那么互换位置
（当然在代码中并不会立马用插入节点的值去替换原先父节点的值，而是继续找父节点的父节点进行比较）

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ed6fec7ef6df3bf6399a4d3c83d72f45.png)

第二步：

然后继续向上比较，直到根节点即可

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/f262324d0cedec26f72785e0920728b9.png)

> Q：为什么使用数组去实现堆呢？不能定义个Node类去实现吗？

> A：数组是基于下标来查找数据的，在删除数据和添加数据的时候可以快速进行定位替换，副作用就是扩容的时候需要拷贝数据

> Q：为什么使用 (k - 1) >>> 1 （k为元素的添加元素的下标）就能找到父元素？

> A：假设我们有以下节点（图形上的标号表示的是在数组中的下标）

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d0e612aed303dfafbe160929773fe204.png)

首先我们先来证明为什么 parentLabel = (k - 1) >>> 1 <br />
这个小顶堆和二叉树很像，但是与二叉树不同的是兄弟节点的大小是没有排序的，在小顶堆中子节点
一定是比父节点大的。

对于一颗完整的二叉树，它的节点个数为 2的(n - 1)次方 （n为层数），如果我要找到标号为 2的(n - 2) 的节点的父节点，怎么找？ <br />
很明显标号为 2的n - 2的节点的父节点肯定是上一层中的节点并且是最后一个节点，其对应的标号为 2^(n - 1) - 2，
那么  2的(n - 1)次方 减去 2 会等于 (k - 1) / 2 = ((2^n - 2) - 1) >>> 1 吗？

```
先按照数学公式来计算

((2^n - 2) - 1) / 2 = 2^n /2 - 2/2 - 0.5 = 2^(n - 1) - 1 - 1 + 0.5 = 2^(n - 1) - 2 + 0.5

但是我们的代码((2^n - 2) - 1) / 2在java中是在取整，也就是说小数位将会被去掉，那么表达式 2^(n - 1) - 2 + 0.5 最终的值为 2^(n - 1) - 2
0.5使用二进制表示就是 0.1000，取整将不会考虑小数点后的值，对于int值的计算，计算机直接将移动到小数位的bit位丢弃

```

上面只是取了个比较特殊的例子进行证明，那如果我随便指定一个标号，比如  2^n - 2 - x ( 0 <= x < 2^(n - 1) )，2^(n - 1)为第n层的节点个数，有人会问了，为啥要
给x限定范围啊，我不能大于2^(n - 1)吗？如果你想计算其他层的节点标号，当前这个公式已经完全可以胜任，你改变n的值就可以算出任何一个节点标号。

```
既然 0 <= x < 2^(n - 1)，那么我们假设在第n层我们有一个节点叫做A，它的父节点为 第 n - 1 层的节点B，这个B与第n-1层最后一个节点隔了 y 个节点，那么有

结论1：B的标号为 2^(n - 1) - 2 - y

结论2：
x 为偶数时，也就是x为2的幂次方的值 x = y << 2;
x 为奇数时，也就是x为2的幂次方的值 x = y << 2 + 1;

证明 (k - 1) >>> 1：

将y代入公式

case 1：x为偶数时，2^n - 2 - y * 2，下面我们来证明 (k - 1) /2 == ((2^n - 2 - y * 2) - 1) / 2 == 2^(n - 1) - 2 - y

((2^n - 2 - y * 2) - 1) / 2 = 2^(n - 1) - 1 - y - 0.5 = 2^(n - 1) - 1 - y - 1 + 0.5 = 2^(n - 1) - 2 - y + 0.5

对表达式2^(n - 1) - 2 - y + 0.5取整后得到2^(n - 1) - 2 - y 左边等于右边，等式成立

case 2：x为奇数时，2^n - 2 - y * 2 - 1，下面我们来证明 (k - 1) /2 == ((2^n - 2 - y * 2 - 1) - 1) / 2 == 2^(n - 1) - 2 - y

((2^n - 2 - y * 2 - 1) - 1) / 2 = 2^(n - 1) - 1 - y - 0.5 - 0.5 = 2^(n - 1) - 1 - y - 1 = 2^(n - 1) - 2 - y

左边等于右边，上面两个case都证明了 某节点的父节点的标号等于这个某节点的标号对2取整

```

总结：PriorityQueue使用数组实现了一个类似二叉树的结构，默认是小顶堆，其特点是，从某个节点开始，其后的节点一定大于这个节点，兄弟之间没有大小排序。

### 1.5 获取与删除元素

```java
public E poll() {
    if (size == 0)
        return null;
    int s = --size;
    modCount++;
    //第一个即为优先级最高的元素
    E result = (E) queue[0];
    //取最后一个元素往前挪
    E x = (E) queue[s];
    queue[s] = null;
    if (s != 0)
        siftDown(0, x);
    return result;
}

```

下面便是移除第一个节点后的处理过程中的代码

```java
private void siftDownComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>)x;
    //找到整颗树中第一个没有子节点的元素
    int half = size >>> 1;        // loop while a non-leaf
    while (k < half) {
        //找出子节点
        int child = (k << 1) + 1; // assume left child is least
        Object c = queue[child];
        //找出右子节点
        int right = child + 1;
        if (right < size &&
            ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
            c = queue[child = right];
        //将最后一个节点与要删除节点的两个字节点中最小的那个比较，如果最后一个节点比这两个子节点小的话就可以成为它们的父节点
        if (key.compareTo((E) c) <= 0)
            break;
        queue[k] = c;
        k = child;
    }
    queue[k] = key;
}
```
下面以图形化的方式来阐明这颗小顶堆的演变过程，假设原始小顶堆为下图的模样

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/93a9f6811c16a9f7b519174784494e17.png)

此时我将poll第一个节点，那么根节点将被移除，根节点的下标为0，我们知道计算某下标为k的节点的父节点下标的公式为 (k - 1) >>> 1，现在我们已知父节点的下标，自然
就可以计算出左子节点的下标，为什么是左子节点的下标而不是右子节点的下标？<br />

```
证明：

根据1.4小节添加元素中的推导过程可知，如果是右子节点，那么标号公式 2^n - 2 - x 中的x一定是一个2次幂的值，计算其父节点的下标时公式为 
(k - 1) / 2 == ((2^n - 2 - x) - 1) / 2 == ((2^n - 2 - y * 2) - 1) / 2 == 2^(n - 1) - 2 - y + 0.5，它是有余数的。

取整后其得出其父节点下标 2^(n - 1) - 2 - y，如果用这个值进行反推子节点下标

parentIndex * 2 + 1 == (2^(n - 1) - 2 - y) * 2 + 1 = 2^n -4 - y * 2 + 1 == 2^n -4 - (y * 2 - 1) = 2^n - 3 - x

右子节点的标号为 2^n - 2 - x，其左子节点的标号为 2^n - 2 - x - 1 = 2^n - 3 - x

```

第一步：为了保持二叉树结构，我们将最后一个节点往前补充，首先先判断最后一个节点是否小于移除节点的两个子节点，如果小于直接变成两个子节点的父节点即可，如果
两个子节点中值有更小的，那么将这个更小的节点上移，得到

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/7f5bd84d4cb6df9fc401d5a274a0dedd.png)


第二步：将500这个节点与之前上挪的节点9（红色）的两个子节点对比，如果这个500比133,144都小，那么直接变成133,144的父节点，如果子节点中的最小节点133比500还小，
那么将133这个节点上挪，500替换原来的133位置

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/61f0b9f0e272b9d86ed6fb1b9fdc30ab.png)


其实讲了poll方法之后，我们可以推导出删除非根节点是怎么一回事了，比如还是以下小顶堆

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/dbe8394a4dd02e14d181d0cbd965b4c9.png)

现在我要删除节点9，那么我们将最后一个节点与删除节点子节点进行比较，也就是按照siftDown方法走一遍，如果最后一个节点直接替换了删除节点，那么还需要按照添加
元素时的siftUp走一遍，这样就保证了小顶堆的特性

> Q：int half = size >>> 1，下标为half的节点为什么是整颗二叉树中的第一个没有任何子节点的节点？

> A：我们知道某下标节点的父节点为 (k - 1) >>> 1，那么最后一个节点的父节点的下一个节点肯定是没有子节点的节点，比如

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/bc4db532e1950c9e0080c1091b16e506.png)

这里的最后一个节点是500，500这个节点下一个节点133是没有子节点的，那么找到整颗二叉树第一个没有子节点的下标的计算方式为


```

元素个数为 size = 2^n - 1

对于二叉树，我们从树的特性上可以直接分析得到最后一个没有子节点的节点的可能是最后一层的第一个节点，也可能是上一层除了第一个节点的其他节点，假设倒数第二层的第一个
无子节点的节点与最后一层的第一个节点相隔y个节点

继续沿用上面的公式 2^n - 2 - x ( 0 <= x < 2^(n - 1) )那么有

结论1：第一个无子节点的标号为 2^(n - 1) - 1 - y

结论2：
x为偶数时，也就是x为2的幂次方的值 x = y << 2;
x为奇数时，也就是x为2的幂次方的值 x = y << 2 + 1;

证明 int half = size >>> 1 算得的值为二叉树中第一个没有子节点的节点

int half = size >>> 1 = (2^n - 1 - x) / 2 = 2^(n - 1) - 0.5 - x/2

case1：x为偶数时有

int half = size >>> 1 = (2^n - 1 - x) / 2 = 2^(n - 1) - 0.5 - （y * 2）/2 = 2^(n - 1) - 0.5 - y = 2^(n - 1) - 1 + 0.5 - y

取整 2^(n - 1) - 1 - y

case2：x为奇数时有

int half = size >>> 1 = (2^n - 1 - x) / 2 = 2^(n - 1) - 0.5 - （y * 2 + 1）/2 = 2^(n - 1) - 0.5 - y - 0.5 = 2^(n - 1) - 1 - y

取整 2^(n - 1) - 1 - y

所以当y等于0时，对应在数组的下标就是最后一层的第一个节点
y等于其他值时就是倒数第二层中的节点


```

## 二、总结

作者Doug Lea使用小顶堆或者大顶堆实现了优先队列，插入数据时，先通过当前要插入的下标k计算出父节点所在的下标，计算方式为 (k - 1) >>> 1，然后与父节点进行比较，如果
父节点更大，那么将父节点挪到k这个位置，插入的值先不替换原先的父节点位置，因为它还需要和祖父节点比较，以此类推。

删除节点时，删除的是第一个节点，让最后一个节点F往前补充，首先与删除节点的两个子节点比较，如果比两个子节点最小的节点还小，那么直接替换删除节点，如果比其中一个节点
大，那么让更小的子节点去替换删除节点，我们给这个更小的子节点取个名字叫minC，此时最后一个节点F替换了minC，如果minC也有子节点，那么需要继续和minC的子节点进行比较，
比较方式重复删除节点的步骤。