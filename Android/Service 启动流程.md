![[ActivityManagerService.png]]
## 流程图
在 app 中启动一个 service，只需调用 `startService()`，主要是通过 `ActivityManagerService` 来完成的。

![[StartService.png]]

1. `ActivityManagerService` 通过 Socket 向 `Zygote` 请求调用 `fork` 创建子进程 `ActivityThread`；
2. `Zygote` 通过 `fork` 创建 `ActivityThread`；
3. `ActivityManagerService` 通过 `Binder` 来和 `ActivityThread` 通信；