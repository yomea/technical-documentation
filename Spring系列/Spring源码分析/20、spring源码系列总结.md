## 一、容器创建流程

### 1.1 创建与初始化

如果在web.xml中指定了contextClass上下文参数，那么以contextClass为准，创建指定的自定义容器，如果没有指定那么创建默认的XmlWebApplicationContext容器

-》 如果在web.xml中配置了locatorFactorySelector上下文参数，那么根据这个参数指定的位置去加载配置文件（如果没有指定，那么默认加载classpath*:beanRefContext.xml），构建一个ClassPathXmlApplicationContext容器作为父容器

-》 从web.xml配置文件中获取spring配置文件路径，创建StandardServletEnvironment，初步解析spring配置文件路径的占位符（因为此时spring的Environment只能获取系统属性和系统变量），将tomcat的ServletConfig，ServletContext作为属性源设置到StandardServletEnvironment中。

-》 调用ApplicationContextInitializer，可以对容器做些自定义操作，比如添加属于自己的属性源等等

### 1.2 refresh

- prepareRefresh：设置context启动时间，设置激活标记，设置是否关闭标记，并且初始化资源，这里会提供初始化属性资源的机会，并对属性资源的校验机会
- obtainFreshBeanFactory：创建context关联的BeanFactory，这个创建的是DefaultListableBeanFactory，刷新容器，加载配置文件
- prepareBeanFactory：设置el表达式解析器，设置需要在bean属性填充阶段忽略的依赖，设置一些未在容器中添加的依赖（比如BeanFactory，ApplicationContext等，为什么不直接添加到容器中？这些对象具有特殊性，beanFactory中的bean有可能存在相同类型的对象，spring不想与这些相同类型的对象产生歧义，让用户迷惑，将通过特殊接口实现来注入，比如需要ApplicationContext，那么实现ApplicationContextAware即可），注册诸如environment，系统属性，系统变量到单例集合中
- postProcessBeanFactory：添加一些beanFactory后置处理器，注册web相关的依赖，比如Request，Response等，注册web相关的scope，比如RequestScope等
- invokeBeanFactoryPostProcessors：调用BeanFactoryPostProcessor，设置属性文件，这是spring添加BeanDefinition的最后机会，我们注解扫描的工作就是从这里完成，比如ConfigurationClassPostProcessor
- registerBeanPostProcessors：从BeanDefinition中寻找后置处理器
- initMessageSource：初始化国际化资源
- initApplicationEventMulticaster：初始化事件广播器
- onRefresh：默认还未实现，在springboot中被用于启动web容器
- registerListeners：注册事件监听器
- finishBeanFactoryInitialization：初始化eager的bean
- finishRefresh：清理资源缓存，初始化生命周期处理器，启动实现了Lifecycle接口的bean，发布ContextRefreshedEvent事件，注册MBeanServer

## 二、bean的一生

-》 getBean("hello")

-》 去掉&前缀，并将别名转换成原名

-》 先到单例集合中获取，如果没有，那么从早期暴露的集合中获取，如果还是没有，那么从早期暴露工厂集合中获取，如果获取到了将进入1流程，没有获取到就进入2流程
1. 判断是否是FactoryBean，看你传入的beanName是否有&前缀，有的，直接返回FactoryBean本身，没有的，先从缓存中获取对象，如果缓存中没有就调用其getObject方法
2. 从父工厂中尝试查找bean，如果没有继续往下
-》 获取对应BeanDefinition，对BeanDefinition进行合并处理，缓存等操作，如果有明确指定的依赖，需要先创建加载

-》记录当前bean正在创建，如果是单例，将beanName存到singletonsCurrentlyInCreation集合中，如果是非单例的会存到prototypesCurrentlyInCreation集合中

-》 解析BeanDefinition中beanClass，检查配置中设置的look-mehtod与replace-method是否存在，并且如果只有唯一个方法的话，那么将会设置MethodOverrides中的方法
为未被重载，好处就是spring在对这类bean进行代理的时候就不需要进行方法的匹配了
对应的cglib代理拦截分别为LookupOverrideMethodInterceptor，ReplaceOverrideMethodInterceptor

-》 实例化前置处理器，如果实例化前置处理器已经进行了bean的实例化，那么将会直接返回

-》 获取BeanWrapper对象，获取实例化一个BeanWrapper，它是一个bean包装类，使得操作一个bean更加的便捷

-》 判断是否是要通过工厂获取bean，如果是工厂方法的，根据参数筛选工厂方法创建实例，如果是更具构造方法的，通过参数筛选合适的构造方法

-》 从BeanDefinition中查看是否已经有解析好的参数（缓存），如果没有需要进行解析，有三个阶段的参数，第一个阶段是配置解析的原始参数，第二个就是将引用（比如RuntimeBeanReference）转换成真正的对象，第三个就是叫第二阶段的参数通过类型转换成目标类型的值

-》 获取bean的构造器或方法，通过public方法降序，参数长度降序进行排序，循环筛选符合条件的方法

-》 小于配置的构造参数的直接break，给可能符合条件的构造进行权重比对，权重小的优先，它有两个模式

    1、宽松模式：
    a.判断当前方法参数类型与未进行转换的参数值的类型相差多少，循环向上寻找父类，每次加2，直到找到相同的类型或者父类为null,如果定义的方法参
    数类型不是未进行转换的参数值类型的父类，那么直接返回Integer的最大值。
    b.判断当前方法参数类型与未进行转换的参数值的类型相差多少，循环向上寻找父类，每次加2，直到找到相同的类型或者父类为null,如果定义的方法参
    数类型不是未进行转换的参数值类型的父类，那么直接返回Integer的最大值。
    c.获取转换后参数值与未被转换时的参数值进行类型差异性权重，两权重值对比，返回差异性小的权重值
    
    》》 所以对于宽松模式，spring选择参数值的类型越接近于方法参数类型的方法
    
    2、严格模式：转换过类型后的参数值与未进行转换的参数值都能与方法的参数类型形成子与父关系的，那么权重都为Integer.MAX_VALUE - 1024
    转换后的参数值出现与方法参数不是子与父关系的，权重为Integer.MAX_VALUE，未转换的参数值出现与方法参数不是子与父关系的
    权重为Integer.MAX_VALUE - 512
    
    》》 所以对于严格模式，是直接看是否符合父与子，然后比较转换的参数与未进行转换的参数的权重，数值小的优先。

-》 通过筛选到的构造器构造对象

-》 应用合并BeanDefinition后置处理器，可以对BeanDefinition进行修改，不会影响到最原始的BeanDefinition，因为它是被clone出来的一份。但后面属性填充，后置处理，使用到却是这个修改后的BeanDefinition

-》 判断是否是单例对象并且是否允许循环依赖，来决定是否提早曝光这个早期bean实例，默认是允许的，需要提早曝光的bean将被包装成ObjectFactory，存于singletonFactories集合中

-》 开始属性的填充

-》 调用实例化后置处理器，如果返回false，那么将停止spring的自动属性的填充

-》 处理在xml中配置的根据名字自动装备或者根据类型自动装配的属性

-》 过滤忽略的依赖

-》 调用属性后置处理，比如我们的@Autowire，@Value可以在这里处理，进行自动装配

-》 检查依赖，这里的检查依赖指的是属性是否存在依赖，spring中有三种类型的依赖检查
1、ALL 表示pvs中必须存在简单类型和复杂类型的属性，否则抛错
2、simple类型（int,long之类） 表示在pvs中不存在简单类型的属性，那么抛错
3、复杂类型（非简单类型） 表示在pvs中不存在复杂类型的属性，抛错
-》 通过BeanDefinitionValueResolver将一些特殊类型的值转化成未转换的值，比如RuntimeBeanReference，将通过beanName从BeanFactory中获取其实际引用的bean对象

-》 通过BeanWrapper中注册的转换服务和属性编辑器转换属性值，并将转换后的值注入到bean中

-》  处理Aware接口

-》 调用BeanPostProcessor初始化前置处理。

-》 如果实现了InitializingBean，调用afterPropertiesSet方法，如果配置时定义初始化方法，那么反射调用初始化方法

-》 调用初始化后置处理

-》 如果允许早期曝光，那么需要进行循环依赖的版本检查（发生aop代理的，两者的引用不同视为两个版本）

-》 是否实现了DisposableBean接口，如果实现了并且方法名destroy未被注册为排除的销毁方法，那么优先调用接口的destroy方法，如果实现DisposableBean接口但是对应销毁方法被排除或者没有实现DisposableBean接口，那么继续检查用户指定的方法名是否为(inferred)或者没有指定方法名但实现了Closeable，会尝试查找是否有close方法或者shutdown方法

-》 注册单例对象到单例集合，其他scop的注册到它所属的scop

-》 调用销毁前置处理器（如果使用了@PreDestroy注解，那么在这里就会调用被@PreDestroy注解的方法）

-》 调用销毁方法（除了被@PreDestroy注解的方法，如果还指定了其他的销毁方法，那么也会调用）

## 三、AOP

我们平时使用的aop配置通常有以下两种方式：
- <aop:config>
- <aop:aspectj-autoproxy>

### 3.1 config标签的解析流程

-》 解析config标签会注册AspectJAwareAdvisorAutoProxyCreator后置处理器，实现了传统查找advisor的能力，创建aop代理的能力

-》 解析属性proxy-target-class（是否使用cglib代理）与expose-proxy（是否需要在本地线程暴露代理对象，通常适用于方法嵌入调用场景）属性

-》 解析pointcut标签，构建AspectJExpressionPointcut（封装切点表达式，用于匹配目标类和方法是否匹配）的BeanDefinition

-》 解析advisor标签，构建DefaultBeanFactoryPointcutAdvisor（维护着advice与切点）的BeanDefinition，解析advice-ref属性，解析order属性，解析pointcut或者pointcut-ref

-》 解析aspect标签，解析declare-parents（引介增强），解析为DeclareParentsAdvisor（维护这需要实现的接口，advice，classFilter（匹配哪些需要去实现接口））
属性implement-interface（需要实现的接口），types-matching（类型匹配，匹配的类将会动态实现指定接口），default-impl（指定接口的默认实现类），delegate-ref（指定默认实现bean，与default-impl相斥，不能同时存在）

-》 解析advice（before，after等），首先会创建一个MethodLocatingFactoryBean，这个类是用获取切面对象的方法对象，所以我们配置的method属性由它维护，它不会被注册到容器中，它将以被包含的BeanDefinition身份注入到advice对象中，紧接着创建了SimpleBeanFactoryAwareAspectInstanceFactory，用于获取切面对象

-》 接下来根据配置的before标签还是after标签，创建对应的advice，将MethodLocatingFactoryBean与SimpleBeanFactoryAwareAspectInstanceFactory作为属性设置到advice中

-》 然后开始创建AspectJPointcutAdvisor的BeanDefinition，这个advisor维护着advice和pointcut

-》 最后将AspectJPointcutAdvisor注册到BeanFactory中

### 3.2 aspectj-autoproxy标签的解析流程

-》 注册后置处理器AnnotationAwareAspectJAutoProxyCreator，bean实例化前置会判断是否存在用户定义的TargetSource，如果存在会进行代码，如果不存在用户定义的TargetSource，那么会进行bean初始化后置处理，解析注解获取advisor并缓存，在同时指定了config标签的情况下，这个后置处理器优先

-》 解析属性proxy-target-class（是否使用cglib代理）与expose-proxy（是否需要在本地线程暴露代理对象，通常适用于方法嵌入调用场景）属性

-》 解析aspectj-autoproxy子元素，添加includePatterns

### 3.3 创建aop代理流程

-》 通过传统方式，从BeanFactory中获取所有advisor类型的bean，然后通过正则includePattern进行过滤

-》 通过注解的方式获取，循环所有已经注册的BeanDefinition，先通过正则过滤调用不符合的bean，然后判断是否有@Aspect注解，如果是，那么创建AspectMetadata

-》 对比切面实例化模式，通常我们在标注@Aspect时不会指定它的value值，所以spring将使用默认的实例化模式PerClauseKind.SINGLETON

-》 如果我们配置了@Aspect("perthis(this(com.test.service.IUserService))")，那么它的实例化模式为PerClauseKind.PERTHIS，对应的切面不允许是单例的

-》 如果我们配置了@Aspect("pertarget(target(com.test.service.IUserService))")，那么它的实例化模式为PerClauseKind.PERTARGET，对应的切面不允许是单例的

-》 获取切面对象除了被@Pointcut标注外的所有方法，然后进行排序，排序规则：以注解数组Around, Before, After, AfterReturning, AfterThrowing下标排序，
如果是没有标注以上数组里的注解的方法，返回注解数组长度，如果排序号相同的，再按方法名排序

-》 循环上面筛选的method，解析其上的advice注解，封装成AspectJExpressionPointcut

-》 创建InstantiationModelAwarePointcutAdvisorImpl，在这个类的构造器中额外做了一些事情，比如检查当前切面的实例化模式是否为perthis,pertarge,perwith，如
果是的话，需要将@Aspect注解中的切点记性union，标记advisor的lazy属性为true

-》 如果实例化模式是singleton，那么在构造器中就会进行advice的实例化，解析绑定切点参数，通常我们使用的就是singleton的切面

-》 根据advice类型创建对应的advice对象，比如AspectJMethodBeforeAdvice，AspectJAfterAdvice等等

-》 解析切面方法的参数，记录JoinPoint，ProceedingJoinPoint，StaticPart，这三个类型的参数任选其一且必须为第一个参数，
其中ProceedingJoinPoint只能用在Around类型的advice中，接着解析return参数，throw参数，最后解析切点参数

-》 如果存在perthis,pertarge,perwith模式的切面，那么会额外添加一个SyntheticInstantiationAdvisor到最advisor集合最前面，这个advisor在目标bean符合它的PerClause切点时，会提前获取一遍切面实例

-》 解析切面字段，查看是标有@DeclareParents注解，然后创建DeclareParentsAdvisor

-》 找出所有的advisor之后，进行匹配，筛选出可以用在当前bean上的advisor

-》 如果advisor中存在AspectJ advice（spring的aop代理，使用特殊编译，运行时编织的不属于这类），那么添加ExposeInvocationInterceptor.ADVISOR到最前面，这个advisor的advice会在调用时在本地线程中设置MethodInvocation对象

-》 将筛选好的advisor进行排序，使用AnnotationAwareOrderComparator排序

-》 构建通用的advice，默认是没有的，我们可以手动配置AnnotationAwareAspectJAutoProxyCreator，设置属性interceptorNames即可

-》 创建代理

### 3.4 构建拦截器与执行流程

-》 循环每个advisor，检查是否已经匹配过，如果没有匹配过，会进行动态匹配，对于PointcutAdvisor，它的方法还是要再次进行匹配的，取出匹配的advisor的advice，如果是已经实现了MethodInterceptor的advice，那么直接添加到拦截器链集合中，如果不是，使用适配器去适配，比如MethodBeforeAdvice需要适配为MethodBeforeAdviceInterceptor

-》 封装成ReflectiveMethodInvocation，通过下标的方式记录当前驱动所在的拦截器位置

-》 假设我们配置Around, Before, After, AfterReturning, AfterThrowing，引介增强DelegatePerTargetObjectIntroductionInterceptor 所有的advice

-》 执行Around，创建ProceedingJoinPoint对象，其实例为MethodInvocationProceedingJoinPoint

-》 绑定JoinPoint，切点参数

-》 调用around切面方法

-》 用户调用ProceedingJoinPoint的proceed方法进入下一个Before切面方法

-》 绑定JoinPoint，切点参数

-》 调用before切面方法，调用methodInvocation的proceed方法进入After

-》 执行after，直接调用methodInvocation的proceed方法进入AfterReturning

-》 执行AfterReturning，直接调用methodInvocation的proceed方法进入AfterThrowing

-》 执行AfterThrowing, 直接调用methodInvocation的proceed方法

-》 调用引介增强拦截器，判断定义当前方法的类是否是用户指定的接口，如果是，那么获取默认实现去执行这个方法

-》 执行目标方法

-》 如果发生错误，会执行AfterThrowing的切面方法，如果没有发生错误，那么会调用AfterReturning的切面方法，然后再调用After方法

-》 如果方法执行后的返回值是目标对象，那么替换为返回代理对象

## 四、事务

spring使用事务有三种方式，第一种使用xml配置的方式配置，第二种使用注解的方式配置，第三种使用编程的方式

### 4.1 解析advice标签流程

-》 创建TransactionInterceptor的BeanDefinition，给这个拦截器的属性transactionManager设置事务管理器

-》 解析attributes标签，解析method标签，包装为RuleBasedTransactionAttribute（维护着方法pattern，事务隔离机制，传播性，回滚规则等信息），最后以
Map<方法Pattern，RuleBasedTransactionAttribute>为属性注入到NameMatchTransactionAttributeSource中，NameMatchTransactionAttributeSource一看就是通过方法名进行匹配对应的事务属性

对于这种方式的advice，在config标签中会被封装成DefaultBeanFactoryPointcutAdvisor

### 4.2 解析annotation-driven标签流程

-》 注册TransactionalEventListenerFactory（这个类在refresh方法中调用），用于处理被@TransactionalEventListener修饰的方法，
创建ApplicationListenerMethodTransactionalAdapter类型的ApplicationListener，注册为TransactionSynchronization，用于监听事物的提交，回滚等事件

-》 创建InfrastructureAdvisorAutoProxyCreator后置处理器，它只筛选infrastructure类型的Advisor

-》 构建AnnotationTransactionAttributeSource的BeanDefinition

-》 构建TransactionInterceptor的BeanDefinition，将AnnotationTransactionAttributeSource设置到TransactionInterceptor中

-》 构建BeanFactoryTransactionAttributeSourceAdvisor的BeanDefinition

### 4.3 事务调用过程

-》 获取事务属性TransactionAttribute，如果是xml方式的，那么就是通过方法名匹配获取，如果是注解的，那么动态解析方法上的@Transactional注解

-》 获取事务管理器

-》 获取事务，先检查本地线程中是否存在事务，如果存在，根据所指定的事务传播性处理，是挂起还是抛错，所谓挂起就是将原来的事务获取出来进行局部变量暂存，
然后清除本地线程中事务信息，按照事务传播性判断是否需要重新创建事务

-》 假设我们使用的是默认的事务传播性TransactionDefinition.PROPAGATION_REQUIRED，那么如果存在事务的话，就沿用这个事务，如果不存在事务，那么新创建一个
事务，从数据源中获取连接，然后设置隔离级别，是否只读，超时时间，设置手动提交，最后将连接绑定到本地线程中

-》 初始化事务同步

-》 如果发生错误，那么将挂起的事务进行恢复

-》 如果是TransactionDefinition.PROPAGATION_NESTED传播属性，那么判断是否允许使用嵌套事务，允许的话，会创建保存点

-》 创建TransactionInfo，维护事务管理器，事务属性，连接点

-》 提交事务，并且触发提交前事件，提交后事件，调用事务同步监听器

编程方式的事务管理，我们只要注入TransactionTemplate即可，同时TransactionTemplate也实现了TransactionDefinition

## 五、类介绍

- XmlWebApplicationContext：一个主要用于读取xml配置的web容器，可以使用web容器中一些资源
- ClassPathXmlApplicationContext：从classPath路径下读取xml的容器
- ApplicationContextInitializer：容器初始化器，在容器创建后，进行调用，用户可以对这个容器做些自己的定制
- StandardServletEnvironment：用于web的环境对象，额外提供ServletConfig，ServletContext属性源
- XmlBeanDefinitionReader：用于读取xml的辅助类
- PathMatchingResourcePatternResolver：匹配Ant形式的路径，用户获取资源
- AntPathMatcher：Ant风格的匹配器
- ConfigurationClassPostProcessor：用于解析注解配置
- NamespaceHandler：定义初始化标签解析器，定义获取BeanDefinition以及装饰BeanDefinition的能力
- NamespaceHandlerSupport：提供查找具体标签解析器的能力，用户只需继承它，实现init方法，与具体的标签解析器就可以了
- BeanDefinitionParser：解析标签的具体实现
- BeanDefinitionDecorator：解析标签的装饰者，比如bean标签中设置了<aop:scoped-proxy />那么会被装饰成对应scope的一个代理类
- AnnotationAwareAspectJAutoProxyCreator：用于代理bean的后置处理
- ClassPathBeanDefinitionScanner：扫描class路径下的类
- ConditionEvaluator：条件执行器，用于过滤条件，对应使用的注解@Condition
- MetadataReader：元数据读取器，内部使用ASM字节码框架实现字节码的读取，不会对类进行加载
- MetadataReaderFactory：用于创建MetadataReader的工厂
- ImportSelector：引入选择器，可以根据引入自己的注解元数据来决定需要加载什么样的配置
- BeanWrapper：包装bean对象，提供更便捷的属性访问
- BeanDefinition：bean定义信息
- AspectJAwareAdvisorAutoProxyCreator：传统查找advisor的能力，创建aop代理的能力
- AspectJExpressionPointcut：封装切点表达式，用于匹配目标类和方法是否匹配
- DefaultBeanFactoryPointcutAdvisor：用于需要从BeanFactory获取advice对象的advisor
- DeclareParentsAdvisor：用于引介增强的advisor
- MethodLocatingFactoryBean：用于xml配置，作为属性设置到advice中，通过bean的创建流程获取对象切面方法对象
- SimpleBeanFactoryAwareAspectInstanceFactory：获取切面实例
- AspectJPointcutAdvisor：维护着切面工厂，切面方法实例，切点
- AspectMetadata：切面元数据，用于获取切面类型，切面实例化模式等等
- MethodInvocation：维护这调用上下文，比如调用的方法实例，参数等信息，调用方法，在aop中有个实现类ReflectiveMethodInvocation，可以驱动链条执行
- JdkDynamicAopProxy：jdk动态代理逻辑实现
- ObjenesisCglibAopProxy：cglib动态代理逻辑实现
- RuleBasedTransactionAttribute：事务属性定义
- NameMatchTransactionAttributeSource：通过方法名匹配获取事务属性定义
- TransactionStatus：事务的状态，比如是否已经提交，是否需要回滚
- PlatformTransactionManager：事务管理器，用于提交，回滚，创建事务
- TransactionSynchronizationManager：用于维护本地线程中的事务资源，同步监听器
- TransactionInterceptor：事务拦截器
- TransactionInfo：事务信息，维护事务管理器，事务状态，事务

## 设计模式

- 参观者模式：ClassVisitor，对BeanDefinition进行读取，而不进行修改
- 责任链模式：aop
- 工厂模式：BeanFactory就是一个工厂,
- 装饰器模式：容器与BeanFactory其实就是一种装饰器模式
- 观察者模式：比如ApplicationListener
- 代理模式：aop
- 策略者模式：比如根据不同的配置，使用不同的代理方式（JDK，Cglib），各种属性的转换器
- 适配器模式：比如aop中的Before通知，它需要被适配成MethodIntercepter
- 模板模式：比如容器中一些方法，调用了一个抽象方法，具体的实现交由子类实现，再比如AbstractAutoProxyCreator，具体获取advisor的方法由子类实现
- 解释器模式：切点表达式的匹配

## 循环依赖的处理
举个例子，有两个bean分别为A和B，他们的依赖关系为A -> B -> A
分以下情况
- A与B的生命周期为单例，并且A与B互相依赖的对方都不在创建bean的构造参数中

```
    @Component
    public class A {
        @Autowire
        private B b;
    }
    
    @Component
    public class B {
        @Autowire
        private A a;
    }
```
1. 将A存入singletonsCurrentlyInCreation集合，表示单例A正在创建。
2. 创建A的早期对象，然后包装成ObjectFactory存放到singletonFactories集合中
3. 准备填充属性，发现属性中依赖了B，于是就去创建B，填充B的属性a时发现依赖了A，那么就从earlySingletonObjects集合中找，如果没有再到singletonFactories集合中找，
   找到后调用getObject方法将早期对象A取出，然后缓存到earlySingletonObjects集合中
4. 如果A没有被自定义初始化前后置处理器重新构建，那么解决了循环依赖，但是如果A发生了重新构建，此时就出了问题，最终的单例A与注入到B的A是不同版本对象，如果我们没有自定义spring的容器，修改属性
   allowRawInjectionDespiteWrapping为true（默认为false）那么会抛出多版本依赖的错误

- A与B的生命周期为单例，并且A创建bean时的构造函数依赖B，创建B的构造参数不依赖A

```
    @Component
    public class A {
    
        @Autowire
        public A (B b) {
            
        }
    }
    
    @Component
    public class B {
        @Autowire
        private A a;
    }
```
1. 将A存入singletonsCurrentlyInCreation集合，表示单例A正在创建。
2. 通过AutowiredAnnotationBeanPostProcessor解析A的构造函数，解析构造参数（如果是xml方式，将直接反射获取A的所有构造器，通过在xml中配置的构造参数筛选）
3. 解析构造参数时，发现依赖B，那么就去创建B，B填充属性的时候发现依赖了A，那么就从earlySingletonObjects集合中找，如果没有再到singletonFactories集合中找
   发现也没有找到，然后继续走创建A的流程，发现singletonsCurrentlyInCreation中已经存在A了，表示这个A正在创建，抛出正在创建A的错误。

> 如果我们先创建的是B呢？

1. 将B存入singletonsCurrentlyInCreation集合，表示单例B正在创建。
2. 创建B的早期对象，然后包装成ObjectFactory存放到singletonFactories集合中
3. 准备填充属性，发现属性中依赖了A，于是就去创建A
4. 通过AutowiredAnnotationBeanPostProcessor解析A的构造函数，解析构造参数（如果是xml方式，将直接反射获取A的所有构造器，通过在xml中配置的构造参数筛选）
5. 解析构造参数时发现依赖了B，那么就从earlySingletonObjects集合中找，如果没有再到singletonFactories集合中找，找到后调用getObject方法将早期对象A取出，
   然后缓存到earlySingletonObjects集合中。
6. 如果B没有被自定义后置处理器代理，那么解决了循环依赖，但是如果B发生了代理，此时就出了问题，最终的单例B与注入到A的B是不同版本对象，如果我们没有自定义spring的容器，修改属性
   allowRawInjectionDespiteWrapping为true（默认为false）那么会抛出多版本依赖的错误

- A与B的生命周期为单例，并且A创建bean时的构造函数依赖B，创建B的构造参数也依赖A

```
    @Component
    public class A {
    
        @Autowire
        public A (B b) {
            
        }
    }
    
    @Component
    public class B {
        @Autowire
        public B (A a) {
            
        }
    }
```
1. 将A存入singletonsCurrentlyInCreation集合，表示单例A正在创建。
2. 通过AutowiredAnnotationBeanPostProcessor解析A的构造函数，解析构造参数（如果是xml方式，将直接反射获取A的所有构造器，通过在xml中配置的构造参数筛选）
3. 解析构造参数时，发现依赖B，那么就去创建B
4. 通过AutowiredAnnotationBeanPostProcessor解析B的构造函数，解析构造参数
5. 解析B的构造参数时发现依赖了A，那么就从earlySingletonObjects集合中找，如果没有再到singletonFactories集合中找
   发现也没有找到，然后继续走创建A的流程，发现singletonsCurrentlyInCreation中已经存在A了，表示这个A正在创建，抛出正在创建A的错误。

- A与B的生命周期为非单例，因为非单例模式的对象spring不会像单例那样进行早期暴露，所以在创建对象的时候只要在prototypesCurrentlyInCreation集合中发现正在创建，那么
  就会抛出创建异常


- A为单例，B非单例，并且B未被标记为范围代理（可以通过<aop:scope-proxy />代理一个非单例对象，会产生一个目标对象和一个代理对象，代理对象是单例的，
  使用定义的原始beanName 但是 目标对象依然是非单例的，并且其beanName会加上scopedTarget.前缀）

```
    @Component
    public class A {
        @Autowire
        private B b;
    }
    
    @Component
    public class B {
        @Autowire
        private A a;
    }
```
1. 将A存入singletonsCurrentlyInCreation集合，表示单例A正在创建。
2. 创建A的早期对象，然后包装成ObjectFactory存放到singletonFactories集合中
3. 准备填充属性，发现属性中依赖了B，于是就去创建B，填充B的属性a时发现依赖了A，那么就从earlySingletonObjects集合中找，如果没有再到singletonFactories集合中找，
   找到后调用getObject方法将早期对象A取出，然后缓存到earlySingletonObjects集合中
4. 如果A没有被自定义后置处理器代理，那么解决了循环依赖，但是如果A发生了代理，此时就出了问题，最终的单例A与注入到B的A是不同版本对象，如果我们没有自定义spring的容器，修改属性
   allowRawInjectionDespiteWrapping为true（默认为false）那么会抛出多版本依赖的错误

> 如果先创建B又会如何

1. 将B存入prototypesCurrentlyInCreation集合，表示非单例bean B正在创建。
2. 准备填充属性，发现属性中依赖了A，于是就去创建A
3. 将A存入singletonsCurrentlyInCreation集合，表示单例A正在创建。
4. 创建A的早期对象，然后包装成ObjectFactory存放到singletonFactories集合中
5. 填充属性，发现依赖了B，于是又去创建B，因为B是非单例的，所以会创建B，此时spring发现B已经存在prototypesCurrentlyInCreation集合中，抛出bean创建失败的错误

。。。。。。其他情况自行分析