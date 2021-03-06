#Shell脚本学习指南

##入门

###脚本编程语言与编译型语言的差异

编译型语言的好处是高效，它们多半运作与底层，所处理的是字节，整数或是其他机器层级的对象

脚本编程语言通常是解释型的。是由解释器读入程序代码，并将其转换成内部的形式，再执行

###为什么要使用shell脚本

脚本语言的好处是：它们多半运行在比编译型语言还高的层级，能够轻易处理文件与目录之类的对象

* 简单性。Shell是一个高级语言，通过它，可以简洁地表达复杂的操作
* 可移植性。使用POSIX所定义的功能，可以做到脚本无须修改就可在不同的系统上执行
* 开发容易。可以在短时间内完成一个功能强大又好用的脚本

|（管道）符号可以在两程序之间建立管道。前一个的输出成为了后一个的输入。

###位于第一行的#！

通过一种方式，告知UNIX内核应该以哪Shell来执行所指定的Shell脚本。

通过脚本文件中特殊的第一行来设置：**在第一行的开头处使用_#!_这两个字符**

当文件开头的两个字符是#!时，内核会扫描该行其余的部分，看是否存在可用来执行程序的解释器的完整路径。（中间若出现任何空白符👌都会略过）

内核还会扫描是否有一个选项要传递给解释器，内核会以被指定的选项来引用解释器，再搭配命令行的其他部分

* “#！”这一行尽量不要超过64个字符
* 脚本是否具可移植性取决于是否有完整的路径名称
* 别在选项“option”之后放置任何空白，因为空白也会跟着选项一起传递给被引用的程序
* 需要知道解释器的完整路径名称。可以用来规避可移植问题
* 一些较旧的系统上，内核不具备解释#！的能力

###Shell的基本元素

####命令与参数

* 格式很简单，以空白割开命令行中各个组成部分
* 命令名称是命令行的第一个项目。通常后面会跟着选项，任何额外的参数都会放在选项之后
* 选项的开头是一个破折号（或减号），后面接着一个字母。选项是可有可无的，有可能需要加上参数。不需要参数的选项可以合并。
* 长选项的开头是一个破折号还是两个，视程序而定
* 分号可用来分割同一行里的多条命令，Shell会依次执行这些命令
* 若使用的是&而不是分号，则Shell将在后台执行其前面的命令，这意味着，Shell不用等到该命令完成，就可以继续执行下一个命令

####变量

Shell变量名称的开头是一个字母或下划线符号，后面可以接着任意长度的字母，数字或下划线符号。变量名称的字符长度并无限制。Shell变量可用来保存字符串值，所能保存的字符数同样没有限制。

先写变量名称，紧接着=字符，最后是新值，中间完全没有任何空格。

想取出Shell变量的值时，需于变量名称前面加上$字符。当所赋予的值内含空格时，加上引号

当将几个变量连接起来时，需要使用引号

####简单的echo输出

echo的任务就是产生输出，可用来提示用户，或是用来产生数据供进一步处理

####华丽的printf输出

printf命令的完整语法分为：
`printf format -string [arguments...]`

* 字符串，用来描述输出的排列方式，最好为此字符串加上引号
* 与格式声明相对应的参数列表。格式声明分为两部分：百分比符号（%）和指示符，比如%s用于字符串，%d用于十进制整数

####基本的I/O重定向

I/O重定向就是通过与终端交互，或是在Shell脚本里设置，重新安排从哪里输入或输出到哪里

**以<改变标准输入**

**以>改变标准输出。>重定向符在目的文件不存在时，会新建一个。若目的文件已存在，它就会被覆盖掉，原本的数据都会丢失**

**以>>附加到文件。>>重定向符在目的文件不存在时，会新建一个。若目的文件已存在，它不会直接覆盖掉文件，而是将程序所产生的数据附加到文件结尾处**

**以|建立管道。将前一个程序的标准输出修改为后一个程序的标准输入**

管道可以使得执行速度比使用临时文件的程序快上十倍。

特殊文件：
* /dev/null。位桶。传送到此文件的数据都会被系统丢掉。读取这个文件则会立即返回文件结束符号
* /dev/tty。当程序打开此文件时，UNIX会自动将它重定向到一个终端，再与程序结合。这在程序必须读取人工输入时特别有用

####基本命令查找

要编写自己的脚本，最好准备自己的bin目录来存放它们，并且让Shell能够自动找到它们

要让修改永久生效，在.profile文件中把自己的bin目录加入$PATH，每次登陆时Shell豆浆读取.profile文件

###访问Shell脚本的参数

位置参数指的是Shell脚本的命令行参数。在Shell函数里，它们同时也可以是函数的参数。各参数都由整数来命名

**当参数超过9，就应该用大括号把数字框起来**

Shell会忽略由#开头的每一行

###简单的执行跟踪

执行跟踪的功能打开。

Shell显示每个被执行的命令，并在前面加上“+”：一个加号后面跟着一个空格

* set -x命令将执行跟踪的功能打开
* set +x命令关闭

###国际化和本地化

一般来说，消息的译文就放在软件附带的文本文件中，再通过gencat或msgfmt编译成紧凑的二进制文件，以利快速查询

编译后的信息文件会被安装到特定的系统目录树中。

用来控制让哪种语言或文化环境生效的功能就叫做locale。

可以用LC_ALL来强制设置单一locale；而LANG则用来设置locale的默认值。

**大多数时候，应避免为任何的LC_XXX变量赋值**

##查找与替换

###查找文本

以grep程序查找文本

* grep，最早的文本匹配程序
* egrep，扩展式grep。使用扩展正则表达式
* fgrep，匹配字符串而非正则表达式，使用优化的算法，能更有效地匹配固定字符串

-F选项，以查找固定字符串。事实上，只要匹配的模式里未含有正则表达式的mete字符，则grep默认行为模式就等同于-F

###正则表达式

正则表达式是由两个基本组成部分所建立：一般字符与特殊字符。

一般字符指的是任何没有特殊意义的字符。

特殊字符常称为元字符，有时也可以视为一般字符。

* \ 用以关闭后续字符的特殊意义。有时则是打开后续字符的特殊意义
* . 匹配任何单个字符
* \* 匹配在它之前的任何数目的单个字符
* ^ 匹配紧接着的正则表达式，在行或字符串的起始处
* $ 匹配签名的正则表达式，在字符串或行结尾处
* [...] 匹配方括号内的任意字符 

####基本正则表达式

BRE是由多个组成部分所构建，一开始提供数种匹配