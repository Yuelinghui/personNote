# ViewModel的使用

`ViewModel`类旨在以生命周期意识的方式存储和管理与UI相关的数据。`ViewModel`类允许数据在`configuration changes`（例如屏幕旋转）后继续存在。

当系统销毁并重建`activity`的时候，可能和UI相关的数据会消失，比如`configuration changes`(例如旋转屏幕)造成的`activity`重建。当然，我们可以在`onSaveInstanceState()`和`onCreate`的时候存储和恢复数据，但是这个只对可序列化的数据有效，而且数据量大的话不建议使用

另一个问题是UI控制器经常需要进行需要一些时间才能返回的异步调用（例如网络请求数据）。UI控制器需要管理这些调用并确保`activity`或者`fragment`在销毁后清理它们以避免潜在的内存泄漏。这种管理需要大量维护，并且在重新创建对象的情况下，这会浪费资源，因为对象可能必须重新发出已经进行的调用。


## 使用ViewModel
`ViewModel`对象在`configuration changes`时自动保留，以便它们保存的数据可立即用于下一个`activity`或`fragment`

```
class  MyViewModel  :  ViewModel()  {  
private  val users:  MutableLiveData<List<User>> by lazy {  MutableLiveData().also { loadUsers()  }  }  

fun getUsers():  LiveData<List<User>>  {  
	return users 
}  

private  fun loadUsers()  {  
// Do an asynchronous operation to fetch users.  
}
```
```
class  MyActivity  :  AppCompatActivity()  {  

override  fun onCreate(savedInstanceState:  Bundle?)  {  

// Create a ViewModel the first time the system calls an activity's onCreate() method. 
// Re-created activities receive the same MyViewModel instance created by the first activity.  

val model =  ViewModelProviders.of(this).get(MyViewModel::class.java) 

model.getUsers().observe(this,  Observer<List<User>>{ users ->  
	// update UI 
})  
}  
}
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTU0Nzk3NjIyLDY0NzUzODg1NF19
-->