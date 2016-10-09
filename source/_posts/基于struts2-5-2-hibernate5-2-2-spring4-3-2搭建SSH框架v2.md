---
title: 基于struts2.5.2+hibernate5.2.2+spring4.3.2搭建SSH框架
date: 2016-09-3 18:42:33
tags:
  - Java Web
  - 服务端
  - SSH
categories: 服务端开发
---

现在在学习后端框架，最后的目标是希望搭建一个基于spring mvc + mybatis + spring的框架，因为之前接触过SSH，所以想从SSH开始，慢慢演化，也巩固一下自己的知识。
之前每次搭建SSH框架都要在网上查各种资料，而且我也发现各种资料基于的SSH版本都比较老，新版本就会遇到各种各样的问题，所以基于这次的搭建流程，写一下遇到的问题和解决方法。

## Contents
- [Contents](#Contents)
- [基础需求](#基础需求)
- [配置Struts框架](#配置Struts框架)
- [搭建Hibernate框架](#搭建Hibernate框架)
- [DAO设计模型](#dao)
- [搭建Spring框架，整合Struts和Hibernate](#搭建Spring框架，整合Struts和Hibernate)

## 基础需求

### 下载 Eclipse J2EE版
J2EE版带server和maven的配置，用起来比较方便，其他也没什么区别，普通版装插件也是可以达到一样效果的

### 下载tomcat 
目前Eclipse J2EE版的server只支持tomcat 8 以下版本，我试过8.5.5也不支持，所以最好下7

### 安装mysql

具体流程可以从网上找，这个简单

创建数据库 create database test；

创建表 
```java
create table user(
 userId int auto_increment,  
 userName varchar(16) not null,  
 password varchar(16) not null,  
 gender int not null,  
 primary key(userId)  
);
```


## 配置Struts框架

### 安装struts的jar包

下载struts-2.5.2包

将包下面lib目录下的以下文件拷贝到项目的WEB-INF/lib下面，当然这里要先创建一个Dynamic Web Project，这个也简单，在Eclipse中点下一步下一步下一步就可以。

为什么是放在WEB-INF/lib下，而不放在项目的lib下，这是因为，web项目在发布后依赖包是去寻找WEB-INF目录下的各种包的。这里我后面遇到一个奇怪的问题，也加深了对这个配置的理解，具体什么问题以后再说。然后你把包放在WEB-INF/lib下的时候，eclipse会自动拷贝一份到项目的lib下，方便编程时候的依赖。

![](http://img.blog.csdn.net/20160922112818884?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### 创建web.xml
在WEB-INF下面创建web.xml，配置struts监听，这个web.xml其实就是整个web项目的入口，所有的配置都是从这里开始，再跳转的其他地方。格式如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="2.5" xmlns="http://java.sun.com/xml/ns/javaee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/javaee 
    http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd">
    <filter>
        <filter-name>struts2</filter-name>
        <filter-class>
           org.apache.struts2.dispatcher.ng.filter.StrutsPrepareAndExecuteFilter
        </filter-class>
    </filter>
 	<filter-mapping>
  		<filter-name>struts2</filter-name>
  		<url-pattern>/*</url-pattern>
  		<!--注意：千万不能写成：*.action ，如果需要：*.action应该配置在struts.xml中-->
 	</filter-mapping>
 
    <welcome-file-list>
    <welcome-file>index.jsp</welcome-file>
    </welcome-file-list>
</web-app>
```

其实熟悉j2ee的人知道，在struts之前，j2ee最基本的跳转是用servlet来做的，struts其实也要基于servlet来做，配置一个filter，name随意。然后在filter-mapping里配置满足一定条件的url请求都交给这个filter来处理，其实也就是struts来处理。这里我们配置为/*，也就是所有的请求都转发给struts处理，这是最简单的，如果需要特殊配置可以在这里再配置。

另外也要注意/* 和 /的区别，按照我个人的理解/*是所有的请求，包括/test.jsp和/test.html这种带后缀名的请求。/是不带后缀名的所有请求，像/test这样的。


### 创建struts.xml

然后所有的请求都给struts处理了，struts本身肯定还需要一个配置文件，来转发各种请求到相应的处理类，这个配置文件是struts.xml，放在src文件夹下，前面说过，web项目的配置文件都是在web-inf下面，为什么这个放在src文件夹下呢，这里就要说到一个eclipse发布映射的问题。

你项目里点右键，选属性，选Deployment Assembly，可以看到这是发包时候的映射关系，src文件夹会发布到WEB-INF/classes，而struts会默认到这个文件夹下面找配置文件。

![](http://img.blog.csdn.net/20160922122622832?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

回到正题，说一下struts的配置文件，格式如下

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE struts PUBLIC "-//Apache Software Foundation//DTD Struts Configuration 2.0//EN" "http://struts.apache.org/dtds/struts-2.0.dtd" >  
<struts>  
    <package  name ="user_curd"  extends ="struts-default"  >  
        <global-results>  
            <!--  下面定义的结果对所有的Action都有效  -->  
            <result  name ="exception"> /error.jsp </result>  
        </global-results>  
  
        <global-exception-mappings>  
            <!--  指Action抛出Exception异常时，转入名为exception的结果。  -->  
            <exception-mapping  exception ="java.lang.Exception"  result ="exception" />  
        </global-exception-mappings>  
  
        <action  name ="test"  class ="TestAction">  
        </action>  
    </package>    
</struts> 
```

配置文件很好懂，下面的action部分就是请求转发，url中对\test的请求会转发到TestAction中处理

### 创建Action类
创建Action处理类，前面也说过了，请求会转发到某个类中进行处理，很显然，我们需要定义这样的类
在src中创建相应的类

```java
package com.helloworld.test;

import java.io.PrintWriter;
import java.util.Date;

import org.apache.struts2.ServletActionContext;

import com.opensymphony.xwork2.ActionSupport;

public class TestAction extends ActionSupport
{
	private String contentType = "text/html;charset=utf-8";     
	public String execute() throws Exception
	{
		//指定输出内容类型和编码  
        ServletActionContext.getResponse().setContentType(contentType);   
        //获取输出流，然后使用  
        PrintWriter out = ServletActionContext.getResponse().getWriter();   
        try{  
            //输出文本信息  
            out.print("Hello World");  
            out.print("Time: " + (new Date()).getTime());   
            out.flush();  
            out.close();  
        }catch(Exception ex){  
            out.println(ex.toString());  
        }
		return SUCCESS;  
	}
}
```

execute方法就是处理请求的方法，具体的使用可以再查相关资料，本文只介绍搭建框架

### 发包运行
此时访问test应该会跳转到该类，然后输出信息

### 配置Struts时遇到的问题：
#### 问题1
java.lang.ClassNotFoundException: org.apache.struts2.dispatcher.ng.filter.StrutsPrepareAndExecuteFilter
最新的Struts框架处理类的包名变了，其实碰到这类问题，自己去lib中看下类所在的位置就可以，每次版本更新可能会变一些东西

![](http://img.blog.csdn.net/20160922123903227?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### 错误2
java.util.concurrent.ExecutionException: org.apache.catalina.LifecycleException: Failed to start component [StandardEngine[Catalina].StandardHost[localhost].StandardContext[/helloworld]]

这是因为lib包多了或少了，参照我前面lib库的文件，检查一下

#### 错误3
Unable to load configuration. - bean - jar:file:/Users/zzw/Documents/j2eeworkspace/.metadata/.plugins/org.eclipse.wst.server.core/tmp0/wtpwebapps/helloworld/WEB-INF/lib/struts2-gxp-plugin-2.5.2.jar!/struts-plugin.xml:8:162

和上个问题一样，这是因为引用包多了，其实不要觉得我把所有包都放进了就行了，如果包多了会做一些初始化的工作，而初始化的过程中就容易有问题

访问http://localhost:8080/helloworld/test成功

## 搭建Hibernate框架

Struts到目前为止就算成功了，接下来我们看引入Hibernate框架

### 官网下载hibernate 5.2.2
### 下载JDBC

http://www.mysql.com/products/connector/ 下载jdbc

### 配置Hibernate的Jar包
拷贝lib\required下的jar包到WEB-INFO\lib目录下，Hibernate就很好，把所有需要的包都放在了required文件夹下

### 创建hibernate.cfg.xml
创建hibernate的配置文件hibernate.cfg.xml，配置数据库连接等等，也是在src目录下，格式如下

```xml
<!DOCTYPE hibernate-configuration PUBLIC
	"-//Hibernate/Hibernate Configuration DTD 3.0//EN"
	"http://hibernate.sourceforge.net/hibernate-configuration-3.0.dtd">

<hibernate-configuration>
	<session-factory>
		<property name="hibernate.connection.driver_class">com.mysql.jdbc.Driver</property>
		<property name="hibernate.connection.url">jdbc:mysql://localhost:3306/User</property>
		<property name="hibernate.connection.username">root</property>
		<property name="hibernate.connection.password">123</property>
		<property name="hibernate.dialect">org.hibernate.dialect.MySQLDialect</property>
	</session-factory>
</hibernate-configuration>
```

这是最简单的配置，连接数据库的

### 创建实体类

我们都知道hibernate是实体-关系映射，所以要创建实体类

```java
package com.helloworld.test;

public class User {
	private int userId;

	private String userName;

	private String passWord;

	private int gender;

	public int getUserId() {
		return userId;
	}

	public void setUserId(int userId) {
		this.userId = userId;
	}

	public String getUserName() {
		return userName;
	}

	public void setUserName(String userName) {
		this.userName = userName;
	}

	public String getPassWord() {
		return passWord;
	}

	public void setPassWord(String passWord) {
		this.passWord = passWord;
	}

	public int getGender() {
		return gender;
	}

	public void setGender(int gender) {
		this.gender = gender;
	}

}
```
就是对应数据库中一个表

### 配置映射关系
明显，这个实体类和表的映射关系也需要配置
添加User.hbm.xml文件映射表结构

```xml
<?xml version="1.0"?>
<!DOCTYPE hibernate-mapping PUBLIC 
	"-//Hibernate/Hibernate Mapping DTD 3.0//EN"
	"http://hibernate.sourceforge.net/hibernate-mapping-3.0.dtd">
<hibernate-mapping>
	<class name="com.helloworld.test.User">
		<id name="userId">
			<generator class="increment" />
		</id>
		<property name="userName" />
		<property name="passWord" />
		<property name="gender" />
	</class>
</hibernate-mapping>
```

这个映射关系配置文件可以放在任何地方，因为下一步我们会在hibernate.xml配置文件中声明这个文件的位置，我目前是放在和User类一起的位置。

### 添加映射关系
按照上一步所说，我们需要把映射关系配置文件的路径配置到hibernate.cfg.xml中去，如下：要写清楚包名，位置，就mapping配置的那部分，如果有多个映射，依次添加

```xml
<!DOCTYPE hibernate-configuration PUBLIC
	"-//Hibernate/Hibernate Configuration DTD 3.0//EN"
	"http://hibernate.sourceforge.net/hibernate-configuration-3.0.dtd">

<hibernate-configuration>
	<session-factory>
		<property name="hibernate.connection.driver_class">com.mysql.jdbc.Driver</property>
		<property name="hibernate.connection.url">jdbc:mysql://localhost:3306/test</property>
		<property name="hibernate.connection.username">root</property>
		<property name="hibernate.connection.password">zzw</property>
		<property name="hibernate.dialect">org.hibernate.dialect.MySQLDialect</property>
		<property name="hibernate.show_sql">true</property>  
		<property name="hibernate.format_sql">true</property>  
		<mapping resource="com/helloworld/test/User.hbm.xml"/>
	</session-factory>
	
</hibernate-configuration>
```


### 测试运行
Hibernate不需要发包web项目，可以本地测试，写一个Test类

```java
package com.helloworld.test;

import java.util.List;

import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.cfg.Configuration;
import org.hibernate.query.Query;

public class HibernateTest {

	public static void main(String[] args) {
		//读取hibernate.cfg.xml文件  
        Configuration cfg = new Configuration().configure();  
          
        //建立SessionFactory  
        SessionFactory factory = cfg.buildSessionFactory();  
          
        //取得session  
        Session session = null;  
        try {  
            session = factory.openSession();  
            //开启事务  
            session.beginTransaction();  
            User user = new User();  
            user.setUserName("zzw"); 
            user.setPassWord("zzw");  
              
            //保存User对象  
            session.save(user);   
            String hql = "from User";  
            Query query = session.createQuery(hql);  
            List<User> roles = query.list();
            for(int i=0;i<roles.size();i++){
            	System.out.print("从数据库加载数据的用户名为"+roles.get(i).getUserName());  
            }
            //提交事务  
            session.getTransaction().commit();  
        }catch(Exception e) {  
            e.printStackTrace();  
            //回滚事务  
            session.getTransaction().rollback();  
        }finally {  
            if (session != null) {  
                if (session.isOpen()) {  
                    //关闭session  
                    session.close();  
                }  
            }  
        }  

	}

}
```
执行成功，这样的话Hibernate框架也算搭建完成了。

## DAO设计模型
提到Hibernate不得不提的是DAO设计模型，为了下一步Spring的配置更加清楚明了，这里我们也采用DAO的设计模型

### 基础概念

这里讲几个概念
POJO（Plain Ordinary Java Object）简单的Java对象，实际就是普通JavaBeans，是为了避免和EJB混淆所创造的简称。这里POJO其实就是User类
DAO (Data Access Object)是一个数据访问接口，数据访问：顾名思义就是与数据库打交道。夹在业务逻辑与数据库资源中间。

简单一点说，就是把数据库相关操作提到DAO中进行，与业务有关的逻辑放在Manager中，为了分层编程。举个例子来说，比如用户注册这个功能，用户注册的页面显示由RegisterAction负责，Action类中有Manager负责具体的业务，RegisterManager中有具体的业务方法register，Manager中有与数据库打交道的DAO类，RegisterManager中应该有UserDAO，负责所有对User表的操作，比如addUser，deleteUser等。这样说应该很容易理解吧，这是一种分层编程的思想，可以降低各个模块之间的耦合度，比如如果你想把用户注册改成管理员注册，只需要把UserDAO改成managerDAO就可以操作manager表了。就这样。DAO设计模式也是很推崇面向接口的编程，下面我用代码为大家讲解。

### DAO类
1，首先声明接口

```java
package com.helloworld.dao;

import org.hibernate.HibernateException;
import org.hibernate.Session;

public interface BaseDao {

	public void saveObject(Object obj) throws HibernateException;

	public Session getSession();

	public void setSession(Session session);
}
```

跟数据库打交道需要获取hibernate的session，所以一个简单的base接口就是几个获取session的方法


然后我们定义HibernateSessionFactory用于在各个DAO中获取Session

```java
package com.helloworld.daoImpl;

import org.hibernate.HibernateException;
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.boot.Metadata;
import org.hibernate.boot.MetadataSources;
import org.hibernate.boot.model.naming.ImplicitNamingStrategyJpaCompliantImpl;
import org.hibernate.boot.registry.StandardServiceRegistry;
import org.hibernate.boot.registry.StandardServiceRegistryBuilder;
import org.hibernate.cfg.Configuration;
import org.hibernate.service.ServiceRegistry;

public class HibernateSessionFactory {

	private static final String CFG_FILE_LOCATION = "/Hibernate.cfg.xml";

	private static final ThreadLocal<Session> threadLocal = new ThreadLocal<Session>();

	private static final Configuration cfg = new Configuration()
			.configure(CFG_FILE_LOCATION);

	private static ServiceRegistry registry;

	private static SessionFactory sessionFactory;

	public static Session currentSession() throws HibernateException {
		Session session = threadLocal.get();

		if (session == null || session.isOpen() == false) {

			if (sessionFactory == null) {
				StandardServiceRegistry standardRegistry = new StandardServiceRegistryBuilder()
						.configure().build();
				Metadata metadata = new MetadataSources(standardRegistry)
						.getMetadataBuilder()
						.applyImplicitNamingStrategy(
								ImplicitNamingStrategyJpaCompliantImpl.INSTANCE)
						.build();
				sessionFactory = metadata
						.getSessionFactoryBuilder().build();
			}

			session = sessionFactory.openSession();
			threadLocal.set(session);

		}

		return session;
	}

	public static void closeSession() throws HibernateException {
		Session session = threadLocal.get();
		threadLocal.set(null);
		if (session != null) {
			session.close();
		}
	}

}
```


下面是跟User表打交道的UserDao

```java
package com.helloworld.daoImpl;

import org.hibernate.HibernateException;
import org.hibernate.Session;

import com.helloworld.dao.BaseDao;

public class UserDao implements BaseDao{
	private Session session;  
	public UserDao(){
		session=HiberanateSessionFactory.currentSession();
       }  
    @Override  
    public Session getSession() {  
        return session;  
    }  
  
    @Override  
    public void setSession(Session session) {  
        this.session = session;  
    }  
  
    @Override  
    public void saveObject(Object obj) throws HibernateException {  
        session.save(obj);  
    }  
}
```

### 业务逻辑类
然后声明业务逻辑类UserManager，这里我只是举个最简单的例子,直接调用了DAO的getUsers方法，不要觉得没用，在日常事务中，我们需要在DAO方法前后做些处理，都是要在Manager中进行处理的。

```java
package com.helloworld.manager;

import java.util.List;

import org.hibernate.HibernateException;

import com.helloworld.dao.BaseDao;
import com.helloworld.daoImpl.UserDao;
import com.helloworld.pojo.User;

public class UserManager {
	private BaseDao dao;  
	
	public UserManager(){
		dao = new UserDao();
		System.out.println("UserManager IN");
	}
   
    public BaseDao getDao() {
		return dao;
	}

	public void setDao(BaseDao dao) {
		this.dao = dao;
	}

	public List<User> getUsers() throws HibernateException {  
    	return dao.getUsers();
    } 
}
```

这时候就可以在测试类里用manager对象进行数据库操作了。比如

```java
package com.helloworld.test;

import java.util.List;

import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.cfg.Configuration;
import org.hibernate.query.Query;

import com.helloworld.dao.BaseDao;
import com.helloworld.daoImpl.HibernateSessionFactory;
import com.helloworld.daoImpl.UserDao;
import com.helloworld.manager.UserManager;
import com.helloworld.pojo.User;

public class HibernateTest {

	public static void main(String[] args) {
		UserManager userManager=new UserManager();
        userManager.getUsers()

	}

}
```

DAO模型介绍到这。

## 搭建Spring框架，整合Struts和Hibernate

接下来用spring整合struts和hibernate

前面提到的DAO设计模式，在用到的时候new 一个DAO对象进行数据库操作，这是最简单的，但是你想想这样会浪费时间，浪费内存，因为没进行一次访问都要生成一个新的对象，其实全局都可以用一个DAO对象。Spring是干嘛的，Spring有两大特性，IoC和AoP，其中IoC中的一种方式便是依赖注入，Spring全局管理一些Bean，像Session，dao都可以是bean，然后你需要的时候就给你注入，这就是依赖注入。其他的特性可以自行百度，另外Spring其实是一套门路很深的框架，不然也不会在Struts和Hibernate都渐渐退居二线的时候，它依然坚挺在第一线。有机会我准备仔细看下Spring的实现原理，与大家分享一下。

总而言之，整个Spring的配置过程其实就是，配置bean，然后把bean配置到各个类中这样。

### 下载4.3.2release的spring
Spring官网改版后找了好久都没有找到直接下载Jar包的链接,下面汇总些网上提供的方法,亲测可用.

直接输入地址,改相应版本即可:http://repo.springsource.org/libs-release-local/org/springframework/spring/3.2.4.RELEASE/spring-framework-3.2.4.RELEASE-dist.zip

在1的方法上输入前面部分,有个树形结构可供选择:http://repo.springsource.org/libs-release-local/org/springframework/spring/

同样的,,有树形结构选择需要的包下载:http://repo.spring.io/milestone/org/springframework/

### 加入Spring的Jar包

将Spring内libs目录下包含所有的jar包（不需要复制结尾为sources和javadoc的jar包）到项目的lib目录下。

这里为了整合Struts还需要加入一个struts的包
记得加入struts-spring-plugin的jar包，不然struts无法使用spring管理的bean对象

### 创建Spring配置文件
编写Spring的配置文件applicationContext.xml。把该文件放在WEB-INF下，跟web.xml同目录。

这里我们使用C3P0来管理数据池，所以把Hibernate内lib/optional/c3p0下的c3p0-0.9.1.jar复制到lib不目下。

applicationContext的配置很复杂，所有的bean都配置在里面，如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">

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

	<!--定义Hibernate的SessionFactory -->
	<!-- SessionFactory使用的数据源为上面的数据源 -->
	<!-- 指定了Hibernate的映射文件和配置信息 -->
	<bean id="sessionFactory"
		class="org.springframework.orm.hibernate5.LocalSessionFactoryBean">
		<property name="dataSource">
			<ref local="dataSource" />
		</property>
		<property name="mappingResources">
			<list>
				<value>com/helloworld/pojo/User.hbm.xml</value>
			</list>
		</property>
		<property name="hibernateProperties">
			<props>
				<prop key="hibernate.dialect">org.hibernate.dialect.MySQLDialect</prop>
				<prop key="show_sql">true</prop>
				<prop key="hibernate.jdbc.batch_size">20</prop>
			</props>
		</property>
	</bean>

	<bean id="transactionManager"
		class="org.springframework.orm.hibernate5.HibernateTransactionManager">
		<property name="sessionFactory" ref="sessionFactory" />
	</bean>

	<bean id="userDao" class="com.helloworld.daoImpl.UserDao">
		<property name="sessionFactory">
			<ref bean="sessionFactory" />
		</property>
	</bean>

	<!--用户注册业务逻辑类 -->
	<bean id="userManager" class="com.helloworld.manager.UserManager">
		<property name="dao">
			<ref bean="userDao" />
		</property>
	</bean>

	<!-- 用户注册的Action -->
	<bean id="testAction" class="com.helloworld.action.TestAction">
		<property name="manager">
			<ref bean="userManager" />
		</property>
	</bean>

	<!-- more bean definitions go here -->

</beans>
```

从配置文件我们看出，hibernate的datasource和session的配置完全被spring接管了，所以hibernate的配置文件是可以删掉的。

### 修改BaseDao和UserDao。

在引入Spring后，需要用Spring进行统一的事务管理，数据源和sessionFactory都交给Spring去生成，因此接口类和实现类BaseDao和UserDao都需要做相应的修改。Spring提供了HibernateDaoSupport类来完成对数据的操作，因此UserDao在实现BaseDao的同时还需要继承HibernateDaoSupport类。并将先前session的操作修改成HibernateTemplate（可通过getHibernateTemplate（）方法来获得）的操作。

```java
package com.helloworld.dao;

import java.util.List;

import org.hibernate.HibernateException;
import org.hibernate.Session;

import com.helloworld.pojo.User;

public interface BaseDao {

	public void saveObject(Object obj) throws HibernateException;
	
	public List<User> getUsers() throws HibernateException;
}

```

```java
package com.helloworld.daoImpl;

import java.util.List;

import org.hibernate.HibernateException;
import org.springframework.orm.hibernate5.support.HibernateDaoSupport;

import com.helloworld.dao.BaseDao;
import com.helloworld.pojo.User;

public class UserDao extends HibernateDaoSupport implements BaseDao{
	
	public UserDao() {
		System.out.println("UserDao IN");
	}
  
    @Override  
    public void saveObject(Object obj) throws HibernateException {  
    	getHibernateTemplate().save(obj);  
    }  
    
    public List<User> getUsers() throws HibernateException{
    	List<User> users=getHibernateTemplate().loadAll(User.class);
    	return users;
    }
}

```

其实HibernateDaoSupport也没干什么大事，就是前面说的session的set get方法，既然每个DAO都需要，那spring就提出来了呗，没什么神秘的。


### 修改业务逻辑实现类

也就是Manager的类，跟DAO一样。在没有加入Spring之前，业务逻辑实现类的Session的获得，dao的实例化，以及事务的管理都是该类执行管理的。加入Spring后，这些都交给Spring去管理。该类的dao的实例化由Spring注入。

### 修改用户注册的testAction类

同样，testAction类中的userManager的实例化也由Spring注入。可以仔细理解一下上面的applicationContext的配置文件，你需要某个对象，只要把该对象配置成bean，比如下面这样

```xml
<bean id="userManager" class="com.helloworld.manager.UserManager">
		...
	</bean>
```

然后用到这个bean的类配置成

```xml
<bean id="testAction" class="com.helloworld.action.TestAction">
		<property name="manager">
			<ref bean="userManager" />
		</property>
	</bean>
```

这样这个类里名字为manager的对象就会自动被注入userManager对象。记得需要有set方法，名字需对应。

### 删除多余类

删除Hibernate的配置文件Hibernate.cfg.xml和工厂类
HibernateSesseionFactory类。他们的工作已经交给Spring去做，已经不再有用。

### 修改web.xml

加载Spring。要想启动时加载Spring的配置文件，需要在web.xml中配置对应的监听器（listenser），并指定Spring的配置文件。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="2.5" xmlns="http://java.sun.com/xml/ns/javaee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/javaee 
    http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd">
    
    <listener>  
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>  
    </listener> 
    <filter>
        <filter-name>struts2</filter-name>
        <filter-class>
           org.apache.struts2.dispatcher.filter.StrutsPrepareAndExecuteFilter
        </filter-class>
    </filter>
 	<filter-mapping>
  		<filter-name>struts2</filter-name>
  		<url-pattern>/*</url-pattern>
 	</filter-mapping>
 	
 	
 
    <welcome-file-list>
    <welcome-file>index.jsp</welcome-file>
    </welcome-file-list>
</web-app>
```


### 修改Struts的配置文件struts.xml

把原来指定的名为register的action的class由原来的路径变为applicationContext.xml文件中该bean的id名，不需要再用具体的包名+类名。

包名加类名的方式会在每次访问的时候都生成一个action对应的对象，交给spring管理后，只会在最开始的时候生成一次。如下

```xml
<action  name ="test"  class ="testAction">
```

整个项目配置之后结构如图

![](http://img.blog.csdn.net/20160922132809919?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


在spring的配置中会遇到各种各样的问题，其他无非就是bean配置上写错了路径，类目，对象名，变量名，等等，所以仔细一点，认真检查一下，肯定能找到原因。

到此为止，SSH框架已经搭建好了，但是据我所知，这套框架目前的使用率已经在降低了，有以下几个原因：

1. struts除了可以做请求转发，还有页面标签，所以你如果只用请求转发的话，这个框架有点多余
2. 现在spring推出了springMVC，是专门做请求转发用的，因为是spring自家推出的，所以和spring的协调性更好，而且在我使用中也感觉springMVC用起来更方便，轻量级
3. HIbernate框架管理数据库很强大，但是同样的问题，重量级。目前因为移动应用的兴起，请求并发量暴增的问题，Mybatis框架对于数据库管理更轻量级，更灵活。这两个框架说不上孰优孰劣，大家可以看下资料。

所以在下一篇文章中，准备先用SpringMVC代替struts。敬请期待
