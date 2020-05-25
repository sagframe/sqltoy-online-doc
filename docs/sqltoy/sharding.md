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
# 2.对象操作分库分表
* 涉及增加、修改、删除操作的分库需要用到分布式事务管理,参见:https://github.com/chenrenfei/sqltoy-showcase/tree/master/trunk/sqltoy-sharding 
* 使用jta进行事务管理
* 在对象上进行注解,sharding配置文件参见:src/java/resources/spring-sqltoy-sharding.xml

```java
package com.sagframe.sqltoy.showcase.vo;

import java.time.LocalDateTime;

import org.sagacity.sqltoy.config.annotation.Sharding;
import org.sagacity.sqltoy.config.annotation.SqlToyEntity;
import org.sagacity.sqltoy.config.annotation.Strategy;

import com.sagframe.sqltoy.showcase.vo.base.AbstractStaffInfoVO;

/**
 * @project sqltoy-oracle
 * @author zhongxuchen
 * @version 1.0.0 Table: sqltoy_staff_info,Remark:员工信息表
 */
@SqlToyEntity
@Sharding(db = @Strategy(name = "hashBalanceDBSharding", fields = { "staffId" })
//分表跟分库类似
//,table = @Strategy(name = "hashBalanceDBSharding", fields = { "staffId" })
)
public class StaffInfoVO extends AbstractStaffInfoVO {
}
```
* 进行对象保存操作
```java
@ExtendWith(SpringExtension.class)
@SpringBootTest(classes = SqlToyApplication.class)
public class CrudCaseServiceTest {
	@Autowired
	private SqlToyCRUDService sqlToyCRUDService;

	// 演示对象操作分库分表,当前策略是采用hash取模方式,保存、修改、加载都会根据取模字段值自动匹配对应数据库
	/**
	 * 创建一条员工记录
	 */
	@Test
	public void saveStaffInfo() {
		List<StaffInfoVO> staffs = new ArrayList<StaffInfoVO>();
		for (int i = 1; i < 10; i++) {
			StaffInfoVO staffInfo = new StaffInfoVO();
			staffInfo.setStaffId("S1907150" + i);
			staffInfo.setStaffCode("S1907150" + i);
			staffInfo.setStaffName("测试员工" + i);
			staffInfo.setSexType("M");
			staffInfo.setEmail("test12@aliyun.com");
			staffInfo.setEntryDate(LocalDateTime.now());
			staffInfo.setStatus(1);
			staffInfo.setOrganId("C0001");
			staffInfo.setPhoto(
					ShowCaseUtils.getBytes(ShowCaseUtils.getFileInputStream("classpath:/mock/staff_photo.jpg")));
			staffInfo.setCountry("86");
			staffs.add(staffInfo);
		}
		sqlToyCRUDService.saveAll(staffs);
	}

	@Test
	public void loadAll() {
		List<StaffInfoVO> staffs = new ArrayList<StaffInfoVO>();
		for (int i = 1; i < 10; i++) {
			StaffInfoVO staffInfo = new StaffInfoVO();
			staffInfo.setStaffId("S1907150" + i);
			staffs.add(staffInfo);
		}
		sqlToyCRUDService.loadAll(staffs);
	}
}
```
