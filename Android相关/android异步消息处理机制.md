# android异步消息处理机制

异步消息处理线程启动后会进入一个无限的循环体之中，每循环一次，从其内部的消息队列中取出一个消息，然后回调相应的消息处理函数，执行完成一个消息后则继续循环。若消息队列为空，线程则会阻塞等待。

Handler 、 Looper 、Message 这三者都与Android异步消息处理线程相关的概念。

其实Looper负责的就是创建一个MessageQueue，然后进入一个无限循环体不断从该MessageQueue中读取消息，而消息的创建者就是一个或多个Handler 。

## Looper

Looper主要是prepare()和loop()这两个方法

先看Looper的构造方法：

![](/assets/Looper_Constructor.jpg)

初始化了mQueue，就是一个消息队列，传入的参数表示是否可以退出。然后初始化了mThread，使用当前线程。

然后我们看prepare()方法：

![](/assets/Looper_prepare.jpeg)

sThreadLocal是一个ThreadLocal对象，可以在一个线程中存储变量。可以看到，最后一行将一个Looper的实例放入了ThreadLocal，首先判断了sThreadLocal是否为null，若是null则抛出异常。这也就说明了Looper.prepare()方法不能被调用两次，同时也保证了一个线程中只有一个Looper实例。

最后我们来看loop()方法：

![](/assets/Looper_loop_1.jpeg)
![](/assets/Looper_loop_2.jpeg)

首先调用myLooper()方法拿到当前的Looper：

![](/assets/Looper_myLooper.jpeg)

接返回了sThreadLocal存储的Looper实例

如果当前的Looper为null就会抛出异常，也就是说looper()必须在prepare()之后运行。

之后拿到该looper实例中的mQueue（消息队列）。然后就该进入无限循环之中：

**从MessageQueue中取出消息，如果没有消息，就阻塞。有消息的话就调用`msg.target.dispatchMessage(msg)`方法，那么msg的target是什么呢？其实就是Handler！**

最后就是释放消息占据的资源。

Looper主要作用：

1. 与当前线程绑定，保证一个线程只会有一个Looper实例，同时一个Looper实例也只有一个MessageQueue。
2. loop()方法，不断从MessageQueue中去取消息，交给消息的target属性的dispatchMessage去处理。

好了，我们的异步消息处理线程已经有了消息队列（MessageQueue），也有了在无限循环体中取出消息的Looper，现在缺的就是发送消息的对象了，于是乎：Handler登场了。

## Handler

使用Handler之前，我们都是初始化一个实例，比如用于更新UI线程，我们会在声明的时候直接初始化，或者在onCreate中初始化Handler实例。

先看Handler的构造函数：

![](/assets/Handler_Construct.jpeg)

茫茫一大堆的构造函数，其实就是最后两个：

1. 传入callback和是否异步。代码里FIND_POTENTIAL_LEAKS是false的，所以基本不用看了。先获取当前线程保存的Looper实例。若mLooper为null就会抛出异常。**但是我们平常在Activity里用的时候并没有初始化Looper啊？那是因为ActivityThread已经准备好Looper了！**后面基本就是赋值了，从Looper对象里拿到了保存的MessageQueue，这样就保证了handler的实例与Looper实例中MessageQueue关联上了。

2. 传入Looper，callback和是否异步。

在Handler中我们经常使用的就是sendMessage()和dispatchMessage()了，先看sendMessage()：

![](/assets/Handler_sendMessage.jpeg)

调用了sendMessageDelayed，然后传入的delayMellis为0。

![](/assets/Handler_sendMessageDalayed.jpeg)

先判断delayMillis，做一个容错处理。（这个发送消息的时间肯定要当前或者靠后，不能靠前的），然后调用sendMessageAtTime：

![](/assets/Handler_sendMessageAtTime.jpeg)

先获取MessageQueue，若为null则抛出异常。然后调用enqueueMessage：

![](/assets/Handler_enqueueMessage.jpeg)

首先，msg.target赋值为this，这也证明了在Looper的loop方法里调用dispatchMessage方法的就是Handler。最终会调用queue的enqueueMessage的方法，也就是说handler发出的消息，最终会保存到消息队列中去。

当Looper循环的时候拿到这个message之后就会调用dispatchMessage方法：

![](/assets/Handler_dispatchMessage.jpeg)

先判断msg里的Callback是否为null，若不为null就调用handlerCallback方法；为null会再判断当前Handler的Callback是否为null，若不为null会走Callback的handlerMessage方法，为null就调用handleMessage方法。

最后的handlerMessage是不是很眼熟，对！我们创建Handler的时候都会覆写这个方法，然后根据msg.what进行消息处理。

## 总结

到此，这个流程已经解释完毕，让我们首先总结一下

1. 首先Looper.prepare()在本线程中保存一个Looper实例，然后该实例中保存一个MessageQueue对象；因为Looper.prepare()在一个线程中只能调用一次，所以MessageQueue在一个线程中只会存在一个。

2. Looper.loop()会让当前线程进入一个无限循环，不端从MessageQueue的实例中读取消息，然后回调msg.target.dispatchMessage(msg)方法。

3. Handler的构造方法，会首先得到当前线程中保存的Looper实例，进而与Looper实例中的MessageQueue相关联。

4. Handler的sendMessage方法，会给msg的target赋值为handler自身，然后加入MessageQueue中。

5. 在构造Handler实例时，我们会重写handleMessage方法，也就是msg.target.dispatchMessage(msg)最终调用的方法。

在Activity中，我们并没有显示的调用Looper.prepare()和Looper.loop()方法，为啥Handler可以成功创建呢，这是因为在Activity的启动代码中，已经在当前UI线程(ActivityThread)调用了Looper.prepare()和Looper.loop()方法。

## Handler的post

平常我们会使用Handler的post方法：

```
mHandler.post(new Runnable() {  
    @Override  
    public void run() {  
        Log.e("TAG", Thread.currentThread().getName());  
        mTxt.setText("XXX");  
    }  
}); 
```

我们来看一下Handler里的post方法：

![](/assets/Handler_post.jpeg)

其实调用的就是sendMessageDelayed，但是msg是通过getPostMessage(r)方法得到的：

![](/assets/Handler_getPostMessage.jpeg)

在getPostMessage中，得到了一个Message对象，然后将我们创建的Runable对象作为callback属性，赋值给了此message。

**注：产生一个Message对象，可以new  ，也可以使用Message.obtain()方法；两者都可以，但是更建议使用obtain方法，因为Message内部维护了一个Message池用于Message的复用，避免使用new重新分配内存。**

当Looper处理这个消息时，还是使用dispatchMessage方法，但是这次msg.callback不为null，就调用`handleCallback(msg)`方法了：

```
 private static void handleCallback(Message message) {
    message.callback.run();
}
```
其实最好调用的就是Runnable的run方法。也就是说这个Runnable并没有创建什么线程，而是发送了一条消息，最好的消息处理是Runnable的run方法。

那么postDelayed方法也好理解了，只不过我们传入了delayMillis，消息处理的时间会有个延迟。
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0OTU4MzgyMDVdfQ==
-->