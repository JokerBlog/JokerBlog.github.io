---
layout:     post
title:      "RecyclerView源码解析"
subtitle:   ""
date:       2019-06-13 09:09:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

    本系列文章楼主打算从几个地方说起。先是将RecyclerView当成一个普通的View，分别分析它的三大流程、事件传递（包括嵌套滑动）；然后是分析RecyclerView的缓存原理，这也是RecyclerView的精华所在;然后分析的是RecyclerView的Adapter、LayoutManager、ItemAnimator和ItemDecoration。最后就是RecyclerView的扩展，包括LayoutManager的自定义和使用RecyclerView常见的坑等。



### 1、概述

        在分析RecyclerView源码之前，我们还是对RecyclerView有一个初步的了解，简单的了解它是什么，它的基本结构有哪些。RecyclerView是Google爸爸在2014年的IO大会提出来（看来RecyclerView的年龄还是比较大了😂），具体目的是不是用来替代ListView的，楼主也不知道，因为那时候楼主还在读高二。但是在实际开发中，自从有了RecyclerView，ListView和GridView就很少用了，所以我们暂且认为RecyclerView的目的是替代ListView和GridView。

       RecyclerView本身是一个展示大量数据的控件，相比较ListView,RecyclerView的4级缓存(也有人说是3级缓存，这些都不重要😂)就表现的非常出色，在性能方面相比于ListView提升了不少。同时由于LayoutManager的存在,让RecyclerView不仅有ListView的特点，同时兼有GridView的特点。这可能是RecyclerView受欢迎的原因之一吧。

        RecyclerView在设计方面上也是非常的灵活，不同的部分承担着不同的职责。其中Adapter负责提供数据，包括创建ViewHolder和绑定数据，LayoutManager负责ItemView的测量和布局,ItemAnimator负责每个ItemView的动画，ItemDecoration负责每个ItemView的间隙。这种插拔式的架构使得RecyclerView变得非常的灵活，每一个人都可以根据自身的需求来定义不同的部分。

        正因为这种插拔式的设计，使得RecyclerView在使用上相比较于其他的控件稍微难那么一点点，不过这都不算事，谁叫RecyclerView这么惹人爱呢😂。



### 2、Measure

不管RecyclerView是多么神奇，它也是一个View，所以分析它的三大流程是非常有必要的。同时，如果了解过RecyclerView的同学应该都知道，RecyclerView的三大流程跟普通的View比较，有很大的不同。



首先，我们来看看measure过程，来看看RecyclerView的onMeasure方法。

```java
protected void onMeasure(int widthSpec, int heightSpec) {
    if (mLayout == null) {
        // 第一种情况
    }
    if (mLayout.isAutoMeasureEnabled()) {
        // 第二种情况
    } else {
        // 第三种情况
    }
}
```

        onMeasure方法还是有点长，这里我将它分为3种情况，我将简单解释这三种情况。

       mLayout即LayoutManager的对象。我们知道，当RecyclerView的LayoutManager为空时，RecyclerView不能显示任何的数据，在这里我们找到答案。

       LayoutManager开启了自动测量时，这是一种情况。在这种情况下，有可能会测量两次。

       第三种情况就是没有开启自动测量的情况，这种情况比较少，因为为了RecyclerView支持warp_content属性，系统提供的LayoutManager都开启自动测量的，不过我们还是要分析的。



首先我们来第一种情况。

#### **(1).当LayoutManager为空时**

这种情况下比较简单，我们来看看源码：

```java
if (mLayout == null) {
    defaultOnMeasure(widthSpec, heightSpec);
    return;
}
```

直接调了defaultOnMeasure方法，我们继续来看defaultOnMeasure方法。

```java
void defaultOnMeasure(int widthSpec, int heightSpec) {
    // calling LayoutManager here is not pretty but that API is already public and it is better
    // than creating another method since this is internal.
    final int width = LayoutManager.chooseSize(widthSpec,
            getPaddingLeft() + getPaddingRight(),
            ViewCompat.getMinimumWidth(this));
    final int height = LayoutManager.chooseSize(heightSpec,
            getPaddingTop() + getPaddingBottom(),
            ViewCompat.getMinimumHeight(this));

    setMeasuredDimension(width, height);
}
```

在defaultOnMeasure方法里面，先是通过LayoutManager的chooseSize方法来计算值，然后就是setMeasuredDimension方法来设置宽高。我们来看看：

```java
public static int chooseSize(int spec, int desired, int min) {
    final int mode = View.MeasureSpec.getMode(spec);
    final int size = View.MeasureSpec.getSize(spec);
    switch (mode) {
        case View.MeasureSpec.EXACTLY:
            return size;
        case View.MeasureSpec.AT_MOST:
            return Math.min(size, Math.max(desired, min));
        case View.MeasureSpec.UNSPECIFIED:
        default:
            return Math.max(desired, min);
    }
}
```

chooseSize方法表达的意思比较简单，就是通过RecyclerView的测量mode来获取不同的值，这里就不详细的解释了。

到此，**第一种情况就分析完毕了。因为当LayoutManager为空时，那么当RecyclerView处于onLayout阶段时，会调用dispatchLayout方法**。而在dispatchLayout方法里面有这么一行代码：

```java
if (mLayout == null) {
    Log.e(TAG, "No layout manager attached; skipping layout");
    // leave the state in START
    return;
}
```

所以，当LayoutManager为空时，不显示任何数据是理所当然的。

现在我们来看看第二种情况，也就是正常的情况。



#### **(2). 当LayoutManager开启了自动测量**

在分析这种情况之前，我们先对了解几个东西。

RecyclerView的测量分为两步，分别调用dispatchLayoutStep1和dispatchLayoutStep2。同时，了解过RecyclerView源码的同学应该知道在RecyclerView的源码里面还一个dispatchLayoutStep3方法。这三个方法的方法名比较接近，所以容易让人搞混淆。本文会详细的讲解这三个方法的作用。

由于在这种情况下，只会调用dispatchLayoutStep1和dispatchLayoutStep2这两个方法，所以这里会重点的讲解这两个方法。而dispatchLayoutStep3方法的调用在RecyclerView的onLayout方法里面，所以在后面分析onLayout方法时再来看dispatchLayoutStep3方法。

我们在分析之前，先来看一个东西--mState.mLayoutStep。这个变量有几个取值情况。我们分别来看看：

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt76fWdGrVY2f0c8qtulNb5I59mRO2PZVnofP4yDsjt2HFbDuFAOLKJG8rShh82ty49wLFibW1sCWeA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



从上表中，我们了解到mState.mLayoutStep的三个状态对应着不同的dispatchLayoutStep方法。这一点，我们必须清楚，否则接下来的代码将难以理解。

好了，前戏准备的差不多，现在应该进入高潮了😂。我们开始正式的分析源码了。

```java
if (mLayout.isAutoMeasureEnabled()) {
    final int widthMode = MeasureSpec.getMode(widthSpec);
    final int heightMode = MeasureSpec.getMode(heightSpec);

    /**
     * This specific call should be considered deprecated and replaced with
     * {@link #defaultOnMeasure(int, int)}. It can't actually be replaced as it could
     * break existing third party code but all documentation directs developers to not
     * override {@link LayoutManager#onMeasure(int, int)} when
     * {@link LayoutManager#isAutoMeasureEnabled()} returns true.
     */
    mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);

    final boolean measureSpecModeIsExactly =
            widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
    if (measureSpecModeIsExactly || mAdapter == null) {
        return;
    }

    if (mState.mLayoutStep == State.STEP_START) {
        dispatchLayoutStep1();
    }
    // set dimensions in 2nd step. Pre-layout should happen with old dimensions for
    // consistency
    mLayout.setMeasureSpecs(widthSpec, heightSpec);
    mState.mIsMeasuring = true;
    dispatchLayoutStep2();

    // now we can get the width and height from the children.
    mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

    // if RecyclerView has non-exact width and height and if there is at least one child
    // which also has non-exact width & height, we have to re-measure.
    if (mLayout.shouldMeasureTwice()) {
        mLayout.setMeasureSpecs(
                MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
        mState.mIsMeasuring = true;
        dispatchLayoutStep2();
        // now we can get the width and height from the children.
        mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
    }
}
```

我将这段代码分为三步。我们来看看：

调用LayoutManager的onMeasure方法进行测量。对于onMeasure方法，我也感觉到非常的迷惑，发现传统的LayoutManager都没有实现这个方法。后面，我们会将简单的看一下这个方法。

如果mState.mLayoutStep为State.STEP_START的话，那么就会执行dispatchLayoutStep1方法，然后会执行dispatchLayoutStep2方法。

如果需要第二次测量的话，会再一次调用dispatchLayoutStep2 方法。

以上三步，我们一步一步的来分析。首先，我们来看看第一步，也是看看onMeasure方法。

LayoutManager的onMeasure方法究竟为我们做什么，我们来看看：

```java
public void onMeasure(Recycler recycler, State state, int widthSpec, int heightSpec) {
    mRecyclerView.defaultOnMeasure(widthSpec, heightSpec);
}
```

默认是调用的RecyclerView的defaultOnMeasure方法,至于defaultOnMeasure方法里面究竟做了什么，这在前面已经介绍过了，这里就不再介绍了。  

View的onMeasure方法的作用通常来说有两个。一是测量自身的宽高，从RecyclerView来看，它将自己的测量工作托管给了LayoutManager的onMeasure方法。所以，我们在自定义LayoutManager时，需要注意onMeasure方法存在，不过从官方提供的几个LayoutManager，都没有重写这个方法。所以不到万得已，最好不要重写LayoutManager的onMeasure方法；二是测量子View,不过到这里我们还没有看到具体的实现。  

接下来，我们来分析第二步，看看dispatchLayoutStep1方法和dispatchLayoutStep2方法究竟做了什么。

在正式分析第二步之前，我们先对这三个方法有一个大概的认识。

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt76fWdGrVY2f0c8qtulNb5IEur6AGgXuKV1GjWNyFdiak7jxjf2yvr9YoPDIeSEzreMOKDibZic5CJ3A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

我们回到onMeasure方法里面，先看看整个执行过程。

```java
if (mState.mLayoutStep == State.STEP_START) {
    dispatchLayoutStep1();
}
// set dimensions in 2nd step. Pre-layout should happen with old dimensions for
// consistency
mLayout.setMeasureSpecs(widthSpec, heightSpec);
mState.mIsMeasuring = true;
dispatchLayoutStep2();
```

如果mState.mLayoutStep == State.STEP_START时，才会调用 dispatchLayoutStep1方法，这里与我们前面介绍mLayoutStep对应起来了。现在我们看看dispatchLayoutStep1方法

```java
private void dispatchLayoutStep1() {
    mState.assertLayoutStep(State.STEP_START);
    fillRemainingScrollValues(mState);
    mState.mIsMeasuring = false;
    startInterceptRequestLayout();
    mViewInfoStore.clear();
    onEnterLayoutOrScroll();
    processAdapterUpdatesAndSetAnimationFlags();
    saveFocusInfo();
    mState.mTrackOldChangeHolders = mState.mRunSimpleAnimations && mItemsChanged;
    mItemsAddedOrRemoved = mItemsChanged = false;
    mState.mInPreLayout = mState.mRunPredictiveAnimations;
    mState.mItemCount = mAdapter.getItemCount();
    findMinMaxChildLayoutPositions(mMinMaxLayoutPositions);

    if (mState.mRunSimpleAnimations) {
       // 找到没有被remove的ItemView,保存OldViewHolder信息，准备预布局
    }
    if (mState.mRunPredictiveAnimations) {
       // 进行预布局
    } else {
        clearOldPositions();
    }
    onExitLayoutOrScroll();
    stopInterceptRequestLayout(false);
    mState.mLayoutStep = State.STEP_LAYOUT;
}
```

本文只简单分析一下这个方法，因为这个方法跟ItemAnimator有莫大的关系，后续在介绍ItemAnimator时会详细的分析。在这里，我们将重点放在processAdapterUpdatesAndSetAnimationFlags里面，因为这个方法计算了mRunSimpleAnimations和mRunPredictiveAnimations。

```java
private void processAdapterUpdatesAndSetAnimationFlags() {
    if (mDataSetHasChangedAfterLayout) {
        // Processing these items have no value since data set changed unexpectedly.
        // Instead, we just reset it.
        mAdapterHelper.reset();
        if (mDispatchItemsChangedEvent) {
            mLayout.onItemsChanged(this);
        }
    }
    // simple animations are a subset of advanced animations (which will cause a
    // pre-layout step)
    // If layout supports predictive animations, pre-process to decide if we want to run them
    if (predictiveItemAnimationsEnabled()) {
        mAdapterHelper.preProcess();
    } else {
        mAdapterHelper.consumeUpdatesInOnePass();
    }
    boolean animationTypeSupported = mItemsAddedOrRemoved || mItemsChanged;
    mState.mRunSimpleAnimations = mFirstLayoutComplete
            && mItemAnimator != null
            && (mDataSetHasChangedAfterLayout
            || animationTypeSupported
            || mLayout.mRequestedSimpleAnimations)
            && (!mDataSetHasChangedAfterLayout
            || mAdapter.hasStableIds());
    mState.mRunPredictiveAnimations = mState.mRunSimpleAnimations
            && animationTypeSupported
            && !mDataSetHasChangedAfterLayout
            && predictiveItemAnimationsEnabled();
}
```

这里我们的重心放在mFirstLayoutComplete变量里面，我们发现mRunSimpleAnimations的值与mFirstLayoutComplete有关，mRunPredictiveAnimations同时跟mRunSimpleAnimations有关。所以这里我们可以得出一个结论,当RecyclerView第一次加载数据时，是不会执行的动画。换句话说，每个ItemView还没有layout完毕，怎么会进行动画。这一点，我们也可以通过Demo来证明，这里也就不展示了。

接下来我们看看dispatchLayoutStep2方法，这个方法是真正布局children。我们来看看：

```java
private void dispatchLayoutStep2() {
    startInterceptRequestLayout();
    onEnterLayoutOrScroll();
    mState.assertLayoutStep(State.STEP_LAYOUT | State.STEP_ANIMATIONS);
    mAdapterHelper.consumeUpdatesInOnePass();
    mState.mItemCount = mAdapter.getItemCount();
    mState.mDeletedInvisibleItemCountSincePreviousLayout = 0;

    // Step 2: Run layout
    mState.mInPreLayout = false;
    mLayout.onLayoutChildren(mRecycler, mState);

    mState.mStructureChanged = false;
    mPendingSavedState = null;

    // onLayoutChildren may have caused client code to disable item animations; re-check
    mState.mRunSimpleAnimations = mState.mRunSimpleAnimations && mItemAnimator != null;
    mState.mLayoutStep = State.STEP_ANIMATIONS;
    onExitLayoutOrScroll();
    stopInterceptRequestLayout(false);
}
```

在这里，我们重点的看两行代码。一是在这里，我们可以看到Adapter的getItemCount方法被调用；二是调用了LayoutManager的onLayoutChildren方法,这个方法里面进行对children的测量和布局，同时这个方法也是这里的分析重点。

系统的LayoutManager的onLayoutChildren方法是一个空方法，所以需要LayoutManager的子类自己来实现。从这里，我们可以得出两个点。

1. 子类LayoutManager需要自己实现onLayoutChildren方法，从而来决定RecyclerView在该LayoutManager的策略下，应该怎么布局。从这里，我们看出来RecyclerView的灵活性。

2. LayoutManager类似于ViewGroup,将onLayoutChildren方法(ViewGroup是onLayout方法)公开出来，这种模式在Android中很常见的。

这里，我先不对onLayoutChildren方法进行展开，待会会详细的分析。

接下来，我们来分析第三种情况--**没有开启自动测量**。



#### **(3).没有开启自动测量**

我们先来看看这一块的代码。

```java
if (mHasFixedSize) {
    mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
    return;
}
// custom onMeasure
if (mAdapterUpdateDuringMeasure) {
    startInterceptRequestLayout();
    onEnterLayoutOrScroll();
    processAdapterUpdatesAndSetAnimationFlags();
    onExitLayoutOrScroll();

    if (mState.mRunPredictiveAnimations) {
        mState.mInPreLayout = true;
    } else {
        // consume remaining updates to provide a consistent state with the layout pass.
        mAdapterHelper.consumeUpdatesInOnePass();
        mState.mInPreLayout = false;
    }
    mAdapterUpdateDuringMeasure = false;
    stopInterceptRequestLayout(false);
} else if (mState.mRunPredictiveAnimations) {
    // If mAdapterUpdateDuringMeasure is false and mRunPredictiveAnimations is true:
    // this means there is already an onMeasure() call performed to handle the pending
    // adapter change, two onMeasure() calls can happen if RV is a child of LinearLayout
    // with layout_width=MATCH_PARENT. RV cannot call LM.onMeasure() second time
    // because getViewForPosition() will crash when LM uses a child to measure.
    setMeasuredDimension(getMeasuredWidth(), getMeasuredHeight());
    return;
}

if (mAdapter != null) {
    mState.mItemCount = mAdapter.getItemCount();
} else {
    mState.mItemCount = 0;
}
startInterceptRequestLayout();
mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
stopInterceptRequestLayout(false);
mState.mInPreLayout = false; // clear
```

例如上面的代码，我将分为2步：

1. 如果mHasFixedSize为true(也就是调用了setHasFixedSize方法)，将直接调用LayoutManager的onMeasure方法进行测量。

2. 如果mHasFixedSize为false，同时此时如果有数据更新，先处理数据更新的事务，然后调用LayoutManager的onMeasure方法进行测量

通过上面的描述，我们知道，如果未开启自动测量，那么肯定会调用LayoutManager的onMeasure方法来进行测量，这就是LayoutManager的onMeasure方法的作用。

至于onMeasure方法怎么进行测量，那就得看LayoutManager的实现类。在这里，我们就不进行深入的追究了。



### 3.Layout

measure过程分析的差不多了，接下来我们就该分析第二个过程--layout。我们来看看onLayout方法：

```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    TraceCompat.beginSection(TRACE_ON_LAYOUT_TAG);
    dispatchLayout();
    TraceCompat.endSection();
    mFirstLayoutComplete = true;
}
```

onLayout方法本身没有做多少的事情，重点还是在dispatchLayout方法里面。

```java
void dispatchLayout() {
    if (mAdapter == null) {
        Log.e(TAG, "No adapter attached; skipping layout");
        // leave the state in START
        return;
    }
    if (mLayout == null) {
        Log.e(TAG, "No layout manager attached; skipping layout");
        // leave the state in START
        return;
    }
    mState.mIsMeasuring = false;
    if (mState.mLayoutStep == State.STEP_START) {
        dispatchLayoutStep1();
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth()
            || mLayout.getHeight() != getHeight()) {
        // First 2 steps are done in onMeasure but looks like we have to run again due to
        // changed size.
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else {
        // always make sure we sync them (to ensure mode is exact)
        mLayout.setExactMeasureSpecsFrom(this);
    }
    dispatchLayoutStep3();
}
```

dispatchLayout方法也是非常的简单，这个方法保证RecyclerView必须经历三个过程--dispatchLayoutStep1、dispatchLayoutStep2、dispatchLayoutStep3。

同时，在后面的文章中，你会看到dispatchLayout方法其实还为RecyclerView节省了很多步骤，也就是说，在RecyclerView经历一次完整的dispatchLayout之后，后续如果参数有所变化时，可能只会经历最后的1步或者2步。当然这些都是后话了😂。

对于dispatchLayoutStep1和dispatchLayoutStep2方法，我们前面已经讲解了，这里就不做过多的解释了。这里，我们就简单的看一下dispatchLayoutStep3方法吧。

```java
private void dispatchLayoutStep3() {
    // ······
    mState.mLayoutStep = State.STEP_START;
    // ······
}
```

为什么这里只是简单看一下dispatchLayoutStep3方法呢？因为这个方法主要是做Item的动画，也就是我们熟知的ItemAnimator的执行，而本文不对动画进行展开，所以先省略动画部分。

在这里，我们需要关注dispatchLayoutStep3方法的是，它将mLayoutStep重置为了State.STEP_START。也就是说如果下一次重新开始dispatchLayout的话，那么肯定会经历dispatchLayoutStep1、dispatchLayoutStep2、dispatchLayoutStep3三个方法。

以上就是RecyclerView的layout过程，是不是感觉非常的简单？RecyclerView跟其他ViewGroup不同的地方在于，如果开启了自动测量，在measure阶段，已经将Children布局完成了；如果没有开启自动测量，则在layout阶段才布局Children。



### 4.draw

接下来，我们来分析三大流程的最后一个阶段--draw。在正式分析draw过程之前，我先来对RecyclerView的draw做一个概述。

RecyclerView分为三步，我们来看看：

1. 调用super.draw方法。这里主要做了两件事：1. 将Children的绘制分发给ViewGroup;2. 将分割线的绘制分发给ItemDecoration。

2. 如果需要的话，调用ItemDecoration的onDrawOver方法。通过这个方法，我们在每个ItemView上面画上很多东西。

3. 如果RecyclerView调用了setClipToPadding,会实现一种特殊的滑动效果--**每个ItemView可以滑动到padding区域**。

我们来看看这部分的代码：

```java
public void draw(Canvas c) {
    // 第一步
    super.draw(c);
    // 第二步
    final int count = mItemDecorations.size();
    for (int i = 0; i < count; i++) {
        mItemDecorations.get(i).onDrawOver(c, this, mState);
    }
    // 第三步
    // TODO If padding is not 0 and clipChildrenToPadding is false, to draw glows properly, we
    // need find children closest to edges. Not sure if it is worth the effort.
    // ······
}
```

熟悉三大流程的同学，肯定知道第一步会回调到onDraw方法里面，也就是说关于Children的绘制和ItemDecoration的绘制，是在onDraw方法里面。

```java
@Override
public void onDraw(Canvas c) {
    super.onDraw(c);

    final int count = mItemDecorations.size();
    for (int i = 0; i < count; i++) {
        mItemDecorations.get(i).onDraw(c, this, mState);
    }
}
```

onDraw方法是不是非常的简单？调用super.onDraw方法将Children的绘制分发给ViewGroup执行；然后将ItemDecoration的绘制分发到ItemDecoration的onDraw方法里面去。从这里，我们可以看出来，RecyclerView的设计实在是太灵活了！

至于其余两步都比较简单，这里就不详细分析了。不过，从这里，我们终于明白了ItemDecoration的onDraw方法和onDrawOver方法的区别。



### 5. onLayoutChildren方法

从整体来说，RecyclerView的三大流程还是比较简单，不过在整个过程中，我们似乎忽略了一个过程--那就是RecyclerView到底是怎么layout children的？

前面在介绍dispatchLayoutStep2方法时，只是简单的介绍了，RecyclerView通过调用LayoutManager的onLayoutChildren方法。LayoutManager本身对这个方法没有进行实现，所以必须得看看它的子类，这里我们就来看看LinearLayoutManager。

由于LinearLayoutManager的onLayoutChildren方法比较长，这里不可能贴出完整的代码，所以这里我先对这个方法做一个简单的概述，方便大家理解。

1. 确定锚点的信息，这里面的信息包括：1.Children的布局方向，有start和end两个方向；2. mPosition和mCoordinate，分别表示Children开始填充的position和坐标。

2. 调用detachAndScrapAttachedViews方法，detach掉或者remove掉RecyclerView的Children。这一点本来不在本文的讲解范围内，但是为了后续对RecyclerView的缓存机制有更好的了解，这里特别的提醒一下。

3. 根据锚点信息，调用fill方法进行Children的填充。这个过程中根据锚点信息的不同，可能会调用两次fill方法。

接下来，我们看看代码：

```java
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    // layout algorithm:
    // 1) by checking children and other variables, find an anchor coordinate and an anchor
    //  item position.
    // 2) fill towards start, stacking from bottom
    // 3) fill towards end, stacking from top
    // 4) scroll to fulfill requirements like stack from bottom.
    // create layout state
    // ······
    // 第一步
    final View focused = getFocusedChild();
    if (!mAnchorInfo.mValid || mPendingScrollPosition != NO_POSITION
            || mPendingSavedState != null) {
        mAnchorInfo.reset();
        mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd;
        // calculate anchor position and coordinate
        updateAnchorInfoForLayout(recycler, state, mAnchorInfo);
        mAnchorInfo.mValid = true;
    }
    // ······
    // 第二步
    detachAndScrapAttachedViews(recycler);
    mLayoutState.mIsPreLayout = state.isPreLayout();
    // 第三步
    if (mAnchorInfo.mLayoutFromEnd) {
        // fill towards start
        updateLayoutStateToFillStart(mAnchorInfo);
        mLayoutState.mExtra = extraForStart;
        fill(recycler, mLayoutState, state, false);
        startOffset = mLayoutState.mOffset;
        final int firstElement = mLayoutState.mCurrentPosition;
        if (mLayoutState.mAvailable > 0) {
            extraForEnd += mLayoutState.mAvailable;
        }
        // fill towards end
        updateLayoutStateToFillEnd(mAnchorInfo);
        mLayoutState.mExtra = extraForEnd;
        mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
        fill(recycler, mLayoutState, state, false);
        endOffset = mLayoutState.mOffset;

        if (mLayoutState.mAvailable > 0) {
            // end could not consume all. add more items towards start
            extraForStart = mLayoutState.mAvailable;
            updateLayoutStateToFillStart(firstElement, startOffset);
            mLayoutState.mExtra = extraForStart;
            fill(recycler, mLayoutState, state, false);
            startOffset = mLayoutState.mOffset;
        }
    } else {
        // fill towards end
        updateLayoutStateToFillEnd(mAnchorInfo);
        mLayoutState.mExtra = extraForEnd;
        fill(recycler, mLayoutState, state, false);
        endOffset = mLayoutState.mOffset;
        final int lastElement = mLayoutState.mCurrentPosition;
        if (mLayoutState.mAvailable > 0) {
            extraForStart += mLayoutState.mAvailable;
        }
        // fill towards start
        updateLayoutStateToFillStart(mAnchorInfo);
        mLayoutState.mExtra = extraForStart;
        mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
        fill(recycler, mLayoutState, state, false);
        startOffset = mLayoutState.mOffset;

        if (mLayoutState.mAvailable > 0) {
            extraForEnd = mLayoutState.mAvailable;
            // start could not consume all it should. add more items towards end
            updateLayoutStateToFillEnd(lastElement, endOffset);
            mLayoutState.mExtra = extraForEnd;
            fill(recycler, mLayoutState, state, false);
            endOffset = mLayoutState.mOffset;
        }
    }
    // ······
}
```

相信从上面的代码都可以找出每一步的执行。现在，我们来详细分析每一步。首先来看第一步--**确定锚点的信息**。

要想看锚点信息的计算过程，我们可以从updateAnchorInfoForLayout方法里面来找出答案，我们来看看updateAnchorInfoForLayout方法：

```java
private void updateAnchorInfoForLayout(RecyclerView.Recycler recycler, RecyclerView.State state,
        AnchorInfo anchorInfo) {
    // 第一种计算方式
    if (updateAnchorFromPendingData(state, anchorInfo)) {
        return;
    }
    // 第二种计算方式
    if (updateAnchorFromChildren(recycler, state, anchorInfo)) {
        return;
    }
    // 第三种计算方式
    anchorInfo.assignCoordinateFromPadding();
    anchorInfo.mPosition = mStackFromEnd ? state.getItemCount() - 1 : 0;
}
```

我相信通过上面的代码注释，大家都能明白updateAnchorInfoForLayout方法到底干了嘛，这里我简单分析一下这三种确定所做的含义，具体是怎么做的，这里就不讨论，因为这里面的细节太多了，深入的讨论容易将我们聪明无比的大脑搞晕😂。

1. 第一种计算方式，表示含义有两种：1.RecyclerView被重建，期间回调了onSaveInstanceState方法，所以目的是为了恢复上次的布局；2.RecyclerView调用了scrollToPosition之类的方法，所以目的是让RecyclerView滚到准确的位置上去。所以，锚点的信息根据上面的两种情况来计算。

2. 第二种计算方法，从Children上面来计算锚点信息。这种计算方式也有两种情况：1. 如果当前有拥有焦点的Child，那么有当前有焦点的Child的位置来计算锚点；2. 如果没有child拥有焦点，那么根据布局方向(此时布局方向由mLayoutFromEnd来决定)获取可见的第一个ItemView或者最后一个ItemView。

3. 如果前面两种方式都计算失败了，那么采用第三种计算方式，也就是默认的计算方式。

以上就是updateAnchorInfoForLayout方法所做的事情，这里就不详细纠结每种计算方式的细节，有兴趣的同学可以看看。

至于第二步，调用detachAndScrapAttachedViews方法对所有的ItemView进行回收,这部分的内容属于RecyclerView缓存机制的部分，本文先在这里埋下一个伏笔，后续专门讲解RecyclerView会详细的分析它，所以这里就不讲解了。

接下来我们来看看第三步，也就是调用fill方法来填充Children。在正式分析填充过程时，我们先来看一张图片：

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt5ljULZtariavOHhUNtLrORkkPLc5DYWQYKicntiajYJmFI9f9Dry7RO1BOBq2X8nibp4GdlavkzVK6bA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

上图形象的展现出三种fill的情况。其中，我们可以看到第三种情况，fill方法被调用了两次。

我们看看fill方法:

fill方法的代码比较长，其实都是来计算可填充的空间，真正填充Child的地方是layoutChunk方法。我们来看看layoutChunk方法。

由于layoutChunk方法比较长，这里我就不完整的展示，为了方便理解，我对这个方法做一个简单的概述，让大家有一个大概的理解。

1. 调用LayoutState的next方法获得一个ItemView。千万别小看这个next方法，RecyclerView缓存机制的起点就是从这个方法开始，可想而知，这个方法到底为我们做了多少事情。

2. 如果RecyclerView是第一次布局Children的话(layoutState.mScrapList == null为true)，会先调用addView，将View添加到RecyclerView里面去。

3. 调用measureChildWithMargins方法，测量每个ItemView的宽高。注意这个方法测量ItemView的宽高考虑到了两个因素：1.margin属性；2.ItemDecoration的offset。

4. 调用layoutDecoratedWithMargins方法，布局ItemView。这里也考虑上面的两个因素的。

至于每一步具体干了嘛，这里就不详细的解释，都是一些基本操作，有兴趣的同学可以看看。

综上所述，便是LayoutManager的onLayoutChildren方法整个执行过程，思路还是比较简单的。



### 6.总结

本文到此就差不多了，在最后，我做一个简单的总结。

1. RecyclerView的measure过程分为三种情况，每种情况都有执行过程。通常来说，我们都会走自动测量的过程。

2. 自动测量里面需要分清楚mState.mLayoutStep状态值，因为根据不同的状态值调用不同的dispatchLayoutStep方法。

3. layout过程也根据mState.mLayoutStep状态来调用不同的dispatchLayoutStep方法。

4. draw过程主要做了四件事：1. 绘制ItemDecoration的onDraw部分。2. 绘制Children。3. 绘制ItemDecoration的drawOver部分。4. 根据mClipToPadding的值来判断是否进行特殊绘制。














