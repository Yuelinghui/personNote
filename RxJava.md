# RxJava

##响应式编程

响应式编程是一种面向数据流和变化传播的编程范式。响应式编程最初是为了简化交互式用户界面的创建和实时系统动画的绘制而提出来的一种方法，但它本质上是一种通用的编程范式。

RxJava就是响应式编程的框架，采用观察者模式设计。
[github地址](https://github.com/ReactiveX/RxJava)

还有一个RxAndroid，用于Android的开发，可以用于Android的开发，添加了Android的接口。[github地址](https://github.com/ReactiveX/RxAndroid)

RxJava的本质就是一个词：**异步**！

同样都是异步，为啥不用AsyncTask/Handler/XXX？**简洁**！异步操作很关键的一点是程序的简洁性，因为在调度过程比较复杂的情况下，异步代码经常会既难写也难被读懂。`AsyncTask`和`Handler`都是为了使异步操作更加简洁。RxJava的优势也是**简洁**，但是它的不同之处在于：**随着逻辑越来越复杂，它依然能够保持简洁**。

![](http://ofowf99vj.bkt.clouddn.com/%E4%B8%BE%E4%B8%AA%E4%BE%8B%E5%AD%90.jpg)

假设现在有一个需求，有一个ImageView，我们从磁盘里拿出图片数组，然后让ImageView挨个展示。因为从磁盘读取图片是耗时工作，所以我们把它放到后台线程执行。而图片显示必须在UI线程执行。我们平常使用Thread和Handler配合完成
```
new Thread() {
 @Override
 public void run() { 
    super.run(); 
    for (File folder : folders) {
     File[] files = folder.listFiles(); 
     for (File file : files) { 
     if (file.getName().endsWith(".png")) { 
     final Bitmap bitmap = getBitmapFromFile(file);
     getActivity().runOnUiThread(new Runnable() { 
         @Override public void run() { 
           imageCollectorView.addImage(bitmap); 
          } 
       }); 
     } 
    } 
   } 
  } 
}.start();

```
如果我们使用RxJava的话，实现方式是这样的
```
Observable.from(folders)
 .flatMap(new Func1<File, Observable<File>>() {
     @Override
     public Observable<File> call(File file) {
         return Observable.from(file.listFiles());
     }
 })
 .filter(new Func1<File, Boolean>() {
     @Override
     public Boolean call(File file) {
         return file.getName().endsWith(".png");
     }
 })
 .map(new Func1<File, Bitmap>() {
     @Override
     public Bitmap call(File file) {
         return getBitmapFromFile(file);
     }
 })
 .subscribeOn(Schedulers.io())
 .observeOn(AndroidSchedulers.mainThread())
 .subscribe(new Action1<Bitmap>() {
     @Override
     public void call(Bitmap bitmap) {
         imageCollectorView.addImage(bitmap);
     }
 });

```
看完RxJava的代码，我们可能会觉得，代码量并没有减少，怎么就简洁了呢？其实，这里的**简洁**是指**逻辑的简洁**。RxJava的实现是一条从上到下的**链式调用**，没有任何嵌套。所以，RxJava的简洁就在于能把复杂的逻辑串成一条链。

###RxJava的观察者模式

RxJava有四个基本概念：`Observable`（被观察者），`Observer`（观察者），`subscribe`（订阅），事件。Observeable和Observer通过subscribe方式实现订阅关系，从而Observerable在需要的时候通过事件来通知Observer。

RxJava的事件回调除了普通事件`onNext`之外，还定义了两个特殊的事件：`onCompleted`和`onError`。

* onCompleted：事件队列完结。RxJava不仅把事件单独处理，还会把他们看做一个队列。Rxjava规定，当不会有新的onNext()发出时，需要触发onCompleted()作为标志。
* onError：事件队列异常。在事件处理过程中发生异常，会触发onError()，同时事件队列自动终止，不允许再有事件发出。
* 在一个正常运行的事件队列中，onCompleted和onError有且只有一个，并且是事件队列中的最后一个。这两者是互斥的，即在队列中调用了一个，另一个就不应该再调用。

###RxJava的基本使用
1. 创建Observer

    Observer即观察者，它决定事件触发的时候会有怎样的动作。
    
    ```
    Observer<String> observer = new Observer<String>() {
         @Override 
         public void onNext(String s) {
             Log.d(tag, "Item: " + s);
         }
         @Override
         public void onCompleted() {
             Log.d(tag, "Completed!");
         }
         @Override
         public void onError(Throwable e) {
             Log.d(tag, "Error!");
         }
    };
    ```
    除了Observer接口之外，RxJava还提供了一个实现了Observer的抽象类：`Subscriber`，   它对Observer接口进行了一些扩展，但是基本使用方法还是一样的。

    实质上，在RxJava的subscribe过程中，Observer也总是会先转换成Subscriber然后再使   用。两者之间的区别在于：
    * onStart()：这是Subscriber增加的方法。它会在subscribe刚开始，而事件还未被发送出去之前被调用，用来做一些准备工作。注意：**onStart()总是在subscribe发生的线程被调用，而不能指定线程**。
    * unsubscribe()：这是Subscriber实现的另一个接口Subscription的方法，用来取消订阅。在这个方法调用后，Subscriber将不在接收事件。因为在subscribe()之后，Observable会持有Subscriber的引用，这个引用如果不能及时被释放，将有内存泄露的风险。所以最好保持一个原则：要在不再使用的时候尽快在合适的地方（例如 onPause() onStop() 等方法中）调用unsubscribe()来解除引用关系，以避免内存泄露的发生。

2. 创建Observable
    Observable即被观察者，它决定了什么时候触发事件以及触发什么样的事件。RxJava使用`onCreate()`方法来创建Observable，并为它定义事件触发规则。
```
Observable observable = Observable.create(new Observable.OnSubscribe<String>() {
     @Override
     public void call(Subscriber<? super String> subscriber) {
         subscriber.onNext("Hello");
         subscriber.onNext("Hi");
         subscriber.onNext("Aloha");
         subscriber.onCompleted();
    }
});

```

