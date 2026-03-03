---
title: "Apache Airflow 基本用法"
date: 2026-03-03T17:04:03+08:00
lastmod: 2026-03-03T17:04:03+08:00
author: 熊大如如
tags: # 标签
  - "airflow"
summary: "基本 airflow 使用方法"
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
前面我们以及简单的介绍了 Airflow 的起源和安装，接下来我们介绍一下 Airflow 的简单用法。

DAG 意为有向无环图，它是 Airflow 的核心概念，它指定了一组任务的集合，并且指定任务的运行顺序和运行规则，如果 DAG 中的任务有循环依赖的情况，Airflow 会报错。

Task 是 Airflow 的最小可执行单元，它一般由操作符、传感器、task（在 airflow2 中由@task 装饰器装饰的 python 函数） 组成，可以执行运行顺序。

## 创建 DAG
下面是一个用传统上下文管理器的格式创建的 DAG

```python
from datetime import datetime

from airflow.sdk import DAG, task
from airflow.providers.standard.operators.bash import BashOperator

with DAG(dag_id="demo", start_date=datetime(2022, 1, 1), schedule="0 0 * * *") as dag:
    hello = BashOperator(task_id="hello", bash_command="echo hello")

    @task()
    def airflow():
        print("airflow")

    # Set dependencies between tasks
    hello >> airflow()
```

这里定义了两个任务 `hello`和 `airflow`

+ hello 在命令行中输出 hello
+ airflow 在 python 中打印 airflow

最后的 `>>`符号代表任务的执行顺序，hello 运行完成后运行 airflow

```python
with DAG(dag_id="demo", start_date=datetime(2022, 1, 1), schedule="0 0 * * *") as dag:
```

这个配置代表每天运行一次这个任务。

这些参数中，`dag_id`表示这个 DAG 的唯一标志，`start_date`表示开始时间，`scheduler`表示运行时间规则，他与 Crontab 的 ` 分时日月周 `一致。

这是传统的创建 dag 的方式，接下来介绍两个新的：

```python
dag = DAG(dag_id='standard_constructor_dag', start_date=datetime(2023, 1, 1))

task_a = EmptyOperator(task_id='task_a', dag=dag)
task_b = EmptyOperator(task_id='task_b', dag=dag)

task_a >> task_b
```

这是最原始的方式，不推荐

装饰器方式：

```python
from airflow.decorators import dag, task

@dag(start_date=datetime(2023, 1, 1), schedule='@daily', catchup=False)
def taskflow_dag():
    @task
    def get_data():
        return {"value": 42}

    @task
    def process_data(data):
        print(f"Processing {data['value']}")

    process_data(get_data())

generated_dag = taskflow_dag()
```

这是 airflow2 之后的装饰器方式，是一种比较 `pythonic`的方式，如果你要选一个，我建议选上下文模式或者装饰器模式，选择一个就够了，记多了容易混淆。

## 任务种类
前面介绍过，任务一般由操作符、传感器、装饰函数构成，接下来分别介绍它们是什么

在创建 DAG 中我们写了 `BashOperator``EmptyOperator`等类调用，这种就是操作符，操作符是 Airflow 内置的类， 如 `PythonOperator`（运行 Python 代码）、`BashOperator`（运行脚本）或 `HttpOperator`示例可看上文

传感器则是一类特殊的 Task，它会持续监听，直到某个条件满足，例如，某个文件出现在 S3 上，或者数据库里有了某条数据，示例如下：这个 DAG 定义了传感器任务 `FileSensor`，等待 `ingest.csv`文件出现才运行下面的任务。

```python
from airflow import DAG
from airflow.sensors.filesystem import FileSensor
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

def process_file():
    print("文件已找到，开始处理数据...")

with DAG(
    dag_id='file_sensor_example',
    start_date=datetime(2023, 1, 1),
    schedule='@daily',
    catchup=False
) as dag:

    wait_for_file = FileSensor(
        task_id='wait_for_incoming_file',
        filepath='data/ingest.csv',
        fs_conn_id='fs_default', # 需要在 Airflow Connection 中配置该 ID 的路径
        poke_interval=30,        # 每 30 秒检查一次
        timeout=600,             # 如果 10 分钟还没等到，任务标记为失败
        mode='poke'              # 运行模式：poke (占用 worker) 或 reschedule (释放 worker)
    )

    process_data = PythonOperator(
        task_id='process_data',
        python_callable=process_file
    )

    wait_for_file >> process_data
```

TaskFlow 在 Airflow 2 中，你可以直接在 Python 函数上加装饰器，使其变成一个 Task

## 定义任务执行规则
不同的任务可以执行不同的运行流程，例如，同时有 a,b,c 三个任务我们可以通过 `>>``<<`等操控任务执行流程，例如：

```python
a >> b >> c
```

这行示例代表了：任务 A 执行成功后执行 任务 B，任务 B 执行成功后执行 任务 C。如果其中任意一个任务失败，整个 DAG 将标记为失败，且后续任务不再运行

之所以后续任务不再运行，是因为 Airflow 默认的任务触发规则是 `all_success` 模式，只有当前序任务全部成功时，下游任务才会被触发。如下：

```python
from airflow.utils.trigger_rule import TriggerRule

task_b = PythonOperator(
    task_id='b',
    python_callable=some_function,
    trigger_rule=TriggerRule.ALL_DONE
)
```

如果你希望无论上游任务成功与否，下游任务都照常运行，可以将触发规则修改为 `all_done`

具体触发规则可参考下表：

| 规则 (Trigger Rule) | 行为描述                                                       | 适用场景                                                           |
| ------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------ |
| `all_success`       | (默认) 所有上游任务必须成功。                                  | 线性流水线，每一步都依赖上一步的结果。                             |
| `all_done`          | 只要上游任务全部运行结束（无论成功、失败还是被跳过），就触发。 | 无论任务是否成功，都要执行的清理工作（如关闭集群、删除临时文件）。 |
| `one_success`       | 只要上游有一个任务成功，就立刻触发。                           | 并行执行多个备选方案，只要有一个跑通了就继续。                     |
| `one_failed`        | 只要上游有一个任务失败，就立刻触发。                           | 告警机制：当一组并行任务中出现错误时，触发通知。                   |
| `all_failed`        | 所有上游任务都失败时才触发。                                   | 异常处理逻辑，专门处理全盘崩溃的情况。                             |
| `none_failed`       | 所有上游都没有失败（即：成功或被跳过）。                       | 包含条件分支（Branching）的任务流，被跳过的路径不应阻塞下游。      |
