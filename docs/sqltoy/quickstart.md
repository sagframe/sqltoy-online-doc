# 请参照trunk下面sqltoy-quickstart 演示项目

* 创建一个springboot的maven项目,并建立包路径,如:com.sqltoy.quickstart
* 修改pom.xml,引入sqltoy boot starter、druid starter、数据库驱动、缓存、junit测试依赖等。

```xml
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>2.3.4.RELEASE</version>
</parent>

<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter</artifactId>
	</dependency>
	<dependency>
		<groupId>com.alibaba</groupId>
		<artifactId>druid-spring-boot-starter</artifactId>
		<version>1.2.1</version>
	</dependency>
        <!-- mysql 数据库 -->
	<dependency>
		<groupId>mysql</groupId>
		<artifactId>mysql-connector-java</artifactId>
		<version>8.0.21</version>
	</dependency>
	<dependency>
		<groupId>com.sagframe</groupId>
		<artifactId>sagacity-sqltoy-starter</artifactId>
		<version>4.16.6</version>
	</dependency>
        <!-- ehcache 用作缓存翻译 -->
	<dependency>
		<groupId>org.ehcache</groupId>
		<artifactId>ehcache</artifactId>
		<version>3.9.0</version>
	</dependency>
	<!--  用于对象输出演示用,实际项目不依赖 -->	
	<dependency>
		<groupId>com.alibaba</groupId>
		<artifactId>fastjson</artifactId>
		<version>1.2.74</version>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
        <!-- junit5  -->
	<dependency>
		<groupId>org.junit.jupiter</groupId>
		<artifactId>junit-jupiter-api</artifactId>
		<scope>test</scope>
	</dependency>
	<dependency>
		<groupId>org.junit.platform</groupId>
		<artifactId>junit-platform-launcher</artifactId>
		<scope>test</scope>
	</dependency>
</dependencies>

```

* 编写springboot main启动程序SqlToyApplication.java

```java
package com.sqltoy;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.transaction.annotation.EnableTransactionManagement;

/**
 * @author zhongxuchen
 */
@SpringBootApplication
@ComponentScan(basePackages = { "com.sqltoy.config", "com.sqltoy.quickstart" })
@EnableTransactionManagement
public class SqlToyApplication {

	/**
	 * @param args
	 */
	public static void main(String[] args) {
		SpringApplication.run(SqlToyApplication.class, args);
	}
}

```

* 在src/main/resources 下面编写application.yml

```xml
spring:
    sqltoy:
        # 这里要注意，指定sql文件的目录(是目录不是具体文件或文件匹配表达式),多个可以用逗号分隔，会自动向下寻找
        sqlResourcesDir: classpath:/com/sqltoy/quickstart
        # 缓存翻译的配置
        translateConfig: classpath:sqltoy-translate.xml
        # debug模式会打印执行sql
        debug: true
        # 提供统一字段:createBy createTime updateBy updateTime 等字段补漏性(为空时)赋值(可选配置)
        #unifyFieldsHandler: com.sqltoy.plugins.SqlToyUnifyFieldsHandler
        # sql执行超过多长时间则进行日志输出,用于监控哪些慢sql(可选配置:默认30秒)
        #printSqlTimeoutMillis: 300000
    datasource:
        name: dataSource
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        username: quickstart
        password: quickstart
        url: jdbc:mysql://192.168.56.109:3306/quickstart?useUnicode=true&serverTimezone=GMT%2B8&useSSL=false
        druid:
           initial-size: 5
           min-idle: 5
           maxActive: 20
           # 配置获取连接等待超时的时间
           maxWait: 60000
           numTestsPerEvictionRun: 3
           keepAlive: true
           # 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
           timeBetweenEvictionRunsMillis: 120000
           # 配置一个连接在池中最小生存的时间，单位是毫秒
           minEvictableIdleTimeMillis: 600000
           validationQuery: SELECT 1 FROM DUAL
           testWhileIdle: true
           testOnBorrow: true
           testOnReturn: false
           removeAbandoned: true
           removeAbandonedTimeout: 300
    
```

* 编写你的第一个sql,在com/sqltoy/quickstart创建sqltoy-quickstart.sql.xml 必须要以*.sql.xml 结尾,注意sql编写的格式,必须
要符合范例的格式:```<sqltoy><sql id=""><value><![CDATA[]]></value></sql></sqltoy>``` 模式

```xml
<?xml version="1.0" encoding="utf-8"?>
<sqltoy xmlns="http://www.sagframe.com/schema/sqltoy"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.sagframe.com/schema/sqltoy http://www.sagframe.com/schema/sqltoy/sqltoy.xsd">
	<!-- 第一个演示sql -->
	<sql id="qstart_query_staffInfo">
		<value>
			<![CDATA[
			select * from sqltoy_staff_info t 
			where #[t.staff_name like :staffName]
				  #[and t.status=:status]
			]]>
		</value>
	</sql>
</sqltoy>

```
* 写一个单元测试类：QueryDemoTest.java 执行单元测试即可打印出结果

```java
package com.sqltoy.quickstart;

import java.util.List;

import javax.annotation.Resource;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.sagacity.sqltoy.dao.SqlToyLazyDao;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit.jupiter.SpringExtension;

import com.alibaba.fastjson.JSON;
import com.sqltoy.SqlToyApplication;

@ExtendWith(SpringExtension.class)
@SpringBootTest(classes = SqlToyApplication.class)
public class QueryDemoTest {
	/**
	 * sqltoy 默认提供统一的lazyDao,正常情况下开发者无需自己写dao层
	 */
	@Resource(name = "sqlToyLazyDao")
	private SqlToyLazyDao sqlToyLazyDao;

	@Test
	public void queryStaffInfo() {
		String[] paramNames = { "staffName", "status" };
		Object[] paramValue = { "陈", 1 };
		//最后一个参数是返回类型 null 则返回普通数组(可以传VO对象、Map.class)
		List staffInfo = sqlToyLazyDao.findBySql("qstart_query_staffInfo", paramNames, paramValue, null);
		System.out.println(JSON.toJSONString(staffInfo));
	}
}

```

# 到此你已经完成了第一个简单的sql执行
