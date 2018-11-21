# Data Binding的使用

## 简介

Data Binding是一个支持库，允许使用声明性格式而不是以编程方式将布局中的UI组件绑定到应用程序的数据源。

例如我们给一个textview添加显示文字：

```
findViewById<TextView>(R.id.text).apply{
    text = viewModel.userName
}
```

我们可以使用Data Binding直接在布局文件中为textview分配文字，这样就不需要写上面那些代码了

```
<TextView
    android:text="@{viewModel.userName}"/>
```

在布局文件中使用Data Binding可以让我们少写很多简单重复的代码，使其更简单，更易于维护。还可以提高应用程序的性能，并且有助于防止内存泄漏和空指针异常。

## 环境配置

建议在项目中使用最新的Gradle插件。

先下载`Support Repository`。要将应用程序配置为使用Data Binding，在app的gradle文件中添加以下内容：

```
android {
    ...

    dataBinding {
        enabled = true
    }
}
```

## 布局和绑定表达式

表达式语言允许编写处理视图调度的事件。Data Binding自动生成将布局中的视图与数据对象绑定所需的类。

```
<layout xmlns:android="http://schemas.android.com/apk/res/android">

    <data>
       <variable name="user" type="com.example.User"/>
   </data>

   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">

       <TextView 
           android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.firstName}"/>

       <TextView 
           android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.lastName}"/>

   </LinearLayout>

</layout>
```

* `layout`后面跟着`data`和`view`根元素。其中user描述了可在此布局中使用的属性。
* 表达式使用`@{}`写入属性。上面的两个textview分别显示的是user的firstName和lastName

### 绑定数据

Data Binding为每个布局文件生成绑定类。默认情况下，类的名称基于布局文件的名称，将其转换为Pascal大小写并向其添加Binding后缀。比如`activit_main.xml`相应的生成的类为`ActivityMainBinding`

```
onCreate() {
    ...
    val activityBinding = DataBindingUtil.setContentView<ActivityMainBinding>(this,R.layout.activity_main)

    activityBinding.user = User("firstName","lastName")
}
```

还可以使用另一种方式：

```
val binding = ActivityMainBinding.infalate(getLayoutInflater())
```

在Fragment或者RecyclerView的ViewHolder中可以这样：

```
val view:ItemBinding = DataBindingUtil.inflate(layoutInflater,R.layout.item,viewGroup,false)
// or
val view = ItemBinding.inflate(layoutInflater,viewGroup,false)
```

### 表达语言

可以在表达式语言中使用以下运算符和关键字：

* 数学的`+`,`-`,`*`,`/`等
* 字符串连接的`+`
* 逻辑运算符`&&`,`||`
* 二进制`&`,`|`,`^`
* 对比`==`,`>`,`<`
* 方法调用
* 三元运算符`?:`
* 数组访问

...

```
android:text="@{String.valueOf(index+1)}"
android:visibility="@{age<13 ?: View.GONE:View.VISIABLE}"
android:text="@{"abc"+id}"
```

#### 空结合运算符

空合并运算符(`??`)，若不为null，则选择左侧参数，若为null，则选择右侧：

```
android:text="@{user.firstName??user.lastName}"
// 等同于
android:text="@{user.firstName != null ? user.firstName : user.lastName}"
```
#### 避免空指针异常

生成的数据绑定代码会自动检查空值并避免空指针异常。比如`@{user.name}`，若user为null，user.name则为其分配默认值null。比如`@{user.age}`，age的类型是int，则使用默认值0。

### 事件处理

Data Binding允许我们编写在视图中调度的表达式来处理事件（例如：`onClick()`）。

可以使用两种机制来处理事件：

* **方法引用**：在表达式中，可以引用符合监听器方法签名的方法。Data Binding将方法引用和所有者对象包装在监听器中，并在目标视图上设置该监听器。若表达式为null，则Data Binding不会创建监听器并设置null监听器

* **监听器绑定**：在事件发生时运行的lambda表达式。

#### 方法引用

事件直接绑定到处理方法上。表达式在编译时处理，若该方法不存在或签名不正确，会收到编译时错误。

方法引用的监听器是在绑定数据时创建的，而不是在出发事件时创建的。若希望在事件发生时运行表达式，则应使用监听器绑定。

```
class MyHandlers{
    fun onClick(view:View){
        ...
    }
}

<data>
    <variable name="handlers" type="com.example.MyHandlers"/>
</data>

<TextView
    android:onClick="@{handlers::onClick}"/>
```

***表达式中方法的签名必须与监听器对象中方法的签名完全匹配**

#### 监听器绑定

监听器绑定是在事件发生时运行的绑定表达式。类似于方法引用，但允许运行任意Data Bing表达式。

在方法引用中，方法的参数必须与事件监听器的参数匹配。在监听器绑定中，只有返回值必须与监听器的预期返回值匹配就行

```
class Presenter{
    fun onSaveClick(task:Tesk){
        ...
    }
}
// 
<data>
    <variable name="task" type="com.example.Task"/>
    <variable name="Presenter" type="com.example.Presenter"/>

    <Button
        android:onClick="@{()->presenter.onSaveClick(task)}"
    />
</data>
```
监听器绑定为监听器参数提供了两种选择：可以忽略方法的所有参数，也可以命名所有参数

监听器表达式非常强大，可以使代码非常易读。另一方面，包含复杂表达式的监听器使布局难以阅读和维护。

### imports，variables和includes

Data Binding提供了imports（使布局文件中的类容易引用），variables（描述可用于绑定表达式的属性）和includes（在整个应用中重复使用复杂的布局）等功能。

`import`可以在`data`元素内使用零个或多个元素

```
<data>
    <import type="android.view.View"/>
</data>

<TextView
    android:visibility="@{user.isAdult ? View.VISIABLE : View.GONE}"
/>
```

当存在类名冲突时，可以将其中一个类重命名为别名：

```
<data>
    <import type="android.view.View"/>
    <import type="com.example.View"
        alias="Vista"/>
</data>
```

可以使用`Vista`来引用`com.example.View`类

可以使用导入的类型来转换表达式的一部分：

```
<TextView
    android:text="@{((User)(user.name)).lastName}"
/>
```

在表达式中引用静态字段和方法时，也可以使用导入的类型：

```
<data>
    <import type="com.example.Utils"/>
</data>
//
<TextView
    android:text="@{Utils.getName(user.lastName)}"
/>
```

`variable`可以定义多个元素。每个元素都描述了一个属性，该属性可以在不居中设置，以便在布局文件中使用

变量可以从包含的布局传递到被包含的布局中：

```
<LinearLayout>
    ...
    <include layout="@layout/name"
        bind:user="@{user}"/>
</LinearLayout>
```

Data Binding不支持`include`作为merge元素的直接子元素：

```
// 以下布局是不支持的
<merge>
    <include layout="@layout/name"
    bind:user="@{user}"/>
</merge>
```

## 使用可观察的数据对象

可观察是指对象通知其他人数据变化的能力。任何普通都可用于Data Binding，但修改对象不会自动更新UI。Data Binding可用于为数据对象提供在数据更改时通知其他对象的能力。有三种不同类型的可观察类：对象，字段和集合。

### 字段
可以使用泛型Observable类和以下特定的类来使字段可观察：

* ObservableBoolean
* ObservableInt
* ObservableByte
* ObservableChar
* ObservableShort
* ObservableLong
* ObservableFloat
* ObservableDouble
* ObservableParcelable

在Java中创建`public final`属性或在Kotlin中创建只读属性（val）
```
class User {
    val firstName = ObservableField<String>()
    val age = ObservableInt()
}
```

**Android Studio 3.1及更高版本允许使用LiveData对象替换可观察对象。**

### 集合

ObservableArrayMap和ObservableArrayList

### 可观察对象

实现Observable接口的类，允许注册希望被通知可观察对象的属性更改的监听器

Observable接口具有添加和删除监听器的机制，但是必须决定何时发送通知。Data Binding提供了BaseObservable实现监听器注册机制的类。实现BaseObservable负责通知属性何时更改。

```
class User : BaseObservable() {
    
    @get:Bindable
    var firstName:String = ""
        set(value) {
            field = value
            notifyPropertyChanged(BR.firstName)
    }
}
```

Data Binding会在模块包中生成`BR`类，包含用户数据绑定的资源ID。`Bindable`注解产生的一个了BR中的一个条目。

## 生成的绑定类

生成的绑定类将布局变量与布局中的视图联系起来。绑定类的名称和包可以自定义。所有生成的绑定类都继承自`ViewDataBinding`类。

### 带id的视图

Data Binding库在绑定类中为每个在布局中具有ID的视图创建不可变字段：

```
...
<TextView 
    android:id="@+id/txt_name"
/>
...

//可以在绑定类中
binding.txtName.setText("abc")
```

Data Binding库为布局中声明的每个变量生成访问器方法。

### 立即绑定

当变量或可观察对象发生更改时，需要立即执行绑定。要强制执行，使用`executePendingBindings()`

### 高级绑定

#### 动态变量

有时，特定的绑定类是未知的。比如RecyclerView.Adapter针对任意布局的操作不知道特定的绑定类，但是它仍然必须在调用onBindViewHolder()方法期间分配绑定值。

```
override fun onBindViewHolder(holder: BindingHolder, position: Int) {
    item: T = mItems.get(position)
    holder.binding.setVariable(BR.item, item);
    holder.binding.executePendingBindings();
}
```

#### 自定义绑定类名称

默认情况下，将根据布局文件的名称生成绑定类。通过调整`class`属性，可以重命名绑定类或将绑定类放在不同的包中：

```
<data class="ContactItem"></data>>
```
可以使用完整包名称：

```
<data class="com.example.ContactItem"></data>>
```

## 绑定适配器

绑定适配器负责对设置值进行适当的框架调用。Data Binding允许指定调用的方法来设置值，提供自己的绑定逻辑，并使用适配器指定返回对象的类型

### 设置属性值

每当绑定值发生更改时，生成的绑定类必须使用绑定表达式在视图上调用setter方法。可以允许Data Binding库自动确定方法，显式声明方法或提供自定义逻辑来选择方法。

#### 自动方法选择

对于属性example，库会自动尝试查找`setExample(arg)`的方法。不考虑属性的名称空间，搜索方法时仅使用属性名称和类型。

#### 指定自定义方法名称

某些属性具有名称不匹配的setter。这时，可以使用`BindingMethods`注解将属性与设置器相关联：

```
@BindingMethods(value = [BindingMethod(type = android.widget.ImageView::class,
                    attribute = "android:tint",
                    method = "setImageTintList")])
```

#### 提供自定义逻辑

某些属性需要自定义绑定逻辑。比如`android:paddingLeft`属性没有关联的setter，但是`setPadding(left,top,right,bottom)`提供了方法。

```
@BindingAdapter("android:paddingLeft")
fun setPaddingLeft(view:View, padding:Int) {
    view.setPadding(padding,view.getPaddingTop(),
            view.getPaddingRight(),view.getPaddingBottom())
}
```

参数类型很重要。第一个参数确定与属性关联的视图类型，第二个参数确定给定属性的绑定表达式中接收的类型。

可以拥有接收多个属性的适配器：

```
@BindingAdapter("imageUrl","error")
fun loadImage(view:ImageView,url:String,error:Drawable) {
    ImageLoader.load(url).error(error).into(view)
}
```

可以在布局中使用适配器：

```
<ImageView 
    app:imageUrl="@{venue.imageUrl}" 
    app:error="@{@drawable/venueError}" 
/>
```

上面的适配器同时有imageUrl和error两个属性作用于一个对象。若要在设置**任何属性**时调用适配器，可以把`requireAll`设置成false：

```
@BindingAdapter(value = ["imageUrl","error"],requireAll = false)
fun loadImage(view:ImageView,url:String,error:Drawable) {
    ImageLoader.load(url).error(error).into(view)
}
```

**发生冲突时，绑定适配器会覆盖默认的适配器**

绑定适配器方法可以选择在其处理程序中使用就只。首先要声明属性的**所有**旧值，然后是新值：

```
@BindingAdapter("android:paddingLeft")
fun setPaddingLeft(view: View, oldPadding: Int, newPadding: Int) {
    if (oldPadding != newPadding) {
        view.setPadding(padding,
                    view.getPaddingTop(),
                    view.getPaddingRight(),
                    view.getPaddingBottom())
    }
}
```

事件处理程序只能与带有一个抽象方法的接口或抽象类一起使用：

```
@BindingAdapter("android:onLayoutChange")
fun setOnLayoutChangeListener(
        view: View,
        oldValue: View.OnLayoutChangeListener?,
        newValue: View.OnLayoutChangeListener?
) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {
        if (oldValue != null) {
            view.removeOnLayoutChangeListener(oldValue)
        }
        if (newValue != null) {
            view.addOnLayoutChangeListener(newValue)
        }
    }
}
```

当监听器具有多个方法时，必须将其拆分为多个监听器。例如：`View.OnAttachStateChangeListener`有两个方法：`onViewAttachedToWindow(View)`和`onViewDetachedFromWindow(View)`：

```
@BindingAdapter(
        "android:onViewDetachedFromWindow",
        "android:onViewAttachedToWindow",
        requireAll = false
)
fun setListener(view: View, detach: OnViewDetachedFromWindow?, attach: OnViewAttachedToWindow?) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB_MR1) {
        val newListener: View.OnAttachStateChangeListener?
        newListener = if (detach == null && attach == null) {
            null
        } else {
            object : View.OnAttachStateChangeListener {
                override fun onViewAttachedToWindow(v: View) {
                    attach?.onViewAttachedToWindow(v)
                }

                override fun onViewDetachedFromWindow(v: View) {
                    detach?.onViewDetachedFromWindow(v)
                }
            }
        }

        val oldListener: View.OnAttachStateChangeListener? =
                ListenerUtil.trackListener(view, newListener, R.id.onAttachStateChangeListener)
        if (oldListener != null) {
            view.removeOnAttachStateChangeListener(oldListener)
        }
        if (newListener != null) {
            view.addOnAttachStateChangeListener(newListener)
        }
    }
}
```

### 对象转换

#### 自动对象转换

当Object从绑定表达式返回一个值时，库会选择用于设置属性值的方法。

```
<TextView
   android:text="@{userMap["lastName"]}"/>
```

userMap表达式中的对象返回一个值，该值自动转换为`setText(CharSequence)`里的参数类型。

#### 自定义转化

在某些情况下，特定类型之间需要自定义转换。比如：`android:background`属性需要一个Drawable，但是color指定的值是整数：

```
<View
   android:background="@{isError ? @color/red : @color/white}"/>
```

int应该转换为ColorDrawable，可以使用带`BindingConversion`注解的方法完成转换：

```
@BindingConversion
fun convertColorToDrawable(color: Int) = ColorDrawable(color)
```

但是，绑定表达式中提供的值类型必须一致。不能在同一个表达式中使用不同的类型：

```
// 下面是不对的
<View
   android:background="@{isError ? @drawable/error : @color/white}"/>
```

## 双向数据绑定

使用单向数据绑定，可以在属性上设置值，并设置对该属性中的更改作出反应的监听器：

```
<CheckBox
    android:id="@+id/rememberMeCheckBox"
    android:checked="@{viewmodel.rememberMe}"
    android:onCheckedChanged="@{viewmodel.rememberMeChanged}"
/>
```

双向数据绑定提供了此过程的快捷方式：

```
<CheckBox
    android:id="@+id/rememberMeCheckBox"
    android:checked="@={viewmodel.rememberMe}"
/>

//

class LoginViewModel : BaseObservable {
    // val data = ...

    @Bindable
    fun getRememberMe(): Boolean {
        return data.rememberMe
    }

    fun setRememberMe(value: Boolean) {
        // Avoids infinite loops.
        if (data.rememberMe != value) {
            data.rememberMe = value

            // React to the change.
            saveData()

            // Notify observers of a new value.
            notifyPropertyChanged(BR.remember_me)
        }
    }
}
```

使用`@={}`符号，接收数据变化的属性，并在同一时间监听用户的更新

### 使用自定义属性进行双向数据绑定

若要对自定义属性使用双向数据绑定，需要使用`@InverseBindingAdapter`和`@InverseBindingMethod`注解。

* 注解设置初始值，并在值更改时使用`@BindingAdapter`进行更新：
    
``` 
@BindingAdapter("time")
@JvmStatic fun setTime(view: MyView, newValue: Time) {
    // Important to break potential infinite loops.
    if (view.time != newValue) {
        view.time = newValue
    }
}
```
* 使用下面的方法从视图中读取值
```
@InverseBindingAdapter("time")
@JvmStatic fun getTime(view: MyView) : Time {
    return view.getTime()
}
```

此时，Data Binding知道数据更改时要做什么（调用带`@BindingAdapter`的方法）以及视图属性更改时调用的内容（调用`InverseBindingListener`）。但是，它不知道属性何时或如何更改。

为此，需要在视图上设置一个监听器。它可以是与自定义视图关联的自定义监听器，也可以是通用事件：

```
@BindingAdapter("app:timeAttrChanged")
@JvmStatic fun setListeners(
        view: MyView,
        attrChange: InverseBindingListener
) {
    // Set a listener for click, focus, touch, etc.
}
```

### 转换器

若绑定到View对象的变量需要在显示之前以某种方式进行格式化，翻译或更改，可以使用`Converter`对象。

```
<EditText
    android:id="@+id/birth_date"
    android:text="@={Converter.dateToString(viewmodel.birthDate)}"
/>
```

viewmodel中的birthDate是Long，我们需要把它格式化成String类型的数据（xxxx-xx-xx）。

由于正在使用双向表达式，因此在这时，还需要提供**逆转换器**让库知道如何将用户提供的字符串转换为数据类型的Long

```
object Converter {
    @InverseMethod("stringToDate")
    fun dateToString(
        view: EditText, oldValue: Long,
        value: Long
    ): String {
        // Converts long to String.
    }

    fun stringToDate(
        view: EditText, oldValue: String,
        value: String
    ): Long {
        // Converts String to long.
    }
}
```
### 使用双向绑定的无限循环

使用双向数据绑定时，不要引入**无限循环**。当用户更改属性时，将调用`@InverseBindingAdapter`注解的方法，并将值分配给属性。反过来，将调用`@BindingAdapter`注解的方法，将触发`@InverseBindingAdapter`方法的另一次调用，以此类推。

因此，通过比较使用注解的方法中的新旧值来打破可能的无限循环非常重要。

