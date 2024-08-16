回到XMLMapperBuilder的parse方法

```
public void org.apache.ibatis.builder.xml.XMLMapperBuilder.parse() {
    if (!configuration.isResourceLoaded(resource)) {
      //解析配置
      configurationElement(parser.evalNode("/mapper"));
      //记录已被加载mapper资源
      configuration.addLoadedResource(resource);
      //构建mapper代理
      bindMapperForNamespace();
    }
    //继续解析那些还未解析的映射，缓存引用，语句
    //解析的方法在第二节已经分析过，是一样的，所以此处不再赘述
    parsePendingResultMaps();
    parsePendingChacheRefs();
    parsePendingStatements();
  }
```
构建mapper代理

```
private void org.apache.ibatis.builder.xml.XMLMapperBuilder.bindMapperForNamespace() {
    //获取namespace
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
      Class<?> boundType = null;
      try {
        //加载mapper接口
        boundType = Resources.classForName(namespace);
      } catch (ClassNotFoundException e) {
        //如果没有，忽略，应为这个namespace不一定要是mapper接口
        //ignore, bound type is not required
      }
      //如果不为空
      if (boundType != null) {
        //如果没有注册这个mapper接口
        if (!configuration.hasMapper(boundType)) {
          // Spring may not know the real resource name so we set a flag
          // to prevent loading again this resource from the mapper interface
          // look at MapperAnnotationBuilder#loadXmlResource
          //记录这个namespace的mapper接口已被加载
          configuration.addLoadedResource("namespace:" + namespace);
          //注册mapper接口
          configuration.addMapper(boundType);
        }
      }
    }
  }
```
注册mapper接口

```
public <T> void org.apache.ibatis.session.Configuration.addMapper(Class<T> type) {
    //(*1*)
    mapperRegistry.addMapper(type);
  }
  
  //(*1*)
  public <T> void addMapper(Class<T> type) {
    //对应的类需要是接口，不是接口忽略
    if (type.isInterface()) {
        //如果已经存在，抛错
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        //注册,type -> MapperProxyFactory
        knownMappers.put(type, new MapperProxyFactory<T>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        //解析注解
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
      
        //如果加载有问题，移除
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }
```
注解解析

```
public MapperAnnotationBuilder(Configuration configuration, Class<?> type) {
    String resource = type.getName().replace('.', '/') + ".java (best guess)";
    this.assistant = new MapperBuilderAssistant(configuration, resource);
    this.configuration = configuration;
    this.type = type;
    //添加需要解析的注解
    sqlAnnotationTypes.add(Select.class);
    sqlAnnotationTypes.add(Insert.class);
    sqlAnnotationTypes.add(Update.class);
    sqlAnnotationTypes.add(Delete.class);

    sqlProviderAnnotationTypes.add(SelectProvider.class);
    sqlProviderAnnotationTypes.add(InsertProvider.class);
    sqlProviderAnnotationTypes.add(UpdateProvider.class);
    sqlProviderAnnotationTypes.add(DeleteProvider.class);
  }
```
MapperAnnotationBuilder.parse

```
public void org.apache.ibatis.builder.annotation.MapperAnnotationBuilder.parse() {

    String resource = type.toString();
    //是否已经加载
    if (!configuration.isResourceLoaded(resource)) {
      //加载xml配置，加载的配置与当前mapper在同一路径下，mapper接口类型
      //点替换成正斜杠，然后加上.xml后缀，具体的解析逻辑和第二小节的是一样的
      loadXmlResource();
      //记录资源被加载
      configuration.addLoadedResource(resource);
      //设置namespace
      assistant.setCurrentNamespace(type.getName());
      //解析缓存
      parseCache();
      //解析缓存引用
      parseCacheRef();
      //获取接口的所有方法
      Method[] methods = type.getMethods();
      for (Method method : methods) {
        try {
          // issue #237
          //不解析桥方法
          if (!method.isBridge()) {
            //解析语句
            parseStatement(method);
          }
        } catch (IncompleteElementException e) {
          //添加未解析的方法
          configuration.addIncompleteMethod(new MethodResolver(this, method));
        }
      }
    }
    //解析尚解析的方法
    parsePendingMethods();
  }
```
1. 解析缓存

```
private void org.apache.ibatis.builder.annotation.MapperAnnotationBuilder.parseCache() {
    //获取@CacheNamespace注解
    CacheNamespace cacheDomain = type.getAnnotation(CacheNamespace.class);
    if (cacheDomain != null) {
      //缓存大小
      Integer size = cacheDomain.size() == 0 ? null : cacheDomain.size();
      //刷新时间
      Long flushInterval = cacheDomain.flushInterval() == 0 ? null : cacheDomain.flushInterval();
      //这部分逻辑和解析xml时是一样的。
      assistant.useNewCache(cacheDomain.implementation(),
      cacheDomain.eviction(), flushInterval, size, cacheDomain.readWrite(), cacheDomain.blocking(), null);
    }
  }
```
2. 解析缓存引用

```
private void parseCacheRef() {
    //获取@CacheNamespaceRef注解
    CacheNamespaceRef cacheDomainRef = type.getAnnotation(CacheNamespaceRef.class);
    if (cacheDomainRef != null) {
      //和解析xml是一样的
      assistant.useCacheRef(cacheDomainRef.value().getName());
    }
  }
```
3. 解析statement

```
 void parseStatement(Method method) {
    //解析参数，除了RowBounds和ResultHandler外，如果超过一个参数，那么参数类型
    //定义为ParamMap，这是一个继承了HashMap的类
    Class<?> parameterTypeClass = getParameterType(method);
    //获取xml脚本驱动解析器
    LanguageDriver languageDriver = getLanguageDriver(method);
    //解析@select,@update,@delete注解，创建SqlSource
    SqlSource sqlSource = getSqlSourceFromAnnotations(method, parameterTypeClass, languageDriver);
    if (sqlSource != null) {
        //解析@Options注解
      Options options = method.getAnnotation(Options.class);
      //接口名+方法名
      final String mappedStatementId = type.getName() + "." + method.getName();
      Integer fetchSize = null;
      Integer timeout = null;
      StatementType statementType = StatementType.PREPARED;
      ResultSetType resultSetType = ResultSetType.FORWARD_ONLY;
      //解析sql命令类型，通过使用的注解来判断，如@Select，那就是select，或者通过@SelectProvider等等
      SqlCommandType sqlCommandType = getSqlCommandType(method);
      boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
      boolean flushCache = !isSelect;
      boolean useCache = isSelect;

      KeyGenerator keyGenerator;
      String keyProperty = "id";
      String keyColumn = null;
      if (SqlCommandType.INSERT.equals(sqlCommandType) || SqlCommandType.UPDATE.equals(sqlCommandType)) {
        // first check for SelectKey annotation - that overrides everything else
        //解析@SelectKey注解
        SelectKey selectKey = method.getAnnotation(SelectKey.class);
        if (selectKey != null) {
            //创建获取主键的MappedStatement并创建SelectKeyGenerator，除了从注解中获取信息之外，其他的操作和解析xml配置是一样的
          keyGenerator = handleSelectKeyAnnotation(selectKey, mappedStatementId, getParameterType(method), languageDriver);
          keyProperty = selectKey.keyProperty();
        } else if (options == null) {
          keyGenerator = configuration.isUseGeneratedKeys() ? new Jdbc3KeyGenerator() : new NoKeyGenerator();
        } else {
            //从@Options注解上获取主键属性名，列名，构建Jdbc3KeyGenerator主键生成器
          keyGenerator = options.useGeneratedKeys() ? new Jdbc3KeyGenerator() : new NoKeyGenerator();
          keyProperty = options.keyProperty();
          keyColumn = options.keyColumn();
        }
      } else {
        keyGenerator = new NoKeyGenerator();
      }

      if (options != null) {
        if (FlushCachePolicy.TRUE.equals(options.flushCache())) {
          flushCache = true;
        } else if (FlushCachePolicy.FALSE.equals(options.flushCache())) {
          flushCache = false;
        }
        useCache = options.useCache();
        fetchSize = options.fetchSize() > -1 || options.fetchSize() == Integer.MIN_VALUE ? options.fetchSize() : null; //issue #348
        timeout = options.timeout() > -1 ? options.timeout() : null;
        statementType = options.statementType();
        resultSetType = options.resultSetType();
      }

      String resultMapId = null;
      ResultMap resultMapAnnotation = method.getAnnotation(ResultMap.class);
      if (resultMapAnnotation != null) {
        String[] resultMaps = resultMapAnnotation.value();
        StringBuilder sb = new StringBuilder();
        for (String resultMap : resultMaps) {
          if (sb.length() > 0) {
            sb.append(",");
          }
          sb.append(resultMap);
        }
        resultMapId = sb.toString();
      } else if (isSelect) {
        resultMapId = parseResultMap(method);
      }
        //这里和解析xml的方式是一样的
      assistant.addMappedStatement(
          mappedStatementId,
          sqlSource,
          statementType,
          sqlCommandType,
          fetchSize,
          timeout,
          // ParameterMapID
          null,
          parameterTypeClass,
          resultMapId,
          getReturnType(method),
          resultSetType,
          flushCache,
          useCache,
          // TODO gcode issue #577
          false,
          keyGenerator,
          keyProperty,
          keyColumn,
          // DatabaseID
          null,
          languageDriver,
          // ResultSets
          options != null ? nullOrEmpty(options.resultSets()) : null);
    }
  }
```
a. 解析参数

```
private Class<?> getParameterType(Method method) {
    Class<?> parameterType = null;
    Class<?>[] parameterTypes = method.getParameterTypes();
    for (int i = 0; i < parameterTypes.length; i++) {
      if (!RowBounds.class.isAssignableFrom(parameterTypes[i]) && !ResultHandler.class.isAssignableFrom(parameterTypes[i])) {
        if (parameterType == null) {
          parameterType = parameterTypes[i];
        } else {
          // issue #135
          //除了RowBounds和ResultHandler外，如果超过一个参数，那么参数类型
          //定义为ParamMap，这是一个继承了HashMap的类
          parameterType = ParamMap.class;
        }
      }
    }
    return parameterType;
  }
```
b. getSqlSourceFromAnnotations

```
private SqlSource org.apache.ibatis.builder.annotation.MapperAnnotationBuilder.getSqlSourceFromAnnotations(Method method, Class<?> parameterType, LanguageDriver languageDriver) {
    try {
        //获取@Select，@Update，@Delete，@Insert注解中其中一个，通过循环筛选
      Class<? extends Annotation> sqlAnnotationType = getSqlAnnotationType(method);
      //获取@SelectProvider，@InsertProvider，@UpdateProvider，@DeleteProvider
      Class<? extends Annotation> sqlProviderAnnotationType = getSqlProviderAnnotationType(method);
      if (sqlAnnotationType != null) {
        if (sqlProviderAnnotationType != null) {
            //不能同时存在
          throw new BindingException("You cannot supply both a static SQL and SqlProvider to method named " + method.getName());
        }
        //获取对应注解
        Annotation sqlAnnotation = method.getAnnotation(sqlAnnotationType);
        //获取sql语句
        final String[] strings = (String[]) sqlAnnotation.getClass().getMethod("value").invoke(sqlAnnotation);
        //构建sqlSource
        return buildSqlSourceFromStrings(strings, parameterType, languageDriver);
      } else if (sqlProviderAnnotationType != null) {
        //解析提供者上的sql和获取提供者
        Annotation sqlProviderAnnotation = method.getAnnotation(sqlProviderAnnotationType);
        return new ProviderSqlSource(assistant.getConfiguration(), sqlProviderAnnotation);
      }
      return null;
    } catch (Exception e) {
      throw new BuilderException("Could not find value method on SQL annotation.  Cause: " + e, e);
    }
  }
```
c. buildSqlSourceFromStrings

```
private SqlSource buildSqlSourceFromStrings(String[] strings, Class<?> parameterTypeClass, LanguageDriver languageDriver) {
    final StringBuilder sql = new StringBuilder();
    //将片段连接起来
    for (String fragment : strings) {
      sql.append(fragment);
      sql.append(" ");
    }
    //使用脚本解析驱动解析
    //(*1*)
    return languageDriver.createSqlSource(configuration, sql.toString().trim(), parameterTypeClass);
  }
  
   //(*1*)
   public SqlSource createSqlSource(Configuration configuration, String script, Class<?> parameterType) {
    // issue #3
    //如果是以<script>开头，那么通过xml的方式解析
    if (script.startsWith("<script>")) {
      XPathParser parser = new XPathParser(script, false, configuration.getVariables(), new XMLMapperEntityResolver());
      //这里面的解析方法就和解析解析xml是一样的了
      return createSqlSource(configuration, parser.evalNode("/script"), parameterType);
    } else {
      // issue #127
      //如果不是脚本，那么直接创建textSqlNode
      script = PropertyParser.parse(script, configuration.getVariables());
      TextSqlNode textSqlNode = new TextSqlNode(script);
      if (textSqlNode.isDynamic()) {
        return new DynamicSqlSource(configuration, textSqlNode);
      } else {
        return new RawSqlSource(configuration, script, parameterType);
      }
    }
  }
```

d. ProviderSqlSource

```
 public ProviderSqlSource(Configuration config, Object provider) {
    String providerMethodName = null;
    try {
      this.sqlSourceParser = new SqlSourceBuilder(config);
      //提供者的type类型
      this.providerType = (Class<?>) provider.getClass().getMethod("type").invoke(provider);
      //获取提供者方法
      providerMethodName = (String) provider.getClass().getMethod("method").invoke(provider);

      for (Method m : this.providerType.getMethods()) {
        if (providerMethodName.equals(m.getName())) {
          if (m.getReturnType() == String.class) {
            //如果找到多个提供者方法，抛出错误
            if (providerMethod != null){
              throw new BuilderException("Error creating SqlSource for SqlProvider. Method '"
                      + providerMethodName + "' is found multiple in SqlProvider '" + this.providerType.getName()
                      + "'. Sql provider method can not overload.");
            }
            //设置提供者方法
            this.providerMethod = m;
            //解析@Param或者是以param1后去参数名列表
            this.providerMethodArgumentNames = extractProviderMethodArgumentNames(m);
          }
        }
      }
    } catch (BuilderException e) {
      throw e;
    } catch (Exception e) {
      throw new BuilderException("Error creating SqlSource for SqlProvider.  Cause: " + e, e);
    }
    if (this.providerMethod == null) {
      throw new BuilderException("Error creating SqlSource for SqlProvider. Method '"
          + providerMethodName + "' not found in SqlProvider '" + this.providerType.getName() + "'.");
    }
  }
```

好了，不管是配置的方式，还是注解的方式，改准备都已经准备好了，可以迎接请求了。