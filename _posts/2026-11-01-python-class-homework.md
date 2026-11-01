---
layout: default
title: 课堂日常作业：函数、递归与蒙特卡罗（第 5 章）
date: 2026-11-01
categories: 课堂作业
tags: [Python, 函数, 递归, turtle, 科赫雪花, 蒙特卡罗]
---

# 课堂日常作业：函数、递归与蒙特卡罗（第 5 章）

> 本文汇总本阶段 Python 课堂日常作业。课堂作业统一存放在独立仓库 [**python-homework**](https://github.com/qls-study-blog/python-homework)，与各实验项目仓库分开管理；博客只做题目说明、思路讲解与源码索引。

## 作业清单

| 序号 | 题目 | 知识点 | 源码 |
|------|------|--------|------|
| 1 | `if __name__ == '__main__':` 的含义，why? | 模块入口、直接运行与导入的区别 | [笔记](https://github.com/qls-study-blog/python-homework/blob/main/notes/if_name_main.md) |
| 2 | P196 习题 5.1 字符田字格绘制 | 自定义函数、参数控制、循环嵌套 | [5.1_tianzige.py](https://github.com/qls-study-blog/python-homework/blob/main/ch5_function/5.1_tianzige.py) |
| 3 | 绘制科赫雪花，根据阶数生成（0~4 阶） | 递归函数、turtle 绘图 | [koch_snowflake.py](https://github.com/qls-study-blog/python-homework/blob/main/ch5_function/koch_snowflake.py) |
| 4 | 蒙特卡罗方法手搓图形 | 随机数、概率估算 π、turtle 散点 | [monte_carlo.py](https://github.com/qls-study-blog/python-homework/blob/main/ch5_function/monte_carlo.py) |

---

## 一、`if __name__ == '__main__':` 的含义，why?

### 题目

解释 Python 中 `if __name__ == '__main__':` 的含义，以及为什么要这样写。

### 要点

每个 `.py` 文件被加载时都会生成内置变量 `__name__`：

- 文件**被直接运行**（`python xxx.py`）时，`__name__` 的值是 `'__main__'`；
- 文件**被其他文件 `import` 导入**时，`__name__` 的值是模块名（去掉 `.py` 的文件名）。

因此这行判断的作用是：**只有当前文件被直接运行时，才执行下面缩进的入口代码；被导入时不执行**。这样模块既能独立运行做自测，又能被其他程序干净地复用函数定义，而不会在导入时自动执行测试或启动代码。

完整讲解与示例见仓库笔记：[`notes/if_name_main.md`](https://github.com/qls-study-blog/python-homework/blob/main/notes/if_name_main.md)。

---

## 二、习题 5.1 字符田字格绘制（P196）

### 题目

编写程序，用字符方式打印田字格；**采用函数方式，以格子数 `n` 为参数**，能够根据参数绘制任意大小的 `n × n` 田字格。

### 思路

把田字格的绘制封装成函数 `draw_grid(n)`：

- 横线由 `+----` 重复 `n` 次后结尾一个 `+`；
- 竖线由 `|    ` 重复 `n` 次后结尾一个 `|`；
- 每个格子高度为 4 行，外层循环 `n` 次，先画一条横线，再画 4 行竖线，最后补一条底边。

核心代码：

```python
def draw_grid(n: int) -> None:
    """用字符打印 n×n 的田字格。"""
    if not isinstance(n, int) or n <= 0:
        print("宽度必须为大于 0 的正整数")
        return

    horizontal = "+----" * n + "+"
    vertical = "|    " * n + "|"

    for _ in range(n):
        print(horizontal)
        for _ in range(4):
            print(vertical)
    print(horizontal)


if __name__ == "__main__":
    n = int(input("请输入田字格格子数 n: "))
    draw_grid(n)
```

以 `n = 2` 为例，运行结果为 2×2 田字格：

```
+----+----+
|    |    |
|    |    |
|    |    |
|    |    |
+----+----+
|    |    |
|    |    |
|    |    |
|    |    |
+----+----+
```

完整源码：[`ch5_function/5.1_tianzige.py`](https://github.com/qls-study-blog/python-homework/blob/main/ch5_function/5.1_tianzige.py)。

---

## 三、科赫雪花（0~4 阶，递归）

### 题目

绘制科赫雪花，根据阶数生成，支持 **0~4 阶**。

### 思路

科赫曲线是典型的递归图形：

- **0 阶**：一条直线段；
- **高阶**：把一条线段三等分，把中间一段替换成等边三角形的两条边（向上凸起），再对每一小段递归处理；
- **雪花**：由三条科赫曲线首尾相接围成等边三角形。

递归函数中先把长度三等分，按"画一段 → 左转 60° → 画一段 → 右转 120° → 画一段 → 左转 60° → 画一段"的顺序完成凸起造型。

核心递归代码：

```python
def koch_curve(t, length: float, depth: int) -> None:
    if depth == 0:
        t.forward(length)
        return

    length /= 3
    koch_curve(t, length, depth - 1)
    t.left(60)
    koch_curve(t, length, depth - 1)
    t.right(120)
    koch_curve(t, length, depth - 1)
    t.left(60)
    koch_curve(t, length, depth - 1)


def koch_snowflake(t, depth: int, length: float = 320) -> None:
    for _ in range(3):
        koch_curve(t, length, depth)
        t.right(120)
```

运行后输入阶数（0~4）即可弹出 turtle 窗口绘制；阶数越高，雪花轮廓越细碎。

完整源码：[`ch5_function/koch_snowflake.py`](https://github.com/qls-study-blog/python-homework/blob/main/ch5_function/koch_snowflake.py)。

---

## 四、蒙特卡罗方法手搓图形

### 题目

使用蒙特卡罗方法手搓图形：在正方形内随机大量投点，落入内切圆内与圆外的点分别用不同颜色绘制，直观形成一个圆，并据此估算圆周率 π。

### 思路

在边长为 2 的正方形（坐标范围 `[-1, 1]`）内随机投点：

- 正方形面积为 4，内切单位圆面积为 π；
- 落入圆内的点 / 总点数 ≈ 圆面积 / 正方形面积 = π / 4；
- 因此 π ≈ 4 × 圆内点数 / 总点数。

圆内点画红色，圆外点画蓝色，散点聚集起来就"手搓"出一个圆。

核心代码：

```python
def monte_carlo_pi(t, n_points: int, scale: int = 200) -> float:
    inside = 0
    for _ in range(n_points):
        x = random.uniform(-1, 1)
        y = random.uniform(-1, 1)
        if x * x + y * y <= 1:
            inside += 1
            t.color("red")
        else:
            t.color("blue")
        t.penup()
        t.goto(x * scale, y * scale)
        t.pendown()
        t.dot(3)
    return 4 * inside / n_points
```

投点数量越大，估算值越接近 π（例如 5 万点时约为 3.13 左右）。完整源码：[`ch5_function/monte_carlo.py`](https://github.com/qls-study-blog/python-homework/blob/main/ch5_function/monte_carlo.py)。

---

## 运行方式

仓库：[**python-homework**](https://github.com/qls-study-blog/python-homework)，仅依赖 Python 标准库（`turtle`、`random`）。

```bash
python ch5_function/5.1_tianzige.py      # 字符田字格（命令行输出）
python ch5_function/koch_snowflake.py   # 科赫雪花（turtle 窗口，输入阶数 0~4）
python ch5_function/monte_carlo.py      # 蒙特卡罗估算 π（turtle 窗口，输入投点数）
```

> 图形题目依赖 turtle 窗口，需在本地带图形界面的环境运行。
