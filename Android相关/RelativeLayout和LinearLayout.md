# RelativeLayout和LinearLayout性能分析

当RelativeLayout和LinearLayout分别作为ViewGroup，表达相同布局时绘制在屏幕上时谁更快一点？

假如我们拿一个简单的布局分析一下：

```
Hello world!
Hello world!
Hello world!
Hello world!
Hello world!
Hello world!
```

要在屏幕上显示着6个TextView，分别用RelativeLayout和LinearLayout布局：

**LinearLayout**

Measure：0.738ms  
Layout：0.176ms  
draw：7.655ms

**RelativeLayout**

Measure：2.280ms  
Layout：0.153ms  
draw：7.696ms

其实两个ViewGroup的`layout`和`draw`相差不多，考虑到误差基本是不分伯仲，但是`RelativeLayout`的`measure`方法却比`LinearLayout`的`measure`方法慢了很多，为啥呢？

## measure的时候都做了什么？

### RelativeLayout的measure

```
...
int count = views.length;
for (int i = 0; i < count; i++) {
      View child = views[i];
      if (child.getVisibility() != GONE) {
	      ...
          measureChildHorizontal(child, params, myWidth, myHeight);
          ...
      }
    }
...
for (int i = 0; i < count; i++) {
      View child = views[i];
      if (child.getVisibility() != GONE) {
        LayoutParams params = (LayoutParams) child.getLayoutParams();
        measureChild(child, params, myWidth, myHeight);
        ...
      }
}
```
根据源码我们发现`RelativeLayout`会对子View做两次`measure`。这是为什么呢？首先`RelativeLayout`中子View的排列方式是基于彼此的依赖关系，而这个依赖关系可能和布局中View的顺序并不相同，在确定每个子View的位置的时候，就需要先给所有的子View排序一下。又因为RelativeLayout允许A，B 2个子View，横向上B依赖A，纵向上A依赖B。所以需要横向纵向分别进行一次排序测量。

### LinearLayout的measure

```
 @Override
 protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    if (mOrientation == VERTICAL) {
      measureVertical(widthMeasureSpec, heightMeasureSpec);
    } else {
      measureHorizontal(widthMeasureSpec, heightMeasureSpec);
    }
  }
```

与RelativeLayout相比LinearLayout的measure就简单明了的多了，先判断线性规则，然后执行对应方向上的测量。

**当zi'shu暂时先不测量这些子视图，因为后面将把父视图剩余的高度按照weight值的大小平均分配给相应的子视图。**源码中使用了一个局部变量totalWeight累计所有子视图的weight值。处理lp.weight>0的情况需要注意，如果变量heightMode是EXACTLY，那么，当其他子视图占满父视图的高度后，weight>0的子视图可能分配不到布局空间，从而不被显示，只有当heightMode是AT_MOST或者UNSPECIFIED时，weight>0的视图才能优先获得布局高度。最后我们的结论是：如果不使用weight属性，LinearLayout会在当前方向上进行一次measure的过程，如果使用weight属性，LinearLayout会避开设置过weight属性的view做第一次measure，完了再对设置过weight属性的view做第二次measure。由此可见，weight属性对性能是有影响的，而且本身有大坑，请注意避让。
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjY2Njg0NDcwXX0=
-->