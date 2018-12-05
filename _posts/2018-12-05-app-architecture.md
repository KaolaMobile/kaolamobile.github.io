---
layout:     post
title:      "浅谈构架模式"
subtitle:   ""
date:       2018-12-05
author:     "石翔飞"
header-img: "img/bg15.jpg"
tags:
    Android Architecture
---

# 浅谈构架模式

<!-- TOC -->

- [浅谈构架模式](#浅谈构架模式)
    - [一、MVP 模式](#一mvp-模式)
        - [1.1 MVP 的优缺点](#11-mvp-的优缺点)
            - [层次比较清晰，逻辑和 UI 分离。](#层次比较清晰逻辑和-ui-分离)
            - [颗粒度过细，导致接口过多。](#颗粒度过细导致接口过多)
            - [UI 为驱动的模型](#ui-为驱动的模型)
            - [V 层与 P 层有耦合](#v-层与-p-层有耦合)
            - [容易产生内存泄露](#容易产生内存泄露)
    - [二、利用 RxLifecycle 解决内存泄露](#二利用-rxlifecycle-解决内存泄露)
        - [2.1 bindUntilEvent](#21-binduntilevent)
        - [2.2 bindToLifecycle](#22-bindtolifecycle)
    - [三、MVVM 模式](#三mvvm-模式)
        - [3.1 MVVM 的优缺点](#31-mvvm-的优缺点)
            - [数据驱动](#数据驱动)
            - [低耦合](#低耦合)
            - [VM 可复用](#vm-可复用)
            - [鸡肋的 Data Binding](#鸡肋的-data-binding)
    - [四、Android Architecture Components（AAC）](#四android-architecture-componentsaac)
        - [4.1 生命周期变化所带来的问题](#41-生命周期变化所带来的问题)
        - [4.2 Lifecycle](#42-lifecycle)
        - [4.3 生命周期组件可感知的例子](#43-生命周期组件可感知的例子)
        - [4.4 如何感知 Activity 生命周期变化](#44-如何感知-activity-生命周期变化)
        - [4.5 如何感知 Fragment 生命周期变化](#45-如何感知-fragment-生命周期变化)
        - [4.6 利用 LifecycleRegistry 统一分发 Event](#46-利用-lifecycleregistry-统一分发-event)
        - [4.7 LiveData](#47-livedata)
        - [4.8 ViewModel](#48-viewmodel)
    - [五、应该采用何种构架模式](#五应该采用何种构架模式)
    - [六、参考](#六参考)

<!-- /TOC -->

## 一、MVP 模式

几乎所有的构架思想都是为了解耦、功能模块化。MVP 也可以较好的做到这一点，但是相应的也会产生一些缺点。

### 1.1 MVP 的优缺点

#### 层次比较清晰，逻辑和 UI 分离。

#### 颗粒度过细，导致接口过多。 
> P 层一般代表一块业务，可能有 N 接口，V 层如果需要考虑成功和失败的情况，会有 2N 个接口；如果一个页面有多个业务，需要实现多个 P，导致接口的数量成倍增长。

#### UI 为驱动的模型
> 需要监听 V，调用 P，数据变化后需要回调 V 改变 UI。同时回调 V 的时候需要检测 Activity 的生命周期。
>
> 更希望数据能转被动为主动，希望数据能更有活性，由数据来驱动UI

#### V 层与 P 层有耦合
> P 与 V 相互持有。功能或 UI 的改动可能需要修改 P 和 V 的接口

#### 容易产生内存泄露
> 页面关闭之后，如果 P 层还持有 V 层的引用，就会导致页面没有真正销毁，从而引起内存泄露。
>
> 当然也有一些解决办法，比如利用形如 BaseActivity、BasePresenter 去做一些清除引用的操作，在指定的时机取消订阅。
也可以利用 RxJava2 + Lifecycle，正如 RxLifecycle 所做的。

## 二、利用 RxLifecycle 解决内存泄露

什么是 RxLifecycle?

> This library allows one to automatically complete sequences based on a second lifecycle stream.
>
> This capability is useful in Android, where incomplete subscriptions can cause memory leaks.

划重点：利用 **second lifecycle stream** 解决 **memory leaks**。

那么，RxLifecycle 又是怎么做到的呢？

### 2.1 bindUntilEvent

从最初 P 层的数据请求开始，以下是在 Lifecycle.Event.ON_DESTROY 的时候停止发送 requestObservable 数据：

```kotlin
requestObservable.doOnSubscribe { mView.showLoadingTranslate() }
        .bindUntilEvent(mView, Lifecycle.Event.ON_DESTROY)
        .subscribe(
                {
                    mView.endLoading()
                    //...
                },
                {
                    mView.endLoading()
                    //..
                }
        )
```

requestObservable 作为上游请求的 Observable，在此记为 **第一个 Observable**。

其中 **.bindUntilEvent**，作为扩展函数调用：
```kotlin
fun <T> Observable<T>.bindUntilEvent(view: BaseRxView, event: Lifecycle.Event): Observable<T> = this.compose(view.bindUntilEvent(event))
```

接着进入 BaseRxView，也就是形如 BaseActivity 或者 BaseFragment 的接口：

```kotlin
interface BaseRxView : BaseView, LifecycleProvider<Lifecycle.Event>, LifecycleObserver, LifecycleOwner {

    var lifecycleSubject: BehaviorSubject<Lifecycle.Event>

    /**
     * 绑定生命周期直到某个订阅的生命周期事件被发送
     */
    override fun <T> bindUntilEvent(event: Lifecycle.Event): LifecycleTransformer<T> {
        return RxLifecycle.bindUntilEvent(lifecycleSubject, event)
    }

    /**
     * 根据lifecycle托管组件的生命周期绑定控制
     */
    override fun <T> bindToLifecycle(): LifecycleTransformer<T> {
        return RxAndroidLifecycle.bindLifecycle(lifecycleSubject)
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_ANY)
    fun onEvent(owner: LifecycleOwner, event: Lifecycle.Event) {
        lifecycleSubject.onNext(event)
        if (event == Lifecycle.Event.ON_DESTROY) {
            owner.lifecycle.removeObserver(this)
        }
    }

    //...
}
```

回到项目中，在我们的 BaseActivity 中初始化 lifecycleSubject，并对 lifecycle 添加观察者：

```kotlin
override var lifecycleSubject: BehaviorSubject<Lifecycle.Event> = BehaviorSubject.create<Lifecycle.Event>().apply {
    lifecycle.addObserver(this@BaseCompatActivity)
}
```

利用 Lifecycle 的特性，在发生 Lifecycle.Event 的时候，会执行到 BaseRxView 中的 onEvent 方法，执行 lifecycleSubject.onNext(event) 发送此事件。

此处的 lifecycleSubject，记为 **第二个 Observable**。

**bindUntilEvent** 方法会对此 Observable 加一层过滤：

```kotin
private fun <R> takeUntilEvent(lifecycle: Observable<R>, event: R): Observable<R> {
    return lifecycle.filter { lifecycleEvent -> lifecycleEvent == event }
}
```

最终我们提到的两个 Observable 会执行到 RxLifecycle 最核心的方法：**takeUntil**:
```kotlin
override fun apply(upstream: Observable<T>): ObservableSource<T> = upstream.takeUntil(observable)
```

官方解释如下：

> Returns an Observable that emits the items emitted by the source Observable until a second ObservableSource
 emits an item.

如果第二个过滤后的 Observable 发射出了一条数据，比如我们最初业务调用时传入的 Lifecycle.Event.ON_DESTROY，那么 takeUntil 就会触发。第一个 Observable，也就是此处的 upstream 就会停止传递数据，执行 Unsubscribe，流自动结束，从而确保 V 的生命周期安全，内存泄露也不会发生。

### 2.2 bindToLifecycle

**bindToLifecycle** 能够在对应的生命周期解除绑定关系，比如我们在 onCreate 执行绑定，那么就会为 onDestroy 解绑。

实现原理也是利用了 **takeUntil**，不过相比 **bindUntilEvent** 中用的 **fitler** 操作符，在 **bindToLifecycle** 中则是用了 **combineLatest**：

```kotlin
private fun <R> takeUntilCorrespondingEvent(lifecycle: Observable<R>,
                                            correspondingEvents: Function<R, R>): Observable<Boolean> {
    return Observable.combineLatest(
            lifecycle.take(1).map(correspondingEvents), //对应绑定生命周期所对应的解绑定生命周期
            lifecycle.skip(1),   //跳过当前的绑定
            BiFunction<R, R, Boolean> { bindUntilEvent, lifecycleEvent ->
                lifecycleEvent == bindUntilEvent  //判断是否到了对应解绑定的生命周期
            })
            .onErrorReturn(LifecycleFunction.RESUME_FUNCTION) //防止绑定生命周期错误导致的订阅终止
            .filter(LifecycleFunction.SHOULD_COMPLETE) //根据是否是对应解绑定的生命周期来进行过滤
}
```

比如 lifecycle.take(1) 发射 Lifecycle.Event.ON_CREATE，经过 map 之后，转变为 Lifecycle.Event.ON_DESTROY；

lifecycle.skip(1) 会依次发射：Lifecycle.Event.ON_START、Lifecycle.Event.ON_RESUME、Lifecycle.Event.ON_PAUSE、Lifecycle.Event.ON_STOP、Lifecycle.Event.ON_DESTROY；

在执行到 Lifecycle.Event.ON_DESTROY 时，满足条件 lifecycleEvent == bindUntilEvent，从而触发 **takeUntil**


> 以上着重说到了 MVP 模式，以及如何利用 RxLifecycle 解决内存泄露的问题。
>
> **但是，如果 P 不持有 V 的引用，不就没有这一系列问题了吗？**

## 三、MVVM 模式

2015年的 Google I/O 上提出了 Data Binding 的库，意味着在 Android 开发中，也支持 MVVM 的开发模式。

> “MVVM 的目标和思想与 MVP 类似，利用数据绑定(Data Binding)、依赖属性(Dependency Property)、命令(Command)、路由事件(Routed Event)等新特性，打造了一个更加灵活高效的架构。”


### 3.1 MVVM 的优缺点

#### 数据驱动
> 借助 Data Binding，可以实现 UI 和数据的双向绑定。并且对 UI 的设置是 Null Safe 的。

#### 低耦合
> 数据和业务逻辑处于一个独立的 ViewModel 中，ViewModel 只需要关注数据和业务逻辑，不需要和 UI 或者控件打交道。ViewModel 不持有 V。

#### VM 可复用
> 一个 ViewModel 可以复用到多个 View 中。同样的一份数据，可以提供给不同的 UI 去做展示。对于版本迭代中频繁的 UI 改动，更新或新增一套 View 即可。如果想在 UI 上做 AB Test，那 MVVM 是不二选择。

#### 鸡肋的 Data Binding
正如 JakeWharton 所说：
> The data binding might be useful in the most trivial cases, but there was no innovation done so it'll be equally as painful as similar solutions on other platforms when complexity increases (e.g., XAML). This library scales to the advanced cases since it forces you to keep binding logic in code where it more correctly belongs.

> XML 中不应出现过于复杂逻辑，否则会使得 XML 很难维护。
>
> 修改接口或者切换分支时，Binding 类易出现编译错误，需要 Clean。因为生成了 Binding 类，所以对编译速度也有些许影响。

对于复杂的 UI，我并不推荐使用 Data Binding。

## 四、Android Architecture Components（AAC）

### 4.1 生命周期变化所带来的问题

不管是组件，还是业务代码，在需要考虑 Activity 生命周期的变化的时候，都有可能写出如下代码，不是很优雅：

```java
class MyActivity extends AppCompatActivity {
    private MyLocationListener myLocationListener;

    public void onCreate(...) {
        myLocationListener = new MyLocationListener(this, location -> {
            // update UI
        });
    }

    @Override
    public void onStart() {
        super.onStart();
        Util.checkUserStatus(result -> {
            // what if this callback is invoked AFTER activity is stopped?
            if (result) {
                myLocationListener.start();
            }
        });
    }

    @Override
    public void onStop() {
        super.onStop();
        myLocationListener.stop();
    }
}
```
```java
GoodsDetailNetManager.loadRecommendData(goodsId, new BaseManager.UIDataCallBack<RecommendGoods>() {
    @Override
    public void onSuccess(RecommendGoods recommendGoods) {
        if (!ActivityUtils.activityIsAlive(getContext())) {
            return;
        }
        //...
        updateView();
    }

    @Override
    public void onFail(int code, String msg) {
         if (!ActivityUtils.activityIsAlive(getContext())) {
             return;
         }
         //...
         updateView();
     }
});
```

具体如何构建可感知生命周期的组件呢？需要先从 Lifecycle 说起。

### 4.2 Lifecycle

Lifecycle 主要用 State 和 Event 这两个枚举类来表达 Activity 或者 Fragment 的生命周期和状态。 LifecycleRegistry 是它的主要实现类，用于分发生命周期的变化。

State 和 Event 之间的关系用下图表示:

![ss](https://haitao.nos.netease.com/0d6597b2-f239-4a31-b2db-b47dd25b9c70_1394_798.png)

### 4.3 生命周期组件可感知的例子

比如在考拉的曝光打点框架中，利用 LifecycleOwner 生命周期的变化，主动进行曝光逻辑，省去业务层需要考虑生命周期的额外处理：

```java
/**
 * 页面重新展现
 * 刷新时间
 */
 @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
 private fun onResumeAttach() {
    // ...
 }

 /**
  * 页面离开
  * 进行曝光
  */
  @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
  fun onPauseDetach() {
    // ...
  }

/**
 * 页面销毁
 * 清除数据
 */
 @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
 private fun clearData() {
    // ...
 }
```

### 4.4 如何感知 Activity 生命周期变化

> ON_CREATE, ON_START, ON_RESUME events in this class are dispatched after the LifecycleOwner's related method returns. ON_PAUSE, ON_STOP, ON_DESTROY events in this class are dispatched before the LifecycleOwner's related method is called. For instance, ON_START will be dispatched after onStart returns, ON_STOP will be dispatched before onStop is called. This gives you certain guarantees on which state the owner is in.

值得注意的是，对于正向过程（ON_CREATE、ON_START、ON_RESUME）和反向过程（ON_PAUSE、ON_STOP、ON_DESTROY），Event 和与之对应的生命周期方法的时序是不同的。

**android.app.Fragment** 和 **android.support.v4.app.Fragment**，在与之对应的 Activity 生命周期方法的执行顺序上，是有所不同的。

> 正向过程，先走 Acitivty，再走 Fragment
> 
> 反向过程，如果是 **android.support.v4.app.Fragment**，先走 Activity，再走 **android.support.v4.app.Fragment**。如果是 **android.app.Fragment**，先走 **android.app.Fragment**，再走Activity。
> 
> 因此，对 Activity 生命周期变化事件的分发，采用添加一个 **android.app.Fragment**（ReportFragment）。

### 4.5 如何感知 Fragment 生命周期变化

> 正向过程，先走 **Frament** ，再走 **FragmentLifecycleCallbacks**。

> 反向过程，先走 ChildFragment，再走 **Fragment**

正向过程，利用 **FragmentLifecycleCallbacks** 中 onFragmentCreated、onFragmentStarted、onFragmentResumed 在对应的 **Fragment** 生命周期方法之后，callback 才被调用。所以在 FragmentLifecycleCallbacks 中做 dispatch。

反向过程，利用 ChildFragment 的 onPause、onStop、onDestroy 在对应的 **Fragment** 的方法之前被调用。所以在 ChildFragment 中做 dispatch。

### 4.6 利用 LifecycleRegistry 统一分发 Event

不管是 Fragment，还是 Activity，最后都会通过 LifecycleRegistry 去分发 Event，在此就不详细分析了。

```java
@Override
public Lifecycle getLifecycle() {
    return mLifecycleRegistry;
}
```

### 4.7 LiveData 

LiveData 是一个生命周期可感知的数据存储类，正如官方文档所说：

> LiveData is an observable data holder class. Unlike a regular observable, LiveData is lifecycle-aware, meaning it respects the lifecycle of other app components, such as activities, fragments, or services. This awareness ensures LiveData only updates app component observers that are in an active lifecycle state.

不妨从 observe 入口看起：

```java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    owner.getLifecycle().addObserver(wrapper);
}
```
Lifecycle 的具体实现（mLifecycleRegistry）会将生命周期的变化传给 LiveData，在 **LifecycleBoundObserver** 中响应生命周期的变化：

```java
@Override
boolean shouldBeActive() {
    return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
}

@Override
public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
    if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
        removeObserver(mObserver);
        return;
    }
    activeStateChanged(shouldBeActive());
}
```

可以看到，在 **LifecycleBoundObserver** 中，只有 STARTED、RESUMED，才会是 active 状态。

在我们对 LiveData 执行 setValue 或者 postValue 之后，最终会执行到 **considerNotify**:

```java
private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    //
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    //noinspection unchecked
    observer.mObserver.onChanged((T) mData);
}
```

显然，只有 active 状态的 LiveData 才会将数据传递给业务中我们注册的 Observer。

以上只是对 LiveData 的简单分析，具体可以参考更多源码。

### 4.8 ViewModel

用于管理数据，ViewModel 持有 LiveData。

> The ViewModel class is designed to store and manage UI-related data in a lifecycle conscious way. The ViewModel class allows data to survive configuration changes such as screen rotations.

那么 ViewModel 是如何做到 **allows data to survive** 的呢？

ViewModel 通过为关联的 Activity 或者 Fragment 创建一个 HolderFragment，并设置 **setRetainInstance(true)**，实现了配置变化时仍能保存对应具体信息的效果。


## 五、应该采用何种构架模式

构架设计归根结底是根据需求进行分层涉及的规范。

不管采用何种构架，首先应保证功能模块清晰、易维护，这在客户端版本快速迭代的今天尤为重要。其次应该防止过度设计。

至于采用何种构架模式，个人觉得不需要硬性规定，看每个人的喜好了，只要你写的足够清晰、易维护。

## 六、参考

https://github.com/ReactiveX/RxJava/wiki

https://github.com/trello/RxLifecycle

https://developer.android.com/topic/libraries/architecture/

https://tech.meituan.com/android_mvvm.html
