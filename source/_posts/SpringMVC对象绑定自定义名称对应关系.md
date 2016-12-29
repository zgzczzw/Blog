---
title: SpringMVC对象绑定时自定义名称对应关系
date: 2016-12-28 18:15:57
tags:
  - SpringMVC
  - 请求对象绑定
categories: 服务端开发
---

例行推广一下我的博客，喜欢这篇文章的朋友可以看我的博客[http://zwgeek.com](http://zwgeek.com)

这个需求来源自一个Post的Controller的请求含有太多的参数，于是想把所有的参数封装到对象中，然后Controller的方法接收一个对象类型的参数，这样后期扩展修改都比较方便，不需要动到方法签名。

有一句俗话说得好，需求是第一生产力，上面的这个需求就催生了这篇文章的一系列调研。

首先，这个需求SpringMVC本身是支持的，你把一个对象放在Controller方法的参数里，SpringMVC本身的支持就能把request中和这个对象属性同名的参数绑定到对象属性上，比如下面这样：

```java
@RequestMapping(value = "/test", method = RequestMethod.GET)
public void test(Test test) {
   LOG.debug(test.toString());
}
```

Test中是这样定义的
```
public class Test {

    private String test1;

    private String test2;

    private String test3;
}
```
由于我使用了lombok，所以没有Getter和Setter方法，大家看demo的时候可以注意下。这个时候我们访问/test?test1=1&test2=2&test3=3，SpringMVC会自动把同名的属性和Request中的参数绑定在一起。

![这里写图片描述](http://img.blog.csdn.net/20161228184306167?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemd6Y3p6dw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

到这里貌似可以结题了，开玩笑，哪有这么简单。大家想这样一种情况，在JAVA中我们用的是驼峰命名法（比如：testName），而在前端JS中我们大多用的是蛇形命名法（比如：test_name）。当然，我们可以要求前端在请求接口的时候用驼峰命名法，但是问题不是这样逃避的。另外，如果我前端接口参数的名字和对象里面属性的名字不一样怎么办呢，比如前端接口里参数叫person，但是我对象里的属性叫people，当然这种情况比较少，但是也不排除在各种复杂的需求中会出现这种情况，所以我们这篇文章的目的就是做到自定义的参数名和属性名映射，想让哪个请求参数对应哪个属性都可以，想点哪里点哪里就是这个意思。

这篇文章只讲解决方案，我会在下一篇文中讲一下SpringMVC数据绑定的实现原理。

虽然不细说原理，但是有几个概念还是要提前说一下的，使用SpringMVC时，所有的请求都是最先经过DispatcherServlet的，然后由DispatcherServlet选择合适的HandlerMapping和HandlerAdapter来处理请求，HandlerMapping的作用就是找到请求所对应的方法，而HandlerAdapter则来处理和请求相关的的各种事情。我们这里要讲的请求参数绑定也是HandlerAdapter来做的。大概就知道这些吧，我们需要写一个自定义的请求参数处理器，然后把这个处理器放到HandlerAdapter中，这样我们的处理器就可以被拿来处理请求了。

首先第一步，我们先来做一个参数处理器SnakeToCamelModelAttributeMethodProcessor
```java
public class SnakeToCamelModelAttributeMethodProcessor extends ServletModelAttributeMethodProcessor {

    ApplicationContext applicationContext;

    public SnakeToCamelModelAttributeMethodProcessor(boolean annotationNotRequired, ApplicationContext applicationContext) {
        super(annotationNotRequired);
        this.applicationContext = applicationContext;
    }

    @Override
    protected void bindRequestParameters(WebDataBinder binder, NativeWebRequest request) {
        SnakeToCamelRequestDataBinder camelBinder = new SnakeToCamelRequestDataBinder(binder.getTarget(), binder.getObjectName());
        RequestMappingHandlerAdapter requestMappingHandlerAdapter = applicationContext.getBean(RequestMappingHandlerAdapter.class);
        requestMappingHandlerAdapter.getWebBindingInitializer().initBinder(camelBinder, request);
        camelBinder.bind(request.getNativeRequest(ServletRequest.class));
    }
}
```

这个处理器要继承ServletModelAttributeMethodProcessor，来看下继承关系
![这里写图片描述](http://img.blog.csdn.net/20161228201634838?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemd6Y3p6dw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

看最上面实现的是HandlerMethodArgumentResolver接口，这个接口代表这个类是用来处理请求参数的，有两个方法
```java
public interface HandlerMethodArgumentResolver {
    boolean supportsParameter(MethodParameter var1);

    Object resolveArgument(MethodParameter var1, ModelAndViewContainer var2, NativeWebRequest var3, WebDataBinderFactory var4) throws Exception;
}

```
supportsParameter返回是否支持这种参数，resolveArgument是具体处理参数的方法。ServletModelAttributeMethodProcessor是处理复杂对象的，也就是除了int，char等等简单对象之外自定义的复杂对象，比如上文中我们提到的Test。

我们自定义的处理器也是处理复杂对象，只是扩展了可以处理名称映射，所以继承这个ServletModelAttributeMethodProcessor即可。好了，处理器写好了，那么接下来怎么做呢，重写父类的bindRequestParameters方法，这个方法就是绑定数据对象的时候调用的方法。

在这个方法中，我们新建了一个自定义的DataBinder-SnakeToCamelRequestDataBinder，然后用HandlerAdapter初始化了这个DataBinder，最后调了DataBinder的bind方法。DataBinder顾名思义就是实际去把请求参数和对象绑定的类，这个自定义的DataBinder怎么写呢，如下：

```java
public class SnakeToCamelRequestDataBinder extends ExtendedServletRequestDataBinder {

    public SnakeToCamelRequestDataBinder(Object target, String objectName) {
        super(target, objectName);
    }

    protected void addBindValues(MutablePropertyValues mpvs, ServletRequest request) {
        super.addBindValues(mpvs, request);

        //处理JsonProperty注释的对象
        Class<?> targetClass = getTarget().getClass();
        Field[] fields = targetClass.getDeclaredFields();
        for (Field field : fields) {
            JsonProperty jsonPropertyAnnotation = field.getAnnotation(JsonProperty.class);
            if (jsonPropertyAnnotation != null && mpvs.contains(jsonPropertyAnnotation.value())) {
                if (!mpvs.contains(field.getName())) {
                    mpvs.add(field.getName(), mpvs.getPropertyValue(jsonPropertyAnnotation.value()).getValue());
                }
            }
        }

        List<PropertyValue> covertValues = new ArrayList<PropertyValue>();
        for (PropertyValue propertyValue : mpvs.getPropertyValueList()) {
            if(propertyValue.getName().contains("_")) {
                String camelName = SnakeToCamelRequestParameterUtil.convertSnakeToCamel(propertyValue.getName());
                if (!mpvs.contains(camelName)) {
                    covertValues.add(new PropertyValue(camelName, propertyValue.getValue()));
                }
            }
        }
        mpvs.getPropertyValueList().addAll(covertValues);
    }
}
```
这个自定义的DataBinder继承自ExtendedServletRequestDataBinder，可扩展的DataBinder，用来给子类复写的方法是addBindValues，有两个参数，一个是MutablePropertyValues类型的，这里面存的就是请求参数的key-value对，还有一个参数是request对象本身。request对象这里用不到，我们用的就是这个MutablePropertyValues类型的mpvs。

其实处理的原理很简单，SpringMVC在做完这一步参数绑定之后就会去通过反射调用Controller中的方法了，调用Controller方法的时候要给参数赋值，赋值的时候就是从这个mpvs里面把对应参数name的value取出来。举个例子，我们的样例Controller中的对象时Test，Test有个属性是Test1，那么在给Test1赋值的时候就会从这个mpvs中去取key为Test1所对应的value。可是你想想，前端请求的参数是test\_1这样的，所以这个mpvs中只有一个key为test\_1的值，那自然就会报错。知道了这种处理方法，就很简单了，我们在这个mpvs中再加一个key为test1，value和test_1的value一样的对象就可以了。

再扩展一点，说到自定义Name，比如test\_1对应属性为test2这样，我们用一个有value的注释，比如这里用到的JsonProperty，在test2上加上注释@JsonProperty("test_1")，这里处理的时候会先把这个注释的值取出来，从mpvs里面查，如果有这个key，那么就把value取出来再加进去一个key为field name的map就可以了。上面SnakeToCamelRequestDataBinder的处理方法大概就是这样了。

然后这里我们抽出了一个工具类，用来处理蛇形string到驼峰string的转换。

```java
public class SnakeToCamelRequestParameterUtil {
    public static String convertSnakeToCamel(String snake) {

        if (snake == null) {
            return null;
        }

        if (snake.indexOf("_") < 0) {
            return snake;
        }

        String result = "";

        String[] split = StringUtils.split(snake, "_");
        int index = 0;
        for (String s : split) {
            if (index == 0) {
                result += s.toLowerCase();
            } else {
                result += capitalize(s);
            }
            index++;
        }

        return result;
    }

    private static String capitalize(String s) {

        if (s == null) {
            return null;
        }

        if (s.length() == 1) {
            return s.toUpperCase();
        }

        return s.substring(0, 1).toUpperCase() + s.substring(1);
    }
}
```

做完了上面这些，应该说处理过程就搞定了，还差最后一步，需要把我们自定义的处理器SnakeToCamelModelAttributeMethodProcessor加到系统的HandlerAdapter中去。方法有很多，如果你不知道HandlerAdapter是什么东西，那八成你用的是系统默认的HandlerAdapter。加起来也很简单。如果你用了

```xml
<mvc:annotation-driven>
```

元素，可以用下面这个方法。

```xml
<mvc:annotation-driven>
    <mvc:argument-resolvers>
        <bean class="package.name.SnakeToCamelModelAttributeMethodProcessor">
            <constructor-arg name="annotationNotRequired" value="true"/>
        </bean>
    </mvc:argument-resolvers>
</mvc:annotation-driven> 
```

如果你用的是JAVA代码配置，可以用

```java
@Configuration
public class WebContextConfiguration extends WebMvcConfigurationSupport {
    @Override
    protected void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        argumentResolvers.add(processor());
    }

    @Bean
    protected SnakeToCamelModelAttributeMethodProcessor processor() {
        return new SnakeToCamelModelAttributeMethodProcessor(true);
    }
} 
```

像我这边，因为项目需要自定义了HandlerAdapter。

```
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">
        ...
    </bean>
```

所以我写了一个注册器，用来把处理器注册进HandlerAdapter，代码如下。

```java
public class SnakeToCamelProcessorRegistry implements ApplicationContextAware, BeanFactoryPostProcessor {

    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {

        RequestMappingHandlerAdapter requestMappingHandlerAdapter = applicationContext.getBean(RequestMappingHandlerAdapter.class);

        List<HandlerMethodArgumentResolver> resolvers = requestMappingHandlerAdapter.getArgumentResolvers().getResolvers();


        List<HandlerMethodArgumentResolver> newResolvers = new ArrayList<HandlerMethodArgumentResolver>();

        for (HandlerMethodArgumentResolver resolver : resolvers) {
            newResolvers.add(resolver);
        }
        newResolvers.add(0, new SnakeToCamelModelAttributeMethodProcessor(true, applicationContext));
        requestMappingHandlerAdapter.setArgumentResolvers(Collections.unmodifiableList(newResolvers));
    }
}
```
看起来可能逻辑比较复杂，为什么要做这一堆事情呢，话要从HandlerAdapter里系统自带的处理器说起。我这边系统默认带了24个处理器，其中有两个ServletModelAttributeMethodProcessor，也就是我们自定义处理器继承的系统处理器。SpringMVC处理请求参数是轮询每一个处理器，看是否支持，也就是supportsParameter方法， 如果返回true，就交给你出来，并不会问下面的处理器。这就导致了如果我们简单的把我们的自定义处理器加到HandlerAdapter的Resolver列中是不行的，需要加到第一个去。

然后ServletModelAttributeMethodProcessor的构造器有一个参数是true，代表什么意思呢，看这句代码
```
public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(ModelAttribute.class)?true:(this.annotationNotRequired?!BeanUtils.isSimpleProperty(parameter.getParameterType()):false);
    }
```
ServletModelAttributeMethodProcessor是否支持某种类型的参数，是这样判断的。首先，对象是否有ModelAttribute注解，如果有，则处理，如果没有，则判断annotationNotRequired，是否不需要注释，如果true，再判断对象是否简单对象。我们的Test对象是没有注释的，所以我们就需要传参为true，表示不一定需要注解。

以上就是所有的配置过程，通过这一系列配置，我们就可以自定义前端请求参数和对象属性名称的映射关系了，通过JsonProperty注解，如果没有注解，会自动转换蛇形命名到驼峰命名。下面是效果，Demo的Controller依然是这个：
```
@RequestMapping(value = "/test", method = RequestMethod.GET)
    @ResponseBody
    public void test(Test test) {
        LOG.debug(test.toString());
    }
```
Test对象这样写

```
public class Test {

    @JsonProperty("test_1")
    private String test1;

    private String test2;

    @JsonProperty("test_3")
    private String test99;
}
```
这样前端请求的参数中的test\_1和test1都会绑定到test1，test\_2和test2都会绑定到test2，test\_3,test99和test\_99都会绑定到test99上。我们试一下这个请求
```
/test?test_1=1&test_2=2&test_3=3
```
Log输出如下

![这里写图片描述](http://img.blog.csdn.net/20161229123336052?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemd6Y3p6dw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

下一篇文章准备详细的讲一下SpringMVC处理请求最后映射到一个Controller方法上的过程。

例行推广一下我的博客，喜欢这篇文章的朋友可以看我的博客[http://zwgeek.com](http://zwgeek.com)


