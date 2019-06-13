# LiveData

LiveData是一个可观察的数据持有者，它可以感知生命周期，只有在`active`的时候才会向观察者更新数据。

##使用LiveData的好处

* 确保UI符合数据状态
* 不会有内存泄漏（观察者绑定到生命周期对象，当生命周期destroy的时候会自行清理）
* 不会因activity的`stopped`而引起崩溃（当activity `stopped`之后，观察者不会收到数据变动信息）
* 不再需要手动生命周期清理
* 始终保持最新的数据（当生命周期变为不活跃了，不会收到数据变动，会在重新变为活跃的时候收到最新的数据）
* 当*configuration change*发生时，可以收到最新的数据
* 资源共享（可以使用单例模式扩展LiveData对象以包装系统服务，以便可以在应用程序中共享它们。LiveData对象连接到系统服务一次，然后任何需要该资源的观察者都可以只观看LiveData对象）

## 观察LiveData对象

```
liveData.observe(this,Observer{data->
// 数据变动的回调
})
```

方法里的`this`就是`LifecycleOwner`对象，比如`activity`，`fragment`等

## 更新LiveData

LiveData
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTIyNzE5NTE3OCw1NTA0NjIyODksNzI4Nj
czNTg1XX0=
-->