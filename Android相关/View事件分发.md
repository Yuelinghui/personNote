# View的事件分发

啥也别说，先上图
![](/assets/view_dispatch.png)

图分为三层，从上到下依次为Activity，ViewGroup，View

1. 如果不中断，一直调用super，整个事件就是一个类U型图。整个事件流向应该是从Activity---->ViewGroup--->View 从上往下调用dispatchTouchEvent方法，一直到叶子节点（View）的时候，再由View--->ViewGroup--->Activity从下往上调用onTouchEvent方法。

2. dispatchTouchEvent 和 onTouchEvent 一旦return true,事件就停止传递了（到达终点）（没有谁能再收到这个事件）。看图中只要return true事件就没再继续传下去了，对于return true我们经常说事件被消费了，消费了的意思就是事件走到这里就是终点，不会往下传，没有谁能再收到这个事件了。

3. dispatchTouchEvent 和 onTouchEvent return false的时候事件都回传给父控件的onTouchEvent处理。
    * 对于dispatchTouchEvent 返回 false 的含义应该是：事件停止往子View传递和分发同时开始往父控件回溯（父控件的onTouchEvent开始从下往上回传直到某个onTouchEvent return true），事件分发机制就像递归，return false 的意义就是递归停止然后开始回溯。
    * 对于onTouchEvent return false 就比较简单了，它就是不消费事件，并让事件继续往父控件的方向从下往上流动。

4. dispatchTouchEvent、onTouchEvent、onInterceptTouchEvent
ViewGroup 和View的这些方法的默认实现就是会让整个事件安装U型完整走完，所以 return super.xxxxxx() 就会让事件依照U型的方向的完整走完整个事件流动路径），中间不做任何改动，不回溯、不终止，每个环节都走到。

5. onInterceptTouchEvent 的作用。Intercept 的意思就拦截，每个ViewGroup每次在做分发的时候，问一问拦截器要不要拦截（也就是问问自己这个事件要不要自己来处理）如果要自己处理那就在onInterceptTouchEvent方法中 return true就会交给自己的onTouchEvent的处理，如果不拦截就是继续往子控件往下传。默认是不会去拦截的，因为子View也需要这个事件，所以onInterceptTouchEvent拦截器return super.onInterceptTouchEvent()和return false是一样的，是不会拦截的，事件会继续往子View的dispatchTouchEvent传递。

6. ViewGroup 和View 的dispatchTouchEvent方法返回super.dispatchTouchEvent()的时候事件流走向。
    * 首先看下ViewGroup 的dispatchTouchEvent，之前说的return true是终结传递。return false 是回溯到父View的onTouchEvent，然后ViewGroup怎样通过dispatchTouchEvent方法能把事件分发到自己的onTouchEvent处理呢，return true和false 都不行，那么只能通过Interceptor把事件拦截下来给自己的onTouchEvent，所以ViewGroup dispatchTouchEvent方法的super默认实现就是去调用onInterceptTouchEvent，记住这一点。
    * 那么对于View的dispatchTouchEvent return super.dispatchTouchEvent()的时候呢事件会传到哪里呢，很遗憾View没有拦截器。但是同样的道理return true是终结。return false 是回溯会父类的onTouchEvent，怎样把事件分发给自己的onTouchEvent 处理呢，那只能return super.dispatchTouchEvent,View类的dispatchTouchEvent（）方法默认实现就是能帮你调用View自己的onTouchEvent方法的。
    
## 总结

说了这么多，不知道有说清楚没有，最后总结一下：

* 对于 dispatchTouchEvent，onTouchEvent，return true是终结事件传递。return false 是回溯到父View的onTouchEvent方法。
* ViewGroup 想把自己分发给自己的onTouchEvent，需要拦截器onInterceptTouchEvent方法return true 把事件拦截下来。
* ViewGroup 的拦截器onInterceptTouchEvent 默认是不拦截的，所以return super.onInterceptTouchEvent()=return false；
* View 没有拦截器，为了让View可以把事件分发给自己的onTouchEvent，View的dispatchTouchEvent默认实现（super）就是把事件分发给自己的onTouchEvent。


### ViewGroup和View 的dispatchTouchEvent 是做事件分发，那么这个事件可能分发出去的四个目标

1. 自己消费，终结传递。------->return true ；
2. 给自己的onTouchEvent处理-------> 调用super.dispatchTouchEvent()系统默认会去调用 onInterceptTouchEvent，在onInterceptTouchEvent return true就会去把事件分给自己的onTouchEvent处理。
3. 传给子View------>调用super.dispatchTouchEvent()默认实现会去调用 onInterceptTouchEvent 在onInterceptTouchEvent return false，就会把事件传给子类。
4. 不传给子View，事件终止往下传递，事件开始回溯，从父View的onTouchEvent开始事件从下到上回归执行每个控件的onTouchEvent------->return false；
    
**注： 由于View没有子View所以不需要onInterceptTouchEvent 来控件是否把事件传递给子View还是拦截，所以View的事件分发调用super.dispatchTouchEvent()的时候默认把事件传给自己的onTouchEvent处理（相当于拦截），对比ViewGroup的dispatchTouchEvent 事件分发，View的事件分发没有上面提到的4个目标的第3点。**
    
### ViewGroup和View的onTouchEvent方法是做事件处理的，那么这个事件只能有两个处理方式：

1. 自己消费掉，事件终结，不再传给谁----->return true;
2. 继续从下往上传，不消费事件，让父View也能收到到这个事件----->return false;View的默认实现是不消费的。所以super==false。

###ViewGroup的onInterceptTouchEvent方法对于事件有两种情况：

1. 拦截下来，给自己的onTouchEvent处理--->return true;
2. 不拦截，把事件往下传给子View---->return false,ViewGroup默认是不拦截的，所以super==false；

### 关于ACTION_MOVE 和 ACTION_UP

上面讲解的都是针对ACTION_DOWN的事件传递，ACTION_MOVE和ACTION_UP在传递的过程中并不是和ACTION_DOWN 一样，你在执行ACTION_DOWN的时候返回了false，后面一系列其它的action就不会再得到执行了。简单的说，就是当dispatchTouchEvent在进行事件分发的时候，只有前一个事件（如ACTION_DOWN）返回true，才会收到ACTION_MOVE和ACTION_UP的事件。

* 如果在某个控件的dispatchTouchEvent 返回true消费终结事件，那么收到ACTION_DOWN 的函数也能收到 ACTION_MOVE和ACTION_UP。

* 在哪个View的onTouchEvent 返回true，那么ACTION_MOVE和ACTION_UP的事件从上往下传到这个View后就不再往下传递了，而直接传给自己的onTouchEvent 并结束本次事件传递过程。

对于ACTION_MOVE、ACTION_UP总结：**ACTION_DOWN事件在哪个控件消费了（return true）， 那么ACTION_MOVE和ACTION_UP就会从上往下（通过dispatchTouchEvent）做事件分发往下传，就只会传到这个控件，不会继续往下传，如果ACTION_DOWN事件是在dispatchTouchEvent消费，那么事件到此为止停止传递，如果ACTION_DOWN事件是在onTouchEvent消费的，那么会把ACTION_MOVE或ACTION_UP事件传给该控件的onTouchEvent处理并结束传递。**
    
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTkzNzY2MDk4M119
-->