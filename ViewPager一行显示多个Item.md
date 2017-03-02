#ViewPager一页显示多个item

##需求
最近遇到了一个比较难搞的需求：

![](http://ofowf99vj.bkt.clouddn.com/viewpager.png)

一个ViewPager的一页要显示多个item，当前的item显示在中间，两边还能看到上一个和下一个item，在网上搜了一下，找到了合适的解决方案

##解决方案

在当前界面布局中添加一个LinearLayout，内部加上ViewPager

```
<LinearLayout
        android:id="@+id/layout_viewpager_container"
        android:layout_width="match_parent"
        android:layout_height="80dp"
        android:clipChildren="false"
        android:gravity="center"
        android:layout_below="@+id/txt_author_desc"
        android:visibility="gone"
        android:orientation="horizontal">

        <android.support.v4.view.ViewPager
            android:id="@+id/horizon_pager"
            android:layout_width="match_parent"
            android:layout_height="80dp"
            android:layout_marginLeft="18dp"
            android:layout_marginRight="18dp"
            android:clipChildren="false" />
</LinearLayout>
```

我们注意父布局和ViewPager都设置了一个属性**android:clipChildren="false"**，这个属性的意思就是子View不必拘束在父控件的内部，当子View的大小超过父控件时，仍然可以显示。这个属性默认是_true_！

但是我们也注意到，ViewPager的高度设置是确定的数值，这个就是我们必须要做的，**必须要给子View设置死高度，不能是wrap_content**，不然的话这个ViewPager就不显示了。

我们还设置了ViewPager的左右margin，这个按照视觉稿来自己定义就可以了

在代码里我们设置：

```
viewPager.setOffscreenPageLimit(3);
viewPager.setPageMargin(10);
linearLayout.setOnTouchListener(new View.OnTouchListener() {
    @Override
    public boolean onTouch(View v, MotionEvent event) {
        return viewPager.dispatchTouchEvent(event);
    }
});
```

首先设置ViewPager加载页码，预加载3项。然后设置两个item之间的距离，就是前面图中两个item之间的距离。最后，我们要把父布局的点击事件交由ViewPager来处理，因为如果不加这个的话，那么ViewPager的滑动就只有点击item才可以，中间的空白的地方就不会响应了。

这样就能实现最开始的需求了。这个方案比较麻烦的地方是ViewPager或者item的高度要确定。