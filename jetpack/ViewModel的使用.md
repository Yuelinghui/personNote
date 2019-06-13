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

注意： **ViewModel绝不能引用View，Lifecycle或任何可能包含对`activity`的`context`引用的类**

## ViewModel的生命周期

获取`ViewMode`l时，`ViewModel`对象的生命周期为传递给`ViewModelProvider`的`this`的生命周期。`ViewModel`保留在内存中，直到它的作用域生命周期永久消失：在`activity`的情况下，是`finished`，在`fragment`的情况下，是`detached`

## 在fragment中间共享数据

`activity`中的两个或多个`fragment`需要相互通信是很常见的。

可以使用`ViewModel`对象解决这个常见的问题。这些`fragment`可以使用其`activity`范围内共享的`ViewModel`来处理通信

```
class  SharedViewModel  :  ViewModel()  { 
	val selected =  MutableLiveData<Item>()  
	fun select(item:  Item)  { 
		selected.value = item 
	}  
}  
  
class  MasterFragment  :  Fragment()  {  
private lateinit var itemSelector:  Selector  
private lateinit var model:  SharedViewModel  

override  fun onCreate(savedInstanceState:  Bundle?)  {  
	super.onCreate(savedInstanceState) 
	model = activity?.run {  ViewModelProviders.of(this).get(SharedViewModel::class.java)  }  ?:  throw  Exception("Invalid Activity") 
	itemSelector.setOnClickListener { item ->  
		// Update the UI  
	}  
}  
}  
  
class  DetailFragment  :  Fragment()  {  
private lateinit var model:  SharedViewModel  

override  fun onCreate(savedInstanceState:  Bundle?)  {  
	super.onCreate(savedInstanceState) 
	model = activity?.run {  ViewModelProviders.of(this).get(SharedViewModel::class.java)  }  ?:  throw  Exception("Invalid Activity") 
	model.selected.observe(this,  Observer<Item>  { item ->  
		// Update the UI  
	})  
}  
}
```

这种方法具有以下优点：

* `activiy`不需要做任何事情，也不需要了解这种沟通
* 除了`SharedViewModel`的接口方法之外，`fragment`不需要彼此了解。如果其中一个`fragment`消失，另一个`f`继续照常工作。
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTUwNDIxMDI1Nyw2NDc1Mzg4NTRdfQ==
-->