---
layout: default
title: Vibe Coding 实操 Lab1（二）：FastAPI 简易记账接口
date: 2026-11-01
categories: 实验内容
tags: [Vibe Coding, Python, FastAPI, pydantic, pytest, REST API]
---

# Vibe Coding 实操 Lab1（二）：FastAPI 简易记账接口

> 本文是 Lab1 考核的第二题，记录考题二（FastAPI 记账 HTTP 接口）的完整实现：需求澄清、接口设计、源码、测试、真实调用验证与复盘，与考题一（Excel 报表脚本）同属一份考核，各自独立成仓库。

## 考核背景

本考核评估使用自然语言 + AI 编码工具完成 Python 工程任务的能力。重点不是手写全部代码，而是**需求拆解、任务分配、过程管控、结果验证**。评分维度：结果 + 过程 + 验证 + 风险 + 复盘。

本题要求基于 FastAPI 实现一个简易记账服务：支持新增流水、按条件筛选分页查询、按日期区间汇总收入、支出与结余；非法参数返回 4xx 状态码与结构化错误信息；用 pytest 覆盖正常与边界 / 异常用例，并说明并发写入的保证方式。

---

## 一、需求澄清（交给 AI 前书面写清）

### 1. 目标

基于 FastAPI 实现一个简易记账 HTTP 服务，支持新增流水、按条件筛选分页查询、按日期区间汇总收入 / 支出 / 结余。数据采用进程内存存储，所有非法输入返回 4xx 状态码与结构化错误信息。

### 2. 输入

**新增流水 `POST /records`**（请求体 JSON）：

| 字段 | 类型 | 含义 | 约束 |
|---|---|---|---|
| amount | 数值 | 金额 | 必须大于 0，不允许为 0、负数或非数值 |
| type | 字符串 | 类型 | 只能为 `income`（收入）或 `expense`（支出） |
| category | 字符串 | 分类 | 非空，去除首尾空格后至少 1 个字符，例如 餐饮、工资、交通 |
| date | 字符串 | 日期 | 格式 `YYYY-MM-DD`，不接受非法日期 |

**筛选查询 `GET /records`**（查询参数全部可选，不传则不参与过滤）：

| 参数 | 含义 | 约束 |
|---|---|---|
| type | 按类型筛选 | `income` / `expense`，非法值返回 422 |
| category | 按分类精确筛选 | 非空才生效 |
| start_date | 起始日期（含） | `YYYY-MM-DD` |
| end_date | 结束日期（含） | `YYYY-MM-DD` |
| page | 页码 | 从 1 开始，默认 1，必须为正整数 |
| page_size | 每页条数 | 默认 10，最大 100，必须为正整数 |

**汇总 `GET /summary`**：`start_date`、`end_date` 均为必填。

### 3. 输出

新增流水返回 201 与刚入库的流水（含自增 id）；列表返回 `total / page / page_size / items`；汇总返回区间内收入合计、支出合计与结余：

```json
{
  "start_date": "2026-11-01",
  "end_date": "2026-11-02",
  "total_income": 5000.0,
  "total_expense": 55.5,
  "balance": 4944.5
}
```

其中结余 `balance = total_income - total_expense`。区间内无流水时，三项金额均为 0，不报错。

### 4. 校验规则

1. 金额 `amount` 必须为正数（`> 0`），否则 422。
2. 类型 `type` 只能是 `income` 或 `expense`，其他取值 422。
3. 分类 `category` 不能为空字符串或全空格，否则 422。
4. 日期必须是合法 `YYYY-MM-DD`，非法格式或不存在的日期（如 2026-13-40）422。
5. 分页参数 `page`、`page_size` 必须为正整数，`page_size` 不超过 100，否则 422。
6. `GET /summary` 缺少日期参数、日期格式非法，返回 4xx。

### 5. 数据存储与并发

本题允许内存或 SQLite；本方案选用**进程内存列表 + 线程锁**，服务重启后数据清空，符合课堂练习定位。所有写入在同一把互斥锁内完成，保证自增 ID 与列表更新的原子性；读取在锁内复制快照后返回，避免遍历期间列表被修改。

### 6. 边界与异常

- 列表筛选结果为空：`items` 为空数组，`total` 为 0，不报错。
- 汇总区间内无流水：收入、支出、结余均为 0。
- 非法参数：统一返回 4xx，响应体为结构化错误 `{"detail": [{"field": ..., "message": ...}]}`。
- 服务仅绑定本地 `127.0.0.1`，不写入外部数据库与第三方密钥。

---

## 二、核心源码

工程分为三个模块：`schemas.py`（pydantic 校验模型）、`store.py`（线程安全内存仓库）、`main.py`（路由与统一错误处理）。

### 2.1 数据模型 `src/schemas.py`

```python
# -*- coding: utf-8 -*-
"""考题二：FastAPI 记账接口 —— 请求 / 响应数据模型。"""

from datetime import date as DateType
from typing import Literal

from pydantic import BaseModel, Field, field_validator


class RecordIn(BaseModel):
    """新增流水的请求体模型。"""

    amount: float = Field(gt=0, description="金额，必须大于 0")
    type: Literal["income", "expense"] = Field(description="类型：income 收入 / expense 支出")
    category: str = Field(description="分类，如 餐饮、工资、交通")
    date: DateType = Field(description="日期，格式 YYYY-MM-DD")

    @field_validator("category")
    @classmethod
    def category_not_blank(cls, value: str) -> str:
        """去除首尾空格后校验分类非空，全空格视为非法输入。"""
        cleaned = value.strip()
        if not cleaned:
            raise ValueError("分类不能为空")
        return cleaned


class RecordOut(BaseModel):
    """流水响应模型（含自增 id）。"""

    id: int
    amount: float
    type: Literal["income", "expense"]
    category: str
    date: DateType


class SummaryOut(BaseModel):
    """区间汇总响应模型。"""

    start_date: DateType
    end_date: DateType
    total_income: float = Field(description="区间内收入合计")
    total_expense: float = Field(description="区间内支出合计")
    balance: float = Field(description="结余 = 收入合计 - 支出合计")
```

### 2.2 内存仓库 `src/store.py`

```python
# -*- coding: utf-8 -*-
"""考题二：内存记账存储（线程安全）。"""

from datetime import date
from threading import Lock
from typing import Optional


class RecordStore:
    """线程安全的内存流水仓库。"""

    def __init__(self) -> None:
        self._records: list[dict] = []
        self._next_id: int = 1
        self._lock = Lock()

    def add(self, item: dict) -> dict:
        """新增一条流水，分配自增 id。"""
        with self._lock:
            record = {
                "id": self._next_id,
                "amount": round(float(item["amount"]), 2),
                "type": item["type"],
                "category": item["category"].strip(),
                "date": item["date"],
            }
            self._records.append(record)
            self._next_id += 1
            return dict(record)

    def query(self, record_type=None, category=None,
              start_date: Optional[date] = None,
              end_date: Optional[date] = None) -> list[dict]:
        """按条件筛选流水，按 id 升序返回快照副本。"""
        with self._lock:
            snapshot = [dict(r) for r in self._records]

        result = []
        for record in snapshot:
            if record_type is not None and record["type"] != record_type:
                continue
            if category is not None and record["category"] != category.strip():
                continue
            if start_date is not None and record["date"] < start_date:
                continue
            if end_date is not None and record["date"] > end_date:
                continue
            result.append(record)
        return result

    def summarize(self, start_date: date, end_date: date) -> dict:
        """统计指定日期区间（含两端）内的收入合计、支出合计与结余。"""
        with self._lock:
            snapshot = [dict(r) for r in self._records]

        total_income = 0.0
        total_expense = 0.0
        for record in snapshot:
            if record["date"] < start_date or record["date"] > end_date:
                continue
            if record["type"] == "income":
                total_income += record["amount"]
            elif record["type"] == "expense":
                total_expense += record["amount"]

        return {
            "start_date": start_date,
            "end_date": end_date,
            "total_income": round(total_income, 2),
            "total_expense": round(total_expense, 2),
            "balance": round(total_income - total_expense, 2),
        }
```

### 2.3 路由与统一错误处理 `src/main.py`

```python
# -*- coding: utf-8 -*-
"""考题二：FastAPI 记账接口主程序。"""

from datetime import date
from typing import Optional

from fastapi import FastAPI, HTTPException, Query, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

from src.schemas import RecordIn, RecordOut, SummaryOut
from src.store import RecordStore


def create_app(store: Optional[RecordStore] = None) -> FastAPI:
    """创建 FastAPI 应用，便于测试时注入独立的内存存储。"""
    app = FastAPI(title="Lab1 考题二：简易记账接口", version="1.0.0")
    app.state.store = store or RecordStore()

    @app.exception_handler(RequestValidationError)
    async def validation_error_handler(_: Request, exc: RequestValidationError) -> JSONResponse:
        """把校验错误统一整理成结构化的 422 响应。"""
        errors = []
        for err in exc.errors():
            field = ".".join(str(x) for x in err["loc"][1:]) or str(err["loc"][0])
            errors.append({"field": field, "message": err["msg"]})
        return JSONResponse(status_code=422, content={"detail": errors})

    @app.post("/records", response_model=RecordOut, status_code=201)
    def add_record(payload: RecordIn):
        """新增一条流水。"""
        return app.state.store.add({
            "amount": payload.amount,
            "type": payload.type,
            "category": payload.category,
            "date": payload.date,
        })

    @app.get("/records")
    def list_records(
        record_type: Optional[str] = Query(None, alias="type", description="income / expense"),
        category: Optional[str] = Query(None, description="分类精确匹配"),
        start_date: Optional[date] = Query(None, description="起始日期（含）YYYY-MM-DD"),
        end_date: Optional[date] = Query(None, description="结束日期（含）YYYY-MM-DD"),
        page: int = Query(1, ge=1, description="页码，从 1 开始"),
        page_size: int = Query(10, ge=1, le=100, description="每页条数，1~100"),
    ):
        """按条件筛选流水并分页返回。"""
        if record_type is not None and record_type not in ("income", "expense"):
            raise HTTPException(
                status_code=422,
                detail=[{"field": "type", "message": "类型只能为 income 或 expense"}],
            )
        records = app.state.store.query(
            record_type=record_type, category=category,
            start_date=start_date, end_date=end_date,
        )
        total = len(records)
        start = (page - 1) * page_size
        items = records[start:start + page_size]
        return {"total": total, "page": page, "page_size": page_size, "items": items}

    @app.get("/summary", response_model=SummaryOut)
    def summary(start_date: date = Query(...), end_date: date = Query(...)):
        """返回指定日期区间内的收入、支出与结余。"""
        return app.state.store.summarize(start_date, end_date)

    @app.get("/")
    def health():
        """健康检查，便于确认服务已启动。"""
        return {"status": "ok", "message": "记账接口已启动"}

    return app


app = create_app()
```

---

## 三、pytest 测试用例 `tests/test_api.py`

每个用例通过 `create_app(RecordStore())` 获得全新内存存储，用例之间互不影响。共 **13 个用例**，分为正常新增、边界 / 异常、查询筛选分页、区间汇总四组。

```python
import os
import sys

from fastapi.testclient import TestClient

sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), "..")))
from src.main import create_app
from src.store import RecordStore


def make_client() -> TestClient:
    """创建带全新内存存储的测试客户端，保证用例之间互不影响。"""
    return TestClient(create_app(RecordStore()))


def test_add_records_success():
    """正常用例：新增收入、支出，断言 201、字段回显正确、id 自增。"""
    client = make_client()
    r1 = client.post("/records", json={
        "amount": 5000.0, "type": "income", "category": "工资", "date": "2026-11-01"
    })
    assert r1.status_code == 201
    assert r1.json()["id"] == 1

    r2 = client.post("/records", json={
        "amount": 35.5, "type": "expense", "category": "餐饮", "date": "2026-11-01"
    })
    assert r2.status_code == 201
    assert r2.json()["id"] == 2


def test_amount_must_be_positive():
    """边界1：金额为 0 与负数均返回 422。"""
    client = make_client()
    zero = client.post("/records", json={
        "amount": 0, "type": "expense", "category": "餐饮", "date": "2026-11-01"
    })
    assert zero.status_code == 422
    negative = client.post("/records", json={
        "amount": -50, "type": "expense", "category": "餐饮", "date": "2026-11-01"
    })
    assert negative.status_code == 422


def test_summary_calculation():
    """汇总：收入合计、支出合计、结余与人工手算一致。"""
    client = make_client()
    client.post("/records", json={"amount": 5000.0, "type": "income", "category": "工资", "date": "2026-11-01"})
    client.post("/records", json={"amount": 35.5, "type": "expense", "category": "餐饮", "date": "2026-11-01"})
    client.post("/records", json={"amount": 20.0, "type": "expense", "category": "交通", "date": "2026-11-01"})
    client.post("/records", json={"amount": 80.0, "type": "expense", "category": "餐饮", "date": "2026-11-02"})
    client.post("/records", json={"amount": 2000.0, "type": "income", "category": "兼职", "date": "2026-11-02"})

    body = client.get("/summary", params={
        "start_date": "2026-11-01", "end_date": "2026-11-02"
    }).json()
    # 收入 5000 + 2000 = 7000；支出 35.5 + 20 + 80 = 135.5；结余 = 6864.5
    assert body["total_income"] == 7000.0
    assert body["total_expense"] == 135.5
    assert body["balance"] == 6864.5
```

完整用例还覆盖：类型非法、日期格式错误、分类全空格、列表查询类型非法、类型 / 分类 / 日期区间筛选、分页、分页非法、空区间归零、汇总缺少日期参数 422，完整代码见仓库 `tests/test_api.py`。

---

## 四、运行结果

启动服务：

```
$ uvicorn src.main:app --host 127.0.0.1 --port 8000
```

真实启动后依次调用新增、筛选、汇总接口，记录如下：

```
POST income: {"id":1,"amount":5000.0,"type":"income","category":"salary","date":"2026-11-01"}
POST expense1: {"id":2,"amount":35.5,"type":"expense","category":"food","date":"2026-11-01"}
POST expense2: {"id":3,"amount":20.0,"type":"expense","category":"transport","date":"2026-11-02"}
GET expense total=2
SUMMARY: {"start_date":"2026-11-01","end_date":"2026-11-02",
          "total_income":5000.0,"total_expense":55.5,"balance":4944.5}
negative amount status=422
invalid query type status=422
```

汇总数值与人工手算一致：支出 35.5 + 20 = 55.5，结余 5000 − 55.5 = 4944.5。

```
$ pytest tests/ -v
collected 13 items
...
======================== 13 passed, 1 warning in 1.50s ========================
```

pytest 完整运行记录见仓库 `docs/images/pytest_output.txt`，真实接口调用记录见 `docs/images/live_api_check.txt`。

---

## 五、AI 协作过程（节选）

| 轮次 | 任务 | 人工修正 | 运行结果 |
|------|------|----------|----------|
| 第1轮 | pydantic 请求 / 响应模型 | 字段名 `date` 与导入类型同名导致注解解析失败，改为类型别名 `DateType` | 模型类成功构建 |
| 第2轮 | 线程安全内存仓库 | 金额入库 `round(..., 2)`，分类入库前 `strip()` | ID 1、2、3 自增，筛选汇总正确 |
| 第3轮 | FastAPI 路由 + 统一 422 处理器 | 补充 `GET /records` 的 `type` 取值校验，非法值直接 422 | 三接口正常，非法输入返回结构化错误 |
| 第4轮 | pytest 测试用例 | `strip_whitespace` 在 pydantic v2 已弃用，改用 `field_validator` 清洗分类；补查询类型非法用例 | 13 passed |

**踩坑记录**：

1. pydantic 模型字段名与导入类型同名（`date: date`）会导致类命名空间内注解解析失败，需要用类型别名规避。
2. pydantic v2 不再支持在 `Field(...)` 上传 `strip_whitespace`，字符串清洗应使用 `field_validator`。
3. 本机 Python 为 3.14.5，依赖安装在项目内 `.venv`，不污染全局环境。

---

## 六、复盘

### 交给 AI 的环节

pydantic 数据模型、内存仓库骨架、FastAPI 路由与错误处理器、pytest 用例框架。

### 自己拍板的环节

- 需求澄清全部先手写：目标、字段约束、筛选条件、分页边界、汇总口径、并发要求，先写成文档再编码。
- 存储选型：在内存与 SQLite 之间选择内存列表 + 线程锁，课堂练习无需落盘、测试隔离简单。
- 接口字段使用英文 `amount / type / category / date`，类型取值 `income / expense`，错误提示用中文。
- 每个用例使用全新存储，保证测试隔离。
- 在题目"正常 + 至少 2 个边界 / 异常"基础上补齐到 13 个用例。

### 人工风险审查

1. **并发 ID 冲突**：确认自增 ID 与列表追加在同一把 `Lock` 内完成，查询先复制快照，不在遍历期间修改内部列表。
2. **非法输入崩溃**：确认金额非正、类型非法、日期非法、分类为空、分页越界都被框架拦截为 422，不会进入业务逻辑抛未捕获异常。
3. **浮点金额误差**：金额入库与汇总结果均 `round(..., 2)`，避免浮点累加出现长尾小数。
4. **错误信息结构化**：422 响应统一为 `detail` 数组，逐条给出字段与原因，而不是框架默认的英文原始堆栈。
5. **环境隔离**：依赖安装在项目内 `.venv`，`.gitignore` 排除虚拟环境、缓存与 IDE 配置。

### 可改进

- 把内存存储抽象成接口，后续切换 SQLite 时路由层不用改动。
- 为汇总接口补充日期区间倒置（`start_date` 晚于 `end_date`）的校验与提示。
- 增加多线程并发写入压测用例，直接验证 ID 不重复。

---

> 📎 项目结构：
> ```
> fastapi-expense-tracker-lab1/
> ├── src/
> │   ├── main.py            # FastAPI 应用、路由、统一 422 处理
> │   ├── schemas.py         # pydantic 请求 / 响应模型
> │   └── store.py           # 线程安全内存仓库
> ├── tests/test_api.py      # 13 个 pytest 用例
> ├── docs/                  # 需求澄清、AI 对话记录、复盘、运行记录
> ├── requirements.txt
> └── README.md
> ```

## 源码仓库

本实验独立成一个项目仓库，完整代码、测试与过程文档见：

👉 [**fastapi-expense-tracker-lab1**（GitHub 源码）](https://github.com/qls-study-blog/fastapi-expense-tracker-lab1)

> 说明：博客仓库只记录文章与说明，完整工程代码统一放在独立的实验项目仓库中。
