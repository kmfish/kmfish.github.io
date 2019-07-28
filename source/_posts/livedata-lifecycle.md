---
title: LifyCycle 的思考
date: 2017-08-29 00:46:56
tags:
---

LifyCycle 的思考

# 大纲
google 的LifeCycle、 livedata
glide里的 lifeCycleListener 也是一样的实现 
分析下原理，对比下两者。
自己的组件怎么使用这个东西
<!--more-->

# Android architecture components

在包 android.arch.lifecycle 包下新增了一些组件，几个核心的包括 LifeCycle, LiveData, ViewModel
我们先重点关注下 LifeCycle开头的这几个。
Interfaces:
LifecycleObserver
LifecycleOwner
LifecycleRegistryOwner

Classes:


## 处理生命周期
### Lifecycle
Lifecycle is a class that holds the information about the lifecycle state of a component (like an activity or a fragment) and allows other objects to observe this state.
![lifecycle](https://developer.android.com/images/topic/libraries/architecture/lifecycle-states.png)
Think of the states as nodes of a graph and events as the edges between these nodes.
A class can monitor the component's lifecycle status by adding annotations to its methods.
```java
public class MyObserver implements LifecycleObserver {
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    public void onResume() {
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    public void onPause() {
    }
}
aLifecycleOwner.getLifecycle().addObserver(new MyObserver());
```
####  LifeCycle 其实就是一个状态机
内部有 state、event，然后接收新的Action，切换到newState。

### LifecycleRegistry implement LifeCycle
该类维护了一个Observer的列表，维护了当前的state，提供了添加、移除Observer、状态变更的接口。 LifecycleActivity、LifecycleFragment就使用该类来实现LifecycleOwner，负责上述这些职责。如果你有一个自己的LifecycleOwner类，那么可以直接组合使用该类。

这个类的核心逻辑，就是响应handleLifecycleEvent（event）
```java
    public void handleLifecycleEvent(Lifecycle.Event event) {
        if (mLastEvent == event) {
            return;
        }
        mLastEvent = event;
        mState = getStateAfter(event);   // 根据新event切换state到新状态
        for (Map.Entry<LifecycleObserver, ObserverWithState> entry : mObserverSet) {  // 通知观察者同步state
            entry.getValue().sync();
        }
    }
  
// ObserverWithState class
    void sync() {
            if (mState == DESTROYED && mObserverCurrentState == INITIALIZED) {
                mObserverCurrentState = DESTROYED;
            }
            while (mObserverCurrentState != mState) {
                // 若观察者的状态和当前状态不一致，则逐个状态向上或向下切换并通知callback， 直至相同为止。
                Event event = mObserverCurrentState.isAtLeast(mState)
                        ? downEvent(mObserverCurrentState) : upEvent(mObserverCurrentState);
                mObserverCurrentState = getStateAfter(event);
                mCallback.onStateChanged(mLifecycleOwner, event);
            }
        }
```

### LifecycleOwner
这是一个只有一个方法的接口，提供一个LifeCycle实例的方法 getLifycycle()。任何应用类可以实现该接口。
一个通用的场景当这个LifeCycle当前未处在一个合适的状态时，避免执行某些回调。譬如，如果一个Fragment transaction回调在Activity state saved之后执行，就会发生crash，所以我们绝不希望此时调用这个回调。
基于此，LifeCycle支持查询其当前状态。
```java
class MyLocationListener implements LifecycleObserver {
    private boolean enabled = false;
    public MyLocationListener(Context context, Lifecycle lifecycle, Callback callback) {
      ...
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    void start() {
        if (enabled) {
          // connect
        }
    }

    public void enable() {
        enabled = true;
        if (lifecycle.getState().isAtLeast(STARTED)) {
            // connect if not connected
        }
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    void stop() {
        // disconnect if connected
    }
}
```
这个类就是生命周期感知的。它可以自己完成初始化和清理动作，而不需要Activity的管理。

能够和LifeCycle工作的类被称为是生命周期感知组件。那些和Android LifeCycle一起工作的类库被鼓励提供生命周期感知的组件，这样他们的客户可以轻松的集成这些类而不用手动管理生命周期。

LiveData就是一个生命周期感知组件的例子。

### LifecycleDispatcher 绑定Activity的生命周期
该类就是绑定并追踪application的Activity生命周期的关键类了，LifeCycle事件都是由它观察了Activity的事件后，再派发出来的。代码不多，实现原理也很简单，
1. 初始化时，向Application注入一个ActivityLifecycleCallbacks来监听其生命周期变化并派发。
2. 当Activity onCreate时，若activity instanceof FragmentActivity 为true，则向其supportfragmentManager注入一个FragmentLifecycleCallbacks接口，用来监听其内部的Fragment生命周期事件并派发，同时对其内部实现了LifecycleRegistryOwner的Fragment也添加一个DestructionReportFragment，用来接收onXXX事件以及派发事件。
然后，继续向其getFragmentManager注入一个ReportFragment，利用它来接收Activity的onXXX事件以及派发通知。

### LifecycleService

http://chaosleong.github.io/2017/05/27/How-Lifecycle-aware-Components-actually-works/

### LiveData
http://chaosleong.github.io/2017/06/14/How-LiveData-actually-works/

# Glide 中的LifeCycle
因为最近在看LifeCycle的代码，偶然搜索类时发现很多类库自己也实现了LifeCycle的机制。譬如Glide里就也有LifeCycle接口，它也实现了自己的LifeCycle管理。以下简单分析下其源码：

其实Glide中的LifyCycle实现和android arch包的实现原理是一样的，也是向当前Activity中添加一个Fragment(RequestManagerFragment)，依靠这个Fragment来接收系统组件的生命周期，然后再通知LifeCycle接口。RequestManager作为LifeCycleListener，最后会接收该LifeCycle的通知，从而控制自己管理的所有request的start、stop行为。图片加载的Target对象(如GlideDrawableImageViewTarget )也会接收通知，来控制动画的播放与停止。

# 使用LifeCycle来增强自己的类库
学习了LifeCycle的思路，我们可以考虑给自己的类库也添加上这个特性，这样可以让类库更加“智能”，可以自动的跟随LifeCycleOwner 组件（也不一定只是系统组件，也可以是外部实现并传递进来的）的生命周期来控制自己的行为。
如一个播放器类库，可以增加对生命周期的支持。当宿主Activity状态变更时，PlayerView可以自己暂停、恢复播放、销毁回收等，不再需要用户手动控制了。
