---
title: Why use fragment？
date: 2014-09-21 21:49:13
tags:
---

针对这个问题，在网上搜索了下，有以下一些看法：

> 1、fragments are more reusable than custom views.


作者认为，fragment能够封装单靠view无法处理的事。自定义控件无法处理跨activity交互等事件，还是需要靠activity来完成，这会增加custom views 和activity的耦合。不利于复用。而fragment可以封装更多的东西，包括AsyncTask，file and database access等。

> 2、the main reason to use Fragments are for the backstack and lifecycle
> features. Otherwise, custom views are more light weight, simpler to
> implement, and have the advantage that they can be nested. 
[see here][1]


主要原因是fragment提供了backstack支持界面状态的回退，以及生命周期特性。（在SDK 17 以后fragment也支持嵌套了）
<!--more-->


- 过去的模式：
如果实现官方文档中的那个例子，手机版上是两个activity（列表和详情），pad版上则在同一activity下了。在fragment推出之前，我们之前的项目里也有实现过类似的需求。两个界面分别为 listView 和 detailView，则activityA里会同时持有两个view，当判断为手机版，则detailview会hide；当为pad版时，才会显示detailView。然后activityA里就会同时包含对两个的view的控制代码，当然也可以提出一个viewController来包含控制代码，activityA来引用viewController。这里的这个ViewController（或viewManager）就类似fragment。按照MVC的思想，activity其实就扮演了Controller的角色（或者再自己封装一个viewManager，但与系统的交互还是由activity来负责管理的。）

- fragment的优点：
fragment同样有自己的生命周期，不用刻意担心内存泄漏的问题，而且fragment还拥有切换保留（retain）机制，可以在状态切换期间保持状态不变（如后台任务等）。另外，由于框架已经实现了对fragment的生命周期管理，所以开发时也不必在activity的生命周期回调事件代码中加入对各个组件的管理代码，代码整体更加简洁清晰。使用fragment，可以帮助开发者省下许多开发和维护的工作量。

- 自己的理解：
通过查看fragment提供的接口，可以发现非常类似activity的接口，可以认为fragment is a “small activity“，所以相对于使用自定义view+ 控制器的方式，使用fragment可以封装更多的和系统的交互，提供更高级的封装。自定义view仅是在视图层面的封装（能够控制view的show，hide，布局的变化等），但fragment可以提供更高层面的控制（与activity交互，状态栈回退，独立的生命周期管理）。旧的方式依然需要依赖和activity的配合才能实现这些需求（如和其他activity的交互 startActiviyForResult，就可以直接在fragment里去调用了）。从整体上看，相当于将activity和可复用组件进一步解耦了，使得组件的复用性和封装性更好。activity不用再去向内部组件通知各项事件。而组件自身封装得更加完备，内部就处理了生命周期事件以及其他与系统的交互事件。在不同activity之间的复用变得更加容易灵活。从代码层面上看，activity里的代码就会少了许多，而各fragment的代码都由自己维护了。代码结构更加清晰简洁，大大降低了维护代价，也提高了项目代码的可读性。


  [1]: http://stackoverflow.com/questions/8617696/what-is-the-benefit-of-using-fragments-in-android-rather-than-views?lq=1









