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
    这里传入一个OnSubscribe对象作为参数，它会被存储在返回的Observerable对象中，相当于一个计划表，当Observerable被订阅时，OnSubscribe的`call()`方法会被调用，事件序列就会按照设定的顺序依次触发（对于上面的代码，观察者Subscriber会触发三次onNext()和一次onCompleted()）。

    `onCreate()`是RxJava最基本的创造事件队列的方法。基于这个方法，RxJava还提供了一些方法用来快捷创建事件队列。

    * just(T ...)：将传入的参数依次发送出去。
    ```
    Observable observable = Observable.just("Hello","Hi","Alopa");
    // 将会依次调用：
    // onNext("Hello");
    // onNext("Hi");
    // onNext("Alopa");
    // onCompleted();
    ```

    * from(T[])/from(Iterable<? extends T>)：将传入的数组或者容器拆分成具体对象后发送出去。
    ```
    String words = new String{"Hello","Hi","Alopa"}; 
    Observalbe observalbe = Observable.from(words);
    // 将会依次调用：
    // onNext("Hello");
    // onNext("Hi");
    // onNext("Alopa");
    // onCompleted();

    ```

3. Subscribe(订阅)
    
    创建了Observer和Observeable之后，再用`Subscribe`把两者联系起来，这样就能工作了。
    
    ```
    Observeable.subscribe(observer);
    //或者
    Observeable.subscribe(subscriber);
    ```
    除了`subscribe(Observer)`和`subscribe(Subscriber)`之外，RxJava还支持不完整定义的回调，RxJava会根据定义自动创建出Subscriber。
   
     ```
     Action1<String> onNextAction = new Action1<String>() { 
     // onNext()
         @Override
         public void call(String s) {
             Log.d(tag, s);
         }
     };

    Action1<Throwable> onErrorAction = new Action1<Throwable>() {
    // onError()
        @Override
        public void call(Throwable throwable) {
            // Error handling
        }
    };

    Action0 onCompletedAction = new Action0() {
    // onCompleted()
        @Override
        public void call() {
            Log.d(tag, "completed");
        }
    };

    // 自动创建 Subscriber ，并使用 onNextAction 来定义 onNext()
    observable.subscribe(onNextAction);

    // 自动创建 Subscriber ，并使用 onNextAction 和 onErrorAction 来定义 onNext() 和    onError()
    observable.subscribe(onNextAction, onErrorAction);

    // 自动创建 Subscriber ，并使用 onNextAction、 onErrorAction 和 onCompletedAction 来定义 onNext()、 onError() 和 onCompleted()
    observable.subscribe(onNextAction, onErrorAction, onCompletedAction);
    ```

    * Action0：RxJava的接口，只有一个方法`call()`，这个方法是无参无返回值的。
    * Action1：RxJava的接口，只有一个方法`call(T param)`，这个方法无返回值，但有一个参数。

好了，基本的方法介绍完了，我们来个简单的例子使用一下上面的方法。
![](http://ofowf99vj.bkt.clouddn.com/%E4%B8%BE%E4%B8%AA%E4%BE%8B%E5%AD%90.jpg)

由Id来取得图片并展示：
```
// 图片id
int resourceId = R.id.bg_img;
// 展示用的ImageView
ImageView img = new ImageView();
// 使用RxJava
Observable.create(new Onsubscribe<Drawable>() {

    @Override
    public void call(Subscriber<? super Drawable> subscriber) {
        Drawable drawable = getTheme().getDrawable(resourceId);
        subscriber.onNext(drawable);
        subscriber.onCompleted();
    }
}).subscribe(new Observer<Drawable>() {

    @Override
    public void onNext(Drawable drawable) {
        img.setImageDrawable(drawable);
    }

    @Override
    public void onCompleted() {
    
    }
    
    @Override
    public void onError(Trowable e) {
        Log.e(TAG,e.toString());
    }
});
```

就像上面的例子，创建Observeable，和Observer，然后用subscribe相连。

然而

![](http://ofowf99vj.bkt.clouddn.com/ranbingluan.jpg)

在RxJava的默认规则中，事件的发出和消费都在同一个线程。观察者的目的在于：**后台处理，前台回调**的异步机制，因此异步对于RxJava异常重要。而要实现异步，则需要用到RxJava的另一个概念：Scheduler。

###线程控制-Scheduler

在不指定线程的情况下，RxJava遵循的是线程不变的原则，即：在哪个线程调用`subscribe()`，就在哪个线程生产事件；在哪个线程生产事件，就在哪个线程消费事件。如果需要切换线程，就需要用到`Scheduler `（调度器）。

在RxJava中，`Scheduler`相当于线程控制器，RxJava通过它来指定每一段代码应该运行在什么样的线程。RxJava已经内置了几个Scheduler。

* `Schedulers.immediate()`：直接在当前线程运行，相当于不指定线程。
* `Schedulers.newThread()`：总是启动新线程，并在新线程中执行操作。
* `Schedulers.io()`：I/O操作所使用的Scheduler。行为模式和newThread()差不多，只是在io()的内部实现中是用一个无数量上限的线程池，可以重用空闲的线程。不要把计算工作放在io()中，避免创建不必要的线程。
* `Schedulers.computation()`：计算所使用的Scheduler。这个计算指的是CPU密集型计算，例如图形的计算。这个使用固定的线程池，大小为CPU的核数。
* 另外，Android还有一个专用的Scheduler，`AndroidSchedulers.mainThread()`，它指定的操作将在Android主线程里执行。

使用`subscribeOn()`和`observeOn`两个方法对线程进行控制。
* `subscribeOn()`：指定了subscribe（)所发生的线程，即`Observable.subscribe()`被激活时的线程，叫做事件产生的线程。
* `observeOn()`：指定Subscriber所运行的线程，叫做事件消费的线程。

```
Observable.just(1,2,3,4)// 发送消息
    .subscribeOn(Schedulers.io())// 指定subscribe()发生在io线程
    .observeOn(AndroidSchedulers.mainThread())// 指定Subscriber的回调在主线程
    .subscribe(new Action1<Integer>() {
        @Override
        public void call(Integer number) {
            Log.e(TAG,"number:" + number);
        }
});
```

###Scheduler的原理

####变换

RxJava提供了对事件序列进行变换的支持，这是它的核心功能之一。所谓的变换，**就是把事件序列中的对象或整个序列进行加工处理，转换成不同的事件或事件序列**。

```
String img = "images/app_logo.jpg";
Observable.just(img)
    .map(new Func1<String,Bitmap>() {
        @Override
        public Bitmap call(String filePath) { // 参数类型是String
            return getBitmapFromPath(filePath); // 返回类型是Bitmap
        }
    })
    .subscribe(new Action1<Bitmap> (){
        @Override
        public void call(Bitmap bitmap) { // 参数类型是Bitmap
            shiwBitmap(bitmap);
        }
});
```

Func1类和Action1非常相似，也是RxJava的接口，用于包装含有一个参数的方法，区别在于Func1有返回值。FuncX和ActionX一样，用于不同参数个数的方法，区别在于FuncX用于包装含有返回值的方法。

* `map()`：事件对象的直接变换，它是RxJava最常用的变换，就是把一种类型的参数变换成另一种类型。
* `flatMap()`：这是一个常用但是非常难理解的变换函数。

假设有个数组“学生”，我们要打印出每个学生选修的课程

比较容易想到的一种方法：
```
Student[] student = ...;
Subscriber<Student> subscriber = new Subscriber<Student>() {
    
    @Override
    public void onNext(Student student) {
        List<Course> courses = student.getCourse();
        for(Course course:course) {
            Log.e(TAG,"course:"+course);
        }
    }
};
Observable.from(student)
    .subscribe(subsriber);
```
这个方法很简单，但是如果我们不想在Subscriber里使用for循环来遍历，而是希望直接传入单个的Course对象，能实现吗？**必须能！！**但是`map()`显然是不行的，因为它是一对一的转换，而我们需要一对多的转换，这个时候就需要**`flatMap()`**了。
```
Student[] students = ...;
Subscriber<Course> subscriber = new Subscriber<Course>() {
    
    @Override
    public void onNext(Course course) {
        Log.e(TAG,"Course" + course);
    }
};
Observable.from(students)
    .flatMap(new Func1<Student,Observable<Course>> () {
        @Override
        public Observable call(Student student) {
            return Observable.from(student.getCourse());
        }
    })
    .subscribe(subscriber);
```
从上面代码可以看出，`flatMap()`和`map()`有一个相同点：它也是把传入的参数转化之后返回另一个对象。不同的是，flatMap()返回的是一个Observable对象，并且这个Observable对象并不是直接发送到Subscriber中。

`flatMap()`的原理是：

1. 使用传入的对象创建一个Observable对象
2. 并不发送这个对象，而是激活它，用它来发送事件
3. 每一个创建出来的Observable发送的对象，都被汇入同一个Observable，而这个Observable负责将这些事件统一交给Subscriber的回调方法。

####lift()

这些变换虽然功能各不相同，但是实质都是**针对事件序列的处理和发送**。而在RxJava内部，它们是基于同一个基础的变换方法：`lift(operator)`。看一下lift()的内部实现
```
public <R> Observable<R> lift(Operator<? extends R, ? super T> operator) {  
    return Observable.create(new OnSubscribe<R>() {
         @Override         
         public void call(Subscriber subscriber) {             
            Subscriber newSubscriber = operator.call(subscriber);  
            newSubscriber.onStart();             
            onSubscribe.call(newSubscriber);        
         }    
     });
}
```
当含有`lift()`时：
1. lift() 创建了一个 Observable 后，加上之前的原始 Observable，已经有两个 Observable 了； 
2. 而同样地，新 Observable 里的新 OnSubscribe 加上之前的原始 Observable 中的原始 OnSubscribe，也就有了两个 OnSubscribe； 
3. 当用户调用经过 lift() 后的 Observable 的 subscribe() 的时候，使用的是 lift() 所返回的新的 Observable ，于是它所触发的 onSubscribe.call(subscriber)，也是用的新 Observable 中的新 OnSubscribe，即在 lift() 中生成的那个 OnSubscribe； 
4. 而这个新 OnSubscribe 的 call() 方法中的 onSubscribe ，就是指的原始 Observable 中的原始 OnSubscribe ，在这个 call() 方法里，新 OnSubscribe 利用 operator.call(subscriber) 生成了一个新的 Subscriber（Operator 就是在这里，通过自己的 call() 方法将新 Subscriber 和原始 Subscriber 进行关联，并插入自己的『变换』代码以实现变换），然后利用这个新 Subscriber 向原始 Observable 进行订阅。 这样就实现了 lift() 过程，有点像一种代理机制，通过事件拦截和处理实现事件序列的变换。
