---
title: "Apache Airflow 概述"
date: 2026-01-22T15:44:16+08:00
lastmod: 2026-01-22T15:44:16+08:00
author: 熊大如如
tags:
  - "airflow"
summary: "Airflow 概述"
mermaid: true
showToc: false
TocOpen: true
hidemeta: false
disableShare: true 
showbreadcrumbs: true
cover:
    image: "https://raw.githubusercontent.com/xxrBear/image/master/blog/airflow-removebg-preview.png"
---
Apache Airflow 诞生于 Airbnb，最初是为了解决公司内部日益臃肿、难以维护的 Cron 定时任务和复杂的 ETL（抽取、转换、加载）流程。

经过十余年的深耕，Airflow 已从一个内部工具成长为 Apache 基金会的顶级项目，成为全球数据工程、自动化运维和任务调度的行业标准。

下面我们通过几点来对比 Cron 和 Airflow 各自的优劣

## 内置集成与自动重试
在 Cron 模式下，开发者需要自己写代码连接数据库、处理超时、封装日志。而 Airflow 及其生态提供了开箱即用的强大支持：

**全栈连接器:**

内置了对对象存储（S3, OSS）、主流数据库（PostgreSQL, MySQL, Snowflake）、基础设施（Docker, Kubernetes）及消息系统（Kafka, RabbitMQ）的官方算子

**Airflow 提供了极为精细的失败处理机制：**

- 自动重试：支持配置重试次数、间隔时间，以及防止系统过载的指数退避策略
- 超时控制：强制限制任务运行时间，防止僵尸进程占用资源
- 异常回调：任务失败时，自动触发邮件、钉钉、Slack 等告警通知

## 可观测性与监控
这解决了 Cron 最致命的短板之一就是任务失败不可观察。Airflow 提供了基于 React 开发的现代化 Web 界面：

- 全局看板：一眼识别哪些任务正在运行、哪些失败、哪些在排队
- 依赖可视化：通过有向无环图直观展示任务的前后调用关系，快速定位阻塞整个流程的
- 在线日志查看：无需登录服务器，直接在浏览器中调取每个任务的详细运行日志，排查效率大幅度提升

## 依赖管理与调度策略
在 Cron 中，实现任务 B 必须在任务 A 成功后运行非常困难。Airflow 原生支持任务级依赖：

```python
# 极其简洁的依赖表达
task_A >> task_B >> task_C
```

此外，它支持灵活的触发规则：

- `all_success`：仅在所有上游成功时执行
- `one_failed`：只要有一个上游失败即触发
- `all_done`：无论结果如何，只要跑完就执行

## 数据回填与上下文
这是 Airflow 的杀手锏之一，Cron 只关心当前时刻，而 Airflow 关心的是数据对应的逻辑时间

- 历史回填：如果昨天的数据逻辑写错了，你只需一条命令，Airflow 就能自动按顺序重跑过去指定时间段的所有任务，并确保数据状态的一致性
- 任务上下文：每个任务执行时都自带环境信息，包括执行日期、DAG ID、运行参数等。你可以利用这些上下文动态拼接表名或存储路径

## 扩展与协作

**横向扩展**

Airflow 自身支持多种执行器。小型任务可用 `LocalExecutor`，大规模集群则可选择 `CeleryExecutor` 或云原生的 `KubernetesExecutor`，实现任务的弹性扩缩容

**安全与审计**

Airflow 支持 RBAC 基于角色的权限访问控制。只需要简单的几步就可以让不同部门的同事只能操作各自的 DAG，且所有手动触发、参数修改行为都有完整的审计日志记录

## 总结

尽管 Airflow 在功能和扩展性上明显强于 cron，但这并不意味着它适用于所有场景。  当任务足够简单、无复杂依赖、对可观测性要求不高时，cron 往往是更合理且成本更低的选择。毕竟工具的价值，在于解决问题，而不是复杂化问题本身。

接下来，我将通过十篇左右的文章，从不同角度系统地介绍 Airflow，带你了解实际使用中的最佳实践。
