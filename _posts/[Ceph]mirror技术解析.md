# RBD Mirror技术分析

​        上一章节对RBD Mirror的原理和基本命令做了梳理，本章节主要说明RBD Mirror的实现。

​	先来分析“如何”启动rbd mirror

## 启动RBD Mirror

​	管理员通过如下命令启动RBD Mirror启动：

~~~
rbd mirror pool enable {pool-name} {mode}
~~~

​	首先调用函数get_enable_arguments解析参数；然后调用函数execute_enable执行启动。



