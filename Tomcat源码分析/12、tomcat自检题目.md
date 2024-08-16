## 一、tomcat原理篇

### 1.1 为什么tomcat需要自定义线程池org.apache.catalina.core.StandardThreadExecutor？

实际上其内部执行任务的仍然是JDK的ThreadPoolExecutor
从继承结果上来看，tomcat的线程池实现了Lifecycle，Executor, ResizableExecutor，
其中JDK的线程池是没有实现Lifecycle与ResizableExecutor这个两个接口的，Lifecycle接口用于监控某个对象的生
命周期，然后通知生命周期监听器做些事情，ResizableExecutor用于调整线程池的大小
（最终是调用JDK的ThreadPoolExecutor的方法调整池子的大小）

### 1.2 tomcat的Digester类是做什么用的？

Digester是一个sax解析xml的处理器，tomcat对xml的解析设置了一些列的Rule，比如ObjectCreateRule，SetNextRule等。
规则对标签的开始（start），标签体（body），标签结束（end）的解析都有对应的方法。tomcat通过这些规则解析
config/server.xml构建起了tomcat的内核架构

### 1.3 tomcat的servlet为什么要包装成StandardWrapper？

StandardWrapper实现了Container接口和Lifecycle接口，我们可以参与它的生命周期处理，容器内部又会构建管道，所以我们还可以自定义管道阀参与请求的处理过程


### 1.4 tomcat发布context的过程？

tomcat发布web应用有三种方式：<br />
- 第一种是通过context描述文件(用Digester构建StandardContext,填充属性)进行发布，描述文件通常在tomcat安装目录/conf/Catalina/host名 目录下。有以下几种情况
    - 描述文件指定的web项目不是war包，docBase（web项目路径）在webappBase外面。
        - 监控context描述文件是否修改
        - 监控这个外部web项目的docBase
        - 添加全局的监控资源，configBase下的context.xml.default和conf下的Context.xml context描述文件
    - 描述文件指定的web项目是war包，docBase在webappBase外面。
        - 监控context描述文件是否修改
        - 监控这个外部web项目的docBase
        - 因为是war包，tomcat将war包解压后会放到webAppBase目录下，这里会将这个目录加入监控
        - 添加全局的监控资源，configBase下的context.xml.default和conf下的Context.xml context描述文件
    - 描述文件指定的web项目是war包，web项目在webappBase里面（即使在context描述文件中指定了docBase路径，也会将StandardContext的docBaes属性暂时设置为null）。
        - 监控war包，war路径由 webAppBase + context解析context描述文件名时的baseName + war后缀
        - 监控context描述文件
        - 监控这个war包解压后的目录，路径为 webAppBase + context解析context描述文件名时的baseName
        - 添加全局的监控资源，configBase下的context.xml.default和conf下的Context.xml context描述文件
    - 描述文件指定的web项目不是war包，web项目在webappBase里面（即使在context描述文件中指定了docBase路径，也会将StandardContext的docBaes属性暂时设置为null）。
        - 监控war包（虽然不存在，但是为了防止后面会手动添加进来，它的初始修改时间设置为0），war路径由 webAppBase + context解析context描述文件名时的baseName + war后缀
        - 监控context描述文件
        - 监控web路径，路径为 webAppBase + context解析context描述文件名时的baseName
        - 添加全局的监控资源，configBase下的context.xml.default和conf下的Context.xml context描述文件
- 第二种是通过war发布，会解压到webappsBase目录下，如果在META-INF文件夹下有context xml描述文件就以第一种方式发布
    -  检查在META-INF目录下是否存在context.xml描述文件
        -  如果存在并且允许拷贝context描述文件，那么会将这个描述文件拷贝到tomcat安装目录/conf/Catalina/host名目录下，描述文件重命名为context的{baseName}.xml，如果这个war包的名字不是很特殊，不存在版本信息，那么其baseName通常是跟war的名字一样
        -  解析context.xml描述文件创建StandardContext
        -  监控war包，拷贝后的context描述文件，解压后的docBase路径
    -  如果不存在context.xml那么直接反射创建StandardContext对象
        - 监控docBase路径和war包
- 第三种是通过文件夹发布，如果在META-INF文件夹下有context xml描述文件就以第一种方式发布<br />
    -  检查在META-INF目录下是否存在context.xml描述文件
        -  如果存在并且允许拷贝context描述文件，那么会将这个描述文件拷贝到tomcat安装目录/conf/Catalina/host名目录下，描述文件重命名为context的{baseName}.xml，如果这个web目录的名字不是很特殊，不存在版本信息，那么其baseName通常是跟web目录的名字一样
        -  解析context.xml描述文件创建StandardContext
        -  监控拷贝后的context描述文件和docBase路径
    -  如果不存在context.xml那么直接反射创建StandardContext对象
        - 监控docBase路径

### 1.5 tomcat的web隔离是怎么实现的？

tomcat在启动每个web项目时会创建一个专属于这个context的webappClassLoader，它有两种模式
设置为true时，表示通过双亲委派的方式进行类加载
- 代理模式 webapp类加器器缓存-》JVM中是否已经加载-》java类加载器（双亲委派）-》父加载器（双亲委派）-》自己加载
- 非代理模式 webapp类加器器缓存-》JVM中是否已经加载-》java类加载器（双亲委派）-》自己加载-》父加载器（双亲委派）

webappClassLoader内部维护了一个叫ParallelWebappClassLoader的URLClassLoader，它的urls构造参数没有传递任何classpath路径，它重写了loadClass方法。
对于web应用路径下的/WEB-INF/classes与/WEB-INF/lib的class通过tomcat的WebResourceRoot进行读取

### 1.6 tomcat对web.xml的解析？

通过ContextConfig这个生命周期监听器解析web.xml，解析web-fragment
将servlet解析为servletDef，filter解析为FilterDef，listener直接解析为类名字符串（因为listener除了类名，没有其他的参数）。
合并web.xml，解析web-fragment，并进行排序，因为有些监听器类可以配置启动的顺序。

扫描WEB-INF/classes下的类，使用字节码工具检查是否标有 @WebServlet,@WebFilter,@WebListener注解的类
扫描ServletContainerInitializer实现类，下面是这个接口的定义

```
public interface ServletContainerInitializer {

    //ServletContainerInitializer的实现类上面需要标注HandlesTypes注解
    //当tomcat扫描jar包或者WEB-INF/classes下的类时会检查它是否属于HandlesTypes注解上指定的类型
    //如果是，那么将通过这个c传递
    //参数ctx就是创建StandardContext时创建的ServletContext，我们可以通过它动态注册servlet，filter，listener和添加属性的操作
    void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException;
}
```

ServletContainerInitializer在StandardContext启动时进行调用 -》创建监听器，触发上下文初始化事件，调用ServletContextListener的contextInitialized方法 -》
初始化Filter -》初始化loadOnStartup不小于零的servlet

### 1.7 tomcat是如何实现热部署的？

在创建StandardContxt阶段（在HostConfig这个监听器中创建），会将web资源的路径，war包（如果有），context描述文件（如果有）
加入监控（以key为路径 -》value为修改时间存储到一个Map中），
然后在启动Engine这个容器的时候会启动一个后台进程，默认每隔10s来一次，发出periodic生命周期事件，HostConfig捕捉到后，对已发布的应用进行资源修改时间比对，发生修改的，
先停止，将webappclassloader置空，servletContext置空，资源什么的全部清空，然后再按照context启动的方式启动，解析context描述文件，web.xml，
创建webAppClassLoader。。。。。。，检查完已发布的context是否需要reload之后，又调用发布context方法，用于检测是否有新的web项目加入进来。

### 1.8 tomcat是如何注册web应用的？

通过MapperListener注册容器到Mapper，首先注册的是Host，Host被包装成一个MappedHost，以host名进行从小到大排序（主要用于二分法快速查找）。<br />
接下来注册的就是context，context被封装成MappedContext，MappedHost用一个成员变量contextList去维护其下的context。由于每个context可能存在多个版本，所以MappedContext
也维护了一个ContextVersion集合，用于表示多个context版本，最后
注册wrapper，分为三种类型的wrapper，通配符wrapper（/\*），扩展名wrapper（\*.），精确匹配的wrapper
对于通配符的wrapper，它会把路径映射的/\*截取掉，包装成MappedWrapper，分类放到一个通配符wrapper集合中，同样的扩展名wrapper（\*.）也是类似的操作

### 1.9 tomcat的通信模型？

tomcat使用reactor线程模式，一个Accepter，多个Poller，Accepter用于接收请求，每个Poller持有一个多用复用器selector，用于注册允许接收的请求。

### 1.10 tomcat是如何限制请求数的？

tomcat使用LimitLatch限制最大连接数，默认最大连接数为10000，连接数可以再server.xml中配置。
LimitLatch是基于并发框架AQS实现的，计数方式使用的是AtomicLong原子类，当超过最大限制时将进入阻塞状态。

### 1.11 tomcat的协议处理方式？

tomcat有多种协议处理实现，比如apr，http。socket端点也有多个实现，比如nio，bio，aio，协议处理与socket端点之间使用桥接模式进行组合。socket负责连接方式，搭桥，协议规定使用什么的规则传输数据。这里举个http协议处理的例子：

当我们发出一个http请求到达tomcat时，触发accepter，accepter轮询一个Poller，把代表这个请求的socketChannel注册到Poller的selector中。
接着读取事件被触发，tomcat首先先取消注册到当前Poller的读事件，因为tomcat读取socketChannel中的数据包括后面查找容器，处理用户业务代码都是通过线程池处理的，所以这里
为了避免占用Poller的线程太久（一个Poller只有一个线程）才这么做的。

现在假设http请求协议如下：

```
GET /index.html?name=xxx&password=123456 HTTP/1.1
Host: www.xxxx.com
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/56.0.2924.87 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8,en;q=0.6

请求体数据

```
tomcat中解析http的动作发生在Http11InputBuffer类中

- 第一步解析http协议的第一行，在tomcat中分了6个步骤
    - 第一阶段：循环跳过空白符
    - 第二阶段：解析方法，比如这里的GET方法
    - 第三阶段：跳过空白符
    - 第四阶段：解析uri和查询字段，这里的uri就是/index.html，查询字段就是name=xxx&password=123456
    - 第五阶段：跳过空白符
    - 第六阶段：解析协议，这里就是HTTP/1.1

- 第二步解析请求头
    - 解析请求头，分为这么几个解析状态：
        - HeaderParseStatus.DONE：表示解析完成
        - HeaderParseStatus.HAVE_MORE_HEADERS：表示还有请求头需要继续解析
        - HeaderParseStatus.NEED_MORE_DATA：表示数据不够，需要继续监听read事件获取数据

    - 解析位置有这么几个状态：

        - HeaderParsePosition.HEADER_START：表示某行请求头的开始，如果是请求头结束的那一行，那么这个阶段就会读取到回车换行符，解析状态修改为HeaderParseStatus.DONE
        - HeaderParsePosition.HEADER_NAME：这个阶段表示读取请求头的名字，比如请求头名字host
        - HeaderParsePosition.HEADER_VALUE_START：这个阶段是读取完请求头名之后的一个阶段，这个阶段主要用于跳过冒号和请求头值之间的空白符
        - HeaderParsePosition.HEADER_VALUE：这个阶段开始读取请求头值，会去除右边的空白符
        - HeaderParsePosition.HEADER_MULTI_LINE：这个阶段表示我们已经解析完一行请求头了，可以检查是否继续读下一行，如果数据够的话，break，重新进入HEADER_START阶段
        - HeaderParsePosition.HEADER_SKIPLINE：跳过某行，一般出现非法字符的时候，这行会被跳过


### 1.12 tomcat的映射servlet过程？

connector.getService().getMapper().map(serverName, decodedURI,
version, request.getMappingData());

首先需要查找符合ip地址（ip地址也可以设置通配符，比如后面几个数字一样，前面的不同，tomcat会截取后面部分的ip地址进行匹配）的Host，找到Host之后再找context，
找context稍微复杂一点，需要按照正斜杠从后外前截取字符串，然后去匹配context的名字，比如我们有一个叫test的context，此时我请求路径为test/a/b/c，那么它首先用
test/a/b/c去匹配这个test，很显然不对，然后继续用test/a/b去匹配，以此类推，直到路径变成test就可以匹配到了，匹配之后可能存在多个版本，如果用户没用提供要请求的
context版本，那么寻找最高的那个版本
找到context之后再找wrapper，查找wrapper最主要的有三个方法，第一个是精确匹配路径，第二是通配符路径匹配，第三是后缀扩展名路径匹配，当然了，如果这三种都没有找到
其实还会有欢迎页的查找，默认servlet的查找。
通配符路径匹配：和查找context的方式一样，通过正斜杠分隔，从后往前找。
后缀扩展名路径匹配：找到最后一个正斜杠，然后截取后缀进行精确匹配即可。

容器都匹配好后，开始执行，从Engine容器的管道阀开始，一直到wrapper的管道阀，重点在wrapper的基础管道阀StandardWrapperValve中
- 构建过滤器链条，通过路径匹配（精确，通配符，后缀）和servlet名字进行匹配
- 执行过滤器链条，最后调用servlet的service方法

### 1.13 tomcat使用到的设计模式？

- 观察者模式：比如生命周期事件监听，对每个阶段做一些处理，初始化的时候准备一些配置，注册mbean之类的，然后启动的时候又做些操作，停止的时候又做些啥，比如销毁资源等等。

- 模板模式：比如LifeCycleBase，它实现了Lifecycle，提供一些通用的模板代码，比如修改状态之类的

- 策略者模式：解析xml时不同的规则实现，协议实现，处理请求和响应的adaptor

- 责任链模式：管道阀，过滤器链

- 组合模式：容器与子容器

- 享元模式：对象池，比如NioChannel

- 桥接模式：协议处理与socket端点


