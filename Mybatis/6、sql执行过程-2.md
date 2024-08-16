接着上一小节的内容

```
public BoundSql org.apache.ibatis.scripting.xmltags.DynamicSqlSource.getBoundSql(Object parameterObject) {
    DynamicContext context = new DynamicContext(configuration, parameterObject);
    rootSqlNode.apply(context);
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
    //解析#{}，返回StaticSqlSource
    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
    //获取BoundSql
    /**
     * public BoundSql getBoundSql(Object parameterObject) {
     *      return new BoundSql(configuration, sql, parameterMappings, *parameterObject);
     *  }
     */
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    //将contextMap中的值以偏好参数的方式设置到boundSql中
    for (Map.Entry<String, Object> entry : context.getBindings().entrySet()) {
      boundSql.setAdditionalParameter(entry.getKey(), entry.getValue());
    }
    return boundSql;
  }
```
BoundSql的声明说明


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
接着之前的doUpdate方法

```
public int org.apache.ibatis.executor.SimpleExecutor.doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Statement stmt = null;
    try {
      //获取配置对象
      Configuration configuration = ms.getConfiguration();
      //创建StatementHandler
      StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
      //参数处理
      stmt = prepareStatement(handler, ms.getStatementLog());
      //调用StatementHandler的update方法
      return handler.update(stmt);
    } finally {
      closeStatement(stmt);
    }
  }
```
prepareStatement

```
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    //(*1*)
    Connection connection = getConnection(statementLog);
    //(*2*)
    stmt = handler.prepare(connection, transaction.getTimeout());
    //参数处理
    handler.parameterize(stmt);
    return stmt;
  }
  
  //(*1*)
  protected Connection getConnection(Log statementLog) throws SQLException {
    Connection connection = transaction.getConnection();
    //如果是debug模式，那么创建connection的代理
    if (statementLog.isDebugEnabled()) {
      return ConnectionLogger.newInstance(connection, statementLog, queryStack);
    } else {
      return connection;
    }
  }
  
  //(*2*)
  public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
      //创建Statement
      //(*3*)
      statement = instantiateStatement(connection);
      //设置超时时间
      setStatementTimeout(statement, transactionTimeout);
      //设置最大获取个数
      setFetchSize(statement);
      return statement;
    } catch (SQLException e) {
      closeStatement(statement);
      throw e;
    } catch (Exception e) {
      closeStatement(statement);
      throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
  }
  
  //(*3*)
  protected Statement instantiateStatement(Connection connection) throws SQLException {
    String sql = boundSql.getSql();
    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
      String[] keyColumnNames = mappedStatement.getKeyColumns();
      if (keyColumnNames == null) {
        //创建prepareStatement，并设置返回生成的主键
        return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
      } else {
        //创建prepareStatement，并设置返回对应keyColumnNames的主键
        return connection.prepareStatement(sql, keyColumnNames);
      }
    } else if (mappedStatement.getResultSetType() != null) {
    //创建prepareStatement，并设置游标属性
      return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    } else {
    //创建prepareStatement
      return connection.prepareStatement(sql);
    }
  }
```
参数处理

```
public void org.apache.ibatis.executor.statement.PreparedStatementHandler.parameterize(Statement statement) throws SQLException {
    //从前面可知这个parameterHandler是DefaultParameterHandler
    //(*1*)
    parameterHandler.setParameters((PreparedStatement) statement);
  }
  
  //(*1*)
  public void DefaultParameterHandler.setParameters(PreparedStatement ps) {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    //获取parameterMappings集合
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
      for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping parameterMapping = parameterMappings.get(i);
        //参数不是OUT模式，OUT模式一般用于存储过程的调用
        if (parameterMapping.getMode() != ParameterMode.OUT) {
          Object value;
          //获取属性名
          String propertyName = parameterMapping.getProperty();
          //是否在偏好参数中存在，比如mybatis生成的中间参数，非用户提供的
          if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
            //获取值
            value = boundSql.getAdditionalParameter(propertyName);
          } else if (parameterObject == null) {
            value = null;
          } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            //只有一个参数，并且存在TypeHandler的情况
            value = parameterObject;
          } else {
            //从传入的参数中获取
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            value = metaObject.getValue(propertyName);
          }
          TypeHandler typeHandler = parameterMapping.getTypeHandler();
          JdbcType jdbcType = parameterMapping.getJdbcType();
          if (value == null && jdbcType == null) {
            //空类型 数据库的NULL
            jdbcType = configuration.getJdbcTypeForNull();
          }
          try {
            //设置参数，假设是整数，那么会使用IntegerTypeHandler，ps.setInt(i + 1, value)
            typeHandler.setParameter(ps, i + 1, value, jdbcType);
          } catch (TypeException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          } catch (SQLException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          }
        }
      }
    }
  }
  
```
> PreparedStatementHandler.update

```
public int org.apache.ibatis.executor.statement.PreparedStatementHandler.update(Statement statement) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    //执行
    ps.execute();
    //获取影响的行数
    int rows = ps.getUpdateCount();
    //获取参数
    Object parameterObject = boundSql.getParameterObject();
    //获取主键生成器
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    //处理主键
    keyGenerator.processAfter(executor, mappedStatement, ps, parameterObject);
    return rows;
  }
```
语句执行后的主键处理，这里我们选择Jdbc3KeyGenerator来分析

```
public void org.apache.ibatis.executor.keygen.Jdbc3KeyGenerator.processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    //(*1*)
    processBatch(ms, stmt, getParameters(parameter));
  }
  
  //获取参数，还记得在调用Executor的update前，mybatis对参数做了特殊处理，如果是集合类型，会被包装成
  //一个map参数，通过collection或list或array做key
   private Collection<Object> getParameters(Object parameter) {
    Collection<Object> parameters = null;
    if (parameter instanceof Collection) {
      parameters = (Collection) parameter;
    } else if (parameter instanceof Map) {
      Map parameterMap = (Map) parameter;
      if (parameterMap.containsKey("collection")) {
        parameters = (Collection) parameterMap.get("collection");
      } else if (parameterMap.containsKey("list")) {
        parameters = (List) parameterMap.get("list");
      } else if (parameterMap.containsKey("array")) {
        parameters = Arrays.asList((Object[]) parameterMap.get("array"));
      }
    }
    if (parameters == null) {
      parameters = new ArrayList<Object>();
      parameters.add(parameter);
    }
    return parameters;
  }
  
  //(*1*)
  public void processBatch(MappedStatement ms, Statement stmt, Collection<Object> parameters) {
    ResultSet rs = null;
    try {
      //获取生成的主键结果集
      rs = stmt.getGeneratedKeys();
      final Configuration configuration = ms.getConfiguration();
      final TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
      final String[] keyProperties = ms.getKeyProperties();
      //获取结果集元数据
      final ResultSetMetaData rsmd = rs.getMetaData();
      TypeHandler<?>[] typeHandlers = null;
      if (keyProperties != null && rsmd.getColumnCount() >= keyProperties.length) {
        //循环参数
        for (Object parameter : parameters) {
          // there should be one row for each statement (also one for each parameter)
          if (!rs.next()) {
            break;
          }
          final MetaObject metaParam = configuration.newMetaObject(parameter);
          if (typeHandlers == null) {
            //批量获取类型处理器
            typeHandlers = getTypeHandlers(typeHandlerRegistry, metaParam, keyProperties, rsmd);
          }
          //填充主键
          //(*2*)
          populateKeys(rs, metaParam, keyProperties, typeHandlers);
        }
      }
    } catch (Exception e) {
      throw new ExecutorException("Error getting generated key or setting result to parameter object. Cause: " + e, e);
    } finally {
      if (rs != null) {
        try {
          rs.close();
        } catch (Exception e) {
          // ignore
        }
      }
    }
  }
  
  //(*2*)
   private void populateKeys(ResultSet rs, MetaObject metaParam, String[] keyProperties, TypeHandler<?>[] typeHandlers) throws SQLException {
    for (int i = 0; i < keyProperties.length; i++) {
      TypeHandler<?> th = typeHandlers[i];
      if (th != null) {
        //获取值，然后反射设置到参数对象中
        Object value = th.getResult(rs, i + 1);
        metaParam.setValue(keyProperties[i], value);
      }
    }
  }
```

## 三、查询sql的执行过程

回到我们的MapperMethod，上次我们分析了修改语句的执行，现在我们来看看查询语句的执行

```
 public Object MapperMethod.execute(SqlSession sqlSession, Object[] args) {
    Object result;
    。。。。。。
      //查询语句
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
    }。。。。。。
  }
```
上面的查询方法分为查询多个，单，map，除了游标外，其他的都是调用的SqlSession的selectList方法


```
public <E> List<E> org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      //获取MappedStatement
      MappedStatement ms = configuration.getMappedStatement(statement);
      //wrapCollection方法不再赘述
      //(*1*)
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
  
  //(*1*)
  public <E> List<E> org.apache.ibatis.executor.CachingExecutor.query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    //获取BoundSql，上面分析修改sql时分析过，就是解析sqlSource的占位符，构建ParameterMapping
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    //创建缓存key
    //(*2*)
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
  
  //(*2*)
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    //缓存key对象
    CacheKey cacheKey = new CacheKey();
    //namespace + "." + id
    cacheKey.update(ms.getId());
    //偏移
    cacheKey.update(Integer.valueOf(rowBounds.getOffset()));
    //个数
    cacheKey.update(Integer.valueOf(rowBounds.getLimit()));
    //sql字符串
    cacheKey.update(boundSql.getSql());
    //参数映射
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (int i = 0; i < parameterMappings.size(); i++) {
      ParameterMapping parameterMapping = parameterMappings.get(i);
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        //获取值
        if (boundSql.hasAdditionalParameter(propertyName)) {
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        //添加查询值
        cacheKey.update(value);
      }
    }
    if (configuration.getEnvironment() != null) {
      // issue #176 这种issue表示修复的bug
      //添加环境名
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }
```
所以构建一个查询语句的缓存key，它需要满足一下条件，所属环境，id，分页，查询条件就能确定唯一的sql，好了，下面继续

```
public <E> List<E> org.apache.ibatis.executor.CachingExecutor.query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
      //获取缓存对象
    Cache cache = ms.getCache();
    if (cache != null) {
        //是否需要刷新缓存，也就是清理缓存，比如有些过期的缓存，需要清理出去
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        //检查OUT模式的也就是程序调用，如果是存储过程这类sql程序，只要发现一个IN模式的参数就抛错不支持的错误
        //从前面生成缓存key可以看到，mybatis不持之OUT模式的参数
        ensureNoOutParams(ms, parameterObject, boundSql);
        @SuppressWarnings("unchecked")
        //从缓存中获取值
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          //如果为空，那么从数据库中获取
          list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          //存入缓存
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    //如果没有定义二级缓存，那么值接从数据库中获取
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```
continue


```
 public <E> List<E> org.apache.ibatis.executor.BaseExecutor.query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      //清理sqlSession级别的缓存
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      //从sqlSession级别的缓存中获取值
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        //处理存储过程，从LocallyCachedOutput中获取值，然后循环parameterMapping，找出所有不是IN模式的参数
        //然后通过反射从LocallyCachedOutput中获取的值获取值，然后设置到parameter参数中
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        //查询数据库
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        //处理延时加载
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      //如果缓存范围只是STATEMENT级别的，那么执行完就清理缓存
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }
```
continue

```
private <E> List<E> org.apache.ibatis.executor.BaseExecutor.queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    //向缓存中put一个占位符
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      //移除
      localCache.removeObject(key);
    }
    //存入缓存
    localCache.putObject(key, list);
    //如果是存储过程，还有把OUT，INOUT类型的参数设置到localOutputParameterCache中
    if (ms.getStatementType() == StatementType.CALLABLE) {
      //直接put进去
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
```
> SimpleExecutor.doQuery

```
public <E> List<E> org.apache.ibatis.executor.SimpleExecutor.doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.<E>query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
```
从上面的代码来看，处理StatementHandler#query之外，其他的都已经分析过了，所以这里不再赘述

```
public <E> List<E> org.apache.ibatis.executor.statement.PreparedStatementHandler.query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    //调用ResultSetHandler，对于Configuration对象来说，它创建默认是DefaultResultSetHandler
    return resultSetHandler.<E> handleResultSets(ps);
  }
```

> DefaultResultSetHandler.handleResultSets

```
public List<Object> org.apache.ibatis.executor.resultset.DefaultResultSetHandler.handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

    
    final List<Object> multipleResults = new ArrayList<Object>();

    int resultSetCount = 0;
    //获取ResultSet，然后进行包装，除此之外，这个ResultSetWrapper还有其他的字段，用于储存中间参数
    //比如从ResultSet解析的元数据，这些动作在构造器中完成。
    //比如列名列表，jdbc返回列的java类型，从configuration对象中获取的类型处理器等
    ResultSetWrapper rsw = getFirstResultSet(stmt);
    //获取结果映射对象集合
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    //校验，当ResultSet有值，却没有ResultMap是会报错，但是即使用户没有配置ResultMapId，只要有ResultType
    //mybatis就会创建一个默认的ResultMap
    validateResultMapsCount(rsw, resultMapCount);
    
    while (rsw != null && resultMapCount > resultSetCount) {
      //循环获取ResultMap
      ResultMap resultMap = resultMaps.get(resultSetCount);
      //处理返回结果集
      //multipleResults在上面new出来的空集合
      handleResultSet(rsw, resultMap, multipleResults, null);
      //兼容代码，用于兼容有问题的JDBC驱动，有些JDBC驱动再使用一次ResultSet，再次使用的时候
      //就没有值了，这里只是mybatis做出的补偿措施，所以按照正常的JDBC驱动，这个方法是不需要的
      //而且handleResultSet方法会把当前使用的ResultSet进行关闭
      rsw = getNextResultSet(stmt);
      //清理中间过程存储的参数
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }
    
    //处理多结果集内容，现在jdbc驱动支持多结果集，比如一个存储过程有多条select语句
    //它可以一次性执行，然后返回多个ResultSet，下面这段代码就是处理待关联结果集的代码
    //处理下一个结果集
     String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
      while (rsw != null && resultSetCount < resultSets.length) {
        //resultSets[resultSetCount] 对应在ResultMap中某个属性的 parentMapping.getResultSet()
        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
          //判断是否存在嵌入的ResultMapId
          String nestedResultMapId = parentMapping.getNestedResultMapId();
          ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
          //将嵌入的ResultMap的属性设置到父ResultMap的属性对象中
          handleResultSet(rsw, resultMap, null, parentMapping);
        }
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
      }
    }
    
    return collapseSingleResultList(multipleResults);
  }
```
处理结果集handleResultSet


```
private void org.apache.ibatis.executor.resultset.DefaultResultSetHandler.handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
    try {
      if (parentMapping != null) {
        handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
      } else {
        if (resultHandler == null) {
          //创建默认的结果处理器
          DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
          //处理每行结果
          handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
          //将处理的结果存储到multipleResults集合中
          multipleResults.add(defaultResultHandler.getResultList());
        } else {
          handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
        }
      }
    } finally {
      // issue #228 (close resultsets)
      //关闭结果集
      closeResultSet(rsw.getResultSet());
    }
  }
```
处理每行结果

```
 public void org.apache.ibatis.executor.resultset.DefaultResultSetHandler.handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    //是否存在嵌入的ResultMap
    if (resultMap.hasNestedResultMaps()) {
      //检查是否存在用户传入的RowBounds，通常对于这种嵌入的ResultMap做内存分页是不安全的
      //这里会抛出错误，当然可以通过设置configuration的safeRowBoundsEnabled为false
      ensureNoRowBounds();
      //检查ResultHandler是否存在，存在的话也会报安全问题
      checkResultHandler();
      handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    } else {
      //(*1*)
      handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    }
  }
  
  //(*1*)
  private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
      throws SQLException {
      //结果上下文，用于记录生成的结果对象
    DefaultResultContext<Object> resultContext = new DefaultResultContext<Object>();
    //跳过行，所谓跳过行就是根据用户设置的rowBounds将ResultSet的指针移动到offset处
    skipRows(rsw.getResultSet(), rowBounds);
    //shouldProcessMoreRows方法是用来控制读取的行数不会超过rowBounds.getLimit
    while (shouldProcessMoreRows(resultContext, rowBounds) && rsw.getResultSet().next()) {
      //通过鉴别器，获取将要使用的ResultMap，如果没有鉴别器，那么继续使用当前resultMap即可
      //我相信大家都能够猜到它是怎么执行的，无非就是从ResultSet获取到对应的字段的值，然后从鉴别器中获取到对应的ResultMapId
      //(*2*)
      ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rsw.getResultSet(), resultMap, null);
      //构建每行值的最终对象
      Object rowValue = getRowValue(rsw, discriminatedResultMap);
      //存储创建好的对象只，对于没有parentMapping的来说，直接设置到resultContext中
      storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
    }
  }
  
  //(*2*)
  public ResultMap org.apache.ibatis.executor.resultset.DefaultResultSetHandler.resolveDiscriminatedResultMap(ResultSet rs, ResultMap resultMap, String columnPrefix) throws SQLException {
    Set<String> pastDiscriminators = new HashSet<String>();
    //获取监听器
    Discriminator discriminator = resultMap.getDiscriminator();
    //如果存在鉴别器，将通过鉴别器获取ResultMap
    while (discriminator != null) {
      final Object value = getDiscriminatorValue(rs, discriminator, columnPrefix);
      //通过值从鉴别器维护的value->ResultMapId map中后去ResultMapId
      final String discriminatedMapId = discriminator.getMapIdFor(String.valueOf(value));
      if (configuration.hasResultMap(discriminatedMapId)) {
        //如果存在，那么获取这个ResultMap
        resultMap = configuration.getResultMap(discriminatedMapId);
        Discriminator lastDiscriminator = discriminator;
        //查看新获取到的ResultMap是否还存在监听器
        discriminator = resultMap.getDiscriminator();
        //如果上一个鉴别器和当前新获取的鉴别器是一样的或者通过鉴别器获取的ResultMap是一样的，那么可以退出了
        if (discriminator == lastDiscriminator || !pastDiscriminators.add(discriminatedMapId)) {
          break;
        }
      } else {
        break;
      }
    }
    return resultMap;
  }
```

每行记录的解析，封装成我们需要的对象

```
private Object org.apache.ibatis.executor.resultset.DefaultResultSetHandler.getRowValue(ResultSetWrapper rsw, ResultMap resultMap) throws SQLException {
    //构建懒加载器
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
    //创建对象
    //(*1*)
    Object resultObject = createResultObject(rsw, resultMap, lazyLoader, null);
    //对象不为空并且当前这个返回的类型不存在对应的TypeHandler（如果存在的话，在createResultObject方法里就已经使用了，直接返回了resultObject）
    if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
      //创建metaObject
      final MetaObject metaObject = configuration.newMetaObject(resultObject);
      //构造器映射是否为空，如果不为空，那么在createResultObject方法中就已经使用了构造映射去构造resultObject对象
      boolean foundValues = !resultMap.getConstructorResultMappings().isEmpty();
      //是否需要自动映射
      if (shouldApplyAutomaticMappings(resultMap, false)) {
        //自动映射
        foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, null) || foundValues;
      }
      //属性映射
      foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, null) || foundValues;
      foundValues = lazyLoader.size() > 0 || foundValues;
      resultObject = foundValues ? resultObject : null;
      return resultObject;
    }
    return resultObject;
  }
  
  //(*1*)
  private Object DefaultResultSetHandler.createResultObject(ResultSetWrapper rsw, ResultMap resultMap, ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {
    //构造器参数类型
    final List<Class<?>> constructorArgTypes = new ArrayList<Class<?>>();
    //构造参数
    final List<Object> constructorArgs = new ArrayList<Object>();
    //创建对象
    //(*2*)
    final Object resultObject = createResultObject(rsw, resultMap, constructorArgTypes, constructorArgs, columnPrefix);
    //如果创建的对象不为空，并且没有TypeHandler
    if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
      //获取属性映射
      final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
      for (ResultMapping propertyMapping : propertyMappings) {
        // issue gcode #109 && issue #149
        //检查是否存在嵌入的查询，如果存在并且是懒加载的，那么创建它的代理，当获取当前对象的某个属性时，发现这个属性需要懒加载
        //那么将会调用lazyLoader中的LoadPair中的ResultLoader，这个ResultLoader内部执行的逻辑和我们现在正在分析的逻辑是一样的，不进行赘述
        if (propertyMapping.getNestedQueryId() != null && propertyMapping.isLazy()) {
          return configuration.getProxyFactory().createProxy(resultObject, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
        }
      }
    }
    return resultObject;
  }
  
  //(*2*)
  private Object DefaultResultSetHandler.createResultObject(ResultSetWrapper rsw, ResultMap resultMap, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)
      throws SQLException {
     //获取将要返回的数据类型
    final Class<?> resultType = resultMap.getType();
    //MetaClass，通过反射提供对某对象的属性信息，构造器等信息的访问
    final MetaClass metaType = MetaClass.forClass(resultType, reflectorFactory);
    //获取构造器参数映射
    final List<ResultMapping> constructorMappings = resultMap.getConstructorResultMappings();
    //首先看这个返回的类型是否存在TypeHandler，如果存在直接使用这个TypeHandler.getResult即可
    if (hasTypeHandlerForResultObject(rsw, resultType)) {
      return createPrimitiveResultObject(rsw, resultMap, columnPrefix);
    } else if (!constructorMappings.isEmpty()) {
      //通过构造参数映射构建对象
      return createParameterizedResultObject(rsw, resultType, constructorMappings, constructorArgTypes, constructorArgs, columnPrefix);
    } else if (resultType.isInterface() || metaType.hasDefaultConstructor()) {
      //如果是接口（一般只支持List，Map，Set等等，mybatis自动给创建子类型），或者存在默认构造器的走这
      return objectFactory.create(resultType);
    } else if (shouldApplyAutomaticMappings(resultMap, false)) {
      //如果啥都没有，那只好自动映射了
      return createByConstructorSignature(rsw, resultType, constructorArgTypes, constructorArgs, columnPrefix);
    }
    throw new ExecutorException("Do not know how to create an instance of " + resultType);
  }
```
对于返回对象的构造，我们这里只研究两种，构造参数构造，自动映射构造，其他的比较简单，不过多的说明
首先先上构造参数映射

```
 Object createParameterizedResultObject(ResultSetWrapper rsw, Class<?> resultType, List<ResultMapping> constructorMappings,
      List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix) {
    boolean foundValues = false;
    for (ResultMapping constructorMapping : constructorMappings) {
      //获取java类型
      final Class<?> parameterType = constructorMapping.getJavaType();
      //获取数据库列名
      final String column = constructorMapping.getColumn();
      final Object value;
      try {
        //是否存在嵌入的查询
        if (constructorMapping.getNestedQueryId() != null) {
            //执行嵌入查询语句，构建对象，这里是构造参数创建，所以这里设置嵌入查询为懒加载是无效的
            //(*1*)
          value = getNestedQueryConstructorValue(rsw.getResultSet(), constructorMapping, columnPrefix);
        } else if (constructorMapping.getNestedResultMapId() != null) {
          //如果是嵌入ResultMap，那么和其他的ResultMap一样，调用getRowValue获取值，不再赘述
          final ResultMap resultMap = configuration.getResultMap(constructorMapping.getNestedResultMapId());
          value = getRowValue(rsw, resultMap);
        } else {
          //其他的，获取TypeHandler获取简单值
          final TypeHandler<?> typeHandler = constructorMapping.getTypeHandler();
          value = typeHandler.getResult(rsw.getResultSet(), prependPrefix(column, columnPrefix));
        }
      } catch (ResultMapException e) {
        throw new ExecutorException("Could not process result for mapping: " + constructorMapping, e);
      } catch (SQLException e) {
        throw new ExecutorException("Could not process result for mapping: " + constructorMapping, e);
      }
      //添加构造参数类型
      constructorArgTypes.add(parameterType);
      //构造参数
      constructorArgs.add(value);
      foundValues = value != null || foundValues;
    }
    //最后通过ObjectFactory创建对象，创建过程就不具体说明了，相信聪明的你能够知道的
    return foundValues ? objectFactory.create(resultType, constructorArgTypes, constructorArgs) : null;
  }
  
  //(*1*)
   private Object getNestedQueryConstructorValue(ResultSet rs, ResultMapping constructorMapping, String columnPrefix) throws SQLException {
    //获取嵌入的查询id
    final String nestedQueryId = constructorMapping.getNestedQueryId();
    //获取对应的查询语句
    final MappedStatement nestedQuery = configuration.getMappedStatement(nestedQueryId);
    //获取将要返回的嵌入语句查询参数类型，理解为子查询即可
    final Class<?> nestedQueryParameterType = nestedQuery.getParameterMap().getType();
    //创建嵌入的查询参数对象，如果constructorMapping的key只有一个，那么就是简单的获取到TypeHandler从rs中获取值返回即可
    //如果constructorMapping的key是composite的，也就是有多个，那么就要循环composite的ParameterMapping，创建查询对象即可
    final Object nestedQueryParameterObject = prepareParameterForNestedQuery(rs, constructorMapping, nestedQueryParameterType, columnPrefix);
    Object value = null;
    if (nestedQueryParameterObject != null) {
      //这段逻辑前面分析过
      final BoundSql nestedBoundSql = nestedQuery.getBoundSql(nestedQueryParameterObject);
      //构建缓存key
      final CacheKey key = executor.createCacheKey(nestedQuery, nestedQueryParameterObject, RowBounds.DEFAULT, nestedBoundSql);
      //获取java类型
      final Class<?> targetType = constructorMapping.getJavaType();
      //创建结果加载器，懒加载中维护的就是这个对象
      final ResultLoader resultLoader = new ResultLoader(configuration, executor, nestedQuery, nestedQueryParameterObject, targetType, key, nestedBoundSql);
      //执行语句获取值，这是一个完整的查询过程，里面所执行的逻辑和我们现在分析的逻辑是一样的，所以不再赘述
      value = resultLoader.loadResult();
    }
    return value;
  }
```

自动映射创建返回对象

```
private Object createByConstructorSignature(ResultSetWrapper rsw, Class<?> resultType, List<Class<?>> constructorArgTypes, List<Object> constructorArgs,
      String columnPrefix) throws SQLException {
      //循环所有的构造器
    for (Constructor<?> constructor : resultType.getDeclaredConstructors()) {
    //寻找到构造参数和数据库ResultSet返回的参数类型是一样的构造器
      if (typeNames(constructor.getParameterTypes()).equals(rsw.getClassNames())) {
        boolean foundValues = false;
        for (int i = 0; i < constructor.getParameterTypes().length; i++) {
          Class<?> parameterType = constructor.getParameterTypes()[i];
          String columnName = rsw.getColumnNames().get(i);
          //获取typeHandler
          TypeHandler<?> typeHandler = rsw.getTypeHandler(parameterType, columnName);
          Object value = typeHandler.getResult(rsw.getResultSet(), prependPrefix(columnName, columnPrefix));
          constructorArgTypes.add(parameterType);
          constructorArgs.add(value);
          foundValues = value != null || foundValues;
        }
        //反射构造器创建对象
        return foundValues ? objectFactory.create(resultType, constructorArgTypes, constructorArgs) : null;
      }
    }
    throw new ExecutorException("No constructor found in " + resultType.getName() + " matching " + rsw.getClassNames());
  }
```
返回的对象是创建好了，但是还没有填充属性啊，填充属性又分为两种，一种是自动映射，另一种就是用户在xml中配置的属性映射
首先来看自动映射


```
private boolean applyAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
    //(*1*)
    //创建映射
    List<UnMappedColumnAutoMapping> autoMapping = createAutomaticMappings(rsw, resultMap, metaObject, columnPrefix);
    boolean foundValues = false;
    if (autoMapping.size() > 0) {
      for (UnMappedColumnAutoMapping mapping : autoMapping) {
        //循环设置值
        final Object value = mapping.typeHandler.getResult(rsw.getResultSet(), mapping.column);
        // issue #377, call setter on nulls
        if (value != null || configuration.isCallSettersOnNulls()) {
          if (value != null || !mapping.primitive) {
            metaObject.setValue(mapping.property, value);
          }
          foundValues = true;
        }
      }
    }
    return foundValues;
  }
  
  //(*1*)
  private List<UnMappedColumnAutoMapping> createAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
    final String mapKey = resultMap.getId() + ":" + columnPrefix;
    //从缓存中获取
    List<UnMappedColumnAutoMapping> autoMapping = autoMappingsCache.get(mapKey);
    //如果没有，重新获取
    if (autoMapping == null) {
      autoMapping = new ArrayList<UnMappedColumnAutoMapping>();
      //从ResultSet中获取列名
      final List<String> unmappedColumnNames = rsw.getUnmappedColumnNames(resultMap, columnPrefix);
      for (String columnName : unmappedColumnNames) {
        String propertyName = columnName;
        if (columnPrefix != null && !columnPrefix.isEmpty()) {
          // When columnPrefix is specified,
          // ignore columns without the prefix.
          if (columnName.toUpperCase(Locale.ENGLISH).startsWith(columnPrefix)) {
            propertyName = columnName.substring(columnPrefix.length());
          } else {
            continue;
          }
        }
        //是否存在与执行sql后的列名一样的属性，第二参数是指定是否需要进行驼峰转换
        final String property = metaObject.findProperty(propertyName, configuration.isMapUnderscoreToCamelCase());
        if (property != null && metaObject.hasSetter(property)) {
          final Class<?> propertyType = metaObject.getSetterType(property);
          //好了，下面就多说了，比较简单
          if (typeHandlerRegistry.hasTypeHandler(propertyType, rsw.getJdbcType(columnName))) {
            final TypeHandler<?> typeHandler = rsw.getTypeHandler(propertyType, columnName);
            autoMapping.add(new UnMappedColumnAutoMapping(columnName, property, typeHandler, propertyType.isPrimitive()));
          } else {
            //AutoMappingUnknownColumnBehavior.NONE，它的doAction方法为空
            configuration.getAutoMappingUnknownColumnBehavior()
                    .doAction(mappedStatement, columnName, property, propertyType);
          }
        } else{
           //AutoMappingUnknownColumnBehavior.NONE，它的doAction方法为空
          configuration.getAutoMappingUnknownColumnBehavior()
                  .doAction(mappedStatement, columnName, (property != null) ? property : propertyName, null);
        }
      }
      //加入缓存
      autoMappingsCache.put(mapKey, autoMapping);
    }
    return autoMapping;
  }
```
属性映射

```
private boolean applyPropertyMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
      //获取sql执行后映射的字段名与用于在ResultMap中定义的自定名进行合并获取已经映射的字段
      //ResultSetWrapper还会记录未映射的字段
    final List<String> mappedColumnNames = rsw.getMappedColumnNames(resultMap, columnPrefix);
    boolean foundValues = false;
    final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
    for (ResultMapping propertyMapping : propertyMappings) {
      String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
      if (propertyMapping.getNestedResultMapId() != null) {
        // the user added a column attribute to a nested result map, ignore it
        column = null;
      }
      //是否为复合的返回结果，是否已被映射的字段
      if (propertyMapping.isCompositeResult()
          || (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH)))
          || propertyMapping.getResultSet() != null) {
          //(*1*)
        Object value = getPropertyMappingValue(rsw.getResultSet(), metaObject, propertyMapping, lazyLoader, columnPrefix);
        // issue #541 make property optional
        //获取属性名
        final String property = propertyMapping.getProperty();
        // issue #377, call setter on nulls
        if (value != DEFERED
            && property != null
            && (value != null || (configuration.isCallSettersOnNulls() && !metaObject.getSetterType(property).isPrimitive()))) {
            //设置值
          metaObject.setValue(property, value);
        }
        if (property != null && (value != null || value == DEFERED)) {
          foundValues = true;
        }
      }
    }
    return foundValues;
  }
  
  //(*1*)
  private Object getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
      //是否存在嵌入的查询，这个嵌入查询我们前面分析构造参数的时候分析过，不再赘述
    if (propertyMapping.getNestedQueryId() != null) {
      return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);
    } else if (propertyMapping.getResultSet() != null) {
      //添加待关联的属性值
      //(*2*)
      addPendingChildRelation(rs, metaResultObject, propertyMapping);   // TODO is that OK?
      return DEFERED;
    } else {
      //获取typeHandler，然后获取值
      final TypeHandler<?> typeHandler = propertyMapping.getTypeHandler();
      final String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
      return typeHandler.getResult(rs, column);
    }
  }
  
  //(*2*)
  private void addPendingChildRelation(ResultSet rs, MetaObject metaResultObject, ResultMapping parentMapping) throws SQLException {
    //创建多结果集缓存key
    CacheKey cacheKey = createKeyForMultipleResults(rs, parentMapping, parentMapping.getColumn(), parentMapping.getColumn());
    //创建待关联对象
    PendingRelation deferLoad = new PendingRelation();
    //设置返回对象
    deferLoad.metaObject = metaResultObject;
    //设置对应返回对象的某个属性映射
    deferLoad.propertyMapping = parentMapping;
    //从缓冲中获取
    List<PendingRelation> relations = pendingRelations.get(cacheKey);
    // issue #255
    if (relations == null) {
      //为空，构建一个新的
      relations = new ArrayList<DefaultResultSetHandler.PendingRelation>();
      pendingRelations.put(cacheKey, relations);
    }
    relations.add(deferLoad);
    //获取缓存的属性映射
    ResultMapping previous = nextResultMaps.get(parentMapping.getResultSet());
    if (previous == null) {
      nextResultMaps.put(parentMapping.getResultSet(), parentMapping);
    } else {
      //如果缓存中的和当前的不一样，抛出错误
      if (!previous.equals(parentMapping)) {
        throw new ExecutorException("Two different properties are mapped to the same resultSet");
      }
    }
  }
```
多结果集下一下接继续探讨。