---
layout: post
title: "虚拟化：进程入门"
date: 2026-07-09 00:00:00 +0800
categories: [OS, OSTEP]
tags: [OS, virtualization, process, OSTEP]
---

# 虚拟化：进程入门

学习 OSTEP（Operating Systems: Three Easy Pieces）第四章《The Abstraction: The Process》的一些笔记。前半部分整理书里关于进程抽象的核心概念；后半部分用 `process-run.py` 模拟器验证进程调度、I/O 与 CPU 的关系。

## 一、进程抽象的核心概念

### 1.1 进程就是运行中的程序

程序本身只是磁盘上的二进制文件，是“死”的。操作系统把程序加载到内存、分配资源、让它跑起来之后，才成为一个**进程（process）**。所以：

> 进程 = 运行中的程序 + 它的状态 + 占用的资源

### 1.2 时分共享（Time Sharing）与空分共享（Space Sharing）

操作系统要同时服务多个进程，本质上用了两种共享技术：

- **时分共享 Time Sharing**：同一资源（比如 CPU）轮流给不同进程使用一小段时间。因为切换得很快，用户感觉多个程序在“同时”运行。
- **空分共享 Space Sharing**：把资源按空间切分，不同进程各占一块。比如内存被划分为不同地址空间，磁盘被划分为不同文件。

CPU 调度用的是时分共享；内存、磁盘用的是空分共享。

### 1.3 进程的机器状态

进程运行时，操作系统需要保存它的“现场”，主要包括：

- **程序计数器 PC**：下一条要执行的指令地址
- **寄存器**：通用寄存器、栈指针等
- **地址空间**：代码段、堆、栈等

这些信息存放在 **进程控制块 PCB（Process Control Block）** 里。切换进程时，OS 保存旧进程的 PCB，加载新进程的 PCB，这就是上下文切换。

### 1.4 进程状态

一个进程在生命周期中会在几个状态之间转换：

- **RUNNING**：正在 CPU 上执行
- **READY**：准备好运行，等待 CPU 调度
- **BLOCKED**：等待某事件完成（通常是 I/O）

状态转换主要由以下事件触发：

- 被调度器选中 → RUNNING
- 时间片用完或被抢占 → READY
- 发起 I/O → BLOCKED
- I/O 完成 → READY

### 1.5 从程序到进程：加载

操作系统加载程序时大致做：

1. 从磁盘读取可执行文件
2. 分配地址空间（代码、静态数据、堆、栈）
3. 初始化栈和堆指针
4. 跳到入口点开始执行

这时，一个静态的程序就变成了动态的进程。

## 二、用 process-run.py 理解调度与 I/O/CPU 重叠

`process-run.py` 是 OSTEP 第四章的模拟器，用来观察不同进程组合和调度策略下的行为。

### 2.1 模拟器基本参数

- `-l X:Y`：生成一个进程，`X` 是指令数，`Y` 是每条指令为 CPU 指令的概率（%）
  - `5:100`：5 条全是 CPU
  - `1:0`：1 条是 I/O
  - `3:50`：3 条，每条 50% 概率是 CPU 或 I/O
- `-S`：进程切换策略
  - `SWITCH_ON_IO`：当前进程发起 I/O 时切换
  - `SWITCH_ON_END`：当前进程完全结束后才切换
- `-I`：I/O 完成后的处理策略
  - `IO_RUN_LATER`：I/O 完成后进程进入就绪队列，等调度
  - `IO_RUN_IMMEDIATE`：I/O 完成后立即运行该进程
- `-L`：一次 I/O 的等待时长（默认 5 个 tick）
- `-c`：直接给出答案（运行模拟）
- `-p`：打印统计信息

> 注意：`io` 指令本身占 1 tick，I/O 等待占 `-L` tick，`io_done` 占 1 tick。

### 2.2 关键实验与发现

#### 实验 1：进程顺序决定总时间

- `-l 4:100,1:0`：CPU 先跑完，再发起 I/O
  - 总时间 11，CPU 利用率 54.55%
- `-l 1:0,4:100`：I/O 先发起，CPU 在 I/O 等待期间运行
  - 总时间 7，CPU 利用率 85.71%

结论：**把 I/O 密集型进程放在前面，可以让 CPU 与 I/O 重叠，显著缩短总时间。**

#### 实验 2：切换策略 SWITCH_ON_END vs SWITCH_ON_IO

| 策略 | 总时间 | CPU 利用率 |
|---|---|---|
| SWITCH_ON_END | 11 | 54.55% |
| SWITCH_ON_IO | 7 | 85.71% |

`SWITCH_ON_END` 的问题：进程阻塞时 CPU 空转；`SWITCH_ON_IO` 则立刻切换到其它可运行进程。

#### 实验 3：I/O 完成后的行为

用 `-l 3:0,5:100,5:100,5:100 -S SWITCH_ON_IO` 测试：

| I/O 完成策略 | 总时间 | CPU 利用率 | I/O 利用率 |
|---|---|---|---|
| IO_RUN_LATER | 31 | 67.74% | 48.39% |
| IO_RUN_IMMEDIATE | 21 | 100.00% | 71.43% |

`IO_RUN_IMMEDIATE` 更好，因为 I/O 完成后立即运行原进程，能尽快发出下一次 I/O，保持 I/O 设备持续忙碌；同时 CPU 也没有空转。

#### 实验 4：随机进程

对 `-s N -l 3:50,3:50` 测试不同种子和策略，结果如下：

| seed | 策略 | 总时间 | CPU 利用率 | I/O 利用率 |
|---|---|---|---|---|
| 1 | SWITCH_ON_IO + LATER | 15 | 53.33% | 66.67% |
| 1 | SWITCH_ON_IO + IMMEDIATE | 15 | 53.33% | 66.67% |
| 1 | SWITCH_ON_END + LATER | 18 | 44.44% | 55.56% |
| 2 | SWITCH_ON_IO + LATER | 16 | 62.50% | 87.50% |
| 2 | SWITCH_ON_IO + IMMEDIATE | 16 | 62.50% | 87.50% |
| 2 | SWITCH_ON_END + LATER | 30 | 33.33% | 66.67% |
| 3 | SWITCH_ON_IO + LATER | 18 | 50.00% | 61.11% |
| 3 | SWITCH_ON_IO + IMMEDIATE | 17 | 52.94% | 64.71% |
| 3 | SWITCH_ON_END + LATER | 24 | 37.50% | 62.50% |

规律：

- `SWITCH_ON_IO` 普遍优于 `SWITCH_ON_END`，避免 CPU 空等。
- `IO_RUN_IMMEDIATE` 在存在多次 I/O 时通常能进一步缩短时间。
- 当 I/O 很少或只发生一次时，两种 I/O 策略差别不明显。

### 2.3 简化的上下文切换图景

把这些实验串起来，可以画出操作系统调度的核心逻辑：

1. CPU 上运行的进程发起 I/O。
2. 如果策略是 `SWITCH_ON_IO`，OS 保存该进程的 PCB，把它设为 BLOCKED，然后选择另一个 READY 进程运行。
3. I/O 设备在后台工作，CPU 同时执行别的进程。
4. I/O 完成后，进程变为 READY。
5. 根据 `IO_RUN_IMMEDIATE` 或 `IO_RUN_LATER`，OS 决定是立刻恢复它，还是让它排队等待调度。

这就是现代操作系统里 **CPU 虚拟化** 的最基本形式：通过时分共享，让多个进程“同时”推进，并通过合理调度隐藏 I/O 延迟。

### 2.4 “CPU4” 与 “4×CPU1”、“IO4” 与 “4×IO1” 的区别

讨论调度时，很容易把“连续执行 4 个 CPU tick”和“分成 4 条 CPU 指令”混为一谈。在 `process-run.py` 里，二者对 CPU 来说基本一样；但对 I/O 来说差别很大。

#### CPU：在模拟器中没有区别

`process-run.py` 里 `-P c4` 表示连续计算 4 个 tick，内部会被展开成 4 条 `DO_COMPUTE`；`-P c1,c1,c1,c1` 也是 4 条 `DO_COMPUTE`。输出都是：

```text
Time        PID: 0           CPU           IOs
  1        RUN:cpu             1
  2        RUN:cpu             1
  3        RUN:cpu             1
  4        RUN:cpu             1

Stats: Total Time 4
Stats: CPU Busy 4 (100.00%)
Stats: IO Busy  0 (0.00%)
```

因为模拟器把每条 CPU 指令都当作 1 tick，拆不拆分不影响调度机会。在实际 CPU 上，如果 4 个 tick 只是 4 条普通指令，本质上也一样：操作系统通过时钟中断在指令边界附近抢占，除非是一条需要多个周期才能完成的复杂指令，否则“CPU4”和“4×CPU1”对调度没有实质区别。

#### I/O：在模拟器和实际系统中都有区别

用 `-P i -L 4`（一次 I/O，等待 4 tick）和 `-P i,i,i,i -L 1`（四次 I/O，每次等待 1 tick）对比：

**一次 I/O 等待 4：**

```text
Time        PID: 0           CPU           IOs
  1         RUN:io             1
  2        BLOCKED                           1
  3        BLOCKED                           1
  4        BLOCKED                           1
  5        BLOCKED                           1
  6*   RUN:io_done             1

Stats: Total Time 6
Stats: CPU Busy 2 (33.33%)
Stats: IO Busy  4 (66.67%)
```

**四次 I/O，每次等待 1：**

```text
Time        PID: 0           CPU           IOs
  1         RUN:io             1
  2        BLOCKED                           1
  3*   RUN:io_done             1
  4         RUN:io             1
  5        BLOCKED                           1
  6*   RUN:io_done             1
  7         RUN:io             1
  8        BLOCKED                           1
  9*   RUN:io_done             1
 10         RUN:io             1
 11        BLOCKED                           1
 12*   RUN:io_done             1

Stats: Total Time 12
Stats: CPU Busy 8 (66.67%)
Stats: IO Busy  4 (33.33%)
```

区别：

1. **总时间不同**：一次 IO4 只需 1 次 `io` + 1 次 `io_done` 开销，共 2 CPU tick + 4 IO wait = 6；四次 IO1 有 4 次 `io` + 4 次 `io_done` 开销，共 8 CPU tick + 4 IO wait = 12。
2. **CPU 利用率不同**：拆成 4 次后 CPU 看起来更“忙”，但完成同样 IO 量的总效率更低。
3. **调度机会不同**：4×IO1 让进程多次在 RUNNING/BLOCKED 之间切换，给 OS 更多机会调度别的进程；一次 IO4 则让进程长时间阻塞，期间 CPU 必须找别的事做，否则空转。

实际系统中也类似：一次大 I/O 通常比多次小 I/O 更高效，因为每次 I/O 都有系统调用、设备初始化、DMA 设置等固定开销；但多次小 I/O 能提高进程的可调度性和响应性。所以真实系统里常有 I/O 合并（I/O merging）或块大小权衡：合并减少开销，拆分提高响应性。

## 三、小结

- 进程是操作系统对“运行中的程序”的抽象。
- 时分共享和空分共享是操作系统共享资源的两种基本手段。
- 进程状态（RUNNING / READY / BLOCKED）和 PCB 是实现调度和上下文切换的基础。
- 进程调度的核心目标之一是**让 CPU 和 I/O 尽量重叠**，减少空等。
- `process-run.py` 的实验清楚地展示了：进程顺序、切换策略、I/O 完成处理策略都会显著影响系统效率。

继续往下读 OSTEP 第五章，就会进入 `fork()`、`exec()`、`wait()` 这些实际的 UNIX 进程 API。
