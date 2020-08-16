# TCP 那些事
## 简介
TCP在网络 OSI 七层模型中的第四层——Transport层，IP在第三层——Network层，ARP在第二层——Data Link层，在第二层上的数据，我们叫Frame，在第三层上的数据叫Packet，第四层的数据叫Segment。

首先，我们需要知道，我们程序的数据首先会打到 TCP 的 Segment 中，然后 TCP 的 Segment 会打到 IP 的 Packet 中，然后再打到以太网 Ethernet 的 Frame 中，传到对端后，各个层解析自己的协议，然后把数据交给更高层的协议处理。

## TCP 头格式
TCP 的头格式为：
* TCP的包是没有IP地址的，那是IP层上的事。但是有源端口和目标端口。
* 一个TCP连接需要四个元组来表示是同一个连接（src_ip, src_port, dst_ip, dst_port）准确说是五元组，还有一个是协议。但因为这里只是说TCP协议，所以只说四元组。

![TCP头格式](https://github.com/AngryHacker/Rookie-Note/blob/master/creative/image/tcp_header.png)

注意上图中的四个非常重要的东西：
1. Sequence Number是包的序号，用来解决网络包乱序（reordering）问题。
2. Acknowledgement Number 就是ACK——用于确认收到，用来解决不丢包的问题。
3. Window 又叫 Advertised-Window，也就是著名的滑动窗口（Sliding Window），用于解决流控的。
4. TCP Flag ，也就是包的类型，主要是用于操控 TCP 的状态机的。

相关字段详细信息如下：

![TCP 头字段](https://github.com/AngryHacker/Rookie-Note/blob/master/creative/image/tcp_flag.png)
