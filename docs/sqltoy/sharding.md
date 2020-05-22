# 1.查询分库分表
* 定义分库分表策略，参见sqltoy-showcase范例下resources/spring/spring-sqltoy-sharding.xml 配置
```xml
<!-- 所有分库分表策略开发者可以自行实现接口进行拓展 -->
<!-- 按照周期范围进行分表查询 -->
<bean id="historyTableStrategy"
	class="org.sagacity.sqltoy.plugins.sharding.impl.DefaultShardingStrategy"
	init-method="initialize">
	<!-- 多少天内查询实时表,可以用逗号分隔,如:value="360,35,1" -->
	<property name="days" value="14" />
	<property name="dateParams"
		value="createTime,beginDate,bizDate,beginTime,bizTime" />
	<!-- 实时表和历史表对照 -->
	<property name="tableNamesMap">
		<map>
			<!-- value可用逗号分隔,跟days对应 ,如:value="a,b,c" -->
			<entry key="SQLTOY_TRANS_INFO_15D"
				value="SQLTOY_TRANS_INFO_HIS" />
		</map>
	</property>
</bean>

<!-- 按照权重进行分库查询分流策略 -->
<bean id="weightBalanceDBStrategy"
	class="org.sagacity.sqltoy.plugins.sharding.impl.DefaultShardingStrategy"
	init-method="initialize">
	<!-- 不同数据库的分配权重 -->
	<property name="dataSourceWeight">
		<map>
			<entry key="dataSource" value="70" />
			<entry key="sharding1" value="30" />
		</map>
	</property>
	<!-- 数据库有效性检测时间间隔秒数,小于等于0表示不自动检测数据库 -->
	<property name="checkSeconds" value="180" />
</bean>

<!-- 按照hash取模进行分库和分表策略 -->
<bean id="hashBalanceDBSharding"
	class="org.sagacity.sqltoy.plugins.sharding.impl.HashShardingStrategy"
	init-method="initialize">
	<!-- 根据hash取模分库 -->
	<property name="dataSourceMap">
		<map>
			<entry key="0" value="dataSource" />
			<entry key="1" value="sharding1" />
			<entry key="2" value="sharding2" />
		</map>
	</property>
</bean>

```
* 查询使用范例,参见sql配置上的sharding-datasource 和sharding-table
```xml
<!-- 演示分库 -->
<sql id="sqltoy_db_sharding_case">
	<!-- 根据userId进行hash取模决定访问具体的数据库 -->	
	<sharding-datasource
		strategy="hashBalanceDBSharding" params="userId" />
	<value>
		<![CDATA[
		select * from sqltoy_user_log t 
		-- userId 作为分库关键字段属于必备条件
		where t.user_id=:userId 
		#[and t.log_date>=:beginDate]
		#[and t.log_date<=:endDate]
		]]>
	</value>
</sql>

<!-- 演示分表 -->
<sql id="sqltoy_15d_table_sharding_case">
	<!--  分表策略根据日期条件决定实际访问具体的表名 -->	
	<sharding-table tables="sqltoy_trans_info_15d"
		strategy="historyTableStrategy" params="beginDate" />
	<value>
		<![CDATA[
		select * from sqltoy_trans_info_15d t 
		where t.trans_date>=:beginDate
		#[and t.trans_date<=:endDate]
		]]>
	</value>
</sql>
```
