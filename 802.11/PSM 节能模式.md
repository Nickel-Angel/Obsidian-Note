802.11 协议中初始的节能模式，其对基础架构模式和 IBSS 模式下的节能机制分别进行了定义，并且在 DCF 和 PCF 模式下，其具体的 MAC 层工作机制也有不同。
节能模式的基本思想是：AP 周期性向对应的节点广播其缓存区情况，从而节点可以知道自己是否被数据缓存了。在休眠结束后，被缓存数据的节点就会进行数据请求，反之就继续休眠。
所以需要搞清楚两个问题：
1. AP 如何广播自己的缓存区信息（即 AID，TIM 与 Bitmap 机制）
2. AP 什么时候广播对应节点的缓存区信息（即 TSF，TBTT，Listen Interval field 与 CFP repetition interval）
## AID，TIM 与 Bitmap
首先 AP 会周期性广播一个长度为 251 字节的 bitmap，来通知各个 STA 有没有数据缓存，STA 会通过自己对应位置上的 bit 是否为 1 来判断是否有数据缓存，然后再决定是否要发送 PS-Poll 帧请求数据。
在此 bitmap 上每个 STA 的数据缓存状态代表的是哪一位，就是 STA 的 AID（Association IDentifier），而 AID 是在 STA 向 AP 发起连接请求的时候，AP 回复连接应答时被分配的，取值范围位 1~2007。但 AID 没有过期时间，AP 不会主动回收。
TIM（Traffic Indication Map）：流量指示图，实际上是一个基于 bitmap 结构的流量指示图，用以标识 AP 的缓存信息。其具体结构如下：
- **Element ID**：元素识别码。
- **Length**：长度。
- **DTIM Count，DTIM Period**：DTIM 计数以及间隔的时间。
在 802.11 协议中，我们可以看到三个概念，TIM，DTIM，ATIM。
TIM 是一种基本的流量指示图的结构，标准的 TIM 中仅仅指示 AP 缓存的单播信息。
DTIM（Delivery Traffic Indication Map）是一种特殊的 TIM，其除了缓存的单播信息，也同时指示 AP 缓存的组播信息。
一般情况下，每一个 beacon 帧中都包含一个 TIM 信息，不过该 TIM 具体是不是 DTIM，则需要考量 DTIM Count 和 DTIM Period 两个参数。
DTIM Period 是一个周期，是一个固定值，代表经过几个 TIM 之后就会出现一个 DTIM。
DTIM count 是一个计数值，是变化的。当 DTIM count=0 时，则代表这个 TIM 是一个DTIM。实际上如果 DTIM Period 设置成 1，那么每一个 TIM 字段中，DTIM count 都等于 0，所以每一个 TIM 就是 DTIM 了。
ATIM是一个帧，在IBSS模式下被使用，由于本文主要讨论的就是基础架构模式下的无线网络，所以这里就不展开了。
- **Bitmap Control，Partial Virtual Bitmap**：该字段就是 bitmap 的具体字段，实际上与我们一开始描述的 bitmap 结构还存在一些区别。
由于完整的 bitmap 不仅是 251 字节，而且还是广播帧，显然是不现实的。而且由于 AID 不会被主动回收，所以一旦前面的 STA 都不再活跃，就中间的在活跃，就会导致空间浪费，所以需要进行省略。
下图就是 TIM 的信息元素，其中 Bitmap Control 的最高位是用于指示是否有组播/广播帧被缓存的，然后剩下的位为 Bitmap offset，标识了 Partial Virtual Bitmap 的开始位置。
而 Partial Vitrual Bitmap 字段即为上文 bitmap 的一部分，它的长度是可变的。
![[TIM_Element.png]]
