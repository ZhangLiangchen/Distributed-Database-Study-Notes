# TiDB 与 OceanBase 分布式数据库底层架构对比研究

<div align="center">

**——从分片、共识、存储到事务、HTAP、存算分离与云原生的源码级横向剖析**

技术研究报告　·　融合学习版

版本基准:TiDB 8.5.x LTS　|　OceanBase 4.2.5_CE / 4.3.5 / 4.4.x　|　TiFlash 7.0 实验 / 7.4 GA

</div>

---

## 摘要

本文以源码级、可追溯、可证伪的方式,对 TiDB(主基准 8.5.x LTS)与 OceanBase(4.2.5_CE / 4.3.5 / 4.4.x)两套国产分布式数据库的底层架构,做 25 个维度的横向对比,覆盖分片、路由、共识、存储引擎、KV/LSM 与关系模型映射、无状态 SQL 层、存算分离、性能、分布式事务、MVCC 与读路径、SQL 优化器、元数据与控制面、DDL、全局/本地索引、HTAP、多租户、容灾高可用、运维观测、兼容性、安全、云原生与 benchmark 设计。每一维度均从**数据路径、控制路径、故障路径**三个层面剖析两者的实现机制、组件调用关系与工程取舍,并以可信度标签(官方实现 / 协议理论 / 工程推测 / 不确定)区分结论强度,源码引用全部 commit-pinned。研究表明:TiDB 以「Multi-Raft over 自动 KV Range 分片 + 无状态计算层」为主线,OceanBase 以「关系对象分区 + Tablet + Log Stream(PALF 复制式 WAL)+ 一体化 OBServer」为主线;二者不是先进性排序,而是在自动化分片、计算弹性、单机执行效率、强一致故障恢复等维度上的不同工程取舍。本文同时给出每一维度的反例与代价、测试开发视角的可验证点,以及高风险事实(版本归属、SPOF、RPO/RTO、benchmark 成绩等)的逐项裁决。

**关键词**:分布式数据库;TiDB;OceanBase;Multi-Raft;Paxos;PALF;存算分离;HTAP;分布式事务;架构对比

## Abstract

This report provides a source-level, traceable, and falsifiable comparison of the underlying architectures of two distributed databases — TiDB (baseline 8.5.x LTS) and OceanBase (4.2.5_CE / 4.3.5 / 4.4.x) — across 25 dimensions, including sharding, routing, consensus, storage engine, KV/LSM-to-relational mapping, stateless SQL layer, compute-storage disaggregation, performance, distributed transactions, MVCC and read path, query optimizer, metadata control plane, online DDL, global/local indexing, HTAP, multi-tenancy, disaster recovery, observability, compatibility, security, cloud-native deployment, and benchmark design. Each dimension is dissected along the **data path, control path, and failure path**, with credibility labels (official implementation / protocol theory / engineering inference / uncertain) marking the strength of each claim and all source-code references commit-pinned. The study shows that TiDB centers on "Multi-Raft over auto-split KV ranges plus a stateless compute layer," whereas OceanBase centers on "relational partitioning plus Tablet plus Log Stream (PALF replicated WAL) plus an all-in-one OBServer." The two are not ranked by advancement but reflect different engineering trade-offs across auto-sharding, compute elasticity, single-node efficiency, and strongly-consistent recovery. Counter-examples, costs, test-engineering verification points, and per-item adjudications of high-risk facts (version attribution, SPOF, RPO/RTO, benchmark results) are provided throughout.

**Keywords**: distributed database; TiDB; OceanBase; Multi-Raft; Paxos; PALF; compute-storage disaggregation; HTAP; distributed transaction; architecture comparison

## 阅读约定

全书 25 章,每章固定 13 节:N.1 本章核心问题、N.2 TiDB 的实现、N.3 OceanBase 的实现、N.4 核心差异对比、N.5 正常路径图、N.6 故障/异常路径图、N.7 性能·可靠性·运维影响、N.8 反例与代价、N.9 测试开发视角的验证点、N.10 容易误解点、N.11 本章结论、N.12 参考文献、N.13 信息可信度自评。图题置于图下、表题置于表上,按章编号(图 N-x / 表 N-x);正文引用以 〔文献[N]〕 标注,指向本章 §N.12;全书引用总表见附录 C。版本与高风险事实裁决见附录 B / 附录 F。

---

### 目录

#### 正文

- **第 1 章　分片**
    - 1.1　本章核心问题
    - 1.2　TiDB 的实现
    - 1.3　OceanBase 的实现
    - 1.4　核心差异对比
    - 1.5　正常路径图
    - 1.6　故障/异常路径图
    - 1.7　性能、可靠性、运维影响
    - 1.8　反例与代价
    - 1.9　测试开发视角的验证点
    - 1.10　容易误解点
    - 1.11　本章结论
    - 1.12　参考文献
    - 1.13　信息可信度自评

- **第 2 章　路由**
    - 2.1　本章核心问题
    - 2.2　TiDB 的实现
    - 2.3　OceanBase 的实现
    - 2.4　核心差异对比
    - 2.5　正常路径图
    - 2.6　故障/异常路径图
    - 2.7　性能、可靠性、运维影响
    - 2.8　反例与代价
    - 2.9　测试开发视角的验证点
    - 2.10　容易误解点
    - 2.11　本章结论
    - 2.12　参考文献
    - 2.13　信息可信度自评

- **第 3 章　共识**
    - 3.1　本章核心问题
    - 3.2　TiDB 的实现：Multi-Raft over raft-rs
    - 3.3　OceanBase 的实现：Multi-Paxos over PALF
    - 3.4　核心差异对比
    - 3.5　正常路径图
    - 3.6　故障/异常路径图
    - 3.7　性能、可靠性、运维影响
    - 3.8　反例与代价
    - 3.9　测试开发视角的验证点
    - 3.10　容易误解点
    - 3.11　本章结论
    - 3.12　参考文献
    - 3.13　信息可信度自评

- **第 4 章　存储引擎**
    - 4.1　本章核心问题
    - 4.2　TiDB / TiKV 的实现
    - 4.3　OceanBase 的实现
    - 4.4　核心差异对比
    - 4.5　正常路径图
    - 4.6　故障 / 异常路径图
    - 4.7　性能、可靠性、运维影响
    - 4.8　反例与代价
    - 4.9　测试开发视角的验证点
    - 4.10　容易误解点
    - 4.11　本章结论
    - 4.12　参考文献
    - 4.13　信息可信度自评

- **第 5 章　KV / LSM 与关系模型映射**
    - 5.1　本章核心问题
    - 5.2　TiDB 的实现：关系 → 不透明 KV → LSM
    - 5.3　OceanBase 的实现：关系原生 SSTable，而非 SQL-on-KV
    - 5.4　核心差异对比
    - 5.5　正常路径图(读 / 写 + SQL→KV 翻译)
    - 5.6　故障 / 异常路径图
    - 5.7　性能、可靠性、运维影响
    - 5.8　反例与代价
    - 5.9　测试开发视角的验证点
    - 5.10　容易误解点
    - 5.11　本章结论
    - 5.12　参考文献
    - 5.13　信息可信度自评

- **第 6 章　无状态 SQL 层**
    - 6.1　本章核心问题
    - 6.2　TiDB 的实现
    - 6.3　OceanBase 的实现
    - 6.4　核心差异对比
    - 6.5　正常路径图
    - 6.6　故障/异常路径图
    - 6.7　性能、可靠性、运维影响
    - 6.8　反例与代价
    - 6.9　测试开发视角的验证点
    - 6.10　容易误解点
    - 6.11　本章结论
    - 6.12　参考文献
    - 6.13　信息可信度自评

- **第 7 章　存算分离**
    - 7.1　本章核心问题
    - 7.2　TiDB 的实现
    - 7.3　OceanBase 的实现
    - 7.4　核心差异对比
    - 7.5　正常路径图
    - 7.6　故障/异常路径图
    - 7.7　性能、可靠性、运维影响
    - 7.8　反例与代价
    - 7.9　测试开发视角的验证点
    - 7.10　容易误解点
    - 7.11　本章结论
    - 7.12　参考文献
    - 7.13　信息可信度自评

- **第 8 章　性能**
    - 8.1　本章核心问题
    - 8.2　TiDB 的实现
    - 8.3　OceanBase 的实现
    - 8.4　核心差异对比
    - 8.5　正常路径图
    - 8.6　故障/异常路径图
    - 8.7　性能、可靠性、运维影响
    - 8.8　反例与代价
    - 8.9　测试开发视角的验证点
    - 8.10　容易误解点
    - 8.11　本章结论
    - 8.12　参考文献
    - 8.13　信息可信度自评

- **第 9 章　分库分表替代**
    - 9.1　本章核心问题
    - 9.2　TiDB 的实现：逻辑表 + KV Region 自动分片
    - 9.3　OceanBase 的实现：分区表 + Tablet + Log Stream + Table Group
    - 9.4　核心差异对比
    - 9.5　正常路径图（逻辑表/分区表写入）
    - 9.6　故障/异常路径图
    - 9.7　性能、可靠性、运维影响
    - 9.8　反例与代价
    - 9.9　测试开发视角的验证点
    - 9.10　容易误解点
    - 9.11　本章结论
    - 9.12　参考文献
    - 9.13　信息可信度自评

- **第 10 章　分布式事务模型**
    - 10.1　本章核心问题
    - 10.2　TiDB 的实现
    - 10.3　OceanBase 的实现
    - 10.4　核心差异对比
    - 10.5　正常路径图
    - 10.6　故障 / 异常路径图
    - 10.7　性能、可靠性、运维影响
    - 10.8　反例与代价
    - 10.9　测试开发视角的验证点
    - 10.10　容易误解点
    - 10.11　本章结论
    - 10.12　参考文献
    - 10.13　信息可信度自评

- **第 11 章　MVCC 与读路径**
    - 11.1　本章核心问题
    - 11.2　TiDB 的实现
    - 11.3　OceanBase 的实现
    - 11.4　核心差异对比
    - 11.5　正常路径图(快照读)
    - 11.6　故障 / 异常路径图
    - 11.7　性能、可靠性、运维影响
    - 11.8　反例与代价
    - 11.9　测试开发视角的验证点
    - 11.10　容易误解点
    - 11.11　本章结论
    - 11.12　参考文献
    - 11.13　信息可信度自评

- **第 12 章　SQL 优化器与执行引擎**
    - 12.1　本章核心问题
    - 12.2　TiDB 的实现
    - 12.3　OceanBase 的实现
    - 12.4　核心差异对比
    - 12.5　正常路径图
    - 12.6　故障/异常路径图
    - 12.7　性能、可靠性、运维影响
    - 12.8　反例与代价
    - 12.9　测试开发视角的验证点
    - 12.10　容易误解点
    - 12.11　本章结论
    - 12.12　参考文献
    - 12.13　信息可信度自评

- **第 13 章　元数据与控制面**
    - 13.1　本章核心问题
    - 13.2　TiDB 的实现
    - 13.3　OceanBase 的实现
    - 13.4　核心差异对比
    - 13.5　正常路径图
    - 13.6　故障/异常路径图
    - 13.7　性能、可靠性、运维影响
    - 13.8　反例与代价
    - 13.9　测试开发视角的验证点
    - 13.10　容易误解点
    - 13.11　本章结论
    - 13.12　参考文献
    - 13.13　信息可信度自评

- **第 14 章　DDL 与 Online Schema Change**
    - 14.1　本章核心问题
    - 14.2　TiDB 的实现
    - 14.3　OceanBase 的实现
    - 14.4　核心差异对比
    - 14.5　正常路径图
    - 14.6　故障/异常路径图
    - 14.7　性能、可靠性、运维影响
    - 14.8　反例与代价
    - 14.9　测试开发视角的验证点
    - 14.10　容易误解点
    - 14.11　本章结论
    - 14.12　参考文献
    - 14.13　信息可信度自评

- **第 15 章　全局索引 / 本地索引**
    - 15.1　本章核心问题
    - 15.2　TiDB 的实现
    - 15.3　OceanBase 的实现
    - 15.4　核心差异对比
    - 15.5　正常路径图（按非分区键点查全局索引 + 回表）
    - 15.6　故障 / 异常路径图
    - 15.7　性能、可靠性、运维影响
    - 15.8　反例与代价
    - 15.9　测试开发视角的验证点
    - 15.10　容易误解点
    - 15.11　本章结论
    - 15.12　参考文献
    - 15.13　信息可信度自评

- **第 16 章　HTAP / AP 架构**
    - 16.1　本章核心问题
    - 16.2　TiDB 的实现
    - 16.3　OceanBase 的实现
    - 16.4　核心差异对比
    - 16.5　正常路径图
    - 16.6　故障 / 异常路径图
    - 16.7　性能、可靠性、运维影响
    - 16.8　反例与代价
    - 16.9　测试开发视角的验证点
    - 16.10　容易误解点
    - 16.11　本章结论
    - 16.12　参考文献
    - 16.13　信息可信度自评

- **第 17 章　多租户与资源隔离**
    - 17.1　本章核心问题
    - 17.2　TiDB 的实现
    - 17.3　OceanBase 的实现
    - 17.4　核心差异对比
    - 17.5　正常路径图
    - 17.6　故障/异常路径图
    - 17.7　性能、可靠性、运维影响
    - 17.8　反例与代价
    - 17.9　测试开发视角的验证点
    - 17.10　容易误解点
    - 17.11　本章结论
    - 17.12　参考文献
    - 17.13　信息可信度自评

- **第 18 章　容灾、高可用与多地多活**
    - 18.1　本章核心问题
    - 18.2　TiDB 的实现
    - 18.3　OceanBase 的实现
    - 18.4　核心差异对比
    - 18.5　正常路径图
    - 18.6　故障/异常路径图
    - 18.7　性能、可靠性、运维影响
    - 18.8　反例与代价
    - 18.9　测试开发视角的验证点
    - 18.10　容易误解点
    - 18.11　本章结论
    - 18.12　参考文献
    - 18.13　信息可信度自评

- **第 19 章　运维 / 观测 / 调优体系**
    - 19.1　本章核心问题
    - 19.2　TiDB 的实现
    - 19.3　OceanBase 的实现
    - 19.4　核心差异对比
    - 19.5　正常路径图(P99 定位的观测数据流)
    - 19.6　故障 / 异常路径图(P99 抖动定位链路)
    - 19.7　性能、可靠性、运维影响
    - 19.8　反例与代价
    - 19.9　测试开发视角的验证点
    - 19.10　容易误解点
    - 19.11　本章结论
    - 19.12　参考文献
    - 19.13　信息可信度自评

- **第 20 章　兼容性模型**
    - 20.1　本章核心问题
    - 20.2　TiDB 的实现
    - 20.3　OceanBase 的实现
    - 20.4　核心差异对比
    - 20.5　正常路径图
    - 20.6　故障/异常路径图
    - 20.7　性能、可靠性、运维影响
    - 20.8　反例与代价
    - 20.9　测试开发视角的验证点
    - 20.10　容易误解点
    - 20.11　本章结论
    - 20.12　参考文献
    - 20.13　信息可信度自评

- **第 21 章　安全 / 权限 / 审计 / 加密**
    - 21.1　本章核心问题
    - 21.2　TiDB 的实现
    - 21.3　OceanBase 的实现
    - 21.4　核心差异对比
    - 21.5　正常路径图
    - 21.6　故障 / 异常路径图
    - 21.7　性能、可靠性、运维影响
    - 21.8　反例与代价
    - 21.9　测试开发视角的验证点
    - 21.10　容易误解点
    - 21.11　本章结论
    - 21.12　参考文献
    - 21.13　信息可信度自评

- **第 22 章　云原生与 Kubernetes**
    - 22.1　本章核心问题
    - 22.2　TiDB 的实现
    - 22.3　OceanBase 的实现
    - 22.4　核心差异对比
    - 22.5　正常路径图
    - 22.6　故障/异常路径图
    - 22.7　性能、可靠性、运维影响
    - 22.8　反例与代价
    - 22.9　测试开发视角的验证点
    - 22.10　容易误解点
    - 22.11　本章结论
    - 22.12　参考文献
    - 22.13　信息可信度自评

- **第 23 章　存储池化 / Shared-storage**
    - 23.1　本章核心问题
    - 23.2　TiDB 的实现
    - 23.3　OceanBase 的实现
    - 23.4　核心差异对比
    - 23.5　正常路径图
    - 23.6　故障 / 异常路径图
    - 23.7　性能、可靠性、运维影响
    - 23.8　反例与代价
    - 23.9　测试开发视角的验证点
    - 23.10　容易误解点
    - 23.11　本章结论
    - 23.12　参考文献
    - 23.13　信息可信度自评

- **第 24 章　迁移传统分库分表系统的方法论**
    - 24.1　本章核心问题
    - 24.2　TiDB 的实现：迁移路径与主键防热点
    - 24.3　OceanBase 的实现：分区表、Table Group 与全局索引
    - 24.4　核心差异对比
    - 24.5　正常路径图：分阶段迁移与切流
    - 24.6　故障/异常路径图
    - 24.7　性能、可靠性、运维影响
    - 24.8　反例与代价
    - 24.9　测试开发视角的验证点
    - 24.10　容易误解点
    - 24.11　本章结论
    - 24.12　参考文献
    - 24.13　信息可信度自评

- **第 25 章　Benchmark 与测试设计**
    - 25.1　本章核心问题
    - 25.2　TiDB 的实现
    - 25.3　OceanBase 的实现
    - 25.4　核心差异对比
    - 25.5　正常路径图
    - 25.6　故障/异常路径图
    - 25.7　性能、可靠性、运维影响
    - 25.8　反例与代价
    - 25.9　测试开发视角的验证点
    - 25.10　容易误解点
    - 25.11　本章结论
    - 25.12　参考文献
    - 25.13　信息可信度自评

#### 附录

- 附录 A　术语表
- 附录 B　版本基准表
- 附录 C　参考文献总表
- 附录 D　源码索引
- 附录 E　图目录
- 附录 F　冲突清单与裁决
- 附录 G　各章核心结论汇编


---


# 第 1 章 分片

> 版本基准：TiDB 8.5.x LTS(对照 7.5.x)；TiKV/PD release-8.5；OceanBase v4.2.5_CE(TP)、4.3.5(AP)、4.4.x(融合态，4.4.1_CE 2025-10-24)。源码引用均为 commit-pinned，统一取 release-8.5 / v4.2.5_CE 基线。

## 1.1 本章核心问题

分片(sharding)是分布式数据库一切横向扩展能力的起点：把一张逻辑上无限大的表，切成若干可以被独立放置、复制、调度、恢复的物理单元，再把这些单元铺到一组节点上。本章要回答的核心问题不是"能不能分片"，而是分片的单元到底是什么、由谁决定边界、它如何同时承担存储、共识、事务与调度这几重身份。

在动手之前必须先把几个常被混为一谈的概念拆开，因为 TiDB 与 OceanBase 恰好在这几个维度上做了不同选择。

![[f1_1.svg]]

**图 1-1　一个分片的多重身份归属映射**

- **逻辑分片**：用户在 SQL 层面感知的切分。TiDB 的逻辑对象是表、索引、SQL 分区；OceanBase 的逻辑对象是 Table 与 `PARTITION BY HASH/RANGE/LIST` 声明的 Partition。
- **物理分片**：存储引擎实际承载数据的最小连续单元。TiDB 是 Region 覆盖的 KV range；OceanBase 是 Tablet 这一存储对象。
- **共识分片**：复制与多数派提交的最小单元。TiDB 是一个 Region 副本组成的 Raft Group；OceanBase 是一个 Log Stream(LS)副本组成的 Multi-Paxos / PALF 复制组。
- **事务分片**：跨分片事务的边界，决定哪些写入走单分片快路径、哪些必须走分布式提交。TiDB 的事务按 key 范围访问多个 Region；OceanBase 4.x 把 LS 作为事务参与者，单 LS 内可走更轻的提交路径，跨 LS 需要分布式提交协议(primary lock、2PC、Percolator 等细节详见第 10 章)。
- **调度分片**：控制面(PD / RootService)做负载均衡、迁移、副本变更时操作的最小对象。

**TiDB 把这几类分片几乎绑成同一个对象：Region。** 一个 Region 既是一段连续 Key Range(物理分片)，又是一个 Raft Group(共识分片)，又是 PD 调度与迁移的单元(调度分片)，也是单分片事务的天然边界。**OceanBase 4.x 则刻意把它们解耦：** 逻辑分片是 Partition，物理分片是 Tablet，共识与调度分片是 LS，三者是"多对一、可重新绑定"的关系。这条主线贯穿全章。

由此本章的核心判断是：TiDB 更像"KV Range 自动分片"，其 Region 与 SQL 表并不一一对应，分片边界是运行时涌现的；OceanBase 更像"关系对象分区 + Tablet + Log Stream 复制"，其 Partition 与 Tablet 关系更直接，但 Tablet 到 LS 的归属又把数据分片与日志/事务分片解耦，分片是设计期建模的产物。这两个方向都不是简单的"谁更细"或"谁更先进"，而是分别在自动 range 管理、SQL 对象管理、日志复制开销、事务边界与运维调度复杂度之间做了不同取舍。本章与路由(第 2 章)、共识(第 3 章)、存储引擎(第 4 章)、事务(第 10 章)紧密相关，涉及时只简短提及并标注"详见第 X 章"。

## 1.2 TiDB 的实现

### 组件与数据路径

TiDB 的分片建立在一条"SQL → KV 编码 → Key Range → Region → Raft Group → TiKV Store"的链路上。

第一步是**编码**。TiDB Server 把一行 SQL 数据编码成一个 KV 对，key 形如 `t{TableID}_r{RowID}`，其中 `tablePrefix = 't'`、`recordPrefixSep = 'r'`；索引则编码为 `t{TableID}_i{IndexID}_{indexedColumnsValue}`，`indexPrefixSep = 'i'`。当 row handle 属于某个 SQL 分区时，编码会用 PartitionID 生成 record prefix。这一编码保证同表、同索引的数据在全局有序 key 空间中前缀聚集、按 RowID/索引列有序排列，因此范围扫描天然高效。**关键推论是：Region 不是"表的分片"本身，而是"编码后 keyspace 的连续范围"；一个 Region 可能覆盖某个表的一段 record key，也可能覆盖索引 key，甚至在边界尚未拆开时跨越相邻对象前缀。**

第二步是**按 Range 切分**。TiKV 不按 hash 而按 key range 把这个大空间切成连续区间 `[start_key, end_key)`(语义可保守写成 `StartKey <= key < EndKey`)，每个区间就是一个 **Region**。官方 deep-dive 的 data-sharding 页明确解释了为何选 Range 而非 Hash：Range "is friendly to range scans"，而 hash "fails to do efficient range queries"；并且 split/merge 只改元数据不搬数据(原文："for the split or merge of Regions, TiKV only needs to change meta-information about the range of the Region, which avoids moving actual data in a large extent")。

第三步，每个 Region **就是一个 Raft Group**，默认 3 副本，分布在不同 TiKV Store 上，通过 Raft 达成多数派提交(共识细节详见第 3 章)。一个 TiKV 节点上同时驻留数以千计乃至上万个 Region 的副本，这就是 **Multi-Raft**：Raftstore 用一个 batch 化的 event loop 驱动所有 Raft Group(原文："TiKV uses an event loop to drive all the processes in a batch manner")。

正常写入路径因此可以这样刻画：TiDB Server 把 SQL 行与索引变更编码为 KV mutation；SQL 层和执行层把表扫描、索引扫描转换为 key ranges，再通过 TiKV client / RegionCache 及 PD 元数据定位 Region location(本章只写到"按 KeyRange 找 Region location"，client-go 内部缓存刷新与 backoff 详见第 2 章)；mutation 按 key 分布落到一个或多个 Region；每个 Region 的 Leader 接收写入，在 Raft Group 内复制并提交，应用到本地存储后返回。**对于同一 SQL 表，row key 和 index key 可能落在不同 Region，因此"表只有一个 Leader"是错误说法。**

### Region 与 SQL 表：非一一对应

这是本章高风险点之一，必须区分**默认行为**与**物理可能性**两个层面，且涉及两个不同的配置项，极易混淆。

- **TiDB-server 侧的 `split-table`(默认 `true`)**：TiDB 配置文件中的 `split-table` "Determines whether to create a separate Region for each table"，默认值为 `true`。**默认情况下 TiDB 恰恰为每张新表单独切出一个 Region**(初始一表一 Region)，因此"默认就跨表"的说法是错误的。官方还建议：当需要创建大量表(如超过 10 万张)时才把它设为 `false`。
- **TiKV coprocessor 侧的 `split-region-on-table`(默认 `false`)**：这是另一个参数，默认 `false` 表示 TiKV **不强制**在表边界切 Region，因此 Region 在物理上**可以**横跨多张小表(TiKV Region 本质是 key range)；开启后才以 table 前缀对齐 split key。

实践上的表现是：**默认配置下，新建一张表时 TiDB 为它切一个 Region；随着数据增长，这个 Region 不断分裂成多个**。`SHOW TABLE REGIONS` 可以直接观察到一张表对应多个 Region、每个 Region 的 `START_KEY/END_KEY` 含表前缀(如 `t_75_`)，record data 与 index data 可以落到各自独立的 Region，这与源码中的 table/index key prefix 相互印证。PD 还有 `enable-cross-table-merge` 配置控制是否允许把不同表的相邻 Region 合并，其默认值为 `false`(默认不跨表合并)。**结论：Region 与 SQL 表并非严格一一对应——一张大表会被切成多个 Region，且 Region 在物理上可以跨表；但默认配置(`split-table=true`)下，新表起步是一表一 Region。** 因此不要用"表分片数"等同于 Region 数来估算事务参与者(见 §1.8)。

### 控制路径：split / merge / 调度

**表 1-1　Region split/merge 与 Tablet auto-split 阈值参数表**

| 触发项 | 阈值／默认值 | 动作 | 系统 |
|---|---|---|---|
| `region-split-size`（`SPLIT_SIZE`） | 256MiB（旧版 96MB） | split | TiKV |
| `region_max_size()` | ≈384MiB（split_size 的 1.5 倍） | split | TiKV |
| `region_split_keys()` | ≈2.56M keys（split_size_mb × 10000） | split | TiKV |
| `BATCH_SPLIT_LIMIT`（单次 batch split 上限） | 10 | split | TiKV |
| 负载分裂 QPS（`DEFAULT_QPS_THRESHOLD` ／ big region） | 3000 ／ 7000，连续 10 秒 | load split | TiKV |
| 负载分裂字节（`DEFAULT_BYTE_THRESHOLD` ／ big region） | 30MiB ／ 100MiB，连续 10 秒 | load split | TiKV |
| 负载分裂 CPU（`REGION_CPU_OVERLOAD_THRESHOLD_RATIO` ／ big region） | 0.25 ／ 0.75，连续 10 秒 | load split | TiKV |
| Raftstore V2（`RAFTSTORE_V2_SPLIT_SIZE`） | 10GiB | split | TiKV |
| `max-merge-region-size`（`defaultMaxMergeRegionSize`） | 54MiB | merge | PD |
| `max-merge-region-keys` | 配置默认 0 → 540000 keys（54 × 10000） | merge | PD |
| `auto_split_tablet_size`（`enable_auto_split` 默认关闭） | 4.4.1_CE 默认值由 128MB 调为 2GB；4.3.5.x 可配为 2G | split | OceanBase |

**按大小拆分。** `components/raftstore/src/coprocessor/config.rs` 中真实常量 `pub const SPLIT_SIZE: ReadableSize = ReadableSize::mb(256);`，即 release-8.5 默认 `region-split-size = 256MiB`。源码注释明确写出版本演进："In version < 8.3.0, the default split size is 96MB. In version >= 8.3.0, the default split size is increased to 256MB"。配套推导阈值(真实代码)：`region_max_size()` ≈ split_size 的 1.5 倍(代码 `region_split_size()/2*3` → 384MiB)；`region_split_keys()` 默认 = `split_size_mb × 10000`(注释假设 KV 平均 100B → 256MB×10000 ≈ 2.56M keys)。一次 batch split 最多产出的 split key 数由 `pub const BATCH_SPLIT_LIMIT: u64 = 10;` 限制。

> **版本口径备注**：release-8.5 源码注释写"≥8.3.0 即 256MB"，而 docs.pingcap.com(Tune Region Performance / Raft Region Size 博客)写"Starting from v8.4.0, default resized from 96 MiB to 256 MiB"，TiDB Cloud 性能文档又把 96 MiB→256 MiB 置于 v8.5.0 上下文。三者对**默认值切换的版本归属**口径不一(8.3 / 8.4 / 8.5)。本章锁定 v8.5.0 默认值为 **256MiB**(无争议)，不把"变更起点"当作确定事实。

在大小阈值之外，还有一条按负载触发的拆分路径。**按负载拆分(load-base-split)。** TiKV 还能对热点 Region 做负载拆分。`components/raftstore/src/store/worker/split_config.rs` 中真实常量：`DEFAULT_QPS_THRESHOLD = 3000`、`DEFAULT_BIG_REGION_QPS_THRESHOLD = 7000`、`DEFAULT_BYTE_THRESHOLD = 30MiB`、`DEFAULT_BIG_REGION_BYTE_THRESHOLD = 100MiB`、`REGION_CPU_OVERLOAD_THRESHOLD_RATIO = 0.25` / `BIG_REGION_... = 0.75`。官方文档对这组阈值的描述与源码完全吻合：当 Region 连续 10 秒 QPS/字节/CPU 超阈值即判为热点，在"能平衡两侧负载、减少跨 Region 访问"的位置拆分，而非简单二分。这是一次源码常量 ↔ 官方文档的成功交叉验证。

**拆分检查机制(split-checker)。** 拆分由 coprocessor 层的 split-checker 驱动，模块 `components/raftstore/src/coprocessor/split_check/` 含 `size.rs / keys.rs / half.rs / table.rs`，真实类型为 `SizeCheckObserver`(`get_region_approximate_size`)、`KeysCheckObserver`、`HalfCheckObserver`、`TableCheckObserver`；`Host<'a, E>` 聚合多个 checker，任一为 `Approximate` 即整体走近似策略，否则全量 Scan。近似大小靠 RocksDB SST 的 TableProperties 估算：TiKV 在 compaction 时把各 sub-range 的数据大小记入 SST table property，split-check 直接汇总区间内的 table properties 得到 Region 近似大小，从而避免全量扫描。

**调度侧执行。** TiKV 通过 StoreHeartbeat 与 RegionHeartbeat 向 PD 报告 store 容量、Region range、副本分布、peer 状态、数据量与读写流量；PD 据此生成 operator 执行 balance region、hot region、scatter、merge、节点下线与故障恢复等调度。TiKV 上报后，PD 通过 `pkg/schedule/splitter/region_splitter.go` 的 `RegionSplitter`(方法 `SplitRegions(ctx, splitKeys, retryLimit)`)把用户或检查器给出的 split key 按当前 Region 归组、下发完成调度侧拆分；`pkg/schedule/checker/split_checker.go` 还会在 placement rule / label boundary 被跨越时拆 Region——**这说明 TiDB 的 Region split 不只服务容量阈值，也服务放置规则与控制面约束**。小 Region 的自动合并由 `pkg/schedule/checker/merge_checker.go` 的 `MergeChecker` 负责，`Check()` 调 `region.NeedMerge(GetMaxMergeRegionSize(), GetMaxMergeRegionKeys())`，并通过 `RecordRegionSplit` 把刚拆分的 Region 放进 `splitCache`(TTL=`split-merge-interval`)以抑制"拆完立刻被合"的抖动。`max-merge-region-size` / `max-merge-region-keys` 的默认数值由 config 层提供。`pkg/schedule/config/config.go` 中 `defaultMaxMergeRegionSize = 54`，即默认 `max-merge-region-size = 54MiB`；源码注释说明该值在 tikv issue #17309 后从 20 提升到 54，以匹配 TiKV 256MiB 的 Region 大小。`max-merge-region-keys` 的默认配置值为 0，此时 `GetMaxMergeRegionKeys()` 返回 `MaxMergeRegionSize × RegionSizeToKeysRatio`(`RegionSizeToKeysRatio = 10000`)，即 54 × 10000 = 540000 keys。

**Raftstore V2(Partitioned Raft KV)。** 自 TiDB v6.6 引入的新引擎让每个 Region 独占一个 RocksDB 实例；此模式下默认拆分阈值被覆盖为 `RAFTSTORE_V2_SPLIT_SIZE = 10GiB`(真实常量)，由 `optimize_for(raftstore_v2)` 设置。

### Keyspace(多租户隔离)

PD 的 `pkg/keyspace/keyspace.go` 提供 `Manager`，用 region label(前缀 `keyspaces/`)把每个 keyspace 的 key range 与 Region 关联，并在 `Bootstrap()` 时为 default keyspace 预拆 Region(`splitKeyspaceRegion`)。需要强调：keyspace 是控制面 / TSO 分组能力，而非 Region 的物理数据分片层级，它是 TiDB 把"逻辑租户"映射回"物理 Region range"的机制。TiDB Cloud Starter/Essential 的内部多租户隔离细节未公开，本章不做实现层断言。

源码定位均为 commit-pinned：TiDB 编码位于 `pingcap/tidb @ release-8.5` 的 `pkg/tablecodec/tablecodec.go`；TiKV split 常量位于 `tikv/tikv @ release-8.5` 的 `components/raftstore/src/coprocessor/config.rs`、`split_check/`、`store/worker/split_config.rs`；PD 调度位于 `tikv/pd @ release-8.5` 的 `pkg/schedule/`、`pkg/keyspace/`(详见 §1.12)。 〔文献[1-6,9-11,20-22]〕

## 1.3 OceanBase 的实现

### 组件与归属关系

与 TiDB 把多重身份压在 Region 上不同，OceanBase 4.x 的分片路径必须分三层写："Table → Partition → Tablet → 属于某个 Log Stream → 由 PALF / Multi-Paxos 复制 → OBServer / Zone"，但关键在于**这几层不是一一对应，而是可重绑定的归属关系**。

- **Partition(逻辑分片)**：用户用 `PARTITION BY HASH/RANGE/LIST` 声明的逻辑切片，是用户创建和管理表数据的逻辑对象，支持二级分区。对二级分区表，每个二级子分区才是物理分区，一级分区只是逻辑概念。
- **Tablet(物理分片)**：OceanBase 4.0 引入 Tablet 表示实际数据存储对象。官方架构文档写明，单分区表创建一个 Tablet，多分区表每个 Partition 创建一个 Tablet，Tablet 是存储有序数据记录的最小存储对象，也是数据均衡的最小单位之一，支持在服务器之间转移。
- **Log Stream / LS(共识 + 调度分片)**：复制与多数派提交的单元。官方架构文档原文："One log stream corresponds to multiple Tablets on the node where it is located"，即 **LS 与 Tablet 是一对多**；"Multi-Paxos uses Log Stream to implement data replication"；"Tablets can be migrated between Log Streams to achieve load balancing"。LS 是 OceanBase 自动创建和管理的实体，包含若干 Tablet 与有序 redo logs，LS 副本通过 Multi-Paxos / PALF 同步日志，LS 同时是事务提交参与者。

合在一起，保守结论是：**"Partition、Tablet、LS 是一条简单一一对应链"也是错误说法。** Partition 与 Tablet 在用户表/分区的常见路径上接近一一对应，但索引、内部 Tablet 和系统对象会让对象计数更复杂；Tablet 是实际存储对象，携带或映射到某个特定 LS；LS 是若干 Tablet 的日志复制与事务边界容器。因此 4.x 中不应把 Partition 直接写成 LS / Paxos Group 边界，否则会把 3.x 之前的模型和 4.x LS 模型混在一起(见 §1.10)。

源码层面这一归属结构非常清晰。`src/storage/ls/ob_ls.h` 中 `class ObLS` 同时持有：`ObLSTabletService ls_tablet_svr_`(管理一组 Tablet)、`logservice::ObLogHandler log_handler_`(LS 的日志/共识入口)、以及 `ObLSTxService` 事务服务、`ObCheckpointExecutor` 等。**因此一个 LS = 一个日志/共识句柄 + 一组 Tablet + 事务/锁/checkpoint 服务；Tablet 本身不自带日志流，它归属于所在 LS，与 LS 一起被调度和复制。** `src/storage/tablet/ob_tablet.h` 中 `ObTablet` 暴露 `get_ls_id()`、`get_tablet_id()`、`get_data_tablet_id()`，创建接口接收 `ls_id`、`tablet_id`、`data_tablet_id`、`create_scn` 等参数；`src/share/tablet/ob_tablet_to_ls_operator.h` 维护 Tablet 到 LS 的映射，可按 tablet IDs 批量获取 LS，也可在 Tablet transfer 时更新 `__all_tablet_to_ls` 的 `ls_id` 与 transfer sequence——从代码层面证明 Tablet 的物理归属单元就是 LS。

每个 LS 还自带固定数量的 inner tablet：`static constexpr int64_t TOTAL_INNER_TABLET_NUM = 3;`(系统用途，如 tx data)，用户 tablet 在此之外。LS 的编号也有约定：`src/share/ob_ls_id.h` 中 `SYS_LS_ID = 1`(每个 Tenant 一个 SYS LS)、`MIN_USER_LS_ID = 1000`(用户数据 LS id ≥ 1000)；SYS LS 同时复用为 GTS、Lock Service、Weak Read、Global AutoInc、Major Freeze 等内部服务的承载 LS。

### 4.x 引入 LS 后相对早期 partition-leader 模型的变化

这是本章必须谨慎处理的高风险事实。在 OceanBase 3.x 及更早，**每个 Partition 自己就是一个 Paxos 复制组，有自己的 partition leader 和独立日志流**；一个节点上有多少分区，就有多少个独立的 Paxos 组在各自跑选举、心跳、日志同步，元数据与心跳开销随分区数线性膨胀。PALF 论文对此有直接描述：OceanBase 1.0–3.0 中，事务处理、日志和数据存储的基本单位是 table partition；随着每台服务器上分区数量增多，过多 Paxos Group 带来复制组资源开销，并使跨大量分区事务的参与者数量过高。

OceanBase 4.0(Paetica 架构，VLDB 2023)把这一模型改为 **Log Stream 模型**：把同一资源单元(Unit)内、本应共置的多个分区 leader 收拢到同一台机器，让这些分区共享同一条 Log Stream 写日志——官方称之为"single-machine log stream design"。**变化的本质是：共识单元的粒度从"每分区一个 Paxos 组"上移到"每 LS 一个 Multi-Paxos 组"，分区/Tablet 退化为 LS 内部被管理的存储对象。** 这带来三个直接后果：(1) 一个节点上的 Paxos 组数量从"分区数量级"骤降到"LS 数量级"，选举/心跳/日志开销大幅下降；(2) Tablet 可以在 LS 之间迁移以做负载均衡，而不必每次都重建一个 Paxos 组；(3) 单机部署时，分布式组件开销可以被压到接近单机数据库，这正是 Paetica"单机-分布式一体化"的目标。

LS 模型在 4.x 内部仍在持续演进。v4.2.5_CE 增加了 transfer-based partition(把指定分区迁移到指定 LS)和 LS 副本级别的 O&M(增删/改类型/迁移 LS 副本、调整 Paxos 副本数、取消容灾任务)。源码可逐项核对：`src/objit/include/objit/common/ob_item_type.h` 定义 `T_TRANSFER_PARTITION = 4570`(注释 "transfer tablet manually")与 `T_TRANSFER_PARTITION_TO_LS = 4571`，对应 `src/rootserver/ob_transfer_partition_command.*` 与 `ObTransferPartitionHelper`；LS 副本级 O&M 由 `src/rootserver/ob_disaster_recovery_worker.h` 的 `ObAdminAlterLSReplicaArg` 驱动，逐一暴露 `do_add_ls_replica_task` / `do_remove_ls_replica_task` / `do_migrate_ls_replica_task` / `do_modify_ls_replica_type_task` / `do_modify_ls_paxos_replica_num_task` / `do_cancel_ls_replica_task`，与官方 4.2.5 LTS 发布说明逐项吻合(发布说明称"redesigned to implement operations at the log stream replica level … adding, deleting, changing the type of, and migrating log stream replicas … modifying the number of Paxos replicas and canceling disaster recovery tasks")。日志栈本身也在变：`src/logservice/ob_log_handler.h` 在 v4.2.5_CE 直接依赖具体的 `palf::PalfEnv`，而 4.4.x 在 ObLogHandler 与 PALF 之间插入了 **ipalf 抽象层**(`ipalf::IPalfEnv` / `IPalfHandle`，新增目录 `src/logservice/ipalf/`)，为多种日志后端预留抽象。

### LS 的复制：PALF

LS 的复制日志即 **PALF(Paxos-backed Append-only Log File system)**。`ObLogHandler` 是 LS 层对 PALF 的封装入口，其 `init(... palf::PalfEnv *palf_env ...)` 与 append/seek 接口大量使用 `palf::LSN`、`palf::PalfGroupBufferIterator` 等类型；PALF 实现位于 `src/logservice/palf/`(含 `palf_handle.h`、`log_engine.*`、`election/`)。PALF 在 VLDB 2024 有专门论文("Replicated Write-Ahead Logging for Distributed Databases")，其核心思想是把日志系统与整个数据库协同设计，把日志能力抽象为 PALF 原语支撑事务、备份、物理 standby 等。在数据路径上，DML 在目标 Tablet 对应 LS 的 Leader 上执行，生成 redo logs，由 PALF 把复制式 WAL(Paxos-backed append-only log)同步到多数派副本并推进事务状态，Follower 回放日志保持状态一致；若事务只涉及一个 LS，可走单 LS 内提交逻辑，若跨多个 LS，事务层需协调多个 LS 参与者(GTS/SCN、MVCC 与事务恢复详见第 10、11、13 章)。PALF 相对教科书 Multi-Paxos 的具体工程增强属于共识范畴，详见第 3 章。

### 均衡与 Tablet 是否能动态分裂

OceanBase 的均衡路径也与 TiDB 不同。TiDB 主要按 Region range 和 peer/Leader 做 split、merge、transfer；OceanBase 4.x 的 balancing layer 在创建新表或增加 Partition 时，会按均衡原则选择合适 LS 创建 Tablet；当租户资源变化、扩容或长时间运行造成 Tablet 分布不均时，会进行 LS split/merge、移动 LS replica，或把一部分 Tablet 迁入新 LS/临时 LS 再迁移合并到目标服务器。**这里的"split/merge"对象不是 TiDB Region，而是 LS / Tablet 组合上的均衡动作。** 4.4.x 源码层面这套均衡调度由 RootServer 内的 `ObTenantBalanceService`(`src/rootserver/ob_tenant_balance_service.*`)统筹，具体执行分散在 `ObPartitionBalance`(分区/Tablet 在 LS 间的均衡)、`ObLSBalanceTaskHelper`(LS 级 split/merge/副本调整)、`ObBalanceTaskExecuteService`(均衡任务执行)与 `ObTransferPartitionHelper`(手动 transfer)等类；运行时按大小自动分裂 Tablet 则走另一条路径，由 `ObServerAutoSplitScheduler`(`src/share/scheduler/ob_partition_auto_split_helper.h`，含 `ObAutoSplitTask` / `ObAutoSplitTaskCache`)与 `ObPartitionSplitTask` 驱动。

早期 OceanBase 没有"Region 那样的自动按大小分裂"——分片粒度主要由用户预先声明的分区决定，这是它与 TiDB 最显著的设计差异之一。但需要精确区分"**功能引入**"与"**默认值调整**"两件事。

- **自动 Tablet/分区分裂功能本身在 4.3.x 已存在**：`enable_auto_split`(默认关闭)与 `auto_split_tablet_size` 参数在 4.3.5.x 即可用，社区资料显示 4.3.5.x 中 `auto_split_tablet_size` 已可配为 2G、`enable_auto_split` 已可启用，且已有 4.3 生产环境使用自动分裂的案例。
- **4.4.1_CE(2025-10-24)做的是默认值调整**：官方 v4.4.1_CE 发布说明的参数变更表逐字记录 `auto_split_tablet_size` 的变更类型为"变更默认值 / Modified"，内容是将默认值从 128MB 调整为 2GB("Changed the default value from 128 MB to 2 GB")，变更类型并非"新增 / New"。同版本还有 `global_index_auto_split_policy`、手动创建 LS、基于权重的分区均衡等条目。

因此**自动分裂功能在锁定版本范围内的 4.3.5.x 已具备，4.4.1_CE 仅调整了 `auto_split_tablet_size` 的默认值(128MB→2GB)**，是其向"运行时自动分裂"靠拢的信号，但 4.4.x 为融合态，部分能力的 CE 与企业版/Cloud 差异需以发布说明为准。此结论全章只说一次。

源码定位均为 commit-pinned：`oceanbase/oceanbase @ v4.2.5_CE` 的 `src/storage/ls/ob_ls.h`、`src/share/ob_ls_id.h`、`src/storage/tablet/ob_tablet.h`、`src/storage/tablet/ob_tablet_binding_helper.h`、`src/share/tablet/ob_tablet_to_ls_operator.h`、`src/share/ls/ob_ls_info.h`、`src/logservice/ob_log_handler.h`、`src/logservice/palf/`；ipalf 抽象层另据 4.4.x 对照(详见 §1.12)。 〔文献[12-17,23-24]〕

## 1.4 核心差异对比

两边的分歧不在"谁分得更细"，而在分片的几重身份是绑定还是解耦。表 1-2 按维度分别陈述差异及其影响，表后再给"为什么"。

**表 1-2　分片:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB | OceanBase 4.x | 影响 |
|---|---|---|---|
| 逻辑分片来源 | 隐式：表/索引/分区编码进有序 KV 空间，由 key range 自动切分，用户通常无需声明 | 显式：用户用 `PARTITION BY` 声明 Partition(可二级) | TiDB 对用户透明，用户对象与 Region 边界弱绑定；OceanBase 需建模分区键，分区语义更显式 |
| 物理分片单元 | Region(连续 Key Range `[start,end)`) | Tablet(每物理分区一个，可在服务器间转移) | 二者都是有序存储单元 |
| 共识单元 | Region = 一个 Raft Group(3 副本) | Log Stream = 一个 Multi-Paxos 组(PALF) | TiDB 每分片一个共识组；OceanBase 多 Tablet 共享一个 LS |
| 分片与共识的关系 | 绑定(物理=共识=调度同一对象) | 解耦(Tablet 物理，LS 共识/调度，可重绑定) | OceanBase 共识组数量 ≈ LS 数 ≪ 分区数 |
| 事务分片 | 事务访问多个 Region 时由事务层协调，本章不展开 | LS 是事务参与者，单 LS 与跨 LS 路径不同 | OceanBase 4.x 用 LS 降低"大量 Partition 即大量事务参与者"的风险 |
| 与 SQL 表的对应 | 默认 `split-table=true`，每张新表起步一表一 Region；大表分裂为多 Region；`split-region-on-table=false` 使 Region 物理上可跨表 | Partition 与表是声明式从属；Tablet 与物理分区常 1:1(索引/内部对象除外) | TiDB Region 可跨表(物理可能，非默认常态)；OceanBase 分区边界用户可控 |
| 默认/自动分裂 | 按 size(256MiB)、keys、load(QPS/字节/CPU)、split-key 自动分裂 | 自动分裂功能 4.3.x 已具备；4.4.1_CE 把 `auto_split_tablet_size` 默认值由 128MB 调为 2GB(见 §1.3) | TiDB 长期内建自动分裂；OceanBase 4.3.x 起补齐、4.4.1 调默认阈值 |
| 热点处理 | load-base-split + 预切分(`AUTO_RANDOM`/`SHARD_ROW_ID_BITS`/`PRE_SPLIT_REGIONS`) | Tablet 在 LS 间 transfer + 分区设计 + 4.4 自动分裂 | TiDB 偏运行时自动；OceanBase 偏设计期 + 迁移 |
| 迁移单元 | Region(snapshot 传输) | Tablet(在 LS 间 transfer)/ LS 副本 | OceanBase 迁移 Tablet 不必重建共识组 |
| 控制面 | PD(分裂/合并/调度/TSO/keyspace) | RootService(sys tenant)+ 各 LS 自治 | 见 §1.7 中心化维度分析 |

**为什么 TiDB 更像"KV Range 自动分片"**：它把表/索引编码进一个全局有序 KV 空间，再纯按 size/keys/load/split-key 自动切 Region，用户基本无感；分片边界是运行时涌现的，不是设计期声明的。**为什么 OceanBase 更像"关系对象分区 + Tablet + 日志流复制"**：它从关系模型出发，先有用户声明的 Partition，落到 Tablet 存储，再把一批 Tablet 挂到 LS 上由 Multi-Paxos / PALF 复制；分片是设计期建模的产物，LS 才是被复制和调度的"原子"。

## 1.5 正常路径图

图 1-2 展示两边的正常写入分片路径(TiDB 左、OceanBase 右)，刻意把 TiDB 的 SQL Table/Index 放在编码前、把 Region 放在 KV keyspace 之后，以避免"Region 与 SQL 表一一对应"的误解；OceanBase 部分则把 Partition、Tablet、LS 拆成三层，避免把 4.x Partition 逻辑概念直接写成复制组。

![[f1_2.svg]]

**图 1-2　分片正常读写/调度路径**

要点对照：TiDB 的写入在"Region=Raft Group"这一层就完成了物理、共识、调度三重身份的统一；OceanBase 的写入则要先确定 Tablet，再按 tablet-to-LS 映射找到它所属的 LS，由 LS 的 PALF 日志做复制，Tablet 自身不参与共识。官方文档中 `SHOW TABLE REGIONS` 示例(一个表 record/index data 拆成多个 Region)与 OceanBase 架构文档"LS 含若干 Tablet 与 redo logs"分别印证了这两条路径的非一一对应特征。

## 1.6 故障/异常路径图

图 1-3 为故障与异常路径对比，覆盖热点拆分、Leader 切换、迁移抖动、控制面不可用。

![[f1_3.svg]]

**图 1-3　分片故障/异常路径**

TiDB 异常路径中，Region split 会改变 Region 边界并生成新的 Raft Group；Leader 故障是 Raft Group 内的选主问题，PD 更多参与元数据、TSO 与后续调度，**不应被简单称为 Region Raft 数据面的"读写单点"**(见 §1.7)。OceanBase 异常路径中，LS Leader 故障的恢复粒度是 LS，会一并影响其下所有 Tablet；Tablet transfer 改变数据对象归属，LS split/merge 或 replica 移动则是均衡层动作。图中只刻画路径与粒度，具体错误码、内部日志字段与 Prometheus metric 名称本章未逐项核验，需进一步查证。 〔文献[19]〕

## 1.7 性能、可靠性、运维影响

**延迟与吞吐。** TiDB 每个 Region 一个 Raft Group，海量小 Region 会让 PD 承载庞大的元数据集合，并放大 Raft 心跳与选举开销；Region 过大又让 snapshot 迁移超时——256MiB 默认值正是在二者之间取的折中。对数据自然按主键或索引范围访问的业务，Region 的连续范围能让范围扫描、Coprocessor 下推和 Region 级并发自然结合；但代价是热点识别与拆分依赖 key 分布，单调递增 key、时间戳索引、少数高频账户或状态行都可能让写入集中到少数 Region。OceanBase 用 LS 收拢共识组，单 LS 内多个 Tablet 共享一条日志复制边界，单节点 Paxos 组数 ≈ LS 数而非分区数，显著降低复制组元数据与日志复制的固定开销；但代价是 LS 粒度的切主会一次性影响其下所有 Tablet，且 Tablet-to-LS 映射与 LS 均衡逻辑多了一层，过多跨 LS 事务会增加协调成本。

**扩展性与迁移。** TiDB 扩容 = PD 把 Region(snapshot)往新节点搬，粒度细、自动化程度高；Region 越大单次迁移越慢。OceanBase 扩容 = balancing layer 做 LS split/merge、移动 LS replica 或在 LS 间 transfer Tablet，Tablet 迁移不必重建共识组，但历史上缺少自动按大小分裂(4.3.x 起补齐，见 §1.3)，热点分区若设计期没拆好，运行期调整成本更高。两边扩容验证都不能只看节点数，要看分片/Leader 是否真正重平衡。

**逻辑中心化与"单点"刻画(必须分维度)。** PD 与 RootService 都不是"单副本失效即全局不可用"的朴素单点，但二者性质并不对称：PD 是 etcd/Raft 多副本的对称高可用；而 OceanBase 官方对 RS 明确保留 **SPOF(single point of failure)** 措辞，因此 RS 不可简单宣称"已无单点风险"。把任一控制面简化成"单点"，会混淆逻辑中心、性能瓶颈、高可用单点、元数据/调度依赖、TSO/GTS 依赖与故障恢复依赖这几个不同维度，下表按维度分别陈述。

**表 1-3　性能、可靠性、运维影响**

| 维度 | TiDB PD | OceanBase RootService |
|---|---|---|
| 逻辑中心化 | 是：全局元数据/调度/TSO 集中 | 是：sys tenant 的 RS 统管元数据/资源/major compaction |
| 高可用单点性质 | 对称多副本：PD 集群基于 etcd/Raft(≥3 奇数节点)，Leader 可重选，官方称解决了单点问题 | 非对称：全局仅一个 RS 在跑(`__all_core_table` leader 所在 OBServer)，靠 Paxos 多副本自动重选；官方明确 "RS still poses the risk of a single point of failure (SPOF)" |
| 性能瓶颈风险 | 有：TSO 由 PD Leader 单点分配(官方称可达约 2.6 亿 ts/s，仍有上限；follower proxy/batch 缓解) | 有：RS 处理元数据/调度可能成瓶颈；GTS 由 SYS LS 承载(详见第 10、13 章) |
| 元数据/调度依赖 | 强：split/merge/balance/路由刷新依赖 PD | 强：partition transfer/副本变更/合并依赖 RS |
| 故障恢复依赖 | PD Leader 重选期间 TSO/调度暂停，缓存路由可短时维持读；PD 不在 Region Raft 数据面同步路径上 | 官方坦言 RS 是 "MetaServer 的 Achilles heel"，RS bug 会影响建连与 major compaction；集群内 Paxos 故障切换官方指标为 **RPO=0、RTO<8s** |

可见两者都是**逻辑中心化但物理高可用**，真正的风险在性能瓶颈与故障恢复依赖，而非"挂一个就全挂"。两点需特别强调：(1) RS 虽有多副本自动重选，但官方仍用 **SPOF** 标签，故本章不写"RS 不是单点"，而写"非对称、残留 SPOF 风险"；(2) **RPO=0、RTO<8s 仅限 OceanBase V4.x LS/Paxos 多副本、少数派故障或多数派完整的集群内故障切换场景**，不绑定 RS 本身、不外推到单副本、shared-storage 或跨云形态。

**运维复杂度。** TiDB 的分片对 DBA 几乎透明，但海量 Region 集群需要专门调优(massive-regions 最佳实践、Hibernate Region 抑制心跳)，Region 过多会增加心跳、调度、raftstore 扫描和元数据压力。OceanBase 需要 DBA 在设计期就规划分区键、Primary Zone、Locality、Unit 分布；Tablet 多而 LS 少可降低复制组开销，但 LS 过载或 Tablet 迁移会影响位置缓存与事务路径，运维心智更偏"关系数据库 + 容量规划"。可观测方面，TiDB 可用 `SHOW TABLE REGIONS` 观察 Region ID、start/end key、Leader、peers 与估算读写流量；OceanBase 在 v4.2.5_CE 提供了一组用户可查的系统视图来观察 Tablet/LS 映射与 replica 分布——`DBA_OB_TABLET_TO_LS` / `CDB_OB_TABLET_TO_LS`(Tablet 到 LS 的归属)、`DBA_OB_LS` / `CDB_OB_LS`(LS 元信息)、`DBA_OB_LS_LOCATIONS` / `CDB_OB_LS_LOCATIONS`(LS 副本分布)、`DBA_OB_TABLET_REPLICAS` / `CDB_OB_TABLET_REPLICAS`(Tablet 副本)，底层内部表为 `__all_tablet_to_ls`(视图随版本可能增减，落地前以目标版本文档为准)。 〔文献[7-8,18]〕

## 1.8 反例与代价

**TiDB 的代价。**

- **小 Region 数量膨胀**：表/分区极多且每个都小，会产生数十万到数百万 Region，PD 调度与 Raft 心跳开销急剧上升；需靠 Region Merge、Hibernate Region、调大 `region-split-size` 缓解。
- **大 Region 迁移迟滞**：Region 过大(如误配或 V2 的 10GiB 级)，snapshot 传输超时、补副本/再平衡变慢、热点无法隔离，大范围查询抖动也会变重。
- **range 编码下的写热点与单 key 边界**：自增主键/单调 key 会把写集中到 key 空间末尾的单个 Region，必须用 `AUTO_RANDOM` / `SHARD_ROW_ID_BITS` + `PRE_SPLIT_REGIONS` 预打散(其中 `PRE_SPLIT_REGIONS` 预切分出的 Region 数为 `2^(PRE_SPLIT_REGIONS)`)。但若热点是单 key 或极窄 key range(同一账户余额、同一全局计数器、同一状态行)，Region split 无法把一个 key 拆成多个共识单元，这属于设计边界，需从业务 key 设计、聚合、缓存或异步化入手。这是 Range 分片相对 Hash 分片的固有代价。
- **Region 不是 SQL 语义边界**：`SHOW TABLE REGIONS` 的输出只是 Region 与表/索引 key range 的当前交集，不是长期稳定的表结构属性；merge、split、placement rule 与手工 split 都会改变它，不应据此把"表分片数"等同事务参与者数。

**OceanBase 的代价。**

- **分区设计前置**：分片粒度依赖用户在 DDL 期声明的分区。Partition 仍决定数据裁剪、局部索引、并行执行与 Tablet 数量；过细 Partition 会增加元数据、DDL、均衡与统计信息复杂度，过粗 Partition 会限制并发、迁移与热点隔离。**LS 不是替业务修复分区键选择的魔法层**；设计期没拆好，运行期热点难以像 Region 那样自动细化(4.3.x 前无自动 Tablet 分裂，见 §1.3)。
- **LS 粒度切主放大影响面**：多个 Tablet 共享一条 LS 日志，LS Leader 抖动、日志盘压力或 replay 延迟可能同时影响这些 Tablet，而非单个分区；但具体影响大小取决于版本、租户资源、locality、Primary Zone 与 workload，故本章不给出固定阈值或默认 LS 个数（推测）这类难以稳定成立的数字。
- **RS 依赖**：元数据/调度强依赖 sys tenant 的 RS，RS 异常的爆炸半径较大(官方将其类比为 "MetaServer 的 Achilles heel"，见 §1.7)。

**不要把 benchmark 当作分片架构优劣排名。** PALF 论文与官方博客说明了 OceanBase 4.x 在日志复制与架构演进上的动机，TiDB 文档也说明了 Region 调整对性能的影响；但任何 TPC-C、sysbench 或内部性能数字都受硬件、版本、参数、数据分布、压测脚本与利益相关方影响。例如 OceanBase 历史上 707M tpmC 的纪录(TPC-C Result ID 120051701、OceanBase v2.2 Enterprise Edition、Alibaba Cloud ECS、1554 data nodes、2020，官方 FDR 记录日 2020-05-17)不代表 4.x/CE/小集群形态。本章只引用机制，不用 benchmark 做绝对排名。

**取舍总结**：TiDB 用"细粒度、自动、对用户透明"换来了海量分片下的控制面压力与对预打散的依赖；OceanBase 用"设计期建模、共识组收拢、Tablet 可迁移"换来了更低的共识固定开销，代价是分片调整不如 TiDB 自动、灵活。两者并无谁更先进之分，更应理解为把复杂度放在了不同位置（推测）的取舍。

## 1.9 测试开发视角的验证点

**可测试功能场景。**

- TiDB：创建普通表、分区表、二级索引表，批量写入后用 `SHOW TABLE REGIONS` 验证 record key、index key、Region ID、start/end key、Leader Store、peers 是否按预期变化；对递增主键执行 `SPLIT TABLE ... REGIONS` 预切分(可验证 `PRE_SPLIT_REGIONS` 切出 `2^(PRE_SPLIT_REGIONS)` 个 Region)并配合 scatter，比较热点 Region 数量、写入延迟与调度完成时间；开/关 `split-region-on-table` 验证 Region 是否按表对齐；调大或调小 `region-split-size` 前后记录 Region 数、split 频率与大查询耗时；触发 Region merge 观察小 Region 数量与 raftstore 负担变化。
- OceanBase：创建单分区表、多分区表、带本地索引的表，验证 Partition 与 Tablet 的数量关系及其与 LS 的从属(v4.2.5_CE 可查用户视图 `DBA_OB_TABLET_TO_LS` / `DBA_OB_LS_LOCATIONS` / `DBA_OB_TABLET_REPLICAS`，底层内部表为 `__all_tablet_to_ls`；视图随版本可能增减，跨版本以目标版本文档为准)；在扩容、缩容、Primary Zone 或 locality 调整后观察 Tablet-to-LS 映射与 LS replica 分布是否变化；构造单 LS 内多 Tablet 事务与跨 LS 事务，比较提交路径、延迟与重试行为；在 4.4.x 上灌入超过 `auto_split_tablet_size` 阈值的数据验证自动分裂是否触发。

**可注入的失效模式。**

- TiDB：kill TiKV Store 观察各 Region Raft 重选 Leader 与 PD 补副本；制造单调写热点观察 load-base-split 是否触发(连续 10s 超 QPS/字节/CPU 阈值)；kill PD Leader 观察 TSO/调度暂停与恢复；手工 split 与 merge 期间做读写混合压测观察抖动。
- OceanBase：kill OBServer 观察 LS 切主与其下 Tablet 整体恢复；制造少数 Follower 故障验证多数派完整时服务保持；kill `__all_core_table` leader 所在 OBServer 观察 RS 在新节点重启(集群内 Paxos 切换官方 RPO=0、RTO<8s，见 §1.7)；Tablet transfer 进行中注入故障观察任务取消/重试与位置缓存刷新。每个失效实验都应记录版本、部署模式、自建或 Cloud 形态、租户/资源规格、数据分布、写入 key 模式与观测命令。

**关键压测与观测指标。** 不应只看总 QPS，至少要拆成：单分片写入延迟、跨分片事务延迟、split/transfer 前后 P95/P99 抖动、Leader 分布倾斜、Region/Tablet/LS 数量变化、调度 backlog、snapshot 或迁移流量、读写放大、失败重试次数。TiDB 可用 `SHOW TABLE REGIONS` 的 `WRITTEN_BYTES`、`READ_BYTES`、`APPROXIMATE_SIZE(MB)`、`APPROXIMATE_KEYS` 做辅助判断，但官方文档提醒这些值是 PD 基于 heartbeat 的估算而非精确统计。**具体 Prometheus metric 名、PromQL 裸名、OceanBase 内部视图/metric 名本章未核验到可长期稳定引用的集合，一律不编造**；测试报告可按用途归类(Region split 相关、PD operator 相关、hot region 相关、LS leader 切换相关、Tablet transfer 相关 metric)，但落地到自动化监控规则前必须按目标版本文档与实际 `/metrics` 输出二次确认。

## 1.10 容易误解点

1. **"一张表恒等于一个 Region / 一个分区对应一个共识组"——错，但要分清默认与物理可能。** TiDB 默认 `split-table=true`，每张新表起步是一表一 Region；一张表随数据增长会被切成多个 Region，而 Region 在物理上**可以**跨表(由 `split-region-on-table=false` 这一另外的 TiKV 参数决定，但跨表并非默认常态)。不要把决定"每表一 Region"的 `split-table`(默认 true)与允许"Region 跨表"的 `split-region-on-table`(默认 false)混为一谈。OceanBase 4.x 也不再是"一分区一 Paxos 组"，而是多个 Tablet 共享一个 LS 的 Multi-Paxos；分区/Tablet 退化为 LS 内被管理的存储对象。

2. **"OceanBase 4.x 的 Partition 就是 Paxos Group / LS 只是换了名字的 partition leader"——错。** PALF 论文说明 1.0–3.0 以 table partition 作为事务、日志、数据存储基本单位，4.0 重新设计为 Stream/LS；官方文档说明 Partition 是逻辑概念、Tablet 是存储对象、LS 承载若干 Tablet 与 redo logs。LS 是把共识粒度从分区上移到 LS 的实质性架构变化：共识组数量从分区数量级降到 LS 数量级，Tablet 可在 LS 间迁移而不重建共识组，这正是 Paetica 单机-分布式一体化的基础(见 §1.3)。

3. **"PD/RootService 是单点，挂了就全挂"——不准确；但也不能反过来说"RS 完全没有单点风险"。** 二者都是逻辑中心化但物理高可用(PD=etcd/Raft 对称多副本；RS 随 `__all_core_table` leader 切换且保证全局唯一在跑)，但二者性质不对称：OceanBase 官方对 RS 仍明确保留 **SPOF** 措辞，因此准确表述是"RS 非对称、残留 SPOF 风险、靠 Paxos 自动重选可恢复"，而非"RS 已无单点"。PD 也不是 Region Raft 数据面的读写单点。真正的风险维度是性能瓶颈(TSO/GTS、调度)与故障恢复依赖，必须按多个维度分别陈述(见 §1.7)。

4. **"TiKV Region 默认 96MB"——对 8.5.x 是过时值。** 8.5.x 默认 `region-split-size=256MiB`；96MB 是旧默认。引用旧博客时极易踩坑(切换版本归属见 §1.3 版本口径备注)。

5. **"OceanBase 4.4.1 才引入自动 Tablet 分裂"——不准确。** 自动分裂功能在 4.3.x 已具备(`enable_auto_split` / `auto_split_tablet_size` 在 4.3.5.x 即可用)；4.4.1_CE 做的只是把 `auto_split_tablet_size` 的默认值上调(见 §1.3)，变更类型是"默认值调整(Modified)"而非"功能新增(New)"。把调参误读为"引入功能"是常见的版本归因错配。

6. **"分片越小越好"——危险的直觉。** TiDB Region 过小会导致 Region 数过多、资源消耗与性能回退；OceanBase 过多 Tablet/Partition 会增加元数据、均衡与 DDL 成本。分片粒度应服务于热点隔离、迁移单位、并发度与事务边界，而不是追求数量。

## 1.11 本章结论

1. TiDB 把物理分片、共识分片、调度分片、单分片事务边界统一为同一个对象 **Region**(= Key Range = Raft Group)，分片边界由 size/keys/load/split-key 在运行时自动涌现，对用户基本透明；Region 与 SQL 表、索引、Partition 非一一对应。
2. OceanBase 4.x 把这几类分片**解耦**：Partition(逻辑)→ Tablet(物理存储对象，常 1:1 物理分区)→ Log Stream(共识/调度/事务参与者)，LS 与 Tablet 是一对多、可重绑定；Tablet 不自带日志流，归属于所在 LS。
3. OceanBase 4.0 用 **Log Stream 取代早期"每分区一个 Paxos 组 / partition leader"模型**，把共识组数量从分区数量级降到 LS 数量级，使 Tablet 可在 LS 间迁移而不重建共识组。这一改动旨在降低复制组与事务参与者开销，并支撑单机-分布式一体化。
4. TiDB 的 8.5.x 默认 Region 大小为 **256MiB**(`SPLIT_SIZE=256MiB`)，并有按 QPS(3000/7000)、字节(30/100MiB)、CPU(0.25/0.75)阈值的 load-base-split；旧资料的 96MB 已过时，默认值切换的版本归属在公开资料中有口径差异(见 §1.3)，不把变更起点当确定事实。
5. Region 与 SQL 表**非严格一一对应**：默认 `split-table=true`，每张新表起步一表一 Region，大表会分裂为多 Region；`split-region-on-table=false`(另一参数)使 Region 在物理上可跨表，但跨表非默认常态。OceanBase 的 Partition/Tablet 边界由用户 DDL 声明，粒度可控但需设计期前置。
6. PD 与 RootService 都是**逻辑中心化但物理高可用**，不是"单副本失效即全挂"的朴素单点；但二者不对称——PD 是 etcd/Raft 对称多副本，而官方对 RS 仍保留 **SPOF** 措辞，RS 被官方类比为 MetaServer 的 Achilles heel。集群内 Paxos 故障切换的官方指标为 **RPO=0、RTO<8s**(仅限 V4.x LS/Paxos 多副本、少数派故障或多数派完整场景)。真正风险在 TSO/GTS 性能瓶颈与故障恢复依赖。
7. OceanBase 自 4.3.x 起已具备**自动 Tablet/分区分裂**(`enable_auto_split` / `auto_split_tablet_size`)，4.4.1_CE(2025-10-24)做的是默认值调整而非功能引入(详见 §1.3)，CE/企业版差异需以发布说明为准。
8. 测试开发验证分片设计时应同时覆盖正常路径、热点 split/transfer、Leader 故障、路由缓存刷新与调度 backlog，而不是只看最终 QPS；两者不存在"谁更先进"，更宜视为把复杂度放在控制面与预打散、还是放在分区建模与共识收拢上的不同取舍（推测）。

## 1.12 参考文献

[1] TiDB Computing. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tidb-computing/.
 （支撑:§1.2 行/索引到 KV 的 t{tid}_r{rid}/t{tid}_i{iid} 编码、PartitionID record prefix 与有序 key 空间。）
[2] TiDB Configuration File. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tidb-configuration-file/.
 （支撑:§1.2/§1.10 split-table 默认 true(每张新表起步一表一 Region)与 TiKV coprocessor split-region-on-table(默认 false，允许 Reg）
[3] SHOW TABLE REGIONS. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/sql-statement-show-table-regions/.
 （支撑:§1.2/§1.5/§1.9 一表对应多 Region、START_KEY/END_KEY 含表前缀、record/index 各自成 Region、估算读写字段为 heartbeat 估算。）
[4] Tune Region Performance. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tune-region-performance/.
 （支撑:§1.2/§1.7 Region 默认大小口径(v8.4.0 文档表述)、region-split-size、Region 过多的资源开销。）
[5] Load Base Split. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/configure-load-base-split/.
 （支撑:§1.2 load-base-split 的 QPS(3000/7000)、字节(30/100MiB)、CPU(0.25/0.75)阈值与连续 10s 判定，与源码常量交叉验证。）
[6] Best Practices for PD Scheduling. 官方文档[EB/OL]. https://docs.pingcap.com/best-practices/pd-scheduling-best-practices/.
 （支撑:§1.2/§1.6/§1.7 PD 通过 Store/RegionHeartbeat 收集信息并执行 balance/hot region/merge/failure recovery 调度。）
[7] TimeStamp Oracle (TSO) in TiDB. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tso/.
 （支撑:§1.7 PD/TSO 为 SQL 事务分配时间戳、PD 异常不能只按 Region Raft 数据面理解。）
[8] Understanding Raft Region Size: TiDB Performance & Recovery. 官方博客[EB/OL]. https://www.pingcap.com/blog/understanding-raft-region-size-tidb-performance-recovery/.
 （支撑:§1.7/§1.8 小 Region 爆炸 vs 大 Region 迁移塌陷、256MiB 折中、Hibernate Region。）
[9] TiKV Deep Dive — Data Sharding. 官方博客[EB/OL]. https://tikv.org/deep-dive/scalability/data-sharding/.
 （支撑:§1.2 为何用 Range 而非 Hash、split/merge 只改 meta 不搬数据。）
[10] TiKV Deep Dive — Multi-raft. 官方博客[EB/OL]. https://tikv.org/deep-dive/scalability/multi-raft/.
 （支撑:§1.2 Multi-Raft 的 batch 化 event loop 与多个 Raft Group 的管理。）
[11] TiKV Deep Dive — RocksDB / split-check. 官方博客[EB/OL]. https://tikv.org/deep-dive/key-value-engine/rocksdb/.
 （支撑:§1.2 split-check 用 SST TableProperties 估算 Region 近似大小、无需全量扫描。）
[12] OceanBase Developer Guide — Architecture. 官方文档[EB/OL]. https://oceanbase.github.io/oceanbase/architecture/.
 （支撑:§1.3/§1.4/§1.5 Partition/Tablet/LS/OBServer/Zone/Tenant/Unit 定义，"one log stream corresponds to multiple Tablet）
[13] OceanBase Database architecture V4.3.5. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001971022.
 （支撑:§1.3/§1.6/§1.7 Tablet 对应特定 LS、一个 LS 关联多个 Tablet、balancing layer、单 LS/跨 LS 事务路径。）
[14] OceanBase Paetica: A Hybrid Shared-nothing/Shared-everything Database. 论文，VLDB 2023[EB/OL]. https://www.vldb.org/pvldb/vol16/p3728-xu.pdf.
 （支撑:§1.3/§1.10 Log Stream 模型取代早期 partition-leader、single-machine log stream design、单机-分布式一体化动机。(PDF 正文未逐行核验，以官方架构文）
[15] PALF: Replicated Write-Ahead Logging for Distributed Databases. 论文，VLDB 2024[EB/OL]. https://www.vldb.org/pvldb/vol17/p3745-xu.pdf.
 （支撑:§1.3/§1.10 1.0–3.0 partition 作为事务/日志/存储单位、4.0 引入 Stream/LS 降低复制组与事务参与者开销、PALF 与数据库协同设计抽象为原语。(PDF 正文未逐行核验，以官方博客）
[16] Introducing OceanBase 4.2.5 LTS for TP Scenarios. 官方博客/发布说明[EB/OL]. https://en.oceanbase.com/blog/17414420480.
 （支撑:§1.3 v4.2.5_CE 的 transfer-based partition、LS 副本级 O&M(add/delete/change-type/migrate)、修改 Paxos 副本数、取消容灾任务。已与 oc）
[17] Release v4.4.1_CE — oceanbase/oceanbase. 发布说明[EB/OL]. https://github.com/oceanbase/oceanbase/releases/tag/v4.4.1_CE.
 （支撑:§1.3 4.4.1_CE(2025-10-24)auto_split_tablet_size 默认值由 128MB 调整为 2GB(Modified)、global_index_auto_split_policy）
[18] 7 Key Technologies to Ensure High Availability in OceanBase. 官方博客[EB/OL]. https://en.oceanbase.com/blog/2615184384.
 （支撑:§1.7 RS 是 __all_core_table leader 所在 OBServer 上的服务组、全局唯一、"MetaServer 的 Achilles heel"且 "RS still poses the r）
[19] RPO and RTO Explained. 官方博客[EB/OL]. https://en.oceanbase.com/blog/rpo-and-rto-explained.
 （支撑:§1.6/§1.7 集群内 Paxos 故障切换官方指标 RPO=0、RTO<8s(对应金融级容灾)，据此明确仅限 V4.x LS/Paxos 多副本场景。）
[20] tikv/tikv @ release-8.5 — components/raftstore/src/coprocessor/config.rs、split_check/、store/worker/split_config.rs. 源码[EB/OL]. https://github.com/tikv/tikv/tree/release-8.5/components/raftstore.
 （支撑:§1.2 SPLIT_SIZE=256MiB、RAFTSTORE_V2_SPLIT_SIZE=10GiB、BATCH_SPLIT_LIMIT=10、coprocessor split-region-on-t）
[21] pingcap/tidb @ release-8.5 — pkg/tablecodec/tablecodec.go. 源码[EB/OL]. https://github.com/pingcap/tidb/tree/release-8.5/pkg/tablecodec.
 （支撑:§1.2 表/索引/Partition 编码进入 KV keyspace、Region 与 SQL 表不一一对应的源码级定位。）
[22] tikv/pd @ release-8.5 — pkg/schedule/splitter/region_splitter.go、pkg/schedule/checker/{merge_checker,split_checker}.go、pkg/schedule/config/config.go、pkg/keyspace/keyspace.go. 源码，commit 6dce4a68e3[EB/OL]. https://github.com/tikv/pd/tree/release-8.5/pkg.
 （支撑:§1.2 PD RegionSplitter 下发 split、placement/label boundary 触发 split、MergeChecker/splitCache 抖动抑制、keyspace 的）
[23] oceanbase/oceanbase @ v4.2.5_CE — src/storage/ls/ob_ls.h、src/share/ob_ls_id.h、src/storage/tablet/{ob_tablet.h,ob_tablet_binding_helper.h}、src/share/tablet/ob_tablet_to_ls_operator.h、src/share/ls/ob_ls_info.h、src/logservice/{ob_log_handler.h,palf/}、src/objit/include/objit/common/ob_item_type.h、src/rootserver/{ob_transfer_partition_command.*,ob_disaster_recovery_worker.h}、src/share/inner_table/ob_inner_table_schema_constants.h. 源码，commit e7c676806f[EB/OL]. https://github.com/oceanbase/oceanbase/blob/v4.2.5_CE/src/storage/ls/ob_ls.h.
 （支撑:§1.3 ObLS 同时持有 ls_tablet_svr_ 与 log_handler_、TOTAL_INNER_TABLET_NUM=3、SYS_LS_ID=1/MIN_USER_LS_ID=1000、ObTablet）
[24] oceanbase/oceanbase @ 4.4.x — src/rootserver/{ob_tenant_balance_service.*,ob_partition_balance.h,ob_ls_balance_helper.h,ob_balance_task_execute_service.h,ob_transfer_partition_task.h}、src/share/scheduler/ob_partition_auto_split_helper.h、src/rootserver/ddl_task/ob_partition_split_task.cpp、src/logservice/ipalf/. 源码，commit d4bef8d29a[EB/OL]. https://github.com/oceanbase/oceanbase/tree/4.4.x/src/rootserver.
 （支撑:§1.3 4.4.x 均衡调度的具体类:ObTenantBalanceService(统筹)、ObPartitionBalance、ObLSBalanceTaskHelper、ObBalanceTaskEx）
## 1.13 信息可信度自评

- **官方文档明确**：TiDB 的 KV 编码、`split-table`(默认 true，每新表一 Region)与 `split-region-on-table`(默认 false，允许跨表)的区分、load-base-split 阈值、Region 大小演进、`SHOW TABLE REGIONS` 行为；OceanBase 的 Partition/Tablet/LS 定义与"一 LS 对多 Tablet"、Tablet 在 LS 间迁移、balancing layer。这部分可信度最高。
- **源码级(commit-pinned)**：TiKV `SPLIT_SIZE=256MiB`/`RAFTSTORE_V2_SPLIT_SIZE=10GiB`/`BATCH_SPLIT_LIMIT=10`/load-base-split 常量；PD `RegionSplitter`/`MergeChecker`/`split_checker`/keyspace,以及 `defaultMaxMergeRegionSize = 54`(MiB)与 `GetMaxMergeRegionKeys()` 默认 54×10000 = 540000 keys(`RegionSizeToKeysRatio = 10000`);OceanBase `ObLS` 结构、`SYS_LS_ID=1`/`MIN_USER_LS_ID=1000`/`TOTAL_INNER_TABLET_NUM=3`、`ObTablet` 携带 ls_id、tablet-to-LS 映射、ObLogHandler 封装 PALF、4.4.x ipalf 抽象层;v4.2.5_CE transfer-partition(`T_TRANSFER_PARTITION`/`_TO_LS`)与 LS 副本级 O&M(`ObAdminAlterLSReplicaArg` 各 task)、用户视图 `DBA_OB_TABLET_TO_LS`/`DBA_OB_LS`/`DBA_OB_LS_LOCATIONS`/`DBA_OB_TABLET_REPLICAS`;4.4.x 均衡调度类 `ObTenantBalanceService`/`ObPartitionBalance`/`ObLSBalanceTaskHelper`/`ObBalanceTaskExecuteService` 与自动分裂 `ObServerAutoSplitScheduler`。这部分为本章最硬的事实。
- **论文级**：Paetica(VLDB 2023)的 LS 模型动机、PALF(VLDB 2024)的设计思想与 1.0–3.0→4.0 演进。两篇 PDF 正文未逐行核验，论断以官方架构文档与发布博客交叉印证，属"论文存在但正文未逐行核验"。
- **工程推测**：两者"复杂度放置位置不同""不存在谁更先进"、LS 聚合的故障半径、压测维度建议，为基于公开机制的取舍判断，已标注（推测）。
- **仍不确定**：OceanBase 具体 Prometheus metric / PromQL 裸名与可长期稳定引用的 metric 名集合、TiDB/OceanBase 内部日志字段与错误码、默认 LS 个数(取决于版本/租户资源/locality)、TiDB Cloud Starter/Essential 隔离机制——均按规范写"需进一步查证"或不做实现层断言，不堆叠免责。(原列于此的 PD `max-merge-region-size/keys` 默认值、OceanBase Tablet/LS 用户视图名、4.4.x 自动均衡调度类名，已在本轮经源码核验升级为事实，见上。)

---


# 第 2 章 路由

## 2.1 本章核心问题

本章核心问题不只是"数据在哪台机器"，而是"定位错了之后如何恢复"。一条 SQL 进入分布式数据库后，在真正读到或写到数据之前，系统必须回答一个看似简单却贯穿全栈的问题：这条语句涉及的数据，现在到底落在哪台机器、哪个副本上?这就是"路由"要解决的上半句；它同样要回答下半句——当缓存里的位置信息错了之后，系统如何自愈。

分片(详见第 1 章)决定了数据如何被切成 Region / Tablet 等物理单元，共识(详见第 3 章)决定了每个单元有一个 Leader 与若干 Follower，而路由则是把"一段 key range / 一组分区键"映射到"一个具体的 IP:port + 副本角色"的过程。它处在控制路径与数据路径的交界处：路由元数据的来源、缓存、刷新属于控制路径，而每条语句据此把请求投递到目标节点属于数据路径。

路由之所以是底层关键，有三个原因。第一，正确性：如果把强一致读路由到一个已经不是 Leader 的旧副本，可能读到陈旧数据；如果把请求发到一个已分裂或已合并的 Region，会读到错误的 key range。第二，性能：多一跳代理、多一次缓存未命中、多一次跨机房 RPC，都会直接体现在 P99 延迟上。第三，可用性：Leader 切换、节点宕机、Region 分裂是常态事件，路由层必须在毫秒到秒级内自愈，否则一次普通的调度就可能引发业务侧大面积报错。

本章用三条示例 SQL 贯穿 TiDB 与 OceanBase 两个系统的路由路径(假设表 `orders` 以 `order_id` 为主键/分区键，`user_id` 为非分片/分区键)：

```sql
SELECT * FROM orders WHERE order_id = 1001; -- 点查
SELECT * FROM orders WHERE order_id BETWEEN 1000 AND 2000; -- 范围查询
SELECT * FROM orders WHERE user_id = 88; -- 非分片/分区键
```

本章只讨论"定位数据与转发请求"的边界，不展开分片策略、共识协议、事务提交与 MVCC 细节。这里要避免两个常见混写：第一，TiDB 的 RegionCache / PD 路由刷新不等于事务的 TSO 或 primary lock 处理；第二，OceanBase 的 ODP location cache 与 OBServer 内部的 Tablet→LS、LS Leader 关系，不能被简化成一个公开参数或单一 metric。

本章还特别强调一个高风险误区："能路由到 Leader"不等于"所有读都必须走 Leader"。两个系统都提供 Leader 读、Follower 读、Stale/弱一致读等多种模式，路由策略随一致性要求而分叉。把这一点笼统化，是路由章节最常见的事实错误。

## 2.2 TiDB 的实现

### 2.2.1 组件与路由元数据来源

TiDB 的路由是客户端侧路由(client-side routing)：无状态的 TiDB Server 自身充当 SQL 计算节点，内部嵌入了 KV 客户端 `client-go`，由它直接决定请求该发往哪个 TiKV Region Leader。整条链路是 `client → TiDB Server → RegionCache/PD → TiKV Region Leader`，没有独立的数据面代理。路由的入口可以概括为一句话：把"SQL 逻辑范围"翻译成"有序 KV key range"，再翻译成"Region → store/peer"。

路由元数据的权威来源是 PD。PD 在 `pkg/core/region.go` 中以 `RegionInfo` 结构维护每个 Region 的元数据：`meta *metapb.Region`、`leader *metapb.Peer`、`learners/voters/witnesses`、`pendingPeers`、`downPeers`、读写 bytes/keys、approximate size/keys、`buckets` 等字段；`RegionFromHeartbeat` 则从 TiKV 的心跳构造这些信息。该结构提供 `GetLeader()`、`GetStartKey()`、`GetEndKey()`、`GetRegionEpoch()` 等访问器。值得注意的是，`RegionInfo` 还带一个 `RegionSource` 枚举(`Storage / Sync / Heartbeat`)，注释明确指出 "Storage means this region's meta info might be stale"——PD 自身就区分 Region 元数据的新鲜度来源，这是后续缓存一致性讨论的起点。

PD Leader 把 Region 信息实时同步给 Follower，PD Follower 在内存中维护这些信息并可直接响应查询请求，因此 TiDB 总能拿到当前的 Region 元数据。TiDB v8.5.0 起 Active PD Follower 特性 GA 后，PD Follower 也可处理 `GetRegion` / `ScanRegions` 请求，降低 Region 信息查询对 PD Leader 的 CPU 压力——这个改进影响的是路由元信息查询的可扩展性，并不改变 SQL 事务的 TSO 边界。

### 2.2.2 RegionCache：客户端侧缓存

如果每条 SQL 都向 PD 查一次 Region 位置，PD 很快会成为瓶颈。TiDB 因此在 `client-go` 内维护 RegionCache——一份本地的 key range → Region → Leader/Peer 映射缓存。需要澄清一个常见的源码归属误区：`client-go/internal/locate/`(RegionCache)在 TiDB 主仓库中不是 vendored 源码，而是外部 Go module 依赖，`go.mod` 中精确锁定为 `github.com/tikv/client-go/v2 v2.0.8-0.20260615130046-a2b634d170d9`(release-8.5 @ 67b4876bd57b)。TiDB 在 `pkg/store/copr/region_cache.go` 中以 `type RegionCache struct { *tikv.RegionCache }` 嵌入式封装复用 client-go 的 RegionCache。

RegionCache 的核心入口在 client-go 中：`func (c *RegionCache) LocateKey(bo *retry.Backoffer, key []byte) (*KeyLocation, error)`，注释为 "LocateKey searches for the region and range that the key is located"(client-go `internal/locate/region_cache.go`，函数原文取自 master 分支)。对范围谓词，TiDB 侧 `region_cache.go` 的 `SplitKeyRangesByLocations` 把 TiDB 逻辑 `KeyRanges` 转成 `tikv.KeyRange`、调用 `BatchLocateKeyRanges`，再按 `KeyLocation.EndKey` 把跨 Region 的连续范围拆成 Region 对齐的若干段。也就是说，SQL 层可以把一个范围谓词整体交给 coprocessor，但在发送前仍要切成 Region 对齐的任务。

### 2.2.3 数据路径：三条 SQL 的路由

点查 `order_id = 1001`：若 `order_id` 是 clustered primary key 或能被唯一索引精确定位，优化器会得到一个或少量点 key/range。TiDB 把主键编码为行 key(形如 `t{tableID}_r{handle}`，详见第 5 章)，调用 `LocateKey` 在 RegionCache 中定位该 key 所属的单个 Region 及其 Leader 地址，直接向该 TiKV 节点发送 Get 请求。缓存命中则得到目标 Region 与 Leader store 地址，缓存未命中或过期则向 PD 查询。优化器对这类查询走 `Point_Get` / `Batch_Point_Get` 算子，绕过 DistSQL/coprocessor 的重路径(`planner.TryFastPlan()` 为 PointGet 提供快捷路径)。默认强一致读与写请求发往 Region Leader。

范围查询 `order_id BETWEEN 1000 AND 2000`：这是一段连续的 key range，可能横跨多个 Region。TiDB 在 copr 层用 `SplitRegionRanges`(内部调用 `SplitKeyRangesByLocations`)把 key range 按 Region 边界切成若干 `LocationKeyRanges`(其字段注释明确 "Location is the real location in PD")；若开启 bucket，还会用 `loc.LocateBucket(...)` 在 Region 内进一步细分。每个子范围生成一个 cop task，并发下发到各自的 Region Leader(或符合 replica read 策略的副本)，TiKV 侧由 coprocessor 执行 TableScan/Selection 等下推算子，TiDB 汇总结果。官方表述为："TiDB groups the related keys by Region… splits the task by Regions and sends them to TiKV concurrently"。范围跨越的 Region 越多，请求扇出、结果合并与慢 Region 的尾延迟越明显。

非分区键 `user_id = 88`：`user_id` 不是主键。若 `user_id` 上有二级索引，优化器走 `IndexLookup`：先扫索引(索引 key 也是一段编码后的 key range，同样经 RegionCache 定位)拿到主键 handle，再回表点查——这是"索引 Region 扫描 + 主表 Region lookup"的两段路径，而非一次简单路由。若无可用索引，则退化为全表扫描：整张表的 key range 被切成覆盖所有数据 Region 的 cop task，广播式下发。这条 SQL 揭示了路由代价与分片/索引设计的耦合：非分区键查询天然要触达更多 Region。

### 2.2.4 控制路径与故障路径

TiKV 用 epoch 标记 Region 版本：epoch 的定义在 kvproto 的 `proto/metapb.proto` 中——`message RegionEpoch { uint64 conf_ver; uint64 version; }`，字段注释逐字为 `conf_ver`="Conf change version, auto increment when add or remove peer"、`version`="Region version, auto increment when split or merge"(增删 peer 时 `conf_ver` 自增，分裂/合并时 `version` 自增)。请求携带 epoch，TiKV 处理前比对，不一致即返回 RegionError；Region 既是 key range 又是 Raft 复制的基本单元，可分裂与合并(详见 TiDB VLDB 2020 论文)。常见 RegionError 及 TiDB 的应对：

- EpochNotMatch:Region 已分裂/合并/副本迁移，请求 epoch 过期。TiDB copr 层在 `coprocessor.go` 的 `handleCopResponse` 中检测到 `resp.pbResp.GetRegionError()` 后，先 `bo.Backoff(tikv.BoRegionMiss(), ...)` 退避，再用 `worker.store.GetRegionCache()` 重新 `buildCopTasks`，即失效缓存后重新定位 Region；在 mpp/batch 路径还会主动 `InvalidateCachedRegionWithReason(id, tikv.EpochNotMatch)`。RegionCache 侧由 `OnRegionEpochNotMatch` 处理，其注释为 "removes the old region and inserts new regions into the cache"(client-go 源码原文)。
- NotLeader：目标副本已非 Leader，通常因 Leader 转移引起。若响应里带新 Leader 提示，TiDB 据此更新本地路由并重发到新 Leader；否则把缓存 Region 标记为 `NoLeader` 失效原因。client-go 的 RegionCache 注释揭示一个微妙之处：启用 Joint Consensus(v5.0)+ Hibernate Region 后，Leader 可能长时间缺失且 PD 里 Leader 信息陈旧，解决办法是"若旧缓存 Region 的失效原因是 no-leader 就总是换一个 peer 试，虽然有小概率当前报 no-leader 的 peer 恰好成了 Leader，此时 TiDB 需重试一次"(client-go 源码原文)。其 `InvalidReason` 枚举为 `Ok / NoLeader / RegionNotFound / EpochNotMatch / StoreNotFound / Other`(源码原文)。
- StaleCommand：命令过期，通常因 Leader 切换或请求乱序。本地核查仅确认 TiDB copr 层在 EpochNotMatch 时失效缓存并重建 task；NotLeader / StaleCommand 的细粒度分类与 Leader 字段刷新逻辑在 client-go 的 RegionCache 中，具体处理路径以 client-go 仓库为准。

需要强调：正确性来自存储侧的共识与 MVCC 版本检查，路由缓存只是性能优化与请求定位层，不能绕过 Region epoch 与 Leader 校验。RegionCache 可能过期，但 TiKV 的 Leader/epoch 校验会拒绝错误请求，系统随即失效缓存、退避并重建任务。

### 2.2.5 读路由的分叉：不只走 Leader

默认 `tidb_replica_read = leader`，所有读走 Leader。但 TiDB 提供多档：`follower`、`leader-and-follower`、`prefer-leader`、`closest-replicas`、`closest-adaptive`、`learner`。

- Follower Read 仍保持强一致：Follower 先用 Raft 的 `ReadIndex` 与 Leader 交互拿到最新 commit index 再本地读；TiDB 用 `zone` 标签识别本地副本(TiDB 节点与 TiKV 节点 zone 相同即视为本地副本)。官方文档明确 TiDB 采用 round-robin(轮询)策略在 Region 的 Follower 间做负载均衡。早期资料据 Follower Read 文档声称的"coprocessor 连接级 / 点查事务级"负载均衡粒度，在该文档中并无对应表述；经核 client-go replica selector 源码，副本选择是**逐请求**粒度——每次 RPC 经 `newReplicaSelector(...)`(参数含 `req *tikvrpc.Request`)新建一个 selector，不在连接或事务上维持跨请求的轮询状态，因此原"连接级/事务级"粒度断言不成立。需注意一处版本差异：官方文档措辞为 round-robin，而 client-go master 的副本选择实为打分式(`ReplicaSelectMixedStrategy` 按 NotSlow/LabelMatches/PreferLeader/NormalPeer/NotAttempted 等位计分)，同分时由 `randIntn(...)` 随机打破平局，并非严格顺序轮询。
- Stale Read 牺牲实时性换吞吐：通过 `AS OF TIMESTAMP`、`TIDB_BOUNDED_STALENESS()`、`tidb_read_staleness`、`tidb_external_ts` 等读历史版本，可路由到任意副本而无需 `ReadIndex` 跨机房交互。它读的是某个时间点的历史版本，应用必须接受时间点语义，不能把 Stale Read 当强一致读使用。

涉及的官方文档/论文/源码模块：PD `pkg/core/region.go`(pd release-8.5 @ 6dce4a68e3e9)；TiDB `pkg/store/copr/region_cache.go`、`pkg/store/copr/coprocessor.go`(tidb release-8.5 @ 67b4876bd57b)；client-go `internal/locate/region_cache.go`(外部依赖，版本号见上，函数级以 client-go 仓库为准)；TiDB VLDB 2020 论文。 〔文献[1-5,10-14]〕

## 2.3 OceanBase 的实现

### 2.3.1 组件与路由元数据来源

![[f2_1.svg]]
**图 2-1　OceanBase 路由的 Partition 至 Tablet 至 Log Stream 三层结构**

与 TiDB 把路由内嵌进计算节点不同，OceanBase 的典型部署是代理侧路由(proxy-side routing)：客户端连 OBProxy / ODP，由 ODP 在连接层解析租户、表、分区键、读一致性与 location cache，尽量把请求转发到合适的 OBServer，再由 OBServer 内部执行 SQL 与访问 Tablet / LS。链路是 `client → ODP/OBProxy → 路由表 → OBServer`。需强调：ODP 不是必选，客户端也可直连 OBServer，但生产中几乎都走 ODP。

ODP 的路由层级可分三段：cluster routing、tenant routing 与表级(intra-tenant)routing。cluster routing 的关键是 cluster name 到 RS list 的映射；RS list 通常包含 RootService 所在 server，但不需要包含全部 OBServer。tenant routing 进一步把连接导向对应租户。进入表级路由后，ODP 才解析 SQL 中的表、分区键、读一致性，并查 location cache 选择后端 OBServer。

路由元数据的权威来源在 OBServer 侧的位置服务(location service)。OBProxy 客户端侧的路由表函数属 `oceanbase/obproxy` 独立仓库，不在 observer 主仓库 checkout 内；其权威元数据来自 sys tenant。ODP 从 sys tenant 获取集群 server 列表、partition 分布信息、zone 信息、tenant 信息。

OBServer 侧的位置服务在 `src/share/location_cache/`(observer 仓库，v4.2.5_CE @ e7c676806fda)。其核心类 `ObLocationService`(`ob_location_service.h`)把路由分为两类查询：

1. Log Stream 位置：`get(...)` 同步取 LS 全部副本地址、`get_leader(...)` 同步取 LS Leader 地址，`get_leader_with_retry_until_timeout(...)`(默认重试间隔 `retry_interval = 100000` 微秒，即 100ms，源码内联注释 `/*100ms*/`)在 Leader 缺失时重试到超时，以及 `nonblock_get/nonblock_get_leader/nonblock_renew` 非阻塞接口。经逐字检索 `src/share/ob_errno.h`，官方实际存在的相关返回码为 `OB_LS_LOCATION_NOT_EXIST = -4721`、`OB_LS_LOCATION_LEADER_NOT_EXIST = -4722`、`OB_LOCATION_LEADER_NOT_EXIST = -4654`(另有 `OB_LEADER_NOT_EXIST = -4209`)；早期资料曾写的 `OB_LS_LEADER_NOT_EXIST` 在官方源码中并不存在(系符号错配)，已据源码原文更正(见 §2.13)。
2. Tablet → Log Stream 映射：`get(tenant_id, tablet_id, ..., ObLSID &ls_id)` 把 tablet 解析到所属 LS(返回码 `OB_MAPPING_BETWEEN_TABLET_AND_LS_NOT_EXIST = -4723`)。

这印证了 OceanBase 路由是两段式：先 `tablet → LS`，再 `LS → leader/副本地址`，而非直接 `partition → 机器`。这与第 1 章结论一致：Partition(逻辑)→ Tablet(物理)→ Log Stream(共识/调度)三层解耦，LS 才是共识与路由的真正单元。源码 `src/storage/ls/ob_ls.h` 暴露 `get_ls_id()` 与 `get_tablet(tablet_id, ...)`，说明 Tablet 在 LS 内由 tablet service 管理。需要诚实指出：Tablet→LS 映射的内部缓存刷新细节、以及 Cloud 形态下的多层代理实现，公开资料不足，不在此编造（不确定）。

### 2.3.2 SQL 解析与分区裁剪

ODP 内置一个 SQL Parser，解析出数据库名、表名、partition key 与 hint:"identify SQL semantics and extract key routing information such as table names and partitioning keys"。据此选择最合适的 OBServer。对强一致 Leader routing，ODP 需要知道要访问的 partition ID 与 location：单分区表可由表名直接获取副本 location；多分区表需基于表名与 SQL 中的分区键计算 partition ID，再获取 leader/follower replica 信息。

真正的分区裁剪逻辑在 OBServer 的 SQL 层：`src/sql/optimizer/ob_table_location.h` 的 `ObTableLocation` 提供 `calculate_tablet_ids(...)`、`calculate_partition_ids_by_rowkey(...)` 等方法，把谓词/rowkey 计算成目标 tablet 集合。点查→单 tablet，范围/非分区键→多 tablet，基于谓词过滤无关分区的过程在执行计划里称为 partition pruning。

ODP 自身则用 `obproxy_expr_calculator`、`obproxy_part_mgr` 等模块在代理侧做分区键计算与 tablet 定位(`oceanbase/obproxy` 仓库 `src/obproxy/proxy/route/`，目录含 `ob_mysql_route`、`ob_ldc_route`、`ob_table_cache`、`ob_partition_cache`、`ob_partition_entry`、`ob_route_diagnosis`、`obproxy_expr_calculator`、`obproxy_part_mgr` 等)（待核实）。

### 2.3.3 数据路径：三条 SQL 的路由

点查 `order_id = 1001`：ODP 解析出分区键 `order_id`，计算出唯一目标 tablet(对应一个 index/partition)，经位置服务定位其所属 LS；若是强一致读，路由到该 LS 的 Leader 所在 OBServer，由 OBServer 生成或复用 local plan、访问对应 Tablet / LS——无需跨机 RPC，这正是代理侧路由的核心收益："the execution plan can be executed locally without remote procedure calls"。若 `orders` 是单分区表，ODP 不需要复杂的 partition ID 计算。

范围查询 `order_id BETWEEN 1000 AND 2000`：分区裁剪后可能命中多个 tablet，分布在不同 LS / OBServer。ODP routing 最佳实践指出 local plan 与 remote plan 都是单分区访问，ODP 的目标是尽量把 remote plan 收敛成 local plan；一旦跨多个分区，就需要由某个 OBServer 的执行层协调多个分区结果，形成 distributed plan(详见第 12 章优化器)。

非分区键 `user_id = 88`：`user_id` 非分区键，ODP 无法裁剪，只能视为全分区扫描——触达该表所有 tablet 对应的 LS，或退化为分布式计划。这是 OceanBase 的"路由失败"常见诱因：分区匹配失败时，路由诊断日志会记录路由细节(见 §2.9)。若 `user_id` 上建了全局索引，则全局索引本身是一张带自己分区的索引表，强一致读可用 index value 作为 partitioning key 计算全局索引表路由，先定位 index partition leader 再由执行层处理回表；若是局部索引，索引随主表分区，仍需扫所有分区(第 15 章索引展开)。这与 TiDB 的"索引 Region 扫描 + 主表 Region lookup"在形态上相似，差别在于 OceanBase 的路由单位是 Partition / Tablet / LS 组合，而非 KV Region。

### 2.3.4 路由表缓存、刷新与读路由分叉

![[f2_2.svg]]
**图 2-2　OBServer 强读 / 弱读副本判定决策链**

ODP 的路由表采多级缓存：先本地缓存，再全局缓存，最后向 OBServer 发起异步查询任务("first from the local cache, then from the global cache, and finally from the OBServer node by initiating an asynchronous query task")。

刷新走反馈触发机制：若某 OBServer 无法本地处理请求(如 Leader 已切走)，它把反馈放进响应包回给 ODP，ODP 据此决定是否强制更新本地缓存的路由表("the OBServer node places the feedback in the response packet… ODP determines whether to forcibly update the routing table")。

这里要纠正一个常见的过度强化：官方文档原文用的是 "Generally(通常)"而非 "only/仅"——原句为 "Generally, the routing table changes only when an OBServer node undergoes a major compaction or when leader switchover is caused by load balancing"。`Generally` 是 hedge，表明这是主要场景而非穷尽充要条件。事实上，路由表刷新/失效还有多类独立触发：系统租户约每 15 秒定时拉取最新信息；OBServer 返回 `OB_TENANT_NOT_IN_SERVER` 等特定错误码时失效缓存；集群拓扑变化(server 加入/移出)、副本变化(新增只读副本或 follower 迁移，ODP 从内部表感知后会失效该租户全部路由信息)、机器故障/副本切换(异步刷新 table location cache 检测后路由到新副本)等。因此本稿不再把"仅 major compaction / Leader 切换才变化"当作排他性事实，而表述为"通常成立的主因之一，但非穷尽条件"。

读路由按一致性分叉：强一致读与 DML 路由到 partition log stream 的 Leader；弱一致读及其它请求均匀分布到 Leader 与 Follower("strong consistency reads are routed to the OBServer node where the leader of the partition log stream is located, whereas… weak consistency reads… are evenly distributed")。读写分离文档进一步说明：业务可通过 hint 或 session/global 变量(如 `read_consistency(weak)` / `ob_read_consistency='weak'`)指定弱一致读，ODP 可用 `proxy_route_policy` 优先把弱一致读转发到 follower；若 follower 不可用可回退到 leader，或在 follower-only 策略下终止连接，同时受 follower latency threshold(滞后阈值)约束。需注意：OceanBase 弱一致读默认是主备均衡路由，官方措辞为"优先(preferentially)路由到 follower"，并不排除 Leader，因此不能描述为只在 Follower 间"全副本选"。多机房部署还有 LDC(Logical Data Center)路由：优先同 IDC、再同城、最后异地，以及读写分离的 RZone-first/RZone-only 等规则。

关于"强一致走 `get_leader`、弱一致走 `get`"的对应关系，本稿明确不作此断言。`get_leader/get` 是 Location Cache 模块为 SQL、事务、CLOG 等多模块提供位置缓存的接口，并非某个"SQL 执行层判定函数"；强/弱一致读的副本路由判定主要发生在 ODP(解析 `READ_CONSISTENCY(WEAK)` hint 后选 OBServer)与位置服务，不是一个干净映射到 `get_leader/get` 的单一判定函数，且弱读也可走 Leader。经核 OBServer SQL 层源码，副本判定的真正落点是 DAS 层而非 `get_leader/get`。优化器 `ObTableLocation::get_is_weak_read(...)` 依据一致性级别(hint `READ_CONSISTENCY(WEAK)`、session 变量、协议弱读标志；其中 DML 与 `SELECT ... FOR UPDATE`、内部表恒为强一致)算出 `is_weak_read`，据此设置 `loc_meta_.select_leader_`(DML/强读置 1、弱读置 0)。执行期 `ObDASLocationRouter::get_tablet_loc(...)` 再按 `select_leader_`(或重试遇 `OB_NOT_MASTER` 时)分叉——置位走 `nonblock_get_leader(...)`，否则走 `nonblock_get_readable_replica(...)`。也就是说，真正的判定变量是 `select_leader_`、底层调用是 `nonblock_get_leader/nonblock_get_readable_replica`，而非 Location Cache 的 `get_leader/get`，故仍不应把它简化为 `get_leader/get` 的 1:1 映射(见 §2.13)。

涉及的官方文档/源码模块：observer `src/share/location_cache/`、`src/share/ob_errno.h`、`src/storage/ls/ob_ls.h`、`src/sql/optimizer/ob_table_location.h`(oceanbase v4.2.5_CE @ e7c676806fda)；obproxy `src/obproxy/proxy/route/`(独立仓库，函数级以该仓库为准)；ODP routing / 路由诊断官方文档；OBProxy 模块与高可用官方博客。 〔文献[6-8,15-19]〕

## 2.4 核心差异对比

把前两节的实现收敛成对照，按路由位置、路由单元、元数据来源、失效触发等维度逐项比较如下。

**表 2-1　路由:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB | OceanBase | 影响 |
|---|---|---|---|
| 路由位置 | 客户端侧(client-go RegionCache 内嵌于 TiDB Server) | 代理侧(ODP/OBProxy 默认)或客户端直连 | TiDB 少一跳但 SQL 节点更"重"；OceanBase 多一跳代理但客户端轻 |
| 路由单元 | Region(= Key Range = Raft Group) | 两段式：Tablet → Log Stream(LS) | OceanBase 多一层 tablet→LS 解析，但 LS 与 tablet 一对多，路由表更稳定 |
| 元数据权威源 | PD(etcd/Raft 多副本)，未命中查 `GetRegion` / `ScanRegions` | sys tenant + OBServer 位置服务(`location_cache`),RS list 指向 RootService 入口 | 二者皆逻辑中心化、物理高可用(详见第 13 章) |
| 缓存层级 | RegionCache(单层，内存) | 本地缓存→全局缓存→OBServer 异步查询(多级) | OceanBase 缓存层次更多，刷新更"懒" |
| 失效/刷新触发 | RegionError(EpochNotMatch/NotLeader/StaleCommand)→ backoff + 重建 task | OBServer 反馈包 → 强制刷新；路由表"通常(Generally)"主因为 major compaction/Leader 切换，另有拓扑/副本变化、错误码失效、定时拉取等触发 | TiDB 由错误驱动即时失效；OceanBase 由反馈驱动 + 多类触发，稳态较稳 |
| 分裂/合并对路由的扰动 | Region 随 size/keys/load 运行时频繁分裂 → epoch 频繁变 → 缓存常失效 | Tablet 分裂较少、LS 与 tablet 解耦 → 路由表稳定 | TiDB 路由缓存抖动更频繁，但每次失效代价小 |
| 范围查询扇出 | 按 key range 切成多个 Region cop task，扇出以 Region 数为主 | partition pruning 后形成 local/remote/distributed plan，扇出以分区/计划为主 | 扇出单位不同：TiDB 看 Region 数，OceanBase 看分区与计划形态 |
| 强一致读路由 | 默认到 Leader；可 Follower Read(ReadIndex)/Stale Read | 到 LS Leader；弱一致读分布到 Leader+Follower;LDC 就近 | 两系统都区分强/弱一致，均非"只走 Leader" |
| 分区裁剪入口 | 优化器 + copr 层 `SplitRegionRanges` | ODP SQL Parser + OBServer `ObTableLocation::calculate_tablet_ids` | 一个在客户端/计算层切，一个在代理 + SQL 层算 |

## 2.5 正常路径图

![[f2_3.svg]]

**图 2-3　路由正常读写/调度路径**

正常路径的关键对照：TiDB 在 SQL 节点内部用 RegionCache 一次定位、并发下发、汇总；OceanBase 先在 ODP 内做 cluster/tenant 路由，再解析分区键、做 partition pruning、查多级缓存，转发到 LS Leader(强一致)或就近副本(弱一致)。

## 2.6 故障/异常路径图

图 2-4 展示路由失效与自愈(以 Region/LS Leader 切换为主线)：

![[f2_4.svg]]

**图 2-4　路由故障/异常路径**

故障路径揭示一个对称结构：两系统都"先用旧缓存试一次，失败再以错误/反馈驱动刷新"，差别在 TiDB 由 client-go 的 RegionError 即时驱动失效，OceanBase 由 OBServer 反馈包驱动强制刷新。两者都把缓存当作性能层，正确性由存储侧的 Leader/epoch/location 校验兜底。控制面(PD / sys tenant)短时不可用时，已缓存的路由仍可服务存量请求，但拓扑变更无法被感知，这是路由层"软依赖控制面"的体现(控制面定位详见第 13 章)。

## 2.7 性能、可靠性、运维影响

延迟：TiDB 客户端侧路由在缓存命中时零额外跳数，点查可达单次 RPC；但 RegionCache 未命中需访问 PD，且 Region 频繁分裂会抬高缓存失效率。OceanBase 代理侧多一跳 ODP，但强一致读一旦命中 LS Leader 即本地执行、省去 OBServer 间 RPC；LDC 路由进一步把跨机房 RPC 收敛到同 IDC。

吞吐：两系统都靠 Follower/弱一致读把读流量从 Leader 卸载。TiDB 的 Follower Read 按官方文档采 round-robin 轮询在 Follower 间分流，副本选择实为逐请求粒度(见 §2.2.5，已据 client-go 源码确认，非连接级/事务级)；OceanBase 弱一致读"均匀分布到 Leader 与 Follower"，follower-first 可卸载 Leader 但可能读到滞后数据。

可用性与故障恢复：Leader 切换后，TiDB 靠 NotLeader 错误 + RegionCache 失效在毫秒级重定位；OceanBase 靠反馈包 + 位置服务 `get_leader_with_retry_until_timeout`(默认 100ms 重试)收敛。OceanBase 路由表在稳态下相对稳定(官方文档 "Generally" 主因是 major compaction 或负载均衡引发的 Leader 切换)，但并非排他——拓扑变更、副本变化、错误码失效、定时拉取等也会触发刷新(详见 §2.3.4)，且 major compaction 期间(详见第 4 章 major freeze 抖动)尤其可能伴随路由刷新。

**表 2-2　性能、可靠性、运维影响**

| 维度 | TiDB(客户端侧) | OceanBase(代理侧) |
|---|---|---|
| 额外网络跳数 | 0(缓存命中) | +1(ODP) |
| 路由缓存抖动来源 | Region 分裂/合并/迁移频繁，epoch 常变 | 主要是 Leader 切换与 major compaction，频率低 |
| 失效驱动方式 | RegionError 即时驱动(客户端) | OBServer 反馈包驱动(代理强制刷新) |
| 控制面依赖强度 | RegionCache 未命中才访问 PD;v8.5 Active PD Follower 分担查询 | 多级缓存兜底，最后才异步查 OBServer；依赖 RS list/sys tenant |
| 跨机房优化 | zone 标签 + closest-replicas / Stale Read | LDC 路由 + 读写分离规则 |
| 运维诊断抓手 | SHOW TABLE REGIONS、Region Cache Miss 指标、KV Backoff Duration、coprocessor 日志字段 | EXPLAIN ROUTE、route_diagnosis_level、obproxy_diagnosis.log、SHOW PROXYINFO IDC |

性能代价主要来自"路由扇出"与"缓存刷新"。点查最怕热点集中在单 Region Leader 或单 LS Leader；范围查最怕跨太多 Region/Partition 后被最慢分片拖住；二级索引最怕先查索引再回表导致两段路由都跨多单位。运维上，TiDB 把路由智能放进 client-go，SQL 节点与存储节点版本需匹配，排障常从 TiDB 侧 coprocessor 日志、backoff 指标、PD Region 信息、TiKV Region 错误入手；OceanBase 把路由放进独立 ODP 组件，需单独运维 ODP 集群，但提供了 `EXPLAIN ROUTE` 这类专门的路由诊断命令(§2.9)，排障时要同时看 ODP 路由诊断、location cache、OBServer 执行计划与 Tenant/LS 状态。客户端直连 TiDB Server 减少了专用 SQL proxy 依赖，但客户端侧仍要做 TiDB Server 负载均衡与连接管理；ODP 模式把租户/分区/副本选择封装进代理，应用更简单，但代理自身的缓存、参数、诊断成为新的运维面。

## 2.8 反例与代价

TiDB 的代价：

- 点查若 key 单调递增，热点可能集中在少量 Region Leader；此时路由路径是正确的，性能问题来自负载集中而非 RegionCache 错误，把这类慢查询归因于 PD 或 RegionCache 是误诊。
- Region 分裂引发缓存雪崩式失效：大批量导入(如 `tidb-lightning`)会触发密集 split，导致 RegionCache 大面积 EpochNotMatch，backoff 重试放大延迟——社区有 ingest 因 EpochNotMatch 失败的真实 issue。范围查询跨很多 Region 时，即便 RegionCache 命中也无法消除网络扇出、cop task 调度、结果合并与尾延迟（推测）。
- 客户端侧路由把复杂度推给每个 TiDB Server:RegionCache 在每个节点各存一份，节点多时 PD 心跳与缓存一致性成本上升；client-go 与 TiKV/PD 版本需协同。
- Hibernate Region + Joint Consensus 的 no-leader 窗口：如 §2.2.4 所述，可能出现"PD 里 Leader 信息陈旧"的窗口，需靠换 peer 探测自愈，极端情况下多一次重试。
- Follower Read 并不是"随便读 follower"：强一致 Follower Read 需要 ReadIndex，Stale Read 是读历史版本，应用必须接受时间点语义。

OceanBase 的代价：

- ODP 是数据面必经一跳：虽可直连 OBServer，但生产几乎都过 ODP，引入额外延迟、额外故障域与额外运维对象；ODP 自身需高可用部署。客户端直连虽少一层代理，但应用很难可靠处理分区/副本/租户路由。
- 非分区键查询无法裁剪：`user_id = 88` 这类语句会被路由成全分区扫描，放大 RPC 与扫描量——这是 OceanBase 路由的"塌陷点"，根因在分区键设计(详见第 1 章)；跨分区范围查询与复杂表达式也可能无法精准前置路由，退化为 remote/distributed plan。
- 全局索引提升了按索引值定位的机会，但全局索引与主表不一定共址，仍可能引入远程 RPC 或分布式事务代价(本章只讨论读路由，不展开索引维护成本)。
- 路由表刷新条件较多，稳态仍稳但非"仅两类场景"：官方文档以 "Generally" 描述 major compaction / Leader 切换为主因，但拓扑变更、副本变化、错误码失效、定时拉取等也会触发刷新(详见 §2.3.4)；若反馈机制未及时触发，短时间内可能向旧 Leader 投递并触发一次重路由。代理缓存一致性、reroute 行为与配置版本都需纳入故障演练。

取舍总结：客户端侧路由(TiDB)省一跳、自治强，代价是缓存抖动与版本耦合；代理侧路由(OceanBase)解耦客户端、就近调度能力强、诊断显式，代价是多一跳与多一个运维组件。没有"更简单/更先进"之分，只有把路由智能放在客户端还是代理的工程取舍。

## 2.9 测试开发视角的验证点

可测试的功能场景：

- 三条示例 SQL 分别验证点查(单 Region/单 tablet)、范围查询(跨 Region/跨 tablet 切分)、非分区键查询(全 Region/全分区扫描)的路由行为，统计触达 Region/Partition 数、cop task 数与是否回表。
- 强一致读 vs Follower Read vs Stale Read(TiDB)/ 强一致读 vs 弱一致读 + LDC(OceanBase)的路由分叉；避免把 Stale 历史读当强一致读测试。
- 全局索引 vs 局部索引查询的路由差异(OceanBase)；单分区表、多分区表、全局索引表分别测试 ODP leader routing、partition pruning、global index routing。

可注入的失效模式：

- 触发 Region 分裂/合并(TiDB)，观察 EpochNotMatch 重试是否正确收敛；手动 transfer leader、split/merge 或滚动重启 TiKV，验证 `NotLeader`、`EpochNotMatch`、`StaleCommand` 场景下 SQL 是否在可接受的重试窗口内恢复。
- 主动 transfer leader / 杀 Leader 节点，验证 NotLeader(TiDB)、反馈刷新(OceanBase)路径与恢复耗时；OceanBase 还应覆盖 schema 变化、Tablet/LS 迁移或 location cache 过期时 ODP 的 reroute/dirty/refresh 行为及用户可见错误率与延迟尖刺。
- 阻断 PD / sys tenant，验证缓存命中时存量请求是否仍可服务、拓扑变更是否被正确阻塞。

关键压测指标：点查 QPS / P99、跨 Region 范围查询吞吐、Leader 切换后的路由恢复时间(routing recovery time)、缓存命中率下的延迟分布、重试次数、backoff 总耗时、每条 SQL 触达的 Region/Partition 数、单 Leader 热点、代理转发耗时、远程计划与 distributed plan 比例。

关键观测指标 / 诊断命令(已查证，不编造)：

- TiDB:`SHOW TABLE REGIONS`(查 Region 的 Leader/Range/Peers)、`information_schema.TIKV_REGION_PEERS / TIKV_REGION_STATUS`;coprocessor 日志中可出现 `region_id`、`store_addr`、`region_err`、`backoff_ms`、`backoff_types`、`kv_process_ms`、`kv_wait_ms`、`kv_read_ms` 等字段；Grafana 的 Region Cache Hit / Region Cache Hit Rate / Region Cache Miss Reason、KV Backoff Duration。其余具体 Prometheus metric 名随版本变化，需进一步查证。
- OceanBase:`EXPLAIN ROUTE <sql>`(ODP 路由诊断，经正常路由流程但不真正转发)、`route_diagnosis_level`(全局配置，默认 2，范围 0–4)、`obproxy_diagnosis.log`;ODP 文档公开的 `SHOW PROXYINFO IDC`、`SHOW PROXYSESSION VARIABLES` 与 OBServer `EXPLAIN` 可查看 partition pruning / 计划形态；诊断点序列含 `SQL_PARSE / ROUTE_INFO / LOCATION_CACHE_LOOKUP / PARTITION_ID_CALC_DONE / ROUTE_POLICY / CONGESTION_CONTROL / HANDLE_RESPONSE`。更细的 metric、内部视图与参数名未在引用资料中确认，具体名称需进一步查证。 〔文献[9]〕

## 2.10 容易误解点

1. "所有读都必须走 Leader"——错。这是本章最高频误区。TiDB 默认走 Leader，但提供 Follower Read(强一致，靠 ReadIndex)与 Stale Read(弱一致，读历史版本)；OceanBase 强一致读走 LS Leader，弱一致读默认"均匀分布到 Leader 与 Follower"、可 follower-first/follower-only。路由策略随一致性要求分叉，不能笼统化。

2. "RegionCache / PD 是事务协调器"——错。TiDB 的路由刷新解决的是"请求发到哪个 Region peer"，而 TSO、primary lock、commit/rollback 属于事务章节边界(详见第 6 章)；同理 OceanBase 的 ODP location cache 也只管定位，不管事务提交。把路由刷新与事务协调混写，是排障时常见的误诊源头。

3. "PD / sys tenant 是路由单点，挂了就全挂"——不准确。二者都是逻辑中心化但物理高可用(详见第 13 章)。更重要的是：路由有客户端/代理缓存兜底，控制面短时不可用时存量请求仍可命中缓存继续服务，只是拓扑变更无法刷新。必须按"逻辑中心化 / 性能瓶颈 / 高可用单点 / 元数据依赖 / 故障恢复依赖"五维度区分，不能简单写"单点"。

4. "OceanBase 路由就是 partition→机器"——不准确。实际是两段式 `tablet → Log Stream → leader/副本`(源码 `ObLocationService` 的两类接口已逐行确认)。Partition 是用户逻辑分区，Tablet 是数据承载对象，LS 是包含多个 Tablet 与 redo log 的复制/事务参与单位，三者不应混成一个概念；LS 而非 partition 才是共识与路由单元，这也是 4.x 与早期"每分区一个复制组"模型的关键差别(详见第 1 章)。内部 Tablet→LS 缓存刷新细节公开不足（不确定）。

## 2.11 本章结论

1. TiDB 是客户端侧路由：无状态 TiDB Server 内嵌 client-go 的 RegionCache，以 `LocateKey` 把 SQL 谓词映射为编码后的 KV key range 并定位 Region/Leader，PD 为权威元数据源；RegionCache 是性能缓存而非正确性边界，路由由 RegionError(EpochNotMatch/NotLeader/StaleCommand)即时驱动失效与 cop task 重建，正确性由存储侧 Leader/epoch 校验兜底。
2. OceanBase 是代理侧两段式路由：ODP 先经 cluster/tenant/表级路由，结合 partition pruning 与"本地→全局→OBServer 异步查询"多级缓存定位 `tablet → Log Stream → leader`，由 OBServer 反馈包驱动强制刷新；路由表"通常(Generally)"主因为 major compaction / 负载均衡引发的 Leader 切换，但非排他——拓扑/副本变化、错误码失效、定时拉取等也会触发刷新。
3. "能路由到 Leader"不等于"只能走 Leader"：两系统都提供强一致读(到 Leader)与弱一致/Follower/Stale 读(到副本/就近)，路由策略随一致性要求分叉。
4. 路由单元差异源于分片模型：TiDB 的 Region 既是 key range 又是 Raft Group(一体)，分裂频繁导致缓存抖动多但每次代价小；OceanBase 的 Partition / Tablet / LS 三层解耦，路由表更稳定但非分区键查询易塌陷为全分区扫描；两系统在点查、范围查、二级索引查上都会出现"一条 SQL 拆成多个底层请求"，区别在扇出单位是 Region 还是 Partition/计划。
5. 客户端侧 vs 代理侧是工程取舍而非优劣：前者省一跳、自治强、版本耦合；后者解耦客户端、就近调度与诊断显式、多一跳与多一组件（推测）。
6. 路由层对控制面是软依赖：PD / sys tenant 短时不可用时缓存可兜底存量请求，但拓扑变更无法感知；真正风险在性能瓶颈与故障恢复依赖，而非"单副本失效即全挂"（官方暗示）。
7. 测试开发不能只测 happy path，必须注入 Leader 切换、缓存过期、split/merge 或 location 刷新，并分别记录用户可见错误、内部重试与尾延迟；同时不宜把"强一致走 `get_leader`、弱一致走 `get`"当作 SQL 执行层的精确 1:1 判定——`get_leader/get` 是 Location Cache 模块为 SQL/事务/CLOG 等多模块提供的位置缓存接口，强/弱一致读的副本判定主要在 ODP 与位置服务层，且弱读可走 Leader。经核源码，OBServer SQL 执行层的真正落点是：优化器 `ObTableLocation::get_is_weak_read` 算出弱读并置 `loc_meta_.select_leader_`，执行期 `ObDASLocationRouter::get_tablet_loc` 按 `select_leader_` 分叉到 `nonblock_get_leader` 或 `nonblock_get_readable_replica`，而非 Location Cache 的 `get_leader/get`。

## 2.12 参考文献

[1] TiKV | Distributed SQL. 官方文档[EB/OL]. https://tikv.org/deep-dive/distributed-sql/dist-sql/.
 （支撑:§2.2 中 TiDB 按 Region 分组 key、切分 cop task 并发下发 TiKV 的正常路由路径描述。）
[2] TiDB Computing | TiDB Docs. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tidb-computing/.
 （支撑:§2.2 中 SQL 层构造 table/index key range、再到 TiKV 扫描过滤的路径说明，以及 Region 为 [StartKey, EndKey) 基本单元。）
[3] Follower Read | TiDB Docs. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/follower-read/.
 （支撑:§2.2.5/§2.7/§2.10 中 tidb_replica_read 各档值、ReadIndex 强一致、zone 标签本地副本、round-robin 轮询负载均衡的描述；原"连接级/事务级"粒度断言已移除，改以 client-go 源码确认的逐请求粒度为准。）
[4] Usage Scenarios of Stale Read | TiDB Docs. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/stale-read/.
 （支撑:§2.2.5 中 Stale Read 的 AS OF TIMESTAMP / tidb_read_staleness 与"可路由到任意副本"的描述。）
[5] TiDB 8.5.0 Release Notes. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/release-8.5.0/.
 （支撑:§2.2.1/§2.7 中 Active PD Follower 在 v8.5.0 GA、PD Follower 可处理 GetRegion/ScanRegions 查询的版本说明。）
[6] ODP routing | OceanBase Database Proxy. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-odp-doc-en-10000000001735865.
 （支撑:§2.3 中 ODP cluster/tenant/表级路由、RS list、多级缓存、从 sys tenant 取 partition/zone/tenant 信息、强一致到 LS Leader/弱一致分布、反馈刷新等描述。）
[7] ODP routing best practices | OceanBase. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-best-practices-10000000001797707.
 （支撑:§2.3.3/§2.4/§2.8 中 local/remote/distributed plan、全局索引路由(index value 作 partitioning key)与路由异常诊断边界。）
[8] Read/Write Splitting | OceanBase. 官方文档[EB/OL]. https://en.oceanbase.com/docs.
 （支撑:§2.3.4/§2.7/§2.8 中弱一致读 hint/变量、proxy_route_policy follower-first/follower-only、follower latency threshold 的描述。）
[9] 排查 ODP 路由问题(路由诊断)| OceanBase. 官方文档[EB/OL]. https://oceanbase.github.io/docs/user_manual/operation_and_maintenance/zh-CN/tool_emergency_handbook/odp_troubleshooting_guide/routing_diagnosis.
 （支撑:§2.9 中 EXPLAIN ROUTE、route_diagnosis_level、诊断点序列、obproxy_diagnosis.log、SHOW PROXYINFO IDC 的描述。）
[10] TiDB: A Raft-based HTAP Database. 论文，VLDB 2020[EB/OL]. https://www.vldb.org/pvldb/vol13/p3072-huang.pdf.
 （支撑:§2.2 中无状态 TiDB Server 经 PD 路由到 TiKV、Region 为分布式计算单元、Region = Raft Group(可分裂/合并)的整体架构描述。）
[11] pingcap/kvproto — proto/metapb.proto(master). 源码[EB/OL]. https://github.com/pingcap/kvproto/blob/master/proto/metapb.proto.
 （支撑:§2.2.4 中 RegionEpoch 字段定义：conf_ver(增删 peer 时自增)、version(分裂/合并时自增)，注释逐字取自源码原文。）
[12] tikv/client-go — internal/locate/region_cache.go(master). 源码[EB/OL]. https://github.com/tikv/client-go/blob/master/internal/locate/region_cache.go.
 （支撑:§2.2.2/§2.2.4 中 LocateKey、OnRegionEpochNotMatch 函数签名与注释、no-leader 换 peer 重试逻辑、InvalidReason 枚举的描述。）
[13] tikv/client-go — internal/locate/replica_selector.go / region_request.go(master). 源码[EB/OL]. https://github.com/tikv/client-go/blob/master/internal/locate/replica_selector.go.
 （支撑:§2.2.5/§2.7 中 Follower Read 副本选择为逐请求新建 selector(newReplicaSelector(...) 入参含 req *tikvrpc.Request，无连接/事务级跨请求状态)的描述。）
[14] pingcap/tidb — pkg/store/copr/ + pingcap/pd — pkg/core/region.go. 源码，tidb release-8.5 @ 67b4876bd57b / pd release-8.5 @ 6dce4a68e3e9[EB/OL]. https://github.com/pingcap/tidb/tree/release-8.5/pkg/store/copr.
 （支撑:§2.2 中 region_cache.go(RegionCache 嵌入封装、SplitKeyRangesByLocations/SplitRegionRanges)、coprocessor.go(handleCopResponse 检测 RegionError 后失效缓存并重建 task)的描述。）
[15] oceanbase/oceanbase — src/share/location_cache/ + ob_errno.h + ob_table_location.h + ob_ls.h. 源码，v4.2.5_CE @ e7c676806fda[EB/OL]. https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE/src/share/location_cache.
 （支撑:§2.3 中 ObLocationService 两类接口(LS 位置 get/get_leader/get_leader_with_retry_until_timeout、tablet→LS 映射)与返回码符号的描述。）
[16] oceanbase/oceanbase — src/sql/optimizer/ob_table_location.cpp + src/sql/das/ob_das_location_router.cpp. 源码，v4.2.5_CE @ e7c676806fda;v4.4.x 亦确认[EB/OL]. https://github.com/oceanbase/oceanbase/blob/v4.2.5_CE/src/sql/das/ob_das_location_router.cpp.
 （支撑:§2.3.4/§2.11(结论 7)中 OBServer SQL 执行层强/弱一致读副本判定链:ObTableLocation::get_is_weak_read(...) 据一致性级别(hint/session/协议弱读标志)算出 is_weak_read 并设置 select_leader_，执行期再据此分叉的描述。）
[17] oceanbase/obproxy — src/obproxy/proxy/route/. 源码，独立仓库[EB/OL]. https://github.com/oceanbase/obproxy/tree/master/src/obproxy/proxy/route.
 （支撑:§2.3.2/§2.6/§2.9 中 OBProxy/ODP route、partition cache、table cache、ldc route、route diagnosis、obproxy_expr_calculator、obproxy_part_mgr 等模块的描述。）
[18] Introduction to OBProxy: Modules and Features. 官方博客[EB/OL]. https://en.oceanbase.com/blog/2615101184.
 （支撑:§2.3.2 中 OBProxy 的 SQL Parser 提取表名/分区键、数据路由模块转发到最高效 OBServer 的描述。）
[19] 7 Key Technologies to Ensure High Availability in OceanBase Database. 官方博客[EB/OL]. https://en.oceanbase.com/blog/2615184384.
 （支撑:§2.3.4/§2.7 中 Leader/Follower 副本读、强/弱一致读路由、定时拉取/拓扑与副本变化触发刷新、LDC 与读写分离规则的描述。）
## 2.13 信息可信度自评

本章路由"正常路径"骨架来自官方文档(TiKV Deep Dive、TiDB Computing、ODP routing、Follower/Stale Read、读写分离)与官方博客，可信度高；TiDB VLDB 2020 论文用于整体架构印证及 Region = Raft Group / split-merge 论点，版本较早，具体 v8.5 行为以文档与源码为准。

源码级事实分两类。已确认到文件/符号级的包括：TiDB `pkg/store/copr/`(EpochNotMatch 失效缓存 + 重建 task、`SplitKeyRangesByLocations`)、PD `pkg/core/region.go`(`RegionInfo` 结构、`RegionFromHeartbeat`)、kvproto `proto/metapb.proto`(`RegionEpoch` 字段定义，逐字确认)、OceanBase observer `src/share/location_cache/ob_location_service.h`(两段式路由的接口签名与 100ms 默认间隔)、`src/share/ob_errno.h`(返回码符号经逐字核验，已纠正一个早期错名符号，见下文)、`src/sql/optimizer/ob_table_location.h`(`calculate_tablet_ids` 分区裁剪)、`src/storage/ls/ob_ls.h`(`get_ls_id`/`get_tablet`)。仅确认到模块/文件名级的包括：client-go RegionCache(外部依赖，函数原文取自 master 分支，版本号由 TiDB go.mod 锁定)与 obproxy 客户端侧路由(`src/obproxy/proxy/route/` 文件名已核验，具体函数未展开)。

工程推测与不确定项已显式标注：客户端侧 vs 代理侧的取舍权衡(结论 5)为（推测）；控制面软依赖(结论 6)为（官方暗示）；强/弱一致读与 `get_leader`/`get` 接口的对应(结论 7)经反查证确认不是 Location Cache `get_leader/get` 的 1:1 映射，且已据 OBServer SQL 层源码定位到真正的判定链(`ObTableLocation::get_is_weak_read` → `loc_meta_.select_leader_` → `ObDASLocationRouter::get_tablet_loc` 分叉 `nonblock_get_leader`/`nonblock_get_readable_replica`)，因而已从推测升级为源码确认。Cloud 形态下的内部多层代理、Tablet→LS 缓存刷新细节仍为（不确定）。obproxy 客户端侧路由的函数级实现因属独立仓库、未 clone 进 groundtruth，仍仅确认到模块/文件名级；client-go RegionCache 各 Prometheus metric 名(外部依赖、随版本变化)亦未在源码或官方文档中逐一锁定，保留为待查。

反查证发现：

1. OceanBase 返回码符号纠错：早期资料写的 `OB_LS_LEADER_NOT_EXIST` 在 `src/share/ob_errno.h` 中并不存在(逐字检索无此 token)，实际存在的相关返回码为 `OB_LS_LOCATION_NOT_EXIST = -4721`、`OB_LS_LOCATION_LEADER_NOT_EXIST = -4722`、`OB_LOCATION_LEADER_NOT_EXIST = -4654`、`OB_MAPPING_BETWEEN_TABLET_AND_LS_NOT_EXIST = -4723`，现已据源码更正。
2. epoch 定义来源改挂：`epoch = conf_ver + version` 的字段定义来自 kvproto `proto/metapb.proto`,Region = Raft Group / split-merge 论点改由 TiDB VLDB 2020 论文承载。
3. Follower Read 负载均衡粒度核实：Follower Read 官方文档支撑各档值、ReadIndex 强一致、zone 本地副本与 round-robin 轮询，但不含"coprocessor 连接级 / 点查事务级"粒度；经核 client-go replica selector 源码，副本选择为逐请求新建 selector(无连接/事务级跨请求状态)，据此把原"连接级/事务级"断言更正为"逐请求粒度"，并记录一处版本差异：文档称 round-robin，而 client-go master 实为打分式(`ReplicaSelectMixedStrategy`)+ 同分随机平局。
4. ODP 路由表变化条件去排他化：官方文档原文为 "Generally, the routing table changes only when…",`Generally` 是 hedge；官方多份文档/博客枚举了定时拉取(系统租户约 15s)、错误码失效、拓扑变更、副本变化、机器故障/副本切换等额外触发，本稿已把"仅…才变化"改述为"通常成立的主因之一，非穷尽条件"。

版本核查备注：本章引用版本严格匹配统一基线——TiDB 8.5.x(release-8.5)、PD/TiKV release-8.5、OceanBase TP 4.2.5_CE(位置服务在 4.3.5 / 4.4.x 同名存在)、OBProxy/ODP 4.2.x/4.3.x。client-go 依赖版本 `v2.0.8-0.20260615130046-...` 为 go.mod 锁定值，函数原文取自其 master 分支(client-go 不随主版本号严格对齐，属正常)。obproxy 源码为独立仓库，ODP global index routing 等能力以 4.3.x 公开文档为准，不回填到 4.2.x。未发现与基线冲突的版本号。

---


# 第 3 章 共识

> 版本基准：TiDB 8.5.x LTS（2024-12-19 GA），对照 7.5.x;TiKV/PD 源码以 `release-8.5` @ `1f8a140b6d46` 为准。OceanBase TP LTS v4.2.5_CE（2025-02），源码以 `v4.2.5_CE` @ `e7c676806fda` 为准，对照 4.3.5(AP)/4.4.x（融合）。本章涉及的共识算法位于 TiKV 的 `components/raftstore/` + 外部 `tikv/raft-rs` + `tikv/raft-engine`，以及 OceanBase 的 `src/logservice/palf/`。

---

## 3.1 本章核心问题

本章核心问题不是「Raft 与 Paxos 谁更好」，而是两家如何把共识协议落进各自的数据库工程。分布式数据库要在多副本之间保证「同一份数据，任何时刻只有一个被多数派承认的真相」，这件事由共识协议承担。当数据被拆成许多可迁移、可复制的单位后，共识层决定了三件最底层的事：一条写入要落到多少副本才算成功（提交语义）、节点宕机或网络分区后谁能合法地继续写（选主与脑裂防护）、以及故障后如何把已提交但未对齐的日志补齐（日志恢复）。它直接决定数据库的 RPO/RTO、写延迟的下限，以及横向扩展时副本管理的复杂度。

本章只讨论共识本身：Raft / Multi-Raft 与 Multi-Paxos / PALF 的机制、它们的工程实现偏离，以及两条技术路线在故障恢复与可测试性上的代价。分片如何切出共识单元属于第 1 章（Region / Log Stream 的边界详见第 1 章）；共识日志与存储引擎 WAL/LSM 的关系详见第 4 章；共识之上的分布式事务（2PC、TSO/GTS）详见第 10 章。本章不重复这些内容，只在交界处标注。沿用第 1 章的两个边界：TiDB 的数据移动和复制边界是 Region,多个 Peer 组成一个 Raft Group;OceanBase 4.x 把 Partition、Tablet、Log Stream（LS）分开，其中 LS 承载多个 Tablet 的有序 redo log,并作为 Paxos 复制和事务提交参与单位。

一个必须先立的原则：本章不评判 Raft 与 Paxos「谁更先进」。Raft 论文自述其产生与 Multi-Paxos 等价的结果，并强调结构不同与可理解性；*Paxos Made Live* 则说明把 Paxos 从理论带到生产数据库远不只是翻译伪代码。两者在安全性上等价，差异在工程取舍——Raft 用「强 Leader + 强约束」换可理解性与实现一致性，Multi-Paxos 用「选举与日志解耦 + 灵活性」换与数据库日志服务的深度协同。

还需先点明为什么两家选了不同路线。这并非偶然，而是历史路径依赖、系统目标、分片模型、事务模型、日志组织方式五者共同作用的结果：TiDB 起步于「KV 分层 + 复用成熟生态」，直接采用 etcd-raft 系的 `raft-rs`，把共识做成可插拔的算法库，Region（key range）天然适配 Raft「一个组管一段连续 key」的离散 index 日志，再由 PD 调度大量较小的 Region;OceanBase 起步于「自研一体化分布式关系库 + 蚂蚁内部超大规模 OLTP」，其存储引擎（LSM）、事务引擎（2PC + MVCC）、物理备库、备份恢复都需要一套能被数据库深度定制的日志服务，于是把事务日志、LSN/SCN、成员关系、Leader 恢复都合进 PALF（Paxos-backed Append-only Log File system）。理解了这一层，后续所有「偏离教科书」的工程增强就不再是孤立技巧，而是各自目标的自然延伸。

--- 〔文献[2-4]〕

## 3.2 TiDB 的实现：Multi-Raft over raft-rs

![[f3_1.svg]]
**图 3-1　TiKV 与 OceanBase 共识栈并列分层**

### 组件分层：谁实现了 Raft

一个反直觉但重要的事实：TiKV 自身并不实现 Raft 共识算法本体。在 `tikv/tikv` `release-8.5`(commit `1f8a140b6d46`)的 `Cargo.toml` 中，Raft 算法以外部 crate 依赖引入——`raft = { version = "0.7.0", default-features = false, features = ["protobuf-codec"] }`，且 `[patch]` 段把 `raft` / `raft-proto` 指向 `git = "https://github.com/tikv/raft-rs", branch = "master"`(经独立抓取 GitHub 上的 `Cargo.toml` 反查证一致)。也就是说，选举、任期、日志复制安全性这些算法核心位于 `tikv/raft-rs`（基于 etcd-raft 移植），而 `components/raftstore/` 是把单个 `raft-rs` 实例封装成 Multi-Raft、并叠加大量工程增强的调度层。日志的持久化（WAL）则委托给另一个独立仓库 `tikv/raft-engine`(`release-8.5` 中固定到 `branch = "tikv-8.5"`)。

因此 TiDB 的共识栈是三层分离的：

- `tikv/raft-rs`（算法）：单 Raft Group 的选举/复制/安全性；
- `components/raftstore/`（Multi-Raft 调度 + 工程增强）：`peer.rs`、`fsm/apply.rs`、`peer_storage.rs`、`hibernate_state.rs`、`read_queue.rs`、`util.rs` 等；
- `tikv/raft-engine`（日志存储）：多 Raft 日志的持久化引擎。

`peer.rs` 在创建 Peer 时构造 raft-rs 的 `raft::Config`，包含 election/heartbeat tick、message size、inflight message、`check_quorum`、`pre_vote`、`max_committed_size_per_ready` 等字段，并通过 `RawNode` 驱动核心 Raft 状态机。这印证了分工：raft-rs 提供 core consensus module,raftstore 提供存储、网络、snapshot、调度消息、apply、metrics 和恢复编排。

> 查证边界：`raft-rs` / `raft-engine` 的内部实现不在本章 checkout 内，仅确认到「TiKV 通过 Cargo 依赖固定版本/分支」级；涉及其内部函数时按此标注。另据 TiKV v8.5.0 `a2c58c94f89cbb410e66d8f85c236308d6fc64f0` 的 `Cargo.lock` 亦确认 `raft-engine` 锁定 `de1ec937529e3a88e093db0cf0d403522565fe64`、`raft-rs` 锁定 `branch=master` 的 `a76fb6ef`，与 release-8.5 基线一致。

### 共识单位：Region = Raft Group

TiKV 把整个 key 空间按 range 切成 Region,每个 Region 是一个独立的 Raft Group（官方 deep-dive 原文："Raft group is divided into multiple Raft groups in terms of partitions, namely, Region"）。一台 TiKV 节点上同时运行成千上万个 Raft Group,这就是 "Multi-Raft"。需要特别注意：Multi-Raft 在 TiKV 语境下不是「Raft 协议的 multi 版本」，而是在一个集群、一个节点上同时管理大量独立的 Raft consensus group。Region 三副本，每副本是一个 Peer,其中一个是 Leader;Leader Peer 提供读写服务，多数派写入后写操作成功。Region 的拆分/合并触发成员与 range 变化（详见第 1 章）。

### 正常写路径（数据路径）

一次写入的共识链路：

1. TiDB Server 通过 Region cache 找到 key 所属 Region 的 Leader,把 KV 请求发到 TiKV 的 Leader Peer（路由详见第 2 章）；
2. Leader 经 `raft-rs` 把请求 propose 成一条 Raft log entry,通过 `AppendEntries` 复制给 Followers;
3. Followers 持久化日志后回 ack;多数派（2/3）确认即 commit,Leader 推进 commit index;
4. commit 后进入 apply 阶段，由 `fsm/apply.rs` 中的 `ApplyDelegate` 把命令应用到底层 RocksDB（状态机），最后向客户端返回结果。

这里 commit 与 apply 必须分开：commit 表示 Raft 日志在多数派上达成一致，apply 表示状态机真正执行。TiKV 在这条路径上做了多项工程并行化。raftstore 以批（batch）方式驱动多个 Raft Group,每个 ready 用一个 RocksDB `WriteBatch` 一次性追加日志，`cmd_batch` 默认 `true`(`config.rs` L353/L614)。apply 委托给独立线程池(`apply_batch_system`)，并允许 apply 未持久化的日志(`max_apply_unpersisted_log_limit` 默认 1024,`config.rs` L94/L544)，从而把「propose→persist→apply」的串行路径拆成可并行的流水线。此外，对 Leader,消息可在日志落盘前先发出（Raft 论文提到的 pipeline 优化），inflight 窗口由 `raft_max_inflight_msgs = 256` 控制(`config.rs` 默认透传给 raft-rs)。这些 batch/pipeline/async apply 优化的目的是让 append、持久化、发送、apply 更并行，但都不能绕过 Raft 的日志匹配和多数派提交条件。

### 控制路径：选主、PreVote、Lease Read

- 选主与任期：`raft-rs` 沿用 Raft 的 term + RequestVote。默认 tick=1s、`raft_election_timeout_ticks = 10`、`raft_heartbeat_ticks = 2`(`config.rs` Default L526-537)，即心跳 ~2s、选举超时 ~10s 量级。选举与日志复制共用同一套 RPC/term,这是 Raft 与 PALF 最大的结构差异。
- PreVote（默认开启）：`config.rs` 中 `pub prevote: bool`(L54),`Default = true`（L527），构造 raft-rs config 时 `pre_vote = self.prevote`(L690),`peer.rs` 中亦可见 `pre_vote: cfg.prevote`。PreVote 让节点在真正发起选举前先「预投票」探明能否赢得多数，避免被网络分区隔离的副本恢复后用更高 term 打断现有 Leader。这是 etcd-raft / raft-rs 的工程增强，Raft 原论文未将其作为核心算法提出。配套的 CheckQuorum 同样属于工程增强层。
- Lease Read / ReadIndex（线性一致读优化）：TiKV 官方 Lease Read 博客把强一致读分为 Raft Log Read、ReadIndex Read、Lease Read 三档——Raft Log Read 最保守但每次读都进入 Raft log,ReadIndex 用 heartbeat 确认 Leader 身份，Lease Read 依赖 Leader Lease 避免每次读都走网络确认。官方博客明确：选举超时 10s,用固定 9s 作为 lease 上限续约，以保证「旧 Leader 的 lease 在新 Leader 选出前必然过期」，防止脑裂期读到陈旧值；且 lease 通过写操作（写本身走 Raft log）而非心跳续约，并使用 monotonic raw clock 规避 NTP 抖动。源码侧 `raft_store_max_leader_lease` 默认 `ReadableDuration::secs(9)`(`config.rs` L255/L593),ReadIndex 路径则实现于 `read_queue.rs`。需强调：Lease Read 是基于时钟和 Leader 有效性的工程优化，必须和 election timeout、clock drift、lease 更新策略配合，不能简化为「Leader 本地读天然强一致」。

PD 的角色边界也属于控制路径。PD 不替 Region 做数据复制，但它保存 Region 元数据、接收 Region heartbeat / store heartbeat,并据副本数、容量、热点和故障状态发起调度（leader transfer、add/remove Peer、split/merge、scatter）。真正的副本日志一致性仍回到每个 Region 的 Raft Group 内完成。因此「PD 是单点」并不准确：它是逻辑中心化的元数据与调度依赖，通常以多成员方式部署；PD 短暂不可用会影响新调度、路由刷新和 TSO 等控制面能力，但不会让已持有有效 Leader 和路由的 Region 突然绕过 Raft safety（此判断属工程推断）（推测）（事务时间戳详见第 10 章）。

### 成员变更：Joint Consensus

Region 增删副本走 Joint Consensus（单步多成员变更的两阶段配置）。`util.rs` 中 `enum ConfChangeKind { Simple, EnterJoint, LeaveJoint }`(L912-919),`confchange_kind(change_num)` 按变更数量判定（0→LeaveJoint、1→Simple、>1→EnterJoint,L922-927），底层基于 raft-rs 的 `eraftpb::ConfChangeV2`(`enter_joint`/`leave_joint`,L1015-1018)，源码中 `in_joint_state` 可确认 IncomingVoter / DemotingVoter 角色存在。Joint Consensus 是 Raft 论文 §6 的扩展协议，由 raft-rs 落地实现。当 PD 调度新增副本时，新 Peer 需先追赶日志或接收 snapshot,避免把空副本直接放进投票多数派；Region split/merge 时 Region epoch 变化，收到过期 epoch 的请求需刷新路由或拒绝陈旧消息。

### 故障路径概览

Leader 宕机 → Followers 选举超时 → PreVote → 正式 RequestVote → 日志最新的副本当选（Raft 安全性：Leader Completeness,新 Leader 必然含有全部已提交日志，不需要回补历史）→ 新 Leader 提交一条空日志确立 commit 点后恢复服务；落后副本通过 log catch-up 或 snapshot 补齐。详细异常路径见 §3.6。

### 涉及的源码模块、文档与论文

- 源码：`tikv/tikv` `release-8.5` @ `1f8a140b6d46` —— `components/raftstore/src/store/`(`config.rs`、`peer.rs`、`fsm/apply.rs`、`util.rs`、`read_queue.rs`、`hibernate_state.rs`)+ `Cargo.toml`；外部 `tikv/raft-rs`(`raft 0.7.0` + patch 到 `master`，仅确认到依赖声明级)、`tikv/raft-engine`(`branch = tikv-8.5`，日志存储，仅确认到模块级)。
- 官方文档/博客：TiKV deep-dive（Multi-Raft、Raft）、TiKV lease-read 博客、TiDB Storage / TiKV Overview。
- 论文：*TiDB: A Raft-based HTAP Database*（VLDB 2020）、Raft 原论文。

--- 〔文献[5-6,9,13-14]〕

## 3.3 OceanBase 的实现：Multi-Paxos over PALF

### PALF 是什么

PALF = Paxos-backed Append-only Log File system（论文正式标题为 *PALF: Replicated Write-Ahead Logging for Distributed Databases*,VLDB 2024,PVLDB 17(12):3745-3758）。它是 OceanBase V4.0.0 起的复制式 WAL 系统，代码在仓内 `src/logservice/palf/`(`v4.2.5_CE` @ `e7c676806fda`，论文 artifact 指向 `oceanbase/oceanbase/.../src/logservice/palf`)。官方多副本日志同步文档写明：OceanBase V4.0.0 将日志服务抽象为 PALF,基于 Paxos 在多副本间同步日志，并用 LSN 在 Log Stream 中唯一定位日志；已明确提交的日志在多副本上完全一致。

与 TiDB 把共识做成可插拔算法库相对，OceanBase 走的是另一条路线。PALF 把共识协议集成进数据库的 WAL 模型：对上提供 append-only logging 接口，使数据库像操作本地文件一样与 PALF 交互，同时把数据库相关能力抽象成 primitives,以保持日志系统和上层之间的边界。其复制路径采用「复制式 WAL / Paxos-backed append-only log + 事务状态推进」：redo 日志经 Paxos 在多数派持久化后，事务状态再被推进为可提交——这与经典 Replicated State Machine（RSM）模型「日志先复制后 apply」的细节编排不同，使事务大小不再受复制缓存限制。论文原文："PALF implements Paxos with a strong leader approach, it keeps the log replication of Raft for simplicity. Different from Raft, PALF decouples leader election from the consensus protocol"——这一句是理解 OceanBase 共识路线的关键：它借用了 Raft 式强 Leader 日志复制的简洁，但把「选谁当 Leader」从共识协议中剥离出去。

### 共识单位：Log Stream(LS)

OceanBase 4.x 的 Paxos 复制单位是 Log Stream（LS），而非分区。官方架构文档说明：LS 是自动创建和管理的实体，包含若干 Tablet 与 ordered redo log,使用 Paxos 在副本间同步日志保证一致；从事务视角，单 LS 修改可用一阶段提交，跨多个 LS 时使用两阶段原子提交协议。这是相对 1.0-3.0「每分区一个 Paxos 组」的重大重构：早期成千上万个 Paxos 组在选主/心跳/日志同步上消耗大量 CPU,且巨型事务会跨上万个 2PC 参与者。4.x 把复制组数量限制到「集群服务器数」量级，单组吞吐要求更高，但分布式事务参与者大幅减少（LS 模型详见第 1 章）。每个 LS 一 Leader 多 Follower,Leader 用 LSN（单调递增的日志序列号，字节连续偏移）标识每条日志，经 Paxos 按 LSN 顺序复制，多数派持久化即 commit。

### 正常写路径（数据路径）

对应论文的复制 WAL 模型与 `src/logservice/palf/` 源码，上层事务或存储模块向 `ObLogHandler` append 日志，LogHandler 调用 PALF:

1. 事务引擎在 Leader 上推进写入，生成 redo 日志；
2. redo 日志经 `PalfHandleImpl::submit_log(...)`(`palf_handle_impl.h` L319/L907)append 到 PALF,Leader 为日志分配 LSN/SCN 相关位置并经 `LogSlidingWindow::submit_log(...)`(`log_sliding_window.h` L240)追加（对应 Multi-Paxos 的 Accept 阶段）；
3. Followers 持久化后回 ack,Leader 经 `LogSlidingWindow::ack_log(src_server, end_lsn)`（L260）收集；
4. `get_majority_match_lsn(...)`（L236）算出多数派匹配位点，`try_advance_committed_end_lsn(end_lsn)`（L297）推进 `committed_end_lsn`（提交）；
5. 提交后通知事务引擎，事务状态被推进为可见；Followers 经 replayservice 回放(`src/logservice/` 下 `applyservice/`、`replayservice/`、`ob_log_service`)。

关键差异：PALF 日志按字节连续的 LSN（滑动窗口）组织，而非 Raft 的离散 entry index;稳态下只跑 Paxos Phase-2(Accept),Phase-1 被复用（见下文故障路径）。PALF 自带块式日志存储与 IO worker(`log_engine.{h,cpp}`、`log_block_mgr`、`log_io_worker`)，即「Paxos 日志服务」自管磁盘格式与刷盘，并提供 `seek`、`locate_by_scn_coarsely`、`advance_base_lsn`、`get_palf_base_info` 等文件式与恢复式接口，与数据库日志服务深度耦合（刷盘语义仅确认到模块/类级）。因此 OceanBase 写路径的性能不能只按一次 Paxos RTT 估算，还要考虑日志分组、磁盘 I/O、SCN 语义、apply service 与 LS 级事务提交之间的耦合。性能侧论文提到 pipeline 复制、自适应 group replication（group commit）、lock-free 写路径。这里的「slot」既不是 SQL 分片也不是 Tablet ID,而是日志序列位置上的复制决议，可查证名称为 LSN、`proposal_id`、`LogGroupEntry`、`SlidingWindow`、`committed_end_lsn` 等。

### 控制路径：独立的 lease 选举

PALF 把「谁是 Leader」交给一个专门的 lease 选举模块 `src/logservice/palf/election/`(含 `election_proposer`、`election_acceptor`、`election_impl`)。`election_proposer.h` 有 `int64_t lease`（L83）、`current_ts < lease` 判活（L86）、`leader_revoke_if_lease_expired_`（L110）、`leader_takeover_if_lease_valid_`（L111）、`register_renew_lease_task_`（L115），中文注释明确「Leader 的 lease 在续约过程中从未中断过，即确认不可能有其他 leader 产生」（L126）。官方文档称默认 lease ~10 秒，临近过期前续约/重选。选举支持优先级（priority），可让靠近上层应用的副本优先当选（论文称这正是把 Leader 位置与共识协议解耦的目的）。这是与 Raft「选举与日志复制同一套 term/RPC」最大的工程结构差异。

LS 侧的角色与状态由 `ob_log_handler.h` 封装：`switch_role`、`get_role`、`change_access_mode`、`get_paxos_member_list`、`get_stable_membership`、`get_election_leader`、`get_palf_base_info`、`is_in_sync` 等。需要注意，LS 调度、Tablet transfer 与事务提交可能同时发生，RootService、负载均衡、location cache、leader coordinator、LogHandler 与 PALF 分担不同职责——前者决定 LS 副本放在哪里、Tablet 如何归属、Leader 位置如何被发现，后者确保某个 LS 内的日志序列在多数派上不分叉。测试时若只盯着 Paxos message 而忽略 location cache 或 LS 状态变化，就容易漏掉「路由已更新但旧 Leader 仍有本地状态」的异常组合（RootService 的中心化边界见 §3.7）。

### 故障路径核心：LogReconfirm（Phase-1 + 日志恢复）

![[f3_2.svg]]
**图 3-2　PALF Reconfirm / Leader 切换状态机**

因为选举独立于日志协议，PALF 必须在新 Leader 上任时单独跑一轮 log reconfirmation 来保证安全性。`LogStateMgr`(`log_state_mgr.h` L42)管理副本状态机，新 Leader 上任先进入 LEADER_RECONFIRM(`is_leader_reconfirm()` L90)，完成日志恢复后才转 LEADER_ACTIVE(`is_leader_active()` L87)。reconfirm 逻辑在 `LogReconfirm`(`log_reconfirm.h` L39)，其状态机包括 `FETCH_MAX_LOG_LSN`、`RECONFIRM_MODE_META`、`RECONFIRM_FETCH_LOG`、`RECONFIRMING`、`START_WORKING` 等状态：`submit_prepare_log_()`（L69）发 Prepare,`handle_prepare_response(server, src_proposal_id, accept_proposal_id, ...)`（L58-60）收集 Promise,初始化时按 `replica_num / 2 + 1` 计算多数派，成员变量 `new_proposal_id_`（L122）、`prepare_log_ack_list_`（L153）、`majority_max_accept_pid_`（L160）、`majority_max_lsn_` 等体现「取多数派 max proposal_id/LSN 恢复已接受日志」；源码注释还提到 ghost logs 风险，说明新 Leader 需经 reconfirm、日志拉取与 START_WORKING 确认历史日志边界后才进入正常写入。

用 Paxos 语言说：`proposal_id` = Paxos ballot;reconfirm = 新 Leader 发 Prepare → 多数派回 Promise（各带其 accept 过的最大 proposal_id/LSN）→ 取多数派最大值，补齐/截断日志。论文表述更直接："the candidate performs an instance of Basic Paxos to learn missing logs from the replica that accepted the most logs in a majority"。这与 Raft「新 Leader 仅靠投票时的日志新旧约束、不回补历史」机制不同但目的相同——都保证已被多数派接受的日志不丢。这套流程与教科书 Multi-Paxos 的 Prepare/Promise、Accept/Accepted、Ballot/Proposal ID 有对应关系，但不能逐字等同，因为 PALF 还要处理 access mode、LSN/SCN、成员变更、learner、rebuild 等数据库日志服务问题。

PALF 还引入了 Raft 没有的 pending follower 角色：旧 Leader 失去 Leader 身份时不直接转为 follower,而是先进入 pending follower 等待新 Leader 的日志，以确定自己手里那些 in-flight 日志到底是否已提交（论文称这是为了给事务引擎返回明确的复制结果——commit 还是 rollback）。这是 PALF「让日志服务像本地文件一样返回明确写结果」设计的一部分(状态迁移见 `log_state_mgr.h` L139-143 的 `follower_*_to_reconfirm_` / `reconfirm_to_*` 私有方法)。

### 成员变更：config_version + degrade/upgrade

PALF 成员变更在 `LogConfigMgr`(`log_config_mgr.h`，实现约 160KB):`is_add_member_list`（L134）、`is_remove_member_list`（L139）、`is_upgrade_or_degrade`（L162）、`is_may_change_replica_num`（L180，涵盖 add/remove/CHANGE_REPLICA_NUM/FORCE_SINGLE_MEMBER 等），以 `LogConfigVersion config_version_`（L298）做版本化，配合 member list、learner list、stable membership、access mode 完成 LS 迁移/重建。相比 Raft Joint Consensus,PALF 多了「副本降级/升级（degrade/upgrade）」语义，用于少数派容灾——配合 arbitration service（仲裁服务），可在两 IDC 场景下把临时不可达副本降级，保持多数派可写。

### 涉及的源码模块、文档与论文

- 源码：`oceanbase/oceanbase` `v4.2.5_CE` @ `e7c676806fda` —— `src/logservice/palf/`(`palf_handle`、`palf_env`、`log_state_mgr`、`log_reconfirm`、`log_sliding_window`、`log_config_mgr`、`log_mode_mgr`、`log_engine`、`log_rpc`、`log_learner`、`election/`)；上层 `src/logservice/{applyservice,replayservice}`、`ob_log_service`、`ob_log_handler`。三版本（4.2.5/4.3.5/4.4.x）均含 `palf/` 目录（文件数 132/132/134），具体版本演进点需进一步逐版本查证；另据 v4.4.2_CE `e859d1b9` 亦确认上述文件存在，与基线一致。
- 官方文档/博客：OceanBase Multi-replica log synchronization、Cluster architecture（Log Stream）、HA 技术博客。
- 论文：*PALF*（VLDB 2024）、*Paxos Made Simple*、*Paxos Made Live*。

--- 〔文献[1,7-8,10]〕

## 3.4 核心差异对比

把前两节的实现拆解并到一处，按算法位置、共识单位、读写路径、选举与日志关系、成员变更、工程增强等维度逐项对照，意图是看清两条路线的取舍落点而非排座次。

**表 3-1　共识:TiDB 与 OceanBase 核心差异对比**

| 维度 | Raft / TiDB(TiKV `release-8.5`) | Multi-Paxos / OceanBase(PALF `v4.2.5_CE`) | 影响 |
|---|---|---|---|
| 共识算法本体位置 | 外部 `tikv/raft-rs` crate(`raft 0.7.0` + patch master),raftstore 仅封装 | 仓内 `src/logservice/palf/` 自研 | TiKV 复用成熟算法库，边界清晰；OB 自研可与 DB 深度协同 |
| 共识单位 | Region(= Raft Group,key range) | Log Stream（聚合多 Tablet 与 ordered redo log） | OB 复制组数 ≈ 服务器数，2PC 参与者更少（见第 1/10 章） |
| 正常写路径 | Leader Peer propose Raft log,`AppendEntries` 复制，多数派 commit 后 apply | LS Leader 经 LogHandler/PALF append,Paxos 复制，多数派 committed 后上层回放 | TiDB 重在大量 Region 并发调度；OB 重在 LS 日志序列与事务日志服务融合 |
| 选举与日志复制关系 | 同一套 raft-rs(term + RequestVote + AppendEntries) | 分离：`election/`（lease 选举）+ `log_reconfirm`/`log_sliding_window`（Paxos 日志） | OB 可按 priority 控制 Leader 位置；但需额外 reconfirm 保证安全 |
| Leader 合法性 | term + 投票 + 日志新旧 + CheckQuorum + PreVote + lease | ballot(proposal_id)+ election + reconfirm + access mode + lease(≈10s,priority) | 机制不同，安全性等价 |
| 新 Leader 安全性/恢复 | 日志最新者当选，不回补历史（Leader Completeness）；落后副本 catch-up / snapshot | `LogReconfirm`:Prepare→多数派 max proposal_id/LSN→恢复已接受日志，必要时 fetch log / START_WORKING | Raft 约束前移到选举；PALF 把恢复后移到 reconfirm |
| 稳态提交 | `AppendEntries` + match index 多数派 | `submit_log`/`ack_log` + `get_majority_match_lsn`/`try_advance_committed_end_lsn` | 均为一轮多数派 RTT |
| 日志 slot 形态 | 离散 entry index | 连续 LSN（字节偏移）滑动窗口 | LSN 利于按字节定位、物理备库同步 |
| 复制模型 | RSM（日志先复制后 apply） | 复制式 WAL / Paxos-backed append-only log + 事务状态推进 | OB 事务大小不受复制缓存限；日志服务与 DB 深度耦合 |
| 成员变更 | Joint Consensus(ConfChangeV2)+ learner / snapshot | `LogConfigMgr` + config_version + degrade/upgrade + member/learner list | OB 多副本降级语义，利于少数派/两 IDC 容灾 |
| 工程增强（已逐项查证） | PreVote（默认 true）、CheckQuorum、Lease Read（9s）、Hibernate Region（默认 true）、Joint Consensus、cmd_batch（默认 true）、async apply、inflight 256 | lease 选举与日志分离、priority、pending follower、config_version + degrade、自带 LogEngine 块存储、pipeline/自适应 group commit | 两者都远偏离教科书，但增强方向不同 |

工程风格小结：Raft/TiDB 约束强、状态模型清晰，工程增强多且大多落在 raftstore 调度层；Multi-Paxos/PALF 灵活、复杂，与数据库日志服务（WAL、LSN/SCN、物理备库、备份恢复）深度耦合。复杂度本身不能简单写成优劣，它取决于各自的分片模型与事务模型。

---

## 3.5 正常路径图

图 3-3 为 TiDB 一次正常写入的 Multi-Raft 共识路径（以单个 Region 为例）。

![[f3_3.svg]]

**图 3-3　共识正常读写/调度路径**

OceanBase 一次正常写入的 PALF 路径（复制式 WAL:redo 日志经 Paxos 多数派持久化后推进事务状态）。

![[f3_4.svg]]

**图 3-4　共识正常读写/调度路径**

两图的对照：TiDB 路径强调 Region Leader 先把变更放入 Raft log,再由多数派提交和 apply;OceanBase 路径强调 redo 日志进入 LS 后由 PALF 作为 append-only log 复制，并通过 LSN/SCN 与上层恢复和事务可见性衔接。

---

## 3.6 故障/异常路径图

图 3-5 对比两者在 Leader 失效后的恢复路径，关键差异是 Raft 把安全性约束放在选举阶段（日志最新者当选，不回补），而 PALF 把约束放在选举后的 reconfirm 阶段（任意优先副本当选 + Paxos Phase-1 回补）。

![[f3_5.svg]]

**图 3-5　共识故障/异常路径**

脑裂防护对比：Raft 靠 term 单调 + 多数派投票 + lease 9s < 选举超时 10s 的时间差，保证旧 Leader lease 过期才会有新 Leader,防 stale read;PALF 靠 lease 选举的「lease 续约从不中断 ⇒ 不可能存在两个 Leader」+ proposal_id 单调（更大 proposal_id 的日志会截断旧的）。两侧的共同点是：旧 Leader 即使本地进程仍活着，只要无法获得多数派或 lease/proposal_id 已失效，就不能安全提交新日志。

表 3-2 把两条故障路径的关键环节并列，便于看清「约束放置点」这一根本差异如何贯穿恢复全过程（本章第二张对比表）。

**表 3-2　故障/异常路径图**

| 故障路径环节 | Raft / TiDB | Multi-Paxos / PALF |
|---|---|---|
| 触发条件 | Follower 选举超时（心跳丢失） | Leader lease 到期 / 主动 revoke |
| 是否先做预探测 | 是，PreVote 探多数派，失败不增 term | 否，直接由 election 模块按 priority 推 candidate |
| 谁可当选 | 仅日志最完整副本（Leader Completeness 约束） | 任意优先级最高的存活副本均可，缺日志靠 reconfirm 补 |
| 上任后是否回补历史日志 | 不回补；靠选举时的日志新旧约束保证完整 | 必须 reconfirm:Prepare→多数派 max proposal_id/LSN→补齐/截断 |
| 旧 Leader 复活处理 | term 落后被拒，自动回 Follower 并截断冲突日志 | 先进 pending follower,等新 Leader 日志判定 in-flight 日志 commit/rollback |
| 未决（in-flight）日志判定 | 由新 Leader 的 commit 推进隐式决定 | 显式判定，据此让事务引擎 commit 或 rollback |
| 已知放大风险 | Hibernate Region 下故障节点的休眠 Region 不主动重选，恢复长尾 | reconfirm + pending follower 引入额外往返，恢复路径更重 |
| 官方恢复指标 | 无单一官方 RTO;受选举超时（~10s 量级）与 Hibernate 影响 | 集群内 Paxos RPO=0（架构自洽，可独立佐证）；RTO<8s 为 OceanBase 自述，第二独立源未取得（多数派存活前提，见 §3.7） |

---

## 3.7 性能、可靠性、运维影响

本节核心判断是：两条路线在稳态性能上的理论下限相同，真正拉开差距的是热点处理、可用性指标口径与运维复杂度。下面按维度分别陈述。

延迟：两者稳态写都是「一轮多数派 RTT + 本地刷盘」，理论下限相同。TiDB 一次写通常受 Region Leader、Raft log 持久化、多数派 RTT、apply 排队影响；OceanBase 一次写受 LS Leader、PALF 日志写入、多数派复制、LS apply/事务提交路径影响。读路径上，TiKV 的 Lease Read/ReadIndex 把强一致读优化为本地读（lease 内无需 RTT）；OceanBase 的 WAL 模型让事务读直接走存储引擎可见状态。PALF 论文在 3 台 32 核 2.5GHz Xeon / 256GB / 10GbE（0.2ms）、三副本的闭环 benchmark（基线为通用共识库 etcd-raft / braft）中报告 PALF 具有更高的多核可扩展性。

> 口径纠偏（论文 "5.98" 的正确语义）：论文中的 5.98 不是「PALF 相对 etcd-raft / braft 的加速比」。经核对 VLDB 2024 原文，原文为："The speedup ratio of PALF is 5.98 when the number of clients increases from 500 to 8000, whereas the speedup ratios of etcd-raft and braft are 1.386 and 1.8 respectively."。即 5.98 是 PALF 自身「客户端从 500 增到 8000」的吞吐自扩展比（多核可扩展性指标），而 1.386 / 1.8 是 etcd-raft / braft 各自的同口径自扩展比——三者衡量的都是「各系统自己的扩展能力」，并非跨系统加速比。补充约束：（1） 该数据测于 OceanBase 4.0（论文明确测试床为 OceanBase 4.0 组件），并非本章锁定的 4.2.5/4.3.5/4.4.x,存在版本错配；（2） 这是单一实验配置（32 核 / 三副本 / 闭环 client 与 leader 同机）下的微基准扩展性结果，而非协议层的稳定不变量，其外推有效性仍属（不确定）；（3） OceanBase 为利益相关方。论文同表给出的绝对吞吐为 PALF 在 8000 客户端下约 1480K append req/s——同样是 4.0 测试床下的实验值，不能外推为生产 OLTP 事实。

吞吐与扩展性：Multi-Raft 与 LS 都靠「多组并行 + 多 Leader 分布」横向扩展。TiKV 的瓶颈常在 raftstore 单线程驱动海量 Region 心跳，增加 Region 数可把 Leader 分散到不同 TiKV 节点，但热点 Region 仍可能集中；若热点集中在单个小 key range,单纯加节点未必立刻提升该 Region 的写吞吐，需要 split、打散热点或业务侧改写访问模式。OceanBase 4.x 则主动把复制组数压到服务器数量级以节省心跳/CPU（这也是它放弃「每分区一 Paxos 组」的直接动因），其扩容收益取决于 LS 与 Unit、Tenant、Primary Zone、Locality 的关系，以及 Tablet 能否迁移到更合适的 LS 或 server;若热点集中在一个 LS 的单 Leader 日志路径，增加机器也需配合 LS 均衡或业务分区策略。

RootService 的边界（逻辑中心化讨论，本章只在共识语境内涉及）：OceanBase 的 RootService 承担 LS 副本放置、Tablet 归属、Leader 位置发现等控制面职责，可从五个维度理解其性质——逻辑中心化（集中保存集群级元数据与调度决策）、性能瓶颈（高频元数据/调度请求可能集中）、高可用单点（自身以多副本部署，需自身的选主保护）、元数据依赖（路由刷新、location cache 依赖其元数据）、故障恢复依赖（LS 调度与副本重建编排经其触发）。但 RootService 不参与单个 LS 内部日志的多数派一致性——某 LS 的日志序列不分叉由 PALF 保证；RootService 短暂不可用影响的是新调度与路由发现，不会让已有有效 Leader 的 LS 绕过 Paxos safety。这与 TiDB 中 PD 的角色边界（见 §3.2）在结构上对称。

可用性 / 故障恢复：OceanBase 官方资料对集群内 Paxos 故障切换给出 RPO=0、RTO<8s（多数派存活前提下），并通过缩短选举 lease、message-driven 相对时间等优化稳定 RTO。其适用范围限于 OceanBase V4.x 的 LS/Paxos 多副本、少数派故障或多数派完整场景，不绑定 RootService、不外推到单副本或 shared-storage/跨云形态。其中 RPO=0 与「多数派 Paxos + WAL 同步落盘」的架构语义自洽，可由可达的 oceanbase.github.io Architecture 页与 PALF 论文独立佐证；但「RTO<8s」这一具体数值目前只见于 OceanBase 自有页面(`en.oceanbase.com` 对自动抓取返回 HTTP 429 bot-challenge,本轮无法程序化复核)，在可达的第三方/官方非营销页面上未取得第二个独立印证，属 OceanBase 自述指标，精确数值需进一步核验。TiKV 侧无单一官方 RTO 数字，恢复时间取决于选举超时（~10s 量级）与是否开启 Hibernate Region。

运维复杂度：TiKV 的增强多为可配置开关(`prevote`、`hibernate_regions`、`cmd_batch` 等)，调参面清晰但 Region 海量时心跳/调度需专门调优；热点多表现为 Region Leader 分布、raftstore CPU、apply 延迟、log lag、snapshot 流量。PALF 把共识、选举、日志存储、备库同步耦合在 logservice 内，运维者面对的是一个更一体化但内部更复杂的子系统，热点更可能和 LS Leader、日志同步延迟、跨 LS 事务、LS 迁移/重建相关。两侧的共性是：运维手册不能只写「等待重新选主」，还应说明如何判断复制落后、apply 堵塞、路由失效、成员变更未完成与日志恢复仍在进行。

版本演进角度：TiKV 侧 Raft 算法本体随 `raft-rs` master 演进(8.5 经 `[patch]` 始终跟 master)，工程增强多在 raftstore 层迭代，Region size、Hibernate Region 默认值、raft-engine 启用状态会随版本变化；`release-8.5` 还共存 `components/raftstore-v2/`（新版引擎），本章对比以稳定的 raftstore 为准。OceanBase 侧，PALF 自 4.0 引入 Stream/LS 模型替代「每分区一 Paxos 组」，4.2.5 / 4.3.5 / 4.4.x 三个 checkout 的 `palf/` 目录文件数为 132/132/134，目录结构基本稳定；具体跨版本语义差异（若需）需逐版本 diff,本章不做未经查证的断言，也不把 4.4.x 的某些归属（CE vs 企业版/Cloud,部分为 roadmap）当成已 GA 事实。

--- 〔文献[15]〕

## 3.8 反例与代价

每一项工程增强都对应一笔代价，下面把两条路线最典型的几处取舍并列，结论是没有「免费的优化」，只有把代价转移到何处的选择。

- Hibernate Region 的恢复代价（TiKV）：空闲 Region 停 tick 以节省 CPU,但 GitHub issue `pingcap/tidb#34906` 记录：当某 TiKV 不健康时，Leader 在该节点上的休眠 Region 不会主动重选，要等请求触达 Follower 才唤醒重选。海量 Region 下会出现「逐个被访问才恢复」的长尾，恢复时间被显著拉长；休眠 Region 续约 lease 也带来尾延迟。这是「省心跳」与「快恢复」之间的直接取舍。
- 海量 Region 的 raftstore 压力（TiKV）：Region 极多时，raftstore 处理心跳本身成为瓶颈，需要 Region 合并、调大 Region size、Hibernate 等组合调优（官方有专门的 massive-regions 最佳实践）。因此「Region 数越多越好」并不成立——太少限制 Leader 分散与调度弹性，太多则放大心跳、tick、metadata、Prometheus 指标与调度压力。
- 选举与日志解耦的复杂度代价（PALF）：换来 priority/备库灵活性的代价是必须引入 reconfirm、pending follower、proposal_id 截断、access mode、mode meta、member/learner/rebuild 等额外机制，正确性推理面更大、实现更复杂——这正是论文反复强调「为给事务引擎返回明确复制结果」而专门设计 pending follower 的原因。同理，「LS 聚合越多 Tablet 越好」也不能绝对化：热点 Tablet 被放进同一 LS 后，Leader 写入、日志同步、apply、迁移共享一条 LS 日志路径，而跨 LS 事务又需要更复杂的提交协议。
- 替代方案取舍：Raft 把安全性约束前移到选举（实现简单、可理解性高），代价是 Leader 必须是日志最完整者、对 Leader 位置的控制弱（只能靠 leadership transfer 扩展，且要求新旧 Leader 都存活）；Multi-Paxos/PALF 把约束后移到 reconfirm,换来 Leader 位置可按 priority 自由控制（利于就近、备库），代价是恢复路径更重。单一大 Raft Group 或单一大 Paxos Group 会让协议实例少，但牺牲水平扩展与热点隔离；每个 Tablet 一个共识组让迁移粒度细，但复制组数量与事务协调成本可能爆炸。没有「谁更先进」，只有目标不同下的取舍，反映的是系统历史路径与事务模型，而非协议名称本身决定胜负。

--- 〔文献[16]〕

## 3.9 测试开发视角的验证点

对测试开发而言，共识层的难点不在正常写入，而在故障与并发叠加时的边界条件。下面从功能场景、失效注入、压测指标、观测指标到测试矩阵分层给出验证点。

可测试的功能场景：TiDB 侧——单 Region 写入、多 Region 并发写入、Region split/merge 后 Leader 路由更新、leader transfer、follower catch-up、snapshot 发送、Hibernate Region 唤醒、ReadIndex / Lease Read 在 Leader 切换期间的线性一致性、成员变更（Joint Consensus）；OceanBase 侧——单 LS 事务提交、跨 LS 事务提交、LS Leader 切换、PALF reconfirm、日志落后副本重建、Tablet-to-LS 映射变化后读写可见性、standby/learner 类副本同步边界、成员变更（config_version）。

可注入的失效模式：网络分区（验证 PreVote 不打断现有 Leader、PALF lease 不产生双主）；Leader 进程 kill（验证选举超时与恢复时间）；磁盘慢/刷盘卡、follower 慢盘、单副本日志落后、部分 RPC 丢包、磁盘空间不足、apply 线程阻塞、snapshot 传输中断（验证 commit 推进与背压）；时钟漂移/回拨（验证 Lease 安全边界、OB message-driven 相对时间）；分区恢复后旧 Leader 复活（验证 Raft term 拒绝 / PALF pending follower 与 proposal_id 截断，确认 in-flight 日志的 commit/rollback 判定）；PD 元数据短暂不可用、OceanBase LS Leader 定位失效、reconfirm 中途 Leader 再次变化。

关键压测指标不要只看 QPS:写 P99 延迟、commit/apply log duration、多数派提交吞吐、Raft message drop、snapshot 流量、Leader 切换 RTO、成员变更期间可用性、海量 Region/LS 下的 CPU 与心跳开销、Region/LS Leader 分布、log lag。

关键观测指标（均为官方查证，不编造）：

- TiKV:已由源码确认的 Raftstore metric 名包括 `tikv_raftstore_append_log_duration_seconds`、`tikv_raftstore_commit_log_duration_seconds`、`tikv_raftstore_apply_log_duration_seconds`、`tikv_raftstore_raft_ready_handled_total`、`tikv_raftstore_raft_sent_message_total`、`tikv_raftstore_raft_dropped_message_total`、`tikv_raftstore_log_lag`、`tikv_raftstore_hibernated_peer_state`。对应官方 Grafana 面板（Raft IO 区的 Append/Commit/Apply log duration;Raft Propose 区的 Propose wait duration / Apply wait duration / Raft log speed;Raft Process 区的 Ready handled、Process ready duration;Raft Message 区的 Vote、Raft dropped messages）。Propose wait duration 高通常意味 raftstore 线程被 append log 或 CPU 拖住；Apply wait duration 高意味 apply 模块繁忙。
- OceanBase:reconfirm / role 状态迁移与 `committed_end_lsn` 推进体现在 logservice 内部状态与系统日志，可观察 LS Leader 分布、日志同步延迟、reconfirm 次数、LS rebuild、apply 延迟、跨 LS 事务提交延迟。与 TiKV 走 Prometheus metric 不同，OceanBase 把这些 PALF/Paxos 状态以 SQL 系统视图暴露，可在源码内核实其字段：(1) `GV$OB_LOG_STAT` / `V$OB_LOG_STAT`（底层虚拟表 `__all_virtual_log_stat`，`table_id=12254`）暴露每个 LS 副本的 `ROLE`、`PROPOSAL_ID`、`CONFIG_VERSION`、`ACCESS_MODE`、`PAXOS_MEMBER_LIST`、`PAXOS_REPLICA_NUM`、`IN_SYNC`、`BASE_LSN`/`BEGIN_LSN`/`END_LSN`/`MAX_LSN` 与对应 `*_SCN`、`ARBITRATION_MEMBER`、`DEGRADED_LIST`、`LEARNER_LIST`——即本章讨论的 proposal_id（Paxos ballot）、config_version、成员/降级/learner 列表、committed/max 位点；(2) `__all_virtual_ha_diagnose`(`table_id=12340`) 暴露 `election_role`/`election_epoch`、`palf_role`/`palf_state`/`palf_proposal_id`、`log_handler_role`/`log_handler_takeover_state`、`max_applied_scn`/`max_replayed_lsn`/`max_replayed_scn`、`enable_sync`/`enable_vote` 等，其中 `palf_state` 取值即 PALF 副本状态机 `INIT/ACTIVE/RECONFIRM/PENDING`（`log_define.h` `replica_state_to_string`），可直接观测 reconfirm 与 pending follower 状态迁移；(3) 降级/仲裁可看 `GV$OB_ARBITRATION_MEMBER_INFO`。诊断命令侧，官方提供 `obdiag`(OceanBase Diagnostic Tool,`oceanbase/obdiag`) 的 `gather`/`analyze`/`check` 系列（如 `obdiag analyze log`）做一键采集与根因分析（视图字段经 `src/share/inner_table/ob_inner_table_schema_def.py` 与 `src/observer/virtual_table/ob_all_virtual_{log_stat,ha_diagnose}.h` 核实，三版本一致）。具体 Prometheus 形态指标名（若有）仍以官方监控产品文档为准。

测试矩阵建议按三层拆开。第一层是协议安全性：多数派提交、日志连续性、旧 Leader 拒写（TiKV 验旧 Leader lease 过期后不返回过期强一致读；OceanBase 验旧 proposal_id 或未 reconfirm 的 Leader 不能继续提交可能覆盖多数派历史的日志）、读线性一致、snapshot 后状态一致。第二层是数据库语义：TiDB 中跨 Region 事务的 primary lock 与 commit 状态不能因单 Region Leader 切换而泄漏不一致；OceanBase 中跨 LS 事务不能因某个 LS reconfirm 或 Leader 变化而出现单边提交。第三层是运维可恢复性：注入故障后系统不仅要最终恢复，还要在日志、metric、诊断命令中给出足够线索，使值班人员能区分「正在追日志」「需要重建副本」「路由缓存陈旧」「控制面调度暂停」与「真实不可用」——TiDB 这部分可用已查证 Raftstore metric 做自动断言，OceanBase 这部分可用已查证的系统视图 `GV$OB_LOG_STAT`（role/proposal_id/config_version/member 列表/LSN 位点）与 `__all_virtual_ha_diagnose`（election_role/palf_state∈{INIT,ACTIVE,RECONFIRM,PENDING}/log_handler_takeover_state/replay 位点）配合 `obdiag` 做断言。

--- 〔文献[11-12]〕

## 3.10 容易误解点

1. 「TiKV 自己实现了 Raft」——不准确。Raft 算法本体在外部 `tikv/raft-rs`,TiKV 的 `raftstore` 是 Multi-Raft 调度 + 工程增强层，WAL 又在 `raft-engine`。把三者混为一谈会误判源码定位。需注意，本地 v8.5.0 仓库并无内嵌 `components/raft-engine/` 目录，`Cargo.lock` 锁定的是独立 `tikv/raft-engine` 仓库依赖。

2. 「Multi-Raft = Multi-Paxos 的同类概念」——不准确。Multi-Raft 在 TiKV 中主要是管理多个独立 Raft Group;Multi-Paxos 是 Paxos 从单值决议扩展到日志序列并复用 Leader/Phase-1 的协议族思路。两者不是一个层面的概念。

3. 「PALF 就是 Raft 换个名字 / 就是教科书 Multi-Paxos」——两头都错。PALF 论文自述「保留 Raft 式强 Leader 日志复制的简洁，但把选举从共识协议解耦」；同时它又不是教科书 Multi-Paxos（稳态复用 Phase-1、独立 lease 选举、pending follower、LSN 字节窗口、复制式 WAL 与数据库恢复接口深度耦合）。它是一个为数据库定制的、介于两者之间的工程系统。

4. 「OceanBase 是每个 Tablet 一个 Paxos Group」——不准确。4.x 文档把 LS 写成包含多个 Tablet 与 ordered redo log 的实体，LS 用 Paxos 同步日志并作为事务提交参与者；Tablet 是数据对象，LS 才是复制和日志组织单位。

5. 「Lease Read = Leader 永远可以本地读」——不准确。Lease Read 的安全性依赖 Leader 有效性、时间假设、election timeout/clock drift 边界和 apply index;Leader 切换、时钟异常、网络分区都必须纳入测试。

6. 「PALF benchmark 数字代表生产性能」「5.98 是 PALF 比 etcd-raft 快的倍数」——两头都错。论文里的 5.98 不是 PALF 对 etcd-raft / braft 的跨系统加速比，而是 PALF 自身在客户端 500→8000 时的吞吐自扩展比（同表 etcd-raft / braft 各自的自扩展比是 1.386 / 1.8）；详见 §3.7 的口径纠偏。即便取其绝对吞吐（约 1480K req/s），它仍是闭环 micro-benchmark、基线为通用共识库、硬件固定、测于 OceanBase 4.0（非锁定版本）、OceanBase 为利益相关方的实验值，不等于端到端 OLTP 生产排名。

7. 「Raft 和 Paxos 谁更先进」——伪命题。安全性等价，差异是工程取舍（强约束 vs 灵活解耦），应按目标、历史路径、分片/事务/日志组织方式来理解，而非排座次。

---

## 3.11 本章结论

1. TiDB 的共识是 Multi-Raft over `tikv/raft-rs`：算法本体在外部 crate(`raft 0.7.0` + patch master),`raftstore` 把单 Raft 包成 Region=Raft Group 并叠加 PreVote（默认开）、CheckQuorum、Lease Read（9s）、Hibernate Region（默认开）、Joint Consensus、cmd_batch、async apply 等工程增强；选举与日志复制共用一套 term/RPC。Multi-Raft 解决的是大量 Region 的复制组管理问题，而非把 Raft 改造成另一种协议。

2. OceanBase 的共识是 Multi-Paxos over PALF:复制单位是 Log Stream,采用复制式 WAL / Paxos-backed append-only log + 事务状态推进（而非经典 RSM 的编排），LSN 字节滑动窗口管理日志，选举（lease + priority）与 Paxos 日志复制解耦，新 Leader 上任先 reconfirm（Prepare→多数派 max proposal_id/LSN→恢复已接受日志）再服务，并与 LSN/SCN、事务提交、恢复路径耦合。

3. 两者安全性等价，核心取舍是约束放置点不同：Raft 把安全约束前移到选举（日志最新者当选、不回补，实现简单），PALF 后移到选举后的 reconfirm（任意优先副本可当选 + Paxos Phase-1 回补 + pending follower 判定未决日志），换来 Leader 位置可控与备库同步等数据库级能力，代价是恢复路径更复杂（推测）。

4. 在线异步 schema change 的思想渊源上，TiDB 的 Online DDL 与 Google F1 的在线异步 schema change 思想一致（此归因为思路层面的对照，而非官方声明的实现依据）（推测）；该话题属第 7/8 章范畴，本章只在共识与控制面交界处点到为止。

5. OceanBase 集群内 Paxos 官方给出 RPO=0、RTO<8s（限 V4.x LS/Paxos 多副本、多数派存活场景）：其中 RPO=0 与多数派 Paxos 架构自洽并可独立佐证；RTO<8s 为 OceanBase 自述指标，本轮未在可达的第二独立源取得印证，且不外推到单副本或 shared-storage/跨云形态。TiKV 无单一官方 RTO,恢复时间受选举超时（~10s 量级）与 Hibernate Region 影响，后者在故障节点上会拖慢重选（不确定）。

6. PALF 论文的吞吐/延迟数据是闭环 micro-benchmark + 通用共识库基线 + 测于 OceanBase 4.0（非本章锁定版本）+ 利益相关方条件下的实验值，不可外推为生产 OLTP 排名；论文中的 "5.98" 是 PALF 自身的多核吞吐自扩展比（客户端 500→8000），不是对 etcd-raft / braft 的跨系统加速比（不确定）。

7. 成员变更上，Raft 用 Joint Consensus,PALF 用 config_version + degrade/upgrade,后者多出「副本降级」语义，配合 arbitration 服务利于少数派/两 IDC 容灾。对测试开发而言，最危险的不是正常写入，而是 Leader 切换、旧 Leader 读、落后副本恢复、成员变更、snapshot/rebuild 与跨复制组事务叠加时的边界条件（推测）。

---

## 3.12 参考文献

[1] PALF: Replicated Write-Ahead Logging for Distributed Databases. 论文，VLDB 2024,PVLDB 17(12[EB/OL]. https://www.vldb.org/pvldb/vol17/p3745-xu.pdf.
 （支撑:§3.3 全部：PALF = 复制式 WAL / Paxos-backed append-only log、选举与共识解耦、reconfirm = Basic Paxos 学习缺失日志、pending follower、）
[2] In Search of an Understandable Consensus Algorithm (Raft). 论文，USENIX ATC 2014[EB/OL]. https://raft.github.io/raft.pdf.
 （支撑:§3.1/§3.2/§3.6 中 Raft 的选举、任期、日志复制、Log Matching、commit、安全性（Leader Completeness）与 Joint Consensus（§6），以及「结果等价于 M）
[3] Paxos Made Simple. 论文[EB/OL]. https://lamport.azurewebsites.net/pubs/paxos-simple.pdf.
 （支撑:§3.1/§3.3/§3.10 中 Paxos 单值实例、状态机命令序列、distinguished proposer、Phase-1/Phase-2 与 Multi-Paxos 正常路径的理论基础。）
[4] Paxos Made Live: An Engineering Perspective. 论文[EB/OL]. https://research.google.com/archive/paxos_made_live.html.
 （支撑:§3.1/§3.8 中「Paxos 工程落地远复杂于伪代码」的判断，避免把 PALF 简化为教科书协议；并辅以 *Paxos vs Raft*（arXiv 2004.05074）对「约束前移 vs 后移」的中立学术对比。）
[5] TiKV Overview / TiDB Storage. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tikv-overview/.
 （支撑:§3.2/§3.4 中 Region、Peer、Raft Group、Leader 读写、多数派写成功、数据经 Raft 接口写入 RocksDB、Region split/merge 的基础事实。）
[6] TiKV Configuration File / Grafana TiKV Dashboard(stable). 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/grafana-tikv-dashboard/.
 （支撑:§3.2/§3.8/§3.9 中 Hibernate Region 等 Raftstore 配置项，及 raftstore 官方面板/指标名（Append/Commit/Apply log duration、Propos）
[7] OceanBase Multi-replica log synchronization / Cluster architecture. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001031453.
 （支撑:§3.3 中 V4.0.0 抽象 PALF、基于 Paxos 同步多副本日志、LSN 定位、已提交日志跨副本一致，以及 LS 含 Tablet 与 ordered redo log、作为事务提交参与者。可达性说明：en）
[8] OceanBase Architecture(Developer Guide). 官方文档[EB/OL]. https://oceanbase.github.io/oceanbase/architecture/.
 （支撑:§3.3 中 Log Stream 与 Tablet 关系、Multi-Paxos 经 LS 实现复制、一 Leader 多 Follower,以及 §3.7 中 RPO=0 的架构自洽佐证。）
[9] tikv/tikv 仓库 release-8.5 @ 1f8a140b6d46 — components/raftstore/src/store/(config.rs/peer.rs/fsm/apply.rs/util.rs/read_queue.rs/hibernate_state.rs)+ Cargo.toml. 源码[EB/OL]. https://github.com/tikv/tikv/tree/release-8.5/components/raftstore.
 （支撑:§3.2:PreVote 默认 true、Lease 9s、Hibernate 默认 true、Joint Consensus（ConfChangeV2 / in_joint_state）、cmd_batch、async）
[10] oceanbase/oceanbase 仓库 v4.2.5_CE @ e7c676806fda — src/logservice/palf/(palf_handle/log_state_mgr/log_reconfirm/log_sliding_window/log_config_mgr/log_mode_mgr/election//log_engine)+ ob_log_handler. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE/src/logservice/palf.
 （支撑:§3.3/§3.6/§3.9:reconfirm（Prepare/Promise/max proposal_id+LSN、FETCH_MAX_LOG_LSN/START_WORKING、replica_num/2+1、g）
[11] oceanbase/oceanbase 仓库 v4.2.5_CE @ e7c676806fda — src/share/inner_table/ob_inner_table_schema_def.py + src/observer/virtual_table/ob_all_virtual_{log_stat,ha_diagnose}.h/.cpp + src/logservice/palf/log_define.h. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE/src/share/inner_table.
 （支撑:§3.9 中 OceanBase PALF/Paxos 可观测系统视图字段:GV$OB_LOG_STAT/V$OB_LOG_STAT(虚拟表 __all_virtual_log_stat table_id=）
[12] OceanBase Diagnostic Tool(obdiag). 官方工具/仓库[EB/OL]. https://github.com/oceanbase/obdiag.
 （支撑:§3.9 中 OceanBase 侧诊断命令名:官方一键诊断工具，提供 gather/analyze/check（如 obdiag analyze log）采集与根因分析，替代「诊断命令名需进一步查证」。）
[13] The TiKV blog — How TiKV Uses "Lease Read". 官方博客[EB/OL]. https://tikv.org/blog/lease-read/.
 （支撑:§3.2 中 Lease Read/ReadIndex 机制、选举超时 10s / lease 上限 9s、写续约、monotonic raw clock、三档强一致读分级。）
[14] TiKV deep-dive — Multi-raft & Raft / The Design and Implementation of Multi-raft. 官方博客/文档[EB/OL]. https://tikv.org/deep-dive/scalability/multi-raft/.
 （支撑:§3.2 中 Region=Raft Group（原文逐字："Raft group is divided into multiple Raft groups in terms of partitions, namely,）
[15] OceanBase RPO and RTO Explained. 官方博客[EB/OL]. https://en.oceanbase.com/blog/rpo-and-rto-explained.
 （支撑:§3.7 中集群内 Paxos RPO=0 / RTO<8s（多数派存活前提）的可用性结论。可达性说明：该域名对自动抓取统一返回 HTTP 429 + x-vercel-mitigated: challenge（真实）
[16] Recover time may be very long if TiKV enables hibernate region(pingcap/tidb#34906). 第三方/社区 issue,官方仓库[EB/OL]. https://github.com/pingcap/tidb/issues/34906.
 （支撑:§3.8 中 Hibernate Region 在故障节点上不主动重选、恢复长尾的反例。可信度局限：为 issue 讨论，行为以版本为准。 > 来源类型覆盖：论文 5（PALF、Raft、Paxos Made Simpl）
## 3.13 信息可信度自评

本章可信度分层如下：

- 源码级（最强）：TiKV 与 OceanBase 的工程增强均来自真实 checkout 的 commit-pinned 事实卡(TiKV `release-8.5` @ `1f8a140b6d46`、OceanBase `v4.2.5_CE` @ `e7c676806fda`)，包括 PreVote/Lease/Hibernate/cmd_batch 的默认值与行号、PALF 的 reconfirm/sliding-window/election/config_mgr 的类与方法签名；另据 v8.5.0 `a2c58c94`、v4.4.2_CE `e859d1b9` 亦确认对应模块存在，与基线一致。其中 `raft-rs`/`raft-engine` 内部仅确认到依赖声明级，`log_engine.cpp` 刷盘语义仅确认到模块/类级，文中已显式标注；本地 v8.5.0 无内嵌 `components/raft-engine/` 目录，按独立仓库依赖处理。

- 论文级：PALF 的复制式 WAL 模型、解耦选举、reconfirm 机制、benchmark 条件直接引自 VLDB 2024 原文；Raft 安全性、Paxos 理论、Paxos 工程化与中立对比引自 Raft 论文、*Paxos Made Simple/Live* 与 *Paxos vs Raft*。注意：论文 benchmark（含 "5.98" 自扩展比、约 1480K req/s）测于 OceanBase 4.0 测试床，与本章锁定版本（4.2.5/4.3.5/4.4.x）存在版本错配，属实验数据，不可外推为锁定版本的生产事实或协议不变量（见 §3.7）。

- 官方文档/博客级：Lease 9s、RPO=0/RTO<8s、Grafana 面板名、OB lease 选举/reconfirm 描述、LS 含 Tablet 与 redo log 等，均来自官方页面。

- 工程推测：§3.7/§3.8/§3.11 中「约束放置点不同」的取舍判断、PD/RootService 控制面边界判断、TiDB Online DDL 与 F1 思想一致的对照，属综合多来源后的工程推断，已标注。

- 不确定 / 不编造：OceanBase 侧的 PALF/Paxos 可观测系统视图与诊断工具已在 checkout 内核实（`GV$OB_LOG_STAT`/`V$OB_LOG_STAT` 经 `__all_virtual_log_stat` 暴露 role/proposal_id/config_version/member 列表/LSN 位点;`__all_virtual_ha_diagnose` 暴露 election_role/palf_state{INIT,ACTIVE,RECONFIRM,PENDING}/log_handler_takeover_state/replay 位点;官方 `obdiag` 诊断工具），已升级为；仅其 Prometheus 形态指标名（若有）仍以官方监控产品文档为准。PALF 三版本逐文件语义差异未 diff,保留「需进一步逐版本查证」;OceanBase「RTO<8s」具体数值仅见于 OceanBase 自有页面，缺第二独立非营销源，保留「自述、需进一步核验」；PALF 论文 benchmark 数字按「实验数据 + 版本错配」处理，不外推。

反查证记录：TiKV 依赖固定值经源码事实卡与 GitHub 在线 `Cargo.toml`/`Cargo.lock` 两路独立确认一致(`raft 0.7.0` + patch master、`raft-engine` branch `tikv-8.5`);Lease 9s 经源码(`config.rs` L255/L593)与官方 lease-read 博客双向印证；PALF「选举与共识解耦」经论文原文与官方多副本日志同步文档双向印证；PALF "5.98" 经原文逐字核对确认为 PALF 自身的多核吞吐自扩展比（非跨系统加速比），并叠加版本错配与不可外推两重限定；TiKV deep-dive 博客经核对不含 PreVote/Joint Consensus,已改由源码独立支撑；`en.oceanbase.com` 全域 429，相关论点尽量改由可达的 oceanbase.github.io Architecture 页、PALF 论文、源码独立印证。版本核查：本章引用版本均匹配锁定基线（TiDB 8.5.x、TiKV release-8.5、OceanBase v4.2.5_CE 对照 4.3.5/4.4.x）；PALF 标题为全称与 acronym 关系，正文已同时给出，无冲突。

---


# 第 4 章 存储引擎

## 4.1 本章核心问题

存储引擎是分布式数据库「落盘」这一步的最终承担者：上层的分片（详见第 1 章）、共识（详见第 3 章）、事务/MVCC（详见第 10、11 章）所产生的所有写入，最终都必须由某个本地引擎以可恢复、可读、可压缩、可回收的方式写到磁盘上。共识层保证多个副本按同一顺序接收写入，但真正决定读写放大、尾延迟、历史版本清理、故障恢复耗时与运维手感的，是写入落到内存与磁盘之后的形态。本章要回答的不是「用了什么引擎」，而是两个更深的问题：

1. **复用还是自研，分界在哪里？** TiDB/TiKV 选择复用通用 KV 引擎 RocksDB（并在其上做 fork、加 CF、加 Titan、再把 Raft 日志独立成 Raft Engine）；OceanBase 则从零自研 LSM-like 引擎，`src/storage/` 全树零处 RocksDB 引用（`grep -rl "rocksdb"` 计数为 0）。这条分界背后是一个工程判断：通用 KV 引擎能在多大程度上承载关系型数据库内核的语义？
2. **自研引擎的优势是什么？** OceanBase 自研存储引擎相比 RocksDB 的优势，到底是「通用 KV 性能优势」，还是「关系型数据库内核协同优势」？结论先行：不能把它写成「自研一定快于 RocksDB」。更准确的说法是，TiKV 把单机持久化能力交给成熟的 RocksDB，并在其上叠加 Raft、Region、MVCC 与调度；OceanBase 则把 MemTable、SSTable、Macroblock、Microblock、Tablet、SCN、merge 与 SQL/事务内核共同设计。本章基于源码与论文论证，证据指向后者（推测）。

把这两个问题讲清楚，需要区分三条路径：**数据路径**（写入如何落盘、读取如何归并）、**控制路径**（谁决定何时 compaction/merge、何时冻结）、**故障路径**（引擎本地损坏、合并抖动、控制面驱动失效时发生什么）。两者都属于 LSM-like 思路，但「compaction」背后的工程语义不同：RocksDB 更像通用 KV 的层级整理，OceanBase 的 merge 还承担全局一致快照、基线数据、SCN 范围与多版本清理的数据库内核职责。本章对 TiDB 与 OceanBase 各走一遍这三条路径，而不是罗列组件。

> 边界说明：本章只负责「存储引擎」层。Region/Tablet 的分片规则见第 1 章；Raft/Paxos 日志的共识语义见第 3 章；write/lock CF 内 MVCC 编码、primary lock、SCN 多版本链的事务语义见第 10、11 章。本章只确认这些数据「物理上如何组织在引擎里」。

## 4.2 TiDB / TiKV 的实现

TiDB 自身不落盘，真正的本地存储在 TiKV。官方文档将 TiKV 描述为通过单机 RocksDB 将数据快速存到磁盘，并通过 Raft 在多机间复制：客户端写入不是直接写 RocksDB，而是先走 Raft 接口，副本达成一致后再应用到本地状态机。这句话很重要：RocksDB 在 TiKV 中不是分布式协议，也不决定 Region leader，它承担的是每个 TiKV 节点上的嵌入式 KV 持久化、LSM compaction、WAL/SST 文件管理、Column Family 隔离与迭代器能力。

TiKV 的存储栈分三层：**抽象层 `engine_traits`** → **RocksDB 实现层 `engine_rocks`** → **底层 RocksDB（经 fork）**，外加一个独立的 **Raft Engine** 承担共识日志。

### 组件与抽象

TiKV 没有把 RocksDB 硬编码进核心逻辑，而是在 `components/engine_traits/src/` 定义了一组抽象 trait（`KvEngine`、`RaftEngine`、`WriteBatch`、`Snapshot` 等），`components/engine_rocks/src/` 提供 RocksDB 的具体实现（`RocksEngine`）。`engine_traits` 这个 crate 的设计目标是抽象 TiKV 持久化所需能力，且不应传递依赖 RocksDB；这说明 TiKV 的工程边界是「上层希望面向 engine trait 编程，当前主实现是 RocksDB」。两者的关键缝合点在 `components/engine_traits/src/engines.rs`：`pub struct Engines<K, R> { pub kv: K, pub raft: R }`——TiKV 从类型层面就把「KV 引擎」与「Raft 日志引擎」拆成两个独立泛型。这正是历史上 **kvdb + raftdb 双 RocksDB 实例**结构的遗留痕迹。

底层 RocksDB 并非 upstream crates.io 版本，而是 TiKV 自家 fork：`components/engine_rocks/Cargo.toml` 的 `[dependencies.rocksdb]` 指向 `git = "https://github.com/tikv/rust-rocksdb.git"`，`features = ["encryption"]`。即 TiKV 通过 `tikv/rust-rocksdb`（内含 vendored RocksDB）绑定，以便加密、CF 级别配置等定制。因此本章不写成「TiKV 架构等价于 RocksDB」，而写成「TiKV 将通用 KV LSM 能力外包给 RocksDB，再由 Raftstore/MVCC/PD 调度补齐分布式数据库语义」。

### 为什么用 RocksDB

RocksDB 是 Facebook 基于 LevelDB 的 LSM-tree 嵌入式 KV 引擎，提供了 TiKV 不必重新实现的基础设施：WAL、MemTable（skiplist）、SSTable、compaction、bloom filter、CF、snapshot、事务接口等。TiKV 用它的核心原因不是「RocksDB 天然懂分布式」，而是它在单机有序 KV、LSM 写入优化、范围扫描、快照、批量写、Column Family 与丰富调参方面足够成熟。TiKV 把「分布式」留给自己（Raft、MVCC、事务、调度），把「单机持久化 LSM」交给 RocksDB，这是典型的复用取舍。官方文档明确「TiKV uses RocksDB internally to store Raft logs and key-value pairs」。

### Column Family 与 MVCC 的物理划分

kvdb 实例内有四个 CF，源码常量在 `components/engine_traits/src/cf_defs.rs`：

- `CF_DEFAULT = "default"`、`CF_LOCK = "lock"`、`CF_WRITE = "write"`、`CF_RAFT = "raft"`
- `DATA_CFS = [CF_DEFAULT, CF_LOCK, CF_WRITE]`（注意 **raft 不在 data cf 内**）

各 CF 的职责可在 `src/storage/mvcc/txn.rs` 直接看到写入位置：`put_lock` 写 `CF_LOCK`，`unlock_key` 删除 `CF_LOCK`，`put_value` 写 `CF_DEFAULT` 并在 key 后追加 timestamp，`put_write` 写 `CF_WRITE` 并在 key 后追加 timestamp。简化地说：**lock CF** 存悲观锁与分布式事务 prewrite 阶段的 primary lock（commit 后很快删除，通常 < 1GB）；**write CF** 存 MVCC 提交记录（commit 版本信息 start_ts/commit_ts 等）与短值；**default CF** 存长值。

分流阈值经源码逐字核实（`components/txn_types/src/types.rs`，`SHORT_VALUE_MAX_LEN = 255`，判定为 `value.len() <= 255`，即边界是「≤255」；在 release-7.5/release-8.5/master 均一致）：被写入的 **value 长度 ≤ 255 字节**时，值内联进 write CF 的 Write 提交记录（`short_value` 字段）中；**> 255 字节**时，实际值落入 default CF（键编码为 user_key + start_ts），而 write CF 仍存这条 MVCC Write 提交记录，其中含 start_ts，作为指向 default CF 数据版本的「版本指针」。这里要厘清一个常见误述：write CF 存的是 MVCC Write 记录（字段为 `write_type`、`start_ts`、`short_value` 等，无任何 index 字段），其中 start_ts 是回查 default CF 数据版本的时间戳指针，而非数据库语义上的「索引」（二级索引是另一个独立 KV 概念）。短值与长值的唯一区别是值是否内联进该 Write 记录，而非「短值存 write CF / 长值改由 write CF 存索引」的二选一结构。

CF 各有独立 SST 文件与配置，但**共享同一个 WAL**，从而不同 CF 可按特性差异化调参而不增加 WAL 写次数。需要明确：**MVCC 的语义在引擎之上**。write/lock/default 三 CF 只是 TiKV 把 Percolator 模型的字节编码塞进了 schema-agnostic 的 RocksDB——RocksDB 本身不理解什么是版本、什么是锁。CF 让不同访问模式与 compaction 策略可以分开调优，但也带来跨 CF 读写一致性、压缩过滤与 compaction 抖动的协同成本。MVCC 编码细节属第 11 章范畴，本章只确认物理 CF 划分。

### kvdb / raftdb → Raft Engine 的历史演进

Raft log 的存储位置经历过一次关键迁移，版本口径需逐一对齐：

- **早期**：raft log 与 kv data 各用一个 RocksDB 实例（raftdb / kvdb）。raftdb 只有一个 CF（`raftdb.defaultcf`）。问题在于：将顺序的 raft 日志置于为通用读写优化的 LSM 之上，日志会被转成 KV 对再经历 compaction，产生显著**写放大**与额外 I/O。
- **演进**：TiKV 自 **v5.4 引入 Raft Engine**（一个专为 multi-raft 日志设计的 log-structured 引擎），自 **v6.1 起默认启用**。官方 v6.1 release note 明确写出，从 v6.1.0 起默认使用 Raft Engine 作为日志存储引擎，并给出特定负载下的 I/O、CPU、吞吐与尾延迟收益。
- **现状（release-8.5）**：`etc/config-template.toml` 的 `[raft-engine]` 段注释为 `Determines whether to use Raft Engine to store raft logs. When it is enabled, configurations of raftdb are ignored.`，默认 `# enable = true`（注释掉即默认 true）。即 8.5 默认走 Raft Engine，`[raftdb]` 段仍保留为 legacy。历史兼容路径中，源码 `components/engine_rocks/src/raft_engine.rs` 仍显示 `RocksEngine` 可通过 `CF_DEFAULT` 读取 Raft state/entry，但这不是 release-8.5 默认日志路径的完整描述。

Raft Engine 是**独立仓库** `github.com/tikv/raft-engine`，TiKV 通过 git 依赖锁定 branch `tikv-8.5`（根 `Cargo.toml`），并用适配层 `components/raft_log_engine/` 把它接到 `engine_traits::RaftEngine` 上（`RaftLogEngine(Arc<RawRaftEngine<ManagedFileSystem>>)`）。其设计借鉴 **bitcask**：GitHub README 逐字称 "a log-structured design similar to bitcask"，每个 Raft Group 持有自己的 memtable（README："each Raft Group holds its own memtable, containing all the key value pairs and the file locations of all log entries"），用户写入顺序追加到 active log file 并周期 rotate（"sequentially written to the active log file, which is periodically rotated below a configurable threshold"）；清理由用户主动调用 `purge_expired_files()` 触发（"only when the user voluntarily calls the `purge_expired_files()` routine"）；支持 lz4 压缩（"supports lz4 compression over log entries"）、追求最小写放大（"Minimum write amplification"）。TiKV 端的调用周期由 `raft-engine-purge-interval` 控制，**默认 10 秒**：`components/raftstore/src/store/fsm/store.rs` 起一个 `purge-worker`，以 `raft_engine_purge_interval` 为周期 `spawn_interval_task` 调用 `manual_purge()`（最终落到 Raft Engine 的 `purge_expired_files()`），该默认值在 `components/raftstore/src/store/config.rs` 定义为 `raft_engine_purge_interval: ReadableDuration::secs(10)`，并与 `etc/config-template.toml` 的 `# raft-engine-purge-interval = "10s"` 一致。官方称其相比 RocksDB 可降低写 I/O 25%~40%、CPU 约 10-12%、尾延迟约 20%，前台吞吐升约 4-5%；但这些数字依赖具体 heavy workload，详见 §4.7。

### Titan：大 value 的 KV 分离

TiKV 还可启用 **Titan**（RocksDB 插件，灵感来自 WiscKey），在 flush/compaction 时把大 value 从 LSM 分离进 blob file，LSM 内只留位置索引，以降低大 value 场景写放大。适用 value ≥ 1KB（部分场景 512B）；代价是范围扫描性能下降（官方测试称比 RocksDB 低 40% 到数倍）、空间放大可能翻倍。自 **v7.6.0 起对新建集群默认启用**。

### TiKV 正常写入路径小结

TiKV 的写入可拆成三段。第一段，SQL 层编码为 KV mutation，定位 Region leader。第二段，leader 将写请求纳入 Raft log，复制给 follower，达到多数派后提交。第三段，apply 线程将已提交日志应用到 RocksDB：锁信息进 lock CF，实际 value 进 default CF（含 start_ts），提交记录进 write CF（含 commit_ts），后台由 RocksDB flush/compaction 形成 SST 文件。控制路径上，PD 负责 Region 元数据、调度与 TSO 等逻辑中心化能力，但其对存储引擎的影响主要体现在 Region split/merge、leader 调度、热点迁移会改变 RocksDB 局部写入与 compaction 压力（PD 的中心化定位详见第 1、13 章，本章不展开）。 〔文献[1-4,11-13]〕

> 源码模块（commit-pinned）：tikv/tikv @ `release-8.5`——`components/engine_traits/src/`、`components/engine_rocks/src/`、`components/raft_log_engine/src/`、`src/storage/mvcc/txn.rs`、`components/txn_types/src/types.rs`；独立仓库 tikv/raft-engine @ branch `tikv-8.5`、tikv/rust-rocksdb、tikv/titan。Raft Engine 内部 WAL/pipe log 的**磁盘二进制格式**在独立仓库，本地未单独 checkout，仅经 git 依赖确认仓库 + branch，该二进制格式细节需进一步查证（README 仅给出逻辑写序与「各 Raft Group 共享同一 log stream」，未给磁盘格式）。另据 v8.5.0 `Cargo.lock` 亦确认 `raft-engine` 与 `rust-rocksdb` 均 pin 到 `tikv/` 下独立 commit（`raft-engine` 锁定 `branch=tikv-8.5#3d844051b8a922d3df0d45f2d523ab8e9d537721`）。

## 4.3 OceanBase 的实现

与 TiKV 的复用路径相对，OceanBase 不复用任何开源存储引擎，`src/storage/` 全树零处 RocksDB。其引擎是为关系型分布式内核量身定制的 LSM-like 结构，分**分布式引擎**（线性扩展、Paxos 多副本、分布式事务）与**单机引擎**（关系型 + 内存数据库技术融合）两层。这套结构与第 1 章的 Tablet/LS 边界相连：Tablet 是数据承载单元，LS（Log Stream）承载日志复制与事务参与，存储引擎在 Tablet 内组织多版本数据。

### 为什么不用 RocksDB

OceanBase 官方给出三条理由：

1. **Schema 支持**：RDBMS 一行有很多强类型列，而 RocksDB "the schema is very simple, there are just keys and values"。
2. **强数据类型**：RDBMS 每列有确定类型，RocksDB value 类型不确定。
3. **复杂大事务**：RDBMS 单事务可能改动数百万行且要求原子，RocksDB 事务 "very simple"。

结论："no open-source storage components can meet our requirements for a distributed RDBMS"。更完整地说，OceanBase 是分布式关系型数据库内核，它需要让存储格式与 SQL 行格式、Tablet、SCN、多版本、Tenant 资源、major baseline、校验、列存转换与备份恢复协同；RocksDB 可以作为通用有序 KV 引擎很好地完成写优化与范围扫描，但它不知道 OceanBase 的 Tablet 生命周期、Major SSTable 基线、SCN 范围、macro/micro block 布局与 merge 语义。

### 数据路径：MemTable → Mini/Minor/Major SSTable

官方架构文档将 Tablet 内部结构描述为四层：MemTable、L0-level Mini SSTable、L1-level Minor SSTable、Major SSTable。DML 先写 MemTable，MemTable 达到一定大小后 flush 成 Mini SSTable，Mini SSTable 数量达到阈值后合并为 Minor SSTable，业务低峰时再将 MemTable、Mini SSTable、Minor SSTable 合并成 Major SSTable。

写入先进的 **MemTable** 是增量数据，可读写、在内存。其内部索引不是 RocksDB 的 skiplist，而是自研 `ObQueryEngine`，底层为 `ObKeyBtree`（B-tree，范围扫描）+ hash（单行 get 加速）（源码逐字：`ObQueryEngine query_engine_`、`namespace keybtree`、`class ObKeyBtree`）。源码 `src/storage/memtable/` 也显示其与 `mvcc/ob_mvcc_engine.*`、`mvcc/ob_query_engine.*`、`ob_memtable_iterator.*`、`ob_memtable_mutator.*`、redo log 生成与 DML 统计代码同处一层。这意味着 OceanBase 的 MemTable 同时服务写入暂存、事务可见性、行级访问、迭代与后续 merge，而不是只提供一个通用 KV memtable。MVCC 行多版本链存于 `mvcc/ob_mvcc_row.*`（属第 11 章）。

SSTable 层级被**显式建模为表类型**，源码 `src/storage/ob_i_table.h` 的 `enum TableType`：`MAJOR_SSTABLE=10, MINOR_SSTABLE=11, MINI_SSTABLE=12, META_MAJOR_SSTABLE=13, ...`。三级语义：

- **Mini SSTable**：冻结的 MemTable 直接转储落盘。
- **Minor SSTable**：多个 mini 合并成更大的 minor。
- **Major SSTable = 基线数据（baseline）**：与旧基线合并产生新基线，只读。

与这三级 SSTable 相对应的合并类型枚举在 `src/storage/compaction/ob_compaction_util.h` 的 `enum ObMergeType`，明确列出 `MINI_MERGE`（仅刷 memtable）、`MINOR_MERGE`（合并多个 mini sstable 成更大的 mini/minor sstable）、`HISTORY_MINOR_MERGE`、`MAJOR_MERGE`（与基线合并产生新基线）、`MEDIUM_MERGE`（分区级中间合并）、`META_MAJOR_MERGE`、`CONVERT_CO_MAJOR_MERGE`、`INC_MAJOR_MERGE` 等类型，并自带注释。其语义与 RocksDB 的 leveled/universal compaction 根本不同（见 §4.4），因此 OceanBase 的 compaction 必须按自身术语写，不应直接套 RocksDB 概念。

### 物理结构：Macroblock / Microblock

![[f4_1.svg]]
**图 4-1　SSTable ⊃ Macroblock ⊃ Microblock 嵌套结构**

官方对象存储文档写明：OceanBase 的某些数据库对象数据存储在一个或多个 SSTable 中；一个 SSTable 包含一个或多个 macroblock，一个 macroblock 包含一个或多个 microblock，一个 microblock 包含一行或多行。

OceanBase 的 **SSTable**（磁盘基线/落盘增量数据）物理上由固定大小 **Macroblock** 组成，默认 2MB：`deps/oblib/src/lib/ob_define.h` 的 `OB_DEFAULT_MACRO_BLOCK_SIZE = 2 << 20; // 2MB`（该常量在 v4.2.5_CE/v4.3.5/v4.4.x 三版本逐版本核对一致），使用点在 `src/observer/ob_server.cpp`（`storage_env_.default_block_size_`）。Macroblock 是写出与 GC 的基本单位，由 `ObBlockManager` 统一管理；写出器为 `ObMacroBlockWriter`、`ObMicroBlockWriter`。源码 `src/storage/blocksstable/ob_macro_block.h` 中 `ObMacroBlock` 提供 `write_micro_block`、`write_index_micro_block`、`flush`、`get_macro_block_meta` 等接口，并维护 row count、micro block count、checksum、bloom filter、compaction_scn、max_merged_trans_version 等字段。可见 macroblock 不只是「一个 2MB 文件块」，它还参与索引、校验、压缩、Bloom filter、merge 信息与事务版本边界。

需要厘清层级归属：Macroblock 是**磁盘 SSTable 存储层**的概念，是 compaction/转储落盘的基本单位；**MemTable 不直接使用 macro block**——MemTable 是内存中的动态增量数据，内部为 `ObQueryEngine`（B-tree + hash）双索引结构，只有当 MemTable 转储（dump）落盘生成 SSTable 时才涉及 macro block。切勿把「2MB Macroblock」归属到 MemTable 层。

Macroblock 内部再切分为变长 **Microblock**（读的基本 I/O 单位），内部按列做 encoding/压缩（bit-packing、字典、RLE、delta 等）以省存储。Microblock 大小由 **CREATE/ALTER TABLE 的 `BLOCK_SIZE` 表选项**控制，**默认 16KB**（官方文档示例与源码常量 `OB_DEFAULT_SSTABLE_BLOCK_SIZE = 16 * 1024` 一致，`BLOCK_SIZE = 16384`）。其**可调范围存在来源冲突**：官方设计博客/概览称 4KB-512KB 或 8-512KB；而 OceanBase 源码的 DDL 解析器（`src/sql/resolver/ddl/ob_ddl_resolver.{h,cpp}`，经对抗反查证）定义 `MIN_BLOCK_SIZE = 1024`（1KB）、`MAX_BLOCK_SIZE = 1048576`（1MB），报错文案为 "block size should between 1024 and 1048576"，三个锁定版本一致——即源码层面真实可调范围是 **1KB-1MB**，与博客的「4-512KB」在上下界均不符。本章按「默认 16KB（BLOCK_SIZE 表选项，源码常量支撑）、可调范围以源码 1KB-1MB 为准」书写，博客所述 4-512KB 与源码冲突，精确边界标注为「存在版本冲突」，不做绝对断言。

### 控制路径：Major Freeze / Major Merge 由 RootService 驱动

**表 4-1　Freeze / Minor / Major 三类合并（compaction）动作对照**

| 动作 | 触发条件 | 数据流向 | 产物 | 主要代价 |
|---|---|---|---|---|
| Freeze（冻结） | MemTable 内存达到「memstore_limit_percentage × freeze_trigger_percentage」 | active MemTable 切为只读 frozen MemTable 并开新活跃表 | frozen MemTable（真正落盘是随后的 mini merge） | 仅切表、代价很轻 |
| Minor compaction（minor merge） | freeze 后自动调度；把若干 mini SSTable 合并 | frozen MemTable 与若干 mini SSTable 压实归并 | 更大的 Minor SSTable（仍属增量侧，不触碰基线） | 增量侧整理，控制读路径归并表数量 |
| Major compaction（major freeze + major merge） | 租户 minor compaction 次数达「major_compact_trigger」，或手动「alter system major freeze」；RootService 每日定时选全局版本号 | 基线 Major SSTable 与所有增量合并，跨副本基于同一全局版本 | 新的、全集群版本一致的只读 Major SSTable（基线），并由 checksum 校验各副本 | 集群级、跨副本协调的重动作 |

这是 OceanBase 与 RocksDB 最根本的区别。要把控制路径讲透，需要看清三个动作的层次关系，它们容易被混为一谈。

第一，**冻结（freeze）**只是把活跃 MemTable 切成只读的冻结 MemTable 并开一个新活跃表，代价很轻；真正落盘是随后的 mini merge。官方 compaction 文档说明 MemTable 可分为 active 与 frozen，当 MemTable 内存达到 `memstore_limit_percentage × freeze_trigger_percentage` 时触发 freeze，active MemTable 变为 frozen MemTable，然后自动调度 minor compaction。

第二，**minor compaction（minor merge）**把若干 mini SSTable 合并成更大的 minor，仍属「增量侧」的整理，目的是控制读路径需要归并的表数量，不触碰基线；minor compaction 会把 frozen MemTable 与 SSTable 压实，并在静态数据中记录 clog replay checkpoint。当租户 minor compaction 次数达到 `major_compact_trigger` 时可自动触发 major compaction，也可手动 `alter system major freeze`。

第三，**major freeze + major merge** 才是周期性、跨副本、与全局版本号绑定的重动作。Major freeze 不是本地自治的 compaction，而是集群级、由 RootService 调度的动作：源码 `src/rootserver/freeze/` 的 `ObDailyMajorFreezeLauncher` 每日定时选定一个**全局版本号**，触发租户所有副本基于该版本做 major merge 生成新基线，并由 checksum 校验各副本一致。官方表述："A user selects a global version on a regular basis... major compaction is initiated for all replicas of tenant data... The baseline data of the same version is physically consistent for all replicas."它把基线 Major SSTable 与所有增量合并，产出一份新的、全集群版本一致的只读基线。

换言之，前两者是「为了让读不至于太慢」的局部维护，第三者是「为了让分布式一致读与基线版本得以推进」的全局事件——这也是为什么前两者由各 OBServer 本地决策，而 major freeze 必须由 RootService 统一驱动。多版本 GC 与快照点绑定：`src/storage/compaction/ob_medium_compaction_func.h` 的 `ObMediumCompactionScheduleFunc` 通过 `medium_snapshot` 分批做分区级合并与多版本清理（源码逐字 `decide_medium_snapshot`/`choose_medium_snapshot`），而非 RocksDB 那种纯按 key range / level 触发。

读路径因此呈现「基线 ⊕ 增量」的归并形态：一次查询要把命中的 Major SSTable（基线）、若干 Minor/Mini SSTable（已落盘增量）与 MemTable（内存增量）按主键有序归并，再按读取快照版本（由 GTS 提供的全局一致快照，详见第 10 章）裁剪可见版本。增量侧表越多，归并扇入越大、读放大越高——这正是 minor compaction 要持续压缩 mini 数量、以及 major merge 要周期推进基线的根本动力。RocksDB 的读路径同样要跨 MemTable 与多层 SSTable 归并，但它没有「基线」这一全局锚点，版本可见性裁剪也不在引擎内完成，而由 TiKV 在引擎之上按 ts 处理。

### 版本演进：列存（自 4.3.0）与融合（4.4.x）

- **列存（列存存储引擎 / CS 列式编码）自 V4.3.0 引入**（与第 5、16 章口径一致）。本章源码级仅有 4.3.5 checkout，故源码层面仅能确认：`4.3.5` 已含 `src/storage/column_store/`——其 column group（cg）/列式 CO SSTable 与 CO merge dag，在同一套 macroblock/microblock + major/minor 框架上做行列混存（PAX 风格）；而 `4.2.5_CE` 无此目录。即源码仅能证实「4.3.5 已含列存目录、4.2.5_CE 无」，不能据此推断列存「自 4.3.5 才引入」——其引入版本以官方 V4.3.0 口径为准。
- **4.4.x（TP+AP 融合分支）**：`src/storage/` 结构与 4.3.5 一致，在同一自研引擎上叠加 TP+AP。develop 分支 `ObMergeType` 还可见 `CONVERT_CO_MAJOR_MERGE`（行存转列存）等新类型。 〔文献[6-10,14-15]〕

> 源码模块（commit-pinned）：oceanbase @ tag `v4.2.5_CE`——`src/storage/memtable/`、`src/storage/blocksstable/`、`src/storage/compaction/`、`src/storage/ob_i_table.h`、`src/rootserver/freeze/`、`deps/oblib/src/lib/ob_define.h`、`src/sql/resolver/ddl/ob_ddl_resolver.{h,cpp}`；列存见 `4.3.5` 的 `src/storage/column_store/`；融合见 `4.4.x` 的 `src/storage/`。`ob_macro_block.h` 的 checksum/Bloom filter/compaction_scn 字段另据 4.4.x 分支亦确认。部分文件标识符被规范化污染，相关点仅确认到模块级。

## 4.4 核心差异对比

表 4-2 是本章主对比表，逐维度对照两者在存储引擎层的取舍：

**表 4-2　存储引擎:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB / TiKV | OceanBase | 影响 |
|---|---|---|---|
| 引擎来源 | 复用 RocksDB（fork `tikv/rust-rocksdb`）+ 独立 Raft Engine | 100% 自研 LSM-like，零 RocksDB | TiDB 省去造轮成本；OB 可深度协同关系内核，但维护成本更高 |
| 数据/日志分离 | kv data（kvdb，RocksDB）与 raft log（Raft Engine，8.5 默认）各用独立引擎 | LSM 数据 + PALF（Paxos-backed append-only log，复制式 WAL，详见第 3 章）分离 | 二者都把「共识日志」从「数据 LSM」剥离以降写放大 |
| MemTable 索引 | RocksDB skiplist | 自研 `ObQueryEngine` = B-tree + hash | OB 单行 get / 范围扫描各有专用索引 |
| 关系语义位置 | 在引擎**之上**（MVCC 编码进 default/write/lock CF，源码 `txn.rs` 可见写入位置） | 在引擎**之内**（schema/列类型/编码/Tablet/SCN 深度耦合） | 核心问题答案的来源（见 §4.8） |
| compaction 触发 | 每实例本地自治，按 key range / level，**无全局快照点** | major freeze 由 RootService 选全局版本号，跨副本协调 | OB 服务于分布式一致读 + checksum；RocksDB compaction 与事务快照无关 |
| 基线概念 | 无「基线」概念 | 有（Major SSTable），基线 + 增量分离 | OB 读路径 = 基线 ⊕ 增量 on-the-fly 归并 |
| 物理块 | RocksDB SST（可变，默认目标文件大小级） | 固定 2MB Macroblock + 变长 Microblock（默认 16KB） | OB 块管理/GC 单位固定，维护 checksum/Bloom/compaction_scn，利于跨介质均衡写 |
| 大 value | Titan 插件 KV 分离（v7.6.0 默认） | 列存编码 + microblock 压缩 | 不同的省空间路径 |

第二张表聚焦「优势到底是什么」这一核心问题，避免把自研简单等同于「更快」：

**表 4-3　存储引擎:TiDB 与 OceanBase 核心差异对比**

| 问题 | RocksDB / TiKV 路线 | OceanBase 自研路线 |
|---|---|---|
| 通用 KV 性能 | RocksDB 是成熟通用 KV LSM，引擎质量、工具与生态是主要收益 | 公开资料不能证明 OceanBase 在任意 KV workload 上天然优于 RocksDB（不确定） |
| 关系型内核协同 | 需把 SQL/MVCC 语义编码进 KV key/value/CF，再由上层恢复语义 | 可把行格式、快照版本、macro/micro block、merge 与校验纳入同一内核设计 |
| 维护成本 | 依赖 RocksDB 行为/版本/参数，TiKV 维护 wrapper 与适配层 | 自研存储格式、压缩、merge、诊断与升级兼容都自己承担 |
| 可替换性 | `engine_traits` 显示抽象意图，但主路径仍重度依赖 RocksDB 能力 | 存储层与事务/SQL/Tablet 深耦合，替换通用引擎成本高（推测） |

## 4.5 正常路径图

![[f4_2.svg]]

**图 4-2　存储引擎正常读写/调度路径**

两条路径放在一起看，关键差别在于：TiKV 的 apply 与 RocksDB compaction 是解耦的后台过程（compaction 何时触发由本地 LSM 状态决定，与事务无关），OceanBase 的 freeze/merge 与存储版本推进关系更近（major merge 与全局版本号、基线、checksum 绑定）。这就是「通用 KV 引擎」与「数据库内核存储引擎」的关键差别。图里故意没有把 TiKV 的 Region 与 OceanBase 的 Tablet/LS 展开成分片章节，因为本章重点是本地存储引擎。

## 4.6 故障 / 异常路径图

![[f4_3.svg]]

**图 4-3　存储引擎故障/异常路径**

控制路径关键差异：TiKV 的 compaction 故障是**本地**的（write stall 影响单实例），控制面（PD）不参与单机 compaction，关键不是「RocksDB 挂了就全库挂」，而是局部 Region、局部 TiKV store、Raft log 与 RocksDB LSM 背压如何互相放大；OceanBase 的 major freeze 故障是**控制面**的（RootService 调度依赖），会阻塞全局新基线推进，关键也不是「merge 等同于后台整理文件」，而是 frozen MemTable、SSTable 层级、major baseline 与 SCN/校验之间要保持一致，merge 延迟会让读路径需要实时合并更多版本与层级，从而影响尾延迟。这正是「逻辑中心化」在存储层的体现——RootService 是调度/元数据依赖。RootService 的中心化是一个需要分维度看的问题：逻辑中心化、性能瓶颈、高可用单点、元数据依赖、故障恢复依赖五个维度各不相同，不能简单一句「单点」概括（系统性分析详见第 1、13 章，本章不重复）。

## 4.7 性能、可靠性、运维影响

**延迟 / 吞吐**。TiKV/RocksDB 路线的可预期性来自成熟 LSM KV。RocksDB 官方对 leveled/tiered compaction 的描述显示，leveled 通常牺牲写放大换取较低空间放大，tiered 降低写放大但增加读/空间放大；TiKV 可通过 CF、block cache、write buffer、compaction guard 等参数进行局部调优。把 raft 日志独立成 Raft Engine 后，官方称写 I/O 降 25%~40%、CPU 降约 10-12%、尾延迟降约 20%、前台吞吐升约 4-5%——但这些数字是 PingCAP 内部 benchmark、特定 heavy workload 下的结果，不是普适增益：收益主要来自消除「日志经 RocksDB compaction 的写放大」，在小日志量/读密集场景增益有限。OceanBase 的基线+增量结构使**写入只碰 MemTable**（顺序内存写），major merge 可在低峰合并 MemTable、Mini SSTable、Minor SSTable 生成 Major SSTable，使读路径回到较短链路，对稳定读延迟有利；但归并层数越多读放大越大，且全局 major freeze/major merge 若配置不当，会与前台负载争抢 CPU、I/O、内存与租户资源。OceanBase 的优势因此不是「每次写入都更快」，而是当数据库内核知道行格式、快照、Tablet、checksum 与块布局时，可以在 merge、读路径与恢复路径之间做整体权衡（此为基于设计取向的推断，推测）。

**可用性 / 故障恢复**。两者数据持久性都靠多副本（Raft/Paxos）而非单机引擎冗余。单机引擎损坏即单副本失效，由共识层补副本。TiKV 将副本一致性主要交给 Raft，本地持久化交给 RocksDB/Raft Engine：Raft leader 丢失后，只要多数派与日志完整，Region 可继续服务；本地 RocksDB 的 compaction 或文件损坏会影响对应 store/peer，但不会天然破坏其他副本。OceanBase 同样依赖副本同步保障持久化，但存储层还要保证 freeze、merge、checksum、SCN 范围与 Major SSTable 基线正确——其可靠性收益是数据生命周期更可由数据库内核统一管理，代价是更多内部状态需要诊断工具理解。RocksDB write stall 是 TiKV 的典型运维风险点，OceanBase major merge 抖动则是其典型运维风险点（见 §4.8）。

**扩展性**。两者都是 share-nothing 水平扩展（OB shared-storage 形态详见第 23 章，本章不展开）。固定 2MB Macroblock 让 OceanBase 在 SSD/HDD 上写出都很规整，利于块级 GC 与跨介质均衡。

**运维复杂度**。TiDB 的问题常落在「哪个 Region/哪个 CF/哪个 TiKV store/哪个 Raft Engine 目录」上；OceanBase 的问题常落在「哪个 Tenant/Tablet/LS/merge 任务/major version/SCN 范围」上。下表对照两者的运维抖动面： 〔文献[5]〕

**表 4-4　性能、可靠性、运维影响**

| 维度 | TiKV | OceanBase |
|---|---|---|
| 主要抖动源 | RocksDB write stall（L0/compaction backlog） | Major merge 周期合并（集群级 I/O/CPU 峰值） |
| 调参面 | per-CF RocksDB 参数 + Raft Engine + Titan | 合并调度参数（时间窗、并发、轮转）+ 编码/压缩 |
| 控制面依赖 | compaction 本地，不依赖 PD | major freeze 依赖 RootService 调度 |
| 典型缓解 | 调 L0 触发阈、限速、加盘 | 错峰合并、progressive/rotation merge、加副本分摊 |

## 4.8 反例与代价

**TiKV / RocksDB 的代价**。RocksDB 是 schema-agnostic 字节 KV：它不知道「列」「类型」「一个事务改了哪些行」，所以关系语义必须由 TiKV 在引擎之上重新编码（default/write/lock CF + MVCC 编码）。代价是：（1）同一逻辑行被拆进多个 CF，跨 CF 读取需多次查找；（2）RocksDB 通用 compaction 不理解事务快照，GC 旧版本要靠 TiKV 额外的 GC worker 协调（详见第 11 章）；（3）大 value 写放大需 Titan 补救，而 Titan 牺牲范围扫描；（4）raft 日志早期置于 LSM 的历史遗留问题要靠独立 Raft Engine 才得以解决，而 Raft Engine 默认化也增加了日志引擎迁移、恢复、格式版本与目录规划的运维事项。这些都是「通用引擎承载关系语义」的协同成本。

**OceanBase 自研的代价**。（1）**复杂度**：整套 memtable/blocksstable/compaction/freeze/column_store 都需自行实现与维护，Macroblock/Microblock、SSTable meta、merge type、column-store convert、checksum、backup/restore 都要保持跨版本兼容——源码中 `ObMergeType` 的类型数量本身就是复杂度信号；（2）**Major merge 抖动**：集群级周期合并是真实的性能波动来源，生产上必须错峰、限流；（3）**与 RootService 强耦合**：major freeze 调度依赖控制面，控制面受阻则新基线无法推进、增量堆积、读放大上升（见 §4.6）；（4）收益并非「免费更快」——自研引擎换来的不是通用 KV 性能上的全面领先，而是关系型分布式语义的深度协同。

**几类典型反例**。第一类是小规模或低写入系统：TiKV 直接复用 RocksDB 可把存储引擎维护成本压到较低；若业务不需要深度定制行存块格式、major baseline、租户级 merge 与多版本清理策略，自研 OceanBase 式存储引擎未必划算（推测）。第二类是极端写入与热点 workload：RocksDB 路线可能遇到 L0 堆积、write stall、CF 内 compaction 不均衡；OceanBase 路线可能遇到 freeze/merge 跟不上、frozen MemTable 或 Mini/Minor SSTable 层级过多导致读路径合并成本上升——两者都不是「LSM 写入一定无限快」。LSM 的本质是把随机写变成顺序写与后台重写，前台收益会以后台 compaction/merge 与空间放大偿还。第三类是错误对比：说「RocksDB 是 KV，所以 OceanBase 自研一定更高级」不准确，说「OceanBase 也是 LSM，所以与 RocksDB 差不多」也不准确——正确对比是 RocksDB/TiKV 侧重通用 KV 引擎加分布式外壳，OceanBase 侧重数据库内核一体化存储，两者优化目标不同，不能脱离 workload、硬件、版本与运维能力排名（推测）。

**取舍总结**：TiDB 用「复用 + 在引擎上叠加语义」换开发速度与生态成熟度；OceanBase 用「自研 + 语义内嵌引擎」换基线/全局快照/行列混存/编码压缩的深度协同。没有谁绝对更优，只有面向不同约束的工程选择。

## 4.9 测试开发视角的验证点

**可测试功能场景**：TiKV 侧——CF 分流正确性（value ≤255B 内联进 write CF 的 Write 记录，>255B 时值入 default CF、write CF 存含 start_ts 的 Write 提交记录）、同一行多次更新后三 CF 读写行为、锁残留处理、MVCC GC 后历史版本读取、手动 compact 对前台延迟的影响、Raft Engine 启用/关闭切换与首次迁移及重启追日志行为、Titan 大 value 分离后的点读/范围扫描。OceanBase 侧——MemTable 达阈值触发 freeze、minor compaction 生成 Minor SSTable、major freeze/major merge 生成 Major SSTable、读路径跨 MemTable/Mini/Minor/Major 归并正确性、BLOCK_SIZE 表选项的边界值（下界 1KB、上界 1MB）拒绝/接受行为、major freeze 全局版本一致性与 checksum 校验、merge 中断后的校验与恢复。

**可注入失效模式**：TiKV——磁盘满 / I/O 错误触发 RocksDB write stall、人为堆积 L0 观察限速、Region leader kill、单 store 磁盘慢、RocksDB compaction 线程受限、Raft Engine 目录 I/O 抖动、lock CF 大量写入后手动 compact。OceanBase——杀掉 RootService 观察 major freeze 是否阻塞、增量是否堆积，OBServer leader 切换、租户 memstore 压力、minor compaction 延迟、major merge 与前台查询并发、macroblock 读 I/O 慢、Tablet 层级增多、注入副本基线不一致观察 checksum 报警。上述场景应关注尾延迟而非只看平均吞吐。

**关键压测指标**：写吞吐/尾延迟在 compaction/merge 期间的塌陷幅度；Raft Engine vs raftdb 的写 I/O 量对比；major merge 窗口内业务延迟抬升；apply backlog、Raft log append 延迟、RocksDB flush/compaction 耗时、L0 文件数、pending compaction bytes、block cache 命中、CF 级写放大（TiKV 侧）；租户 MemStore 使用率、freeze 次数、minor/major merge 耗时、merge backlog、读路径合并层数、macroblock/microblock I/O、checksum/merge error（OceanBase 侧）。

**可用诊断命令与参数（仅写已查证项）**：TiKV 官方文档确认 `tikv-ctl compact` 可选择 `-c default|lock|write` 与 `-d kv|raft`，`tikv-ctl raft` 可查看 Raft 状态机信息；配置文档确认 `lock-cf-compact-interval`、`lock-cf-compact-bytes-threshold`、`raft-engine.format-version` 等参数存在。OceanBase 官方文档确认可通过 `alter system minor freeze` 与 `alter system major freeze` 手动触发，并确认 `memstore_limit_percentage`、`freeze_trigger_percentage`、`major_compact_trigger` 等参数存在。合并/compaction 状态的内部视图名已在 v4.2.5_CE 源码 `src/share/inner_table/ob_inner_table_schema_def.py` 与 `src/observer/virtual_table/` 逐字核实：底层系统表 `__all_merge_info`、`__all_zone_merge_info`；SQL 可见视图含 `GV$OB_MERGE_INFO` / `V$OB_MERGE_INFO`、`GV$OB_COMPACTION_PROGRESS`、`GV$OB_TABLET_COMPACTION_PROGRESS`、`GV$OB_TABLET_COMPACTION_HISTORY`（对应虚拟表 `ALL_VIRTUAL_TABLET_COMPACTION_HISTORY`）、`GV$OB_COMPACTION_DIAGNOSE_INFO`、`GV$OB_COMPACTION_SUGGESTIONS`，以及 major compaction 进度的 `DBA_OB_MAJOR_COMPACTION` / `CDB_OB_MAJOR_COMPACTION`、`DBA_OB_ZONE_MAJOR_COMPACTION` / `CDB_OB_ZONE_MAJOR_COMPACTION`。具体 Prometheus metric 名与上述视图的各列名尚未逐字核验，需进一步查证，不在此编造。

## 4.10 容易误解点

1. **「TiKV 用 RocksDB 所以存储引擎只是 RocksDB / 更简单 / 性能更弱」**——错。RocksDB 只负责 TiKV 节点内的 KV 持久化与 LSM 行为；分布式复制、Region、MVCC、事务、调度与 Coprocessor 都在 RocksDB 之外。复用 RocksDB 也不等于简单：TiKV 在其上做了 fork、四 CF 编码、Titan、独立 Raft Engine 一整套工程。性能强弱取决于 workload，不能一概而论。

2. **「OceanBase major merge 就是 RocksDB 的 compaction」**——错，语义根本不同。RocksDB compaction 是**每实例本地自治、按 key range/level 触发、与事务快照无关**的 LSM 文件层级整理与读写/空间放大权衡；OceanBase major freeze/major merge 是**集群级、RootService 选全局版本号、跨副本协调、产出全局一致基线 + checksum 校验**，并与全局快照、SCN、多版本清理、Tablet 生命周期相关。把二者等同会误判其抖动来源与控制面依赖。

3. **「OceanBase 自研引擎所以通用性能碾压 RocksDB」**——错（本章核心问题）。公开资料能支撑的是 OceanBase 为关系型数据库内核协同自研了 MemTable/SSTable/Macroblock/Microblock 与 merge 体系（全局快照/基线、schema/行列耦合、多版本 GC 与快照点绑定）；不能据此推出它在任意 KV workload 上天然胜出，且要付出复杂度与 major merge 抖动的代价。

4. **「raftdb 还在用」**——在 8.5 已默认被 Raft Engine 取代，`[raftdb]` 仅 legacy 保留；启用 Raft Engine 后 raftdb 配置被忽略。不应把 raftdb RocksDB 写成当前默认结论。

## 4.11 本章结论

1. TiDB/TiKV **复用 RocksDB**（fork `tikv/rust-rocksdb`），KV 数据主路径仍以 RocksDB 为本地持久化引擎，源码通过 `engine_traits`/`engine_rocks` 抽象与实现这层能力；数据走 default/lock/write 三 MVCC CF（raft CF 为历史遗留），关系语义在引擎**之上**编码（源码 `txn.rs` 可见 lock/value/write record 的写入位置）；raft 日志已从 RocksDB 独立为 **Raft Engine**（v5.4 引入、v6.1 默认、release-8.5 默认启用），Titan 自 v7.6.0 对新集群默认启用。

2. OceanBase **100% 自研** LSM-like 引擎（`src/storage/` 零 RocksDB），三级 SSTable（Mini/Minor/Major）+ 2MB Macroblock + 变长 Microblock，MemTable 用自研 `ObQueryEngine`（B-tree + hash）；这套结构与 Tablet、SCN、merge、校验等数据库内核状态紧密结合，关系语义在引擎**之内**。

3. 二者最根本的差异在控制路径：RocksDB compaction **本地自治、无全局快照**；OceanBase **major freeze 由 RootService 选全局版本号、跨副本协调出全局一致基线 + checksum**——这服务于分布式一致读与租户级合并。

4. 本章核心问题答案：OceanBase 自研引擎的优势应表述为**关系型数据库内核协同优势**（全局快照/基线、schema/行列耦合、多版本 GC 绑定快照点），**不是通用 KV 性能上的全面领先**；因为公开资料没有给出与 RocksDB 在相同 KV workload 下严格 apples-to-apples 的结论。代价是复杂度与 major merge 抖动、以及对 RootService 的调度依赖（推测）。

5. 通用 KV 引擎（RocksDB）与关系内核存储引擎（OceanBase）的分界，本质是「把关系语义放在引擎之上（复用）还是之内（自研）」的工程取舍；两种路线都有尾延迟风险（TiKV 常见于 RocksDB/Raft Engine/CF/Region 局部压力，OceanBase 常见于 freeze/merge/Tablet 层级/租户资源压力），并无绝对优劣（推测）。

6. Microblock 默认大小为 **16KB**（由 `BLOCK_SIZE` 表选项控制，源码常量 `OB_DEFAULT_SSTABLE_BLOCK_SIZE = 16 * 1024` 与官方文档 `BLOCK_SIZE = 16384` 一致）；但可调范围存在来源冲突——官方博客称 4-512KB / 8-512KB，而 DDL 解析器源码定义 `MIN_BLOCK_SIZE = 1024`（1KB）、`MAX_BLOCK_SIZE = 1048576`（1MB）。本章以源码 1KB-1MB 为准，博客所述数字标注为存疑（存在版本冲突）。

## 4.12 参考文献

[1] TiDB Storage / RocksDB Overview. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tidb-storage/.
 （支撑:§4.2 TiKV 通过 RocksDB 本地持久化、写入先走 Raft 接口、kvdb/raftdb 双实例、四 CF(raft/lock/default/write）职责、255B 分流阈值、共享 WAL。）
[2] TiKV Configuration File / RocksDB Config. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tikv-configuration-file/.
 （支撑:§4.2、§4.7、§4.9 中 storage.engine、per-CF 配置、rocksdb.defaultcf/lockcf/writecf 与 raftdb.defaultcf、lock-cf-c）
[3] Titan Overview. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/titan-overview/.
 （支撑:§4.2 Titan KV 分离、大 value 阈值、范围扫描代价、v7.6.0 默认启用。）
[4] TiDB 6.1.0 Release Notes. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/release-6.1.0/.
 （支撑:§4.2、§4.8 中「v6.1 起 Raft Engine 成为默认日志存储引擎」及官方给出的特定负载收益。）
[5] RocksDB Compaction Wiki. 官方文档[EB/OL]. https://github.com/facebook/rocksdb/wiki/Compaction.
 （支撑:§4.7、§4.10 中 leveled/tiered compaction 对读写/空间放大的基本权衡。）
[6] OceanBase Database Architecture / Tablet 内部结构. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001971022.
 （支撑:§4.3 OceanBase LSM-like、Tablet 内 MemTable/Mini/Minor/Major SSTable 四层结构、baseline vs incremental。）
[7] Minor compaction and major compaction. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001104016.
 （支撑:§4.3、§4.9 中 MemTable freeze、minor/major compaction 触发条件与 memstore_limit_percentage、freeze_trigger_percentag）
[8] Database object storage / block storage. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001106604.
 （支撑:§4.3 SSTable → macroblock → microblock 层级关系与典型大小（2MB / 约 16KB);microblock 可调范围与源码 DDL 解析器（1KB-1MB）冲突，见 §4.3 标注）
[9] Design A Storage Engine for Distributed Relational Database from Scratch. 官方博客；Medium 镜像可逐字核对[EB/OL]. https://en.oceanbase.com/blog/2597046272.
 （支撑:§4.3 为何不用 RocksDB(schema/强类型/大事务，逐字 "the schema is very simple..." 等）、macroblock 2MB("A macroblock is a 2MB fi）
[10] OceanBase: A 707 Million tpmC Distributed Relational Database System. 论文，PVLDB 2022,Vol.15 No.12[EB/OL]. https://vldb.org/pvldb/vol15/p3385-xu.pdf.
 （支撑:§4.3、§4.7 OceanBase 作为分布式关系内核，其存储与事务/SQL 引擎共同设计、基线/增量、major compaction 与全局快照版本的背景；该论文 707M tpmC 系 2020 年 v2.2）
[11] Raft Engine: a Log-Structured Embedded Storage Engine for Multi-Raft Logs in TiKV. 技术文章，InfoQ 版[EB/OL]. https://www.infoq.com/articles/raft-engine-tikv-database/.
 （支撑:§4.2、§4.8 raftdb 写放大问题、Raft Engine bitcask 式设计、性能数字（写 I/O 25-40%、CPU ~10-12%、尾延迟 ~20%、吞吐 ~4-5%)、版本归属（v5.4 引入 /）
[12] tikv/raft-engine 仓库（branch tikv-8.5)— README.md、src/. 源码[EB/OL]. https://github.com/tikv/raft-engine.
 （支撑:§4.2 Raft Engine 独立仓库、bitcask 式 memtable+log file 构造、purge_expired_files()、lz4 压缩、最小写放大（README 逐字）。purge_ex）
[13] tikv/tikv 仓库（release-8.5)— components/engine_traits/src/、components/engine_rocks/src/、components/raft_log_engine/src/、components/raftstore/src/store/config.rs、components/raftstore/src/store/fsm/store.rs、etc/config-template.toml、src/storage/mvcc/txn.rs、components/txn_types/src/types.rs. 源码[EB/OL]. https://github.com/tikv/tikv.
 （支撑:§4.2 cf_defs.rs 四 CF 常量、Engines<K,R> KV/Raft 拆分、txn.rs 三 CF 写入位置、SHORT_VALUE_MAX_LEN = 255、engine_roc）
[14] oceanbase/oceanbase 仓库（tag v4.2.5_CE)— src/storage/{memtable,blocksstable,compaction}/、ob_i_table.h、ob_macro_block.h、src/rootserver/freeze/、deps/oblib/src/lib/ob_define.h、src/sql/resolver/ddl/ob_ddl_resolver.{h,cpp}、src/share/inner_table/ob_inner_table_schema_def.py、src/observer/virtual_table/. 源码[EB/OL]. https://github.com/oceanbase/oceanbase.
 （支撑:§4.3 零 RocksDB、enum TableType(Mini/Minor/Major)、enum ObMergeType、ObMacroBlock 字段（checksum/Bloom/compacti）
[15] An Interpretation of the Source Code of OceanBase: Storage Engine. 第三方解读，阿里云社区[EB/OL]. https://www.alibabacloud.com/blog/599324.
 （支撑:辅助支撑 §4.3 源码级模块结构、microblock 默认 16KB；第三方可信度局限，microblock 默认值/范围最终以源码常量与 DDL 解析器为准。）
## 4.13 信息可信度自评

本章主干事实有**源码逐字支撑**:TiKV 四 CF 常量、CF 分流阈值（`SHORT_VALUE_MAX_LEN = 255`，边界 ≤255)、`txn.rs` 三 CF 写入位置、`Engines<K,R>` 结构、fork RocksDB 依赖、Raft Engine 独立仓库与适配层、`[raft-engine]` 配置默认、OceanBase 零 RocksDB、`enum TableType`、`enum ObMergeType`、`ObMacroBlock` 字段、`OB_DEFAULT_MACRO_BLOCK_SIZE = 2<<20`(2MB，属磁盘 SSTable 层）、`OB_DEFAULT_SSTABLE_BLOCK_SIZE = 16*1024`(16KB microblock)、DDL 解析器 BLOCK_SIZE 1KB-1MB 校验、`ObQueryEngine`/`ObKeyBtree`、RootService 驱动 major freeze——均来自 commit-pinned checkout(TiKV release-8.5;OceanBase v4.2.5_CE，列存 4.3.5，融合 4.4.x)。**官方文档**支撑 CF 职责、255B 阈值、Titan、Tablet 四层结构、freeze/compaction 触发参数、BLOCK_SIZE 默认 16384、v6.1 Raft Engine 默认化。**论文**(PVLDB 2022）支撑 OceanBase LSM 基线/增量/全局快照设计层，不用于任何绝对性能排名。**第三方解读**仅 2 条（阿里云源码解读、InfoQ Raft Engine 文章），可信度低于官方与源码。

**反查证更正与冲突**:(1)CF 分流——「write CF 存索引」经源码核实为机制误述，已更正为「write CF 存含 start_ts 的 MVCC Write 提交记录」(§4.2);(2)Macroblock 层级——`2<<20` 常量与版本均坐实，但属磁盘 SSTable 存储层而非 MemTable，已加澄清防止误置（§4.3);(3)Microblock 默认 16KB 经源码常量 + 官方文档坐实，可调范围源码 DDL 解析器为 1KB-1MB，与官博 4-512KB/8-512KB 冲突，以源码为准、博客数字标（存在版本冲突）(§4.3/§4.11);(4)Raft Engine 性能数字与版本归属经 InfoQ 版 + GitHub README + TiKV 6.1「What's New」旁证（写 I/O 25-40%、v5.4 引入 / v6.1 默认）;(5)`ObMergeType` 经 v4.2.5_CE checkout 与 develop 分支对照，develop 多出 `CONVERT_CO_MAJOR_MERGE` 等列存类型（版本差异，非冲突）。

**仍不确定项**:Raft Engine 内部 WAL/pipe log 磁盘二进制格式（独立仓库未单独 checkout，README 仅给逻辑写序);microblock 可调范围精确边界（源码 1KB-1MB 与官博 4-512KB 冲突，属来源冲突而非缺证）；具体 Prometheus metric 名、以及 OceanBase 合并相关内部视图的各列名（视图名本身已源码核实，见下）。**经源码核验确认**:(a)`purge_expired_files()` 的 TiKV 默认调用间隔已由 release-8.5 源码核实为 **10 秒**(`raft_engine_purge_interval = ReadableDuration::secs(10)`，模板 `# raft-engine-purge-interval = "10s"`,§4.2);(b)OceanBase 合并/compaction 内部视图名已由 v4.2.5_CE 源码 `ob_inner_table_schema_def.py`/`virtual_table/` 逐字核实（`__all_merge_info`、`GV$OB_MERGE_INFO`、`GV$OB_COMPACTION_PROGRESS`、`GV$OB_TABLET_COMPACTION_HISTORY`、`DBA_OB_MAJOR_COMPACTION` 等，§4.9)。

**版本核查备注**：本章版本严格按锁定基线——TiDB 主基准 8.5.x LTS（对照 7.5.x)、TiKV release-8.5;OceanBase TP LTS v4.2.5_CE（源码基线）、AP 4.3.5（列存，列存自 V4.3.0 引入）、融合 4.4.x。Raft Engine v6.1 默认、Titan v7.6.0 默认为官方历史归属，与锁定头不冲突；未发现与锁定头的版本冲突。

---


# 第 5 章 KV / LSM 与关系模型映射

## 5.1 本章核心问题

本章回答一个看似简单、却贯穿分布式数据库底层的核心问题：为什么底层看起来是 Key-Value(KV)或日志结构合并树(LSM-Tree)，对外却能表现为一个完整的关系型数据库？

要把这个问题拆清楚，必须区分三个常被混为一谈的层次：

1. **逻辑数据模型**：用户看到的是表、行、列、主键、二级索引，SQL 优化器、事务、索引、schema 都在这一层成立。
2. **逻辑存储抽象**：关系如何被编码成可定位、可排序、可扫描的内部键。在 TiDB 中，关系被压平成一个**有序的、不透明的 KV 命名空间**(key 单调有序，value 是字节串)；这里的"KV"是一种**逻辑抽象或访问接口**，不是物理结构。
3. **物理存储结构**：真正落盘的是 **LSM-Tree**(TiKV 用 RocksDB,OceanBase 用自研 SSTable 引擎)。LSM 决定了写放大、读放大、空间放大、compaction、write stall 等一系列物理代价。

本章的关键判断是：**KV 与 LSM 不是同一件事**。KV 是"关系如何被寻址"的问题，LSM 是"字节如何被持久化"的问题。TiDB 走的是典型的 **SQL-on-KV** 路线——TiDB Server 把 table、index、rowID、handle 编码到 TiKV 的有序 keyspace，再用 Region 与 TiKV 的 MVCC / Column Family / RocksDB 承载数据；而 OceanBase **并不是 SQL-on-KV**。OceanBase 官方同样把存储引擎描述为基于 LSM-tree，但其读写接口更贴近关系型内核：MemTable、SSTable、Tablet、宏块 / 微块直接承载行、主键范围、索引和多版本结果，而不是把 SQL 层外显为一个通用 KV 编码层。它是**关系原生(relational-native)的 LSM 引擎**(详见 §5.3)。这条分界线决定了两者在回表、范围扫描、列存、HTAP、运维诊断上的根本差异，因此是底层架构对比中不可回避的一节。

本章还会区分三种极易混淆的"日志 / 持久化结构"——**Raft / Paxos 共识 log、LSM 的 WAL、SSTable**。它们目的、生命周期、清理策略、故障恢复语义完全不同：共识 log 决定多数派提交与状态机重放，WAL 保护本地 MemTable 的崩溃恢复，SSTable 是排序稳定的数据文件。把三者混写，会直接导致测试指标、故障恢复判断和性能瓶颈定位都偏掉(§5.3、§5.7)。

本章聚焦"关系↔KV↔LSM 的映射"。存储引擎本身的更细节(LSM 内部层级、Titan 键值分离等)属第 4 章范围，MVCC 多版本可见性判定属第 11 章，事务 2PC 流程属第 10 章，本章只在映射必要处简短引用。之所以说这是底层关键，是因为"关系如何映射到字节"决定了上层几乎所有可观察行为：一次主键点查到底是一个 KV 读还是两个；一次范围扫描能不能转成一段连续磁盘读；一个二级索引查询要不要回表、回几次；一张宽表的大字段读取要不要多跳；批量导入会不会因为 compaction 落后而触发写抖动。这些行为并非 SQL 优化器层面能完全决定的，它们在"关系→KV→LSM"这条映射链路上被固化下来。理解了这条链路，才能解释为什么同样支持标准 SQL、同样声称分布式强一致的两套系统，在 HTAP、宽行、海量小行、写密集等场景下表现差异明显。 〔文献[12]〕

## 5.2 TiDB 的实现：关系 → 不透明 KV → LSM

TiDB 的 SQL-on-KV 路线由三段串联组成：**TiDB Server(SQL / 编码)→ tablecodec(关系→KV)→ TiKV(KV→MVCC→RocksDB / LSM)**。

### 5.2.1 关系 → KV 编码

TiDB 的关系到 KV 映射是显式设计。官方文档给出 table data 与 index data 两类 KV：行记录 key 形如 `tablePrefix{TableID}_recordPrefixSep{RowID}`，索引 key 形如 `tablePrefix{TableID}_indexPrefixSep{IndexID}_indexedColumnsValue`，非唯一索引还会把 `RowID` 编入 key 后缀。每张表分配集群内唯一的 `tableID`、表内唯一的 `indexID`、表内唯一的 `handle`(行标识)。

映射规则的"命名空间前缀"是**字面字节常量**，在 `pkg/tablecodec/tablecodec.go` 中真实定义：`tablePrefix = []byte{'t'}`、`recordPrefixSep = []byte("_r")`、`indexPrefixSep = []byte("_i")`、`metaPrefix = []byte{'m'}`；`idLen = 8`(int64)；`prefixLen = 1 + idLen + 2`，即 `t` + 8 字节 tableID + `_r` / `_i`；`RecordRowKeyLen = prefixLen + idLen`。官方文档在"Summary of mapping relationships"里把分隔符示意为 `r` / `i`，源码常量则是 `_r` / `_i`(下划线即分隔符)，二者描述粒度不同、并无实质冲突；本章写源码级细节时用 `_r` / `_i`，写文档示意时只把它视作可排序 keyspace 的结构表达。

因此：

- **行记录 key** = `t{tableID}_r{handle}`，value = 编码后的整行列数据；入口函数 `EncodeRowKey(tableID, encodedHandle)`、`EncodeRowKeyWithHandle(tableID, handle)`、解码 `DecodeRowKey`。
- **二级索引 key** = `t{tableID}_i{indexID}{indexedValues}`；唯一索引 value = handle，非唯一索引把 handle 拼进 key 尾部、value 为空；入口函数 `EncodeIndexSeekKey(tableID, idxID, encodedValue)`、`GenIndexKey(...)`。tableID / idxID 用 `codec.EncodeInt` 做 **memcomparable** 编码(保证字节序=数值序，从而范围扫描可直接转化为 KV 的有序 Scan)。

> 以上前缀常量、函数签名、`idLen=8` 均来自 commit-pinned checkout(pingcap/tidb · release-8.5 · `67b4876bd57b` · `pkg/tablecodec/tablecodec.go` L49–L63、L106–L114、L330、L715、L1106–L1134、L1225)；另据 v8.5.0 tag(`d13e52ed6e22`)亦确认 `_r` / `_i` 常量与 `EncodeRowKey` / `EncodeIndexSeekKey` 签名一致。调用链更深的细节仅确认到模块级。

这套映射的直接后果是：**SQL 算子被"翻译"成 KV 操作**：

**表 5-1　关系 → KV 编码**

| SQL 操作 | KV 操作 | 说明 |
|---|---|---|
| 主键点查(整数主键、聚簇) | 一次 `Get` `t{tid}_r{pk}` | 一个 KV pair 即整行 |
| 二级索引点查 + 回表 | `Get` / `Scan` 索引 key → 取 handle → 再 `Get` 行 key | 两次 KV 访问(回表) |
| 范围扫描 | `Scan` 一段连续 key 区间 | memcomparable 保证物理连续 |
| 全表扫描 | `Scan` `t{tid}_r` 前缀区间 | 跨多个 Region 并发 |

主键与 rowID 还有版本边界。早期直觉常写成"整数主键就是 RowID"，但自 v5.0 起还要区分 **聚簇 vs 非聚簇索引**，它决定了一行需要几个 KV pair。聚簇表(`tidb_enable_clustered_index` 默认 `ON`)用主键值直接做 handle，行数据 key 由用户主键数据构成，一行只需一个 KV pair，等值主键查询可省掉一次索引到行的回表；非聚簇表用隐藏列 `_tidb_rowid` 做 handle，主键退化成一条独立的唯一索引指向 `_tidb_rowid`，**一行至少两个 KV pair**。这不是 SQL 语义差异，而是物理 key 编码差异，且该选择建表后不可互转。这是行格式之外、影响写放大与回表代价的第一层设计。

### 5.2.2 行格式版本：v2(新行格式)vs v1

行 key 之外，行内列数据的编码同样存在版本差异，是本章必查项。系统变量 `tidb_row_format_version` 控制新写入数据的行格式，官方文档写明默认值为 2，自 v4.0 起新集群默认使用 version 2，而从早于 v4.0 升级而来的集群不会自动改写旧格式，修改该变量也只影响后续新写入数据。因此旧数据(v1)与新数据(v2)可能长期共存，本章不把 TiDB 行格式简化成单一格式。

新行格式(俗称 v2)版本号常量在 `pkg/util/rowcodec/common.go` 中真实定义为 `const CodecVer = 128`；判断函数 `rowcodec.IsNewFormat(b)` 即 `rowData[0] == CodecVer`。源码上 `EncodeRow` 在 rowcodec encoder 启用时走新版编码、否则走 `EncodeOldRow`(pingcap/tidb · release-8.5 · `67b4876bd57b`)。

版本号从 128 起步是有意为之：旧格式(v1)行首字节是 varint 编码的列 ID，取值不会落在 128 这一段，因此用 ≥128 作新格式标志可**兼容升级**(官方设计文档 `2018-07-19-row-format.md` 明确 "Its value starts from 128, so we can upgrade TiDB in a compatible way")。v2 物理布局(`row.go` 头注释)为：`VER | FLAGS | NOT_NULL_COL_CNT | NULL_COL_CNT | NOT_NULL_COL_IDS | NULL_COL_IDS | NOT_NULL_COL_OFFSETS | NOT_NULL_COL_DATA | CHECKSUM`。FLAGS 位 `rowFlagLarge=0x01`、`rowFlagChecksum=0x02`。

- **large 标志触发条件**：`max(colID) > 255` 或整行 value 总长 > 65535；触发后 colID 与 offset 由 1 字节扩为 4 字节。
- **v2 相对 v1 的工程收益**：列 ID 数组有序 → 取某列从 v1 的 O(N) 顺序解码降为 **O(logN) 二分查找**；NULL 列独立计数、不占数据区。

这一段映射完成后，TiDB Server 把"行 KV"提交给 TiKV；**到这里关系信息就被压平成不透明字节串**，TiKV 不再理解"列"，列的解释权完全留在 TiDB Server 的 `tablecodec` / `rowcodec`。

### 5.2.3 KV → MVCC → LSM

**表 5-2　TiKV 三列族(default / lock / write CF)解剖**

| 列族(CF) | 存什么 | key / value 编码要点 | 作用 |
|---|---|---|---|
| `CF_DEFAULT="default"` | 真实长值(> 255 字节才落此处) | key 用 `key.append_ts(start_ts)`；`put_value` / `delete_value` | 承载实际数据 value，宽行 / 大字段读取需经 write CF 指针再来取值 |
| `CF_LOCK="lock"` | primary lock / 二阶段锁信息 | key 不带时间戳(同一 key 同时至多一把锁)；`put_lock`→`Modify::Put(CF_LOCK,…)`、`unlock_key`→`Modify::Delete` | 记录 prewrite / 悲观锁，提交后释放 |
| `CF_WRITE="write"` | MVCC 版本元信息(提交记录)；短值 ≤ 255 内联 | key 通过 `key.append_ts(commit_ts)` 追加时间戳；`WriteType{Put,Delete,Lock,Rollback}`(flag `P/D/L/R`) | 记录 commit / write record；短值内联可一次访问即得数据 |

TiKV 用 **Percolator 模型 + RocksDB 三 Column Family** 把上面的逻辑 KV 落到物理 LSM。MVCC 模块(tikv/tikv · release-8.5 · `1f8a140b6d46` · `src/storage/mvcc/mod.rs`)对外导出 `Key, Lock, Mutation, Write, WriteType, TimeStamp, SHORT_VALUE_MAX_LEN` 等类型；`txn.rs` 显示事务写入把 lock 放到 `CF_LOCK`、把实际 value 放到 `CF_DEFAULT`、把 commit / write record 放到 `CF_WRITE`，`reader.rs` 维护 data / lock / write 三类 cursor 并解析 `short_value`。三个 CF(`components/engine_traits/src/cf_defs.rs`)为：

- `CF_LOCK="lock"`：存 primary lock / 二阶段锁信息，**key 不带时间戳**(同一 key 同时至多一把锁)；写路径 `put_lock`→`Modify::Put(CF_LOCK,...)`、`unlock_key`→`Modify::Delete(CF_LOCK,...)`(`txn.rs` L122–L161)。
- `CF_WRITE="write"`：存 MVCC 版本元信息(提交记录),key 通过 `key.append_ts(commit_ts)` 追加时间戳；`put_write` / `delete_write`(L186–L193)。版本类型枚举 `WriteType{Put,Delete,Lock,Rollback}`(字面 flag `P/D/L/R`，`components/txn_types/src/write.rs` L16–L28)。
- `CF_DEFAULT="default"`：存真实长值，key 用 `key.append_ts(start_ts)`；`put_value` / `delete_value`(L175–L181)。

**短值内联**是回表 / 读放大的关键优化：常量 `SHORT_VALUE_MAX_LEN=255`(`components/txn_types/src/types.rs` L22)。当 value ≤ 255 字节，值直接内联进 write CF 记录，读取时一次访问 write CF 即得数据；> 255 字节才落 default CF，读取需"write CF 拿到指针 → default CF 取值"两步。官方 RocksDB 文档亦印证：write CF 存"真实写入数据与 MVCC 元数据"，default CF 存"长于 255 字节的数据"。

物理上，这些 KV 进入 RocksDB(LSM)：先写 WAL，再进内存 MemTable(SkipList)，满了 flush 成 SSTable，默认最多 6 层，compaction 在层间合并。Percolator 的两阶段(详见第 10 章)与 MVCC 可见性(详见第 11 章)在此只作映射说明：`(key, commit_ts)→write_info` 落 write CF、`(key, start_ts)→value` 落 default CF，使**同一 key 的多版本在 LSM 中物理相邻、新版本在前**。 〔文献[1-4,8-10,14-16]〕

> Raft log 走**独立**通道。官方 TiKV 文档说明底层有 KV RocksDB 与 Raft RocksDB 两类实例：default / write / lock CF 承载用户数据、MVCC 与锁信息，RaftDB 承载 Raft log。在日志存储引擎层面，Raft Engine 自 v5.4 起作为可选引入，**自 v6.1.0 起成为默认**(取代早期 raftdb / RocksDB)，官方《TiDB 6.1.0 Release Notes》原文为 "Since v6.1.0, TiDB uses Raft Engine as the default storage engine for logs"，且 "Compared with RocksDB, Raft Engine can reduce TiKV I/O write traffic by up to 40%"。这佐证了本章要点——**共识 log、LSM WAL、SSTable 是三套不同结构**:Raft log 用于副本一致性、apply 后即可截断；LSM WAL 用于单机 crash 恢复 MemTable、flush 后即失效；SSTable 是持久基线、靠 compaction 清理(§5.3)。

## 5.3 OceanBase 的实现：关系原生 SSTable，而非 SQL-on-KV

**表 5-3　多种日志 / 持久化结构对照**

| 结构 | 角色 | 落盘 / 持久化形式 | 所属系统 |
|---|---|---|---|
| Raft log | 副本一致性共识日志，apply 后即可截断 | 独立通道，早期由 Raft RocksDB / raftdb 承载，v6.1.0 起默认 Raft Engine | TiKV |
| LSM WAL | 单机 crash 恢复 MemTable，flush 后即失效 | 写入 RocksDB(LSM)前先写 WAL | TiKV |
| MemTable | 承接新写入的内存有序结构，满后 flush | 内存 SkipList，未落盘 | TiKV |
| SSTable | 持久基线、排序稳定的数据文件，靠 compaction 清理 | MemTable flush 落盘，默认最多 6 层 | TiKV |
| PALF | Paxos-backed 共识日志，与 LSM 数据物理分离 | 独立的 Paxos-backed Append-only Log File system | OceanBase |
| MemTable | 增量数据(行操作链)，DML 先写入 | 内存可读写结构，dump / minor compaction 后转储 | OceanBase |
| SSTable | 静态基线数据，生成后不再修改 | 只读 SSTable，宏块 2MB + 微块 PAX 编码 | OceanBase |

与 TiDB 的 SQL-on-KV 路线相对，OceanBase **不是 SQL-on-KV**。其存储引擎(`src/storage/blocksstable/`)直接以表 / 行 / 列 / rowkey / 索引块的概念组织数据，**没有 TiDB 那种把关系压平成不透明 KV 的 `tablecodec` 中间层**。这是本章最重要的对照结论(必查项)，下面从结构、编码、compaction 三条线展开。

### 5.3.1 LSM 双层 + 宏块 / 微块结构

OceanBase 的公开资料明确使用 LSM-tree 表述：数据分为静态 baseline data(基线数据，持久在只读 SSTable、生成后不再修改)和动态 incremental data(增量数据，在可读写 MemTable)；DML 先写 MemTable，查询时同时读取 MemTable 与 SSTable，在内存中**融合(merge)** 后返回 SQL 层。这可以称为基于 LSM-tree 或 LSM-like，但不应等同于 TiDB 的 SQL-on-KV——源码中 `ObMemtable::get/scan/set/lock` 使用 `ObDatumRowkey`、`ObDatumRange` 与行 / 列描述，`ObSSTable::get/scan/multi_get/multi_scan` 也直接暴露行键与范围访问接口。

SSTable 的物理粒度由官方文档与源码一致确认：OceanBase 的 SSTable 由一个或多个 macroblock 组成，macroblock 再由一个或多个 microblock 组成，microblock 内含一行或多行。

- **宏块(macroblock)**：固定 `OB_DEFAULT_MACRO_BLOCK_SIZE = 2<<20`，即 **2MB**(`deps/oblib/src/lib/ob_define.h` L1817；`ob_server.cpp` L2281 设 `default_block_size_`)。宏块是 compaction 写入与空间复用的基本单位，也是写 I/O 的基本单位；每个宏块对应主键的一段范围。源码 `src/storage/blocksstable/ob_macro_block.h` 的 `ObMacroBlock` 暴露 `write_micro_block`、`write_index_micro_block`、`flush`、`get_micro_block_count`、`get_row_store_type`；`ob_sstable.h` 暴露 `scan_macro_block` 与 `scan_micro_block`，与文档关系一致。
- **微块(microblock)**：宏块内再分多个变长微块(经压缩后变长，默认约 16KB，且文档注明该大小指压缩前大小)，是**读 I/O 的最小单位**。其可调范围由建表 DDL 校验强制：`ob_ddl_resolver.h` 中 `MIN_BLOCK_SIZE = 1024`(=1KB)、`MAX_BLOCK_SIZE = 1048576`(=1MB)，`ob_ddl_resolver.cpp`(约 L1471)校验越界即报 "block size should between 1024 and 1048576"(v4.2.5_CE / v4.3.5_CE / v4.4.2_CE 三版逐字一致)。微块头有版本与魔数常量：`MICRO_BLOCK_HEADER_MAGIC=1005`、当前 `MICRO_BLOCK_HEADER_VERSION=3`(`ob_block_sstable_struct.h` L47–L59)。

关键差异：`blocksstable` 目录里的文件名本身就是关系语义——`ob_row_reader/writer`、`ob_datum_row`、`ob_datum_rowkey`、`ob_micro_block_row_scanner/getter`、`ob_index_block_builder`。存储层直接"理解"行、rowkey、索引块。OceanBase 的 MemTable 更像关系型内核的增量行存储：官方资料描述 MemTable 中有带 MVCC 的 B-tree-like 结构，按主键有序支持 table scan，另有内存 Hash index 加速单行 get；源码 `src/storage/memtable/ob_memtable.h` 可确认存在 `set`、`lock`、`get`、`scan`、`multi_get`、`multi_scan`、`replay_row` 等接口。其内部双索引由 `ObQueryEngine`(`src/storage/memtable/mvcc/ob_query_engine.h`)承载、确为 **Keybtree + Hash 双索引并存**：成员 `keybtree_`(类型 `keybtree::ObKeyBtree<ObStoreRowkeyWrapper, ObMvccRow *>`)与 `keyhash_`(类型 `ObMtHash`)同时存在，二者都把 rowkey 映射到同一 `ObMvccRow *` 行版本链；写入时 `keyhash_.insert` 与 `keybtree_.insert` 双写，单行 `get` 走 `keyhash_.get(...)`(哈希定位)、范围 `scan` 走 `keybtree_.set_key_range(...)`(B-tree 有序迭代)——印证"内存 Hash index 加速单行 get、B-tree-like 结构支持有序 table scan"的分工(oceanbase v4.2.5_CE · `e7c676806fda` · `ob_query_engine.h` L34/L68/L76/L251/L253、`ob_query_engine.cpp` 的 `keyhash_.get` / `keybtree_.set_key_range`)。

换句话说，TiKV 拿到的是一串 `t..._r...→<bytes>`，它不知道这串字节里有几列、列是什么类型，列的解释权完全留在 TiDB Server 的 `tablecodec` / `rowcodec`。而 OceanBase 的存储层拿到的就是带 schema 的行——它知道第几列是整数、第几列是字符串，因而可以在微块层面直接对某一列做字典或差值编码、对某一列建跳数索引(skip index)，并在 major compaction 时把基线数据直接重排成列存。这个能力差异并非源于实现优劣，而是**架构分层位置不同带来的必然结果**：把映射层下沉进存储引擎，换来的是列感知与一体化，代价是存储引擎本身更厚重、与 SQL 层耦合更深；把映射层留在 SQL 层、存储只做 KV，换来的是存储层可独立演进(TiKV 可单独作为 KV 库使用)，但纯列式分析必须再外挂一个理解列的引擎(TiFlash)。

### 5.3.2 PAX 行列混合编码

OceanBase 微块采用 **PAX(Partition Attributes Cross)** 风格：微块内一组行**按列做编码**，利用同列数据的局部性，兼顾行访问与列局部性。行存类型枚举(`deps/oblib/src/common/ob_store_format.h`，v4.2.5_CE · `e7c676806fda`)为：

```cpp
enum ObRowStoreType : uint8_t {
 FLAT_ROW_STORE = 0, // 平铺行存
 ENCODING_ROW_STORE = 1, // 行存 + 列内编码(PAX)
 SELECTIVE_ENCODING_ROW_STORE = 2,
 MAX_ROW_STORE
};
```

`encoding/` 子目录含真实列编码器：`ob_dict_encoder/decoder`(字典)、`ob_const_encoder`(常量)、`ob_column_equal_encoder`、`ob_integer_base_diff_encoder`(差值)、`ob_hex_string_encoder`、`ob_inter_column_substring_encoder`(列间子串)等——即"同微块内按列做字典 / 常量 / 差值"的 PAX 混合编码。第二层再叠加通用压缩(zstd / lz4 等)。

**版本演进对照**(高风险项，严格按锁定版本归属)：TP 基线 **4.2.5** 行存类型枚举到 `SELECTIVE_ENCODING_ROW_STORE=2`，**无 `CS_ENCODING`**(经 v4.2.5_CE tag 源码逐字核实)。`CS_ENCODING_ROW_STORE=3`(列式编码引擎)并非 4.3.5 新增——它自 **4.3.0**(官方列存引擎首次引入的版本，4.3.0_CE_BETA tag 即已存在)就已加入，到锁定基准 **4.3.5_CE**(`b28b9bb12f3b`)已是既有枚举项；同期新增 `src/storage/blocksstable/cs_encoding/` 目录(`ob_cs_encoding_util`、`ob_dict_column_encoder/decoder`、`ob_cs_micro_block_transformer` 等)。官方资料明确列存引擎 / 列式编码自 **V4.3.0** 起引入(4.3.3 为首个实时分析 GA),4.3 起基线数据可按行存、列存、或行列冗余三种模式存储，增量数据保持行存；v4.3.3 文档进一步描述 hybrid rowstore-columnstore 表，优化器可为 TP 查询选择 row-based storage、为 AP 查询选择 columnar storage。这印证 OceanBase 的列存是在**同一关系原生引擎内**演进出来的，而非外挂一个独立列存(对照 TiDB 用独立的 TiFlash，详见第 16 章)。需注意：`CS_ENCODING_ROW_STORE` 本质是 PAX 微块的行存编码格式枚举，不应直接等同于"独立列存"特性的有无；这类 PAX / hybrid row-column 表述也只能用于 OceanBase 存储层，不能反推 TiKV 的 default / write / lock CF 具有同样的微块结构。

### 5.3.3 compaction:dump / minor / major

- **dump(转储)/ minor compaction**:MemTable 达阈值后冻结(Frozen)、新开 Active MemTable，Frozen 转储成 mini / minor SSTable；类似 RocksDB / HBase 的 minor compaction，缓解内存压力，**默认复用未脏宏块**以降低写 / 空间放大。
- **major compaction(每日合并)**：读全量基线 + 增量，合并写回新基线；开销最大，VLDB 2022 论文称之为 "asymmetric read/write + daily incremental major compaction" 的设计。

**OBKV 边界提醒(必查)**:OceanBase 有 OBKV(OBKV-HBase / OBKV-Table / OBKV-Redis)接入路径，但这**只是 observer 层的 API 接入，不是底层存储模型**。官方 OBKV-Table 文档说明它是 table model-based API，可与 SQL 语句无缝集成，支持通过 primary key 读写，并支持服务端二级索引；其 KV model 本质是只有 K / V 两列、并按 K 或 K 前缀分区的简化关系表。HBase 兼容代码位于 `src/observer/table/`(如 `ob_htable_filter_operator`)，底层仍是 `src/storage/blocksstable` 的关系原生 SSTable。因此**不应把 OBKV 误判为 OceanBase 的"KV 底层"**——它把 HBase 的行 / cell 或 K / V 映射成 OceanBase 表的行，绕过部分 SQL 解析直连存储，本质是"换一种 API 访问同一个关系引擎"(进一步反例见 §5.8)。 〔文献[11,13,17]〕

## 5.4 核心差异对比

两套系统的分界线不在"是否用 LSM",而在"映射层放在哪一层"。表 5-4 按维度分别陈述这条分界线的具体落点——从关系到存储的中间层、物理结构、MVCC 落地，到 KV API 的暴露方式。

**表 5-4　KV/LSM 与关系模型映射:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB(SQL-on-KV) | OceanBase(关系原生 SSTable) | 影响 |
|---|---|---|---|
| 关系→存储中间层 | 有：`pkg/tablecodec` 把行 / 索引压平成不透明 KV | 无：`blocksstable` 直接理解行 / 列 / rowkey / 索引块 | OB 省去映射层，列存 / 编码更自然；TiDB 更易从 keyspace 解释路由与热点 |
| 物理结构 | RocksDB(LSM)+ 三 CF(lock / write / default)| 自研 SSTable(宏块 2MB + 变长微块 + PAX)| 写 / 读放大特征不同；OB 微块按列编码 |
| MVCC 落地 | key `append_ts`，版本物理相邻于 write / default CF | MemTable 行操作链 + 基线 / 增量融合 | TiDB 版本=多 KV;OB 版本=行链 |
| 主键 / 行内编码 | 聚簇 / 非聚簇影响回表；行格式 v2(`CodecVer=128`),large 标志 | `ObRowStoreType`：FLAT / ENCODING / SELECTIVE(TP 4.2.5)、+CS_ENCODING(自 4.3.0 引入)| OB 列内字典 / 差值编码内建；TiDB 行存 + 独立 TiFlash 列存 |
| 回表(二级索引)| 索引 KV → handle → 行 KV(两次访问)，短值 ≤255 内联 write CF 优化 | 索引块同引擎内，行级语义保留 | 两者都有回表；TiDB 回表是跨 KV pair |
| KV API 暴露 | TiKV 本身就是 KV 数据库(可直接用)| OBKV 仅 observer 层接入，K / V 两列简化表，非存储模型 | 不可把 OBKV 当 OB 的 KV 底层 |
| 共识 log 通道 | 独立：v6.1+ Raft Engine | 独立：PALF(Paxos-backed Append-only Log File system)| log 与 LSM 数据物理分离(详见第 3 章)|
| 常见误判 | "TiDB 是 KV 所以不是关系型" | "OceanBase 有 OBKV 所以底层是 KV" | 两者都错：一个是 SQL-on-KV 映射，一个是关系型内核提供 KV API |

> 表注：OceanBase 宏块 2MB、行存枚举、TiDB 前缀常量 / `CodecVer=128` / `SHORT_VALUE_MAX_LEN=255` 均来自 commit-pinned 源码核对(§5.2 / §5.3 引用)，非二手转述。

## 5.5 正常路径图(读 / 写 + SQL→KV 翻译)

![[f5_1.svg]]

**图 5-1　KV/LSM 与关系模型映射正常读写/调度路径**

正常写路径：TiDB 端"SQL→tablecodec 编码→TiKV MVCC 三 CF→RocksDB WAL / MemTable / SSTable"，Raft log 走 Raft Engine；OceanBase 端"SQL→关系原生 blocksstable→MemTable→(dump)SSTable"，共识走 PALF。正常读路径：TiDB 主键点查=一次 KV `Get`，二级索引查询=索引 Scan，覆盖索引直接返回、否则按 handle 回表；OceanBase 读=按表 / 分区 / Tablet 与主键范围定位后，基线 SSTable + 增量 MemTable 在 row cache / block cache / microblock 解码后融合返回。若是 hybrid rowstore-columnstore 表，优化器还可能按 TP / AP 查询形态选择 row-based 或 columnar storage。

## 5.6 故障 / 异常路径图

![[f5_2.svg]]

**图 5-2　KV/LSM 与关系模型映射故障/异常路径**

四类异常的重点是**先判定卡在哪里**：

- **A. Leader / 副本切换 + 路由失效**：Region leader 迁移或 split / merge 会让客户端缓存的路由失效，之后向 PD(TiDB)或 RootService(OceanBase)刷新分片位置再 backoff 重试(详见第 2、3 章)。
- **B. compaction 抖动 / write stall**——LSM 共性。TiKV 用 scheduler 层流控替代 RocksDB 原生 write stall(当 `compaction pending bytes` 超 `soft / hard-pending-compaction-bytes-limit`，线性提高 discard ratio)，L0 文件过多、MemTable flush 跟不上即触发；OceanBase 用分级转储 + 资源隔离避免转储拖垮读。
- **C. 节点 crash**——MemTable 内存数据靠 WAL(单机)/ 共识 log(副本)重放恢复，这正是三种 log 语义不同之处。
- **D. 事务未提交**——lock CF 中的 prewrite lock 或悲观锁长时间存在会阻塞读写，需 resolve lock / rollback / commit 恢复；写入变成 committed write record 后锁才释放(详见第 10 章)。

OceanBase 中对应地要区分 LS / PALF 层复制与日志恢复、MemTable freeze / merge 卡顿、SSTable major / minor compaction 消耗、宏块 / 微块读放大或缓存命中不足。本章只强调：**compaction / write stall 是 KV / LSM 映射带来的物理代价，而非逻辑模型的缺陷**。

## 5.7 性能、可靠性、运维影响

**写放大 / 读放大 / 空间放大**是 LSM 的固有三角，两库都受其约束但缓解手段不同：

**表 5-5　性能、可靠性、运维影响**

| 影响维度 | TiDB / TiKV | OceanBase |
|---|---|---|
| 写放大来源 | RocksDB 层间 compaction；早期 raftdb 也放大，v6.1 默认 Raft Engine 官方称降写 I/O 最高约 40% | major compaction 重写基线；mini / minor 缓冲 |
| 读放大来源 | 多层 SSTable + 回表(>255 字节回 default CF)| 基线 + 增量融合；微块按列解码 |
| 空间放大 | 多版本 + 未 compact 的旧 SSTable；靠 GC 清理旧版本 | 未脏宏块复用降低空间放大；每日合并回收 |
| write stall 缓解 | scheduler 层流控(discard ratio)| 分级转储 + 资源隔离 |
| 列存收益 | 行存 TiKV + 独立 TiFlash(详见第 16 章)| 同引擎内 PAX / CS_ENCODING 列编码 |
| 可靠性边界 | Raft log 复制一致性、RocksDB WAL 本地崩溃恢复、MVCC CF 事务可见性 | PALF / 事务日志与 MemTable / SSTable 路径分离；major compaction 可做一致性校验但不替代日志恢复 |

- **延迟**：正常点查均为内存 / 单层命中，亚毫秒级；尾延迟主要来自 compaction 抖动与 Leader 切换窗口。TiDB 的二级索引查询因回表存在两次 KV 访问，P99 天然高于聚簇主键点查；OceanBase 的读延迟则受"基线 SSTable + 增量 MemTable 融合"层数影响，增量越厚(临近 major compaction)读路径越长，MemTable Hash index 与 row cache 则有利于单行查询。
- **吞吐**：LSM 把随机写转顺序写，写吞吐因此较高；但 compaction 占用 I/O，峰值写入易触发流控。原始 LSM-tree 论文强调它通过延迟和批量合并降低高频 insert / delete 的索引维护成本，代价是查询可能要同时访问内存组件与磁盘组件。因此 LSM 的写优势本质是"延迟支付"——写入时只追加，合并 / 重排的代价被推迟到 compaction，吞吐曲线并非平直，而是带周期性凹陷。也正因如此，"LSM 写入快"必须带上条件：若 compaction 追不上，写入会被 stall；若层级文件或历史版本太多，读放大与空间放大又会反过来吞掉收益。
- **可用性 / 扩展性**：KV / LSM 单元(Region / Tablet)是扩展与故障恢复的最小粒度(详见第 1 章)。映射层决定了这个单元里装的是"不透明 KV 区间"(TiDB)还是"带 schema 的行 / 列块"(OceanBase)，这进而影响迁移、快照、增量同步的实现方式。
- **运维复杂度**：两者都需关注 compaction 进度与 pending bytes，但侧重点不同。TiKV 的 compaction 更连续后台化、靠 scheduler 流控削峰，调参面常落在 block cache、CF 写缓冲、compaction、Region 热点，需盯 `compaction pending bytes` 防止流控放大写延迟；OceanBase 的每日合并是显式可调度事件(可设触发时间窗、避开业务高峰)，调参面常落在 MemTable freeze、minor / major compaction、宏块微块缓存、租户资源隔离，需盯 major compaction 的进度视图防止"合并不掉"。两种运维心智模型不同，但都绕不开"LSM 必须 compact、compact 必占资源"这一物理事实。 〔文献[5]〕

## 5.8 反例与代价

- **热点递增写**：TiDB 的表数据与索引数据共享前缀、在 keyspace 中相邻，若主键或索引值单调递增，写入可能集中在少数 Region 或 LSM 层级上——此时"关系到 KV 编码很清楚"反而暴露热点，需要 `AUTO_RANDOM`、`SHARD_ROW_ID_BITS`、预切 Region、业务侧打散等策略缓解。其中预切的 Region 数量由 `PRE_SPLIT_REGIONS` 建表选项控制，实际切分份数为 **2^(PRE_SPLIT_REGIONS)**。
- **小值海量行场景下，非聚簇表的双 KV pair 代价**：每行至少两个 KV pair + 回表，写放大与点查放大都翻倍；TiDB 默认聚簇索引正是为缓解此问题。
- **大 value(>255 字节)回表代价**：短值内联只覆盖 ≤255 字节，且二级索引主要占 write CF 空间；宽行 / 大字段读取需 write CF → default CF 两跳，读放大固定增加。这对点查、CDC old value、备份和 compaction 都有影响——若把"行就是一个 KV value"理解得太字面，会忽略短值、索引、MVCC write record 与 default CF 的分离。
- **把 OceanBase 简化成 KV**：OBKV-Table 可跳过部分复杂 SQL 解析路径，但官方文档同时说明它功能不等同 SQL mode——聚合、排序等复杂能力仍应使用 SQL mode，KV model 本质是只有 K / V 两列的简化关系表。所以 OBKV 适合低延迟主键访问或宽表 / NoSQL 风格业务，却不能拿来定义 OceanBase 整个底层存储模型(见 §5.3)。
- **compaction / merge 的尾延迟是 LSM 通病**：重写入 / 批量导入后 pending bytes 堆积可触发流控甚至 write stall，表现为前台写延迟尖刺；OceanBase major compaction(每日合并)读全量基线，大库期间 I/O 压力显著，需调度到低峰并配资源隔离。TiKV 侧表现为 MemTable flush、L0 文件、compaction 线程、block cache 与 CF 写缓冲；OceanBase 侧表现为 minor freeze、minor / major compaction、宏块微块读写。二者都不是"追加写就完事"。相对 B-Tree 引擎，这种周期性抖动正是 LSM 的代价：B-Tree 写吞吐更低，但没有这类周期性凹陷、曲线更平稳。
- **范围扫描跨 Region / Tablet 的代价**：memcomparable 保证了"同一 key 区间物理有序"，但一段大范围扫描往往跨越多个 Region(TiDB)或 Tablet(OceanBase)，需要把扫描拆分、并发下发、再归并。TiDB 通过 Coprocessor 把过滤 / 聚合下推到各 TiKV 节点、以列式 Chunk(类 Arrow)回传后归并，减少网络传输；但跨大量 Region 的扫描仍受调度与 backoff 影响。这说明"KV 有序"只解决了单区间寻址问题，跨区间的并发与一致性快照仍需上层协调。
- **TiDB 不透明 KV 的代价**：存储层不理解列，纯列式分析必须外挂 TiFlash(列存)；OceanBase 在同引擎做列存，省一套异构同步，但引擎本身更复杂。相对纯 B-Tree / 堆表引擎，KV / LSM 路线牺牲了稳定的读延迟与无抖动的写曲线，换来更高的顺序写吞吐、更友好的压缩比与更简单的水平扩展单元。在以点查为主、对尾延迟敏感、写入平稳的 OLTP 场景，B-Tree 仍有其优势；但在写密集、需弹性扩展、数据量大且需压缩的分布式场景，LSM 的取舍通常更占优——这也是两库都选择 LSM 作为物理底座的根本原因。**两者是工程取舍，不存在"谁更简单 / 更先进"**。

## 5.9 测试开发视角的验证点

**可测功能场景**：聚簇 vs 非聚簇表的 KV pair 数差异(可用 `tikv-ctl` 扫描 key 数验证，或用执行计划判断是否回表)；二级索引回表正确性(构造覆盖索引与非覆盖索引场景，看是否走 table / index range scan)；行格式 v2 large 标志在 `max(colID)>255` 或行长 >65535 时的切换；短值 255 字节边界(write CF 内联 vs default CF 落值)；OceanBase 不同 `ObRowStoreType` 的读写正确性。如需观察具体 key，可用官方 SQL 函数 `TIDB_DECODE_KEY()`：官方文档说明它把一条 TiDB 编码 key 解码为 JSON，解码失败时原样返回参数；成功时 JSON 含 `table_id`、非聚簇表的隐藏行号 `_tidb_rowid`，聚簇表则在 `handle` 下给出主键列名与值。更细的诊断接口字段以运行版本为准。

**可注入失效模式**：路由层注入 Region leader 迁移、split / merge 后的 stale route；事务层注入 primary lock 未提交、二级锁残留、悲观锁冲突；LSM 层注入 compaction backlog、L0 文件增加、block cache 压力、磁盘 I/O 饱和，观察流控；Leader 切换窗口(详见第 3 章)；OceanBase 侧注入 MemTable freeze、minor / major compaction、宏块缓存未命中、租户内存压力。

**压测指标不要只看 TPS**：至少要看 p95 / p99 延迟、回表次数、扫描 key / row 数、写放大趋势、磁盘写带宽、compaction / merge 耗时、锁等待、缓存命中、空间增长速度。可查证的观测类别如下，**但不逐项写具体 Prometheus metric 名，避免跨版本编造**：

- TiKV(Prometheus / Grafana TiKV-Details)：`compaction pending bytes`(待 compact 数据量)、`write stall duration`(正常应为 0)；配置项 `soft-pending-compaction-bytes-limit`、`hard-pending-compaction-bytes-limit`；以及 RocksDB engine size / flow、scheduler scan details、lock / write / default CF 相关观测项。诊断命令：`tikv-ctl`(扫描 key、查看 region / CF)。
- OceanBase 内部视图：视图名与字段在源码 `src/share/inner_table/ob_inner_table_schema_def.py` 的视图定义中可逐字核对(oceanbase v4.2.5_CE · `e7c676806fda`)——`CDB_OB_MAJOR_COMPACTION`(sys 租户全局合并信息;视图列 `TENANT_ID / FROZEN_SCN / FROZEN_TIME / GLOBAL_BROADCAST_SCN / LAST_SCN / LAST_FINISH_TIME / START_TIME / STATUS / IS_ERROR / IS_SUSPENDED / INFO`,底层取自 `__ALL_VIRTUAL_MERGE_INFO`)、`GV$OB_COMPACTION_PROGRESS`(节点级合并进度;视图列含 `SVR_IP / SVR_PORT / TENANT_ID / TYPE / ZONE / COMPACTION_SCN / STATUS / TOTAL_TABLET_COUNT / UNFINISHED_TABLET_COUNT / DATA_SIZE / UNFINISHED_DATA_SIZE / COMPRESSION_RATIO / START_TIME / ESTIMATED_FINISH_TIME / COMMENTS`,`STATUS` 取值如 `NODE_RUNNING`)、`GV$OB_TABLET_COMPACTION_PROGRESS`(tablet 级;视图列含 `... LS_ID / TABLET_ID / COMPACTION_SCN / TASK_ID / STATUS / DATA_SIZE / UNFINISHED_DATA_SIZE / PROGRESSIVE_COMPACTION_ROUND / CREATE_TIME / START_TIME / ESTIMATED_FINISH_TIME`)、`GV$OB_COMPACTION_DIAGNOSE_INFO`(异常诊断;含 `DIAGNOSE_INFO` 列)。仅对象存储延迟、缓存命中率等具体运维阈值非视图 schema 可定，**需进一步查证**。
- RocksDB / TiKV 层流控阈值默认值可核对：`soft-pending-compaction-bytes-limit` 默认 **192GiB**(`storage.flow-control` 与各 `rocksdb.{default,write,lock,raft}cf` 均为 192GiB),`hard-pending-compaction-bytes-limit` 默认 **256GiB**(各 RocksDB CF 级;启用 flow-control 时 `storage.flow-control` 的 hard 限为 1024GiB,并覆盖 CF 级设置);源码侧亦见 CF 默认 `ReadableSize::gb(192)`(tikv/tikv · release-8.5 · `1f8a140b6d46` · `src/config/mod.rs`)。OceanBase write stall(写入限流)具体触发阈值参数在已 clone 的 v4.2.5_CE 源码中参数名经脱敏、且 en.oceanbase.com 文档被 WAF / 限流挡住,**具体数值需进一步查证**。

**回归测试要覆盖版本差异**:TiDB v8.5.x 中行格式 version 2 是新写入默认，但升级集群可继续写 version 1,clustered index 自 v5.0 起影响主键物理组织；OceanBase 4.2 / 4.3 / 4.4 文档在列存与 hybrid row-column 表述上有版本差异，本章只把"基于 LSM-tree 的 MemTable / SSTable、macroblock / microblock"作为稳定内核事实，不把 4.4.x 新特性扩展到所有版本(CS_ENCODING 自 4.3.0 引入见 §5.3)。 〔文献[6-7]〕

## 5.10 容易误解点

1. **"TiKV 是 KV，所以 TiDB 底层就是 KV 数据库；OceanBase 也一样"** —— 错。TiDB 确为 SQL-on-KV(TiKV 本身是可独立使用的 KV 库，关系模型显式编码到有序 KV，SQL 优化器 / 事务 / 索引 / schema 仍在关系层成立)，但 OceanBase 是**关系原生**存储，存储层直接理解行 / 列 / rowkey；**OBKV 只是接入 API，不是底层存储模型**(见 §5.3、§5.8)。把两者都套用 SQL-on-KV 是本章最常见误判。
2. **"KV 就是 LSM"** —— 错。KV 是逻辑寻址抽象(key 有序、value 字节串)，LSM 是物理持久化结构(MemTable / SSTable / compaction)。TiKV 的"KV"落到 RocksDB 这个"LSM"上；两层可分别替换。
3. **"Raft log = WAL = SSTable 都是日志"** —— 错。共识 log 保副本一致(决定多数派提交与状态机重放，apply 后截断)、LSM WAL 保单机 crash 恢复(flush 后失效)、SSTable 是排序稳定的持久基线(靠 compaction / GC 清理)，三者目的、生命周期、清理与恢复语义都不同。
4. **"行格式版本号从 128 是随便取的"** —— 错。是为兼容旧 varint 列 ID 编码而刻意选 ≥128，使新旧格式可共存升级。

## 5.11 本章结论

1. **KV 是逻辑抽象、LSM 是物理结构，二者必须分开看**：关系如何被寻址(KV)与字节如何被持久化(LSM)是两个独立问题。
2. **TiDB 走 SQL-on-KV**：`pkg/tablecodec` 把行 / 索引压平成不透明 KV(行 key=`t{tid}_r{handle}`、索引 key=`t{tid}_i{iid}{vals}`、行格式 `CodecVer=128`)，Get / Scan / 回表都可从 key range 推导；TiKV 再以 Percolator 三 CF + `append_ts` 落到 RocksDB，短值 ≤255 字节内联 write CF。聚簇 / 非聚簇主键与 v2 large 标志、255 字节短值边界共同决定一行的 KV pair 数与回表代价。
3. **OceanBase 不是 SQL-on-KV，是关系原生 SSTable**：`blocksstable` 直接组织行 / 列 / rowkey / 索引块，宏块 2MB + 变长微块(可调 1KB–1MB)+ PAX 行列混合编码(`ObRowStoreType`)，`CS_ENCODING` 列式编码自 4.3.0 引入(非 4.3.5)；MemTable / SSTable / macroblock / microblock 更贴近自研关系型存储内核；**OBKV 仅 observer 层接入、K / V 两列简化表，非存储模型**。
4. **三种 log 语义不同**：Raft / PALF 共识 log、LSM WAL、SSTable 在目的 / 生命周期 / 清理 / 恢复上各异，不能混写；Raft Engine 自 v5.4 引入、**v6.1.0 起成为默认**独立承载共识日志，官方《6.1.0 Release Notes》称相比 RocksDB 可降 TiKV 写 I/O 最高约 40%。
5. **LSM 三大放大与 compaction / write stall 是映射的物理代价**：收益来自批量、顺序、压缩友好，代价集中在 compaction / merge、读放大、写放大、空间放大与尾延迟；TiKV 用 scheduler 层流控、OceanBase 用分级转储 + 资源隔离 + 未脏宏块复用来缓解。
6. **两者是工程取舍而非优劣**：TiDB 分层解耦、列存外挂 TiFlash；OceanBase 同引擎一体、内建列编码，各有代价（推测）。

## 5.12 参考文献

[1] TiDB Computing(关系→KV 映射). 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tidb-computing/.
 （支撑:§5.2 行 key=t{tableID}_r{rowID}、索引 key、唯一 / 非唯一索引 value 的编码规则，以及 range scan 与 SQL 层到 TiKV 的数据路径。）
[2] Clustered Indexes. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/clustered-indexes/.
 （支撑:§5.2 / §5.8 聚簇 / 非聚簇表的 KV pair 数差异、_tidb_rowid handle、默认 ON、建表后不可互转及回表成本。）
[3] System Variables: tidb_row_format_version. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/system-variables/.
 （支撑:§5.2 / §5.9 行格式 version 2 默认值、v4.0 起新集群默认、升级兼容只影响新写入数据的版本边界。）
[4] RocksDB Overview(TiKV 三 / 四 CF、WAL / MemTable / SSTable、255 字节边界). 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/rocksdb-overview/.
 （支撑:§5.2 write / default / lock / raft CF 职责、长于 255 字节落 default CF、LSM 写路径。）
[5] Key Monitoring Metrics of TiKV(compaction pending bytes / write stall). 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/grafana-tikv-dashboard/.
 （支撑:§5.7 / §5.9 compaction pending bytes、write stall duration、scheduler 层流控与观测指标。）
[6] TiKV Configuration File(soft / hard-pending-compaction-bytes-limit 默认值). 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tikv-configuration-file/.
 （支撑:§5.9 soft-pending-compaction-bytes-limit 默认 192GiB、hard-pending-compaction-bytes-limit 默认 256GiB(CF 级)/ 10）
[7] TiDB Functions: TIDB_DECODE_KEY. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tidb-functions/.
 （支撑:§5.9 TIDB_DECODE_KEY() 把编码 key 解码为含 table_id / _tidb_rowid / handle 的 JSON、失败时原样返回参数。）
[8] TiKV Deep Dive — Percolator. 官方博客[EB/OL]. https://tikv.org/deep-dive/distributed-transaction/percolator/.
 （支撑:§5.2 三 CF↔Percolator data / lock / write 列映射、(key,ts) 编码、primary lock 语义。）
[9] How TiKV reads and writes. 官方博客[EB/OL]. https://tikv.org/blog/how-tikv-reads-writes/.
 （支撑:§5.2 / §5.6 KV RocksDB / Raft RocksDB 两类实例、Region 路由缓存失效与重试、CF 对应 Percolator 的解释。）
[10] TiDB 6.1.0 Release Notes. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/release-6.1.0/.
 （支撑:§5.2 / §5.7 / §5.11 "Since v6.1.0, TiDB uses Raft Engine as the default storage engine for logs"、"reduce TiKV）
[11] OceanBase: A 707 Million tpmC Distributed Relational Database System. 论文，PVLDB Vol.15 No.12, 2022[EB/OL]. https://vldb.org/pvldb/vol15/p3385-xu.pdf.
 （支撑:§5.3 LSM 基线 / 增量、宏块 2MB / 微块、daily incremental major compaction、关系原生存储。该 707M tpmC 为 TPC-C Result ID 120051701）
[12] The Log-Structured Merge-Tree (LSM-Tree). 论文[EB/OL]. https://dsf.berkeley.edu/cs286/papers/lsm-acta1996.pdf.
 （支撑:§5.1 / §5.7 LSM 通过延迟批量合并降低高频写索引成本、读路径需同时访问内存与磁盘组件的代价。）
[13] OceanBase Block storage / LSM-tree architecture / OBKV-Table data models. 官方文档[EB/OL]. https://en.oceanbase.com/docs/.
 （支撑:§5.3 / §5.7 / §5.8 / §5.10 OceanBase 基于 LSM-tree、baseline / incremental、SSTable / macroblock(2MB)/ microblock(）
[14] tidb new row format design doc(release-8.5). 源码 / 设计文档[EB/OL]. https://github.com/pingcap/tidb/blob/release-8.5/docs/design/2018-07-19-row-format.md.
 （支撑:§5.2 行格式 v2：CodecVer=128 起步原因、large 行条件、O(logN) 查列、v1 / v2 差异。）
[15] pingcap/tidb 仓库(release-8.5 @ commit 67b4876bd57b；另据 v8.5.0 @ d13e52ed6e22 亦确认)— pkg/tablecodec/、pkg/util/rowcodec/. 源码[EB/OL]. https://github.com/pingcap/tidb/tree/release-8.5/pkg/tablecodec.
 （支撑:§5.2 前缀常量(t / _r / _i)、idLen=8、EncodeRowKey / EncodeIndexSeekKey / GenIndexKey、CodecVer=128、la）
[16] tikv/tikv 仓库(release-8.5 @ commit 1f8a140b6d46)— src/storage/mvcc/、components/engine_traits/、components/txn_types/、src/config/mod.rs. 源码[EB/OL]. https://github.com/tikv/tikv/tree/release-8.5/src/storage/mvcc.
 （支撑:§5.2 三 CF 常量、put_lock / put_value / put_write、key.append_ts、SHORT_VALUE_MAX_LEN=255、WriteType、rea）
[17] oceanbase/oceanbase 仓库(v4.2.5_CE @ e7c676806fda;4.3.5_CE @ b28b9bb12f3b；块大小范围另核 v4.4.2_CE @ e859d1b9)— src/storage/blocksstable/、src/storage/memtable/(含 mvcc/ob_query_engine.{h,cpp})、deps/oblib/src/common/ob_store_format.h、deps/oblib/src/lib/ob_define.h、src/sql/resolver/ddl/ob_ddl_resolver.{h,cpp}、src/observer/table/、src/share/inner_table/ob_inner_table_schema_def.py. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE/src/storage/blocksstable.
 （支撑:§5.3 宏块 2MB、微块默认约 16KB、可调范围 1KB–1MB(MIN_BLOCK_SIZE=1024 / MAX_BLOCK_SIZE=1048576)、微块头版本=3、ObRowStoreType）
## 5.13 信息可信度自评

- **源码级(最高可信)**:TiDB 前缀常量 / `idLen=8` / `CodecVer=128` / large 标志，TiKV 三 CF / `append_ts` / `SHORT_VALUE_MAX_LEN=255` / `WriteType`，OceanBase 宏块 2MB / 微块默认约 16KB / 可调范围 1KB–1MB(`MIN_BLOCK_SIZE=1024` / `MAX_BLOCK_SIZE=1048576`)/ 微块头版本 3 / `ObRowStoreType` 枚举 / `CS_ENCODING`(自 4.3.0 引入)/ OBKV 在 observer 层——均来自 commit-pinned checkout(§5.12 源码引用)，经 rg + Read 核对至文件 / 行级，非二手转述。
- **官方文档明确**：行 / 索引 key 编码规则、聚簇 / 非聚簇 KV pair 数、行格式 version 2 兼容规则、255 字节回 default CF、compaction pending bytes / write stall 指标、OceanBase LSM-tree / MemTable / SSTable / 宏块微块层次与合并观测视图、OBKV 为 K / V 两列接入 API。
- **论文层**:OceanBase 存储引擎(VLDB 2022)、原始 LSM-Tree 论文关于 LSM / 宏块 / 列存 / 放大权衡的描述属设计与理论层；TPC-C 707M tpmC 为论文 / 榜单实验数据(见 §5.12)，非"生产事实"，仅作存储设计佐证。
- **官方暗示**:OceanBase MemTable 内部双索引(Hashtable + BTree)、Hash index 命中收益等仅写到官方资料 / 源码模块级可确认的程度，精确结构以源码为准。
- **工程推测**:§5.8 / §5.11 第 6 条"分层 vs 一体的取舍"为基于公开资料的合理推测，未经官方逐条确认，已标注。
- **不确定 / 需进一步查证**(经源码核验后收窄):OceanBase write stall 具体触发阈值(v4.2.5_CE 源码参数名脱敏、en.oceanbase.com 文档被 WAF / 限流挡住)、对象存储延迟与缓存命中率等具体运维数值,仍需进一步查证。已升级为事实者:RocksDB / TiKV `soft/hard-pending-compaction-bytes-limit` 默认值(192GiB / 256GiB,官方文档 + 源码,见 §5.9)、OceanBase 四个合并相关视图名与字段(源码视图定义,见 §5.9)、`TIDB_DECODE_KEY()` 行为与输出(官方文档,见 §5.9)、OceanBase MemTable Keybtree + Hash 双索引结构(源码 `ObQueryEngine`,见 §5.3)。
- **反查证一致 / 无实质冲突**：行 key 前缀官方文档示意为 `r` / `i`、源码常量为 `_r` / `_i`，下划线即分隔符，描述粒度不同、无实质冲突，本章按源码常量书写；RocksDB CF 数文档同时出现"3 CF(default / write / lock)"与"含 raft 的 4 CF"两种表述，分别指用户数据 CF 与含 Region 元数据 raft CF，粒度不同、不矛盾；微块可调范围经源码核为 1KB–1MB；`CS_ENCODING` 引入版本经源码核为 4.3.0；均与锁定版本基线(TiDB 8.5.x、v6.1 Raft Engine 默认、OceanBase TP 4.2.5_CE / AP 4.3.x / 4.4.x)一致。

---


# 第 6 章 无状态 SQL 层

## 6.1 本章核心问题

"无状态 SQL 层"是云原生分布式数据库弹性能力的根基，但这个词很容易被写得过满。本章要回答的核心问题是：一个承担协议接入、SQL 解析、优化、执行调度与事务协调的计算节点，能不能在不丢失正确性的前提下被随时替换、随时扩缩、随时池化?

所谓"无状态"(stateless)，严格定义不是"节点里没有任何数据"，而是：节点本地不持有权威(authoritative)、不可重建的持久状态。一切权威状态(用户数据、事务提交记录、共识日志、元数据)都外置到独立的存储/日志/元数据层；节点本地只保留可从外部权威源重新拉取并重建的软状态与连接状态——schema 缓存(InfoSchema)、plan 缓存、session 变量、prepared statement、本地临时表、游标、进行中的事务上下文。因此"无状态 ≠ 无缓存"，也不等于"连接可以在任意时刻无损漂移"——这是全章必须反复澄清的一条主线。

这个问题之所以是分布式数据库底层的关键，在于它直接决定三件事：(1)弹性——计算能否秒级扩缩、按需池化；(2)故障替换——一个计算节点崩溃时，负载均衡器能否把流量切到其他节点而不需要漫长的数据恢复；(3)云原生形态——计算与存储能否独立计费、独立伸缩。

TiDB 与 OceanBase 的差异正好落在这条边界上。TiDB classic 从诞生起就把 TiDB Server 定位为典型的无状态 SQL 层：它接 MySQL 协议、做 SQL 计算，把实际数据读写发往 TiKV/TiFlash，并依赖 PD 提供元数据、调度与 TSO，因此可经 TiProxy/LVS/HAProxy/ProxySQL/F5 暴露统一入口、适合扩缩容与替换。但 TiDB Server 内部仍有 schema lease、InfoSchema、session 状态、prepared statement/plan cache、DDL owner 候选角色与事务上下文，这些状态决定了连接迁移能做到什么程度。

OceanBase 4.x 的 shared-nothing 形态则相反——OBServer 是一个一体化进程：SQL 引擎、多租户、会话、事务、PALF(Paxos-backed Append-only Log File system)共识日志、存储引擎全部在同一进程内深度耦合，因此不是典型的无状态 SQL 层。ODP/OBProxy 位于客户端与 OBServer 之间，主要做连接管理、最优路由、转发与 session 状态同步，而不是一个完整的 SQL optimizer/executor 层。到 shared-storage(Bacchus)方向，OceanBase 才通过对象存储、共享日志服务、共享缓存与异步后台服务把计算节点推向更无状态的形态；但这属于 shared-storage/Cloud 演进，不能倒推为 classic OBServer 的默认属性。本章逐层拆解这三种形态的数据路径、控制路径与故障路径，并明确各自的代价。 〔文献[1,9]〕

> 本章只聚焦"SQL 计算层是否有状态"这一维度。分片(详见第 1 章)、共识(详见第 3 章)、存储引擎(详见第 4、5 章)、元数据控制面(详见第 13 章)、shared-storage 存算分离的完整展开(详见第 7、23 章)均不在本章重复深入。

## 6.2 TiDB 的实现

### 组件与权威状态边界

TiDB classic 形态由三层组成：TiDB Server(计算)、TiKV(行存)/TiFlash(列存)、PD(元数据与 TSO)。其中 TiDB Server 是无状态 SQL 层：官方架构文档明确表述 "The TiDB server is a stateless SQL layer... It does not store data and is only for computing and SQL analyzing, transmitting actual data read request to TiKV nodes (or TiFlash nodes)"。这一定位解释了它的扩展方式：增加 TiDB Server 主要增加 SQL 解析、优化、执行调度、连接承载与部分 DDL/后台任务能力，而不是搬迁用户数据副本；下线 TiDB Server 也不会触发 Region 副本迁移。

TiDB Server 本地确实持有大量"状态"，但这些状态全部是可重建缓存或连接级状态：

- schema 缓存(InfoSchema)：权威 schema 元数据存在 TiKV(以 KV 编码)，TiDB 本地只持有一份 `InfoCache`。源码侧 `Domain` 结构持有 `infoCache *infoschema.InfoCache`，由 `infoschema.NewCache(do, 1)` 构造(`pkg/domain/domain.go`)。这是本地缓存而非权威状态。
- plan 缓存(plan cache)：执行计划在本地编译并 LRU 缓存，丢了可重编译。
- 会话状态(session)：连接级的用户变量、prepared statement、当前库等，绑定 TCP 连接。
- 事务状态：活跃事务的写 buffer(MemBuffer)在 TiDB 本地暂存，事务的快照时间戳来自 PD 的 TSO；但提交后的权威记录(primary lock、commit 记录)落在 TiKV(Percolator 模型，详见第 10、11 章)。一个尚未提交的事务，其状态钉在发起它的那个 TiDB Server 上。源码侧 `pkg/session/txnmanager.go` 中 `txnManager` 挂在 `SessionVars.TxnManager` 上，`EnterNewTxn` 按事务模式与隔离级别选择 optimistic/pessimistic provider，事务结束清空 provider。这就是为什么"活跃事务无法跨节点迁移"是无状态层的硬边界：本地写 buffer 与未决的 primary lock 协调上下文，无法在另一个实例上无缝接管。

由此可见，TiDB Server 的本地状态可分两类：一类是可丢弃可重建的(schema/plan/region 缓存)，节点崩溃后由新连接在新实例上重新拉取重建；另一类是绑定连接生命周期的(会话上下文、活跃事务、本地临时表)，它们随连接存在，连接断则失效。无状态化解决的是第一类，第二类则靠会话迁移(可迁移部分)与应用重试(不可迁移部分)兜底。

### 数据路径与控制路径

从数据路径看，一条普通读请求进入 TiDB Server 后，先被 MySQL 协议层(`pkg/server/`)解析，再进入 parser、planner/optimizer 与 executor。优化器依赖 InfoSchema、统计信息与 session 变量生成计划；执行器把计划拆成 KV scan、point get、coprocessor request、commit/lock 等请求，发送到实际数据所在的 TiKV，或在合适场景走 TiFlash/MPP 路径。这里的"无状态"意味着 TiDB Server 不拥有 Region 的 Raft log、SSTable 或 MVCC 数据副本。一个直接推论是：扩容 TiDB Server 能缓解 SQL CPU、连接数、优化器与执行器内存压力，但不能直接提升某个热点 Region、单 key 锁冲突或 TiKV 写入瓶颈(详见 §6.7)（推测）。

从控制路径看，TiDB Server 也不是完全"薄"的 stateless proxy。它需要从 PD/TiKV 体系获取 TSO、Region 路由、schema 版本与 DDL job 状态；`Domain` 周期性加载 schema、统计信息等元数据。这些控制状态大多可在节点重启后重建，但重建期间可能出现冷启动、schema reload、统计信息未热、plan cache miss、DDL owner 重新选举等现象。换言之，TiDB 的无状态 SQL 层更接近"持久状态外置、计算节点可替换"，而非"每次请求都完全不依赖本地上下文"。

### schema lease 机制——"无状态 ≠ 无缓存"的核心证据

TiDB 是分布式系统，多个无状态 TiDB Server 必须对 schema 保持一致视图。其机制是 schema lease + 至多两个相邻 schema 版本并存(online DDL 的基础，详见第 14 章)。InfoSchema 是 TiDB Server 内存中的元数据视图，但它受 schema lease 与 DDL owner 推进的 schema version 约束，不是随意缓存。

源码确认(repo: `github.com/pingcap/tidb`,ref: `release-8.5`,commit: `67b4876bd57b`)：

- schema lease 默认值 45 秒：官方配置文档明确 `lease`(DDL lease)默认值为 `45s`，官方 SQL FAQ 同时印证 "if TiDB fails to load the latest schema within a DDL lease (45s by default)， the `Information schema is out of date` error might occur"。
 > 版本/事实冲突说明：该 45s 默认值在 8.5.x 与 7.5.x 两个锁定版本中均成立(官方文档一致)，但实现该默认值的源码构造在两版本间不同。在 v8.5.x 中 `pkg/config/config.go` 以常量 `DefSchemaLease = 45 * time.Second` 表达，并经 `Lease: DefSchemaLease.String()` 赋默认值；但该常量是 v8.3.0 才引入的产物(v8.2.0 及更早、以及整个 7.5.x 均无此常量，7.5.x 直接写裸字符串 `Lease: "45s"`)。因此不可把 `DefSchemaLease = 45 * time.Second` 当作跨两个锁定版本通用的实现；跨版本通用的只有"默认 schema lease = 45s"这一结论。
- 续租与 reload 循环：`pkg/domain/domain.go` 的 `func (do *Domain) loadSchemaInLoop(ctx context.Context)`，以 `do.schemaLease / 2` 为周期(源码注释原文 `// Use lease/2 here as recommend by paper.`)触发 `do.Reload()`。
- 触发 reload 的三种事件：① 定时 ticker；② etcd 全局 schema version 变更通知(`syncer.GlobalVersionCh()`)；③ etcd 会话断开(`syncer.Done()`)。

关键的"无状态 ≠ 无缓存"行为：当 TiDB 实例与 etcd 会话断开，集群把该实例视为 down，etcd 删除其 `/tidb/ddl/all_schema_versions/tidb-id` key；为防 schema 版本不一致，该实例在断连期间会停止 schema validator、禁止执行 DML。换言之，无状态节点的本地缓存(schema)一旦失去与权威源的同步通道，节点会主动拒绝服务而不是读脏数据。schema lease 过期或 schema 变更时，旧 schema 上的事务会报 `ErrInfoSchemaExpired`/`ErrInfoSchemaChanged`(`SchemaValidator.Check` 在成功、schema changed、unknown/expired 三种结果走不同返回路径，定义于 `pkg/domain/schema_checker.go`)，而不是返回错误结果。这正是无状态设计的安全闭环：缓存允许陈旧，但一旦陈旧到可能危及正确性，节点便停止服务、等待重建。

### DDL owner——控制面角色，不破坏无状态定位

TiDB 当前实现中同一时刻只有一个 TiDB node 被选为 DDL Owner，负责处理集群内 DDL 任务；Owner 宕机后通过 etcd 选举出新 Owner 继续工作。这属于控制面角色落在某个 TiDB Server 上，而不是用户数据持久化在这个 TiDB Server 上。对测试与运维来说，需要把"TiDB Server 可替换"与"DDL owner 切换会影响 DDL 推进速度/可见性"分开验证——前者是无状态红利，后者是控制面收敛代价。

### plan cache 的演进：session 级 → instance 级

prepared statement 的客户端协议只把 statement ID 传回服务器，后续 Execute 依赖服务器端保存的 prepared AST/plan cache 对象。源码侧 `pkg/server/conn_stmt.go` 中 prepared statement 执行从当前 session 的 `SessionVars.PreparedStmts` 取 `PlanCacheStmt`，说明 statement ID、AST/plan cache 与连接内 session 绑定。prepared plan cache 默认是 session 级：官方文档明确 "The LRU linked list is designed as a session-level cache because `Prepare`/`Execute` cannot be executed across sessions"，自 v6.1.0 默认开启，且 temporary table 等查询不进入缓存。这意味着每个连接维护独立 plan 缓存，节点切换后缓存冷启动。

从 v8.4.0 起，TiDB 引入 instance 级 plan cache(`tidb_enable_instance_plan_cache`)，让同一实例内所有 session 共享 plan 缓存以节省内存。该特性当前为实验特性、默认关闭：源码侧(release-8.5)`EnableInstancePlanCache = atomic.NewBool(false)` 即默认关闭。

> 版本归属反查证：官方 release-8.4.0 release notes 与 system-variables.md(release-8.4/release-8.5/master 三分支一致)均标注 instance plan cache 为 "New in v8.4.0"、实验特性(Warning: not recommended for production)、默认关闭；7.5.x(LTS，早于 8.4.0)中根本不存在该特性，不应将其与 7.5.x 并列为生产事实。
>
> 数值冲突说明：`tidb_instance_plan_cache_max_size` 的默认上限存在源码与官方文档不一致。源码(release-8.5/v8.4.0)`DefTiDBInstancePlanCacheMaxMemSize = 100 * size.MB`，而 `pkg/util/size` 中 `MB = 1024*1024`，故源码值 = 104857600 字节 = 100 MiB；但官方 system-variables.md(三分支一致)明确写该变量 "Default value: `125829120` (which is 120 MiB)"。两者矛盾，本章不采信单一来源数字，以官方文档 120 MiB 为准并标注冲突，不再表述为"上限 100MB"。

无论 session 级还是 instance 级，plan cache 都是本地可重建缓存；节点替换后命中率从零回升，带来短暂的编译开销(详见 §6.7/§6.8)。

### 会话状态外置与 TiProxy 连接保持

**表 6-1　会话状态可迁移性分类**

| 状态类别 | 典型例子 | 是否可随连接迁移 | 影响 / 原因 |
|---|---|---|---|
| 可序列化迁移 | 用户变量、prepared statement、当前库 | 可迁移 | 经 `SHOW`/`SET SESSION_STATES` 重放，配 TiProxy 连接级平滑切换 |
| 须等待后迁移 | 活跃事务、未读完游标/结果集 | 须等待 | 等事务结束或结果取尽再迁，属「等待」语义 |
| 直接拒绝迁移 | 本地临时表、表锁、advisory lock | 不可迁移 | 不返回会话状态，报 `ErrCannotMigrateSession`(`session:8146`) |
| 绑定连接生命周期 | 本地临时表(仅 TiDB 内存、会话可见) | 不可迁移 | 仅存实例内存、不持久，连接断即失效，跨节点不可见 |

无状态 SQL 层的难点是会话状态：用户变量、prepared statement、当前库等绑定在 TCP 连接上，节点一旦更换便会丢失。TiDB 的解法是让会话状态可序列化、可迁移。源码确认 `pkg/sessionctx/sessionstates/session_states.go` 的 `type SessionStates struct`，注释原文 "SessionStates contains all the states in the session that should be migrated when the session is migrated to another server. It is shown by show session_states and recovered by set session_states"。字段包括 `UserVars`、`SystemVars`、`PreparedStmts`、`CurrentDB`、`Bindings`、`ResourceGroupName` 等。

TiProxy(v7.6.0 实验引入、v8.0.0 GA)正是基于此做连接保持：官方文档把 TiProxy 定位为 PingCAP 官方 proxy，位于 client 与 TiDB Server 之间，提供 load balancing、connection persistence、service discovery 与 connection migration。在计划内 scale-in、rolling upgrade、rolling restart 时，TiProxy 对原实例发 `SHOW SESSION_STATES` 拿到 JSON，连到新实例发 `SET SESSION_STATES '{...}'` 重放，实现连接级平滑切换，客户端连接保持不变。

但并非所有会话状态都可迁移——这是无状态化的代价点。源码定义了迁移失败错误 `ErrCannotMigrateSession`(`session_states.go`，错误码 `session:8146`，消息 "cannot migrate the current session: %s"，经 TiDB `errors.toml` 确认)。官方 Session Manager 设计文档(`docs/design/2022-07-20-session-manager.md`)区分了两类阻断机制：

- 直接拒绝返回 session states：当 session 含本地临时表(local temporary tables)、表锁(table locks)或 advisory locks(咨询锁)时，TiDB 不返回会话状态，Session Manager 报连接失败(原文 "When the session contains local temporary tables, table locks, or advisory locks, the TiDB won't return the session states and Session Manager will report connection failure")。
- 必须等待完成：活跃事务需等事务结束再迁、未读完的游标/结果集需等数据取尽，二者属"等待"语义而非直接拒绝。

此外长 DDL(`ADD INDEX`)/`LOAD DATA` 也可能迁移失败；TiProxy 文档进一步给出：单语句/单事务执行时间超过 `graceful-wait-before-shutdown` 减 10 秒时连接无法迁移，且 TiDB 意外宕机时连接仍会断开。这再次说明 TiDB Server 是"SQL 计算层可水平扩展"，不是"所有连接状态都全局复制"。

### 本地临时表——无状态层的例外

本地临时表是典型的 session-local 内存状态，绑定连接、跨节点不可迁移。官方文档：本地/全局临时表的数据 "only stored in the memory of TiDB instances"、"not persisted"，本地临时表 "visible only to the current session"、会话结束自动删除。源码侧 `pkg/sessionctx/variable/session.go` 的 `LocalTemporaryTables` 字段持有 `*infoschema.SessionTables`。这是无状态 SQL 层为兼顾 MySQL 语义而保留的有状态例外(详见 §6.8)。

由此 TiDB 的故障路径有两类。第一类是 SQL 节点故障：未提交事务、prepared statement、session 变量、本地临时表这类连接状态会丢失或需要客户端/代理处理；已提交数据由 TiKV/TiFlash 与 PD 体系继续保证。第二类是 schema/控制状态变化：DDL owner 切换、schema lease 过期、InfoSchema 未更新会导致 DDL 推进延迟或 DML 返回 schema 相关错误，但不意味着 TiDB Server 保存了唯一数据副本(详见 §6.6)。

必查模块小结(commit-pinned)：`pkg/domain/`(schema lease/checker)、`pkg/session/`(txn manager/session)、`pkg/server/`(协议/conn_stmt)、`pkg/sessionctx/sessionstates/`(会话迁移)、`pkg/planner/core/`(plan cache)、`pkg/config/config.go`(lease 常量)，均确认于 release-8.5 @ `67b4876bd57b`。 〔文献[2-7,13]〕

## 6.3 OceanBase 的实现

### OBServer：一体化进程，不是典型无状态 SQL 层

![[f6_1.svg]]
**图 6-1　OBServer 单进程内模块组成（与 TiDB 多组件分离对照）**

OceanBase shared-nothing(传统形态)的核心是 OBServer 一体化进程。OceanBase v4.2.5_CE 与 4.3.5 系统架构文档说明，shared-nothing 模式下所有节点平等，每个节点都有自己的 SQL engine、storage engine 与 transaction engine，运行在普通 PC server 集群上；层间交互 "Function calls are made for interaction between the layers on a single server"，"RPCs are used only when it is necessary to communicate between nodes"——节点内函数调用、节点间才走 RPC。官方博客进一步解释，OceanBase 4.0 以后希望用同一代码适配 standalone 与 distributed 模式，把 SQL、transaction、storage engine 的单机/分布式能力做一体化；VLDB 2023 的 OceanBase Paetica 论文论证了这种单机/分布式一体化设计在 V4.0 的实现。这不是"SQL 层和存储层完全分离"的架构，而是把多层能力放进同一个 OBServer 进程内。

源码确认(repo: `github.com/oceanbase/oceanbase`,ref: `v4.2.5_CE` tag,commit: `e7c676806fda`，文件 `src/observer/ob_server.h`)：`class ObServer` 在同一个类里同时 include 并持有——

- `sql::ObSql sql_engine_;`(SQL 引擎)
- `sql::ObSQLSessionMgr session_mgr_;`(会话管理器——会话状态在本进程内)
- `omt::ObMultiTenant multi_tenant_;`(多租户管理)
- `share::schema::ObMultiVersionSchemaService &schema_service_;`(schema 服务)
- `blocksstable::ObStorageEnv storage_env_;`(存储环境)
- `transaction::ObWeakReadService weak_read_service_;`、`share::ObLocationService location_service_;` 以及 `rootserver/ob_root_service.h` 引入的 RootService 相关运行时

对外接口侧亦印证耦合：`ObServer` 提供 `get_sql_session_mgr()`、`get_root_service()`、`get_mysql_proxy()`、`get_location_service()` 等；构造路径初始化 SQL connection pool、DDL connection pool、RPC proxy、schema service、location service、session manager、RootService monitor、multi-tenant、weak read service 与 table service 等对象。

更关键的耦合证据在多租户注册路径：`src/observer/omt/ob_multi_tenant.cpp` 中，每个租户通过 MTL(Multi-Tenant Library)在同一进程内注册了完整的有状态模块——`ObStorageLogger`(存储 WAL)、`ObTransService`(事务服务)、`ObLogService`(PALF 共识日志服务)、`ObLSService`(Log Stream 服务，承载 tablet/分区副本)，以及 SQL DTL/DAS、timestamp service、transaction ID service、tablet scheduler、checkpoint、compaction、table lock、DDL scheduler、plan cache 等模块。同文件 `start_sql_nio_server` 按 tenant 创建 `ObSqlNioServer`，网络线程数来自 `tenant_sql_net_thread_count` 或租户的 `unit_min_cpu()`——可见 OBServer 上的 SQL 服务天然运行在 Tenant/Unit 资源约束里，与存储、事务、后台任务共享同一租户资源模型。

这说明：在 OBServer 单进程内，每个租户的计算 + 会话 + 事务 + 共识日志(PALF)+ 存储(Log Stream/SSTable)都挂在一起。OBServer 因此携带持久状态(本地副本数据、共识日志)，无法像 TiDB Server 那样即时无状态替换。官方也明确 OceanBase 是 stateful replication 模型，"stores user data in partitions, with multiple replicas"。

### ODP/OBProxy：代理与会话同步边界，不是 SQL 计算层

OceanBase classic 的 SQL 接入路径表面上也有代理、SQL、存储几个概念，但边界与 TiDB 不同。ODP/OBProxy 接收客户端 SQL 请求，选择合适 OBServer 并转发，再把执行结果返回客户端；其核心能力是连接管理、最优路由、高性能转发、高可用与私有协议，而不是独立执行完整 SQL 计划。官方 ODP session status synchronization 文档说明，一个 client 到 ODP 的连接背后可能对应多个 ODP 到 OBServer 的 server connections，因此为保证结果正确，需要同步多个 server connection 的 session status。如果某些状态无法透明同步，代理就必须保持粘性或把请求路由回状态所在节点。因此 OceanBase 的"代理层"不是把 SQL 执行层变无状态，而是在一体化 OBServer 前面做路由与连接语义维护——不应写成 OceanBase 的"无状态 SQL 计算层"。

OceanBase 的 session 状态也存在，只是处理方式更贴近 OBServer 内核。`src/sql/session/ob_basic_session_info.h` 中 `ObBasicSessionInfo` 保存 tenant、effective tenant、rpc tenant、sys vars cache、session manager、事务描述、mutex 等，并提供 `is_in_transaction()`、`has_explicit_start_trans()`、`sync_default_sys_vars()`、`get_local_ob_enable_plan_cache()` 等接口。

### 数据路径 / 控制路径 / 故障路径

在上述一体化结构之上，可进一步从三条路径观察其运行特征。

- 数据路径(正常)：OBProxy/ODP 按租户、库表、分区位置、事务状态等信息路由，把请求送到持有 Leader 副本的 OBServer；被选中的 OBServer 内 SQL→事务→存储全程函数调用，读 MemTable/SSTable，写经 PALF 复制式 WAL 以 Paxos 多数派落盘并推进事务状态(详见第 3、4 章)。如果路由不准，OBServer 仍可通过远程计划或内部转发完成请求，但性能与故障路径会变化。
- 控制路径：schema 用 `ObMultiVersionSchemaService` 多版本管理，与 TiDB 的 lease 机制不同——它不是按租约定时过期，而是把多个 schema 版本作为一组 `ObSchemaMgr` 缓存在带槽位的版本缓存里(`src/share/schema/ob_schema_mgr_cache.h` 中 `MAX_SCHEMA_SLOT_NUM = 256`，`ob_multi_version_schema_service.h` 中 `MAX_VERSION_COUNT = 64`)，刷新由两类任务驱动而非租约：`src/observer/ob_server_schema_updater.h` 的 `ObServerSchemaTask::TYPE` 枚举区分 `REFRESH`(心跳触发，注释 `schema refresh task caused by heartbeat`)与 `ASYNC_REFRESH`(SQL 触发，注释 `async schema refresh task caused by sql`)；失效(回收)则走 `RELEASE`(`schema memory release task`)与 `ObSchemaMgrCache::try_eliminate_schema_mgr`/`get_recycle_schema_version` 把旧版本逐出回收，由 `schema_refresh_mutex_` 串行化("assert only one thread can refresh schema")。DDL、Unit/Resource Pool、balancer 收敛进 sys tenant 的 RootService(详见第 13 章)。
- 故障路径：OBServer 宕机 → 其 Log Stream 的 Follower 副本经 Paxos 选举 + LogReconfirm 重建 Leader(详见第 3 章)，数据不丢(RPO=0，适用范围见 §6.7)；这与 TiDB Server 故障"无数据恢复、直接切流量"形成本质对比。

这个差异会直接改变运维动作。TiDB 中增加 SQL 节点通常优先观察连接均衡、TiDB CPU、SQL 执行内存与 plan cache 预热；OceanBase classic 中增加 OBServer 后，还要考虑该节点是否承载租户 Unit、是否有 Tablet/LS 迁入、是否成为某些 LS leader、compaction/merge 与事务日志压力是否同步变化。如果只把 OBServer 当成无状态 SQL 节点，就容易低估扩容后的数据均衡时间、租户资源重分配与故障恢复路径。

### 4.4.x 融合版：仍是一体化

这一耦合结构并未随版本演进而松动。即便到 TP+AP 融合的 4.4.x,OBServer 仍是 SQL 引擎 + 多租户同进程的一体化结构，`sql::ObSql sql_engine_;`、`omt::ObMultiTenant multi_tenant_;` 仍是同一 `ObServer` 类成员。

### shared-storage(Bacchus)：趋向无状态计算

OceanBase 最新的 shared-storage 形态在 SN 基础上引入存算分离。官方 shared-storage 文档(4.3.5+ 文档/4.4.x 架构；社区版默认不编译)说明，SS 模式下 RW 节点处理读写、RO 节点处理只读任务，SSWriter 生成并上传增量与基线数据到对象存储，所有 compute nodes 共享访问这些数据、只在本地缓存 hot data。Bacchus 论文(arXiv:2602.23571,2026-02 提交；OceanBase Cloud 推出)进一步提出面向对象存储的 LSM-tree shared-storage 架构，用 shared service-oriented PALF 日志把日志同步做成共享服务，从而 "rendering compute nodes stateless"；配合 Shared Block Cache Service(共享缓存)与异步后台服务(compaction、DDL、backup、recovery)，使 "compute, cache, and storage resources can be resized rapidly and independently"。

> 高风险提示：Bacchus 趋向无状态计算的具体代码路径未在本任务核对的 v4.2.5_CE/4.4.x checkout 中确认(二者均为 shared-nothing 一体化结构，社区版默认不编译 shared-storage 模块)。Bacchus 在生产 OLTP 上的延迟数据属论文实验数据，不得引用为生产事实，须显式标注 benchmark 条件与论文出处。本节相关论断一律标注为推测，且不能回写为 classic/shared-nothing OBServer 的属性。

必查模块小结(commit-pinned)：`src/observer/ob_server.h`(ObServer 一体化成员与对外接口)、`src/observer/ob_server.cpp`(构造期对象初始化)、`src/observer/omt/ob_multi_tenant.cpp`(租户 MTL 注册事务/PALF/LS、`start_sql_nio_server`)、`src/sql/session/ob_basic_session_info.h`(session 信息)，确认于 v4.2.5_CE @ `e7c676806fda`;4.4.x 成员级亦确认一致。 〔文献[8,10-12,14]〕

## 6.4 核心差异对比

**表 6-2　无状态 SQL 层:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB(TiDB Server) | OceanBase shared-nothing(OBServer) | 影响 |
|---|---|---|---|
| SQL 层是否无状态 | 是，典型无状态 SQL 层，不存储用户数据 | 否，一体化进程，同时承载 SQL/事务/存储引擎 | TiDB 计算可秒级扩缩/替换；OB 节点替换需副本恢复 |
| 权威状态位置 | 全外置：TiKV(数据)+ PD(元数据/TSO) | 内置：本进程内含副本数据 + PALF 日志 | TiDB 计算/存储独立伸缩；OB 计算与存储绑定 |
| 本地保留什么 | 仅可重建缓存 + 连接状态(schema/plan/session) | 副本数据、共识日志、MemTable、会话、租户运行时 | TiDB 丢了能重建；OB 丢了要 Paxos 恢复 |
| 代理角色 | TiProxy：连接迁移/保持/服务发现，有长事务/本地临时表等限制 | ODP/OBProxy：连接、路由、转发、session 状态同步 | 二者都不是万能状态复制层；连接语义仍限制迁移 |
| schema 一致性机制 | schema lease(默认 45s,7.5.x/8.5.x 一致)+ etcd 同步 + 至多两版本并存 | ObMultiVersionSchemaService 多版本 | 机制不同；TiDB 断连即停 DML 防脏 |
| 会话状态 | 可序列化迁移(`SHOW`/`SET SESSION_STATES`) | 在 session_mgr_ 进程内，跨节点靠 ODP 同步/重连 | TiDB 配 TiProxy 平滑切换；OB 连接切换更依赖 ODP |
| 故障替换语义 | 切流量，无数据恢复 | 副本选主 + 日志恢复，RPO=0(范围见 §6.7) | TiDB 替换快；OB 数据强一致代价是恢复重 |
| 计算池化/云原生 | 原生支持(计算/存储独立扩展) | shared-nothing 不支持；Bacchus 趋向无状态计算 | 形态差异决定弹性上限 |

对照解读：这张表不能简化为"TiDB 更先进 / OceanBase 更复杂"。两者是不同的工程取舍：TiDB 用"计算无状态 + 存储有状态分离"换弹性，代价是每个 SQL 几乎都要跨网络访问 TiKV(网络跳转，详见 §6.7)；OceanBase 用"计算存储同进程"换节点内零网络跳转的执行效率(函数调用 vs RPC)，代价是计算无法独立于存储弹性伸缩。Paetica 论文的核心动机正是论证一体化设计在单机场景"no extra overhead"。

## 6.5 正常路径图

图 6-2 对照两者的正常读写路径，凸显"TiDB 计算层跨网络访问存储"与"OBServer 节点内函数调用"的本质差异。这张图刻意没有把 ODP/OBProxy 画成 OceanBase 的 SQL engine，也没有把 TiProxy 画成 TiDB Server 的状态复制层：TiProxy/ODP 都在入口侧改善连接与路由，真正的 SQL 解析、优化与执行分别在 TiDB Server 与 OBServer 内完成。

![[f6_2.svg]]

**图 6-2　无状态 SQL 层正常读写/调度路径**

要点：TiDB 路径里，计算节点本地只有可重建缓存(虚线)，真实数据读写是跨网络 RPC(实线)；OBServer 路径里，SQL→事务→存储→PALF 全部进程内函数调用(点线)，只有跨节点共识才走网络(双线)。

## 6.6 故障/异常路径图

![[f6_3.svg]]

**图 6-3　无状态 SQL 层故障/异常路径**

对照解读：故障路径上的关键区别是恢复粒度。TiDB Server 异常时，持久化用户数据不在 TiDB Server 上，风险集中在当前连接、未提交事务、prepared statement、本地临时表、schema 缓存与 DDL owner 切换——核心代价是冷缓存与不可迁移会话。OBServer 异常时，恢复不仅是连接重连，还涉及该节点上租户资源、Tablet/LS 副本、日志与事务参与者状态；ODP 可协助刷新路由，但不能替代 OBServer 内核的副本选主与日志恢复路径，核心代价是恢复比"切流量"重，但换来 RPO=0。这再次说明二者代价不在同一层面。

## 6.7 性能、可靠性、运维影响

无状态与一体化的取舍并不停在架构图上，而是落到延迟、缓存、扩展与运维四条可观测的曲线上。以下按维度分别陈述。

延迟：TiDB 无状态层的代价是网络跳转——几乎每条 SQL 都要跨网络访问 TiKV(以及向 PD 取 TSO，详见第 13 章)，点查相比"计算存储同节点"多一跳 RTT;TiDB 用 coprocessor 下推、batch、plan cache、region cache 缓解，但分布式 join/事务协调仍受 Region 分布影响。OBServer 节点内函数调用省掉这一跳，本地化机会更多，但一体化节点上的 compaction、事务、SQL 共享同一租户资源，可能相互影响；跨节点分布式事务/查询仍走 RPC。

吞吐与缓存命中：TiDB plan cache(session 级默认开、instance 级实验态默认关)直接影响编译开销；节点替换或连接漂移后 plan/schema/statistics cache 冷启动，短期 miss 率升高、P99 抖动，缓存可重建但热度被稀释。OBServer 内 plan/session/location/storage cache 与租户更近、随进程常驻，但进程故障涉及更广状态恢复，代价更大。

一个更细的性能判断：TiDB Server 扩容对"SQL 计算饱和"很有效，对"存储热点或事务冲突"效果有限。当 CPU 集中在 SQL 解析、优化、表达式计算、排序聚合、连接管理时，增加 TiDB Server 可能明显降低排队；但若瓶颈来自单 Region 热点、悲观锁等待、提交阶段冲突、TiKV I/O 或跨 Region 读放大，SQL 节点扩容只会把更多请求更快地送到同一个下游瓶颈（推测）。因此压测报告应把 TiDB Server CPU、TiKV CPU/I/O、PD TSO/调度、事务冲突与网络流量拆开，而不是只给一个整体 TPS 数字。

可用性与故障恢复：TiDB Server 无状态 → 故障即切流量、无数据恢复，这是无状态层在可用性上的主要优势；配 TiProxy 可做到连接级平滑(对不可迁移会话除外)。但 TiDB Server 故障对应用仍可能是连接中断、事务失败、prepared statement ID 失效或本地临时表消失——测试时不能只看"服务是否仍可连"。OBServer 故障 → 依赖 Paxos 选主与日志恢复，RPO=0 但恢复更重(详见第 3 章)。需要限定的是：RPO=0、RTO<8s 这一指标仅适用于 OceanBase V4.x 的 LS/Paxos 多副本、少数派故障或多数派完整的场景，不绑定 RootService、也不外推到单副本、shared-storage 或跨云形态。

扩展性：TiDB 计算与存储可独立横向扩展(加 TiDB Server 扩计算、加 TiKV 扩存储)；OceanBase shared-nothing 加 OBServer 同时扩计算与存储，二者绑定，这是 OceanBase 推出 shared-storage 的核心动因之一。

运维复杂度：TiDB 多一个独立计算层与 TiProxy/etcd schema 同步链路，组件更多，但 SQL 层与存储层容量可分开观察(关注连接数、plan cache、schema lease、DDL owner、事务大小)；OceanBase 一体化进程组件更少、部署更"像单机"，但单进程内多子系统耦合，诊断面更宽(连接路由、location cache、租户资源、LS leader 切换、compaction 抖动)。shared-storage/Bacchus 又引入共享缓存、共享日志与对象存储的新运维面。两者无绝对优劣。

内存放大：无状态多实例的一个隐性代价是缓存冗余——每个 TiDB Server 各自缓存全量 schema 与各自的 plan，实例数越多冗余越大；instance plan cache 把单实例内的 session 级冗余收敛为实例级共享(`tidb_instance_plan_cache_max_size` 官方文档默认 120 MiB，源码与文档数值存在冲突，见 §6.2)，但跨实例冗余仍在。OBServer 单进程聚合，不存在跨实例 schema 冗余，但租户间内存靠 Unit/Resource Pool 隔离(详见第 17 章)，单进程承载多租户带来另一种内存治理复杂度。shared-storage 形态中，若 compute、本地热数据 cache 与 Shared Block Cache Service 的边界配置不当，也可能出现本地 cache 重复、共享 cache 争用或冷数据反复拉取（推测）。可见"无状态"与"一体化"在内存开销上各有其代价。

从可靠性语义看，无状态 SQL 层降低的是"计算节点带状态恢复"的压力，而不是降低事务正确性的要求。TiDB Server 可以丢弃本地计算状态，但正在执行的事务仍必须通过 TiKV 的事务协议决定提交、回滚或重试；Bacchus 的 stateless compute 可以趋向无状态，但共享日志服务、对象存储元数据、后台恢复服务仍必须保证一致性与可用性。因此架构上越强调计算无状态，测试上越要验证下游 durable state 的恢复边界，包括重复提交、客户端超时后未知结果、连接重建后的事务状态查询与 schema 版本变化。

计算池化视角：无状态 SQL 层最大的云原生价值是计算池化——多个逻辑集群可共享一个无状态计算池，按需调度；TiDB Cloud 的 Serverless 形态正基于此(其内部隔离机制未公开，本章不做实现层断言)。OceanBase shared-nothing 因计算与存储绑定，池化粒度是整节点而非计算；Bacchus 试图让计算可池化。

### 性能瓶颈来源对比(第二张对比表)

**表 6-3　性能瓶颈来源对比(第二张对比表)**

| 瓶颈来源 | TiDB(无状态层) | OceanBase(一体化) |
|---|---|---|
| 网络跳转 | 每 SQL 跨网络访 TiKV + 取 TSO(主要代价) | 节点内函数调用消除；仅跨节点事务/查询走 RPC |
| 缓存冷启动 | 节点替换/连接漂移后 plan/region cache miss | 进程故障代价大，但日常缓存常驻 |
| 事务协调 | 2PC over Percolator 跨 TiDB+TiKV+PD | 事务引擎在 OBServer 内，跨节点事务才放大 |
| schema 同步 | etcd 全局版本 + lease，断连停 DML | sys tenant schema service，多版本 |
| 故障恢复重量 | 轻：切流量、重建缓存 | 重：Paxos 选主 + 日志重放 + 租户/LS 恢复 |
| 内存放大 | 多实例各自缓存 schema/plan(instance cache 缓解) | 单进程聚合，租户间靠 Unit 隔离 |

## 6.8 反例与代价

无状态 SQL 层并非银弹，以下场景或维度暴露其代价：

1. session-heavy workload：大量 prepared statement、server-side cursor、本地临时表、用户变量、长事务与连接粘性会削弱"任意 SQL 节点可替换"的效果。TiProxy 文档已明确本地临时表、长事务与 cursor 是连接迁移限制——本地临时表、表锁、advisory lock 使会话被直接拒绝返回 session states，活跃事务与未读游标则须等待结束/取尽才能迁移(`ErrCannotMigrateSession`，错误码 `session:8146`)。OceanBase ODP 也需同步多个 server connection 的 session status，说明代理层面对 session 语义不能随意忽略。无状态化在这些 MySQL 语义面前是部分无状态。

2. 本地临时表是有状态例外：它绑定连接、只在 TiDB 内存、跨节点不可见。用本地临时表的工作负载本质上把状态钉在了某个节点，削弱了无状态红利。

3. 复杂分布式执行：无状态 SQL 层让计算节点易扩展，但 join、聚合、排序、事务提交与锁冲突仍要跨存储节点通信。TiDB Server 不存数据意味着大查询可能需要从 TiKV/TiFlash 拉取大量中间结果，或依赖 Coprocessor/MPP 下推；如果统计信息不准、plan cache 误命中或数据分布变化，SQL 层扩容并不自动解决尾延迟。这正是 OceanBase 一体化设计与 Paetica 论文强调"单机无额外开销"所针对的场景（推测）。

4. plan cache 冷启动塌陷点：大规模滚动升级或频繁扩缩时，新实例 plan cache 从零回升，短期编译开销与 P99 抖动；参数化程度低、SQL 多样的负载尤其明显。instance plan cache 仍是实验特性、默认关闭，不能假定生产已开启。

5. 控制面/元数据路径：TiDB 的 schema lease 与 DDL owner 说明，SQL 层无状态并不消除 DDL 协调——无状态节点失去与 etcd 同步通道时主动停 DML(报 `ErrInfoSchemaExpired`)，是用可用性换正确性的取舍，etcd/PD 抖动会放大为 TiDB 侧 DML 不可用，需在监控上对齐(详见 §6.9)。OceanBase classic 则把更多控制与元数据运行时放入 OBServer/tenant/RootService 体系，故障定位时要区分连接路由失败、location cache 不准、tenant 资源不足、LS leader 切换、compaction 抖动等路径。

6. shared-storage 的冷启动与共享服务瓶颈：Bacchus 把 compute nodes 写成 stateless，但 stateless compute 仍依赖共享日志、共享缓存、对象存储与异步后台服务。一旦 Shared Block Cache Service 命中率下降、对象存储尾延迟上升、共享日志服务成为热点，compute 节点数量增加也可能只是在放大下游瓶颈（推测）。论文实验可说明设计目标与条件下的结果，不能作为生产 OLTP 延迟承诺，且生产成熟度需以官方 GA 文档为准。

## 6.9 测试开发视角的验证点

可测试的功能场景：

- TiDB：滚动升级时配 TiProxy，验证已有连接的会话状态(用户变量、prepared stmt、当前库)是否经 `SHOW`/`SET SESSION_STATES` 完整迁移；验证含本地临时表/活跃事务/未读游标的连接是否如预期被拒绝迁移；scale-out/scale-in 后连接池是否重建 prepared statements；显式事务跨 TiDB Server 重启是否按预期失败并由客户端重试。
- OceanBase：杀掉一个 OBServer，验证其 Log Stream 副本经 Paxos 选主后业务恢复、RPO=0；验证 ODP 在 session 变量变化后能否保持多个 server connections 语义一致、路由刷新与 LS leader 恢复是否符合预期。

建议把验证分为四组。第一组无状态收益验证：只跑 autocommit 短事务与只读查询，逐步增减 TiDB Server 或 OceanBase SS compute，观察连接均衡、吞吐与错误率。第二组连接状态验证：启用 prepared statement、session 变量、临时表、游标与长事务，执行计划内维护，看代理能否保持连接或给出可预期错误。第三组控制面验证：TiDB 强制 DDL owner resign、延迟 schema reload,OceanBase 刷新 location/route 或切换 LS leader，观察 SQL 错误是否可重试、是否出现旧 schema 或错误路由。第四组冷启动验证：新 SQL/compute 节点上线后，观察 plan cache、schema cache、location cache、block cache 预热对 P99 的影响。

可注入的失效模式：

- TiDB：kill 一个 TiDB Server；切断 TiDB↔etcd/PD 连接 → 验证该实例停 schema validator、DML 报 `ErrInfoSchemaExpired`/`Information schema is out of date`、恢复后自动 reload；DDL 进行中制造 schema 变更 → 验证旧 schema 事务报 `ErrInfoSchemaChanged`、至多两版本并存语义；强制 DDL owner resign；制造长事务/游标/本地临时表再执行 TiProxy 计划维护。
- OceanBase：网络分区 → 验证少数派副本不可写、多数派继续服务；注入 ODP 到某 OBServer 的连接失败、OBServer 进程重启、location 过期、租户 CPU/内存资源不足、LS leader 切换，以及 shared-storage 模式下本地 cache 冷启动。

关键压测指标应区分 SQL 层与存储层：TiDB 观察 QPS/TPS、P95/P99、连接数、TiDB Server CPU/内存、optimizer/execute 阶段耗时、plan cache hit、schema reload/lease 错误、事务大小与 lock wait;OceanBase 观察 ODP 路由耗时、OBServer SQL 执行耗时、租户 CPU/内存、事务提交耗时、LS leader 切换、compaction/merge 抖动与 cache 命中。

可查证的观测/诊断字段(已核验者)：TiDB 可用 `ADMIN SHOW DDL` 查看 DDL owner;plan cache 命中可用 `@@last_plan_from_cache` 与 `EXPLAIN FORMAT='plan_cache'` 验证；schema 一侧 `Information schema is out of date` 错误、`ErrInfoSchemaExpired`/`ErrInfoSchemaChanged` 错误码(源码确认)，schema version 通过 etcd `/tidb/ddl/all_schema_versions/` 路径同步(源码确认)；TiProxy 的 `graceful-wait-before-shutdown` 参数影响可迁移性(官方文档)。其余具体 Prometheus metric 名、内部视图名须以部署版本文档为准，需进一步查证，不在此编造。

## 6.10 容易误解点

1. "无状态 = 节点里没数据" ——错。无状态指不持有不可重建的权威持久状态；TiDB Server 本地有大量缓存与连接状态(InfoSchema、schema lease、prepared statement、plan cache、session 变量、txn manager)，只是都能从 TiKV/PD/etcd 重建或随连接生命周期失效。"无状态 ≠ 无缓存"是本章主线。

2. "无状态层节点切换零代价、对应用完全透明" ——不全对。schema/plan 缓存切换后要重建(冷启动代价)；会话状态需 TiProxy 才能平滑迁移，且 TiDB 意外宕机、长事务、未读游标、本地临时表等场景根本无法迁移(`ErrCannotMigrateSession`)。应用仍需重连、重试与幂等设计，透明是有条件的。

3. "OceanBase 比 TiDB 复杂 / TiDB 比 OceanBase 简单" ——这是被禁止的简化。OBServer 一体化是针对节点内零网络跳转与单机/分布式一体化的工程取舍(Paetica 论文)，代价是计算无法独立弹性；TiDB 无状态是针对弹性的取舍，代价是网络跳转与会话迁移复杂度。二者是不同维度的取舍，不存在单一"先进性"排序。

4. "OBServer 是 SQL 层、像 TiDB Server" / "OceanBase 有 ODP 就有独立无状态 SQL 层" ——错。OBServer 在同一进程内还含事务、PALF 共识日志、Log Stream 存储与租户运行时，是携带持久状态的一体化进程，不是可即时替换的无状态 SQL 层(源码 `ob_server.h` 成员确认)；ODP/OBProxy 主要是连接、路由与 session 状态同步层，classic OBServer 才执行 SQL。

## 6.11 本章结论

1. 无状态 SQL 层的严格定义是"本地不持有不可重建的权威持久状态"，而非"无缓存"；TiDB Server 是典型无状态 SQL 层，本地仅有可从 TiKV/PD/etcd 重建的 schema/plan/session 缓存与连接状态，这些状态影响连接迁移与错误处理。
2. TiDB 用 schema lease(默认 45s,7.5.x/8.5.x 一致；lease/2 续租 reload)+ etcd 全局版本 + 至多两版本并存维持无状态节点间 schema 一致；与 etcd 断连时主动停 schema validator、禁 DML(报 `ErrInfoSchemaExpired`)，是"无状态 ≠ 无缓存"的核心证据(注：45s 默认值跨两版本通用，但 `DefSchemaLease` 常量仅 v8.3.0+ 存在，7.5.x 用裸字符串 `Lease: "45s"`，见 §6.2)。
3. TiDB 把会话状态做成可序列化(`SessionStates`)，配 TiProxy(v8.0.0 GA)经 `SHOW`/`SET SESSION_STATES` 做连接级平滑切换；但 TiDB 意外宕机、活跃事务、本地临时表、未读游标等无法迁移(`ErrCannotMigrateSession`,`session:8146`)，无状态化是"部分无状态"；DDL owner 是落在某 TiDB Server 上的控制面角色，不破坏其无状态定位。
4. OceanBase shared-nothing 的 OBServer 是一体化进程：SQL 引擎、会话、多租户、事务、PALF 日志、Log Stream 存储在同一 `ObServer` 类/进程内，因此携带持久状态、不是典型无状态 SQL 层；ODP/OBProxy 仅做连接、路由、转发与 session 状态同步，不是完整 SQL optimizer/executor；节点故障靠 Paxos 选主与日志恢复(RPO=0)而非切流量。
5. OceanBase shared-storage(Bacchus)试图通过 shared service-oriented PALF + 共享缓存 + 异步后台服务让计算节点趋向无状态，但该论断应限定在 shared-storage/Cloud/论文语境，不能泛化到 classic 默认形态；其具体代码路径未在社区版 checkout 确认，生产延迟数据属论文实验、不得当生产事实。
6. instance 级 plan cache(`tidb_enable_instance_plan_cache`)v8.4.0 引入、实验态、默认关闭(源码 `EnableInstancePlanCache=false`；默认上限官方文档为 120 MiB，与源码 100 MiB 存在数值冲突，见 §6.2)，7.5.x 不含该特性；节点替换后 plan cache 冷启动是已知的性能塌陷点。
7. 二者是不同维度的工程取舍而非先进性排序：TiDB 以网络跳转与会话迁移复杂度换计算弹性与秒级故障替换；OceanBase 以计算存储绑定换节点内零网络跳转(函数调用 vs RPC)与强一致恢复。无状态的代价集中在网络跳转、事务协调、session 语义、cache 命中率与内存放大，需经压测与故障注入验证，而不能被"stateless"一词掩盖（推测）。

## 6.12 参考文献

[1] TiDB Architecture. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tidb-architecture/.
 （支撑:§6.1/§6.2/§6.4 "TiDB Server 是无状态 SQL 层、不存数据、只做计算、转发请求到 TiKV/TiFlash、经 TiProxy/LVS/HAProxy 横向扩展"的核心定义。）
[2] SQL FAQs / DDL — TiDB Development Guide. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/sql-faq/.
 （支撑:§6.2/§6.8/§6.9 schema lease(DDL lease 45s)、Information schema is out of date、至多两个相邻 schema 版本并存与 online DDL）
[3] Temporary Tables. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/temporary-tables/.
 （支撑:§6.2/§6.8/§6.10 本地/全局临时表"仅存 TiDB 内存、不持久、本地表会话可见"的语义。）
[4] SQL Prepared Execution Plan Cache. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/sql-prepared-plan-cache/.
 （支撑:§6.2/§6.7/§6.9 prepared plan cache 默认 session 级、v6.1.0 默认开启、temporary table 不进缓存、@@last_plan_from_cache 验证。）
[5] TiProxy Overview. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tiproxy-overview/.
 （支撑:§6.2/§6.6/§6.8/§6.10 TiProxy 连接迁移/保持能力、graceful-wait-before-shutdown 与长事务/cursor/本地临时表/意外宕机的迁移限制。）
[6] TiDB 8.5.0 Release Notes. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/release-8.5.0/.
 （支撑:§6.2 instance plan cache 版本归属反查证(v8.4.0 引入、实验态、默认关闭)与 TiProxy(v8.0.0 GA)演进的版本核对。）
[7] session-manager 设计文档(pingcap/tidb docs/design). 官方设计文档[EB/OL]. https://github.com/pingcap/tidb/blob/master/docs/design/2022-07-20-session-manager.md.
 （支撑:§6.2/§6.6/§6.8 "哪些会话状态可迁移、活跃事务/本地临时表/表锁/advisory lock/游标不可迁移"的设计依据。）
[8] Integrated Architecture of OceanBase Database. 官方博客[EB/OL]. https://oceanbase.github.io/docs/blogs/showcases/integrated-architecture.
 （支撑:§6.3/§6.7 OBServer 一体化、"节点内函数调用、节点间 RPC"、standalone/distributed 同一代码一体化的描述。）
[9] OceanBase System Architecture(v4.2.5_CE / V4.3.5)与 shared-storage 文档. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001970960.
 （支撑:§6.1/§6.3/§6.4 shared-nothing 每节点具备 SQL/storage/transaction engine，以及 SS 模式 RW/RO 节点、SSWriter 上传增量/基线到对象存储、com）
[10] ODP Session Status Synchronization. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-odp-doc-en-10000000002135951.
 （支撑:§6.3/§6.8/§6.10 一个客户端连接对应多个 OBServer server connection、需同步 session status 的论点。）
[11] OceanBase Paetica: A Hybrid Shared-nothing/Shared-everything Database. 论文，VLDB 2023[EB/OL]. https://www.vldb.org/pvldb/vol16/p3728-xu.pdf.
 （支撑:§6.3/§6.4/§6.8 OBServer 单机/分布式一体化设计动机、"单机无额外开销"的论证(V4.0 实现)。）
[12] OceanBase Bacchus: a High-Performance and Scalable Cloud-Native Shared Storage Architecture. 论文，arXiv:2602.23571,2026-02[EB/OL]. https://arxiv.org/abs/2602.23571.
 （支撑:§6.3/§6.7/§6.8 shared-storage 形态通过 shared service-oriented PALF + 共享缓存 + 异步后台服务让计算节点趋向无状态；论文实验数据不得当生产事实。）
[13] pingcap/tidb 仓库(release-8.5 @ 67b4876bd57b). 源码[EB/OL]. https://github.com/pingcap/tidb/tree/release-8.5.
 （支撑:§6.2 schema lease 默认 45s（DefSchemaLease 常量 v8.3.0+ 特定、7.5.x 用裸字符串）、loadSchemaInLoop(lease/2)、schema_check）
[14] oceanbase/oceanbase 仓库(v4.2.5_CE @ e7c676806fda). 源码[EB/OL]. https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE.
 （支撑:§6.3 OBServer 一体化(ob_server.h 的 sql_engine_/session_mgr_/multi_tenant_/storage_env_ 同类成员、对外接口、构造期对象初）
## 6.13 信息可信度自评

本章的强证据来自三类。其一是官方文档明确：TiDB Server 无状态定义、schema lease(DDL lease)45s 默认、临时表内存语义、prepared plan cache session 级、TiProxy v8.0.0 GA 与连接迁移限制、instance plan cache 默认上限 120 MiB，以及 OceanBase shared-nothing 每节点 SQL/storage/transaction engine、ODP session status synchronization、shared-storage RW/RO/SSWriter，均有官方文档直接表述。其二是源码级信息：`loadSchemaInLoop` 续租、`schema_checker.go` 的 Check 分支、`txnmanager.go`/`conn_stmt.go`、`SessionStates`/`ErrCannotMigrateSession`(`session:8146`)、instance plan cache 默认关闭常量，以及 OBServer `ob_server.h` 一体化成员与对外接口、`omt` 租户 MTL 注册事务/PALF/LS 与 `start_sql_nio_server`、`ob_basic_session_info.h`，均来自 commit-pinned checkout(TiDB release-8.5@67b4876bd57b、OceanBase v4.2.5_CE@e7c676806fda)并经 GitHub 反查证文件存在。其三是论文级信息：OceanBase 一体化设计动机(Paetica,VLDB 2023)、Bacchus 趋向无状态计算(arXiv,2026-02)，适合描述架构方向，不适合当作 classic CE 默认行为或生产 benchmark 事实。

工程推测/不确定：二者"不同维度取舍、无单一先进性排序"，以及"扩容 SQL 节点只解决计算饱和、不解决存储热点/事务冲突""共享服务瓶颈削弱 stateless compute 收益""session-heavy workload 削弱迁移红利"等，均为基于公开资料的工程判断（推测）。其余两项已落地：OBServer 多版本 schema 的刷新/失效语义已在 v4.2.5_CE/4.3.5/4.4.x checkout 源码确认(版本槽位缓存 + 心跳/SQL 触发的 `REFRESH`/`ASYNC_REFRESH` 任务 + `RELEASE`/`try_eliminate_schema_mgr` 回收，见 §6.3)，不再属待查；TiProxy 内部连接保持/重定向逻辑则仅确认到 TiDB 侧接口级(SessionStates 迁移)。唯一仍需进一步查证的是 Bacchus 让计算无状态的具体代码路径——社区版默认不编译 shared-storage，该路径未在本任务核对的 checkout 中落地。

反查证发现的冲突：① schema lease 实现的跨版本错配——默认值 45s 在 7.5.x/8.5.x 均成立(官方文档一致)，但 `DefSchemaLease = 45 * time.Second` 常量是 v8.3.0 才引入，7.5.x 与 v8.2.0 及更早均用裸字符串 `Lease: "45s"`，故本章弱化为"默认 45s"这一跨版本通用结论。② instance plan cache 默认上限数值冲突——源码 `DefTiDBInstancePlanCacheMaxMemSize = 100 * size.MB`(100 MiB)与官方 system-variables.md 的 `125829120`(120 MiB)矛盾，本章以官方文档 120 MiB 为准并标注冲突。③ instance plan cache 版本归属——经官方 release notes 与 system-variables.md 确认为 v8.4.0 引入、实验态、默认关闭，7.5.x 不含该特性。④ 会话迁移属 TiDB/TiProxy(Session Manager)，`ErrCannotMigrateSession`(`session:8146`)及不可迁移条件均为 PingCAP 事实，与 OceanBase 体系无关，不可跨系统混淆，并区分"直接拒绝"(本地临时表/表锁/advisory lock)与"须等待"(活跃事务/未读游标)两类机制。⑤ 源码 commit 统一基线——全章 TiDB 源码统一锁定 release-8.5、OceanBase 源码统一锁定 v4.2.5_CE，不与 v8.5.0、v4.4.2_CE 等其他 commit 双锁并存；Bacchus 论文版本号(arXiv:2602.23571)与提交日期(2026-02)与锁定头一致。其余事实多源一致，未发现实质矛盾。

---


# 第 7 章 存算分离

## 7.1 本章核心问题

「存算分离」(compute-storage separation / disaggregation)在云数据库语境里常被当作营销术语泛用，但它在工程上有严格内涵：**让计算节点不再持有权威的、不可重建的持久状态，而把数据、复制日志、元数据、事务状态、缓存与后台任务状态中的一项或多项外置到可独立扩缩、可被多个计算节点共享或快速重绑定的存储层**。换言之，真正要拆开的是五类状态：用户数据、复制日志、元数据、事务状态、缓存与后台任务。若只把 SSTable 或数据文件放到对象存储，而提交日志仍在计算节点本地、compaction 仍抢占前台 CPU、元数据/路由仍要求大规模数据迁移，那么系统在扩缩容、故障恢复和成本弹性上仍保留强耦合。

理解这一点后，本章要回答四个容易被混淆的问题。

1. **存算分离 ≠ 无状态 SQL 层。** TiDB Server 与 TiKV 进程分离、TiDB Server 无状态，这是「计算与存储进程分离」，但 TiKV 自身仍把 Region 数据落本地 RocksDB、用 Raft 复制三副本，是典型的 shared-nothing 存算一体。无状态 SQL 层只解决了「SQL 解析/优化/执行」这一层的弹性(详见第 6 章)，并没有让**存储层**变成可被任意计算节点即时接管的共享底座。
2. **「只把数据放对象存储」≠ 完整存算分离。** 数据、日志、元数据、事务状态、缓存五类状态各有外置代价；只外置冷数据(如备份、外部表)而在线读写仍走本地盘，只是「分层存储」，不是计算节点无状态化。一个系统只有在多项状态都从前台计算节点解耦时，扩缩容和故障恢复才真正从「搬数据」转向「重建工作集」（推测）。
3. **shared-storage 不是普遍更优的架构，而是一组取舍。** 对象存储带来近乎无限的容量、11 个 9 的持久性和按需扩缩，但访问延迟显著高于本地 NVMe(典型云对象存储 GET 首字节多在数十毫秒量级，具体随云厂商/区域/对象大小而异，官方未给出统一规格),IOPS 也有上限——这对延迟敏感、缓存命中率不稳定的 OLTP 而言是真实代价。
4. **TiDB 与 OceanBase 走了形态不同但目标趋同的路线。** TiDB 把存算分离先落在 **TiFlash 分析侧**(v7.0.0 实验性引入、**v7.4.0 GA**)，再通过 **TiDB X / TiDB Cloud** 把在线 KV 也搬上对象存储；OceanBase 则**自 4.3.5 LTS 正式引入** shared-storage 存算分离产品(代号 **Bacchus**)，并在 **4.4.x** 进一步完善(双架构并存、单副本租户)，把 SSTable 外置到对象存储、本地盘退化为多级缓存，日志用共享 PALF 日志服务承载。

为避免把「存算分离」写成单一标签，本章用如下「状态外置检查表」对齐四种形态。它也解释了为什么 Aurora、Socrates、PolarFS 只能作背景对照：它们都在不同层次上拆 compute、log、storage 或 file system，但 TiDB/OceanBase 的真实路径由各自的 Region/Raft、Tablet/Log Stream、PALF、TiFlash、Cloud service 边界决定，不能把外部系统的机制直接移植为本章事实。

**表 7-1　本章核心问题**

| 状态项 | TiDB classic | TiFlash disaggregated | TiDB X(公开资料) | OceanBase shared-storage |
|---|---|---|---|---|
| 用户数据 | TiKV 本地 RocksDB 保存 Region 数据 | 列存数据由 Write Node 上传 S3,Compute Node 远端读取并缓存 | 对象存储作为唯一真相源 | Tenant data 存放对象存储，本地缓存 hot data |
| 复制日志 | TiKV Region/Raft Group 本地复制与持久化 | 仍接收 TiKV Raft logs，不负责 OLTP 主提交 | Raft WAL 本地/EBS 持久 + 后台上传对象存储 | 共享日志服务化，副本经 PALF 共识 |
| 事务状态 | TiDB/TiKV 协同，primary lock、MVCC 与 Region 写入相关(详见第 10/11 章) | 主要服务一致性分析读，不改变 OLTP 提交 | Cloud 内部细节尚未公开到可逐项证明的程度（不确定） | 事务与 LS、SCN/GTS、共享日志协同，SS 下计算节点更易重建 |
| 缓存 | TiKV 本地 block cache / RocksDB cache 与节点绑定 | Compute Node 本地 NVMe 缓存是性能关键 | hot data local cache 是公开方向（官方暗示） | memory cache、本地持久缓存、分布式 macro 缓存三级设计 |
| 后台任务 | RocksDB compaction、Region 调度、GC 仍影响存储节点资源 | Write Node 管理 S3 数据组织，Compute Node 避免大部分存储维护 | compaction、DDL、statistics、backup 等重任务服务化（官方暗示） | 后台服务承担 upload、compaction、DDL、GC 等 |

本章的关键性在于：存算分离改变的是分布式数据库**最底层的状态归属**，进而牵动扩缩容速度、缓存命中、compaction 归属、故障恢复、成本与延迟。它与无状态 SQL 层(第 6 章)、shared-storage 与 K8s(第 23 章)、HTAP 列存(第 16 章)都直接相关，跨章只标注、不展开。 〔文献[8,11]〕

## 7.2 TiDB 的实现

TiDB 在「存算分离」这一议题上有**三个层次**，必须分开看，否则会把不同成熟度的形态混为一谈。

### 7.2.1 第一层：TiDB classic —— 进程分离但 TiKV 存算一体

TiDB classic(TiUP 自建 / TiDB Cloud Dedicated 托管)由 TiDB Server + TiKV + PD + 可选 TiFlash 组成。TiDB Server 暴露 MySQL 协议入口、解析 SQL、优化并生成分布式执行计划，官方文档称其不存储数据、只做计算与 SQL 分析，实际读请求发往 TiKV 或 TiFlash;PD 保存 TiKV 数据分布与拓扑元数据、分配 TSO 并向 TiKV 下发调度命令；TiKV 存储用户 KV 数据，Region 是 key range 的基本存储单元。其中 TiKV 在 release-8.5 仍是存算一体——Region 数据落本地 RocksDB,Raft 复制三副本，对象存储**不承载在线 KV 数据面**。

这是一种「组件级计算/存储分离」，不是 shared-storage。写路径上，TiDB Server 把 SQL 转换为对多个 Region 的 KV 读写；每个 Region 对应一个 Raft Group，数据、Raft 状态、RocksDB LSM、compaction 与 local cache 都在 TiKV 节点侧完成。扩 TiDB Server 很快，因为 SQL 层不持久化用户数据；扩 TiKV 则要经历 Region 调度、snapshot 复制、RocksDB 数据落盘与热点再均衡，不是简单地挂载远端共享数据。因此「TiDB Server 无状态」推不出「TiDB classic 完整存算分离」。

控制路径也体现这一边界。PD 可重新调度 Region replica、转移 leader、按 store 状态做均衡，但它调度的是存储节点上的副本布局，不是让任意 TiKV 立即访问同一份远端数据文件；当某个 TiKV 节点失效时，系统依赖剩余 Raft peer 继续服务并补副本，恢复成本与缺失 Region 数量、snapshot 传输、下游 compaction、网络与磁盘能力相关。因此 classic 的高可用是「多副本 shared-nothing 高可用」，不是「无状态计算节点重启即接管」。

源码事实印证了「能写对象存储不等于存算分离」:TiKV 仓库存在统一的 cloud blob 存储抽象(`components/cloud/`，含 `aws/src/s3.rs`、`azure/src/azblob.rs`、`gcp/src/gcs.rs`)与 `external_storage/` 组件，但在本批 checkout(`tikv/tikv @ release-8.5 @ 1f8a140b6d46`)核查范围内，这些组件主要服务于 **backup / log backup / BR / CDC** 等外置数据通道，**而非在线读写**。TiKV 能写 S3，但写的是备份，不是工作集。

> TiDB Cloud 当前以 Starter、Essential、Premium、Dedicated 等 plan 呈现：Starter 文档称为 fully managed、multi-tenant TiDB offering,Essential 支持自动调整存储与计算资源，Dedicated 仍允许自定义 TiDB/TiKV/TiFlash 规模。本章只引用产品层描述，把 Dedicated 看作托管 classic，把 Starter/Essential 看作公开文档描述的多租户服务形态，**不推断其内部租户隔离、调度池或数据放置实现**(高风险项，见 §7.13)。

### 7.2.2 第二层：TiFlash disaggregated —— 分析侧存算分离(v7.0.0 实验性 / v7.4.0 GA)

TiFlash 默认是 coupled storage and compute，即每个 TiFlash 节点既存储又计算。**自 TiDB v7.0.0 起以实验性形态支持 disaggregated storage and compute、并于 v7.4.0 GA**——v7.0.0 Release Notes 原文为「TiFlash supports the disaggregated storage and compute architecture and supports object storage in this architecture (experimental)」,v7.4.0 Release Notes 与特性文档则写明「the disaggregated storage and compute architecture for TiFlash becomes GA starting from v7.4.0」。该架构把 TiFlash 进程拆成两类节点：

- **Write Node(写节点)**：从 TiKV 接收 Raft 日志，转成列式格式，**周期性打包上传到 S3 或 S3 兼容对象存储**；同时负责 S3 上数据的组织、compaction 与 GC；用本地盘(通常 NVMe SSD)缓存最新写入数据，避免过度占用内存。
- **Compute Node(计算节点)**：执行查询时先从 Write Node 取数据快照，再从 Write Node 读取尚未上传的新数据，从 S3 读取其余历史数据，用本地 NVMe SSD 作数据文件缓存；**无状态、可秒级伸缩**——低峰期可缩到 0，高峰期秒级扩容。

**数据路径**:TiKV(行存/Raft)→ Write Node(列转换 + 本地缓存)→ S3(列式快照，权威副本)；查询时 TiDB Server → Compute Node → Write Node / S3 / 本地缓存。这解决的是 TiFlash AP 副本的存储/计算弹性，**不是 TiKV OLTP 路径的替换**：事务提交和行存主副本仍由 TiKV Region/Raft Group 维护。

**控制/路由路径**由源码印证(`pingcap/tidb @ release-8.5 @ 67b4876bd57b`):

- `pkg/config/config.go` 定义了一组 disaggregated-tiflash 开关字段：`DisaggregatedTiFlash bool`(toml `disaggregated-tiflash`,**默认 `false`**)、`UseAutoScaler bool`(toml `use-autoscaler`,**默认也为 `false`**)、`TiFlashComputeAutoScalerType`、`TiFlashComputeAutoScalerAddr` 等。**`use-autoscaler` 是与 `disaggregated-tiflash` 相互独立的开关，而非 disaggregated 模式的依赖。** `config.toml.example` 官方注释逐字写道：「use-autoscaler indicates whether use AutoScaler **or PD** for tiflash_compute nodes, only meaningful when disaggregated-tiflash is true.」——即 compute 节点的发现/调度在 AutoScaler 与 PD 之间二选一，且**默认走 PD**。`autoscaler-addr` 默认指向一个 K8s service(`tiflash-autoscale-lb.tiflash-autoscale.svc.cluster.local`),`autoscaler-type` 取值 `mock/aws/gcp`，表明外部 AutoScaler 是 **TiDB Cloud / Serverless** 路径的组件。self-managed/classic 私有化分离部署的官方文档(`tiflash-disaggregated-and-s3`)完全未提及 AutoScaler,compute 节点默认由 **PD** 发现、由 **TiUP**(`tiup cluster scale-out`)手动扩缩。故「classic 依赖外部 AutoScaler」属把 Cloud-only 机制错挂到 classic，本章不作此断言。
- `pkg/util/tiflashcompute/dispatch_policy.go` 定义任务分发策略 `DispatchPolicyRR`(`round_robin`)与 `DispatchPolicyConsistentHash`(`consistent_hash`)，同目录 `topo_fetcher.go` 拉取 compute 节点拓扑(AutoScaler 模式下从 AutoScaler 拉取，PD 模式下从 PD 获取)。**一致性哈希把同一数据的 cop task 稳定路由到同一 compute 节点，以提高本地缓存命中**——这是「计算无状态 + 缓存归属」的直接工程体现。
- `pkg/store/copr/batch_coprocessor.go` 按 `config.GetGlobalConfig().DisaggregatedTiFlash` 分支，在分离模式下用一致性哈希选 `tiflash_compute` 目标节点构建 `batchCopTask`，经 `cache.GetTiFlashComputeStores(...)` 获取 compute 节点列表。

约束也很关键：coupled 与 disaggregated TiFlash **不能在同一集群共存、不能原地切换**，迁移后 TiFlash 数据要重新复制；S3 模式下 TiFlash 文档还列出 **Encryption at Rest 不可用**的限制(计算节点拿不到非本节点文件的密钥)。这些限制说明「对象存储」带来成本与扩缩容优势，也带来迁移、加密、缓存命中和数据再同步代价(见 §7.8)。

> 边界提醒：TiFlash 引擎本体(Write Node 上传 checkpoint、S3 page store、Compute Node 拉取)位于独立仓库 `pingcap/tiflash`(可确认 `dbms/src/Flash/Disaggregated/`、`dbms/src/Storages/S3/`、`dbms/src/Storages/DeltaMerge/` 等模块级路径),**不在本批 checkout**，具体函数实现不作源码级断言，按官方文档标注。

### 7.2.3 第三层：TiDB X / TiDB Cloud —— 在线 KV 上对象存储

把**在线 KV 数据面**也搬上对象存储的是 **TiDB X**，文档把它描述为从 classic shared-nothing 演进到 cloud-native shared-storage。需要谨慎处理版本与产品归属：

- **TiDB Serverless / Cloud Starter 形态**:AWS Storage 官方博客、PingCAP「S3 is the new network」博客与第三方解读交叉描述：**SST 文件主体存 S3，增量 WAL / Raft Log 存 EBS 并经 Raft 复制，SST 在多个 TiKV 实例本地 NVMe 缓存，实现单位数毫秒读延迟**；因为「S3 的读写 IO 不在性能关键路径上」，用 S3 作共享存储层不损延迟；权威数据已在对象存储，故备份只依赖增量 Raft 日志与元数据，秒级完成。
- **TiDB X 架构(官方文档已上线)**:`docs.pingcap.com/tidbcloud/tidb-x-architecture/` 描述 TiDB X 把对象存储作为「**所有持久数据的唯一真相源**」;Raft 日志先持久化到本地盘、Raft WAL chunk 后台上传对象存储；含「new RF engine(Raft engine)与重新设计的 KV engine」;**每个 Region 独立一棵 LSM 树**、Region 默认 **256 MiB**；扩容时新 TiKV 节点不需从既有节点拷贝大量数据，而是连对象存储**按需加载**，称扩缩快 5×~10×；并把索引构建、大批量导入、compaction、统计、DDL 等后台重负载隔离到独立 elastic compute pool，以减少在线 OLTP 抖动。
- **产品/版本归属核查**:PingCAP 官方博客「The Making of TiDB X」明确 TiDB X **是驱动 TiDB Cloud 的核心引擎，不是用户自部署的独立产品或部署选项**，推荐入口为 TiDB Cloud Essential；架构文档称其「currently available in TiDB Cloud Starter, Essential, and Premium」。其 Virtual Clusters、S3 snapshots、stateless TiKV、logical shards 等说法来自官方博客与公开说明。本章按官方文档书写，**不把这些叙述写成自建 TiDB 9.0 GA 行为**，对于 Cloud Starter / Essential 的内部多租户隔离机制，官方仍未公开，**不做实现层断言**(高风险项，见 §7.13)（官方暗示）。

源码定位方面，锁定源码集含 `tidb @ release-8.5`、`tikv @ release-8.5`、`pd @ release-8.5`，可确认 classic 路径相关模块(TiDB `pkg/config/`、`pkg/store/copr/`,TiKV `components/raftstore/src/`、`components/engine_rocks/src/`、`src/storage/`);TiFlash 引擎本体在远端仓库可定位模块、具体函数需进一步查证(均 commit-pinned 见 §7.12)。 〔文献[1-4,12-14,16-17]〕

## 7.3 OceanBase 的实现

### 7.3.1 shared-nothing 基线(4.2.5 TP)与 shared-storage 的真实版本分界

**表 7-2　OceanBase 存算分离(shared-storage)版本能力时间线**

| 版本 | 能力 / 里程碑 | 形态 / 约束 |
|---|---|---|
| 4.2.5(TP LTS) | `ob_parameter_seed.ipp` 中 `_ss_*` 配置参数为 0 | 纯以 shared-nothing 为主 |
| 4.3.5(LTS) | 首次正式引入 shared-storage 产品(官方称 marks the official introduction of the shared storage database product)；已存在 `_ss_*` shared-storage 缓存参数 | 保留 shared-nothing 的同时引入 shared-storage；仅支持事务型实例 |
| 4.4.x | 把双架构能力完善；`_ss_*` 参数集大幅扩充(达 20+ 个) | 双架构并存可选、单副本租户 |

与 TiDB 把存算分离分作三个层次不同，OceanBase 从单一引擎切入。其传统形态是 **shared-nothing(SN)**：各节点地位平等，OBServer 一体化进程内同时承载 SQL engine、transaction engine、storage engine、PALF(Paxos-backed Append-only Log File system)日志与 Log Stream 存储，数据落本地、Paxos 复制多副本。应用通常经 OBProxy / ODP 连接，ODP 把 SQL 请求转发到合适的 OBServer。Paetica 论文(VLDB 2023)论证一体化架构在单机上「无额外开销」。这是 OceanBase 长期的工程取舍——**以计算存储绑定换取节点内零网络跳转**(函数调用而非 RPC)与强一致恢复。

OceanBase 的存储抽象不是 TiDB 的 Region，而是 Partition、Tablet、Log Stream/LS 三层：Partition 面向 SQL 表逻辑，Tablet 是存储层对象，LS 承载一个或多个 Tablet 的 redo logs 与复制边界。DML 修改 Tablet 时把 redo logs 记录到对应 LS,LS 与 Tablet 通常有多个副本分布在不同 Zone,leader 执行修改、follower 复制并重放，一致性由 Multi-Paxos 保证。因此 SN 形态的计算与存储没有像 TiDB Server/TiKV 那样拆成独立进程层，OBServer 是「计算+事务+存储」的复合节点，但其内部通过 Tenant、Unit、Resource Pool、Tablet、LS 做资源与复制边界隔离。

shared-storage 的真实版本分界需要厘清。`_ss_*` shared-storage 缓存参数的跨版本核查表明：

- **4.2.5_CE** 的 `ob_parameter_seed.ipp` 中 `_ss_*` 配置参数为 **0**;
- **4.3.5_CE 已存在 `_ss_*` shared-storage 缓存参数**(经 `raw.githubusercontent.com` 直拉 `v4.3.5_CE` tag 核实，含 `_ss_micro_cache_memory_percentage`「the percentage of tenant memory size used by microblock_cache in shared_storage mode」、`_ss_major_compaction_prewarm_level`、`_enable_ss_replica_prewarm` 等),**并非命中 0**;
- **4.4.x** 把 `_ss_*` 参数集大幅扩充(达 20+ 个)，并非首次出现。

独立佐证：OceanBase 官方称 **4.3.5 LTS「marks the official introduction of the shared storage database product」**(共享存储产品首次正式引入即在 4.3.5，该版本仅支持事务型实例)，与源码中 4.3.5 已出现 `_ss_*` 缓存参数完全吻合。故真正的版本分界是 **4.2.5 → 4.3.5**(shared-storage / Bacchus 在 4.3.5 引入)，而非 4.3.5 → 4.4.x。综上，**TP LTS 4.2.5 仍纯以 shared-nothing 为主**;4.3.5 在保留 shared-nothing 的同时**首次正式引入 shared-storage 产品**,4.4.x 则把双架构能力完善(双架构并存可选、单副本租户)。

### 7.3.2 shared-storage(Bacchus，4.3.5 引入 / 4.4.x 完善)

**表 7-3　shared-storage `_ss_*` 参数速查**

| 参数名 | 默认值 | 作用 | 版本 / 形态 |
|---|---|---|---|
| `_ss_micro_cache_memory_percentage` | 20 | shared_storage 模式下 microblock_cache 占租户内存大小的百分比 | 4.3.5_CE 已有 |
| `_ss_major_compaction_prewarm_level` | 0 | major compaction 预热级别 | 4.3.5_CE 已有 |
| `_ss_local_cache_expiration_time` | 0s | 本地缓存过期时间 | 4.3.5 后续 BP 新增 |
| `_ss_micro_cache_size_max_percentage` | 20 | micro cache 大小占比上限，范围 [1,99] | 4.4.x 扩充 |
| `_ss_local_cache_control` | — | 本地缓存控制 | 4.4.x 扩充 |
| `_ss_macro_cache_miss_threshold_for_prefetch` | 10 | 同一 macro block 本地缓存 miss 次数达此阈值即触发预取，范围 [1,10000] | 4.4.x 扩充 |
| `_ss_micro_cache_arc_limit_percent` | 70 | 基于 ARC 算法的淘汰触发盘占比，范围 [10,90] | 4.4.x 扩充 |

OceanBase 的 shared-storage 架构代号 **Bacchus**,**自 4.3.5 LTS 正式引入产品形态、4.4.x 完善双架构与单副本能力**。OceanBase shared-storage 文档称该模式采用 storage-compute separation，每个 Tenant 的数据和日志存放在共享对象存储，本地只缓存 hot data and logs。论文 *OceanBase Bacchus: a High-Performance and Scalable Cloud-Native Shared Storage Architecture for Multi-Cloud*(arXiv:2602.23571,2026-02，拟投 PVLDB Vol.20)以 **OceanBase 4.4.x** 为评测对象描述其设计与实验；源码在 `github.com/oceanbase/oceanbase`(`_ss_*` 参数自 4.3.5 起、4.4.x 扩充)。

**角色与组件。** 文档中的角色包括 RW node、RO node、SSWriter 与共享日志服务：RW 处理读写，RO 处理读，SSWriter 生成并上传 incremental 与 baseline data，共享日志服务提供专门的强一致日志；RO 可从最近的日志服务副本读取日志并在本地 replay hot data。这比「只把 SSTable 放对象存储」更完整，因为日志服务、缓存、上传、后台任务都被拆出前台计算节点。

**三级缓存 + 对象存储主存。** 论文(§2)把架构分两大块：**计算/缓存侧**含 Memory Cache(存最热的 micro-block)、本地持久缓存(本地云盘，作为数据持久化第一保障与写缓存，利用低延迟特性)、以及数据预取(prefetch)/预热(warming)机制；**共享侧**含分布式缓存(由 **Shared Block Cache Service** 提供，服务 macro-block，与前两级构成三级缓存)、共享日志服务(副本经 PALF 达成共识、跨集群并行、存元数据日志与数据日志、**消除日志冗余且免去对共享存储的实时 IO**、降低延迟)，以及后台任务(baseline compaction、DDL、备份恢复、GC)。

**stateless 计算节点 + 副本降为 1。** 论文摘要与 §2 称：对象存储承载全量数据，数据层可**降到单副本**(借对象存储自身持久性),**日志层仍多副本经 PALF 达成 RPO=0**；计算节点无持久本地状态、支持 serverless 与快速弹性。源码硬证据(跨版本对照):

- **4.3.5_CE**(shared-storage 首次正式引入)`ob_parameter_seed.ipp` 已含 `_ss_*` 缓存参数，如 `_ss_micro_cache_memory_percentage`、`_ss_major_compaction_prewarm_level`、`_enable_ss_replica_prewarm`，后续 BP 版本陆续新增 `_ss_local_cache_expiration_time` 等；
- **4.4.x** 把 `_ss_*` 参数集扩充至 20+ 个(`ob_parameter_seed.ipp` 中 `_ss_*` 唯一参数名 4.3.5_CE 为 9 个、4.4.x 为 28 个)，如 `_ss_micro_cache_size_max_percentage`(默认 **20**，范围 [1,99])、`_ss_local_cache_control`、`_ss_macro_cache_miss_threshold_for_prefetch`(同一 macro block 本地缓存 miss 次数达此阈值即触发预取，默认 **10**，范围 [1,10000])、`_ss_micro_cache_arc_limit_percent`(基于 ARC 算法的淘汰触发盘占比，默认 **70**，范围 [10,90])等；`ObSSLocalCacheControlMode`(`src/share/shared_storage/ob_ss_local_cache_control_mode.h`)以 `is_micro_cache_enable()`、`is_macro_read_cache_enable()`、`is_macro_write_cache_enable()` 三个方法区分 micro cache / macro 读缓存 / macro 写缓存三类，模式枚举字面值为 `MODE_DEFAULT=0b00 / MODE_ON=0b01 / MODE_OFF=0b10`,进一步印证「本地多级缓存 + ARC 淘汰 + 预取」是 Bacchus 的缓存归属机制。

上述参数默认值(`_ss_micro_cache_memory_percentage`=20、`_ss_micro_cache_size_max_percentage`=20、`_ss_macro_cache_miss_threshold_for_prefetch`=10、`_ss_micro_cache_arc_limit_percent`=70、`_ss_major_compaction_prewarm_level`=0、`_enable_ss_replica_prewarm`=True、`_ss_local_cache_expiration_time`=0s)、`ObSSLocalCacheControlMode` 模式枚举字面值与 `is_*_cache_enable()` 方法名均已在目标 commit 源码核实。

**对象存储后端接入。** 4.4.x `src/share/object_storage/` 提供设备配置管理(`ObDeviceConfigMgr`)、连通性检查、设备清单、zone 级存储表操作；`_object_storage_io_timeout`(对象存储 IO 超时，默认 **20s**，范围 [1s,1200s];4.3.5 与 4.4.x 默认一致)、`_object_storage_condition_put_mode`(条件写模式，默认 **"none"**，可选 `none / if-match`,4.4.x 新增)等参数让 shared-storage 数据面行为可观测可调。对照之下，4.3.5 已有对象存储接入框架、`_object_storage_io_timeout` 与 `_ss_*` 缓存参数(shared-storage 首发),4.4.x 在此基础上大幅扩充缓存控制与对象存储后端参数。

**写路径(论文 §2/§3)**：前台 RW 节点负责 SQL/事务执行，把提交所需 WAL 写入共享日志服务(日志先记本地云盘保低延迟，历史 CLog 文件迁移到对象存储)；一个 leader 负责写 PALF 日志服务；SSWriter 或后台服务把增量与基线数据组织到对象存储；RO 节点从共享存储 + 缓存读、并据日志/热数据 replay 维护本地可服务状态。

**版本与可用性。** OceanBase Cloud 文档称 shared-storage 与 shared-nothing 并存供选；**shared-storage 产品自 4.3.5 LTS 正式引入(仅事务型实例)**,**4.4.0 是支持双架构的重大升级**,V4.4.1 是 Shared Storage Architecture 的 second LTS 版本并增强共享日志服务横向扩展与 Azure Blob 支持，**单副本租户能力在 4.4.x 补齐**(SYS 租户单副本、ALTER TENANT 2F→1F)。单副本部署模式下，单实例只保一份全功能副本仍支持跨 zone 容灾，但 **RTO 为分钟级**(高于多副本模式，但显著降本)。这里须区分 Cloud、企业版、CE 与论文：公共头锁定 4.4.1_CE 与 4.4.2 LTS，本章按公共头处理 4.4.x,**不把 Cloud release、论文 Bacchus 与 CE 默认部署混为一谈**(社区版默认不编译 shared-storage)。

> 关于 RTO 数值的版本与场景边界：**RPO=0 / RTO<8s 仅适用于 OceanBase V4.x 在 LS/Paxos 多副本、少数派故障或多数派完整的场景**(RTO 由 v4.0 之前的约 30s 优化到 v4.0 的 <8s，系 v4.0 特性、详见第 18 章)，不绑定共享日志服务、不外推到单副本 shared-storage 或跨云。单副本部署 RTO 退化到分钟级，只适用于具备 shared-storage 单副本能力的 4.4.x，不可归属于 4.2.5(纯 shared-nothing)。`v4.4.1_CE` release notes 另提到 cos 驱动因不再支持 append 写而被移除、影响 shared storage 等模块——即 4.4.x 仍在演进，具体对象存储后端支持随版本变化。

源码定位方面，本地 `oceanbase @ v4.2.5_CE / v4.3.5_CE / 4.4.x` 可确认 `src/logservice/palf/`、`src/logservice/`、`src/share/shared_storage/`、`src/share/object_storage/`、`src/storage/macro_cache/`、`src/observer/` 等模块，并检索到 `is_shared_storage_mode`、SSWriter、共享日志、shared storage cache 等痕迹；但部分 shared-storage 具体实现仍含未展开的模块引用，本章只做模块级定位，不编造函数名与内部视图（不确定）。 〔文献[6-7,9-10,15]〕

## 7.4 核心差异对比

**表 7-4　存算分离:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB | OceanBase | 影响 |
|---|---|---|---|
| 在线 KV 默认形态 | classic:TiKV 存算一体(shared-nothing)，数据落本地 RocksDB + Raft 三副本 | 4.2.5:shared-nothing,OBServer 一体化 + Multi-Paxos;4.3.5 起 shared-storage 可选 | 二者**默认都不是存算分离**；存算分离均为可选/新形态 |
| 存算分离首发位置 | **分析侧**:TiFlash disaggregated,**v7.0.0 实验性引入、v7.4.0 GA**(Write Node + Compute Node + S3) | **统一引擎**:Bacchus shared-storage,**4.3.5 LTS 正式引入、4.4.x 完善**(对象存储承载 SSTable) | TiDB 先解决 OLAP 弹性；OceanBase 直接对 OLTP/HTAP 做 shared-storage |
| 数据划分/复制边界 | Region / Raft Group | Tablet 映射到 Log Stream/LS,LS 使用 Multi-Paxos | 两者都不是「全库共享一个日志」；扩缩容与热点迁移粒度不同 |
| 在线 KV 上对象存储 | TiDB X / TiDB Cloud:SST 上 S3、WAL 上 EBS/本地、本地 NVMe 缓存（官方暗示） | Bacchus:SSTable 上对象存储、本地盘退化为多级缓存、共享 PALF 日志服务 | 殊途同归：**数据外置 + 计算缓存 + 日志单独保障** |
| 日志归属 | Raft WAL 本地/EBS 持久 + 后台上传对象存储；Raft 保一致 | 共享日志服务(service-oriented)，副本经 PALF 共识，消除日志冗余 | OceanBase 把日志做成**共享服务**，更彻底解耦；TiDB 仍每节点 Raft |
| 缓存机制 | Compute Node 本地 NVMe 缓存 + 一致性哈希路由提命中(classic 下 compute 节点默认由 PD 发现);TiKV in-memory engine 加速热 Region(非存算分离) | 三级缓存(memory micro / 本地持久 / 分布式 macro)+ ARC 淘汰 + 预取/预热 | OceanBase 缓存层级更显式、参数更细粒度可控 |
| 数据层副本数 | 仍 3 副本(对象存储 11 个 9 之上叠 Raft)（官方暗示） | 可**降为单副本**(借对象存储持久性)，日志层多副本经 PALF 保 RPO=0 | OceanBase 单副本模式 RTO 退化到分钟级，换更低成本 |
| 故障恢复 | 节点失效：从 S3 拷 SST 到新节点(无需源节点参与) | 计算节点无状态、增量持久 + 共享日志重建，多副本 RPO=0；单副本 RTO 分钟级 | 都靠「数据已在共享层」加速替换；副本数决定 RTO 档位 |
| 适用边界 | classic 适合自建/托管强一致 OLTP/HTAP;TiFlash disagg 适合冷热明显、峰谷明显的 AP 副本；TiDB X 面向 Cloud 弹性（官方暗示） | SN 适合低延迟、高并发、本地盘强绑定；SS/Cloud/Bacchus 面向成本、弹性、多云与热冷分层 | shared-storage 不是单向升级，而是成本/弹性与尾延迟/缓存复杂度之间的取舍 |
| 源码可核查度 | 路由侧 commit-pinned 到 release-8.5；引擎本体在独立仓库 | `_ss_*` 参数/缓存模式 commit-pinned 到 4.3.5/4.4.x；论文数值不可作源码 | 引擎与论文数值都需「文档/论文」标注，非源码事实 |

## 7.5 正常路径图

图 7-1 刻意把 **TiDB classic OLTP**、**TiFlash disaggregated AP** 与 **OceanBase shared-storage** 三条正常路径分开，以避免把不同成熟度的「无状态」混为一谈：TiDB Server 的 stateless 只覆盖 SQL 层，TiFlash Compute Node 的 stateless 只覆盖列存分析副本，只有 OceanBase shared-storage 的 OBServer 才在 shared-storage 语境中向更无状态的计算节点移动。

![[f7_1.svg]]

**图 7-1　存算分离正常读写/调度路径**

OceanBase Bacchus 正常路径要点：写请求经 RW 节点 → WAL 写共享日志服务(本地云盘记录保低延迟，PALF 多副本共识)→ 事务状态推进、MemTable 与基线数据后台上传对象存储；读请求经 RO/RW 节点先查三级缓存(memory micro → 本地持久 → 分布式 macro，由 Shared Block Cache Service 提供)，未命中再读对象存储。官方对缓存效果以「冷热分层、命中率最大化」等**定性**描述为主，具体命中率百分比数值无法在官方公开来源核实，本章不作为 SLA 数值断言。

## 7.6 故障/异常路径图

存算分离把「数据已在共享层」变成故障恢复的优势，但也引入新的异常类型：**对象存储抖动、缓存未命中风暴、控制面/协调器不可用**。图 7-2 按路径类型展开，可与 §7.5 的正常路径对照阅读。

![[f7_2.svg]]

**图 7-2　存算分离故障/异常路径**

异常路径揭示了存算分离的真实成本。TiDB classic 的 Region leader 切换依赖 Raft 多数派、RegionCache 刷新与重试，这不是数据共享，而是多副本本地数据继续服务。TiFlash disaggregated 的 Compute Node 可更快替换，但若缓存冷、Write Node 或 S3 访问抖动，查询尾延迟会被放大；在「缓存未命中风暴」下(工作集大于缓存或无冷热区分),miss 比例上升使大量读穿透对象存储，P99/P999 显著抬升——OceanBase 端可由 macro cache miss 阈值触发预取(`_ss_macro_cache_miss_threshold_for_prefetch`，默认 10)、ARC 淘汰再平衡(`_ss_micro_cache_arc_limit_percent`，默认 70)缓解,TiDB 端靠一致性哈希尽量稳定路由保命中。控制面侧，TiFlash 分离的 AutoScaler 不可用只影响 Cloud 路径的弹性伸缩(退化为固定池、已起节点继续服务),classic 默认走 PD 不受影响；OceanBase 共享日志服务的 leader 失效则经 PALF 重选主 + LogReconfirm，多副本保 RPO=0、单副本数据层 RTO 退化到分钟级。这些都要按实际部署核查(见 §7.9)。

## 7.7 性能、可靠性、运维影响

**延迟。** 存算分离的核心代价是**多了一跳到共享存储**。对象存储访问延迟显著高于本地 NVMe(典型云对象存储 GET 首字节多在数十毫秒量级，尾延迟更高，具体随云厂商/区域/对象大小而异；官方未发布统一规格数值，本章按定性表述)。两家的应对都是「让对象存储不在关键路径上」:TiDB X / Serverless 称「S3 的读写 IO 不在性能关键路径上」，靠本地 NVMe 缓存达单位数毫秒读延迟；Bacchus 靠三级缓存最大化命中(官方仅有「命中率最大化/冷热分层」等定性表述，未发布具体命中率数值)。**结论性的工程事实是：延迟表现高度依赖缓存命中率**——命中即本地盘速度，未命中即对象存储速度，P99/P999 对 miss 比例极敏感；这一点不依赖具体百分比即成立。

**吞吐与成本。** 这是 shared-storage 最确定的收益方向，但下列数字须明确归属。以下 benchmark 数字**全部来自 OceanBase Bacchus 论文(arXiv:2602.23571)，是 OceanBase 4.4.x 对比 StarRocks / HBase 的论文实验结果，与 TiDB / TiFlash 无关**(该论文全文未把 TiDB/TiFlash 作为被测系统，仅作相关工作对照)。论文报告：对象存储比传统云盘便宜约 **85%**;Bacchus 相对存算一体存储成本降低 **OLTP 约 59% / OLAP 约 89%**。OLTP benchmark(SysBench / 多 HBase 变体对比，指定硬件):PUT 总 TPS 约 **51,612**(论文标注为各方案中最高)。OLAP(TPC-H 100GB / 指定实例)：比 StarRocks 查询执行快约 **48.5%**(仅 TPC-H 100GB),ClickBench 提速达约 **89%**。**论文同段亦给出反例，一并列出以免宣传化裁剪**:TPC-H 1TB 热跑仅「marginal improvement」、Q9 反而慢约 **35.71%**;ClickBench 部分查询 inferior to StarRocks；写吞吐与 HBase「roughly equivalent」(约 3,770 vs 3,993 ops/s)。这些数字必须连同硬件、版本(Bacchus = OceanBase 4.4.x、对比 HBase / StarRocks 3.3.13 等)、benchmark 工具与「approximately / best among all experiments」等限定一并理解，**是受控实验数据而非生产事实，不可外推为生产 OLTP 延迟、更不可挂到 TiDB 名下**。

> 兜底背景(全章仅此一处):OceanBase 历史上的 707M tpmC 系 TPC-C Result ID 120051701、OceanBase v2.2 Enterprise Edition、Alibaba Cloud ECS、1554 data nodes、2020 年(官方 FDR 记录日 2020-05-17)，不代表 4.x / CE / 小集群，更与本章 Bacchus 实验值无关。

**扩展性与故障恢复。** 这是 shared-storage 最具说服力的优势：**扩缩容不再需要在节点间拷贝大量数据**。TiDB X 称新 TiKV 节点连对象存储按需加载，扩缩快 5×~10×;TiFlash compute 节点秒级伸缩、可缩到 0；节点失效从 S3 拷 SST 到新节点即可。OceanBase 单副本 + 共享日志使数据层副本可降为 1、多副本经 PALF 保 RPO=0，但单副本 RTO 退化到分钟级。代价是冷启动依赖 snapshot/日志回放/cache warm-up，且共享日志服务、对象存储、SSWriter、cache metadata 的组合依赖都要纳入可用性与压测模型，不能只看进程是否存活。

**运维复杂度。** 存算分离把运维焦点从「数据迁移/再平衡」转移到「**缓存调优 + 对象存储 SLA 管理**」。新增可调维度：缓存大小百分比、预取阈值、ARC 淘汰占比、对象存储 IO 超时/条件写模式——既是可观测抓手也是新的调参负担；可观测面要同时覆盖前台、日志服务、对象存储、缓存与后台任务。下表从**性能瓶颈来源与故障路径**维度对比存算分离与 shared-nothing(第二张对比表)。

**表 7-5　性能、可靠性、运维影响**

| 维度 | shared-nothing(TiDB classic / OB 4.2.5) | 存算分离(TiFlash disagg / TiDB X / Bacchus) | 工程含义 |
|---|---|---|---|
| 读延迟决定因素 | 本地盘 + Raft/Paxos 往返(确定) | **缓存命中率**(命中=本地盘速度，未命中=对象存储数十 ms 量级，呈双峰) | 存算分离尾延迟对 miss 比例极敏感 |
| 扩容瓶颈 | 节点间数据再平衡拷贝(慢) | 连对象存储按需加载，**免拷贝**(快 5×~10×) | 存算分离扩缩弹性是其核心卖点 |
| 主要故障路径 | 副本失效 → Raft/Paxos 重选主补副本 | 计算节点崩溃(无状态秒级重建)+ **缓存未命中风暴** + 对象存储抖动 | 故障类型从「数据复制」转为「缓存重建 + 存储 SLA」 |
| 数据持久性来源 | 本地多副本(Raft/Paxos) | 对象存储 11 个 9 + 日志层多副本/Raft 保 RPO=0 | 持久性与可用性解耦，副本数可降 |
| 控制面新依赖 | PD / RootService(详见第 13 章) | TiFlash 分离：PD(classic 默认)或 AutoScaler(Cloud 路径);OB：共享日志服务 leader | 新增协调器即新增故障点 |
| 后台任务归属 | compaction/GC 与前台争抢本地 CPU/IO | 倾向后台服务化隔离，但引入调度/元数据发布/对象生命周期复杂度 | 重任务从「就地执行」转为「服务化协调」 |
| 不适合场景 | 超大冷数据(成本高、扩缩慢) | 延迟抖动敏感、无冷热区分的随机大工作集 OLTP | 二者各有禁区，非优劣排序 |

## 7.8 反例与代价

**1. 延迟敏感、缓存命中率不稳的 OLTP 不适合。** OceanBase 官方材料将 shared-storage 定位为可容忍一定延迟抖动的场景，单副本部署明确标注适合测试/学习/非关键业务，产品文档也建议用于低实时性要求、历史库、大写入量和简单查询。当工作集大于本地缓存、或访问模式无明显冷热区分时，缓存命中率下滑会让读穿透到数十毫秒量级的对象存储，P999 尾延迟剧烈抖动。**这是 shared-storage 的结构性代价，不是调优能完全消除的**(此为架构层面的确定性结论，不依赖具体延迟数值)。对于极热、极低延迟、更新密集且工作集接近全量的 OLTP，对象存储的低成本优势会被热 cache、日志服务、网络与一致性开销抵消，容量成本优势随之缩小（推测）。

**2. 「只把数据放对象存储」≠ 存算分离。** TiKV 的 cloud/external_storage 组件能写 S3，但只用于 backup/BR/CDC，在线读写仍存算一体——这是「分层存储」而非计算无状态化。同理，若只把冷数据归档到对象存储而工作集仍在本地盘，弹性收益有限。

**3. 数据局部性强、自建运维成熟的场景更适合 shared-nothing。** 数据局部性强、单租户规模稳定、自建团队成熟时，TiDB classic 或 OceanBase SN 可把数据、日志、compaction、cache 留在固定节点，问题定位直接、网络路径短；迁移到 shared-storage 后，故障面变成前台计算、日志服务、对象存储、缓存预热、后台服务与控制面的组合，观测与压测难度更高。**共享存储也成为新的争用与依赖点**:Bacchus 论文坦承 LSM-tree 类 shared-storage 的核心瓶颈正是「缺乏高效的跨节点共享日志机制」(直接持久化日志延迟高，日志复制又难兼顾一致性与扩展性)，其解法是把日志做成 PALF 共享服务，但这本身就是相当复杂的工程（推测）。

**4. 缓存一致性与后台任务归属的新复杂度。** shared-storage 不是把 compaction 变没了，而是改变谁执行、何时执行、如何避免影响前台。Bacchus 把 baseline compaction、DDL、backup、recovery 放到后台服务，是为了把重任务从计算节点拆开，但也引入后台调度、元数据发布、对象生命周期、GC 与 cache invalidation 的新复杂度。

**5. 单副本省成本的代价是 RTO 退化。** OceanBase 单副本 shared-storage RTO 是分钟级，适合测试/学习/非关键业务，**不适合高业务连续性要求的核心系统**(官方明确限定适用场景，RTO 口径见 §7.3.2)。

**6. 架构不可原地切换、合规/加密能力受限。** TiFlash disaggregated 与 coupled 不能原地互切，迁移需重新复制全部 TiFlash 数据，同集群不允许两种架构混存；且 disaggregated 模式不支持 Encryption at Rest(计算节点拿不到非本节点文件的密钥)——这类限制在金融、政企场景可能直接改变架构选择。OceanBase shared-storage 也必须区分 Cloud、企业版、CE 与论文，不应默认 CE 本地部署就具备 Cloud 产品的全部共享服务、对象存储与多云能力（不确定）。

**与替代方案对比。** 相对 shared-nothing(CockroachDB/Yugabyte/Spanner 路线),shared-storage(Aurora/Socrates/PolarDB/Neon/TiDB X/Bacchus)在 buffer pool miss 时多一跳网络延迟，但换来弹性与成本；业界 OLTP on cloud 的趋势确实在向 shared-storage 倾斜，但这是**云经济学驱动的取舍，不是「更先进」**。

## 7.9 测试开发视角的验证点

**先区分三条路径。** 功能测试应先把 TiDB classic OLTP、TiFlash disaggregated AP、OceanBase shared-storage 三条路径分开：

- **TiDB classic**：验证 TiDB Server 扩缩容时 session/事务/RegionCache 的重试边界，以及 TiKV scale-out 后 Region 调度与热点迁移是否符合预期。
- **TiFlash disaggregated**：开 `disaggregated-tiflash`，覆盖 Write Node 上传延迟、Compute Node 缓存命中/未命中、Write Node 不可达、S3 endpoint 抖动、Compute Node 秒级 scale-in/out(缩到 0 再拉起)、一致性哈希是否把同数据稳定路由到同 compute 节点、架构切换前必须清空旧 TiFlash replica 等场景。
- **OceanBase shared-storage**：覆盖 RW/RO 切换、共享日志服务多副本异常、SSWriter 上传滞后、对象存储延迟、RO replay 热数据、单副本跨 zone 容灾、Tenant 资源变化与 cache warm-up。

**把「前台成功」与「后台收敛」拆开。** 一次写入在前台提交成功后，TiFlash disaggregated 可能仍有上传、列式组织与 Compute Node 缓存刷新；OceanBase shared-storage 可能仍有基线数据上传、GC、cache 预热与 RO replay;TiDB classic 在 Region 调度后也可能存在 snapshot 接收、RocksDB compaction 与热点再均衡。因此验收脚本不能在 SQL 返回成功后立即结束，还应等待系统达到稳定观测窗口，再比较尾延迟、远端读比例、后台队列与错误日志是否回落（推测）。

**可注入的失效模式。**

- **对象存储延迟/错误注入**：模拟 S3/OSS 慢响应或 5xx，观察前台读写背压(OB 由 `_object_storage_io_timeout` 控制超时，默认 20s,范围 [1s,1200s])。
- **缓存未命中风暴**：清空本地缓存或换大于缓存的工作集，观察 P99/P999 抬升、预取是否触发(OB 4.4.x 由 `_ss_macro_cache_miss_threshold_for_prefetch` 等 macro cache 预取参数控制，默认 10、范围 [1,10000])。
- **Compute/Write Node 崩溃**：验证无状态重建与列存从 Raft 日志重放。
- **AutoScaler 不可用(TiFlash Cloud 路径)**：验证弹性退化为固定池、已起节点继续服务；classic 默认走 PD 不受影响。
- **共享日志服务 leader 失效(OB)**：验证 PALF 重选主 + RPO=0(机制详见第 3 章)。

**关键压测/观测指标。** 不要只看平均 TPS/QPS，应看 P95/P99/P999 latency、缓存命中率、远端对象存储读写量、日志提交延迟、后台任务队列时间、扩缩容完成时间、重试率与错误分类。其中：

- **缓存命中率**是 shared-storage 第一观测指标。TiKV in-memory engine 相关命中指标 **Region Cache Hit / Region Cache Hit Rate** 出自官方 **TiKV-Details Grafana Dashboard 的「In Memory Engine」分区**(`tikv-in-memory-engine` 特性页本身仅指向该 Dashboard 分区、并未在页面上列出该 metric 名);TiFlash compute 节点本地缓存命中需另行关注；OB shared-storage 端的 micro/macro 本地缓存命中率可经系统视图 `GV$OB_SS_LOCAL_CACHE`(单机版 `V$OB_SS_LOCAL_CACHE`，底层虚拟表 `__all_virtual_ss_local_cache_info`)查看,其列含 `CACHE_NAME`、`HIT_RATIO`、`TOTAL_HIT_CNT`、`TOTAL_MISS_CNT`、`HOLD_SIZE`、`ALLOC_DISK_SIZE`、`USED_DISK_SIZE`、`USED_MEM_SIZE`(该视图自 4.3.5_CE 引入,4.4.x 列集一致)。
- **观测项避免编造**:TiFlash 文档明确给出 `INFORMATION_SCHEMA.TIFLASH_REPLICA`、`ALTER TABLE ... SET TIFLASH REPLICA` 与 TiUP scale-out/scale-in 示例，可用于部署与复制状态验证；OceanBase 的 `GV$OB_SQL_AUDIT` 是 SQL 执行诊断视图(不等于合规意义上的 Security Audit)。其余 OB 参数名(`_ss_*`、`_object_storage_*`)的**存在性**已 commit-pinned 跨版本核查，本轮进一步在源码核实了关键参数的**默认值**(如 `_object_storage_io_timeout`=20s、`_ss_micro_cache_memory_percentage`/`_ss_micro_cache_size_max_percentage`=20、`_ss_macro_cache_miss_threshold_for_prefetch`=10、`_ss_micro_cache_arc_limit_percent`=70)、`ObSSLocalCacheControlMode` 枚举字面值与 `is_*_cache_enable()` 方法名,以及 shared-storage 本地缓存命中观测视图 `GV$OB_SS_LOCAL_CACHE`(列含 `HIT_RATIO`/`TOTAL_HIT_CNT`/`TOTAL_MISS_CNT`);仍无法在本批源码逐项确认的少数参数默认值/字面值,一律保留「需进一步查证」，不据记忆填数。

## 7.10 容易误解点

**误解 1：存算分离 = 无状态 SQL 层。** 错。无状态 SQL 层(TiDB Server)只是计算与存储**进程分离**；存算分离要求**存储层本身**变成可被计算节点共享/快速重绑定的底座。TiDB classic 有无状态 SQL 层但 TiKV 仍存算一体(Region 数据、Raft 日志、RocksDB LSM、compaction 状态都在 TiKV 侧)——它**不是**完整存算分离。

**误解 2:「数据上了对象存储」就是存算分离。** 错。要看是**工作集在线读写**走对象存储(+本地缓存)，还是只把**备份/冷数据**归档上去，且还要处理提交日志、事务状态、cache consistency、后台 compaction、元数据发布与故障恢复。TiKV 能写 S3，但写的是 backup/BR/CDC；只有 TiDB X / Bacchus 这类把在线 KV/SSTable 主存外置、并把日志与后台任务一并拆出的才是真正的存算分离。

**误解 3：存算分离一定更快或一定更慢。** 都不对。它把延迟从「确定的本地盘速度」变成「**取决于缓存命中率的双峰分布**」：命中=本地盘速度，未命中=对象存储(数十毫秒量级)。所以它对**有清晰冷热区分**的负载友好，对**随机大工作集 OLTP**不友好；性能是命中率的函数，不是一个固定结论。

**误解 4:TiDB X / OceanBase Bacchus / TiFlash disaggregated 可互相替换。** 错。TiFlash disaggregated 是 AP 副本路径；TiDB X 是 Cloud 下一代核心引擎方向(非自建独立产品);Bacchus 是 OceanBase 4.4.x / Cloud shared-storage 的论文与产品路线，需按版本和形态标注，不能混用。

**误解 5:Bacchus 的 benchmark 数字 = 生产延迟，或 = TiDB 成绩。** 双错。论文数字(约 59%/89% 降本、约 48.5% OLAP 提速等)有明确硬件/版本/工具前提，是受控实验、不能当生产 SLA，且论文同段还含 Q9 慢 35.71%、TPC-H 1TB 仅微弱提升、写吞吐与 HBase 大致持平等反例；这些是 OceanBase Bacchus(OceanBase 4.4.x)的成绩，论文未把 TiDB/TiFlash 作为被测系统，切勿挂到 TiDB 名下(见 §7.7)。

**误解 6:TiFlash 存算分离自 v7.0.0 就 GA、且 classic 依赖外部 AutoScaler。** 双错。其一，v7.0.0 仅**实验性**引入，**v7.4.0 才 GA**；其二，classic 分离模式下 compute 节点**默认由 PD 发现**,`use-autoscaler` 是独立开关(默认 false)、外部 AutoScaler 是 TiDB Cloud/Serverless 路径的构件，classic 不依赖它。

## 7.11 本章结论

1. **存算分离 ≠ 无状态 SQL 层，也 ≠「数据上对象存储」。** TiDB classic 有无状态 SQL 层，但 TiKV 在 release-8.5 仍是 shared-nothing 存算一体(cloud/external_storage 仅服务 backup/BR/CDC，非在线数据面)；真正要拆开的是用户数据、复制日志、元数据、事务状态、缓存与后台任务五类状态。
2. **TiDB 的存算分离先落在分析侧再扩到在线 KV。** TiFlash disaggregated **v7.0.0 实验性引入、v7.4.0 GA**(Write Node + Compute Node + S3，默认关闭；classic 下 compute 节点默认由 **PD** 发现，`use-autoscaler` 为独立开关、外部 AutoScaler 属 Cloud 路径；路由侧 commit-pinned)，但不改变 TiKV OLTP 主路径；TiDB X / TiDB Cloud 把在线 KV 也搬上对象存储(SST 上 S3、WAL 上 EBS/本地、Raft 一致、Region 256 MiB 独立 LSM)，官方文档已上线。
3. **OceanBase 自 4.3.5 LTS 正式引入 shared-storage(Bacchus)、4.4.x 完善，4.2.5 仍 shared-nothing。** `_ss_*` 缓存参数**自 4.3.5_CE 已出现、4.4.x 扩充至 20+**(跨版本源码对照，版本分界为 4.2.5→4.3.5);Bacchus 用三级缓存(memory micro / 本地持久 / 分布式 macro)+ ARC + 预取，以及共享 PALF 日志服务，数据层可降单副本(日志多副本保 RPO=0)。
4. **两家殊途同归：数据外置 + 计算缓存 + 日志单独保障。** 差异在日志归属(TiDB 每节点 Raft WAL vs OceanBase 共享日志服务)与缓存粒度(OB 的 micro/macro 三级 + ARC + 预取更显式)。
5. **存算分离的确定收益是成本与弹性，确定代价是延迟依赖缓存命中。** OceanBase Bacchus 论文(benchmark、非生产、且为 OceanBase 成绩非 TiDB)报对象存储便宜约 85%、降本 OLTP 约 59% / OLAP 约 89%(同段另有 Q9 慢 35.71% 等反例)；对象存储延迟为数十毫秒量级(官方未发布统一规格，命中率具体数值无法核实)，命中率高时才接近本地盘速度——该「延迟依赖命中率」结论不依赖具体数值即成立。
6. **shared-storage 不适合延迟抖动敏感、无冷热区分的 OLTP，且架构不可原地切换、单副本 RTO 退化到分钟级。** 这些是结构性代价、非调优可消除；核心代价还包括日志服务依赖、后台任务调度与更复杂的观测面。
7. **TiDB X 资料层级较锁定上下文已上升**(已有架构文档、被定位为「驱动 TiDB Cloud 的核心引擎」)，但 Cloud Starter/Essential 内部隔离机制仍未公开，故不做实现层断言（官方暗示）。

## 7.12 参考文献

[1] TiFlash Disaggregated Storage and Compute Architecture and S3 Support. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tiflash-disaggregated-and-s3/.
 （支撑:§7.2.2 Write Node/Compute Node 角色、v7.0.0 实验性引入 / v7.4.0 GA、S3 列式主存、秒级伸缩、本地 NVMe 缓存；§7.8 架构不可原地切换、S3-only、不）
[2] TiDB 7.0.0 / 7.4.0 Release Notes. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/release-7.0.0/.
 （支撑:§7.2.2 版本/状态：v7.0.0「experimental」首次引入、v7.4.0「become generally available」GA(硬证据)。）
[3] TiDB X Architecture. 官方文档[EB/OL]. https://docs.pingcap.com/tidbcloud/tidb-x-architecture/.
 （支撑:§7.2.3/§7.4/§7.7 对象存储作真相源、Region 256 MiB 独立 LSM、RF/KV engine、5×~10× 扩缩、workload isolation。）
[4] Select Your Plan. 官方文档[EB/OL]. https://docs.pingcap.com/tidbcloud/select-cluster-tier/.
 （支撑:§7.2.1 TiDB Cloud Starter / Essential / Premium / Dedicated 的产品边界。）
[5] TiKV MVCC In-Memory Engine + TiKV-Details Grafana Dashboard. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tikv-in-memory-engine/.
 （支撑:§7.4/§7.9 TiKV in-memory engine 是热 Region 缓存(非存算分离);Region Cache Hit / Region Cache Hit Rate metric 名出自 Grafan）
[6] OceanBase Cloud — Storage architecture(shared nothing / shared storage). 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-cloud-10000000002561170.
 （支撑:§7.3.2/§7.7/§7.8 shared-storage 单副本分钟级 RTO、独立扩缩、需容忍延迟抖动、RW/RO/SSWriter/共享日志角色(本轮该域名持续返回 HTTP 429，具体命中率/延迟数值未能自）
[7] OceanBase Cloud — Release notes for 2025 / Supported database versions. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-cloud-10000000001879252.
 （支撑:§7.3.2 shared storage product 正式引入(首版仅事务型实例)、V4.4.0 支持双架构、V4.4.1 为 second LTS 版本、单副本租户能力 4.4.x 补齐、成本表述边界。）
[8] OceanBase Database V4.3.5 LTS 发布说明 / 博客(shared storage product 首次正式引入). 官方[EB/OL]. https://en.oceanbase.com/blog/12062885632.
 （支撑:§7.1/§7.3.1/§7.3.2 4.3.5 LTS「marks the official introduction of the shared storage database product」——shar）
[9] OceanBase Database architecture / Shared storage architecture V4.3.5. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001971022.
 （支撑:§7.3.1/§7.3.2/§7.5 OceanBase SN/SS、Tenant、Tablet、Log Stream、RW/RO/SSWriter/共享日志与 hot cache 的官方实现描述。）
[10] OceanBase Bacchus: a High-Performance and Scalable Cloud-Native Shared Storage Architecture for Multi-Cloud. 论文，arXiv 2026-02，拟投 PVLDB Vol.20[EB/OL]. https://arxiv.org/abs/2602.23571(PDF:.
 （支撑:§7.3.2/§7.7/§7.8 三级缓存、共享 PALF 日志服务、stateless 计算节点、单副本 RPO=0、降本约 59%/89%、约 85% 便宜、SysBench/TPC-H/ClickBench 条件与）
[11] Amazon Aurora / Socrates / PolarFS. 论文[EB/OL]. https://www.amazon.science/publications/amazon-aurora-design-considerations-for-high-throughput-cloud-native-relational-databases.
 （支撑:§7.1/§7.8 背景对照：redo log 下推存储、log 作一等公民、durability 与 availability 解耦、shared-storage 文件系统路线；仅作外部系统对照，不移植为本章事实。）
[12] pingcap/tidb 仓库(release-8.5，另对照 v7.5.0). [EB/OL]. .
 （支撑:§7.2.2 DisaggregatedTiFlash 默认 false、UseAutoScaler 为独立开关亦默认 false(注释「whether use AutoScaler or PD ...）
[13] tikv/tikv 仓库(release-8.5,commit-pinned 1f8a140b6d46). [EB/OL]. .
 （支撑:§7.2.1 cloud blob 抽象与 external_storage 仅服务 backup/BR/CDC、非在线数据面。）
[14] pingcap/tiflash 仓库. [EB/OL]. https://github.com/pingcap/tiflash.
 （支撑:§7.2.2 TiFlash 引擎本体模块级定位；不在本批 checkout，具体函数需进一步查证。）
[15] oceanbase/oceanbase 仓库. commit-pinned:v4.2.5_CE @ e7c676806fda、4.3.5 @ b28b9bb12f3b、4.4.x @ d4bef8d29a4c[EB/OL]. .
 （支撑:§7.3.1/§7.3.2 _ss_* 缓存参数：4.2.5_CE 为 0,4.3.5_CE 含 9 个唯一 _ss_* 参数(含 _ss_micro_cache_memory_percentage 等,）
[16] How PingCAP transformed TiDB into a serverless DBaaS using Amazon S3 and Amazon EBS / S3 Is the New Network / The Making of TiDB X. 官方博客，AWS + PingCAP[EB/OL]. https://aws.amazon.com/blogs/storage/how-pingcap-transformed-tidb-into-a-serverless-dbaas-using-amazon-s3-and-amazon-ebs/.
 （支撑:§7.2.3 SST 主体 S3(11 个 9)、WAL/Raft Log 在 EBS、本地缓存、S3 不在关键路径、快速扩缩；TiDB X 是驱动 TiDB Cloud 的核心引擎(非独立产品)、推荐入口 Essent）
[17] Secrets Behind TiDB Serverless Architecture. 第三方解读，Medium[EB/OL]. https://dataturbo.medium.com/secrets-behind-tidb-serverless-architecture-8b277f00cb7c.
 （支撑:§7.2.3 单位数毫秒读延迟、增量 WAL 在 EBS 经 Raft、SST 缓存本地 NVMe、秒级备份(第三方，要点已与 AWS / PingCAP 官方博客交叉验证后采用)。）
## 7.13 信息可信度自评

本章证据分层如下。**源码级 commit-pinned 事实**(可信度最高):TiDB 的 `DisaggregatedTiFlash` / `UseAutoScaler` 默认值与分发策略(release-8.5，对照 v7.5.0，确认到文件级+字段名+默认值)、TiKV cloud/external_storage 模块用途、OceanBase 的 `_ss_*` 参数跨版本对照(**4.2.5_CE 为 0、4.3.5_CE 含 9 个唯一 `_ss_*` 参数、4.4.x 扩充至 28 个**，本批 clone 源码核实到文件级+参数名+关键默认值)、`ObSSLocalCacheControlMode` 三类缓存方法与模式枚举字面值、shared-storage 本地缓存命中观测视图 `GV$OB_SS_LOCAL_CACHE`(含 `HIT_RATIO`/`TOTAL_HIT_CNT`/`TOTAL_MISS_CNT` 列)。**官方文档级事实**:TiFlash disaggregated 架构与约束(v7.0.0 experimental / v7.4.0 GA,Release Notes 硬证据)、TiDB X 架构、OceanBase 4.3.5 LTS 首引 shared-storage、Cloud shared-storage 单副本 RTO 与版本归属。**论文级事实**:Bacchus 的三级缓存、共享 PALF 日志、单副本设计与全部 benchmark 数值——这些是 OceanBase 受控实验数据(非 TiDB、非生产事实)，已显式标注硬件/版本/工具并附反例。Aurora/Socrates/PolarFS 仅作背景对照。**工程推测/不确定**：存算分离的故障面与压测模型组合、TiDB X Cloud 内部隔离与 OB 内部视图等未公开细节，均标注边界、不做实现层断言。

诚实声明的边界：其一，TiFlash 引擎本体(`pingcap/tiflash`)与 Bacchus 论文实现数值不在本批 checkout,S3 page store、checkpoint upload、Bacchus 内部数值一律用「官方文档/论文」标注；其二，OB `_ss_*` / `_object_storage_*` 关键参数的具体默认值、`ObSSLocalCacheControlMode` 枚举字面值与 `is_*_cache_enable()` 方法名,以及 shared-storage 本地缓存命中视图 `GV$OB_SS_LOCAL_CACHE` 的列名,本轮已在 commit-pinned 源码逐项核实并升级为确定事实;仅余少数未在本批源码出现的参数默认值/字面值仍标「需进一步查证」，不据记忆填数；其三，对象存储延迟与缓存命中率的**具体数值**无法在官方/源码/论文中核实(且 en.oceanbase.com 本轮持续 429)，本章相关数值表述保留定性并标「需进一步查证」；其四，内部 Prometheus metric/PromQL 裸名无法核查者同样写「需进一步查证」，不编造(OB shared-storage 本地缓存命中已可经系统视图 `GV$OB_SS_LOCAL_CACHE` 查得,见 §7.9)。

版本核查备注：本章版本号匹配公共头锁定值——**TiFlash disaggregated v7.0.0 实验性 / v7.4.0 GA**、TiDB 8.5.x(对照 7.5.x)、TiKV/PD release-8.5、**OceanBase shared-storage 自 4.3.5 LTS 引入 / 4.4.x 完善**、TP 4.2.5_CE、Bacchus(arXiv:2602.23571,OceanBase 4.4.x)。TiDB X 既有官方文档又有博客/Medium、资料层级不一致，本章不把它写成自建 GA 部署、也不写 Starter/Essential 内部隔离实现；OceanBase supported versions 页面含 4.5.0/4.6.0 等更新内容，本章按公共头锁定的 4.4.x 书写，不用后续版本覆盖公共头。

---


# 第 8 章 性能

## 8.1 本章核心问题

性能是分布式数据库最容易被宣传化、也最容易被误读的维度。本章核心问题不是「TiDB 与 OceanBase 谁更快」，而是：在分布式架构下，性能由哪些底层路径决定；两套系统各自把性能优势压在哪里、把代价留在哪里；公开 benchmark 在多大程度上能映射到生产。

分布式数据库的性能不是单一标量，必须拆成相互正交的几类分别讨论，它们的关键路径、瓶颈来源彼此独立，不能简单排序成一条「谁更快」的序列：

- OLTP 性能：高并发短事务的吞吐与 P99/P999 延迟，核心瓶颈在一次写入要经过多少次时间戳获取（TSO/GTS）、多少轮共识日志多数派往返、热点分片、锁等待与写入限速（write stall）。
- AP / 分析性能：大查询的扫描吞吐与并行加速，核心在列存、向量化、MPP/PX 并行调度、统计信息与算子下推（优化器细节详见第 12 章，HTAP 架构详见第 16 章）。
- HTAP 混合性能：TP 与 AP 同集群混跑时彼此的干扰幅度，核心在物理/资源隔离手段。
- 弹性扩容性能：加节点后达到新吞吐的速度，核心在数据搬迁量与 Region / Log Stream-LS / Tablet 再均衡调度，以及后台 compaction/merge 的推进。
- 故障恢复性能：节点失效后恢复服务的 RTO 与对在线流量的抖动，核心在 leader 切换、日志追赶、路由缓存失效与控制面恢复速度。

本章只讲性能来源与代价，不做「谁更快」的总榜。任务是把每一类的底层路径讲清楚，并对每一个 benchmark 数字标注其 workload、硬件、版本、部署形态、调优方式与利益相关方。涉及 TSO/GTS、PD、RootService 时，本章严格遵循「逻辑中心化 / 性能瓶颈 / 高可用依赖 / 元数据调度依赖 / 故障恢复依赖」多维拆分，不写「单点」（与第 1、13 章保持一致）。

先把 benchmark 边界声明清楚。TiDB Cloud v8.5 的 TPC-C 报告给出的是 TiDB Cloud Dedicated 环境、`go-tpc tpcc`、1000 warehouses、不同线程下的 tpmC，并明确 tpmC 取 NewOrder 结果，属厂商自测文档，不是 TPC 官方审计成绩。OceanBase 707M tpmC 则是 Ant Financial 2020 年 TPC-C 官方审计成绩，能说明 OceanBase 在大规模、强一致 OLTP benchmark 下做过极限工程验证，但不能外推为 v4.x/CE 在任意硬件与业务上的生产延迟（其完整审计身份与适用边界全章只展开一次，见 §8.10）。

--- 〔文献[7]〕

## 8.2 TiDB 的实现

**表 8-1　TSO 攒批（batch）优化前后量化对照（§8.2）**

| 指标 / 机制 | 攒批前 | 攒批后 |
|---|---|---|
| TSO 请求量 | 约 16.2 万/s | 约 6.48 万/s |
| PD leader CPU 利用率 | 约 4600% | 约 1400% |
| PD server TSO handle time（P999） | 2ms | 0.5ms |

TiDB 的性能由三段路径叠加决定：无状态 SQL 层（TiDB Server）→ 时序服务（PD/TSO）→ 共识存储层（TiKV/Multi-Raft over RocksDB），分析侧另有 TiFlash 列存 + MPP。一个读写事务的客户端可见延迟可拆成解析/编译（parse/compile）、TSO 等待、定位 key 所属 Region、悲观锁加锁往返、prewrite + commit 两阶段提交（每阶段经 Region 的 Raft 多数派持久化）。TiDB 官方《Latency Breakdown》把延迟拆为 Token wait（通常 <1ms 可忽略）、Parse、Compile、TSO wait、Execute 等段。其中 TSO wait 与 Raft commit 是分布式特有的两段加价：每个事务至少要向 PD 取一次时间戳，每次写至少要一次 Region 内多数派往返。

TSO 路径——逻辑中心化而非「单点」。TiDB 的全局时间戳由 PD leader 统一分配。源码层面，`pkg/tso/tso.go` 定义常量 `maxLogical = int64(1 << 18)`（262144），即单个物理毫秒内逻辑位上限，决定单 PD/TSO 节点每毫秒可分配的 TS 数量级（pingcap/pd · release-8.5 · commit `6dce4a68e3e9`）。为摊销跨节点 RPC，客户端侧做批量化：`client/clients/tso/dispatcher.go` 的 `tsoDispatcher` 用 `tsoBatchController` 在 `maxBatchWaitInterval` 窗口内攒批，一次 RPC 取多个 TS。

这套设计在高并发短事务下会成为真实的吞吐压力点。PingCAP 官方《Best Practices on Public Cloud》给出了结论与量化数据：当 QPS 超过约 100 万、TSO 请求超过约 16.2 万/s 时，PD leader 的 CPU 利用率达到约 4600%（即逼近耗尽），TSO allocation 长尾、Region 信息请求与调度压力会叠加；开启 `tidb_tso_client_batch_max_wait_time` 客户端攒批后，TSO 请求量降至约 6.48 万/s、PD leader CPU 由约 4600% 降至约 1400%，PD server TSO handle time 的 P999 由 2ms 降至 0.5ms（文档原文「the CPU utilization reaches approximately 4,600% on the PD leader... the TSO requests per second decreased to 64,800, CPU utilization significantly reduced from approximately 4,600% to 1,400%, and the P999 value of PD server TSO handle time decreased from 2ms to 0.5ms」）。自 v8.2.0 起 PD 还支持把 TSO 微服务与调度微服务分离部署以进一步扩展。因此 PD/TSO 应按以下维度理解：逻辑中心化（集群级单一时间源）+ 性能瓶颈风险（高并发取时间戳、Region 信息请求、调度压力）+ 高可用（etcd/Raft 对称多副本，leader 失效自动选举）+ 故障恢复依赖（窗口上界持久化 etcd 防回退，详见第 13 章），而非物理单点。

写路径瓶颈集中在三处：Region 热点、Raft apply、RocksDB write stall/flow control。

- Region 热点。自增行 ID 或单调主键会把写集中到单个 Region/单个 TiKV 节点形成热点。官方文档给出 `AUTO_RANDOM`（替代 `AUTO_INCREMENT`，生成随机唯一值，文档原文「AUTO_RANDOM is often used in place of AUTO_INCREMENT to avoid write hotspot in a single storage node caused by TiDB assigning consecutive IDs」）与 `SHARD_ROW_ID_BITS`（打散 row ID 到 2^n 个区间）两种缓解手段。建表时还可用 `PRE_SPLIT_REGIONS` 预拆 Region，预拆出的 Region 数为 2^(PRE_SPLIT_REGIONS)，把初始写入摊到多个 Region 上以避免单 Region 热点。Region 拆分阈值由 `components/raftstore/src/coprocessor/config.rs` 定义（tikv/tikv · release-8.5 · commit `1f8a140b6d46`）：`region-split-size` 在 raftstore v1 锁定基准 8.5.x 下默认 `256MiB`；该默认值自 v8.4.0 起由 `96MiB` 提升（官方 tikv-configuration-file 文档：「Before v8.4.0, the default value is `"96MiB"`」），故对照基准 7.5.x 仍为 `96MiB`，这是版本口径差异而非冲突。Partitioned-Raft-KV（v2）默认 `RAFTSTORE_V2_SPLIT_SIZE = 10GiB`，`region_max_size` 默认为 `region_split_size / 2 * 3`。拆分速度跟不上写入热点，是热点瓶颈的来源之一。
- Raft apply。写入命中 Region leader 后，leader 追加 Raft log、复制给 follower，多数派确认后进入 apply——把已提交日志落到状态机、更新 MVCC/Region 状态。apply 是写延迟链条里一个明确的工程阶段而非协议理论中的独立步骤：若 apply worker 被大批写入、snapshot、compaction 或 CPU 抢占拖慢，用户侧会看到尾延迟上升。apply 与 store 线程池规模直接限制 apply 吞吐：`components/batch-system/src/config.rs` 中 `BatchSystemConfig::default()` 为 `pool_size: 2`，raftstore `Config::default()` 中 apply/store 池默认各 2 线程、`store_io_pool_size: 1`（官方 tikv-configuration-file 文档亦确认 `apply-pool-size`/`store-pool-size` 默认值均为 2）。
- RocksDB write stall / flow control。`src/config/mod.rs` 中各 CF 默认 `level0_slowdown_writes_trigger: 20`、`level0_stop_writes_trigger: None`、`soft_pending_compaction_bytes_limit: 192GB`、`hard_pending_compaction_bytes_limit: None`。L0 文件数或 pending compaction bytes 超阈值即触发写减速/停写。关键工程取舍是：TiKV 在 `storage.flow_control.enable` 为真时会禁用底层多个 CF 的 RocksDB write stall，改在 scheduler 层做反馈控流（官方配置文档亦说明这一替代）。其性能含义是 TiKV 不把后台压力完全交给底层库自行 stall，而尝试在更高层做反馈控制；代价是若上游持续高写入、compaction 追不上，系统转为排队/限流/抖动，吞吐看起来更平稳，但 P99/P999 仍可能恶化。

AP/HTAP 路径与 OLTP 路径分开看。TiFlash 是 TiKV 的列式扩展，通过 Raft Learner 异步复制 Region 数据，并以 Raft index 与 MVCC 校验提供 Snapshot Isolation；优化器在计划选择列副本时把 AP 查询下推到 TiFlash MPP（CH-benCHmark 即混合 TPC-C/TPC-H 的 HTAP 测试，TP 数据在线写入 TiKV、最新数据复制到 TiFlash）。优势来自行列分离与资源拆分，而非 TiKV 行存突然适合大扫描；代价是复制延迟、TiFlash 副本存储成本、MPP 调度与统计信息失准时的计划风险。TiFlash 存算分离（disaggregated）架构于 v7.0.0 实验性引入、v7.4.0 GA（详见第 7、16 章），不在本章 8.5.x OLTP 主线展开。

扩容性能的核心是 PD 调度与 Region split/merge。v8.5 性能高亮提到默认 Region size 从 `96MiB` 提升到 `256MiB`（自 v8.4.0 起，8.5.x 沿用），并给出小 Region merge 速度与 TiKV 扩容时长的改善测试结果；Tune Region Performance 文档也说明 Region 过多会带来元数据/调度的资源消耗与性能回退。这里要谨慎：更大的 Region 可降低元数据与调度开销，但也可能加重热点 Region 内部的扫描/写入集中度，尤其在单调递增主键、热点索引或小范围高并发更新场景下。

8.5 的几项可量化性能演进（数值来自 TiDB Cloud v8.5 Performance Highlights / Release Notes，TiDB v8.5.0 于 2024-12-19 GA）：TiKV MVCC In-Memory Engine 在热点更新场景下使查询延迟降约 50%、吞吐升约 100%（官方实测：启用前 1498.3 QPS/80.76ms avg、启用后 2881.5 QPS/41.99ms avg，硬件 TiDB 16vCPU×1 + TiKV 16vCPU×6）；针对云盘 IO 抖动引入 leader 提前 apply 已提交但未持久化日志的优化，IO 抖动场景下 P99/999 延迟最多降约 98%；instance 级 plan cache（v8.4 引入）。

涉及的源码/文档：`pkg/tso/`、`client/clients/tso/`（pd · release-8.5@`6dce4a68e3e9`）；`components/raftstore/src/coprocessor/config.rs`、`components/batch-system/src/config.rs`、`src/config/mod.rs`（tikv · release-8.5@`1f8a140b6d46`）；TiDB《Latency Breakdown》《Best Practices on Public Cloud》《PD Microservices》《TiFlash Overview》《8.5.0 Release Notes》。

--- 〔文献[1-4,9-11]〕

## 8.3 OceanBase 的实现

**表 8-2　两侧源码级默认参数对照（§8.2 / §8.3）**

| 维度 / 参数 | TiDB 默认 | OceanBase 默认 | 说明 / 影响 |
|---|---|---|---|
| 写减速触发 | level0_slowdown_writes_trigger = 20 | writing_throttling_trigger_percentage = 60（设 100 关闭） | 功能等价的写限速：内存/L0 写压过大时主动减速，代价是写延迟上升 |
| 停写触发 | level0_stop_writes_trigger = None | — | TiKV 默认不设硬停写阈值 |
| pending compaction 软限 | soft_pending_compaction_bytes_limit = 192GB | — | 超阈值触发写减速 |
| pending compaction 硬限 | hard_pending_compaction_bytes_limit = None | — | TiKV 默认不设硬上限 |
| minor freeze 触发 | — | freeze_trigger_percentage = 20 | memstore 占比达此值触发 minor freeze |
| 合并 / freeze 时间窗口 | — | major_freeze_duty_time = 02:00 | OceanBase 每日低峰做 major merge，可错峰；TiDB compaction 本地自治、无全局窗口 |
| 写限速最长持续 | — | writing_throttling_maximum_duration = 2h | 写限速持续时间上界 |
| Region 拆分阈值（raftstore v1） | region-split-size = 256MiB（基准 8.5.x；7.5.x 仍为 96MiB） | — | 拆分速度跟不上写入即成热点来源 |
| Region 拆分阈值（Partitioned-Raft-KV v2） | RAFTSTORE_V2_SPLIT_SIZE = 10GiB | — | v2 默认拆分尺寸 |
| apply / store 线程池 | apply-pool-size = 2、store-pool-size = 2、store_io_pool_size = 1 | — | 池规模直接限制 apply 吞吐 |
| 单物理毫秒逻辑位上限 | maxLogical = 1<<18（262144） | — | 决定单 PD/TSO 节点每毫秒可分配 TS 数量级 |

OceanBase 是一体化进程（OBServer 内含 SQL/事务/PALF 日志/存储/多租户），性能路径更内聚在 OBServer 内部：客户端经 ODP 路由到合适 OBServer 后，SQL、事务、日志、存储与资源隔离都围绕 Tenant、Partition、Tablet、Log Stream-LS 展开。官方架构文档说明：一个 OBServer 上可有多个 Unit，每个 Unit 可持有多个 LS 的副本，每个 LS 可包含多个 Tablet；LS 使用改进 Paxos 持久化 redo log，leader 在多数派 replica 持久化后确认日志。这条路径把事务参与单位、日志复制与本地存储调度更靠近内核；代价是热点 Partition、热点 LS、Tenant 资源配置与 major merge 计划都会影响同一个 OBServer 内部的延迟。

OLTP 数据路径与 GTS。OceanBase 的全局时间戳是 GTS，按租户隔离部署、采用 client-server 模型：每租户每节点有一个 GTS client 处理本节点请求。源码层面，核心类 `ObGtsSource`（`src/storage/tx/ob_gts_source.h` L69，oceanbase/oceanbase · v4.2.5_CE · commit `e7c676806fda`）提供 `get_gts(...)` 与 `refresh_gts(...)`；本地缓存 `ObGtsLocalCache` 用 `srr_`/`latest_srr_` 管理时间戳，`no_rpc_on_road()` 判断是否已有在途 GTS RPC 以做请求合并；内部维护 `get_gts_cache_cnt_` 等计数作为性能观测点。即 GTS 走「本地缓存 + 异步 RPC 刷新 + 在途请求去重 + 批量重试」模型，以此摊薄短事务取时间戳的开销。

在此之上还有一条关键工程短路：单机/单 Log Stream-LS 事务自 4.0 起免 GTS RPC、走单机提交协议而非分布式 2PC。对金融级高并发短事务，大量事务落在单 LS 内，这条短路把分布式时序与两阶段提交的代价从最常见路径上拿掉，是 OceanBase OLTP 优势的主要来源之一。与 PD/TSO 一样，GTS 必须多维拆解：逻辑中心化（租户级时间源，GTS leader 位于对应租户）+ 性能瓶颈风险（高并发取时间戳，靠本地缓存/批量/去重摊销）+ 高可用（GTS 依赖 `__all_dummy` Leader，Paxos 多副本）+ 故障恢复依赖（GTS leader 切换影响该租户事务时序），不可写「单点」。

写路径的几处瓶颈——PALF 日志延迟、major merge 抖动、写限速、多租户竞争——可以把写延迟拆成日志生成、PALF/Paxos 多数派持久化、replay/apply、checkpoint/compaction 后台推进，而不是只说「Paxos 快或慢」：

- PALF 日志延迟。写事务需 PALF（Paxos-backed Append-only Log File system）多数派持久化，整体是复制式 WAL + 事务状态推进的模型。`src/logservice/palf/` 含 `LogGroupBuffer`（`log_group_buffer.h` L26）、`election/` 选举子目录、`fixed_sliding_window.h` 等；`src/logservice/palf/log_engine.h` 暴露落盘接口 `append_log(const LSN &lsn, const LogWriteBuf &write_buf, const share::SCN &scn)`（L166，另有数组重载 L167）及 `read_log`/`truncate`（L168/L173）和 `submit_flush_log_task`/`submit_flush_*_meta_task`/`submit_truncate_log_task` 等 flush log/meta 异步任务接口（L134–158）；多数派确认在 `log_sliding_window.h`：follower 回 ack 由 `ack_log(const common::ObAddr &src_server, const LSN &end_lsn)`（L260）处理，`get_majority_match_lsn(LSN &majority_match_lsn)`（L236）与内部 `get_majority_lsn_(...)`（声明 L421，实现 `log_sliding_window.cpp` L2855）按 member list 计算多数派匹配位点，`try_advance_committed_end_lsn(const LSN &end_lsn)`（L297）据此推进 committed end；`src/storage/ls/ob_ls.h` 聚合 log handler、replay、apply、checkpoint 等结构。跨 Zone 副本同步的网络/磁盘延迟即「Paxos 日志延迟」，与 TiKV Raft commit 同类。
- Major merge IO 抖动。OceanBase 每日凌晨低峰做一次 major compaction（全量重写 SSTable），是 IO/CPU 抖动的主要来源。源码 `ObDailyMajorFreezeLauncher::try_launch_major_freeze()`（`src/rootserver/freeze/ob_daily_major_freeze_launcher.cpp` L122）触发，合并调度落在 `ObTenantTabletScheduler` + DAG。官方博客直言合并「对 I/O 和 CPU 施压，业界 LSM-Tree 数据库都无法回避该问题」，并描述了凌晨合并后删历史数据线程导致的延迟毛刺。
- 写限速（write stall 类比）。`src/share/parameter/ob_parameter_seed.ipp` 中的真实默认值——`major_freeze_duty_time = "02:00"`、`freeze_trigger_percentage = 20`（memstore 占比触发 minor freeze）、`writing_throttling_trigger_percentage = 60`（达此占比触发写限速，设 100 关闭）、`writing_throttling_maximum_duration = "2h"`。这与 TiKV 的 L0 slowdown/stop 是功能等价的同类机制：内存写压过大时主动限速避免失控，代价是写延迟上升。
- 多租户资源竞争。OMT（One Machine Multi-Tenant）按 Unit 的 min/max CPU + token 隔离租户（`ObTenant`，`src/observer/omt/ob_tenant.h` L371，提供 `set_unit_max_cpu`、`cpu_quota_concurrency` 等）。官方多租户线程 FAQ 说明 OceanBase 通过 Tenant resource pool 的 CPU 规格限制活跃线程，并在较新版本实现 cgroup-based isolation；其内部隔离强度与 Cloud 形态的具体边界不宜据此反推。热点租户挤占即多租户竞争瓶颈（多租户细节详见第 17 章）。

AP/HTAP 路线与 TiDB 不同：OceanBase 走同一套内核内融合，列存/CS_ENCODING 自 v4.3.0 引入，4.3 官方博客描述了列存、列存索引、向量化执行、物化视图与实时分析能力，并指出 TPC-H 1T 在 4.3 相比 4.2 提升约 25%；但 TPC-H 不涉及大宽表，未完全发挥列存引擎。优势是减少外部同步链路；代价是 TP 与 AP 共享硬件与 Tenant 资源，隔离依赖 cgroup、IOPS、调度策略与运维配置。

故障恢复性能同样不能写成某个中心点决定一切。RootService、sys tenant、GTS provider 是元数据、调度或事务时间服务的关键依赖，可能影响租户创建、负载均衡、schema/资源变更与恢复推进；但用户事务是否中断，取决于相关 LS 的多数派、leader 切换、日志追赶与 ODP 路由刷新。RootService 应按五维度讨论：逻辑中心化、性能瓶颈风险、高可用多数派、元数据/调度依赖、故障恢复依赖，而非定性为单点（详见 §8.7、第 13 章）。

涉及的源码/文档：`src/storage/tx/ob_gts_source.{h,cpp}`、`src/logservice/palf/`、`src/storage/ls/ob_ls.h`、`src/rootserver/freeze/`、`src/storage/compaction/`、`src/share/parameter/ob_parameter_seed.ipp`、`src/observer/omt/`（oceanbase · v4.2.5_CE@`e7c676806fda`）；OceanBase Database Architecture 文档、多租户线程 FAQ、4.3 新特性博客、PVLDB 2022 论文、事务引擎/内存管理博客。

--- 〔文献[5-6,8,12]〕

## 8.4 核心差异对比

在分别梳理两套系统的性能路径与瓶颈来源后，下面按维度分别并置：表 8-3 对照优势来源与关键路径，表 8-4 对照瓶颈点。两表只陈述差异落点，不评高下——同一维度上的取舍各有代价。

**表 8-3 性能优势来源与路径主对比表**

| 维度 | TiDB（8.5.x） | OceanBase（v4.2.5_CE TP） | 影响 |
|---|---|---|---|
| OLTP 写入基本单位 | Region / Raft Group，强调大量 Region 自动拆分调度 | Log Stream-LS / Paxos Group，强调 LS 聚合 Tablet 降低复制组数 | TiDB 单元多而小、靠 PD 调度；OB 单元聚合、降低事务参与复杂度 |
| SQL 与存储耦合 | 无状态 SQL 层 + TiKV 存储层 | OBServer 内集成 SQL/事务/日志/存储 | TiDB SQL 层扩展直接；OB 内核协同强，资源隔离与故障定位更内聚复杂 |
| 时序服务 | PD/TSO 集群级单一，客户端批量摊销 | GTS 租户级隔离，本地缓存 + 在途去重 | 均非单点；OB 租户隔离时序，TiDB 跨租户共享 TSO |
| 短事务短路 | 1PC（单 Region）/ async commit | 单 LS 免 GTS RPC + 单机提交 | OB 在高并发短事务上摊销更彻底 |
| 共识日志 | Multi-Raft over RocksDB，apply 池默认 2 | Multi-Paxos over PALF（复制式 WAL + 事务状态推进） | 同类多数派往返，瓶颈点不同 |
| 写限速机制 | RocksDB L0 slowdown(20)/stop + pending bytes(soft 192GB)，可切 flow control | writing_throttling(60%) + freeze(20%) | 功能等价，均为写延迟塌陷点 |
| 合并抖动 | RocksDB compaction 本地自治、无全局窗口 | 每日 major merge 全局协调（02:00） | OB 抖动集中可错峰，TiDB 分散难预测 |
| 多租户隔离 | Resource Control（RU 软隔离），v7.1 GA | OMT 原生硬隔离（Unit CPU/内存 + cgroup） | OB 内建，TiDB 靠 RU/K8s（详见第 17 章） |
| AP 引擎 | TiFlash 独立列存 + Raft Learner + MPP | 同内核列存（自 v4.3.0）+ 向量化 + PX 并行 | 组件化 vs 一体化融合 |
| 弹性扩容 | 加 TiKV → PD 再均衡 Region；classic 仍需搬数据 | 加节点 → Unit/LS/Tablet 再均衡，RootService 参与 | 均非零拷贝（TiDB X/Bacchus shared-storage 才趋零拷贝，详见第 7 章） |

**表 8-4 性能瓶颈来源对比表**

| 瓶颈类别 | TiDB 瓶颈来源 | OceanBase 瓶颈来源 |
|---|---|---|
| 时序获取 | TSO 高并发（PD leader CPU 逼近上限，阈值随硬件而异） | GTS 高并发（靠本地缓存/批量/去重缓解） |
| 热点 | Region 热点（单调主键）、热点 Store | 热点 Partition、热点 LS |
| 共识延迟 | Raft apply 池（默认 2 线程）、多数派往返 | PALF 多数派持久化、跨 Zone 同步 |
| 写抖动 | RocksDB write stall（L0/pending bytes）/ flow control | major merge IO 抖动 + writing_throttling |
| 控制面 | PD 调度（hot_region scheduler，rank v2 默认）、Region heartbeat | RootService 调度、major freeze 协调、sys tenant |
| 多租户 | 无内建，靠 RU + K8s | OMT Resource Pool 超卖/热点租户挤占 |
| 跨边界事务 | 跨 Region 2PC + 多次 TSO/网络往返 | 跨 LS 树形 2PC |

> 注：`hot_region` 调度器（pd · release-8.5，`pkg/schedule/schedulers/hot_region.go`，`hotScheduler` 结构 L212）默认 `RankFormulaVersion = "v2"`、`MinHotByteRate = 100`、`MinHotKeyRate = 10`。

---

## 8.5 正常路径图

图 8-1 给出两套系统的 OLTP/HTAP 主干路径，只表达调用关系与网络边界，不能按箭头数量直接推断性能（二者都通过缓存、batch、pipeline、异步复制减少路径成本）。

![[f8_1.svg]]

**图 8-1　性能正常读写/调度路径**

两条正常路径的共性是：都把「不跨边界」事务短路（TiDB 1PC 单 Region / OceanBase 单 LS），跨边界时把客户端可见延迟压到约一次多数派往返（详见第 10 章）。差异在时序获取——TiDB 每个事务都要向集群级 PD 取 TSO（靠批量摊销），OceanBase 单 LS 事务则可完全免 GTS RPC。TiDB 的关键网络边界在 TiDB Server、PD、TiKV、TiFlash 之间；OceanBase 的关键边界更多在 ODP 到 OBServer、LS leader 到 follower、Tenant 资源与后台任务之间。

---

## 8.6 故障/异常路径图

异常路径的性能损失通常表现为 P99/P999 而非平均吞吐。图 8-2 把两套系统的两类典型扰动（IO 抖动/合并 + leader 失效）合到一张图。

![[f8_2.svg]]

**图 8-2　性能故障/异常路径**

两套系统的故障恢复哲学不同：TiDB 偏惰性（RegionError 驱动客户端刷新路由、惰性 resolve lock），若 PD 同时承压，Region 信息与 TSO 请求会叠加抖动；OceanBase 偏主动状态机驱动（leader 重选重放日志，详见第 10 章），若 major merge 与前台 TP 同时竞争 IO/CPU，Tenant 隔离配置就成为关键变量。在 IO 抖动这一项上，TiDB 8.5 的 leader 提前 apply 优化是针对云盘场景的专门工程改进。OceanBase 在 V4.x LS/Paxos 多副本、少数派故障或多数派完整场景下的恢复指标见 §8.7（不外推到单副本或 shared-storage/跨云形态）。

---

## 8.7 性能、可靠性、运维影响

延迟。OLTP 短事务延迟下界由时序获取 + 一次多数派往返决定。OceanBase 单 LS 免 GTS 短路使最常见路径少一次时序 RPC，这是其金融级短事务延迟优势的结构性来源；TiDB 靠客户端 TSO 批量化把时序开销摊薄，但跨 Region 事务仍要多次往返。两者尾延迟（P99/P999）都受写抖动主导：TiDB 是 RocksDB write stall/flow control + 云盘 IO 抖动（8.5 已显著缓解尾延迟），OceanBase 是 major merge 凌晨毛刺（可错峰但难完全消除）与 Tenant 资源竞争。

吞吐。OLTP 吞吐在大集群下都会触及时序服务上限：如 §8.2 所述，TiDB 的 PD/TSO 在高并发下 PD leader CPU 逼近饱和（具体阈值随硬件而异），需客户端批量化 + 微服务分离；OceanBase GTS 按租户隔离，单租户高并发同样受 GTS 吞吐约束。两侧吞吐增长还分别依赖 Region 分布/PD 调度/TiKV apply-store 线程/RocksDB 后台能力，与 Unit/LS/Partition 分布/Tenant 资源/日志盘数据盘/merge 调度。AP 吞吐由列存扫描 + 并行度决定（详见第 12、16 章）。

可用性与故障恢复。OceanBase 厂商口径为集群级 RPO=0、RTO<8s，但需带前提：RPO=0 依赖 Paxos 多数派，RTO<8s 仅在 V4.x LS/Paxos 多副本（分布式 ≥3 副本）、少数派 OBServer 故障或多数派完整的场景下成立，阿里云文档把它明确限定为「Test results show... RTO of less than 8 seconds」即特定测试条件下的测试结果/设计目标；独立同行评审来源（PVLDB 2022）给出的是 RTO<30s。该指标不绑定 RootService，也不外推到单副本、shared-storage 或跨云形态。其恢复按 LS 粒度并行——只有失主的 LS 需重选，其余继续服务。TiDB 靠 PD 检测 + Region leader 秒级重选，无状态 SQL 层可秒级替换（详见第 6 章）。两侧可靠性都应按多副本多数派 + 控制面恢复分析：TiDB 的 PD 是元数据/TSO/调度关键依赖，OceanBase 的 RootService/sys tenant/GTS 是关键依赖，均不写「单点」。

扩展性。TiDB classic 加 TiKV 后 PD 再均衡 Region，吞吐随节点近线性提升、P95 下降；v8.5 实测小 Region merge 速度与 TiKV 扩容时长改善（仍需看数据量与热点分布）。OceanBase 加节点后做 Unit/LS/Tablet 再均衡、RootService 参与调度，资源规划不当会让单 Tenant 或单 LS 成为瓶颈。两者 classic 形态扩容都需搬数据，真正的近零拷贝扩容要到 TiDB X / OceanBase Bacchus shared-storage（按版本与产品形态限定：OceanBase shared-storage/Bacchus/SSWriter 见 4.3.5+ 文档与 4.4.x 架构，社区版默认不编译；详见第 7 章）。

运维复杂度。TiDB 多组件（TiDB/TiKV/PD/TiFlash）独立扩缩、故障域清晰，但参数面广（RocksDB 调优、PD 调度），定位要跨多组件与 Grafana；OceanBase 一体化进程外观简洁，但 Tenant/Unit/LS/merge/cgroup/ODP 的内部关系更密，运维焦点在 major merge 错峰、租户 Resource Pool 配额、写限速参数，诊断模型更复杂。

Benchmark 局限性必须单列。TPC-C 规范本身说明 benchmark 结果高度依赖 workload、应用需求、系统设计与实现，不应替代客户应用的容量规划或产品评估。TiDB v8.5 Cloud TPC-C 报告为厂商自测，线程 200 时给出 103,395 tpmC；OceanBase 707M tpmC 为 2020 年 TPC 审计成绩，硬件与版本均与前者不在一个量级（完整审计身份见 §8.10）。两者不是同一硬件、同一版本、同一审计级别、同一调优边界，只能支撑「各自路径曾在特定条件下表现如何」，不能支撑「某产品在所有 OLTP 场景绝对更快」。

---

## 8.8 反例与代价

TiDB 的代价：

1. TSO 跨地域延迟：PD leader 集中分配时间戳，跨地域部署时远端事务取 TSO 要跨地域 RTT，这是 TiDB OLTP 在地理分散场景的结构性代价。
2. Region 热点塌陷：单调递增主键或热点二级索引不做 `AUTO_RANDOM`/`SHARD_ROW_ID_BITS`/`PRE_SPLIT_REGIONS` 会把写打到单 Region，吞吐塌到单节点水平。
3. RocksDB write stall：写入持续超过 compaction 能力时 L0 堆积触发 slowdown/stop（或转为 flow control 排队），延迟从毫秒级跳到秒级。
4. 跨 Region 事务：涉及多 Region 的事务要多次 TSO + 多个 Region Raft 往返，延迟显著高于单 Region。
5. AP 路径退化：AP 查询没有合适 TiFlash 副本或统计信息误导计划时，大扫描落回 TiKV，失去列存加速。

这些都不是「架构失败」，而是行存 OLTP、Region 自动拆分与组件化 HTAP 这一组取舍各自留下的代价。

OceanBase 的代价：

1. major merge 抖动：每日全量合并对 IO/CPU 施压，凌晨延迟毛刺；只能错峰、限速缓解，不能消除（LSM 通病）。
2. 多租户资源竞争：Resource Pool 超卖或热点租户挤占，TP/AP 混跑时若 cgroup/IOPS 隔离不足，共享 OBServer 上的其它租户受牵连。
3. 写限速塌陷点：memstore 达 `writing_throttling_trigger_percentage`（60%）主动限速，写延迟上升。
4. GTS/PALF 跨 Zone 延迟：跨 Zone 多数派持久化的网络延迟直接计入写延迟。
5. 跨 LS 事务比例高：提交路径变长（树形 2PC）；RootService/GTS/sys tenant 承压会影响元数据、版本或恢复推进；ODP location cache 失效后短时间重试放大。

OceanBase 一体化内核能减少跨组件链路，但也意味着错误资源规划会在同一 OBServer 内部形成连锁竞争。

取舍。两者无优劣，只是把复杂度放在不同位置：TiDB 用组件化 + 客户端批量化换取弹性与云原生，更易横向扩展 SQL 层与替换 AP 路径，代价是把时序、热点、compaction 的调优与跨组件观测暴露给运维；OceanBase 用一体化 + 单 LS 短路换取短事务延迟与硬隔离，更易做事务/日志/存储协同优化，代价是 major merge 抖动、多租户竞争与内核内部调度复杂。

---

## 8.9 测试开发视角的验证点

可测功能场景应分层覆盖。OLTP 用 Sysbench oltp_read_write / oltp_point_select（常用 50/100/200 线程、1000 表 × 1 万行）；AP 用 TPC-H 22 query；HTAP 用 CH-benCHmark（TPC-C + TPC-H 混跑）；弹性扩容看加节点后吞吐爬升曲线；故障恢复看 kill leader 后 RTO。具体到两侧：TiDB 至少覆盖单 Region 热点写、预拆 Region 后高并发写、跨 Region 事务、PD TSO 高并发、TiKV compaction 压力、TiFlash 副本延迟、TiFlash 与 TiKV 计划切换、TiKV 扩容期间长跑；OceanBase 至少覆盖热点 Partition/LS、跨 LS 事务、Tenant CPU/IO 限额、major merge 期间 TP 延迟、ODP location cache 失效、LS leader 切换、AP 与 TP 混跑、Unit 扩缩容与资源池调整。

可注入失效模式：

- TiDB：制造 Region 热点（单调主键灌入）、触发 write stall（写速超 compaction）、kill TiKV Region leader 所在进程、注入云盘 IO 延迟、延迟或丢包 PD/TSO 请求压满 TSO、降低 TiFlash 副本同步能力。
- OceanBase：强制 major merge 并观测抖动、压到 writing_throttling 阈值、kill LS leader 所在 OBServer、延迟 ODP 到 OBServer 的路由刷新、制造 follower lag、对 sys tenant 或 RootService 相关操作施压、热点租户挤占。

注入后应观察吞吐、平均延迟、P95/P99/P999、重试率、错误码、leader 切换耗时、日志追赶时间与后台任务队列。

关键压测指标按 workload 区分：OLTP 看 tpmC/QPS/TPS、事务冲突率、lock wait、commit latency、Raft/Paxos 日志延迟；AP 看 QphH 与 query latency；HTAP 看 TP 写入受 AP 查询影响的延迟变化、列存副本同步延迟；扩容看数据迁移速度、balance 完成时间、热点消散时间；故障恢复看 RTO、重试风暴、leader election 与缓存刷新时间。

关键观测项采保守写法，只写已核验项，未核验者一律显式标注。TiDB 侧 `tidb_tikvclient_request_seconds{type='Get'}`、`tidb_tikvclient_txn_cmd_duration_seconds{type='lock_keys'}` 由《Latency Breakdown》确认；PD TSO handle time 的原始 metric 为 `pd_server_handle_tso_duration_seconds`（histogram，P999 经 `histogram_quantile` 计算；定义于 pd · release-8.5 · `server/metrics.go` L93-99，namespace `pd`/subsystem `server`/name `handle_tso_duration_seconds`）；RocksDB write stall 相关 metric 为 `tikv_engine_write_stall`、`tikv_engine_write_stall_reason`、`tikv_engine_stall_micro_seconds`（tikv · release-8.5 · `components/engine_rocks/src/rocks_metrics.rs` L1597/L1311/L1399），flow control 一侧为 `tikv_scheduler_throttle_flow`、`tikv_scheduler_throttle_action_total`、`tikv_scheduler_throttle_duration_seconds`（`src/storage/metrics.rs` L442/L484/L508），且 `tikv_engine_write_stall`、`tikv_scheduler_throttle_flow` 均出现在官方 TiKV-Details Grafana dashboard（`metrics/grafana/tikv_details.json`）中。OceanBase 侧 GTS 的 `get_gts_cache_cnt_`/`get_gts_with_stc_cnt_` 等内部计数为源码确认；合并/日志诊断视图可用 `GV$OB_COMPACTION_PROGRESS`、`GV$OB_TABLET_COMPACTION_HISTORY`、`GV$OB_COMPACTION_DIAGNOSE_INFO`、`GV$OB_COMPACTION_SUGGESTIONS`、`GV$OB_MERGE_INFO`、集群级 `CDB_OB_MAJOR_COMPACTION`/`CDB_OB_ZONE_MAJOR_COMPACTION`，PALF/LS 日志同步状态可用 `GV$OB_LOG_STAT`、`GV$OB_LOG_TRANSPORT_DEST_STAT`，内存与限速观测可用 `GV$OB_MEMSTORE`/`GV$OB_MEMSTORE_INFO`（均定义于 oceanbase · v4.2.5_CE · `src/share/inner_table/ob_inner_table_schema_constants.h`，schema 经 `OB_GV_OB_LOG_STAT_TNAME` 等创建）；Tenant 线程/cgroup 隔离强度、具体诊断命令参数仍需进一步查证。需要特别区分：`GV$OB_SQL_AUDIT` 是 SQL 执行诊断视图（用于定位慢 SQL/执行计划），不等于合规意义上的 Security Audit。为避免编造，本章正文不写未核实的 metric/视图/函数名。

---

## 8.10 容易误解点

1. 「OceanBase TPC-C 707 万…还是 7.07 亿？谁更快？」——该成绩是 707,351,007 tpmC（7.07 亿，约 707M），来自 TPC-C Result ID 120051701、OceanBase v2.2 Enterprise Edition、Alibaba Cloud ECS、1554 个 data nodes、2020 年（官方 FDR 记录日 2020-05-17）的 TPC 审计。它说明 OceanBase 曾在极大规模、严格审计、强一致 OLTP benchmark 上跑通，但这是 2020 年早期企业版 + 超大专用硬件的成绩，不代表 v4.x/CE 或普通小集群的生产性能，也不是与 TiDB 的同台对比；把它读成「OceanBase 比 TiDB 快」是典型误读。该兜底全章只展开这一次，其余位置以指代方式引用。

2. 「TPC-C 数字越大，生产 OLTP 一定越快。」——错。TPC-C 只能说明特定版本、硬件、配置、审计条件与事务模型下的表现。OceanBase 707M 是超大硬件、v2.2 企业版、TPC 审计；TiDB v8.5 Cloud TPC-C 是 Cloud Dedicated 自测（线程 200 时 103,395 tpmC）。二者审计级别、版本、硬件、调优边界都不同，都不是任意业务的性能承诺。

3. 「TSO/GTS、PD、RootService/sys tenant 是单点，所以是性能死穴。」——错。它们是逻辑中心化或关键依赖，配高可用多副本 + 批量/缓存摊销。真正风险是高并发取时间戳的吞吐压力与跨地域时序延迟，以及元数据/调度承压，而非「单副本失效即全挂」。必须按逻辑中心化/性能/高可用/元数据调度依赖/故障恢复多维分析（与第 1、13 章一致），简单写「单点」会误导故障模型。

4. 「Raft 与 Paxos 谁更先进决定性能。」——错。性能主要来自 Region/LS 组织、日志批量与 pipeline、apply/replay、存储引擎、调度与资源隔离。Raft/Paxos 名称本身不足以推导吞吐或延迟；TiKV Witness 与 OceanBase Arbitration 也不可简单类比，其投票/日志语义需各自验证后才能下结论（详见第 13 章）。

5. 「vendor benchmark 数字可直接比、且适用于锁定版本。」——不可，且存在版本错配。OceanBase「5–6×Greenplum 6.22.1」（个别查询最高 9×）仅出自 OceanBase v4.0（TPC-H 100GB，3×32 核 128GB），锁定基准 4.2.5/4.3.5/4.4.x 没有对 Greenplum 的对应基准（后续分析型基准改用 StarRocks/ClickHouse）；TiDB「MPP 2–3×（个别 7–8×）快于 Greenplum/Spark」出自 TiDB v5.x（对 Greenplum 6.15.0 + Spark 3.1.1，TPC-H 100GB），锁定基准 7.5/8.5 只有 TPC-C 报告、没有对应的 Greenplum/Spark TPC-H 报告，TiFlash 存算分离架构亦无此类对比发布。两组数字都是各自厂商自测、对手为数年前冻结的旧版本、硬件不同、被测库版本比基准新若干大版本，既不能横向相减，也不能外推到锁定版本当作生产事实。（推测）

---

## 8.11 本章结论

1. 分布式数据库性能必须拆成 OLTP / AP / HTAP / 弹性 / 故障恢复五类正交维度分别讨论，不能排序成单一「谁更快」；两套系统各自把优势压在不同路径、把代价留在不同位置，且一种 workload 下的优势不能自动外推到另一种。（推测）
2. OceanBase 金融级 OLTP 短事务优势的结构性来源是单 LS 事务免 GTS RPC + 单机提交短路 + GTS 本地缓存/在途去重，并依托 OBServer 一体化内核协同（Tenant/Unit/Partition/Tablet/LS/PALF/MemTable·SSTable/merge 联动）；代价是 major merge 抖动、多租户竞争与 GTS/RootService/sys tenant 依赖。
3. TiDB 的优势在云原生弹性、组件化 SQL 层扩展与 HTAP（TiFlash Raft Learner + MPP），OLTP 靠客户端 TSO 批量化摊薄时序开销；瓶颈常落在热点 Region、Raft apply（池默认 2）、RocksDB compaction/flow control、跨 Region 事务与 PD/TSO 压力（高并发下 PD leader CPU 仍是吞吐上限，阈值随硬件而异）。
4. 两系统的写抖动机制是功能等价的同类：TiKV RocksDB L0 slowdown(20)/stop + pending bytes(soft 192GB)（可切 flow control）对位 OceanBase writing_throttling(60%) + freeze(20%) + major merge；均为写延迟塌陷点。
5. 对 TSO/GTS、PD、RootService/sys tenant 不可写「单点」：均为逻辑中心化 + 高可用多副本 + 摊销机制，应拆成逻辑中心化、性能风险、高可用多数派、元数据/调度依赖与恢复依赖；风险在吞吐与跨地域延迟。
6. OceanBase 707M tpmC（TPC-C Result ID 120051701、v2.2 EE、Alibaba Cloud ECS、1554 data nodes、2020）是历史超大硬件审计成绩，不可外推 v4.x/CE 或生产，也不是与 TiDB 的同台对比；TiDB v8.5 Cloud TPC-C 为厂商自测，二者不可直接横向排名。
7. 所有 vendor benchmark（OceanBase v4.0 的 5–6×Greenplum 6.22.1、TiDB v5.x 的 2–3×Greenplum 6.15.0/Spark 3.1.1、OceanBase RTO<8s）均为厂商自测自述且版本错配，与锁定基准（OceanBase 4.2.5/4.3.5/4.4.x、TiDB 7.5/8.5）不对应，缺独立审计者一律弱化为厂商声明，benchmark 不映射生产。（推测）
8. 未核实的 Prometheus metric、内部视图、诊断命令、函数名与参数阈值一律写「需进一步查证」，本章不编造。（不确定）

---

## 8.12 参考文献

[1] TiDB Latency Breakdown. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/latency-breakdown/.
 （支撑:§8.2 OLTP 延迟分段构成（TSO wait / parse / compile / lock / commit）与可观测 metric（tidb_tikvclient_request_seconds、txn）
[2] TiDB Best Practices on Public Cloud / PD Microservices. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/best-practices-on-public-cloud/.
 （支撑:§8.2、§8.6 PD/TSO 高并发瓶颈结论与量化数据（QPS 约 100 万 / TSO 约 16.2 万/s 时 PD leader CPU 约 4600%；开启 tidb_tso_client_batch_m）
[3] TiDB Cloud Performance Highlights / TPC-C Report for TiDB v8.5.0. 官方文档[EB/OL]. https://docs.pingcap.com/tidbcloud/v8.5-performance-highlights/.
 （支撑:§8.2、§8.7 8.5 性能演进（MVCC IME 延迟降 50%/吞吐升 100%、IO 抖动 P99/999 降 98%、Region size、小 Region merge、TiKV 扩容改善）与 Cloud）
[4] TiDB 8.5.0 Release Notes / Tune Region Performance / TiFlash Overview / AUTO_RANDOM. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/release-8.5.0/.
 （支撑:§8.2 8.5 GA 日期（2024-12-19）、leader 提前 apply、instance plan cache、Region size 版本归属（v8.4.0 起 96MiB→256MiB，7.5.x 仍）
[5] OceanBase Database Architecture (V4.3.5) / FAQ about multi-tenant threads (V4.3.3) / Exploring OceanBase 4.3. 官方文档/博客[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001971022.
 （支撑:§8.3、§8.4、§8.7 OBServer/Unit/LS/改进 Paxos 多数派持久化、Tenant resource pool 与 cgroup 隔离、列存（自 v4.3.0）/向量化/物化视图/TP·AP 资）
[6] OceanBase Transaction Engine: Features and Applications. 官方博客[EB/OL]. https://oceanbase.medium.com/introduction-to-the-oceanbase-transaction-engine-features-and-applications-27159d10b9d2.
 （支撑:§8.3 GTS 按租户 client-server 部署、单机/单 LS 事务免分布式提交短路；major merge 对 I/O 与 CPU 施压的官方表述。）
[7] TPC Benchmark C Full Disclosure Report: Ant Financial OceanBase v2.2. 审计报告[EB/OL]. https://tpc.org/results/fdr/tpcc/ant_financial~tpcc~alibaba_cloud_elastic_compute_service_cluster~fdr~2020-05-17~v01.pdf.
 （支撑:§8.1、§8.10 OceanBase 707,351,007 tpmC（Result ID 120051701、v2.2 EE、Alibaba Cloud ECS、1554 data nodes、2020 审计），界）
[8] OceanBase: A 707 Million tpmC Distributed Relational Database System. 论文，PVLDB 2022[EB/OL]. https://vldb.org/pvldb/vol15/p3385-xu.pdf.
 （支撑:§8.3、§8.7 OceanBase 分布式事务/存储/日志设计与 RTO<30s 学术口径；不作为 v4.x/CE 生产性能证明。）
[9] How to Troubleshoot RocksDB Write Stalls in TiKV. 第三方解读，PingCAP 工程师署名 Medium[EB/OL]. https://pingcap.medium.com/how-to-troubleshoot-rocksdb-write-stalls-in-tikv-bbb04b88b935.
 （支撑:§8.2、§8.4 write stall 触发机制与阈值语义。局限：blog 列出的部分默认值（如 stop=36、soft 64GB）与 release-8.5 源码（stop=None、soft 192GB）不一致）
[10] TiDB Best Practices on Public Cloud（官方文档，PD/TSO 高并发量化数据）. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/best-practices-on-public-cloud/.
 （支撑:§8.2 PD/TSO 高并发量化数据（QPS 约 100 万 / TSO 约 16.2 万/s → PD leader CPU 约 4600%；开启 tidb_tso_client_batch_max_wait_ti）
[11] tikv/tikv（release-8.5 @ commit 1f8a140b6d46）— components/raftstore/src/coprocessor/config.rs、components/batch-system/src/config.rs、src/config/mod.rs、components/engine_rocks/src/rocks_metrics.rs、src/storage/metrics.rs、metrics/grafana/tikv_details.json；pingcap/pd（release-8.5 @ commit 6dce4a68e3e9）— pkg/tso/、client/clients/tso/、pkg/schedule/schedulers/hot_region.go、server/metrics.go. 源码[EB/OL]. https://github.com/tikv/tikv/blob/release-8.5/components/raftstore/src/coprocessor/config.rs.
 （支撑:§8.2、§8.4、§8.9 Region 拆分阈值（v1 默认 256MiB/v2 10GiB）、apply/store 池（默认 2）、write stall 默认值（L0 slowdown 20、soft）
[12] oceanbase/oceanbase（v4.2.5_CE @ commit e7c676806fda）— src/storage/tx/ob_gts_source.h、src/logservice/palf/log_engine.h、src/logservice/palf/log_sliding_window.{h,cpp}、src/storage/ls/ob_ls.h、src/rootserver/freeze/、src/share/parameter/ob_parameter_seed.ipp、src/observer/omt/ob_tenant.h、src/share/inner_table/ob_inner_table_schema_constants.h. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/blob/v4.2.5_CE/src/share/parameter/ob_parameter_seed.ipp.
 （支撑:§8.3、§8.4、§8.9 GTS（ObGtsSource/本地缓存/在途去重）、写限速默认值（freeze 20%/throttling 60%/merge 02:00）、major freeze、PALF/LS 模）
## 8.13 信息可信度自评

官方文档明确：TiDB 延迟分段（逐字确认）、8.5 性能数值、Region 热点缓解手段（`AUTO_RANDOM` 逐字确认）、TiFlash Raft Learner/Snapshot Isolation、OceanBase OBServer/LS/Paxos 架构与写限速/合并参数语义，均来自 docs.pingcap.com 与 OceanBase 官方文档及博客，可信度充分。PD/TSO 高并发瓶颈的具体数字（约 100 万 QPS / 16.2 万 TSO/s 触发 PD CPU 约 4600%，攒批后 TSO 约 6.48 万/s、PD CPU 约 1400%、PD server TSO handle time P999 由 2ms 降至 0.5ms）虽源页为客户端渲染 SPA、正文无法直接回取，但已据《Best Practices on Public Cloud》文档检索快照逐字核对，可信度据此确认。

源码级信息：Region 拆分阈值（`256MiB` v1 / `10GiB` v2）、apply 池规模（2）、RocksDB write stall 默认值（L0 slowdown 20、soft 192GB）、OceanBase GTS 类结构（`ObGtsSource`/`ObGtsLocalCache`）、写限速默认值（freeze 20%/throttling 60%/merge 02:00）、OMT `ObTenant`、PD `hotScheduler`/`maxLogical=1<<18`（=262144），均经 GitHub commit-pinned 逐字确认（TiKV release-8.5@`1f8a140b6d46`、PD release-8.5@`6dce4a68e3e9`、OceanBase v4.2.5_CE@`e7c676806fda`），其中 `maxLogical=1<<18` 与 apply/store 池默认 2 另经官方文档独立佐证。版本口径：Region 默认 `256MiB` 自 v8.4.0 起生效，本章主基准 8.5.x 适用，对照基准 7.5.x 仍为 `96MiB`，属版本差异而非冲突；单位用官方的「MiB」。PALF 的 append/多数派确认函数已源码确认到签名 + 行号级（`log_engine.h` 的 `append_log` L166-167；`log_sliding_window.h` 的 `ack_log` L260、`get_majority_match_lsn` L236、`try_advance_committed_end_lsn` L297、内部 `get_majority_lsn_` 声明 L421/实现 `log_sliding_window.cpp` L2855，均 oceanbase v4.2.5_CE@`e7c676806fda`）；TiDB oracle/TSO 客户端在 client-go，本仓未 clone，仅确认到依赖声明级。可观测 metric 与诊断视图名已升级：PD `pd_server_handle_tso_duration_seconds`、TiKV `tikv_engine_write_stall`/`tikv_scheduler_throttle_flow` 系列（release-8.5 源码 + 官方 Grafana dashboard 确认），OceanBase `GV$OB_COMPACTION_PROGRESS`/`GV$OB_TABLET_COMPACTION_HISTORY`/`GV$OB_LOG_STAT` 等（v4.2.5_CE inner_table schema 确认），见 §8.9。

论文/审计：OceanBase 707,351,007 tpmC 为 TPC 2020 官方审计（Result ID 120051701、v2.2 EE、Alibaba Cloud ECS、1554 data nodes），PVLDB 2022 论文佐证设计，可信度高，但只适用于历史超大硬件审计场景，不外推 v4.x/CE/小集群；TiDB v8.5 Cloud TPC-C 为厂商自测（非 TPC 审计）。

工程推测 / 不确定（交叉核验判证伪，已下调标签）：
- OceanBase RTO<8s：数值与版本本身可在 OceanBase/阿里云官方来源查到，但性质是厂商测试结果/设计目标（阿里云文档原文「Test results show... RTO of less than 8 seconds」），且带强前提（V4.x LS/Paxos 多副本、分布式 ≥3 副本、少数派 OBServer 故障）；独立同行评审来源（PVLDB 2022）给出 RTO<30s。本章按「厂商口径，带前提条件，非无条件生产事实」书写，不绑定 RootService、不外推单副本/shared-storage/跨云。
- OceanBase 5–6×Greenplum 6.22.1 出自 v4.0、TiDB MPP 2–3×Greenplum/Spark 出自 v5.x，均与锁定基准不对应，属版本错配 + 营销基准，弱化为厂商声明。
- TiDB Online DDL 的异步 schema change 与 Google F1 在线异步 schema change 思想一致，但本章不作「官方基于 F1」一类归因（该归因属工程推测，详见相关章节）。

交叉核验冲突：
1. write stall 默认值冲突：第三方 write-stall 博客列出 `level0-stop-writes-trigger=36`、`soft=64GB`、`hard=256GB`；release-8.5 源码实为 `level0_stop_writes_trigger: None`、`soft: 192GB`、`hard: None`。本章以源码为准。
2. OceanBase RTO 指标冲突：早期文档/PVLDB 2022 为 RTO<30s，厂商近期文档为 RTO<8s 且限定为测试结果/设计目标并带部署前提。本章不把 RTO<8s 当无条件生产事实。
3. benchmark 版本错配冲突：OceanBase「5–6×Greenplum 6.22.1」绑定 v4.0、TiDB「2–3×Greenplum 6.15.0/Spark 3.1.1」绑定 v5.x，均与锁定基准及 TiFlash 存算分离架构不对应。本章标注为版本错配 + 营销基准，不外推。

未核实的 Prometheus metric、OceanBase 内部视图、诊断命令、未展开函数名与参数阈值均标为「需进一步查证」，本章正文不编造这些细节。未发现与版本基准必须改写的其它冲突；与基准不完全一致的外部信息均按锁定值保守书写。

---


# 第 9 章 分库分表替代

## 9.1 本章核心问题

在单机 MySQL/Oracle 的容量、写入吞吐、单表 DDL、热点行列与故障半径触顶之后，过去十余年互联网业务的主流解法是「应用层分库分表」：在数据库之外引入一个中间件层（Apache ShardingSphere、MyCat、Vitess，或公司自研的 sharding SDK），由它根据分片键（sharding key）把一张逻辑大表水平拆成成百上千张物理子表，再分散到多个数据库实例上。中间件负责 SQL 改写、路由、结果归并、分布式事务补偿与扩容迁移；应用则被迫长期感知分片拓扑。一个典型形态是按 `user_id % 64` 把订单写入不同库表，再由中间件统一对外。Apache ShardingSphere 把自己定位为在已有数据库之上提供 sharding、弹性扩缩、加密等能力的生态，并提供 JDBC 与 Proxy 两类接入形态，价值在于复用既有数据库。

这一方案有效但代价沉重。传统分库分表中间件解决的本质是三件事：一是单库容量/吞吐瓶颈——把一张大表按分片键水平拆开，分散到多个库多个实例；二是透明访问——通过 SQL 改写与路由，让应用尽量像访问单库一样访问分片集群；三是部分跨分片能力——ShardingSphere 用 binding table（绑定表）让分片规则相同的关联表在同一分片内 JOIN，用 broadcast table（广播表）把字典类小表复制到所有分片以便本地关联。但这些能力有明确边界：ShardingSphere 官方文档把「跨库事务」「跨分片的分页、排序、聚合」「开发者必须理解分片拓扑与路由规则」列为其固有约束，且只能在 LOCAL/XA/BASE 三种分布式事务模型间取舍，而 XA 因高并发下性能不足「未被互联网大厂大规模采用」，多数业务转向最终一致的柔性事务。

换言之，中间件能优化「分片规则相同、访问局部」的情形，却无法在内核层提供强一致跨分片事务、全局唯一约束、自由跨分片 JOIN——这些要么被禁止，要么被推回应用层用发号器、Saga、对账脚本兜底。代价可概括为：分库分表把「水平扩展」这一能力，以「牺牲单库的强一致事务、全局索引、自由 JOIN、透明运维」为代价交给了应用层；分片键一旦选定就极难变更，扩容（re-shard）往往意味着停机割接与数据搬迁。这正是 TiDB 与 OceanBase 试图从内核层根除的痛点。

本章要回答的核心问题是：TiDB 与 OceanBase 这两类分布式数据库，各自用什么内核机制把「分库分表」从应用层下沉到数据库内核？分片键/分区键/主键/索引的设计是否就此不再重要？以及——把复杂度下沉到内核，究竟消除了什么、没有消除什么？

本章的立场先放在前面，并贯穿始终：分布式数据库把「分片管理」的工程复杂度从应用层搬进了内核，但分布式系统的物理代价（跨节点 RPC、跨分片事务的多数派往返、热点、全局唯一性的跨分片维护）并不会因此消失，只是换了承担者。从迁移视角看（推测），TiDB 更像「替代分库分表中间件」的通用逻辑表方案，应用看到一张表，内核把数据编码为 KV 并按 Region 自动拆分、复制、调度；OceanBase 更像「把分区表、事务、本地性、多表共置内核化」的关系型分布式数据库，应用仍要显式设计 Partition 与 Table Group，但跨表共置与 LS 事务边界进入了内核。这与第 1 章（分片即 Region/Tablet）、第 10 章（分布式事务）、第 15 章（全局/本地索引）的结论一脉相承，本章聚焦「建模与替代」视角，跨章内容只标注「详见第 X 章」。 〔文献[10]〕

## 9.2 TiDB 的实现：逻辑表 + KV Region 自动分片

**表 9-1　AUTO_RANDOM / SHARD_ROW_ID_BITS / PRE_SPLIT_REGIONS 选择决策**

| 机制 | 解决的问题 | 适用场景 | 关键约束 / 默认 |
|---|---|---|---|
| AUTO_RANDOM | 自增整数主键单调递增、新行 handle 全落在 keyspace 末尾同一 Region 形成写热点 | 用聚簇 BIGINT 主键、且不要求按 ID 单调递增的写密集表，把主键高位填随机 shard bit 打散 | 仅用于聚簇主键且必须 BIGINT；与 AUTO_INCREMENT、DEFAULT 互斥；shard bit 默认 5、最大 15 |
| SHARD_ROW_ID_BITS | 无聚簇主键 / 无主键表以隐式 _tidb_rowid 作 handle 时 rowid 单调集中 | 没有聚簇主键或无主键的表，把 rowid 散列进多个区间（=4 → 16 区间，=6 → 64 区间） | 与聚簇索引互斥；上限 15 |
| PRE_SPLIT_REGIONS | 即使打散 key，建表瞬间仍只有 1 个 Region，冷启动写入先集中该 Region 成热点 | 建表即需铺开多 Region 的冷启动场景，按 2^(PRE_SPLIT_REGIONS) 预切表数据 Region 并 scatter 打散 | 表须带 AUTO_RANDOM 或 SHARD_ROW_ID_BITS；tidb_scatter_region 默认 off、tidb_wait_split_region_finish 默认 true |

TiDB 替代分库分表的本质是：应用看到的永远是「一张逻辑表」，物理上由 TiKV 把这张表的 keyspace 自动横切成多个 Region。中间没有「子表」概念，应用不感知分片数量，迁移时不再从「库 0、库 1、表 0001」起步，而是从 SQL 层的逻辑表起步。

**数据路径：逻辑表如何落到物理分片。** TiDB 的 SQL 层把一张表的所有行编码进一段连续的 keyspace。源码 `pkg/tablecodec/tablecodec.go`（pingcap/tidb @ `release-8.5`，commit `67b4876bd57b`，line 49–82）定义了行 key 前缀常量：`tablePrefix = {'t'}`、`recordPrefixSep = "_r"`、`indexPrefixSep = "_i"`；一行记录的 key 形如 `t{tableID}_r{handle}`，索引项形如 `t{tableID}_i{indexID}{indexValue}`，其中 `prefixLen = 1 + 8 + 2`，`RecordRowKeyLen = prefixLen + 8`（8 字节 handle）。这意味着一张逻辑表的全部行天然构成一段有序、连续的字节区间，TiKV 只需按「大小/key 数/负载」在这段区间上自动切 Region，就完成了「自动分片」，应用层无须任何分片逻辑（table/index/handle 如何编码为有序 KV 详见第 5 章）。应用执行 `INSERT INTO orders ...` 时，TiDB Server 解析 SQL、选择计划、生成表行与索引的 KV key：若表用 clustered primary key，则主键值进入 row handle；若用 `AUTO_RANDOM`，则在主键高位填入随机 shard bit，使连续写入不再总落到同一段有序 key 尾部。

**自动分片的触发点。** Region 的拆分阈值在 TiKV 侧。源码 `components/raftstore/src/coprocessor/config.rs`（tikv/tikv @ `release-8.5`，commit `1f8a140b6d46`，line 75）给出 `SPLIT_SIZE = ReadableSize::mb(256)`，注释明确标注：8.3.0 之前默认 96MiB，>=8.3.0 提升到 256MiB，故 release-8.5 默认即 256MiB。`region_max_size() = region_split_size()/2*3`（line 120–123，即 split_size 的 1.5 倍，约 384MiB），超过即由 split-checker 触发拆分；一次批量拆分上限 `BATCH_SPLIT_LIMIT = 10`（line 79）。拆分由 size/keys/load 在运行时自动涌现，split checker 计算出候选 split key 后把 split 请求交给 Region split 路径，拆完上报 PD，由 PD 调度副本与 Leader 均衡（分片即 Region 的统一边界详见第 1 章，PD 调度详见第 13 章）。

关于 96→256MiB 的版本起点，官方文档与 PingCAP 博客统一表述为「自 v8.4.0 起默认从 96MiB 改为 256MiB」，而 TiKV `release-8.5` 源码 `config.rs` 注释写的是「>=8.3.0 即 256MiB」；二者对「哪个版本开始」表述不一，但对「锁定基准 8.5.x 的默认值 = 256MiB」完全一致。本章按锁定版本写 8.5.x 默认 256MiB，对照 7.5.x 仍为 96MiB，版本起点差异如实记录但不据此立论。（存在版本冲突）

**控制路径：写热点的打散。** 自动分片解决了「容量」，但未解决「写热点」。当主键是自增整数时，新行的 handle 单调递增，所有 INSERT 都落在 keyspace 末尾的同一个 Region，直到它拆分、再在新 Region 末尾继续追加，官方文档明确描述了这一现象。TiDB 提供两件内核工具，替代「分库分表里手工设计散列分片键」：

- `AUTO_RANDOM`：把 BIGINT 主键的高位填入随机 shard bit，使主键值分散而非单调。源码 `pkg/meta/autoid/autoid.go`（line 73–88）确认常量 `AutoRandomShardBitsDefault = 5`（默认 5 个 shard bit）、`AutoRandomShardBitsMax = 15`、`RowIDBitLength = 64`；官方文档给出 `AUTO_RANDOM(S, R)` 语义，`S` 是 shard bits、默认 5、范围 1–15，64 位结构为符号位 1 + 保留位 + shard bits + 自增位，最佳实践是把 shard bit 设为 `log2(TiKV 节点数)`。关键约束：`AUTO_RANDOM` 只能用于 clustered（聚簇）主键、且必须是 BIGINT，并与 `AUTO_INCREMENT`、`DEFAULT` 互斥（建表时由 `pkg/ddl/create_table.go` 校验，且元数据 `AutoRandomBits`/`AutoRandomRangeBits`/`PreSplitRegions` 记录在 `TableInfo`）。
- `SHARD_ROW_ID_BITS`：用于没有聚簇主键或无主键的表（此时 TiKV 以隐式 `_tidb_rowid` 作 handle）。`SHARD_ROW_ID_BITS = 4` 把 rowid 散列进 16 个区间，`= 6` 则 64 个区间；上限 `MaxShardRowIDBits = 15`（源码 `pkg/sessionctx/variable/tidb_vars.go` line 1316，建表时 `pkg/ddl/create_table.go` line 946–949 对超限值截断）。它与聚簇索引互斥。

**正常路径：建表即铺开多 Region（pre-split / scatter）。** 即使打散了 key，建表瞬间仍只有 1 个 Region，冷启动写入仍会先集中在该 Region 形成热点。TiDB 支持在表带 `AUTO_RANDOM` 或 `SHARD_ROW_ID_BITS` 时指定 `PRE_SPLIT_REGIONS=N`，建表后按 `2^(PRE_SPLIT_REGIONS)` 预切表数据 Region。源码 `pkg/ddl/split_region.go`（line 82–104）的 `preSplitPhysicalTableByShardRowID` 按 `step = 1 << (ShardBits - PreSplitRegions)` 切分，注释给出 `sharding_bits=4 / PreSplitRegions=2 → 4 个 Region` 的算例；scatter 作用域有 `ScatterTable/ScatterGlobal/off`（`getScatterConfig`, line 71），由系统变量 `tidb_scatter_region`（默认 off）与 `tidb_wait_split_region_finish`（默认 true）控制。这就是「中间件需要 DBA 手工预建子表」在 TiDB 里被内核化的对应物：TiDB 把「建多少物理表、写哪张表」的工作，转成「如何让 keyspace 产生足够分散的 Region，并让 PD/TiKV 调度这些 Region」。

**故障路径。** Region leader 变化、RegionCache 过期、split/merge 正在发生时，TiDB Server 通过 Region epoch 与重试机制兜底，应用看到的是延迟抖动，而不是「分库分表中某个物理表不可达」的直接错误。这对应第 2 章的路由结论：RegionCache 是性能缓存，正确性由 TiKV 的 Region leader/epoch/MVCC 校验兜底。 〔文献[1-6,11-12]〕

## 9.3 OceanBase 的实现：分区表 + Tablet + Log Stream + Table Group

与 TiDB 的「隐式自动分片」相对，OceanBase 走的是另一条路线：它把「分区表（partitioned table）」这一关系数据库的经典能力做成了分布式内核的一等公民，并在其上叠加 Tablet、Log Stream、Table Group 三层。应用仍显式写标准 `PARTITION BY` DDL，但 Partition 不再只是单机表空间管理手段，而是与 Tablet、LS、Paxos 复制、事务参与、迁移与调度全面关联。

**数据路径：Partition → Tablet → Log Stream。** OceanBase 官方架构文档给出三层定义：一张表按分区规则水平拆成多个 Partition（逻辑分片，原文 "Each shard is called a table partition, or Partition"）；每个物理分区对应一个 Tablet（物理存储对象，原文 "Each physical partition has a storage layer object used to store data, called Tablet … used to store ordered data records"）；Log Stream（LS）承载 Tablet 的 redo 日志并经 Multi-Paxos 复制，原文 "One log stream corresponds to multiple Tablets on the node where it is located" 且 "Tablets can be migrated between Log Streams to achieve load balancing of resources"。源码独立佐证（经直接拉取 v4.2.5_CE 标签 raw 文件确认）：Tablet 默认大小常量 `OB_DEFAULT_TABLET_SIZE = (1<<27)`（= 134217728 字节 = 128MB），见 `deps/oblib/src/lib/ob_define.h`（oceanbase/oceanbase @ `v4.2.5_CE`，commit `e7c676806fda`；4.4.x 亦同值）；`class ObLS` 持有 `ObLSTabletService ls_tablet_svr_`（`src/storage/ls/ob_ls.h`），即 Tablet 归属于 LS、由 LS 统一管理日志与事务。（LS 与 Tablet 一对多、可重绑定、取代早期「每分区一 Paxos 组」模型详见第 1 章；PALF（Paxos-backed Append-only Log File system）共识详见第 3 章。）

应用写入 `orders` 时，SQL 先根据 Partition key 找到目标 Partition，再由 schema 与 Tablet-to-LS 映射定位 Tablet 所在 LS，写入对应 OBServer leader，redo 进入该 LS 的 PALF 复制路径。这里的关键不是「应用知道订单在哪个库」，而是「数据库知道订单属于哪个 Partition、该 Partition 的 Tablet 在哪个 LS、相关表能否共置」。源码侧可见 `ObSimpleTableSchemaV2::get_tablet_ids` 按 partition level 收集 Tablet ID，`get_part_id_by_tablet` 与 `get_tablet_id_by_object_id` 在 Partition/Object 与 Tablet 间查询映射（`src/share/schema/ob_table_schema.h`）。新表 Tablet 的 LS 归属由 `ObNewTableTabletAllocator::prepare`（`src/rootserver/ob_balance_group_ls_stat_operator.cpp`）选择，其策略按表类型分派：带 tablegroup 的表走 `alloc_ls_for_in_tablegroup_tablet`——该函数读取 tablegroup schema、用 `get_table_schemas_in_tablegroup` 取同组成员后调 `alloc_tablet_for_tablegroup` 把分区共置到既有组员的 LS；普通表走 `alloc_ls_for_normal_table_tablet`；本地索引走 `alloc_ls_for_local_index_tablet`（绑定数据表所在 LS）；sharding=NONE 组内的全局索引也走 `alloc_ls_for_in_tablegroup_tablet` 与数据表共置。即「tablegroup schema 进入 Tablet/LS 创建路径并据此共置」由源码确认。

**控制路径：分区键的内核枚举。** 分区键设计在 OceanBase 里直接体现为内核枚举。源码 `src/share/schema/ob_schema_struct.h` 给出 `ObPartitionLevel`（ZERO=非分区 / ONE=一级 / TWO=二级）与 `ObPartitionFuncType`（枚举成员含 `HASH/KEY/KEY_IMPLICIT/RANGE/RANGE_COLUMNS/LIST/LIST_COLUMNS`）。官方架构文档正文仅举 hash/range/list 三类为例（原文 "Partition support includes hash, range, list and other types"），`KEY`/`KEY_IMPLICIT` 等子类型以源码枚举为准（文档措辞为 "and other types"）。官方分区文档建议 HASH/KEY 用于把行均匀映射到 Partition、适合数据均衡分布。这对应业务建模：订单表可按 `user_id` 做 HASH 一级分区、再按月份做 RANGE 二级分区（官方分区建议文档举此例）。注意：二级分区中一级分区是纯逻辑，只有二级子分区才是物理 Tablet。

**正常路径：Table Group 多表共置——中间件做不到的能力。** 这是 OceanBase 区别于「分库分表中间件」的核心，也是它替代传统分库分表时最重要的关系型增强之一。`CREATE TABLEGROUP` 把多张表声明为一组，并通过 `SHARDING` 属性约束它们的物理共置。源码 `deps/oblib/src/lib/ob_define.h` 定义三个字符串常量 `OB_PARTITION_SHARDING_NONE = "NONE"`、`OB_PARTITION_SHARDING_PARTITION = "PARTITION"`、`OB_PARTITION_SHARDING_ADAPTIVE = "ADAPTIVE"`（经直接拉取 v4.2.5_CE raw 文件确认三常量存在）；`src/sql/resolver/ddl/ob_tablegroup_resolver.h` 校验 sharding 取值必须为三者之一，否则 `OB_INVALID_ARGUMENT`；`ObTablegroupSchema` 继承 `ObPartitionSchema` 并保存 `tablegroup_id_`/`tablegroup_name_`/`sharding_`，调度耦合在 `src/rootserver/ob_balance_group_ls_stat_operator.cpp`：按 sharding=NONE/PARTITION/ADAPTIVE 决定组内分区是否共置到同一 LS（NONE→整组同 LS；PARTITION→同序分区共置；ADAPTIVE→自适应）。官方文档给出语义：不指定 SHARDING 时默认 ADAPTIVE；`SHARDING='PARTITION'` 要求成员表分区类型、分区数、分区键值完全一致（例如 HASH 表引用列数与分区数一致，RANGE 表列数、分区数与值域定义一致）；「同一 table group 的相关表组成 partition group，每个 partition group 落在同一台服务器上」，从而「避免分布式事务」、支持 partition-wise join。这正是分库分表中间件无法内核化的能力：中间件即便把 order 与 order_detail 按相同规则拆分，跨表 JOIN 与跨表事务仍是应用层/XA 的责任；OceanBase 用 Table Group 把「同 user 的订单与明细落在同一 LS」做成内核约束，使 JOIN 与事务退化为单 LS 本地操作。

**全局唯一性的内核接管。** `src/share/schema/ob_schema_struct.h` 的 `ObIndexType` 区分 `INDEX_TYPE_NORMAL_LOCAL`/`UNIQUE_LOCAL`/`NORMAL_GLOBAL`/`UNIQUE_GLOBAL` 等枚举成员；`is_global_index_table()`（`ob_table_schema.h`）判定全局索引。本地索引与主表分区一一共置（写本地原子），全局索引独立分区可让唯一约束跨分区生效——但全局索引写入需跨 Tablet/LS，代价高于本地索引（对称取舍详见第 15 章）。分区表的 PRIMARY KEY / UNIQUE 约束必须包含分区键，才能用本地唯一索引保证全局唯一——这是 LOCAL 唯一索引与主表分区一一共置的直接推论（由 `UNIQUE_LOCAL`/`UNIQUE_GLOBAL` 枚举区分体现）。

**故障路径。** LS leader 变化、Tablet transfer、Table Group 约束变化会影响路由与事务路径。源码中 `ObTransferPartitionCommand` 能针对 table/object 与目标 LS 插入 transfer partition task，说明 Partition/Tablet 到 LS 的关系不是静态物理表名，而是内核可管理的映射。对测试开发而言，这比传统分库分表更容易统一注入，但也更需要理解 LS 级提交与恢复。 〔文献[7-9,13]〕

## 9.4 核心差异对比

![[f9_1.svg]]

**图 9-1　多表物理落点对照**

**表 9-2　分库分表替代:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB | OceanBase | 影响 |
|---|---|---|---|
| 替代范式 | 替代分库分表中间件：逻辑表无子表，keyspace 自动横切 Region | 把分区表 + 事务 + 本地性 + 多表共置内核化 | TiDB 迁移更像去掉 sharding 中间件；OceanBase 迁移更像把分区模型内核化 |
| 应用看到的对象 | 逻辑表，底层编码为 KV key range 与 Region | 逻辑表加显式 Partition/Table Group，底层是 Tablet 与 LS | TiDB 对应用更「透明无感」；OceanBase 保留显式分区设计 |
| 物理分片单元 | Region（默认 256MiB，size/keys/load 自动 split） | Tablet（默认 128MB，可自动分裂/迁移） | 粒度相近；Region 与 SQL 表非严格一一对应，Tablet 与物理分区一一对应 |
| 分片是否需声明 | 不需要：建表即一表一 Region，后续自动涌现 | 需要声明 `PARTITION BY`，但无须建子表 | TiDB 默认零分区设计；OceanBase 要求显式分区键设计 |
| 写热点打散 | `AUTO_RANDOM`（shard bit 默认 5/最大 15，仅聚簇 BIGINT PK）、`SHARD_ROW_ID_BITS`（仅非聚簇/无 PK，最大 15）、`PRE_SPLIT_REGIONS` | 选 HASH/KEY 分区键打散；RANGE 分区配合 time 列做冷热；Table Group 共置 + LS 均衡 | TiDB 用「随机化主键高位」；OceanBase 用「分区函数」——前者隐式，后者显式 |
| 冷启动铺开 | `PRE_SPLIT_REGIONS=N → 2^N Region`，scatter 打散 | 建分区表即铺开 N 个 Tablet 到多 LS | 都能建表即多分片，避免冷启动单点 |
| 多表共置 | 无内核级共置约束；靠 PD 自然均衡，跨表事务仍走 2PC | Table Group(NONE/PARTITION/ADAPTIVE) 把相关表分区共置同一 LS | OceanBase 可把跨表事务/JOIN 本地化，TiDB 不保证共置 |
| 跨参与者事务 | 跨 Region 走 Percolator-like 2PC，单 Region 1PC 短路（第 10 章） | 树形 2PC，单 LS 本地化；Table Group 进一步消除分布式事务（第 10 章） | 都把「不跨边界」短路，OceanBase 多一层共置手段 |
| 全局唯一/全局索引 | 全局索引把 partition id 编入 KV 索引项 | `*_GLOBAL` 索引类型；PK/UK 须含分区键或建全局唯一索引 | 都内核接管全局唯一，写入跨分片代价对称落在写侧 |
| 扩容迁移 | PD/TiKV 调度 Region，应用通常不感知物理迁移 | Tablet/LS 迁移，Table Group 与 Partition 规则影响数据布置 | TiDB 运维抽象更自动；OceanBase 运维可控性更强但建模复杂 |

**业务建模示例：订单 + 订单明细 + 支付流水。** 以一个典型电商场景说明两套建模思路的差异。业务有三张表：`orders`（订单）、`order_detail`（订单明细，关联 order_id）、`payment`（支付流水）。这是写密集、按 user_id 维度查询、订单与明细常需 JOIN 的工作负载。下面的两套 DDL 不是唯一方案，而是迁移传统订单库表时的保守基线。

TiDB 建模（让内核自动分片，只需打散写热点）：

```sql
-- 订单表:AUTO_RANDOM 主键打散写入热点,聚簇主键;PRE_SPLIT_REGIONS 建表即铺开
CREATE TABLE orders (
 order_id BIGINT PRIMARY KEY AUTO_RANDOM(5), -- 默认 5 shard bit,32 路打散
 user_id BIGINT NOT NULL,
 amount DECIMAL(18,2),
 created_at DATETIME,
 KEY idx_user_time (user_id, created_at) -- 按 user_id/时间查走二级索引
) PRE_SPLIT_REGIONS=4; -- 建表即铺 16 个 Region

-- 明细表:与订单用 order_id 聚簇主键,保持单订单明细局部集中
CREATE TABLE order_detail (
 order_id BIGINT NOT NULL,
 item_no INT NOT NULL,
 sku_id BIGINT,
 amount DECIMAL(18,2),
 PRIMARY KEY (order_id, item_no) CLUSTERED,
 KEY idx_sku (sku_id)
);

-- 支付流水:AUTO_RANDOM 内部主键,与外部订单号语义分离
CREATE TABLE payment (
 pay_id BIGINT PRIMARY KEY AUTO_RANDOM(5),
 order_id BIGINT NOT NULL, amount DECIMAL(18,2),
 paid_at DATETIME,
 KEY idx_order (order_id),
 KEY idx_paid_at (paid_at)
) PRE_SPLIT_REGIONS=4;
```

TiDB 侧应用看到三张普通表，不再计算 `orders_0237`、不再维护路由表；`AUTO_RANDOM(5)` + `PRE_SPLIT_REGIONS=4` 缓解冷启动与自增主键写热点。但主键随机化不是免费的：按时间顺序扫描必须走二级索引，`idx_user_time`/`idx_paid_at` 本身可能成为单调写入热点；订单与明细用 `order_id` 聚簇可让单订单明细局部集中，但 TiDB 不保证 `orders` 与 `order_detail` 的相关行落在同一 Region/节点，故 `orders JOIN order_detail` 与跨表事务可能跨 Region（走 2PC，单 Region 时才 1PC 短路）。若业务把订单号当作时间顺序或外部可读序列，最好拆成「内部随机主键」与「外部订单号」两个概念。

OceanBase 建模（显式分区 + Table Group 共置以本地化事务）：

```sql
-- 用 Table Group 把同 Partition 的相关表共置到同一 Log Stream
CREATE TABLEGROUP tg_order SHARDING = 'PARTITION';

-- 三表都按 (tenant_id, order_id) 做 KEY 分区,分区数一致
CREATE TABLE orders (
 tenant_id BIGINT NOT NULL, order_id BIGINT NOT NULL,
 user_id BIGINT NOT NULL, created_at DATETIME NOT NULL, status TINYINT,
 PRIMARY KEY (tenant_id, order_id), -- 分区键须入主键以保证全局唯一
 KEY idx_user_time (tenant_id, user_id, created_at)
) TABLEGROUP = tg_order
PARTITION BY KEY(tenant_id, order_id) PARTITIONS 64;

CREATE TABLE order_detail (
 tenant_id BIGINT NOT NULL, order_id BIGINT NOT NULL,
 item_no INT NOT NULL, sku_id BIGINT, amount DECIMAL(18,2),
 PRIMARY KEY (tenant_id, order_id, item_no)
) TABLEGROUP = tg_order
PARTITION BY KEY(tenant_id, order_id) PARTITIONS 64;

CREATE TABLE payment (
 tenant_id BIGINT NOT NULL, order_id BIGINT NOT NULL, pay_id BIGINT NOT NULL,
 pay_channel VARCHAR(32), paid_at DATETIME,
 PRIMARY KEY (tenant_id, order_id, pay_id),
 KEY idx_paid_at (tenant_id, paid_at)
) TABLEGROUP = tg_order
PARTITION BY KEY(tenant_id, order_id) PARTITIONS 64;
```

OceanBase 侧三表按 `(tenant_id, order_id)` KEY 分区，`SHARDING='PARTITION'` 的 Table Group 要求成员表分区类型/数/键一致，从而把「同一订单的 orders/order_detail/payment 分区」共置到同一 LS——同订单的下单、明细、支付落在一处，跨表 JOIN 退化为 partition-wise join、跨表事务退化为单 LS 本地事务，无须分布式 2PC。这套模型保留了类似传统 shard key 的显式设计：`(tenant_id, order_id)` 既是 Partition key，也进入主键与常用索引前缀。代价是分区键必须入每张表的主键（全局唯一约束的要求），且若有「按 user_id 查全量订单」的需求，而 Partition key 不含 `user_id`，查询要么依赖二级索引、要么接受跨 Partition 探测，必要时在 `user_id` 上额外建全局索引（写入跨分片，详见第 15 章）；若支付流水按 `paid_at` 时间范围清理，仅按订单维度分区会让冷热分层更难。

这个例子清晰体现两条路线的分工：TiDB 把「分片」对应用隐去（应用零分片设计），但跨表局部性不可声明、由调度自然决定；OceanBase 保留「分区」的显式设计，换来了 Table Group 这种中间件无法提供的声明式共置能力。两者都要求回到真实 workload 设计键与索引，而不是机械地认为「换了数据库就不必关心 shard key」。

## 9.5 正常路径图（逻辑表/分区表写入）

图 9-2 把两条写入路径并排陈列，刻意按「应用看到的逻辑表 → 内核如何定位物理分片」的同一纵深展开，以便对照两者在哪一层接管了分片职责。

![[f9_2.svg]]

**图 9-2　分库分表替代正常读写/调度路径**

正常写入路径上，两者的关键差异是：TiDB 在 SQL 层把行压平成连续 KV 后由 TiKV 自动切分，应用不感知分片数；OceanBase 在代理/SQL 层用分区键做裁剪定位 Tablet，Table Group 决定相关表是否共置同一 LS（ODP 路由详见第 2 章）。两者都隐藏了应用层路由表，但都没有隐藏网络复制、leader 所在节点、范围扫描与索引维护——TiDB 的控制路径主要是 PD/Region 调度与 DDL 元数据，OceanBase 的控制路径更内聚在 schema、RootService/DDL、Tablet/LS 映射与 Table Group 约束中。

## 9.6 故障/异常路径图

图 9-3 展示两类系统在「替代分库分表」场景下最典型的异常路径——热点未消除、Leader 切换、跨分片事务、以及全局索引/共置失效。

![[f9_3.svg]]

**图 9-3　分库分表替代故障/异常路径**

故障路径的要点：热点、切主抖动、跨分片事务、全局索引写放大，这四类问题在分库分表时代是应用/中间件的责任，在分布式数据库里被内核接管，但并未消失——它们变成了内核的调度、共识、事务恢复负担。传统分库分表的异常常表现为某个物理库表不可达、路由规则不一致、扩容双写校验失败；分布式数据库的异常更常表现为缓存失效重试、leader 迁移、split/merge 或 Tablet transfer 带来的尾延迟。相应地（推测），运维视角需要监控的指标也随之从「中间件配置」变成「Region/Tablet 热点图 + 切主延迟 + 2PC 残锁 + 远程执行比例」；测试时不能只看 SQL 是否成功，还要看重试次数、远程执行比例、热点是否集中、事务参与者数量是否突然增多。这两类观测分属互不相通的命名空间：TiKV 侧是 Prometheus 指标，源码 `components/raftstore/src/coprocessor/metrics.rs`、`components/raftstore/src/store/worker/metrics.rs` 注册了 `tikv_raftstore_region_size`/`tikv_raftstore_region_count`/`tikv_raftstore_region_keys` 与 `tikv_raftstore_check_split_total`/`tikv_raftstore_check_split_duration_seconds` 等（tikv/tikv @ `release-8.5`）；OceanBase 侧是 SQL 字典视图，`src/share/inner_table/ob_inner_table_schema_def.py` 声明了 `DBA_OB_TABLET_TO_LS`/`DBA_OB_TABLEGROUPS`/`DBA_OB_TABLE_LOCATIONS`/`GV$OB_TRANSACTION_PARTICIPANTS` 等（oceanbase @ `v4.2.5_CE`）。二者不可互相对应，且具体名集随版本变化，落地前仍应以目标版本的 Grafana/字典视图为准。

## 9.7 性能、可靠性、运维影响

**延迟。** 单分片（单 Region / 单 LS）操作延迟与单机相近；一旦跨分片，延迟下界被推到至少一次多数派往返。TiDB 单 Region 写可走 1PC、async commit prewrite 后即返回；OceanBase 单 LS 走树形 2PC、prepare 后返回（详见第 10 章）。替代分库分表后，「延迟取决于操作是否跨边界」这一物理事实没变，只是边界从「子库」变成了「Region/LS」。

**吞吐与热点。** 自动分片让吞吐随节点近线性扩展，但前提是 key 分布均匀。`AUTO_RANDOM`（默认 5 shard bit → 32 路）与分区键 HASH/KEY 都是「用随机性换热点消除」；代价是牺牲主键/分区键上的范围有序性——`AUTO_RANDOM` 适合只要求唯一、不要求按 ID 单调递增的主键，按主键的 `ORDER BY`/范围扫描需显式排序或改走二级索引。Region 是范围切分，单调 key、单调二级索引与热点用户仍可能集中到少数 Region；`AUTO_RANDOM` 与 HASH/KEY Partition 都是缓解手段，不是业务倾斜的数学消除器。

**可用性与扩展性。** 两者都用 size/load 触发的自动分裂 + 调度均衡替代「手工 re-shard」。TiDB 侧由 PD 在后台自动迁移热点 Region，无需人工 re-shard 或停机割接，应用通常不感知物理迁移；OceanBase 侧用 Tablet 迁移/transfer + Table Group 调度做均衡。可靠性上两者都把副本复制、leader 选举与故障恢复内核化（TiDB 经 Region leader 的 Raft 复制日志并等待 quorum，OceanBase 经多副本 PALF/LS 的日志复制路径），但这不是对业务无感的魔法：leader 迁移、网络分区、磁盘抖动会变成尾延迟、重试与部分请求超时。

**运维复杂度。** 表面上运维大幅简化（无需 DBA 维护分片表、无需中间件分片规则），但复杂度并未消失，而是从「中间件配置 + 应用分片逻辑」转移为「分区键/分片键/索引设计 + 热点观测 + 调度调参」。TiDB 更关注 Region 分布、store 热点、split/merge 与 PD 调度；OceanBase 更关注 Tenant 资源、Partition/Tablet 分布、LS leader、Table Group 共置与 transfer。下面两张表分别对比「瓶颈来源」与「典型业务场景的建模取舍」。

**表 9-3　性能、可靠性、运维影响**

| 瓶颈来源 | 分库分表中间件 | TiDB | OceanBase |
|---|---|---|---|
| 写热点 | 应用层散列分片键设计 | `AUTO_RANDOM`/`SHARD_ROW_ID_BITS` 不当 → 末尾 Region 热点 | 分区键选错(自增列做 RANGE)→ 末尾 Tablet 热点 |
| 跨分片事务 | 应用层 2PC/Saga/XA | 跨 Region 2PC(单 Region 1PC 短路) | 跨 LS 树形 2PC(Table Group 共置可消除) |
| 跨分片 JOIN | 中间件归并或禁止 | 走 Coprocessor/TiFlash MPP(第 12/16 章) | partition-wise join / broadcast join,共置后本地完成 |
| 全局唯一性 | 应用层全局发号器 | `AUTO_RANDOM` 主键全局唯一 + 全局索引 | PK/UK 含分区键 或 `UNIQUE_GLOBAL` 索引 |
| 范围扫描 | 跨子表归并 | `AUTO_RANDOM` 牺牲主键有序，需走二级索引 | RANGE 分区利于范围裁剪；HASH/KEY 分区不利范围扫 |
| 调度抖动 | 无(静态分片) | Region split / PD 迁移抖动 | Tablet transfer / major merge / 切主抖动 |

**表 9-4　性能、可靠性、运维影响**

| 业务场景 | TiDB 建模建议 | OceanBase 建模建议 | 主要风险 |
|---|---|---|---|
| 订单主表高并发写入 | `AUTO_RANDOM` 主键配合 `PRE_SPLIT_REGIONS`,避免自增尾部热点 | 用能均匀散列的 Partition key,如 `KEY(tenant_id, order_id)` | 主键随机削弱时间局部性;Partition key 选错造成热点 |
| 订单明细随订单写入 | `order_id, item_no` 做 clustered PK,保持单订单局部性 | 与订单主表同 Partition key 并放入同一 Table Group | TiDB 不保证多表共置;OceanBase 要求 Partition 定义一致 |
| 支付流水按时间查询/清理 | 建 `paid_at` 索引或时间维度方案，评估单调写热点 | 时间 RANGE 与订单维度共置目标冲突，需权衡 | 热数据与历史归档目标冲突 |
| 用户维度订单列表 | `idx_user_time(user_id, created_at)` 支持点查与时间分页 | Partition key 若不含 `user_id`,需索引或接受跨 Partition 探测 | 大用户热点、索引写放大 |
| 迁移传统 64 库 1024 表 | 先恢复逻辑表，再用 Region/索引/热点观测调优 | 先确定 Tenant、Partition 数、Table Group、Locality，再迁数据 | 「无路由代码」不等于「无建模成本」 |

性能上，TiDB 的迁移心智更简单：大量 SQL 可从 `orders_${suffix}` 恢复为 `orders`，应用层路由、结果归并、分页拼接、跨库 ID 生成都可收敛到数据库能力。OceanBase 的优势是对关系型共置更显式：订单、明细、支付按同一 Partition key 并放入 Table Group，数据库可围绕 Partition/Tablet/LS 组织路由、事务与迁移，代价是 schema 设计更硬——传统分库分表中「后面再改 shard key」已经很难，到 OceanBase 中「后面再改 Partition key/Table Group」同样不是轻操作。

## 9.8 反例与代价

**TiDB 的代价。**

1. `AUTO_RANDOM` 牺牲主键有序性：适合写密集、按主键随机点查的场景；若业务大量「按主键范围扫描/分页/取最新 N 条」，随机主键会让这些查询丧失顺序读优势，必须另建索引或用其他列。
2. `SHARD_ROW_ID_BITS` 与聚簇索引互斥：只对非聚簇/无 PK 表生效；若已用聚簇主键又有热点，只能改用 `AUTO_RANDOM`，二者不可叠加。
3. 无多表共置保证：TiDB 不提供 OceanBase 式 Table Group，两张相关表的 Region 是否落在同节点由 PD 自然均衡决定，不可声明式锁定，因此跨表事务难以稳定保持 1PC 短路。

**OceanBase 的代价。**

1. 分区键设计前置且难改：必须在建表时确定分区键；`PARTITION` 模式 Table Group 要求成员表分区类型/数/键完全一致，后期改分区是重操作。
2. 全局索引/跨分区写入升级为分布式事务：全局唯一索引让唯一约束跨分区生效，但每次写入跨 Tablet/LS，长尾延迟与写放大显著（第 15 章）。
3. major merge / Tablet 迁移抖动：分区表的全局基线合并与负载均衡会带来周期性抖动（第 4 章）。

把代价归纳成四类更具迁移指导意义。第一类，强 shard key 语义没有消失：若系统 90% 查询是 `WHERE user_id = ?`，传统分库分表按 `user_id` 路由很直接；迁到 TiDB 后若主键改成 `AUTO_RANDOM` 而 `user_id` 只作普通二级索引，查询仍可做，但索引维护与大用户热点问题转移到了 TiKV keyspace；迁到 OceanBase 后，按 `order_id` 分区会让用户维度查询更分散，按 `user_id` 分区又会让同订单的明细与支付共置变差——二者都要求回到真实 workload。第二类，跨参与者事务仍然存在：一笔订单同时写主表、明细、支付、库存索引，TiDB 中可能触及多个 Region，OceanBase 中跨多个 LS 时进入 2PC，数据库提供一致性与恢复路径，但延迟、锁等待、冲突重试仍要压测（第 10 章）。第三类，全局索引与范围扫描可能比传统路由更贵：传统分库分表按月拆支付流水、按月归档很容易；TiDB 逻辑表若按随机主键写入，时间范围依赖二级索引，历史冷热要靠索引、TTL、Placement 或分区表组合；OceanBase 若为订单共置选 `(tenant_id, order_id)`，按 `paid_at` 清理就可能跨很多 Partition——迁移方案不能只比写入路径，也要比归档、审计、对账、风控回溯、用户分页与大商户热点。第四类，运维故障从应用代码变成数据库内部状态：分库分表中路由错误可在应用配置里查，TiDB 的异常可能是 Region scatter 未完成、leader 分布倾斜、二级索引 Region 热点，OceanBase 的异常可能是 Table Group 约束不一致、Tablet transfer 中、LS leader 过于集中——更可统一治理，但要求 DBA、测试与后端理解底层对象。

**共同代价（本章核心论断）。** 无论哪条路线，「跨分片」这一物理边界一旦被业务访问模式频繁穿越，2PC 的多数派往返、全局索引的跨分片写、热点 key 的排队就无法被任何内核消除。分布式数据库做的是「把复杂度下沉、把常见情况优化到接近单机」，而非「让分布式代价归零」。对比传统分库分表，真正的取舍是用「内核调度/共识/事务恢复的复杂度」换「应用层不再写分片逻辑」——这对绝大多数业务划算，但对「极端低延迟、强范围扫描、几乎无跨分片」的工作负载，精心设计的单库或静态分片仍可能更优。

## 9.9 测试开发视角的验证点

**功能等价性必须逐项对比。** 传统 `orders_0000..orders_1023` 合并成 TiDB 逻辑表或 OceanBase 分区表后，按订单号点查、按用户分页、按时间范围对账、订单-明细 JOIN、支付回查、幂等插入与唯一约束冲突都要逐项对比。具体的可测场景：

- 建一张 `AUTO_RANDOM` 主键表，批量并发 INSERT，验证写入是否分散到多个 Region（对比自增 PK），并验证 `AUTO_RANDOM` 主键不会破坏外部订单号语义。
- OceanBase 建 HASH/KEY 分区表，验证按分区键的点查是否被裁剪到单 Tablet；建 Table Group 后验证成员表分区定义一致、且同一订单的主表/明细/支付是否按预期落在可共置的 Partition/Tablet 路径、跨表事务是否退化为单 LS 本地事务。
- 全局唯一索引：并发插入相同唯一键到不同分区，验证冲突被正确拒绝。

**可注入的失效模式。** TiDB 侧：Region leader 切换、TiKV 节点下线、Region split/scatter 期间持续写入、单热点用户高频写、单调时间索引写入。OceanBase 侧：LS leader 切换、Tablet transfer、某个 OBServer 下线、Table Group 约束下新增表失败、跨 LS 事务冲突。共同：跨分片事务 prewrite/prepare 后注入 Leader 宕机，验证残留 primary lock 的 ResolveLock（TiDB）/ 2PC 日志重放（OceanBase）恢复（第 10 章）；故意用自增列做分区键/主键制造末尾热点，观测热点是否如预期出现。传统分库分表迁移还要注入双写期间的幂等重放、数据校验差异与历史表合并失败。

**压测指标不要只看平均 TPS。** 至少要看 P95/P99 写入延迟、冲突重试、热点集中度、跨参与者事务比例、索引写入放大、范围扫描耗时、归档任务对在线写入的影响、扩容或迁移期间的尾延迟。测试数据分布要贴近生产：大商户、大用户、秒杀商品、支付渠道集中、时间窗口突刺、历史月份冷数据都要单独造数——只用均匀随机 `order_id` 压测，最容易把迁移风险掩盖掉。

**观测手段（分系统、分命名空间，不跨产品等同）。** 必须强调：TiDB/TiKV 的 Prometheus 指标（小写 snake_case、带组件前缀如 `tikv_*`，经独立 exporter/scrape 管道采集）与 OceanBase 的 SQL 数据字典视图（Oracle 风格大写标识符、经 sys 租户 `SELECT` 访问）是两套完全不同命名空间、不同产品、不可互相对应的东西——本章绝不把某个 Prometheus 指标名等同于某个 OceanBase 字典视图名（此类「指标名 = 视图名」的对应关系属类别错误，不成立）。可直接查证的诊断方向是：

- TiDB（Prometheus 时序指标 + SQL 命令）：可观测维度包括 Region 数量、热点 Region 读写流量、scheduler/split 相关指标——这些指标名在 TiKV 源码中以 register 宏逐字定义，如 `tikv_raftstore_region_size`/`tikv_raftstore_region_count`/`tikv_raftstore_region_keys`（`components/raftstore/src/coprocessor/metrics.rs`）、`tikv_raftstore_check_split_total`/`tikv_raftstore_check_split_duration_seconds`（`components/raftstore/src/store/worker/metrics.rs`，tikv/tikv @ `release-8.5`）；具体可观测面随版本与 Grafana 面板演进，落地仍以目标版本 metrics 列表为准。可直接查证的 SQL 侧手段：系统变量 `tidb_scatter_region`、`tidb_wait_split_region_finish` 与命令 `SPLIT TABLE ... REGIONS n`、`SHOW TABLE ... REGIONS` 可用于验证分片分布与预切 Region（SQL 语句来自官方文档）。
- OceanBase（SQL 数据字典视图 + DDL 视图）：可用 `SHOW CREATE TABLEGROUP`、`SHOW TABLEGROUPS`、执行计划与官方诊断工具观察 Partition pruning、远程执行与 Table Group 配置；Tablet→LS→节点分布、事务参与者、Table Group 与迁移可经字典视图核对，其视图名在源码 `src/share/inner_table/ob_inner_table_schema_def.py` 中以 `table_name` 逐字声明（v4.2.5_CE/4.3.5/4.4.x 均含）：Tablet 与 LS 映射查 `DBA_OB_TABLET_TO_LS`、副本分布查 `DBA_OB_TABLET_REPLICAS`、LS 位置查 `DBA_OB_LS_LOCATIONS`、表/分区位置查 `DBA_OB_TABLE_LOCATIONS`、Table Group 查 `DBA_OB_TABLEGROUPS`/`DBA_OB_TABLEGROUP_TABLES`、迁移任务查 `DBA_OB_TRANSFER_PARTITION_TASKS`、事务参与者查 `GV$OB_TRANSACTION_PARTICIPANTS`。这些是 SQL 命名空间内的对象，与上一条 TiKV 的 Prometheus 指标名分属互斥命名空间、不可互相对应；列定义随版本演进，落地仍以目标版本字典视图为准。注意 `GV$OB_SQL_AUDIT` 是 SQL 执行诊断视图，不等于合规意义上的 Security Audit。

## 9.10 容易误解点

**误解一：「用了 TiDB/OceanBase 就不用再设计分片键/分区键/索引了。」** 错。分片/分区键设计仍然关键，只是工具变了。TiDB 侧体现在 `AUTO_RANDOM` shard bit 数（默认 5、最大 15）、`PRE_SPLIT_REGIONS`、以及把哪列设为聚簇主键；OceanBase 侧体现在 `ObPartitionFuncType` 的选择（HASH/KEY 防热点 vs RANGE 利冷热）与 Table Group sharding 模式（决定是否共置 → 是否本地事务）。键设计错误（如自增列做分区键）会直接制造热点，内核无法代为弥补。订单号若要有业务序列语义，应与内部主键分离。

**误解二：「分布式数据库自动分片 = 跨分片操作也免费了。」** 错。自动分片只消除了「应用层分片管理」，跨分片事务的 2PC 多数派往返、全局索引的跨分片写、跨 Tablet JOIN 的网络代价都还在。OceanBase 的 Table Group 是少有的能在内核层「把相关表共置以避免分布式事务」的机制，但它要求分区规则一致、且事务是否落在单 LS 仍取决于 Tablet/LS 映射、Partition key、实际访问行与运行时状态，跨 LS 仍有 2PC；TiDB 甚至不提供声明式共置。

**误解三：「TiDB 的 Region 和 OceanBase 的 Tablet，与 SQL 表一一对应。」** 不完全对。OceanBase 的物理分区与 Tablet 一一对应；但 TiDB 的 Region 是纯字节区间，默认一表起步一 Region，Region 物理上可跨表、也会随数据增长拆成多个，与 SQL 表非严格一一对应（详见第 1 章）。

## 9.11 本章结论

1. TiDB 替代的是「分库分表中间件」：逻辑表无子表，行经 `tablecodec` 编码为连续 keyspace `t{tableID}_r{handle}`，由 TiKV 按 256MiB（release-8.5 默认）自动 split Region，`AUTO_RANDOM`/`SHARD_ROW_ID_BITS`/`PRE_SPLIT_REGIONS` 打散热点并铺开冷启动，迁移心智接近「去掉分库分表中间件」。
2. OceanBase 把「分区表 + 事务 + 本地性 + 多表共置」内核化：Partition→Tablet（默认 128MB）→Log Stream，分区键由 `ObPartitionFuncType`（HASH/KEY/RANGE/LIST）选择，Table Group（NONE/PARTITION/ADAPTIVE）把相关表分区共置同一 LS 以本地化事务与 JOIN，迁移心智接近「显式建模业务 Partition」。
3. 分片/分区键、主键、索引设计仍然关键，只是从「应用层手工分片」变为「内核参数与 DDL 设计」；订单/明细/支付的建模应围绕访问模式选择，TiDB 重在避免连续 key 与二级索引热点，OceanBase 重在让共同访问的表共享 Partition key 与 Table Group 约束；设计错误（自增列做键）照样制造热点。（推测）
4. Table Group 是 OceanBase 区别于分库分表中间件、也区别于 TiDB 的核心能力：它能声明式地把 order 与 order_detail 共置同一 LS，使跨表事务退化为本地事务——这是中间件做不到、TiDB 不提供声明式版本的能力。
5. 两条路线都把复杂度从应用层下沉到内核，但都未消除分布式系统的物理代价：跨分片事务的多数派往返、全局索引的跨分片写、热点 key 的排队、切主/迁移抖动依旧存在，只是承担者从应用/中间件变成了内核调度与共识层。（推测）
6. 迁移传统分库分表系统时，必须同时验证写入、点查、分页、范围扫描、全局索引、历史归档、扩容迁移与故障切换，而不是只验证 SQL 兼容。（推测）

## 9.12 参考文献

[1] AUTO_RANDOM. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/auto-random/.
 （支撑:§9.2、§9.7、§9.8、§9.10 中 AUTO_RANDOM 的 64 位结构、默认 5 shard bit、范围 1–15、仅聚簇 BIGINT PK 与互斥约束、与 PRE_SPLIT_REGIONS）
[2] SHARD_ROW_ID_BITS. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/shard-row-id-bits/.
 （支撑:§9.2 中非聚簇/无 PK 表用 SHARD_ROW_ID_BITS 打散 _tidb_rowid（4→16、6→64 区间）的描述。）
[3] Split Region. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/sql-statement-split-region/.
 （支撑:§9.2、§9.9 中 PRE_SPLIT_REGIONS=N → 2^(PRE_SPLIT_REGIONS) Region、scatter、SPLIT TABLE ... REGIONS、tidb_wait_）
[4] Clustered Indexes. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/dev/clustered-indexes/.
 （支撑:§9.2、§9.8 中聚簇 vs 非聚簇存储差异、AUTO_RANDOM 仅聚簇、SHARD_ROW_ID_BITS 仅非聚簇的互斥约束。）
[5] Raft Region Size: The Key to TiDB Performance & Recovery. 官方博客[EB/OL]. https://www.pingcap.com/blog/understanding-raft-region-size-tidb-performance-recovery/.
 （支撑:§9.2 中「自 v8.4.0 起 Region 默认从 96MiB 改为 256MiB」的官方博客表述（与 TiKV 源码注释 ">=8.3.0" 的版本起点冲突据此记录；锁定基准 8.5.x 默认 = 256MiB）
[6] TiDB: A Raft-based HTAP Database. 论文，VLDB 2020[EB/OL]. https://www.vldb.org/pvldb/vol13/p3072-huang.pdf.
 （支撑:§9.2、§9.7 中 TiDB 将大表映射为 KV map、切成 Region、每个 Region 由 Raft 复制、PD 管理 Region 与位置的理论与实现背景。）
[7] OceanBase: A 707 Million tpmC Distributed Relational Database System. 论文，VLDB 2022[EB/OL]. https://vldb.org/pvldb/vol15/p3385-xu.pdf.
 （支撑:§9.3、§9.7 中 OceanBase shared-nothing 分区/Paxos 复制/分区为独立对象、事务处理与多副本架构背景；本章不使用其 benchmark 成绩做排名。）
[8] OceanBase Architecture - Developer Guide. 官方文档，WebFetch 逐句确认可达[EB/OL]. https://oceanbase.github.io/oceanbase/architecture/.
 （支撑:§9.3 中 Partition→Tablet→Log Stream 三层模型（原文 "Each shard is called a table partition, or Partition"、"called Tabl）
[9] OceanBase: Architecture Overview and Key Concepts. 官方博客 / Medium[EB/OL]. https://oceanbase.medium.com/oceanbase-architecture-overview-and-key-concepts-858f6b00b47b.
 （支撑:§9.3、§9.4 中「同一 table group 的相关表组成 partition group、落在同一服务器、避免分布式事务」的共置语义与分区表建议示例（订单按 user_id 一级、按月二级）。该页仅描述 Has）
[10] Apache ShardingSphere — Overview / Sharding & Distributed Transaction. 第三方解读 / 开源中间件文档[EB/OL]. https://shardingsphere.apache.org/document/current/en/features/sharding/.
 （支撑:§9.1 中传统分库分表中间件的 JDBC/Proxy 接入形态、解决的问题与固有约束（跨库事务、跨分片聚合、binding/broadcast table、LOCAL/XA/BASE 取舍）。第三方/开源项目文档，非）
[11] pingcap/tidb 仓库（release-8.5 @ commit 67b4876bd57b）. 源码[EB/OL]. https://github.com/pingcap/tidb/tree/release-8.5.
 （支撑:pkg/tablecodec/tablecodec.go（行 key 编码 t{tableID}_r{handle}）、pkg/meta/autoid/autoid.go（AUTO_RANDOM 常量）、）
[12] tikv/tikv 仓库（release-8.5 @ commit 1f8a140b6d46）. 源码[EB/OL]. https://github.com/tikv/tikv/tree/release-8.5.
 （支撑:components/raftstore/src/coprocessor/config.rs（SPLIT_SIZE=256MiB、region_max_size=split_size/2*3、BATCH_S）
[13] oceanbase/oceanbase 仓库（v4.2.5_CE @ commit e7c676806fda；4.4.x 同值）. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE.
 （支撑:deps/oblib/src/lib/ob_define.h（OB_DEFAULT_TABLET_SIZE = (1<<27) = 128MB、OB_PARTITION_SHARDING_NONE/PARTIT）
## 9.13 信息可信度自评

本章源码级事实（行 key 编码前缀、`AUTO_RANDOM`/`SHARD_ROW_ID_BITS` 常量、Region split 阈值、Tablet 默认大小、Table Group 三种 SHARDING 常量、分区函数与索引类型枚举、共置与 transfer 调度逻辑）均来自 commit-pinned 源码，标注且给出真实文件路径与符号，可信度最高；commit 统一锁定到 pingcap/tidb · release-8.5、tikv/tikv · release-8.5、oceanbase/oceanbase · v4.2.5_CE，凡仅确认到模块级者已显式标注。`AUTO_RANDOM`/`SHARD_ROW_ID_BITS`/Split/Clustered Index 的行为描述来自 docs.pingcap.com 官方文档，可信度高。OceanBase 的 Partition→Tablet→LS 三层、分区类型、Table Group 共置语义以 oceanbase.github.io 官方架构文档（WebFetch 逐句确认）+ 源码枚举为主锚点，辅以官方 Medium 架构概览与 VLDB 2022 论文交叉，可信度高；公开文档版本分散在 V4.0.0/V4.2.x 与当前站点更新之间，正文只写 4.x 通用机制并用源码交叉验证，避免把 Cloud 行为写成 CE 事实。业务建模建议、迁移压测重点与热点风险判断属（推测），需在真实 workload 下验证。ShardingSphere 作为「被替代者」背景，引自开源项目官方文档，属第三方解读类（占比远低于上限），已标注其局限。

反查证发现的冲突与处理：其一，Region 默认大小 96→256MiB 的版本起点，docs.pingcap.com 与 PingCAP 博客写 v8.4.0，TiKV release-8.5 源码 `config.rs` 注释写 >=8.3.0，二者对锁定基准 8.5.x 默认 = 256MiB 无分歧，已在 §9.2 与 §9.11 如实记录、按锁定值书写。其二，`OB_DEFAULT_TABLET_SIZE` 曾因 GitHub 登录态代码搜索 0 命中被误判 refuted，本次改用直接 WebFetch `raw.githubusercontent.com` 上 `v4.2.5_CE` 标签的 `deps/oblib/src/lib/ob_define.h` 逐字读到常量名与取值（128MB）及三个 `OB_PARTITION_SHARDING_*` 常量，故维持；因 WebFetch 行号多次调用不一致，本章对该文件只断言路径 + 常量名 + 取值，不固定行号。其三，「Prometheus 指标名 = OceanBase 字典视图名」被判证伪（类别错误），§9.9 已声明两者属互斥命名空间、不可对应；本次实测在源码内逐字核对了两侧的真实符号——TiKV 的 `tikv_raftstore_region_size`/`region_count`/`region_keys`、`tikv_raftstore_check_split_total`/`check_split_duration_seconds` 等取自 `release-8.5` 的 `coprocessor/metrics.rs`、`store/worker/metrics.rs`，OceanBase 的 `DBA_OB_TABLET_TO_LS`/`DBA_OB_TABLEGROUPS`/`DBA_OB_TABLE_LOCATIONS`/`GV$OB_TRANSACTION_PARTICIPANTS` 等取自 `v4.2.5_CE` 的 `ob_inner_table_schema_def.py`，故 §9.9 已升级为带源码出处的确定视图/指标名，但仍强调二者不可互等、列定义随版本演进。本章未对 shared-storage、Bacchus、RootService 单点等高风险项做断言（本章不涉及）。

---


# 第 10 章 分布式事务模型

## 10.1 本章核心问题

分布式数据库与单机数据库最本质的分野，不在于数据被切成了多少 Region / Tablet，而在于：当一条事务的写集落在多个独立复制组上时，系统如何让「要么全部生效、要么全部不生效」这一原子性承诺，跨越机器边界、跨越网络分区、跨越进程崩溃依然成立。本章要回答的核心问题是：

> 一条事务跨多个 Region / 多个 Log Stream-LS 时，TiDB 和 OceanBase 分别如何保证原子性、一致性和可恢复性？数据路径、控制路径和故障路径分别是什么？

事务层不是把单机 ACID 扩大一圈这么简单，它要同时处理时间戳、锁、MVCC 可见性、复制日志、leader 切换、参与者恢复和客户端返回时机。**这是分布式数据库底层的关键，原因有三。**第一，原子提交协议（2PC 及其变种）决定了事务的提交延迟下界——它是 OLTP 尾延迟的主要来源之一。第二，提交协议与底层共识层（Raft / Paxos，详见第 3 章）的耦合方式，决定了「事务提交」与「日志落盘」两件事的语义边界：一次事务写既要满足 ACID 的 A/D，又要满足共识的多数派持久化，这两层如何叠加直接影响可恢复性。第三，提交协议的故障恢复路径（协调者崩溃、参与者 Leader 切换、客户端断连后残留锁）是测试开发最容易制造缺陷、也最难验证的区域。

事务模型必须放在各自的复制边界上理解。TiDB 的数据复制边界是 Region / Raft Group：一条 SQL 可能因为主表行、二级索引、唯一约束检查或多 key 写入而跨 Region。OceanBase 4.x 的事务参与与日志复制边界是 Log Stream-LS，Tablet 是数据承载对象：一条 SQL 可能因为访问多个 Tablet 而落到一个或多个 LS。

本章一个必须谨慎处理的论断是：「TiDB 的所有事务都是潜在的分布式事务」。从实现路径看，TiDB 事务协议天然支持跨 Region，任何涉及主表行 + 二级索引、或多个 key 的写都可能跨 Region，因而走完整的两阶段提交；但单 key 写、单 Region、1PC、async commit、以及 OceanBase 的单 LS 事务，都是对「完整 2PC」的短路优化，必须单独说明，不能笼统断言「潜在分布式」等于「实际都付完整 2PC 代价」。

本章不重复 TSO/GTS 的元数据控制面机制（详见第 13 章），不重复 Raft/PALF 的共识细节（详见第 3 章），也不展开 MVCC 多版本可见性判定（详见第 11 章），只在事务提交协议需要时引用其结论。

---

## 10.2 TiDB 的实现

TiDB 的事务模型源自 Google Percolator（OSDI 2010）。Percolator 在 Bigtable 之上用客户端协调的两阶段提交实现快照隔离（Snapshot Isolation），其核心是把「事务状态」编码进三类列（在 TiKV 中映射为三个 Column Family，详见第 4、5 章）：

- `CF_DEFAULT`（对应 Percolator 的 `data` 列）：存实际值，key 带 `start_ts`；
- `CF_LOCK`（`lock` 列）：存锁信息；
- `CF_WRITE`（`write` 列）：存提交记录，key 带 `commit_ts`，值里记录该版本对应的 `start_ts`。

事务开始时，客户端（TiDB Server）向 PD 取一个 `start_ts`（TSO，详见第 13 章），作为事务快照版本号，读取 `start_ts` 之前已提交的版本。提交分两阶段：

**表 10-1　Percolator 2PC 三列族（default / lock / write CF）逐阶段 KV 布局**

| 阶段 | default CF（data 列） | lock CF（lock 列） | write CF（write 列） |
|---|---|---|---|
| Prewrite | 写暂存值，key 带 start_ts | 放锁:一 key 选为 primary，其余 secondary，secondary lock 内部记录指向 primary lock | —（本阶段无变化） |
| Commit | —（本阶段无变化） | 删除锁:先删 primary lock，再对所有 secondary 重复（可异步并行） | 写入提交记录，key 带 commit_ts，值记该版本对应 start_ts |

Prewrite 阶段。对写集中每个 key，在 `lock` 列放锁、在 `data` 列写暂存值；若该 key 上已有锁、或存在比 `start_ts` 更新的 `write` 记录，则发生写冲突，事务回滚。其中一个 key 被选为 primary，其余为 secondary；每个 secondary lock 内部记录指向 primary lock 的位置，primary lock 是整个事务状态的「真相之源」（source of truth）：primary 已提交则事务可 roll forward，primary 已回滚或超时则 secondary 可回滚。

Commit 阶段。prewrite 全部成功后取第二个时间戳 `commit_ts`，先提交 primary——删除 primary lock、同时在 `write` 列写入 `commit_ts` 的提交记录；随后对所有 secondary 重复该过程（可异步在后台并行完成）。一旦 primary 提交成功，整个事务即被视为已提交，TiDB 即可向客户端返回成功。这条路径与 TiDB 的 VLDB 论文描述一致：选 primary key、按 Region 分组并行锁所有 key、prewrite 成功后取 commit ts、提交 primary，再异步清理 secondary。

源码上，prewrite 与 commit 的实现入口在 TiKV `src/storage/txn/actions/`：`prewrite.rs` 含函数 `prewrite<S: Snapshot>(...)` 与 `prewrite_with_generation<S: Snapshot>(...)`，参数含 `secondary_keys: &Option<Vec<Vec<u8>>>`；提交类型用真实枚举 `enum CommitKind { TwoPc, OnePc(TimeStamp), Async(TimeStamp) }`，事务类型用 `enum TransactionKind { Optimistic(bool), Pessimistic(TimeStamp) }`，后者即乐观/悲观事务的区分。`commit.rs` 的入口 `commit<S: Snapshot>(.., commit_ts: TimeStamp) -> MvccResult<Option<ReleasedLock>>` 负责把 Lock 转成 Write 记录（`txn.put_write`）并释放锁（`txn.unlock_key`）。Lock / Write 结构定义在 `components/txn_types/src/lock.rs` 与 `write.rs`：`Lock` 含 `primary: Vec<u8>`、`min_commit_ts: TimeStamp`、`use_async_commit: bool`、`secondaries: Vec<Vec<u8>>` 等字段（注释明确 `secondaries` 仅在 primary lock 且 `use_async_commit==true` 时有效）；`enum WriteType { Put, Delete, Lock, Rollback }` 即 MVCC Write 记录的四种类型。

**悲观事务与乐观事务。** TiDB v3.0.8 后新集群默认悲观事务。乐观事务直接执行、提交时才在 prewrite 检测冲突，冲突则回滚——适合低冲突场景，但冲突时可能到 commit 才失败、批量回滚代价高。悲观事务改变的是加锁时机，不改变最终 2PC 框架：在 2PC 之前增加一个 Acquire Pessimistic Lock 阶段，DML 或 `SELECT FOR UPDATE` 执行时即向 TiKV 申请悲观锁并持久化，提交时再走 prewrite/commit。官方文档明确悲观事务提交过程与乐观事务相同、仍采用 2PC；额外的加锁阶段需要写入 TiKV，通常要等 Raft commit/apply 后返回，pipelined locking 则是在满足锁条件后先通知 TiDB、异步写锁以降低延迟。高冲突场景下「先上锁」的代价小于「后回滚」的代价，但每条 DML 多一次 RPC，延迟更高。悲观锁的 `for_update_ts` 记录最近一次加锁/读的时间戳，用于读到「提交于事务开始之后」的最新数据（当前读，非可重复读）。client 侧入口在 TiDB `pkg/store/driver/txn/txn_driver.go` 的 `LockKeys` / `StartFairLocking` / `RetryFairLocking`。

**async commit 与 1PC 优化。** 经典 Percolator 2PC 的延迟瓶颈在于：客户端必须等 primary commit 落盘后才能向应用返回成功，这要求至少两次串行的网络往返 + 落盘。async commit（TiDB 5.0 引入）把事务的提交状态提前到所有涉及 key 的 prewrite 全部成功之后：只要 prewrite 全部成功，事务即被视为成功，客户端立即返回，真正的 commit 异步在后台完成。其正确性关键在 `min_commit_ts`：每个 key 的 prewrite 响应携带一个 `min_commit_ts`，取值为 `max(并发管理器的 max_ts, start_ts, for_update_ts)`；客户端选取所有 prewrite 返回中最大的 `min_commit_ts` 作为最终 `commit_ts`。为保证一旦某事务的全部 secondary lock 写好后，该事务状态可被任意 reader 独立恢复，async commit 把「全部 key 列表」写进 primary lock（在 TiKV 即 `Lock.secondaries`），读到锁的事务可以通过检查 primary 和 secondaries 判断最终状态。commit.rs 中有真实校验 `if commit_ts < lock.min_commit_ts { ... }` 报错——这是 async commit 正确性的硬约束，保证选定的 commit_ts 不会小于任何已读快照看到的下界。

1PC（单阶段提交）是更激进、约束更强的优化：当一个事务的所有 key 都能在单个 prewrite 请求（即落在同一个 Region）中完成、且未使用 binlog 等条件满足时，TiKV 可在 prewrite 的同时直接提交、彻底跳过 commit 阶段；`commit_ts` 即计算出的 `min_commit_ts`。源码上对应 prewrite.rs 的 `self.try_one_pc()`。1PC 崩溃后只可能是未 prewrite 或已完整提交，不留下需要恢复的锁。需要强调：1PC 只对单 Region 事务有效，跨 Region 时自动回退到 2PC（或 async commit），不能当成普遍路径。

**事务提交与 Raft 提交的关系。** 事务层的优化交代清楚后，需谨慎区分它与共识层的边界——这是本章最容易被误读的一处。TiKV `src/storage/txn/` 与 `src/storage/mvcc/` 模块的职责是：把 prewrite/commit 命令转换成对三个 CF 的 KV 修改；单机内对同一 key 的并发事务命令，用内存 latch（`src/storage/txn/latch.rs`、`scheduler.rs`）串行化，这与 Raft 层无关。这些 KV 修改最终经由 raftstore 作为 Raft proposal 复制落盘——raftstore 提交链路在 `components/raftstore/`，不在上述两个模块内（本章仅确认到模块级，详见第 3 章）。

因此对「prewrite 与 commit 是否都要落到对应 Region 的 Raft log」这一问题，准确的回答是：prewrite 和 commit 都是对 Region 数据的写操作，各自都需要经过该 Region 的 Raft 复制、达成多数派持久化才算成功。一个跨 N 个 Region 的事务，prewrite 阶段产生 N 组 Raft 写，commit 阶段（primary 所在 Region）再产生写——所以一次跨 Region 事务的提交链路上，叠加了多次独立的 Raft 多数派往返。跨 Region 原子性由 Percolator-like 2PC 的 primary lock 状态判定保证，单 Region 内的持久性与副本一致性由 Raft 保证，二者是正交的两层。源码上这条「存储层 → Raft 复制」的桥接已可逐函数核对：`src/server/raftkv/mod.rs` 的 `RaftKv::async_write(...)` 把上层 `WriteData.modifies`（即 prewrite/commit 产生的 KV 修改）逐条转成 `Request`、组装成 `RaftCmdRequest`，在 one-pc 事务时置 `WriteBatchFlags::ONE_PC` 标志，再调用 `self.router.send_command(cmd, cb, extra_opts)` 投递到 raftstore；raftstore 侧由 `components/raftstore/src/store/peer.rs` 的 `Peer::propose<T>(...)` → `propose_normal<T>(...)` 将其作为 Raft proposal 复制、达成多数派后 apply。换言之，prewrite 与 commit 各自经由同一条 `async_write → send_command → propose_normal` 链路落到对应 Region 的 Raft log。

**协调者边界：client-go 才是 2PC 主循环所在。** 与共识层边界同样易被误解的，是协调者究竟落在哪一层：TiDB Server 并非 2PC 主循环本体。`pkg/store/driver/txn/txn_driver.go` 的 `Commit` 直接委托给内嵌的 `txn.KVTxn.Commit(ctx)`，并把 `EnableAsyncCommit` / `Enable1PC` / `IsolationLevel` / `Pessimistic` 等选项以 `SetOption` 传给 KV 层。真正「选 primary key、并发 prewrite secondaries、commit primary、异步 commit secondaries」的循环，位于外部库 `github.com/tikv/client-go`（其 `txnkv/transaction/2pc.go` / `commit.go` / `prewrite.go`），不在 tidb 仓库内。该依赖在 tidb release-8.5 的 `go.mod` 中精确锁定为 `github.com/tikv/client-go/v2 v2.0.8-0.20260615130046-a2b634d170d9`（伪版本编码 commit `a2b634d170d9`）。

**数据路径与控制路径的拆分。** 从数据路径看，TiDB 写事务可以拆成三层：SQL 层把行、索引、唯一约束检查转换成有序 KV key；client 层根据 Region cache 把 key 分组并发送 RPC；TiKV 在 Region leader 上执行事务命令并经 Raft 复制。只要写集跨多个 Region，TiDB 就要面对多个 Region leader 的尾延迟、部分成功、路由过期、Region split 后 epoch 变化等问题。从控制路径看，PD TSO 和 Region 元数据是两个不同依赖：TSO 决定 `start_ts` / `commit_ts` 的全局顺序，Region 元数据决定请求发往哪个 TiKV leader；TSO 不替某个 Region 提交 prewrite，Region leader 也不自行生成全局提交时间戳。PD 短暂不可用会影响新事务取时间戳、路由刷新和 `commit_ts` 获取，但已落入某个 Region 的 Raft 写入仍由该 Region 的 Raft Group 维护复制安全——因此 PD 是逻辑中心化的时间戳与元数据依赖，而非物理单点（详见第 13 章）。

**隔离级别。** TiDB 文档明确：其 MySQL 兼容的 `REPEATABLE-READ` 实现的是 Snapshot Isolation / SI，不同于 ANSI RR 和 MySQL InnoDB RR；`Read Committed` 只在悲观事务模式下生效，乐观事务即便设置 RC 仍按 RR/SI 运行。SI 以事务开始时的快照为主，RC 以语句级较新快照减少冲突，但仍要配合悲观锁与当前读语义，不能把二者混写（隔离级别底层细节见 §10.11）。

涉及的源码/文档：Percolator 论文（OSDI 2010）；TiKV deep-dive 与 tidb-dev-guide；TiKV 源码 `src/storage/txn/`、`src/storage/mvcc/`、`components/txn_types/src/`（release-8.5 @ `1f8a140b6d46`）；TiDB 源码 `pkg/store/driver/txn/txn_driver.go`（release-8.5 @ `67b4876bd57b`）。

--- 〔文献[1-2,5-8,11-12,16]〕

## 10.3 OceanBase 的实现

**单 Log Stream-LS 事务 vs 跨 LS 事务。** 与 TiDB 把参与者粒度落在 Region/key 不同，OceanBase 4.x 的事务参与者粒度是 Log Stream-LS。官方架构文档写明：LS 是一组 Tablet 与 ordered redo logs 的实体，使用 Paxos 同步副本日志；从事务视角，单个 LS 内的修改可用单分支原子提交（one-phase atomic commit），跨多个 LS 的修改使用 OceanBase 两阶段原子提交协议，LS 即分布式事务参与者。这带来一个关键结构性差异：

- 单 LS 事务：事务全部读写落在同一个 LS 上，无需分布式协调，可走单分支提交——只需把 redo log 通过该 LS 的 Paxos 复制到多数派即提交成功，无 2PC 开销。
- 跨 LS 事务：写集落在多个 LS，进入分布式两阶段提交，参与者是 LS 而非单个 Tablet。

关键推论是：OceanBase 不以单个 Tablet 作为 2PC 参与者，也不能把「跨 Partition」等同于「必然跨 LS」。若多个 Tablet 位于同一 LS，虽然 SQL 层看起来访问了多个数据对象，提交层仍可按一个 LS 参与者处理；只有 Tablet 分散在多个 LS 时，coordinator 才需要组织多个 LS leader 进入 2PC。这解释了 OceanBase 4.x 为何强调 LS 聚合 Tablet：它用更粗的日志与事务参与单位，换取更少的跨参与者协调，代价是 LS 映射、均衡、迁移和恢复更复杂。

事务服务总入口在 `src/storage/tx/`（`ob_trans_service.cpp` / `ob_trans_service_v4.cpp`、`ob_tx_api.cpp`、`ob_trans_ctx_mgr_v4.cpp`），承载单 LS 与跨 LS 的事务上下文管理；`src/storage/tx_storage/` 负责 LS 级存储与 checkpoint（`ob_ls_service.*`、`ob_ls_map.*`、`ob_access_service.*`）。「单 LS 短路提交」的判定已可定位到具体函数：`src/storage/tx/ob_trans_part_ctx.cpp` 的 `ObPartTransCtx::commit(...)` 在提交时检查 `exec_info_.participants_`——当 `participants_.count() == 1 && participants_[0] == ls_id_ && 0 == intermediate_participants_.count() && !is_dup_tx_` 成立时，置 `trans_type_ = TransType::SP_TRANS` 并走 `one_phase_commit_()`（其内部即 `do_local_tx_end_(TxEndAction::COMMIT_TX)`，无 2PC 协调）；否则置 `TransType::DIST_TRANS`、`set_2pc_upstream_(ls_id_)` 进入 `two_phase_commit()`。该判定在 v4.2.5_CE / 4.3.5 / 4.4.x 三个版本逐字一致。

**优化版树形两阶段提交（tree-shaped 2PC）。** OceanBase 跨 LS 事务采用的不是教科书 2PC，而是一种协调者下沉、树形拓扑、为 Leader 切换设计的优化 2PC。其状态机在源码 `src/storage/tx/ob_committer_define.h` 中定义为：

![[f10_1.svg]]
**图 10-1　OceanBase 树形 2PC 协调者拓扑**

```cpp
enum class ObTxState : uint8_t {
 UNKNOWN=0, INIT=10, REDO_COMPLETE=20, PREPARE=30,
 PRE_COMMIT=40, COMMIT=50, ABORT=60, CLEAR=70, MAX=100 };
enum class Ob2PCRole : int8_t { UNKNOWN=-1, ROOT=0, INTERNAL, LEAF };
```

相比教科书 2PC，这里多出 `REDO_COMPLETE`、`PRE_COMMIT`、`CLEAR` 三个工程态；角色 `ROOT/INTERNAL/LEAF` 即树形 2PC 的协调者 / 中间节点 / 叶子参与者。每个阶段对应一类持久化日志 `ObTwoPhaseCommitLogType`（OB_LOG_TX_INIT / COMMIT_INFO / PREPARE / PRE_COMMIT / COMMIT / ABORT / CLEAR）和一类消息 `ObTwoPhaseCommitMsgType`。

提交器抽象 `ObTxCycleTwoPhaseCommitter`（`ob_two_phase_committer.h`）的头注释逐字写道：它是 "the implementation of the Optimized Oceanbase two phase commit that introduces hierarchical cycle structure to commit the txn successfully during the transfer process"——即为应对 Leader transfer 而设计的层级/树形优化 2PC。其 handler 含标准的 `handle_2pc_prepare_*` / `handle_2pc_commit_*` / `handle_2pc_abort_*`，以及优化态 `handle_2pc_pre_commit_*`（注释：用 precommit 降低单机读延迟）、`handle_2pc_clear_*`（降低 SQL commit 延迟）、`handle_2pc_prepare_redo_*`（只持久化 redo + commit info，对应 XA / prepare-redo），并含 orphan 消息 handler 与 leader transfer 钩子。源码侧另可见 `ob_one_phase_committer.*`、`ob_two_phase_upstream_committer.cpp`、`ob_two_phase_downstream_committer.cpp`、`ob_tx_2pc_ctx_impl.cpp`，其中 `ob_tx_2pc_ctx_impl.cpp` 含 `do_prepare` / `do_pre_commit` / `do_commit` / `do_abort` 等阶段，并有 pre-commit 不写 PALF log、leader switch、GTS/max_commit_ts 与 tree-style two phase commit 的注释。

公开文档把角色分为 coordinator 与 participant：coordinator 推进协议，participant 按请求完成 prepare、commit、rollback；prepare 阶段 participant 判断能否提交、持久化 prepare logs 并返回，commit 阶段 coordinator 在收到 prepare ACK 后进入 COMMIT、可向用户返回成功，再向各 participant 发送 commit request，participant 释放资源、解锁行、提交 commit logs 并释放事务上下文。

**协调者下沉（coordinator-less）** 是 OceanBase 2PC 与 Percolator 最深的分歧。OceanBase 不为协调者单独持久化「全局事务状态日志」，而是让分布式事务的第一个参与者充当协调者；故障时通过查询所有参与者的状态来恢复事务。其收益体现在两处：官方表述为提交只需 2 次（而非 3 次）Paxos 同步，且客户端可在 prepare 阶段结束后立即返回，而非等到 commit 写完——这把跨 LS 事务对用户可见的提交延迟压到约 1 次 Paxos 同步。VLDB 2022 OceanBase 论文称其为 Paxos-based 2PC，目标是减少 coordinator 状态持久化与 Paxos 同步次数；这应理解为工程优化，不等于没有 2PC 风险、也不等于没有参与者日志持久化。

在此之上，2026 年 arXiv 论文《A Tree-Structured Two-Phase Commit Framework for OceanBase》进一步把这套设计形式化：以 Log Stream-LS 作为原子参与者替代分区级协调，使「100 个共置分区」的事务只与 1 个 LS 参与者交互；树形拓扑下「每个有非空子节点的节点既是其子节点的协调者、又是其父节点的参与者」；事务响应延迟为 2H 次消息往返 + 1 次日志同步（H 为平均树高），并引入 prepare-unknown / trans-unknown 等状态以避免参与者上下文丢失时错误 abort。由于该论文为 2026 年资料、而本章源码基准锁定在 4.2.5_CE，此处只把它作为论文层与源码注释可相互印证的方向：源码中确有 tree-style two phase commit 与 transfer 相关注释，但具体算法是否全部等同论文、产品版本归属，仍需以官方 release note 进一步核验。需特别澄清两点数字出处（详见 §10.13 反查证）：论文摘要/引言所述「协调开销下降约 99%」是 N=100 个共置分区聚合为 1 个 LS 参与者的解析性算例（参与者数量级缩减），并非 TPC-C 工作负载下实测的端到端开销；「单机事务延迟约 1.2ms」同样出现在摘要/引言作为设计特性，而非评测章节在 TPC-C 配置下报告的实测值。两者均不得作为 benchmark 实测结果引用。

**提交与 Log Stream / Paxos / SCN 的关系。** OceanBase 的事务日志只有 redo log（亦称 clog），无 undo log。事务的 redo 与 2PC 各阶段日志，通过 `ObLogHandler`（logservice → PALF/Paxos）的 `submit_log(...)` 持久化并复制到对应 LS：源码 `src/storage/tx/ob_tx_log_adapter.h` 中 `ObTxPalfParam` 持有 `logservice::ObLogHandler *handler_`，实现类 `ObLSTxLogAdapter` 持 `log_handler_`；`ObTxRedoSubmitter`（`ob_tx_redo_submitter.h`）负责把 redo 提交进日志流。SCN（`share::SCN`）即日志/版本时间戳，贯穿 redo 落盘与 MVCC 版本可见性。提交语义是：Leader 仅当多数派副本都收到 clog 并落盘后才提交该记录（Multi-Paxos）。

需要点明 OceanBase 与 TiKV 在日志—状态机耦合方式上的方向差异（详见第 3 章）：OceanBase 走复制式 WAL，即 Paxos-backed append-only log 的追加 + 事务状态推进；TiKV 走 Raft RSM 模型，Raft log 先达成多数派、apply 才改状态机。换言之，OceanBase 的「事务提交 = redo 多数派持久化」与「存储引擎改动」在时序上的耦合方式，与 TiDB 不同，但这并不意味着可见的 MemTable/存储引擎改动会先于日志持久化对外生效——单 LS 事务的 commit version、redo logs 与事务结束状态都要进入 LS WAL，并依赖 LS 的 Paxos 多副本提交来获得持久性与高可用。

**GTS 全局时间戳。** OceanBase 用租户级 GTS 提供外部一致性：若事务 T1 完成时 T2 才开始提交，则 T1 的 commit 版本必小于 T2。GTS 文档说明每个 Tenant 启用 Global Timestamp Service，事务提交时取得的时间戳作为 commit version 写入对应 LS 的 WAL；GTS provider 来自租户级 log stream 1 的 leader，预分配范围会写日志并通过副本同步防止回退。源码上 GTS 来源在 `src/storage/tx/ob_gts_source.h` 的类 `ObGtsSource`，方法 `int get_gts(const MonotonicTs stc, ObTsCbTask *task, int64_t &gts, MonotonicTs &receive_gts_ts)`；含本地缓存（`gts_cache_leader_` 等）与常量 `GET_GTS_QUEUE_COUNT = 1`，说明走「本地缓存 + 向 GTS leader 取」路径。GTS 缓存的后台刷新周期亦可在源码确认：`src/storage/tx/ob_ts_mgr.h` 定义 `#define REFRESH_GTS_INTERVEL_US (500 * 1000)`（即 500ms），由 `ob_ts_mgr.cpp` 中专职刷新线程 `ObTsMgr::run1()` 以 `ob_usleep(REFRESH_GTS_INTERVEL_US, ...)` 周期性对各租户执行 `ObGtsRefreshFunctor`；此外 `ob_gts_source.h` 中 `refresh_gts_location` 的位置刷新间隔为 `refresh_location_interval_(100 * 1000)`（100ms）。该刷新周期常量在 v4.2.5_CE / 4.3.5 / 4.4.x 三版本一致。自 4.0 起单机（单 LS）事务的提交版本号可不经 RPC 直接获取 GTS，以降低延迟。读版本的取法按隔离级别区分：RC 在语句开始取读版本，RR/Serializable 在事务开始取读版本，以此形成全局一致快照。

GTS 与 TiDB TSO 同为「集中授时 + 本地缓存优化」，但 GTS 按租户隔离（详见第 13 章），而 TSO 是集群级单一服务。OceanBase 的控制路径中，GTS/SCN、location cache、LS leader、PALF 日志复制分别承担不同职责：GTS/SCN 给读写版本排序，location/leader 信息决定请求发往哪个 LS leader，PALF/Paxos 决定某条 LS WAL 是否被多数派持久化。某个环节不可用时表现各异——GTS provider 切换主要影响新版本获取与等待，LS leader 切换影响请求路由与 participant 状态推进，PALF 多数派不可用则影响该 LS 日志提交。把这些都写成「中心服务不可用」会遮蔽真实故障边界。（推测）

--- 〔文献[3-4,9,13,15]〕

## 10.4 核心差异对比

把前两节的实现细节并置，可沿以下维度逐项对照两套事务模型的取舍。

**表 10-2　分布式事务模型:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB | OceanBase | 影响 |
|---|---|---|---|
| 协议谱系 | Percolator-like 客户端协调 2PC，primary lock 判定事务最终状态 | 优化版树形 2PC（协调者下沉），以 LS 参与者的事务日志状态为锚点 | TiDB 事务真相编码进 KV 列；OB 编码进 2PC 状态机日志 |
| 参与者粒度 | Region / key（写 key 按 Region 分组，Region 内 Raft 持久化） | Log Stream-LS（一组 Tablet）；跨 LS 才进入分布式 2PC | OB 把多分区收敛为 LS，参与者数大幅减少 |
| 协调者 | 客户端（client-go）逻辑协调，primary lock 为真相源 | 第一个参与者充当协调者，无独立协调者日志 | OB 故障恢复靠查参与者重建，无协调者日志 SPOF |
| 客户端何时返回 | 普通 2PC：primary commit 成功即返回；async commit：prewrite 全成功即返回 | coordinator 收到 prepare ACK 后即可返回成功 | 两者都把可见延迟压到约 1 次往返，且都有「用户已成功、后台仍在清理/推进」的窗口 |
| 时间戳 | TSO（集群级单一），提供 `start_ts`/`commit_ts` | GTS/SCN（租户级），提供 read version / commit version | 见第 13 章；不能把二者细节互相套用，OB 单 LS 事务可免 GTS RPC |
| 单分区/单 LS 优化 | 1PC（单 Region 跳过 commit 阶段） | 单 LS 事务走单分支提交，免 2PC | 都对「不跨边界」事务短路 |
| 提交与共识关系 | prewrite+commit 各自经 Region Raft 多数派 | redo/prepare/commit 各阶段日志经 LS Paxos 多数派 | 都是「事务层 + 共识层」两层叠加，共识负责单参与者复制持久性 |
| 锁信息存放 | lock 列（KV），primary 内嵌 secondary 列表（async commit） | 事务上下文 + 2PC 日志（redo/clog） + tx data | TiDB 残留锁需 reader 主动清；OB 靠参与者状态恢复 |
| 隔离级别 | RR 名称下实现 SI，RC 仅悲观事务模式生效 | MySQL 模式：RC/RR（默认 RC）；Oracle 模式：RC/RR/Serializable（默认 RC） | 见 §10.11/§10.13，均以 SI 为底层；OB MySQL 模式不含 Serializable |

关键对照：TiDB 把事务的「真相」分布式地编码进每个 key 的 lock/write 列，primary lock 是状态锚点；残留锁由后续 reader 主动触发 lock resolver 清理。OceanBase 把「真相」编码进 2PC 状态机及其 Paxos 日志与 tx data，事务恢复由参与者 LS 的 Leader 在重选/重放时被动驱动。前者是「惰性、读触发」的恢复，后者是「主动、状态机驱动」的恢复——这是两条故障路径的根本分野。

--- 〔文献[10]〕

## 10.5 正常路径图

图 10-2 为 TiDB 一条跨两个 Region 的乐观事务（async commit）正常提交路径。注意 prewrite 与 commit 都要各自经过 Region 的 Raft 多数派。

![[f10_2.svg]]

**图 10-2　分布式事务模型正常读写/调度路径**

图 10-3 为 OceanBase 一条跨两个 Log Stream-LS 的事务正常提交路径（优化版树形 2PC，协调者下沉）。

![[f10_3.svg]]

**图 10-3　分布式事务模型正常读写/调度路径**

两图对照可见同一条结构性规律：客户端可见的「成功」都被提前到「相关日志多数派持久化」这一点（TiDB 是 prewrite 全成功、OceanBase 是 prepare ACK），真正改写 write CF / 写 COMMIT 日志、释放锁的收尾动作都异步在后台进行。

---

## 10.6 故障 / 异常路径图

TiDB 的故障恢复核心是：任何 reader 读到一把「别人的锁」时，主动沿 secondary lock 的指向到 primary 处查事务状态，据此清理全部 secondary 锁。最棘手的窗口不是「全部失败」，而是「部分 prewrite 成功」、「primary commit 成功但 TiDB Server 崩溃」、「secondary 清理正在后台进行时读者遇到锁」这几类。

![[f10_4.svg]]

**图 10-4　分布式事务模型故障/异常路径**

源码上，`CheckTxnStatus` 的 TTL 判定为 `lock.ts.physical() + lock.ttl < current_ts.physical()` → 返回 `TxnStatus::TtlExpire`；`resolve_lock` 命令用 `txn_status` 把 `lock_ts` 映射到 `commit_ts`（回滚事务映射到 0）；async commit 事务因 primary 不一定先于 secondary 提交，需用 `check_secondary_locks.rs` 查询全部 secondary 锁状态来推断事务结局；`txn_heart_beat.rs` 负责为活跃大事务续 TTL。需要强调：每个推进动作（提交 secondary、回滚）本身又会写对应 Region，因此仍受该 Region 的 Raft 可用性影响——这是「TiDB 2PC commit」与「TiKV Raft commit 不是同一个 commit」的直接体现。

OceanBase 无独立协调者日志，跨 LS 故障恢复靠参与者 LS 的 Leader 重选后重放 PALF 中的 2PC 日志、并通过 orphan 消息 / 状态查询重建拓扑。传统 2PC 的危险在于 coordinator 或 participant 在决议前后崩溃，系统需要能判断事务是 commit、abort 还是 unknown；OceanBase 让 participant 持久化 prepare logs 与 commit logs，事务状态不只是内存变量，新 Leader 先经 PALF/Paxos 恢复日志边界，再据持久化状态继续推进。

![[f10_5.svg]]

**图 10-5　分布式事务模型故障/异常路径**

论文与源码均指出：`ObTxCycleTwoPhaseCommitter` 的「hierarchical cycle structure」正是为「在 transfer 过程中仍能成功提交事务」而设计；论文引入 prepare-unknown / trans-unknown 态处理参与者丢失上下文的情形，避免强制 abort 并屏蔽用户感知到的歧义态，最终由 Transaction Data Table 存储的 committed/aborted 决议解析。两条故障路径的对照很清楚：TiDB 的恢复锚点是可查询的 primary lock 状态，OceanBase 的恢复锚点是 LS 参与者持久化的 2PC 日志与 tx data。

---

## 10.7 性能、可靠性、运维影响

**延迟。** TiDB async commit / 1PC 把乐观提交压到约 1 次网络往返 + 1 次 Raft 多数派落盘（1PC 单 Region 时）。但跨 Region 事务的代价是结构性的：每多一个 Region，prewrite 阶段就多一组独立的 Raft 多数派往返；而「所有事务皆潜在分布式」意味着即使一行更新，只要带二级索引就可能跨 Region。一次事务涉及的 Region 越多，prewrite 的并行尾延迟、primary commit 所在 Region 的可用性、secondary 清理的后台压力越明显。OceanBase 单 LS 事务完全免 2PC、跨 LS 事务把可见延迟压到约 1 次 Paxos 同步；其参与者粒度是 LS 而非 key，把「100 个共置分区」收敛为 1 个参与者，从参与者数量级上降低协调开销（论文给出的「99% 下降」即此 N=100 解析算例，非实测，见 §10.13）。

**吞吐与扩展性。** 两者都靠「事务尽量本地化」获得扩展性：TiDB 靠 Region 局部性 + 1PC，OceanBase 靠 LS 局部性 + 单 LS 短路、Primary Zone 把相关数据聚到同一 LS。树形 2PC 论文（评测基于 OceanBase 4.4.1 社区版，落在锁定 4.4.x 区间内）报告其吞吐在 2→8 OBServer 扩展时呈近线性（TPC-C，3000 仓，500 并发连接）——这是论文中真实的实验设定与定性结论，但仍属论文 benchmark、非生产事实（见 §10.13）；需注意论文中的「99% 协调开销下降」与「1.2ms 单机延迟」是解析算例/设计目标而非该 TPC-C 配置下的实测值，不可与扩展性实验并列引用为同一组 benchmark 数据。

**热点。** TiDB 的热点 key/Region 会集中在一个 Region leader，二级索引写放大也会增加参与 Region 数；乐观事务在冲突高时可能到 commit 才失败，悲观事务把等待提前到 DML 阶段、以锁等待换取更少提交失败。OceanBase 的 LS 聚合多个 Tablet 减少了 Partition 级 2PC 数量，但也让一个 LS 的 leader、WAL、Paxos 多数派同步与 apply 成为共享路径；热点 Tablet 所在 LS leader 可能成为日志瓶颈，跨 LS 事务增加协调成本。（推测）

**可靠性与恢复。** 二者都不是把 2PC 与共识混成一个协议：2PC 决定跨参与者原子性，共识日志决定单参与者副本一致与持久。TiDB 残留锁惰性清理——客户端崩溃后，锁靠后续 reader 或 GC 触发 resolve，极端情况下「无人来读」的锁会停留到 TTL/GC；primary lock 状态必须经其 Region Raft log 持久化后才有恢复意义。OceanBase 状态机驱动恢复——Leader 切换后主动重放并重建，代价是 2PC 状态机 + reconfirm 更重（详见第 3 章）；LS participant 状态必须进入 PALF/Paxos 保护的 WAL/tx data 后才有恢复意义。

**表 10-3　性能、可靠性、运维影响**

| 性能/可靠性维度 | TiDB | OceanBase |
|---|---|---|
| 单「不跨边界」事务延迟 | 1PC：1 次 Raft 多数派 | 单 LS：1 次 Paxos 多数派 |
| 跨边界提交可见延迟 | async commit：prewrite 全成功即返回 | prepare ACK 后返回（约 1 次 Paxos 同步） |
| 跨边界事务的协调成本 | 随 Region 数线性增长（每 Region 一组 Raft） | 随 LS 数增长，但 LS 收敛了多分区 |
| 残留锁/未决事务恢复触发 | 惰性，reader/GC 触发 lock resolver | 主动，Leader 重选 + 日志重放 |
| 协调者单点 | 无独立协调者日志（primary lock 为锚） | 无独立协调者日志（第一个参与者代行） |
| 时间戳服务依赖 | TSO 集群级（跨地域延迟敏感） | GTS 租户级（单 LS 可免 RPC） |

**运维复杂度与观测。** TiDB 的事务问题常表现为「锁冲突/残留锁」，诊断需理解 primary/secondary 与 lock resolver；可从官方 dashboard 观察 prewrite、commit、取 commit_ts、async commit / 1PC 成功失败、事务类型 TPS 等方向（具体 metric 名以部署版本的 monitoring 文档为准）。OceanBase 的事务问题常表现为「长事务卡 2PC / leader transfer 期间未决」，诊断需理解树形 2PC 状态机与 LS 拓扑；可观察事务响应时间、事务日志耗时、LS leader 分布、跨 LS 事务比例等方向。事务级内部视图名已在源码确认（`GV$OB_TRANSACTION_PARTICIPANTS` / `GV$OB_TRANSACTION_SCHEDULERS`，详见 §10.9）；对应的 Prometheus / OCP 监控指标名则以部署版本的监控文档为准（待核实）。

--- 〔文献[14]〕

## 10.8 反例与代价

**TiDB 的代价。** 其一，「所有事务皆潜在分布式」：一个看似单行的 UPDATE，若更新了带二级索引的列，主表 key 与索引 key 可能落在不同 Region，即触发跨 Region 2PC、丧失 1PC 资格——这是高写入、宽索引表的隐性延迟来源。其二，async commit 把恢复责任转嫁给 reader：若一批锁的事务客户端崩溃且这批 key 长期无人读，锁会停留到 TTL 过期才被 GC/后续读清理，期间冲突事务被阻塞；参与 key 数过多还会增加 primary lock 中 secondary 信息的恢复成本。其三，乐观事务在高冲突下的批量回滚代价高，这也是 TiDB 默认改悲观的原因；但悲观锁每条 DML 多一次 RPC、锁请求本身要写 TiKV，在长事务、宽写集上放大延迟，并带来死锁/等待治理成本。其四，跨地域部署时 TSO 单一授时的往返延迟会直接计入每个事务（详见第 13 章）。

**OceanBase 的代价。** 其一，树形 2PC 状态机更复杂（`REDO_COMPLETE/PRE_COMMIT/CLEAR` 等工程态 + orphan 处理 + transfer 钩子），实现与调试门槛高于 Percolator 风格；leader transfer 与 2PC 交织时的未决态处理是公认难点（论文专门引入 unknown 态）。其二，单 LS 短路的收益依赖数据布局——若业务把强相关的热点 Tablet 散落到多个 LS，跨 LS 2PC 不可避免，且 LS 聚合会把日志写入、Paxos 同步、状态推进与恢复集中到 LS leader 路径上；Primary Zone / Locality 调优（详见第 1、13 章）是把延迟压下来的前提，运维负担转移到「布局设计」。其三，协调者下沉省了协调者日志，但恢复时需查询全部参与者，参与者多、网络抖动时恢复链路变长；Tablet transfer、leader 切换与 GTS/SCN 等待叠加时，故障状态比传统 2PC 更复杂。

**替代方案也有代价。** 每个 key 或 Tablet 都做独立参与者会让并行性更细，但 2PC fan-out 与恢复状态会膨胀；所有数据集中到一个参与者可降低协议复杂度，但牺牲扩展性与热点隔离。TiDB 与 OceanBase 的差别来自分片/复制边界与历史工程路径，而非「Percolator 必然更简单」或「Paxos-based 2PC 必然更先进」。二者并无「谁更先进」之分，只是把复杂度放在了不同位置：TiDB 选择「协议简单 + 恢复惰性（读触发）」，代价是潜在分布式无处不在、锁清理时机不可控；OceanBase 选择「协议复杂（树形优化 2PC）+ 恢复主动（状态机驱动）」，代价是实现复杂度与布局敏感。（推测）

---

## 10.9 测试开发视角的验证点

**可测试的功能场景。** TiDB 侧应覆盖单 key、单 Region 多 key、跨 Region 多 key、主表加二级索引、唯一索引冲突、乐观提交冲突、悲观锁等待、pipelined pessimistic lock、async commit 命中与回退、1PC 命中与不命中。OceanBase 侧应覆盖单 LS 事务、跨 LS 事务、多个 Tablet 同 LS、Tablet transfer 期间提交、MySQL 模式与 Oracle 模式隔离级别差异、SCN/read version 在 RC 与 RR 下的可见性。隔离级别正确性是共同重点：SI 下的写偏序（write skew）应被允许、幻读应被避免。

**可注入的失效模式。**
- TiDB：prewrite 后、primary commit 前 kill 客户端 → 验证残留锁能否被后续 reader 经 CheckTxnStatus / ResolveLock 正确清理；构造 async commit 事务后杀 client → 验证 `check_secondary_locks` 推断结局；TTL 过期边界；Region leader 隔离、primary key 所在 Region 慢盘、secondary Region 超时、PD TSO 短暂不可用、TiKV apply 堵塞。
- OceanBase：跨 LS 事务 prepare 后、commit 前对某参与者 LS 触发 leader transfer / kill leader → 验证新 Leader 重放 2PC 日志后事务结局一致；对协调者（第一个参与者）注入故障 → 验证 coordinator-less 恢复；participant prepare log 落后、PALF 多数派抖动、GTS provider 切换、coordinator 与某些 LS 网络分区、Tablet transfer 与 2PC 重叠。

**关键压测指标。** 跨 Region/LS 事务比例对 P99 提交延迟的影响；冲突率对回滚率/重试率的影响；1PC / 单 LS 命中率。

**一致性断言要写到故障之后，而非只看 COMMIT 返回码。** TiDB 应断言 primary commit 成功后任何读者最终能看到全事务结果，primary 未提交且过期后不能只提交一部分 secondary；并区分「用户可见成功」与「后台收尾完成」——primary commit 后返回成功时 secondary locks 可能仍未清理，测试不能把这一瞬间的 lock 误判为脏数据，而要验证后续 resolver 能正确推进。OceanBase 应断言 prepare 后 participant 不能把未知状态当 abort，commit 决议后所有 LS 最终同向完成或可恢复查询到确定状态；coordinator 进入 COMMIT 后返回成功，测试也不能在客户端返回时立即检查所有 participant 都已释放资源，而要检查系统能否在 leader 切换、网络恢复或重试后收敛到一致状态。这类断言更接近事务恢复正确性，需用历史读、并发读写与故障注入组合验证。

**关键观测指标 / 诊断命令**（仅列可查证者，不编造）：
- TiDB：官方文档《Troubleshoot Lock Conflicts》给出锁冲突排查路径；TiDB 提供事务/锁相关系统表与 `tikv_gc_*` 相关变量用于 GC 与锁清理观测；async commit / 1PC 可由系统变量（如 `tidb_enable_async_commit`、`tidb_enable_1pc`）开关——具体 Prometheus metric 名需对照部署版本的 monitoring 文档查证。
- OceanBase：事务/2PC 状态可经内部视图族观测，精确视图名已在源码 `src/share/inner_table/ob_inner_table_schema_constants.h` 确认为 `GV$OB_TRANSACTION_PARTICIPANTS` / `V$OB_TRANSACTION_PARTICIPANTS`（TID 21225/21226，底层取自 `__all_virtual_trans_stat`，列含 `TX_TYPE` LOCAL/DISTRIBUTED、`LS_ID`、`PARTICIPANTS`、`STATE`、`COORD`、`ROLE`、`PENDING_LOG_SIZE` 等）与 `GV$OB_TRANSACTION_SCHEDULERS` / `V$OB_TRANSACTION_SCHEDULERS`（TID 21353/21354，列含 `COORDINATOR`、`PARTICIPANTS`、`ISOLATION_LEVEL`、`STATE`、`SNAPSHOT_VERSION` 等）；其 `STATE` 列的取值映射（`PREPARE`/`PRECOMMIT`/`COMMIT`/`ABORT`/`CLEAR`）与 §10.3 的 `ObTxState` 枚举一一对应。源码侧 2PC 状态由 `ObTxState` 枚举驱动、日志类型由 `ObTwoPhaseCommitLogType` 标识，可用于日志级排查。三个版本（v4.2.5_CE / 4.3.5 / 4.4.x）视图名一致。

---

## 10.10 容易误解点

**误解一：「TiDB 所有事务都是分布式事务，所以都走 2PC，延迟必高。」** 不准确。从实现路径看任何多 key 写都「潜在分布式」，但 1PC（单 Region 全部 key）直接跳过 commit 阶段、async commit 让跨 Region 事务在 prewrite 后即返回。真正会退化为完整 2PC 的是「写集跨多个 Region」——而是否跨 Region 取决于数据/索引布局，不是事务条数。所以正确表述是：潜在分布式 ≠ 实际都付完整 2PC 代价。

**误解二：「TiDB 的 2PC commit 和 TiKV 的 Raft commit 是同一个 commit。」** 不准确。前者是跨 key/Region 的事务提交阶段，由 primary lock 状态判定；后者是单 Region Raft log 多数派提交，负责副本复制持久。prewrite 和 commit 都会对 Region 状态产生写入，因此都需要落到相应 Region 的 Raft 复制路径——一次跨 Region 事务的提交链路上叠加了多次独立的 Raft 多数派往返。

**误解三：「OceanBase 的『单 LS 事务』就是『单机事务』。」** 不准确。单 LS 事务指参与者只涉及一个 LS，该 LS 仍是多副本、redo 仍要经 Paxos/PALF 多数派持久化；它免掉的是跨 LS 的 2PC 协调，而非免掉复制与持久化。跨 LS 才进入分布式 2PC，且参与者是 LS，不是单个 Tablet。

**误解四：「OceanBase 的优化 2PC 没有协调者，所以没有单点。」** 不准确。「无独立协调者日志」指的是不为协调者单独持久化全局状态、改由第一个参与者代行并靠查询参与者恢复；但这不等于「没有逻辑中心」——协调角色仍存在于第一个参与者 LS 的 Leader 上。按本研究的「单点」五维度（逻辑中心化/性能瓶颈/高可用单点/元数据依赖/故障恢复依赖）判断：它消除的是「协调者日志 SPOF」，但协调逻辑、GTS 授时（详见第 13 章）仍是需要分别讨论的依赖维度。

**误解五：「async commit / prepare 后返回 = 还没落盘就告诉客户端成功，可能丢数据。」** 不准确。两者都要求相关日志已多数派持久化才返回：TiDB async commit 在所有 key 的 prewrite（经 Raft 多数派）成功后才返回；OceanBase 在 prepare（redo 经 Paxos 多数派）成功后才返回。「省掉」的是协调者写全局 commit 日志这一步的等待，而非省掉持久化本身——commit_ts/SCN 的确定性与 min_commit_ts 校验保证了正确性。

---

## 10.11 本章结论

1. TiDB 事务模型是 Percolator-like 客户端协调 2PC：primary lock 是事务状态的真相之源，事务「真相」分布式编码进每个 key 的 lock/write 列；2PC 主循环在外部库 client-go，TiDB Server 与 TiKV `src/storage/txn/` 只负责委托与生成 MVCC 修改。
2. OceanBase 事务模型是协调者下沉的优化版树形 2PC，以 Log Stream-LS 为原子参与者；状态机 `ObTxState` 多出 `REDO_COMPLETE/PRE_COMMIT/CLEAR` 工程态，角色 `ROOT/INTERNAL/LEAF` 构成树形拓扑，专为 leader transfer 设计（三版本枚举一致）。
3. 两者都让「不跨边界」的事务短路：TiDB 用 1PC（单 Region）、OceanBase 用单 LS 单分支提交；跨边界时都把客户端可见延迟压到约 1 次多数派往返（TiDB async commit prewrite 后返回 / OceanBase prepare ACK 后返回）。
4. 事务提交与共识是两层叠加、不可混为一谈：TiDB 的 prewrite 与 commit 各自要经对应 Region 的 Raft 多数派；OceanBase 的 redo / 2PC 各阶段日志经对应 LS 的 Paxos 多数派，SCN 即日志/版本时间戳。
5. 故障恢复哲学相反：TiDB 惰性、读触发（reader 经 CheckTxnStatus→ResolveLock 清残留锁，每个推进动作仍受 Region Raft 可用性影响）；OceanBase 主动、状态机驱动（Leader 重选重放 2PC 日志 + orphan 消息重建，依赖 tx data 决议解析）。
6. 隔离级别均以 SI 为底层：TiDB 对外称 RR 实为 SI、另支持 RC（RC 仅悲观事务模式生效）；OceanBase MySQL 模式仅支持 RC/RR、Oracle 模式支持 RC/RR/Serializable，两种模式默认均为 Read Committed（MySQL 模式不含 Serializable）；OceanBase 的 Serializable（仅 Oracle 模式）实际实现为快照隔离、允许 write skew。
7. 二者无「谁更先进」：TiDB 把复杂度放在「潜在分布式无处不在 + 锁清理时机不可控」，OceanBase 把复杂度放在「树形 2PC 状态机 + 数据布局敏感」，是不同的工程取舍。（推测）

---

## 10.12 参考文献

[1] Large-scale Incremental Processing Using Distributed Transactions and Notifications (Percolator). 论文，Google OSDI 2010[EB/OL]. https://research.google/pubs/large-scale-incremental-processing-using-distributed-transactions-and-notifications/.
 （支撑:§10.2/§10.6 TiDB 事务模型的协议理论基座：2PC + primary/secondary lock + 三列编码 + 时间戳、崩溃恢复与 roll forward/rollback 判定。）
[2] TiDB: A Raft-based HTAP Database. 论文，VLDB 2020[EB/OL]. https://www.vldb.org/pvldb/vol13/p3072-huang.pdf.
 （支撑:§10.2 TiDB 乐观/悲观事务、PD timestamp、选 primary key、prewrite、primary commit 后返回与 secondary 异步清理的系统设计描述。）
[3] A Tree-Structured Two-Phase Commit Framework for OceanBase: Optimizing Scalability and Consistency. 论文，arXiv 2026[EB/OL]. https://arxiv.org/abs/2603.00866.
 （支撑:§10.3/§10.6/§10.8 OceanBase 树形 2PC 的 LS 原子参与者、树形拓扑、prepare-unknown/trans-unknown 态恢复与延迟模型；评测基于 OceanBase 4.4.1）
[4] OceanBase: A 707 Million tpmC Distributed Relational Database System. 论文，VLDB 2022[EB/OL]. https://www.vldb.org/pvldb/vol15/p3385-xu.pdf.
 （支撑:§10.3 OceanBase Paxos-based 2PC、timestamp Paxos group 与减少协调者状态持久化/Paxos 同步次数的论文层描述（该 707M tpmC 审计成绩的完整身份与适用边界见）
[5] TiDB Transaction Isolation Levels. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/transaction-isolation-levels/.
 （支撑:§10.2/§10.11 TiDB 在 RR 名称下实现 SI、RC 仅悲观事务模式生效、write skew 与幻读行为。）
[6] TiDB Optimistic / Pessimistic Transaction Mode. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/pessimistic-transaction/.
 （支撑:§10.2/§10.7 悲观锁、for_update_ts、提交仍走 2PC、pipelined locking 与 Raft commit/apply 关系，以及乐观事务取 start_ts、按 Region 分组）
[7] TiKV Deep Dive — Percolator. 官方文档[EB/OL]. https://tikv.org/deep-dive/distributed-transaction/percolator/.
 （支撑:§10.2/§10.6 prewrite/commit 两阶段、lock/write/data 三 CF、读路径遇锁处理与崩溃恢复的精确步骤。）
[8] Async Commit / 1PC / Lock Resolver — TiDB Development Guide. 官方设计文档[EB/OL]. https://pingcap.github.io/tidb-dev-guide/understand-tidb/async-commit.html.
 （支撑:§10.2 async commit 取消等 primary commit、min_commit_ts 计算（max_ts/start_ts/for_update_ts）与 secondaries 列表恢复，及 1PC）
[9] Cluster architecture — Log stream / GTS. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001103745.
 （支撑:§10.3/§10.4 OceanBase LS 含 Tablet 与 ordered redo logs、用 Paxos 同步日志、单 LS one-phase / 跨 LS two-phase atomic comm）
[10] OceanBase MySQL 模式的事务隔离级别（V4.2.2）. 官方文档[EB/OL]. https://www.oceanbase.com/docs/common-oceanbase-database-cn-1000000000510596.
 （支撑:§10.4/§10.9/§10.11 OceanBase MySQL 模式隔离级别集合：官方明确 MySQL 模式仅支持读已提交/可重复读、不含 Serializable，Serializable 仅 Oracle 模式）
[11] tikv/tikv（release-8.5 @ 1f8a140b6d46）— src/storage/txn/、src/storage/mvcc/、components/txn_types/src/、src/server/raftkv/mod.rs、components/raftstore/src/store/peer.rs. 源码[EB/OL]. https://github.com/tikv/tikv/blob/release-8.5/src/storage/txn/actions/prewrite.rs.
 （支撑:§10.2/§10.6 prewrite/commit/CommitKind/TransactionKind/Lock 字段、min_commit_ts 校验、check_txn_status/check_seconda）
[12] pingcap/tidb（release-8.5 @ 67b4876bd57b）— pkg/store/driver/txn/txn_driver.go、go.mod. 源码[EB/OL]. https://github.com/pingcap/tidb/blob/release-8.5/pkg/store/driver/txn/txn_driver.go.
 （支撑:§10.2 TiDB Server 仅委托给 client-go 的 KVTxn、下传 async commit/1PC/隔离级别/悲观事务选项，2PC 主循环不在 tidb 仓库内的边界声明；并据 go.mod 确）
[13] oceanbase/oceanbase（v4.2.5_CE @ e7c676806fda）— src/storage/tx/、src/storage/tx_storage/、src/storage/tx/ob_trans_part_ctx.cpp、src/storage/tx/ob_ts_mgr.{h,cpp}、src/storage/tx/ob_gts_source.h. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/blob/v4.2.5_CE/src/storage/tx/ob_committer_define.h.
 （支撑:§10.3/§10.6 ObTxState/Ob2PCRole 枚举、one-phase/two-phase committer 与 leader transfer 注释、ObTxCycleTwoPhaseCommitt）
[14] oceanbase/oceanbase（v4.2.5_CE @ e7c676806fda）— src/share/inner_table/ob_inner_table_schema_constants.h 及 ob_inner_table_schema.21201_21250.cpp / .21351_21400.cpp. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/blob/v4.2.5_CE/src/share/inner_table/ob_inner_table_schema_constants.h.
 （支撑:§10.7/§10.9 OceanBase 事务内部视图精确名与定义：GV$OB_TRANSACTION_PARTICIPANTS/V$OB_TRANSACTION_PARTICIPANTS（TID 21225/）
[15] Introduction to the OceanBase Transaction Engine: Features and Applications. 官方博客[EB/OL]. https://oceanbase.medium.com/introduction-to-the-oceanbase-transaction-engine-features-and-applications-27159d10b9d2.
 （支撑:§10.3 协调者下沉（第一个参与者代行）、prepare 后返回、GTS 外部一致性、只有 redo log 无 undo log。）
[16] Deep Dive into Distributed Transactions in TiKV and TiDB. 第三方解读，Medium[EB/OL]. https://dataturbo.medium.com/deep-dive-into-distributed-transactions-in-tikv-and-tidb-80337b4104cb.
 （支撑:§10.2 对 Percolator-on-TiKV 整体路径的交叉理解。第三方解读，细节以官方源码/文档为准，存在版本滞后风险。 ---）
## 10.13 信息可信度自评

本章级事实主要有两类支撑：一是直接核对的 GitHub 源码（TiKV release-8.5 @ `1f8a140b6d46` 的 prewrite/commit/lock 类型与 min_commit_ts 校验、OceanBase v4.2.5_CE @ `e7c676806fda` 的 `ObTxState`/`Ob2PCRole` 枚举与 `ObTxCycleTwoPhaseCommitter` 注释，均经实际核对 GitHub 源文件逐字确认，并另据 v4.4.2_CE 源码交叉确认枚举与 committer 结构一致），二是官方文档/设计文档（TiKV deep-dive、tidb-dev-guide、OceanBase 官方架构/GTS/隔离级别文档与官方博客）。论文级事实来自 Percolator 论文、TiDB VLDB 论文、OceanBase VLDB 2022 论文与树形 2PC arXiv 论文。源码引用均 commit-pinned 至统一基线。经源码核验补强后，以下原「需进一步查证」项已升级为源码确认事实：TiKV raftstore 提交链路（`RaftKv::async_write → router.send_command → Peer::propose_normal`，§10.2）、OceanBase 单 LS 短路提交判定（`ObPartTransCtx::commit` 的 `participants_.count()==1 && participants_[0]==ls_id_` 分支，§10.3）、GTS 缓存刷新周期常量（`REFRESH_GTS_INTERVEL_US = 500ms`，§10.3）、OceanBase 事务内部视图精确名（`GV$OB_TRANSACTION_PARTICIPANTS` / `GV$OB_TRANSACTION_SCHEDULERS`，§10.9），均经 v4.2.5_CE / 4.3.5 / 4.4.x 或 tikv release-8.5 源文件逐字核对。仍保留「需进一步查证」者：TiDB 跨 Region 2PC 主循环位于外部库 client-go，其依赖版本已由 tidb `go.mod` 锁定为 `v2.0.8-0.20260615130046-a2b634d170d9`，但该仓库本身未 checkout 到本基线，故主循环函数体未逐字核对；OceanBase 事务对应的 Prometheus/OCP 监控指标名未逐项核验。未臆造。

**反查证发现的冲突（已据官方源更正）：**

1. OceanBase MySQL 模式隔离级别集合。经核对 OceanBase 官方简体文档《MySQL 模式的事务隔离级别》（V4.2.2，落在锁定 4.2.x 族），官方明确：MySQL 模式仅支持读已提交（RC）与可重复读（RR），Serializable 仅 Oracle 模式支持，两种模式默认隔离级别均为读已提交。早期英文页（en.oceanbase.com，历史路径含 V2.2.77）「MySQL 模式含 Serializable」的表述与现行简体官方页冲突，据「版本/口径错配、缺乏坚实独立支撑」规则，§10.4/§10.9/§10.11 统一采用「MySQL 模式 RC/RR、Oracle 模式 RC/RR/Serializable、默认均 RC」，并把「Serializable 实为快照隔离、允许 write skew」限定到 Oracle 模式语义。

2. 树形 2PC 论文的数字出处。经核对 arXiv 论文全文，只有「2→8 节点近线性扩展」是 TPC-C 3000 仓/500 并发实验的真实结论；「协调开销下降约 99%」是摘要/引言中 N=100 个共置分区聚合为 1 个 LS 参与者的解析算例（参与者数量级缩减），「单机延迟约 1.2ms」是摘要/引言陈述的设计目标/特性，两者均未在评测章节作为该 TPC-C 配置下的实测值给出。将三者打包成「一套论文 benchmark 数据」属数值出处不符，已在 §10.3/§10.7 拆分：近线性扩展标，99%/1.2ms 改标。论文评测版本为 OceanBase 4.4.1 社区版，落在锁定 4.4.x 区间内，版本无错配。

3. 协调者恢复时间口径。官方博客提及的协调角色恢复时间，与第 1/3/13 章公共头记录的集群级 RPO=0/RTO<8s 指标不是同一口径（前者是 2PC 协调角色恢复、后者是数据分区级容灾 SLA），本章按「不同口径」区别引用，未混用，亦未将该指标外推到本章。

**版本核查备注：** 本章 TiDB 源码锁定 release-8.5、主基准 8.5.x（对照 7.5.x）；OceanBase 源码锁定 v4.2.5_CE（TP LTS），并标注树形 2PC 枚举与 committer 结构在 4.3.5 / 4.4.x 一致；隔离级别引 OceanBase 官方 V4.2.2 简体页（与锁定 4.2.x 族一致），据其确认 MySQL 模式不含 Serializable、默认 RC，无与锁定版本冲突的强断言。OceanBase tree-shaped 2PC 的具体产品化版本归属仍以 arXiv 论文与源码注释为主，若后续获得官方 release note 更明确的发布说明，应优先替换本章对应风险备注。

---


# 第 11 章 MVCC 与读路径

## 11.1 本章核心问题

多版本并发控制（MVCC）是分布式数据库实现「读不阻塞写、写不阻塞读」与快照隔离（Snapshot Isolation）的物理基础。但 MVCC 的核心问题不是「能不能保存多个版本」，而是：当分布式复制、事务提交、后台清理与副本延迟同时存在时，一条读请求如何判断哪个版本可见、是否需要等待锁、是否必须访问 Leader，以及历史版本何时可以安全删除。本章要回答的核心问题是：**一条读请求，从拿到读取版本（read timestamp / snapshot version）开始，到底层存储引擎返回「该版本应当可见的那一行」，中间经过哪些数据结构、控制判断与故障分支？**

这个问题之所以关键，是因为它把前面几章的概念落到了真正的字节布局上，是事务章与存储章的交汇点：

- 第 10 章讲的事务(Percolator 两阶段 / OceanBase 两阶段提交)决定了一个版本「何时生成、何时可见」；本章关注这个版本「物理上长什么样、读时如何被挑出来」(详见第 10 章)。
- 第 4 章讲的存储引擎(RocksDB CF / OceanBase 自研 LSM)决定了多版本「落在哪一层」；本章关注多版本在 MemTable 与 SSTable 中的**组织方式**与**清理时机**(详见第 4 章)。
- 第 13 章讲的 TSO/GTS 决定了读版本号「从哪里来」；本章默认读版本号已经拿到，关注它如何驱动可见性判断(详见第 13 章)。

具体到两个系统，本章聚焦四组追问：(1)多版本怎么编码——TiKV 是「user key + 倒序时间戳后缀」，OceanBase 是「行级 trans node 链表 + SSTable 隐藏版本列」；(2)读路径要不要走 Leader——快照读、当前读、悲观锁读、stale read / follower read / 弱一致读各走哪条路；(3)历史版本怎么回收——TiDB 的 GC safepoint 与 compaction filter，OceanBase 的 row compaction 与 minor/major merge;(4)长事务、版本堆积、merge 进度如何影响读延迟。

贯穿全章有一条判断标准：**读路径是否需要 Leader，取决于「读的是最新线性化结果，还是可证明安全的历史版本」**。在 TiDB 中，普通强一致读通常需要 Region Leader 或 ReadIndex 验证，stale read 在 safe-ts 允许时可读本地副本；在 OceanBase 中，默认强一致读走 Leader，弱读可优先读 Follower，但要受 bounded staleness 与弱读快照约束。读路径是数据库在「读多写少」负载下最热的代码路径，其设计取舍直接决定 P99 延迟与 CPU 水位。本章只讨论 MVCC 范围，不展开共识协议、分片路由与完整 2PC 流程(分别见第 3、1、10 章)。

---

## 11.2 TiDB 的实现

### 11.2.1 多版本编码：Percolator 三 CF + 倒序时间戳

TiDB 的 MVCC 基础来自 Google Percolator 论文的三列思想：每个逻辑 cell 拆成 data、write metadata、lock metadata——lock 保存未提交事务及 primary lock 位置，write metadata 记录可见提交信息，data 保存实际值。TiKV 没有照搬 Bigtable 的列族命名，而是在单个 RocksDB 实例上划分三个数据列族(Column Family)承载用户 MVCC 数据(write CF 存版本信息、default CF 存真实数据的三 CF 划分，见 pingcap.medium.com MVCC in TiKV):

- `CF_DEFAULT`(数据列):`(user_key, start_ts) -> value`，存放真实行数据。
- `CF_LOCK`(锁列):`user_key -> lock_info`，存放未提交事务的锁(primary lock / secondary lock / 悲观锁 / 共享锁)。
- `CF_WRITE`(提交列):`(user_key, commit_ts) -> write_info`，存放已提交版本的「提交记录」，其 value 内含该版本数据的 `start_ts`，即指向 `CF_DEFAULT` 中对应数据的指针。

源码层面，三个 CF 名字在引擎 trait 中是固定常量：`CF_DEFAULT="default"`、`CF_LOCK="lock"`、`CF_WRITE="write"`，数据 CF 分组 `DATA_CFS` 长度为 3(default/lock/write)，`ALL_CFS` 再追加 `CF_RAFT`(tikv/tikv `components/engine_traits/src/cf_defs.rs` @ release-8.5，commit `1f8a140b6d46`)。写入侧由 `src/storage/mvcc/txn.rs` 落库：`put_lock` 写 `CF_LOCK`，`put_value` 把 key 附加事务 `start_ts` 后写 `CF_DEFAULT`，`put_write` 把 key 附加提交时间 `commit_ts` 后写 `CF_WRITE`；`unlock_key` 在提交或回滚时删除 lock CF 条目。

MVCC 的「版本」物理上由 **key 后缀的时间戳**承载。`Key::append_ts(ts)` 把 8 字节时间戳以**降序编码**(`encode_u64_desc`)附在 user key 之后(`components/txn_types/src/types.rs` @ release-8.5)。降序编码是读路径性能的物理基础：同一 user key 的多个版本中，commit_ts 越大的内部 key 排得越靠前，因此对某个读版本 `ts` 做一次 RocksDB `seek`，即可直接定位「commit_ts ≤ ts 的最新版本」，无需逐版本回扫。

`CF_WRITE` 里的提交记录有四种类型：`enum WriteType { Put, Delete, Lock, Rollback }`，flag 字节分别为 `P/D/L/R`；`struct Write` 字段含 `write_type`、`start_ts`、`short_value`、`has_overlapped_rollback`，并可携带 `gc_fence`/`last_change` 等扩展(`components/txn_types/src/write.rs`)。值得注意的是 Rollback 记录也复用 write CF——这意味着写 CF 不只是「提交版本表」，还混入了回滚标记。`CF_LOCK` 的锁类型为 `enum LockType { Put, Delete, Lock, Pessimistic, Shared }`(flag `P/D/L/S/H`)，锁体可携带 `for_update_ts`、`min_commit_ts`、`txn_size` 等可选字段(`components/txn_types/src/lock.rs`)。

**短值内联优化**:`SHORT_VALUE_MAX_LEN = 255`，当 value ≤ 255 字节时直接内联进 write CF 的 `short_value` 字段，读时无需回 default CF 取数；超过 255 字节才把 value 落 default CF(`components/txn_types/src/types.rs`)。这是一项降低读放大的优化——大量短行(整型、短字符串)读取只需一次 write CF seek。

### 11.2.2 数据路径：快照读如何挑出可见版本

![[f11_1.svg]]

**图 11-1　TiKV 三列族物理布局与一次快照读跨 CF 的指针追踪**

在上述编码之上，读路径要做的只有一件事：据读版本号从多版本中挑出可见的那一行。TiDB 读路径由 TiDB Server 拿到 `start_ts`(从 PD 的 TSO 申请，详见第 13 章)，下推到 TiKV 后由 MVCC reader 完成。点读路径从 `PointGetter` 看得最清楚：它在 SI 等需要检查锁的隔离级别下先读 lock CF，若锁与读时间戳冲突则返回锁冲突或按可访问锁读取；若无冲突，在 write CF 中寻找该 user key 上 `commit_ts ≤ read_ts` 的第一个可见 write record。

一次快照读的逻辑序列：

1. **检查锁**：在 `CF_LOCK` 中查 user key 是否存在 `lock_ts ≤ ts` 的锁。若存在且非自身事务，可能命中未决事务，需走 lock resolver 等待或回查 primary lock 状态(详见第 10 章)。
2. **定位提交版本**：在 `CF_WRITE` 上 seek `(user_key, ts)`，取出 commit_ts ≤ ts 的最新提交记录，读出其中的 `start_ts`。这一「读时按 commit_ts 反查 start_ts」的过程，tikv.org Percolator Deep Dive 原文表述为 "Get the latest record in the row's `write` column whose `commit_ts` is in range `[0, ts]`. The record contains the `start_ts`..."，随后 "Get the row's value in the `data` column whose timestamp is exactly `start_ts`"(tikv.org Percolator Deep Dive)。
3. **按 WriteType 分支取数**：遇到 `Put` 时，若 write record 内联了 `short_value` 则直接返回值；否则用 `(user_key, start_ts)` 回 `CF_DEFAULT` 取真实 value。遇到 `Delete` 返回空；遇到 `Lock` 或 `Rollback` 则继续向更旧版本迭代。

因此 TiKV 的两个时间戳不能混用：write CF 按 commit_ts 排序以决定可见性，default CF 按 start_ts 定位数据负载，read_ts/start_ts 是读事务选择可见版本的边界。范围读路径使用 `MvccReader` 和 scanner，原则类似：write cursor 决定每个 user key 在 read_ts 下的可见 write，data cursor 只在需要 value 时访问 default CF，lock cursor 用于锁检查；`MvccReader` 提供 `get(key, ts)`、`get_write(key, ts)`、`seek_write(key, ts)`、`get_txn_commit_record(key)` 等接口，`seek_write` 通过 `cursor.near_seek(key.append_ts(ts))` 在 write CF 上按降序 ts 寻找 ≤ ts 的提交记录(`src/storage/mvcc/reader/reader.rs`)。点读分支在 `PointGetter::load_data` 中亦逐字可核：`SI`/`RcCheckTs` 下先 `load_and_check_lock` 读 lock CF，无冲突后 seek write CF 取 commit_ts ≤ ts 的记录，再按 `match write.write_type` 分支——`Put` 返回内联 `short_value` 或回 default CF 取 `(user_key, start_ts)`、`Delete` 返回 `None`、`Lock | Rollback` 借 `last_change` 快捷指针或继续向旧版本迭代(`src/storage/mvcc/reader/point_getter.rs`)。源码中 `SnapshotReader` 的注释明确说它表示某个 `start_ts` 上的 MVCC 逻辑视图，而底层 RocksDB snapshot 仍包含多个时间戳版本——这解释了为何 TiDB 的「快照读」不是复制一个物理快照，而是以 TSO 分配的读时间戳在多版本 keyspace 中过滤。

### 11.2.3 读类型：快照读 / 当前读 / 悲观锁读 / stale read / follower read

同样是「读」，路由与代价却因读类型而分岔。下面按是否加锁、读最新还是历史、是否必须经 Leader 逐类陈述。

- **快照读(snapshot read)**：不加锁，读事务开始前已提交的版本。普通 `SELECT` 即快照读。
- **当前读(current read)**：加锁，读「最新已提交版本」。`UPDATE/DELETE/INSERT/SELECT ... FOR UPDATE` 是当前读，在悲观事务下对最新版本加悲观锁(docs.pingcap.com pessimistic-transaction)。当前读是**不可重复读**的，这点与快照读的可重复语义不同。这条路径要把读路径与锁路径接上：悲观事务下的 `SELECT ... FOR UPDATE` 走锁检查/加锁语义，避免读到旧版本后又被并发写破坏。TiKV 源码中还存在内存悲观锁表：`MvccReader::load_lock` 会先尝试读取 in-memory pessimistic lock，再读 lock CF；读锁表时还校验 term 和 Region epoch，防止 Leader/epoch 变化后把已经清理的锁误报给客户端。这类路径的关键代价是冲突时需要 lock resolution——读请求发现锁后要查询 primary lock 或事务状态，决定等待、回滚、清锁或重试；这不是单纯的 LSM seek 成本，而是跨事务状态与复制状态的控制路径成本。
- **follower read**：由 `tidb_replica_read` 系统变量控制，取值含 `leader`(默认)、`follower`、`leader-and-follower`、`prefer-leader`、`closest-replicas`、`closest-adaptive`、`learner`(docs.pingcap.com follower-read)。follower 读为了不破坏快照隔离与线性一致，往往仍要向 Leader 发 `ReadIndex`：follower 先向 Leader 取最新 commit index，等本地 apply 追平后再服务读(详见第 3 章共识)。因此 follower read **不省 Leader 的一次往返**，只省 Leader 的数据扫描 CPU。
- **stale read**：由 `AS OF TIMESTAMP`/`TIDB_BOUNDED_STALENESS()` 或 `tidb_read_staleness` 指定一个稍旧的历史读版本。其安全边界由 safe-ts/resolved-ts 决定：官方文档说明，若某个 Region peer 上读请求时间戳 `ts ≤ safe-ts`，TiDB 可以安全地从该 peer 本地读取，而 TiKV 通过保证 safe-ts 不大于 resolved-ts 来维护这个条件(docs.pingcap.com troubleshoot-stale-read / stale-read)。因此 stale read **任意副本均可服务、无需 ReadIndex 往返**——读版本早于 resolved-ts，副本本地即可判定该版本已全局确定，在跨地域部署中可避免跨中心网络延迟。TiKV 侧 `advance-ts-interval` 默认 20s，调小可让 resolved-ts 推进更频繁、stale read 可读更「新」的历史点，代价是 CPU 上升。

这是「读路径是否必须走 Leader」的第一张分界线：最新强一致读需要 Leader 或 Leader 参与的线性化证明(直读 Leader 或 follower 的 ReadIndex)，历史读只要本副本已经推进到足够安全的时间戳，就可以绕开 Leader 读本地数据。

### 11.2.4 控制路径：GC safepoint 与历史版本清理

TiDB 的 GC 由 TiDB Server 侧的 GC worker 主导，官方文档把 GC 分为 Resolve Locks、Delete Ranges、Do GC 三步：先扫描 safe point 前的锁并清理，再快速清理 `DROP TABLE/INDEX` 产生的范围，最后由 TiKV 扫描并删除各 key 不再需要的旧版本(docs.pingcap.com garbage-collection-overview)。默认参数(常量块):`gcDefaultRunInterval=10min`、`gcDefaultLifeTime=10min`、`gcMinLifeTime=10min`、`gcDefaultConcurrency=2`，GC mode 默认 `distributed`(`central` 模式已废弃)(tidb/tidb `pkg/store/gcworker/gc_worker.go` @ release-8.5，commit `67b4876bd57b`)。`safe_point = now - gc_life_time`，经 oracle 转成 TSO 后写入 `mysql.tidb` 系统表 key `tikv_gc_safe_point`；GC worker 由 leader 选举产生(`tikv_gc_leader_uuid`)。TiKV 侧由 `GcManager` 周期轮询该 safe point，生产默认轮询间隔为常量 `POLL_SAFE_POINT_INTERVAL_SECS = 10`(秒,见 `AutoGcConfig::new`;`new_test_cfg` 测试配置则缩到 100ms)(tikv/tikv `src/server/gc_worker/gc_manager.rs` @ release-8.5)。

物理清理在 TiKV 侧由 **compaction filter** 完成(8.x 主用路径):`WriteCompactionFilterFactory` 在 RocksDB compaction 时读取 `gc_context.safe_point`，若 `safe_point == 0` 则不 GC，否则通过 `check_need_gc(safe_point, ratio_threshold, ...)` 决定是否随 compaction 物理删除过期版本(tikv/tikv `src/server/gc_worker/compaction_filter.rs` @ release-8.5)。`GcConfig` 默认 `ratio_threshold=1.1`、`batch_keys=512`、`enable_compaction_filter=true`，且 compaction filter 默认仅在 `cluster_version > 5.0.0` 时生效(`src/server/gc_worker/config.rs`)。删除时由版本状态机决策：保留维持语义所需的边界版本，删除旧 Rollback/Lock 和更老版本，并在删除非 short value 的 Put 时同步删除 default CF 中的 value。compaction filter 只作用于 write CF(MVCC 数据所在)，避免了早期「逐 key scan + 写 tombstone」式 GC 带来的额外读放大与 tombstone 顺序扫描退化(docs.pingcap.com garbage-collection-configuration)。

**长事务对 GC 的影响**:safepoint 必须小于所有活跃事务的最小 `start_ts`，否则会清掉活跃事务仍需读的版本。自 TiDB v6.1.0 起，GC safe point 计算会考虑事务 startTS，一个长期不提交的事务会卡住 safepoint 推进、阻塞空间回收；同时引入 `tidb_gc_max_wait_time`，控制活跃事务阻塞 GC safepoint 的最长时间，超时后强制推进 safepoint(docs.pingcap.com garbage-collection-configuration;GitHub issue #32725)。其代价是历史版本与存储放大增加。

### 11.2.5 实践增强：In-Memory Engine(IME)

GC 解决的是过期版本的物理回收；在回收尚未发生时，读路径仍可能面临多版本放大。针对这一点，TiKV 8.5 引入独立 component `in_memory_engine`(对应 TiDB 文档中的 MVCC In-Memory Engine / Range Cache)，源码版权头标注 `Copyright 2023`，默认 `enable=false`(默认关闭)(tikv/tikv `components/in_memory_engine/src/` @ release-8.5)。其默认配置：`gc_run_interval=180s`、`mvcc_amplification_threshold=10`、`capacity` 取 block cache 的 10%(上限 5GB)(`components/in_memory_engine/src/config.rs`)。

官方文档将 IME 定位为加速「扫描大量 MVCC 历史版本、total_keys 远大于 processed_keys」的查询：它缓存最近写入的 MVCC 版本，有独立于 TiDB GC 的内存 GC 机制(IME 通过 `(next+prev)/processed_keys` 衡量 MVCC 放大)，命中时减少需要扫描的版本数，缺失必要历史版本时回退 RocksDB。这里不能把 IME 写成替代 RocksDB 的内存数据库——它是针对 MVCC 读放大的缓存/索引化路径，受容量、Region 缓存命中、内存 GC 和 fallback 影响。官方博客称在频繁更新/删除的订单类负载上，p99 SELECT 延迟从约 400ms 降到约 10ms("IME reduced p99 SELECT latency from 400 milliseconds to just 10 milliseconds");CPU 消耗按原文口径为「约降 2 核(8 核 32GB 配置下，≈25%)」("IME lowered CPU consumption by about 2 cores per TiKV node (on an 8-core, 32GB setup)")(pingcap.com/blog IME)。源码仅能确认「8.5 release 中 IME component 存在且默认关闭」；docs.pingcap.com 将该配置标注为 `new-in-v850`，本章据锁定版本(8.5.x)按「8.5 引入、默认关闭」书写。

--- 〔文献[1,4-7,11-12,15-16]〕

## 11.3 OceanBase 的实现

### 11.3.1 时间与版本核心：SCN 与读一致性映射

OceanBase 的时间与版本核心是 SCN。公开文档层面，OceanBase 4.x 支持强一致读、弱一致读、快照读与多副本读写分离；读一致性三档由枚举 `enum ObConsistencyLevel { FROZEN, WEAK, STRONG }` 定义(`deps/oblib/src/lib/ob_define.h` @ v4.2.5_CE，commit `e7c676806fda`)。SQL 层可通过 Global hint 指定：hint 解析字段为 `ObConsistencyLevel read_consistency_`，函数 `merge_read_consistency_hint(read_consistency, frozen_version)`(`src/sql/resolver/dml/ob_hint.h`)。在事务控制路径上，SQL 层把 STRONG 映射为当前读、把 WEAK/FROZEN 映射为有界过期读(bounded staleness read)，弱读路径取得弱读快照后再读。这说明 OceanBase 的读一致性不是简单的「读任意副本」，而是先确定读快照，再由路由和事务/存储层确保读取的版本满足该快照。

### 11.3.2 多版本编码：行级 trans node 链表(MemTable)

与 TiKV 把版本系在 key 后缀上不同，OceanBase 的 MVCC 不靠 LSM key 后缀时间戳，而是**行级多版本双向链表**。每个 key 对应一个 `ObMvccRow`(行体)，字段含 `ObMvccTransNode *list_head_`(多版本链表头)、`latest_compact_node_`、`total_trans_node_cnt_`、`max_trans_version_`、`ObRowLatch latch_`(行级自旋锁)、`ObMvccRowIndex *index_`；每个版本节点 `ObMvccTransNode` 是存放在 MemTable 上、用于 MVCC 的多版本数据节点，写入时只保存更新列，字段含 `prev_/next_` 指针、`trans_version_:SCN`、`scn_:SCN`、`tx_end_scn_`、`modify_count_`、checksum 等(oceanbase/oceanbase `src/storage/memtable/mvcc/ob_mvcc_row.h` @ v4.2.5_CE，commit `e7c676806fda`)。即：OB MemTable 的多版本是「一行挂一条 trans node 链表，新版本插表头」。

链表节点直接携带强/弱一致读屏障与事务状态 flag:`F_WEAK_CONSISTENT_READ_BARRIER`、`F_STRONG_CONSISTENT_READ_BARRIER`、`F_COMMITTED`、`F_ELR`(Early Lock Release)、`F_ABORTED`、`F_DELAYED_CLEANOUT` 等，`set_safe_read_barrier(is_weak_consistent_read)` 据强/弱一致读设置不同 barrier(`ob_mvcc_row.h`)。这证实 OB 在**行级版本节点**上就直接区分强一致读 / 弱一致读，并支持 ELR(提前放锁)与延迟清理。

**长版本链优化**:`INDEX_TRIGGER_COUNT = 500`，当查找插入位置前访问的节点数超过该阈值时，构建辅助索引 `ObMvccRowIndex`，缓解长事务 / 热点行导致的链表遍历开销(`ob_mvcc_row.h`)。

### 11.3.3 可见性判断依赖 tx_table

与 TiKV 把提交信息编码进 write CF 不同，OceanBase 的 trans node 本身**不直接存最终提交版本**，而是回查 `tx_table`。读路径据 `tx_table` 判断某个 trans node 对应的事务是否可见：事务状态 `enum TxDataState { RUNNING, COMMIT, ELR_COMMIT, ABORT, ... }`，`ObTxData` 由「state + commit_version + 回滚序列 undo_status」组成(`src/storage/tx/ob_tx_data_define.h`；模块 `src/storage/tx_table/`)。读时：若版本对应事务仍 `RUNNING` 或已 `ABORT` 则不可见；`COMMIT`/`ELR_COMMIT` 则据 `commit_version` 与快照版本比较决定可见性。`ObTxDataCheckData` 含 `state_`、`commit_version_`、`end_scn_`、`is_rollback` 供读路径回查(`src/storage/tx_table/ob_tx_table_define.h`)。

`tx_table` 在源码中拆成 tx data table 与 tx context table;`ObTxDataTable` 的 undo status 使用固定长度 `ObUndoAction` 数组并以单链表串接，回滚部分动作时动态分配 undo action。需要强调：这个事实只支撑「事务状态/undo 信息参与可见性和回滚判断」，并不等价于「所有历史行版本都保存在 undo 表里」——OceanBase 的历史行可见性由 MemTable 多版本、SSTable 多版本、事务状态和 SCN 快照共同决定。这种「版本节点 + 事务表分离」的设计，使得提交决议(尤其延迟清理 delayed cleanout)可以与写入解耦。

### 11.3.4 多版本编码：SSTable 隐藏版本列

前述链表与 tx_table 回查描述的是内存态；落盘后，OceanBase 的多版本不再是链表，而是**多版本 rowkey**：在原始 rowkey 之后追加两个隐藏列——`OB_HIDDEN_TRANS_VERSION_COLUMN_ID = 7`(trans_version)与 `OB_HIDDEN_SQL_SEQUENCE_COLUMN_ID = 8`(sql_sequence)(`deps/oblib/src/lib/ob_define.h` 常量；使用处见 `src/storage/ob_storage_schema.cpp` 等)。读 SSTable 时按 snapshot version 对 trans_version 过滤，挑出可见版本。

读 MemTable 时，OceanBase 会在单行读和范围读间选择不同路径：`ObMemtableSingleRowReader` 在范围读时构造 `ObMvccScanRange`，调用 `memtable_->get_mvcc_engine().scan(...)` 获得 row iterator，取到 `ObMvccValueIterator` 后再由 `ObReadRow::iterate_row` 聚合出当前读快照下可见的一行；若判断为单行 key 且非 delete-insert 特殊路径，可直接调用 `memtable_->get(...)`。`ObMultiVersionValueIterator` 用于迭代冻结 MemTable 的多版本与事务未提交 row，并带有 `merge_scn_` 相关接口。这与 TiKV「user key + 倒序 ts」殊途同归，但 OB 是把版本作为**显式隐藏列**编码进关系行，而非不透明 KV 后缀。本章用 v4.2.5(TP，行存)checkout 验证「行存 + 隐藏列」方案；4.3.x 列存(CS_ENCODING，自 V4.3.0 引入)下列存版本块的具体组织未在本章 checkout 内展开，凡涉及之处均存疑待考。

### 11.3.5 控制路径：row compaction 与 minor/major merge

与 TiDB 把回收摊进 compaction 不同，OceanBase 的历史版本清理绑定 merge 节奏，分内存与后台两层。OceanBase 官方文档说明其存储引擎基于 LSM-tree 架构，数据存于 MemTable 和 SSTable;MemTable 超过阈值后冻结并通过 minor compaction 生成 SSTable，major compaction 在低峰或 minor compaction 数量达到阈值时把基线 SSTable 与增量 SSTable 合并成一个 SSTable(en.oceanbase.com minor/major compaction)。多版本清理因此分两层：

- **MemTable 内行压缩(row compaction)**:`ObMvccRowCompactor::compact(snapshot_version, ...)` 按 snapshot version 把一行内多个 trans node 压实成一个 compact node，并在压缩时 cleanout 已决议事务节点(`src/storage/memtable/ob_row_compactor.{h,cpp}`)。
- **后台 merge(转储/合并)**:merge 类型枚举 `enum ObMergeType { MINOR_MERGE, HISTORY_MINOR_MERGE, META_MAJOR_MERGE, MINI_MERGE, MAJOR_MERGE, MEDIUM_MERGE, DDL_KV_MERGE, ... }`(`src/storage/compaction/ob_compaction_util.h` @ v4.2.5_CE)。MINI_MERGE 仅 flush memtable 成 mini SSTable;MINOR_MERGE 把若干 mini SSTable 合成更大的 SSTable;MAJOR_MERGE / MEDIUM_MERGE 是全局基线合并，选全局版本号(详见第 4 章)。多版本清理与读放大随 merge(尤其 major/medium merge)推进而下降。

`ObMergeType` 枚举在 4.3.5 分支已扩展(新增 `BACKFILL_TX_MERGE`、`MDS_MINI_MERGE`、`MDS_MINOR_MERGE`、`CONVERT_CO_MAJOR_MERGE`(注释「convert row store major into columnar store cg sstables」)、`INC_MAJOR_MERGE` 等列存/增量相关类型)，与 v4.2.5_CE 的枚举不完全一致。本章按锁定的 v4.2.5_CE commit 书写枚举(差异处见 oceanbase 4.3.5 分支 `src/storage/compaction/ob_compaction_util.h`)。

### 11.3.6 读路径与读一致性：是否走 Leader

回到读侧，读请求据一致性要求决定走哪类副本：

- **强一致读(STRONG，默认)**：必须路由到 Leader 副本读最新已提交版本。ODP 把强一致读与 DML 路由到 partition log stream 的 Leader 所在 OBServer(en.oceanbase.com read-write splitting)。
- **弱一致读(WEAK / stale read)**：可由 Follower 服务，允许读到稍旧但有界的版本，仍返回事务快照中的已提交数据、不返回未提交事务。SQL hint 文本为 `SELECT /*+ read_consistency(weak) */ ...`。ODP 侧 `obproxy_read_consistency=1` 启用弱读；租户级参数 `max_stale_time_for_weak_consistency` 默认 `"5s"`(源码 `DEF_TIME_WITH_CHECKER`，范围 `[5s,+∞)`、不小于 `weak_read_version_refresh_interval`)，副本落后超过该阈值即不可读、触发重试其他副本；集群级参数 `weak_read_version_refresh_interval` 默认 `"100ms"`(范围 `[50ms,)`)控制弱读版本刷新(`src/share/parameter/ob_parameter_seed.ipp` @ v4.2.5_CE)。monotonic weak read 进一步避免后续读回退到更旧版本。弱读的安全版本由 **weak read service** 维护：模块 `src/storage/tx/wrs/` 维护 LS / 租户 / 集群级弱读版本(weak read timestamp)。其推进算法在源码中可核为「逐级取最小」：server 级 `ObTenantWeakReadServerVersionMgr` 取本机所有有效 partition/LS 可读快照版本的最小值(`update_with_part_info` 中 `target_version > version` 即下调),cluster 级 `ObTenantWeakReadClusterVersionMgr::get_version` 取所有未被跳过(非超时落后)server 版本中 ≥ base_version 的最小值并落 inner table 的 `min_version/max_version`；最终弱读版本再由 `ObWeakReadUtil::generate_min_weak_read_version` 以 `GTS - max_stale_time` 设下界(常量 `DEFAULT_MAX_STALE_TIME_FOR_WEAK_CONSISTENCY = 5*1000*1000L` us)(`src/storage/tx/wrs/ob_tenant_weak_read_server_version_mgr.cpp`、`ob_tenant_weak_read_cluster_version_mgr.cpp`、`ob_weak_read_util.{h,cpp}`)。

源码仅确认到枚举 `ObConsistencyLevel` 与解析字段 `read_consistency_`，未逐字确认 hint 字符串；`/*+ read_consistency(weak) */` 文本由两个独立 OceanBase 官方文档页面交叉确认——弱一致性读页与读写分离最佳实践页(后者原文给出 `SELECT /*+read_consistency(weak)*/ * FROM t1` 并明确「强一致读/DML 路由 Leader、弱读优先转发 Follower」)，两页均给出 `max_stale_time_for_weak_consistency` 默认 5s，详见 §11.12。

--- 〔文献[2-3,8-10,13-14,17]〕

## 11.4 核心差异对比

在分别梳理两系统的实现后，表 11-1 沿多版本编码、可见性判定、读路由与版本清理等维度做横向对比。

**表 11-1　MVCC 与读路径:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB / TiKV | OceanBase | 影响 |
|---|---|---|---|
| 版本时间源 | TSO 生成 start_ts/read_ts/commit_ts;write CF key 用 commit_ts，default CF key 用 start_ts | SCN 作为事务提交、读快照、merge 的版本边界 | TiDB 的版本选择直接暴露在 KV key 编码中；OB 的版本判断内聚于事务和存储引擎 |
| 多版本编码(内存) | RocksDB write/default/lock 三 CF，user key + 倒序时间戳后缀 | 行级 `ObMvccRow` 挂 `ObMvccTransNode` 双向链表，新版本插表头 | OB 行级链表读写同一行更集中，但长链需建索引；TiKV 靠 RocksDB seek 天然有序 |
| 多版本编码(落盘) | write CF 存提交记录(含 start_ts)，short value≤255B 内联 | SSTable rowkey 追加隐藏列 trans_version(列 ID 7)/sql_sequence(列 ID 8) | TiKV 是不透明 KV 后缀；OB 是关系行内显式隐藏列 |
| 可见性判定 | 提交信息内联 write CF，seek 即得 commit_ts→start_ts | trans node 回查 tx_table 的 state/commit_version | OB 提交决议与写入解耦，支持 ELR / 延迟清理；TiKV 决议直接落 write CF |
| 锁处理 | 读先检查 lock CF / 内存悲观锁，冲突时 lock resolution | SQL/事务层定当前读或弱读快照，行锁/事务状态在 MVCC iterator 与 tx_table 中参与判断 | TiKV 锁冲突常显式暴露为 lock resolve;OB 更强调事务服务与行版本可见性协同 |
| 强一致读是否走 Leader | follower read 用 ReadIndex 仍需 Leader 一次往返；stale read(ts ≤ safe-ts)任意副本免往返 | 强一致读必须走 Leader；弱读取 bounded-staleness 快照可优先 Follower | 二者强读默认都依赖 Leader，弱/旧读才下放副本 |
| 历史版本清理 | GC safepoint(now − gc_life_time，默认 10min)+ Resolve Locks/Do GC + compaction filter 随 compaction 删 write CF 过期版本 | MemTable 内 row compaction + 后台 mini/minor/major merge | TiKV GC 是「时间窗 + compaction 顺带」；OB 清理绑定 merge 节奏与全局基线 |
| 长事务影响 | 卡 GC safepoint 推进，v6.1+ 可由 `tidb_gc_max_wait_time` 强推 | 长版本链触发 `INDEX_TRIGGER_COUNT=500` 建索引；merge 受活跃事务版本约束 | 二者长事务都阻碍版本回收，机制不同 |
| 内存加速 | In-Memory Engine(8.5，默认关闭)，缓存热点 region 近期多版本 | MemTable 即热数据，行压缩 + 行级索引缓解放大 | TiKV 把内存加速做成可选独立引擎；OB 内建于 MemTable |

---

## 11.5 正常路径图(快照读)

图 11-2 把两系统快照读 / 弱读的正常路径并置：左侧 TiDB 经 safe-ts 分岔到本地副本或 Leader/ReadIndex，再走三 CF；右侧 OceanBase 经一致性类型分岔到 Leader 或 Follower，再走 MemTable + SSTable 多版本合并。

![[f11_2.svg]]

**图 11-2　MVCC 与读路径正常读写/调度路径**

---

## 11.6 故障 / 异常路径图

正常路径之外，读路径的真正复杂度集中在异常分支。图 11-3 把两系统的故障分支并置：TiDB 侧主要是未决锁、Region epoch 变化、GC safepoint 越过 read_ts;OceanBase 侧主要是弱读副本落后、强读时 Leader 不可用、merge 落后、长事务卡回收。

![[f11_3.svg]]

**图 11-3　MVCC 与读路径故障/异常路径**

OceanBase 强读时 Leader 切换期间(Paxos 重选、LogReconfirm 进行中)读路由会短暂失败并由 ODP 重试，这一窗口的恢复指标边界受部署形态约束(见 §11.7)。

---

## 11.7 性能、可靠性、运维影响

**延迟**：两系统快照读的核心代价都是「多版本放大」，但放大物理形态不同。TiKV 中，若一行被频繁更新且 GC 未及时回收，write CF seek 后可能要跳过大量 Rollback/Lock/Delete 或不可见 Put 才命中可见版本；IME 即针对此场景设计(p99 从约 400ms 降约 10ms,benchmark 条件见原文)。OceanBase 中，读延迟更像「读时合并预算」问题：MemTable 长 trans node 链(>500 触发建索引)、minor SSTable 堆积与 major merge 落后，都会让一次读合并更多版本和块；major/medium merge 推进后读放大下降，但 merge 本身占用 IO/CPU，会引入读延迟毛刺(详见第 4 章 major merge 抖动)。

TiDB 的性能瓶颈常出现在三处：第一是最新强一致读的复制验证成本，尤其跨地域 ReadIndex；第二是 MVCC 版本链过长时 write CF 要跳过大量不可见版本；第三是 lock resolution 在高冲突或大量遗留锁时引入额外 RPC 和重试。OceanBase 则集中在 Leader 路由、事务快照获取与多层版本聚合：弱读可用 Follower 资源分摊 Leader 压力，但副本 replay 落后超过 `max_stale_time_for_weak_consistency` 时不能继续读该副本；强一致读默认走 Leader，因此热点 Leader 与副本延迟会同时压在延迟尾部。

**吞吐 / 扩展性**：强一致读两系统都依赖 Leader——TiKV follower read 用 ReadIndex 仍需 Leader 一次往返，OceanBase 强一致读必须走 Leader。真正能水平分摊读负载的是「放宽一致性」的路径：TiKV 的 stale read(任意副本免往返)与 OceanBase 的弱一致读(Follower 服务)。这意味着读扩展性的上限由业务能否接受有界陈旧度决定，而非副本数。

**可用性 / 故障恢复**：读路径的可用性绑定 Leader 可用性。Leader 切换期间(TiKV Raft 重选 / OceanBase PALF(Paxos-backed Append-only Log File system)LogReconfirm，详见第 3 章)，强一致读会短暂失败并重试。弱读 / stale read 在此期间反而更稳——它们不依赖最新 Leader。需注意 RPO=0/RTO<8s 仅适用于 OceanBase V4.x LS/Paxos 多副本、少数派故障或多数派完整场景，不绑定 RootService、不外推单副本 shared-storage / 跨云(详见第 3 章)。

**运维复杂度**:TiDB 的 GC 是「时间窗 + compaction 顺带」的弱耦合模型，运维主要盯 `tikv_gc_safe_point` 是否推进、是否有长事务卡住；OceanBase 的清理绑定 merge 节奏，运维需关注 major merge 进度、转储是否堆积、弱读版本是否落后。

表 11-2 从「读路径性能瓶颈来源」维度对两系统做二次对比，可见瓶颈位置不同决定了优化手段不同：

**表 11-2　性能、可靠性、运维影响**

| 瓶颈/影响维度 | TiDB / TiKV | OceanBase | 可验证信号 |
|---|---|---|---|
| 读放大主因 | write CF 同一 user key 多版本堆积，seek 跳过过期版本 | MemTable trans node 长链 + 多层 SSTable 未合并历史版本 | total_keys/processed_keys、慢查询耗时、merge backlog |
| 放大缓解手段 | GC compaction filter 删除 + 可选 IME 缓存近期版本 | row compaction + major/medium merge + 行级索引 | IME Region cache hit/miss、major merge 进度 |
| 强读延迟下界 | Leader 上一次 MVCC 定位(seek)+ 可能回 default CF + 可能 ReadIndex | Leader 上遍历链表 + 回查 tx_table + 读 SSTable 合并 | 点读 P99、ReadIndex / lock resolve 次数 |
| 副本读条件 | stale read 要 ts ≤ safe-ts;follower 最新读仍有 ReadIndex 成本 | 弱读受最大落后时间与 monotonic read 约束，落后副本被排除 | 注入副本延迟，验证是否回退 Leader 或换副本 |
| 抖动来源 | compaction 触发不规律 → 过期版本静默滞留 | major/medium merge 占 IO/CPU → 读延迟毛刺 | TiKV 抖动隐性、OB 抖动集中可观测 |
| 关键运维抓手 | `tikv_gc_safe_point` 推进、长事务、compaction 进度 | major merge 进度、弱读版本落后量、转储堆积 | 监控对象由清理机制决定 |

---

## 11.8 反例与代价

前面的机制各有取舍，这里集中陈述它们的代价面——每一项设计的好处都对应一个需要付出的成本。

- **TiKV 倒序时间戳后缀的代价**：它把「同一行的所有版本」物理相邻排列，对点查友好，但对「跨行范围扫描 + 高更新」场景，扫描指针会在大量历史版本间跳跃(MVCC 放大)，这正是 IME 针对的场景。代价是 IME 需要额外内存(默认占 block cache 10%)且默认关闭，需人工评估开启；且文档明确缺少所需历史版本时回退 RocksDB——若瓶颈是网络、Leader ReadIndex、锁冲突或 block cache miss,IME 只能覆盖其中一部分，不是所有读延迟的万能解。
- **OceanBase 行级链表的代价**：热点行的 trans node 链会很长，虽有 `INDEX_TRIGGER_COUNT=500` 兜底建索引，但链表遍历 + 回查 tx_table 在高并发热点行上仍是开销；且 trans node 与 tx_table 分离意味着读时多一次事务表回查(可借助 tx_data_cache 缓解)。
- **强一致读不可水平扩展**：两系统强读默认都压在 Leader 上。若业务把所有读都设为强一致，副本再多也无助于读吞吐——这是「一致性 ↔ 可扩展」的根本取舍，不是实现缺陷。
- **弱读 / stale read 的有界陈旧度代价**：换取副本可读、低延迟，代价是可能读到几秒前的数据(OceanBase 默认 5s,TiKV 取决于 resolved-ts 推进)。bounded staleness 只约束最大落后时间、不保证读到最新，monotonic weak read 解决「不倒退」也不等于「最新」。对库存扣减、唯一性判断、支付状态确认、读己之写类业务，应使用当前读或强一致事务读。
- **GC lifetime / merge 都不是免费旋钮**：调大 `tidb_gc_life_time` 可保护长事务或历史读，但会保留更多旧版本，增加 RocksDB 空间、compaction 与 MVCC seek 成本；OceanBase 的 minor/major merge 决定历史版本与多层 SSTable 的收敛速度，merge 落后转化为读时合并成本，merge 过猛又消耗 IO/CPU 干扰前台读写。TiKV 把清理摊进 compaction，平时无抖动，但若 compaction 长期不发生，过期版本会静默堆积、读放大上升而无明显告警；OB 把清理集中到 merge，代价是 merge 期间资源抖动但更集中可观测。
- **「副本读 = 任意 follower 都能读」是危险简化**:TiDB 要 safe-ts 或 ReadIndex 证明，OceanBase 要弱读快照、最大落后时间和副本可读性判断，二者都不是「副本存在即可读」。

---

## 11.9 测试开发视角的验证点

**可测功能场景**:

- 快照隔离正确性：并发事务下，快照读是否只看见 start_ts 之前已提交版本；同一 key 多次写入后用不同 read_ts 或 `AS OF TIMESTAMP` 类 stale read 验证可见版本，并检查 Delete、Rollback、Lock 版本不被错误返回。
- 当前读 / 悲观锁读：一个事务持锁，另一个事务读 / `SELECT FOR UPDATE` / stale read，验证等待、冲突、绕过或 lock resolution 行为；当前读是否读到最新版本并加锁。
- OceanBase 强 / 弱读：默认 `SELECT`、`/*+ read_consistency(weak) */`、session 弱一致，在 Follower 延迟下验证读到的是已提交快照且不读未提交；落后超阈值副本是否正确触发重试。
- follower read 一致性：`tidb_replica_read=follower` 下读结果是否与 leader 读一致(ReadIndex 正确性)。
- 长事务与清理：TiDB 让长事务 startTS 早于 GC safe point 推进边界，OceanBase 让长快照跨越 freeze/merge，验证历史版本不会被提前清理。

**可注入失效模式**:

- Leader 切换中读：在强一致读进行时杀 Leader，观察重试与延迟尖刺。
- Region epoch 变化 / primary lock 延迟：制造 epoch not match 与 primary lock 所在 key 延迟，观察 Region cache 刷新与 lock resolution。
- 长事务卡 GC：开一个长期不提交事务，观察 `tikv_gc_safe_point` 是否停滞、`tidb_gc_max_wait_time` 是否在阈值后强推。
- merge 抖动注入：OceanBase 强制触发 major merge，观察读延迟 p99 毛刺；制造 minor freeze 堆积、major merge 暂停。
- MVCC 放大注入：对同一行高频更新且关闭 GC，观察 scan 延迟随版本数上升；开启 IME 后对比。
- 弱读副本落后：注入 follower replay 落后超过最大 staleness，验证副本是否被排除或重试。

**关键压测 / 观测指标**：关键看「读放大」与「清理滞后」，而非只看 QPS。

- TiKV:`total_keys`/`processed_keys` 比值(MVCC 放大)、`tikv_gc_compaction_filtered`(compaction filter 过滤的版本数)、GC speed、lock resolve 次数、coprocessor scan 延迟；官方 Grafana 文档已列出 IME 的 Region Cache Hit/Hit Rate/Miss Reason、Region GC/Load/Eviction Duration、Seek duration 等面板项。
- TiDB GC 状态：`mysql.tidb` 表 `tikv_gc_safe_point`、`tikv_gc_last_run_time`、`tikv_gc_leader_uuid`。
- OceanBase：弱读命中 Follower 比例、follower replay lag、weak read 版本落后量、major merge 进度、minor/major compaction 耗时、tx_data / tx_table 命中相关计数、慢查询中存储读耗时。

诊断起点：TiDB 可用官方文档中的 GC 配置、Stale Read/safe-ts troubleshooting 与 TiKV dashboard 面板，`tikv-ctl` 可扫描 default/write/lock CF 或做 MVCC 恢复类检查(生产使用需谨慎并按官方手册执行);OceanBase 可用 `/*+ read_consistency(weak) */` hint 与读写分离最佳实践验证 SQL 是否走弱读。上述具体 Prometheus metric、内部视图名、ODP 路由诊断字段未逐一在本章 checkout 确认者一律标「需进一步查证」，本章不编造。

---

## 11.10 容易误解点

1. **「follower read = 不碰 Leader」是错的**。TiDB follower read 仍需 ReadIndex 向 Leader 取 commit index 以保证线性一致，只省了 Leader 的数据扫描 CPU，不省网络往返。真正免 Leader 往返的是 **stale read**(读旧版本、任意副本本地可判，条件是 `ts ≤ safe-ts`)。同理「副本读 = 任意 follower 都能读」也是危险简化(见 §11.8)。

2. **「MVCC 读不加锁，所以不会遇到锁」是错的**。TiDB 的 SI 点读会先检查 lock CF，因为未提交写可能影响 read_ts 下的可见性；OceanBase 的 MVCC iterator 也要处理未提交 row、事务状态和行锁冲突。

3. **「write CF 是索引」是错的**。TiKV 的 write CF 不是二级索引，而是「提交版本记录表」：它存某 user key 在某 commit_ts 的提交事件，value 内含指向 default CF 数据的 start_ts(短值还会内联)，并混存 Rollback 标记(详见第 4 章)。

4. **「OceanBase 没有 undo / 多版本靠覆盖」是错的**。OceanBase 的历史版本通过 MemTable 行级 trans node 链 + SSTable 隐藏 trans_version 列实现，可见性靠回查 tx_table 的 commit_version，绝非就地覆盖；tx_table 中的 undo status 参与回滚判断，但不等价于「所有历史行版本都存在 undo 表里」。

5. **「Stale/weak read 就是不一致读」是不准确的**。TiDB stale read 与 OceanBase weak read 都读历史或有界落后的**已提交快照**，目标是牺牲新鲜度，不是允许脏读。

6. **「GC safepoint 一推进版本立刻消失」是错的**。safepoint 推进只是「允许回收」，TiKV 实际物理删除发生在 write CF 的 compaction filter 触发时；若长期无 compaction，过期版本会静默滞留，scan 仍会扫到它们。同理「历史版本越多只影响存储不影响读」也是错的——版本堆积会直接转化为 write CF seek 跳过更多版本、OB 读时合并更多 SSTable 层。

---

## 11.11 本章结论

1. TiKV 的 MVCC = Percolator 三 CF(default/lock/write)+ user key 倒序时间戳后缀，短值(≤255B)内联 write CF；快照读靠一次 write CF seek 定位 commit_ts≤ts 的最新提交记录再回查数据，提交决议直接落 write CF；读可见性由 read_ts、write CF 的 commit_ts 和 default CF 的 start_ts 三者共同决定，lock CF / 内存悲观锁负责并发写保护。

2. OceanBase 的 MVCC = MemTable 行级 `ObMvccRow`/`ObMvccTransNode` 双向链表(新版本插表头，>500 建索引)+ SSTable 隐藏列 trans_version(列 ID 7)/sql_sequence(列 ID 8)；可见性不内联，而是回查 tx_table 的 state/commit_version，据此支持 ELR 与延迟清理；读一致性在 SQL/事务层被映射为当前读与有界过期读。

3. 强一致读两系统默认都依赖 Leader(TiKV follower read 仍需 ReadIndex 往返、OceanBase 强读必走 Leader)，只有放宽一致性的路径才下放副本：TiKV stale read 的安全条件不是「副本存在即可读」而是 `ts ≤ safe-ts`(且 safe-ts 受 resolved-ts 约束),OceanBase 弱一致读经 `/*+ read_consistency(weak) */` 走 Follower、由 weak read service 提供安全版本(默认陈旧上限 5s)且仍读已提交快照。

4. 历史版本清理两条不同哲学：TiDB 用 GC safepoint(now−gc_life_time，默认 10min)+ Resolve Locks/Do GC + write CF compaction filter「随 compaction 顺带删除」，清理前必须先 resolve safe point 前的锁；OceanBase 用 MemTable 行压缩 + mini/minor/major merge，清理节奏绑定全局基线合并。

5. 长事务是两系统共同的版本回收敌人：TiDB 中长事务卡 GC safepoint 推进(v6.1+ 可由 `tidb_gc_max_wait_time` 强推),OceanBase 中长事务拉长 trans node 链并约束 merge 可回收的版本；二者都用历史读安全换取版本保留成本上升。

6. 读延迟与版本堆积/merge 进度强相关：MVCC 放大(扫描版本数 ≫ 处理版本数)是核心痛点，TiKV 主要表现为 CF 级 MVCC seek / lock resolution、以可选的 In-Memory Engine(8.5，默认关闭)缓解，OceanBase 主要表现为多层 MemTable/SSTable 读时合并、以 major merge 推进 + 行级索引缓解。

7. 二者多版本编码殊途同归(都把版本系到 key/行上)，但落点不同：TiKV 是不透明 KV 后缀，OceanBase 是关系行内显式隐藏列 + 独立事务表——据此推测，前者更通用，后者则更贴合关系内核的协同。

---

## 11.12 参考文献

[1] Percolator: Large-scale Incremental Processing Using Distributed Transactions and Notifications. 论文，OSDI 2010[EB/OL]. https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Peng.pdf.
 （支撑:§11.2.1 中 TiKV 三 CF(data/lock/write metadata)、primary lock 与快照隔离的协议理论来源。）
[2] OceanBase: A 707 Million tpmC Distributed Relational Database System. 论文，PVLDB 2022[EB/OL]. https://vldb.org/pvldb/vol15/p3385-xu.pdf.
 （支撑:§11.3 中 OceanBase 作为分布式关系数据库使用 MVCC、LSM 多版本、GTS/SCN 驱动快照读的论文级背景；不把 benchmark 当本章性能事实。）
[3] PALF: Replicated Write-Ahead Logging for Distributed Databases. 论文，PVLDB 2024[EB/OL]. https://www.vldb.org/pvldb/vol17/p3745-xu.pdf.
 （支撑:§11.3、§11.6 中 OceanBase 事务日志 / LS / SCN 背景，以及读一致性依赖日志推进(weak read 安全版本与 PALF 推进的关系)。）
[4] TiKV | Percolator(Deep Dive). 官方文档[EB/OL]. https://tikv.org/deep-dive/distributed-transaction/percolator/.
 （支撑:§11.2.1–11.2.2 中三 CF 的 key 编码 (key,start_ts)/(key,commit_ts) 与快照读「按 commit_ts 反查 start_ts」的读路径解读(原文逐字含 "Ge）
[5] TiKV MVCC In-Memory Engine. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tikv-in-memory-engine/.
 （支撑:§11.2.5、§11.7、§11.8 中 IME 的 8.5 引入、默认关闭、capacity/gc-run-interval/mvcc-amplification-threshold 默认值、适用场景(total_k）
[6] Garbage Collection Overview / Configuration. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/garbage-collection-overview/.
 （支撑:§11.2.4、§11.7 中 GC 三阶段(Resolve Locks / Delete Ranges / Do GC)、GC mode(distributed)、compaction filter 作用于 write）
[7] Follower Read / Stale Read / Troubleshoot Stale Read. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/follower-read/.
 （支撑:§11.2.3、§11.4 中 tidb_replica_read 取值、ReadIndex 一致性保证、stale read ts ≤ safe-ts 本地读条件与 safe-ts 受 resolved-ts）
[8] OceanBase 弱一致性读 Weak Consistency Read. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001105870.
 （支撑:§11.3.6、§11.4 中 /*+ read_consistency(weak) */ hint、max_stale_time_for_weak_consistency(默认 5s)、bounded stal）
[9] Best practices for read-write splitting in OceanBase. 官方文档，弱读 hint 第二独立来源[EB/OL]. https://en.oceanbase.com/docs/common-best-practices-10000000001714402.
 （支撑:与上一条交叉确认 §11.3.6 中 SELECT /*+read_consistency(weak)*/ hint、max_stale_time_for_weak_consistency(默认 5s)、ODP）
[10] Minor compaction and major compaction. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001104016.
 （支撑:§11.3.5、§11.7 中 MemTable/SSTable、minor compaction、major compaction 与读时合并代价。）
[11] tikv/tikv 仓库(release-8.5 @ commit 1f8a140b6d46)— components/txn_types/、components/engine_traits/src/cf_defs.rs、src/storage/mvcc/、src/storage/txn/actions/gc.rs、src/server/gc_worker/、components/in_memory_engine/. 源码[EB/OL]. https://github.com/tikv/tikv/tree/release-8.5.
 （支撑:§11.2 中 CF 常量、WriteType/LockType、append_ts 倒序编码、SHORT_VALUE_MAX_LEN=255、PointGetter/MvccReader/SnapshotReade）
[12] tidb/tidb 仓库(release-8.5 @ commit 67b4876bd57b)— pkg/store/gcworker/gc_worker.go. 源码[EB/OL]. https://github.com/pingcap/tidb/tree/release-8.5.
 （支撑:§11.2.4 中 GC worker 默认参数(gcDefaultLifeTime=10min、distributed mode)、safe_point 计算与 mysql.tidb 系统表 key。确认到）
[13] oceanbase/oceanbase 仓库(v4.2.5_CE @ commit e7c676806fda)— src/storage/memtable/mvcc/ob_mvcc_row.h、src/storage/memtable/、src/storage/tx_table/、src/storage/tx/ob_tx_data_define.h、src/storage/compaction/ob_compaction_util.h、src/sql/resolver/dml/ob_hint.h、deps/oblib/src/lib/ob_define.h、src/storage/tx/wrs/、src/share/parameter/ob_parameter_seed.ipp. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE.
 （支撑:§11.3 中 ObMvccRow/ObMvccTransNode 链表与读屏障 flag、INDEX_TRIGGER_COUNT=500、TxDataState 枚举、隐藏列 ID 7/8、ObMemtableSing）
[14] oceanbase/oceanbase 仓库(4.3.5 分支 @ commit b28b9bb12f3b)— src/storage/compaction/ob_compaction_util.h、deps/oblib/src/lib/ob_define.h、src/storage/blocksstable/cs_encoding/. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/tree/4.3.5.
 （支撑:反查证 §11.3.4、§11.3.5、§11.13 中版本间差异：4.3.5 仍以隐藏列 OB_HIDDEN_TRANS_VERSION_COLUMN_ID=7/OB_HIDDEN_SQL_SEQUENCE_CO）
[15] Query Performance Unleashed: TiDB's In-Memory Engine (IME). 官方博客[EB/OL]. https://www.pingcap.com/blog/accelerating-query-performance-tidb-in-memory-engine/.
 （支撑:§11.2.5 中 IME 解决 MVCC 放大、p99 约 400ms→10ms、CPU「约降 2 核(8 核 32GB 配置下，≈25%)」的 benchmark 描述(条件见原文，不可外推)。）
[16] MVCC in TiKV / How TiKV reads and writes. 官方博客[EB/OL]. https://pingcap.medium.com/mvcc-in-tikv-f0aa318c564a.
 （支撑:§11.2.1 中 write CF 存版本信息、default CF 存真实数据的三 CF 划分与读写工程背景。注：MVCC in TiKV 一文读路径明言 "we will not elaborate on it"）
[17] An Interpretation of the Source Code of OceanBase (6): Storage Engine. 第三方解读，阿里云博客[EB/OL]. https://www.alibabacloud.com/blog/an-interpretation-of-the-source-code-of-oceanbase-6-detailed-explanation-of-storage-engine_599324.
 （支撑:辅助佐证 §11.3.2/11.3.5 中 MemTable 行操作链表与 compaction 合并历史版本；第三方解读，存在版本与措辞局限，关键事实以源码与官方文档为准。 ---）
## 11.13 信息可信度自评

本章最强证据来自官方文档与锁定源码。**源码级信息**(TiKV CF 常量、WriteType/LockType、倒序时间戳编码、SHORT_VALUE_MAX_LEN=255、PointGetter/MvccReader/SnapshotReader 读分支、`put_lock/put_value/put_write`、GC/IME 默认值；OceanBase ObMvccRow/TransNode、TxDataState、隐藏列 ID 7/8、ObMemtableSingleRowReader/ObMultiVersionValueIterator、ObMergeType、ObConsistencyLevel/read_consistency_、weak read service 模块)均 commit-pinned 至 tikv release-8.5@`1f8a140b6d46`、tidb release-8.5@`67b4876bd57b` 与 oceanbase v4.2.5_CE@`e7c676806fda`，确认到文件级，个别仅到函数签名/模块级处已显式标注。**官方文档级**(三 CF 语义、`(write,commit_ts)→start_ts` 读路径、GC 三阶段、compaction filter、follower/stale read 与 safe-ts/resolved-ts、`tidb_replica_read`、IME 版本归属与适用场景、弱读 hint 与 `max_stale_time_for_weak_consistency`、minor/major compaction)来自 docs.pingcap.com、tikv.org、en.oceanbase.com，弱读 hint 由两个独立 en.oceanbase.com 官方页交叉确认。**论文级**(Percolator 协议理论、OceanBase PVLDB 2022、PALF PVLDB 2024)用于理论背景，不替代当前版本实现事实。**官方博客**(IME 性能、MVCC in TiKV、How TiKV reads/writes)的 benchmark 数据明确标注条件、不可外推生产。**第三方解读**仅 1 条(阿里云 OceanBase 源码解读)，占比远低于 30%，且仅作佐证。

**反查证冲突记录**:(1)`ObMergeType` 枚举在 OceanBase develop 分支已扩展到约 15 个变体(新增列存/增量 major 相关类型)，与锁定的 v4.2.5_CE 不完全一致——本章按 v4.2.5_CE commit 书写并标注差异。(2)OceanBase 弱读 hint 文本 `/*+ read_consistency(weak) */` 源码仅确认到枚举 `ObConsistencyLevel` 与解析字段 `read_consistency_`，未逐字确认字符串，故由两个独立官方文档页面交叉确认。(3)TiDB IME 的「GA / 实验」状态属文档层，源码仅能确认「8.5 release 中 component 存在且默认 `enable=false`」，文档与源码无冲突。(4)MVCC in TiKV(Medium)一文读路径明言 "we will not elaborate on it"，其支撑收窄为三 CF 划分，「按 commit_ts 反查 start_ts」读路径解读改挂 tikv.org Percolator Deep Dive;IME CPU 用原文「约 2 核(8 核 32GB 配置下，≈25%)」口径。(5)OceanBase 弱读文档页含 V4.x 小节，本章只引用其中 V4.x 稳定语义，4.2.5 细节以源码限定版本边界。

**需进一步查证项**:4.3.x 列存版本块组织(本章用 4.2.5 行存 checkout 验证;已确认 4.3.5 仍以隐藏列 ID 7/8 承载多版本、列存仅在 major 经 `CONVERT_CO_MAJOR_MERGE` 由行存转出，但列存微块内多版本的具体编码未在本章展开);未核实的 Prometheus metric、OceanBase 内部视图(如 `__all_virtual_weak_read_stat` 已在 4.0 废弃)、ODP 路由诊断字段与未展开函数名一律标「需进一步查证」，正文不编造。注:本轮已将 TiKV `GcManager` poll safe_point 生产默认间隔(`POLL_SAFE_POINT_INTERVAL_SECS=10s`)、OceanBase weak read service 逐级取最小的版本推进算法两项由源码核实并升级入正文。涉及 RPO=0/RTO<8s 处已按 V4.x LS/Paxos 多副本、少数派故障前提书写，不外推单副本/shared-storage/跨云。未发现与版本基准必须改写的其它冲突。

---


# 第 12 章 SQL 优化器与执行引擎

## 12.1 本章核心问题

一条 SQL 从文本到结果，中间隔着两道工程鸿沟：怎么把声明式语义编译成最优的物理算子树(优化器)，以及怎么把这棵树切片、调度到多个节点上并行跑完(执行引擎)。在单机数据库里，优化器错误通常只表现为多扫一张表或选错一个索引，代价局限在本地 CPU 与内存；到了分布式数据库，这两件事被「数据分布」彻底重写——同一条 `JOIN`，数据落在不同 Region / Tablet 上，优化器必须决定让数据动还是让计算动，执行引擎必须决定 broadcast、shuffle 还是 partition-wise。一次错误的基数估计、错误的 Join 顺序或错误的分发策略，会被放大成跨 Region / Tablet 访问、shuffle、broadcast、远程 RPC 与并行线程拥塞：本应在存储层就地过滤掉的 99% 数据，可能原封不动地跨节点搬到计算层，把一个毫秒级查询拖成分钟级。

本章回答四个层层递进的问题：

1. TiDB 与 OceanBase 各自如何把逻辑计划优化成物理计划(RBO 规则链 + CBO 代价模型 + 统计信息)?
2. 可下推的算子(谓词、聚合、TopN、Limit)如何被推到离数据最近的地方，以减少跨层 / 跨节点传输?
3. 分布式 Join 的三种分发策略(broadcast / shuffle / partition-wise)在两个系统里如何表达与选择?
4. 当优化器估错时，故障路径长什么样，如何观测、如何兜底?

为什么这是分布式数据库底层的关键?因为在分布式形态下，优化器的每一个决策都同时是一个数据搬运决策。单机优化器选 Hash Join 还是 Merge Join，影响的只是本地资源；分布式优化器选 broadcast 还是 shuffle，直接决定了多少字节要穿过网络。一张维度表究竟「广播到每个节点」还是「让事实表按键重分布过来」，在数据量估对时差异巨大，估错时则代价高昂。更进一步，优化器还要回答一个单机系统不存在的问题：这条计划应该整体下推到某个节点本地执行(零网络)，还是切成多个并行片段跨节点协同?这正是 OceanBase「local / remote / distributed 三态」与 TiDB「root / cop / mpp 三层 task」要解决的同一类问题的两种工程答案。优化器之上还叠着执行引擎：计划选定后，谁来切片、谁来调度并行 worker、数据如何在算子间流动(pull 还是 push、batch 还是 streaming)，决定了同一份计划能否真正跑出理论吞吐。

版本边界先行声明。TiDB 以 8.5.x LTS 为主、对照 7.5.x；OceanBase 以 4.2.5_CE(TP)、4.3.5(AP)、4.4.x(融合)为边界。本章聚焦优化器与执行引擎本身：索引对计划的影响(详见第 15 章)、HTAP 列存与行存的协同(详见第 16 章)、统计信息所依赖的 TSO / 快照(详见第 11、13 章)只做必要交叉引用，不展开。

## 12.2 TiDB 的实现

带着上述四个问题，先看 TiDB 的工程答案。TiDB 的优化器位于无状态的 TiDB Server 进程内(详见第 6 章)，一条 SQL 的路径可分成四段：解析后得到 AST；planner 从 AST 构造初始逻辑计划；规则优化与 CBO 基于统计信息、cost model 和物理属性选择物理计划；executor 按 root / cop / mpp task 执行并汇总结果。源码集中在 `pkg/planner/`(逻辑上含 plan building、logical optimization、physical optimization 与 tidy up)，执行器在 `pkg/executor/`。

**逻辑优化(RBO)。** 入口是 `logicalOptimize`，按一个全局规则链 `optRuleList` 顺序触发各规则。该链在 `pkg/planner/core/optimizer.go` 中以 `var optRuleList = []base.LogicalOptRule{...}` 定义(release-8.5 @ `67b4876bd57b`，L78–L105 区间)，逐条核对到真实标识符：`GcSubstituter`、`ColumnPruner`(列裁剪，链首链尾各一次)、`DecorrelateSolver`(子查询去关联)、`AggregationEliminator`、`MaxMinEliminator`、`ConstantPropagationSolver`、`ConvertOuterToInnerJoin`、`PPDSolver`(谓词下推，Predicate Push Down)、`OuterJoinEliminator`、`PartitionProcessor`(分区裁剪)、`AggregationPushDownSolver`(聚合下推)、`PushDownTopNOptimizer`(TopN 下推)、`JoinReOrderSolver`(Join 重排)等。每条规则有独立文件，如 `rule_predicate_push_down.go`、`rule_aggregation_push_down.go`、`rule_topn_push_down.go`、`rule_join_reorder.go`(内部又分 `_dp.go` 动态规划与 `_greedy.go` 贪心两套 Join 枚举)。这些规则不看代价，只做等价变换：把过滤条件尽量下沉，把无用列剪掉，把能消除的外连接转内连接，使后续物理枚举的输入更小。

**物理优化(CBO)。** 入口是 `physicalOptimize`(`optimizer.go` L1049–L1090)。它分三步：先 `RecursiveDeriveStats` 自底向上推导每个算子的统计信息(行数、NDV、直方图)，再 `preparePossibleProperties` 为 Join / Agg 准备候选物理属性(如有序性)，最后以 `property.PhysicalProperty{TaskTp: RootTaskType, ExpectedCnt: MaxFloat64}` 为根需求调用 `logic.FindBestTask(prop, planCounter, opt)`。`FindBestTask` 是一个自顶向下、带 memoization 的 System-R / Volcano 风格搜索：`exhaustPhysicalPlans` 枚举满足所需物理属性的候选算子，`enumeratePhysicalPlans4Task` 用动态规划挑最小代价组合，`attach2Task` 把子任务拼到父算子上，代价综合 CPU / Memory / Network / IO 各维度并乘以可调的 `tidb_opt_*_factor` 因子。物理 Join 算子有 `PhysicalHashJoin`、`PhysicalMergeJoin`、`PhysicalIndexJoin` 及其 `IndexHashJoin` / `IndexMergeJoin` 变体(`physical_plans.go` L81–L95)。物理优化为逻辑节点选择具体实现的同时，会把要求的顺序、分布、行数上界向子节点传递——这解释了为什么同一 SQL 在添加索引、更新统计信息、增加 TiFlash replica 后可能出现完全不同的树形计划：变化不只发生在最后一个 scan 算子，而是沿着 Join 顺序、下推边界与 root 汇总方式向上传播。

release-8.5 中确实另存在一套独立的 Cascades 风格 memo 框架(`pkg/planner/cascades/`、`pkg/planner/memo/`)。TiDB Development Guide 明确把 `cascades` 描述为 next generation Cascades model planner，under development and disabled by default；8.5 默认物理优化主路径是 `physicalOptimize`→`FindBestTask`，本章把主线写成 core / System-R / Volcano 风格 CBO，而不把 Cascades 当默认路径书写。

**任务类型与下推。** TiDB EXPLAIN 把算子归到四类 task：`root`(TiDB 本地执行)、`cop[tikv]`(TiKV Coprocessor)、`cop[tiflash]` / `batchCop`(TiFlash Coprocessor / batch coprocessor)、`mpp[tiflash]`(TiFlash MPP)。`TableReader`、`IndexReader`、`IndexLookUp` 通常在 root 汇总 Coprocessor 返回的数据；可下推的算子(Selection、部分 Aggregation、TopN、Limit、TableScan / IndexScan)在满足表达式与算子支持条件时被编译成 DAG，经 Coprocessor 接口下推到 TiKV / TiFlash 就地执行，减少 root 层搬运——执行器侧由 `pkg/executor/coprocessor.go` 的 `NewCoprocessorDAGHandler` / `HandleRequest` / `buildDAGExecutor` 构建 DAG 执行器。以 `order by a limit 10` 为例，TopN 被下推后每个 cop task 只回传 10 行，而非全表。这正是 TiDB SQL 层与存储层的分工：TiDB Server 负责全局计划、root 算子与最终结果；TiKV Coprocessor 偏向 Region 局部的扫描、过滤、聚合；TiFlash MPP 承担面向列式副本的分布式分析执行。

**统计信息是 CBO 的硬输入。** TiDB 用统计信息估算每个 plan step 的 row count，并对索引访问、Join 顺序等候选计划计算 cost，最终选整体 cost 最低者。若 EXPLAIN 出现 `stats:pseudo`，意味着估算可能不准；大批量导入或更新后应运行 `ANALYZE TABLE` 刷新统计信息。这里的风险不是「数字不漂亮」，而是错误估算会改变是否选 Index Join、Hash Join、全表扫或 TiFlash MPP，进而改变网络路径。其中 Index Join 是最容易出现「看似局部、实则分布式放大」的算子：它的理想场景是外表行数较小、内表 join key 有可用索引、内表点查或 range scan 能快速完成；一旦外表估算偏小或 key 分布高度离散，内表 worker 会构造大量 key range 并向多个 Region 发起读取，root 层等待 fetch / build / probe 的成本显著上升。测试时不能只问「用了哪个 Join」，还要问 build / probe 哪侧、数据在哪层被过滤、是否回表、是否跨 TiKV / TiFlash 节点重分布。

**MPP 路径。** 当查询命中 TiFlash 列存副本且适合大规模并行时，优化器走 MPP。计划中出现 `ExchangeSender` 与 `ExchangeReceiver` 即表示 MPP 模式生效；物理计划按二者切成 Fragment(`pkg/planner/core/fragment.go`，`type Fragment struct{ ExchangeReceivers []...; ExchangeSender ... }`，`mppTaskGenerator.generateMPPTasks`)，数据分发方式由 `tipb.ExchangeType` 决定：`PassThrough`(单上游，回传)、`Broadcast`(广播，用于 Broadcast Hash Join)、`Hash`(按哈希重分布，用于 Shuffled Hash Join / Hash Aggregation)。执行器侧由 `pkg/executor/mpp_gather.go` 的 `MPPGather` 经 `mpp.NewExecutorWithRetry` 驱动。是否走 broadcast 由表大小与 `tidb_broadcast_join_threshold_size` 决定，小表广播、否则 shuffle。需注意，设置 `tidb_enforce_mpp=1` 会让 TiDB 忽略 cost 倾向于 MPP，但仍可能因缺少 TiFlash replica、replica 未同步或算子 / 函数不受支持而无法选择 MPP——这种「强制但仍可能被阻断」的边界很容易被误解。

**DXF 辨析。** TiDB 的 DXF(Distributed eXecution Framework)于 v7.1.0 引入(据 DXF 官方文档页)，v7.5.0 GA(据 7.5.0 release notes，该 GA 字样不在 DXF 文档页而在 release notes)，用于 `ADD INDEX`、`IMPORT INTO` 等后台分布式任务的统一调度与执行，文档还把 TTL、ANALYZE、BR 等大数据量后台场景归为同类，并说明该功能不适用于 TiDB Cloud Starter / Essential；系统变量 `tidb_enable_dist_task` 在 v7.1–v8.0 默认 OFF、v8.1.0 起默认 ON(据 DXF 文档页)。源码在 `pkg/disttask/framework/`(`scheduler/`、`taskexecutor/`、`handle/` 等)。提交入口的函数名随版本变化：v8.5.0 的 `handle/handle.go` 中为 `func SubmitTask(...)`(逐行核对，函数体为 GetTaskManager→GetTaskByKeyWithHistory→CreateTask→GetTaskByID→`failpoint.InjectCall("afterDXFTaskSubmitted")`→NotifyTaskChange)；而在 v7.5.x 中该入口函数名为 `SubmitGlobalTask`(v7.5.5 同文件确认，且该版本函数体内无任何 failpoint)，并非 `SubmitTask`。换言之，v8.5.0 的 DXF 提交路径上确实存在且仅存在一个具名 failpoint `afterDXFTaskSubmitted`(位于 `SubmitTask` 内、`NotifyTaskChange` 之前)，`handle/` 目录内别无其他具名 failpoint。必须强调：DXF 与上一段的 MPP 查询执行是两套不同机制——DXF 调度的是 DDL / 导入这类后台作业，MPP(`fragment.go` + `mpp_gather.go`)才是 TiFlash 上的分析型查询并行执行，把二者混为一谈是常见误读。

涉及的源码模块：`github.com/pingcap/tidb` @ `release-8.5` commit `67b4876bd57b` — `pkg/planner/core/`(optimizer.go、find_best_task.go、physical_plans.go、fragment.go)、`pkg/planner/cardinality/`(基数估计，已确认到文件级)、`pkg/planner/cascades/`(非默认)、`pkg/executor/`(coprocessor.go、mpp_gather.go、join/、internal/mpp/、mppcoordmanager/)、`pkg/disttask/framework/`(DXF)。 〔文献[1-5,11-12]〕

## 12.3 OceanBase 的实现

与 TiDB 的进程分离形态相对，OceanBase 的优化器与执行引擎内嵌在一体化的 OBServer 进程中(详见第 6 章)，源码分布在 `src/sql/optimizer/`、`src/sql/executor/`、`src/sql/engine/`。4.3.5 架构文档把请求路径描述为 Parser、Resolver、Transformer、Optimizer、Code Generator、Executor：Transformer 做等价改写，Optimizer 同时考虑 SQL 语义、对象数据特征与物理分布，解决访问路径、Join 顺序、Join 算法与 distributed plan generation，Code Generator 生成可执行代码，Executor 发起执行。这意味着 OceanBase 优化器不是在逻辑计划完成后才「附加」分布式属性，物理分布本身就是计划搜索的重要输入——这也是它区别于 TiDB 的核心结构。

**优化器入口与代价模型。** 优化器主类是 `ObOptimizer`(`src/sql/optimizer/ob_optimizer.h`，4.3.5 @ `b28b9bb12f3b`，L176 起)。同目录关键文件：`ob_join_order.cpp/.h`(Join order 枚举)、`ob_opt_est_cost.cpp/.h` 与 `ob_opt_est_cost_model.cpp/.h` 及 `ob_opt_cost_model_parameter.cpp/.h`(代价模型与参数)、`ob_log_join_filter.cpp/.h`(join filter / bloom filter join 的逻辑算子)、`ob_px_resource_analyzer.cpp/.h`(PX 资源分析)、`ob_skyline_prunning.h`(skyline 剪枝，用于裁剪冗余索引访问路径)。优化器内部具体阶段划分仅确认到文件级。4.3 起为列存引入了专门的代价模型与向量化引擎，优化器可基于代价自动决定走行存还是列存。

**计划态：本地 / 远程 / 分布式(及未初始化 / 不确定)。** 这是 OceanBase 优化器的标志性设计。物理计划类型为枚举 `ObPhyPlanType`，定义在 `src/sql/ob_sql_define.h`，逐字核对完整有 5 个成员：`OB_PHY_PLAN_UNINITIALIZED = 0`、`OB_PHY_PLAN_LOCAL`、`OB_PHY_PLAN_REMOTE`、`OB_PHY_PLAN_DISTRIBUTED`、`OB_PHY_PLAN_UNCERTAIN`。其中 `UNINITIALIZED` 是初始占位、`UNCERTAIN` 表示编译期无法静态确定(如绑定变量决定的计划)，真正描述「数据在哪执行」的是 LOCAL / REMOTE / DISTRIBUTED 三个操作态——本章谈「计划三态」特指这三个操作态，而非该枚举只有三个成员。其语义是：数据全在本节点 → local plan(无网络)；数据全在另一节点 → remote plan(整条计划 RPC 到对端执行，只回结果)；数据跨多节点 → distributed plan(走 PX 并行)。官方文档进一步说明 parallel execution 既可用于 distributed plan，也可用于 local plan，通过多个执行线程并行处理计划。这一分层让单分区点查可以走极轻的本地路径，只有真正跨分区的查询才付出分布式代价。值得注意的是，plan cache 按 Tenant 与 server 维度缓存 local / remote / distributed 三类计划，说明计划类型不仅影响一次执行，也影响后续复用与失效边界；压测中若只清理应用端连接而不观察服务端 plan cache，容易把第一次执行的 plan 误判为稳定行为。请求最终由谁路由也需厘清：OBProxy / ODP 负责把请求路由到合适 OBServer，但 SQL 优化与执行由 OBServer 内部完成。

**表 12-1　TiDB task 类型 vs OceanBase 计划态平行对照**

| 维度 | TiDB | OceanBase |
|---|---|---|
| 划分依据 | EXPLAIN 按 task 分层（算子在哪一层执行） | 物理计划按计划态分（数据在哪执行），`ObPhyPlanType` 枚举 |
| 本地执行 | `root`（TiDB 本地汇总） | `OB_PHY_PLAN_LOCAL`（数据全在本节点，无网络） |
| 单节点远程 | — | `OB_PHY_PLAN_REMOTE`（整条计划 RPC 到对端执行，只回结果） |
| 下推／存储层就地执行 | `cop[tikv]` / `cop[tiflash]` / `batchCop`（Coprocessor 就地扫描过滤聚合） | （并入分布式态，由 PX 内 DFO 切分下推） |
| 跨节点并行 | `mpp[tiflash]`（TiFlash MPP，Fragment + Exchange） | `OB_PHY_PLAN_DISTRIBUTED`（走 PX 并行，QC→DFO→SQC→GI） |
| 其他枚举成员 | — | `OB_PHY_PLAN_UNINITIALIZED`（初始占位）/ `OB_PHY_PLAN_UNCERTAIN`（编译期无法静态确定） |
| 并行执行框架归属 | 查询并行 = MPP（仅 TiFlash 列存）；DXF 为另一套（后台 DDL／导入） | PX（行存列存通用） |

**分布式 Join 分发(broadcast / shuffle / partition-wise)。** 三种分发策略在优化器逻辑层由 `enum DistAlgo` 表达(定义在 `src/sql/optimizer/ob_log_operator_factory.h`)。逐字核对该枚举，真实成员包含：`DIST_BASIC_METHOD`(local / remote join)、`DIST_BROADCAST_NONE` / `DIST_NONE_BROADCAST`(broadcast join，小表广播)、`DIST_HASH_HASH`(hash-hash repartition，即 shuffle)、`DIST_PARTITION_WISE`(纯 partition-wise join，两表按相同分区键共置则各分区就地 join，零网络重分布)、`DIST_EXT_PARTITION_WISE`(扩展 partition-wise)、`DIST_PARTITION_NONE` / `DIST_NONE_PARTITION`(partial partition-wise / repartition 到目标分区)、`DIST_HASH_NONE` / `DIST_NONE_HASH`、`DIST_PULL_TO_LOCAL` 等共二十余个(以 `1UL << n` 位标志定义)。`ob_join_order.h` 中可见对 `DIST_PARTITION_WISE || DIST_EXT_PARTITION_WISE` 的判断，且该体系在 4.2.5 同样使用。需澄清的是 OceanBase 分发概念分两层：优化器逻辑层用 `DistAlgo` 选定分发「算法」，到 PX 物理层则落到另一枚举 `ObPQDistributeMethod`(`ob_sql_define.h`，成员含 `NONE / PARTITION / RANDOM / HASH / BROADCAST / BC2HOST / SM_BROADCAST / PARTITION_HASH / HYBRID_HASH_BROADCAST / LOCAL` 等)，官方 `PQ_DISTRIBUTE` hint 暴露的 `HASH / BROADCAST / PARTITION / NONE` 即对应后者。二者是同一分发机制在优化器层与执行层的两套枚举，并存不冲突。

**表 12-2　分布式 Join 分发「四套命名」映射**

| 概念／角色 | TiDB 命名 | OceanBase 命名 | 说明 |
|---|---|---|---|
| 分发策略（优化器逻辑层） | MPP 内 `tipb.ExchangeType`（Broadcast / Hash / PassThrough） | `DistAlgo` 枚举（`DIST_BROADCAST_NONE` / `DIST_HASH_HASH` / `DIST_PARTITION_WISE` 等二十余个） | 同为「广播／重分布／共置」选择；OceanBase 在优化器逻辑层先选分发算法 |
| 分发方式（执行物理层） | `tipb.ExchangeType` 直接驱动 MPP 数据分发 | PX 物理层 `ObPQDistributeMethod`（NONE / PARTITION / HASH / BROADCAST 等），`PQ_DISTRIBUTE` hint 暴露 HASH / BROADCAST / PARTITION / NONE | 二者是同一分发机制在优化器层与执行层的两套枚举，并存不冲突 |
| 数据交换算子 | `ExchangeSender` / `ExchangeReceiver` | EXCHANGE OUT / IN（经数据传输层 DTL 传输） | 均为算子间跨节点数据收发；OceanBase 经 DTL 流转 |
| 执行切片单元 | Fragment（按 `ExchangeSender` / `ExchangeReceiver` 切分） | DFO（Data Flow Operation，按数据传输算子切分） | 都把物理计划按数据交换边界切成并行片段 |
| 并行调度角色 | MPPGather 下发、TiFlash 节点并行执行 | QC（Query Coordinator）调度 → SQC（Sub Query Coordinator）→ PX worker；GI（Granule Iterator）切粒度 | TiDB 由 root 层 MPPGather 驱动；OceanBase 由 QC→SQC→GI 多级调度 |

这套分发能否生效与分区设计强相关。分区表文档说明 HASH partitioning 适合需要 parallel DML、partition pruning、partition-wise joins 的场景；composite partitioning 可在一个或两个维度上做 partition pruning，也可做 partial / full partition-wise joins。换言之，当两张大表在 join key 上具备兼容分区布局时，优化器有机会避免全量 shuffle，让每个分区或分区组局部 join；反过来，如果分区键与 join 条件不匹配，执行引擎就需要更多跨节点数据交换。

**PX 并行执行引擎。** OceanBase 的并行执行模型称为 PX，源码在 `src/sql/engine/px/`。其组成：分布式计划按数据传输算子切成多个 DFO(Data Flow Operation)，由 QC(Query Coordinator)调度；QC 把 DFO 下发到各 OBServer，节点上临时拉起 SQC(Sub Query Coordinator)申请线程与资源、构建执行上下文，再启动该节点的 PX worker；Granule Iterator(GI)负责把扫描任务切成 partition granule 或 block granule 以均衡并行粒度。源码可见 `ob_dfo.cpp/.h`、`ob_dfo_scheduler.cpp/.h`、`ob_px_coord_op.cpp/.h`、`ob_px_sub_coord.cpp/.h`、`ob_px_sqc_handler.cpp/.h`、`ob_granule_iterator_op.cpp/.h`、`ob_granule_pump.cpp/.h`，且在 4.2.5 / 4.4.x 中均存在(`src/sql/dtl/`、`src/sql/engine/px/exchange/` 含 PX receive / transmit 与 DTL、runtime filter 相关文件)。DFO 之间是生产者-消费者关系，一个 DFO 在下一阶段会从消费者转为生产者；数据经 EXCHANGE OUT / IN 算子与数据传输层(DTL)传输，QC 本身是一个特殊的 EXCHANGE IN，兼具收数与调度子 DFO 的功能。并行度 DOP 由 `PARALLEL` hint 或租户配置(`px_workers_per_cpu_quota` 等)决定。PX 的收益与代价都来自拆分：当扫描与 Join 能均匀分摊到多个分区时响应时间下降；但数据倾斜、join key 热点或上游过滤选择性很低时，某些 worker 会成为长尾，exchange 队列与内存成为瓶颈。本章仅做模块级源码引用，不把 `src/sql/engine/px/` 内部类名泛化成用户可观察接口。

**4.x AP 优化与 bloom filter join。** OceanBase 4.3 引入列存 + Vectorized Engine 2.0，显著提升 OLAP 能力。在 PX 并行 Join 中，OceanBase 以 join filter / bloom filter 做运行期过滤：build 侧构建 bloom filter，下推到 probe 侧扫描提前裁剪不匹配行。执行引擎侧实现在 `src/sql/engine/px/ob_px_bloom_filter.cpp/.h` 及 SIMD 加速版 `ob_px_bloom_filter_simd.cpp`，join filter 算子在 `src/sql/engine/join/ob_join_filter_op.cpp/.h`，优化器侧逻辑算子在 `ob_log_join_filter.cpp/.h`。此外 4.x 还有一批自适应技术：基于滑动窗口的自适应 join filter(过滤效果不佳则后续窗口停用)、自适应 HASH GROUP BY(按实时 NDV 决定是否两阶段聚合)、自适应混合哈希 shuffle(对高频值特殊处理以抗数据倾斜)。需要按版本与形态厘清能力归属：列存与 CS_ENCODING 自 V4.3.0 引入，4.3.5 为 AP LTS、4.4.x 为 TP + AP 融合演进，这些 AP 能力应理解为 4.x 的演进方向，不声称在 4.2.5_CE(TP)中完全等价，也不引用未查证的企业版 / Cloud 内部开关。

**执行框架。** 执行器经 Job / Task 切分 + RPC 直传 / 直收在多节点执行，算子均派生自 `ObOperator`(`src/sql/engine/ob_operator.cpp/.h`)，内存受租户级 SQL memory manager(`ob_tenant_sql_memory_manager.cpp/.h`)约束；物理 Join 算子(Hash / Merge / Nested Loop，含向量化版本)在 `src/sql/engine/join/`；并行 DML(PDML)在 `src/sql/engine/pdml/`。executor 内部 Job→Task 切分细节仅确认到模块 / 文件级。

涉及的源码模块：`github.com/oceanbase/oceanbase` @ 4.3.5 `b28b9bb12f3b`(并交叉 4.2.5 `e7c676806fda`、4.4.x `d4bef8d29a4c`)— `src/sql/optimizer/`、`src/sql/executor/`、`src/sql/engine/px/`、`src/sql/engine/join/`、`src/sql/engine/pdml/`、`src/sql/dtl/`(均已确认到文件级)。 〔文献[6-10,13]〕

## 12.4 核心差异对比

**表 12-3　SQL 优化器与执行引擎:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB | OceanBase | 影响 |
|---|---|---|---|
| SQL 层形态 | TiDB Server 独立无状态 SQL 计算层，数据在 TiKV / TiFlash | OBServer 内聚 SQL、事务、存储访问，OBProxy / ODP 前置路由 | TiDB 强调 SQL 计算与存储分离；OceanBase 优化器更直接感知本地 / 远程 / 分布式计划 |
| 优化器范式 | 两阶段：RBO 规则链(`optRuleList`)+ CBO(`FindBestTask`，System-R / Volcano 风格)；另存 `cascades` 框架但非默认 | CBO(`ObOptimizer` + 列存专用代价模型)；Transformer 改写后，优化器结合物理分布生成计划；自动行 / 列选择 | 两者都是 CBO 主导，差异在代价模型对存储格式与物理分布的感知粒度 |
| 计划分层模型 | 按 task 分：root / cop[tikv] / cop[tiflash] / mpp[tiflash] | 按计划态分：local / remote / distributed(`OB_PHY_PLAN_*` 枚举，5 成员) | OceanBase 显式区分本地 / 远程 / 分布式，单分区查询路径更轻 |
| 并行执行框架 | 查询并行 = MPP(仅 TiFlash 列存，Fragment + Exchange)；DXF 是另一套(后台 DDL / 导入) | PX(行存列存通用，QC→DFO→SQC→GI) | TiDB 大规模查询并行强绑定 TiFlash；OceanBase PX 不依赖独立列存节点 |
| 分布式 Join 表达 | `tipb.ExchangeType`：Broadcast / Hash / PassThrough(MPP 内) | 优化器层 `DistAlgo` 枚举(broadcast / hash-hash / partition-wise / partial-pw 等二十余个)+ PX 物理层 `ObPQDistributeMethod` | OceanBase 分发策略更细，partition-wise 在分区对齐时零重分布 |
| 下推机制 | Coprocessor DAG 下推到 TiKV；TiFlash Coprocessor / MPP | 算子下推 + PX 内 DFO 切分；bloom filter / join filter 运行期下推 | TiDB 下推到独立存储进程(跨进程 RPC)；OceanBase 下推在同进程 / 同节点内更多 |
| 运行期自适应 | 较少(以静态 CBO + hint + plan cache 为主) | 自适应 join filter / HASH GROUP BY / 混合 hash shuffle | OceanBase 更强调对估错与倾斜的运行期纠偏 |
| 统计信息 | 直方图(等深)+ Top-N + CM Sketch(`cardinality/` 模块) | 列级统计 + 列存统计指标(列存专用代价模型) | 统计过期都会造成跨节点数据搬运放大，详见 §12.9 |
| EXPLAIN 风格 | id / estRows / task / access object / operator info，`task` 列暴露执行层 | ID / OPERATOR / NAME / EST.ROWS / COST + Outputs & filters / partitions / range | 见 §12.9 示例 |

> 上表「单点」类表述均已规避：OceanBase 的 QC 是逻辑调度中心而非高可用单点，PX worker 故障由查询级重试 / 报错处理，不等同于集群级 SPOF(控制面五维度区分详见第 13 章)。

## 12.5 正常路径图

图 12-1 为一条跨分区分析型 `JOIN ... GROUP BY` 在两个系统中的正常执行路径对比。

![[f12_1.svg]]

**图 12-1　SQL 优化器与执行引擎正常读写/调度路径**

正常路径的共同点是：优化器先把 SQL 语义变成可执行计划，再由执行器驱动扫描、过滤、Join、聚合。差异在于 TiDB 的 `task` 列直接暴露 root / coprocessor / MPP 位置，OceanBase 则把 local / remote / distributed 与 PX 拆分放在 OBServer 的 SQL engine 内部，并在前置加一道 OBProxy / ODP 路由。

## 12.6 故障/异常路径图

图 12-2 展示「优化器估错导致跨节点数据爆炸」这一最典型的执行引擎故障路径，以及两系统的兜底机制差异。故障路径不只有节点宕机：对优化器章节更常见的异常是统计信息过期、数据倾斜、NULL 语义、TiFlash replica 不满足 MPP 条件、OceanBase partition pruning 未命中、PX worker 或 exchange 通道成为瓶颈。

![[f12_2.svg]]

**图 12-2　SQL 优化器与执行引擎故障/异常路径**

另一类故障是控制面 / 路由侧：TiDB Region 发生 Leader 切换或 split 时，cop task 收到 RegionError(`EpochNotMatch` / `NotLeader`)触发 RegionCache 失效与 cop task 重建，执行信息可能显示 backoff、RPC 重试或 MPP store fail 相关行为(详见第 2 章)；OceanBase 在 major compaction 或负载均衡切主后路由表刷新，distributed plan 的 SQC 在目标 Tablet 迁移后需重新定位 Log Stream Leader，worker / exchange 通道或目标 OBServer 异常时需中断、清理或重新执行子计划(详见第 1、2 章)。这类故障由路由层吸收，不等同于事务失败，通常不会让查询直接失败，但会带来抖动与 tail latency。具体内部错误码、事件名与诊断视图名公开资料不足以写死，本章只写可观察类别。

## 12.7 性能、可靠性、运维影响

理清两套实现后，可以从性能、可靠性、运维三个维度看它们的实际影响。

**延迟与吞吐。** 两系统的优化核心都是「让数据少动」。下推(谓词 / 聚合 / TopN)把过滤前移到存储层，直接削减跨层传输：一个 `count(*)` 若在 TiKV cop 层就完成局部聚合，回传 TiDB 的只是每个 Region 的部分计数而非全部行；TopN 若下推，cop 层每个 Region 只回 N 行。partition-wise join(OceanBase，见 §12.3)与就地 cop 计算(TiDB)在数据共置时可做到近零重分布——这正是分布式优化器能把延迟压到接近单机的关键。具体到数据流动方式，TiDB 的 MPP 是 push 模型：ExchangeSender 主动把按哈希切好的数据推给上游 ExchangeReceiver，小批流式传输，避免全量物化；OceanBase 的 PX 同样是 DFO 间生产者-消费者推送，经数据传输层(DTL)流转。对 OLAP 大查询，TiDB 依赖 TiFlash MPP 的列存 + 向量化，OceanBase 依赖 PX + 列存(4.3+)+ bloom filter，真正的差异在 workload 形态：TiDB 把 AP 流量物理隔离到 TiFlash 节点，OceanBase 在同一 OBServer 内用租户级资源隔离区分 TP / AP。OceanBase AP 优化并非「开并行度越大越好」：官方博客对 adaptive join filter 的讨论说明，join filter 有创建、发送、应用三类开销，当选择性低时减少的 shuffle 不足以抵消开销，甚至会变慢——要把分区布局、统计信息、PX degree、join filter 选择性与租户资源一起验证。

**可用性与故障恢复。** 查询执行层的失败基本是「单查询级」——某个 cop task / PX worker 失败，代价是这条查询重试或报错，不影响集群。控制面侧(PD / RootService)的角色切换才影响计划编译与调度，但二者均为逻辑中心化、物理高可用(详见第 13 章)，不可简单写「单点」。优化器与执行器的异常恢复通常表现为 retry、backoff、plan invalidation 或重新调度，而不是协议层的提交恢复。

**扩展性。** TiDB 的查询并行扩展强绑定 TiFlash 节点数(横向加 TiFlash 提升 MPP 并行度)，但增加 TiDB Server 并不自动减少 TiKV / TiFlash 热点；OceanBase 的 PX 并行度受租户 Unit 的 CPU 配额约束，扩并行 = 扩租户资源或加节点，且并发线程增加而数据分布不变时反而可能加剧热点。

**运维复杂度。** TiDB 需要分别管理「行存 OLTP 计划」与「TiFlash MPP 计划」，并维护副本同步、`tidb_broadcast_join_threshold_size` 等阈值；OceanBase 在一套引擎内管理行 / 列与三态计划，但 PX 调优(DOP、`PQ_DISTRIBUTE` hint、SQL memory)与 major merge 抖动(详见第 4 章)是其特有运维面。两边都需把计划稳定性作为持续指标，而非上线前一次 EXPLAIN：数据导入、热点租户、列存副本状态、分区增加、索引变更、统计信息自动更新都会让计划改变。一个务实做法是把核心 SQL 的计划形态纳入回归，记录版本、SQL digest、表规模、统计信息时间、EXPLAIN 文本与慢查询 runtime stats，并在升级或数据量跨越阈值时对比是否出现新的 broadcast、shuffle、全分区扫描或大量回表；这一做法属工程经验推测，需结合各自环境验证。

**表 12-4　性能、可靠性、运维影响**

| 性能瓶颈来源对比 | TiDB | OceanBase |
|---|---|---|
| 跨节点传输爆炸 | broadcast 误选 / IndexJoin 内表估值爆炸 → cop OOM | broadcast 误选 / hash-hash 倾斜 → 单 worker 长尾 |
| 计算层瓶颈 | TiDB Server 汇总 root task 单点串行段 | QC 汇总 + 单 DFO 长尾 |
| 存储层瓶颈 | TiKV cop 线程池 / TiFlash 扫描带宽 | OBServer SQL 内存配额 / major merge 抖动 |
| 统计信息失真 | 伪统计 fallback、过时统计 | 估错由运行期自适应部分纠偏 |
| 缓解手段 | hint / plan cache / 关闭过时伪统计 | 自适应 join filter / GROUP BY / hash shuffle + hint |

## 12.8 反例与代价

优化器的代价不在「跑得慢一点」，而在它把局部失误放大成跨节点搬运。下面四类反例各自对应一种放大路径。

**第一类：统计信息错导致跨节点放大。** 以 `orders` 与 `customers` 按 `customer_id` join 为例，优化器把 `customers` 过滤后结果误估为很小：TiDB TiFlash MPP 可能选 broadcast，小表实际很大时会把大量行广播到每个 TiFlash 节点，叠加 gRPC 回传过快导致 cop OOM；OceanBase 同理，PX 可能为 hash repartition 或 join filter 分配不合适的执行策略，造成 exchange 通道拥塞。这是两系统跨节点数据爆炸的头号原因，也是 §12.6 的核心反例。配套兜底：TiDB 的 `tidb_enable_pseudo_for_outdated_stats` 自 v6.3.0 默认 OFF，以缓解统计缺失 / 过时时 fallback 到伪统计、可能选全表扫描的问题。

**第二类：索引看起来存在，但路径仍不便宜。** TiDB Index Join 的内表 fetch / build / probe 都有运行期成本；外表行数过多时，内表索引回表分散在大量 Region，RPC 会主导延迟。已知踩坑：某些场景 TiDB 对 IndexJoin 内表 `IndexRangeScan` 给出极高估计导致选错计划，官方曾尝试修复又因引发回归而 revert(pingcap/tidb issue #44855)，最终以 optimizer-fix-control 开关提供。OceanBase 普通索引与全局索引在 EXPLAIN 中体现不同访问路径；若谓词不能在索引上过滤，`filter_before_indexback` 类似信息会暴露部分过滤只能在回表后做，网络与 IO 都会上升。

**第三类：分区设计与查询模式脱节。** OceanBase composite partitioning 可支持 partition pruning 与 partial / full partition-wise join，但前提是查询谓词与 join key 能落在分区维度上；否则两表强行共置会退化为 repartition，partition-wise 的零重分布优势消失，更多分区只带来更多元数据、更多 PX granule 与更多调度成本。需特别厘清：TiDB Region 自动 split 并不等于 SQL 层天然获得 partition-wise join——Region 是 KV key range 与 Raft Group 边界，不能当作 SQL join 分区语义。

**第四类：AP 能力与并行框架的形态依赖。** TiDB 查询并行依赖 TiFlash：没有 TiFlash 副本时，大规模分析查询只能靠 TiKV cop + TiDB 单点汇总，缺乏跨节点 shuffle 能力，复杂多表 JOIN 的 OLAP 性能受限；DXF 对 TiDB Cloud Starter / Essential 不可用。OceanBase 4.3 列存 / 向量化 / 列存代价模型是 4.x 演进主题，但企业版、CE、Cloud 与不同 LTS 的能力归属要逐条核对。此外，计划态(LOCAL / REMOTE / DISTRIBUTED 三操作态)+ DistAlgo 二十余种分发 + PX 调度带来更大的计划搜索空间与更高的优化器 / 调度复杂度，distributed plan 的编译与调度本身有开销，单分区轻查询必须能正确收敛到 local plan；数据倾斜下 hash-hash 单 worker 成长尾，虽有自适应混合 hash shuffle 兜底但切换本身有成本；major merge 抖动会传导到 AP 查询扫描延迟(详见第 4 章)。

**取舍本质。** TiDB 用「物理隔离的列存 + 强绑定的 MPP」换取 TP / AP 干扰最小化，代价是多一套存储与计划；OceanBase 用「一体化引擎 + 通用 PX + 运行期自适应」换取无需独立 AP 节点，代价是更高的优化器 / 调度复杂度与租户内资源争用风险。无优劣，是把复杂度放在不同位置。本章不编造 optimizer hint、metric、内部视图或函数名，能查证的只列命令与 EXPLAIN 字段。 〔文献[14]〕

## 12.9 测试开发视角的验证点

将上述机制与代价落到验证层面，可从功能场景、失效注入与压测指标三方面着手。

**可测试的功能场景。** 谓词下推是否生效；`ORDER BY ... LIMIT` 是否 TopN / Limit 下推；聚合是否在 cop 或 mpp 层先做 partial aggregation；TiDB Index Join 与 HashJoin 是否随统计信息变化而切换；TiFlash MPP 是否出现 `ExchangeSender` / `ExchangeReceiver`；OceanBase 单分区查询是否走 local / remote 简单计划；跨分区 Join 是否出现 distributed / PX exchange；partition-wise join 是否因分区键对齐而减少重分布；行存 / 列存自动选择；plan cache 命中与失效；hint(`BROADCAST_JOIN`、`PQ_DISTRIBUTE`、`PARALLEL`)是否生效。

**可注入的失效模式。** 删除或锁定统计信息(看是否走伪统计 / 退化计划)、人为制造数据倾斜(单值占比 90%+)观测 broadcast / hash-hash 与自适应纠偏、让 TiFlash replica 未同步、对 TiDB 查询设置 `tidb_enforce_mpp=1` 后观察 warning、在 OceanBase 中构造分区键与 join key 不匹配的大表 join、压低 PX 可用资源或限制 SQL 内存配额触发落盘、构造 NULL 值比例极高的 join key、Region split / Tablet 迁移期间跑长查询观测路由失效重试。目标不是追求某个固定计划，而是验证计划变化是否符合统计信息、索引与分布条件——此判定标准为工程推测，应在具体负载上复核。

**关键压测指标(分三层)。** SQL 层看 P50 / P95 / P99 latency、rows returned、scan rows、network bytes、memory peak、spill；TiDB 看 `EXPLAIN ANALYZE` 中 `estRows` vs `actRows`、`cop_task`、`rpc_info`、HashJoin / IndexJoin runtime 字段、`execution info` 中的 `threads`；OceanBase 看 `EXPLAIN EXTENDED` 中 operator tree、`partitions`、`range`、`outputs & filters`、是否出现 distributed / PX exchange。`SHOW STATS_META` 看 `modify_count` / `row_count` 比例判断统计是否过时(官方建议比例偏高时考虑 `ANALYZE`)。系统变量 `tidb_broadcast_join_threshold_size`、`tidb_index_join_batch_size`、`tidb_enable_pseudo_for_outdated_stats`、`tidb_enable_dist_task` 可作开关型观测 / 控制点。可在源码核对的诊断视图与 metric：OceanBase 用于 SQL 执行诊断的 GV$ 视图名在 `src/share/inner_table/ob_inner_table_schema_constants.h` 中以常量定义并逐字核对——`GV$OB_SQL_AUDIT`(SQL 执行审计 / 诊断)、`GV$SQL_PLAN_MONITOR`(SQL 计划执行监控，用于观测 PX / 并行执行的算子级运行信息)、`GV$OB_PLAN_CACHE_PLAN_STAT` 与 `GV$OB_PLAN_CACHE_PLAN_EXPLAIN`(plan cache 计划统计与执行计划)、`GV$OB_SQL_PLAN`，4.2.5 与 4.3.5 源码均存在；TiDB 侧 Coprocessor / distsql 相关 Prometheus metric 在 `pkg/metrics/distsql.go` 中以 `Namespace: "tidb"` + `Subsystem: "distsql"` 定义，如 `tidb_distsql_handle_query_duration_seconds`、`tidb_distsql_scan_keys_num`、`tidb_distsql_copr_cache`、`tidb_distsql_copr_resp_size` 等。trace event 的具体名称未逐条查证，本章不编造、只标其名称待查证；需特别说明 OceanBase 的 `GV$OB_SQL_AUDIT` 是 SQL 执行诊断视图，不等于合规意义上的 Security Audit。

**功能验证可按「同一 SQL、三种数据分布」组织。** 第一组均匀分布，预期优化器按选择性选索引、广播小表或局部 join；第二组热点分布(如 1% key 占 80% 行)，预期 runtime stats 能暴露 worker 长尾或 `actRows` 与 `estRows` 的偏差；第三组统计信息过期，预期刷新统计后计划恢复到更合理路径。这比单纯跑 TPC-H 更能揭示优化器在业务数据上的边界。测试报告需记录版本、租户 / 集群形态、是否存在 TiFlash replica、OceanBase 是否启用列存 / AP 能力、表分区与索引定义；据工程经验推测，缺少这些上下文时，EXPLAIN 片段很难被其他章节复用。

**EXPLAIN 风格对比(同一逻辑「带过滤 + 排序取 TopN」查询)。**

TiDB(`explain select * from t order by a limit 10;`，官方文档原样)：

```text
| id | estRows | task | operator info |
| TopN_7 | 10.00 | root | test.t.a, offset:0, count:10 |
| └─TableReader_15 | 10.00 | root | data:TopN_14 |
| └─TopN_14 | 10.00 | cop[tikv] | test.t.a, offset:0, count:10 |
| └─TableFullScan | 10000.00 | cop[tikv] | table:t, keep order:false |
```

读法：从下往上、缩进表父子；`task` 列直接告诉你算子跑在哪一层(`cop[tikv]` = 下推到 TiKV)；可见 TopN 被下推，cop 层只回 10 行。一个跨表 `JOIN ... COUNT(*)` 的 TiFlash MPP 计划骨架(取自官方 MPP 文档同类示例的算子风格)则会出现 `HashAgg`、`HashJoin`、`ExchangeSender`(ExchangeType： Broadcast / PassThrough)、`ExchangeReceiver`、`TableFullScan` 等 `mpp[tiflash]` 算子，直观展示小表 Broadcast、大表本地扫描的分发选择。

OceanBase(同类查询，结构示意，字段以官方 EXPLAIN 为准)：

```text
| ID | OPERATOR | NAME | EST.ROWS | COST |
| 0 | LIMIT | | 10 | ... |
| 1 | └─TOP-N SORT | | 10 | ... |
| 2 | └─TABLE SCAN | t | 10000 | ... |
Outputs & filters:
 ...
```

跨分区 Join 的 OceanBase 骨架则会出现 `SCALAR GROUP BY`、`PX COORDINATOR`、`EXCHANGE IN / OUT DISTR`、`HASH JOIN`、`PX PARTITION ITERATOR`、`TABLE SCAN` 等算子，COST 列直接给代价值，并附 `Outputs & filters`(含 `partitions(...)`、`range(...)`、join 条件)与 `Outline Data` 段。两者最大风格差异：TiDB 用 `task` 列暴露「算子在哪一层执行」，OceanBase 则用计划态(local / remote / distributed)与 PX 算子(EXCHANGE / PX COORD)暴露「数据如何在节点间流动」。示例仅用于说明 EXPLAIN 风格，不用于 benchmark 排名。

## 12.10 容易误解点

1. **「TiDB 有 `pkg/planner/cascades`，所以默认使用 Cascades」——错。** 源码存在只说明模块存在；开发指南把 Cascades model planner 写为 under development and disabled by default，8.5 主线是 core / System-R / Volcano 风格 CBO(`physicalOptimize`→`FindBestTask`)。

2. **「TiDB DXF 就是分布式查询执行框架」——错。** DXF(`pkg/disttask/framework`)是 `ADD INDEX` / `IMPORT INTO` 等后台任务的调度框架，引入于 v7.1、v7.5 GA(据 7.5.0 release notes)、v8.1 默认开；TiFlash 上的分析型查询并行是 MPP(`fragment.go` + `mpp_gather.go`)，二者是两套独立机制。提交入口函数名在 v8.5.0 为 `SubmitTask`、在 v7.5.x 为 `SubmitGlobalTask`(版本相关)。

3. **「partition-wise join 总是最快」——错。** 它只在两表按相同分区键共置时零重分布；若分区键不匹配，优化器会退化为 hash-hash repartition 或 broadcast，优势不复存在，甚至因强行共置而引入额外代价。

4. **「broadcast join 一定比 shuffle 省」——错。** 仅当被广播表足够小时成立。小表估错(统计过时)是两系统跨节点数据爆炸的头号原因：TiDB 把大表广播到每个 TiFlash 节点导致 cop 内存耗尽，OceanBase 同理。这正是 §12.8 的核心反例。

5. **「OceanBase PX / join filter 一定提升 OLAP」——错。** PX 能提高并行度，但也引入 exchange、线程调度与内存压力；官方博客明确说明 join filter 在低选择性时，创建、发送、应用的开销可能超过减少的 shuffle。所谓「OceanBase 比 TiDB 更复杂所以更慢 / TiDB 更简单所以更弱」也是误读：二者只是把复杂度放在不同位置(TiDB 外置到 TiFlash + MPP，OceanBase 内置到一体化引擎 + PX + 运行期自适应)。各类 benchmark 成绩均有特定硬件、数据规模与利益相关方，不可当作绝对排名：4.3 列存 + 向量化「显著提升 OLAP 能力」有多来源交叉佐证，但具体某一版本对比的精确百分比(如曾流传的 TPC-H 提升数值)因来源页对自动化请求限流而未能逐字复核，本章不把任何单一百分比当作既定事实。

## 12.11 本章结论

1. TiDB 优化器是「RBO 规则链(`optRuleList`)→ CBO(`physicalOptimize` / `FindBestTask`，System-R / Volcano 风格)」两阶段，task 分 root / cop / mpp 三层，可下推算子经 Coprocessor DAG 下沉到 TiKV、大规模并行靠 TiFlash MPP(Fragment + ExchangeType：Broadcast / Hash / PassThrough)；阅读计划应先看 `task` 列以定位数据在哪里被过滤、聚合与 join。

2. TiDB 源码存在 Cascades 相关目录，但开发指南把默认生产路径限定在 core / System-R / Volcano 风格 CBO，不能把源码目录直接写成默认优化器行为。

3. OceanBase 优化器是 CBO(`ObOptimizer` + 列存专用代价模型，4.3 起自动行 / 列选择)，把对象数据特征与物理分布纳入计划生成，其标志是计划态(`ObPhyPlanType` 枚举共 5 成员，操作态为 `OB_PHY_PLAN_LOCAL / REMOTE / DISTRIBUTED`)与优化器层 `DistAlgo` 分发枚举(broadcast / hash-hash / partition-wise，PX 物理层对应 `ObPQDistributeMethod`)，并行执行靠 PX(QC→DFO→SQC→GI)，运行期有 bloom filter join 与一批自适应纠偏。

4. 两系统优化器的最大风险点一致：估错(尤其小表估小 + broadcast 误选 + 数据倾斜)会把本应就地过滤的数据跨节点搬运，从「多扫一点」升级为「多搬很多」，造成 cop OOM 或单 worker 长尾；TiDB 以 hint / plan cache / 关闭过时伪统计兜底，OceanBase 以运行期自适应 + hint 兜底；据此推测，这是工程取舍的不同放置点而非优劣之分。

5. DXF ≠ MPP：DXF 是 TiDB 后台分布式任务框架(v7.1 引入、v7.5 GA 据 release notes、v8.1 默认开；提交入口 v8.5.0 为 `SubmitTask`、v7.5.x 为 `SubmitGlobalTask`)，与查询执行的 MPP 是两套机制，常被混淆；OceanBase PX 也不等于所有 SQL 自动并行。

6. EXPLAIN 风格差异本质：TiDB 用 `task` 列暴露「算子在哪层执行」，OceanBase 用计划态 + PX / EXCHANGE 算子暴露「数据如何在节点间流动」；未查证的 optimizer hint、Prometheus metric、OceanBase 内部视图与源码函数名不应写死，测试报告应优先附 EXPLAIN / EXPLAIN ANALYZE 原文与版本号。

## 12.12 参考文献

[1] Query Execution Plan Overview / Explain Statements. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/explain-overview/.
 （支撑:§12.2、§12.9 TiDB EXPLAIN 列、task 位置、stats:pseudo、IndexJoin estRows 版本边界与 TopN / Limit 下推 EXPLAIN 示例）
[2] Cost-based Optimization — TiDB Development Guide. 官方开发指南[EB/OL]. https://pingcap.github.io/tidb-dev-guide/understand-tidb/cbo.html.
 （支撑:§12.2、§12.7 Volcano / System-R 风格 CBO、physicalOptimize 三步、FindBestTask 与 copTask / rootTask / mppTask 抽象、代）
[3] SQL Physical Optimization / Use TiFlash MPP Mode. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/use-tiflash-mpp-mode/.
 （支撑:§12.2、§12.5、§12.9 MPP Fragment 切分、ExchangeSender / Receiver、Broadcast / Shuffled Hash Join、TopN / Limit 支持与 EX）
[4] Explain Statements That Use Joins / EXPLAIN ANALYZE. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/sql-statement-explain-analyze/.
 （支撑:§12.2、§12.6、§12.9 IndexJoin / IndexHashJoin / HashJoin / MergeJoin 语义、runtime stats、actRows、cop_task、rpc_）
[5] TiDB Distributed eXecution Framework (DXF). 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tidb-distributed-execution-framework/.
 （支撑:§12.2、§12.10 DXF 引入 v7.1、tidb_enable_dist_task 自 v8.1.0 默认 ON、ADD INDEX / IMPORT INTO 场景、Starter / Essential）
[6] OceanBase Database Architecture / Understand an Execution Plan V4.x. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000000829726.
 （支撑:§12.3、§12.5、§12.9 OceanBase SQL layer pipeline(Parser / Resolver / Transformer / Optimizer / Code Generator /）
[7] About Partitioned Tables V4.x. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000000829632.
 （支撑:§12.3、§12.8 partition pruning、parallel DML、partial / full partition-wise join 的前提条件）
[8] Adaptive Techniques in the OceanBase SQL Execution Engine. 官方博客[EB/OL]. https://oceanbase.github.io/docs/blogs/tech/adaptive-sql-execution-engine.
 （支撑:§12.3、§12.7、§12.8、§12.10 自适应 join filter(滑动窗口，创建 / 发送 / 应用三类开销与低选择性代价)、自适应 HASH GROUP BY、自适应混合 hash shuffle 抗倾）
[9] OceanBase Database V4.3 列存 / Vectorized Engine 2.0. 官方博客[EB/OL]. https://en.oceanbase.com/blog/12062885632.
 （支撑:§12.3、§12.10 4.3 列存引擎、向量化引擎、列存专用代价模型与自动行 / 列选择(定性结论)。可达性备注：同上整站 429，本轮未逐字复核，任一单一百分比改判为需进一步查证，核心定性结论由论文与社区镜像交叉佐）
[10] Building an OceanBase-based Distributed Nearly Real-time Analytical Processing Database System. 论文，arXiv[EB/OL]. https://arxiv.org/html/2602.07584.
 （支撑:§12.3、§12.10 列存专用代价模型、行 / 列 / 混合三数据格式、向量化，以及 benchmark 硬件条件与 caveat）
[11] The Cascades Framework for Query Optimization. 论文[EB/OL]. https://15721.courses.cs.cmu.edu/spring2016/papers/graefe-ieee1995.pdf.
 （支撑:§12.2、§12.10 Cascades / Volcano 优化器理论背景，仅解释框架概念，不用于证明 TiDB 默认实现）
[12] pingcap/tidb @ release-8.5 commit 67b4876bd57b — pkg/planner/、pkg/executor/、pkg/disttask/framework/. 源码，commit-pinned[EB/OL]. https://github.com/pingcap/tidb/blob/67b4876bd57b/pkg/planner/core/optimizer.go.
 （支撑:§12.2 optRuleList 规则链、logicalOptimize / physicalOptimize / FindBestTask、fragment.go Exchange 切分、cop）
[13] oceanbase/oceanbase @ 4.3.5 commit b28b9bb12f3b(交叉 4.2.5 e7c676806fda / 4.4.x d4bef8d29a4c) — src/sql/optimizer/、src/sql/engine/px/、src/sql/engine/join/. 源码，commit-pinned[EB/OL]. https://github.com/oceanbase/oceanbase/blob/b28b9bb12f3b/src/sql/ob_sql_define.h.
 （支撑:§12.3 ObOptimizer、ObPhyPlanType 枚举(5 成员，逐字核对)、DistAlgo 枚举(ob_log_operator_factory.h，逐字核对)、ObPQDistrib）
[14] pingcap/tidb issue #44855 — Index Lookup Join estRows 反例. 第三方 / issue[EB/OL]. https://github.com/pingcap/tidb/issues/44855.
 （支撑:§12.8 IndexJoin 内表 IndexRangeScan 估值爆炸、修复后 revert 的踩坑反例；issue 性质，仅作佐证，可信度局限已注明）
## 12.13 信息可信度自评

本章官方文档明确的部分：TiDB 的 RBO / CBO 两阶段、MPP Fragment 与 Exchange 类型、TopN / Index / Hash / Merge Join 的 EXPLAIN 语义、`tidb_enforce_mpp` 边界、DXF 的引入版本(v7.1，DXF 文档页)/ GA(v7.5,7.5.0 release notes)/ 默认开(v8.1，DXF 文档页)与 `tidb_enable_dist_task` 默认值演进，以及 OceanBase 的 SQL layer pipeline、计划态、PX 模型、partition-wise join 前提、列存专用代价模型与自适应技术，均有 docs.pingcap.com / en.oceanbase.com 官方来源。源码级信息：TiDB `optRuleList` 规则名、`physicalOptimize`→`FindBestTask`、`fragment.go` 的 ExchangeType，以及 OceanBase `ObPhyPlanType`、`DistAlgo`、`ObPQDistributeMethod`、PX / bloom filter 文件，均来自 commit-pinned 源码并经在 GitHub 上逐字核对，确认到文件级；优化器内部阶段划分、Cascades 与默认 CBO 的关系仅到模块级。论文级信息：arXiv 2602.07584(OceanBase NRTAP)提供列存代价模型与 benchmark 条件；Graefe Cascades 论文仅用于解释优化器框架概念，不用于证明 TiDB 默认实现。OceanBase adaptive join filter 与 4.x AP / vectorized 代价主要来自官方博客，可信度低于产品文档，本章已标注为版本相关的工程演进。两系统估错故障路径的具体触发链(§12.6 / §12.8)综合官方 troubleshooting 文档与 issue 推演，标注为官方实现 + 工程推测。

反查证发现与处置：其一，`afterDXFTaskSubmitted` failpoint 的归属经重核已厘清：在 v8.5.0 @ `67b4876bd57b` 的 `pkg/disttask/framework/handle/handle.go` 中,`SubmitTask` 函数体内确实存在 `failpoint.InjectCall("afterDXFTaskSubmitted")`(L87,位于 `NotifyTaskChange` 之前),`handle/` 目录内别无其他具名 failpoint；而 v7.5.5 同文件的入口 `SubmitGlobalTask` 内无任何 failpoint。早先「该 failpoint 系误记、文件无 failpoint」的表述有误(`failpoint.Inject` 与 `failpoint.InjectCall` 是两个不同 API,逐字搜 `failpoint.Inject` 会漏掉 `InjectCall`),现更正为：v8.5.0 提交链上存在且仅存在该一个具名 failpoint。其二，DXF 提交入口名版本错配已修正：`SubmitTask` 仅对 v8.5.x 成立，v7.5.x 同位置为 `SubmitGlobalTask`。其三，OceanBase「三态」措辞已精确化：`ObPhyPlanType` 实有 5 个成员(含 `UNINITIALIZED` / `UNCERTAIN`)，改写为「操作态 LOCAL / REMOTE / DISTRIBUTED 三态」。其四，`DistAlgo` 枚举经源码逐字核对确为 OceanBase 优化器真实符号(与同名 Python 语言纯属巧合)，并澄清分发分优化器层 `DistAlgo` 与 PX 物理层 `ObPQDistributeMethod` 两层。其五，DXF「v7.5 GA」引用归属已更正，改挂 7.5.0 release notes。其六，曾流传的 TPC-H 提升精确百分比因来源页 429 无法逐字复核，降级为需进一步查证，仅保留「列存 + 向量化显著提升 OLAP」的定性结论。未编造任何 Prometheus metric 名、内部视图名；其中可在源码逐字核对者已升级为事实：OceanBase 的 SQL 执行诊断 GV$ 视图(`GV$OB_SQL_AUDIT` 为执行诊断视图、非合规审计,以及 `GV$SQL_PLAN_MONITOR` / `GV$OB_PLAN_CACHE_PLAN_STAT` / `GV$OB_PLAN_CACHE_PLAN_EXPLAIN` / `GV$OB_SQL_PLAN`)在 `src/share/inner_table/ob_inner_table_schema_constants.h` 中以常量定义,4.2.5 / 4.3.5 均存在；TiDB distsql / Coprocessor 相关 metric(`tidb_distsql_*` 系列)在 `pkg/metrics/distsql.go` 定义。仅 trace event 的具体名称仍标注需进一步查证。本章 TiDB 锁定 8.5.x(对照 7.1 / 7.5 / 8.1 取 DXF 沿革)，OceanBase 取 4.2.5 / 4.3.5 / 4.4.x 三 checkout 交叉确认源码，未发现与锁定版本冲突的事实。

---


# 第 13 章 元数据与控制面

## 13.1 本章核心问题

分布式数据库的"数据面"(SQL 计算、KV/LSM 存储、共识复制)负责读写行、复制日志、执行事务，但它能否正确、高效地工作，取决于一套通常被低估的子系统：控制面(control plane)与元数据(metadata)。本章核心问题不是"谁有一个中心节点"，而是控制信息如何被复制、缓存、租户化与失效恢复。具体拆成四问：

- 谁来记录"哪份数据在哪台机器、谁是 Leader、表结构是什么版本"?
- 谁来分配全局单调递增的时间戳，保证跨节点事务的可串行化基准?
- 谁来决定副本搬迁、Leader 均衡、副本补齐等调度动作?
- 当这套控制面自身发生 Leader 切换、网络分区、缓存过期、时钟抖动时，数据面会受到什么连锁影响?

这是分布式数据库底层的关键，因为控制面承担了单机数据库里"由内核进程独占内存"才能保证的两类强一致语义：全局时序与全局元数据版本。把这两类语义从单机内存搬到分布式集群，必然引入一个"逻辑中心化"的角色。本章的主线，是对比 TiDB 的 PD(Placement Driver)+ TiDB DDL owner 与 OceanBase 的 sys tenant + RootService + GTS 两套控制面。

本章特别避免把 PD、GTS、sys tenant 或 RootService 简写成"单点"。更精确的拆法（属工程推测的分析框架）是五个维度：逻辑中心化是否存在、性能瓶颈风险是否集中、高可用是否依赖多数派、元数据/调度是否依赖它、故障恢复是否需要它恢复服务。下文对 PD/TSO/GTS/sys tenant/RootService 一律按这五维度讨论，不简单贴"单点"标签。

跨章边界：本章不展开 Region 分片如何涌现(详见第 1 章)、共识协议本身(详见第 3 章)、事务两阶段提交与 MVCC(详见第 10、11 章)、DDL 状态机五状态(详见第 14 章)、多租户资源隔离(详见第 17 章)。本章只聚焦"元数据如何被存储、传播、调度，以及控制面故障如何冲击数据面"。

## 13.2 TiDB 的实现

TiDB 的控制面被拆成两个相对独立的角色：负责集群级元数据/时序/调度的 PD，与负责 SQL 层 schema 元数据的 TiDB DDL owner。两者都"逻辑中心化、物理高可用"，但实现机制不同。

### 13.2.1 PD:Region/Store 元数据、TSO、调度

**表 13-1　TSO 关键常量参数**

| 常量/参数 | 值 | 作用 |
|---|---|---|
| maxLogical | 1 << 18(即 262144) | 逻辑位上限，逻辑位用尽强制推进物理时间；每秒可分配上限约 2^18 × 1000 ≈ 262,144,000 |
| defaultLeaderLease | 5 | leader lease，决定预申请时间窗窗长 |
| DefaultTSOSaveInterval | 5 秒 | 由 leader lease 推出的窗长，窗上界持久化到 etcd |
| defaultTSOUpdatePhysicalInterval | 50ms | 物理推进间隔默认值(对应 tso-update-physical-interval) |
| minTSOUpdatePhysicalInterval | 1ms | 物理推进间隔最小值 |
| maxTSOUpdatePhysicalInterval | 10s | 物理推进间隔最大值 |
| UpdateTimestampGuard | 1ms(time.Millisecond) | 时间戳更新保护间隔 |
| MaxSuffixBits | 4 | 后缀位数 |
| jetLagWarningThreshold | 150ms | 时钟偏移告警经验阈值 |

PD 自身内嵌 etcd，用 etcd 的 Raft 保证 PD 集群(推荐至少三个节点、奇数部署)的高可用与元数据强一致。PD 的四类职责是：维护 TiKV 上数据分布的实时元数据、管理集群拓扑、分配全局单调递增时间戳、在节点间均衡负载与容量(PingCAP 官方文档与博客明确列出这四项)。这意味着 PD 是逻辑中心化的控制面，但不是固定单进程高可用单点：PD 集群内部依赖嵌入式 etcd/Raft 维护元数据与 Leader。

数据路径的边界很重要。普通 SQL 读写不会把行数据经过 PD：TiDB Server 通过 region cache 定位 Region，向 TiKV Region Leader 读写。PD 只在缓存 miss、Region split/merge、Leader/peer 变化、Store 状态变化、调度、TSO 获取等控制路径上参与。因此 PD 故障的直接影响通常不是"已有缓存的读写立刻全部停止"，而是新 TSO、Region 元数据刷新、调度、故障恢复和 DDL 协调可能受阻。据此推测，实际影响取决于缓存命中、事务形态和 PD Leader 可恢复时间。

元数据如何进入 PD?TiKV 通过两条心跳通道向 PD 上报状态——Store 心跳携带磁盘总量/可用量/Region 数/读写速率，Region 心跳由 Region Leader 上报自身位置、其他副本位置、下线副本数、读写速率。PD 据此在内存中维护全局 Region 路由表与 Store 状态表，并把核心元数据持久化进内嵌 etcd。

TSO 控制路径是控制面中对时序正确性最敏感、对延迟最敏感的一环。PD Leader 在内存中维护 `timestampOracle` 结构，其 `tsoObject` 含 `physical time.Time`、`logical int64`、`updateTime time.Time` 三个字段(源码 `pkg/tso/tso.go`)。时间戳是 64 位混合逻辑时钟，低 18 位为逻辑时钟、高 46 位为物理时钟；常量 `maxLogical = int64(1 << 18)`(即 262144)是逻辑位上限，逻辑位用尽会强制推进物理时间。由此推出每秒可分配上限约 `2^18 * 1000 ≈ 262,144,000` 个时间戳。源码中 `AllocatorManager` 区分 Global TSO Allocator 与 Local TSO Allocator,`GlobalTSOAllocator` 在发号前检查 leadership，并把 timestamp window 同步到持久存储路径。关键性能设计：PD 预申请一个时间窗，把"窗上界"持久化到 etcd，随后窗内时间戳全部在内存中递增分配，避免每次取号都落盘。这一机制的源码依据是 `pkg/tso/global_allocator.go` 中 `GlobalTSOAllocator` 的 `UpdateTSO()` 与 `GenerateTSO(ctx, count uint32)`。由此可见，TSO 是逻辑中心化的时间服务，但高可用依赖 PD/etcd Leader 选举；跨 AZ 部署时，PD Leader 与 TiDB 之间的网络 RTT 会直接进入事务延迟预算。

关于"预申请时间窗默认值"：早期 TiKV/PD Wiki 与多篇衍生博客称该窗默认 3 秒。经对锁定基准 PD `release-8.5` 源码 `server/config/config.go` 直接核对：窗长由 leader lease 决定——`defaultLeaderLease = int64(5)`,`DefaultTSOSaveInterval = time.Duration(defaultLeaderLease) * time.Second`，即对 TiDB 8.5.x 默认为 5 秒(官方 PD 配置文档亦明确：`lease` 自 v8.5.2 起默认 5 秒、之前为 3 秒)。"3 秒"是 v8.5.2 之前(及旧 Wiki/博客时代)的过时值，对锁定版本属版本错配，故本章按 5 秒书写。物理推进间隔由配置接口 `GetTSOUpdatePhysicalInterval()`(`pkg/tso/config.go`)注入，对应配置项 `tso-update-physical-interval`；经 `server/config/config.go` 核对，常量 `defaultTSOUpdatePhysicalInterval = 50 * time.Millisecond`(默认 50ms，最小 `minTSOUpdatePhysicalInterval = 1ms`，最大 `maxTSOUpdatePhysicalInterval = 10s`)，官方 PD 配置文档同列默认 50ms——此处为可查证的明确默认。源码还含 `UpdateTimestampGuard = time.Millisecond`、`MaxSuffixBits = 4`、`jetLagWarningThreshold = 150 * time.Millisecond`(时钟偏移告警经验阈值)等常量。

调度控制路径在 `pkg/schedule/`，核心是 `Coordinator`(`coordinator.go`)，它组合 checker、scheduler、operator controller、heartbeat stream、Region scatter/splitter，符合"观察状态、生成 operator、通过心跳/流把调度意图下发"的控制循环。其 `patrolRegions()` 周期巡检每个 Region 是否需要调度，巡检间隔默认常量 `defaultPatrolRegionInterval = 10 * time.Millisecond`(`pkg/schedule/config/config.go`)，对应配置项 `patrol-region-interval`。子模块包括 `checker/`(RuleChecker/MergeChecker/JointStateChecker/SplitChecker)、`operator/`、`placement/`(Placement Rules,`RuleManager` 负责规则生命周期与存储加载)、`scatter/`、`schedulers/`、`splitter/`、`hbstream/` 等(仅确认到模块级)。Placement Rules 可按 key range、replica 数、Raft role、位置标签、是否参与 Leader 选举等规则生成调度。PD 生成的调度动作只有三类基本算子：加副本、删副本、转移 Leader，通过 Raft 命令执行。算子并不主动下发，而是搭载在 Region Leader 心跳的响应里返回，且"算子只是对 Region Leader 的建议，可被跳过"——这是理解调度抖动可控性的关键。

Keyspace 是 TiDB 在多 keyspace 方向的元数据分区机制之一。PD 在 `pkg/keyspace/` 管理 keyspace,bootstrap 时对 `DefaultKeyspaceID = uint32(0)`(常量定义在 `pkg/mcs/utils/constant/constant.go`,`DefaultKeyspaceName = "DEFAULT"`、`NullKeyspaceID = uint32(0xFFFFFFFF)`)创建默认 keyspace 并拆分初始 Region。`pkg/keyspace/tso_keyspace_group.go` 中 `GroupManager` 维护 keyspace group 与 keyspace 的映射、TSO service 地址发现、split/merge 状态与存储事务。从 v8.0.0 起 PD 支持微服务模式，把 TSO 分配与调度拆成两个可独立部署的微服务(`pkg/tso/keyspace_group_manager.go` 支撑 TSO 按 keyspace group 分配)；配多副本时微服务自动进入主备容错模式。此外，Active Followers(允许 follower 处理元数据读写)在 7.6 实验、8.5 GA，官方内部测试称 5 节点下 GetRegion 处理提升约 4.5 倍。公开文档对 TiDB Cloud 内部隔离机制披露并不充分，其底层实现尚不确定；因此本章只把 Keyspace/Keyspace Group 写为源码可见的 PD 元数据与 TSO 分组机制，不推断 Cloud 多租户底座内部实现。

### 13.2.2 TiDB DDL owner:schema 版本与元数据传播

与 PD 管理的集群级元数据相对，SQL 层 schema 元数据由 TiDB 自身管理，而非 PD。TiDB 使用在线、异步 DDL,DDL owner 负责执行集群内 DDL 任务，任一时刻只有一个 TiDB node 被选为 owner。TiDB 通过 etcd 选举出唯一 DDL owner，选举路径常量 `DDLOwnerKey = "/tidb/ddl/fg/owner"`(`pkg/ddl/ddl.go`),`pkg/ddl/owner_mgr.go` 通过 etcd client 创建 `owner.NewOwnerManager(..., DDLOwnerKey)`,owner manager prompt 为 `"ddl"`，并有 `ddlSchemaVersionKeyLock = "/tidb/ddl/schema_version_lock"`,owner 判定通过 `ownerManager.IsOwner()`。这表明 DDL owner 是通过 etcd 协调的逻辑中心化角色，而不是数据 Region Leader;owner 宕机后其他节点会被重新选为新 owner 继续推进 DDL，因此不是物理单点。

schema 版本传播路径由三条 etcd 路径常量(`pkg/ddl/util/util.go`)构成骨架：`DDLGlobalSchemaVersion = "/tidb/ddl/global_schema_version"`(最新版本)、`DDLAllSchemaVersions = "/tidb/ddl/all_schema_versions"`(各 server 当前版本)、`DDLAllSchemaVersionsByJob = "/tidb/ddl/all_schema_by_job_versions"`。owner 推进一个 schema 版本后，通过 etcd Watch 通知各 TiDB 实例，owner 需等待所有在线实例确认新版本后才推进下一状态。

schema lease 与两版本约束是 TiDB 在元数据异步传播下保护一致性的核心。每个 TiDB 实例由 `Domain` 持有 `schemaLease time.Duration`(`pkg/domain/domain.go`)，默认 45 秒；`loadSchemaInLoop` 以 `time.NewTicker(do.schemaLease / 2)` 周期 Reload。这里"lease/2 Reload"的设计与 Google F1 在线异步 schema change 的思想一致(源码注释 "Use lease/2 here as recommend by paper")；二者是否存在直接渊源属工程推测，本章只就思想层面对照。一致性由 `SchemaValidator`(`pkg/domain/schema_validator.go`)保障，其结构含 `lease`、`latestSchemaVer`、`latestSchemaExpire`、`deltaSchemaInfos`(变更历史队列);`pkg/domain/` 还维护 `InfoCache`、InfoSyncer 等，`schema_checker.go` 在事务提交前校验 schema version，可返回 `ErrInfoSchemaChanged` 或 `ErrInfoSchemaExpired`。这套机制保证任意时刻全集群至多两个相邻 schema 版本并存，使得并发 DML 始终安全。

从控制面/数据面边界看，TiDB 的优点是组件职责清晰：TiDB Server 可水平扩展，TiKV 负责数据复制，PD 负责全局元数据、调度和 TSO。代价是控制面有明确的低延迟依赖，尤其是在写事务取 TSO、Region 元数据频繁变化、大规模调度或 DDL owner 切换时；此判断属基于机制的工程推测。

源码模块与文档：
- PD:tikv/pd `release-8.5` @ `6dce4a68e3e9`,`pkg/tso/`、`pkg/schedule/`、`pkg/keyspace/`、`server/config/config.go`(已确认到文件级/常量值)。
- TiDB:pingcap/tidb `release-8.5` @ `67b4876bd57b`,`pkg/ddl/`、`pkg/domain/`(已确认到文件级/常量值)。 〔文献[1-7,13-14,17-18]〕

## 13.3 OceanBase 的实现

与 TiDB 的两段式划分不同，OceanBase 把控制面收敛进一个特殊租户——sys tenant，并由其中的 RootService(RS)聚合控制面职责，时序服务 GTS 则按租户隔离。OceanBase 的控制面与数据面同驻 OBServer 进程，但通过 tenant、RootService、schema service、GTS/SCN 等机制分层——这与 TiDB"PD 集群级 + DDL owner SQL 级"的划分恰成对照。

### 13.3.1 sys tenant 与 RootService

![[f13_1.svg]]

**图 13-1　OceanBase 资源层级与内部表 / GTS 映射**

OceanBase 支持多租户，每个 Tenant 相当于独立数据库实例，CPU、内存、I/O 等资源在租户之间隔离；集群初始化后默认存在 sys tenant，该系统租户存储集群元数据并以 MySQL 兼容模式运行。RootService 不是一个独立进程，而是一组运行在"`__all_core_table` 表 Leader 所在 OBServer"上的服务；官方架构文档说明 RootService 运行在某个 OBServer 节点上，当该 OBServer 失败时会选举另一个 OBServer 运行 RootService。当该副本由 Leader 变为 Follower 时，该副本上的 RS 即停止，从而保证集群内同时只有一个 RS 在运行。`__all_core_table` 是集群启动时生成的第一张表，用 Paxos 实现高可用与 Leader 选举(副本数可配)。

源码上，`class ObRootService`(oceanbase/oceanbase `v4.2.5_CE` @ `e7c676806fda`,`src/rootserver/ob_root_service.h`)聚合了控制面核心成员：`ObMultiVersionSchemaService *schema_service_`、`ObDDLService ddl_service_`、`ObUnitManager unit_manager_`、`ObRootBalancer root_balancer_`，并提供 `get_schema_service()/get_unit_mgr()/get_root_balancer()/get_ddl_service()` 访问器；还含 server manager、zone manager、灾备 task manager 与 LS table operator 等依赖。`src/rootserver/README` 列出 RootBalancer、LeaderCoordinator、HeartbeatChecker、异步 task queue、major freeze、daily merge 等控制任务。RS 由 sys tenant 内 Paxos 选出的角色担任，聚合 schema、DDL、Unit/Resource Pool、负载均衡职责，并管理 region/zone/OBServer/resource pool/resource unit 等元数据，决定 Unit 在 server 上的分布，通过自动复制和迁移让 Log Stream/LS 副本分布符合 Locality，并响应 DDL 生成新 schema。RS 下还含完整的灾备调度模块 `ob_disaster_recovery_worker/task_mgr/task_executor`(副本补齐、迁移调度，仅确认到模块级)。

租户元数据路径：Unit/Resource Pool 由 `class ObUnitManager`(`src/rootserver/ob_unit_manager.h`)管理；Locality 工具在 `ob_locality_util.h`;Primary Zone 变更检查在 `ob_alter_primary_zone_checker.*` 与 `ob_balance_ls_primary_zone.*`;Locality 变更收尾在 `ob_alter_locality_finish_checker.*`。资源层级为：UNIT(最小资源单位)→ Resource Pool → Tenant；创建租户时构造 `ObTenantSchema`，其中携带 locality 与 primary_zone，并因"跨租户不能同事务"而分三个事务完成(分配 Resource Pool + 建系统表分区 → 建内部用户/库等元数据 → 状态 CREATING→NORMAL)。Primary Zone 决定主副本分布，调整后 OceanBase 在线发起主副本切换。

sys tenant 的高可用不能简单写成"一个管理库"。结合 OceanBase 论文中 timestamp Paxos group 的描述以及架构文档对 Paxos replica 的说明，合理写法是：sys tenant 和相关内部表依赖 OceanBase 的 Paxos/LS 复制获得高可用，而不是依赖某个固定 OBServer。两个关键内部表名可锁定：集群首表是 `__all_core_table`(`OB_ALL_CORE_TABLE_TID = 1`,`src/share/inner_table/ob_inner_table_schema_constants.h`)，sys tenant 的 GTS provider 与其 Leader 相关；用户租户的 GTS provider 则与 `__all_dummy`(`OB_ALL_DUMMY_TID = 135`，同文件)的 Leader 副本相关——官方 GTS 文档明确"用户租户 GTS server 依赖 `__all_dummy` 表的 leader 副本"，源码亦定义该表常量。至于 CE 版本中其余全部内部表清单、各 LS 与 GTS provider 在异常拓扑下的完整切换细节，公开资料未逐一展开，仍不确定，此处保留待核实。

### 13.3.2 多版本 Schema Service 与 DDL epoch

schema service 是 OceanBase 元数据传播的关键。`class ObMultiVersionSchemaService : public ObServerSchemaService`(`src/share/schema/ob_multi_version_schema_service.h`)提供按 `(tenant_schema_version, sys_schema_version)` 取 schema guard 的接口；`ObServerSchemaService` 提供 tenant schema version、inner table schema version、schema refresh、publish schema、多版本 schema 初始化等接口，且 schema key 中包含 tenant_id。源码注释明确：schema 拆分后，用户租户的系统表 schema 引用自系统租户，取 guard 时除租户 schema_version 外还须额外传入 sys tenant 的 schema_version——这是 OB 元数据强依赖 sys tenant 的直接证据。由此可写：OceanBase 的 schema 元数据天然带 Tenant 维度，schema cache/refresh 不是单个全局 schema map，而是围绕 tenant、多版本 schema 与 inner table 持久元数据展开。`RefreshSchemaMode` 含 `NORMAL/FORCE_FALLBACK/FORCE_LAZY`;`ObSchemaVersionUpdater` 保证版本单调不减，版本退化返回 `OB_OLD_SCHEMA_VERSION`。

OB 防 DDL 脑裂的机制是 DDL epoch:`struct ObDDLEpoch { uint64_t tenant_id_; int64_t ddl_epoch_; }`(`src/share/schema/ob_ddl_epoch.h`),`ObDDLEpochMgr` 注释为 "use ddl_epoch to promise only ddl_service master execute ddl trans"，方法含 `get_ddl_epoch / promote_ddl_epoch(tenant_id, wait_us, ddl_epoch_ret) / check_and_lock_ddl_epoch(...)`。即在 DDL 事务里比对并锁定 epoch，保证仅当前 DDL service master 能执行 DDL 事务，可对照 TiDB 的 DDL owner。

### 13.3.3 GTS：租户级时序服务

GTS/SCN 是 OceanBase 事务版本控制的关键控制路径。OceanBase 的 GTS 按租户隔离：每租户每节点有一个 GTS 客户端，每租户只有一个 GTS 服务端；用户租户的 GTS provider 与 tenant-level internal table leader 相关，默认三副本，高可用能力与普通表一致；system tenant 的 provider 与 `__all_core_table` leader 相关。架构文档在更高层次上说 GTS 在 Tenant 内生成连续递增时间戳，并通过多副本保证可用性，其底层机制与复制层的 Log Stream 副本同步一致。OceanBase 论文从协议实现角度描述 timestamp Paxos group，节点周期性向 timestamp Paxos leader 获取 timestamp。这些来源共同支持"GTS 是逻辑中心化的时间序服务，但 HA 依赖 Paxos/LS 多副本与 Leader 选举"的表述，不支持把 GTS 写成固定单点。

源码上 `class ObGtsSource`(`src/storage/tx/ob_gts_source.h`)持有 `tenant_id_`、`ObGTSLocalCache gts_local_cache_`、`queue_[TOTAL_GTS_QUEUE_COUNT]`(`GET_GTS_QUEUE_COUNT=1`、`WAIT_GTS_QUEUE_COUNT=1`、`TOTAL=2`)、`gts_cache_leader_`，构造默认 `log_interval_=3s`、`refresh_location_interval_=100ms`；方法含 `get_gts/wait_gts_elapse/refresh_gts/get_gts_leader_`,`is_external_consistent()` 恒返回 true。性能优化：GTS 请求批量化以减少请求数；单机事务自 4.0 起无需访问 GTS，提交版本号虽要取 GTS 满足外部一致性，但这是接口调用而非真正 RPC，效率高。

ODP/OBProxy 与控制面的关系主要在路由层。ODP/OBProxy 接收 SQL 后转发到最优 OBServer，核心能力包括连接管理、最优路由和高性能转发；cluster routing 需要获得 cluster name 到 RS list 的映射，RS list 通常包含 RootService 所在服务器，随后 ODP 通过 `proxyro@sys` 登录并读取集群服务器信息。因此 ODP 是访问层和路由缓存层，不是 RootService 本身；当 RootService/RS list 或路由信息异常时，连接建立与路由刷新会受影响。据此推测，已有连接是否受影响取决于缓存、会话状态与路由失效类型。

源码模块与文档：oceanbase/oceanbase TP `v4.2.5_CE` @ `e7c676806fda`,`src/rootserver/`、`src/share/schema/`、`src/storage/tx/`(已确认到文件级/类成员)。上述 `ob_root_service.h`、`ob_gts_source.h` 在 `4.3.5`(`b28b9bb12f3b`)、`4.4.x`(`d4bef8d29a4c`)checkout 中均存在。经逐版本 diff 核对:本章引用的核心成员在三个版本间保持一致——`ObRootService` 聚合的 `schema_service_`、`ddl_service_`、`unit_manager_`、`root_balancer_` 四个成员及其 `get_schema_service()/get_unit_mgr()/get_root_balancer()/get_ddl_service()` 访问器在 4.2.5/4.3.5/4.4.x 中逐字相同(仅行号偏移);`ObGtsSource` 的 `tenant_id_`、`gts_local_cache_`、`queue_[TOTAL_GTS_QUEUE_COUNT]`、常量 `GET_GTS_QUEUE_COUNT=1`/`WAIT_GTS_QUEUE_COUNT=1`、构造默认 `log_interval_=3s`/`refresh_location_interval_=100ms`、`is_external_consistent()` 恒返回 true、`gts_cache_leader_` 在三版本中亦逐字一致(4.4.x 仅新增 `get_real_tenant_id_()` 等辅助方法,不改变上述成员)。故本章对这些成员的引用对 4.2.x–4.4.x 均成立。 〔文献[8-9,11-12,15-16,19]〕

## 13.4 核心差异对比

**表 13-2　元数据与控制面:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB(PD + DDL owner) | OceanBase(sys tenant + RS + GTS) | 影响 |
|---|---|---|---|
| 控制面承载体 | 独立 PD 进程集群(内嵌 etcd)+ TiDB 内 DDL owner | sys tenant 内 RootService(普通 OBServer 上)+ schema service + GTS | TiDB 控制面与数据面进程分离、边界清晰；OB 控制面寄生在 OBServer 之上，管理路径更内聚但更依赖租户/内部表语义 |
| 元数据高可用机制 | etcd Raft(对称多副本)| Paxos(`__all_core_table` Leader / LS 多副本)| 都是多副本共识；TiDB 用 Raft、OB 用 Paxos，工程取舍非"谁更先进"；两者都不是固定单进程元数据源 |
| 时序服务粒度 | 集群级(默认 keyspace 单一 TSO)| 租户级 GTS(每租户独立)| OB 多租户下时序天然隔离；TiDB 单 TSO 是全局共享点 |
| 时序持久化与延迟敏感点 | 内存递增 + 时间窗(默认 5s，绑定 leader lease)持久化到 etcd；写事务对 PD TSO RTT 敏感 | GTS Leader 维护本地 cache,Paxos 复制；对租户内 GTS provider / LS leader 布局敏感 | TiDB 重选 PD 后靠 etcd 窗上界防回退；OB 靠租户 Paxos |
| schema 唯一写者保证 | DDL owner(etcd 选举)| DDL epoch(`promote/check_and_lock`)| 机制不同但目标一致：防双 master 双写 |
| schema 版本传播 | etcd Watch + 三条版本路径，owner 等各实例确认；保证至多两相邻版本并存 | 多版本 schema service，需 sys + 租户双版本 | TiDB 强调 SQL 层异步 schema 变更；OB 强调 RootService/schema service 一体化并依赖 sys tenant 版本 |
| schema 时效约束 | schema lease(默认 45s),`lease/2` Reload | RefreshSchemaMode + 版本单调 | TiDB 失联超 lease 则禁 DML(强约束)|
| 调度对象与职责 | PD Coordinator(巡检 10ms 默认)+ Placement Rules，以 KV Region/Store 为中心 | RS root_balancer + disaster recovery worker，以 Tenant/Unit/Resource Pool/Locality/Primary Zone/LS 为中心 | 都做副本补齐/Leader 均衡/Locality，但调度对象层级不同 |
| 路由感知 | TiDB/TiKV 直连 PD 取路由(region cache)| ODP 从 sys tenant 取分区/zone/租户信息 | OB 多一层 ODP 缓存，有缓存过期窗口 |

在总体差异之上，再用一张"控制面非单点五维度"对照表，正面回应"不能简单写单点"这一判断。

**表 13-3　元数据与控制面:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB PD / TSO / DDL owner | OceanBase sys tenant / RS / GTS |
|---|---|---|
| 逻辑中心化 | 是：单一 TSO allocator(`GlobalTSOAllocator`)、单一 DDL owner 都有当前 leader/owner 角色，客户端与 TiDB Server 需知道当前服务者 | 是：RS 聚合控制面、每租户单一 GTS Leader、LS leader 都有当前 leader/provider 角色，ODP/OBServer 需获得路由与 leader 信息 |
| 性能瓶颈风险 | 高频小事务/跨地域取号、Region heartbeat 风暴、大规模 operator、DDL owner 积压会集中到 PD/owner 路径，属工程推测(可 PARALLEL 缓解) | Tenant GTS、RS 任务队列、schema refresh、ODP 路由刷新、租户资源紧张会集中到内部控制路径，亦属推测(批量化 + 单机事务免 GTS 缓解) |
| 高可用单点 | 否：etcd Raft 多副本，Leader 宕机自动重选，多数派可用时可重新选 leader | 否：RS/GTS 角色由 Paxos 选举重建，RS 可迁移、GTS/内部表依赖 Paxos/LS 多副本 |
| 元数据/调度依赖 | 强：Region 元数据刷新、Placement Rules、Store 状态、DDL owner/schema version 依赖控制面恢复 | 强：用户租户 schema 需 sys tenant 版本，Unit/Resource Pool/Locality/Primary Zone/tenant 元数据与 LS 副本布局依赖 RS/sys tenant/schema service |
| 故障恢复依赖 | TSO 窗上界 etcd 持久化保证重选不回退；Store 下线、Region 补副本、leader balance、DDL 接续需 PD/owner 恢复 | DDL epoch 防双 master;OBServer 下线、LS 迁移/补副本、RS 接管、租户资源变更与 schema 推进需内部控制面恢复 |

这个矩阵也解释了两类数据库"正常读写还能不能继续"为什么不能一刀切。就 TiDB 而言（以下属工程推测）：如果 region cache 仍有效、事务不需要新 TSO、目标 TiKV Region Leader 没有变化，某些读路径可能短时间不触发 PD；但新写事务、悲观锁等待后的提交、强一致读写、DDL、Region cache miss 和副本恢复都会把 PD 可用性重新拉回关键路径。OceanBase 一侧同样按推测理解：如果已有连接、ODP 路由缓存、OBServer 本地 schema cache 与目标 LS Leader 都有效，部分 SQL 也可能不立即触发 RS；但新建连接、路由刷新、DDL、GTS/SCN 获取、LS Leader 变化、租户资源变更和 RS 恢复任务会更快暴露控制面异常。

关于"逻辑中心化即单点"的纠偏：集群内角色切换由共识选举自动收敛，真正风险在 TSO/GTS 的性能瓶颈与故障恢复依赖，而非"单副本失效即全挂"(与第 1 章结论一致)。OceanBase 官方公布的 RPO=0、RTO<8s(早期 RTO<30s)，其归属须谨慎：经多源核对(en.oceanbase.com 高可用与 RPO/RTO 文档、V4.0 Overview、阿里云 ApsaraDB for OceanBase 文档)，这些数值是 OceanBase V4.x 在 LS/Paxos 多副本、少数派故障或多数派完整场景下的集群级/数据分区级容灾切换指标(满足金融"六级容灾"，需同城 3 IDC 等多副本拓扑)，描述的是"数据主副本失败→重新选主"的数据服务恢复能力。官方对 RootService 角色切换只有定性描述(`__all_core_table` Leader 漂移后在另一 OBServer 自动重建)，并未公布独立的 RTO/RPO 数字；且把"RPO=0(事务数据零丢失)"挂到管理元数据的 RS 上属指标范畴错置。故本章不把 RTO<8s 归为 RootService 指标，亦不外推到单副本 shared-storage 或跨云场景，仅作为集群数据容灾的通用官方数字引用。 〔文献[10]〕

## 13.5 正常路径图

图 13-2 先展示 TiDB 一次分布式读写事务对控制面的正常依赖路径(取 start_ts → 查路由/校验 schema → 访问 TiKV → DDL owner 推进 schema)。

![[f13_2.svg]]

**图 13-2　元数据与控制面正常读写/调度路径**

与之相对，OceanBase 的正常路径经由 ODP 起步(ODP → 路由到 OBServer → 租户 GTS 取版本 → LS Leader 读写 → RS 维护元数据)。

![[f13_3.svg]]

**图 13-3　元数据与控制面正常读写/调度路径**

两条正常路径对照来看：TiDB 的 SQL 数据流避开 PD，但事务时间戳、Region 元数据刷新、调度和 DDL owner 都会走控制面；OceanBase 的 SQL 流经过 ODP 到 OBServer，Tenant 内 GTS/SCN、LS Leader、schema cache 与 RootService 共同决定请求走本地还是远程路径。

## 13.6 故障/异常路径图

下面用两张图刻画典型控制面故障如何冲击数据面。

故障 A:TiDB 实例与 etcd 失联 → schema lease 过期 → 禁 DML(源码级证据)。

![[f13_4.svg]]

**图 13-4　元数据与控制面故障/异常路径**

该路径有源码注释直接支撑(`domain.go` 容错语义)：丢失 etcd session 后实例主动禁 DML，以维护"至多两相邻 schema 版本"的全局不变量，这是控制面故障被显式约束为"局部不可用而非全局错误"的设计。

故障 B:PD Leader 切换 / TSO 抖动 / DDL owner 切换 / OBServer 主副本切换 / GTS provider 切换。

![[f13_5.svg]]

**图 13-5　元数据与控制面故障/异常路径**

关键点：TiDB 侧 PD 重选后靠 etcd 持久化的窗上界保证时间戳不回退(`global_allocator.go`)，代价是切换窗口内取号短暂停顿；DDL owner 切换由 etcd 重选，旧 schema 事务由 SchemaValidator 保护不丢一致性；ODP 侧路由缓存过期靠"先报错再拉新主重试"收敛，代价是少量请求错误与重试放大；OB 侧 GTS/LS leader 切换位于事务版本路径，表现为提交等待或延迟抖动，且 GTS/SCN 不回退。这些都不是"全局不可用"，但都会在数据面表现为延迟尖刺。需强调：RootService 失败不等于所有 OBServer 停止服务，但资源管理、负载均衡、DDL/schema 管理和恢复任务依赖 RS 接管。

## 13.7 性能、可靠性、运维影响

延迟。TiDB 每个写事务通常需向 PD 取一次 TSO(一次 RTT)。跨地域部署时，TiDB 与 PD 跨区会让"取 TSO + 读 index"成为两次 RTT，官方给出两类缓解：把 `tidb_tso_client_rpc_mode` 切到 `PARALLEL`/`PARALLEL-FAST`，或用代理把 `get_ts_and_read_index` 合并为一次 RTT(官方博客称多地域读延迟与流量降约 50%)。注意 PARALLEL/PARALLEL-FAST 会缩短 batching 等待但提高 TSO RPC 数量，并非无代价优化，PD workload 和网络抖动需一起观察。OceanBase 因 GTS 是租户级且与表分区 Leader 常处同 region，本地 region 内事务取号即 region 内通信；单机事务自 4.0 起免 GTS RPC，进一步压低延迟。CE 版本的具体调参名公开文档未给出，尚不确定。

吞吐。TiDB 单 TSO 每秒上限约 2.62 亿时间戳(`maxLogical=1<<18`)，对绝大多数负载非瓶颈；但高频小事务会让 TSO 取号 RPC 成为热点。OceanBase GTS 用批量取号摊薄请求，且各租户 GTS 互不影响，多租户混部下吞吐隔离更好。

调度震荡。TiDB 的 `balance-region`/`hot-region-scheduler`、store limit、Placement Rules 配置不当可能导致 operator 过多、迁移频繁或热点迁移不足。OceanBase 的 Locality、Primary Zone、Unit/Resource Pool 变更会触发副本/leader/资源重排，若规划不当会影响跨 server 访问与分布式事务比例。建租户文档说明 Unit config、Resource Pool、Tenant 是创建顺序，`RESOURCE_POOL_LIST`、Primary Zone、Locality 是建租户的重要属性，并提醒多 Unit 租户需提前设计业务数据分布以避免分布式事务影响性能。这说明 OceanBase 控制面的一个核心代价是：资源与副本布局是租户级工程决策，不只是后台自动调度。

可用性。两者控制面都靠共识多副本——PD 用 etcd Raft(推荐 3 节点，1 节点宕机仍可用),OB 用 Paxos(RS/GTS 角色随对应特殊表 Leader 由 Paxos 选举自动重建)。可靠性影响还要区分"控制面停摆"和"数据面已失去多数派":TiDB 中 PD 多数派不可用时，调度和新 TSO 受影响，但 TiKV Region 自身是否还能读写取决于对应 Raft Group 多数派是否存在，两者相关但不是同一个多数派；OceanBase 中 RootService 所在 OBServer 故障和某个业务 LS 丢多数派也不是同一类事故，前者影响集群管理与 schema/资源/恢复任务接续，后者直接影响该 LS 上的数据读写可用性，GTS provider 切换则位于事务版本路径，表现为提交等待、重试或延迟抖动。关于 OceanBase 公布的 RPO=0、RTO<8s 的口径与归属限定，见 §13.4 与 §13.13。差异在于：TiDB 控制面是独立进程，可单独扩缩容与定位；OB 控制面寄生在 OBServer 上，RS 随 `__all_core_table` Leader 漂移。

扩展性。TiDB 从 v8.0.0 起可把 TSO/调度拆为 PD 微服务独立扩展，Active Followers(8.5 GA)让 follower 分担元数据读，keyspace group 让 TSO 可按 keyspace 切分。OB 通过增加租户/Unit 横向扩展，GTS 随租户天然分片。

故障恢复与运维。TiDB 关键不变量是"TSO 不回退"(窗上界 etcd 持久化)与"至多两相邻 schema 版本";OB 关键不变量是"单 DDL master"(ddl_epoch)与"sys+租户双 schema 版本一致"。运维上，TiDB 更适合按组件建 runbook：先判断 PD Leader/etcd 多数派，再看 TSO 延迟和 Region heartbeat，再看 TiDB domain/DDL owner，再看 TiKV Region Leader，组件边界清晰、监控指标独立。OceanBase 更适合按 Tenant 与系统租户分层：先判断 sys tenant 与 RootService 状态，再看目标 Tenant 资源、schema version、GTS/SCN、LS Leader 与 ODP 路由；控制面隐藏在 sys tenant 内，排查需进 sys tenant 查内部视图，ODP 还多一层路由缓存需诊断。两种 runbook 都不应只写"重启中心节点"：据工程推测，错误重启可能放大 leader 震荡或延长元数据恢复。

## 13.8 反例与代价

第一类反例是"把控制面拆出去就一定简单"。TiDB 的 PD 独立部署让职责清晰，但写事务 TSO、Region cache miss、调度和 DDL owner 都会把控制面 RTT 或可用性暴露到业务路径上。跨地域单 TSO 会放大延迟，虽有 PARALLEL 模式与 follower read 缓解，但强一致读仍绕不开取号——这是"全局时序换强一致"的固有代价，而非实现缺陷。大规模集群中，PD 元数据量与调度事件增多会带来扩展压力；PingCAP 2025 年 PD metadata scaling 博客把 PD 的 scalability bottleneck、stability risk、storage limitation 作为演进动因；据官方资料暗示，这属官方博客层面的演进信号，并不能反推 8.5.x 已解决所有瓶颈。

第二类反例是"内聚控制面就没有外部依赖"。OceanBase 把 RootService、schema service、GTS 与 OBServer/sys tenant 体系放在同一集群内，减少独立组件数量，但控制面仍依赖内部 Paxos/LS、sys tenant 资源和 RS 接管。RS 随 `__all_core_table` Leader 漂移，sys tenant 自身压力或异常会波及整个集群的元数据服务，sys tenant 必须谨慎保护。据此推测，如果 sys tenant 资源、内部表 Leader、RS 所在节点或 ODP 路由信息异常，故障表象可能跨越 SQL、路由、schema 和资源调度多个层面，排障路径并不比独立 PD 更短。

第三类反例是"TSO/GTS 是纯理论时间戳，不影响性能"。TiDB 官方文档已把 TSO RPC duration、PD TSO allocation 等作为可观察对象；OceanBase GTS 文档也列出 snapshot retrieval 与 commit version retrieval 优化。时间服务既是正确性机制，也是高 QPS 下的排队和网络延迟来源。

第四类代价与具体旋钮。TiDB 的 schema lease 是一对必须权衡的旋钮：实例与 PD/etcd 失联超过一个 lease(默认 45s)，该实例主动禁 DML 并报 "Information schema is out of date"，在网络抖动或 PD 压力大时会放大为局部不可用；调小 lease 加速 DDL 收敛但增加 etcd 轮询压力。OceanBase 的多版本 schema 双版本依赖也是一类隐性代价：用户租户 schema guard 必须额外带 sys tenant 的 schema_version,sys tenant schema 刷新滞后会牵连用户租户；ODP 弱一致读路由(follower_first)在 follower 全不可用时回退到 Leader，也可能带来路由不确定性。

第五类是版本公开度不同。TiDB 自建 classic 的 PD、DDL/domain 源码和文档较容易交叉验证；OceanBase V4.x 的 RootService/schema service 源码可见，但某些 GTS/sys tenant 细节公开文档较旧，相关细节尚不确定，CE 对应部分需继续用源码或官方设计文档核查。TiDB Cloud 与 OceanBase Cloud 的多租户内部隔离机制均不应在本章做实现层断言。

取舍对照：TiDB 把控制面做成独立、可观测、可单独扩缩的组件，代价是多一套进程与跨区 TSO 延迟；OceanBase 把控制面收进 sys tenant、时序按租户隔离，代价是控制面与数据面耦合、ODP 多一层缓存。两条路线没有"更优"，只有"更适配何种部署形态"。

## 13.9 测试开发视角的验证点

可测试的功能场景。TiDB 侧：TSO 单调性(并发取号验证 `start_ts` 严格递增，跨 PD Leader 切换后不回退);schema 两版本约束(并发 DDL + DML，验证任意实例落后不超过一个 schema 版本);Region leader 转移与 Region split/merge 后 region cache 是否正确刷新；Placement Rules 修改后副本分布是否收敛；DDL owner 切换期间 `ADMIN SHOW DDL` 是否出现 owner 变更但 DDL job 不丢失；长事务遇到 schema version 变化时是否返回可预期错误或重试。OceanBase 侧：RootService 所在 OBServer 故障后 RS 是否迁移；DDL/schema 变更期间不同 OBServer 上 schema refresh 是否一致；GTS provider/LS leader 切换时提交版本是否单调不回退；Primary Zone/Locality 调整后 leader 与副本布局是否按预期收敛；ODP 在 RS list 变更、RS 迁移、OBServer 下线后是否刷新路由；对一个租户压测取号验证不影响其他租户(GTS 租户隔离)。

可注入的失效模式。TiDB 侧：杀 PD Leader、隔离少数派 PD、提高 TiDB 到 PD 的 RTT、制造 Region heartbeat 延迟、在 DDL backfill 中重启 DDL owner、制造 TiKV Store 下线/恢复、切断 TiDB↔etcd(观察 lease 过期后是否正确禁 DML 并报 "Information schema is out of date"、恢复后是否自动放行)。OceanBase 侧：停止 RootService 所在 OBServer、隔离少数副本、让 ODP 无法访问 RS list、在租户资源紧张时触发 DDL、修改 Primary Zone 前后观察跨 server 访问比例。两侧均可把系统时钟前推，观察 jetLag 告警(`jetLagWarningThreshold = 150ms`)与 schema 行为。

关键压测指标：事务 p95/p99、TSO/GTS 取号 QPS 与耗时、PD CPU/raft apply 延迟、Region heartbeat 延迟、operator 创建/完成速率、schema 版本传播延迟、DDL job 阶段耗时、副本补齐恢复时间。

关键观测指标(查证后，不编造)。TiDB/PD:Grafana 中 PD TSO RPC Duration(PD Client 段)、PD server TSO handle duration(PD 段)、`patrol-region-interval` 巡检频率、`balance-region/balance-leader` 调度诊断(v6.3.0 起)；相关错误信息 `[domain:8027]Information schema is out of date`；可调参数 `schemaLease`(默认 45s)、`tidb_tso_client_rpc_mode`、store limit。OceanBase：官方文档确认 `DBA_OB_TENANTS`、`DBA_OB_RESOURCE_POOLS`、`DBA_OB_UNITS`、`GV$OB_SERVERS` 等视图存在，可按 Tenant 和 sys tenant 分层排障；GTS/RS 排障还可用三个源码确认的内部视图：`V$OB_TIMESTAMP_SERVICE`(GTS/时间戳服务诊断视图，租户内可查,列含 `TENANT_ID/TS_TYPE/TS_VALUE/SVR_IP/SVR_PORT`，底层 `__all_virtual_timestamp_service` 还含 `service_role`/`role`/`service_epoch`，视图定义按 `ROLE='LEADER'` 且最大 `SERVICE_EPOCH` 过滤，即定位当前 GTS leader)、`GV$OB_LOG_STAT`(LS/日志流副本与角色状态)、`DBA_OB_ROOTSERVICE_EVENT_HISTORY`(RootService 事件历史);三者均在 `src/share/inner_table/ob_inner_table_schema_constants.h` 定义且在 4.2.5/4.3.5/4.4.x 三版本中均存在。ODP 诊断日志 `obproxy_diagnosis.log`(按 `trace_type` 定位断连节点)；更细的 Prometheus metric 名、等待事件与 GTS/RS 相关内部函数名仍需进一步查证(避免编造)。

测试结论要分层验收：控制面 leader 切换测试只能证明"控制面角色可恢复"，不能证明业务 LS/Region 在多数派丢失时仍可写；反过来，单个 Region/LS leader 切换成功，也不能证明 PD 或 RootService 在大规模拓扑变化时不会积压调度任务。据此推测，压测报告应分别记录控制面恢复时间、数据面可写中断时间、元数据缓存失效时间和路由刷新时间，避免把不同故障域混成一个可用性数字。

## 13.10 容易误解点

1. "PD/RS/GTS 是单点，挂了就全挂"——错。它们是逻辑中心化但物理高可用：PD 是 etcd Raft 多副本、Leader 宕机自动重选；RS 角色随 `__all_core_table` Leader 由 Paxos 选举，失败后由其他 OBServer 接管；GTS Leader 由租户 Paxos 选举。真正需要写清的是 PD Leader 故障、多数派不可用、TSO/GTS RTT、Region cache miss、调度恢复各自的影响范围，以及性能瓶颈与故障恢复依赖，而不是简单写"单副本失效即全挂"(五维度见 §13.4)。

2. "TiDB 的 PD 管所有元数据，包括表结构"——错。PD 管的是集群级元数据(Region/Store/TSO/调度/keyspace),SQL 层 schema 由 TiDB 自身的 DDL owner + schema lease 管理，存于 etcd 与 TiKV，不在 PD 的调度逻辑里。OceanBase 才是把 schema 服务聚合进 RS/sys tenant。

3. "TiDB 单 TSO 一定是吞吐瓶颈"——不准确。单 TSO 每秒上限约 2.62 亿时间戳，对多数负载非瓶颈；真正痛点是跨地域延迟(取号 RTT)与高频小事务的取号 RPC 热点，官方分别用 PARALLEL 模式与合并 RTT 缓解。把"逻辑中心化"等同于"吞吐瓶颈"是过度简化。

4. "元数据更新是同步广播到所有 SQL 节点"——过度简化。TiDB 使用异步在线 DDL、schema version、metadata lock 与 schema validator;OceanBase 使用 tenant 维度 schema service 和多版本 schema。两者都用版本化和校验处理传播延迟，而不是假定瞬时一致缓存。

## 13.11 本章结论

1. TiDB 控制面是两段式：PD(独立进程 + 内嵌 etcd,etcd Raft)管 Region/Store 元数据、集群级单一 TSO(`GlobalTSOAllocator`,`maxLogical=1<<18`，窗上界持久化到 etcd 防回退，默认窗长 5s 绑定 leader lease)、调度与 keyspace;TiDB DDL owner(etcd 选举)+ schema lease(默认 45s,`lease/2` Reload)管 SQL schema 版本传播，保证至多两相邻版本并存。

2. OceanBase 控制面收敛进 sys tenant:RootService(随 `__all_core_table` Leader 由 Paxos 选举，失败后另一 OBServer 接管)聚合 schema service、DDL service、Unit/Resource Pool、负载均衡；GTS 按租户级隔离(依赖 tenant internal table Leader,system tenant 依赖 `__all_core_table` Leader，单机事务自 4.0 起免 GTS RPC),schema 元数据天然带 Tenant 维度且需 sys+租户双版本。

3. 二者时序服务的隔离粒度根本不同：TiDB 是集群级单一 TSO(窗上界持久化到 etcd 防回退),OceanBase 是租户级 GTS(租户 Paxos/LS 维护)；这决定了多租户混部下的时序隔离与跨地域延迟特性差异。

4. 元数据"唯一写者"机制对称但实现不同：TiDB 用 etcd 选举的 DDL owner,OceanBase 用 DDL epoch(`promote/check_and_lock`)保证仅一个 DDL service master，二者均防双写脑裂。

5. 控制面故障对数据面的冲击是被显式约束的局部不可用而非全局错误：TiDB 实例失联超 lease 会主动禁 DML(源码注释级证据),PD 重选靠 etcd 窗上界防 TSO 回退，DDL owner 切换由 etcd 重选且旧 schema 事务由 SchemaValidator 保护；OB 靠租户 Paxos/LS 与 ddl_epoch 收敛，GTS/SCN 切换不回退。控制面停摆与数据面失去多数派是不同故障域，不应混成一个可用性数字。

6. 对 PD/TSO/GTS/sys tenant/RS 必须按逻辑中心化、性能瓶颈、高可用单点、元数据依赖、故障恢复依赖五维度区分，不能写"单点"；真正风险在跨地域 TSO 延迟与时序服务瓶颈，而非单副本失效即全挂。官方公布的 RPO=0、RTO<8s 是 OceanBase V4.x 在 LS/Paxos 多副本、少数派故障或多数派完整场景下的集群级/数据分区级容灾指标，并非 RootService 角色切换的专属 SLA；据此推测，该指标不应外推到单副本 shared-storage 或跨云场景(见 §13.4)。

7. PD 微服务化(TSO/调度拆分自 v8.0.0、Active Followers 8.5 GA)与 OceanBase 控制面在新架构方向上的演进，代表两条控制面演进路径；但前者 GA 文档相对稀薄、后者部分细节公开度不足，均不应作为生产事实断言。TiDB Cloud 与 OceanBase Cloud 等托管形态内部控制面机制公开资料不足、细节尚不确定，本章不做实现层断言。

## 13.12 参考文献

[1] TiDB Architecture. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tidb-architecture/.
 （支撑:§13.2 对 PD 作为元数据管理、调度、事务 ID/TSO 分配组件及 PD 高可用部署建议(推荐 3 节点、奇数部署)的描述。）
[2] TiDB Scheduling. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tidb-scheduling/.
 （支撑:§13.2、§13.7 中 PD 调度信息收集(Store/Region 心跳)、三类算子、算子搭载心跳响应下发、Region/leader/replica 关系与调度影响面的描述。）
[3] PD Microservices. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/pd-microservices/.
 （支撑:§13.2/§13.7 中 PD 自 v8.0.0 拆分 TSO/调度微服务、主备容错的描述。）
[4] TimeStamp Oracle (TSO) in TiDB. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tso/.
 （支撑:§13.2 中 PD TSO 在事务时间戳分配中的角色、事务开始处取号的描述。）
[5] Best Practices for DDL Execution in TiDB. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/ddl-introduction/.
 （支撑:§13.2 中 DDL owner、在线异步 DDL、任一时刻唯一 owner、DDL job 推进路径的描述。）
[6] PD Configuration File / Troubleshooting Map. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/pd-configuration-file/.
 （支撑:§13.2 中 lease 自 v8.5.2 默认 5 秒(之前 3 秒)、tso-update-physical-interval 默认 50ms/最小 1ms，以及 §13.6/§13.8/§13.9 中 s）
[7] Configure Placement Rules. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/configure-placement-rules/.
 （支撑:§13.2 中 Placement Rules 按 key range/replica 数/Raft role/位置标签/是否参与 Leader 选举控制副本的描述。）
[8] Cluster architecture, OceanBase Database V4.2.1. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001103745.
 （支撑:§13.3 中 RootService 运行在某 OBServer、失败后另一 OBServer 接管、提供资源管理/灾备/负载均衡/schema 管理职责的描述。）
[9] OceanBase Database architecture V4.3.5 / GTS. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001971022.
 （支撑:§13.3 中 Tenant 多副本、sys tenant 存集群元数据、GTS 在 Tenant 内生成连续递增时间戳并多副本保证可用、GTS provider 与内部表 leader 相关、单机事务免 GTS、外部一）
[10] OceanBase 高可用与 RPO/RTO 文档(7 Key Technologies / RPO and RTO Explained / V4.0 Overview). 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001103745.
 （支撑:§13.4/§13.7/§13.13 中 RPO=0、RTO<8s(早期 RTO<30s)为 V4.x LS/Paxos 多副本下集群级/数据分区级容灾指标(满足金融六级容灾)、非 RootService 角色切换专属）
[11] PALF: Replicated Write-Ahead Logging for Distributed Databases. 论文，VLDB 2024[EB/OL]. https://www.vldb.org/pvldb/vol17/p3745-xu.pdf.
 （支撑:§13.3 中 PALF 作为 OceanBase Log Stream 的复制式 WAL / Paxos-backed append-only log，承载共识与事务状态推进的描述。）
[12] OceanBase: A 707 Million tpmC Distributed Relational Database on a Single-Server Cluster. 论文，VLDB 2022[EB/OL]. https://vldb.org/pvldb/vol15/p3385-xu.pdf.
 （支撑:§13.3 中 OceanBase timestamp Paxos group 与节点周期性向 timestamp leader 获取时间戳的协议级背景描述。）
[13] tikv/pd 仓库(release-8.5 @ 6dce4a68e3e9)— pkg/tso/、pkg/schedule/、pkg/keyspace/、server/config/config.go. 源码[EB/OL]. https://github.com/tikv/pd/tree/release-8.5/pkg/tso.
 （支撑:§13.2 中 maxLogical=1<<18、UpdateTimestampGuard、jetLagWarningThreshold、AllocatorManager/GlobalTSOAlloca）
[14] pingcap/tidb 仓库(release-8.5 @ 67b4876bd57b)— pkg/ddl/、pkg/domain/. 源码[EB/OL]. https://github.com/pingcap/tidb/tree/release-8.5/pkg/ddl.
 （支撑:§13.2 中 DDLOwnerKey、owner_mgr.go、三条 schema 版本 etcd 路径、SchemaValidator/schema_checker.go、schemaLease）
[15] oceanbase/oceanbase 仓库(v4.2.5_CE @ e7c676806fda；另据 4.3.5 @ b28b9bb12f3b、4.4.x @ d4bef8d29a4c checkout 跨版本 diff 核对)— src/rootserver/、src/share/schema/、src/storage/tx/. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE/src/rootserver.
 （支撑:§13.3 中 ObRootService 聚合成员与 README 控制任务、ObUnitManager、ObServerSchemaService/ObMultiVersionSchemaService）
[16] oceanbase/oceanbase 仓库 — src/share/inner_table/ob_inner_table_schema_constants.h、ob_inner_table_schema_def.py(v4.2.5_CE @ e7c676806fda；4.3.5 @ b28b9bb12f3b、4.4.x @ d4bef8d29a4c 均确认存在). 源码[EB/OL]. https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE/src/share/inner_table.
 （支撑:§13.3.1 中 __all_core_table(OB_ALL_CORE_TABLE_TID = 1)为集群首表、用户租户 GTS provider 对应 __all_dummy(OB_ALL_DUMM）
[17] Time Synchronization in TiDB: Timestamp Oracle (TSO). 官方博客[EB/OL]. https://www.pingcap.com/blog/how-an-open-source-distributed-newsql-database-delivers-time-services/.
 （支撑:§13.2 中 TSO 18+46 位结构、2^18*1000 上限、预申请时间窗(窗上界持久化 etcd)、PD 重选读 etcd 防回退的描述。 > 注：该博客(及旧 Wiki)称窗默认 3 秒，系 v8.5.2 之）
[18] Metadata Management Evolved: Scaling TiDB's Placement Driver. 官方博客[EB/OL]. https://www.pingcap.com/blog/scaling-metadata-management-innovations-tidb-placement-driver/.
 （支撑:§13.2/§13.8 中 PD 四类职责、TSO/调度/Router 微服务、Active Followers(7.6 实验/8.5 GA、约 4.5x)、大规模元数据扩展瓶颈/稳定性风险演进动因的描述。）
[19] An Interpretation of the Source Code of OceanBase (5): Life of Tenant. 第三方解读，阿里云社区[EB/OL]. https://www.alibabacloud.com/blog/an-interpretation-of-the-source-code-of-oceanbase-5-life-of-tenant_599317.
 （支撑:§13.3 中 sys tenant、ObRootService、ObUnitManager、ObTenantSchema(locality/primary_zone)、UNIT→Resource Pool→）
## 13.13 信息可信度自评

本章可信度分层如下。源码级信息(`maxLogical`、`defaultLeaderLease=5`/`DefaultTSOSaveInterval=5s`/`defaultTSOUpdatePhysicalInterval=50ms`、各 etcd 路径常量、`SchemaValidator`、`ObRootService` 聚合成员、`ObDDLEpochMgr`、`ObGtsSource` 常量与默认值等)直接来自 commit-pinned checkout 的真实文件，已逐一在 GitHub 反查证，可信度最高。官方文档明确的有：PD 四职责与高可用部署、调度心跳与算子机制、Placement Rules、PD 微服务自 v8.0.0、`lease` 默认值(v8.5.2 起 5s)与 `tso-update-physical-interval` 默认 50ms、schema lease 默认 45s 与 "Information schema is out of date" 语义、DDL owner 唯一性、GTS 客户端-服务端模型与内部表 leader 依赖、ODP 从 sys tenant 取路由。论文级有 PALF(VLDB 2024)与 707M tpmC 论文(VLDB 2022)的 timestamp Paxos group 背景。官方博客提供了 2^18*1000 上限、Active Followers 版本归属与 PD 扩展演进信号(其 TSO 窗"3s"为过时值，已按源码改为 5s)。第三方解读仅 1 条，其关键结论均已与官方文档或源码交叉印证。

已补强项(源码/官方核对后升级)：(a) `ob_root_service.h`/`ob_gts_source.h` 本章引用的核心成员经 4.2.5/4.3.5/4.4.x 逐版本 diff 确认逐字一致(见 §13.3.3 与源码注),不再属"是否随版本变化待查证";(b) GTS provider 的具体内部表名已锁定——sys tenant 对应 `__all_core_table`(TID 1)、用户租户对应 `__all_dummy`(TID 135),官方 GTS 文档与源码常量双重确认;(c) GTS/RS 排障内部视图已锁定为 `V$OB_TIMESTAMP_SERVICE`、`GV$OB_LOG_STAT`、`DBA_OB_ROOTSERVICE_EVENT_HISTORY`(均源码确认且跨三版本存在)。

仍需进一步查证/不确定项：OceanBase 的 RS/GTS 具体 Prometheus metric 名、等待事件与内部函数名;CE 版本中其余全部内部表清单与各 LS/GTS provider 在异常拓扑下的完整切换细节;TiDB Cloud 与 OceanBase Cloud 内部控制面机制未公开到足够实现细节。这些处均已显式标注，不编造。

反查证要点：其一，TSO 预申请时间窗默认值，旧 Wiki/博客称 3s，经 PD `release-8.5` 源码 `defaultLeaderLease=int64(5)` → `DefaultTSOSaveInterval=5s` 与配置文档(`lease` 自 v8.5.2 默认 5s)核实，对锁定基准 8.5.x 全章改为 5s；物理推进间隔实为 `defaultTSOUpdatePhysicalInterval=50ms`，为明确默认而非待查证项。其二，OceanBase RPO=0/RTO<8s 经多源核对为 V4.x LS/Paxos 多副本下集群级/数据分区级容灾切换指标(金融六级容灾)，官方未为 RootService 角色切换单列 RTO/RPO，本章不绑定 RootService、不外推单副本 shared-storage/跨云；早期 RTO<30s 与较新 RTO<8s 同为"数据主切换"口径的演进。其三，关于 TiDB 在线异步 schema change 与 Google F1，本章按"思想一致"的工程推测书写，不作"官方基于 F1"的归因。

版本核查备注：本章版本号均按锁定头书写——TiDB 8.5.x LTS(对照 7.5.x 特性归属)、PD/TiKV release-8.5、OceanBase TP v4.2.5_CE(源码基线，另据 4.4.2_CE 模块确认)。修订后未发现与锁定头冲突的版本断言。

---


# 第 14 章 DDL 与 Online Schema Change

## 14.1 本章核心问题

在单机数据库里，一条 `ALTER TABLE` 往往可以用一把元数据锁加一次表重建解决：加锁、改 schema、必要时拷贝数据、放锁。但在分布式 SQL 数据库里，schema 不再是"一份内存结构"，而是分散在数十甚至上百个无状态计算节点上的多份缓存副本。这些节点没有统一时钟、网络存在延迟、各自独立地把 DML 路由到底层存储。于是一个看似简单的问题变得棘手：

**当一部分节点已经认为"索引存在"、另一部分节点还认为"索引不存在"时，正在并发执行的事务如何不破坏数据一致性?**

DDL 的难点从来不是"改一行元数据"，而是在多个 SQL 节点、多个存储副本、长事务、后台回填和路由缓存同时存在时，保证老 schema 与新 schema 短时间共存也不会读到不存在的数据、漏写索引、重复删除或破坏约束。如果处理不当，会出现两类典型异常(借用 F1 论文的术语):

- **孤儿数据(orphan data)**：某节点按旧 schema 删除了一行，但另一个已加了索引的节点没有同步删除对应索引项，留下指向不存在记录的索引条目；
- **完整性异常(integrity anomaly)**：某节点按新 schema 要求"每行必须有索引项"，但旧 schema 节点插入的行没有写索引，违反约束。

这个问题之所以是分布式数据库底层的关键，在于它把三个核心机制串到了一起：**schema 多版本管理**(本章)、**MVCC 快照**(详见第 11 章)、**事务并发控制**(详见第 10 章)。同时，DDL 还是控制面和数据面的交界：控制面推进 schema version、DDL job 与 task；数据面继续处理 DML、扫描旧数据、构造新索引或 Tablet/SSTable，并在失败后接续或回滚。DDL 不只是一条"运维命令"，而是一条贯穿元数据控制面与数据存储面的完整数据路径。(中间状态与相邻版本兼容的正确性模型，F1 论文，VLDB 2013)

本章聚焦 TiDB 与 OceanBase 如何各自回答这个问题。TiDB 走的是"多中间状态、相邻状态两两兼容、全集群最多两版本共存"的在线异步渐进协议(这一思路与 Google F1 在线异步 schema change 一致，详见 §14.2 的来源说明);OceanBase 走的是"按 DDL 类型分档(instant / online / offline)加隐藏表重定义加租户级多版本 schema cache"的工程化路线。版本上，TiDB 以 release-8.5 源码为主，并对 v6.5 Fast Online DDL、v8.5 DDL 改进作版本说明；OceanBase 以 v4.2.5_CE 源码、V4.2.x/V4.3.x 官方文档与可交叉验证的实现资料为主。至于各家云上托管形态(TiDB Cloud Starter/Essential、OceanBase Cloud)的内部 DDL 隔离或弹性 worker 机制，公开度不足、细节尚不确定，本章不将其作为实现事实展开。

--- 〔文献[1]〕

## 14.2 TiDB 的实现

### schema state 状态机：渐进式状态迁移

TiDB Online DDL 采用**在线、异步**的 schema 变更方式，其工程思想与 Google F1 的在线异步 schema change 一致：F1 把一次 schema 变更拆成相邻兼容的中间状态，让不同 server 可以在不同时间看到不同 schema version，但每一步都避免数据库损坏。官方文档(docs.pingcap.com 的 `ddl-introduction`,GitHub 源码位于 pingcap/docs 的 `best-practices/ddl-introduction.md`)描述其核心是把一次 schema 变更拆成"多个相互兼容的小版本状态"(multiple small version states that are mutually compatible)，并保证集群中各节点所持小版本"相差不超过两个版本"，由唯一的 **DDL Owner**(经 etcd 选举)消费 job 队列推进。

需要特别说明归因边界：**该官方文档正文并不出现 "F1" / "Google" 字样**(EN release-8.5 与 ZH 版均经 GitHub 源码核验),TiDB 源码与设计文档 `2018-10-08-online-DDL.md` 同样未出现 "F1" / "Rae"。因此本章对 F1 的引用仅作**思想层面的对照**——协议结构(多兼容中间状态、最多两版本共存)与 F1 论文一致，但不存在"官方/官方博客明文基于 F1"的归因。需要强调，这一关系判断属于工程推测，并非官方明文结论；F1 的形式化正确性模型本身则见 §14.12 的 VLDB 2013 论文。

源码层面(`pkg/meta/model/job.go`,release-8.5 @ 67b4876bd57b),`type SchemaState byte` 的枚举(按 `iota` 顺序固定)为：

```text
StateNone(0) → StateDeleteOnly(1) → StateWriteOnly(2) → StateWriteReorganization(3)
→ StateDeleteReorganization(4) → StatePublic(5) → StateReplicaOnly(6) → StateGlobalTxnOnly(7)
```

以 **add index** 为例，状态机走的是 `None → DeleteOnly → WriteOnly → WriteReorganization → Public`。需要强调：这条五态链是理解 add 类 DDL 的主链，**不是 TiDB 全部 DDL 的完整状态枚举**——drop 路径走反向的 `DeleteReorganization`,TiFlash 副本相关走 `ReplicaOnly`，跨节点全局事务相关走 `GlobalTxnOnly`。各状态语义(源码注释加官方文档交叉确认):

**表 14-1　schema state 状态机：渐进式状态迁移**

| 状态 | 索引可读 | 索引可写 | 含义 |
|---|---|---|---|
| `DeleteOnly` | 否 | 仅删除 | 索引对内部事务可见，但只能删除索引项(防止旧节点删行后留下孤儿索引) |
| `WriteOnly` | 否 | 增/删/改 | 索引项可被增删改，但用户查询不读它(让新写入开始填充索引) |
| `WriteReorganization` | 否 | 增/删/改 | 在此状态调度 backfill worker，为存量行回填索引 |
| `Public` | 是 | 是 | 索引对用户完全可见，DDL 完成 |

源码注释明确要求"Please add the new state at the end to keep the values consistent across versions"，即枚举值跨版本稳定。F1 论文中 delete-only / write-only 的定义解释了为什么这种"可写不可读"的状态看似反直觉却能保证相邻版本兼容：为什么不能 `None → Public` 一步到位?因为那会让"无索引的节点"与"有完整索引的节点"同时服务读写，产生前文所述的孤儿/完整性异常。**每相邻两个状态保持兼容**，加上"全集群最多两版本共存"的不变式，就保证了任意时刻在用的两个 schema 不会互相破坏(F1 论文，VLDB 2013)。

### 最多两版本共存不变式与 schema version 推进

TiDB 的 `pkg/ddl/doc.go`(package 注释)写明了这个不变式的原文 :"At any time, for each schema object ... there are at most 2 versions can exist for it, current version N loaded by all TiDBs and version N+1 pushed forward by DDL, before we can finish ... we need to make sure all TiDBs have synchronized to version N+1."

每推进一步状态，schema version 加 1。源码 `pkg/ddl/schema_version.go` 的 `updateSchemaVersion` 注释为"increments the schema version by 1 and sets SchemaDiff"。推进后，owner 调用 `waitVersionSynced(...)`(内部走 `schemaVerSyncer.WaitVersionSynced`)，等待所有 TiDB 节点都同步到新版本，才允许进入下一个状态。在非 MDL 路径下，旧机制依赖 schema lease(默认 45s，详见第 13 章)做兜底：每个正常节点会在 lease 内自动 reload schema,owner 探测时间超过 `2 * lease` 即认为全集群已同步。

truncate partition、reorganize partition、exchange partition 等都走统一的 schema diff 加 version 推进路径：源码 `SetSchemaDiffForTruncateTable`、`SetSchemaDiffForReorganizePartition`、`SetSchemaDiffForExchangeTablePartition` 等函数证明了这一点。

### 控制路径：DDL job 队列与并行调度

控制路径从 TiDB Server 接收 DDL SQL 开始。DDL 语句被解析后封装为一个 **DDL job**，持久化到内部存储，由集群中唯一的 **DDL owner**(通过 etcd 选举，详见第 13 章)消费推进；其他 TiDB Server 通过 schema version、InfoSchema/Domain 缓存和 schema lease 观察新版本。在 release-8.5 中，调度已从旧版的"严格单队列串行"演进为"按 job 并行":`pkg/ddl/job_scheduler.go` 的 `jobScheduler` 持有两个 worker pool ——`generalDDLWorkerPool`(处理元数据类 general job)与 `reorgWorkerPool`(处理需要回填的 reorg job),`schedule()` / `loadAndDeliverJobs()` 在两池有空闲时并行投递 job。Owner 机制由 `ownerListener.OnBecomeOwner / OnRetireOwner` 管理，与官方文档"only one node can be elected as the owner"(任意时刻仅一个 owner)一致。

DDL job 的内部分派进一步说明了"元数据推进"与"数据重组"是两类路径：job worker 把 `ActionAddColumn`、`ActionModifyColumn`、`ActionAddIndex`、`ActionReorganizePartition` 等动作分派到不同 handler，而 `Job.MayNeedReorg()` 把 `ActionAddIndex`、`ActionAddPrimaryKey`、`ActionReorganizePartition`、`ActionRemovePartitioning`、`ActionAlterTablePartitioning` 等列为可能需要 reorg 的动作(`MODIFY COLUMN` 是否需要 reorg 取决于上下文)。这说明 TiDB 的 DDL job 不是单一"alter table 任务"，而是被拆成元数据状态推进与数据重组两类。

更进一步，自 v7.1 引入、**自 v8.1.0 起默认开启**的 **DXF(Distributed eXecution Framework)** 把单个 add index 任务切成多个 subtask，跨多个 TiDB 节点并行执行回填。**版本提示**:`tidb_enable_dist_task` 的默认值随版本而异——**8.5.x 默认 `ON`**(官方文档："Beginning in v8.1.0, this variable is enabled by default")，而**锁定的 7.5.x 默认为 `OFF`**(官方 release-7.5 system-variables 原文"Default value: `OFF`"),7.5 系列需手动 `SET GLOBAL tidb_enable_dist_task = ON` 才启用。本章涉及两个锁定基准时，凡述及"DXF 默认开启"均特指 8.5.x。开启后，大表建索引的回填吞吐可随 TiDB 节点数近似线性扩展。

### 数据路径：reorg 回填与 Fast Online DDL(ingest)

官方 DDL best practice 把逻辑 DDL 与物理 DDL 分开：逻辑 DDL 主要改元数据；物理/reorg DDL 会扫描和改写用户数据，典型包括 `ADD INDEX` 和有损列类型变更。回填(backfill / reorg)就是把存量数据补进新结构的过程，也是大表 DDL 的主要耗时来源。因此可以推测，大表 DDL 的主要瓶颈不是 schema version 递增，而是旧数据扫描、索引 KV 生成、TiKV ingest 或事务写入、Region split/compaction 以及和在线 DML 的冲突控制。TiDB 有两条回填路径：

1. **传统 txn 回填**:reorg worker 分批扫描存量行，以正常事务方式写入索引 KV。受两个 sysvar 控制(`pkg/ddl/reorg.go` 调用 `variable.GetDDLReorgWorkerCounter()` / `GetDDLReorgBatchSize()`):
 - `tidb_ddl_reorg_worker_cnt`(回填并发，默认 4，取值范围 [1, 256];16 为常见推荐值而非上限)
 - `tidb_ddl_reorg_batch_size`(每批行数，默认 256，范围 [32, 10240])

2. **Fast Online DDL(ingest 模式)**:v6.5 GA 引入，专门优化 `ADD INDEX`。源码 `pkg/ddl/ingest/` 目录的 `engine.go` 真实 import `github.com/pingcap/tidb/pkg/lightning/backend` 与 `.../lightning/backend/local`。旧实现扫描全表后用小批量事务把索引 KV 写回 TiKV，有 2PC 开销以及与用户事务冲突后的回滚重试；Fast Online DDL 改为只扫描所需列、在 TiDB 本地把索引 KV 排好序攒成有序 SST，再用 ingest 模式直接灌入 TiKV,**绕过两阶段事务回填**。配套 `mem_root` / `disk_root` 做内存与磁盘配额管理，`checkpoint` 做断点续传。开关为 `tidb_ddl_enable_fast_reorg`(源码注释"whether to use lighting backfill process for adding index")。需要强调：这不是"无代价 instant DDL"，本地临时空间、TiKV ingest 压力、排序与流控都进入运维边界，DXF 文档也要求为 TiDB `temp-dir` 准备足够空间。

官方 benchmark：据 PingCAP 官方博客《How TiDB Achieves 10x Performance Gains in Online DDL》，该组数字是 **TiDB 6.5.0 对比自家旧版 6.1.0** 的相对加速比——在空闲 workload 下建索引快约 10×，在 sysbench OLTP_READ_WRITE 10K QPS 场景下快 8~13×(同文还称约为 CockroachDB 22.2 的 3×)。这是 6.5.0 vs 6.1.0 的同产品版本自比(收益来自 6.5 引入的 ingest/Fast DDL 后端),**并非锁定基准 8.5.x/7.5.x 上的测试**。因此该比率不能外推到锁定版本，也不能读成"比某竞品快 N×"的横向绝对吞吐——它是 PingCAP 自测、特定硬件/workload 下的相对数据，不代表生产绝对值。

### MDL：老查询与新 schema 共存

`tidb_enable_metadata_lock`(MDL)在 v6.5 起默认开启，8.5 默认 `true`(`pkg/sessionctx/variable/tidb_vars.go`:`DefTiDBEnableMDL = true`)。MDL 的作用是把"两版本共存"的窗口收得更紧：官方文档原文"Metadata lock can ensure that the metadata versions used by all transactions in a TiDB cluster differ by one version at most"。执行 DML 时，TiDB 在事务上下文记录访问到的元数据对象及版本，提交时清理；DDL 推进状态时，会**等待持有旧版本的 DML 提交**后才继续。这解决了老版本臭名昭著的 `Information schema is changed` 报错(v6.3 引入 MDL、v6.5 默认启用)。调度侧 `pkg/ddl/job_scheduler.go` 的 `cleanMDLInfo(job, ownerID)` 负责随 job 生命周期清理 MDL 信息。

需要纠正一个常见直觉：这**不是** MySQL 传统意义上"一把 MDL 锁住所有 DML"。官方文档明确说 DML 不会因 MDL 而阻塞执行；被阻塞的是 DDL 的 metadata state change。把老查询与新 schema 的共存按"读路径、写路径、提交路径"拆开看会更清楚：读路径上，旧事务继续使用它第一次访问表时确定的 metadata，不会在同一事务中突然看到 public 后的新列或新索引；写路径上，中间状态决定是否需要维护新列默认值、隐藏列或新索引 KV；提交路径上，如果旧 metadata 与当前 schema version 的差异已越过兼容边界，TiDB 需通过 MDL 等待或通过 schema validator 让事务失败重试。由此可以推测出一个测试要点：验证 `ADD INDEX` 时不能只在 public 后跑 `EXPLAIN`，还要在 delete-only/write-only/reorg 期间并发 update 参与索引的列，确认最终索引与表数据一致。

### 故障路径：回滚与中断恢复

DDL job 持久化到存储，owner 故障后新 owner 会接管未完成的 job 继续推进或回滚，这是 TiDB DDL"可恢复"的基础。失败回滚由 `pkg/ddl/rollingback.go` 一组函数处理，覆盖 add index、modify column、add/drop column、exchange/truncate/reorganize partition 等：真实函数名如 `convertAddIdxJob2RollbackJob`、`rollingbackModifyColumn`、`rollingbackDropIndex`、`onRollbackReorganizePartition`、`rollbackExchangeTablePartition`。

关键约束在于：DDL job 不是任意阶段都能安全取消，取消能力由 `Job.IsRollbackable()` 判定(`cancelRunningJob` 中 `!job.IsRollbackable()` 才返回 `ErrCannotCancelDDLJob`)，且对不同 action 和 schema state 有细粒度限制。`pkg/meta/model/job.go`(release-8.5 @ 67b4876bd57b)的注释"We can't cancel if index current state is in StateDeleteOnly or StateDeleteReorganization or StateWriteOnly ... otherwise there will be an inconsistent issue between record and index"——这条不可取消约束**仅适用于 `ActionDropIndex` / `ActionDropPrimaryKey`(DROP 索引/主键)**，目的是避免 record 与 index 不一致；reorganize partition 到某些状态后也不再可回滚。**对 `ActionAddIndex`(ADD INDEX)并无该限制**:add index 经历 `None→DeleteOnly→WriteOnly→WriteReorganization→Public`，在 `DeleteOnly / WriteOnly / WriteReorganization` 这些进行中状态**都是可 cancel 的**(cancel 触发 `convertAddIdxJob2RollbackJob` 回滚、删除已写入的 index KV)，只有到达 `Public/done`(任务已"almost finished")后才不可取消。注意 `DeleteReorganization` 出现在 DROP/reorg 链路、**不在 add index 正向路径**上。官方文档 `sql-statement-admin-cancel-ddl` 也只说"If the jobs you want to cancel are finished, the cancellation operation fails"，与源码一致。据此可以推测，测试大表 DDL 失败恢复时，不能只验证 `ADMIN CANCEL DDL` 能否返回成功，还要记录 job 所在 schema state、是否已进入 backfill/ingest、是否产生 delete range、是否需要 GC 清理旧 key range。`ADMIN CANCEL DDL JOBS` 和 8.5 新增的 `ADMIN ALTER DDL JOBS`(见下)则是运维干预入口。

### v8.5 的 DDL 改进与边界

v8.5 的几项改进要按 release notes 写清边界。v8.5.0 引入 `ADMIN ALTER DDL JOBS`，允许在线调整单个运行中 DDL job 的 `THREAD`、`BATCH_SIZE`、`MAX_WRITE_SPEED` 等参数；SQL 文档进一步说明支持的 job 类型包括 `ADD INDEX`、`MODIFY COLUMN`、`REORGANIZE PARTITION`，且 TiDB Cloud Starter/Essential 不支持该语句，v8.5.5 之前部分参数对分布式 `ADD INDEX` 的适用还有条件限制。因此本章不把 v8.5.0 写成"所有 DDL 全量在线可调"。v8.5.0 还把 Accelerated Table Creation 对新建集群默认开启(8.0 GA)：官方文档说明该能力通过合并同 schema 的并发 `CREATE TABLE` 提升批量建表性能，且仅适用于不含 foreign key constraints 的 `CREATE TABLE`。

> 涉及的官方资料/源码模块(均 commit-pinned):
> - 源码：`github.com/pingcap/tidb @ release-8.5 @ 67b4876bd57b` —— `pkg/meta/model/job.go`(SchemaState / IsRollbackable / MayNeedReorg)、`pkg/ddl/`(doc.go / schema_version.go / reorg.go / rollingback.go / job_scheduler.go / job_worker.go / ingest/)、`pkg/sessionctx/variable/tidb_vars.go`(sysvar)、`pkg/parser/ast/misc.go`(AdminAlterDDLJob)。以上均确认到文件级。
> - 官方文档：docs.pingcap.com 的 `ddl-introduction` / `metadata-lock` / `system-variables` / `release-8.5.0` / `sql-statement-admin-alter-ddl` / `tidb-distributed-execution-framework` / `sql-statement-admin-cancel-ddl`。
> - 设计文档：`docs/design/2018-10-08-online-DDL.md`(zimulala/Xia Li, 2018)。

--- 〔文献[2-8,13]〕

## 14.3 OceanBase 的实现

### 按 DDL 类型分档：instant / online / offline

OceanBase 没有像 TiDB 那样用一套统一的 schema state 状态机，而是**按 DDL 操作的代价分三档**，在源码 `src/share/ob_ddl_common.h` 的 `enum ObDDLType` 里有枚举级铁证(v4.2.5_CE @ e7c676806fda):

**表 14-2　按 DDL 类型分档：instant / online / offline**

| 档位 | 枚举值区间 | 典型操作 | 数据代价 |
|---|---|---|---|
| **instant(纯元数据)** | ≥10001 | `DDL_ADD_COLUMN_ONLINE`(末尾加列)、`DDL_CHANGE_COLUMN_NAME`、`DDL_DROP_COLUMN_INSTANT`、`DDL_ADD_COLUMN_INSTANT`(在指定列前后加列) | 只改 schema，数据在后续 compaction 异步补全 |
| **long-running 单表** | <1000 | `DDL_CREATE_INDEX=5`、`DDL_DROP_INDEX=6`、`DDL_CHECK_CONSTRAINT`、`DDL_TRUNCATE_PARTITION=506`、`DDL_TRUNCATE_TABLE=503` | 单表内长事务，需要构建/补全数据 |
| **double-table 重定义(offline)** | ≥1000 | `DDL_MODIFY_COLUMN=1001`、`DDL_ADD_PRIMARY_KEY=1002`、`DDL_ALTER_PARTITION_BY=1005`、`DDL_DROP_COLUMN=1006`、`DDL_TABLE_REDEFINITION=1010` | 建隐藏表加全量数据补全加切换 |

辅助判定函数 `is_simple_table_long_running_ddl(...)`、`is_double_table_long_running_ddl(...)`、`is_long_running_ddl(...)` 真实存在。500 段(如 `DDL_TRUNCATE_PARTITION=506`、`DDL_DROP_PARTITION=504`)被归为"refuse concurrent trans"类，即执行时会拒绝并发事务而非完全在线。这一按代价分档的做法，与 TiDB 把所有 DDL 都纳入同一渐进状态机的思路形成对比。

instant DDL 的可行性来自 OceanBase 的 LSM-like 存储引擎(详见第 4 章):**官方博客原文** "Adding columns ... the data can be asynchronously completed during compactions" ——加列只改 schema，旧 SSTable 里缺这一列的行在读取时按默认值补、在 major compaction 时物理补齐。4.4 分支(@ d4bef8d29a4c)交叉确认 `DDL_DROP_COLUMN_INSTANT=10004`、`DDL_ADD_COLUMN_INSTANT=10006`、`DDL_COMPOUND_INSTANT=10007` 枚举值一致，instant DDL 在融合分支延续。

### 控制路径：RootService 侧的 DDL 调度框架

从用户路径看，DDL SQL 到达某个 OBServer 后进入解析/resolve，再由 RootService/DDL service 处理 schema 变更与后台任务。V4.2.1 架构文档说明 RootService 响应 DDL 请求并生成新 schema。OceanBase 的 DDL 中枢调度器即在 RootService 侧(`src/rootserver/ddl_task/ob_ddl_scheduler.h/.cpp`):`class ObDDLScheduler : public lib::TGRunnable`，真实方法 `schedule_ddl_task(...)`、`on_ddl_task_finish(...)`、`inner_schedule_ddl_task(...)`，内部含 `DDLScanTask`、`HeartBeatCheckTask`(以扫描加心跳驱动长事务 DDL 任务推进)。

每类长 DDL 对应一个 task 类(`src/rootserver/ddl_task/`):`ob_index_build_task`、`ob_column_redefinition_task`、`ob_table_redefinition_task`、`ob_constraint_task`、`ob_drop_primary_key_task`、`ob_ddl_retry_task` 等，基类为 `ObDDLTask`。需要纠正一处模块路径：DDL 代码实际分布在 `src/rootserver/`(直接，如 `ob_ddl_operator.cpp` / `ob_ddl_service*.cpp`)加 `src/rootserver/ddl_task/` 加 `src/rootserver/parallel_ddl/`，而非某些早期描述里的 `src/rootserver/ddl/`(真实 checkout 中不存在);`src/share/schema/` 路径正确。

### 数据路径：隐藏表重定义与单副本数据补全

对 online 类(如 create index),`ObIndexBuildTask : public ObDDLTask` 的真实方法 `send_build_replica_sql()`、`wait_data_complement()`、`create_schedule_queue()`，成员 `ObDDLTabletScheduler tablet_scheduler_`；更细一层，`ObIndexSSTableBuildTask::process()` 通过 schema guard 获取数据表与索引表 schema，生成 build replica SQL，并使用 DDL SQL proxy 按 MySQL/Oracle mode 选择执行代理，流程中带有 `snapshot_version`、`parallelism`、LS、Tablet checksum 检查等字段。OceanBase **官方博客** 描述了数据补全机制：对 create index 这类需要实时补全的 DDL，系统"can coordinate DDL and DML operations to get a version number where data completion is finished, and the transaction data of earlier versions is all committed" ——即协调出一个快照版本号，在该版本之前的事务全部已提交，补全只需对这个快照做一次性扫描排序。这与 TiDB"协调 reorg 快照后只扫存量"的思路在概念上同构。

对 offline 类(modify column / add primary key 等)，走"隐藏表加数据补全":`ob_table_redefinition_task.cpp` 的 `send_build_replica_request()` / `check_build_replica_end(...)`，从原表 `hidden_table_schema` 拷贝到隐藏表，补全完成后切换。数据补全由 `ObDDLSingleReplicaExecutor`(`ob_ddl_single_replica_executor.h`)驱动(注释明确提及 `data complement`)。由此可得两个谨慎结论：其一，OceanBase 的 index build 与 schema service、RootService 任务、Tablet/LS 组织深度耦合；其二，公开资料不足以把每个 internal table、metric 或 view 名字写死，除非在文档或源码中直接查到。

性能上，OceanBase 用 **distributed sort 加 bypass write(旁路写)** 加速：**官方博客原文** "OceanBase migrates data using distributed sorting and bypass writing ... avoids MemTable writes and multiple compactions"，并称建索引"10–20 times faster than database A, and 3–4 times faster than MySQL"。该 benchmark 来自 OceanBase 博客《How to Make DDL Execution Efficient and Transparent in a Distributed Database?》，测试版本是 **OceanBase 4.0**,**并非锁定基准 4.2.5/4.3.5/4.4.x**，无任何在锁定版本上复现该比率的官方数据；对照对象"database A"是被匿名化的某分布式数据库，既未具名也未公开配置，无法独立证伪；测试条件(1000 万行、单机、DOP=10、排序内存 128MB)为厂商自测，对手配置未披露。故该数字仅作"工程思路有效"的旁证，不作为锁定版本下的生产事实。值得一提的是，这与 TiDB Fast DDL 的 ingest(有序 SST 直写)在工程思路上殊途同归：都绕过正常事务路径、直接产出有序底层结构。

### schema 版本传播：租户级多版本 schema 服务

OceanBase 的 schema 不走 TiDB 式的全局单调 version，而是**按租户级版本号 cache**(`src/share/schema/ob_multi_version_schema_service.h/.cpp`)。`ObMultiVersionSchemaService` 提供 `get_tenant_refreshed_schema_version(...)`、`get_schema_guard(...)`(带 `tenant_schema_version` 加 `sys_schema_version` 加 `RefreshSchemaMode`);`ObSchemaVersionUpdater` 单调推进版本(`version < new_schema_version_` 才提升)。`src/rootserver/ob_ddl_operator.cpp` 中多处调用 `schema_service_.gen_new_schema_version(...)`，并把新版本写入 Tenant、table、partition 等 schema ;observer 通过 refresh 拉取新版本。这与 TiDB 的 InfoSchema/Domain 机制目标相似：都不假定所有 SQL 节点瞬时看到新 schema，而是用版本化 schema 加 guard/check 保护执行期一致性，差别在于 OceanBase 以 Tenant 为隔离维度。

> schema 的"广播/推送/刷新"接口在 `ob_multi_version_schema_service.h` 的 `/*------ refresh schema interface ------*/` 段有明确函数名(v4.2.5_CE @ e7c676806fda;4.4 @ d4bef8d29a4c 交叉确认):`broadcast_tenant_schema(tenant_id, table_schemas)`(向租户广播一批表 schema)、`async_refresh_schema(tenant_id, schema_version)`(注释"Trigger an asynchronous refresh task and wait for the refresh result")、`refresh_and_add_schema(tenant_ids, ...)`,以及 protected 的 `publish_schema(tenant_id)`(`.cpp` 内体即调用 `add_schema(...)` 把新版本落地)。observer 侧通过 `get_tenant_received_broadcast_version(...)` / `set_tenant_received_broadcast_version(...)` / `get_tenant_broadcast_consensus_version(...)` 跟踪已收到的广播版本。即版本推进后由 schema service 主动广播加 observer 异步 refresh，二者配合完成跨节点传播。

### 串行化保证：DDL epoch 单写者

OceanBase 用 **ddl_epoch** 保证全租户只有一个 DDL 写者(`src/share/schema/ob_ddl_epoch.h/.cpp`):`class ObDDLEpochMgr`，头文件注释"use ddl_epoch to promise only ddl_service master execute ddl trans" / "promote ddl_epoch when ddl_service init" / "compare ddl_epoch in ddl trans"。即每租户用 ddl_epoch 保证只有当前 ddl_service master(RootService leader)能执行 DDL 事务，**防止脑裂下双写 schema**。这在角色上与 TiDB 的"单 DDL owner"等价，但实现机制不同：TiDB 靠 etcd 选举单 owner,OceanBase 靠租户级 epoch 比对。

### 并行 DDL(4.2.x 起)与分区/Tablet 变更

OceanBase 4.2.x 起为高频 DDL 提供独立的并行路径(`src/rootserver/parallel_ddl/`)：真实 helper 类 `ob_create_table_helper`、`ob_create_index_helper`、`ob_create_view_helper`、基类 `ob_ddl_helper`、`ob_index_name_checker` 等，绕过传统串行 DDL trans。4.4 分支进一步新增 `ob_create_materialized_view_helper`、`ob_create_table_like_helper`、`ob_create_tablegroup_helper`，并行 DDL 覆盖面扩大。官方资料另指出，V4.2 中 `CREATE INDEX` 支持 `PARALLEL` hint 并行执行，而其他 DDL 可能需通过 session variable 或表的 `PARALLEL` 属性启用；不过具体的 hint 语法、语句覆盖与 CE/企业版差异尚不确定，需以对应版本文档为准。

分区变更与 Tablet 变化是 OceanBase DDL 的另一关键差异。OceanBase 4.x 的逻辑分片模型包含 Partition、Tablet、Log Stream / LS(详见第 1 章):Partition 是用户可见的 SQL 结构，Tablet 是数据承载对象，LS 是复制与事务日志边界。DDL 变更分区时，不只是更新 SQL 层元数据，还可能涉及 Tablet 创建、绑定、balance 和 checksum/状态推进 ——`src/rootserver/ob_tablet_creator.cpp`、`src/rootserver/parallel_ddl/ob_tablet_balance_allocator.cpp`、`src/share/schema/ob_partition_sql_helper.cpp` 是可定位模块(只确认到模块级，不编造具体函数语义)。这也解释了 `TRUNCATE PARTITION`、repartition、table redefinition 为什么比普通 metadata DDL 更难测：DDL 若改变 Partition 到 Tablet 的映射，既要让新 schema 在各 OBServer 上可见，又要保证相关 Tablet 的创建、清理、checksum、位置与 LS 内事务顺序不破坏正在执行的读写。公开资料没有给出每一步的稳定内部视图名，相关细节尚不确定，本章只把它作为测试维度看待。

> 涉及的官方资料/源码模块(均 commit-pinned):
> - 源码：`github.com/oceanbase/oceanbase @ v4.2.5_CE @ e7c676806fda` —— `src/share/ob_ddl_common.h`(ObDDLType)、`src/rootserver/ddl_task/`(ObDDLScheduler/各 task)、`src/share/schema/ob_multi_version_schema_service.*`、`src/share/schema/ob_ddl_epoch.*`、`src/rootserver/parallel_ddl/`、`src/rootserver/ob_ddl_service.*` / `ob_ddl_operator.*`、`src/rootserver/ob_tablet_creator.cpp`;4.4 交叉确认 `@ d4bef8d29a4c`。确认到文件/类级，个别广播函数仅到模块级(见上)。
> - 官方文档/博客：en.oceanbase.com 的"Online and offline DDL operations"(V4.3.5)、"Cluster architecture"(V4.2.1)、官方博客"How to Make DDL Execution Efficient and Transparent"。

--- 〔文献[9-12,14]〕

## 14.4 核心差异对比

**表 14-3　DDL 与 Online Schema Change:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB(release-8.5 @67b4876bd57b) | OceanBase(v4.2.5_CE @e7c676806fda) | 影响 |
|---|---|---|---|
| **Online DDL 模型** | 统一 schema state 多态机(8 态，核心 5 态)，全集群最多两版本共存(思路与 F1 一致，非官方明文归因) | 按 DDL 类型分三档：instant(元数据)/ long-running 单表 / double-table 隐藏表重定义 | TiDB 模型统一、推理简单；OB 分档更贴合代价、instant DDL 几乎零成本 |
| **控制面入口** | SQL 层 DDL owner(etcd 选举)推进 job,schema version 同步到各 TiDB Server | RootService/DDL service 处理请求并生成新 schema,schema service 负责多版本刷新 | TiDB owner 是 SQL 层逻辑 owner;OB DDL 更内聚在 RootService/sys tenant 控制面 |
| **回填/补全** | reorg(txn)加 Fast DDL ingest(lightning local backend，有序 SST 直写) | 隐藏表加单副本数据补全(distributed sort 加 bypass write)，涉及 schema guard、inner SQL、SSTable/Tablet checksum | 两者都绕过正常事务路径产出有序结构，思路趋同；重瓶颈位置不同 |
| **串行化保证** | 单 DDL owner(etcd 选举)加 MDL | ddl_epoch 单写者(仅 ddl_service master) | 都是"逻辑单写者"，防脑裂双写；机制不同 |
| **版本传播** | 全局 schema version 加 1 加 WaitVersionSynced(加 lease 兜底) | 租户级多版本 schema cache refresh | TiDB 全局统一版本；OB 按租户隔离，多租户互不影响 |
| **老查询共存** | MDL 让 DDL 等旧 DML 提交，最多差一版本(DML 不被 MDL 阻塞) | 多版本 schema guard 加快照版本协调 | 两者都保证老查询在旧 schema 下跑完；观察点与报错路径不同 |
| **并行 DDL** | general/reorg 两 worker pool 加 DXF 跨节点并行 | `parallel_ddl/` helper 路径(4.2.x 起；4.4 扩展)加 PX/PDML | TiDB 可跨节点分布式回填；OB 并行集中在建表/建索引，依赖租户资源 |
| **限速旋钮** | `tidb_ddl_reorg_worker_cnt`(默认 4,[1,256])/ `tidb_ddl_reorg_batch_size`(默认 256,[32,10240])/ `tidb_ddl_reorg_max_write_speed`(默认 0=不限，v6.5.12/v7.5.5/v8.0.0 引入)，可经 `ADMIN ALTER DDL JOBS` 在线调 | 源码确认的租户级 DDL 资源旋钮：`ddl_thread_score`(默认 0=按默认值，[0,100],DDL 工作线程权重)、`_enable_ddl_worker_isolation`(默认 False,DDL 线程隔离)、`_parallel_ddl_control`(并行 DDL 开关)、`_ob_ddl_timeout`(默认 1000s);无 TiDB 式按 MB/s 的写带宽限速等价物，更多依赖 DOP/PX/PDML 与租户资源 | TiDB 旋钮公开且可运行时改；OB 以线程权重/隔离与租户资源为主，测试更应关注 DOP 与租户资源池 |
| **模式边界** | MySQL 兼容语义；部分 Cloud Starter/Essential 不支持 `ADMIN ALTER DDL JOBS` | MySQL mode / Oracle mode 的 online DDL 支持矩阵不同，Oracle mode 属企业版边界 | 迁移与测试矩阵不能只按 SQL 文本，要按版本、模式、产品形态拆开 |

---

## 14.5 正常路径图

图 14-1 展示 TiDB add index 在正常路径下的状态推进与节点同步，并对照 OceanBase create index 的抽象路径(Mermaid):

![[f14_1.svg]]

**图 14-1　DDL 与 Online Schema Change正常读写/调度路径**

关键点：**回填发生在 WriteReorganization 状态**。此时新写入已经在更新索引(WriteOnly 起)，回填只处理存量行，因此不会漏掉边写边建期间的增量。MDL 保证每一步推进前，持有旧版本的 DML 都已提交。

OceanBase create index 的正常路径围绕 RootService DDL task 推进：

![[f14_2.svg]]

**图 14-2　DDL 与 Online Schema Change正常读写/调度路径**

两条正常路径的共同点是"先让新对象进入受限状态，再后台补齐旧数据，最后发布可读"。差异在于 TiDB 的状态机与 DDL job 暴露更清楚，OceanBase 的实现更围绕 RootService DDL task、schema service、PX/inner SQL 与 Tablet/LS 状态推进。

---

## 14.6 故障/异常路径图

图 14-3 展示 TiDB 侧几条典型异常路径：owner 故障接管、回填失败回滚、不可逆点约束(Mermaid)。

![[f14_3.svg]]

**图 14-3　DDL 与 Online Schema Change故障/异常路径**

> 注：对 **DROP index / DROP primary key**，源码 `IsRollbackable()` 另有约束——处于 `DeleteOnly / WriteOnly / DeleteReorganization` 时不可 cancel(避免 record/index 不一致)；但本图针对 **ADD index** 正向路径，其各进行中状态均可 cancel，仅到达 `Public/done` 后不可取消。

OceanBase 的对应故障路径(以 create index 为例，Mermaid):

![[f14_4.svg]]

**图 14-4　DDL 与 Online Schema Change故障/异常路径**

这两张图刻意不把故障简化为"DDL 失败就回滚"。TiDB 在某些 schema state 之后不再 rollbackable;OceanBase 的 DDL task 也可能处于等待 checksum、Tablet 状态或 RootService 接管阶段。由此可见两者的共性：**控制面(owner / RS leader)故障后，DDL 因持久化 job/task 而可恢复**，但恢复语义不同——TiDB 受 schema state 不可逆点约束，OceanBase 受 ddl_epoch 单写者约束。据此可以推测，测试报告应记录 job/task ID、schema version、状态、是否已进入数据回填、是否已对外 public，而不是只记录 SQL 返回码。

---

## 14.7 性能、可靠性、运维影响

**表 14-4　性能、可靠性、运维影响**

| 影响面 | TiDB | OceanBase |
|---|---|---|
| **延迟** | 逻辑 DDL 多为元数据更新；reorg DDL 受扫描、排序、TiKV ingest、Region split 与 compaction 影响 | online DDL 可并发读写，但 index/table redefinition/partition DDL 会消耗 PX、租户 CPU/IO、Tablet/LS 与 checksum 资源 |
| **吞吐** | Fast Online DDL 提升 index build，但增加本地临时空间与 TiKV ingest 压力；DXF 可跨节点分布式执行 `ADD INDEX` | Parallel DDL 依赖 DOP/PX/PDML 与租户资源；公开资料不足以给出统一吞吐公式 |
| **可用性** | DDL owner 故障可接续；MDL 降低旧事务提交失败；PD/etcd、TiDB owner、TiKV ingest 都是恢复依赖 | RootService 可接管，schema service 多版本刷新；DDL task 失败恢复与 RootService/sys tenant/LS 状态有关 |
| **运维** | `ADMIN SHOW DDL JOBS` / `ADMIN ALTER DDL JOBS` / 变量配置加 Grafana 面板观察；源码 `pkg/metrics/ddl.go` 确认的 DDL 指标(Namespace `tidb`、Subsystem `ddl`)如 `tidb_ddl_waiting_jobs`、`tidb_ddl_running_job_count`、`tidb_ddl_add_index_total`、`tidb_ddl_backfill_percentage_progress`、`tidb_ddl_scan_rate`、`tidb_ddl_batch_add_idx_duration_seconds` | 观察 RootService 日志、DDL task、租户资源、SQL audit、Tablet/LS 状态；内部视图字符串名与 Prometheus metric 名(源码字面量脱敏、视图文档受 WAF 限制)按版本查证 |
| **版本边界** | v6.5 Fast Online DDL;v8.5 `ADMIN ALTER DDL JOBS` 与 Accelerated Table Creation;v8.5.5 前分布式 add index 参数限制需注意 | V4.x online/offline DDL 矩阵按 MySQL/Oracle mode 分开；V4.2 起 parallel DDL 资料需与具体版本/模式交叉验证 |

**限速与提速**:DDL 本身不在事务关键路径上，但回填会消耗 TiKV / observer 的 CPU、IO 和带宽，影响在线业务延迟。对 TiDB，大表 `ADD INDEX` 的限速与提速是一组同源旋钮：`tidb_ddl_reorg_worker_cnt`(默认 4)、`tidb_ddl_reorg_batch_size`(默认 256)、`tidb_ddl_reorg_max_write_speed`(默认 0 即不限)可经 `ADMIN ALTER DDL JOBS` 在特定 job 上调整；DXF 文档建议默认 reorg worker 为 4、最大建议 16，并提示同时调度任务数上限。Fast DDL 的 ingest 模式还可用 `tidb_ddl_reorg_max_write_speed` 限制单 TiDB 到单 TiKV 的写带宽。OceanBase 则用 bypass write 减少 MemTable 与 compaction 压力，从源头降低对在线业务的冲击；但公开资料没有给出针对每类 DDL 的统一"限速参数表"，大表 DDL 的性能更依赖租户资源与并行执行框架。可以推测，DOP 不应无限拉高，因为同租户 OLTP、PX worker、IO 与 clog/compaction/merge 都共享资源池边界。

**可用性边界**：两者的 DDL 都是 online 的(对绝大多数操作不阻塞读写)，但都有边界。OceanBase 的 offline 类(modify column 等)虽然对外仍是"online"(不长时间锁表)，却需要隐藏表全量重建，代价大；TiDB 的某些操作(如 modify column 改变类型)同样需要 reorg。truncate/drop partition 在 OceanBase 被归为"refuse concurrent trans"类(500 段)，会拒绝并发事务，这是一个容易被忽略的语义差别。

**故障恢复与可观测**：两者 DDL 都持久化、可在控制面切换后续推，这是分布式 DDL 相对单机"原地改表"的关键优势——单机重建表中途宕机往往需要手工清理，而这里由系统幂等接管。但"可接续"不等于"任意时刻可无损取消":TiDB 源码明确显示部分 job state 不可回滚；OceanBase 的 index build task 也要等待 Tablet checksum/report 状态，异常时可能返回 EAGAIN 或重试。因此可以推测，运维 runbook 应优先选择 pause/alter/cancel 的官方路径，而不应直接清理元数据或删除内部 task 记录。

值得在容量规划上强调：DDL 负载不像普通 SQL 那样短平快。DDL 可能先占用控制面队列，再占用长时间后台 worker，最后还触发存储层清理或校验；即使 SQL 会话已经返回，后续 GC、delete range、Tablet 清理或 schema history 保留仍可能继续消耗资源。因此可以推测，应把 DDL 窗口当作后台批处理 workload，而非只按前台 QPS 估算；变更平台也应记录 DDL 语句、版本、job/task ID、起止时间、取消/回滚结果与业务指标变化，以免下一次迁移重复踩同一资源瓶颈。

---

## 14.8 反例与代价

**第一，Online DDL 不等于 instant DDL，也不等于零影响。** TiDB 的 `ADD COLUMN` 在很多场景主要是 metadata 变更，但 `ADD INDEX` 和有损 `MODIFY COLUMN` 仍要扫描旧数据；Fast Online DDL 只是把索引回填从事务批写改成文件式批量导入与并行 ingest，减少 2PC 与冲突开销，本地临时空间、TiKV ingest 压力仍在。OceanBase 的 online DDL 也只表示执行期间通常允许读写，不表示没有后台 task、Tablet 变化、PX 资源与 schema refresh 成本。Fast DDL 还依赖可写、足够空间的 temp-dir(推荐 SSD)，空间不足会回退到非加速回填，性能明显下降。

**第二，TiDB 取消能力受 schema state 与 DDL 类型约束。** 对 **DROP index / DROP primary key**，处于 `DeleteOnly / WriteOnly / DeleteReorganization` 时不可 cancel(避免 record 与 index 不一致)；对 **ADD index**，进行中各状态可 cancel，但一旦推进到 `Public/done` 即不可取消。换言之，真正的"不可逆点"是任务**完成**(而非某个中间状态)，运维需在完成前干预(详见 §14.2 故障路径与 §14.10)。

**表 14-5　DDL cancel／回滚能力矩阵(DDL 类型 × schema state 阶段)**

| DDL 类型 | schema state 阶段 | 是否可 cancel／回滚 | 关键约束 |
|---|---|---|---|
| ADD index | DeleteOnly／WriteOnly／WriteReorganization(进行中) | 可 | cancel 触发 `convertAddIdxJob2RollbackJob` 回滚、删除已写入的 index KV |
| ADD index | Public／done(任务已完成) | 否 | 真正的不可逆点是任务完成,需在完成前干预 |
| DROP index／DROP primary key | DeleteOnly／WriteOnly／DeleteReorganization | 否 | `IsRollbackable()` 注释:避免 record 与 index 不一致 |
| DROP index／DROP primary key | 其余阶段 | — | — |

**第三，MDL 不是"锁全表直到 DDL 完成"。** TiDB 文档说明 DML 不会因 MDL 阻塞执行，被阻塞的是 DDL 在 metadata state change 时被旧事务挡住，这与 MySQL 传统 MDL 的直觉不同。测试时若只观察 `ALTER TABLE` 会话等待，很容易误判为所有业务写被阻塞，应同时压测 DML 成功率、提交报错与 DDL state change 耗时。

**第四，F1 状态机不是万能迁移脚本。** F1 论文证明的是在共享数据、无全局 membership、server 异步切换 schema 时，某些中间状态序列可避免 corruption;TiDB 实现时还需要 DDL owner、schema version、MDL、backfill、GC/delete range、TiKV ingest 等工程部件。把 F1 论文直接套到 OceanBase 也不严谨，因为 OceanBase 公开实现不是按 F1 五态命名，而是 RootService/schema service/DDL task 体系。

**第五，OceanBase 的 offline 与模式边界代价显式暴露。** modify column / add primary key 等需要拷贝整张表到隐藏表，空间翻倍、耗时随表大小线性增长；instant 加列的"异步补全"最终要在 major compaction 落地，大表上 compaction 抖动会影响延迟(详见第 4 章);truncate/drop partition 属 500 段"refuse concurrent trans"，非完全无锁，需在低峰执行。此外 MySQL mode 与 Oracle mode 不能混写：官方 online/offline DDL 矩阵按模式区分，公共头又限定社区版以 MySQL 模式为主、Oracle 兼容属企业版边界，因此"Oracle mode 支持某 DDL online"能否推出 CE MySQL mode 同样支持，目前尚不确定，不能直接画等号。

**取舍对比**:TiDB 的统一状态机让"任意 DDL 都走同一套渐进协议"，可推理性强、回滚路径统一，但对 instant 类操作(如末尾加列)也要走多次版本同步，理论上比 OceanBase 的"纯元数据 instant"多一点延迟；OceanBase 分档让常见的加列/改列名几乎零成本，但代价是状态空间更复杂(每类 task 单独实现)、offline 类重建代价显式暴露给用户。**归根结底，这是"模型统一 vs 代价贴合"的工程取舍，把复杂度放在了不同位置，没有谁绝对更优。** 还需注意，大表 DDL 的失败代价通常不在 SQL 层返回码，而在中间产物：TiDB 可能留下待 GC 的 key range、ingest 临时文件、暂停 job;OceanBase 可能留下 DDL task、临时/隐藏 schema、Tablet 状态或 checksum 等待。公开资料不足以列出所有内部对象名，具体清单尚不确定，本章只给出测试方向。

---

## 14.9 测试开发视角的验证点

**可测试的功能场景**:
- TiDB:`ADD INDEX` 并发 insert/update/delete;`ADD COLUMN` 新旧事务交错；`DROP INDEX` 后 optimizer 不再使用旧索引；`MODIFY COLUMN` 区分无损/有损变更；`TRUNCATE PARTITION`、`REORGANIZE PARTITION` 与分区表全局/局部索引；DDL owner 重启后的 job 接续；`ADMIN ALTER DDL JOBS` 在 `ADD INDEX`、`MODIFY COLUMN`、`REORGANIZE PARTITION` 上动态调参。这些分别覆盖五状态机各分支。
- OceanBase:MySQL mode 下 `ADD INDEX`、`ADD COLUMN`、`DROP COLUMN`、`MODIFY COLUMN`、`TRUNCATE PARTITION`、`ADD/DROP/TRUNCATE PARTITION`、repartition/table redefinition、并行 `CREATE INDEX`，覆盖 instant / online / offline 三档；Oracle mode 只在企业版/对应模式环境验证，不混入 CE 结论。
- **边写边建一致性**：在 add index 的 WriteOnly→WriteReorganization 期间持续 INSERT/UPDATE/DELETE，验证回填不漏增量、不留孤儿索引(这是 F1 协议正确性的核心测试点)；并发测试要让长事务、短事务、DDL task、PX worker 与 compaction/merge 同时存在，观察 schema version 是否单调、老查询是否稳定完成或按预期报错、新查询是否只在 public 后使用新结构。
- **老查询跨 DDL 共存**：开一个长事务(旧 schema)，并发跑 DDL，验证 MDL 让 DDL 等旧事务提交、长事务不报 `Information schema is changed`。

**可注入的失效模式**:
- DDL 推进到某中间状态时 kill DDL owner / RootService leader，验证新控制面幂等接管；
- TiDB：隔离 TiDB 与 PD/etcd、让 TiKV 返回 ingest busy、填满 TiDB `temp-dir`、在 `WriteReorganization` 阶段取消/暂停、制造长事务持旧 metadata、升级前后检查是否存在长耗时 DDL，验证回退非加速路径；
- OceanBase:DDL task 执行节点故障、PX worker 资源不足、Tablet checksum/report 延迟、ODP 路由缓存刷新、Tenant CPU/IO 受限、分区 split/repartition 中断，验证清理半成品(CLEANUP_GARBAGE_TASK);
- 对 ADD index，在 DeleteOnly/WriteOnly/WriteReorganization 各状态与到达 Public 后分别执行 `ADMIN CANCEL DDL JOBS`，验证"进行中可 cancel、已完成不可 cancel"的边界；对 DROP index，验证 DeleteOnly/WriteOnly/DeleteReorganization 的不可取消约束。

**回放一致性专项**:TiDB 可以在 DDL 期间持续写入可校验数据，DDL 完成后用 primary key 扫描、旧索引访问、新索引访问与 checksum/业务聚合结果互相对照；OceanBase 可以在分区 DDL 期间按 Partition key 写入边界值，完成后分别检查分区裁剪、全表扫描、索引访问与跨 LS 事务结果。可以推测，这类用例比单纯观察 SQL 成功更有价值，因为 Online DDL 最怕的不是立刻报错，而是新旧 schema 短暂共存时留下长期潜伏的数据/索引不一致。

**关键压测指标**:DDL 端到端耗时、前台读写 p95/p99、事务提交失败率、job row count 进度、回填速率(rows/s)、reorg worker/batch/max write speed、TiDB temp-dir 使用、TiKV ingest/CPU/IO、Region split/compaction 抖动。

**关键观测/诊断命令**(均已查证，不编造):
- TiDB:`ADMIN SHOW DDL`、`ADMIN SHOW DDL JOBS`、`ADMIN SHOW DDL JOB QUERIES`、`ADMIN CANCEL DDL JOBS`、`ADMIN ALTER DDL JOBS`(8.5)。
- TiDB sysvar:`tidb_ddl_reorg_worker_cnt`、`tidb_ddl_reorg_batch_size`、`tidb_ddl_reorg_max_write_speed`、`tidb_ddl_enable_fast_reorg`、`tidb_enable_dist_task`、`tidb_enable_metadata_lock`。
- OceanBase 资源/限速参数(源码 `ob_parameter_seed.ipp` 确认):`ddl_thread_score`(DDL 线程权重，默认 0,[0,100])、`_enable_ddl_worker_isolation`(DDL 线程隔离，默认 False)、`_parallel_ddl_control`(并行 DDL 开关)、`_ob_ddl_timeout`(默认 1000s)。
- OceanBase DDL 任务通过 RootService 内部视图观测。源码常量(如 `OB_ALL_VIRTUAL_DDL_TASK_STATUS_TNAME` / `OB_ALL_DDL_ERROR_MESSAGE_TNAME` / `OB_ALL_DDL_OPERATION_TNAME`)表明确有 DDL task status / error message / operation 内部视图，但其**具体字符串字面量在本 groundtruth 中被脱敏**；对应公开视图名(`DBA_OB_*` / `GV$OB_*`)与 Prometheus metric 名仍不确定，需以目标版本 `SHOW FULL TABLES`、官方视图文档进一步确认，故不在此编造。

测试报告应保留执行时的版本、模式、系统变量与关键配置快照，否则同一条 DDL 在小版本升级或租户资源变化后很难复现实验结论。

---

## 14.10 容易误解点

1. **"Online DDL = 完全不影响业务" —— 错。** Online 指"不长时间锁表、读写可继续"，但回填会抢占底层资源，大表建索引仍可能显著抬升在线延迟。必须主动限速(TiDB 三个 reorg 旋钮 / OceanBase bypass write 加低峰执行)。TiDB 默认 `tidb_ddl_reorg_max_write_speed=0`(不限)，这是个需要警惕的默认值。

2. **"五态链是 TiDB 全部 DDL 的完整状态机" —— 不准确。** `None → DeleteOnly → WriteOnly → WriteReorganization → Public` 是理解 add 类 DDL 的主链；release-8.5 源码还存在 `DeleteReorganization`、`ReplicaOnly`、`GlobalTxnOnly` 等状态，分别服务 drop/reorg、TiFlash 副本、全局事务等场景。

3. **"TiDB 的渐进状态机和 OceanBase 是一回事" —— 不准确。** 二者都解决"多版本 schema 安全共存"，但 TiDB 是**统一渐进状态机**(任何 DDL 都走 None→...→Public),OceanBase 是**按类型分档**(instant 纯元数据 / online 补全 / offline 重定义)。OceanBase 的 instant 加列不走多次版本同步，这是两套不同的设计哲学。

4. **"DDL 随时可以 cancel" / "任意 index 中间状态都不可 cancel" —— 都不准确。** 取消能力**取决于 DDL 类型加 schema state**：对 **DROP index / DROP primary key**，处于 `DeleteOnly / WriteOnly / DeleteReorganization` 时不可 cancel(`IsRollbackable()` 注释：避免 record/index 不一致)；对 **ADD index**，进行中各状态(DeleteOnly/WriteOnly/WriteReorganization)**均可 cancel**，仅在到达 `Public/done`(任务已完成)后不可取消。官方文档 `sql-statement-admin-cancel-ddl` 亦只说"已完成的 job 取消会失败"。

5. **"官方文档明文写 TiDB Online DDL 基于 Google F1" —— 不成立。** 经 GitHub 源码核验，docs.pingcap.com 的 `ddl-introduction`(GitHub 源码 `best-practices/ddl-introduction.md`,EN release-8.5 与 ZH 版)正文**并不出现 "F1" / "Google" 字样**，只描述"online and asynchronous"、"multiple small version states that are mutually compatible"、"相差不超过两个版本"与 DDL Owner;TiDB 源码与设计文档 `2018-10-08-online-DDL.md` 同样未出现 "F1" / "Rae"。因此本章对 F1 的关系仅是**思想层面的对照**(协议结构与 F1 论文一致)，无官方明文直接归因——这一归因属工程推测。

6. **"OceanBase online DDL 可脱离模式与版本一概而论" —— 不准确。** MySQL mode、Oracle mode、CE/企业版，以及 V4.2/V4.3/V4.4 的支持矩阵与并行能力边界各不相同，尚不确定能否一概而论，不能把单个文档表格扩展成所有形态都支持。

---

## 14.11 本章结论

1. TiDB Online DDL 采用在线异步、多中间状态渐进协议，用 `None→DeleteOnly→WriteOnly→WriteReorganization→Public` 五状态机(源码共 8 态)加"全集群最多两版本共存"不变式，保证渐进式变更下无孤儿/完整性异常。该协议与 Google F1 在线异步 schema change 思想一致，但官方文档/源码/设计文档均未明文将其归因于 F1(此归因关系为工程判断，非官方明文)。

2. OceanBase 不用统一状态机，而是按代价分三档：instant(纯元数据，异步 compaction 补全)/ long-running 单表(create index，快照版本协调加 bypass write 补全)/ double-table offline(modify column 等，隐藏表重定义)，枚举级铁证见 `ObDDLType`;DDL 更围绕 RootService、DDL task、schema service、PX/inner SQL、Tablet/LS 状态推进。

3. 两者的回填/补全都绕过正常事务路径直产有序底层结构(TiDB Fast DDL 的 lightning local backend ingest;OceanBase 的 distributed sort 加 bypass write)，思路趋同但落点不同；相关性能数字(TiDB 6.5 vs 6.1 的 10×/8~13×、OceanBase 4.0 vs 匿名 database A 的 10–20×)均为旧版本/利益相关方自测，非锁定基准下的生产事实。

4. 串行化都靠"逻辑单写者":TiDB 单 DDL owner(etcd 选举)加 MDL;OceanBase ddl_epoch(仅 ddl_service master 可执行 DDL trans，防脑裂双写)。两者机制不同但角色等价；MDL 不锁 DML 执行，只让 DDL 的 metadata state change 等待旧事务。

5. DDL 可恢复性是分布式相对单机的核心优势：job/task 持久化，控制面故障后幂等接管续推或回滚；但"可接续"不等于"任意时刻可无损取消"——TiDB 的 cancel 能力受 DDL 类型加 schema state 约束(DROP index 在 DeleteOnly/WriteOnly/DeleteReorganization 不可 cancel,ADD index 仅在到达 Public/done 后不可 cancel),OceanBase 受 ddl_epoch 约束、且需等待 Tablet checksum/report。

6. Online ≠ 零影响：回填消耗底层资源，大表 DDL 必须限速；TiDB 默认 `tidb_ddl_reorg_max_write_speed=0`(不限)，是需要警惕的默认值；OceanBase truncate/drop partition 属"refuse concurrent trans"类，非完全无锁。

7. TiDB 8.5 的 DDL 改进包括 `ADMIN ALTER DDL JOBS`(运行时调 THREAD/BATCH_SIZE/MAX_WRITE_SPEED,v8.5.5 前对分布式 add index 有参数限制)、Accelerated Table Creation 默认开启(8.0 GA，仅限无 foreign key 的 `CREATE TABLE`);DXF 让 add index 跨节点并行回填——`tidb_enable_dist_task` **在 8.5.x 默认 ON(自 v8.1.0 起)，但在锁定的 7.5.x 默认 OFF**，需注意版本差异。

8. 大表 DDL 测试必须同时观察业务 p99、后台 job/task 进度、schema version、失败重试、资源隔离与版本/模式边界；可以推测，未经查证的 metric、函数、内部视图名都应保留为"需进一步查证"。

---

## 14.12 参考文献

[1] Online, Asynchronous Schema Change in F1. 论文，VLDB 2013 / PVLDB vol.6[EB/OL]. http://www.vldb.org/pvldb/vol6/p1045-rae.pdf.
 （支撑:§14.1 / §14.2 / §14.8 / §14.10 中 delete-only / write-only 中间状态、最多两版本共存不变式、孤儿数据与完整性异常、相邻版本兼容性形式化正确性模型的理论描述。）
[2] Best Practices for DDL Execution in TiDB(ddl-introduction). 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/ddl-introduction/.
 （支撑:§14.2 中 TiDB Online DDL"online and asynchronous"、"multiple small version states that are mutually compatible(相）
[3] Metadata Lock. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/metadata-lock/.
 （支撑:§14.2 / §14.8 中 MDL 保证"最多差一版本"、DDL 等旧 DML 提交、DML 不被 MDL 阻塞执行、v6.5 默认启用、解决 Information schema is changed 报错的描述。）
[4] TiDB 8.5.0 Release Notes. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/release-8.5.0/.
 （支撑:§14.2 / §14.7 / §14.11 中 ADMIN ALTER DDL JOBS、tidb_ddl_reorg_max_write_speed、Accelerated Table Creation 默认开启等）
[5] ADMIN ALTER DDL JOBS. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/sql-statement-admin-alter-ddl/.
 （支撑:§14.2 / §14.7 / §14.9 中可调 job 类型(ADD INDEX / MODIFY COLUMN / REORGANIZE PARTITION)、参数、Cloud Starter/Essential）
[6] System Variables(tidb_ddl_reorg_* / tidb_enable_dist_task). 官方文档，含 v7.5 与 v8.5 两版交叉核验[EB/OL]. https://docs.pingcap.com/tidb/stable/system-variables/.
 （支撑:§14.2 / §14.7 中 reorg worker_cnt(默认 4,[1,256])/ batch_size(默认 256,[32,10240])/ max_write_speed(默认 0,[0,1PiB])默）
[7] TiDB Distributed eXecution Framework (DXF). 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tidb-distributed-execution-framework/.
 （支撑:§14.2 / §14.7 中 DXF 把 add index 切 subtask 跨节点并行、Fast Online DDL 前置条件(temp-dir 空间)、reorg worker 默认 4 / 建议最大 16、）
[8] github.com/pingcap/tidb @ release-8.5 @ 67b4876bd57b — pkg/meta/model/job.go、pkg/ddl/、pkg/sessionctx/variable/tidb_vars.go、pkg/parser/ast/misc.go、pkg/metrics/ddl.go. 源码[EB/OL]. https://github.com/pingcap/tidb/tree/release-8.5.
 （支撑:§14.2 / §14.6 / §14.7 / §14.10 全节：SchemaState 8 态枚举、IsRollbackable / MayNeedReorg 边界、doc.go 两版本不变式、schema_vers）
[9] github.com/oceanbase/oceanbase @ v4.2.5_CE @ e7c676806fda — src/share/ob_ddl_common.h、src/rootserver/ddl_task/、src/rootserver/parallel_ddl/、src/share/schema/、src/share/parameter/ob_parameter_seed.ipp. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE.
 （支撑:§14.3 / §14.4 / §14.6 / §14.9 全节：ObDDLType 三档枚举、ObDDLScheduler 调度、ObIndexBuildTask/ObIndexSSTableBuildTask 数据补）
[10] Online and offline DDL operations, OceanBase Database V4.3.5. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001974221.
 （支撑:§14.3 / §14.8 中 OceanBase online/offline DDL 定义、事务等待策略、MySQL/Oracle mode 支持矩阵边界的描述。）
[11] Cluster architecture, OceanBase Database V4.2.1. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001103745.
 （支撑:§14.3 中 RootService 响应 DDL 请求并生成新 schema 的描述。）
[12] How to Make DDL Execution Efficient and Transparent in a Distributed Database. 官方博客，OceanBase[EB/OL]. https://en.oceanbase.com/blog/4934724608.
 （支撑:§14.3 中 instant DDL 异步 compaction 补全、create index 快照版本协调、distributed sort 加 bypass write 的机制描述。注：文中"10–20× vs）
[13] How TiDB Achieves 10x Performance Gains in Online DDL. 官方博客，PingCAP[EB/OL]. https://www.pingcap.com/blog/how-tidb-achieves-10x-performance-gains-in-online-ddl/.
 （支撑:§14.2 中 Fast Online DDL(ingest 模式、有序 SST 直灌、绕过 2PC)机制。注：10×(空闲)/ 8~13×(10K QPS OLTP)为 TiDB 6.5.0 对自家 6.1.0 的）
[14] An Interpretation of the Source Code of OceanBase (7): Implementation Principle of Database Index. 第三方解读[EB/OL]. https://www.alibabacloud.com/blog/an-interpretation-of-the-source-code-of-oceanbase-7-implementation-principle-of-database-index_599325.
 （支撑:§14.3 中 create index 从 observer 到 RootService 再到异步索引构建任务的流程；第三方源码解读，需以官方文档与源码交叉验证。 ---）
## 14.13 信息可信度自评

本章 TiDB 部分可信度较高，且分两层证据：**源码级**(commit-pinned 到 tidb release-8.5 @ 67b4876bd57b 的真实文件/类/枚举，如 SchemaState 8 态、IsRollbackable、MayNeedReorg)与**官方文档级**(docs.pingcap.com 的 ddl-introduction / MDL / 8.5 与 7.5 system variables / release notes / admin-alter-ddl / admin-cancel-ddl / DXF),F1 论文、官方文档、release notes、官方博客与源码之间能互相验证 schema state、DDL Owner、MDL、Fast Online DDL、`ADMIN ALTER DDL JOBS` 与 Accelerated Table Creation 的主要论点。部分(delete-only/write-only/两版本不变式/孤儿异常/形式化模型)来自 VLDB 2013 论文(F1)原文。OceanBase 部分可信度为模块级到机制级：官方文档确认 RootService 负责 DDL/schema、online/offline DDL 定义与模式矩阵，v4.2.5_CE 源码(4.4 @ d4bef8d29a4c 交叉确认)确认 `ObDDLType` 三档枚举、`ddl_task`、`parallel_ddl`、`share/schema` 目录与各 task 类。

**反查证发现并已处理的关键点**:
1. **F1 归因**:docs.pingcap.com 的 `ddl-introduction`(GitHub 源码 `best-practices/ddl-introduction.md`,EN release-8.5 与 ZH 版)正文不含 "F1"/"Google" 字样，TiDB 源码与设计文档 `2018-10-08-online-DDL.md` 同样无 "F1"/"Rae"。故本章将 F1 关系写为"思想层面对照(无官方明文直接归因)"，该归因判断用（推测），不作官方明文归因。
2. **`tidb_enable_dist_task` 版本默认值**:ON-by-default 自 v8.1.0 起，8.5.x 默认 ON、**7.5.x 默认仍为 OFF**，已在 §14.2 / §14.4 / §14.11 按版本区分标注。
3. **index cancel 约束**：经源码 `IsRollbackable()` 核验，不可取消约束仅对 `ActionDropIndex` / `ActionDropPrimaryKey` 成立；ADD index 在进行中各状态均可 cancel，仅到 `Public/done` 后不可取消，`DeleteReorganization` 不在 add index 正向路径上。已在 §14.2 / §14.6 / §14.8 / §14.10 / §14.11 修正。
4. **benchmark 版本错配**:TiDB 的 10×/8~13× 实为 6.5.0 vs 6.1.0 版本间自比，OceanBase 的 10–20× 实为 4.0 vs 匿名 database A，均非锁定基准下数据，标签为并在正文显式说明。
5. **OceanBase DDL 模块路径**：实际为 `src/rootserver/`(直接)加 `src/rootserver/ddl_task/` 加 `src/rootserver/parallel_ddl/`，而非 `src/rootserver/ddl/`(不存在);`src/share/schema/` 正确。
6. **v8.5 `ADMIN ALTER DDL JOBS` 边界**:release notes 与 SQL 文档均确认其存在，但 v8.5.5 前对分布式 `ADD INDEX` 参数适用有限制，本章按"v8.5 引入、具体 job/版本有限制"书写。Oracle mode online DDL 支持来自文档矩阵，但社区版以 MySQL 模式为主、Oracle 兼容属企业版边界，不作 CE 能力泛化。

**反查证补强(本轮已升级)**:(a) OceanBase schema "广播/推送/刷新"精确函数名已在源码核实为 `broadcast_tenant_schema` / `async_refresh_schema` / `refresh_and_add_schema` / `publish_schema`(`ob_multi_version_schema_service.h/.cpp`,4.4 交叉确认),不再标"待查"。(b) OceanBase DDL 资源旋钮已在 `ob_parameter_seed.ipp` 核实为 `ddl_thread_score`、`_enable_ddl_worker_isolation`、`_parallel_ddl_control`、`_ob_ddl_timeout`(4.4 交叉确认),并据此修正 §14.4 限速行——其性质是线程权重/隔离/超时与并行开关，而非 TiDB 式 MB/s 写带宽限速。(c) TiDB DDL Prometheus 指标已在源码 `pkg/metrics/ddl.go` 核实(Namespace `tidb`、Subsystem `ddl`),如 `tidb_ddl_waiting_jobs`、`tidb_ddl_running_job_count`、`tidb_ddl_add_index_total`、`tidb_ddl_backfill_percentage_progress`、`tidb_ddl_scan_rate`、`tidb_ddl_batch_add_idx_duration_seconds`,已在 §14.7 运维行补入；OceanBase 侧 metric 字符串名仍待查(见下)。

**仍不确定 / 需进一步查证 （不确定）**:OceanBase DDL 进度的具体内部视图**字符串名**(源码中相关常量标识符如 `OB_ALL_VIRTUAL_DDL_TASK_STATUS_TNAME` / `OB_ALL_DDL_ERROR_MESSAGE_TNAME` / `OB_ALL_DDL_OPERATION_TNAME` 确认存在并表明确有 DDL task status / error message / operation 视图，但其对应字符串字面量在本 groundtruth 中被脱敏、且 en.oceanbase.com 视图文档正文受 WAF 限制无法取证)、对应 Prometheus metric 名——均未在文中编造，显式标注待查。

**版本核查备注**：本章版本号严格按锁定基线书写——TiDB 主基准 release-8.5(并显式区分 7.5.x 的 `tidb_enable_dist_task` 默认值差异),OceanBase v4.2.5_CE(4.4 @ d4bef8d29a4c 交叉确认);Fast Online DDL GA 于 6.5、DXF 引入于 7.1 / 默认开于 8.1、MDL 默认开于 6.5、Accelerated Table Creation GA 于 8.0(8.5 默认开)、`tidb_ddl_reorg_max_write_speed` 新增于 6.5.12/7.5.5/8.0.0——这些为特性引入版本，与锁定主基准不冲突。benchmark 数字所属版本(TiDB 6.5、OceanBase 4.0)与锁定基准的错配已在正文显式标注。

---


# 第 15 章 全局索引 / 本地索引

## 15.1 本章核心问题

二级索引是关系数据库把「非主键列上的点查/范围查」从全表扫描降到对数级的核心手段。在单机数据库里，索引只是主表之外的一份有序冗余结构；但在分布式数据库里，表被切成多个分片（TiDB 的 Region、OceanBase 的 Tablet/Partition），索引也必须被切分。一旦索引键与分区键不一致，问题就不再停留在「有没有索引」这个表层功能上，而是索引的逻辑约束、物理 key/Tablet 布局、路由路径与事务参与者之间如何互相牵制。由此出现一个根本性的选择：

- 本地索引（Local Index）：索引的分片与主表分片严格对齐——每个主表分区有一份只覆盖本分区数据的独立索引。索引项与所索引的行永远在同一个分片/同一个共识组里，写入可以做到本地原子；但代价是，若查询条件不含分区键，系统必须在所有分区上各查一遍本地索引（扇出等于分区数）。
- 全局索引（Global Index）：索引是一份横跨整张表、按自己的规则独立分片的结构，一个索引键值能唯一定位到一行。点查非分区键时只需命中一个索引分片即可，无需扇出；但代价是，索引项与所索引的行可能落在不同分片/不同机器/不同共识组，因此一次写入往往要跨分片，触发分布式事务（2PC），回表读也可能跨机 RPC。

本章要回答的就是这组工程取舍在 TiDB 与 OceanBase 上各自如何落地：索引怎样编码成物理结构、回表怎样路由、写入是否跨分片、跨分片时一致性靠什么保证。贯穿全章的典型场景是「按 `order_id` 分区却要按 `user_id` 查询」——如果索引仍与 `order_id` 分区对齐，查询 `user_id` 会从「命中一个分区」退化成「跨所有分区探测」；如果建立 `user_id` 的全局索引，查询路径更直接，但写入路径也会从「同一参与者内维护行和索引」变成「行、索引、回表位置分属不同 key range / Tablet / Log Stream-LS」。本章与第 1 章（分片）、第 10 章（分布式事务）、第 11 章（MVCC）、第 12 章（优化器）、第 14 章（DDL）强相关，但只聚焦索引视角，跨章内容仅做指引。

一个必须先纠正的绝对化表述：并非「TiDB 所有事务都是分布式事务」。是否分布式取决于一次事务的写集落在几个 Region：单 key、单 Region 可走 1PC，跨 Region 才是完整 2PC（详见第 10 章）。索引设计（本地 vs 全局）恰恰是决定写集是否跨 Region 的关键变量之一——这正是本章要讲清楚的。

边界说明：TiDB 以 8.5.x LTS 为基准（对照 7.5.x 仅作版本说明），TiKV/PD 取 release-8.5；OceanBase 以 v4.2.5_CE 文档与源码为主，并对 4.3.5/4.4.x 的差异作版本注记。事务协议细节详见第 10 章，DDL backfill 状态机详见第 14 章，本章只分析索引引入的额外读写参与者与路由变化。

---

## 15.2 TiDB 的实现

### 15.2.1 二级索引如何编码成 KV

![[f15_1.svg]]

**图 15-1　TiDB 五类 key 字节布局**

TiDB 是 SQL-on-KV 架构：SQL 层（tablecodec）把行和索引压平成不透明的 KV，再交给 TiKV 落 RocksDB（详见第 5 章）。也就是说，TiDB 的二级索引本身就是 TiKV keyspace 中的一段有序 key，而不是 TiKV 内部某棵局部 B-tree；其实际物理分布由 Region 范围切分、Raft Group 复制与 PD 调度决定，不应把「global index」误解成「一个单点索引分片」。编码规则如下（`pkg/tablecodec/tablecodec.go`，release-8.5）：

- 前缀常量：`tablePrefix = []byte{'t'}`、`recordPrefixSep = []byte("_r")`、`indexPrefixSep = []byte("_i")`、`metaPrefix = []byte{'m'}`；`prefixLen = 1 + 8(tableID) + 2 = 11`。
- 行（record）key：`t{tableID}_r{handle}`，value 为各列编码。
- 唯一索引 key：`t{tableID}_i{indexID}{编码列值}`，value 为 handle（回表用）。
- 非唯一索引 key：`t{tableID}_i{indexID}{编码列值}{handle}`，value 可为空（handle 编进 key 尾部以保证 key 唯一）。

TiDB 用 key 的第 10 个字节区分两类 KV：`IsIndexKey` 判 `k[10]=='i'`、`IsRecordKey` 判 `k[10]=='r'`（均要求 `len(k)>11 && k[0]=='t'`）。这两个谓词与官方 docs.pingcap.com《TiDB Computing》给出的 `t10_i1_10_1 --> null`、`t10_r1 --> [...]` 示例完全一致，构成 TiDB 行/索引 KV 的真实判据。索引列值经 memcomparable 编码，保证序列化前后大小关系不变，从而索引天然按列值有序。

### 15.2.2 回表（table lookup）

二级索引 value/key 里存的是 handle（主键句柄）。读路径在执行计划上通常表现为 `IndexLookup`：先在 index key range 上扫描满足条件的 handle，再用 `GenIndexKey`/`DecodeIndexHandle` 系列函数（`tablecodec.go`）解出 handle，拼出行 key `t{tableID}_r{handle}` 回主表读完整行。对全局索引，handle 解码后会封装为 `kv.NewPartitionHandle(pid, handle)`，即额外带上 partition id 用于定位具体分区（见 `DecodeIndexHandle`/`DecodeHandleInIndexValue`，release-8.5）。

### 15.2.3 索引写入是否可能跨 Region

会。TiDB 建表时行数据与索引数据通常分配到不同的 Region（官方与第三方实践均观察到「4 个 Region 存行 + 1 个 Region 存索引」这类布局）。因此即便是本地索引，一次 `INSERT` 也常常要写「行所在 Region」和「索引所在 Region」两个或更多 Region。TiKV 的 2PC 把事务 mutation 按 Region 分组，prewrite 阶段向各 Region leader 并行下发（详见第 10 章）。但跨 Region 并不等于一定走完整的高延迟 2PC：

- 若整个事务的写集恰好落在同一个 Region（单行小事务、写集压在一个 Region leader），走 1PC：官方称 sysbench `oltp_update_non_index` 吞吐提升约 87%、平均延迟降约 50%。
- 否则触发 async commit（5.0 引入）：prewrite 全部成功即判定事务状态并提前返回，把客户端可见延迟压到约一次多数派往返，sysbench `oltp_update_index` 平均延迟降约 42%。

可见「索引写入跨 Region」是常态，但是否升级为高延迟分布式提交，取决于写集分布与这两个优化是否生效——这正是不能说「TiDB 所有事务都是分布式事务」的原因。从索引角度看，每增加一个二级索引就增加一组 KV mutation；若 global index key 与行所在 Region 不同，写入至少可能跨 Region，且 unique global index 还要做全表作用域的唯一性检查，不能只在当前 partition 内判断。这就是全局索引提升非分区键查询的同时，会放大写入冲突面、锁解析面与 Region 热点风险的根源。

### 15.2.4 分区表上的本地索引与全局索引

在非分区表上，local/global 这个二分没有意义：官方文档明确 `GLOBAL`/`LOCAL` 只影响 partitioned table，对非分区表没有差异。在分区表（详见第 14 章 DDL）上，TiDB 默认索引是本地索引：每个分区使用自己的 physical table ID，local index key 前缀实际是 partition id，因而一个 `user_id` 索引在 N 个分区里形成 N 段独立 key range。查询没有 `order_id` 或分区键条件时，优化器需对多个分区分别构造访问路径，即使每个分区内都有 `user_id` 本地索引，仍会产生多段索引扫描与多路回表。本地索引带来一个 MySQL 同源约束——「分区表上每个唯一键（含主键）必须包含分区表达式的全部列」，否则报错 8264「Global Index is needed for index, since the unique index is not including all partitioning columns」。

全局索引正是为打破该约束而生。版本时间线（docs.pingcap.com 与各版 release notes 交叉核对）：

**表 15-1　分区表上的本地索引与全局索引**

| 版本 | 全局索引状态 |
|---|---|
| v7.6.0 | 引入 `tidb_enable_global_index` 变量，功能开发中，不建议生产 |
| v8.3.0 | 作为实验特性发布，可用 `GLOBAL` 关键字显式创建 |
| v8.4.0 | GA；`tidb_enable_global_index` 废弃并恒为 `ON` |
| v8.5.0 | 全局索引可包含分区表达式的全部列（#56230） |
| v8.5.4 | 支持非唯一列建全局索引（此前仅唯一列） |

不同页面对「何时 GA」措辞略有差异：有的概述写「v8.3.0 引入」，release notes 与官方 Global Indexes 文档明确 GA 在 v8.4.0，v8.3.0 为实验态。本章按官方 release notes 书写：实验态 v8.3.0、GA v8.4.0。源码侧印证：`pkg/sessionctx/variable/sysvar.go` 中 `tidb_enable_global_index` 默认值为 `On`，validation 注释「is always turned on. This variable has been deprecated」（release-8.5）。

全局索引改变的是索引 key 前缀与回表信息。`pkg/table/tables/index.go` 中，`GenIndexKey` 在 `idxInfo.Global` 为真时使用逻辑 table ID 作为 index key 的 `phyTblID`，否则使用 partition physical ID；`pkg/tablecodec/tablecodec.go` 又在全局索引 value 的 options 中写入 partition id。具体编码（`pkg/meta/model/index.go` 与 `tablecodec.go`，release-8.5，已与 GitHub raw 源码逐字核对）：

- `IndexInfo.Global bool json:"is_global"` 标记是否全局；`GlobalIndexVersion uint8 json:"global_index_version,omitempty"` 标版本。
- 版本常量（`pkg/meta/model/index.go`，release-8.5）：`GlobalIndexVersionLegacy uint8 = 0`（partition id 不入 key，EXCHANGE PARTITION 后会 handle 冲突，见 issue #65289）、`GlobalIndexVersionV1 uint8 = 1`（partition id 同时编入 key 与 value，防冲突，release-8.5 实现态）、`GlobalIndexVersionV2 uint8 = 2`（partition id 仅入 key，源码注释逐字「GlobalIndexVersionV2 is the next, not yet implemented format (version 2) where partition ID is encoded in the key ONLY!」，未落地）。
- key 中 partition id 用 `PartitionIDFlag byte = 126` 标记：格式 `PartitionIDFlag + partition_id(8B) + inner_handle`。`GenIndexKey` 在 `GlobalIndexVersion >= V1` 且非聚簇表时写入该段；聚簇表报错「clustered index is not supported in GlobalIndexVersionV1+」。

需提醒一处版本边界：上述 `GlobalIndexVersionLegacy/V1/V2` 三常量是全局索引 GA（v8.4.0）及其后的 key 编码版本化产物，在本章锁定的主基准 release-8.5（`pkg/meta/model/index.go`）中确实存在并逐字核对一致；但在更早的 release-7.5 分支中不存在（7.5 的 `IndexInfo` 仅有 `Global bool`）。本章一切关于该常量的陈述均以 release-8.5 为准，不向 7.5.x 外推（详见 §15.13）。

clustered index 需要单独拿出来说，它与 global index 是两层概念。clustered primary key 控制行数据 key 的物理组织：primary key 本身就是 row handle，行数据只需一条 KV；nonclustered primary key 则需要 `_tidb_rowid -> row` 加 primary key index `-> _tidb_rowid` 两类 KV。在分区表上，官方文档要求默认 clustered primary key 必须包含分区键；若业务必须让主键排除分区键，可显式写成 `PRIMARY KEY(...) NONCLUSTERED GLOBAL`——其含义是把全表唯一性约束转移到 table-level 全局索引上维护，而非让主表行按该主键全局重排，代价是写入与分区 DDL 要维护这份全局结构。

结论：TiDB 全局索引是一份以 `tableID`（而非 `partitionID`）为前缀、不带分区前缀的独立索引，把 partition id 编进索引项；点查时直接命中全局索引一段连续 key range，据 `PartitionHandle` 回表到对应分区，无需扫所有分区。读路径的源码佐证在 `pkg/planner/core/planbuilder.go`：`getPhysicalID` 在非全局索引且有 partition info 时返回 partition physical ID，`buildPhysicalIndexLookUpReaders` 对每个 partition 构造 reader；对全局索引则返回 table ID，并在回表侧增加 physical table ID 列，因为回表必须知道实际 partition。

DDL/backfill 边界也更重。官方全局索引文档明确：`DROP PARTITION`、`TRUNCATE PARTITION`、`REORGANIZE PARTITION` 会触发 table-level 全局索引更新，DDL 要等这些更新完成后才返回；包含全局索引的表不支持 `EXCHANGE PARTITION`。本地索引可利用分区独立性做快删快换，全局索引会打破这部分独立性，因为删除一个分区的行还要清理散布在全局索引 keyspace 中的条目。

涉及的官方资料/源码模块：
- 官方文档：docs.pingcap.com《TiDB Computing》《Global Indexes》《Clustered Indexes》《Partitioning》、各版 release notes。
- 设计文档：`pingcap/tidb` 仓库 `docs/design/2020-08-04-global-index.md`。
- 源码（commit-pinned）：`pingcap/tidb` @ `release-8.5`（commit `67b4876bd57b`）—— `pkg/tablecodec/tablecodec.go`（编码/前缀/Flag/回表解码）、`pkg/table/tables/index.go`（`GenIndexKey`）、`pkg/meta/model/index.go`（`Global` 字段与版本常量）、`pkg/planner/core/planbuilder.go`（`getPhysicalID` 回表）、`pkg/sessionctx/variable/`（sysvar）。以上均确认到文件级。

--- 〔文献[1-5,10,14]〕

## 15.3 OceanBase 的实现

### 15.3.1 索引即索引组织表（IOT）

与 TiDB 的 SQL-on-KV 路线不同，OceanBase 是关系原生存储，更接近「索引也是表」的模型：表本身是索引组织表（IOT），二级索引在 schema 层就是一张独立的、可独立分区的索引表，行里冗余了主键列以便回表。官方源码解读明确：索引表「有自己的 schema、内存结构、磁盘数据和分布式位置信息」，回表「通过主键快速找到主表完整行，这一过程称为 index back to table」。

官方文档把分区表上的索引分成两类：local index 与主表使用相同分区策略，每个 local index partition 对应一个 table partition；global index 可以采用独立分区策略，也可以是非分区索引对象，覆盖整张分区表的所有行——即 local 是 partition-level、global 是 table-level，local index partition 与主表 partition 一一对应，global index table partition 与主表 partition 没有对应关系。

源码层面，`src/rootserver/ob_ddl_operator.cpp` 在创建索引时给 index schema 分配新的 `table_id`，再调用 `create_table(index_schema, trans)`；drop index 时调用 `drop_table(*index_table_schema, trans)`。同一文件在处理辅助表时区分 `is_global_index_table()`：全局索引不继承数据表的 tablegroup，因为 tablegroup 内表分区数相等的假设不一定适用于全局索引。因此，OceanBase 的全局索引不只是 SQL 元数据上的一个 flag，它有 index table schema、partition、Tablet 与位置属性；local index 则通过「与主表同分区/同位置」的关系获得局部维护优势。

### 15.3.2 本地 / 全局 / 全局本地存储 三类索引

`ObIndexType` 枚举（`src/share/schema/ob_schema_struct.h`，v4.2.5_CE，commit `e7c676806fda`，已与 GitHub raw 逐值核对）：

- `INDEX_TYPE_NORMAL_LOCAL=1`、`INDEX_TYPE_UNIQUE_LOCAL=2`
- `INDEX_TYPE_NORMAL_GLOBAL=3`、`INDEX_TYPE_UNIQUE_GLOBAL=4`、`INDEX_TYPE_PRIMARY=5`
- `INDEX_TYPE_NORMAL_GLOBAL_LOCAL_STORAGE=7`、`INDEX_TYPE_UNIQUE_GLOBAL_LOCAL_STORAGE=8`

枚举里有个关键工程优化：在非分区表上建的 global index 被当作 local index 处理以获得更好访问性能（源码注释：「we regard i1 as a local index for better access performance. Since it is non-partitioned, it's safe to do so」），这就是 `*_GLOBAL_LOCAL_STORAGE` 类型的由来——逻辑上是全局索引，物理存储退化为本地。该枚举值在 4.2.5 / 4.3.5 / 4.4.x 三个 checkout 中一致（分别 commit `e7c676806fda` / `b28b9bb12f3b` / `d4bef8d29a4c`），索引模型在 4.x 内稳定。

schema 谓词（`src/share/schema/ob_table_schema.h`，v4.2.5_CE）：`is_global_index_table()` 等于 NORMAL_GLOBAL || UNIQUE_GLOBAL || SPATIAL_GLOBAL；`is_unique_index()` 覆盖三类 UNIQUE。即 OceanBase 全局索引在 schema 层就是独立可分区的索引表。

### 15.3.3 默认 scope 与建模规则

`src/sql/resolver/ddl/ob_create_index_resolver.cpp`（v4.2.5_CE，已逐字复核）在索引 scope 未指定时：`global_ = is_partitioned || lib::is_oracle_mode()`，源码注释逐字「partitioned index must be global, MySQL default index mode is local, and Oracle default index mode is global」。即：

- MySQL 模式默认本地索引，Oracle 模式默认全局索引；
- 当索引语句自身携带分区定义（`T_PARTITION_OPTION`）时，该索引必为全局索引——这里 `is_partitioned` 指的是「索引 DDL 自身带了独立分区子句」，而非「基表是分区表」；
- 全局索引可带独立 `T_PARTITION_OPTION` 分区定义（`resolve_index_partition_node`），证明其是可独立分区的索引表。

这里有一处容易反向误读：上一条「携带分区子句的索引必为全局」不能反推成「凡是分区表上的索引都必须是全局索引」。恰恰相反，OceanBase 把与基表分区对齐的索引称为局部索引（又名分区索引：分区键等同表分区键、分区数等同表分区数）；语法上 `CREATE INDEX ... LOCAL` 显式支持 `local_partitioned_index` 子句。也就是说，最典型的「分区索引」恰恰是 local index；global index 反而可选非分区。源码 `is_partitioned` 触发 global 的语义，仅限「索引 DDL 自身写了分区子句」这一情形（详见 §15.13）。由于公共头限定社区版以 MySQL 模式为主，本章不把 Oracle 模式或 Cloud 文档里的默认 global 行为直接泛化到 OceanBase CE MySQL 模式。

由于表是 IOT，分区键必须是主键的子集（官方文档），这样给定主键才能快速定位分区。本地唯一索引要保证全局唯一，必须把分区键并入唯一键；全局唯一索引则无此限制——这正是全局唯一索引的核心价值。

### 15.3.4 全局索引的路由与一致性

路由上，local index 的关键是「位置关系」。`src/sql/ob_sql.cpp` 注释写明 storage local index table 与主表位置相同，因 local index table schema 不完整，reroute 时返回主表给 proxy；`src/sql/das/ob_das_location_router.h` 的注释允许 `table_schema` 是 data table/local index/global_index，并把 related table 关系用于 data table 与 local indexes 的双向关联。这意味着本地索引扫描与回表更容易保持在同一分区/Tablet/LS 位置集合内，但前提是查询能确定目标分区；若只按非分区列 `user_id` 查，本地索引仍可能需要遍历所有相关分区。

全局索引的路由则走索引表本身。优化器 `ObLogTableScan`（`src/sql/optimizer/ob_log_table_scan.h`，v4.2.5_CE，已逐字复核）有成员 `is_index_global_`，`inline uint64_t get_location_table_id() const { return is_index_global_ ? index_table_id_ : ref_table_id_; }`。也就是说，全局索引扫描的分区路由按「索引表自身的 location」计算，而非主表；本地索引才用主表 `ref_table_id_`。该函数在 4.2.5_CE / 4.3.5_CE / 4.4.2_CE 三 tag 逐字一致。回表由 DAS 执行层的 `ObDASLookupIter`（`src/sql/das/iter/ob_das_lookup_iter.h`）承担索引扫描到主表 lookup。

在代理层，ODP 文档说明：对带全局索引的分区表发起强一致读时，ODP 可用 SQL 里的 index value 按全局索引表的 partitioning key 计算路由；首次 index-based query 可能随机路由，OceanBase 返回 index table leader 后，ODP 建立从 SQL 到 index table 名称的映射，后续可直接用 index table 计算路由。这解决了「`order_id` 分区但 `user_id` 查询」的第一跳路由问题。但要把「路由命中」和「执行完成」拆开：ODP 能把请求发到全局索引表的合适 leader，可一旦查询需要主表中不在索引表里的列，OBServer 仍要执行 index lookup 再访问主表 Tablet；若主表 Tablet 所在 LS leader 不在同一 OBServer，仍可能产生远程访问或分布式执行片段。因此全局索引表路由主要减少第一跳找索引表的不确定性，它不是覆盖索引，也不自动消除回表。

一致性上，OceanBase 的代价直接落在 LS 与事务参与者上。架构文档说明：Partition 对应 Tablet，每个 Tablet 属于一个 LS，一个 LS 可关联多个 Tablet；Tablet 上 DML 生成 redo log 写入 LS；单 LS 事务可由 LS WAL 保证原子性，跨 LS 则由优化 2PC 保证原子性。主表与全局索引表的写入要原子，必须放进同一个事务；当主表分区与索引分区不在同一 LS 时，提交走以 Log Stream 为原子参与者的树形 2PC（详见第 10 章）。源码侧这条「同事务多参与者 → 2PC」的入口已确认到事务层文件级：被触达的每个 LS 经 `ObTxDesc::update_parts`（`src/storage/tx/ob_trans_define_v4.cpp`）进入参与者集合；提交时 `ObPartTransCtx::commit()`（`src/storage/tx/ob_trans_part_ctx.cpp`）按 `exec_info_.participants_.count()` 判定分支——单 LS 走 `SP_TRANS`/`one_phase_commit_()`，跨 LS 走 `DIST_TRANS`/`two_phase_commit()`。全局索引在分布式环境下不可避免涉及分布式事务与跨机查询，因此依赖 GTS 维护全局一致性快照，官方资料据此暗示其只能在 GTS 开启时使用（GTS 按租户级隔离，详见第 13 章）。官方文档也明确提醒：全局索引可能使 DML 变成分布式事务，因而建议尽量使用本地索引，只有必要时才使用全局索引。

DDL/backfill 上，`src/rootserver/ddl_task/ob_index_build_task.cpp` 里的 `ObIndexSSTableBuildTask::process()` 通过 multi-version schema guard 获取数据表 schema 与 index schema，生成 build replica SQL，带 `snapshot_version`、`parallelism`、LS leader、partition names 等信息推进索引构建。这支持一个保守结论：OceanBase 建索引不是简单元数据操作，而是 index table schema 创建、旧数据补齐、SSTable/Tablet 校验和状态发布的组合；全局索引因为重排全表数据到独立索引表，通常比本地索引更容易引入跨分区排序、PX/inner SQL、租户资源与跨 LS 一致性代价。

涉及的官方资料/源码模块：
- 官方文档：en.oceanbase.com《Indexes on partitioned tables》《Best practices for creating indexes on large tables》《OceanBase Database architecture V4.3.5》、ODP《Global index table-based routing for strong-consistency reads》。
- 官方博客：阿里云开发者社区《一文详解 OceanBase 2.0 的「全局索引」》、Alibaba Cloud《OceanBase 源码解读（7）：数据库索引实现原理》。
- 论文：《A Tree-Structured Two-Phase Commit Framework for OceanBase》（2PC 框架；注：论文未直接讨论全局索引跨分片写，本章仅借其说明 LS 作为原子参与者的提交机制）。
- 源码（commit-pinned）：`oceanbase/oceanbase` @ `v4.2.5_CE`（commit `e7c676806fda`）—— `src/share/schema/ob_schema_struct.h`（`ObIndexType`）、`ob_table_schema.h`（谓词）、`src/sql/resolver/ddl/ob_create_index_resolver.cpp`（默认 scope）、`src/sql/optimizer/ob_log_table_scan.h`（`is_index_global_` 路由）、`src/sql/das/iter/ob_das_lookup_iter.h`（回表）、`src/sql/das/ob_das_location_router.h`（位置路由）、`src/rootserver/ob_ddl_operator.cpp` 与 `src/rootserver/ddl_task/ob_index_build_task.cpp`（索引建表与 backfill，另据 v4.4.2_CE@`e859d1b9c9a6` 亦确认）。索引扫描/回表的 DAS、optimizer 标志位与事务层 1PC/2PC 入口现均确认到文件级：主表与全局索引表 DML 在不同 LS 落地时，每个被触达的 LS 经 `ObTxDesc::update_parts`/`update_part`（`src/storage/tx/ob_trans_define_v4.cpp`）累积进事务参与者集合 `tx.parts_`；提交时 `ObPartTransCtx::commit()`（`src/storage/tx/ob_trans_part_ctx.cpp`）按 `exec_info_.participants_.count()` 分支——单参与者（同 LS）走 `TransType::SP_TRANS` 与 `one_phase_commit_()`，多参与者（跨 LS）走 `TransType::DIST_TRANS` 与 `two_phase_commit()`。该 1PC/2PC 分支结构在 4.2.5_CE 与 4.4.x（commit `d4bef8d29a4c`）逐字一致。仍未定位到文件级的是「全局索引在 partition split/merge/transfer 下的路由一致性维护代码」，仅确认到模块级（涉及 `transfer_epoch`/`intermediate_participants_` 等，详见第 10 章），按「需进一步查证」处理。

--- 〔文献[6-9,11-12,15-16]〕

## 15.4 核心差异对比

**表 15-2　全局索引/本地索引:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB | OceanBase | 影响 |
|---|---|---|---|
| 索引物理抽象 | SQL-on-KV：table/index 都是 TiKV 有序 KV，tablecodec 压平为 `t{tid}_i{idx}...` 落 RocksDB | 关系原生 IOT：索引是独立 index table（有自己的 schema/Tablet/位置），blocksstable 组织 | TiDB 更适合按 keyspace/Region 分析；OceanBase 更适合按 index table/Tablet/LS 分析 |
| 本地索引编码 | partition id 作 key 前缀，每个分区一段独立 index key range | 索引表分区与主表分区一一对应（local storage），位置关系更紧 | 都保证索引项与行同分片，利于分区维护与局部写入 |
| 全局索引编码 | 逻辑 tableID 作前缀，partition id 编入索引项（`PartitionIDFlag=126`，`GlobalIndexVersionV1`） | 独立分区的索引表，按索引自身分区规则路由 | 都做到「一键定位一行」，不扫全分区 |
| 默认索引（分区表） | 本地索引；唯一键须含全部分区列，否则要求全局索引（error 8264） | MySQL 模式默认本地、Oracle 模式默认全局；索引自身带分区子句时必为全局（分区表索引本身仍可为 local） | 默认行为不同，迁移需注意唯一性作用域 |
| 全局索引版本支持 | 实验 v8.3.0 / GA v8.4.0 / 含全部分区列 v8.5.0 / 非唯一列 v8.5.4（锁定基准 8.5.x） | 4.x 全程具备，枚举稳定（2.0 即引入） | OceanBase 全局索引更成熟稳定 |
| 唯一约束 | 全局唯一索引可解除「唯一键须含分区表达式全列」的限制；clustered PK 边界需 `NONCLUSTERED GLOBAL` 显式处理 | 本地唯一索引只保证分区内唯一；全局唯一索引承载表级唯一性，代价更高 | 迁移 Oracle/分区表 schema 时不能只看约束名，要看唯一性作用域 |
| 回表路由 | handle 带 partition id（`PartitionHandle`）定位分区，`IndexLookup` 两段 | 优化器按索引表 location 路由（`is_index_global_`→`index_table_id_`），`ObDASLookupIter` 回主表 | 都是「索引定位→回表读行」两段 |
| 跨分片写一致性 | client-go 2PC（primary lock）；1PC/async commit 短路 | 树形 2PC（LS 为原子参与者）；单 LS 短路；依赖 GTS 快照 | 都靠 2PC + 短路优化，放置点不同 |
| 全局索引一致性前提 | 与普通跨 Region 事务同机制，无独立开关 | 必须开启 GTS | OceanBase 显式把全局索引绑定到 GTS |
| 分区 DDL | 全局索引使 DROP/TRUNCATE/REORGANIZE PARTITION 等待全局索引更新；不支持 EXCHANGE PARTITION | 全局索引建表/补数据/校验更重；本地索引更保留分区独立性 | 数据归档型分区表通常更偏本地索引 |

---

## 15.5 正常路径图（按非分区键点查全局索引 + 回表）

![[f15_2.svg]]

**图 15-2　全局索引/本地索引正常读写/调度路径**

正常路径的共同点是「先通过 `user_id` 的全局结构缩小候选行，再回表取完整记录」。差异在于：TiDB 的第一跳是 TiKV 全局索引 key range，回表靠 index value 中的 partition id；OceanBase 的第一跳是全局索引表的路由与 Tablet/LS leader，回表再访问主表 Tablet/LS。

正常写路径（`INSERT`/`UPDATE` 同时改主表行 + 全局索引项）：两系统都把「行所在分片」与「索引所在分片」纳入同一事务。插入一行订单时先按 `order_id` 计算目标分区生成 row key：若存在本地 `user_id` 索引，则 index key 沿用同一分区前缀；若存在全局 `user_id` 索引，则 index key 改用 tableID 前缀、value 写入 partition id，这条 index mutation 的路由因而由 `user_id` 编码值决定。更新索引列时还需删除旧 index entry、插入新 entry，并保持行与索引在同一事务提交结果下可见。若两者同分片（TiDB 同 Region / OceanBase 同 LS）走单分片短路（1PC / 单 LS），否则走 2PC。

---

## 15.6 故障 / 异常路径图

![[f15_3.svg]]

**图 15-3　全局索引/本地索引故障/异常路径**

这张图刻意区分路由失效与事务一致性两类异常。

路由失效一类（回表跨机时主表分区 leader 切换）：TiDB 由 RegionError 驱动 RegionCache 失效再向 PD 问 leader、OceanBase 由 OBServer 反馈驱动 ODP 刷新 tablet→LS 路由（详见第 2 章），表现为延迟抖动而非错误返回。Region leader 变更不改变全局索引的逻辑 key，ODP 路由缓存失效也不改变索引表 schema。

事务一致性一类（跨分片写在提交阶段被打断）：TiDB 走 Percolator 2PC，prewrite 行+索引各分片后 primary lock 落在某 key，若 commit 阶段崩溃，其他读事务遇残留 lock 时经 CheckTxnStatus 查 primary、ResolveLock 推进或回滚，最终索引项与行一致——这是读触发的惰性恢复；OceanBase 走以 Log Stream 为原子参与者的树形 2PC，prepare 各参与 LS 后协调者 LS 落 commit 日志，若参与者 leader 切换则新 leader 重放 2PC 日志，由状态机主动驱动恢复。两者哲学相反（惰性 vs 主动），但都保证索引与行最终/原子一致（详见第 10 章）。

代价路径一类（分区 DDL 触发全局索引维护）：TiDB 全局索引项按 tableID 前缀散布、不连续，无法一次 range delete 删掉某分区全部索引项，需加 reorg 态扫分区记录逐条删索引（类似 add index），DDL 必须等更新完成才返回；OceanBase 删主表分区时索引分布不一致，需维护独立索引表、可能影响业务窗口。

---

## 15.7 性能、可靠性、运维影响

延迟与吞吐——全局索引的收益集中在「查询条件不含分区键、但选择性较高」的场景。以订单表为例，`PARTITION BY HASH(order_id)` 对按 `order_id` 点查非常友好，但 `WHERE user_id=? ORDER BY created_at LIMIT 20` 若无 `user_id` 全局索引，需在多个分区上找候选再合并。本地索引读非分区键等于分区数次扇出（cop task / 子计划数随分区线性增长）；全局索引把扇出降到命中的索引分片数（常为 1）。官方称 TiDB sysbench `select_random_points` 在 100 分区下最高约 53 倍提升，并显著降低 RU 与 cop task 数量；OceanBase 第三方实测的 EXPLAIN 也显示全局索引把「扫 10 个分区」变成单个 DISTRIBUTED TABLE SCAN + TABLE LOOKUP。代价对称地落在写侧：全局索引把部分读放大转移成写放大，使原本本地的写升级为跨分片 2PC，写延迟与吞吐随之受冲击；unique global index 还要做全表作用域的冲突检查。

可用性与故障恢复——两者都不依赖「单个索引节点」：TiDB 的全局索引 key 会被 Region split/merge 与 PD 调度分散复制，OceanBase 的全局索引表也有 Tablet/LS 复制与 leader 切换。但这不代表没有关键依赖：TiDB 仍依赖 schema/DDL、PD/Region 元数据与 TiKV Region 可用，OceanBase 仍依赖 RootService/schema service、ODP 路由缓存与 LS leader 恢复。回表跨机时任一端 leader 切换都触发路由失效与重试，表现为延迟抖动；跨分片写的崩溃恢复哲学相反（TiDB 惰性 ResolveLock vs OceanBase 主动重放 2PC 日志），但都保证索引与行最终/原子一致（详见 §15.6 与第 10 章）。测试报告应写明「路由缓存失效后的重试路径」或「DDL task 是否恢复」，而非笼统的「索引故障」。

扩展性与热点——本地索引天然随主表分区水平扩展、无额外热点风险；全局索引若按低基数列分区或分区数过少，热点会跟随全局索引 key 集中到部分 Region/Tablet，成为写热点与扩展瓶颈，需为全局索引单独设计分区。容量规划还要考虑索引条目的生命周期：一条订单从创建、支付、发货到关闭可能多次更新状态与时间字段，若全局索引以这些字段为前导列，每次更新都会改变写入热度与跨参与者提交比例。因此，全局索引更适合稳定、高选择性的查询键，不宜把频繁变化、低基数或强时间单调的字段随意放在前导列。若业务确实需要如此，应提前用散列分区、复合索引顺序或应用侧改写降低热点，并在压测中观察 P99 而非只看平均延迟。

运维复杂度——全局索引应作为表设计的一部分提前规划，而不是查询慢了就补一个索引。它把 `DROP/TRUNCATE/REORGANIZE PARTITION` 这类原本快速的元数据/区间删除操作，变成需要 reorg 扫描或重建的重操作（TiDB 加 reorg 态、OceanBase 删后维护独立索引表），数据归档场景执行时间随索引数增长，运维需为分区滚动预留窗口。

**表 15-3　性能、可靠性、运维影响**

| 对比维度 | 本地索引 | 全局索引 | 性能瓶颈来源 |
|---|---|---|---|
| 非分区键点查 | 扫所有分区（扇出=分区数） | 命中单索引分片 | 本地索引：cop task / 子计划扇出 |
| 写入路径 | 索引项与行同分片，可本地原子 | 跨分片→2PC | 全局索引：跨分片协调 + GTS/TSO 取版本 + unique 全表检查 |
| 分区 DDL | 随分区独立删，快 | reorg 扫描 / 维护独立索引表，慢 | 全局索引：索引项不连续 / 分布不一致 |
| 唯一约束（不含分区键） | 不支持（须并入分区键） | 支持（全局唯一） | 本地索引：约束受分区键裹挟 |
| 扩展性热点 | 随主表分区扩展，无额外热点 | 取决于索引自身分区设计 | 全局索引：索引分区/前导列设计不当成热点 |

--- 〔文献[13]〕

## 15.8 反例与代价

- 写多读少 / 高并发写：全局索引把本地写变为跨分片 2PC，写放大与提交延迟显著。在高并发写场景采用全局索引前应先做压力测试。若查询都带分区键，本地索引已足够，不宜仅因「看起来更通用」而选用全局索引。
- 频繁分区滚动 / 归档：`DROP/TRUNCATE PARTITION` 是按时间归档的常用手段；带全局索引会触发 reorg 或重建，把原本秒级的操作拖成长耗时，且 TiDB 明确含全局索引的表不支持 `EXCHANGE PARTITION`（常用的快速换数据手段失效）。
- 本地索引不等于「只能查分区键」：本地索引可以建在 `user_id` 上，但它是每个分区一份；若 SQL 没有分区键，系统仍要查多个分区的本地索引。对小分区数或低频查询，这可能足够；对数百分区的高频查询，全局索引才更可能减少 RPC、cop task 或 ODP 路由/远程计划数量。
- 唯一约束的作用域最容易误解：OceanBase 文档明确本地唯一索引在分区表上只保证分区内唯一，要作为表级唯一约束必须包含分区键；TiDB v8.4 GA 后全局唯一索引可解除「唯一键须含分区表达式」的限制。因此从 Oracle 迁移带全局唯一索引的分区表时，不能直接改成本地唯一索引，否则唯一性语义可能收窄到分区内。
- 聚簇表 / 主键约束：TiDB 全局索引 V1 不支持聚簇表（「clustered index is not supported in GlobalIndexVersionV1+」），且聚簇主键必须含全部分区列；OceanBase IOT 要求分区键是主键子集。两系统都无法在「主键完全不含分区键」上随意建全局聚簇结构。需注意 clustered index 与 global index 是两层概念：前者是行数据 key 的物理组织方式，后者是分区表上索引 key 的全表作用域，`PRIMARY KEY(...) NONCLUSTERED GLOBAL` 指主键作为非聚簇全局索引维护唯一性，不是让主表行按该主键全局重排。
- DDL backfill 不是索引逻辑之外的杂项：添加全局索引需要扫描旧数据并构造索引项，TiDB 有在线 DDL/backfill 路径、OceanBase 有 RootService DDL task 与 inner SQL/PX、SSTable/Tablet 校验路径。在大表上，索引设计评审必须同时评审建索引窗口与回滚策略，否则查询收益可能被构建期间的资源冲击抵消。
- 「把所有查询列都塞进索引」也不是通用答案：覆盖索引能减少回表，但会扩大 index key/value 或 index table row，让写入、compaction/merge、统计信息与 DDL backfill 都变重。若只是偶发管理查询，让它走多分区扫描可能比为主链路引入宽全局索引更可维护；若它是核心用户路径，则应把覆盖列、排序列与回表列作为同一个设计问题一起评审。
- 取舍本身：替代方案是冗余表/物化（把按 `user_id` 组织的数据另存一份），用应用层双写或异步同步换取读路由简单，但牺牲强一致与一致性窗口。全局索引的价值正是把这份「冗余 + 一致性维护」下沉到数据库内核、用一次 2PC 换强一致；代价就是写侧的分布式开销。这是工程取舍而非优劣。

---

## 15.9 测试开发视角的验证点

可测功能场景：① 分区表按非分区键唯一/非唯一全局索引点查 vs 全表扫描的计划差异（`WHERE user_id=?` 是否从多分区 IndexLookup 变成全局 IndexLookup）；② 全局索引唯一约束跨分区生效（插入跨分区重复键应报冲突）；③ TiDB 8.5.4 非唯一全局索引、8.5.0 含分区表达式全列、clustered 与 `NONCLUSTERED GLOBAL` 主键的 `SHOW CREATE TABLE`；④ OceanBase MySQL 模式默认本地 vs Oracle 模式默认全局的 scope 行为，以及 `WHERE user_id=?` 是否命中全局索引表路由（ODP `EXPLAIN ROUTE` 是否显示 index table 路由）。

可注入失效模式：回表跨机时杀主表分区 leader，验证路由失效→重试→最终成功（不丢不错）；跨分片写 prewrite/prepare 后杀协调端，验证 TiDB ResolveLock 惰性恢复 / OceanBase 2PC 日志重放主动恢复；`DROP PARTITION` 中断，验证全局索引 reorg/重建的可恢复性；OceanBase 还应覆盖 ODP 路由缓存过期、索引表 leader 切换、主表 LS leader 与索引表 LS leader 分别切换、RootService/DDL task 执行节点故障、PX/inner SQL 资源不足。

关键压测指标与方法：以「读收益是否覆盖写代价」为目标，而非只看单条查询是否变快。建议构造两组表——一组 `order_id` 分区 + 本地 `user_id` 索引，一组 `order_id` 分区 + 全局 `user_id` 索引——以相同数据量、分区数、`user_id` 倾斜度运行混合 workload，分别记录写入 P99、查询 P99、事务重试、Region/LS 热点、分区 DDL 耗时与索引构建窗口。代价模型还取决于统计信息、`user_id` 倾斜、是否覆盖索引、回表列宽与并发度，故实验必须固定统计信息状态与数据分布，否则计划选择波动会污染本地/全局索引的对比。

一致性校验应包含「索引可见但行不可见」和「行可见但索引不可见」两个反向查询。TiDB 可在写入后分别用主键、本地/全局索引、全表扫描聚合结果做交叉校验，并在 DDL/backfill 期间持续更新 `user_id`，配合 `ADMIN CHECK TABLE`；OceanBase 可在全局索引 build 期间同时跑强一致读、弱一致读与 DML，对比索引表 lookup 与主表分区查询的结果集，并记录 LS leader 切换前后的查询错误码与重试行为。只在最终 `COUNT(*)` 上一致不足以证明索引路径无误，还要检查覆盖查询、回表查询、更新索引列、删除行、回滚事务、唯一冲突这些路径是否共同一致。测试数据应包含跨分区的重复 `user_id`、重复唯一键、NULL、前缀索引值与极端热点值，迁移场景还要把历史分区、空分区与新建分区都放进同一套校验脚本里。

关键观测指标 / 诊断（可查证项）：
- TiDB：用 `EXPLAIN` / `EXPLAIN ... PARTITIONS` 看是否命中全局索引、`IndexLookUp` 算子、cop task 数与 partition 信息；系统变量 `tidb_enable_global_index`（已废弃恒 ON）；`IndexInfo.is_global` 体现在表结构元信息；慢查询里的 processed keys、Region 热点、DDL job 状态。
- OceanBase：`EXPLAIN` 输出中的 `DISTRIBUTED TABLE SCAN` + `TABLE LOOKUP` 算子、`EXPLAIN ROUTE`、`GV$OB_SQL_AUDIT` 的 `plan_type` 字段（注：`GV$OB_SQL_AUDIT` 是 SQL 执行诊断视图，用于定位执行计划与路由，不等于合规意义上的 Security Audit）；`is_index_global_` 是优化器内部标志（非用户视图）；全局索引依赖 GTS，GTS 关闭则不可用。
- 除上述已由文档确认的项外，具体 Prometheus metric 名 / 内部视图名需进一步查证，不在本章臆造。

---

## 15.10 容易误解点

1. 「TiDB 所有事务都是分布式事务」——错。是否分布式取决于写集落在几个 Region：单 Region 走 1PC，跨 Region 才是完整 2PC，且 async commit 把多数派往返压到约一次。索引设计（本地把索引项压在行同分片附近 vs 全局强制跨分片）是决定写集分布的关键变量。
2. 「全局索引一定更快」——错。它只在「查非分区键 / 跨分区唯一约束」时更快；读侧省的扇出，写侧用跨分片 2PC 偿还。读多写少、查询带分区键时本地索引反而更优。
3. 「TiDB global index 是一个中心化索引服务」——错。它仍是 TiKV keyspace 中的有序 KV，只是分区表上 key 前缀用逻辑 tableID、value 携带 partition id，再由 Region/Raft Group 复制与调度，没有任何单点索引分片。
4. 「OceanBase global index 是本地索引加一个全局 flag」——错。它是独立的 index table/partitioning；local index 才与主表分区和位置强相关，global index table 与主表 partition 没有一一对应关系。
5. 「OceanBase 全局索引和 TiDB 全局索引是一回事」——部分误解。概念同源（独立分片、一键定位一行），但实现层面：TiDB 是把 partition id 编进 KV 索引项（`PartitionIDFlag`）、回表用 `PartitionHandle`；OceanBase 是一张真正独立可分区的索引表、按索引表自身 location 路由。且 OceanBase 显式把全局索引一致性绑定到 GTS，TiDB 则复用通用跨 Region 事务机制，无独立「必须开启某服务」的前提。
6. 「非分区表上的全局索引也会跨机」——错（OceanBase）。OceanBase 把非分区表上的 global index 当作 local index 存储（`*_GLOBAL_LOCAL_STORAGE`），因为非分区时这样做安全且访问性能更好。
7. 「ODP 全局索引表路由就是一致性协议」——错。ODP 只优化「找到索引表 leader」这一跳路由，内部一致性仍由 OBServer、Tablet/LS、GTS/SCN 与事务层保证；它也不是覆盖索引，不自动消除回表。

---

## 15.11 本章结论

1. 本地索引等于索引项与行同分片（写本地原子、读非分区键须扇出）；全局索引等于独立分片的一键定位结构（读省扇出、写跨分片 2PC）。这是一组对称取舍，不存在普适更优。
2. TiDB 二级索引是 SQL-on-KV 编码：行 `t{tid}_r{handle}`、索引 `t{tid}_i{idx}{列值}[+handle]`，用 key 第 10 字节区分（'i'/'r'）；回表用索引 value/key 里的 handle 拼行 key 读主表，执行计划表现为 `IndexLookup`。
3. TiDB 分区表全局索引把 partition id 编入索引项（`PartitionIDFlag=126`，`GlobalIndexVersionV1`，release-8.5 实现；V2 仅入 key 尚未实现），key 前缀改用逻辑 tableID；实验态 v8.3.0、GA v8.4.0、含全部分区列 v8.5.0、非唯一列支持 v8.5.4，`tidb_enable_global_index` 已废弃恒 ON。
4. OceanBase 全局索引是 schema 层一张独立可分区的索引表（`ObIndexType` NORMAL/UNIQUE_GLOBAL=3/4），路由按索引表自身 location 计算（`is_index_global_`→`index_table_id_`），回表经 `ObDASLookupIter`，ODP 可按 index value 做强一致读的索引表路由；非分区表上的全局索引退化为本地存储。
5. OceanBase 全局索引一致性显式依赖 GTS：主表与全局索引写入同事务、跨 LS 时走以 Log Stream 为原子参与者的树形 2PC（事务层入口 `ObPartTransCtx::commit()` 按参与者 LS 计数在 `SP_TRANS`/`one_phase_commit_()` 与 `DIST_TRANS`/`two_phase_commit()` 间分支，已源码确认到文件级），GTS 提供全局一致快照，据官方资料暗示 GTS 关闭则全局索引不可用；ODP 路由是性能优化，而非一致性协议本身。
6. 「按 `order_id` 分区、按 `user_id` 查」：若 `user_id` 是高频 OLTP 点查且选择性高，两系统都建议在 `user_id` 上建全局索引（TiDB `... GLOBAL`、OceanBase `... global`），点查命中索引分片再回表，避免扫全部分区；写入则因跨分片触发分布式事务，需以压测评估写放大。本地替代方案是把 `user_id` 做成含分区键的本地复合/唯一索引或冗余表，牺牲灵活性换写入本地化；若更看重归档与分区维护，本地索引或调整分区键可能更稳——此处建模取舍属工程推测。
7. 代价对称落在写侧与运维侧：全局索引使本地写升级为 2PC，且 `DROP/TRUNCATE/REORGANIZE PARTITION` 触发 reorg/重建（TiDB 含全局索引表还不支持 `EXCHANGE PARTITION`），分区滚动归档场景需特别评估。

---

## 15.12 参考文献

[1] TiDB Computing（行/索引 KV 编码规则）. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tidb-computing/.
 （支撑:§15.2.1 记录 key t{tid}_r{rowID}、唯一/非唯一索引 key 编码与回表 RowID 的官方权威描述。）
[2] Global Indexes（TiDB 全局索引）. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/global-indexes/.
 （支撑:§15.2.4/§15.7/§15.8 全局索引版本时间线（v7.6→v8.3 实验→v8.4 GA→v8.5.0/8.5.4）、GLOBAL 语法、聚簇/EXCHANGE PARTITION 限制、分区 DDL 代）
[3] Clustered Indexes（TiDB 聚簇索引）. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/clustered-indexes/.
 （支撑:§15.2.4/§15.8 clustered/nonclustered 主键的物理含义、NONCLUSTERED GLOBAL 与分区键约束。）
[4] Partitioning（分区表唯一键约束）. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/partitioned-table/.
 （支撑:§15.2.4「唯一键须含全部分区列、否则需全局索引（error 8264）」与聚簇约束。）
[5] TiDB 8.4.0 Release Notes（全局索引 GA）. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/release-8.4.0/.
 （支撑:§15.2.4 GA 归属 v8.4.0、tidb_enable_global_index 废弃恒 ON（逐字含「In v8.4.0, this feature becomes generally available）
[6] Indexes on partitioned tables（OceanBase 分区表索引）. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000000829644.
 （支撑:§15.3.1/§15.3.3/§15.4 本地 vs 全局索引定义、分区键须为主键子集、全局索引独立分区、本地唯一索引分区内唯一。注：该站对自动化客户端返回 HTTP 429（反爬限流），非死链，人工浏览器可达，内容由）
[7] Global index table-based routing for strong-consistency reads（ODP）. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-odp-doc-en-10000000001177667.
 （支撑:§15.3.4/§15.5 ODP 按全局索引表路由强一致读、首跳随机后建立 SQL 到索引表名映射的机制。）
[8] OceanBase Database architecture V4.3.5. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001971022.
 （支撑:§15.3.4/§15.6/§15.7 Partition/Tablet/LS、DML redo log、单 LS WAL 原子性与跨 LS 2PC 关系。）
[9] A Tree-Structured Two-Phase Commit Framework for OceanBase. 论文[EB/OL]. https://arxiv.org/html/2603.00866.
 （支撑:§15.3.4/§15.6 以 Log Stream 为原子参与者的 2PC 提交与恢复（注：论文未直接讨论全局索引跨分片写，仅用于说明提交机制）。）
[10] TiDB: A Raft-based HTAP Database. 论文[EB/OL]. https://www.vldb.org/pvldb/vol13/p3072-huang.pdf.
 （支撑:§15.2 Region/Raft Group/PD 调度作为 KV keyspace 物理分布边界的背景。）
[11] 一文详解 OceanBase 2.0 的「全局索引」功能. 官方博客，阿里云开发者社区[EB/OL]. https://developer.aliyun.com/article/672720.
 （支撑:§15.3.1/§15.3.4 全局索引是独立（可分区）索引表、利用 Paxos+改良 2PC 与全局一致快照维护主表/索引一致。）
[12] OceanBase 源码解读（7）:数据库索引实现原理. 官方博客，Alibaba Cloud[EB/OL]. https://www.alibabacloud.com/blog/599325.
 （支撑:§15.3.1 索引即独立表、本地/全局存储区别、回表、index table 构建路径。）
[13] 技术分享 | OceanBase 使用全局索引的必要性. 第三方解读，爱可生开源社区[EB/OL]. https://opensource.actionsky.com/20230411-oceanbase/.
 （支撑:§15.7/§15.8 EXPLAIN 中 DISTRIBUTED TABLE SCAN + TABLE LOOKUP、回表 RPC 代价、高并发写需压测的实测视角（第三方，以其 EXPLAIN 实测佐证官方结论）。）
[14] pingcap/tidb @ release-8.5（commit 67b4876bd57b）— pkg/tablecodec/tablecodec.go、pkg/table/tables/index.go、pkg/meta/model/index.go、pkg/planner/core/planbuilder.go. 源码[EB/OL]. https://github.com/pingcap/tidb/tree/release-8.5.
 （支撑:§15.2 索引/行前缀常量、IsIndexKey/IsRecordKey（byte 10）、PartitionIDFlag=126、IndexInfo.Global 与 GlobalIndexVers）
[15] oceanbase/oceanbase @ v4.2.5_CE（commit e7c676806fda）— src/share/schema/ob_schema_struct.h、src/sql/optimizer/ob_log_table_scan.h、src/sql/resolver/ddl/ob_create_index_resolver.cpp、src/sql/das/iter/ob_das_lookup_iter.h、src/sql/das/ob_das_location_router.h、src/rootserver/ob_ddl_operator.cpp、src/rootserver/ddl_task/ob_index_build_task.cpp. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE.
 （支撑:§15.3 ObIndexType 枚举值、默认 scope 规则、is_index_global_ 路由（get_location_table_id）、DAS 回表与位置路由、索引建表/backfill；枚）
[16] oceanbase/oceanbase @ v4.2.5_CE（commit e7c676806fda）与 4.4.x（commit d4bef8d29a4c）— src/storage/tx/ob_trans_part_ctx.cpp、src/storage/tx/ob_trans_define_v4.cpp. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE.
 （支撑:§15.3.4/§15.11 全局索引跨 LS 写入触发 2PC 的事务层入口：ObTxDesc::update_parts/update_part 累积参与者 LS，ObPartTransCtx::commi）
## 15.13 信息可信度自评

- 官方文档明确：TiDB KV 编码规则、全局索引版本时间线与限制、分区表唯一键约束、clustered 主键边界；OceanBase 本地/全局索引定义、分区键约束、ODP 全局索引表路由、Partition/Tablet/LS 关系。这些有 docs.pingcap.com / en.oceanbase.com 直接背书。
- 源码级信息（最高置信）：TiDB `tablecodec.go`/`index.go` 的前缀常量、`PartitionIDFlag=126`、`GlobalIndexVersionLegacy/V1/V2` 常量（逐字复核于 release-8.5 `pkg/meta/model/index.go`，V2 注释「not yet implemented」；该常量族属 8.4+ GA 后产物，不存在于 7.5）、`GenIndexKey`/`getPhysicalID` 回表逻辑、issue #65289，以及 OceanBase `ObIndexType` 枚举值、`get_location_table_id()` 路由表达式（逐字复核），均已对 GitHub raw 源码逐字/逐值核对，commit-pinned 到文件级。
- 论文 / 官方博客：OceanBase 2PC 树形框架、2.0 全局索引设计、源码解读系列，以及 TiDB Raft-based HTAP 论文；用于解释提交、一致性与物理分布背景。
- 已升级（源码确认）：OceanBase 全局索引写入触发 2PC 的事务层入口已从「标志位级」补到文件级——被触达 LS 经 `ObTxDesc::update_parts`（`src/storage/tx/ob_trans_define_v4.cpp`）进入参与者集合，`ObPartTransCtx::commit()`（`src/storage/tx/ob_trans_part_ctx.cpp`）按 `exec_info_.participants_.count()` 在 `SP_TRANS`/`one_phase_commit_()`（单 LS）与 `DIST_TRANS`/`two_phase_commit()`（跨 LS）间分支，4.2.5_CE 与 4.4.x（commit `d4bef8d29a4c`）逐字一致。仍按「需进一步查证」保留：全局索引在 partition split/merge/transfer 下的路由一致性维护代码仅确认到模块级（涉及 `transfer_epoch`/`intermediate_participants_` 等，详见第 10 章）。OceanBase MySQL/Oracle/Cloud 模式在默认 local/global 行为上有模式差异，本章按源码与模式边界保守书写，不把 Oracle/Cloud 默认值泛化到 CE MySQL 模式。「按 `order_id` 分区、按 `user_id` 查」的建模取舍属工程推测。
- 仍不确定：具体 Prometheus metric 名 / 内部视图名未查证，文中明确标注「需进一步查证」，未编造。OceanBase 官方文档 `en.oceanbase.com/...829644` 经核为非死链，但对自动化客户端施加站点级反爬限流（返回 HTTP 429），人工浏览器可达，其内容由阿里云开发者社区与 Alibaba Cloud 两篇博客独立佐证，自动链路检查若标红属误报。
- 版本号严格遵守锁定基准（TiDB 8.5.x 主基准对照 7.5.x / OceanBase 4.2.5_CE，4.3.5/4.4.x 仅作差异注记），`GlobalIndexVersion` 常量仅以 8.5 为准、不外推 7.5.x；全局索引相关行为以 GA（v8.4.0）及其后为前提，不向无全局索引的 7.5.x 外推。

---


# 第 16 章 HTAP / AP 架构

## 16.1 本章核心问题

HTAP（Hybrid Transactional and Analytical Processing）的难点不是「给 OLTP 数据库挂一个列存」这么简单，而是要在同一份逻辑数据、同一套事务一致性语义之下，同时服务两类诉求相反的负载。OLTP 要求低延迟、高并发的点写点读，数据布局偏向行存加索引；OLAP 要求大范围扫描、列裁剪、向量化与并行，数据布局偏向列存加压缩。两者对存储格式、内存结构、CPU 与 I/O 的诉求基本对立。把它们放进同一条数据路径，需要写入保持事务语义与复制可靠性、分析查询获得接近专用列存系统的扫描与 shuffle 能力、混合负载又不能把在线交易的延迟尾部拖垮。本章要回答的底层问题是：

1. 数据如何「既是行又是列」：同一份数据要怎样在行格式与列格式之间复制或转换，而不引入第二个数据库、不破坏事务一致性？
2. 复制通道与一致性如何叠加在共识层之上：列存副本是同步还是异步？异步如何还能「读到最新」（快照一致）？这与共识层（详见第 3 章）是什么关系？
3. 执行引擎如何在行列之间选择：优化器凭什么把一个查询的某些算子下推到列存、某些留在行存？MPP 与并行执行如何调度？
4. TP 与 AP 如何隔离：AP 的大扫描如何不拖垮 OLTP 的尾延迟？隔离是物理（独立节点、独立 zone）还是逻辑（cgroup、资源组）？

TiDB 与 OceanBase 给出了两条路线。TiDB 是组件化行列分离：TiKV 行存加 TiFlash 独立列存引擎，通过 Raft Learner 跨引擎复制。OceanBase 是一体化增强：在同一个存储引擎、同一套 PALF（Paxos-backed Append-only Log File system）日志之上，让 LSM-tree 的基线数据落成列格式、增量数据保持行格式，并尽量让 rowstore、columnstore、PX、Tenant 资源隔离与 ODP 路由都留在同一套内核与租户模型内。本章逐一梳理这两条路线的数据路径、控制路径、故障路径与代价。

需要先把「新鲜度」这件事拆开，因为它是后文一切误解的源头。TiFlash 的列存副本通过 Raft Learner 接收 TiKV Region 的 Raft 日志，写入路径不等待 Learner，但读路径会通过 Raft index 与 MVCC 校验来提供快照隔离（Snapshot Isolation）。因此它既不是外部 CDC 式的异步 ETL，也不是把 TiFlash 加入写入 quorum 后同步提交。OceanBase 的一体化路线则把代价放在另一处：版本归属、部署形态以及 CE / 企业版 / Cloud 边界更需要逐项核查。 〔文献[4]〕

> 本章与第 4 章（存储引擎）、第 11 章（MVCC 与读路径）、第 12 章（优化器）、第 23 章（shared-storage）强相关；涉及处仅标注「详见第 X 章」，不展开。

## 16.2 TiDB 的实现

### 组件与角色

TiDB 的 HTAP 是一个 multi-Raft 之上的行列双存储系统：行存是 TiKV，列存是 TiFlash。TiFlash 在官方定位中是「TiKV 的列式存储扩展」，不是独立数据库，它与 TiKV 副本共享同一逻辑身份（同一 Region）。数据面可以理解为两套物理布局服务同一套 SQL：TiKV 负责行存事务写入、点查与索引友好查询，TiFlash 负责列存扫描、聚合、Join 与 MPP 任务。

TiDB 在 KV 层用枚举区分请求目标。源码 `pkg/kv/kv.go`（release-8.5 @ `67b4876bd57b`）定义 `type StoreType uint8`，常量 `TiKV`（0）、`TiFlash`（1）、`TiDB`（2）、`UnSpecified`（255），`Name()` 返回 `"tikv"/"tiflash"/"tidb"`。识别 TiFlash 靠 engine label：`pkg/util/engine/engine.go`（同 commit）中 `IsTiFlash(store)` 判定 store 的 Labels 含 `key=="engine"` 且 value 为 `"tiflash"` 或 `"tiflash_compute"`，后者对应存算分离模式的 Compute Node。MPP 任务专指 TiFlash：`pkg/planner/property/task_type.go` 的 `TaskType` 枚举含 `RootTaskType`、`CopSingleReadTaskType`、`CopMultiReadTaskType`、`MppTaskType`，源码注释明确 `MppTaskType` 当前唯一目标是 TiFlash 节点。

### 数据路径：Raft Learner 跨引擎复制

用户执行 `ALTER TABLE t SET TIFLASH REPLICA n` 声明某表需要 n 个列存副本。该语句走 DDL job：源码 `pkg/ddl/executor.go` 的 `AlterTableSetTiFlashReplica` / `ModifySchemaSetTiFlashReplica`，对应 `model.ActionSetTiFlashReplica`，写入 `tblInfo.TiFlashReplica`（含 `Count`、`Available`）。这一步触发 PD 为相关 Region 生成形如 `"<count>/learner/engine=tiflash/host"` 的 placement rule，PD 据此把列存副本调度到带 `engine=tiflash` 标签的 store（源码 `pkg/pd` @ `6dce4a68e3e9` 的 `pkg/schedule/placement/`，`rule_manager.go` / `fit_test.go` 含 learner 加 engine 组合规则，仅确认到模块级）。复制进度可通过 `INFORMATION_SCHEMA.TIFLASH_REPLICA` 的 `AVAILABLE` 与 `PROGRESS` 观察，但要注意 `AVAILABLE=1` 表示至少一个副本完成，不等于「所有副本绝对无滞后」，这是后文故障路径的一个关键前提。

关键设计是：TiFlash 副本在 TiKV / Raft 视角是 Raft Learner。TiKV 源码 `components/raftstore/src/store/peer.rs`（release-8.5 @ `1f8a140b6d46`，约 2328-2334 行）在 unsafe recovery 选 forced leader 时跳过 Learner，注释直接点名：`// Learners like TiFlash are ignored ...`，判定 `peer.get_role() == PeerRole::Learner`。Learner 只接收日志，不投票、不计入提交多数派、不参与选举（`is_learner(&self.peer)` 在 peer.rs 多处用于排除 Learner 参与 leader 逻辑，仅确认到模块级）。

这一角色设定回答了「为什么不是普通异步复制、为什么不是同步 Follower」。这里容易被宣传话术含混带过，需分两层澄清：

- 它走的是 Raft 日志复制通道，不是数据库层的另一条异步 binlog 通道。官方文档原文："TiFlash does not rely on additional replication channels, but directly receives data from TiKV in a many-to-many manner"，并把该副本描述为 "asynchronously replicated as a special role, Raft Learner"。
- 它在 Raft 协议层对 leader 是异步的：Learner 不进多数派，leader 不必等 Learner 应答即可提交。VLDB 2020 论文原文："A learner does not participate in leader elections, nor is it part of a quorum for log replication. Log replication from the leader to a learner is asynchronous; the leader does not wait for ..."。为什么不让 TiFlash 当同步 Follower：若 TiFlash 进多数派，一旦 TiFlash 慢或宕，TiKV 凑不齐多数派，OLTP 写入可用性就被列存拖累，这与 HTAP 的隔离目标冲突。Learner 把这种耦合解除，代价转移到读路径。

### 一致性如何保证：读时 ReadIndex 对齐

异步复制并不意味着「读到脏数据」。这里的准确表述是：复制提交对 OLTP 是异步 Learner，查询可见性通过读时协议保证，而不是「最终一致列存」。TiFlash 提供与 TiKV 相同的快照隔离级别一致性，其机制落在读路径。

VLDB 论文与官方文档一致描述：当应用从 TiFlash 读某个时间戳 `ts` 的数据时，TiFlash 的 Learner 副本向其 TiKV Leader 发 `ReadIndex` 请求，Leader 据此 hold back 该读请求，直到对应 Raft log 已复制到该 Learner，再按 `ts` 做 MVCC 可见性过滤并返回。论文 4.2.4 节原文："Like follower read, learner nodes provide snapshot isolation ... the learner sends a read index request to its leaders to get the newest data that covers the requested timestamp ... the learner replays and stores the logs"。

因此 TiFlash 的「实时」和「强一致」不是靠把复制做成同步，而是靠读时阻塞等待日志追平（类似 follower read 的 ReadIndex 语义，详见第 11 章），并按事务 timestamp 做 MVCC 可见性判断。这把一致性成本从写路径挪到了读路径，且对 TiKV Leader 的额外负担很小（只是一次 ReadIndex，不需要让日志同步多数派）。

这条路径也解释了 TiFlash 的两个常见故障表象，测试开发应特别留意。其一，新建副本时先有 snapshot、后有增量 Raft log，所以 DDL 返回不等于列存副本可读；若 AP 查询过早进入，计划层可能因为 replica unavailable 或 progress 未完成而拒绝 MPP。其二，Region split / merge、leader transfer、TiFlash 节点重启都会改变 Learner 追日志与本地 apply 的节奏，但正确性不靠「查询碰巧读到最新文件」，而靠读请求携带的时间戳、Region 状态与副本进度共同约束。测试时不能只看 SQL 是否成功，还要看同一条查询在 TiKV、TiFlash cop、TiFlash MPP 三条路径下是否返回同一 snapshot 结果，以及 TiFlash 落后时究竟是阻塞、报错、warning 还是 fallback。

### 列存引擎 DeltaTree

副本落地后，数据在 TiFlash 端由专门的列存引擎承载。TiFlash 内部不是 LSM-tree，而是自研 DeltaTree 引擎：分 Delta 层（增量，序列化为 Page 存入 PageStorage）与 Stable 层（基线，不可变文件，Merge Delta 时整体替换）；列存格式类似 Parquet 的 row group / 列块，用 B+-tree 风格结构合并 Raft 更新而非 LSM；配 Rough Set Index（粗粒度索引）做过滤。

> TiFlash 列存引擎（DeltaTree / DTFile）、MPP 物理算子、MinTSO Scheduler、Pipeline Executor 的 C++ 实现位于独立仓库 `pingcap/tiflash`（本研究的 checkout 中不含该仓库的完整符号），故 DeltaTree 内部具体 C++ 类名 / 函数名无法源码级查证，上述描述据官方文档与设计文档，不编造内部符号。VLDB 论文给出过一个可量化代价：DeltaTree 的写放大（16.11）大于 LSM tree（4.74），论文称其 "acceptable"。注意方向是 DeltaTree 更高（为读优化付出的写代价），而非更优；这是 2020 年原型在论文实验台上的实测值，非生产事实。

### 执行路径：优化器选引擎、MPP 与调度

引擎选择是一个三层漏斗。第一层是 isolation-read 范围闸门：`pkg/config/config.go` 中 `IsolationRead.Engines` 默认 `["tikv","tiflash","tidb"]`，校验只允许这三值，会话或实例级控制优化器可选哪些引擎（系统变量字面名为 `tidb_isolation_read_engines`）。第二层是 CBO 代价估算：在 isolation-read 允许范围内，优化器据读代价统计自动在 TiFlash（列）与 TiKV（行）之间选择，可在一条查询里同时用两者；`READ_FROM_STORAGE(tiflash[...])` hint 可以在 engine isolation 允许的前提下对特定表施加偏好，但若强制 TiFlash 而目标表没有副本，文档要求返回错误或 warning，不能把 hint 当作无条件路由。第三层是 MPP 选择：`tidb_allow_mpp` 控制能否选 MPP，`tidb_enforce_mpp` 控制是否忽略 CBO 代价强制走 MPP，默认 `allow=on, enforce=off`；MPP 需要 tiflash 引擎，否则 planbuilder 给出字面错误 `"MPP mode may be blocked because '%v'(value: '%v') not match, need 'tiflash'."`（源码 `pkg/planner/core/planbuilder.go`）。文档进一步明确：没有 TiFlash replica、replica 未完成、算子或函数不支持 MPP 时，即使 enforce 也不能选。

MPP 是 TiFlash 从「列存 coprocessor」走向 AP 引擎的核心。查询被切成多个 query fragment，fragment 之间通过 `ExchangeSender` / `ExchangeReceiver` 做 HashPartition、Broadcast 或 PassThrough 交换（如 Join 时一表广播、另一表本地扫描）。

MinTSO Scheduler 是 TiFlash 端的 AP 查询准入调度器，处理高并发 MPP 的线程与任务调度问题：在 pipeline model 之前，每个 MPP task 可能申请多条线程，高并发下线程数随任务数线性增长，需要调度器控制 TiFlash 侧资源使用。TiDB 侧把 query 时间戳下发，`pkg/kv/mpp.go` 的 `MPPQueryID{ QueryTs, LocalQueryID, ServerID }` 中 `QueryTs` 注释为 `timestamp of query execution, used for TiFlash minTSO schedule`（仅确认到 TiDB 侧传参字段）。TiFlash 端据此实现 MinTSO 调度：Min TSO 是某 TiFlash 实例上所有运行中查询的最小 TSO，保证持有最小 TSO 的查询能被调度执行（无查询时该值为 uint64 最大值），从而避免大查询互相饿死或死锁。配置项 `task_scheduler_thread_soft_limit`（单资源组线程上限）、`task_scheduler_thread_hard_limit`（全局线程上限）、`task_scheduler_active_set_soft_limit`（并发查询数上限）。TiDB 侧还能观测到排队：执行统计含 `minTSO_wait: <ms>ms`（`pkg/util/execdetails/execdetails.go`，来源 `tiflashWaitSummary.GetMinTSOWaitNs()`）。

Pipeline 执行模型自 v7.2.0 引入（v7.4.0 GA），受 Morsel-Driven Parallelism 论文启发做细粒度 task 调度；启用 TiFlash 资源管控时自动启用，资源管控用 Token Bucket 限流加优先级调度。

### 存算分离 TiFlash(disaggregated)

TiFlash 有 classic（coupled）与 disaggregated 两条部署线。classic 模式下每个 TiFlash 节点同时承担存储与计算，本地盘保存列存数据。disaggregated 自 v7.0.0 实验性引入、v7.4.0 GA，把节点拆为 Write Node、Compute Node 与 S3 三层：Write Node 接 TiKV 的 Raft 日志、转列格式、周期性打包上传 S3，并用本地 NVMe 缓存近期写入；Compute Node 无状态，读取 Write Node 的最新未上传数据加 S3 的历史数据执行查询，可秒级独立伸缩。TiDB 侧开关 `DisaggregatedTiFlash bool`（默认 false）加 `TiFlashComputeAutoScalerType/Addr`（`pkg/config/config.go`）。

这里有一个对 8.5.x 章节尤其重要的边界：官方文档明确两种架构不能在同一 TiDB 集群共存，也不能原地互转，切换时需移除旧 TiFlash 并重新复制数据。因此 classic TiFlash 的 Learner 本地列存路径不能直接套到 disaggregated 的 S3 / cache / Write Node / Compute Node 路径上，后续讨论 TiFlash 时必须标注是哪种形态。本节只点到存算分离的 HTAP 弹性含义，与第 23 章 shared-storage 强相关。

> 关于 DXF 的澄清：DXF 与 MPP 极易混淆，须先把边界划清。TiDB DXF（Distributed eXecution Framework，v7.1.0 引入）不是 MPP 分析查询引擎。官方文档明确 DXF 调度的是 `ADD INDEX`、`IMPORT INTO`、TTL、`ANALYZE`、Backup / Restore 等会扫描大量数据的后台重任务（当前明确支持 ADD INDEX 与 IMPORT INTO），目的是把它们放进统一调度与资源管理框架，减少对 TP / AP 前台查询的冲击（变量 `tidb_enable_dist_task`，v8.1.0 起默认 ON，详见第 14 章）。因此本章里 MinTSO 与 DXF 的分工是：MinTSO Scheduler 是 AP 查询本身的调度器，DXF 是保护 AP / TP 不被后台批任务拖累的框架，二者作用层次不同，不可混为一谈。边界上还要注意：文档注明 DXF 不支持 TiDB Cloud Starter 与 Essential，因此不能把自建或 Dedicated 的 DXF 行为泛化到所有 Cloud 形态。

### 控制路径的三层分解

从控制路径看，TiDB 的 HTAP 决策分成三层，这一分层也让线上问题可被定位到具体子系统。第一层是元数据与 placement：表是否有 TiFlash replica、replica count 多少、哪些 Region 需要 Learner，由 DDL、InfoSchema、PD rule 与 TiFlash manager 协同推进。第二层是计划空间：优化器只有在 engine isolation 允许、表副本可用、算子可下推、统计信息足够可信时，才会把 TiFlash 或 MPP 作为候选。第三层是执行时调度：MPP task 的 dispatch、connection、cancel、retry、resource group 信息进入 TiDB 到 TiFlash 的请求路径。这样分层后，故障也能分解：没有副本是元数据 / placement 问题，选不到 MPP 是计划候选或算子支持问题，选到后慢则更可能是 TiFlash 执行、shuffle、线程或 I/O 问题。本地 release-8.5 中 `pkg/planner/core`、`pkg/store/copr`、`pkg/executor/internal/mpp`、`pkg/executor/mppcoordmanager`、`pkg/domain/infosync/tiflash_manager.go` 分别覆盖计划生成、cop / MPP 请求、MPP coordinator、TiFlash replica 管理与进度 / placement 逻辑（仅确认到文件与模块级，不编造函数语义）。 〔文献[1,5-10,16-17]〕

## 16.3 OceanBase 的实现

### 一体化路线：同一引擎、同一 PALF 日志

OceanBase 的 HTAP 不是「再加一个列存引擎」，而是让同一个 LSM-like 存储引擎（详见第 4 章）同时承载行与列：基线数据（major compaction 产物）可落成列格式，增量数据（MemTable、Mini / Minor SSTable）保持行格式。整套路线仍以 Tenant、Unit、Tablet、Log Stream（LS）、RootService 调度体系为基础，在同一套 SQL optimizer、PX executor、storage / blocksstable 与列式编码路径里增强 AP。OceanBase 自述其设计哲学为「all-in-one / 一体化」：一套 Paxos 日志、行列共存于同一引擎、事务 / SQL / 存储层统一，以省去 TP 与 AP 两套系统之间的数据同步成本（官方暗示）。

在此之上，需要把并行执行能力与列存能力区分开：并行执行是既有底座、非新增。OceanBase 的 PX（Parallel eXecution，以 DFO = Data Flow Operation 为调度单元）在 4.2.5 即已具备，源码 `src/sql/engine/px/`（`ob_dfo.{h,cpp}`、`ob_dfo_scheduler.{h,cpp}` 等）在 v4.2.5_CE @ `e7c676806fda` 与 4.3.5 @ `b28b9bb12f3b` 均存在。分布式执行计划据数据传输算子切成多个 DFO，DFO 间以生产者-消费者并行（PX coordinator 同时启动相邻两个 DFO）。官方文档将 DOP 定义为一个 DFO 使用的 worker thread 数，PX framework 支持手工指定 DOP 或由数据库自动选择，并特别提醒 DOP 不是越大越好，过高会增加线程调度与框架调度开销。结论是：并行与分布式执行不是 4.3 才有；4.3 新增的是列存格式、列存表、列存副本三件套，二者要区分。

### 版本演进：4.2.5 到 4.3.x 到 4.4.x

这是本章最易发生版本错配之处，以下以源码「存在 / 缺失」差异为依据逐版本厘清（参见 16.4 版本归属表）。

- 4.2.5_CE（TP LTS）：面向 TP 场景的 LTS，全 MySQL 5.7 兼容。无任何列存表 / 列存副本 / 列存参数：`ObColumnGroupType`、`WITH_COLUMN_GROUP`、`has_cs_replica`、`default_table_store_format`、`_enable_column_store` 在 v4.2.5_CE 源码中全部缺失。
- 4.3.x（AP 增强线）：
 - 4.3.0 引入列存存储引擎（列式编码、向量化引擎、基于列存的代价模型）。可按业务类型创建 rowstore table、columnstore table 或 hybrid rowstore-columnstore table：列存表语法示例使用 `WITH COLUMN GROUP(each column)`，hybrid 表使用 `WITH COLUMN GROUP(all columns, each column)` 保留行存冗余并增加列组；EXPLAIN 中可出现 `COLUMN TABLE FULL SCAN`。源码层：`enum ObColumnGroupType`（`src/share/schema/ob_schema_struct.h`）、列存 DDL 语法 `WITH_COLUMN_GROUP`（`src/sql/parser/sql_parser_mysql_mode.y`）、租户参数 `default_table_store_format`（默认 `"row"`，合法值 `row/column/compound`）与 `_enable_column_store`（默认 `True`）均在 4.3.5 存在、4.2.5 缺失。
 - 4.3.3（官方称「首个面向实时分析的 GA 版本」）引入列存副本（columnstore replica / cs_replica），提供只读列存副本的 3+1 部署形态：在原有三副本基础上新增一个独立 zone 存只读列存副本，该 zone 内所有用户表以列存格式存储，AP 连接可使用独立 ODP，通过 session 级 `ob_route_policy=COLUMN_STORE_ONLY` 在弱一致读模式访问列存副本，从而在 HTAP 下做到 TP / AP 物理资源隔离，且列存与行存可独立 major compaction。文档同时注明该能力需 ODP V4.3.2 或更高版本。源码层：`has_cs_replica`（`src/rootserver/ob_tablet_creator.cpp`）、`is_columnstore_all_server_`（`src/rootserver/ob_disaster_recovery_worker.h`）在 4.3.5 / 4.4 存在、4.2.5 缺失。
 > 成熟度须区分「版本 GA」与「特性 GA」：4.3.3 是数据库整体的 GA 版本，但列存副本这个特性本身在 4.3.3.0 / 4.3.3.1 仍是实验性，官方明确「不推荐在生产环境中应用」。官方博客《OceanBase 4.3.3 功能解析：列存副本》原文：「但在 4.3.3.0 和 4.3.3.1 这两个版本中，列存副本功能被界定为实验性质，因此并不推荐在生产环境中应用。」该特性真正进入生产可用对应 4.3.5（首个 AP LTS）。因此准确表述为：列存副本 4.3.3 引入（实验性，非生产可用），4.3.5 LTS 起生产可用，不要写成「列存副本 4.3.3 GA」。
 - 4.3.5（AP LTS）：列存能力线 LTS 化（向量索引、列存增强）；列存副本特性自此生产可用（由 4.3.3 的实验性转正）。
- 4.4.x（TP+AP 融合 LTS）：4.4.2 LTS 自述「深度融合 4.2.5 LTS 的 TP 能力与 4.3.5 LTS 的 AP 能力」，并引入向量类型 / 向量索引、hybrid query 等。但 4.4「融合」的具体自适应行列、实时列存物化等增强行为超出可逐字源码证据：相关枚举 / 参数在 4.4 分支（@ `d4bef8d29a4c`）仍在，而具体融合细节属官方暗示，须以官方文档为准。CE 边界也要保守：CE 源码可确认 `src/storage/column_store`、`src/storage/blocksstable/cs_encoding`、`src/sql/engine/px`、`src/share/vector_index`、`src/storage/vector_index` 等模块存在（另据 v4.4.2_CE @ `e859d1b9c9a6` 亦确认），但单凭目录存在不能推出所有企业版 / Cloud 特性在 CE 中生产可用。

> 列存表三种存储模式（版本须精确）：通过 `default_table_store_format`（`row`/`column`/`compound`）与建表 DDL `WITH COLUMN GROUP(...)` 控制，对应纯行存（默认）、纯列存（`each column`）、行列冗余（`compound` = `all columns, each column`）。其中 `compound` 在同一份基线里冗余存行加列，是 TP / AP 融合的存储形态。注意 hybrid 表并非所有查询都走列存：文档示例说明 optimizer 会基于成本决定 rowstore scan 或 columnstore scan，简单全表扫描的默认计划也可能偏 rowstore，因此判断 AP 能力是否生效必须看 EXPLAIN，而非只看表「有没有列存」。

### 数据、控制与读路径

- 写路径（TP）：DML 落 MemTable（行），与列存无关，以避免影响 TP、日志同步与备份恢复。这保证了列存对 OLTP 写入零侵入。
- 基线生成（控制路径）：major compaction 由 sys tenant / RootService 选全局版本号协调（详见第 4 章），把增量并入基线，基线按表声明落成行 / 列 / compound 格式。
- 读路径（AP 实时性）：由于增量恒为行格式、基线才是列格式，AP 查询需做 on-the-fly merge，把静态列存基线与动态行存增量在查询时合并，达到近 0 数据延迟（near-real-time）。这与 TiDB「列存内部自己 merge delta 到 stable」在思路上相通，区别在于 OceanBase 在同一引擎内合并，TiDB 在独立的 TiFlash 引擎内合并。
- 列存副本读路径：AP 走独立 zone 的只读列存副本，用弱一致读（详见第 11 章），与 TP 主副本物理隔离。

控制路径整体更内聚。创建 rowstore、columnstore 或 hybrid 表后，表结构、列组信息、统计信息与执行计划都留在 OceanBase schema / optimizer 体系内；PX 执行时由 SQL 层把计划拆成 DFO，再把 worker 分配到可执行的 OBServer 上。AP 查询是否影响 TP，取决于资源隔离是否真正落地：Tenant 的 Unit 规格限制 CPU 与内存基本盘，ODP 可把 AP 连接导向独立入口，只读列存副本可把部分弱一致 AP 读移出 TP 主路径，但存储 I/O、网络、merge / compaction 与缓存仍可能成为共享资源。也就是说，OceanBase 的优势不是「没有隔离问题」，而是隔离旋钮更多地内嵌在数据库自己的租户、路由与执行框架里。

> 列存副本的 Paxos 投票语义：官方文档与博客已精确公开列存副本（COLUMNSTORE，简称 C 副本）的成员语义，它与只读副本（R）同属非 Paxos 副本，不组成 Paxos 成员组、不参与选举投票、不参与日志投票，仅作为 observer 实时追日志并本地回放，因此不计入多数派 quorum、不影响事务提交延迟。官方博客《OceanBase 4.3.3 功能解析：列存副本》的副本类型对照表对 C 副本明确标注：选举投票「不参与」、日志投票「不参与」。
>
> 与 TiFlash Learner 的类比须加一处限定：二者在 learner / observer、非投票、不进多数派、读优化这一层确实同构；但一致性等级相反，TiFlash 通过 ReadIndex 加 MVCC 提供快照隔离（SI，强一致读），而 OceanBase 列存副本的 AP 查询只能走弱一致读。因此「功能上类比 TiFlash Learner」成立于成员角色，不成立于一致性模型，叙述时不可混同。

OceanBase 还有 shared-storage 一条演进线，本章只点到为止，详细留给第 23 章。Bacchus 架构用对象存储承载 LSM-tree 数据并通过服务化 PALF logging、Shared Block Cache Service 等降低计算节点状态（按版本与产品形态限定：4.3.5+ 文档、4.4.x 架构方向，社区版默认不编译）。对本章的意义是：OceanBase 的 AP / HTAP 不只在 shared-nothing 内核里加列存，也在 Cloud / shared-storage 方向继续拆分计算、缓存、对象存储与日志服务（官方暗示）。

这一对照让两条路线的差异更清楚。TiDB classic 的行列分离把 AP 能力显式放在 TiFlash 组件里，用户能比较直接地扩 TiFlash 节点、看 TiFlash 指标、把 AP SQL 绑定到 TiFlash；风险是组件间进度、计划与资源协调。OceanBase 把 AP 能力逐步下沉到表布局、PX 与租户模型，用户面对的是一套数据库但更多内部策略；风险是同一套内核里多个负载相互影响时，需要更强的诊断能力把问题拆成 optimizer、PX、storage、Tenant、ODP 几类。 〔文献[2-3,11-15,18]〕

## 16.4 核心差异对比

两条路线的分野，不在「谁有列存」，而在列存被放进系统的哪一层。下表按 HTAP 路线、复制通道、一致性、版本、并行、计划与隔离逐维度并列陈述，再在表后给出「为什么如此取舍」。

**表 16-1 行列分离（TiDB）对一体化增强（OceanBase）主对比**

| 维度 | TiDB | OceanBase | 影响 |
|---|---|---|---|
| HTAP 路线 | 组件化：TiKV 行存加 TiFlash 独立列存引擎（DeltaTree） | 一体化：同一 LSM 引擎，基线列、增量行，同一 PALF 日志，支持 row / column / compound 表 | TiDB 行列物理隔离更彻底、边界清晰；OB 省去跨系统同步但版本与形态矩阵更复杂 |
| 列存复制通道 | Raft Learner（非投票、对 leader 异步） | 同引擎 major compaction 落列加只读列存副本（独立 zone） | TiDB 列存可独立扩缩；OB 列存与行存生命周期更耦合 |
| 一致性保证点 | 读时 ReadIndex 等日志追平加 MVCC，得 SI（强一致快照） | 强读走主副本；列存副本仅弱一致读；查询时 on-the-fly merge 增量 | TiDB 列存可强一致读；OB 列存副本只能弱一致（一致性等级与 TiFlash 相反） |
| 列存引入版本 | TiFlash 随 TiDB 演进，disaggregated v7.0.0 实验性引入、v7.4.0 GA | 列存引擎 4.3.0；列存副本 4.3.3 引入（实验性）、4.3.5 LTS 起生产可用；融合 4.4.x | OB 列存是较新能力，4.2.5 完全无；列存副本「特性 GA」晚于「版本 GA」 |
| 并行 / MPP | TiFlash MPP（ExchangeSender / Receiver）加 MinTSO 调度加 pipeline | PX / DFO / DOP 并行、向量化执行，4.2.5 即有（非列存专属） | OB 并行底座更早，内嵌于 Tenant / Unit / LS 资源模型；TiDB MPP 在独立节点池、与列存绑定 |
| 计划选择 | CBO 在 TiKV / TiFlash 间选，可用 isolation 与 hint 约束 | optimizer 基于 row / column / hybrid 表统计与成本选 rowstore 或 columnstore scan | 测试时都要固定统计信息与 hint / 变量，否则计划波动会掩盖引擎差异 |
| TP / AP 隔离方式 | 物理：TiKV / TiFlash 独立节点；存算分离更进一步 | 物理（列存副本独立 zone）加逻辑（cgroup / 资源组 / IOPS / ODP 路由） | 两者都做物理隔离，OB 逻辑隔离手段更细但更依赖资源规划 |
| 写入对列存的侵入 | 零（Learner 异步，leader 不等） | 零（增量恒行存，不影响 TP 写） | 两者都把列存代价挡在写路径之外 |

表 16-1 的每一行其实指向同一句话：两者都把列存的代价挡在了写路径之外，区别只在把它转移到了哪里——TiDB 转移到读时对齐与跨组件协调，OceanBase 转移到同引擎内的合并与 compaction。代价不会消失，下表把它们逐项摊开。

**表 16-2 性能瓶颈与代价来源对比**

| 瓶颈 / 代价维度 | TiDB | OceanBase |
|---|---|---|
| 列存写放大 | DeltaTree 写放大 16.11 高于 LSM 4.74（2020 论文实测，读优化的写代价） | 增量行存加周期 major compaction 转列，compaction 抖动 |
| 数据新鲜度成本 | 读时 ReadIndex 阻塞等日志追平，复制未完成时计划可能被阻塞 | 查询时 on-the-fly merge 行增量加列基线，增量积压时变重 |
| AP 查询互相干扰 | MinTSO 调度加 pipeline 加 Token Bucket 限流防饿死 | cgroup CPU / IOPS 隔离加资源组加资源计划加 ODP 路由 |
| 存储成本 | 行加列两份（再加 TiFlash 多副本） | row / column / compound 可选；compound 行列冗余更费空间 |
| 版本 / 形态约束 | classic 与 disaggregated 不可混部；disaggregated 依赖 S3;DXF 不支持 Starter / Essential | 列存能力需 4.3.x+;4.2.5 LTS 无列存；列存副本 4.3.3 实验性、4.3.5 起生产可用；CE / 企业版 / Cloud 边界须区分 |

**表 16-3　OceanBase 列存能力版本归属矩阵**

| 能力 | 引入版本 | 生产可用起 | 备注 |
|---|---|---|---|
| 列存存储引擎（列式编码 cs_encoding / 向量化 / 列存代价模型） | 4.3.0 | — | 4.2.5_CE 完全无列存（源码级反查证，相关枚举 / 参数缺失） |
| 列存表（row / column / compound，`WITH COLUMN GROUP`） | 4.3.0 | — | `default_table_store_format` 默认 `row`；compound 行列冗余 |
| 列存副本（cs_replica，3+1 独立 zone 只读） | 4.3.3 | 4.3.5 | 4.3.3.0 / 4.3.3.1 仍为实验性，不推荐生产；「版本 GA」≠「特性 GA」 |
| 向量索引 / 列存增强（AP LTS 化） | 4.3.5 | 4.3.5 | 4.3.5 为首个 AP LTS |
| TP+AP 融合（深度融合 4.2.5 TP 与 4.3.5 AP） | 4.4.x（4.4.2 LTS） | — | 自适应行列、实时列存物化等融合细节属官方暗示 |
| 并行执行 PX / DFO / DOP（向量化执行，非列存专属） | 4.2.5 | — | 既有底座、非 4.3 新增；4.2.5_CE 即存在 |

## 16.5 正常路径图

图 16-1 刻意把两套路线并排画在同一张图里，并各自标出「行 / 列在哪里分流」与「一致性在哪一步对齐」，以便直接对照两者把代价放在何处；图后只点要点，不逐节点解说。

![[f16_1.svg]]

**图 16-1　HTAP/AP 架构正常读写/调度路径**

正常路径的共同点是 SQL 层先做代价判断，再把数据访问交给适合的物理布局。差异在于：TiDB 把行 / 列放到不同进程，优化器在引擎漏斗内选目标，列存读靠 ReadIndex 加 MVCC 对齐，PD / placement 负责把 Region Learner 调度到 TiFlash;OceanBase 把行 / 列放到同一引擎，增量行加基线列在查询时合并，ODP 与路由策略决定请求是否进入独立 zone 的只读列存副本或普通 OBServer 路径。

## 16.6 故障 / 异常路径图

图 16-2 刻意以「列存出问题时，故障如何传播」为主线，把两条路线的故障分支并列，重点不是穷举异常，而是显出「AP 失败」与「TP 受影响」之间那条并不简单的边界。

![[f16_2.svg]]

**图 16-2　HTAP/AP 架构故障/异常路径**

故障路径的测试重点不同，但有一条共同纠正：两者都不应简单写成「AP 失败不影响 TP」。TiFlash Learner 慢时，TP 写入不应因为列存副本落后而阻塞，但 AP 查询可能遇到 replica 未完成、Region unavailable、MPP 建连失败或计划 fallback;OceanBase 列存 / PX 异常更常表现为 ODP 路由策略、弱读可见性、DOP 过大、DFO 数据倾斜、Tenant 资源争抢或只读列存副本的实验能力边界。AP 查询即便不破坏 TP 正确性，仍可能占用 CPU、I/O、网络、缓存与调度资源，只是隔离边界与故障传播路径不同。

## 16.7 性能、可靠性、运维影响

HTAP 的性能成败，最终落在「TP 与 AP 同时跑时彼此干扰多少」这一个问题上。先看一组隔离实证。VLDB 2020 论文在 CH-benCHmark（TPC-C 加类 TPC-H）同时跑 TP / AP，相同 TP 客户端下增加 AP 客户端，TP 吞吐最多降 10%（原文 "degrade the TP throughput at most 10%"）；相同 AP 客户端下增加 TP,AP 吞吐最多降约 5%（原文 "at most only a 5% drop"），印证 TiKV 与 TiFlash 日志复制的高隔离性。日志复制延迟：10 warehouses 最大小于 300ms（原文 "less than 300 ms on 10 warehouses"，多数小于 100ms）；100 warehouses 多在小于 1000ms（原文 "most are less than 1000 ms"，约「one second」量级）。

> 版本与语境强约束：以上全部数字均出自 2020 年 TiDB 原始架构（TiKV 加 TiFlash DeltaTree，非存算分离）在特定实验台（CH-benCHmark、3+3 服务器、约 10 分钟跑测）下的 PingCAP 自报受控微基准，不可外推为任意生产环境事实，亦不对应本章锁定的 TiDB 8.5.x / 7.5.x、TiFlash disaggregated（v7.0.0 实验性引入、v7.4.0 GA）、OceanBase 4.x 等现代版本（论文 PDF 全文检索无这些版本号、无 disaggregated、无 OceanBase）。引用其数值时须连同原始实验条件理解。

OceanBase 性能：列存副本使列存 / 行存可独立 major compaction，适合高并发读写 HTAP。OceanBase Mercury 论文（基于 4.3.3）报告对 StarRocks / ClickHouse 在 TPC-H / ClickBench 上 1.3 倍到 3.1 倍加速、对 Doris（更新密集）约 5 倍优势，这是论文 benchmark 数据，须连同 benchmark 条件理解，不当生产排名。

性能取向上，TiFlash 对宽表少列扫描、大聚合、大 Join 更自然，TiKV 对点查、小范围索引查询更自然；CBO 的作用不是「自动永远选对」，而是依赖统计信息、表副本状态、算子支持与成本模型做折中。OceanBase 列存表与 hybrid 表也不是所有查询都应走列存，同样要看实际计划。

混跑压测时，更合理的方法是固定三组变量，否则 OLTP p99 抖动与 AP 吞吐没有可解释性。第一组是数据变量：数据规模、冷热比例、列选择比例、索引覆盖、统计信息刷新时间、TiFlash replica 是否可用、OceanBase 表形态是 row / column / hybrid。第二组是资源变量：TiKV / TiFlash / OBServer 的 CPU、内存、磁盘、网络是否隔离，TiDB resource group、TiFlash resource control、OceanBase Tenant Unit 与 DOP 是否固定。第三组是时间变量：AP 查询是否与写入高峰重叠，replica build、merge / compaction、TTL、ADD INDEX、IMPORT INTO 是否同时运行。

可用性与故障域：两者都把列存代价挡在 OLTP 写路径之外（TiDB 靠 Learner 非多数派，OB 靠增量恒行存），故列存故障不直接降低 OLTP 写可用性。TiDB 的优势是故障域清楚：TiFlash 出问题通常先影响 AP 查询或列存副本进度，TiKV quorum 仍保护 TP 写入，代价是 TiFlash 与 TiKV 的 Region / placement / DDL 生命周期要协调。OceanBase 的优势是内核路径统一，row / column / hybrid、PX、Tenant 资源都在同一系统内调度，代价是定位问题时要同时看 SQL 计划、PX DFO、Tenant 资源、ODP 路由与存储 merge / compaction 状态，故障因果更容易交织。

扩展性：TiDB disaggregated TiFlash（Compute Node 秒级伸缩加 S3）使 AP 算力弹性更强，但需重建副本；OceanBase 列存副本以「加 zone」扩 AP 容量，Bacchus 方向则在 Cloud 形态进一步解耦（条件化引用，见第 23 章）。

运维复杂度：TiDB 需运维 TiKV 加 TiFlash 两类引擎加 PD placement rule，并区分 self-managed、Dedicated、Starter / Essential 与 TiFlash classic / disaggregated;OceanBase 单引擎但需调资源组 / cgroup / 资源计划、管理 major compaction 时间窗，并区分 4.3.0 / 4.3.3 / 4.3.5 / 4.4.2 以及 CE / 企业版 / Cloud。版本维护上最容易出错的是把「能力存在」写成「所有形态默认可用」。

benchmark 数据都须连同条件读，单独拎出数字几乎必然失真。VLDB 2020 说明的是当时 TiFlash 的设计目标与实验条件，不能代表 8.5.x 绝对性能；OceanBase 4.3 / Mercury 数据须标注版本与条件；Bacchus 的 SysBench / TPC-H 是论文实验。本章不做 TiDB 与 OceanBase 的绝对排名。

## 16.8 反例与代价

HTAP 不是默认就该开的能力，列存对一部分负载是纯成本而非收益。先列不适合的场景，再列已知踩坑。

不适合的场景：

- 纯 OLTP 且无分析需求时，TiFlash 与列存副本是纯成本（多一份列存加同步开销）；OceanBase 直接用 4.2.5 TP LTS（无列存）更合适。
- 极低延迟、要求「写后立即列存可见且零等待」的 AP：TiDB 读时 ReadIndex 会引入等待，且复制未完成时计划可能被阻塞；OceanBase on-the-fly merge 在增量积压时变重。
- 不应把所有查询都强制走列存。点查、小范围索引查询、低选择性但结果很小的查询可能 TiKV 更便宜；若统计信息过旧，CBO 可能误选。强制 `tidb_enforce_mpp`、`READ_FROM_STORAGE` 或 OceanBase DOP hint 只适合可解释的测试 / 隔离场景。

已知踩坑与代价点：

- TiDB：DeltaTree 写放大（16.11）显著高于 LSM（4.74），写多读少的列存场景代价高；MPP 并发过高时 MinTSO 排队（`minTSO_wait` 飙升）致 AP 长尾；TiFlash replica 建立与重建会消耗 TiKV / TiFlash snapshot、CPU、I/O 与网络，库级批量设置 replica 是资源密集操作，文档明确中断后已执行部分不回滚、未执行部分不继续，应避开 TP 高峰；disaggregated 引入 S3 延迟、cache 命中率、Write Node 最新数据访问、两种架构不可混部等新问题，classic 经验不能无条件迁移。据社区 issue，`SET TIFLASH REPLICA 0` 历史上可能存在 PD placement rule 未清理的工程问题（推测）。
- OceanBase：major compaction 转列耗 CPU / IO，未做资源隔离时直接抬高 OLTP 尾延迟；compound（行列冗余）存储空间翻倍，不适合存储成本极敏感或写放大无法接受的业务；PX 的 DOP 过高会让线程调度、内存、临时文件与网络 exchange 成为瓶颈；增量未及时 merge 时 AP 延迟上升；只读列存副本 3+1 形态在 4.3.3 文档中是实验能力，不应作为所有生产部署默认建议。一体化不等于「AP 对 TP 零影响」（见 §16.6）。

取舍上，TiDB 用「多一个引擎加多份存储加读时对齐」换彻底的物理隔离与列存独立弹性；OceanBase 用「同引擎复杂度加 compaction 抖动加增量合并成本」换「无第二系统、无跨系统同步」。二者无优劣之分，只是把复杂度放在不同位置（与第 4、6 章一致的取舍叙事）。

## 16.9 测试开发视角的验证点

测 HTAP 的关键，不是「列存能不能跑」，而是「计划是否如预期选对引擎」与「AP 是否真没拖垮 TP」。下面按可测功能、可注入失效、压测观测三类展开。

可测功能场景：

- TiDB：同一表在无 TiFlash replica、replica 创建中、`AVAILABLE=1`、TiFlash 节点故障、statistics stale 五种状态下，比较 CBO、engine isolation、`READ_FROM_STORAGE`、`tidb_allow_mpp`、`tidb_enforce_mpp` 的计划与错误 / 告警。用 `EXPLAIN` / `EXPLAIN ANALYZE` 验证 `cop[tiflash]`、`batchCop[tiflash]`、`ExchangeSender`、`ExchangeReceiver`、`MppVersion` 是否出现；用 `INFORMATION_SCHEMA.TIFLASH_REPLICA` 观察 `AVAILABLE` 与 `PROGRESS`。对 classic 与 disaggregated 分别压测同一组 SQL，记录 S3 / cache 命中、Write Node 最新数据读取、Compute Node 扩缩容后的冷启动影响，不能把 classic 本地盘结果当作 disaggregated 结果。
- OceanBase：对 rowstore、columnstore、hybrid 表分别执行点查、宽表少列扫描、聚合、Join，用 EXPLAIN 验证 `COLUMN TABLE FULL SCAN` 或 rowstore plan；`default_table_store_format` 设 `column` / `compound` 验证默认格式；调整 DOP 观察 DFO 并发、线程、临时空间与 TP 延迟。若使用 3+1 只读列存副本，验证 ODP 版本、`ob_route_policy=COLUMN_STORE_ONLY`、弱一致读可见性、AP ODP 与 TP ODP 的连接隔离。在同一 Tenant 内与独立 Tenant / Resource Pool 内各跑一轮 AP，比较 TP p99、PX 排队、临时文件、merge 进度与 ODP 路由命中，以区分「列存扫描本身慢」与「共享资源导致 TP 受影响」。

可注入失效模式：

- TiDB：杀 TiFlash 节点验证 OLTP 写不受影响加 AP 读 hold / 回落；延迟 TiKV 到 TiFlash 复制验证 ReadIndex 等待；snapshot 限速；TiFlash replica 数超过剩余节点数；MPP 建连失败；复制落后导致 `Region Unavailable`。
- OceanBase：OBServer 节点重启；ODP location cache 失效；PX worker 资源不足；DOP 过高；触发 major compaction 同时压 TP，验证资源隔离是否守住尾延迟；列存副本 zone 故障验证 AP 回落行存；Tenant CPU / I/O 限制触发。

关键压测与观测指标：

- 压测：TP / AP 混跑下的 TP 吞吐降幅、AP 吞吐降幅、log replication delay 分布、AP P99。
- TiDB 观测：执行统计 `minTSO_wait`、`pipeline_breaker_wait`、`pipeline_queue_wait`（`pkg/util/execdetails/execdetails.go`）；TiFlash 配置 `task_scheduler_thread_hard_limit` 等；`TIFLASH_REPLICA` 系统视图；`EXPLAIN ANALYZE`、`SHOW WARNINGS`、TiDB Dashboard / Performance Overview 的 TiFlash panel。
- OceanBase 观测：`default_table_store_format` / `_enable_column_store` 租户参数；cgroup CPU `max/weight`、归一化 IOPS；`COLUMN TABLE FULL SCAN` / `COLUMN TABLE GET` 执行计划算子名；`EXPLAIN`、PX / DFO operator 信息、Tenant 资源使用、ODP 路由诊断；已查证参数 `ob_route_policy`、`PARALLEL`、`_force_parallel_query_dop`。

> 具体 Prometheus metric 裸名、内部视图字段名除上文已查证者外需进一步查证，不在此编造；若要落到告警规则，应以当前集群 dashboard JSON 或官方监控文档为准。

## 16.10 容易误解点

1. 「TiFlash 是异步复制所以读到的是旧数据」是错的。它在 Raft 协议层对 leader 异步（不进多数派），但读路径用 ReadIndex 等日志追平加 MVCC 可见性判断，提供与 TiKV 相同的 SI 一致性；副本没追上或不可用时，查询会被阻塞、报错或在允许时 fallback。异步是写路径解耦，一致是读路径保证，二者不矛盾。
2. 「MPP 开了就一定更快」是错的。MPP 适合大扫描、大聚合、大 Join，小查询可能被 Exchange、调度、建连与线程开销拖慢；`tidb_enforce_mpp` 与 OceanBase DOP hint 都是实验与调优工具，不是普遍优化开关。
3. 「OceanBase 4.x 一直有列存」是错的。列存表 / 列存副本 / 列存参数三件套是 4.3.x 引入（4.3.0 列存引擎、4.3.3 引入列存副本），4.2.5_CE 源码中完全缺失；而 PX / DFO 并行执行是 4.2.5 就有的既存能力，不要把「并行」也归到 4.3（见 §16.3）。
4. 「4.3.3 是 GA 版本所以列存副本特性也 GA 了」是错的。要区分「版本 GA」与「特性 GA」：4.3.3 确是数据库整体的 GA 版本，但列存副本特性在 4.3.3.0 / 4.3.3.1 仍是实验性、官方不推荐生产，该特性生产可用对应 4.3.5。写「列存副本 4.3.3 GA」是把两者混为一谈。
5. 「列存副本的 Paxos 投票语义官方没公开」是错的。官方文档与博客已精确公开：列存副本（C）是非 Paxos 副本，不参与选举投票、不参与日志投票、不计入多数派。其与 TiFlash Learner 仅在「非投票 observer」层同构，一致性等级相反（TiFlash SI 强读对列存副本弱读）。
6. 「TiDB DXF 是 AP 的 MPP 引擎」是错的。DXF 调度的是 ADD INDEX / IMPORT INTO 等后台批任务，目的是不干扰 TP / AP 在线负载；AP 查询的真正调度器是 TiFlash 端的 MinTSO Scheduler，二者层次不同。
7. 「OceanBase 4.4.2 的所有 AP / AI 能力都等同于 CE 中可用的源码实现」是错的。官方博客可支持版本演进方向，本地 CE 源码可确认模块存在，但企业版 / Cloud / CE 的生产可用边界必须以对应版本文档与 release notes 为准（官方暗示）。

## 16.11 本章结论

1. 路线分野：TiDB 是组件化行列分离（TiKV 行加 TiFlash 列，Raft Learner 跨引擎复制），隔离边界清晰但副本生命周期与计划选择更复杂；OceanBase 是一体化增强（同引擎、同 PALF，基线列加增量行），运维面更内聚但版本与形态矩阵更复杂。
2. TiFlash 用 Raft Learner 而非同步 Follower，也不是外部异步 ETL：Learner 不进多数派、对 leader 异步，把列存故障与 OLTP 写可用性解耦；读一致性靠读时 ReadIndex 等日志追平加 MVCC 提供 SI，因此必须同时测试复制滞后与读时可见性。
3. TiFlash disaggregated v7.0.0 实验性引入、v7.4.0 GA，拆为 Write Node 加 Compute Node 加 S3，与 classic / coupled 不可混部或原地切换；8.x 讨论 TiFlash 时必须标注是哪种形态。
4. OceanBase 列存版本归属须精确：列存引擎 4.3.0、列存副本 4.3.3 引入（实验性、不推荐生产）且 4.3.5 LTS 起生产可用、TP / AP 融合 4.4.x;4.2.5_CE 完全无列存（源码级反查证），而并行执行 PX / DFO 4.2.5 即有。须区分「4.3.3 版本 GA」与「列存副本特性 GA」，不可简写为「列存副本 4.3.3 GA」；CE / 企业版 / Cloud 边界不能省略。
5. 执行调度分不同层次：TiDB AP 调度是 TiFlash MinTSO Scheduler（保最小 TSO 查询、防饿死），DXF 只管后台批任务隔离；OceanBase 靠 cgroup / 资源组 / IOPS / ODP 做逻辑隔离加列存副本独立 zone 做物理隔离。两套 MPP / PX 的瓶颈常来自 shuffle / DFO 调度、线程、内存、临时文件、数据倾斜与资源隔离，而非单纯「列存是否存在」。
6. 隔离实证有量化但勿外推：VLDB 2020 CH-benCHmark 下 TP 降不超过 10%、AP 降不超过 5%，日志复制延迟 10 仓小于 300ms、100 仓小于 1000ms，均为 2020 年原始架构特定 benchmark 条件下的 PingCAP 自报实测，非任意生产事实、亦不对应本章锁定的现代版本。
7. 列存副本成员语义已明确：OceanBase 列存副本只读、弱一致、独立 zone，官方已精确公开其为非 Paxos 副本、不参与选举 / 日志投票、不计入多数派；它与 TiFlash Learner 在「非投票 observer」层同构，但一致性等级相反，类比须限定在成员角色层。

## 16.12 参考文献

[1] TiDB: A Raft-based HTAP Database. 论文，VLDB 2020[EB/OL]. https://vldb.org/pvldb/vol13/p3072-huang.pdf.
 （支撑:§16.2 中 Raft Learner 异步复制不进多数派、读时 ReadIndex 加 MVCC 提供 SI、DeltaTree 写放大 16.11 对 LSM 4.74，以及 §16.7 中 TP / AP 混跑）
[2] Building an OceanBase-based Distributed Nearly Real-time Analytical Processing Database System（OceanBase Mercury）. 论文，arXiv 2026[EB/OL]. https://arxiv.org/html/2602.07584v1.
 （支撑:§16.3 / §16.7 中 OceanBase 基线列、增量行、on-the-fly merge 近实时，以及对 StarRocks / ClickHouse / Doris 的 TPC-H / ClickBench）
[3] OceanBase Bacchus: a Cloud-Native Shared Storage Architecture. 论文，arXiv 2026[EB/OL]. https://arxiv.org/abs/2602.23571.
 （支撑:§16.3 / §16.7 中 OceanBase shared-storage / Bacchus 演进方向与 benchmark 条件限定（详见第 23 章）。）
[4] HTAP Databases: A Survey. 论文，arXiv 2024[EB/OL]. https://arxiv.org/pdf/2404.15670.
 （支撑:§16.1 / §16.4 中把 TiDB 归入「分布式行存加列存副本（Raft learner）」架构类别的独立学术佐证（注：该综述未讨论 OceanBase）。）
[5] TiFlash Overview. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tiflash-overview/.
 （支撑:§16.2 中 TiFlash 为 TiKV 列存扩展、Raft Learner 异步复制、不依赖额外复制通道、提供 SI、优化器据代价选 TiKV / TiFlash。）
[6] Use TiDB to Read TiFlash Replicas. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/use-tidb-to-read-tiflash/.
 （支撑:§16.2 / §16.9 中 CBO、engine isolation、READ_FROM_STORAGE hint 与 fallback / 错误边界、TIFLASH_REPLICA 的 AVAILABLE）
[7] Use TiFlash MPP Mode. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/use-tiflash-mpp-mode/.
 （支撑:§16.2 中 MPP fragment 切分、ExchangeSender / Receiver、tidb_allow_mpp / tidb_enforce_mpp 与阻塞条件。）
[8] TiFlash MinTSO Scheduler 与 Configure TiFlash. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tiflash-configuration/.
 （支撑:§16.2 中 MinTSO Scheduler 的 task_scheduler_thread_soft_limit / hard_limit / active_set_soft_limit 与最小 TSO）
[9] TiFlash Disaggregated Storage and Compute Architecture and S3 Support. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tiflash-disaggregated-and-s3/.
 （支撑:§16.2 中 Write Node 加 Compute Node 加 S3 存算分离（v7.0.0 实验性引入、v7.4.0 GA）、秒级弹性与 classic / disaggregated 不可混部限制。）
[10] TiDB Distributed eXecution Framework （DXF）. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tidb-distributed-execution-framework/.
 （支撑:§16.2 / §16.10 中 DXF 调度 ADD INDEX / IMPORT INTO 等后台任务、tidb_enable_dist_task、Starter / Essential 不支持边界，澄清 DXF）
[11] Columnar storage | OceanBase Database V4.3.x. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001717168.
 （支撑:§16.3 中 rowstore / columnstore / hybrid 表、WITH COLUMN GROUP、COLUMN TABLE FULL SCAN、PX / DOP 定义与「DOP 非越大越好」）
[12] OceanBase 4.3.3 功能解析：列存副本. 官方博客，CSDN 官方账号 OceanBaseGFBK[EB/OL]. https://blog.csdn.net/OceanBaseGFBK/article/details/143581723.
 （支撑:§16.3 中两处：列存副本在 4.3.3.0 / 4.3.3.1 为实验性、不推荐生产（非「特性 GA」）；列存副本（C）为非 Paxos 副本、选举投票与日志投票均「不参与」。作为 en.oceanbase.com）
[13] OceanBase 4.3.3 Released | The first GA version for real-time analysis. 官方博客[EB/OL]. https://en.oceanbase.com/blog/15464061184.
 （支撑:§16.3 中 4.3.3 为首个面向实时分析的版本、引入列存副本（独立 zone、只读、弱一致、ob_route_policy=COLUMN_STORE_ONLY、需 ODP V4.3.2+、TP / AP 物理隔）
[14] Why is resource isolation important for HTAP?. 官方博客[EB/OL]. https://oceanbase.github.io/docs/blogs/feat/resource-isolation.
 （支撑:§16.3 / §16.7 / §16.8 中 cgroup CPU（max / weight）、归一化 IOPS、资源组与资源计划的 TP / AP 逻辑隔离。）
[15] The Story Behind OceanBase's Integrated Architecture / OceanBase 4.4.2 LTS 发布说明. 官方博客[EB/OL]. https://oceanbase.github.io/docs/blogs/arch/all-in-one.
 （支撑:§16.3 中一体化 / all-in-one 设计哲学（一套 Paxos 日志、行列共存、省去跨系统同步）与 4.4.2 LTS 融合 4.2.5 TP 加 4.3.5 AP 的版本演进口径。）
[16] pingcap/tidb 仓库（release-8.5 @ 67b4876bd57b；pd @ 6dce4a68e3e9）. 源码[EB/OL]. https://github.com/pingcap/tidb.
 （支撑:pkg/kv/{kv,mpp}.go、pkg/util/engine/engine.go、pkg/planner/property/task_type.go、pkg/config/config.go、p）
[17] tikv/tikv 仓库（release-8.5 @ 1f8a140b6d46）. 源码[EB/OL]. https://github.com/tikv/tikv.
 （支撑:components/raftstore/src/store/peer.rs。支撑 §16.2 中 TiFlash 副本为 Raft Learner、注释 "Learners like TiFlash are ign）
[18] oceanbase/oceanbase 仓库（v4.2.5_CE @ e7c676806fda；4.3.5 @ b28b9bb12f3b；4.4.x @ d4bef8d29a4c）. 源码[EB/OL]. https://github.com/oceanbase/oceanbase.
 （支撑:src/share/schema/ob_schema_struct.h、src/sql/parser/sql_parser_mysql_mode.y、src/rootserver/ob_tablet_creat）
## 16.13 信息可信度自评

- 官方文档明确：TiFlash 为 TiKV 列存扩展、Raft Learner 异步复制、读时 ReadIndex 加 MVCC 提供 SI、MPP 变量与算子、MinTSO 配置项、pipeline 模型、disaggregated v7.0.0 实验性引入与 v7.4.0 GA、classic 与 disaggregated 不可混部、DXF 调度对象与 Cloud 边界；OceanBase V4.3.0 列存表、4.3.3 列存副本引入（实验性）与 4.3.5 起生产可用、列存副本为非 Paxos 非投票成员、三种存储模式、PX / DOP、资源隔离手段。这些有官方文档或官方博客直接支撑。
- 论文级信息：Learner 不进多数派与异步复制的协议描述、DeltaTree 写放大 16.11 对 LSM 4.74、TP / AP 混跑 10% / 5% 降幅、日志复制延迟（10 仓小于 300ms、100 仓小于 1000ms）（VLDB 2020）；OceanBase 近实时 merge 与加速比（Mercury，基于 4.3.3）；shared-storage 演进（Bacchus,2026）；TiDB 架构归类（HTAP Survey,2024）。这些论文数据均为各自论文特定 benchmark 条件下的受控实验结果，不外推生产、不绑定本章锁定的现代版本。
- 源码级信息：TiDB 侧 StoreType / engine label / MppTaskType / isolation-read / QueryTs / SET TIFLASH REPLICA / minTSO_wait 与 placement 相关模块；TiKV 侧 peer.rs Learner 注释；OceanBase 侧列存三件套与 PX 的 4.2.5 / 4.3.5 / 4.4 存在性差异，均 commit-pinned。TiFlash 列存引擎 C++ 内部符号因仓库未完整 checkout 而无法源码级查证，相关描述据官方文档与设计文档，不编造内部符号。
- 工程推测与不确定：4.4「融合」的具体自适应行列、实时物化行为（标（官方暗示），以官方文档为准）；TiDB `SET TIFLASH REPLICA 0` 的 placement rule 清理工程问题（据社区 issue，标（推测））。OceanBase 列存副本的 Paxos 投票语义已确认官方公开（非 Paxos、不投票），不再标「需进一步查证」。
- 反查证记录：其一，HTAP Survey 经直读原文确认未讨论 OceanBase，故仅用其佐证 TiDB 架构归类，不引用其对 OceanBase 的「对比」。其二，多处来源把 TiFlash 描述为「异步复制」易误读为弱一致，经 VLDB 论文 4.2.4 与官方文档交叉验证，确认协议层异步加读路径 ReadIndex 对齐等于 SI。其三，部分资料曾写作「列存副本 4.3.3 GA」，经官方 CSDN 博客反查证订正为「4.3.3 引入（实验性）、4.3.5 起生产可用」，区分版本 GA 与特性 GA。其四，列存副本投票语义「官方未公开」为误，经副本类型对照表反查证订正为非 Paxos、非投票、不计入多数派。其五，VLDB 100 仓日志延迟部分资料曾误写小于 1500ms，经直读论文 PDF 全文核验，原文为「most are less than 1000 ms」，已订正为小于 1000ms 并强化语境约束。
- 版本核查备注：本章版本号（TiDB 8.5.x / 7.5.x、OceanBase 4.2.5 / 4.3.0 / 4.3.3 / 4.3.5 / 4.4.x）与公共头锁定一致。唯一需注明处是 TiFlash disaggregated：经直读 PingCAP 官方 Release Notes 核对，官方原文为 v7.0.0 实验性引入（experimental）、v7.4.0 GA，与第 7、23 章对齐，本章已据官方按此口径统一表述。

---


# 第 17 章 多租户与资源隔离

## 17.1 本章核心问题

分布式数据库要在同一套物理资源上承载多个互不信任、互相争抢的工作负载时，必须回答一个根本问题：当 A 业务突然刷写百万行、跑一个全表扫描、或者触发后台合并(compaction/merge)时，B 业务的 P99 延迟会不会被拖垮?这就是"噪声邻居(noisy neighbor)"问题。围绕它，系统需要在 CPU、内存、IO、磁盘空间、日志盘、后台任务这些维度上做隔离，并进一步决定：租户是否拥有独立的元数据、独立的备份恢复边界、独立的副本放置、独立的运维边界。

本章核心问题不是"能不能限流"，而是隔离落在哪一层、用什么机制、隔离到什么强度，以及边界是否一路穿透到调度、备份恢复、故障恢复和诊断。多租户从来不是"多建几个 database/schema"那么简单。两条技术路线代表了两种哲学：

- **OceanBase 的原生多租户**：把 Tenant 做成数据库内核的一等公民。每个用户租户拥有独立的内存空间、独立的工作线程池、独立的 Log Stream(LS)/日志盘、独立的内部元数据(下沉到配对的 meta 租户)，并叠加 cgroup 子树与租户级三参数 IO 调度(语义借鉴 mClock)。它更接近**硬隔离(hard isolation)**,Tenant 因此更像"数据库实例级边界"；代价是租户成为系统第一复杂度来源——schema、事务、日志、合并、备份、诊断全部要按租户切分。
- **TiDB 的 Resource Control**：在共享的 TiDB/TiKV 进程之上，用 RU(Request Unit)令牌桶 + TiKV 优先级调度 + Runaway 规则实现**软隔离(soft isolation)**，并以 Placement Rules、TiDB Node Group 等拓扑能力作补充。它不给租户独立进程、独立内存空间，而是把"一份共享底座 + 流控/优先级"做到极致，Resource Group 更像"共享集群内的资源配额与优先级边界"，换取弹性与运维简单。

理解这条分界线，需区分"资源限流"与"原生多租户"——后者还涉及元数据、备份、副本放置、运维边界。本章逐维度拆解。**结论是，TiDB 的 Resource Control 不等价于 OceanBase 的原生多租户。**两侧都能服务 SaaS 或多业务整合，但抽象层次不同，这是全章的主线。多租户与元数据控制面强相关(详见第 13 章)，与安全/权限边界相关(详见第 21 章)；本章只聚焦资源隔离与租户模型本身，不替这两章展开"谁能访问什么数据"。

本章还要避免一类高风险混写：把 Cloud 产品层文案直接当作自建内核实现。TiDB Cloud Starter/Essential 的内部多租户隔离机制没有公开到内核实现层，公开资料只说明 Starter 是 fully managed multi-tenant offering、资源由 serverless 技术共享，并提到 Dedicated 提供 isolated infrastructure/resources 的升级路径；后者究竟如何落地，公开信息有限、尚不确定。OceanBase 侧的 cgroup 资料同样要分版本与形态(见 §17.3、§17.10)。

> 本章版本基准遵循公共头锁定：TiDB 8.5.x LTS(对照 7.5.x),TiKV/PD release-8.5;OceanBase v4.2.5_CE(TP)、4.3.5(AP)、4.4.x(融合，4.4.1_CE 2025-10-24)。源码引用 commit-pin 到统一基线：OceanBase v4.2.5_CE @ `e7c676806fda`、OceanBase 4.4.x @ `d4bef8d29a4c`、TiDB release-8.5 @ `67b4876bd57b`。

## 17.2 TiDB 的实现

与 OceanBase 把租户做成内核一等公民不同，TiDB classic 的多业务共享路径可以拆成三层：SQL 层的资源组与流控、RU 计量模型、数据放置与拓扑隔离。它的资源管理单位不是"租户"，而是 **Resource Group**。

### 17.2.1 第一层：Resource Group 与 RU(Request Unit)模型

计量单位是 **RU(Request Unit)**——CPU、IOPS、IO 带宽的统一抽象，表示一次数据库请求消耗的资源量。用户按 RU/s 设定配额，而不是直接声明"这个组占几核 CPU、几 GB 内存、几块盘"。

源码层面，`type ResourceGroupSettings struct`(`pkg/meta/model/resource_group.go`,release-8.5 @ `67b4876bd57b`,43 行)字段含 `RURate`(json `ru_per_sec`)、`Priority`、`CPULimiter`(json `cpu_limit`)、`IOReadBandwidth`、`IOWriteBandwidth`、`BurstLimit`、`Runaway`，默认 `Priority = MediumPriorityValue`。RU 落地为 PD resource_manager 的令牌桶：`RU_PER_SEC` 即令牌桶填充速率 `FillRate`,`BURSTABLE` 即 `BurstLimit`(`pkg/ddl/resourcegroup/group.go:80–84`)。同一条链路上，`pkg/sessionctx/variable/tidb_vars.go` 定义系统变量 `tidb_enable_resource_control`,`pkg/resourcegroup/runaway/` 处理 Runaway 记录与 watch，说明 Resource Group 是 TiDB 元数据与执行路径的一部分，而非外挂标签。

RU 消耗系数(官方文档)：读 2 个 storage read batch = 1 RU、8 个 read request = 1 RU、64KiB payload = 1 RU；写 1 个 write batch = 1 RU、1 个 write request = 1 RU、1KiB payload = 1 RU(写按副本数分别计，默认 3 副本);CPU 3ms = 1 RU。需注意，RU 读写成本的标定常量位于 PD 侧、TiDB 仓库内仅消费 `rmpb`：上述每一项系数都精确落地在 PD `pkg/mcs/resourcemanager/server/config.go`(release-8.5 @ `6dce4a68e3e9`)的默认常量——`defaultReadBaseCost = 1./8`(8 read request=1RU)、`defaultReadPerBatchBaseCost = 1./2`(2 read batch=1RU)、`defaultWriteBaseCost = 1`、`defaultWritePerBatchBaseCost = 1`、`defaultReadCostPerByte = 1./(64*1024)`(64KiB=1RU)、`defaultWriteCostPerByte = 1./1024`(1KiB=1RU)、`defaultCPUMsCost = 1./3`(3ms CPU=1RU),由 `RequestUnitConfig.Adjust()` 注入。

控制路径分两层:① TiDB 层 SQL 流控(token bucket)，负载超配额时排队或降优先级；② TiKV 层优先级调度(`PRIORITY` 高者先，未设则按 `RU_PER_SEC` 比例)。系统变量 `tidb_enable_resource_control`(TiDB)、`resource-control.enabled`(TiKV)自 v7.0.0 起默认开；TiFlash 侧 `enable_resource_control` 自 v7.4.0 引入；功能整体 v7.1 LTS 转 GA。这里的"隔离"是流量进入系统前的准入、优先级与限流，是一种 flow-based 的软隔离：当某组负载超过配额时，它排队或降优先级，其他组不必因同一共享集群内的噪声邻居而完全失控。

用户经 `CREATE USER ... RESOURCE_GROUP` / `ALTER USER` 绑定到组，会话默认继承登录用户的组；未绑定者落 `default` 组(`RU_PER_SEC = UNLIMITED` 即 INT 最大值 2147483647、`BURSTABLE`)。`CALIBRATE RESOURCE` 可经 TPC-C/sysbench 基准或实际负载估算集群总 RU 容量(仅 Self-Managed 可用，TiDB Cloud 不可用；时间窗 10m–24h)。

### 17.2.2 Runaway Query：噪声邻居的 SQL 级防护

RU 限的是稳态速率，Runaway 规则补足的是另一条故障路径——单条失控 SQL。其实现见 `pkg/resourcegroup/runaway/checker.go`(release-8.5 @ `67b4876bd57b`)。三类阈值：`ExecElapsedTimeMs`(超时)、`RequestUnit`(超 RU)、`ProcessedKeys`(Coprocessor 处理 key 数)；动作枚举 `rmpb.RunawayAction` 含 `DryRun`/`NoneAction` 等，并支持 watch list(`examineWatchList`、`markedByQueryWatchRule`)。

文档侧对应：`QUERY_LIMIT` 含 `EXEC_ELAPSED`/`PROCESSED_KEYS`/`RU`;`ACTION` 含 `DRYRUN`(v7.2.0)、`COOLDOWN`(降至最低优先级)、`KILL`、`SWITCH_GROUP`(v8.4.0);`WATCH` 列表匹配 `EXACT`/`SIMILAR`/`PLAN`(v7.3.0)。粒度在 **SQL/group 级，而非 OS 级硬隔离**；它能切断超预期 SQL，但不是 Tenant 级故障域，且属事后纠偏，已消耗资源不可回收。

### 17.2.3 第三层：数据放置与拓扑隔离、后台任务

RU 与 Runaway 解决"流量怎么进、失控怎么切"，数据放置解决"数据落在哪"。Placement Rules 可让 PD 对连续 key range 的副本数、位置、host 标签、leader 角色等产生调度约束；TiDB Cloud Dedicated 还公开了 TiDB Node Group 这类计算节点物理分组能力。它们能把某些库表或 workload 放到指定存储/计算拓扑上，但仍是 table/database/range 或节点组层面的组合能力，而不是一个同时拥有独立 schema 命名空间、资源池、Unit、Locality、日志盘、租户级备份恢复与租户级诊断入口的原生 Tenant 对象。

后台任务隔离独立成一档：`tidb-resource-control-background-tasks` 用 `BACKGROUND=(TASK_TYPES='br,ddl', UTILIZATION_LIMIT=30)` 配置，`TASK_TYPES` 含 `lightning`/`import`/`br`/`ddl`/`stats`,`UTILIZATION_LIMIT`(0–100,v8.4.0)限制后台任务在每个 TiKV 节点的资源占比上限；当前所有 group 的后台任务统一绑到 `default` 组管理。RU 可观测方面，`RUStatsWriter`(`pkg/domain/ru_stats.go:44`)定期把各 group 的 RU 历史写入 `mysql.request_unit_by_group`(v7.6.0 引入)。

### 17.2.4 数据路径与控制路径

从**数据路径**看，一条 SQL 先在 TiDB Server 完成解析、优化和执行计划构造，Resource Group 名称在 session 或 statement context 中确定；随后 TableReader、IndexReader、IndexLookUp、PointGet、Coprocessor 等路径把请求发往 TiKV 或 TiFlash,KV 请求携带 Resource Group tag 供下游调度与统计。这个过程中 Resource Group 主要影响请求进入下游前的准入、速率与优先级，而 Region 的 leader、副本位置、MVCC 可见性和事务提交仍遵循 TiDB/TiKV 原有路径。也就是说，同一 Resource Group 下的不同表可以落在不同 Region、不同 Store；同一 Region 所在 Store 也可能同时服务多个 Resource Group。

从**控制路径**看，Resource Group 的创建、修改、删除属于 TiDB 元数据变更，最终进入 InfoSchema/Domain 可见范围，并被 SQL 执行器、权限检查、session 变量和慢日志/TopSQL 等路径消费。它不像 OceanBase `CREATE TENANT` 那样触发 Unit 规划、Resource Pool 绑定和租户级服务初始化。运维上要把两类动作分开：调整 TiDB Resource Group 是调整共享集群中的"交通规则"，通常更轻、回滚更快；调整 OceanBase Tenant/Unit 是调整某个数据库服务的"资源容器"，影响面更深，必须同时核对资源池、Zone、Locality 与后台任务。

### 17.2.5 TiDB 的工程定位：soft isolation

综合上述源码与文档可归纳：TiDB Resource Group 实现于"同一组共享进程/TiKV 节点"之上，经 RU 令牌桶 + Runaway + TiKV 优先级调度做**软隔离**。`ResourceGroupSettings` 虽含 `CPULimiter`/`IOReadBandwidth`/`IOWriteBandwidth`，但其底层是节点内调度配额，**非**独立 Unit/进程/内存空间。TiDB 没有"独立内存空间/独立线程池/独立日志盘/独立内部元数据"这套租户对象；将这一点定位为与 OceanBase 原生多租户的根本差异，属本章基于源码的工程推测。官方博客逐字支撑的是 "workload consolidation"(把多应用合并到单一后端)与 v7.1 GA 事实，并未逐字出现 "soft isolation" 或 "非完整多租户"，后二者同属本章对源码与官方资料的归纳定位。

### 17.2.6 TiDB Cloud Starter/Essential(闭源，谨慎)

TiDB Cloud Starter(原 Serverless,2025-08 改名)是**共享底座多租户**形态。官方文档说 Starter 是多租户共享底座，资源由 serverless 技术共享；FAQ 说若需要 isolated infrastructure and resources 应升级 Dedicated。据此只写用户可见策略：Starter 通过托管服务、TLS、静态加密、自动升级和计费/配额体验承载小规模/弹性 workload;Essential 据现有公开资料看属更高一档的 Cloud 形态，但其具体定位尚不确定。

第三方解读(Medium)进一步描述其数据物理隔离(每租户独立 LSM-Tree SST 文件)、元数据隔离(PD 从集中式演进为 disaggregated PD)、经 TiProxy SQL 网关按租户路由、空闲 TiDB 实例可回收复用，这些均属基于第三方资料的推测。相关细节未在 release-8.5 开源代码中体现，属闭源 Cloud 层，本章**不做实现层断言**，不可引用为生产事实；其内部是否使用 Keyspace、哪些 TiKV/PD 资源池共享、如何做热点租户迁移，公开资料不足。不能把 TiDB Keyspace、Resource Control 或 Cloud 多租户实现强行等同。

异常路径也对应这条抽象边界：如果 Resource Group 配额太小，正常 SQL 不会被"租户故障隔离"切断，而是排队、降优先级或被 Runaway 规则处理；如果一个业务的数据热点集中在某些 Region,Placement Rules 只能约束副本位置，不能自动把该业务变成独占存储实例；备份恢复发生在共享集群中时，BR/快照恢复按库表、集群和服务能力工作，而非 Resource Group 级备份恢复。 〔文献[1-4,12-14]〕

## 17.3 OceanBase 的实现

与 TiDB 把隔离做成共享底座上的一层标签不同，OceanBase 的多租户模型更靠近内核主干：它把资源抽象自底向上拼装，**不依赖 Docker 或虚拟机**，而在数据库进程内以更轻量的方式实现隔离。这一论断有论文逐字出处：VLDB 2022 论文 §2.5.3 Resource Isolation(p.3388)原文 "a Resource Unit ... includes CPU, Memory, IOPS, Disk Size, and Session Number" 且 "OceanBase does not rely on Docker or virtual machine technology, and it implements resource isolation within the database ... the tenant isolation of OceanBase database is more lightweight"。

### 17.3.1 租户模型：Tenant → Resource Pool → Unit → Zone → Locality

![[f17_1.svg]]

**图 17-1　OceanBase 租户资源容器层级**

层级关系如下：

- **Unit**：最小资源分配单位，聚合 CPU/内存/Log Disk/IOPS 等全部维度，一个 Unit 只能落在一个 OBServer 上；同一 Tenant 在同一 Server 上至多一个 Unit。
- **Unit Config(unit_config)**:Unit 的资源规格模板(`max_cpu`/`min_cpu`/`memory_size`/`log_disk_size`/`max_iops`/`min_iops`/`iops_weight`)。
- **Resource Pool**：由多个 Unit 组成，绑定一组 Zone；一个 Resource Pool 只能授予一个租户，一个租户可含多个 Pool。
- **Tenant**：引用一个或多个 Resource Pool，设置 `primary_zone`、`locality`、兼容模式(MySQL/Oracle)等；它是独立的 database service。
- **Zone / Locality**:`zone_list` 决定副本分布的可用区，副本数 = Zone 数；`locality` 描述各 Zone 的副本类型与分布，`primary_zone` 决定 Leader 优先落点。

创建业务 Tenant 的路径通常是：在 `sys` tenant 下创建 Resource Unit 配置，创建 Resource Pool 指定 `unit_num`、`zone_list`，再创建 Tenant 并关联 `pool_list`、`locality`、`primary_zone`。这意味着 Tenant 的可用资源不是单纯 SQL 层标签，而是与 Zone、Unit 放置和副本拓扑一起进入 RootService/Unit 管理路径。源码层面，`ObResourcePool`(`src/share/unit/ob_resource_pool.h`,v4.2.5_CE @ `e7c676806fda`)的字段 `name_`、`unit_count_`、`unit_config_id_`、`zone_list_`、`tenant_id_`(41–45 行)精确落地了 Pool→Unit(config)→Zone→Tenant 的映射，`is_granted_to_tenant()`(37 行)判断池是否已授予某租户。

### 17.3.2 sys / meta / user 三层租户：tenant_id 的奇偶编码

OceanBase 把租户分三层，这是其多租户复杂度的根，也是本章最关键的分界：

- **sys 租户**(系统租户)：存集群级元数据，运行 RootService、schema service、balancer 等控制面，负责资源创建、租户创建与维护(详见第 13 章)。
- **user 租户**：应用可见的"独立数据库实例"，拥有自己的账号、database、表、系统变量与事务/存储上下文。
- **meta 租户**：每创建一个 user 租户，系统自动配对创建一个 meta 租户，生命周期与 user 租户一致，用于存放该用户租户的**集群私有数据**(配置项、位置信息、副本信息、Log Stream 状态、备份恢复相关信息)，这些数据不需要跨实例物理同步/物理备份。公开文档明确：meta 租户没有独立 Resource Unit，系统为它保留资源、资源从对应 user 租户中扣除、默认资源设置不可修改。

源码铁证(`deps/oblib/src/lib/ob_define.h`,v4.2.5_CE @ `e7c676806fda`):

```text
OB_SYS_TENANT_ID = 1 // sys 租户(第 893 行)
OB_SERVER_TENANT_ID = 500 // server 内部(894 行)
OB_USER_TENANT_ID = 1000 // 用户租户起点(899 行)
```

三层由 **tenant_id 奇偶性**区分(注释 1550–1554 行):`tenant_id == 1` 为 sys；奇数为 meta；偶数为 user。配对函数 `gen_user_tenant_id()`(1595 行)= meta_id + 1,`gen_meta_tenant_id()`(1608 行)= user_id − 1。即每个 user 租户(偶数)成对绑定一个 meta 租户(user−1，奇数)。这是 OB 把"用户租户内部表/元数据"下沉到 meta 租户隔离的实现根基：**事务不能跨租户**，用户租户的内部状态必须有一个与之同生命周期、又独立隔离的容器，meta 租户正是这个容器。这个设计解释了为什么 OceanBase 的"租户"会穿透 schema、事务、日志、合并、诊断与备份——它不是外部标签，而是很多内核服务初始化和调度的上下文。

### 17.3.3 多租户入口：omt 模块与租户运行时

租户在内核里由 `omt`(One Machine Tenant)模块承载(`src/observer/omt/`,v4.2.5_CE @ `e7c676806fda`)。模块 README 原文："ObMultiTenant 是入口类，同时是负责时间片计时的线程""ObTenantNodeBalancer 是负责同租户资源节点间负载均衡的后台线程"。

- `class ObMultiTenant`(`ob_multi_tenant.h:78`)：租户生命周期入口，含 `create_tenant()`、`create_tenant_without_unit(tenant_id, min_cpu, max_cpu)`、`mark_del_tenant()`、`del_tenant()`。创建走 **SLOG 两阶段**(`write_create_tenant_prepare_slog`/`commit_slog`/`abort_slog`)，状态机含 `STEP_WRITE_PREPARE_SLOG`，保证租户创建的崩溃一致性。
- `class ObTenant : public share::ObTenantBase`(`ob_tenant.h:371`)：租户运行时对象，内含资源组容器 `class ObResourceGroup`(每个 group_id 对应一个线程池 + 队列，用 `FixedHash2` 管理)。

这意味着租户在 OBServer 进程内不是一个标签，而是一组**独立的线程池、队列、内存上下文**。这种穿透性在 4.4.x 分支(@ `d4bef8d29a4c`)的 `src/observer/omt/ob_multi_tenant.cpp` 中同样可见：MTL(per-tenant module)初始化绑定了租户级 storage meta、schema service、compaction、freeze、DAG scheduler、tenant balance、tenant transfer、archive/snapshot/restore、SQL session manager、IO manager 等服务；`src/rootserver/ob_unit_manager.*`、`src/rootserver/ob_unit_placement_strategy.*` 管 Unit 资源与放置；`src/share/resource_manager/`、`src/share/resource_limit_calculator/` 对应资源管理与限制计算。这里只确认到模块/文件级，不编造具体函数调用链。

### 17.3.4 CPU 隔离：cgroup + 进程内时间片双轨

OceanBase CPU 隔离是**双轨制**:

1. **cgroup 轨**(`src/share/resource_manager/ob_cgroup_ctrl.h`):`class ObCgroupCtrl`(134 行)写 `cpu.shares`(`set_cpu_shares`,163 行)与 `cpu.cfs_quota_us`(`set_cpu_cfs_quota`,167 行)；目录布局把"选举"与"用户 SQL"分到 cgroup 根下两个目录，用户 SQL 目录再按租户、租户内 user 细分子目录。官方博客明确：cgroup 当前支持 `max` 与 `weight`,`min` 由 weight 近似保留时间片。关键：`is_valid()` 为 false(cgroup 目录无权限/OS 不支持)时**跳过 cgroup 机制**(注释 217–218 行)——cgroup 是 best-effort，失败可降级。V4.2 文档同时提醒，worker threads 与多数 background threads 可按 tenant 标识，配置 cgroup 可控制租户 CPU，但有性能代价，单租户、小规格或业务强关联租户场景不建议为隔离而强开。
2. **进程内时间片轨**(`src/observer/omt/ob_tenant.cpp`):`cpu_quota_concurrency()`(1230 行)读租户配置，默认 **4**(1233 行 fallback);worker 数 = `2 + max(1, unit_min_cpu() * cpu_quota_concurrency)`(1239 行)。即"每租户活跃工作线程数 = unit_min_cpu × cpu_quota_concurrency"，这是 cgroup 之外的软隔离主轴，即便 cgroup 不可用也生效。

### 17.3.5 内存隔离：物理切分，硬上限

内存是 OceanBase 隔离最"硬"的维度——按 meta/user 租户**物理隔离**(`ob_unit_resource.h` 注释 52–53 行："MEMORY is isolated by META and USER tenant")。常量：`META_TENANT_MIN_MEMORY = 512MB`、`USER_TENANT_MIN_MEMORY = 512MB`、`UNIT_MIN_MEMORY = 1GB`、`META_TENANT_MEMORY_PERCENTAGE = 10`(meta 租户默认占 10%)。租户内存超限会直接报"over tenant memory limit / no memory"，而非占用邻租户内存；`memory_limit_percentage`(observer 级，建议 80)、`memstore_limit_percentage`、`freeze_trigger_percentage` 进一步控制 MemStore 占比与冻结时机。这与 CPU 的"空闲可超借"形成对比：**CPU 可弹性、内存是硬墙**。

### 17.3.6 IO 隔离：租户级三参数(MIN/MAX/WEIGHT IOPS)

IO 隔离用租户级三参数 IOPS 调度，而非简单令牌桶。创建 Unit 时 CPU 与 memory 是必填核心规格，`LOG_DISK_SIZE` 可按 memory 推导，IOPS 由 `MIN_IOPS`、`MAX_IOPS`、`IOPS_WEIGHT` 控制或按 CPU 自动计算。三参数语义:

- **MIN_IOPS**(reservation)：保底，独占式保障；
- **MAX_IOPS**(limitation)：封顶；
- **IOPS_WEIGHT**(proportion)：争抢时按权重比例分配。

源码层面由 `class ObTenantIOClock`(`src/share/io/io_schedule/ob_io_mclock.h:58`,v4.2.5_CE @ `e7c676806fda`)按 `ObTenantIOConfig` 调度，以 16KB 归一化 IO 计量。三参数 reservation/limitation/proportion 体系出自 mClock 论文，官方对 mClock 的唯一表述是"These ideas are inspired by this paper ... 'mClock' ... by VMware Inc."(resource-isolation 博客原文)；讲三参数与 V4.1 增强的 io-isolation 博客通篇未出现 mClock 一词。但**源码层面调度类确实以 mClock 命名并落地三标签**:文件 `src/share/io/io_schedule/ob_io_mclock.h`(include guard `OCEANBASE_LIB_STORAGE_IO_MCLOCK`)定义 `class ObMClock`,内含三个时钟 `reservation_clock_`(= min_iops)、`limitation_clock_`(= max_iops)、`proportion_clock_`(= weight),由 `init(min_iops, max_iops, weight, proportion_ts)` 一一绑定到 MIN/MAX/WEIGHT 三参数(v4.2.5_CE @ `e7c676806fda`,4.4.x @ `d4bef8d29a4c` 同构);`ObTenantIOClock` 持有一组 `ObMClock`(`group_clocks_`/`other_group_clock_`/`unit_clock_`)按租户/消费组调度。即 OceanBase IO 调度器是 mClock 三标签算法的工程实现，并非外部令牌桶贴标签。是否为论文标准 mClock 的逐步证明级一致实现(而非工程化变体)仍不在源码可断言范围，本节保守表述为"mClock 三标签的工程实现"。OceanBase 同时强调其 IO 隔离是自研异步 IO 架构、绕过 OS cache 用 Direct IO，并把 IO 隔离与 block cache 关联，防 OLAP 扫描污染 OLTP 缓存。磁盘 IO 隔离在 V4.1 **增强**(V4.0 已支持租户间 IOPS 隔离，V4.1 在此基础上简化体验)，建立在 V4.0 的 CPU/内存隔离之上。

### 17.3.7 租户内再切分：DBMS_RESOURCE_MANAGER 资源计划

**表 17-1　OceanBase 租户内再切分(DBMS_RESOURCE_MANAGER)单侧资源隔离维度／机制／常量**

| 资源维度 | 隔离机制 | 关键常量／默认 |
|---|---|---|
| CPU(管理优先级) | consumer group 资源计划(`switch_resource_plan`) | `MGMT_P1` 默认 `mgmt_p1_ = 1`,区间 [0,100] |
| CPU(利用率上限) | consumer group 资源计划 | `UTILIZATION_LIMIT` 默认 `utilization_limit_ = 1`,区间 [0,100] |
| IO(MIN_IOPS) | 组内 IO 百分比切分 | `min_iops_ = 0`,区间 [0,100] 且 min ≤ max |
| IO(MAX_IOPS) | 组内 IO 百分比切分 | `max_iops_ = 100`,区间 [0,100] |
| IO(WEIGHT_IOPS) | 组内 IO 按权重比例 | `weight_iops_ = 0`,区间 [0,100] |

在租户级隔离之上，租户内还能按 consumer group 再切 CPU 与 IO(兼容 Oracle Resource Manager 语义)。`ObResourcePlanManager`(`src/share/resource_manager/ob_resource_plan_manager.h:31`)的 `switch_resource_plan(tenant_id, plan_name)`(46 行)切换计划；`ObResourcePlanDirective`(`ob_resource_plan_info.h`)默认值：`mgmt_p1_ = 1`、`utilization_limit_ = 1`、`min_iops_ = 0`、`max_iops_ = 100`、`weight_iops_ = 0`，校验区间 [0,100] 且 min ≤ max。即把一个租户的 CPU(MGMT_P1/UTILIZATION_LIMIT)与 IO(MIN/MAX/WEIGHT 百分比)再分给如 `tp_group`/`ap_group` 等消费组，按 username/function/列谓词映射。

### 17.3.8 日志、合并、备份的按租户隔离与生命周期

隔离不止于 CPU/内存/IO，日志、合并、备份同样按租户切分：

- **日志/日志盘**：每个 user 租户独立持有 Log Stream 及日志盘。V4.0 之后文档强调 log disk 按租户管理，避免单个租户大量写入耗尽日志盘影响其他租户。常量 `USER_TENANT_MIN_LOG_DISK_SIZE = 1.5GB`(= 3×512MB，因 user 租户至少 3 个 LS 副本、每副本 512M)、`UNIT_MIN_LOG_DISK_SIZE = 2GB`、`MEMORY_TO_LOG_DISK_FACTOR = 3`。4.4 按存储形态分叉：`UNIT_MIN_LOG_DISK_SIZE_SN = 2GB`(shared-nothing)、`UNIT_MIN_LOG_DISK_SIZE_SS = 3GB`(shared-storage 形态；社区版默认不编译该形态)。
- **合并(merge)**:major freeze 由 sys 租户的 RootService 选全局版本协调(详见第 4 章)，但合并的 IO/CPU 压力靠上述三参数 IO 调度 + cgroup 限制在租户内，并可经后台资源隔离与前台流量再切分。
- **备份恢复**：支持租户级物理备份与按时间点恢复、物理 standby 租户(V4.x);user 租户的私有恢复信息存于配对 meta 租户。RESTORE 文档要求新租户的 `pool_list` 为必填，`locality` 必须匹配 pool_list 的 zone,`primary_zone` 也必须满足 locality/zone 约束。也就是说，OceanBase 的备份恢复、扩缩容与容灾规划天然需要按 Tenant 计算资源池、Zone、Unit 与副本位置，而不只是恢复某几张表。

`ALTER TENANT` 可修改 user 租户的 locality、primary zone、resource pool list 等。从生命周期看，OceanBase 创建 Tenant 不是只登记一条元数据记录：控制面要先确认 Zone 中有足够资源承载 Unit，再把 Resource Pool 绑定到 Tenant，并按 Locality/Primary Zone 约束建立副本与服务上下文；运行期 SQL 在 user 租户上下文访问 schema、事务、Tablet 与 Log Stream/LS，租户私有的内部元数据进入对应 meta 租户；异常期若需扩缩容、restore、迁移 Unit 或调整 Locality,RootService/Unit 管理与租户级后台服务都要参与。这说明 OceanBase 的多租户复杂度不是某个单点模块造成的，而是资源、元数据、调度与存储服务共同选择了 Tenant 作为边界。

### 17.3.9 cgroup 的可选化：动态降级机制

需要厘清一个常见误传：OceanBase 数据库内核侧的 cgroup **并非到某个 4.4.x 小版本才"从必需变为可选"**。源码可确认的事实如下(4.4.x 分支 @ `d4bef8d29a4c`):

- `enable_cgroup` 是集群参数，默认 `True`、`DYNAMIC_EFFECTIVE`，文档串明确 "when set to false, cgroup will not init"。即 cgroup 自 V4.0 起即可关闭，本就是可配置项。
- `ObCgroupCtrl::check_cgroup_status()`(`ob_cgroup_ctrl.cpp:162`)做**动态**状态机，按 `enable_cgroup × is_valid()` 四象限处理，支持运行期在"用 cgroup"与"不用 cgroup(降级到进程内调度)"间切换并清理残留目录。无 cgroup 权限或被关闭时，隔离退化为 §17.3.4 进程内时间片 + §17.3.6 三参数 IO 调度，这是降级路径而非 cgroup 被废弃。
- `enable_global_background_resource_isolation` 默认 `False`，可为后台任务单独分配 vCPU，是后台合并/迁移与前台流量的全局隔离开关；该方向在 V4.3.3 文档已有"前后台全局 CPU 隔离"配置。

至于"租户资源隔离配置不再依赖 cgroup / 简化配置"的说法，其真实出处是 OceanBase Cloud(托管服务)2025 发布记录与 OCP 平台层开关(平台侧提供运行集群时按需启用/禁用 cgroup)，属 Cloud/O&M 形态的公开变化，不能泛化为 CE 某小版本所有场景的内核行为(见 §17.10)。OceanBase Database 的 CPU 隔离仍采用内核态方案，V4 默认硬隔离基于 Linux cgroup 的 CPU controller，数据库侧的 cgroup 并未被移除或变为里程碑式的"可选"。

### 17.3.10 故障隔离的能力与边界

OceanBase 更容易表达"某业务租户出了问题，不要拖垮其他业务租户"，但据现有资料推测，这并不意味着完全没有共享风险。多个 Tenant 仍共享 OBServer 进程、网络、磁盘设备、操作系统、ODP 入口与集群控制面；当物理盘、网络或 RootService/Unit balance 路径成为瓶颈时，仍需按节点和集群维度排查。差别在于，OceanBase 给了 DBA 一个更完整的租户级抓手：可以看 Unit 放置、租户资源、日志盘、合并进度与 Locality;TiDB 则更多从 Resource Group、Region 热点、Store 资源与 SQL 画像拼出隔离诊断。 〔文献[5-11]〕

## 17.4 核心差异对比

两表按维度分别陈述：表 17-2 对照两侧的隔离能力，表 17-3 对照两侧的代价与瓶颈来源。

**表 17-2 主对比：原生多租户 vs Resource Control**

| 维度 | OceanBase(原生多租户) | TiDB(Resource Control) | 影响 |
|---|---|---|---|
| 隔离单位 | Tenant(Unit/Pool/Zone/Locality) | Resource Group/用户/会话/语句 + 放置策略 | OB 是数据库实例级边界，TiDB 是共享集群内 group/会话级 |
| 隔离强度 | 偏 hard：独立内存空间/线程池/日志盘 | soft：共享进程，RU 令牌桶 + 优先级 | OB 抗噪声邻居更强，TiDB 更弹性 |
| 内存 | 物理切分，硬上限(超限报错) | 无独立内存空间，经 RU/限速间接约束 | OB 防内存挤兑，TiDB 不防 |
| CPU | cgroup + 进程内时间片(unit_min_cpu×cpu_quota_concurrency) | RU(3ms=1RU)+ TiKV 优先级 | 二者均"空闲可超借" |
| IO | 三参数 IOPS(MIN/MAX/WEIGHT，语义借鉴 mClock) | RU 折算 IO + 优先级调度 | OB 给保底带宽(reservation),TiDB 无 reservation 语义 |
| 内部元数据 | 配对 meta 租户独立隔离 | 共享系统库，无租户级元数据切分 | OB 元数据可独立备份/恢复 |
| 备份恢复 | 租户级物理备份/PITR/standby 租户(RESTORE 需 pool_list/locality/primary_zone) | 无租户级备份边界(BR 按集群/库表) | OB 租户可独立容灾，恢复前须备齐资源池与拓扑 |
| 副本放置 | 租户级 locality/primary_zone | Placement Rules 按 key range/table/database 放置，无租户级根 | OB 按租户规划，TiDB 按数据对象灵活放置 |
| 运维边界 | 租户可独立创建/删除/扩缩/升级配额 | group 是配置项，非独立运维实体 | OB 接近"实例即服务"，变更更轻者在 TiDB |
| 后台任务隔离 | per-tenant MTL(freeze/compaction/DAG/snapshot)+ enable_global_background_resource_isolation(4.4) | BACKGROUND TASK_TYPES + UTILIZATION_LIMIT(v8.4) | 两侧都把后台与前台切开 |
| Cloud 多租户 | OceanBase Cloud/OCP 公开大量租户资源管理能力，仍需分形态 | Starter/Essential 内部隔离未公开，不可实现层推断 | 两侧都需避免把产品文案写成内核实现 |

**表 17-3 性能瓶颈/代价来源对比**

| 来源 | OceanBase | TiDB |
|---|---|---|
| 复杂度集中点 | 租户成为内核第一复杂度(schema/事务/日志/合并/备份/诊断全按租户切) | RU 标定与 group 规划成为运维心智负担 |
| 噪声邻居残留 | 内存硬隔离；CPU/IO 空闲可超借时仍可能瞬时抖动 | 软隔离上限：无独立内存，极端写放大可影响共享 RocksDB/节点 |
| 抖动来源 | major freeze/merge 的 IO/CPU 抖动(经三参数 IO/后台隔离缓解) | Runaway 仅事后纠偏；RU 估算偏差致限流过紧/过松 |
| 元数据/资源底噪 | 每 user 租户配对 meta 租户、最小内存/日志盘，小租户多时碎片与底噪不可忽视 | 共享元数据，group 近乎零额外底噪 |
| 隔离失效路径 | cgroup 不可用→降级进程内调度(4.4 分支动态状态机) | RU 关闭/未绑定→落 default 组(UNLIMITED) |

两表合看，分歧的根源是一处取舍：OceanBase 把复杂度沉进内核换硬隔离边界，TiDB 把复杂度留在运维调参换弹性与简单，两侧没有排名，只有适配场景的不同。

## 17.5 正常路径图

图 17-2 把 OceanBase 用户租户的正常请求路径(资源在多层被约束)与 TiDB Resource Group 的正常路径并置对比。

![[f17_2.svg]]

**图 17-2　多租户与资源隔离正常读写/调度路径**

**说明**:TiDB 路径先把请求归入 Resource Group，再经 RU、优先级与下游请求标签影响执行，数据放置是另一个可组合机制；TiDB Server 与 TiKV 节点是共享的，隔离只发生在令牌桶与调度器这两个"逻辑闸门"。OceanBase 路径先进入某个 user 租户，上下文再穿透 SQL、事务、Tablet/LS、Unit 资源与 meta 租户；每一跳(omt→tenant→cgroup/mem/io→LS)都是租户私有对象，内存为硬墙。前者适合在共享集群里快速给 workload 做限流与优先级，后者适合把"数据库实例"作为资源与运维边界。

## 17.6 故障/异常路径图

图 17-3 把四类典型异常路径统一为故障流：噪声邻居、cgroup 失效降级、Runaway 触发、合并抖动。

![[f17_3.svg]]

**图 17-3　多租户与资源隔离故障/异常路径**

异常路径里，TiDB 的风险主要是软隔离的边界：RU 可以限制进入系统的速率与优先级，但同一 TiKV 节点上的 compaction、热点 Region、磁盘队列或某些全局后台任务仍可能让多个组同时受影响；Runaway 能切断超预期 SQL，但它不是 Tenant 级故障域。OceanBase 的风险主要是内核多租户复杂度：Unit、Log Stream/LS、Tablet、merge、backup、schema、diagnostics 都按 Tenant 牵动，隔离更深，恢复与变更路径也更长。

## 17.7 性能、可靠性、运维影响

将上述机制落到工程影响上，可从延迟/吞吐、可靠性、扩展性、运维复杂度四个维度归纳。

- **延迟/吞吐**:OceanBase 内存硬隔离 + IO reservation(MIN_IOPS)给前台租户更确定的延迟下界，代价是 CPU/IO 在争抢时仍可能抖动，且合并期需保底带宽；Unit 规格决定租户可用资源，扩容通常涉及 Unit config、Resource Pool、Zone 放置。TiDB 软隔离吞吐弹性更好(空闲资源可被 BURSTABLE 组吸收，RU 让吞吐可按组预算)，但缺内存/IO reservation，极端负载下隔离上限低，整体上限仍由集群容量决定。高优先级组可牺牲低优先级组延迟。
- **可靠性**：可靠性要分"隔离"与"恢复"两层看。OceanBase 租户级备份/PITR/standby 租户提供租户粒度容灾，且 RESTORE 明确需要 `pool_list`/`locality`/`primary_zone`，恢复时系统要为新租户准备资源与副本拓扑；cgroup 失效会**降级而非中断**(4.4 动态状态机)，可靠性路径更韧。TiDB Resource Control 可在资源紧张时限制某组 SQL，但若某业务误删表、需 PITR，恢复边界仍按 TiDB 支持的库表/集群能力设计，而非按 Resource Group 自动恢复。
- **扩展性**:OceanBase 多租户在"一套集群承载大量小实例"上密度高(不靠 VM/容器)，但每 user 租户配对 meta 租户、独立线程池/最小 1GB 内存/最小 2GB 日志盘，小租户极多时元数据与资源底噪、Unit 碎片不可忽视。TiDB 共享底座，group 近乎零额外资源底噪，但隔离强度天花板低。
- **运维复杂度**:OceanBase 把 Unit/Pool/Zone/Locality/资源计划做成必须理解的运维面，复杂但边界清晰，创建、扩缩容、迁移、restore 都要理解 Unit、Resource Pool、Locality、meta 租户(详见第 13 章控制面)。TiDB 运维简单、低侵入，已有集群中新增一个 Resource Group 比新增一个完整租户资源池更轻；若业务只是希望"报表 SQL 不要拖垮在线交易",Resource Control/Runaway 可能已足够，代价是更依赖管理员持续校准 RU、优先级、SQL 画像与热点分布，当热点是底层 Region 或磁盘队列时，单纯增减某组 RU 未必能解决。

## 17.8 反例与代价

两种模型各有用错的方式：把硬隔离套到不需要它的极小租户上，或把软隔离当成强租户来依赖。下面分别列出。

- **OceanBase 不合适的场景**：海量"超小租户"(每租户 KB~MB 级数据)。每 user 租户最小 1GB 内存 + 1.5GB 日志盘 + 配对 meta 租户 + 独立线程池，加上 sys/meta/user 三层、Unit、Locality、ODP 路由、合并、备份、监控的管理成本，底噪被放大，Unit balance、合并窗口与诊断噪声会变成新问题，密度反不如共享底座。此时 TiDB Resource Group 或 Cloud Starter 的共享模型更经济。公开 cgroup 文档也提醒，资源隔离会带来性能损失，单租户、小规格或强业务关联场景不建议为隔离而强开 cgroup。
- **TiDB 不合适的场景**：把 Resource Group 当作"强租户"。它能限制 RU/s、priority 与 Runaway，但不天然拥有独立 schema catalog、独立日志空间、独立备份恢复对象或独立副本 Locality。需要**强内存/IO 保底**的多业务混部(如金融多租户严格 SLA)，软隔离无内存硬墙、无 IO reservation，单个失控写入仍可能波及共享节点，Runaway 是事后纠偏、已消耗资源不可回收；若业务要求强审计边界、客户级恢复、客户级密钥与完全隔离的资源账本，只用 Resource Group 会把很多责任推回应用、运维与 Cloud 产品层。
- **共同代价**:OceanBase 的代价是复杂度——schema/事务/日志/合并/备份/诊断全部按租户切分，任何一处都要处理"按租户隔离",Unit 太小会致租户内抖动、太大又降低池化效率。TiDB 的代价是 RU 抽象与现实资源的偏差——RU 是 CPU/IO 的折算，标定不准则限流与真实瓶颈错配。
- **资源隔离 ≠ 安全隔离**:Resource Group 或 Tenant 能降低噪声邻居与资源争抢，但不能自动替代账号权限、网络边界、审计、加密、SQL 注入防护与客户数据建模。TiDB 中同一集群的多业务若共享数据库账号，Resource Group 不会替应用补上行级租户过滤；OceanBase 中多业务即使拆成不同 Tenant，也仍要分别配置账号、权限、连接入口、备份权限与审计策略。多租户内核边界解决的是"资源与运维域怎么分"，安全章节(第 21 章)仍要回答"谁能访问什么数据"。
- **取舍**：这不是"谁更先进"，而是把复杂度放在哪里——OceanBase 放在内核与运维面换硬隔离，TiDB 放在运维调参换弹性与简单。

## 17.9 测试开发视角的验证点

**可测功能场景**:

- OceanBase:`create resource unit ... max_cpu/min_cpu/max_iops/min_iops/iops_weight` → `create resource pool` → `create tenant`，验证 `unit_config`、`locality`、`primary_zone` 生效与 `ALTER TENANT` 调整后的资源变化；`DBMS_RESOURCE_MANAGER` 建 consumer group 验证组内 CPU/IO 切分；构造多个 Tenant 的随机读、顺序写、日志密集写、merge 期间查询，验证 CPU/memory/IOPS/log disk 与合并任务的隔离效果。
- TiDB:`CREATE RESOURCE GROUP rg RU_PER_SEC=... [BURSTABLE] [PRIORITY=...]`、`CREATE USER ... RESOURCE_GROUP rg`、`QUERY_LIMIT(EXEC_ELAPSED/RU/PROCESSED_KEYS, ACTION)`、`CALIBRATE RESOURCE`；把 OLTP、小报表、批导入分别放进不同 Resource Group，观察热点 Region、TiKV IO、TiDB CPU 与 SQL 延迟是否仍出现跨组影响。

**可注入失效模式**:

- 噪声邻居：邻租户/邻 group 跑全表扫描、错误笛卡尔 join 或高频写，另一组跑固定 QPS 小事务，观测本租户/组 P95/P99 与 RU 消耗趋势。
- cgroup 失效：容器内移除/无权限 cgroup，验证 OceanBase 4.4 降级到进程内调度仍可服务；Cloud 形态不应假设能直接操作 OS cgroup。
- 内存挤兑：OceanBase 单租户写满 MemStore 验证只本租户报错；TiDB 验证软隔离上限。
- 热点租户：OceanBase 单 Tenant 写满日志盘预算、触发大合并、缩小/迁移 Unit、锁定 Tenant 或调整 Locality，观察其他 Tenant 是否保持稳定。
- Runaway：构造超时/超 RU/超 PROCESSED_KEYS 查询，验证 DRYRUN/COOLDOWN/KILL/SWITCH_GROUP 与 WATCH 命中。

**压测矩阵(三条轴)**：第一条轴是租户大小(TiDB 用不同 RU/s 与 priority,OceanBase 用不同 Unit config、`unit_num` 与 Locality 表示大中小)；第二条轴是 workload 类型(小事务、范围扫描、批量导入、DDL/backfill、备份恢复、merge/compaction 期间查询分开测，因为争抢资源不同)；第三条轴是故障时序(先让基准租户稳定运行，再注入噪声邻居、热点写入、长查询、Unit 迁移或 cgroup 配置变化)。三轴组合才能看出"平均吞吐正常但尾延迟被邻居拖垮"的问题。验收应设负向断言：对 TiDB，要证明 Runaway 被 KILL/SWITCH_GROUP 后原组短事务延迟恢复，而非仅看到问题 SQL 消失；对 OceanBase，要证明某 Tenant 的 IOPS/log disk 压力不会让其他 Tenant 的小事务出现同等幅度抖动。观测失败时不应立即得出产品缺陷结论，而要继续排查物理节点共用、热点 leader、后台任务、cgroup 开关、Unit 放置与测试流量是否真正按预期绑定。

**关键观测/诊断(经查证，不编造)**:

- OceanBase 内部视图：`DBA_OB_UNIT_CONFIGS`、`GV$OB_UNITS`(各 OBServer 上 Unit 信息)、`GV$OB_TENANTS`、`GV$OB_SERVER_SCHEMA_INFO` 等；Tenant 维度 CPU/memory/IOPS/log disk/session/major-minor merge 进度、Unit 分布、LS leader 分布、restore/backup 任务状态；obdiag 可拉取租户信息。资源/租户维度的内部视图名可在源码 `src/share/inner_table/ob_inner_table_schema_constants.h`(v4.2.5_CE @ `e7c676806fda`)直接核对，已确认存在的有 `DBA_OB_UNIT_CONFIGS`、`GV$OB_UNITS`、`DBA_OB_UNITS`、`DBA_OB_RESOURCE_POOLS`、`DBA_OB_TENANTS`、`GV$OB_SERVER_SCHEMA_INFO`、`GV$OB_TENANT_MEMORY`、`GV$OB_MEMORY`、`GV$OB_TENANT_RESOURCE_LIMIT` 及 `__all_virtual_tenant_resource_limit(_detail)`、`__all_virtual_io_quota`、`__all_virtual_unit` 等(注:租户清单视图标准名为 `DBA_OB_TENANTS`,无同名 `GV$OB_TENANTS`)。其余视图名仍按目标版本核对。
- TiDB:`information_schema.RESOURCE_GROUPS`、`mysql.request_unit_by_group`(RU 历史)、`information_schema.RUNAWAY_WATCHES`、TiDB Dashboard Resource Manager 页；每组 RU 消耗、读/写 RU、SQL latency、Runaway 记录数量、TiKV/TiFlash CPU/IO 与热点 Region 分布。
- TiDB 侧 RU 相关 Prometheus 指标可在 PD 源码 `pkg/mcs/resourcemanager/server/metrics.go`(release-8.5 @ `6dce4a68e3e9`)直接核对：namespace `resource_manager`、subsystem `resource_unit`/`resource`/`server`,如 `resource_manager_resource_unit_read_request_unit_sum`、`..._write_request_unit_sum`、`..._read/write_request_unit_max_per_sec`、`resource_manager_resource_kv_cpu_time_ms_sum`、`..._sql_cpu_time_ms_sum`、`..._read/write_byte_sum`、`resource_manager_server_group_config`,TiDB 侧另有 `tidb_session_resource_group_query_total`(`pkg/metrics/session.go`)。OceanBase 内核不内置等价 Prometheus exporter,其 Grafana 面板取自 OBServer 内部视图/OCP/obagent,具体面板指标名仍以官方 Grafana/OCP 模板为准（待核实），不在此编造。

## 17.10 容易误解点

1. **"TiDB Resource Control = OceanBase 多租户"——错。** Resource Control 是共享底座上的软隔离(RU 令牌桶 + 优先级 + Runaway)，没有独立内存空间、独立线程池、独立日志盘、租户级元数据/备份/副本放置；OceanBase 原生多租户在这五个维度都有独立对象，更接近 hard isolation。可以把两者都纳入"多租户方案"，但能力不等价。

2. **"sys/meta/user 是权限层级"——错。** 它们是**资源与元数据隔离层**，由 tenant_id 奇偶编码区分(sys=1、奇数=meta、偶数=user),meta 租户与 user 租户成对绑定(user−1)、同生命周期且资源从 user 租户扣除，用来隔离用户租户的私有内部数据，而非访问控制层级。把它简化成"数据库名隔离"会漏掉 Unit、Resource Pool、Locality、日志盘、IOPS、合并、备份恢复与诊断这些主要复杂度。

3. **"cgroup 是 OceanBase 隔离的唯一支柱"——错。** cgroup 是 best-effort 一轨；进程内时间片(unit_min_cpu×cpu_quota_concurrency)与三参数 IO 调度是另一轨。cgroup 自 V4.0 起即可关闭，4.4 分支提供动态降级状态机，隔离不中断。把 OceanBase CPU 隔离简化为"cgroup"会误判其降级行为。

4. **"Cloud 文案等于内核实现"——需分形态。** 一类是 OceanBase 的 cgroup:V4.2 Database 文档仍描述可配置 cgroup 控制租户 CPU，而 OceanBase Cloud 2025 release notes 说"tenant resource isolation configuration no longer relies on cgroup"，后者只能限定为 Cloud/O&M 形态的公开变化，不可写成"CE 某 4.4.x 版本不再需要 cgroup"(数据库内核侧 cgroup 自 V4.0 起即可关闭，不存在"从必需变可选"的里程碑，见 §17.3.9)。另一类是 TiDB Cloud Starter：公开说明共享资源，但不公开内部调度实现，不可反推 Keyspace、PD、TiKV 池或调度细节。可靠写法是把 Cloud/CE/企业版、Starter/Essential/Dedicated 分开，把已公开能力与推测能力分开。

5. **"RU 是精确的 CPU/IO 计量"——需谨慎。** RU 是 CPU(3ms=1RU)与 IO 的折算抽象，标定来自基准与成本系数(系数在 PD 侧)。RU 与真实瓶颈存在偏差，CALIBRATE 仅估算，不应当作物理配额理解。

## 17.11 本章结论

1. OceanBase 是**原生多租户**:Tenant→Resource Pool→Unit→Zone→Locality 自底向上拼装，每 user 租户拥有独立内存空间(硬上限)、独立线程池、独立 Log Stream/日志盘，并成对绑定 meta 租户隔离内部元数据(tenant_id 奇偶编码：sys=1、奇=meta、偶=user，资源从 user 租户扣除);Tenant 是贯穿 SQL、schema、事务、存储、日志、合并与恢复的内核边界。

2. OceanBase 资源隔离是**多机制叠加**:CPU 双轨(cgroup `cpu.shares`/`cfs_quota` + 进程内 `unit_min_cpu×cpu_quota_concurrency`)、内存物理硬隔离、IO 三参数(MIN/MAX/WEIGHT IOPS，语义借鉴 mClock 论文、非源码级移植)、租户内 DBMS_RESOURCE_MANAGER 再切分；4.4 分支 cgroup 可动态降级、`enable_global_background_resource_isolation` 切后台 vCPU。cgroup 自 V4.0 起即可关闭，Cloud 形态的"不依赖 cgroup"不可泛化到 CE 所有场景(见 §17.3.9、§17.10)。

3. TiDB Resource Control 是**软隔离**:RU(CPU/IO 统一抽象，3ms=1RU)令牌桶(RU_PER_SEC/BURSTABLE)+ TiKV 优先级 + Runaway 规则，辅以 Placement Rules/TiDB Node Group 拓扑能力，实现于共享 TiDB/TiKV 之上，无独立内存/线程/日志盘/租户级元数据，适合共享集群内 workload QoS,v7.1 GA。

4. 二者**不等价、无优劣**:OceanBase 把复杂度放进内核与运维面换 hard isolation 与租户级备份/副本放置/运维边界；TiDB 把复杂度放进 RU 调参换弹性与运维简单。综合判断，选型主要取决于是否需要强内存/IO 保底与租户级容灾，这一归纳属工程推测。

5. 隔离失效路径不同：OceanBase 内存硬墙使邻租户 OOM 自限、cgroup 失效降级不中断；TiDB 无独立内存，极端负载下软隔离有天花板，而 Runaway 仅属事后纠偏——后一判断为本章工程推测。

6. TiDB Cloud Starter/Essential 内部隔离机制未公开，只能写"多租户共享资源"与"Dedicated 提供隔离基础设施/资源"的用户可见边界，其内部实现尚不确定，本章**不做实现层断言**。测试多租户不要只测平均吞吐，应重点测噪声邻居、热点租户、后台 merge/backup/restore、日志盘压力与资源配置变更下的 P95/P99 抖动。

## 17.12 参考文献

[1] Use Resource Control to Achieve Resource Group Limitation and Flow Control. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tidb-resource-control-ru-groups/.
 （支撑:§17.2.1 RU 定义/消耗系数、RU_PER_SEC/PRIORITY/BURSTABLE、TiDB 流控 + TiKV 优先级两层与版本时间线；§17.2.3 Placement/后台任务的资源组边界。）
[2] Manage Runaway Queries / Use Resource Control to Manage Background Tasks. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tidb-resource-control-runaway-queries/.
 （支撑:§17.2.2 Runaway 三阈值与 DRYRUN/COOLDOWN/KILL/SWITCH_GROUP、WATCH 匹配；§17.2.3 UTILIZATION_LIMIT/TASK_TYPES 与版本归属。）
[3] What is TiDB Cloud / TiDB Cloud Starter FAQs. 官方文档[EB/OL]. https://docs.pingcap.com/tidbcloud/tidb-cloud-intro/.
 （支撑:§17.2.6、§17.10 中 Starter 多租户共享资源、Dedicated 才提供 isolated infrastructure/resources 的用户可见边界。）
[4] What is TiDB Resource Control?. 官方博客[EB/OL]. https://www.pingcap.com/blog/tidb-resource-control-workload-consolidation-transactional-apps/.
 （支撑:逐字支撑 §17.2.1/§17.2.5 的 workload consolidation 与 v7.1 GA;"soft isolation/非完整多租户"为本章归纳定位，博客未逐字出现。）
[5] Resource isolation overview / Why is resource isolation important for HTAP?. 官方博客[EB/OL]. https://oceanbase.github.io/docs/blogs/feat/resource-isolation.
 （支撑:§17.3.4 cgroup CPU 目录布局、§17.3.6 mClock "inspired by" 归属、§17.3.7 DBMS_RESOURCE_MANAGER 消费组与前后台隔离。）
[6] OceanBase I/O isolation experiences. 官方博客[EB/OL]. https://oceanbase.github.io/docs/blogs/feat/io-isolation.
 （支撑:§17.3.6 MIN_IOPS/MAX_IOPS/IOPS_WEIGHT 三参数(reservation/limitation/proportion)与 V4.1 IO 隔离增强；本页不含 mClock 一词，故归属下）
[7] Create a tenant / Tenant and resource management / Configure cgroups. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001103995.
 （支撑:§17.3.1 sys tenant 创建流程、§17.3.2 meta 租户资源扣除、§17.3.6 log disk/IOPS 隔离与 Unit 参数、§17.3.4 V4.2 形态下 cgroup 控制租户 CPU）
[8] Release notes for 2025 | OceanBase Cloud. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-cloud-10000000001879252.
 （支撑:§17.3.9、§17.10、§17.13 中 OceanBase Cloud 2025 "tenant resource isolation configuration no longer relies on cgro）
[9] OceanBase: A 707 Million tpmC Distributed Relational Database System. 论文，VLDB 2022[EB/OL]. https://vldb.org/pvldb/vol15/p3385-xu.pdf.
 （支撑:§17.3 "不依赖 Docker/VM、数据库内轻量资源隔离、Resource Unit 含 CPU/内存/IOPS/磁盘"的设计层论断，逐字出自 §2.5.3 Resource Isolation(printed）
[10] oceanbase/oceanbase 仓库(v4.2.5_CE @ e7c676806fda)— deps/oblib/src/lib/ob_define.h、src/observer/omt/、src/share/unit/、src/share/resource_manager/、src/share/io/io_schedule/. 源码[EB/OL]. https://github.com/oceanbase/oceanbase.
 （支撑:§17.3.1 ObResourcePool 映射、§17.3.2 tenant_id 奇偶编码常量、§17.3.3 omt 入口类、§17.3.4 ObCgroupCtrl、§17.3.5 内存常量、§17.3.6 O）
[11] oceanbase/oceanbase 仓库(4.4.x @ d4bef8d29a4c)— src/share/resource_manager/ob_cgroup_ctrl.cpp、src/observer/omt/ob_multi_tenant.cpp、src/rootserver/ob_unit_manager.*、src/share/parameter/ob_parameter_seed.ipp、src/share/unit/ob_unit_resource.h. 源码[EB/OL]. https://github.com/oceanbase/oceanbase.
 （支撑:§17.3.3 per-tenant MTL 服务、§17.3.8 Log Disk 按 SN/SS 分叉、§17.3.9 enable_cgroup 默认 True/DYNAMIC、check_cgroup_statu）
[12] pingcap/tidb 仓库(release-8.5 @ 67b4876bd57b)— pkg/meta/model/resource_group.go、pkg/ddl/resourcegroup/group.go、pkg/resourcegroup/runaway/checker.go、pkg/domain/ru_stats.go、pkg/sessionctx/variable/tidb_vars.go、pkg/metrics/session.go. 源码[EB/OL]. https://github.com/pingcap/tidb.
 （支撑:§17.2.1 ResourceGroupSettings 字段与 RU→令牌桶映射、§17.2.2 Runaway 三阈值、§17.2.3/§17.2.4 RUStatsWriter 落表 mysql.request_）
[13] tikv/pd 仓库(release-8.5 @ 6dce4a68e3e9)— pkg/mcs/resourcemanager/server/config.go、pkg/mcs/resourcemanager/server/metrics.go. 源码[EB/OL]. https://github.com/tikv/pd.
 （支撑:§17.2.1 RU 读写/CPU 标定常量(defaultReadBaseCost = 1/8、defaultReadPerBatchBaseCost = 1/2、defaultWriteBaseCost =）
[14] Secrets Behind TiDB Serverless Architecture. 第三方解读，Medium[EB/OL]. https://dataturbo.medium.com/secrets-behind-tidb-serverless-architecture-8b277f00cb7c.
 （支撑:§17.2.6 Cloud Starter 数据/元数据物理隔离(per-tenant LSM-Tree SST)、disaggregated PD、TiProxy 路由、空闲实例回收复用；第三方非官方，作为【工程推测/）
## 17.13 信息可信度自评

- **官方文档/博客明确**:TiDB RU 模型/消耗系数、RU_PER_SEC/PRIORITY/BURSTABLE、Runaway 三阈值与动作、UTILIZATION_LIMIT/TASK_TYPES 版本归属、Placement Rules;OceanBase Tenant/Resource Pool/Unit/meta 租户/log disk/IOPS、IO 三参数(MIN/MAX/WEIGHT)、cgroup CPU 目录布局、V4.0/V4.1 隔离演进、RESTORE 所需租户参数——均有 docs.pingcap.com / oceanbase 官方文档/博客支撑。mClock 仅为"受启发(inspired by)"归属，非官方"源码实现"表述。
- **源码级信息**:OceanBase tenant_id 奇偶编码常量、omt 入口类与 per-tenant MTL 服务、ObCgroupCtrl、内存/日志盘下限常量、ObTenantIOClock 三参数、`class ObMClock` 三时钟(reservation/limitation/proportion = MIN/MAX/WEIGHT IOPS)、ObResourcePlanManager 默认值、资源/租户内部视图名;TiDB ResourceGroupSettings 字段、Runaway checker 阈值、RUStatsWriter 落表、`tidb_session_resource_group_query_total`;PD(release-8.5)RU 标定常量(`defaultReadBaseCost = 1/8`…`defaultCPUMsCost = 1/3`)与 resource_manager Prometheus 指标名——commit-pin 到统一基线，确认到文件/符号级，未编造具体函数调用链。
- **论文级**:OceanBase 不依赖 Docker/VM 的轻量资源隔离与 Resource Unit 维度，逐字出自 VLDB 2022 论文 §2.5.3 Resource Isolation(printed page 3388);707M tpmC 仅作系统背景，不外推为多租户性能。
- **工程推测**:TiDB soft vs OceanBase hard 的总体定位(soft isolation 为本章归纳措辞，非官方逐字术语)、二者代价取舍、OceanBase 故障隔离能力边界、TiDB Cloud Starter 内部机制(基于第三方 Medium 解读)。
- **经源码核验确认**:① OceanBase IO 调度类已源码确认为 mClock 三标签工程实现(`ObMClock` 三时钟 reservation/limitation/proportion),官方文案仍仅"inspired by",故定位为"mClock 三标签的工程实现"而非"论文标准 mClock 的逐步证明级一致实现"(见 §17.3.6);③ TiDB RU 读写/CPU 标定系数已在 PD `pkg/mcs/resourcemanager/server/config.go` 源码确认(非 TiDB 仓库,见 §17.2.1);④ TiDB/PD 侧 RU Prometheus 指标名已源码确认(见 §17.9),仅 OceanBase 侧 Grafana 面板指标名仍以官方/OCP 模板为准；⑥ VLDB 论文资源隔离小节定位为 §2.5.3 p.3388(已核 PDF)。
- **仍不确定/需进一步查证**:② "tenant resource isolation no longer relies on cgroup" 仅限 OceanBase Cloud/OCP 形态，不可泛化到 CE 内核所有场景(见 §17.3.9、§17.10);⑤ TiDB Cloud Starter/Essential 内部隔离为闭源，未做实现层断言；以及 OceanBase 侧 Grafana 面板指标名、目标版本完整内部视图清单仍需按版本核对。

---


# 第 18 章 容灾、高可用与多地多活

## 18.1 本章核心问题

容灾（Disaster Recovery）与高可用（High Availability）是分布式数据库与单机数据库最本质的分水岭。单机数据库的高可用本质是「换一台机器」，而分布式数据库把高可用内建于数据复制协议本身：数据天然存在多副本，故障时不是「恢复一台机器」，而是「在剩余副本上选出新的服务者」。本章讨论的不是一张「备份功能清单」，而是分布式数据库在节点、AZ、城市乃至地域故障下如何继续给出一致的数据状态，以及这种能力要付出多少写延迟、带宽、运维与演练成本。

承接第 1 章的 Region/Tablet/Log Stream-LS 边界、第 3 章的 Raft/PALF 共识边界、第 13 章的 PD/RootService 控制面依赖、第 17 章的租户边界，本章把容灾拆成三层：在线副本高可用、异步或半同步灾备链路、备份归档与 PITR。本章要回答的核心问题是：

1. **TiDB 与 OceanBase 各自如何把共识协议（Multi-Raft / Multi-Paxos over PALF）转化为可用的容灾能力**，以及副本放置、跨地域部署、备份恢复如何与共识层叠加。两者都把在线 HA 建在多数派复制之上，但抽象不同：TiDB 的 OLTP 数据以 Region / Raft Group 为复制单元，OceanBase 4.x 以 Log Stream（LS）承载多个 Tablet 的日志复制和恢复。
2. **副本角色到底有几种**：Voter / Learner / Witness（TiKV）与 Normal Replica / Arbitration Replica（OceanBase）在底层是不同机制还是同一概念的不同叫法？这是本章必须查证、不可类比的关键点——二者在「是否存日志」「是否每次写都投票」两个维度上存在实质差异，且 TiKV Witness 在锁定版本下并非 GA 特性。
3. **RPO/RTO 究竟如何理解**：RPO=0 是协议自洽的强约束，RTO<8s 是厂商自述/设计目标（随部署模式而异），二者的可信度层级不同，且都是条件化指标。
4. **跨地域写延迟的物理下限**：为什么多数派提交（majority commit）在跨地域部署下无法逃脱光速约束。只要单个对象仍要求强一致多数派提交，远距离 RTT 就会进入写延迟下限——若两地相距 1000km，光在光纤中单程约 5ms、往返约 10ms，数据库优化只能减少排队、批处理与路由开销，不能消除物理传播时间。
5. **金融级容灾为什么复杂**：它不只是一个技术指标，而是合规、监管、演练、回切、数据校验的系统工程。

本章不重复第 3 章（共识协议本身）、第 13 章（PD/RootService 控制面）、第 1 章（分片即共识单元）的深入内容，而是聚焦「共识之上」的容灾工程层。共识协议保证的是单个 Raft Group / Log Stream 的副本一致性；容灾工程要解决的是**把成千上万个共识组协调成一个能抵御机房级、城市级故障的整体**，并在故障发生时给出可预期、可演练、可审计的恢复行为。

边界说明：TiDB 以 8.5.x LTS 为基准（对照 7.5.x 仅作版本说明），TiKV/PD 取 release-8.5；OceanBase 以 v4.2.5_CE（TP）文档与源码为主，并对 4.3.5（AP）/4.4.x（融合）的差异作版本注记。

---

## 18.2 TiDB 的实现

TiDB 的容灾体系建立在 **Multi-Raft** 之上（详见第 3 章）。每个 Region 是一个独立的 Raft Group，由多个 Peer 组成，默认 3 副本：Leader 接收写入，日志复制到多数 Voter（2/3）确认后才提交；PD 通过 Store label、Placement Rules、调度器与复制模式约束副本放置。普通三 AZ 同城部署下，默认三副本可在少数派 AZ/节点故障后选主继续服务；多数派丢失时，一致性系统不会自动「猜测」缺失日志，通常需要 unsafe recovery 或专业工具介入，否则 RPO=0 与可用性不能同时保证。

在此基础上，TiDB 把高可用拆为三层能力：**副本放置（Placement Rules）→ 双中心强同步（DR Auto-Sync）→ 集群外复制与备份（TiCDC / BR / PITR）**。

### 18.2.1 副本角色与 Placement Rules

PD 的 Placement 模块定义了底层副本角色。源码（pingcap/pd · release-8.5 · commit 6dce4a68e3e9 · `pkg/schedule/placement/rule.go`）中，`PeerRoleType` 只有四个常量：`Voter = "voter"`、`Leader = "leader"`、`Follower = "follower"`、`Learner = "learner"`，且 `validateRole` 只接受这四种。关键在于 `MetaPeerRole()` 的映射：`Learner` 映射到 `metapb.PeerRole_Learner`，其余全部映射到 `metapb.PeerRole_Voter`——即**在底层 Raft 成员层面，只存在 Voter 与 Learner 两类角色**，Leader/Follower 只是 Voter 在运行时的状态约束。官方 Placement Rules 文档同样说明，Placement Rules 控制副本数量、位置、是否参与 Raft election 以及是否成为 Leader。

**Witness 不是第三类成员角色，而是 Rule 上的一个正交布尔标志。** 同一文件中 `Rule struct` 含字段 `IsWitness bool`，注释为 "when it is true, it means the role is also a witness"。在 TiKV 侧（tikv/tikv · release-8.5 · `components/raftstore/src/store/peer.rs`），`is_witness()` 返回 `self.peer.is_witness`，witness peer 在选举/调度中 `priority` 被设为 `-1`，且有专门的管理命令 `AdminCmdType::BatchSwitchWitness`，快照路径用专门标志区分；TiKV 的 local read 路径遇到 `peer.is_witness` 会返回 `Error::IsWitness`，因此 witness 不能当作保存完整数据并提供读服务的副本。

因此三类机制（以源码字段为准）的区别是：

- **Voter**：参与投票、存数据、可成 Leader。
- **Learner**：**不投票**、存数据（异步追日志）、不计入多数派——TiFlash 列存副本即 Raft Learner（详见第 16 章），也常用于扩容或灾备预热。
- **Witness**（`IsWitness` 标志）：设计意图为「轻量投票者」——`priority -1`、独立 snapshot 标志、专用 BatchSwitchWitness 命令。按 tikv/tikv #11498 的设计描述，Witness 是「只保留 Raft log、不 apply 到状态机」的 **log-only 副本**（即仍持久化 Raft 日志，只是不存全量数据、不提供读、不能当 leader），正常运行时参与写多数派投票。**注意它并非「零存储」：它存 Raft 日志。**

> 反查证与成熟度说明：经独立核查，TiKV Witness 在锁定版本 TiDB 8.5.x / 7.5.x 中**并非 GA 的生产推荐特性**。官方 TiKV Overview 文档与 Placement Rules in SQL 对外只暴露 Leader/Follower/Learner/Voter，不暴露 witness 角色；Witness 在 TiKV 主要以 feature-request / 跟踪 issue 形式存在（tikv/tikv #11498、#11499，tikv/raft-rs #145），其设计被描述为实验性、可能变更或移除；TiDB v8.0.0 release notes 还记录「witness 相关、未 GA 却默认开启的 scheduler 被移除」。因此本章对 Witness 的源码字段陈述按代码事实书写，但**不将其当作 8.5.x 已 GA 的生产容灾形态**，具体生产语义与稳定性**需进一步查证**。此外需注意，网络上常见的「witness=不服务读、参与每次写投票、不存数据、不能当 leader」四点定义实为 Google Spanner / CockroachDB 的 witness replica 规范，不应直接当作 TiKV 的官方实现照搬。

由此 Witness 与 Learner 是底层**不同机制**——Witness 偏向「投票、存日志不存全量数据」，Learner 是「存数据不投票」，二者正交，不可互相类比。Witness 与 OceanBase 的 Arbitration Replica 也**不是同一种副本**，二者在「是否存日志」「是否每次写都投票」上存在实质差异（见 §18.3.1 与 §18.10）。

### 18.2.2 DR Auto-Sync：双中心强同步状态机

DR Auto-Sync（Data Replication Auto-Synchronous）是 TiDB 同城双中心（两 AZ）的强一致容灾方案。它要解决一个矛盾：跨两个 AZ 各放部分副本，既要在正常时保证两中心强同步（RPO=0），又要在一个中心整体失联时自动降级为可用（牺牲跨中心同步换可用性）。

源码（pingcap/pd · release-8.5 · `pkg/replication/replication_mode.go`）给出了状态机的内部实现：复制模式常量 `modeMajority = "majority"` 与 `modeDRAutoSync = "dr-auto-sync"`；DR 内部状态常量为小写下划线形式 `drStateSync = "sync"`、`drStateAsyncWait = "async_wait"`、`drStateAsync = "async"`、`drStateSyncRecover = "sync_recover"`。

> 状态数辨析：**官方用户文档对外只正式枚举三态：`sync` / `async` / `sync-recover`**；`async_wait` 是 SYNC→ASYNC 之间的内部过渡态（文档仅在 `pause-region-split` 说明里顺带提及），并非公开状态机的第四个正式成员。因此严谨表述应为「对外三态 + 一个内部过渡态（async_wait）」，而非「官方四态集合」。本章下文沿用源码常量名（小写下划线）以贴合实现。

判定逻辑：

- `canSync := primaryHasVoter && drHasVoter`——每个 Region 必须在两个 DC 各有至少 1 个 voter 副本，才能进入 sync。
- `hasMajority`（此处为源码中的通用谓词，语义为 `totalUpVoter*2 > totalVoter`，即存活 voter 是否过半；非对外稳定 API 标识符）。
- 关键注释（源码原文为英文）：大意为「若不再拥有多数派，集群将始终不可用，切到 async 也无济于事」——即**多数派真正丢失时，降级到 async 无法恢复集群可用性**，这澄清了「降级是万能的」这一常见误解。
- `ACIDConsistent` 标志在状态为 `sync_recover` 时被置为 false——SYNC_RECOVER 恢复期不保证 ACID 一致。

配置项（`server/config/config.go`）：`ReplicationMode` 默认 `"majority"`，非法值强制回落 majority；DR Auto-Sync 配置含主/灾备标签与副本数，以及 `WaitStoreTimeout`、`PauseRegionSplit`、`WaitRecoverTimeout` 等；`WaitStoreTimeout` 默认常量 `defaultDRWaitStoreTimeout = time.Minute`（即 60 秒），`wait-recover-timeout` 默认 `0s`。官方文档给出的典型部署是 6 副本：主中心 3 Voter，灾备中心 2 Voter + 1 Learner，两 AZ 间距 <50km、网络延迟 <1.5ms、带宽 >10Gbps；网络故障超时后切 async。版本核对显示 v6.5、v7.5、v8.5/stable 文档均可见该能力，更早版本的具体名称**需进一步查证**。

**注意：DR Auto-Sync 没有 `arbitration_timeout` 参数——该参数属于 OceanBase 仲裁服务，与 TiDB 无关（详见 §18.3.1）。** DR Auto-Sync 的超时参数是 `wait-store-timeout`（默认 60s）与 `wait-recover-timeout`（默认 0s）。

正常路径（sync）：写请求经主中心 Leader 复制到本中心 follower 与灾备中心 voter，需两中心都有副本确认才提交。故障路径（灾备中心失联超 60s）：状态机 sync → async_wait → async，Raft 回退到经典 majority（只在主中心内凑多数派），灾备中心追不上的副本被降级。网络恢复后 async → sync_recover → sync 逐步追平。这里有一个关键边界：主 AZ 故障而灾备侧具备完整同步数据时，DR Auto-Sync 文档仍提示需要专业工具或支持介入恢复——它不是一个任意时刻自动跨城主备切换的魔法开关。

### 18.2.3 三地两中心（5 副本）与跨地域延迟

![[f18_1.svg]]

**图 18-1　TiDB 三地两中心（5 副本）多地多中心地理部署拓扑**

TiDB 原生支持「三地两中心」（Three AZ in Two Regions）：5 副本布局为 AZ1 两副本、AZ2 两副本（同城 Seattle）、AZ3 一副本（异地 San Francisco）。官方文档给出确切延迟：同城 AZ1↔AZ2 <3ms，异地 AZ3↔AZ1/AZ2 约 20ms（专线）。**Region Leader 被限制只在同城两 AZ**，以避免 Leader 落到异地导致每次写都要付 20ms 跨地域 RTT。其前提是多数派可在同城两 AZ（2+2=4 副本）内达成，异地那 1 副本只作为城市级容灾的「保底投票者」，不进多数派关键路径。

跨地域的另一个成本是 TSO：TiDB 的全局时间戳由 PD 统一分配（详见第 13 章），异地 TiDB Server 取 TSO 需向主城 PD 发一次 RTT。PingCAP 官方博客记录，在 100ms 模拟跨地域延迟下，通过把 TSO 与 read index 合并、Follower Read 优化，读延迟从 200ms 降到 100ms（两 RTT 降为一 RTT）。

### 18.2.4 集群外复制与备份：TiCDC / BR / PITR

在集群内多副本之外，TiDB 还提供面向跨集群复制与备份恢复的三组组件，它们以非零 RPO 换取距离、成本与恢复粒度：

- **TiCDC**：跨集群异步增量复制，用于「主-从」双集群容灾。它能把 TiDB 增量变更同步到另一个 TiDB 或 MySQL 兼容数据库；其灾难场景下的 eventually consistent replication 自 v6.1.1 GA，文档给出的条件化指标是：上游崩溃前复制正常且延迟小，可在 5 分钟内恢复下游，RTO ≤ 5min、P95 RPO ≤ 10s。这个路径牺牲 RPO=0，换来跨地域容灾和读写扩展空间。注意：TiCDC 源码在独立仓库 pingcap/tiflow，不在 tidb/tikv/pd 主仓库内，本章未做 commit-pinned 源码定位，具体实现细节**待核实**。
- **BR（Backup & Restore）**：代码内嵌于 tidb 主仓库（pingcap/tidb · release-8.5 · commit 67b4876bd57b · `br/pkg/`），含 `backup/`、`restore/`、`stream/`（日志备份）、`checkpoint/`；PITR 任务入口在 `br/pkg/task/stream.go` 与 `restore.go`。全量备份给出基线，log backup 持续写外部存储，`checkpoint[global]` 表示可恢复到的进度；恢复时只能恢复到 checkpoint 前的时间点。TiKV 源码（`components/backup-stream/src/metrics.rs`）中定义了一组 log backup 指标，如 `tikv_log_backup_store_checkpoint_ts`、`tikv_log_backup_task_status`、`tikv_log_backup_flush_duration_sec`、`tikv_log_backup_initial_scan_duration_sec`、`tikv_log_backup_incremental_scan_bytes`、`tikv_log_backup_errors`、`tikv_log_backup_fatal_errors` 等，可作为观测点。
- **PITR（Point-in-Time Recovery）**：全量快照备份 + 连续日志备份，可恢复到故障前某一秒。官方标 BR 方案 RPO < 5min、RTO 小时级；自 v8.5.5 起支持日志备份任务运行中并行执行 PITR。

故障路径上，TiDB 的关键分支是「少数派故障」和「多数派故障」。少数派 TiKV/Region peer 故障时，PD 会补副本或调度 Leader，客户端 RegionCache 遇到 not leader / epoch 变化后刷新路由。PD 自身是控制面关键依赖，但不能简单写成单点：PD 多副本保证元数据与 TSO 的高可用，同时在调度、复制模式状态推进和恢复流程上是逻辑中心化依赖（关于控制面单点的多维度讨论见 §18.3.2 对 RootService 的对照分析）。

--- 〔文献[1-5,9,11]〕

## 18.3 OceanBase 的实现

与 TiDB 相对，OceanBase 的容灾建立在 **Multi-Paxos over PALF**（PALF = Paxos-backed Append-only Log File system，详见第 3 章）之上。复制单位是 **Log Stream（LS）**：官方 Log streams 文档说明，Log Stream 记录数据库所有变更，副本、主备库和备份数据都依赖日志中的变更信息；一个 tenant 内 Log Stream 可以有多个副本通过 Paxos 同步，少数副本故障时可实现 RPO=0、RTO 小于 8s。这里的 RPO/RTO 是条件化指标，前提是少数派故障且部署满足文档约束，不能扩大成任意城市级灾难都 8 秒恢复（边界详见 §18.7）。

副本分布由 **Locality** 描述、主副本优先级由 **Primary Zone** 表达。相比 TiDB，OceanBase 在容灾层多出两个一等公民：**Arbitration Replica（仲裁副本）** 与 **物理 standby（物理备库）**。

### 18.3.1 副本类型：Normal Replica 与 Arbitration Replica

**表 18-1　Witness / Arbitration / Learner 正交对照矩阵（仅按验证维度对照，Witness 与仲裁成员不可简单类比）**

| 对照维度 | TiKV Witness | OceanBase 仲裁(Arbitration) | Learner |
|---|---|---|---|
| 是否存用户数据 | 否（不存全量数据） | 否（不存用户数据，无 MemTable / SSTable） | 是（存数据） |
| 是否存/同步日志 | 存 Raft 日志（log-only，不 apply 到状态机） | 不逐条同步日志，只存日志流元数据 | 存数据，异步追日志 |
| 是否每次写都投票 | 是（正常运行时参与每次写多数派投票） | 否（正常不投日志，仅半数全功能副本故障时做降级仲裁） | 否（不投票，不计入多数派） |
| 能否当主 / 提供读 | 不能当 Leader；local read 拒读，不提供读 | 不能被选为主提供服务 | 不进多数派；可作异步只读（TiFlash 列存即 Learner） |
| 底层机制 / 成熟度 | Rule 上正交布尔标志 is_witness（priority -1）；8.5.x / 7.5.x 非 GA | PALF 枚举 ARBITRATION_REPLICA + 独立仲裁服务；v4.1 起 GA，仅 2F+1A / 4F+1A | PeerRole 枚举 Learner（底层 Raft 成员） |

PALF 层的日志副本类型枚举（oceanbase/oceanbase · v4.2.5_CE · commit e7c676806fda · `src/logservice/palf/log_define.h`）完整可读、未脱敏：

```cpp
enum LogReplicaType {
 INVALID_REPLICA = 0,
 NORMAL_REPLICA, // full replica
 ARBITRATION_REPLICA, // arbitration replica
};
```

`log_replica_type_to_string` 输出 `"NORMAL_REPLICA"` / `"ARBITRATION_REPLICA"`。同一枚举在 4.4.x（另据 commit d4bef8d29a4c 亦确认）的同名文件中一致存在——**仲裁副本能力跨 4.2.5 与 4.4.x 持续存在**（跨版本源码对照）。

**Arbitration Replica / 仲裁成员的语义（经源码 + 多个独立来源交叉确认）**：它是 PALF 层的一个日志副本类型枚举，对应的部署形态是独立的轻量**仲裁服务/仲裁成员**（官方称谓为「仲裁服务」与「仲裁成员」，2F1A = 2 全功能副本 + 1 仲裁成员）。OCP 文档进一步说明：在两 IDC 场景中，传统 2F+1L 通过两个 full-featured replica 加一个 log replica 保持 Paxos HA，而 arbitration service 的目标是用 2F+1A 替代 2F+1L，降低日志副本所在 OBServer 的存储与 CPU 成本。该文档同时给出边界：集群版本须晚于 V4.1.0.0；arbitration replica 版本与 full-featured replica 相同或稍新；仅支持 2F+1A 和 4F+1A。其语义为**只存储日志流的元数据、不存储用户数据、无 MemTable / SSTable**，资源开销极小，**不能被选为主提供服务**。

> 关键机制辨析（与 TiKV Witness 的实质差异）：仲裁成员采用**降级（degradation）模型**，而非「对每次写提交都投票」的 Spanner 式 witness。正常运行时，由 2 个全功能副本（2F）自身构成 Paxos 多数派完成日志同步，**仲裁成员不逐条参与日志多数派投票（Paxos Accept）**，仅参与选举、Paxos Prepare 及成员组变更投票；只有当半数全功能副本故障时，仲裁服务才介入，把故障成员移出成员列表（触发「日志流降级」）以低多数派恢复服务并保 RPO=0，故障恢复后再「升级」加回。对照之下，TiKV Witness 按其设计是「存 Raft 日志、正常运行时参与每次写多数派投票」的 log-only 副本。因此**两者不是同一种「投票不存数据」副本**：OceanBase 仲裁成员连日志都不逐条同步、平时不投日志、仅在故障时仲裁降级；TiKV Witness 存日志且每次写都投票。这一差异详见 §18.10。

这意味着在 2F1A（2 Full + 1 Arbitration）部署中，若两个 full 副本同时故障，仲裁成员**无法独自恢复服务**——它不是数据源，只是仲裁者。

仲裁服务有显式状态机：`ObArbitrationServiceStatus`（`src/share/ob_arbitration_service_status.h`）枚举 `{ INVALID, ENABLING, ENABLED, DISABLING, DISABLED }`，启停各有中间态。仲裁服务/仲裁成员状态有配套的内部虚表与外部视图提供可观测性：源码（`src/share/inner_table/ob_inner_table_schema_constants.h`）定义内部虚表 `__all_virtual_arbitration_member_info`、`__all_virtual_arbitration_service_status`，以及对外视图 `DBA_OB_ARBITRATION_SERVICE`、`GV$OB_ARBITRATION_MEMBER_INFO` / `V$OB_ARBITRATION_MEMBER_INFO`、`GV$OB_ARBITRATION_SERVICE_STATUS` / `V$OB_ARBITRATION_SERVICE_STATUS`（4.2.5 与 4.4.x 一致）。源码侧，PALF 的 `ObLogHandler` 在 `OB_BUILD_ARBITRATION` 条件下暴露 `add_arbitration_member`、`remove_arbitration_member` 等接口，因此正文只确认能力与接口存在，**不宣称所有 CE 构建默认启用**（社区版默认不编译该能力）。

**降级逻辑**：仲裁服务跑在 LS Leader 上，当 full 副本在仲裁超时参数（OceanBase 租户级参数 `arbitration_timeout`）内未回日志确认，触发进一步检查。关键规则：**仅当故障副本数（含已降级）等于 full 副本总数的一半时才执行 LS 降级**。例：4F1A 时坏 1 个不降级（3F 仍多数派），坏 2 个才降级；2F1A 时坏 1 个 full 副本即触发降级，降级后剩 1F+1A 仍可服务（RPO=0）。PALF 的日志配置管理模块实现了仲裁成员加入与成员降级到 learner 的机制（因 OceanBase CE 源码部分标识符被脱敏，**仅确认到模块级**，具体符号名/文件名不照搬）。`arbitration_timeout` 在源码（`src/share/parameter/ob_parameter_seed.ipp`）中定义为租户级 `DEF_TIME` 参数，**默认 `5s`、取值范围 `[3s,+∞]`、动态生效**，注释为「The timeout before automatically degrading when arbitration member exists」；4.2.5 / 4.3.5 / 4.4.x 三分支取值一致，官方文档亦载默认 5s（如需禁用降级可设足够大的值如 30d）。

> 关键纠偏：OceanBase 的 Arbitration Replica（仲裁成员）与 TiKV 的 Witness **不是同一种副本，不能等价**。粗看二者都「偏轻量、不存全量用户数据」，但实质机制不同：（1）**是否存日志**——TiKV Witness 是 log-only 副本（存 Raft 日志，只是不 apply），OceanBase 仲裁成员连日志都不逐条同步、只存日志流元数据；（2）**是否每次写都投票**——TiKV Witness 正常运行时参与每次写多数派投票，OceanBase 仲裁成员正常时不投日志、仅在半数全功能副本故障时做降级仲裁；（3）**成熟度/官方实现层级不对称**——OceanBase 仲裁服务自 v4.1.0 起 GA（覆盖 4.2.5/4.3.5/4.4.x），而 TiKV Witness 在锁定的 8.5.x/7.5.x 中并非 GA。本章按查证后的真实差异书写，不直接类比。

### 18.3.2 Primary Zone、Locality 与部署形态

OceanBase 的副本与放置语义比 TiDB 更贴近租户与 Locality:

- **Primary Zone**（`src/share/ob_primary_zone_util.*`）：主副本所在 zone 的优先级表达，影响 leader 倾向与强一致读路由，自 v1.4 起可含多 zone，形如 `"zone1,zone2;zone3"`（`;` 分隔优先级、`,` 同级）。
- **Locality**（`src/rootserver/ob_locality_util.*`、`src/storage/ob_locality_manager.*`）：副本分布策略，描述每个 zone 放几个什么类型的副本。

通过调整租户 Locality，OceanBase 支持多种部署：**同城三中心（3F）**、**两地三中心**、**三地五中心（2+2+1，5 副本）**，都通过 Zone/Region、Locality、Primary Zone 与资源池/Unit 约束落地。三地五中心要求每副本部署到不同机房，事务需同步到至少两个城市，实现城市级无损容灾。RootService 侧有专门的灾备调度模块 `ob_disaster_recovery_worker/service/task`（4.4.x · `src/rootserver/`）处理副本补齐、迁移、替换租户。其好处是容灾拓扑成为租户级可审计配置；代价是租户扩缩容、Unit 数、Zone 资源、合并任务与日志成员变更会互相牵连。

这里需要厘清 RootService 的定位：它在调度、负载均衡、副本恢复与租户角色推进上是**逻辑中心化**依赖，但不宜简单地贴上「单点故障」标签。按五个维度分别看——（1）**逻辑中心化**：元数据与调度入口收敛在 RootService;（2）**性能瓶颈**：大规模集群下副本补齐、合并任务编排压力集中，但不在每次用户事务的关键路径上；（3）**高可用单点**：RootService 自身由所在租户的 Log Stream 多副本承载，故障后可重新选主，不是无副本的孤点；（4）**元数据依赖**：路由、Locality、备份归档状态机都依赖其元数据视图；（5）**故障恢复依赖**：副本恢复、降级/升级仲裁等流程由 RootService 推进。把它定论为「SPOF / Achilles heel」会掩盖其多副本高可用与「不在用户事务热路径」的事实，这与 TiDB 把 PD 视为逻辑中心化控制面依赖、而非裸单点的判断是一致的（见 §18.2.4）。

### 18.3.3 物理 standby：基于 PALF 日志镜像

OceanBase 物理备库（Physical Standby）的核心是 `ObLogRestoreHandler`（`src/logservice/restoreservice/ob_log_restore_handler.h`）。类注释明确："The interface to submit log for physical restore and physical standby"——**同一处理器同时服务物理恢复与物理 standby**。注释区分两种 standby 来源：`for standby based on archive`（基于日志归档）与 `for standby based on net service`（基于网络服务/直连主库拉日志）。注释还描述了切主前的完整性检查："Before the standby tenant switchover to primary, check if all primary logs are restored in the standby"。配套的 `ObStandbyService` 包含 tenant role transition、recover tenant、检查 `log_restore_source` 等接口。

**实现机制**：物理 standby 通过 PALF 的 raw-write 把主库日志（来自归档或网络服务）写入备库的 PALF。两种模式有明确的延迟差异——archive-based 模式下 standby tenant 只从主或备租户的归档读取 redo log，通常分钟级延迟；network-based 模式下 standby tenant 通过网络直接连接主或另一个 standby tenant，从在线日志或归档日志读取，可到秒级同步。这印证了「物理 standby 基于 PALF 日志镜像」——PALF 论文（VLDB 2024）亦表述为「restorers 从 archivers 接收日志并镜像到 PALF group 的 leader」。日志归档（`src/logservice/archiveservice/`）把 PALF 日志推到外部存储，日志恢复（`restoreservice/`）从归档/远端拉回 raw-write 回放，两者共同支撑备份恢复 / PITR / 物理 standby。需说明：公开证据支撑「基于 Log Stream 日志源抓取和回放，底层与 PALF log handler 相关」，但「PALF group mirror」是否为官方公开实现术语**需进一步查证**，本章不照搬该名称。

主备保护模式有三档：最大性能、最大保护、最大可用，可切换。

> 版本架构纠偏（重要）：**自 OceanBase v4.1.0 起，物理备库的产品形态由「集群级主备（主集群 + 备集群）」变更为「租户级主备（主租户 + 备租户）」——集群不再有主备角色概念，只是承载租户的容器（container），主备角色属于租户。** 本章锁定版本 4.2.5 / 4.3.5 / 4.4.x 全部在 4.1.0 之后，因此「一个主集群带 N 个备集群」这一旧形态说法不适用于锁定版本；同步单元是「备租户」而非「备集群」（OCP 界面虽仍保留「备集群」封装，底层为租户级）。官方对租户级仅描述「一主多备」架构（经 Log Restore Source 同步），**并未公布固定上限**；OceanBase 4.x 的各项 max value 限制文档把「同一主租户支持的最大备租户数」列为「无限制」。此前坊间流传的「一个主集群最多带 31 个备集群」是 OceanBase **v3.x 集群级主备**架构下的数字，套到锁定的 4.x 版本属版本错配 + 概念失效，**该数字不适用于本章锁定版本，已删除**。

### 18.3.4 备份恢复与 PITR

OceanBase 物理备份以 Tenant 为单位，由 data backup 和 log archiving 组合；文档要求先启用 log archiving 再做 data backup。Tenant restore 可全量恢复或快速恢复，不完整恢复可指定 SCN 或 timestamp。日志归档由 OBServer 自动写到指定备份路径，目录中按 Log Stream 组织 archive files、checkpoint metadata 与 piece 信息。

4.2.5 物理备份/恢复增强：引入 SET/PIECE 级物理恢复（`ADD RESTORE SOURCE` 从新路径加载数据备份集 SET 或日志归档 PIECE，按需恢复到指定时间点）；存储介质扩展到 NFS/OSS/COS + AWS S3 / 华为 OBS / GCP GCS；支持 I/O 负载自适应仲裁。这一层对抗的是存储介质损坏、误操作、跨集群重建，不是在线多数派故障的实时切主。

--- 〔文献[6-8,10,14]〕

## 18.4 核心差异对比

第一张表对照副本与复制层的核心差异：

**表 18-2　容灾、高可用与多地多活:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB | OceanBase | 影响 |
|---|---|---|---|
| 共识基础/复制单元 | Multi-Raft（Region = Raft Group） | Multi-Paxos over PALF（LS = Log Stream，承载多个 Tablet） | TiDB 故障与调度更细粒度、副本数随 Region 数膨胀；OceanBase 收敛到少量 LS，减少复制组数量但让 LS 状态更关键（详见第 1/3 章） |
| 放置控制 | PD label + Placement Rules + replication mode | Tenant/Locality/Primary Zone/Unit/Resource Pool | TiDB 偏 KV range 调度；OceanBase 偏租户与资源编排，拓扑写入租户级配置 |
| 轻量/不存全量数据副本 | **Witness**（Rule 标志位 `is_witness`，priority -1；log-only、每次写都投票；8.5.x 非 GA） | **Arbitration 仲裁成员**（PALF 枚举 + 独立仲裁服务；不存用户数据/不逐条同步日志、正常不投日志、仅故障时降级仲裁；v4.1 起 GA、仅 2F+1A/4F+1A） | **不是同一种副本**：存日志/投票语义/成熟度均不同，**不可直接类比**（见 §18.10） |
| 不投票存数据副本 | **Learner**（TiFlash 列存即 Learner） | Read-only / 列存副本（详见第 16 章） | 都用于异步只读扩展，不进多数派 |
| 双中心强同步 | DR Auto-Sync（对外三态 sync/async/sync-recover + 内部过渡态 async_wait） | Paxos 多副本 + Locality/Primary Zone（最大保护/可用模式；两地三中心/三地五中心） | 都能 RPO=0；前者更明确暴露复制模式切换，后者把拓扑写入租户级配置 |
| 跨地域异步复制 | TiCDC（集群外异步，独立仓库 tiflow）+ BR/PITR | 物理 standby（PALF 日志镜像，内核内一体）+ log archiving | TiDB 组件化、OceanBase 一体化；RPO/RTO 从 0 变为条件化秒/分钟级 |
| RPO（集群内多数派） | 0 | 0 | 二者强一致语义等价 |
| RTO（自述/设计目标） | 分钟级（多副本方案）/ ≤5min（主从） | <8s（随 v4.0 架构重构落地，由更早版本的 <30s 优化而来；**仅适用于多副本 Paxos 部署**，shared-storage 单副本为分钟级）（证据有限） | OceanBase 自述更激进；数字均需注明部署前提，不可当绝对排名 |
| 备份/PITR | BR（内嵌 tidb 仓库 `br/pkg/`）+ 日志备份 | 物理备份 + SET/PIECE 归档恢复 | 都支持 PITR；存储后端都覆盖主流对象存储 |

第二张表专看故障半径与典型行为：

**表 18-3　容灾、高可用与多地多活:TiDB 与 OceanBase 核心差异对比**

| 故障类型 | TiDB 典型行为 | OceanBase 典型行为 | 主要风险/备注 |
|---|---|---|---|
| 单节点/单 store 故障 | Region 自动选新 Leader（秒级），PD 补副本 | LS 自动选主（多副本部署下自述 RTO<8s，含检测+恢复+重连）（证据有限），RootService 推进副本恢复 | 抖动期路由缓存失效、尾延迟升高；均为共识层自愈 |
| 单 AZ 少数派故障 | 多数派仍在则 RPO=0，服务可继续 | 少数副本故障文档给出 RPO=0 / RTO<8s 条件化目标 | 调度风暴、重建流量、热点 Leader 重分布 |
| 灾备中心整体失联 | sync→async_wait→async（`wait-store-timeout` 默认 60s 超时后降 majority），主 AZ 多数 Voter 丢失时恢复需工具/人工 | 仲裁服务在仲裁超时后降级故障副本（超时参数 `arbitration_timeout`，源码确认默认 5s、范围 [3s,+∞]） | 降级阈值参数与默认值不同；`arbitration_timeout` 仅属 OceanBase，TiDB 侧无此参数 |
| 多数派丢失（过半 voter 挂） | **降级无用**（源码注释：不再拥有多数派则集群始终不可用，切 async 无济于事）；unsafe recovery 有数据风险 | 2F1A 两 full 都挂则 LS 无主、仲裁成员无法独自恢复服务；不自动突破 Paxos 多数派安全性 | 二者都受多数派物理约束，可用性与 RPO=0 冲突，需明确业务选择 |
| 网络恢复后追平 | async→sync_recover→sync（恢复期 `ACIDConsistent=false`） | 降级成员重新加回、日志追平 | TiKV issue #15366:sync_recover→sync 切换曾致 QPS 抖动 |
| 误删/逻辑污染 | BR/PITR 恢复到 checkpoint 前 | 物理备份 + log archiving 按 SCN/timestamp 恢复 | 备份滞后、日志断链、恢复演练不足 |

---

## 18.5 正常路径图

图 18-2 展示 TiDB DR Auto-Sync 同城双中心的**正常写路径（SYNC 状态）**，并对照 OceanBase LS 的正常复制路径：

![[f18_2.svg]]

**图 18-2　容灾、高可用与多地多活正常读写/调度路径**

关键点：SYNC 状态要求 `canSync = primaryHasVoter && drHasVoter`，即每个 Region 在两中心各有 voter，提交必须等到**跨中心副本确认**——这是 RPO=0 的来源，代价是每次写要付一次 <1.5ms 的跨 AZ RTT。PD 持续监控，只要两中心都健康就维持 SYNC。

OceanBase 2F1A 正常路径类似：LS Leader 把 PALF 日志同步到本地 full 副本与异地 full 副本（由 2 个全功能副本构成 Paxos 多数派），仲裁成员不参与日志同步、正常时不投日志，只在选举/成员变更/故障降级时介入；主副本由 Primary Zone 优先级决定。其异步灾备分支（日志归档 → 物理 standby tenant）独立于在线多数派路径之外：

![[f18_3.svg]]

**图 18-3　容灾、高可用与多地多活正常读写/调度路径**

---

## 18.6 故障/异常路径图

图 18-4 展示 TiDB DR Auto-Sync 在**灾备中心失联 → 降级 → 恢复**的故障路径，并对照 OceanBase 2F1A 仲裁降级：

![[f18_4.svg]]

**图 18-4　容灾、高可用与多地多活故障/异常路径**

图 18-5 把故障决策抽象为通用判定树，适用于两系统：多数派是否仍可达，决定了能否自动恢复，还是必须人工/工具介入或回退到备份/PITR:

![[f18_5.svg]]

**图 18-5　容灾、高可用与多地多活故障/异常路径**

两条故障路径的共同物理约束：**无论降级机制多精巧，一旦真正丢失多数派（TiDB 侧存活 voter 不再过半、PD 源码注释即「切 async 也无济于事」；OceanBase 两 full 副本皆失），都无法在不牺牲一致性的前提下恢复服务**。这是多数派共识的硬下限，不是工程缺陷。

---

## 18.7 性能、可靠性、运维影响

**延迟**：跨地域部署的写延迟受光速物理下限约束。光在光纤中约 200,000km/s（约真空 2/3），折算约 5μs/km。多数派提交的延迟由「仲裁内最近的那个副本」决定——三副本跨三地时，必须等至少两地确认，延迟由最远的那段链路主导。CockroachLabs 工程博客给出量级：北弗吉尼亚↔俄勒冈约 3700km，理论单向 ≥18.5ms、往返 ≥37ms，真实跨大陆 60–80ms，跨洲（纽约↔伦敦）>80ms，纯物理决定。TiDB 三地两中心文档实测同城 <3ms、异地约 20ms，正是这一规律的体现；DR Auto-Sync 文档限定两 AZ <50km、延迟 <1.5ms，不是随意参数，而是对强一致写物理约束的承认。PingCAP DR 博客也把 2-2-1 强一致方案的主要代价写成跨地域网络延迟导致的响应时间下降。结论：**跨城强同步写，延迟下限就是跨城 RTT，任何软件优化都无法突破。**

**吞吐与扩展性**：容灾副本越多、跨地域越远，提交关键路径越长，单事务吞吐越低，写放大、同步带宽、调度复杂度与恢复流量也越大。TiDB 用 Follower Read / Stale Read（详见第 11 章）把读下放到就近副本规避跨地域读延迟；OceanBase 用弱一致读 / 只读副本同理。可靠性收益也不是单调增加：副本越多可承受的节点故障越多，但成本与复杂度随之上升。OceanBase arbitration service 文档直说 2F+1L 虽能保持 HA，但 log replica 所在 OBServer 的日志存储与 CPU 开销会影响成本竞争力，所以引入 2F+1A（仲裁成员与 Witness 的语义差异见 §18.10）。

**可用性与故障恢复**：OceanBase 自述集群内 RPO=0、RTO<8s。其中 RPO=0 由 Paxos 同步把 redo 日志复制到多数派（2/3 或 3/5）后才确认提交，属协议自洽强约束。关于 RTO:**8s 这一里程碑随 OceanBase v4.0 的核心选举/共识协议重构落地（消息驱动的相对时间选举、缩短选举租约等），由更早版本的 RTO<30s 优化而来——是 4.0 的特性，不是 4.1。** 其 8s 拆解（厂商口径）为：故障检测（1–2s，lease ≤4s、探活去 NTP 依赖）+ 数据库恢复（3–6s，leader-driven 重选 + 路由消息刷新）+ 应用重连（7–8s，OBProxy 探活摘黑名单）（证据有限）。

> 两点限定：（1）<8s 是厂商自述/设计目标（"aims to provide RTO<8s without extensive tuning"），公开资料缺少完全独立的生产环境故障切换实测，本章按弱化指标处理；（2）**该数值仅适用于多副本 Paxos 的集群内部署**；官方亦自述 shared-storage 单副本部署的 RTO 为分钟级。因此「集群内 RTO<8s」必须带「多副本部署」前提，不可作无条件结论。

TiDB 多副本方案 RTO 为分钟级，但 Region 级 Leader 切换通常秒级——RTO 的「分钟级」主要来自控制面/客户端层面而非单 Region 选主。

**运维复杂度**：DR Auto-Sync 与 2F1A 都引入了「降级/恢复」这一额外状态维度，运维必须理解状态机转换（否则会被 SYNC_RECOVER 期 QPS 抖动、ASYNC 期 RPO 不再为 0 等行为困扰）。复杂度集中在三类环节：第一是**故障切换**——要区分自动 Leader 切换、租户角色 switchover/failover、主备集群应用入口切换；第二是**回切**——灾后不是把流量切回来就结束，还要做日志追平、数据校验、拓扑复原、旧主处理；第三是**演练**——金融级容灾还要满足合规、监管报备、RPO/RTO 证明、演练记录、审计留痕与回滚预案。数据库内核能保证日志与复制协议的局部正确性，不能替业务完成所有切换编排。TiKV issue #15366 即真实案例：v6.5.4 在 sync_recovery→sync 切换约 15 分钟后 QPS 跌 >30%，影响 6.5.x/7.1.x LTS。

--- 〔文献[12-13]〕

## 18.8 反例与代价

1. **跨地域强同步不适合低延迟 OLTP**：若业务对写延迟敏感（单位毫秒），强行三地强同步会把每个写事务的延迟抬到跨城 RTT 量级（数十毫秒），这是物理代价，非调优可解。若业务需要单行或单账户在多个远距地域同时写、还要求每次提交后全球立即可见，多数派协议会把远距 RTT 写进每次提交；改用异步复制就必须接受非零 RPO、冲突检测或业务补偿。此时应考虑单城多中心 + 异地异步备库（牺牲异地 RPO）。

2. **多地多活与强一致的根本冲突**：真正的「多地多活」（每地都能本地强一致写同一份数据）与「全局强一致」在物理上不可兼得。要本地低延迟写，就得放松跨地一致（走 CRDT / 最终一致 / 冲突合并，如 Redis Active-Active 的 strong eventual consistency）；要全局强一致，跨地写就得付多数派跨城 RTT。TiDB 与 OceanBase 都选择**强一致优先**，因此它们的「多地」本质是「多地容灾 + 单写入点」而非「多地各自强一致写」。所谓「多活」在金融语境下通常指**单元化多活**（不同业务单元的主副本分散在不同地，各自就近，而非同一数据多地可写）。

3. **DR Auto-Sync 的适用边界**：它适合低延迟同城双 AZ，不适合把相距很远的城市硬塞进同步复制模式。若主 AZ 多数 Voter 丢失，即使 DR AZ 有完整数据，文档仍提示需要专业工具或支持介入，所以不能承诺全自动无感切主。如 §18.2.2 所述，源码注释明确「切到 async 也无济于事」；OceanBase 2F1A 两 full 皆挂则无主。二者都说明：容灾设计能抵御「少数派故障」，但**多数派故障意味着数据潜在丢失或不可用**，任何标榜「永不停机」的说法都需对照这一物理事实。

4. **仲裁副本/Witness 降低了成本但也降低了冗余度**：2F1A 比 3F 省一个 full 副本的存储，但若一个 full 副本永久损坏，系统进入「1F+1A」的脆弱态——此时再坏一个即无主。仲裁副本降低存储成本，但它不是数据副本；当业务需要本地读或本地恢复数据块时，仍要依赖 full-featured / log / backup 路径。这是成本与可靠性之间的取舍，而非免费收益。

5. **TiCDC 主从的 RPO 非零**：跨集群异步复制本质是异步，亚秒级 RPO 也意味着故障瞬间可能丢失最后若干毫秒的已提交数据；它也不能覆盖上游崩溃前已经复制滞后的窗口。把 TiCDC 当「零丢失容灾」是误用。

6. **OceanBase 复杂拓扑的管理面代价**：Locality、Primary Zone、Unit、Tenant、LS、备份归档与 standby role transition 组合后表达能力很强，但任何资源池缩容、Zone 调整、arbitration service 版本不匹配、日志归档落后，都可能阻塞切换或恢复。表达力的另一面是运维面的牵连风险。

---

## 18.9 测试开发视角的验证点

**可测试的功能场景**：
- DR Auto-Sync 状态转换：注入灾备中心网络隔离，验证 sync→async_wait→async（超 `wait-store-timeout` 60s）→ 恢复后 sync_recover→sync，并验证 `ACIDConsistent` 在 sync_recover 期为 false（其中 async_wait 为内部过渡态）。同时验证 Placement Rules 是否让 Voter/Learner/Witness 落在预期 label 上。
- OceanBase 2F1A 降级：停一个 full 副本，验证仲裁超时（`arbitration_timeout`，默认值以官方文档为准）后仲裁服务降级、剩 1F+1A 仍可写且 RPO=0；再停另一 full，验证 LS 变为无主、仲裁成员不接管。并验证 Locality/Primary Zone 配置是否落地。
- 主备 switchover/failover：验证物理 standby 切主前的「所有主库日志已恢复」完整性检查；覆盖 archive-based 与 network-based 两种日志源。
- 跨集群异步：TiCDC 下游在上游故障时是否能恢复到最近一致点；BR/PITR 是否能恢复到 `checkpoint[global]` 之前的指定时间点，并校验数据一致性。

**可注入的失效模式**：杀 TiKV/OBServer 进程、隔离 Leader 所在 AZ、城市级专线中断、延迟或丢包跨城链路、单 store/单 LS 进程崩溃、暂停对象存储写入、磁盘 I/O 卡顿（触发 OceanBase IO 负载自适应仲裁）、网络分区（脑裂场景验证多数派侧可用、少数派侧拒写）、模拟 PD/RootService 短时不可用、让日志归档慢于生成速度、时钟漂移、在恢复前后做校验表或 checksum。

**关键压测指标**：跨地域写 P95/P99 延迟（对照跨城 RTT 物理下限）、降级/恢复期间的 QPS 抖动（参考 TiKV #15366）、Leader 切换窗口（对照 RTO）、RPO 估算、日志复制 lag、恢复吞吐、回切耗时、业务错误率、PITR 恢复速度。

**关键观测指标/视图/命令（经查证）**：
- TiDB:PD 的复制模式状态（`majority`/`dr-auto-sync`）可经文档中的 replication mode status API 观察；DR 内部状态（对外 sync/async/sync-recover，内部含 async_wait）与 `ACIDConsistent` 标志。BR log status 输出 `checkpoint[global]`;log backup 指标在 TiKV 源码（`components/backup-stream/src/metrics.rs`）中定义，除 `tikv_log_backup_store_checkpoint_ts`、`tikv_log_backup_task_status` 外，还包含 `tikv_log_backup_flush_duration_sec`、`tikv_log_backup_initial_scan_duration_sec`、`tikv_log_backup_incremental_scan_bytes`、`tikv_log_backup_errors`、`tikv_log_backup_fatal_errors`、`tikv_log_backup_enabled` 等一组指标，可作观测点。
- OceanBase：仲裁服务相关可观测对象在源码（`src/share/inner_table/ob_inner_table_schema_constants.h`）中确认：内部虚表 `__all_virtual_arbitration_member_info` / `__all_virtual_arbitration_service_status`，对外视图 `DBA_OB_ARBITRATION_SERVICE`、`GV$OB_ARBITRATION_MEMBER_INFO` / `V$OB_ARBITRATION_MEMBER_INFO`、`GV$OB_ARBITRATION_SERVICE_STATUS` / `V$OB_ARBITRATION_SERVICE_STATUS`；物理备份的 checkpoint_scn、Log Stream 归档目录与 standby 日志源语义可查；物理 standby 同步延迟监控。租户级参数 `arbitration_timeout` 用于仲裁超时（源码确认默认 5s、范围 [3s,+∞]）。

---

## 18.10 容易误解点

1. **「Witness = Learner = 仲裁成员」——错，且 Witness 与仲裁成员也不是同一种副本。** Learner（TiKV）是「存数据不投票」。Witness（TiKV）与 OceanBase 仲裁成员粗看都「轻量、不存全量用户数据」，但实质不同：（a）**存日志**——TiKV Witness 是 log-only 副本（存 Raft 日志、不 apply），OceanBase 仲裁成员连日志都不逐条同步、只存日志流元数据；（b）**投票语义**——TiKV Witness 正常运行时参与每次写多数派投票，OceanBase 仲裁成员正常不投日志、仅在半数全功能副本故障时做降级仲裁；（c）**成熟度**——OceanBase 仲裁服务 v4.1 起 GA，TiKV Witness 在 8.5.x/7.5.x 非 GA。底层实现（Rule 标志位 vs PALF 枚举 + 独立仲裁服务）亦不同，**三者均不可直接划等号**。测试要分别验证是否参与多数派、是否保存完整数据、是否可读、是否受版本/构建形态限制。另注：网络上「witness=不读、每次写投票、不存数据、不能当 leader」的四点定义源自 Spanner/CockroachDB，不应当作 TiKV 官方实现照搬。

2. **「降级到 async 就能永远可用」——错。** DR Auto-Sync 源码注释明确「若不再拥有多数派，切到 async 也无济于事」。降级只能在「还有多数派」时维持可用；真正丢失多数派时，降级无济于事。这是多数派共识的本质，不是 bug。

3. **「RPO=0 就等于自动无感切换」——错，二者不是同等可信、也不是同一回事。** RPO=0 是 Paxos/Raft 多数派同步提交的协议自洽强约束（只要写入提交即多数派落盘）；而 RTO 还取决于多数派是否仍在、路由刷新、角色切换、应用连接、数据校验与人工策略。RTO<8s 是 OceanBase 厂商自述/设计目标，随 **v4.0** 的选举/共识协议重构落地（由更早版本的 <30s 优化而来，**不是 4.1**），公开资料缺完全独立的生产实测来源，且高度依赖部署模式——仅多副本 Paxos 部署 <8s，shared-storage 单副本为分钟级。本章对这两类数值均按弱化处理，**证据有限**。

4. **「多地多活 = 每地都能强一致写同一数据」——错。** 受光速与多数派约束，全局强一致下跨地写必付跨城 RTT;TiDB/OceanBase 的「多活」是单元化多活/多地容灾，而非同一数据多地可写的 active-active（后者需放弃强一致，走冲突合并）。真正降低体验延迟通常要做业务分区、读本地化、异步复制或冲突规避。

---

## 18.11 本章结论

1. TiDB 副本角色底层只有 Voter 与 Learner 两类（`MetaPeerRole` 映射证实），Witness 是 Rule 上与 Role 正交的布尔标志（`is_witness`，priority -1、专用 BatchSwitchWitness 命令，local read 拒绝普通读），按设计是「存 Raft 日志、每次写都投票」的 log-only 机制，与「存数据不投票」的 Learner 不同，且在 8.5.x/7.5.x 非 GA。

2. OceanBase 的 Arbitration 仲裁成员是 PALF 枚举（`NORMAL_REPLICA`/`ARBITRATION_REPLICA`，跨 4.2.5 与 4.4.x 持续存在），配独立仲裁服务状态机（自 v4.1 GA，仅 2F+1A/4F+1A，社区版默认不编译）；它「只存日志流元数据、不存用户数据、正常运行时不逐条投日志、仅在半数全功能副本故障时做降级仲裁、不能被选主」。它与 TiKV Witness **不是同一种副本**——二者在存日志/投票语义/成熟度上均不同，**不可类比**。

3. TiDB DR Auto-Sync 对外正式枚举三态（sync/async/sync-recover，另有内部过渡态 async_wait；源码常量为小写下划线形式，commit-pinned 确认），`canSync = primaryHasVoter && drHasVoter`、多数派谓词语义为 `totalUpVoter*2 > totalVoter`；源码注释明确「多数派真正丢失时降级到 async 也救不了集群」，sync_recover 期 `ACIDConsistent=false`。其超时参数为 `wait-store-timeout`（默认 60s）与 `wait-recover-timeout`（默认 0s），**无 `arbitration_timeout`（那是 OceanBase 参数）**。

4. OceanBase 物理 standby 与物理恢复共用 `ObLogRestoreHandler`，通过 PALF raw-write 把主库日志（来自归档或网络服务）镜像到备库 PALF;archive-based 模式分钟级、network-based 模式秒级。「物理 standby 基于 PALF 日志镜像」得到模块级源码与 VLDB 2024 论文双重支撑；自 v4.1.0 起产品形态为租户级主备（主租户/备租户），不再有集群级主备角色。

5. 跨地域强同步写的延迟下限是跨城 RTT（光纤约 5μs/km，跨大陆往返数十毫秒），由多数派提交「最近副本」规则决定，任何软件优化都无法突破物理光速；TiDB/OceanBase 都用 Follower/弱一致读规避跨地域读延迟，但写延迟无法绕过。

6. 多地多活与全局强一致存在根本冲突：要本地低延迟写就得放松跨地一致（走最终一致/冲突合并），要全局强一致跨地写就得付多数派跨城 RTT;TiDB/OceanBase 均选强一致优先，其「多活」是单元化多活/多地容灾，而非同一数据多地可写的 active-active。

7. 集群内 RPO=0 是两系统共有的协议自洽强约束；RTO 指标（OceanBase 自述<8s，随 v4.0 架构重构落地、且仅多副本部署适用；TiDB 分钟级）均为厂商自述/设计目标且高度依赖部署条件，**证据有限**，不可当绝对排名；TiCDC 主从、BR/PITR 等异步方案 RPO 非零，不应当作「零丢失容灾」。

8. 金融级容灾不止是技术指标，而是合规（如分布式数据库金融应用规范灾难恢复要求）、监管（银行异地灾备等级要求）、定期演练、回切（switchover/failback）与数据校验的系统工程；据此**推测**，两系统的两地三中心/三地五中心方案正是面向这一监管语境设计。

---

## 18.12 参考文献

[1] Two Availability Zones in One Region Deployment（DR Auto-Sync）. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/two-data-centers-in-one-city-deployment/.
 （支撑:§18.2.2 DR Auto-Sync 对外三态、6 副本布局（3 Voter + 2 Voter + 1 Learner）、<50km/<1.5ms/10Gbps、wait-store-timeout 默认 60s）
[2] Placement Rules. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/configure-placement-rules/.
 （支撑:§18.2.1 PD Placement Rules 控制副本数量、位置、是否参与 Raft election 与 Leader 约束。）
[3] Three Availability Zones in Two Regions Deployment. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/three-data-centers-in-two-cities-deployment/.
 （支撑:§18.2.3 三地两中心 5 副本布局、同城<3ms/异地约20ms、Leader 限制在同城两 AZ。）
[4] Overview of TiDB Disaster Recovery Solutions / Replicate Data to MySQL-compatible Databases. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/dr-solution-introduction/.
 （支撑:§18.2.4 与 §18.4 多副本/TiCDC 主从/BR 各方案的 RPO/RTO 量级表；TiCDC eventually consistent replication v6.1.1 GA、RTO≤5min、P9）
[5] TiDB Log Backup and PITR Command Manual / Guide. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/br-pitr-manual/.
 （支撑:§18.2.4、§18.9 BR 日志备份、checkpoint[global]、PITR 恢复点与 v8.5.5 并行 PITR。）
[6] PALF: Replicated Write-Ahead Logging for Distributed Databases. 论文，VLDB 2024[EB/OL]. https://www.vldb.org/pvldb/vol17/p3745-xu.pdf.
 （支撑:§18.3.3 物理 standby 基于 PALF 日志镜像（restorers/archivers、archive vs net service）、PALF 作为复制式 WAL / 容灾原语基座。）
[7] Log streams / Use the arbitration service of OceanBase Database in a two-IDC scenario. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001105853.
 （支撑:§18.3 Log Stream 记录变更、Paxos 多副本、少数副本故障 RPO=0/RTO<8s 条件；§18.3.1 OceanBase 2F+1A、版本须晚于 V4.1.0.0、2F+1A/4F+1A 支持范围）
[8] Data transfer / Overview of physical backup and restore. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001714611.
 （支撑:§18.3.3/§18.3.4 physical standby 的 archive-based 与 network-based 两种模式及秒/分钟级差异、物理备份由 data backup 与 log archivin）
[9] pingcap/pd 仓库（release-8.5 @ commit 6dce4a68e3e9）— pkg/replication/replication_mode.go、pkg/schedule/placement/rule.go、server/config/config.go. 源码[EB/OL]. .
 （支撑:§18.2.1/§18.2.2 副本角色枚举（Voter/Learner + IsWitness 标志）、DR Auto-Sync 状态常量（对外三态 + 内部过渡态 async_wait）、canSync 与多数派谓词）
[10] oceanbase/oceanbase 仓库（v4.2.5_CE @ commit e7c676806fda；4.4.x 另据 d4bef8d29a4c 亦确认）— src/logservice/palf/log_define.h、src/share/ob_arbitration_service_status.h、src/logservice/restoreservice/ob_log_restore_handler.h、src/logservice/archiveservice/、src/share/parameter/ob_parameter_seed.ipp、src/share/inner_table/ob_inner_table_schema_constants.h. 源码[EB/OL]. .
 （支撑:§18.3.1/§18.3.3 LogReplicaType 枚举（NORMAL/ARBITRATION）、仲裁服务状态机、ObLogHandler 仲裁接口（OB_BUILD_ARBITRATION 条件编译）、物理）
[11] How We Reduced Multi-region Read Latency and Network Traffic by 50% / How Disaster Recovery (DR) Works in TiDB. 官方博客[EB/OL]. https://www.pingcap.com/blog/how-we-reduced-multi-region-read-latency-and-network-traffic-by-50/.
 （支撑:§18.2.3/§18.7 跨地域 TSO 一次 RTT 成本、Follower Read 优化、100ms 延迟下读延迟 200→100ms;2-2-1 强一致方案跨地域延迟代价。）
[12] [dr-autosync] v6.5.4 QPS drop during switch sync_recovery to sync（Issue #15366）. 源码/缺陷追踪[EB/OL]. https://github.com/tikv/tikv/issues/15366.
 （支撑:§18.7/§18.9 DR Auto-Sync sync_recover→sync 切换期 QPS 抖动真实缺陷（6.5.x/7.1.x LTS）。）
[13] The Fundamental Trade-offs in Distributed Databases. 第三方解读[EB/OL]. https://www.cockroachlabs.com/blog/fundamental-tradeoffs-distributed-databases/.
 （支撑:§18.7/§18.8 跨地域写延迟物理下限（光纤约200,000km/s、跨大陆往返≥37ms）、多数派「最近副本」规则、低延迟/区域容灾/强一致不可兼得。第三方厂商立场，作工程量级参考。）
[14] OceanBase 4.X-2F1A 仲裁高可用方案初探. 第三方解读[EB/OL]. https://cloud.tencent.com/developer/article/2451338.
 （支撑:§18.3.1 仲裁服务运行于 LS Leader、故障副本=full 半数时降级、仲裁成员只存日志流元数据 / 无 MemTable / SSTable / 不能选主、正常不逐条投日志。第三方深度解读，已与官方源码）
## 18.13 信息可信度自评

本章**官方文档明确**的部分：TiDB DR Auto-Sync 对外三态语义、6 副本布局、网络延迟门限、wait-store-timeout 默认 60s、wait-recover-timeout 默认 0s、三地两中心延迟数字、DR 方案 RPO/RTO 量级表、TiCDC v6.1.1 GA 与条件化指标、BR/PITR 能力，以及 OceanBase Log Stream / Paxos 多副本、arbitration service 2F+1A 边界、physical standby archive/network 两模式、物理备份 SCN/timestamp 恢复——分别来自 docs.pingcap.com 与 en.oceanbase.com。**源码级确认**：TiDB 副本角色枚举与 IsWitness 标志、DR Auto-Sync 状态常量与 canSync/多数派谓词/ACIDConsistent 逻辑、wait-store-timeout 默认常量（commit-pinned 至 pingcap/pd release-8.5）、TiKV `is_witness`/local read 拒读、log backup 指标；OceanBase LogReplicaType 枚举（NORMAL/ARBITRATION，跨 4.2.5 与 4.4.x）、仲裁服务状态机、ObLogHandler 仲裁接口（OB_BUILD_ARBITRATION）、ObLogRestoreHandler/ObStandbyService 物理 standby 注释（commit-pinned 至 oceanbase v4.2.5_CE，4.4.x 亦核对）。**论文/官方博客交叉**：PALF 物理 standby 原语（VLDB 2024 论文 + OceanBase 官方对论文定位的博客）。**工程量级参考（第三方）**：跨地域光速延迟下限来自 CockroachLabs 博客，作量级而非精确生产数据；2F1A 降级细节经第三方与官方源码枚举交叉验证。**工程推测**：金融级演练、回切、应用入口切换、审计留痕等运维影响。

**已实测补强（本轮升级）**：`arbitration_timeout` 默认值（5s）与范围（[3s,+∞]）经 OceanBase 源码 `ob_parameter_seed.ipp` 跨 4.2.5/4.3.5/4.4.x 确认并与官方文档一致；OceanBase 仲裁相关虚表/视图名（`__all_virtual_arbitration_member_info`、`__all_virtual_arbitration_service_status`、`DBA_OB_ARBITRATION_SERVICE`、`GV$/V$OB_ARBITRATION_MEMBER_INFO`、`GV$/V$OB_ARBITRATION_SERVICE_STATUS`）经源码 `ob_inner_table_schema_constants.h` 确认；TiKV log backup Prometheus 指标名经源码 `components/backup-stream/src/metrics.rs` 确认——三项已从「需进一步查证」升级为源码确认事实。

**仍不确定/标注谨慎**：OceanBase RTO<8s 为厂商自述/设计目标，缺独立生产实测，且仅多副本部署适用，已弱化；TiCDC 源码在独立仓库 tiflow，未做 commit-pinned，具体实现细节未核;OceanBase Primary Zone/Locality/仲裁成员降级因 CE 源码标识符脱敏，仅确认到模块级，未照搬脱敏符号名；「PALF group mirror」是否为官方公开术语未确认；DR Auto-Sync 在 v6.5 更早版本的具体特性名缺权威版本归属，保留谨慎，均未编造。

**反查证冲突记录**：
- **RTO 8s 的版本归属**：经多源核查，30s→8s 的恢复时间优化随 OceanBase **V4.0** 的核心选举/共识协议重构落地（2022 年发布），而非 4.1。锁定版本 4.2.5/4.3.5/4.4.x 继承该指标，但「4.1 起」为历史归属错误。此外 8s 为厂商自述/设计目标且仅多副本部署适用（shared-storage 单副本为分钟级），已弱化。
- **OceanBase「一主集群带 31 备集群」**：该数字属 v3.x **集群级主备**架构；自 **v4.1.0 起物理备库改为租户级主备（主租户/备租户），集群不再有主备角色概念**。锁定版本在 4.1.0 之后，「主集群带 N 个备集群」概念已失效，官方对租户级一主多备未公布固定上限（max value 文档列为「无限制」）。31 这一数字已删除。
- **`arbitration_timeout` 归属**：它是 OceanBase 租户级参数，TiDB/PD/TiKV 无此参数；DR Auto-Sync 超时参数为 wait-store-timeout（60s）/wait-recover-timeout（0s）。已把该参数限定在 OceanBase 一侧；其默认值（5s）与范围（[3s,+∞]）现已由 OceanBase 源码 `ob_parameter_seed.ipp` 跨 4.2.5/4.3.5/4.4.x 直接确认并与官方文档一致，已从「需进一步查证」升级为源码确认事实。
- **DR Auto-Sync 状态数**：官方对外正式枚举三态 sync/async/sync-recover;async_wait 为内部过渡态。源码层确有四个常量，但不宜把「四态集合」当作对外官方模型。
- **Witness 与仲裁成员**：经核查二者实质不同：TiKV Witness 是存 Raft 日志、每次写都投票的 log-only 副本，且在 8.5.x/7.5.x **非 GA**；OceanBase 仲裁成员不逐条同步日志、正常不投日志、仅故障时降级仲裁，自 v4.1 GA。坊间「witness 四点定义」实出自 Spanner/CockroachDB。已全章改写为「不可类比」。
- **版本核查备注**：本章引用版本（TiDB release-8.5、v8.5.5;OceanBase v4.2.5_CE、4.4.x）与锁定上下文一致；commit 统一用 release-8.5 / v4.2.5_CE 基线，4.4.x 仅作「另据…亦确认」对照，不双锁并存。

---


# 第 19 章 运维 / 观测 / 调优体系

## 19.1 本章核心问题

分布式数据库的"底层架构"不止于共识协议、存储引擎与事务模型，还包括一个常被低估却决定生产可用性的维度:当系统在线上抖动时，运维者能否在分钟级把"现象"还原为"根因"。一条 P99 突刺、一次热点写入、一轮 compaction 卡顿，在单机数据库里可能只是一行慢日志;在 TiDB 与 OceanBase 这类把数据切成 Region / Tablet、把状态散布到数十个节点、把控制面收敛到 PD / RootService 的系统里，同一现象的根因可能横跨 SQL 层、调度层、共识层、存储引擎层与代理层。

因此本章要回答的核心问题是:两套系统各自把"可观测性"做成了什么样的底层结构，这套结构如何支撑"指标 → 视图 → 日志 → 处置"的定位链路，以及它们在调优对象、故障定位、运维复杂度上的工程取舍。这不是产品功能罗列，而是要追踪观测数据的采集路径(数据从内核状态机如何流到一张视图 / 一个 metric)、聚合路径(如何按 digest / SQL_ID 收敛)与消费路径(运维者按什么链路逐层下钻)。

为把"指标 → 视图 → 日志 → 处置"写成可执行路径，本章先把观测信息分成三类，这条分类贯穿后文每一条定位链路。第一类是直接信号:延迟分位数、吞吐、错误率、CPU、I/O、队列时间、等待时间、compaction / merge 积压、日志错误;它们能说明"异常存在",但不能单独说明"根因成立"。第二类是关联键:TiDB 侧的 SQL digest、plan digest、Region ID、Store ID、PD 调度时间线,OceanBase 侧的 Tenant、trace id、SQL ID、LS ID、Tablet ID、ODP 路由信息;它们决定能否把多个面板和日志拼成同一事件。第三类是处置证据:限流、扩容、调度、调整 merge 窗口、修复执行计划或回滚发布之后，同一时间窗内 P99、错误率、重试与后台任务是否回落。缺第三类证据，很多"根因"只能算相关性。

一个关键判断贯穿全章:TiDB 的观测体系是外置组件拼装式(Prometheus + Grafana + Dashboard + 系统表),OceanBase 的观测体系是内核内置视图式(gv$ / v$ 虚拟表直接暴露内核环形缓冲)。这一根本差异决定了二者在"无外部依赖即可自诊断"与"需要完整监控栈才能看全"之间的取舍。这种差异也决定了两侧现场的典型陷阱:TiDB 现场容易在 TiDB Server、TiKV、PD、TiFlash、Prometheus 与日志之间跳转、丢失时间线;OceanBase 现场容易把 Tenant 资源、Unit 规格、ODP 路由、LS / Tablet 与 merge / compaction 混在一起讨论。两者都要求把同一时间窗、同一对象、同一 SQL 或同一 trace 固定下来再判断。本章与第 13 章(元数据与控制面)、第 4 章(存储引擎)、第 17 章(多租户)相邻，凡涉及 TSO / GTS、major freeze、租户隔离的底层机理，均只标"(详见第 X 章)"而不重复展开。

## 19.2 TiDB 的实现

**表 19-1　四类观测载体角色辨析**

| 载体 | 回答什么问题 | 聚合/采样方式 | 何时可信(交叉印证) | 能力盲区 |
|---|---|---|---|---|
| Top SQL | 哪些 SQL / plan 在采样窗口内消耗 CPU 或形成负载 | CPU 按 SQL 聚合,采样精度 1 秒、上报间隔 60 秒 | 与 Statement Summary、Slow Query 指向同一类 SQL 时可信度最高 | 不适合直接证明锁冲突、网络、GC / safepoint 或 RocksDB stall |
| Statement Summary | 同一 digest 的聚合行为是否变化 | LRU 缓存,按 SQL digest 与 plan digest 聚合 | 与 Top SQL、Slow Query 指向同一类 SQL | 内存有界(max_stmt_count 默认 3000、history 24 窗),高基数未参数化 SQL 会淘汰 |
| Slow Query | 哪些具体执行样本超过阈值或满足记录条件 | 超过 tidb_slow_log_threshold(默认 300ms)逐条记录 | 与 Top SQL、Statement Summary 指向同一类 SQL | 单次极慢但 Top SQL 不高时,可能是锁等待、网络等待或后台抖动 |
| Continuous Profiling | 各实例 CPU / Heap / Mutex / Goroutine 消耗在哪 | 持续从每个 TiDB / TiKV / PD 实例采集快照,渲染火焰图 | 与时间分解(database time)结论叠加 | CPU 归因类载体,write stall、TSO 等待、锁冲突等非 CPU 主因看不出 |

TiDB 的观测体系由四类组件拼装而成，彼此通过明确的数据通道耦合。TiUP / TiDB Operator 部署时默认拉起 Prometheus 抓取各组件(TiDB Server / TiKV / PD / TiFlash)暴露的 metrics endpoint,Grafana 提供 Performance Overview、TiKV-Details、TiKV-FastTune、TiKV-Trouble-Shooting、PD 等 dashboard;TiDB Dashboard 内嵌在 PD 中，提供 Top SQL、SQL Statements、Slow Queries、Key Visualizer 与诊断报告入口。锁定源码确认这一内嵌关系:`pingcap/pd`(release-8.5 @ `6dce4a68e3e9`)`pkg/dashboard/dashboard.go` 注册 Dashboard API / UI service builder,`pkg/dashboard/keyvisual/keyvisual.go` 把 Key Visualizer 数据源绑定到 PD core 的周期 getter,说明 Dashboard 与 PD 的 Region / 统计路径直接相连。

**(1) 监控数据面:Prometheus + Grafana。** Performance Overview dashboard 以"database time"为顶层抓手:database time 定义为 TiDB 每秒处理 SQL 的延迟之和，可沿三个维度拆解——按 SQL 类型(`DB Time = Select + Insert + Update + Delete + Commit`)、按处理阶段(`Get Token + Parse + Compile + Execute`)、按执行组件(`Execute ≈ TiDB Executor + KV Request + PD TSO Wait + Retried`)。这套自顶向下的 database-time 方法论是 TiDB P99 定位的主线([TiDB Performance Analysis and Tuning](https://docs.pingcap.com/tidb/stable/performance-tuning-methods/))。

**(2) SQL 观测面:Statement Summary。** 语句摘要在 TiDB Server 进程内以 LRU 缓存维护，按 SQL digest 与 plan digest 聚合 latency、执行次数、扫描行等统计。源码事实卡逐字确认:`pingcap/tidb`(release-8.5 @ `67b4876bd57b`)`pkg/util/stmtsummary/statement_summary.go` 中，全局入口 `var StmtSummaryByDigestMap = newStmtSummaryByDigestMap()`,核心类型为 LRU 缓存 `stmtSummaryByDigestMap`(注释:"a LRU cache that stores statement summaries");聚合键 `StmtDigestKey` 由 `schemaName, digest, prevDigest, planDigest, resourceGroupName` 构成;按 digest 聚合的 `stmtSummaryByDigest`、按时间窗的 `stmtSummaryByDigestElement`、执行信息输入结构 `StmtExecInfo`、写入入口 `AddStatement(sei *StmtExecInfo)` 均在该文件;v2 实现 `pkg/util/stmtsummary/v2/stmtsummary.go` 的 `GlobalStmtSummary` 维护窗口、存储与读取接口。默认值(逐字确认于 `pkg/sessionctx/variable/tidb_vars.go`,同 commit):`DefTiDBStmtSummaryRefreshInterval = 1800`(秒)、`DefTiDBStmtSummaryHistorySize = 24`、`DefTiDBStmtSummaryMaxStmtCount = 3000`、`DefTiDBStmtSummaryMaxSQLLength = 4096`,对应系统变量 `tidb_stmt_summary_refresh_interval` / `tidb_stmt_summary_history_size` / `tidb_stmt_summary_max_stmt_count`。

**(3) 慢查询面:Slow Query。** 慢日志默认阈值常量 `DefaultSlowThreshold = 300`(注释:"default slow log threshold in millisecond",`pkg/util/logutil/log.go` L43-44,release-8.5);系统变量 `tidb_slow_log_threshold` 默认值取该常量,`SetGlobal` 写入 `config.GetGlobalConfig().Instance.SlowThreshold`。慢日志可经 `INFORMATION_SCHEMA.SLOW_QUERY` / `CLUSTER_SLOW_QUERY` 表查询，或 `ADMIN SHOW SLOW recent N` / `ADMIN SHOW SLOW TOP N` 命令;内存中 TopN 慢查询由 `pkg/domain/topn_slow_query.go` 维护。慢日志逐字段记录 `Query_time / Parse_time / Compile_time / Process_time / Wait_time / Prewrite_time / Commit_time / Total_keys / Process_keys / Digest` 等(官方文档与源码一致,[Identify Slow Queries](https://docs.pingcap.com/tidb/stable/identify-slow-queries/))。

**(4) CPU 归因面:Top SQL + Continuous Profiling。** Top SQL(v5.4 GA)按 SQL statement 把各 TiDB / TiKV 实例的 CPU load 聚合，并展示 QPS、平均延迟与执行计划，精度可至 1 秒，官方称开启后平均性能影响通常 <3%。源码事实卡确认 `pkg/util/topsql/topsql.go` 中 `SetupTopSQL` 启动 reporter 并注册 statement stats collector,`AttachAndRegisterSQLInfo` 把 SQL digest 写入 context 并设置 goroutine labels,采样 collector 在 `collector/cpu.go`。Continuous Profiling(v5.3 引入)持续从每个 TiDB / TiKV / PD 实例采集 CPU / Heap / Mutex / Goroutine 快照，渲染为火焰图，官方称性能损耗 <0.5%。Top SQL 的采样精度与上报间隔默认值逐字确认于 `pkg/util/topsql/state/state.go`(release-8.5 @ `67b4876bd57b`):`DefTiDBTopSQLPrecisionSeconds = 1`(注释 "The refresh interval of top-sql",即采样精度 1 秒,与官方文档"precision of up to 1 second"一致)、`DefTiDBTopSQLReportIntervalSeconds = 60`(注释 "The report data interval of top-sql",即上报间隔 60 秒)、`DefTiDBTopSQLMaxTimeSeriesCount = 100`、`DefTiDBTopSQLMaxMetaCount = 5000`,均由 `GlobalState` 在进程内驱动(8.5 无对应系统变量)。Continuous Profiling 的采集间隔默认值由独立的 tidb-dashboard 仓库管理、不在本批 checkout 内,仍标"需进一步查证"。

这四类载体的角色必须分清，交叉印证才可信。Top SQL 回答"哪些 SQL / plan 在采样窗口内消耗 CPU 或形成负载";Statement Summary 回答"同一 digest 的聚合行为是否变化";Slow Query 回答"哪些具体执行样本超过阈值或满足记录条件"。三者指向同一类 SQL 时可信度最高;只有其中一个异常时，应继续查其他证据——例如 Top SQL 看到某 SQL CPU 占比高、但 slow query 不明显，可能是大量短查询造成总负载;slow query 看到单次极慢、但 Top SQL 不高，可能是少量锁等待、网络等待或后台抖动。也正因如此,Top SQL 适合高 CPU、高扫描量的负载归因，不适合直接证明锁冲突、网络、GC / safepoint 或 RocksDB stall(见 §19.10)。

**存储引擎观测:RocksDB 与 Raftstore。** TiKV 的尾延迟最终要落到存储引擎指标。源码确认 `tikv/tikv`(release-8.5 @ `1f8a140b6d46`)`components/engine_rocks/src/rocks_metrics_defs.rs` 含 `rocksdb.estimate-pending-compaction-bytes`、`rocksdb.num-files-at-level`、`io_stalls.slowdown_for_pending_compaction_bytes`、`io_stalls.memtable_compaction`、block cache hit / miss、WAL sync、compaction / flush 读写字节等内核属性名。这些内核属性进一步导出为 Prometheus metric 字符串,逐字确认于同仓 `components/engine_rocks/src/rocks_metrics.rs`(release-8.5 @ `1f8a140b6d46`):`tikv_engine_pending_compaction_bytes`(help "Pending compaction bytes")、`tikv_engine_stall_micro_seconds`(help "Stall micros",即 write stall 时长)、`tikv_engine_num_files_at_level`(标签 `db,cf,level`)、`tikv_engine_compaction_duration_seconds`、`tikv_engine_cache_efficiency`(block cache hit/miss)、`tikv_engine_wal_file_synced`(help "Number of times WAL sync is done")、`tikv_engine_write_stall_reason`、`tikv_engine_compaction_flow_bytes` 等,均经 `register_*_vec!` 注册,与官方 TiKV 监控文档一致([Key Monitoring Metrics of TiKV](https://docs.pingcap.com/tidb/stable/grafana-tikv-dashboard/))。细分 RocksDB 调优参数(如 `max-background-jobs`、`rate-bytes-per-sec`)随负载与 CPU 数动态推导,在正文不硬写具体默认值。

**控制路径与调度观测。** 热点 Region 由 PD 侧调度。源码事实卡确认 `pkg/schedule/schedulers/hot_region.go` 中热点调度器类型 `types.BalanceHotRegionScheduler`、核心结构 `hotScheduler`,并存 rank v1 / v2 两套热点排序算法(`hot_region_rank_v1.go` / `hot_region_rank_v2.go`),热点统计在 `pkg/statistics/hot_regions_stat.go`。Region 拆分阈值由 TiKV 侧 split-checker 决定，默认值随版本变化，书写时须锁定具体版本:`components/raftstore/src/coprocessor/config.rs` 逐字确认 `SPLIT_SIZE = ReadableSize::mb(256)`(8.5.x 默认 256MiB);同一文件在 release-7.5 分支为 `SPLIT_SIZE = ReadableSize::mb(96)`(7.5.x 默认 96MiB)。该默认值从 96MiB → 256MiB 的变更发生在 v8.4.0(官方 TiKV 配置文档明确:"region-split-size 默认 256MiB;Before v8.4.0, the default value is 96MiB",[TiKV Configuration File](https://docs.pingcap.com/tidb/stable/tikv-configuration-file/);v7.5 文档同步确认默认 96MiB)。本章锁定 v8.5.0 默认 256MiB,7.5.x 版本口径备注为 96MiB,不再以单一数值覆盖两版本。raftstore v2 的 `RAFTSTORE_V2_SPLIT_SIZE = ReadableSize::gb(10)`(10GiB,经 `optimize_for(raftstore_v2)` 启用)在 release-7.5 与 release-8.5 两分支均一致，无版本差异;`region_max_size = region_split_size / 2 * 3`(分片决定路由与共识单元，详见第 1、2 章)。

**GC / safepoint 观测。** `pkg/store/gcworker/gc_worker.go` 逐字确认 `gcWorkerTickInterval = time.Minute`、`gcDefaultRunInterval = time.Minute*10`、`gcDefaultLifeTime = time.Minute*10`;系统表键 `gcSafePointKey = "tikv_gc_safe_point"`(注释:"All versions after safe point can be accessed. (DO NOT EDIT)")、`gcLifeTimeKey = "tikv_gc_life_time"`;safepoint 推进入口 `w.saveTime(gcSafePointKey, *newSafePoint)`。GC 三步为 Resolve Locks → Delete Ranges → Do GC,safe point = 当前时间 − GC life time,且 safe point 不会超过正在运行事务的 start_ts——长事务因此会"顶住"safepoint(MVCC 历史版本清理，详见第 11 章)。

**诊断包面:PingCAP Clinic / Diag。** Clinic / Diag 是独立仓库工具(非核心数据库 checkout),可经 SCP、SSH、组件 HTTP、Prometheus HTTP 与 SQL 采集日志、配置、metrics 与变量信息，产出健康报告。本章对其只做角色断言，不做源码级断言。

控制面观测同样要避免"单点化"叙述。PD 负责 TSO、元数据、调度与内嵌 Dashboard,是排障必须观察的对象;但从故障分析角度，应分开看 PD leader 状态、调度积压、Region heartbeat、store 状态、TSO 延迟与业务请求是否同时异常。若业务 P99 上升而 PD 调度日志、Region 分布与 TiKV 存储面板没有同步异常，根因就不应直接落到 PD;反之,Region leader 迁移、调度剧烈变化或热点打散期间的短时抖动，也要用时间线证明它与业务异常重叠(控制面五维度见 §19.10)。 〔文献[1-3,5,9]〕

## 19.3 OceanBase 的实现

与 TiDB 的外置拼装相对,OceanBase 把可观测性内置进数据库内核:绝大多数诊断能力以 `gv$ / v$` 虚拟表(全局 / 本地视图)直接暴露，底层是内核维护的环形缓冲与内部基表，无需外部组件即可经 SQL 查询。外部工具层有 OBD、OCP、OCP-Agent、Prometheus 集成与 obdiag;内核层有 SQL Audit、SQL Trace、`GV$ / V$` 动态性能视图、Tenant / Unit 资源视图、LS / Tablet 状态视图、merge / compaction 视图与 Trace ID。OCP-Agent 文档说明 `ocp_monagent` 采集 OceanBase Database、ODP 与主机监控，按 Prometheus 格式生成时序数据，并额外采集 SQL Audit 数据用于 SQL 诊断与告警。

**(1) SQL 诊断核心:SQL Audit 与 SQL Trace。** SQL Audit 是 SQL 执行诊断视图，不等于合规 Security Audit。源码事实卡逐行确认:`oceanbase/oceanbase`(v4.2.5_CE @ `e7c676806fda`)中虚拟表迭代器类 `ObGvSqlAudit : public common::ObVirtualTableScannerIterator`(`src/observer/virtual_table/ob_gv_sql_audit.h`);视图与表 ID 为 `GV$OB_SQL_AUDIT` = 21014(`OB_GV_OB_SQL_AUDIT_TID`)、`V$OB_SQL_AUDIT` = 21026,底层基表 `__all_virtual_sql_audit` = 11031。`GV$OB_SQL_AUDIT` 从 `__all_virtual_sql_audit` 读取并暴露 `TRACE_ID`、`SQL_ID`、`QUERY_SQL`、`PLAN_ID`、`ELAPSED_TIME`、`QUEUE_TIME`、`EXECUTE_TIME`、`WAIT_TIME_MICRO`、`TOTAL_WAIT_TIME_MICRO`、`BLOCK_CACHE_HIT`、`DISK_READS`、`RETRY_CNT`、`RPC_COUNT`、`TRANS_STATUS` 等列(视图列定义逐字确认于 `ob_inner_table_schema_def.py`,`rpc_count as RPC_COUNT, elapsed_time as ELAPSED_TIME, queue_time as QUEUE_TIME, execute_time as EXECUTE_TIME, retry_cnt as RETRY_CNT`,table_id=21014)。后端环形缓冲由请求管理器维护:`ObMySQLRequestManager::record_request(...)`(`ob_mysql_request_manager.{h,cpp}`),页大小常量 `SQL_AUDIT_PAGE_SIZE = (1LL << 21) - ACHUNK_PRESERVE_SIZE`(注释 "2M - 17k")。开关参数 `enable_sql_audit`(集群级,`DEF_BOOL(enable_sql_audit, OB_CLUSTER_PARAMETER, "true", ...)`,默认开启、动态生效，逐字确认于 `src/share/parameter/ob_parameter_seed.ipp`);租户级另有 `ob_enable_sql_audit`、`ob_sql_audit_percentage`。SQL Audit 因内存有界，采用 FIFO 淘汰:源码逐字确认淘汰高 / 低水位线为按队列大小百分比设定的 `HIGH_LEVEL_EVICT_PERCENTAGE = 0.9`(90%)与 `LOW_LEVEL_EVICT_PERCENTAGE = 0.8`(80%)(注释 "按行淘汰的高低水位线，以 queue 大小的百分比设定",`src/observer/mysql/ob_mysql_request_manager.h` L84-86,v4.2.5_CE;4.3.5 同值)——占用达 90% 触发淘汰、回落至 80% 停止。诊断时通过 `RETRY_CNT`(重试次数高 → 可能锁冲突或切主)、`ELAPSED_TIME`、`QUEUE_TIME`、`EXECUTE_TIME`、`RPC_COUNT` 等列定位异常 RT 请求，官方诊断博客对这些列的归因描述与此一致([sql_audit 性能视图分析](https://en.oceanbase.com/blog/12526873344))。4.4.x(@ `d4bef8d29a4c`)中 `OB_GV_OB_SQL_AUDIT_TID = 21014` 不变，视图稳定。与 SQL Audit 配套,SQL Trace 把一次请求拆成可读阶段:`SHOW TRACE` 展示前一条 SQL 各阶段耗时，由 session 变量 `ob_enable_show_trace` 控制(默认关闭),源码 `last_trace_id()` 可从 physical plan context 读取最近 trace id,支撑"SQL Audit 查 trace id → SHOW TRACE / 日志按 trace id 搜索"的关联路径([SQL Trace](https://en.oceanbase.com/docs/common-oceanbase-database-10000000001106685))。

**(2) 合并 / Compaction 状态面。** 源码事实卡逐行确认视图名与表 ID:`GV$OB_COMPACTION_PROGRESS` = 21227 / `V$...` = 21228、`GV$OB_TABLET_COMPACTION_PROGRESS` = 21229、`GV$OB_TABLET_COMPACTION_HISTORY` = 21231、`GV$OB_MERGE_INFO` = 21118、`DBA_OB_MAJOR_COMPACTION` = 21186 / `DBA_OB_ZONE_MAJOR_COMPACTION` = 21184;底层基表 `__all_virtual_server_compaction_progress` = 11107、`__all_zone_merge_info` = 392、`__all_merge_info` = 393。运维上,`CDB_OB_MAJOR_COMPACTION` / `DBA_OB_MAJOR_COMPACTION` 的 `STATUS` 为 `COMPACTING` 表示合并进行中、回到 `IDLE` 表示完成——此状态映射逐字确认于源码视图定义:`DBA_OB_MAJOR_COMPACTION`(table_id=21186)视图为 `(CASE MERGE_STATUS WHEN 0 THEN 'IDLE' WHEN 1 THEN 'COMPACTING' WHEN 2 THEN 'VERIFYING' ELSE 'UNKNOWN' END) AS STATUS`(`ob_inner_table_schema_def.py` L20499 起,v4.2.5_CE;4.3.5 同);`GV$OB_COMPACTION_PROGRESS` 的 `STATUS=NODE_RUNNING` 与 `UNFINISHED_TABLET_COUNT`(长时间不更新即可疑卡合并)用于查未完成 tablet——`UNFINISHED_TABLET_COUNT` 为该视图基表 `__all_virtual_server_compaction_progress` 列(逐字确认于 `ob_inner_table_schema_def.py` L9685、视图定义 `ob_inner_table_schema.21201_21250.cpp`),`NODE_RUNNING` 为 DAG 状态枚举字面量(`ObIDag::ObIDagStatusStr[]` 第 2 项 = `"NODE_RUNNING"`,`src/share/scheduler/ob_dag_scheduler.cpp` L344)。merge / compaction 诊断要落到 Tablet / LS 层:源码确认 `ob_all_virtual_tablet_compaction_info.cpp` 迭代当前 Tenant 的 Tablet,读取 LS ID、Tablet ID、major snapshot version 与 medium compaction 信息。这些列名与状态值的语义与官方诊断博客一致([Diagnosis and Tuning: How to Troubleshoot Compaction Issues?](https://en.oceanbase.com/blog/13455861760))。

**(3) LS / Tablet / 日志状态面。** `DBA_OB_LS` = 21318 / `CDB_OB_LS` = 21319、`GV$OB_LOG_STAT` = 21302(PALF 复制式 WAL 的日志副本状态),基表 `__all_virtual_ls_info` = 12287、`__all_virtual_tablet_info` = 12288。Tablet 可在 Log Stream 之间迁移以实现负载均衡(LS 取代早期"每分区一 Paxos 组"模型，详见第 1 章;[OceanBase Architecture](https://oceanbase.github.io/oceanbase/architecture/))。4.4.x 中 `OB_DBA_OB_LS_TID = 21318`、`OB_GV_OB_COMPACTION_PROGRESS_TID = 21227` 不变。

**(4) 租户资源 / 会话 / 事务面。** `GV$OB_TENANT_MEMORY` = 21146、`GV$OB_UNITS` = 21217、`DBA_OB_TENANTS` = 21158、`GV$OB_TENANT_RESOURCE_LIMIT` = 21550;会话 / 事务 / 计划缓存(慢 SQL / 锁冲突 / 长事务定位)由 `GV$OB_PROCESSLIST` = 21221、`GV$OB_SESSION` = 21459、`GV$OB_PLAN_CACHE_STAT` = 20001、`GV$OB_TRANSACTION_PARTICIPANTS` = 21225、`GV$OB_TRANSACTION_SCHEDULERS` = 21353 暴露。此外 OceanBase 4.x 引入 ASH(Active Session History,`GV$ACTIVE_SESSION_HISTORY`,借鉴 Oracle 机制),按固定间隔对活跃会话采样，用于负载剖析。采样间隔约 1 秒逐字确认于源码:`const static int64_t REFRESH_INTERVAL = 1 * 1000L * 1000L;`(注释 "refresh sess info every 1 second",`src/share/ash/ob_active_sess_hist_task.cpp` L65-66,v4.2.5_CE;4.3.5 同),虚拟表接口为 `ObVirtualASH`(`src/observer/virtual_table/ob_virtual_ash.{h,cpp}`)([GV$ACTIVE_SESSION_HISTORY V4.2.0](https://en.oceanbase.com/docs/common-oceanbase-database-10000000001030063))。

**memstore / freeze / 写限流调优参数。** 逐字确认于 `ob_parameter_seed.ipp`(v4.2.5_CE):集群级 `memstore_limit_percentage` 默认 "0"(表示自动)、租户级 `_memstore_limit_percentage` 默认 "0";`freeze_trigger_percentage`(租户级，默认 "20")—— memstore 触发 minor freeze 的占比阈值;`writing_throttling_trigger_percentage`(租户级，默认 "60")—— 写入限流触发占比;`major_freeze_duty_time`(默认 "02:00")、`merger_check_interval`(默认 "10m")。官方调优博客建议 OLTP 场景把 `freeze_trigger_percentage` 调到 60–75 区间以平衡读写，这是运维建议值而非源码默认值，二者不冲突([Manage memory in OceanBase](https://oceanbase.medium.com/manage-memory-in-oceanbase-database-2e0f79aade73))。

**诊断包面与工具边界。** obdiag 是官方 CLI 诊断工具，可扫描、收集并分析 OceanBase 日志、SQL Audit 记录与进程栈，支持 check / gather / analyze / rca / display;它不是内核组件，而是诊断包与 RCA 工具，场景化采集覆盖 compaction、cpu_high、io、long_transaction、memory、perf_sql、rootservice_switch、suspend_transaction、obproxy_log 等方向。ODP 路由诊断不在 oceanbase / oceanbase 主仓库内(独立仓库 `oceanbase/obproxy`),其连接诊断依赖 `obproxy_diagnosis.log` 的 `trace_type` 字段(CLIENT_VC_TRACE / SERVER_VC_TRACE / SERVER_INTERNAL_TRACE / PROXY_INTERNAL_TRACE)定位断连方，仅官方文档层级，不做源码级断言。OCP / OCP-Agent 与 obdiag 的分工要分清:前者是持续监控与管控平面，适合保存趋势、发现告警、进入租户与主机维度;后者是事件发生后的诊断采集与分析工具，适合把日志、SQL Audit、stack、perf、obproxy_log 打包。生产现场通常先用 OCP 或 Prometheus 面板确定异常窗口，再用 SQL Audit / Trace 固定请求，再用 obdiag 或日志检索保留证据。

**采集路径小结。** OceanBase 观测的采集路径有一条共性:内核状态机在执行 SQL、做合并、维护副本时，同步把关键状态写进进程内的内存结构(环形缓冲、统计计数器),再由虚拟表迭代器(如 `ObGvSqlAudit`)在被查询时按行输出。这意味着诊断数据"天生在库内、随 SQL 可取",不必经过"导出到外部时序库再回查"的环节;代价则是这些内存结构容量有限，一旦被新数据冲刷，历史就需要靠 OBAgent → Prometheus 的时序采集或 OCP / obdiag 的归档来留存。这与 TiDB"进程内采集 + 外部聚合消费"的两段式形成镜像对照，也正是 §19.4 主对比表第一行差异的底层成因。 〔文献[6-8,10-15]〕

## 19.4 核心差异对比

**表 19-2　运维/观测/调优体系:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB | OceanBase | 影响 |
|---|---|---|---|
| 观测架构形态 | 外置组件拼装(Prometheus + Grafana + Dashboard + 系统表) | 内核内置 gv$ / v$ 虚拟表 | OB 无外部依赖即可 SQL 自诊断;TiDB 看全集群需完整监控栈 |
| 观测轴心 | 组件 + Region(TiDB Server / PD / TiKV / TiFlash / Region / Raft Group) | Tenant + OBServer + Unit + LS + Tablet | TiDB 按组件拆图,OB 按租户与内核对象追踪 |
| SQL 诊断主载体 | Statement Summary(LRU,按 digest 聚合)+ Slow Query | SQL Audit(内存环形缓冲,FIFO 淘汰)+ SQL Trace + ASH | 都内存有界、按指纹聚合;OB 经 SQL 直查,TiDB 经系统表 / Dashboard |
| CPU / 阶段归因 | Top SQL(CPU 按 SQL 聚合)+ Continuous Profiling 火焰图 | SQL Audit 阶段与等待列 + ASH(会话采样)+ obdiag perf | TiDB Top SQL 偏 CPU 归因;OB SQL Audit / Trace 偏请求阶段与等待事件 |
| 慢 / 异常 SQL 阈值 | `tidb_slow_log_threshold` 默认 300ms | `enable_sql_audit` 全量审计(非阈值制) | TiDB 阈值过滤减负;OB 全量但内存淘汰、保留窗口取决内存 |
| 关联键 | SQL digest / plan digest / Region ID / Store ID | Tenant / trace id / SQL ID / LS ID / Tablet ID | 决定能否把多面板与日志拼成同一事件 |
| 调度 / 热点观测 | PD hot_region scheduler + Key Visualizer 热力图 | RootService balancer + LS / Tablet 视图 | TiDB 热点可视化更成熟;OB 热点须多视图组合判断 |
| 后台任务抖动 | RocksDB compaction / write stall、Raftstore、GC safepoint | major / minor merge、compaction、memstore、Tablet medium compaction | TiDB 看 RocksDB stall 与 Region;OB 看 merge 进度与 Tenant 资源 |
| 诊断工具 | PingCAP Clinic / Diag(远程采集 + 健康报告) | obdiag(gather / analyze / check / rca) | 二者都是独立仓库工具，提供 RCA 自动化 |
| 数据保留 | Statement Summary 历史 24 窗 × 30min;Top SQL 30 天 | SQL Audit / ASH 内存有界，落盘需外部归档 | OB 内存视图易丢历史，长期分析须导出 |

这张主对比表的核心结论:两套体系不是"谁更全",而是把可观测性放在了不同层。TiDB 把观测做成可插拔的外部生态，优点是组件成熟、可视化强、与云原生监控栈天然融合，代价是依赖完整监控栈、组件多、现场易在多组件间丢失时间线;OceanBase 把观测内置为内核虚拟表，优点是零外部依赖、SQL 即诊断、与 Oracle 运维习惯对齐，代价是内存视图保留窗口短、维度多需组合判断、长期趋势分析须外部归档。表 19-3 进一步把常见线上问题映射到两侧的定位主载体:

**表 19-3　运维/观测/调优体系:TiDB 与 OceanBase 核心差异对比**

| 线上问题 | TiDB 定位主载体 | OceanBase 定位主载体 |
|---|---|---|
| P99 抖动 | Performance Overview(database time)→ TiKV-FastTune | sql_audit RT 列 + ASH + OCP |
| 热点 Region / 热点 LS | PD hot region + Key Visualizer | LS / Tablet 视图 + RootService balancer 状态 |
| compaction / merge 抖动 | TiKV RocksDB write stall / pending bytes | GV$OB_COMPACTION_PROGRESS + observer.log |
| 锁冲突 | DATA_LOCK_WAITS / DEADLOCKS / TIDB_TRX | GV$OB_TRANSACTION_PARTICIPANTS + 错误码 6004 / 6005 |
| 慢 SQL | Statement Summary / SLOW_QUERY / Dashboard | GV$OB_SQL_AUDIT + SHOW TRACE + obdiag gather sql_audit |
| 统计信息过期 | auto analyze 状态 + `tidb_enable_pseudo_for_outdated_stats` | 计划缓存 + outline + DBMS_STATS |
| GC / safepoint 卡住 | tikv_gc_safe_point + 长事务 start_ts | undo / 多版本保留(机理详见第 11 章) |
| 路由失败 / 连接堆积 | RegionCache 失效 + TiDB 日志 | obproxy_diagnosis.log 的 trace_type |

## 19.5 正常路径图(P99 定位的观测数据流)

图 19-1 展示两套系统在"正常运维"下，一次 SQL 执行如何把观测数据沉淀到各诊断载体，以及运维者自顶向下的下钻路径。它的关键不在箭头数量，而在"同一请求如何留下可关联证据":TiDB 侧常用 SQL digest、plan digest、Region / Store 信息与时间戳串联,OceanBase 侧常用 Tenant、SQL ID、trace id、LS / Tablet 与时间戳串联。

![[f19_1.svg]]

**图 19-1　运维/观测/调优体系正常读写/调度路径**

正常路径的关键区别清晰可见:TiDB 的观测数据要"流出"到 Prometheus / Dashboard 才能被消费，而 OceanBase 的核心诊断数据"留在"内核里，运维者用一条 SQL 即可拉取。Dashboard、OCP 与 Grafana 给出趋势,SQL 视图给出请求级或聚合级解释，日志给出组件内部状态，诊断包负责把现场冻结下来;若缺少其中任一层，仍可定位一部分问题，但结论应降级为"可能"或"需进一步验证"。

## 19.6 故障 / 异常路径图(P99 抖动定位链路)

图 19-2 把两侧典型 P99 抖动的"指标 → 视图 → 日志 → 处置"故障定位链路统一表达:从一个 P99 抖动告警出发，先做分层定位(客户端 / SQL 层 / 存储层 / 控制面),再用关联键收敛到同一事件，落到日志与诊断包，最后给出可回滚的处置。

![[f19_2.svg]]

**图 19-2　运维/观测/调优体系故障/异常路径**

两侧的具体落点不同。TiDB 一支典型链路(写入 P99 突刺，疑似热点 Region / write stall):指标层 Grafana Performance Overview 见 Duration P99 抬升、Execute Time 中 Commit / Prewrite 占比升高 → 下钻 TiKV-Details 见 RocksDB `Write stall duration` > 0、`Compaction pending bytes` 堆积，或 PD 见 hot region 写流量集中单 store / 单 Region → 视图 Key Visualizer 出现亮带(热点 key range)、`pd-ctl hot write` → 日志 TiKV split-check / apply 延迟、TiDB slow log → 处置 `SHARD_ROW_ID_BITS` / `AUTO_RANDOM` 打散、调 region split、调 PD 热点调度，必要时 PingCAP Clinic 采集时间窗。OceanBase 一支典型链路(读 P99 突刺，疑似 major compaction 抖动 / memstore 写限流):指标层 OCP / Prometheus 见租户 RT 抬升 → 视图 `GV$OB_SQL_AUDIT` 见 `RETRY_CNT` 升高 / `EXECUTE_TIME` 拉长、`GV$OB_COMPACTION_PROGRESS` 见 `STATUS=NODE_RUNNING` 且 `UNFINISHED_TABLET_COUNT` 居高(合并卡住)、`GV$OB_TENANT_MEMORY` 见 memstore 逼近 freeze 阈值 → 日志 observer.log 的 minor freeze / writing throttling 记录 → 处置 错峰 `major_freeze_duty_time`、调 `freeze_trigger_percentage` / `writing_throttling_trigger_percentage`,或 `obdiag rca run --scene=major_hold` 自动根因分析。

异常路径要特别警惕"同症不同因"。同样是 P99 上升,TiDB 可能是单 Region 热写导致 TiKV 局部 I/O 排队，也可能是统计信息变化导致执行计划从点查变成大范围扫描，还可能是长事务拖住 GC / safepoint 后形成版本链压力;OceanBase 可能是 Tenant 资源队列变长，也可能是某个 LS / Tablet 在 merge / compaction 期间放大 I/O,或是 ODP 路由、后端连接、事务等待造成请求排队。只看客户端 P99，这些场景几乎不可区分;只有把 SQL、对象、后台任务与日志放进同一时间轴，才能把"并发出现"与"因果触发"分开。这两条链路也印证第 4 章的结论:TiKV 的 compaction 本地自治、抖动表现为 write stall;OceanBase 的 major compaction 由 RootService 协调全局版本，抖动表现为合并卡顿 + 写限流，因此定位入口与处置手段完全不同。控制面不可用(PD 选主中 / RootService 重建)时，两侧热点调度与负载均衡都会暂停，但读写路径因缓存 / Leader 仍在而通常不立即中断(详见第 13 章)。 〔文献[4]〕

## 19.7 性能、可靠性、运维影响

**表 19-4　调优旋钮分层对照**

| 抽象层 | TiDB 旋钮(默认值) | OceanBase 旋钮(默认值) | 作用 |
|---|---|---|---|
| SQL / 计划 | tidb_auto_analyze_ratio(0.5)、tidb_enable_pseudo_for_outdated_stats(—) | 计划缓存 / outline / DBMS_STATS(—) | 统计信息过期触发自动 analyze、控制计划是否退化与稳定 |
| 内存 / freeze | RocksDB block cache(—) | freeze_trigger_percentage(20)、writing_throttling_trigger_percentage(60) | OB 侧 memstore 触发 minor freeze 与写入限流的占比阈值 |
| 存储 / split | region split size(v8.5.0 默认 256MiB;7.5.x 96MiB) | LS / Tablet 均衡(—) | 分片大小决定路由与共识单元、负载倾斜 |
| 合并 / compaction | RocksDB compaction(—) | major_freeze_duty_time(02:00)、merger_check_interval(10m) | 全局 major 合并触发时间与合并检查间隔 |
| 调度 / 均衡 | PD 热点调度 / Region 均衡权重(—) | RootService balancer / LS 均衡(—) | 热点打散与跨节点负载均衡 |

**延迟与吞吐定位效率。** TiDB 的 database-time 方法论把 P99 拆成 SQL 类型 / 阶段 / 组件三维，配合 Performance Overview 的颜色编码(蓝 = 读 Cop / Get、绿 = 写 Prewrite / Commit、深棕 = TSO wait),能快速判断瓶颈在 TiDB Executor、KV 请求还是 TSO 等待。写路径的 `Storage Async Write Duration = Store Duration + Apply Duration`、`Commit Log Duration`(Raft 复制)、`Append / Apply Log Duration` 进一步把尾延迟归到共识层还是落盘层。OceanBase 则以 SQL Audit 的逐请求列(`QUEUE_TIME` / `EXECUTE_TIME` / `TOTAL_WAIT_TIME_MICRO` / `RPC_COUNT` / `RETRY_CNT`)+ `SHOW TRACE` 阶段耗时 + ASH 采样实现等价归因:若等待占比上升，优先进入锁、资源队列、I/O 与后台任务方向;若执行耗时上升，优先进入计划、扫描行、缓存与并发方向。OB 的优势是粒度到单次请求、无需预聚合，代价是内存淘汰使历史窗口短。

**P99 抖动定位案例(TiDB)。** 假设某 OLTP 集群每天上午整点出现写入 P99 从 5ms 抖到 80ms 的周期性突刺。按 database-time 主线:先在 Performance Overview 看 Duration 面板确认 P99 抬升集中在 Commit / Prewrite(绿色块变厚),排除读路径与 TSO;再下钻 TiKV-Details / TiKV-FastTune,发现 `Write stall duration` 在突刺时段 > 0、`Compaction pending bytes` 同步堆积，指向 RocksDB 写堆积;继续看 PD,`hot region` 写流量集中在少数 Region / store,Key Visualizer 出现亮带，定位到业务用了单调递增主键导致写热点。处置:对热点表启用 `SHARD_ROW_ID_BITS` / `AUTO_RANDOM` 打散写入，并视情况调 PD 热点调度权重;若需保留现场供原厂分析，用 PingCAP Clinic 一键采集这段时间窗的 metrics + 日志 + 配置。整条链路严格遵循"指标(Duration P99)→ 指标下钻(write stall)→ 视图(Key Visualizer / hot region)→ 日志(TiKV split-check)→ 处置(打散热点)"。

**P99 抖动定位案例(OceanBase)。** 同类周期性突刺在 OceanBase 上的典型成因是 major compaction 与业务高峰撞车。定位链路:先查 `GV$OB_SQL_AUDIT` 按 SQL 指纹聚合，发现突刺时段 `EXECUTE_TIME` 整体拉长且部分请求 `RETRY_CNT` 升高;查 `GV$OB_COMPACTION_PROGRESS` 发现 `STATUS=NODE_RUNNING`、`UNFINISHED_TABLET_COUNT` 居高，确认 major compaction 正在进行;再查 `GV$OB_TENANT_MEMORY` 看 memstore 是否同时逼近 `freeze_trigger_percentage`,若 observer.log 出现 writing throttling 记录则说明叠加了写限流。处置:把 `major_freeze_duty_time` 错峰到真正的业务低谷、按读写比把 `freeze_trigger_percentage` 调到 60–75、必要时调 `writing_throttling_trigger_percentage`;若仍不确定根因，可直接运行 `obdiag rca run --scene=major_hold`，让工具串起视图与日志给出"卡合并"根因。两个案例对照可见:同一现象(周期性 P99 突刺),TiDB 入口是 write stall + 热点 Region,OceanBase 入口是 compaction 进度 + memstore 阈值，处置手段也分别落在"打散 KV 热点"与"错峰全局合并 + 调写限流"两个完全不同的旋钮上。

**可用性观测。** TiDB 经 PD 的 Region heartbeat、TiKV 的 raftstore CPU(应 <80% × store-pool-size)、单 TiKV region count(建议 <50K)等指标判断集群健康;OceanBase 经 `GV$OB_LOG_STAT` 看 PALF 副本状态、`DBA_OB_LS` 看 LS 分布。两侧的故障切换 RPO / RTO 指标差异详见第 3、13 章，此处不重复。

**扩展性与故障恢复运维。** TiDB 扩容靠 PD 把 Region 调度到新 store,运维者用 `pd-ctl` / Grafana 观察迁移进度;OceanBase 扩容靠 RootService balancer 迁移 Tablet / Unit,经 LS / Tablet 视图观察。GC / safepoint 卡住是 TiDB 特有运维风险:当监控显示 TiKV auto GC safepoint 长时间不推进，通常意味着存在老 start_ts 的长事务顶住了 safepoint,或 TiDB 侧 GC worker 异常，需查 `mysql.tidb` 表的 `tikv_gc_safe_point` 与运行中事务。

**调优对象的层次差异。** 两套系统的"可调旋钮"分布在不同抽象层。TiDB 的调优对象按组件横向铺开:TiDB Server 侧调 `tidb_tso_client_rpc_mode`(DEFAULT / PARALLEL,缓解 TSO 等待占比)、plan cache、`tidb_distsql_scan_concurrency`;TiKV 侧调 region split size(v8.5.0 默认 256MiB;7.5.x 为 96MiB,见 §19.2)、RocksDB block cache、scheduler / coprocessor 线程池、GC 并发;PD 侧调热点调度策略、Region 均衡权重;TiFlash 侧调 MPP 并行度(详见第 16 章)。统计信息过期是高频调优点——`tidb_auto_analyze_ratio`(默认 0.5)触发自动 analyze,`tidb_enable_pseudo_for_outdated_stats` 控制统计过期后是否退化为伪统计而引发计划突变。OceanBase 的调优对象则以租户为中心收敛:租户资源(Resource Unit 的 CPU / 内存 / IOPS / 磁盘)、memstore(`memstore_limit_percentage` / `freeze_trigger_percentage`)、kv cache / block cache、I/O(min / max / weight iops,详见第 17 章)、merge / compaction(`major_freeze_duty_time` / `merger_check_interval`)、PX 并行度、ODP 路由与 LS 均衡。这一差异是架构投射:TiDB 无独立租户内存空间，旋钮按物理组件分布;OceanBase 原生多租户，旋钮天然挂在租户级参数上。

**观测体系自身的可靠性与成本。** 观测平面本身也有故障模式:Prometheus 或 OCP-Agent 抓取失败造成指标缺口，日志轮转或保留周期过短丢失关键窗口,SQL Audit / Trace 的开关与采样策略影响证据完整度,Dashboard 权限或集群元信息不一致影响现场协作。工程上应把观测平面纳入演练——确认告警能触发、诊断包能采集、慢日志与 SQL Audit 能查询、trace id 能落到日志、Grafana / OCP 面板与数据库时间同步，否则真正故障时团队会先排查观测工具、错过数据库现场。细粒度观测也不是免费的:Top SQL、Continuous Profiling、SQL Audit、Trace、详细日志都带来采样、存储、权限与性能开销，合理做法是分层策略——常态保留低成本指标与聚合摘要，告警时提高采样或打开 trace,故障后及时恢复默认级别，而非长期打开所有开关。

## 19.8 反例与代价

**TiDB 侧代价。** 以下几项代价偏工程推测，需结合具体部署验证。(1)观测依赖外部栈:Statement Summary、Top SQL 数据虽在进程内，但全集群关联分析、长期趋势、跨组件下钻强依赖 Prometheus + Grafana + Dashboard;一旦监控栈本身故障或采样间隔过粗,P99 定位将失去依据，这在没有部署完整监控的小集群或边缘场景是真实痛点。(2)Statement Summary / Top SQL 内存有界:`tidb_stmt_summary_max_stmt_count=3000`、history 24 窗,SQL 基数极高(大量未参数化 SQL)时会淘汰，丢失低频但关键的慢 SQL。(3)GC safepoint 与长事务耦合:长事务 / 僵尸事务会顶住 safepoint 导致空间不回收，是典型运维塌陷点。(4)Top SQL 能力边界:官方明确其适合高 CPU load SQL 归因，不适合非性能问题与事务锁冲突这类非 CPU 主因问题。

**OceanBase 侧代价。** 以下几项同属工程推测，可信度低于前文的源码事实。(1)内存视图保留窗口短:SQL Audit、ASH 都是内存环形缓冲、FIFO 淘汰,SQL Audit 高水位 90% 触发淘汰至低水位 80%(源码逐字确认，见 §19.3);突发流量下历史很快被冲掉，事后复盘必须提前经 obdiag / OCP 归档，否则现场无从追溯。SQL Trace 默认关闭、SQL Audit 受配置 / 采样 / 保留窗口 / 权限影响，源码能确认 `ob_sql_audit_percentage`、`ob_enable_sql_audit` 等名称，但现场值与保留策略需另查。(2)视图众多、需组合判断:OceanBase 没有 TiDB Key Visualizer 那样的一图热点可视化，热点 LS / Tablet 要联查多张视图 + RootService 状态，对运维者的内核理解要求更高。(3)major compaction 抖动是固有代价:全局 major freeze 由 RootService 选版本号、跨副本协调(详见第 4 章),即便错峰也可能与业务高峰冲突,`freeze_trigger_percentage` / `writing_throttling_trigger_percentage` 调参不当会引发写限流连锁。

**共同的塌陷点。** 下列共性塌陷点为工程推测，宜按现场证据逐项核对。(1)观测反噬性能:TiDB 的 Continuous Profiling / Top SQL 虽声称 <0.5% / <3% 损耗，但在高并发短查询场景,CPU 采样与上报本身会和业务争抢 CPU;OceanBase 的 SQL Audit 全量审计在 SQL 基数爆炸时会快速消耗审计内存、加剧淘汰。(2)集群级指标遮蔽单热对象:TiDB 集群级 CPU 正常并不排除单 Region 热点,OceanBase 集群级 CPU 正常也不排除某 Tenant、LS 或 Tablet 局部热点。(3)采样间隔 > 抖动持续时间:若 P99 突刺只持续几百毫秒，而 Prometheus 抓取间隔是 15s、ASH 采样是 1s,突刺可能在时序图上被平滑掉，只在逐请求的 slow log / sql_audit 里留痕——这解释了为什么排查瞬时抖动必须回到"逐条记录"而非"聚合曲线"。(4)证据越多越确定的幻觉:不同来源的时间戳若没对齐，证据越多越容易拼错故事。TiDB 中 TiDB Server / TiKV / PD 日志、Prometheus 样本与 Dashboard 事件,OceanBase 中 SQL Audit、observer 日志、obproxy 日志、OCP 告警与 obdiag 采集，都应校准时间;跨时区、机器时间漂移、采样粒度差异与查询窗口过宽都会制造虚假关联。因此每次处置都应记录三项:变更前的证据、变更动作的精确时间、变更后的指标恢复情况。

**基准与生产不可直接套用。** 这一判断属工程推测，但与多数现场经验吻合:TPC-C / TPC-H 或公开压测能展示某些负载下的能力，而生产 P99 往往由发布、缓存、流量突刺、热点 key、后台任务、长事务、权限变更、备份恢复、网络抖动或运维操作触发;调优结论必须写明负载模型、数据规模、版本、参数、拓扑与观测工具，否则"某项指标提升"不能迁移为通用工程结论。

**取舍本质。** 两者都不是"更先进",而是把观测复杂度放在了不同位置:TiDB 放在外部生态的运维与集成上,OceanBase 放在内核视图的学习曲线与内存约束上。延迟敏感、需长期趋势审计的场景,TiDB 的外置时序库更友好;追求零外部依赖、SQL 即诊断、与 Oracle 运维习惯对齐的场景,OceanBase 的内核视图更顺手。无论哪套，真正的工程纪律是:先用时间分解定位"瓶颈在哪个阶段",再决定下钻哪类载体(火焰图 / 视图 / 日志),避免一上来就盯着单一指标做归因。

## 19.9 测试开发视角的验证点

**可测试的功能场景。**(1)注入热点写入(单调递增主键)验证 TiDB Key Visualizer 是否出现亮带、PD 是否触发 hot region 调度;在 OceanBase 验证 LS / Tablet 视图是否反映负载倾斜。(2)构造高基数未参数化 SQL,验证 Statement Summary 淘汰(`tidb_stmt_summary_max_stmt_count`)与 SQL Audit FIFO 淘汰行为。(3)启动长事务，验证 TiDB safepoint 是否停止推进、GC 是否阻塞。

**可注入的失效模式。** kill TiKV / OBServer 节点观察 Leader 切换在 `GV$OB_LOG_STAT` / PD 上的反映;人为触发 major compaction 高峰观察写限流;制造锁冲突观察 `DATA_LOCK_WAITS` / `GV$OB_TRANSACTION_PARTICIPANTS`;断开 PD / RootService 观察热点调度暂停而读写不立即中断;制造 TiKV 节点磁盘 I/O 降速、ODP 后端连接堆积、统计信息过期等场景。

**端到端案例(TiDB)。** 写入峰值后 P99 从 20ms 升到 200ms,Grafana TiKV 面板出现 `rocksdb.estimate-pending-compaction-bytes` / `io_stalls.slowdown_for_pending_compaction_bytes` 相关上升,Top SQL 显示某写入 SQL CPU 占比上升,Key Visualizer 出现单表或单索引亮带。链路:指标 → Grafana Overview / TiKV-Details / TiKV-Trouble-Shooting;视图 → Top SQL / statement summary / slow query / Key Visualizer;日志 → TiKV 日志 / PD scheduler 日志 / TiDB slow log / Diag 包;处置 → 对热点写限流或打散、检查统计信息与执行计划、评估 Region split / scatter、扩容或调后台 I/O，具体参数取值随负载与拓扑而定。

**端到端案例(OceanBase)。** 某 Tenant 在 major merge 窗口 P99 抖动,OCP 显示 Tenant CPU 或 I/O 抖动,SQL Audit 中同一类 SQL 的 `ELAPSED_TIME` / `QUEUE_TIME` / `TOTAL_WAIT_TIME_MICRO` 上升,`SHOW TRACE` 显示某阶段耗时异常,Tablet compaction 视图显示相关 LS / Tablet 正在推进 compaction。链路:指标 → OCP / OCP-Agent Prometheus 数据;视图 → `GV$OB_SQL_AUDIT` / `SHOW TRACE` / Tenant 资源视图 / LS / Tablet / compaction 视图;日志 → observer 日志 / obproxy 日志 / obdiag gather 包;处置 → 调 merge 窗口或资源、隔离热点 Tenant、处理慢 SQL / 锁等待、检查 ODP 路由，具体阈值随读写比与租户规格而定。

**关键压测指标。** P50 / P95 / P99 / P999 延迟、QPS、错误率、重试率、tso-cmd / tso-request 比值(TiDB TSO batch 效率)、write stall duration、compaction pending bytes、memstore 占用百分比、锁等待时间、队列时间、cache hit、后台任务积压。

**观测断言与误报 / 漏报回归。** 好的测试要包含"观测断言":TiDB 热点写入用例不仅断言 P99 上升，还断言 Key Visualizer 出现局部亮带、PD hot region 或 Store 维度倾斜、Top SQL / statement summary 能看到同一 SQL 类别、TiKV 日志或 RocksDB 指标有相邻时间证据;OceanBase merge 叠加用例不仅断言 P99 上升，还断言 Tenant 维度资源变化、SQL Audit 等待或队列字段变化、trace id 可追到阶段耗时、Tablet / LS compaction 视图能落到对象。回归测试还要覆盖误报与漏报:误报包括业务流量自然上升而内部状态健康、客户端连接池配置错误被误判为 SQL 慢、Prometheus 抓取延迟造成面板断点、单次慢查询被放大成系统性故障;漏报包括短时 P999 抖动被 1 分钟平均值抹平、单 Tenant 或单 Region 热点被集群级 CPU 掩盖、SQL Audit / Trace 未开启导致请求级证据缺失。把这些反例写进测试，运维手册才不会把"能看到指标"误当成"能解释故障"。

**关键观测指标 / 视图 / 日志字段 / 诊断命令(均已查证，未查证者标注)。**
- TiDB metric:`Write stall duration`(应为 0)、`Compaction pending bytes`、gRPC message duration P99、raftstore CPU、coprocessor `Wait duration`(P99.99 应 <10s)、scheduler command / latch wait duration(应 <1s)([Key Monitoring Metrics of TiKV](https://docs.pingcap.com/tidb/stable/grafana-tikv-dashboard/));源码确认的 RocksDB 内核属性名与导出的 Prometheus metric 字符串(`tikv_engine_pending_compaction_bytes` / `tikv_engine_stall_micro_seconds` / `tikv_engine_num_files_at_level` / `tikv_engine_cache_efficiency` / `tikv_engine_wal_file_synced` 等,`rocks_metrics.rs` @ `1f8a140b6d46`)见 §19.2。
- TiDB 视图 / 表:`INFORMATION_SCHEMA.SLOW_QUERY`、`STATEMENTS_SUMMARY`、`DATA_LOCK_WAITS`、`DEADLOCKS`、`CLUSTER_TIDB_TRX`、`mysql.tidb`(`tikv_gc_safe_point`)。
- OceanBase 视图:`GV$OB_SQL_AUDIT`(列 `RETRY_CNT` / `ELAPSED_TIME` / `QUEUE_TIME` / `EXECUTE_TIME` / `TOTAL_WAIT_TIME_MICRO` / `RPC_COUNT`)、`GV$OB_COMPACTION_PROGRESS`(`STATUS` / `UNFINISHED_TABLET_COUNT`)、`DBA_OB_MAJOR_COMPACTION`、`GV$OB_TENANT_MEMORY`、`GV$ACTIVE_SESSION_HISTORY`、`GV$OB_TRANSACTION_PARTICIPANTS`。
- 诊断命令:TiDB `ADMIN SHOW SLOW TOP N`、`pd-ctl hot write`、PingCAP Clinic `diag collect`;OceanBase `SHOW TRACE`、`obdiag gather sql_audit` / `obdiag gather ash` / `obdiag rca run --scene=major_hold` / `obdiag rca run --scene=lock_conflict`。
- 日志字段:TiDB slow log `Total_keys` / `Process_keys` / `Wait_time`;OceanBase `obproxy_diagnosis.log` 的 `trace_type`(CLIENT_VC_TRACE 等)。

## 19.10 容易误解点

**误解一:"OceanBase 没有外部监控就看不了性能,TiDB 必须装 Grafana"。** 纠正:恰好相反——OceanBase 的核心 SQL 诊断数据(SQL Audit、ASH、compaction 进度)都内置在 gv$ / v$ 虚拟表里，裸装 OceanBase 用一条 SQL 即可诊断;反而是 TiDB 的全集群关联分析与可视化强依赖 Prometheus + Grafana + Dashboard 这套外部栈。OBAgent + Prometheus 在 OceanBase 这边主要服务于时序图表与长期趋势，而非"否则看不到"。但"有 Prometheus"也不等于"可观测":没有 SQL digest、trace id、Region / LS / Tablet 维度与日志时间线,P99 抖动仍然无法归因。

**误解二:"慢 SQL 阈值越低，问题暴露越全"。** 纠正:TiDB `tidb_slow_log_threshold` 默认 300ms 是经过权衡的——阈值过低会让慢日志被海量正常 SQL 淹没、写日志本身成为负担。OceanBase SQL Audit 走的是另一条路:全量审计但内存有界 FIFO 淘汰(高水位 90% 淘汰至低水位 80%,见 §19.3)。前者靠阈值过滤、后者靠内存淘汰，两种"取舍"都意味着你看到的不是全集，不能假设慢日志 / 审计里就是全部慢 SQL。此外 Top SQL 不等同于慢日志:前者关联 SQL / plan digest 与 CPU load,后者记录超过阈值或符合条件的查询;SQL Audit 也并非"一定能回答所有慢 SQL 问题",仍需 Trace、日志、OCP 指标与 obdiag 联动。

**误解三:"P99 抖动看 CPU 火焰图就够了"。** 纠正:Top SQL / Continuous Profiling / ASH 解决的是 CPU 归因类问题;但 P99 抖动同样可能来自 write stall、TSO 等待、锁冲突、compaction 卡顿、网络——这些在火焰图里可能完全看不出来。TiDB 官方明确 Top SQL"无法定位非 CPU 高负载导致的性能问题，如事务锁冲突"。正确做法是先用 database-time(TiDB)或 sql_audit 时间分解 / SHOW TRACE(OceanBase)定位"时间花在哪个阶段",再决定是否下钻火焰图。把火焰图当作"第一现场"而非"第二现场"是常见误区。

**误解四:"两套系统的'单点'风险都体现在观测控制面"。** 纠正:运维者常把 PD / RootService 简单理解为"挂了就全瞎"的单点，这不准确。对观测控制面必须按五维度区分(详见第 13 章):逻辑中心化、性能瓶颈、高可用单点、元数据依赖、故障恢复依赖。PD 是 etcd / Raft 对称多副本,RootService 随核心表 Leader 由 Paxos 重建，二者都是逻辑中心化但物理高可用;它们短暂不可用时，热点调度、Region / Tablet 均衡、自动 analyze 等"后台运维动作"会暂停，但已建立的读写路径(Leader 仍在、客户端缓存仍有效)通常不立即中断。把"调度面短暂不可用"误读为"集群读写全停",会导致故障定级与处置方向都错。同理,GTS / SCN、PALF 日志复制、compaction 也都要按角色拆开:它们可能影响恢复、调度或事务时间线，但在没有指标与日志证据时不能被写成单一根因。

**误解五:"后台任务都是坏事""一次恢复就证明根因成立""指标名写得越细越专业"。** 纠正:RocksDB compaction、major / minor merge、Tablet compaction、GC / safepoint 都是系统维持长期健康所必需，问题通常在于前台负载、后台资源、窗口与热点对象叠加，而非后台任务本身。恢复可能来自流量自然回落、缓存预热、后台任务结束或重试成功，只有处置时间与指标、视图、日志共同闭环才提高因果可信度。指标名也不是越细越好——没有官方文档或锁定源码支撑的 metric、参数、函数与视图名会降低可信度，无法查证时应明确写"需进一步查证"。

## 19.11 本章结论

1. TiDB 的可观测性是外置组件拼装式:Statement Summary(LRU / 按 digest 聚合,release-8.5 源码确认)、Slow Query(默认 300ms 阈值)、Top SQL(CPU 按 SQL 聚合)、Continuous Profiling 在进程内采集，但全集群关联与可视化经 Prometheus + Grafana + Dashboard 完成;database-time 三维分解是其 P99 定位主线。
2. OceanBase 的可观测性是内核内置 gv$ / v$ 虚拟表式:SQL Audit(`GV$OB_SQL_AUDIT`=21014,内存环形缓冲 FIFO 淘汰)、SQL Trace、compaction 进度(`GV$OB_COMPACTION_PROGRESS`=21227)、LS / Tablet / 租户内存视图、ASH 直接暴露内核状态,SQL 即诊断、零外部依赖，视图名与表 ID 均逐行确认于 v4.2.5_CE 源码且 4.4.x 稳定。
3. P99 抖动必须按"指标 → 视图 → 日志 → 处置"推进，并用关联键(TiDB 的 SQL digest / Region / Store、OceanBase 的 trace id / Tenant / LS / Tablet)把多面板与日志拼成同一事件;链路因架构而异:TiDB 经 write stall / 热点 Region / TSO wait 入手(TiKV-FastTune、Key Visualizer),OceanBase 经 sql_audit RT 列 + compaction 进度 + memstore 阈值入手(obdiag major_hold RCA),印证第 4 章"TiKV compaction 本地自治 vs OceanBase major freeze 全局协调"的结论。
4. 两套体系把可观测性放在不同层:TiDB 的代价是依赖外部监控栈 + Statement Summary / Top SQL 内存有界 + GC safepoint 与长事务耦合;OceanBase 的代价是内存视图保留窗口短(SQL Audit 高水位 90% 淘汰至 80%,源码逐字确认)+ 多视图组合判断 + major compaction 抖动固有。Top SQL 对 CPU 型问题有效但不能替代锁冲突 / 网络 / GC / RocksDB stall 的证据链;SQL Audit / Trace 对请求阶段归因有效但不能替代 OCP 指标与日志。
5. 诊断工具上,PingCAP Clinic / Diag(采集 + 健康报告)与 obdiag(gather / analyze / check / rca,支持 lock_conflict、major_hold 等 RCA 场景)均为独立仓库工具，提供半自动根因分析，但都不在核心数据库 checkout 内，本章对其只做角色断言不做源码级断言。
6. PD、RootService、GTS / SCN 等控制面只能按逻辑中心化、性能瓶颈、高可用单点、元数据依赖、故障恢复依赖分别分析，不能简单写"单点";未查证的 Prometheus metric、内部视图、参数与函数名不写死，统一标"需进一步查证"。
7. （推测）观测体系的选择不是先进性排序而是工程取舍:延迟敏感 + 需长期趋势审计偏向 TiDB 外置时序库;零外部依赖 + SQL 即诊断 + Oracle 运维习惯偏向 OceanBase 内核视图。
8. TiKV RocksDB 导出的 Prometheus metric 字符串(`tikv_engine_pending_compaction_bytes` / `tikv_engine_stall_micro_seconds` 等,`rocks_metrics.rs` @ `1f8a140b6d46`)与 TiDB Top SQL 默认采样精度 1 秒 / 上报间隔 60 秒(`pkg/util/topsql/state/state.go` @ `67b4876bd57b`)已逐字核对;Continuous Profiling 采集间隔(独立 tidb-dashboard 仓库)与 TSO batch 默认上报间隔数值仍未逐条核对,标注"需进一步查证",未编造。

## 19.12 参考文献

[1] Performance Analysis and Tuning(TiDB Docs). 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/performance-tuning-methods/.
 （支撑:§19.2 / §19.7 database time 三维分解方法论与 P99 自顶向下定位主线。）
[2] Identify Slow Queries(TiDB Docs). 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/identify-slow-queries/.
 （支撑:§19.2 慢查询默认 300ms 阈值、tidb_slow_log_threshold、慢日志字段与 SLOW_QUERY 表(与源码 DefaultSlowThreshold=300 反查证一致)。）
[3] TiDB Dashboard Top SQL Page(TiDB Docs). 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/top-sql/.
 （支撑:§19.2 / §19.10 Top SQL 的能力边界、CPU 归因与不适用场景(锁冲突等非 CPU 主因问题)。）
[4] Key Monitoring Metrics of TiKV(TiDB Docs). 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/grafana-tikv-dashboard/.
 （支撑:§19.6 / §19.9 write stall、compaction pending bytes、gRPC / coprocessor / scheduler 阈值等故障定位指标。）
[5] TiKV Configuration File(TiDB Docs). 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/tikv-configuration-file/.
 （支撑:§19.2 region-split-size 默认值版本差异:v8.5.0 默认 256MiB、"Before v8.4.0 默认 96MiB",与 release-7.5 / release-8.5 源码逐分支核）
[6] SQL Trace(OceanBase Docs). 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001106685.
 （支撑:§19.3 / §19.9 SHOW TRACE、ob_enable_show_trace 与 SQL 阶段耗时分析，以及"SQL Audit → trace id → SHOW TRACE / 日志"关联路径。）
[7] OceanBase Diagnostic Tool(obdiag,官方文档 + 工具仓库). 官方文档 / 工具[EB/OL]. https://en.oceanbase.com/docs/obdiag-en.
 （支撑:§19.3 / §19.6 / §19.9 obdiag check / gather / analyze / rca / display 角色、采集对象与 lock_conflict / major_hold RCA）
[8] GV$ACTIVE_SESSION_HISTORY(OceanBase Docs V4.2.0). 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001030063.
 （支撑:§19.3 ASH 视图与 4.x 采样机制。该 doc-id 页本轮被 WAF 限流，正文"约 1 秒"采样间隔另由 commit-pinned 源码 src/share/ash/ob_active_sess_his）
[9] pingcap/tidb · tikv/tikv · pingcap/pd(release-8.5). 源码[EB/OL]. https://github.com/pingcap/tidb/tree/release-8.5/pkg/util/stmtsummary.
 （支撑:§19.2 Statement Summary(statement_summary.go,LRU / StmtDigestKey)、Top SQL(topsql.go,默认采样精度 / 上报间隔常量 DefTi）
[10] oceanbase/oceanbase(v4.2.5_CE @ e7c676806fda;4.4.x @ d4bef8d29a4c). 源码[EB/OL]. https://github.com/oceanbase/oceanbase.
 （支撑:§19.3 GV$OB_SQL_AUDIT(ob_gv_sql_audit.h,表 ID 21014)、SQL Audit 高 / 低淘汰水位 HIGH_LEVEL_EVICT_PERCENTAGE=0.9 /）
[11] Use OCP-Agent to pull time-series monitoring data(OceanBase Docs). 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-ocp-10000000001554343.
 （支撑:§19.3 OCP-Agent / ocp_monagent、Prometheus 格式监控、SQL Audit 数据与日志采集边界。）
[12] PALF: Replicated Write-Ahead Logging for Distributed Databases(VLDB 2024). 论文[EB/OL]. https://www.vldb.org/pvldb/vol17/p3745-xu.pdf.
 （支撑:§19.3 / §19.7 OceanBase LS / PALF 复制式 WAL(Paxos-backed append-only log)对故障恢复与运维观测的底层背景。）
[13] OceanBase: A 707 Million tpmC Distributed Relational Database System(VLDB 2022). 论文[EB/OL]. https://dl.acm.org/doi/10.14778/3554821.3554830.
 （支撑:§19.3 多租户 Resource Unit(CPU / 内存 / IOPS / 磁盘)、前后台线程池 / 内存隔离与 LSM-tree + major / minor compaction 架构基础;开放获取版另见）
[14] Diagnosis and Tuning with OceanBase: sql_audit / Compaction 诊断博客. 官方博客[EB/OL]. https://en.oceanbase.com/blog/12526873344.
 （支撑:§19.3 / §19.6 / §19.7 SQL Audit 高 / 低水位淘汰与 RETRY_CNT 等诊断列语义、GV$OB_COMPACTION_PROGRESS 的 STATUS=NODE_RUNNI）
[15] Manage memory in OceanBase Database. 第三方解读 / OceanBase Medium 转载[EB/OL]. https://oceanbase.medium.com/manage-memory-in-oceanbase-database-2e0f79aade73.
 （支撑:§19.3 / §19.7 OLTP 场景 freeze_trigger_percentage 运维建议值(60–75)与 memstore / minor freeze 抖动说明;注明此为运维建议而非源码默认值。）
## 19.13 信息可信度自评

本章可信度分层如下。源码级确凿(均 commit-pinned 到统一基线 release-8.5 三仓与 v4.2.5_CE,4.4.x 交叉确认稳定):TiDB Statement Summary LRU 结构与默认值、Slow Query 300ms 阈值常量、Top SQL `SetupTopSQL` / `AttachAndRegisterSQLInfo` 及其默认采样精度 1 秒 / 上报间隔 60 秒(`DefTiDBTopSQLPrecisionSeconds=1` / `DefTiDBTopSQLReportIntervalSeconds=60`)、RocksDB 内核属性名与导出的 Prometheus metric 字符串(`tikv_engine_pending_compaction_bytes` / `tikv_engine_stall_micro_seconds` 等,`rocks_metrics.rs`)、GC worker 常量、split-checker `SPLIT_SIZE`(v8.5.0=256MiB,7.5.x=96MiB 版本相关)、`RAFTSTORE_V2_SPLIT_SIZE=10GB`、hot_region scheduler、PD Dashboard / Key Visualizer 绑定;OceanBase GV$OB_SQL_AUDIT / compaction / LS / 租户视图的视图名与表 ID、SQL Audit 高 / 低淘汰水位(0.9 / 0.8)、ASH 约 1 秒采样、`last_trace_id()`、`enable_sql_audit` / `freeze_trigger_percentage` / `writing_throttling_trigger_percentage` 默认值。官方文档明确:database-time 方法论、TiKV 监控阈值、SQL Trace / `SHOW TRACE`、obdiag 命令族、OCP-Agent 采集边界、Top SQL 能力边界。论文支撑:OceanBase 多租户 Resource Unit 与线程 / 内存隔离架构(VLDB 2022)、PALF 复制式 WAL(VLDB 2024,属协议理论)。工程推测:两套体系的运维代价对比与场景适配建议(§19.8),不作官方背书。仍不确定 / 未逐条核对:TiDB Continuous Profiling 采集间隔(由独立 tidb-dashboard 仓库管理,不在本批 checkout 内)、TSO batch 默认上报间隔数值——均显式标注"需进一步查证",未编造(TiKV RocksDB 导出的 Prometheus metric 字符串与 Top SQL 采样精度 / 上报间隔已于本轮源码逐字升级为事实)。版本核查:OceanBase SQL Audit 视图自 4.0 起为 `GV$OB_SQL_AUDIT`,本章按锁定基线 4.2.5_CE 书写，与官方文档一致;唯一版本敏感量 `region-split-size` 已显式区分 7.5.x(96MiB)与 v8.5.0(256MiB),不以单一数值覆盖两版本。

---


# 第 20 章 兼容性模型

## 20.1 本章核心问题

![[f20_1.svg]]

**图 20-1　兼容性五层光谱分层栈：TiDB 与 OceanBase 各层落点对照**

"兼容 MySQL"或"兼容 Oracle"在分布式数据库的宣传页上是一句轻巧的承诺，在工程上却对应着一组高昂的代价。本章要回答的根本问题不是"能不能用 MySQL 客户端连上"这么单一，而是：**当一个 SQL 生态被搬到分布式内核上重新解释时，哪些语法、类型、事务语义、错误行为、DDL 行为和诊断接口仍能保持预期，哪些必须在迁移时显式改造?** 一句"兼容 MySQL 5.7"背后，至少要拆成五层光谱来核验：

1. **协议层**：能不能用 MySQL 客户端/驱动直接连?握手包、认证、命令字、结果集编码是否一致?
2. **方言层(SQL grammar)**：`CREATE PROCEDURE`、`CREATE TRIGGER`、`SEQUENCE`、外键、`JSON` 函数能不能被解析成 AST?
3. **执行层(planner/executor)**：解析出来的东西能不能真的执行?这是最容易被混淆的一层——**有语法 ≠ 能执行**。一个 `CREATE PROCEDURE` 可能解析通过，却在执行器里根本没有接入，实际并不可用。
4. **语义层**：`sql_mode`、隔离级别字符串的实际语义、非法日期处理、`NULL` 排序、大小写折叠、自增 ID 形态、错误码——这些"细到字节"的语义差异，才是迁移真正翻车的地方。
5. **发布/授权层**：某个兼容模式是开源社区版就有，还是企业版/Cloud 专属?这一层既不在协议里也不在源码可见性里，而在商业边界上。

本章用两条对照路线拆解这五层：

- **TiDB**：选择**单一方言深度兼容**——只兼容 MySQL，把 MySQL 协议与方言做深，但明确放弃存储过程、触发器、事件、UDF 等一批 MySQL 服务端编程能力(官方文档逐字列为"不支持")。它的兼容是"窄而真"：支持的部分尽量真能执行，不支持的部分干脆不做语法占位(触发器连语法都没有)。外键则是一个演进样本——长期"有语法但谨慎"，直到 v8.5.0 才正式 GA。
- **OceanBase**：选择**双方言、模式化兼容**——租户(Tenant)级在 **MySQL 模式**与 **Oracle 模式**之间二选一(创建时确定、之后不可改)，内核共享一套从 resolver 到执行的双模式分叉逻辑，并自带 PL 引擎、SEQUENCE、SYNONYM、PACKAGE、TRIGGER 等 Oracle 风格对象。它的兼容是"宽而分层"：社区版只给 MySQL 模式的语法前端，Oracle 模式的语法前端属企业版/Cloud，但 Oracle 语义分支在共享内核里到处都是。

理解这条分界线，关键是把"兼容"从一个布尔标签拆成五层光谱，并始终追问：**这一条是解析层、执行层、还是发布层?** 本章与前文各章互为参照：第 10/11 章已说明事务协议不能只看 SQL 表面，隔离级别的"名义"与"实际"可能分离；第 12 章已说明优化器差异会影响同一 SQL 的计划与性能；第 14 章已说明 Online DDL 的状态推进会改变 DDL 兼容性边界；第 21 章讨论安全/权限边界。本章把这些结论收束到迁移与测试问题上：可以推测，**兼容性不是一张 SQL parser 检查表，而是 parser、resolver、catalog、optimizer、事务、锁、DDL、错误码与工具链共同给出的一份行为合同。** 涉及其他章节处仅标"详见第 X 章"，不展开。

> 本章版本基准遵循公共头锁定：TiDB 8.5.x LTS(对照 7.5.x)；OceanBase v4.2.5_CE(TP)/4.3.5(AP)/4.4.x(融合)。源码引用 commit-pin 到统一基线 TiDB release-8.5 @ `67b4876bd57b`、OceanBase v4.2.5_CE @ `e7c676806fda`、OceanBase 4.4.x @ `d4bef8d29a4c`。本章兼容性集中在 SQL 前端/解析/语义层，故未使用 tikv/pd 的 checkout。

--- 〔文献[1-2]〕

## 20.2 TiDB 的实现

### 兼容定位：MySQL 协议 + MySQL 方言，单一方言

TiDB 的兼容模型只有一个目标：**MySQL**。它没有"Oracle 模式"概念。VLDB 2020 论文《TiDB: A Raft-based HTAP Database》开篇即言："TiDB supports the MySQL protocol and is accessible by MySQL-compatible clients"(逐字)，架构上 SQL 引擎层与分布式存储层分离。(论文，PDF 直接抽取确认)这意味着兼容的第一层(协议)是设计起点：MySQL 客户端、JDBC/驱动可直连。

但协议兼容只是入口。一条 SQL 进入 TiDB Server 后，要经 parser/AST、name resolution、privilege check、optimizer、executor，再落到 TiKV/TiFlash 或 TiDB 内部 DDL/元数据路径。于是兼容的第二层是语义与执行行为，第三层才是存储与分布式事务能否复刻 MySQL 单机行为。TiDB 官方《MySQL Compatibility》文档声明高度兼容 MySQL 5.7 与 8.0 的常用协议、语法与生态工具，但**逐字列出一批不支持的特性**："Stored procedures and functions, Triggers, Events, User-defined functions"——即存储过程与函数、触发器、事件、用户自定义函数均**不支持**。(官方文档，WebFetch 逐字确认)文档解释原因为"有更好的解决方式 / 当前需求与投入不匹配 / 某些特性在分布式系统中难以实现"。

这定义了 TiDB 兼容的边界哲学：**窄而真**。支持的部分(DML、绝大多数 DDL、JSON、窗口函数、CTE、外键等)力求真能执行；放弃的部分明确放弃，不做"能解析不能跑"的语法占位——下文会看到触发器连语法 token 都不构造 DDL 产生式。

### SQL 方言基线：默认 sql_mode 与隔离级别常量

TiDB 的 MySQL 方言基线由解析器常量锁定。默认 `sql_mode`(`pkg/parser/mysql/const.go` L309–310，release-8.5 @ `67b4876bd57b`)：

```text
DefaultSQLMode = "ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,
 NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,
 NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION"
```

这与官方《MySQL Compatibility》文档逐字一致，文档并注明：该默认值与 MySQL 5.7 默认对齐，但与 MySQL 8.0 略有差异(8.0 默认不含 `NO_AUTO_CREATE_USER`)。(官方文档+源码，双向确认)同文件还定义 `ModeANSI`/`ModeOracle`/`ModeANSIQuotes`/`ModeRealAsFloat`/`ModeStrictTransTables` 等 SQLMode 位(L453 起)及 `ANSI`/`TRADITIONAL` 组合映射(L570/L578)——这里务必区分：**`ModeOracle` 是 MySQL 自身 `sql_mode` 体系里的一个兼容位，不是"TiDB 有 Oracle 模式"**，二者必须分开理解(见 §20.10 误解点)。

隔离级别字符串常量(`pkg/parser/ast/misc.go` L71–74)：`ReadCommitted = "READ-COMMITTED"`、`ReadUncommitted = "READ-UNCOMMITTED"`、`Serializable = "SERIALIZABLE"`、`RepeatableRead = "REPEATABLE-READ"`——四个标准 MySQL 隔离级别字符串。但**字符串对齐不等于语义对齐**：TiDB 在 `REPEATABLE-READ` 名义下实现的是 Snapshot Isolation，与 ANSI Repeatable Read 和 MySQL InnoDB Repeatable Read 都存在差异，尤其在 optimistic transaction 下，更新可见性检查与失败重试路径与 InnoDB 当前读、gap lock、semi-consistent read 不同。(官方文档《Transaction Isolation Levels》)因此迁移 MySQL 应用时，不能只看 `transaction_isolation` 变量的显示值，必须用并发读写、`SELECT ... FOR UPDATE`、唯一键冲突、外键级联与 DDL 并发来验证真实行为。事务语义本身详见第 10/11 章，本章只锁定"方言字符串与 MySQL 对齐、但语义不可由变量名推定"这一兼容事实。

### 外键：从"有语法"到 v8.5.0 GA(真有内核实现)

外键是 TiDB 兼容深度演进的典型样本。两个开关默认都开启：

- `tidb_enable_foreign_key`：全局变量，默认 **ON**(`pkg/sessionctx/variable/sysvar.go` 约 L1798，`Value: BoolToOnOff(true)`；包级 `EnableForeignKey = atomic.NewBool(true)`，`tidb_vars.go` L1772)。
- `foreign_key_checks`：`ScopeGlobal|ScopeSession` 的 `TypeBool`，默认 ON(`DefTiDBForeignKeyChecks = true`，`tidb_vars.go` L1597;sysvar 表 `sysvar.go` L1784)。官方文档亦逐字确认 `foreign_key_checks` "is set to `ON`" 默认开。(官方文档+源码，双向确认)

关键是外键**不是纯语法占位**：`pkg/ddl/foreign_key.go`(release-8.5 @ `67b4876bd57b`)有完整的执行/校验函数链——`onCreateForeignKey` 把外键 DDL 推过 `StateNone → StateWriteOnly → StateWriteReorganization → StatePublic`(与第 14 章 DDL 状态机一致)，并在中间阶段进行约束检查；还有 `checkTableForeignKeysValid`、`checkTableForeignKey`、`checkModifyColumnWithForeignKeyConstraint`、`checkIndexNeededInForeignKey`、`checkDropColumnWithForeignKeyConstraint`、`checkTableHasForeignKeyReferred` 等函数。这说明 8.5 外键已进入 DDL 状态机与引用完整性校验路径，不再只是 mysqldump 友好的语法保留。

"GA"这一发布层定性，源码本身不含 "GA" 字样，但**官方文档已坐实**：TiDB 自 v6.6.0 起支持外键，**自 v8.5.0 起该特性正式 GA**(《Foreign Key》文档与《8.5.0 Release Notes》均逐字载 "Support foreign keys (GA) ... becomes generally available (GA) in v8.5.0")。(官方文档，WebFetch 逐字确认)官方外键文档同时提醒：外键可能带来性能退化，需在性能敏感场景充分测试(见 §20.7)。这是少数"语法→执行→GA"三层都齐全的兼容样本。

### 自增 ID：兼容性与分布式架构的交叉点

自增 ID 是"表面兼容、语义偏移"的典型例子。TiDB 默认通过缓存分配 `AUTO_INCREMENT`，多个 TiDB 实例各自持有一段 ID 缓存，因此可能产生较大间隙；文档提供 `AUTO_ID_CACHE 1` 的 MySQL-compatible 模式，使 ID 在多个 TiDB 实例间保持严格递增且间隙较小，但 failover、滚动升级、并发事务仍可能产生小间隙。(官方文档)如果业务把自增 ID 的"连续无洞"当成业务语义(如审计连号、回放顺序)，而不是仅作唯一标识，就需要在迁移前修正模型；反过来，若业务追求写入扩展性，则可能(此处为推测)主动选择 `AUTO_RANDOM` 等 TiDB 侧能力，接受与 MySQL 表面行为不同的 ID 形态。

### 存储过程：有语法、无执行(能解析不能跑)

存储过程是"有语法 ≠ 能执行"的典型例证。语法层：`pkg/parser/parser.y`(`CreateProcedureStmt:` 约 L16496)能解析 `CREATE PROCEDURE` / `DROP PROCEDURE` 并构造 `ast.ProcedureInfo`；`pkg/parser/ast/procedure.go` 定义了 `ProcedureInfo`、`ProcedureBlock`、`DropProcedureStmt`、`ProcedureIfBlock` 等 AST 节点。

但执行层**完全缺位**：全仓 `rg "ProcedureInfo"` 在 `pkg/executor`、`pkg/planner`、`pkg/session` 下**无任何引用**(仅 parser 自身命中)；旁证是 `pkg/executor/show.go` 中 `fetchShowProcedureStatus()` 为空返回，`pkg/parser/parser_test.go` L1241 注释 "PROCEDURE and FUNCTION are currently not supported."。这与 §20.2 开头官方文档"不支持存储过程与函数"互相印证——**release-8.5 中存储过程仅停留在解析层，无执行计划/执行器接入，不可实际运行**。所以这条不是"TiDB 支持存储过程"，而是"TiDB 能解析存储过程语法但不能执行它"。这一源码事实容易被误读为"TiDB 其实支持存储过程"；本章按官方对外文档定性为不支持，源码痕迹只作"解析层痕迹不等于支持"的反例，相应判断带有推测成分。

### 触发器：连语法都没有(比存储过程更彻底地不做)

触发器比存储过程更彻底地未做实现——**连 DDL 语法产生式都不存在**。`TRIGGER`/`TRIGGERS` 是词法 token(`pkg/parser/misc.go` L860–861)，但在 grammar 中只用于两处：(1) 权限 `TriggerPriv`(`parser.y` 约 L14643–14645，`$$ = mysql.TriggerPriv`);(2) `SHOW TRIGGERS`(`parser.y` 约 L11978–11981，`Tp: ast.ShowTriggers`，且 `pkg/executor/show.go` 的 `fetchShowTriggers()` 同为空返回)。**不存在 `CreateTriggerStmt` / `CREATE TRIGGER` 产生式**(`rg -i "CreateTriggerStmt|create trigger"` 在 ast/parser.y 无命中)。

即 TiDB 8.5 对触发器是"权限位 + SHOW 语句"级别的形式存在，而无任何创建/执行能力——比存储过程(至少能解析)更彻底地不支持。提交 `CREATE TRIGGER` 会直接语法报错，而不是"解析通过但不执行"。这与官方文档把 "Triggers" 列入不支持清单一致。

### 兼容性的工程哲学：窄而真

综合上述，TiDB 的兼容模型可归纳为**单一方言(MySQL)、协议优先、执行优先**。支持的特性力求真能执行(外键 v8.5 GA、JSON、窗口函数、CTE)；不支持的服务端编程特性(存储过程/触发器/事件/UDF)要么只保留解析层(存储过程)、要么连语法都不做(触发器)，且官方文档明确列出不支持清单，不以语法占位充当兼容。代价是：依赖存储过程/触发器的 MySQL 应用无法平迁，这部分逻辑必须上移到应用层。此处对迁移代价的判断为推测；"窄而真"亦属本章归纳措辞，由源码与官方文档共同支撑。

控制路径上，普通 DML 进入优化器与执行器、读写 TiKV Region；外键 DML 需补充父/子表约束检查与级联动作；DDL 则进入 DDL owner/元数据状态机，再经 schema version 推送到各 TiDB Server。其典型故障路径集中在：schema 版本滞后、事务冲突、外键级联导致事务膨胀、以及 TiCDC/BR/Lightning 与外键或 partition DDL 的组合限制(详见 §20.6/§20.7)。

--- 〔文献[3-6,11]〕

## 20.3 OceanBase 的实现

### 兼容定位：租户级 MySQL/Oracle 双模式

与 TiDB 的单一方言相对，OceanBase 的兼容模型核心是**模式化(mode)**：兼容性绑定在**租户(Tenant)**这一级，每个租户在 **MySQL 模式**与 **Oracle 模式**之间二选一，创建时确定、之后不可更改，所选模式影响数据类型、SQL 函数与内部视图。源码层面，`enum class CompatMode {INVALID = -1, MYSQL, ORACLE};`(`deps/oblib/src/lib/worker.h` L33，class `Worker` 内，v4.2.5_CE @ `e7c676806fda`，MYSQL=0、ORACLE=1)——**只有这两个有效值**。

模式取数与系统视图映射：`ObCompatModeGetter`(`src/share/ob_get_compat_mode.cpp`/`.h`)按 `tenant_id` 解析租户兼容模式；内表视图(`src/share/inner_table/ob_inner_table_schema.21151_21200.cpp`)以 `CASE compatibility_mode WHEN 0 THEN 'MYSQL' WHEN 1 THEN 'ORACLE' ELSE NULL END` 把模式暴露给用户(view 定义文本中 0→'MYSQL'、1→'ORACLE')。

模式的发布/授权层归属由官方文档承载：Community Edition 官方文档写明仅提供 MySQL 模式，Oracle 模式需按企业版/Cloud 等产品形态核查。(官方文档《Compatibility modes》明确 "Community Edition provides only the MySQL mode"；模式创建后不可改亦以官方文档为准)精确表述以官方文档逐字为准，并由下文源码可见性旁证。

### 关键事实：CE 源码不含 Oracle 模式语法前端

"OceanBase 兼容 Oracle"这一表述容易掩盖一个事实：**社区版 checkout 的 SQL 解析器目录里只有 MySQL 模式的词法/语法文件**。`src/sql/parser/` 下仅有 `sql_parser_mysql_mode.l`、`sql_parser_mysql_mode.y`、`non_reserved_keywords_mysql_mode.c`；`find src/sql -iname "*oracle_mode*"` 在 **v4.2.5_CE 与 4.4.x 两个 checkout 均无命中**(无 `*oracle_mode*` 词法/语法文件)。(源码确认到目录级)

这与"Oracle 兼容模式为企业版/Cloud 特性"的发布层结论相互印证，**但必须严格限定**：源码只能证明"CE 不含 Oracle 模式 SQL 语法前端文件"，不能证明"内核完全无 Oracle 能力"。发布归属(谁能用 Oracle 模式)是商业/授权层结论，需官方文档佐证，不能仅凭源码可见性下断言。

### Oracle 语义分支遍布共享 resolver/执行层

与上一节配对的另一半事实：即便 CE 缺 Oracle 语法前端，**Oracle 语义处理路径仍内建于内核共享层**。`rg -c "is_oracle_mode" src/sql/resolver`(v4.2.5_CE @ `e7c676806fda`)命中分布在约 **799** 个文件(文件计数)。(源码确认到文件计数级)更底层的证据是：`src/sql/session/ob_basic_session_info.h` 中 `set_compatibility_mode()` 把 `ObCompatibilityMode` 写入 SQL mode，`get_compatibility_mode()` 从 SQL mode 还原兼容模式，注释提示 session 保留 compatibility mode 用于传递，其他需要 mode 的地方尽量使用线程上的 `is_oracle|mysql_mode` 并用检查函数保持一致。

这揭示了 OceanBase 兼容性的本质：**它不是"换个 parser"那么简单，而是从 session/thread 上下文到 resolver 再到执行的全链路双模式分叉**。同一段语义处理代码，根据当前租户是 MySQL 还是 Oracle 模式走不同分支——这意味着兼容性会渗透到类型推导、函数解析、视图、错误映射、PL 执行、对象管理与锁/DDL 的几乎每一处。两节必须配对理解：**CE 缺 Oracle 语法前端 ≠ 内核无 Oracle 能力**。

### MySQL 模式的兼容范围与"扩展"

在 MySQL 模式下，OceanBase 社区版官方文档称兼容 MySQL 5.7 大多数特性与语法、全部 MySQL 5.7 JSON 函数及部分 MySQL 8.0 JSON 函数，同时列出数据类型、SQL syntax、system views、charset/collation、functions、partition support、backup/recovery、storage engine、optimizer 等差异；该页还明确 CE 不支持 spatial data types。值得注意的是：partition 支持甚至有 OceanBase 自己的扩展，例如非模板化 subpartitioning——这让"兼容 MySQL"同时包含"支持 MySQL 常用语义"与"提供 MySQL 没有的扩展"两层含义。后者会在双向迁移时反过来降低可迁移性(见 §20.8)。

### Oracle 风格对象：SEQUENCE / SYNONYM / PACKAGE / TRIGGER 都有内核 resolver

OceanBase 内核含一批 TiDB 完全没有的对象类型，均有真实 DDL resolver/stmt(v4.2.5_CE @ `e7c676806fda`)。(源码确认到文件级)

- **TRIGGER**：`src/sql/resolver/ddl/ob_trigger_resolver.cpp`/`.h`(含 simple/instead/compound DML trigger resolver 接口)、`ob_trigger_stmt.cpp`/`.h`——完整触发器 DDL 解析/语句对象，**与 TiDB(无 CREATE TRIGGER 语法)形成直接对比**。MySQL 模式 trigger 官方文档列出限制：只能建在 permanent table、trigger 中不能启动或结束事务、外键动作不能 fire trigger。
- **SEQUENCE**：`ob_create_sequence_resolver.cpp`、`ob_alter_sequence_resolver.cpp`、`ob_drop_sequence_resolver.cpp`、`ob_sequence_stmt.h`，DML 引用解析见 `src/sql/resolver/dml/ob_sequence_namespace_checker.cpp`——原生支持 SEQUENCE(create/alter/drop + DML 引用)，官方 SQL 文档说明 `CREATE SEQUENCE` 可创建独立 sequence，并通过 `CURRVAL`/`NEXTVAL` pseudocolumn 取值，sequence 可独立于表生成值且默认允许缓存与非严格顺序。
- **SYNONYM / PACKAGE**：`ob_create_synonym_resolver.cpp`、`ob_drop_synonym_resolver.h`、`ob_alter_package_resolver.h`、`ob_drop_package_resolver.cpp` 等——同义词、包对象在内核有 DDL resolver/stmt。

此外 `src/sql/resolver/ob_stmt_type.h` 定义了 routine、procedure call、sequence、trigger 等语句类型。这些对象的**用户可见暴露面**在 MySQL/Oracle 模式间有差异(需结合模式判别与 edition/编译宏边界)，源码确认到"对象级 resolver 存在"，其在各模式下的精确可用性需结合官方文档，**不就模式暴露面单独下源码断言**。

### PL 引擎：存储过程/包是可执行的内核能力

OceanBase 兼容性与 TiDB 最本质的差异在 **PL(过程化语言)引擎**：`src/pl/` 目录(v4.2.5_CE @ `e7c676806fda`)含独立的 PL 编译/代码生成/包管理/异常处理引擎——`ob_pl.cpp`、`ob_pl_compile.cpp`、`ob_pl_code_generator.cpp`、`ob_pl_package.cpp`、`ob_pl_package_manager.cpp`、`ob_pl_resolver.cpp`、`ob_pl_exception_handling.cpp` 等。(源码确认到文件级)ODC 文档亦描述 stored procedure 的创建、IN/OUT/INOUT 参数与 `CALL` 调用。

即：**OceanBase 的存储过程与包(package)是可执行的内核能力**(有编译、有代码生成、有包管理、有异常处理)，而非解析层占位。这与 TiDB §20.2"存储过程仅解析、不可执行"形成本质差异——一个有 PL 虚拟机，一个只有 PL 语法 AST。但需补一句形态边界：Oracle 模式下的 PL 完整能力主要面向企业版/Cloud，CE 仅 MySQL 模式，不能由源码 `src/pl/` 的存在直接反推"CE 能跑 Oracle PL"。

### 模式之外：`compatible` 版本参数与日期语义开关(易混淆点)

兼容性在 OceanBase 还有一个**极易与"兼容模式"混淆**的参数：`compatible`。它是**租户级数据格式版本参数，不是 MySQL/Oracle 兼容模式**。`src/share/parameter/ob_parameter_seed.ipp` L576(v4.2.5_CE @ `e7c676806fda`)：`DEF_VERSION(compatible, OB_TENANT_PARAMETER, "4.2.5.0", "compatible version for persisted data", ...)`。即 `compatible` 是"持久化数据兼容版本号"(本 checkout 默认 `4.2.5.0`)，用于跨版本升级的数据格式门控——与 `CompatMode`(MySQL/Oracle)是两个不同的概念。

同文件 L400 `_enable_mysql_compatible_dates`(默认 `"False"`)控制是否允许 MySQL 风格的非法日期。这体现兼容性细到"日期合法性语义"的开关级行为：即便在 MySQL 模式下，某些 MySQL 宽松语义(如非法日期)仍需显式开关才生效。这类语义层开关是迁移中容易出问题的地方，远比"支不支持某语法"隐蔽。

--- 〔文献[7-9,12]〕

## 20.4 核心差异对比

两侧的差异不在"谁兼容得多"，而在兼容的形状不同。下面三张表分别从"整体定位维度""有语法/能执行/发布归属三层""迁移来源到目标"三个角度并置对比；正文已在 §20.2/§20.3 给过各条的"为什么"，此处表格只作维度归集。

**表 20-1 主对比：TiDB 单方言深度兼容 vs OceanBase 双模式兼容**

| 维度 | TiDB | OceanBase | 影响 |
|---|---|---|---|
| 兼容方言 | 仅 MySQL(无 Oracle 模式)，对齐 MySQL 5.7/8.0 常用语法 | MySQL 模式 / Oracle 模式(租户级二选一；CE 仅 MySQL 模式 | OB 覆盖 Oracle 迁移，TiDB 不覆盖 |
| 兼容绑定层级 | 集群/会话(单方言，无模式概念) | 租户(`CompatMode` 创建时定、不可改) | OB 同集群可混跑两模式租户 |
| 协议 | MySQL 协议(论文逐字) | MySQL 协议；Oracle 模式另有 Oracle 协议面 | 二者 MySQL 侧均可直连 |
| 存储过程 | 仅解析、不可执行(executor 无接入) | PL 引擎可执行(编译/代码生成/包/异常) | OB 可平迁存过逻辑，TiDB 不行 |
| 触发器 | 无 CREATE 语法(仅权限位+SHOW) | 有完整 DDL resolver/stmt(MySQL 模式有事务/外键动作等限制) | OB 支持，TiDB 连语法都没有 |
| SEQUENCE/SYNONYM/PACKAGE | 不支持 | 内核有 resolver/stmt | OB 覆盖 Oracle 对象模型 |
| 外键 | v8.5.0 GA，进入 DDL 状态机+校验链，默认开 | 支持(按模式与文档验证)；迁移工具另有目标端 trigger/DDL 同步限制 | 二者均有真实外键实现 |
| 事务隔离 | `REPEATABLE-READ` 名义下为 Snapshot Isolation，与 MySQL RR 有差异 | 兼容模式影响 SQL 行为，底层事务详见第 10 章 | 迁移验收须用并发行为而非变量名判断 |
| 自增/sequence | `AUTO_INCREMENT` 有缓存间隙；`AUTO_ID_CACHE 1` 更接近 MySQL | MySQL 模式有自增；Oracle 模式有 sequence(可独立于表、默认非严格连续) | ID 设计影响热点、唯一性、回放与审计 |
| 双模式实现位置 | 不适用(单方言) | 共享 resolver 约 799 文件含 `is_oracle_mode` 分支 | OB 兼容是全链路分叉，代价大 |
| 语义开关 | sql_mode(MySQL 体系)等 | CompatMode + `_enable_mysql_compatible_dates` 等细粒度开关 | 二者都有"细到字节"的语义差异面 |

**表 20-2 "有语法 / 能执行 / 发布归属"三层对照**

| 特性 | 有语法(grammar) | 能执行(executor) | 发布/授权层 | 证据 |
|---|---|---|---|---|
| TiDB 存储过程 | 是(`ProcedureInfo`) | **否**(executor/planner 无引用) | 开源 | parser.y / 全仓 rg 无执行引用 + show.go 空返回 |
| TiDB 触发器 | **否**(无 CREATE 产生式) | 否 | 开源(仅权限位+SHOW) | parser.y 无 `CreateTriggerStmt` |
| TiDB 外键 | 是 | **是**(`ddl/foreign_key.go` 校验链) | v8.5.0 GA(开源) | 源码状态机/校验链 + 官方 Release Notes |
| OB 存储过程/包 | 是(PL) | **是**(`src/pl/` 引擎) | 模式/形态相关 | `ob_pl*.cpp` 编译/codegen |
| OB 触发器 | 是 | 是(有 resolver/stmt) | 模式/形态相关 | `ob_trigger_resolver.cpp` |
| OB Oracle 模式语法前端 | CE **无**(`*oracle_mode*` 文件缺失) | 语义分支在共享层(799 文件) | 企业版/Cloud(发布层，以官方文档为准) | `find` 无命中 + `is_oracle_mode` 计数 |

**表 20-3 迁移视角：从某来源迁到三个目标**

| 迁移来源 | 迁到 TiDB 8.5 | 迁到 OceanBase MySQL 模式 | 迁到 OceanBase Oracle 模式 |
|---|---|---|---|
| MySQL 普通 OLTP | 优先核查事务隔离、外键、AUTO_INCREMENT、partition DDL、不支持特性 | 优先核查函数/视图/partition/optimizer 差异与工具限制 | 一般不作为直接目标，除非业务本就要重写到 Oracle 方言 |
| MySQL 用 trigger/procedure | 需应用层重写或外移(官方不支持) | 可评估保留，但要测 trigger 限制、事务控制、返回结果、动态 SQL | 需方言转换，非低成本 MySQL 迁移路径 |
| Oracle 应用 | 非 Oracle 兼容目标，需重写 SQL/PL | MySQL 模式不承诺 Oracle PL 兼容 | 目标形态需企业版/Cloud 核查；重点测 PL/sequence/package/hint/view/错误码 |
| 依赖连续自增 ID | `AUTO_ID_CACHE 1` 可降间隙，但不承诺绝对无洞 | 需按自增实现测 failover 与并发 | sequence 默认允许缓存与非严格顺序，业务不能假设连续 |
| 强依赖 MySQL 运维工具 | 连接与常见 SQL 生态友好，但 TiDB-specific DDL/变量/诊断需适配 | MySQL 客户端可接入，但内部视图/备份恢复/optimizer 差异需适配 | 应配合 Oracle-compatible 工具链与 OceanBase 工具链 |

---

## 20.5 正常路径图

图 20-2 展示两侧"一条 SQL 进入后如何被兼容层处理"的正常路径并置对比。关键在于：兼容模式不是只在入口处识别 SQL 方言，它会一路渗透到事务、外键、自增 ID、对象命名、错误映射与工具链。

![[f20_2.svg]]

**图 20-2　兼容性模型正常读写/调度路径**

**说明**：TiDB 路径是"单一 parser → 单一语义基线 → 看语句类型与是否有执行器"的直线，兼容深度体现在"有没有执行器接入"这一个判定点；OceanBase 路径在入口就按租户 `CompatMode` 把 session 上下文分叉两个方向，但分叉后汇入**同一套**带 `is_oracle_mode()` 分支的共享 resolver/执行层——双模式不是两个数据库，而是一套内核的两条语义分支。

---

## 20.6 故障/异常路径图

图 20-3 展示四类兼容性失效路径：能解析不能执行、语义层静默偏差、DDL/对象差异、以及触发器/PL/外键副作用。它们的共同特征是——**最危险的失效往往不报语法错，而是结果或行为静默偏移**。

![[f20_3.svg]]

**图 20-3　兼容性模型故障/异常路径**

四类路径的具体表现：

- **[A] 能解析不能执行(TiDB 存储过程)**：parser 成功构造 `ast.ProcedureInfo`，看起来"支持"，但 planner/executor 无引用，无法真正运行；迁移时易误判"已兼容"，上线才发现存过逻辑全失效。教训：有语法 ≠ 能执行，须按"执行器是否接入"判定。
- **[B] 模式不可改(OceanBase)**：建租户时选 MySQL 模式，`CompatMode` 写入 `MYSQL=0` 后不可变更；后续想跑 Oracle 语法只能新建 Oracle 模式租户并迁数据。迁移决策错误不是一句 `ALTER` 能修复，而可能涉及导出、转换、重建租户、重新同步。
- **[C] CE 缺 Oracle 语法前端(OceanBase 社区版)**：在 CE 集群尝试 Oracle 模式 SQL，`src/sql/parser` 无 `*oracle_mode*` 词法/语法文件，语法前端不可用(企业版/Cloud 才有)；注意内核共享层仍有 `is_oracle_mode` 分支(约 799 文件)，但发布层结论以官方文档为准。
- **[D] 语义层静默差异(两侧通病，最危险)**：迁移看似"语法都支持"，但 TiDB 默认 sql_mode 含 `NO_AUTO_CREATE_USER`(对齐 MySQL 5.7 而非 8.0)、`REPEATABLE-READ` 名义下是 Snapshot Isolation、`AUTO_INCREMENT` 缓存导致 ID 间隙；OceanBase 非法日期需 `_enable_mysql_compatible_dates` 显式开、MySQL/Oracle 模式视图与错误码不同。这些不报语法错却改变结果，是最难排查的兼容 bug。

TiDB 的典型异常路径还包括：外键级联把单次 DML 扩大为多表读写、`AUTO_INCREMENT` 缓存导致审计脚本误判 ID 间隙。OceanBase 的典型异常路径还包括：MySQL 模式 procedure/trigger 仍有事务控制、返回结果、动态 SQL、外键动作触发 trigger 等限制；Oracle 模式的 PL/sequence/hint/系统视图要按企业版/Cloud 版本矩阵验证。

---

## 20.7 性能、可靠性、运维影响

**兼容性会直接进入性能路径。** 外键不是免费的声明式注释：官方文档明确提醒外键可能导致性能退化，源码也显示外键添加要走 DDL 状态推进与约束检查。在写入路径上，父表存在性检查、子表引用检查、级联更新/删除会扩大事务的 key range 与锁集合。据此可以推测，测试时应把"无外键同样 SQL 的 QPS/延迟"与"开启外键后的 QPS/延迟、锁等待、事务大小"分开观测。

**兼容性会进入可靠性路径。** TiDB 的 `REPEATABLE-READ` 与 MySQL 同名但实现差异明确写在文档中，应用如果依赖 InnoDB 当前读、gap lock、semi-consistent read 或隐式重试行为，就可能在并发场景出现不同结果。OceanBase 的租户模式创建后不可更改，意味着迁移决策错误不是简单 `ALTER DATABASE` 能修复，而可能涉及导出、转换、重建租户、重新同步。

**运维复杂度方面**，TiDB 对 MySQL 生态友好，但不支持特性会迫使存储过程、触发器、事件、UDF 等逻辑上移到应用或任务系统；可以推测，这会增加应用部署、幂等、重试、审计的复杂度。OceanBase 的 `CompatMode` 是建租户时的一次性决策(创建后不可改)，运维需在租户规划阶段就锁定方言；同集群可混跑 MySQL/Oracle 模式租户，提高密度但也提高诊断复杂度——同一段 resolver 在不同租户走不同分支，且 CE、企业版、Cloud 的可用能力不能混写，测试矩阵更大。

**对迁移工具而言，兼容性是端到端问题。** OceanBase OMS 文档关于 TiDB 到 OceanBase MySQL 模式租户的迁移明确提到：增量同步需要 TiCDC 和 Kafka；源端不应在 schema/full migration 期间修改 schema；目标存在 trigger 可能导致迁移失败；DDL synchronization 不支持。这类限制说明迁移成败不只取决于目标数据库是否支持某个 SQL 对象，还取决于迁移工具能否稳定表达该对象及其变更历史。

**版本演进**：TiDB 兼容深度随版本推进(外键 v6.6 引入→v8.5.0 GA)，"是否 GA"需查官方 Release Notes，不能仅凭"有语法"判定生产可用性；OceanBase 4.4.2 官方博客提到 MySQL 递归 CTE `UNION DISTINCT`、Oracle `INTERVAL` 分区等兼容增强，但这是博客级演进提示，资料层级低于 SQL reference，不外推为所有形态、所有版本的实现断言。

--- 〔文献[10]〕

## 20.8 反例与代价

- **TiDB 不合适的场景**：重度依赖数据库端编程的 Oracle/MySQL 存量系统。存储过程仅能解析不能执行、触发器连语法都没有、无 SEQUENCE/PACKAGE/SYNONYM——这类逻辑必须上移应用层重写，且这不是简单 SQL 改写，而会改变事务边界与失败恢复路径。若团队预算不允许重写，TiDB 的"窄而真"反而成为迁移阻力。
- **OceanBase 不合适的场景**：只需轻量 MySQL 兼容、且使用社区版的团队，若误以为"OceanBase 兼容 Oracle"即等同于 CE 可跑 Oracle 模式，则会落空——CE checkout 无 Oracle 模式语法前端，Oracle 模式属企业版/Cloud(发布层)。把"内核有 Oracle 语义分支"误读为"CE 能跑 Oracle SQL"会导致选型误判。
- **把"数据库偶然行为"当业务合同的系统**：例如依赖 `AUTO_INCREMENT` 连续无洞、依赖 InnoDB RR 的特定锁等待、依赖错误码文本、依赖 `SHOW WARNINGS` 的附加计划细节、依赖某个 information_schema 列含义。TiDB 与 OceanBase 都可能在这些地方与源库不同；据此可以推测，兼容性验收必须把这些偶然依赖显性化。
- **双向迁移或长期双写的代价**：OceanBase MySQL 模式提供 MySQL 没有的 partition/subpartition 扩展；TiDB 提供 `AUTO_RANDOM`、placement、TiFlash、TiDB-specific DDL 等扩展。一旦迁移后使用目标库扩展，再回到原 MySQL 或迁到另一个目标就不再是低成本迁移。
- **共同代价——语义层是真正的难点**：两侧都把"协议兼容/语法兼容"做得较为完整，但 sql_mode、非法日期、NULL 排序、大小写折叠、错误码这些**语义层差异**不报语法错却静默改变行为，是迁移成本真正集中的地方。任何"100% 兼容"宣传都应被还原为"协议/语法/执行/语义/发布"五层分别核验。
- **取舍**：这不是"谁更兼容"，而是兼容的**形状不同**——TiDB 把复杂度收窄到单方言+执行优先+明示不支持清单，把兼容代价交给应用迁移；OceanBase 把复杂度铺开到双模式+全链路分叉+社区版/企业版按模式切边界，减少部分业务改造，但增加 mode-aware resolver、session 状态、PL runtime、对象依赖、错误映射与测试矩阵复杂度。**归根结底是把复杂度放在不同位置，没有谁绝对更优——这一总体判断带有推测成分。**

---

## 20.9 测试开发视角的验证点

**可测功能场景**：

- TiDB：验证"有语法 ≠ 能执行"——`CREATE PROCEDURE p() BEGIN ... END` 应能解析但无法实际调用执行；`CREATE TRIGGER ...` 应直接语法报错(无产生式)；`CREATE TABLE ... FOREIGN KEY ...` 应真生效并校验引用完整性(`ddl/foreign_key.go` 校验链)；`SELECT @@sql_mode` 应为默认七项；`@@tidb_enable_foreign_key`、`@@foreign_key_checks` 应为 ON；`AUTO_INCREMENT` 与 `AUTO_ID_CACHE 1` 的间隙行为；`READ-COMMITTED` 与 `REPEATABLE-READ` 下的并发可见性/重试行为。
- OceanBase MySQL 模式：trigger/procedure/function 创建与调用、事务控制限制、外键动作不触发 trigger、MySQL 5.7/部分 8.0 JSON 函数、partition/subpartition、information_schema/mysql 视图差异；并验证 `_enable_mysql_compatible_dates` 开关对非法日期的影响。
- OceanBase Oracle 模式：租户形态可用性、PL、SEQUENCE(`NEXTVAL`/`CURRVAL`)、PACKAGE、SYNONYM、Oracle 数据类型、系统视图、Oracle error 行为；创建租户后验证 `CompatMode` 不可改；社区版不作为 Oracle 模式验收对象。

**可注入/可观测的兼容陷阱**：

- 存储过程"假兼容"：TiDB 上建存储过程不报错但调用失效，验证执行器确实无接入。
- 模式不可改：OceanBase 建 MySQL 模式租户后尝试改为 Oracle 模式应失败。
- CE 缺 Oracle 前端：社区版上 Oracle 模式语法应不可用(语法前端缺失)。
- 语义静默差异：同一非法日期/`GROUP BY`/NULL 排序语句在两侧对比结果差异。
- 并发与失效注入：并发更新同一行、并发插入外键 child、父表删除触发级联、大事务 rollback、DDL 与 DML 并发、schema migration 期间源端 DDL、节点重启导致自增缓存间隙，以及 OceanBase 租户模式选错后的恢复演练(这一组注入清单为推测性建议)。

**关键观测/诊断(经查证，不编造)**：

- TiDB：可查证的配置/变量包括 `transaction_isolation`、`AUTO_ID_CACHE`、`foreign_key_checks`、`tidb_enable_foreign_key`；`SHOW VARIABLES LIKE 'sql_mode'`、`SHOW TRIGGERS`(仅 SHOW，不可创建)。诊断用的 `information_schema` 视图名在源码中已坐实：变量观测面为 `SESSION_VARIABLES`、`GLOBAL_VARIABLES`、`VARIABLES_INFO`，触发器/存储过程观测面为 `TRIGGERS`、`ROUTINES`(对应常量 `TableSessionVar`/`tableGlobalVariables`/`TableVariablesInfo`/`tableTriggers`/`tableRoutines`，`pkg/infoschema/tables.go` L96/L117/L206/L100/L113，release-8.5 @ `67b4876bd57b`);其中 `TRIGGERS`/`ROUTINES` 因触发器/存过未接入执行而对迁移对象多为空集(与 `fetchShowTriggers`/`fetchShowProcedureStatus` 空返回一致)。至于 Prometheus metric 名、日志字段名等运维诊断标识，未在源码视图常量中固定，以官方文档为准。**（不确定）**
- OceanBase：可查证的概念包括租户 compatibility mode、MySQL/Oracle 模式、sequence `NEXTVAL`/`CURRVAL`、stored procedure `CALL`、trigger 限制；兼容模式经内表视图暴露(`CASE ... 0 THEN 'MYSQL' WHEN 1 THEN 'ORACLE'`)。对用户暴露的具体视图名/列名在源码中已坐实：用户可查 `DBA_OB_TENANTS` 视图的 `COMPATIBILITY_MODE` 列得到本租户兼容模式(`CASE compatibility_mode WHEN 0 THEN 'MYSQL' WHEN 1 THEN 'ORACLE' ELSE NULL END`);视图名常量 `OB_DBA_OB_TENANTS_TNAME = "DBA_OB_TENANTS"`(`src/share/inner_table/ob_inner_table_schema_constants.h` L3949)，视图定义见 `ob_inner_table_schema.21151_21200.cpp` L413，Oracle 模式另有同名变体常量 `OB_DBA_OB_TENANTS_ORA_TNAME = "DBA_OB_TENANTS"`(同文件 L4520)，v4.2.5_CE @ `e7c676806fda`。至于 Prometheus metric、trace 字段等运维诊断标识，未在所用资料中直接出现，需进一步查证，不在此编造。**（不确定）**

**验收断言不要只写"SQL 执行成功"。** 更稳妥的断言可以推测为：返回行集、错误码、warning、锁等待、事务提交/回滚结果、迁移工具日志、目标对象定义、触发器副作用、外键级联结果、重复执行幂等性。

---

## 20.10 容易误解点

1. **"OceanBase 兼容 Oracle = 社区版能跑 Oracle SQL"——错。** CE checkout 的 `src/sql/parser/` 只含 MySQL 模式词法/语法文件，`find ... -iname "*oracle_mode*"` 在 v4.2.5_CE 与 4.4.x 均无命中，Oracle 模式语法前端属企业版/Cloud(发布层)。但反过来也别走极端：内核共享层有约 799 文件含 `is_oracle_mode` 分支，**"CE 缺 Oracle 语法前端" ≠ "内核无 Oracle 能力"**(§20.3 两节必须配对理解)。

2. **"TiDB 支持存储过程(因为能 CREATE PROCEDURE)"——错。** 语法层能解析成 `ast.ProcedureInfo`，但 executor/planner/session 全仓无引用，且 `fetchShowProcedureStatus()` 空返回，**不可实际执行**(测试注释 "PROCEDURE and FUNCTION are currently not supported.")。判定兼容必须看"执行器是否接入"，而非"语法是否解析通过"——源码里有 parser/AST ≠ 产品支持。

3. **"TiDB 不支持触发器 = 像存储过程那样能解析不能跑"——错。** 触发器比存储过程更彻底：**连 `CREATE TRIGGER` 语法产生式都没有**，`TRIGGER` token 仅用于权限位与 `SHOW TRIGGERS`。提交即语法报错，不是"解析通过但不执行"。

4. **"OceanBase 的 `compatible` 参数 = 兼容模式"——错。** `compatible`(`ob_parameter_seed.ipp` L576，默认 `"4.2.5.0"`)是**持久化数据格式版本号**，用于跨版本升级门控；MySQL/Oracle 兼容模式是 `CompatMode` 枚举(MYSQL=0/ORACLE=1)。两者名字像、含义完全不同。

5. **"TiDB 的 `ModeOracle` sql_mode 位 = TiDB 有 Oracle 模式"——错。** `ModeOracle` 是 MySQL `sql_mode` 体系里的一个兼容位(`pkg/parser/mysql/const.go`)，与"数据库整体的 Oracle 兼容模式"无关。TiDB 没有 OceanBase 那种租户级 Oracle 模式。

6. **"隔离级别变量名相同 = 行为等价"——错。** TiDB 以 `REPEATABLE-READ` 名义对外兼容 MySQL，实现的却是 Snapshot Isolation，与 InnoDB RR 在当前读、gap lock、隐式重试上都有差异。迁移验收必须用并发行为验证，而不是把变量显示值当作 InnoDB 行为等价证明。

7. **"协议/语法兼容 = 迁移无忧"——错。** 最危险的是**语义层静默差异**：sql_mode 默认值差异(TiDB 含 `NO_AUTO_CREATE_USER`，同 MySQL 5.7 而非 8.0)、OceanBase 非法日期需 `_enable_mysql_compatible_dates` 显式开、NULL 排序/大小写折叠/错误码差异、`AUTO_INCREMENT` 间隙——这些不报语法错却改变行为。

---

## 20.11 本章结论

1. **兼容性必须拆成五层**(协议/语法/执行/语义/发布)而非一个布尔标签；最常见的误判是把"有语法"当成"能执行"，把"内核有语义分支"当成"该版本能用"——两者都需按执行器接入与发布归属分别核验。可以推测，兼容性的本质是 parser、resolver、catalog、optimizer、事务、锁、DDL、错误码与工具链共同给出的一份行为合同，而不是一张 SQL parser 检查表。

2. **TiDB 是单方言深度兼容(窄而真)**：仅 MySQL 协议+方言(论文逐字 "supports the MySQL protocol")，默认 sql_mode 七项与 MySQL 5.7 对齐；外键 v8.5.0 GA 且进入 DDL 状态机与内核校验链(`ddl/foreign_key.go`，默认开)；但存储过程仅解析不可执行(executor 无引用)、触发器连 `CREATE` 语法都没有、事件/UDF 官方文档明示不支持。

3. **TiDB 的 `REPEATABLE-READ` 是以 Snapshot Isolation 对外兼容 MySQL 名称**，不能把隔离级别变量名当作 InnoDB 行为等价证明；`AUTO_INCREMENT` 在分布式缓存下可能有间隙，`AUTO_ID_CACHE 1` 更接近 MySQL 但不承诺绝对无洞。

4. **OceanBase 是租户级双模式兼容(宽而分层)**：`CompatMode` 仅 MYSQL=0/ORACLE=1 两值、创建时定不可改；CE 源码无 Oracle 模式语法前端文件，但兼容模式进入 session/SQL mode 执行上下文、共享 resolver 约 799 文件含 `is_oracle_mode` 分支，且内核有可执行 PL 引擎(`src/pl/`)与 SEQUENCE/SYNONYM/PACKAGE/TRIGGER resolver——兼容是从 session 上下文到执行的全链路分叉，Oracle 模式发布归属(企业版/Cloud)以官方文档为准。

5. **本质差异在存储过程/触发器**：OceanBase 有可执行 PL 引擎与触发器 DDL，TiDB 存储过程仅解析、触发器无语法——这是两侧兼容深度最尖锐的对照，直接决定 Oracle/重存过 MySQL 应用能否平迁；但 OceanBase 覆盖面更宽的代价是 mode、版本、edition 与测试矩阵复杂度更大。

6. **真正的兼容深水区是语义层**：sql_mode 默认值、非法日期开关(`_enable_mysql_compatible_dates`)、NULL 排序/大小写/错误码、自增 ID 形态等不报语法错却静默改变行为，是迁移最难排查的成本；`compatible` 版本参数 ≠ 兼容模式，二者不可混淆。

7. **迁移路径建议**：从 MySQL 迁到 TiDB，应优先排查 trigger/procedure/UDF、外键、事务隔离、自增 ID、partition DDL 与 TiCDC/BR/Lightning 组合限制；从 MySQL/Oracle 迁到 OceanBase，应先确定租户模式与产品形态，再验证 PL/trigger/sequence/视图/错误码。综合来看(此为推测性总括)，没有谁更兼容，只有兼容的形状不同，选型取决于是否需要 Oracle 兼容与数据库端编程能力。

---

## 20.12 参考文献

[1] TiDB: A Raft-based HTAP Database. 论文，VLDB 2020，PVLDB 13(12[EB/OL]. https://www.vldb.org/pvldb/vol13/p3072-huang.pdf.
 （支撑:§20.1/§20.2 "TiDB supports the MySQL protocol and is accessible by MySQL-compatible clients"(逐字)与 SQL/存储层分离的架构）
[2] OceanBase: A 707 Million tpmC Distributed Relational Database System. 论文，VLDB 2022[EB/OL]. https://vldb.org/pvldb/vol15/p3385-xu.pdf.
 （支撑:§20.1/§20.3 OceanBase SQL engine、多租户与分布式关系数据库背景。注：其 707M tpmC 为 TPC-C Result ID 120051701、OceanBase v2.2 Enter）
[3] MySQL Compatibility. TiDB 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/mysql-compatibility/.
 （支撑:逐字支撑 §20.2/§20.10 不支持清单("Stored procedures and functions, Triggers, Events, User-defined functions")、§20.2 默认）
[4] Foreign Key Constraints. TiDB 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/foreign-key/.
 （支撑:§20.2/§20.7 外键 v6.6 引入、v8.5.0 GA、foreign_key_checks 默认 ON 与性能退化提示。WebFetch 逐字确认 "Starting from v8.5.0, this）
[5] TiDB Transaction Isolation Levels. TiDB 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/transaction-isolation-levels/.
 （支撑:§20.2/§20.7/§20.10 TiDB 以 Snapshot Isolation 对外兼容 REPEATABLE-READ 名称、与 MySQL/ANSI Repeatable Read 的差异。）
[6] TiDB 8.5.0 Release Notes. TiDB 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/release-8.5.0/.
 （支撑:§20.2 外键 GA 发布层归属，逐字 "Support foreign keys (GA) ... becomes generally available (GA) in v8.5.0"。WebFetch 逐字确认。）
[7] Compatibility modes(创建时确定不可改；Community Edition 仅 MySQL 模式). OceanBase 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001103755.
 （支撑:§20.3/§20.10 租户创建时选择 MySQL/Oracle 模式、创建后不可改，以及 "Community Edition provides only the MySQL mode" 的发布层结论。URL 格式符）
[8] Compatibility with MySQL(Community Edition). OceanBase 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000000829643.
 （支撑:§20.3/§20.4 OceanBase MySQL 模式对 MySQL 5.7、多数语法、JSON 函数、视图、partition、optimizer 的兼容与差异(含不支持 spatial、非模板化 subpart）
[9] Compatibility with Oracle Database(Enterprise). OceanBase 官方文档[EB/OL]. https://en.oceanbase.com/docs/enterprise-oceanbase-database-en-10000000000905632.
 （支撑:§20.3/§20.4 Oracle 模式数据类型、SQL、PL、SEQUENCE/SYNONYM/PACKAGE、系统视图与不支持项边界，并说明该资料属企业版文档(发布层归属)。）
[10] OceanBase 4.4.2 LTS 发布(MySQL 递归 CTE UNION DISTINCT、Oracle INTERVAL 分区等兼容增强). OceanBase 官方博客[EB/OL]. https://en.oceanbase.com/blog/26083442944.
 （支撑:§20.7 兼容能力随版本演进的发布层背景，按官方博客而非手册级资料处理，不外推为所有形态的实现断言。）
[11] pingcap/tidb 仓库(release-8.5 @ 67b4876bd57b)— pkg/parser/mysql/const.go、pkg/parser/ast/misc.go、pkg/parser/parser.y、pkg/parser/ast/procedure.go、pkg/parser/misc.go、pkg/executor/show.go、pkg/sessionctx/variable/sysvar.go、pkg/sessionctx/variable/tidb_vars.go、pkg/ddl/foreign_key.go、pkg/infoschema/tables.go. 源码[EB/OL]. https://github.com/pingcap/tidb/tree/release-8.5.
 （支撑:§20.2 默认 sql_mode 与隔离级别常量、外键开关默认+ddl/foreign_key.go 状态机/校验链、存储过程仅解析(executor 无 ProcedureInfo 引用、fetchShow）
[12] oceanbase/oceanbase 仓库(v4.2.5_CE @ e7c676806fda，旁核 4.4.x @ d4bef8d29a4c)— deps/oblib/src/lib/worker.h、src/share/ob_get_compat_mode.{cpp,h}、src/share/inner_table/ob_inner_table_schema.21151_21200.cpp、src/share/inner_table/ob_inner_table_schema_constants.h、src/sql/session/ob_basic_session_info.h、src/sql/parser/、src/sql/resolver/ (递归)(含 ob_stmt_type.h、ob_trigger_resolver)、src/pl/、src/share/parameter/ob_parameter_seed.ipp. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE.
 （支撑:§20.3 CompatMode(MYSQL=0/ORACLE=1)+内表视图 0→MYSQL/1→ORACLE、CE 无 *oracle_mode* 语法前端、is_oracle_mode 约 799 文件）
## 20.13 信息可信度自评

本章证据分层清晰，主线可信度较高。

- **官方文档/博客明确**：TiDB 不支持存储过程/触发器/事件/UDF(逐字清单)、默认 sql_mode 七项(对齐 MySQL 5.7)、外键 v6.6 引入 / v8.5.0 GA / `foreign_key_checks` 默认 ON、`REPEATABLE-READ` 名义下的 Snapshot Isolation 差异、`AUTO_INCREMENT` 与 `AUTO_ID_CACHE 1`——均经 docs.pingcap.com 逐字确认；OceanBase 租户 MySQL/Oracle 双模式、创建后不可改、CE 仅 MySQL 模式、Oracle 模式企业版/Cloud 归属、MySQL/Oracle 模式兼容边界——以 en.oceanbase.com 官方文档为准。

- **论文级**：TiDB MySQL 协议兼容与 SQL/存储层分离来自 VLDB 2020 论文(已用 pdftotext 抽出逐字句；OceanBase 论文仅用于架构背景，707M tpmC 不作为生产性能结论。

- **源码级**：TiDB 默认 sql_mode/隔离级别常量、外键开关默认值与 `ddl/foreign_key.go` 状态机/校验链、存储过程仅解析(executor 无 `ProcedureInfo` 引用、`fetchShowProcedureStatus`/`fetchShowTriggers` 空返回)、触发器无 `CreateTriggerStmt`；OceanBase `CompatMode` 枚举与内表视图映射、compatibility mode 进入 session/SQL mode、CE 无 Oracle 语法前端文件、`is_oracle_mode` 约 799 文件、PL 引擎与对象 resolver、`compatible` 参数与日期开关——commit-pin 到统一基线，确认到文件/符号级。

- **工程推测**："有语法 ≠ 能执行"的五层兼容框架、"窄而真 / 宽而分层"的总体定位、"兼容是一份行为合同"均为本章归纳措辞；TiDB 存储过程"仅解析不可执行"基于"parser 外无 `ProcedureInfo` 引用 + show 空返回 + 测试注释"的强推断。

- **反查证发现**：TiDB v8.5.0 源码确有 `CREATE PROCEDURE` 的 parser/AST 痕迹，但官方文档仍将其列为不支持；本章按官方对外文档定性为不支持，源码痕迹仅作"解析层痕迹不等于支持"的反例，不反推为能力承诺。同理，OceanBase 开源仓库存在 Oracle/PL/trigger/sequence 相关 resolver 路径，但 compatibility modes 文档明确 CE 仅 MySQL 模式，本章不由源码反推 CE 的 Oracle 模式能力。

- **源码已坐实(本轮升级)**：诊断/观测视图名已由源码核实——TiDB 变量观测面为 `information_schema.SESSION_VARIABLES`/`GLOBAL_VARIABLES`/`VARIABLES_INFO`、触发器/存过观测面为 `TRIGGERS`/`ROUTINES`(`pkg/infoschema/tables.go`，release-8.5);OceanBase 用户可查 `DBA_OB_TENANTS` 视图的 `COMPATIBILITY_MODE` 列读取本租户兼容模式(`ob_inner_table_schema_constants.h` + `ob_inner_table_schema.21151_21200.cpp`，v4.2.5_CE)。

- **仍不确定 / 需进一步查证(未编造)**：OceanBase 模式选择的精确语法、模式不可改的逐字表述、CE/企业版边界以官方文档为准；各对象在两模式下的精确暴露面差异(源码仅确认到对象级 resolver 存在)；TiDB/OceanBase 两侧 Prometheus metric、trace/日志字段名，以及未在所用资料中直接出现的参数名——均显式标注待查，不在正文编造。

---


# 第 21 章 安全 / 权限 / 审计 / 加密

## 21.1 本章核心问题

前面几章讨论的分片、共识、事务、存储引擎，关注的是「数据如何被正确、可扩展地存取」。本章关注另一条正交但同样底层的主线：谁被允许做什么、做过的事情能否被追溯、落盘与传输中的数据在被偷走后是否仍然安全。安全能力在分布式数据库里不是一组开关，而是连接、权限、审计、密钥、备份与运维控制面的组合。本章把它收敛为三条可验证的路径：正常请求如何被认证与授权、离线数据如何被加密与恢复、异常情况下如何证明没有越权或漏审。

这条主线之所以比单机数据库更复杂，有三个根本原因。

第一，安全边界被分布式拓扑撕开。单机 MySQL 的认证、权限、审计都在一个进程里完成；TiDB 把无状态 SQL 层(认证/权限)、TiKV(落盘加密)、PD(元数据)、BR(备份加密)分到不同进程，加密职责必须跨组件分层；OceanBase 把核心能力收进 OBServer，却又叠加了多租户这层硬边界，sys 租户对所有 user 租户拥有跨租户运维与诊断能力，跨租户访问与特权账号管理本身就是新的攻击面。两者都不能简单写成「支持权限、支持审计、支持加密」，必须明确落在哪个组件、哪个产品形态、哪个版本。

第二，企业级安全能力高度依赖产品形态。审计、透明加密(TDE)、Oracle 兼容这类能力，在开源/社区版、企业版、云服务三种形态下的可用性差异极大。本章的硬约束是：不据源码断言「企业版独有功能清单」，所有 edition 边界以官方文档为准。源码层根本无法判定 license 边界——CE 仓库里存在 TDE/安全审计的代码路径，但这些路径受编译宏保护、且官方文档明确 CE 不支持，二者必须分开看待(详见 §21.3、§21.8)。

第三，合规与金融场景把这些能力从「可选项」变成「准入门槛」。等保、金融行业规范、GDPR 等要求落到具体技术点：国密 SM4 落盘加密、SQL 级审计留痕、密钥轮换可观测、传输强制 TLS、最小权限。

基于这三点，本章把四个主题——权限/认证、审计、加密(传输 + 静态)、安全运维边界——在 TiDB 与 OceanBase 上逐一对照，重点写清每项能力落在哪个组件、哪个产品形态、哪个版本，以及测试开发视角下如何验证权限越界、审计漏报、备份解密、密钥轮换。多租户隔离的资源维度详见第 17 章，本章只讲安全边界维度。

边界与基线说明：TiDB 以 8.5.x LTS 为基准(对照 7.5.x 仅作版本口径说明),TiKV/PD 取 release-8.5，源码 commit 统一锁定 pingcap/tidb · release-8.5 · 67b4876bd57b、tikv/tikv · release-8.5 · 1f8a140b6d46;OceanBase 以 v4.2.5_CE 文档与源码为主(commit e7c676806fda)，对 4.3.5/4.4.x 的差异作版本注记。源码只支撑实现路径，不替代企业版与 Cloud 的授权矩阵。

---

## 21.2 TiDB 的实现

### 21.2.1 权限模型：静态权限 + 动态权限 + SEM

TiDB 完整复用 MySQL 5.7 的静态权限位系统，并在其上叠加 SQL Roles 与 MySQL 8.0 风格的动态权限。静态权限位枚举常量真实存在于 `pkg/parser/mysql/privs.go`:`SelectPriv`、`CreatePriv`、`SuperPriv` 等；`AllGlobalPrivs`(行 317)、`AllDBPrivs`、`AllTablePrivs` 为真实定义的权限集合变量；`StaticGlobalOnlyPrivs`(行 329)包含 `ProcessPriv, ShowDBPriv, SuperPriv, CreateUserPriv, ShutdownPriv, ReloadPriv, FilePriv, ReplicationClientPriv` 等只能在全局生效的权限。

动态权限自 v5.1 引入，目的是把过于宽泛的 `SUPER` 拆细。源码 `pkg/privilege/privileges/privileges.go` 中 `UserPrivileges` 统一负责静态权限、动态权限与扩展认证插件回调，`var dynamicPrivs []string` 列出默认动态权限。其条目数随版本变化，锁定版本时必须区分：

- release-8.5(本章源码基准)默认 16 项——`BACKUP_ADMIN`、`RESTORE_ADMIN`、`SYSTEM_USER`、`SYSTEM_VARIABLES_ADMIN`、`ROLE_ADMIN`、`CONNECTION_ADMIN`、`PLACEMENT_ADMIN`、`DASHBOARD_CLIENT`、6 个 `RESTRICTED_*` 系列(`RESTRICTED_TABLES_ADMIN`、`RESTRICTED_STATUS_ADMIN`、`RESTRICTED_VARIABLES_ADMIN`、`RESTRICTED_USER_ADMIN`、`RESTRICTED_CONNECTION_ADMIN`、`RESTRICTED_REPLICA_WRITER_ADMIN`)、`RESOURCE_GROUP_ADMIN`、`RESOURCE_GROUP_USER`。
- release-7.5 默认 15 项——较 8.5 少 `RESOURCE_GROUP_USER`(该项是 8.5 线才加入的)，其余 15 项一致。

直接比对 pingcap/tidb 各 tag 的 `dynamicPrivs` 切片可确认：8.5.x 为 16 项(含 `RESOURCE_GROUP_USER`),7.5.x 全线为 15 项(无 `RESOURCE_GROUP_USER`)。因此「默认 16 项」只对 8.5.x 成立，对 7.5.x 不成立；笼统称「16 项」会被判证伪。本章源码基准锁定 release-8.5，正文以 16 项为准并同时标注 7.5.x 为 15 项；权限名清单、6 个 `RESTRICTED_*`、以及插件扩展机制在两版本均无误，仅总数随版本不同。

`RegisterDynamicPrivilege(privName string)`(校验非空、≤32 字符、去重后 append 到 `dynamicPrivs`)允许插件在 `OnInit` 中注册新动态权限，官方设计文档 `2021-03-09-dynamic-privileges.md` 佐证，并因此明确写「具体列表可能因 TiDB 安装而异」。基于此机制，可以只授予某账号 `BACKUP_ADMIN` + `RESTORE_ADMIN`，使其只能执行备份恢复而无法触及其他系统操作。这说明 TiDB 的安全边界是 user@host、role、动态权限、插件与配置的组合，而非 OceanBase 式的原生 Tenant。

SEM(Security Enhanced Mode，安全增强模式)是 TiDB 对 MySQL 的扩展，默认关闭，本质是管理员权限收敛机制而非租户隔离。其设计文档(`docs/design/2021-03-09-security-enhanced-mode.md`)说明核心动机：在 DBaaS(如 TiDB Cloud)场景下，`SUPER` 既被终端用户需要、又被运维方(SRE)需要，因此必须把「对终端用户安全」与「基础设施级控制」分开。SEM 状态由 `enable-sem` 配置(对应 sysvar `tidb_enable_enhanced_security`)控制，工程实现在 `pkg/util/sem/sem.go`：

- `Enable()`(行 73)把 sysvar 置 `On`，强制 `Hostname` 为 `DefHostname`，并写日志 `tidb-server is operating with security enhanced mode (SEM) enabled`;
- `IsInvisibleSchema`(行 98)隐藏 `metrics_schema`;
- `IsInvisibleTable`(行 104)隐藏 `mysql` 库的 `expr_pushdown_blacklist`/`gc_delete_range`/`tidb`/`global_variables` 等、`information_schema` 的 `cluster_config`/`cluster_log`/`tidb_hot_regions` 等诊断表、`performance_schema` 的 profile 表；
- `IsInvisibleSysVar`(行 136)隐藏一批诊断 sysvar(含 `tidb_audit_redact_log`，源码注释标其为「by a plugin」安装的变量);
- `IsRestrictedPrivilege`(行 169)判定 `RESTRICTED_` 前缀(11 字符)的动态权限不被 `SUPER` 满足——这是 SEM 的核心机制：开启 SEM 之后，持有 `SUPER` 也无法越过 `RESTRICTED_USER_ADMIN` 用户、无法访问受限诊断对象。

值得记下的一个底层事实：`tidb_audit_redact_log` 常量(源码符号 `tidbAuditRetractLog`，行 64)在注释里被明确标为「by a plugin」安装，这印证了审计在开源 TiDB 内核里是插件化能力，而非内核内置(见 §21.2.3)。测试 SEM 时应验证 `SUPER` 用户在 SEM 下不能绕过受限对象，而不是只验证普通 GRANT/REVOKE。

### 21.2.2 认证

支持的认证插件常量定义在 `pkg/parser/mysql/const.go`(行 188-196):`AuthNativePassword="mysql_native_password"`、`AuthCachingSha2Password="caching_sha2_password"`、`AuthTiDBSM3Password="tidb_sm3_password"`(国密 SM3，需 TiDB-JDBC)、`AuthSocket="auth_socket"`、`AuthTiDBAuthToken="tidb_auth_token"`、`AuthLDAPSimple`/`AuthLDAPSASL`。官方兼容性文档给出版本归属：`caching_sha2_password` 自 5.2.0、`auth_socket` 自 5.3.0、`tidb_sm3_password` 自 6.3.0、`tidb_auth_token` 自 6.4.0、LDAP 两种自 7.1.0。

- 连接 TLS:TiDB Server 用 `ssl-cert`、`ssl-key`、可选 `ssl-ca`、`tls-version` 等参数启用安全连接；账户层还可用 `REQUIRE SSL` 或 `REQUIRE X509` 强制某用户必须通过 TLS 或证书认证。若未配置 CA，连接可能已加密但不能充分防中间人，因此生产验证应检查证书链与客户端校验，而不是只看连接是否成功(传输加密的组件配置见 §21.2.4)。
- LDAP:`pkg/privilege/privileges/ldap/` 模块下 simple 与 SASL 两种绑定方式各有独立实现文件(`simple.go`/`sasl.go`)。
- JWT / auth token:`tidb_auth_token` 通过 JWT(jwx 库 + openid)实现，`defaultTokenLife = 15 * time.Minute`(privileges.go 行 74)，主要服务 TiDB Cloud 的免密登录。
- 密码策略：自 v6.5.0 起 TiDB 内核内置密码复杂度、过期、复用、失败登录锁定四类策略。失败登录锁定通过 `CREATE/ALTER USER` 的 `FAILED_LOGIN_ATTEMPTS N` 与 `PASSWORD_LOCK_TIME N|UNBOUNDED` 配置，仅支持账号级、不支持全局级——因为分布式架构不能像单机 MySQL 那样把锁定状态存在单机内存里。

### 21.2.3 审计

审计是 TiDB 安全能力里产品形态差异最大的一块，需按内核、企业版、Cloud 三层分别叙述，不能写成开源默认能力：

- 开源内核：只提供审计的插件 SPI 框架。`pkg/plugin/audit.go`(行 107-119)定义 `AuditManifest` 结构，提供四个回调：`OnConnectionEvent`、`OnGeneralEvent`、`OnGlobalVariableEvent`、`OnParseEvent`；事件枚举真实存在(`GeneralEvent` 的 `Starting/Completed/Error`、`ConnectionEvent` 的 `Connected/Disconnect/ChangeUser/PreAuth/Reject`、`ParseEvent` 的 `PreParse/PostParse`)。审计插件本体不在开源仓库——源码层只确认到 SPI 接口级，具体审计日志格式与落盘属于插件/企业能力。
- TiDB 企业版：自 v7.1 提供重新设计的数据库审计能力，替代旧的 TiDB Audit Plugin，支持 text/JSON 多种格式、可指定审计事件与存储位置；插件经 TiUP 或 TiDB Operator 部署。其增强能力须按企业版文档或发布说明限定。
- TiDB Cloud：审计能力按 tier 分层。Dedicated 集群支持把数据库审计日志写入客户自己的云存储(S3/GCS/Azure Blob)，默认关闭、需申请且需 filter 规则才记录；Starter 不支持；Essential 提供 Beta 审计能力，且默认对敏感数据做 redaction。Essential 文档另提醒：审计日志可能不按顺序存储，消费端需自行排序验证。这是同一品牌不同形态间能力差异显著的典型例子，合规评估时不能一概而论。

### 21.2.4 加密：分层职责

TiDB 的加密能力明确按组件分层，这是本章一个易被误解的关键点：

**表 21-1　加密：分层职责**

| 加密层 | 承担组件 | 源码确认 |
|---|---|---|
| 传输 TLS(client↔TiDB、组件互信) | TiDB Server | `pkg/config/config.go` 的 `type Security struct`(行 598):`SSLCA`、`ClusterSSLCA` 等；`AutoTLS bool`(行 615，默认 false)；集群内部 TLS 经 `tikvcfg.NewSecurity()` 桥接(行 677) |
| 算子溢写临时文件加密 | TiDB Server | `SpilledFileEncryptionMethod`(行 611):`plaintext`(默认)/ `aes128-ctr` |
| 静态数据落盘加密(at-rest) | TiKV(非 TiDB Server) | `components/encryption/`(见下) |
| 备份加密(at-rest 之外的独立链路) | BR | `br/pkg/encryption/manager.go`(见 §21.8) |

也就是说，TiDB Server 层本身不做静态数据加密，它只负责传输 TLS + 算子溢写文件(aes128-ctr)，真正的数据落盘加密由 TiKV 承担，备份加密由 BR 独立承担。

落到 TiKV 这一侧，其静态加密是一个独立 component(`components/encryption/src/`)，采用信封加密(envelope encryption):master key 由用户或外部 KMS 管理，data key 由 TiKV 生成并实际加密数据。

- 算法(`crypter.rs` 行 19-22):`Aes128Ctr`(key 16B)、`Aes256Ctr`(key 32B)、`Sm4Ctr`(key 16B，国密，v6.3.0+)；官方文档另列 `aes192-ctr`。
- 默认配置(`config.rs` 行 25-47):`data_encryption_method = Plaintext`(默认关闭)、`data_key_rotation_period = 7 days`(168 小时)、`enable_file_dictionary_log = true`；另支持 master key 与 previous master key 配置以支撑轮换。
- 主密钥后端(`MasterKeyConfig` 枚举，行 224):`Plaintext` / `File` / `Kms`。KMS 后端 `KmsBackend`(`master_key/kms.rs` 行 40)支持 AWS / Azure / GCP KMS。File 后端要求密钥文件为 hex 编码(`key_len*2 + 1`)。
- 数据密钥轮换：`manager/mod.rs` 的 `maybe_rotate_data_key`(行 315)、`generate_data_key`(行 442)、`DataKeyManager`(行 450)实现自动轮换；数据密钥本身用 AES256-GCM 加密(认证加密)。轮换不重写旧文件，而是通过 RocksDB 正常 compaction 渐进生效。

加密范围覆盖数据文件、WAL、MANIFEST、含用户数据的临时文件；不覆盖加密密钥本身、core dump、info 日志(需另开 log redaction)。PD 的 at-rest 加密仍标注为 experimental;TiFlash 用同样算法、可与 TiKV 共享主密钥。BR 备份加密是独立机制：TiDB 8.5.0 发布说明确认 full backup 与 log backup 的 client-side encryption 在 8.5.0 GA；源码 `br/pkg/encryption/manager.go` 支持 plaintext data key 与 master-key-based 两条解密路径(展开见 §21.8)。

从测试开发视角看，TiDB 的安全主线应拆成三层验收：入口层验证用户是否必须使用 TLS、证书是否被客户端校验、旧驱动在证书策略收紧后是否按预期失败；SQL 权限层验证同一账号在 role 启用前后、动态权限授予前后、SEM 开关前后是否只能执行预期命令；离线数据层验证同一份备份在具备对象存储权限但缺少 KMS 解密权限时必须失败、在具备 KMS 但缺少恢复 SQL 权限时也必须失败。只有三层同时成立才能说「备份链路被保护」，不能只看备份文件是否加密。换言之，TiDB 备份安全的关键不是单一 SQL 权限，而是 `BACKUP_ADMIN`/`RESTORE_ADMIN`、BR 运行节点凭据、对象存储 policy、KMS key policy 与审计日志共同组成的链路。据此推测：密文可读但 KMS 不可用会导致恢复失败，KMS 权限过宽则削弱备份加密意义。

还要分清 TiDB classic 与 TiDB Cloud 的控制面差异。classic 集群更多依赖用户自管证书、配置文件、KMS、对象存储与运维账号；Cloud 形态会叠加项目、组织、Cloud IAM、private endpoint、审计开关与云厂商 KMS 权限。正文不能把 Cloud 的控制面直接写到 classic，也不能把 classic 的自主管理风险反推为 Cloud 默认风险。较稳妥的表述是一种推测：TiDB 的安全能力跨越数据库内核、存储组件、备份工具与外部云资源，最终安全性取决于这些控制面的组合配置。

--- 〔文献[1-7]〕

## 21.3 OceanBase 的实现

### 21.3.1 权限模型：租户级 + MySQL/Oracle 双模

OceanBase 的权限模型有两个 TiDB 不具备的维度：租户级隔离与双兼容模式。MySQL-compatible tenant 与 Oracle-compatible tenant 的账户、角色、系统权限与对象权限语义不同；Community Edition 以 MySQL 模式为主，Oracle 兼容与高阶安全能力须按企业版/Cloud 文档核查。

权限位定义在 `src/share/schema/ob_priv_type.h`:`typedef int64_t ObPrivSet`(行 20)，权限以 bit-shift 宏定义。真实存在的权限位包括 `OB_PRIV_CREATE_USER`、`OB_PRIV_SELECT`、`OB_PRIV_SUPER`，以及几个安全相关的特权位：`OB_PRIV_AUDIT`(行 102，审计专用权限)、`OB_PRIV_ALTER_TENANT`(行 111，租户级特权)、`OB_PRIV_ALTER_SYSTEM`、`OB_PRIV_CREATE_TABLESPACE`(行 123，表空间/加密相关)。聚合宏 `OB_PRIV_ALL` 组合上述位。

权限检查中枢是 `class ObPrivilegeCheck`(`src/sql/privilege_check/ob_privilege_check.h` 行 33，实现在同目录 `.cpp`):`get_stmt_need_privs`(行 87)把语句映射到 `ObStmtNeedPrivs`(由 `ObNeedPriv` 数组构成)，并按 DDL、DCL、DML 收集所需权限，`ObSessionPrivInfo` 承载会话级权限。MySQL 模式与 Oracle 模式在不同文件分流(`ob_privilege_check` vs `ob_ora_priv_check`)，这是双模权限模型的源码级铁证；4.3.5 同路径四个文件齐全。需要强调：该源码只显示权限检查进入 SQL resolver/执行前路径，不等于完整的产品授权矩阵，OceanBase 的测试不能只按 MySQL `GRANT` 覆盖。

权限层级上，官方文档给出三级：用户级(`GRANT ... ON *.*`)、数据库级(`ON db.*`)、对象级(`ON db.table`)。MySQL 模式权限类型包含 `SELECT/CREATE/INSERT/UPDATE/DELETE/ALTER/INDEX/CREATE USER/SUPER/PROCESS/CREATE VIEW/CREATE SYNONYM` 等；Oracle 模式则有 `DBA/CONNECT/CREATE SESSION/SELECT ANY TABLE/SELECT ANY DICTIONARY` 等系统权限与角色。

在用户、数据库、对象三级之上，sys 租户在权限模型里是特殊存在，也是本章的高风险对象：它维护系统表、执行集群级运维(详见第 13、17 章)，对 user 租户拥有跨租户特权。源码 `src/observer/virtual_table/ob_gv_sql_audit.cpp` 显示，SQL Audit 虚拟表打开时，sys 租户会提取全部 tenant id，普通租户只加入自身 effective tenant id；官方 `GV$OB_SQL_AUDIT` 文档也说明该视图 tenant-specific，只有 sys 租户能查询其他租户的记录。因此 OceanBase 的租户边界应写成「业务租户隔离 + sys 租户特权运维路径」，而非绝对不可见。业务租户的隔离可以降低横向越权风险，但 sys 租户承担集群级运维、诊断与资源管理职责，天然拥有更高可见性；安全评审不能只检查业务账号之间是否隔离，还要检查 sys 租户的登录来源、口令与密钥保管、操作审批、SQL Audit 查询留痕、ODP/OBServer 管理通道，以及运维自动化脚本是否复用了过宽凭据。由此可以推测，这正是 OceanBase 多租户安全边界的核心：sys 租户既是隔离的保障者，也是最高价值的攻击目标。

### 21.3.2 审计：两套独立子系统

**表 21-2　审计能力「产品形态 × 可用性」矩阵**

| 形态/版本 | 审计机制 | 是否可用 | 默认状态 | 存储落点 | 关键约束/格式 |
|---|---|---|---|---|---|
| TiDB 开源内核 | 审计插件 SPI 框架（`AuditManifest`，四回调 `OnConnectionEvent`/`OnGeneralEvent`/`OnGlobalVariableEvent`/`OnParseEvent`） | 仅 SPI 接口，无审计本体 | — | — | 审计插件本体不在开源仓库；日志格式与落盘属插件/企业能力 |
| TiDB 企业版（v7.1） | 重新设计的数据库审计能力，替代旧 TiDB Audit Plugin | 可用 | — | 可指定存储位置 | text/JSON 多格式、可指定审计事件；经 TiUP 或 TiDB Operator 部署 |
| TiDB Cloud Dedicated | 数据库审计日志 | 支持 | 默认关闭，需申请 | 客户自己的云存储（S3/GCS/Azure Blob） | 需 filter 规则才记录 |
| TiDB Cloud Starter | — | 不支持 | — | — | — |
| TiDB Cloud Essential | Beta 审计能力 | 支持（Beta） | 默认对敏感数据 redaction | — | 审计日志可能不按顺序存储，消费端需自行排序验证 |
| OceanBase `GV$OB_SQL_AUDIT`（含 CE） | SQL 执行级诊断视图，内存环形缓冲 | 可用 | — | 内存环形缓冲（保留窗口短、会被淘汰） | 记录客户端/服务端 IP、SQL 文本、执行时间、计划 ID、错误码等；主要用于性能诊断而非合规审计；4.0.0+ 名 |
| OceanBase Security Audit（企业能力，CE 不支持） | 基于 `AUDIT`/`NOAUDIT` 语句的策略化安全审计 | CE 不支持（受 `OB_BUILD_AUDIT_SECURITY` 编译宏保护） | — | 可存文件或内部表（`DBA_AUDIT_TRAIL`/`USER_AUDIT_TRAIL` 等视图查询） | 支持 statement/object auditing，不支持 unified auditing 与 FGA、privilege/network auditing；官方明确 CE 不支持 |

OceanBase 的审计有两条容易被混淆的链路，源码层确认它们是独立子系统：

1. `GV$OB_SQL_AUDIT`(SQL 执行级诊断视图)：由 `src/observer/virtual_table/ob_gv_sql_audit.{h,cpp}` 实现，头文件定义列枚举(从 `SERVER_IP = OB_APP_MIN_COLUMN_ID` 行 68 起，含 `TRACE_ID`、`FLT_TRACE_ID`、`PL_TRACE_ID` 等列),`inner_get_next_row`(行 49)为取行入口。它是一个内存环形缓冲，记录每条 SQL 的客户端/服务端 IP、SQL 文本、执行时间、等待时间、计划 ID、错误码等，主要用于性能诊断而非合规审计；在 4.0.0+ 版本叫 `GV$OB_SQL_AUDIT`(旧称 `gv$sql_audit`)，官方按文档书写外部视图名，源码文件名仅作实现路径引用。因为是内存环形缓冲，保留窗口短、会被淘汰(第 19 章已展开：内存达阈值后从 90%→80% 淘汰)，不能当作长期审计留痕。

2. Security Audit(策略化安全审计 / audit trail)：这是独立于 `GV$OB_SQL_AUDIT` 的、基于 `AUDIT`/`NOAUDIT` 语句的安全审计子系统，源码涉及 `src/sql/monitor/ob_security_audit_utils.h`、`ob_security_audit.h`、`ob_audit_action_type.h`，以及 schema 侧 `src/share/schema/ob_security_audit_mgr.{h,cpp}`(含 tenant/user/session/action/status/SQL text 字段与规则管理器)。审计触发链路在 `ob_sql.cpp`、`ob_result_set.cpp`、`ob_sql_session_mgr.cpp` 中以 `handle_security_audit` / `check_allow_audit` / `get_audit_units` / `record_audit_data` 串联；审计结果可经 `DBA_AUDIT_TRAIL`/`USER_AUDIT_TRAIL` 等视图查询。关键约束是：这套安全审计代码受 `OB_BUILD_AUDIT_SECURITY` 编译宏保护，官方安全概览明确把 security audit 列为企业安全能力、且 CE 不支持；源码存在不等于 CE 默认可用(见 §21.8)。

 源码标识符确认：在 v4.2.5_CE checkout 中，该安全审计子系统的真实命名可直接核对——`src/sql/monitor/ob_security_audit_utils.h` 定义 `enum class ObAuditTrailType{...}` 与 `class ObSecurityAuditUtils final`(位于 `namespace oceanbase::sql`),`src/sql/monitor/ob_security_audit.h` 定义审计数据载体 `class ObSecurityAuditData`,schema 侧 `src/share/schema/ob_security_audit_mgr.h` 定义规则管理器 `class ObSAuditMgr`(配合 `enum ObSAuditOperationType`、`enum ObSAuditType`)。调用链入口在 `src/sql/ob_sql.cpp`、`src/sql/ob_result_set.cpp` 中以 `ObSecurityAuditUtils::handle_security_audit` / `ObSecurityAuditUtils::check_allow_audit` / `ObSecurityAuditUtils::get_audit_units` 串联(定义见 `ob_security_audit_utils_os.cpp`),`enum AuditActionType`(`ob_audit_action_type.h`)枚举审计动作类型。4.4.x 分支同名类/枚举一致。

按 OceanBase 官方对 Oracle 兼容性的说明：支持标准 statement auditing 与 object auditing，不支持 unified auditing 与 Fine-Grained Auditing(FGA),privilege auditing 与 network auditing 也不支持；审计结果可存文件或内部表。此外还有审计 operation/action 元数据虚表(`ob_all_virtual_audit_operation.h`、`ob_all_virtual_audit_action.h`)以及暴露主密钥版本的 `ob_all_virtual_master_key_version_info.h`——该文件定义虚表类 `class ObAllVirtualMasterKeyVersionInfo : public common::ObVirtualTableScannerIterator`(对应 TDE 密钥轮换可观测性)。

### 21.3.3 加密：TDE + 传输加密

OceanBase 的静态加密是透明加密(TDE)。官方 Data storage encryption 文档说明 TDE 加密 baseline data 与 clogs 等磁盘静态数据，数据落盘加密、读入内存后是明文。加密算法定义在 `src/share/ob_encryption_util.h`:`enum ObCipherOpMode`(行 42)定义大量模式，AES 系列(`ob_aes_128_ecb=1`、`ob_aes_256_ecb=3`、`ob_aes_256_cbc=6`、`ob_aes_256_gcm=23` 等)与 SM4 国密系列(`ob_sm4_cbc=24`、`ob_sm4_ecb=25`、`ob_sm4_ctr=28`、`ob_sm4_gcm=29` 等)并存；`class ObEncryptionUtil`(行 129)提供 `encrypt_data`/`decrypt_data`、`is_aes_encryption`/`is_sm4_encryption`、`get_key_length`；引擎枚举含 `OB_AES_ENGINE=2`、`OB_SM4_ENGINE=3`。默认表密钥算法常量(行 151-157):`DEFAULT_TABLE_KEY_AES_ENCRYPT_ALGORITHM = ob_aes_128_ecb`、`SYS_DATA_ENCRYPT_ALGORITHM = ob_aes_128_cbc`。4.3.5 同名文件存在。

两级密钥架构(经官方 Cloud 文档、源码 keystore schema、第三方实测三源交叉验证)：每个租户有一把 master key，用于加密各加密表空间的 tablespace key;tablespace key 才直接加密数据。master key 经 keystore schema 持久化(`ob_keystore_sql_service.cpp` 把 `master_key`/`master_key_id` 列写入内部表)，字段 `master_key_`、`master_key_id_` 在 `ob_schema_struct.h` 中真实存在，getter 为 `get_master_key()`/`get_master_key_id()`/`get_master_key_str()`(行 7211-7226 一带);表空间加密绑定 `master_key_id`(`ob_tablespace_sql_service.cpp`)。外部 KMS 配置项常量真实存在于 `src/share/config/ob_server_config.h`:`EXTERNAL_KMS_INFO = "external_kms_info"`、`SSL_EXTERNAL_KMS_INFO = "ssl_external_kms_info"`、`TDE_METHOD = "tde_method"`(行 50-52),4.4.x 分支同名常量一致。底层加密接口 `ob_micro_block_encryption.h`、`ob_clog_encrypter.h` 存在，但受 `OB_BUILD_TDE_SECURITY` 宏保护——官方文档明确 Community Edition 不支持 TDE，源码存在不能推出 CE 默认支持(见 §21.8)。

启用方式(第三方实测，MySQL 租户):`ALTER SYSTEM SET tde_method='internal';`(`tde_method` 取 `none`/`internal`)→ `ALTER INSTANCE ROTATE INNODB MASTER KEY;` 生成主密钥 → `CREATE TABLESPACE xx encryption='y';`(默认 aes-256)→ 建表时 `TABLESPACE xx`；可经 `V$OB_ENCRYPTED_TABLES` 视图查看加密状态。官方 Cloud 文档另注：Oracle 兼容租户支持 AES-256/128/192 与 SM4-CBC,MySQL 租户主密钥类型仅 AES-256;Cloud TDE 覆盖范围包括 tablespace files、transaction log files 与 database backups，并支持服务密钥或云厂商 KMS。

传输加密：除静态加密外，OceanBase 还支持客户端/驱动到 OBServer/OBProxy 的 SSL/TLS，以及 OBServer 节点间的 OB-RPC 加密，官方 Data transmission encryption 文档区分 MySQL protocol 与 OB-RPC protocol 两条路径，推荐 TLSv1.2+，并支持国密 RPC 传输加密与 TLS 免密登录。TLS 解决链路窃听与中间人风险，但不替代 SQL 权限、Tenant 边界、审计规则或密钥管理。需注意一个运维陷阱：云上 SSL 链路加密对直连地址(绕过代理)不生效，且一旦开启不可关闭(阿里云 ApsaraDB for OceanBase 文档明确)。

产品形态边界(关键)：透明加密(TDE)、Oracle 兼容模式、security audit 在 CE 与企业版/Cloud 的可用性差异以官方文档为准。源码层无法判定 license/edition 边界，本章不据源码断言「企业版独有」；CE checkout 中存在 TDE/审计相关 schema/算法代码，但受编译宏保护、官方文档明确 CE 不支持，不代表 CE 在功能/license 上开放该能力。

--- 〔文献[8-10,12]〕

## 21.4 核心差异对比

**表 21-3　安全/权限/审计/加密:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB | OceanBase | 影响 |
|---|---|---|---|
| 安全边界主轴 | SQL user@host、role、动态权限、SEM、TLS、TiKV/BR/KMS、Cloud IAM 的组合 | Tenant、sys 租户、MySQL/Oracle 双模权限、ODP/OBServer、TDE/KMS | TiDB 更依赖组件与外部 IAM 组合；OceanBase 必须重点审计 sys 租户 |
| 权限模型基底 | MySQL 5.7 静态权限 + SQL Roles + 动态权限(8.5.x 默认 16 项 / 7.5.x 默认 15 项，可插件扩展) | 租户级 + MySQL/Oracle 双模(`ObPrivilegeCheck` 分流);bit-shift `ObPrivSet` | OB 多一层租户边界与 Oracle 角色体系；OceanBase 测试不能只按 MySQL `GRANT` 覆盖 |
| 特权拆分机制 | SEM 使 `SUPER` 不可越过 `RESTRICTED_*`，隐藏诊断对象 | sys 租户特权 + `OB_PRIV_ALTER_TENANT`/`ALTER_SYSTEM` 等位 | TiDB 用 SEM 服务 DBaaS 隔离；OB 用租户层级天然隔离 user 与 sys |
| 认证方式 | native/sha2/SM3/auth_socket/JWT token/LDAP simple+SASL;`REQUIRE SSL`/`X509` | MySQL/Oracle 标准认证 + SSL 免密 + 国密 RPC | TiDB 认证插件谱系更广(含 JWT 云免密)；两者都可强制用户级 TLS |
| 审计落点 | 开源仅 SPI(`AuditManifest`)；企业版插件 / Cloud Dedicated 落客户云存储 | 内核内置：`GV$OB_SQL_AUDIT`(内存，诊断)+ Security Audit(`AUDIT` 策略，可落表/文件，企业能力 CE 不支持) | OB「SQL 即可观测」、零外部依赖但内存视图保留短；TiDB 审计强依赖 edition/形态 |
| 静态加密承担者 | TiKV component(TiDB Server 不做 at-rest);PD experimental;BR client-side 加密 8.5.0 GA | OBServer 内置 TDE(表空间级，CE 不支持);Cloud TDE 可覆盖备份 | TiDB 加密跨组件分层、概念易错；OB 加密一体化、表空间粒度 |
| 加密算法 | AES128/192/256-CTR + SM4-CTR(v6.3+) | AES-128/192/256(ECB/CBC/GCM 等)+ SM4(CBC/ECB/CTR/GCM) | 两者都支持国密；OB cipher 模式更全(含 GCM 认证模式) |
| 密钥体系 | 信封加密：master key(File/AWS/Azure/GCP KMS)→ data key(AES256-GCM 包裹),7 天自动轮换 | 两级：tenant master key → tablespace key;keystore 持久化 + 外部 KMS | 概念同构；TiDB 数据密钥自动轮换周期默认值明确(7d),OB 主密钥手动 `ROTATE` |
| 传输加密 | client↔TiDB TLS + 组件互信 TLS;`AutoTLS` 默认 false | MySQL protocol 与 OB-RPC 均可 SSL(TLSv1.2+)+ 国密 RPC | 都需显式配置；OB 云上「直连绕代理不生效」是运维陷阱 |
| 产品形态依赖度 | 极高：审计/TDE/SEM 在开源/企业/Cloud Starter/Essential/Dedicated 差异大 | 高：TDE/security audit/Oracle 兼容在 CE/企业/Cloud 差异(CE 不支持 TDE/security audit) | 合规评估必须先锁形态，不能按品牌一概而论 |

这张表的读法不是给某一方打总分，而是定位安全责任边界。TiDB 更像由多个组件拼成的安全链：TiDB Server 负责认证授权，TiKV 负责静态加密，BR 负责备份加密，Cloud 或外部平台负责 IAM/KMS/对象存储，任一环节松动整体结论都要降级。OceanBase 更强调租户内聚：Tenant、LS、资源管理、SQL Audit 与 TDE 围绕同一租户边界展开，但 sys 租户与企业版能力是必须单独标出的上层控制面。还需谨记一条写作边界：权限模型差异不等于安全强弱差异——真实攻击往往走混合路径(例如低权限 SQL 账号配合对象存储误授权读取备份，或普通租户无法越权但 sys 租户运维账号泄露导致跨租户可见)，因此可以推测，比较结论应聚焦「需要检查哪些控制点」，而不是抽象判断哪个模型更安全。

---

## 21.5 正常路径图

图 21-1 展示一条「带 TLS + 认证 + 权限校验 + 审计 + 落盘加密 + 备份加密」的完整正常路径在两系统中的走向。

![[f21_1.svg]]

**图 21-1　安全/权限/审计/加密正常读写/调度路径**

正常路径的共同结构是：认证决定谁进入，权限决定能做什么，TLS 保护传输，TDE/备份加密保护离线介质，审计负责事后重建行为。关键差异在控制点分布：TiDB 的认证/权限/审计在无状态 SQL 层，落盘加密在下游 TiKV、备份加密在 BR(跨组件、跨进程)，还叠加 Cloud IAM;OceanBase 的全链路在 OBServer 一体化完成(经 OBProxy 透传 SSL)，审计直接写内核，控制点更多围绕 Tenant、sys 租户、LS 与 key service。

正常路径有四类可验证的不变量。身份不变量：连接身份、证书身份、数据库用户与运维/Cloud 身份不能混淆，尤其不能让同一个共享账号同时承担应用读写、备份恢复与集群管理。授权不变量：DDL、DCL、备份、恢复、查询系统视图、查询审计表必须分别授权，不能由单个泛化管理员权限兜底。密钥不变量：data key、master key、previous master key、KMS 权限与备份密钥材料必须能解释清楚归属与轮换状态。审计不变量：关键路径至少能重建「谁、何时、从哪里、对哪个租户或库表、执行了什么动作、结果如何」。这些不变量一旦缺失，安全功能就容易只剩「配置项存在」。据此推测，会出现以下几类情形：TLS 打开但客户端不校验证书，连接仍可被中间人替换；TDE 打开但 KMS 与对象存储权限授予同一过宽角色，离线备份仍可被旁路解密；审计打开但审计存储与 DBA 共享权限，审计日志仍可能被删改。

---

## 21.6 故障 / 异常路径图

图 21-2 归纳安全相关的六类典型异常路径及其后果。

![[f21_2.svg]]

**图 21-2　安全/权限/审计/加密故障/异常路径**

这些故障路径揭示一个共性教训：配置项显示「已开启」不等于安全实际生效。密钥可达性、审计落点持久性、TLS 真实链路覆盖，都必须独立验证。安全故障也不只有「拒绝服务」一种形态：证书失效会导致不可用，证书校验缺失会导致中间人风险；权限过紧会阻断恢复，权限过宽会越权；审计过窄会漏报，审计过宽会消耗资源并记录敏感 SQL;KMS 失效会阻断解密，KMS 过宽会降低加密价值。其中主密钥可达性是信封/两级加密体系的最大可靠性风险——master key 在 KMS 或 File，一旦不可达加密数据全部不可读，一旦丢失则永久不可恢复(BR 文档对备份密钥的措辞同理)，这要求 KMS 多 region 冗余、File 后端异地备份、轮换前验证。

处置层面要分清安全故障与可用性故障：KMS 不可达可能是安全策略收紧、也可能是网络故障；审计 bucket 不可写可能导致漏审、也可能业务继续运行但取证失败；证书轮换失败可能保护了安全性但牺牲了连接可用性。因此演练报告应同时记录业务影响、数据保密性影响、取证完整性影响与恢复步骤，据此判断系统是 fail open、fail closed 还是进入了部分可用状态。可以推测，若只能记录「测试失败」，就无法做出这一判断。

---

## 21.7 性能、可靠性、运维影响

**表 21-4　性能、可靠性、运维影响**

| 能力 | 收益 | 代价 | 验证重点 |
|---|---|---|---|
| TLS / 证书 | 防窃听、防中间人、用户级 TLS 要求 | 握手与证书生命周期成本、CPU | CA 校验、证书轮换、旧客户端失败 |
| 权限 / SEM / Tenant | 最小权限、管理员降权、租户边界 | 权限矩阵复杂 | 撤权即时性、越权、sys 租户审计 |
| 审计 | 合规取证、异常追踪 | 写入成本、SQL 文本敏感性 | 漏报、重复、redaction、审计存储故障 |
| 静态加密 / 备份加密 | 介质丢失时降低泄露 | 密钥管理复杂、恢复依赖 KMS、加解密算力 | key 轮换、KMS 不可用、密文恢复 |

延迟与吞吐：静态加密对吞吐有真实代价。TiKV 官方明确 SM4 在最坏情况下吞吐下降 50%~80%，加大 block cache 可降到约 10%;AES-CTR 因有硬件指令(AES-NI)代价小得多。OceanBase 官方对 TDE 的说法是「update 场景性能不受影响，其他场景轻微损耗」，但这属厂商自述、缺独立 benchmark，故证据有限，宜谨慎引用；官方同时强调内存中数据仍为明文，说明 TDE 保护的是静态介质而非运行时访问。传输 TLS 在两系统都主要影响连接建立与 CPU，因此都建议仅在需要时开启。具体 Prometheus metric、内部参数与诊断字段名未逐项双重查证，需进一步查证后再引用，不编造。

可靠性：安全控制会改变故障域。启用 TLS 后 CA、证书链与客户端 trust store 变成可用性依赖；启用静态加密后 KMS 与 key cache 变成恢复依赖，主密钥丢失即数据永久不可恢复(见 §21.6)；启用审计后审计服务与日志目的地变成取证依赖。审计的可靠性陷阱在「保留」：OceanBase `GV$OB_SQL_AUDIT` 是内存环形缓冲、取证窗口短，真正合规留痕须走 Security Audit 落表/文件；TiDB 开源无审计本体，合规场景必须上企业版/Cloud Dedicated。比较两套系统时，不能只比内核是否可用，还要比这些外部依赖失效时的降级语义、告警能力与恢复步骤(对应 §21.6 的 fail open / fail closed 判别)。

扩展性与运维复杂度：TiDB 加密跨组件(TiKV/TiFlash/PD/BR)分别配置，运维面更分散但每个组件职责清晰，SEM 让 DBaaS 能安全地把 admin 权限给租户、降低多租户运维风险。OceanBase 一体化，TDE/审计/权限在 OBServer 内、SQL 即可诊断观测、零外部依赖，但代价是 sys 租户成为高价值单点目标，且内存审计视图需配套 Security Audit 才能长期留痕。运维代价主要来自轮换与例外处理：证书会过期、KMS key 会轮换、审计存储会扩容、备份会跨 region 复制、临时故障时会有人请求放宽权限。较好的验收方式是把轮换、回滚、紧急恢复、权限收回写成自动化用例——临时账号创建后必须过期、临时 KMS policy 必须回收、审计存储扩容失败必须告警、恢复演练结束后必须证明测试密钥与测试备份已隔离或销毁。密钥轮换是合规高频检查点：TiDB data key 默认 7 天自动轮换(master key 手动),OceanBase master key 经 `ALTER INSTANCE ROTATE` 手动轮换，轮换后旧表空间仍可读(版本元数据可经虚表观测)。可以推测，两者都必须做到可观测、可回滚验证。

---

## 21.8 反例与代价

- 加密不是「全表全程密文」。TiKV/OceanBase 的 TDE 主要保护落盘静态数据，不覆盖 core dump、info 日志、运行时内存与网络传输中的明文(那是 TLS 的事)。CryptDB 论文展示真正的端到端加密查询处理需要 proxy、SQL-aware encryption 与泄露/性能取舍，与 TDE 不是同一类能力——TDE 不能阻止合法 SQL 或运行时明文访问，也不能作为防 DBA 查询、防应用泄露或防 SQL 注入的证据。若误以为开了 at-rest 加密即可高枕无忧，而忽略关闭 core dump、开启 log redaction、配置 TLS，数据仍可能从日志或内存转储泄露，这是最常见的「半加密」错觉。
- 审计不能替代权限，且审计本身有隐私代价。审计回答「谁做了什么」，阻止越权仍靠权限、网络与密钥控制；审计日志常包含 SQL text、客户端地址、用户名、库表名甚至字面量，会成为第二份敏感数据集，需要 redaction、独立只读账号与访问控制。把 OceanBase `GV$OB_SQL_AUDIT` 当合规留痕是典型误用——它会被淘汰、本质是诊断视图，合规必须走 `AUDIT` 策略 + audit trail 落表/文件。
- SEM 有其代价。开启 SEM 会隐藏 `metrics_schema`、大量 `information_schema.cluster_*` 诊断表、一批调优 sysvar，意味着自助诊断与调优能力被削弱——运维人员若不持有对应 `RESTRICTED_*` 权限，排障时会发现「视图查不到、变量看不见」。SEM 适合 DBaaS 多租户隔离，不适合需要深度自助调优的自建场景盲目开启。
- 源码存在代码不等于产品可用。OceanBase CE 仓库有 TDE(`ob_micro_block_encryption.h`/`ob_clog_encrypter.h`,`OB_BUILD_TDE_SECURITY`)与 security audit(`ob_security_audit.h`/`ob_security_audit_mgr.cpp`,`OB_BUILD_AUDIT_SECURITY`)代码路径，但受编译宏保护，且官方文档明确 CE 不支持 TDE/security audit;TiDB 开源仓库只有审计 SPI 没有审计本体。edition 能力一律以官方文档为准，不能据源码断言企业版独有功能清单。
- 「加密备份」不等于「可恢复备份」。若备份密钥、KMS 权限、对象存储版本与恢复账号没有联合演练，真正恢复时可能因缺 key、缺权限或 key policy 已变更而失败。据此推测：密文可读但 KMS 不可用会导致恢复失败，KMS 权限过宽则削弱备份加密意义。因此必须测试「密文可读但 key 不可用」的失败路径。
- 国密落盘加密的性能代价。SM4 在 TiKV 上最坏 50%~80% 吞吐下降(见 §21.7)，等保合规要求国密时，这是必须提前压测的性能代价点，不能上线后才发现。
- 多租户的 sys 特权代价与「默认开启」陷阱。OceanBase sys 租户对 user 租户有跨租户特权，这是隔离的代价——sys 租户被攻破则全集群暴露；「租户隔离」不等于「无跨租户可见性」，诊断视图、运维命令与审计查询必须按租户身份分别验证(见 §21.3.1)。TiDB 没有这层超级租户，但代价是没有原生的租户级硬隔离(Resource Control 是软隔离，详见第 17 章)。此外「默认开启」尤其危险：TiDB Cloud 审计文档明确有默认关闭、申请或 Beta 等形态差异，OceanBase CE 文档明确不支持部分企业安全能力，未查到默认开启证据时本章不写默认开启。若只需传输安全而无需落盘加密，两系统都可只开 TLS 省去加密 CPU 代价；若合规强制 at-rest + 审计 + 国密，则两系统都要付出性能 + 运维 + 形态(可能需企业版/Cloud)的综合代价，不存在「零成本合规」。

--- 〔文献[11]〕

## 21.9 测试开发视角的验证点

验收断言应尽量写成「允许 / 拒绝 / 记录 / 可恢复」的四元组：允许断言说明合法用户在合法通道下能完成任务；拒绝断言说明缺少任一关键权限时必须失败；记录断言说明成功与失败都能在审计或外部日志中定位；可恢复断言说明加密数据在合法密钥链完整时能恢复、在关键密钥缺失时不能恢复。可以推测，这个四元组比单纯「功能成功」更贴近安全测试，因为安全能力通常同时要求成功路径可用、失败路径可控。

可测试的功能场景：

- 权限最小化：授予 `BACKUP_ADMIN`+`RESTORE_ADMIN` 后验证该账号能备份但不能 `DROP TABLE`(TiDB)；授予对象级 `SELECT ON db.t` 后验证不能跨库(OceanBase)。
- SEM 越权：开 SEM 后用持 `SUPER` 的账号尝试操作 `RESTRICTED_USER_ADMIN` 用户、查询 `metrics_schema`，应全部被拒(对应 `IsRestrictedPrivilege`/`IsInvisibleSchema`)。
- 双模权限：OceanBase 同集群下 MySQL 租户与 Oracle 租户权限语义不同，验证 `GRANT DBA`(Oracle)与 `GRANT SUPER`(MySQL)不可混用。
- 租户可见性：以业务租户 A、业务租户 B、sys 租户三类身份分别查询 `GV$OB_SQL_AUDIT`，普通租户不得看到其他租户记录，sys 租户的跨租户查询必须被记录。

可注入的失效模式：

- KMS / 主密钥：断开 TiKV/OBServer 到 KMS 的网络，验证启动/解密失败行为与告警；执行轮换后立即读历史数据，验证旧 data key/tablespace key 仍可解密；故意用错 master key 验证拒绝启动。
- 备份链路：恢复账号有对象存储读权限但无 KMS 解密权限必须失败；KMS key policy 允许解密但账号无 `RESTORE_ADMIN` 必须失败；撤销 `BACKUP_ADMIN` 后复用旧连接执行备份验证被拒。
- 审计漏报：TiDB 不装审计插件时执行敏感 DML，确认确实无留痕(反向验证形态陷阱)；让 OceanBase `GV$OB_SQL_AUDIT` 内存打满，验证历史记录被淘汰；审计 filter 排除 DDL/DCL 或审计 bucket 不可写时验证降级语义。
- TLS 真实性：抓包验证链路是否真为密文(而非仅看配置「已开启」)；以非 TLS、TLS 但不校验 CA、TLS 且校验 CA 三类连接验证用户级 TLS 要求；OceanBase 云上故意走直连地址验证 SSL 不生效；证书过期后验证自动化运维脚本是否静默回退到非 TLS。

最小用例集的组织方式：TiDB 侧创建普通业务账号、备份账号、恢复账号与只读审计账号，分别授予 role、动态权限与对象存储/KMS 权限，跑通三类 TLS 连接、SEM 下 `SUPER` 受限验证、一次加密 full backup 与 log backup，再通过撤销 KMS 解密权限验证恢复失败。OceanBase 侧以租户维度组织：业务租户 A/B、sys 租户三类身份分别执行 SQL Audit 查询，启用 TLS 后分别验证客户端协议与 OB-RPC 通道，在支持 TDE 的形态中验证新表、历史数据、clog、备份与恢复的密钥链，在 CE 形态中则明确记录 TDE/security audit 不作为默认能力测试——这样既避免把企业版能力误写到 CE，也避免把 sys 租户风险埋在「租户隔离」口号里。据此推测，安全测试还应纳入数据最小化：测试数据不直接使用生产敏感值，日志导出与样例截图脱敏，审计读取账号只读且独立于业务 DBA。

关键观测指标 / 视图 / 命令(已查证，不编造):

- TiDB:`tidb_audit_redact_log` sysvar(审计插件安装，SEM 下隐藏)、`tidb_redact_log`(日志脱敏，ON/MARKER)；权限可经 `SHOW GRANTS`、`information_schema` 权限表查询；TiKV 加密配置 `data-key-rotation-period`、`data_encryption_method`;`tikv-ctl` 可做加密相关诊断；可检查 Cloud audit/console audit、TLS 连接状态、BR 日志、KMS 与对象存储审计日志。
- OceanBase:`GV$OB_SQL_AUDIT`(SQL 级诊断/可观测，4.0+ 名)、`DBA_AUDIT_TRAIL`/`USER_AUDIT_TRAIL`/`DBA_AUDIT_OBJECT`/`DBA_AUDIT_SESSION`(安全审计 trail 视图)、`V$OB_ENCRYPTED_TABLES`(加密表状态，第三方实测确认)、`ob_all_virtual_master_key_version_info`(主密钥版本虚表，源码确认类 `ObAllVirtualMasterKeyVersionInfo`)；参数 `tde_method`、`audit_trail`。未查证到的 metric/视图/参数名不在此列。

关键压测指标：开 SM4 加密前后的写吞吐/P99(验证 50%~80% 吞吐下降)、开 TLS 前后连接建立延迟与 CPU、审计开启后的写放大与延迟尾部。

---

## 21.10 容易误解点

1. 「TiDB Server 负责数据加密」——错。TiDB Server 只做传输 TLS + 算子溢写临时文件(aes128-ctr)；静态数据落盘加密由 TiKV 承担(`components/encryption/`)，备份加密由 BR 承担，TiFlash 另有自己的加密，PD 加密为 experimental。把加密误认为单层、单组件，会导致合规架构图画错、密钥管理责任划错。
2. 「开了 at-rest 加密就安全了」——不完整。at-rest 只保护落盘文件；传输需要 TLS、日志需要 redaction、core dump 需要关闭，三者正交，缺一面就有泄露面。「配置项已开启」也不等于「实际生效」(OceanBase 云上直连绕代理 SSL 失效、TiDB `AutoTLS` 默认 false)。
3. 「OceanBase 的 `GV$OB_SQL_AUDIT` 就是合规审计」——错。它是内存环形缓冲的诊断视图，会被淘汰、保留窗口短(第 19 章)。合规留痕必须用基于 `AUDIT`/`NOAUDIT` 的 Security Audit 子系统落表/文件，且该子系统是企业能力、CE 不支持；两者是独立子系统，不可混淆。
4. 「OceanBase 有 Tenant 就不会跨租户可见」——错。普通租户与 sys 租户的可见性不同，sys 租户能查询其他租户的 `GV$OB_SQL_AUDIT`、承担集群级运维，是最高风险的审计对象之一；诊断视图、运维命令与审计查询必须按租户身份分别验证。
5. 「企业版功能可以从开源源码推断」——禁止。源码层无法判定 license/edition 边界，CE checkout 里存在 TDE/security audit 代码但受编译宏保护、官方文档明确 CE 不支持；TiDB 开源仓库只有审计 SPI 没有审计本体。edition 能力一律以官方文档为准。
6. 「BR 加密备份成功就等于恢复演练通过」——错。恢复需要 SQL 权限、对象存储权限、KMS 权限与兼容的密钥材料同时成立；源码路径里出现的函数/类名也不等于可作为用户可配置参数引用，未找到公开文档支撑的参数、函数、指标、视图名均标「需进一步查证」。

---

## 21.11 本章结论

1. 权限模型：TiDB 是「MySQL 静态位 + SQL Roles + 动态权限 + SEM」的单方言深度模型，OceanBase 是「租户级 + MySQL/Oracle 双模」的分层模型。TiDB 的动态权限可插件扩展(默认条目数随版本变化：8.5.x 16 项、7.5.x 15 项，差 `RESOURCE_GROUP_USER`)、SEM 使 `SUPER` 不可越过 `RESTRICTED_*`;OceanBase 用 sys 租户层级与 `ObPrivilegeCheck` 双模分流实现隔离。这是工程取舍，无优劣，且权限模型差异不等于安全强弱差异。

2. 审计是本章产品形态差异最大的能力。TiDB 开源内核仅提供审计 SPI(`AuditManifest`，无本体)，审计落地依赖企业版插件或 Cloud Dedicated(写客户云存储),Starter 不支持、Essential 为 Beta;OceanBase 内核内置两套独立审计——`GV$OB_SQL_AUDIT`(内存诊断、保留短)与 Security Audit(`AUDIT` 策略、可落表，但属企业能力、CE 不支持)。

3. 加密职责放置点不同但密钥体系同构。TiDB 静态加密在 TiKV(信封加密，master→data key，默认 7 天轮换，File/AWS/Azure/GCP KMS),TiDB Server 只做 TLS + 溢写文件，BR client-side 备份加密 8.5.0 GA;OceanBase TDE 在 OBServer 内一体化(两级 tenant master key→tablespace key,keystore + 外部 KMS,CE 不支持)。两者都支持国密 SM4，但 SM4 落盘在 TiKV 上有 50%~80% 最坏吞吐代价。

4. 「单点」必须按维度拆解，安全边界亦然。OceanBase sys 租户是隔离的保障者也是最高价值攻击目标(逻辑特权与跨租户可见性集中，非物理单副本失效);TiDB 无超级租户但 PD/TSO 仍是逻辑中心化(详见第 13 章)。需要推测性地区分的是：安全语境下「单点」指特权集中维度，与可用性单点并不相同。

5. 合规/金融场景的核心动作是「先锁产品形态、再验实际生效」。审计、TDE、Oracle 兼容、国密的可用性随开源/企业/Cloud/CE 剧烈变化，不能按品牌一概而论；源码存在接口或宏保护代码不等于目标发行版默认可用；且「配置已开启」需独立验证「实际生效」(链路抓包、密钥可达、审计落点持久)。

6. 加密的最大可靠性风险是主密钥可达性而非算法强度。master key 丢失即数据永久不可恢复(BR 文档同理),KMS 多 region 冗余、轮换前验证、密钥版本可观测(OceanBase `master_key_version` 虚表)是底线；TDE/备份加密解决离线介质风险，不解决合法用户越权、运行时明文、审计日志泄露或 Cloud 管理面过宽问题。

---

## 21.12 参考文献

[1] Privilege Management / Security Compatibility with MySQL | TiDB Docs. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/privilege-management/.
 （支撑:§21.2.1/§21.2.2 静态权限、SQL Roles、动态权限完整列表、SUPER 语义、认证插件谱系与版本归属、密码策略账号级特性。）
[2] TiDB Security Enhanced Mode 设计文档(pingcap/tidb docs/design/2021-03-09-security-enhanced-mode.md). 官方设计文档[EB/OL]. https://github.com/pingcap/tidb/blob/master/docs/design/2021-03-09-security-enhanced-mode.md.
 （支撑:§21.2.1 SEM 的 DBaaS 动机、RESTRICTED_* 引入、隐藏对象清单与 enable-sem 配置。）
[3] Enable TLS Between TiDB Clients and Servers | TiDB Docs. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/enable-tls-between-clients-and-servers/.
 （支撑:§21.2.2/§21.9 TiDB TLS、REQUIRE SSL/REQUIRE X509 与未配 CA 时的中间人风险。）
[4] Encryption at Rest / TiDB 8.5.0 Release Notes | TiDB Docs. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/encryption-at-rest/.
 （支撑:§21.2.4/§21.8 TiKV 信封加密、算法、KMS 后端、7 天轮换、SM4 性能代价、PD experimental、TiFlash 共享主密钥，以及 BR full/log backup client-si）
[5] TiDB Cloud Dedicated / Essential Database Audit Logging | TiDB Docs. 官方文档[EB/OL]. https://docs.pingcap.com/tidbcloud/tidb-cloud-auditing/.
 （支撑:§21.2.3/§21.4 Cloud 审计按 tier 分层(Dedicated 支持、默认关闭、需申请；Starter 不支持；Essential Beta + redaction + 日志可能乱序)。）
[6] pingcap/tidb 仓库(release-8.5 @ 67b4876bd57b)— pkg/util/sem/sem.go、pkg/privilege/privileges/privileges.go、pkg/plugin/audit.go、pkg/config/config.go、br/pkg/encryption/manager.go、pkg/parser/mysql/privs.go/const.go. 源码[EB/OL]. https://github.com/pingcap/tidb/tree/release-8.5.
 （支撑:§21.2 SEM 函数行为、动态权限默认条目(8.5 为 16 项；7.5 为 15 项，差 RESOURCE_GROUP_USER)、AuditManifest SPI、Security 配置结构、BR 加密）
[7] tikv/tikv 仓库(release-8.5 @ 1f8a140b6d46)— components/encryption/src/(crypter.rs/config.rs/manager/mod.rs/master_key/). 源码[EB/OL]. https://github.com/tikv/tikv/tree/release-8.5/components/encryption.
 （支撑:§21.2.4 TiKV 加密算法、默认配置、信封加密、KMS/File 后端、data key 轮换与 previous master key;commit-pinned。）
[8] oceanbase/oceanbase 仓库(v4.2.5_CE @ e7c676806fda)— src/share/ob_encryption_util.h、src/share/schema/ob_priv_type.h、src/sql/privilege_check/、src/observer/virtual_table/ob_gv_sql_audit.*、src/sql/monitor/ob_security_audit.h、src/storage/blocksstable/ob_micro_block_encryption.h. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE.
 （支撑:§21.3 权限位、双模权限检查、ObCipherOpMode 算法、SQL audit 虚表(sys 租户跨租户提取 tenant id)、Security Audit 子系统真实命名(ObSecurityAud）
[9] OceanBase Database Security Overview / Data Transmission Encryption / Data Storage Encryption / GV$OB_SQL_AUDIT | OceanBase Docs. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001971084.
 （支撑:§21.3 OceanBase 认证/访问控制/数据加密/security audit 与 CE 不支持 TDE/security audit、MySQL protocol 与 OB-RPC 传输加密、TDE 两级密钥与）
[10] Transparent Data Encryption (TDE) | OceanBase Cloud Docs / SSL Link Encryption | ApsaraDB for OceanBase. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-cloud-10000000002267258.
 （支撑:§21.3.3 Cloud TDE 两级密钥、MySQL/Oracle 租户算法差异、Cloud TDE 覆盖备份、KMS 集成、传输加密启用方式与「直连绕代理 SSL 不生效」运维陷阱。）
[11] CryptDB: Protecting Confidentiality with Encrypted Query Processing(SOSP 2011). 论文[EB/OL]. https://people.csail.mit.edu/nickolai/papers/raluca-cryptdb.pdf.
 （支撑:§21.8 端到端加密查询处理与 TDE 不是同一类能力、需要额外架构与泄露/性能取舍。）
[12] 技术分享 | OceanBase 安全审计之透明加密(爱可生开源社区). 第三方解读[EB/OL]. https://opensource.actionsky.com/技术分享-oceanbase-安全审计之透明加密/.
 （支撑:§21.3.3 TDE 启用 SQL 语法(tde_method/ROTATE ... MASTER KEY/CREATE TABLESPACE encryption='y')与 V$OB_ENCRYPTE）
## 21.13 信息可信度自评

- **官方文档明确**:TiDB 动态权限条目数与版本归属(8.5.x 16 项 / 7.5.x 15 项)、SEM 隐藏对象与 `enable-sem`、`REQUIRE SSL/X509`、TiKV 加密算法/信封加密/7 天轮换/SM4 性能代价、BR 备份加密 8.5.0 GA、认证插件谱系、密码策略、TiDB Cloud 审计 tier 分层、OceanBase 权限三级/双模、传输加密双协议、TDE 两级密钥与内存明文、CE 不支持 TDE/security audit、SSL 启用方式——均有官方文档支撑。
- **源码级信息**:TiDB SEM 函数(`Enable`/`IsInvisible*`/`IsRestrictedPrivilege`)、动态权限列表、`AuditManifest` SPI、Security 配置结构、BR 加密两路径(release-8.5 @ 67b4876bd57b);TiKV 加密 component(release-8.5 @ 1f8a140b6d46);OceanBase `ObPrivilegeCheck`、`ObCipherOpMode`、`ob_priv_type.h` 权限位、`GV$OB_SQL_AUDIT` 列枚举与 sys 租户跨租户提取、Security Audit/TDE 编译宏保护(v4.2.5_CE @ e7c676806fda)——均来自真实 checkout。OceanBase 安全审计子系统的真实命名已在本 checkout 直接核对(`ObSecurityAuditUtils`/`ObAuditTrailType`/`ObSecurityAuditData`/`ObSAuditMgr`/`handle_security_audit`/`check_allow_audit`/`get_audit_units`),master key getter 为 `get_master_key()`/`get_master_key_id()`/`get_master_key_str()`,外部 KMS 配置常量为 `EXTERNAL_KMS_INFO="external_kms_info"`/`SSL_EXTERNAL_KMS_INFO="ssl_external_kms_info"`(此前误记为脱敏占位，本轮经源码 grep 升级为确定符号，4.4.x 分支一致)。
- **论文**:CryptDB 仅用于区分 TDE 与加密查询处理，不用于宣称 TiDB 或 OceanBase 已实现 CryptDB 类方案，标。
- **第三方解读(占比约 8%)**：爱可生开源社区文章提供 TDE 启用 SQL 实操，作为 en.oceanbase.com 持续 HTTP 429 时的交叉验证源，版本/参数细节以官方文档为准。
- **工程推测**：安全语境下「sys 租户为高价值特权目标」「PD/TSO 逻辑中心化」「四元组验收框架」「fail open/fail closed 判别」属基于公开架构的推断，标（推测）。
- **反查证发现(动态权限版本数值错配，已修正)**：比对 pingcap/tidb 各 tag `dynamicPrivs` 切片确认 8.5.x 为 16 项(含 `RESOURCE_GROUP_USER`)、7.5.x 为 15 项(无 `RESOURCE_GROUP_USER`)，正文与对比表、结论均按版本分列，不再笼统称「16 项」。
- **反查证发现(CE 边界与编译宏)**:OceanBase CE 仓库存在 TDE/security audit 代码路径，但受 `OB_BUILD_TDE_SECURITY`/`OB_BUILD_AUDIT_SECURITY` 宏保护、官方文档明确 CE 不支持；源码文件名 `ob_gv_sql_audit.cpp` 与官方外部视图名 `GV$OB_SQL_AUDIT` 不同，正文按官方视图名书写。TiKV SM4 性能代价(50%~80%)在官方 encryption-at-rest 文档单一来源给出，未找到独立第二来源量化，标（证据有限）。
- **版本核查备注**：本章版本严格按基线书写(TiDB 8.5.x / 7.5.x,OceanBase v4.2.5_CE,4.3.5/4.4.x 仅作版本注记)，源码引用全部 commit-pinned 到统一基线；OceanBase 自建 4.4.x CE 的备份加密命令、Prometheus metric 与部分内部统计项未充分查证，统一标「需进一步查证」，不编造参数名。en.oceanbase.com 本轮持续返回 HTTP 429，相关官方内容改以搜索摘要 + 源码 + 阿里云官方文档 + 第三方交叉确认。

---


# 第 22 章 云原生与 Kubernetes

## 22.1 本章核心问题

把一个分布式数据库搬上 Kubernetes(以下简称 K8s)，本质上不是"能不能把二进制塞进容器"，而是要让两套控制面对齐：数据库内核早已有自己的成员管理、副本调度、故障切换与升级编排；K8s 的 StatefulSet / Deployment / Scheduler 也有一套以 Pod 为最小调度单元、以 PVC 绑定持久卷为存储语义的声明式编排模型。这两套控制循环并不天然兼容：K8s 默认假设 Pod 是可替换的、存储是网络可迁移的；而分布式数据库的副本是有共识身份的，存储常常是节点本地的低延迟盘。

关键在于，K8s 只能管理 Pod、Service、StatefulSet、Job、PVC 这些平台对象；而 Region、Raft Group、Tenant、Unit、Log Stream / LS、Leader、Primary Zone 等数据库内部对象仍由数据库自己的控制面管理。两层控制面如果各管各的，最容易出问题的地方往往不是初次部署，而是扩缩容、滚动升级、节点故障、PV 迁移、备份恢复和故障后重建。

本章要回答的关键问题是：

- TiDB 与 OceanBase 各自如何把"内核的编排需求"翻译成"K8s 的 Custom Resource(CR)+ Controller"，即 Operator 模型。
- K8s 原生的 StatefulSet 在"有共识身份的有状态服务"上有哪些结构性局限，哪些数据库语义不能交给 StatefulSet / PV 自动兜底，两家如何绕过。
- 本地盘(local PV)与云盘(cloud disk)的取舍如何影响 Pod 故障恢复、PV 迁移与调度反亲和。
- 托管服务(TiDB Cloud / OceanBase Cloud)背后的架构与自建 K8s 部署差异在哪里。

两家的形态差异会在 K8s 上被放大。TiDB classic 的组件边界天然接近 K8s 的分层：TiDB Server 是 SQL 计算入口、TiKV 是持久化 KV / Raft 存储节点、PD 是调度与元数据控制面，TiFlash / BR / 监控也可分别编排。OceanBase classic 则把 SQL、事务、存储、多租户资源与 RootService / sys tenant 体系更内聚地放在 OBServer 与集群内部；K8s 只看到一组 OBServer Pod，数据库内部却仍要维护 Tenant、Unit、Resource Pool、Zone、Locality、Primary Zone 与 Log Stream / LS。

需要说明的是，本章主题对应的算子仓库(`pingcap/tidb-operator`、`oceanbase/ob-operator`)本身是独立仓库，与内核仓库(`pingcap/tidb`、`tikv/tikv`、`oceanbase/oceanbase`)分离；本章源码引用区分这两类来源：内核侧的容器适配证据 commit-pinned 到 `release-8.5` / `v4.2.5_CE` 统一基线，算子侧 CR 字段以算子官方文档与仓库为准。**两层控制面一旦冲突，后果就是数据丢失、脑裂或长时间不可用——这正是云原生在分布式数据库底层最吃重的地方。**

## 22.2 TiDB 的实现

**组件与天然适配点。** TiDB classic 由三个组件构成：无状态 SQL 层(TiDB Server)、分布式 KV(TiKV)、元数据与调度中心(PD)。这种彻底的组件化是它在 K8s 上的天然优势：TiDB Server 无本地持久状态(本地仅有可重建的 schema / plan / session 缓存，详见第 6 章)，可用近似无状态服务的方式编排；TiKV 与 PD 才需要持久卷。需要强调的是，"无持久状态"不等于"无会话状态"：锁定基线的 `pkg/server/server.go` 显示 TiDB Server 是一个 MySQL 协议服务器，持有连接 map、internal sessions、listener 与 health 状态；它适合被 K8s 当作可横向扩缩的 SQL Pod 管理，但长事务、连接、prepared statement、临时对象仍会受 Pod 生命周期影响。

**Operator 模型(控制路径)。** TiDB Operator 是 TiDB on Kubernetes 的核心入口，官方将其定义为 TiDB 集群在 K8s 上的自动运维系统，生命周期覆盖部署、升级、扩缩容、备份、故障恢复与配置变更。控制路径上，用户创建 `TidbCluster`(描述集群期望状态)以及配套的 `TidbMonitor`、`TidbInitializer`、`Backup`、`Restore`、`BackupSchedule` 等 CR;`tidb-controller-manager`(一组自定义 controller，持续对比期望态与实际态并调谐)监听 CR 中的期望状态，为 PD、TiKV、TiDB、TiFlash、监控等组件生成或更新 StatefulSet / Deployment / Service / Job，再由 K8s 原生 controller 创建 Pod、PVC 与 Job。控制面还包括：`tidb-scheduler`(K8s 调度器扩展，注入 TiDB 专属调度策略；v1.6 起官方已不推荐部署)、`tidb-admission-webhook`(动态准入，对 Pod / StatefulSet 等做修改与校验)、`discovery`(每个集群一个发现 Pod，供组件互相发现，尤其用于 PD 初始集群成员协商)。

`TidbCluster` 的 spec 把每个组件作为子规格：`spec.pd` / `spec.tikv` / `spec.tidb` / `spec.tiflash` / `spec.ticdc` 等，通用字段含 `replicas`、`baseImage`(如 `pingcap/tikv`)、`version`(镜像 tag，如 `v8.5.x`)、`config`(TOML)、`storageClassName`、`requests` / `limits`。这一层只说明 K8s 对象如何收敛，并不代表 Region 调度由 K8s 完成：PD 仍负责 Region / Store / placement rule / 调度。锁定基线的 PD 源码中，`Rule` 明确包含 peer role、count、label constraints、location labels、isolation level，说明副本拓扑需要 PD placement 与 K8s node / pod affinity 共同满足——这是两层独立的拓扑控制，不能互相替代。

**数据路径与编排的关系。** 数据读写路径仍是 client → TiDB Server →(PD 取 TSO / RegionCache 路由)→ TiKV Region Leader,K8s 只负责把这些 Pod 拉起来并维持期望副本数，不介入单次请求的数据路径。对存储层而言，TiKV Pod 比 SQL 层更"重"：锁定基线的 `src/storage/config.rs` 可见 `data_dir`、engine、reserve space、block cache、flow control、IO rate limit 等字段，`validate_engine_type` 还会根据磁盘上 engine 数据目录修正配置。这意味着 TiKV 的 PVC / PV 不是可替换的缓存，而是 Region 数据、Raft 状态、engine 格式与 IO 假设的承载体——这一性质决定了后文 local PV 故障路径的全部复杂度。

**正常运维路径：滚动升级与扩缩容。** 滚动升级的核心原则是"先让数据库内部安全，再让 K8s 改 Pod"。对 TiKV,Operator 在重启某个容器前，先调用 PD 接口把该 TiKV 上的所有 Region Leader 驱逐(evict leader)出去；等 Leader 数降到 0(或超时)后再重建容器，然后逐个推进下一个 TiKV，避免升级期间请求打到正在重启的 Leader。对 PD,Operator 也不只是改 StatefulSet 的 `partition`:它结合 PD API 获取成员健康与 Leader 状态，缩容前 transfer Leader 并调用 PD 删除成员，升级时比较 StatefulSet revision、检查 PD member health，避免直接重启 Leader 触发不必要的选举。扩缩容通过调整 StatefulSet 副本数实现；TiKV 扩容是新增 Pod / PVC 后由 TiKV 加入集群、再由 PD 调度 Region，缩容则需数据库层先迁走 Region 或下线 store,K8s 删 Pod 只是最后一步。缩容时 Operator 对 PVC 加 `deferDeleting` 注解以保数据安全，按原位置扩容时若不先清理残留 PVC，新 Pod 会携带旧数据。

**故障路径：自动 failover。** 当 TiKV Pod 故障，store 状态先变 `Disconnected`，经 PD 的 `max-store-down-time`(默认 `30m`)变为 `Down`；之后 Operator 再等 `tikvFailoverPeriod`(默认 `5m`)，仍未恢复则把 Pod 记入 `TidbCluster` 的 `.status.tikv.failureStores`，在计算 StatefulSet 副本时把它算进去，从而创建一个新 Pod 补副本。关键细节：TiKV / TiFlash 的 failover 与 PD / TiDB 不同——故障 store 恢复后，Operator 默认不会自动删掉新建的 Pod(因为缩容 TiKV 会触发数据迁移)，需显式 `spec.tikv.recoverFailover: true` 或 `recoverByUID` 回收；`maxFailoverCount` 默认 `3`，防止 failover 失控创建过多 Pod。

**容器自感知(内核侧源码证据)。** TiDB 内核自身要"知道自己在 Pod 里"。`InContainer()` 读 `/proc/self/cgroup`(cgroup V1)与 `/proc/self/mountinfo`(cgroup V2),V1 路径显式匹配 `"docker"`、`"kubepods"`、`"containerd"` 三个字符串(命中任一即判定在容器内)。启动时 `cmd/tidb-server/main.go` 调用 `maxprocs.Set(...)`(uber-go/automaxprocs)，按 cgroup CPU quota 把 K8s 给 Pod 设的 CPU limit 映射成 Go 运行时并行度；`cgmon` 后台 goroutine 以 `refreshInterval = 10 * time.Second` 周期刷新 maxprocs 与内存上限，支持 Pod limit 运行期变化。内存侧 `cgroup_memory.go` 对 cgroup V2 读 `memory.max`，代码注释明确：K8s 下 `memory.max` 等于 Pod limit，但"better be adjusted by some factor, like 0.9, to avoid OOM"——即需乘约 0.9 系数防 OOM。GOMAXPROCS 暴露为 Prometheus 指标 `tidb_server_maxprocs`,Grafana 面板以 `tidb_server_maxprocs{k8s_cluster=..., tidb_cluster=...}` 形式使用，其中 `k8s_cluster` / `tidb_cluster` 是 TiDB Operator 注入的标准标签。

![[f22_1.svg]]

**图 22-1　cgroup → GOMAXPROCS 容器自感知流程**

**备份恢复(CR 驱动 + 云盘快照路径)。** TiDB Operator 用 `Backup` / `Restore` / `BackupSchedule` CR 描述任务：官方文档列出 `backupMode` 可为 snapshot、volume-snapshot、log,`restoreMode` 可为 snapshot、volume-snapshot、pitr，并说明备份期间会临时处理 `tikv_gc_life_time`，极端情况下若 Operator 无法访问数据库需人工检查恢复。以 EBS 卷快照备份为例：用户发起请求生成 CRD,Operator 据此创建 Backup Worker / Restore Worker(K8s Job);Backup Worker 先从 PD 取集群级 `resolved_ts` 标记事务一致点，禁用 PD 调度(replica / merge / GC)，对所有 TiKV 卷发起 EBS snapshot，轮询直到全部 completed。备份元数据模型(`br/pkg/config/ebs.go`)直接内嵌 `corev1.PersistentVolume` / `PersistentVolumeClaim` 与 `crd_tidb_cluster` 字段，`EBSVolumeType` 仅允许 `gp3` / `io1` / `io2`。这里的工程含义是：备份不是 K8s 对 PVC 做一份快照就结束——它与 TiKV MVCC、GC safe point、对象存储权限、Job 重试与集群可访问性强相关，volume snapshot 只适用于明确支持的模式和存储插件，不能替代数据库一致性校验。 〔文献[1-4,9-10,12]〕

> **架构演进(v1 → v2):** TiDB Operator v2 是重大重构，移除了对 StatefulSet 的依赖，改为直接管理 Pod、ConfigMap、PVC，并把 v1 单一的 `TidbCluster` CRD 拆成 `Cluster` / `ComponentGroup` / `Instance` 三层 CRD(每个 `Instance` 管一个 Pod)。撰写时 v1.6 为当前 v1 稳定线(K8s 依赖升级到 v1.28、`tidb-scheduler` 不再推荐部署、PD 微服务模式为实验特性、支持组件并行扩缩容);v2 已 GA 并冻结为补丁稳定线——v2.0.0 于 2025-12-05 作为首个 GA 发布、v2.0.1 于 2026-03-25 发布为补丁版，`main` 分支的活跃演进已推进到 v2.1.x(beta)/ v2.2.x(alpha)。本章正文以 v1 架构为主线，因为锁定基准 TiDB 8.5.x 的生产部署主体仍是 v1.6;v2 作为演进态在 §22.8 讨论。

## 22.3 OceanBase 的实现

**表 22-1　OBD / OCP / ob-operator 三方边界**

| 工具 | 定位 | 典型环境 | 是否写入集群状态 | 并用风险 |
|---|---|---|---|---|
| OBD | 部署与包管理入口，命令式部署 | 裸机 / 虚机 / 实验环境 | — | 不应与 ob-operator CR 混成同一层抽象 |
| OCP | 企业 / 云运维管控平台,关注生命周期、监控告警、备份恢复与巡检 | 企业 / 云环境 | 与 ob-operator 并用时须确认最终写入者 | 与 ob-operator 双控制面可同时改拓扑 / 参数 / 备份策略致冲突 |
| ob-operator | 把集群 / Zone / OBServer / Tenant / 备份恢复 / O&M 投影成 K8s CR,经 K8s API 驱动 | K8s 环境 | 通过 CR 状态驱动集群对象 | 与 OCP 并用时须确认由哪一套控制面写入集群状态 |

**组件与适配难点。** OceanBase 4.x 是一体化内核：同一个 OBServer 进程内承载 SQL 引擎、会话、多租户、事务、PALF(Paxos-backed Append-only Log File system)日志与存储引擎(详见第 6 章)。它没有 TiDB 那样可独立伸缩的无状态 SQL 层——每个 OBServer 都是带持久状态的对等节点。这使它在 K8s 上的适配天然比 TiDB 更"重"：没有可随意增删的无状态副本，所有节点都涉及共识与本地存储。这一形态差异贯穿后续 Operator 模型与故障恢复的讨论。

OceanBase 的 K8s 入口主要是 ob-operator，在非 K8s 或混合环境中还常见 OBD / OCP。三者边界要分清：OBD 更像 OceanBase 开源软件的部署与包管理入口，适合裸机、虚机或实验环境的命令式部署；OCP 更偏企业 / 云环境的运维管控平台，关注集群生命周期、监控告警、备份恢复与巡检；ob-operator 则把 OceanBase 的集群、Zone、OBServer、Tenant、备份恢复与 O&M 操作投影成 K8s CR，让平台侧通过 K8s API 驱动这些对象。因此在 K8s 场景下，不应把 OBD 的配置文件、OCP 的运维视图和 ob-operator 的 CR 状态混成同一层抽象。若某环境同时使用 OCP 与 ob-operator，还要确认最终写入集群状态的是哪一套控制面，否则据此推测容易出现两个控制面同时修改拓扑、参数或备份策略的冲突。

**Operator 模型(控制路径)。** ob-operator 基于 kubebuilder v3 开发，定义了一组 CR,API group 为 `oceanbase.oceanbase.com`(如 `obclusters.oceanbase.oceanbase.com`)。它与 TiDB Operator 的最大差异是显式暴露了更多 OceanBase 内部对象：官方架构文档列出的 CR 包括 `OBCluster`、`OBZone`、`OBServer`、`OBParameter`、`OBTenant`、`OBTenantBackupPolicy`、`OBTenantBackup`、`OBTenantRestore`、`OBTenantOperation`、`OBTenantVariable`、`OBClusterOperation`、`OBResourceRescue`、`K8sCluster` 等。这说明 ob-operator 不是只管理一个 StatefulSet，而是在 K8s API 中表达 OceanBase 集群、Zone、OBServer、Tenant 与备份 / 恢复 / O&M 任务。控制面由 Controller Manager 统一注册，Controller 把实际态(Status)向期望态(Spec)对齐，配套 Webhook 与一组 ResourceManager。部署上 ob-operator 依赖 cert-manager(用于证书管理)，最小要求约 2 CPU / 10 GB 内存 / 100 GB 存储，安装方式为 `kubectl apply` cert-manager 与 `deploy/operator.yaml`。撰写时 ob-operator 最新 release 为 2.3.4(GitHub Releases API `published_at` 与官方 changelog 均为 2026-01-13，见 §22.13 版本核查备注)。

**CR 模型如何映射 Zone / Unit / Server。** OceanBase 的拓扑模型(Zone / Unit / Tenant / Locality / Resource Pool，详见第 1、17 章)在 ob-operator 里通过 `OBCluster` 的 `topology` 字段表达：

```yaml
spec:
 topology:
 - zone: zone1
 replica: 1 # 该 Zone 内 OBServer 数
 nodeSelector: {...}
 affinity: {...}
 tolerations: [...]
 observer:
 resource: {cpu: 2, memory: 10Gi}
 storage:
 dataStorage: {storageClass: local-path, size: 50Gi}
 redoLogStorage: {storageClass: local-path, size: 50Gi}
 logStorage: {storageClass: local-path, size: 20Gi}
```

每个 Zone 按 `replica` 展开为相应数量的 OBServer Pod;Pod 直接由 controller 生成(不通过 StatefulSet),Pod 名遵循 `{cluster_name}-{cluster_id}-{zone}-uuid` 格式。值得注意的是存储被显式拆成三类：`dataStorage`(SSTable 数据)、`redoLogStorage`(PALF 日志 / clog)、`logStorage`(运行日志)，分别可配不同 StorageClass——这与 OceanBase 内核把"数据盘 / 日志盘"分离的设计一致，日志盘对延迟敏感。

但 K8s 层给 OBServer Pod 分配 request / limit 与 PVC 只是"外层资源"，数据库内部还要满足 Tenant 的 Unit 与 Resource Pool 约束。锁定基线的内核源码中，`src/rootserver/ob_unit_manager.cpp` 显示 Unit 管理会加载 unit config、resource pool、unit 并构建 tenant pools,`ObUnitLoad::get_demand` 会按 CPU、内存、log disk、data disk 读取 UnitConfig。因此一个 OBServer Pod 是否可删、可迁、可扩，不只取决于 StatefulSet ordinal 或 PVC 是否存在，还取决于该节点上 Tenant / Unit / LS / 副本分布是否已经完成数据库内部迁移。租户层面用 `OBTenant` CR 创建，把 Unit / Resource Pool 的资源规格(CPU / 内存 / IOPS，详见第 17 章)声明在 CR 里。

**正常路径：扩缩容、升级与备份。** ob-operator 创建 `OBCluster` 后，按 `topology` 创建每个 Zone 的 OBServer Pod / PVC / Service，引导集群，随后用户必须创建业务 Tenant。集群扩容通过修改 `spec.topology`(增加 Zone 或某 Zone 的 `replica`),controller 调谐时创建新 OBServer Pod 并执行 `ALTER SYSTEM ADD SERVER` 把新节点加入集群；租户级伸缩通过调整 `OBTenant` 的 Unit 规格；升级通过修改镜像版本字段，由 controller 编排滚动替换。若开启备份，`OBTenantBackupPolicy` 与 `OBTenantBackup` 用于 Tenant 级备份，备份目标可以是 NFS 或 OSS，策略定义归档与数据备份参数、生效后生成具体任务。这比 TiDB 的 BR CR 更贴近 Tenant 语义：OceanBase 的备份恢复、物理备库、failover / switchover 往往围绕 Tenant 角色与 Tenant 资源展开，而非围绕单个 Pod / PVC。

**故障路径：OBServer 故障恢复。** 当少数 OBServer Pod 故障，ob-operator 检测到 Pod 异常，创建一个新 OBServer 节点加入集群，OceanBase 内核把异常节点上的副本数据复制到新节点，再删除异常节点。判断时不能只看"Pod Running"——Pod Running 只能证明容器进程存在，不能证明 OBServer 已加入集群、Tenant 可用、Unit 分配健康、RootService 已完成相应任务；官方给出的排查入口包括查看 `obclusters.oceanbase.oceanbase.com`、`obtenants.oceanbase.oceanbase.com`、`operationContext`、operator 日志与 OBServer 容器日志。这里有一个 K8s 特有的硬约束：OceanBase 4.2.3.0 之前，内核不能用虚拟 IP 通信，Pod IP 一变 observer 就无法启动，要原地重启必须固定节点 IP;4.2.3.0+ 才支持给每个 OBServer Pod 分配恒定 ClusterIP 的 K8s service 模式，或借 Calico 复用故障节点 IP 直接重用既有数据(免复制)。恢复前提是"至少 3 节点 + 租户至少 3 副本"，且只能处理少数派故障；多数派 OBServer 故障则无法标准恢复，需走备份 / 恢复。

**内核 README 对 K8s 的官方指引随版本出现。** `oceanbase/oceanbase` 内核 README 在 4.3.5(commit `b28b9bb12f3b`)与 4.4.x(commit `d4bef8d29a4c`)含 "Start with Kubernetes" 章节指向 ob-operator；而 v4.2.5_CE(commit `e7c676806fda`)README 对 `kubernetes` / `operator` 零命中——即内核 README 对 ob-operator 的官方指引在 4.3.x / 4.4.x 才出现。需强调：ob-operator 仓库本体的 CR 字段定义不在内核 checkout 内，本节 CR 字段以 ob-operator 官方文档为准，未编造内核常量。判断 OBServer / Unit / Tenant / LS 是否健康的内部视图名可在内核源码核实：`DBA_OB_TENANTS`、`DBA_OB_UNITS`、`DBA_OB_SERVERS`、`GV$OB_UNITS`、`GV$OB_SERVERS`、`DBA_OB_LS`、`GV$OB_LOG_STAT` 均为内核内置视图(`src/share/inner_table/ob_inner_table_schema_constants.h` 定义对应 `TID` 与表名);其中资源不足报错 `OB_ZONE_RESOURCE_NOT_ENOUGH`(-4733)的提示文案直接引导运维"check resource info by views: DBA_OB_UNITS, GV\$OB_UNITS, GV\$OB_SERVERS"。其余 Prometheus metric 名、日志字段名、OceanBase 容器化专用指标名未核验者统一标"需进一步查证"。 〔文献[6-8,11]〕

## 22.4 核心差异对比

两家在 K8s 上的差异，根源不在 Operator 写得好不好，而在内核形态：TiDB 组件化、OceanBase 一体化。表 22-2 按内核形态、CR 模型、Pod 编排、拓扑、存储、Pod IP、故障切换、升级协同、多租户、备份、依赖等维度分别陈述，每行末列点出该差异在工程上的影响。

**表 22-2　云原生与 Kubernetes:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB(TiDB Operator) | OceanBase(ob-operator) | 影响 |
|---|---|---|---|
| 内核形态 | 组件化：无状态 SQL + TiKV + PD | 一体化：对等 OBServer 进程，RootService / sys tenant 与 Tenant / Unit 体系耦合 | TiDB 天然有可弹性伸缩的无状态层；OB 所有节点带状态 |
| 核心 CR | v1 `TidbCluster`(单 CR);v2 拆 `Cluster` / `ComponentGroup` / `Instance` | `OBCluster` + `OBZone` + `OBServer` + `OBTenant` 等多 CR | OB 把 Zone / 租户作为一等 CR,TiDB v1 集中在一个 CR、v2 才细分 |
| Pod 编排 | v1 经 StatefulSet;v2(已 GA)直管 Pod | controller 直管 Pod(非 StatefulSet) | 两者都不能把 StatefulSet 当数据库 HA 机制；TiDB v1 仍受其约束 |
| 拓扑表达 | K8s affinity + PD placement rule / location labels / isolation level | K8s topology + OceanBase Zone / Locality / Primary Zone / Tenant Unit | TiDB 拓扑控制分两层；OceanBase 拓扑更内核化 |
| 存储模型 | 按组件配 storageClass;TiKV `data_dir` / engine / Raft 状态使 PVC 强绑定 store 语义 | 三类存储分离(data / redoLog / log)各配 storageClass，同时受 Unit resource demand 约束 | OB 显式把日志盘与数据盘分离，延迟敏感的 PALF 单独配置 |
| Pod IP 依赖 | 无状态层不依赖固定 IP;TiKV 靠 store id | 4.2.3.0 前依赖固定 IP，需 K8s service / Calico 兜底 | OB 对 Pod IP 漂移更敏感，是 K8s 适配的历史难点 |
| 故障切换驱动 | Operator + PD(evict leader / failureStores) | Operator 检测 + 内核 Paxos 自切主 + 补副本 | 两者都靠内核共识切主，Operator 负责补 Pod / 补副本 |
| 升级 Leader 协同 | 升级前 PD evict 全部 Region Leader、transfer PD Leader | 滚动替换由内核 Paxos 切主 | 都做"先转移 Leader 再重启"的协同 |
| 多租户 | RU 软隔离，无独立租户 Pod | `OBTenant` CR + Unit 硬隔离，在共享 OBServer 上切片 | OB 租户不是独立 Pod 而是进程内资源切片，K8s 看不到租户边界 |
| 备份恢复 | `Backup` / `Restore` / `BackupSchedule`,snapshot / log / PITR / volume-snapshot 按 CR 编排 | `OBTenantBackupPolicy` / `OBTenantBackup` / `OBTenantRestore`，更偏 Tenant 级 | 恢复目标与一致性边界不同，测试不能只看 Job 成功 |
| cert-manager | 不强依赖 | 强依赖(证书管理) | OB 部署前置依赖更多 |

第二张对比表聚焦 StatefulSet 能力与分布式数据库需求的落差，见 §22.8；运维复杂度对比表见 §22.7。 〔文献[5]〕

## 22.5 正常路径图

图 22-2 展示两套控制循环如何叠加：K8s 把 CR 收敛成 Pod / PVC / Service / Job，数据库控制面让 Region 或 Tenant / Unit / LS 达到安全状态，二者并行存在。

![[f22_2.svg]]

**图 22-2　云原生与 Kubernetes正常读写/调度路径**

正常路径要点：控制面的 reconcile 循环与数据面的请求路径是解耦的——K8s 只在拉起 Pod、滚动升级、补副本时介入，单次 SQL 请求不经过 controller。正常变更应当从数据库语义出发、由 Operator 调整 K8s 对象，而不是人工直接删 Pod 或改 PVC;TiDB 侧升级时 `tidb-controller-manager` 主动调 PD 驱逐 Leader，是控制面与数据面唯一的强协同点。

## 22.6 故障/异常路径图

图 22-3 展示一个 TiKV / OBServer 所在 K8s 节点宕机后的故障路径，以及 local PV 带来的"卷无法跨节点"陷阱。

![[f22_3.svg]]

**图 22-3　云原生与 Kubernetes故障/异常路径**

K8s local volume 文档明确：local volume 需要 PV nodeAffinity，并受底层节点可用性约束；节点不健康时 local volume 对 Pod 不可访问，Pod 也无法运行。因此一个典型反例是：TiKV 或 OBServer 使用 local PV，节点永久损坏，调度器遵守 nodeAffinity 无法把绑定该 PV 的 Pod 放到其他节点。正确路径是让 PD / Operator(TiDB)或 RootService(OceanBase)按节点故障处理、补副本、确认副本健康后再清理旧 PV / PVC；错误路径是平台工程师强行删除 PVC 让同名 Pod 在新盘启动，这可能绕过数据库内部下线 / 补副本 / 恢复流程，造成旧数据丢失、重复成员身份或副本数不足。

OceanBase 侧的对称故障还要叠加一个 K8s 特有风险：4.2.3.0 之前 Pod IP 漂移会让 observer 无法原地启动，即使 PV 还在新 Pod 也起不来，必须靠固定 IP / K8s service ClusterIP / Calico 复用 IP 兜底。这是"内核身份(IP / server)与 K8s Pod 可替换假设冲突"的典型表现。

## 22.7 性能、可靠性、运维影响

K8s 带来的性能与可靠性影响，几乎都不在编排层本身，而在它逼出的存储与拓扑选型。下面分延迟吞吐、可靠性、扩展性三个角度展开。

**延迟与吞吐。** K8s 编排层本身不在数据路径上，对稳态延迟 / 吞吐影响有限；真正的性能影响来自存储选型。TiKV 强烈推荐 local SSD(低延迟、低抖动)，因为 Raft 已在应用层提供副本冗余，丢一块本地盘可由 PD 调度从其它副本补回——用 Raft 冗余换本地盘延迟优势是 TiDB 的核心取舍。代价是节点不可达时恢复更依赖数据库副本与人工处置；cloud disk 更易重新 attach，但延迟、IO 抖动、attach / detach 时间与跨可用区限制会进入 P99。PD 因偏元数据可用 local SAS 或网络 SSD；监控 / Binlog / 备份无副本则必须用网络盘。OceanBase 同理把延迟敏感的 PALF 日志盘单独配置。容器层还有一个隐性影响：CPU limit 经 cgroup 映射成 GOMAXPROCS(TiDB)或内核线程并行度；若 Pod limit 设得过小，会直接压低并行度，甚至触发 CPU throttling(见 §22.2 cgroup 自感知)。

**可靠性与故障恢复。** 两家集群内都满足 RPO=0(共识层保证，详见第 3、18 章),K8s 只影响"补副本 / 换 Pod"的 RTO 维度。StatefulSet 只提供稳定身份与有序更新，不提供数据库一致性：若绕过 Operator 直接改 StatefulSet,K8s 会认为变更成功，但 PD 仍可能保留旧成员、Region 迁移未完成、PVC 未清理或误用旧数据；OceanBase 同理，Pod Running 不等于 Tenant 资源健康。TiKV 的 `max-store-down-time`(30m)+ `tikvFailoverPeriod`(5m)是有意保守的：防止节点短暂抖动就触发大规模数据迁移，这意味着默认配置下单 TiKV 节点宕机后约 35 分钟才会自动补 Pod，期间靠 Raft 多数派维持可用、冗余度暂时降低。

可靠性验收应分三层看恢复目标。第一层是 K8s 层：Pod 是否重新调度、PVC 是否重新绑定、Service endpoint 是否恢复、事件中是否有 volume node affinity 或 attach / detach 失败。第二层是数据库成员层：TiDB 要看 PD member、TiKV store、Region peer、Leader 分布是否恢复到目标；OceanBase 要看 OBServer 是否重新纳管、Zone 是否满足拓扑、Unit 是否迁移完成、Tenant 是否可读写。第三层是业务层：连接池是否重连、写入是否超时重试、长事务是否被中断、备份恢复后的校验是否通过。只有第一层恢复不能代表数据库恢复，只有第二层恢复也不能代表业务恢复——据此可以推测，这正是有状态数据库在 K8s 上比无状态服务更难验收之处。

**扩展性。** TiDB 的无状态 SQL 层可近乎线性弹性扩容(加 TiDB Server Pod 即可，无数据迁移);TiKV / OBServer 扩容则必然伴随数据迁移(Region / Tablet 再平衡)，受网络与磁盘带宽限制。OceanBase 因无独立无状态层，计算弹性弱于 TiDB——这正是其 shared-storage(Bacchus)试图改善的方向(详见第 7、23 章)。

调度策略要避免"看起来分散、实际同故障域"的假象。Pod anti-affinity 可让 TiKV 或 OBServer 分散到不同节点，topology label 可表达 zone / rack / host，但存储插件是否真正跨可用区、PV 是否与 Pod 在同一故障域、云盘是否允许跨区 attach、local PV 是否只在单机可用，都要逐项确认。TiDB 中还要把这些 label 与 PD placement rule 对齐，OceanBase 中还要把它们与 Zone / Locality / Primary Zone 规划对齐；若只在 K8s 层做 Pod 分散，数据库内部副本仍可能集中在同一磁盘池、同一机架或同一云可用区，据此推测故障时未必能达到预期容灾效果。

**运维复杂度对比表：**

**表 22-3　性能、可靠性、运维影响**

| 运维维度 | TiDB on K8s | OceanBase on K8s | 说明 |
|---|---|---|---|
| 前置依赖 | Operator(可选 cert-manager / local-provisioner) | 强依赖 cert-manager + local-path-provisioner | OB 前置组件更多 |
| 最小拓扑 | PD×3 + TiKV×3 起 | ≥3 节点 + 租户≥3 副本 | 都需 3 副本保多数派 |
| Pod IP 稳定性要求 | 低(无状态层无所谓) | 高(4.2.3.0 前需固定 IP) | OB 历史包袱更重 |
| 升级编排 | Operator 主动 evict leader / transfer PD Leader | 内核 Paxos 切主 | 都做 Leader 转移 |
| 备份模型 | CRD 驱动 Backup / Restore Job + EBS 快照 | `OBTenantBackup` / `Restore` CR(租户级) | OB 备份以租户为粒度 |
| 故障补副本时延(默认) | 约 35min(30m + 5m)后建替补 Pod | 检测 Pod 异常即建新 OBServer | TiDB 默认更保守 |
| 控制面观测形态 | 多组件协作，观测对象分散(Pod / PVC / PD member / store / Region / SQL session / BR Job) | 更内聚，ob-operator 暴露多 CR 状态与 `operationContext` | TiDB 像"多组件编排",OB 像"内核资源模型投影到 K8s" |
| 多租户在 K8s 的可见性 | RU 软隔离，K8s 不可见 | `OBTenant` CR 可见，但租户非独立 Pod | OB 租户是进程内切片 |

## 22.8 反例与代价

下面六个反例的共同根因只有一个：把 K8s 的"Pod / PVC 可替换"假设当成了数据库的安全保证。它们多数不会在初次部署暴露，而在节点故障、扩缩容或升级时集中爆发。

**反例 1:local PV + 公有云 VM 本地盘 → 节点没了数据没了。** 这是最经典的 K8s 部署反例。TiKV 用 local PV 换低延迟，但其 nodeAffinity 把卷死锁在某个节点上。若该节点是公有云上用 instance-store(本地盘)的 VM，节点一旦回收，该 PV 上的数据物理消失且无法重新挂载到其它节点。官方明确：虚拟机用本地盘时，节点故障后数据无法取回。集群层面靠 Raft 三副本仍能恢复冗余；但若同时丢两个副本(如反亲和未配好，两个 TiKV 落在同一物理节点或同一 AZ)，就会丢多数派 → 该 Region 不可用甚至数据丢失。代价：local PV 的性能优势是以"单盘不可迁移 + 强依赖正确的反亲和与 AZ 打散"为前提的。

**反例 2:Volume Node Affinity Conflict 导致替补 Pod 永久 Pending。** 当 TiKV 节点宕机，Operator 想建替补 Pod，但若它复用了原 local PV 的 PVC,PV 的 nodeAffinity 锁定已宕机的节点，替补 Pod 调度时报 `volume node affinity conflict` 卡在 Pending。此时真正的恢复路径是"让 Raft 在健康节点新建一个全新副本(新 PVC / 新 PV)"，而非复用旧卷；若运维误以为"等节点回来挂回旧盘"，就会陷入长时间不可用。更糟的错误是直接删除 PVC 让同名 Pod 在新盘启动——这可能让旧 store 身份与新数据目录关系不清、Region 元数据恢复受阻，其具体风险据推测取决于 Operator 与数据库的保护逻辑。

**反例 3:把 StatefulSet 滚动升级当作数据库滚动升级。** K8s 可以按 ordinal 更新 Pod，但它不知道 PD Leader、TiKV Region Leader、OceanBase Unit / LS 是否正在迁移。这正是 TiDB Operator 在数据库 API 层做 Leader transfer、member health 检查与 member delete 的原因(见 §22.2);OceanBase 侧 ob-operator 给出 `OBClusterOperation`、`OBTenantOperation` 等资源，说明集群与 Tenant 运维任务需专门 CR 表达，不能只靠 `kubectl rollout restart`。

**反例 4:OceanBase Pod IP 漂移(4.2.3.0 前)。** 即使 PV 完好，OBServer Pod 重建拿到新 IP 也起不来，因为旧版内核以 IP 作为 server 身份通信。这暴露了一体化内核与 K8s "Pod 可替换"假设的根本冲突：数据库节点有强身份，K8s Pod 默认无强身份。ob-operator 用"固定 IP / 恒定 ClusterIP service / Calico 复用 IP"三种兜底，本质都是给 Pod 强行加上稳定网络身份(见 §22.6)。

**反例 5:StatefulSet 的结构性局限(TiDB v1)。** StatefulSet 有几个对分布式数据库不友好的约束：(a) `volumeClaimTemplate` 创建后不可改，导致原生扩卷困难；(b) 强制有序滚动更新，叠加 Leader 转移会造成重复的 Leader 调度抖动；(c) 同一 controller 下所有 Pod 配置必须相同，逼出复杂的启动脚本来区分参数；(d) 它没有定义 raft 成员的 API，导致"重启 Pod"与"移除 raft 成员"语义冲突。代价：这正是 TiDB Operator v2 与 ob-operator 都选择"直管 Pod"而非依赖 StatefulSet 的原因——但直管 Pod 意味着 Operator 要自己实现 StatefulSet 已有的有序性、唯一标识、PVC 绑定等逻辑，把复杂度从 K8s 内核搬进了 Operator。下表对照 StatefulSet 能力与分布式数据库的真实需求：

**表 22-4　反例与代价**

| StatefulSet 能做什么 | 分布式数据库还需要什么 | TiDB 风险点 | OceanBase 风险点 |
|---|---|---|---|
| 稳定 Pod 名称与 ordinal | 成员身份、store / member / OBServer 身份一致性 | 旧 PVC 在原 ordinal 重建可能携带旧 store 数据，需 Operator / PD 流程兜底 | OBServer Pod 重建后还要与集群成员、Zone、Unit 状态对齐 |
| 稳定 PVC 绑定 | 数据库一致性快照与日志恢复 | volume snapshot 不等于 BR / PITR 一致性 | PVC 存在不代表 Tenant / LS 已恢复 |
| 有序滚动更新 | Leader / 副本迁移、健康检查、流量排空 | PD / TiKV 升级前后要结合 PD API / Region 状态 | RootService / OBServer / Tenant 操作可能有长事务或迁移任务 |
| 有序扩缩容 | 安全下线、数据迁移、资源池调整 | 缩 TiKV 前须迁走 Region，缩 PD 前处理 Leader / member | 缩 OBServer 前需处理 Unit / LS / Tenant 资源 |
| 重建失败 Pod | 判定数据副本是否可用、是否需人工恢复 | local PV 节点不可达时 Pod 可能无法调度 | local PV / 多盘配置与 Unit 状态错配会放大恢复复杂度 |

**反例 6:用 Cloud 托管服务倒推自建 K8s。** TiDB Cloud Dedicated 可理解为托管 classic 的产品形态，但 Starter / Essential 内部隔离机制未公开；OceanBase Cloud 也不能直接等同于 CE + ob-operator。云厂商可能有专用调度、存储、网络与控制面，因此用户在自建 K8s 中用 local-path-provisioner 或普通 CSI 时，并不确定能否期待同样的恢复语义。

**与替代方案的取舍。** 不上 K8s(裸机 TiUP / OBD 部署)能避开上述所有 PV / 调度陷阱，但失去声明式编排与弹性；上 K8s 则换来统一编排、自愈与云可移植，代价是要正确处理本地盘、反亲和、Pod 身份这些有状态服务的难点。两家都没有把运维复杂度消除，只是把人工 runbook 编码进 Operator。

## 22.9 测试开发视角的验证点

测有状态数据库在 K8s 上的难点，不是验证某个 CR 能不能创建，而是验证"K8s 报告成功"与"数据库真的健康"之间的落差。下面分功能场景、失效注入、控制面/数据面解耦、压测与观测四类组织验证点。

**可测试的功能场景。** 应覆盖：初始化三副本集群、扩 TiDB Server、扩 TiKV / OBServer、缩容指定 Pod、滚动升级、备份恢复、PITR / 日志恢复、跨可用区 / Zone 拓扑、节点维护、Operator 重启、K8s API 临时不可用、对象存储 / NFS 权限错误。TiDB 侧要验证 `TidbCluster`、`Backup`、`Restore`、`BackupSchedule` CR 的状态变化与 PD / TiKV / BR 实际状态一致——例如滚动升级期间持续压测，验证 evict leader 是否真正把请求从重启节点摘除(P99 不应出现明显尖刺)；扩容后验证 Region 是否均衡迁移，缩容时验证 PVC 是否被正确 `deferDeleting`；用 EBS 快照备份后，在拓扑不一致(故意改 TiKV 数量 / 卷数量)的集群上恢复，验证 Restore Worker 是否如设计所述校验拓扑并报错退出。OceanBase 侧要验证 `OBCluster`、`OBTenant`、`OBTenantBackupPolicy`、`OBTenantBackup`、`OBTenantRestore`、`OBTenantOperation` 的状态与 OBServer / Tenant 可用性一致。

**可注入的失效模式。** 删除 TiDB Server Pod；删除 TiKV Pod 但保留 PVC;local PV 节点 NotReady;cloud disk attach 延迟；PD Leader 所在 Pod 滚动升级；operator controller-manager 重启；backup Job 失败 / 对象存储凭据错误；OBServer Pod 重启；Tenant 创建中断；Unit / Zone 资源不足；NFS 不可写。其中最高风险的组合是"节点损坏 + local PV 不可达 + 副本恢复"，因为它同时触发 K8s 调度、PV nodeAffinity、数据库副本恢复与人工清理流程：要验证替补 Pod 是否卡 `volume node affinity conflict`，以及 Raft / Paxos 是否在健康节点重建副本。OceanBase 还应专门模拟 Pod IP 漂移(删 Pod 让其重建拿新 IP)，验证 4.2.3.0+ 的 ClusterIP service 模式能否原地恢复；网络分区场景则隔离一个 Zone / AZ，验证多数派是否仍可用、少数派恢复后是否自动追平。

**控制面与数据面解耦的回归用例。** 应专门验证"控制面中断时的数据面行为":Operator 暂停或重启时，已运行的 TiDB / TiKV / PD 或 OBServer 不应因为 controller 暂时不可用而立即停止服务，但新建、扩缩、备份、恢复、故障替换等 reconciliation 任务会延迟。也要验证相反情况：数据库控制面可用但 K8s 控制面异常时，数据库可继续处理已有流量，但 Pod 重建、PVC attach、Job 创建与 Service endpoint 更新可能失败。把这两类故障分开，可避免把 K8s API 不可用误判为数据库不可用，也避免把数据库内部副本不可用误判为单纯的 Pod 调度问题。

**关键压测与观测指标。** 压测指标应按层拆开：SQL P50 / P95 / P99、写入吞吐、事务冲突 / 重试、TiKV / OBServer 磁盘 IO、Raft / Paxos 日志延迟、Region / LS 迁移速度、备份吞吐、恢复耗时、Pod 重启到服务恢复耗时、PVC attach / detach 时间、Operator reconciliation 延迟。已查证的观测项：TiDB 内核暴露 `tidb_server_maxprocs`(GOMAXPROCS 值),Grafana 面板用 `tidb_server_maxprocs{k8s_cluster=..., tidb_cluster=...}`，标签为 Operator 注入;`TidbCluster` 的 `.status.tikv.failureStores` / `.status.pd.failureMembers` 反映 failover 状态；PD 侧 `max-store-down-time`、Operator 侧 `tikvFailoverPeriod` / `maxFailoverCount` 是关键参数。可稳定使用的 K8s 诊断入口包括 `kubectl get/describe pods,pvc,pv,statefulset,events`、`kubectl logs`、`kubectl get obclusters.oceanbase.oceanbase.com -o yaml` 等官方文档出现的命令形态。OceanBase 侧判断 Server / Unit / Tenant / LS 健康的内部视图可在内核源码核实:`DBA_OB_SERVERS` / `GV$OB_SERVERS`(节点状态)、`DBA_OB_UNITS` / `GV$OB_UNITS`(Unit 分配)、`DBA_OB_TENANTS`(租户)、`DBA_OB_LS`(日志流)、`GV$OB_LOG_STAT`(日志同步状态)均为内核内置视图,资源不足报错 `OB_ZONE_RESOURCE_NOT_ENOUGH` 亦提示用 `DBA_OB_UNITS` / `GV$OB_UNITS` / `GV$OB_SERVERS` 排查。其余 Prometheus metric、参数名、日志字段名，以及 OceanBase 容器化专用指标名，若未从当前版本文档或源码确认，统一标"需进一步查证"，本章不编造。

## 22.10 容易误解点

**误解 1:"TiDB 在 K8s 上更简单，因为它更简单"。** 不准确。TiDB 的优势是组件化 + 无状态 SQL 层带来的弹性，不是内核"更简单"(其 Multi-Raft、Percolator 事务、PD 调度并不简单，详见第 3、10、13 章)。OceanBase 在 K8s 上更"重"也不是因为"更复杂落后"，而是一体化内核 + 强节点身份与 K8s 的可替换 Pod 假设冲突更多——这是工程取舍，不是优劣排序。

**误解 2:"用了 K8s 就有了高可用，丢节点无所谓"。** 错。K8s 的自愈是"重建 Pod"，但数据冗余靠的是数据库内核的 Raft / Paxos 副本，不是 K8s。若 local PV 与反亲和配错导致两个副本落在同一故障域，K8s 无法补救——丢多数派就是丢多数派。K8s 与共识层是两层独立的可用性保障，不能互相替代(见 §22.7 三层验收)。

**误解 3:"StatefulSet 就是为数据库设计的，直接用就好"。** StatefulSet 提供了稳定网络标识与有序 PVC，确实比 Deployment 适合有状态服务，但它对分布式数据库仍有结构性局限(卷模板不可改、强制有序更新引发 Leader 抖动、Pod 配置须相同、无 raft 成员语义)。"StatefulSet 适合数据库"不等于"StatefulSet 能保证数据库安全"：数据库安全还需要 Leader transfer、副本补齐、事务恢复、备份一致性与内部元数据收敛。TiDB Operator v2 与 ob-operator 都选择绕开它直管 Pod，正说明这一点。

**误解 4:"TiDB SQL 层无状态，任意杀 TiDB Pod 对用户完全无感"。** TiDB Server 不持久化用户数据，但持有连接与 session 状态(见 §22.2 源码证据)；连接迁移、长事务与临时状态仍需负载均衡 / 客户端 / 运维策略配合，杀 Pod 会中断其上的活跃连接。

## 22.11 本章结论

1. 两家都用 Operator 模式把内核编排翻译成 K8s CR，但 CR 模型粒度不同：TiDB v1 集中在 `TidbCluster` 单 CR(v2 已 GA，拆为 `Cluster` / `ComponentGroup` / `Instance` 三层并直管 Pod),OceanBase 把 `OBCluster` / `OBZone` / `OBServer` / `OBTenant` 等作为多个一等 CR,Zone 容灾模型与租户直接进 CR。TiDB CR 更围绕组件编排，OceanBase CR 更贴近数据库内部资源对象。
2. StatefulSet 对分布式数据库有结构性局限(卷模板不可改、强制有序更新引发 Leader 重复调度、Pod 配置须相同、无 raft 成员语义)，其有序更新不能替代数据库级滚动升级；TiDB Operator v2 与 ob-operator 都选择直管 Pod 绕开它，而 TiDB v1.6(锁定基准生产主体)仍依赖 StatefulSet,Operator 必须结合数据库 API、成员健康与副本状态执行变更。
3. TiDB 的云原生优势来自组件化与无状态 SQL 层(弹性扩容免数据迁移),OceanBase 因一体化内核所有节点带状态、计算弹性更弱，其 K8s 适配难点在 Tenant / Unit / Resource Pool / Zone / Locality 与 Pod / PVC / topology 的双向映射，而非容器启动本身；据此推测，这也正是其 shared-storage / Bacchus 的改进方向(详见第 7、23 章)。
4. local PV vs cloud disk 是核心取舍：TiKV 推荐 local SSD，用 Raft 副本冗余换本地盘低延迟，代价是单盘不可迁移、强依赖正确的反亲和与 AZ 打散；配错会在节点故障时丢数据或卡 `volume node affinity conflict`。PD placement rule 与 K8s affinity 是两层拓扑控制，不能互相替代。
5. OceanBase 的 Pod IP 强身份是历史 K8s 适配难点：4.2.3.0 前 Pod IP 漂移会让 observer 无法启动，需固定 IP / ClusterIP service / Calico 兜底，体现一体化内核与"Pod 可替换"假设的冲突。
6. K8s 不替代共识层的高可用：数据冗余靠内核 Raft / Paxos,K8s 只负责重建 Pod / 补副本；两层独立，默认 failover 时延(TiDB 约 35min)有意保守以避免抖动触发大规模迁移。验收应分 K8s 层、数据库成员层、业务层三层确认。
7. 托管服务(TiDB Cloud / OceanBase Cloud)在底层引入存算分离：TiDB Serverless 采用三层存储——EC2 instance store 缓存数据文件、EBS 承载延迟敏感的元数据与 WAL、S3 作为主持久层(享 11 9s 持久性);OceanBase Cloud 走 Bacchus shared-storage。这与自建 local PV 部署架构不同，且其内部隔离 / 调度细节未公开，不应由自建 Operator 文档倒推。常见的"TiProxy 持有连接做实例回收"说法不在所引 AWS 官方博客中，目前只能将其推测为未经官方源证实的工程推断，本章对此不作事实断言。 〔文献[13]〕

## 22.12 参考文献

[1] TiDB Operator Overview / Architecture. 官方文档[EB/OL]. https://docs.pingcap.com/tidb-in-kubernetes/stable/architecture/.
 （支撑:§22.2、§22.5 中 TiDB Operator 生命周期、TidbCluster 等 CR、tidb-controller-manager / tidb-scheduler / admission-）
[2] Comparison Between TiDB Operator v2 and v1. 官方文档[EB/OL]. https://docs.pingcap.com/tidb-in-kubernetes/dev/v2-vs-v1/.
 （支撑:§22.2、§22.8 中 StatefulSet 局限与 v2 直管 Pod、Cluster / ComponentGroup / Instance 三层 CRD 的描述。）
[3] Persistent Storage Class Configuration / Backup-Restore CR on Kubernetes. 官方文档[EB/OL]. https://docs.pingcap.com/tidb-in-kubernetes/stable/configure-storage-class/.
 （支撑:§22.2、§22.7、§22.8 中 TiKV 推荐 local SSD、各组件存储选型、local PV 节点故障丢数据，以及 Backup / Restore / BackupSchedule 的 sn）
[4] Automatic Failover. 官方文档[EB/OL]. https://docs.pingcap.com/tidb-in-kubernetes/stable/use-auto-failover/.
 （支撑:§22.2、§22.6、§22.7 中 max-store-down-time(30m)、tikvFailoverPeriod(5m)、failureStores、maxFailoverCount(3)与）
[5] StatefulSets / Volumes: local volume. 官方文档[EB/OL]. https://kubernetes.io/docs/concepts/storage/volumes/.
 （支撑:§22.4、§22.6、§22.8 中 StatefulSet 稳定身份 / 稳定存储 / 有序更新及其局限，以及 local PV nodeAffinity、节点不可用导致 Pod 无法运行的反例。）
[6] Introduction / Architecture | ob-operator. 官方文档[EB/OL]. https://oceanbase.github.io/ob-operator/docs/developer/arch.
 （支撑:§22.3、§22.4 中 OBCluster / OBZone / OBServer / OBTenant 等 CR 列表、API group oceanbase.oceanbase.com、kub）
[7] Create a cluster / Recover from node failure | ob-operator. 官方文档[EB/OL]. https://oceanbase.github.io/ob-operator/docs/manual/ob-operator-user-guide/high-availability/disaster-recovery-of-ob-operator.
 （支撑:§22.3、§22.6、§22.7 中 topology zone→replica 映射、data / redoLog / log 三类存储分离、Pod IP 固定要求(4.2.3.0+ ClusterIP serv）
[8] PALF: Replicated Write-Ahead Logging for Distributed Databases. 论文，VLDB 2024,PVLDB Vol.17 p3745[EB/OL]. https://www.vldb.org/pvldb/vol17/p3745-xu.pdf.
 （支撑:§22.3 中 PALF 作为 OceanBase 日志层、日志盘延迟敏感需单独配置的背景论证(与第 3、6 章一致)。）
[9] pingcap/tidb 内核仓库(release-8.5 @ 67b4876bd57b)— pkg/server/server.go、pkg/util/cgroup/、pkg/util/cgmon/、pkg/metrics/server.go、br/pkg/config/ebs.go 及设计文档 docs/design/2022-07-03-block-storage-snapshot-based-backup-restore.md. 源码[EB/OL]. https://github.com/pingcap/tidb/tree/release-8.5/pkg/util/cgroup.
 （支撑:§22.2 中 TiDB Server 协议 / session 状态、容器自感知(kubepods 匹配)、cgroup→GOMAXPROCS(automaxprocs)、memory.max×0.9 防 OO）
[10] tikv/tikv 与 tikv/pd 内核仓库(release-8.5)— src/storage/config.rs、placement Rule. 源码[EB/OL]. https://github.com/tikv/tikv/tree/release-8.5/src/storage.
 （支撑:§22.2、§22.4 中 TiKV data_dir / engine / block cache / flow control / IO rate limit 与 PV 强绑定判断，以及 PD Rule 含）
[11] oceanbase/oceanbase 内核仓库(v4.2.5_CE @ e7c676806fda)— src/rootserver/ob_unit_manager.cpp、src/share/inner_table/ob_inner_table_schema_constants.h、src/share/ob_errno.def 及 README 跨版本对照. 源码[EB/OL]. https://github.com/oceanbase/oceanbase.
 （支撑:§22.3 中 Unit / Resource Pool 资源模型(ObUnitLoad::get_demand 按 CPU / 内存 / log disk / data disk 读取 UnitConfig)，以及）
[12] TiDB Operator Source Code Reading (I / IV). 官方博客，PingCAP[EB/OL]. https://www.pingcap.com/blog/tidb-operator-source-code-reading-4-implement-component-control-loop/.
 （支撑:§22.2、§22.7、§22.8 中升级前 TiKV member manager 经 BeginEvictLeader 添加 evict-leader scheduler、PD 升级 / 缩容前 transfer）
[13] How PingCAP transformed TiDB into a serverless DBaaS using Amazon S3 and Amazon EBS. 官方博客，AWS Storage Blog[EB/OL]. https://aws.amazon.com/blogs/storage/how-pingcap-transformed-tidb-into-a-serverless-dbaas-using-amazon-s3-and-amazon-ebs/.
 （支撑:仅支撑 §22.11 结论 7 中 TiDB Serverless 的三层存储分层(EC2 instance store 缓存 / EBS 承载元数据 + WAL / S3 主持久层、11 9s 持久性)。该博客不涉及）
## 22.13 信息可信度自评

本章可信度分层如下。**官方文档明确**的部分：TiDB Operator 的 CR 列表与控制组件、StatefulSet / local volume 语义、failover 参数(30m / 5m / 3)、存储选型建议、Backup / Restore 字段、v1.6 现状；ob-operator 的 CR 列表、`topology` 与三类存储、Pod IP / ClusterIP 约束、cert-manager 依赖、节点恢复约束、Tenant 级备份目标(NFS / OSS)——均来自 docs.pingcap.com、kubernetes.io 与 oceanbase.github.io 官方站点。**源码级信息**:TiDB Server 的协议 / session 状态、容器自感知(`kubepods` 匹配)、cgroup→GOMAXPROCS、`memory.max`×0.9、`tidb_server_maxprocs` 指标、EBS 备份内嵌 PV/PVC 模型，均 commit-pinned 到 `pingcap/tidb` `release-8.5`@`67b4876bd57b` 并定位到文件；TiKV `src/storage/config.rs`、PD placement `Rule` pin 到 `tikv/tikv`、`tikv/pd` `release-8.5`;OceanBase Unit / Resource Pool 资源模型与内核 README 对 ob-operator 的指引(4.3.5 / 4.4.x 有、4.2.5_CE 无)pin 到 `oceanbase/oceanbase` `v4.2.5_CE`@`e7c676806fda` 并以 4.4.2_CE 交叉确认。**论文**:PALF(VLDB 2024)用于日志层背景。**官方博客**:TiDB Serverless 的三层存储分层来自 AWS Storage Blog 明确表述；PD / TiKV 升级缩容协同来自 PingCAP 源码阅读博客。**工程推测 / 不确定**:"TiProxy 持连接做实例回收"不在该 AWS 博客内、亦无其它官方源证实，降级为未证实工程推测，不作实现层断言；TiDB Cloud Starter / Essential 与 OceanBase Cloud 的内部隔离与调度机制公开资料不足，不应由自建 Operator 文档倒推；OCP 与 ob-operator 双控制面错配风险为工程推测；v2 直管 Pod 的具体 reconcile 逻辑因算子仓库不在 checkout 范围，以官方文档为准未引内核常量。

**版本核查备注：** 本章引用版本均与锁定基线一致——TiDB 内核 `release-8.5`、OceanBase 内核 4.2.5_CE / 4.3.5 / 4.4.x；源码 commit 统一用 `release-8.5` / `v4.2.5_CE` 基线(OceanBase Unit 资源模型经 4.4.2_CE 交叉确认)。算子版本不在锁定表内，按反查证核实的实际版本与日期书写并注明查证时间窗(2026-06):ob-operator 2.3.4 发布日期为 2026-01-13(GitHub Releases API `published_at` 与官方 changelog 权威，早先一次网页抓取曾误读为 2025-01-13，逻辑反证前序 2.3.3 发布于 2025-09-08、2.3.4 不可能早于其 8 个月，已修正);TiDB Operator v2 已 GA(v2.0.0 = 2025-12-05、v2.0.1 = 2026-03-25,`main` 演进至 v2.1 beta / v2.2 alpha),v1.6 仍是当前 v1 稳定线；关于 ob-operator 文档版本号(英文文档曾出现 V2.3.1 与部署页 Helm `2.1.0` 示例并存)，以最新 release 2.3.4 与官方 changelog 为准，Helm 示例视为滞后样例不作版本事实。未发现与锁定表冲突的内核版本号。

---


# 第 23 章 存储池化 / Shared-storage

## 23.1 本章核心问题

Shared-storage(存储池化)讨论的不是“SQL 层能不能横向扩展”这么简单，而是更底层的一组结构性问题：数据库把持久化状态放在哪里、日志如何成为恢复边界、后台 compaction / merge 是否还和前台事务竞争同一台机器的资源、计算节点失败后能否少搬数据快速恢复。

传统分布式数据库(包括本研究对照的 TiDB classic 与 OceanBase shared-nothing 形态)遵循 **shared-nothing** 范式：每个存储节点拥有本地盘上的一份(副本级)数据，通过共识协议(Raft / Paxos)在多副本间维持强一致。这一范式在自建机房非常自然，但在云上暴露三个结构性矛盾：

1. **成本**：为保证可用性，数据要存 3 副本(乃至跨 AZ)，而云对象存储(S3 / OSS / Azure Blob)单位价格通常远低于云盘(块存储)。Bacchus 论文采用“对象存储单位成本约为云盘 15%”(即便宜约 85%)的**定价建模假设**(§1.2 / 表 3)，3 副本本地盘的单位成本远高于“对象存储 + 缓存”。需注意此 85% 是论文的**通用定价模型假设**(如 EBS gp2 $0.10/GB vs S3 Standard $0.023/GB 的列表价算术)，而非实测生产成本。
2. **弹性**：存算一体下，扩容意味着搬数据、重做副本，扩缩容是“分钟到小时”级而非“秒”级；计算需求(QPS 峰谷)与存储需求(数据量缓慢增长)被强行绑定。
3. **恢复**：节点故障要靠副本重建 / 搬迁恢复，数据量越大恢复越慢。

**Shared-storage** 的核心思路是把“持久化的唯一真相源”下沉到一个共享的存储服务(通常以对象存储为底座)，让计算节点退化为**无状态(stateless)**的“缓存 + 执行”层。这样存储只存一份(由对象存储内部 EC / 多副本保证耐久)，计算节点可秒级增删，扩缩容不再搬数据。

但这条路有其代价。对象存储的**首字节延迟在数十毫秒到 100ms+ 量级**:AWS 官方文档明确 S3 Standard 的小对象 / 首字节延迟约 **100–200ms**(详见 §23.7 交叉验证)，与 OLTP 要求的亚毫秒至毫秒级随机点查根本冲突。于是 shared-storage 数据库的工程难点，几乎都落在**“如何用多级缓存 + 日志服务化 + 后台异步化，把对象存储的高延迟挡在 OLTP 关键路径之外”**。

为了把后文的对比锚定清楚，可以先把 shared-storage 拆成三层问题：

- **第一层是日志**:Aurora、Socrates、Neon、Bacchus 都把日志或 WAL 作为恢复、复制或页面重构的核心，但它们把日志服务、页面服务、对象存储、缓存服务放在不同位置。
- **第二层是数据对象**:TiFlash disaggregated 把 TiFlash 列存数据放到 S3 兼容对象存储；OceanBase shared-storage / Bacchus 论文把 SSTable / macro-block 一类对象放到对象存储，并用多级缓存补偿远端延迟。
- **第三层是云产品化**:TiDB Cloud Starter / Essential、OceanBase Cloud、Serverless 形态需要把弹性、计费、隔离和后台任务服务化，但公开资料通常不会暴露完整的租户隔离和调度实现，所以本章只写公开可证的形态。

这里要先立一条边界，以免把“逻辑池化”误当“物理共享盘”:TiDB classic 的“存储池”更准确地说是**逻辑存储池**。TiKV Store 本地持有 Region 数据与 Raft 状态，PD 通过 Region heartbeat、split、scatter、placement 等路径调度副本和热点；它不是“对象存储承载 SSTable”的 shared-storage。这个边界与第 1 / 3 / 7 / 16 章一致：Region 是 TiDB 的分布与共识边界，TiFlash disaggregated 是 AP 副本路径，**不能反写成“TiKV OLTP 主路径已经 shared-storage”**。

本章解决的问题是：**TiDB 与 OceanBase 各自如何把存储池化?对象存储承载什么、本地盘还留什么、日志是否共享、compaction 谁来做、计算节点如何无状态化?为什么这是云数据库的重要方向，却不一定适合所有 OLTP 场景?** 本章把存算分离的“分析侧”与“无状态 SQL 层”留给第 7 章与第 6 章，聚焦“存储物理池化”这一层；与 K8s 的协同详见第 22 章，与 HTAP 列存的关系详见第 16 章。 〔文献[12]〕

## 23.2 TiDB 的实现

谈 TiDB 的“存储池化”，核心不是它有没有池化，而是把哪三种形态区分清楚——否则极易把“逻辑池化”误当“物理共享盘”。下面按 TiKV、TiFlash disaggregated、TiDB X 依次拆开。

### 23.2.1 TiKV：逻辑存储池，非物理 shared-storage

TiDB classic(TiKV 行存底座)**本质仍是 shared-nothing**。TiKV 将 keyspace 切成 Region，每个 Region 对应一个 Raft Group，多个 Peer 分布在不同 Store 上；每个副本在本地盘上维护一份独立的 RocksDB(详见第 4 章)。PD 负责收集 Region / Store 状态并做调度(详见第 1 章、第 13 章)。所谓“存储池”是**逻辑层面的**:PD 把所有 TiKV 的容量视作一个资源池，通过 Region 的 split / merge / 调度(balance-region、balance-leader、scatter)把数据均摊，应用看到的是“无限容量的 KV 池”，但物理上**没有任何对象存储承载在线数据**。正常写入时，TiDB Server 根据 key range 找到 Region leader，写入经 TiKV Raft 复制后落本地引擎；后台 RocksDB compaction、Raft log、Region split / merge 仍发生在 TiKV Store 侧。这种“池化”带来自动再均衡和容量扩展，但扩容仍需要 Region 迁移、leader 调整和副本补齐，**不能像共享对象存储那样让新计算节点直接挂载同一份持久数据**。

源码反查证(commit-pinned)：在 `tikv/tikv` release-8.5(commit `1f8a140b6d46`)中,`components/external_storage/src/lib.rs` 头注释明确写 “External storage support. Cloud provider backends can be found under components/cloud”；其使用者是 `components/backup/`、`components/sst_importer/`、`components/backup-stream/`,即 **BR 备份、SST 导入、CDC 日志**，而非在线读写引擎。`components/cloud/aws/src/s3.rs` 是 AWS S3 SDK 封装，服务备份与静态加密。控制路径侧,`tikv/pd`(scheduling service)中可见 `RegionHeartbeat`、`SplitRegions`、`ScatterRegions` 等调度入口，印证“池化”发生在 PD 的逻辑调度层。因此：**TiKV 的对象存储能力仅用于备份 / 恢复 / 导入，在线数据仍是每副本本地 RocksDB,TiKV 不是 disaggregated 引擎**。

### 23.2.2 TiFlash disaggregated：分析侧的真 shared-storage

TiDB 的第一个**生产级在线 shared-storage** 是 **TiFlash 存算分离架构**(分析侧，详见第 16 章 HTAP)。其形态与版本：

- **引入**:v7.0.0(实验性)。
- **GA**:**v7.4.0**(release notes 明确 “becomes GA after a series of improvements since its introduction as an experimental feature in v7.0.0”)。

> 版本核查备注：TiDB 官方 release notes 与第 7 章结论一致表明 **v7.0.0 仅实验性引入、v7.4.0 才 GA**(release notes 明文 “becomes GA after a series of improvements since its introduction as an experimental feature in v7.0.0”)。本章统一书写为“v7.0 实验 / v7.4 GA”。

**组件与数据路径**(官方文档 docs.pingcap.com,tiflash-disaggregated-and-s3):TiFlash 进程拆为两类节点，数据可存入 Amazon S3 或 S3 兼容对象存储。

- **Write Node**：从 TiKV 接收 Raft 日志(作为 Learner)，把行数据转列存(DeltaTree / ColumnFile)，周期性把一段时间内的更新打包**上传到 S3**；本地 NVMe 缓存近期写入；管理 S3 上的对象组织与过期删除。
- **Compute Node**:**无状态**查询执行器，从 Write Node 取数据快照(含尚未上传 S3 的最新数据)、从 S3 读大部分历史数据，用本地 NVMe SSD 做缓存避免重复远程读；**秒级伸缩**，无查询时可缩到 0。

源码反查证(计算侧胶水，commit-pinned)：在 `pingcap/tidb` release-8.5(commit `67b4876bd57b`)中可确认计算侧实现：

- `pkg/config/config.go`:`DisaggregatedTiFlash bool`(toml `disaggregated-tiflash`,默认 `false`)、`TiFlashComputeAutoScalerType`(默认 AWS)、`UseAutoScaler` 等。
- `pkg/ddl/placement/common.go`:engine label 体系——`EngineLabelTiFlashCompute = "tiflash_compute"`、`EngineRoleLabelWrite = "write"`,即 Write / Compute 节点拆分在 TiDB 侧由 `engine` 与 `engine_role` 两类 label 体现。
- `pkg/store/copr/batch_coprocessor.go`:存算分离模式下，batch coprocessor 先按 Region location 拆分 range，再按 Compute Node 拓扑用 **一致性哈希(consistent hash)**(亦支持 round-robin)把 MPP / coprocessor 任务派发到 `tiflash_compute` 节点(缓存亲和)，失活时报 “detect aliveness failed, no alive ComputeNode”。
- `pkg/util/tiflashcompute/topo_fetcher.go`:计算节点由 AutoScaler 经 K8s service 动态拉起并返回拓扑(`resume-and-get-topology`)。

这个源码事实只支撑“TiFlash AP 执行路径可转向 Compute Node、由 Write Node 维护 S3 上的列存数据”,**不支撑“TiKV OLTP 数据也在 S3 上直接提交”**。

> 边界说明：TiFlash disaggregated 的**存储侧**(Write Node、S3 上传 / 下载、DeltaTree 上对象存储、共享 S3 page storage)实现在 `pingcap/tiflash` 仓库，**不在本次 ground-truth checkout 中**。本章只能从 TiDB 计算侧 label / 派发胶水间接证明其存在，存储侧实现细节因此标记为（待核实）。

### 23.2.3 TiDB X：行存(OLTP)侧的下一代 shared-storage

TiDB classic 的行存(TiKV)长期是 shared-nothing，但 PingCAP 正在推进 **TiDB X**——把对象存储作为**在线 KV 数据的唯一真相源**。其资料层级在本研究锁定期已从“仅个人博客”上升到“有官方 TiDB Cloud 文档页”，但仍非传统意义的稳定 GA 文档，故关键实现断言一律标注（推测）。

> **版本与产品形态澄清(决定性)**:TiDB X 是 **TiDB Cloud 专有的全新云原生架构**(2025-10-08 SCaiLE Summit 公布，面向 Cloud Starter / Essential / Premium / BYOC,2025 年末起在各 Cloud 层逐步预览 / 可用),**并非随自托管 TiDB 8.5.x / 7.5.x 发行线交付**。锁定基准里的 8.5.x / 7.5.x 恰是经典 shared-nothing 单 LSM(全局 mutex、单 TiKV 节点 ~6TiB 上限)的 LTS 版本——正是 TiDB X 要取代的对象。**不可把 TiDB X 的架构事实锁进 8.5.x / 7.5.x 版本号**。另：本章对比表中曾出现的代号“Bacchus”是 **OceanBase** shared-storage 架构(arXiv 论文)的名称，**与 TiDB X 无关**，切勿混用。

按 TiDB Cloud 官方文档(docs.pingcap.com/tidbcloud,tidb-x-architecture)与 PingCAP 官方博客(2025-10-08 / 2025-12-17)，四项技术要素均为官方文档**明文记载**:

- **对象存储是真相源**:“TiDB X uses object storage, such as Amazon S3, as the single source of truth for all data”，其上是行引擎 + 列引擎构成的共享缓存层。
- **存储引擎重写、告别 RocksDB**：从“每节点一棵大 LSM”改为 **“LSM forest”——每个 Region 一棵独立 LSM 树**，以消除全局 mutex 争用(此设计动机与第 4 / 5 章讨论的 RocksDB DB-mutex 瓶颈一脉相承)。
- **日志**:“Raft log 先持久化到本地盘，Raft WAL 分片在后台上传到对象存储”——即**本地盘仍承载实时 WAL，对象存储承载历史日志与数据**。
- **compaction**:MemTable 满后 Region leader 把 SST 上传对象存储，在**弹性 compaction worker** 上做远程 compaction，再由 TiKV 节点加载回压缩结果——把 compaction 从前台 OLTP 卸载。
- **产品形态**：博客称 TiDB Cloud Starter 已可用、Essential “public preview soon”、Premium / BYOC “later in 2025”。

一个有用的设计旁证是 2023 年 AWS Storage Blog 对 TiDB Serverless 的描述(合作性架构介绍，**非** 2026 年 TiDB X 的生产实现):S3 做最终数据存储，EBS 放 WAL / metadata 等低延迟关键数据，实例本地盘缓存频繁访问数据，并把 Analyze、Compaction、DDL 等后台工作从前台 workload 中剥离。本章仅用它说明“对象存储 + 低延迟层 + 本地缓存 + 后台任务服务化”这一**设计动机**；将其映射到 TiDB X 当前实现仍属推测，不当作实现证明。

> 高风险提醒：TiDB X 的存储引擎重写、LSM forest、远程 compaction worker 等均**未在本次 ground-truth checkout(release-8.5)中**出现——8.5 仍是经典 TiKV。故 TiDB X 的实现层细节属于“有官方 Cloud 架构文档 + 博客明文支撑、但无锁定基准对应稳定版源码”的状态：既不可当作 classic 8.5 的事实，亦不可将其架构归属到 8.5.x / 7.5.x。

把三条路径并起来看：classic OLTP 路径仍是 TiDB Server → TiKV Region leader → Raft → 本地 Store;TiFlash disaggregated 路径是 TiDB MPP / cop task → TiFlash Compute Node → 本地缓存或 S3 数据，由 Write Node 接收并维护列存；TiDB X / Cloud shared-storage 路径公开资料强调对象存储与弹性计算，但内部日志、缓存、隔离、compaction placement 的具体名称仍**需进一步查证**。 〔文献[6-8,14-17]〕

## 23.3 OceanBase 的实现

与 TiDB 把行存与分析侧分阶段池化不同，OceanBase 的 shared-storage 把 TP 与 AP 统一搬上对象存储，即论文 **Bacchus**(arXiv:2602.23571,2026-02-27,PVLDB Vol.20 接收稿；作者 Quanqing Xu、Chuanhui Yang 等，Ant Group)。其工程实现集中在 **4.4.x 分支**，本节实现层断言以 ground-truth checkout(`oceanbase/oceanbase` ref `4.4.x`)与论文交叉验证为准。

OceanBase 4.x classic 仍是 shared-nothing:OBServer 同时承担 SQL、事务、存储、日志等职责，Tenant、Unit、Tablet、Log Stream / LS 与 PALF 共同构成数据分布和恢复边界。OceanBase V4.3.5 官方文档开始明确列出 shared-storage architecture，说明它在 shared-nothing 之上引入存算分离，包含 RW 节点、RO 节点和 SSWriter,SSWriter 负责生成并上传增量与基线数据到对象存储，计算节点只缓存 hot data。官方文档证明 OceanBase 对外已把 SS 作为产品架构描述，但不同版本、CE / 企业版 / Cloud 可用性仍需按版本矩阵确认。

> **commit 溯源说明**:SS 内部实现不绑定到单一 commit 哈希——社区版可见的相关 commit 其提交信息往往与共享存储 / micro cache / SSWriter 无直接关系，不足以作为 SS 实现的源码级出处。故本章 SS 源码断言**按 4.4.x 分支与具体文件路径**定位(如 `src/share/shared_storage/`、`ob_parameter_seed.ipp` 等)，涉及的精确 commit 哈希**需进一步查证**。

### 23.3.1 版本归属与门控(源码反查证)

源码跨分支对照(按分支 + 文件路径定位，**关键证据**):

**表 23-1　版本归属与门控(源码反查证)**

| 分支 | shared_storage 目录 | macro_cache 目录 | `__all_virtual_ss_*` 虚表 | SHARED_STORAGE_MODE |
|---|---|---|---|---|
| TP 4.2.5_CE | 无 | 无 | 无 | 无 |
| AP 4.3.5 | 无 | 无 | 仅 2 个 | 有 `is_shared_storage_mode()` 框架 |
| 融合 4.4.x | `src/share/shared_storage/` | `src/storage/macro_cache/` | 20+ 个 | 完整实现 |

：shared-storage(Bacchus)框架自 **4.3.5 LTS**(首个 AP 场景 LTS)开始接入、**4.4.x** 完整化；4.2.5 完全没有(SS 内部实现基本是 4.4.x 才齐备的——4.2.5_CE 既无 SS 微缓存参数也无 SSLOG / SS 虚表，4.3.5_CE 仅有少量，故在 4.2.5 / 4.3.5 上当作完整事实即版本错配)。

关于 **4.4.1_CE(2025-10-24 发布)**，经 GitHub release notes 全文核对后：

- **可证事实**：发布日期 2025-10-24；支持用 `ALTER TENANT` 将 locality 从 2F 缩减到 **1F(单副本)**；新增租户级参数 **`ls_scale_out_factor`**(每服务节点 log stream 数，默认 1、范围 [1,10])，对应**日志服务横向扩展能力**。这三点均在 release body 中逐字命中。
- **不被 release notes 支撑的断言(已删除 / 降级)**:GitHub release body(中英双语全文 grep)中**不包含** “第二个 LTS”“Shared Storage Architecture”“Azure Blob 支持”等字样。反查证进一步指出：4.4 线真正的 CE LTS 是 **v4.4.2_CE(2026-03 发布，官方明文“作为 LTS 版本，推荐 TP / AP / HTAP 业务”)**;“V4.4.1 is the second LTS version of the Shared Storage Architecture”这一说法来自**企业版 / 云形态**文档，指企业 / 云形态的 V4.4.1,**而非带 `_CE` 后缀的社区版构建**(且 CE 构建中 SS 默认不编译，见下文双重门控)。故“4.4.1_CE 为第二个 SS LTS / 新增 Azure Blob”属 **CE × 企业版的版本-版别错配 + release notes 不支撑**，本章不作此断言；若需“第二个 SS LTS”的定性，应另寻 OceanBase 企业版 / 云官方公告佐证(具体来源需进一步查证)。

**双重门控**:`src/share/ob_server_status.h` 的 `enum ObServerMode` 含 `SHARED_STORAGE_MODE`(与 `NORMAL_MODE` 并列的整服务器启动模式);`GCTX.is_shared_storage_mode()`(`ob_server_struct.h`)实现为 `#ifdef OB_BUILD_SHARED_STORAGE … #else return false; #endif`。即 **shared-storage 是“编译期 `OB_BUILD_SHARED_STORAGE` + 运行期 startup_mode”双门控**；社区版默认不编译该路径，核心类(SSWriter / SSMicroCache 等)在社区版只见声明、虚表与接口，实现位于闭源编译单元。故 OB SS 实现细节统一标注“需进一步查证”。

### 23.3.2 对象存储承载什么：PRIVATE vs SHARED 对象类型

**表 23-2　Bacchus PRIVATE vs SHARED 对象类型落点**

| 对象类型 | 落点 | compaction 层级 | 性质 |
|---|---|---|---|
| PRIVATE_DATA_MACRO | 本地盘 | — | 数据 |
| PRIVATE_META_MACRO | 本地盘 | — | 元数据 |
| PRIVATE_TABLET_META | 本地盘 | — | 元数据 |
| PRIVATE_SLOG_FILE | 本地盘 | — | 日志 |
| PRIVATE_CKPT_FILE | 本地盘 | — | ckpt |
| SHARED_MINI_DATA_MACRO | 对象存储 | mini | 数据 |
| SHARED_MICRO_DATA_MACRO | 对象存储 | micro | 数据 |
| SHARED_MAJOR_DATA_MACRO | 对象存储 | major | 数据 |
| SHARED_MAJOR_META_MACRO | 对象存储 | major | 元数据 |
| SHARED_TABLET_META | 对象存储 | — | 元数据 |

`src/storage/blocksstable/ob_storage_object_type.h`(整枚举在 `OB_BUILD_SHARED_STORAGE` 下)的 `enum class ObStorageObjectType` 把宏块明确区分两类：

- **PRIVATE_***(落 server 本地盘):`PRIVATE_DATA_MACRO` / `PRIVATE_META_MACRO` / `PRIVATE_TABLET_META` / `PRIVATE_SLOG_FILE` / `PRIVATE_CKPT_FILE`——即 **tablet 元数据、slog、checkpoint 仍有 server 私有本地部分**。
- **SHARED_***(落对象存储):`SHARED_MINI_DATA_MACRO` … `SHARED_MAJOR_DATA_MACRO` / `SHARED_MAJOR_META_MACRO` / `SHARED_MICRO_DATA_MACRO` / `SHARED_TABLET_META` 等——mini / minor / major 各 compaction 层级单独成对象类型，数据 / 元数据宏块落对象存储。

论文侧对应：用户数据先写内存 **MemTable**，事务持久化靠 **CLog**(复制式 WAL,Paxos-backed append-only log + 事务状态推进，详见第 3 章);MemTable 达阈值冻结，经 **mini compaction** 生成 **mini SSTable**(称为日志的 “checkpoint”)；为加速弹性，额外引入 **micro compaction / micro SSTable**(更小的 SSTable)尽快推进 checkpoint。**先把 micro / mini SSTable 放在计算节点本地盘缓存，再由某个特定副本在后台上传到对象存储**；随后 **minor compaction** 把对象存储里多个 micro / mini / minor SSTable 合并成单个 minor SSTable，并做宏块级复用控制写放大，合并后 GC 删除输入 SSTable。这里的关键工程意图是：SSTable / macro-block 的 append-only、低成本、大容量特性天然契合对象存储，而把对象存储放在冷 / 持久层、由日志服务和缓存保护事务路径——Bacchus 的核心不是“对象存储很快”，而是“别让对象存储延迟进提交关键路径”。

### 23.3.3 日志如何共享：服务化 PALF + SSLOG

这是 Bacchus 区别于 Aurora / PolarDB 的核心创新。**PALF**(Paxos-backed Append-only Log File system，首见于 PALF VLDB 2024 论文)在 shared-nothing 形态是每节点的复制式 WAL(详见第 3 章);Bacchus 把它**服务化**：数据日志和元数据日志(journal)都进入 shared log service，避免每个计算节点都持有完整冗余日志，同时避免每次事务提交都直接打到高延迟对象存储。

：“多个分区共享一个 log stream”，日志在共享存储层的 **LogServer 节点托管的 PALF 服务**里持久化与同步(三独立副本，PALF 达成共识)。前台事务的 CLog 先写**本地缓存(ECS 云盘)**取低延迟，leader 把历史 CLog 文件搬到日志服务；为 PITR 做近实时归档(Append + MultiUpload 增量上传)。**写并发用“分片单写者”模型而非真多写**：每个 log stream 只有一个 leader 写 CLog，消除日志级写冲突，多个 RW 节点可各自为不同 log stream 当 leader 实现横向扩展(对应 §23.3.1 的 `ls_scale_out_factor`)。

源码侧对应:`src/share/ob_ls_id.h` 定义 `SSLOG_LS_ID = 1001`(注释 “SSLOG LS for meta or sys tenant”)、`is_sslog_ls()`、`static const ObLSID SSLOG_LS`;`ob_ls_location_service.cpp` 在 SS 模式下 `ls_id = is_shared_storage_mode() ? SSLOG_LS : SYS_LS`。即 SS 模式引入专用 **SSLOG Log Stream(LS 1001)** 承载共享元数据日志。

> **日志是否仍落本地盘?**：**是，部分落本地**。CLog 先写本地缓存 / 云盘取低延迟，历史日志归档进共享日志服务;`PRIVATE_SLOG_FILE` / `PRIVATE_CKPT_FILE` 对象类型佐证 slog / checkpoint 有 server 私有本地部分。共享日志服务与本地实时日志之间的精确边界**需进一步查证**。

### 23.3.4 SSWriter：对象存储无互斥能力的破解

日志共享之外，数据宏块的写入同样要面对一个底层约束：对象存储接口**缺乏互斥能力**，多 server 同时写同一共享对象会冲突。Bacchus 的解法是 **SSWriter(Shared Storage Writer)单写者租约**。

：每个 log stream 的 leader 选一个负载较低的副本作 SSWriter;**在租约有效期内，只有该副本能为该 log stream 下所有 tablet 执行对象存储写任务**。所有共享 tablet 元数据修改都经 SSWriter,SSWriter 把变更广播给其他节点。GC 也用类似的 lease-based 协调(GC Coordinator 周期性向 SSLog 写带过期时间的租约记录，失效则重选)。

源码侧：虚表 `__all_virtual_sswriter_lease_mgr`、`__all_virtual_sswriter_group_stat` 暴露租约 / 分组状态;`ob_storage_rpc`、`ob_ls.h`、`ob_checkpoint_service.cpp` 引用 `ObSSWriter / sswriter_lease`。源码搜索也显示部分 `storage/shared_storage/...` 头文件来自未完整公开的构建路径，核心实现在闭源编译单元，仅确认到模块 / 虚表级——因此本章只写“源码可见接口、观测入口和参数模板”，不编造 closed module 函数名或内部 metric。

### 23.3.5 三级缓存：memory / local cache / distributed cache

Bacchus 用**三级缓存**桥接对象存储高延迟与 OLTP 性能：

1. **Memory cache**(最热，micro-block / index);
2. **Local cache**(计算节点本地盘，次热，并承担节点级快速恢复)：空间分三部分——tablet 元数据缓存、dump / 临时文件缓存、micro-block 缓存。前两类小、通常全缓存；micro-block 缓存用 **ARC(Adaptive Replacement Cache)** 算法，LRU / LFU 的实数据块存本地、ghost 块存对象存储。
3. **Distributed cache(Shared Block Cache Service)**(warm 数据)：由 BlockServer 节点提供、存 macro-block 的**只读分布式缓存**，每个 AZ 部署一组、同 AZ 内 RO / RW 节点共享，去掉节点间冗余副本，减少 RO / RW 扩缩容时的重复拷贝。缓存粒度从本地的 micro-block 递增到分布式的 macro-block。设计借鉴了 AWS S3 Express One Zone 与 **Microsoft Socrates 的两层缓存**(稀疏 + 密集)。

源码侧:`src/share/shared_storage/ob_ss_local_cache_control_mode.h` 的 `ObSSLocalCacheControlMode` 位域三开关(micro / macro read / macro write，各 2 bit)，与论文三级缓存对应；租户级 SS 服务 `ObSSMacroCacheMgr / ObSSMemMacroCache / ObSSMicroCache / ObSSLocalCachePrewarmService / ObSSWriterService`(均 `OB_BUILD_SHARED_STORAGE` 下注册)。缓存淘汰相关参数中,`_ss_micro_cache_arc_limit_percent`=70([10,90])经源码核对成立。

> **参数默认值辨析**:经 4.4.x 分支 `ob_parameter_seed.ipp` 逐字核对,`_ss_micro_cache_size_max_percentage`(磁盘 / size 维度,“percentage of tenant **disk** size used by ss_micro_cache”)默认值为 **`20`**、range **[1, 99]**;名字相近的 `_ss_micro_cache_memory_percentage`(memory 维度,“percentage of tenant **memory** size used by microblock_cache”)默认值同为 **`20`**、range **[1, 50]**。即在当前 4.4.x 源码中,**micro cache 占租户磁盘的默认上限约为 20%**(由 size_max_percentage=20 给出);两参数虽默认值相同,但分属磁盘与内存两个维度、range 不同,不可混用。此前“size_max 默认是 5”的写法与 4.4.x 源码不符,已据源码更正;具体取值仍随版本演进,以对应分支源码为准。

### 23.3.6 compaction 谁来做：卸载到共享存储层

Bacchus 把 compaction 分两类：**Dumping(micro / mini compaction)** 与 **Merging(minor / major compaction)**。关键设计是**把 minor / major compaction 任务从数据库层卸载到共享存储层**执行，由共享存储层的机器跑，实现前后台分离；**major compaction(MC)按对象存储特性设计、分 7 个阶段**，且因 MC 计算密集，共享存储层还能进一步把它卸载到欠载 Resource Pool 的机器(完成后预热到本地缓存、机器归还 Resource Pool)。同理，DDL、backup、recovery 也被列为异步后台服务。把这些后台任务从前台事务节点剥离、可独立扩缩容，是 shared-storage 降低抖动的关键机制之一。 〔文献[1-2,4,9,11,13]〕

## 23.4 核心差异对比

两家的差异不在“是否池化”，而在池化的落点、激进程度与日志归属。下表按 classic 存储、真 shared-storage 落点、对象存储承载、日志、单写者、缓存、compaction、计算无状态化、可用性依赖、源码可证性十个维度分别陈述。

### TiDB 与 OceanBase shared-storage 主对比

**表 23-3　TiDB 与 OceanBase shared-storage 主对比**

| 维度 | TiDB | OceanBase | 影响 |
|---|---|---|---|
| classic 在线存储 | TiKV shared-nothing 本地 RocksDB，对象存储仅备份 / 导入；PD 调度 Region | shared-nothing 本地自研 LSM,OBServer 承载 Tenant / Tablet / LS / PALF,4.2.5 无对象存储 | 两家经典形态均**非**物理 shared-storage |
| 真 shared-storage 落点 | 先在**分析侧**(TiFlash disaggregated,v7.4 GA)；行存侧靠 **TiDB X**(Cloud 预览 / 演进，非 8.5 / 7.5 OSS) | **TP + AP 统一**(Bacchus,4.3.5 接入、4.4.x 完整；CE 真正 LTS 为 4.4.2_CE，见 §23.3.1) | OB 把 OLTP 主路径搬上对象存储更激进 |
| 对象存储承载 | TiFlash：列存 DeltaTree;TiDB X:KV + WAL 分片 | mini / minor / major SSTable(SHARED_* 宏块)+ 共享元数据对象 | 都把“基线 + 增量”持久化下沉对象存储，但不替代低延迟提交路径 |
| 日志归属 | 每节点 Raft WAL;TiDB X 本地 WAL + 后台上传分片 | **服务化 PALF**(LogServer 托管)+ 本地缓存 CLog + SSLOG LS 1001 | OB 日志“服务化共享”,TiDB 仍偏每节点 WAL，决定恢复速度与无状态程度 |
| 单写者协调 | 一致性哈希派发(TiFlash)/ Region leader(TiDB X) | **SSWriter 单写者租约**(对象存储无互斥) | 都需机制规避多写者冲突 |
| 缓存层次 | 本地 NVMe 缓存(TiFlash / TiDB X) | **三级**:memory / local(ARC)/ distributed(BlockServer) | OB 多一层 AZ 级共享分布式缓存；cache 命中率决定对象存储延迟是否暴露给 OLTP |
| compaction / 后台任务 | TiDB X：远程弹性 compaction worker | minor / major **卸载到共享存储层**,MC 7 阶段；DDL / backup / recovery 异步化 | 都把 compaction 移出前台 OLTP，后台任务能否独立扩缩容是降抖动关键 |
| 计算无状态化 | TiFlash Compute Node 秒级伸缩、可缩到 0 | RW / RO 计算节点经共享缓存 + 异步后台无状态化 | 弹性是共同收益 |
| 可用性依赖 | PD / TSO 是关键依赖但非简单单点；TiKV 多数派决定 Region 可用性 | sys tenant / RootService / GTS / PALF 等是关键依赖，亦非简单单点 | 需区分逻辑中心化 / 性能瓶颈 / 元数据依赖 / 故障恢复依赖 |
| 源码可证性 | 计算侧 label / 派发可证(release-8.5)；存储侧在 tiflash 仓库 | 框架 / 虚表 / 参数可证(4.4.x)；核心实现闭源编译开关下 | 两侧都有“可证边界” |

把 TiDB / OceanBase 放进更宽的谱系一起看，能看清各家在“真相源、日志归属、存储引擎”上的不同取法；下表按这几维横向陈列，并刻意在末列标出“不宜直接类比点”，以免用一个 shared-storage 抹平架构差异。 〔文献[3,5,10]〕

### shared-storage 形态横向对比(TiDB / OceanBase / Aurora / Neon / PolarDB / Socrates)

**表 23-4　shared-storage 形态横向对比(TiDB / OceanBase / Aurora / Neon / PolarDB / Socrates)**

| 系统 | 真相源 | 日志(WAL)归属 | 存储引擎 | 计算→存储交互 | 缓存层次 | 不宜直接类比点 |
|---|---|---|---|---|---|---|
| **OceanBase Bacchus** | 对象存储(S3 / OSS / Blob) | 服务化 PALF(LogServer)+ SSLOG | 自研 LSM(SSTable) | 单写者 SSWriter 上传宏块 | memory / local-ARC / 分布式 BlockServer | 论文实验非生产 SLA |
| **TiFlash disaggregated** | S3 / S3 兼容 | TiKV Raft(Learner 拉取) | 列存 DeltaTree | Write Node 上传 / Compute 一致性哈希 | 本地 NVMe | 不改变 TiKV OLTP 主路径 |
| **TiDB X** | 对象存储(S3) | 本地 Raft WAL + 后台上传分片 | LSM forest(每 Region 一棵) | Region leader 上传 SST | 本地缓存 + 共享缓存层 | 不写 Cloud 内部隔离机制 |
| **Amazon Aurora** | 6 副本分布式存储(3 AZ) | redo 下推存储，“日志即数据库” | B-tree(MySQL / PG) | 4/6 写、3/6 读 quorum;10GB 段 | buffer pool + 存储节点物化 | page / redo 架构，非 LSM SSTable |
| **Neon** | 对象存储(S3)+ Pageserver | Safekeeper(Paxos quorum),WAL 为 source of truth | Postgres + 按页重放 | Pageserver 重放 WAL 物化页 | Pageserver 层文件 + 计算本地 | PostgreSQL page / WAL 语义不同 |
| **PolarDB** | PolarFS 共享盘(RDMA) | 共享盘 redo,RW / RO log shipping | B-tree(MySQL / PG) | RDMA + 用户态 I/O(POSIX-like) | buffer pool | 依赖 PolarFS / RDMA，不等同对象存储 |
| **Microsoft Socrates** | XStore(Azure 标准存储) | **独立 XLOG 服务**,log first-class | SQL Server B-tree | Page Server 物化页；LZ / XLOG 分离 | 主存 + SSD(RBPEX)两层 | SQL Server page server 架构不同 |

> 阅读提示：Aurora / PolarDB 是 **B-tree 系**(就地更新，对象存储吸收频繁原地更新困难，Bacchus 论文明确把这列为 B-tree shared-storage 的高并发写瓶颈);Bacchus / TiDB X 是 **LSM 系**(append-only，天然契合对象存储)。Neon / Socrates 都把“日志当一等公民”独立成服务——这与 Bacchus 的“服务化 PALF”思路同源，但 Bacchus 用 Paxos 共识保证日志服务自身高可用，且日志服务同时承载数据日志与元数据日志(journal)，是其自述的差异点。务必注意：这几套系统的 page / redo、WAL、SSTable、PolarFS 语义各不相同，不能只用“shared-storage”一个词替代架构分析。

## 23.5 正常路径图

下面两张图分开刻画 TiDB 与 OceanBase 的正常路径，避免把“TiKV OLTP 主路径”误并入 shared-storage。

TiDB 侧(classic OLTP 与 TiFlash disaggregated 并存):

![[f23_1.svg]]

**图 23-1　存储池化/Shared-storage正常读写/调度路径**

OceanBase Bacchus 正常写 / 读路径：

![[f23_2.svg]]

**图 23-2　存储池化/Shared-storage正常读写/调度路径**

读图要点：TiFlash Compute Node 可读取 S3 与本地 cache，但 **TiKV Region leader 仍是 OLTP 写入关键节点**;Bacchus 则把日志服务和对象存储放进一套 shared-storage，计算节点失败后理论上可通过 shared log、metadata 与 cache 重新接入。正常**读路径**(文字)：计算节点按“memory → local cache(ARC micro-block)→ distributed cache(macro-block)→ 对象存储”逐级回退。命中前三级是毫秒级；只有冷读穿透到对象存储才触发 **100ms+** 延迟。论文生产 trace 中,OLTP 工作负载的 **private macro-block 缓存命中率全程保持 100%**(论文原文 “maintains a 100% hit ratio throughout”),shared macro-block 与整体命中率 **near 100%**、仅工作负载切换时出现“brief, shallow dips”,故对象存储访问被有效压制。注:HTAP 第二组 trace 中,private macro cache 仍“often at 100%”,但 OLAP / 冷读会让 shared / overall 命中率出现可快速恢复的短降——这是论文有意为 OLAP 牺牲部分命中率换成本的设计,而非缺陷。

## 23.6 故障 / 异常路径图

按系统分支组织异常路径(对象存储 / cache / 计算节点 / 控制面抖动):

![[f23_3.svg]]

**图 23-3　存储池化/Shared-storage故障/异常路径**

Bacchus 内部几条更细的异常语义：

![[f23_4.svg]]

**图 23-4　存储池化/Shared-storage故障/异常路径**

**关键异常语义**:

- **切主 vs 双写**：对象存储无互斥，若旧 SSWriter 未感知失联仍写、新 SSWriter 又被选出，会双写。Bacchus 靠**租约有效期 + leader 重选时机控制**避免——这与第 3 章 PALF reconfirm、第 13 章 lease 机制一脉相承。
- **冷启动塌陷**：这是 shared-storage 最突出的结构性弱点。新节点 / 缩容回弹后缓存全空，读全穿透对象存储(100ms+),OLTP 在线用户会经历“分钟级”降级，直到预热完成(详见 §23.8)。
- **“无状态”不是“没有状态”**：异常路径最容易写错的就是“无状态”。TiDB X 与 Bacchus 都把计算节点朝无状态化推进，但 SQL session、事务上下文、缓存热度、未完成后台任务、拓扑发现和元数据租约仍会影响恢复。无状态化的准确含义是“可恢复的持久状态尽量移到共享服务或对象存储，节点本地只留可丢弃或可重建的状态”。
- **major merge 抖动**：与 shared-nothing 形态同源(详见第 4 章 OceanBase major freeze 全局协调),shared-storage 下额外叠加“基线在对象存储、需预热回本地”的成本，版本切换期未预热会短暂命中率下降→延迟抖动。

## 23.7 性能、可靠性、运维影响

**延迟**：核心矛盾是对象存储延迟。独立来源交叉验证：AWS 官方《Best practices design patterns: optimizing Amazon S3 performance》明确——对延迟敏感的应用在 S3 Standard 上可达到“小对象延迟(及大对象首字节延迟)**roughly 100–200 milliseconds**”，并指出若要单位数毫秒延迟需改用 CloudFront / ElastiCache 缓存或 S3 Express One Zone;Bacchus 论文亦自述对象存储延迟 “100ms+”、本地缓存 “ms+”。p99 因供应商内部重试 / 限流会进一步抬高(数据库层无法消除)。结论：**只要命中多级缓存就是毫秒级，一旦 cold miss 穿透对象存储就是约百毫秒级**。因此 shared-storage 的延迟分布是**双峰的**——这正是它“不一定适合所有 OLTP”的根因。相对地，TiFlash disaggregated 对 AP 查询更自然：scan / aggregation 可以通过预取、批量 I/O、本地 cache 与 MPP 摊薄对象存储成本；OLTP 单行更新则更依赖日志服务和低延迟缓存层。

**吞吐**：写吞吐靠 LSM append + 本地缓存 + 异步上传维持。论文 OLTP 实验(SysBench)中：

- vs HBase 1.4.7(500GB 数据集)：平均写吞吐 **3,770.86 vs 3,993.79 ops/s**(大致相当)，但 HBase 出现间歇性“写跌零”,Bacchus 靠自研快速 dump 策略完全消除写 stall。
- vs 多 HBase 变体(16C32G、200 并发、5 亿记录):PUT 实验 Bacchus 总 TPS **51,612**、单节点均 **25,806 TPS**，均值 / 分位延迟均最优；GET 实验两节点吞吐略低于三节点 Tencent HBase，但单节点 QPS **28,208** 领先。
- vs PolarDB MySQL 8.0.1 / Aurora 3.08(32C128G)：高并发(400–1500 线程)下 Bacchus TPS 显著更高、Insert / Update 高线程下超 Aurora **2×**，但**高线程下延迟爬升更陡**,Aurora 在读密集负载 RT 更稳。

> benchmark 局限：以上为**论文实验数据**，绑定特定硬件(阿里云 ecs.r7.2xlarge / AWS db.r5.xlarge / Intel 8575C 等)、特定版本(OB 4.4.x vs HBase 1.4.7 / PolarDB 8.0.1 / Aurora 3.08 / StarRocks 3.3.13)、特定并发与数据规模，**非官方审计、利益相关方为 OceanBase**，不可外推为“生产事实”或绝对排名。

**成本**：论文给出的核心数字是——相比非共享存储的 OceanBase,Bacchus **存储成本下降 OLTP 约 59%、OLAP 约 89%**；底层杠杆是对象存储单位价格约为传统云盘的 15%(便宜约 **85%**)。**但必须强调这不是生产实测成本**：(1) 59% / 89% 出自论文 §7.5 “Storage Cost Comparison” / 表 3，是**单一合成规模(100TB)下的列表价算术**(EBS gp2 $0.10/GB vs S3 Standard $0.023/GB)，并叠加副本 3→1 的建模假设，而非真实账单；(2) 基线是 **OceanBase-vs-OceanBase**(新 Bacchus 共享存储 vs 旧 shared-nothing OceanBase)，降本主要由“砍副本 3→1”这一架构因素驱动，并非纯存储介质价差；(3) 论文唯一的生产侧数据只覆盖缓存命中率，不涉及成本。故 shared-storage 的成本 / 弹性是**方向性收益确定**，但具体百分比是论文建模值、绑定 OceanBase 4.4.x,**不可外推为 TiDB / TiFlash 或任意生产环境的承诺**。

**可用性 / 恢复**：论文称 Bacchus 通过无状态计算节点 + 快速增量持久化达 **RPO=0**(日志多数派 Paxos 持久化保证)。需注意该 RPO=0 仅限 V4.x LS / Paxos 多副本、少数派故障或多数派完整场景，不外推到单副本 shared-storage 或跨云。**单副本数据 + 共享日志**形态下，计算节点故障的实际 RTO 取决于新节点缓存预热速度——冷启动期延迟塌陷意味着“恢复服务”≠“恢复性能”。可靠性维度还要注意：shared-storage 把一部分本地磁盘故障风险转移给对象存储和共享服务，但引入新依赖——对象存储 API、bucket 权限、跨 AZ 网络、cache service、shared log service、元数据服务。Socrates 的关键启发是把 log 与 storage 分离、把 durability 与 availability 拆开分析；Bacchus 也把 PALF shared log 单独服务化。

**扩展性 / 弹性**：扩缩容免拷贝数据(存储在对象存储只一份)，计算 / 缓存 / 存储可独立伸缩；缓存节点扩缩不影响计算节点。这是 shared-storage 相对 shared-nothing 的结构性优势。但能否线性扩展仍受热点 key、cache 命中、日志服务吞吐和对象存储带宽限制——不是“把文件放 S3”就自动线性。

**运维复杂度**：多了对象存储、日志服务(LogServer)、分布式缓存(BlockServer)三类外部依赖，以及缓存预热、SSWriter 租约、SSLog GC 等新运维面。复杂度并未消失，而是从“搬数据 / 补副本”转移到“管缓存 / 管日志服务 / 调对象存储”。一次 P99 抖动可能来自 SQL plan、cache miss、distributed cache 队列、对象存储限流、background compaction、log service backpressure、调度 / 配额等多处，“多层 I/O 归因”的难度上升。

把上面几条影响落到具体路径上，可以看出瓶颈不是消失而是迁移。下表按点查、写提交、扩容、恢复、成本、后台任务逐路径对比 shared-nothing 与 shared-storage，末列点明瓶颈往哪迁。

### 性能瓶颈来源对比(shared-nothing vs shared-storage)

**表 23-5　性能瓶颈来源对比(shared-nothing vs shared-storage)**

| 路径 | shared-nothing(经典) | shared-storage(Bacchus / TiDB X) | 瓶颈迁移 |
|---|---|---|---|
| 点查命中 | 本地盘 LSM，毫秒级 | 多级缓存命中，毫秒级 | 持平 |
| 点查 cold miss | 本地盘，毫秒级 | **对象存储 100ms+** | 显著恶化 |
| 写提交 | 本地 WAL fsync + 多数派 | 本地缓存 CLog + 服务化 PALF 多数派 | 大致持平 |
| 扩容 | 搬 Region / 补副本，分钟–小时 | 拉计算节点，秒级 | 显著改善 |
| 故障恢复 | 副本重建 / 切主 | 切主 + 缓存预热 | “可用快、性能慢” |
| 存储成本 | 3 副本本地盘 | 对象存储单份(便宜约 85%，论文建模值) | 显著改善(方向性) |
| 后台任务 | 与前台共节点资源 | compaction / DDL / backup 卸载到共享层 | 抖动可控性改善 |

## 23.8 反例与代价

shared-storage 的收益是方向性的，代价却往往是结构性的。下面六条反例不是要否定它，而是划清它“不适合”的边界——每一条都对应一个无法靠调参消除的取舍。

**1. 无冷热区分、随机访问全集的纯 OLTP 不适合**。OLTP 写模式是“对全数据集分散行的随机更新”，缓存局部性差；一次 cache miss 穿透对象存储就要承受**约百毫秒级**首字节延迟(AWS 官方：S3 Standard 小对象 / 首字节延迟 roughly 100–200ms)，足以破坏亚毫秒 p99 SLA。对象存储为大对象顺序吞吐设计，而 OLTP 是 KB 级页随机小 I/O，缓存不命中时二者契合度差。值得注意的是 Bacchus 论文**自己**也把目标场景限定为**“有明显冷热区分的 OLTP”**(历史库、时序、多模)，而非任意 OLTP——这是关键的自我设限，不可忽略；不过其外推到一般 OLTP 的边界证据有限，本章只取论文明确给出的范围。这条反例的架构层背景(disaggregated OLTP 中“网络成为系统瓶颈”)在第三方综述中亦有讨论，但具体延迟惩罚量级以 AWS 官方延迟文档与论文自述为据。

**2. 冷启动 / 缩容回弹的性能塌陷**。缩到 0 或故障切换后缓存全空，每个操作都吃满对象存储延迟，在线用户经历分钟级降级。Bacchus 用多种预热(Baseline Switching / Leader-Follower / Replication Migration / Cloud Disk Scaling)缓解，但**预热本身需要时间和带宽，不可能瞬时**。

**3. 尾延迟不可控**。对象存储 p99 受供应商内部限流 / 重试影响(尖刺可达数百毫秒级)，数据库层无法消除，只能靠缓存“赌”不 miss。对延迟抖动敏感的金融级短事务(如支付 50ms p99 SLA)无法承受。

**4. 架构不可原地切换**。TiFlash 官方文档明确：disaggregated 与 coupled 架构**不能原地切换、不能在同一集群混用**，迁移需重做数据复制。OceanBase 4.4.1_CE 也仅支持新建集群、不支持从旧版本升级到 SS 模式。这是重大的部署刚性代价。对 classic TiDB 已稳定承载的业务，直接迁到 TiDB X 或 TiFlash disaggregated 不能只看存储成本，还要考虑内部隔离、后台任务、恢复路径等公开资料尚有限的部分。

**5. OLAP 也有反例**。论文 TPC-H 100GB 中 Bacchus 整体比 StarRocks 3.3.13 快约 48.5%、Q14 提速 327.93%，但 **Q9 慢 35.71%**(Q9 涉及分组 + 排序 + 聚合 + 子查询，复杂度高);TPC-H 1TB 热运行仅边际改善(Q9 仅 +3.0%);ClickBench 整体提速约 89% 但在小部分查询上劣于 StarRocks。**收益并非全面占优，存在确定的回退点**。

**6. 多云一致性能并非“放 S3”那么简单**。使用通用对象存储可避免部分 vendor lock-in，但日志服务、cache service、控制面、权限模型、性能等级仍会绑定具体云环境。可以推测，真正跨云一致性能的难点在网络、对象存储语义、限流和故障域，而非“把文件放 S3”。

**取舍总结**:shared-storage 用“对象存储便宜 + 弹性免搬数据”换“延迟取决于缓存命中 + 部署刚性 + 单副本恢复退化”。对延迟敏感、无冷热区分、要求稳定 p99 的核心 OLTP，经典 shared-nothing(本地 NVMe)仍是更稳妥选择——这与第 7 章存算分离结论一致。 〔文献[18]〕

## 23.9 测试开发视角的验证点

测试 shared-storage，核心不是验证“能不能跑通”，而是验证“缓存失效时延迟与命中率如何回升”——双峰延迟与冷启动塌陷才是真正要打的靶。下面按功能场景、失效注入、压测指标三组组织。

**功能场景按三组分测**:

- **TiDB classic**:Region split / merge、PD scatter、hot Region、leader transfer、TiKV Store 下线与 snapshot 补副本。
- **TiFlash disaggregated**:Compute Node 扩缩容(含缩到 0 再扩容，验证秒级恢复)、Write Node 故障、S3 bucket 权限错误、Compute local cache 清空、MPP task retry、coupled / disaggregated 配置误用。
- **OceanBase SS / Bacchus**:RW / RO 节点切换、SSWriter 上传阻塞、PALF shared log 抖动、local persistent cache 清空后命中率预热曲线、distributed cache 节点扩缩容(验证同 AZ RO / RW 命中同一 BlockServer 缓存、去冗余)、对象存储限流和后台 compaction backlog。其中“写入后冷读穿透”是核心用例：清空缓存或新建计算节点后立即点查，验证延迟从毫秒跳到 100ms+，确认多级缓存逐级回退路径正确。

**失效注入按路径来**:

- **网络层**：对象存储高延迟 / 5xx / 限流注入，验证尾延迟放大与缓存吸收能力。
- **缓存层**:cold cache、cache server 重启，验证预热是否生效。
- **日志层**:leader 切换、append 延迟注入，验证恢复进度与命中率回升曲线。
- **单写者**:SSWriter 租约持有者网络分区，验证租约过期后接管、无双写(对象一致性)。
- **控制面**：元数据查询变慢。
- **后台任务层**:compaction / merge 与 DDL 同时运行，验证抖动幅度。

**关键压测指标**:P50 / P99 / P999 延迟、cache hit ratio、远端对象存储请求比例、写吞吐(ops/s)在持续压力下是否“写跌零”(Bacchus 强调消除写 stall)、日志 append / commit 延迟、background compaction backlog、Region / LS 迁移速率、恢复时间、以及扩缩容期间前台吞吐跌幅与冷 / 热运行查询总时长对比(论文方法学)。

**关键观测指标 / 内部视图(真实可查证，不编造)**:

- OceanBase 4.4.x SS 系统视图:`__all_virtual_ss_local_cache_info`(本地缓存)、`__all_virtual_ss_object_type_io_stat`(按对象类型 IO 统计)、`__all_virtual_ss_gc_status` / `__all_virtual_ss_gc_detect_info`(GC)、`__all_virtual_sswriter_lease_mgr` / `__all_virtual_sswriter_group_stat`(SSWriter 租约 / 分组)、`__all_virtual_ss_tablet_meta` / `__all_virtual_ss_sstable_mgr` / `__all_virtual_ss_ls_meta` 等。
- OceanBase SS 关键配置项与默认值:`_ss_micro_cache_arc_limit_percent`=70([10,90]),`_object_storage_io_timeout`=20s([1s,1200s])。对照普通模式 `_data_storage_io_timeout`=10s——**对象存储 IO 超时(20s)显著大于本地存储(10s)，从参数即可印证对象存储延迟更高**。其余 SS 缓存参数中,经 4.4.x 分支逐字核对:`_ss_micro_cache_size_max_percentage`(磁盘 / size 维度)默认 **20**、range [1, 99];`_ss_micro_cache_memory_percentage`(memory 维度)默认 **20**、range [1, 50];`_ss_macro_cache_miss_threshold_for_prefetch`(prefetch 阈值)默认 **10**、range [1, 10000]。两个 `_ss_micro_cache_*` 百分比虽默认值相同,但分属磁盘与内存两个维度、range 不同,不可混用(此前“size_max 默认是 5”与源码不符,已更正)。具体取值随版本演进,引用时以对应分支源码为准。`_ss_local_cache_control`(三缓存开关，见 §23.3.5 位域)经源码确认存在。
- TiDB 侧：配置 `disaggregated-tiflash`、`flash.disaggregated_mode`(`tiflash_write` / `tiflash_compute`)、`storage.remote.cache.dir` / `storage.remote.cache.capacity`(Compute Node 本地缓存)、`storage.s3.*`;引擎 label `engine=tiflash_compute`、`engine_role=write`;以及 `tiflash_compute` 调度变量与 topology cache 失效路径。
- 具体 Prometheus metric / PromQL 裸名、TiFlash disaggregated 的缓存命中率 / 远程读 metric 名：**需进一步查证**，不编造；线上观测名称以对应版本文档和 dashboard 为准。

## 23.10 容易误解点

**误解 1:“TiDB classic 已经是 shared-storage / 存算分离”。** 纠正：TiDB classic 的 **TiKV 行存(release-8.5)仍是 shared-nothing 本地 RocksDB + Region / Raft**,`external_storage` 只服务备份 / 导入。真正的在线 shared-storage 是 **TiFlash disaggregated(分析侧，v7.4 GA)** 与 **TiDB X(行存侧，Cloud 预览 / 演进，标)**。“无状态 SQL 层”(第 6 章)、“存算分离”(第 7 章)、“存储物理池化”(本章)是三件不同的事，不可混为一谈。

**误解 2:“对象存储越便宜，OLTP 就越适合 / shared-storage = 数据搬上对象存储就行”。** 纠正：对象存储适合容量与持久层，低延迟 OLTP 需要日志服务、本地缓存、分布式缓存、批量 I/O 与单写者协调共同保护。难点几乎全在**如何把对象存储 100ms+ 延迟挡在 OLTP 关键路径外**——多级缓存、日志服务化、SSWriter 单写者、compaction 卸载、缓存预热缺一不可。仅把数据丢上 S3 而无这套机制，OLTP 性能会塌陷。

**误解 3：把 Bacchus 论文成本 / 性能数字当生产事实。** 纠正：“降本 OLTP 59% / OLAP 89%”“对象存储便宜 85%”并非生产实测，而是论文 §7.5 / 表 3 的**定价建模**(固定 100TB 列表价算术 + 副本 3→1 假设，基线为 OceanBase-vs-OceanBase);“超 Aurora 2×”等是 SysBench / TPC-H **benchmark 数据**，绑定 OB 4.4.x 特定硬件 / 版本 / 并发，利益相关方为 OceanBase，存在 Q9 慢 35.71% 等明确反例。既不可作为“生产环境一定降本 59%”的承诺，也不可套用到 TiDB / TiFlash(版本 / 产品不匹配)。

**误解 4:“shared-storage 延迟更高，所以更差”。** 纠正：这是把“协议理论”当“工程实现”。命中缓存时毫秒级，与 shared-nothing 持平；只有 cold miss 才高延迟。是否“更差”取决于**工作负载的冷热区分度与缓存命中率**，而非架构本身——这是工程取舍，不是优劣排序。

## 23.11 本章结论

1. **TiDB 与 OceanBase 的经典形态(TiKV release-8.5 / OceanBase 4.2.5)都不是物理 shared-storage**，而是 shared-nothing 本地盘：TiKV 的对象存储能力仅服务备份 / 导入(源码反查证),OBServer 本地承载 Tenant / Tablet / LS / PALF 与存储引擎，所谓“存储池”是 PD / 调度逻辑层面的逻辑池，扩容仍涉及 Region / 副本迁移重平衡。

2. **真正的在线 shared-storage 在两家落点不同**:TiDB 先在分析侧落地(TiFlash disaggregated,v7.0 实验、**v7.4 GA**,Write / Compute 拆分 + S3)，行存 OLTP 侧靠 **TiDB X**(对象存储为真相源、LSM forest、远程 compaction；有官方 Cloud 架构文档明文，但属 **TiDB Cloud 专有的预览 / 演进架构，不随自托管 8.5.x / 7.5.x 发行线交付**，不可锁进这些 OSS 版本号);OceanBase 的 **Bacchus** 更激进地把 TP + AP 统一搬上对象存储(4.3.5 接入、4.4.x 完整；4.4 线 CE 的真正 LTS 是 **4.4.2_CE**——“4.4.1_CE 为第二个 SS LTS”经反查证为 CE × 企业版的版本错配，已更正)。

3. **Bacchus 的三大工程支柱**:(a)**服务化 PALF 共享日志**(LogServer 托管、多分区共享 log stream、CLog 先写本地缓存再归档、SSLOG LS 1001 承载共享元数据);(b)**SSWriter 单写者租约**破解对象存储无互斥；(c)**三级缓存**(memory / local-ARC / 分布式 BlockServer)+ 多种预热 + compaction 卸载到共享存储层。数据落 SHARED_* 宏块对象，slog / checkpoint 仍有 PRIVATE 本地部分。社区版 SS 默认不编译(`OB_BUILD_SHARED_STORAGE`)，核心实现仅确认到模块 / 声明 / 虚表 / 参数级。

4. **shared-storage 的确定收益是成本与弹性方向**(对象存储单位价格约为云盘 15%；扩缩容免搬数据);**确定代价是延迟取决于缓存命中**(对象存储约百毫秒级、本地 ms+，延迟分布双峰，AWS 官方文档佐证 S3 Standard 首字节 ~100–200ms)，且存在冷启动塌陷、尾延迟不可控、架构不可原地切换、单副本恢复“可用快但性能慢”。Bacchus 论文称降本 OLTP 59% / OLAP 89% / 对象存储便宜 85% 均为论文 §7.5 / 表 3 的定价建模 / 单点假设(100TB 列表价 + 副本 3→1,OB-vs-OB 基线，绑定 4.4.x)，非生产实测，不可外推为 TiDB 或任意生产承诺。

5. **不一定适合所有 OLTP**：对无冷热区分、随机访问全集、要求稳定 p99 的核心金融级短事务，shared-storage 的 cache miss 会摧毁亚毫秒 SLA;Bacchus 论文自身也把目标限定为“有明显冷热区分的 OLTP”，且含 Q9 慢 35.71% 等明确反例。据此可以推测，经典 shared-nothing(本地 NVMe)在此类场景仍更稳妥。

6. **与同类系统的谱系**:Bacchus / TiDB X 是 **LSM 系**(append-only 契合对象存储),Aurora / PolarDB 是 **B-tree 系**(就地更新);Neon / Socrates 与 Bacchus 同样把“日志当一等公民”独立成服务，但 Bacchus 用 Paxos(PALF)保证日志服务自身高可用且统一承载数据日志与元数据日志，是其自述差异点。这几套系统的 page / redo、WAL、SSTable、PolarFS 语义不同，不能只用“shared-storage”一个词替代架构分析。

7. **可用性依赖需分维度看，不下单点定论**:OceanBase SS 形态下 sys tenant / RootService / GTS / PALF 等是关键依赖，但应按逻辑中心化、性能瓶颈、高可用单点、元数据依赖、故障恢复依赖五个维度分别讨论；论文 RPO=0 / RTO 指标仅限 V4.x LS / Paxos 多副本、少数派故障或多数派完整场景，不外推单副本 shared-storage 或跨云。

## 23.12 参考文献

[1] OceanBase Bacchus: a High-Performance and Scalable Cloud-Native Shared Storage Architecture for Multi-Cloud. 论文，arXiv:2602.23571,2026-02,PVLDB Vol.20[EB/OL]. https://arxiv.org/abs/2602.23571.
 （支撑:§23.3 全节(对象存储承载 SSTable、服务化 PALF、SSWriter、三级缓存、compaction 卸载)、§23.7 成本 / 性能数据(59% / 89%、85%、vs HBase / Aurora）
[2] PALF: Replicated Write-Ahead Logging for Distributed Databases. 论文，VLDB 2024[EB/OL]. https://www.vldb.org/pvldb/vol17/p3745-xu.pdf.
 （支撑:§23.3.3 PALF 作为共享日志服务底座的共识与复制式 WAL 语义、§23.6 切主 reconfirm 语义。）
[3] Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases. 论文，SIGMOD 2017[EB/OL]. https://pages.cs.wisc.edu/~yxy/cs764-f20/papers/aurora-sigmod-17.pdf.
 （支撑:§23.4 表 23-3 对照(日志即数据库、redo 下推、6 副本 4/6 写 3/6 读 quorum、10GB 段)与 §23.1 shared-storage 背景。）
[4] Socrates: The New SQL Server in the Cloud. 论文，SIGMOD 2019[EB/OL]. https://www.microsoft.com/en-us/research/wp-content/uploads/2019/05/socrates.pdf.
 （支撑:§23.3.5 / §23.4 / §23.7 对照(独立 XLOG 日志服务、Page Server、两层缓存——Bacchus 明确借鉴其两层缓存；log 与 storage 分离、durability 与 avai）
[5] Cloud-Native Database Systems at Alibaba: Opportunities and Challenges. 论文[EB/OL]. https://users.cs.utah.edu/~lifeifei/papers/vldb-cloud-native.pdf.
 （支撑:§23.4 表 23-3 中 PolarDB 采用 shared-storage、PolarProxy、RW / RO、PolarFS / RDMA 的背景对照。）
[6] TiFlash Disaggregated Storage and Compute Architecture and S3 Support. 官方文档，docs.pingcap.com[EB/OL]. https://docs.pingcap.com/tidb/stable/tiflash-disaggregated-and-s3/.
 （支撑:§23.2.2 Write Node / Compute Node 职责、S3 用法、架构不可原地切换 / 混用限制(§23.8)、配置项。）
[7] Configure TiFlash. 官方文档，docs.pingcap.com[EB/OL]. https://docs.pingcap.com/tidb/stable/tiflash-configuration/.
 （支撑:§23.2 / §23.9 中 TiFlash disaggregated 模式下 storage.s3、storage.remote.cache、disaggregated_mode 等配置项的版本化表述。）
[8] TiDB X Architecture. 官方文档，docs.pingcap.com/tidbcloud[EB/OL]. https://docs.pingcap.com/tidbcloud/tidb-x-architecture/.
 （支撑:§23.2.3 TiDB X 以对象存储为真相源、LSM forest、本地 WAL + 后台上传、远程 compaction、产品形态(Starter 可用)。四项技术要素为官方明文，但属 TiDB Cloud 专有预）
[9] Shared storage architecture / Multi-level caching in shared storage. 官方文档，en.oceanbase.com/docs[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000003461903.
 （支撑:§23.3 / §23.4 中 OceanBase shared-storage 的 RW / RO / SSWriter、对象存储与 hot data local cache、三级缓存(memory / local /）
[10] Neon architecture. 官方文档，neon.com[EB/OL]. https://neon.com/docs/introduction/architecture-overview.
 （支撑:§23.4 表 23-3 中 Neon 对 compute ephemeral、shared durable storage、WAL as source of truth、object storage foundatio）
[11] Release v4.4.1_CE · oceanbase/oceanbase. 官方文档 / Release,GitHub[EB/OL]. https://github.com/oceanbase/oceanbase/releases/tag/v4.4.1_CE.
 （支撑:§23.3.1 可证事实：4.4.1_CE 发布日期 2025-10-24、ALTER TENANT 将 locality 2F→1F 支持单副本、新增 ls_scale_out_factor 参数(日志服务横向）
[12] Best practices design patterns: optimizing Amazon S3 performance. 官方文档，AWS[EB/OL]. https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html.
 （支撑:§23.1 / §23.7 / §23.8 对象存储延迟量级：AWS 官方明文 S3 Standard 小对象 / 首字节延迟 “roughly 100–200 milliseconds”，需 CloudFront /）
[13] oceanbase/oceanbase(ref 4.4.x)— src/share/shared_storage/、src/storage/macro_cache/、src/storage/blocksstable/ob_storage_object_type.h、src/share/ob_ls_id.h、src/observer/virtual_table/ob_all_virtual_ss_*、src/share/parameter/ob_parameter_seed.ipp、src/storage/blockstore/ob_shared_object_reader_writer.h. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/tree/4.4.x/src/share/shared_storage.
 （支撑:§23.3 PRIVATE / SHARED 对象类型、SHARED_STORAGE_MODE 双门控、SSLOG LS 1001、SSWriter 租约虚表、三级缓存开关与参数、Shared Object read /）
[14] pingcap/tidb(ref release-8.5,commit 67b4876bd57b)— pkg/config/config.go、pkg/ddl/placement/common.go、pkg/store/copr/batch_coprocessor.go、pkg/util/tiflashcompute/topo_fetcher.go. 源码[EB/OL]. https://github.com/pingcap/tidb/tree/release-8.5.
 （支撑:§23.2.2 TiFlash 计算侧胶水：disaggregated 配置、engine / engine_role label、一致性哈希派发、AutoScaler 拓扑。存储侧在 pingcap/tiflash 仓）
[15] tikv/tikv(ref release-8.5,commit 1f8a140b6d46)+ tikv/pd scheduling service — components/external_storage/、components/cloud/aws/、PD RegionHeartbeat / SplitRegions / ScatterRegions. 源码[EB/OL]. https://github.com/tikv/tikv/tree/release-8.5.
 （支撑:§23.2.1 TiKV 为 shared-nothing 本地盘、对象存储仅服务 BR 备份 / SST 导入的反证，以及“逻辑存储池”由 PD 调度层实现的事实。）
[16] Object Storage: The New Backbone of Database Architecture. 官方博客，pingcap.com/blog[EB/OL]. https://www.pingcap.com/blog/object-storage-new-backbone-data-architecture/.
 （支撑:§23.2.3 TiDB X 设计动机(对象存储为真相源、S3 是新网络)；官方博客，TiDB X 论断标，不作为生产内部实现证明。）
[17] How PingCAP transformed TiDB into a serverless DBaaS using Amazon S3 and Amazon EBS. 第三方解读 / AWS 合作博客[EB/OL]. https://aws.amazon.com/blogs/storage/how-pingcap-transformed-tidb-into-a-serverless-dbaas-using-amazon-s3-and-amazon-ebs/.
 （支撑:§23.2.3 中 TiDB Serverless 使用 S3 / EBS / instance store 多层存储与后台任务服务化的公开合作案例；可信度局限是 AWS 合作博客，不代表 TiDB X 当前全部生产实现）
[18] Notes On: Disaggregated OLTP Systems. 第三方综述博客[EB/OL]. https://transactional.blog/notes-on/disaggregated-oltp.
 （支撑:范围已降级：仅支撑 §23.8 “disaggregated OLTP 架构存在、网络成为系统瓶颈”这一弱论点；其原先归因的定量延迟断言经核对页面并不包含，已改引 AWS S3 官方延迟文档与 Bacchus 论文自述。）
## 23.13 信息可信度自评

- **官方文档明确**:TiFlash disaggregated 的 Write / Compute 职责、S3 用法、不可原地切换(docs.pingcap.com);TiFlash v7.4.0 GA(release notes);OceanBase Cloud shared-storage 与多级缓存框架(en.oceanbase.com，抓取曾限流，已与源码 / 论文交叉);4.4.1_CE 发布日期 2025-10-24、2F→1F 单副本、`ls_scale_out_factor` 参数(GitHub release，已据全文核对)；对象存储延迟 ~100–200ms(AWS S3 官方性能文档)。
- **源码级**:TiKV shared-nothing 反证与 PD 调度入口、TiDB TiFlash 计算侧 label / 派发(release-8.5 commit-pinned);OceanBase SS 的 PRIVATE / SHARED 对象类型、SHARED_STORAGE_MODE 双门控、SSLOG LS 1001、SS 虚表名、`_ss_micro_cache_arc_limit_percent`=70、`_object_storage_io_timeout`=20s(4.4.x 分支，按文件路径定位)。核心 SS 类在 `OB_BUILD_SHARED_STORAGE` 编译开关下，社区版仅声明 / 虚表 / 参数可见，实现层仅确认到模块 / 声明级。
- **论文(实验 / 建模数据，非生产事实)**:Bacchus 的服务化 PALF、SSWriter、三级缓存、compaction 卸载来自 arXiv:2602.23571；成本数字 59% / 89% / 85% 是论文 §7.5 / 表 3 的定价建模(100TB 列表价 + 副本 3→1、OB-vs-OB 基线)，性能数字是 SysBench / TPC-H benchmark。绑定 OceanBase 4.4.x 特定硬件 / 版本 / 并发，利益相关方为 OceanBase，不可外推为生产事实，亦不可套用到 TiDB / TiFlash。Aurora / Socrates / PolarDB / Neon 内容主要作架构对照，适合分析取舍，不外推生产 SLA。
- **官方文档(Cloud 预览)/ 工程推测**:TiDB X 的存储引擎重写、LSM forest、远程 compaction worker——有官方 Cloud 架构文档 + PingCAP 博客明文支撑，但属 TiDB Cloud 专有预览架构，不随自托管 8.5.x / 7.5.x 交付，故不可锁进这些 OSS 版本号；TiDB Serverless S3 / EBS 路径来自 AWS 合作博客，作为设计动机背景使用；“shared-storage 不适合无冷热区分纯 OLTP”为论文自述 + AWS 延迟文档交叉的（推测）。
- **仍不确定 / 需进一步查证**:Bacchus 共享日志服务与本地实时日志的精确边界；TiFlash disaggregated 的 Prometheus metric 具体名(实现于 pingcap/tiflash,不在本次 checkout);TiDB X 的日志 / 缓存 / 隔离 / compaction placement 的内部组件名(Cloud 专有,官方文档仅给通用描述);SS 实现的精确 commit 哈希(实现横跨多 commit 且核心在闭源编译单元,不绑定单一哈希);OceanBase Cloud 文档与一体化架构博客 WAF 限流未能逐字核对的部分(已用源码 + 论文交叉补强)。以上一律标“需进一步查证”，不编造。**本轮已补强**:`_ss_micro_cache_*` 参数默认值已据 4.4.x 源码逐字核对(size_max / memory 百分比均默认 20,见 §23.3.5 / §23.9);论文生产 trace 的 OLTP private macro-block 命中率已据论文 §7 核实为 100%(整体 near 100%,见 §23.5)。

**反查证发现的主要冲突与处理**:

1. **TiFlash 存算分离 GA 版本**:TiDB 官方 release notes 与第 7 章结论表明 **v7.0.0 仅实验性引入、v7.4.0 才 GA**，故本章统一书写为“v7.0 实验 / v7.4 GA”，不采用任何“v7.0 起 GA”的口径。
2. **TiDB X 版本 / 产品形态错配(决定性)**:TiDB X 是 TiDB Cloud 专有的全新架构(2025-10-08 SCaiLE Summit 公布)，不随自托管 8.5.x / 7.5.x 发行线交付；8.5 / 7.5 恰是其要取代的经典 shared-nothing 单 LSM。本章把 TiDB X 断言改标，并澄清代号“Bacchus”属 OceanBase 而非 TiDB X。
3. **“4.4.1_CE 为第二个 SS LTS / Azure Blob”错配(决定性)**：经 GitHub release notes 全文核对，v4.4.1_CE 正文不含 “LTS” / “Azure Blob” / “Shared Storage Architecture”;4.4 线 CE 的真正 LTS 是 4.4.2_CE。该断言来自企业版 / 云文档(指企业 / 云形态 V4.4.1)，与带 `_CE` 后缀的社区版构建错配(CE 中 SS 默认不编译)。本章撤回该断言，仅保留 release notes 可证的 2025-10-24 / 2F→1F / `ls_scale_out_factor`。
4. **SS 参数默认值与 commit 归因**:经 4.4.x 分支 `ob_parameter_seed.ipp` 逐字核对,`_ss_micro_cache_size_max_percentage`(磁盘 / size 维度)默认 **20**、range [1, 99],`_ss_micro_cache_memory_percentage`(memory 维度)默认 **20**、range [1, 50]——两者默认值相同但维度 / range 不同,不可混用;此前“size_max 默认为 5”的写法与源码不符,本轮已据源码更正(故 micro cache 占租户磁盘默认上限约 20% 可作源码确认事实)。SS 实现仍不绑定单一 commit 哈希,按分支 + 文件路径定位。
5. **对象存储延迟来源**：对象存储 cold miss 延迟量级以 AWS S3 官方性能文档(S3 Standard 首字节 ~100–200ms)与 Bacchus 论文自述(100ms+)为据；第三方综述仅佐证“网络成为瓶颈”弱论点，不承载任何定量延迟断言。

---


# 第 24 章 迁移传统分库分表系统的方法论

## 24.1 本章核心问题

传统互联网系统在单机 MySQL 容量见顶后，普遍走上「分库分表」道路：用 ShardingSphere、MyCat 或自研中间件，按分片键(sharding key)把一张逻辑表水平拆成成百上千张物理子表，分布在多个 MySQL 实例上。这条路线把分片逻辑放在应用层与中间件层，数据库内核对此一无所知，中间件同时承担路由、分片键管理、全局 ID 分配、跨库查询改写、历史库拆分与冷热数据治理。它的代价已是业界共识：跨片 JOIN 与跨片事务困难、全局唯一 ID 需外部生成器、分片键一旦选错难以重分片、运维需手工扩容与数据搬迁、二级索引无法跨片(详见第 9 章对分库分表替代的系统性分析)。

本章核心问题不是「把 N 个物理库合成 1 个逻辑库」，而是：当一个在分库分表中间件上运行多年的系统要迁移到 TiDB 或 OceanBase 这类把分片下沉到内核的分布式数据库时，应当遵循什么方法论?关键在于分清哪些复杂度可以由内核接管，哪些复杂度只是从应用与中间件转移到了 Region、Partition、Table Group、事务参与者、全局索引与资源隔离模型中。这不是一次「数据导出导入」，而是一次涉及主键重设计、索引重设计、SQL 兼容性回归、双写灰度、数据校验、可回滚切流的系统工程。

它之所以是分布式数据库底层的关键议题，有三个原因。其一，迁移决策直接受内核分片模型支配：TiDB 用逻辑表、编码 KV keyspace、Region/Raft Group、PD 调度与分布式事务替代应用层分片路由；OceanBase 用 Partition→Tablet→Log Stream-LS、Table Group 与 Tenant/Resource Pool 把业务分区显式纳入内核(详见第 1 章)。两种内核模型决定了「主键/分区键怎么设计才不制造热点」。其二，迁移后一部分复杂度真正消失了(应用不再感知分片数量、不再手工扩容)，但另一部分复杂度只是改变了暴露位置：跨片 2PC 延迟、全局索引写放大、热点 key 排队、范围查询与历史数据生命周期问题依然存在——据此推测，分清这两类才是评估迁移收益的核心。其三，迁移的可回滚性依赖底层的双向同步与一致性校验能力，这又落回到内核的 binlog/CDC 与事务一致性保证上。

据此，本章给出一条方法论主线：先做分片语义盘点，再做目标模型设计，然后执行全量迁移、增量同步或双写、数据校验、灰度切流与回滚演练。这是一章工程/运维内容，源码层只在主键防热点(TiDB)与分区/Table Group/全局索引/兼容模式(OceanBase)这些已 commit-pinned 确认的内核特性上做实现层断言；迁移工具 TiDB DM 与 OceanBase OMS 位于独立仓库、不在本批 checkout 内，相关内容一律据官方文档撰写。凡涉及监控 metric、内部视图名、参数名或函数名而未经官方资料与源码共同确认者，统一标「需进一步查证」。

---

## 24.2 TiDB 的实现：迁移路径与主键防热点

与 OceanBase 把共置与分区设计交给用户显式声明不同，TiDB 把分片打散交给 Region 自动分裂与 PD 调度，迁移时的核心动作集中在「合并到逻辑表」与「重设计主键以防热点」。

### 24.2.1 迁移工具链与数据路径

![[f24_1.svg]]

**图 24-1　DM 组件拓扑**

TiDB 官方迁移工具链按「全量 + 增量」两段式组织:

- **Dumpling**：从上游 MySQL/MariaDB/Aurora 导出全量逻辑数据(SQL/CSV)。
- **TiDB Lightning**：把全量数据快速物理导入 TiDB，适合大数据集(约 ≥1 TiB)，但导入期对集群可用性有显著影响。
- **TiDB DM(Data Migration)**：一体化迁移平台，既能做全量也能做基于 binlog 的增量复制，且支持分库分表合并——把上游多个 MySQL 实例的分片表 DML 与 DDL 合并迁移到下游 TiDB 的同一张逻辑表。
- **sync-diff-inspector**：全量数据一致性校验工具，可比对 MySQL↔TiDB，在分库分表映射场景下支持按 route rules 做映射校验。

官方推荐的分场景策略是：小数据集(约 ≤1 TiB)直接用 DM 完成全量 + 增量；数据量约 1 TiB 以上、可暂停目标 TiDB 写入的分库合并，用 Dumpling 导出、Lightning 快速导入合并后的目标表，再用 DM 接增量 binlog 复制。

DM 内部数据路径分三阶段：**dump phase**(全量导出)→ **load phase**(导入下游)→ **sync phase**(解析上游 binlog 做增量复制)。DM-worker 可把上游 binlog 拉到本地形成 relay log，多个同上游任务复用本地 relay log。在此之上，safe mode 在断点重入时把 INSERT 改写为 REPLACE、把 UPDATE 拆为 DELETE+REPLACE，从而保证同一 binlog event 可重复下发而幂等(代价是性能开销)。控制路径上，DM-master 负责任务调度与 DDL 协调，dmctl 是运维入口。

DM 的分库分表合并需要配置从上游 schema/table 到下游 schema/table 的 route rules，并处理 sharding DDL lock、主键/唯一键冲突与 auto-increment 主键重叠风险。它有悲观与乐观两种模式:

- **悲观模式(默认)**：要求各分片表结构相同；上游某分片发生 DDL 时，阻塞该表的 DML 直到所有分片的同一 DDL 对齐，保证下游不出错。
- **乐观模式**：允许分片表结构不同，DDL 不阻塞继续迁移，但可能漏掉不兼容变更，需人工兜底。

best practices 还给出一条对故障路径至关重要的原则：把不同 sharding group 拆成独立迁移任务，以免一个分片组的 DDL 锁或异常拖住所有迁移(见 §24.6)。

DM 的已知限制必须在评估期就纳入：仅支持 MySQL 5.6–8.0 与 MariaDB 10.1.2+;TiDB 并不兼容 MySQL 全部 DDL，不兼容语法需人工干预；不支持迁移 MySQL 向量数据类型;v5.4.0 以前不支持 GBK 字符集表。这些限制大多在迁移开始后才会暴露代价，因此应前置到选型阶段核对。

### 24.2.2 原分片键的去留

迁到 TiDB 后，原来的 `db_idx`/`table_idx` 路由维度通常不应继续作为应用侧物理路由条件，但分片键本身仍有价值：它往往是高频查询谓词、幂等键、审计维度或冷热归档维度。保守做法是把分片键保留为普通业务列或索引列，先不删除依赖它的查询路径；推测较稳妥的做法是等目标库上的慢查询、Top SQL、执行计划与校验结果稳定后，再决定是否合并索引或移除冗余列。**分片键不会消失，它的语义从「路由规则」变成「执行计划、索引与对账维度」。**

### 24.2.3 主键防热点三件套(源码级确认)

![[f24_2.svg]]

**图 24-2　AUTO_RANDOM 64 位行键位域布局**

分库分表系统普遍用 `AUTO_INCREMENT` 自增主键。迁移到 TiDB 后，自增主键是首要的热点来源：TiDB 用连续递增的行 ID 做 row key，连续写会集中落到同一个 TiKV 节点(详见第 9 章)。官方给出三件防热点工具，均在 release-8.5 源码中确认为编译期常量。

**(1) `AUTO_RANDOM`**——替换自增主键，把高位打散。其 64 位布局为「符号位 + 保留位 (64−R) + 分片位 S + 自增位」。源码 `pkg/meta/autoid/autoid.go` 中常量为(`github.com/pingcap/tidb` `release-8.5`@`67b4876bd57b`):

**表 24-1　主键防热点三件套(源码级确认)**

| 常量 | 值 | 含义 |
|---|---|---|
| `AutoRandomShardBitsDefault` | 5 | 默认分片位，即 2⁵=32 路打散 |
| `AutoRandomShardBitsMax` | 15 | 最大分片位 |
| `AutoRandomRangeBitsDefault` / `Max` | 64 | 默认/最大 range 位 |
| `AutoRandomRangeBitsMin` | 32 | 最小 range 位 |
| `AutoRandomIncBitsMin` | 27 | 自增部分最小位数 |

校验逻辑在同文件 `AutoRandomShardBitsNormalize()`(shard 超 Max 报 `ErrInvalidAutoRandom`)与 `AutoRandomRangeBitsNormalize()`(range 须 ∈[32,64])。官方文档默认配置下 `SHOW WARNINGS` 显示可用隐式分配次数约 288230376151711743 次。

但 `AUTO_RANDOM` 有硬约束，迁移时不能简单地把旧自增列「改个关键字」就完成：必须是聚簇主键(CLUSTERED)的第一列(主键非聚簇直接报错)、仅支持 `bigint` 列、与 `AUTO_INCREMENT`/`DEFAULT` 互斥、不支持减少 shard bits、显式插入需 `@@allow_auto_random_explicit_insert = true`。从自增迁移的标准动作是 `ALTER TABLE t MODIFY COLUMN id BIGINT AUTO_RANDOM(5);`。

**(2) `SHARD_ROW_ID_BITS`**——用于没有显式整型主键的表。这类表 TiDB 用隐式 `_tidb_rowid` 做行 ID，默认单调递增同样造热点。`SHARD_ROW_ID_BITS=4` 把 `_tidb_rowid` 打散到 2⁴=16 个范围。源码 `pkg/meta/model/table.go` 中 `TableInfo` 含 `ShardRowIDBits`、`MaxShardRowIDBits`、`PreSplitRegions` 字段；`SHARD_ROW_ID_BITS` 对 clustered index 有适用限制(`pkg/ddl/executor.go`)。

**(3) `PRE_SPLIT_REGIONS`**——建表时预切 Region，解决「迁移初期一张新表只有 1 个 Region、写全压到一处」的冷启动热点。官方 Split Region 文档逐字写明：一张表的预切 Region 数 = **2^(PRE_SPLIT_REGIONS)**(不是 2^(n−1))；决定性示例 `SHARD_ROW_ID_BITS=4 PRE_SPLIT_REGIONS=2` 会切出「4 + 1」个 Region，其中 4(=2²)个用于行数据、1 个用于索引数据。约束为 `PRE_SPLIT_REGIONS` 须 ≤ `SHARD_ROW_ID_BITS` 或 `AUTO_RANDOM`。锁定源码 `pkg/ddl/split_region.go` 印证：当表设置 sharding bits 且 `PreSplitRegions` 大于 0 时，DDL 路径会按 shard row id 预切 Region 并可等待 scatter 完成；`tidb_scatter_region` 控制是否等 Region 切分散布完成再返回建表结果。

> **反查证(预切公式)**：本章早期核对中曾据源码旧注释写「预切数 = 2^(PreSplitRegions−1)」，经对抗反查证伪——该 off-by-one 公式来自 2019 年最初引入该选项的 PR #10138(「split 2^(pre_split_regions-1) for table row data」)，但随后 PR #11794/#11797(「ddl: refine pre-split region logic」)已显式移除减 1(位移上限由 `1<<(ShardRowIDBits−1)` 改为 `1<<ShardRowIDBits`)。锁定基准 TiDB 8.5.x/7.5.x 的官方文档与源码现行实现均为 **2^(PRE_SPLIT_REGIONS)**，本章以官方文档为准采用 2^N；该公式的引用从源码旧注释改挂到 Split Region 文档。
>
> **反查证(默认分片位)**:`AUTO_RANDOM` 分片位默认值「5」在官方文档(auto-random，标注 v3.1.0 引入)与 release-8.5 源码常量 `AutoRandomShardBitsDefault = 5` 两个独立来源一致，无版本冲突。

因此迁移设计不能只问「要不要保留自增 ID」，而要逐表判断：历史 ID 是否需要无损保留、下游主键是否可能冲突、写入是否集中在最新 ID 区间、目标表是否需要预切 Region。

全局 ID 方面：迁移后若业务对自增连续性有强依赖，TiDB 的 `AUTO_INCREMENT` 在分布式下因各 TiDB 节点缓存批量 ID(默认 `AUTO_ID_CACHE` 30000)，会出现不连续(唯一且递增但有空洞)。官方建议：能接受不连续就用 `AUTO_RANDOM`；需要近似连续可用 `AUTO_ID_CACHE=1`(MySQL 兼容模式)或 `SEQUENCE`，或继续用应用侧 Snowflake/Leaf 分布式 ID 生成器。

### 24.2.4 跨 Region 事务与大表索引

除主键外，迁移还有两处配套验证。

**跨 Region 事务验证。** 分库分表时代「跨片事务」要么做不到、要么靠应用层补偿，迁移到 TiDB 后这类事务变成内核的 Percolator-like 2PC：事务先写 primary lock，再写 secondary lock,commit point 由 primary lock 的提交决定，其他 key 后续推进(详见第 10 章)。验证点是确认原来被应用拆开的事务能否合并为单条 ACID 事务，以及合并后是否引入跨 Region 的多数派往返延迟。测试时应故意构造「同一订单写主表、明细表、库存表、账户流水表」的跨 key-range 事务，观察提交延迟、冲突回滚、锁清理与重试行为。这里的关键(推测)在于不要把「能提交」误读为「所有事务都和单分片一样快」。

**大表索引与冷热数据。** 分库分表系统常把大表索引风险拆散到多个小表中；合并到 TiDB 后，新增或重建索引会集中为目标表的 DDL/backfill 工作。TiDB 用 Online DDL 五状态机加索引(详见第 14 章)，大表加索引必须限速，且全局索引在 v8.4.0 起 GA(v8.3.0 为实验态，详见第 15 章)。应在正式切流前完成主索引、唯一约束与高频二级索引建设，把低频历史查询迁到分区、归档或单独分析链路；若要使用 partitioned table 或 global index，应按第 15 章的索引边界单独验证，不在迁移窗口中临时引入未经压测的新索引形态。冷热分层用 Placement Rules in SQL，粒度可到分区，把热分区钉到 NVMe 节点、冷分区放低成本节点。

--- 〔文献[1,3-4,10,12]〕

## 24.3 OceanBase 的实现：分区表、Table Group 与全局索引

与 TiDB 把分片打散交给自动调度不同，OceanBase 把共置与分区设计交给用户显式声明，迁移时的核心动作也随之从「改主键定义」转向「重设计分区键」。

### 24.3.1 迁移工具链与数据路径

OceanBase 官方迁移工具是 **OMS(OceanBase Migration Service)**，官方文档把它定位为面向 OceanBase 的一站式数据迁移、实时同步与增量订阅工具，可覆盖添加数据源、创建迁移项目、全量迁移与增量同步等流程。社区版 OMS 支持 MySQL 5.5/5.6/5.7/8.0，覆盖 InnoDB/TokuDB/X-Engine 引擎。对 MySQL 分库分表源，实践上应先通过兼容性工具或手工审查完成 schema 转换，再用 OMS 执行全量与增量阶段。

OMS 迁移按五个迁移类型组织，这是本章双写/灰度/回滚方法论的工具基础:

1. **Schema Migration(结构迁移)**
2. **Full Data Migration(全量迁移)**
3. **Incremental Synchronization(增量同步)**
4. **Full Verification(全量校验)**
5. **Reverse Increment(反向增量)**

其中反向增量是回滚能力的关键：切流到 OceanBase 后，OMS 反向把 OceanBase 的增量变更同步回源 MySQL，使源库保持可回退状态。OMS 还支持双向同步(forward + reverse)，通过反循环复制(anti-circular replication)机制避免已同步数据被反向任务重复同步；为保证一致性，同一主键/唯一键的业务写入只应在双向同步的一端执行，否则需配置冲突处理策略(覆盖或忽略)。选择「全量 + 增量」时，OMS 要求源库归档日志至少保留 7 天。

### 24.3.2 分区表与分区键(源码定位)

迁移到 OceanBase 的核心动作是把分库分表的分片键设计为分区表的分区键(HASH/RANGE/LIST/KEY 分区)。分区 DDL 解析在源码 `src/sql/resolver/ddl/`(含 `ob_partitioned_stmt.h` 等，`v4.2.5_CE`@`e7c676806fda`):分区表达式解析入口为 `ObDDLResolver::resolve_part_func()`(`ob_ddl_resolver.cpp` L5241),「主键/唯一键必须覆盖分区键」这条约束由 `ObDDLResolver::check_key_cover_partition_column()`(L7691)与其调用的 `check_key_cover_partition_keys()`(L7635)实现——分区列若不是索引列子集即报 `OB_EER_UNIQUE_KEY_NEED_ALL_FIELDS_IN_PF`，函数注释写明「本函数用于检查分区列是否满足唯一性要求」。

设计要点：分区表的 PRIMARY KEY 与 UNIQUE 约束必须包含分区键(否则无法保证全局唯一)；分区方法要与数据量、查询谓词、数据分布和 partition pruning 对齐——HASH 适合高基数且低倾斜的 user/order 类字段，RANGE 适合时间或数值范围(大日志类表用 RANGE+datetime),LIST 适合离散状态/地域，KEY 在 MySQL 模式下用于多列或非整数键的均匀分布；Table Group 内各表分区数必须相同。迁移时不要机械复刻原来的 `user_id % 128` 物理分库数，而要重新评估当前数据分布、未来增长、热点账户、范围查询与关联查询。若原系统用历史库/冷库承载时间维度，推测可考虑 HASH 主分区加 RANGE 子分区；但具体分区数与子分区规则需用真实数据压测，落地参数仍需进一步查证。

### 24.3.3 Table Group：声明式共置(源码级确认)

OceanBase 提供了分库分表中间件做不到、TiDB 也没有声明式对应物的能力——**Table Group(分区组)**，把同分片键的多张表的分区共置到同一 Log Stream-LS，从而把跨表事务与 JOIN 本地化(详见第 9 章)；同一组内若 join 条件包含分区键，可执行 partition-wise join。源码 `src/sql/resolver/ddl/ob_create_tablegroup_resolver.cpp`(L114–123)确认：CREATE TABLEGROUP 默认 sharding 属性为 `OB_PARTITION_SHARDING_ADAPTIVE`(字符串 `"ADAPTIVE"`)，仅当用户未指定时设置(`if get_sharding().empty()`)。三档取值定义于 `deps/oblib/src/lib/ob_define.h`(L667–669):

**表 24-2　Table Group：声明式共置(源码级确认)**

| 常量 | 字符串值 | 语义 |
|---|---|---|
| `OB_PARTITION_SHARDING_NONE` | `"NONE"` | 不限制加入表 |
| `OB_PARTITION_SHARDING_PARTITION` | `"PARTITION"` | 加入表分区类型/数量/键值须一致 |
| `OB_PARTITION_SHARDING_ADAPTIVE` | `"ADAPTIVE"` | 分区+子分区方法须一致(默认) |

同一 Table Group 内分区规则一致的表，相同分区数据落在同一 OBServer，从而消除跨机 JOIN、避免分布式事务。迁移时把订单表、订单明细表等同按 `user_id` 分区并放入同一 PARTITION sharding 的 Table Group，即可让按 `user_id` 的事务本地化。因此 Table Group 不是「表名前缀合并」的语法糖，而是共置、调度与执行计划的控制面约束。

### 24.3.4 全局索引：代价并未消失(源码级枚举)

Table Group 把同分片键的相关表共置以本地化跨表事务，但对「非分片键查询」无能为力。分库分表时代这类查询要扫所有分片，迁移到 OceanBase 后用全局索引解决，但代价转移进内核——这正对应旧系统中「按 `order_id` 分片却按 `user_id` 查订单」的老问题。源码 `src/share/schema/ob_schema_struct.h` 的 `enum ObIndexType`(L296–317,v4.2.5_CE)确认枚举:`INDEX_TYPE_NORMAL_LOCAL=1`/`UNIQUE_LOCAL=2`(局部索引)、`NORMAL_GLOBAL=3`/`UNIQUE_GLOBAL=4`(全局索引)、`NORMAL_GLOBAL_LOCAL_STORAGE=7`/`UNIQUE_GLOBAL_LOCAL_STORAGE=8`。源码注释明确：非分区表上声明的全局索引会被当作局部索引存储以提升访问性能(`...we regard i1 as a local index for better access performance. Since it is non-partitioned, it's safe to do so`)。

局部索引与主表分区一一对应、同机共置，写入本地原子、不引入分布式事务；全局索引是独立分区的索引表，跨分区唯一性/查询有额外代价——官方文档明确提示 global index 可能让 DML 变成 distributed transaction 并降低写入性能，且全局索引构建需多副本拷贝 + checksum 校验(详见第 15 章)。读路径变短，可能换来写路径变重、远程回表与分区 DDL 维护成本。

### 24.3.5 MySQL 模式还是 Oracle 模式；租户隔离

源码 `deps/oblib/src/lib/worker.h`(L33)确认 `enum class CompatMode { INVALID = -1, MYSQL, ORACLE };`(即 MYSQL=0、ORACLE=1)，租户级兼容模式由该枚举驱动，配套 `is_mysql_mode()`/`is_oracle_mode()`。迁移路线选择：从 MySQL 分库分表迁来一律走 **MySQL 模式**(支持锁函数、非法日期、XA 事务以平滑迁移);Oracle 模式仅在源库确实依赖 Oracle 语义、且目标产品形态具备相应能力时才考虑。关键产品边界：CE 源码虽定义 ORACLE 枚举，但 Oracle 兼容功能是企业版/Cloud 能力，社区版以 MySQL 模式为主(CE/企业版/Cloud 差异详见第 20 章)；且租户兼容模式建后不可改，必须在建租户前确定。

租户与 Resource Pool 方面，迁移可借 OceanBase 原生多租户把不同业务线/冷热库放进不同租户做硬隔离(详见第 17 章)。关键取舍是：Tenant/Resource Pool 应作为业务域、环境与资源隔离边界，而不是按原分片数一比一创建，否则只会把旧分库的运维复杂度原样搬进 OceanBase 的租户运维。建租户的 SQL 路径是「unit config → resource pool → tenant」三步:`CREATE RESOURCE UNIT`(`T_CREATE_RESOURCE_UNIT`)定义 CPU/内存/IOPS 规格、`CREATE RESOURCE POOL ... UNIT=... UNIT_NUM=... ZONE_LIST=(...)`(`T_CREATE_RESOURCE_POOL`)绑定 unit 与 zone、`CREATE TENANT ... RESOURCE_POOL_LIST=(...)` 建租户(语句类型定义于 `src/sql/resolver/ob_stmt_type.h`,`v4.2.5_CE`)。对应可查的内部视图为 `DBA_OB_UNIT_CONFIGS`、`DBA_OB_RESOURCE_POOLS`、`DBA_OB_UNITS`/`GV$OB_UNITS`、`DBA_OB_TENANTS`(视图名常量定义于 `src/share/inner_table/ob_inner_table_schema_constants.h`)。

--- 〔文献[6-9,11,13-16]〕

## 24.4 核心差异对比

表 24-3 按迁移关心的维度分别陈述两种内核的取舍，重点不在排名，而在「同一复杂度落到哪里」。

**表 24-3　迁移传统分库分表系统的方法论:TiDB 与 OceanBase 核心差异对比**

| 维度 | TiDB | OceanBase | 对迁移的影响 |
|---|---|---|---|
| 分片下沉模型 | Region 自动分裂，应用零感知，无声明式共置 | Partition→Tablet→LS,Table Group 声明式共置 | OB 可声明式把相关表共置避跨片事务；TiDB 靠自动调度，简单但不可控 |
| 迁移工具主线 | DM(全量+增量+分库分表合并)+ Dumpling/Lightning + sync-diff-inspector | OMS(结构/全量/增量/全量校验/反向增量) | TiDB 偏「合并到逻辑表 + Region 自动调度」；OB 偏「先设计 Partition/Table Group/Tenant 再迁入」 |
| 原分片键 | 多数保留为业务列/索引列，不再由应用按它路由 | 常转化为分区键、子分区键或 Table Group 共置键 | 分片键不消失，语义从路由规则变成执行计划、pruning 与对账维度 |
| 主键防热点 | `AUTO_RANDOM`(源码常量)/`SHARD_ROW_ID_BITS`/`PRE_SPLIT_REGIONS`(预切数=2^N，官方文档) | 分区键打散 + 主键须含分区键 | TiDB 需改主键定义(聚簇/bigint/首列约束);OB 需重设计分区键 |
| 自增 ID | 分布式下不连续；`AUTO_ID_CACHE`/`SEQUENCE` 折中 | 分区表主键须含分区键，序列/外部 ID | 两边都不能假设连续自增；应用强依赖连续 ID 须改造 |
| 非分片键查询 | 全局索引(v8.4.0 GA)或本地索引扇出 | 全局索引(GLOBAL，独立分区索引表) | 两边全局索引写入都升级为分布式事务，代价对称转移进内核 |
| 跨参与者事务 | 跨 Region 由 Percolator-like 2PC 处理 | 跨 LS 走树形 2PC,Table Group 可本地化 | 原来应用规避的跨片事务变成数据库内部延迟与冲突问题 |
| 资源隔离 | Resource Group/RU 是共享集群 workload QoS，不等同租户 | Tenant/Resource Pool 是更深的资源与元数据边界 | 不要把旧分库数量等同于目标资源隔离单元数量 |
| 兼容模式 | 单一 MySQL 方言(窄而深) | MySQL/Oracle 双模式(宽而分层)，建后不可改 | TiDB 无模式选择；OB 须前置决策，Oracle 模式属企业版 |
| 回滚机制 | 反向需自建 TiCDC/双写 | OMS 反向增量 + 双向同步内建 | OB 工具链内建反向通道；TiDB 回滚需更多自建，且都是数据方向/位点/冲突问题，非「改连接串」 |

贯穿全表的判断是：OceanBase 把共置、分区与隔离边界交给用户显式声明，迁移前置成本高但行为可控；TiDB 把分片打散交给自动调度，迁入简单但热点与共置不可直接编排。两条路线没有绝对优劣，差别在于复杂度暴露在内核的哪一层、由谁负责。

---

## 24.5 正常路径图：分阶段迁移与切流

迁移正常路径的第一原则是「先建目标模型，再迁数据」。如果先把所有分片表粗暴 merge 成单表，再补主键、索引、Partition 或 Table Group，迁移窗口会同时承受数据重写、DDL backfill、热点与业务重试，风险叠加。

![[f24_3.svg]]

**图 24-3　迁移传统分库分表系统的方法论正常读写/调度路径**

该路径的关键控制点：增量同步建立后才做校验，校验一致才进双写；切读先于切写(读切错可秒级回退、写切错代价大)；切写后立即建立反向增量，使源库在观察期内始终是可回退的「热备」。

---

## 24.6 故障/异常路径图

故障路径的总原则是缩小爆炸半径：把不同分片组/租户/业务域拆成独立迁移任务与独立切流批次，使一处异常不致拖垮全局。

![[f24_4.svg]]

**图 24-4　迁移传统分库分表系统的方法论故障/异常路径**

故障路径的核心教训：热点、跨片事务延迟、全局索引写放大这三类故障在切流后才暴露，因为它们与真实写负载强相关，在校验阶段(只读比对)看不出来。这正是「灰度切读 → 灰度切写」分两步、且切写后保留反向增量的根本原因——回滚的前提是反向链路已建立且源库未被破坏性改写。

---

## 24.7 性能、可靠性、运维影响

**延迟。** 迁移后单分片内事务延迟通常下降(免去中间件解析与跨实例协调)，但原本被应用层拆开、规避了的跨片事务，合并为内核分布式事务后会暴露多数派往返延迟——TiDB 的 prewrite/commit 经 Region Raft、跨 Region 写需 ≥1 次多数派往返；OceanBase 跨 LS 走树形 2PC(详见第 8、10 章)。写延迟还受跨 AZ 网络约束：Raft 多数派要求数据复制到至少两个 AZ。

**吞吐与扩展性。** 两者都获得在线水平扩展能力，扩缩容不再需要应用层手工重分片。这是迁移最确定的收益。

**可用性与故障恢复。** 从「中间件 + 多 MySQL 主从」变为内核 Raft/Paxos 多数派(RPO=0)，节点故障自动切主(详见第 3、18 章)，不再依赖人工 failover 脚本。

**运维复杂度。** 应用侧通常能删除大量库表路由、跨库聚合、分片扩容脚本与手工 DDL 分发逻辑，但数据库侧随之出现新的观测对象。消失的运维项——手工分片扩容、跨实例 DDL 协调、分片元数据维护；新增的运维项——TiDB 侧的 Region 分布与热点写入、PD 调度、TiKV 写入/compaction、DDL backfill,OceanBase 侧的 Partition/Tablet/LS 分布、Table Group 共置是否生效、global index 写入代价、major/minor merge 与 Tenant 资源竞争、TSO/GTS 时序服务监控(详见第 19 章)。**运维复杂度并未降低，而是换了一批对象。**OceanBase 侧这些新观测对象有源码确认的内部视图入口:Partition/Tablet/LS 分布看 `DBA_OB_LS`、`DBA_OB_TABLET_TO_LS`,major/minor merge 进度看 `DBA_OB_MAJOR_COMPACTION`/`CDB_OB_MAJOR_COMPACTION` 与 `GV$OB_COMPACTION_PROGRESS`/`GV$OB_TABLET_COMPACTION_PROGRESS`,租户资源竞争看 `DBA_OB_TENANTS`/`GV$OB_UNITS`(视图名常量见 `src/share/inner_table/ob_inner_table_schema_constants.h`,`v4.2.5_CE`)。TiDB 侧具体 Prometheus metric 名以官方监控文档为准，本章不逐一断言。

### 24.7.1 性能瓶颈来源对比

表 24-4 按瓶颈来源逐项对照迁移前后的暴露位置，可以看出大多数瓶颈并未消失，只是从应用/中间件层移进了内核。

**表 24-4　性能瓶颈来源对比**

| 瓶颈来源 | 分库分表(迁移前) | TiDB(迁移后) | OceanBase(迁移后) |
|---|---|---|---|
| 单点写热点 | 自增主键集中在某分片 | 自增 row key 集中单 Region；靠 `AUTO_RANDOM`/预切散开 | 分区键选错致单 Tablet/LS 热点 |
| 跨片/跨分区事务 | 应用层规避或补偿 | 跨 Region 2PC，多数派往返 | 跨 LS 树形 2PC;Table Group 可本地化 |
| 非分片键查询 | 扫全部分片 | 全局索引(写升级 2PC)或扇出 | 全局索引(写升级分布式事务) |
| 全局唯一 ID | 外部生成器(Snowflake/Leaf) | 自增不连续/`SEQUENCE`/`AUTO_RANDOM` | 序列/外部 ID，主键须含分区键 |
| 时序服务 | 无(应用生成) | PD 集群级 TSO，高并发吞吐压力 | 租户级 GTS，单机事务免 RPC |
| 后台抖动 | 各 MySQL 独立 compaction | TiKV RocksDB compaction 本地自治 | major/minor merge 全局协调抖动 |

### 24.7.2 分阶段迁移 checklist

**表 24-5　分阶段迁移 checklist**

| 阶段 | TiDB 要点 | OceanBase 要点 | 退出条件 |
|---|---|---|---|
| 评估 | 盘点 schema、分片规则、route rules、auto-increment 冲突、SQL 兼容性、目标主键与索引 | 盘点分区键、Table Group、global/local index、Tenant/Resource Pool、MySQL/Oracle 模式 | 所有表有目标模型、回滚策略与校验规则 |
| 迁移 | 小数据量用 DM；大数据量用 Dumpling + Lightning 后接 DM 增量 | 用 OMS 做全量迁移，记录源位点与目标 schema 版本 | 全量完成，失败表可重跑，源库仍为权威 |
| 双写/同步 | DM 增量或应用层双写；双写必须幂等并记录冲突 | OMS 增量或应用层双写；按 Tenant/表组拆分项目 | 增量延迟稳定，冲突清单为空或可解释 |
| 校验 | sync-diff-inspector 做分库分表映射校验，叠加业务对账 | OMS 全量校验、SQL checksum、业务对账；具体参数需进一步查证 | 行数、checksum、抽样、金额/库存等业务账一致 |
| 切流 | 只读窗口、灰度写入、逐业务域切换连接串 | 按租户/业务域灰度，观察 Partition/LS 热点 | 错误率、延迟、数据校验在阈值内 |
| 回滚 | 保留源库主写窗口；目标写入可补偿或丢弃重放 | 保留 OMS/应用位点；明确回滚后的目标增量处理 | 回滚演练成功且 RPO/RTO 被业务接受 |

--- 〔文献[5]〕

## 24.8 反例与代价

**何时不该迁移。** 若系统的分片键选得极好、跨片事务与跨片查询极少、容量与 QPS 长期稳定，且核心事务天然只访问一个小库、一台主机、一组缓存，那么迁到分布式数据库可能把原来的本地事务变成跨副本日志复制与分布式事务路径，收益主要在扩展性与运维统一，不能预期所有 P99 都下降。若还强依赖某些 MySQL 特性(如向量类型、特定存储过程/触发器，TiDB 不支持向量类型迁移、存储过程能解析不能执行，详见第 20 章)，迁移收益有限而风险与成本高，据此推测继续维护分库分表可能更划算。

**已知踩坑：**

1. **「改个关键字」式迁移主键。** 旧自增列直接改 `AUTO_RANDOM` 会因不满足聚簇/bigint/首列约束而失败；且 `AUTO_RANDOM` 不支持减少 shard bits、显式插入需开会话变量，迁移期老数据带显式 ID 写入会触发限制。它还改变 ID 可读性、局部有序性与历史 ID 导入方式。
2. **分区键选错难回头。** OceanBase 分区表主键必须含分区键；若按 `order_id` 分区却高频按 `user_id` 查，要么扫全分区要么建全局索引(写放大)；改分区键等于重建表。
3. **全局索引滥用。** 把每个非分片键查询都加全局索引，会把大量本地写变成跨分区分布式事务，写吞吐显著下降。旧系统常用「分片键 + 局部唯一」规避全局唯一约束，迁移后若要求手机号、订单号、外部流水号全局唯一，需在全局唯一索引与应用幂等表之间权衡。
4. **应用强依赖分片物理性。** 某些系统把分片号写入订单号、缓存 key、消息 topic、报表表名与人工运维流程。此时迁移不能只替换 JDBC URL，推测必须先把分片号从业务语义中解耦，否则目标库虽能承载逻辑表，外围系统仍会按旧表名与旧分片号运行。
5. **双写竞态与自增冲突。** 双写期源库与新库各自分配 ID，主键空间不隔离会冲突；`AUTO_RANDOM` 多实例显式插入需 `ALTER TABLE t AUTO_RANDOM_BASE=0` 防碰撞。
6. **切流前未压测真实写负载。** 校验阶段只读比对掩盖了热点与 2PC 延迟，切流瞬间集中暴露。
7. **工具链边界误判。** DM/OMS 能处理大量通用迁移工作，但不能替应用判断冲突语义：两个分片都存在 `id=100` 到底是冲突、历史重复，还是分片内局部 ID?DDL 不一致是等所有分片追平、手工改表，还是拆成两张目标表?这些必须在迁移规则里明确，不能等工具报错后临场决策。

**与替代方案取舍。** 相比「继续分库分表 + 扩中间件」，迁移到 TiDB/OceanBase 把分片复杂度从应用换到内核，获得弹性与强一致，代价是接受跨片 2PC 物理延迟下限、内核调度抖动与一套全新运维知识。两条路线无绝对优劣，取决于跨片事务/查询占比与团队运维能力。

---

## 24.9 测试开发视角的验证点

**可测试的功能场景：**

- 分库同名表合并为单表，验证 route rules、目标 schema、默认值、字符集、时区、NULL 语义与唯一键。
- 自增 ID、雪花 ID、业务 ID、组合主键四类表分别导入并重放增量，验证主键保序需求与去重规则、历史 ID 是否需无损保留。
- 典型跨分片事务改写为目标库单事务，覆盖订单、支付、库存、账户流水、优惠券等多表路径，确认能合并为单条 ACID 事务并提交。
- 历史库/冷库合并后，验证按时间、用户、订单、状态的查询计划是否落到预期索引或 partition pruning。
- 非分片键查询路径(全局索引 vs 扇出)正确性；全局唯一约束在迁移后是否仍生效；在线 DDL 变更不丢数据。
- TiDB `AUTO_RANDOM` 表与 OceanBase HASH/RANGE/KEY 分区表分别做热点写入压测。

**可注入的失效模式：**

- 上游某分片执行 DDL、其他分片延迟或不执行，观察 DM sharding DDL lock / OMS schema 转换处理。
- 全量导入过程中制造主键冲突、唯一键冲突、非法日期、超长字符串与字符集不兼容。
- 增量同步阶段暂停一个源库 binlog 拉取，观察延迟、积压与恢复后的数据顺序(模拟全量耗时过长导致上游 binlog 被 purge)。
- 灰度切流阶段让部分应用仍写源库、部分写目标库，验证幂等表、冲突表与回滚补偿。
- 目标库扩容、Leader/副本切换、Table Group 调整或 Tenant 资源收缩期间重放业务流量；反向增量链路中断，验证回滚可用性。

**关键压测指标：** 切流后真实写负载下的吞吐与 P50/P95/P99 写延迟(对比迁移前)、错误率、锁等待、事务重试、单 Region/单 Tablet 热点 QPS 是否排队、跨片事务比例上升后的吞吐拐点、全局索引写入的额外延迟、DDL backfill 对前台流量的影响、迁移延迟、校验耗时与回滚耗时。

**关键观测指标 / 诊断入口(可查证项，不编造):**

- TiDB 侧：用 TiDB Dashboard 的热力图(Key Visualizer)定位写热点 Region;`SHOW WARNINGS` 看 `AUTO_RANDOM` 可用隐式分配次数；DM 的 `query-status` 看同步阶段与延迟；sync-diff-inspector 报告看 diff 行；Latency Breakdown(database-time)分解 P99(详见第 19 章)。具体 Prometheus metric 名以官方 TiDB 监控文档为准，本章不逐一断言。
- OceanBase 侧：用 `GV$OB_SQL_AUDIT`(SQL 执行诊断视图，看慢 SQL 与执行计划类型，注意它不等于合规 Security Audit)看慢 SQL;看 major merge 进度用 `DBA_OB_MAJOR_COMPACTION`/`CDB_OB_MAJOR_COMPACTION`(集群/租户级)与 `GV$OB_COMPACTION_PROGRESS`/`GV$OB_TABLET_COMPACTION_PROGRESS`(节点/tablet 级,后者含 `UNFINISHED_DATA_SIZE`/`ESTIMATED_FINISH_TIME` 字段)、卡住可看 `GV$OB_COMPACTION_DIAGNOSE_INFO`;Partition/Tablet/LS 分布看 `DBA_OB_LS`/`DBA_OB_TABLET_TO_LS`(视图名常量见 `src/share/inner_table/ob_inner_table_schema_constants.h`,`v4.2.5_CE`)(详见第 19 章);OMS 控制台看各迁移类型进度与全量校验结果，其具体校验参数名因 OMS 在独立仓库、未在本批 checkout 内,仍以官方文档为准。

---

## 24.10 容易误解点

**误解一：「迁移到分布式数据库后，分库分表的所有问题都消失了」。** 错。真正消失的是应用层分片管理(感知分片数、手工扩容、跨实例 DDL)。但跨片事务延迟、热点 key 排队、全局唯一约束代价、非分片键查询代价只是改变了暴露位置，以跨 Region/LS 2PC、全局索引写放大、Region/Tablet 调度抖动的形式继续存在。设计错误(如用自增列做主键)照样在新系统里制造热点。

**误解二：「迁到分布式数据库就不需要分片键了」。** 错。分片键不再一定用于应用侧路由，但仍会影响主键、索引、分区键、Table Group、冷热治理与对账维度(见 §24.2.2、§24.3.2)。TiDB 主键防热点要满足聚簇/bigint/首列约束并选 `AUTO_RANDOM` 或 `SHARD_ROW_ID_BITS`+`PRE_SPLIT_REGIONS`;OceanBase 分区表主键必须包含分区键，且要权衡是否需要 Table Group 共置、是否需要全局索引。主键/分区键/索引是一套联动设计，不是单点替换。

**误解三：「`AUTO_RANDOM` 可以无脑替换所有自增主键」。** 错。它缓解单调写热点，但会改变 ID 可读性、局部有序性与历史 ID 导入方式，且受聚簇/bigint/首列/不可减位等硬约束；源码相关 DDL 路径还有 primary key 与溢出检查(见 §24.2.3)。

**误解四：「`global index` 只提升查询性能」。** 错。它确实能改善非分区键检索，但 OceanBase 文档明确提示可能让 DML 变成 distributed transaction;TiDB 的 partitioned table 全局索引也需按版本(v8.4.0 GA)与写入路径单独评估(见 §24.3.4、第 15 章)。读路径变短，常以写路径变重为代价。

**误解五：「数据校验通过就可以放心切流」。** 错。全量/增量校验是只读比对，验证不了真实写负载下的热点与 2PC 延迟。性能回归必须在双写或灰度切流阶段用真实写负载压测，且切写后保留反向增量链路以保证可回滚——校验一致 ≠ 切流安全。

---

## 24.11 本章结论

1. 迁移分库分表系统不是数据搬迁，而是主键/分区键/索引重设计 + 双写灰度 + 一致性校验 + 可回滚切流的系统工程，迁移工具(TiDB DM、OceanBase OMS)只是其中一环；据此推测，其目标不是消灭复杂度，而是把路由、事务、索引与资源隔离从应用层迁入分布式数据库内核与工具链，并重新定义验证边界。

2. TiDB 主键防热点三件套 `AUTO_RANDOM`(默认 shard bits=5、max=15,release-8.5 源码常量)/`SHARD_ROW_ID_BITS`/`PRE_SPLIT_REGIONS`(预切数=2^(PRE_SPLIT_REGIONS)、须 ≤ `SHARD_ROW_ID_BITS` 或 `AUTO_RANDOM`，以官方 Split Region 文档为准)三者协同；但 `AUTO_RANDOM` 受聚簇/bigint/首列/不可减位等硬约束，不能简单替换旧自增列。

3. OceanBase 迁移核心是把分片键设计为分区键、用 Table Group(NONE/PARTITION/ADAPTIVE，默认 ADAPTIVE)声明式共置相关表以本地化跨表事务；全局索引(ObIndexType GLOBAL=3/4)解决非分片键查询但把写入升级为分布式事务，代价转移进内核而非消失。

4. 迁移路线选择上，从 MySQL 分库分表迁来一律走 OceanBase MySQL 模式(CompatMode MYSQL=0),Oracle 模式属企业版/Cloud 且租户建后不可改；Tenant/Resource Pool 应按业务域而非原分片数划分；TiDB 为单一 MySQL 方言无模式选择。

5. 双写灰度应「先切读后切写」，切写后立即建立反向增量(OMS Reverse Increment / TiDB 侧 TiCDC 自建，TiCDC 位于 tiflow 仓库)，使源库在观察期内保持可回退；回滚是数据方向、位点与冲突解决问题，不是「改回连接串」，其前提是反向链路已建立且源库未被破坏性改写。

6. 迁移后消失的复杂度是应用层分片管理与手工扩容，未消失的复杂度是跨片 2PC 延迟、热点排队、全局索引写放大、后台 compaction/merge 抖动——据此推测，它们随真实写负载在切流后才暴露，校验阶段(只读比对)看不出。

7. 性能回归必须用真实写负载压测，且热点/跨片延迟问题在切流瞬间集中出现，故灰度切流与反向回滚链路被推测为方法论的不可省环节(此判断尚不确定)；无法核验的 metric、内部视图、参数与函数名一律保持「需进一步查证」，不写成版本无关事实。

---

## 24.12 参考文献

[1] Migrate and Merge MySQL Shards of Large Datasets to TiDB | TiDB Docs. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/migrate-large-mysql-shards-to-tidb/.
 （支撑:§24.2 中大数据量分库合并采用 Dumpling + Lightning 全量导入、再用 DM 增量复制的流程，及小数据集直接用 DM 的分场景策略。）
[2] TiDB Data Migration 文档族：Overview / Relay Log / Safe Mode / Shard Merge Best Practices / Best Practices. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/dm-overview/(另：relay-log、dm-safe-mode、shard-merge-best-practices、dm-best-practices.
 （支撑:按页归属分清支撑边界：① dm-overview 支撑分库分表合并、支持版本(MySQL 5.6–8.0/MariaDB 10.1.2+)、DDL 兼容限制；② relay-log 支撑 DM-worker 拉 binl）
[3] AUTO_RANDOM | TiDB Docs. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/auto-random/.
 （支撑:§24.2 中 AUTO_RANDOM 64 位布局、shard bits 默认 5/范围 1–15、range bits 32–64、聚簇/bigint/首列约束、ALTER TABLE ... AUTO_RAN）
[4] SHARD_ROW_ID_BITS 与 SPLIT REGION / Pre-split Regions | TiDB Docs. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/shard-row-id-bits/(另：sql-statement-split-region.
 （支撑:shard-row-id-bits 页支撑 SHARD_ROW_ID_BITS=4 ⇒ 2⁴=16 范围;Split Region 页支撑预切 Region 数 = 2^(PRE_SPLIT_REGIONS)）
[5] Data Check in the Sharding Scenario(sync-diff-inspector)| TiDB Docs. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/shard-diff/.
 （支撑:§24.7、§24.9 中用 sync-diff-inspector 对分库分表映射做数据校验。）
[6] OceanBase Migration Service(OMS / What is OMS). 官方文档[EB/OL]. https://en.oceanbase.com/docs/oms-en.
 （支撑:§24.3 中 OMS 五迁移类型(结构/全量/增量/全量校验/反向增量)、双向同步与反循环复制、归档日志 7 天要求、回滚通道。）
[7] Best practices for table design and index optimization | OceanBase Docs. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-best-practices-10000000002431625.
 （支撑:§24.3 中分区键(HASH/RANGE/LIST/KEY)与数据分布/pruning 对齐、分区表主键须含分区键、子分区设计、Table Group 分区数一致与 partition-wise join。）
[8] Local and Global Indexes / Indexes on partitioned tables | OceanBase Docs. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000000829644.
 （支撑:§24.3、§24.8、§24.10 中 local/global index 取舍，及 global index 可能让 DML 变成 distributed transaction 的写入代价。）
[9] Create a table group | OceanBase Docs. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-cloud-10000000001733899.
 （支撑:§24.3 中 Table Group 三档 sharding(NONE/PARTITION/ADAPTIVE)、默认 ADAPTIVE、共置消除跨机 JOIN/分布式事务。）
[10] Large-scale Incremental Processing Using Distributed Transactions and Notifications(Percolator,OSDI 2010). 论文[EB/OL]. https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Peng.pdf.
 （支撑:§24.2 中 TiDB 跨 Region 事务的 primary lock / secondary lock 与 commit point 协议理论背景(为何验证须覆盖锁清理与提交决议)。）
[11] OceanBase: A 707 Million tpmC Distributed Relational Database System(VLDB 2022). 论文[EB/OL]. https://vldb.org/pvldb/vol15/p3385-xu.pdf.
 （支撑:§24.3/§24.7 中 OceanBase 分区、Log Stream 与共置以降低分布式事务的架构背景。该 707M tpmC(TPC-C Result ID 120051701、v2.2 企业版、Alibaba）
[12] pingcap/tidb(release-8.5 @ commit 67b4876bd57b)— pkg/meta/autoid/autoid.go、pkg/meta/autoid/errors.go、pkg/meta/model/table.go、pkg/ddl/split_region.go、pkg/ddl/executor.go. 源码[EB/OL]. https://github.com/pingcap/tidb/blob/release-8.5/pkg/meta/autoid/autoid.go.
 （支撑:§24.2 中 AutoRandomShardBitsDefault=5/Max=15/RangeBits 32–64/IncBitsMin=27 常量、AUTO_RANDOM 列约束与 rebase）
[13] oceanbase/oceanbase(v4.2.5_CE @ commit e7c676806fda)— ob_create_tablegroup_resolver.cpp、deps/oblib/src/lib/ob_define.h、src/share/schema/ob_schema_struct.h、deps/oblib/src/lib/worker.h、src/sql/resolver/ddl/. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/blob/v4.2.5_CE/src/share/schema/ob_schema_struct.h.
 （支撑:§24.3 中 Table Group 默认 ADAPTIVE、sharding 三常量字符串、ObIndexType 全局/局部索引枚举(GLOBAL=3/4、LOCAL_STORAGE=7/8)、CompatM）
[14] oceanbase/oceanbase(v4.2.5_CE @ commit e7c676806fda)— src/sql/resolver/ddl/ob_ddl_resolver.cpp/.h、src/sql/resolver/ob_stmt_type.h、src/share/inner_table/ob_inner_table_schema_constants.h. 源码[EB/OL]. https://github.com/oceanbase/oceanbase/blob/v4.2.5_CE/src/sql/resolver/ddl/ob_ddl_resolver.cpp.
 （支撑:§24.3.2 中分区表达式解析入口 ObDDLResolver::resolve_part_func()(L5241)、「主键/唯一键须覆盖分区键」约束由 check_key_cover_partition_co）
[15] Create a tenant / Compaction monitoring views | OceanBase Docs. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001228779.
 （支撑:交叉印证 §24.3.5/§24.7/§24.9 中建租户三步流程(unit config → resource pool → tenant)与 DBA_OB_UNIT_CONFIGS/DBA_OB_RESOURC）
[16] An Interpretation of the Source Code of OceanBase(7): Database Index. 第三方解读，阿里云[EB/OL]. https://www.alibabacloud.com/blog/599325.
 （支撑:交叉印证 §24.3 中局部索引同机共置避分布式事务、全局索引独立分区且构建需多副本拷贝 + checksum 的代价。第三方来源，作可信度有限的源码级佐证，核心断言以 OB 官方源码为准。 ---）
## 24.13 信息可信度自评

本章可信度分布如下。**源码级信息(最高可信)**——TiDB 主键防热点常量(`AUTO_RANDOM` shard bits 5/15、range 32–64、约束错误文案、`ShardRowIDBits`/`PreSplitRegions` 字段、sharding bits 驱动预切+scatter)、OceanBase Table Group 三档 sharding 字符串、`ObIndexType` 全局/局部索引枚举、`CompatMode` 枚举，均在 commit-pinned 的统一基线 checkout(TiDB `release-8.5`@`67b4876bd57b`、OceanBase `v4.2.5_CE`@`e7c676806fda`)与 GitHub 在线源码双向确认；`PRE_SPLIT_REGIONS` 预切数值公式不据源码旧注释，改以官方 Split Region 文档为准(2^(PRE_SPLIT_REGIONS))，因仓库旧注释含已废弃的 2^(n−1) off-by-one。**官方文档明确**——分库分表合并/版本/DDL 限制、relay log、safe mode、shard merge best practices、Pre-split 公式与约束、OMS 五迁移类型/反向增量/双向同步、`SHARD_ROW_ID_BITS` 行为、分区表主键须含分区键、分区方法与数据分布对齐、local/global index 代价、兼容模式选择。**协议理论**——Percolator(OSDI 2010)用于解释 TiDB 跨 Region 事务验证为何需覆盖锁清理与提交决议，不直接等同 TiDB 8.5 全部工程细节。**论文背景**——OceanBase VLDB 2022 用于分区共置架构背景，707M tpmC 仅作 benchmark 条件提示，不外推。**第三方佐证**——阿里云 OceanBase 源码解读博客仅作局部/全局索引代价旁证，核心以官方源码为准(占比低，远低于 30% 上限)。**源码补强(本轮升级)**——OceanBase 分区键校验函数名(`resolve_part_func`/`check_key_cover_partition_column`/`check_key_cover_partition_keys`)、建租户 DDL 语句类型与 Tenant/Resource Pool 内部视图名(`DBA_OB_UNIT_CONFIGS`/`DBA_OB_RESOURCE_POOLS`/`DBA_OB_UNITS`/`DBA_OB_TENANTS`)、Partition/Tablet/LS 与 major merge 监控视图名(`DBA_OB_LS`/`DBA_OB_TABLET_TO_LS`/`DBA_OB_MAJOR_COMPACTION`/`GV$OB_COMPACTION_PROGRESS` 等)已在 `v4.2.5_CE`@`e7c676806fda` 源码中确认,并与 OceanBase 官方文档交叉印证,由「需进一步查证」升级为源码级事实。**工程推测/不确定**——「未消失的复杂度在切流后才暴露」「校验一致≠切流安全」「分片键保留为业务列」「资源隔离单元不等于原分片数」是基于公开资料与系统原理的合理推断，已显式标注；仍无法核验的 TiDB Prometheus metric 名、OMS 全量校验的具体参数名(OMS 在独立仓库、不在本批 checkout 内)、以及随真实数据决定的分区数/子分区规则一律保持「需进一步查证」，不写成版本无关事实。

**版本与边界核查。** 本章源码以统一基线 TiDB 8.5.x、OceanBase TP 4.2.5_CE 书写；全局索引 GA 锁定 v8.4.0(v8.3.0 实验态)，与第 15 章一致。OceanBase 官方文档站点核验中多次返回 HTTP 429,OMS 五迁移类型/Table Group 三档 sharding 等断言由 OB 官方源码与阿里云第三方解读交叉确认，核心事实不依赖被限流页面。未在本章写入 TiDB Cloud Starter/Essential 内部隔离、OceanBase Bacchus/shared-storage 或 Cloud-only 延迟数据；TiCDC 归属 tiflow 仓库、br 备份与 DM/OMS 工具链不在本批 checkout 内，相关内容据官方文档撰写，不做源码断言。

---


# 第 25 章 Benchmark 与测试设计

## 25.1 本章核心问题

本章是全书最后一章，任务是把前 24 章的架构结论转译为可执行的测试方法论。读者对象是测试开发工程师：你将拿到一套测试矩阵、可注入的失效模式、可观测的真实指标名，以及"为什么不能只看总 TPS"的工程论证。

分布式数据库的 benchmark，本质上不是"谁更快"的排名问题，而是"在什么 workload、什么硬件、什么版本、什么调优、由谁主导"这五个前提下的可复现观测问题。一份只报告"总 TPS = X"的结论，对理解底层差异的帮助有限：它把数据路径、控制路径、故障路径与后台路径的全部差异，压缩成了一个标量。把这些路径混成一个数字之后，结果既不能解释差异，也不能指导容量规划。

本章站在测试开发视角，回答三件事：

1. **如何设计一份公平、可复现的 TiDB vs OceanBase benchmark**，让两套架构在尽量等价的前提下被观测。PingCAP 官方博客《Why Benchmarking Distributed Databases Is So Hard》明确指出：产品特性与配置不同时极难对比，尤其当测试者是其中一个系统的专家、另一个系统的新手时，容易产生"benchmarketing"（用倾斜的结果做营销）。
2. **为什么不能只看总 TPS**。前 8 章已论证，两套系统把性能优势与代价压在不同路径上（详见第 8 章）：OceanBase 靠单 Log Stream（LS）免 GTS RPC、单机提交短路摊薄金融级短事务延迟，代价是 major merge 抖动；TiDB 靠客户端 TSO 批量化摊薄时序开销，优势在弹性与 HTAP。这些差异只有在 P99/P999 长尾、抖动窗口、跨分片比例、故障注入下才会暴露，总 TPS 会把它们抹平。
3. **如何把"被测系统侧可观测对象"接进测试**。benchmark 工具（go-tpc/obtpcc/sysbench）是独立外部仓库、参数须查证、不能 commit-pin；但被测系统侧的观测对象——GC/safepoint、compaction/merge、Raft/Paxos apply lag、热点分裂、租户资源隔离、故障注入点——在 TiDB/TiKV/PD/OceanBase 源码中均可 commit-pin。这是本章区别于"工具教程"的关键：**测试的预期结果要锚定到被测系统的真实阈值与真实视图**。

与前文的核心约束一致：TiDB 的关键观察边界来自 TiDB Server、PD、TiKV、TiFlash 与 Region / Raft Group；OceanBase 的关键观察边界来自 Tenant、Unit、Partition、Tablet、Log Stream（LS）、PALF 与 ODP。Region / Raft Group 与 Log Stream（LS）是不同的复制边界与事务参与边界，跨参与者比例必须分别统计、不能直接按表数比较。两套系统的性能风险也落在不同路径：TiDB 在热点 Region、Raft apply、RocksDB compaction、跨 Region 事务、PD/TSO；OceanBase 在热点 Partition/LS、PALF 日志延迟、minor/major merge、多租户资源竞争、GTS/RootService/sys tenant 依赖。Benchmark 因此必须覆盖数据路径、控制路径、故障路径与后台路径，而不是只跑一个读写混合脚本。

一句话：benchmark 的价值不在数字本身，而在"数字 + 五前提 + 可观测归因"三元组。本章给出这三元组的可执行版本，凡涉及具体参数、指标名、视图名一律以官方文档或源码事实卡为准，查不到的不编造。 〔文献[1]〕

## 25.2 TiDB 的实现

### 25.2.1 数据路径与压测工具侧（外部仓库，工具参数须查证）

TiDB benchmark 的正常数据路径可拆为四层：SQL 客户端通过无状态的 TiDB Server 完成解析、优化、执行；点查与 OLTP 写入被编码为 KV 请求进入 TiKV；TiKV 按 Region / Raft Group 复制日志并 apply；PD 提供 Region 元数据、调度与 TSO。压测时这三条路径要分别观测——只看 TiDB Server 的 QPS 会漏掉 TiKV 侧 apply lag 与 RocksDB write stall。

TiDB 生态的主力压测工具是 **go-tpc**（`github.com/pingcap/go-tpc`），它是一个 Go 实现的 TPC workload 工具箱，支持 tpcc / tpch / ch（CH-benCHmark）/ rawsql 四类 workload，可对 MySQL、TiDB、PostgreSQL、CockroachDB 等兼容库压测。TiUP 集成了它，`tiup install bench` 后即可用 `tiup bench tpcc ...` 调用。官方 TPC-C 文档给出的标准命令为：

```bash
# 装载（1000 仓，20 并发）
tiup bench tpcc -H <tidb_hosts> -P 4000 -D tpcc --warehouses 1000 --threads 20 prepare
# 压测（100 并发，跑 10 分钟）
tiup bench tpcc -H <tidb_hosts> -P 4000 -D tpcc --warehouses 1000 --threads 100 --time 10m run
# 校验 / 清理
tiup bench tpcc -H <tidb_host> -P 4000 -D tpcc --warehouses 4 check
tiup bench tpcc -H <tidb_host> -P 4000 -D tpcc --warehouses 4 cleanup
```

不同 workload 回答不同问题，不能互相替代：TPC-C 用 NewOrder、Payment、OrderStatus、Delivery、StockLevel 五类事务模拟 OLTP；TPC-H/CH 用 `tiup bench tpch --sf <scale> prepare|run`，`--check=true` 开启结果校验，`--tiflash-replica 1` 可为 TiFlash 建列存副本以测 MPP 路径；CH-benCHmark 把 TPC-C 联机交易与 TPC-H 类查询组合，用于观察 HTAP 干扰，需先部署 TiFlash 并让 OLTP 数据同步过去。官方 Sysbench 文档另说明 Sysbench 1.0+、prepared statement、TiDB/TiKV 日志级别、Default CF/Write CF 比例都会影响结果，须先做单系统最佳实践校准。因此 TiDB 的 benchmark 至少要分出 TiKV OLTP 路径、PD/TSO 路径、TiFlash HTAP 路径与观测路径。**注意工具参数会随版本变化，具体 flag 须以 go-tpc / 官方文档当前版本为准。**

官方 TiDB Cloud v8.5.0 的 TPC-C 报告即用此工具：1000 仓装载，50/100/200 三档并发，每档跑 2 小时，在 2×TiDB（16 vCPU/32 GiB）+ 3×TiKV（16 vCPU/64 GiB）的 TiDB Cloud Dedicated（AWS us-west-2）上，tpmC 从 50 并发的 43,146 到 200 并发的 103,395。这组数字的价值在于它把 cluster type、TiDB version、AWS region、节点规格、benchmark executor、`go-tpc` 命令、warehouse、thread 与 test duration 都列出——这正是公平 benchmark 的可复现前提，而非孤立的"10 万 tpmC"。需要明确边界：该报告是 PingCAP 官方性能报告，公开页面未显示 TPC 官方审计状态，本章只把它作为 v8.5.0 测试披露模板，不把其中 tpmC 写成与其他数据库的 TPC 官方排名。对于 Starter / Essential 形态，公开资料不足以说明内部隔离机制，不得把 Cloud benchmark 外推为多租户共享底座的实现事实。

### 25.2.2 被测系统侧可观测对象（可 commit-pin）

**表 25-1　commit-pin 观测事实卡汇总**

| 观测对象 / 测试项 | 系统 | commit-pin 文件 或 视图名 | 真实常量 / 阈值（逐字） | 对应矩阵测试项 |
|---|---|---|---|---|
| GC / safepoint | TiDB | pkg/store/gcworker/gc_worker.go | gcDefaultLifeTime = 10min、gcMinLifeTime = 10min、gcDefaultRunInterval = 10min、gcDefaultConcurrency = 2、gcModeDefault = gcModeDistributed | 长事务 + GC |
| Raft apply / commit lag | TiDB | components/raftstore/src/store/metrics.rs | tikv_raftstore_apply_log_duration_seconds、tikv_raftstore_apply_wait_time_duration_secs、tikv_raftstore_commit_log_duration_seconds、tikv_raftstore_append_log_duration_seconds；告警 .99 阈值 > 1（1 秒） | Raft/Paxos apply lag、节点故障期间压测 |
| 热点写 / 负载分裂 | TiDB | components/raftstore/src/store/worker/split_config.rs | DEFAULT_QPS_THRESHOLD = 3000、DEFAULT_BIG_REGION_QPS_THRESHOLD = 7000、DEFAULT_BYTE_THRESHOLD = 30 MiB、DEFAULT_DETECT_TIMES = 10 | 热点写 |
| 热点调度（控制面） | TiDB | hot_region_config.go（initHotRegionScheduleConfig()） | MinHotByteRate = 100、MinHotKeyRate = 10、MinHotQueryRate = 10、MaxPeerNum = 1000、SrcToleranceRatio = 1.05、DstToleranceRatio = 1.05、GreatDecRatio = 0.95、MinorDecRatio = 0.99、RankFormulaVersion = "v2" | 扩容期间压测（热点重平衡） |
| TSO 单测 | TiDB | tools/pd-tso-bench/README.md | benchmark GetTS performance（未给数值阈值，写 —） | PD/TSO 路径 |
| 单条 SQL 延迟 / P99 长尾 | OceanBase | GV$OB_SQL_AUDIT（ob_gv_sql_audit.{cpp,h}） | 字段 RETRY_CNT / QUEUE_TIME / GET_PLAN_TIME；审计内存占比由 ob_sql_audit_percentage 控制 | 单分片事务、P50/P95/P99 长尾 |
| compaction / major merge 抖动 | OceanBase | GV$OB_TABLET_COMPACTION_PROGRESS、GV$OB_COMPACTION_DIAGNOSE_INFO、CDB_OB_MAJOR_COMPACTION、__all_virtual_*compaction* | — | compaction/merge、OLTP+OLAP 混跑 |
| 租户资源隔离 | OceanBase | __all_virtual_tenant_resource_limit*（ob_all_virtual_tenant_resource_limit.{cpp,h} 与 _detail.{cpp,h}） | — | 租户资源隔离测试 |
| Paxos（PALF）日志同步 / LS 落后 | OceanBase | __all_virtual_log_stat（ob_all_virtual_log_stat.{cpp,h}） | — | Paxos apply lag、LS 迁移 |

测试开发的关键不是"会跑工具"，而是"会看被测系统在压力下的内部状态"。以下对象均已在源码事实卡 commit-pin（统一基线 TiKV/PD `release-8.5`、pingcap/tidb `release-8.5`），是 TiDB 侧测试的真实观测点：

- **GC / safepoint（对应"长事务 + GC"测试项）**：`pingcap/tidb` @ `release-8.5` commit `67b4876bd57b`，文件 `pkg/store/gcworker/gc_worker.go`。真实常量：`gcDefaultLifeTime = 10min`、`gcMinLifeTime = 10min`（低于此被强制抬到 10 分钟）、`gcDefaultRunInterval = 10min`、`gcDefaultConcurrency = 2`、`gcModeDefault = gcModeDistributed`。系统表 key 常量真实：`tikv_gc_life_time`、`tikv_gc_safe_point`、`tikv_gc_run_interval`。sysvar 名 `tidb_gc_life_time`（`pkg/sessionctx/variable/tidb_vars.go`）可被 `SET GLOBAL` 调长，用于构造"长事务不被 GC 回收"的压测场景。
- **Raft apply / commit lag（对应"Raft/Paxos apply lag""节点故障期间压测"）**：`tikv/tikv` @ `release-8.5` commit `1f8a140b6d46`，文件 `components/raftstore/src/store/metrics.rs`。该文件定义 write/admin command、snapshot、Raft ready、proposal、message drop、compaction guard 等 label 维度，说明 TiKV 源码有面向 Raftstore 内部路径的观测结构。真实 Prometheus 指标名（逐字，注意 apply wait 用 `_secs` 后缀、其余 log duration 系列用 `_seconds`，且全部带 `tikv_raftstore_` 前缀才可直接用于 PromQL）：`tikv_raftstore_apply_log_duration_seconds`、`tikv_raftstore_apply_wait_time_duration_secs`、`tikv_raftstore_commit_log_duration_seconds`、`tikv_raftstore_append_log_duration_seconds`。官方 Grafana 文档对应面板：Apply log duration、Apply wait duration、Commit/Append log duration、Write stall duration（文档明确写"正常应为 0"，这是该页给出的唯一数值期望）。需要纠正一处查无实据的数值：早期资料曾流传"Apply log duration .99 期望 < 100ms"，经对官方 Grafana TiKV 文档与 TiKV 告警规则（`tikv/metrics/alertmanager/tikv.rules.yml`）逐一反查，该 100ms 阈值在任何官方来源中均不存在——Grafana 文档只给面板描述、不给该数值；官方 `.99` 告警阈值实际为 `histogram_quantile(0.99, ...apply_log_duration_seconds_bucket...) > 1`（即 **1 秒**，append_log 同为 > 1s），与 100ms 相差 10 倍。故此处仅保留"apply 期望低延迟"的定性表述，**具体 P99 阈值以官方告警规则的 1s 为准**。
- **TSO 单测（对应"PD/TSO 路径"）**：`tikv/pd` @ `release-8.5`，`tools/pd-tso-bench/README.md` 明确 `pd-tso-bench` 用于 benchmark GetTS performance，可把 TSO 延迟从 SQL TPS 中拆出来单测。这给"控制面延迟须与 SQL TPS 分开报告"提供了现成入口。
- **热点写 / 负载分裂（对应"热点写"）**：`tikv/tikv` @ `release-8.5` commit `1f8a140b6d46`，文件 `components/raftstore/src/store/worker/split_config.rs`。真实常量：`DEFAULT_QPS_THRESHOLD = 3000`、`DEFAULT_BIG_REGION_QPS_THRESHOLD = 7000`、`DEFAULT_BYTE_THRESHOLD = 30 MiB`、`DEFAULT_DETECT_TIMES = 10`。官方文档侧反查证一致：`split.qps-threshold` 默认 3000（region-split-size ≥ 4GB 时为 7000），byte-threshold 默认 30 MiB（100 MiB），且需连续满足 **10 秒** 才触发 load-based split。Region 默认大小为 256MiB（锁 v8.5.0，更早版本起点差异只作版本口径备注）。
- **热点调度（控制面）**：`pingcap/pd` @ `release-8.5` commit `6dce4a68e3e9`，目录 `pkg/schedule/schedulers/`，确认存在 `hot_region.go`、`hot_region_rank_v1.go`、`hot_region_rank_v2.go`、`balance_region.go` 等文件。balance-hot-region-scheduler 的默认阈值常量在 `hot_region_config.go` 的 `initHotRegionScheduleConfig()` 中逐字定义：`MinHotByteRate = 100`、`MinHotKeyRate = 10`、`MinHotQueryRate = 10`、`MaxPeerNum = 1000`、`SrcToleranceRatio = 1.05`、`DstToleranceRatio = 1.05`（容忍 5% 差异）、`GreatDecRatio = 0.95`、`MinorDecRatio = 0.99`、`RankFormulaVersion = "v2"`。官方 PD Control 文档侧 `balance-hot-region-scheduler` 配置项默认值逐一吻合（`min-hot-byte-rate=100`、`min-hot-key-rate=10`、`min-hot-query-rate=10`、`src/dst-tolerance-ratio=1.05`、`max-peer-number=1000`、`rank-formula-version=v2`）。
- **进程内故障注入（对应"节点故障""网络分区"）**：`tikv/tikv` @ `release-8.5` commit `1f8a140b6d46`，`Cargo.toml` 定义 `failpoints` feature（聚合 `fail/failpoints`、`raftstore/failpoints`、`tikv_util/failpoints`、`engine_rocks/failpoints`、`raft_log_engine/failpoints` 等子 feature）；`components/raftstore/src/store/fsm/store.rs` 内有 10 处 `fail_point!(...)` 注入点，逐字确认含 `"send_raft_message_full"`（line 409）、`"memtrace_raft_messages_overflow_check_send"`、`"begin_raft_poller"`、`"on_raft_ready"`、`"after_shutdown_apply"`、`"after_acquire_store_meta_on_maybe_create_peer_internal"`、`"on_mock_store_completed_target_count"`。范围上：整库 `*.rs` 共出现 `fail_point!(` 约 492 处，仅 `components/raftstore/` 下去重后即有约 130 个不同命名注入点。须以 `failpoints` feature 编译才生效，适合确定性故障注入。
- **Grafana 面板版本（可比性）**：pingcap/tidb `release-8.5` 的 `pkg/metrics/grafana/README.md` 说明 Grafana JSON 由 jsonnet 生成，因此压测报告应固定 dashboard 文件版本，避免面板变化导致历史结果不可比。

把上述观测点放回请求链路，才能看清各指标采自哪一段。数据路径上，go-tpc 经 MySQL 协议打到无状态 TiDB Server，TiDB 经 client-go 把请求路由到 TiKV（Region/Leader，详见第 2 章），写经 Percolator 2PC + Region Raft 落盘（详见第 10 章）。控制路径上，PD 负责 TSO 发号、Region 调度与热点重平衡。

故障路径上，TiDB 测试需覆盖 TiDB Server 连接迁移、TiKV leader 切换、Region split/merge、snapshot、PD leader 或 TSO 压力、GC safe point、TiFlash 同步滞后与滚动升级。Jepsen 对 TiDB 2.1.7 到 3.0.0-rc.2 的历史测试不是性能 benchmark，但它提示测试开发应把网络分区、进程 pause、crash、clock skew 与跨 Region 事务历史一起验证；更重要的是，一致性测试只能证明发现的 bug，不能证明"永远正确"（见 §25.10）。这类结果不能直接套用到 TiDB 8.5，但方法仍有价值。 〔文献[2-8,13,16]〕

## 25.3 OceanBase 的实现

与 TiDB 的"外置监控栈 + 工具侧压测"不同，OceanBase 把压测入口与可观测对象都更多地收敛进内核与官方部署工具。其数据路径也与 TiDB 不同：用户请求进入 OBProxy/ODP 或直连 OBServer，在 Tenant 内被调度到 Partition / Tablet 所在的服务路径；事务参与边界通常落在 Log Stream（LS）；日志通过 PALF（Paxos-backed Append-only Log File system）以复制式 WAL 形式 append 并恢复，配合事务状态推进；后台 major/minor merge、Tablet/LS 迁移、Tenant 资源控制会影响延迟尾部。测试设计不能把 OceanBase 简化成"单机 MySQL 加分区"，也不能把 Log Stream（LS）与 TiDB Region 一一等价。

### 25.3.1 压测工具侧（外部仓库，工具参数须查证）

OceanBase 的标准压测路径是 **OBD（OceanBase Deployer）** 封装的 `obd test` 命令，底层调用 obtpcc（TPC-C）、obtpch（TPC-H）与 ob-sysbench。官方 OBD performance test 文档与 `oceanbase/obdeploy` 仓库给出的命令为：

```bash
# TPC-C（先装 OBClient、obtpcc、java）
obd test tpcc <deploy_name> --tenant=tpcc --warehouses 10 --run-mins 1
# TPC-H（先装 obtpch）
obd test tpch <deploy_name> --tenant=tpch ...
# Sysbench（脚本 oltp_read_only / oltp_write_only / oltp_read_write）
obd test sysbench <deploy_name> --tenant=sysbench --script-name=oltp_read_only.lua \
 --table-size=1000000 --threads=32 --rand-type=uniform
```

`obd test tpcc` 关键 flag：`--tenant`（默认 test）、`--warehouses`（默认 10）、`--run-mins`（默认 10）；`obd test sysbench` 关键 flag：`--tenant`、`--script-name`（默认 point_select.lua）、`--table-size`（默认 20000）、`--threads`（默认 16）、`--rand-type`（uniform/gaussian/pareto/special）。该文档还提醒：**不要用 `sys` tenant 跑测试**，并建议磁盘 IOPS 高于 10000。OBD 会根据运行环境自动调优测试参数与生成配置文件，这一点在公平对比时需特别注意：**OBD 的自动调优意味着 OceanBase 侧用了"厂商最佳实践配置"，对比时 TiDB 侧也应使用官方推荐配置，否则不公平**（回到 §25.1 的 benchmarketing 风险）。obtpcc/obtpch/ob-sysbench 的具体参数会随工具版本变化，正式测试前须复核当前工具版本的 flag。

OceanBase 官方 V4.2.1 TPC-H benchmark report 给出了更完整的 AP 披露样例：三节点 Alibaba Cloud ECS `ecs.g7.8xlarge`，每节点 32 CPU cores / 128 GB，系统盘、clog 盘与 data 盘分开，tenant resource unit、resource pool、locality、CE/EE V4.2.1.0、TPC-H V3.0.0、CentOS 7.9、100 GB 数据规模与每个查询的响应时间都列出。需要明确边界：这是 OceanBase 官方文档测试报告，不等同于 TPC 官方 Full Disclosure Report（FDR），本章只把它作为"如何披露 AP workload 条件"的样例，不把结果写成绝对排名。

### 25.3.2 被测系统侧可观测对象（可 commit-pin）

OceanBase 的可观测性是内核内置 gv$/v$ 虚拟表式（详见第 19 章），无需外部监控栈即可 SQL 取数。以下视图均已在源码事实卡 commit-pin（`oceanbase/oceanbase` @ `v4.2.5_CE` commit `e7c676806fda`，目录 `src/observer/virtual_table/`）：

- **单条 SQL 延迟 / P99 长尾归因（通用 P50/P95/P99 项）**：视图名常量 `GV$OB_SQL_AUDIT`（`OB_GV_OB_SQL_AUDIT_TNAME`）、`V$OB_SQL_AUDIT`、底层 `__all_virtual_sql_audit`，实现于 `ob_gv_sql_audit.{cpp,h}`。它是内存环形缓冲，记录每条 SQL 的执行时间、等待事件、重试次数、排队时间、取计划时间。诊断 P99 抖动时查 RETRY_CNT（高→锁冲突或切主）、QUEUE_TIME（高→排队）、GET_PLAN_TIME（高且 IS_HIT_PLAN=0→plan cache miss）。审计内存占比由 `ob_sql_audit_percentage` 控制。需注意它是 SQL 执行诊断视图，不等于合规意义上的 Security Audit。
- **compaction / major merge 抖动（对应"compaction/merge""OLTP+OLAP 混跑"）**：虚拟表 `ob_all_virtual_tablet_compaction_info`、`_history`、`_progress`、`ob_all_virtual_compaction_diagnose_info`、`ob_all_virtual_compaction_suggestion`、`ob_all_virtual_server_compaction_progress`。对应可查询的 GV$ 视图：`GV$OB_TABLET_COMPACTION_PROGRESS`、`GV$OB_COMPACTION_DIAGNOSE_INFO`、`CDB_OB_MAJOR_COMPACTION`（STATUS 长期 COMPACTING 即异常）；`GV$OB_COMPACTION_PROGRESS` 中 STATUS="NODE_RUNNING" 时看 UNFINISHED_TABLET_COUNT 是否长期不更新。这是 OceanBase 版的"compaction 卡住"判定入口，对位 TiKV 的 RocksDB compaction 指标。
- **租户资源隔离（对应"租户资源隔离测试"）**：虚拟表 `ob_all_virtual_tenant_resource_limit.{cpp,h}` 与 `_detail.{cpp,h}`，即 `__all_virtual_tenant_resource_limit*`。压测租户隔离时从此读各租户的资源上限/明细（详见第 17 章 OceanBase 原生多租户的硬隔离）。
- **Paxos（PALF）日志同步 / LS 落后（对应"Paxos apply lag""LS 迁移"）**：虚拟表 `ob_all_virtual_log_stat.{cpp,h}`，即 `__all_virtual_log_stat`。它对位 TiKV 的 `tikv_raftstore_apply_log_duration_seconds`——但二者语义不同：OceanBase 是复制式 WAL / Paxos-backed append-only log + 事务状态推进，TiKV 是基于 Raft 的复制状态机（RSM，详见第 3 章），apply lag 的含义需按各自共识模型解读，不可直接互换比较。

此外，OceanBase 源码树自带面向日志层的内部测试入口：`oceanbase/oceanbase` @ `v4.2.5_CE` 基线下的 `mittest/palf_cluster/README.md` 是一个 PALF cluster test framework，用途是在真实集群中评估 PALF performance，观测维度含 Throughput、Latency、IOPS（另据 `e859d1b9` 亦确认该框架存在）。这说明日志复制路径值得单独压测；但该框架需应用 diff、编译 mittest，不是用户级 benchmark 工具，不能替代 OBD/obtpcc/ob-sysbench。

shared-storage / Bacchus 需要额外谨慎：相关架构在 4.3.5+ 文档与 4.4.x 架构中按版本与产品形态限定描述，社区版默认不编译。Bacchus 论文提出对象存储上的 LSM-tree shared-storage 架构、shared service-oriented PALF logging、Shared Block Cache Service，并报告 SysBench/TPC-H 实验与成本下降；这些结果是论文实验条件下的研究报告，不是 OceanBase CE classic 的生产 OLTP 延迟事实。若写入测试矩阵，只能作为 shared-storage 形态的候选 workload，并标注（详见第 7/23 章）。

把这些视图放回请求链路便于定位指标来源。数据路径上，obtpcc 经 OBProxy/ODP 路由到 OBServer（分区键→Tablet→Log Stream（LS）→Leader，详见第 2 章），写经协调者下沉的树形 2PC + LS Paxos（详见第 10 章）。控制路径上，RootService（随 `__all_core_table` Leader 重建，详见第 13 章）聚合 schema/DDL/balancer，GTS 按租户级隔离发号。OceanBase 侧压测的独特观测点是 **major freeze/merge 的全局协调**——它由 RootService 选全局版本号、跨副本协调出一致基线（详见第 4 章），因此 major merge 期间的吞吐塌陷是 OceanBase 特有的、可预期的抖动窗口。 〔文献[9-10,15]〕

## 25.4 核心差异对比

两套系统的测试设计差异不在工具命令，而在被观测对象落在哪条路径。下表按分布式边界、压测工具、可观测形态、延迟归因、控制面、apply lag、热点、compaction、GC、故障注入十个维度分别陈述，每行末列直接给出对测试设计的影响。

### TiDB vs OceanBase 测试设计主对比

**表 25-2　TiDB vs OceanBase 测试设计主对比**

| 维度 | TiDB | OceanBase | 对测试设计的影响 |
|---|---|---|---|
| 分布式边界 | Region / Raft Group 是 KV range 与复制边界 | Partition、Tablet、Log Stream（LS）分层，LS 承载日志复制与事务参与 | 跨参与者比例须分别统计跨 Region 与跨 LS，不能直接按表数比较 |
| 主力压测工具 | go-tpc / TiUP bench、Sysbench、CH-benCHmark（MySQL 协议） | OBD `obd test`（obtpcc/obtpch/ob-sysbench） | 工具均为外部仓库、参数须查证；OBD 自带自动调优，对比须对齐配置 |
| 可观测形态 | 外置组件拼装（Prometheus/Grafana/Dashboard） | 内核内置 gv$/v$ 虚拟表，SQL 即诊断 | TiDB 须部署监控栈；OceanBase 零外部依赖但内存视图保留窗口短 |
| 延迟归因入口 | TiKV Grafana 面板 + Slow Query | `GV$OB_SQL_AUDIT`（RETRY_CNT/QUEUE_TIME/GET_PLAN_TIME） | 长尾归因路径不同，需各自掌握 |
| 控制面单测 | PD TSO；`pd-tso-bench` 可单测 GetTS | RootService/sys tenant/GTS/ODP 元数据与路由依赖 | 控制面延迟、不可用、恢复时间须与 SQL TPS 分开报告 |
| apply lag 指标 | `tikv_raftstore_apply_log_duration_seconds` 等 4 个 histogram | `__all_virtual_log_stat`（LS/PALF 状态） | RSM vs 复制式 WAL，apply lag 语义不可直接互换 |
| 热点分裂阈值 | load-based split：单 Region QPS>3000（大 Region 7000）连续 10s | LS/Tablet 调度由 RootService balancer 驱动 | 热点写测试的"预期结果"可锚定 TiDB 真实阈值；OB 侧仅到调度器级 |
| compaction 观测 | RocksDB compaction 指标（本地自治） | `__all_virtual_*compaction*`（major freeze 全局协调） | OB 有可预期的 major merge 抖动窗口，须纳入测试矩阵 |
| GC / 历史版本 | GC safepoint（默认 10min）+ write CF compaction filter | row compaction + mini/minor/major merge | "长事务+GC"项：TiDB 锚定 10min 窗口，OB 锚定 merge 节奏 |
| 故障注入（进程内） | TiKV `fail_point!`（failpoints feature 编译） | OceanBase CE 内置 errsim tracepoint（`OB_E`/`ERRSIM_POINT_DEF`，租户参数 `errsim_module_types`+`errsim_module_error_percentage` 控制） | 两侧均有确定性进程内注入；OB 走 errsim 事件点，非外部工具 |
| 故障注入（集群层） | Chaos Mesh（PingCAP 开源，K8s CRD） | Chaos Mesh 同样适用（通用 K8s 工具） | 集群层注入两者通用 |

### 关键指标"为什么不能只看总 TPS"对照

**表 25-3　关键指标"为什么不能只看总 TPS"对照**

| 指标类别 | 总 TPS 会掩盖什么 | TiDB 观测点 | OceanBase 观测点 |
|---|---|---|---|
| P99/P999 长尾 | 平均吞吐高但偶发数十秒卡顿（GC 暂停、锁竞争、快照开销） | Grafana 分位面板 / Slow Query | `GV$OB_SQL_AUDIT` 按 RT 排序取 top-N |
| 事务冲突率 | 高冲突下重试拉高真实延迟 | client-go 重试 / lock 冲突指标 | `GV$OB_SQL_AUDIT.RETRY_CNT` |
| 跨分片事务比例 | 跨 Region/LS 事务触发 2PC 多数派往返 | 跨 Region 2PC 比例 | 跨 LS 2PC 分支数 + GTS/SCN 等待 |
| compaction/merge 抖动 | major merge 期间吞吐塌陷 | RocksDB write stall duration | `GV$OB_COMPACTION_PROGRESS` |
| apply lag | 共识层落后导致读延迟 | `tikv_raftstore_apply_log_duration_seconds` | `__all_virtual_log_stat` |
| 热点分布 | 单 Region/LS 热点排队 | load-based split（QPS>3000） | RootService 热点重平衡 |

## 25.5 正常路径图

图 25-1 展示一次公平 benchmark 的执行与观测：从固定五前提，到两套系统各自的数据/控制路径，再到同步采集被测系统内部指标。

![[f25_1.svg]]

**图 25-1　Benchmark 与测试设计正常读写/调度路径**

这张图的测试含义是：客户端侧 TPS/QPS 只是入口，真正需要关联的是执行路径上的参与者数量、复制日志延迟、后台存储任务与控制面响应。TiDB 的 CH-benCHmark 需要让 TiFlash 副本可用并确认 OLAP 查询确实走 TiFlash/MPP 路径；OceanBase 的 TPC-H/Sysbench/TPC-C 需确认 Tenant 不是 `sys` tenant，资源池、Locality、Primary Zone 与 ODP/直连方式一致。正常路径的关键不在压测本身，而在**预热阶段必须等 compaction/merge 稳定**（否则测的是冷启动而非稳态），以及**采集阶段同步抓被测系统内部指标**（否则只剩一个无法归因的标量）。

## 25.6 故障/异常路径图

故障注入是 benchmark 的"压力测试"维度——稳态 TPS 表现良好，并不代表故障期间仍然可用。图 25-2 展示故障注入下的异常路径与观测点：

![[f25_2.svg]]

**图 25-2　Benchmark 与测试设计故障/异常路径**

这条故障路径揭示了几类必须测的失效模式：**节点故障**（切主期间 TPS 跌零、P999 飙升，观测 RTO/RPO 与 tpmC 恢复曲线）、**网络分区/延迟/丢包**（跨参与者事务等待、只读行为、恢复）、**确定性进程内注入**（TiKV `fail_point!` 触发特定分支，验证错误处理）、**控制面依赖抖动**（PD/TSO 与 RootService/GTS 受阻时新事务能否取号或变更元数据）、**时钟漂移**（TimeChaos 对 PD 注入时序扰动，验证 TSO/GTS 时序服务是否被破坏）。每个故障用例后都要记录注入窗口、影响对象、恢复时间、错误率、P99/P999、事务冲突率与迁移/merge 进度，并同屏对齐。Jepsen 历史上正是用 libfaketime 注入指数分布的时钟偏移（最高 232 秒）与 5x 时钟加速测 TiDB 2.1.7，虽未直接由时钟引发异常，但发现了 auto-retry 机制导致的 read skew 与 lost update。

故障注入建议使用 Chaos Mesh 等工具做 Pod、network、IO 或 workflow 级实验。Chaos Mesh 文档说明其故障注入覆盖基础资源故障、平台故障与应用层故障，并支持串并行 Chaos workflow、YAML 管理与 Dashboard。 〔文献[11]〕

## 25.7 性能、可靠性、运维影响

公平 benchmark 的第一原则是"同等业务问题，而不是同等命令行"。Sysbench 可以快速比较点查、读写混合、update-index、insert 等基本 OLTP 压力，但默认脚本的热点、索引形态与事务比例未必贴近业务；两边都要先做单系统最佳实践校准，再做横向对比。

**延迟**：总 TPS 高不等于延迟好。TiDB 8.5 的工程实践显示，即使吞吐稳定，P999 也可能从"数十秒尾巴"恶化——根因是 GC 暂停、锁竞争、存储快照开销三类罕见但灾难性的停顿；8.5 通过把任务移出关键路径、重排避免停顿，把 P999 压到亚秒级，慢查询爆发减少约 30%–90%、TiKV CPU 平均下移约 10%–25%。这正是"不能只看总 TPS"的实证：同样的 TPS 下，P999 可以差两个数量级。长尾往往来自热点、跨参与者事务、存储后台任务、控制面抖动与故障恢复，据此可以推测，P99/P999 与恢复时间比平均 TPS 更接近工程风险。

**吞吐与 vendor benchmark 的可比性**：引用 vendor TPC-C 成绩必须连同硬件、版本、审计方、利益相关方一起引用。TPC 官方结果页（tpc.org）会披露 tpmC、Price/tpmC、系统可用日期、数据库、OS、TP monitor、提交日期等字段，合格结果必须和系统配置一起读；非审计测试只能写"基于 TPC-C workload 的内部测试"，不能写成"通过 TPC-C 排名"。其中常被误引的 707M tpmC，是 TPC-C Result ID 120051701、OceanBase v2.2 Enterprise Edition、Alibaba Cloud ECS、1554 data nodes、官方 FDR 记录日 2020-05-17 的审计成绩，不代表 4.x/CE/小集群或普通生产环境。TiDB Cloud v8.5.0 的 103,395 tpmC 则是 2+3 节点小集群成绩——两者硬件规模相差三个数量级，直接比较数字并无意义。

**可用性 / 故障恢复**：集群内 RPO=0 是两系统共有的协议自洽强约束。RTO<8s 仅限 OceanBase V4.x LS/Paxos 多副本、少数派故障或多数派完整场景；它不绑定 RootService，也不能外推到单副本 shared-storage 或跨云形态（独立学术来源仅支持更宽松的 RTO 上界，详见第 8/18 章）。测试时必须实测故障窗口的 TPS 跌零时长与恢复曲线，而非引用厂商指标。

**扩展性**：扩容期间压测是必测项——扩容触发 Region/LS 迁移与热点重平衡（TiDB load-based split、PD hot_region scheduler；OceanBase RootService balancer），迁移期间会有抖动，稳态吞吐无法反映这个窗口。

**运维复杂度**：TiDB 观测依赖外部 Prometheus/Grafana 栈，好处是生态成熟、坏处是组件多；OceanBase 内置 gv$ 视图 SQL 即诊断，好处是零外部依赖、坏处是内存视图保留窗口短（详见第 19 章）。测试报告应记录两者的观测成本差异。第二张瓶颈来源对照见下： 〔文献[12,14]〕

### 性能瓶颈来源与测试结论写法

**表 25-4　性能瓶颈来源与测试结论写法**

| 瓶颈来源 | TiDB 观测方式 | OceanBase 观测方式 | 测试结论写法 |
|---|---|---|---|
| 热点写 | 热点 Region、leader CPU、Raft apply、lock conflict | 热点 Partition/Tablet/LS、Tenant CPU、事务等待 | 写"热点 key/参与者导致长尾"，不写"系统整体慢" |
| 跨参与者事务 | 跨 Region key range、2PC lock/commit 阶段 | 跨 LS 事务参与者与 GTS/SCN 等待 | 报告跨参与者比例与 P99/P999 的关系 |
| 存储后台任务 | RocksDB compaction、GC safe point、snapshot、split/merge | minor/major merge、Tablet/LS 迁移 | 压测须包含后台任务开启/关闭或可控窗口 |
| 控制面压力 | PD TSO、Region heartbeat、scheduler、placement rules | RootService、sys tenant、GTS、ODP location cache | 区分逻辑中心化、性能瓶颈、HA 单点、元数据依赖、恢复依赖 |
| HTAP 干扰 | TiFlash learner 同步、MPP 查询、TiKV OLTP 写入 | AP 查询、PX、列存/行存、merge、Tenant 资源 | 同时报 OLTP 延迟与 OLAP 完成时间 |

## 25.8 反例与代价

**benchmark 方法论的反例**：

1. **只跑单连接 / 单参与者点查**：几乎一定会把压测工具自己变成瓶颈，测的是工具不是数据库；若 workload 全部命中单 Region 或单 LS，总 TPS 主要反映 SQL 层、网络、缓存、单 leader 与客户端线程，无法覆盖跨参与者事务、调度迁移、merge、GC 或 TiFlash/列存路径。这样的结果可作 smoke test，不能作架构对比结论。PingCAP 官方明确警告这一点，TPC-C/sysbench 用多连接正是为此。
2. **只看稳态、不测抖动**：major merge / compaction、扩容迁移、切主都会制造抖动窗口，稳态 TPS 把它们抹平。OceanBase major merge 是全局协调的可预期抖动，必须单独测。
3. **只追求公开报告数字 / 配置不对齐**：一侧用厂商自动调优（OBD）、另一侧用默认配置，是典型 benchmarketing。TiDB Cloud v8.5.0 报告与 OceanBase V4.2.1 TPC-H 报告的 workload、硬件、版本、审计状态与商业利益相关方都不同，任何把这些数字拼成 TiDB vs OceanBase 绝对排名的写法都不成立。两侧都应使用各自官方推荐配置。
4. **硬件 / 版本不匹配却直接比数字**：707M tpmC（1554 data nodes、2020、v2.2 企业版，见 §25.7）与小集群 tpmC 不可比。引用任何 vendor benchmark 必须连同硬件/版本/审计方/利益相关方一起引用。
5. **压测与生产隔离模型不一致**：OceanBase Tenant 是内核级资源与元数据边界，TiDB Resource Control 是共享集群内 workload QoS 与软隔离机制；TiDB Cloud Starter / Essential 的内部隔离机制未公开。若测试目标是多租户隔离，必须设计邻居租户、资源上限、后台任务、ODP/连接池、读写混合与突发流量，不能把单租户 Dedicated 或自建集群结果外推到共享云形态。
6. **把论文实验数据当生产事实**：OceanBase Bacchus shared-storage 的延迟数据是论文实验条件下的，不得引为"生产 OLTP 延迟"（见 §25.3.2，详见第 7/23 章）。

**测试设计的代价**：完整测试矩阵（下节 13 项 × 两系统）成本极高，故障注入需要 K8s + Chaos Mesh 环境，Jepsen/Elle 一致性校验是 NP 完全问题（检查器只能证伪不能证全对），时钟漂移测试需要可控时间环境。故障注入还可能触发重调度、数据迁移与后台恢复，使后续实验不再处于同一初始状态——每个故障用例后应验证副本补齐、Region/LS 均衡、GC/merge 完成、错误率恢复与拓扑回归，否则下一轮数据会被上一轮故障污染。这些都意味着**完整 benchmark 是工程项目而非脚本**。取舍上，可以推测一种务实路径：生产前优先覆盖与自身 workload 最接近的 3–5 项（如金融偏短事务加高冲突、互联网偏热点写加大范围扫描），而非盲目跑满矩阵。

## 25.9 测试开发视角的验证点

完整测试矩阵如下。参数中已公开查证的写出来源边界；无法确认的不编造，写"需进一步查证"或"按官方工具文档确认"。

### TiDB vs OceanBase 完整测试矩阵

**表 25-5　TiDB vs OceanBase 完整测试矩阵**

| # | 测试项 | 工具 / 注入方式 | 关键压测指标 | TiDB 观测点 | OceanBase 观测点 | 预期 / 踩坑 |
|---|---|---|---|---|---|---|
| 1 | 单分片事务 | go-tpc / obtpcc / Sysbench | TPS, P50/P95/P99/P999 | TiKV apply lag | `GV$OB_SQL_AUDIT` | 走单 Region/单 LS 短路；prepared statement/日志级别/客户端瓶颈影响结果 |
| 2 | 跨分片事务 | 自定义 workload（控制 key 分布跨 Region/LS） | P99, 2PC 往返 | 跨 Region 2PC 比例 | 跨 LS 分支数 + GTS/SCN 等待 | 跨边界升级为 2PC；只按表分区不一定等于跨参与者 |
| 3 | 热点写 | sysbench update-index / 自增键 | 单 Region/LS QPS | load-based split（QPS>3000 连续 10s） | RootService 热点重平衡 | TiDB 应自动 split；自增键做主键必现热点 |
| 4 | 大范围扫描 | TPC-H / go-tpc tpch / obtpch | QphH, 扫描吞吐， spill | Coprocessor / TiFlash MPP | PX 并行 / 列存 | OB 列存自 V4.3.0 引入；TiFlash 需建副本；统计信息须一致 |
| 5 | OLTP+OLAP 混跑 | CH-benCHmark（go-tpc ch） | tpmC + QphH 互扰 | TiFlash 隔离 | major merge 抖动 + 资源组 | 只报 OLAP 总时间会掩盖 OLTP 抖动 |
| 6 | 扩容期间压测 | 压测中加节点 | 迁移期抖动， TPS 曲线 | PD hot_region / Region 迁移 | RootService balancer / LS 迁移 | 迁移窗口必有抖动；扩容后未等均衡就复测 |
| 7 | 节点故障期间压测 | Chaos Mesh pod-kill / TiKV fail_point! | RTO, RPO, P999 | apply lag 突增， RegionError | LS Paxos 切主， `__all_virtual_log_stat` | 多数派可用时服务恢复；把短暂重试误判为数据错误 |
| 8 | DDL 并发压测 | ADD INDEX + 在线 workload | DDL 期吞吐降幅 | `tidb_ddl_reorg_worker_cnt`/`batch_size` 影响 latch wait | ObDDLType 分档（详见第14章） | reorg 参数须限速；目标列被并发写时影响显著 |
| 9 | 长事务 + GC | 开长事务 + 调 `tidb_gc_life_time` | safepoint 推进， 空间放大 | GC safepoint（默认 10min） | row compaction / merge | 长事务阻塞 GC，默认窗口 10min |
| 10 | 大表加索引 | 大表 ADD INDEX | reorg 速度， 在线影响 | reorg worker/batch、backfill | online/offline DDL 路径 | 大表 DDL 必须限速，Online≠零影响 |
| 11 | 租户资源隔离 | 多租户/多 workload 并发 | 隔离有效性， 干扰度 | Resource Control RU（软隔离） | `__all_virtual_tenant_resource_limit*`（硬隔离） | OB 原生硬隔离 vs TiDB RU 软隔离（详见第17章）；Cloud Starter 机制不可断言 |
| 12 | 网络分区 | Chaos Mesh NetworkChaos / tc | 提交成功率， 只读行为， 恢复 | PD/TSO 时序 | GTS 时序 | 无脑裂，历史可校验；Jepsen 风格非吞吐 benchmark |
| 13 | 时钟漂移 | Chaos Mesh TimeChaos / NTP 控制 | 一致性是否破坏 | PD/TSO 可见性 | GTS 可见性 | 注入时序扰动验证 SI 不破坏；生产环境谨慎 |

### 可注入的失效模式

- **集群层（Chaos Mesh，K8s CRD）**：PodChaos（`action: pod-failure`/`pod-kill`/`container-kill`，`mode: one/all/fixed/fixed-percent`）、NetworkChaos（延迟/丢包/乱序/分区）、IOChaos（IO 延迟/读写失败）、TimeChaos（时钟漂移）、KernelChaos。Chaos Mesh 由 PingCAP 开源、起源于 TiDB 测试平台，但作为通用 K8s 工具同样适用于 OceanBase。
- **进程内（确定性）**：TiKV `fail_point!`（需 `failpoints` feature 编译），适合精确触发特定代码分支。OceanBase 社区版亦内置进程内故障注入：`oceanbase/oceanbase` @ `v4.2.5_CE` commit `e7c676806fda` 的 `deps/oblib/src/lib/utility/ob_tracepoint.h` 定义 errsim tracepoint 框架——宏 `ERRSIM_POINT_DEF`、`OB_E(EventTable::EN_...)`（运行期检查事件点）、`TP_SET_EVENT(name, error_in, occur, trigger_freq, ...)`（按错误码/触发次数/频率/会话条件设置注入），全树共定义约 531 个 `EN_*` 事件点；触发由租户参数 `errsim_module_types`（设置注入模块列表）与 `errsim_module_error_percentage`（注入错误百分比 [0,100]，`src/share/parameter/ob_parameter_seed.ipp`）等 `ERRSIM_DEF_*` 配置控制，模块管理见 `src/share/errsim_module/`。该机制官方用户文档未公开发布，故仅以 CE 源码为据。
- **一致性校验（Jepsen 风格）**：用 Knossos 检查单键线性一致，用 Elle 基于环检测（cycle detection）推断事务隔离异常（G0 脏写/G1 脏读/G2 反依赖环/lost update），用 bank/long-fork/set 测试覆盖 SI。建议为 TiDB 与 OceanBase 各设计三类 history：单 key register/bank、跨 Region 或跨 LS 转账、DDL/表创建与并发写入；故障含 partition、pause、crash、clock skew。**注意：线性一致与可串行化校验都是 NP 完全问题，检查器只能证伪、不能证全对**。

### 关键观测指标（按层归档，均已查证或显式标注待查）

- **客户端侧**：TPS/QPS、P50/P95/P99/P999、错误码、重试次数。
- **TiDB/TiKV**：`tikv_raftstore_apply_log_duration_seconds`、`tikv_raftstore_apply_wait_time_duration_secs`、`tikv_raftstore_commit_log_duration_seconds`、`tikv_raftstore_append_log_duration_seconds`、Write stall duration、Compaction duration、Store size；GC 字段 `tikv_gc_safe_point`、`tidb_gc_life_time`；TSO 延迟用 `pd-tso-bench` 单测。
- **OceanBase**：`GV$OB_SQL_AUDIT`（RETRY_CNT/QUEUE_TIME/GET_PLAN_TIME）、`GV$OB_TABLET_COMPACTION_PROGRESS`、`GV$OB_COMPACTION_DIAGNOSE_INFO`、`CDB_OB_MAJOR_COMPACTION`、`__all_virtual_log_stat`、`__all_virtual_tenant_resource_limit`。
- **跨系统未逐一核验项**：各内部视图字段级 schema、部分 Prometheus 序列裸名与日志字段名——具体名称需进一步查证，不写成事实。

Jepsen 风格一致性验证应独立于性能 benchmark：它通过真实分布式系统、客户端操作、故障注入与历史校验来验证不变量，不以固定请求率模拟真实负载，也不是容量规划工具；对 OceanBase 企业版特性或 TiDB Cloud 共享形态，公开资料不足时只写测试假设，不写实现结论。

## 25.10 容易误解点

1. **"总 TPS 高 = 数据库更好"**——错。总 TPS 把 P99/P999 长尾、抖动窗口、故障恢复全部抹平；TPS 只在相同 workload、相同数据规模、相同硬件、相同审计口径、相同调优目标下才有可比性。TiDB 8.5 的实践证明，同样 TPS 下 P999 可差两个数量级。正确做法是报告"TPS + 分位延迟 + 抖动归因"三元组。

2. **"TPC-C/TPC-H 文档里的数字就是 TPC 官方排名"**——错。TiDB Cloud v8.5.0 报告与 OceanBase V4.2.1 TPC-H report 是厂商官方测试披露，TPC 官方结果页/FDR 才是审计结果入口。最常被误用的 707M tpmC 是 TPC-C Result ID 120051701、1554 data nodes、2020、v2.2 企业版的审计成绩（见 §25.7），与任何小集群 / 现代 CE 版本 / 普通硬件的数字都不可比。非审计测试只能写"基于 TPC workload 的内部测试"。

3. **"Jepsen/Elle 通过了就一定正确"**——错。线性一致与可串行化校验是 NP 完全问题，检查器只能在给定历史中证伪（发现反例），无法证明系统在所有历史下都正确。Jepsen 2019 年正是在 TiDB 2.1.7"声称满足 SI"的情况下，发现了 auto-retry 导致的 read skew 与 lost update。"通过 Jepsen"应理解为"在该测试覆盖范围内未发现反例"，而非"绝对正确"；它验证安全性历史，不能替代吞吐/延迟压测，二者应并列运行。

## 25.11 本章结论

1. benchmark 的价值在"数字 + 五前提（workload／硬件／版本／调优／利益相关方）+ 可观测归因"三元组，而非孤立标量；据此推测，公平 benchmark 的最小单位不是数据库产品名，而是 workload、版本、硬件、部署形态、参数、数据分布、故障窗口与观测口径的组合，只报告总 TPS 几乎无价值。
2. 不能只看总 TPS：必须分别观测 P50/P95/P99/P999 长尾、事务冲突率、跨分片比例、compaction/merge 抖动、apply lag、热点分布、GC/safepoint 与控制面延迟——这些只在长尾与故障注入下暴露。TiDB 按 TiDB Server、PD/TSO、TiKV Region / Raft Group、TiFlash、GC/compaction 拆分观测；OceanBase 按 Tenant、ODP、Partition、Tablet、Log Stream（LS）、PALF、merge 拆分观测。
3. 压测工具（go-tpc/obtpcc/ob-sysbench）是外部独立仓库、参数须查证；但被测系统侧观测对象可 commit-pin：TiDB 侧 GC 默认 10min、load-based split QPS 阈值 3000/大 Region 7000（连续 10s）、4 个 raftstore apply/commit/append histogram、`pd-tso-bench`；OceanBase 侧 `GV$OB_SQL_AUDIT`、compaction 系列虚拟表、`__all_virtual_log_stat`、租户资源限制视图——测试预期须锚定这些真实阈值与视图。
4. 完整测试矩阵须覆盖 13 类场景（单／跨分片、热点写、大范围扫描、HTAP 混跑、扩容／故障期压测、DDL 并发、长事务加 GC、大表加索引、租户隔离、网络分区、时钟漂移）；可以推测，稳态 TPS 表现良好并不代表抖动窗口与故障窗口同样可用。
5. 故障注入分三层：Chaos Mesh（集群层 K8s CRD：PodChaos/NetworkChaos/IOChaos/TimeChaos）、进程内确定性注入（TiDB 侧 TiKV `fail_point!` 需 failpoints feature 编译；OceanBase CE 侧内置 errsim tracepoint，由 `errsim_module_types`+`errsim_module_error_percentage` 控制，源码确认）、Jepsen／Elle（一致性校验，只能证伪不能证全）；一个验证可用性与恢复，一个验证历史一致性，据此推测，二者都不能单独代表生产吞吐能力。
6. vendor benchmark 必须连同硬件、版本、审计方、利益相关方一起引用，不可外推到锁定基准或生产：707M tpmC 是 1554 data nodes、2020、v2.2 企业版的 TPC 审计成绩；论文实验数据（如 Bacchus 延迟）不得当生产事实。
7. Raft apply lag（TiKV，RSM 模型）与 PALF/LS 日志状态（OceanBase，复制式 WAL + 事务状态推进）语义不同，不可直接互换比较；compaction 抖动 TiDB 是 RocksDB 本地自治、OceanBase 是 major freeze 全局协调，测试归因路径不同（详见第 3/4 章）。

## 25.12 参考文献

[1] Why Benchmarking Distributed Databases Is So Hard. 官方博客[EB/OL]. https://www.pingcap.com/blog/why-benchmarking-distributed-databases-is-so-hard/.
 （支撑:§25.1/§25.8 关于公平 benchmark、benchmarketing 风险、多连接避免工具瓶颈、拓扑/硬件/配置可复现的论述。）
[2] How to Run TPC-C Test on TiDB. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/benchmark-tidb-using-tpcc/.
 （支撑:§25.2.1/§25.9 go-tpc/TiUP bench 的 TPC-C prepare/run/check/cleanup 命令与 warehouses/threads/time 参数、五类事务口径。）
[3] How to Test TiDB Using Sysbench. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/benchmark-tidb-using-sysbench/.
 （支撑:§25.2.1/§25.7/§25.9 Sysbench 工具适配、prepared statement、日志级别、Default/Write CF 比例对结果的影响。）
[4] How to Run CH-benCHmark Test on TiDB. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/benchmark-tidb-using-ch/.
 （支撑:§25.2.1/§25.7/§25.9 CH-benCHmark HTAP 混合负载、TiFlash 与 TPC-C/TPC-H 组合。）
[5] TiDB Cloud TPC-C Performance Test Report for TiDB v8.5.0. 官方文档[EB/OL]. https://docs.pingcap.com/tidbcloud/v8.5-performance-benchmarking-with-tpcc/.
 （支撑:§25.2.1/§25.7/§25.8 TiDB v8.5.0 的测试方法论（1000 仓、50/100/200 并发、2 小时、tpmC 与硬件拓扑）与非审计边界。）
[6] Load Base Split. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/configure-load-base-split/.
 （支撑:§25.2.2/§25.4/§25.9 热点写测试 load-based split 的 QPS 阈值 3000/7000、byte 阈值、连续 10s 触发条件（反查证源码 split_config.rs）。）
[7] Key Monitoring Metrics of TiKV. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/grafana-tikv-dashboard/.
 （支撑:§25.2.2/§25.9 TiKV apply/commit/append log duration、apply wait、write stall 等 Grafana 面板名与描述（该页明确"Write stall d）
[8] PD Control User Guide. 官方文档[EB/OL]. https://docs.pingcap.com/tidb/stable/pd-control/.
 （支撑:§25.2.2 balance-hot-region-scheduler 默认配置项（min-hot-byte-rate=100、min-hot-key-rate=10、min-hot-query-rate）
[9] Performance test | OceanBase Deployer. 官方文档/源码[EB/OL]. https://github.com/oceanbase/obdeploy/blob/master/docs/en-US/300.obd-command/300.test-command-group.md.
 （支撑:§25.3.1 obd test sysbench/tpcc/tpch 的 tenant/script-name/table-size/threads/warehouses/run-mins 参数、不使用 sys t）
[10] TPC-H benchmark report of OceanBase Database V4.2.1. 官方文档[EB/OL]. https://en.oceanbase.com/docs/common-oceanbase-database-10000000001103503.
 （支撑:§25.3.1/§25.7/§25.8 OceanBase V4.2.1 TPC-H 报告的硬件、版本、tenant、数据规模与非绝对排名边界。）
[11] Simulate Pod Faults (PodChaos) / Basic Features | Chaos Mesh. 官方文档[EB/OL]. https://chaos-mesh.org/docs/simulate-pod-chaos-on-kubernetes/.
 （支撑:§25.6/§25.9 Chaos Mesh PodChaos 的 pod-failure/pod-kill/container-kill action、CRD 结构、注入模式与 Chaos workflow。）
[12] Reducing P999 Latency in Distributed Databases with TiDB 8.5. 官方博客[EB/OL]. https://www.pingcap.com/blog/tidb-8-5-reduce-p999-latency-distributed-database/.
 （支撑:§25.7/§25.10 "不能只看总 TPS"：P999 从数十秒到亚秒、根因 GC 暂停/锁竞争/快照开销、慢查询减少 30%–90%。）
[13] Jepsen: TiDB 2.1.7. 论文/官方分析[EB/OL]. https://jepsen.io/analyses/tidb-2.1.7.
 （支撑:§25.2.1/§25.6/§25.9/§25.10 Jepsen 一致性验证：Knossos/bank/long-fork 检查器、libfaketime 时钟漂移、auto-retry 导致 read skew/lo）
[14] OceanBase: A 707 Million tpmC Distributed Relational Database System. 论文，VLDB[EB/OL]. https://vldb.org/pvldb/vol15/p3385-xu.pdf.
 （支撑:§25.7/§25.8/§25.10 707M tpmC 的 benchmark 条件（Result ID 120051701、v2.2 企业版、1554 data nodes、2020、TPC 审计）及 TPC 官方结）
[15] OceanBase Bacchus: Shared Storage Architecture for Multi-Cloud. 论文[EB/OL]. https://arxiv.org/abs/2602.23571.
 （支撑:§25.3.2/§25.8 Bacchus shared-storage 架构、SysBench/TPC-H 论文实验与"不可外推为生产 OLTP 延迟"的适用范围限制。）
[16] TiKV、PD、OceanBase 源码仓库（commit-pinned 统一基线，详见附录 D 源码索引）[EB/OL]. https://github.com/tikv/tikv; https://github.com/tikv/pd; https://github.com/oceanbase/oceanbase.
 （支撑:§25.2.2/§25.3.2 的 apply/commit/append log duration 指标名、load-based split 阈值常量、fail_point! 机制与注入点规模、PD hot_reg）
## 25.13 信息可信度自评

本章可信度分布如下：

- **官方文档明确**：go-tpc/TiUP bench/Sysbench/CH-benCHmark 与 OBD `obd test` 的命令与参数、TiDB v8.5.0 TPC-C 测试方法论、OceanBase V4.2.1 TPC-H AP 披露样例、load-based split 的 QPS/byte/CPU 阈值与 10s 触发条件、TiKV Grafana 面板名与描述（该页只给描述、未给 apply P99 数值阈值）、Chaos Mesh PodChaos action 与 CRD 结构、OceanBase compaction/sql_audit 诊断视图——均来自 docs.pingcap.com、chaos-mesh.org、oceanbase 官方文档与 obdeploy 仓库，可信度高。
- **源码级信息**：GC 常量（10min）、4 个 raftstore apply/commit/append histogram 指标名（`tikv_raftstore_*`，apply wait 用 `_secs` 后缀）、apply/append log duration 的 `.99` 官方告警阈值 1s（来自 `tikv.rules.yml`）、load-based split 阈值常量、`fail_point!` 机制、`pd-tso-bench`、OceanBase 七组 `__all_virtual_*compaction*` 虚拟表、`GV$OB_SQL_AUDIT`/`__all_virtual_log_stat`/`__all_virtual_tenant_resource_limit` 视图与 `palf_cluster` 测试框架——均来自源码事实卡 commit-pin（统一基线 TiKV/PD `release-8.5`、tidb `release-8.5`、oceanbase `v4.2.5_CE`），可信度高且可复核。
- **论文 / 第三方分析**：707M tpmC 条件（VLDB 论文 + TPC 官方结果页）、Bacchus shared-storage 实验、Jepsen TiDB 2.1.7 一致性发现、Elle 检查器的 NP 完全性——来自 VLDB/arXiv 论文与 jepsen.io 官方分析，可信度高但 Jepsen 版本较老、仅作方法论与历史风险示例。
- **工程推测**：测试矩阵的"预期/踩坑"列、生产前优先覆盖 3–5 项的建议、OBD 自动调优对公平性的影响、P99/P999 比平均 TPS 更接近工程风险——为基于公开资料的合理工程判断，标注（推测）。
- **已由源码补强升级为事实**：PD hot_region scheduler 默认阈值常量（`MinHotByteRate=100`/`MinHotKeyRate=10`/`MinHotQueryRate=10`/`MaxPeerNum=1000`/`Src+DstToleranceRatio=1.05`/`RankFormulaVersion="v2"`，源码 `hot_region_config.go` 与官方 PD Control 文档双向吻合）、TiKV `fail_point!` 注入点规模（store.rs 10 处，整库约 492 处、raftstore 去重约 130 个命名点）、OceanBase CE 进程内故障注入机制（errsim tracepoint：`ERRSIM_POINT_DEF`/`OB_E`/`TP_SET_EVENT`、约 531 个 `EN_*` 事件、`errsim_module_types`+`errsim_module_error_percentage` 控制）——均经 commit-pin 源码核对后由"需进一步查证"升级为。
- **仍不确定 / 需进一步查证**：各 OceanBase 虚拟表的逐字段级完整 schema、部分 Prometheus 序列裸名与日志字段名（未逐一穷举核验）——均显式标注"需进一步查证"，未编造。

反查证要点：早期资料流传的"apply log duration .99 期望 < 100ms"经反查 Grafana 文档与 `tikv.rules.yml` 后证伪，官方 `.99` 告警阈值实为 > 1s，已在 §25.2.2 改以官方告警 1s 为准；load-based split 阈值"文档 3000/7000 ↔ 源码 `DEFAULT_QPS_THRESHOLD`/`DEFAULT_BIG_REGION_QPS_THRESHOLD`"完全吻合；707M tpmC 在 VLDB 论文、TPC 官方结果页（Result ID 120051701）三处一致，明确为 v2.2 企业版 / 1554 data nodes / 2020 成绩，不外推。TiDB Cloud v8.5.0 TPC-C 报告与 OceanBase V4.2.1 TPC-H report 均为厂商官方测试披露，未见 TPC 官方审计声明，已按非审计边界处理。本章未引入与锁定基准（TiDB 8.5.x/7.5.x、OceanBase v4.2.5_CE/4.3.5/4.4.x）冲突的版本号。

---


# 附录


# 附录 A 术语表

> 全书统一术语(不译词保持英文原形)。

| 术语 | 说明 |
|---|---|
| Region | TiDB/TiKV 的分片单元 = 一段连续 Key Range = 一个 Raft Group;由 size/keys/load 自动分裂合并。 |
| Raft Group | 一个 Region 的多副本共识组,基于 tikv/raft-rs(Multi-Raft)。 |
| Tablet | OceanBase 4.x 的物理数据存储对象;一个 Partition 对应一个 Tablet。 |
| Partition | OceanBase 用户逻辑分区对象(RANGE/LIST/HASH)。 |
| Log Stream / LS | OceanBase 4.x 的共识与事务边界容器;一个 LS 关联多个 Tablet,经 Paxos/PALF 复制日志。 |
| PALF | Paxos-backed Append-only Log File system;OceanBase 的复制式 WAL / append-only 日志服务。 |
| TSO | Timestamp Oracle;PD 提供的集群级全局时间戳。 |
| GTS | Global Timestamp Service;OceanBase 租户级全局时间戳。 |
| SCN | System Change Number;OceanBase 版本号/提交号。 |
| Coprocessor | TiKV 下推算子(point/scan/聚合)的执行框架;TiFlash 亦有列存 coprocessor / MPP。 |
| primary lock | Percolator 2PC 中作为事务状态真相源的主锁。 |
| MemTable / SSTable | LSM 引擎的内存写缓冲 / 磁盘有序文件。 |
| RootService (RS) | OceanBase sys tenant 内的逻辑中心化控制面(schema/DDL/Unit/balancer);逻辑中心化但物理高可用,不等于 SPOF。 |
| PD | TiDB 的元数据与调度控制面(内嵌 etcd,etcd Raft);提供 TSO、Region 路由、调度。 |
| Resource Pool / Unit / Tenant | OceanBase 资源隔离层级:Resource Pool 由若干 Unit 组成,分配给 Tenant。 |
| Locality / Primary Zone | OceanBase 副本分布与主副本偏好的描述。 |
| TiFlash disaggregated | TiFlash 存算分离形态(Write Node + Compute Node + S3);v7.0.0 实验性引入、v7.4.0 GA。 |
| Bacchus | OceanBase 云原生 shared-storage 架构(对象存储承载 SSTable + 服务化共享日志 + 三级缓存 + SSWriter 单写者租约)。 |
| SSWriter | Bacchus 中破解对象存储无互斥的单写者租约机制。 |
| MPP | Massively Parallel Processing;TiFlash 的分布式列存执行引擎。 |
| Witness / Arbitration | TiKV Witness 副本 / OceanBase 仲裁成员;两者不可简单类比,仅按验证维度讨论。 |
| async commit / 1PC | TiDB 把跨/单 Region 事务提交延迟压到约一次多数派往返的优化。 |
| major freeze / merge | OceanBase 由 RootService 选全局版本号、跨副本协调出全局一致基线的合并动作。 |
| Raft Engine | TiKV 自 v6.1.0 默认承载共识日志的存储引擎(降写 I/O)。 |

---


# 附录 B 版本基准表

> 全书锁定版本基准。正文与各章版本表述均以本表为准;跨来源版本冲突按本表锁定值书写。

| 组件 / 形态 | 锁定版本 | 说明 |
|---|---|---|
| TiDB LTS 主基准 | 8.5.x(2024-12 GA) | 对照 7.5.x LTS |
| TiKV / PD | release-8.5 | 源码 commit 基线 `tikv@1f8a140b…` / `pd@6dce4a68…`;部分章节另据 `v8.5.0 @a2c58c94` tag 亦确认 |
| TiFlash 存算分离(disaggregated) | **v7.0.0 实验性引入、v7.4.0 GA** | Write Node + Compute Node + S3;不写「v7.0 GA」 |
| OceanBase TP LTS | 4.2.5_CE | 源码 commit `oceanbase@e7c67680…`(v4.2.5_CE tag) |
| OceanBase AP LTS | 4.3.5 | 列存增强;列存/CS_ENCODING 自 **V4.3.0** 引入 |
| OceanBase TP+AP 融合 | 4.4.x | 4.4.1_CE(2025-10-24):`auto_split_tablet_size` 默认 128MB→2GB;部分章节另据 `v4.4.2_CE @e859d1b9` 亦确认 |
| OceanBase shared-storage | Bacchus | 4.3.5+ 文档 / 4.4.x 架构;社区版默认不编译(`OB_BUILD_SHARED_STORAGE`) |
| OBProxy / ODP | 4.2.x / 4.3.x | 路由诊断自 4.2.1 |
| 707M tpmC 历史成绩 | OceanBase v2.2 EE / 2020 | TPC-C Result ID 120051701、Alibaba Cloud ECS、1554 data nodes、官方 FDR 记录日 2020-05-17;不代表 4.x/CE/小集群 |

> 高风险版本归属一览:TiFlash disagg(7.0 实验 / 7.4 GA)、TiKV Region 256 MiB(锁 v8.5.0)、OceanBase 列存(4.3.0)、auto_split 默认值变更(4.4.1_CE)、RPO=0/RTO<8s(限 V4.x LS/Paxos 多副本)。

---


# 附录 C 参考文献总表

> 全书 25 章 §N.12 去重引用,共 **275** 条;类型分布:官方文档 149, 论文 28, 源码 58, 官方博客 33, 第三方 7。

| # | 标题/来源 | 类型 | URL | 被引章 |
|---|---|---|---|---|
| 1 | Understanding Raft Region Size: TiDB Performance & Recovery | 官方博客 | https://www.pingcap.com/blog/understanding-raft-region-size-tidb-performance-recovery/ | 1,9 |
| 2 | TiKV Deep Dive — Data Sharding | 官方博客 | https://tikv.org/deep-dive/scalability/data-sharding/ | 1 |
| 3 | TiKV deep-dive — Multi-raft & Raft / The Design and Implementation of Multi-raft | 官方博客 | https://tikv.org/deep-dive/scalability/multi-raft/ | 1,3 |
| 4 | TiKV Deep Dive — RocksDB / split-check | 官方博客 | https://tikv.org/deep-dive/key-value-engine/rocksdb/ | 1 |
| 5 | 7 Key Technologies to Ensure High Availability in OceanBase Database | 官方博客 | https://en.oceanbase.com/blog/2615184384 | 1,2 |
| 6 | OceanBase RPO and RTO Explained | 官方博客 | https://en.oceanbase.com/blog/rpo-and-rto-explained | 1,3 |
| 7 | TiDB Computing（行/索引 KV 编码规则） | 官方文档 | https://docs.pingcap.com/tidb/stable/tidb-computing/ | 1,2,5,15 |
| 8 | TiDB Configuration File | 官方文档 | https://docs.pingcap.com/tidb/stable/tidb-configuration-file/ | 1 |
| 9 | SHOW TABLE REGIONS | 官方文档 | https://docs.pingcap.com/tidb/stable/sql-statement-show-table-regions/ | 1 |
| 10 | Tune Region Performance | 官方文档 | https://docs.pingcap.com/tidb/stable/tune-region-performance/ | 1 |
| 11 | Load Base Split | 官方文档 | https://docs.pingcap.com/tidb/stable/configure-load-base-split/ | 1,25 |
| 12 | Best Practices for PD Scheduling | 官方文档 | https://docs.pingcap.com/best-practices/pd-scheduling-best-practices/ | 1 |
| 13 | TimeStamp Oracle (TSO) in TiDB | 官方文档 | https://docs.pingcap.com/tidb/stable/tso/ | 1,13 |
| 14 | OceanBase Developer Guide — Architecture | 官方文档 | https://oceanbase.github.io/oceanbase/architecture/ | 1,3,9 |
| 15 | OceanBase Database Architecture (V4.3.5) / FAQ about multi-tenant threads (V4.3.3) / Explo | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001971022 | 1,4,7,8,13,15 |
| 16 | oceanbase/oceanbase @ 4.4.x — src/rootserver/{ob_tenant_balance_service.*,ob_partition_bal | 官方文档 | https://github.com/oceanbase/oceanbase/tree/4.4.x/src/rootserver | 1 |
| 17 | Introducing OceanBase 4.2.5 LTS for TP Scenarios | 源码 | https://en.oceanbase.com/blog/17414420480 | 1 |
| 18 | Release v4.4.1_CE — oceanbase/oceanbase | 源码 | https://github.com/oceanbase/oceanbase/releases/tag/v4.4.1_CE | 1,23 |
| 19 | tikv/tikv @ release-8.5 — components/raftstore/src/coprocessor/config.rs、split_check/、stor | 源码 | https://github.com/tikv/tikv/tree/release-8.5/components/raftstore | 1,3 |
| 20 | pingcap/tidb 仓库(release-8.5 @ commit 67b4876bd57b；另据 v8.5.0 @ d13e52ed6e22 亦确认)— pkg/table | 源码 | https://github.com/pingcap/tidb/tree/release-8.5/pkg/tablecodec | 1,5 |
| 21 | tikv/pd @ release-8.5 — pkg/schedule/splitter/region_splitter.go、pkg/schedule/checker/{mer | 源码 | https://github.com/tikv/pd/tree/release-8.5/pkg | 1 |
| 22 | oceanbase/oceanbase @ v4.2.5_CE — src/storage/ls/ob_ls.h、src/share/ob_ls_id.h、src/storage/ | 源码 | https://github.com/oceanbase/oceanbase/blob/v4.2.5_CE/src/storage/ls/ob_ls.h | 1 |
| 23 | OceanBase Paetica: A Hybrid Shared-nothing/Shared-everything Database | 论文 | https://www.vldb.org/pvldb/vol16/p3728-xu.pdf | 1,6 |
| 24 | PALF: Replicated Write-Ahead Logging for Distributed Databases(VLDB 2024) | 论文 | https://www.vldb.org/pvldb/vol17/p3745-xu.pdf | 1,3,11,13,18,19,22,23 |
| 25 | Introduction to OBProxy: Modules and Features | 官方博客 | https://en.oceanbase.com/blog/2615101184 | 2 |
| 26 | TiKV / Distributed SQL | 官方文档 | https://tikv.org/deep-dive/distributed-sql/dist-sql/ | 2 |
| 27 | Follower Read / Stale Read / Troubleshoot Stale Read | 官方文档 | https://docs.pingcap.com/tidb/stable/follower-read/ | 2,11 |
| 28 | Usage Scenarios of Stale Read / TiDB Docs | 官方文档 | https://docs.pingcap.com/tidb/stable/stale-read/ | 2 |
| 29 | TiDB 8.5.0 Release Notes / Tune Region Performance / TiFlash Overview / AUTO_RANDOM | 官方文档 | https://docs.pingcap.com/tidb/stable/release-8.5.0/ | 2,6,8,14,20 |
| 30 | ODP routing / OceanBase Database Proxy | 官方文档 | https://en.oceanbase.com/docs/common-odp-doc-en-10000000001735865 | 2 |
| 31 | ODP routing best practices / OceanBase | 官方文档 | https://en.oceanbase.com/docs/common-best-practices-10000000001797707 | 2 |
| 32 | Read/Write Splitting / OceanBase | 官方文档 | https://en.oceanbase.com/docs | 2 |
| 33 | 排查 ODP 路由问题(路由诊断)/ OceanBase | 官方文档 | https://oceanbase.github.io/docs/user_manual/operation_and_maintenance/zh-CN/tool_emergency_handbook/odp_troubleshooting_guide/routing_diagnosis | 2 |
| 34 | tikv/client-go — internal/locate/replica_selector.go / region_request.go(master) | 官方文档 | https://github.com/tikv/client-go/blob/master/internal/locate/replica_selector.go | 2 |
| 35 | pingcap/kvproto — proto/metapb.proto(master) | 源码 | https://github.com/pingcap/kvproto/blob/master/proto/metapb.proto | 2 |
| 36 | tikv/client-go — internal/locate/region_cache.go(master) | 源码 | https://github.com/tikv/client-go/blob/master/internal/locate/region_cache.go | 2 |
| 37 | pingcap/tidb — pkg/store/copr/ + pingcap/pd — pkg/core/region.go | 源码 | https://github.com/pingcap/tidb/tree/release-8.5/pkg/store/copr | 2 |
| 38 | oceanbase/oceanbase — src/share/location_cache/ + ob_errno.h + ob_table_location.h + ob_ls | 源码 | https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE/src/share/location_cache | 2 |
| 39 | oceanbase/oceanbase — src/sql/optimizer/ob_table_location.cpp + src/sql/das/ob_das_locatio | 源码 | https://github.com/oceanbase/oceanbase/blob/v4.2.5_CE/src/sql/das/ob_das_location_router.cpp | 2 |
| 40 | oceanbase/obproxy — src/obproxy/proxy/route/ | 源码 | https://github.com/oceanbase/obproxy/tree/master/src/obproxy/proxy/route | 2 |
| 41 | TiDB: A Raft-based HTAP Database | 论文 | https://www.vldb.org/pvldb/vol13/p3072-huang.pdf | 2,9,10,15,20 |
| 42 | The TiKV blog — How TiKV Uses "Lease Read" | 官方博客 | https://tikv.org/blog/lease-read/ | 3 |
| 43 | TiKV Overview / TiDB Storage | 官方文档 | https://docs.pingcap.com/tidb/stable/tikv-overview/ | 3 |
| 44 | Key Monitoring Metrics of TiKV(compaction pending bytes / write stall) | 官方文档 | https://docs.pingcap.com/tidb/stable/grafana-tikv-dashboard/ | 3,5,19,25 |
| 45 | OceanBase Multi-replica log synchronization / Cluster architecture | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001031453 | 3 |
| 46 | Recover time may be very long if TiKV enables hibernate region(pingcap/tidb#34906) | 官方文档 | https://github.com/pingcap/tidb/issues/34906 | 3 |
| 47 | oceanbase/oceanbase 仓库 v4.2.5_CE @ e7c676806fda — src/logservice/palf/(palf_handle/log_sta | 源码 | https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE/src/logservice/palf | 3 |
| 48 | oceanbase/oceanbase 仓库 v4.2.5_CE @ e7c676806fda — src/share/inner_table/ob_inner_table_sch | 源码 | https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE/src/share/inner_table | 3,13 |
| 49 | OceanBase Diagnostic Tool(obdiag) | 源码 | https://github.com/oceanbase/obdiag | 3 |
| 50 | In Search of an Understandable Consensus Algorithm (Raft) | 论文 | https://raft.github.io/raft.pdf | 3 |
| 51 | Paxos Made Simple | 论文 | https://lamport.azurewebsites.net/pubs/paxos-simple.pdf | 3 |
| 52 | Paxos Made Live: An Engineering Perspective | 论文 | https://research.google.com/archive/paxos_made_live.html | 3 |
| 53 | TiDB Storage / RocksDB Overview | 官方文档 | https://docs.pingcap.com/tidb/stable/tidb-storage/ | 4 |
| 54 | TiKV Configuration File(soft / hard-pending-compaction-bytes-limit 默认值) | 官方文档 | https://docs.pingcap.com/tidb/stable/tikv-configuration-file/ | 4,5,19 |
| 55 | Titan Overview | 官方文档 | https://docs.pingcap.com/tidb/stable/titan-overview/ | 4 |
| 56 | TiDB 6.1.0 Release Notes | 官方文档 | https://docs.pingcap.com/tidb/stable/release-6.1.0/ | 4,5 |
| 57 | RocksDB Compaction Wiki | 官方文档 | https://github.com/facebook/rocksdb/wiki/Compaction | 4 |
| 58 | Minor compaction and major compaction | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001104016 | 4,11 |
| 59 | Database object storage / block storage | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001106604 | 4 |
| 60 | Design A Storage Engine for Distributed Relational Database from Scratch | 源码 | https://en.oceanbase.com/blog/2597046272 | 4 |
| 61 | Raft Engine: a Log-Structured Embedded Storage Engine for Multi-Raft Logs in TiKV | 源码 | https://www.infoq.com/articles/raft-engine-tikv-database/ | 4 |
| 62 | tikv/raft-engine 仓库（branch tikv-8.5)— README.md、src/ | 源码 | https://github.com/tikv/raft-engine | 4 |
| 63 | tikv/tikv 仓库（release-8.5)— components/engine_traits/src/、components/engine_rocks/src/、comp | 源码 | https://github.com/tikv/tikv | 4,16 |
| 64 | oceanbase/oceanbase 仓库（tag v4.2.5_CE)— src/storage/{memtable,blocksstable,compaction}/、ob_ | 源码 | https://github.com/oceanbase/oceanbase | 4,16,17,19,22 |
| 65 | An Interpretation of the Source Code of OceanBase: Storage Engine | 源码 | https://www.alibabacloud.com/blog/599324 | 4 |
| 66 | OceanBase: A 707 Million tpmC Distributed Relational Database on a Single-Server Cluster | 论文 | https://vldb.org/pvldb/vol15/p3385-xu.pdf | 4,5,8,9,11,13,17,20,24,25 |
| 67 | TiKV / Percolator(Deep Dive) | 官方博客 | https://tikv.org/deep-dive/distributed-transaction/percolator/ | 5,10,11 |
| 68 | How TiKV reads and writes | 官方博客 | https://tikv.org/blog/how-tikv-reads-writes/ | 5 |
| 69 | Clustered Indexes（TiDB 聚簇索引） | 官方文档 | https://docs.pingcap.com/tidb/stable/clustered-indexes/ | 5,15 |
| 70 | System Variables(tidb_ddl_reorg_* / tidb_enable_dist_task) | 官方文档 | https://docs.pingcap.com/tidb/stable/system-variables/ | 5,14 |
| 71 | RocksDB Overview(TiKV 三 / 四 CF、WAL / MemTable / SSTable、255 字节边界) | 官方文档 | https://docs.pingcap.com/tidb/stable/rocksdb-overview/ | 5 |
| 72 | TiDB Functions: TIDB_DECODE_KEY | 官方文档 | https://docs.pingcap.com/tidb/stable/tidb-functions/ | 5 |
| 73 | OceanBase Block storage / LSM-tree architecture / OBKV-Table data models | 官方文档 | https://en.oceanbase.com/docs/ | 5 |
| 74 | tidb new row format design doc(release-8.5) | 源码 | https://github.com/pingcap/tidb/blob/release-8.5/docs/design/2018-07-19-row-format.md | 5 |
| 75 | tikv/tikv 仓库(release-8.5 @ commit 1f8a140b6d46)— src/storage/mvcc/、components/engine_trait | 源码 | https://github.com/tikv/tikv/tree/release-8.5/src/storage/mvcc | 5 |
| 76 | oceanbase/oceanbase 仓库(v4.2.5_CE @ e7c676806fda;4.3.5_CE @ b28b9bb12f3b；块大小范围另核 v4.4.2_CE  | 源码 | https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE/src/storage/blocksstable | 5 |
| 77 | The Log-Structured Merge-Tree (LSM-Tree) | 论文 | https://dsf.berkeley.edu/cs286/papers/lsm-acta1996.pdf | 5 |
| 78 | Integrated Architecture of OceanBase Database | 官方博客 | https://oceanbase.github.io/docs/blogs/showcases/integrated-architecture | 6 |
| 79 | TiDB Architecture | 官方文档 | https://docs.pingcap.com/tidb/stable/tidb-architecture/ | 6,13 |
| 80 | SQL FAQs / DDL — TiDB Development Guide | 官方文档 | https://docs.pingcap.com/tidb/stable/sql-faq/ | 6 |
| 81 | Temporary Tables | 官方文档 | https://docs.pingcap.com/tidb/stable/temporary-tables/ | 6 |
| 82 | SQL Prepared Execution Plan Cache | 官方文档 | https://docs.pingcap.com/tidb/stable/sql-prepared-plan-cache/ | 6 |
| 83 | TiProxy Overview | 官方文档 | https://docs.pingcap.com/tidb/stable/tiproxy-overview/ | 6 |
| 84 | OceanBase System Architecture(v4.2.5_CE / V4.3.5)与 shared-storage 文档 | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001970960 | 6 |
| 85 | ODP Session Status Synchronization | 官方文档 | https://en.oceanbase.com/docs/common-odp-doc-en-10000000002135951 | 6 |
| 86 | github.com/pingcap/tidb @ release-8.5 @ 67b4876bd57b — pkg/meta/model/job.go、pkg/ddl/、pkg/ | 官方文档 | https://github.com/pingcap/tidb/tree/release-8.5 | 6,9,11,14,15,20,21,23 |
| 87 | session-manager 设计文档(pingcap/tidb docs/design) | 源码 | https://github.com/pingcap/tidb/blob/master/docs/design/2022-07-20-session-manager.md | 6 |
| 88 | oceanbase/oceanbase 仓库(v4.2.5_CE @ commit e7c676806fda)— src/storage/memtable/mvcc/ob_mvcc | 源码 | https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE | 6,9,11,14,15,20,21 |
| 89 | OceanBase Bacchus: a High-Performance and Scalable Cloud-Native Shared Storage Architectur | 论文 | https://arxiv.org/abs/2602.23571 | 6,16,23,25 |
| 90 | OceanBase Database V4.3.5 LTS 发布说明 / 博客(shared storage product 首次正式引入) | 官方博客 | https://en.oceanbase.com/blog/12062885632 | 7,12 |
| 91 | How PingCAP transformed TiDB into a serverless DBaaS using Amazon S3 and Amazon EBS / S3 I | 官方博客 | https://aws.amazon.com/blogs/storage/how-pingcap-transformed-tidb-into-a-serverless-dbaas-using-amazon-s3-and-amazon-ebs/ | 7,22,23 |
| 92 | Secrets Behind TiDB Serverless Architecture | 官方博客 | https://dataturbo.medium.com/secrets-behind-tidb-serverless-architecture-8b277f00cb7c | 7,17 |
| 93 | TiFlash Disaggregated Storage and Compute Architecture and S3 Support | 官方文档 | https://docs.pingcap.com/tidb/stable/tiflash-disaggregated-and-s3/ | 7,16,23 |
| 94 | TiDB 7.0.0 / 7.4.0 Release Notes | 官方文档 | https://docs.pingcap.com/tidb/stable/release-7.0.0/ | 7 |
| 95 | TiDB X Architecture | 官方文档 | https://docs.pingcap.com/tidbcloud/tidb-x-architecture/ | 7,23 |
| 96 | Select Your Plan | 官方文档 | https://docs.pingcap.com/tidbcloud/select-cluster-tier/ | 7 |
| 97 | TiKV MVCC In-Memory Engine + TiKV-Details Grafana Dashboard | 官方文档 | https://docs.pingcap.com/tidb/stable/tikv-in-memory-engine/ | 7,11 |
| 98 | OceanBase Cloud — Storage architecture(shared nothing / shared storage) | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-cloud-10000000002561170 | 7 |
| 99 | OceanBase Cloud — Release notes for 2025 / Supported database versions | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-cloud-10000000001879252 | 7,17 |
| 100 | pingcap/tiflash 仓库 | 源码 | https://github.com/pingcap/tiflash | 7 |
| 101 | OceanBase Bacchus: a High-Performance and Scalable Cloud-Native Shared Storage Architectur | 论文 | https://arxiv.org/abs/2602.23571(PDF: | 7 |
| 102 | Amazon Aurora / Socrates / PolarFS | 论文 | https://www.amazon.science/publications/amazon-aurora-design-considerations-for-high-throughput-cloud-native-relational-databases | 7 |
| 103 | Introduction to the OceanBase Transaction Engine: Features and Applications | 官方博客 | https://oceanbase.medium.com/introduction-to-the-oceanbase-transaction-engine-features-and-applications-27159d10b9d2 | 8,10 |
| 104 | TiDB Latency Breakdown | 官方文档 | https://docs.pingcap.com/tidb/stable/latency-breakdown/ | 8 |
| 105 | TiDB Best Practices on Public Cloud（官方文档，PD/TSO 高并发量化数据） | 官方文档 | https://docs.pingcap.com/tidb/stable/best-practices-on-public-cloud/ | 8 |
| 106 | TiDB Cloud Performance Highlights / TPC-C Report for TiDB v8.5.0 | 官方文档 | https://docs.pingcap.com/tidbcloud/v8.5-performance-highlights/ | 8 |
| 107 | How to Troubleshoot RocksDB Write Stalls in TiKV | 源码 | https://pingcap.medium.com/how-to-troubleshoot-rocksdb-write-stalls-in-tikv-bbb04b88b935 | 8 |
| 108 | tikv/tikv（release-8.5 @ commit 1f8a140b6d46）— components/raftstore/src/coprocessor/config. | 源码 | https://github.com/tikv/tikv/blob/release-8.5/components/raftstore/src/coprocessor/config.rs | 8 |
| 109 | oceanbase/oceanbase（v4.2.5_CE @ commit e7c676806fda）— src/storage/tx/ob_gts_source.h、src/l | 源码 | https://github.com/oceanbase/oceanbase/blob/v4.2.5_CE/src/share/parameter/ob_parameter_seed.ipp | 8 |
| 110 | TPC Benchmark C Full Disclosure Report: Ant Financial OceanBase v2.2 | 第三方 | https://tpc.org/results/fdr/tpcc/ant_financial~tpcc~alibaba_cloud_elastic_compute_service_cluster~fdr~2020-05-17~v01.pdf | 8 |
| 111 | OceanBase: Architecture Overview and Key Concepts | 官方博客 | https://oceanbase.medium.com/oceanbase-architecture-overview-and-key-concepts-858f6b00b47b | 9 |
| 112 | AUTO_RANDOM / TiDB Docs | 官方文档 | https://docs.pingcap.com/tidb/stable/auto-random/ | 9,24 |
| 113 | SHARD_ROW_ID_BITS | 官方文档 | https://docs.pingcap.com/tidb/stable/shard-row-id-bits/ | 9 |
| 114 | Split Region | 官方文档 | https://docs.pingcap.com/tidb/stable/sql-statement-split-region/ | 9 |
| 115 | Clustered Indexes | 官方文档 | https://docs.pingcap.com/tidb/dev/clustered-indexes/ | 9 |
| 116 | tikv/tikv 仓库(release-8.5 @ commit 1f8a140b6d46)— components/txn_types/、components/engine_t | 源码 | https://github.com/tikv/tikv/tree/release-8.5 | 9,11,23 |
| 117 | Apache ShardingSphere — Overview / Sharding & Distributed Transaction | 第三方 | https://shardingsphere.apache.org/document/current/en/features/sharding/ | 9 |
| 118 | TiDB Transaction Isolation Levels | 官方文档 | https://docs.pingcap.com/tidb/stable/transaction-isolation-levels/ | 10,20 |
| 119 | TiDB Optimistic / Pessimistic Transaction Mode | 官方文档 | https://docs.pingcap.com/tidb/stable/pessimistic-transaction/ | 10 |
| 120 | OceanBase 高可用与 RPO/RTO 文档(7 Key Technologies / RPO and RTO Explained / V4.0 Overview) | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001103745 | 10,13,14 |
| 121 | OceanBase MySQL 模式的事务隔离级别（V4.2.2） | 官方文档 | https://www.oceanbase.com/docs/common-oceanbase-database-cn-1000000000510596 | 10 |
| 122 | oceanbase/oceanbase（v4.2.5_CE @ e7c676806fda）— src/share/inner_table/ob_inner_table_schema | 官方文档 | https://github.com/oceanbase/oceanbase/blob/v4.2.5_CE/src/share/inner_table/ob_inner_table_schema_constants.h | 10 |
| 123 | tikv/tikv（release-8.5 @ 1f8a140b6d46）— src/storage/txn/、src/storage/mvcc/、components/txn_t | 源码 | https://github.com/tikv/tikv/blob/release-8.5/src/storage/txn/actions/prewrite.rs | 10 |
| 124 | pingcap/tidb（release-8.5 @ 67b4876bd57b）— pkg/store/driver/txn/txn_driver.go、go.mod | 源码 | https://github.com/pingcap/tidb/blob/release-8.5/pkg/store/driver/txn/txn_driver.go | 10 |
| 125 | oceanbase/oceanbase（v4.2.5_CE @ e7c676806fda）— src/storage/tx/、src/storage/tx_storage/、src | 源码 | https://github.com/oceanbase/oceanbase/blob/v4.2.5_CE/src/storage/tx/ob_committer_define.h | 10 |
| 126 | Deep Dive into Distributed Transactions in TiKV and TiDB | 源码 | https://dataturbo.medium.com/deep-dive-into-distributed-transactions-in-tikv-and-tidb-80337b4104cb | 10 |
| 127 | Async Commit / 1PC / Lock Resolver — TiDB Development Guide | 第三方 | https://pingcap.github.io/tidb-dev-guide/understand-tidb/async-commit.html | 10 |
| 128 | Large-scale Incremental Processing Using Distributed Transactions and Notifications (Perco | 论文 | https://research.google/pubs/large-scale-incremental-processing-using-distributed-transactions-and-notifications/ | 10 |
| 129 | A Tree-Structured Two-Phase Commit Framework for OceanBase: Optimizing Scalability and Con | 论文 | https://arxiv.org/abs/2603.00866 | 10 |
| 130 | OceanBase: A 707 Million tpmC Distributed Relational Database System | 论文 | https://www.vldb.org/pvldb/vol15/p3385-xu.pdf | 10 |
| 131 | Query Performance Unleashed: TiDB's In-Memory Engine (IME) | 官方博客 | https://www.pingcap.com/blog/accelerating-query-performance-tidb-in-memory-engine/ | 11 |
| 132 | Garbage Collection Overview / Configuration | 官方文档 | https://docs.pingcap.com/tidb/stable/garbage-collection-overview/ | 11 |
| 133 | OceanBase 弱一致性读 Weak Consistency Read | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001105870 | 11 |
| 134 | Best practices for read-write splitting in OceanBase | 官方文档 | https://en.oceanbase.com/docs/common-best-practices-10000000001714402 | 11 |
| 135 | An Interpretation of the Source Code of OceanBase (6): Storage Engine | 官方文档 | https://www.alibabacloud.com/blog/an-interpretation-of-the-source-code-of-oceanbase-6-detailed-explanation-of-storage-engine_599324 | 11 |
| 136 | oceanbase/oceanbase 仓库(4.3.5 分支 @ commit b28b9bb12f3b)— src/storage/compaction/ob_compacti | 源码 | https://github.com/oceanbase/oceanbase/tree/4.3.5 | 11 |
| 137 | MVCC in TiKV / How TiKV reads and writes | 源码 | https://pingcap.medium.com/mvcc-in-tikv-f0aa318c564a | 11 |
| 138 | Percolator: Large-scale Incremental Processing Using Distributed Transactions and Notifica | 论文 | https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Peng.pdf | 11,24 |
| 139 | Adaptive Techniques in the OceanBase SQL Execution Engine | 官方博客 | https://oceanbase.github.io/docs/blogs/tech/adaptive-sql-execution-engine | 12 |
| 140 | Query Execution Plan Overview / Explain Statements | 官方文档 | https://docs.pingcap.com/tidb/stable/explain-overview/ | 12 |
| 141 | SQL Physical Optimization / Use TiFlash MPP Mode | 官方文档 | https://docs.pingcap.com/tidb/stable/use-tiflash-mpp-mode/ | 12,16 |
| 142 | Explain Statements That Use Joins / EXPLAIN ANALYZE | 官方文档 | https://docs.pingcap.com/tidb/stable/sql-statement-explain-analyze/ | 12 |
| 143 | TiDB Distributed eXecution Framework (DXF) | 官方文档 | https://docs.pingcap.com/tidb/stable/tidb-distributed-execution-framework/ | 12,14,16 |
| 144 | OceanBase Database Architecture / Understand an Execution Plan V4.x | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000000829726 | 12 |
| 145 | About Partitioned Tables V4.x | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000000829632 | 12 |
| 146 | pingcap/tidb @ release-8.5 commit 67b4876bd57b — pkg/planner/、pkg/executor/、pkg/disttask/f | 源码 | https://github.com/pingcap/tidb/blob/67b4876bd57b/pkg/planner/core/optimizer.go | 12 |
| 147 | oceanbase/oceanbase @ 4.3.5 commit b28b9bb12f3b(交叉 4.2.5 e7c676806fda / 4.4.x d4bef8d29a4c | 源码 | https://github.com/oceanbase/oceanbase/blob/b28b9bb12f3b/src/sql/ob_sql_define.h | 12 |
| 148 | Cost-based Optimization — TiDB Development Guide | 第三方 | https://pingcap.github.io/tidb-dev-guide/understand-tidb/cbo.html | 12 |
| 149 | pingcap/tidb issue #44855 — Index Lookup Join estRows 反例 | 第三方 | https://github.com/pingcap/tidb/issues/44855 | 12 |
| 150 | Building an OceanBase-based Distributed Nearly Real-time Analytical Processing Database Sy | 论文 | https://arxiv.org/html/2602.07584 | 12 |
| 151 | The Cascades Framework for Query Optimization | 论文 | https://15721.courses.cs.cmu.edu/spring2016/papers/graefe-ieee1995.pdf | 12 |
| 152 | Metadata Management Evolved: Scaling TiDB's Placement Driver | 官方博客 | https://www.pingcap.com/blog/scaling-metadata-management-innovations-tidb-placement-driver/ | 13 |
| 153 | TiDB Scheduling | 官方文档 | https://docs.pingcap.com/tidb/stable/tidb-scheduling/ | 13 |
| 154 | PD Microservices | 官方文档 | https://docs.pingcap.com/tidb/stable/pd-microservices/ | 13 |
| 155 | Best Practices for DDL Execution in TiDB(ddl-introduction) | 官方文档 | https://docs.pingcap.com/tidb/stable/ddl-introduction/ | 13,14 |
| 156 | PD Configuration File / Troubleshooting Map | 官方文档 | https://docs.pingcap.com/tidb/stable/pd-configuration-file/ | 13 |
| 157 | Configure Placement Rules | 官方文档 | https://docs.pingcap.com/tidb/stable/configure-placement-rules/ | 13,18 |
| 158 | An Interpretation of the Source Code of OceanBase (5): Life of Tenant | 官方文档 | https://www.alibabacloud.com/blog/an-interpretation-of-the-source-code-of-oceanbase-5-life-of-tenant_599317 | 13 |
| 159 | tikv/pd 仓库(release-8.5 @ 6dce4a68e3e9)— pkg/tso/、pkg/schedule/、pkg/keyspace/、server/config | 源码 | https://github.com/tikv/pd/tree/release-8.5/pkg/tso | 13 |
| 160 | pingcap/tidb 仓库(release-8.5 @ 67b4876bd57b)— pkg/ddl/、pkg/domain/ | 源码 | https://github.com/pingcap/tidb/tree/release-8.5/pkg/ddl | 13 |
| 161 | oceanbase/oceanbase 仓库(v4.2.5_CE @ e7c676806fda；另据 4.3.5 @ b28b9bb12f3b、4.4.x @ d4bef8d29a | 源码 | https://github.com/oceanbase/oceanbase/tree/v4.2.5_CE/src/rootserver | 13 |
| 162 | Time Synchronization in TiDB: Timestamp Oracle (TSO) | 源码 | https://www.pingcap.com/blog/how-an-open-source-distributed-newsql-database-delivers-time-services/ | 13 |
| 163 | How to Make DDL Execution Efficient and Transparent in a Distributed Database | 官方博客 | https://en.oceanbase.com/blog/4934724608 | 14 |
| 164 | How TiDB Achieves 10x Performance Gains in Online DDL | 官方博客 | https://www.pingcap.com/blog/how-tidb-achieves-10x-performance-gains-in-online-ddl/ | 14 |
| 165 | Metadata Lock | 官方文档 | https://docs.pingcap.com/tidb/stable/metadata-lock/ | 14 |
| 166 | ADMIN ALTER DDL JOBS | 官方文档 | https://docs.pingcap.com/tidb/stable/sql-statement-admin-alter-ddl/ | 14 |
| 167 | Online and offline DDL operations, OceanBase Database V4.3.5 | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001974221 | 14 |
| 168 | An Interpretation of the Source Code of OceanBase (7): Implementation Principle of Databas | 官方文档 | https://www.alibabacloud.com/blog/an-interpretation-of-the-source-code-of-oceanbase-7-implementation-principle-of-database-index_599325 | 14 |
| 169 | Online, Asynchronous Schema Change in F1 | 论文 | http://www.vldb.org/pvldb/vol6/p1045-rae.pdf | 14 |
| 170 | 一文详解 OceanBase 2.0 的「全局索引」功能 | 官方博客 | https://developer.aliyun.com/article/672720 | 15 |
| 171 | Global Indexes（TiDB 全局索引） | 官方文档 | https://docs.pingcap.com/tidb/stable/global-indexes/ | 15 |
| 172 | Partitioning（分区表唯一键约束） | 官方文档 | https://docs.pingcap.com/tidb/stable/partitioned-table/ | 15 |
| 173 | TiDB 8.4.0 Release Notes（全局索引 GA） | 官方文档 | https://docs.pingcap.com/tidb/stable/release-8.4.0/ | 15 |
| 174 | Local and Global Indexes / Indexes on partitioned tables / OceanBase Docs | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000000829644 | 15,24 |
| 175 | Global index table-based routing for strong-consistency reads（ODP） | 官方文档 | https://en.oceanbase.com/docs/common-odp-doc-en-10000000001177667 | 15 |
| 176 | An Interpretation of the Source Code of OceanBase(7): Database Index | 源码 | https://www.alibabacloud.com/blog/599325 | 15,24 |
| 177 | 技术分享 / OceanBase 使用全局索引的必要性 | 第三方 | https://opensource.actionsky.com/20230411-oceanbase/ | 15 |
| 178 | A Tree-Structured Two-Phase Commit Framework for OceanBase | 论文 | https://arxiv.org/html/2603.00866 | 15 |
| 179 | OceanBase 4.3.3 功能解析：列存副本 | 官方博客 | https://blog.csdn.net/OceanBaseGFBK/article/details/143581723 | 16 |
| 180 | OceanBase 4.3.3 Released / The first GA version for real-time analysis | 官方博客 | https://en.oceanbase.com/blog/15464061184 | 16 |
| 181 | Resource isolation overview / Why is resource isolation important for HTAP? | 官方博客 | https://oceanbase.github.io/docs/blogs/feat/resource-isolation | 16,17 |
| 182 | The Story Behind OceanBase's Integrated Architecture / OceanBase 4.4.2 LTS 发布说明 | 官方博客 | https://oceanbase.github.io/docs/blogs/arch/all-in-one | 16 |
| 183 | TiFlash Overview | 官方文档 | https://docs.pingcap.com/tidb/stable/tiflash-overview/ | 16 |
| 184 | Use TiDB to Read TiFlash Replicas | 官方文档 | https://docs.pingcap.com/tidb/stable/use-tidb-to-read-tiflash/ | 16 |
| 185 | TiFlash MinTSO Scheduler 与 Configure TiFlash | 官方文档 | https://docs.pingcap.com/tidb/stable/tiflash-configuration/ | 16,23 |
| 186 | Columnar storage / OceanBase Database V4.3.x | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001717168 | 16 |
| 187 | pingcap/tidb 仓库(release-8.5 @ 67b4876bd57b)— pkg/meta/model/resource_group.go、pkg/ddl/reso | 源码 | https://github.com/pingcap/tidb | 16,17 |
| 188 | TiDB: A Raft-based HTAP Database | 论文 | https://vldb.org/pvldb/vol13/p3072-huang.pdf | 16 |
| 189 | Building an OceanBase-based Distributed Nearly Real-time Analytical Processing Database Sy | 论文 | https://arxiv.org/html/2602.07584v1 | 16 |
| 190 | HTAP Databases: A Survey | 论文 | https://arxiv.org/pdf/2404.15670 | 16 |
| 191 | What is TiDB Resource Control? | 官方博客 | https://www.pingcap.com/blog/tidb-resource-control-workload-consolidation-transactional-apps/ | 17 |
| 192 | OceanBase I/O isolation experiences | 官方博客 | https://oceanbase.github.io/docs/blogs/feat/io-isolation | 17 |
| 193 | Use Resource Control to Achieve Resource Group Limitation and Flow Control | 官方文档 | https://docs.pingcap.com/tidb/stable/tidb-resource-control-ru-groups/ | 17 |
| 194 | Manage Runaway Queries / Use Resource Control to Manage Background Tasks | 官方文档 | https://docs.pingcap.com/tidb/stable/tidb-resource-control-runaway-queries/ | 17 |
| 195 | What is TiDB Cloud / TiDB Cloud Starter FAQs | 官方文档 | https://docs.pingcap.com/tidbcloud/tidb-cloud-intro/ | 17 |
| 196 | Create a tenant / Tenant and resource management / Configure cgroups | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001103995 | 17 |
| 197 | tikv/pd 仓库(release-8.5 @ 6dce4a68e3e9)— pkg/mcs/resourcemanager/server/config.go、pkg/mcs/r | 源码 | https://github.com/tikv/pd | 17 |
| 198 | How We Reduced Multi-region Read Latency and Network Traffic by 50% / How Disaster Recover | 官方博客 | https://www.pingcap.com/blog/how-we-reduced-multi-region-read-latency-and-network-traffic-by-50/ | 18 |
| 199 | Two Availability Zones in One Region Deployment（DR Auto-Sync） | 官方文档 | https://docs.pingcap.com/tidb/stable/two-data-centers-in-one-city-deployment/ | 18 |
| 200 | Three Availability Zones in Two Regions Deployment | 官方文档 | https://docs.pingcap.com/tidb/stable/three-data-centers-in-two-cities-deployment/ | 18 |
| 201 | Overview of TiDB Disaster Recovery Solutions / Replicate Data to MySQL-compatible Database | 官方文档 | https://docs.pingcap.com/tidb/stable/dr-solution-introduction/ | 18 |
| 202 | TiDB Log Backup and PITR Command Manual / Guide | 官方文档 | https://docs.pingcap.com/tidb/stable/br-pitr-manual/ | 18 |
| 203 | Log streams / Use the arbitration service of OceanBase Database in a two-IDC scenario | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001105853 | 18 |
| 204 | Data transfer / Overview of physical backup and restore | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001714611 | 18 |
| 205 | OceanBase 4.X-2F1A 仲裁高可用方案初探 | 官方文档 | https://cloud.tencent.com/developer/article/2451338 | 18 |
| 206 | [dr-autosync] v6.5.4 QPS drop during switch sync_recovery to sync（Issue #15366） | 源码 | https://github.com/tikv/tikv/issues/15366 | 18 |
| 207 | The Fundamental Trade-offs in Distributed Databases | 第三方 | https://www.cockroachlabs.com/blog/fundamental-tradeoffs-distributed-databases/ | 18 |
| 208 | Performance Analysis and Tuning(TiDB Docs) | 官方文档 | https://docs.pingcap.com/tidb/stable/performance-tuning-methods/ | 19 |
| 209 | Identify Slow Queries(TiDB Docs) | 官方文档 | https://docs.pingcap.com/tidb/stable/identify-slow-queries/ | 19 |
| 210 | TiDB Dashboard Top SQL Page(TiDB Docs) | 官方文档 | https://docs.pingcap.com/tidb/stable/top-sql/ | 19 |
| 211 | SQL Trace(OceanBase Docs) | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001106685 | 19 |
| 212 | OceanBase Diagnostic Tool(obdiag,官方文档 + 工具仓库) | 官方文档 | https://en.oceanbase.com/docs/obdiag-en | 19 |
| 213 | GV$ACTIVE_SESSION_HISTORY(OceanBase Docs V4.2.0) | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001030063 | 19 |
| 214 | Use OCP-Agent to pull time-series monitoring data(OceanBase Docs) | 官方文档 | https://en.oceanbase.com/docs/common-ocp-10000000001554343 | 19 |
| 215 | pingcap/tidb · tikv/tikv · pingcap/pd(release-8.5) | 源码 | https://github.com/pingcap/tidb/tree/release-8.5/pkg/util/stmtsummary | 19 |
| 216 | Diagnosis and Tuning with OceanBase: sql_audit / Compaction 诊断博客 | 源码 | https://en.oceanbase.com/blog/12526873344 | 19 |
| 217 | Manage memory in OceanBase Database | 源码 | https://oceanbase.medium.com/manage-memory-in-oceanbase-database-2e0f79aade73 | 19 |
| 218 | OceanBase: A 707 Million tpmC Distributed Relational Database System(VLDB 2022) | 论文 | https://dl.acm.org/doi/10.14778/3554821.3554830 | 19 |
| 219 | OceanBase 4.4.2 LTS 发布(MySQL 递归 CTE UNION DISTINCT、Oracle INTERVAL 分区等兼容增强) | 官方博客 | https://en.oceanbase.com/blog/26083442944 | 20 |
| 220 | MySQL Compatibility | 官方文档 | https://docs.pingcap.com/tidb/stable/mysql-compatibility/ | 20 |
| 221 | Foreign Key Constraints | 官方文档 | https://docs.pingcap.com/tidb/stable/foreign-key/ | 20 |
| 222 | Compatibility modes(创建时确定不可改；Community Edition 仅 MySQL 模式) | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001103755 | 20 |
| 223 | Compatibility with MySQL(Community Edition) | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000000829643 | 20 |
| 224 | Compatibility with Oracle Database(Enterprise) | 官方文档 | https://en.oceanbase.com/docs/enterprise-oceanbase-database-en-10000000000905632 | 20 |
| 225 | Privilege Management / Security Compatibility with MySQL / TiDB Docs | 官方文档 | https://docs.pingcap.com/tidb/stable/privilege-management/ | 21 |
| 226 | Enable TLS Between TiDB Clients and Servers / TiDB Docs | 官方文档 | https://docs.pingcap.com/tidb/stable/enable-tls-between-clients-and-servers/ | 21 |
| 227 | Encryption at Rest / TiDB 8.5.0 Release Notes / TiDB Docs | 官方文档 | https://docs.pingcap.com/tidb/stable/encryption-at-rest/ | 21 |
| 228 | TiDB Cloud Dedicated / Essential Database Audit Logging / TiDB Docs | 官方文档 | https://docs.pingcap.com/tidbcloud/tidb-cloud-auditing/ | 21 |
| 229 | OceanBase Database Security Overview / Data Transmission Encryption / Data Storage Encrypt | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001971084 | 21 |
| 230 | Transparent Data Encryption (TDE) / OceanBase Cloud Docs / SSL Link Encryption / ApsaraDB  | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-cloud-10000000002267258 | 21 |
| 231 | 技术分享 / OceanBase 安全审计之透明加密(爱可生开源社区) | 官方文档 | https://opensource.actionsky.com/技术分享-oceanbase-安全审计之透明加密/ | 21 |
| 232 | TiDB Security Enhanced Mode 设计文档(pingcap/tidb docs/design/2021-03-09-security-enhanced-mod | 源码 | https://github.com/pingcap/tidb/blob/master/docs/design/2021-03-09-security-enhanced-mode.md | 21 |
| 233 | tikv/tikv 仓库(release-8.5 @ 1f8a140b6d46)— components/encryption/src/(crypter.rs/config.rs/ | 源码 | https://github.com/tikv/tikv/tree/release-8.5/components/encryption | 21 |
| 234 | CryptDB: Protecting Confidentiality with Encrypted Query Processing(SOSP 2011) | 论文 | https://people.csail.mit.edu/nickolai/papers/raluca-cryptdb.pdf | 21 |
| 235 | TiDB Operator Source Code Reading (I / IV) | 官方博客 | https://www.pingcap.com/blog/tidb-operator-source-code-reading-4-implement-component-control-loop/ | 22 |
| 236 | TiDB Operator Overview / Architecture | 官方文档 | https://docs.pingcap.com/tidb-in-kubernetes/stable/architecture/ | 22 |
| 237 | Comparison Between TiDB Operator v2 and v1 | 官方文档 | https://docs.pingcap.com/tidb-in-kubernetes/dev/v2-vs-v1/ | 22 |
| 238 | Persistent Storage Class Configuration / Backup-Restore CR on Kubernetes | 官方文档 | https://docs.pingcap.com/tidb-in-kubernetes/stable/configure-storage-class/ | 22 |
| 239 | Automatic Failover | 官方文档 | https://docs.pingcap.com/tidb-in-kubernetes/stable/use-auto-failover/ | 22 |
| 240 | StatefulSets / Volumes: local volume | 官方文档 | https://kubernetes.io/docs/concepts/storage/volumes/ | 22 |
| 241 | Introduction / Architecture / ob-operator | 官方文档 | https://oceanbase.github.io/ob-operator/docs/developer/arch | 22 |
| 242 | Create a cluster / Recover from node failure / ob-operator | 官方文档 | https://oceanbase.github.io/ob-operator/docs/manual/ob-operator-user-guide/high-availability/disaster-recovery-of-ob-operator | 22 |
| 243 | pingcap/tidb 内核仓库(release-8.5 @ 67b4876bd57b)— pkg/server/server.go、pkg/util/cgroup/、pkg/u | 源码 | https://github.com/pingcap/tidb/tree/release-8.5/pkg/util/cgroup | 22 |
| 244 | tikv/tikv 与 tikv/pd 内核仓库(release-8.5)— src/storage/config.rs、placement Rule | 源码 | https://github.com/tikv/tikv/tree/release-8.5/src/storage | 22 |
| 245 | Shared storage architecture / Multi-level caching in shared storage | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000003461903 | 23 |
| 246 | Neon architecture | 官方文档 | https://neon.com/docs/introduction/architecture-overview | 23 |
| 247 | Best practices design patterns: optimizing Amazon S3 performance | 官方文档 | https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html | 23 |
| 248 | Object Storage: The New Backbone of Database Architecture | 官方文档 | https://www.pingcap.com/blog/object-storage-new-backbone-data-architecture/ | 23 |
| 249 | Notes On: Disaggregated OLTP Systems | 官方文档 | https://transactional.blog/notes-on/disaggregated-oltp | 23 |
| 250 | oceanbase/oceanbase(ref 4.4.x)— src/share/shared_storage/、src/storage/macro_cache/、src/sto | 源码 | https://github.com/oceanbase/oceanbase/tree/4.4.x/src/share/shared_storage | 23 |
| 251 | Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases | 论文 | https://pages.cs.wisc.edu/~yxy/cs764-f20/papers/aurora-sigmod-17.pdf | 23 |
| 252 | Socrates: The New SQL Server in the Cloud | 论文 | https://www.microsoft.com/en-us/research/wp-content/uploads/2019/05/socrates.pdf | 23 |
| 253 | Cloud-Native Database Systems at Alibaba: Opportunities and Challenges | 论文 | https://users.cs.utah.edu/~lifeifei/papers/vldb-cloud-native.pdf | 23 |
| 254 | Migrate and Merge MySQL Shards of Large Datasets to TiDB / TiDB Docs | 官方文档 | https://docs.pingcap.com/tidb/stable/migrate-large-mysql-shards-to-tidb/ | 24 |
| 255 | TiDB Data Migration 文档族：Overview / Relay Log / Safe Mode / Shard Merge Best Practices / Be | 官方文档 | https://docs.pingcap.com/tidb/stable/dm-overview/(另：relay-log、dm-safe-mode、shard-merge-best-practices、dm-best-practices | 24 |
| 256 | SHARD_ROW_ID_BITS 与 SPLIT REGION / Pre-split Regions / TiDB Docs | 官方文档 | https://docs.pingcap.com/tidb/stable/shard-row-id-bits/(另：sql-statement-split-region | 24 |
| 257 | Data Check in the Sharding Scenario(sync-diff-inspector)/ TiDB Docs | 官方文档 | https://docs.pingcap.com/tidb/stable/shard-diff/ | 24 |
| 258 | OceanBase Migration Service(OMS / What is OMS) | 官方文档 | https://en.oceanbase.com/docs/oms-en | 24 |
| 259 | Best practices for table design and index optimization / OceanBase Docs | 官方文档 | https://en.oceanbase.com/docs/common-best-practices-10000000002431625 | 24 |
| 260 | Create a table group / OceanBase Docs | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-cloud-10000000001733899 | 24 |
| 261 | Create a tenant / Compaction monitoring views / OceanBase Docs | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001228779 | 24 |
| 262 | pingcap/tidb(release-8.5 @ commit 67b4876bd57b)— pkg/meta/autoid/autoid.go、pkg/meta/autoid | 源码 | https://github.com/pingcap/tidb/blob/release-8.5/pkg/meta/autoid/autoid.go | 24 |
| 263 | oceanbase/oceanbase(v4.2.5_CE @ commit e7c676806fda)— ob_create_tablegroup_resolver.cpp、de | 源码 | https://github.com/oceanbase/oceanbase/blob/v4.2.5_CE/src/share/schema/ob_schema_struct.h | 24 |
| 264 | oceanbase/oceanbase(v4.2.5_CE @ commit e7c676806fda)— src/sql/resolver/ddl/ob_ddl_resolver | 源码 | https://github.com/oceanbase/oceanbase/blob/v4.2.5_CE/src/sql/resolver/ddl/ob_ddl_resolver.cpp | 24 |
| 265 | Why Benchmarking Distributed Databases Is So Hard | 官方博客 | https://www.pingcap.com/blog/why-benchmarking-distributed-databases-is-so-hard/ | 25 |
| 266 | Reducing P999 Latency in Distributed Databases with TiDB 8.5 | 官方博客 | https://www.pingcap.com/blog/tidb-8-5-reduce-p999-latency-distributed-database/ | 25 |
| 267 | How to Run TPC-C Test on TiDB | 官方文档 | https://docs.pingcap.com/tidb/stable/benchmark-tidb-using-tpcc/ | 25 |
| 268 | How to Test TiDB Using Sysbench | 官方文档 | https://docs.pingcap.com/tidb/stable/benchmark-tidb-using-sysbench/ | 25 |
| 269 | How to Run CH-benCHmark Test on TiDB | 官方文档 | https://docs.pingcap.com/tidb/stable/benchmark-tidb-using-ch/ | 25 |
| 270 | TiDB Cloud TPC-C Performance Test Report for TiDB v8.5.0 | 官方文档 | https://docs.pingcap.com/tidbcloud/v8.5-performance-benchmarking-with-tpcc/ | 25 |
| 271 | PD Control User Guide | 官方文档 | https://docs.pingcap.com/tidb/stable/pd-control/ | 25 |
| 272 | Performance test / OceanBase Deployer | 官方文档 | https://github.com/oceanbase/obdeploy/blob/master/docs/en-US/300.obd-command/300.test-command-group.md | 25 |
| 273 | TPC-H benchmark report of OceanBase Database V4.2.1 | 官方文档 | https://en.oceanbase.com/docs/common-oceanbase-database-10000000001103503 | 25 |
| 274 | Simulate Pod Faults (PodChaos) / Basic Features / Chaos Mesh | 官方文档 | https://chaos-mesh.org/docs/simulate-pod-chaos-on-kubernetes/ | 25 |
| 275 | Jepsen: TiDB 2.1.7 | 论文 | https://jepsen.io/analyses/tidb-2.1.7 | 25 |

---


# 附录 D 源码索引

> 全书 commit-pinned 源码引用所涉仓库(基线见附录 B)。

| 仓库 | 被引章 |
|---|---|
| github.com/facebook/rocksdb | 4 |
| github.com/oceanbase/obdeploy | 25 |
| github.com/oceanbase/obdiag | 3,19 |
| github.com/oceanbase/obproxy | 2 |
| github.com/oceanbase/oceanbase | 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,19,20,21,22,23,24 |
| github.com/pingcap/go-tpc | 25 |
| github.com/pingcap/kvproto | 2 |
| github.com/pingcap/pd | 19 |
| github.com/pingcap/tidb | 1,2,3,5,6,9,10,11,12,13,14,15,16,17,19,20,21,22,23,24 |
| github.com/pingcap/tiflash | 7,23 |
| github.com/tikv/client-go | 2,10 |
| github.com/tikv/pd | 1,8,13,17,22 |
| github.com/tikv/raft-engine | 4 |
| github.com/tikv/raft-rs | 3 |
| github.com/tikv/rust-rocksdb.git | 4 |
| github.com/tikv/tikv | 1,3,4,5,8,9,10,11,16,18,19,21,22,23 |

> 锁定基线:`tikv@1f8a140b`(release-8.5)、`pd@6dce4a68`、`oceanbase@e7c67680`(v4.2.5_CE);部分章另据 `v8.5.0@a2c58c94`、`v4.4.2_CE@e859d1b9` 亦确认。

---


# 附录 E 图目录

> 全书 Mermaid 图清单(共 79 张)。每章除 §N.5 正常路径图、§N.6 故障路径图外，另含结构 / 层级 / 位域 / 流程等补充图，均以白底纵向 SVG 嵌入。

| 图号 | 章 | 所在小节 | 图题 |
|---|---|---|---|
| 图 1-1 | 1 分片 | §1.1 | 一个分片的多重身份归属映射 |
| 图 1-2 | 1 分片 | §1.5 | 分片正常读写/调度路径 |
| 图 1-3 | 1 分片 | §1.6 | 分片故障/异常路径 |
| 图 2-1 | 2 路由 | §2.3.1 | OceanBase 路由的 Partition 至 Tablet 至 Log Stream 三层结构 |
| 图 2-2 | 2 路由 | §2.3.4 | OBServer 强读 / 弱读副本判定决策链 |
| 图 2-3 | 2 路由 | §2.5 | 路由正常读写/调度路径 |
| 图 2-4 | 2 路由 | §2.6 | 路由故障/异常路径 |
| 图 3-1 | 3 共识 | §3.2 | TiKV 与 OceanBase 共识栈并列分层 |
| 图 3-2 | 3 共识 | §3.3 | PALF Reconfirm / Leader 切换状态机 |
| 图 3-3 | 3 共识 | §3.5 | 共识正常读写/调度路径 |
| 图 3-4 | 3 共识 | §3.5 | 共识正常读写/调度路径 |
| 图 3-5 | 3 共识 | §3.6 | 共识故障/异常路径 |
| 图 4-1 | 4 存储引擎 | §4.3 | SSTable ⊃ Macroblock ⊃ Microblock 嵌套结构 |
| 图 4-2 | 4 存储引擎 | §4.5 | 存储引擎正常读写/调度路径 |
| 图 4-3 | 4 存储引擎 | §4.6 | 存储引擎故障/异常路径 |
| 图 5-1 | 5 KV / LSM 与关系模型映射 | §5.5 | KV/LSM 与关系模型映射正常读写/调度路径 |
| 图 5-2 | 5 KV / LSM 与关系模型映射 | §5.6 | KV/LSM 与关系模型映射故障/异常路径 |
| 图 6-1 | 6 无状态 SQL 层 | §6.3 | OBServer 单进程内模块组成（与 TiDB 多组件分离对照） |
| 图 6-2 | 6 无状态 SQL 层 | §6.5 | 无状态 SQL 层正常读写/调度路径 |
| 图 6-3 | 6 无状态 SQL 层 | §6.6 | 无状态 SQL 层故障/异常路径 |
| 图 7-1 | 7 存算分离 | §7.5 | 存算分离正常读写/调度路径 |
| 图 7-2 | 7 存算分离 | §7.6 | 存算分离故障/异常路径 |
| 图 8-1 | 8 性能 | §8.5 | 性能正常读写/调度路径 |
| 图 8-2 | 8 性能 | §8.6 | 性能故障/异常路径 |
| 图 9-1 | 9 分库分表替代 | §9.4 | 多表物理落点对照 |
| 图 9-2 | 9 分库分表替代 | §9.5 | 分库分表替代正常读写/调度路径 |
| 图 9-3 | 9 分库分表替代 | §9.6 | 分库分表替代故障/异常路径 |
| 图 10-1 | 10 分布式事务模型 | §10.3 | OceanBase 树形 2PC 协调者拓扑 |
| 图 10-2 | 10 分布式事务模型 | §10.5 | 分布式事务模型正常读写/调度路径 |
| 图 10-3 | 10 分布式事务模型 | §10.5 | 分布式事务模型正常读写/调度路径 |
| 图 10-4 | 10 分布式事务模型 | §10.6 | 分布式事务模型故障/异常路径 |
| 图 10-5 | 10 分布式事务模型 | §10.6 | 分布式事务模型故障/异常路径 |
| 图 11-1 | 11 MVCC 与读路径 | §11.2.2 | TiKV 三列族物理布局与一次快照读跨 CF 的指针追踪 |
| 图 11-2 | 11 MVCC 与读路径 | §11.5 | MVCC 与读路径正常读写/调度路径 |
| 图 11-3 | 11 MVCC 与读路径 | §11.6 | MVCC 与读路径故障/异常路径 |
| 图 12-1 | 12 SQL 优化器与执行引擎 | §12.5 | SQL 优化器与执行引擎正常读写/调度路径 |
| 图 12-2 | 12 SQL 优化器与执行引擎 | §12.6 | SQL 优化器与执行引擎故障/异常路径 |
| 图 13-1 | 13 元数据与控制面 | §13.3.1 | OceanBase 资源层级与内部表 / GTS 映射 |
| 图 13-2 | 13 元数据与控制面 | §13.5 | 元数据与控制面正常读写/调度路径 |
| 图 13-3 | 13 元数据与控制面 | §13.5 | 元数据与控制面正常读写/调度路径 |
| 图 13-4 | 13 元数据与控制面 | §13.6 | 元数据与控制面故障/异常路径 |
| 图 13-5 | 13 元数据与控制面 | §13.6 | 元数据与控制面故障/异常路径 |
| 图 14-1 | 14 DDL 与 Online Schema Change | §14.5 | DDL 与 Online Schema Change正常读写/调度路径 |
| 图 14-2 | 14 DDL 与 Online Schema Change | §14.5 | DDL 与 Online Schema Change正常读写/调度路径 |
| 图 14-3 | 14 DDL 与 Online Schema Change | §14.6 | DDL 与 Online Schema Change故障/异常路径 |
| 图 14-4 | 14 DDL 与 Online Schema Change | §14.6 | DDL 与 Online Schema Change故障/异常路径 |
| 图 15-1 | 15 全局索引 / 本地索引 | §15.2.1 | TiDB 五类 key 字节布局 |
| 图 15-2 | 15 全局索引 / 本地索引 | §15.5 | 全局索引/本地索引正常读写/调度路径 |
| 图 15-3 | 15 全局索引 / 本地索引 | §15.6 | 全局索引/本地索引故障/异常路径 |
| 图 16-1 | 16 HTAP / AP 架构 | §16.5 | HTAP/AP 架构正常读写/调度路径 |
| 图 16-2 | 16 HTAP / AP 架构 | §16.6 | HTAP/AP 架构故障/异常路径 |
| 图 17-1 | 17 多租户与资源隔离 | §17.3.1 | OceanBase 租户资源容器层级 |
| 图 17-2 | 17 多租户与资源隔离 | §17.5 | 多租户与资源隔离正常读写/调度路径 |
| 图 17-3 | 17 多租户与资源隔离 | §17.6 | 多租户与资源隔离故障/异常路径 |
| 图 18-1 | 18 容灾、高可用与多地多活 | §18.2.3 | TiDB 三地两中心（5 副本）多地多中心地理部署拓扑 |
| 图 18-2 | 18 容灾、高可用与多地多活 | §18.5 | 容灾、高可用与多地多活正常读写/调度路径 |
| 图 18-3 | 18 容灾、高可用与多地多活 | §18.5 | 容灾、高可用与多地多活正常读写/调度路径 |
| 图 18-4 | 18 容灾、高可用与多地多活 | §18.6 | 容灾、高可用与多地多活故障/异常路径 |
| 图 18-5 | 18 容灾、高可用与多地多活 | §18.6 | 容灾、高可用与多地多活故障/异常路径 |
| 图 19-1 | 19 运维 / 观测 / 调优体系 | §19.5 | 运维/观测/调优体系正常读写/调度路径 |
| 图 19-2 | 19 运维 / 观测 / 调优体系 | §19.6 | 运维/观测/调优体系故障/异常路径 |
| 图 20-1 | 20 兼容性模型 | §20.1 | 兼容性五层光谱分层栈：TiDB 与 OceanBase 各层落点对照 |
| 图 20-2 | 20 兼容性模型 | §20.5 | 兼容性模型正常读写/调度路径 |
| 图 20-3 | 20 兼容性模型 | §20.6 | 兼容性模型故障/异常路径 |
| 图 21-1 | 21 安全 / 权限 / 审计 / 加密 | §21.5 | 安全/权限/审计/加密正常读写/调度路径 |
| 图 21-2 | 21 安全 / 权限 / 审计 / 加密 | §21.6 | 安全/权限/审计/加密故障/异常路径 |
| 图 22-1 | 22 云原生与 Kubernetes | §22.2 | cgroup → GOMAXPROCS 容器自感知流程 |
| 图 22-2 | 22 云原生与 Kubernetes | §22.5 | 云原生与 Kubernetes正常读写/调度路径 |
| 图 22-3 | 22 云原生与 Kubernetes | §22.6 | 云原生与 Kubernetes故障/异常路径 |
| 图 23-1 | 23 存储池化 / Shared-storage | §23.5 | 存储池化/Shared-storage正常读写/调度路径 |
| 图 23-2 | 23 存储池化 / Shared-storage | §23.5 | 存储池化/Shared-storage正常读写/调度路径 |
| 图 23-3 | 23 存储池化 / Shared-storage | §23.6 | 存储池化/Shared-storage故障/异常路径 |
| 图 23-4 | 23 存储池化 / Shared-storage | §23.6 | 存储池化/Shared-storage故障/异常路径 |
| 图 24-1 | 24 迁移传统分库分表系统的方法论 | §24.2.1 | DM 组件拓扑 |
| 图 24-2 | 24 迁移传统分库分表系统的方法论 | §24.2.3 | AUTO_RANDOM 64 位行键位域布局 |
| 图 24-3 | 24 迁移传统分库分表系统的方法论 | §24.5 | 迁移传统分库分表系统的方法论正常读写/调度路径 |
| 图 24-4 | 24 迁移传统分库分表系统的方法论 | §24.6 | 迁移传统分库分表系统的方法论故障/异常路径 |
| 图 25-1 | 25 Benchmark 与测试设计 | §25.5 | Benchmark 与测试设计正常读写/调度路径 |
| 图 25-2 | 25 Benchmark 与测试设计 | §25.6 | Benchmark 与测试设计故障/异常路径 |

---


# 附录 F 冲突清单与裁决

> 本书在融合两份独立研究稿时,对高风险事实逐项做了第三方核验与裁决,统一写法如下。证据优先级:源码/官方文档/论文原文/TPC 官方结果 > 第三方解读;证据不足处采用保守、可证伪表达,未核验细节不写成事实。

| # | 冲突主题 | 裁决与全书统一写法 |
|---|---|---|
| 1 | TiFlash disaggregated GA 版本 | v7.0.0 实验性引入、v7.4.0 GA;不写「v7.0 GA」。 |
| 2 | TiKV Region 默认大小版本归属 | 锁 v8.5.0 classic RaftKV 默认 256 MiB;8.3/8.4 起点差异仅作版本口径备注。 |
| 3 | PRE_SPLIT_REGIONS 公式 | 采用官方 Split Region 文档公式 `2^(PRE_SPLIT_REGIONS)`(纠正早期 off-by-one)。 |
| 4 | PALF / MemTable / WAL | 写「复制式 WAL / Paxos-backed append-only log + 事务状态推进」;删除「先改可见 MemTable / 存储引擎后写日志」强断言。 |
| 5 | RootService 是否 SPOF | 正文不下「SPOF/Achilles heel」定论;按逻辑中心化 / 性能瓶颈 / 高可用单点 / 元数据依赖 / 故障恢复依赖五维度讨论。 |
| 6 | OceanBase RPO=0 / RTO<8s 边界 | 仅限 V4.x LS/Paxos 多副本、少数派故障或多数派完整场景;不绑定 RootService、不外推单副本 shared-storage。 |
| 7 | shared-storage / Bacchus / SSWriter | 按版本与产品形态限定(4.3.5+ 文档 / 4.4.x 架构;社区版默认不编译);不写「4.2.5 已支持」或「CE 默认启用」。 |
| 8 | OceanBase 列存版本 | CS_ENCODING 列式编码自 V4.3.0 引入(非 4.3.5 新增)。 |
| 9 | auto_split_tablet_size | 仅作 4.4.1_CE release note 的 128MB→2GB 配置变更;4.3.x 引入与全版本默认需复核。 |
| 10 | SQL Audit vs Security Audit | `GV$OB_SQL_AUDIT` 是 SQL 执行诊断视图,不等于合规安全审计。 |
| 11 | TiDB Online DDL 与 F1 | 写「与 F1 在线异步 schema change 思想一致」;不写「官方/官方博客基于 F1」(归因属（推测）)。 |
| 12 | Witness vs Arbitration | 只写不可类比和验证维度;细粒度投票 / 日志语义未核验则不下结论。 |
| 13 | OceanBase 707M tpmC | TPC-C Result ID 120051701、OceanBase v2.2 EE、Alibaba Cloud ECS、1554 data nodes、2020(FDR 记录日 2020-05-17);不代表 4.x/CE/小集群。 |
| 14 | metric / PromQL / 虚拟表 / 默认阈值 | 只写 dashboard / 视图观测层级;未核验裸 metric、阈值、视图名、采样间隔不入正文事实,标「需进一步查证」。 |

## 仍建议发布前复核的开放项(摘要)

- OceanBase 侧具体 Prometheus metric 名 / 内部视图名 / 部分参数默认值:多处标「需进一步查证」,未写成事实。
- Bacchus 对象存储延迟(>80ms)与缓存命中率(99%~99.9%):官方未发布统一规格数值,全书按定性表述并标「需进一步查证」。
- `en.oceanbase.com` 整站对自动访问返回限流(WAF challenge),部分官方页正文未能程序化逐字核验,已用源码 + 论文 + 镜像交叉补强。
- 部分 OceanBase 4.4.x shared-storage 实现的精确 commit / 参数枚举:仅确认到模块/文件级,未照搬脱敏符号名。

---


# 附录 G 各章核心结论汇编

> 取自各章 §N.11 本章结论的要点。

### 第 1 章 分片
1. TiDB 把物理分片、共识分片、调度分片、单分片事务边界统一为同一个对象 Region(= Key Range = Raft Group)，分片边界由 size/keys/load/split-key 在运行时自动涌现，对用户基本透明；Region 与 SQL 表、索引、Partition 非一一对应。
2. OceanBase 4.x 把这几类分片解耦：Partition(逻辑)→ Tablet(物理存储对象，常 1:1 物理分区)→ Log Stream(共识/调度/事务参与者)，LS 与 Tablet 是一对多、可重绑定；Tablet 不自带日志流，归属于所在 LS。
3. OceanBase 4.0 用 Log Stream 取代早期"每分区一个 Paxos 组 / partition leader"模型，把共识组数量从分区数量级降到 LS 数量级，使 Tablet 可在 LS 间迁移而不重建共识组。(机制层面)；其降低复制组与事务参与者开销、支撑单机-分布式一体化的动机。

### 第 2 章 路由
1. TiDB 是客户端侧路由：无状态 TiDB Server 内嵌 client-go 的 RegionCache，以 LocateKey 把 SQL 谓词映射为编码后的 KV key range 并定位 Region/Leader,PD 为权威元数据源；RegionCache 是性能缓存而非正确性边界，路由由 RegionError(EpochNotMatch/NotLeader
2. OceanBase 是代理侧两段式路由：ODP 先经 cluster/tenant/表级路由，结合 partition pruning 与"本地→全局→OBServer 异步查询"多级缓存定位 tablet → Log Stream → leader，由 OBServer 反馈包驱动强制刷新；路由表"通常(Generally)"主因为 major compaction / 负载
3. "能路由到 Leader"不等于"只能走 Leader"：两系统都提供强一致读(到 Leader)与弱一致/Follower/Stale 读(到副本/就近)，路由策略随一致性要求分叉。

### 第 3 章 共识
1. TiDB 的共识是 Multi-Raft over tikv/raft-rs：算法本体在外部 crate(raft 0.7.0 + patch master),raftstore 把单 Raft 包成 Region=Raft Group 并叠加 PreVote（默认开）、CheckQuorum、Lease Read（9s）、Hibernate Region（默认开）、Joint
2. OceanBase 的共识是 Multi-Paxos over PALF:复制单位是 Log Stream,采用复制式 WAL / Paxos-backed append-only log + 事务状态推进（而非经典 RSM 的编排），LSN 字节滑动窗口管理日志，选举（lease + priority）与 Paxos 日志复制解耦，新 Leader 上任先 reconfirm
3. 两者安全性等价，核心取舍是约束放置点不同：Raft 把安全约束前移到选举（日志最新者当选、不回补，实现简单），PALF 后移到选举后的 reconfirm（任意优先副本可当选 + Paxos Phase-1 回补 + pending follower 判定未决日志），换来 Leader 位置可控与备库同步等数据库级能力，代价是恢复路径更复杂。+

### 第 4 章 存储引擎
1. TiDB/TiKV 复用 RocksDB(fork tikv/rust-rocksdb),KV 数据主路径仍以 RocksDB 为本地持久化引擎，源码通过 engine_traits/engine_rocks 抽象与实现这层能力；数据走 default/lock/write 三 MVCC CF(raft CF 历史遗留），关系语义在引擎之上编码（源码 txn.rs 可见 loc
2. OceanBase 100% 自研 LSM-like 引擎（src/storage/ 零 RocksDB)，三级 SSTable(Mini/Minor/Major)+ 2MB Macroblock + 变长 Microblock,MemTable 用自研 ObQueryEngine(B-tree + hash)；这套结构与 Tablet、SCN、merge、校验等数据库内核状态
3. 二者最根本的差异在控制路径：RocksDB compaction 本地自治、无全局快照;OceanBase major freeze 由 RootService 选全局版本号、跨副本协调出全局一致基线 + checksum——这服务于分布式一致读与租户级合并。

### 第 5 章 KV / LSM 与关系模型映射
1. KV 是逻辑抽象、LSM 是物理结构，二者必须分开看：关系如何被寻址(KV)与字节如何被持久化(LSM)是两个独立问题。
2. TiDB 走 SQL-on-KV：pkg/tablecodec 把行 / 索引压平成不透明 KV(行 key=t{tid}_r{handle}、索引 key=t{tid}_i{iid}{vals}、行格式 CodecVer=128),Get / Scan / 回表都可从 key range 推导；TiKV 再以 Percolator 三 CF + append_ts 落到 Ro
3. OceanBase 不是 SQL-on-KV，是关系原生 SSTable：blocksstable 直接组织行 / 列 / rowkey / 索引块，宏块 2MB + 变长微块(可调 1KB–1MB)+ PAX 行列混合编码(ObRowStoreType)，CS_ENCODING 列式编码自 4.3.0 引入(非 4.3.5);MemTable / SSTable / macr

### 第 6 章 无状态 SQL 层
1. 无状态 SQL 层的严格定义是"本地不持有不可重建的权威持久状态"，而非"无缓存"；TiDB Server 是典型无状态 SQL 层，本地仅有可从 TiKV/PD/etcd 重建的 schema/plan/session 缓存与连接状态，这些状态影响连接迁移与错误处理。
2. TiDB 用 schema lease(默认 45s,7.5.x/8.5.x 一致；lease/2 续租 reload)+ etcd 全局版本 + 至多两版本并存维持无状态节点间 schema 一致；与 etcd 断连时主动停 schema validator、禁 DML(报 ErrInfoSchemaExpired)，是"无状态 ≠ 无缓存"的核心证据(注：45s 默认值跨两
3. TiDB 把会话状态做成可序列化(SessionStates)，配 TiProxy(v8.0.0 GA)经 SHOW/SET SESSION_STATES 做连接级平滑切换；但 TiDB 意外宕机、活跃事务、本地临时表、未读游标等无法迁移(ErrCannotMigrateSession,session:8146)，无状态化是"部分无状态"；DDL owner 是落在某 TiDB

### 第 7 章 存算分离
1. 存算分离 ≠ 无状态 SQL 层，也 ≠「数据上对象存储」。 TiDB classic 有无状态 SQL 层，但 TiKV 在 release-8.5 仍是 shared-nothing 存算一体(cloud/external_storage 仅服务 backup/BR/CDC，非在线数据面)；真正要拆开的是用户数据、复制日志、元数据、事务状态、缓存与后台任务五类状态。
2. TiDB 的存算分离先落在分析侧再扩到在线 KV。 TiFlash disaggregated v7.0.0 实验性引入、v7.4.0 GA(Write Node + Compute Node + S3，默认关闭；classic 下 compute 节点默认由 PD 发现，use-autoscaler 为独立开关、外部 AutoScaler 属 Cloud 路径；路由侧 com
3. OceanBase 自 4.3.5 LTS 正式引入 shared-storage(Bacchus)、4.4.x 完善，4.2.5 仍 shared-nothing。 _ss_* 缓存参数自 4.3.5_CE 已出现、4.4.x 扩充至 20+(跨版本源码对照，版本分界为 4.2.5→4.3.5);Bacchus 用三级缓存(memory micro / 本地持久 / 分布式

### 第 8 章 性能
1. 分布式数据库性能必须拆成 OLTP / AP / HTAP / 弹性 / 故障恢复五类正交维度分别讨论，不能排序成单一「谁更快」；两套系统各自把优势压在不同路径、把代价留在不同位置，且一种 workload 下的优势不能自动外推到另一种。
2. OceanBase 金融级 OLTP 短事务优势的结构性来源是单 LS 事务免 GTS RPC + 单机提交短路 + GTS 本地缓存/在途去重，并依托 OBServer 一体化内核协同（Tenant/Unit/Partition/Tablet/LS/PALF/MemTable·SSTable/merge 联动）；代价是 major merge 抖动、多租户竞争与 GTS/Ro
3. TiDB 的优势在云原生弹性、组件化 SQL 层扩展与 HTAP（TiFlash Raft Learner + MPP），OLTP 靠客户端 TSO 批量化摊薄时序开销；瓶颈常落在热点 Region、Raft apply（池默认 2）、RocksDB compaction/flow control、跨 Region 事务与 PD/TSO 压力（高并发下 PD leader CP

### 第 9 章 分库分表替代
1. TiDB 替代的是「分库分表中间件」：逻辑表无子表，行经 tablecodec 编码为连续 keyspace t{tableID}_r{handle}，由 TiKV 按 256MiB（release-8.5 默认）自动 split Region，AUTO_RANDOM/SHARD_ROW_ID_BITS/PRE_SPLIT_REGIONS 打散热点并铺开冷启动，迁移心智接近「去
2. OceanBase 把「分区表 + 事务 + 本地性 + 多表共置」内核化：Partition→Tablet（默认 128MB）→Log Stream，分区键由 ObPartitionFuncType（HASH/KEY/RANGE/LIST）选择，Table Group（NONE/PARTITION/ADAPTIVE）把相关表分区共置同一 LS 以本地化事务与 JOIN，迁移心
3. 分片/分区键、主键、索引设计仍然关键，只是从「应用层手工分片」变为「内核参数与 DDL 设计」；订单/明细/支付的建模应围绕访问模式选择，TiDB 重在避免连续 key 与二级索引热点，OceanBase 重在让共同访问的表共享 Partition key 与 Table Group 约束；设计错误（自增列做键）照样制造热点。

### 第 10 章 分布式事务模型
1. TiDB 事务模型是 Percolator-like 客户端协调 2PC：primary lock 是事务状态的真相之源，事务「真相」分布式编码进每个 key 的 lock/write 列；2PC 主循环在外部库 client-go，TiDB Server 与 TiKV src/storage/txn/ 只负责委托与生成 MVCC 修改。
2. OceanBase 事务模型是协调者下沉的优化版树形 2PC，以 Log Stream-LS 为原子参与者；状态机 ObTxState 多出 REDO_COMPLETE/PRE_COMMIT/CLEAR 工程态，角色 ROOT/INTERNAL/LEAF 构成树形拓扑，专为 leader transfer 设计（三版本枚举一致）。
3. 两者都让「不跨边界」的事务短路：TiDB 用 1PC（单 Region）、OceanBase 用单 LS 单分支提交；跨边界时都把客户端可见延迟压到约 1 次多数派往返（TiDB async commit prewrite 后返回 / OceanBase prepare ACK 后返回）。

### 第 11 章 MVCC 与读路径
1. TiKV 的 MVCC = Percolator 三 CF(default/lock/write)+ user key 倒序时间戳后缀，短值(≤255B)内联 write CF；快照读靠一次 write CF seek 定位 commit_ts≤ts 的最新提交记录再回查数据，提交决议直接落 write CF；读可见性由 read_ts、write CF 的 commit_ts
2. OceanBase 的 MVCC = MemTable 行级 ObMvccRow/ObMvccTransNode 双向链表(新版本插表头，>500 建索引)+ SSTable 隐藏列 trans_version(列 ID 7)/sql_sequence(列 ID 8)；可见性不内联，而是回查 tx_table 的 state/commit_version，据此支持 ELR 与延
3. 强一致读两系统默认都依赖 Leader(TiKV follower read 仍需 ReadIndex 往返、OceanBase 强读必走 Leader)，只有放宽一致性的路径才下放副本：TiKV stale read 的安全条件不是「副本存在即可读」而是 ts ≤ safe-ts(且 safe-ts 受 resolved-ts 约束),OceanBase 弱一致读经 /*+

### 第 12 章 SQL 优化器与执行引擎
1. TiDB 优化器是「RBO 规则链(optRuleList)→ CBO(physicalOptimize / FindBestTask，System-R / Volcano 风格)」两阶段，task 分 root / cop / mpp 三层，可下推算子经 Coprocessor DAG 下沉到 TiKV、大规模并行靠 TiFlash MPP(Fragment + Exchan
2. TiDB 源码存在 Cascades 相关目录，但开发指南把默认生产路径限定在 core / System-R / Volcano 风格 CBO，不能把源码目录直接写成默认优化器行为。
3. OceanBase 优化器是 CBO(ObOptimizer + 列存专用代价模型，4.3 起自动行 / 列选择)，把对象数据特征与物理分布纳入计划生成，其标志是计划态(ObPhyPlanType 枚举共 5 成员，操作态为 OB_PHY_PLAN_LOCAL / REMOTE / DISTRIBUTED)与优化器层 DistAlgo 分发枚举(broadcast / hash

### 第 13 章 元数据与控制面
1. TiDB 控制面是两段式：PD(独立进程 + 内嵌 etcd,etcd Raft)管 Region/Store 元数据、集群级单一 TSO(GlobalTSOAllocator,maxLogical=1<<18，窗上界持久化到 etcd 防回退，默认窗长 5s 绑定 leader lease)、调度与 keyspace;TiDB DDL owner(etcd 选举)+ sche
2. OceanBase 控制面收敛进 sys tenant:RootService(随 __all_core_table Leader 由 Paxos 选举，失败后另一 OBServer 接管)聚合 schema service、DDL service、Unit/Resource Pool、负载均衡；GTS 按租户级隔离(依赖 tenant internal table Leade
3. 二者时序服务的隔离粒度根本不同：TiDB 是集群级单一 TSO(窗上界持久化到 etcd 防回退),OceanBase 是租户级 GTS(租户 Paxos/LS 维护)；这决定了多租户混部下的时序隔离与跨地域延迟特性差异。

### 第 14 章 DDL 与 Online Schema Change
1. TiDB Online DDL 采用在线异步、多中间状态渐进协议，用 None→DeleteOnly→WriteOnly→WriteReorganization→Public 五状态机(源码共 8 态)加"全集群最多两版本共存"不变式，保证渐进式变更下无孤儿/完整性异常。该协议与 Google F1 在线异步 schema change 思想一致，但官方文档/源码/设计文档均未
2. OceanBase 不用统一状态机，而是按代价分三档：instant(纯元数据，异步 compaction 补全)/ long-running 单表(create index，快照版本协调加 bypass write 补全)/ double-table offline(modify column 等，隐藏表重定义)，枚举级铁证见 ObDDLType;DDL 更围绕 RootSe
3. 两者的回填/补全都绕过正常事务路径直产有序底层结构(TiDB Fast DDL 的 lightning local backend ingest;OceanBase 的 distributed sort 加 bypass write)，思路趋同但落点不同；相关性能数字(TiDB 6.5 vs 6.1 的 10×/8~13×、OceanBase 4.0 vs 匿名 databas

### 第 15 章 全局索引 / 本地索引
1. 本地索引等于索引项与行同分片（写本地原子、读非分区键须扇出）；全局索引等于独立分片的一键定位结构（读省扇出、写跨分片 2PC）。这是一组对称取舍，不存在普适更优。
2. TiDB 二级索引是 SQL-on-KV 编码：行 t{tid}_r{handle}、索引 t{tid}_i{idx}{列值}[+handle]，用 key 第 10 字节区分（'i'/'r'）；回表用索引 value/key 里的 handle 拼行 key 读主表，执行计划表现为 IndexLookup。
3. TiDB 分区表全局索引把 partition id 编入索引项（PartitionIDFlag=126，GlobalIndexVersionV1，release-8.5 实现；V2 仅入 key 尚未实现），key 前缀改用逻辑 tableID；实验态 v8.3.0、GA v8.4.0、含全部分区列 v8.5.0、非唯一列支持 v8.5.4，tidb_enable_globa

### 第 16 章 HTAP / AP 架构
1. 路线分野：TiDB 是组件化行列分离（TiKV 行加 TiFlash 列，Raft Learner 跨引擎复制），隔离边界清晰但副本生命周期与计划选择更复杂；OceanBase 是一体化增强（同引擎、同 PALF，基线列加增量行），运维面更内聚但版本与形态矩阵更复杂。
2. TiFlash 用 Raft Learner 而非同步 Follower，也不是外部异步 ETL:Learner 不进多数派、对 leader 异步，把列存故障与 OLTP 写可用性解耦；读一致性靠读时 ReadIndex 等日志追平加 MVCC 提供 SI，因此必须同时测试复制滞后与读时可见性。
3. TiFlash disaggregated v7.0.0 实验性引入、v7.4.0 GA，拆为 Write Node 加 Compute Node 加 S3，与 classic / coupled 不可混部或原地切换；8.x 讨论 TiFlash 时必须标注是哪种形态。

### 第 17 章 多租户与资源隔离
1. OceanBase 是原生多租户:Tenant→Resource Pool→Unit→Zone→Locality 自底向上拼装，每 user 租户拥有独立内存空间(硬上限)、独立线程池、独立 Log Stream/日志盘，并成对绑定 meta 租户隔离内部元数据(tenant_id 奇偶编码：sys=1、奇=meta、偶=user，资源从 user 租户扣除);Tenant 是
2. OceanBase 资源隔离是多机制叠加:CPU 双轨(cgroup cpu.shares/cfs_quota + 进程内 unit_min_cpu×cpu_quota_concurrency)、内存物理硬隔离、IO 三参数(MIN/MAX/WEIGHT IOPS，语义借鉴 mClock 论文、非源码级移植)、租户内 DBMS_RESOURCE_MANAGER 再切分；4.4
3. TiDB Resource Control 是软隔离:RU(CPU/IO 统一抽象，3ms=1RU)令牌桶(RU_PER_SEC/BURSTABLE)+ TiKV 优先级 + Runaway 规则，辅以 Placement Rules/TiDB Node Group 拓扑能力，实现于共享 TiDB/TiKV 之上，无独立内存/线程/日志盘/租户级元数据，适合共享集群内 work

### 第 18 章 容灾、高可用与多地多活
1. TiDB 副本角色底层只有 Voter 与 Learner 两类（MetaPeerRole 映射证实），Witness 是 Rule 上与 Role 正交的布尔标志（is_witness，priority -1、专用 BatchSwitchWitness 命令，local read 拒绝普通读），按设计是「存 Raft 日志、每次写都投票」的 log-only 机制，与「存数据
2. OceanBase 的 Arbitration 仲裁成员是 PALF 枚举（NORMAL_REPLICA/ARBITRATION_REPLICA，跨 4.2.5 与 4.4.x 持续存在），配独立仲裁服务状态机（自 v4.1 GA，仅 2F+1A/4F+1A，社区版默认不编译）；它「只存日志流元数据、不存用户数据、正常运行时不逐条投日志、仅在半数全功能副本故障时做降级仲裁、不能
3. TiDB DR Auto-Sync 对外正式枚举三态（sync/async/sync-recover，另有内部过渡态 async_wait；源码常量为小写下划线形式，commit-pinned 确认），canSync = primaryHasVoter && drHasVoter、多数派谓词语义为 totalUpVoter*2 > totalVoter；源码注释明确「多数派真正

### 第 19 章 运维 / 观测 / 调优体系
1. TiDB 的可观测性是外置组件拼装式:Statement Summary(LRU / 按 digest 聚合,release-8.5 源码确认)、Slow Query(默认 300ms 阈值)、Top SQL(CPU 按 SQL 聚合)、Continuous Profiling 在进程内采集，但全集群关联与可视化经 Prometheus + Grafana + Dashboar
2. OceanBase 的可观测性是内核内置 gv$ / v$ 虚拟表式:SQL Audit(GV$OB_SQL_AUDIT=21014,内存环形缓冲 FIFO 淘汰)、SQL Trace、compaction 进度(GV$OB_COMPACTION_PROGRESS=21227)、LS / Tablet / 租户内存视图、ASH 直接暴露内核状态,SQL 即诊断、零外部依赖，视图
3. P99 抖动必须按"指标 → 视图 → 日志 → 处置"推进，并用关联键(TiDB 的 SQL digest / Region / Store、OceanBase 的 trace id / Tenant / LS / Tablet)把多面板与日志拼成同一事件;链路因架构而异:TiDB 经 write stall / 热点 Region / TSO wait 入手(TiKV-Fa

### 第 20 章 兼容性模型
1. 兼容性必须拆成五层(协议/语法/执行/语义/发布)而非一个布尔标签；最常见的误判是把"有语法"当成"能执行"，把"内核有语义分支"当成"该版本能用"——两者都需按执行器接入与发布归属分别核验。兼容性的本质是 parser、resolver、catalog、optimizer、事务、锁、DDL、错误码与工具链共同给出的一份行为合同，而不是一张 SQL parser 检查表。
2. TiDB 是单方言深度兼容(窄而真)：仅 MySQL 协议+方言(论文逐字 "supports the MySQL protocol")，默认 sql_mode 七项与 MySQL 5.7 对齐；外键 v8.5.0 GA 且进入 DDL 状态机与内核校验链(ddl/foreign_key.go，默认开)；但存储过程仅解析不可执行(executor 无引用)、触发器连 CREAT
3. TiDB 的 REPEATABLE-READ 是以 Snapshot Isolation 对外兼容 MySQL 名称，不能把隔离级别变量名当作 InnoDB 行为等价证明；AUTO_INCREMENT 在分布式缓存下可能有间隙，AUTO_ID_CACHE 1 更接近 MySQL 但不承诺绝对无洞。

### 第 21 章 安全 / 权限 / 审计 / 加密
1. 权限模型：TiDB 是「MySQL 静态位 + SQL Roles + 动态权限 + SEM」的单方言深度模型，OceanBase 是「租户级 + MySQL/Oracle 双模」的分层模型。TiDB 的动态权限可插件扩展(默认条目数随版本变化：8.5.x 16 项、7.5.x 15 项，差 RESOURCE_GROUP_USER)、SEM 使 SUPER 不可越过 RESTR
2. 审计是本章产品形态差异最大的能力。TiDB 开源内核仅提供审计 SPI(AuditManifest，无本体)，审计落地依赖企业版插件或 Cloud Dedicated(写客户云存储),Starter 不支持、Essential 为 Beta;OceanBase 内核内置两套独立审计——GV$OB_SQL_AUDIT(内存诊断、保留短)与 Security Audit(AUDIT
3. 加密职责放置点不同但密钥体系同构。TiDB 静态加密在 TiKV(信封加密，master→data key，默认 7 天轮换，File/AWS/Azure/GCP KMS),TiDB Server 只做 TLS + 溢写文件，BR client-side 备份加密 8.5.0 GA;OceanBase TDE 在 OBServer 内一体化(两级 tenant master k

### 第 22 章 云原生与 Kubernetes
1. 两家都用 Operator 模式把内核编排翻译成 K8s CR，但 CR 模型粒度不同：TiDB v1 集中在 TidbCluster 单 CR(v2 已 GA，拆为 Cluster / ComponentGroup / Instance 三层并直管 Pod),OceanBase 把 OBCluster / OBZone / OBServer / OBTenant 等作为多个一
2. StatefulSet 对分布式数据库有结构性局限(卷模板不可改、强制有序更新引发 Leader 重复调度、Pod 配置须相同、无 raft 成员语义)，其有序更新不能替代数据库级滚动升级；TiDB Operator v2 与 ob-operator 都选择直管 Pod 绕开它，而 TiDB v1.6(锁定基准生产主体)仍依赖 StatefulSet,Operator 必须结合
3. TiDB 的云原生优势来自组件化与无状态 SQL 层(弹性扩容免数据迁移),OceanBase 因一体化内核所有节点带状态、计算弹性更弱，其 K8s 适配难点在 Tenant / Unit / Resource Pool / Zone / Locality 与 Pod / PVC / topology 的双向映射，而非容器启动本身；这正是其 shared-storage / B

### 第 23 章 存储池化 / Shared-storage
1. TiDB 与 OceanBase 的经典形态(TiKV release-8.5 / OceanBase 4.2.5)都不是物理 shared-storage，而是 shared-nothing 本地盘：TiKV 的对象存储能力仅服务备份 / 导入(源码反查证),OBServer 本地承载 Tenant / Tablet / LS / PALF 与存储引擎，所谓“存储池”是 PD
2. 真正的在线 shared-storage 在两家落点不同:TiDB 先在分析侧落地(TiFlash disaggregated,v7.0 实验、v7.4 GA,Write / Compute 拆分 + S3)，行存 OLTP 侧靠 TiDB X(对象存储为真相源、LSM forest、远程 compaction；有官方 Cloud 架构文档明文，但属 TiDB Cloud 专有
3. Bacchus 的三大工程支柱:(a)服务化 PALF 共享日志(LogServer 托管、多分区共享 log stream、CLog 先写本地缓存再归档、SSLOG LS 1001 承载共享元数据);(b)SSWriter 单写者租约破解对象存储无互斥；(c)三级缓存(memory / local-ARC / 分布式 BlockServer)+ 多种预热 + compacti

### 第 24 章 迁移传统分库分表系统的方法论
1. 迁移分库分表系统不是数据搬迁，而是主键/分区键/索引重设计 + 双写灰度 + 一致性校验 + 可回滚切流的系统工程，迁移工具(TiDB DM、OceanBase OMS)只是其中一环；其目标不是消灭复杂度，而是把路由、事务、索引与资源隔离从应用层迁入分布式数据库内核与工具链，并重新定义验证边界。
2. TiDB 主键防热点三件套 AUTO_RANDOM(默认 shard bits=5、max=15,release-8.5 源码常量)/SHARD_ROW_ID_BITS/PRE_SPLIT_REGIONS(预切数=2^(PRE_SPLIT_REGIONS)、须 ≤ SHARD_ROW_ID_BITS 或 AUTO_RANDOM，以官方 Split Region 文档为准)三者协
3. OceanBase 迁移核心是把分片键设计为分区键、用 Table Group(NONE/PARTITION/ADAPTIVE，默认 ADAPTIVE)声明式共置相关表以本地化跨表事务；全局索引(ObIndexType GLOBAL=3/4)解决非分片键查询但把写入升级为分布式事务，代价转移进内核而非消失。

### 第 25 章 Benchmark 与测试设计
1. benchmark 的价值在"数字 + 五前提（workload/硬件/版本/调优/利益相关方）+ 可观测归因"三元组，而非孤立标量；公平 benchmark 的最小单位不是数据库产品名，而是 workload、版本、硬件、部署形态、参数、数据分布、故障窗口与观测口径的组合，只报告总 TPS 几乎无价值。
2. 不能只看总 TPS：必须分别观测 P50/P95/P99/P999 长尾、事务冲突率、跨分片比例、compaction/merge 抖动、apply lag、热点分布、GC/safepoint 与控制面延迟——这些只在长尾与故障注入下暴露。TiDB 按 TiDB Server、PD/TSO、TiKV Region / Raft Group、TiFlash、GC/compacti
3. 压测工具（go-tpc/obtpcc/ob-sysbench）是外部独立仓库、参数须查证；但被测系统侧观测对象可 commit-pin：TiDB 侧 GC 默认 10min、load-based split QPS 阈值 3000/大 Region 7000（连续 10s）、4 个 raftstore apply/commit/append histogram、pd-tso-b

---
