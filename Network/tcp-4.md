# TCP 那些事（四）拥塞处理
## 拥塞处理
上面我们知道了，TCP通过 Sliding Window 来做流控（Flow Control），但是 TCP 觉得这还不够，因为 Sliding Window 需要依赖于连接的发送端和接收端，其并不知道网络中间发生了什么。TCP 的设计者觉得，一个伟大而牛逼的协议仅仅做到流控并不够，因为流控只是网络模型 4 层以上的事，TCP的还应该更聪明地知道整个网络上的事。

具体一点，我们知道TCP通过一个 timer 采样了 RTT 并计算 RTO，但是，如果网络上的延时突然增加，那么，TCP 对这个事做出的应对只有重传数据，但是，重传会导致网络的负担更重，于是会导致更大的延迟以及更多的丢包，于是，这个情况就会进入恶性循环被不断地放大。试想一下，如果一个网络内有成千上万的 TCP 连接都这么行事，那么马上就会形成“网络风暴”，TCP 这个协议就会拖垮整个网络。这是一个灾难。

所以，TCP 不能忽略网络上发生的事情，而无脑地一个劲地重发数据，对网络造成更大的伤害。对此 TCP 的设计理念是：TCP 不是一个自私的协议，当拥塞发生的时候，要做自我牺牲。就像交通阻塞一样，每个车都应该把路让出来，而不要再去抢路了。

关于拥塞控制的论文请参看《Congestion Avoidance and Control》(PDF)

拥塞控制主要是四个算法：1）慢启动，2）拥塞避免，3）拥塞发生，4）快速恢复。这四个算法不是一天都搞出来的，这个四算法的发展经历了很多时间，到今天都还在优化中。 备注:
* 1988年，TCP-Tahoe 提出了1）慢启动，2）拥塞避免，3）拥塞发生时的快速重传
* 1990年，TCP Reno 在Tahoe的基础上增加了4）快速恢复

### 慢热启动算法 – Slow Start
首先，我们来看一下 TCP 的慢热启动。慢启动的意思是，刚刚加入网络的连接，一点一点地提速，不要一上来就像那些特权车一样霸道地把路占满。新同学上高速还是要慢一点，不要把已经在高速上的秩序给搞乱了。

慢启动的算法如下(cwnd 全称 Congestion Window)：
1. 连接建好的开始先初始化 cwnd = 1，表明可以传一个 MSS 大小的数据。
2. 每当收到一个 ACK，cwnd++; 呈线性上升
3. 每当过了一个 RTT，cwnd = cwnd*2; 呈指数上升
4. 还有一个 ssthresh（slow start threshold），是一个上限，当 cwnd >= ssthresh 时，就会进入“拥塞避免算法”（后面会说这个算法）

所以，我们可以看到，如果网速很快的话，ACK也会返回得快，RTT也会短，那么，这个慢启动就一点也不慢。下图说明了这个过程。

![慢启动](tcp_slow_start.png)

这里，我需要提一下的是一篇 Google 的论文《An Argument for Increasing TCP’s Initial Congestion Window》。Linux 3.0 后采用了这篇论文的建议——把 cwnd 初始化成了 10个MSS。 而Linux 3.0以前，比如 2.6，Linux 采用了RFC3390，cwnd 是跟 MSS 的值来变的，如果 MSS< 1095，则 cwnd = 4；如果 MSS>2190，则 cwnd=2；其它情况下，则是3。

### 拥塞避免算法 – Congestion Avoidance
前面说过，还有一个 ssthresh（slow start threshold），是一个上限，当 cwnd >= ssthresh 时，就会进入“拥塞避免算法”。一般来说 ssthresh 的值是65535，单位是字节，当 cwnd 达到这个值时后，算法如下：
1. 收到一个 ACK 时，cwnd = cwnd + 1/cwnd
2. 当每过一个 RTT 时，cwnd = cwnd + 1
这样就可以避免增长过快导致网络拥塞，慢慢的增加调整到网络的最佳值。很明显，是一个线性上升的算法。

### 拥塞状态时的算法
前面我们说过，当丢包的时候，会有两种情况：
* 等到 RTO 超时，重传数据包。TCP认为这种情况太糟糕，反应也很强烈。
    * sshthresh =  cwnd /2
    * cwnd 重置为 1
    * 进入慢启动过程
* Fast Retransmit 算法，也就是在收到 3 个 duplicate ACK 时就开启重传，而不用等到 RTO 超时。
    * TCP Tahoe 的实现和 RTO 超时一样。
    * TCP Reno 的实现是：
        * cwnd = cwnd /2
        * sshthresh = cwnd
        * 进入快速恢复算法——Fast Recovery
上面我们可以看到 RTO 超时后，sshthresh 会变成 cwnd 的一半，这意味着，如果 cwnd <= sshthresh 时出现的丢包，那么 TCP 的 sshthresh 就会减了一半，然后等 cwnd 又很快地以指数级增涨爬到这个地方时，就会成慢慢的线性增涨。我们可以看到，TCP 是怎么通过这种强烈地震荡快速而小心得找到网站流量的平衡点的。

### 快速恢复算法 – Fast Recovery

#### TCP Reno
这个算法定义在 RFC5681。快速重传和快速恢复算法一般同时使用。快速恢复算法是认为，你还有 3 个 Duplicated Acks 说明网络也不那么糟糕，所以没有必要像 RTO 超时那么强烈。 注意，正如前面所说，进入 Fast Recovery 之前，cwnd 和 sshthresh已被更新：
* cwnd = cwnd /2
* sshthresh = cwnd

然后，真正的 Fast Recovery 算法如下：
* cwnd = sshthresh  + 3 * MSS （3的意思是确认有3个数据包被收到了）
* 重传 Duplicated ACKs 指定的数据包
* 如果再收到 duplicated Acks，那么 cwnd = cwnd +1
* 如果收到了新的 Ack，那么，cwnd = sshthresh ，然后就进入了拥塞避免的算法了。

如果你仔细思考一下上面的这个算法，你就会知道，上面这个算法也有问题，那就是——它依赖于3个重复的Acks。注意，3个重复的Acks并不代表只丢了一个数据包，很有可能是丢了好多包。但这个算法只会重传一个，而剩下的那些包只能等到 RTO 超时，于是，进入了恶梦模式——超时一个窗口就减半一下，多个超时会超成 TCP 的传输速度呈级数下降，而且也不会触发 Fast Recovery 算法了。

通常来说，正如我们前面所说的，SACK 或 D-SACK 的方法可以让 Fast Recovery 或 Sender 在做决定时更聪明一些，但是并不是所有的 TCP 的实现都支持 SACK（SACK 需要两端都支持），所以，需要一个没有 SACK 的解决方案。而通过 SACK 进行拥塞控制的算法是 FACK（后面会讲）

#### TCP New Reno
于是，1995年，TCP New Reno（参见 RFC 6582 ）算法提出来，主要就是在没有 SACK 的支持下改进 Fast Recovery 算法的
* 当 sender 这边收到了 3 个 Duplicated Acks，进入 Fast Retransimit 模式，开发重传重复 Acks 指示的那个包。如果只有这一个包丢了，那么，重传这个包后回来的Ack会把整个已经被 sender 传输出去的数据 ack 回来。如果没有的话，说明有多个包丢了。我们叫这个 ACK 为 Partial ACK。
* 一旦 Sender 这边发现了 Partial ACK 出现，那么，sender 就可以推理出来有多个包被丢了，于是乎继续重传 sliding window 里未被ack的第一个包。直到再也收不到了 Partial Ack，才真正结束 Fast Recovery 这个过程

我们可以看到，这个“Fast Recovery的变更”是一个非常激进的玩法，他同时延长了Fast Retransmit和Fast Recovery的过程。

### 算法示意图
下面我们来看一个简单的图示以同时看一下上面的各种算法的样子：
![拥塞处理](tcp_new_reno.png)

### FACK算法

FACK 全称 Forward Acknowledgment 算法，论文地址在这里（PDF）Forward Acknowledgement: Refining TCP Congestion Control 这个算法是基于 SACK 的，前面我们说过 SACK 是使用了 TCP 扩展字段 Ack 了有哪些数据收到，哪些数据没有收到，他比 Fast Retransmit 的 3 个 duplicated acks 好处在于，前者只知道有包丢了，不知道是一个还是多个，而 SACK 可以准确的知道有哪些包丢了。 所以，SACK 可以让发送端这边在重传过程中，把那些丢掉的包重传，而不是一个一个的传，但这样的一来，如果重传的包数据比较多的话，又会导致本来就很忙的网络就更忙了。所以，FACK 用来做重传过程中的拥塞流控。

* 这个算法会把 SACK 中最大的 Sequence Number 保存在 snd.fack 这个变量中，snd.fack 的更新由 ack 带动，如果网络一切安好则和 snd.una 一样（snd.una 就是还没有收到 ack 的地方，也就是前面 sliding window 里的category #2 的第一个地方）
* 然后定义一个 awnd = snd.nxt – snd.fack（snd.nxt 指向发送端 sliding window 中正在要被发送的地方——前面 sliding windows 图示的 category#3 第一个位置），这样 awnd 的意思就是在网络上的数据。（所谓 awnd 意为：actual quantity of data outstanding in the network）
* 如果需要重传数据，那么，awnd = snd.nxt – snd.fack + retran_data，也就是说，awnd 是传出去的数据 + 重传的数据。
* 然后触发 Fast Recovery 的条件是： ( ( snd.fack – snd.una ) > (3*MSS) ) || (dupacks == 3) ) 。这样一来，就不需要等到 3 个 duplicated acks 才重传，而是只要 sack 中的最大的一个数据和 ack 的数据比较长了（3个MSS），那就触发重传。在整个重传过程中 cwnd 不变。直到当第一次丢包的 snd.nxt<=snd.una（也就是重传的数据都被确认了），然后进来拥塞避免机制——cwnd 线性上涨。

我们可以看到如果没有 FACK 在，那么在丢包比较多的情况下，原来保守的算法会低估了需要使用的 window 的大小，而需要几个 RTT 的时间才会完成恢复，而 FACK 会比较激进地来干这事。 但是，FACK 如果在一个网络包会被 reordering 的网络里会有很大的问题。

### 其它拥塞控制算法简介

#### TCP Vegas 拥塞控制算法

这个算法1994年被提出，它主要对 TCP Reno 做了些修改。这个算法通过对 RTT 的非常重的监控来计算一个基准 RTT。然后通过这个基准RTT来估计当前的网络实际带宽，如果实际带宽比我们的期望的带宽要小或是要多的活，那么就开始线性地减少或增加 cwnd 的大小。如果这个计算出来的 RTT 大于了 Timeout 后，那么，不等 ack 超时就直接重传。（Vegas 的核心思想是用 RTT 的值来影响拥塞窗口，而不是通过丢包） 这个算法的论文是《TCP Vegas: End to End Congestion Avoidance on a Global Internet》这篇论文给了 Vegas 和 New Reno 的对比：
![Vegas对比](tcp_vegas.png)
关于这个算法实现，你可以参看Linux源码：/net/ipv4/tcp_vegas.h， /net/ipv4/tcp_vegas.c

#### HSTCP(High Speed TCP) 算法

这个算法来自 RFC 3649（Wikipedia词条）。其对最基础的算法进行了更改，他使得 Congestion Window 涨得快，减得慢。其中：

* 拥塞避免时的窗口增长方式： cwnd = cwnd + α(cwnd) / cwnd 
* 丢包后窗口下降方式：cwnd = (1- β(cwnd))*cwnd

注：α(cwnd) 和 β(cwnd) 都是函数，如果你要让他们和标准的 TCP 一样，那么让 α(cwnd)=1，β(cwnd)=0.5 就可以了。 对于 α(cwnd) 和 β(cwnd) 的值是个动态的变换的东西。 关于这个算法的实现，你可以参看 Linux 源码：/net/ipv4/tcp_highspeed.c


#### TCP BIC 算法
2004 年，产内出 BIC 算法。现在你还可以查得到相关的新闻《Google：美科学家研发BIC-TCP协议 速度是DSL六千倍》。 BIC全称 Binary Increase Congestion control，在 Linux 2.6.8 中是默认拥塞控制算法。BIC 的发明者发这么多的拥塞控制算法都在努力找一个合适的 cwnd – Congestion Window，而且 BIC-TCP 的提出者们看穿了事情的本质，其实这就是一个搜索的过程，所以 BIC 这个算法主要用的是 Binary Search——二分查找来干这个事。 关于这个算法实现，你可以参看 Linux 源码：/net/ipv4/tcp_bic.c


#### TCP WestWood算法
westwood 采用和 Reno 相同的慢启动算法、拥塞避免算法。westwood 的主要改进方面：在发送端做带宽估计，当探测到丢包时，根据带宽值来设置拥塞窗口、慢启动阈值。 那么，这个算法是怎么测量带宽的？每个 RTT 时间，会测量一次带宽，测量带宽的公式很简单，就是这段 RTT 内成功被 ack 了多少字节。因为，这个带宽和用 RTT 计算 RTO 一样，也是需要从每个样本来平滑到一个值的——也是用一个加权移平均的公式。 另外，我们知道，如果一个网络的带宽是每秒可以发送X个字节，而 RTT 是一个数据发出去后确认需要的时候，所以，X * RTT应该是我们缓冲区大小。所以，在这个算法中，ssthresh 的值就是 est_BD * min-RTT(最小的RTT值)，如果丢包是 Duplicated ACKs 引起的，那么如果 cwnd > ssthresh，则 cwin = ssthresh。如果是 RTO 引起的，cwnd = 1，进入慢启动。关于这个算法实现，你可以参看 Linux 源码： /net/ipv4/tcp_westwood.c

#### 其它
更多的算法，你可以从Wikipedia的 TCP Congestion Avoidance Algorithm 词条中找到相关的线索

## Refer
整理自：
* [TCP 的那些事儿（上）](https://coolshell.cn/articles/11564.html)
* [TCP 的那些事儿（下）](https://coolshell.cn/articles/11609.html)