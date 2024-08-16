## 一、tomcat目录

catalinaHome：表示tomcat产品的安装目录

catalinaBase：表示tomcat实例的目录，通常是放置配置文件，jar包，web应用的目录，不过在通常情况下和catalinaHome是一样的


## 二、类加载器

tomcat中有三个默认的类加载器，他们的层级结构如下：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/118dbb85cb507da7e49ef6a2ecc753ed.png)

那么这几个类加载的作用是啥？

他们都是URLClassLoader的实例，他们的加载路径可以通过配置文件配置，可以设置系统属性catalina.config指定配置文件的路径，如果没有配置，那么默认加载\${catalinaBase}/lib下的catalina.properties配置文件

- commonClassLoader：这个类加载默认加载 common.loader="\${catalina.base}/lib","\${catalina.base}/lib/*.jar","\${catalina.home}/lib","\${catalina.home}/lib/*.jar"

- serverClassLoader：默认配置中没有配置路径，它的父级加载器是commonClassLoader，这个加载是tomcat的核心加载器，伴随着整个tomcat的启动，用于加载tomcat自身使用的核心类，工具包，像tomcat的session管理，Server，connector等都是用的这个加载（加载conf/server.xml中的配置时使用的就是这个类加载器），当然实际上它没有配置路径的话，它的实例就是commonClassLoader，如果我们自定义了一些管道阀，监听器什么的，那么我们可以设置这些类的加载路径，或者直接扔到commonClassLoader的加载路径也行。

- sharedClassLoader：默认配置中没有配置路径，它的父级加载器是commonClassLoader，这个加载在初始化Boostrap的时候，会被设置到org.apache.catalina.startup.Catalina#parentClassLoader属性上，作为Web级别的类加载器的父级加载器，它所加载的类在所有web应用中共享




## 三、类介绍

- Digester：这是一个实现了DefaultHandler2的sax xml解析处理器，用于解析tomcat的配置文件，它是通过匹配标签，以规则的方式进行解析xml，比如ObjectCreateRule等

- LifecycleListener：生命周期事件监听，比如初始化，启动，停止

- ContainListener：容器事件监听器，比如添加容器等事件

- StandardRoot：实现了WebResourceRoot接口，内部维护多个WebResourceSet（WebResource的集合），这个类在StandardContext使用了，具有维护和查找资源的能力，挂载路径的能力

- StandardManager：session管理器，使用ConcurrentHashMap实现，启动context容器的ContainerBackgroundProcessor，后台线程监控session失效

。。。。。。

## 四、使用的设计模式

- 观察者模式：比如生命周期事件监听，对每个阶段做一些处理，初始化的时候准备一些配置，注册mbean之类的，然后启动的时候又做些操作，停止的时候又做些啥，比如销毁资源等等。

- 模板模式：比如LifeCycleBase，它实现了Lifecycle，提供一些通用的模板代码，比如修改状态之类的

- 策略者模式：解析xml时不同的规则实现，协议实现，处理请求和响应的adaptor

- 责任链模式：管道阀，过滤器链

- 组合模式：容器与子容器

- 享元模式：对象池，比如NioChannel

- 桥接模式：协议处理与socket端点
  。。。。。。

## 五、发布流程

### 5.1 发布

首先发布conf/catalina/主机名/xml描述 -》 发布webapps目录下的war包 -》 发布webapps目录下的目录

添加context容器并且启动context时进行fixDocBase，如果是war包，那么会进行解压到tomcat的webapps目录下

### 5.2 context的启动过程简介

context 启动时，会解析web.xml，web-fragment.xml创建ServletDef，FilterDef，监听器直接用一个set装类名即可

-》加载META-INF/services/下的配置文件，配置文件名字为javax.servlet.ServletContainerInitializer，去除#注解，获取类名处理注解@WebServlet,@WebFilter,@WebListener,并根据HandleType是注解还是类型对资源进行筛选设置到initializerClassMap<ServletContainerInitializer, Set<Class<?>>>中 -》 合并web配置片段，host级别配置，全局配置

-》 调用ServletContainerInitializer，通常我们可以利用这个初始化器动态添加filter，servlet

-》创建监听器，如果监听的时候容器启动时间，那么会立马触发

-》启动session管理器，启动时主要是加载系统临时目录中的SESSION.ser文件，将之前关闭tomcat钝化的session进行反序列化

-》创建filter对象，调用其init方法

-》启动loadOnStartup大于0的，内部使用TreeMap进行排序，key值为loadOnStartup，value为Wrapper集合通过InstanceManager反射创建Servlet，然后调用其初始化方法


### 5.3 注册

Mapper 包含MappedHost 包含 一个contextList 包含 多个context 包含 多个ContextVersion(通配符匹配，扩展名匹配，精确匹配)

注册采用路径排序，查找通过二分法查找

### 5.4 通信

创建Acceptor组 reactor用于接受请求
创建Poller组 reactor用于注册请求，并处理请求，当发生一个事件，比如读事件，那么先需求这个通道的读事件，然后交由线程池处理，解析http，如果数据不够，会继续注册读事件到poller持有的select中

Acceptor接收的请求，通过原子轮询的方式注册到poller的selector上

这种reactor模型和netty的是一样的

### 5.5 http解析

如以下http协议：

```
GET /index.html?name=xxx&password=123456 HTTP/1.1
Host: www.xxxx.com
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/56.0.2924.87 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8,en;q=0.6

请求体数据
```


当触发read事件时，tomcat需要对这个请求的http协议进行解析，首先先解析协议，分以下阶段解析：
- 第一阶段：循环跳过空白符
- 第二阶段：解析方法，比如这里的GET方法
- 第三阶段：跳过空白符
- 第四阶段：解析uri和查询字段，这里的uri就是/index.html，查询字段就是name=xxx&password=123456
- 第五阶段：跳过空白符
- 第六阶段：解析协议，这里就是HTTP/1.1

如果在读取的过程中，数据不够，无法完整的读取完开始行，那么将socket状态设置为SocketState.LONG，tomcat重新注册读事件，等待下次触发read事件

读取完开始行之后，就开始请求头的读取了，解析请求头，分为这么几个解析状态：
- HeaderParseStatus.DONE：表示解析完成
- HeaderParseStatus.HAVE_MORE_HEADERS：表示还有请求头需要继续解析
- HeaderParseStatus.NEED_MORE_DATA：表示数据不够，需要继续监听read事件获取数据

解析位置有这么几个状态：
- HeaderParsePosition.HEADER_START：表示某行请求头的开始，如果是请求头结束的那一行，那么这个阶段就会读取到回车换行符，解析状态修改为HeaderParseStatus.DONE
- HeaderParsePosition.HEADER_NAME：这个阶段表示读取请求头的名字，比如请求头名字host
- HeaderParsePosition.HEADER_VALUE_START：这个阶段是读取完请求头名之后的一个阶段，这个阶段主要用于跳过冒号和请求头值之间的空白符
- HeaderParsePosition.HEADER_VALUE：这个阶段开始读取请求头值，会去除右边的空白符
- HeaderParsePosition.HEADER_MULTI_LINE：这个阶段表示我们已经解析完一行请求头了，可以检查是否继续读下一行，如果数据够的话，break，重新进入HEADER_START阶段
- HeaderParsePosition.HEADER_SKIPLINE：跳过某行，一般出现非法字符的时候，这行会被跳过

读取完请求头后，设置请求头的结束位置，这个位置又是请求体的开始位置

### 5.6 映射容器

```
connector.getService().getMapper().map(serverName, decodedURI,version, request.getMappingData());
```
通过主机地址二分法找到Host（分忽略大小写的精确匹配和去掉*开头的精确匹配），如果没有尝试看看有没有配置默认的host -》 找到host后再找context，一样的通过二分法查找，找到后通过前缀匹配的方式匹配，如果失败，那么以/进行从后往前分割，找到符合的context，比如有一下context

从小到大为：a    a/b    a/d
然后我们有个请求是a/c，二分法发现这个uri大于a/b小于a/d，那么取a/b进行尝试a/c是否为a/b的前缀，显然不是，那么从后往前匹配，把原来的a/c退化成a进行匹配，很显然
可以匹配到名为a的context，如果实在是找不到，那么就找那个没有名字的context，名字就是ROOT的

由于可能存在多个版本的问题，精确匹配版本，匹配不到的，用最后的最高的那个版本

-》 找到context之后，再找servlet，先通过精确匹配路径，再次通配符前缀匹配，没找到看下是否是没有设置servlet路径的uri，如果是，那么猜测是要请求根路径，将原来uri加上正斜杠，设置重定向路径，让浏览器重定向请求，如果设置了servletPath，没有匹配到的，再通过扩展名找，还没有，通过欢迎页找，欢迎页也分为几个小的子流程，那就是精确匹配，通配符前缀匹配以及物理文件，还是没有找到的话，那么就使用默认servlet

找到之后就可以进行处理了，通过各个容器的管道阀，比较重要的就是StandardWrapperValve，这个管道阀做了比较多的事情，通过路径的精确，前缀，扩展名匹配路径，通过servletName匹配servlet的方式创建过滤器链，过滤器链。

