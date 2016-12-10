---
title: Activity启动流程番外篇
date: 2016-10-26 17:29:01
tags:
  - Android
  - Activity启动
  - 系统
categories: Android
---

前两篇文章分析了Activity的启动流程的大部分。

第一篇文章讲了程序在调用startActivity之后发生的一些操作[Activity启动流程分析](http://zwgeek.com/2016/10/09/Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/)

第二篇文章讲了一个Android程序从最开始启动到一个Activity呈现到用户之间发生的一些操作[Activity启动流程分析（二）](http://zwgeek.com/2016/10/25/Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%902/)

在这两篇文章之前，我就提出了三个问题，前面也分析的差不多了，准备在这篇文章中来回答这三个问题，所以如果你还没有看前两篇文章，还是希望你能大概读一下之前两篇文章再来看这篇番外篇。

好，回归正题，提了三个问题，分别是哪三个问题呢。

- getApplication() ，getApplicationContext()，getBaseContext()分别有什么区别？
- 单个进程中能存在多个application么？
- 为什么不能使用service或者application作为创建dialog的context参数？




#### 第一个问题
getApplication() ，getApplicationContext()，getBaseContext()分别有什么区别？

对于这个问题，我们首先看一下这三个方法返回的对象分别是什么。

getApplication()
![这里写图片描述](http://img.blog.csdn.net/20161026174737355)

getApplication返回的是Activity中的mApplication对象，这个对象是在attach的时候赋给Activity的

![这里写图片描述](http://img.blog.csdn.net/20161026174850543)

第二篇文章中也讲到了attach的调用是在ActivityThread的performLaunchActivity中，传递的Application是通过makeApplication生成的。

![这里写图片描述](http://img.blog.csdn.net/20161026175138524)

我们来看下makeApplication

![这里写图片描述](http://img.blog.csdn.net/20161026175239338)

方法中我们可以发现makeApplication是会返回当前的mApplication的，那当前的这个mApplication到底是不是程序最开始创建的那个Application呢，也就是在bindApplicaiton中生成的Application。

![这里写图片描述](http://img.blog.csdn.net/20161026175707140)

要看这两个Application是否相同，我们可以跟踪创建它的LoadApk类，也就是这里的data.info和前面的r.packageInfo

好吧，继续追吧，看看能不能看到这两个对象的源头data.info是在handleBindApplication方法中通过getPackageInfoNoCheck获取到的。

![这里写图片描述](http://img.blog.csdn.net/20161026180630012)

getPackageInfoNoCheck调用了getPackageInfo方法。看下这个方法

![这里写图片描述](http://img.blog.csdn.net/20161026180758006)

这里可以很明显的看出，PackageInfo在ActivityThread中是单例的，这个对象跟packageName挂钩，那么我们也知道，一个Android程序的packageName是唯一的。所以基本可以确定前面两个创建Application的packageInfo是一样的，为了保险起见，我们再追一下另外一个packageInfo吧，也就是r.packageInfo。

![这里写图片描述](http://img.blog.csdn.net/20161026181449946)

看到了吧，也是调用前面的getPackageInfoNoCheck方法获取到的。所以正确流程下，一个Android程序中Application是唯一的，getApplication返回的就是这个对象。为什么说是正常流程下呢，这对应的是我们的第二个问题，后面会做出解答。

第二个方法getApplicationContext()
这个方法是在ContextWrapper中的，Activity是继承在这个类的。

![这里写图片描述](http://img.blog.csdn.net/20161026181724666)

这里又关系到mBase这个对象，这个对象其实也是在Activity attch的时候绑定进来的，就是图中这个地方创建的appContext，这个对象是ContextImpl类型的。

![这里写图片描述](http://img.blog.csdn.net/20161026181855572)

我们进去看下它的getApplicationContext方法

![这里写图片描述](http://img.blog.csdn.net/20161026182021733)

如果packageInfo不为null，那么返回packageInfo的Application，否则返回mainThread的Application

![这里写图片描述](http://img.blog.csdn.net/20161026182206188)

创建这个ContextImpl的packageInfo其实就是前面分析的r.packageInfo，所以这个ApplicationContext跟前面的Application是完全一样的。如果packageInfo是空，返回的是ActivityThread中的Application，而ActivityThread中的Application是handleBindApplication中生成的那个，跟前面也是一样的。

综上所述，getApplicationContext跟getApplication是相同的，接下来我们再看下getBaseContext()

![这里写图片描述](http://img.blog.csdn.net/20161026182544622)

这里返回的就是前面说的mBase，这个mBase是一个ContextImpl类型的对象，是为Activity专门创建的，与Application就不一样了。

好，这就是第一个问题的答案。

#### 第二个问题
单个进程中能存在多个application么？

忽然意识到第二个问题其实在第一个问题中已经做了部分回答，一个LoadedApk中只能存在一个application。

但是事实是这样吗，我们来看下ActivityThread中创建Application的地方。正如第一个问题分析的，虽然在第一句话的时候，限制了mApplication的单例性，但是请注意后面对Application的管理。

![这里写图片描述](http://img.blog.csdn.net/20161031110946531)

![这里写图片描述](http://img.blog.csdn.net/20161031110409263)

ActivityThread是一个进程，这里很明显我们已经能看出，Android中一个进程是允许存在多个Application的，但是正常情况下，是不会出现多个Application的，那么不正常的情况是什么呢，你可以通过反射得到这个mAllApplications，然后手动把自己的Application放进去。为了什么？当然是一些奇技淫巧了，后面有机会讲到插件化实现的时候会说到。

反正这里你知道一个结论就行了，一个进程是允许多个Application的，但是Android的正常流程下，一个进程只有一个Application。

#### 第三个问题
为什么不能使用service或者application作为创建dialog的context参数？

用service或者Application启动dialog的时候会报错

```java
Caused by: android.view.WindowManager$BadTokenException: Unable to add window -- token null is not for an application
                           at android.view.ViewRootImpl.setView(ViewRootImpl.java:685)
                           at android.view.WindowManagerGlobal.addView(WindowManagerGlobal.java:342)
                           at android.view.WindowManagerImpl.addView(WindowManagerImpl.java:93)
                           at android.app.Dialog.show(Dialog.java:316)

```
我们来看一下这个错误是哪里报出的，是在ViewRootImpl的setView中

![这里写图片描述](http://img.blog.csdn.net/20161026184204831)

对res进行判断，而这个res是这句话返回的，既然异常是关于token的，也就是说在addToDisplay的时候对当前view的token进行了检查。

![这里写图片描述](http://img.blog.csdn.net/20161027112934008)

mWindowSession是一个Session类型的，addToDidplay代码如下

![这里写图片描述](http://img.blog.csdn.net/20161028150931848)

其实如果你分析的多了，不用看也知道mService肯定是WindowManagerService，这个Service和ActivityManagerService一样，在Framework层是很重要的一个类，负责窗口绘制等。

异常报出的case是ADD_NOT_APP_TOKEN，找一下

![这里写图片描述](http://img.blog.csdn.net/20161028151340745)

所以看得出来，报出这个异常有两个条件，一是type是APPLICATION_WINDOW，二是要有APPWindowToken。

先看看这个TYPE吧

![这里写图片描述](http://img.blog.csdn.net/20161028151709929)

通过看这个类的注释，大概知道什么意思，WINDOW分为几大类
1-99是Application window
1000-1999是sub window
2000-2999是system window

挨个来看注释

##### Application Window
![这里写图片描述](http://img.blog.csdn.net/20161028152220400)

Application Window就是普通的应用程序window，像Activity，Dialog都属于这一类

##### Sub Window

![这里写图片描述](http://img.blog.csdn.net/20161028152438321)

SubWIndow，从注释可以看出，这类window必须有一个可依附的其他Window，然后在坐标轴Z上它是它所依附的Window的下一个，然后它的坐标体系也依附于它所依附的Window的坐标。

下面指出了一些Sub Window，Panel，media等等。

##### SYSTEM WINDOW

![这里写图片描述](http://img.blog.csdn.net/20161028152806777)

注释说这是系统window，一般不是由程序创建的，而在下面它给出的几类中，我们也看到了很多熟悉的身影。

![这里写图片描述](http://img.blog.csdn.net/20161028152932700)

状态栏，搜索栏，Toast都是SYSTEM WINDOW

这只是一个小插曲，我们回归正题，那么很明显Dialog应该是一个Application Window，不然它就不会报错了，我们从代码验证一下。

讲真，我有点打脸，因为我没有找到代码，Dialog的构造函数是这样的。

![这里写图片描述](http://img.blog.csdn.net/20161028155113014)

创建Window的代码在PolicyManager中，代码如下

![这里写图片描述](http://img.blog.csdn.net/20161028155158458)

这里感觉有点打脸了，难道代码跟不下去了？网上查了一些资料，看到一个比较合理的解释是，因为我的源码是SDK的源码，而不是真正编译系统的源码，所以这块内容被屏蔽掉了。没关系，万能的百度说正确的代码是下面这个样子的。

![这里写图片描述](http://img.blog.csdn.net/20161028162358849)

看到了静态区动态加载了Policy，这就好说了，我们去看Policy的makeNewWindow吧。

![这里写图片描述](http://img.blog.csdn.net/20161028162517980)

返回的是一个PhoneWindow，我没有从代码中找到明确把PhoneWindow设置类型为TYPE_APPLICATION的地方，于是想看一下Window的默认Type是什么。

![这里写图片描述](http://img.blog.csdn.net/20161028170422891)

![这里写图片描述](http://img.blog.csdn.net/20161028170435500)

![这里写图片描述](http://img.blog.csdn.net/20161028170445195)

所以，默认type是TYPE_APPLICATION类型的。PhoneWindow也就是TYPE APPLICATION类型的。

然后我们再看下一点，Dialog的Token是什么
Dialog的Token是null，这个可以从代码上看出来。第二个参数设置的就是token。

![这里写图片描述](http://img.blog.csdn.net/20161028165305907)

只是如果View的Token是null，WindowManager又会怎么操作呢。我们从异常处反向追一下代码。前面说到了，异常是从WindowManagerService的addWindow方法报出来的，检验的是Token是否是AppToken。

![这里写图片描述](http://img.blog.csdn.net/20161031102436168)

addWindow的调用链是这样的，我正向来说一下比较好理解，Dialog的show方法时，调用了mWindowManager的addView方法。

![这里写图片描述](http://img.blog.csdn.net/20161031102800676)

mWindowManager的实现是WindowManagerImpl，调用的是WindowManagerGlobal的addView

![这里写图片描述](http://img.blog.csdn.net/20161031102949037)

WindowManagerGlobal的addView方法调到了ViewRootImpl的setView

![这里写图片描述](http://img.blog.csdn.net/20161031103246376)

然后正如前面说到的，setView方法调到了通过WindowSession调用了WindowManagerService的addWindow，检查Token并报出异常。

这样就很明确了，这个token跟Dialog中的mWindowManager有关，初始化的时候已经看到mWindowManger跟Context有关。

![这里写图片描述](http://img.blog.csdn.net/20161031103849988)

如果这个Context是Activity，我们看下mWindowManager是如何表现的。在Activity中，如果要获取WindowService，Activity会把自己的mWindowManager返回过去。

![这里写图片描述](http://img.blog.csdn.net/20161031104032942)

而这个WindowManager的初始化是在Activity的attach方法中，从以下代码中我们看到，这个WIndowManager设置过appToken。

![这里写图片描述](http://img.blog.csdn.net/20161031104348943)

否则的话，如果Dialog的Context是其他，如Application，因为这几个Context没有重写getSystemService方法，所以调用的是父类ContextWrapper中的getSystemService。

![这里写图片描述](http://img.blog.csdn.net/20161031104822710)

在第二篇文章[Activity启动流程分析（二）](http://zwgeek.com/2016/10/25/Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%902/) 中分析过，ContextWrapper中的mBase是ContextImpl类型的。

ContextImpl中的getSystemtService获取的是SYSTEM_SERVICE_MAP中的Service。

![这里写图片描述](http://img.blog.csdn.net/20161031105217598)

这个map中的service是在static块中放进去的

![这里写图片描述](http://img.blog.csdn.net/20161031105401819)

其中也包括Window Service

![这里写图片描述](http://img.blog.csdn.net/20161031105444244)

这里很简单，就是返回了一个WindowManagerImpl的实例，而我们知道WindowManagerImpl的实例中是没有AppToken的。

综上所述，只有Activity的WindowManager是设置过AppToken的，所以，只有Context是Activity的时候，才能保证Dialog的show方法不返回异常。

为什么创建一个Dialog一定要带AppToken呢，其实不是什么技术上有问题，而是android的一种安全措施，原因在注释中也写的很清楚。

![这里写图片描述](http://img.blog.csdn.net/20161031113058149)

一个TYPE_APPLICATION类型的window必须带APPTOKEN，已区别它显示在哪个窗口上，这就限制了Dialog只能显示在创建它的Activity上，是一种安全措施。

ok，之前分析文章提出的三个问题已经作答完毕。

例行广告，喜欢这篇文章的同学可以关注我的博客[http://zwgeek.com](http://zwgeek.com)


