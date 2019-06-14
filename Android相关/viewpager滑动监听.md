# ViewPager的滑动监听

ViewPager现在使用的非常普遍，可以和TabLayout组合起来实现滑动切换页面的效果。

ViewPager提供了滑动监听
```
public void setOnPageChangeListener(OnPageChangeListener onPageChangeListener)
```
这个Listener有三个回调方法

```
public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels)

public void onPageSelected(int position) 

public void onPageScrollStateChanged(int state)
```

这次主要想记录的是第一个方法（onPageScrolled）里的三个参数

* 当我们向右滑动时，position先是当前页面的值，当我们滑动超过一半时，才回返回下一个页面的值。当我们向左滑动时，position返回的是上一个页面的值，不会再像向右滑动时一样。这就是为啥我们在根据position做判断时，经常会发现当向左滑动的时候position直接就变为上一个页面了

* posotionOffset标记的是滑动的百分比。当我们向右滑动时，值从0变到1，当我们到下一个页面的时候再返回0。当我们向左滑动的时候，这个值从1变为0

* positionOffsetPixels返回的是滑动的距离（像素），向右滑动时，这个值变大，向左滑动时，这个值变小

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE5NTE0NDEyOThdfQ==
-->