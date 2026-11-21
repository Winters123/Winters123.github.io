---
layout: page
title: 项目
permalink: /projects/
---

围绕**高性能网络与网络系统安全**，课题组在可编程数据平面、时间敏感网络、在网计算与网络教学平台等方向持续攻关，形成了从开源系统、国产芯片到教学平台的完整成果链条。以下按典型项目分节介绍。

## FastRMT / Menshen — 高速可编程数据平面与流水线隔离

<p class="proj-subtitle">开源 FPGA 级 RMT 架构 · NSDI 2022 隔离研究支撑</p>

**研究背景**：传统 FPGA 网络开发门槛高、周期长，业界长期缺少开放的高速可编程数据平面参考实现，微架构创新难以快速验证。FastRMT 基于 RMT（可重构匹配-动作表）模型，在 FPGA 上以模块化方式实现解析、匹配、动作各级流水线，让研究者像搭积木一样定制转发逻辑。

<div class="proj-figures">
    <img src="{{ '/assets/img/fastrmt-data-flow.png' | relative_url }}" alt="FastRMT 数据流">
    <img src="{{ '/assets/img/menshen-daisy-chain.png' | relative_url }}" alt="Menshen Daisy Chain 隔离机制">
</div>
<p class="proj-figure-cap">FastRMT 数据流：可配置模块串联成处理流水线 ｜ Menshen 隔离机制：过滤器 + 门控的 Daisy Chain</p>

### Menshen：流水线模块间隔离（NSDI 2022）

FastRMT 的可编程流水线支撑了与纽约大学、QMUL 合作的 <a href="https://www.usenix.org/conference/nsdi22/presentation/wang-tao">Menshen</a>（NSDI 2022）研究：针对 PISA 流水线中一个模块的错误会波及整条流水线的问题，提出模块间轻量隔离机制——在模块间插入由硬件自动生成的隔离处理器（过滤器 + 门控构成 Daisy Chain），以少量额外硬件资源为代价，保证各模块互不干扰。

<div class="proj-figure">
    <img src="{{ '/assets/img/menshen-pisa-isolation.png' | relative_url }}" alt="Menshen PISA 流水线隔离架构">
</div>
<p class="proj-figure-cap">Menshen 在 PISA 流水线各级间插入隔离处理器（出处：NSDI 2022 论文）</p>

FastRMT 支持 100 Gbps 线速处理，单级匹配延迟约 20 ns，提供 200 余个可配置模块，目前已被 3 类产品采用。

**产业落地**：已被 Xilinx OpenNIC 开源项目、迈普通信网卡、星载交换芯片等核心产品采用，是课题组产业转化最成功的开源系统。项目主页：<a href="http://www.xperis.com.cn/">xperis.com.cn</a> · 代码：<a href="https://github.com/Winters123/FastRMT">GitHub</a>

## FlexTSN / OpenTSN — 时间敏感网络与国产 TSN 芯片

<p class="proj-subtitle">灵活 TSN 交换模型 · 融入全球首个 TSN 开源项目与国产芯片</p>

**研究背景**：工业互联网、车载网络等场景要求微秒级确定性时延，而传统以太网"尽力而为"的转发无法满足。TSN（时间敏感网络）通过时间感知整形、门控调度等机制提供确定性保障，但交换芯片长期依赖进口。FlexTSN 提出软硬件协同的灵活 TSN 交换实现模型，让时间敏感调度逻辑可裁剪、可移植。

<div class="proj-figure">
    <img src="{{ '/assets/img/flextsn-node.png' | relative_url }}" alt="FlexTSN 节点架构">
</div>
<p class="proj-figure-cap">FlexTSN 节点架构：软硬件协同的 TSN 交换实现模型</p>

### OpenTSN 开源项目与国产芯片落地

FlexTSN 后续融入 <a href="https://gitee.com/opentsn">OpenTSN</a>——全球首个 TSN 开源项目，围绕其灵活交换模型发表多篇论文（FooDog，IEEE/ACM ToN 2025；Megabits Down to Kilobits，DAC 2025；Hebo，IWQoS 2024 等）。核心架构进一步融入银河衡芯"枫林一号"HX-DS09 TSN 交换芯片，实现从开源模型到国产芯片的落地。

<div class="proj-figure">
    <img src="{{ '/assets/img/fenglin-chip.png' | relative_url }}" alt="枫林一号 TSN 交换芯片">
</div>
<p class="proj-figure-cap">银河衡芯"枫林一号"HX-DS09 TSN 交换芯片（130 nm 流片）</p>

FlexTSN 于 2019 年开源发布，相关架构采用 130 nm 工艺流片，交换功耗低于 0.5 W。

**芯片落地**：支撑银河衡芯国产 TSN 交换芯片自主可控，相关成果获湖南省教学成果二等奖，并持续服务确定性网络方向的科研与教学。

## FAST — 软硬件协同网络加速教学科研平台

<p class="proj-subtitle">FPGA 转发平面 + CPU 控制平面 · 推广至 8 所高校</p>

**研究背景**：网络系统教学长期面临"理论抽象、实验黑盒"的困境：学生只能配置现成设备，看不到也改不了转发逻辑。FAST 以 FPGA 为转发平面核心、CPU 运行控制平面，把交换机内部流水线完整开放给教学与科研，让学生从"用网络"走向"造网络"。

<div class="proj-figures">
    <img src="{{ '/assets/img/fast-software-arch.png' | relative_url }}" alt="FAST 软件架构">
    <img src="{{ '/assets/img/fast-hardware-arch.png' | relative_url }}" alt="FAST 硬件架构">
</div>
<p class="proj-figure-cap">FAST 软件架构：CPU 上的控制平面与协议栈 ｜ FAST 硬件架构：FPGA 上的可重构转发流水线</p>

作为负责人设计开发的 FAST（FPGA Accelerated Switching plaTform）可重构 SDN 交换平台，已推广至北京邮电大学、电子科技大学、东南大学、北京大学等 8 所高校，支撑计算机网络、路由与交换技术等 4 门以上课程教学，并支撑外部团队发表 2 篇顶会顶刊论文（SIGCOMM、IEEE/ACM ToN）。

**教学影响**：支撑港理工、北大等团队发表 SIGCOMM、IEEE/ACM ToN 等顶会顶刊论文；基于 FAST 的 DNS 劫持与防护教学案例获 CPEC 2026 优秀案例一等奖。项目主页：<a href="https://fast-switch.github.io/">fast-switch.github.io</a>

## 科研项目

主持及参与国家级、省部级与横向科研项目<strong>10 余项</strong>，近五年项目总经费超过 <strong>1000 万元</strong>。研究内容覆盖可编程网络与可编程数据平面、DPU 软硬协同加速、算力网络与在网计算、时间敏感网络以及网络系统安全等方向。

项目类型包括国家级科研项目、省部级人才计划与重点研发课题、全国重点实验室基金、省自然科学基金以及企业横向合作，多数成果已在国产芯片、网卡、交换设备与教学平台中落地。科研经费充足，可为研究生的科研训练、实验平台建设与学术交流提供稳定支持。