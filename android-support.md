##Support Design Lib包的使用

Google在2015的IO大会上，给我们带来了更加详细的**Material Design**设计规范，同时，也给我们带来了全新的**Android Design Support Library**，在这个support库里面，Google给我们提供了更加规范的MD设计风格的控件。最重要的是，Android Design Support Library的兼容性更广，直接可以**向下兼容到Android 2.2**。

里面的新控件实现一些特殊效果确实是非常厉害，不得不说这是Google推出的一款良心之作。

###SnackBar

Snackbar提供了一个介于Toast和AlertDialog之间轻量级控件，它可以很方便的提供消息的提示和动作反馈。SnackBar显示一段时间之后就会消失，和Toast一样。 

```
Snackbar.make(view, "Snackbar comes out",Snackbar.LENGTH_LONG) 
.setAction("Action", new View.OnClickListener() { 
    @Override public void onClick(View v) {
        Toast.makeText(MainActivity.this, "Toast comes out", Toast.LENGTH_SHORT).show();
    } 
}).show();

```
使用方法和Toast类似，第一个参数是SnackBar显示的基准元素，Action可以设置多个。

###TextInputLayout

TextInputLayout作为一个父容器控件，包装了新的EditText。通常，单独的EditText会在用户输入第一个字母之后隐藏hint提示信息，但是现在你可以使用TextInputLayout来将EditText封装起来，提示信息会变成一个显示在EditText之上的floating label，这样用户就始终知道他们现在输入的是什么。

```
<android.support.design.widget.TextInputLayout
    android:id="@+id/til_pwd"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"> 

    <EditText
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>

</android.support.design.widget.TextInputLayout>

```
注意：**TextInputLayout是把EditText包含起来的，不能单独使用**

在代码中使用

```
TextInputLayout textInputLayout = (TextInputLayout) findViewById(R.id.til_pwd); 
// 设置hint
textInputLayout.setHint("Password");
// 获取EditText
EditText editText = textInputLayout.getEditText();
```

TextInputLayout有一个`setError(String error)`方法，就是我们在校验输入的信息时，如果发现信息错误，调用这个方法就会在EditText下方显示红色的错误信息。

###Floating Action Button

FAB是一个负责显示界面基本操作的圆形按钮，把它当成一个Button来使用就可以。

我们可以指定anchor，即显示的锚点`app:layout_anchor="@id/app_bar"`

FAB是继承自ImageView的，我们可以使用ImageView的任意方法，比如使用`android:src="@drawable/icon_fab"`来指定它显示的图标

###TabLayout

Tab滑动切换View并不是一个新的概念，但是Google却是第一次在support库中提供了完整的支持，而且，Design library的TabLayout 既实现了固定的选项卡 - view的宽度平均分配，也实现了可滚动的选项卡 - view宽度不固定同时可以横向滚动。

```
TabLayout tabLayout = (TabLayout) findViewById(R.id.tabs);
tabLayout.addTab(tabLayout.newTab().setText("tab1"));
tabLayout.addTab(tabLayout.newTab().setText("tab2"));
tabLayout.addTab(tabLayout.newTab().setText("tab3"));

```

但是我们一般不会这么用，使用Tab都是配合着ViewPager来的，所以我们需要设置ViewPager

```
mViewPager = (ViewPager) findViewById(R.id.viewpager);
TabLayout tabLayout = (TabLayout) findViewById(R.id.tabs);
tabLayout.setupWithViewPager(mViewPager);
```

使用setupWithViewPager就把TabLayout和ViewPager联系在一起了。

###NavigationView

NavigationView在MD设计中非常重要，之前Google也提出了使用DrawerLayout来实现导航抽屉。这次，在support library中，Google提供了NavigationView来实现导航菜单界面，

```
<android.support.v4.widget.DrawerLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/dl_main_drawer"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true">

    <android.support.design.widget.NavigationView
        android:id="@+id/nv_main_navigation"
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:layout_gravity="start"
        app:headerLayout="@layout/navigation_header"
        app:menu="@menu/drawer_view"/>

</android.support.v4.widget.DrawerLayout>
```

有两个属性非常重要**`app:headerLayout`和`app:menu`**，通过这两个属性，我们可以很方便的指定导航页面的头部布局和菜单布局

可以通过设置一个OnNavigationItemSelectedListener，使用其**setNavigationItemSelectedListener()**来获得元素被选中的回调事件。它为你提供被点击的 菜单元素 ，让你可以处理选择事件，改变复选框状态，加载新内容，关闭导航菜单

```
navigationView.setNavigationItemSelectedListener( new NavigationView.OnNavigationItemSelectedListener() {
    @Override 
    public boolean onNavigationItemSelected(MenuItem menuItem) { 
        menuItem.setChecked(true);
        mDrawerLayout.closeDrawers(); 
        return true; 
    }
 });
```
###AppBarLayout

AppBarLayout跟它的名字一样，把容器类的组件全部作为AppBar

```
<android.support.design.widget.AppBarLayout
android:id="@+id/appbar"
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"> 

    <android.support.v7.widget.Toolbar
        android:id="@+id/toolbar"
        android:layout_width="match_parent"
        android:layout_height="?attr/actionBarSize"
        android:background="?attr/colorPrimary"/>

    <android.support.design.widget.TabLayout
        android:id="@+id/tabs"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>

</android.support.design.widget.AppBarLayout>

```

这里就是把Toolbar和TabLayout放到了APPBarLayout里，让他们当做一个整体来做AppBar

###CoordinatorLayout

CoordinatorLayout是一个增强型的FrameLayout，它可以说是这次Design support library更新的重中之重。它从另一个层面去控制子View触摸事件的响应。

1. 给这个可滚动组件设置了layout_behavior 
2. 给另一个控件设置了layout_scrollFlags那么，当设置了layout_behavior的控件滑动时，就会触发设置了layout_scrollFlags的控件发生状态的改变。

设置的layout_scrollFlags有以下几种选项：

* scroll：所有想滚动出屏幕的view都需要设置这个flag- 没有设置这个flag的view将被固定在屏幕顶部。
* enterAlways：个flag让任意向下的滚动都会导致该view变为可见，启用快速“返回模式”。
* enterAlwaysCollapsed: 当你的视图已经设置minHeight属性又使用此标志时，你的视图只能已最小高度进入，只有当滚动视图到达顶部时才扩大到完整高度。
* exitUntilCollapsed：同样顾名思义，这个flag时定义何时退出，当你定义了一个minHeight，这个view将在滚动到达这个最小高度的时候消失。

注意：**所有使用scroll flag的view都必须定义在没有使用scroll flag的view的前面，这样才能确保所有的view从顶部退出，留下固定的元素。**

CoordinatorLayout协调布局，其实它的核心点是在**Behavior**。我们可以自定义Behavior，来实现View的联动。

View的联动动画有很多种实现方式，但是这种使用CoordinatorLayout，自定义Behavior的方式减少了联动View的依赖，核心代码都在Behavior中

```
public class ColumnToolBarBehavior extends CoordinatorLayout.Behavior<NavBarView>

```

###CollapsingToolbarLayout

CollapsingToolbarLayout提供了一个可以折叠的Toolbar，`app:layout_collapseMode=”pin”`来确保Toolbar在view折叠的时候仍然被固定在屏幕的顶部。除了固定住view，你还可以使用`app:layout_collapseMode=”parallax”`以及使用`app:layout_collapseParallaxMultiplier=”0.7”`来设置视差因子,来实现视差滚动效果（比如CollapsingToolbarLayout里面的一个ImageView），这中情况和CollapsingToolbarLayout的app:contentScrim=”?attr/colorPrimary”属性一起配合更完美。

还要设置`app:layout_scrollFlags="scroll|exitUntilCollapsed"`，同时还要给底部的滑动view设置`app:layout_behavior="@string/appbar_scrolling_view_behavior">`

###总结
Design support library确实非常好用，之前实现起来非常复杂的效果，使用这里的控件都能轻易实现，不用**重复造轮子**。当然，使用是一方面，了解内部实现机制确实是非常必要的。