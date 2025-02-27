---
title: 分布式系统 in 2010s ：存储之数据库篇
author: ['黄东旭']
date: 2019-12-26
summary: 无论哪个时代，存储都是一个重要的话题，今天先聊聊数据库。
tags: ['分布式系统前沿技术']
---

回看这几年，分布式系统领域出现了很多新东西，特别是云和 AI 的崛起，让这个过去其实不太 sexy 的领域一下到了风口浪尖，在这期间诞生了很多新技术、新思想，让这个古老的领域重新焕发生机。站在 2010s 的尾巴上，我想跟大家一起聊聊分布式系统令人振奋的进化路程，以及谈一些对 2020s 的大胆猜想。

无论哪个时代，存储都是一个重要的话题，今天先聊聊数据库。在过去的几年，数据库技术上出现了几个很明显的趋势。

## 存储和计算进一步分离

我印象中最早的存储-计算分离的尝试是 Snowflake，Snowflake 团队在 2016 年发表的论文[《The Snowflake Elastic Data Warehouse》](http://pages.cs.wisc.edu/~remzi/Classes/739/Spring2004/Papers/p215-dageville-snowflake.pdf)是近几年我读过的最好的大数据相关论文之一，尤其推荐阅读。Snowflake 的架构关键点是在无状态的计算节点 + 中间的缓存层 + S3 上存储数据，计算并不强耦合缓存层，非常符合云的思想。从最近 AWS 推出的 RedShift 冷热分离架构来看，AWS 也承认 Snowflake 这个搞法是先进生产力的发展方向。另外这几年关注数据库的朋友不可能不注意到 Aurora。不同于 Snowflake，Aurora 应该是第一个将存储-计算分离的思想用在 OLTP 数据库中的产品，并大放异彩。Aurora 的成功在于将数据复制的粒度从 Binlog降低到 Redo Log ，极大地减少复制链路上的 IO 放大。而且前端复用了 MySQL，基本做到了 100% 的应用层 MySQL 语法兼容，并且托管了运维，同时让传统的 MySQL 适用范围进一步拓展，这在中小型数据量的场景下是一个很省心的方案。

虽然 Aurora 获得了商业上的成功，但是从技术上，我并不觉得有很大的创新。熟悉 Oracle 的朋友第一次见 Aurora 的架构可能会觉得和 RAC 似曾相识。Oracle 大概在十几年前就用了类似的方案，甚至很完美的解决了 Cache Coherence 的问题。另外，Aurora 的 Multi-Master 还有很长的路要走，从最近在 ReInvent 上的说法来看，目前 Aurora 的 Multi-Master 的主要场景还是作为 Single Writer 的高可用方案，本质的原因应该是目前 Multi-Writer 采用乐观冲突检测，冲突检测的粒度是 Page，在冲突率高的场合会带来很大的性能下降。

我认为 Aurora 是一个很好的迎合 90% 的公有云互联网用户的方案：100% MySQL 兼容，对一致性不太关心，读远大于写，全托管。但同时，Aurora 的架构决定了它放弃了 10% 有极端需求的用户，如全局的 ACID 事务+ 强一致，Hyper Scale（百 T 以上，并且业务不方便拆库），需要实时的复杂 OLAP。这类方案我觉得类似 TiDB 的以 Shared-nothing 为主的设计才是唯一的出路。作为一个分布式系统工程师，我对任何不能水平扩展的架构都会觉得不太优雅。

## 分布式 SQL 数据库登上舞台，ACID 全面回归

回想几年前 NoSQL 最风光的时候，大家恨不得将一切系统都使用 NoSQL 改造，虽然易用性、扩展性和性能都不错，但是多数 NoSQL 系统抛弃掉数据库最重要的一些东西，例如 ACID 约束，SQL 等等。NoSQL 的主要推手是互联网公司，对于互联网公司的简单业务加上超强的工程师团队来说当然能用这些简单工具搞定。

但最近几年大家渐渐发现低垂的果实基本上没有了，剩下的都是硬骨头。

最好的例子就是作为 NoSQL 的开山鼻祖，Google 第一个搞了 NewSQL （Spanner 和 F1）。在后移动时代，业务变得越来越复杂，要求越来越实时，同时对于数据的需求也越来越强。尤其对于一些金融机构来说，一方面产品面临着互联网化，一方面不管是出于监管的要求还是业务本身的需求，ACID 是很难绕开的。更现实的是，大多数传统公司并没有像顶级互联网公司的人才供给，大量历史系统基于 SQL 开发，完全迁移到 NoSQL 上肯定不现实。

在这个背景下，分布式关系型数据库，我认为这是我们这一代人，在开源数据库这个市场上最后一个 missing part，终于慢慢流行起来。这背后的很多细节由于篇幅的原因我就不介绍，推荐阅读 PingCAP TiFlash 技术负责人 maxiaoyu 的一篇文章《[从大数据到数据库](https://zhuanlan.zhihu.com/p/97085692)》，对这个话题有很精彩的阐述。

## 云基础设施和数据库的进一步整合

在过去的几十年，数据库开发者都像是在单打独斗，就好像操作系统以下的就完全是黑盒了，这个假设也没错，毕竟软件开发者大多也没有硬件背景。另外如果一个方案过于绑定硬件和底层基础设施，必然很难成为事实标准，而且硬件非常不利于调试和更新，成本过高，这也是我一直对定制一体机不是太感兴趣的原因。但是云的出现，将 IaaS 的基础能力变成了软件可复用的单元，我可以在云上按需地租用算力和服务，这会给数据库开发者在设计系统的时候带来更多的可能性，举几个例子：

1. Spanner 原生的 TrueTime API 依赖原子钟和 GPS 时钟，如果纯软件实现的话，需要牺牲的东西很多（例如 CockroachDB 的 HLC 和 TiDB 的改进版 Percolator 模型，都是基于软件时钟的事务模型）。但是长期来看，不管是 AWS 还是 GCP 都会提供类似 TrueTime 的高精度时钟服务，这样一来我们就能更好的实现低延迟长距离分布式事务。

2. 可以借助 Fargate + EKS 这种轻量级容器 + Managed K8s 的服务，让我们的数据库在面临突发热点小表读的场景（这个场景几乎是 Shared-Nothing 架构的老大难问题），比如在 TiDB 中通过 Raft Learner 的方式，配合云的 Auto Scaler 快速在新的容器中创建只读副本，而不是仅仅通过 3 副本提供服务；比如动态起 10 个 pod，给热点数据创建 Raft 副本（这是我们将 TiKV 的数据分片设计得那么小的一个重要原因），处理完突发的读流量后再销毁这些容器，变成 3 副本。

3. 冷热数据分离，这个很好理解，将不常用的数据分片，分析型的副本，数据备份放到 S3 上，极大地降低成本。

4. RDMA/CPU/超算 as a Service，任何云上的硬件层面的改进，只要暴露 API，都是可以给软件开发者带来新的好处。

例子还有很多，我就不一一列举了。总之我的观点是云服务 API 的能力会像过去的代码标准库一样，是大家可以依赖的东西，虽然现在公有云的 SLA 仍然不够理想，但是长远上看，一定是会越来越完善的。

所以，数据库的未来在哪里？是更加的垂直化还是走向统一？对于这个问题，我同意这个世界不存在银弹，但是我也并不像我的偶像，AWS 的 CTO，Vogels 博士那么悲观，相信未来是一个割裂的世界（AWS 恨不得为了每个细分的场景设计一个数据库）。过度地细分会加大数据在不同系统中流动的成本。解决这个问题有两个关键：

1. 数据产品应该切分到什么粒度？

2. 用户可不可以不用知道背后发生了什么？

第一个问题并没有一个明确的答案，但是我觉得肯定不是越细越好的，而且这个和 Workload 有关，比如如果没有那么大量的数据，直接在 MySQL 或者 PostgreSQL 上跑分析查询其实一点问题也没有，没有必要非去用 Redshift。虽然没有直接的答案，但是我隐约觉得第一个问题和第二个问题是息息相关的，毕竟没有银弹，就像 OLAP 跑在列存储引擎上一定比行存引擎快，但是对用户来说其实可以都是 SQL 的接口。

SQL 是一个非常棒的语言，它只描述了用户的意图，而且完全与实现无关，对于数据库来说，其实可以在 SQL 层的后面来进行切分，在 TiDB 中，我们引入 TiFlash 就是一个很好的例子。动机很简单：

1. 用户其实并不是数据库专家，你不能指望用户能 100% 在恰当的时间使用恰当的数据库，并且用对。

2. 数据之间的同步在一个系统之下才能尽量保持更多的信息，例如，TiFlash 能保持 TiDB 中事务的 MVCC 版本，TiFlash 的数据同步粒度可以小到 Raft Log 的级别。

另外一些新的功能仍然可以以 SQL 的接口对外提供，例如全文检索，用 SQL 其实也可以简洁的表达。这里我就不一一展开了。

我其实坚信系统一定是朝着更智能、更易用的方向发展的，现在都 21 世纪了，你是希望每天拿着一个 Nokia 再背着一个相机，还是直接一部手机搞定？

>本文是「分布式系统前沿技术」专题文章，目前该专题在持续更新中，欢迎大家保持关注。
