---
layout: default
title: Vibe Coding 实操 Lab1（一）：Excel 报表生成脚本
date: 2026-10-15
categories: Python 课程作业
tags: [Vibe Coding, Python, pandas, Excel]
---

# Vibe Coding 实操 Lab1（一）：Excel 报表生成脚本

> 本文是课程作业的第一篇，记录考题一（Excel 报表生成脚本）的完整实现：需求澄清、源码、测试、运行结果与复盘。

## 考核背景

本考核评估使用自然语言 + AI 编码工具完成 Python 工程任务的能力。重点不是手写全部代码，而是**需求拆解、任务分配、过程管控、结果验证**。评分维度：结果 + 过程 + 验证 + 风险 + 复盘。

---

## 一、需求澄清（交给 AI 前书面写清）

### 1. 目标
读取 `sales_raw.xlsx`，清洗脏数据，按销售员和地区汇总销售额，输出 `report.xlsx`。**不得修改原始文件。**

### 2. 输入
- 文件：`data/sales_raw.xlsx`
- 字段：日期、销售员、产品、数量、单价、地区
- 脏数据类型：空值、重复行、负数量、数量为 0

### 3. 输出
- 文件：`output/report.xlsx`
- 四个工作表：
  1. `明细清洗后`：清洗后的完整明细
  2. `按销售员`：销售员、销售额
  3. `按地区`：地区、销售额
  4. `总览`：月度总额

### 4. 清洗规则
- 删除关键字段（日期、销售员、产品、数量、单价、地区）缺失的行
- 剔除数量 ≤ 0 的行
- 按全字段去重

### 5. 统计规则
- 销售额 = 数量 × 单价
- 按销售员汇总、按地区汇总、月度总额

### 6. 边界与异常（不崩溃）
- 文件不存在 → 中文提示
- 空文件 → 中文提示
- 无有效数据 → 中文提示
- 原始文件只读，禁止写回

---

## 二、核心源码 `src/sales_report.py`

```python
import pandas as pd
import os
import sys

def clean_sales(df: pd.DataFrame) -> pd.DataFrame:
    """清洗销售数据 DataFrame"""
    if df.empty:
        return df.copy()

    df_clean = df.copy()
    # 数量、单价强制转数值，脏数据变 NaN
    df_clean['数量'] = pd.to_numeric(df_clean['数量'], errors='coerce')
    df_clean['单价'] = pd.to_numeric(df_clean['单价'], errors='coerce')

    required_cols = ['日期', '销售员', '产品', '数量', '单价', '地区']
    df_clean = df_clean.dropna(subset=required_cols)      # 删除关键字段缺失行
    df_clean = df_clean[df_clean['数量'] > 0]              # 剔除数量 <= 0
    df_clean = df_clean.drop_duplicates()                  # 全字段去重
    df_clean = df_clean.reset_index(drop=True)
    return df_clean


def build_stats(df: pd.DataFrame):
    """统计：按销售员、按地区汇总销售额以及月度总额"""
    if df.empty:
        empty_by_sales = pd.DataFrame(columns=['销售员', '销售额'])
        empty_by_region = pd.DataFrame(columns=['地区', '销售额'])
        empty_total = pd.DataFrame(columns=['月度总额'])
        return empty_by_sales, empty_by_region, empty_total

    df_stats = df.copy()
    df_stats['销售额'] = df_stats['数量'] * df_stats['单价']

    by_sales = df_stats.groupby('销售员', as_index=False)['销售额'].sum()
    by_sales = by_sales.sort_values(by='销售额', ascending=False).reset_index(drop=True)

    by_region = df_stats.groupby('地区', as_index=False)['销售额'].sum()
    by_region = by_region.sort_values(by='销售额', ascending=False).reset_index(drop=True)

    total = pd.DataFrame({'月度总额': [df_stats['销售额'].sum()]})
    return by_sales, by_region, total


def main():
    """主流程：读取、清洗、统计并导出 Excel 报表"""
    input_path = 'data/sales_raw.xlsx'
    output_path = 'output/report.xlsx'
    required_cols = ['日期', '销售员', '产品', '数量', '单价', '地区']

    if not os.path.exists(input_path):
        print(f"错误：找不到输入文件 '{input_path}'，请检查文件路径是否正确。")
        sys.exit(1)

    try:
        df = pd.read_excel(input_path, engine='openpyxl')
    except Exception as e:
        print(f"错误：读取文件 '{input_path}' 失败，请检查文件是否损坏。\n详细信息：{e}")
        sys.exit(1)

    if df.empty:
        print("错误：读取到的 Excel 文件为空，没有数据可供处理。")
        sys.exit(1)

    missing_cols = [col for col in required_cols if col not in df.columns]
    if missing_cols:
        print(f"错误：输入文件缺少以下关键字段：{', '.join(missing_cols)}。")
        sys.exit(1)

    df_clean = clean_sales(df)
    if df_clean.empty:
        print("提示：经过数据清洗后，没有留下任何有效的销售数据，无法生成报表。")
        sys.exit(0)

    by_sales, by_region, total = build_stats(df_clean)

    os.makedirs(os.path.dirname(output_path), exist_ok=True)
    try:
        with pd.ExcelWriter(output_path, engine='openpyxl') as writer:
            df_clean.to_excel(writer, sheet_name='明细清洗后', index=False)
            by_sales.to_excel(writer, sheet_name='按销售员', index=False)
            by_region.to_excel(writer, sheet_name='按地区', index=False)
            total.to_excel(writer, sheet_name='总览', index=False)
    except Exception as e:
        print(f"错误：写入输出文件 '{output_path}' 失败，请检查目录权限。\n详细信息：{e}")
        sys.exit(1)

    total_amount = total['月度总额'].iloc[0]
    print("✅ 报表生成成功！")
    print(f" - 清洗后有效数据行数：{len(df_clean)}")
    print(f" - 月度总销售额：{total_amount}")
    print(f" - 输出文件路径：{os.path.abspath(output_path)}")


if __name__ == "__main__":
    main()
```

---

## 三、pytest 测试用例 `tests/test_excel.py`

覆盖：正常流程、数量≤0、关键字段缺失、重复行去重、空 DataFrame 异常。

```python
import sys
import os
import pandas as pd
import pytest

sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), '..')))
from src.sales_report import clean_sales, build_stats


def get_base_df():
    data = {
        '日期': ['2023-01-01', '2023-01-02'],
        '销售员': ['张三', '李四'],
        '产品': ['A', 'B'],
        '数量': [2, 4],
        '单价': [10, 40],
        '地区': ['华东', '华北']
    }
    return pd.DataFrame(data)


def test_normal_flow():
    """正常用例：清洗与统计结果"""
    df_clean = clean_sales(get_base_df())
    assert len(df_clean) == 2
    by_sales, by_region, total = build_stats(df_clean)
    assert total['月度总额'].iloc[0] == 180
    assert by_sales[by_sales['销售员'] == '张三']['销售额'].iloc[0] == 20
    assert by_sales[by_sales['销售员'] == '李四']['销售额'].iloc[0] == 160


def test_clean_invalid_quantity():
    """边界1：剔除数量 <= 0 的行"""
    df = get_base_df()
    invalid = pd.DataFrame({
        '日期': ['2023-01-03', '2023-01-04'],
        '销售员': ['王五', '赵六'], '产品': ['C', 'D'],
        '数量': [0, -5], '单价': [10, 20], '地区': ['华南', '华西']
    })
    df_clean = clean_sales(pd.concat([df, invalid], ignore_index=True))
    assert len(df_clean) == 2
    assert (df_clean['数量'] > 0).all()


def test_clean_missing_fields():
    """边界2：删除关键字段缺失的行"""
    df = get_base_df()
    missing = pd.DataFrame({
        '日期': ['2023-01-05'], '销售员': [None], '产品': ['E'],
        '数量': [1], '单价': [10], '地区': ['华东']
    })
    df_clean = clean_sales(pd.concat([df, missing], ignore_index=True))
    assert len(df_clean) == 2
    assert df_clean['销售员'].notna().all()


def test_clean_duplicates():
    """边界3：全字段去重"""
    df = get_base_df()
    df_clean = clean_sales(pd.concat([df, df.iloc[[0]]], ignore_index=True))
    assert len(df_clean) == 2


def test_empty_dataframe():
    """异常用例：空 DataFrame 不报错且结构正确"""
    empty = pd.DataFrame(columns=['日期', '销售员', '产品', '数量', '单价', '地区'])
    assert clean_sales(empty).empty
    by_sales, by_region, total = build_stats(empty)
    assert by_sales.empty and list(by_sales.columns) == ['销售员', '销售额']
    assert by_region.empty and list(by_region.columns) == ['地区', '销售额']
    assert total.empty and list(total.columns) == ['月度总额']
```

---

## 四、运行结果

```
$ python src/sales_report.py
✅ 报表生成成功！
 - 清洗后有效数据行数：3
 - 月度总销售额：180
 - 输出文件路径：...\output\report.xlsx
```

```
$ pytest
5 passed in 0.24s
```

测试通过截图见 `docs/images/test_excel_result.png`。

---

## 五、AI 协作过程（节选）

| 轮次 | 任务 | 人工修正 | 运行结果 |
|------|------|----------|----------|
| 第1轮 | clean_sales 清洗函数 | 补了 `单价` 的 to_numeric | 6 行 → 3 行，负数量/0/重复均剔除 |
| 第2轮 | build_stats 统计函数 | 无，直接可用 | 按销售员：李四160、张三20；月度180，人工核对一致 |
| 第3轮 | main 主流程 + 异常处理 | 无，直接可用 | 四工作表齐全，数值核对一致 |
| 第4轮 | pytest 测试用例 | 无，直接可用 | 5 passed in 0.24s |

**踩坑记录**：首次贴代码 VS Code 用 GBK 编码导致中文乱码 import 失败，改为 UTF-8 后正常；pandas 读 xlsx 需显式 `engine='openpyxl'`。

---

## 六、复盘

### 交给 AI 的环节
清洗函数、统计函数、主流程、pytest 测试用例。

### 自己拍板的环节
- 需求澄清全部手写，未交给 AI
- 样例数据自造（老师未提供原始文件）
- 确认 `数量`、`单价` 都要 to_numeric
- 测试用例确保 ≥2 个边界用例 + 1 个异常用例
- 输出到 `output/`，原始文件只读

### 人工风险审查
1. 确认 `clean_sales`、`build_stats` 都用 `df.copy()`，不污染原始数据
2. 确认 main() 对文件不存在/读取失败/空文件/缺字段/清洗后无数据都有中文提示
3. 确认输出用 ExcelWriter 写到 output/，未触碰 data/
4. 编码与 openpyxl 引擎风险已规避

### 可改进
- 更早设置 VS Code UTF-8 编码
- AI 对话记录每轮立刻写
- 可再加"单价为字符串"的边界用例

---

> 📎 项目结构：
> ```
> excel_vibe/
> ├── data/sales_raw.xlsx      # 原始样例数据
> ├── output/report.xlsx       # 生成报表
> ├── src/sales_report.py      # 核心代码
> ├── tests/test_excel.py      # 测试用例
> └── docs/                    # 需求澄清、AI对话记录、复盘
> ```
