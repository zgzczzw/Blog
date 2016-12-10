---
title: Activity启动流程分析（二）
date: 2016-10-25 17:03:13
tags:
  - Android
  - Activity启动
  - 系统
categories: Android
---

广告时间，大家喜欢我的文章，可以关注我的博客[http://zwgeek.com](http://zwgeek.com)

前面说到，希望分析一下Activity的启动流程，整个过程准备分为三篇文章来写
- 程序调用startActivity后发生的操作
- 如果被startActivity的程序是需要启动的程序，程序在最开始初始化时发生的操作。例如在Launcher中启动一个程序。
- 如果被startActivity的程序是已经启动的程序，发生的操作。例如程序自己调用startActivity启动一个自己程序中的Activity

第一篇文章也已经讲完了程序调用startActivity之后发生的事情。

还没有读过的同学可以看这里[Activity启动流程分析](http://zwgeek.com/2016/10/09/Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/)

这篇文章就来讲第二个部分，一个程序被启动之后发生的事情。第一篇文章中还提到了几个问题，也会在这篇文章中做出解答。

- getApplication() ，getApplicationContext()，getBaseContext()分别有什么区别？
- 单个进程中能存在多个application么？
- 为什么不能使用service或者application作为创建dialog的context参数？

第一篇文章最后讲到了Process.start方法，说到了这个方法会启动一个线程，并且运行ActivityThread的main方法来正式开启一个Android程序。所以，很显然，这篇文章就要从ActivityThread的main方法来开始。

首先，来看一下这个Android程序的起点方法，一睹芳容。

![这里写图片描述](http://img.blog.csdn.net/20161025184513316)

这个方法重要的是在Looper prepare之后的部分，Handler跟loop的机制比较简单，可以先百度一下，我后面可能也会写篇文章说一下。简单来说loop就是一个无限循环，通过循环去去查询有没有handler发过来命令，如果有就处理，没有就继续循环。

这样的话，我们就能看出，主进程在做了一个初始化工作之后就把自己放在了一个loop循环中，要跟这个程序打交道，怎么办呢，就是通过获取它的handler，然后发命令，比如现在需要调用onResume方法，通过handler告诉主进程looper要调用onResume，looper就会做相应的处理了。当然这个是关于Android生命周期方法的调用问题，我们也是要单独拉出来讲的，这里就不细说了。

其实这篇文章的重点是这一句话，thread.attach(false)，ActivityThread的初始化操作。来看下这个方法具体做了什么操作。

![这里写图片描述](http://img.blog.csdn.net/20161025191248464)

这个方法看下来还好，也都是配置一些监听器，像ViewRoot监听，内存监听，等等。重要的还是我选中的这一段，首先创建了一个ApplicationThread，然后把这个ApplicationThread交给了RuntimeInit，很多人开发过程中最头痛的就是RuntimeException，其实这里就是异常监控的初始化过程。然后创建了一个ActivityManagerNative，第一篇文章中就提过，ActivityManagerNative在创建的时候就会和ActivityManagerService绑定。接下来程序就可以通过AMN来访问AMS了。可是大家有想过，Client可以通过AMN来访问AMS，但是Server端怎么访问Client端呢，看这句话attachApplication，其实这里就是程序把自己的一个控制器交给了Server端，然后Server端就可以通过这个控制器来操作Client端了。不信我们来看下ApplicationThread的方法。

![这里写图片描述](http://img.blog.csdn.net/20161026103756562)

是不是看到很多控制生命周期的方法，是的，AMS就是通过这个ApplicationThread来控制Client端的。

那么我们来总结一下这里，程序通过AMN来绑定AMS后，自己创建了一个桥梁applicationThread，然后把这个桥梁交给AMS，意思就是说，这是我小弟，以后联系我可以通过他。另一方面，在这个attach中client的各种初始化已经完成了。接下来的工作就通过attachApplication这个方法移交给AMS端了。

那接下来我们来看AMS端的attachApplication方法

![这里写图片描述](http://img.blog.csdn.net/20161026104634204)

AMS端先通过Binder查询到程序的pid，然后调用attachApplicationLocked，继续往下看，这个方法就是AMS在接到“一个新进程启动了”这件事之后做了一些工作，很复杂，但AMS毕竟是老板，多做一些是正常的。

![这里写图片描述](http://img.blog.csdn.net/20161026110532870)

![这里写图片描述](http://img.blog.csdn.net/20161026110642120)

先是用一个ProcessRecord来记录所有和Process有关的信息。

![这里写图片描述](http://img.blog.csdn.net/20161026110745261)

这里有一个generateProvider的操作，但是我感觉应该不是生成程序的ContentProvider，因为此时Manifest文件还没有被解析，这里应该是为Application生成一些系统必要的ContentProvider。至于对不对，后面再验证吧。

后面做了一些配置工作后，调用了这个方法

![这里写图片描述](http://img.blog.csdn.net/20161026111207622)

是的，这个方法才是主线剧情。通过这个方法，AMS将自己初始化的一些成果，告诉了Client端，并将控制权重新交回给Client端。在看bindApplication之前，我们看下AMS后面的工作。

![这里写图片描述](http://img.blog.csdn.net/20161026111410272)

源码的注释写的比较清楚了，判断有没有其他组件在等这个进程启动，如果有，那么这个进程已经启动了，就该通知他们做事了。这不关我们的事，回到正题吧，看看控制权回到Client那边后，又做了什么。不过这里我们也再一次验证了AMS通过ApplicationThread这个类来和Client端打交道。

好，老板做完事情了，工作又回到小弟手中了。什么？你说老板其实什么都没做，你可以去财务领工资了。

但是ApplicationThread毕竟是个桥梁，实际的工作还是得给app的老大ActivityThread来做，所以这个bindApplication方法也是记录了一些AMS传回来的信息之后，又把工作给了ActivityThread。

我们来看下这个bindApplication做了什么，首先记录了一些配置信息

![这里写图片描述](http://img.blog.csdn.net/20161026112941262)

然后在VM中注册APP的信息

![这里写图片描述](http://img.blog.csdn.net/20161026113020966)

这里注释也说了，有两种情况，两个package是共享runtime的。

- 设置了shareUserId
- 设置了ProcessName

在share的情况下是不用再VM中注册的，我的理解是，share的组件并不是一个完整的app，而他所属的原来的app其实已经注册过了。

这里有个问题是，工作是怎么给ActivityThread的呢？看bindApplication的第三部分

![这里写图片描述](http://img.blog.csdn.net/20161026113436006)

ApplicationThread是ActivityThread的内部类，内部类代表什么呢，它其实是持有一个外部类ActivityThread的对象引用的。可以这么说在ApplicationThread中其实是可以调到ActivityThread的所有方法的。那么它为什么要用这种sendMessage，然后通过Handler处理的这种方式呢。我们来想一下Handler的一个作用是什么。切换线程，在任何情况下，不管bindApplication这个方法运行在哪个线程中，只要通过handler这种方式，都可以回到ActivityThread所在的线程，也就是主线程。这就保证了什么呢，保证了Android的所有生命周期方法都是运行在主线程的，也就是我们常说的UI线程。

简单跟下sendMessage

![这里写图片描述](http://img.blog.csdn.net/20161026125950474)

![这里写图片描述](http://img.blog.csdn.net/20161026130023435)

![这里写图片描述](http://img.blog.csdn.net/20161026130056888)

mH是一个H类型的对象，这个H就是ActivityThread内部的Handler

![这里写图片描述](http://img.blog.csdn.net/20161026130144586)

看下这个Handler的handleMessage方法，跟下对BIND_APPLICATION消息的处理。

![这里写图片描述](http://img.blog.csdn.net/20161026130314369)

进到了handleBIndApplication方法，这里就是Application的启动过程了，让我们来仔细看下这个方法处理的步骤

![这里写图片描述](http://img.blog.csdn.net/20161026133403057)

国际惯例，前面也是各种记录，配置，初始化的工作，我们可以完全忽略。

![这里写图片描述](http://img.blog.csdn.net/20161026133429604)

这个地方是一个小的知识点，在3.1以前的版本上，AsyncTask会改变默认的executor，我们看下改变之后的executor是什么样的。

![这里写图片描述](http://img.blog.csdn.net/20161026133859665)

而默认的executor是这样的

![这里写图片描述](http://img.blog.csdn.net/20161026133929525)

所以3.1以前版本的AsyncTask是并行执行任务的，而3.1以后版本反而是顺序执行任务的，当然，这个配置可以通过AsyncTask的参数而改变。

然后我们回到handleBindApplication，后面会继续设置时区，位置，屏幕参数等。

![这里写图片描述](http://img.blog.csdn.net/20161026134148091)

之后创建了一个Context对象，注意，这是我们到目前位置接触到的第一个Context。我们知道在Android中Application，Activity等等其实都是Context的子类，但是他们又是不同的。这里创建的这个context对应的是我们开发过程中的哪个呢，让我们继续往下看。

![这里写图片描述](http://img.blog.csdn.net/20161026134554910)

然后又是一堆配置，其中包括UI线程不能执行网络操作的配置

![这里写图片描述](http://img.blog.csdn.net/20161026134728344)

然后是关于调试的相关配置，开启一个调试端口，其实关于调试也是需要讲很多的，调试本身也是C/S结构的，客户端开一个端口，等着服务端来连接进行调试，这里就是客户端打开端口的操作。

![这里写图片描述](http://img.blog.csdn.net/20161026134924665)

设置了一个默认的HTTP代理

![这里写图片描述](http://img.blog.csdn.net/20161026135017838)

后面一段是创建了一个Instrumentation对象，这里不截图的。之前第一篇文章我们也提到过，跟ActivityStart有关的操作都是由Instrumentation这个类管理的，被我们亲切的称为大管家，其实是为了监视我们的操作。。。

然后，这个方法讲了这么多，前面大家基本可以忽略，到现在才是重点。

![这里写图片描述](http://img.blog.csdn.net/20161026135315421)

这一段，首先创建了一个Application，恩，我们开发过程中遇见的Application就是这里生成的。后面一段是初始化ContentProvider，这个我们后面讲ContentProvider启动过程的时候会看到，不过这里能知道的一个点就是ContentProvider的启动时间是相当早的，在Application的onCreate之前。然后这里的providers确实是之前我们说到的AMS生成的，然后一路传过来的。恩，先不细看了，因为这篇文章主要想说Activity的启动过程。

后面调用了Instrumentation的onCreate方法，是个空方法，可Override，再后面看到吗，通过Instrumentation大管家呼叫了Application的OnCreate方法

![这里写图片描述](http://img.blog.csdn.net/20161026135758475)

至此，我们开发者接触到的Android生命周期中的第一个方法，Application的onCreate被执行了。至于前面生成的Context，我看了一下，传给了Instrumentation成为了Instrumentation中的appContext，但是我并没有找到跟Application对象结合的方法。这个继续往后看吧。

到此为止，这个handleBindApplication方法就结束了，创建了一个appContext，一个Instrumentation，一个Application，并且调用了Application的onCreate。中间还涉及到ContentProvider的初始化操作，我们先忽略。那么Activity在哪里，为什么感觉自己被带偏了。我又一路往前找。终于在AMS的attachApplicationLocked中，我看到了这一步。

![这里写图片描述](http://img.blog.csdn.net/20161026141040256)

bindApplication在执行完我们上面说的那一堆之后，调用了StackSupervisor的attachApplicationLocked，好，我们来看一下。同时，这里的调用顺序也保证了Application的onCreate方法在Activity之前进行。

![这里写图片描述](http://img.blog.csdn.net/20161026141633514)

方法中有一个realStartActivity的方法，名字很形象，前面我们调用过那么多次的startActivity，但是真正的Activity在这里才生成。

这个realStartActivity嘟噜嘟噜的扯了好多，不知道在干吗，但是终于看到了一个熟悉的影子

![这里写图片描述](http://img.blog.csdn.net/20161026150656396)

就这样控制权又回到了ActivityThread

![这里写图片描述](http://img.blog.csdn.net/20161026151007131)

这里先创建了一个ActivityClientRecord，这个就是Client端管理生成的activity对象的包装类，后面生成的Activity类都会被ActivityClientRecord包装一层，然后保存到ActivityThread的mActivities中。

![这里写图片描述](http://img.blog.csdn.net/20161026151225913)

跟Application那边一样，scheduleLaunchActivity最终会被handleLaunchActivity处理，我们略过中间过程，直接看handleLaunchActivity吧。

![这里写图片描述](http://img.blog.csdn.net/20161026151527821)

这个方法触发了Activity的两个生命周期方法，分别是我标出来的onCreate的onResume，然后后面那一段我的理解是Activity被创建出来，并且调用了onResume之后并没有被显示，那么就立刻调用onPause，但其实不是很懂这个地方。让我想想再回来补充吧。

接下来看performLaunchActivity吧

![这里写图片描述](http://img.blog.csdn.net/20161026152020367)

首先更新了ActivityClientRecord的信息，包括ActivityInfo，ComponentName等，我们开发过程中也是经常用到，这些信息都是存在ActivityClientRecord中的。

![这里写图片描述](http://img.blog.csdn.net/20161026152153386)

接下来创建了一个Activity，天哪，我们分析了这么久，终于看到Activity了。创建过程很简单，通过反射new了一个类出来，这个时候的Activity是还没有生命周期的。需要把Activity托管给AMS，才能有生命周期。

![这里写图片描述](http://img.blog.csdn.net/20161026152429639)

接着，我们获取到之前创建的那个Application，为Activity创建了一个Context，然后通过Activity的attach方法把这些绑定起来。

![这里写图片描述](http://img.blog.csdn.net/20161026152636325)

生成Context的方法和之前为Instrumentation生成Context的方法差不多，返回的是一个ContextImpl类型的对象，保存了Activity的上下文。

![这里写图片描述](http://img.blog.csdn.net/20161026152944396)

attach方法将所有的对象包括Instrumentation，Application, ActivityThread等等全部在Activity中保存了一份。

![这里写图片描述](http://img.blog.csdn.net/20161026153551690)

回到performLaunchActivity，attach之后通过Instrumentation大管家调用了Activity的onCreate方法

![这里写图片描述](http://img.blog.csdn.net/20161026154101442)

然后将生成的activity交给ActivityClientRecord，并保存在mActivities中，这就完成了Activity的生成，并托管给系统，之后系统都可以在适当的时候通过token来获取到相应的Activity，并调用其生命周期。

这样performLaunchActivity就结束了，我们返回上一层handleLaunchActivity继续往下看，Activity在生成之后是会立刻调用onResume的。这两个生命周期有什么区别呢， 其实就在于onCreate跟onResume之间执行的这几句话，说实话，在创建的时候区别不大。不同的是onResume未来还会被调用，但是onCreate只有创建的时候才会被调用。

![这里写图片描述](http://img.blog.csdn.net/20161026154453198)

其实到这来一个Activity的启动流程就已经结束了，但是我们顺便来看下handleResumeActivity的工作吧

![这里写图片描述](http://img.blog.csdn.net/20161026154706405)

看这来，从mActivities中根据token获取了ActivityClientRecord，并进一步获得了里面的activity，然后执行了onResume方法，我刚想说，咦，这次调用没有通过大管家哎，然后看了一下performResume方法里面，其实还是通过Instrumentation调用的。

![这里写图片描述](http://img.blog.csdn.net/20161026154913320)

然后，还更新了ActivityClientRecord的相关信息等。

其实到这里onResume已经调用完成了，那么handleResumeActivity后面的这一堆在干什么呢？

![这里写图片描述](http://img.blog.csdn.net/20161026155148379)

通过方法名我们知道，Activity在onResume之后才开始处理显示的逻辑，这里就是通知AMS，Activity onResume已经调用完了，接下来要显示了，那么AMS就会通知WindowManger来显示Activity，这就是另外一件事了，我们在这里就不细细讨论了。

呼~终于写完了，整个流程主线还是很清楚的，AMS和AMN的分工明确。 

整个流程总结一下，是下面这种关系

![这里写图片描述](http://img.blog.csdn.net/20161026165138600)

最后还是广告时间，如果喜欢这篇文章，可以关注我的博客[http://zwgeek.com](http://zwgeek.com)


