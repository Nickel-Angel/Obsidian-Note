 ![[ActivityManagerService.png]]
## 流程图
在 app 中启动一个 service，只需调用 `startService()`，主要是通过 `ActivityManagerService` 来完成的。

![[Android/img/StartService.png]]

1. `ActivityManagerService` 通过 Socket 向 `Zygote` 请求调用 `fork` 创建子进程 `ActivityThread`；
2. `Zygote` 通过 `fork` 创建 `ActivityThread`；
3. `ActivityManagerService` 通过 `Binder` 来和 `ActivityThread` 通信；
4. `ActivityThread` 启动运行进程。

![[StartService whole.png]]
更具体的流程在这里。这里当 `ActivityManager` 打开服务的时候，如果检测到了对应的进程已存在，则会直接调用走 5，即调用 `realStartServiceLocked`。
其中这里的进程间通信是通过 `Binder` 来通信的，而对于 7 则是通过 `Handler` 的发送机制来通信的。

