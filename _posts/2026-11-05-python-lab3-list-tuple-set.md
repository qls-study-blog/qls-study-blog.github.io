---
layout: default
title: Python 实验（三）：列表、元组、集合深度探究
date: 2026-11-05
categories: 实验内容
tags: [Python, 列表, 元组, 集合, 可哈希, 切片删除]
---

# Python 实验（三）：列表、元组、集合深度探究

> 本文按《Python 列表、元组、集合 实验实训指导书》完成五个任务：切片删除、嵌套容器可哈希辨析、集合运算与推导式、去重方案对比、两班选课综合统计。全程只用 Python 标准库，测试用 `unittest`，完整工程与报告见文末源码仓库。

## 实验背景

本实验属"验证＋综合设计＋分析探究型"，2 学时。重点不在写多少代码，而在**把容易想当然的底层行为用实验证伪或证实**：切片删除到底按哪一份索引执行、元组为什么改了内容还能"不可变"、集合为什么不能嵌套、去重为什么会丢顺序。

实验环境：Python 3.14.5、VS Code、无第三方库依赖。

---

## 一、实验目的与需求澄清

### 1. 实验目的

1. 掌握 `list`、`tuple`、`set` 的可变性与可哈希判定规则，区分"引用不可变"与"元素内容不可变"；
2. 掌握切片语法与 `del` 切片删除的执行逻辑，理解"索引一次性计算"的陷阱；
3. 熟练使用交、并、差、对称差与集合推导式；
4. 准确判断嵌套容器的可哈希性；
5. 理解 `set` 去重丢失顺序的问题，实现保序去重；
6. 独立分析 `TypeError: unhashable type` 与切片删除预期不符等问题；
7. 根据需求自主选择容器完成小型数据处理任务。

### 2. 五种容器的特性对照

| 容器 | 有序性 | 可变性 | 是否可哈希 | 底层结构 |
|---|---|---|---|---|
| `list` | 有序 | 可变 | 不可哈希 | 动态数组，存对象引用 |
| `tuple` | 有序 | 不可变（引用地址不变） | 元素全可哈希时才可哈希 | 定长数组 |
| `set` | 无序 | 可变 | 不可哈希 | 哈希表 |
| `frozenset` | 无序 | 不可变 | 可哈希 | 只读哈希表 |
| `dict` | 有序（3.7+） | 可变 | 不可哈希 | 哈希表 |

### 3. 输出要求

- `python src/main.py` 一次输出五个任务结果，任务1 必须含"预判值 / 实际值 / 删除前索引 / 原因"；
- `unittest` 覆盖五任务，含边界与异常；
- 报告覆盖指导书第七节结构，并完整作答五道必答思考题。

### 4. 约束

1. 不引入第三方库（`unittest` 替代 pytest）；
2. 函数不修改传入的列表，统一返回新对象；
3. 集合类结果展示前统一 `sorted()`，避免哈希随机化导致报告数值漂移；
4. 保序去重必须保留第一次出现顺序，不能用 `sorted(set(...))` 代替。

---

## 二、核心源码

工程按任务拆成五个模块，`main.py` 统一输出。

### 2.1 任务1：切片删除 `src/task1_slice_delete.py`

```python
def removed_indices(length: int, sl: slice) -> List[int]:
    """按切片在原始序列上一次性算出将删除的索引。"""
    return list(range(*sl.indices(length)))


def delete_by_slice(seq: Sequence, sl: slice) -> List:
    """返回删除切片元素后的新列表，不修改传入的序列。"""
    removed = set(removed_indices(len(seq), sl))
    return [item for index, item in enumerate(seq) if index not in removed]


def remove_even_while_iterating(seq: Sequence) -> List[int]:
    """复现「边 for 遍历边 remove」的缺陷写法。"""
    data = list(seq)
    for item in data:
        if item % 2 == 0:
            data.remove(item)
    return data


def remove_even_safely(seq: Sequence) -> List[int]:
    """安全写法一：列表推导式生成新列表。"""
    return [item for item in seq if item % 2 != 0]


def remove_even_with_while(seq: Sequence) -> List[int]:
    """安全写法二：显式下标 + while，删除后不移动下标。"""
    data = list(seq)
    index = 0
    while index < len(data):
        if data[index] % 2 == 0:
            del data[index]
        else:
            index += 1
    return data
```

`slice.indices(n)` 把 `start/stop/step` 规范化成可套进 `range` 的三元组，负步长与省略端点都由它处理，所以算出来就是**删除前的原始索引**。

### 2.2 任务2：可哈希探测 `src/task2_hash.py`

```python
def probe_hashable(obj) -> dict:
    """把对象尝试放进 set，返回是否成功以及失败原因。"""
    container = set()
    try:
        container.add(obj)
        return {"ok": True, "error": None, "container": container}
    except TypeError as exc:
        return {"ok": False, "error": str(exc), "container": container}


def nested_set_demo() -> dict:
    """用 frozenset 实现「集合里放集合」。"""
    a = frozenset({1, 2})
    b = frozenset({3, 4})
    nested = {a, b}
    return {"nested": nested, "size": len(nested), "intersection": a & b, "union": a | b}
```

### 2.3 任务3：集合运算 `src/task3_set_ops.py`

```python
A = {2, 4, 6, 8, 10, 12}
B = {3, 6, 9, 12, 15}


def set_operations(a: set, b: set) -> dict:
    return {
        "intersection": a & b,
        "union": a | b,
        "a_minus_b": a - b,
        "b_minus_a": b - a,
        "symmetric_difference": a ^ b,
    }


def multiples_of_four(start: int = 1, end: int = 30) -> set:
    return {x for x in range(start, end + 1) if x % 4 == 0}
```

### 2.4 任务4：去重 `src/task4_dedup.py`

```python
def dedup_unordered(data: List) -> List:
    """方案1：list(set(data))，最简洁，但丢失原始顺序。"""
    return list(set(data))


def dedup_ordered_loop(data: List) -> List:
    """方案2：遍历 + 成员判断，保留第一次出现顺序。"""
    result = []
    for item in data:
        if item not in result:
            result.append(item)
    return result


def dedup_ordered_dict(data: List) -> List:
    """拓展：dict.fromkeys 一行实现保序去重。"""
    return list(dict.fromkeys(data))


def dedup_nested(nest: List[List]) -> List[List]:
    """嵌套列表去重：先转元组判重，最后按需转回列表。"""
    seen = set()
    result = []
    for inner in nest:
        key = tuple(inner)
        if key in seen:
            continue
        seen.add(key)
        result.append(list(key))
    return result
```

### 2.5 任务5：综合统计 `src/task5_course_stats.py`

```python
def course_stats(class1: List[int], class2: List[int]) -> dict:
    unique1 = dedup_ordered_loop(class1)
    unique2 = dedup_ordered_loop(class2)

    set1 = set(class1)
    set2 = set(class2)
    both = set1 & set2                 # ② 同时选两门
    only_one = set1 ^ set2             # ③ 只选其中一门
    only_class1 = set1 - set2          # ④ 选 1 班没选 2 班

    both_tuple = tuple(sorted(both))   # ⑤ 交集转元组

    return {
        "class1_unique": unique1,
        "class2_unique": unique2,
        "both": sorted(both),
        "only_one": sorted(only_one),
        "only_class1": sorted(only_class1),
        "both_tuple": both_tuple,
    }
```

---

## 三、测试用例 `tests/test_containers.py`

用标准库 `unittest`，共 **35 个用例**，按任务分成五个测试类。几个代表性用例：

```python
def test_loop_remove_skips_consecutive_even(self):
    """边界：连续偶数时边遍历边删除会漏删。"""
    self.assertEqual(remove_even_while_iterating([2, 4, 6]), [4])
    self.assertEqual(remove_even_while_iterating([1, 2, 3, 4, 5, 6]), [1, 3, 5])


def test_tuple_with_list_unhashable(self):
    result = probe_hashable((1, [2, 3]))
    self.assertFalse(result["ok"])
    self.assertIn("unhashable", result["error"])


def test_nested_list_dedup(self):
    nest = [[1, 2], [3, 4], [1, 2], [5, 6]]
    self.assertEqual(dedup_nested(nest), [[1, 2], [3, 4], [5, 6]])


def test_tuple_can_join_set(self):
    """⑤ 交集的整数元组可哈希，能再放回集合。"""
    holder = {self.result["both_tuple"]}
    self.assertIn((107, 109), holder)
```

边界与异常用例包括：空列表切片删除、`del ls[::-1]` 删空、切片删除不修改原列表、连续偶数漏删、空区间集合推导式、空列表去重、嵌套去重不污染原数据、两班为空时的统计。

---

## 四、运行结果与预判对比

### 4.1 三组切片删除（重点）

| 语句 | 源列表 | 删除前算出的索引 | 预判 | 实际 | 一致 |
|---|---|---|---|---|---|
| `del ls1[::3]` | `[0..9]` | 0、3、6、9 | `[1,2,4,5,7,8]` | `[1,2,4,5,7,8]` | ✓ |
| `del ls2[1::3]` | `[0..9]` | 1、4、7 | `[0,2,3,5,6,8,9]` | `[0,2,3,5,6,8,9]` | ✓ |
| `del ls3[::-2]` | `[0..5]` | 5、3、1 | `[0,2,4]` | `[0,2,4]` | ✓ |

```
$ python src/main.py
===== 任务1 切片删除深度探究 =====
del ls1[::3] 源列表 [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
  删除索引（删除前一次性计算）：[0, 3, 6, 9]
  预判：[1, 2, 4, 5, 7, 8]
  实际：[1, 2, 4, 5, 7, 8]  一致=True
del ls2[1::3] 源列表 [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
  删除索引（删除前一次性计算）：[1, 4, 7]
  预判：[0, 2, 3, 5, 6, 8, 9]
  实际：[0, 2, 3, 5, 6, 8, 9]  一致=True
del ls3[::-2] 源列表 [0, 1, 2, 3, 4, 5]
  删除索引（删除前一次性计算）：[5, 3, 1]
  预判：[0, 2, 4]
  实际：[0, 2, 4]  一致=True
```

负步长最容易误判：`ls3[::-2]` 是**从末尾取 5，再往前跳到 3、1**。若按删除过程中的动态索引理解，删掉 5 后列表只剩 5 个元素、索引整体偏移，结果会完全不同。

### 4.2 循环删除的缺陷（本次最值得记的一处）

```
[1,2,3,4,5,6] 循环删偶数 → [1, 3, 5]
[2,4,6] 循环删偶数       → [4]（漏删）
```

第一组**恰好删干净了**，看起来写法没问题。原因是相邻元素都不是待删元素，被跳过的下标正好由后面的偶数补上。换成连续偶数 `[2,4,6]`：删除 2 后 `4` 左移到索引 0，迭代下标已推进到 1，直接跳到 `6`，`4` 被漏删。

**结论**：边遍历边 `remove` 会漏删，是否暴露取决于数据分布，属于"偶发正确、实则错误"。安全写法两种，均已测试：

```python
# 写法一：列表推导式，遍历原序列、生成新列表
data = [x for x in data if x % 2 != 0]

# 写法二：显式下标 + while，删除后不移动下标
index = 0
while index < len(data):
    if data[index] % 2 == 0:
        del data[index]
    else:
        index += 1
```

### 4.3 任务2 可哈希实测

| 对象 | 结果 | 报错原文（Python 3.14） |
|---|---|---|
| `(1, 2, 3)` | 成功 | — |
| `(1, [2, 3])` | 报错 | `cannot use 'tuple' as a set element (unhashable type: 'list')` |
| `[1, 2]` | 报错 | `cannot use 'list' as a set element (unhashable type: 'list')` |
| `frozenset({1, 2})` | 成功 | — |
| `{1, 2}` | 报错 | `cannot use 'set' as a set element (unhashable type: 'set')` |

"集合里放集合"的正确写法是外层 `set`、内层 `frozenset`：`{frozenset({1,2}), frozenset({3,4})}`。

### 4.4 任务3 集合运算

```
交集 A & B = {12, 6}
并集 A | B = {2, 3, 4, 6, 8, 9, 10, 12, 15}
差集 A - B = {8, 2, 10, 4}
差集 B - A = {9, 3, 15}
对称差 A ^ B = {2, 3, 4, 8, 9, 10, 15}
1~30 内能被 4 整除的集合：{4, 8, 12, 16, 20, 24, 28}
```

### 4.5 任务4 去重对比

| 方案 | 代码 | 结果 | 优点 | 缺点 |
|---|---|---|---|---|
| 无序 | `list(set(data))` | `[1,2,3,5,7,9]` | 一行完成、O(n) | 丢顺序；元素必须可哈希 |
| 手写保序 | 遍历 + `not in` | `[5,2,7,9,3,1]` | 保序、直观 | 最坏 O(n²) |
| 字典键 | `list(dict.fromkeys(data))` | `[5,2,7,9,3,1]` | 保序且 O(n) | 依赖 3.7+ 字典保序 |

嵌套列表去重：`[[1,2],[3,4],[1,2],[5,6]] → [[1,2],[3,4],[5,6]]`。

### 4.6 任务5 综合统计

```
① 1 班不重复编号（保序）：[101, 103, 105, 107, 109]
① 2 班不重复编号（保序）：[102, 104, 106, 108, 109, 107]
② 同时选两门：[107, 109]
③ 只选其中一门（对称差）：[101, 102, 103, 104, 105, 106, 108]
④ 选了 1 班没选 2 班：[101, 103, 105]
⑤ 交集转元组：(107, 109)
```

口径校验：`s1 ^ s2 = (s1 - s2) | (s2 - s1)`，即 `{101,103,105} | {102,104,106,108}` 正好等于 ③ 的结果，与手算一致。

### 4.7 测试结果

```
$ python -m unittest discover -s tests -v
Ran 35 tests in 0.001s

OK
```

完整运行输出见仓库 `docs/images/run_output.txt` 与 `docs/images/unittest_output.txt`。

---

## 五、AI 协作过程（节选）

| 轮次 | 任务 | 人工修正 | 运行结果 |
|---|---|---|---|
| 第1轮 | 五任务拆模块 + 切片索引计算方案 | 要求必须输出"删除前索引集合"作为证据，不能只打印结果 | 三组索引与预判一致 |
| 第2轮 | 复现循环删除并给安全改写 | AI 用 `[1,2,3,4,5,6]→[1,3,5]` 就宣布"有 bug"，指出该数据其实删干净了，要求补能暴露缺陷的数据 | 追加 `[2,4,6]→[4]` 才证实 |
| 第3轮 | 可哈希探测 | 要求捕获 `TypeError` 并保留原始文本，不吞异常 | 2 成功 3 报错，符合预期 |
| 第4轮 | 集合运算与推导式 | 要求结果展示前统一 `sorted()` | 与手算一致 |
| 第5轮 | 三种去重 + 嵌套去重 | 要求嵌套去重不修改原列表 | `[[1,2],[3,4],[5,6]]`，原数据未变 |
| 第6轮 | 任务5 统计 | ① 必须保序，不能用 `sorted(set(...))` | 五问数值与手算一致 |
| 第7轮 | `unittest` 用例 | 要求覆盖空输入、空区间、反向切片、连续偶数 | `Ran 35 tests … OK` |

**踩坑记录**

1. AI 首版对"循环删除缺陷"的结论不成立，用 `[1,2,3,4,5,6]` 得到"正确"结果；换成连续偶数才暴露——**验证要主动构造能推翻结论的数据**。
2. Python 3.14 的报错文案变为 `cannot use 'tuple' as a set element (unhashable type: 'list')`，测试只断言关键字 `unhashable`，避免锁死版本文案。
3. 集合输出顺序不稳定，同一运算两次打印出现 `{12, 6}` 与 `{6, 12}`，报告与测试统一 `sorted()`。
4. 直接 `python src/main.py` 时 `sys.path[0]` 是 `src/`，`from src.xxx import` 会失败，需在入口显式把仓库根插入 `sys.path`。

---

## 六、必答思考题

### 1. `del ls[::3]` 按删除前索引还是动态索引？

按**删除前一次性计算**。`ls=[0..9]` 执行 `del ls[::3]` 删除的是索引 0、3、6、9，结果 `[1,2,4,5,7,8]`。若按动态索引，删掉 0 后"索引 3"会指向原来的 4，删除集合将完全不同。负步长同理，`del ls[::-2]` 删的是 5、3、1。

### 2. `t=(1,[2,3])` 为什么 `t[1].append(4)` 合法，但 `t` 不能入集合？

元组的"不可变"约束的是**元组保存的引用地址**，不是引用所指对象的内容。`t[1].append(4)` 改的是列表内容，元组存的仍是同一个列表对象，引用没变，所以合法。但可哈希要求整体哈希值在生命周期内固定，元组含 `list` 时内容一变哈希就变，因此 Python 判定其不可哈希。

### 3. 为什么普通 set 不能嵌套，frozenset 可以？

`set` 可变，`add`/`remove` 会改变内容，哈希值随之变化，不满足"元素必须可哈希"。`frozenset` 创建后不可修改，哈希固定，可作为集合元素或字典键。

### 4. for 遍历同时 remove 为什么删不干净？安全方案？

`for` 由内部下标驱动，删除当前元素后后续元素左移，迭代下标却照常 +1，紧跟的元素被跳过；被跳过的若也是待删元素就漏删。安全方案：列表推导式、`while` + 显式下标（删除后不递增）、倒序遍历删除、遍历副本删原列表。

### 5. `list(set(data))` 的缺陷？保序去重有哪些实现？

缺陷：① 丢失原始顺序；② 元素必须可哈希，嵌套 list 直接报错；③ 额外构建哈希表。保序实现：手写遍历 + `not in`、`dict.fromkeys`、`OrderedDict.fromkeys`。本次实测后两者结果均为 `[5, 2, 7, 9, 3, 1]`。

---

## 七、复盘

### 交给 AI 的环节

五任务模块骨架、`slice.indices()` 索引计算、可哈希 `try/except` 包装、三种去重与嵌套去重、`unittest` 用例框架。

### 自己拍板的环节

- 只用标准库，测试选 `unittest` 而非 pytest，不引入依赖；
- 集合结果统一 `sorted()` 展示，保证报告可复核；
- 任务1 追加"连续偶数"对照实验，用数据证明缺陷，而不是照抄结论；
- 任务5 ① 坚持保序，没有用 `sorted(set(...))` 图省事；
- 报告结构严格按指导书第七节，五道思考题全部作答；
- 函数不修改入参，示例之间互不污染。

### 人工风险审查

1. 逐条核对切片删除的索引与剩余元素，负步长单独确认"从末尾往前"；
2. 循环删除用两组数据对照，如实写明"本次恰好删干净"，不把巧合当正确；
3. `probe_hashable` 只捕获 `TypeError` 并保留原始错误，其他异常照常抛出；
4. 无序去重只断言"元素集合一致 + 长度"，不断言具体顺序；
5. 所有报告数值均取自实际运行输出，不手写估计。

### 可改进

- 补一组"含不可哈希元素"的去重用例，直接展示 `set(nest)` 抛错；
- 任务5 ⑤ 补"交集含列表则无法入集合"的反例；
- 35 个用例可按任务拆成 5 个测试文件，便于定位失败。

---

> 📎 项目结构：
> ```
> python-lab3-containers/
> ├── src/
> │   ├── main.py                 # 五任务统一入口
> │   ├── task1_slice_delete.py   # 切片删除与循环删除缺陷
> │   ├── task2_hash.py           # 可哈希探测与 frozenset 嵌套
> │   ├── task3_set_ops.py        # 集合运算与集合推导式
> │   ├── task4_dedup.py          # 三种去重 + 嵌套去重
> │   └── task5_course_stats.py   # 综合选课统计
> ├── tests/test_containers.py    # 35 个 unittest 用例
> └── docs/                       # 实验报告、需求澄清、AI对话记录、复盘、运行记录
> ```

## 源码仓库

本实验独立成一个项目仓库，完整代码、测试与实验报告见：

👉 [**python-lab3-containers**（GitHub 源码）](https://github.com/qls-study-blog/python-lab3-containers)

> 说明：博客只记录文章与结论，完整工程代码统一放在独立的实验项目仓库中。
