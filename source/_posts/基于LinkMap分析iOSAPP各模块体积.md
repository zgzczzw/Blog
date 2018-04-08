---
title: 基于LinkMap分析iOSAPP各模块体积
date: 2018-01-08 16:59:22
tags:
  - iOS
  - LinkMap
  - Python
categories: iOS
---


广告时间，大家喜欢我的文章，可以关注我的博客[zwgeek.com](zwgeek.com)

## 1. 前言

做客户端开发经常会有需要分析客户端体积的需求。比如引入了一个第三方库，这个库到底多大呢？同时，有些动态库封装了所有架构（比如x86_64,arm）的代码，但编译的时候实际打到安装包里的只有当前架构的那部分，那么这部分体积是多少呢？有时候一个模块写了很多方法，但是这些方法都没有被调用到，编译的时候实际打进安装包里的代码又有多少呢？只有真正了解了自己的安装包体积是有哪些部分构成的，才能有针对性的去优化体积。像Android的话，简单一点可以解压开apk文件去看每个模块，或者引入的Library的大小。但是iOS的安装包是个二进制文件，这又怎么去分析呢。这篇文章给大家讲下使用Xcode提供的LinkMap文件去分析iOS安装包的体积构成，同时提供一个python的脚本，去自动化的分析iOS安装包体积。

关于LinkMap的分析，也是在Bang神的指点下才知道的，所以特别鸣谢下Bang的文章[传送门：iOS APP可执行文件的组成](http://blog.cnbang.net/tech/2296/)，这篇文章跟Bang的讲的都差不多，我就是记一下自己的理解。

## 2. LinkMap详解
LinkMap，顾名思义，指的就是iOS安装包的一张地图，通过这张地图，你可以看到安装包里各个部分都是什么内容。通过分析各部分内容所占的内存空间，就可以知道各部分内容的体积大小了。

### 2.1 生成LinkMap
首先，我们要生成LinkMap，这是Xcode提供的功能，默认是不生成的，需要更改一下配置，才会去生成。需要更改的地方有两个。

在Target的Build Settings中更改Write Link Map File 为 Yes，这样就可以生成Link Map文件了，但是这个文件在哪呢。通过修改Build Settings的Path To Link Map可以指定LinkMap文件的生成目录，默认是生成在Build文件加下，也可以像我这样指定直接生成在桌面上。

![这里写图片描述](https://img-blog.csdn.net/2018040817355259?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pnemN6enc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

![这里写图片描述](https://img-blog.csdn.net/20180408173810292?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pnemN6enc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

修改完以上两个配置后，就可以重新Build程序了，没有什么意外的话，你指定的路径下应该已经有LinkMap文件了。接下来我们看看LinkMap文件的格式。

### 2.2 LinkMap文件格式
我这边的demo是个比较简单的Hello world程序，所以生成的LinkMap很小，不过稍微复杂一点的程序，LinkMap就会很大了。下面说下文件格式。

```
# Path
# Arch: x86_64
# Object files:
# Sections:
# Symbols:
# Dead Stripped Symbols:

```

一个完整的LinkMap文件是分为这几块的，以#为分隔，我们一个个看下。

#### Path

```
# Path: /tmp/xcode/TestCleanPackage-gjpgqvilgwaxwyfhkccvcfzwbtdf/Build/Products/Debug-iphonesimulator/TestCleanPackage.app/TestCleanPackage
```
Path记录的是这个LinkMap对应的安装包的地址。

#### Arch
```
# Arch: x86_64
```
Arch指的是这个LinkMap对应的架构

#### Object files
```
# Object files:
[  0] linker synthesized
[  1] /tmp/xcode/TestCleanPackage-gjpgqvilgwaxwyfhkccvcfzwbtdf/Build/Intermediates.noindex/TestCleanPackage.build/Debug-iphonesimulator/TestCleanPackage.build/TestCleanPackage.app.xcent
[  2] /tmp/xcode/TestCleanPackage-gjpgqvilgwaxwyfhkccvcfzwbtdf/Build/Intermediates.noindex/TestCleanPackage.build/Debug-iphonesimulator/TestCleanPackage.build/Objects-normal/x86_64/ViewController.o
[  3] /tmp/xcode/TestCleanPackage-gjpgqvilgwaxwyfhkccvcfzwbtdf/Build/Intermediates.noindex/TestCleanPackage.build/Debug-iphonesimulator/TestCleanPackage.build/Objects-normal/x86_64/main.o
[  4] /tmp/xcode/TestCleanPackage-gjpgqvilgwaxwyfhkccvcfzwbtdf/Build/Intermediates.noindex/TestCleanPackage.build/Debug-iphonesimulator/TestCleanPackage.build/Objects-normal/x86_64/UnUsedClass.o
[  5] /tmp/xcode/TestCleanPackage-gjpgqvilgwaxwyfhkccvcfzwbtdf/Build/Intermediates.noindex/TestCleanPackage.build/Debug-iphonesimulator/TestCleanPackage.build/Objects-normal/x86_64/AppDelegate.o
[  6] /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator11.1.sdk/System/Library/Frameworks//Foundation.framework/Foundation.tbd
[  7] /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator11.1.sdk/usr/lib/libobjc.tbd
[  8] /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator11.1.sdk/System/Library/Frameworks//UIKit.framework/UIKit.tbd
```

Object files是编译后生成的文件列表，比如这个程序class都编译成了.o文件，像我们比较熟悉的AppDelegate.o文件等等。还有引进来的几个库，比如UIKit.tbd。

##### Sections
```
# Sections:
# Address	Size    	Segment	Section
0x100001620	0x000003C3	__TEXT	__text
0x1000019E4	0x00000036	__TEXT	__stubs
0x100001A1C	0x0000006A	__TEXT	__stub_helper
0x100001A86	0x00000A69	__TEXT	__objc_methname
0x1000024EF	0x00000048	__TEXT	__objc_classname
0x100002537	0x0000086D	__TEXT	__objc_methtype
0x100002DA4	0x0000007A	__TEXT	__cstring
0x100002E1E	0x0000018C	__TEXT	__entitlements
0x100002FAC	0x00000048	__TEXT	__unwind_info
0x100003000	0x00000010	__DATA	__nl_symbol_ptr
0x100003010	0x00000048	__DATA	__la_symbol_ptr
0x100003058	0x00000018	__DATA	__objc_classlist
0x100003070	0x00000010	__DATA	__objc_protolist
0x100003080	0x00000008	__DATA	__objc_imageinfo
0x100003088	0x00000CD8	__DATA	__objc_const
0x100003D60	0x00000020	__DATA	__objc_selrefs
0x100003D80	0x00000008	__DATA	__objc_classrefs
0x100003D88	0x00000008	__DATA	__objc_superrefs
0x100003D90	0x00000008	__DATA	__objc_ivar
0x100003D98	0x000000F0	__DATA	__objc_data
0x100003E88	0x000000C0	__DATA	__data
```

Section是各种数据类型所在的内存空间，Section主要分为两大类，\_\_Text和\_\_DATA。\_\_Text指的是程序代码，\_\_DATA指的是已经初始化的变量等。
具体分类如下表所示。

>以下是__TEXT段的section

>__text  主程序代码

>__stubs 和__stub_helper   用于动态链接库的stub

>__cstring    c语言字符串

>__const    const修饰的常量

>__objc_methname    objc的方法名称

>__objc_methtype    objc方法类型

>__objc_classname    objc类方法

 

>以下是__DATA段的section

>__objc_ivars   objc类的实例变量

>__objc_classlist    objc类列表

>__objc_protolist    objc协议列表

>__objc_imageinfo    objc镜像信息

>__objc_const    objc常量

>__objc_selfrefs    objc自引用(self)

>__objc_protorefs    objc协议引用

>__objc_superrefs    objc超类引用

>__cfstring   使用Core Foundation字符串

>__bss   BSS

每个Section前面的两个16进制的数字代表的就是这个Section相对于安装包初始内存的偏移和这个Section的大小。比如：

```
0x100001620	0x000003C3	__TEXT	__text
```

\_\_text这个Section的偏移地址是0x100001620，这块的大小是0x000003C3，也就是963个字节。

#### Symbols
iOS开发的同学对Symbols这个单词肯定不陌生，什么Crash要有对应的符号表，编译的时候经常保持找不到Symbols等。Symbols简单来说就是类名，变量名，方法名等等符号。所以这一块也详细列出了这个安装包内各个方法所占的内存大小。

```
# Address	Size    	File  Name
0x100001620	0x00000040	[  2] -[ViewController viewDidLoad]
0x100001660	0x00000040	[  2] -[ViewController didReceiveMemoryWarning]
0x1000016A0	0x00000010	[  2] -[ViewController unusedMethod1]
...
```

这一块太多了，我只列出一小部分，这块同样有四列，一二列和Sections的情况一样，分别是偏移地址和大小。第四列是方法的符号，类名+方法名。第三列是文件序号，这个序号是哪里来的的，就是前面提到的Object files里文件的序号，比如这里viewDidLoad的序号是2，去Object files去找序号是2的文件。

```
[  2] /tmp/xcode/TestCleanPackage-gjpgqvilgwaxwyfhkccvcfzwbtdf/Build/Intermediates.noindex/TestCleanPackage.build/Debug-iphonesimulator/TestCleanPackage.build/Objects-normal/x86_64/ViewController.o
```

也就是说这个方法来自ViewController.o这个文件。

通过这种对应关系，我可以知道一个.o文件里有多少方法被编译进了安装包，每个方法所占的体积，加起来我就知道每个.o文件的大小了。后面的程序也就是把这个过程给自动化了。

#### Dead Stripped Symbols
最后还有一部分是没用的符号，这部分我也不知道是怎么产生的，但可以肯定的是这部分不应该太大。

```
# Dead Stripped Symbols:
#        	Size    	File  Name
<<dead>> 	0x00000018	[  2] CIE
<<dead>> 	0x00000018	[  3] CIE
<<dead>> 	0x00000006	[  5] literal string: class
<<dead>> 	0x00000008	[  5] literal string: v16@0:8
<<dead>> 	0x00000018	[  5] CIE
```

## 3.自动分析
通过前面对LinkMap文件格式的解析，我们知道在LinkMap里，我们可以知道每个文件所占的体积大小，并且可以通过文件的前缀，知道文件所属的动态库，这样也就可以知道动态库的大小。只是这个过程太过繁琐，所以我们去把它自动化了。

项目开源在：

[LinkMapParser](https://github.com/zgzczzw/LinkMapParser)

#### 使用方法
##### 3.1 安装Python环境
LinkMapParser是一个Python脚本，运行该脚本需要开发者的机器有Python的运行环境，安装Python的方法可以查阅相关资料。Python版本为2.7。

##### 3.2 生成link map文件
Xcode默认是不生成link map文件的。生成link map文件需修改项目中的Build Settings，选择Target的Build Settings，修改Write Link Map File为Yes，修改Path to Link Map File为你需要的地址，然后编译程序，即可在该地址生成相应的link map文件。

##### 3.3 运行工具
该工具支持分析一个link map文件和比较两个link map文件，运行的命令分别为：

###### 分析一个link map文件

```
python parselinkmap.py $map_link_file_path
```
输出结果类似于：

```
================================================================================
        demoData/TestCleanPackage-LinkMap-normal-x86_64.txt各模块体积汇总
================================================================================
Creating Result File : demoData/BaseLinkMapResult.txt
AppDelegate.o                                     0.01M
ViewController.o                                  0.00M
TestCleanPackage.app.xcent                        0.00M
UnUsedClass.o                                     0.00M
main.o                                            0.00M
libobjc.tbd                                       0.00M
linker synthesized                                0.00M
Foundation.tbd                                    0.00M
UIKit.tbd                                         0.00M
总体积:                                           0.01M
```

demo中只有一个Bundle，可以看出各个class文件在安装包中所占大小，如AppDelegate占用0.01M。

比较两个link map文件

```
python parselinkmap.py $base_map_link_file_path $target_map_link_file_path
```

LinkMapParser会分析两个map link文件，然后比较各个模块的体积是否有变化，最后列出体积变大的模块。

输出结果类似于：

```
================================================================================
                     demoData/BaseLinkMap.txt各模块体积汇总
================================================================================
Creating Result File : demoData/BaseLinkMapResult.txt
AppDelegate.o                                     0.01M
ViewController.o                                  0.00M
TestCleanPackage.app.xcent                        0.00M
UnUsedClass.o                                     0.00M
main.o                                            0.00M
libobjc.tbd                                       0.00M
linker synthesized                                0.00M
Foundation.tbd                                    0.00M
UIKit.tbd                                         0.00M
总体积:                                           0.01M






================================================================================
                    demoData/TargetLinkMap.txt各模块体积汇总
================================================================================
Creating Result File : demoData/TargetLinkMapResult.txt
AppDelegate.o                                     0.64M
ViewController.o                                  0.00M
TestCleanPackage.app.xcent                        0.00M
UnUsedClass.o                                     0.00M
main.o                                            0.00M
libobjc.tbd                                       0.00M
linker synthesized                                0.00M
Foundation.tbd                                    0.00M
UIKit.tbd                                         0.00M
总体积:                                           0.64M






================================================================================
                                    比较结果
================================================================================
模块名称                                          基线大小  目标大小  是否新模块
AppDelegate.o                                     0.01M     0.64M
```

好的，LinkMap就介绍到这里。

广告时间，大家喜欢我的文章，可以关注我的博客[zwgeek.com](zwgeek.com)


