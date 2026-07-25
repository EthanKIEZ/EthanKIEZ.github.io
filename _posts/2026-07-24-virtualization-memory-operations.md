---
title: "虚拟化：内存操作"
date: 2026-07-24 00:00:00 +0800
categories:
  - 操作系统
  - OSTEP
tags:
  - OS
  - 虚拟化
  - OSTEP
  - 内存
  - malloc
  - free
  - 内存泄漏
  - API
  - CMake
excerpt: "整理内存操作与 CMake 构建的基础知识：malloc/free 的工作方式、size_t、多分配内存的用途、常见内存错误、不同语言的内存管理、API 概念，以及 CMake 基础用法。"
toc: true
toc_label: "目录"
toc_icon: "list"
toc_sticky: true
---

> **适合人群**：正在学习 C/C++ 内存管理、CMake 构建基础，想系统理解 `malloc`/`free`、内存错误、API 概念和 CMake 基础用法的读者。
{: .notice--info}

## 1. 堆内存分配基础

### 1.1 `malloc()` 的输入输出

`malloc()` 是 C 语言在**堆**上申请内存的函数：

```c
void *malloc(size_t size);
```

| 部分 | 含义 |
|------|------|
| 输入 `size` | 要申请多少**字节**的内存 |
| 返回值 `void *` | 申请成功后返回这块内存的**起始地址**；失败返回 `NULL` |

例如申请一个 `int` 的空间：

```c
int *p = (int *)malloc(sizeof(int));
*p = 42;
free(p);
```

`malloc` 只负责找一块足够大的空闲内存，**不会初始化内容**。它返回的是 `void *`，通常需要强制转换成具体类型。

### 1.2 `size_t` 是什么

`size_t` 是 C/C++ 中专门表示**大小、长度、字节数**的**无符号整数类型**。

常见出现场景：

```c
size_t n = sizeof(int);
size_t len = strlen("abc");
malloc(n * sizeof(int));
```

它由平台决定具体是多少位：32 位系统上通常 4 字节，64 位系统上通常 8 字节。目的是让表示"大小"的代码具有可移植性。

因为它是无符号的，要避免做 `i - 1 < 0` 这类有符号比较。

### 1.3 `free()` 怎么知道释放多大

`free` 不是简单地"从某个地址开始释放"，而是根据 `malloc` 返回的原始地址，往前找到**元数据头部**，读出 `size`，只释放这一块。

```text
堆上的实际内存块：
┌─────────────────┬──────────────────────┐
│  元数据（头部）  │   返回给你的可用内存   │
│  size=40 bytes  │   10 * sizeof(int)   │
├─────────────────┼──────────────────────┤
↑                 ↑
malloc 内部记录    x 指向这里
```

如果你把 `x` 移动了再 `free`，就会出错：

```c
int *x = malloc(10 * sizeof(int));
x++;        // x 不再指向 malloc 返回的位置
free(x);    // 错误！找不到正确的元数据
```

`free` 必须接收 `malloc` 返回的原始地址，不能传栈上变量、字符串常量或偏移后的指针。

### 1.4 `malloc` 多分配的内存还存什么

除了记录 `size`，多分配的内存还承担很多堆管理器内部工作：

| 用途 | 说明 |
|------|------|
| 内存对齐 | 把 chunk 大小凑到 8 或 16 字节倍数，提高访问效率 |
| `prev_size` | 记录前一个相邻 chunk 的大小 |
| 标志位 | 当前 chunk 是否在使用、是否由 `mmap` 分配等 |
| 空闲链表指针 | 空闲 chunk 用来串成双向链表，方便复用 |
| 边界标记 | 用于合并相邻的空闲块 |
| 安全 canary | 检测堆溢出（部分实现） |
| 调试信息 | 分配位置、调用栈等（调试版本中） |

分配时头部供管理器用，返回后面的区域给用户；释放后用户区域会被改用来维护空闲链表。

---

## 2. 字符串与内存状态

### 2.1 `strlen()` 的输入输出

```c
size_t strlen(const char *str);
```

| 部分 | 含义 |
|------|------|
| 输入 `str` | 指向以 `\0` 结尾的字符串的指针 |
| 返回值 | 字符串中字符个数，**不包括 `\0`** |

```c
char *src = "hello";
size_t len = strlen(src);  // len == 5
```

`strlen` 从给定地址开始遍历，遇到第一个 `\0` 停止。它不会检查数组边界，所以传一个没有 `\0` 的字符数组可能越界。

常见用法是配合 `malloc` 和 `strcpy`：

```c
char *dst = malloc(strlen(src) + 1);  // +1 给 '\0'
strcpy(dst, src);
free(dst);
```

### 2.2 `unallocated` 是什么意思

`unallocated` 表示**未分配的、空闲的**资源。在内存语境下，指一块内存现在没有被任何程序、变量或进程使用。

```c
int *p = malloc(100);  // 这 100 字节变成 allocated
free(p);               // 释放，又变成 unallocated
```

类比停车场：allocated 是已停车位，unallocated 是空车位。

---

## 3. 不同语言的内存管理

### 3.1 Python 不需要 `malloc`/`free`

Python 的内存管理是自动的：创建对象时解释器在堆里分配内存，没有引用指向时自动回收。

```python
x = 42
lst = [1, 2, 3]
s = "hello"
```

这些全在堆上，不需要手动释放。创建对象直接：

```python
obj = MyClass()
```

### 3.2 `MyClass()` 的输入输出

`MyClass()` 创建类实例：

| 部分 | 含义 |
|------|------|
| 输入 | 传给 `__init__` 初始化方法的参数 |
| 输出 | 堆上新创建的实例对象的**引用** |

```python
obj = MyClass("Alice", 20)
```

`obj` 不是对象本身，而是指向堆中对象的引用。`__init__` 只负责设置对象状态，不返回值。

---

## 4. 常见内存错误

下面是 7 种典型内存管理错误，每种都给出示例。

这些错误中，有些是**程序会立即崩溃**的（如 bad-free、double-free、严重越界），有些则是**程序能正常运行但留下隐患**的（如内存泄漏、轻微越界）。调试器通常只能帮你定位崩溃位置，无法自动发现泄漏或越界；而专业的内存检测工具可以在运行时拦截这些问题。

核心原则：**谁申请，谁释放；释放后不再用；只释放原始地址。**

### 4.1 忘记分配内存

```c
char *src;
strcpy(src, "hello");  // src 指向随机地址
```

**正确做法**：

```c
char *src = malloc(100);
strcpy(src, "hello");
free(src);
```

### 4.2 没有分配足够的内存

```c
char *src = malloc(5);   // 只有 5 字节
strcpy(src, "hello");    // "hello" 需要 6 字节（含 '\0'）
```

**正确做法**：

```c
char *src = malloc(strlen("hello") + 1);
strcpy(src, "hello");
free(src);
```

### 4.3 忘记初始化分配的内存

```c
int *arr = malloc(10 * sizeof(int));
printf("%d\n", arr[0]);  // 读到垃圾值
```

**正确做法**：

```c
// 方法 1：calloc 自动清零
int *arr = calloc(10, sizeof(int));

// 方法 2：malloc 后手动初始化
int *arr = malloc(10 * sizeof(int));
for (int i = 0; i < 10; i++) {
    arr[i] = 0;
}

free(arr);
```

### 4.4 忘记释放内存

```c
void leak() {
    int *p = malloc(100);
    // 使用 p...
}  // p 被销毁，但堆内存还在
```

这就是**内存泄漏**。正确做法：

```c
void no_leak() {
    int *p = malloc(100);
    // 使用 p...
    free(p);
}
```

### 4.5 在用完之前释放内存

```c
char *p = malloc(100);
strcpy(p, "hello");
free(p);
printf("%s\n", p);  // use-after-free
```

正确顺序：先使用，再释放，释放后置空：

```c
char *p = malloc(100);
strcpy(p, "hello");
printf("%s\n", p);
free(p);
p = NULL;
```

### 4.6 反复释放内存

```c
int *p = malloc(100);
free(p);
free(p);  // double free
```

**正确做法**：

```c
free(p);
p = NULL;  // 对 NULL 调用 free 是安全的
```

### 4.7 错误地调用 `free()`

```c
int x;
free(&x);  // 错误：x 在栈上

char *p = "hello";
free(p);   // 错误：字符串常量在只读数据段

int *q = malloc(100);
q++;
free(q);   // 错误：q 已不是 malloc 返回的原始地址
```

**正确做法**：只 `free` `malloc`/`calloc`/`realloc` 返回的原始指针。

### 4.8 `free(NULL)` 是安全的

C 标准规定：`free(NULL)` 不执行任何操作，是安全的。

```c
int *p = NULL;
free(p);  // 合法，什么都不做
```

这带来一个实用好处：释放指针后置空，即使误写第二次 `free` 也不会崩溃：

```c
free(p);
p = NULL;
// 即使后面误写 free(p)，也是安全的
```

但要注意，未初始化的指针不是 `NULL`：

```c
int *p;     // 随机值
free(p);    // 危险！不是 free(NULL)
```

---

## 5. 什么是 API

**API** 是 Application Programming Interface（应用程序编程接口）的缩写。凡是程序与程序、程序与操作系统、程序与库之间约定好的调用接口，都可以称为 API。

常见例子：

| 类型 | 例子 |
|------|------|
| 操作系统 API | `open()`、`read()`、`write()`、`fork()` |
| 标准库 API | `malloc()`、`strlen()`、`printf()` |
| 第三方库 API | `curl_easy_init()`、`requests.get()` |
| 框架 API | Flask 的 `app.route()`、React 的 `useState()` |
| Web API | 通过 HTTP 调用的服务接口 |

判断标准：是否是某层系统/库/服务对外暴露的、有文档约定的接口，供其他程序使用。

---

## 6. CMake 基础

### 6.1 CMake 是什么

CMake 是一个**跨平台的构建系统生成工具**。它本身不直接编译代码，而是根据 `CMakeLists.txt` 生成对应平台的构建文件（如 Linux/macOS 的 `Makefile`、Windows 的 Visual Studio 工程），再由 `make`、`ninja` 或 IDE 真正编译。

### 6.2 基本流程

```bash
# 第一步：生成构建文件
cmake -S . -B build

# 第二步：编译所有程序
cmake --build build
```

| 命令 | 含义 |
|------|------|
| `cmake -S .` | `-S` 指定**源代码目录**（Source），`.` 表示当前目录 |
| `-B build` | `-B` 指定**构建输出目录**（Build），这里叫 `build` |
| `cmake --build build` | 调用 `build/` 里的构建文件，真正执行编译 |

### 6.3 `set()`：定义变量

`set()` 用来定义变量，方便复用：

```cmake
set(ASAN_FLAGS -fsanitize=address -g -O0 -fno-omit-frame-pointer)
```

后面用 `${ASAN_FLAGS}` 引用：

```cmake
target_compile_options(q01_null PRIVATE ${ASAN_FLAGS})
```

### 6.4 `foreach` 循环

```cmake
foreach(PROG q01_null q04_leak q05_overflow q06_use_after_free q07_bad_free q08_vector)
    add_executable(${PROG} ${PROG}.c)
    target_compile_options(${PROG} PRIVATE ${ASAN_FLAGS})
    target_link_options(${PROG} PRIVATE ${ASAN_FLAGS})
endforeach()
```

`PROG` 是循环变量，每轮取列表中的一个程序名。CMake 没有 C 语言的花括号 `{}`，所以用 `foreach ... endforeach()` 成对关键字表示代码块。

### 6.5 `PRIVATE` 是什么

`PRIVATE` 控制选项/库的可见范围：

| 关键字 | 含义 |
|--------|------|
| `PRIVATE` | 只给当前目标用，不传给依赖它的目标 |
| `PUBLIC` | 当前目标用，也传给依赖它的目标 |
| `INTERFACE` | 当前目标不用，只传给依赖它的目标 |

在独立可执行文件中用 `PRIVATE` 即可。

### 6.6 示例：把 `null.c` 编译成带 ASan 的可执行文件

假设 `null.c` 内容如下：

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    int *p = NULL;
    free(p);  // free(NULL) 是安全的
    printf("free(NULL) completed safely\n");
    return 0;
}
```

#### 命令行编译

```bash
clang -fsanitize=address -g -O0 -fno-omit-frame-pointer null.c -o null
```

| 参数 | 作用 |
|------|------|
| `-fsanitize=address` | 启用 ASan |
| `-g` | 保留调试信息 |
| `-O0` | 关闭优化 |
| `-fno-omit-frame-pointer` | 保留帧指针，调用栈更准确 |
| `null.c` | 源文件 |
| `-o null` | 输出可执行文件名为 `null` |

#### 运行

```bash
./null
```

输出：

```text
free(NULL) completed safely
```

没有 `ERROR: AddressSanitizer:` 报错，表示 `free(NULL)` 安全。

---

## 7. 总结

- `malloc` 申请堆内存，返回起始地址；`free` 根据头部元数据释放对应块
- `size_t` 是无符号整数，专门表示大小和长度
- `malloc` 多分配的内存用于对齐、链表指针、边界标记、标志位和安全检测
- Python 自动管理内存，`MyClass()` 返回实例对象的引用
- 7 种常见内存错误：忘记分配、分配不够、忘记初始化、忘记释放、用完前释放、反复释放、错误调用 `free`
- `free(NULL)` 是安全的，但 `free` 必须接收 `malloc`/`calloc`/`realloc` 返回的原始地址
- API 是约定好的调用接口
- CMake 是构建系统生成工具，`set()` 定义变量，`foreach` 批量处理，`PRIVATE` 控制可见性

> 掌握这些内存操作与构建工具细节，是完成 OSTEP 内存章节和写出稳定 C/C++ 程序的基础。
{: .notice--primary}
