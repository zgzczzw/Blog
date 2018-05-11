---
title: [编辑中...]Tensorflow入门2-使用CNN网络识别股票自选股图片
date: 2018-05-04 07:57:10
tags:
---

## 前言

上一篇文章讲了CNN网络工作原理和Tensorflow的一个解决MNIST问题的例子，[Tensorflow入门1-CNN网络及MNIST例子讲解](http://zwgeek.com/2018/04/08/Tensorflow%E5%85%A5%E9%97%A8/)。这篇文章会讲一下如果在MNIST问题的例子上改造成适合你自己图片分类业务的代码。

## 1. 业务解释

简单描述一下我的需求，类似于微信的截图分享，用户截图后，判断用户截图是否为自选股图片，如果是，则弹出提示询问用户是否要把这张图片用于导入自选股。可以减少用户的操作路径，让用户一键操作。

自选股图片指的是截图上有股票信息，比如下面这张图。

![这里写图片描述](https://img-blog.csdn.net/2018041114410466?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pnemN6enc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

代码依然在之前MNIST问题的例子基础上改造，这篇文章只讲需要改造的地方。

## 2. 准备数据

### 准备图片数据
MNIST问题是有现成的训练集和测试集，而自己的业务当然要自己准备这些数据，这个过程有个高大上的名字叫特征工程，找到需要的图片数据及标签就是这个过程的目的。

在这个例子中数据就是我的手机截图，因为这个业务如果上线后，预测的图片也只是手机截屏，而没有杂七杂八的图片，所以准备数据的范围就很小了。在准备的手机截图中，需要有一部分是股票自选股图片，其他的随意，但最好有各种各样的截屏图片。以下我拿来训练的图片。

![](https://zgzczzw-blog-image.oss-cn-beijing.aliyuncs.com/Tensorflow%E5%85%A5%E9%97%A82/Snip20180504_1.png)

和MNIST一样，将数据分为训练，验证和测试三部分，如果资源少验证集可以和训练集一样，测试集基本是训练集的1/5就可以了。

### 图片打标
还有需要每张图片的标签，在这个业务中我期望的分类是两类，不是自选股图片和是自选股图片，就以0和1来标识。

图片打标的方法有很多，这里列几个：

- 单独创建一个label文件，与图片文件一一对应，里面写上标签数据
- 在图片文件的文件名中标识
- 将不同类别的图片放在不同的文件夹中，以文件夹区分

在这个例子中，因为采集的数据量比较少，我采用第一种方法。

## 3.代码改造



其中有两层卷积层，两层池化层，最后输出层是两层全连接层。代码如下：

```
def deepnn(x):

    with tf.name_scope('reshape'):
        x_image = tf.reshape(x, [-1, IMG_W, IMG_H, CHANNEL])

    # 第一个卷积层，从图像中提取32的特征
    with tf.name_scope('conv1'):
        W_conv1 = weight_variable([5, 5, 3, 32])  # y =wX+b
        b_conv1 = bias_variable([32])
        h_conv1 = tf.nn.relu(conv2d(x_image, W_conv1) + b_conv1)

    # 2*2的最大化池化层
    with tf.name_scope('pool1'):
        h_pool1 = max_pool_2x2(h_conv1)

    # 第二个卷积层
    with tf.name_scope('conv2'):
        W_conv2 = weight_variable([5, 5, 32, 64])
        b_conv2 = bias_variable([64])
        h_conv2 = tf.nn.relu(conv2d(h_pool1, W_conv2) + b_conv2)

    # 第二个池化层
    with tf.name_scope('pool2'):
        h_pool2 = max_pool_2x2(h_conv2)

    # 全连接层1
    with tf.name_scope('fc1'):
        W_fc1 = weight_variable([8 * 16 * 64, 128])
        b_fc1 = bias_variable([128])

        h_pool2_flat = tf.reshape(h_pool2, [-1, 8 * 16 * 64])
        h_fc1 = tf.nn.relu(tf.matmul(h_pool2_flat, W_fc1) + b_fc1)

    # 全连接层2
    with tf.name_scope('fc2'):
        W_fc2 = weight_variable([128, 2])
        b_fc2 = bias_variable([2])

        y_conv = tf.add(tf.matmul(h_fc1, W_fc2), b_fc2, name="output")
    return y_conv
```

下面一层一层的靠谱这个网络是干嘛的



全连接层用于做这种特征分类，至此



#### 编译iOS的包
在tensorflow的目录下执行
cd tensorflow/tensorflow/contrib/makefile
./build_all_ios.sh

build_all_ios.sh脚本会依次执行download_dependencies.sh,compile_ios_protobuf.sh and compile_ios_tensorflow.sh三个脚本。

####### 编译过程中遇到的错误：
我遇到过i386编译失败，于是我把i386的架构从脚本中移除了，因为现在iphone已经基本用不到这个架构了。

修改compile_ios_tensorflow.sh中的这一句

```
# BUILD_TARGET="i386 x86_64 armv7 armv7s arm64"
BUILD_TARGET="x86_64 armv7 armv7s arm64"
```

编译完成后会在makefile的gen文件加下生成每个架构的.a文件，.a文件需要和头文件配合使用，我们尝试和头文件一起打包成可以直接使用的framwork文件。

我们注意到在makefile里面有一个create_ios_framework脚本，但是这个脚本的git提交时间是10个月前，可能已经不能用了，所以我们尝试修改这个脚本。tensorflow的[文档](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/examples/ios#building-the-tensorflow-ios-libraries-from-source)中说tf运行一共需要四个.a文件。

- tensorflow/contrib/makefile/gen/lib/libtensorflow-core.a
- tensorflow/contrib/makefile/gen/protobuf_ios/lib/libprotobuf.a
- tensorflow/contrib/makefile/gen/protobuf_ios/lib/libprotobuf-lite.a
- tensorflow/contrib/makefile/downloads/nsync/builds/lipo.ios.c++11/nsync.a

和以下头文件：

- the root folder of tensorflow,
- tensorflow/contrib/makefile/downloads/nsync/public
- tensorflow/contrib/makefile/downloads/protobuf/src
- tensorflow/contrib/makefile/downloads,
- tensorflow/contrib/makefile/downloads/eigen, and
- tensorflow/contrib/makefile/gen/proto

那么我们去检查create_ios_framework脚本中是否对这些文件都进行了处理。这里是拷贝静态文件，但是我们发现少了libprotobuf-lite.a和nsync.a，处理一下。

```
echo "Copying static libraries"
cp $SCRIPT_DIR/gen/lib/libtensorflow-core.a \
   $FW_DIR_TFCORE/tensorflow_experimental
cp $SCRIPT_DIR/gen/protobuf_ios/lib/libprotobuf.a \
   $FW_DIR_TFCORE/libprotobuf_experimental.a
```

改成这样

```
echo "Copying static libraries"
cp $SCRIPT_DIR/gen/lib/libtensorflow-core.a \
   $FW_DIR_TFCORE/tensorflow_experimental
cp $SCRIPT_DIR/gen/protobuf_ios/lib/libprotobuf.a \
   $FW_DIR_TFCORE/libprotobuf_experimental.a
cp $SCRIPT_DIR/gen/protobuf_ios/lib/libprotobuf-lite.a \
   $FW_DIR_TFCORE/libprotobuf_lite_experimental.a
cp $SCRIPT_DIR/../downloads/nsync/builds/lipo.ios.c++11/nsync.a \
   $FW_DIR_TFCORE/libprotobuf_lite_experimental.a
```

然后再看对头文件的封装


2018-04-26 21:28:14.552915: E tensorflow/core/common_runtime/session.cc:69] Not found: No session factory registered for the given session options: {target: "" config: } Registered factories are {}.

Other Linker Flags add "-force_load path"

Add Accelerate