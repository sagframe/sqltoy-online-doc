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
	<cache-update-checkers>
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

  
