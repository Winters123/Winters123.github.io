---
layout: post
title:  "Paper Thoughts of PhD"
categories: Reading Memo
tags:  papers SIGCOMM NSDI DCN
author: Xiangrui Yang
---

* content
{:toc}


### Memo of PhD Reading List


#### README

> This memo is to record the insight thoughts of paper that I have read, in case there will be occasions that I need to remind myself of the core idea of these papers (mostly presented in Sigcomm and NSDI, maybe a few papers in CoNext). 
>
> I'll write some quite brief words about the core ideas as well as my opinions about these papers. There is no strict format for this document and most of them will be short. I hope this memo will be a good assistant of mine and also, this will improve my English writing skills for the future career. And, if this memo becomes something that people can refer to when they just want to get the basic ideas of papers without spending too much time on them, I may upload this memo on the Internet to help other young students. 
>
> Ok, let's start!  

### mOS: a reusable network stack for flow monitoring middleboxes (NSDI 2017)

**Key innovations**

1. novel and highly decoupled APIs exposed to middlebox developers, which will enable the development of  vary kinds of middleboxes with far less code.

   > ```c
   > 1 /* an example of mOS APIs */
   > 2 static void mOSAppInit(mctx_t m)
   > 3 {
   > 4 monitor_filter_t ft = {0};
   > 5 int s; event_t hev;
   > 6
   > 7 // creates a passive monitoring socket with its scope
   > 8 s = mtcp_socket(m, AF_INET, MOS_SOCK_MONITOR_STREAM, 0);
   > 9 ft.stream_syn_filter = "dst net 216.58 and dst port 80";
   > 10 mtcp_bind_monitor_filter(m, s, &ft);
   > 11
   > 12 // sets up an event handler for MOS_ON_REXMIT
   > 13 mtcp_register_callback(m,s,MOS_ON_REXMIT,MOS_HK_RCV,OnRexmitPkt);
   > 14
   > 15 // defines a user-defined event that detects an HTTP request
   > 16 hev = mtcp_define_event(MOS_ON_CONN_NEW_DATA, IsHTTPRequest, NULL);
   > 17
   > 18 // sets up an event handler for hev
   > 19 mtcp_register_callback(m, s, hev, MOS_HK_RCV, OnHTTPRequest);
   > 20 }
   > ```
   >
   > And this will probably encourage code reuse.  

2. High-performance packet I/O to support real time middleboxes.

3. Flexible event composition mechanism that enables user-defined events for different uses.  **highly usable**


**Points need further consideration**

1. According to figure5, **flow events can both be triggered from Sender tcp stack and receiver tcp stack**. But how we specify the location where a user defined event should be registered?

2. the difference between i-forest and d-forest is that **only events with events_handlers with be added into i-forest**.  Thus, I guess the i-forest is used to trace the callback functions while in the process.

   > In bro, this challenge is resolved by using a general protocol analysis tree (using tcp protocol as the root) for all the flows. However, the author thought that this will bring lots of work load if users want to add novel user-defined events.

3. probably one of the main differences between **bro** and **mOS** is the **event generation** scheme. 

   > 1. bro uses a general analysis tree for overall event generation. In this case, the large analysis tree will analyze all the protocols all the way from tcp to application levels. During the analyze, all the events that users concerned will be added to the queue.
   > 2. mOS instantiates lots of **invocation forests** to analyze packets and flow of different sockets. Meanwhile, sockets that should be processed with the same logic share the same **invocation forest** in order to reduce memory cost in high-performance networks. This will also enable parallel process in multi-core systems.

4. mOS predefines the execution orders of all the event handlers. And all derived events inherit the priority of the built-in events. This is quite easy to understand.

5. **One of the key problem this paper solves is how to find the same i-forest efficiently.**

6. mOS leverages 

**Tips:**

1. To understand many stateful schemes in networking, the details of TCP is highly important. [Here](http://www.cppblog.com/weiym/archive/2014/05/29/207145.aspx) is a useful website to get a comprehensive understanding of TCP mechanisms.

### A Case for Stateful Middlebox Networking Stack [SIGCOMM Demo 2015]

**Key notes**

This work is presented 2 years (1.5 years to be accurate) before mOS. From the introduction, I believe that the expression of background and the motivation are quite similar to mOS. **The main challenge for developing middleboxes is the lack of a universal stateful middlebox networking stack which exposes a nice network abstraction layer such as socket.**  

### PortLand  [SIGCOMM 2009]

**Key Observation**

In the year before 2009, data center networks were facing the challenge of scalable, manageable, fault-tolerant and efficient features. The main limitations maybe inherent for Ethernet/IP style protocols. To be specific:

***Note***

> ***I guess the best contribution of PortLand is the description of state-of-art architecture of data center networks.***

**The tradeoffs of L2 network and L3 network**: 

1. L3 networks:

   > **Pros**
   >
   > By careful assigning prefixes to networks, the scale of forwarding tables can be greatly reduced. (Because the router uses the longest-prefix-match for routing)
   >
   > By leveraging the TTL provided by IP protocol, the loop is less of an issue.  
   >
   > By using intra-domain routing protocols (OSPF, ISIS), the link and device failure could be detect by these protocols automatically.
   >
   > **Cons**
   >
   > Much administrative burdens.
   >
   > Using DHCP & configured switch subnet identifier can lead to unreachable hosts and difficult to diagnose errors.
   >
   > Less support for host virtualization.

2. L2 networks:

   >**Pros**
   >
   >Doesn't need to configure before deployed
   >
   >Easy for VM migration (Don't have to change IP address)
   >
   >**Cons**
   >
   >a lack of scalability (commodity switch couldn't support hundreds of thousands of forwarding table entries)

***The philosophy behind PortLand***

In Portland, the authors don't want to change anything on the server side. Instead, the switch (the leaf switch) will take the responsibility of 1) ARP protocol (by the help of a centralized controller). 2) As for virtualization supporting, Portland uses MAC-in-MAC scheme to avoid IP change in VM.





### NetPilot [SIGCOMM 2012]

### Reactor [NSDI 2014]

This work is about replacing conventional packet switching with a combination of optical-circuit switching and packet switching on the ToR switches. Basically, this work concentrates on the switching technology, which I believe is not close related to data center networks (though this will greatly reduce the switch processing delay, I hold the view that this work should be done by tier-1 vendors)



Related works (using optical switching as a way to reduce latency) include  ***Integrating Microsecond Circuit Switching into the DC (sigcomm13)***



###High Throughput Data Center Topology Design [NSDI 2014]

**Key insights**

[Introduction] ***The goal of this work is to develop an understanding of how to design high throughput network topologies at limited cost, even when heterogeneous components are involved, and to apply this understanding to improve real-world data center networks.***

**Methodology**

Leaving routing and congestion control as system-level problems, the paper will study the the relationship between throughput and network topologies.

1. formulation scheme to find the outbound throughput of homogeneous topology (N switch with r-regular graph)

   > $T_{H}(N,r,f)$ denotes the ***throughput measurement*** of an $r$-regular subgraph $H$ of $K_{N}$ under uniform traffic with $f$ flows. The average path length of a network is $$ \langle D \rangle $$. Then we have :
   > $$
   > T_{H}(N,r,f) \leq \frac {Nr} {\langle D \rangle f}
   > $$
   >

   > From [Reference](https://onlinelibrary.wiley.com/doi/abs/10.1002/net.3230040405), we have the minimum value of $\langle D \rangle$. Therefore, we can have the maximum value of $T_{h}(N, r, f)$. 


### Rapid Precise Packet Loss Notification in DC [NSDI 2014]



***Challenges:***

1. self-clocking stall caused by insufficient ACKs results in **in cast**.

   > I still couldn't understand why insufficient ACKs could cause ***self-clocking stall***, I even don't what is self-clocking stall.

2. inaccurate packet-loss notification caused by ambiguous loss indication results in the TCP out-of-order problem.

3. slow packet loss detection leads to long query completion time.

   > I think what the author wants to say is that ***if the packet loss couldn't be detected rapidly, then we need to wait for timeout in order to finish query completion.***



```python
#from this paper, I think the Incast and Outcast TCP communication was a large problem during 2009-2014, because of the appearance of cloud-computing (map-reduce and parallel data searching)

#the main concentration lies within adapting TCP (the congestion control and retranssmion algorithm)
```



***insights of TCP in Data Centers***

The most beautiful part of the paper is the insight thoughts of the performance of TCP and DCTCP in data centers. 	

> ***need to learn***: The authors established a testbed (consisting of 1 aggr, 4 ToR, and several servers, and built DCTCP to give a comprehensive evaluation of 4 challenges of using TCP in data center networks.)![Rapid PLN1](/Rapid PLN1.PNG)
>
> This figure shows that: 1) the DCTCP could also suffer from low goodput as TCP when the senders get larger. This is due to the limitation of congestion window. 





### GPUNFV: a GPU-Accelerated NFV System

### Low-Latency Software Rate Limiters for Cloud Networks



软件限速器虽然可以对数据中心的带宽进行分配，但是由于在hypervisor中引入了新的一层，增加了排队时延，在另一方面造成了时延的上升。减小该时延的方法大类上分为两种：基于**算法**或基于**硬件卸载**。本文采用了基于DCTCP中的拥塞窗口的控制算法对排队时延进行降低，但是极大降低了吞吐率； 另一种作者提到的方法是发表在NSDI 2014的叫做**SeNIC**的方法（本文作者认为该方法缺乏灵活性，但是硬件本身缺乏灵活性，若能够满足现实情况则为一个假问题），该方法使用硬件实现限速器，在提供QoS的基础上减小了排队时延，并且相比较DCTCP，对吞吐率的影响也较小。

### The case for Flexible Low-Level Backend for Software Data Plane

VPP能够在DPDK的基础上，利用有向图对报文向量（packet vector）进行批处理，从而减小i-cache与d-cache的失效几率，提升在CPU上报文处理的效率。但是VPP面临的问题是需要开发人员将NVF代码依照VPP中自定义的有向图中的各节点进行拆分和重写，对开发人员的low-level network process的理解提出了较高的要求。PVPP希望借助P4语言设计一种VPP的compiler，从而自动化地将VNF的代码编译为VPP代码，从而能够高效、方便地进行基于VPP的NVF的开发。



### LOS:  A high performance and compatible User-level Network Operation System

目前基于kernel-bypass的技术面临着最大的问题是无法支持传统基于kernel的网络应用（改变了API，将network stack变成了一个application中的线程）。 作者希望重新设计一套kernel-bypass的network stack，整合现有技术的优势，接管network related APIs, 获取etc中network related files的访问权，支持新的事件通知机制，检测应用崩溃，回收资源，并且向上提供与kernel network stack相同的API。 

另外，DPDK等kernel bypass技术均需要绑定核心进行网络处理，作者提出为了支持大规模的网络应用，需要在多multi-core上进行network process。 为此，作者使用了lock-less shared-queue技术实现进程间通信，从而实现低开销的multi-core based network process。

作者基于单核架构实现了初始版本的LOS， 可以支持在LOS上成功运行Nginx和NetPIPE（未对代码进行任何修改）。LOS实现了17~41%的throughput的提升和20~50%的延时降低，相比于Linux内核。

**系统设计**

LOS基于DPDK实现网络I/O，同时重新为每个socket分配文件描述符，从而避免使用VFS（这对处理突发的小流很有帮助）

兼容性设计中，需要解决的问题主要有两个：使得应用能够共用LOS的API，





### SoftRDMA: Rekindling High Performance Software RDMA over Commodity Ethernet

 

文中首先指出， 基于NIC的RDMA能够有效降低CPU开销和数据传输时延。另外，数据传输全部基于用户态进行，从而避免系统调用和上下文切换的开销（context switching overhead）。

作者借助的技术有： 零拷贝/一拷贝、内核旁路、Memory pooling、内存预分配、内存重用。利用这些技术实现基于软件的RDMA， 在达到与硬件RDMA相似性能（吞吐，时延）的情况下，提供了更好的灵活性（RDMA的协议更新）

**网卡驱动如何支持网卡DMA将数据写入内存（即driver所分配的buffer ring中）**

**零拷贝是如何做到的？**

在全卸载的RNIC模型中，由于应用首先会通过内存管理模块向内存注册，所以经过RNIC的协议处理，NIC就会知道应该将报文数据放在内容的什么位置。***作者认为，对于监测类的middlebox应用，可以使用这种方式；但是如果需要在NIC支持复杂功能或者协议处理，单拷贝是无法避免的。*** 

三种网络处理的线程模型（threading model）

作者在阐述自己使用的threading model时，提到了收包的复杂性要高于发包。

**疑问**：作者在阐述为什么使用3.3（c3）的threading model可以将底层raw API和高层应用程序很好结合起来时没有理解。



### Fastpass (Sigcomm 2016)

疑问：

1. FCP协议的原理？ 没看懂
2. 快速着色算法
3. **The average latency with Fastpass is no worse than than 2x the average latency of an optimal scheduler with half as much network capacity on this worst-case workload. **
4. 在进行timeslot allocation时， arbiter需要的数据结构是什么样的？
5. 

### Automatic Test Packet Generation（CoNext12）

解决的问题？

​	当前网络规模越来越大，拓扑和组成也越来越复杂，但是网络测试与debug的工具仍然仅限于`ping`与`tracert`，难以快速确定switch与router的规则是否配置正确/生效，以及当前网络中存在的性能瓶颈。

方法：

   	1. 对网络测试行为进行抽象，作者认为网络测试存在三层共3个方面的检测。A=B方面的研究较多，但是有2/3的问题在于B=C的检测，所以ATPG将针对这个方面进行检查。
  2. 对网络问题以及需解决迫切程度进行调研，将调研结果在所提出模型中进行映射，找出现行方法难以解决的问题。
  3. 对网络进行建模
  4. ATPG包产生算法以及故障定位算法



### A Packet Generator on the NetFPGA Platform (FCCM'09)

解决的问题：

​	基于NetFPGA实现对PCAP文件的重放（按照格式与时间间隔发包）



## 网络测试

### OSNT [Sigcomm15 demo]

OSNT的报文产生模块是重放PCAP文件，并且用户指定inter-departure time。

OSNT的软件提供了一些良好抽象的API用于控制报文产生于报文监测的功能。

在这个demo中，作者首先使用OSNT验证了FPGA与CPU结合的方式可以进行基本的交换机性能测试，然后通过OSNT与OFLOPS-Turbo结合的方式说明了FPGA与CPU结合也可以定制化的测量，最终目的还是为了打造一个测试环境。

### OFLOPS [PAM2012]

作者提出一种openflow协议测试框架，在该框架上，可以开发各种应用用于对openflow交换机的性能进行测试。

![1534943981697](../2018-11-20-memo/1534943981697.png)

作者认为，对于OpenFlow交换机的性能测试有助于理解OpenFlow的应用场景与use case, 作者提出了OFLOPS的针对openflow的测试框架，对转发表的一致性、流建立时延以及流量监测的能力进行测试。作者在实验中对三种硬件交换机与OVS的性能进行了测试，并且通过对测试结果的分析得到了一些产品不会写到说明书中的内容（如对openflow部分功能的支持情况等。。。）

##### packet modification

##### Flow Table Update Rate

##### Flow Monitoring

##### OpenFlow Command Interaction



### SOFT [CoNext 12] EPFL

作者采用symbolic execution的方式对OF交换机的互操作性进行测试。使用variables的输入去测试针对每个功能的输出，并对多个OF设备的输出与



![1535418860150](/../2018-11-20-memo/1535418860150.png)

The increasing adoption of SDN, and OpenFlow in particular, brings great hope for increasing network extensibility and lowing the cost of deploying new network functionalities.

The model checking and symbolic execution are used to check the interoperationability of two switches. The main contributions lie in the control of input space and limiting the mount of total code path.



### Blueswitch (ANCS 2015)

this piece of work concentrates on how to maintain the consisitency of switches within a network, so as to guarantee that all the packets are forwarded, dropped, or modified using the same version of policy.

The key observation of this paper is that the key building block of packet policy consisitency is the provision of packet consistency inside the data path of a single switch.

**The authors present a series of low level API on switch to support bundle features on OpenFlow switches**

>as the authors argued that current commercial OF switch doesn't support the bundle features of OpenFlow specification. 

The true story in this paper is that a flow mod message (updating flow entry) will cause the inconsistency of TCAM and RAM table (which will cause packet processing error further). Thus, what the authors did is to maintain the atomic operation between TCAM and RAM. 

![1535547549423](/../2018-11-20-memo/1535547549423.png)

1. where does metadata generated? 

   > the metadata is generated in hardware of NetFPGA

2. how to tag $V_{p}$ into metadata?

   > this is controlled by hardware, when 

3. How the flow table is updated, in the senario of two TCAMs? 

   > if the flow update is replaced by adding a new flow entry in the TCAM?

4. what if a packet is tagged with updated $V_{p}$, but the $S_{i}$ state remains as open? 



### Swing [TON 2009]

**Goal**

The goal of this work is to design a framework capable of generating live network traffic representative of a wide range of both current and future scenarios.

**Meaning**

valuable for a variety of settings: high-speed router design, capacity planning, queue management studies, bandwidth measurement tools, network emulation and simulation.

**Challenges**

1. Require an underlying model to simply demonstrate the semantical features of a given trace.
2. Traffic generated by the model should reproduce the essencial characteristics of the original trace.

***The authors believe that packet generation is a tool to aid higher level studies.*** It would be much more necessary to evaluate the outcome of packet generation (**They use realism, responsiveness and aximally random**)

### High-Fidelity Switch Models for SDN Emulation (HotSDN 13)

**Motivation**

In traditional switching hardware, the control plane rarely affects the performance of data plane; it is reasonable to assume the switch forwards between ports and at line rate for all flows.

However, 



### OpenFlow (SIGCOMM CCR 2008)

**key insights**

1. the key abstraction of network programmability is flow table on the switch. ***in order to protect insides of switch (IP of vendors)***, instead of let researchers program on the switch, openflow leverages a remote cntroller to modify the flow table using the openflow protocol. 
2. another key step is that the author find in order to enable the research of novel network protocols, using the **match action** pattern would be a good choice. i.e., give support to several match fields, and three actions, the openflow can be used to conduct network experiments.
3. the intuition of OpenFlow is to enble network research on real traffic, the author wants to persuade network vendors to support OpenFlow in their products, promoting network innovations.





1. 抽象为几个阶段，针对每个阶段，枚举需要做的task。
2. P4程序都用来实现什么样的网络功能了？
3. P4结构不支持的话，那就需要ANT去测程序是否正常运行？
4. 一种是逻辑问题，一种是映射过程中出现的问题。
5. 测试是否deparser能够将所有egress pipeline中的合法报文头发出？
6. 



### P4V [Sigcomm 2018]

***Reference***

1. [Cisco firewall bugs in ASA V8.2-V8.4](https://www.forwardnetworks.com/2017/02/23/network-path-not-found-how-to-use-an-army-of-tests-to-understand-and-diagnose-your-network/)

















