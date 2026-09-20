---
layout: page
title: 项目
permalink: /projects/
---

## 代表性系统

<div class="proj-grid">

<div class="proj-card">
    <div class="proj-tag-row">
        <span class="proj-tag tag-blue">开源系统</span>
        <span class="proj-tag tag-blue">FPGA 数据平面</span>
        <span class="proj-tag tag-blue">NSDI 2022</span>
    </div>
    <h3><a href="https://github.com/Winters123/FastRMT">FastRMT</a> · <a href="https://isolation.quest/">Menshen</a></h3>
    <p class="proj-subtitle">首个开源的 FPGA 级 RMT 架构实现与流水线隔离机制</p>
    <div class="proj-bg"><strong>研究背景</strong>：RMT（可重构匹配-动作表）架构自 SIGCOMM 2013 提出以来长期只有闭源商用芯片实现，学术界缺少开放、可修改的 FPGA 级实现作为研究底座；同时，多租户共享可编程设备场景下的模块间隔离机制仍是空白。</div>
    <p>FastRMT 是首个开源的 FPGA 级 RMT（可重构匹配-动作表）架构实现，提供由 Parser、Key Extractor、Lookup Engine、Action Engine 组成的全可编程报文处理流水线，为可编程数据平面研究提供开放基础。</p>
    <div class="proj-figure">
        <img src="{{ '/assets/img/fastrmt-data-flow.png' | relative_url }}" alt="FastRMT 流水线数据流图">
        <div class="proj-figure-cap">FastRMT 多级流水线数据流：PHV 在各级间的提取、匹配与改写</div>
    </div>
    <div class="proj-sub-block">
        <h4>Menshen：可编程流水线的模块间隔离（<a href="https://www.usenix.org/conference/nsdi22/presentation/wang-tao">NSDI 2022</a>）</h4>
        <p>与纽约大学、伦敦玛丽女王大学合作提出。每个报文携带程序 ID（PID），基于 overlays 技术实现<strong>每报文粒度的程序切换</strong>（仅需 2 个时钟周期，不影响流水线吞吐）；通过 daisy chain 重配置路径实现<strong>无中断模块更新</strong>。原型覆盖 NetFPGA 交换机与 Corundum 智能网卡双平台，并完成 ASIC 综合验证。代码已开源：<a href="https://github.com/multitenancy-project/menshen">menshen</a> / <a href="https://github.com/multitenancy-project/menshen-compiler">menshen-compiler</a>。</p>
        <div class="proj-figure">
            <img src="{{ '/assets/img/menshen-daisy-chain.png' | relative_url }}" alt="Menshen 流水线与 daisy chain 重配置路径">
            <div class="proj-figure-cap">Menshen：daisy chain 重配置路径实现无中断模块更新</div>
        </div>
    </div>
    <div class="proj-metrics">
        <div class="proj-metric"><strong>100 Gbps</strong><span>线速报文处理</span></div>
        <div class="proj-metric"><strong>2 cycles</strong><span>每报文程序切换</span></div>
        <div class="proj-metric"><strong>1 GHz / ~6%</strong><span>ASIC 时序 / 额外面积</span></div>
    </div>
    <div class="proj-impact">被 Xilinx OpenNIC、迈普通信网卡、星载交换芯片等核心产品采用</div>
</div>

<div class="proj-card">
    <div class="proj-tag-row">
        <span class="proj-tag tag-blue">芯片落地</span>
        <span class="proj-tag tag-blue">确定性网络</span>
        <span class="proj-tag tag-blue">开源项目</span>
    </div>
    <h3>FlexTSN · <a href="https://gitee.com/opentsn">OpenTSN</a></h3>
    <p class="proj-subtitle">灵活的 TSN 交换实现模型，已融入 OpenTSN 开源项目</p>
    <div class="proj-bg"><strong>研究背景</strong>：TSN 标准体系庞大且持续演进，但长期缺少一种面向 TSN 的通用交换实现模型，关键技术难以快速搭建原型并完成验证；同时国内 TSN 核心芯片长期依赖国外芯片或 IP，舰船、航空航天等高端装备的确定性组网亟需自主可控方案。</div>
    <p>FlexTSN 基于模块化与功能松耦合思想，将 TSN 交换节点解耦为<strong>时间同步、输入调度、分组交换、输出调度、资源与状态管理</strong>五大功能模块，提出 AIAO（Any-In-Any-Out）可编程调度原语，符合 IEEE 802.1Qbv 标准，支持 TSN 交换机的快速重构与关键技术敏捷验证。</p>
    <div class="proj-figures">
        <img src="{{ '/assets/img/flextsn-node.png' | relative_url }}" alt="FlexTSN 交换节点功能模块">
        <img src="{{ '/assets/img/fenglin-chip.png' | relative_url }}" alt="枫林一号 HX-DS09 TSN 芯片实物与测试板">
    </div>
    <div class="proj-figure-cap">左：FlexTSN 交换节点五大功能模块；右：基于 OpenTSN 研制的"枫林一号"HX-DS09 芯片实物与测试板</div>
    <p>FlexTSN 后续融入 <a href="https://gitee.com/opentsn">OpenTSN</a>——全球首个 TSN 开源项目（2.0 → 4.0 持续演进，集成 TSNBuilder 流量规划与测试工具链），支撑发表 CCF ToN、《计算机研究与发展》等多篇学术论文。</p>
    <div class="proj-metrics">
        <div class="proj-metric"><strong>AIAO</strong><span>可编程调度原语</span></div>
        <div class="proj-metric"><strong>5 大模块</strong><span>松耦合解耦</span></div>
        <div class="proj-metric"><strong>0.46 W</strong><span>枫林一号芯片功耗</span></div>
    </div>
    <div class="proj-impact">核心架构融入银河衡芯"枫林一号"HX-DS09 国产 TSN 交换芯片（130 nm 流片，亚微秒级时钟同步精度），支撑国产芯片自主可控</div>
</div>

<div class="proj-card">
    <div class="proj-tag-row">
        <span class="proj-tag tag-blue">开源框架</span>
        <span class="proj-tag tag-blue">教学平台</span>
    </div>
    <h3><a href="https://fast-switch.github.io/">FAST Framework</a></h3>
    <p class="proj-subtitle">以 FPGA 为转发平面核心的可重构 SDN 交换架构（IWQoS 2019）</p>
    <div class="proj-bg"><strong>研究背景</strong>：网络硬件系统的教学与科研门槛高——NetFPGA 等平台代码与具体硬件紧耦合、模块复用率低、调试复杂，学生与研究者难以快速构建自定义交换系统，亟需平台无关、模块可重构的软硬件协同框架。</div>
    <p>FAST（FPGA bAsed SDN swiTching）将报文处理流程拆解为多个独立处理阶段，为每个阶段建立标准模块库，开发者可自由组合处理模块、"离线重构"报文处理流水线，实现软硬件协同的网络加速，显著降低网络硬件系统的学习与开发门槛。</p>
    <div class="proj-figures">
        <img src="{{ '/assets/img/fast-software-arch.png' | relative_url }}" alt="FAST 软件架构">
        <img src="{{ '/assets/img/fast-hardware-arch.png' | relative_url }}" alt="FAST 硬件架构">
    </div>
    <div class="proj-metrics">
        <div class="proj-metric"><strong>8 所</strong><span>一流院校采用</span></div>
        <div class="proj-metric"><strong>离线重构</strong><span>模块库流水线</span></div>
        <div class="proj-metric"><strong>SIGCOMM / ToN</strong><span>成果支撑</span></div>
    </div>
    <div class="proj-impact">推广至北京邮电大学、电子科技大学、东南大学、北京大学等 8 所一流院校，支撑香港理工、北大等团队发表顶会顶刊论文</div>
</div>

</div>

## 承担科研项目（部分）

<div class="fund-list">


<div class="fund-item">
    <div class="fund-main">
        <span class="fund-role role-pi">主持</span>
        <span class="fund-title">电信网****关键技术（国家级项目）</span>
    </div>
    <div class="fund-meta"><span class="fund-period">2026.06 – 2029.06</span><span class="fund-amount">150 万元</span></div>
</div>

<div class="fund-item">
    <div class="fund-main">
        <span class="fund-role role-pi">主持</span>
        <span class="fund-title">湖南省芙蓉计划青年人才（省部级）</span>
    </div>
    <div class="fund-meta"><span class="fund-period">2026.06 – 2029.06</span><span class="fund-amount">30 万元</span></div>
</div>

<div class="fund-item">
    <div class="fund-main">
        <span class="fund-role role-pi">子课题负责人</span>
        <span class="fund-title">电信*****技术（横向项目）</span>
    </div>
    <div class="fund-meta"><span class="fund-period">2024.10 – 2026.09</span><span class="fund-amount">200 / 480 万元</span></div>
</div>


<div class="fund-item">
    <div class="fund-main">
        <span class="fund-role role-sub">子课题负责人</span>
        <span class="fund-title">新型韧性***信息通信系统与演示验证（省部级）</span>
    </div>
    <div class="fund-meta"><span class="fund-period">2022.11 – 2024.05</span><span class="fund-amount">120 / 360 万元</span></div>
</div>



<div class="fund-item">
    <div class="fund-main">
        <span class="fund-role role-sub">课题负责人</span>
        <span class="fund-title">数据中心**网络处理加速技术（省部级）</span>
    </div>
    <div class="fund-meta"><span class="fund-period">2023.08 – 2026.05</span><span class="fund-amount">75 / 300 万元</span></div>
</div>



<div class="fund-item">
    <div class="fund-main">
        <span class="fund-role role-pi">主持</span>
        <span class="fund-title">国防科大第三批高层次人才青年英才</span>
    </div>
    <div class="fund-meta"><span class="fund-period">2022.06 – 2026.06</span><span class="fund-amount">40 万元</span></div>
</div>

<div class="fund-item">
    <div class="fund-main">
        <span class="fund-role role-pi">主持</span>
        <span class="fund-title">面向DPU的软硬协同加速机理研究（全国重点实验室基金）</span>
    </div>
    <div class="fund-meta"><span class="fund-period">2022.01 – 2023.12</span><span class="fund-amount">30 万元</span></div>
</div>

<div class="fund-item">
    <div class="fund-main">
        <span class="fund-role role-pi">主持</span>
        <span class="fund-title">弹性智能网络多路径并行传输关键技术（科工局重点实验室基金）</span>
    </div>
    <div class="fund-meta"><span class="fund-period">2023.01 – 2024.12</span><span class="fund-amount">20 万元</span></div>
</div>

<div class="fund-item">
    <div class="fund-main">
        <span class="fund-role role-pi">主持</span>
        <span class="fund-title">算网融合的 DPU 异构协同处理模型及机理研究（省自然科学基金）</span>
    </div>
    <div class="fund-meta"><span class="fund-period">2023.01 – 2025.12</span><span class="fund-amount">5 万元</span></div>
</div>

</div>
