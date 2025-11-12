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

当 [Service 启动流程](obsidian://open?vault=Obsidian%20Vault&file=Android%2FService%20%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B) 中，其中调用 `realStartServiceLocked` 中，会设置一个超时的提示。

```java
void scheduleServiceTimeoutLocked(ProcessRecord proc) { 
	if (proc.executingServices.size() == 0 || proc.thread == null) {
		return; 
	} 
	long now = SystemClock.uptimeMillis(); 
	Message msg = mAm.mHandler.obtainMessage(
		ActivityManagerService.SERVICE_TIMEOUT_MSG); 
	msg.obj = proc; // 当超时后仍没有 remove 该 SERVICE_TIMEOUT_MSG 消息，则执行 service Timeout 流程 
	mAm.mHandler.sendMessageAtTime(msg, proc.execServicesFg ? 
		(now + SERVICE_TIMEOUT) : (now + SERVICE_BACKGROUND_TIMEOUT)); 
}
```

该方法通过设置一个延迟发送的消息来防止超时。
## 拆炸弹

在 `ActivityThread` 响应创建服务时，会调用 `Service` 的 `onCreate()` 方法，在此之后，就会调用 `serviceDoneExecutingLocked`，进行炸弹引线的拆除工作。

```java
private void serviceDoneExecutingLocked(ServiceRecord r, 
	boolean inDestroying, boolean finishing) { 
	// ... 
	if (r.executeNesting <= 0) { 
		if (r.app != null) { 
			r.app.execServicesFg = false; 
			r.app.executingServices.remove(r); 
			if (r.app.executingServices.size() == 0) { 
				// 当前服务所在进程中没有正在执行的 service 
				mAm.mHandler.removeMessages(
				ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
			} 
		// ... 
		} 
	// ... 
}
```

这里移除了延迟发送的超时消息。

## 引爆炸弹

如果在指定时间内，未完成启动，那么这条超时消息就会发送到 `ActivityManager` 中，则其收到消息会进入 `handleMessage` 函数：

```java
final class MainHandler extends Handler { 
	public void handleMessage(Message msg) { 
		switch (msg.what) { 
			case SERVICE_TIMEOUT_MSG:
				// ...
				mServices.serviceTimeout((ProcessRecord)msg.obj); 
				break; 
			// ... 
		} 
	// ... 
	} 
}
```

然后 `serviceTimeout` 会输出超时信息，之后调用 `appNotResponding`。

# BroadcastReceiver

