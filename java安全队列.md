#Java中线程安全队列

在Java多线程应用中，队列的使用率很高，多数生产消费模型的首选数据结构就是队列。Java提供的线程安全的Queue可以分为阻塞队列和非阻塞队列，其中阻塞队列的典型例子是BlockingQueue，非阻塞队列的典型例子是ConcurrentLinkedQueue，在实际应用中要根据实际需要选用阻塞队列或者非阻塞队列。

##BlockingQueue

可以提供阻塞功能的队列

* add(e)，remove()，element()方法不会阻塞线程。当不满足约束条件时，会抛出IllegalStateException 异常。例如：当队列被元素填满后，再调用add(e)，则会抛出异常。
* offer(e)，poll()，peek()方法即不会阻塞线程，也不会抛出异常。例如：当队列被元素填满后，再调用offer(e)，则不会插入元素，函数返回false。
* **要想要实现阻塞功能，需要调用put(e)，take()方法**。当不满足约束条件时，会阻塞线程。

以ArrayBlickingQueue为例：

![](/assets/BlockingQueue_add.jpeg)

![](/assets/BlockingQueue_offer.jpeg)

offer和add方法都是向队列中添加元素，区别：**在一个已满的队列中添加数据，add方法会抛出异常，而offer方法不会抛出异常，只是返回添加失败（false）**

![](/assets/BlockingQueue_remove.jpeg)
![](/assets/BlockingQueue_poll.jpeg)

remove和poll方法都是从队列中删除第一个元素，两个方法区别：**在一个空队列中删除元素，remove方法会抛出异常，poll方法不会抛出异常，只是返回null**

![](/assets/BlockingQueue_element.jpeg)
![](/assets/BlockingQueue_peek.jpeg)

element和peek都是用于查询队列中第一个元素，区别：**在一个空队列中查询，element方法会抛出异常，peek方法不会抛出异常，只是返回null**

![](/assets/BlockingQueue_put.jpeg)

![](/assets/BlockingQueue_take.jpeg)

put和take方法都调用了await方法，**当前线程在接到信号或被中断之前一直处于等待状态**

![](/assets/BlockingQueue_enqueue.jpeg)

![](/assets/BlockingQueue_dequeue.jpeg)

这两个方法最后都调用了signal方法，**唤醒一个等待线程**

BlockingQueue接口的具体实现类有：ArrayBlockingQueue，LinkedBlockingQueue和PriorityBlockingQueue。

##ConcurrentLinkedQueue

它是一个无锁的并发线程安全的队列。

对比锁机制的实现，使用无锁机制的难点在于要充分考虑线程间的协调。简单的说就是多个线程对内部数据结构进行访问时，如果其中一个线程执行的中途因为一些原因出现故障，其他的线程能够检测并帮助完成剩下的操作。这就需要把对数据结构的操作过程精细的划分成多个状态或阶段，考虑每个阶段或状态多线程访问会出现的情况。

ConcurrentLinkedQueue有两个volatile的线程共享变量：head，tail。要保证这个队列的线程安全就是保证对这两个Node的引用的访问（更新，查看）的原子性和可见性，由于volatile本身能够保证可见性，所以就是对其修改的原子性要被保证。

在使用ConcurrentLinkedQueue时要注意，如果直接使用它提供的函数，比如add或者poll方法，这样我们自己不需要做任何同步。

但如果是非原子操作，比如：

```
if(!queue.isEmpty()) {
    queue.poll(obj);
}
```

我们很难保证，在调用了isEmpty()之后，poll()之前，这个queue没有被其他线程修改。所以对于这种情况，我们还是需要自己同步：

```
synchronized(queue) {
    if(!queue.isEmpty()) {
        queue.poll(obj);
    }
}
```

**这种需要进行自己同步的情况要视情况而定，不是任何情况下都需要这样做。 **

**ConcurrentLinkedQueue的size()是要遍历一遍集合的，所以尽量要避免用size而改用isEmpty()，以免性能过慢。**

  
