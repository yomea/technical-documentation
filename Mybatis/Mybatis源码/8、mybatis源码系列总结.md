## 一、类介绍

- SqlSessionFactoryBean：实现了spring的FactoryBean接口，一个工厂类，用户构建SqlSessionFactory
- Configuration：配置对象，xml解析后的描述由被它维护
- XMLMapperBuilder：用于构建基本的ParamterMap，ResultMap的建造器
- XMLStatementBuilder：用于构建sql语句对象的建造器
- ParamterMap：参数映射复杂类型
- ParamterMapping：复杂类型中每个字段的映射
- ResultMap：复杂结果集映射
- ResultMapping：复杂类型的属性映射
- TypeHandler：类型处理器，将java类型通过对应类型jdbc设值方法，比如整型类型设置到sql中，java.sql.PreparedStatement.setInt(int, int)，反过来将数据库中的int类型变成Integer类型java对象java.sql.ResultSet.getInt(int)
- Discriminator：鉴别器，用于根据条件动态设置对应的属性对象
- SqlCommandType：表示sql语句的类型，比如insert，update，select，delete 等等
- SqlSource：它有多个实现类，比如DynamicSqlSource（动态sql，包含\${}或者动态标签的），StaticSqlSource（静态sql，已经解析过的，比如#{}，\${}都被解析过的）
- MappedStatement：语句，我们配置的select，update，delete标签会被包装成MappedStatement
- SqlNode：表示xml中关于sql的元素的java描述，比如TextSql（指文本，sql语句）,IfSqlNode（if标签）,ForEachSqlNode（for标签）

## 二、大致流程

### 2.1 配置准备阶段

mybatis通过xpath进行解析xml，将xml描述的内容转化成java定义，全部维护在Configuration对象中，解析映射配置文件的ResultMap时，对应的描述java类为ResultMap
ResultMap标签中的元素被映射为java类型ResultMapping，ResultMapping分为id属性描述，构造器参数描述，普通属性描述，以下是ResultMapping的属性结构

```
public class ResultMapping {
  //配置对象
  private Configuration configuration;
  //属性名
  private String property;
  //列名
  private String column;
  //java类型
  private Class<?> javaType;
  //jdbc类型
  private JdbcType jdbcType;
  //类型处理器
  private TypeHandler<?> typeHandler;
  //嵌入的ResultMapId，比如某些复杂对象，需要用其他的ResultMap进行映射，通常会使用association
  //collection进行映射
  private String nestedResultMapId;
  //嵌入的查询id，有点时候可能对于一些复杂对象，可能没有直接使用ResultMap，可以直接指定查询的id
  private String nestedQueryId;
  //{id,name,age}
  private Set<String> notNullColumns;
  //列名前缀
  private String columnPrefix;
  //标识，ResultFlag.ID，ResultFlag.CONSTRUCTOR
  private List<ResultFlag> flags;
  //复合列名设置的ResultMapping，比如{id=user_id, name=user_name}
  private List<ResultMapping> composites;
  //结果集，用于多结果集的情况
  private String resultSet;
  //外键
  private String foreignColumn;
  //对这个值是否懒加载
  private boolean lazy;

  ResultMapping() {
  }
  。。。。。。
 }

```
ResultMap的结构如下

```
public class ResultMap {
  //resultMap标签上的id属性
  private String id;
  //resultMap映射的复杂java对象类型
  private Class<?> type;
  //当前resultMap标签下定义的所有resultMapping
  private List<ResultMapping> resultMappings;
  //当前resultMap标签下定义的id resultMapping
  //当没有定义id resultMapping，那么这个结合存储的内容和resultMappings集合是一样的
  private List<ResultMapping> idResultMappings;
  //当前resultMap标签下定义的构造参数 resultMapping
  private List<ResultMapping> constructorResultMappings;
  //当前resultMap标签下定义的属性resultMapping
  private List<ResultMapping> propertyResultMappings;
  //当前resultMap标签下定义的所有列名，包括每个resutMapping中的复合resultMapping的字段名
  private Set<String> mappedColumns;
  //鉴别器，维护着一个Map<case条件, ResultMap>
  private Discriminator discriminator;
  //是否存在嵌入的resultMap
  private boolean hasNestedResultMaps;
  //是否存在嵌入的查询，某属性是通过另外一个select获取的属性
  private boolean hasNestedQueries;
  //是否进行自动映射，没有进行映射配置的字段
  private Boolean autoMapping;

  private ResultMap() {
  }
  
  。。。。。。
}
```
解析完了ResultMap后开始解析sql语句，sql最后会包装成MappedStatement

```
public final class MappedStatement {
  //mapper配置文件的路径，牢记自己是从什么进化而来
  private String resource;
  //配置对象
  private Configuration configuration;
  //MappedStatement语句的id
  private String id;
  //每次最大获取量
  private Integer fetchSize;
  //获取超时时间
  private Integer timeout;
  //语句类型，STATEMENT, PREPARED, CALLABLE
  private StatementType statementType;
  //结果集类型，比如滚动敏感等
  private ResultSetType resultSetType;
  //解析后SqlNode封装对象
  private SqlSource sqlSource;
  //二级缓存对象
  private Cache cache;
  //参数映射
  private ParameterMap parameterMap;
  //返回值映射
  private List<ResultMap> resultMaps;
  //是否需要刷新
  private boolean flushCacheRequired;
  //是否使用缓存
  private boolean useCache;
  //结果集顺序
  private boolean resultOrdered;
  //sql命令类型
  private SqlCommandType sqlCommandType;
  //主键生成器
  private KeyGenerator keyGenerator;
  //主键属性名
  private String[] keyProperties;
  //主键列名
  private String[] keyColumns;
  //嵌入的ResultMap
  private boolean hasNestedResultMaps;
  //数据库id
  private String databaseId;
  //mybatis的日志对象
  private Log statementLog;
  //用于解析sql的脚本解析器
  private LanguageDriver lang;
  //结果集，用于多结果集
  private String[] resultSets;

  MappedStatement() {
    // constructor disabled
  }
  。。。。。。
}
```
在添加接口映射时，还会对接口进行注解的解析，比如@Insert，@Update，@Delete，@Select等，如果这些注解上的sql语句是以<script>开头的，那么以xml的方式进行解析，主要是为了支持动态标签

MapperScannerConfigurer是一个实现了BeanDefinitionRegistryPostProcessor接口的类，在spring构建完BeanFactory之后会进行后置调用，这是spring提供给用户最后添加BeanDefinition的机会，mybatis继承了spring的ClassPathBeanDefinitionScanner，实现了自己的ClassPathMapperScanner扫描器，扫描指定路径下的接口，将这些接口封装成MapperFactoryBean，MapperFactoryBean是一个实现了FactoryBean接口的类。

### 2.2 执行sql流程

MapperProxyFactory创建接口代理对象，逻辑实现为MapperProxy，它实现了JDK的InvocationHandler接口，它在invoke方法中调用了MapperMethod的execute方法，在MapperMethod的构造器中解析方法参数，多个参数的会封装成ParamMap，除了mybatis的参数RowBounds，ResultHandler之外，如果只有一个参数的话，那么不会包装成ParamMap，但如果是集合类型（包括数组），它又会在在接下来的sqlSession.wrapCollection方法中包装成StrictMap

```
//如果参数是集合类型
if (object instanceof Collection) {
  StrictMap<Object> map = new StrictMap<Object>();
  //然后就变成了一个map
  map.put("collection", object);
  if (object instanceof List) {
    map.put("list", object);
  }
  return map;
} else if (object != null && object.getClass().isArray()) {
  StrictMap<Object> map = new StrictMap<Object>();
  //如果是数组
  map.put("array", object);
  return map;
}
return object;

```
接着创建对应的StatementHandler，如果有必要的话会被插件代理，解析sqlSource，解析\${}，替换需要的值，解析#{}，构建成表达式级别的ParamMapping，通过ParamHandler处理参数，获取TypeHandler设置对应的参数，执行sql获取结果，如果是插入语句，并且需要获取主键，那么会填充主键，最后根据ResultMap映射参数，如果映射不完整，会触发自动映射。
在mybatis3.4中可以解析一下几种类型的#{}表达式
- #{age} 译为{“property”:“age”}

- #{age,typeHandler=int,javaType=java.lang.Integer} 译为{“property”:“age”, “typeHandler”:“int”,“javaType”:“java.lang.Integer”}

- #{age:int} 译为{“property”:“age”, “jdbcType”:“int”}

- #{age:int,typeHandler=int,javaType=java.lang.Integer} 译为{“property”:“age”, “jdbcType”:“int”, “typeHandler”:“int”,“javaType”:“java.lang.Integer”}

- #{(表达式)} 译为{“expression”:“表达式”}

- #{(表达式),typeHandler=int,javaType=java.lang.Integer} 译为{“expression”:“表达式”, “typeHandler”:“int”,“javaType”:“java.lang.Integer”}

- #{(表达式):varchar} 译为{“expression”:“表达式”, “jdbcType”:“int”}

- #{(表达式):varchar,typeHandler=int,javaType=java.lang.Integer} 译为{“expression”:“表达式”, “jdbcType”:“int”, “typeHandler”:“int”,“javaType”:“java.lang.Integer”}

解析后的BoundSql

```
public class BoundSql {

  //解析#{}后的sql语句
  private String sql;
  //#{}解析出来的参数映射
  private List<ParameterMapping> parameterMappings;
  //参数
  private Object parameterObject;
  //偏好设置
  private Map<String, Object> additionalParameters;
  //偏好参数封装的MetaObject
  private MetaObject metaParameters;
  
  。。。。。。
}

```
懒加载的逻辑在ResultLoader中，原理很简单就是创建一个Executor，然后从事物工厂中获取事务，然后调用query即可

对于嵌入式的ResultMap的解析，主要是要知道cachekey的生成逻辑，mybatis生成cacheKey分以下三种：

- 有主键的，以主键字段名和字段值构建cacheKey
- 无主键的并且属性为Map的，那么使用所有字段名和字段值构建cacheKey
- 无主键的并且属性不是Map的，那么使用没有映射的字段名和字段值构建cacheKey

对于嵌入属性的嵌入属性，那么把父cacheKey加入到cacheKey中，保证唯一性

## 二、设计模式

- 建造者模式：比如构建MappedStatement的Builder
- 装饰器模式：比如Executor会被CacheExecutor装饰，再比如二级缓存会被一些LRU策略装饰
- 动态代理模式：插件，Mapper
- 责任链模式：多个插件形成了拦截器链
- 策略者模式：不同的SqlNode实现
- 组合模式：MixSqlNode与各个SqlNode组成的一颗树
- 适配器模式：mybatis的日志
- 工厂模式：MapperProxyFactory，用于创建mapper代理，ObjectWrapperFactory，用于包装对象，以达到便捷访问的目的


插件执行责任链示意图：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/42f85a44ee821ab025b6e71efbbec3d4.png)
        