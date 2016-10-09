---
title: 实现类知乎android客户端关注和取消关注的按钮点击效果
date: 2016-08-23 17:41:31
tags:
  - Android
  - View
  - 知乎
  - 按钮点击
categories: Android
---

前端时间在看Android各个客户端上比较出色的动画效果，发现两个动画做的很好的客户端，一个是豌豆荚，一个是知乎。接下来我可能会对这两个客户端的各种效果进行模仿实现。首先让我们看知乎的关注按钮点击效果，关注按钮点击后会有一层遮挡，从你点击的位置慢慢扩散开来，然后变成被点击状态，感觉非常赞。这篇文章从以下几个方面讨论这个效果。

- Android中实现类似效果的几种方式
  - 用Ripple实现类似效果
  - 用Paint画出类似效果
- 反编译知乎客户端代码
- 实现最终效果

先说明一下，项目代码已上传至github，不想看长篇大论的也可以先去下代码，对照代码，哪里不懂点哪里。

## Contents
- [Contents](#Contents)
- [Android中实现类似效果的几种方式](#Android中实现类似效果的几种方式)
    - [用Ripple实现类似效果](#用Ripple实现类似效果)
    - [用Paint画出类似效果](#用Paint画出类似效果)
- [反编译知乎代码](#反编译知乎代码)
- [知乎实现原理](#知乎实现原理)
- [实现最终效果](#实现最终效果)

代码在这

[https://github.com/zgzczzw/ZHFollowButton](https://github.com/zgzczzw/ZHFollowButton)


首先，让我们我先详细观察了一些知乎的效果，其中有一个很神奇的地方，如图：

![](http://img.blog.csdn.net/20160920201423285?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](http://img.blog.csdn.net/20160920201821119?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](http://img.blog.csdn.net/20160920201844952?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](http://img.blog.csdn.net/20160920201912015?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


注意看第二张图，这个圆形在扩散的时候，圆形底下的字还在，而且新的字也在圆形上，就这个效果实现起来最难。


## Android中实现类似效果的几种方式

### 用Ripple实现类似效果

ripple即波纹效果，是Android API 21以后引入的一种material design的元素，是触摸反馈的一种，也就是说点击的时候会出现水波扩散的样式，demo（见最后）中第一个按钮就是用了ripple效果。

实现方式很简单，实现一个这样的drawable

![](http://img.blog.csdn.net/20160920201948796?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

第一个color是波纹颜色，item里面指定background正常的颜色，可以是一个shape，也可以是一个drawable，还可以是一个selector。

设置为按钮的background即可

![](http://img.blog.csdn.net/20160920202005481?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

如果整个程序的theme用了meterial，那基本所有的带点击效果的控件，比如button都自带这个波纹效果。不过需要注意的是这一套API是21以后才提供的，所以需要做兼容处理。

效果如下：

![](http://img.blog.csdn.net/20160920202022234?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](http://img.blog.csdn.net/20160920202034700?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

从图中可以看出即使我设置了波纹为红色（#FF0000），点击后的效果也是淡红色，我猜测因为是水波纹效果，为了不影响按钮本身展示的内容，android系统自动做了透明度的处理，另外从图中也可以明显的看出，水波纹和显示的内容是上下两层的，互不影响，水波纹是在background层面上。这个效果做普通的点击反馈还不错，但绝对实现不出知乎这种用波纹刷新出内容的效果。所以很容易能看出知乎的点击效果不是用ripple做出来的。



### 用Paint画出类似效果

可能很多人看到知乎关注按钮的效果后，想到的第一种实现方式就是这个，用 paint在点击的地方画圆形，然后让画的圆形半径慢慢变大，实现出扩散出去的样式，我实现了一下，代码如下：

```java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    if (mShouldDoAnimation) {
        mMaxRadius = getMeasuredWidth() + 50;
        if (mRevealRadius > mMinBetweenWidthAndHeight / 2)
            mRevealRadius += mRevealRadiusGap * 4;
        else
            mRevealRadius += mRevealRadiusGap;//半径变大
        Paint mPaint = new Paint();
        if (!mIsPressed) {
            mPaint.setColor(Color.WHITE);
        } else {
            mPaint.setColor(Color.RED);
        }//设置画笔颜色
        mPaint.setStyle(Paint.Style.FILL);
        canvas.drawCircle(mCenterX, mCenterY, mRevealRadius, mPaint);

        if (mRevealRadius <= mMaxRadius) {
            //一定时间后再刷新
            postInvalidateDelayed(INVALIDATE_DURATION);
        } else {
            if (mIsPressed) {
                setTextColor(Color.WHITE);
                this.setBackgroundColor(Color.RED);
            } else {
                setTextColor(Color.BLACK);
                this.setBackgroundColor(Color.WHITE);
            }
            mShouldDoAnimation = false;
            invalidate();
        }
    }
}
```
效果如图：

![](http://img.blog.csdn.net/20160920202051969?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](http://img.blog.csdn.net/20160920202110748?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](http://img.blog.csdn.net/20160920202122766?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](http://img.blog.csdn.net/20160920202144061?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](http://img.blog.csdn.net/20160920202157467?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


本来觉得差不多就是这样，但是跟知乎的效果比较一下，还是能发现差别的。用paint画圆能实现的是在点击的地方画一个圆，然后半径慢慢变大慢慢扩散。但是问题在于，画的这个圆会盖住显示的内容，而且画的圆上也不能显示内容。我试过用drawText，也实现不了字和圆一起的效果，解决方法只有，

- 画的过程中改背景色和上面文字。
- 然后，画完圆之后把圆擦掉，把下面的背景色和文字显示出来。

这样就会出现一次文字闪烁的问题，首先文字会消失掉，然后画完圆之后才显示出来。因为圆在扩散的时候是看不到文字的，只有等圆消失了，文字才能显示出来。而知乎的效果是文字和圆一起刷出来，而且底下的文字还在，中间也没有文字闪烁的问题，整个过程行云流水，看起来很顺畅，好像用圆形揭开了幕布一样。

综上所述，知乎不是用这两种方式实现的，其实如果不是我自己实现了一下，真的以为第二种方法就是知乎采用的，但是目前看来，很遗憾，知乎采用了一种更好的方式来实现这个效果。

那怎么办呢，我也没什么思路，怎么才能在画圆的时候把字也画在圆上，然后圆下面的背景也还有呢。没什么思路，看看知乎的代码吧，反编译。

## 反编译知乎代码

反编译的过程我简单说一下：

到知乎官网下载最新的知乎apk
用apktool反编译apk，得到资源文件

![](http://img.blog.csdn.net/20160920202223063?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

在资源文件中搜索follow，这里一开始我搜的是ripple，因为我觉得这个效果总归应该和ripple有关，没结果，于是搜了follow，没想到还真搜出来了。

![](http://img.blog.csdn.net/20160920202246843?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

**RevealFollowButton**这明显就是我们要的波纹展开的控件，这就好说了，下一步就是去代码里找到这个控件了。这里要记一下，这个控件的位置**com.zhihu.android.app.ui.widget.RevealFollowButton**。


反编译代码
将apk改名成rar，打开，可以找到里面的class文件

![](http://img.blog.csdn.net/20160920202302501?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

知乎用了multidex，所以会有两个class文件，都拖出来放在dex2jar里反编译一下，就能生成两个jar包了，把jar包放在GUI里看一下，就能看到代码了，虽然代码被混淆过，但是基本逻辑还是能看出来的。

## 知乎实现原理

![](http://img.blog.csdn.net/20160920202319360?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

然后根据前面xml里的路径找到RevelFollowButton的位置，打开代码看就可以了。


这是类的继承关系，RevealFollowButton继承自RevealFrameLayout，然后继承自ZHFrameLayout，这个ZHFrameLayout的父类就是FrameLayout了，从名字我们能看出，RevelFollowButton和RevealFrameLayout就是这个效果实现的两个类了。

![](http://img.blog.csdn.net/20160920202333360?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](http://img.blog.csdn.net/20160920202347376?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


看到这个效果的实现是基于Framelayout，我就知道我们之前讨论的方法其实都走错了方向，如果告诉你用framelayout来实现这个效果，你会怎么做？

我的想法是加入两个TextView到这个layout里，然后一个Visible一个gone，如此切换，后来看过代码后，也证明我的这个想法是对的。

![](http://img.blog.csdn.net/20160920202400813?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


看，这里有两个TextView。如此的话，其实切换TextView是很容易实现的，问题是怎么实现波纹切换的效果，那第一件事就是看onDraw函数了，对于GroupView来说是drawChild方法。

![](http://img.blog.csdn.net/20160920202919460?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

RevealFollowButton的drawChild方法没什么内容，基本是调用了父类，那么我们来看RevealFrameLayout的drawChild方法。

这里有两部分逻辑，如果满足一个条件，就做第一部分，一开始我也不知道这个条件是什么，混淆后的代码能看懂大逻辑，像这种小逻辑只能走一步看一步了。所以假设这个条件永远false吧，看第二部分，看到这里瞬间明白了，原来是采用切割画布的方式，把画布切成一个圆的，就能做到显示的内容也在圆上，而不是内容被覆盖在圆下面了。然后同理，把这个圆形区域不断扩大，然后不断刷新，就是实现波形刷出内容的效果了。代码如下吧

```java

protected boolean drawChild(Canvas canvas, View paramView, long paramLong) {
    int i = canvas.save();
    mPath.reset();
    //mCenterX mCenterY是点击的位置，在onTouchEvent里设置
    //mRevealRadius是圆的半径，会渐渐变大
    mPath.addCircle(mCenterX, mCenterY, mRevealRadius, Path.Direction.CW);
    canvas.clipPath(this.mPath);
    boolean bool2 = super.drawChild(canvas, paramView, paramLong);
    canvas.restoreToCount(i);
    return bool2;
}
```

按照上面说的，肯定还有一个类似于定时器的东西，能不断改变圆形的半径，然后刷新，其实这个在代码里找找很容易就找到了。RevealFrameLayout里除了这个drawChild，没有别的代码了。所以我们来看RevealFollowButton。


RevealFollowButton里面跟定时器有关的就是这句了

![](http://img.blog.csdn.net/20160920202426142?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


一个Animator对象，其实这句代码我是没看懂的，但逻辑很简单，设置一个Animator，定时500ms，在这个过程中修改圆形半径，然后刷新。

`Math.hypot(getWidth(), getHeight()))`


其中这个方法是根据勾股定理获取三角形的斜边长度，想想我们所要绘制的圆形半径最长是多少，没错，就是TextView的对角线长度。所以，整个逻辑就很简单了。

我搞了下代码，就这样吧

![](http://img.blog.csdn.net/20160920202443596?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


整个方法的代码如下吧，还包括控制FollowTv和unFollowTv哪个显示

```java
protected void setFollowed(boolean isFollowed, boolean needAnimate) {
    mIsFollowed = isFollowed;
    if (isFollowed) {
        mUnFollowTv.setVisibility(View.VISIBLE);
        mFollowTv.setVisibility(View.VISIBLE);
        mFollowTv.bringToFront();
    } else {
        mUnFollowTv.setVisibility(View.VISIBLE);
        mFollowTv.setVisibility(View.VISIBLE);
        mUnFollowTv.bringToFront();
    }
    if (needAnimate) {
        ValueAnimator animator = ObjectAnimator.ofFloat(mFollowTv, "empty", 0.0F, (float) Math.hypot(getMeasuredWidth(), getMeasuredHeight()));
        animator.setDuration(500L);
        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                mRevealRadius = (Float) animation.getAnimatedValue();
                invalidate();
            }
        });
        animator.start();
    }
}
```

根据当前状态把Follow的Textview或UnFollow的TextView显示出来，然后设置一个定时器不断扩大所要绘制圆的半径，根据这个半径裁剪画布成一个渐渐变大的圆形，然后内容就渐渐显示出来了。

## 实现最终效果

这个效果实现出来之后，试着运行一下，还不错，但是总觉得有地方不对，于是细细观察，终于发现了，知乎的那个效果在刷新的时候，底下的背景不是白色的，还是之前的状态，比如要变成关注的时候，背景中的未关注还是在的，而我们实现的这个，刷新的时候背景是白色的。


这是知乎的

![](http://img.blog.csdn.net/20160920202500751?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

这是我的

![](http://img.blog.csdn.net/20160920202513237?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

所以还是没有知乎那么行云流水，所以我们是少了什么吗。这时候想起来了，之前在RevealFrameLayout的drawChild里有一个判断条件，当时我们不知道它的逻辑是干什么的，现在看来。那部分逻辑就是处理这个的，画子控件的时候，要画两个，FollowTextView和UnFollowTextView，要随圆形刷出的控件我们采用裁剪画布的方式慢慢画出。那作为背景的另一个控件就不需要慢慢画出，只要完全画出来就行了。所以，猜想这里这个判断条件就是判断当前控件是不是要随圆形刷出的控件，如果不是，就直接画出来就行了。所以修改代码如下：

```java
protected boolean drawChild(Canvas canvas, View paramView, long paramLong) {
    if (drawBackground(paramView)) {
        return super.drawChild(canvas, paramView, paramLong);
    }
    int i = canvas.save();
    mPath.reset();
    mPath.addCircle(mCenterX, mCenterY, mRevealRadius, Path.Direction.CW);
    canvas.clipPath(this.mPath);
    boolean bool2 = super.drawChild(canvas, paramView, paramLong);
    canvas.restoreToCount(i);
    return bool2;
}
```

判断的方法如下：

```java
private boolean drawBackground(View paramView) {
    if (mIsFollowed && paramView == mUnFollowTv) {
        return true;
    } else if (!mIsFollowed && paramView == mFollowTv) {
        return true;
    }
    return false;
}
```
至此，整个效果就和知乎完全一样了，刷新过程行云流水，非常赞。效果如下

![](http://img.blog.csdn.net/20160920202528190?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](http://img.blog.csdn.net/20160920202540940?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](http://img.blog.csdn.net/20160920202552455?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](http://img.blog.csdn.net/20160920202620051?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](http://img.blog.csdn.net/20160920202635129?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](http://img.blog.csdn.net/20160920202650911?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


实现代码已上传至github：

[https://github.com/zgzczzw/ZHFollowButton](https://github.com/zgzczzw/ZHFollowButton)
