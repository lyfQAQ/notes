## 调试常用命令

| 命令                             | 作用                                | 备注 |
| -------------------------------- | ----------------------------------- | ---- |
| gdb attach pid                   | 跟踪进程                            |      |
| r                                | 运行程序                            |      |
| bt 10 frame 1+info frame         | 打印堆栈信息切换到栈帧1，并查看详情 |      |
| q                                | 退出gdb                             |      |
| info b                           | 显示断点信息                        |      |
| d 1                              | 删除第一个断点                      |      |
| b hello.cpp:12                   | 设置断点                            |      |
| b if x == 10 && y == 2           | 设置条件断点                        |      |
| b myfunc                         | 根据函数名设置断点                  |      |
| info r                           | 看寄存器                            |      |
| layout src layout splitfocus src | 用界面跟踪代码分割窗口              |      |
| p myvariable                     | 查看变量                            |      |
| watch i                          | 监控变量i                           |      |
| display i                        | 持续显示变量i                       |      |
| l                                | 查看源码                            |      |
| si                               | 查看汇编                            |      |

## 可视化工具

[gdb-dashboard](https://github.com/cyrus-and/gdb-dashboard)

> gcc编译时加上-g参数，可以使可执行程序加上gdb调试信息。