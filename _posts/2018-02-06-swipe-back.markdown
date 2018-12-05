---
layout:     post
title:      "考拉 Android 全局滑动返回及联动效果的实现"
subtitle:   "滑动返回在 Android 的方案及实践"
date:       2018-02-06
author:     "许方镇"
header-img: "img/bg7.jpg"
tags:
    Android
    SwipeBack
---

## 前言
首次通过右滑来返回到上一个页面的操作是在 IOS7上出现。到目前android应用上支持这种操作的依然不多。分析其主要原因应该是android已有实体的返回按键，这样的功能变得不重要，但我觉得有这样的功能便于单手操作，能提升app的用户体验，特别是从ios转到android的用户。写这篇博文希望可以对大家有所帮助，希望自己的app上有滑动返回功能的可以参考下。

## 原理的简单描述
Android系统里有很多滑动相关的API和类，比如ViewDragHelper就是一个很好的滑动助手类。首先设置Window的背景为透明，再通过ViewDragHelper对Activity上DecorView的子view进行滑动，当滑动到一定距离，手指离开后就自动滑到最右侧，然后finish当前的activity，这样即可实现滑动返回效果。为了能够 “全局的”、“联动的” 实现滑动返回效果，在每个activity的DecorView下插入了SwipeBackLayout，当前activity滑动和下层activity的联动都在该类中完成。

## 效果图
 ![下层联动滑动返回](http://nos.netease.com/knowledge/9ae17fc9-fef4-45c4-8675-41bb5523037b) 

## 布局图
 ![Hierarchy View](http://nos.netease.com/knowledge/be2b7969-5ec3-41a6-ab04-96a550597154) 

## 实现主要类：
SwipeBackActivity //滑动返回基类<br>
SwipeBackLayout //滑动返回布局类<br>
SwipeBackLayoutDragHelper  //修改ViewDragHelper后助手类<br>
TranslucentHelper //代码中修改透明或者不透明的助手类<br>

## 代码层面的讲解
### 一. 设置activity为透明、activity跳转动画（TranslucentHelper 讲解）
这个看起来很简单，但如果要兼容到API16及以下，会遇到过一个比较麻烦的页面切换动画问题：

#### 1.1、通过activity的主题style进行设置
```
<item name="android:windowBackground">@color/transparent</item>
<item name="android:windowIsTranslucent">true</item>
```

**遇到问题：** 如果在某个activity的主题style中设置了android:windowIsTranslucent属性为true，那么该activity切换动画与没设置之前是不同的，有些手机切换动画会变得非常跳。所以需要自定义activity的切换动画。
接下来我们会想到通过主题style里的windowAnimationStyle来设置切换动画
```
<item name="android:activityOpenEnterAnimation">@anim/activity_open_enter</item><!-- activity 新创建时进来的动画-->
<item name="android:activityOpenExitAnimation">@anim/activity_open_exit</item><!-- 上层activity返回后，下层activity重新出现的动画-->
<item name="android:activityCloseEnterAnimation">@anim/activity_close_enter</item><!-- 跳到新的activity后，该activity被隐藏时的动画-->
<item name="android:activityCloseExitAnimation">@anim/activity_close_exit</item><!-- activity 销毁时的动画-->
```
**实践证明：** 当android:windowIsTranslucent为true时，以上几个属性是无效的，而下面两个属性还是可以用。但是这两个属性一个是窗口进来动画，一个是窗口退出动画，明显是不够。
```
<item name="android:windowEnterAnimation">@anim/***</item>
<item name="android:windowExitAnimation">@anim/***</item>
```

结合overridePendingTransition(int enterAnim, int exitAnim)可以复写窗口进来动画和窗口退出动画，这种我觉得最终可能是可以实现的，不过控制起来比较复杂：
比如有A、B、C三个页面：
A跳到B，进场页面B动画从右进来，出场页面A动画从左出去，可以直接在style中写死
```
<item name="android:windowEnterAnimation">@anim/***</item>
<item name="android:windowExitAnimation">@anim/***</item>
```

如果B返回到A，进场页面A动画从左进来，出场页面B动画从右出去，此时需要通过复写onBackPressed() 方法，<br>
在其中添加overridePendingTransition(int enterAnim, int exitAnim)方法来改变动画。

如果B是finish()后到A页面，在finish()后面加上overridePendingTransition ……<br>
由于onBackPressed() 方法最终会调finish()，所以实际上只需要复写finish()，在其中添加overridePendingTransition……

但是假如B finish()后跳到C，则又不应该执行overridePendingTransition……，那么就需要判断finish执行后是否要加 overridePendingTransition……<br>
对于一个较为庞大的项目，采取这种方法需要对每个页面进行排查，因此是不可行的，而对于刚刚起步的应用来说则是一个选择。<br>

#### 1.2、通过透明助手类（TranslucentHelper）进行设置

透明助手类（TranslucentHelper）里主要又有两个方法，一个是让activity变不透明，一个是让activity变透明，这两个都是通过反射来调用隐藏的系统api来实现的。因为较低的版本不支持代码中修改背景透明不透明，所以在类中有个静态变量mTranslucentState 来记录是否可以切换背景，这样低版本就不需要每次都反射通过捕获到的异常来做兼容方案。
另外：发现有些手机支持背景变黑，但不支持背景变透明（中兴z9 mini 5.0.2系统）

```java
public class TranslucentHelper {
    private static final String TRANSLUCENT_STATE = "translucentState";
    private static final int INIT = 0;//表示初始
    private static final int CHANGE_STATE_FAIL = INIT + 1;//表示确认不可以切换透明状态
    private static final int CHANGE_STATE_SUCCEED = CHANGE_STATE_FAIL + 1;//表示确认可以切换透明状态
    private static int mTranslucentState = INIT;

    interface TranslucentListener {
        void onTranslucent();
    }

    private static class MyInvocationHandler implements InvocationHandler {
        private TranslucentListener listener;

        MyInvocationHandler(TranslucentListener listener) {
            this.listener = listener;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            try {
                boolean success = (boolean) args[0];
                if (success && listener != null) {
                    listener.onTranslucent();
                }
            } catch (Exception ignored) {
            }
            return null;
        }
    }

    static boolean convertActivityFromTranslucent(Activity activity) {
        if (mTranslucentState == INIT) {
            mTranslucentState = PreferencesUtils.getInt(TRANSLUCENT_STATE, INIT);
        }
        if (mTranslucentState == INIT) {
            convertActivityToTranslucent(activity, null);
        } else if (mTranslucentState == CHANGE_STATE_FAIL) {
            return false;
        }

        try {
            Method method = Activity.class.getDeclaredMethod("convertFromTranslucent");
            method.setAccessible(true);
            method.invoke(activity);
            mTranslucentState = CHANGE_STATE_SUCCEED;
            return true;
        } catch (Throwable t) {
            mTranslucentState = CHANGE_STATE_FAIL;
            PreferencesUtils.saveInt(TRANSLUCENT_STATE, CHANGE_STATE_FAIL);
            return false;
        }
    }

    static void convertActivityToTranslucent(Activity activity, final TranslucentListener listener) {
        if (mTranslucentState == CHANGE_STATE_FAIL) {
            if (listener != null) {
                listener.onTranslucent();
            }
            return;
        }

        try {
            Class<?>[] classes = Activity.class.getDeclaredClasses();
            Class<?> translucentConversionListenerClazz = null;
            for (Class clazz : classes) {
                if (clazz.getSimpleName().contains("TranslucentConversionListener")) {
                    translucentConversionListenerClazz = clazz;
                }
            }

            MyInvocationHandler myInvocationHandler = new MyInvocationHandler(listener);
            Object obj = Proxy.newProxyInstance(Activity.class.getClassLoader(),
                    new Class[] { translucentConversionListenerClazz }, myInvocationHandler);

            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
                Method getActivityOptions = Activity.class.getDeclaredMethod("getActivityOptions");
                getActivityOptions.setAccessible(true);
                Object options = getActivityOptions.invoke(activity);

                Method method = Activity.class.getDeclaredMethod("convertToTranslucent",
                        translucentConversionListenerClazz, ActivityOptions.class);
                method.setAccessible(true);
                method.invoke(activity, obj, options);
            } else {
                Method method =
                        Activity.class.getDeclaredMethod("convertToTranslucent", translucentConversionListenerClazz);
                method.setAccessible(true);
                method.invoke(activity, obj);
            }
            mTranslucentState = CHANGE_STATE_SUCCEED;
        } catch (Throwable t) {
            mTranslucentState = CHANGE_STATE_FAIL;
            PreferencesUtils.saveInt(TRANSLUCENT_STATE, CHANGE_STATE_FAIL);
            new Handler().postDelayed(new Runnable() {
                @Override
                public void run() {
                    if (listener != null) {
                        listener.onTranslucent();
                    }
                }
            }, 100);
        }
    }
}
```
让activity变不透明的方法比较简单；让activity变透明的方法参数里传入了一个listener接口 ，主要是当antivity变透明后会回调，因为这个接口也在activity里，而且是私有的，所以我们只能通过动态代理去获取这个回调。最后如果版本大于等于5.0，还需要再传入一个ActivityOptions参数。

在实际开发中，这两个方法在android 5.0以上是有效的，在5.0以下需要当android:windowIsTranslucent为true时才有效，这样又回到了之前的问题activity切换动画异常。

**最终决解方法：**setContentView之前就调用 convertActivityFromTranslucent方法，让activity背景变黑，这样activity切换效果就正常。

**总结：**在style中设置android:windowIsTranslucent为true ，setContentView之前就调用 convertActivityFromTranslucent方法，当触发右滑时调用convertActivityToTranslucent，通过动态代理获取activity变透明后的回调，在回调后允许开始滑动。


### 二. 让BaseActivity继承SwipeBackActivity（SwipeBackActivity讲解）
先直接看代码，比较少
```java
public abstract class SwipeBackActivity extends CoreBaseActivity {
    /**
     * 滑动返回View
     */
    private SwipeBackLayout mSwipeBackLayout;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        if (!isSwipeBackDisableForever()) {
            TranslucentHelper.convertActivityFromTranslucent(this);
            mSwipeBackLayout = new SwipeBackLayout(this);
        }
    }

    @Override
    protected void onPostCreate(Bundle savedInstanceState) {
        super.onPostCreate(savedInstanceState);
        if (!isSwipeBackDisableForever()) {
            mSwipeBackLayout.attachToActivity(this);
            mSwipeBackLayout.setOnSwipeBackListener(new SwipeBackLayout.onSwipeBackListener() {
                @Override
                public void onStart() {
                    onSwipeBackStart();
                }

                @Override
                public void onEnd() {
                    onSwipeBackEnd();
                }
            });
        }
    }

    @Override
    public void onWindowFocusChanged(boolean hasFocus) {
        super.onWindowFocusChanged(hasFocus);
        if (!isSwipeBackDisableForever() && hasFocus) {
            getSwipeBackLayout().recovery();
        }
    }

    /**
     * 滑动返回开始时的回调
     */
    protected void onSwipeBackStart() {}

    /**
     * 滑动返回结束时的回调
     */
    protected void onSwipeBackEnd() {}

    /**
     * 设置是否可以边缘滑动返回，需要在onCreate方法调用
     */
    public void setSwipeBackEnable(boolean enable) {
        if (mSwipeBackLayout != null) {
            mSwipeBackLayout.setSwipeBackEnable(enable);
        }
    }

    public boolean isSwipeBackDisableForever() {
        return false;
    }

    public SwipeBackLayout getSwipeBackLayout() {
        return mSwipeBackLayout;
    }
}
```
SwipeBackActivity中包含了一个SwipeBackLayout对象 

在onCreate方法中：<br>
1.activity转化为不透明。<br>
2.new了一个SwipeBackLayout。

在onPostCreate方法中：<br>
1.attachToActivity主要是插入SwipeBackLayout、窗口背景设置……<br>
2.设置了滑动返回开始和结束的监听接口，建议在滑动返回开始时，把PopupWindow给dismiss掉。

onWindowFocusChanged 方法中<br>
如果是hasFocus == true，就recovery()这个SwipeBackLayout，这个也是因为下层activity有联动效果而移动了SwipeBackLayout，所以需要recovery()下，防止异常情况。

isSwipeBackDisableForever 方法是一个大开关，默认返回false，在代码中复写后返回 true，则相当于直接继承了SwipeBackActivity的父类。

setSwipeBackEnable 方法是一个小开关，设置了false之后就暂时不能滑动返回了，可以在特定的时机设置为true，就恢复滑动返回的功能。

**总结说明：**下层activity设置了setSwipeBackEnable 为 false，上层activity滑动时还是可以联动的，比如MainActivity。而isSwipeBackDisableForever 返回true就不会联动了，而且一些仿PopupWindow的activity需要复写这个方法，因为activity需要透明。

### 三、滑动助手类的使用和滑动返回布局类的实现（SwipeBackLayout讲解）

直接贴SwipeBackLayout源码：
```java
/**
 * 滑动返回容器类
 */
public class SwipeBackLayout extends FrameLayout {
    /**
     * 滑动销毁距离比例界限，滑动部分的比例超过这个就销毁
     */
    private static final float DEFAULT_SCROLL_THRESHOLD = 0.5f;
    /**
     * 滑动销毁速度界限，超过这个速度就销毁
     */
    private static final float DEFAULT_VELOCITY_THRESHOLD = ScreenUtils.dpToPx(250);
    /**
     * 最小滑动速度
     */
    private static final int MIN_FLING_VELOCITY = ScreenUtils.dpToPx(200);
    /**
     * 左边移动的像素值
     */
    private int mContentLeft;
    /**
     * 左边移动的像素值 / （ContentView的宽+阴影）
     */
    private float mScrollPercent;
    /**
     * （ContentView可见部分+阴影）的比例 （即1 - mScrollPercent）
     */
    private float mContentPercent;
    /**
     * 阴影图
     */
    private Drawable mShadowDrawable;
    /**
     * 阴影图的宽
     */
    private int mShadowWidth;
    /**
     * 内容view，DecorView的原第一个子view
     */
    private View mContentView;
    /**
     * 用于记录ContentView所在的矩形
     */
    private Rect mContentViewRect = new Rect();
    /**
     * 设置是否可滑动
     */
    private boolean mIsSwipeBackEnable = true;
    /**
     * 是否正在放置
     */
    private boolean mIsLayout = true;
    /**
     * 判断背景Activity是否启动进入动画
     */
    private boolean mIsEnterAnimRunning = false;
    /**
     * 是否是透明的
     */
    private boolean mIsActivityTranslucent = false;
    /**
     * 进入动画(只在释放手指时使用)
     */
    private ObjectAnimator mEnterAnim;
    /**
     * 退拽助手类
     */
    private SwipeBackLayoutDragHelper mViewDragHelper;
    /**
     * 执行滑动时的最顶层Activity
     */
    private Activity mTopActivity;
    /**
     * 后面的Activity的弱引用
     */
    private WeakReference<Activity> mBackActivityWeakRf;
    /**
     * 监听滑动开始和结束
     */
    private onSwipeBackListener mListener;

    public SwipeBackLayout(Context context) {
        super(context);
        init(context);
    }

    public SwipeBackLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
        init(context);
    }

    public SwipeBackLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init(context);
    }

    private void init(Context context) {
        mViewDragHelper = SwipeBackLayoutDragHelper.create(SwipeBackLayout.this, new ViewDragCallback());
        mViewDragHelper.setEdgeTrackingEnabled(SwipeBackLayoutDragHelper.EDGE_LEFT);
        mViewDragHelper.setMinVelocity(MIN_FLING_VELOCITY);
        mViewDragHelper.setMaxVelocity(MIN_FLING_VELOCITY * 2);
        try {
            mShadowDrawable = context.getResources().getDrawable(R.drawable.swipeback_shadow_left);
        } catch (Exception ignored) {
        }
    }

    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        try {
            if (!mIsSwipeBackEnable) {
                super.onLayout(changed, left, top, right, bottom);
                return;
            }
            mIsLayout = true;
            if (mContentView != null) {
                mContentView.layout(mContentLeft, top, mContentLeft + mContentView.getMeasuredWidth(),
                        mContentView.getMeasuredHeight());
            }
            mIsLayout = false;
        } catch (Exception e) {
            super.onLayout(changed, left, top, right, bottom);
        }
    }

    @Override
    public void requestLayout() {
        if (!mIsLayout || !mIsSwipeBackEnable) {
            super.requestLayout();
        }
    }

    @Override
    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        try {
            //绘制阴影
            if (mContentPercent > 0
                    && mShadowDrawable != null
                    && child == mContentView
                    && mViewDragHelper.getViewDragState() != SwipeBackLayoutDragHelper.STATE_IDLE) {
                child.getHitRect(mContentViewRect);
                mShadowWidth = mShadowDrawable.getIntrinsicWidth();
                mShadowDrawable.setBounds(mContentViewRect.left - mShadowWidth, mContentViewRect.top,
                        mContentViewRect.left, mContentViewRect.bottom);
                mShadowDrawable.draw(canvas);
            }
            return super.drawChild(canvas, child, drawingTime);
        } catch (Exception e) {
            return super.drawChild(canvas, child, drawingTime);
        }
    }

    @Override
    public void computeScroll() {
        mContentPercent = 1 - mScrollPercent;
        if (mViewDragHelper.continueSettling(true)) {
            ViewCompat.postInvalidateOnAnimation(this);
        }
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        if (!mIsSwipeBackEnable) {
            return false;
        }
        try {
            return mViewDragHelper.shouldInterceptTouchEvent(event);
        } catch (ArrayIndexOutOfBoundsException e) {
            return super.onInterceptTouchEvent(event);
        }
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        if (!mIsSwipeBackEnable) {
            return false;
        }
        try {
            mViewDragHelper.processTouchEvent(event);
            return true;
        } catch (Exception e) {
            return super.onTouchEvent(event);
        }
    }

    /**
     * 将View添加到Activity
     */
   public void attachToActivity(Activity activity) {
        //插入SwipeBackLayout
        ViewGroup decor = (ViewGroup) activity.getWindow().getDecorView();
        ViewGroup decorChild = (ViewGroup) decor.getChildAt(0);
        decor.removeView(decorChild);
        if (getParent() != null) {
            decor.removeView(this);
        }
        decor.addView(this);
        this.removeAllViews();
        this.addView(decorChild);

        //mContentView为SwipeBackLayout的直接子view,获取window背景色进行赋值
        activity.getWindow().setBackgroundDrawableResource(R.color.transparent_white);
        TypedArray a = activity.getTheme().obtainStyledAttributes(new int[] {
                android.R.attr.windowBackground
        });
        mContentView = decorChild;
        mContentView.setBackgroundResource(a.getResourceId(0, 0));
        a.recycle();

        //拿到顶层activity和下层activity，做联动操作
        mTopActivity = activity;
        Activity backActivity = ActivityUtils.getSecondTopActivity();
        if (backActivity != null && backActivity instanceof SwipeBackActivity) {
            if (!((SwipeBackActivity) backActivity).isSwipeBackDisableForever()) {
                mBackActivityWeakRf = new WeakReference<>(backActivity);
            }
        }
    }

    /**
     * 设置是否可以滑动返回
     */
    public void setSwipeBackEnable(boolean enable) {
        mIsSwipeBackEnable = enable;
    }

    public boolean isActivityTranslucent() {
        return mIsActivityTranslucent;
    }

    /**
     * 启动进入动画
     */
    private void startEnterAnim() {
        if (mContentView != null) {
            ObjectAnimator anim =
                    ObjectAnimator.ofFloat(mContentView, "TranslationX", mContentView.getTranslationX(), 0f);
            anim.setDuration((long) (125 * mContentPercent));
            mEnterAnim = anim;
            mEnterAnim.start();
        }
    }

    protected View getContentView() {
        return mContentView;
    }

    private class ViewDragCallback extends SwipeBackLayoutDragHelper.Callback {
        @Override
        public boolean tryCaptureView(View child, int pointerId) {
            if (mIsSwipeBackEnable && mViewDragHelper.isEdgeTouched(SwipeBackLayoutDragHelper.EDGE_LEFT, pointerId)) {
                TranslucentHelper.convertActivityToTranslucent(mTopActivity,
                        new TranslucentHelper.TranslucentListener() {
                            @Override
                            public void onTranslucent() {
                                if (mListener != null) {
                                    mListener.onStart();
                                }
                                mIsActivityTranslucent = true;
                            }
                        });
                return true;
            }
            return false;
        }

        @Override
        public int getViewHorizontalDragRange(View child) {
            return mIsSwipeBackEnable ? SwipeBackLayoutDragHelper.EDGE_LEFT : 0;
        }

        @Override
        public void onViewPositionChanged(View changedView, int left, int top, int dx, int dy) {
            super.onViewPositionChanged(changedView, left, top, dx, dy);
            if (changedView == mContentView) {
                mScrollPercent = Math.abs((float) left / mContentView.getWidth());
                mContentLeft = left;
                //未执行动画就平移
                if (!mIsEnterAnimRunning) {
                    moveBackActivity();
                }
                invalidate();
                if (mScrollPercent >= 1 && !mTopActivity.isFinishing()) {
                    if (mBackActivityWeakRf != null && ActivityUtils.activityIsAlive(mBackActivityWeakRf.get())) {
                        ((SwipeBackActivity) mBackActivityWeakRf.get()).getSwipeBackLayout().invalidate();
                    }
                    mTopActivity.finish();
                    mTopActivity.overridePendingTransition(0, 0);
                }
            }
        }

        @Override
        public void onViewReleased(View releasedChild, float xvel, float yvel) {
            if (xvel > DEFAULT_VELOCITY_THRESHOLD || mScrollPercent > DEFAULT_SCROLL_THRESHOLD) {
                if (mIsActivityTranslucent) {
                    mViewDragHelper.settleCapturedViewAt(releasedChild.getWidth() + mShadowWidth, 0);
                    if (mContentPercent < 0.85f) {
                        startAnimOfBackActivity();
                    }
                }
            } else {
                mViewDragHelper.settleCapturedViewAt(0, 0);
            }
            if (mListener != null) {
                mListener.onEnd();
            }
            invalidate();
        }

        @Override
        public int clampViewPositionHorizontal(View child, int left, int dx) {
            return Math.min(child.getWidth(), Math.max(left, 0));
        }

        @Override
        public void onViewDragStateChanged(int state) {
            super.onViewDragStateChanged(state);

            if (state == SwipeBackLayoutDragHelper.STATE_IDLE && mScrollPercent < 1f) {
                TranslucentHelper.convertActivityFromTranslucent(mTopActivity);
                mIsActivityTranslucent = false;
            }
        }

        @Override
        public boolean isTranslucent() {
            return SwipeBackLayout.this.isActivityTranslucent();
        }
    }

    /**
     * 背景Activity开始进入动画
     */
    private void startAnimOfBackActivity() {
        if (mBackActivityWeakRf != null && ActivityUtils.activityIsAlive(mBackActivityWeakRf.get())) {
            mIsEnterAnimRunning = true;
            SwipeBackLayout swipeBackLayout = ((SwipeBackActivity) mBackActivityWeakRf.get()).getSwipeBackLayout();
            swipeBackLayout.startEnterAnim();
        }
    }

    /**
     * 移动背景Activity
     */
    private void moveBackActivity() {
        if (mBackActivityWeakRf != null && ActivityUtils.activityIsAlive(mBackActivityWeakRf.get())) {
            View view = ((SwipeBackActivity) mBackActivityWeakRf.get()).getSwipeBackLayout().getContentView();
            if (view != null) {
                int width = view.getWidth();
                view.setTranslationX(-width * 0.3f * Math.max(0f, mContentPercent - 0.15f));
            }
        }
    }

    /**
     * 回复界面的平移到初始位置
     */
    public void recovery() {
        if (mEnterAnim != null && mEnterAnim.isRunning()) {
            mEnterAnim.end();
        } else {
            mContentView.setTranslationX(0);
        }
    }

    interface onSwipeBackListener {
        void onStart();

        void onEnd();
    }

    public void setOnSwipeBackListener(onSwipeBackListener listener) {
        mListener = listener;
    }
}
```

* attachToActivity<br>
上面讲到SwipeBackLayout是在activity的onCreate时被创建，在onPostCreate是插入到DecorView里，主要是因为DecorView是在setContentView时与Window关联起来插入SwipeBackLayout方法如代码所示，不难，然后是设置了window的背景色为透明色，mContentView为SwipeBackLayout的直接子view,获取window背景色进行赋值。由于考拉项目已经有很多activity，而这些activity中android:windowBackground设置的颜色大部分是白色，少部分是灰色和透明的，所以需要在代码中设置统一设置一遍透明的，原来的背景色则赋值给SwipeBackLayout的子View就可以达到最少修改代码的目的。最后拿到顶层activity和下层activity，做联动操作

* SwipeBackLayoutDragHelper 和 ViewDragCallback <br>
SwipeBackLayout中包含了一个滑动助手类（SwipeBackLayoutDragHelper）的对象，该类是在ViewDragHelper的基础上进行修改得来的。<br>
修改点：<br>
1.右侧触发区域 EDGE_SIZE由 20dp 改到 25dp<br>
2.提供滑动最大速度 的设置方法<br>
3.在ViewDragHelper 的内部类Callback方法中提供是否activity为透明的回调接口<br>
4.在最终调用滑动的方法dragTo中添加判断逻辑，activity为透明时才支持滑动<br>
SwipeBackLayoutDragHelper 在init 方法中初始化，通过onInterceptTouchEvent和onTouchEvent拿到滑动事件，通过ViewDragCallback的一些方法返回相应的滑动回调，
ViewDragCallback实现了SwipeBackLayoutDragHelper.Callback里的以下几个接口，其中其中isTranslucent()是自己添加进去的。
 ![ViewDragCallback](http://nos.netease.com/knowledge/4c782465-c849-418d-9ec7-9b5e060daa69) 

* tryCaptureView方法当触摸到SwipeBackLayout里的子View时触发的，当返回true，表示捕捉成功，否则失败。判断条件是如果支持滑动返回并且是左侧边距被触摸时才可以，我们知道这个时候的的背景色是不透明的，如果直接开始滑动则是黑色的，所以需要在这里背景色改成透明的，如果直接调用 TranslucentHelper.convertActivityToTranslucent(mTopActivity, null)后直接返回true，会出现一个异常情况，就是滑动过快时会导致背景还来不及变成黑色就滑动出来了，之后才变成透明的，从而导致了会从黑色到透明的一个闪烁现象，解决的办法是在代码中用了一个回调和标记，当变成透明后设置了mIsActivityTranslucent = true;通过mIsActivityTranslucent 这个变量来判断是否进行移动的操作。由于修改activity变透明的方法是通过反射的，不能简单的设置一个接口后进行回调，而是通过动态代理的方式来实现的（InvocationHandler），在convertToTranslucent方法的第一个参数刚好是一个判断activity是否已经变成透明的回调，看下面代码中 if 语句里的注释和回调，如果窗口已经变成透明的话，就传了一个drawComplete (true)。通过动态代理，将translucentConversionListenerClazz 执行其方法onTranslucentConversionComplete的替换成myInvocationHandler中执行invoke方法。其中赋值给success的args[0]正是 drawComplete。

* isTranslucent是自己添加了一个方法，主要是返回activity是否是透明的默认为true，在SwipeBackLayout重写后将mIsActivityTranslucent返回。仔细看SwipeBackLayoutDragHelper方法的话，会发现最后通过dragTo方法对view进行移动，因此在进行水平移动前判断下是否是透明的，只有透明了才能移动

* onViewPositionChanged view移动过程中会持续调用，这里面的逻辑主要有这几个：
1.实时计算滑动了多少距离，用于绘制左侧阴影等<br>
2.使下面的activity进行移动moveBackActivity();<br>
3.当view完全移出屏幕后，销毁当前的activity<br>

* onViewReleased是手指释放后触发的一个方法。如果滑动速度大于最大速度或者滑动的距离大于设定的阈值距离，则直接移到屏幕外，同时触发下层activity的复位动画，否则移会到原来位置 。

* onViewDragStateChanged当滑动的状态发生改变时的回调，主要是停止滑动后，将背景改成不透明，这样跳到别的页面是动画就是正常的。

* clampViewPositionHorizontal 返回水平移动距离，防止滑出父 view。

*  getViewHorizontalDragRange对于clickable=true的子view，需要返回大于0的数字才能正常捕获。

其他方法都较为简单，注释也写了，就不多说了，最后毫不吝啬的贴上SwipeBackLayoutDragHelper的dragTo代码，就多了if (mCallback.isTranslucent())

```java
private void dragTo(int left, int top, int dx, int dy) {
        int clampedX = left;
        int clampedY = top;
        final int oldLeft = mCapturedView.getLeft();
        final int oldTop = mCapturedView.getTop();
        if (dx != 0) {
            clampedX = mCallback.clampViewPositionHorizontal(mCapturedView, left, dx);
            if (mCallback.isTranslucent()) {
                ViewCompat.offsetLeftAndRight(mCapturedView, clampedX - oldLeft);
            }
        }
        if (dy != 0) {
            clampedY = mCallback.clampViewPositionVertical(mCapturedView, top, dy);
            ViewCompat.offsetTopAndBottom(mCapturedView, clampedY - oldTop);
        }

        if (dx != 0 || dy != 0) {
            final int clampedDx = clampedX - oldLeft;
            final int clampedDy = clampedY - oldTop;
            if (mCallback.isTranslucent()) {
                mCallback.onViewPositionChanged(mCapturedView, clampedX, clampedY,
                        clampedDx, clampedDy);
            }
        }
    }
```