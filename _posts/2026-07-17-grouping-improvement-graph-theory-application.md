---
title: "分组改进：图论应用（待完成）"
date: 2026-07-17 21:00:00 +0800
categories:
  - Python
  - 算法
tags:
  - Python
  - 图论
  - 模拟退火
  - 组合优化
  - 分组算法
mathjax: false
excerpt: "从图论视角拆解 TournamentOptimizerImproved 类的入口模块：导入库、初始化参数、必要检查与命令行入口。"
---

> 本文对应源码文件：`/Users/jiayifan/PycharmProjects/PythonProject2/分组_改进.py`

最近在处理一个积分赛分组问题：给定若干队伍，要求每组人数固定、每队参赛场次固定，并且尽量让任意两支队伍都交手过。从图论角度看，这相当于用若干 `K₅` 子图去覆盖完全图 `K₁₅` 的所有边。为了求解它，我写了一个基于**模拟退火**的 Python 类 `TournamentOptimizerImproved`。下面把它的前四个模块拆开讲解，帮助理解整体结构。

---

## 模块 1：导入工具库

```python
import random
import math
import time
from itertools import combinations
from collections import defaultdict
```

这些库的分工很清晰：

| 库 | 作用 |
|---|---|
| `random` | 生成随机初始分组、随机交换、模拟退火中的概率接受 |
| `math` | 计算理论下界、模拟退火的温度衰减 |
| `time` | 统计求解运行时间 |
| `itertools.combinations` | 快速列出一个组内所有两两组合 |
| `collections.defaultdict` | 记录每对队伍的交手次数，默认值为 0 |

这里没有一个外部依赖，全部是 Python 标准库。如果想用精确求解器，可以额外安装 `ortools`。

---

## 模块 2：类与初始化参数

```python
class TournamentOptimizerImproved:
    def __init__(self, t, n, m):
        self.t = t              # 每组队伍数，例如 5
        self.n = n              # 每场比赛的组数，例如 3
        self.m = m              # 每支队伍参赛场次，例如 4
        self.total_teams = t * n       # 总队伍数，例如 15
        self.total_matches = m         # 总比赛场数
        self.required_pairs = self.total_teams * (self.total_teams - 1) // 2
```

`__init__` 是 Python 类的**初始化方法**，创建对象时自动调用。它把用户输入的三个参数扩展成后续需要的一系列常量：

- `total_teams`：总队伍数，由 `t × n` 得到；
- `total_matches`：总比赛场数，这里等于 `m`，因为模型假设**每场比赛所有队伍都参赛**；
- `required_pairs`：需要覆盖的不同的对手对总数，即 `C(total_teams, 2)`。

> 注意：这里 `total_matches = m` 是对原代码的一个关键修正。原代码写成 `n * m`，会导致每支队伍实际参赛 `n × m` 次，与参数 `m` 的语义矛盾。

---

## 模块 3：数学必要条件检查

```python
def check_necessary_conditions(self):
    print(f"\n📊 参数检查: {self.total_teams}支队伍, ...")

    max_opponents = self.m * (self.t - 1)
    required_opponents = self.total_teams - 1
    if max_opponents < required_opponents:
        print(f"❌ [单队容量不足] ...")
        return False

    pairs_per_match = self.n * self.t * (self.t - 1) // 2
    min_matches = math.ceil(self.required_pairs / pairs_per_match)

    if self.total_matches < min_matches:
        print(f"❌ [理论下界拦截] ...")
        return False

    match_capacity = self.total_matches * pairs_per_match
    if match_capacity < self.required_pairs:
        print(f"❌ [全局容量不足] ...")
        return False

    print(f"✅ 数学必要条件检查通过 ...")
    return True
```

这个方法在做三件事，相当于给问题做一个**快速体检**：

### 1. 单队容量检查

每支队伍打 `m` 场，每场遇到 `t - 1` 个对手，所以最多能遇到 `m × (t - 1)` 个不同的对手。若要覆盖全部 `total_teams - 1` 个对手，必须满足：

```
m × (t - 1) ≥ total_teams - 1
```

否则，即使安排得再巧，某支队伍也永远见不到足够的对手。

### 2. 理论最少场数检查

每场比赛能提供 `n × C(t, 2)` 个不同的对手对。要覆盖 `required_pairs` 对，至少需要：

```
min_matches = ceil(required_pairs / (n × C(t, 2)))
```

如果 `total_matches < min_matches`，直接判定无解。

### 3. 全局容量检查

所有比赛加起来能提供的对手对总数必须不少于需要覆盖的对数：

```
total_matches × n × C(t, 2) ≥ required_pairs
```

这三个检查都通过，才进入后面的模拟退火；否则直接返回 `False`，避免做无效计算。

---

## 模块 10：命令行入口

```python
def main():
    print("=== 积分赛分组求解器（改进版）===")
    try:
        t = int(input("每组队伍数量 (t): ").strip() or "5")
        n = int(input("每场比赛的组数 (n): ").strip() or "3")
        m = int(input("每支队伍参赛场次 (m): ").strip() or "5")

        optimizer = TournamentOptimizerImproved(t, n, m)
        solution = optimizer.solve(max_iter=300000)
        if solution is None:
            return
        result, uncovered = solution

        if result:
            print("\n=== 最终分组方案 ===")
            for i, match in enumerate(result):
                print(f"第 {i + 1} 场比赛:")
                for j, group in enumerate(match):
                    print(f"  组{j + 1}: {[x + 1 for x in group]}")
            if uncovered == 0:
                print("\n✅ 该方案覆盖全部对手对。")
            else:
                print(f"\n⚠️ 该方案未覆盖 {uncovered} 对对手。")

    except ValueError:
        print("❌ 输入无效，请输入整数！")


if __name__ == "__main__":
    main()
```

### `main()` 的作用

`main()` 是程序的**交互入口**：

1. 打印标题；
2. 读取用户输入的 `t`、`n`、`m`；
3. 创建 `TournamentOptimizerImproved` 实例；
4. 调用 `solve()` 进行求解；
5. 打印最终分组方案；
6. 捕获非法输入并提示。

注意输出时把队伍编号加了 1：`[x + 1 for x in group]`，这样用户看到的是 1~15，而不是程序内部的 0~14。

### `if __name__ == "__main__"`

这是 Python 的经典保护写法：

- 当文件被**直接运行**时，`__name__` 等于 `"__main__"`，于是执行 `main()`；
- 当文件被**作为模块导入**时，`__name__` 等于模块名，`main()` 不会自动执行。

这样既能当脚本用，又能被其他程序安全导入复用。

---

## 小结

这四个模块完成了整件事的“开头”和“结尾”：

1. **模块 1**：准备工具库；
2. **模块 2**：根据用户参数计算常量；
3. **模块 3**：快速排除明显无解的情况；
10. **模块 10**：接收输入、调用求解、输出结果。

中间模块（状态初始化、邻域移动、增量更新、模拟退火主循环）负责具体的搜索过程，将在后续文章中继续拆解。
