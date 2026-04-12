---
title: "Apache Airflow 配置项"
date: 2026-03-17T17:16:04+08:00
lastmod: 2026-03-17T17:16:04+08:00
author: 熊大如如

tags: # 标签
  - "airflow"
summary: "Apache Airflow 配置项"
draft: false # 是否为草稿
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
    image: ""  # 文章的图片
---
## 概述
本章节我们学习如何配置 airflow，默认的 airflow 配置不够强大，我们需要自定义配置项

## 数据库配置
默认的 airflow 使用的是 sqlite 数据库，在生产环境中，我们可能需要设置 MySQL 或者 PostgreSQL 数据库，对于数据库修改我们需要修改配置文件 `airflow.conf`

+ MySQL

```python
sql_alchemy_conn = mysql+mysqldb://airflow_user:password@localhost:3306/airflow_db
```

+ PG

```python
sql_alchemy_conn = postgresql+psycopg2://airflow_user:password@localhost:5432/airflow_db
```

## 配置安全模型
默认的 airflow 使用的安全模型功能很简单，而且每次会生产一个不同的密码，这不能用在生产环境，我们需要使用 `fab`安全模型

1. 设置安全模型后端

```python
auth_backends = airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager
```

2. 数据库迁移

```python
airflow db init
```
这将生成 `fab` 需要的数据库表

3. 创建用户

```python
airflow users create \
    --username admin \
    --firstname First \
    --lastname Last \
    --role Admin \
    --email admin@example.com \
    --password your_password
```
创建新的用户

设置之后就可以使用自定义用户登录了

## 优化配置项
以默认的 airflow 配置运行任务性能不好，观察你的系统，分析性能瓶颈再修改以下配置项来优化执行

| 参数                        | 建议值           | 说明                                            |
| --------------------------- | ---------------- | ----------------------------------------------- |
| `parallelism`               | `64 - 256`       | 全局限制：整个 Airflow 环境同时运行的任务实例数 |
| `max_active_tasks_per_dag`  | `16 - 32`        | 防止单个 DAG 占满所有工作槽位                   |
| `max_active_runs_per_dag`   | `3 - 5`          | 限制单个 DAG 同时运行的批次数，防止压垮系统     |
| `parsing_processes`         | `CPU 核心数 - 1` | 调度器解析 DAG 文件的并发进程数，提升解析速度   |
| `scheduler_heartbeat_sec`   | `5`              | 调度器心跳间隔                                  |
| `min_file_process_interval` | `30`             | DAG 文件最小解析间隔                            |
| `dag_dir_list_interval`     | `60`             | DAG 目录扫描频率                                |
