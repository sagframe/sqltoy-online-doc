# 配置，请参见:trunk/sqltoy-showcase/src/java/main/resources/spring/spring-sqltoy.xml
```xml
<bean id="sqlToyContext" class="org.sagacity.sqltoy.SqlToyContext"
		init-method="initialize" destroy-method="destroy">
	<!-- 指定sql.xml 文件的路径实现目录的递归查找，可以用逗号或分号指定多个路径 -->
	<property name="sqlResourcesDir"
		value="classpath:com/sagframe/sqltoy/showcase" />
	<!-- 非必须属性:跨数据库函数自动替换(非必须项),适用于跨数据库软件产品,如mysql开发，oracle部署 -->
	<property name="functionConverts" value="default" />
	<property name="unifyFieldsHandler">
		<bean class="com.sagframe.sqltoy.plugins.SqlToyUnifyFieldsHandler" />
	</property>
	<!-- 缓存翻译管理器,非必须属性 -->
	<property name="translateConfig" value="classpath:sqltoy-translate.xml" />
	<!-- 非必须属性:集成elasticsearch,可以配置多个地址 -->
	<property name="elasticEndpoints">
		<list>
			<bean class="org.sagacity.sqltoy.config.model.ElasticEndpoint">
				<constructor-arg value="${es.default.url}" />
				<constructor-arg value="${es.version}" />
				<property name="id" value="default" />
				<!-- 6.3.x+版本支持xpack sql查询 <property name="nativeSql" value="true" /> -->
				<property name="nativeSql" value="false" />
				<property name="username" value="${es.username}" />
				<property name="password" value="${es.password}" />
			</bean>
		</list>
	</property>
</bean>
```

# 查询编写，使用eql模式，elastic支持sql和json模式,通过mode=sql 来区分是否使用sql
* 参见:src/java/main/com/sagframe/sqltoy/showcase 目录下的sqltoy-showcase.sql.xml 文件
* elastic只支持单集合查询
* 缓存翻译、数据旋转等用法跟普通sql一致
```xml
<eql id="es_find_company"
	fields="company_id,company_name,company_type" mode="sql">
	<value>
	<![CDATA[
	select * from cc_company_info where company_type='1' limit 10
	]]>
	</value>
</eql>

<eql id="es_find_company_page"
	fields="company_id,company_name,company_type" mode="sql">
	<value>
	<![CDATA[
	select * from cc_company_info where company_type='1' 
	]]>
	</value>
</eql>

<eql id="es_find_company_page_count"
	fields="company_id,company_name,company_type" mode="sql">
	<value>
	<![CDATA[
	select count(1) count from cc_company_info where company_type='1' 
	]]>
	</value>
</eql>

<!-- 基于elasticsearch json rest原生查询,当存在_source 提供了字段时fields属性可以不用填写 -->
<eql id="sys_elastic_test_json" fields="" index="cc_company_info">
	<!-- 如果需要依然可以使用translate 缓存翻译,column 对应_source 中定义的字段 -->
	<!-- <translate cache="" columns=""/> -->
	<value>
<![CDATA[
	{
		    "_source": [
			"company_id",
				"company_name",
				"company_type"
		    ], 
		    "query": {
			"bool":{
				"filter":[
					<#>{"terms":{"company_type":@(:companyTypes)}}</#>
				]
			}
		    }
		}
]]>
</value>
</eql>
```

# java调用,参见test 目录下的ElasticCaseServiceTest，通过sqlToyLazyDao.elastic()调用
```java
	@Autowired
	private SqlToyLazyDao sqlToyLazyDao;

	/**
	 * 演示普通的查询
	 */
	@Test
	public void testSqlSearch() {
		// elasticsearch-sql https://github.com/NLPchina/elasticsearch-sql
		String sql = "es_find_company";
		List<CompanyInfoVO> result = (List<CompanyInfoVO>) sqlToyLazyDao.elastic().sql(sql)
				.resultType(CompanyInfoVO.class).find();
		for (CompanyInfoVO company : result) {
			System.err.println(JSON.toJSONString(company));
		}
	}

	/**
	 * 演示分页查询，基于sql分页请使用elasticsearch-sql插件
	 */
	@Test
	public void testSqlFindPage() {
		// elasticsearch-sql https://github.com/NLPchina/elasticsearch-sql
		String sql = "es_find_company_page";
		PaginationModel pageModel = new PaginationModel();
		PaginationModel result = (PaginationModel) sqlToyLazyDao.elastic().sql(sql).resultType(CompanyInfoVO.class)
				.findPage(pageModel);
		System.err.println("resultCount=" + result.getRecordCount());
		for (CompanyInfoVO company : (List<CompanyInfoVO>) result.getRows()) {
			System.err.println(JSON.toJSONString(company));
		}
	}

	@Test
	public void testJsonSearch() {
		String sql = "sys_elastic_test_json";
		String[] paramNames = { "companyTypes" };
		Object[] paramValues = { new Object[] { "1", "2" } };

		List<CompanyInfoVO> result = (List<CompanyInfoVO>) sqlToyLazyDao.elastic().sql(sql).names(paramNames)
				.values(paramValues).resultType(CompanyInfoVO.class).find();
		for (CompanyInfoVO company : result) {
			System.err.println(JSON.toJSONString(company));
		}
	}

	@Test
	public void testJsonFindPage() {
		String sql = "sys_elastic_test_json";
		String[] paramNames = { "companyTypes" };
		Object[] paramValues = { new Object[] { "1", "2" } };
		PaginationModel pageModel = new PaginationModel();
		PaginationModel result = (PaginationModel) sqlToyLazyDao.elastic().sql(sql).names(paramNames)
				.values(paramValues).resultType(CompanyInfoVO.class).findPage(pageModel);
		System.err.println("resultCount=" + result.getRecordCount());
		for (CompanyInfoVO company : (List<CompanyInfoVO>) result.getRows()) {
			System.err.println(JSON.toJSONString(company));
		}
	}
```
