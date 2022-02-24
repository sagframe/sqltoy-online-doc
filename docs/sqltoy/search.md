# 1. sqltoy提倡只写service逻辑部分，dao通过SqlToyLazyDao完成
# 2. sqltoy的sql完整规范
```xml
<?xml version="1.0" encoding="utf-8"?>
<sqltoy xmlns="http://www.sagframe.com/schema/sqltoy" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.sagframe.com/schema/sqltoy http://www.sagframe.com/schema/sqltoy/sqltoy.xsd">
<!-- id 命名建议遵循: moduleName+functionName 模式,避免不同模块之间重复-->	
<sql id="sqltoy_sql_specs">
      <!-- filters 用来对参与查询或执行的参数值进行转化处理 -->
      <filters>
	        <!-- 最常用的为前2个eq、to-date，注意to-date的用法,比较精妙的是cache-arg、比较细思极恐的是primary  -->		
		<!-- 参数等于某个值则将参数值设置为null -->
		<eq params="organType" value="-1" />
		<!-- 将参数条件值转换为日期格式,format可以是yyyy-MM-dd这种自定义格式也可以是: 
		  first_day:月的第一天;last_day:月的最后一天,first_year_day:年的第一天,last_year_day年的最后一天,increment-time 加减时间，increment-unit默认为days -->
		<to-date params="" format="yyyyMMdd" increment-time="1" increment-unit="days"/>
	      	<!-- 场景:前端只有单日期条件时，构造一个endDate形成日期范围过滤,一般和to-data(利用increment-time 加一天) 功能结合使用 -->
	      	<clone param="beginDate" as-param="endDate"/>
		<to-number params="" data-type="decimal" />
	        <!-- 在参数的左边加% ,sqltoy默认规则是:参数里面有%符号不做处理，没有%符号则两边加% -->
	      	<l-like params="staffName"/>
	        <!-- 在参数的右边边加% -->
	        <r-like params="staffName"/>
		<!-- 通过缓存将名称用类似like模式匹配出对应的编码作为条件进行精准查询 -->
		<cache-arg param="" cache-name="" cache-type="" alias-name="" />
		<!-- 首要参数，比如页面上精准输入了订单编号，此时除特定条件外其他条件全部设置为null不参与查询
		     特定的比如:授权访问机构(避免越权访问)             -->
		<primary param="orderId" excludes="organIds" />
		<!-- 将数组转化成in 的参数条件并增加单引号 -->
		<to-in-arg params="" />
	      	<!-- 将日期格式化字符串，可以结合to-number 实现将日期转换为数字，如月份202112 -->
	      	<date-format params="" format=""/>	
		<!-- 空白转为null，一般无需配置，默认就是所有空白自动转为null -->
	        <blank params="*" excludes="staffName" />
		<!-- 参数值在某个区间则转为null -->
		<between params="" start-value="0" end-value="9999"	excludes="" />
		<!-- 将前端传过来的字符串切割成数组 -->
		<split params="staffAuthOrgs" data-type="string" split-sign="," />
		<!-- 参数小于等于某个值时转为null -->
		<lte params="" value="" />
		<!-- 参数小于某个值时转为null -->
		<lt params="" value="" />
		<!-- 参数大于等于某个值时转为null -->
		<gte params="" value="" />
		<!-- 参数大于某个值时转为null -->
		<gt params="" value="" />
		<!-- 字符替换,默认根据正则表达进行全部替换，is-first为true时只替换首个 -->
		<replace params="" regex="" value="" is-first="false" />
		<!-- 排他性参数,当某个参数是xxx值时,将其他参数设置为特定值 -->
		<exclusive param="" compare-type="eq" compare-values=""	set-params="" set-value="" />
	</filters>

	<!-- 缓存翻译,可以对例如:A,B 这种拼连的进行翻译(要指定分隔符号和再次拼装符号 split-regex="," link-sign=",")
	    uncached-template 是针对未能匹配时显示的补充,${value} 表示显示key值,可以key=[${value}未定义这种写法 -->
	<translate cache="dictCache" cache-type="POST_TYPE" columns="POST_TYPE" cache-indexs="1" uncached-template="" />
	<!-- 安全掩码:tel\姓名\地址\卡号 -->
	<!--最简单用法: <secure-mask columns="" type="tel"/> -->
	<secure-mask columns="" type="name" head-size="3" tail-size="4" mask-code="*****" mask-rate="50" />
	<!-- 分库策略 -->
	<sharding-datasource strategy="multiDataBase" />
	<!-- 分表策略 -->
	<sharding-table tables="" strategy="hisRealTable" params="" />
	<!-- 分页优化,缓存相同查询条件的分页总记录数量, alive-max:表示相同的一个sql保留100个不同条件查询 
	 alive-seconds:相同的查询条件分页总记录数保留时长(单位秒) -->
	<page-optimize alive-max="100" alive-seconds="600" />
	<!-- 日期格式化 -->
	<date-format columns="" format="yyyy-MM-dd HH:mm:ss" />
	<!-- 数字格式：包括:#,###.00(可自定义)、captial(数字转中文大写)、capital-rmb(大写金额),财务单据上经常要用到 -->
	<number-format columns="" format="capital-rmb" />
	<value>
	<![CDATA[
	-- sql 中是可以直接写注释的
	select t1.*,t2.ORGAN_NAME from 
	@fast(select * from sys_staff_info t
		  where #[t.sexType=:sexType]
			#[and t.JOIN_DATE>:beginDate]
			#[and t.STAFF_NAME like :staffName]
			-- 是否虚拟员工@if()做逻辑判断
			#[@if(:isVirtual==true||:isVirtual==0) and t.IS_VIRTUAL=1]
			) t1,sys_organ_info t2
         where t1.ORGAN_ID=t2.ORGAN_ID
	]]>	
	</value>
	<!-- count-sql(只针对分页查询有效,sqltoy分页针对计算count的sql进行了智能处理, 
	 一般不需要额外定义countsql,除极为苛刻的性能优化，sqltoy提供了极度优化的口子) -->
	<count-sql><![CDATA[]]></count-sql>
	<!-- 汇总和求平均 -->
	<summary sum-columns="" average-columns="" average-radix-sizes="2" reverse="false" sum-site="left" average-skip-null="false">
		<global sum-label="" label-column="" />
		<group sum-label="" label-column="" group-column="" />
	</summary>
	<!-- 拼接某列,mysql中等同于group_concat\oracle 中的WMSYS.WM_CONCAT功能,id-columns表示以哪列值为分组(可以多列) -->
	<link id-columns="" sign="," column="" distinct="true"/>
	<!-- 行转列 (跟unpivot互斥) -->
	<pivot category-columns="" group-columns="" start-column=""	end-column="" default-value="0" />
	<!-- 列转行 -->
	<unpivot columns-to-rows="1:xxx,2:xxxx" new-columns-labels="" />
     </sql>
</sqltoy>
```
# sqltoy的核心逻辑:规则很单一,靠filters配置逻辑来规整统一
* #[t1.type in (:types)]等同于 if(types==null) 则剔除#[] 中间的语句
* #[]是支持嵌套的,如:#[and t1.status=:status #[and t.amt>=:amt]] 当status为null这段全部不参与查询
```sql
select * from table t1
where #[t1.type in (:types)]
      #[and t1.bizDate>=:beginDate]
      #[and t1.bizDate<=:endDate]
      #[and t1.status=:status #[and t.amt>=:amt]]
```

# 常规用法说明
* 常规sql,上面规范是完整功能罗列别吓着了

```xml
<sql id="sqltoy_sql_specs">
	<!-- filters 用来对参与查询或执行的参数值进行转化处理 -->
	<filters>
		<!-- 参数等于某个值则将参数值设置为null -->
		<eq params="organType" value="-1" />
	</filters>
	<!-- 缓存翻译,可以对例如:A,B 这种拼连的进行翻译(要指定分隔符号后最后拼装符号 split-regex="," link-sign=",")
	    uncached-template 是针对未能匹配时显示的补充,${value} 表示显示key值,可以key=[${value}未定义 
		这种写法 -->
	<translate cache="dictCache" cache-type="POST_TYPE" columns="POST_TYPE"/>
	<!-- 日期格式化 -->
	<date-format columns="" format="yyyy-MM-dd HH:mm:ss" />
	<!-- 数字格式 -->
	<number-format columns="" format="capital-rmb" />
	<value>
	<![CDATA[
	-- sql 中是可以直接写注释的
	select t1.*,t2.ORGAN_NAME from 
	@fast(select * from sys_staff_info t
		  where #[t.sexType=:sexType]
			#[and t.JOIN_DATE>:beginDate]
			#[and t.STAFF_NAME like :staffName]
			-- 是否虚拟员工@if()做逻辑判断
			#[@if(:isVirtual==true||:isVirtual==0) and t.IS_VIRTUAL=1]
			) t1,sys_organ_info t2
         where t1.ORGAN_ID=t2.ORGAN_ID
	]]>	
	</value>
</sql>
```

* filter中的eq过滤是最常用的,比如页面上有一个下拉框:sexType,默认选项是:全部(value="-1"),
  也就是sexType传过来值为-1时,就不做sexType条件过滤
  
```html 
<select name="sexType">
	<option value="-1">全部</option>
	<option value="F">男性</option>
	<option value="M">女性</option>
</select>
```

# 常用功能简介
* loadBySql 通过sql查询提取一条记录

```java

// 通过sql获取单条记录
public <T> T loadBySql(final String sqlOrNamedSql, final String[] paramsNamed, final Object[] paramsValue,
		final Class<T> voClass);


// 通过对象实体传参数,框架结合sql中的参数名称来映射对象属性取值
public <T extends Serializable> T loadBySql(final String sqlOrNamedSql, final T entity);


@Autowired
private SqlToyLazyDao sqlToyLazyDao;

//根据对象加载数据
@Test
public void loadByEntity() {
   OrganInfoVO parentOrgan = sqlToyLazyDao.load(new OrganInfoVO("100008"));
   System.out.print(JSON.toJSONString(parentOrgan));
}

//普通sql加载对象,最后一个参数可以是null(返回二维List)，也可以是HashMap返回List<Map>
@Test
public void loadBySql() {
      List<OrganInfoVO> subOrgans = sqlToyLazyDao.findBySql("sqltoy_treeTable_search", new String[] { "nodeRoute" },
			new Object[] { ",100008," }, OrganInfoVO.class);
      System.out.print(JSON.toJSONString(subOrgans));
}

```

* getSingleValue 根据sql查询获取单一数值

```java

//获取查询结果的第一条、第一列的值，例如执行:select max(x) from 等
public Object getSingleValue(final String sqlOrNamedSql, final String[] paramsNamed, final Object[] paramsValue);

```

* findBySql 通过sql查询返回一个List集合

* findPageBySql 通过sql查询返回一个分页模型(rows\pageNo\pageSize\recordCount\totalPage)
* findTopBySql 通过topSize返回前多少条记录，topSize>1 则取固定记录，topSize<1 则按比例提取记录
* getRandomResult 通过randomSize 提取随机记录
* getCount 获取查询结果的总记录数量
* isUnique 查询当前表中指定的值是否唯一
* updateFetch 查询记录并锁定再通过反调修改数据库的值，并返回修改后的结果,一次交互完成查询修改操作，场景用于:秒杀、库存台账、资金台账这种事务性极强的环节

