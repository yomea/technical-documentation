## 一、类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/579f8208e18b4b68f63d4d342ccd91a0.png)


- ChannelBuffer：接口，用于定义设置读索引，写索引，获取字节数据等方法。
- AbstractChannelBuffer：模板类，实现了操作索引的方法，具体的读取数据的方法交由子类实现。

## 二、字段说明

### 2.1 AbstractChannelBuffer

```java
//读索引
private int readerIndex;
//写索引
private int writerIndex;
//标记读索引
private int markedReaderIndex;
//标记写索引
private int markedWriterIndex;
```
### 2.2 CompositeChannelBuffer

```java
//字节顺序，大端或者小端
private final ByteOrder order;
//通道buffer的引用
private ChannelBuffer[] components;
//每个ChannelBuffer大小的叠加，比如现在应用了两个ChannelBuffer，一个大小为5，另一个为10，那么
//indices数组包含两个元素，一个是5，另一个是15
private int[] indices;
//记录上次访问的components数组下标
private int lastAccessedComponentId;

```

## 三、组合ChannelBuffer

```java
// 设置ChannelBuffer引用，计算容量
private void setComponents(List<ChannelBuffer> newComponents) {
    assert !newComponents.isEmpty();

    // Clear the cache.
    lastAccessedComponentId = 0;

    // Build the component array.
    components = new ChannelBuffer[newComponents.size()];
    for (int i = 0; i < components.length; i ++) {
        ChannelBuffer c = newComponents.get(i);
        if (c.order() != order()) {
            throw new IllegalArgumentException(
                    "All buffers must have the same endianness.");
        }

        assert c.readerIndex() == 0;
        assert c.writerIndex() == c.capacity();
        //记录ChannelBuffer的引用
        components[i] = c;
    }

    // Build the component lookup table.
    //构建指示容量的数组
    indices = new int[components.length + 1];
    indices[0] = 0;
    for (int i = 1; i <= components.length; i ++) {
        //每个元素都是当前ChannelBuffer的容量加上前一个元素的容量
        indices[i] = indices[i - 1] + components[i - 1].capacity();
    }

    // Reset the indexes.
    //将readIndex设置为0，writeIndex设置为indices[components.length]
    setIndex(0, capacity());
}

```

使用一个ChannelBuffer[]数组记录ChannelBuffer的引用，使用一个indices[]数组记录容量大小

## 四、获取数据

```java
//获取指定下标的数据
public byte getByte(int index) {
    //获取ChannelBuffer数组中的下标
    int componentId = componentId(index);
    //获取指定小标的字节数据
    return components[componentId].getByte(index - indices[componentId]);
}
```

确定ChannelBuffer数组中的下标

```java
private int componentId(int index) {
    //获取上次访问的ChannelBuffer下标
    int lastComponentId = lastAccessedComponentId;
    //如果下标指示容量大小小于等于当前需要索引的数据，那么说明要获取的数据就在
    //lastComponentId的右边
    if (index >= indices[lastComponentId]) {
        if (index < indices[lastComponentId + 1]) {
            return lastComponentId;
        }

        // Search right
        for (int i = lastComponentId + 1; i < components.length; i ++) {
            if (index < indices[i + 1]) {
                lastAccessedComponentId = i;
                return i;
            }
        }
    } else {
        // Search left
        for (int i = lastComponentId - 1; i >= 0; i --) {
            if (index >= indices[i]) {
                lastAccessedComponentId = i;
                return i;
            }
        }
    }

    throw new IndexOutOfBoundsException();
}
```
从上次访问过的下标开始，确定往右还是往左获取数据


## 五、总结

从上面的分析可以看出，CompositeChannelBuffer只是储存了ChannelBuffer的引用，并用indices数组记录容量大小，当获取数据时，通过indices数据决定使用
哪个ChannelBuffer，而不是将ChannelBuffer的数据拷贝成一个新的Buffer。