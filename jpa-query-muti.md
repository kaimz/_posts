---
title: JPA多表查询的解决办法
date: 2017-11-9 17:50:03
tags: [java,jpa]
categories: 学习笔记
---

实际业务中，多表关联查询应用实例很多，怎么使用JPA进行多表查询，可以选择不同的方法优化。
记下在JPA中多表查询的有效使用方法。
<!--more-->

### 使用关系映射
就是使用一对多，多对一，一对一这种关系进行关联映射，

一个一对多迭代Tree的例子：
```java

import javax.persistence.*;
import java.util.List;

/**
 * 根据组织取得实时轨迹Tree的业务类
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/11/9 15:21</pre>
 */
@Entity
@Table(name = "\"DEV_ORGANIZE\"")
public class OrganizeMappedEntity extends BaseEntity {
    // 名称，为了前面取出数据的key一致性，换个名称。
    @Column(name = "\"NAME\"")
    private String no;
    // 父组织外键
    @Column(name = "\"PARENTID\"")
    private String parentid;

    @OneToMany(targetEntity = OrganizeMappedEntity.class,
            mappedBy = "parentid", cascade = {CascadeType.ALL}, fetch = FetchType.EAGER)
    private List children;

    public String getNo() {
        return no;
    }

    public void setNo(String no) {
        this.no = no;
    }

    public String getParentid() {
        return parentid;
    }

    public void setParentid(String parentid) {
        this.parentid = parentid;
    }

    public List getChildren() {
        return children;
    }

    public void setChildren(List children) {
        this.children = children;
    }
}

```
平常使用这种方法最多了，因为方便，少写代码，但是平时不一定需要查询所有，而且在数据比较多的情况下，开销比较大，就得使用下面第二种方法。

### 使用JPQL多表查询
``JPQL``全称``Java Presistence Query Language``，Java持久化查询语言。和Hibernate的HQL语句差不多。

现在测试下，从A表和B表各取出一个字段吧。
创建一个业务实体DTO：
```java
import java.io.Serializable;


//学习学习
public class TestEntity implements Serializable{

	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;
        //A表字段No
	private String no;
	//B表字段tel
	private String tel;
	
	private Long num;
	
	public  TestEntity(Long num) {
		this.num = num;
	}
	
    public Long getNum() {
		return num;
	}

	public void setNum(Long num) {
		this.num = num;
	}
	public TestEntity () {
      
	}
    //通过构造函数注入值
	public TestEntity (String no, String tel) {
    	this.no = no;
    	this.tel = tel;
    }

	public static long getSerialversionuid() {
		return serialVersionUID;
	}

	public String getNo() {
		return no;
	}

	public void setNo(String no) {
		this.no = no;
	}

	public String getTel() {
		return tel;
	}

	public void setTel(String tel) {
		this.tel = tel;
	}
	//重写写，
	@Override
	public String toString() {
		return "TestEntity{" +
				"no='" + no + '\'' +
				", tel='" + tel + '\'' +
				", num=" + num +
				'}';
	}
}

```
这样可以使用业务实体类的构造函数就行绑定数据了。

Dao层查询数据库的JPQL语句：
```java

import com.devframe.database.BasePagingAndSortingRepository;
import com.devframe.entity.DeviceEntity;
import com.devframe.entity.TestEntity;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface DeviceDao extends PagingAndSortingRepository<DeviceEntity, String> {
    /**
     * 只为学习。。。
     * @return List<TestEntity>
     */
	@Query(value = "SELECT new com.devframe.entity.TestEntity(a.no, b.tel) FROM com.devframe.entity.DeviceEntity a, com.devframe.entity.OrganizeEntity b WHERE a.orgid = b.id")
	List<TestEntity> gettest();

}

```
使用join查询出两个表相关联的所有列。

单元测试：
```java

import com.devframe.dao.DeviceDao;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

import javax.annotation.Resource;

/**
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/11/9 17:40</pre>
 */
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:spring/applicationContext-base.xml"})
public class DeviceDaoTest {
    @Resource
    private DeviceDao dao;

    @Test
    public void testGettest() {
        System.out.println(dao.gettest());
    }
}

```
数据有点多
```json
 TestEntity{no='N57008', tel='null', num=null}, TestEntity{no='N33505', tel='null', num=null}, TestEntity{no='N88001', tel='null', num=null},省略...
```

### 使用Map转换
JPA 提供查询结果的转换的方法，它提供一种使用SQL查询结果的列明作为键值，查询出来的Map结果转换成`JSONArray`或者`JSONObject`，再返回给客户端就可以了。
```java
    @Override
    public JSONArray mapBySql(String sql) {
        EntityManager em = emf.createEntityManager();
        Query query = em.createNativeQuery(sql);
        // 注意这一行设置返回格式为查询结果的列名，并且大小写也是保持一致的，如果业务需要的键值即列名和数据库存储的有差别，查询SQL中重新使用 {as} 重新设置列名即可。
        query.unwrap(SQLQuery.class).setResultTransformer(Transformers.ALIAS_TO_ENTITY_MAP);
        return (JSONArray) JSONArray.toJSON(mapBySql(query.getResultList()));
    }
```
如果没有特殊要求的话也推荐使用这种，开发起来方便很多，如果不想直接返回数据，也可以将查询出来的LIst转换下，迭代取出MAP里的键值对，就是实体类的属性值，进行处理。
需要注意的是，在Hibernate3.2版本上才有这个方法，具体JPA哪个版本没仔细查。 

