# 认识sqltoy-orm
* 基于java语言开发的一套数据库ORM框架
* 同时拥有hibernate对象操作的简洁性、以及比mybatis(或plus)等更强大、更简洁的sql查询
* 最为贴切项目实践，提供了绝大多数用户项目过程中的最直接处理问题的方法调用，除了常用的查询和分页外，还提供如:

>1. 取随机记录、取top记录(固定数量和按比例)
>2. 锁记录修改并返回结果updateFetch
>3. 树形表结构递归查询,利用表结构设计实现无关数据库的递归查询
>4. 行列互转\无限级的分组汇总平均，极为简单的sql加短短的配置代替:晦涩冗长还自以为很牛的大sql
>5. 还有很多入骨三分的特性请后续章节关注

# 支持的数据库

| 数据库类型  |  支持版本     | 支持功能       | 说明|
|------         |----           |-----           |----      |
| oracle        | 11g/12c~19c   | 完整功能       |主流数据库|
| mysql         | 5.6/5.7/8.x   | 完整功能       |主流数据库|
| sqlserver     | 2012~2019+    | 完整功能       |主流数据库|
| postgresql    | 9.5~12.2+     | 完整功能       |主流数据库|
| db2           | 9.5~11.5+     | 完整功能       |常用数据库|
| StarRocks     | 1.x+          | 完整功能       |MPP数据库|
| clickhouse    | 18.6+         | 查询功能/插入(建议只用于查询)  |列存储分布式OLAP数据库，适合超大规模数据亚秒级查询,特性上更接近传统数据库|
| elasticsearch | 5.x+       | 查询功能       |检索引擎，分布式存储，可支持超大规模毫秒级查询,可实现类似数据库单表查询|
| sqlite        | 3.21+         | 完整功能       |常用嵌入式数据库|
| mongodb       | 4.x+          | 查询功能       |文档存储，不支持sql写法复杂易忘       |
| greenplum     | 6.x+          | 完整功能       |MPP数据库|
| sybase iq     | 15.4+         | 完整功能       |列式存储，高效查询,应用已经不广泛,不推荐使用|
