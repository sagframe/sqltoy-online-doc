# 版本号4.12.2(GA版本) 日期2020.5.21
1、增加代码中直接写sql查询时自动根据情况补齐select c1,c2,.. from table where 
```java
findBySql("where xx=:param and xx1=:parm1", {"param","param1"}, {value1,value2},
			StaffInfoVO.class)

自动补齐where 前面的select col1,col2... from STAFF_INFO
```
2、优化quickvo，剔除对log4j的依赖改用jdk自带log，大幅减小jar的大小
3、增加三个单表查询、修改、删除方法，更加简化单表操作，便于内部逻辑快捷处理

```java
public <T> List<T> findEntity(Class<T> resultType, EntityQuery entityQuery);

public Long updateByQuery(Class entityClass, EntityUpdate entityUpdate);

public Long deleteByQuery(Class entityClass, EntityQuery entityQuery);

```

用法：
```java

/**
 * findEntity 模式,简化sql编写模式,面向接口服务层提供快捷数据查询和处理
 * 1、通过where指定条件
 * 2、支持lock
 * 3、支持order by (order by 在接口服务 层意义不大)
 * 4、自动将对象属性映射成表字段
 */
@Test
public void findEntity() {
    //条件利用sqltoy特有的#[]充当动态条件判断,#[]是支持嵌套的
   String conditions = "#[staffName like ?] #[ and status=?]";
   List<StaffInfoVO> staffVOs = sqlToyLazyDao.findEntity(StaffInfoVO.class,			 
               EntityQuery.create().where(conditions).lock(LockMode.UPGRADE)
                      .orderBy("staffName").orderByDesc("createTime").values("陈", 1));
    System.err.println(JSON.toJSONString(staffVOs));
}
```

# 版本号4.11.9 日期2020.5.8
1、支持保留字处理，对象操作自动增加保留字符号，跨数据库sql自动适配

* 首先尽量避免使用保留字
* sqltoy支持保留字处理主要考虑一些已有项目

2、重新编写修复StringUtil 分隔符号切割函数splitExcludeSymMark( 在sqltoy简单场景下不受影响)
3、增加缓存数据获取为空的日志提醒,，给开发更多信息
4、查询返回结果支持List<Object[]>,同时支持resultType 直接给Map.class等接口（之前必须是HashMap等实现类）
5、quickvo支持yml格式的配置文件
6、增强sql执行输出

# 保留字支持
```xml
<bean id="sqlToyContext" name="sqlToyContext"  class="org.sagacity.sqltoy.SqlToyContext" 
init-method="initialize" destroy-method="destroy">
<!-- 项目中使用到的保留字定义，多个保留字则用逗号分隔 -->
<property name="reservedWords" value="maxvalue,minvalue"/>
<!--  其他配置项目此处省略 -->
</bean>
```
```xml
# yml模式
spring:
    sqltoy:
        reservedWords: maxvalue,minvalue
```

# 注意事项
* 保留字分对象操作和自定义sql两个部分，对象操作框架自动完成保留字的处理
* 自定义sql中的保留字需要根据当前数据库增加保留字符号，当作为产品用于不同数据库时框架会自动适配
```sql
-- sqlserver
select t.[maxvalue],t.name from table t 
-- mysql 
select t.`maxvalue`,t.name from table t 
```

# 版本号4.10.5(GA版本) 日期2020.3.31
* 1.缓存翻译对应的缓存更新增加了增量更新机制
* 2.增加了环比计算功能，同时优化了unpivot 列转行配置和实现策略
* 3.部分代码优化

# 版本号4.10.3(GA版本) 日期2020.3.17
* 1.优化ehcache缓存配置策略,设置off-heap默认为零,并提供全局默认值设置
* 2.优化sql加载顺序策略,classes 下面的sql优先于jar包中的，从而便于程序发版和本地调试
* 3.修复注解式sql调用参数合规性验证缺陷

# 版本号4.9.10 日期2020.3.7
* 1.正式启用jdk自身xml解析代替dom4j
* 2.增加了SqlToyCRUDService参数合法性验证和提示
* 3.第一个发版到中央仓库的生产可用版本(4.9.9版本因替换dom4j产生了一处bug)

# 版本号4.9.8 日期2020.2.26
* 1.完整优化sqltoy spring boot starter模式的集成，并简化配置

# 版本号4.9.5 日期2020.2.3
* 1.增加clickhouse的集成
* 2.修复beanutils类中引入LocalDate 错误(非java.time.LocalDate)
* 3.优化数据库方言配置策略,便于今后更高版本的支持扩展


# 版本号4.9.3 日期2020.1.16
* 1.调整日志集成由log4j2 改为slf4j,便于开发者选择
* 2.增强跨数据库函数解析能力，排除带转义符的单引号双引号对解析过程的影响
* 3.增强数据库cte 公共表表达式的支持
