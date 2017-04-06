---
title: android 子线程中更新界面？被ProgressBar给迷惑了
date: 2015-09-22 14:51:37
tags:
---

在看apidemos的例子RetainedFragement时，看到在Thread中执行了 这么一句
```java
mProgressBar.setProgress(progress);
```
且执行正常，progressbar确实一直在更新。
顿觉疑惑，View在更新时，会检查当前线程是否是创建view所在的线程（即UI线程），若不一致，则会抛出异常的。
<!--more-->

in ViewRootImpl.java 中：
```java
    void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
```
后来查看了setProgress（）的源码后，才恍然大悟，这个方法内已经处理了子线程里调用的情况了。
  
```java
@android.view.RemotableViewMethod
    public synchronized void setProgress(int progress) {
        setProgress(progress, false);
    }

    @android.view.RemotableViewMethod
    synchronized void setProgress(int progress, boolean fromUser) {
        if (mIndeterminate) {
            return;
        }

        if (progress < 0) {
            progress = 0;
        }

        if (progress > mMax) {
            progress = mMax;
        }

        if (progress != mProgress) {
            mProgress = progress;
            refreshProgress(R.id.progress, mProgress, fromUser);
        }
    }

    private synchronized void refreshProgress(int id, int progress, boolean fromUser) {
        if (mUiThreadId == Thread.currentThread().getId()) {
            doRefreshProgress(id, progress, fromUser, true);
        } else {
            if (mRefreshProgressRunnable == null) {
                mRefreshProgressRunnable = new RefreshProgressRunnable();
            }

            final RefreshData rd = RefreshData.obtain(id, progress, fromUser);
            mRefreshData.add(rd);
            if (mAttached && !mRefreshIsPosted) {
                post(mRefreshProgressRunnable);
                mRefreshIsPosted = true;
            }
        }
    }
```
关键就是refreshProgress（）了，这里处理了若当前线程不是ui thread，则将更新消息post到ui thread中去执行了。而且可以发现这几个方法都是加了同步控制的，是线程安全的，保证了多线程调用也是正常的了。

真的是“源码之下无秘密“啊。

p.s：
之前一次面试的时候，对方有问到能否在子线程里更新界面？
我回到不能，因为会抛出异常。对方接着问，为什么？
我就答不上来了，现在的理解是，因为android下的界面更新不是线程安全的，所以要保证在单一线程中同步执行。其他线程要更新UI的话，需要通过handler，Looper消息机制把更新事情pass到UI thread的消息队列中，由UI thread来完成界面的更新绘制。
面试官又继续问道，能否在子线程里去更新progressbar的进度呢？
我当时回答的是不能，必须通过handler去发消息。但现在看来progressbar的代码后，这个问题的答案其实应该是可以的，因为progressbar自己内部就处理了子线程的更新问题。

学东西，要能做到知其然，知其所以然。我对知识的掌握层次还太浅了，仅仅停留在会使用工具的层面。平时要多思考，自己给自己提出一些问题（what，why），才能加深对知识的理解。





















