---
title: "虚拟化：新增进程"
date: 2026-07-16 00:00:00 +0800
categories:
  - 操作系统
  - OSTEP
tags:
  - OS
  - 虚拟化
  - 进程
  - OSTEP
  - fork
  - exec
  - wait
  - waitpid
excerpt: "系统整理 Unix 进程创建三件套 fork、exec、wait/waitpid 的用法，并以 7 道 OSTEP 课后题的实践代码加深理解。"
toc: true
toc_label: "目录"
toc_icon: "list"
toc_sticky: true
---

> **适合人群**：正在学习 OSTEP（Operating Systems: Three Easy Pieces）第五章 Process API，或者想系统理解 Unix 进程创建机制的读者。
{: .notice--info}

## 我们要学什么

Unix 进程创建的核心可以概括为三句话：

> `fork()` 创建，`exec()` 换装，`wait()` 收尾。

这三个系统调用配合起来，构成了 Unix/Linux 进程控制的基础。本文会依次讲解：

- `fork()`：如何创建子进程，父子进程的关系是什么
- `exec()` 家族：如何用新程序替换当前进程
- `wait()` / `waitpid()`：如何等待子进程结束并回收资源
- 相关概念：文件描述符、环境变量、PATH、缓冲区

最后以 7 道实践题把理论落地。

---

## 1. fork()：创建子进程

`fork()` 是 Unix 中创建进程的主要方式。

### 函数原型

```c
#include <unistd.h>

pid_t fork(void);
```

### 返回值

| 返回值 | 含义 |
|--------|------|
| `-1` | 创建失败 |
| `0` | 当前在**子进程**中 |
| `> 0` | 当前在**父进程**中，值为子进程的 PID |

### 关键特性

- 调用一次，返回两次
- 子进程复制父进程的地址空间（现代 OS 使用写时复制 COW）
- 子进程从 `fork()` 返回处继续执行，而不是从 `main()` 开始
- 父子进程拥有独立的 PID 和独立的资源

### 缓冲区问题

`fork()` 会复制父进程的 stdout 缓冲区。如果父进程在 `fork()` 前调用了 `printf` 但没有刷新缓冲区，子进程也会复制到这份未输出的内容，导致重复打印。

解决办法：在 `fork()` 前调用 `fflush(stdout)`。

---

## 2. exec() 家族：用新程序替换当前进程

`exec()` 不会创建新进程，而是**用新程序替换当前进程的地址空间**。如果成功，不会返回；失败返回 `-1`。

### 六个主要变体

```c
int execl (const char *path, const char *arg, ...);
int execv (const char *path, char *const argv[]);
int execle(const char *path, const char *arg, ..., char *const envp[]);
int execve(const char *path, char *const argv[], char *const envp[]);
int execlp(const char *file, const char *arg, ...);
int execvp(const char *file, char *const argv[]);
```

### 命名规律

| 字母 | 含义 |
|------|------|
| `l` / `v` | 参数传递方式：`l` = 列表列举，`v` = 数组 |
| `e` | 自定义环境变量 `envp[]` |
| `p` | 使用 `PATH` 环境变量搜索可执行文件 |

### 输入与输出

| 情况 | 返回值 |
|------|--------|
| 成功 | **不返回**（当前进程已被替换） |
| 失败 | 返回 `-1`，设置 `errno` |

### argv[0] 的约定

`argv[0]` 通常放程序名本身，只是约定，不影响执行。`execvp()` 的第一个参数决定真正执行哪个程序，第二个参数数组决定新程序看到的 `argv[]`。

---

## 3. wait() / waitpid()：等待子进程结束

### wait()

```c
#include <sys/wait.h>

pid_t wait(int *status);
```

等价于 `waitpid(-1, status, 0)`，阻塞等待任意一个子进程结束。

### waitpid()

```c
pid_t waitpid(pid_t pid, int *status, int options);
```

#### pid 参数

| `pid` | 含义 |
|-------|------|
| `> 0` | 等待指定 PID 的子进程 |
| `-1` | 等待任意子进程 |
| `0` | 等待同进程组的任意子进程 |
| `< -1` | 等待指定进程组的任意子进程 |

#### options 参数

| 选项 | 含义 |
|------|------|
| `0` | 阻塞等待 |
| `WNOHANG` | 非阻塞，没有子进程结束时立即返回 `0` |

#### 解析 status

```c
if (WIFEXITED(status)) {
    printf("exit code: %d\n", WEXITSTATUS(status));
} else if (WIFSIGNALED(status)) {
    printf("signal: %d\n", WTERMSIG(status));
}
```

---

## 4. 相关概念

### 文件描述符 fd

`fd` 是操作系统给已打开文件分配的非负整数编号。每个进程有独立的 fd 表。`fork()` 会复制 fd 表，所以父子进程可以共享同一个打开文件。

### 环境变量

环境变量是操作系统传给进程的全局配置字符串。`exec` 带 `e` 的变体可以自定义环境变量，不带 `e` 的变体继承当前环境。

### PATH

`PATH` 是用 `:` 分隔的目录列表，shell 和 `execvp()` 按顺序在这些目录里找可执行文件。当前目录通常不在 `PATH` 里，所以要执行当前目录下的程序需要加 `./`。

### fflush(stdout)

`fflush(stdout)` 强制把 stdout 缓冲区里的内容立即输出。常用于 `fork()` 前避免输出被父子进程重复打印。

#### stdout 的三种缓冲模式

| 模式 | 触发输出条件 | 典型场景 |
|------|-------------|---------|
| **无缓冲** | 每个字符立即输出 | `stderr` |
| **行缓冲** | 遇到 `\n` 或缓冲区满 | 终端里的 `stdout` |
| **全缓冲** | 缓冲区满、手动刷新、程序退出 | 重定向到文件/管道的 `stdout` |

#### 为什么有时不加 `\n` 也能输出

- 程序退出时会自动刷新所有输出流
- 缓冲区恰好满了
- 手动调用了 `fflush(stdout)`
- 显式设置了无缓冲或行缓冲模式

#### 什么时候会“卡住”

```c
printf("loading");   // 没有 \n
sleep(10);           // 10 秒内可能看不到输出
```

在终端里，`stdout` 是行缓冲，没有 `\n` 就留在缓冲区。如果输出重定向到文件，变成全缓冲，要等缓冲区满或程序退出才输出。

---

## 5. 实践 1~7

下面七道题对应 OSTEP 第五章课后题，代码均位于 `system/ch05/` 目录下。

### 实践 1：fork 与变量复制

**思路**：在 `fork()` 前定义变量 `x`，观察父子进程修改 `x` 时是否互相影响。

<details markdown="1">
<summary><strong>点击查看代码</strong></summary>

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    int x = 100;
    printf("%d\n", x);

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        exit(1);
    } else if (pid == 0) {
        printf("%d\n", x);
        x = 200;
        printf("%d\n", x);
    } else {
        printf("%d\n", x);
        x = 300;
        printf("%d\n", x);
    }

    return 0;
}
```

</details>

**运行结果**：

```text
100
100
300
100
100
200
```

**结论**：`fork()` 后父子进程有各自独立的地址空间副本，修改 `x` 互不影响。

---

### 实践 2：fork 与文件描述符

**思路**：父进程打开文件，然后 `fork()`。观察父子进程都向同一个 fd 写入时会发生什么。

<details markdown="1">
<summary><strong>点击查看代码</strong></summary>

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/wait.h>

int main() {
    int fd = open("fork_output.txt", O_WRONLY | O_CREAT | O_APPEND, 0644);
    if (fd < 0) {
        perror("open failed");
        exit(1);
    }

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        exit(1);
    } else if (pid == 0) {
        write(fd, "1\n", 2);
    } else {
        write(fd, "2\n", 2);
        wait(NULL);
    }

    close(fd);
    return 0;
}
```

</details>

**运行结果**（`cat fork_output.txt`）：

```text
2
1
```

**结论**：子进程继承父进程的 fd，两者指向同一个打开文件。`O_APPEND` 保证写入追加到文件末尾，不会互相覆盖。

---

### 实践 3：确保子进程先打印

**思路**：让子进程打印 `hello`，父进程打印 `goodbye`，并确保 `hello` 一定先出现。

<details markdown="1">
<summary><strong>点击查看代码</strong></summary>

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        exit(1);
    } else if (pid == 0) {
        printf("hello\n");
    } else {
        wait(NULL);
        printf("goodbye\n");
    }

    return 0;
}
```

</details>

**运行结果**：

```text
hello
goodbye
```

**结论**：父进程调用 `wait(NULL)` 阻塞等待子进程结束后才打印，顺序得到保证。

---

### 实践 4：exec 变体

**思路**：用 `fork()` 创建子进程，在子进程中分别用 6 种 exec 变体运行 `ls -l /bin/ls`。

<details markdown="1">
<summary><strong>点击查看代码</strong></summary>

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>

char *envp[] = {"PATH=/bin:/usr/bin:/usr/local/bin", "USER=tester", NULL};
char *argv[] = {"ls", "-l", "/bin/ls", NULL};

void run_exec(const char *name, void (*exec_func)(void)) {
    pid_t pid = fork();
    if (pid < 0) {
        perror("fork failed");
        return;
    }
    if (pid == 0) {
        printf("%s\n", name);
        fflush(stdout);
        exec_func();
        perror("exec failed");
        exit(1);
    } else {
        wait(NULL);
    }
}

void do_execl(void)  { execl("/bin/ls", "ls", "-l", "/bin/ls", NULL); }
void do_execle(void) { execle("/bin/ls", "ls", "-l", "/bin/ls", NULL, envp); }
void do_execlp(void) { execlp("ls", "ls", "-l", "/bin/ls", NULL); }
void do_execv(void)  { execv("/bin/ls", argv); }
void do_execvp(void) { execvp("ls", argv); }
void do_execvP(void) { execvP("ls", "/bin:/usr/bin:/usr/local/bin", argv); }

int main() {
    run_exec("execl",  do_execl);
    run_exec("execle", do_execle);
    run_exec("execlp", do_execlp);
    run_exec("execv",  do_execv);
    run_exec("execvp", do_execvp);
    run_exec("execvP", do_execvP);
    return 0;
}
```

</details>

**运行结果**：

```text
execl
-rwxr-xr-x  1 root  wheel  154624 Jun 25 10:29 /bin/ls
execle
-rwxr-xr-x  1 root  wheel  154624 Jun 25 10:29 /bin/ls
execlp
-rwxr-xr-x  1 root  wheel  154624 Jun 25 10:29 /bin/ls
execv
-rwxr-xr-x  1 root  wheel  154624 Jun 25 10:29 /bin/ls
execvp
-rwxr-xr-x  1 root  wheel  154624 Jun 25 10:29 /bin/ls
execvP
-rwxr-xr-x  1 root  wheel  154624 Jun 25 10:29 /bin/ls
```

**结论**：6 种 exec 变体最终都执行了 `ls -l /bin/ls`，区别在于参数传递方式和程序搜索路径。

---

### 实践 5：wait() 的返回值

**思路**：父进程用 `wait()` 等待子进程，观察返回值；再让子进程调用 `wait()`，观察会发生什么。

<details markdown="1">
<summary><strong>点击查看代码</strong></summary>

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <errno.h>

int main() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        exit(1);
    } else if (pid == 0) {
        printf("child pid = %d\n", getpid());

        pid_t ret = wait(NULL);
        printf("child wait returned: %d\n", ret);
        if (ret == -1) {
            perror("child wait error");
        }

        exit(5);
    } else {
        int status;
        pid_t ret = wait(&status);
        printf("parent wait returned: %d\n", ret);
        if (WIFEXITED(status)) {
            printf("child exit code: %d\n", WEXITSTATUS(status));
        }
    }

    return 0;
}
```

</details>

**运行结果**：

```text
child pid = 69295
child wait returned: -1
child wait error: No child processes
parent wait returned: 69295
child exit code: 5
```

**结论**：父进程的 `wait()` 返回子进程 PID；子进程没有子进程，调用 `wait()` 返回 `-1`，`errno` 为 `ECHILD`。

---

### 实践 6：waitpid() 的用法

**思路**：用 `waitpid()` 替代 `wait()`，指定等待某个子进程。

<details markdown="1">
<summary><strong>点击查看代码</strong></summary>

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        exit(1);
    } else if (pid == 0) {
        printf("child pid = %d\n", getpid());
        sleep(1);
        exit(6);
    } else {
        int status;
        pid_t ret = waitpid(pid, &status, 0);
        printf("waitpid returned: %d\n", ret);
        if (WIFEXITED(status)) {
            printf("child exit code: %d\n", WEXITSTATUS(status));
        }
    }

    return 0;
}
```

</details>

**运行结果**：

```text
child pid = 69297
waitpid returned: 69297
child exit code: 6
```

**结论**：`waitpid(pid, &status, 0)` 可以指定等待某个子进程，并获取其退出状态。比 `wait()` 更灵活。

---

### 实践 7：关闭标准输出后 printf

**思路**：子进程关闭 `STDOUT_FILENO`，再调用 `printf()`，观察输出行为。

<details markdown="1">
<summary><strong>点击查看代码</strong></summary>

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        exit(1);
    } else if (pid == 0) {
        close(STDOUT_FILENO);
        printf("this should not appear\n");
    } else {
        wait(NULL);
        printf("parent done\n");
    }

    return 0;
}
```

</details>

**运行结果**：

```text
parent done
```

**结论**：子进程关闭标准输出后，`printf()` 写入 fd 1 失败，没有任何输出显示。

#### 为什么 `printf()` 不报错

`printf()` 只负责把数据写入 `stdout` 缓冲区。遇到 `\n` 时，缓冲区会尝试刷新，调用 `write(1, buf, len)`。但此时 fd 1 已被关闭，`write()` 返回 `-1`，`errno = EBADF`。

`printf()` 本身不检查 `write()` 是否成功，它只返回成功写入缓冲区的字符数，所以程序不会报错。要检测这种输出错误，可以用 `ferror(stdout)`：

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    close(STDOUT_FILENO);
    printf("this should not appear\n");

    if (ferror(stdout)) {
        perror("stdout error");
    }

    return 0;
}
```

输出：

```text
stdout error: Bad file descriptor
```

---

## 总结

| 函数 | 作用 | 是否创建新进程 |
|------|------|---------------|
| `fork()` | 创建子进程 | 是 |
| `exec()` 家族 | 用新程序替换当前进程 | 否 |
| `wait()` / `waitpid()` | 等待并回收子进程 | 否 |

Unix 进程创建的哲学：

> 先 `fork()` 复制，再 `exec()` 换装，最后 `wait()` 收尾。

理解这三个系统调用，就掌握了 Unix 进程控制的核心。
