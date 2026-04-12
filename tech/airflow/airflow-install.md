---
title: "Apache Airflow 安装与调试"
date: 2026-01-26T16:38:02+08:00
lastmod: 2026-01-26T16:38:02+08:00
author: 熊大如如
tags: # 标签
  - "airflow"
description: ""
weight:
slug: ""
summary: "Apache Airflow 的安装与调试"
draft: false # 是否为草稿
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
    image: "https://raw.githubusercontent.com/xxrBear/image/master/blog/airflow-removebg-preview.png"
---

## 概述

本篇章我们介绍一下如何在源码和 Docker 两种方式下安装 Airflow3.0.6 环境，Airflow 不支持在 windows 上的部署，所以在这里我们使用的是 Windows 上的 Linux 子系统。

至于说为什么不使用 Airflow 最近版本 3.1，这是因为博主在真实使用中踩到了最新版本的 bug，包括但不限于 fab 验证出错，数据库连接异常。

## 源码安装

### 系统说明
- 操作系统：WSL Ubuntu22.04
- Airflow 版本：Airflow3.0.6
- Python 版本：3.10
- 数据库：sqlite

### 更新系统组件

```shell
sudo apt-get update
sudo apt-get install -y build-essential libssl-dev libffi-dev python3-dev libsqlite3-dev
```

### 项目准备

```shell
# 安装 airflow
mkdir ~/airflow-repo
cd ~/airflow-repo

uv venv .venv -p 3.10
uv init
uv add apache-airflow==3.0.6
```

这将为你初始化一个虚拟环境，并安装`Apache Airflow3.0.6`我们可以测试一下是否安装成功了：

```shell
airflow verison  # 3.0.6
```

### 启动项目

现在我们已经有了`airflow`项目的环境，在我们启动`airflow`之前我们还需要设置 `airflow`的家目录，默认会在家目录下生成日志文件和配置文件等重要文件夹

```shell
export AIRFLOW_HOME=~/airflow-repo
```

`airflow`提供了一个命令行命令`airflow standalone`这将启动 `airflow` 所需的所有组件

```shell
airflow standalone
```

这将自动在 `AIRFLOW_HOME` 目录下生成 `sqlite` 数据库文件并迁移数据表和生成 `airflow.cfg` 配置文件，此时，你可以访问 `localhost:8080` 路径来访问`Airflow` 了。

默认的用户是`admin`默认密码在`~/airflow-repo/simple_auth_manager_passwords.json`文件中或者你可以在输出日志中找到账户和密码。

## Docker 安装

我们前面介绍了怎么源码安装 `airflow`接下来让我们介绍一下如何使用 docker 安装 airflow 服务

```sql
# 拉取镜像
docker pull apache/airflow:3.0.6-python3.10
```

启动所有`airflow`组件

```shell
docker run -d --name airflow \
  -p 8080:8080 \
  apache/airflow:3.0.6-python3.10 \
  bash -c "\
        airflow standalone"
```

这样你就可以访问 `localhost:8080`来访问 airflow 了

- 单独启动各组件

如果你不想一次性启动所有的组件，你可以复制出配置文件 `airflow.cfg`然后添加必须的文件夹挂载到容器中，注意，你需要将 `logs``dags``plugins`等文件夹添加可执行权限，测试建议 `777`一劳永逸。

```shell
docker run -d --name airflow \
  -p 8080:8080 \
  -v "$PARENT_DIR/logs:/opt/airflow/logs" \
  -v "$PARENT_DIR/dags:/opt/airflow/dags" \
  -v "$PARENT_DIR/plugins:/opt/airflow/plugins" \
  -v "$PARENT_DIR/airflow.cfg:/opt/airflow/airflow.cfg" \
  apache/airflow:3.0.6-python3.10 \
  bash -c "\
    airflow scheduler & \
    airflow triggerer & \
    airflow dag-processor & \
    airflow api-server"
```

你可以在 `dags`文件夹中添加你的定时任务，然后调式测试程序是否正常工作。

## 调试
### 断点调试
虽然我们还没有介绍过 Airflow 的任务，但是我们启动了`airflow`后，我们肯定想断点调试任务，我们可以这样：

> 或者这里可以调整到别的地方？
>

```python
import datetime

import pendulum
from airflow.providers.standard.operators.python import PythonOperator

from airflow import DAG


def test_dag(**context):
    print(f"start_date:{datetime.datetime.now()}")
    print(f"start_date:{datetime.datetime.now()}")


with DAG(
    dag_id="dag_test",
    description="A test process DAG",
    schedule=None,
    start_date=pendulum.now(),
    tags=["dag_test"],
) as dag:
    t1 = PythonOperator(task_id="dag_test", python_callable=test_dag)

    t1


if __name__ == "__main__":
    dag.test()
```

使用`VSCode`或者`Pycharm`直接`debug`这个文件，这里最重要的就是 `dag.test()`方法
