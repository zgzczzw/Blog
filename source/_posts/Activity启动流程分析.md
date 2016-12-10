---
title: Activity启动流程分析
date: 2016-10-09 14:08:25
tags:
  - Android
  - Activity启动
  - 系统
categories: Android
---
广告时间，大家喜欢我的文章，可以关注我的博客[zwgeek.com](zwgeek.com)

今天想和大家一起分享一下Activity的启动流程。这起源于我发现了一个好的现象，其实不知道大家发现没有，随着Android的发展，这几年Android开发者的素质也越来越高，我说的素质指的是对问题深度的理解，对Android总体的运行原理的分析，而不仅仅局限在应用开发层面了。还记得最开始接触Android的时候，那时候不管面试还是干嘛的，上来就问你生命周期，生命周期。感觉能说清楚生命周期已经是一件很厉害的事了。我也曾一度觉得开发者只要清楚onCreate什么时候调用就可以了，onCreate就是开发者能接触的最上层的内容了。根本没有考虑过程序应该有个入口，onCreate到底是怎么被调用的，更不用说还想着去看看源码。而现在，好像大家比起关心生命周期，关心起了更深层的问题，也就是一个Android程序是怎么被启动的，然后才是它的生命周期是怎么被调用的。我最先意识到这个现象时，是很开心的，这说明Android开发者越来越高级，也就意味着现在开发者不光使用Android，而且也能为Android社区做一些力所能及的贡献了。

这篇文章的目标是总结网上各种对Activity启动流程的分析。现在网上对Activity启动流程的分析已经多如牛毛了。那么我为什么还觉得有必要再写一次呢。一是自己边分析边写更有利于理解。二是我发现网上的分析习惯性的忽略细节，只着眼于一条大的主线，也不是说这样不好，这样确实比较容易理解，我希望我这篇文章能关注到每个细节点，每个方法的作用。

整个过程准备分为三篇文章来写
- 程序调用startActivity后发生的操作，也就是这篇文章。
- 如果被startActivity的程序是需要重新启动的程序，程序在最开始初始化时发生的操作。例如在Launcher中启动一个程序或启动另外的程序。已经完成，可查阅[Activity启动流程分析（二）](http://zwgeek.com/2016/10/25/Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%902/)
- 如果被startActivity的程序是已经启动的程序，发生的操作。例如程序自己调用startActivity启动一个自己程序中的Activity


整个分析过程中，希望读者带着这样一些问题
- getApplication() ，getApplicationContext()，getBaseContext()分别有什么区别？
- 单个进程中能存在多个application么？
- 为什么不能使用service或者application作为创建dialog的context参数？

讲了这么多，我们可以看代码吧，从哪里开始看呢，就从Activity的startActivity方法作为起点吧。也就是这个方法

![这里写图片描述](http://img.blog.csdn.net/20161009143617855)
Bundle为空的情况，又调用了带Bundle的同名方法，bundle传null，如下

![这里写图片描述](http://img.blog.csdn.net/20161009143857577)

startActivity方法本质其实是调用startActivityForResult的，requestCode传了-1，这个倒是我之前没想到的，一直以为这两个方法是分开的。所以requestCode为-1的情况应该是被系统默认忽略的（后面的分析过程中也发现requestCode为负都是被忽略的）。startActivityForResult的两个方法跟startActivity一样，最终也是进到了带Bundle的方法。

这里先看下这个方法的注释
![这里写图片描述](http://img.blog.csdn.net/20161009145144243)

- requestCode为负数时无效，跟调startActivity一样
- 这个方法仅仅试用于能够在当前Task（关于Task是什么，后面也会讲到，暂时可以考虑成是一个Activity的集合）启动的Activity，只有在这个集合中的Activity才能互相传递Result，如果Activity不能在当前Task启动，则这个方法会立即返回失败
- 在onCreate和onResume中调用startActivity时，前一个Activiyt不会显示，直到后面启动的Activity关闭返回结果，这是为了防止前一个Activity显示造成的闪烁
- 方法会抛出一个ActivityNotFoundException

![这里写图片描述](http://img.blog.csdn.net/20161009152006663)

这个方法一共有两种情况，mParent为null和不为null，mParent其实是指的ActivityGroup，但是这种用法现在已经被弃用了，所以这条线可以不用关注。无伤大雅，我们顺便来看一下吧。

先看一下下个启动过程，当mParent不为null的时候，直接调用mParent的startActivityFromChild方法，如下
![这里写图片描述](http://img.blog.csdn.net/20161009153000134)

在startActivityFromChild中又通过mInstrumentation调用了其中的execStartActivity方法。

在继续之前，有两个小插曲，先说一下startActivityForResult的最后一步，mActivityTransitionState.startExitOutTransition方法，处理退出当前Activity的相关事宜，这个不太用关注。

然后是mInstrumentation，看过网上文章的同学应该熟悉Instrumentation这个类，Instrumentation这个类是一个管家类，所有跟Activity有关的实际操作都交给它做，恩，与其说大管家，不如说是打杂的。这样有什么好处呢，这个类可以做一些统计工作，因为它知道Activity有关的所有事情。

然后，我们继续，回过头来看mParent为null时候的调用，其实也是调用了mInstrumentation的execStartActivity方法。可以看出，这两个分支最后殊途同归，到了execStartActivity方法，只是有两个参数不同，这就值得我们去看下这两个参数有什么影响，第一个和第四个，两个Activity类型的参数。其实这里mToken也是不一样的，mToken相当于activity的唯一标识符，这个token关系到后面新创建的Activity加入到哪个task栈中。这三个参数体会一下，后面也会遇到一些关于Activity的问题基本都和这里传递的参数有关。

```java
if (mParent == null)
mInstrumentation.execStartActivity(this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
```

```java
if(mParent != null)
mInstrumentation.execStartActivity(this, mMainThread.getApplicationThread(), mToken, child,
                intent, requestCode, options);
```

这里先调用mInstrumentation的execStartActivity方法，返回一个ActivityResult。如果ActivityResult不为空，调用mMainThread的sendActivityResult方法。这其实是关于Activity通过startActivityForResult启动Activity然后返回Result的过程，这部分准备单独拉一篇文章来讲，这里就不多说了。

然后mMainThread是一个ActivityThread类型的，说起ActivityThread可厉害了，这个类其实就是应用程序的入口，main方法就在这个类里，我们的第二篇文章就主要是对这个类的分析。

然后这里mMainThread.sendActivityResult方法的作用就是调用onActivityResult，其实看到这里我不太明白，onActivityResult应该是新启动的Activity关闭后才调用。为什么这里execStartActivity之后就调用了，道理上这里Activity才启动，并没有关闭啊。我能想到的原因之一是execStartActivity是一个阻塞方法，只有当Activity关闭之后才会返回结果，然后继续往下执行，不知道我想的对不对，后面分析验证。

关于sendActivityResult是怎么调到onActivityResult的，和所有生命周期方法的调用一样，通过Handler，这个后面会统一讲一下。

下面我们去看Instrumentation的execStartActivity方法了。例行先看一下注释
![这里写图片描述](http://img.blog.csdn.net/20161009162711191)

这个方法执行一个有应用程序发出的startActivity的请求，默认的实现会更新所有ActivityMonitor的信息，你可以用这个类监控activity的启动情况，并且做一些额外操作，这也是前面提到的可以在Instrumentation中监控每个Activity的启动情况。

当ActivityMonitor想要中断Activity启动时，这个方法依然会返回一个正确的ActivityResult。结合下面的代码，我前面提到的那个问题就明白了，这个方法并不阻塞，而且正常流程情况下也不会返回result，触发onActivityResult。只有当你设置了一个监控器，并且监控器是想阻止这个Acitivity启动的时候，这个方法不会真正去启动一个Activity，但是还是会正常返回Result值。

另外前面也提到，有两个参数要重点看下，第一个参数Context，是正在启动的Activity属于哪个Context，这个参数永远都是Parent，当没有Parent的时候是自己。第四个参数Activity，是具体执行startActivity任务的那个Activity，同时这个Activity负责处理onActivityResult。明白了这两个参数后，我们来看具体的代码。

![这里写图片描述](http://img.blog.csdn.net/20161009165506838)

基本都在注释里写清楚了，核心方法是调了ActivityManagerNative.getDefault().startActivity方法getDefault是用系统提供的单例方法构造的一个单例对象，感觉这个方法也挺有意思，下次可以研究下。

![这里写图片描述](http://img.blog.csdn.net/20161009170243301)

然后我们看ActivityManagerNative类中的startActivity方法
![这里写图片描述](http://img.blog.csdn.net/20161009170645365)

我们来看ActivityManagerNative这个类，会发现它继承了Binder，如果你对Android的Binder体系熟悉的话，你应该能明显的看出来，这个方法是一个调用远端进程方法，封装数据，然后调用了Remote的transact方法。Binder也是Android中可以拖出来讲很久的一个体系，后面也会拖一篇文章来讲这个。所以ActivityManagerNative就是一个Binder对象，我们需要找到他对应的远端Service。追一下mRemote赋值的地方，会发现是在创建的时候。

![这里写图片描述](http://img.blog.csdn.net/20161009171802041)


插播一句，这里Singleton类是java中实现的单例模式，具体这个单例实现的效果如果，然后有什么优化，不是很清楚，准备有空分析下这个类。但是想想系统实现的应该不会太差，所以写单例模式的时候也可以用这个类来。

我们回归正题，这个IBinder也可以跟到，简单来说，ActivityManagerNative的远端服务就是ActivityManagerService。整个Android程序其实是一个C/S结构，本地程序是一个Client，还一个Service端负责所有的程序调度。启动Activity肯定是系统的事啊，所以这里程序就把整个工作推给系统来做了。通过Binder模式，我们调到了远端Service的startActivity方法，来看这个方法。

![这里写图片描述](http://img.blog.csdn.net/20161009172405939)

继续跟startActivityAsUser方法
![这里写图片描述](http://img.blog.csdn.net/20161009172721029)

首先会检查，当前进程是否是孤立进程。孤立进程是Service的一个属性。

```java
android:isolatedProcess
```

如果Service设置了这个属性，那么这个Service将运行在一个特殊的进程中，这个进程和系统其他进程是分开的。这个进程没有任何权限。和这样的Service进行交互就只能通过系统API（也就是bind和start）。

简单来说，非孤立进程可以被拿来重用，重新运行新的app，孤立进程由于权限等问题，不能被重用，只能被销毁。所以在执行启动操作的时候要判断当前进程是否孤立进程，如果是，则不允许做start的操作。

回到正题，startActivityAsUser方法调用了StackSupervisor的startActivityMayWait方法，StackSupervisor是一个Activity栈的管理器，Activity栈是用来存储已经生成的的Activity对象的。这个我们会在后面说明。来看下StackSupervisor的startActivityMayWait方法。我看了一下整个流程，前面处理了Intent，AcitivtyStack等，为startActivity做准备，后面返回Result，都不是很重要。所以我们继续往下追，只寻找关于startActivity的部分。startActivityMayWait中的这一句

![这里写图片描述](http://img.blog.csdn.net/20161009180248576)

调用了startActivityLocked方法，startActivityLocked中，也是先进行了一系列的检查，包括intent中各种flag的判定，以一个err命令贯穿整个方法，如果中间有任何异常err就会被改变，就不能继续往下进行。

![这里写图片描述](http://img.blog.csdn.net/20161009180410115)


然后，调用到startActivityUncheckedLocked中，这个方法也是一个超级长的方法，对startActivity做了各种检查和准备， 主要是对Activity分各种Flag进行相应的处理。这个方法里面就有我们常见的Activity的各种启动模式的配置。如图

![这里写图片描述](http://img.blog.csdn.net/20161025140715881)

前面说到startActivityForResult的时候，提过必须在一个task中，处理也是这里。如果不是一个task，会response一个异常码。

![这里写图片描述](http://img.blog.csdn.net/20161025140911557)

这里其他代码的处理太细节化了，我们抓一下主线，生成一个ActivityStack，并
调用Stack的startActivityLocked方法
![这里写图片描述](http://img.blog.csdn.net/20161009180849750)

这样我们就到了ActivityStack类里面，但是没过多久，startActivityLocked通过调用resumeTopActivitiesLocked又回到了ActivityStackSupervisor。

![这里写图片描述](http://img.blog.csdn.net/20161025143058036)

![这里写图片描述](http://img.blog.csdn.net/20161025143315259)

![这里写图片描述](http://img.blog.csdn.net/20161025143426099)

做了一些检查，然后调到了resumeTopActivityInnerLocked，在resumeTopActivityInnerLocked中，因为thread和app都为null，所以执行了startSpecificActivityLocked方法，这就重新回到了ActivityStackSupervisor中

![这里写图片描述](http://img.blog.csdn.net/20161025144451400)


![这里写图片描述](http://img.blog.csdn.net/20161009182225273)


![这里写图片描述](http://img.blog.csdn.net/20161009182516474)

我们可以看到startSpecificActivityLocked中，如果app和thread为null，会调用startProcessLocked启动一个新的线程。后面也会说到，新线程启动后还会回到这里，去调用真正的realStartActivityLocked，我们可以发现源码里的名字起的也是相当到位。

这里暂时thread和app都是null，所以通知ActivityManagerService的startProcessLocked方法启动一个进程

在ActivityManagerService的startProcessLocked方法中，前面处理了一下线程重用等的优化，用processRecord记录了要启动的进程的信息

![这里写图片描述](http://img.blog.csdn.net/20161025145835484)

然后调了同名的startProcessLocked方法，但是参数由String的processName变成了ProcessRecord

![这里写图片描述](http://img.blog.csdn.net/20161009183016089)

在这个方法中，通过Process.start启动了一个进程，并指定了进程的入口，也就是ActivityThread类的main方法。至于这个进程的启动过程涉及到Android内核层的东西了，这篇文章暂时不看这么细，我们就简单的理解为，新进程启动后，就会去调用指定的ActivityThread的main方法就好。

![这里写图片描述](http://img.blog.csdn.net/20161009183252014)

最后总结一下这个过程，如果你不想关注太多的细节，只是提炼调用顺序的话，整个过程是这样的。

![这里写图片描述](http://img.blog.csdn.net/20161025165024179)

这篇文章大概就到这里了。

最开始的时候提到，这篇文章介绍的是从头启动一个程序的流程，所以到目前为止，我们可以看到，AMS侧的准备工作已经做好了也记录了Activity的所有信息，但是，Activity包括运行Activity的线程其实还没有正式启动。下一篇文章中我们会详细说明一个Application启动后的流程，也就是main方法中做了什么。

例行广告，大家喜欢我的文章，可以关注我的博客[zwgeek.com](zwgeek.com)

