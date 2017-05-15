---
title: Proguard指南
date: 2017-05-09 12:01:35
tags: 
  - Android
  - Proguard
  - 混淆
  - 译文
categories: Android
 
---

之前在使用fastjson的时候遇到一些坑，这些坑中有一个和混淆选项有关，后来发现了Proguard其实是有一个官网的，里面介绍了各种情况。而我们平时开发可能就是单纯的从网上拷贝一个最佳实践的Proguard配置文件，而完全不在意各种配置项是什么意思，所以我想利用空闲时间翻译一下这个Proguard指南。以后配置Proguard的时候心里也会有底。

Proguard的官网是[Proguard Manual](https://www.guardsquare.com/en/proguard/manual/introduction)


## 简介
ProGuard是一个集Java类文件压缩，优化，混淆和预校验的工具。压缩指的是发现并删除无用的类，字段，方法和属性。优化指的是分析并优化字节码数据。混淆指的是把固定的类名，字段名和方法名用更短的没有意义的名字代替。前面这些步骤让代码量更小，运行更快，并且不容易被反编译。预校验步骤能够提前在class里加入Java6+及Java ME需要的预校验信息，来达到加快代码校验速度的目的。

当然，上面所说的这四步都是可选的。例如，开发者可以只用ProGuard来列出程序中的无用代码，或者可以只做预校验来让代码可以在Java6+上运行的更快。

![这里写图片描述](https://www.guardsquare.com/files/media/guardsquare2016/Website/ProGuard/ProGuard_build_process.png)

 上面的图展示了ProGuard压缩，优化，混淆和预校验Java Code的过程。

ProGuard首先会读取Jar文件（或者aar，war，ear，zip，apk，目录等）。然后压缩，优化，混淆，预校验这些文件。开发者也可以选择让ProGuard循环优化一个文件，也就是优化一个文件多次。ProGuard会把优化的结果输出到一个或多个Jar文件（或者aar，war，ear，zip，apk，目录等）中，同时开发者可以选择让ProGuard把输入文件中包含的源代码的类名或类内容用混淆名称代替。

需要注意的是，ProGuard只能接收Library文件（也就是jar，aar，war，ear，zip，apk，目录等）作为输入文件，所以开发者需要先编译代码变成Library文件。ProGuard会重新处理Library文件中的类依赖关系，但是Library本身不会变，处理完成后，你还要把这些文件放在你最好的应用程序中替换原来的Library文件


### 程序入口

入口，也可以被称作入口函数或入口类，在Proguard中是一个很重要的概念。为了确定哪些代码应该被保护，哪些代码可以被删减或混淆，开发者必须指定一个或多个入口。入口通常是有Main方法的类，Java应用程序的面板，或Android的Acitivity等。

- 压缩时，Proguard会从入口处循环遍历来确认哪些类和成员变量被用过。没有用过的类和成员变量将被删除。

- 优化时，Proguard会优化上一步产生的代码，和其他优化一样，不是入口的类和方法可以被修改成private，static或final的，无用的参数可以被移除，一些方法也可能会被内联化。

- 混淆时，Proguard会重命名哪些非入口的类和类的成员变量。同时，会保证入口函数仍然可以以他们原来的名字被访问到。

- 入口点在预校验步骤不是必要的。

程序入口可以用keep命令来定义，keep命令在用法简介一章会介绍，并且在样例代码一章会给出一些示例。

### 反射

在大多数的代码自动化处理中，反射总是会出一些或多或少的问题。在ProGuard中，如果你的类或者类的成员变量是被动态创建和调用的，也就是说， 根据类名或方法名被调用，那么这些类和成员变量必须被指定为入口。举个例子，Class.forName() 在运行期可构造任意的类，所以以此来推断哪个类应该被保护几乎不可能，而且这些类名还有可能是从配置文件中读出来的。所以开发者应该在ProGuard的配置文件中用简单的keep选项来指明这些类。

但是，现在ProGuard已经能够预测并处理以下这些情况了。

- Class.forName("SomeClass")
- SomeClass.class
- SomeClass.class.getField("someField")
- SomeClass.class.getDeclaredField("someField")
- SomeClass.class.getMethod("someMethod", new Class[] {})
- SomeClass.class.getMethod("someMethod", new Class[] { A.class })
- SomeClass.class.getMethod("someMethod", new Class[] { A.class, B.class })
- SomeClass.class.getDeclaredMethod("someMethod", new Class[] {})
- SomeClass.class.getDeclaredMethod("someMethod", new Class[] { A.class })
- SomeClass.class.getDeclaredMethod("someMethod", new Class[] { A.class, B.class })
- AtomicIntegerFieldUpdater.newUpdater(SomeClass.class, "someField")
- AtomicLongFieldUpdater.newUpdater(SomeClass.class, "someField")
- AtomicReferenceFieldUpdater.newUpdater(SomeClass.class, SomeType.class, "someField")

当然，上面示例中的类和变量名不是固定的，但是构造方法应该和上面样例中展示的一样，以便ProGuard能够识别出他们。这样的话，相关的类和变量就可以在压缩的时候被识别出来，然后被保护。

未来，ProGuard希望提供一些建议如果需要开发者手动keep一些类或成员变量。例如，ProGuard会注意下面这种情况：
```
(SomeClass)Class.forName(variable).newInstance()
```
在这种情况下，可能代表SomeClass这个类或接口的实现类需要被保护。给出这样的提示后，开发者可以根据情况来编辑配置文件。

为了更好的处理提示结果，开发者需要对自己处理的代码比较熟悉。因为混淆代码可能会产生一大堆的提示让开发者去判断类是否需要被保护。如果开发者对代码的内部实现不熟悉，可能会引发一些异常。
