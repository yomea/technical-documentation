多结果集

下面的例子来自http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html

有两条sql语句
```
SELECT * FROM BLOG WHERE ID = #{id}

SELECT * FROM AUTHOR WHERE ID = #{id}
```

有下xml配置

```

<resultMap id="blogResult" type="Blog">
  <id property="id" column="id" />
  <result property="title" column="title"/>
  <association property="author" javaType="Author" resultSet="authors" column="author_id" foreignColumn="id">
    <id property="id" column="id"/>
    <result property="username" column="username"/>
    <result property="password" column="password"/>
    <result property="email" column="email"/>
    <result property="bio" column="bio"/>
  </association>
</resultMap>



<select id="selectBlog" resultSets="blogs,authors" resultMap="blogResult" statementType="CALLABLE">
  {call getBlogsAndAuthors(#{id,jdbcType=INTEGER,mode=IN})}
</select>
```
可以看到，假设不支持多结果集，那么对于Blog关联了Author的情况就需要多条sql语句来执行了，容易产生n+1的问题

现在的JDBC驱动支持多数据集的返回了，所以只要执行一次即可

那么拿到的第一个ResultSet就不包含Author的信息，在mybatis中是先通过一个PendingRelation来记录父属性映射，这里的父属性映射从这里的配置来看，它就是author属性

mybatis首先创建一个PendingRelation，它有两个属性


```
private static class PendingRelation {
    //属性的MetaObject对象，属性还未填充
    public MetaObject metaObject;
    //这个属性对应的属性映射
    public ResultMapping propertyMapping;
  }
```
这个author需要循环到下一个ResultSet时才能真正获取都值

下面继续嵌入ResultMap的分析,对于嵌入的，通常我们将使用多表关联的方式去执行sql

```
private void org.apache.ibatis.executor.resultset.DefaultResultSetHandler.handleRowValuesForNestedResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {

    //DefaultResultContext 维护这生成的返回对象
    final DefaultResultContext<Object> resultContext = new DefaultResultContext<Object>();
    //跳过rowBounds.getOffset
    skipRows(rsw.getResultSet(), rowBounds);
    Object rowValue = previousRowValue;
    
    //shouldProcessMoreRows 表示是否继续 受rowBounds.getLimit的限制
    while (shouldProcessMoreRows(resultContext, rowBounds) && rsw.getResultSet().next()) {
      //解析鉴别器，获取ResultMap
      final ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rsw.getResultSet(), resultMap, null);
      //创建缓存key（注意内部会循环ResultMapping获取对应的字段值做为缓存key的一部分，用于分组）
      final CacheKey rowKey = createRowKey(discriminatedResultMap, rsw, null);
      //从缓存中获取分组对象
      Object partialObject = nestedResultObjects.get(rowKey);
      // issue #577 && #542
      //是否清空嵌套引用，避免过多的对象导致内存不够用（这个属性的作用是官方的解释，有点蒙圈）
      if (mappedStatement.isResultOrdered()) {
        //如果嵌入的对象值为空，并且前一行的值不为空
        if (partialObject == null && rowValue != null) {
          //清理缓存
          nestedResultObjects.clear();
          //如果不是多结果集的，那么这个rowValue直接存储到resultContext中
          storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
        }
        //创建值
        //(*1*)
        rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
      } else {
        //partialObject 分组对象
        //(*1*)
        rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
        if (partialObject == null) {
          storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
        }
      }
    }
    //如果mappedStatement.isResultOrdered()为true，那么previousRowValue设置为null
    if (rowValue != null && mappedStatement.isResultOrdered() && shouldProcessMoreRows(resultContext, rowBounds)) {
      storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
      previousRowValue = null;
    } else if (rowValue != null) {
      previousRowValue = rowValue;
    }
  }
  
  //(*1*)
  private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, CacheKey combinedKey, String columnPrefix, Object partialObject) throws SQLException {
    final String resultMapId = resultMap.getId();
    //分组对象
    Object resultObject = partialObject;
    if (resultObject != null) {
      //包装分组对象为MetaObject，用于反射访问属性
      final MetaObject metaObject = configuration.newMetaObject(resultObject);
      //存储到ancestorObjects.put(resultMapId, resultObject);
      putAncestor(resultObject, resultMapId, columnPrefix);
      //获取子ResultMap，并创建对象，然后填充到父对象中（metaObject）
      //(*1*)
      applyNestedResultMappings(rsw, resultMap, metaObject, columnPrefix, combinedKey, false);
      //移除父对象
      ancestorObjects.remove(resultMapId);
    } else {
    
      final ResultLoaderMap lazyLoader = new ResultLoaderMap();
      //如果没有分组对象，创建分组对象，创建对象的过程不再赘述
      resultObject = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
      if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        //MetaObject
        final MetaObject metaObject = configuration.newMetaObject(resultObject);
        boolean foundValues = !resultMap.getConstructorResultMappings().isEmpty();
        //自动映射属性
        if (shouldApplyAutomaticMappings(resultMap, true)) {
          foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
        }
        //应用属性映射
        foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
        //存储ancestorObjects.put(resultMapId, resultObject);
        putAncestor(resultObject, resultMapId, columnPrefix);
        //获取嵌入的ResultMap，然后创建其对象，最后设置到父对象中（metaObject）
        //(*1*)
        foundValues = applyNestedResultMappings(rsw, resultMap, metaObject, columnPrefix, combinedKey, true) || foundValues;
        //移除父对象
        ancestorObjects.remove(resultMapId);
        foundValues = lazyLoader.size() > 0 || foundValues;
        resultObject = foundValues ? resultObject : null;
      }
      if (combinedKey != CacheKey.NULL_CACHE_KEY) {
        //缓存子对象
        nestedResultObjects.put(combinedKey, resultObject);
      }
    }
    return resultObject;
  }


//(*1*)
private boolean applyNestedResultMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String parentPrefix, CacheKey parentRowKey, boolean newObject) {
    boolean foundValues = false;
    //循环ResultMapping
    for (ResultMapping resultMapping : resultMap.getPropertyResultMappings()) {
      //获取嵌入的nestedResultMapId
      final String nestedResultMapId = resultMapping.getNestedResultMapId();
      //如果存在嵌入的resultMapId并且没有指定resultSet（有表示是多结果集的操作，不属于嵌入ResultMap）
      if (nestedResultMapId != null && resultMapping.getResultSet() == null) {
        try {
          //列前缀，parentPrefix +  resultMapping.getColumnPrefix
          final String columnPrefix = getColumnPrefix(parentPrefix, resultMapping);
          //从configuration中获取嵌入的ResultMap，并且检查是否存在鉴别器，如果有还会使用鉴别器获取ResultMap
          final ResultMap nestedResultMap = getNestedResultMap(rsw.getResultSet(), nestedResultMapId, columnPrefix);
          //根据嵌入的nestedResultMapId去获取对象，这种主要用于互相关联的两个对象，比如Husband关联wife，wife关联husband
          Object ancestorObject = ancestorObjects.get(nestedResultMapId);
          if (ancestorObject != null) {
            if (newObject) {
              //那么将这个ancestorObject设置到metaObject对象中
              linkObjects(metaObject, resultMapping, ancestorObject); // issue #385
            }
          } else {
            final CacheKey rowKey = createRowKey(nestedResultMap, rsw, columnPrefix);
            //构建联合缓存key
            final CacheKey combinedKey = combineKeys(rowKey, parentRowKey);
            //从nestedResultObjects获取缓存的值
            Object rowValue = nestedResultObjects.get(combinedKey);
            boolean knownValue = (rowValue != null);
            instantiateCollectionPropertyIfAppropriate(resultMapping, metaObject); // mandatory   
            //如果指定了NotNullColumn，那么通过ResultSet检查指定的所有字段的值是否为null，如果有一个为null，那么都会返回false
            //如果没有指定NotNullColumn，那么直接返回true
            if (anyNotNullColumnHasValue(resultMapping, columnPrefix, rsw.getResultSet())) {
              //递归调用
              rowValue = getRowValue(rsw, nestedResultMap, combinedKey, columnPrefix, rowValue);
              if (rowValue != null && !knownValue) {
                //将rowValue值设置到metaObject的resultMapping指定的属性上
                linkObjects(metaObject, resultMapping, rowValue);
                foundValues = true;
              }
            }
          }
        } catch (SQLException e) {
          throw new ExecutorException("Error getting nested result map values for '" + resultMapping.getProperty() + "'.  Cause: " + e, e);
        }
      }
    }
    return foundValues;
  }

```

所以mybatis，对于嵌入的ResultMap是通过缓存key来分组的，设置嵌入的值，如果嵌入的值还有嵌入的值，可以继续递归获取值，继续嵌入，还会缓存ResultMapId -> 对象的关系，因为有些对象之间会相互引用，那就没有必要重新创建了。

比如我们有Blog对象，它内部又一个属性autors，它是一个List<Autor>,那么对于join查询，我们肯定要根据ResultSet返回的结果进行分组，那么分组的key是怎么创建的呢？首先对于第一层对象的cacheKey创建代码如下：

```
private CacheKey org.apache.ibatis.executor.resultset.DefaultResultSetHandler#createRowKey(ResultMap resultMap, ResultSetWrapper rsw, String columnPrefix) throws SQLException {
    final CacheKey cacheKey = new CacheKey();
    //首先ResultMapId，表示这个结果集所属ResultMap
    cacheKey.update(resultMap.getId());
    //获取主键ResultMapping
    List<ResultMapping> resultMappings = getResultMappingsForRowKey(resultMap);
    //如果没有主键映射
    if (resultMappings.size() == 0) {
    	//并且映射的属性是Map的话，那么将所有的字段名和对应的字段值设置为cacheKey的一部分
      if (Map.class.isAssignableFrom(resultMap.getType())) {
        createRowKeyForMap(rsw, cacheKey);
      } else {
      //如果属性类型不是Map，那么将所有未进行映射的字段名和字段值做为cacheKey的一部分
        createRowKeyForUnmappedProperties(resultMap, rsw, cacheKey, columnPrefix);
      }
    } else {
    //将主键字段名和字段值做为cachekey的一部分，如果主键映射有嵌入映射的话，那么将嵌入映射的字段名和字段值做为cachekey的一部分
      createRowKeyForMappedProperties(resultMap, rsw, cacheKey, resultMappings, columnPrefix);
    }
    //如果没有映射任何字段，返回CacheKey.NULL_CACHE_KEY
    //按道理，如果存在一个字段映射都会update 2次，比如只有一个主键的情况下就是update 2次
    if (cacheKey.getUpdateCount() < 2) {
      return CacheKey.NULL_CACHE_KEY;
    }    
    return cacheKey;
  }
```
对于嵌入属性的嵌入属性，依次通过以下方法设置cachekey

```
private CacheKey org.apache.ibatis.executor.resultset.DefaultResultSetHandler#combineKeys(CacheKey rowKey, CacheKey parentRowKey) {
    if (rowKey.getUpdateCount() > 1 && parentRowKey.getUpdateCount() > 1) {
      CacheKey combinedKey;
      try {
        combinedKey = rowKey.clone();
      } catch (CloneNotSupportedException e) {
        throw new ExecutorException("Error cloning cache key.  Cause: " + e, e);
      }
      combinedKey.update(parentRowKey);
      return combinedKey;
    }
    return CacheKey.NULL_CACHE_KEY;
  }
```
非常简单，就是将父cachekey作为自己cachekey的一部分，保证唯一性。