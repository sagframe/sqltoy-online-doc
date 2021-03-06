# 1. sqltoy的sql只能写在xml中吗?
* 不是的，sql可以直接写在代码中也可以通过@ListSql 和@PageSql两个注解完成(但一般很少用注解)。
* sql参数的名字是sqlOrNamedSql 表示可以直接是sql也可以是xml中定义的sql id。(是所有场景都是这个规则，下面的loadBySql只是一个说明范例)
```java
/**
 * @todo 通过sql获取单条记录
 * @param sqlOrNamedSql 直接代码中写的sql或者xml中定义的sql id
 * @param paramsNamed
 * @param paramsValue
 * @param voClass
 * @return
 */
public <T> T loadBySql(final String sqlOrNamedSql, final String[] paramsNamed, final Object[] paramsValue,
		final Class<T> voClass);
```
# 2. sqltoy的查询结果只能是VO list吗?
* 不是的,所有的查询Class<T> voClass 这里分1、voClass 常规的vo类型; 2、null 表示ArrayList;3、HashMap.class\LinkedHashMap.class
```
/**
 * @TODO 查询集合
 * @param <T>
 * @param sql
 * @param paramsNamed
 * @param paramsValue
 * @param voClass 分null(返回二维List)、voClass、HashMap.class、LinkedHashMap.class等
 * @return
 */
protected <T> List<T> findBySql(final String sql, final String[] paramsNamed, final Object[] paramsValue,
		final Class<T> voClass) {
	QueryExecutor query = new QueryExecutor(sql, paramsNamed, paramsValue);
	if (voClass != null) {
		query.resultType(voClass);
	}
	return (List<T>) findByQuery(query).getRows();
}
```
	
# 3. sqltoy查询#[]支持嵌套吗?
* sqltoy 动态操作#[]是支持嵌套的且是无限层嵌套,可以#[and t.status=:status  #[ and t.xxx=:xxx]]

# 4. sqltoy-orm可以不用写sql完成crud吗？
* orm的概念其实就是基于对象完成对数据库的操作，sqltoy-orm提供了基于对象的数据库操作，类似于hibernate jpa！

# 5. 如何开始crud？
* 请参照sqltoy-showcase/tools/quickvo 先通过数据库产生POJO，然后参照showcase下面/src/test/java下面的CrudCaseServiceTest！

# 6. quickvo支持yml配置文件吗？
* 不支持,不用纠结这个环节，quickvo只是一个工具，如果项目使用的是yml则可以给quickvo单独一个properties配置文件，或者直接通过quickvo.xml 中的property来定义，如下
```xml
<property name="jdbc.connection.url">
  <![CDATA[
	jdbc:mysql://192.168.56.109:3306/sqltoy?useUnicode=true&characterEncoding=utf-8&serverTimezone=GMT%2B8&useSSL=false
	]]>
</property>
<property name="jdbc.connection.driver_class" value="com.mysql.cj.jdbc.Driver"/>
```
# 7. quickvo提示没有匹配到表生成vo
* 请检查catalog 或schema 配置是否正确，且大小写是否正确！oracle、sqlServer、postgresql用schema,mysql\DB2则需配置:catalog！
具体可以了解jdbc的conn.getMetaData().getTables(catalog,schemaPattern,tablePattern,types) 方法规范

# 8. 为什么SqlToyLazyDao save或update操作数据库未发生改变？
* 这个属于事务配置文件，lazyDao应该在service层调用，service层上应该配置事务，事务配置可以注解模式，也可以通过aop 在方法规则上进行控制，注解配置范例可以参照SqlToyCRUDServiceImpl类
* 如果在Controller层调用SqlToyCRUDService则会自动完成数据库保存，因为SqlToyCRUDService方法上已经加了事务注解。
```java
  @Transactional
  public Object save(Serializable entity) {
	return sqlToyLazyDao.save(entity);
  }
```
# 9. 如何批量执行一个自定义的sql?
* sqltoy 提供了batchUpdate方法，可以将sql写在xml中，比如merge into 
```java
/**
 * @todo 批量执行sql修改或删除操作(返回updateCount)
 * @param sqlOrNamedSql
 * @param dataSet
 * @param reflectPropertyHandler 反调函数(一般不需要)
 * @param autoCommit
 */
protected Long batchUpdate(final String sqlOrNamedSql, final List dataSet,
		final ReflectPropertyHandler reflectPropertyHandler, final Boolean autoCommit) {
	//例如sql 为:merge into table  update set xxx=:param
	//dataSet可以是VO List,可以根据属性自动映射到:param
	return batchUpdate(sqlOrNamedSql, dataSet, sqlToyContext.getBatchSize(), reflectPropertyHandler, null,
			autoCommit, this.getDataSource(null));
}

```

# 10. 如何根据参数执行一个修改类sql?
* sqltoy提供了executeSql方法，跟执行查询类似,sql可以写在代码中也可以写在xml文件中
```java
/**
 * @todo 执行无返回结果的SQL(返回updateCount)
 * @param sqlOrNamedSql
 * @param paramsNamed
 * @param paramsValue
 */
protected Long executeSql(final String sqlOrNamedSql, final String[] paramsNamed, final Object[] paramsValue) {
	return executeSql(sqlOrNamedSql, paramsNamed, paramsValue, false, this.getDataSource(null));
}
```

# 11. 为什么我手写VO和表映射无法执行
* 请使用quickvo生成跟数据库的映射关系，sqltoy从来不鼓励手工写跟数据库映射，也没有必要去尝试！
* 是不是因为复杂所以不鼓励? 不是，是因为从来就没有手工写过，再简单也记不住!

# 12. 多数据源如何实现事务管理
* 请引入Jta进行事务管理，Atomikos+ spring Jta,多数据源同时要支持读和写本身就是一个特有的场景，建议可以针对此进行服务化
* 参照地址: https://www.cnblogs.com/wangyong/p/4174326.html
* sqltoy 一写多读可以按照常规模式进行配置，无需使用Jta。
