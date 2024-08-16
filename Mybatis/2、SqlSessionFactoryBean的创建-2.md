接着上一小节的内容

a. 处理构造器元素

```
//constructor标签
private void processConstructorElement(XNode resultChild, Class<?> resultType, List<ResultMapping> resultMappings) throws Exception {
    List<XNode> argChildren = resultChild.getChildren();
    for (XNode argChild : argChildren) {
      List<ResultFlag> flags = new ArrayList<ResultFlag>();
      //添加构造其标记
      flags.add(ResultFlag.CONSTRUCTOR);
      //如果还是主键构造参数，那么再加一个表示主键的标记
      if ("idArg".equals(argChild.getName())) {
        flags.add(ResultFlag.ID);
      }
      //构建ResultMap
      resultMappings.add(buildResultMappingFromContext(argChild, resultType, flags));
    }
  }
```
构建ResultMap对象

```
private ResultMapping org.apache.ibatis.builder.xml.XMLMapperBuilder.buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags) throws Exception {
    //获取属性名
    String property = context.getStringAttribute("property");
    //获取列名
    String column = context.getStringAttribute("column");
    //获取java类型
    String javaType = context.getStringAttribute("javaType");
    //获取jdbc类型
    String jdbcType = context.getStringAttribute("jdbcType");
    //获取嵌入的select语句
    String nestedSelect = context.getStringAttribute("select");
    //获取嵌入的ResultMap，有些构造参数可能是一些复杂的类型，一般用于多表关联
    String nestedResultMap = context.getStringAttribute("resultMap",
        //如果没有resultMap属性，那么看当前标签是否是association，collection等，递归调用ResultMap解析方法
        processNestedResultMappings(context, Collections.<ResultMapping> emptyList()));
        //获取notNullColumn
    String notNullColumn = context.getStringAttribute("notNullColumn");
    //获取列前缀
    String columnPrefix = context.getStringAttribute("columnPrefix");
    //获取类型处理器
    String typeHandler = context.getStringAttribute("typeHandler");
    //获取resultSet
    String resultSet = context.getStringAttribute("resultSet");
    //外键
    String foreignColumn = context.getStringAttribute("foreignColumn");
    //取值方式，懒加载还是饥饿加载
    boolean lazy = "lazy".equals(context.getStringAttribute("fetchType", configuration.isLazyLoadingEnabled() ? "lazy" : "eager"));
    //解析java类型
    Class<?> javaTypeClass = resolveClass(javaType);
    @SuppressWarnings("unchecked")
    Class<? extends TypeHandler<?>> typeHandlerClass = (Class<? extends TypeHandler<?>>) resolveClass(typeHandler);
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    //构建ResultMapping对象
    //(*1*)
    return builderAssistant.buildResultMapping(resultType, property, column, javaTypeClass, jdbcTypeEnum, nestedSelect, nestedResultMap, notNullColumn, columnPrefix, typeHandlerClass, flags, resultSet, foreignColumn, lazy);
  }
  
  //(*1*)
  public ResultMapping buildResultMapping(
      Class<?> resultType,
      String property,
      String column,
      Class<?> javaType,
      JdbcType jdbcType,
      String nestedSelect,
      String nestedResultMap,
      String notNullColumn,
      String columnPrefix,
      Class<? extends TypeHandler<?>> typeHandler,
      List<ResultFlag> flags,
      String resultSet,
      String foreignColumn,
      boolean lazy) {
      //如果没有指定，那么解析java类型
    Class<?> javaTypeClass = resolveResultJavaType(resultType, property, javaType);
    //解析类型处理器，如果么有指定，那么从类型处理器注册中获取
    TypeHandler<?> typeHandlerInstance = resolveTypeHandler(javaTypeClass, typeHandler);
    //解析复合列名，比如column的名字为 {id=user_id,name=user_name}
    List<ResultMapping> composites = parseCompositeColumnName(column);
    //如果存在复合的类名，那么原始列名置空
    if (composites.size() > 0) {
      column = null;
    }
    //构建ResultMapping
    return new ResultMapping.Builder(configuration, property, column, javaTypeClass)
        .jdbcType(jdbcType)
        //namespace + "." + nestedSelect
        .nestedQueryId(applyCurrentNamespace(nestedSelect, true))
        //namespace + "." + nestedResultMap
        .nestedResultMapId(applyCurrentNamespace(nestedResultMap, true))
        .resultSet(resultSet)
        .typeHandler(typeHandlerInstance)
        .flags(flags == null ? new ArrayList<ResultFlag>() : flags)
        .composites(composites)
        //解析 {id,name,age}这样的字段
        .notNullColumns(parseMultipleColumnNames(notNullColumn))
        .columnPrefix(columnPrefix)
        .foreignColumn(foreignColumn)
        .lazy(lazy)
        //会进行额外的校验，比如nestedQueryId与nestedResultMapId不能同时存在
        .build();
  }
 
   
```
b. 鉴别器

```
private Discriminator processDiscriminatorElement(XNode context, Class<?> resultType, List<ResultMapping> resultMappings) throws Exception {
    //获取列名
    String column = context.getStringAttribute("column");
    //获取java类型
    String javaType = context.getStringAttribute("javaType");
    //获取jdbc类型
    String jdbcType = context.getStringAttribute("jdbcType");
    //获取类型处理器
    String typeHandler = context.getStringAttribute("typeHandler");
    Class<?> javaTypeClass = resolveClass(javaType);
    @SuppressWarnings("unchecked")
    Class<? extends TypeHandler<?>> typeHandlerClass = (Class<? extends TypeHandler<?>>) resolveClass(typeHandler);
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    Map<String, String> discriminatorMap = new HashMap<String, String>();
    //循环case标签
    for (XNode caseChild : context.getChildren()) {
        //获取值
      String value = caseChild.getStringAttribute("value");
      //获取resultMap，或者是select，association，collection
      String resultMap = caseChild.getStringAttribute("resultMap", processNestedResultMappings(caseChild, resultMappings));
      discriminatorMap.put(value, resultMap);
    }
    //构建鉴别器
    return builderAssistant.buildDiscriminator(resultType, column, javaTypeClass, jdbcTypeEnum, typeHandlerClass, discriminatorMap);
  }
  
```
鉴别器字段声明

```
public class Discriminator {
  //维护鉴别器中指定的属性，列名等等
  private ResultMapping resultMapping;
  //维护case标签中的value -》 resultMapId之间的关系
  private Map<String, String> discriminatorMap;

  Discriminator() {
  }
  。。。。。。
}
```

ResultMapping字段声明

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
  //结果集
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

当一个ResultMap下的所有ResultMapping解析完毕后，那么就是构建ResultMap对象，并注册

```
public ResultMap org.apache.ibatis.builder.MapperBuilderAssistant.addResultMap(
      String id,
      Class<?> type,
      String extend,
      Discriminator discriminator,
      List<ResultMapping> resultMappings,
      Boolean autoMapping) {
      //namespace + '.' + id
    id = applyCurrentNamespace(id, false);
    //namespace + '.' + extend
    extend = applyCurrentNamespace(extend, true);

    if (extend != null) {
        //判断是否存在需要继承的ResultMap，如果暂时没有，那么抛出异常，像这种只能全部解析完配置之后才能继续解析
      if (!configuration.hasResultMap(extend)) {
        throw new IncompleteElementException("Could not find a parent resultmap with id '" + extend + "'");
      }
      //获取父ResultMap
      ResultMap resultMap = configuration.getResultMap(extend);
      List<ResultMapping> extendedResultMappings = new ArrayList<ResultMapping>(resultMap.getResultMappings());
      //移除已经存在的ResultMapping
      extendedResultMappings.removeAll(resultMappings);
      // Remove parent constructor if this resultMap declares a constructor.
      boolean declaresConstructor = false;
      //判断是否定义了构造器
      for (ResultMapping resultMapping : resultMappings) {
        if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
          declaresConstructor = true;
          break;
        }
      }
      //如果存在构造器了，那么以子类的为准，移除父ResultMapp的（因为构建一个对象只能选择其中一个进行构建）
      if (declaresConstructor) {
        Iterator<ResultMapping> extendedResultMappingsIter = extendedResultMappings.iterator();
        while (extendedResultMappingsIter.hasNext()) {
          if (extendedResultMappingsIter.next().getFlags().contains(ResultFlag.CONSTRUCTOR)) {
            extendedResultMappingsIter.remove();
          }
        }
      }
      //把父ResultMap的ResultMapping全部加进来
      resultMappings.addAll(extendedResultMappings);
    }
    //构建ResultMap对象
    ResultMap resultMap = new ResultMap.Builder(configuration, id, type, resultMappings, autoMapping)
        .discriminator(discriminator)
        .build();
    //注册到configuration对象中
    configuration.addResultMap(resultMap);
    return resultMap;
  }
```
构建ResultMap

```
public ResultMap org.apache.ibatis.mapping.ResultMap.Builder.build() {
        //如果没有resultMap.id，肯定要报错啊
      if (resultMap.id == null) {
        throw new IllegalArgumentException("ResultMaps must have an id");
      }
      resultMap.mappedColumns = new HashSet<String>();
      resultMap.idResultMappings = new ArrayList<ResultMapping>();
      resultMap.constructorResultMappings = new ArrayList<ResultMapping>();
      resultMap.propertyResultMappings = new ArrayList<ResultMapping>();
      //循环所有解析到的resultMapping，进行分类存储到ResultMap中
      for (ResultMapping resultMapping : resultMap.resultMappings) {
        //是否存在嵌入的查询
        resultMap.hasNestedQueries = resultMap.hasNestedQueries || resultMapping.getNestedQueryId() != null;
        //判断是否存在嵌入的resultMap
        resultMap.hasNestedResultMaps = resultMap.hasNestedResultMaps || (resultMapping.getNestedResultMapId() != null && resultMapping.getResultSet() == null);
        //获取当前映射的列名
        final String column = resultMapping.getColumn();
        if (column != null) {
            //转换成大写
          resultMap.mappedColumns.add(column.toUpperCase(Locale.ENGLISH));
        } else if (resultMapping.isCompositeResult()) {
            //如果是符合列名
          for (ResultMapping compositeResultMapping : resultMapping.getComposites()) {
            final String compositeColumn = compositeResultMapping.getColumn();
            if (compositeColumn != null) {
                //设置列名
              resultMap.mappedColumns.add(compositeColumn.toUpperCase(Locale.ENGLISH));
            }
          }
        }
        //如果是构造器，那么将当前resultMapping分类到构造器resultMapping集合中
        if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
          resultMap.constructorResultMappings.add(resultMapping);
        } else {
            其他的一律为属性resultMapping
          resultMap.propertyResultMappings.add(resultMapping);
        }
        //如果包含id的，将resultMapping存储到idresultMapping集合中
        if (resultMapping.getFlags().contains(ResultFlag.ID)) {
          resultMap.idResultMappings.add(resultMapping);
        }
      }
      if (resultMap.idResultMappings.isEmpty()) {
        //如果没有找到idresultMapping，那么就把所有的resultMapping都设置idresultMapping
        resultMap.idResultMappings.addAll(resultMap.resultMappings);
      }
      // lock down collections
      //锁定集合，不允许再更改
      resultMap.resultMappings = Collections.unmodifiableList(resultMap.resultMappings);
      resultMap.idResultMappings = Collections.unmodifiableList(resultMap.idResultMappings);
      resultMap.constructorResultMappings = Collections.unmodifiableList(resultMap.constructorResultMappings);
      resultMap.propertyResultMappings = Collections.unmodifiableList(resultMap.propertyResultMappings);
      resultMap.mappedColumns = Collections.unmodifiableSet(resultMap.mappedColumns);
      return resultMap;
    }
  }
```

ResultMap字段说明

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
  //鉴别器
  private Discriminator discriminator;
  //是否存在嵌入的resultMap
  private boolean hasNestedResultMaps;
  //是否存在嵌入的查询
  private boolean hasNestedQueries;
  //是否进行自动映射
  private Boolean autoMapping;

  private ResultMap() {
  }
  
  。。。。。。
}
```
注册到configuration对象中

```
public void org.apache.ibatis.session.Configuration.addResultMap(ResultMap rm) {
    //resultMap的id -》 ResultMap对象
    resultMaps.put(rm.getId(), rm);
    //检查当前ResultMap的监听器光联的ResultMap是否存在configuration配置对象中
    //如果存在那么标记当前ResultMap的hasNestedResultMaps为true
    checkLocallyForDiscriminatedNestedResultMaps(rm);
    //检查configuration配置对象中的所有ResultMap的鉴别器关联的ResultMap是否存在configuration配置对象中
    //存在的会标记hasNestedResultMaps为true
    checkGloballyForDiscriminatedNestedResultMaps(rm);
  }
```

5. 解析sql片段


```
private void org.apache.ibatis.builder.xml.XMLMapperBuilder.sqlElement(List<XNode> list) throws Exception {
    if (configuration.getDatabaseId() != null) {
      sqlElement(list, configuration.getDatabaseId());
    }
    sqlElement(list, null);
  }

  private void sqlElement(List<XNode> list, String requiredDatabaseId) throws Exception {
    for (XNode context : list) {
      //获取sql标签上的databaseId标识
      String databaseId = context.getStringAttribute("databaseId");
      //获取sql标签上的id
      String id = context.getStringAttribute("id");
      //namespace + '.' + id
      id = builderAssistant.applyCurrentNamespace(id, false);
      //(*1*)
      if (databaseIdMatchesCurrent(id, databaseId, requiredDatabaseId)) {
        //储存匹配的sql片段
        //id -> sql片段XNode
        sqlFragments.put(id, context);
      }
    }
  }
  
  //(*1*)
  private boolean databaseIdMatchesCurrent(String id, String databaseId, String requiredDatabaseId) {
    if (requiredDatabaseId != null) {
      //如果不匹配返回false
      if (!requiredDatabaseId.equals(databaseId)) {
        return false;
      }
    } else {
        //如果没有要求dataBaseId，自己却定义了databaseId，相当于找不到娘的孩子，所以返回为true
      if (databaseId != null) {
        return false;
      }
      // skip this fragment if there is a previous one with a not null databaseId
      //如果已经存在，并且存在databaseId的那么跳过
      if (this.sqlFragments.containsKey(id)) {
        XNode context = this.sqlFragments.get(id);
        if (context.getStringAttribute("databaseId") != null) {
          return false;
        }
      }
    }
    return true;
  }
  
```
解析select，insert，update，delete

```
private void buildStatementFromContext(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
        //(*1*)
      buildStatementFromContext(list, configuration.getDatabaseId());
    }
    buildStatementFromContext(list, null);
  }

//(*1*)
  private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
      //XMLStatementBuilder
      final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
      try {
        //解析语句节点
        statementParser.parseStatementNode();
      } catch (IncompleteElementException e) {
        //如果解析失败，加入到待解析列表中
        configuration.addIncompleteStatement(statementParser);
      }
    }
  }
```

XMLStatementBuilder是用于解析sql语句的构建器，它也继承了BaseBuilder类


```
public void XMLStatementBuilder.parseStatementNode() {
    //获取语句
    String id = context.getStringAttribute("id");
    //获取databaseId
    String databaseId = context.getStringAttribute("databaseId");
    //是否匹配databaseId
    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
      return;
    }
    //获取能够获取的最大size
    Integer fetchSize = context.getIntAttribute("fetchSize");
    //超时时间
    Integer timeout = context.getIntAttribute("timeout");
    //获取parameterMapId
    String parameterMap = context.getStringAttribute("parameterMap");
    //获取参数类型
    String parameterType = context.getStringAttribute("parameterType");
    //解析参数类型
    Class<?> parameterTypeClass = resolveClass(parameterType);
    //获取resultMapId
    String resultMap = context.getStringAttribute("resultMap");
    //获取返回值类型
    String resultType = context.getStringAttribute("resultType");
    //获取sqlSource解析驱动器
    String lang = context.getStringAttribute("lang");
    LanguageDriver langDriver = getLanguageDriver(lang);
    //解析返回类型
    Class<?> resultTypeClass = resolveClass(resultType);
    //结果集类型
    String resultSetType = context.getStringAttribute("resultSetType");
    //语句类型，默认是StatementType.PREPARED类型，除此之外，还有其他的类型StatementType.STATEMENT, StatementType.CALLABLE
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    //解析结果集返回类型，比如滚动返回策略等等
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
    //获取节点名
    String nodeName = context.getNode().getNodeName();
    //获取sql命令的类型UNKNOWN, INSERT, UPDATE, DELETE, SELECT, FLUSH;
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    //刷新缓存，如果没有设置，默认不是select语句的都会刷新缓存
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    //是否开启缓存，如果没有配置并且当前语句是select的，那么默认是开启缓存的
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
    //结果顺序
    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

    // Include Fragments before parsing
    //构建include标签的转换器
    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
    //将include标签用sql片段的内容进行替换
    //(*1*)
    includeParser.applyIncludes(context.getNode());

    // Parse selectKey after includes and remove them.
    //处理selectKey
    processSelectKeyNodes(id, parameterTypeClass, langDriver);
    
    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
    //创建SqlSource
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    //获取resultSets
    String resultSets = context.getStringAttribute("resultSets");
    //获取主键属性名
    String keyProperty = context.getStringAttribute("keyProperty");
    //对应的主键列名
    String keyColumn = context.getStringAttribute("keyColumn");
    //主键生成器
    KeyGenerator keyGenerator;
    //主键语句id id + !selectKey
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    //namespace + "." + keyStatementId
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    //是否已经存在这个key生成器，存在就获取出来
    if (configuration.hasKeyGenerator(keyStatementId)) {
      keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
        //是否需要使用key生成，并且它得是插入语句
      keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
          configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
          ? new Jdbc3KeyGenerator() : new NoKeyGenerator();
    }
    //构建MappedStatement并添加到配置对象中
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered, 
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
  }
  
  
  //(*1*)
  private void applyIncludes(Node source, final Properties variablesContext) {
    //如果是include标签
    if (source.getNodeName().equals("include")) {
      // new full context for included SQL - contains inherited context and new variables from current include node
      Properties fullContext;
        //获取include标签上的refid
      String refid = getStringAttribute(source, "refid");
      // replace variables in include refid value
      //解析${}占位符
      refid = PropertyParser.parse(refid, variablesContext);
      //通过refid获取sql片段node
      Node toInclude = findSqlFragment(refid);
      //获取include标签的内的属性
      Properties newVariablesContext = getVariablesContext(source, variablesContext);
      if (!newVariablesContext.isEmpty()) {
        // merge contexts
        //合并属性
        fullContext = new Properties();
        fullContext.putAll(variablesContext);
        fullContext.putAll(newVariablesContext);
      } else {
        // no new context - use inherited fully
        fullContext = variablesContext;
      }
      //toInclude为sqlNode
      applyIncludes(toInclude, fullContext);
      //如果toInclude片段与source来源不是同一个文档
      if (toInclude.getOwnerDocument() != source.getOwnerDocument()) {
        //那么将toInclude的节点引入到source的文档流中
        toInclude = source.getOwnerDocument().importNode(toInclude, true);
      }
      //使用source替换成toInclude节点
      source.getParentNode().replaceChild(toInclude, source);
      while (toInclude.hasChildNodes()) {
        //将toInclude的子节点插入到toInclude的前面
        toInclude.getParentNode().insertBefore(toInclude.getFirstChild(), toInclude);
      }
      //移除掉toInclude
      toInclude.getParentNode().removeChild(toInclude);
    } else if (source.getNodeType() == Node.ELEMENT_NODE) {
      NodeList children = source.getChildNodes();
      for (int i=0; i<children.getLength(); i++) {
        //如果节点类型，获取其子阶段继续递归
        applyIncludes(children.item(i), variablesContext);
      }
    } else if (source.getNodeType() == Node.ATTRIBUTE_NODE && !variablesContext.isEmpty()) {
      // replace variables in all attribute values
      //如果是属性值，那么将属性值解析占位符后重新设置值
      source.setNodeValue(PropertyParser.parse(source.getNodeValue(), variablesContext));
    } else if (source.getNodeType() == Node.TEXT_NODE && !variablesContext.isEmpty()) {
      // replace variables ins all text nodes
      //替换掉占位符，然后重新设置节点值
      source.setNodeValue(PropertyParser.parse(source.getNodeValue(), variablesContext));
    }
  }
```

1. 处理selectKey

```
private void org.apache.ibatis.builder.xml.XMLStatementBuilder.processSelectKeyNodes(String id, Class<?> parameterTypeClass, LanguageDriver langDriver) {
    //获取selectKey节点
    List<XNode> selectKeyNodes = context.evalNodes("selectKey");
    if (configuration.getDatabaseId() != null) {
      parseSelectKeyNodes(id, selectKeyNodes, parameterTypeClass, langDriver, configuration.getDatabaseId());
    }
    //(*1*)
    parseSelectKeyNodes(id, selectKeyNodes, parameterTypeClass, langDriver, null);
    //移除selectKeyNodes节点
    removeSelectKeyNodes(selectKeyNodes);
  }
  
  //(*1*)
   private void parseSelectKeyNodes(String parentId, List<XNode> list, Class<?> parameterTypeClass, LanguageDriver langDriver, String skRequiredDatabaseId) {
    for (XNode nodeToHandle : list) {
        //parentId就是前面的语句id
      String id = parentId + SelectKeyGenerator.SELECT_KEY_SUFFIX;
      String databaseId = nodeToHandle.getStringAttribute("databaseId");
      if (databaseIdMatchesCurrent(id, databaseId, skRequiredDatabaseId)) {
        parseSelectKeyNode(id, nodeToHandle, parameterTypeClass, langDriver, databaseId);
      }
    }
  }

  /**
   * <selectKey resultClass="long" keyProperty="id">  
   *    SELECT LAST_INSERT_ID() AS ID  
   * </selectKey>
   */
  private void parseSelectKeyNode(String id, XNode nodeToHandle, Class<?> parameterTypeClass, LanguageDriver langDriver, String databaseId) {
    //获取返回值类型
    String resultType = nodeToHandle.getStringAttribute("resultType");
    //解析成对应的java类型，首先从类型别名中查找
    Class<?> resultTypeClass = resolveClass(resultType);
    //语句类型
    StatementType statementType = StatementType.valueOf(nodeToHandle.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    //获取在java中的主键属性名
    String keyProperty = nodeToHandle.getStringAttribute("keyProperty");
    //获取列名
    String keyColumn = nodeToHandle.getStringAttribute("keyColumn");
    //在语句执行之前或者之后获取主键
    boolean executeBefore = "BEFORE".equals(nodeToHandle.getStringAttribute("order", "AFTER"));

    //defaults
    //默认不开启一级缓存
    boolean useCache = false;
    boolean resultOrdered = false;
    KeyGenerator keyGenerator = new NoKeyGenerator();
    Integer fetchSize = null;
    Integer timeout = null;
    boolean flushCache = false;
    String parameterMap = null;
    String resultMap = null;
    ResultSetType resultSetTypeEnum = null;
    //创建SqlSource
    SqlSource sqlSource = langDriver.createSqlSource(configuration, nodeToHandle, parameterTypeClass);
    //sql命令类型为select
    SqlCommandType sqlCommandType = SqlCommandType.SELECT;
    //注册语句
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered,
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, null);
    //namespace + "." + id
    id = builderAssistant.applyCurrentNamespace(id, false);
    //获取已经注册的语句
    MappedStatement keyStatement = configuration.getMappedStatement(id, false);
    //添加查询key生成器
    configuration.addKeyGenerator(id, new SelectKeyGenerator(keyStatement, executeBefore));
  }
  
```

创建SqlSource

```
public SqlSource org.apache.ibatis.scripting.xmltags.XMLLanguageDriver.createSqlSource(Configuration configuration, XNode script, Class<?> parameterType) {
    XMLScriptBuilder builder = new XMLScriptBuilder(configuration, script, parameterType);
    return builder.parseScriptNode();
  }
```
XMLScriptBuilder也继承了BaseBuilder


```
public SqlSource XMLLanguageDriver.parseScriptNode() {
    //解析成SqlNode
    //(*1*)
    List<SqlNode> contents = parseDynamicTags(context);
    //包装成MixedSqlNode，内部持有一个List<SqlNode>
    MixedSqlNode rootSqlNode = new MixedSqlNode(contents);
    SqlSource sqlSource = null;
    if (isDynamic) {
      //如果是动态sql，那么封装成DynamicSqlSource，动态sql的条件就是存在${}占位符，或者存在动态标签
      sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
    } else {
        //否则就是静态的RawSqlSource
      sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
    }
    return sqlSource;
  }
    //解析标签
    //(*1*)
  List<SqlNode> parseDynamicTags(XNode node) {
    List<SqlNode> contents = new ArrayList<SqlNode>();
    NodeList children = node.getNode().getChildNodes();
    for (int i = 0; i < children.getLength(); i++) {
      XNode child = node.newXNode(children.item(i));
      //如果是文本节点
      if (child.getNode().getNodeType() == Node.CDATA_SECTION_NODE || child.getNode().getNodeType() == Node.TEXT_NODE) {
        String data = child.getStringBody("");
        //封装为TextSqlNode
        TextSqlNode textSqlNode = new TextSqlNode(data);
        //判断是否为动态sql，只要判断是否存在${}占位符即可
        if (textSqlNode.isDynamic()) {
          contents.add(textSqlNode);
          //记录为动态sql
          isDynamic = true;
        } else {
          //如果不是动态上去了，重新封装成静态文本sqlNode
          contents.add(new StaticTextSqlNode(data));
        }
      } else if (child.getNode().getNodeType() == Node.ELEMENT_NODE) { // issue #628
        String nodeName = child.getNode().getNodeName();
        //根据节点名获取其节点处理器
        //(*2*)
        NodeHandler handler = nodeHandlers(nodeName);
        if (handler == null) {
          throw new BuilderException("Unknown element <" + nodeName + "> in SQL statement.");
        }
        //处理节点
        handler.handleNode(child, contents);
        isDynamic = true;
      }
    }
    return contents;
  }
  
  //(*2*)
  NodeHandler nodeHandlers(String nodeName) {
    Map<String, NodeHandler> map = new HashMap<String, NodeHandler>();
    map.put("trim", new TrimHandler());
    map.put("where", new WhereHandler());
    map.put("set", new SetHandler());
    map.put("foreach", new ForEachHandler());
    map.put("if", new IfHandler());
    map.put("choose", new ChooseHandler());
    map.put("when", new IfHandler());
    map.put("otherwise", new OtherwiseHandler());
    map.put("bind", new BindHandler());
    return map.get(nodeName);
  }
```
从上面的代码中可以看到，mybatis针对不同的动态标签实现了不同的SqlNode

下面我们随便挑选一个处理器来学习一下，比如IfHandler

```
public void handleNode(XNode nodeToHandle, List<SqlNode> targetContents) {
      //递归调用parseDynamicTags方法解析
      List<SqlNode> contents = parseDynamicTags(nodeToHandle);
      //将解析后的SqlNode封装成MixedSqlNode
      MixedSqlNode mixedSqlNode = new MixedSqlNode(contents);
      String test = nodeToHandle.getStringAttribute("test");
      //构建IfSqlNode对象
      IfSqlNode ifSqlNode = new IfSqlNode(mixedSqlNode, test);
      targetContents.add(ifSqlNode);
    }
```
从上面的SqlNode解析中可以看出，mybatis对每个被标签包括的SqlNode的子集都会使用一个叫MixedSqlNode的进行封装

至于动态的其他功能，比如apply方法，这是后面的内容，此处只大概提一下

最终形成的SqlSource对象

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0d6f5961975cc29d8d48e23f77e6715a.png)

好了，所有的东西都准备妥当，那么接下来就是构建MappedStatement的时候了


```
public MappedStatement MapperBuilderAssistant.addMappedStatement(
      String id,
      SqlSource sqlSource,
      StatementType statementType,
      SqlCommandType sqlCommandType,
      Integer fetchSize,
      Integer timeout,
      String parameterMap,
      Class<?> parameterType,
      String resultMap,
      Class<?> resultType,
      ResultSetType resultSetType,
      boolean flushCache,
      boolean useCache,
      boolean resultOrdered,
      KeyGenerator keyGenerator,
      String keyProperty,
      String keyColumn,
      String databaseId,
      LanguageDriver lang,
      String resultSets) {
    //如果存在未解析的缓存引用，抛出错误
    if (unresolvedCacheRef) {
      throw new IncompleteElementException("Cache-ref not yet resolved");
    }
    //namespace + "." + id
    id = applyCurrentNamespace(id, false);
    //是否是select语句
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    
    MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
        .resource(resource)
        .fetchSize(fetchSize)
        .timeout(timeout)
        .statementType(statementType)
        .keyGenerator(keyGenerator)
        .keyProperty(keyProperty)
        .keyColumn(keyColumn)
        .databaseId(databaseId)
        .lang(lang)
        .resultOrdered(resultOrdered)
        .resultSets(resultSets)
        //根据resultMapId获取ResultMap，如果没有设置resultMap，那么创建一个内联的resultMap
        //(*1*)
        .resultMaps(getStatementResultMaps(resultMap, resultType, id))
        .resultSetType(resultSetType)
        .flushCacheRequired(valueOrDefault(flushCache, !isSelect))
        .useCache(valueOrDefault(useCache, isSelect))
        //设置二级缓存
        .cache(currentCache);
    //通过parameterMapId获取parameterMap，和resultMap一样，在有parameterType，没有parameterMap时，创建一个
    //内联的ParameterMap
    ParameterMap statementParameterMap = getStatementParameterMap(parameterMap, parameterType, id);
    if (statementParameterMap != null) {
      statementBuilder.parameterMap(statementParameterMap);
    }
    //构建parameterMap
    MappedStatement statement = statementBuilder.build();
    //注册MappedStatement， id -> MappedStatement
    configuration.addMappedStatement(statement);
    return statement;
  }
  
  //(*1*)
  private List<ResultMap> getStatementResultMaps(
      String resultMap,
      Class<?> resultType,
      String statementId) {
    //namespace + “.” + resultMap
    resultMap = applyCurrentNamespace(resultMap, true);

    List<ResultMap> resultMaps = new ArrayList<ResultMap>();
    if (resultMap != null) {
      //通过逗号分割
      String[] resultMapNames = resultMap.split(",");
      for (String resultMapName : resultMapNames) {
        try {
            //获取ResultMap
          resultMaps.add(configuration.getResultMap(resultMapName.trim()));
        } catch (IllegalArgumentException e) {
          throw new IncompleteElementException("Could not find result map " + resultMapName, e);
        }
      }
    } else if (resultType != null) {
      //如果没有设置resultMap，但是有设置返回类型，那么构建一个内联ResultMap
      ResultMap inlineResultMap = new ResultMap.Builder(
          configuration,
          statementId + "-Inline",
          resultType,
          new ArrayList<ResultMapping>(),
          null).build();
      resultMaps.add(inlineResultMap);
    }
    return resultMaps;
  }
```
MappedStatement声明说明

```
public final class MappedStatement {
  //mapper配置文件的路径
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
  //结果集
  private String[] resultSets;

  MappedStatement() {
    // constructor disabled
  }
  。。。。。。
}
```

第三小节，继续mapper代理的注册和未解析映射的解析
