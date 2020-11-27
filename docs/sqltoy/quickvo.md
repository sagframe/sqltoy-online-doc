# quickvo的作用
> quickvo 是用于通过连接数据库读取表结构信息生产POJO对象的工具。

# 使用说明
* 1.请参见trunk下面sqltoy-showcase 项目
* 2.进入tools\quickvo 目录,创建libs目录，将sagacity-quickvo-xx.jar 和数据库驱动放于其中。
* 3.编写quickvo.xml 配置相关数据库和POJO任务信息

```xml
<?xml version="1.0" encoding="UTF-8"?>
<quickvo xmlns="http://www.sagframe.com/schema/quickvo"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.sagframe.com/schema/quickvo http://www.sagframe.com/schema/sqltoy/quickvo.xsd">
	<property file="../../src/main/resources/application.properties" />
	<property name="project.version" value="1.0.0" />
	<property name="project.name" value="sqltoy-showcase" />
	<property name="project.package" value="com.sagframe.sqltoy" />
	<property name="include.schema" value="false" />
	<!-- set method 是否支持返回对象自身(默认是true),即: public VO  setName(String name){this.name=name;return this;} -->
	<property name="field.support.linked.set" value="true" />
	<!-- oracle支持schema,mysql\DB2:catalog -->
	<datasource name="sqltoy" url="${jdbc.connection.url}"
		driver="${jdbc.connection.driver_class}" catalog="${jdbc.connection.catalog}"
		username="${jdbc.connection.username}"	password="${jdbc.connection.password}" />
	<tasks dist="../../src/main/java" encoding="UTF-8">
		<!-- include 是基于正则表达式进行表名匹配的 -->	
		<task active="true" author="zhongxuchen" include="^SQLTOY_\w+" datasource="sqltoy">
		    <!-- name后面的VO可根据情况设定，也可直接为#{subName} -->
		    <vo package="${project.package}.showcase.vo" substr="Sqltoy" name="#{subName}VO" />
		</task>
	</tasks>
	<!-- 主键策略配置:
	    identity类型的会自动产生主键策略，其他场景sqltoy根据主键类型和长度自动分配相应的策略方式. 
		strategy分:sequence\assign\generator 三种策略：
			sequence 需要指定数据库对应的sequence名称。
			assign   为手工赋值，
			generator为指定具体产生策略,目前分:default:22位长度的主键\nanotime:26位纳秒形式\snowflake雪花算法\uuid\redis
    -->
	<primary-key>
		<table name="SQLTOY_\w+|SYS_\w+" strategy="generator" generator="default" />
		<!--<table name="xxxTABLE" strategy="sequence" sequence="SEQ_XXXX"/> -->
		<!--<table name="sys_staff_info" strategy="generator" generator="snowflake"/> -->
		<!--<table name="sys_staff_info" strategy="generator" generator="redis"/>  -->
	</primary-key>

	<!-- 基于redis产生有规则的业务主键 -->
	<business-primary-key>
		<!-- 1位购销标志_2位设备分类代码_6位日期_3位流水 (如果当天超过3位会自动扩展到4位) -->
		<table name="SQLTOY_DEVICE_ORDER_INFO" column="ORDER_ID" generator="redis"
			signature="${psType}@case(${deviceType},PC,PC,NET,NT,OFFICE,OF,SOFTWARE,SF,OT)@day(yyMMdd)"
			related-columns="psType,deviceType" length="12" />
	</business-primary-key>
	
	<!-- 主子表的级联关系 update-cascade:delete 表示对存量数据进行删除操作,也可以写成:ENABLED=0(sql片段,置状态为无效) -->
	<!--orderBy 定义加载子表时的排序方式 -->
	<cascade>
		<table name="SQLTOY_DICT_DETAIL" update-cascade="delete" load="STATUS=1" orderBy=""/>
	</cascade>

	<!-- 数据类型对应关系，native-types表示特定数据库返回的字段类型; jdbc-type：表示对应jdbc标准的类型(见:java.sql.Types), 
		主要用于vo @Column注解中，设置其类型,方便操作数据库插入或修改时设置类型;java-type:表示对应java对象的属性类型 
		import-types: 可以自行定义需要引入的类型，多个用逗号分隔
	-->
	<type-mapping>
		<!-- 保留1个范例,一般无需配置 -->
		<sql-type native-types="NUMBER,DECIMAL,NUMERIC"	precision="1..8" scale="0" jdbc-type="INTEGER" java-type="Integer" />
		<!-- 泛型注意xml转义符号，jdbc-type 可以直接填数字，这里java-type="List<StaffInfoVO>" -->
		<sql-type native-types="json" jdbc-type="1021"
			java-type="List&lt;StaffInfoVO&gt;" import-types="com.sqltoy.quickstart.vo.StaffInfoVO"/>
	</type-mapping>
</quickvo>
```
* 4.主键策略说明，实现类包名:org.sagacity.sqltoy.plugins.id.impl

策略名称  | 长度    |类型| 对应实现类   |是否有序|策略说明 
-------   | ------- |---|------       |-----|----
default   | 22|全数字 |DefaultIdGenerator|有序|13位当前毫秒+6位纳秒+3位主机ID(IPv4或v6后3位)
nanotime  | 26| 全数字 |NanoTimeIdGenerator|有序|yyMMddHHmmssSSS+6位纳秒+2位(线程Id+随机数)+3位主机ID
snowflake | 16| 全数字 |SnowflakeIdGenerator|有序|16位雪花算法
UUID | 32| 字母数字混合 |UUIDGenerator| 无序|32位通用UUID
redis | 自定义| 自定义 |RedisIdGenerator|有序|利用redis进行分组key的计数，然后key+计数值，如6位日期200315和3位计数值012,组合成200315012


* 5.编写quickvo.bat脚本
```xml
java -cp ./libs/* org.sagacity.quickvo.QuickVOStart quickvo.xml
```
* 6.执行quickvo.bat 则会自动生成VO对象并分别放于设置的包路径下面

** sqltoy 生成vo分成2部分:AbstractVO 和 VO,AbstractVO 不允许手工改动保持跟数据库一致，VO则可以自行扩展属性，比如:员工表需要展示机构名称

* AbstractStaffInfoVO 跟数据库完全保持一致

```java
package com.sagframe.sqltoy.showcase.vo.base;

import java.io.Serializable;
import org.sagacity.sqltoy.config.annotation.Entity;
import org.sagacity.sqltoy.config.annotation.Id;
import org.sagacity.sqltoy.config.annotation.Column;
import java.time.LocalDate;
import java.time.LocalDateTime;

/**
 * @project sqltoy-showcase
 * @version 1.0.0
 * Table: sqltoy_staff_info,Remark:员工信息表   
 */
@Entity(tableName="sqltoy_staff_info",pk_constraint="PRIMARY")
public abstract class AbstractStaffInfoVO implements Serializable,
	java.lang.Cloneable {

	/**
	 * 根据包路径+类名称计算生成
	 */
	private static final long serialVersionUID = 6555146524698728551L;
	
	/**
	 * 员工ID
	 */
	@Id(strategy="generator",generator="org.sagacity.sqltoy.plugin.id.DefaultIdGenerator")
	@Column(name="STAFF_ID",length=22L,type=java.sql.Types.VARCHAR,nullable=false)
	protected String staffId;
	
	/**
	 * 工号
	 */
	@Column(name="STAFF_CODE",length=22L,type=java.sql.Types.VARCHAR,nullable=false)
	protected String staffCode;
	
	/**
	 * 姓名
	 */
	@Column(name="STAFF_NAME",length=30L,type=java.sql.Types.VARCHAR,nullable=false)
	protected String staffName;
	
	/**
	 * 部门
	 */
	@Column(name="ORGAN_ID",length=22L,type=java.sql.Types.VARCHAR,nullable=false)
	protected String organId;
	
	// 这里只截取了部分代码，后面部分省略
}

```

* StaffInfoVO 继承抽象VO，可自行扩展属性
```java
@SqlToyEntity
public class StaffInfoVO extends AbstractStaffInfoVO {	
	/**
	 * 
	 */
	private static final long serialVersionUID = 4609820466201465046L;
	 
	/** default constructor */
	public StaffInfoVO() {
		super();
	}
	
	//自定义属性机构名称
	/**
	 * 员工所在机构名称
	 */
	private String organName;
	
	/**
	 * @return the organName
	 */
	public String getOrganName() {
		return organName;
	}

	/**
	 * @param organName the organName to set
	 */
	public void setOrganName(String organName) {
		this.organName = organName;
	}

	//下面部分省略，可提供开发者扩展属性定义
}
```

# 生成对象后，我们就可以开始基于对象的增删改查了！
