众所周知，TCP在建立连接时需要经过三次握手。许多初学者经常对这个过程感到困惑：SYN是干什么的，怎么一会儿是1一会儿是0？怎么既有大写的ACK又有小写的ack？为什么ACK在第二次握手才开始出现？初始序列号 isn 有什么讲究？isn 和 seq 有什么关系？ack 的值到底是什么？

别慌，别着急，看完这篇文章，我相信上述问题对你来说就会迎刃而解。

我将TCP三次握手所涉及到的具体操作总结为“设标志位，发序列号”。这里先告诉你一下，一般来说标志位的名称全部大写，序列号的缩写名称全部小写。

在开始讲解之前，我们先来看一下TCP段头的结构：

是不是感觉头昏眼花？这么多英文，这么多组成！其实，三次握手的过程只涉及到两个序列号（Sequence、Acknowledgement Sequence）和两个标志位（ACK、SYN）。注意我这里的大小写结构，与上图是完全对应的。下面我将分别讲解这些字段的含义。

### 序列号

序列号包括 `seq` 和 `ack`，`seq` 是发送序列号，`ack` 是确认序列号。

- **Sequence**，缩写为 `seq`，中文名为序列号，定义如下：

  > The Sequence number indicates the position in the byte stream of the first byte in the TCP Data field. For example, if the Initial Sequence number is 1,000 and this is the first segment, then the Sequence number is 1,000. If the segment is 500 bytes long, then the sequence number in the next segment will be 1,500 and so on.

  所以我们可以把 `seq` 理解为发送序列号。注意，在上述定义中提到了 Initial Sequence number（缩写 `isn`，中文名初始序列号），它其实也是 `seq`，只不过是第一次握手时用的序列号，所以叫初始序列号。发送端（客户端）和接收端（服务器端）的初始序列号都是随机生成的。初始序列号必须随机生成，这是有道理的。

### TCP 的“身份证”

为了了解 `isn` 的设计思想，我们首先需要知道 TCP 的“身份证”是什么。

> A TCP connection is uniquely identified by five pieces of information in the TCP and IP headers. The IP source and destination addresses uniquely identify the end points, and the IP Protocol ID for TCP tells us the connection is TCP. The TCP source and destination ports identify the application processes on the end hosts. Together, at any instant, all 5 fields uniquely identify the TCP connection Internet-wide.

即，一个TCP连接的ID是由源端口地址、目的端口地址、源IP地址、目的IP地址共同组成的。

### isn 随机性的重要性

假设有两台主机 A 和 B 建立了一个TCP连接，A 向 B 发送了一些数据后连接断开。假设在连接断开前，有一些报文没有正常到达 B ，而是一直处于传输网络中。之后，A 和 B 之间又建立了一个连接，且 ID 四元组与上次的连接一样。此时，如果上一个连接的报文姗姗来迟，到达了 B ，且序列号在 B 的合理区间内，那么 B 会以为这个报文是本次连接的正常报文，导致数据混乱。因此，我们需要随机的 isn。

如果你还是没看懂，不妨看看网友们的解答：

- **甲**：ISN不同是为了防止老连接的包或伪造包干扰新连接。
- **乙**：如果初始化序列号 isn 是固定的，那么可能会出现数据错乱的情况。
- **丙**：如果 TCP 建立连接时每次都选择相同的固定初始序号，滞留的 TCP 报文段可能被误认为是新连接的报文段。
- **丁**：不同的五元组连接独立，随机 isn 是为了防止攻击者猜测序列号，确保连接安全。

### 确认序列号

- **Acknowledgement Sequence**，缩写为 `ack`，中文名为确认序列号，定义如下：

  > The Acknowledgment sequence number tells the other end which byte we are expecting next. It also says that we have successfully received every byte up until the one before this byte number.

  也就是说，`ack` 的值用于通知另一端“我接下来想要的数据”，同时还告诉另一端“我已经收到了你刚刚发过来的所有数据”。

### 标志位

在TCP段头中有六种标志位（Flag），我们通常将他们的名字全部大写。这很重要，标志位的名称全部大写！三次握手中的标志位包括 `SYN` 和 `ACK`。

- **ACK** 的定义：

  > The ACK flag tells us that the Acknowledgement sequence number is valid and we are acknowledging all of the data up until this point.

  只有当 ACK 这个标志位为1的时候，ack序列号值才有效。ACK就像是ack的开关一样。

- **SYN** 的定义：

  > The SYN flag tells us that we are signalling a synchronize, which is part of the 3way handshake to set up the connection.

  SYN 是 synchronize（同步）的缩写，在第一次握手和第二次握手时，SYN=1，在第三次握手时，SYN=0。

### 三次握手示意图

#### TCP三次握手过程的文字描述

1. **第一次握手**：客户端将标志位 SYN 置为1，随机产生一个值序列号 seq=x，并将该数据包发送给服务端，客户端进入 `SYN_SENT` 状态，等待服务端确认。
2. **第二次握手**：服务端收到数据包后由标志位 SYN=1 知道客户端请求建立连接，服务端将标志位 SYN 和 ACK 都置为1，ack=x+1，随机产生一个值 seq=y，并将该数据包发送给客户端以确认连接请求，服务端进入 `SYN_RCVD` 状态。
3. **第三次握手**：客户端收到确认后检查，如果正确则将标志位 ACK 为1，ack=y+1，并将该数据包发送给服务端，服务端进行检查如果正确则连接建立成功，客户端和服务端进入 `ESTABLISHED` 状态，完成三次握手，随后客户端和服务端之间可以开始传输数据了。

怎么样，是不是特别清楚了！如果不是特别清楚的话，建议再读一遍。如果读了还是不清楚，那你就洗洗睡吧。

---

**注**：本文参考了以下资料：

- [TCP协议中的三次握手和四次挥手(图解)](https://www.cnblogs.com/dormant/p/5422124.html)
- [知乎：TCP三次握手](https://www.zhihu.com/question/49794331)
- [知乎：TCP中SYN，ACK，seq ack的含义](https://www.zhihu.com/question/53658729/answer/255221757)
- [CSDN博客](http://interviewtop.top/#/list)
- [GitHub资料](https://github.com/khanhnamle1994/computer-networking/blob/master/Unit2-Transport/2-1-tcp-service-model-notes.pdf)
