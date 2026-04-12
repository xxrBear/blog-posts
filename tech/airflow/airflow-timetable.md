---
title: "Apache Airflow 自定义时间表"
date: 2026-04-01T15:43:16+08:00
lastmod: 2026-04-01T15:43:16+08:00
author: 熊大如如
tags: # 标签
  - "airflow"
description: ""
weight:
slug: ""
summary: ""
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
Airflow 可以自定义任务的执行时间，为了那些在时间表中难以自定义的时间，例如农历放假时间等

```python
class AfterWorkdayTimetable(Timetable):
    def __init__(self, schedule_at: Time):
        self._schedule_at = schedule_at

    @lru_cache(maxsize=512)
    def _is_holiday(self, d) -> bool:
        return fetch_date_info(d).get("is_holiday", False)

    def get_next_workday(self, d, incr=1):
        next_start = d
        while self._is_holiday(next_start.date()):
            next_start = next_start.add(days=incr)
        return next_start

    def next_dagrun_info(self, *, last_automated_data_interval, restriction):
        E8 = Timezone("Asia/Shanghai")

        # 没有开始日期返回 None
        if restriction.earliest is None:
            return None
        earliest = restriction.earliest.astimezone(E8)

        if last_automated_data_interval is not None:
            last_start = last_automated_data_interval.start.astimezone(E8)
            next_start = DateTime.combine(
                (last_start + timedelta(days=1)).date(), Time.min, tzinfo=E8
            )
        elif not restriction.catchup:
            next_start = max(
                earliest, DateTime.combine(Date.today(), Time.min, tzinfo=E8)
            )
        elif earliest.time() != Time.min:
            next_start = DateTime.combine(earliest.date(), Time.min, tzinfo=E8)
        else:
            next_start = earliest  # 不再多加一天

        next_start = self.get_next_workday(next_start)

        if restriction.latest is not None and next_start > restriction.latest:
            return None

        # 用 .set() 安全设置时间
        run_after = next_start.set(
            hour=self._schedule_at.hour,
            minute=self._schedule_at.minute,
            second=0,
            microsecond=0,
        )

        return DagRunInfo(
            data_interval=DataInterval(start=next_start, end=next_start),
            run_after=run_after,
        )
```

## 核心参数

### last_automated_data_interval
这是该 DAG 上一次非手动触发运行的数据区间，如果是首次调度则为 `None`

`DataInterval` 只有两个属性：

| 属性     | 类型                | 说明                   |
| -------- | ------------------- | ---------------------- |
| `.start` | `pendulum.DateTime` | 上次运行数据区间的起点 |
| `.end`   | `pendulum.DateTime` | 上次运行数据区间的终点 |


两个值都保证是含时区信息

### restriction
封装了 DAG 和 Task 上 `start_date`、`end_date`、`catchup` 参数综合计算后的调度约束

| 属性        | 类型              | 说明         | 来源                       |
| ----------- | ----------------- | ------------ | -------------------------- |
| `.earliest` | pendulum.DateTime | None         | 允许调度的最早时刻，含边界 | DAG/Task 的 `start_date` |
| `.latest`   | pendulum.DateTime | None         | 允许调度的最晚时刻，含边界 | DAG/Task 的 `end_date`   |
| `.catchup`  | bool              | 是否补跑历史 | DAG 的 catchup 参数        |


`earliest` 和 `latest` 如果不为 `None`，都是闭区间，即恰好等于边界时刻也允许调度。Airflow 创建的 `TimeRestriction` 实例保证这两个值都是时区时间

一个容易忽略的点：`earliest` 和 `latest` 约束的是 DAG Run 的逻辑日期（即数据区间的起点），而不是任务实际执行的时刻（通常在区间结束之后）

### DagRunInfo

`DagRunInfo` 是一个 NamedTuple，有两个字段：`run_after` 是 Scheduler 最早创建并调度该次 Run 的时刻（必须是 aware datetime）；`data_interval` 是该次 Run 要处理的数据区间

```python
# 两种构造方式
DagRunInfo(
    run_after=pendulum.datetime(...),
    data_interval=DataInterval(start=..., end=...),
)

# 快捷方式：run_after 自动等于 end
DagRunInfo.interval(start=start, end=end)
```

由于通常希望在数据区间结束后立即调度，`end` 和 `run_after` 一般是同一个时刻，`DagRunInfo.interval()` 这个快捷方法正是保证了这一点
