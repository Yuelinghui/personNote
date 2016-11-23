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