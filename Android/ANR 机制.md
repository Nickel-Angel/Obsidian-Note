ANR（Application Not responding）是指应用程序未响应，Android 系统对于一些事件需要在一定的时间范围内完成，如果超过预定时间能未能得到有效响应或者响应时间过长，都会造成 ANR。

造成 ANR 的四种情况： 
* Service Timeout
* BroadcastQueue Timeout
* ContentProvider Timeout
* InputDispatching Timeout

触发 ANR 的过程可分为三个步骤：埋炸弹，拆炸弹，引爆炸弹
# Service

当 `ActivityManager` 线程中 `AMS.MainHandler` 收到 `SERVICE_TIMEOUT_MSG` 消息时触发。

Service 分为前台服务和后台服务：
* 前台服务，`SERVICE_TIMEOUT` 为 20s；
* 后台服务，`SERVICE_BACKGROUND_TIMEOUT` 为 200s。

由变量 `ProcessRecord.execServicesFg` 来决定是否前台启动。
## 埋炸弹

