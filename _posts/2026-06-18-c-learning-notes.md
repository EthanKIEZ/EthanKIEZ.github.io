---
title: "C 语言入门学习笔记：6 个实战项目与核心概念"
date: 2026-06-18 12:00:00 +0800
categories:
  - 学习笔记
tags:
  - C
  - CMake
  - CLion
  - 指针
  - 文件操作
toc: true
toc_label: "目录"
toc_icon: "list"
excerpt: "从零开始学习 C 语言，通过 6 个实战项目掌握基础语法、函数、数组、结构体、指针、链表和文件操作。"
---

## 前言

今天开始系统学习 C 语言，通过 **6 个实战项目** 从基础语法逐步深入到指针、链表和文件操作。这篇文章记录学习过程中的关键知识点，方便日后复习。

## 项目结构

使用 **CMake** 统一管理所有项目，每个项目一个子目录，结构清晰：

```text
untitled2/
├── CMakeLists.txt              # 根 CMakeLists，统一管理子项目
├── project01_calculator/       # 基础计算器
├── project02_guessing_game/    # 猜数字游戏
├── project03_student_manager/  # 学生成绩管理
├── project04_contact_book/     # 简易通讯录
├── project05_linked_list/      # 链表实现
└── project06_todo_list/        # 命令行 TODO 清单
```

根 `CMakeLists.txt` 使用 `add_subdirectory()` 管理子项目：

```cmake
cmake_minimum_required(VERSION 3.20)
project(c_learning C)

set(CMAKE_C_STANDARD 11)

add_subdirectory(project01_calculator)
add_subdirectory(project02_guessing_game)
add_subdirectory(project03_student_manager)
add_subdirectory(project04_contact_book)
add_subdirectory(project05_linked_list)
add_subdirectory(project06_todo_list)
```

## 项目 1：基础计算器

**知识点**：变量、输入输出、`if-else`、`switch`、循环

学会了使用 `scanf` 读取用户输入，用循环实现重复计算，并处理除数为 0 的情况。

```c
double num1, num2;
char operator;
scanf("%lf", &num1);
scanf(" %c", &operator);  // %c 前加空格，跳过残留换行符
scanf("%lf", &num2);
```

## 项目 2：猜数字游戏

**知识点**：随机数、循环、条件判断

使用 `srand()` 和 `rand()` 生成随机数，并通过循环控制猜测次数。

```c
srand((unsigned int)time(NULL));
secret_number = rand() % 100 + 1;  // 生成 1~100 的随机数
```

## 项目 3：学生成绩管理

**知识点**：数组、函数、循环、简单统计

练习了数组传参、函数声明与定义分离、求平均/最大/最小值。

```c
double calculate_average(const int scores[], int count) {
    int sum = 0;
    for (int i = 0; i < count; i++) {
        sum += scores[i];
    }
    return (double)sum / count;
}
```

## 项目 4：简易通讯录

**知识点**：结构体 `struct`、文件读写

第一次使用结构体把多个数据打包，并用 `fprintf`/`fscanf` 把数据持久化到文件。

```c
typedef struct {
    char name[NAME_LENGTH];
    char phone[PHONE_LENGTH];
    int age;
} Contact;
```

### 为什么用 fscanf 而不是 scanf？

| 函数 | 读取来源 |
|------|---------|
| `scanf(...)` | 键盘（`stdin`） |
| `fscanf(file, ...)` | 指定文件 |

加载联系人时显然要从文件读，所以用 `fscanf`。

## 项目 5：链表实现

**知识点**：指针、动态内存分配 `malloc/free`

理解了指针、结构体指针、`->` 运算符，以及如何通过 `malloc` 动态创建节点。

```c
typedef struct Node {
    int data;
    struct Node *next;
} Node;

Node *new_node = (Node *)malloc(sizeof(Node));
new_node->data = data;   // 等价于 (*new_node).data
new_node->next = NULL;
```

## 项目 6：命令行 TODO 清单

**知识点**：文件 I/O、结构体、字符串处理

综合运用了前面学到的内容，实现了添加、查看、完成、保存 TODO 的功能。

## 编译与链接

### 编译 vs 链接

- **编译**：把每个 `.c` 文件翻译成 `.o` 目标文件
- **链接**：把多个 `.o` 文件合并成一个可执行程序

```bash
gcc -c main.c -o main.o   # 编译
gcc -c math.c -o math.o   # 编译
gcc main.o math.o -o app  # 链接
```

### 为什么需要头文件？

头文件存放函数声明，多个 `.c` 文件共享同一份接口，避免重复声明。

```c
// mylib.h
void greet(const char *name);
int square(int n);

// main.c
#include "mylib.h"
```

## 核心概念速查

### 类型与占位符

| 类型 | 占位符 |
|------|--------|
| `int` | `%d` |
| `double` | `%lf` |
| `char` | `%c` |
| `char[]` | `%s` |
| `void*` | `%p` |

口诀：**`d` 整、`u` 无符、`f` 浮点、`c` 字符、`s` 字符串、`p` 指针。**

### 运算符优先级

口诀：**括单算，关等位，逻三赋逗**

即：括号/单目 → 算术 → 关系/相等 → 位运算 → 逻辑 → 三目 → 赋值 → 逗号。

### 函数位置关系

使用函数前必须先声明。要么把定义放 `main()` 前面，要么在前面加函数声明。

### 指针相关

- `Node *head = NULL;`：`head` 本身是空指针
- `*head`：访问 `head` 指向的节点
- `new_node->data`：等价于 `(*new_node).data`

## 总结

今天从零开始搭建了 C 语言学习环境，完成了 6 个由浅入深的项目，理解了编译链接、头文件、指针等核心概念。下一步计划继续练习动态内存管理和更复杂的数据结构。
