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

```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTkwMDI2MjQwXX0=
-->