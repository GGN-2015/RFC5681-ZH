# [RFC 5681] TCP 拥塞控制

> 译者：于吴桐，王曦，郭冠男
>
> 如遇翻译中存在谬误，请联系 premierbob@qq.com 进行更正，感激不尽。
>
> 目前本文的翻译工作尚未结束，请勿参考/引用本文内容，如遇问题概不负责。

## 链接目录

[toc]

## 摘要

本文定义了 TCP 所采用的四种相互关联的拥塞控制算法：慢启动(slow start)，拥塞避免(congestion avoidance)，快速重传(fast retransmit) 以及快速恢复(fast recovery)。除此之外，本文还详细地阐述了 TCP 在经历了一段较长时间的闲置(idle) 后应如何再次启动数据传输，同时讨论多种确认生成方法(acknowledgment generation methods)。本篇 RFC 的提出，废除了 RFC 2581。

## 本文的地位

该文档为互联网社区规定了一种标准的追踪协议(track protocol)，请读者讨论并为本文地改善提出建议。本协议的地位请参照当前最新版的“互联网官方协议标准”(STD 1)。我们不限制您发布本文的权力。

## 版权声明

版权所有©2009 IETF Trust 和本文作者。保留所有权力。

BCP 78 以及 IETF Trust 关于 IETF 文档的法律条款对本文的约束力在本文发布之日起生效(http://trustee.ietf.org/license-info)。 请务必仔细阅读上述文档，其中约定了您对本文的权力与限制。

本文件可能包含 2008 年 11 月 10 日前发布或公开的 IETF 材料。这些材料的版权所有者可能并未授予 IETF Trust 即在 IETF 标准流程之外修改这些材料的权力。在从上述材料的版权所有者处获得充分的许可前，不得在 IETF 标准流程之外修改本文，以 RFC 的格式发布本文或将本文翻译成英语外的其他语言不受此条款的限制。

<center>第 1 页</center>

---

## 目录

1. 简介
2. 定义
3. 拥塞控制算法
   1. 慢启动和拥塞避免(Slow Start and Congestion Avoidance)
   2. 快速重传和快速恢复(Fast Retransmit/Fast Recovery)
4. 其他需要考虑的问题
   1. 重启闲置连接
   2. 确认生成(Generating Acknowledgment)
   3. 损失恢复原理(Loss Recovery Mechanisms)
5. 安全问题
6. RFC 2581 相较于 RFC 2001 的改变
7. 本文相较于 RFC 2581 的改变
8. 鸣谢
9. 参考文献
   1. 一般性参考文献
   2. 信息性参考文献

## 1. 简介

本文定义了四种 TCP [RFC 793] 拥塞控制算法：慢启动，拥塞避免，快速重传以及快速恢复。这些算法的设计源于 [Jac88] 和 [Jac90] 两篇文章。这些算法对 TCP 的使用服从 [RFC1122] 中的标准化约定。关于和性增长(additive-increase)与乘性降低(multiplicative-decrease)的更早的工作见 [CJ89]。

> 译者注：和性增长/乘性降低 (AIMD)
>
> [和性增长/乘性降低 - 维基百科，自由的百科全书 (wikipedia.org)](https://zh.wikipedia.org/wiki/和性增长/乘性降低)

需要注意的是，[Ste94] 提供了这些算法的应用示例，[WS95] 解释了这些算法的 BSD 版代码实现。

除了定义这些拥塞控制算法外，本文还定义了 TCP 连接在一段较长时间的闲置后应如何恢复，同时阐述了一些与 TCP ACK 生成的相关问题。

在 [RFC2581] 废除了 [RFC2001] 后，本文废除了 [RFC2581]。

本文按照如下顺序组织。第二部分提供全篇其他部分要使用到的各个定义。第三部分阐述拥塞控制算法。第四部分概述了一些与拥塞控制相关的其他值得关注的问题。第五部分概述了安全相关的问题。

<center>第 2 页</center>

---

对于本文中诸如 “**必须**(MUST)”，“**不得**(MUST NOT)”，“**需要**(REQUIRED)”，“**可以**(SHALL)”，“**不可**(SHALL NOT)”，“**应该**(SHOULD)”，“**不应**(SHOULD NOT)”，“**建议**(RECOMMENDED)”，“**可以**(MAY)”，“**可选**(OPTIONAL)” 等关键词的解释请参照 [RFC2119]。

> 译者注：鉴于英语和汉语之间表述的差异，下文中对上述词语的使用会注明英文原文。为了翻译的通顺，下文中并不保证一定使用上述中文对应词汇（如 “必须” “不得” 等）。

## 2. 定义

本节定义了在全文范围内使用到的术语。

**消息段**(SEGMENT)： TCP/IP 数据和确认包(acknowledge packet) 统称为消息段。

**发送端最大消息段长度**(SENDER MAXIMUM SEGMENT SIZE 简称 SMSS)：SMSS 指发送端(sender) 能够发送的最大消息段长度。该值可能由网络最大传输单元(the maximum transmission unit of the network)、传输路径 MTU 发现算法(the path MTU discovery algorithm) [RFC1191, RFC4821]、RMSS(见下一词条) 以及其他因素决定。

> 译者注：Path MTU Discovery(PMTUD)
>
> [Path MTU Discovery - Wikipedia](https://en.wikipedia.org/wiki/Path_MTU_Discovery)

**接收端最大消息段长度**(RECEIVER MAXIMUM SEGMENT SIZE 简称 RMSS)：RMSS 指接收端愿意(is willing to) 接受的最长消息段长度。在连接建立的初期，该值可通过 MSS 选项(MSS option)由接收端发送。如果 MSS 选项没有被启用，该值等于 $536$ 字节 [RFC1122]。 该长度的计算不含 TCP/IP 请求头与选项(TCP/IP headers and options)。

> 译者注：MSS 即  Maximum Segment Size
>
> 以下内容摘自：[tcp协议 OPTION选项内容介绍–MSS（Maximum segment size） - 彩荷z (caihez.com)](http://blog.caihez.com/?p=683)
>
> MSS 是 TCP选项中最经常出现，也是最早出现的选项。MSS选项占 $4$ byte。MSS是每一个TCP报文段中数据字段的最大长度，注意：只是数据部分的字段，不包括 TCP的头部。TCP在三次握手中，每一方都会通告其期望收到的 MSS（MSS只出现在 SYN 数据包中）如果一方不接受另一方的 MSS 值则定位默认值 $536$ byte。
> MSS 值太小或太大都是不合适。太小，例如MSS值只有 $1$ byte，那么为了传输这 $1$ byte数据，至少要消耗 $20$ 字节 IP头部 $+20$ 字节  TCP 头部 = $40$ byte，这还不包括其二层头部所需要的开销，显然这种数据传输效率是很低的。MSS过大，导致数据包可以封装很大，那么在 IP传输中分片的可能性就会增大，接受方在处理分片包所消耗的资源和处理时间都会增大，如果分片在传输中还发生了重传，那么其网络开销也会增大。因此合理的 MSS 是至关重要的。MSS 的合理值应为保证数据包不分片的最大值。对于以太网 MSS 可以达到 $1460$ byte.

**最大长度消息段**(FULL-SIZED SEGMENT)：最大长度消息段指长度恰为本次通信所允许的最大长度的消息段（换言之，数据部分长度恰为 SMSS 的消息段）。

**接收窗**(RECEIVER WINDOW 简称 rwnd)：接收端最后一次发布的接收窗长度。

> 译者注：上文的原文为
>
> RECEIVER WINDOW (rwnd): The most recently advertised receiver window.

**拥塞窗**(CONGESTION WINDOW 简称 cwnd)：拥塞窗是一个用于限制 TCP 通信过程中所能发送的最大数据量的 TCP 状态变量(TCP state variable)。在任一时刻，TCP 所发送数据的序列号(sequence number) **不得**(MUST NOT) 超过最大确认序列号(highest acknowledged sequence number)与 $cwnd$ 和 $rwnd$ 的较小值之和。

> 译者注：简而言之，对于 TCP 发送的所有数据，都有
>
> Sequence Number $\leq$ Highest Acknowledged Sequence Number + $\min\{cwnd, rwnd\}$

**初始窗**(INITIAL WINDOW 简称 IW)：初始窗指三次握手完成后发送端的拥塞窗长度。

**丢失窗**(LOSS WINDOW 简称 LW)：损失窗指当 TCP 发送端通过重传计时器(retransmission timer) 检测到数据丢失时发送端的拥塞窗大小。

**重启窗**(RESTART WINDOW 简称 RW)： 重启窗指 TCP 从一段闲置时间(idle period)中恢复时拥塞窗的大小（如果使用慢启动算法，更多讨论详见 4.1 节）。

<center>第 3 页</center>

---

**在传数据大小**(FLIGHT SIZE)：已发送但还未被累积确认(cumulatively acknowledged)的数据的总量。

> 译者注：累积确认(cumulatively acknowledged)
>
> 下文内容来自：[累积确认_Beyond_cn的博客-CSDN博客_累计确认](https://blog.csdn.net/beyond_cn/article/details/7653473)
>
> 累积确认这个概念应该不只适用于 TCP协议，也适用其他层，比如链路层。
>
> 一般地讲，如果发送方发了包1，包2，包3，包4；接受方成功收到包1，包2，包3。那么接受方可以发回一个确认包，序号为4(4表示期望下一个收到的包的序号；当然你约定好用3表示也可以)，那么发送方就知道包1到包3都发送接收成功，必要时重发包4。一个确认包确认了累积到某一序号的所有包，而不是对每个序号都发确认包。
>
> 具体到TCP，它对字节编号。比如发送方发了包1，包2，包3；包1含字节0到10，包2含字节11到20，包3含字节21到30。接受方成功收到包1，包2。那么接受方发回一个包含确认序号21的包，发送方就知道字节0到20(包1，包2)都成功收到，必要时要重发的只需从字节21开始。

**重复确认**(DUPLICATE ACKNOWLEDGMENT)：一个确认消息(acknowledgment)被认为是“**重复的**”("duplicate")当且仅当下述五个条件全部满足：

(a) 对于这个确认消息(ACK) 的接收端而言，网络中存在正在向他传输且尚未确认的数据；

> 译者注：我们将  outstanding data 翻译为了 “正在传输且尚未确认的数据”，原文为：
>
>  (a) the receiver of the ACK has outstanding data,

(b) 该确认消息不携带任何数据；

(c) SYN 位和 FIN 位均处于关闭(off) 状态；

(d) 在当前连接(connection) 中，该确认消息的编号(number) 最大（TCP.UNA，见 [RFC793]）；

(e) 该确认消息中发布(advertise) 的接收窗长度与上一个确认消息相同。

对于使用了选择性确认(selective acknowledgments 简称 SACKs，见 [RFC2018, RFC2883]) 的 TCP 而言，可以使用 SACK 信息来确定某个 ACK 是否是“**重复的**”（例如：若当前 ACK 包含了一段未知的 SACK 信息）。

## 3. 拥塞控制算法

本节定义了四种在 [Jac88] 和 [Jac90] 中得到论证的拥塞控制算法：慢启动，拥塞避免，快速重传以及快速恢复。在某些情况下，TCP 发送端比这些算法中所规定的行为更加**保守**(conservative) 或许会对通信更有利；但 TCP 的行为**不得**(MUST NOT) 比这些算法所规定的行为更**激进**(aggressive)（换言之，当根据下述算法计算得到的 $cwnd$ 值不再允许发送端发送数据时，发送端**不得**(MUST NOT) 再发送任何数据）。

请注意，本文中所定义的算法同样适用于那些以丢失情况(loss) 作为拥塞信号的网络。也可使用 [RFC3168] 中定义的显示拥塞通知(Explicit Congestion Notification 简称 ECN) 以实现同样的目的。

### 3.1 慢启动和拥塞避免

为了控制网络中超出接收能力范围数据(outstanding data) 的数量，TCP 发送端**必须**(MUST) 遵守慢启动和拥塞避免算法。为了实现这一算法，我们向 TCP 连接前的状态信息中增加了如下两个变量——拥塞窗与接受窗。其中，拥塞窗(cwnd) 是指发送端在接收到一个确认消息(ACK) 前最多还能继续发送数据的数量，接受窗(rwnd) 是指接收端所建议(advertised) 的超出接收能力范围数据数量的上限。$cwnd$ 和 $rwnd$ 中的较小值决定了数据的传输过程。

状态信息中还有一个名为慢启动阈值(slow start threshold 简称 ssthresh)的变量，如下文所述，该变量的取值决定着当前时刻数据传输过程中使用的算法是慢启动还是拥塞避免。

<center>第 4 页</center>

---

为了防止突发性到来的大量数据拥塞整个网络，如果 TCP发送端在开始向网络发送信息时没有掌握任何关于网络容量的信息，我们就**要求**(requires) 它缓慢地探测(slowly probe) 网络以判断网络的可用容量。在传输刚刚启动或者重传计时器所检测到的丢失已被修复时，我们使用慢启动算法以实现这种“ 缓慢地探测”。慢启动算法还可用于启动 TCP 发送端所使用的“ACK 时钟”(the ACK clock)。 TCP 在使用 “慢启动”、“拥塞控制”或者“丢失恢复”三种算法将数据发送到网络环境中时，都需要依赖于该时钟。

> 译者注：对这一句的翻译存疑，原文为：
>
> Slow start additionally serves to start the "ACK clock" used by the TCP sender to release data into the network in the slow start, congestion avoidance, and loss recovery algorithms.

$cwnd$ 的初始值 $IW$ **必须**(MUST) 不超过下述算法给出的界限。

```
如果 SMSS > 2190 字节:
	IW = 2 * SMSS 且 禁止(MUST NOT) 超过 两个消息段
如果 SMSS > 1095 字节 且 SMSS <= 2190 字节:
	IW = 3 * SMSS 且 禁止(MUST NOT) 超过 三个消息段
如果 SMSS <= 1095 字节:
	IW = 4 * SMSS 且 禁止(MUST NOT) 超过 四个消息段
```

如 [RFC3390] 所述，SYN/ACK 以及针对 SYN/ACK 的确认消息**不得**(MUST NOT) 增加拥塞窗的大小。更进一步地说，如果 SYN 或 SYN/ACK 消息段 发生了丢失，发送端正确地发送一个 SYN 后**必须**(MUST) 将初始窗设为一个至多包含 SMSS 个字节地消息段。

[RFC3390] 中详细地讨论了与设置 $IW$ 有关的基本原理。

对于初始拥塞窗含有多个消息段且协议中采用了路径 MTU 发现算法时 [RFC1191]，如果发送端发现自己采用了过大的 MSS 值，则**应该**(SHOULD) 减小 $cwnd$ 的值，以防止在很短的时间内发送大量较短的消息段。具体实现上，**应该**(SHOULD) 将 $cwnd$ 的值按比例缩小，即按照原先的 MSS 值与探测得到的新最大消息段长度的比例进行缩放。

$ssthresh $的初始值**应该**(SHOULD) 尽可能定得高一些（例如将 ssthresh 设为接收端所发布的最大可能窗口大小），但当拥塞发生时 $ssthresh$ **必须**(MUST) 减小。当将 $ssthresh$ 的值设置得尽可能高时，网络环境提供的条件决定发送的速率(sending rate)，否则限制发送速率的将会是一些主机自身设置的门限(host limit)。而当端系统(end system) 对网络路径已经有了一个比较充分的了解时，仔细设置初始 $ssthresh$ 有利于通信的正常进行（例如，发送端系统在 一个较为合适的 $ssthresh$ 的限制下能够保证不会在网络路径沿途造成阻塞）。

<center>第 5 页</center>

---

当 $cwnd \lt ssthresh$ 时，TCP 发送端采用慢启动算法；当 $cwnd \gt ssthresh$ 时 ，TCP 发送端采用拥塞避免算法。当 $cwnd=ssthresh$ 时，发送端既可以采用慢启动算法，也可以采用拥塞避免算法。

在慢启动的过程中，TCP 发送端每收到一个新的累积确认应答 ACK，它至多可以让 $cwnd$ 的取值增加 SMSS。当 $cwnd$ 的值超过 $ssthresh$ 或者拥塞已被检测到时，慢启动算法结束。尽管传统的 TCP 一般严格按照约定，每收到一个新的累积确认应答 ACK 时就将 $cwnd$ 的值增加 SMSS，但我们**建议**(RECOMMEND) 按照如下的方法调整 $cwnd$:

```
cwnd += min(N, SMSS)            (2)
```

当前收到的 ACK 为先前未被累积确认过的 N 个字节进行了累积确认，上述表达式中的 N 即表示此含义。这个调整源自于恰当字节计数(Appropriate Byte Counting [RFC3465])，当接收端使用一种名为“ACK 分片”([SCW99]) 的技术恶意诱导发送端以期人为地大幅提升 $cwnd$ 时，采用该算法比传统的 TCP 实现更加鲁棒(robust)。ACK 分片是指接收端尽管已经完整地收到了某一含有数据消息段，但接收端却不直接为整个消息段中的全部信息进行累积确认，而是将确认过程拆分为多次，每次发送一个 ACK，每个 ACK 只为刚才收到的部分消息进行确认。如果 TCP 协议的发送端每收到一个 ACK 就直接将 $cwnd$ 的值提高 SMSS，那么在收到了许多“ACK 分片”后，就可能会向网络中发送过大的消息。

在冲突避免的过程中，$cwnd$ 大约每隔每隔 RTT 个时间单位就自增一个最大消息段的长度，其中 RTT 指数据传输的往返时间(round-trip time)。拥塞避免算法持续进行，直到检测到拥塞为止在拥塞避免算法执行过程中增加 $cwnd$ 的基本标准如下：

* **可以**(MAY) 以 SMSS 个字节为单位增加 $cwnd$；
* 每经历 RTT 个时间单位，就**应该**(SHOULD) 按照等式 (2) 对 $cwnd$ 进行一次自增；
* 一次自增不得超过 SMSS 个字节。

我们注意到 [RFC3465] 允许 $cwnd$ 在慢启动过程中以一种试验性的原则以超过 SMSS 个字节为单位进行自增。然而，这种行为在标准的 TCP 中是不被允许的。

我们**建议**(RECOMMENDED)，使用一个状态变量统计收到的所有 ACK 总共为多少字节消息进行了累积确认。（这种做法的缺点是必须额外使用一个状态变量。）当该变量的值达到 $cwnd$ 时，然后将 $cwnd$ 自增至多 SMSS 个字节。请注意，在拥塞避免的过程中，  我们**禁止**(MUST NOT) $cwnd$ 在 一个 RTT 时间内自增超过 SMSS 个字节。这种方法既能保证 TCP 发送者每 RTT 个时间单位增加一个最大段长，又能在 “ACK 分片”攻击到来时提供一定的鲁棒性。

<center>第 6 页</center>

---

在冲突避免过程中，另一个 **可以**(MAY) 供 TCP 发送端使用常用的公式如下所述：

```
cwnd += SMSS*SMSS/cwnd          (3)
```

每当发送端接收到针对一些还未被确认的消息数据的累积确认消息时，$cwnd$ 就会按照上述算式进行相应调整。等式 (3) 为我们提供了一个更为保守的近似算法，能够基本保证每隔 RTT 个单位时间，$cwnd$ 就增加约“最大消息段长度”个字节。（注：倘若接收端每收到两个消息段才做出一次回复，算法 (3) 所实现的效果比我们所要求的要保守——能够做到大概每 2 $\times$ RTT 个时间单位回复一次。）

实现该算法时要注意以下三点：

1. 因为 TCP 相关算法的实现通常都采用整数作为算术的基本类型，当 $cwnd$ 严格大于 SMSS 的平方时，$SMSS^2/cwnd$ 在整数运算意义下将等于零，我们在此时**应该**将结果向上取整即视为 $1$，以保证 $cwnd$ 一定增加。

2. 旧版的实现在等式 (3) 右侧还有一个附加的常数，这是不正确的，这种行为实际上会导致性能下降 [RFC2525]。

3. 有些算法实现以字节为单位记录 $cwnd$ 的值，而也有一些算法以“最大消息段长度”为单位。使用“最大消息段长度”为 $cwnd$ 的单位时等式 (3) 就不可用了，这种情况下一般倾向于前文中讨论过的基于计数的方法(counting approach)。

当 TCP 发送端使用重传计时器检测到消息段丢失且重传计时器还未对该段进行过重传时，$ssthresh$ 的值**必须**(MUST) 被重新设置为一个不超过等式  (4) 给出的值：

```
ssthresh = max(FlightSize / 2, 2 * SMSS)
```

如上文中所述，FlightSize 表示网络中正在传输且尚未确认的数据量(outstanding data)。

另一方面，当 TCP 发送端使用重传计时器监测到段丢失，且该段已经被重传计时器重传了至少一次时，$ssthresh$ 保持不变，不进行修改。

<center>第 7 页</center>

---

实现时要注意：一个很容易犯的错误是错将上述算式中的 $FlightSize$ 写作了 $cwnd$ ，在某些版本的实现中 $cwnd$ 可能会偶发性地大幅上升远远超过 $rwnd$ 的大小。

具体而言，当超时(timeout 见 [RFC2988]) 发生时， $cwnd$ **必须**(MUST) 被设置为一个不超过损失窗(loss window, LW) 的数，（无论初始窗 IW 取值为何）LW 总是等于一个最大消息段数据部分的长度。因此，在重传计时器对发生丢失的段进行了重传之后，TCP 的发送端会从重新让 $cwnd$ 从一个最大消息段长度按照慢启动算法逐步递增至新的 $ssthresh$ 值，增加至超过 $ssthresh$ 后转而使用冲突避免算法。

### 3.2 快速重传/快速恢复



#  未完待续 ...

2022-10-19