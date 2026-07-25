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
  - CMake
excerpt: "整理内存操作与 CMake 构建的基础知识：malloc/free 工作方式、常见内存错误、calloc/alloca 区别、进程地址空间，以及如何用 CMake 批量管理带 ASan 的练习程序。"
toc: true
toc_label: "目录"
toc_icon: "list"
toc_sticky: true
---

> **适合人群**：正在学习 OSTEP 内存操作章节，想系统理解 `malloc`/`free`、常见内存错误、进程地址空间，以及 CMake 基础用法的读者。
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

```c
int *p = (int *)malloc(sizeof(int));
*p = 42;
free(p);
```

`malloc` 只负责找一块足够大的空闲内存，**不会初始化内容**，返回 `void *` 通常需要强制转换。

### 1.2 `size_t` 是什么

`size_t` 是 C/C++ 中专门表示**大小、长度、字节数**的**无符号整数类型**。

```c
size_t n = sizeof(int);
size_t len = strlen("abc");
malloc(n * sizeof(int));
```

它由平台决定位数：32 位系统通常 4 字节，64 位系统通常 8 字节。因为它无符号，要避免 `i - 1 < 0` 这类有符号比较。

### 1.3 `free()` 怎么知道释放多大

`free` 根据 `malloc` 返回的原始地址，往前找到**元数据头部**，读出 `size`，只释放这一块。

```text
堆上的实际内存块：
┌─────────────────┬──────────────────────┐
│  元数据（头部）  │   返回给你的可用内存   │
│  size=40 bytes  │   10 * sizeof(int)   │
├─────────────────┼──────────────────────┤
↑                 ↑
malloc 内部记录    x 指向这里
```

如果移动指针后再 `free` 会出错：

```c
int *x = malloc(10 * sizeof(int));
x++;
free(x);  // 错误！找不到正确的元数据
```

`free` 必须接收 `malloc`/`calloc`/`realloc` 返回的原始地址。

### 1.4 `malloc` 多分配的内存还存什么

除了记录 `size`，多分配的内存还承担堆管理器内部工作：

| 用途 | 说明 |
|------|------|
| 内存对齐 | 把 chunk 大小凑到 8 或 16 字节倍数 |
| `prev_size` | 记录前一个相邻 chunk 的大小 |
| 标志位 | 当前 chunk 是否在使用、是否由 `mmap` 分配等 |
| 空闲链表指针 | 空闲 chunk 串成双向链表，方便复用 |
| 边界标记 | 用于合并相邻的空闲块 |
| 安全 canary | 检测堆溢出（部分实现） |
| 调试信息 | 分配位置、调用栈等（调试版本） |

### 1.5 `calloc()` 与 `alloca()` 的区别

| | `calloc` | `alloca` |
|--|---------|---------|
| 分配位置 | **堆** | **栈** |
| 初始化 | 自动清零为 0 | 不初始化 |
| 释放方式 | 必须手动 `free` | 函数返回时自动释放 |
| 标准性 | 标准 C | 非标准 C，是扩展 |
| 可移植性 | 好 | 差（Windows 上叫 `_alloca`） |

```c
// calloc：在堆上分配 10 个 int，全部初始化为 0
int *arr = calloc(10, sizeof(int));
free(arr);

// alloca：在栈上临时分配
void foo(int n) {
    int *tmp = alloca(n * sizeof(int));
    // 函数返回时自动释放
}
```

`alloca` 空间有限、容易栈溢出，现代代码中不推荐常规使用。

### 1.6 `free(NULL)` 是安全的

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

但未初始化的指针不是 `NULL`：

```c
int *p;     // 随机值
free(p);    // 危险！不是 free(NULL)
```

---

## 2. 进程地址空间

一个进程的地址空间通常包括：

| 区域 | 内容 |
|------|------|
| 代码段（Text） | 程序指令 |
| 数据段（Data） | 已初始化的全局/静态变量 |
| BSS 段 | 未初始化的全局/静态变量 |
| 堆（Heap） | 动态分配的内存 |
| 栈（Stack） | 局部变量、函数调用帧 |
| 只读数据段（ROData） | 字符串常量、`const` 全局变量 |
| 内存映射区域 | 共享库、文件映射 |
| 内核空间 | 操作系统内核，用户程序不可直接访问 |

例如：

```c
int global_initialized = 10;    // 数据段
int global_uninitialized;        // BSS 段
const char *msg = "hello";       // msg 在数据段，"hello" 在只读数据段

int main(int argc, char *argv[]) {
    int local;                   // 栈
    int *p = malloc(100);        // 堆
    free(p);
    return 0;
}
```

---

## 3. 常见内存错误

下面是 7 种典型内存管理错误。核心原则：**谁申请，谁释放；释放后不再用；只释放原始地址。**

### 3.1 忘记分配内存

```c
char *src;
strcpy(src, "hello");  // src 指向随机地址
```

正确做法：先 `malloc` 再使用。

### 3.2 没有分配足够的内存

```c
char *src = malloc(5);   // 只有 5 字节
strcpy(src, "hello");    // "hello" 需要 6 字节（含 '\0'）
```

正确做法：`malloc(strlen(src) + 1)`。

### 3.3 忘记初始化分配的内存

```c
int *arr = malloc(10 * sizeof(int));
printf("%d\n", arr[0]);  // 读到垃圾值
```

正确做法：用 `calloc` 或手动初始化。

### 3.4 忘记释放内存

```c
void leak() {
    int *p = malloc(100);
    // 使用 p...
}  // p 被销毁，但堆内存还在
```

这就是**内存泄漏**。程序能正常运行，但内存无法回收。

### 3.5 在用完之前释放内存

```c
char *p = malloc(100);
strcpy(p, "hello");
free(p);
printf("%s\n", p);  // use-after-free
```

正确顺序：先使用，再释放，释放后置空。

### 3.6 反复释放内存

```c
int *p = malloc(100);
free(p);
free(p);  // double free
```

正确做法：`free(p); p = NULL;`。

### 3.7 错误地调用 `free()`

```c
int x;
free(&x);  // 错误：x 在栈上

char *p = "hello";
free(p);   // 错误：字符串常量在只读数据段

int *q = malloc(100);
q++;
free(q);   // 错误：q 已不是 malloc 返回的原始地址
```

只 `free` `malloc`/`calloc`/`realloc` 返回的原始指针。

---

## 4. CMake 基础

### 4.1 CMake 是什么

CMake 是一个**跨平台的构建系统生成工具**。它本身不直接编译代码，而是根据 `CMakeLists.txt` 生成对应平台的构建文件（如 Linux/macOS 的 `Makefile`、Windows 的 Visual Studio 工程），再由 `make`、`ninja` 或 IDE 真正编译。

### 4.2 基本流程

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

### 4.3 `set()`：定义变量

`set()` 用来定义变量，方便复用：

```cmake
set(ASAN_FLAGS -fsanitize=address -g -O0 -fno-omit-frame-pointer)
```

后面用 `${ASAN_FLAGS}` 引用：

```cmake
target_compile_options(q01_null PRIVATE ${ASAN_FLAGS})
```

### 4.4 `macro()` 与 `set()` 的区别

| | `set()` | `macro()` |
|--|---------|-----------|
| 作用 | 定义变量，存数据 | 定义宏，封装命令 |
| 调用 | `${变量名}` | `宏名(参数)` |

```cmake
macro(build_asan_prog name)
    add_executable(${name} ${name}.c)
    target_compile_options(${name} PRIVATE -fsanitize=address -g -O0)
    target_link_options(${name} PRIVATE -fsanitize=address)
endmacro()

build_asan_prog(q01_null)
build_asan_prog(q04_leak)
```

### 4.5 `foreach` 循环

```cmake
foreach(PROG q01_null q04_leak q05_overflow q06_use_after_free q07_bad_free q08_vector)
    add_executable(${PROG} ${PROG}.c)
    target_compile_options(${PROG} PRIVATE ${ASAN_FLAGS})
    target_link_options(${PROG} PRIVATE ${ASAN_FLAGS})
endforeach()
```

`PROG` 是循环变量，每轮取列表中的一个程序名。CMake 没有 C 语言的花括号 `{}`，所以用 `foreach ... endforeach()` 成对关键字表示代码块。

### 4.6 `PRIVATE` 是什么

`PRIVATE` 控制选项/库的可见范围：

| 关键字 | 含义 |
|--------|------|
| `PRIVATE` | 只给当前目标用，不传给依赖它的目标 |
| `PUBLIC` | 当前目标用，也传给依赖它的目标 |
| `INTERFACE` | 当前目标不用，只传给依赖它的目标 |

在独立可执行文件中用 `PRIVATE` 即可。

### 4.7 完整示例

以下 `CMakeLists.txt` 批量生成 6 个带 ASan 的练习程序：

```cmake
set(ASAN_FLAGS -fsanitize=address -g -O0 -fno-omit-frame-pointer)

foreach(PROG q01_null q04_leak q05_overflow q06_use_after_free q07_bad_free q08_vector)
    add_executable(${PROG} ${PROG}.c)
    target_compile_options(${PROG} PRIVATE ${ASAN_FLAGS})
    target_link_options(${PROG} PRIVATE ${ASAN_FLAGS})
endforeach()
```

写了这段配置后，你不需要手动敲 `clang -fsanitize=address ...` 编译每个文件，直接 `cmake --build build` 即可。

---

## 5. OSTEP 练习程序对应表

| 程序 | 对应问题 | 演示的错误/现象 |
|------|---------|----------------|
| `q01_null` | 问题 1/3 | `free(NULL)` 安全，不报错 |
| `q04_leak` | 问题 4 | `malloc` 后忘记 `free`，内存泄漏 |
| `q05_overflow` | 问题 5 | `data[100]` 堆缓冲区越界 |
| `q06_use_after_free` | 问题 6 | 释放数组后继续访问 |
| `q07_bad_free` | 问题 7 | 向 `free` 传数组中间地址 |
| `q08_vector` | 问题 8 | 用 `realloc` 实现动态数组 |

---

## 6. 总结

- `malloc` 申请堆内存，返回起始地址；`free` 根据头部元数据释放对应块
- `size_t` 是无符号整数，专门表示大小和长度
- `malloc` 多分配的内存用于对齐、链表指针、边界标记、标志位和安全检测
- `calloc` 在堆上分配并清零；`alloca` 在栈上临时分配，不标准且风险高
- `free(NULL)` 是安全的，但 `free` 必须接收 `malloc`/`calloc`/`realloc` 返回的原始地址
- 进程地址空间还包括数据段、BSS、只读数据段、内存映射区域和内核空间
- 7 种常见内存错误：忘记分配、分配不够、忘记初始化、忘记释放、用完前释放、反复释放、错误调用 `free`
- CMake 是构建系统生成工具，`set()` 定义变量，`macro()` 封装命令，`foreach` 批量处理，`PRIVATE` 控制可见性

> 掌握这些内存操作与构建工具细节，是完成 OSTEP 内存章节和写出稳定 C/C++ 程序的基础。
{: .notice--primary}
