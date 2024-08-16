## 一、servlet初始化流程

-》 初始化servlet（org.springframework.web.servlet.HttpServletBean.init()）

-》 读取配置的属性，通过BeanWrapperImpl的方式设置值

-》 创建spring容器，调用refresh方法

-》 触发ContextRefreshedEvent事件，调用org.springframework.web.servlet.FrameworkServlet.onApplicationEvent(ContextRefreshedEvent)

-》 调用org.springframework.web.servlet.DispatcherServlet.onRefresh(ApplicationContext)方法

-》 初始化文件上传解析器

-》 初始化国际化解析器，如果在容器中未找到，也就是用户没有配置，那么加载DispatcherServlet.properties配置文件，这个配置文件里配置了
一个AcceptHeaderLocaleResolver解析，这个解析器从request中获取Locale

-》 初始化主题解析器，如果没有配置，那么从DispatcherServlet.properties配置

-》 初始化mapping处理器，先从BeanFactory中查出所有HandlerMapping，然后进行排序，如果没有就获取配置文件DispatcherServlet.properties中的处理器

-》 初始化HandlerAdapter，从BeanFactory中获取所有HandlerAdapter，然后排序，如果没有就从配置文件DispatcherServlet.properties中找

-》 初始化异常处理器，从BeanFactory中找出所有的HandlerExceptionResolver，然后排序，如果没有就从配置文件DispatcherServlet.properties中找

-》 初始化RequestToViewNameTranslator，这个转换器的作用就是将request请求的uri转换成视图名，如果没有就从配置文件DispatcherServlet.properties中找

-》 初始化ViewResolver，从BeanFactory中找出所有的ViewResolver，并且排序，如果没有就从配置文件DispatcherServlet.properties中找

-》 初始化FlashMapManager，用于管理flashMap的恢复与更新以及保存，用于重定向保存参数


## 二、请求流程

=》 构建LocaleContext，一个用于获取Locale的上下文，它将被绑定到本地线程中，有两种绑定方式，一种是仅仅是绑定在当前线程中，另一种是绑定到线程中之后，如果这个线程创建了子线程，那么这个子线程会继承这个属性，默认只是绑定到当前线程中

-》 构建ServletRequestAttributes（用于存储request与session范围的属性）实例，它将会被绑定到本地线程中

-》 创建异步管理器WebAsyncManager，用于处理异步调用请求，并且添加RequestBindingInterceptor拦截器，这个拦截器主要用于在异步的时候可以构建LocaleContext，
ServletRequestAttributes和上面两个流程的创建方式是一样的

-》 尝试从session中获取List<FlashMap>，然后移除过期的FlashMap（继承了HashMap，这里面的值，我们可以在controller中设置RedirectAttributes参数）

-》 根据request参数筛选出符合条件的FlashMap，匹配规则：
org.springframework.web.servlet.FlashMap.targetRequestPath：匹配请求路径
org.springframework.web.servlet.FlashMap.targetRequestParams：它的key在Request参数中存在并且对应的值在request中存在，这个map是request中map的子集，
然后进行排序，获取第一个匹配的FlashMap

以上两个成员变量没有设置的话，那么直接认为是匹配的

-》 从List<FlashMap>中移除匹配到的FlashMap，然后设置回request的session中，将匹配到的FlashMap以input模式设置到Request中

-》 给request设置output模式的空FlashMap，以便后面解析RedirectAttributes使用

-》 判断request是否为enctype="multipart/form-data"，如果是，解析为 input字段名 -> List<MultipartFile>

-》 通过HandlerMapping获取handler，构建成拦截器链

-》 用过handler获取HandlerAdapter

-》 检查资源是否已经修改，如果没有修改的，设置304编码，并返回

-》 调用拦截器的preHandle方法，如果拦截器返回false，那么立马调用afterCompletion方法，从当前返回false的拦截器开始向前执行

-》 调用handler处理请求

-》 如果异步请求开始了，那么直接返回

-》 如果返回了ModelAndView并且没有设置视图时，通过viewNameTranslator解析视图名

-》 调用拦截器的postHandle方法，从后向前执行

-》 如果发生了错误，调用全局异常处理器处理，如果没有发生错误，或者返回的是错误视图，那么渲染视图

## 三、RequestMappingHandlerMapping

### 3.1 初始化流程（实现了InitializingBean接口）

-》 创建RequestMappingInfo.BuilderConfiguration，用于后面创建请求映射信息RequestMappingInfo

-》 获取所有被@Controller或者@RequestMapping修饰的类

-》 循环每个类上（包括每个类的父类，接口）的方法，符合以下条件的方法会被过滤掉：Object定义的方法，桥方法，未被@RequestMapping修饰的方法

-》 通过方法上的@RequestMapping注解解析方法，需要的参数，映射名，要求的请求头等等，构建成RequestMappingInfo

-》 将方法上@RequestMapping注解创建的RequestMappingInfo与类上的@RequestMapping注解创建的RequestMappingInfo进行联合

-》 为每个@RequestMapping方法创建HandlerMethod对象，并且注册到MappingRegistry中

### 3.2 寻找handler过程

-》 调用getHandler方法，先以直接url的方式从MappingRegistry中获取匹配的HandlerMethod

-》 匹配请求方法，请求参数，请求头，consumer媒体类型，produces媒体类型等等

-》 然后进行排序，如果第一个匹配的handlermethod和第二个handlermethod相等，那么抛错

-》 根据pattern路径与请求路径解析路径变量，比如pattern路径为/test/{a}，请求路径为/test/0;arg2=1;arg3=2,3,4,5,6

-》 那么路径变量为a = "0;arg2=1;arg3=2,3,4,5,6"，如果开启了矩阵，那么路径变量变为a = "0"， 矩阵变量为 a = {arg2:[1],arg2:[2,3,4,5,6]}

-》 返回匹配到的HandlerMethod，通过AntPatchMatcher进行拦截器url的匹配，组装成一个处理器执行链（HandlerExecutionChain ）

## 四、RequestMappingHandlerAdapter

### 4.1 初始化流程（实现了InitializingBean接口）

-》 初始化获取@ControllerAdvice标注的bean开始

-》 获取所有的BeanDefinition，过滤出被@ControllerAdvice注解标注的类，然后包装成ControllerAdviceBean

-》 从获取到的ControllerAdviceBean解析出被@ModelAttribute标注且未被@RequestMapping标注的方法

-》 解析出被@InitBinder标注的方法

-》 获取目标类继承了RequestBodyAdvice的ControllerAdviceBean

-》 获取目标类继承了ResponseBodyAdvice的ControllerAdviceBean

-》 初始化获取@ControllerAdvice标注的bean结束

-》 获取参数解析器，用于解析HandlerMethod上的参数

-》 获取初始化绑定参数解析器，用于解析被@InitBinder注释的方法上的参数

-》 获取返回值处理器，用于处理返回值

### 4.2 调用handler流程

-》 获取当前handler上的initBinder方法和被@ControllerAdvice注解的类的initBinder方法，创建InvocableHandlerMethod

-》 创建数据绑定工厂ServletRequestDataBinderFactory

-》 获取当前处理handler的ModelAttribute方法和被@ControllerAdvice注解的类的ModelAttribute方法，创建InvocableHandlerMethod

-》 把当前将要处理的HandlerMethod包装成ServletInvocableHandlerMethod

-》 创建ModelAndViewContainer，将InputFlashMap参数设置到ModelAndViewContainer中

-》 如果当前handler被@SessionAttributes注释，那么将会把@SessionAttributes注解指定的属性从session中获取，然后合并到ModelAndViewContainer中

-》 开始调用ModelMethod，会过滤掉参数被@ModelAttribute注释的并且指定的属性名（没有指定的把方法参数名作为属性名）在ModelAndViewContainer中不存在的model方法

-》 调用ModelMethod方法，将返回值以属性名为key，返回值为value的形式存入ModelAndViewContainer中

-》 为WebAsyncManager（servlet3.0支持的异步管理器）设置拦截器处理器，做些超时处理，异步前置处理和后置处理等等

-》 **解析方法参数，筛选参数解析器，通过参数解析器进行解析，比如被@RequestBody标注的，如果参数复杂对象，那么
会使用webDataBinder进行处理，将request中的参数设置到对应的bean中，对应的bean会被包装成BeanWrapperImpl**

-》 调用HandlerMethod方法

-》 筛选返回值处理器，通过返回值处理器处理返回值，比如被@ResponseBody标注的，会使用RequestResponseBodyMethodProcessor进行处理。

## 类介绍

- RequestAttributes：定义获取属性，移除属性，添加属性，注册请求完成回调，定义了request scope与session scope

- AbstractRequestAttributes：实现了RequestAttributes接口的一部分方法

- ServletRequestAttributes：继承了AbstractRequestAttributes

- RequestMappingHandlerMapping：处理被@Controller或者@RequestMapping注释的类，将他们被@RequestMapping注释的方法包装成HandlerMethod，将映射信息包装成
  RequestMappingInfo，以key为RequestMappingInfo，value为HandlerMethod的方式注册到MappingRegistry中去

- RequestMappingInfo：请求映射信息

- HandlerMethod：把被@RequestMapping注释的方法包装成HandlerMethod

- RequestCondition：请求条件，比如实现类ProducesRequestCondition

- ContentNegotiationManager：内容协商管理器，用户获取媒体类型

- ContentNegotiationStrategy：内容协商策略，spring内置有根据扩展符获取媒体类型的，有根据请求头获取媒体类型的

- HandlerMethodArgumentResolver：参数解析器，用于解析方法上的参数，比如RequestResponseBodyMethodProcessor，这个解析器用于解析被@RequestBody标注的参数

- HttpMessageConverter：消息转换器，比如在RequestResponseBodyMethodProcessor中，有个转换器MappingJackson2HttpMessageConverter，
  我们可以把请求体的数据以json的格式转换成对象

- HandlerMethodReturnValueHandler：返回值处理器，比如RequestResponseBodyMethodProcessor，用于处理被@ResponseBody标注的方法的返回值

- WebBindingInitializer：web数据绑定初始化器，用于初始化一些属性

- WebDataBinder：web数据绑定器，用于创建BeanWrapperImpl，用于设置转换服务，属性编辑器

- BeanWrapper：对象包装器，用于更便捷的操作对象，可以通过路径表达式设置属性

- WebAsyncManager：web异步请求管理器

- HandlerInterceptor：拦截器

- HandlerExceptionResolver：异常解析器

- RequestMappingHandlerAdapter：用于处理类型为HandlerMethod的处理器


