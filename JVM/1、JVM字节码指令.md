
> 本篇博文整理自我在博客园上发表的文章[《JVM指令》](https://www.cnblogs.com/honger/p/6815198.html)

> 本篇指令码表，参考自ASM文档手册，如果你对asm感兴趣，可到[ASM官网](https://asm.ow2.io/)下载手册学习。您还可以到http://homepages.inf.ed.ac.uk/kwxm/JVM/codeByNm.html网站去学习字节码更详细的内容
>
## 一、本地变量操作指令
	I,L,F,D,A这些前缀表示对int，long，float，double，引用进行操作

表一&nbsp;&nbsp;&nbsp;&nbsp;本地变量指令集
|指令|意义|
|--|--|
|ILOAD_n(0~3), LLOAD_n(0~3), FLOAD_n(0~3), DLOAD_n(0~3) 超过三的 直接 xLoad n,如ILOAD 4，LLOAD 5  | 将局部变量表中第n个槽的(int\|long\|float\|double)类型变量推送到操作数栈 |
|ALOAD_n(0~3) 超过3的 ALOAD n，如：ALOAD 5|将引用类型的局部变量第n个槽的推送到操作数栈|
|ISTORE_n(0~3), LSTORE_n(0~3), FSTORE_n(0~3), DSTORE_n(0~3) 超过三的xSTORE n|将操作数栈顶的(int\|long\|float\|double)类型值弹出存到局部变量表的第n个槽中|
|ASTORE_n(0~3) 超过3的 ASTORE n|将栈顶引用类型的值存到局部变量表中的第n个槽中|
|IINC var incr|将局部变量表中的第var个变量增加incr，并把新值存到局部变量表|

从上表可知本地变量操作表对应的下标是从0开始的，比如下面一段程序
```
public void print(int age) {
　　int a = age;
　　a++;
}
```
其对应的字节码指令如下

```
stack=1, locals=3, args_size=2//这里的参数为什么是2，因为参数里面有个this，这个this是隐藏的，在JVM中是以参数的形式传递进去的
 iload_1//将局部变量表中的第1个槽，也就是age这个值，0是this，压入操作数栈栈顶

 istore_2//将操作数栈顶的值，这里就是age，存到局部变量表的第二个槽，也就是a

 iinc 2, 1//将局部变量表中的第二个槽的a加1

 return//方法返回
```
>  注意，如果局部变量中有long或者double类型的值，那么会占用局部变量两个槽，如有局部变量int age,long l, double d, short s, byte b,那么对应的槽应该是1,2,4,6,7
byte,short,char,int,boolean类型的操作指令统一使用ILOAD或者ISTORE这些指令

## 二、栈操作指令
表二&nbsp;&nbsp;&nbsp;&nbsp;JVM栈指令
<table>
	<tr><th>指令</th><th>栈操作前</th><th>栈操作后</th></tr>
	<tr><td>POP</td><td>...  , v</td><td>... (v被弹出)</td></tr>
	<tr><td rowspan='2'>POP2</td><td>...  , v1  , v2</td><td>...（v1和v2被弹出）</td></tr>
	<tr><td>...  , w</td><td>...  (w表示占用两个槽的变量，如long，double之类）</td></tr>
	<tr><td>DUP</td><td>...  , v</td><td>...  , v , v            (复制一份)</td></tr>
	<tr><td rowspan='2'>DUP2</td><td>...  , v1  , v2</td><td>...  , v1  , v2  , v1 , v2         （复制栈中的两个值）</td></tr>
	<tr><td>...  , w</td><td>...  , w, w （复制一个long，double型的）</td></tr>
	<tr><td>SWAP</td><td>...  , v1  , v2</td><td>...  , v2  , v1 （交换）</td></tr>
	<tr><td>DUP_X1</td><td>...  , v1  , v2</td><td>...  , v2 , v1  , v2      （复制栈顶值v2，并将值放置到前一个槽的前面）</td></tr>
	<tr><td rowspan='2'>DUP_X2</td><td>...  , v1  , v2  , v3</td><td>...  , v3 , v1  , v2  ,  v3  （复制栈顶值v3，并将值放置到前二个槽的前面）</td></tr>
	<tr><td>...  , w ,  v</td><td>...  ,  v  , w  , v  （复制栈顶值v，并将复制的值插入到前两个槽的前面，w占两个槽）</td></tr>
	<tr><td rowspan='2'>DUP2_X1</td><td>...  , v1  , v2  , v3</td><td>...  , v2 , v3 , v1  , v2  ,  v3 （复制栈顶v2  , v3，并将值放置到前二个槽的前面）</td></tr>
	<tr><td>...  , v ,  w</td><td>...  , w  , v ,  w  （其中w占两个槽）</td></tr>
	<tr><td rowspan='4'>DUP2_X2</td><td>...  , v1  , v2  , v3  , v4</td><td>...  , v3 , v4 , v1  , v2  , v3  ,   v4 （复制栈顶两个槽，然后插入到v2前两个槽的前面）</td></tr>
	<tr><td>...  , w , v1  ,  v2</td><td>...  , v1  , v2  , w , v1  ,  v2（其中w占两个槽）</td></tr>
	<tr><td>....  , v1  , v2  ,  w</td><td>...  , w  , v1  , v2  ,  w（其中w占两个槽）</td></tr>
	<tr><td>...  , w1  , w2</td><td>...  , w2  , w1  , w2（其中w占两个槽）</td></tr>
</table>

eg:

```
public void print(int age, String name) {
　　this.age = age;
　　this.name = name;
}
```
对应的字节码指令为

```
aload_0   //将this入栈

dup       //复制一个this

aload_1  //将age入栈

putfield #n  //给age复制，这里的n表示一个数字，#n表示索引，对应常量池中的常量 

aload_2 //将name入栈

putfield #n //给name复制 
```
## 三、常量操作
表三&nbsp;&nbsp;&nbsp;&nbsp;常量操作指令
<table>
	<tr><th>指令</th><th>栈操作前</th><th>栈操作后</th></tr>
	<tr><td>ICONST_n (−1 ≤ n ≤ 5)</td><td>...</td><td>...  , n  （ 将整型常量n入栈）</td></tr>
	<tr><td>LCONST_n (0 ≤ n ≤ 1)</td><td>...</td><td>	
...  , nL　   （将长整型常量n入栈）　</td></tr>
<tr><td>FCONST_n (0 ≤ n ≤ 2)</td><td>...</td><td>	
...  , nF      （将float常量入栈）　</td></tr>
<tr><td>DCONST_n (0 ≤ n ≤ 1)</td><td>...</td><td>	
...  , nD     （将double常量入栈）</td></tr>
<tr><td>BIPUSH b, −128 ≤ b < 127</td><td>...</td><td>	
...  , b       （将byte常量入栈）</td></tr>
<tr><td>SIPUSH s, −32768 ≤ s < 32767</td><td>...</td><td>	
...  , s       （将短整型入栈）</td></tr>
<tr><td>LDC cst  (int, float, long, double, String or Type)</td><td>...</td><td>	
...  , cst    （将常量池中值入栈）</td></tr>
<tr><td>ACONST_NULL</td><td>...</td><td>	
... , null    （将null值入栈）</td></tr>
</table>

eg:

```
public void print（）{

　　int a1 = 1;              //ICONST_1将1入栈

　　　　　　　　　　　　　//ISTORE_1 将1存入局部变量表1中，即a1

　　int a2 = 10;           //BIPUSH 10

　　　　　　　　　　　　　//ISTORE 2

　　int a3 = 100;       // SIPUSH 100

　　　　　　　　　　　　//ISTORE 3

　　float a4 = 123f;   //LDC #4这个#4是引用了常量池里的值，123

　　　　　　　　　　　　//FLOAD 4
}
```
## 四、算术和逻辑操作指令
表四&nbsp;&nbsp;&nbsp;&nbsp;算术和逻辑操作指令表
| 指令 |栈操作前  |栈操作后 |
|--|--|--|
| IADD, LADD, FADD, DADD |...  , a ,  b  |...  , a + b （将栈顶的两个值相加，并把结果入栈） |
|ISUB, LSUB, FSUB, DSUB|...  , a ,  b|...  , a - b|
|IMUL, LMUL, FMUL, DMUL|	...  , a ,  b|...  , a * b|
|IDIV, LDIV, FDIV, DDIV|...  , a ,  b|...  , a / b|
|IREM, LREM, FREM, DREM|	...  , a ,  b|...  , a % b|
|INEG, LNEG, FNEG, DNEG|...  , a|...  , -a|
|ISHL, LSHL|...  , a ,  n|...  , a << n|
|ISHR, LSHR|...  , a ,  n|...  , a >> n|
|IUSHR, LUSHR|...  , a ,  n|...  , a >>> n|
|IAND, LAND|...  , a ,  b|...  , a & b|
|IOR, LOR|...  , a ,  b|...  , a \| b|
|IXOR, LXOR|...  , a ,  b|...  , a ^ b|
|LCMP|...  , a ,  b|...  , a == b ? 0 : (a < b ? -1 : 1)|
|FCMPL, FCMPG|...  , a ,  b|...  , a == b ? 0 : (a < b ? -1 : 1) (L结尾的表示当值为NaN时返回-1，G返回1)|
|DCMPL, DCMPG|...  , a ,  b|...  , a == b ? 0 : (a < b ? -1 : 1) (L结尾的表示当值为NaN时返回-1，G返回1)|

## 五、转换
表五&nbsp;&nbsp;&nbsp;&nbsp;转换指令表
| 指令 |栈操作前  |栈操作后 |
|--|--|--|
|I2B|...  , i|... , (byte) i|
|I2C|...  , i|... , (char) i|
|I2S|...  , i|... , (short) i|
|L2I, F2I, D2I|...  , a|... , (int) a|
|I2L, F2L, D2L|...  , a|... , (long) a|
|I2F, L2F, D2F|...  , a|... , (float) a|
|I2D, L2D, F2D|...  , a|... , (double) a|
|CHECKCAST class|...  , o|... , (class) o|

## 六、对象，字段，方法操作
表六&nbsp;&nbsp;&nbsp;&nbsp;
| 指令 |栈操作前  |栈操作后 |
|--|--|--|
|NEW class| ...| ... , new class|
|GETFIELD c f t| ...  , o |...  , o.f （获取o对象的f字段）|
|PUTFIELD c f t |...  , o ,  v |... （将栈顶值设置到o对象的f字段中）|
|GETSTATIC c f t |... |...  , c.f （获取c类的某个静态字段f值入栈）|
|PUTSTATIC c f t |...  , v |...设置栈顶值给c类的f字段|
|INVOKEVIRTUAL c m t |...  , o , v1  , ...  , vn |... , o.m(v1, ... vn)  （调用对象方法）|
|INVOKESPECIAL c m t |...  , o , v1  , ...  , vn |... , o.m(v1, ... vn) （比如调用父类方法super.m()）|
|INVOKESTATIC c m t |...  , v1  , ...  , vn |... , c.m(v1, ... vn) （调用静态方法）|
|INVOKEINTERFACE c m t |...  , o , v1  , ...  , vn |... , o.m(v1, ... vn) （调用接口方法）|
|INVOKEDYNAMIC m t bsm |...  , o , v1  , ...  , vn |... , o.m(v1, ... vn) （动态调用，比如拉姆达）|
|INSTANCEOF class |...  , o |... , o instanceof class|
|MONITORENTER |...  , o |... （进入synchronized）|
|MONITOREXIT |...  , o |...（退出synchronized）|

## 七、数组操作
表7 &nbsp;&nbsp;&nbsp;&nbsp;数组操作指令表
| 指令 |栈操作前  |栈操作后 |
|--|--|--|
| NEWARRAY type (for any primitive type) |...  , n  |... , new type[n]（构建一个n长度的数组）|
|ANEWARRAY class|...  , n|... , new class[n] |
|MULTIANEWARRAY [...[t n|...  , i1  , ...  , in|...  ,  new t[i1]...[in]...（n维数组）|
|BALOAD, CALOAD, SALOAD |...  , o ,  i|... , o[i] （操作字节，字符，短整型数组，使用数组引用o，读取器下标i的值，然后入栈）|
|IALOAD, LALOAD, FALOAD, DALOAD|...  , o ,  i|... , o[i]（操作int,long,float,double数组，使用数组引用o，读取器下标i的值，然后入栈）|
|AALOAD|...  , o ,  i|... , o[i] （操作对象数组，使用数组引用o，读取器下标i的值，然后入栈）|
|BASTORE, CASTORE, SASTORE|...  , o , i ,  j|...（将栈顶值j存储到o[i]中，即o[i] = j）|
|IASTORE, LASTORE, FASTORE, DASTORE|...  , o , i ,  a|...|
|AASTORE|...  , o , i ,  p|...|
|ARRAYLENGTH|...  , o|... , o.length|

## 八、跳转语句
表八&nbsp;&nbsp;&nbsp;&nbsp;跳转指令表
| 指令 |栈操作前  |栈操作后 | 含义|
|--|--|--|--|
| IFEQ | ...  , i |...|jump if i == 0|
|IFNE|...  , i|...|jump if i != 0|
|IFLT|...  , i|...|jump if i < 0|
|IFGE|...  , i|...|jump if i >= 0|
|IFGT|...  , i|...|jump if i > 0|
|IFLE|...  , i|...|jump if i <= 0|
|IF_ICMPEQ|...  , i ,  j|...|jump if i == j|
|IF_ICMPNE|...  , i ,  j|...|jump if i != j|
|IF_ICMPLT|...  , i ,  j|...|jump if i < j|
|IF_ICMPGE|...  , i ,  j|...|jump if i >= j|
|IF_ICMPGT|	...  , i ,  j|...|jump if i > j|
|IF_ICMPLE|...  , i ,  j|	...|jump if i <= j|
|IF_ACMPEQ|...  , o ,  p|...|jump if o == p|
|IF_ACMPNE|...  , o ,  p|...|	jump if o != p|
|IFNULL|...  , o|...|jump if o == null|
|IFNONNULL|...  , o|...|jump if o != nul|
|GOTO|...|...|jump always|
|TABLESWITCH|...  , i|...|jump always|
|LOOKUPSWITCH|...  , i|...|jump always|

## 九、return
表九&nbsp;&nbsp;&nbsp;&nbsp;return指令表
| 指令 |栈操作前  |栈操作后 |
|--|--|--|
|IRETURN, LRETURN, FRETURN, DRETURN  | ...  , a ||
|ARETURN|...  , o||
|RETURN|...||
|ATHROW|...  , o||

## 十、泛型描述符
如：
```
public class Test<T> ==> <T:Ljava/lang/Object;>

public class Test<T> extends ArrayList<E> ==> <T:Ljava/lang/Object;>Ljava/util/ArrayList<TE;>;

static <T> Class<? extends T> m (int n) ==> <T:Ljava/lang/Object;>(I)Ljava/lang/Class<+TT;>;

List<E> ==> Ljava/util/List<TE;>;

List<?> ==> Ljava/util/List<*>;

List<? extends Number> ==> Ljava/util/List<+Ljava/lang/Number;>;

List<? super Integer> ==> Ljava/util/List<-Ljava/lang/Integer;>;

List<List<String>[]> ==> Ljava/util/List<[Ljava/util/List<Ljava/lang/String;>;>;

HashMap<K, V>.HashIterator<K> ==> Ljava/util/HashMap<TK;TV;>.HashIterator<TK;>;

```

> 注意：如果是定义泛型，比如class Test<T>,方法中的<T>这类T,在写泛型签名的时候应当写成T:Ljava/lang/Object;而不是TT;在其他非定义泛型的位置，写成TT;

## 十一、Java类型描述符
表十&nbsp;&nbsp;&nbsp;&nbsp;Java类型描述符表
| java类型 | 类型描述符 |
|--|--|
| boolean | Z |
|char|C|
|byte|B|
|short|S|
|int|I|
|long|J|
|float|F|
|double|D|
|Object|Ljava/lang/Object;|
|int[]|[I|
|Object[][]|[[Ljava/lang/Object;|

## 十二、方法描述符
表十一&nbsp;&nbsp;&nbsp;&nbsp;方法描述符表
|方法| 方法描述符 |
|--|--|
| void m(int i, float f) |(IF)V  |
|int m(Object o)|(Ljava/lang/Object;)I|
|int[] m(int i, String s)|(ILjava/lang/String;)I|
|Object m(int[] i)|([I)Ljava/lang/Object;|
