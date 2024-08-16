## 一、简介

&nbsp;&nbsp;&nbsp;&nbsp;TreeMap是一个有序的Map，它直接由红黑树构成，所以学习
TreeMap只要学会红黑树就可以了，下面是TreeMap的类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/cbc8dbeca5740460d1e1e57cb464db93.png)

- Map：定义基本的增删除改查操作
- AbstractMap：模板方法，实现了一些基本方法
- SortedMap：从名字上来看就知道这是一个有序的map，它所定义的接口方法要求有序
- NavigableMap：可导航的，可以升序/降序的方式访问节点，访问指定节点的前一个节点
  后一个节点等等

## 二、红黑树

### 2.1 红黑树的特性

- 红黑树顾名思义就是节点具有颜色的特性，有黑色与红色两种
- 根节点为黑色
- 不能出现连续的红色的节点
- 叶子节点（NULL节点）都是黑色的
- 从根节点出发，到达任何叶子节点的黑色节点数量个数相同

#### 2.1.1 红黑树节点的插入

> 前提：插入节点都默认为红色，然后进行红黑树的调整，我们把当前插入的节点命名为N，父节点为P，祖父节点为G，叔叔节点为U，有数字的为其他节点(只是标号，不是值)，没有任何标记的为NULL节点

1. 没有节点的情况<br />
   没有节点，那么我们直接构建根节点，这个节点着色为黑色，以下是TreeMap的构建根节点的代码

```java
public V put(K key, V value) {
    Entry<K,V> t = root;
    //如果没有根节点
    if (t == null) {
        compare(key, key); // type (and possibly null) check
        //创建根节点，默认是黑色的
        root = new Entry<>(key, value, null);
        //大小赋值为1
        size = 1;
        //这个变量用于记录TreeMap的修改次数
        modCount++;
        return null;
    }
    。。。。。。省略节点插入与修正红黑色代码
```

在继续研究其他情况之前，我们先把TreeMap的put方法中的余下代码先分析完

```java
public V put(K key, V value) {
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;
    //如果设置了比较器，优先使用比较器
    if (cpr != null) {
        do {
            parent = t;
            //将传递进来的key与父key进行比较，用于确定传递进来的节点应该放在left，还是right
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    else {
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            //若果没有指定比较器，那么存放在TreeMap的key必须实现了Comparable接口
            Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    //插入节点
    Entry<K,V> e = new Entry<>(key, value, parent);
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;
    //插入节点后，我们的红黑树不一定能够满足特性，需要进行修正
    fixAfterInsertion(e);
    //递增节点大小
    size++;
    //递增修改次数
    modCount++;
    return null;
}
```
上面这段代码先查看是否已经创建了根节点，如果没有创建，那么创建，并且其颜色为黑色，如果已经创建了根节点，那么将传递进来的节点与父节点进行比较，根据比较的结
果来决定插入某个节点的left还是right，当插入完节点后，此时的红黑树可能不再满足红黑色的特性，需要进行调整

2. （插入节点的父节点的颜色为红色 AND 插入节点的父节点是其祖父节点的左子节点 AND 叔叔节点为红色），如下图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0f9d33fc8ad454f6c250eb2fa97f6272.png)

当我们插入一个红色节点N的时候如下图：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/90e7683eb1701a5672940c5e5f750e2a.png)

为什么插入新的节点要直接插入红色节点呢？假设我们插入黑色节点，那么势必会让插入了新节点的分支都增加一个黑色节点个数，所以为了尽量保证红黑树的特性，我们通常
直接插入红色节点，插入红色节点就一定不会导致某分支黑色节点个数的增加。如果插入的新节点的父节点不是红色节点，那么很好，不需要做任何其他的处理，如果是红色
节点，那么就违反红黑树不能有相连的红色节点的规则。通过插入新节点后的图可以知道这颗红黑树有以下问题：

- N和P组成了连续的两个红色节点，违反了红色节点不能连续的规则

那如果处理呢？对应的TreeMap代码如下：

```java
//。。。。。。。

//父节点为红色
while (x != null && x != root && x.parent.color == RED) {
        //如果当前节点父节点是祖父节点的左子节点
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            //叔叔节点
            Entry<K,V> y = rightOf(parentOf(parentOf(x)));
            //如果叔叔节点是红色的，那么设置父节点为黑色
            if (colorOf(y) == RED) {
                //父节点着色为黑色，用于解决N与P节点红色节点直接相连的问题
                setColor(parentOf(x), BLACK);
                //由于P节点被设置为黑色，那么从祖父节点G->p这条分支多了一个黑色节点
                //为了保持从祖父节点到其下其他分支的黑色节点个数一样，那么设置叔叔节点为黑色
                setColor(y, BLACK);
                //由于P与U节点都被设置了黑色，那么任何其他节点到G节点的分支都增加了1个黑色节点，为了保持同其他分支的黑色节点个数
                //一样，那么设置祖父节点为红色
                setColor(parentOf(parentOf(x)), RED);
                //当前值设置为祖父节点，然后继续判断父节点是否为红色节点，如果是红色的，那么继续循环，处理其他情况
                x = parentOf(parentOf(x));
            } else {
                
//。。。。。。。
//将根节点着色为黑色
root.color = BLACK;

```
为了解决N和P组成了连续的两个红色节点的问题，TreeMap将P节点设置为了红色，变化后的红黑树如下：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4cb54bfc3ccdcdf99fbfb09b0f057ffb.png)

但是问题又来了G->P这条路的黑色节点原本和G->U分支的黑色节点个数是一样的，现在不一样了，那么为了保持一样，TreeMap把叔叔节点变成了黑色

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/87b827c5b335e5bd7622dc5eb682f780.png)

将P和U都设置为黑色之后，1->G下所有的分支的黑色节点个数都加了1，为了保持与其他分支的黑色节点个数一致，需要把G设置为红色

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/6251e9f901349acef390925b0292e386.png)

最后将参照点设置为祖父节点，为什么要将祖父节点设置新的参照点？<br />
因为一开始我们将N插入，那么发生这个N以下的分支是符合红黑树规则的，而N以上的分支可能因为N的插入而不
符合红黑树特性，所以进行了各种操作使得祖父节点以下的分支是符合红黑树结构的，但祖父节点以上的分支构成的红黑树可能不符合规则，然后继续往上，直到根节点以下
的分支都符合规则了，那么就意味着整棵树已经符合规则了

所以修正红黑树都是从一个小树杈符合红黑树结构开始，再到大树杈乃至整颗树符合规则为止 --》 化零为整，所以思想上其实挺简单的，但是实际操作起来还是挺难的

3. （插入节点的父节点的颜色为红色 AND 插入节点的父节点是祖父节点的左子节点 AND 叔叔节点为黑色 AND 插入节点为父节点的左子节点），如下图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/002846a8d8118d0ffb6db5f9af15a26f.png)

插入新节点N之后如下：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/46e042b74420afc417b2890790f7afaf.png)

插入新节点之后N与P形成了连续的红色节点，以下就是TreeMap的解决方法

```java
//。。。。。。

while (x != null && x != root && x.parent.color == RED) {
    if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
        Entry<K,V> y = rightOf(parentOf(parentOf(x)));
        if (colorOf(y) == RED) {
           。。。。。。
        } else {
            if (x == rightOf(parentOf(x))) {
               。。。。。。
            }
            
            //将父节点P设置为黑色，为了解决N与P红色相连的问题
            setColor(parentOf(x), BLACK);
            //由于父节点变成了黑色，那么从G->P分支的黑色节点较其他分支的多了一个黑色节点
            //为了解决这个问题将祖父节点设置为红色
            setColor(parentOf(parentOf(x)), RED);
            //但是从G->U的分支就少了一个黑色节点，为了让G开始以下的left与right分支不会发生黑色的增加或者减少，那么可以将
            //原来的P节点移动到G节点，G节点移动到U节点，这样的话就满足条件了
            rotateRight(parentOf(parentOf(x)));
        }
    } 
    
//。。。。。。
```
N与P形成了连续的红色节点，为了解决这样的问题，那么TreeMap将P节点设置为黑色，那么就解决了这个问题

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0fcb9c424649560954e1e6ab6916d5d1.png)

从上图看，N与P已经不是连续的红色节点了，但是把P设置黑色，就意味着走P的分支会比其他的分支多上一个黑色节点，那么怎么办？<br />
TreeMap将祖父节点G设置为了红色，那为什么不能以P节点进行右旋呢？右旋之后原先的P->N分支确实保持了黑色节点个数，但是原先的P->右子节点的黑色节点个数并没有减少
所以设置祖父节点G设置为了红色

如下图：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/32f930f45a2af85e6586f22a5b14c830.png)

虽然把祖父节点设置为了红色，解决G -> P黑色增加的问题，但是这会导致G -> 其他分支（P除外的节点）的黑色节点少一个，那怎么办呢？继续向上把祖父节点的父节点设置
黑色？祖父节点的父节点还不一定是红色的呢，那这意思就得一直找红色节点设置为黑色喽。这是不靠谱的，从祖父节点以上的节点可能全是黑色的
TreeMap有更好的办法，那就是右旋，顾名思义就是将节点顺时针移动，将P节点移动到G节点位置，G节点移动到U节点（图中是P的右子节点，它是一个NULL节点），U节点保持
G节点的右子节点跟着继续往下移动。如图：

移动前

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/5c521a76d2ab9cbff39be8cbcdb40da5.png)

移动后

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/64a5deaf8df79c33673fca3c3799c249.png)

移动后P成G的父节点，G变成了P的父节点，P原来的右子节点将成为G节点的左子节点，可以看出即使进行旋转，他们的大小关系依然是正确的

4. （插入节点的父节点的颜色为红色 AND 插入节点的父节点是祖父节点的左子节点 AND 叔叔节点为黑色 AND 插入节点为父节点的右子节点），如下图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ae8c03ed156713e0990f078333e84efe.png)

插入新节点后：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/03063ef1f501a2e5c27139ffd627aadd.png)

和前面的问题一样N与P形成连个相连的红色节点，那么TreeMap是怎么解决的呢？代码如下：

```java
//。。。。。。
while (x != null && x != root && x.parent.color == RED) {
            if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
                Entry<K,V> y = rightOf(parentOf(parentOf(x)));
                if (colorOf(y) == RED) {
                    setColor(parentOf(x), BLACK);
                    setColor(y, BLACK);
                    setColor(parentOf(parentOf(x)), RED);
                    x = parentOf(parentOf(x));
                } else {
                    //如果新插入的节点是父节点的右子节点
                    if (x == rightOf(parentOf(x))) {
                        x = parentOf(x);
                        //将父节点进行左旋
                        rotateLeft(x);
                    }
                    setColor(parentOf(x), BLACK);
                    setColor(parentOf(parentOf(x)), RED);
                    rotateRight(parentOf(parentOf(x)));
                }
            } else {
            
//。。。。。。
```
TreeMap的操作方式，是将P节点进行左旋

移动前：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/943f10be7e55635a58ba0e164f00b109.png)

移动后：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4470e652223699f0e85e4f0bbae98ba2.png)

从图中看，进行左旋后似乎没有改变什么？那么这么做的目的是什么？ <br />
很明显，TreeMap这么做的目的就是将当前的情况转变成第3中情况，也就是（插入节点的父节点的颜色为红色 AND 插入节点的父节点是祖父节点的左子节点 AND
叔叔节点为黑色 AND 插入节点为父节点的左子节点）的情况。如果觉得模糊，那么我们将标号N与P进行互换

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/e17788c9eb76196731c027c7d4f50445.png)

既然已经构建成了第三种情况，那么按照第三种情况的操作方式去右旋节点即可


> 如果您看懂了插入节点的父节点是祖父节点的左子节点的红黑树插入过程，那么对于插入节点的父节点是祖父节点的右子节点的情况就可以不用看了，因为它们的思路是一样
的

5. （插入节点的父节点的颜色为红色 AND 插入节点的父节点是祖父节点的右子节点 AND 叔叔节点为红色 AND 插入节点为父节点的右子节点）

插入前：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/b9f94a51d75731fab814621e244c2958.png)

插入后：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/9d75bda901538f3426f4ec367b0b4264.png)

这种情况和第2种情况完全相反，TreeMap的代码如下：

```java
//。。。。。。
else {
    Entry<K,V> y = leftOf(parentOf(parentOf(x)));
    if (colorOf(y) == RED) {
        //父节点设置为黑色，解决N与P红色节点相连的问题
        setColor(parentOf(x), BLACK);
         //由于P节点设置为了黑色，导致经过P节点的分支多了一个黑色节点
        //那么将祖父节点设置红色
        setColor(parentOf(parentOf(x)), RED);
        //祖父节点设置为红色后，G -> U 分支的黑色节点少了一个
        //那么将U节点设置为黑色
        setColor(y, BLACK);
        x = parentOf(parentOf(x));
    } else {
        if (x == leftOf(parentOf(x))) {
            x = parentOf(x);
            rotateRight(x);
        }
        setColor(parentOf(x), BLACK);
        setColor(parentOf(parentOf(x)), RED);
        rotateLeft(parentOf(parentOf(x)));
    }
}
//。。。。。。
```

从代码上其操作方式和第二种情况差不多，不过多赘述，最终结果如下：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4fecf6e37d4c38a86dec59fd800809ed.png)

6. （插入节点的父节点的颜色为红色 AND 插入节点的父节点是祖父节点的右子节点 AND 叔叔节点为黑色 AND 插入节点为父节点的右子节点）

插入前：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/9a41e9a790ac2522e50384d218ab5d69.png)

插入后：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/73197622704491d7bd514eec311206fc.png)

TreeMap的解决方法

```java
//。。。。。。
else {
    Entry<K,V> y = leftOf(parentOf(parentOf(x)));
    if (colorOf(y) == RED) {
        //。。。。。。
    } else {
        if (x == leftOf(parentOf(x))) {
            //。。。。。。
        }
        //将父节点设置为黑色，这样就解决了N与P红色相连的问题
        setColor(parentOf(x), BLACK);
        //将父节点设置为黑色之后，经过P节点的分支的黑色节点个数增加了一个
        setColor(parentOf(parentOf(x)), RED);
        //进行左旋，解决P节点的分支的黑色节点个数增加的问题
        rotateLeft(parentOf(parentOf(x)));
    }
}
//。。。。。。
```
这种情况和第3情况相反，第三种情况是进行右旋，这里是左旋，因为我们插入的节点的父节点是祖父节点的右子节点，所以旋转方向相反，其他的着色方式一样，所以不再
赘述，下面进行旋转过后的红黑树

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/20fbb5782900bae9d875990a8723bc5d.png)

> Q&A：结合前面的第三种情况可以得知，旋转都发生在对红色节点上，为什么
因为旋转一个红色节点不会导致增加了这个红色节点的分支增加黑色节点个数，而失去这个红色节点的分支也不会因为缺失了一个红色节点而减少黑色节点个数，所以使用
红色节点进行旋转

7. （插入节点的父节点的颜色为红色 AND 插入节点的父节点是祖父节点的右子节点 AND 叔叔节点为黑色 AND 插入节点为父节点的左子节点）

插入前：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/e83c4e16c0d6afd2aed98d1c6dbd2d32.png)

插入后：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/64dcdca2086546290a86f592140cb2ee.png)

TreeMap的解决方案：

```java
//。。。。。。
else {
    Entry<K,V> y = leftOf(parentOf(parentOf(x)));
    if (colorOf(y) == RED) {
        //。。。。。。
    } else {
        if (x == leftOf(parentOf(x))) {
            x = parentOf(x);
            //以父节点进行右旋，构建成第6中情况
            rotateRight(x);
        }
        //。。。。。。
    }
}
//。。。。。。
```
TreeMap跟第4种情况的套路一样，它先将当前情况构建成第6种情况，然后按照第6种情况进行操作即可

看到这，可能有人会问，我就不把它设置成第6种情况，我就是要先试试以下这种方式

第一步：我把父节点P设置为黑色，用于解决N与P红色节点相连的情况

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/abba7e9d91cd6f67a7672cf14120b45c.png)

第二步：为了解决P节点设置为黑色导致的黑色节点增加的问题，我把祖父节点G设置为红色

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/3fd1816bc323f4af182c7499bdf4e0fd.png)

第三步：我把祖父节点设置为红色后，会导致从祖父节点到左子节点的黑色节点个数减去1个，为了解决这个问题，我以祖父节点G进行左旋（为啥不右旋，左子节点可能会是NULL，而右子节点是我们插入节点的分支，一定不为NULL，如果左子节点不为NULL，也可以右旋，因为旋转节点是红色的，所以效果是一样的，为了减少对左子节点的判断，不如直接左旋）


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/cd421e189026bf551e68775c84459b88.png)

看吧，这颗红黑树变成了和第4情况一样了

> 总结：平衡一颗红黑树，我们要抓住从部分到整体的方式进行整理，这里的部分经常以祖父节点，父节点，叔叔节点入手进行平衡树的修复。
插入新节点都是以红色节点的身份插入，避免导致黑色节点的个数增加，如果插入节点的父节点是黑色的，那么这颗树已经平衡，如果是父节点是红色的，那么就需要进行
平衡修复，修复分为简单修复和复杂修复，修复的复杂度由叔叔节点决定：
1.如果叔叔节点是红色节点，那么就是简单修复，只要把父节点，叔叔节点设置为黑色，祖父节点设置为红色，那么从祖父节点以下的这部分已经平衡，然后以祖父节点作为
新的当前节点，找到它的父节点，祖父节点，叔叔节点做为新的部分进行修复，直到根节点。
2.如果叔叔节点是黑色的，那么就是复杂修复，需要通过旋转操作来达到平衡的目的，旋转操作通常以红色节点作为旋转目标，因为红色节点的旋转不会导致增加红色节点的分支和减少红色节点的分支损失黑色节点，旋转完毕后，以之前的旋转点的位置的节点作为当前节点继续修复操作

#### 2.1.2 红黑树节点的删除

如果我们要删除的节点是一颗红色的节点，那么不需要进行平衡的修复，因为减少一个红色节点不会导致黑色节点的减少

如果我们要删除的节点是一颗黑色的节点，那么会导致经过这个黑色节点的分支的黑色节点个数减少一个，下面为TreeMap删除节点的代码

```java
private void deleteEntry(Entry<K,V> p) {
    //修改次数加1
    modCount++;
    //大小减1
    size--;

    // If strictly internal, copy successor's element to p and then make p
    // point to successor.
    //如果要删除的节点同时存在左子节点和右子节点，那么需要比P稍大的那个节点，这个节点将替换掉这个P
    if (p.left != null && p.right != null) {
        //找到下一个大于P的节点
        Entry<K,V> s = successor(p);
        //替换key
        p.key = s.key;
        //替换value
        p.value = s.value;
        //以替换P的那个节点作为新的删除对象
        p = s;
    } // p has 2 children

    // Start fixup at replacement node, if it exists.
    //如果到达了这里，当前这个要被删除的p节点要么只有一个孩子，要么没有孩子
    //这段代码的意思就是找下一个继承节点
    Entry<K,V> replacement = (p.left != null ? p.left : p.right);
    
    //判断是否存在继承者
    if (replacement != null) {
        // Link replacement to parent
        //将祖父节点设置为继承节点的父节点
        replacement.parent = p.parent;
        //如果删除节点没有父节点，那么就是删除节点是根节点，只需把继承节点设置为根节点即可
        if (p.parent == null)
            root = replacement;
        //删除节点为其父节点的左子节点
        else if (p == p.parent.left)
            p.parent.left  = replacement;
        else
            //删除节点为其父节点的右子节点
            p.parent.right = replacement;

        // Null out links so they are OK to use by fixAfterDeletion.
        //断绝关系
        p.left = p.right = p.parent = null;

        // Fix replacement
        //如果删除节点是黑色节点，需要进行平衡修复
        if (p.color == BLACK)
            fixAfterDeletion(replacement);
    } else if (p.parent == null) { // return if we are the only node.
        //如果是根节点
        root = null;
    } else { //  No children. Use self as phantom replacement and unlink.
        //如果删除节点是黑色节点，需要进行平衡修复
        if (p.color == BLACK)
            fixAfterDeletion(p);
        //绝后的节点，断绝父子关系
        if (p.parent != null) {
            //如果当前要删除的节点为父节点的左子节点，那么断绝左子节点的关系
            if (p == p.parent.left)
                p.parent.left = null;
            //如果当前要删除的节点为父节点的右子节点，那么断绝右子节点的关系
            else if (p == p.parent.right)
                p.parent.right = null;
            //彻底断绝父子关系，老死不相往来
            p.parent = null;
        }
    }
}
```

对于删除节点，TreeMap会做一些处理，如果这个删除节点有两个孩子，那么它会找到下一个比它大的节点，替换它的key和value，然后替换它的节点设置为新的删除节点。
下面我们用图来说明TreeMap的操作

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0ffefaad058b205dba747f86f9c2809b.png)

假设此时我要删除标号为1的节点，由于红黑树是有序的，那么下一个要替换它的节点是按照循序第一个比它大的节点，在图中是标号为6的节点，那么我们只要把标号6的内容
设置到标号1的节点，最后删除标号6的节点即可，但是不一定都是这么顺利的，被删除节点可能是一个黑色节点，这个时候就需要进行红黑树的修复

对于删除节点最终分为两大类，第一种就是没有任何子节点的节点，第二种就是存在一个子节点的节点

如果删除节点为红色，那么只要断绝它与其他节点的关系即可，如果是黑色节点，那么经过这个这个黑色节点的分支都会少一个黑色节点，那么就需要进行红黑树的修复
这里有个问题，那就是进行红黑树修复的时候，传入的基准节点会根据删除节点是否有子节点发生变化，如下代码：

```java
    //。。。。。。省略部分代码
    // Fix replacement
    if (p.color == BLACK)
        fixAfterDeletion(replacement);
        
        
    //。。。。。。省略部分代码
    
     } else { //  No children. Use self as phantom replacement and unlink.
            if (p.color == BLACK)
                fixAfterDeletion(p);
                
    //。。。。。。省略部分代码
```
第一段代码是有子节点的情况，它传入的基准节点为它的继承节点<br />
第二段代码的节点是没有子节点的，它传入的基准节点是它本身<br />
why?<br />

对于这个问题，我们要按位置看，因为这个位置是发生变更的点，假设有以下红黑树

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/e6eacfb78271a4cfb11866f8007c384a.png)

假设我们要删除标号为P的节点，那么它的后继节点就是标号为N的红色节点，N节点将会替换到P的位置，祖父节点变成父节点，叔叔节点变成兄弟节点

如果我们要删除黑色的标号为9的节点，这个9号节点没有子节点，那么它这个位置会被NULL节点替换，我们可以认为这个NULL节点就是它的后继节点，但是NULL节点不维护
与其他节点的关系，所以就继续复用9号节点来代替这个NULL节点（如果不使用这个9号节点，我们完全可以new出一个维护着与其他节点关系的黑色节点）

1. 第一种情况

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d54cba28dca6102a1ce968559ef26a60.png)

此时我要删除P节点，删除P节点后的红黑树

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/dee2467c682072a2938ff9e647fc3102.png)

从图中可以看到，P节点删除之后，后继节点顶替了P节点的位置，这里就出了问题了，G -> N的分支都减少了一个黑色节点，形成了第一种情况

> 情况描述：N节点是父节点P的左子节点 AND N节点为红色 AND N节点的兄弟节点S是红色节点

TreeMap对这种情况的处理代码如下：

```java
//。。。。。。
while (x != root && colorOf(x) == BLACK) {
    
//。。。。。。
}
setColor(x, BLACK);
```
由于顶替P节点的N节点是一个红色节点，所以直接将这个N节点设置为黑色即可

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/6c4e9a7448ef1041e965f6a4920dcb50.png)

2. 第二种情况

有如下红黑树的中一部分，灰色的C节点表示已被删除，蓝色的的节点表示未知颜色

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/82c8b1aab82833652ca251d2e6061242.png)

> 情况描述：C节点是其父节点的左子节点，C节点的兄弟节点I是黑色的，并且兄弟节点I的右子节点是红色节点，左子节点是黑色节点

TreeMap处理代码如下：

```java
//。。。。。。
 while (x != root && colorOf(x) == BLACK) {
 
  //。。。。。。
    } else {
    if (colorOf(rightOf(sib)) == BLACK) {
        //。。。。。。
    }
    //将C节点的父节点颜色设置给兄弟节点，这是为了后面左旋之后能保持父节点这个位置的节点颜色不变
    setColor(sib, colorOf(parentOf(x)));
    //将C节点的父节点设置为黑色，这是因为C节点删除之后，经过C分支的黑色节点少了一个，等会将父节点进行左旋后能够增加一个黑色节点
    setColor(parentOf(x), BLACK);
    //将兄弟节点的右子节点设置为黑色，这是因为左旋之后兄弟节点分支会少去一个黑色节点，所以这里补上一个黑色节点
    setColor(rightOf(sib), BLACK);
    //以父节点进行左旋
    rotateLeft(parentOf(x));
    x = root;
    }
    
    //。。。。。。
    
    setColor(x, BLACK);
}
```
对于现在这种情况，我们已知的信息是删除的C节点是黑色，父节点未知颜色（可能是黑，也可能是红），兄弟节点I是黑色，兄弟节点I的右子节点是红色。

从代码上看，TreeMap的修复思路是：如果保持父节点这个位置的颜色不变，其下的任意一个分支的黑色节点个数都一样的话，那么这颗红黑树就平衡了。所以TreeMap将父节点
设置为了黑色，是为了左旋给C分支增加黑色节点做准备，然后为了保持父节点位置的颜色不变，就把兄弟节点设置成原来父节点位置的颜色，不管原来的父节点是黑色也好，还
时红色也好，进行左旋之后都会导致I -> K分支少去一个黑色节点，此时兄弟节点的右子红色节点K派上了用场，可以将其直接设置为黑色，以保持黑色节点个数的平衡

好了，下面我们来看下图像化的操作过程

(1) 将C的父节点的颜色赋值给兄弟节点I

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/37fff519490b508862842874b1763756.png)

(2) 将c的父节点B的颜色设置为黑色

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/30d8d8d398e07afe59d4336730796520.png)

(3) 将c的兄弟节点I右子节点设置为黑色

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/cfc7dc618ef019fe4cc5cc8e3ae63435.png)

(4) 以C的父节点B进行左旋，然后将基准节点直接设置为root，跳出循环

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/c9c2085d4cff0a9cb531f32758755cff.png)

修复后的红黑树最顶端的节点颜色和一开始未修复时的颜色是一样，并且这个蓝色节点以下的各个分支的黑色节点个数都变成了一样的，那么可以肯定通过上面4个步骤就完成了
红黑树的修复。

但是鄙人认为还有其他的修复方式，下图是未修复前的标准红黑树中的一部分节点

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/6a7a082611c7d9a2f34c82f2e844b9ae.png)

由于删除C节点之后，会导致经过C节点的分支减去一个黑色节点，所以我们现在要做的时候怎么给它补上一个黑色节点，我们现在已知的情况是删除的C节点是黑色，
父节点未知颜色（可能是黑，也可能是红），兄弟节点I是黑色，兄弟节点I的右子节点是红色。

- 假设父节点B是红色，如下图：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/f2284c340a6fa6d9bf9079be135cf04a.png)

对于这样的我们直接左旋即可，旋转后的图如下：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/5cabb542a6c04c6667500d4ce70bd00e.png)

- 假设父节点B是黑色，我们尝试左旋，经过C节点的分支补充上了一个黑色节点，但是 I -> K的分支少了一个黑色节点，那么我们可以直接将红色节点K设置为黑色

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/9e2a4c6aebd7c2e89964c805f172a897.png)

3. 第三种情况

下图为删除C后的标准红黑树的一部分节点

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/505cb0ef6c2e0162152c006771afe16b.png)

> 情况描述：C节点是父节点的左子节点，C节点的兄弟节点I为黑色的，其右子节点是黑色节点，左子节点是红色节点

TreeMap的处理方法

```java
//。。。。。。
 while (x != root && colorOf(x) == BLACK) {
 
 // 。。。。。。
    } else {
    //C的兄弟节点的右子节点是黑色的，首先需要将其修正成第四情况
    if (colorOf(rightOf(sib)) == BLACK) {
        //将兄弟节点I的左子节点设置为黑色，为后面右旋做准备
        setColor(leftOf(sib), BLACK);
        //将兄弟节点I设置为红色，为后面右旋做准备
        setColor(sib, RED);
        //右旋，此时红黑树的状态变成了第二种情况，然后按照第二种情况处理即可
        rotateRight(sib);
        sib = rightOf(parentOf(x));
    }
    //。。。。。。
    }
    
    。。。。。。
    
    setColor(x, BLACK);
}
```
(1) 将C的兄弟节点的左子节点设置为黑色，为后面右旋做准备

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/1b56818be75eb5abe28cc057e8c93165.png)

(2) 将兄弟节点I设置为红色，为后面右旋做准备

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/f005d77ed75c90f602d007240ba76242.png)

(3) 以兄弟节点I进行右旋，得到了第二种情况，然后按照第二种情况处理就行了

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/18b8431a8cb39950aa1fffec73bacb7a.png)


4. 第四种情况

以下为某标准红黑树中的一部分节点，灰色的C节点表示删除节点，蓝色B节点表示颜色未知节点

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ef10acb2e1e5a20a04bbea9f55dcb093.png)

> 情况描述：C节点为父节点的左子节点，C节点的兄弟节点K是黑色的，兄弟节点的两个子节点也是黑色的

```java
while (x != root && colorOf(x) == BLACK) {
//。。。。。。
//兄弟节点的两个子节点都为黑色，如果是NULL节点，也是黑色
if (colorOf(leftOf(sib))  == BLACK &&
    colorOf(rightOf(sib)) == BLACK) {
    //将兄弟节点设置为红色，让兄弟节点分支和C节点分支一样缺少一个黑色节点，这是为了将基准设置为父节点做准备的
    setColor(sib, RED);
    //将基准设置为父节点
    x = parentOf(x);
}
//。。。。。。
//将N节点设置为黑色
setColor(x, BLACK);

}

```
TreeMap的做法是将兄弟节点直接置为红色，如下图：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/94b2b965e12aa92cd8e83afb5835a072.png)

这样一来，从父节点B以下的所有分支都减少了一个黑色节点，然后TreeMap将基准节点设置为父节点，只要修复到父节点的黑色节点个数，即可达到平衡。如果这个父节点是红
色节点，那么很好，我们只要把这个父节点设置为黑色就可以让红黑树平衡，如果这个父节点是黑色的，那么需要继续循环找到其他对应的情况来处理


5. 第五种情况

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/1dc29d305a3a840c6d3e26cce55e31f0.png)

> 情况描述：C节点为父节点的左子节点，C节点的兄弟节点K是红色节点，由于一开始是一颗平衡的红黑树，那么可以确定父节点B一定是黑色的，兄弟节点的两个子节点为黑色

下面是TreeMap对这种情况的转换操作

```java
//。。。。。。
while (x != root && colorOf(x) == BLACK) {
    if (x == leftOf(parentOf(x))) {
        Entry<K,V> sib = rightOf(parentOf(x));
        if (colorOf(sib) == RED) {
            //将兄弟节点设置为黑色
            //将父节点设置为红色
            //这是为了左旋避免K->D或K->I分支的黑色个数增加
            setColor(sib, BLACK);
            setColor(parentOf(x), RED);
            //左旋
            rotateLeft(parentOf(x));
            //更新成新的兄弟节点
            sib = rightOf(parentOf(x));
        }
//。。。。。。
setColor(x, BLACK);
```
> 重点：对于这种情况，TreeMap的逻辑就是将它转换成其他情况，装换成其他的情况的最重要的步骤就是将兄弟节点变成黑色的，怎么把兄弟节点变成黑色呢？我们现在知道C
节点的兄弟节点K是红色，那么它的孩子节点一定是黑色的，尽管可能是NULL节点，它也是黑色的，那么我们可以通过左旋将兄弟节点K的左子节点移过来，成为C节点的新兄弟节
点，但是左旋的时候我们要保持兄弟节点K分支的黑色节点个数，现在父节点B是黑色，如果直接左旋肯定会导致黑色节点的减少，所以TreeMap的做法就是将兄弟节点K由红色变
为黑色，父节点B由黑色变为红色，然后进行左旋

以下为具体的图像转换

(1) 将兄弟节点设置为黑色

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/461d0297101cc9fcbed50d5599d31c39.png)

(2) 父节点B设置为红色

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/cd29362fb0e5b1b0f2b6669d7b1ed484.png)


(3) 以父节点进行左旋

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/b94a144e457d37d8531d8d11f38d783f.png)

看，此时C节点的兄弟节点变成了D节点，这就转换成了第4中情况，第四中情况是直接将兄弟节点D设置为红色，然后将基准节点设置为父节点，由于父节点是红色节点，所以
直接将父节点设置为黑色即可。

6. 其他情况与上面对称，也就是相反的情况


```java
while (x != root && colorOf(x) == BLACK) {
            if (x == leftOf(parentOf(x))) {
               //。。。。。。
            } else { // symmetric
                Entry<K,V> sib = leftOf(parentOf(x));
                //与第二种情况对称，处理方式对称
                if (colorOf(sib) == RED) {
                    setColor(sib, BLACK);
                    setColor(parentOf(x), RED);
                    rotateRight(parentOf(x));
                    sib = leftOf(parentOf(x));
                }
                //与第三种情况对称，处理方式一样
                if (colorOf(rightOf(sib)) == BLACK &&
                    colorOf(leftOf(sib)) == BLACK) {
                    setColor(sib, RED);
                    x = parentOf(x);
                } else {
                    //与第5中情况对称，处理方式对称
                    if (colorOf(leftOf(sib)) == BLACK) {
                        setColor(rightOf(sib), BLACK);
                        setColor(sib, RED);
                        rotateLeft(sib);
                        sib = leftOf(parentOf(x));
                    }
                    setColor(sib, colorOf(parentOf(x)));
                    setColor(parentOf(x), BLACK);
                    setColor(leftOf(sib), BLACK);
                    rotateRight(parentOf(x));
                    x = root;
                }
            }
        }

        setColor(x, BLACK);
```
上面一段代码的处理方式和前面5中情况是对称的，所以不再重复赘述


> 总结

- 如果删除的节点或者继任节点是红色的，那么不会破坏红黑树的规则
- 如果删除节点或者继任节点是黑色的，那么会导致经过这个黑色节点的分支都缺少一个黑色节点。删除节点或者继任节点的兄弟节点决定了这个红黑树的修复方式
    - 如果兄弟节点是红色的，那么需要进行旋转让兄弟节点变为黑色节点
    - 如果兄弟节点是黑色节点，那么他的子节点的颜色也决定着修复方式
        - 如果兄弟节点的两个子节点（包含NULL节点）都是黑色的，那么会将兄弟节点变为红色，这是为了让兄弟分支与删除的黑色节点个数保持一致，对于这种情况，如果这颗红黑树全是黑色的，那么很显然修复后的红黑树的各个分支都会比原来少一个黑色节点，多一个红色节点
        - 如果兄弟节点的子节点是一红一黑，那么可以通过旋转的方式进行局部修复


经过以上的分析，我们学习了TreeMap红黑树的插入与删除，像查询比较简单就不过多的叙述了，感兴趣的自己去看，顺便提下TreeSet，它内部使用的就是一个TreeMap，只是
value变成了一个固定的Object对象。