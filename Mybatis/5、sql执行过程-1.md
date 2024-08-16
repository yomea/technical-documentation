
## 一、前情回顾

上一节讲到，mybatis使用JDK的动态代理为每个接口生成了代理类，其实际处理类为MapperProxy

```
public Object org.apache.ibatis.binding.MapperProxy.invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (Object.class.equals(method.getDeclaringClass())) {
      try {
        return method.invoke(this, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```

## 二、MapperMethod

MapperMethod类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/258aa9e989e72ade38708787f3e1595d.png)

### 2.1 MapperMethod执行前的准备工作

现在我们来看到MapperMethod的创建

```
public class MapperMethod {

  //维护当前将要执行的sql的命令类型和namespace + id
  private final SqlCommand command;
  private final MethodSignature method;

  public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    //创建SqlCommand
    this.command = new SqlCommand(config, mapperInterface, method);
    this.method = new MethodSignature(config, mapperInterface, method);
  }
  
  。。。。。。

}
```
MethodSignature字段声明

```
public static class MethodSignature {
    //是否返回多个值
    private final boolean returnsMany;
    //是否返回map
    private final boolean returnsMap;
    //是否返回void
    private final boolean returnsVoid;
    //是否返回游标
    private final boolean returnsCursor;
    //返回类型
    private final Class<?> returnType;
    //@MapKey("key")里的key值
    private final String mapKey;
    //resultHandler参数下标
    private final Integer resultHandlerIndex;
    //rowBounds参数下标
    private final Integer rowBoundsIndex;
    //参数下标 -》 参数名
    private final SortedMap<Integer, String> params;
    //是否存在命名参数，也就是接口参数被@Param标注
    private final boolean hasNamedParameters;
    
    。。。。。。
    
}
    
```
SqlCommand对象的创建

```
 public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
      //接口名加方法名
      String statementName = mapperInterface.getName() + "." + method.getName();
      MappedStatement ms = null;
      //判断是否存在更具接口名加方法名的MappedStatement存在
      if (configuration.hasStatement(statementName)) {
        //获取MappedStatement
        ms = configuration.getMappedStatement(statementName);
        //如果用户使用的namespace不是接口名
      } else if (!mapperInterface.equals(method.getDeclaringClass())) { // issue #35
        //尝试通过接口名加上方法名获取MappedStatement
        String parentStatementName = method.getDeclaringClass().getName() + "." + method.getName();
        if (configuration.hasStatement(parentStatementName)) {
          ms = configuration.getMappedStatement(parentStatementName);
        }
      }
      if (ms == null) {
        //如果没有找到MappedStatement，并且方法上存在@Flush注解，那么sql命令类型变成只是刷新
        if(method.getAnnotation(Flush.class) != null){
          name = null;
          type = SqlCommandType.FLUSH;
        } else {
          throw new BindingException("Invalid bound statement (not found): " + statementName);
        }
      } else {
        name = ms.getId();
        type = ms.getSqlCommandType();
        if (type == SqlCommandType.UNKNOWN) {
          throw new BindingException("Unknown execution method for: " + name);
        }
      }
    }
```
MethodSignature的创建

```
public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
      //解析返回类型
      Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
      //如果是class类型的，直接设置为返回值
      if (resolvedReturnType instanceof Class<?>) {
        this.returnType = (Class<?>) resolvedReturnType;
        //如果是参数化类型，那么需要获取其rawType类型，比如List<String>，rawType就是List
      } else if (resolvedReturnType instanceof ParameterizedType) {
        this.returnType = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
      } else {
        this.returnType = method.getReturnType();
      }
      //是否为void返回值
      this.returnsVoid = void.class.equals(this.returnType);
      //是否返回集合类型，或者数组
      this.returnsMany = (configuration.getObjectFactory().isCollection(this.returnType) || this.returnType.isArray());
      //是否返回游标
      this.returnsCursor = Cursor.class.equals(this.returnType);
      //返回值是否为map，并且这个方法方法上存在@MapKey注解，那么会返回@MapKey的value()值
      this.mapKey = getMapKey(method);
      //是否返回map
      this.returnsMap = (this.mapKey != null);
      //是否存在命名参数，判断参数注解中是否存在@Param即可
      this.hasNamedParameters = hasNamedParams(method);
      //寻找是否存在RowBounds参数，如果存在且只能有一个的话，将返回其所在参数的位置下标
      this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
      //寻找是否存在ResultHandler参数，如果存在且只能有一个的话，将返回其所在参数的位置下标
      this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
      //解析参数下标 -》 参数名
      this.params = Collections.unmodifiableSortedMap(getParams(method, this.hasNamedParameters));
    }
```
解析参数名


```
 private SortedMap<Integer, String> org.apache.ibatis.binding.MapperMethod.MethodSignature.getParams(Method method, boolean hasNamedParameters) {
      final SortedMap<Integer, String> params = new TreeMap<Integer, String>();
      //反射获取所有参数类型
      final Class<?>[] argTypes = method.getParameterTypes();
      for (int i = 0; i < argTypes.length; i++) {
        //排除mybatis提供的特殊参数RowBounds（用于假分页，也就是去除10条记录，然后要求取5条，就在内存中将他们截取）和ResultHandler（返回结果处理器）
        if (!RowBounds.class.isAssignableFrom(argTypes[i]) && !ResultHandler.class.isAssignableFrom(argTypes[i])) {
          //参数下标名（不完全是，如果中间出现RowBounds和ResultHandler，那么这个不能认为是下标名）
          String paramName = String.valueOf(params.size());
          if (hasNamedParameters) {
            //如果存在@Param，那么使用@Param指定的名字覆盖下标名
            paramName = getParamNameFromAnnotation(method, i, paramName);
          }
          params.put(i, paramName);
        }
      }
      return params;
    }
```

### 2.2 MapperMethod的执行

```
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    //sql命令类型为insert
    if (SqlCommandType.INSERT == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.insert(command.getName(), param));
    } else if (SqlCommandType.UPDATE == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
    } else if (SqlCommandType.DELETE == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
    } else if (SqlCommandType.SELECT == command.getType()) {
      if (method.returnsVoid() && method.hasResultHandler()) {
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        result = executeForMap(sqlSession, args);
      } else if (method.returnsCursor()) {
        result = executeForCursor(sqlSession, args);
      } else {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = sqlSession.selectOne(command.getName(), param);
      }
    } else if (SqlCommandType.FLUSH == command.getType()) {
        result = sqlSession.flushStatements();
    } else {
      throw new BindingException("Unknown execution method for: " + command.getName());
    }
    //返回值为null，但是返回类型确实原型类型，并且不是void，那么抛错
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```
参数转换，我们传递的参数可能是多个参数，也可能只是一个参数

```
public Object org.apache.ibatis.binding.MapperMethod.MethodSignature.convertArgsToSqlCommandParam(Object[] args) {
        //除rowBounds和resultHandler外的参数个数
      final int paramCount = params.size();
      if (args == null || paramCount == 0) {
        return null;
        //如果只有一个参数并且没有被@Param注解标注，那么直接返回这个参数即可
      } else if (!hasNamedParameters && paramCount == 1) {
        return args[params.keySet().iterator().next().intValue()];
      } else {
        //其他情况，转换成paramMap，它继承了HashMap
        final Map<String, Object> param = new ParamMap<Object>();
        int i = 0;
        for (Map.Entry<Integer, String> entry : params.entrySet()) {
          //参数名 -》 参数值
          param.put(entry.getValue(), args[entry.getKey().intValue()]);
          // issue #71, add param names as param1, param2...but ensure backward compatibility
          //填充泛化的参数名，从1开始
          final String genericParamName = "param" + String.valueOf(i + 1);
          if (!param.containsKey(genericParamName)) {
            param.put(genericParamName, args[entry.getKey()]);
          }
          i++;
        }
        return param;
      }
    }
```

## 三、修改语句的执行

这里的修改语句包括insert，update，delete，这里放在一起分析，因为它们调用的方法是一样的

这里先说明一下，sqlSession的insert，update，delete方法的返回值都是int，返回被修改的记录数，mybatis会根据我们接口方法的返回类型对这个返回的参数做一些处理

```
private Object org.apache.ibatis.binding.MapperMethod.rowCountResult(int rowCount) {
    final Object result;
    //如果返回为void，那么直接返回空值
    if (method.returnsVoid()) {
      result = null;
      //Integer或者int类型
    } else if (Integer.class.equals(method.getReturnType()) || Integer.TYPE.equals(method.getReturnType())) {
      result = Integer.valueOf(rowCount);
      //Long类型或者long类型
    } else if (Long.class.equals(method.getReturnType()) || Long.TYPE.equals(method.getReturnType())) {
      result = Long.valueOf(rowCount);
      //Boolean类型或者boolean类型
    } else if (Boolean.class.equals(method.getReturnType()) || Boolean.TYPE.equals(method.getReturnType())) {
      result = Boolean.valueOf(rowCount > 0);
    } else {
      throw new BindingException("Mapper method '" + command.getName() + "' has an unsupported return type: " + method.getReturnType());
    }
    return result;
  }
```
sqlSession的insert，delete方法调用的sqlSession的update方法，然后update方法最终调用Executor的update方法

```
public int org.apache.ibatis.session.defaults.DefaultSqlSession.update(String statement, Object parameter) {
    try {
      //标记数据为脏数据
      dirty = true;
      //根据statementId获取MappedStatement
      MappedStatement ms = configuration.getMappedStatement(statement);
      
      //这里有个wrapCollection方法，需要注意，我们在MapperMethod方法中对于一个方法除去rowBounds和resultHandler类型参数外，只要多余一个参数的
      //就会变成paramMap，0个的为null，1个的为参数自身类型。
      //(*1*)
      return executor.update(ms, wrapCollection(parameter));
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
  
  //(*1*)
  private Object org.apache.ibatis.session.defaults.DefaultSqlSession.wrapCollection(final Object object) {
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
  }
  
```

在前一节对sqlSessionTemplate的分析中，我们可以了解到，只要我们没有主动配置Executor的类型，那么默认就是SIMPLE的，也就是SimpleExecutor，然后被装饰为CacheExecutor（添加缓存功能）

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/9ed63c347b62451224b569a8d0d4c28d.png)

```
public int org.apache.ibatis.executor.CachingExecutor.update(MappedStatement ms, Object parameterObject) throws SQLException {
    //刷新缓存
    //(*1*)
    flushCacheIfRequired(ms);
    return delegate.update(ms, parameterObject);
  }
  
  //(*1*)
  private void flushCacheIfRequired(MappedStatement ms) {
    //获取该命名空间下的缓存
    Cache cache = ms.getCache();
    if (cache != null && ms.isFlushCacheRequired()) {     
      //清理缓存
      tcm.clear(cache);
    }
  }
```

BaseExecutor.update

```
public int org.apache.ibatis.executor.BaseExecutor.update(MappedStatement ms, Object parameter) throws SQLException {
    //构建ErrorContext上下文，记录着执行过程的一些记录，比如来源，动作，id
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    //如果当前执行器状态被关闭，不允许再次使用，执行器和SqlSession是属于同一生命周期的
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    //(*1*)
    clearLocalCache();
    return doUpdate(ms, parameter);
  }
  
  //(*1*)
  public void org.apache.ibatis.executor.BaseExecutor.clearLocalCache() {
    if (!closed) {
      //清除sqlSession级别的缓存，因为Executor与sqlSession同寿，一般用sqlSession级别去定义这个缓存生命周期
      localCache.clear();
      //清除输出参数缓存，用于参数模式为OUT，INOUT这种，存储过程
      localOutputParameterCache.clear();
    }
  }
```

SimpleExecutor.doUpdate

```
public int org.apache.ibatis.executor.SimpleExecutor.doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Statement stmt = null;
    try {
      //获取配置对象
      Configuration configuration = ms.getConfiguration();
      //构建语句处理器
      StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
      //ms.getStatementLog() mybatis的日志对象，debug开启的时候代理connection
      //处理参数
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.update(stmt);
    } finally {
      closeStatement(stmt);
    }
  }
```
构建语句处理器

```
public StatementHandler Configuration.newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    //路由语句处理器
    //(*1*)
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    //插件处理，插件的处理，我们后面再分析，暂时先略过
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
  }
  
  //(*1*)
  public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    //根据sql语句的类型创建不同的语句处理器，一般我们使用的是预编译语句类型，有？的那种。
    switch (ms.getStatementType()) {
      case STATEMENT:
        delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case PREPARED:
        delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case CALLABLE:
        delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      default:
        throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }
    
```

StatementHandler类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/e92219f28515e5ef6b9267904f59b522.png)

PreparedStatementHandler的构建

```
public PreparedStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    super(executor, mappedStatement, parameter, rowBounds, resultHandler, boundSql);
  }
  
protected BaseStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {

    this.configuration = mappedStatement.getConfiguration();
    this.executor = executor;
    this.mappedStatement = mappedStatement;
    this.rowBounds = rowBounds;
    
    this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    this.objectFactory = configuration.getObjectFactory();

    if (boundSql == null) { // issue #435, get the key before calculating the statement
      //处理标记为执行语句前的主键生成
      generateKeys(parameterObject);
      //处理SqlSource
      boundSql = mappedStatement.getBoundSql(parameterObject);
    }

    this.boundSql = boundSql;
    //构建参数处理器
    //(*1*)
    this.parameterHandler = configuration.newParameterHandler(mappedStatement, parameterObject, boundSql);
    //构建返回值集处理器
    //(*2*)
    this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, rowBounds, parameterHandler, resultHandler, boundSql);
  }
  
  //(*1*)
  public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    //我们使用的是XMLLanguageDriver脚本处理驱动，它创建的是DefaultParameterHandler
    ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
    //插件处理，其实就是动态代理
    parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
    return parameterHandler;
  }

 //(*2*)
 public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler,
      ResultHandler resultHandler, BoundSql boundSql) {
    //创建默认的结果集处理器
    ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
    //插件处理，其实就是动态代理
    resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
    return resultSetHandler;
  }

```
从预编译语句处理的构造器中可以看到，在获取BoundSql前会进行before的主键生成

```
protected void org.apache.ibatis.executor.statement.BaseStatementHandler.generateKeys(Object parameter) {
    //获取主键生成器
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    ErrorContext.instance().store();
    //生成主键
    keyGenerator.processBefore(executor, mappedStatement, null, parameter);
    ErrorContext.instance().recall();
  }
```
通常在语句之前进行生成主键的操作都是想oracle这样的数据库，也就是先从序列器中获取递增的主键，然后插入到数据库中，也就是说通常使用的主键生成器是SelectKeyGenerator，也就是通过sql语句来获取

```
public void SelectKeyGenerator.processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    if (executeBefore) {
      processGeneratedKeys(executor, ms, parameter);
    }
  }
  
private void processGeneratedKeys(Executor executor, MappedStatement ms, Object parameter) {
    try {
        //参数不为空，生成主键的语句不为空，生成主键的属性名不为空
      if (parameter != null && keyStatement != null && keyStatement.getKeyProperties() != null) {
        //获取主键属性名
        String[] keyProperties = keyStatement.getKeyProperties();
        //获取配置对象
        final Configuration configuration = ms.getConfiguration();
        //将参数进行包装
        final MetaObject metaParam = configuration.newMetaObject(parameter);
        if (keyProperties != null) {
          // Do not close keyExecutor.
          // The transaction will be closed by parent executor.
          //构建一个新的Executor，专属于主键sql执行的Executor
          Executor keyExecutor = configuration.newExecutor(executor.getTransaction(), ExecutorType.SIMPLE);
          //调用Executor指定查询方法，这个查询方法将放在后面进行分析
          List<Object> values = keyExecutor.query(keyStatement, parameter, RowBounds.DEFAULT, Executor.NO_RESULT_HANDLER);
          if (values.size() == 0) {
            throw new ExecutorException("SelectKey returned no data.");            
          } else if (values.size() > 1) {
            throw new ExecutorException("SelectKey returned more than one value.");
          } else {
            //获取结果并包装成MetaObject
            MetaObject metaResult = configuration.newMetaObject(values.get(0));
             //只有一个主键的情况
            if (keyProperties.length == 1) {
              if (metaResult.hasGetter(keyProperties[0])) {
                //通过反射的方式设置值
                setValue(metaParam, keyProperties[0], metaResult.getValue(keyProperties[0]));
              } else {
                // no getter for the property - maybe just a single value object
                // so try that
                //通过反射的方式设置值，假设这是一个插入语句，我要插入一个User对象，values存储着一个主键，那么
                //我就给这个参数设置了一个主键
                setValue(metaParam, keyProperties[0], values.get(0));
              }
            } else {
              //设置复合主键
              //(*1*)
              handleMultipleProperties(keyProperties, metaParam, metaResult);
            }
          }
        }
      }
    } catch (ExecutorException e) {
      throw e;
    } catch (Exception e) {
      throw new ExecutorException("Error selecting key or setting result to parameter object. Cause: " + e, e);
    }
  }
  
  //(*1*)
  private void org.apache.ibatis.executor.keygen.SelectKeyGenerator.handleMultipleProperties(String[] keyProperties,
      MetaObject metaParam, MetaObject metaResult) {
      
      //获取列名
    String[] keyColumns = keyStatement.getKeyColumns();
      
    if (keyColumns == null || keyColumns.length == 0) {
      // no key columns specified, just use the property names
      for (int i = 0; i < keyProperties.length; i++) {
        //循环反射设置值
        setValue(metaParam, keyProperties[i], metaResult.getValue(keyProperties[i]));
      }
    } else {
      if (keyColumns.length != keyProperties.length) {
        //如果列数和要设置的属性值个数不同，报错
        throw new ExecutorException("If SelectKey has key columns, the number must match the number of key properties.");
      }
      for (int i = 0; i < keyProperties.length; i++) {
        //通过列名获取值，并反射设置值到参数中
        setValue(metaParam, keyProperties[i], metaResult.getValue(keyColumns[i]));
      }
    }
  }
  
```

好了，主键设置好了，那么自然可以安心的插入到表中了
在执行我们的sql前，我们的sql在mybatis中还是一个sqlSource，具体的就是一个个sqlNode

```
public BoundSql org.apache.ibatis.mapping.MappedStatement.getBoundSql(Object parameterObject) {
    //获取BoundSql，维护被解析后的sql，参数映射
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    //获取参数映射集合
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings == null || parameterMappings.isEmpty()) {
      //如果为空，那么创建一个新的，List<ParameterMapping>从MappedStatement的parameterMap属性中获取
      boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
    }

    // check for nested result maps in parameter mappings (issue #30)
    //检查嵌入在ParameterMapping的ResultMap
    for (ParameterMapping pm : boundSql.getParameterMappings()) {
      String rmId = pm.getResultMapId();
      if (rmId != null) {
        ResultMap rm = configuration.getResultMap(rmId);
        if (rm != null) {
          //如果参数中确实存在这样的嵌入ResultMap，那么与原来的hasNestedResultMaps进行相与
          //主要是对应的参数可能是一个复杂类型，需要进行映射
          hasNestedResultMaps |= rm.hasNestedResultMaps();
        }
      }
    }
    //返回boundSql
    return boundSql;
  }
```
通常我们使用的sql都属于比较复杂的sql，难免会存在一些动态标签，所以我们就选DynamicSqlSource来分析构建BoundSql的过程

```
 public BoundSql org.apache.ibatis.scripting.xmltags.DynamicSqlSource.getBoundSql(Object parameterObject) {
    //构建动态sql上下文
    //(*1*)
    DynamicContext context = new DynamicContext(configuration, parameterObject);
    //开始沿着sqlNode树往下执行,主要是拼接sql，比如有if这样的标签，会执行text判断
    rootSqlNode.apply(context);
    //sql建造者
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    //获取参数类型
    Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
    //进行#{}的替换，替换成？，里面的表达式解析成ParameterMapping，尚不会进行Ognl表达式的执行
    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    for (Map.Entry<String, Object> entry : context.getBindings().entrySet()) {
      //设置偏好参数设置，其中包括_parameter,_databaseId
      boundSql.setAdditionalParameter(entry.getKey(), entry.getValue());
    }
    return boundSql;
  }
  
  //(*1*)
  public DynamicContext(Configuration configuration, Object parameterObject) {
    //如果有参数值，并且不是Map的话，那么包装成ContextMap，它继承自HashMap
    //它有个成员变量引用这个参数的MetaObject
    if (parameterObject != null && !(parameterObject instanceof Map)) {
      MetaObject metaObject = configuration.newMetaObject(parameterObject);
      bindings = new ContextMap(metaObject);
    } else {
      //如果没有传入参数，构建一个参数MetaObject为null的ContextMap
      bindings = new ContextMap(null);
    }
    //添加_parameter -> parameterObject
    bindings.put(PARAMETER_OBJECT_KEY, parameterObject);
    //_databaseId -> configuration.getDatabaseId()
    bindings.put(DATABASE_ID_KEY, configuration.getDatabaseId());
  }
```

> SqlNode.apply

我们挑选一个来分析，其他的请自行分析

```
<foreach collection="list" index="index" item="item" open="(" separator="," close=")">

public boolean org.apache.ibatis.scripting.xmltags.ForEachSqlNode.apply(DynamicContext context) {
    //获取上下文绑定的参数
    Map<String, Object> bindings = context.getBindings();
    //表达式执行，从绑定的参数中获取对应的集合对象参数
    final Iterable<?> iterable = evaluator.evaluateIterable(collectionExpression, bindings);
    //如果是空集合，直接返回true
    if (!iterable.iterator().hasNext()) {
      return true;
    }
    boolean first = true;
    //在循环前添加open字符串，就是foreach标签上的open属性
    applyOpen(context);
    int i = 0;
    //循环
    for (Object o : iterable) {
      //记录旧的上下文
      DynamicContext oldContext = context;
      //如果是第一次循环
      if (first) {
        //构建一个前缀上下文，它继承了DynamicContext，第二参数表示它的前缀，每次进行append的时候都会先拼接上前缀
        context = new PrefixedContext(context, "");
      } else if (separator != null) {
        //第二次以后，会拼接分割符，每次append都会首先拼接separator
        context = new PrefixedContext(context, separator);
      } else {
            //其他的情况，拼接空字符
          context = new PrefixedContext(context, "");
      }
      //获取下标
      int uniqueNumber = context.getUniqueNumber();
      // Issue #709 
      if (o instanceof Map.Entry) {
        @SuppressWarnings("unchecked") 
        Map.Entry<Object, Object> mapEntry = (Map.Entry<Object, Object>) o;
        //向上下文中绑定key为index，value为mapEntry.getKey()（也就是专属于Map.Entry的下标） 和绑定key为__frch_index_uniqueNumber，value为mapEntry.getKey()的参数值
        applyIndex(context, mapEntry.getKey(), uniqueNumber);
        //向上下文中绑定key为item，value为mapEntry.getValue() 和绑定key为__frch_item_uniqueNumber，value为mapEntry.getValue()的参数值
        applyItem(context, mapEntry.getValue(), uniqueNumber);
      } else {
        //其他集合类型，绑定的是key为index，value为i（集合下标）和绑定key为__frch_index_uniqueNumber，value为i的参数值
        applyIndex(context, i, uniqueNumber);
        //其他集合类型，绑定的是key为item，value为循环到的值 和绑定key为__frch_item_uniqueNumber，value为循环到的值 
        applyItem(context, o, uniqueNumber);
      }
      //调用子节点的apply方法，但是你会发现这个DynamicContext变成了它的子类FilteredDynamicContext，那他有什么特别的呢？
      //(*1*)
      contents.apply(new FilteredDynamicContext(configuration, context, index, item, uniqueNumber));
      if (first) {
        //前缀是否已经被应用，默认是true，所以第二次之后就是拼接separator
        first = !((PrefixedContext) context).isPrefixApplied();
      }
      //恢复
      context = oldContext;
      i++;
    }
    //拼接close字符串
    applyClose(context);
    return true;
  }
  
  //(*1*)
  public void org.apache.ibatis.scripting.xmltags.ForEachSqlNode.FilteredDynamicContext.appendSql(String sql) {
        //解析占位符#{}
      GenericTokenParser parser = new GenericTokenParser("#{", "}", new TokenHandler() {
        @Override
        public String handleToken(String content) {
            //如果表达式是这样的#{item}或者#{item.name}，那么context为item，与下面的表达式符合，
            //那么就会被替换成__frch_item_index，最终返回值就是#{__frch_item_index}
          String newContent = content.replaceFirst("^\\s*" + item + "(?![^.,:\\s])", itemizeItem(item, index));
          //如果newContent和原来的一样，没有做什么改变，那么可能是#{itemIndex}，那么按照相同的逻辑处理它
          if (itemIndex != null && newContent.equals(content)) {
            newContent = content.replaceFirst("^\\s*" + itemIndex + "(?![^.,:\\s])", itemizeItem(itemIndex, index));
          }
          return new StringBuilder("#{").append(newContent).append("}").toString();
        }
      });
      //那么问题来了，为什么要这么做，很简单，如果不进行替换循环后拼接的sql语句全是一样的#{item}，你知道这个item取什么值吗？
      delegate.appendSql(parser.parse(sql));
    }
  
```
不管有多少父sqlNode，它都有叶子节点，其中TextSqlNode，StaticTextSqlNode就是属于叶子节点
StaticTextSqlNode就是完全静态的sql，不存在${}这样的占位符（可以有#{}），apply时直接append就可以了，但是TextSqlNode不同，它存在${}才会被构建成TextSqlNode
一下就是TextSqlNode的apply方法

```
public boolean TextSqlNode.apply(DynamicContext context) {
    GenericTokenParser parser = createParser(new BindingTokenParser(context, injectionFilter));
    context.appendSql(parser.parse(text));
    return true;
  }
```
占位符解析器是GenericTokenParser，它处理${}占位符，解析的逻辑就不做具体的分析了，相对于spring的占位符解析来说，它属于很简单的解析方式了，因为它只做了反斜杠的处理，没有占位符的嵌套，它不需要递归调用，也没有默认值的情况，tokenHandler是BindingTokenParser，看下它是怎么处理表达式的

```
public String BindingTokenParser.handleToken(String content) {
      //获取参数
      Object parameter = context.getBindings().get("_parameter");
      if (parameter == null) {
        //如果参数为null，那么设置value -》 null
        context.getBindings().put("value", null);
      } else if (SimpleTypeRegistry.isSimpleType(parameter.getClass())) {
        //如果是简单类型，设置value -> 参数
        context.getBindings().put("value", parameter);
      }
      //使用Ognl表达式处理
      Object value = OgnlCache.getValue(content, context.getBindings());
      String srtValue = (value == null ? "" : String.valueOf(value)); // issue #274 return "" instead of "null"
      //正则匹配，如果匹配不成功，抛出错误
      checkInjection(srtValue);
      return srtValue;
    }
```

> SqlSourceBuilder.parse

```
public SqlSource SqlSourceBuilder.parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
    //ParameterMapping处理器
    ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
    GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
    String sql = parser.parse(originalSql);
    return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
  }
```
先说明一下这个ParameterMappingTokenHandler，它实现了TokenHandler，专用处理占位符，但是它需要占位符解析器协助，比如${text}，需要有占位符解析器解析出里面的text，传递给TokenHandler处理

从上面的代码中可以看到，处理这样的参数符解析器是GenericTokenParser，处理的占位符为#{}，下面看到ParameterMappingTokenHandler是怎么处理#{}里面的内容的

```
public String ParameterMappingTokenHandler.handleToken(String content) {
      //构建ParameterMapping
      parameterMappings.add(buildParameterMapping(content));
      //返回问号
      return "?";
    }
```

构建ParameterMapping

```
private ParameterMapping org.apache.ibatis.builder.SqlSourceBuilder.ParameterMappingTokenHandler.buildParameterMapping(String content) {
      //解析ParameterMapping
      Map<String, String> propertiesMap = parseParameterMapping(content);
      。。。。。。
    }
```
解析#{}表达式内容ParameterExpression，它继承HashMap

```
public ParameterExpression(String expression) {
    parse(expression);
  }
  
  private void parse(String expression) {
    //跳过空白字符
    int p = skipWS(expression, 0);
    //解析表达式（表达式）
    if (expression.charAt(p) == '(') {
      //(*1*)
      expression(expression, p + 1);
    } else {
      //解析属性
      //(*5*)
      property(expression, p);
    }
  }
  
  //(*1*)
  private void expression(String expression, int left) {
    int match = 1;//分组个数
    int right = left + 1;
    while (match > 0) {
      //如果找到分组的闭合符号，分组个数减1
      if (expression.charAt(right) == ')') {
        match--;
        //如果又找到了一个（，说明存在嵌入的分组表达式
      } else if (expression.charAt(right) == '(') {
        match++;
      }
      right++;
    }
    //截取表达式字段
    put("expression", expression.substring(left, right - 1));
    //解析jdbcType
    //(*2*)
    jdbcTypeOpt(expression, right);
  }
  
  //(*2*)
  private void jdbcTypeOpt(String expression, int p) {
    //跳过空白字符
    p = skipWS(expression, p);
    if (p < expression.length()) {
        //下一个字符为冒号，比如#{age:Integer}，这种解析为属性名：jdbcType
      if (expression.charAt(p) == ':') {
        //(*3*)
        jdbcType(expression, p + 1);
        //下一个字符为逗号
      } else if (expression.charAt(p) == ',') {
        //(*4*)
        option(expression, p + 1);
      } else {
        throw new BuilderException("Parsing error in {" + new String(expression) + "} in position " + p);
      }
    }
  }
  
  //(*3*)
  private void jdbcType(String expression, int p) {
    //跳过空白字符
    int left = skipWS(expression, p);
    //一直跳过，知道发现逗号或者到头了
    int right = skipUntil(expression, left, ",");
    if (right > left) {
      //获取jdbcType
      put("jdbcType", trimmedStr(expression, left, right));
    } else {
      throw new BuilderException("Parsing error in {" + new String(expression) + "} in position " + p);
    }
    //(*4*)
    option(expression, right + 1);
  }
  
  //(*4*)
  private void option(String expression, int p) {
    //跳过空白符
    int left = skipWS(expression, p);
    if (left < expression.length()) {
      //一直跳过知道发现等号或者到头了
      int right = skipUntil(expression, left, "=");
      String name = trimmedStr(expression, left, right);
      left = right + 1;
      //一直跳过，直到发现逗号或者到头了
      right = skipUntil(expression, left, ",");
      String value = trimmedStr(expression, left, right);
      //比如javaType=long,jdbcType=big int这种
      put(name, value);
      //递归
      option(expression, right + 1);
    }
  }
  
  //(*5*)
  private void property(String expression, int left) {
    if (left < expression.length()) {
      //一直跳过，直到发现逗号或者冒号
      int right = skipUntil(expression, left, ",:");
      //设置属性名，比如#{age,typeHandler=int}
      put("property", trimmedStr(expression, left, right));
      //(*3*)
      jdbcTypeOpt(expression, right);
    }
  }
```
经过以上分析，#{}里面的内容可以是一下几种形式
- #{age} 译为{"property":"age"}

- #{age,typeHandler=int,javaType=java.lang.Integer} 译为{"property":"age", "typeHandler":"int","javaType":"java.lang.Integer"}

- #{age:int} 译为{"property":"age", "jdbcType":"int"}

- #{age:int,typeHandler=int,javaType=java.lang.Integer} 译为{"property":"age", "jdbcType":"int", "typeHandler":"int","javaType":"java.lang.Integer"}

- #{(表达式)} 译为{"expression":"表达式"}

- #{(表达式),typeHandler=int,javaType=java.lang.Integer} 译为{"expression":"表达式", "typeHandler":"int","javaType":"java.lang.Integer"}

- #{(表达式):varchar} 译为{"expression":"表达式", "jdbcType":"int"}

- #{(表达式):varchar,typeHandler=int,javaType=java.lang.Integer} 译为{"expression":"表达式", "jdbcType":"int", "typeHandler":"int","javaType":"java.lang.Integer"}

解析之后开始构建parameterMapping

```
private ParameterMapping org.apache.ibatis.builder.SqlSourceBuilder.ParameterMappingTokenHandler.buildParameterMapping(String content) {
      //解析ParameterMapping
      Map<String, String> propertiesMap = parseParameterMapping(content);
      //获取属性名
      String property = propertiesMap.get("property");
      Class<?> propertyType;
      //判断绑定参数中是否存在有这个属性
      if (metaParameters.hasGetter(property)) { // issue #448 get type from additional params
        //获取这个属性的类型
        propertyType = metaParameters.getGetterType(property);
        //在参数绑定中没有值的话，那么认为只有一个参数，判断是否存在对应的参数类型的参数类型
      } else if (typeHandlerRegistry.hasTypeHandler(parameterType)) {
        propertyType = parameterType;
        //获取jdbcType类型，如果是游标类型，那么属性类型设置为ResultSet
      } else if (JdbcType.CURSOR.name().equals(propertiesMap.get("jdbcType"))) {
        propertyType = java.sql.ResultSet.class;
      } else if (property != null) {
        //直接从参数中找
        MetaClass metaClass = MetaClass.forClass(parameterType, configuration.getReflectorFactory());
        //如果存在，获取其类型
        if (metaClass.hasGetter(property)) {
          propertyType = metaClass.getGetterType(property);
        } else {
            //实在没有，设置为Object
          propertyType = Object.class;
        }
      } else {
        propertyType = Object.class;
      }
      ParameterMapping.Builder builder = new ParameterMapping.Builder(configuration, property, propertyType);
      Class<?> javaType = propertyType;
      String typeHandlerAlias = null;
      for (Map.Entry<String, String> entry : propertiesMap.entrySet()) {
        String name = entry.getKey();
        String value = entry.getValue();
        //java类型，如果设置了java类型，那么会覆盖上面解析java类型
        if ("javaType".equals(name)) {
          javaType = resolveClass(value);
          builder.javaType(javaType);
          //设置jdbcType
        } else if ("jdbcType".equals(name)) {
          builder.jdbcType(resolveJdbcType(value));
          //设置参数模式，IN,OUT,INOUT
        } else if ("mode".equals(name)) {
          builder.mode(resolveParameterMode(value));
          //保留小数位数
        } else if ("numericScale".equals(name)) {
          builder.numericScale(Integer.valueOf(value));
          //嵌入的resultMapId
        } else if ("resultMap".equals(name)) {
          builder.resultMapId(value);
          //类型处理器
        } else if ("typeHandler".equals(name)) {
          typeHandlerAlias = value;
          //jdbcTypeName名
        } else if ("jdbcTypeName".equals(name)) {
          builder.jdbcTypeName(value);
        } else if ("property".equals(name)) {
          // Do Nothing
        } else if ("expression".equals(name)) {
            //暂时不支持表达式
          throw new BuilderException("Expression based parameters are not supported yet");
        } else {
          throw new BuilderException("An invalid property '" + name + "' was found in mapping #{" + content + "}.  Valid properties are " + parameterProperties);
        }
      }
      //如果typeHandlerAlias不为空，那么获取Typehandler
      if (typeHandlerAlias != null) {
        builder.typeHandler(resolveTypeHandler(javaType, typeHandlerAlias));
      }
      //构建ParameterMapping
      return builder.build();
      
}

```
好了，#{}解析完了，回到SqlSourceBuilder的parse方法

```
public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
    ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
    GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
    String sql = parser.parse(originalSql);
    //构建成静态的sqlSource
    return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
  }
```
下一小节，我们继续分析修改语句的分析过程。
