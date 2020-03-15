# 什么是缓存翻译
  缓存翻译主要是利用缓存对sql查询的字段利用缓存转义的过程，从而避免数据库表关联查询。例如：
  * 原始做法是表关联查询
  ```xml
  <sql id="sqltoy_query_order_info">
		<value>
			<![CDATA[
			 select t1.staff_id,t2.staff_name 
      from biz_order_info t1 left join sys_staff_info t2 on t1.staff_id=t2.staff_id
				]]>
		</value>
	</sql>
 
  ```
  * 缓存翻译做法
  ```xml
  select staff_id,staff_id staffName from biz_order_info
  <sql id="sqltoy_query_order_info">
    <!-- 员工名称翻译 -->
		<translate cache="staffIdNameCache"	columns="staffName" />
		<value>
			<![CDATA[
			 select staff_id,staff_id staffName from biz_order_info
				]]>
		</value>
	</sql>
  ```
  
# 配置和启用缓存翻译
* 在src/main/resources 目录下面增加:sqltoy-translate.xml 文件，如改变文件名称则需要额外配置
```yml
spring:
 sqltoy:
      # 指定缓存翻译配置文件
      translateConfig: classpath:sqltoy-translate.xml
```
  
  
