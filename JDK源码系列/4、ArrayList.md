## 一、简介

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/7533fa5ffb9312da5c73db1580a0fa4f.png)

ArrayList顾名思义，数组列表，它是由数组实现的一个List，下面是它的一些成员变量

```java
//默认的初始化容量
private static final int DEFAULT_CAPACITY = 10;

//空数组，当用户指定容量为零时使用
private static final Object[] EMPTY_ELEMENTDATA = {};

//默认容量大小的空数组，当用户没有指定容量时使用
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

//用于存放我们添加的元素的数组
transient Object[] elementData;

//List的大小
private int size;
```
构造器

```java
//指定容量的构造器
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        //使用指定的容量创建一个数组
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        //如果指定容量为零，那么赋值一个空数组
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}


public ArrayList() {
    //没有指定容量时，赋值默认容量大小的空数组
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}


public ArrayList(Collection<? extends E> c) {
    //将传入的集合转换成数组，然后赋值，转换构成是基于数组的拷贝
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        //判断传入的集合数组类型是不是Object数组类型，如果不是，将其内容进行拷贝到Object数组，为什么？
        //假设当前我们的List的元素类型是Number，而传入的是一个Integer集合，如果让它直接赋值个elementData，那么下次我这个集合想要添加一个Long类型的
        //元素就会抛出错误，我们明明使用的是一个Number类型的List，应当可以添加任何Number类型的子类类型的元素，所以此处需要将其拷贝成Object数组
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        //如果传入的是空集合，那么将空数组赋值
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```
## 二、ArrayList基本操作

### 2.1 添加元素

```java
public boolean add(E e) {
    //确保容量足够
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //size自增，将元素添加进去
    elementData[size++] = e;
    return true;
}
                |
                |
                V
private void ensureCapacityInternal(int minCapacity) {
    //检查是否已经初始化
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        //如果没有，确定容量大小，如果传入的容量大小小于默认的容量，那么使用默认容量，默认容量是10
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    
    ensureExplicitCapacity(minCapacity);
}
                |
                |
                V
private void ensureExplicitCapacity(int minCapacity) {
        //修改计数
        modCount++;

        // overflow-conscious code
        //如果数组容量不够了，那么需要扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

```
扩容

```java
private void grow(int minCapacity) {
    // overflow-conscious code
    //老的容量大小
    int oldCapacity = elementData.length;
    //oldCapacity * 1.5
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    //如果老容量的1.5倍依然不够
    if (newCapacity - minCapacity < 0)
        //那么直接将传入的容量作为list的新容量
        newCapacity = minCapacity;
    //新容量大于定义预期的最大容量
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        //对容量做调整，将容量设置为Integer.MAX_VALUE
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    //进行数组的拷贝
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```
可以看到，ArrayList进行扩容的时候默认是以1.5倍进行扩容的

### 2.2 元素的拷贝

```java
//original：拷贝源数组，newLength：新数组的长度，newType：新数组的类型
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
    @SuppressWarnings("unchecked")
    //如果指定的新数组的类型为Object[],那么直接构建Object数组，其他的进行反射创建数组
    //最后将源数组中的元素拷贝到新数组
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}
```

### 2.3 元素的删除

指定下标删除元素

```java
public E remove(int index) {
    //检查指定下标是否越界，内部直接 index >= size，如果为true便抛出IndexOutOfBoundsException
    rangeCheck(index);
    
    //修改计数加1
    modCount++;
    //获取指定下标的元素
    E oldValue = elementData(index);
    
    //计算从index后面需要移动的元素有多少个
    int numMoved = size - index - 1;
    if (numMoved > 0)
        //从index+1开始，拷贝numMoved个元素
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    //将最后一个位置置空
    elementData[--size] = null; // clear to let GC do its work
    //返回被删除的元素
    return oldValue;
}
```
删除指定元素

```java
public boolean remove(Object o) {
    //如果指定元素值为null，那么循环找到第一个为null的元素删除
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                //指定下标删除
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            //寻找到对应的对象，然后指定下标删除
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
```

### 2.4 SubList

SubList是ArrayList的内部类，它跟ArrayList一样继承了AbstractList

```java
public List<E> subList(int fromIndex, int toIndex) {
    //下标越界检查
    subListRangeCheck(fromIndex, toIndex, size);
    return new SubList(this, 0, fromIndex, toIndex);
}

//SubList构造器
SubList(AbstractList<E> parent,
                int offset, int fromIndex, int toIndex) {
    //表示ArrayList
    this.parent = parent;
    //相对于父List的偏移
    this.parentOffset = fromIndex;
    //如果用户指定了offset，那么对SubList的操作都会在offset + fromIndex下标位置之后
    this.offset = offset + fromIndex;
    //圈定范围大小
    this.size = toIndex - fromIndex;
    //记录修改次数，下次SubList发生修改时会和父List的modCount进行核对，如果对不上就抛出错误
    this.modCount = ArrayList.this.modCount;
}
```
SubList的set方法

```java
public E set(int index, E e) {
    //检查当前指定的下标是否已经越出SubList圈定的范围
    rangeCheck(index);
    //用SubList记录的父Lsit上次的modCount与当前set操作时的父List的modCount是否相同，如果不同将抛出ConcurrentModificationException异常
    checkForComodification();
    //替换掉原来旧的元素
    E oldValue = ArrayList.this.elementData(offset + index);
    ArrayList.this.elementData[offset + index] = e;
    return oldValue;
}
```
SubList的add方法

```java
public void add(int index, E e) {
    //检查当前下标是否越界
    rangeCheckForAdd(index);
    //用SubList记录的父Lsit上次的modCount与当前set操作时的父List的modCount是否相同，如果不同将抛出ConcurrentModificationException异常
    checkForComodification();
    //调用父List的add方法
    parent.add(parentOffset + index, e);
    //更新新的修改计数
    this.modCount = parent.modCount;
    this.size++;
}

//父ArrayList的add方法
public void add(int index, E element) {
    //检查下标是否越界
    rangeCheckForAdd(index);
    
    //确保容量
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //将原来的index下标后的元素拷贝到index+1后
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    //将空出来的index重新赋值为指定元素值
    elementData[index] = element;
    size++;
}
```

## 三、总结

&nbsp;&nbsp;&nbsp;&nbsp;ArrayList相对于Map系列的集合更为简单，其使用的数据结构就是一个数组，与之类似的List还有LinkedList和Vector
- Vector和ArrayList一样，使用了数组，只不过它是线程安全的，它的方法直接使用synchronized修饰
- LinkedList使用的结构为链表