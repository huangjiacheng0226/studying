# 学习笔记

这是一个记录个人学习过程的仓库，按专题保存各个技术方向的学习笔记。内容以 Java 后端为主，同时覆盖 Python、数据库、缓存、Linux 等方向，后续会随着学习进度持续补充和修订。

笔记全部使用 Markdown 编写，重点放在「把概念讲清楚」上：每个新概念先给定义，再讲原理和用法，需要对比的地方配表格，涉及流程的地方配 Mermaid 流程图。

## 专题一览

| 目录 | 内容 | 篇数 | 状态 |
| --- | --- | :---: | --- |
| [java](./java) | Java 基础语法、面向对象、常用 API、集合与 Lambda、异常与 IO 流、多线程与网络编程、反射与动态代理 | 10 | 已整理 |
| [javaweb](./javaweb) | HTML/CSS、JavaScript/Vue/Ajax、Maven、HTTP、Spring Boot、数据库、MyBatis、员工管理实战及项目部署 | 21 | 已整理 |
| [mysql](./mysql) | MySQL 概述、SQL、事务、索引、性能优化、InnoDB、集群进阶，以及分布式 ID、分布式事务、备份恢复与设计规范 | 24 | 已整理 |
| [redis](./redis) | Redis 基础与数据类型、短信登录与缓存、优惠券秒杀与分布式锁、消息队列、Feed 流、GEO 统计、持久化与集群、底层结构与多级缓存 | 16 | 已整理 |
| [linux](./linux) | Linux 基础概念、常用命令、用户和权限、网络与端口、进程管理、环境变量、压缩解压、Shell 脚本、磁盘、SSH、排障与部署 | 18 | 已整理 |
| [python](./python) | Python 核心语法、面向对象、模块，以及 AI 应用、网络爬虫、数据分析、Web 开发四个实战项目 | 13 | 已整理 |
| `spring` | Spring、Spring MVC、Spring Boot 和常用开发模式 | — | 计划中 |

每个专题目录下都有一份 `README.md`，写了该专题的内容结构、篇目清单、笔记特点和阅读建议。

## 目录结构

```text
studying/
├── README.md
├── java/
├── javaweb/
├── mysql/
├── redis/
├── linux/
└── python/
```

各专题独立管理，目录名称使用小写英文，便于在命令行和开发工具中使用。专题目录中的笔记按 `01-`、`02-`、`03-` 的顺序编号，部分目录会有一篇 `00-` 开头的目录或总纲。

## 推荐学习顺序

Java 后端方向：

1. [Java 基础](./java)
2. [Linux 基础](./linux)
3. [MySQL 基础](./mysql)
4. [JavaWeb](./javaweb)
5. [Redis](./redis)
6. Spring 与 Spring Boot

Python + AI 方向：

1. Python 核心语法（[`python`](./python) 目录中的 `01` 至 `07` 篇）
2. 按兴趣选一个实战项目练手：AI 应用、网络爬虫、数据分析、Web 开发（`08` 至 `11` 篇）
3. Python 进阶语法与 Web 框架（FastAPI），可配合下方「配套学习资料」一起学

学习每个专题时，建议按照“概念理解 → 命令或 API → 示例代码 → 综合练习”的顺序整理内容。

## 各专题导读

- Java：按入门顺序拆分为一份总纲和九篇分类文档，从基础语法一路到多线程、IO 流与反射，直接进入 [`java`](./java) 目录阅读即可。
- JavaWeb：按课程顺序拆分为二十一篇，从 HTML/CSS、JavaScript 到后端框架和项目部署，直接进入 [`javaweb`](./javaweb) 目录阅读即可。
- MySQL：按入门、核心、进阶、分布式与运维四段拆分，含一篇目录和二十三篇分类文档，直接进入 [`mysql`](./mysql) 目录阅读即可。
- Redis：分为三部分。`01` 至 `03` 篇是基础入门、数据类型与常用命令、应用实践总结；`04` 至 `09` 篇是分布式缓存实战，用短信登录、优惠券秒杀、探店关注等场景串起缓存、分布式锁、消息队列、Feed 流和 GEO 统计；`10` 至 `16` 篇讲持久化、高可用与集群、过期与内存淘汰、最佳实践、底层数据结构、网络模型和多级缓存。直接进入 [`redis`](./redis) 目录阅读即可。
- Linux：共十八篇，从操作系统概念、虚拟机与远程连接讲起，覆盖常用命令、用户权限、网络配置、进程管理、压缩解压，再延伸到 Shell 脚本与定时任务、文本处理、磁盘管理、SSH 与文件同步、日志排障和部署实战。直接进入 [`linux`](./linux) 目录阅读即可。
- Python：按课程顺序拆分为十三篇，从零基础语法讲到四个实战项目，直接进入 [`python`](./python) 目录阅读即可。这十三篇面向完全零基础的同学编写：每篇都是先给出概念定义，再做深入讲解，配对比表格做横向和纵向比较，涉及流程的地方配 Mermaid 流程图并附文字说明，全程用大白话和生活化的例子来讲。其中第 `02` 到 `07` 篇是 Python 核心语法，是后面所有实战项目的基础，建议手敲代码而不是直接复制。

## 配套学习资料

以下资料通过百度网盘分享，与本仓库的笔记配套使用。提取码已随链接给出，链接失效时可用提取码手动输入。

| 资料 | 涵盖内容 | 网盘链接 | 提取码 |
| --- | --- | :---: | :---: |
| JavaWeb + AI | JavaWeb 与 AI 方向的配套课程资料 | [百度网盘](https://pan.baidu.com/s/1s2q3-VPiAfZE-FcrxSouLQ?pwd=txfy) | `txfy` |
| Python + AI 从入门到大神一套通关 | Python 基础到 AI 应用的完整课程 | [百度网盘](https://pan.baidu.com/s/1gLxp1NSGGSPztmNDUl4IaA?pwd=pkbf) | `pkbf` |
| Python Web 开发：FastAPI 从入门到实战 | FastAPI 后端开发，对应 Python Web 方向 | [百度网盘](https://pan.baidu.com/s/1NZV46gwzE6QC31BrVHW1Rw?pwd=kg5f) | `kg5f` |
| Python 进阶（大模型语言进阶） | Python 高级语法与大模型相关语言基础 | [百度网盘](https://pan.baidu.com/s/1y38y7QWsjFNJP1mcsGxEpg?pwd=rz9r) | `rz9r` |
| Redis | Redis 全套课程资料 | [百度网盘](https://pan.baidu.com/s/1A9lupK-A9JuSGyouZ1zISQ?pwd=bsxq) | `bsxq` |

## 笔记规范

- 每篇笔记使用清晰的主标题和分级小标题。
- 新概念先给出简明解释，再提供可运行的示例。
- 多个概念之间存在差异时，使用表格进行比较。
- 代码示例只保留必要注释，重点说明关键步骤和使用限制。
- 涉及流程、执行顺序的内容使用 Mermaid 流程图，并在图下附文字说明。
- 每篇笔记末尾增加知识点总结，记录需要复习的内容。
- 文件名使用两位数字编号，例如 `01-基础概念.md`。

## 学习进度

- [x] Java 基础
- [x] JavaWeb
- [x] MySQL
- [x] Redis
- [x] Linux 基础
- [x] Python 基础与实战
- [ ] Spring / Spring Boot
- [ ] Python 进阶与 FastAPI

## 使用方式

可以直接在线阅读 Markdown 文件，也可以克隆仓库到本地：

```bash
git clone https://github.com/huangjiacheng0226/studying.git
cd studying
```

仓库中的流程图使用 Mermaid 语法编写，GitHub 和 Typora 可直接渲染；VS Code 需要安装 `Markdown Preview Mermaid Support` 插件。如果使用的工具不支持渲染，每张图下方都有文字版流程说明，不影响理解。

本仓库会随着学习过程持续补充和修订。
