# sql查询框架默认提供了SqlToyLazyDao，让开发者可以在service中直接调用，无需自己写dao，sqltoy原则上不提倡开发者写dao
# 针对查询主要提供以下功能
* loadBySql 通过sql查询提取一条记录
```java
/**
 * @todo 通过sql获取单条记录
 * @param sqlOrNamedSql
 * @param paramsNamed
 * @param paramsValue
 * @param voClass
 * @return
 */
public <T> T loadBySql(final String sqlOrNamedSql, final String[] paramsNamed, final Object[] paramsValue,
		final Class<T> voClass);

/**
 * @todo 通过对象实体传参数,框架结合sql中的参数名称来映射对象属性取值
 * @param sqlOrNamedSql
 * @param entity
 * @return
 */
public <T extends Serializable> T loadBySql(final String sqlOrNamedSql, final T entity);
  
//代码范例
  
/**
 * 根据对象加载数据
 */
@Test
public void loadByEntity() {
	OrganInfoVO parentOrgan = sqlToyLazyDao.load(new OrganInfoVO("100008"));
	System.out.print(JSON.toJSONString(parentOrgan));
}

/**
 * 普通sql加载对象,最后一个参数可以是null(返回二维List)，也可以是HashMap返回List<Map>
 */
@Test
public void loadBySql() {
	List<OrganInfoVO> subOrgans = sqlToyLazyDao.findBySql("sqltoy_treeTable_search", new String[] { "nodeRoute" },
			new Object[] { ",100008," }, OrganInfoVO.class);
	System.out.print(JSON.toJSONString(subOrgans));
}
```

* getSingleValue 根据sql查询获取单一数值
```java
  /**
	 * @TODO 获取查询结果的第一条、第一列的值，一般用select max(x) from 等
	 * @param sqlOrNamedSql
	 * @param paramsNamed
	 * @param paramsValue
	 * @return
	 */
	public Object getSingleValue(final String sqlOrNamedSql, final String[] paramsNamed, final Object[] paramsValue);
  
```
* findBySql 通过sql查询返回一个List集合
* findPageBySql 通过sql查询返回一个分页模型(rows\pageNo\pageSize\recordCount\totalPage)
* findTopBySql 通过topSize返回前多少条记录，topSize>1 则取固定记录，topSize<1 则按比例提取记录
* getRandomResult 通过randomSize 提取随机记录
* getCount 获取查询结果的总记录数量
* isUnique 查询当前表中指定的值是否唯一
* updateFetch 查询记录并锁定再通过反调修改数据库的值，并返回修改后的结果,一次交互完成查询修改操作，场景用于:秒杀、库存台账、资金台账这种事务性极强的环节

