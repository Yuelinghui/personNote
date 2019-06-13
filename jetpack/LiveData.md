# LiveData

LiveData是一个可观察的数据持有者，它可以感知生命周期，只有在`active`的时候才会向观察者更新数据。

##使用LiveData的好处

* 确保UI符合数据状态
* 不会有内存泄漏（观察者绑定到生命周期对象，当生命周期destroy的时候会自行清理）
* 不会因activity的`stopped`而引起崩溃（当activity `stopped`之后，观察者不会收到数据变动信息）
* 不再需要手动生命周期清理
* 始终保持最新的数据（当生命周期变为不活跃了，不会收到数据变动，会在重新变为活跃的时候收到最新的数据）
* 当*configuration change*发生时，可以收到最新的数据
* 资源共享
<!--stackedit_data:
eyJoaXN0b3J5IjpbMzE3OTU1MjM2LDcyODY3MzU4NV19
-->