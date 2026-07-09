---
layout: post
title: "虚拟化：进程入门"
date: 2026-07-09 00:00:00 +0800
categories:
  - 操作系统
  - OSTEP
tags:
  - OS
  - 虚拟化
  - 进程
  - OSTEP
  - process-run.py
excerpt: "结合 OSTEP 第四章理论与 process-run.py 模拟实验，理解进程抽象、时分/空分共享、进程状态，以及进程调度中 CPU 与 I/O 的重叠策略。"
toc: true
toc_label: "目录"
toc_icon: "list"
toc_sticky: true
---

# 虚拟化：进程入门

> **适合人群**：正在学习操作系统、尤其是 OSTEP（Operating Systems: Three Easy Pieces）第四章的读者。
{: .notice--info}

> **配套工具**：本文所有模拟实验均使用 OSTEP 官方提供的 `process-run.py` 完成。
{: .notice--info}

## 一、进程抽象的核心概念

### 1.1 进程就是运行中的程序

> **知识点**：程序是静态的二进制文件，进程是运行中的程序及其状态的集合。
{: .notice--primary}

程序本身只是磁盘上的二进制文件，是“死”的。操作系统把程序加载到内存、分配资源、让它跑起来之后，才成为一个**进程（process）**。所以：

> 进程 = 运行中的程序 + 它的状态 + 占用的资源

### 1.2 时分共享（Time Sharing）与空分共享（Space Sharing）

> **知识点**：操作系统通过时分共享轮流使用 CPU，通过空分共享划分内存/磁盘空间。
{: .notice--primary}

操作系统要同时服务多个进程，本质上用了两种共享技术：

- **时分共享 Time Sharing**：同一资源（比如 CPU）轮流给不同进程使用一小段时间。因为切换得很快，用户感觉多个程序在“同时”运行。
- **空分共享 Space Sharing**：把资源按空间切分，不同进程各占一块。比如内存被划分为不同地址空间，磁盘被划分为不同文件。

CPU 调度用的是时分共享；内存、磁盘用的是空分共享。

### 1.3 进程的机器状态

> **知识点**：进程的机器状态包括 PC、寄存器和地址空间，这些信息保存在 PCB 中。
{: .notice--primary}

进程运行时，操作系统需要保存它的“现场”，主要包括：

- **程序计数器 PC**：下一条要执行的指令地址
- **寄存器**：通用寄存器、栈指针等
- **地址空间**：代码段、堆、栈等

这些信息存放在 **进程控制块 PCB（Process Control Block）** 里。切换进程时，OS 保存旧进程的 PCB，加载新进程的 PCB，这就是上下文切换。

### 1.4 进程状态

> **知识点**：进程在 RUNNING、READY、BLOCKED 等状态之间转换，状态变化由调度、I/O 等事件触发。
{: .notice--primary}

一个进程在生命周期中会在几个状态之间转换：

- **RUNNING**：正在 CPU 上执行
- **READY**：准备好运行，等待 CPU 调度
- **BLOCKED**：等待某事件完成（通常是 I/O）

状态转换主要由以下事件触发：

- 被调度器选中 → RUNNING
- 时间片用完或被抢占 → READY
- 发起 I/O → BLOCKED
- I/O 完成 → READY

可以用下面的状态图直观表示：

<svg xmlns="http://www.w3.org/2000/svg" width="600" height="300" viewBox="0 0 600 300" style="max-width:100%;height:auto;background:#fafafa;border:1px solid #e0e0e0;border-radius:6px;">
  <defs>
    <marker id="arrow" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto" markerUnits="strokeWidth">
      <path d="M0,0 L0,6 L9,3 z" fill="#555" />
    </marker>
  </defs>
  <style>
    .node { fill:#e3f2fd; stroke:#1976d2; stroke-width:2; rx:6; }
    .done { fill:#e8f5e9; stroke:#388e3c; stroke-width:2; rx:6; }
    .text { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif; font-size:14px; text-anchor:middle; dominant-baseline:middle; }
    .label { font-size:12px; fill:#333; text-anchor:middle; }
    .arrow { stroke:#555; stroke-width:2; fill:none; marker-end:url(#arrow); }
  </style>
  <rect x="60" y="120" width="90" height="50" class="node"/>
  <text x="105" y="145" class="text">READY</text>
  <rect x="280" y="120" width="100" height="50" class="node"/>
  <text x="330" y="145" class="text">RUNNING</text>
  <rect x="280" y="210" width="100" height="50" class="node"/>
  <text x="330" y="235" class="text">BLOCKED</text>
  <rect x="490" y="120" width="80" height="50" class="done"/>
  <text x="530" y="145" class="text">DONE</text>
  <line x1="150" y1="145" x2="280" y2="145" class="arrow"/>
  <text x="215" y="135" class="label">被调度器选中</text>
  <path d="M 330 120 Q 330 70 215 70 Q 100 70 105 120" class="arrow"/>
  <text x="215" y="60" class="label">时间片用完 / 被抢占</text>
  <line x1="330" y1="170" x2="330" y2="210" class="arrow"/>
  <text x="360" y="190" class="label">发起 I/O</text>
  <path d="M 280 235 Q 180 235 150 170" class="arrow"/>
  <text x="200" y="225" class="label">I/O 完成</text>
  <line x1="380" y1="145" x2="490" y2="145" class="arrow"/>
  <text x="435" y="135" class="label">执行完毕</text>
</svg>

### 1.5 从程序到进程：加载

> **知识点**：操作系统通过加载程序、分配地址空间、初始化栈/堆，把静态程序变成动态进程。
{: .notice--primary}

操作系统加载程序时大致做：

1. 从磁盘读取可执行文件
2. 分配地址空间（代码、静态数据、堆、栈）
3. 初始化栈和堆指针
4. 跳到入口点开始执行

这时，一个静态的程序就变成了动态的进程。

### 1.6 数据结构：进程列表与 PCB

> **知识点**：操作系统用进程列表和 PCB 跟踪每个进程；xv6 的 `struct proc` 是一个典型实现。
{: .notice--primary}

操作系统本身也是程序，它需要用数据结构来跟踪系统中的所有进程。

- **进程列表 / 任务列表（Process List / Task List）**：保存系统中所有进程信息的列表，每个条目就是一个 PCB。
- **进程控制块 PCB（Process Control Block）**，也叫进程描述符：保存单个进程的所有关键状态。

下面是 xv6 内核中 `struct proc` 的简化版本，它展示了 OS 需要记录哪些信息：

```c
// 进程可能处于的状态
enum proc_state { UNUSED, EMBRYO, SLEEPING,
                  RUNNABLE, RUNNING, ZOMBIE };

struct proc {
    char *mem;              // 进程内存起始地址
    uint sz;                // 进程内存大小
    char *kstack;           // 内核栈底部
    enum proc_state state;  // 当前状态
    int pid;                // 进程 ID
    struct proc *parent;    // 父进程
    void *chan;             // 如果非空，表示正在某个事件上睡眠
    int killed;             // 是否被终止
    struct file *ofile[NOFILE]; // 打开的文件
    struct inode *cwd;      // 当前工作目录
    struct context context; // 寄存器上下文，用于上下文切换
    struct trapframe *tf;   // 当前中断的陷阱帧
};
```

关键字段含义：

| 字段 | 作用 |
|------|------|
| `state` | 进程当前状态，如 RUNNABLE（READY）、RUNNING、SLEEPING（BLOCKED）、ZOMBIE |
| `context` | 保存寄存器上下文，上下文切换时保存/恢复 |
| `parent` | 指向父进程，构成进程树 |
| `ofile` | 进程打开的文件描述符 |
| `cwd` | 当前工作目录 |

为了高效调度，操作系统还会维护不同的队列或列表：

- **就绪队列（Ready Queue）**：所有等待 CPU 的进程
- **等待队列（Wait Queue）**：按等待事件分类的阻塞进程
- **僵尸进程（Zombie）**：已终止但尚未被父进程回收的进程

> **Zombie 状态**：进程结束后，内核会保留它的退出码，直到父进程调用 `wait()` 来“收尸”。如果父进程一直不 `wait()`， terminated 进程就会一直占用 PCB，变成僵尸进程。
{: .notice--warning}

## 二、用 process-run.py 理解调度与 I/O/CPU 重叠

> **核心问题**：进程调度的本质是如何在多个进程之间分配 CPU，同时尽可能让 I/O 设备也忙碌起来。
{: .notice--info}

`process-run.py` 是 OSTEP 第四章的模拟器，用来观察不同进程组合和调度策略下的行为。

### 2.1 模拟器基本参数

- `-l X:Y`：生成一个进程，`X` 是指令数，`Y` 是每条指令为 CPU 指令的概率（%）
  - `5:100`：5 条全是 CPU
  - `1:0`：1 条是 I/O
  - `3:50`：3 条，每条 50% 概率是 CPU 或 I/O
- `-S`：进程切换策略，决定操作系统何时把 CPU 从当前进程移交给另一个进程
  - `SWITCH_ON_IO`：当前进程执行 `io` 指令发起 I/O 时，操作系统立即把它置为 `BLOCKED`，并切换到下一个 `READY` 进程。也就是说，**进程一旦开始等 I/O，就主动出让 CPU**，避免 CPU 空转。
  - `SWITCH_ON_END`：只有当当前进程执行完所有指令（`DONE`）后，操作系统才切换进程。如果当前进程因 I/O 阻塞，操作系统也不会切走，CPU 只能空转，直到该进程 I/O 完成。
- `-I`：I/O 完成后的调度策略，决定 I/O 完成时谁获得 CPU
  - `IO_RUN_LATER`：I/O 完成后，发起 I/O 的进程回到 `READY` 队列，但不抢占当前正在运行的进程；调度器按正常规则稍后调度它。只有当系统中没有其它可运行进程时，它才会立即执行。
  - `IO_RUN_IMMEDIATE`：I/O 完成后，发起 I/O 的进程**立即被调度运行**。如果 CPU 上正有别的进程在运行，该进程会被抢占回 `READY`，I/O 完成进程立刻获得 CPU。
- `-L`：一次 I/O 的等待时长（默认 5 个 tick），即 I/O 设备实际工作的时间
- `-c`：直接给出答案（运行模拟）
- `-p`：打印统计信息

> **注意**：`io` 指令本身只占 **1 个 CPU tick**（用于向操作系统发起 I/O 请求），随后进程会阻塞 `-L` 个 tick（即 I/O 设备实际工作的时间，默认 5），I/O 完成时再用 **1 个 CPU tick** 执行 `io_done`。所以一次完整的 I/O 总耗时 = `1 + -L + 1`。
{: .notice--warning}

### 2.2 process-run.py 完整代码

<details markdown="1">
<summary>点击展开 / 收起 <code>process-run.py</code> 完整代码</summary>

```python
#! /usr/bin/env python

from __future__ import print_function
import sys
from optparse import OptionParser
import random


# to make Python2 and Python3 act the same -- how dumb
def random_seed(seed):
    try:
        random.seed(seed, version=1)
    except:
        random.seed(seed)
    return


# process switch behavior
SCHED_SWITCH_ON_IO = 'SWITCH_ON_IO'
SCHED_SWITCH_ON_END = 'SWITCH_ON_END'

# io finished behavior
IO_RUN_LATER = 'IO_RUN_LATER'
IO_RUN_IMMEDIATE = 'IO_RUN_IMMEDIATE'

# process states
STATE_RUNNING = 'RUNNING'
STATE_READY = 'READY'
STATE_DONE = 'DONE'
STATE_WAIT = 'BLOCKED'

# members of process structure
PROC_CODE = 'code_'
PROC_PC = 'pc_'
PROC_ID = 'pid_'
PROC_STATE = 'proc_state_'

# things a process can do
DO_COMPUTE = 'cpu'
DO_IO = 'io'
DO_IO_DONE = 'io_done'


class scheduler:
    def __init__(self, process_switch_behavior, io_done_behavior, io_length):
        # keep set of instructions for each of the processes
        self.proc_info = {}
        self.process_switch_behavior = process_switch_behavior
        self.io_done_behavior = io_done_behavior
        self.io_length = io_length
        return

    def new_process(self):
        proc_id = len(self.proc_info)
        self.proc_info[proc_id] = {}
        self.proc_info[proc_id][PROC_PC] = 0
        self.proc_info[proc_id][PROC_ID] = proc_id
        self.proc_info[proc_id][PROC_CODE] = []
        self.proc_info[proc_id][PROC_STATE] = STATE_READY
        return proc_id

    # program looks like this:
    #   c7,i,c1,i
    # which means
    #   compute for 7, then i/o, then compute for 1, then i/o
    def load_program(self, program):
        proc_id = self.new_process()
        for line in program.split(','):
            opcode = line[0]
            if opcode == 'c':  # compute
                num = int(line[1:])
                for i in range(num):
                    self.proc_info[proc_id][PROC_CODE].append(DO_COMPUTE)
            elif opcode == 'i':
                self.proc_info[proc_id][PROC_CODE].append(DO_IO)
                # add one compute to HANDLE the I/O completion
                self.proc_info[proc_id][PROC_CODE].append(DO_IO_DONE)
            else:
                print('bad opcode %s (should be c or i)' % opcode)
                exit(1)
        return

    def load(self, program_description):
        proc_id = self.new_process()
        tmp = program_description.split(':')
        if len(tmp) != 2:
            print('Bad description (%s): Must be number <x:y>' % program_description)
            print('  where X is the number of instructions')
            print('  and Y is the percent change that an instruction is CPU not IO')
            exit(1)

        num_instructions, chance_cpu = int(tmp[0]), float(tmp[1]) / 100.0
        for i in range(num_instructions):
            if random.random() < chance_cpu:
                self.proc_info[proc_id][PROC_CODE].append(DO_COMPUTE)
            else:
                self.proc_info[proc_id][PROC_CODE].append(DO_IO)
                # add one compute to HANDLE the I/O completion
                self.proc_info[proc_id][PROC_CODE].append(DO_IO_DONE)
        return

    def move_to_ready(self, expected, pid=-1):
        if pid == -1:
            pid = self.curr_proc
        assert (self.proc_info[pid][PROC_STATE] == expected)
        self.proc_info[pid][PROC_STATE] = STATE_READY
        return

    def move_to_wait(self, expected):
        assert (self.proc_info[self.curr_proc][PROC_STATE] == expected)
        self.proc_info[self.curr_proc][PROC_STATE] = STATE_WAIT
        return

    def move_to_running(self, expected):
        assert (self.proc_info[self.curr_proc][PROC_STATE] == expected)
        self.proc_info[self.curr_proc][PROC_STATE] = STATE_RUNNING
        return

    def move_to_done(self, expected):
        assert (self.proc_info[self.curr_proc][PROC_STATE] == expected)
        self.proc_info[self.curr_proc][PROC_STATE] = STATE_DONE
        return

    def next_proc(self, pid=-1):
        if pid != -1:
            self.curr_proc = pid
            self.move_to_running(STATE_READY)
            return
        for pid in range(self.curr_proc + 1, len(self.proc_info)):
            if self.proc_info[pid][PROC_STATE] == STATE_READY:
                self.curr_proc = pid
                self.move_to_running(STATE_READY)
                return
        for pid in range(0, self.curr_proc + 1):
            if self.proc_info[pid][PROC_STATE] == STATE_READY:
                self.curr_proc = pid
                self.move_to_running(STATE_READY)
                return
        return

    def get_num_processes(self):
        return len(self.proc_info)

    def get_num_instructions(self, pid):
        return len(self.proc_info[pid][PROC_CODE])

    def get_instruction(self, pid, index):
        return self.proc_info[pid][PROC_CODE][index]

    def get_num_active(self):
        num_active = 0
        for pid in range(len(self.proc_info)):
            if self.proc_info[pid][PROC_STATE] != STATE_DONE:
                num_active += 1
        return num_active

    def get_num_runnable(self):
        num_active = 0
        for pid in range(len(self.proc_info)):
            if self.proc_info[pid][PROC_STATE] == STATE_READY or \
                    self.proc_info[pid][PROC_STATE] == STATE_RUNNING:
                num_active += 1
        return num_active

    def get_ios_in_flight(self, current_time):
        num_in_flight = 0
        for pid in range(len(self.proc_info)):
            for t in self.io_finish_times[pid]:
                if t > current_time:
                    num_in_flight += 1
        return num_in_flight

    def check_for_switch(self):
        return

    def space(self, num_columns):
        for i in range(num_columns):
            print('%10s' % ' ', end='')

    def check_if_done(self):
        if len(self.proc_info[self.curr_proc][PROC_CODE]) == 0:
            if self.proc_info[self.curr_proc][PROC_STATE] == STATE_RUNNING:
                self.move_to_done(STATE_RUNNING)
                self.next_proc()
        return

    def run(self):
        clock_tick = 0

        if len(self.proc_info) == 0:
            return

        # track outstanding IOs, per process
        self.io_finish_times = {}
        for pid in range(len(self.proc_info)):
            self.io_finish_times[pid] = []

        # make first one active
        self.curr_proc = 0
        self.move_to_running(STATE_READY)

        # OUTPUT: headers for each column
        print('%s' % 'Time', end='')
        for pid in range(len(self.proc_info)):
            print('%14s' % ('PID:%2d' % (pid)), end='')
        print('%14s' % 'CPU', end='')
        print('%14s' % 'IOs', end='')
        print('')

        # init statistics
        io_busy = 0
        cpu_busy = 0

        while self.get_num_active() > 0:
            clock_tick += 1

            # check for io finish
            io_done = False
            for pid in range(len(self.proc_info)):
                if clock_tick in self.io_finish_times[pid]:
                    io_done = True
                    self.move_to_ready(STATE_WAIT, pid)
                    if self.io_done_behavior == IO_RUN_IMMEDIATE:
                        # IO_RUN_IMMEDIATE
                        if self.curr_proc != pid:
                            if self.proc_info[self.curr_proc][PROC_STATE] == STATE_RUNNING:
                                self.move_to_ready(STATE_RUNNING)
                        self.next_proc(pid)
                    else:
                        # IO_RUN_LATER
                        if self.process_switch_behavior == SCHED_SWITCH_ON_END and self.get_num_runnable() > 1:
                            # this means the process that issued the io should be run
                            self.next_proc(pid)
                        if self.get_num_runnable() == 1:
                            # this is the only thing to run: so run it
                            self.next_proc(pid)
                    self.check_if_done()

            # if current proc is RUNNING and has an instruction, execute it
            instruction_to_execute = ''
            if self.proc_info[self.curr_proc][PROC_STATE] == STATE_RUNNING and \
                    len(self.proc_info[self.curr_proc][PROC_CODE]) > 0:
                instruction_to_execute = self.proc_info[self.curr_proc][PROC_CODE].pop(0)
                cpu_busy += 1

            # OUTPUT: print what everyone is up to
            if io_done:
                print('%3d*' % clock_tick, end='')
            else:
                print('%3d ' % clock_tick, end='')
            for pid in range(len(self.proc_info)):
                if pid == self.curr_proc and instruction_to_execute != '':
                    print('%14s' % ('RUN:' + instruction_to_execute), end='')
                else:
                    print('%14s' % (self.proc_info[pid][PROC_STATE]), end='')

            # CPU output here: if no instruction executes, output a space, otherwise a 1
            if instruction_to_execute == '':
                print('%14s' % ' ', end='')
            else:
                print('%14s' % '1', end='')

            # IO output here:
            num_outstanding = self.get_ios_in_flight(clock_tick)
            if num_outstanding > 0:
                print('%14s' % str(num_outstanding), end='')
                io_busy += 1
            else:
                print('%10s' % ' ', end='')
            print('')

            # if this is an IO start instruction, switch to waiting state
            # and add an io completion in the future
            if instruction_to_execute == DO_IO:
                self.move_to_wait(STATE_RUNNING)
                self.io_finish_times[self.curr_proc].append(clock_tick + self.io_length + 1)
                if self.process_switch_behavior == SCHED_SWITCH_ON_IO:
                    self.next_proc()

            # ENDCASE: check if currently running thing is out of instructions
            self.check_if_done()
        return (cpu_busy, io_busy, clock_tick)


#
# PARSE ARGUMENTS
#

parser = OptionParser()
parser.add_option('-s', '--seed', default=0, help='the random seed', action='store', type='int', dest='seed')
parser.add_option('-P', '--program', default='', help='more specific controls over programs', action='store',
                  type='string', dest='program')
parser.add_option('-l', '--processlist', default='',
                  help='a comma-separated list of processes to run, in the form X1:Y1,X2:Y2,... where X is the number of instructions that process should run, and Y the chances (from 0 to 100) that an instruction will use the CPU or issue an IO (i.e., if Y is 100, a process will ONLY use the CPU and issue no I/Os; if Y is 0, a process will only issue I/Os)',
                  action='store', type='string', dest='process_list')
parser.add_option('-L', '--iolength', default=5, help='how long an IO takes', action='store', type='int',
                  dest='io_length')
parser.add_option('-S', '--switch', default='SWITCH_ON_IO',
                  help='when to switch between processes: SWITCH_ON_IO, SWITCH_ON_END', action='store', type='string',
                  dest='process_switch_behavior')
parser.add_option('-I', '--iodone', default='IO_RUN_LATER',
                  help='type of behavior when IO ends: IO_RUN_LATER, IO_RUN_IMMEDIATE', action='store', type='string',
                  dest='io_done_behavior')
parser.add_option('-c', help='compute answers for me', action='store_true', default=False, dest='solve')
parser.add_option('-p', '--printstats',
                  help='print statistics at end; only useful with -c flag (otherwise stats are not printed)',
                  action='store_true', default=False, dest='print_stats')
(options, args) = parser.parse_args()

random_seed(options.seed)

assert (options.process_switch_behavior == SCHED_SWITCH_ON_IO or options.process_switch_behavior == SCHED_SWITCH_ON_END)
assert (options.io_done_behavior == IO_RUN_IMMEDIATE or options.io_done_behavior == IO_RUN_LATER)

s = scheduler(options.process_switch_behavior, options.io_done_behavior, options.io_length)

if options.program != '':
    for p in options.program.split(':'):
        s.load_program(p)
else:
    # example process description (10:100,10:100)
    for p in options.process_list.split(','):
        s.load(p)

assert (options.io_length >= 0)

if options.solve == False:
    print('Produce a trace of what would happen when you run these processes:')
    for pid in range(s.get_num_processes()):
        print('Process %d' % pid)
        for inst in range(s.get_num_instructions(pid)):
            print('  %s' % s.get_instruction(pid, inst))
        print('')
    print('Important behaviors:')
    print('  System will switch when ', end='')
    if options.process_switch_behavior == SCHED_SWITCH_ON_IO:
        print('the current process is FINISHED or ISSUES AN IO')
    else:
        print('the current process is FINISHED')
    print('  After IOs, the process issuing the IO will ', end='')
    if options.io_done_behavior == IO_RUN_IMMEDIATE:
        print('run IMMEDIATELY')
    else:
        print('run LATER (when it is its turn)')
    print('')
    exit(0)

(cpu_busy, io_busy, clock_tick) = s.run()

if options.print_stats:
    print('')
    print('Stats: Total Time %d' % clock_tick)
    print('Stats: CPU Busy %d (%.2f%%)' % (cpu_busy, 100.0 * float(cpu_busy) / clock_tick))
    print('Stats: IO Busy  %d (%.2f%%)' % (io_busy, 100.0 * float(io_busy) / clock_tick))
    print('')
```

</details>

> **原文出处**：
> - 模拟器代码：[remzi-arpacidusseau/ostep-homework/cpu-intro/process-run.py](https://github.com/remzi-arpacidusseau/ostep-homework/blob/master/cpu-intro/process-run.py)
> - 配套教材页面：[Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/)
{: .notice--info}

> **阅读建议**：对照代码重点关注 `run()` 方法里的主循环，以及 `next_proc()`、`move_to_wait()`、`move_to_ready()` 这几个函数，它们共同实现了简化的上下文切换逻辑。
{: .notice--tip}

### 2.3 关键实验与发现

#### 实验 1：进程顺序决定总时间

- `-l 4:100,1:0`：CPU 先跑完，再发起 I/O
  - 总时间 11，CPU 利用率 54.55%
- `-l 1:0,4:100`：I/O 先发起，CPU 在 I/O 等待期间运行
  - 总时间 7，CPU 利用率 85.71%

> **结论**：把 I/O 密集型进程放在前面，可以让 CPU 与 I/O 重叠，显著缩短总时间。
{: .notice--success}

#### 实验 2：切换策略 SWITCH_ON_END vs SWITCH_ON_IO

| 策略 | 总时间 | CPU 利用率 |
|---|---|---|
| SWITCH_ON_END | 11 | 54.55% |
| SWITCH_ON_IO | 7 | 85.71% |

从策略含义理解：

- `SWITCH_ON_IO` 下，PID 0 一发 I/O 就进入 `BLOCKED`，操作系统立刻调度 PID 1，CPU 在 I/O 等待期间继续工作。
- `SWITCH_ON_END` 下，PID 0 发 I/O 后操作系统不会切走，CPU 只能空等 PID 0 的 I/O 完成，再做 `io_done`，最后才轮到 PID 1。

> **结论**：`SWITCH_ON_IO` 允许 CPU 与 I/O 重叠；`SWITCH_ON_END` 则让 CPU 在 I/O 等待期间空转。
{: .notice--success}

#### 实验 3：I/O 完成后的行为

用 `-l 3:0,5:100,5:100,5:100 -S SWITCH_ON_IO` 测试：

| I/O 完成策略 | 总时间 | CPU 利用率 | I/O 利用率 |
|---|---|---|---|
| IO_RUN_LATER | 31 | 67.74% | 48.39% |
| IO_RUN_IMMEDIATE | 21 | 100.00% | 71.43% |

从策略含义理解：

- `IO_RUN_LATER` 下，PID 0 的 I/O 完成后只是回到 `READY`，不会抢回 CPU。此时 CPU 正忙着运行 PID 1/2/3，所以 PID 0 只能等它们全部结束后才执行 `io_done`，导致 I/O 设备在最后长时间空闲。
- `IO_RUN_IMMEDIATE` 下，PID 0 的 I/O 一完成就立即被调度，抢占当前运行的进程，执行 `io_done` 并立刻发出下一次 I/O。这样 I/O 设备几乎不停，CPU 也能被其它进程填满，总时间更短。

> **结论**：`IO_RUN_IMMEDIATE` 能让 I/O 设备保持忙碌；`IO_RUN_LATER` 在存在多个 CPU 进程时容易造成 I/O 空闲。
{: .notice--success}

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

### 2.4 简化的上下文切换图景

把这些实验串起来，可以画出操作系统调度的核心逻辑：

1. CPU 上运行的进程发起 I/O。
2. 如果策略是 `SWITCH_ON_IO`，OS 保存该进程的 PCB，把它设为 BLOCKED，然后选择另一个 READY 进程运行。
3. I/O 设备在后台工作，CPU 同时执行别的进程。
4. I/O 完成后，进程变为 READY。
5. 根据 `IO_RUN_IMMEDIATE` 或 `IO_RUN_LATER`，OS 决定是立刻恢复它，还是让它排队等待调度。

这就是现代操作系统里 **CPU 虚拟化** 的最基本形式：通过时分共享，让多个进程“同时”推进，并通过合理调度隐藏 I/O 延迟。

下面用甘特图展示 `-l 1:0,4:100 -S SWITCH_ON_IO` 的调度过程：

<svg xmlns="http://www.w3.org/2000/svg" width="700" height="220" viewBox="0 0 700 220" style="max-width:100%;height:auto;background:#fafafa;border:1px solid #e0e0e0;border-radius:6px;">
  <style>
    .axis { stroke:#999; stroke-width:1; }
    .label { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif; font-size:13px; text-anchor:middle; dominant-baseline:middle; }
    .row-label { font-size:14px; font-weight:bold; text-anchor:end; dominant-baseline:middle; }
    .cpu { fill:#bbdefb; stroke:#1976d2; stroke-width:1; rx:3; }
    .io { fill:#c8e6c9; stroke:#388e3c; stroke-width:1; rx:3; }
    .title { font-size:15px; font-weight:bold; text-anchor:start; }
  </style>
  <text x="20" y="25" class="title">SWITCH_ON_IO：-l 1:0,4:100</text>
  <text x="90" y="85" class="row-label">CPU</text>
  <text x="90" y="145" class="row-label">I/O</text>
  <line x1="100" y1="170" x2="650" y2="170" class="axis"/>
  <text x="100" y="190" class="label">1</text>
  <text x="175" y="190" class="label">2</text>
  <text x="250" y="190" class="label">3</text>
  <text x="325" y="190" class="label">4</text>
  <text x="400" y="190" class="label">5</text>
  <text x="475" y="190" class="label">6</text>
  <text x="550" y="190" class="label">7</text>
  <text x="625" y="190" class="label">8</text>
  <rect x="100" y="65" width="75" height="36" class="cpu"/>
  <text x="137" y="83" class="label">PID0 发 I/O</text>
  <rect x="175" y="65" width="300" height="36" class="cpu"/>
  <text x="325" y="83" class="label">PID1 执行 CPU</text>
  <rect x="550" y="65" width="75" height="36" class="cpu"/>
  <text x="587" y="83" class="label">io_done</text>
  <rect x="175" y="125" width="375" height="36" class="io"/>
  <text x="362" y="143" class="label">I/O 设备工作</text>
</svg>

### 2.5 “CPU4” 与 “4×CPU1”、“IO4” 与 “4×IO1” 的区别

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

> 这里的 `RUN:io` 和 `RUN:io_done` 各占 1 CPU tick，中间 4 个 `BLOCKED` tick 才是 I/O 设备实际工作的 4 个时钟周期。
{: .notice--info}

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

1. **总时间不同**：一次 IO4（`-L 4`，即 I/O 设备实际工作 4 tick）只需 1 次 `io` + 1 次 `io_done` 的 CPU 开销，共 **2 CPU tick + 4 IO wait = 6**；四次 IO1（每次 `-L 1`）虽然 I/O 设备总工作时间也是 4 tick，但有 4 次 `io` + 4 次 `io_done` 的 CPU 开销，共 **8 CPU tick + 4 IO wait = 12**。
2. **CPU 利用率表象不同**：拆成 4 次后 CPU 看起来“更忙”，但完成同样 I/O 量的总效率更低。
3. **调度机会不同**：4×IO1 让进程多次在 RUNNING/BLOCKED 之间切换，给 OS 更多机会调度别的进程；一次 IO4 则让进程长时间阻塞，期间 CPU 必须找别的事做，否则空转。

实际系统中也类似：一次大 I/O 通常比多次小 I/O 更高效，因为每次 I/O 都有系统调用、设备初始化、DMA 设置等固定开销；但多次小 I/O 能提高进程的可调度性和响应性。所以真实系统里常有 I/O 合并（I/O merging）或块大小权衡：合并减少开销，拆分提高响应性。
