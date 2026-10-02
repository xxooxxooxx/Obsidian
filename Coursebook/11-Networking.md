---
bibliography:
- networking/networking.bib
link-citations: true
title: "**CS341 系统编程课程手册**"
---

- [[#^networking|网络]]
  - [[#^the-osi-model|OSI 模型]]
  - [[#^layer-3-the-internet-protocol|第 3 层：互联网协议]]
    - [[#^whats-the-deal-with-ipv6|IPv6 到底是怎么回事？]]
    - [[#^whats-my-address|我的地址是什么？]]
  - [[#^layer-4-tcp-and-client|第 4 层：TCP 与客户端]]
    - [[#^note-on-network-orders|关于网络字节序的说明]]
    - [[#^tcp-client|TCP 客户端]]
    - [[#^sending-some-data|发送一些数据]]
  - [[#^layer-4-tcp-server|第 4 层：TCP 服务器]]
    - [[#^example-server|示例服务器]]
    - [[#^sorry-to-interrupt|抱歉打断一下]]
  - [[#^layer-4-udp|第 4 层：UDP]]
    - [[#^udp-attributes|UDP 的属性]]
    - [[#^udp-client|UDP 客户端]]
    - [[#^udp-server|UDP 服务器]]
  - [[#^layer-7-http|第 7 层：HTTP]]
    - [[#^whats-my-name|我叫什么名字？]]
  - [[#^non-blocking-io|非阻塞 IO]]
    - [[#^epoll|epoll]]
    - [[#^epoll-example|Epoll 示例]]
    - [[#^assorted-epoll-gotchas|Epoll 的各种坑]]
  - [[#^remote-procedure-calls|远程过程调用]]
    - [[#^privilege-separation|权限分离]]
    - [[#^stub-code-and-marshaling|桩代码与编组]]
    - [[#^interface-description-language|接口描述语言]]
    - [[#^transferring-structured-data|传输结构化数据]]
  - [[#^topics|主题]]
  - [[#^questions|问题]]


# 网络 ^networking

**我想象中的 Web，我们至今仍未见到。未来
仍然比过去大得多** —— **Tim Berners-Lee**

在过去的 10 到 20 年里，网络可以说已经成为
计算机最重要的用途。如今我们大多数人都无法忍受
一个没有 WiFi 或任何连通性的地方，所以作为
程序员，理解网络以及如何编写跨
网络通信的程序就至关重要。虽然这听起来
很复杂，但 POSIX 定义了很好的
标准，让连接到外部世界变得容易。POSIX 同时
让你能掀开引擎盖去优化每个连接中
那些细小的部分，从而写出性能极高的程序。

作为一项补充（下一章你会读到更多相关内容），
我们在表示大小时会很严格。这意味着当我们
提到 Kilo-、Mega- 等 SI 前缀时，指的
总是 10 的幂。千字节是一千字节，兆字节是一千个
千字节，依此类推。如果我们需要提到 `1024` 字节，
我们会使用更准确的术语 Kibibyte。
Mebibyte 和 Gibibyte 分别是
Megabyte 和 Gigabyte 的对应物。我们做出这一区分
是为了确保我们不会差了 24 倍。这个误称
的原因会在文件系统一章中解释。

## OSI 模型 ^the-osi-model

开放系统互连 7 层模型（OSI 模型）是一系列
层，它们为各种形式的无线电通信（在我们
这里就是互联网）的基础设施和协议定义标准。
这 7 层模型如下：

1.  第 1 层：物理层。这些是把波特
   沿线路传送的实际波。顺带一提，比特并不是
   直接沿线路传输的，因为在大多数介质中
   你可以改变一个波的两个特性
   —— 振幅和频率 —— 从而在每个时钟
   周期获得更多比特。

2.  第 2 层：链路层。这是各个实体对
   某些事件的反应方式（错误检测、有噪声的信道等）。
   以太网和 WiFi 就住在这里。

3.  第 3 层：网络层。这是互联网的核心。
   底下两层协议处理两台直接相连的不同
   计算机之间的通信。这一层负责把
   数据包从一个端点路由到另一个端点。

4.  第 4 层：传输层。这一层规定
   数据的各个切片是如何被接收的。
   底下三层不保证数据包的接收顺序，
   也不保证数据包被丢弃时会发生什么。
   通过使用不同的协议，这一层可以做到。

5.  第 5 层：会话层。这一层确保
   如果前面各层中的某个连接被中断，
   可以在更低的层上建立一个新的连接，
   而在终端用户看来好像什么都没发生。

6.  第 6 层：表示层。这一层处理加密、
   压缩和数据转换。例如不同操作系统之间的
   可移植性，比如把换行转换为 Windows
   的换行符。

7.  第 7 层：应用层。HTTP 和 FTP 都在
   这一层定义。这通常是我们定义跨
   互联网协议的地方。作为程序员，只有当我们认为
   能创造出比下面所有算法都更贴合
   我们需求的算法时，我们才会往下走。

本书不会深入讲解网络。我们将聚焦于
第 3、4、7 层的某些方面，因为如果你打算
用互联网做点什么，这些是必须了解的——
而你职业生涯中的某个时刻肯定会做这类事。
关于另一个定义：协议是由
互联网工程任务组（IETF）提出的一组
规范，规定了协议的实现者在特定情形下
应如何让他们的程序或电路运行。

## 第 3 层：互联网协议 ^layer-3-the-internet-protocol

下面是互联网协议（IP）的简短介绍，它是
把信息数据报从一台机器发送到
另一台机器的主要方式。"IP4"，或者更准确地
说 IPv4，是互联网协议的第 4 个版本，
它描述了如何把信息数据包跨
网络从一台机器发送到另一台机器。IPv4 在几十年里
承载了大部分互联网流量，如今仍被广泛使用，
不过 IPv6 正在追上来（见下文）。
IPv4 的一个显著局限是源地址和
目的地址被限制为 32 位。IPv4 设计于
这样一个时代：40 亿台设备连到同一个网络的
想法还难以想象，或者至少不值得为此把包大小
再加大。IPv4 地址通常写成由句点分隔的
四个八位组，例如 "255.255.255.0"。

每个 IPv4 数据报都包含一个小的首部——通常是
20 个八位组，其中包含源地址和
目的地址。从概念上讲，每个源地址和
目的地址都可以分成两部分：高若干位是
网络号，低若干位表示该网络上
某个特定的主机号。

较新的分组协议 IPv6 解决了 IPv4 的
许多局限，比如让路由表更简单以及 128 位地址。
最初的采用过程很缓慢：2018 年，只有很小一部分
网络流量使用 IPv6（<a href="#ref-internet_society_2018">[6]</a>）。
到 2020 年代中期，Google 测得大约一半的用户流量
通过 IPv6 到达
（<a href="#ref-google_ipv6_stats">[4]</a>）。我们把
IPv6 地址写成由冒号分隔的八组四位十六进制数，
例如 "1F45:0000:0000:0000:0000:0000:0000:0000"。由于这样可能
变得难以驾驭，我们可以把连续的全零组中的一段
替换成 "::"，于是上面的地址变成 "1F45::"。"::"
在一个地址中只能出现一次；否则你就无法分辨
每一个各代表多少个零组。一台机器可以同时
拥有一个 IPv6 地址和一个 IPv4
地址。

有一些特殊的 IP 地址。IPv4 中一个是 `127.0.0.1`，
IPv6 中则是 `0:0:0:0:0:0:0:1` 或 `::1`，它们
也称为 localhost。发往
127.0.0.1 的报文永远不会离开这台机器；该地址
被规定为指向本机。还有很多其他地址是通过
某些八位组为 0 或 255（即最大值）来表示的。
你不需要知道所有这些术语，只要记住：
一台机器在整个互联网中实际能拥有的
全局 IP 地址数量，要小于"原始"地址的
数量。本书介绍 IP 如何处理路由，
以及如何为上层协议分片和重新
分片。下面还有一个更深入的旁注。

### IPv6 到底是怎么回事？ ^whats-the-deal-with-ipv6

<figure data-latex-placement="H">
<p><img
src="附件/ipv6_datagram.png"
alt="以 32 位为一行绘制的 IPv6 首部：先是版本号、流量类和流标签；然后是载荷长度、下一首部和跳数限制；接着是源地址和目的地址这两大块。" /></p>
<figcaption>IPv6 数据报的可整除性</figcaption>
</figure>

在上面那个首部中，最前面的 32 位包含一个 4 位
版本号、一个 8 位流量类，以及一个 20 位流标签。
接下来的 32 位包含 16 位
载荷长度、8 位下一首部字段和 8 位跳数限制，
随后是 128 位的源地址和目的地址。

IPv6 的一项重大特性是地址空间。多年前
世界就用尽了 IPv4 地址（IANA 在 2011 年
发放了最后一批免费地址块），此后一直在用各种
取巧办法绕过。IPv6 有足够多的内部和
外部地址，以至于即使我们发现了外星
文明，大概也不会用光。

另一项重大特性是通过 IPsec 实现安全。
IPv4 设计时几乎或完全没有考虑安全性。
因此现在在更高的层上有一个类似 TLS 的
密钥交换，让你能够加密
通信。

另一项特性是处理被简化。为了让互联网变快，
IPv4 和 IPv6 的首部都是在硬件中校验的。
这意味着所有首部选项都是随着到达
在电路中被处理的。问题在于，
随着 IPv4 规范不断扩充以包含大量首部，
硬件也不得不变得越来越先进才能支持那些首部。
IPv6 重新排列了首部的顺序，使得
数据包可以以更少的硬件周期被丢弃和路由。
就互联网而言，在试图路由全世界的
流量时，每一个周期都很重要。

### 我的地址是什么？ ^whats-my-address

要获得当前机器 IP 地址的一个链表，请使用
`getifaddrs`，它除了其他接口之外还会返回 IPv4 和 IPv6 IP
地址的链表。我们可以检查每个条目，
并用 `getnameinfo` 打印主机的 IP 地址。`ifaddrs` 结构体
包含 family 字段，但不包含该结构体的 sizeof。
因此我们需要根据
family 手动确定结构体大小。

``` objectivec
(family == AF_INET) ? sizeof(struct sockaddr_in) : sizeof(struct sockaddr_in6)
```

完整代码如下。

``` objectivec
int required_family = AF_INET; // Change to AF_INET6 for IPv6
struct ifaddrs *myaddrs, *ifa;
getifaddrs(&myaddrs);
char host[256], port[256];

for (ifa = myaddrs; ifa != NULL; ifa = ifa->ifa_next) {
  int family = ifa->ifa_addr->sa_family;
  if (family == required_family && ifa->ifa_addr) {
    int ret = getnameinfo(ifa->ifa_addr,
    (family == AF_INET) ? sizeof(struct sockaddr_in) :
    sizeof(struct sockaddr_in6),
    host, sizeof(host), port, sizeof(port)
    , NI_NUMERICHOST | NI_NUMERICSERV)
    if (0 == ret) {
      puts(host);
    }
  }
}
```

要从命令行获取你的 IP 地址，请使用 `ifconfig` 或者 Windows 的
`ipconfig`。

不过这条命令会为每个接口产生大量输出，
所以我们可以用 grep 过滤输出。

    ifconfig | grep inet

    Example output:
        inet6 fe80::1%lo0 prefixlen 64 scopeid 0x1
        inet 127.0.0.1 netmask 0xff000000
        inet6 ::1 prefixlen 128
        inet6 fe80::7256:81ff:fe9a:9141%en1 prefixlen 64 scopeid 0x5
        inet 192.168.1.100 netmask 0xffffff00 broadcast 192.168.1.255

要获取某个远程网站的 IP 地址，函数 `getaddrinfo`
可以把人类可读的域名（例如 `www.illinois.edu`）转换成
一个 IPv4 和一个 IPv6 地址。它会返回一个
addrinfo 结构体的链表：

``` objectivec
struct addrinfo {
  int              ai_flags;
  int              ai_family;
  int              ai_socktype;
  int              ai_protocol;
  socklen_t        ai_addrlen;
  struct sockaddr *ai_addr;
  char            *ai_canonname;
  struct addrinfo *ai_next;
};
```

比如说，假设你想查出 `www.bbc.com` 上某个
Web 服务器的数字 IPv4 地址。我们分两个阶段来做。
首先，用 getaddrinfo 构建一个可能连接的
链表。其次，用 `getnameinfo` 把其中某个的
二进制地址转换成可读的形式。

``` objectivec
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>

struct addrinfo hints, *infoptr; // So no need to use memset global variables

int main() {
  hints.ai_family = AF_INET; // AF_INET means IPv4 only addresses

  // Get the machine addresses
  int result = getaddrinfo("www.bbc.com", NULL, &hints, &infoptr);
  if (result) {
    fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(result));
    exit(1);
  }

  struct addrinfo *p;
  char host[256];

  for(p = infoptr; p != NULL; p = p->ai_next) {
    // Get the name for all returned addresses
    getnameinfo(p->ai_addr, p->ai_addrlen, host, sizeof(host), NULL, 0, NI_NUMERICHOST);
    puts(host);
  }

  freeaddrinfo(infoptr);
  return 0;
}
```

可能的输出。

    212.58.244.70
    212.58.244.71

传入 `AF_UNSPEC`（未指定）则同时接受 IPv4 或 IPv6。
只需把上面代码中的 ai_family 属性替换为
下面这句。

    hints.ai_family = AF_UNSPEC

如果你好奇计算机是如何把主机名映射到地址的，
我们会在第 7 层讨论。剧透一下：那是一个
叫 DNS 的服务。
在进入下一节之前，有一点很重要：
一个网站可以有多个 IP 地址。这可能是
出于针对不同机器做优化的考虑。
如果 Google 或 Facebook 只有一台服务器
把*所有*传入请求都路由到其他计算机，
它们就得在那台计算机或那个数据中心上
花掉巨额的钱。
相反，它们可以给不同区域不同的 IP 地址，
让计算机自己去挑。通过
非首选的 IP 地址访问网站也没什么不好。
页面可能加载得慢一些。

## 第 4 层：TCP 与客户端 ^layer-4-tcp-and-client

<figure data-latex-placement="H">
<p><img
src="附件/tcp_header.png"
alt="以 32 位为一行绘制的 TCP 首部：源端口和目的端口；序号；确认号；数据偏移、保留位、标志和窗口；校验和与紧急指针；最后是变长选项和填充。" /></p>
<figcaption>补充：TCP 首部规范</figcaption>
</figure>

在上面的首部中，每一行都是 32 位宽。16 位的源端口和
目的端口排在最前，然后是 32 位的序号和
确认号，然后是数据偏移、保留位、标志以及
一个 16 位的窗口大小，然后是校验和与紧急指针，
最后是若干选项，并填充到 32 位的整数倍。

如今互联网上的大多数服务都使用 TCP，因为它
高效地隐藏了互联网底层那种
基于分组的复杂性。TCP 也就是
传输控制协议，是一种基于连接的协议，
构建在 IPv4 和 IPv6 之上，因此可以被描述为 "TCP/IP"
或"TCP over IP"。TCP 在两台机器之间建立一条
*管道*，并把互联网底层的
基于分组的性质抽象掉。因此，在
大多数情况下，通过 TCP 连接发送的字节都会被
完整无误地送达。高性能、容错的代码甚至
都不会假设字节一定被送达！

TCP 有许多特性使它区别于另一种传输
协议 UDP。

1.  端口 有了 IP，你只能把数据包发到一台机器。
    如果你希望一台机器处理多路数据流，
    就必须用 IP 手动实现。TCP 给程序员
    提供了一组虚拟端口。客户端指定
    它们希望数据包被发送到的端口，
    TCP 协议确保等待该端口上数据包的
    应用程序能收到它。一个进程可以监听
    某个特定端口上的传入数据包。不过，只有
    拥有超级用户（root）权限的进程才能监听
    小于 1024 的端口。任何进程都可以监听
    1024 或更高的端口。一个最常用的端口
    是 80 号。它用于未加密的 HTTP 请求或网页。
    比如，如果一个 Web 浏览器连接到 `http://www.bbc.com/`，
    它连的就是 80 端口。

2.  重传 由于网络错误或
    拥塞，数据包可能会被丢弃。因此它们需要
    被重传。同时，重传又不该导致更多数据包
    被丢弃。这需要在"把网络淹掉"和
    速度之间取得平衡。

3.  乱序报文。报文可能因为各种
    原因在 IP 层被更有利地路由。如果
    后面的报文先于另一个报文到达，
    协议应当能检测并重新排序。

4.  重复报文。报文可能到达两次。报文可能
    到达两次。因此协议需要能够在
    序号发生溢出的情况下区分两个
    报文。

5.  差错纠正。TCP 有一个
    处理比特错误的校验和。
    不过这很少被用到。

6.  流量控制。流量控制是在接收端
    执行的。这样做可能是为了避免
    慢速接收方被数据包淹没。
    处理 10000 个或一千万个并发
    连接的服务器可能需要让接收方放慢速度
    但保持连接，因为负载过重。另一个
    问题是确保本地网络的流量
    保持稳定。

7.  拥塞控制。拥塞控制是在发送方
    一侧执行的。拥塞控制是为了
    防止发送方用太多数据包淹没
    网络。这一点很重要，
    以确保每条 TCP 连接都被
    公平对待。也就是说，从同一台计算机
    出发的两条分别去 google 和 youtube 的连接，
    彼此获得相同的带宽和 ping 延迟。
    完全可以定义一个协议独占
    所有带宽、把其他协议撇在一边，但这
    往往是恶意的，因为很多时候
    把一台计算机限制在单条 TCP 连接上
    会得到同样的结果。

8.  面向连接／面向生命周期。你可以把一条 TCP
    连接想象成通过管道发送的一系列
    字节。不过 TCP 连接是有
    "生命周期"的。TCP 通过 SYN、SYN-ACK、ACK
    来处理连接的建立。这意味着
    客户端会发送一个 SYNchronization
    （同步）包，告诉 TCP 从哪个起始序号开始。
    然后接收方会发送一个 SYN-ACK 消息
    来确认该同步号。接着客户端会用
    最后的一个包来 ACKnowledge（确认）它。
    此时连接对两端的读写都已
    打开。TCP 会发送数据，
    数据的接收方会确认它收到了一个包。
    然后每隔一段时间，如果没有包被发送，
    TCP 会交换零长度的包以确认
    连接仍然存活。在任何时刻，
    客户端和服务器都可以发送一个 FIN 包，
    表示服务器将不再传输。这个包
    可以通过一些比特位来改变，使其
    只关闭某条连接的读端或写端。
    当所有端都关闭后，连接就结束了。

不过 TCP 并不提供很多东西。

1.  安全性。连接到一个自称是某个
   网站的 IP 地址并不会核实这个说法
   （不像 TLS）。你可能正在把
   数据包发给一台恶意计算机。

2.  加密。任何人都能窃听明文
   TCP。传输中的数据包是
   明文。像你的密码
   之类的重要内容很容易被旁观者一眼看穿。

3.  会话重连。如果一条 TCP 连接断开，
   就必须创建一个全新的连接，
   传输也必须从头再来。
   这由更高层的协议来处理。

4.  界定请求边界。TCP 天生是面向连接的。
   在 TCP 上通信的应用程序
   需要找到一种独特的方式
   来告诉对方这个请求或响应结束了。
   HTTP 用两个回车来界定首部，
   并使用一个长度字段，或者一直监听
   直到连接关闭。

### 关于网络字节序的说明 ^note-on-network-orders

整数既可以用最低有效字节在前的顺序
表示，也可以用最高有效字节在前的顺序
表示。只要机器自身内部保持一致，
两种方式都说得通。对于网络通信，
我们需要在约定的格式上取得标准。

`htons(xyz)` 以网络字节序返回 16 位无符号整数
'short' 值 xyz。`htonl(xyz)` 以网络字节序返回 32 位无符号整数
'long' 值 xyz。任何更长的整数都需要
由双方约定顺序。

这些函数被读作"从主机到网络"。它们的
反向函数（`ntohs`、`ntohl`）把网络序的字节值
转换为主机字节序。那么，主机序是
小端还是大端呢？答案是
—— 取决于你的机器！它取决于运行这段代码的
主机的实际体系结构。如果该体系结构恰好
和网络序相同，那么这些函数返回相同的
整数。对于 x86 机器，主机序和网络序是不同的。

除非另有约定，每当你读写底层的 C 语言
网络结构体（即端口和地址信息）时，
请记得使用上面这些函数以确保与机器
格式之间的正确转换。否则，
所显示或所指定的值可能是错误的。

这不适用于那些事先协商好字节序的
协议。如果两台计算机的 CPU 都忙于在
网络序之间转换消息 —— 在高性能
系统中的 RPC 就是这样 —— 那么
协商一下如果双方体系结构字节序相同
就以小端发送，可能更值得。

为什么网络序被定义为大端？简单的答案是
RFC1700 就是这么规定的（<a href="#ref-RFC1700">[5]</a>）。如果你想
了解更多信息，我们会引用那篇主张
采用某个特定版本的著名文章（<a href="#ref-cohen_1980">[2]</a>）。最
重要的一点是它是标准。没有
统一标准会发生什么？我们有 4 种互不
兼容的 USB 插头类型（Standard、Micro、
Mini 和 USB-C）。请在此处
加入相关的 XKCD
<a href="https://xkcd.com/927/">https://xkcd.com/927/</a>。

### TCP 客户端 ^tcp-client

连接到远程机器有三个基本的系统调用。

1.  `int getaddrinfo(const char *node, const char *service, const struct addrinfo *hints, struct addrinfo **res);`

    The `getaddrinfo` call if successful, creates a linked-list of
    `addrinfo` structs and sets the given pointer to point to the first
    one.

    Also, you can use the hints struct to only grab certain entries like
    certain IP protocols, etc. The addrinfo structure is passed into
    `getaddrinfo` to define the kind of connection you’d like. For
    example, to specify stream-based protocols over IPv6, you can use
    the following snippet.

    ``` objectivec
    struct addrinfo hints;
    memset(&hints, 0, sizeof(hints));

    hints.ai_family = AF_INET6; // Only want IPv6 (use AF_INET for IPv4)
    hints.ai_socktype = SOCK_STREAM; // Only want stream-based connection
    ```

    The other modes for ‘family‘ are `AF_INET` and `AF_UNSPEC` which
    mean IPv4 and unspecified respectively. This could be useful if you
    are searching for a service that you aren’t entirely sure which IP
    version. Naturally, you get the version in the field back if you
    specified UNSPEC.

    Error handling with `getaddrinfo` is a little different. The return
    value *is* the error code. To convert to a human-readable error use
    `gai_strerror` to get the equivalent short English error text.

    ``` objectivec
    int result = getaddrinfo(...);
    if(result) {
      const char *mesg = gai_strerror(result);
      ...
    }
    ```

2.  `int socket(int domain, int socket_type, int protocol);`

    The socket call creates a network socket and returns a descriptor
    that can be used with `read` and `write`. In this sense, it is the
    network analog of `open` that opens a file stream – except that we
    haven’t connected the socket to anything yet!

    Sockets are created with a domain `AF_INET` for `IPv4` or `AF_INET6`
    for `IPv6`, `socket_type` is whether to use UDP, TCP, or some other
    socket type, the `protocol` is an optional choice of protocol
    configuration; for our examples we can leave this as 0 for default.
    This call creates a socket object in the kernel with which one can
    communicate with the outside world/network. You can use the result
    of `getaddressinfo` to fill in the `socket` parameters, or provide
    them manually.

    The socket call returns an integer - a file descriptor - and, for
    TCP clients, you can use it as a regular file descriptor. You can
    use `read` and `write` to receive or send packets.

    TCP sockets are similar to `pipes` and are often used in situations
    that require IPC. We don’t mention it in the previous chapters
    because it is overkill using a device suited for networks to simply
    communicate between processes on a single thread.

3.  `connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);`

    Finally, the connect call attempts the connection to the remote
    machine. We pass the original socket descriptor and also the socket
    address information which is stored inside the addrinfo structure.
    There are different kinds of socket address structures that can
    require more memory. So in addition to passing the pointer, the size
    of the structure is also passed. To help identify errors and
    mistakes it is good practice to check the return value of all
    networking calls, including `connect`.

    ``` objectivec
    // Pull out the socket address info from the addrinfo struct:
    connect(sockfd, p->ai_addr, p->ai_addrlen)
    ```

4.  （可选）要清理代码，请在第一层的
    `addrinfo` 结构体上调用 `freeaddrinfo(struct addrinfo *ai)`。

有一个旧函数 `gethostbyname` 已被废弃。
它是把主机名转换成 IP 地址的旧方法。端口地址仍然
需要用 `htons` 函数手动设置。用更新的 `getaddrinfo`
写出同时支持 IPv4 和 IPv6 的代码要容易
得多。

这就是创建一个*简单* TCP 客户端所需的全部。
不过网络通信提供了许多不同层次的
抽象，以及每一层都可以设置的若干
属性和选项。例如，我们还没有谈到
可以操作套接字选项的 `setsockopt`。
你也可以对更底层的协议做些手脚，
因为内核提供了有助于此的原语。
注意，创建原始套接字需要 root 权限。
此外，你还需要有大量"初始化"或
启动代码，并且要准备好你的数据报
因为格式不对而被丢弃。更多信息请见
<a href="https://beej.us/guide/bgnet/html/split/man-pages.html#getaddrinfoman">https://beej.us/guide/bgnet/html/split/man-pages.html#getaddrinfoman</a>。

### 发送一些数据 ^sending-some-data

一旦连接成功，我们就可以像对待任何普通的
文件描述符那样读或写。请记住，如果你
连的是某个网站，你
需要遵守 HTTP 协议规范才能拿回任何
有意义的结果。有现成的库可以做到这一点。
通常，你不会在套接字层面直接连接。
读或写的字节数
可能比预期的要少。因此，检查 `read` 和 `write`
的返回值很重要。下面是一个向
合规 URL 发送请求的简单 HTTP 客户端。
首先，我们从枯燥的部分
和解析代码开始。

``` objectivec
typedef struct _host_info {
  char *hostname;
  char *port;
  char *resource;
} host_info;

host_info *get_info(char *uri) {
  // ... Parses the URI/URL
}

void free_info(host_info *info) {
  // ... Frees any info
}

int main(int argc, char *argv[]) {
  if(argc != 2) {
    fprintf(stderr, "Usage: %s http://hostname[:port]/path\n", *argv);
    return 1;
  }
  char *uri = argv[1];
  host_info *info = get_info(uri);
  host_info *temp = send_request(info);

  return 0;
}
```

发送请求的代码如下。我们必须做的
第一件事是连接到一个地址。

``` objectivec
struct addrinfo current, *result;
memset(&current, 0, sizeof(struct addrinfo));
current.ai_family = AF_INET;
current.ai_socktype = SOCK_STREAM;

getaddrinfo(info->hostname, info->port, &current, &result);

connect(sock_fd, result->ai_addr, result->ai_addrlen)

freeaddrinfo(result);
```

接下来的这段代码发送请求。下面是每个首部
的含义。

1.  "GET %s HTTP/1.1" 这是把请求方法与
   路径插值后的结果。它的意思是
   用 HTTP/1.1 协议版本对该路径执行 GET 方法。

2.  "Host: %s" 这是我们想交谈的
   服务器名字。HTTP/1.1
   要求必须有这个首部，因为一个 IP 地址常常
   承载许多不同的网站，
   服务器需要知道我们指的是哪一个。

3.  "Connection: close" 意思是
   一旦响应结束，
   就请关闭连接。HTTP/1.1 默认
   保持连接，以便更多请求可以复用它们。
   让服务器关闭可以
   让我们的客户端保持简单：响应的末尾
   就是文件的末尾，所以我们不必去解析 `Content-Length`。

4.  "Accept: \*/\*" 这表示
   客户端愿意接受任何内容。

更健壮的一段代码还会检查
写入是否失败或该调用
是否被中断。

``` objectivec
char *buffer;
asprintf(&buffer,
  "GET %s HTTP/1.1\r\n"
  "Host: %s\r\n"
  "Connection: close\r\n"
  "Accept: */*\r\n\r\n",
  info->resource, info->hostname);

write(sock_fd, buffer, strlen(buffer));
free(buffer);
```

最后一段代码是发送请求的驱动代码。
如果你想把该文件描述符作为一个 FILE 对象打开
以使用那些便捷函数，尽可以使用下面的代码。
只是要小心别忘了把
缓冲设为零，否则你可能会对
输入做双重缓冲，那会带来性能问题。

``` objectivec
void send_request(host_info *info) {
  int sock_fd = socket(AF_INET, SOCK_STREAM, 0);
  // Re-use address is a little overkill here because we are making a
  // Listen only server and we don't expect spoofed requests.
  int optval = 1;
  int retval = setsockopt(sock_fd, SOL_SOCKET, SO_REUSEADDR, &optval,
  sizeof(optval));
  if(retval == -1) {
    perror("setsockopt");
    exit(1);
  }
  // Connect using code snippet

  // Send the get request

  // Open so you can use getline
  FILE *sock_file = fdopen(sock_fd, "r+");
  setvbuf(sock_file, NULL, _IONBF, 0);

  ret = handle_okay(sock_file);
  fclose(sock_file);
  close(sock_fd);
}
```

上面的例子展示了使用
超文本传输协议向服务器发起请求。一般来说，
它包含六个部分：

1.  方法。GET、POST 等。

2.  资源。"/" "/index.html" "/image.png"

3.  协议 "HTTP/1.1"

4.  一个新行（`\r\n`）。请求总是带一个回车。

5.  其他各种旋钮或开关参数，称为首部，每行一个。
   HTTP/1.1 至少要求有 `Host` 首部。

6.  请求的实际正文，它跟在一个空行（连续两个
   换行）之后。接收方要么按 `Content-Length` 首部中
   给出的字节数读取正文，要么
   在没有指定大小时一直读到发送方关闭连接为止。

服务器返回的第一行用一个 3 位响应码
描述了所用的 HTTP 版本，
以及该请求是否成功。

    HTTP/1.1 200 OK

如果客户端请求了一个不存在的路径，例如
`GET /nosuchfile.html HTTP/1.1`，那么第一行就会包含
众所周知的 `404` 响应码。

    HTTP/1.1 404 Not Found

更多信息请见 RFC 9110（它于 2022 年取代了 RFC 7231），
它是当前关于 HTTP 语义的规定，
包括请求方法和状态码（<a href="#ref-rfc9110">[3]</a>）。

HTTP/1.1 并不是故事的结局。若没有 `Connection: close`，
HTTP/1.1 连接在响应之后会保持打开
（持久连接），因此浏览器可以发送
许多请求，而不必每次都付出一次新的
TCP 握手代价。HTTP/2 保留了相同的方法、
首部和状态码，但把它们作为二进制帧发送，
并在单条 TCP 连接上
同时复用多个请求
（<a href="#ref-rfc9113">[7]</a>）。HTTP/3 更进一步，它
运行在构建于 UDP 之上的 QUIC 上，
这样一个丢失的报文不会像在单条 TCP 流上那样
卡住它后面的每一个请求（TCP
队头阻塞）（<a href="#ref-rfc9114">[1]</a>）。你的
浏览器会说上面所有这些协议，但上面那个文本协议
仍然是你手动与服务器交谈时会看到的。

## 第 4 层：TCP 服务器 ^layer-4-tcp-server

创建一个最小 TCP 服务器所需的四个系统调用是
`socket`、`bind`、`listen` 和 `accept`。每一个都有
特定用途，并且大致应该按上面的顺序调用。

1.  `int socket(int domain, int socket_type, int protocol)`

    To create an endpoint for networking communication. A new socket by
    itself stores bytes. Though we’ve specified either a packet or
    stream-based connection, it is unbound to a particular network
    interface or port. Instead, socket returns a network descriptor that
    can be used with later calls to bind, listen and accept.

    As one gotcha, these sockets must be declared passive. Passive
    server sockets do not send or receive data themselves. Instead, they
    wait for incoming connections. Additionally, a passive server socket
    is not used to talk to any particular client, so it remains open
    when a peer disconnects. Instead, the client communicates with a
    separate active socket on the server that is specific to that
    connection.

    Since a TCP connection is defined by the sender address and port
    along with a receiver address and port, for a particular server port
    there can be one passive server socket but multiple active sockets.
    One for each currently open connection. The server’s operating
    system maintains a lookup table that associates a unique tuple with
    active sockets so that incoming packets can be correctly routed to
    the correct socket.

2.  `int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);`

    The `bind` call associates an abstract socket with an actual network
    interface and port. It is possible, though uncommon, to call bind on
    a TCP client; normally the operating system picks the client’s port
    automatically. The port information used by bind can be set manually
    (many older IPv4-only C code examples do this), or be created using
    `getaddrinfo`.

    By default, a port is not released immediately when the server
    socket is closed. Instead, the port enters a “TIME-WAIT” state. This
    can lead to significant confusion during development because the
    timeout can make valid networking code appear to fail.

    To be able to immediately reuse a port, specify `SO_REUSEADDR`
    before binding to the port.

    ``` objectivec
    int optval = 1;
    setsockopt(sfd, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof(optval));

    bind(...);
    ```

    Here’s
    <a href="http://stackoverflow.com/questions/14388706/socket-options-so-reuseaddr-and-so-reuseport-how-do-they-differ-do-they-mean-t">http://stackoverflow.com/questions/14388706/socket-options-so-reuseaddr-and-so-reuseport-how-do-they-differ-do-they-mean-t</a>.

3.  `int listen(int sockfd, int backlog);`

    The `listen` call specifies the queue size for the number of
    incoming, unhandled connections. These are the connections
    unassigned to a file descriptor by `accept`. Typical values for a
    high-performance server are 128 or more.

4.  `int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);`

    Once the server socket has been initialized the server calls
    `accept` to wait for new connections. Unlike `socket` `bind` and
    `listen`, this call will block, unless the nonblocking option has
    been set. If there are no new connections, this call will block and
    only return when a new client connects. The returned TCP socket is
    associated with a particular tuple
    `(client IP, client port, server IP, server port)` and will be used
    for all future incoming and outgoing TCP packets that match this
    tuple.

    Note the `accept` call returns a new file descriptor. This file
    descriptor is specific to a particular client. It is a common
    programming mistake to use the original server socket descriptor for
    the server I/O and then wonder why networking code has failed.

    The `accept` system call can optionally provide information about
    the remote client, by passing in a sockaddr struct. Different
    protocols have different variants of the `struct sockaddr`, which
    are different sizes. The simplest struct to use is the
    `sockaddr_storage` which is sufficiently large to represent all
    possible types of sockaddr. Notice that C does not have any model of
    inheritance. Therefore we need to explicitly cast our struct to the
    ‘base type’ struct sockaddr.

    ``` objectivec
    struct sockaddr_storage clientaddr;
    socklen_t clientaddrsize = sizeof(clientaddr);
    int client_id = accept(passive_socket,
      (struct sockaddr *) &clientaddr,
      &clientaddrsize);
    ```

    We’ve already seen `getaddrinfo` that can build a linked list of
    addrinfo entries and each one of these can include socket
    configuration data. What if we wanted to turn socket data into IP
    and port addresses? Enter `getnameinfo` that can be used to convert
    local or remote socket information into a domain name or numeric IP.
    Similarly, the port number can be represented as a service name. For
    example, port 80 is commonly used as the incoming connection port
    for incoming HTTP requests. In the example below, we request numeric
    versions for the client IP address and client port number.

    ``` objectivec
    socklen_t clientaddrsize = sizeof(clientaddr);
     int client_id = accept(sock_id, (struct sockaddr *) &clientaddr, &clientaddrsize);
     char host[NI_MAXHOST], port[NI_MAXSERV];
     getnameinfo((struct sockaddr *) &clientaddr,
      clientaddrsize, host, sizeof(host), port, sizeof(port),
      NI_NUMERICHOST | NI_NUMERICSERV);
    ```

    One can use the macros `NI_MAXHOST` to denote the maximum length of
    a hostname, and `NI_MAXSERV` to denote the maximum length of a port.
    `NI_NUMERICHOST` gets the hostname as a numeric IP address and
    similarly for `NI_NUMERICSERV` although the port is usually numeric,
    to begin with. The
    <a href="https://man.openbsd.org/getnameinfo.3#NI_NUMERICHOST">https://man.openbsd.org/getnameinfo.3#NI_NUMERICHOST</a>.

5.  `int close(int fd)` 和 `int shutdown(int fd, int how)`

    Use the `shutdown` call when you no longer need to read any more
    data from the socket, write more data, or have finished doing both.
    When you call `shutdown` on socket on the read and/or write ends,
    that information is also sent to the other end of the connection. If
    you shut down the socket for further writing at the server end, then
    a moment later, a blocked `read` call could return 0 to indicate
    that no more bytes are expected. Similarly, a write to a TCP
    connection that has been shut down for reading will generate a
    SIGPIPE.

    Use `close` when your process no longer needs the socket file
    descriptor.

    If you `fork`-ed after creating a socket file descriptor, all
    processes need to close the socket before the socket resources can
    be reused. If you shut down a socket for further read, all processes
    are affected because you’ve changed the socket, not the file
    descriptor. Well written code will `shutdown` a socket before
    calling `close` on it.

创建服务器时有一些坑。

- 使用被动的服务器套接字的文件描述符
  （上文已描述）

- 没有为 getaddrinfo 指定 SOCK_STREAM 要求

- 无法复用已存在的端口。

- 没有初始化未使用的结构体条目

- 如果端口当前正在被使用，`bind` 调用
  会失败。端口是
  按机器算的 —— 不是按进程或按用户算的。
  换句话说，当另一个进程正在使用某个端口时，
  你不能使用端口 1234。更糟的是，默认情况下
  一个进程结束后端口会被"占用"。

### 示例服务器 ^example-server

下面是一个能工作的简单服务器示例。
注意：这个示例是不完整的。
例如，该套接字文件描述符会一直保持打开，
而 `getaddrinfo` 分配的内存也没有释放。
首先，我们获取当前机器的
地址信息。

``` objectivec
struct addrinfo hints, *result;
memset(&hints, 0, sizeof(struct addrinfo));
hints.ai_family = AF_INET;
hints.ai_socktype = SOCK_STREAM;
hints.ai_flags = AI_PASSIVE;

int s = getaddrinfo(NULL, "1234", &hints, &result);
if (s != 0) {
  fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(s));
  exit(1);
}
```

然后我们建立套接字、绑定它并开始监听。

``` objectivec
int sock_fd = socket(AF_INET, SOCK_STREAM, 0);

// Bind and listen
if (bind(sock_fd, result->ai_addr, result->ai_addrlen) != 0) {
  perror("bind()");
  exit(1);
}

if (listen(sock_fd, 10) != 0) {
  perror("listen()");
  exit(1);
}
```

我们终于准备好监听连接了，于是我们通知
用户，并接受我们的第一个客户端。

``` objectivec
struct sockaddr_in *result_addr = (struct sockaddr_in *) result->ai_addr;
printf("Listening on file descriptor %d, port %d\n", sock_fd, ntohs(result_addr->sin_port));

// Waiting for connections like a passive socket
printf("Waiting for connection...\n");
int client_fd = accept(sock_fd, NULL, NULL);
printf("Connection made: client_fd=%d\n", client_fd);
```

之后，我们就可以把这个新的文件描述符当作
字节流来对待，很像一根管道。

``` objectivec
char buffer[1000];
// Could get interrupted
int len = read(client_fd, buffer, sizeof(buffer) - 1);
buffer[len] = '\0';

printf("Read %d chars\n", len);
printf("===\n");
printf("%s\n", buffer);
```

### 抱歉打断一下 ^sorry-to-interrupt

有一个概念我们需要讲清楚：
你必须在自己的网络代码中处理
中断。这意味着你用来读写的套接字
或已接受的文件描述符的调用可能会
被打断 —— 多数时候你会遇到一两次
中断。实际上，你的任何一个系统调用
都可能被打断。我们现在就提出这一点
是因为你通常是在等待网络，
而网络比进程慢
一个数量级。也就是说被打断的
概率更高。

该怎么处理中断呢？我们来试个简单的例子。

``` objectivec
while bytes_read isn't count {
  bytes_read += read(fd, buf, count);
  if error is EINTR {
    continue;
  } else {
    break;
  }
}
```

我们可以向你保证下面这段代码*会遇到错误*。
你能看出为什么吗？表面上看，
它在读或写之后会重新发起调用。
但当错误是 EINTR 时还会发生什么？
缓冲区里的内容是正确的吗？
你还能发现哪些问题？

## 第 4 层：UDP ^layer-4-udp

UDP 是一种构建在 IPv4 和 IPv6 之上的
无连接协议。
它用起来很简单。确定目的地址和端口，
然后发送你的数据包！
不过，网络不保证这些
报文是否会被送达。网络拥塞时
报文可能会被丢弃。报文
可能被复制或者乱序到达。

UDP 的一个典型使用场景是：
当获取最新数据比收全所有数据
更重要的时候。比如，一个游戏可能
持续发送玩家位置的更新。一个流式视频信号
可能用 UDP 发送画面更新。

### UDP 的属性 ^udp-attributes

- 不可靠——数据报协议。通过 UDP 发送的
  报文在前往目的地的途中可能被丢弃。
  这尤其容易让人困惑，因为
  如果你只在回环设备上测试 —— 也就是
  localhost 或对多数用户来说是 127.0.0.1 —— 那么
  报文很少会丢失，因为根本没有
  网络报文被发送。

- 简单——UDP 协议本应比 TCP 少
  很多花哨的东西。
  也就是说对 TCP 来说有很多可配置参数，
  实现中也有很多边界情况。UDP 是
  发完就不管。

- 无状态／事务型——UDP 协议是无状态的。这让
  协议更简单，也让它能表达
  请求查询或响应查询这类简单
  事务。由于没有三次
  握手，发送 UDP 消息的开销也更小。

- 手工的流量／拥塞控制——你必须手工管理
  流量和拥塞控制，这是一把双刃剑。
  一方面，你对一切都有完全的控制。
  另一方面，TCP 有
  *数十年*的优化，因此基于 UDP 的自定义协议
  只有在你的特定用例中能胜过 TCP 时才值得。

- 组播——这是只有 UDP 才能做到的一件事。
  这意味着
  你可以把一条消息发给某个特定组的、
  连接到某个特定路由器的
  每一个对等端。

完整而详细的描述见原始 RFC
（<a href="#ref-rfc768">[8]</a>）。

虽然看起来在你不想
丢数据的场景中你绝不会用 UDP，但
很多协议是基于 UDP 来通信的
并且需要完整的数据。看看
简单文件传输协议，它只用 UDP 就能
可靠地在链路上传输一个文件。
当然，涉及的配置更多，不过
在 UDP 和 TCP 之间做选择时，
要考虑的不只是上面这些因素。

### UDP 客户端 ^udp-client

UDP 客户端相当多才多艺。下面是一个
把报文发往命令行指定服务器的简单客户端。
注意，这个
客户端发出报文后并不等待确认。它是
发完就不管。下面的示例还用了 `gethostbyname`，
因为某些
旧有功能在搭建客户端时仍然相当好用。

``` objectivec
struct sockaddr_in addr;
memset(&addr, 0, sizeof(addr));
addr.sin_family = AF_INET;
addr.sin_port = htons((uint16_t)port);
struct hostent *serv = gethostbyname(hostname);
```

前面那段代码抓取了一个按主机名
匹配的 `hostent` 条目。
虽然这不可移植，但能把事情办成。
第一步是连接它并让它可复用 ——
与 TCP 套接字一样。请注意
我们传的是 `SOCK_DGRAM` 而不是 `SOCK_STREAM`。

``` objectivec
int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
int optval = 1;
setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof(optval));
```

然后，我们可以把我们的 `hostent` 结构体
复制到 `sockaddr_in` 结构体中。完整定义在
man 手册中都有，所以直接复制过来是安全的。

``` objectivec
memcpy(&addr.sin_addr.s_addr, serv->h_addr, serv->h_length);
```

然后，UDP 还有最后一块有用的东西：
我们可以为接收一个
报文设置超时，而 TCP 不行，因为 UDP 不是
面向连接的。做到这一点的代码片段
如下。

``` objectivec
struct timeval tv;
tv.tv_sec = 0;
tv.tv_usec = SOCKET_TIMEOUT;
setsockopt(sockfd, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));
```

现在，套接字已经连好并可以使用了。
我们可以用 `sendto` 发送一个报文。
我们也应当检查返回值。请注意，
如果报文没有被送达我们不会得到错误，
因为那是 UDP 协议的一部分。
不过对于非法的结构体、
错误的地址等，我们会得到错误码。

``` objectivec
char *to_send = "Hello!"
int send_ret = sendto(sock_fd, // Socket
   to_send, // Data
   strlen(to_send), // Length of data
   0, // Flags
   (struct sockaddr *)&ipaddr, // Address
   sizeof(ipaddr)); // How long the address is
```

上面的代码只是通过 UDP 发送了"Hello"。
它完全不知道报文是否到达、
是否被处理等等。

### UDP 服务器 ^udp-server

有多种函数调用可用于通过 UDP
套接字发送数据。我们会使用较新的 `getaddrinfo`
来帮助搭建套接字结构。
请记住 UDP 是一个简单的、基于报文的
（"数据报"）协议。
两台主机之间没有需要建立的连接。首先，
初始化 hints addrinfo 结构体，请求一个 IPv6 的、
被动的数据报套接字。

``` objectivec
memset(&hints, 0, sizeof(hints));
hints.ai_family = AF_INET6;
hints.ai_socktype =  SOCK_DGRAM;
hints.ai_flags =  AI_PASSIVE;
```

接下来，用 getaddrinfo 指定端口号。
我们不需要
指定主机，因为我们是在创建一个服务器套接字，
而不是在向远程主机发送报文。
小心不要传入 "localhost" 或回环地址的
任何其他同义词。我们最后可能会
被动地监听自己，从而导致 bind 错误。

``` objectivec
getaddrinfo(NULL, "300", &hints, &res);

sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
bind(sockfd, res->ai_addr, res->ai_addrlen);
```

端口号小于 1024，所以该程序需要 `root`
权限。我们也可以指定一个服务名
而不是数字端口值。

到目前为止，这些调用与 TCP 服务器的
很相似。对于一个基于流的服务，
我们会调用 `listen` 并接受连接。
而对我们的 UDP 服务器来说，
程序可以开始等待报文到来了。

``` objectivec
struct sockaddr_storage addr;
int addrlen = sizeof(addr);

// ssize_t recvfrom(int socket, void* buffer, size_t buflen, int flags, struct sockaddr *addr, socklen_t * address_len);

byte_count = recvfrom(sockfd, buf, sizeof(buf), 0, &addr, &addrlen);
```

addr 结构体将保存关于
到达报文发送者（源）的信息。请注意
`sockaddr_storage` 这个类型足够大，
能容纳所有可能的套接字地址类型 —— IPv4、IPv6
或任何其他互联网协议。完整的 UDP 服务器
代码如下。

``` objectivec
#include <string.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
#include <unistd.h>
#include <arpa/inet.h>

int main(int argc, char **argv) {
  struct addrinfo hints, *res;
  memset(&hints, 0, sizeof(hints));
  hints.ai_family = AF_INET6; // INET for IPv4
  hints.ai_socktype =  SOCK_DGRAM;
  hints.ai_flags =  AI_PASSIVE;

  getaddrinfo(NULL, "300", &hints, &res);

  int sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);

  if (bind(sockfd, res->ai_addr, res->ai_addrlen) != 0) {
    perror("bind()");
    exit(1);
  }
  struct sockaddr_storage addr;
  int addrlen = sizeof(addr);

  while(1){
    char buf[1024];
    ssize_t byte_count = recvfrom(sockfd, buf, sizeof(buf), 0, &addr, &addrlen);
    buf[byte_count] = '\0';

    printf("Read %d chars\n", byte_count);
    printf("===\n");
    printf("%s\n", buf);
  }

  return 0;
}
```

请注意，如果你对一个报文只做了部分读取，
该报文的其余数据就被丢弃了。
一次 recvfrom 调用就是一个报文。
为确保空间足够，请用 64 KiB 作为
存储空间。

## 第 7 层：HTTP ^layer-7-http

OSI 模型的第 7 层处理应用层的接口。
也就是说你可以忽略这一层以下的一切，
把互联网当成一种与另一台计算机
通信的方式；那种通信可以是安全的，
会话也可能重连。常见的第 7 层协议
有下面这些：

1.  HTTP(S) - 超文本传输协议。发送任意数据，
   并在 Web 服务器上执行远程动作。
   其中的 S 代表安全，
   即 TCP 连接使用 TLS 协议来确保
   通信不会被旁观者轻易读懂。

2.  FTP - 文件传输协议。把一个文件
   从一台计算机传输到另一台。

3.  TFTP - 简单文件传输协议。与上面相同，
   但使用 UDP。

4.  DNS - 域名系统。把主机名翻译成 IP 地址。

5.  SMTP - 简单邮件传输协议。允许
   把纯文本邮件发送到邮件服务器。

6.  SSH - 安全 Shell。允许一台计算机连接到
   另一台计算机并远程执行命令。

7.  Bitcoin - 去中心化的加密货币。

8.  BitTorrent - 点对点文件共享协议。

9.  NTP - 网络时间协议。这个协议帮助你
   保持计算机时钟与外部世界同步。

### 我叫什么名字？ ^whats-my-name

还记得我们之前谈论过
把网站转换成 IP 地址吗？这里用到一个
叫 "DNS"（域名系统）的系统。如果某个 IP
地址不在机器的缓存中，它就会向
本地 DNS 服务器发送一个 UDP 报文。
这台服务器可能会去查询其他上游 DNS 服务器。

DNS 本身速度快但不安全。DNS 请求是未加密的，
容易受到"中间人"攻击。比如，一家咖啡馆的
网络连接很容易劫持你的 DNS 请求，
为某个域名返回不同的 IP 地址。常见的
劫持方式是：拿到 IP 地址之后通常
再通过 HTTPS 建立连接。HTTPS 使用
所谓的 TLS（以前叫 SSL）来
保护传输，并验证该主机名
被某个证书颁发机构所认可。证书颁发机构
常常被入侵，所以不要轻易把一把绿色的锁
等同于安全。即便加上这一层安全性，
美国政府自 2008 年起就要求其
联邦机构部署 DNSSEC；DNSSEC 包含额外的、
以安全为中心的技术，以极高概率
验证某个 IP 地址确实与某个主机名相关联。

扯远一点，DNS 简而言之是这样运作的：

1.  向你的 DNS 服务器发送一个 UDP 报文

2.  如果那台 DNS 服务器缓存了
   这个报文，就直接返回结果

3.  如果没有，就向更高层的 DNS 服务器
   询问答案。缓存并返回
   该结果

4.  如果任一报文在猜测的超时时间内
   没有得到回应，就重新发送
   该请求。

如果你想了解全部细节，可以随便去看维基百科
上的那一页。本质上，DNS 服务器
是有层次结构的。首先是
点号层次结构。这个层次结构先解析顶级域名
`.edu`、`.gov` 等。然后解析下一层，
也就是 `illinois.edu`。
接着本地解析器可以解析任意多个 URL。比如，
Illinois 的 DNS 服务器同时处理 `cs.illinois.edu` 和
`cs341.cs.illinois.edu`。你的子域名数量
是有限制的，但人们常用它把请求路由到
不同的服务器，从而不必购买许多
高性能服务器来承担请求路由。

## 非阻塞 IO ^non-blocking-io

当你调用 `read()` 而数据尚不可用时，它会一直等待
直到数据就绪之后该函数才返回。当你在
从磁盘读数据时，这个延迟很短；但当你在
从一个缓慢的网络连接读取时，请求会花上
很长时间。而且数据可能永远不会
到来，从而导致意外的关闭。

POSIX 允许你在一个文件描述符上设置一个标志，
使得对该文件描述符的任何 `read()`
调用都会立即返回，无论它是否
已经完成。当你的文件描述符处于这种模式时，
你对 `read()` 的调用会启动读取操作，
而在它工作期间你可以去做别的有用的事情。
这被称为"非阻塞"模式，因为对
`read()` 的调用不会阻塞。

要把一个文件描述符设为非阻塞。

``` objectivec
// fd is my file descriptor
int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);
```

对于套接字，你可以把
`SOCK_NONBLOCK` 加到 `socket()` 的第二个参数中，以非阻塞模式创建它：

``` objectivec
fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
```

当一个文件处于非阻塞模式而你调用 `read()` 时，它会
立即返回可用的那些字节。假设有 100 个字节
已经从套接字另一端的服务器到达，而你调用了
`read(fd, buf, 150)`。‘read’ 会立即返回一个
100 的值，意思是你要求读取的 150 个字节中
只读到了 100 个。假设你又用 `read(fd, buf+100, 50)`
试图读取剩下的数据，但
最后那 50 个字节还没到达。`read()` 会返回 -1
并把全局错误变量 **errno** 设为 `EAGAIN` 或
`EWOULDBLOCK`。这就是系统告诉你数据
尚未就绪的方式。

`write()` 在非阻塞模式下同样有效。假设你想
通过套接字向远程服务器发送 40,000 个字节。
系统一次只能发送有限
数量的字节。在非阻塞模式下，`write(fd, buf, 40000)`
会返回它能立即发送的字节数，
大约是 23,000。如果你紧接着再次调用
`write()`，它会返回 -1
并把 errno 设为 `EAGAIN` 或 `EWOULDBLOCK`。这就是系统
告诉你它仍在忙着发送最后一块数据、
暂时还没准备好发送更多的方式。

有几种方法可以检查你的 IO 是否已经到达。
我们来看看如何用 *select* 和 *epoll* 做到这一点。
我们拥有的第一个接口是 select。
在 POSIX 社区中，如果很多
人有它的替代方案，他们就不太愿意用 select，
而在大多数情况下确实存在替代方案。

``` objectivec
int select(int nfds,
fd_set *readfds,
fd_set *writefds,
fd_set *exceptfds,
struct timeval *timeout);
```

给定三组文件描述符，`select()` 会等待其中
任意一个文件描述符变为"就绪"。

1.  `readfds` —— `readfds` 中的一个文件描述符在有
    可读数据或已到达 EOF 时就绪。

2.  `writefds` —— `writefds` 中的一个文件描述符在
    对 write() 的调用会成功时就绪。

3.  `exceptfds` —— 与系统相关，没有良好定义。
    这一项直接传 NULL 就行。

`select()` 返回就绪文件描述符的总数。如果在
*timeout* 规定的时间内它们
一个都没就绪，它就会返回 0。`select()` 返回之后，
调用者需要
遍历 readfds 和／或 writefds 中的文件描述符，
看看哪些是
就绪的。由于 readfds 和 writefds
同时充当输入和输出参数，
当 `select()` 指示存在就绪的文件描述符时，
它已经把它们覆盖为只反映就绪的那些
文件描述符。
除非调用者只打算调用 `select()` 一次，
否则在调用之前保存一份 readfds 和 writefds
的副本是个好主意。这里
是一个完整的示例代码片段。

``` objectivec
fd_set readfds, writefds;
FD_ZERO(&readfds);
FD_ZERO(&writefds);
for (int i=0; i < read_fd_count; i++)
FD_SET(my_read_fds[i], &readfds);
for (int i=0; i < write_fd_count; i++)
FD_SET(my_write_fds[i], &writefds);

struct timeval timeout;
timeout.tv_sec = 3;
timeout.tv_usec = 0;

int num_ready = select(FD_SETSIZE, &readfds, &writefds, NULL, &timeout);

if (num_ready < 0) {
  perror("error in select()");
} else if (num_ready == 0) {
  printf("timeout\n");
} else {
  for (int i=0; i < read_fd_count; i++)
  if (FD_ISSET(my_read_fds[i], &readfds))
  printf("fd %d is ready for reading\n", my_read_fds[i]);
  for (int i=0; i < write_fd_count; i++)
  if (FD_ISSET(my_write_fds[i], &writefds))
  printf("fd %d is ready for writing\n", my_write_fds[i]);
}
```

<a href="http://pubs.opengroup.org/onlinepubs/9699919799/functions/select.html">http://pubs.opengroup.org/onlinepubs/9699919799/functions/select.html</a>
select 的问题，以及为什么很多人不用它或 poll，
在于 select 必须线性地
遍历每一个对象。如果在
遍历对象的过程中，之前的对象
改变了状态，select 就必须
重新开始。如果我们每个集合中
都有大量文件描述符，这效率极低。
还有一个替代方案，
但也好不到哪里去。

### epoll ^epoll

`epoll` 不属于 POSIX，但 Linux 支持它。
它是一种等待众多文件描述符的
更高效方式。它会准确告诉你
哪些描述符已经就绪。它甚至还
让你能为每个描述符存一小段数据，
比如一个数组下标或一个
指针，从而更容易访问与该
描述符相关联的数据。

首先，你必须用 <a href="http://linux.die.net/man/2/epoll_create">http://linux.die.net/man/2/epoll_create</a>
创建一个特殊的文件描述符。
你不会读写这个文件描述符。你会把它
传给其他 epoll_xxx 函数，并在最后对它调用 close()。

``` objectivec
int epfd = epoll_create(1);
```

对于每个你想用 epoll 监视的文件描述符，
你都需要用 <a href="http://linux.die.net/man/2/epoll_ctl">http://linux.die.net/man/2/epoll_ctl</a>
配合 `EPOLL_CTL_ADD` 选项把它加入 epoll 的数据结构中。
你可以往里加任意多个文件
描述符。

``` objectivec
struct epoll_event event;
event.events = EPOLLOUT;  // EPOLLIN==read, EPOLLOUT==write
event.data.ptr = mypointer;
epoll_ctl(epfd, EPOLL_CTL_ADD, mypointer->fd, &event)
```

要等待其中一些文件描述符变为就绪，
请使用 <a href="http://linux.die.net/man/2/epoll_wait">http://linux.die.net/man/2/epoll_wait</a>。
它所填写的 epoll_event 结构体
会包含你在加入这个文件描述符时
放在 event.data 中的数据。这
让你很容易查到与该文件描述符
相关联的数据。

``` objectivec
int num_ready = epoll_wait(epfd, &event, 1, timeout_milliseconds);
if (num_ready > 0) {
  MyData *mypointer = (MyData*) event.data.ptr;
  printf("ready to write on %d\n", mypointer->fd);
}
```

假设你原本在等待向一个文件描述符写数据，
但现在你想等待从它读数据。
只需用 `epoll_ctl()` 配合
`EPOLL_CTL_MOD` 选项来改变你正在
监视的操作类型。

``` objectivec
event.events = EPOLLOUT;
event.data.ptr = mypointer;
epoll_ctl(epfd, EPOLL_CTL_MOD, mypointer->fd, &event);
```

要把一个文件描述符从 epoll 中取消订阅，
同时让其他描述符保持
活跃，请用 `epoll_ctl()` 配合 `EPOLL_CTL_DEL` 选项。

``` objectivec
epoll_ctl(epfd, EPOLL_CTL_DEL, mypointer->fd, NULL);
```

要关闭一个 epoll 实例，关掉它的文件描述符即可。

``` objectivec
close(epfd);
```

除了非阻塞的 `read()` 和 `write()` 之外，
在非阻塞套接字上对 `connect()` 的
任何调用也会是非阻塞的。要等待
连接完成，请用 `select()` 或 epoll 等待
该套接字变为可写。偏好 epoll 而不是
select 是有充分理由的，因为 select 的接口
存在根本性问题。它只能
监视编号低于 `FD_SETSIZE`（通常是 1024）的描述符，
而且每次
调用的耗时与被监视的描述符数量成正比。

<a href="https://idea.popcount.org/2017-01-06-select-is-fundamentally-broken/">https://idea.popcount.org/2017-01-06-select-is-fundamentally-broken/</a>

### Epoll 示例 ^epoll-example

我们来拆解一下 man 手册中的 epoll 代码。
我们假设已经准备好了一个
TCP 服务器套接字 `int listen_sock`。我们必须做的
第一件事是创建 epoll 设备。

``` objectivec
epollfd = epoll_create1(0);
if (epollfd == -1) {
  perror("epoll_create1");
  exit(EXIT_FAILURE);
}
```

下一步是以电平触发模式加入监听套接字。

``` objectivec
// This file object will be `read` from (connect is technically a read operation)
ev.events = EPOLLIN;
ev.data.fd = listen_sock;

// Add the socket in with all the other fds. Everything is a file descriptor
if (epoll_ctl(epollfd, EPOLL_CTL_ADD, listen_sock, &ev) == -1) {
  perror("epoll_ctl: listen_sock");
  exit(EXIT_FAILURE);
}
```

然后在一个循环中，我们等待并查看 epoll 是否有事件。

``` objectivec
struct epoll_event ev, events[MAX_EVENTS];
nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
if (nfds == -1) {
  perror("epoll_wait");
  exit(EXIT_FAILURE);
}
```

如果我们在某个客户端套接字上收到一个事件，
那意味着该客户端有
数据可以读取，我们就执行该操作。否则，
我们需要用一个新客户端来更新
我们的 epoll 结构。

``` objectivec
if (events[n].data.fd == listen_sock) {
  int conn_sock = accept(listen_sock, (struct sockaddr *) &addr, &addrlen);
  // Must set to non-blocking
  setnonblocking(conn_sock);

  // We will read from this file, and we only want to return once
  // we have something to read from. We don't want to keep getting
  // reminded if there is still data left (edge triggered)
  ev.events = EPOLLIN | EPOLLET;
  ev.data.fd = conn_sock;
  epoll_ctl(epollfd, EPOLL_CTL_ADD, conn_sock, &ev)
}
```

上面的函数为了简洁还省略了一些错误检查。
请注意，这段代码性能不错，
因为我们是以电平触发模式加入了服务器套接字，
而我们把每个客户端文件描述符以
边沿触发方式加入。边沿触发模式
把更多的计算留给了应用程序
—— 应用程序必须持续读或写，
直到该文件描述符没有字节可读可写 —— 但它
避免了饥饿。更高效的实现
还会以边沿触发方式加入监听套接字，
以便同样清理掉积压的连接。

在开始编程之前，请通读 `man 7 epoll` 的
大部分内容。
里面有很多坑。下面会详细
介绍其中比较常见的一些。

### Epoll 的各种坑 ^assorted-epoll-gotchas

使用 epoll 有若干问题。这里我们详细
说明其中几个。

1.  有两种模式。电平触发和边沿触发。
    电平触发是指只要该文件描述符上有事件，
    每次调用 `epoll_wait` 都会返回它。
    在边沿触发下，只有当它
    从零事件变为有事件时，调用者
    才会得到该文件描述符。这意味着，
    如果你忘了在文件描述符上
    读、写、accept 等，
    直到拿到 EWOULDBLOCK，那么这个
    文件描述符就会被丢掉。

2.  如果你在任何时候复制了一个文件描述符
    并把它加入 epoll，那么你会
    从该文件描述符和复制出来的那个
    各自收到一个事件。

3.  你可以把一个 epoll 对象加入 epoll。

4.  视情况而定，你可能会从 Epoll 收到一个
    已被关闭的文件描述符。这不是 bug。
    之所以会这样，是因为
    epoll 工作在内核对象层面，
    而不是文件描述符层面。
    如果内核对象存活更久且设置了正确的标志，
    一个进程就可能拿到一个已关闭的文件描述符。
    这也意味着
    一旦你关闭了文件描述符，
    就没有办法移除那个内核对象。

5.  Epoll 有 `EPOLLONESHOT` 标志，它会在一个文件
    描述符被 `epoll_wait` 返回之后将其移除。

6.  Epoll 使用电平触发模式可能让某些文件
    描述符挨饿，因为无法预知应用程序
    会从每个描述符中读取多少数据。

在 `man 7 epoll` 处阅读更多内容，或者去看看附录中
一个更好的版本 `kqueue`。

## 远程过程调用 ^remote-procedure-calls

RPC 也就是远程过程调用，其理念是
我们可以在另一台机器上执行一个过程。
实践中，这个过程可能在
同一台机器上执行。不过，它可能
处在不同的上下文中。例如，
该操作以另一个用户、不同的权限和
不同的生命周期来运行。

一个例子是，你可能向某个 docker
守护进程发送一次远程过程调用来改变容器的状态。
并非每个应用都需要
访问整台系统机器，但它们应该能够
访问它们自己创建的容器。

### 权限分离 ^privilege-separation

远程代码将以来自调用者的不同用户身份和
不同权限运行。实践中，
这次远程调用的权限可能比调用者
更多，也可能更少。原则上，
这可以用来提升系统的安全性，
方法是确保各组件都以最小权限运行。
遗憾的是，需要仔细评估安全方面的
顾虑，以确保 RPC 机制不会被
劫持去执行非预期的动作。例如，
某个 RPC 实现可能会隐式地
信任任何已连接的客户端去执行任何动作，
而不是只对数据的某个子集执行某个
动作子集。

### 桩代码与编组 ^stub-code-and-marshaling

桩代码是用来隐藏执行一次
远程过程调用之复杂性的必要代码。
桩代码的职责之一是把
必要的数据*编组（marshal）*成一种
可以作为字节流发送给远程服务器的格式。

``` objectivec
// On the outside, 'getHighScore' looks like a normal function call
// On the inside, the stub code performs all of the work to send and receive data to and from the remote machine.

int getHighScore(char* game) {
  // Marshal the request into a sequence of bytes:
  char* buffer;
  asprintf(&buffer,"getHighScore(%s)!", game);

  // Send down the wire (we do not send the zero byte; the '!' signifies the end of the message)
  // fd is an already-connected socket to the remote server
  write(fd, buffer, strlen(buffer) );
  free(buffer);

  // Wait for the server to send a response
  char response[64];
  ssize_t bytesread = read(fd, response, sizeof(response) - 1);
  if (bytesread < 0) return -1;

  // Example: unmarshal the bytes received back from text into an int
  response[bytesread] = 0; // Turn the result into a C string

  int score = atoi(response);
  return score;
}
```

使用字符串格式可能效率略低。这种编组
的一个好例子是 Golang 的 gRPC 或 Google RPC。
也有一个 C 语言版本，如果感兴趣可以去看看。

服务器端的桩代码会接收请求，把请求
反序列化（unmarshal）成一种合法的
内存表示，调用底层
实现，然后把结果回传给调用者。
很多时候底层库会替你做这些事。

要实现 RPC，你需要决定并记录
你会用哪些约定把数据序列化成字节序列。
即使是一个简单的
整数，也有若干种常见选择。

1.  有符号还是无符号？

2.  ASCII、Unicode 转换格式 8（UTF-8）还是其他某种编码？

3.  固定字节数，还是根据大小可变？

4.  如果用二进制，是小端还是大端格式？

要编组一个结构体，先决定哪些字段需要被
序列化。发送所有数据项可能
并无必要。比如，某些项
可能与特定的 RPC 无关，或者可以由服务器
根据已有的其他数据项重新
计算出来。

要编组一个链表，则没必要发送链指针；
改为把各个值流式发送。作为
反序列化的一部分，服务器可以
从这个字节序列重建一个链表结构。

从头节点／顶点出发，一棵简单的树
可以被递归遍历，从而生成数据的一个
序列化版本。而一张有环的图
通常需要额外的内存，以确保每条边和每个顶点
都恰好被处理一次。

### 接口描述语言 ^interface-description-language

手写桩代码既痛苦、繁琐、又容易出错，
而且难以维护。从已经实现的代码中
逆向推导出线上协议
也很困难。更好的办法是描述那些
数据对象、消息和服务，从而自动生成
客户端和服务器代码。接口描述
语言的一个现代例子是 Google 的
Protocol Buffer .proto 文件。

即便如此，远程过程调用仍然明显更慢
（慢 10 到 100 倍），也比本地调用
更复杂。RPC 必须把数据编组成
一种与线上格式兼容的形式。这可能
需要多次遍历数据结构、
临时内存分配以及对
数据表示的转换。

健壮的 RPC 桩代码必须
智能地处理网络故障和
版本问题。比如，服务器可能必须处理
来自那些仍在运行早期版本桩代码的
客户端的请求。

安全的 RPC 需要实现额外的安全检查，
包括认证和授权、校验数据，以及
加密客户端与主机之间的通信。很多时候，
RPC 系统可以高效地替你做这些。
考虑一下如果你在同一台机器上
同时有 RPC 客户端和服务器会怎样。
启动一个 Thrift 或 Google
RPC 服务器可能会校验并把请求路由到一个
本地套接字上，那个套接字
根本不会经由网络发送。

### 传输结构化数据 ^transferring-structured-data

我们来考察用 3 种不同格式传输数据的
三种方法 —— JSON、XML 和
Google Protocol Buffers。JSON 和 XML 是
基于文本的协议。下面是
JSON 和 XML 消息的示例。

``` xml
<ticket><price currency='dollar'>10</price><vendor>travelocity</vendor></ticket>
```

    { 'currency':'dollar' , 'vendor':'travelocity', 'price':'10' }

Google Protocol Buffers 是一种开源的高效二进制协议，
它非常强调在低 CPU 开销和
极少内存拷贝的前提下实现高吞吐。
这意味着用多种语言编写的客户端和
服务器桩代码可以从 .proto 规范文件
自动生成，以便把数据编组到
二进制流中以及从二进制流中解出。

<a href="https://developers.google.com/protocol-buffers/docs/overview">https://developers.google.com/protocol-buffers/docs/overview</a>
通过忽略消息中出现的未知字段
来缓解版本问题。更多信息请见
Protocol Buffers 的介绍。

总体思路是把实际的业务逻辑以及
各种编组代码抽象掉。如果你的应用
曾经在解析 XML、JSON 或 YAML 时
受限于 CPU，那就换成 protocol buffers 吧！

## 主题 ^topics

- IPv4 与 IPv6

- TCP 与 UDP

- 丢包／基于连接

- 获取地址信息

- DNS

- TCP 客户端调用

- TCP 服务器调用

- shutdown

- recvfrom

- epoll 与 select

- RPC

## 问题 ^questions

- 什么是 IPv4？IPv6？它们之间有什么
  区别？

- 什么是 TCP？UDP？分别说出
  它们的优点和缺点。在什么场景下
  该用一个而不是另一个？

- 哪个协议是无连接的，哪个是基于连接的？

- DNS 是什么？DNS 走的路径是什么？

- socket 是做什么的？

- 建立一个 TCP 客户端需要哪些调用？

- 建立一个 TCP 服务器需要哪些调用？

- 套接字的 shutdown 与关闭有什么区别？

- 什么时候可以用 `read` 和 `write`？那
  `recvfrom` 和 `sendto` 呢？

- `epoll` 相比 `select` 有哪些优势？那
  `select` 相比 `epoll` 呢？

- 什么是远程过程调用？什么时候该用它，
  而不是 HTTP 或在本地运行代码？

- 什么是编组／反编组？为什么 HTTP *不是*一种 RPC？

<div id="refs" class="references csl-bib-body hanging-indent">

<div id="ref-rfc9114" class="csl-entry">

Bishop, Mike. 2022. *HTTP/3*. No. 9114. Request for Comments. RFC 9114;
RFC Editor.
<a href="https://doi.org/10.17487/RFC9114">https://doi.org/10.17487/RFC9114</a>.

</div>

<div id="ref-cohen_1980" class="csl-entry">

Cohen, Danny. 1980. “ON HOLY WARS AND a PLEA FOR PEACE.” In *IETF*.
IETF.
<a href="https://www.ietf.org/rfc/ien/ien137.txt">https://www.ietf.org/rfc/ien/ien137.txt</a>.

</div>

<div id="ref-rfc9110" class="csl-entry">

Fielding, Roy T., Mark Nottingham, and Julian Reschke. 2022. *HTTP
Semantics*. No. 9110. Request for Comments. RFC 9110; RFC Editor.
<a href="https://doi.org/10.17487/RFC9110">https://doi.org/10.17487/RFC9110</a>.

</div>

<div id="ref-google_ipv6_stats" class="csl-entry">

Google. 2026. *IPv6 Adoption Statistics*.
<a href="https://www.google.com/intl/en/ipv6/statistics.html">https://www.google.com/intl/en/ipv6/statistics.html</a>.

</div>

<div id="ref-RFC1700" class="csl-entry">

Reynolds, J., and J. Postel. 1994. *Assigned Numbers*. RFC No. 1700. RFC
Editor; Internet Requests for Comments; RFC Editor.

</div>

<div id="ref-internet_society_2018" class="csl-entry">

“State of IPv6 Deployment 2018.” 2018. In *Internet Society*. Internet
Society.
<a href="https://www.internetsociety.org/resources/2018/state-of-ipv6-deployment-2018/">https://www.internetsociety.org/resources/2018/state-of-ipv6-deployment-2018/</a>.

</div>

<div id="ref-rfc9113" class="csl-entry">

Thomson, Martin, and Cory Benfield. 2022. *HTTP/2*. No. 9113. Request
for Comments. RFC 9113; RFC Editor.
<a href="https://doi.org/10.17487/RFC9113">https://doi.org/10.17487/RFC9113</a>.

</div>

<div id="ref-rfc768" class="csl-entry">

*User Datagram Protocol*. 1980. No. 768. Request for Comments. RFC 768;
RFC Editor.
<a href="https://doi.org/10.17487/RFC0768">https://doi.org/10.17487/RFC0768</a>.

</div>

</div>
