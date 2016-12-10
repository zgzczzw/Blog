---
title: Activity的管理结构分析及源码解析
date: 2016-11-02 17:03:13
tags:
  - Android
  - Activity启动
  - 系统
categories: Android
---


例行广告，喜欢这篇文章的朋友可以关注我的博客[http://zwgeek.com](http://zwgeek.com)

之前几篇文章分析了Activity的启动流程，当时因为要抓启动的主线，所以中间涉及到一些类之间的关系都一笔带过了。后来再重新看前面文章的时候发现没有这部分的讲解，很影响理解，所以今天准备把这些详细拿出来讲一下。

没看过Activity启动流程分析的同学可以去看一下，因为这篇文章中会直接引用启动流程中已经说过的一些点。以下是传送门。

[Activity启动流程分析](http://zwgeek.com/2016/10/09/Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/)

[Activity启动流程分析(二)](http://zwgeek.com/2016/10/25/Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%902/)

[Activity启动流程番外篇](http://zwgeek.com/2016/10/26/Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E7%95%AA%E5%A4%96%E7%AF%87/)

这篇文章是带着这样一个问题来讲的，我们知道Android分为Server和Client两部分，那么Activity在这两部分中是怎么被组织被管理的。管理Activity的各部分组件又是什么时候生成的，并且在管理过程中起到了什么作用。

### Activity的管理结构

先放一张结果图

![这里写图片描述](http://img.blog.csdn.net/20161102165944689)

这就是整个Android中对Activity的管理结构，左边是Client端也就是APP部分，右边是Server端也就是Android System部分。

我们先来看Server端，在Server端跟Activity有关系的最大的类就是ActivityManagerService了，但是它更像是一个上帝类，而不是管理结构中的一部分，所以我就没有把它列在图上。

那么我们依次来看图上的部分
ActivityStackSupervisor，这个类像是一个工具类，会封装一些管理Activity的操作，但相信我，真正做事情的肯定不是它。

下面就是管理结构了，最外层是ActivityStack，直译过来是Activity栈，但却不是我们通常意义上理解的Activity栈，因为这个ActivityStack在Android中只有两个，HomeStack和FocusStack，跟Launcher有关的Activity都在HomeStack中，其他所有Activity都在FocusStack中。

然后ActivityStack中有TaskRecord，这个TaskRecord才是我们理解的Activity栈，一系列有关的Activity都在一个TaskRecord中，并且只有在一个TaskRecord中的Activity才能调用startActivityForResult，这个是前面提到的。这个TaskRecord跟两个参数息息相关。

- FLAG_ACTIVITY_NEW_TASK
- affinity

这两个参数熟悉开发的朋友应该都很熟悉吧。

然后在TaskRecord中有ActivityRecord，这个ActivityRecord在Server端就对应着一个Activity。是的，在Server端并没有真正的Activity实例，而只有Activity的token，而这个token就是存在ActivityRecord中的appToken，ActivityRecord还记录了一些其他的必要信息。

Token的类型是IApplicationToken.Stub，也是个Binder对象，服务端只存储Activity对应的Token。而真正的Activity实例是存储在Client端的。Server端就是通过token去Client端找到对应的Activity实例的。

接下来我们看Client端，在Client端也不是直接存储Activity的，因为还有一些Activity的信息要记录，所以Client端存的是Activity的包装类ActivityClientRecord，在ActivityClientRecord中包含有真正的Activity实例。

整个管理结构就是这样的，其实很简单。但是要从Android那繁杂错乱的源码中梳理出这层关系，还真是花了我好长时间。下面我们从源码中来验证这层管理关系。

#### 1. ActivityStackSupervisor

```java
/** The stack containing the launcher app. Assumed to always be attached to
     * Display.DEFAULT_DISPLAY. */
    private ActivityStack mHomeStack;

    /** The stack currently receiving input or launching the next activity. */
    private ActivityStack mFocusedStack;

    /** If this is the same as mFocusedStack then the activity on the top of the focused stack has
     * been resumed. If stacks are changing position this will hold the old stack until the new
     * stack becomes resumed after which it will be set to mFocusedStack. */
    private ActivityStack mLastFocusedStack;
```

ActivityStackSupervisor中就这三个ActivityStack，正如前面说的mHomeStack是存所有与Launcher有关的Activity。mFocusStack是存所有其他的Activity。mLastFocusedStack是一个备份用的，从注释可以看出，是用来备份mFocusStack的。

至于为什么这样做，当然为了用户体验了，用户可以随时回到桌面就是因为这种管理结构。

#### 2. ActivityStack

```java
/**
     * The back history of all previous (and possibly still
     * running) activities.  It contains #TaskRecord objects.
     */
    private ArrayList<TaskRecord> mTaskHistory = new ArrayList<TaskRecord>();
```

ActivityStack中存有一个TaskRecord的List，不同的TaskRecord就保存在这个List中。

另外，为了方便调用，AMS中这些类都持有它上一级管理者的引用，比如在ActivityStack代码中你就可以看到这样的引用。

```java
  /** Run all ActivityStacks through this */
    final ActivityStackSupervisor mStackSupervisor;
```

#### 3. TaskRecord

```java
/** List of all activities in the task arranged in history order */
    final ArrayList<ActivityRecord> mActivities;
```

TaskRecord中存的是一个ActivityRecord的List，这个List就是我们传统意义上理解的Activity栈，这个List中Activity的顺序也就是Activity在屏幕上被启动的顺序。

同样的，TaskRecord也持有它的上一级管理器ActivityStack的引用。

```java
/** Current stack */
    ActivityStack stack;
```

#### 4. ActivityRecord
在AMS这边的Activity，就是ActivityRecord了，它的变量中有真正Activity对应的token。

```java
final IApplicationToken.Stub appToken; // window manager token
```

同时，他也持有它父管理器的一些引用
```java
TaskRecord task;        // the task this is in.
```

```java
final ActivityManagerService service; // owner
```

以上就是Server端对Activity的管理结构，接下来我们看Client端的管理结构

#### 5. ActivityThread
Client这边最大的类是ActivityThread，我们知道，在ActivityThread中就管理着所有的Acitivity。

```java
final ArrayMap<IBinder, ActivityClientRecord> mActivities
            = new ArrayMap<IBinder, ActivityClientRecord>();
```

ActivityThread中有一个ActivityClientRecord的Map，Map的key就是Activity的token。

#### 6. ActivityClientRecord

在ActivityClientRecord中，存有真正的Activity实例和他对应的token。

```java
IBinder token;
Activity activity;
```

#### 7. 相互调用
接下来我们用一小段代码来说明一下Server端是怎么查找到Activity的。

```java
//调用onResume              next.app.thread.scheduleResumeActivity(next.appToken, next.app.repProcState,mService.isNextTransitionForward(), resumeAnimOptions);
```

这是在ActivityStack中的一段调用Resume的方法，看了前面的Activity启动流程分析，我们知道scheduleResumeActivity最终会被ActivityThread的handleResumeActivity处理。

handleResumeActivity又交给了performResumeActivity去处理这件事，我们看performResumeActivity的代码。

```java
 public final ActivityClientRecord performResumeActivity(IBinder token,
            boolean clearHide) {
        ActivityClientRecord r = mActivities.get(token);
        ...
        r.activity.performResume();
```

看，Client端会首先根据token从mActivities中找到ActivityClientRecord，然后取出AcitivityClientRecord中的Activity，去调用它的onResume方法。



### 管理结构的创建时间点

好的，已经讲完了Activity的管理结构，我们知道在Activity的启动过程中，这个管理结构的类会被依次创建，那么他们是在什么时间点被创建的呢？

前面分析Activity的时候我没有特别去讲解这个结构的创建，这里就补充一下这个点。我会说每个结构是在哪个方法中被创建的，但是方法是在什么时候调用的， 不清楚的同学可以去查阅Activity启动流程分析。

ActivityStackSupervisor因为很重要，所以在ActivityManagerService构造函数的时候就创建了。

```java
 public ActivityManagerService(Context systemContext) {
        ...
        mStackSupervisor = new ActivityStackSupervisor(this);
        ...
 }
```

ActivityRecord是在ActivityStackSupervisor的startActivityLocked中被创建的，如下图

![这里写图片描述](http://img.blog.csdn.net/20161102175329326)

然后ActivityStack和TaskRecord在同个类的startActivityUncheckedlocked方法中被创建，如下图。

![这里写图片描述](http://img.blog.csdn.net/20161102175501924)

adjustStackFocus中会创建ActivityStack，但是正如前面说的，只有当mFocusStack没有被创建的时候才会重新创建，如果mFocusStack已经有啦，那么，就用mFocusStack。

至此，Server端这边的结构都被创建出来了，然后再看Client端的结构。

ActivityClientRecord是在ActivityThread的scheduleLaunchActivity，如图

![这里写图片描述](http://img.blog.csdn.net/20161102175850425)

注意，这个时候Activity实例还没有被创建出来。而Activity的创建是在什么点呢。

![这里写图片描述](http://img.blog.csdn.net/20161102180002769)


在performLaunchActivity中，并且创建完了之后把自己加入到ActivityClientRecord中，然后把ActivityCLientRecord加入到mActivities中。

至此，所有的管理结构都创建完成，Activity也基本启动完成了。

例行广告，喜欢这篇文章的朋友可以关注我的博客[http://zwgeek.com](http://zwgeek.com)




