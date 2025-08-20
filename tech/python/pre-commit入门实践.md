---
title: "pre-commit 入门实践"
date: 2025-02-14T08:30:59+08:00
lastmod: 2025-02-14T08:30:59+08:00
author: ["熊大如如"]
tags: # 标签
  - "python"
description: ""
weight:
slug: ""
summary: "python项目 pre-commit 实践"
draft: false # 是否为草稿
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
  image: "https://cdn.jsdelivr.net/gh/xxrBear/image/icons8-python-500.png"
---

## 简介

`pre-commit` 预提交是 `git` 的一个钩子脚本，它可以在我们 `git commit` 时识别简单问题。当我们每次提交时会自动运行钩子脚本，以自动指出代码中的问题，如缩进异常、缺少分号、尾随空格和调试语句。在代码审查前指出这些问题，这允许代码审查者专注于提交的代码本身，而不会浪费时间在琐碎的查看代码风格是否规范上。

`git` 自带了一个 `pre-commit` 钩子脚本，但是本文讨论的 `pre-commit` 是 python 开发的一个第三方库，是一个用于管理代码提交前自动检查（Git Hook）的工具框架，专为开发者设计，旨在通过自动化检查确保代码质量、统一团队规范。

`pre-commit` 会在你提交代码之前运行你指定的检查。如果检查发现任何错误，它会阻止你提交代码。你需要修复错误后才能提交代码。

## 安装与使用

### 下载 `pre-commit`

```python
pip install pre-commit
```

### 安装 git 钩子

- 它会在你的 Git 仓库中自动创建或更新 `.git/hooks/pre-commit` 文件

```shell
pre-commit install
```

### 添加 `.pre-commit-config.yaml`文件

```shell
repos:
  - repo: https://github.com/PyCQA/flake8
    rev: 6.1.0
    hooks:
      - id: flake8
  - repo: https://github.com/PyCQA/isort
    rev: 5.12.0
    hooks:
      - id: isort
  - repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
      - id: black
```

- repo: 这是要使用的 Git 仓库 URL。`pre-commit` 会从这些仓库中下载相应的工具或脚本
- rev: 这里指定了你要使用的工具的版本，即 Git 标签
- hooks: 在每个仓库配置下，你可以定义一个或多个钩子（hooks）。这些钩子会执行相应的工具
- id: 每个钩子的唯一标识符。这里的钩子 `id` 分别是 `flake8`、`isort` 和 `black`，这表示每次提交时，`pre-commit` 会运行相应的工具
- flake8: 检查 python 代码的风格、潜在错误等
- isort: 会整理 python 代码中的导入顺序
- black: 会重新格式化你的代码，使其符合标准格式

### <font style="color:rgb(64, 64, 64);">首次全量检查</font>

```shell
pre-commit run --all-files
```

- 这将对项目代码运行上述脚本

## 实践

### 创建一个 python 模块

```shell
mkdir learn-pre-commit

vim main.py
```

### 复制 main.py 内容

```shell
def helloWorld():
  print("Hello World!")


if __name__ == "__main__":
    helloWorld()
```

我们可以看到，这个 py 文件的代码是不符合 Python PEP8 规范的

- 缩进错误，只有两个空格
- 函数命令风格错误

### 提前检查

```shell
pre-commit run --all-files
```

### 结果

```shell
(.venv) @bear ➜ learn-pre-commit git(master) pre-commit run --all-files
flake8...................................................................Failed
- hook id: flake8
- exit code: 1

main.py:2:3: E111 indentation is not a multiple of 4

isort....................................................................Passed
black....................................................................Failed
- hook id: black
- files were modified by this hook

reformatted main.py

All done! \u2728 \U0001f370 \u2728
1 file reformatted.
```

提示：flake8 失败，black 失败

### 查看 main.py 文件

```shell
def helloWorld():
    print("Hello World!")


if __name__ == "__main__":
    helloWorld()
```

我们看到缩进已经正常了，这是 `black`的功劳，但是函数风格依旧没有变，我们需要手动修改如下

```shell
def hello_world():
    print("Hello World!")


if __name__ == "__main__":
    hello_world()
```

### 再次使用 pre-commit

上面我们直接运行了钩子脚本，其实在我们 `commit`的是否，脚本会自动运行，现在让我们使用这种方式

```shell
git add .

git commit -m '修改 main.py 文件'
```

### 再次查看结果

```shell
(.venv) @bear ➜ learn-pre-commit git(master) git add .
(.venv) @bear ➜ learn-pre-commit git(master) git ci -m "修改 main.py 文件"
flake8...................................................................Passed
isort....................................................................Passed
black....................................................................Passed
[master 3afec46] 修改 main.py 文件
 1 file changed, 2 insertions(+), 2 deletions(-)
```

你会看到，已符合 PEP8 风格，`pre-commit`脚本全部通过

## 总结

`pre-commit` 是一个非常有用的工具，可以帮助你的项目更加自动化和规范化，也省去了因为代码风格不同的扯皮 😅

思考一下，如何让开发组的其他同事一起用 `pre-commit`呢？

这就是本期分享，如有不足，请指正~
