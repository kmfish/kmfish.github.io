---
title: 通过flatmap组合多个SqlBrite的QueryObservable 使用的问题
date: 2016-12-21 19:16:18
tags: SqlBrite RxJava
---

# 阅读背景
[SqlBrite](https://github.com/square/sqlbrite)  是Square公司提供的一个数据库轻量级的封装框架。提供了RxJava的Observable风格的DB操作接口，其中一个特性是 其query操作得到的QueryObservable会一直保持对该次查询的表后续变更的事件的订阅，后续针对同一张（或多张）表的变更，均会再次发射数据给它的Subcriber，从而可以方便的实现界面的更新。

<!--more-->

自定义的词汇含义：
 - 订阅链、事件流： 这是指任意个Observable组合后，最终被某个Subscriber订阅时，所确定的一条订阅关系链。
如：　
```java
 ObservableA.flatMap(ObservableB)
                        .flatMap(ObservableC)
                       .subscribe(new Subscriber<T>() {
...
}
```

# 问题背景
我们最近的一个项目中使用了SqlBrite  + SqlDelight框架 来实现我们的数据库层。同时也使用了RxJava。所以在项目代码中，有许多针对数据库操作的接口都是以Observable方式返回的，就自然的出现了一些以各种方式组合多个QueryObservable，再提供给外层的Subscriber订阅的情况。如下例子：

```java
queryA.flatMap(new Func1<String, Observable<String>>() {
	@Override
	public Observable<String> call(String s) {
		return getQueryB(s);
	}
}).flatMap(new Func1<String, Observable<String>>() {
	@Override
	public Observable<String> call(String s) {
		return getQueryC(s);
	}
}).subcribe(new Action1(String str) {
	  // update UI
});

private QueryObservable getQueryB(String params) {
	return britedatabase.createQuery(....);
}

private QueryObservable getQueryC(String params) {
	return britedatabase.createQuery(....);
}
```
上述代码段中的queryA、queryB、queryC均是使用SqlBrite的createQuery接口返回的QueryObservable，所以它们均具有一直订阅着自己关注表的变更的特性（假设它们分别查询了A、B、C三张表）。咋看之下，这个流程似乎也没什么问题，我们的代码一开始也就是这样写的了。但后来遇到些数据错乱的情况时，才定位到这个问题。
设想这样的流程：
1. 当完成对这个事件流的订阅后，A表发生了1次变化，则queryA会发射1个数据到流里，那么接下来的getQueryB、getQueryC 方法均会执行1次，又由于createQuery每次都会创建QueryOBservable的实例，所以getQueryB执行1次，就创建了1个QueryObservableB的实例了，getQueryC同理，然后subcriber处会接收到1次数据。
2. 经过1之后，此时B表发生 了1次变化，那么getQueryC会执行一次，subscriber会收到1次事件，真的是这样吗？结论肯定是NO了。实际测试发现，此时subcriber会收到2次数据。

# 追根溯源
有点摸不着头脑了吧，我们再回顾之前的流程，会发现getQueryB 方法在整个流程中执行了2遍，说明创建了两个QueryObservableB的实例。当B表变化时，看起来像是这两个queryB的实例接收到了SqlBrite发射的数据了。
那究竟这两个queryB被谁订阅了呢？这就需要去看下RxJava的源码了，queryA和 getQueryB 是使用了flatMap操作符来连接的，所以我们看一下flatMap的实现：
```java
public final <R> Observable<R> flatMap(Func1<? super T, ? extends Observable<? extends R>> func) {
	if (getClass() == ScalarSynchronousObservable.class) {
		return ((ScalarSynchronousObservable<T>)this).scalarFlatMap(func);
	}
       // 重点就在这一行了，我们可以发现flatMap其实就是先map，然后再merge
	return merge(map(func));   
}

public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
	return create(new OnSubscribeMap<T, R>(this, func));  //  map的实现就是OnSubscribeMap，其实就是包装了一个内部订阅者来订阅上游的Observable，当收到上游的数据时，执行func来转换数据类型，再发射新数据给下游订阅者。
}

public final class OperatorMerge<T> implements Operator<T, Observable<? extends T>> {
  static final class MergeSubscriber<T> extends Subscriber<Observable<? extends T>> {
         .. .
        @Override
        public void onNext(Observable<? extends T> t) {
            if (t == null) {
                return;
            }
            if (t == Observable.empty()) {
                emitEmpty();
            } else
            if (t instanceof ScalarSynchronousObservable) {
                tryEmit(((ScalarSynchronousObservable<? extends T>)t).get());
            } else {
                InnerSubscriber<T> inner = new InnerSubscriber<T>(this, uniqueId++);     // 可以看到这里有个InnerSubscriber
                addInner(inner);
                t.unsafeSubscribe(inner);
                emit();
            }
        }
        
}
```
merge 操作符由于其代码比较多且有些复杂，我也只是简单分析了下，其内部使用了一个MergeSubscriber来代理上游的数据，然后让下游订阅它。
MergeSubscriber内部使用了InnerSubscriber的集合来订阅从上游接收到的每个Observable，然后再把接收的具体的data依次发射到下游，可以参考
[merge的弹珠图](https://raw.githubusercontent.com/wiki/ReactiveX/RxJava/images/rx-operators/merge.oo.png) 
[flatMap的弹珠图](https://raw.githubusercontent.com/wiki/ReactiveX/RxJava/images/rx-operators/flatMap.png)

经过对flatMap的分析，我们可以知道queryA发射数据，然后map操作符内部订阅了queryA，经过func转换后发射了多个QueryObservableB给merge操作符（因为这个func1是从String --> Observable<String>），merge操作符会在内部分别订阅收到的多个QueryOBservableB。分析到这里，就明白了我们的场景下，两个QueryOBservableB的实例都是被merge操作符给订阅了，所以当B表变化，SqlBrite会发射数据给到这两个queryB，从而最终传递到subscriber处，就得到了2次数据。

# 解决方案探讨
针对这个问题，我们尝试了几种解决思路：
- 避免这样的多个QueryObservable通过flatMap连接的情况，直接通过sql语句来做联合查询。
这样是回避了问题场景，但也确实有一些场景下还是会需要组合多个QueryObservable的，如先queryA，然后再到Server上查询一个结果b，最后再拿b去作为queryC的查询参数，这种情况下，queryA、queryC还是需要连接在一个事件流中了。
- 当多个QueryObservable连接时，从第二个开始的query，均不再监听表的变化，如添加一个 .first()。但这样的话，若B、C表将来变化时，最终的subcriber是不会更新的。又考虑，可以在queryA里去监听整个事件流中涉及到的所有DB table，无论任何一张表变化，整个事件流均重新query一遍。但再一细想，这个做法也是不行的，queryA根本不可能掌握到整个事件流里到底需要监听哪些表，因为queryB、queryC并没有暴露这个信息，除非去看queryB、queryC的实现，但那样又都全部耦合了。而且queryA、queryB、queryC这样的方式，还有一个好处就是一个QueryObservable可以被复用，根据需要，既可以独立使用，也可以结合其他的query组合使用，而2方案的话，也会破坏这个特性了。

# 我们采用的方案
实现一个QueryObservableManager对象，来统一管理每次事件流中的QueryObservable的订阅，其内部维护一个当前事件流中已有的query订阅关系集合。对原始的createQuery生成的QueryObservable进行一层装饰，由调用者传递一个context参数，用来唯一标识出一个事件流中的一个QueryObservable。当DecorationObservable被订阅时，根据context查询，把之前存在的订阅关系进行退订。DecorationObservable内部再订阅raw query Observable。这让最终流的订阅者增加了两个因素的变化，一个是QueryObservableManager，一个是context参数。
- 如何标识一个QueryObservable的订阅
通过分析，如果针对单个订阅链使用一个QueryObservableManager，那么在一个事件链中的不同QueryObservable可以利用return 该Observable的Method栈信息(StackTraceElement)来做context，来唯一标识。如上文例子的 getQueryB 这个方法。因为之前提到，getQueryB是可以复用的，在不同的订阅链中都可能出现，所以在不同订阅链的情况下，就不能仅仅以方法调用栈来标识了（此时两个实例的method栈信息是一样的，但其实是需要同时保持两个订阅的）。但对于一个订阅链来说，出现两个相同的栈时，意味着有多个相同的query被订阅了，此时就需要把之前的query给退订，保留最后一次订阅。一句话，**一个订阅链中，同一个getQuery 方法的Observable订阅只能保持一个。** 所以，这个context就不用外部传了，直接在QueryObservableManager 内部来确定即可。

- 如何在一个订阅链中，使用一个QueryObservableManager ？
通过将所有的类似 getQueryB的方法，均增加一个参数QueryObservableManager，在最终Observable被订阅时，由订阅者构造一个QueryObservableManager 实例，从而保证了在一个订阅链中，使用的都是同一个QueryObservableManager 对象。

# Demo测试项目
[QueryObservableManage实现](https://github.com/kmfish/TestSqlBriteDemo/blob/master/app/src/main/java/sqlbrite/demos/yy/com/sqlbrite/db/BriteQueryObservableFactory.java)

[Demo 项目](https://github.com/kmfish/TestSqlBriteDemo)

# 相关链接
[SqlBrite项目主页上也有人遇到这个问题了](https://github.com/square/sqlbrite/issues/102)
