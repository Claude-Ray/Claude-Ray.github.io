---
title: Node.js APM 产品调研
date: 2019-05-19 21:52:35
tags: [Node.js,APM]
categories: Node.js
description: 近期调研了当下 Node.js 主流的 APM 产品，为期不到一周，本文以介绍基本状况为主，引申对 APM 中间件选型乃至自研的一些思考。APM 虽和业务关系紧密，但实属于运维管理范畴，笔者从业务的角度思考，易受限于使用场景，请路过的读者及时指教问题。
---

# 前言

[Application Performance Management](https://en.wikipedia.org/wiki/Application_performance_management)（简称 APM）是监控服务的一套技术手段，致力于监控并管理程序的性能和可用性。

不妨思考一下，当有用户反映操作无响应，如何排查问题？惯例是先自己尝试重现，如果没重现，再换一个人试试，bla bla...

从用户击下按键开始，到服务呈现最终效果的过程中，有哪些因素会导致阻塞？

是否真的未响应？有没有可能是网络过慢导致？

丢包？用户网络质量还是机房故障？出现在机群哪一层？

代码BUG？自己的还是别人的？

能快速定位是哪一环节出了问题吗？

用户的请求从客户端 -> CDN -> 代理 -> 中间层 -> 服务1 -> 服务2 -> ...

线上应用的性能表现是极为复杂的，运维的感知总是慢于业务，人工监控或草木皆兵、或亡羊补牢。因此需要一个合理的监控措施以便诊断服务质量，提高运维和业务的工作效率，间接服务于提升用户体验。

尽管上面的例子和请求链路相关，许多人就将 APM 和 `分布式链路跟踪系统` 混为一谈，其实并不恰当。 APM 当然可以承载 Tracing 工作，但除此之外还包含内存、CPU、RT、TPS、QPS等等监控职能，链路分析仅是其中一环罢了。为了更清晰地阐述，下面先简单介绍一下 APM 的基本定义。

## 关于 APM
APM 是 `Gartner` 抽象出的一个管理模型，有如下定义。

![APM Coneptual Framework](/image/node-apm-product-research/apm-coneptual-framework.jpg)

通俗的说法如下

1. 终端用户体验：反馈真实用户的体验，包括高峰时的服务、组件平均响应时间。
2. 应用架构映射：能否分析真实请求链路。
3. 应用事务分析：要求有序、完整地记录事务信息，能够定位两个操作是否为同一个用户，且信息具备唯一性。
4. 深度应用诊断：用户反馈问题时，能精准定位问题点，通常需要做更底层的监控。同时又有着部署简单、副作用低的要求，这是 APM 应用的主要难点所在。
5. 分析与报告：数据要实时且精准，大数据的存储与查询，目前已经不难应对。

2016 年， Gartner 又将上述 5 个维度更新为 3 个新维度。

- 数字体验监控 (DEM)：对应用户体验监控
- 应用发现、跟踪、诊断 (ADTD)：整合并了应用架构映射、事务分析、深度应用诊断
- 应用分析 (AA)：对应分析与报告

除了 DEM ，基本概念和旧维度呈对应关系，为了方便理解，下面提一下 Gartner 对 DEM 的定义。

> an availability and performance monitoring discipline that supports the optimization of the operational experience and behavior of a digital agent, human or machine, as it interacts with enterprise applications and services. For the purposes of this evaluation, it includes real-user monitoring (RUM) and synthetic transaction monitoring (STM) for both web- and mobile-based end users.

实质依然是通过用户的角度分析关键业务（RUM），通过测试消除潜在的错误和性能瓶颈（STM），以数字化增强监控分析能力。

# 主流 APM 简介

## 商业软件
### New Relic
> https://newrelic.com/nodejs

New Relic 是专研 APM 的代表性公司，并凭借其技术产品于 2014 年上市。抛开其他语言市场的激烈角逐，它在 Node.js APM 产业是真正的龙头。虽然监控服务需要付费，数据上传到云端才能使用，但其 SDK 源代码完全开放，可以清晰地看到它对各探针的实现，对接入方开发者十分友好，也因此成为各监控服务的模仿对象。

是付费用户的首选。

### AppDynamics
> https://www.appdynamics.com/nodejs

AppDynamics 一直是 New Relic 的竞争对手，有意思的是，两家公司的创始人分别是来自同一家公司的首席架构师、CEO ，并在同一年创立。

AppDynamics 公司在 2016 年上市，它提供的服务也非常强大，但和 New Relic 的市场定位不同，New Relic 初期战略主要针对小型创业公司，而 AppDynamics 则专攻企业。且 AppDynamics 可以部署在公司内部数据中心，而不只是作为云端服务。

其 SDK 代码不完全开放，使用了经过编译的 jar 包和二进制文件。

### Dynatrace
> https://www.dynatrace.com/technologies/nodejs-monitoring/

Dynatrace 和 New Relic、AppDynamics 并称为 APM 产业三大领军者，但在 Node.js 的市场占有率和热度很低，不开放源代码，无法深度化定制，难以认可它已经是成熟的产品。

### Atatus
> https://www.atatus.com/for/nodejs

支持功能一般，只能算二线产品，况且其 SDK 做了代码混淆。

### 听云
> https://doc.tingyun.com/server/html/node/install.html

国内团队，可以限量免费试用。估计很多人体验过，其很多功能（代码）借鉴自 New Relic，试用体验相仿，好在针对国内市场做了一些本地化，不过多介绍。

有 New Relic 做“后盾”，服务不会差到哪去，且提供免费版，小型项目不妨一用。

### OneAPM
> https://www.oneapm.com/ai/nodejs.html

类似听云，但给人的感觉是实力逊于听云，SDK 几乎照搬 New Relic，口碑毁于肆意打广告，不太讨喜。

## 开源免费方案

### Alinode
由于这个方案存在较多争议，因此额外给了一些篇幅补充介绍。

#### 开发者介绍
hyj1991 解释过 Alinode 对 Node Runtime 增加了哪些改动：

- 增加了一些 V8 没有对外暴露的接口，比如 GC Trace 来动态输出 GC 日志
- 埋了一些点以性能损耗更低的方式采集进程级别的 CPU 和 Memory 数据
- 增加了动态开启 CPU / Memory / GC 状态采集的开关

而对于负责开发者业务相关的 API 和功能逻辑，并无任何改动，这也是为什么 AliNode 和官方的 Runtime 可以无缝对切的原因。

至于安全问题，主要是担心会采集业务数据上报，但是实际上 AliNode 内核的上述改动，都不会直接向云端发送任何数据，而都是以本地 Log 的方式写入大家配置的 NODE_LOG_DIR 目录下，日志文件以 `node-日志.log` 的形式命名，不放心的话可以查看此文件内容。

实际上大家在控制台看到的数据，最后不管使用的是 egg-alinode 还是 agenthub 均是通过 agentx 这个库采集上报的，这个库首先它是开源的，大家可以自行阅读相关采集代码观察是否上报了敏感数据。最后实在对安全问题存在疑虑的，可以通过 Wireshark 等抓包工具，来抓取 AliNode 输出的日志和 Agentx 上报的数据内容，看看是否上报的数据中存在大家非常担心的敏感数据。

#### 小结
严格来说 Alinode 不能算作开源方案，但其实它只有后台和启动代码是闭源的，尤其对小团队而言接入成本低，是非常值得考虑的接入方案。

而且由于日志收集进程开源，除了没有开源的收集 v8 内部性能的功能，其他信息都可以通过接入自己的日志系统或后台进行分析。

但从企业角度出发，还是完全由自己掌控的方案更放心。

### Easy Monitor
> https://github.com/hyj1991/easy-monitor

终于提到了一个完全开源的方案了，它的开发者同样是 hyj1991，可惜功能实在简单，仅提供性能监控，维护度较低。

目前最大的价值是作为学习项目，而不是投入生产环境。

### Pandora.js
> https://midwayjs.org/pandora/zh-cn/

来自阿里 midwayjs 团队，是类似于 PM2 的一个启动进程，因此最大优点是无代码侵入。

目前功能尚不完备，支持star、restart、stop，调试时为了查看 debugger 记录，可加上 `--inspect` 参数，暂不支持平滑 reload。

Dashboard 只能单机部署单机监控，无法集群监控，目前 midway 的使用方案是结合 ElasticSearch。

据说已经在阿里内部落地，现在正重构 2.0 版本，等待此项目成熟可考虑使用。很希望能替代臃肿的 PM2。

### Prometheus
> https://prometheus.io/

在国外非常流行，相比业务方，更多地被运维熟知，是一种监控和报警的开源生态。SDK 和界面有多重组合方式，Node.js 一般结合 `prom-client`(非官方 npm 包) + `Granfana` 使用。

只做性能采集，不支持 trace 跟踪。目前已知缺陷是内存占用较高和日志量巨大，数据可以选择本地存储或远程接口存储。

开源的一大选择方案，落地可能对运维团队要求较高。

### Elastic APM
> https://www.elastic.co/solutions/apm

- APM: https://www.elastic.co/guide/en/apm/get-started/current/index.html
- Node Client: https://www.elastic.co/guide/en/apm/agent/nodejs/current/index.html
- Kibana APM: https://www.elastic.co/guide/en/kibana/current/xpack-apm.html

这是 Elastic 体系下的完全开源的 APM 解决方案，也提供商业付费服务。文档一如既往地丰富，上面列举了其中三个入口。

日志采集进程为 golang 编写的 apm-server，最终将数据存储到 ElasticSearch，Kibana 内置了 APM 基础看板。

官方提供 API 来支持深度定制，golang 降低了二次开发的成本，更不用担心 Kibana 看板功能不够用。总之，是相当全面的解决方案，之后会单独开一篇文章介绍。

# 选型概述

## 要点维度
### 性能监控
进程级的 CPU、内存指标监控，这是 APM 最基本功能，普遍支持。

更高级的是 V8 监控，heap 信息，做出 profile 等诊断。但实际上 Node 最新版本已经暴露出 V8 heap 的接口，profile 也完全可以需要时主动创建，重要性反而不是特别高。

### 代码级监控
监控到代码细节，不是简单的错误定位，而是分析哪段代码有内存泄露、提供 SQL 慢查询日志等更实用的功能。

### 事务监控
分析业务流程、请求响应时间等。

### 框架支持
要实现对 npm 依赖的监控，例如路由的追踪，是需要明确定制化到特定包的，例如 express、koa、bluebird、sequelize 等。

这里只考虑了主流的 Express 和 Koa HTTP 框架，有特殊需要的请进一步到官方了解。

### 链路追踪
一个请求从创建到响应的链路分析。

### 分布式部署
能否识别集群的部署状况。

### 代码侵入
对代码侵入程度越低，接入越方便，避免对业务造成影响。

虽然移除一个模块时代码的改动大小是评判侵入性高低的一般标准，并且下面也将遵循此标准来进行评价。但必须留意，除了引入 SDK 的方式，如何使用 async hook、如何对第三方库做 Patch(类似 Java 动态字节码探针)，再低的侵入都可能埋下隐患。

### 社区活跃度
社区活力是很重要的软指标，但我们拿不到服务商未公开的统计数据，只能以 npm 最近的下载量作为参考。

### 数据安全
只有开源和小部分商业方案支持数据内网存储，这样才能最大地保障数据安全。

但如果项目本身不设置重要数据，这个问题就得结合自身情况重新考量。

### 外部依赖
如果选择自行存储数据，就需要考虑部署数据库等依赖的成本。

方案能否落地，取决于项目相关的数据团队、运维团队，此维度也请各位自行评估。

## 关于 Node.js
Node.js 在服务端尤其是中间层表现不俗，但如果经验不足的情况下将 Node 落地，没有任何监控，承担的风险太大。

小团队的发布流程和运维支撑通常较弱，难以避免线上故障，这一阶段，快速响应和处理才能降低损失。

考虑到 APM 的接入成本，有必要分清接入功能的优先级。

在虚拟化、容器化的现在，大部分性能指标的采集工作都可以在业务之外进行，因此性能采集只要能做到进程级的监控即可。

分布式全链路追踪，不是只在 Node 一端接入就有效果的，必然要结合运维和后端技术栈，选择使用 Zipkin 等更流行的 opentracing 开源方案。

如果上面的观点你还算认可，那么我们选型上应该更关注 APM 服务对代码级监控的支持力度、侵入程度、社区活跃度和接入成本。

## 选型
综上，APM 的选型参考如下

|名称|Express|Koa|性能|代码级|事务|链路|分布式|侵入|实现方式|npm周下载量(+)|
|-|-|-|-|-|-|-|-|-|-|-|
|newrelic     |✓|✓|✓|✓|✓|✓|✓|低|探针|30.1k|
|appdynamics  |✓|✓|✓|✓|✓|✓|✓|低|探针|9.5k|
|dynatrace    |✓|✓|✓|✓|✓|✓|✓|低|探针|0.1k|
|atatus       |✓|✓|✓|×|✓|✓|✓|低|探针|0.6k|
|tingyun      |✓|✓|✓|✓|✓|✓|✓|低|探针|0.3k|
|one apm      |✓|×|✓|✓|✓|✓|✓|低|探针|0.1k|
|alinode      |✓|✓|✓|✓|✓|✓|✓|无|run time|-|
|easy monitor |✓|✓|✓|×|×|×|✓|低|探针|0.1k|
|pandora      |✓|✓|✓|×|✓|✓|×|极低|进程启动器|0.2k|
|Prometheus   |✓|✓|✓|×|✓|✓|✓|低|探针|20.1k|
|elastic apm  |✓|✓|✓|✓|✓|✓|✓|低|探针|24.6k|

## 自研
普遍场景下，在日志层面做好业务监控即可，参考下面的流程，只要不涉及 Stack、SQL 等代码级监控，就能以较低开发成本、低侵入式地实现核心业务监控。

1. 基于 Node.js 自带的 API 完成探针，并在关键业务做好相关埋点，将数据(日志)打点到 Kafka。
2. 另起服务进程，专门消费 Kafka 的监控队列，推送到 ElasticSearch 服务中。
3. 通过 Kibana 的看板完成一系列定制化的数据分析。

如果是更复杂的需求，则不建议造轮子，而是基于开源方案二次开发。

# 结论
无论如何，我更倾向选用开源方案，但目前开原方案的代码级监控都是短板，那么这也成为自研和深度定制的困难所在。

末尾重申，APM 属于运维技术的范畴，业务方自己搭建时，最好结合自身需求定制，当心过犹不及。

# Reference
- [维基百科-Application performance management](https://en.wikipedia.org/wiki/Application_performance_management)
- [什么是真正的APM？](http://network.51cto.com/art/201503/469273.htm)
- [关于Nodejs的性能监控思考？](https://www.zhihu.com/question/315261661)
