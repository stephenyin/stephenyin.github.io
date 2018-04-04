---
layout: post
title:  "QUIC 协议学习"
date:   2018-04-04 10:17:33 +0800
categories: QUIC Protocol
tags: Protocol
---
# Overview
![](http://ww1.sinaimg.cn/large/6ba3b183gy1fpq9awol8pj20qe0c80tu.jpg)
## Key advantages
### User space dev
QUIC 实现在 user space，而不是在kernal space. 快速迭代开发.
### Connection establishment latency
* 0-RTT 握手过程
    QUIC 握手的过程是需要一次数据交互，0-RTT 时延即可完成握手过程中的密钥协商，比 TLS 相比效率提高了 5 倍，且具有更高的安全性。
    QUIC 在握手过程中使用 Diffie-Hellman 算法协商**初始密钥**，初始密钥依赖于服务器存储的一组配置参数，该参数会周期性的更新。
    初始密钥协商成功后，服务器会提供一个临时随机数，双方根据这个数再生成**会话密钥**。

    **具体握手过程如下**
    1. 客户端判断本地是否已有服务器的全部配置参数，如果有则直接跳转到(5)，否则继续
    2. 客户端向服务器发送 inchoate client hello(CHLO) 消息，请求服务器传输配置参数
    3. 服务器收到 CHLO，回复 rejection(REJ) 消息，其中包含服务器的部分配置参数
    4. 客户端收到 REJ，提取并存储服务器配置参数，跳回到(1) 
    5. 客户端向服务器发送 full client hello 消息，开始正式握手，消息中包括客户端选择的公开数。此时客户端根据获取的服务器配置参数和自己选择的公开数，可以计算出初始密钥。
    6. 服务器收到 full client hello，如果不同意连接就回复 REJ，同(3)；如果同意连接，根据客户端的公开数计算出初始密钥，回复 server hello(SHLO) 消息，SHLO 用初始密钥加密，并且其中包含服务器选择的一个临时公开数。
    7. 客户端收到服务器的回复，如果是 REJ 则情况同(4)；如果是 SHLO，则尝试用初始密钥解密，提取出临时公开数
    8. 客户端和服务器根据临时公开数和初始密钥，各自基于 SHA-256 算法推导出会话密钥
    9. 双方更换为使用会话密钥通信，初始密钥此时已无用，QUIC 握手过程完毕。之后会话密钥更新的流程与以上过程类似，只是数据包中的某些字段略有不同。
    
    **如果客户端的版本不被服务器接受，则将导致 1-RTT 的延迟。服务器将发送一个版本协商包给客户端。这个包将设置版本标记，并将包含服务器支持的版本的集合。
    当客户端从服务器收到一个版本协商包，它将选择一个可接受的协议版本并使用这个版本重发所有包。**

### Improved congestion control
**BBR**
### Multiplexing without head-of-line blocking
QUIC 的多路复用和 HTTP2 类似。在一条 QUIC 连接上可以并发发送多个 HTTP 请求 (stream)。
QUIC 一个连接上的多个 stream 之间没有依赖。这样假如 stream2 丢了一个 udp packet，也只会影响 stream2 的处理。不会影响 stream2 之前及之后的 stream 的处理。

这也就在很大程度上缓解甚至消除了队头阻塞的影响。

多路复用是 HTTP2 最强大的特性，能够将多条请求在一条 TCP 连接上同时发出去。但也恶化了 TCP 的一个问题，队头阻塞 QUIC 最基本的传输单元是 Packet，不会超过 MTU 的大小，整个加密和认证过程都是基于 Packet 的，不会跨越多个 Packet。这样就能避免 TLS 协议存在的队头阻塞。

Stream 之间相互独立，比如 Stream2 丢了一个 Pakcet，不会影响 Stream3 和 Stream4。不存在 TCP 队头阻塞。
总体来说，QUIC 在传输大量数据时，比如视频，受到队头阻塞的影响很小。
### Forward error correction
**FEC**
### Connection migration
QUIC连接由一个 64-bit 连接 ID 标识，它由客户端随机地产生。在IP地址改变和 NAT 重绑定时，QUIC 连接可以继续存活，因为连接 ID 在这些迁移过程中保持不变。由于迁移客户端继续使用相同的会话密钥来加密和解密数据包，QUIC还提供了迁移客户端的自动加密验证。因此无需重新进行握手。

该特性除了可以减少无谓的连接重连之外，还可以充分利用设备的不同网络接口，进行资源的并行下载。因为虽然这些网络接口有不同的IP，但只要他们能够共享连接UUID，就能够并行的从服务器下载数据。

# BBR
### 基于带宽延迟的拥塞控制（BDP-based Congestion Control | model-based congestion control)
> BDP : bandwidth-delay product

* 网络内尚未被确认的的数据包数量 = 网络链路上能容纳的数据包数量 = 链路带宽 * 往返延迟
* 管道容量 = 最大带宽 * 最小RTT
### 拥塞控制
* 接收方的拥塞控制-- 由接收方告知，只关注自身缓存情况，不关注网络，这里不讨论
* 发送方的拥塞控制
    * 现在广泛使用的 CUBIC/(new)Reno 都是基于丢包的，在算法上重点输出拥塞窗口（cwnd）；
    * BBR 输出 cwnd 和 pacing_rate，且pacing_rate为主，cwnd为辅.也就是说早期的拥塞控制输出cwnd只是告诉tcp可以发多少数据，而没有说怎么发，恰好，BBR输出的pacing_rate就是告诉TCP怎么发。

![](http://ww1.sinaimg.cn/large/6ba3b183gy1fpr70vovobj219q0p440d.jpg)
![](http://ww1.sinaimg.cn/large/6ba3b183gy1fpr740vreej21940sstb7.jpg)
![](http://ww1.sinaimg.cn/large/6ba3b183gy1fpr768xqavj21a60sc77v.jpg)
根据"测不准原理", 要同时测量链路的最大带宽和最小延迟是测不准的. 因为要测量最大带宽就需要缓冲区有一定量的数据包,此时延迟就会较高;要测量最低延迟, 就需要缓冲区基本为空, 但此时带宽就较低.
### Core estimators and control logic
Controls sending based on the model, to move toward network's best operating point 
* On each ACK, update our model of the network path 
    * bottleneck_bandwidth = windowed_max(delivered / elapsed, 10 round trips)
    * min_rtt = windowed_min(rtt, 10 seconds)
* Send rate near available bandwidth (primary) 
    * Pacing rate = pacing_gain * BtlBw
* Volume of data in flight near BDP (secondary) 
    * Max inflight = cwnd_gain * BDP = cwnd_gain * BtlBw * RTprop
 
![](http://ww1.sinaimg.cn/large/6ba3b183gy1fpr78i2vflj21cg0p4gpr.jpg)
### STARTUP: exponential growth to quickly fill pipe (like slow-start)
    * stop growth when bw estimate plateaus, not on loss or delay (Hystart)
    * pacing_gain = 2.89, cwnd_gain = 2.89

![](http://ww1.sinaimg.cn/large/6ba3b183gy1fpr7cl0gewj212a0lwq60.jpg)
连接刚建立时, BBR 采用标准 TCP 慢启动, 指数增长发送速率. 区别是 TCP 遇到丢包就进入拥塞避免阶段; BBR 根据收到的确认包, 发现有效带宽不再增长时,进入拥塞避免阶段.
1. 链路上的错误丢包率较高时, TCP 带宽占用较低, 而对 BBR 没有影响;
2. 链路上的错误丢包率较低时, TCP 总要占满网络 buffer 才放弃, 而 BBR 当发送速率开始占用网络 buffer 时就及时放弃

**慢启动过程中, 开始时的延迟估计就是延迟的最小值, 结束时的有效带宽就是带宽的最大值.**
### DRAIN: drain the queue created in STARTUP
    * pacing_gain = 0.35 = 1/2.89, cwnd_gain = 2.89

![](http://ww1.sinaimg.cn/large/6ba3b183gy1fpr7d5ldarj211s0lm0vn.jpg)
![](http://ww1.sinaimg.cn/large/6ba3b183gy1fpr8bzlqqrj21bw0swqe5.jpg)

慢启动结束后, BBR 要把多占用的带宽*延迟消耗掉, 进入 drain 阶段, 指数降低发送码率, 直到延迟不在降低.
drain 之后 BBR 进入稳定状态, 交替探测 BW 和 RTT, 由于网络带宽的变化比 rtt 变化更频繁, bbr 绝大多数时间在探测 BW. 
### PROBE_BW: explore max BW, drain queue, cruise
* 带宽探测是一个正反馈系统: 定期尝试增加发包速率, 如果收到的确认速率也增加了就进一步增加发送速率.
    * cycle pacing_gain to explore and fairly share bandwidth (cwnd_gain = 2 in all phases)
        * [ 1.25, 0.75, 1, 1, 1, 1, 1, 1] (1 phase per min RTT)
        * Pacing_gain = 1.25 => probe for more bw
        * Pacing_gain = 0.75 => drain queue and yield bw to other flows
        * Pacing_gain = 1.0 => cruise with full utilization and low, bounded queue

以每8个 RTT 为周期, 第一个 RTT BBR 尝试增加 1/4 的发包速率, 第二个 RTT 降低 1/4 的发送速率. 剩下的6个 RTT 以预估网络容量发包.
![](http://ww1.sinaimg.cn/large/6ba3b183gy1fpr7e2zagpj21240ls0vn.jpg)

* 网络带宽增加一倍的情况. 1.25 ^ 3 = 1.95, 三个周期(24个 RTT 收敛)

![](http://ww1.sinaimg.cn/large/6ba3b183gy1fpqfcqozppj20ic0fwdnj.jpg)
* 网络带宽减少一倍的情况
 
![](http://ww1.sinaimg.cn/large/6ba3b183gy1fpqfnccsltj20i80fsgux.jpg)

多出来的包占用了 buffer, 导致 RTT 增加. 由于带宽估计采用窗口时间里的极大值, 需要一定时间带宽下降才会反馈到带宽估计中.

* 带宽不变的情况
![](http://ww1.sinaimg.cn/large/6ba3b183gy1fpqfv7kzzvj20qa0feqcb.jpg)


### PROBE_RTT: if needed, occasionally send slower to probe min RTT
    * Maintain inflight of 4 for at least max(1 round trip, 0.2 sec)Ȁ pacing_gain = 1.0

![](http://ww1.sinaimg.cn/large/6ba3b183gy1fpr7etnx02j21480lo430.jpg)
BBR 每过10秒钟, 没有发现更低的延迟,就进入延迟探测阶段. 延迟探测阶段持续时间为 max(200 ms, RTT), 这段时间只发送 4个包, 将这段时间得到的最小延迟作为新的延迟估计. 差不多有 2% 的时间在探测最小 RTT.

### BBR state transition diagram
![](http://ww1.sinaimg.cn/large/6ba3b183gy1fpr845zd6mj20k80ladh2.jpg)
![](http://ww1.sinaimg.cn/large/6ba3b183gy1fpr85244xaj21gi0s849j.jpg)
BBR 采取了交替测量带宽和延迟的方法:
测量最小RTT（10s内），最大Bandwidth（10 Round Trips）
BBR总是在测量最小RTT（10s内），最大Bandwidth（10 Round Trips），并且尽量控制输出到网络的数据包（in-flight）靠近 BDP（without buffer），这样既能保证带宽利用率，又能避免Bufferbloat问题。

# FEC
* 实现基于 XOR，提供 N + 1 冗余。
* 将被弃用。
* 对于尾部丢包的情况，更简单和更有效的选择是尾部丢包探测（TLP : tail loss probe）。
## 优点
* 低计算开销并且相对易于实现。
## 缺点
* 一个分组中只能恢复一个数据包。如果丢失了两个或多个数据包，则会浪费 FEC 数据包。
* 因为 QUIC 的 FEC 是基于数据包的，而 QUIC 的重新传输是基于帧的，所以重新发送帧不会帮助恢复丢失的数据包。
## 数据
* 来自 Chrome Stable 的数据表明，只有28％的丢包情况是单个数据包。
## 策略
* 将 FEC 数据包视为普通数据包。发送 FEC 数据包时，会消耗 CWND，并且如果 FEC 数据包丢失，则会导致 CWND 减少。
* QUIC 的拥塞窗口不会阻止 FEC 数据包的发送
* 设置 FEC 分组大小为当前 CWND 的 1/2。
* 在四分之一的静止 RTT 之后发送 FEC 数据包，以覆盖发送数据包的 RTT 的后半部分。这是激进的 TLP 风格的 FEC。
    * Sending an FEC packet after a quarter of an RTT of quiescence to cover the last half an RTT of sent packets.  This is aggressive TLP-style FEC.