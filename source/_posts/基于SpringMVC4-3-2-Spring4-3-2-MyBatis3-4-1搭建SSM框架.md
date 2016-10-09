---
title: 基于SpringMVC4.3.2+Spring4.3.2+MyBatis3.4.1搭建SSM框架
date: 2016-09-28 17:07:08
tags:
  - Java Web
  - 服务端
  - SpringMVC
  - MyBatis
  - SSM
categories: 服务端开发
---

终于到了框架搭建的最后一步，实现我们的终极目标SpringMVC+Spring+MyBatis的SSM框架，这篇文章也是基于之前搭建的SpringMVC+Spring+Hibernate框架演变过来的，所以没看过之前几篇文章的同学请乘传送带。

[基于struts2.5.2+hibernate5.2.2+spring4.3.2搭建SSH框架](http://zwgeek.com/2016/09/03/%E5%9F%BA%E4%BA%8Estruts2-5-2-hibernate5-2-2-spring4-3-2%E6%90%AD%E5%BB%BASSH%E6%A1%86%E6%9E%B6v2/)

[搭建SpringMVC+Spring4.3.2+Hibernate5.2.2框架](http://zwgeek.com/2016/09/27/%E6%90%AD%E5%BB%BASpringMVC-Spring4-3-2-Hibernate5-2-2%E6%A1%86%E6%9E%B6/)


目录

1. [删掉Hibernate相关Jar包](#删掉Hibernate相关Jar包)
2. [加入MyBatis的Jar包](#加入MyBatis的Jar包)
3. [加入MyBatis Spring支持包](#加入MyBatis Spring支持包)
4. [配置Mybatis](#配置Mybatis)
5. [配置数据库映射](#配置数据库映射)
6. [修改Mybatis配置文件](#修改Mybatis配置文件)
7. [修改DAO类](#修改DAO类)


在之前搭建SpringMVC+Spring+Hibernate的基础上，我们替换Hibernate至Mybatis，其实很简单了，所以这篇文章也很短，其他关于Spring，DAO设计模式的介绍都在之前文章中说过了。另外至于为什么最后选择使用Mybatis而不是Hibernate的原因，也在之前说过了。所以看这篇文章之前还是要看下之前两篇文章的。好，接下来我们开始替换工作。

### 删掉Hibernate相关Jar包
首先删掉Hibernate相关的jar包，在这里我删掉了所有以Hibernate开头的jar包

*注意：同时要删掉spring-orm的jar包，因为这个包依赖hibernate那边的包，那边的包删掉后这个还在的话会报错*

### 加入MyBatis的Jar包
然后我们引入Mybatis需要的包


### 加入MyBatis Spring支持包
这个包要单独从MyBatis官网下载，注意每个版本支持的Mybatis和Spring版本不一样，官网也有说明，这里因为我们MyBatis和Spring都是用的最新版本，所以mybatis-spring要最新的1.3.1。

![这里写图片描述](http://img.blog.csdn.net/20160928174741789)

如果版本不对的话，会报getTimeOut的异常，如果遇到这个异常，只要检查mybatis-spring这个jar包的版本号就可以了。

### 配置Mybatis
其实Mybatis和Hibernate总体的理念是差不多的，包括POJO，DAO的设计等等，不同的是他们对数据库的映射，及操作方式。抱歉这里我也是入门级的，无法评论孰优孰劣，具体关于Mybatis和Hibernate的差异可以自行百度，然后等我用一段时间后，有什么心得也会补充进来。

所以需要改的地方其实不多，除了配置文件就是几个数据库映射，下面一一为大家讲解。

### 配置数据库映射

Mybatis对数据库的映射是可以写成xml文件的，当然还有另外一种实现是用接口，这里只说一下xml形式的方法。

值得注意的是Mybatis不需要映射数据表，我简单说一下原因，大家试着理解一下。Hibernate需要映射数据表，是因为Hibernate将对某张表的增删改查操作都用HQL实现了一遍，这样有一个好处就是开发者不用关心sql语言，就算数据库换了也没关心，Hibernate会做一个HQL->SQL的转换。缺点就是对SQL的优化很难。而Mybatis与Hibernate在这个点上完全不同，Mybatis不关心表结构，你需要自己配置增删改查的SQL语句，至于优点和缺点也恰好和Hibernate相反。下面我们看下Mybatis的配置文件。


```xml
mapping/userMapper
<?xml version="1.0" encoding="UTF-8" ?>
  <!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!-- 为这个mapper指定一个唯一的namespace，namespace的值习惯上设置成包名+sql映射文件名，这样就能够保证namespace的值是唯一的 
	例如namespace="me.gacl.mapping.userMapper"就是me.gacl.mapping(包名)+userMapper(userMapper.xml文件去除后缀) -->
<mapper namespace="com.helloworld.mapping.userMapper">
	<!-- 在select标签中编写查询的SQL语句， 设置select标签的id属性为getUser，id属性值必须是唯一的，不能够重复 使用parameterType属性指明查询时使用的参数类型，resultType属性指明查询返回的结果集类型 
		resultType="me.gacl.domain.User"就表示将查询结果封装成一个User类的对象返回 User类就是users表所对应的实体类 -->
	<!-- 根据id查询得到一个user对象 -->
	<select id="getUser" parameterType="int" resultType="com.helloworld.pojo.User">
		select *
		from user where userId=#{id}
	</select>
</mapper>
```

大家看到在配置文件中我们配置了一条查询语句，配置了输入参数id，和返回参数。当然配置返回参数的类型后，Mybatis可以自动将返回的语句映射成一个JAVA类。

### 修改Mybatis配置文件

在前面框架配置的基础上，我们删掉spring-common.xml，因为前面也说过这个common配置是对datasource和session等关于Hibernate的配置，创建spring-mybatis.xml，将mybatis配置文件写入这个文件中。这也是前面说到配置文件分开的优势，你看，如果要修改持久层框架配置，只需要简单替换就可以了。

配置如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:mvc="http://www.springframework.org/schema/mvc"
	xsi:schemaLocation="http://www.springframework.org/schema/beans  
                        http://www.springframework.org/schema/beans/spring-beans-3.1.xsd  
                        http://www.springframework.org/schema/context  
                        http://www.springframework.org/schema/context/spring-context-3.1.xsd  
                        http://www.springframework.org/schema/mvc  
                        http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd">

	<!-- 定义数据源的信息 -->
	<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource"
		destroy-method="close">
		<property name="driverClass">
			<value>com.mysql.jdbc.Driver</value>
		</property>
		<property name="jdbcUrl">
			<value>jdbc:mysql://localhost:3306/test</value>
		</property>
		<property name="user">
			<value>root</value>
		</property>
		<property name="password">
			<value>zzw</value>
		</property>
		<property name="maxPoolSize">
			<value>80</value>
		</property>
		<property name="minPoolSize">
			<value>1</value>
		</property>
		<property name="initialPoolSize">
			<value>1</value>
		</property>
		<property name="maxIdleTime">
			<value>20</value>
		</property>
	</bean>

	<!-- spring和MyBatis完美整合，不需要mybatis的配置映射文件 -->
	<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
		<property name="dataSource" ref="dataSource" />
		<!-- 自动扫描mapping.xml文件 -->
		<property name="mapperLocations" value="classpath:com/helloworld/mapping/*.xml"></property>
	</bean>

	<!-- DAO接口所在包名，Spring会自动查找其下的类 -->
	<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
		<property name="basePackage" value="com.helloworld.dao" />
		<property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"></property>
	</bean>

	<!-- (事务管理)transaction manager, use JtaTransactionManager for global tx -->
	<bean id="transactionManager"
		class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<property name="dataSource" ref="dataSource" />
	</bean>

</beans>
```


配置文件也比较容易理解，因为和Hibernate的概念都是一样的。然后只要定义了mapperLocations，Spring会自动扫描路径下的所有mapper文件，做成Bean。同样的，配置DAO所在的包名，也能自动将session注入到DAO中。

### 修改DAO类

前面DAO类继承HibernateDaoSupport是对Hibernate的支持，这里自然也要改掉了，Mybatis需要继承SqlSessionDaoSupport，如下：

```java
package com.helloworld.daoImpl;

import java.sql.Connection;

import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.support.SqlSessionDaoSupport;

import com.helloworld.dao.BaseDao;
import com.helloworld.pojo.User;

public class UserDao extends SqlSessionDaoSupport implements BaseDao{

	/**
	  * 获取相关的数据库连接
	  */
	 public Connection getConnection() {
	  return getSqlSession().getConnection();
	 }
	
	public UserDao() {
		System.out.println("UserDao IN");
	}
    
    public User getUser(){
    	String statement = "com.helloworld.mapping.userMapper.getUser";//映射sql的标识字符串
        //执行查询返回一个唯一user对象的sql
        User user = getSqlSession().selectOne(statement, 1);
    	return user;
    }

	@Override
	public void saveObject(Object obj)
	{
		// TODO Auto-generated method stub
		
	}
}
```

至于Mapper文件中配置的查询文件怎么用，这里也给了一个简单的例子，非常简单，看一下就可以了。

配置完成之后整个项目的目录结构如下

![这里写图片描述](http://img.blog.csdn.net/20160928174844008)

至此，Mybatis框架已经配置完成，可以和Hibernate一样写个Test类，看能不能正常获取到数据库中的数据，也可以将项目发布，看访问页面能不能获取的数据。

如果没什么意外，现在应该已经能够通过访问页面来访问数据库然后显示给用户了。一个WEB项目最简单的流程也就跑通了，然后就可以在这个简单流程上开发自己相应的业务了。那接下来我也要去做我的业务了，开发过程中有什么心得也会发在博客中跟大家分享，敬请期待。








