---
title: "LeetCode 5. 最长回文子串：动态规划与 C++ 基础"
date: 2026-06-24 21:00:00 +0800
categories:
  - 算法
tags:
  - C++
  - 动态规划
  - LeetCode
  - 回文串
  - string
  - vector
mathjax: true
excerpt: "用 C++ 动态规划解决 LeetCode 第 5 题《最长回文子串》，理解二维 DP 数组和 C++ 基础用法。"
---

> 题目来源：[LeetCode 5. 最长回文子串](https://leetcode.cn/problems/longest-palindromic-substring/)

## 题目描述

给你一个字符串 `s`，找到 `s` 中最长的回文子串。

### 什么是回文串

回文串就是**正着读和反着读都一样的字符串**。

例如：

- `"aba"` 是回文串
- `"abba"` 是回文串
- `"abc"` 不是回文串

### 例子

输入：`s = "babad"`

输出：`"bab"` 或 `"aba"`

因为 `"bab"` 和 `"aba"` 都是最长回文子串，返回任意一个即可。

---

## 核心思想：动态规划

这道题可以用**动态规划**解决。

### 回文串的性质

一个较长的回文串，如果去掉首尾两个字符，**内部仍然是一个回文串**。

例如：

```
"ababa" 是回文串
去掉首尾的 a，得到 "bab"，也是回文串
```

反过来也成立：

> 如果 `s[i] == s[j]`，并且 `s[i+1..j-1]` 是回文串，那么 `s[i..j]` 也是回文串。

这就是动态规划的**状态转移方程**。

### 二维 DP 数组

我们用一个二维数组 `dp[i][j]` 来表示：

```cpp
dp[i][j] = true   // s[i..j] 是回文串
dp[i][j] = false  // s[i..j] 不是回文串
```

为什么用二维？因为子串由**左边界 `i` 和右边界 `j`** 两个下标共同决定。

### 状态转移

```cpp
if (s[i] != s[j]) {
    dp[i][j] = false;           // 首尾不同，不是回文
} else {
    if (j - i < 3) {
        dp[i][j] = true;        // 长度 1 或 2 且首尾相同，是回文
    } else {
        dp[i][j] = dp[i + 1][j - 1];  // 看内部子串是否为回文
    }
}
```

### 填表顺序

子串长度从短到长依次计算：

1. 先算长度为 1 的子串：都是回文
2. 再算长度为 2 的子串
3. 然后长度 3、4、5……

因为长串依赖于它内部的短串，短串必须先算出来。

---

## 完整代码

```cpp
#include <iostream>
#include <string>
#include <vector>

using namespace std;

class Solution {
public:
    string longestPalindrome(string s) {
        int n = s.size();

        // 长度小于 2 的字符串本身就是最长回文子串
        if (n < 2) {
            return s;
        }

        int maxLen = 1;  // 当前找到的最长回文子串长度
        int begin = 0;   // 当前找到的最长回文子串起始位置

        // dp[i][j] 表示 s[i..j] 是否为回文串
        vector<vector<int>> dp(n, vector<int>(n));

        // 初始化：单个字符一定是回文
        for (int i = 0; i < n; i++) {
            dp[i][i] = true;
        }

        // 枚举子串长度，从 2 开始
        for (int L = 2; L <= n; L++) {
            // 枚举左边界 i
            for (int i = 0; i < n; i++) {
                // 由长度 L 和左边界 i 计算右边界 j
                int j = L + i - 1;

                // 右边界越界，退出当前循环
                if (j >= n) {
                    break;
                }

                // 判断 s[i..j] 是否为回文
                if (s[i] != s[j]) {
                    dp[i][j] = false;
                } else {
                    if (j - i < 3) {
                        dp[i][j] = true;
                    } else {
                        dp[i][j] = dp[i + 1][j - 1];
                    }
                }

                // 更新最长回文子串
                if (dp[i][j] && j - i + 1 > maxLen) {
                    maxLen = j - i + 1;
                    begin = i;
                }
            }
        }

        return s.substr(begin, maxLen);
    }
};

int main() {
    Solution sol;
    string s = "babad";
    string result = sol.longestPalindrome(s);

    cout << "输入：" << s << endl;
    cout << "最长回文子串：" << result << endl;

    return 0;
}
```

---

## 涉及的知识点

### 1. `string` 类型

```cpp
string s = "babad";
int n = s.size();
```

- `string` 是 C++ 标准库中的字符串类型
- `s.size()` 返回字符串长度
- `s[i]` 访问第 `i` 个字符
- `s.substr(begin, len)` 返回从 `begin` 开始、长度为 `len` 的子串

### 2. `vector<vector<int>>` 二维数组

```cpp
vector<vector<int>> dp(n, vector<int>(n));
```

- `vector` 是动态数组
- `vector<vector<int>>` 是二维数组，可以看作一个表格
- 这里创建了一个 `n × n` 的二维数组，初始值都是 `false`（即 0）

### 3. `class` 和成员函数

```cpp
class Solution {
public:
    string longestPalindrome(string s) {
        ...
    }
};
```

- `class` 定义一个类
- `public:` 后面的成员可以被外部访问
- `longestPalindrome` 是 `Solution` 类的一个成员函数（方法）

调用方式：

```cpp
Solution sol;                          // 创建对象
string result = sol.longestPalindrome(s);  // 调用方法
```

### 4. `using namespace std;`

```cpp
using namespace std;
```

这行代码让我们可以直接写 `cout`、`string`、`vector`，而不用写 `std::cout`、`std::string`、`std::vector`。

---

## 关键点总结

| 要点 | 说明 |
|------|------|
| 为什么用二维数组 | 子串由起点 `i` 和终点 `j` 两个下标确定 |
| 初始化 | 单个字符一定是回文，即 `dp[i][i] = true` |
| 状态转移 | 首尾相同且内部是回文，则整体是回文 |
| 填表顺序 | 按子串长度从小到大枚举 |
| 易错点 | 最后要返回的是子串，不是长度；注意处理最后的 `substr` |

---

## 复杂度分析

- **时间复杂度**：$O(n^2)$
  - 外层枚举长度 $L$：$O(n)$
  - 内层枚举左边界 $i$：$O(n)$
  - 总共 $O(n^2)$

- **空间复杂度**：$O(n^2)$
  - 需要一个 $n \times n$ 的二维数组

---

## 一句话总结

> 用 `dp[i][j]` 记录每个子串是否为回文，从小到大填表，最后找出最长的那个。这道题是理解**二维动态规划**和**区间 DP** 的经典入门题。
