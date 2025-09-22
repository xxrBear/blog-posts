---
title: "xlwings 简易教程"
date: 2025-09-22T13:28:49+08:00
lastmod: 2025-09-22T13:28:49+08:00
author: 熊大如如
tags: # 标签
  - "xlwings"
description: ""
weight:
slug: ""
summary: "xlwings如何操作Excel教程"
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

### 1.xlwings 介绍

#### 启动 Excel 应用并打开工作簿

```python
import xlwings as xw

# 启动 Excel 应用
app = xw.App(visible=True)  # visible=True 会显示 Excel 窗口，设置为 False 时窗口隐藏
# 打开工作簿
wb = app.books.open('path_to_your_workbook.xlsx')

# 对工作簿进行操作
sheet = wb.sheets[0]  # 访问第一个工作表
print(sheet.range('A1').value)  # 读取 A1 单元格的值
```

#### 创建新工作簿

```python
import xlwings as xw

# 启动 Excel 应用
app = xw.App(visible=True)
# 创建新的工作簿
wb = app.books.add()

# 添加数据
sheet = wb.sheets[0]
sheet.range('A1').value = "Hello, World!"

# 保存文件
wb.save('new_workbook.xlsx')

# 关闭工作簿
wb.close()
```

### 2. 读取和写入数据
#### 读取单元格的值
你可以通过 `range` 来访问单元格，读取其值：

```python
sheet = wb.sheets[0]
value = sheet.range('A1').value  # 获取 A1 单元格的值
print(value)
```

#### 写入数据到单元格
你可以直接将数据写入到单元格中：

```python
sheet = wb.sheets[0]
sheet.range('A1').value = "Hello, Excel!"
```

#### 读取一个范围的数据
你可以读取一个单元格区域的数据，返回的是一个二维列表：

```python
values = sheet.range('A1:B3').value  # 获取 A1 到 B3 区域的值
print(values)
```

#### 写入一个范围的数据
同样地，你也可以向一个区域写入数据：

```python
sheet.range('A1:B3').value = [[1, 2], [3, 4], [5, 6]]  # 向 A1:B3 区域写入数据
```

### 3. 工作簿和工作表的管理
#### 获取所有工作表的名称
```python
sheet_names = wb.sheets.names
print(sheet_names)
```

#### 访问特定工作表
你可以通过工作表的名称或索引来访问特定的工作表：

```python
sheet = wb.sheets['Sheet1']  # 通过工作表名称访问
# 或者通过索引访问
sheet = wb.sheets[0]  # 通过索引访问第一个工作表
```

#### 创建新的工作表
```python
wb.sheets.add('NewSheet')  # 创建新的工作表并命名
```

#### 删除工作表
```python
wb.sheets['NewSheet'].delete()  # 删除名为 'NewSheet' 的工作表
```

### 4. Excel 公式操作
你可以通过 `xlwings` 操作 Excel 公式，就像在 Excel 中直接输入公式一样：

```python
sheet.range('A1').value = 10
sheet.range('A2').value = 20
sheet.range('A3').formula = "=A1+A2"  # 在 A3 单元格中设置公式
```

### 5. 运行宏
`xlwings` 还允许你在 Excel 中运行 VBA 宏。你可以通过 `macro` 方法调用已定义的宏。

```python
# 运行名为 'MyMacro' 的宏
wb.macro('MyMacro')()
```

### 6. 自动化操作
你可以创建一个 Excel 实例并将 `visible` 参数设置为 `False`，这样 Excel 就会在后台运行，不显示窗口。

```python
import xlwings as xw

# 启动 Excel 应用，但不显示窗口
app = xw.App(visible=False)
# 打开工作簿
wb = app.books.open('path_to_your_workbook.xlsx')

# 执行操作
sheet = wb.sheets[0]
sheet.range('A1').value = "Hello, World!"

# 保存并关闭工作簿
wb.save('modified_workbook.xlsx')
wb.close()

# 退出 Excel 应用
app.quit()
```

### 7. 与 pandas 集成
`xlwings` 可以与 `pandas` 集成，轻松地将 Excel 表格导入到 `pandas` DataFrame 中，或者将 `DataFrame` 导入到 Excel 表格中。

#### 从 Excel 导入数据到 pandas
```python
import xlwings as xw
import pandas as pd

# 启动 Excel 应用
app = xw.App(visible=False)
# 打开工作簿
wb = app.books.open('path_to_your_workbook.xlsx')

# 将工作表的内容导入到 pandas DataFrame 中
sheet = wb.sheets[0]
df = sheet.range('A1').expand().options(pd.DataFrame).value

print(df)

# 退出应用
app.quit()
```

#### 从 pandas 导入数据到 Excel
```python
import xlwings as xw
import pandas as pd

# 创建一个简单的 DataFrame
df = pd.DataFrame({'A': [1, 2, 3], 'B': [4, 5, 6]})

# 启动 Excel 应用
app = xw.App(visible=False)
# 创建新的工作簿
wb = app.books.add()

# 将 DataFrame 数据写入工作表
sheet = wb.sheets[0]
sheet.range('A1').value = df

# 保存文件
wb.save('output.xlsx')

# 退出应用
app.quit()
```

### 8. 退出应用
不要忘记在脚本结束时调用 `app.quit()` 来退出 Excel 应用程序，这样可以避免 Excel 进程在后台继续运行。

```python
app.quit()
```

### 9. 与 Excel 中的图表交互
`xlwings` 允许你创建和修改 Excel 图表。

#### 创建一个简单的图表
```python
import xlwings as xw

# 启动 Excel 应用
app = xw.App(visible=True)
# 创建一个新的工作簿
wb = app.books.add()

# 获取第一个工作表
sheet = wb.sheets[0]

# 向单元格写入数据
sheet.range('A1').value = ['Jan', 'Feb', 'Mar', 'Apr', 'May']
sheet.range('B1').value = [10, 20, 30, 40, 50]

# 插入图表
chart = sheet.charts.add()  # 创建图表对象
chart.chart_type = 'column_clustered'  # 设置图表类型为柱状图
chart.set_source_data(sheet.range('A1:B5'))  # 设置数据源

# 保存文件
wb.save('chart_example.xlsx')

# 退出应用
app.quit()
```

#### 修改现有图表
如果你已经有了图表，可以通过 `xlwings` 修改它的属性。

```python
import xlwings as xw

# 打开已有的工作簿
app = xw.App(visible=True)
wb = app.books.open('chart_example.xlsx')
sheet = wb.sheets[0]

# 获取现有图表
chart = sheet.charts[0]
chart.chart_type = 'line'  # 将柱状图更改为折线图

# 保存并退出
wb.save()
app.quit()
```

### 10. 自定义 Excel 任务栏按钮
`xlwings` 支持通过 VBA 创建自定义的任务栏按钮，调用 Python 函数。

#### 创建按钮并绑定到 Python 函数
```python
import xlwings as xw

# 启动 Excel 应用
app = xw.App(visible=True)
wb = app.books.add()

# 获取工作表
sheet = wb.sheets[0]

# 添加一个按钮到工作表
button = sheet.buttons.add(left=100, top=100, width=100, height=40)
button.text = 'Run Python'

# 绑定 Python 函数到按钮
button.on_action = lambda: print("Button clicked!")  # 这里可以调用一个 Python 函数

# 保存并退出
wb.save('button_example.xlsx')
app.quit()
```

### 11. 调用 Python 函数作为 Excel 宏
`xlwings` 允许你创建自定义的 Excel 函数（UDF），即通过 Python 编写的 Excel 函数，这样就可以在 Excel 中像使用内置函数一样使用你的 Python 函数。

#### 创建自定义函数并在 Excel 中使用
```python
import xlwings as xw

# 定义 Python 函数
def multiply(x, y):
    return x * y

# 启动 Excel 应用
app = xw.App(visible=True)
wb = app.books.add()

# 注册自定义函数
xw.func(multiply)

# 使用该函数
sheet = wb.sheets[0]
sheet.range('A1').value = 5
sheet.range('B1').value = 10
sheet.range('C1').formula = '=multiply(A1, B1)'

# 保存并退出
wb.save('udf_example.xlsx')
app.quit()
```

### 12. 从 Excel 中读取并修改图像
你可以通过 `xlwings` 插入或修改工作簿中的图像。

#### 插入图像
```python
import xlwings as xw

# 启动 Excel 应用
app = xw.App(visible=True)
wb = app.books.add()

# 获取工作表
sheet = wb.sheets[0]

# 插入图像到工作表
sheet.pictures.add('path_to_image.jpg', left=100, top=100)

# 保存并退出
wb.save('image_example.xlsx')
app.quit()
```

### 13. 操作 Excel 数据验证
你可以使用 `xlwings` 设置 Excel 单元格的数据验证，例如下拉菜单、日期选择等。

#### 设置数据验证（下拉菜单）
```python
import xlwings as xw

# 启动 Excel 应用
app = xw.App(visible=True)
wb = app.books.add()

# 获取工作表
sheet = wb.sheets[0]

# 设置数据验证，添加下拉菜单
validation_range = sheet.range('A1')
validation_range.validation.add(
    type='list',
    alert_style='warning',
    operator='between',
    formula1='"Option 1,Option 2,Option 3"'  # 选项
)

# 保存并退出
wb.save('validation_example.xlsx')
app.quit()
```

### 14. 操作 Excel 的条件格式
`xlwings` 也支持在 Excel 中应用条件格式。例如，你可以高亮显示某些单元格。

#### 设置条件格式
```python
import xlwings as xw

# 启动 Excel 应用
app = xw.App(visible=True)
wb = app.books.add()

# 获取工作表
sheet = wb.sheets[0]

# 向 A1:A5 添加一些数据
sheet.range('A1').value = 10
sheet.range('A2').value = 20
sheet.range('A3').value = 30
sheet.range('A4').value = 40
sheet.range('A5').value = 50

# 添加条件格式：将大于 30 的单元格设置为红色
sheet.range('A1:A5').api.FormatConditions.Add(
    Type=1, Operator=5, Formula1='30'
).Interior.Color = 255  # 设置为红色

# 保存并退出
wb.save('conditional_format_example.xlsx')
app.quit()
```

### 15. 向 Excel 插入超链接
你还可以使用 `xlwings` 在工作表中插入超链接。

```python
import xlwings as xw

# 启动 Excel 应用
app = xw.App(visible=True)
wb = app.books.add()

# 获取工作表
sheet = wb.sheets[0]

# 插入超链接
sheet.range('A1').value = 'Click Here'
sheet.range('A1').api.Hyperlinks.Add(
    Anchor=sheet.range('A1'),
    Address='https://www.example.com',
    TextToDisplay='Click Here'
)

# 保存并退出
wb.save('hyperlink_example.xlsx')
app.quit()
```
