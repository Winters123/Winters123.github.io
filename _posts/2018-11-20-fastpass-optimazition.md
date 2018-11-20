---
layout: post
title:  "FastPass, 集中式数据中心流调度"
categories: Data Center
tags:  Data_Center 流调度 
author: Xiangrui Yang
---

* content
{:toc}


## FASTPASS proposal 思路

1.       详细解释实验场景与过程以及结果
2.       提出实验中简化过但是影响较大的问题
3.       为何这些问题可能在大规模部署中出现问题

4.       量化分析这些假设放在大规模环境下将会出现什么问题

5.       有没有好的解决方案？ （雏形）



#### 实验目的

FastPass的设计目标是同时在数据中心网络提供**零队列、高吞吐、对流/应用间的资源分配**。实验中通过将FastPass与传统TCP机制进行对比从而验证FastPass在这几个方面的效果。

**网络节点**

1. 交换机平均队列长度
2. ping时延（在相同的背景workload下）
3. 多条TCP流间的fairness
4. NIC队列时延
5. TCP重传次数的减少比

**调度器**

1. 调度器引入的throughput开销
2. 调度器能够支持的总throughput

#### 实验场景

作者的实验在一个Rack内进行（包含32台server，一台ToR交换机，并通过10Gbps NIC相连）

服务器： 2*Intel CPU， 8核16线程 per CPU,  148GB RAM, 关闭TSO从而增强对NIC报文队列的控制， 一台服务器用于运行arbiter。

交换机： 标准32/4端口ToR交换机

实验拓扑：

![topo](/2018-11-20-fastpass/topo1.png)



#### 过程与结果分析

##### Throughput/Queue

作者在S1-S4上运行```iperf```，分别与R建立20条TCP连接，持续20分钟。在baseline与fastpass模式下分别运行一次，并对实验结果进行对比。

***在throughput上***，baseline平均为9.43Gbps, fastpass平均为9.28Gbps。

> 作者认为，fastpass在throughput上引入了1.6%的减小是因为 **1%的PFC的开销** + **0.6%的PFC重传开销**。 作者说这里0.6%是由于有些报文没有在threshold内发出，所以端系统再次向仲裁器请求时隙和路径。

***分析***：在仅控制5个节点的实验环境下，向仲裁器的重传请求占据了初次请求的60%。这可能是由三个方面的原因导致： request/response路径时延、或者仲裁器计算时延。根据计算， request/response的时延约为4.8us (1500B / 10Gbps) ，所以较可能的情况是arbiter的处理过程引入较大时延，导致所分配的时隙超时。

【CPU时延分析】

根据fastpass源码，作者使用DPDK进行request/response处理（时延为？）， 计算过程经过分析能够在1-20us内完成。

![topo2](/2018-11-20-fastpass/topo2.PNG)

【潜在解决方案】

   规模增大后，仲裁器CPU收发报文的时延变得难以估计，并且很有可能继续增大，导致re-request的几率进一步增大。使用FPGA作为仲裁器可以避免收发报文时延的进一步增大。 【FPGA是不是快于DPDK？】





***在queue长度上***， fastpass中ToR switch的队列长度为18KB（中位数），而传统方法在ToR switch中引入的队列长度为4.35MB（中位数），减小了242倍。同时从图中可以看出，尾时延的降低幅度也较大（尾时延的降低幅度大约为13倍）。 

![queue](/2018-11-20-fastpass/queue.PNG)

***分析***：以5-hop规模的数据中心为例，若取4.35MB/18KB作为传统/fastpass的数据中心的交换节点的队列长度，链路速率为10Gbps，则传统方式引入的队列时延为:
$$
\left (  4.35MB\times 10^{6}\times8\right )\div 10^{10}\times5hops=17.4ms
$$
而采用fastpass在相同场景下引入的时延为下公式所示，理论上在交换节点的排队时延减小幅度应为214倍。
$$
\left (  18KB\times 10^{3}\times8\right )\div 10^{10}\times5hops=72us
$$
在没有fastpass进行调度的情况下作者认为交换机节点的队列长度应当是满的，这是由于TCP的发送速率只有在出现packet drop的时候才会降低，而packet drop一般是由于交换机queue full的时候才会出现。**从这个角度考虑，确实较小的switch queue将会带来更小的时延与丢包率**。

> fastpass的最终目的就是用fastpass的双向时延换取数据报文的零队列发送，根据该实验所展示的结果，中间节点的排队时延降低了2个数量级，时延降低了超过10ms（在满负荷情况下）。而fastpass本身带来的时延开销小于1ms， 所以应当是值得的。



【非零队列的原因】

作者提出没有达到零队列的原因是**jitter in endpoint transmission times**， 并且指出可以通过在内核中使用更细粒度的锁实现更小的发送时间抖动。

作者在源码中通过在qdisc队列里进行报文的调度实现fastpass指定的时隙发送报文，然而，在每次操作报文时都需要对qdisc加锁实现，这可能是问题的根源。

另外，使用qdisc还会涉及到进程间调度，所以无法保证qdisc何时才会将数据交付网卡进行发送。

***需要读fastpass和linux kernel的这部分源码*** 

【潜在的解决方案】

如果在内核中不对qdisc修改，而在报文从网卡队列发出后再根据fastpass需求在FPGA中进行调度，则可能可以解决这个问题。

##### Latency

作者在该实验中依然使用```iperf``` 从S1-S4向R发送TCP流量，同时使用S5向R以**10ms**为周期发送```ping```请求，从而测试fastpass对RTT时延是否能够减小。 首先``iperf``保证了baseline与fastpass两个场景处在相同的workload下， 从而使得RTT能够真实反映fastpass对时延的影响。实验结果如下表所示，总体RTT下降约为15倍（尾时延降低降低超过10倍）。

![queue](/2018-11-20-fastpass/latency.PNG)

![ping](/2018-11-20-fastpass/ping.PNG)

> 上述结果表明在该场景下，即使考虑到FastPass在其中增加的2倍（**Ping的往返报文都需要fastpass进行仲裁**）RTT与仲裁时间，仍然可以取得超过十倍的时延收益。fastpass的意义应当在于整体缩小了交换节点的排队时延（**前提是数据中心中traffic时延主要来自于交换节点**）。

***分析***：在该实验中，报文仅经过了一跳的中间节点，若在标准5 hops的数据中心中进行通信，则可能能够获取更高的时延收益。根据上一节的计算可得，每一跳的时延收益约为$	\left ( 17400us-72us \right )\div5=3.46ms$, 与作者在本节得到的实验结果几乎相同。则可估算，在5 hops数据中心通信中，将会获取$15ms-20ms$的时延收益。如果fastpass能够扩展到整个数据中心中， 将极大降低数据中心网络的整体通信时延。

##### Fairness/Convergence

在该部分将测试fastpass实现多sender间throughput分配公平性的问题，同时也将测试达到均衡throughput的收敛时间。

【实验过程】：作者使用S1-S5向R发送bulk TCP流，前30s为S1->R， 之后每30s新加入一条bulk flow，然后每隔30s再结束一条TCP流，整个实验持续270s。使用```iperf```记录每秒每条流的throughput，得到的结果如下图所示。

![fairness](/2018-11-20-fastpass/fairness.PNG)

***分析***：图中的第一部分显示了传统TCP分布式的拥塞控制所能够达到的流量均衡性的效果（**先前的研究人员为了实现TCP流throughput的均衡性应该想了不少办法，但是在链路情况未知的情况下要综合考虑congestion与流量的均衡性应当是很难的问题**）， 作者在2017年发表的一篇论文中也指出， TCP的congestion control需要经过几个RTT才能大致达到分配资源的稳定，而通过fastpass能够在微秒级别实现链路资源的均衡分配。

【throughput较低的原因】能够注意到，在1-2条流的时候，整体的throughput较低，大约只能达到传统TCP传输的65%。作者文中指出该情况是由于fastpass引入一些报文处理的额外开销，而这些开销可以通过TSO降低。还有另一个原因是fastpass需要在qdisc中对报文进行操作，也会引入一定的开销。

>  为什么这种情况在TCP流数目较小的情况下才会出现？**作者这里没有讲的很清楚，需要看源码并询问作者。**



【潜在解决方案】这里throughput较低与第一节1.6%的throughput本质是一个问题，都是由于fastpass在内核中对报文进行操作导致的。可以借助作者的思路在内核中使用更细粒度的锁进行优化，但是可能在NIC上提供支持，即在不修改内核的情况下在SmartNIC上实现fastpass对报文的调度应当可以解决throughput penalty的问题。



##### Arbiter

仲裁器主要有四个方面性能将会对Fastpass的整体性能构成影响： request/response队列时延， FPC的通信开销，时隙计算开销与路径计算开销。本节将分别对这四个方面的实验结果进行分析。

###### request/response 队列时延

【测试目的】：测试一条request报文到达仲裁器网卡后需要经历多久的排队才能进行处理。

【测试方法】： 作者每隔10s新建一条TCP流用于向一台server发送数据，同时在仲裁器的CPU上测试NIC的pulling速率，能够预期，随着throughput的增加，polling rate将会逐渐降低（为什么rate会降低呢？）。实验结果如图所示。

![polling](/2018-11-20-fastpass/polling rate.PNG)

> 没有理解throughput增加，polling rate会降低？ 感觉应该是不变。
>
> 可能的原因： arbiter使用一个核用于NIC的报文收发，与预处理（使得计算核可以直接进行运算），随着收到的报文速率增大，该核需要更多资源用于预处理，这将导致NIC polling rate降低。

###### FCP通信开销

【测试目的】测试fastpass相比较传统方法，在网络中引入的额外throughput。

【测试方法】作者在仲裁器上记录发送和接收报文的速率（在不同的workload下），发现request报文的开销占到了总throughput的1/500， allocation开销占到了总throughput的1/300，如下图所示。**在调度压力为150Gbps时，通信核接收request的开销为0.3Gbps, 发送allocation的速率为0.5Gbps。**如下图所示，当处理throughput到达160Gbps时，由于NIC polling rate的下降，将会出现整体throughput的下降（即出现了request报文丢弃）。

![polling](/2018-11-20-fastpass/arb throughput.PNG)

【分析】实验中说明，基于单核实现的arbiter的request接收/处理时可扩展性面临着较大压力（大约为160Gbps，即0.3Gbps的request接收速率）。但是利用RSS等技术进行多核扩展可以成倍提升arbiter接收/处理request的能力。

###### 时隙计算开销

【实验目的】测试多核处理time allocation的能力

【实验方法】使用一个称为“压力测试核”的核用于发送request请求，所发送的请求满足泊松分布，sender和receiver是在256个节点中随机挑选，一个request对应着10个MTU的分配。实验结果如下所示。总的来看，在8-core上能够实现2.21Tbps的流量调度，即221台server在10Gbps链路上的调度。

![polling](/2018-11-20-fastpass/time.PNG)

###### 路径计算开销

【实验目的】测试完成路径选择需要多少核的参与。

【实验方法】实验场景与方法与上述时隙计算开销测试方法相同。测试结果如下图所示。

![polling](/2018-11-20-fastpass/path.PNG)

##### facebook测试

作者将fastpass部署在fastpass中用于支持一项时延敏感业务（响应用户的web请求），该业务与search业务相似，流量特征为partition-aggregate模式。 即每个rack内拥有30-34个aggregator- leaf节点，rack间相互隔离。

**为了能够保证低延迟，aggregator给leaf发送的请求（500K-200M）是根据当前的链路状态决定的：若网络负载低，则给尽可能多的leaf发送请求，并且尽可能向用户返回质量更高的结果**；若网络负载高（P99增加），则相反。

每个server的流量模型如下图所示：

![polling](/2018-11-20-fastpass/server.PNG)

作者提到整个workload约每十分钟进行渐变， 所以整个实验共进行了135分钟，前30分钟和后25分钟是正常TCP拥塞控制模式，中间切换为fastpass模式，从而进行对比实验。

在130分钟的整个实验结果如下图所示：

![polling](/2018-11-20-fastpass/facebook.PNG)

***分析*** 

第二和第三个图通过量化分析都可以得到，但是第一个图显示fastpass对P99 latency没有影响（与传统方式相同）,文中也是这样描述的，与之前实验所得和我们的量化分析差别很大，**已经询问了作者，作者解释Facebook的该组机柜的throughput的利用率平均为1%，在ToR交换机上几乎没有排队时延，这使得latency的减小效果一般**。另外，作者仅在一个rack内部署了fastpass，而该业务涉及到其它rack内的工作依然使用传统TCP拥塞控制的方法，所以对latency的影响不大。

> 在该测试中对应的业务与map-reduce类似，所有请求先由aggregator收到，并分发到leaf中进行计算；leaf计算完成后将分布式的结果返回aggregator， 并由aggregator交付用户。整个过程涉及**2-4**次资源请求，即fastpass带来的时延收益大约为$1/2-1/4$。

作者在邮件中提到，这部分实验是为了验证对降低报文重传率的影响（降低了2.5X），而丢包/重传对map-reduce类型的任务影响是巨大的。

#### 可用性评价

##### throughput

FastPass基于Linux内核的版本会带来throughput的开销，这是因为qdisc（加解锁）以及进程间调度会导致发送时间抖动，在前述的测试实验中，也证明了这会为throughput带来最高（单条TCP流的情况下）33%的throughput损耗。

另外，FastPass本身的控制报文在实验验证中引入的开销为1.6%，几乎可以忽略不计。

##### latency/queue

以第一节的量化分析为例，fastpass对于交换节点的queue长度减小的效果是巨大的（在5-hop的数据中心中，队列时延降低超过两个数量级），这部分是可用性提升



##### fairness

#####arbiter









### 基于FPGA/CPU的FastPass优化设计方案



#### 优化动机

​	FastPass能够利用在集中调度器上运行调度算法，在整个网络环境中达到packet-level的细粒度调度，从而消除报文在交换节点的排队，带来降低时延、减少重传和降低对转发节点packet buffer能力的要求的目的。

​	根据FastPass[1]对基于内核的fastpass的测试结果，该原型存在着以下几点不足：

1. 报文在server的内核中将会根据仲裁器返回的结果进行细粒度的调度，由于中断以及qdisc加解锁的开销，导致server的吞吐率下降（在单条TCP流的情况下吞吐率下降超过30%）。
2. 【可能】FastPass选择在内核上进行调度而非在网卡上进行调度，将可能出现时隙偏移并无法做到更加精确的时隙控制，造成交换节点无法达到真正的零队列，这在大规模部署场景下可能引入问题。
3. 调度器采用多核分别执行request/response、时隙分配、路径分配任务的架构。根据作者的压力测试的结果，16核server能够进行集中调度最大traffic的throughput为160Gbps， 只能支持单rack内的调度（甚至难以支持单cluster）。在大规模部署场景下的可扩展性存在问题。


​	上述问题主要来自于三个方面：第一，将fastpass的客户端部署在操作系统内核，使得结果不可预知性增加， 同时增加了开销；第二，采用单台server作为整个数据中心的集中调度器，难以支持大规模部署；采用单层的client-server架构，且client-server通信频率高（微秒级），在大规模部署中引入开销大。

​	为了能够利用fastpass的优势，同时消除以上几点不足，本文将基于SmartNIC（FPGA）上实现FastPass的客户端，从而解决问题一与问题二；另外将利用层次化的扩展方式，解决问题三。

​	

#### 优化思路

​	本章将主要对FastPass的客户端（如果按照层级部署将会涉及到至少两层客户端）、FastPass调度器以及可靠的FastPass传输协议提出优化设计。

​	优化的FastPass的部署方案整体而言如图所示，其整体而2层架构，分别对rack内和跨rack的报文进行调度。在rack内（即1st Tier中），每个server的SmartNIC将在本rack内的fastpass调度器控制下对报文进行转发。而对在fabric中，来自不同ToR交换机的报文将在跨rack的fastpass调度器的控制下对报文进行转发。 ![topology](/2018-11-20-fastpass/FPtopo.png)

##### 基于SmartNIC的FastPass客户端

>这里面临的最大问题是原文中TCP重传优化是借助内核同步数据发送的反压机制实现的。如果将Fastpass卸载到FPGA，内核在发送TCP数据后就会向应用返回发送成功的消息，这样就无法利用反压的机制实现TCP重传次数的减小（TCP重传可能引起很大的问题，可能会使得Fastpass带来的时延可预测性被抵消，再次引入了不确定性）。目前觉得最简单的方式是修改内核中负责发送skb的函数，使其以阻塞方式等待来自FPGA的发送完成的消息。**这中间涉及到FPGA与主机的协议设计**。
>
>最合理的方式是尽可能不对主机进行修改，从而借助本身的反压机制控制TCP的发送速率。先看内核代码看看有没有机会...

​	在本文中，我们使用SmartNIC支持fastpass客户端，相比较基于kernel的fastpass客户端，基于FPGA的版本能够消除kernel引入的中断和互斥锁带来的吞吐降低和时隙偏移的问题，同时对kernel的修改较少，利于大规模部署。

​	基于SmartNIC的FastPass客户端主要包含以下四个子功能，下面将分别介绍这四个子功能的设计思路。

1. 基于PTP的时钟同步；
2. 基于fastpass的报文精确调度；
3. TCP发送窗口反压机制；
4. 基于目的IP地址的报文分类机制。



###### 1. 基于PTP的时钟同步

【参照付文文设计】



###### 2. 基于FastPass的报文精确调度

​	该功能的主要任务是根据fastpass调度器所分配的时隙调度报文进行精确发送。在被调度前，报文将被缓存在FPGA片上存储单元或者外接DDR的存储单元中，其中缓存的报文将会根据返回的时隙经过调度后按序发出。

​	FPGA本身需要实现从缓存报文中抽取```ip.src, ip.dst```，并向FastPass调度器发送时隙请求报文。待返回时隙分配响应报文后，将按照所分配时隙将缓存报文进行调度后发送。

![fpc](/2018-11-20-fastpass/FPC.png)

​	【如何流水】为了能够进行流水，从而消除报文调度发送带来的开销，需要扩展报文存储模块的大小，使得Pkt Buffer内可以存储多个Pkt Batch, 在进行第一个Batch的调度时，依旧可以在硬件流水线中接收来自OS的报文，从而避免在batch未得到调度发送时整个Traffic发送处在阻塞状态。

> 1. FPGA应当维持的batch的大小是由每个batch发送调度请求、接收调度应答和执行调度操作的总时间内所能收到的报文的总大小决定的。可知在不同的部署场景下batch的大小以及数量是不同的，但是需要为Pkt Buffer设计门限值，从而满足最低需求 -> **从这个角度讲，Pkt_Buffer越大越好。**
> 2. 但是从另一个角度，Pkt_Buffer过大将会极大提升SmartNIC的成本，同时在SmartNIC中增加报文排队，需要根据1中最低要求设置buffer大小。当来自OS的报文超过这个值是，需要借助SmartNIC的反压机制阻止来自OS的报文进入SmartNIC。 -> **这也是设计FP对TCP发送窗口反压机制的原因**。

###### 3. FastPass发送窗口反压机制

​	该功能主要是在SmartNIC缓存报文达到某阈值时通知OS停止发送TCP报文。主要包含以下四步流程

1. 在SmartNIC上，当所缓存的报文达到阈值时，向OS内核发送消息告知其缓存空间已满；

2. OS在收到来自SmartNIC的消息后，将通知上层应用暂停数据包发送；
3. 当SmartNIC成功调度一个batch发出后，将再次向OS内核发送消息告知可以继续发送数据包；
4. OS在收到消息3后，将通知上层应用继续数据包的发送。

> 目前来看这个机制不好实现，因为现在kernel中对TCP窗口的反压机制是通过发送函数是否返回来实现的，即数据包还未发送完成时，将不会向上层应用返回，这种阻塞机制保证了TCP的正常运行，而不在内核产生丢包。而目前关于如何通过消息机制实现类似的阻塞机制还没有好的想法。可能**还需要探索别的方法**。
>
> ***计划先阅读该部分源码研究是否可行，若不可行 ，探索NIC驱动有无反压机制，以及借助该机制实现的可行性。***

​	付：***不改内核，利用ECN机制降低TCP发送速率。***


###### 4. 基于目的IP地址的报文分类机制

**场景**： 

> SmartNIC收到的报文同时包含需要发往同一Rack的其它server（甚至同一server的其它VM）和其它Rack两类。SmartNIC在这种情况下如果将所有报文统一考虑就会导致head-in-line的阻塞。
>
> 现在多数数据中心用交换机都支持根据端口隔离Pkt buffer，从而防止head-in-line阻塞。在这种情况下可以在SmartNIC上统一处理。
>
> 所以SmartNIC需要实现报文根据目的地址分类，位于同一rack内的流量由SmartNIC向调度器发送请求；而跨rack的流量由位于ToR上连的FPGA做处理，SmartNIC不对这种报文进行处理。

​	根据上述场景，可知需要在SmartNIC上实现基于目的IP地址的报文分类机制。为了能够灵活支持该功能，SmartNIC需要感知当前主机所处的rack的IP子网，并将其配置到FPGA中， 从而能够将发往同一rack内和跨rack的报文进行区分。



##### 基于分层结构的可扩展FastPass调度器

​	根据[1]中的实验结果，基于多核的FP的调度器（使用单核进行request/response）大约可以支持160Gbps的traffic的调度。为了将FP扩展到整个数据中心，需要设计层次化的FastPass调度器，从而提升FastPass的可扩展性。本文考虑中、大型数据中心的需求，设计两层的FastPass调度器架构。

###### Rack内FP调度器设计	

​	根据[2]中在微软6000节点（与十万节点数据中心规模相同）中测量结果，其内部主要业务（搜索，社交网络内容组合，广告筛选）均基于Partition/Aggregate机制实现，并且这种机制主要实现在同一Rack内。也就是说，同一Rack内小流（即DCTCP[2]描述的foreground flow）主要为Partition/Aggregate业务流量，根据[2]的测试结果可知，Work和Aggregator的任务交付流量一般在2KB左右（不超过4个报文），并且由于Partition/Aggregrate的任务模式，单个work响应无法按时提交（超时）将会使得aggregator无法向用户返回结果，代价很大。所以在同一rack内进行packet-level调度是有必要的。

​	首先考虑同一Rack内的FP调度器部署，由于同一Rack内负载较轻，采用与[1]中相同的基于多核的server作为FP调度器可以完成Rack内调度。并且rack内ToR交换机端口与server是一一对应的，也无需进行路径分配，此部分设计可以大部参照[1]中设计。



###### 跨Rack FP调度器设计

​	根据[3]对微软数据中心东西向流量测量结果，跨东西向的流量中95%左右的流处在**1KB-1MB**之间（**现在是否有变化？**），然而**90%**的流量都属于大小为**100MB-1GB**的流中。

> 从测量结果看，由于大部分traffic都处在100MB-1GB的流中，数据中心跨rack的流量很适合以bulk方式分配时隙和路径。但是也存在报文数量很小的流，为了防止这部分流被饿死（被丢弃或者超时），在跨Rack的FP调度算法上需要给予小流**一定的优先级**。

​	从另一个角度看，由于规模不局限在rack而扩大到了整个数据中心，packet-level的request/response过程将会在整个Fabric中引入很大的通信开销。所以也必须以bulk的方式进一步压缩FP通信开销。

​	因此，每个ToR交换机的上连FPGA将首先收集同属一个IP三元组（根据[2]，微软DC中99.9%流量为TCP流）的pkt batch，并以batch为单位向跨Rack的FP调度器请求时隙与路径。而为了防止小流被饿死，需要识别小流，并给予其一定的优先级。这里存在两个难以解决的问题，第一是如何识别小流，第二是如何对小流进行高优先级的调度。为了规避这两个问题，首先分析fastpass路径和时隙分配的特点，由于在路径和时隙分配的过程中，FP调度器只需要关注被调度报文的```ip.src, ip.dst```，所以在数据中心跨fabric的流量中，如果能够依靠```ip.src, ip.dst```聚合大小流，则可能可以解决小流被饿死的问题。**该方法可行的前提是跨fabric流量中大小流能够通过```ip.src, ip.dst```进行聚合，需要真实的测量结果证实该假设是否成立**。

​	若上述假设成立，跨Rack的FP调度过程如下： 首先，每个ToR交换机的上连FPGA将依靠```ip.src, ip.dst```对报文进行分类，每一组```ip.src, ip.dst```相同的报文称为一个batch, 当多数batch中报文数量到达阈值或FPGA的总packet buffer达到所设定阈值时，将向跨rack的FP调度器发送调度请求消息；当收到来自调度器的响应消息后，则依据所分配的时隙和路径发送packet batch。

​	通过简单的量化分析该机制理论上会产生的效果。

​	首先计算该方法引入的排队时延。假设ToR交换机的上连端口为40Gbps，一个packet batch包含报文数为N，FPGA支持一次FP请求的总报文数为M。FPGA接收一个完整MTU的时间为100ns, 收到共M个报文的时间为**M*100ns**。报文在FP架构下引入的时延约为：
$$
L_{FP-ToR}=M\times100ns+T_{RTT}+T_{arbiter}=100\times100ns+50us+50us=110us
$$
​	在M取100， 另外两部分均为几十微秒的情况下，该方法引入的时延为亚毫秒级，开销并不算大。另外再计算该方法可为packet batch传输带来的时延减小的效果。传统情况下，假设Leaf与Fabric交换机的packet buffer大小为4MB（很多商用交换机的packet buffer为16MB，但是可能启用了端口隔离，采用4MB进行折中），各端口速率为40Gbps。由于packet在fabric中一般经过3跳，若每一跳均需经历的满负荷的排队时延，则这部分的时延为：
$$
L_{w/oFP}=3\times\frac{4MB}{40Gbps}=2.4ms
$$
​	而如果采用FP机制，在fabric中经过3跳的总的排队时延为：
$$
L_{wFP}=3\times \alpha + L_{FP-ToR}=3\times \alpha+110us\simeq110us
$$
 	其中，$\alpha$为使用FastPass机制仍然难以消除的一部分开销（[1]测试中使用FastPass后每个节点的队列长度约为P99 4KB，引入时延约为800ns, 可忽略不计）总体来看，理想情况下仍然可以取得一个数量级左右排队时延的收益。

​	然后再说明该机制运行的整体流程，如下图所示即为ToR交换机上连的FPGA执行packet batch调度发送的流程。首先FPGA收到的所有报文将首先根据```ip.src, ip.dst```进行分类，packet buffer中将根据IP二元组对报文分类缓存，如果达到触发请求路径和时隙的条件时，FPC的控制模块将会把packet buffer中报文摘要作为FastPass request的负载发至跨rack的FP调度器，当收到response消息时，FPC的控制模块将会控制报文发送模块按照调度器所分配的时隙和路径将packet batch进行发送。

![FPC2](/2018-11-20-fastpass/FPC2.png)

​	

##### 可靠、轻量级的FastPass协议

​	可完全采用fastpass[1]中的协议规范进行设计。

##### 时隙、路径分配算法

​	FastPass[1]的原文中，作者使用max-min fairness算法进行时隙分配，经过测试可以在收敛性和公平性上达到良好的效果。在路径分配上，作者在假定所有路径权重相同的情况下使用二分图的着色算法进行报文的路径分配。

​	下面分别从rack内fastpass调度和跨rack的fastpass调度两个方面分别探讨本文可采用的时隙和路径分配算法。首先在rack内的fastpass调度中，由于路径与端口一一对应，所以无需进行路径分配而仅需进行时隙分配。并且从ToR交换机的角度看，记其下行端口数为$N$，时隙分配的规模为$N\times N$，该规模与crossbar调度规模相同，所以可以根据需求的借用不同的crossbar调度算法实现rack内调度，可参考的工作有iSLP[4]等。

​	跨rack的调度规模大于rack内调度。以[2]中微软的一个中等规模数据中心为例，其共包含6000个server节点，单个rack容纳server数量为44， 则共计有$6000\div 44\simeq136$个rack。则其时隙分配的规模为$136\times136$，该规模仍然能够参照crossbar的调度算法进行设计。另外在跨rack的fastpass中，报文以batch的方式被分配路径和时隙，也就存在着同时分配多条路径可能性。比如对于包含$N$个报文的batch大小，可以分配一个MTU的时隙与$N$条路径进行转发。

​	综上，目前有两种可选择的分配方式：分配$N$个时隙与1条路径或分配1个时隙与$N$条路径。第一种方式需要调度器计算一次时隙与一次路径，在ToR上连FPGA上修改每个报文相关域指定为同一条路径，并以相同的路径将报文发出。第二种方式需要调度器计算一次时隙与$N$条路径，计算开销更大，并且在ToR上连FPGA上修改batch内每个报文通过不同路径进行转发。简单来看，第一种方法为调度器引入的开销更小，并且出现TCP乱序的可能性更小。 





#### 设计方案

##### 基于SmartNIC的Rack内FP客户端

#####基于FPGA的跨Rack的FP客户端

##### 基于DPDK的Rack内调度器

​	Rack内调度器与[1]中所使用的调度器相同，唯一的区别是在本文中，我们将无需对rack的报文路径进行分配（路径是唯一的），所以一定程度上减轻了调度器的压力。

##### 基于DPDK的跨Rack调度器

#### 实验方案

#### 开发进度

#### 预期结果

#### 参考文献

1. fastpass
2. DCTCP
3. VL2
4. iSLP



##### 疑问

1. 若用fastpass实现数据中心的流调度，还需要ECMP么?  感觉就不需要了。

2. fastpass给报文分配路径，但是此时需要考虑数据中心中的服务链机制。

   > 但是服务链是在underlay还是overlay？ 如果是在overlay就没有问题，如果在underlay则fastpass的路径规划就比较鸡肋了。

3. 数据中心内underlay网络的地址管理机制？ 地址分配机制？ 

4. 当前数据中心内相同业务一般集中在同rack内，以Partition/Aggregrator类型业务为例，数据交换量不大，但是时延要求高。 这一点在当前数据中心是否还成立？

5. 数据中心内东西向跨Fabric的流量主要用于支持哪些业务？ 现在的流量特征是否还与VL2中的测试结果相同？

6. 使用```<ip.src, ip.dst>```作为划分流的标准，则跨fabric的大小流是否大概率被划分在同一流中？ 





