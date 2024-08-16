## 一、简介

ThreadLocal可用于解决多线程并发的问题，其原理是每个线程都有一个代表其自身的Thread对象，每个Thread对象内部都有一个ThreadLocalMap字段，这个ThreadLocalMap是用于保存
数据的容器。另一方面ThreadLocal也可以解决跨层跨方法传通用值的问题。

## 二、字段说明

```java
//这个值等于下面两个字段相加的结果 nextHashCode.getAndAdd(HASH_INCREMENT)
//用于决定ThreadLocal对象的hash值
private final int threadLocalHashCode = nextHashCode();

private static AtomicInteger nextHashCode =
    new AtomicInteger();

private static final int HASH_INCREMENT = 0x61c88647;
```

## 三、ThreadLocalMap

### 3.1 成员变量

```java

//键值对继承弱引用，弱引用在触发垃圾回收的时候如果被扫描到就会被回收掉
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    //表示键值对的值
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}

//2的n次幂的值
private static final int INITIAL_CAPACITY = 16;

//用于储存键值对的数组
private Entry[] table;

//表示当前保存的键值对的个数
private int size = 0;

//发生rehash的阈值，是容量的2/3
private int threshold; // Default to 0
```
ThreadLocalMap并没有实现集合中的Map接口，它内部使用一个2的n次幂大小的数组去储存键值对，键值对使用ThreadLocalMap的内部类Entry表示，Entry是一个继承了WeakReference
的类，在发生GC的时候其应用的key如果被扫描到那将会被回收。


## 四、set方法

```java
private void java.lang.ThreadLocal.ThreadLocalMap#set(ThreadLocal<?> key, Object value) {

    
    Entry[] tab = table;
    //键值对数组长度
    int len = tab.length;
    //计算数组下标
    int i = key.threadLocalHashCode & (len-1);

    //如果已经被其他的ThreadLocal占用，那么往后顺延寻找其他的坑
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        //如果对应ThreadLocal的键值对已经存在，那么替换value值
        if (k == key) {
            e.value = value;
            return;
        }
        
        if (k == null) {
            //替换掉那些脏数据，也就是已经被回收的键值对
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    //创建键值对并插入到对应的位置
    tab[i] = new Entry(key, value);
    //递增键值对个数
    int sz = ++size;
    //如果没有扫描到需要清理的坑位并且键值对个数已经超过了容量的2/3，那么需要进行脏数据的清理，如果容量已经已经大于容量的1/2，那么需要进行扩容
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
首先通过hash计算出对应的数组下标i，然后检查对应下标是否已经存在值，如果存在的key相同，那么替换掉value，如果不同，那么从i开始往后找为null的坑位，在寻找的过程中如
果发现了被回收的脏数据，那么可以将当前需要插入的数据替换掉这个脏数据所占据的坑位。最后检查容量，如果键值对个数超过容量的2/3那么进行脏数据的清理，如果超过容量的
1/2那么进行容量的扩容（2倍扩容）。

## 五、替换过期数据方法

```java
//key表示需要插入的键值对的键
//value表示需要插入的键值对的值
//staleSlot表示被回收的坑位下标
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    //需要清理的坑位
    int slotToExpunge = staleSlot;
    //从当前需要清理的坑位开始往前寻找需要清理的更靠前的坑位
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
         //记录需要清理的新坑位
        if (e.get() == null)
            slotToExpunge = i;

    // Find either the key or trailing null slot of run, whichever
    // occurs first
    //从被回收坑位往下
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        //如果找到相同key的entry，替换值
        if (k == key) {
            e.value = value;
            //因为tab[i]位置的值会替换到tab[staleSlot]中，所以这个位置的值变成了无效的
            tab[i] = tab[staleSlot];
            //替换掉无效的坑位
            tab[staleSlot] = e;

            // Start expunge at preceding stale entry if it exists
            //如果在staleSlot之前的坑位没有其他的无效数据，那么刚才被 tab[i] = tab[staleSlot] 的坑位i被设置为需要清理的坑位
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            //expungeStaleEntry方法用于清理坑位并为某些键值对重新hash，返回
            //cleanSomeSlots方法用于继续往后扫描log2N次，如果发现一个需要清理的坑位，那么重新扫描log2N次
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        //如果往后寻找为null的坑位的过程中，出现了需要清理的坑位并且staleSlot之前的坑位没有需要清理的坑位，那么将当前坑位i设置为新的清理坑位
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // If key not found, put new entry in stale slot
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // If there are any other stale entries in run, expunge them
    if (slotToExpunge != staleSlot)
        //expungeStaleEntry方法用于清理坑位并为某些键值对重新hash，返回
        //cleanSomeSlots方法用于继续往后扫描log2N次，如果发现一个需要清理的坑位，那么重新扫描log2N次
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```
这个方法包含以下步骤：

1. 在寻找插入坑位的过程中，如果发现了脏数据坑位staleSlot，那么首先会从当前脏数据坑位staleSlot开始往前查找更早的脏数据坑位，以便后面从更早的坑位开始往后进行清理和
   rehash。
2. 从上次的脏数据坑位staleSlot开始往后找不为null并且key相同的键值对坑位i
    - 如果找到，那么替换value值，然后将键值对替换掉脏数据坑位staleSlot的值，而被替换的坑位i将设置成脏数据坑位，如果这个i是更早的（第一步没有发现更早的脏坑位）脏
      数据坑位，那么用slotToExpunge记录，记录完更早的脏数据坑位后开始从slotToExpunge开始往后清理脏数据坑位（置空），如果遇到有效的数据坑位，那么rehash，从rehash后
      的位置往后寻找为null的坑位插入。
    - 如果没有找到相同key的键值对并且在这过程中发现了更早的脏数据坑位，那么使用slotToExpunge记录。
3. 这一步是在第二步没有找到相同key的前提下的
    - 此时会替换掉staleSlot脏数据坑位。
    - 清理脏数据坑位，清理完后继续扫描至少log2(tab.length)次方，用于试探性的检查是否还有需要清理的脏数据坑位，如果发现了新的脏数据，那么重新至少
      扫描log2(tab.length)次方，扫描log2(tab.length)次是基于时间复杂度来考虑的。

## 五、清理脏数据坑位

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    //清理掉脏数据坑位
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    //size减一
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    //从开始需要清理的坑位开始往后循环知道entry为null
    //如果遇到脏数据，将坑位置空，size减一
    //遇到有效数据，需要重新rehash，然后从rehash出的位置继续往后寻找新的为空的坑位，然后插入进去
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}

```

首先将确定的脏数据坑位置空，然后从当前脏数据坑位开始往后，只要不为空，那么就去判断是否为脏数据坑位，如果是脏数据坑位置空，如果是有效数据位，那么重新rehash，重新
选择一个为null的坑位插入。

## 六、rehash和resize

### 6.1 rehash

```java
private void rehash() {

    //清理脏数据坑位，从0开始循环
    expungeStaleEntries();

    // Use lower threshold for doubling to avoid hysteresis
    //大于容量的1/2，resize
    if (size >= threshold - threshold / 4)
        resize();
}
```

### 6.2 resize

```java
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    //两倍容量
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;
    //全部重新rehash
    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}
```

## 七、get方法

```java
public T get() {
    //获取表示当前线程的Thread对象
    Thread t = Thread.currentThread();
    //获取当前线程的ThreadLocalMap对象，这个类是java.lang.ThreadLocal的内部类，内部维护一个Entry数组，容量为2的n次幂，阈值为2/3
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        //取值
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //设置初始值
    return setInitialValue();
}
```

从线程中获取对应ThreadLocal为key的键值对，如果没有找到或者ThreadLocalMap未初始化，那么创建ThreadLocalMap并初始化

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            //清理脏数据
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```



## 八、总结

ThreadLocal使用Thread对象中的ThreadLocalMap储存数据，ThreadLocalMap没有实现Map接口，它内部使用一个2的n次幂大小的数组储存键值对Entry，key为ThreadLocal对象，value
为用户设置的值，另外这个Entry继承了WeakRefence，也就是说在ThreadLocal没有其他比弱引用更强引用存在的情况下，触发了gc就会回收掉这个ThreadLocal，这样做的目的是为了
避免内存泄漏并且对于一个key为空的键值对，在用户下次修改ThreadLocal的值时，会进行清理。
    