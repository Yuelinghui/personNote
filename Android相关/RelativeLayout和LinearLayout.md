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
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0NTg2OTY4OTJdfQ==
-->