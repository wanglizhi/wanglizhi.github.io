---
layout:     post
title:      "iBatis vs Hibernate"
subtitle:   ""
date:       2016-07-26 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - iBatis
    - Hibernate
---

#### 对比1

#### 序言

       最近一直用mybatis做开发，以前用过[hibernate](http://lib.csdn.net/base/17),能感受到一些它们在使用上的区别，不过总想抽出时间来好好比较比较弄弄清楚它们各自的优劣，以便更好进行选择和深入的了解。

       网上也看了很多资料，结合自己的使用体会，粗率地概括和总结了一下，以供大家参考。

#### 二、具体运用上的不同

## 1、所需的jar包

Mybatis：只需要3个（mybatis-3.1.1.jar，mybatis-3.1.1-javadoc.jar，mybatis-3.1.1-sources.jar）      

Hibernate:根据功能不同大概需要十几个

## 2、映射关系

 Mybatis：实体类与sql之间的映射

Hibernate：实体类与[数据库](http://lib.csdn.net/base/14)之间隐射

## 3、配置文件

Student：

属性:int  id,String name,String password;

方法配置:

 getStudentByName；   //通过name查找

getStudentById       //通过id查找

insertStudent        //添加Student 

updateStudent        //更改Student

deleteStudent        //通过id删除Student

deleteStudentById   //通过name伤处Student

selectStudentmohu   //通过name模糊查询

### Mybatis：

总配置文件：mybatisConfig.xml

<configuration>

<typeAliases>

<typeAlias alias="Student" type="com.niit.model.Student"/>

</typeAliases>

<environments default="development">

<environment id="development">

<transactionManager type="JDBC"/>

<dataSource type="POOLED">

<property name="driver" value="com.[MySQL](http://lib.csdn.net/base/14).jdbc.Driver"/>

<property name="url" value="jdbc:mysql://127.0.0.1:3306/mybatis"/>

<property name="username" value="root"/>

<property name="password" value="root"/>

</dataSource>

</environment>

</environments>

<mappers>

<mapper resource="com/niit/model/StudentMap.xml"/>

</mappers>

</configuration>

 

l 实体类映射文件：StudentMap.xml（一个或多个）

```xml
<mapper namespace="com.niit.model.StudentMap">

<select id="getStudentById" resultType="Student" parameterType="int">

select * from student where id=#{id}

</select>

<insert id="insertStudent" parameterType="Student">

insert into student(id, name, password) value(#{id}, #{name}, #{password})

</insert>

</mapper> 

```



### Hibernate：

l 总配置文件：hibernate.cfg.xml

```xml
 <hibernate-configuration>

  <session-factory>

  <property name="connection.username">root</property>

  <property name="connection.url">jdbc:mysql://127.0.0.1:3306/sample</property>

  <property name="dialect">org.hibernate.dialect.MySQLDialect </property>

  <property name="connection.password">123</property>

  <property name="connection.driver_class">com.mysql.jdbc.Driver</property>

  <property name="hibernate.show_sql">True</property>

  <mapping resource="com/niit/model/Student.hbm.xml" />

  </session-factory>

  </hibernate-configuration>
```



 

l 实体类配置文件：Student.hbm.xml（一个或多个）

```xml
<hibernate-mapping package="com.niit.model.">

    <class name="Student" table="student">

       <id name="id" column="id" type="int"><generator class="identity"/> </id>

       property name="name" type="[Java](http://lib.csdn.net/base/17).lang.String"  

           <column name="name" length="20" not-null="true" />

       </property>      

       <property name="password" type="java.lang.String"> 

           <column name="password" length="20" not-null="true" />

       </property>  

   </class>

</hibernate-mapping>
```



## 4、基本用法（增删改查模糊）

### Mybatis：

    @Test

l select by name 

public void test() throws IOException

{

String resource = "mybatisConfig.xml";

Reader reader = Resources.getResourceAsReader(resource);  

SqlSessionFactory sessionFactory = new SqlSessionFactoryBuilder().build(reader);  

SqlSession session = sessionFactory.openSession();  

        session.selectOne("com.niit.model.StudentMap.getStudentByName","b");

}

l select by id 

 session.selectOne("com.niit.model.StudentMap.getStudentById",2);        

                 

l insert  

          session.insert("com.niit.model.StudentMap.insertStudent", student);          

l //update

           session.insert("com.niit.model.StudentMap.updateStudent", student);

                       

l //delete by name

   session.insert("com.niit.model.StudentMap.deleteStudent", "wl");

                  

l //delete by id

   session.insert("com.niit.model.StudentMap.deleteStudentById", 3);

          

l //select muhu(模糊查询)

   session.selectList("com.niit.model.StudentMap.selectStudentmohu", "b");

### Hibernate：

l //select by id 

Configuration cfg = new Configuration().configure(); 
    SessionFactory sf = cfg.buildSessionFactory();  
    Session session = sf.openSession(); 

Session.get(Student.class,id); 

l //select by name 

session.createQuery("from Student as s where s.name =’a’").list();

l //insert  

session.save(Student); 

l //update  

Session.update(Student) ;

 

l //delete by name

Session.delete (Student) ;

 

l //delete by id

User user = new User(); 

User user = new User(); 
    user.setId(1);

    session.delete(user); 

l //select muhu(模糊查询)

     session.createQuery("from Student as s where s.name like '%"+str+"%'").list();

## 5、与Spring的整合

### Mybatis：

### 配置数据源文件

  <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource">

  <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"></property>

  <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"></property>
        <property 

             name="url" value="jdbc:mysql://127.0.0.1:3306/[spring](http://lib.csdn.net/base/17)?useUnicode=true&characterEncoding=UTF-8>

        </property>

        </property>
        <property name="username" value="root"></property>
        <property name="password" value="1234"></property>
        <property name="maxActive" value="100"></property>
        <property name="maxIdle" value="30"></property>
        <property name="maxWait" value="500"></property>
        <property name="defaultAutoCommit" value="true"></property>
 </bean>

 

### 配置sqlsessionfactory（将数据源注入）

 

<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">

<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="configLocation" value="classpath:MyBatis-Configuration.xml"></property>

<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="configLocation" value="classpath:MyBatis-Configuration.xml"></property>
        <property name="dataSource" ref="dataSource" />

<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="configLocation" value="classpath:MyBatis-Configuration.xml"></property>
        <property name="dataSource" ref="dataSource" />
    </bean>

 

### 在Dao实现层中通过spring Ioc 愉快使用SqlSessionFactory

    <bean id="userDao" class="org.mybatis.spring.mapper.MapperFactoryBean">
        <property name="mapperInterface" value="com.mybatis.UserDao"></property>
        <property name="sqlSessionFactory" ref="sqlSessionFactory"></property>
    </bean>
spring整合mybatis需要的jar包 

### Hibernate：

### 配置数据源文件

      <bean

class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">

    <property name="locations">

    <value>classpath:jdbc.properties</value>

   </property>

 </bean>

 

<bean id="dataSource" destroy-method="close"

class="org.apache.commons.dbcp.BasicDataSource">

<property name="driverClassName" value="${jdbc.driverClassName}" />

<property name="url" value="${jdbc.url}" />

<property name="username" value="${jdbc.username}" />

<property name="password" value="${jdbc.password}" />

</bean>

### 配置sessionfactory（将数据源注入）

  <bean id="sf"

class="org.springframework.orm.hibernate3.annotation.AnnotationSessionFactoryBean">

<property name="dataSource" ref="dataSource" />

<property name="packagesToScan">

<list>

<value>com.niit.model</value></list>

</property>

<property name="hibernateProperties">

<props>

<prop key="hibernate.dialect">org.hibernate.dialect.MySQLDialect</prop>

<prop key="hibernate.show_sql">true</prop>

</props>

</property>

</bean>

### 配置hibernatetemplete（将SessionFactory注入）

<bean id="hibernateTemplate" class="org.springframework.orm.hibernate3.HibernateTemplate">

<property name="sessionFactory" ref="sf"></property>

</bean>

## 6、注解支持

mybatis:

启用注解并注入testMapper

      1.@Repository("testBaseDAO")  

2. @Autowired  

public void setTestMapper(@Qualifier("testMapper") TestMapper testMapper) 

{  

        this.testMapper = testMapper;   

    }  

 

@SelectProvider(type = TestSqlProvider.class, method = "getSql")或@Select("select * from ....")（SelectBuilder/SqlBuilder）

@InsertProvider(type = TestSqlProvider.class, method = "insertSql") 

@DeleteProvider(type = TestSqlProvider.class, method = "deleteSql") 

@Options(flushCache = true, timeout = 20000) 

@UpdateProvider(type = TestSqlProvider.class, method = "updateSql") 

@Param("id") 

@Result(id = true, property = "id", column = "test_id")

### Hibernate：

比较基础不解释

# 三、各种“效果”上的不同(10点)

1. Hibernate是全自动ORM框架，而Mybatis是半自动的。hibernate完全可以通过对象关系模型实现对数据库的操作，拥有完整的JavaBean对象与数据库的映射结构来自动生成sql。而mybatis仅有基本的字段映射，对象数据以及对象实际关系仍然需要通过手写sql来实现和管理。

2. hibernate数据库移植性远大于mybatis。 hibernate通过它强大的映射结构和hql语言，大大降低了对象与数据库（oracle、mysql等）的耦合性，而mybatis由于需要手写sql，因此与数据库的耦合性直接取决于[程序](http://www.xuebuyuan.com/)员写sql的方法，如果sql不具通用性而用了很多某数据库特性的sql语句的话，移植性也会随之降低很多，成本很高。

3. hibernate拥有完整的日志系统，mybatis则欠缺一些。hibernate日志系统非常健全，涉及广泛，包括：sql记录、关系异常、优化警告、缓存提示、脏数据警告等；而mybatis则除了基本记录功能外，功能薄弱很多。

1. 缓存方面都可以使用第三方缓存，但是Hibernate的二级缓存配置在SessionFactory生成的配置文件中进行详细配置，然后再在具体的表-对象映射中配置是那种缓存，而Mybatis的二级缓存配置都是在每个具体的表-对象映射中进行详细配置，这样针对不同的表可以自定义不同的缓存机制。并且Mybatis可以在命名空间中共享相同的缓存配置和实例，通过Cache-ref来实现。

4．Mybatis非常简单易学，hibernate相对较复杂，门槛较高。 
5．二者都是比较优秀的开源产品 
6．当系统属于二次开发,无法对数据库结构做到控制和修改,那Mybatis的灵活性将比hibernate更适合 
7．系统数据处理量巨大，性能要求极为苛刻，这往往意味着我们必须通过经过高度优化的sql语句（或存储过程）才能达到系统性能设计指标，在这种情况下Mybatis会有更好的可控性和表现，可以进行细粒度的优化。 
8．Mybatis需要手写sql语句，也可以生成一部分，hibernate则基本上可以自动生成，偶尔会写一些hql。同样的需求,Mybatis的工作量比hibernate要大很多。类似的，如果涉及到数据库字段的修改，hibernate修改的地方很少，而Mybatis要把那些sql mapping的地方一一修改。 
9．以数据库字段一一对应映射得到的po和hibernte这种对象化映射得到的po是截然不同的，本质区别在于Mybatis这种po是扁平化的，不像hibernate映射的po是可以表达立体的对象继承，聚合等等关系的，这将会直接影响到你的整个软件系统的设计思路。 
10．hibernate现在已经是主流o/r mapping框架，从文档的丰富性，产品的完善性，版本的开发速度都要强于Mybatis。

# 四、总结

mybatis：小巧、方便、高效、简单、直接、半自动

hibernate：强大、方便、高效、复杂、绕弯子、全自动



## Hibernate与Mybatis对比

### 1. 简介

Hibernate：Hibernate是当前最流行的ORM框架之一，对JDBC提供了较为完整的封装。Hibernate的O/R Mapping实现了POJO 和[数据库](http://lib.csdn.net/base/14)表之间的映射，以及SQL的自动生成和执行。

Mybatis：Mybatis同样也是非常流行的ORM框架，主要着力点在于 POJO 与 SQL 之间的映射关系。然后通过映射配置文件，将SQL所需的参数，以及返回的结果字段映射到指定 POJO 。相对Hibernate“O/R”而言，Mybatis 是一种“Sql Mapping”的ORM实现。

### 2. 开发速度

1. 难易度

   Hibernate的真正掌握要比Mybatis困难，Hibernate比mybatis更加重量级一些。

   Mybatis框架相对简单很容易上手，但也相对简陋些。

2. 开发工作量

   Mybatis需要我们手动编写SQL语句，回归最原始的方式，所以可以按需求指定查询的字段，提高程序的查询效率。

   Hibernate也可以自己写SQL语句来指定需要查询的字段，但这样破坏了Hibernate封装以及简洁性。

### 3. 数据库移植性

Mybatis由于所有SQL都是依赖数据库书写的，所以扩展性，迁移性比较差。

Hibernate与数据库具体的关联都在XML中，所以HQL对具体是用什么数据库并不是很关心。

### 4. 缓存机制对比

1. 相同点

   Hibernate和Mybatis的二级缓存除了采用系统默认的缓存机制外，都可以通过实现你自己的缓存或为其他第三方缓存方案，创建适配器来完全覆盖缓存行为。

2. 不同点

   Hibernate的二级缓存配置在SessionFactory生成的配置文件中进行详细配置，然后再在具体的表-对象映射中配置是那种缓存。

   MyBatis的二级缓存配置都是在每个具体的表-对象映射中进行详细配置，这样针对不同的表可以自定义不同的缓存机制。并且Mybatis可以在命名空间中共享相同的缓存配置和实例，通过Cache-ref来实现。

3. 两者比较

   因为Hibernate对查询对象有着良好的管理机制，用户无需关心SQL。所以在使用二级缓存时如果出现脏数据，系统会报出错误并提示。而MyBatis在这一方面，使用二级缓存时需要特别小心。如果不能完全确定数据更新操作的波及范围，避免Cache的盲目使用。否则，脏数据的出现会给系统的正常运行带来很大的隐患。

### 5. 两者对比总结

#### 两者相同点

- Hibernate与MyBatis都可以是通过SessionFactoryBuider由XML配置文件生成SessionFactory，然后由SessionFactory 生成Session，最后由Session来开启执行事务和SQL语句。其中SessionFactoryBuider，SessionFactory，Session的生命周期都是差不多的。如下图所示：

![](http://img.blog.csdn.net/20150430002033414)

- Hibernate和MyBatis都支持JDBC和JTA事务处理。

#### Hibernate优势

- Hibernate的DAO层开发比MyBatis简单，Mybatis需要维护SQL和结果映射。
- Hibernate对对象的维护和缓存要比MyBatis好，对增删改查的对象的维护要方便。
- Hibernate数据库移植性很好，MyBatis的数据库移植性不好，不同的数据库需要写不同SQL。
- Hibernate有更好的二级缓存机制，可以使用第三方缓存。MyBatis本身提供的缓存机制不佳。

#### Mybatis优势

- MyBatis可以进行更为细致的SQL优化，可以减少查询字段。
- MyBatis容易掌握，而Hibernate门槛较高。



参考：

[Mybatis与Hibernate的详细对比](http://blog.csdn.net/jiuqiyuliang/article/details/45378065)

