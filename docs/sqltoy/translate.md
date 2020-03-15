# 什么是缓存翻译
  缓存翻译主要是利用缓存对sql查询的字段利用缓存转义的过程，从而避免数据库表关联查询。例如：
  * 原始做法是表关联查询
  ```xml
  <sql id="sqltoy_query_order_info">
	<value>
	<![CDATA[
       	select t1.staff_id,t2.staff_name 
       	from biz_order_info t1 
           	 left join sys_staff_info t2 
                 on t1.staff_id=t2.staff_id
	]]>
	</value>
  </sql>
 
  ```
  * 缓存翻译做法
  ```xml
  select staff_id,staff_id staffName from biz_order_info
  <sql id="sqltoy_query_order_info">
        <!-- 员工名称翻译 -->
	<translate cache="staffIdNameCache" columns="staffName" />
	<value>
	<![CDATA[
        select staff_id,staff_id staffName from biz_order_info
	]]>
	</value>
  </sql>
  ```
  
# 配置和启用缓存翻译
* 在src/main/resources 目录下面增加:sqltoy-translate.xml 文件，如改变文件名称则需要额外配置
* 修改application.yml 文件,设置translateConfig 属性
```yml
spring:
 sqltoy:
      # 指定缓存翻译配置文件
      translateConfig: classpath:sqltoy-translate.xml
```

# 缓存翻译配置说明
* 缓存配置中分:cache-translates 和cache-update-checkers 两部分,cache-translates负责将数据加载到缓存，cache-update-checkers则负责检查数据是否发生变化清理缓存，当下次使用缓存时会自动重新获取数据放入缓存，从而实现缓存的刷新。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<sagacity
	xmlns="http://www.sagframe.com/schema/sqltoy-translate"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.sagframe.com/schema/sqltoy-translate http://www.sagframe.com/schema/sqltoy/sqltoy-translate.xsd">
	<cache-translates>
		<!-- 基于sql直接查询的方式获取缓存 -->
		<sql-translate cache="dictKeyNameCache"	datasource="dataSource">
			<sql>
			<![CDATA[
				select t.DICT_KEY,t.DICT_NAME,t.STATUS
				from SQLTOY_DICT_DETAIL t
		        where t.DICT_TYPE=:dictType
		        order by t.SHOW_INDEX
			]]>
			</sql>
		</sql-translate>

		<!-- 员工ID和姓名的缓存 -->
		<sql-translate cache="staffIdNameCache" datasource="dataSource">
			<sql>
			<![CDATA[
				select STAFF_ID,STAFF_NAME,STATUS
				from SQLTOY_STAFF_INFO
			]]>
			</sql>
		</sql-translate>
	</cache-translates>

	<!-- 缓存刷新检测,可以提供多个基于sql、service、rest服务检测 -->
	<!-- cluster-time-deviation 集群节点时间偏差,默认为1秒 -->	
	<cache-update-checkers cluster-time-deviation="1">
		<!-- 基于sql的缓存更新检测,#not_debug# 放于注释中表示轮询检测时无需打印sql日志 -->
		<sql-checker check-frequency="15" datasource="dataSource">
			<sql><![CDATA[
			--#not_debug#--
			select distinct 'staffIdName' cacheName,null cache_type
			from SQLTOY_STAFF_INFO t1
			where t1.UPDATE_TIME >=:lastUpdateTime
			-- 数据字典key和name缓存检测
			union all 
			select distinct 'dictKeyName' cacheName,t2.DICT_TYPE cache_type
			from SQLTOY_DICT_DETAIL t2
			where t2.UPDATE_TIME >=:lastUpdateTime
			]]></sql>
		</sql-checker>
	</cache-update-checkers>
</sagacity>

```

* cache-translates 属性包含:

属性名称| 是否必填 | 默认值| 说明
-------|--------|-----|----
disk-store-path|非必填|null| 缓存持久化到磁盘的文件存储路径,一般不需要设置
default-heap| 非必填|10000| 一级内存堆的大小(条)，性能最高，单位:EntryUnit.ENTRIES 
default-off-heap| 非必填|0| 二级内存堆的大小(依旧在内存中,但有一个持久化序列化操作影响一定性能)，单位:MB
default-disk-size| 非必填|0| 三级持久化到硬盘存储上的大小(效率最低)，单位:MB

* cache-translates 缓存的数据结构为:key、name、扩展属性1、扩展属性2模式，一般第一列为key,第二列为通过key转化的名称，例如：

企业ID| 企业名称 | 企业全称| 企业MDM码|状态
-------|--------|-----|----|---
S001| 农业银行 |中国农业银行股份有限公司|xxxx|1

* 使用的时候通过cache-indexs来设置取第几列,默认第一列
```xml
<translate cache="enterpriceCache" columns="enterpriseFullName" cache-indexs="2"/>
```

* cache-translates下面包含四种获取数据的方式
> sql-translate: 直接通过sql数据库查询模式加载

属性名称|是否必选| 说明
-------|--------|-----
cache  | 必填  | 定义缓存的名称
datasource| 非必填|默认就是当前sqltoy选用的数据库,多数据源情况根据实际指定
sql| 非必填| 这里sql是指对应*.sql.xml 文件中定义的一个sql语句对应的id,如不指定sql对的id则通过<![CDATA[]]>模式直接提供sql
heap|非必填| 针对具体缓存指定一级堆栈大小,参见default-heap介绍
off-heap|非必填|针对具体缓存指定二级堆栈大小，单位MB
disk-size| 非必填|针对具体缓存指定三级持久化到硬盘存储上的大小(效率最低)，单位:MB

> service-translate: 通过spring获取具体的bean方法来获取缓存,返回结果为二维度List

属性名称|是否必选| 说明
-------|--------|-----
cache  | 必填  | 定义缓存的名称
service| 必填| 指定具体的service类型或名称,类型则例如:com.xxx.DictService
method | 必填| 对应service的方法,sqltoy会传递一个cacheType参数,用于类似于分类查询，如无分类则传null
heap|非必填| 针对具体缓存指定一级堆栈大小,参见default-heap介绍
off-heap|非必填|针对具体缓存指定二级堆栈大小，单位MB
disk-size| 非必填|针对具体缓存指定三级持久化到硬盘存储上的大小(效率最低)，单位:MB

> rest-translate: 通过http rest请求获取缓存,返回结果为二维度List

属性名称|是否必选| 说明
-------|--------|-----
cache  | 必填  | 定义缓存的名称
url| 必填| rest对应的url请求,如:http://xxxx/common/getDictByType
username | 非必填| 如需要身份认证提供认证信息
password | 非必填| 如需要身份认证提供认证信息
heap|非必填| 针对具体缓存指定一级堆栈大小,参见default-heap介绍
off-heap|非必填|针对具体缓存指定二级堆栈大小，单位MB
disk-size| 非必填|针对具体缓存指定三级持久化到硬盘存储上的大小(效率最低)，单位:MB

* cache-update-checkers 缓存更新检测,下面可以设置多种类型和多个缓存更新检测
> 每次检测sqltoy传递的参数为:lastUpdateTime 缓存最后检测的时间
> 缓存变更检测器也分:sql-checker\service-checker\rest-checker,返回数据结构
  如果存在缓存分类则精确清理某个缓存下面某一类的数据,如无分类则清楚整个缓存,如员工表发生变更则清除staffIdNameCache信息,下次调用则全部重新获取。
  
缓存名称           |缓存分类
-------           |--------
dictKeyNameCache  | postType  
staffIdNameCache  | null






  
