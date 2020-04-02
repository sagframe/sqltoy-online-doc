# 文档阅读导航

> 欢迎使用和交流sqltoy-orm，请点击左侧导航栏开始了解sqltoy!

# QQ技术交流群:531812227

# 详细功能说明请参见:
  * sagacity-sqltoy/docs/睿智平台SqlToy4.10使用手册.doc

# 1. 感受sqltoy之美--缓存翻译、缓存条件检索

* 缓存翻译和缓存检索化繁为简---查询订单表(简化为单商品订单便于演示)

订单号|客户ID|商品ID|下单日期|商品数量|商品价格|订单金额|订单状态|业务员ID|部门
------|------|-----|-----|-----|-----|----|---|----|----
S0001|C10001|101|2020-03-10|10|3000|30000|02|1001|N002


* 要求查询日期在2020年1月10日至3月20日、客户名称中含<<星云科技>>字符的全部订单信息，要求显示商品名称、客户名称、业务员姓名、部门名称、订单状态中文

* 往常的做法:硬功夫硬碰硬系列

> 关联客户表做like

> 关联商品表查询品名

> 关联员工信息表显示员工名字

> 关联机构表显示机构名称

> 关联数据字典翻译状态

* 你是这么做的吗?看一下sqltoy怎么做吧！是不是变成了单表查询，效率毫无疑问秒杀多表关联无数倍!
```xml
<sql id="order_showcase">
	<!-- 通过缓存对最终结果代码进行翻译，显示名称 -->
	<translate cache="organIdName" columns="ORGAN_NAME" />
	<translate cache="staffIdName" columns="STAFF_NAME" />
	<translate cache="goodsIdName" columns="GOODS_NAME" />
	<translate cache="customIdName" columns="CUSTOM_NAME" />
	<translate cache="dictKeyName" cache-type="ORDER_STATUS" columns="STATUS_NAME" />
	<filters>
		<!-- 将查询参数customName通过缓存进行类似like检索获取匹配的customId数组作为查询条件 -->
		<cache-arg cache-name="customIdName" param="customName"	alias-name="customIds" />
	</filters>
	<value>
	<![CDATA[
	select
		ORDER_ID ,
		TOTAL_QUANTITY,
		TOTAL_AMT,
		ORGAN_ID ,
		ORGAN_ID ORGAN_NAME,-- 缓存翻译
		STAFF_ID ,
		STAFF_ID STAFF_NAME,
		SIGN_TIME,
		CUSTOM_ID,
		CUSTOM_ID CUSTOM_NAME,
		GOODS_ID ,
		GOODS_ID GOODS_NAME,
		STATUS,
		STATUS STATUS_NAME
	from od_order_info t1
	where #[SIGN_TIME>=:beginTime]
	      #[and SIGN_TIME <=:endTime]
	      -- 这里就是缓存条件检索
	      #[and CUSTOM_ID in (:customIds)]
	]]>
	</value>
</sql>
```
# 2. 感受sqltoy之美--数据旋转
* 水果销售记录表

品类|销售月份|销售笔数|销售数量(吨)|销售金额(万元)
----|-------|-------|----------|------------
苹果|2019年5月|12 | 2000|2400
苹果|2019年4月|11 | 1900|2600
苹果|2019年3月|13 | 2000|2500
香蕉|2019年5月|10 | 2000|2000
香蕉|2019年4月|12 | 2400|2700
香蕉|2019年3月|13 | 2300|2700

* sqltoy 实现行转列(category-columns 支持多列,即多维度分组如:
> 月份下面:5月(上旬、中旬、下旬)、4月(上旬、中旬、下旬)

> 电子产品例如: 手机(华为、三星、小米)、笔记本(苹果、联想、戴尔))

```xml
<!-- 列转行 -->
<sql id="pivot_case">
	<value>
	<![CDATA[
	select t.fruit_name,t.order_month,t.sale_count,t.sale_quantity,t.total_amt 
	from sqltoy_fruit_order t
	order by t.fruit_name ,t.order_month
	]]>
	</value>
	<!-- 行转列,将order_month作为分类横向标题，从sale_count列到total_amt 三个指标旋转成行 -->
	<pivot start-column="sale_count" end-column="total_amt"
		group-columns="fruit_name" category-columns="order_month" />
</sql>
```
* 效果:简单一个pivot代替了复杂的sql，而且不用考虑不同数据库的实现差异

<table>
<thead>
	<tr>
	<th rowspan="2">品类</th>
	<th colspan="3">2019年3月</th>
	<th colspan="3">2019年4月</th>
	<th colspan="3">2019年5月</th>
	</tr>
	<tr>
	    <th>笔数</th><th>数量</th><th>总金额</th>
	    <th>笔数</th><th>数量</th><th>总金额</th>
	    <th>笔数</th><th>数量</th><th>总金额</th>
	</tr>
	</thead>
	<tbody>
	<tr>
		<td>香蕉</td>
		<td>13</td>
		<td>2300</td>
		<td>2700</td>
		<td>12</td>
		<td>2400</td>
		<td>2700</td>
		<td>10</td>
		<td>2000</td>
		<td>2000</td>
	</tr>
		<tr>
		<td>苹果</td>
		<td>13</td>
		<td>2000</td>
		<td>2500</td>
		<td>11</td>
		<td>1900</td>
		<td>2600</td>
		<td>12</td>
		<td>2000</td>
		<td>2400</td>
	</tr>
	</tbody>
</table>

# 3. 感受sqltoy之美--分组汇总
# 4. 感受sqltoy之美--环比计算











