##深入理解init

init是一个进程，它是Linux系统中用户控件的第一个进程。因为Android是基于Linux内核的，所以init也是Android系统中用户控件的第一个进程，它的进程号是1。init被赋予了很多及其重要的工作职责。

* init进程负责创建系统中的几个关键进程，尤其是zygote。
* Android系统有很多属性，于是init就提供了一个property setvice（属性服务）来管理它们。

###init初步分析

init进程的入口函数是main。主要的工作流程精简为：
* 即系两个配置文件。

```
// 解析init.rc配置文件
parse_config_file("/init.rc");
...
// 通过读取/proc/cpuinfo得到及其的Hardware名
get_hardware_name();
snprintf(temp,sizeof(tmp),"/init.&s.rc",hardware);
// 解析这个和机器相关的配置文件
parse_config_file(tmp);
```

* 执行各个阶段的动作，创建zygote的工作就是在其中的某个阶段完成的。解析完配置文件后，会得到一系列的Action，init将动作执行的时间划分为四个阶段：early-init,init,early-boot,boot。由于有些工作**必须在其他动作完成后才能执行，所以就有了先后之分**。

```
action_for_each_trigger("early-init",action_add_queue_tail);
...
action_for_each_trigger("init",action_add_queue_tail);
...
action_for_each_trigger("early-boot",action_add_queue_tail);
action_for_each_trigger("boot",action_add_queue_tail);
```

* 调用property_init初始化属性相关的资源，并且通过property_start_service启动属性服务

```
property_init();
...
property_set_fd = start_property_sevice();
```

* init进入一个**无限循环**，并且等待一些事情的发生。重点关注init如何处理来自socket和来自属性服务器的相关事情。

###解析配置文件

在init中会解析两个配置文件：**系统配置文件**（init.rc）和**与硬件平台相关的配置文件**（init.(hardward_name).rc）。使用的同一个函数：**parse_config_file**

```
int parse_config_file(const char *fn) 
{
    char *data;
    // 读取配置文件内容
    data = read_file(fn,0);
    if(!data) return -1;
    parse_config(fn,data);
    return 0;
}
```

读取完文件内容后，调用parse_config进行解析。parse_config先会找到配置文件的一个section，然后针对不同的section使用不同的解析函数来解析。

分析init.rc：
* 一个section的内容从这个标识section的关键字开始，到下一个标识section的地方结束。
* init.rc中出现了名为boot和init的section，这里的boot和init就是前面介绍的4个动作中的两个。也就是说，在boot阶段执行的动作都是由boot这个section定义的。
* zygote被放在了一个service section中。

###解析service

解析section的入口函数式parse_new_section，其中，解析service时，用到了parse_service和parse_line_service这两个函数。

* parse_service函数只是搭建了一个service的架子，具体的内容尚需由后面的解析函数来填充。
* parse_line_service将根据配置文件的内容填充service结构体。

zygote解析完之后：

* service_list链表将解析后的service全部链接到了一起，并且是一个双向链表。
* socketinfo也是一个双向链表，zygote只有一个socket。
* onrestart通过commands指向一个commands链表，zygote有三个commands。

###init控制service

init.rc中有`class_start default`，class_start标识一个COMMAND，对应的处理函数为do_class_start，它位于boot section的范围内。

```
// 将boot section节的command加入执行队列
action_for_each_trigger("boot",action_add_queue_tail);
// 执行队列里的命令
drain_action_queue();
```

当init进程执行到这的时候，do_class_start就会被执行了。

zygote这个service的classname的值是"default"，service会调用`service_start_if_not_disabled`，然后这里面又调用了`service_start`

`service_start`里通过fork和execv共同创建了zygote。调用fork创建子进程，

onrestart是在zygote重启时用的。

###属性服务

Windows平台有注册表，存储一些类似key/value的键值对。Android平台也提供了一个类似的机制，称之为**属性服务**，用于程序可通过这个属性机制，查询或设置属性。

init.c和属性服务有关的代码

```
property_init();
property_set_fd = start_propety_service();
```

在`property_init`中，先调用`init_property_area`函数，创建一块用于存储属性的存储区域，然后加载default.prop文件中的内容。

虽然属性区域是由init进程创建的，但Android系统希望其他进程也能读取这块内存里的东西。

