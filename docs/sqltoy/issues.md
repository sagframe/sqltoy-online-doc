# sqltoy-orm可以不用写sql完成crud吗？
* orm的概念其实就是基于对象完成对数据库的操作，sqltoy-orm提供了基于对象的数据库操作，类似于hibernate jpa！

# 如何开始crud？
* 请参照sqltoy-showcase/tools/quickvo 先通过数据库产生POJO，然后参照showcase下面/src/test/java下面的CrudCaseServiceTest！

# quickvo支持yml配置文件吗？
* 不支持,不用纠结这个环节，quickvo只是一个工具，如果项目使用的是yml则可以给quickvo单独一个properties配置文件，或者直接通过quickvo.xml 中的property来定义，如下
```xml
<property name="jdbc.connection.url">
  <![CDATA[
	jdbc:mysql://192.168.56.109:3306/sqltoy?useUnicode=true&characterEncoding=utf-8&serverTimezone=GMT%2B8&useSSL=false
	]]>
</property>
<property name="jdbc.connection.driver_class" value="com.mysql.cj.jdbc.Driver"/>
```
# quickvo提示没有匹配到表生成vo
* 请检查catalog 或schema 配置是否正确，且大小写是否正确！oracle、sqlServer、postgresql用schema,mysql\DB2则需配置:catalog！
具体可以了解jdbc的conn.getMetaData().getTables(catalog,schemaPattern,tablePattern,types) 方法规范

# 为什么SqlToyLazyDao save或update操作数据库未发生改变？
* 这个属于事务配置文件，lazyDao应该在service层调用，service层上应该配置事务，事务配置可以注解模式，也可以通过aop 在方法规则上进行控制，注解配置范例可以参照SqlToyCRUDServiceImpl类
```java
  @Transactional
	public Object save(Serializable entity) {
		return sqlToyLazyDao.save(entity);
	}
```
