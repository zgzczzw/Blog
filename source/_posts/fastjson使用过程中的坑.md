---
title: fastjson使用过程中的坑
date: 2017-05-09 11:44:21
tags:
  - FastJson
  - Json序列化
  - Android
categories: Android
---
例行推广一下我的博客，喜欢这篇文章的朋友可以关注我的博客[http://zwgeek.com](http://zwgeek.com)

最近在工作中用到了fastjson，遇到了一些坑，在这里总结一下。

### 简介
首先，介绍一下fastjson。fastjson是由alibaba开源的一套json处理器。与其他json处理器（如Gson，Jackson等）和其他的Java对象序列化反序列化方式相比，有比较明显的性能优势。详情可以参考fastjson提供的benchmark。

[benchmark for fastjson](https://github.com/eishay/jvm-serializers/wiki)

fastjson的使用方法也比较简单，详细使用可以参考[fastjson首页](https://github.com/Alibaba/fastjson/wiki/%E9%A6%96%E9%A1%B5)，这里只做一个简单介绍。

首先可通过Gradle或Maven或直接引用jar包的方式来引入fastjson。
```
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>VERSION_CODE</version>
</dependency>
```
或
```
compile 'com.alibaba:fastjson:VERSION_CODE'
```

#### 序列化方式
```
String jsonString = JSON.toJSONString(class);
```
#### 反序列化方式
```
Class class = JSON.parseObject(jsonString, Class.class);
```
#### 泛型反序列化
泛型对象比如List的序列化方法和普通对象一样，反序列化的时候需要指定一下泛型对象，例如：
```
List<VO> list = JSON.parseObject("...", new TypeReference<List<VO>>() {});
```
以上就是fastjson的简单用法，接下来说一下我在使用过程中遇到的坑。

###遇到的问题

### 1. 序列化的类必须有一个无参构造方法
被序列化的类需要有一个无参的构造方法。否则会报错
```
Exception in thread "main" com.alibaba.fastjson.JSONException: default constructor not found. class User
```
如果你没有重写构造方法，那么每个类都自带一个无参的构造方法，但是如果你重写了一个有参的构造方法，那么默认的无参构造方法会被覆盖，这时候就需要你手动写一个无参的构造方法进去。所以我建议保险起见，需要被json序列化的类最好都手动写一个无参的构造方法进去。

对于这个问题，我怀疑fastjson是这样处理的（没有验证代码）。它在反序列化的时候会先用无参的构造方法构造一个类实例。然后将每一个值解析出来赋值到这个实例中。所以很明显，除了无参的构造方法，你还需要保证被序列化的成员变量能被访问到，经过我测试fastjson支持以下两种访问方式。

- public访问权限

- get和set方法


### 2. 子类不能重写父类成员变量
 正如第二点提到的，fastjson是通过类的成员变量名称来识别成员变量的。所以如果子类复写的父类的成员变量，会导致fastjson识别错乱。

这个问题我在Android版本的使用过程中遇到了，后来在Java版本的测试中却无法复现这个问题，所以并不是所有的复写父类的成员变量都不能被识别。只有在类结构比较复杂的时候才容易出问题，不管怎么样，我建议使用过程中尽量避免复写父类的成员变量。

### 3. 混淆要对泛型进行保护
这个问题其实也不是fastjson的问题，但这个问题却是困扰我最久的一个问题，本地运行没有问题，但是一打Release就有问题。这种情况99%是由于Proguard引起的，但我当时Proguard配置文件中已经keep了所有的类，按理说不是由于类被混淆而找不到造成的。

后来我发现这个问题只有在遇到List的时候才会出错，所以开始怀疑是不是泛型在Proguard的时候出现了问题，后来发现了Proguard的官网，果然Proguard有对泛型的保护。后来在fastjson的issue中也发现有人提出了这个问题，并且发现除了泛型之外还需要对注解进行保护。

[fastjson issue传送门](https://github.com/alibaba/fastjson/issues/765)
[Proguard指南传送门](https://www.guardsquare.com/en/proguard/manual/examples#annotations)

通过以上两篇文章，我们知道需要在Proguard的配置文件中加入以下属性。

```
-keepattributes Signature
-keepattributes *Annotation*
```
If your application, applet, servlet, library, etc., uses annotations, you may want to preserve them in the processed output. Annotations are represented by attributes that have no direct effect on the execution of the code. However, their values can be retrieved through introspection, allowing developers to adapt the execution behavior accordingly. By default, ProGuard treats annotation attributes as optional, and removes them in the obfuscation step.

The "Signature" attribute is required to be able to access generic types when compiling in JDK 5.0 and higher.

由这个问题我深深的感觉到自己最Proguard的不熟悉，以前仅仅会从网上拷贝一个默认的混淆配置问题，而对于细节配置全然不知，所以我决定接下来翻译下Proguard官网。

以上就是使用fastjson中遇到的问题，以及解决方案。fastjson的性能确实不错，而且对Model的入侵性低，使用简单。是序列化和反序列化的好帮手。

例行推广一下我的博客，喜欢这篇文章的朋友可以关注我的博客[http://zwgeek.com](http://zwgeek.com)