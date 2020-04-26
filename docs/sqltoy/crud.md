# CRUD最特别的是update及其forceUpdateProps策略
* 正常情况下对象修改属性为null不做修改(符合实际项目过程特征:数据变更在每个环节只变更部分属性)
* hibernate则强调先load然后update，这在分布式高并发集群模式下是有问题的，load后再update，可能别人已经改变过，而你又复写回去了。
* 因此sqltoy的策略就不会有这种情况发生。

* 请参见github trunk/sqltoy-showcase/src/test/java/com/sagframe/sqltoy/showcase目录
* CrudCaseServiceTest.java首先Autowired SqlToyCRUDService,目的在于让开发者无需为简单的对象增删改和加载再写一个service和对应的方法。
```java
@ExtendWith(SpringExtension.class)
@SpringBootTest(classes = SqlToyApplication.class)
public class CrudCaseServiceTest {
	@Autowired
	private SqlToyCRUDService sqlToyCRUDService;
  
        //方法体.....
 }
```
# 对象加载load/loadAll
```java
  @Test
  public void load() {
	StaffInfoVO staff = sqlToyCRUDService.load(new StaffInfoVO("S190715003"));
	System.err.println(JSON.toJSONString(staff));
  }

  @Test
  public void loadAll() {
	// 组织批量数据
	List<StaffInfoVO> staffs = new ArrayList<StaffInfoVO>();
	String[] ids = { "S190715001", "S190715002" };
	for (String id : ids) {
		staffs.add(new StaffInfoVO(id));
	}
	sqlToyCRUDService.loadAll(staffs);
  }
```
# 新增记录
```java
@Test
public void saveStaffInfo() {
	StaffInfoVO staffInfo = new StaffInfoVO();
	staffInfo.setStaffId("S190715003");
	staffInfo.setStaffCode("S190715003");
	staffInfo.setStaffName("测试员工3");
	staffInfo.setSexType("M");
	staffInfo.setEmail("test3@aliyun.com");
	staffInfo.setEntryDate(LocalDate.now());
	staffInfo.setStatus(1);
	staffInfo.setOrganId("C0001");
	staffInfo.setPhoto(ShowCaseUtils.getBytes(ShowCaseUtils.getFileInputStream("classpath:/mock/staff_photo.jpg")));
	staffInfo.setCountry("86");
	sqlToyCRUDService.save(staffInfo);
}

@Test
public void saveAllStaffInfo() {
	// 组织批量数据
	List<StaffInfoVO> staffs = new ArrayList<StaffInfoVO>();
	// 构造员工信息集合
	for (String[] id : dataSets) {
		staffs.add(new StaffInfoVO(id));
	}
	sqlToyCRUDService.saveAll(staffs);
	//忽视已经存在的
	//sqlToyCRUDService.saveAllIgnoreExist(staffs);
}
```
# 修改记录
* 1.sqltoy 修改操作分类:单记录修改、批量修改、级联修改
```java
public interface SqlToyCRUDService {
/**
 * @todo 非深度修改对象,属性为null不做修改
 * @param entity
 */
public Long update(Serializable entity);

/**
 * @todo 修改对象，设置强制修改的属性
 * @param entity
 * @param forceUpdateProps
 */
public Long update(Serializable entity, String[] forceUpdateProps);

/**
 * @todo 深度修改对象，属性值为null则强制修改更新数据库值为null
 * @param entity
 */
public Long updateDeeply(Serializable entity);

/**
 * @todo 批量对对象进行修改(以首条记录为基准决定哪些字段会被修改)
 * @param entities
 */
public <T extends Serializable> Long updateAll(List<T> entities);

/**
 * @todo 批量对象修改，通过forceUpdateProps指定哪些字段需要强制修改
 * @param entities
 * @param forceUpdateProps
 */
public <T extends Serializable> Long updateAll(List<T> entities, String[] forceUpdateProps);

/**
 * @todo 批量深度集合修改
 * @param entities
 */
public <T extends Serializable> Long updateAllDeeply(List<T> entities);
}
```
* 代码示例
```java 
/**
 * 修改员工记录信息
 */
// 演示sqltoy修改数据的策略
// sqltoy 默认的update
// 操作并不需要先将记录查询出来(普通hibernate则需要先取数据，然后对需要修改的地方进行重新设置值，确保其他字段值不会被覆盖)。
// sqltoy 利用各种数据库自身特性,未null的字段会被忽略掉不参与更新操作(如果需要强制更新参见下一个范例:forceUpdate)
@Test
public void updateStaffInfo() {
   StaffInfoVO staffInfo = new StaffInfoVO("S190715001");
   // 只修改所在机构,其他字段不会被影响(避免先查询后修改提升了效率,同时避免高并发场景下先查询再修改数据冲突)
   staffInfo.setOrganId("C0002");
   sqlToyCRUDService.update(staffInfo);
}

/**
 * 对员工信息特定字段进行强制修改
 */
@Test
public void forceUpdate() {
    StaffInfoVO staffInfo = new StaffInfoVO("S190715001");
    staffInfo.setOrganId("C0002");
    staffInfo.setAddress("测试地址");
    // 第二个数组参数设置需要强制修改的字端,如果该字段的值为null，数据库中的值将被null覆盖
    sqlToyCRUDService.update(staffInfo, new String[] { "address" });
}
```
# 保存或修改，分saveOrUpdate/saveOrUpdateAll
```java
public interface SqlToyCRUDService {
/**
 * @todo 单条记录保存或修改
 * @param entity 实体对象
 * @return
 */
public Long saveOrUpdate(Serializable entity);

/**
 * @todo 修改或保存单条记录
 * @param entity 实体对象
 * @param forceUpdateProps 强制修改的对象属性
 */
public Long saveOrUpdate(Serializable entity, String[] forceUpdateProps);

/**
 * @todo 批量保存或修改对象
 * @param <T>
 * @param entities 对象集合
 * @return
 */
public <T extends Serializable> Long saveOrUpdateAll(List<T> entities);

/**
 * @todo 批量保存或修改对象
 * @param <T>
 * @param entities 对象集合
 * @param forceUpdateProps 需强制修改的属性
 * @return
 */
public <T extends Serializable> Long saveOrUpdateAll(List<T> entities, String[] forceUpdateProps);
}
```
* 示例代码
```java 
@Test
public void saveOrUpdate() {
	StaffInfoVO staffInfo = new StaffInfoVO();
	staffInfo.setStaffId("S190715003");
	staffInfo.setStaffCode("S190715003");
	staffInfo.setStaffName("测试员工3");
	staffInfo.setSexType("M");
	staffInfo.setEmail("test3@aliyun.com");
	staffInfo.setEntryDate(LocalDate.now());
	staffInfo.setStatus(1);
	staffInfo.setOrganId("C0001");
	staffInfo.setPhoto(ShowCaseUtils.getBytes(ShowCaseUtils.getFileInputStream("classpath:/mock/staff_photo.jpg")));
	staffInfo.setCountry("86");
	sqlToyCRUDService.saveOrUpdate(staffInfo);
}
```
# 删除记录,分delete/deleteAll 两个方法
```java
public interface SqlToyCRUDService { 
/**
 * @todo 删除单条对象
 * @param entity
 */
public Long delete(Serializable entity);

/**
 * @todo 批量删除对象
 * @param entities
 */
public <T extends Serializable> Long deleteAll(List<T> entities);
}
```
* 示例代码
```java
@Test
public void delete() {
	sqlToyCRUDService.delete(new StaffInfoVO("S190715001"));
}

@Test
public void deleteAll() {
	// 组织批量数据
	List<StaffInfoVO> staffs = new ArrayList<StaffInfoVO>();
	String[] ids = { "S190715001", "S190715002" };
	for (String id : ids) {
		staffs.add(new StaffInfoVO(id));
	}
	sqlToyCRUDService.deleteAll(staffs);
}
```
# 级联操作，分:级联加载、级联保存、级联修改
> 级联操作的核心在于数据表存在外键关联，通过quickvo自动生成的VO上有:@OneToMany
```java
@Entity(tableName = "sqltoy_complexpk_head", pk_constraint = "PRIMARY")
public abstract class AbstractComplexpkHeadVO implements Serializable, java.lang.Cloneable {
/**
 * 主键关联子表信息
 */
@OneToMany(fields = { "transDate", "transCode" }, mappedTable = "sqltoy_complexpk_item", mappedColumns = {
		"TRANS_DATE", "TRANS_CODE" }, mappedFields = { "transDate", "transCode" })
protected List<ComplexpkItemVO> complexpkItemVOs = new ArrayList<ComplexpkItemVO>();
}
```
* 示例代码
```java
/**
 * 演示级联保存
 */
@Test
public void cascadeSave() {
	// 主表记录
	ComplexpkHeadVO head = new ComplexpkHeadVO();
	head.setTransDate(LocalDate.now());
	head.setTransCode("S0001");
	head.setTotalCnt(BigDecimal.valueOf(10));
	head.setTotalAmt(BigDecimal.valueOf(10000));

	// 子表记录1
	ComplexpkItemVO item1 = new ComplexpkItemVO();
	item1.setProductId("P01");
	item1.setPrice(BigDecimal.valueOf(1000));
	item1.setAmt(BigDecimal.valueOf(5000));
	item1.setQuantity(BigDecimal.valueOf(5));
	head.getComplexpkItemVOs().add(item1);

	// 子表记录2
	ComplexpkItemVO item2 = new ComplexpkItemVO();
	item2.setProductId("P02");
	item2.setPrice(BigDecimal.valueOf(1000));
	item2.setAmt(BigDecimal.valueOf(5000));
	item2.setQuantity(BigDecimal.valueOf(5));
	head.getComplexpkItemVOs().add(item2);

	sqlToyCRUDService.save(head);
}

/**
 * 演示级联加载
 */
@Test
public void cascadeLoad() {
     ComplexpkHeadVO head = sqlToyCRUDService.loadCascade(new ComplexpkHeadVO(LocalDate.now(), "S0001"));
     //打印级联加载的字表数据
     System.err.println(JSON.toJSONString(head.getComplexpkItemVOs()));
}
```

# sql执行
# 批量sql执行
# 存储过程调用

# 更多的CRUD可以参见SqlToyLazyDao(CRUDService面向上层不易暴露过多参数，比如不易直接传sql)
