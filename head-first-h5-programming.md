#Head First H5 Programming



##认识HTML5

`<!doctype html>`

这不仅仅是HTML5的doctype，也是HTML将来所有版本的doctype。doctype不会再变了，不仅如此，它甚至在老版本的浏览器中也能正常工作。

`<meta charset="utf-8">`

新的meta标记简单了。用H5指定meta标记时，只需随标记提供一个字符编码。

`<link rel="stylesheet" href="XXX.css">`

需要把type属性去掉。因为已经宣布CSS作为Ht的标准样式。

`<script src="XXX.js"></script>`

`<script>
    var abc = true;
</script>`
对于H5，JavaScript现在已经成为标准，同时也是默认的脚本语言，所有同样可以从script标记中去除type属性。

H5是由一个技术“家族”构成的。HTML标记本身已经扩展为包括有一些新的元素；另外通过CSS3对CSS也有大量补充，可以大大增强指定页面样式的能力。此外还有一个高能“充电器”：JavaScript，有一组全新的JavaScript API可供你使用。

* 浏览器加载一个文档，其中包括HTML和CSS
* 浏览器加载页面时，还会为你的文档创建一个内部模型。对与HTML中的每个元素，浏览器会创建一个表示该元素的对象，把它与所有其他元素放在一个类似树的结构中
* 这个树成为文档对象模型，或者简称为**DOM**
* 浏览器加载页面时，还会加载JavaScript代码，页面加载之后就开始执行这些代码。JavaScript通过DOM与页面交互
* 通过JavaScript这些API，可以访问音频和视频等等

H5的底层设计原则之一就是允许你的页面妥善地降级，若用户的浏览器未能提供一个新特性，就应当提供有一个有意义的候选功能。

包围JavaScript代码的`<script>`和`</script>`标记，告诉页面它们之间包围的内容是JavaScript，而不是HTML。

其实H5就是：**标记 + JavaScript API + CSS**

##介绍JavaScript和DOM

JavaScript代码可以对用户的动作做出相应，更新或修改页面，与Web服务通信，总的来说，可以让页面感觉更像是一个应用而不只是一个文档。

###创建变量

```
var scoops = 10;
```
**变量是用来存放值的容器。JavaScript变量没有严格的类型，所有任何变量都可以存放数字，串或布尔值**。

一旦创建了一个变量，可以在任何时刻改变它的值，甚至可以改为一个不同类型的值。
```
var scroops = 10;
scroops = 4;
scroops = "Hello World";
scroops = true;
```

若不给变量初始化的值，变量会赋值为**undefined**，这是JavaScript的一个特殊值和类型。

###命名变量

* 要以一个字母，下划线或者美元符（$）开头
* 然后可以使用任意多个字母，数字，下划线或美元符
* 要避开JavaScript的所有保留字
* JavaScript是区分大小写的，myvariable和MyVariable是两个不同的变量

###放在哪

* 放在`<head>`元素中。在页面的head部分放置一个`<script>`元素，一旦浏览器开始解析head部分就会执行这个代码，然后才解析页面的其余部分
* 通过引用一个单独是JavaScript文件夹来增加脚本。文件的URL放在script标记的**src**属性中
* 可以把代码直接放在HTML的体中。

###如何与页面交互

JavaScript与页面中的标记交互使用的是文档对象模型（DOM）。

通过DOM，JavaScript就能与页面通信，反之亦然。这就像一个小通道，允许JavaScript访问任何元素，这个方法为**getElementById**。

使用innerHTML属性改变元素的内容。

告诉浏览器执行代码之前需要等待，需要用到两部分JavaScript：一个是**window**对象，另外还有一个函数。

浏览器运行访问元素的代码之前先要等待DOM完全加载。

DOM能做什么：
* 从DOM得到元素
* 想DOM创建或增加元素
* 从DOM删除元素
* 获取和设值元素的属性

###创建数组

```
var array = new Array();
array[0] = 1;
array[1] = 2;
array[3] = 3;
```
或者
```
var array = [1,2,3,4];
```

添加数组的新元素的时候，只需使用下一个魏永的索引就可以。获取数组的大小使用属性**length**。

**undefined和null是两个不同的值。undefined表示一个变量未赋值；null表示这个变量有一个空值**。

##一点点交互

按钮点击事件：**onclick**。

创建新元素：`document.createElement("元素名称");`

向DOM增加新元素：
```
var father = document.getElementById("id");
father.appendChild("子元素");
```

要得到用户在一个表单输入本文域中输入的文本，要使用这个输入域的value属性。

若用户没有向表单输入文本域输入任何内容，这个域的值将是空串（“”）。

使用appendChild向一个父元素增加多个子元素，每个新的子元素会追加到其他子元素的后面。

可以使用Web存储API（localStorage）在用户的浏览器中存储数据。


##正式JavaScript

可以返回一个值作为调用函数的结果，这是可选的。

JavaScript传递参数时：
* 基本数据类型是**值传递**，即若在函数体中改变了参数的值，对原来的数没有任何影响
* 传递数组或对象是**引用传递**，即在函数体中改变了参数值，原来的值也会跟着改变

若一个函数没有return函数，返回的是**undefined**。

变量的生命周期
* 只要页面存在，全局变量就活着。重新加载同样的页面，全局变量会重新创建
* 局部变量通常在函数结束时就消失

只有一个全局作用域，加载的各个文件会看到同样的一组变量，并在相同的空间创建全局变量。所有**一定要当心变量的使用，以避免冲突**。若不同文件有两个同名的函数，将使用浏览器最后看到的那个函数。

函数也是值，也可以把一个函数赋值给变量。甚至可以不指定函数名。
```
var f = function(num) {
    return num + 1;
}
var result = f(1);
```
最后result = 2!(有点像Java里的匿名内部类。。。)

###用JavaScript创建对象

```
var fido = {
    name:"Fido";
    weight:40;
    breed:"Mixed"
    loves:["walks","balls","play"]
}
```

访问对象属性：
* 用“点”来访问对象属性`fido.name`
* 使用类似访问数组的方法访问属性`fido["name"]`

枚举对象的所有属性使用for-in循环
```
var prop;
for(prop in fido) {
}
```

任何时刻都可以增加或删除属性。要向一个对象增加属性，只需为一个新属性赋一个值:
```
fido.age = 5;
```

可以用delete关键字删除任何属性:`delete fido.age;`

若属性成功删除，delete表达式会返回true。

