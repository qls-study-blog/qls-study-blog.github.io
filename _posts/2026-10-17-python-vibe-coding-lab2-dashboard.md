---
layout: default
title: Vibe Coding 实操 Lab2：校园地图与天气数据看板
date: 2026-10-17
categories: Python 课程作业
tags: [Vibe Coding, Python, Streamlit, Folium, 数据看板, 高德地图]
---

# Vibe Coding 实操 Lab2：校园地图与天气数据看板

> 本文是课程作业的第二篇，记录实验二（数据看板的设计与实现）的完整实现：需求分析、技术选型、核心功能开发、踩坑与复盘。

## 实验背景

本次实验要求设计并实现一个数据看板，掌握从需求分析、版面设计、数据处理到看板落地的完整流程。基础任务包括：

1. 看板需求分析与版面设计
2. 实验数据准备与预处理
3. 基础可视化图表搭建（至少3种图表）
4. 静态数据看板整合实现
5. 实验结果验证与总结

选做任务包括动态效果优化、图表联动、多维度分析；扩展任务包括实时数据看板、数据预警。

---

## 技术选型

| 模块 | 技术 | 说明 |
|------|------|------|
| Web框架 | Streamlit | 快速搭建数据看板交互界面 |
| 地图渲染 | Folium + streamlit-folium | 高德底图，支持交互点击 |
| 地图数据源 | 高德驾车路径规划 API | 获取校园公交真实轨迹 |
| 天气数据 | 和风天气 API | 实时天气 + 未来24小时预报 |
| 图表库 | Plotly | 交互式柱状图/折线图/饼图 |
| 数据处理 | Pandas | 数据清洗与汇总 |

---

## 项目结构

```
campus_dashboard/
├── app.py                  # 主入口：页面布局与交互逻辑
├── config.py               # API密钥与配置
├── data/
│   ├── weather.py          # 和风天气API数据获取
│   ├── campus.py           # 校园建筑/站点坐标数据
│   └── mock.py             # 模拟数据 + 高德路径规划调用
└── charts/
    ├── map_view.py         # Folium地图渲染
    ├── analytics.py        # 柱状图/折线图/饼图/表格
    ├── bar_chart.py        # 人流量排行
    ├── line_chart.py       # 温度降水趋势
    ├── pie_chart.py        # 人群活动结构
    └── table_view.py       # 建筑明细表格
```

---

## 核心功能实现

### 一、天气概览区

对接和风天气 API，展示当前温度、相对湿度、风速、天气状况四个核心指标：

```python
weather_data = fetch_weather_data()
now_weather = weather_data["now"]

col1, col2, col3, col4 = st.columns(4)
with col1:
    st.metric(label="当前温度", value=f"{now_weather['temp']} ℃")
with col2:
    st.metric(label="相对湿度", value=f"{now_weather['humidity']} %")
```

### 二、校园地图（核心亮点）

地图基于 Folium + 高德底图实现，包含三层可切换图层：

#### 1. 5路公交轨迹（橙色线）

这是本次开发耗时最久的部分。最初用硬编码坐标画直线，完全不贴合真实道路。后来改用**高德驾车路径规划 API**，传入起点、终点和途经点（真实公交站点坐标），获取高德返回的真实道路轨迹。

```python
def get_bus_route_from_amap():
    params = {
        "origin": f"{stops.iloc[0]['经度']},{stops.iloc[0]['纬度']}",
        "destination": f"{stops.iloc[-1]['经度']},{stops.iloc[-1]['纬度']}",
        "waypoints": waypoints,  # 真实公交站点 + 东缘途经点
        "extensions": "all",
        "key": AMAP_KEY,
    }
    resp = requests.get(
        "https://restapi.amap.com/v3/direction/driving",
        params=params, timeout=8
    ).json()
    # 解析 steps 中的 polyline 坐标点
```

#### 2. 回环裁剪算法

高德 API 返回的轨迹偶尔会出现"绕圈"问题——在校园内部路禁行的情况下，高德会绕到校外主干道再回来，产生多余的环路。我们实现了两层裁剪：

- **向北折返检测**：正常公交轨迹一路向南（纬度单调递减），遇到纬度明显向北增大（>20米）的冗余点直接剔除
- **环路检测**：如果当前点非常接近之前某个已保留的点（约50米内重复），说明绕了一圈回来，去掉中间环路

```python
def _trim_loops(coords):
    trimmed = [coords[0]]
    for i in range(1, len(coords)):
        prev = trimmed[-1]
        cur = coords[i]
        # 向北折返 → 剔除
        if cur[0] > prev[0] + 0.0002:
            continue
        # 环路检测 → 绕回来就去掉中间段
        loop_back = -1
        for j in range(len(trimmed) - 2, max(-1, len(trimmed) - 30), -1):
            if (abs(cur[0] - trimmed[j][0]) < 0.0005
                    and abs(cur[1] - trimmed[j][1]) < 0.0005):
                loop_back = j
                break
        if loop_back >= 0:
            trimmed = trimmed[:loop_back + 1]
            continue
        trimmed.append(cur)
    return trimmed
```

#### 3. 通勤车（紫色大标 🚌）

标记教职工通勤车发车点（教四楼、行知楼），弹窗内嵌学校官网的官方发车时刻表。

#### 4. 观光车呼叫点（绿色图标）

标记校园观光车各呼叫站点位置。

#### 5. 坐标拾取器

侧边栏提供坐标拾取工具，开启后点击地图任意位置即可获取经纬度，支持记录、查看列表、下载 CSV——这在校正站点坐标时非常实用。

### 三、数据分析图表区

看板下半部分展示四组图表：

- **各类型建筑人流量排行**（柱状图）：食堂、教学楼、图书馆等建筑的人流量对比
- **未来24小时温度与降水趋势**（折线图）：来自和风天气预报数据
- **校园人群活动结构**（饼图）：教学、就餐、运动、通勤等活动占比
- **校园建筑明细**（表格）：建筑名称、类型、人流量等明细数据

### 四、暗色模式

侧边栏提供暗色模式开关，通过动态 CSS 注入切换整个看板的配色方案（背景色、卡片色、文字色）。

---

## 踩坑与复盘

### 坑1：高德 API 轨迹不贴合预期

**问题**：最初不加任何途经点时，高德把一号食堂到二号食堂的轨迹绕到了西边，经过行知楼，不是我们想要的沿东缘主干道的弧线。

**解决**：在东缘那条实际南北路上加一个途经点 `(110.3045, 25.0622)`，引导高德沿东缘道路向南，而不是向西折回。

### 坑2：轨迹出现多余环路

**问题**：高德 API 在某些路段会绕一个圈（比如二号食堂到南大门之间，先向西沿中心环线走一段，再回头向东），视觉上多了一圈。

**解决**：实现 `_trim_loops` 环路检测算法，识别"绕一圈回到之前经过的点"的模式，自动裁掉中间冗余段。

### 坑3：改代码后页面没变化

**问题**：修改 Python 文件后只刷新浏览器页面，地图轨迹纹丝不动。

**解决**：Streamlit 需要重启进程才会重新加载代码。Ctrl+C 终止后重新 `streamlit run app.py`。

---

## 运行效果

```bash
# 安装依赖
pip install streamlit folium streamlit-folium plotly pandas requests

# 运行
streamlit run app.py
```

打开 `http://localhost:8501` 即可看到完整的校园地图与天气数据看板。

---

## 总结

这次实验完整走了一遍数据看板从需求到落地的流程：

- **需求分析**：从实验要求出发，确定了天气概览 + 校园地图 + 数据分析三大模块
- **技术选型**：Streamlit 快速搭建界面，Folium 处理地图，Plotly 做图表，API 获取实时数据
- **核心难点**：高德路径规划 API 的轨迹调优——通过途经点引导 + 回环裁剪算法，让公交轨迹贴合真实道路
- **扩展功能**：坐标拾取器、暗色模式、可切换图层等，提升了看板的实用性和交互性

整体来看，Vibe Coding 的核心不是让 AI 直接生成完美代码，而是**需求拆解 + 过程管控 + 结果验证**——你得知道自己要什么效果，然后不断调试、验证、修正。
