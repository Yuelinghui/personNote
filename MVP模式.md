##MVP模式简介

MVP模式是从经典的MVC模式演化来的，它们俩基本是相通的，Model是**提供数据的**，View是**界面显示**，Presenter/Controller是**负责逻辑处理**

###MVP和MVC的区别

在MVC里，View是可以直接访问Model的！从而，View里会包含Model信息，不可避免的还要包括一些业务逻辑。 在MVC模型里，更关注的Model的不变，而同时有多个对Model的不同显示，即View。所以，在MVC模型里，Model不依赖于View，但是View是依赖于Model的。不仅如此，因为有一些业务逻辑在View里实现了，导致要更改View也是比较困难的，至少那些业务逻辑是无法重用的。

在MVP里，Presenter完全把Model和View进行了分离，主要的程序逻辑在Presenter里实现。而且，Presenter与具体的View是没有直接关联的，而是通过定义好的接口进行交互，从而使得在变更View时候可以保持Presenter的不变，即重用！

MVP模式的图示：
![](http://ofowf99vj.bkt.clouddn.com/mvp.png)
View和Presenter双向持有，Presenter持有Model对象注意是单向的，View和Model不会互相有瓜葛。

在MVC模式中，View一般是Fragment或者xml，Activity一般都是**Controller**的角色，它负责获取数据，处理数据，并把数据和View关联起来。在MVP模式里，Activity和Fragment就是**View**了，业务逻辑抽出到Presenter中。

我们用google官方提供的MVP的例子来解释。[github地址](https://github.com/googlesamples/android-architecture.git)

首先，我们看一下项目目录
![](http://ofowf99vj.bkt.clouddn.com/android-architecture.png)
这个项目是一个类似记事本的app，代码比较简单，我们可以看目录结构是按照模块划分的，分为**添加事项**（addedittask），**所有事件的统计**（statistics），**事项详情**（taskdetail），**首页事项列表**（tasks），**事件数据**（data，使用数据库保存本地）和**工具类**（util），最外层还有两个类，**BasePresenter**和**BaseView**，这两个类就是P和V的基类，是接口类。
```
public interface BaseView<T> {
    void setPresenter(T presenter);
}


public interface BasePresenter {
    void start();
}

```

每个模块都分为**Activity**，**Contract**，**Fragment**和**Presenter**。我们就从M，V，P这三个方面来讨论一个模块（**以addedittask为例**）

在介绍MVP各个部分之前，我们要先介绍Contract，这个结合View和Presenter，其实就是一个“胶水代码”，把View和Presenter**粘合**在一块的功能。

###Contract

我们看AddEditTaskContract接口，里面定义了这个View和Presenter需要实现的方法，然后在Presenter里会根据获取的数据来操作View。

###Model

其实整个项目只有一个model，就是data里的TasksLocalDataSource，所有的数据都是从这里获取，即从数据库获取。

TasksDataSource接口定义了获取数据的方法，TasksLocalDataSource实现了这个接口，从数据库里获取数据，并使用LoadTasksCallback回调。

###View

AddEditTaskActivity和AddEditTaskFragment就是展示的View，先看Activity

* 生成了Toolbar和ActionBar并进行设置。
* 生成了具体展示的Fragment，并且展示。
* 生成了这个模块的Presenter

我觉得Activity其实就是声明并且初始化View和Presenter，然后操作一下Toolbar和ActionBar，具体怎么展示数据是放在Fragment里面的。

然后是Fragment，实现了Contract里的View接口，主要任务就是定义布局并展示数据。所以Fragment里的代码非常简洁，就是通过实现Contract.View接口来获取需要展示的数据并显示出来，根本没有处理数据的逻辑。Fragment在onResume()的时候设置了自己对应的Presenter。就像最开始那张图一样，View是持有Presenter的。

###Presenter

这个是MVP里最重要的一环，它连接了Model和View，负责中间的数据，逻辑处理。

AddEditTaskPresenter实现了Contract.Presenter接口和GetTaskCallback接口，并且我们看到Presenter还持有Contract.View的实例对象。调用start()来获取数据

```
@Overridepublic void start() {
    if (!isNewTask()) {
        populateTask();
    }
}
```
当调用start()方法时，判断Task初始化状态，然后调用populateTask()。

```
@Overridepublic void populateTask() {
    if (isNewTask()) {
        throw new RuntimeException("populateTask() was
            called but task is new.");
    }
    mTasksRepository.getTask(mTaskId, this);}
```

使用TasksRepository来从数据库里获取数据，然后在`onTaskLoaded()`回调中拿到数据。

```
@Overridepublic void onTaskLoaded(Task task) {
    if (mAddTaskView.isActive()) {
        mAddTaskView.setTitle(task.getTitle());
        mAddTaskView.setDescription(task.getDescription());
     }
}

```