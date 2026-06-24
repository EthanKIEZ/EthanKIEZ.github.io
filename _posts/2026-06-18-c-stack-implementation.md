---
title: "C 语言栈的实现：从零写一个简单的顺序栈"
date: 2026-06-18 18:00:00 +0800
categories:
  - C语言
tags:
  - C
  - 栈
  - 数据结构
  - 指针
---

> 适合人群：学过 C 语言基础（数组、结构体、指针、函数），想自己动手实现数据结构的新手。

## 什么是栈

栈（Stack）是一种**后进先出**（LIFO，Last In First Out）的数据结构。

想象一摞盘子：

- 放盘子时只能放在最上面
- 拿盘子时只能拿最上面的
- 最后放上去的盘子，最先被拿走

这就是栈。

## 栈的两个核心操作

| 操作 | 作用 | 形象说法 |
|------|------|---------|
| push | 把元素放到栈顶 | 放盘子 |
| pop  | 把栈顶元素拿走 | 拿盘子 |

## 完整代码

下面是用 C 语言数组实现的顺序栈：

```c
#include <stdio.h>
#include <stdbool.h>

#define STACK_CAPACITY 100

typedef struct {
    int items[STACK_CAPACITY];  // 存放栈元素的数组
    int top;                     // 栈顶元素的下标
} Stack;

// 初始化栈
void stack_init(Stack *s) {
    s->top = -1;
}

// 判断栈是否为空
bool stack_is_empty(const Stack *s) {
    return s->top == -1;
}

// 判断栈是否已满
bool stack_is_full(const Stack *s) {
    return s->top == STACK_CAPACITY - 1;
}

// 入栈
bool stack_push(Stack *s, int value) {
    if (stack_is_full(s)) {
        printf("栈已满，无法入栈\n");
        return false;
    }
    s->items[++(s->top)] = value;
    return true;
}

// 出栈
bool stack_pop(Stack *s, int *value) {
    if (stack_is_empty(s)) {
        printf("栈为空，无法出栈\n");
        return false;
    }
    *value = s->items[(s->top)--];
    return true;
}

// 查看栈顶元素
bool stack_peek(const Stack *s, int *value) {
    if (stack_is_empty(s)) {
        printf("栈为空\n");
        return false;
    }
    *value = s->items[s->top];
    return true;
}

// 打印栈
void stack_print(const Stack *s) {
    printf("Stack [bottom -> top]: [");
    for (int i = 0; i <= s->top; i++) {
        printf("%d", s->items[i]);
        if (i < s->top) {
            printf(", ");
        }
    }
    printf("]\n");
}

int main(void) {
    Stack s;
    stack_init(&s);

    stack_push(&s, 10);
    stack_push(&s, 20);
    stack_push(&s, 30);
    stack_print(&s);  // [10, 20, 30]

    int value;
    stack_pop(&s, &value);
    printf("出栈：%d\n", value);  // 30
    stack_print(&s);              // [10, 20]

    return 0;
}
```

## 逐段讲解

### 1. 结构体设计

```c
typedef struct {
    int items[STACK_CAPACITY];
    int top;
} Stack;
```

- `items`：真正存放数据的数组
- `top`：栈顶元素在数组中的下标

这里 `top` 不是地址，也不是元素个数，而是**栈顶元素的下标**。

### 2. 初始化：`top = -1`

```c
void stack_init(Stack *s) {
    s->top = -1;
}
```

为什么用 `-1` 表示空栈？

因为数组下标从 `0` 开始。如果空栈时 `top = 0`，那就无法区分：

- 空栈（没有元素）
- 有一个元素在 `items[0]`

用 `-1` 表示空栈，逻辑非常自然：

| 状态 | top 值 | 含义 |
|------|--------|------|
| 空栈 | -1 | 没有元素 |
| 一个元素 | 0 | items[0] |
| 两个元素 | 1 | items[1] 是栈顶 |

### 3. 入栈：`s->items[++(s->top)] = value`

```c
s->items[++(s->top)] = value;
```

这行代码做了两件事：

1. `++(s->top)`：先把 `top` 加 1
2. `s->items[新的 top] = value`：把值放到新的栈顶位置

注意是**前置++**：先加后用。

如果原来是 `-1`，加 1 变成 `0`，元素放到 `items[0]`。  
如果原来是 `0`，加 1 变成 `1`，元素放到 `items[1]`。

### 4. 出栈：`*value = s->items[(s->top)--]`

```c
*value = s->items[(s->top)--];
```

这行代码也做了两件事：

1. 先取出 `items[top]` 的值
2. 再把 `top` 减 1

注意是**后置--**：先用后减。

### 5. 传指针的原因

```c
void stack_init(Stack *s);
bool stack_push(Stack *s, int value);
```

为什么参数是 `Stack *s` 而不是 `Stack s`？

因为 C 语言函数参数是**值传递**。如果写成 `Stack s`，函数里修改的只是 `s` 的副本，真正的栈不会被改变。

传指针才能让函数修改调用者手里的那个栈。

## 常见盲点与误区

### 误区 1：空栈时 top 设为 0

```c
// 错误示范
void stack_init(Stack *s) {
    s->top = 0;  // 这样会让栈看起来已经有一个元素
}
```

如果 `top = 0` 表示空栈，那么 `items[0]` 到底是有数据还是没有数据？会和“有一个元素”冲突。

### 误区 2：`++top` 和 `top++` 写反

```c
// 入栈时不能用后置++
s->items[(s->top)++] = value;  // 错误！
```

这样写会先使用旧的 `top` 作为下标，然后再加 1。结果会导致：

- 新元素写到了错误的位置
- `top` 指向的位置和实际写入位置不一致

### 误区 3：忘记检查栈满/栈空

```c
// 危险！没有检查栈满
bool stack_push(Stack *s, int value) {
    s->items[++(s->top)] = value;  // top 可能越界
    return true;
}
```

如果一直 push，`top` 会超过数组最大下标，造成**数组越界**，程序行为不可预测。

### 误区 4：把数据结构的栈和程序运行时的栈搞混

C 语言里有两个“栈”：

1. **数据结构栈**：我们今天写的这个，程序员自己管理
2. **运行时栈**：程序运行时由 CPU 和编译器自动维护，保存函数调用的返回地址、局部变量等

它们都遵循 LIFO，但用途完全不同。详情可以参考我之前的文章《C 语言栈和汇编运行时栈有什么不同》。

### 误区 5：出栈后不保存返回值

```c
// 错误示范：丢了弹出的值
stack_pop(&s, &value);
// 然后又不使用 value
```

出栈操作通常是为了使用那个值，如果不需要，可以专门写一个只清空栈顶而不返回值的函数。

### 误区 6：栈顶元素下标和元素个数搞混

```c
// 元素个数不是 top，而是 top + 1
int count = s->top;      // 错误！
int count = s->top + 1;  // 正确
```

因为 `top` 是下标，从 0 开始，所以元素个数要加 1。

## 运行结果

```bash
gcc -std=c11 stack.c -o stack
./stack
```

输出：

```
Stack [bottom -> top]: [10, 20, 30]
出栈：30
Stack [bottom -> top]: [10, 20]
```

## 课后练习

1. 给栈增加一个 `stack_size()` 函数，返回当前元素个数。
2. 把固定大小的数组改成用 `malloc` 动态分配内存。
3. 实现一个判断括号是否匹配的函数 `is_balanced(char *str)`，用栈来判断。
4. 尝试用链表实现栈，比较数组和链表两种实现的优缺点。

## 总结

实现一个顺序栈并不难，核心就是维护好一个 `top` 指针：

- 空栈时 `top = -1`
- push 时用 `++top` 先移动再写入
- pop 时用 `top--` 先取出再移动
- 一定要记得检查栈满和栈空

最容易出错的地方不是算法本身，而是**下标管理和++/--的位置**。多写几遍，熟练了就不会错了。
