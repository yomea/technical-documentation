## 一、何为encoder？何为layout

我们的日志要输出到不同的appender，有的时候有编码要求，比如GBK，UTF-8等，就需要encoder，另外，我们的日志是需要一定的格式的，而不是随意
输出的，那么就需要用到layout，接下来我们先来分析一些比较通用的一个编码器 -》PatternLayoutEncoder

## 二、PatternLayoutEncoder

### 2.1 类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/419c2e524910cecae8f88f49b200a47a.png#pic_center)


现在我们对这个类图做个简单的介绍

- LifeCycle：定义了start与stop方法，在使用某对象之前需要先start，关闭时使用stop，通常一些
  初始化准备操作会写在这个start方法中，而释放资源，关闭流会写在stop方法中
- ContextAware：日志上下文aware，类似spring中的各种aware，比如ApplicationContextAware，继承该接口的类，会在使用前
  注入日志上下文对象，除此之外还有一些addInfo，addWarn等添加状态的接口，在具体的实现中会记录到一个集合中，超过大小之后会
  缓存在环形缓存中，另外还会触发状态监听器
- ContextAwareBase：ContextAware的基本实现
- Encoder：定义了三个接口，headerBytes 写入页眉，footerBytes写入页脚，比如说我们的日志写入的是html文本，那么页头与页脚通常是写入
  `</table>啊，</body></html>`之类的标签，encode方法说对我们输入的日志消息进行解析
- EncoderBase：实现了基本的 LifeCycle 的接口方法
- LayoutWrappingEncoder：实现了基本的页脚，页眉，日志信息编码方法
- PatternLayoutEncoderBase：设置pattern
- PatternLayoutEncoder：start方法设置 PatternLayout

我们首先来看到LayoutWrappingEncoder的编码方法

```java

public byte[] encode(E event) {
    // 调用 PatternLayout 的 doLayout 方法，对消息进行格式化
    String txt = layout.doLayout(event);
    // 通过字符集进行编码
    return convertToBytes(txt);
}

private byte[] convertToBytes(String s) {
    if (charset == null) {
        return s.getBytes();
    } else {
        return s.getBytes(charset);
    }
}


```
可以看到上面的代码调用layout对消息进行了格式化，那么整个layout的对象是从哪里设置进来的呢？我们可以看到PatternLayoutEncoder的start方法

```java
@Override
public void start() {
    PatternLayout patternLayout = new PatternLayout();
    patternLayout.setContext(context);
    patternLayout.setPattern(getPattern());
    patternLayout.setOutputPatternAsHeader(outputPatternAsHeader);
    patternLayout.start();
    this.layout = patternLayout;
    super.start();
}
```
从上面的代码可以看出，它使用的是 PatternLayout，那么它是怎么对消息进行格式化的呢？我们可以看下它的start方法，不过在看这个方法之前我们
先来看下它的类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/8b955430e49c9f9cb512e3f2ee17ae8d.png#pic_center)


我们对这个类图先做个简单介绍

- LifeCycle：定义了start与stop方法，在使用某对象之前需要先start，关闭时使用stop，通常一些
  初始化准备操作会写在这个start方法中，而释放资源，关闭流会写在stop方法中
- ContextAware：日志上下文aware，类似spring中的各种aware，比如ApplicationContextAware，继承该接口的类，会在使用前
  注入日志上下文对象，除此之外还有一些addInfo，addWarn等添加状态的接口，在具体的实现中会记录到一个集合中，超过大小之后会
  缓存在环形缓存中，另外还会除非状态监听器
- ContextAwareBase：ContextAware的基本实现
- Layout：主要定义格式化的接口
- PatternLayoutBase：主要在start方法中定义了如何解析格式pattern，额外的设置了一个模板方法，用于获取转换器
- patternlayout：设置默认的转换器

下面我们在来看到layout的start方法

```java
public void start() {
    if (pattern == null || pattern.length() == 0) {
        addError("Empty or null pattern.");
        return;
    }
    try {
        // 解析消息格式pattern，通过TokenStream对字符进行词法解析，形成token
        Parser<E> p = new Parser<E>(pattern);
        if (getContext() != null) {
            p.setContext(getContext());
        }
        // 根据解析出来的token形成语法树
        Node t = p.parse();
        // 编译成 Conventer，getEffectiveConverterMap()方法返回一个词法与Conventer的映射关系，比如
        // %d{HH:mm}中的 d =》DateConventer
        this.head = p.compile(t, getEffectiveConverterMap());
        if (postCompileProcessor != null) {
            postCompileProcessor.process(context, head);
        }
        // 批量设置Context上下文对象
        ConverterUtil.setContextForConverters(getContext(), head);
        // 批量调用start方法
        ConverterUtil.startConverters(this.head);
        super.start();
    } catch (ScanException sce) {
        StatusManager sm = getContext().getStatusManager();
        sm.add(new ErrorStatus("Failed to parse pattern \"" + getPattern() + "\".", this, sce));
    }
}
```

我们可以看到layout的start方法对消息pattern进行了词法解析，最终编译成了 Converter ，假设我们现在的消息格式pattern是这样的
%d{yyyy-MM-dd HH:mm:ss.SSS}-----%logger，那么解析出来的Converter是这样的 DateConventer -> LiteralConverter -> LoggerConverter

DateConventer 用来解析 %d{yyyy-MM-dd HH:mm:ss.SSS} ，LiteralConverter是字面量转换器，用来输出 ----- ， LoggerConverter用来输出
logger输出类。

