# LiveData

LiveData是一个可观察的数据持有者，它可以感知生命周期，只有在`active`的时候才会向观察者更新数据。

##使用LiveData的好处

* 确保UI符合数据状态
* 不会有内存泄漏（观察者绑定到生命周期对象，当生命周期destroy的时候会自行清理）
* 不会因activity的`stopped`而引起崩溃（当activity `stop`）
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIwNDU4NzI4NjAsNzI4NjczNTg1XX0=
-->