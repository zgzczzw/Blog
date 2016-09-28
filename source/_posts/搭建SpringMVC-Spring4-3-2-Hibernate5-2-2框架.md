---
title: 搭建SpringMVC+Spring4.3.2+Hibernate5.2.2框架
date: 2016-09-27 13:15:09
tags:
  - Java Web
  - 服务端
  - SpringMVC
categories: 服务端开发
---
之前说过，我最终想搭建一个SpringMVC+Spring+MyBatis的框架，然后从SSH框架开始慢慢演化，这篇博客将讲解怎样将SSH框架中的Struts部分替换为SpringMVC做请求转发。

至于为什么要替换成SpringMVC，我在[基于struts2.5.2+hibernate5.2.2+spring4.3.2搭建SSH框架](http://zwgeek.com/2016/09/23/%E5%9F%BA%E4%BA%8Estruts2-5-2-hibernate5-2-2-spring4-3-2%E6%90%AD%E5%BB%BASSH%E6%A1%86%E6%9E%B6/)这篇博客里说过，以下几点：

- struts除了可以做请求转发，还有页面标签，所以你如果只用请求转发的话，这个框架有点多余
- 现在spring推出了springMVC，是专门做请求转发用的，因为是spring自家推出的，所以和spring的协调性更好，而且在我使用中也感觉springMVC用起来更方便，轻量级

关于SSH框架的搭建可以去这篇文章查看

- [基于struts2.5.2+hibernate5.2.2+spring4.3.2搭建SSH框架](http://zwgeek.com/2016/09/23/%E5%9F%BA%E4%BA%8Estruts2-5-2-hibernate5-2-2-spring4-3-2%E6%90%AD%E5%BB%BASSH%E6%A1%86%E6%9E%B6/)

在上面的基础上把Struts换成SpringMVC，其实很简单，为什么这么说呢，因为搭建SSH框架的时候，我们把Spring的所有jar包都加入到项目里了，不知道你有没有注意到，有这样一个jar包

其实这个jar包就已经是对MVC的支持了，所以可以说我们上一个框架已经支持SpringMVC了，所以问题就变成了去掉Struts框架，所以很简单。

首先可以删掉Struts的所有jar包，主要是以下两个，其他common开头的jar包因为spring也在用，所以可以不删掉。
struts2-core和struts2-spring-plugin，现在lib如下，可做参考：
![](http://img.blog.csdn.net/20160927142022944)
![](http://img.blog.csdn.net/20160927142042958)

然后删掉struts.xml配置文件

接下来我们来看怎么让SpringMVC生效

首先把web.xml的拦截规则改成给SpringMVC拦截，如下

```xml
<servlet>
  	<servlet-name>springMVC</servlet-name>
  	<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
  	<init-param>
  		<param-name>contextConfigLocation</param-name>
  		<param-value>classpath*:config/spring/spring-mvc.xml</param-value>
  	</init-param>
  	<load-on-startup>1</load-on-startup>
  </servlet>
  <servlet-mapping>
  	<servlet-name>springMVC</servlet-name>
  	<url-pattern>/</url-pattern>
  </servlet-mapping>
```

然后在web.xml中配置SpringMVC的配置文件

```xml
<!-- 加载所有的配置文件 -->
  <context-param>
  	<param-name>contextConfigLocation</param-name>
  	<param-value>classpath*:config/spring/spring-*.xml</param-value>
  </context-param>
```

修改之后的web.xml如下
```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://java.sun.com/xml/ns/javaee" xmlns:web="http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd" xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd" id="WebApp_ID" version="2.5">
  <display-name>json_test</display-name>
  <welcome-file-list>
    <welcome-file>login.jsp</welcome-file>
  </welcome-file-list>
  
  <!-- 加载所有的配置文件 -->
  <context-param>
  	<param-name>contextConfigLocation</param-name>
  	<param-value>classpath*:config/spring/spring-*.xml</param-value>
  </context-param>
  
  <!-- 配置Spring监听 -->
  <listener>
  	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
  
  <!-- 配置SpringMVC -->
  <servlet>
  	<servlet-name>springMVC</servlet-name>
  	<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
  	<init-param>
  		<param-name>contextConfigLocation</param-name>
  		<param-value>classpath*:config/spring/spring-mvc.xml</param-value>
  	</init-param>
  	<load-on-startup>1</load-on-startup>
  </servlet>
  <servlet-mapping>
  	<servlet-name>springMVC</servlet-name>
  	<url-pattern>/</url-pattern>
  </servlet-mapping>
  
  <!-- 配置字符集 -->
  <filter>
  	<filter-name>encodingFilter</filter-name>
  	<filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
  	<init-param>
  		<param-name>encoding</param-name>
  		<param-value>UTF-8</param-value>
  	</init-param>
  	<init-param>
  		<param-name>forceEncoding</param-name>
  		<param-value>true</param-value>
  	</init-param>
  </filter>
  <filter-mapping>
  	<filter-name>encodingFilter</filter-name>
  	<url-pattern>/*</url-pattern>
  </filter-mapping>
  
  <!-- 配置Session -->
  <filter>
  	<filter-name>openSession</filter-name>
  	<filter-class>org.springframework.orm.hibernate5.support.OpenSessionInViewFilter</filter-class>
  </filter>
  <filter-mapping>
  	<filter-name>openSession</filter-name>
  	<url-pattern>/*</url-pattern>
  </filter-mapping>
</web-app>
```

字符集的配置是为了让所有请求和回复都用统一的字符集，比较方便。

OpenSession是为了延长Session的生命周期用的，可以自行baidu一下，网上是这样写的。

> OpenSessionInViewFilter是Spring提供的一个针对Hibernate的一个支持类，其主要意思是在发起一个页面请求时打开Hibernate的Session，一直保持这个Session，直到这个请求结束，具体是通过一个Filter来实现的。
　

> 由于Hibernate引入了Lazy Load特性，使得脱离Hibernate的Session周期的对象如果再想通过getter方法取到其关联对象的值，Hibernate会抛出一个LazyLoad的Exception。所以为了解决这个问题，Spring引入了这个Filter，使得Hibernate的Session的生命周期变长。

另外我们来看加载配置文件的地方

```xml
<!-- 加载所有的配置文件 -->
  <context-param>
  	<param-name>contextConfigLocation</param-name>
  	<param-value>classpath*:config/spring/spring-*.xml</param-value>
  </context-param>
```

这句话的意思是载入config/spring/目录下的所有以spring开头的配置文件。前面说过spring的配置其实就是在配置各种bean，在配置SSH的时候我们把所有bean都写在了applicationContext里面，导致这个文件很大很复杂，这样不利于修改配置和查找异常。其实我们可以把不同的bean配置在不同的文件中，然后在web.xml中告诉程序去哪里找配置文件，就像这里这样。我们可以把关于mvc的bean配置在sprng-mvc.xml中，把业务逻辑bean配置在spring-beans中，把基础bean（比如session，datasource等）配置在spring-common.xml中。

前面一篇文章也说过，eclipse在发布web项目的时候会把src目录映射成classpath目录，当然这是可以配置的。所以我们就在src目录下建立配置文件，如下

![](http://img.blog.csdn.net/20160927142443663)

接下来看一下mvc的配置是怎样的

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:mvc="http://www.springframework.org/schema/mvc"
	xsi:schemaLocation="http://www.springframework.org/schema/beans 
	http://www.springframework.org/schema/beans/spring-beans.xsd
	http://www.springframework.org/schema/context
	http://www.springframework.org/schema/context/spring-context-3.2.xsd
	http://www.springframework.org/schema/mvc
	http://www.springframework.org/schema/mvc/spring-mvc-3.2.xsd">
	
	<!-- 注解扫描包 -->
	<context:component-scan base-package="com.mvc.controller" />

	<!-- 开启注解 -->
	<mvc:annotation-driven />
	
	<!-- 静态资源(js/image)的访问 -->
	<mvc:resources location="/js/" mapping="/js/**"/>

	<!-- 定义视图解析器 -->	
	<bean id="viewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
		<property name="prefix" value="/"></property>
		<property name="suffix" value=".jsp"></property>
	</bean>
</beans>
```

现在主流的对SpringMVC的配置时基于注解的，至于怎么注解，我们后面再说，这个配置文件中前两部分就是配置注解用的，base-package指明注解在哪些包下，你当然可以写所有的包，让框架都去扫描一遍，但没必要。在SpringMVC框架中，控制转发的类叫Controller（对应Struts中的Action），所以注解通常也都是在com.xxx.xxx.controller包中，所以这里只扫描相应的包就可以。

mvc:resources定义静态文件的位置，因为我们前面在web.xml用/设置了过滤器，会拦截所有的请求，同时也会影响对静态文件的请求，这里配置之后，所有/js/**的请求都会去js文件夹去找，而不会跳转到controller控制类

ViewResolver是定义视图解析器，做什么用的的，简单来说，controller控制类中某个方法返回“index”的意思其实是跳转到index页面，但通常这个页面会有前缀（也就是相对路径）和后缀（扩展名），这里这样配置后，return "index"，就会跳转到/index.jsp页面。

解析完配置文件，我们看下Controller具体怎么写，其实就是在Struts框架中的Action类。代码如下：

```java
package com.mvc.controller;

import java.io.IOException;
import java.io.PrintWriter;
import java.util.List;

import javax.annotation.Resource;
import javax.servlet.http.HttpServletResponse;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

import com.mvc.manager.UserManager;
import com.mvc.pojo.User;

@Controller
public class UserController
{
	@Resource(name = "userManager") // 获取spring配置文件中bean的id为userManager的，并注入
	private UserManager userManager;

	@RequestMapping("/toAddUser")
	public String toAddUser()
	{
		return "/addUser";
	}

	@RequestMapping("/getAllUser")
	public void getAllUser(HttpServletResponse response)
	{
		System.out.println("getAllUser IN");
		List<User> user = userManager.getUsers();

		PrintWriter out = null;
		response.setContentType("application/json");

		try
		{
			out = response.getWriter();
			for (int i = 0; i < user.size(); i++)
			{
				out.write(user.get(i).getUserName());
			}
		}
		catch (IOException e)
		{
			e.printStackTrace();
		}
	}
}

```

关于注解的详细解析，准备在专门写一篇文章，简单来说，就是在方法上加上

```java
@RequestMapping("/getAllUser")
```

注解，对getAllUser的请求就会跳转到这个方法进行处理，像前面提到的，如果方法返回字符串，处理完之后就会做相应的跳转，如果方法返回void，则不进行任何跳转。

到目前为止SpringMVC就配置完了，这时你访问 http://host:port/projectName/getAllUser ，就能显示了。好，我们的第二个目标达成了，下个目标就是把Hibernate替换成Mybatis，然后就万事大吉了。

再列一下其他文件吧

spring-beans.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">

	<bean id="userDao" class="com.mvc.daoImpl.UserDao">
		<property name="sessionFactory">
			<ref bean="sessionFactory" />
		</property>
	</bean>

	<!--用户注册业务逻辑类 -->
	<bean id="userManager" class="com.mvc.manager.UserManager">
		<property name="dao">
			<ref bean="userDao" />
		</property>
	</bean>
</beans>
```

spring-common.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
	xmlns:mvc="http://www.springframework.org/schema/mvc"
	xsi:schemaLocation="http://www.springframework.org/schema/beans 
	http://www.springframework.org/schema/beans/spring-beans.xsd">

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
		<property name="dataSource" ref="dataSource" />
		<property name="mappingResources">
			<list>
				<value>com/mvc/pojo/User.hbm.xml</value>
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

	<!-- 配置一个事务管理器 -->
	<bean id="transactionManager"
		class="org.springframework.orm.hibernate5.HibernateTransactionManager">
		<property name="sessionFactory" ref="sessionFactory" />
	</bean>

	<!-- 配置事务，使用代理的方式 -->
	<bean id="transactionProxy"
		class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean"
		abstract="true">
		<property name="transactionManager" ref="transactionManager"></property>
		<property name="transactionAttributes">
			<props>
				<prop key="add*">PROPAGATION_REQUIRED,-Exception</prop>
				<prop key="modify*">PROPAGATION_REQUIRED,-myException</prop>
				<prop key="del*">PROPAGATION_REQUIRED</prop>
				<prop key="*">PROPAGATION_REQUIRED</prop>
			</props>
		</property>
	</bean>
</beans>
```

其他像DAO类，manager类就不细说了，不懂得可以看我上一篇文章[基于struts2.5.2+hibernate5.2.2+spring4.3.2搭建SSH框架](http://zwgeek.com/2016/09/23/%E5%9F%BA%E4%BA%8Estruts2-5-2-hibernate5-2-2-spring4-3-2%E6%90%AD%E5%BB%BASSH%E6%A1%86%E6%9E%B6/)

整个项目的结构如下

![](http://img.blog.csdn.net/20160927143143944)
