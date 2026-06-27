# TiDB 与 OceanBase 分布式数据库底层架构对比研究(融合学习版)

> 从分片、共识、存储到事务、HTAP、存算分离与云原生的**源码级**横向剖析。

📖 **在线阅读(GitHub Pages):** https://zhangliangchen.github.io/Distributed-Database-Study-Notes/

## 简介

这是一份面向工程师的深度学习材料,对 TiDB 与 OceanBase 两套分布式数据库的底层架构做逐主题、源码级的横向对比。全文 **25 章 + 7 附录**,约 **25 万字**,含 **79 张图、91 张表**。

- **正确性优先**:关键事实尽量锚定到 commit-pinned 的源码、官方文档、论文或 TPC 官方结果;证据不足处采用保守、可证伪的表达,未核验细节标注「需进一步查证」,不写成事实。
- **图表**:每章含正常路径图、故障路径图,以及结构 / 层级 / 位域 / 流程等补充图,均为白底纵向 SVG。
- **冲突裁决**:对两份独立研究稿在融合时出现的高风险事实,逐项做了第三方核验与裁决(见附录 F)。

## 版本基准

| 组件 | 锁定版本 |
|---|---|
| TiDB | 8.5.x LTS(对照 7.5.x);TiKV / PD `release-8.5` |
| TiFlash 存算分离 | v7.0.0 实验性引入、v7.4.0 GA |
| OceanBase | TP 4.2.5_CE / AP 4.3.5 / TP+AP 融合 4.4.x |

完整版本基准见 [附录 B](FULL_PAPER_FUSED_V2.md#附录-b-版本基准表)。

## 章节概览

分片 · 路由 · 共识 · 存储引擎 · KV/LSM 映射 · 无状态 SQL 层 · 存算分离 · 性能 · 分库分表替代 · 分布式事务 · MVCC 与读路径 · SQL 优化器与执行引擎 · 元数据与控制面 · DDL/Online Schema Change · 全局/本地索引 · HTAP/AP 架构 · 多租户与资源隔离 · 容灾高可用与多地多活 · 运维/观测/调优 · 兼容性模型 · 安全/权限/审计/加密 · 云原生与 Kubernetes · 存储池化/Shared-storage · 迁移方法论 · Benchmark 与测试设计

## 文件说明

| 文件 | 用途 |
|---|---|
| [`FULL_PAPER_FUSED_V2.md`](FULL_PAPER_FUSED_V2.md) | **主交付稿**,图片以 SVG 嵌入(`figures/`),GitHub Pages 渲染用 |
| [`FULL_PAPER_FUSED_V2.mermaid.md`](FULL_PAPER_FUSED_V2.mermaid.md) | 保留 ```mermaid``` 源码块(可编辑;GitHub 可原生渲染 Mermaid) |
| `FULL_PAPER_FUSED_V2.obsidian.md` | Obsidian 维链版(`![[fN_k.svg]]`) |
| `figures/` | 79 张白底纵向 SVG 图 |
| `index.html` | Pages 站点入口(客户端渲染全文 + 侧栏目录) |

## 阅读方式

- **在线**:打开上方 GitHub Pages 链接,左侧目录可跳转任意章节。
- **GitHub 内**:直接点 [`FULL_PAPER_FUSED_V2.md`](FULL_PAPER_FUSED_V2.md) 阅读(图片随仓库一并渲染)。
- **本地**:克隆后用支持 Mermaid 的 Markdown 阅读器打开 `*.mermaid.md`,或在浏览器起一个本地静态服务器打开 `index.html`。

## 启用 GitHub Pages(仓库管理员一次性操作)

Settings → Pages → Build and deployment → Source 选 **Deploy from a branch** → Branch 选 **`main`** / 目录 **`/ (root)`** → Save。约 1 分钟后站点上线于上方链接。

---

*学习笔记,仅供学习与技术交流。文中观点与结论以所引源码 / 官方资料为准;转载请注明出处。*
