## 一、SqlSessionFactoryBean概览

与spring进行整合的时候，通常我们会配置以下信息

```
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"></property>
        <property name="configLocation" value="classpath:mybatis/mybatis-config.xml"></property>
        <!-- 定义别名 -->
        <property name="typeAliasesPackage" value="com.booway.pojo"></property>

        <!-- 配置映射文件的位置，如果配置文件与mapper接口在同一个位置，可以不写 -->
        <!-- <property name="mapperLocations" value="classpath:mybatis/mapper/*.xml"></property>  -->
        
         <!-- <property name="mapperLocations">
            <array>
                <value>classpath:mybatis/mapper/User.xml</value>
            </array>
        </property> -->
        
    </bean>

    <!-- 将mybatis实现的接口注入到spring容器中 -->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"></property>
        <property name="basePackage" value="com.booway.mapper"></property>
    </bean>
```
首先我们先来分析一下SqlSessionFactoryBean，来看看它类继承图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/be1fc4d1a9449d577439e257854ae181.png)


```
public class SqlSessionFactoryBean implements FactoryBean<SqlSessionFactory>, InitializingBean, ApplicationListener<ApplicationEvent> {

      //mybatis的主配置文件资源
      private Resource configLocation;
      
      //配置对象，基本上mybatis解析后的配置都存在这个配置对象中
      private Configuration configuration;
      //mapper配置资源
      private Resource[] mapperLocations;
      //数据源
      private DataSource dataSource;
      //事务工厂
      private TransactionFactory transactionFactory;
      //配置文件中设置的属性
      private Properties configurationProperties;
      //SqlSessionFactory的构造器
      private SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
      //SqlSessionFactory，用于构建SqlSession
      private SqlSessionFactory sqlSessionFactory;
    
      //EnvironmentAware requires spring 3.1
      //环境名
      private String environment = SqlSessionFactoryBean.class.getSimpleName();
      //是否快速失败
      private boolean failFast;
      //插件，也就是拦截器
      private Interceptor[] plugins;
      //类型处理器
      private TypeHandler<?>[] typeHandlers;
      //类型处理器扫描基包路径
      private String typeHandlersPackage;
      //别名类
      private Class<?>[] typeAliases;
      //别名扫描路径
      private String typeAliasesPackage;
      //别名超类
      private Class<?> typeAliasesSuperType;
    
      //issue #19. No default provider.
      //数据库id提供者
      private DatabaseIdProvider databaseIdProvider;
      //vfs工具
      private Class<? extends VFS> vfs;
      //缓存
      private Cache cache;
      //对象工厂，用于创建工厂
      private ObjectFactory objectFactory;
      //对象包装对象，用于提供访问属性，信息的便捷功能
      private ObjectWrapperFactory objectWrapperFactory;
      
      。。。。。。
}
```
可以看到SqlSessionFactoryBean实现了InitializingBean，FactoryBean，ApplicationListener接口，我们按照spring加载这个bean的顺序来分析它
首先分析SqlSessionFactoryBean的初始化方法


```
public void afterPropertiesSet() throws Exception {
    //dataSource是必须滴
    notNull(dataSource, "Property 'dataSource' is required");
    //sqlSessionFactoryBuilder也是必须滴
    notNull(sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
    //configuration对象与configLocation不能同时指定
    state((configuration == null && configLocation == null) || !(configuration != null && configLocation != null),
              "Property 'configuration' and 'configLocation' can not specified with together");
    //开始构建SqlSessionFactory
    this.sqlSessionFactory = buildSqlSessionFactory();
  }
```
> SqlSessionFactory

```
protected SqlSessionFactory buildSqlSessionFactory() throws IOException {

    Configuration configuration;

    XMLConfigBuilder xmlConfigBuilder = null;
    //如果指定了Configuration对象，那么仅仅是将当前设置配置属性设置进去
    if (this.configuration != null) {
      configuration = this.configuration;
      if (configuration.getVariables() == null) {
        configuration.setVariables(this.configurationProperties);
      } else if (this.configurationProperties != null) {
        configuration.getVariables().putAll(this.configurationProperties);
      }
      //构建解析主配置文件的builder
    } else if (this.configLocation != null) {
      xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null, this.configurationProperties);
      //获取配置对象
      configuration = xmlConfigBuilder.getConfiguration();
    } else {
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Property `configuration` or 'configLocation' not specified, using default MyBatis Configuration");
      }
      //如果啥配置都没有指定，那么直接构建对象
      configuration = new Configuration();
      configuration.setVariables(this.configurationProperties);
    }

    if (this.objectFactory != null) {
      //设置对象工厂，通过这种方式，我们可以操纵mybatis创建对象的方式
      configuration.setObjectFactory(this.objectFactory);
    }

    if (this.objectWrapperFactory != null) {
      //设置对象包装工厂，通过这种方式我们可以操纵mybatis访问对象的方式
      configuration.setObjectWrapperFactory(this.objectWrapperFactory);
    }

    if (this.vfs != null) {
      //设置vfs文件访问实现类名
      configuration.setVfsImpl(this.vfs);
    }

    if (hasLength(this.typeAliasesPackage)) {
      //解析别名包
      String[] typeAliasPackageArray = tokenizeToStringArray(this.typeAliasesPackage,
          ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
      for (String packageToScan : typeAliasPackageArray) {
        //注册别名
        configuration.getTypeAliasRegistry().registerAliases(packageToScan,
                typeAliasesSuperType == null ? Object.class : typeAliasesSuperType);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Scanned package: '" + packageToScan + "' for aliases");
        }
      }
    }

    if (!isEmpty(this.typeAliases)) {
       //注册别名
      for (Class<?> typeAlias : this.typeAliases) {
        configuration.getTypeAliasRegistry().registerAlias(typeAlias);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Registered type alias: '" + typeAlias + "'");
        }
      }
    }

    if (!isEmpty(this.plugins)) {
      for (Interceptor plugin : this.plugins) {
        //添加插件
        configuration.addInterceptor(plugin);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Registered plugin: '" + plugin + "'");
        }
      }
    }

    if (hasLength(this.typeHandlersPackage)) {
      //类型处理器包
      String[] typeHandlersPackageArray = tokenizeToStringArray(this.typeHandlersPackage,
          ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
      for (String packageToScan : typeHandlersPackageArray) {
        configuration.getTypeHandlerRegistry().register(packageToScan);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Scanned package: '" + packageToScan + "' for type handlers");
        }
      }
    }

    if (!isEmpty(this.typeHandlers)) {
       //注册类型处理器
      for (TypeHandler<?> typeHandler : this.typeHandlers) {
        configuration.getTypeHandlerRegistry().register(typeHandler);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Registered type handler: '" + typeHandler + "'");
        }
      }
    }
    //设置数据库id，数据库id就是对应数据源一个唯一名字
    if (this.databaseIdProvider != null) {//fix #64 set databaseId before parse mapper xmls
      try {
        configuration.setDatabaseId(this.databaseIdProvider.getDatabaseId(this.dataSource));
      } catch (SQLException e) {
        throw new NestedIOException("Failed getting a databaseId", e);
      }
    }

    if (this.cache != null) {
      //添加用户设置的缓存
      configuration.addCache(this.cache);
    }

    if (xmlConfigBuilder != null) {
      try {
        //解析主配置文件
        xmlConfigBuilder.parse();

        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Parsed configuration file: '" + this.configLocation + "'");
        }
      } catch (Exception ex) {
        throw new NestedIOException("Failed to parse config resource: " + this.configLocation, ex);
      } finally {
        ErrorContext.instance().reset();
      }
    }

    if (this.transactionFactory == null) {
      //一般使用默认的spring事务管理工厂
      this.transactionFactory = new SpringManagedTransactionFactory();
    }
    //构建环境，就是用来维护environment名，事务工厂，数据源
    configuration.setEnvironment(new Environment(this.environment, this.transactionFactory, this.dataSource));

    if (!isEmpty(this.mapperLocations)) {
      for (Resource mapperLocation : this.mapperLocations) {
        if (mapperLocation == null) {
          continue;
        }

        try {
          //构建解析mapper的builder
          XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(),
              configuration, mapperLocation.toString(), configuration.getSqlFragments());
          //解析mapper配置文件
          xmlMapperBuilder.parse();
        } catch (Exception e) {
          throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", e);
        } finally {
          ErrorContext.instance().reset();
        }

        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Parsed mapper file: '" + mapperLocation + "'");
        }
      }
    } else {
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Property 'mapperLocations' was not specified or no matching resources found");
      }
    }
    //构建SqlSessionFactory
    return this.sqlSessionFactoryBuilder.build(configuration);
  }
```
上面我们大致说明了几个重要的代码，现在我们将这些步骤进行拆分讲解。

## 二、XMLConfigBuilder的构建与主配置的解析

### 2.1 XMLConfigBuilder的类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/8490917dad478e8406a7bce73a3263df.png)

configuration：配置对象，配置文件解析后的信息都由它来维护
typeAliasRegistry：别名注册器，从类图中可以看到，它用一个map维护别名-》真正的类型
typeHandlerRegistry：类型处理器注册器，从类图上来看维护的map有点多

```
//维护jdbc类型与类型处理器的关系，eg:{varchar:类型处理器1，clob：类型处理器2，text：类型处理器3。。。。。。}
private final Map<JdbcType, TypeHandler<?>> JDBC_TYPE_HANDLER_MAP = new EnumMap<JdbcType, TypeHandler<?>>(JdbcType.class);
//{javaType:{jdbcType : 类型处理器}}，比如{String:{varchar:类型处理器1，clob：类型处理器2，text：类型处理器3。。。。。。}}
private final Map<Type, Map<JdbcType, TypeHandler<?>>> TYPE_HANDLER_MAP = new HashMap<Type, Map<JdbcType, TypeHandler<?>>>();
//用于Object对象的类型处理器
private final TypeHandler<Object> UNKNOWN_TYPE_HANDLER = new UnknownTypeHandler(this);
//注意：这里是类型处理器的类型-》类型处理器
private final Map<Class<?>, TypeHandler<?>> ALL_TYPE_HANDLERS_MAP = new HashMap<Class<?>, TypeHandler<?>>();
```
parsed：用于标记是否已经解析过了，用于控制只解析一遍
parser：mybatis封装的xpath解析
localReflectorFactory：反射工厂，用于获取Reflector对象

### 2.2 XMLConfigBuilder构造

```
//inputStream：主配置文件流，environment：环境名，默认为null，props：配置属性，一般就是用户在spring xml中直接配置的
public XMLConfigBuilder(InputStream inputStream, String environment, Properties props) {
    //(*1*)
    //构建XPathParser的参数说明：inputStream：主配置文件的文档流，true：表示是否需要校验文档，XMLMapperEntityResolver用于后去dtd文件的实体解析器，
    //用于校验，environment：环境名，props：外界配置的属性
    this(new XPathParser(inputStream, true, props, new XMLMapperEntityResolver()), environment, props);
  }
  
  //(*1*)
  //parse：mybatis封装的xpath解析器
private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
    //(*2*)
    super(new Configuration());
    ErrorContext.instance().resource("SQL Mapper Configuration");
    this.configuration.setVariables(props);
    this.parsed = false;
    this.environment = environment;
    this.parser = parser;
  }
  
  //(*2*)
  public BaseBuilder(Configuration configuration) {
    this.configuration = configuration;
    this.typeAliasRegistry = this.configuration.getTypeAliasRegistry();
    this.typeHandlerRegistry = this.configuration.getTypeHandlerRegistry();
  }
```
从上面的代码可以看到，XMLConfigBuilder维护的configuration是直接new出来的，这个configuration对象内部维护很多的配置信息，当然在目前那些配置信息还是空的，那么什么时候才会被填充配置信息呢？XMLConfigBuilder的parse方法就是做这种事情的。在分析parse之前先简单说明下，mybatis构建document对象用的是JDK提供的w3c标准的xml文档对象，这里就不过多说明了。

### 2.3 XPathParser基础

下面首先我们来看下XPathParser是如何解析xml的，先打好mybatis解析xml的基础，然后在学习配置文件的解析

```
//expression：xpath语法路径，比如/configuration，表示寻找根路劲下的configuration标签
//关于xpath的语法，可以到w3school中学习，地址：http://www.w3school.com.cn/xpath/index.asp
public XNode XPathParser.evalNode(String expression) {
    return evalNode(document, expression);
  }

  public XNode XPathParser.evalNode(Object root, String expression) {
    //调用javax.xml.xpath包下的Xpath对象解析文档元素, XPathConstants也是javax.xml.xpath包下的，用于指定当前
    //执行表达式后将返回的文档元素类型，很明显这里是NODE类型的
    //(*1*)
    Node node = (Node) evaluate(expression, root, XPathConstants.NODE);
    if (node == null) {
      return null;
    }
    //封装成mybatis的XNode对象
    return new XNode(this, node, variables);
  }
  
  //(*1*)
  private Object evaluate(String expression, Object root, QName returnType) {
    try {
      return xpath.evaluate(expression, root, returnType);
    } catch (Exception e) {
      throw new BuilderException("Error evaluating XPath.  Cause: " + e, e);
    }
  }
```
上面代码中对xpath解析后的Node对象进行了进一步的封装，封装对象是XNode，那么封装它走什么呢？一下是它的属性列表

```
public class XNode {
  //org.w3c.dom.Node
  private Node node;
  //标签名
  private String name;
  //标签体内容
  private String body;
  //标签属性
  private Properties attributes;
  //属性变量，用于解析标签属性上的${key},key对应的value就是从这个variables中拿
  private Properties variables;
  //XPathParser
  private XPathParser xpathParser;
  
  。。。。。。
}
```

下面是XNode的构造器

```
public XNode(XPathParser xpathParser, Node node, Properties variables) {
    this.xpathParser = xpathParser;
    this.node = node;
    this.name = node.getNodeName();
    this.variables = variables;
    //解析属性值
    this.attributes = parseAttributes(node);
    //解析标签体
    this.body = parseBody(node);
  }
```
可以看到在构建XNode对象的时候就构建了属性值和标签体，具体的解析过程，就进行过多说明了，相信大家看下就会。

XPathParser除了解析Node之后还提供了解析其他类型的方法，比如

```

public Integer evalInteger(Object root, String expression) {
    //解析成Integer值
    return Integer.valueOf(evalString(root, expression));
  }

public String evalString(Object root, String expression) {
    //解析表达式，获取的值为String，比如我们解析标签上的某个属性，就可以使用这个方法
    String result = (String) evaluate(expression, root, XPathConstants.STRING);
    //解析${}占位符
    result = PropertyParser.parse(result, variables);
    return result;
  }
  
```

### 2.4 主配置文件的解析

上面我们学习了mybatis是如何解析xml的，那么现在我们开始XMLConfigBuilder的解析

```
public Configuration XMLConfigBuilder.parse() {

    //只允许解析一遍
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    //解析根元素（document）下的configuration元素（相对于配置文件来说configuration是根元素）
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }

  private void XMLConfigBuilder.parseConfiguration(XNode root) {
    try {
      //获取主配置文件设置的settings属性
      Properties settings = settingsAsPropertiess(root.evalNode("settings"));
      //issue #117 read properties first
      //解析在主配置中配置的属性变量，将于在spring xml中配置的SQLSessionFactoryBean中的属性进行合并
      propertiesElement(root.evalNode("properties"));
      //从settings中获取Vfs的实现
      loadCustomVfs(settings);
      //解析typeAliases，比如package后者是一个个别名标签
      typeAliasesElement(root.evalNode("typeAliases"));
      //解析插件拦截器
      pluginElement(root.evalNode("plugins"));
      //解析主配置文件中配置的对象工厂
      objectFactoryElement(root.evalNode("objectFactory"));
      //解析主配置文件中配置的对象包装工厂，对象包装一般就是为了提供更加便捷的对象信息访问
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      //解析反射工厂
      reflectionFactoryElement(root.evalNode("reflectionFactory"));
      //设置settings属性到configuration对象中
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      //环境的解析，通常我们需要配置多套的数据源环境时，可以使用这种方法，不过一般我们与spring结合的话，这个标签是不设置的
      environmentsElement(root.evalNode("environments"));
      //解析数据库id提供者，用于解析databaseId，用于配置多数据源时可用
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      //解析类型处理器
      typeHandlerElement(root.evalNode("typeHandlers"));
      //解析mapper配置文件路径，如果是直接的配置文件，那么会立马解析
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```
上面的XMLConfigBuilder的parse方法将主要的一些配置解析到了configuration对象中，具体的解析过程，这里不会具体的去分析，结合前面学习的mybatis的xml解析方式，我相信谁都能懂。

## 三、别名注册

从上面的分析我们知道，注册别名的类是TypeAliasRegistry，在构建configuration对象的时候，在成员变量中就直接new出了TypeAliasRegistry对象

```
public TypeAliasRegistry() {
    registerAlias("string", String.class);

    registerAlias("byte", Byte.class);
    registerAlias("long", Long.class);
    registerAlias("short", Short.class);
    registerAlias("int", Integer.class);
    registerAlias("integer", Integer.class);
    registerAlias("double", Double.class);
    registerAlias("float", Float.class);
    registerAlias("boolean", Boolean.class);

    registerAlias("byte[]", Byte[].class);
    registerAlias("long[]", Long[].class);
    registerAlias("short[]", Short[].class);
    registerAlias("int[]", Integer[].class);
    registerAlias("integer[]", Integer[].class);
    registerAlias("double[]", Double[].class);
    registerAlias("float[]", Float[].class);
    registerAlias("boolean[]", Boolean[].class);

    registerAlias("_byte", byte.class);
    registerAlias("_long", long.class);
    registerAlias("_short", short.class);
    registerAlias("_int", int.class);
    registerAlias("_integer", int.class);
    registerAlias("_double", double.class);
    registerAlias("_float", float.class);
    registerAlias("_boolean", boolean.class);

    registerAlias("_byte[]", byte[].class);
    registerAlias("_long[]", long[].class);
    registerAlias("_short[]", short[].class);
    registerAlias("_int[]", int[].class);
    registerAlias("_integer[]", int[].class);
    registerAlias("_double[]", double[].class);
    registerAlias("_float[]", float[].class);
    registerAlias("_boolean[]", boolean[].class);

    registerAlias("date", Date.class);
    registerAlias("decimal", BigDecimal.class);
    registerAlias("bigdecimal", BigDecimal.class);
    registerAlias("biginteger", BigInteger.class);
    registerAlias("object", Object.class);

    registerAlias("date[]", Date[].class);
    registerAlias("decimal[]", BigDecimal[].class);
    registerAlias("bigdecimal[]", BigDecimal[].class);
    registerAlias("biginteger[]", BigInteger[].class);
    registerAlias("object[]", Object[].class);

    registerAlias("map", Map.class);
    registerAlias("hashmap", HashMap.class);
    registerAlias("list", List.class);
    registerAlias("arraylist", ArrayList.class);
    registerAlias("collection", Collection.class);
    registerAlias("iterator", Iterator.class);

    registerAlias("ResultSet", ResultSet.class);
  }
```
可以看到，TypeAliasRegistry的构造器中注册了许多默认的别名

下面我们直接先来看看扫描基包的方式，是如何注册别名的

```
public void org.apache.ibatis.type.TypeAliasRegistry.registerAliases(String packageName, Class<?> superType){
    //解析工具
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<Class<?>>();
    //ResolverUtil.IsA从名字上就可以知道，xx是A吗？
    //(*1*)
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    //获取所有扫描到的类
    Set<Class<? extends Class<?>>> typeSet = resolverUtil.getClasses();
    for(Class<?> type : typeSet){
      // Ignore inner classes and interfaces (including package-info.java)
      // Skip also inner classes. See issue #6
      //跳过匿名内部类，接口，和成员类
      if (!type.isAnonymousClass() && !type.isInterface() && !type.isMemberClass()) {
        registerAlias(type);
      }
    }
  }
  
  //(*1*)
  public ResolverUtil<T> find(Test test, String packageName) {
    //将点替换成正斜杠
    String path = getPackagePath(packageName);

    try {
      //获取这个包下的所有子文件
      List<String> children = VFS.getInstance().list(path);
      for (String child : children) {
        //获取class文件
        if (child.endsWith(".class")) {
          //使用前面的IsA进行匹配
          addIfMatching(test, child);
        }
      }
    } catch (IOException ioe) {
      log.error("Could not read package: " + packageName, ioe);
    }

    return this;
  }
```
注册别名

```
public void registerAlias(Class<?> type) {
    //默认的别名是对应类型的简单名称
    String alias = type.getSimpleName();
    //获取注解@Alias
    Alias aliasAnnotation = type.getAnnotation(Alias.class);
    if (aliasAnnotation != null) {
      //获取
      alias = aliasAnnotation.value();
    } 
    registerAlias(alias, type);
  }
  
  public void registerAlias(String alias, Class<?> value) {
    if (alias == null) {
      throw new TypeException("The parameter alias cannot be null");
    }
    // issue #748
    String key = alias.toLowerCase(Locale.ENGLISH);
    if (TYPE_ALIASES.containsKey(key) && TYPE_ALIASES.get(key) != null && !TYPE_ALIASES.get(key).equals(value)) {
      throw new TypeException("The alias '" + alias + "' is already mapped to the value '" + TYPE_ALIASES.get(key).getName() + "'.");
    }
    TYPE_ALIASES.put(key, value);
  }
```

## 四、类型处理器注册

类型注册其类型注册器是TypeHandlerRegistry

```
public TypeHandlerRegistry() {
    register(Boolean.class, new BooleanTypeHandler());
    register(boolean.class, new BooleanTypeHandler());
    register(JdbcType.BOOLEAN, new BooleanTypeHandler());
    register(JdbcType.BIT, new BooleanTypeHandler());

    register(Byte.class, new ByteTypeHandler());
    register(byte.class, new ByteTypeHandler());
    register(JdbcType.TINYINT, new ByteTypeHandler());

    register(Short.class, new ShortTypeHandler());
    register(short.class, new ShortTypeHandler());
    register(JdbcType.SMALLINT, new ShortTypeHandler());

    register(Integer.class, new IntegerTypeHandler());
    register(int.class, new IntegerTypeHandler());
    register(JdbcType.INTEGER, new IntegerTypeHandler());

    register(Long.class, new LongTypeHandler());
    register(long.class, new LongTypeHandler());

    register(Float.class, new FloatTypeHandler());
    register(float.class, new FloatTypeHandler());
    register(JdbcType.FLOAT, new FloatTypeHandler());

    register(Double.class, new DoubleTypeHandler());
    register(double.class, new DoubleTypeHandler());
    register(JdbcType.DOUBLE, new DoubleTypeHandler());

    register(Reader.class, new ClobReaderTypeHandler());
    register(String.class, new StringTypeHandler());
    register(String.class, JdbcType.CHAR, new StringTypeHandler());
    register(String.class, JdbcType.CLOB, new ClobTypeHandler());
    register(String.class, JdbcType.VARCHAR, new StringTypeHandler());
    register(String.class, JdbcType.LONGVARCHAR, new ClobTypeHandler());
    register(String.class, JdbcType.NVARCHAR, new NStringTypeHandler());
    register(String.class, JdbcType.NCHAR, new NStringTypeHandler());
    register(String.class, JdbcType.NCLOB, new NClobTypeHandler());
    register(JdbcType.CHAR, new StringTypeHandler());
    register(JdbcType.VARCHAR, new StringTypeHandler());
    register(JdbcType.CLOB, new ClobTypeHandler());
    register(JdbcType.LONGVARCHAR, new ClobTypeHandler());
    register(JdbcType.NVARCHAR, new NStringTypeHandler());
    register(JdbcType.NCHAR, new NStringTypeHandler());
    register(JdbcType.NCLOB, new NClobTypeHandler());

    register(Object.class, JdbcType.ARRAY, new ArrayTypeHandler());
    register(JdbcType.ARRAY, new ArrayTypeHandler());

    register(BigInteger.class, new BigIntegerTypeHandler());
    register(JdbcType.BIGINT, new LongTypeHandler());

    register(BigDecimal.class, new BigDecimalTypeHandler());
    register(JdbcType.REAL, new BigDecimalTypeHandler());
    register(JdbcType.DECIMAL, new BigDecimalTypeHandler());
    register(JdbcType.NUMERIC, new BigDecimalTypeHandler());

    register(InputStream.class, new BlobInputStreamTypeHandler());
    register(Byte[].class, new ByteObjectArrayTypeHandler());
    register(Byte[].class, JdbcType.BLOB, new BlobByteObjectArrayTypeHandler());
    register(Byte[].class, JdbcType.LONGVARBINARY, new BlobByteObjectArrayTypeHandler());
    register(byte[].class, new ByteArrayTypeHandler());
    register(byte[].class, JdbcType.BLOB, new BlobTypeHandler());
    register(byte[].class, JdbcType.LONGVARBINARY, new BlobTypeHandler());
    register(JdbcType.LONGVARBINARY, new BlobTypeHandler());
    register(JdbcType.BLOB, new BlobTypeHandler());

    register(Object.class, UNKNOWN_TYPE_HANDLER);
    register(Object.class, JdbcType.OTHER, UNKNOWN_TYPE_HANDLER);
    register(JdbcType.OTHER, UNKNOWN_TYPE_HANDLER);

    register(Date.class, new DateTypeHandler());
    register(Date.class, JdbcType.DATE, new DateOnlyTypeHandler());
    register(Date.class, JdbcType.TIME, new TimeOnlyTypeHandler());
    register(JdbcType.TIMESTAMP, new DateTypeHandler());
    register(JdbcType.DATE, new DateOnlyTypeHandler());
    register(JdbcType.TIME, new TimeOnlyTypeHandler());

    register(java.sql.Date.class, new SqlDateTypeHandler());
    register(java.sql.Time.class, new SqlTimeTypeHandler());
    register(java.sql.Timestamp.class, new SqlTimestampTypeHandler());

    // mybatis-typehandlers-jsr310
    try {
      register("java.time.Instant", "org.apache.ibatis.type.InstantTypeHandler");
      register("java.time.LocalDateTime", "org.apache.ibatis.type.LocalDateTimeTypeHandler");
      register("java.time.LocalDate", "org.apache.ibatis.type.LocalDateTypeHandler");
      register("java.time.LocalTime", "org.apache.ibatis.type.LocalTimeTypeHandler");
      register("java.time.OffsetDateTime", "org.apache.ibatis.type.OffsetDateTimeTypeHandler");
      register("java.time.OffsetTime", "org.apache.ibatis.type.OffsetTimeTypeHandler");
      register("java.time.ZonedDateTime", "org.apache.ibatis.type.ZonedDateTimeTypeHandler");
    } catch (ClassNotFoundException e) {
      // no JSR-310 handlers
    }

    // issue #273
    register(Character.class, new CharacterTypeHandler());
    register(char.class, new CharacterTypeHandler());
  }
```
对于类型处理器，都会实现一个叫TypeHandler的接口

```
public interface TypeHandler<T> {
  //设置参数
  void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;

  T getResult(ResultSet rs, String columnName) throws SQLException;

  T getResult(ResultSet rs, int columnIndex) throws SQLException;

  T getResult(CallableStatement cs, int columnIndex) throws SQLException;

}
```
扫描包名注册

```
public void org.apache.ibatis.type.TypeHandlerRegistry.register(String packageName) {
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<Class<?>>();
    //扫描packageName下的TypeHandler类型的class
    resolverUtil.find(new ResolverUtil.IsA(TypeHandler.class), packageName);
    Set<Class<? extends Class<?>>> handlerSet = resolverUtil.getClasses();
    for (Class<?> type : handlerSet) {
      //Ignore inner classes and interfaces (including package-info.java) and abstract classes
      if (!type.isAnonymousClass() && !type.isInterface() && !Modifier.isAbstract(type.getModifiers())) {
        //(*1*)
        register(type);
      }
    }
  }
  
  //(*1*)
  public void register(Class<?> typeHandlerClass) {
    boolean mappedTypeFound = false;
    //获取@MappedTypes注解
    MappedTypes mappedTypes = typeHandlerClass.getAnnotation(MappedTypes.class);
    if (mappedTypes != null) {
      for (Class<?> javaTypeClass : mappedTypes.value()) {
        //注册
        register(javaTypeClass, typeHandlerClass);
        mappedTypeFound = true;
      }
    }
    if (!mappedTypeFound) {
      //构建typeHandler对象，从typeHandler的泛型参数中获取javaType注册
      register(getInstance(null, typeHandlerClass));
    }
  }
```

```
private <T> void register(Type javaType, TypeHandler<? extends T> typeHandler) {
    //获取@MappedJdbcTypes注解
    MappedJdbcTypes mappedJdbcTypes = typeHandler.getClass().getAnnotation(MappedJdbcTypes.class);
    if (mappedJdbcTypes != null) {
      for (JdbcType handledJdbcType : mappedJdbcTypes.value()) {
        //注册
        //(*1*)
        register(javaType, handledJdbcType, typeHandler);
      }
      if (mappedJdbcTypes.includeNullJdbcType()) {
        register(javaType, null, typeHandler);
      }
    } else {
      //注册
      register(javaType, null, typeHandler);
    }
  }
  
  //(*1*)
  private void register(Type javaType, JdbcType jdbcType, TypeHandler<?> handler) {
    if (javaType != null) {
      Map<JdbcType, TypeHandler<?>> map = TYPE_HANDLER_MAP.get(javaType);
      if (map == null) {
        map = new HashMap<JdbcType, TypeHandler<?>>();
        TYPE_HANDLER_MAP.put(javaType, map);
      }
      map.put(jdbcType, handler);
    }
    ALL_TYPE_HANDLERS_MAP.put(handler.getClass(), handler);
  }
```
上面的代码虽然没有给出很多的注释，但是比较简单，大致看下就行了


## 五、XMLMapperBuilder

### 5.1 XMLMapperBuilder概览

XMLMapperBuilder是一个Mapper配置文件解析构建者

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ba06a5266844e491dc8e5cdfc6dc5f48.png)

builderAssistant：构建助手，用于构建Mapper相关的对象，比如ParameterMapping，ResultMapping等
sqlFragments：sql片段，比如
```
<sql id='fields'>
    id,name,age
</sql>
```
XMLMapperBuilder的构造函数

```
//inputStream：mapper配置文件流，configuration：配置对象，resource：mapper配置文件的路径，sqlFragments：sql片段id->XNode
public XMLMapperBuilder(InputStream inputStream, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    //XPathParser的构建不再赘述
    this(new XPathParser(inputStream, true, configuration.getVariables(), new XMLMapperEntityResolver()),
        configuration, resource, sqlFragments);
  }
```

### 5.2 mapper配置文件的解析

```
public void org.apache.ibatis.builder.xml.XMLMapperBuilder.parse() {
    //判断当前资源是否已经加载
    if (!configuration.isResourceLoaded(resource)) {
      configurationElement(parser.evalNode("/mapper"));
      //记录已加载的mapper资源
      configuration.addLoadedResource(resource);
      //构建Mapper
      bindMapperForNamespace();
    }
    //解析待解析的ResultMap
    parsePendingResultMaps();
    //解析待解析的cache引用
    parsePendingChacheRefs();
    //解析待解析的Statement
    parsePendingStatements();
  }
```

#### 5.2.1 mapper文件个元素的解析

```
private void configurationElement(XNode context) {
    try {
      //获取mapper标签上的namespace属性
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      //设置namespace到构建助手类中
      builderAssistant.setCurrentNamespace(namespace);
      //解析cache-ref标签
      cacheRefElement(context.evalNode("cache-ref"));
      //解析cache
      cacheElement(context.evalNode("cache"));
      //解析parameterMap，这种方式已经过时，不建议在使用
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      //解析resultMap
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      //解析sql片段
      sqlElement(context.evalNodes("/mapper/sql"));
      //解析select，insert，update，delete
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. Cause: " + e, e);
    }
  }
```
1. 解析cache-ref标签

```
private void cacheRefElement(XNode context) {
    if (context != null) {
      //cacheRefMap.put(namespace, referencedNamespace);
      configuration.addCacheRef(builderAssistant.getCurrentNamespace(), context.getStringAttribute("namespace"));
      /* public CacheRefResolver(MapperBuilderAssistant assistant, String cacheRefNamespace) {
       *    this.assistant = assistant;
       *    this.cacheRefNamespace = cacheRefNamespace;
       * }
       */
      CacheRefResolver cacheRefResolver = new CacheRefResolver(builderAssistant, context.getStringAttribute("namespace"));
      try {
        //解析缓存引用
        //(*1*)
        cacheRefResolver.resolveCacheRef();
      } catch (IncompleteElementException e) {
        //如果解析失败，那么存放到待解析列表，等所有解析结束之后再次解析
        configuration.addIncompleteCacheRef(cacheRefResolver);
      }
    }
  }
  
  //(*1*)
  public Cache CacheRefResolver.resolveCacheRef() {
    //(*2*)
    return assistant.useCacheRef(cacheRefNamespace);
  }
  
  //(*2*)
  public Cache org.apache.ibatis.builder.MapperBuilderAssistant.useCacheRef(String namespace) {
    if (namespace == null) {
      throw new BuilderException("cache-ref element requires a namespace attribute.");
    }
    try {
      unresolvedCacheRef = true;
      //从configuration的Map<String, Cache> caches中获取值
      Cache cache = configuration.getCache(namespace);
      if (cache == null) {
        //解析失败
        throw new IncompleteElementException("No cache for namespace '" + namespace + "' could be found.");
      }
      currentCache = cache;
      unresolvedCacheRef = false;
      return cache;
    } catch (IllegalArgumentException e) {
      throw new IncompleteElementException("No cache for namespace '" + namespace + "' could be found.", e);
    }
  }
```

2. 解析cache

```
private void org.apache.ibatis.builder.xml.XMLMapperBuilder.cacheElement(XNode context) throws Exception {
    if (context != null) {
      //获取缓存对象类型，如果没有设置，那么默认为PERPETUAL
      String type = context.getStringAttribute("type", "PERPETUAL");
      //解析别名
      Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
      //设置缓存策略：最近使用原则
      String eviction = context.getStringAttribute("eviction", "LRU");
      //获取LRU策略
      Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
      //获取刷新频率
      Long flushInterval = context.getLongAttribute("flushInterval");
      //获取缓存大小
      Integer size = context.getIntAttribute("size");
      //是否只读
      boolean readWrite = !context.getBooleanAttribute("readOnly", false);
      //是否阻塞
      boolean blocking = context.getBooleanAttribute("blocking", false);
      //获取缓存配置属性
      Properties props = context.getChildrenAsProperties();
      //设置缓存
      //(*1*)
      builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
    }
  }
  
  //(*1*)
  public Cache org.apache.ibatis.builder.MapperBuilderAssistant.useNewCache(Class<? extends Cache> typeClass,
      Class<? extends Cache> evictionClass,
      Long flushInterval,
      Integer size,
      boolean readWrite,
      boolean blocking,
      Properties props) {
    //CacheCacheBuilder（缓存建造器）
    Cache cache = new CacheBuilder(currentNamespace)
        //valueOrDefault在第一个参数值为null的时候会使用第二个参数，PerpetualCache为默认的缓存实现
        .implementation(valueOrDefault(typeClass, PerpetualCache.class))
        //设置装饰器，默认LruCache
        .addDecorator(valueOrDefault(evictionClass, LruCache.class))
        .clearInterval(flushInterval)
        .size(size)
        .readWrite(readWrite)
        .blocking(blocking)
        .properties(props)
        .build();
    //namespace -》 cache对象
    configuration.addCache(cache);
    //org.apache.ibatis.builder.MapperBuilderAssistant.currentCache = cache
    currentCache = cache;
    return cache;
  }
  
  //缓存对象的构建
  public Cache org.apache.ibatis.mapping.CacheBuilder.build() {
    //确保有Cache的实现类，如果没有定义，那么设置的将是PerpetualCache
    setDefaultImplementations();
    //寻找缓存的构造器，其构造参数为String，也就是用于传递namespace的参数
    Cache cache = newBaseCacheInstance(implementation, id);
    //设置属性，这个属性就是在cache标签下面设置的property标签
    setCacheProperties(cache);
    // issue #352, do not apply decorators to custom caches
    //如果是mybatis的缓存就进行装饰，自定义的缓存只会被LoggingCache装饰
    if (PerpetualCache.class.equals(cache.getClass())) {
      for (Class<? extends Cache> decorator : decorators) {
        cache = newCacheDecoratorInstance(decorator, cache);
        //设置属性
        setCacheProperties(cache);
      }
      //使用标准的装饰器进行装饰，比如根据是否设置了clearInterval，用ScheduledCache进行装饰
      cache = setStandardDecorators(cache);
    } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
      cache = new LoggingCache(cache);
    }
    return cache;
  }
```
3. 解析parameterMap（已过时，这类信息都由#{}表达式代替）

```
//  /mapper/parameterMap
private void org.apache.ibatis.builder.xml.XMLMapperBuilder.parameterMapElement(List<XNode> list) throws Exception {
    for (XNode parameterMapNode : list) {
      //解析parameterMap的id属性
      String id = parameterMapNode.getStringAttribute("id");
      //解析参数类型
      String type = parameterMapNode.getStringAttribute("type");
      //解析参数class对象
      Class<?> parameterClass = resolveClass(type);
      //获取parameter标签
      List<XNode> parameterNodes = parameterMapNode.evalNodes("parameter");
      List<ParameterMapping> parameterMappings = new ArrayList<ParameterMapping>();
      for (XNode parameterNode : parameterNodes) {
        //属性名
        String property = parameterNode.getStringAttribute("property");
        //java类型
        String javaType = parameterNode.getStringAttribute("javaType");
        //jdbc类型
        String jdbcType = parameterNode.getStringAttribute("jdbcType");
        //resultMap，sql返回值映射，通常配置了parameterMap属性的select标签什么的，不需要再配置
        //resultMap,resultType,parameterType
        String resultMap = parameterNode.getStringAttribute("resultMap");
        //模式，比如IN，OUT，INOUT
        String mode = parameterNode.getStringAttribute("mode");
        //类型处理器
        String typeHandler = parameterNode.getStringAttribute("typeHandler");
        //小数位
        Integer numericScale = parameterNode.getIntAttribute("numericScale");
        //解析参数模式ParameterMode.IN, ParameterMode.OUT，ParameterMode.INOUT
        ParameterMode modeEnum = resolveParameterMode(mode);
        //解析java类型
        Class<?> javaTypeClass = resolveClass(javaType);
        //jdbc类型
        JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
        @SuppressWarnings("unchecked")
        //解析类型处理器
        Class<? extends TypeHandler<?>> typeHandlerClass = (Class<? extends TypeHandler<?>>) resolveClass(typeHandler);
        //构建ParameterMapping
        //（*1*）
        ParameterMapping parameterMapping = builderAssistant.buildParameterMapping(parameterClass, property, javaTypeClass, jdbcTypeEnum, resultMap, modeEnum, typeHandlerClass, numericScale);
        parameterMappings.add(parameterMapping);
      }
      builderAssistant.addParameterMap(id, parameterClass, parameterMappings);
    }
  }
  
  //（*1*）
   public ParameterMapping buildParameterMapping(
      Class<?> parameterType,
      String property,
      Class<?> javaType,
      JdbcType jdbcType,
      String resultMap,
      ParameterMode parameterMode,
      Class<? extends TypeHandler<?>> typeHandler,
      Integer numericScale) {
    //拼接namespace =》 namespace + "." + resultMap
    resultMap = applyCurrentNamespace(resultMap, true);

    // Class parameterType = parameterMapBuilder.type();
    //如果javaType类型为空，那么进行解析，通常会使用MetaClass进行反射
    Class<?> javaTypeClass = resolveParameterJavaType(parameterType, property, javaType, jdbcType);
    //如果没有提供typeHandler，那么从typeHandler注册中获取
    TypeHandler<?> typeHandlerInstance = resolveTypeHandler(javaTypeClass, typeHandler);
    //构建参数map
    return new ParameterMapping.Builder(configuration, property, javaTypeClass)
        .jdbcType(jdbcType)
        .resultMapId(resultMap)
        .mode(parameterMode)
        .numericScale(numericScale)
        .typeHandler(typeHandlerInstance)
        //构建，在返回之前，如果typeHandler依然为空，会继续从typeHandler注册器中获取
        //校验，eg：如果参数类型为ResultSet，那么resultMap必须存在
        .build();
  }
```
4. 解析resultMap

```
private void org.apache.ibatis.builder.xml.XMLMapperBuilder.resultMapElements(List<XNode> list) throws Exception {
    for (XNode resultMapNode : list) {
      try {
        resultMapElement(resultMapNode);
      } catch (IncompleteElementException e) {
        // ignore, it will be retried
      }
    }
  }
  
  private ResultMap resultMapElement(XNode resultMapNode) throws Exception {
    return resultMapElement(resultMapNode, Collections.<ResultMapping> emptyList());
  }
  //resultMap节点解析
  private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings) throws Exception {
    ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());
    //获取resultMap节点属性id，如果没有那么使用默认的 resultMapNode.getValueBasedIdentifier()
    String id = resultMapNode.getStringAttribute("id",
        resultMapNode.getValueBasedIdentifier());
    //获取返回值类型
    String type = resultMapNode.getStringAttribute("type",
        resultMapNode.getStringAttribute("ofType",
            resultMapNode.getStringAttribute("resultType",
                resultMapNode.getStringAttribute("javaType"))));
    //获取继承的ResultMap配置id
    String extend = resultMapNode.getStringAttribute("extends");
    //是否自动映射，自动映射将采用反射的方式进行映射
    Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
    //解析成javaType
    Class<?> typeClass = resolveClass(type);
    //鉴别器
    Discriminator discriminator = null;
    List<ResultMapping> resultMappings = new ArrayList<ResultMapping>();
    //添加偏好设置
    resultMappings.addAll(additionalResultMappings);
    List<XNode> resultChildren = resultMapNode.getChildren();
    for (XNode resultChild : resultChildren) {
      //解析构造器标签
      if ("constructor".equals(resultChild.getName())) {
        processConstructorElement(resultChild, typeClass, resultMappings);
        //解析鉴别器
      } else if ("discriminator".equals(resultChild.getName())) {
        discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
      } else {
        List<ResultFlag> flags = new ArrayList<ResultFlag>();
        //标记，标记当前属性代表主键
        if ("id".equals(resultChild.getName())) {
          flags.add(ResultFlag.ID);
        }
        //构建resultMap
        resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
      }
    }
    ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend, discriminator, resultMappings, autoMapping);
    try {
        //解析
      return resultMapResolver.resolve();
    } catch (IncompleteElementException  e) {
      configuration.addIncompleteResultMap(resultMapResolver);
      throw e;
    }
  }
```













