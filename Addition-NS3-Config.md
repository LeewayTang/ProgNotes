如果试图在ns3下模拟一些网络条件，一般关注下面的要素：
- 信道延迟
- 信道丢包
- 设备流量控制

# 信道延迟

直接给Channel设置一个TimeValue作为Delay属性即可。

# 信道丢包

需要给信道设置一个丢包模型，一般使用的是`RateErrorModel`，即根据设置的概率进行随机丢包。

# 设备流量控制

## 数据率限制

直接给设备设置一个发送数据率的上限，即给DataRate属性设置一个DataRateValue。

## 发送队列控制

继承自`QueueDisc`抽象类，在数据包传输行为中决定了对其的实际作用。
- FifoQueueDisc 先入先出，无特殊处理。
- PrioQueueDisc 有多个彼此之间有优先级关系的子队列，每个子队列都可以是任意的队列类型。
- TBFQueueDisc 令牌桶过滤器队列，利用令牌桶来控制带宽和突发流量。
- RedQueueDisc 随机早期检测队列，它使用一个平均队列长度来判断网络拥塞的程度，并根据一个概率函数来随机丢弃数据包。它可以有效地避免全局同步和尾部丢弃等问题。
- CoDelQueueDisc 它是一个控制延迟（CoDel）队列控制，它使用一个目标延迟和一个间隔时间来判断网络拥塞的程度，并根据一个适应性算法来动态调整数据包的丢弃阈值。它可以有效地降低网络延迟和缓冲区膨胀等问题。