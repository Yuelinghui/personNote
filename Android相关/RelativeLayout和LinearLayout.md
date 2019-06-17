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

** RelativeLayout

Measure：2.280ms  
Layout：0.153ms  
draw：7.696ms
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE3MzI0OTI5MzldfQ==
-->