---
layout:     post
title:      "嵌套滚动设计和源码分析"
subtitle:   "VerticalNestedScrollLayout 的使用"
date:       2018-04-03
author:     "许方镇"
header-img: "img/bg5.jpg"
tags:
    Android
    NestedScroll
---

## VerticalNestedScrollLayout的使用

### 简介
VerticalNestedScrollLayout实现了垂直嵌套滚动的通用组件。其内部有且仅有两个直接子View: **头部**和**主体**。

两个子View一般写在布局中，如下：VerticalNestedScrollLayout有两个直接子View，NestedScrollViewh 和 FrameLayout。

```xml
<com.kaola.base.ui.scroll.VerticalNestedScrollLayout
        xmlns:vnsl="http://schemas.android.com/apk/res-auto"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        vnsl:isScrollDownWhenFirstItemIsTop="true"
        >

        <android.support.v4.widget.NestedScrollView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            >
           ⋯⋯
           
        </android.support.v4.widget.NestedScrollView>


        <FrameLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            >
            
            ⋯⋯
            
            <android.support.v7.widget.RecyclerView
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:overScrollMode="never"
                />
            
            ⋯⋯
    
        </FrameLayout>
    </com.kaola.base.ui.scroll.VerticalNestedScrollLayout>
```

VerticalNestedScrollLayout作为嵌套滚动的父组件，需要配合支持嵌套滚动的子View组件进行。

#### 布局介绍中：

1. 第一个子View为NestedScrollViewh，是系统实现了嵌套滚动的组件，本质是继承了FrameLayout实现了NestedScrollingParent和NestedScrollingChild接口的组件。因此NestedScrollViewh是既可以做父组件，也可以做子组件。
2. 第二个子View是FrameLayout，是不支持嵌套滚动的，但是FrameLayout的子View里有RecyclerView，RecyclerView实现了NestedScrollingChild。**嵌套滚动不需要直接子View或者父View支持嵌套滚动，间接也可以，内部有遍历寻找的逻辑**。


#### VerticalNestedScrollLayout支持的属性和接口回调：
1. isScrollDownWhenFirstItemIsTop 在往下滑的时候，是否只用当主体置顶时，头部才能下来
2. isAutoScroll 头部是否支持自动滚动到最上或者最下
3. headerRetainHeight 头部保留的高度，常见的使用比如头部布局最下方有个SmartTabLayout,为了让SmartTabLayout吸附在顶部，设置headerRetainHeight为SmartTabLayout的高度。
4. VerticalNestedScrollLayout还支持滚动中、滚动到顶部、底部的回调。


```
public interface OnScrollYListener {
        void onScrolling(int scrollY, boolean isTop);

        void onScrollToTop();

        void onScrollToBottom();
    }
```

#### 常见的问题：
1. 第一个子View用了普通的ViewGroup（如LinearLayout）,导致头部不能滑动，只能滑动下方主体部分。此时需要使用NestedScrollView来代替普通的ViewGroup
2. 第一个子View中有支持横向滑动的RecyclerView，横向滑动和竖向滑动产生嵌套滚动，导致横向滑动时也可以竖向滑动，此时需要禁用横向滑动的RecyclerView的嵌套滚动。

```mRecyclerView.setNestedScrollingEnabled(false); ```

#### VerticalNestedScrollLayout实现原理

VerticalNestedScrollLayout是继承LinearLayout实现NestedScrollingParent的父嵌套滚动组件，在initFromAttributes方法里设置其方向为垂直，并且获取布局中的属性。三个属性也可以通过set⋯⋯方法进行设置

```java
private void initFromAttributes(Context context, AttributeSet attrs, int defStyleAttr) {
    setOrientation(LinearLayout.VERTICAL);
    mParentHelper = new NestedScrollingParentHelper(this);

    TypedArray a = context.obtainStyledAttributes(attrs, com.kaola.base.R.styleable.VerticalNestedScrollLayout,
            defStyleAttr, 0);
    mIsScrollDownWhenFirstItemIsTop =
            a.getBoolean(R.styleable.VerticalNestedScrollLayout_isScrollDownWhenFirstItemIsTop, false);
    mIsAutoScroll = a.getBoolean(R.styleable.VerticalNestedScrollLayout_isAutoScroll, false);
    mHeaderRetainHeight = (int) a.getDimension(R.styleable.VerticalNestedScrollLayout_headerRetainHeight, 0);
    a.recycle();
}
```

通过onFinishInflate方法获取头部（mHeaderView）和主体（mBodyView）

```java
@Override
protected void onFinishInflate() {
    super.onFinishInflate();
    mHeaderView = getChildAt(0);
    mBodyView = getChildAt(1);
}
```

并且在addView方法中限制了添加View。

```java
@Override
public void addView(View child, int index, ViewGroup.LayoutParams params) {
    if (getChildCount() > 1) {
        throw new IllegalStateException("VerticalNestedScrollLayout can host only two direct child");
    }
    super.addView(child, index, params);
}
```

然后是比较重要的测量方法，主要有以下几步：
1. 测量头部的高度
2. 获取最大滚动距离，为头部自动滚动做准备
3. 测量主体的高度
4. 设置VerticalNestedScrollLayout的高度

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    //如果不设置无限制高度，mHeaderView高度如果大于屏幕的高，将只会显示屏幕的高
    mHeaderView.measure(widthMeasureSpec, MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED));
    //最大滚动距离：头部减去保留的高度
    mMaxScrollHeight = mHeaderView.getMeasuredHeight() - mHeaderRetainHeight;
    //设置主体的高度：代码中设置match_parent
    if (mBodyView.getLayoutParams().height < getMeasuredHeight() - mHeaderRetainHeight) {
        mBodyView.getLayoutParams().height = getMeasuredHeight() - mHeaderRetainHeight;
    }
    //设置自身的高度
    setMeasuredDimension(getMeasuredWidth(), mBodyView.getLayoutParams().height + mHeaderView.getMeasuredHeight());
}
```

![测量前后.png](http://nos.netease.com/knowledge/9b172f7d-0bcc-4223-a516-69920ec9b72f?download=Popo截图201843161918.png) 

红框表示屏幕，测量后VerticalNestedScrollLayout的高度实际上是变高了，如果没测量就进行嵌套滚动，往上滑动时，底部会出现空白区域

下面就是NestedScrollingParent接口中方法的实现了，重点介绍onNestedPreScroll 和 onNestedPreFling方法。

```java
@Override
public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {

    if (canScroll(target, dy)) {
        scrollBy(0, dy);
        consumed[1] = dy;
        ⋯⋯
    }
    ⋯⋯
}
```

该方法是子View开始滚动之前，调用的，就是子View滚动前让父View先滚，这里需要判断父View是否要滚动。代码中
hiddenTop是隐藏头部的行为、showTop是展示头部的行为，满足其中一个，就需要滚动父View。
代码如下：

```java
private boolean canScroll(View target, int dy) {
    boolean hiddenTop = dy > 0 && getScrollY() < mMaxScrollHeight;
    boolean showTop = dy < 0 && getScrollY() > 0;
    if (mIsScrollDownWhenFirstItemIsTop) {
        showTop = showTop && !target.canScrollVertically(-1);
    }
    return hiddenTop || showTop;
}
```

如果执行consumed[1] = dy;说明父View消费了所有的垂直滑动距离，如果consumed[1] = dy * 0.5f;则父View消费一半，这样用户看到的就是头部和主体部分同时滚动的视觉效果。

```java
@Override
public boolean onNestedPreFling(View target, float velocityX, float velocityY) {
    if (mIsScrollDownWhenFirstItemIsTop && target.canScrollVertically(-1)) {
        return false;
    }

    if (mScrollAnimator != null && mScrollAnimator.isStarted()) {
        mScrollAnimator.cancel();
    }
    mIsFling = true;
    if (velocityX == 0 && velocityY != 0) {
        if (velocityY < 0) {
            autoDownScroll();
        } else {
            autoUpScroll();
        }
        if (mIsScrollDownWhenFirstItemIsTop) {
            return true;
        }
    }
    return false;
}
```

上面是Fling时的处理逻辑，主要实现了自动滚动，如果没有这段，则头部看起来没有惯性，用户体检较差。

方法中canScrollVertically(-1)判断了target是否可以往下拉。比如RecyclerView没有置顶，还可以往下拉，mRecyclerView.canScrollVertically(-1)返回true

然后通过velocityY判断是自动滚到顶部还是底部；返回true表示父View消费了Fling事件，false则不消费。

### 嵌套滚动原理篇·内部实现

嵌套滚动中的两个接口，在上文中已经提到。NestedScrollingParent和NestedScrollingChild
接口中的方法如下：

NestedScrollingChild

* startNestedScroll : 起始方法, 主要作用是找到接收滑动距离信息的外控件.
* dispatchNestedPreScroll : 在内控件处理滑动前把滑动信息分发给外控件.
* dispatchNestedScroll : 在内控件处理完滑动后把剩下的滑动距离信息分发给外控件.
* stopNestedScroll : 结束方法, 主要作用就是清空嵌套滑动的相关状态
* setNestedScrollingEnabled和isNestedScrollingEnabled : 一对get&set方法, 用来判断控件是否支持嵌套滑动.
* dispatchNestedPreFling和dispatchNestedFling : 跟Scroll的对应方法作用类似, 

NestedScrollingParent

* onStartNestedScroll : 对应startNestedScroll, 内控件通过调用外控件的这个方法来确定外控件是否接收滑动信息.
* onNestedScrollAccepted : 当外控件确定接收滑动信息后该方法被回调, 可以让外控件针对嵌套滑动做一些前期工作.
* onNestedPreScroll : 关键方法, 接收内控件处理滑动前的滑动距离信息, 在这里外控件可以优先响应滑动操作, 消耗部分或者全部滑动距离.
* onNestedScroll : 关键方法, 接收内控件处理完滑动后的滑动距离信息, 在这里外控件可以选择是否处理剩余的滑动距离.
* onStopNestedScroll : 对应stopNestedScroll, 用来做一些收尾工作.
* getNestedScrollAxes : 返回嵌套滑动的方向, 区分横向滑动和竖向滑动, 作用不大
* onNestedPreFling和onNestedFling : 同上略


#### 嵌套滚动的过程：

子view接受到滚动事件后发起嵌套滚动，询问父View是否要先滚动，父View处理了自己的滚动需求后，回到子View处理自己的滚动需求，假如父View消耗了一些滚动距离，子View只能获取剩下的滚动距离做处理。子View处理了自己的滚动需求后又回到父View，剩下的滚动距离做处理。惯性fling的类似。

将上面过程用源码来解释（子View为RecyclerView，父View为继承了NestedScrollingParent的视图）大体如下：

NestedScrollingChild 的 startNestedScroll是嵌套滚动的发起，查看RecyclerView中该方法的调用地方，在onInterceptTouchEvent和onTouchEvent的action ==MotionEvent.ACTION_DOWN时，忽略onInterceptTouchEvent，直接看onTouchEvent。

查看RecyclerView的startNestedScroll，发现是调了NestedScrollingChildHelper里的startNestedScroll方法，查看startNestedScroll，发现有个遍历的过程，找到onStartNestedScroll返回true的父View，再执行onNestedScrollAccepted后停止遍历。到目前嵌套滚动执行的方法顺序如下：

**(子)startNestedScroll → (父)onStartNestedScroll → (父)onNestedScrollAccepted**


```java
public boolean startNestedScroll(@ScrollAxis int axes, @NestedScrollType int type) {
    if (hasNestedScrollingParent(type)) {
        // Already in progress
        return true;
    }
    if (isNestedScrollingEnabled()) {
        ViewParent p = mView.getParent();
        View child = mView;
        while (p != null) {
            if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes, type)) {
                setNestedScrollingParentForType(type, p);
                ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes, type);
                return true;
            }
            if (p instanceof View) {
                child = (View) p;
            }
            p = p.getParent();
        }
    }
    return false;
}

```
 
接下来在RecyclerView的onTouchEvent的 MotionEvent.ACTION_MOVE里调用了dispatchNestedPreScroll和scrollByInternal

```java
case MotionEvent.ACTION_MOVE: {
   ⋯⋯
    if (dispatchNestedPreScroll(dx, dy, mScrollConsumed, mScrollOffset, TYPE_TOUCH)) {
        dx -= mScrollConsumed[0];
        dy -= mScrollConsumed[1];
        vtev.offsetLocation(mScrollOffset[0], mScrollOffset[1]);
        // Updated the nested offsets
        mNestedOffsets[0] += mScrollOffset[0];
        mNestedOffsets[1] += mScrollOffset[1];
    }
    ⋯⋯

    if (mScrollState == SCROLL_STATE_DRAGGING) {
        mLastTouchX = x - mScrollOffset[0];
        mLastTouchY = y - mScrollOffset[1];

        if (scrollByInternal(
                canScrollHorizontally ? dx : 0,
                canScrollVertically ? dy : 0,
                vtev)) {
            getParent().requestDisallowInterceptTouchEvent(true);
        }
       ⋯⋯
    }
} break;
```

看dispatchNestedPreScroll源码：发现调了父View的onNestedPreScroll，并且传入dy 和 consumed。用于做消费计数。

onNestedPreScroll事件在不同父View中有不同实现，具体可以看一下VerticalNestedScrollLayout里该方法的实现

```java
public boolean dispatchNestedPreScroll(int dx, int dy, @Nullable int[] consumed,
            @Nullable int[] offsetInWindow, @NestedScrollType int type) {
    if (isNestedScrollingEnabled()) {
        final ViewParent parent = getNestedScrollingParentForType(type);
        if (parent == null) {
            return false;
        }

        if (dx != 0 || dy != 0) {
            ⋯⋯
            consumed[0] = 0;
            consumed[1] = 0;
            ViewParentCompat.onNestedPreScroll(parent, mView, dx, dy, consumed, type);

            ⋯⋯
            return consumed[0] != 0 || consumed[1] != 0;
        } else if (offsetInWindow != null) {
            offsetInWindow[0] = 0;
            offsetInWindow[1] = 0;
        }
    }
    return false;
}
```

scrollByInternal让RecyclerView自己滚动后又调用了dispatchNestedScroll

```java
boolean scrollByInternal(int x, int y, MotionEvent ev) {
        ⋯⋯
        if (y != 0) {
            consumedY = mLayout.scrollVerticallyBy(y, mRecycler, mState);
            unconsumedY = y - consumedY;
        }
        ⋯⋯
    if (dispatchNestedScroll(consumedX, consumedY, unconsumedX, unconsumedY, mScrollOffset,
            TYPE_TOUCH)) {
        // Update the last touch co-ords, taking any scroll offset into account
        mLastTouchX -= mScrollOffset[0];
        mLastTouchY -= mScrollOffset[1];
        if (ev != null) {
            ev.offsetLocation(mScrollOffset[0], mScrollOffset[1]);
        }
        mNestedOffsets[0] += mScrollOffset[0];
        mNestedOffsets[1] += mScrollOffset[1];
    }⋯⋯
    return consumedX != 0 || consumedY != 0;
}
```


看dispatchNestedScroll方法,最终调用了父View的onNestedScroll方法。

```java
public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed, @Nullable int[] offsetInWindow,
            @NestedScrollType int type) {
        if (isNestedScrollingEnabled()) {
            final ViewParent parent = getNestedScrollingParentForType(type);
            if (parent == null) {
                return false;
            }

            if (dxConsumed != 0 || dyConsumed != 0 || dxUnconsumed != 0 || dyUnconsumed != 0) {
                ⋯⋯

                ViewParentCompat.onNestedScroll(parent, mView, dxConsumed,
                        dyConsumed, dxUnconsumed, dyUnconsumed, type);

                return true;
            } else if (offsetInWindow != null) {
                // No motion, no dispatch. Keep offsetInWindow up to date.
                offsetInWindow[0] = 0;
                offsetInWindow[1] = 0;
            }
        }
        return false;
    }
```

到目前我们也可以看到父View的嵌套滚动方法都是子View调起来的，子View的接口都在TouchEvent事件里。嵌套滚动执行的方法顺序如下：

**(子)startNestedScroll → (父)onStartNestedScroll → (父)onNestedScrollAccepted→ (子)dispatchNestedPreScroll → (父)onNestedPreScroll→ (子)dispatchNestedScroll→ (父)onNestedScroll**


后面的MotionEvent.ACTION_UP中:

调用fling方法执行了嵌套滚动相关的fling事件
resetTouch();执行了stopNestedScroll事件

过程类似不在赘述。
嵌套滚动执行的方法顺序如下：

**(子)startNestedScroll → (父)onStartNestedScroll → (父)onNestedScrollAccepted→ (子)dispatchNestedPreScroll → (父)onNestedPreScroll→ (子)dispatchNestedScroll→ (父)onNestedScroll→ (子)dispatchNestedPreFling → (父)onNestedPreFling→ (子)dispatchNestedFling → (父)stopNestedScroll**


### 辅助类NestedScrollingChildHelper和NestedScrollingParentHelper


从LOLLIPOP(SDK21)开始，嵌套滑动的相关逻辑作为普通方法直接写进了View和ViewGroup类里。而SDK21之前的版本
官方在android.support.v4兼容包中提供了两个接口NestedScrollingChild和NestedScrollingParent, 还有两个辅助类NestedScrollingChildHelper和NestedScrollingParentHelper来帮助控件实现嵌套滑动。

兼容的原理

两个接口NestedScrollingChild和NestedScrollingParent分别定义上面提到的View和ViewParent新增的普通方法

在嵌套滑动中会要求控件要么是继承于SDK21之后的View或ViewGroup, 要么实现了这两个接口, 这是控件能够进行嵌套滑动的前提条件。

那么怎么知道调用的方法是控件自有的方法, 还是接口的方法? 在代码中是通过ViewCompat和ViewParentCompat类来实现.

ViewCompat和ViewParentCompat通过当前的Build.VERSION.SDK_INT来判断当前版本, 然后选择不同的实现类, 这样就可以根据版本选择调用的方法.

例如如果版本是SDK21之前, 那么就会判断控件是否实现了接口, 然后调用接口的方法, 如果是SDK21之后, 那么就可以直接调用对应的方法。

>参考：https://www.jianshu.com/p/1806ed9737f6
