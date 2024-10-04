---
layout: post
title: 从一次HTTP请求看TCP IP协议
categories: network
description: 
keywords: network, TCP, IP, HTTP
---

这篇笔记的目的通过分析一次HTTP请求，梳理常见的TCP/IP协议，温故而知新。 <!--more-->

## 背景知识
### TCP/IP网络模型

在开始之前，我们先介绍一点TCP/IP协议的基础知识。下图是TCP/IP协议的4层模型，分层类似与软件开发中的封装，每层都有自己的功能约定。通过分层，上一层的开发只需要考虑直接依赖的下一层，对于更低层则不需要太关注。一般来说
- 应用层是具体应用的协议，由应用自己定义以实现其功能，比如HTTP, FTP, DNS等
- 传输层负责将数据发送到目标主机。HTTP在传输层使用的是TCP协议，TCP负责将数据可靠的传送到目标主机。此外还有UDP协议，不保证可靠性。
- 网络层负责将单个数据报发送到目的地。与传输层不同，网络层负责的是路由选择、IP寻址等，不涉及传输控制。包括IP协议、ARP协议等。
- 数据链路层和物理层则负责与具体硬件打交道，负责数据在物理链路上的传输。通过路由表，我们得知每个IP包的下一跳是哪个IP，该由当前主机的哪个网络接口发出。这个下一跳的主机，就是链路层的目的地。IP层负责让数据包在从源主机到目标主机，而链路层负责让数据包到达下一站。以单车旅行为例，IP层负责的是从杭州到北京的，而链路层负责的是从杭州到上海，从上海到天津，再从天津到北京这一段段的起点和终点间的旅程。而TCP协议，则是组队出发、失败后再次出发。

发送数据时，用户数据经过应用层、传输层、网络层，最后到链路层和物理层。每层都会将上一次的数据作为payload，同时加上该层的首部。
- 首部是协议约定的控制字段。比如HTTP首部定义了请求方法，host等；TCP首部定义了端口、窗口大小、序号等；IP首部定义了源IP、目标IP等
- payload是业务自定义的数据。协议本身不依赖这些数据，只负责传输它们。对于应用层，payload可能是HTTP request body，FTP的传输文件内容，DNS查询结果；对于传输层及以下，payload是上一层的header+payload。


![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20240913220214988.png)



### HTTP协议及请求过程

打开Chrome -> 开发者工具 -> Network，再访问常见的网页，比如zoom.us, 就可以看到一个HTTP请求及其回复。

要从本机（浏览器）发送一个HTTP请求，首先要确定，这个请求发给哪台主机。这里我们使用域名，请求过程中会通过DNS服务将域名换成zoom.us某台目标服务器的IP地址，这就是这次通信的目标主机，也就是下面截图中的Remote Address。

![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20240909214444008.png)

这台主机上可能有多个应用程序，不同应用之间通过不同的端口收发数据。IP层的地址是IP，TCP层的地址其实就是端口。我们要发送给提供HTTP(S)服务的应用，默认是443端口。

HTTP请求报文一般包括以下要素
- Method：如POST/GET/DELETE/PUT
- Path: 指定要访问的资源，接口
- Http版本号，目前大多数网站使用的是Http/1.1
- 请求header
	- 包括一些业务自定义的header，如截图中的:authority, :method
	- 一些约定的header, 如Cookie, Accept等。指定了Client接受的内容格式、编码、语言、压缩算法。
- Body：一些请求参数


HTTP协议格式如下图所示

![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20240911232041901.png)

请求报文示例如下

```
POST /contact_form.php HTTP/1.1
Host: developer.mozilla.org
Content-Length: 64
Content-Type: application/x-www-form-urlencoded

name=Joe%20User&request=Send%20me%20one%20of%20your%20catalogue
```


请求报文经过编码后变成字节流，分段后交由TCP层传输到目标主机。

目前大部分网站都使用HTTPS协议，它其实是在HTTP的基础上，在传输层使用TLS加密，来达到安全通信的目标。发送HTTPS请求时首先要进行TLS握手。TLS握手的流程，简单说，先利用证书进行非对称加密的通信，协商一个对称加密的秘钥，后续通信使用对称加密的秘钥。这样做可以兼顾非对称加密的安全性和对称加密的高性能。TLS握手完成后，会生成会话ID、会话ticket等，后续HTTPS请求可以通过这些信息直接使用上次的对称加密秘钥，不会每次请求前都重新握手、太浪费性能。关TLS握手的细节可以参考 [TLS 1.0 至 1.3 握手流程详解](https://www.cnblogs.com/enoc/p/tls-handshake.html "发布于 2022-03-22 18:24")

![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20240913224224758.png)


![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20240913222831147.png)


### TCP协议 (传输层)

TCP协议处于传输层，负责将**一批**数据可靠的传送到目标主机。TCP协议为了**保证可靠**，每个请求都需要接收方的**确认**。如果超时没收到确认，发送端需要重发。

这引出一个问题，如果每次都需要等上一个数据包被确认，才发出下一个数据包，等待时间太长、没有充分利用网络带宽 => RTT越长，性能越差。为了避免这个问题，TCP协议引入了**窗口**概念。窗口内，不需要等上一个数据包被确认，就可以发出下一个数据包，这样可以提高并发、减少等待时间。

但窗口带来两个新的问题
1. 并发发送，可能包会乱序。
2. 窗口大小如何确定。窗口太小，就不能充分利用网络带宽；窗口太大，可能加剧网络拥堵，大量丢包，反而让性能下降。

针对问题1，需要对每个数据包进行**编号**。针对问题2，TCP协议通过窗口控制和拥塞控制解决。

TCP的首部信息如下所示，对应上述提到的编号、确认、窗口等概念。
![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20240912235640045.png)





### IP协议(网络层)

TCP的数据加上IP首部后，经路由选择、转发到达目标主机。这一过程在网络层，参与的协议主要包括IP协议，ARP协议。


IPv4和Ipv6首部格式如下
![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20240913000244793.png)
![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20240913000310967.png)

源地址和目标地址定义了这个IP包从哪来，到哪去。IP包在网络上由经过的每个主机上的路由表，确定这个IP包的下一站是哪个主机、需要经当前主机的哪个网络接口到达。主机上的路由表由RIP、OSPF、BGP等路由协议维护更新。


谈到网络层，就不得不提NAT，由于IPv6尚未普及、IP地址不够，我们的网络设备（电脑，手机，手表）所分配到的只是一个本地网络地址，使用互联网时往往公用一个全局IP地址，也就是常说的公网IP、出口IP。这里利用的就是NAT技术。即使IPv6普及，每个网络设备都能有一个全局IP，处于网络管理的角度考虑，组织内往往还会使用NAT。

NAT简单说就是有这样一个设备，它叫NAT路由器，由内部网络发往互联网IP包，其源地址会被替换为NAT路由器的全局IP，TCP首部的源端口会被替换为NAT路由器的某一空闲端口（可能时随机分配的，也可能是按某一策略分别的）。NAT路由器会缓存下这个空闲端口对应的内部网络的原始源地址和端口。

外部互联网收到IP包后，看到的源地址和端口都是NAT路由器的，其回复会发往NAT路由器，回复的IP包目标IP地址就是NAT路由器的全局IP地址，其目标端口是NAT路由器的空闲端口。NAT路由器会根据该目标端口从缓存中拿到内部网络的原始源地址和端口，并进行替换。替换后的IP包，能正确抵达内部网络的主机。

在这一个过程中，NAT对通信双方是透明的，通过NAT，内部网络的主机即使没有全局地址也能与互联网进行通信。但需要注意的是，这种通信只能有内部网络主动发起。互联网上的主机无法主动与内部网络的主机进行通信，因为内部网络的主机是没有**全局地址**的，除非NAT路由器已经提前将内部主机的IP和NAT的空闲端口映射、并将该映射信息同步到需要发起请求的外网主机。

![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20240917162558331.png)


网络层常涉及的协议还有
- ICMP：辅助IP协议，进行网络诊断，如IP包是否能到达目标主机、失败原因。我们常用的Ping命令就是基于该协议。
- DHCP：动态IP地址分配
- ARP：由IP地址，获取链路层MAC地址


### 链路层

通过路由表，我们得知每个IP包的下一跳是哪个IP，该由当前主机的哪个网络接口发出。这个下一跳的主机，就是链路层的目的地。IP层负责让数据包在从源主机到目标主机，而链路层负责让数据包到达下一站。以单车旅行为例，IP层负责的是从杭州到北京的，而链路层负责的是从杭州到上海，从上海到天津，再从天津到北京这一段段的起点和终点间的旅程。而TCP协议，则是组队出发、失败后再次出发。

链路层使用MAC地址作为目的地。为什么我们不能使用MAC地址作为互联网的通信地址呢，原因是MAC地址是有硬件厂家设置的、销往各地，不能像普通地址一样是分层的、每一层能用于寻址。IP地址类似我们的邮递地址，“浙江省杭州市XX区”，地址的每一部分都在逐级的缩小范围、都能用于寻址，并最终定位到目的地。

下一跳主机的IP地址到MAC地址的转换，由ARP协议完成。目标主机收到后，由IP层确定是不是最终目的地，是，则再交由上层处理；不是，则根据路由表确定下一跳的目的地，再继续互联网的旅程，直到达到最终目的地。

### 应用处理
目标主机收到数据后，层层解包，去除首部，数据最终从链路层抵达应用层，由提供HTTP服务的应用收到请求报文，进行处理，产生回复数据。回复数据包的目标IP是请求数据包的源IP，目标端口是源端口（该端口其实就是请求主机上浏览器在监听的端口，一般由系统随机分配）。经过类似的流程，回复数据会抵达请求主机的浏览器应用程序，并向用户呈现内容（参考第一张流程图）。


### Cisco Packet Tracer

Cisco Packet Tracer是一个网络仿真软件，非常易用，方便初学者进行网络系统的学习、设计和实验。更多信息请参考
- [Networking Academy Cisco Packet Tracer](https://www.netacad.com/cisco-packet-tracer)
- [Cisco Packet Tracer Full Course (EXPLAINED)](https://www.youtube.com/@digidev7060)


## 跟踪一个HTTP请求

我们在PT(Packet Tracer)中构建以下拓扑结构的网络。

![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20241003205931794.png)

其中
- PC0: 测试用client
- test.srv既是DNS server，也是DHCP server，也是HTTP server

几个重要的设备网络配置如下

![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20241004120504866.png)
![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20241004122724065.png)
![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20241004122826414.png)





打开simulation, event filter中只保留我们关注的event类型、避免干扰，然后打开PC0的web browser，访问test.srv, 如下所示

![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20241003210707519.png)
![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20241003210732615.png)




接下来我们分析下这次请求过程。

### DNS查询
PC0拿到域名test.srv后,首先会通过DNS协议向DNS sever查找该域名的IP。

由于DNS sever的IP（209.165.200.225）与PC0不在同一子网，DNS查询请求的下一跳地址被设为default gateway (也即网路拓扑中的router)的IP，192.168.0.1。如果PC0还没有记录router的MAC地址，会通过ARP协议查询router的MAC地址，收到router的回复后，会记录router的MAC地址，并将DNS查询请求在链路层的目标MAC地址为router的MAC地址。

接着PC0发出DNS查询请求，router收到DNS查询请求。数据包的目标IP地址是209.165.200.225，router查询路由表发现DNS server是与router直接相连的，因此数据包的下一跳地址被设为DNS sever的IP（209.165.200.225）。此时，路由器会做两件事
- 通过ARP协议查询DNS sever的MAC地址
- 地址转换，并将将数据包的源IP和源端口替换成路由器的外网IP和某一分配的端口，并将内网机器PC0的IP地址与该端口的映射信息维护在NAT表。这个操作是由于数据包将从内网发到外网，路由器要进行NAT的工作。

完成后，DNS查询请求将由路由器发往DNS server, server收到查询请求后会回复该域名对应的IP地址，也即209.165.200.225

![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20241004002514927.png)


### 建立TCP连接

PC0收到DNS server回复后，即知道test.srv的IP地址是209.165.200.225，即可请求server建立TCP连接, 这就是经典的三次握手过程。这个过程在网上有详细的阐述，这里引用 [网上的材料](https://www.9tut.com/tcp-and-udp-tutorial)


![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20241004003742712.png)

> [!NOTE] TCP three-way handshake
> 1. First host A will send a **SYN** **message** (a TCP segment with SYN flag set to 1, SYN is short for SYNchronize) to indicate it wants to setup a connection with host B. This message includes a sequence (SEQ) number for tracking purpose. This sequence number can be any 32-bit number (range from 0 to 232) so we use “x” to represent it.
>2. After receiving SYN message from host A, host B replies with **SYN-ACK message** (some books may call it “SYN/ACK” or “SYN, ACK” message. ACK is short for ACKnowledge). This message includes a SYN sequence number and an ACK number:  
> + SYN sequence number (let’s called it “y”) is a random number and does not have any relationship with Host A’s SYN SEQ number.  
> + ACK number is the next number of Host A’s SYN sequence number it received, so we represent it with “x+1”. It means “I received your part. Now send me the next part (x + 1)”.
>The SYN-ACK message indicates host B accepts to talk to host A (via ACK part). And ask if host A still wants to talk to it as well (via SYN part).
>3. After Host A received the SYN-ACK message from host B, it sends an **ACK message** with ACK number “y+1” to host B. This confirms host A still wants to talk to host B.

### 发起HTTP请求

建立TCP连接后，PC0就会发送HTTP请求到test.srv，Src Port 1025是PC0分配给web browser的端口，web browser通过监听这个端口接收HTTP响应。

HTTP请求
```
HTTP Data:Accept-Language: en-us  
Accept: */*  
Connection: close  
Host: test.srv
```


HTTP请求各层信息如下
![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20241004102317567.png)

该HTTP请求路径为PC0 -> router -> cable modem0 -> cloud0 -> test.srv，路由寻址类似前面DNS查询，主要包括路由查询、内外网地址转换，不再赘述。

### 响应HTTP请求
test.srv收到HTTP请求后，数据经过物理层、链路层、网络层和传输层，在传输层由TCP组装所有数据（所有数据是指请求数据在传输过程中可能被分片，单个数据包可能只包括一个片段的数据），发送到应用层，端口为80。HTTP server监听80端口，收到HTTP请求后，会准备响应数据，并将其发送回client (PC0)。
对于HTTP响应数据包，其
- 目标IP：即为请求数据包的源IP，也是路由器的外网地址
- 目标端口：即为请求数据包的源端口，也是路由器在地址转换时分配的端口，1025（NAT）

![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20241004110440676.png)

```
HTTP Data:Connection: close  
Content-Length: 369  
Content-Type: text/html  
Server: PT-Server/5.2

...
```

HTTP响应路径为 test.srv->cloud0->cable modem0-> router->PC0，该过程和发送HTTP请求到server类似，不再赘述。

PC0收到响应时数据包的IP和端口已经转换为内网地址和端由，由209.165.200.226:1025到192.168.0.101:1025 （这里转换前后端口没变仅仅是测试环境下的巧合）。

PC0的web browser监听1025端口，接收到HTTP响应后，显示页面。
![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20241004120018946.png)


### 关闭TCP连接

由于测试环境没有启用keep-alive, client (PC0)收到HTTP response后会断开TCP连接，这一过程就是常说的四次挥手。这个过程在网上有详细的阐述，这里引用 [网上的材料](https://www.9tut.com/tcp-and-udp-tutorial)

![](/assets/images/2024-09-09-从一次HTTP请求看TCP%20IP协议/image-20241004115520619.png)


> [!NOTE] TCP four-way termination
> Suppose Host A wants to end the connection to host B, Host A will send a FIN message (a TCP segment with FIN flag set to 1), FIN is short for FINISH. The purpose of FIN message is to enable TCP to gracefully terminate an established connection. Host A then enters a state called the FIN-WAIT state. In FIN-WAIT state, Host A continues to receive TCP segments from Host B and proceed the segments already in the queue, but Host A will not send any additional data.
> 
>Device B will confirm it has received the FIN message with an ACK (with sequence x+1). From this point, Host B will no longer accept data from Host A. Host B can continue sending data to Host A. If Host B does not have any more data to send, it will also terminate the connection by sending a FIN message. Host A will then ACK that segment and terminate the connection.


## 结语

到此位置，我们简单的回顾了下TCP/IP网络的基础知识，并在Packet Tracer中模拟了1次HTTP请求，追踪了这一过程中各个网络层协议、各个设备间的交互。