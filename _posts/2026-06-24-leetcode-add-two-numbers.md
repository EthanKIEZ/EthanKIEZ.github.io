---
title: "LeetCode 2. 两数相加：链表模拟竖式加法"
date: 2026-06-24 20:00:00 +0800
categories:
  - 算法
tags:
  - C++
  - 链表
  - LeetCode
  - 数据结构
mathjax: true
excerpt: "用 C++ 实现 LeetCode 第 2 题《两数相加》，理解逆序链表和模拟竖式加法的思路。"
---

> 题目来源：[LeetCode 2. 两数相加](https://leetcode.cn/problems/add-two-numbers/)

## 题目描述

给你两个**非空**的链表，表示两个非负整数。它们每位数字都是按照**逆序**的方式存储的，并且每个节点只能存储一位数字。

请你将两个数相加，并以相同形式返回一个表示和的链表。

你可以假设除了数字 0 之外，这两个数都不会以 0 开头。

### 例子

输入：

```
l1 = [2, 4, 3]   // 表示 342
l2 = [5, 6, 4]   // 表示 465
```

输出：

```
[7, 0, 8]        // 表示 807
```

因为 $342 + 465 = 807$。

---

## 什么是链表

链表是一种线性数据结构，每个节点存两个东西：

- **数据**：这里是一个数字（0-9）
- **下一个节点的地址**：指向后面那个节点

```
2 -> 4 -> 3 -> nullptr
```

在 C++ 里通常这样定义：

```cpp
struct ListNode {
    int val;           // 当前节点存储的数字
    ListNode *next;    // 指向下一个节点的指针
    ListNode(int x) : val(x), next(nullptr) {}  // 构造函数
};
```

---

## 解题思路：模拟竖式加法

这道题最自然的想法和小学做的竖式加法一模一样：

```
  2 -> 4 -> 3
+ 5 -> 6 -> 4
-------------
  7 -> 0 -> 8
```

从低位到高位逐位相加，同时处理进位。

因为链表本身就是逆序存储的，所以链表头就是最低位，我们可以直接从头开始遍历，不需要反转。

### 算法步骤

1. 同时遍历两个链表
2. 取出当前位的两个数字，如果某个链表已经走完，就当作 0
3. 计算和：`sum = n1 + n2 + carry`
4. 当前位结果：`sum % 10`
5. 新的进位：`sum / 10`
6. 创建新节点，接到结果链表末尾
7. 遍历结束后，如果还有进位，再追加一个节点

---

## 完整代码

```cpp
#include <iostream>

// 链表节点定义
struct ListNode {
    int val;                // 节点存储的数字
    ListNode *next;         // 指向下一个节点的指针
    ListNode(int x) : val(x), next(nullptr) {}  // 构造函数
};

// Solution 类：题目解法
class Solution {
public:
    // 接收两个链表头节点，返回相加后的链表头节点
    ListNode* addTwoNumbers(ListNode* l1, ListNode* l2) {
        // head：结果链表的头节点
        // tail：结果链表的尾节点，方便在末尾追加
        ListNode *head = nullptr, *tail = nullptr;

        // carry：进位值，初始为 0
        int carry = 0;

        // 同时遍历两个链表，只要有一个没走完就继续
        while (l1 || l2) {
            // 如果 l1 当前节点存在就取它的值，否则当作 0
            int n1 = l1 ? l1->val : 0;

            // 如果 l2 当前节点存在就取它的值，否则当作 0
            int n2 = l2 ? l2->val : 0;

            // 当前位的和 = 两个数字 + 上一位的进位
            int sum = n1 + n2 + carry;

            // 创建新节点，值为 sum 的个位
            if (!head) {
                // 结果链表为空时，第一个节点同时作为 head 和 tail
                head = tail = new ListNode(sum % 10);
            } else {
                // 把新节点接到 tail 后面
                tail->next = new ListNode(sum % 10);
                tail = tail->next;  // tail 后移
            }

            // 更新进位：sum 的十位
            carry = sum / 10;

            // 两个链表指针都向后移动
            if (l1) l1 = l1->next;
            if (l2) l2 = l2->next;
        }

        // 如果最后还有进位，需要再追加一个节点
        if (carry > 0) {
            tail->next = new ListNode(carry);
        }

        return head;
    }
};

// 辅助函数：创建链表 [a, b, c, ...]
ListNode* createList(int arr[], int n) {
    if (n == 0) return nullptr;
    ListNode *head = new ListNode(arr[0]);
    ListNode *cur = head;
    for (int i = 1; i < n; i++) {
        cur->next = new ListNode(arr[i]);
        cur = cur->next;
    }
    return head;
}

// 辅助函数：打印链表
void printList(ListNode *head) {
    while (head) {
        std::cout << head->val;
        if (head->next) std::cout << " -> ";
        head = head->next;
    }
    std::cout << std::endl;
}

int main() {
    // 构造 l1 = [2, 4, 3]，表示 342
    int a1[] = {2, 4, 3};
    ListNode *l1 = createList(a1, 3);

    // 构造 l2 = [5, 6, 4]，表示 465
    int a2[] = {5, 6, 4};
    ListNode *l2 = createList(a2, 3);

    Solution s;
    ListNode *result = s.addTwoNumbers(l1, l2);

    std::cout << "结果链表：";
    printList(result);  // 输出：7 -> 0 -> 8

    return 0;
}
```

---

## 关键细节

### 1. 链表长度不同怎么办

```cpp
int n1 = l1 ? l1->val : 0;
int n2 = l2 ? l2->val : 0;
```

如果某个链表已经走完了，当前位就当作 0。这样短的链表相当于后面补了无数个 0。

### 2. 为什么用 `sum % 10` 和 `sum / 10`

```cpp
carry = sum / 10;          // 十位，作为下一位的进位
new ListNode(sum % 10);    // 个位，作为当前位结果
```

例如 `sum = 13`：
- `13 % 10 = 3`，当前位写 3
- `13 / 10 = 1`，进位 1

### 3. 最后的进位别忘记

```cpp
if (carry > 0) {
    tail->next = new ListNode(carry);
}
```

比如 `5 + 5 = 10`，遍历完两个链表后 `carry = 1`，必须再追加一个节点，否则结果就少了一位。

### 4. 为什么链表头是低位

因为加法是从低位开始的。链表头就是最低位，正好方便我们从前往后遍历。

如果题目改成高位在前，就需要先反转链表，或者用栈来辅助。

---

## 复杂度分析

- **时间复杂度**：$O(\max(m, n))$，其中 $m$、$n$ 分别是两个链表的长度。最多遍历较长链表的长度加一次进位处理。
- **空间复杂度**：$O(\max(m, n))$，需要创建一个新的链表来存结果。

---

## 总结

| 要点 | 内容 |
|------|------|
| 核心思想 | 模拟竖式加法，从低位到高位逐位相加 |
| 关键点 | 处理进位、链表长度不同、最后的进位 |
| 为什么链表头在前 | 链表头是低位，符合加法从低位开始的顺序 |
| 易错点 | 遍历结束后忘记处理剩余进位 |

这道题是链表入门题，也是很多链表题的基础。理解它之后，类似的数字相加、大数运算问题都会比较顺手。
