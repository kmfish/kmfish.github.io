---
title: 修复一例 BlockingQueue.poll 导致的线程While循环无限执行占用cpu的bug
date: 2016-05-26 16:19:27
tags: 性能优化
---

# 起因
基于部分用户反馈使用我们的app时，玩游戏过程中会有卡顿现象出现，从而进行cpu使用率排查。
<!--more-->

# 发现问题
今天先操作进入一个房间后，使用android的traceview跟踪了一段2s左右的cpu使用数据，然后通过traceview对其进行分析。

{% asset_img post1.png  图一 %}

如图一所示，左边列出了该时间段内的线程，右边则图形化的显示了它们的cpu使用情况。很直观的能够发现，"GroupMsgTransport"这个线程几乎一直在占用cpu，比main线程还多的多。所以首先引起怀疑。

{% asset_img post2.png 图二 %}

再进一步查看详细的cpu使用情况，发现占用cpu time最多的几项都是在对一个BlockingQueue的操作，选择这些行之后，也定位到了 "GroupMsgTransport"线程。说明这条线程一直在对Queue执行poll操作。通过在代码中搜索关键字，查找到了这个罪魁祸首。

# 定位原因
最终定位到这段问题代码：
```java
@Override
     public void run() {
         android.os.Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

         while (run) {
             final ImGroupMsgInfo info;
             info = mMQ.poll();    // 罪魁祸首的一行， poll()方法在队列为空时会return null，从而就导致该循环无限执行下去，空耗了cpu
             if (info != null) {
                      //...
             } else {
                     // do nothing
             }
         }
     }
 
```

# 修复
知道了根本原因后，修复的方法也很简单，将poll换为take() 即可，take方法在队列为空时会block住当前thread，从而不再占用cpu。
```java
@Override
     public void run() {
         android.os.Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

         while (run) {
            ImGroupMsgInfo msgInfo = null;
            try {
                msgInfo = mMQ.take(); // 修改此处为take()
            } catch (InterruptedException e) {
                MLog.error(TAG, e);
            }
 
             final ImGroupMsgInfo info = msgInfo; 
             if (info != null) {
                      //...
             } else {
                     // do nothing
             }
         }
     }
```

# 结果对比

修改之前，cpu占用高达近50%：

{% asset_img post3.png 图三 %}
 
修改之后，cpu占用不到10%：
{% asset_img post4.png 图四 %}


# 后记
在修复了这一处问题后，又继续在项目里搜索了下使用poll()的地方，结果就又发现了一处一模一样的bug。。。
以后可以多留意while(true)循环，要注意循环是否有正确的退出时机，或block的时机，避免出现类似问题。






