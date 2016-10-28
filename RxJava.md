# RxJava

##响应式编程

响应式编程是一种面向数据流和变化传播的编程范式。响应式编程最初是为了简化交互式用户界面的创建和实时系统动画的绘制而提出来的一种方法，但它本质上是一种通用的编程范式。

RxJava就是响应式编程的框架，采用观察者模式设计。
[github地址](https://github.com/ReactiveX/RxJava)

还有一个RxAndroid，用于Android的开发，可以用于Android的开发，添加了Android的接口。[github地址](https://github.com/ReactiveX/RxAndroid)

RxJava的本质就是一个词：**异步**！

同样都是异步，为啥不用AsyncTask/Handler/XXX？**简洁**！异步操作很关键的一点是程序的简洁性，因为在调度过程比较复杂的情况下，异步代码经常会既难写也难被读懂。`AsyncTask`和`Handler`都是为了使异步操作更加简洁。RxJava的优势也是**简洁**，但是它的不同之处在于：**随着逻辑越来越复杂，它依然能够保持简洁**。

![](http://ofowf99vj.bkt.clouddn.com/%E4%B8%BE%E4%B8%AA%E4%BE%8B%E5%AD%90.jpg)

假设现在有一个需求，有一个ImageView，我们从磁盘里拿出图片，然后让ImageView展示。因为从磁盘读取图片是耗时工作，所以我们把它放到后台线程执行。