---
title: "虚拟化：进程创建（待完成）"
date: 2026-07-10 00:00:00 +0800
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
  - waitpid
excerpt: "整理 OSTEP 第五章 Process API 的核心内容：fork、wait/waitpid、exec 六种变体、环境变量与参数，以及它们如何组合出 Unix 的进程创建机制。"
toc: true
toc_label: "目录"
toc_icon: "list"
toc_sticky: true
---

> **适合人群**：正在学习 OSTEP（Operating Systems: Three Easy Pieces）第五章，或者想系统理解 Unix 进程创建 API 的读者。
{: .notice--info}

> **说明**：本文为学习过程中问答整理的初稿，标记“待完成”，后续会补充更多图示、实验和 xv6 源码对照。
{: .notice--warning}

## 我们要学什么

OSTEP 第四章讲的是进程抽象和 CPU 调度，第五章则进入 **Process API**：操作系统给用户程序提供了哪些系统调用来创建、控制和替换进程。

本章核心就三个函数：

- `fork()`：创建一个新进程
- `wait()` / `waitpid()`：等待子进程结束
- `exec()`：用新程序替换当前进程

这三个函数看起来简单，但组合起来却构成了 Unix/Linux 进程模型的基础：`fork()` 创建，`exec()` 换装，`wait()` 收尾。

---

## 1. fork()：创建子进程

`fork()` 是 Unix 中创建进程的唯一方式（除了特殊的 `clone()` 等）。

### 函数原型

```c
#include <unistd.h>

pid_t fork(void);
```

### 返回值

| 返回值 | 含义 |
|---|---|
| `-1` | 创建失败 |
| `0` | 在**子进程**中返回 |
| `> 0` | 在**父进程**中返回，值为子进程的 PID |

### 关键特性

- 调用一次，返回两次。
- 子进程复制父进程的地址空间（现代 OS 用写时复制 COW，不会立即复制所有页）。
- 子进程从 `fork()` 返回处继续执行。
- 父子进程拥有独立的 PID 和独立的资源（打开文件、当前目录等会继承）。

### 简单例子

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
    } else if (pid == 0) {
        printf("我是子进程，PID = %d\n", getpid());
    } else {
        printf("我是父进程，子进程 PID = %d\n", pid);
    }

    return 0;
}
```

输出顺序不确定，取决于调度。

---

## 2. wait() / waitpid()：等待子进程

子进程结束后，内核不会立即释放它的 PCB，需要父进程调用 `wait()` 或 `waitpid()` 来“收尸”。

### 2.1 wait()

```c
pid_t wait(int *status);
```

- **输入**：`status` 指向整数，用于接收子进程退出状态（可为 `NULL`）。
- **输出**：成功返回子进程 PID，失败返回 `-1`。
- **行为**：阻塞等待**任意一个**子进程结束。
- **等价于**：`waitpid(-1, status, 0)`。

### 2.2 waitpid()

```c
pid_t waitpid(pid_t pid, int *status, int options);
```

#### 输入参数

| 参数 | 作用 |
|---|---|
| `pid` | 指定等待哪个子进程 |
| `status` | 接收退出状态 |
| `options` | 控制选项，如是否阻塞 |

#### pid 取值

| 取值 | 含义 |
|---|---|
| `pid > 0` | 等待 PID 等于该值的特定子进程 |
| `pid == -1` | 等待任意子进程（等价于 `wait()`） |
| `pid == 0` | 等待与调用进程**同进程组**的任意子进程 |
| `pid < -1` | 等待进程组 ID 为 `\|pid\|` 的任意子进程 |

#### options 常用选项

| 选项 | 作用 |
|---|---|
| `0` | 默认阻塞等待 |
| `WNOHANG` | 非阻塞：没有子进程结束时立即返回 `0` |
| `WUNTRACED` | 也报告被暂停的子进程 |
| `WCONTINUED` | 也报告由暂停恢复运行的子进程 |

#### 返回值

| 返回值 | 含义 |
|---|---|
| `> 0` | 成功，返回结束子进程的 PID |
| `0` | 使用 `WNOHANG` 且目前没有子进程结束 |
| `-1` | 出错 |

### 2.3 为什么需要 wait()

1. **回收资源**：子进程结束后 PCB 仍保留，父进程 `wait()` 后内核才彻底释放。
2. **获取退出状态**：通过 `status` 判断子进程是正常退出还是被信号终止。
3. **避免僵尸进程**：父进程不 `wait()`，子进程会变成 Zombie。

### 2.4 退出状态解析

`wait()` / `waitpid()` 得到的 `status` 是一个整数，里面编码了子进程的退出信息。不能直接打印它，需要用专门的宏来解析。

#### 常用宏

| 宏 | 作用 |
|---|---|
| `WIFEXITED(status)` | 子进程是否正常退出（调用 `exit()` 或从 `main` 返回） |
| `WEXITSTATUS(status)` | 获取正常退出时的退出码（0 ~ 255） |
| `WIFSIGNALED(status)` | 子进程是否被信号终止 |
| `WTERMSIG(status)` | 获取导致终止的信号编号 |
| `WIFSTOPPED(status)` | 子进程是否被信号暂停（需 `WUNTRACED`） |
| `WSTOPSIG(status)` | 获取导致暂停的信号编号 |
| `WIFCONTINUED(status)` | 子进程是否由暂停恢复（需 `WCONTINUED`） |

#### 基本解析模板

```c
int status;
pid_t pid = wait(&status);

if (WIFEXITED(status)) {
    printf("正常退出，退出码：%d\n", WEXITSTATUS(status));
} else if (WIFSIGNALED(status)) {
    printf("被信号 %d 终止\n", WTERMSIG(status));
}
```

#### 两种常见情况

**正常退出**

```c
if (pid == 0) {
    exit(7);   // 子进程正常退出，退出码 7
}

wait(&status);

if (WIFEXITED(status)) {
    int code = WEXITSTATUS(status);   // code = 7
    printf("退出码：%d\n", code);
}
```

**被信号终止**

```c
if (pid == 0) {
    while (1);   // 子进程死循环
}

kill(pid, SIGKILL);   // 父进程发送 SIGKILL
wait(&status);

if (WIFSIGNALED(status)) {
    int sig = WTERMSIG(status);   // sig = 9
    printf("被信号 %d 终止\n", sig);
}
```

#### status 里到底存了什么

`status` 的低 16 位大致这样组织：

- 高 8 位：正常退出码
- 低 7 位：终止信号编号
- 第 8 位：core dump 标志

具体位布局因系统而异，但这些宏屏蔽了差异，直接调用即可。

#### 完整示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <signal.h>

int main() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        exit(1);
    }

    if (pid == 0) {
        printf("子进程 PID = %d\n", getpid());
        sleep(2);
        exit(3);   // 正常退出，退出码 3
    }

    int status;
    pid_t ret = wait(&status);

    printf("父进程等到子进程 %d\n", ret);

    if (WIFEXITED(status)) {
        printf("正常退出，码 = %d\n", WEXITSTATUS(status));
    } else if (WIFSIGNALED(status)) {
        printf("被信号终止，信号 = %d\n", WTERMSIG(status));
    } else if (WIFSTOPPED(status)) {
        printf("被信号暂停，信号 = %d\n", WSTOPSIG(status));
    }

    return 0;
}
```

---

## 3. exec() 家族：用新程序替换当前进程

`exec()` 不是创建新进程，而是**用新程序替换当前进程的地址空间**。如果成功，不会返回；失败返回 `-1`。

### 3.1 六个变体

```c
int execl (const char *path, const char *arg, ...);
int execv (const char *path, char *const argv[]);
int execle(const char *path, const char *arg, ..., char *const envp[]);
int execve(const char *path, char *const argv[], char *const envp[]);
int execlp(const char *file, const char *arg, ...);
int execvp(const char *file, char *const argv[]);
```

### 3.2 命名规则

| 字母 | 含义 |
|---|---|
| `l` / `v` | 参数传递方式：`l` = list（逐个列举），`v` = vector（数组） |
| `e` | 自定义环境变量 `envp[]` |
| `p` | 使用 `PATH` 环境变量搜索可执行文件 |

### 3.3 六种变体对比

| 函数 | 路径/文件名 | 参数 | 环境变量 | PATH 搜索 |
|---|---|---|---|---|
| `execl` | 完整路径 | list | 继承 | 否 |
| `execv` | 完整路径 | vector | 继承 | 否 |
| `execle` | 完整路径 | list | 自定义 | 否 |
| `execve` | 完整路径 | vector | 自定义 | 否 |
| `execlp` | 文件名 | list | 继承 | 是 |
| `execvp` | 文件名 | vector | 继承 | 是 |

### 3.4 最常用组合

```c
char *args[] = {"ls", "-l", "/home", NULL};
execvp("ls", args);

// 如果失败才执行到这里
perror("execvp failed");
exit(1);
```

### 3.5 PATH 在哪里给出

`PATH` 是环境变量，通常在 shell 配置文件（如 `~/.bashrc`、`~/.zshrc`、`/etc/profile`）中设置，子进程继承而来。

`execvp()` 内部通过 `getenv("PATH")` 拿到它，按 `:` 分割，逐个目录查找可执行文件。

### 3.6 e 和 p 能否同时出现

标准 POSIX 的 6 个 `exec` 变体中没有同时带 `e` 和 `p` 的。但 GNU 扩展提供了：

```c
int execvpe(const char *file, char *const argv[], char *const envp[]);
```

如果需要同时有 `p` 和 `e`，可以自己实现：

```c
// 从自定义 envp 中查找 PATH，搜索可执行文件，最后调用 execve
static char *my_getenv(const char *name, char *const envp[]) {
    size_t len = strlen(name);
    for (int i = 0; envp[i]; i++) {
        if (strncmp(envp[i], name, len) == 0 && envp[i][len] == '=')
            return envp[i] + len + 1;
    }
    return NULL;
}

static int search_path(const char *file, char *const envp[],
                       char *pathbuf, size_t bufsize) {
    if (strchr(file, '/')) {
        if (strlen(file) >= bufsize) { errno = ENAMETOOLONG; return -1; }
        strcpy(pathbuf, file);
        return 0;
    }

    char *path = my_getenv("PATH", envp);
    if (!path || !*path) { errno = ENOENT; return -1; }

    char *copy = strdup(path);
    int found = -1;
    for (char *dir = strtok(copy, ":"); dir; dir = strtok(NULL, ":")) {
        if (*dir == '\0') dir = ".";
        snprintf(pathbuf, bufsize, "%s/%s", dir, file);
        if (access(pathbuf, X_OK) == 0) { found = 0; break; }
    }
    free(copy);

    if (found) { errno = ENOENT; return -1; }
    return 0;
}

int my_execvpe(const char *file, char *const argv[], char *const envp[]) {
    char pathbuf[1024];
    if (search_path(file, envp, pathbuf, sizeof(pathbuf)) < 0)
        return -1;
    execve(pathbuf, argv, envp);
    return -1;  // 只有 execve 失败才会到这里
}
```

---

## 4. 环境变量 vs 程序参数

两者都是传给进程的字符串，但用途不同。

| 对比项 | 程序参数 `argv` | 环境变量 `envp` |
|---|---|---|
| 来源 | 命令行显式输入 | shell 配置，子进程继承 |
| 命名风格 | `-l`、`--help` | 全大写 `PATH`、`HOME` |
| 用途 | 本次命令的具体选项/文件 | 系统配置、运行环境 |
| 作用范围 | 只影响本次运行 | 影响整个进程会话 |
| 获取方式 | `argv[i]` | `getenv("NAME")` |

例如命令：

```bash
USER=admin PATH=/bin:/usr/bin ls -l /home
```

- `ls`、`-l`、`/home` 是参数
- `USER=admin`、`PATH=/bin:/usr/bin` 是环境变量

---

## 5. fork + exec + wait 的标准用法

现代 shell 执行外部命令时，本质上就是：

```c
pid_t pid = fork();

if (pid == 0) {
    // 子进程：替换为要执行的程序
    execlp("ls", "ls", "-l", NULL);
    perror("exec failed");
    exit(1);
} else if (pid > 0) {
    // 父进程：等待子进程结束
    int status;
    waitpid(pid, &status, 0);
} else {
    perror("fork failed");
}
```

这种 `fork()` 后立即 `exec()` 的模式是 Unix 进程创建的精髓：先复制父进程，再换装成新程序。

---

## 6. 常见问题

**Q：`fork()` 后父子进程谁先运行？**

不确定，取决于调度器。如果需要同步，应使用 `wait()`、`信号量`、`pipe` 等机制。

**Q：`exec()` 成功后为什么不返回？**

因为当前进程的地址空间（代码、数据、堆、栈）已被新程序完全替换，原来的调用上下文已不存在。

**Q：为什么不把 fork 和 exec 合并成一个函数？**

分开设计更灵活。例如 shell 的重定向：可以在 `fork()` 后、`exec()` 前修改子进程的文件描述符，而不影响父进程。

```c
if (fork() == 0) {
    close(1);                    // 关闭标准输出
    open("out.txt", O_WRONLY);   // 新 open 返回 fd 1
    execvp("ls", args);          // ls 的输出会进入 out.txt
}
```

**Q：`waitpid(WNOHANG)` 返回 0 是什么意思？**

表示当前没有子进程结束，父进程不会被阻塞，可以继续做其他事情。

**Q：重定向是怎么实现的？**

重定向通过修改进程的文件描述符实现。例如 `ls > out.txt`，shell 会在子进程中 `close(1)` 再 `open("out.txt")`，让 fd 1 指向文件，然后 `exec()` 运行 `ls`。

**Q：`kill()` 是做什么的？**

`kill()` 不是只能杀死进程，它是向进程发送信号。常见信号：`SIGKILL` 强制终止、`SIGTERM` 请求终止、`SIGSTOP` 暂停、`SIGCONT` 继续。

---

## 7. 课后题实践

下面是 OSTEP 第五章课后题 1~4 的代码实现与观察。

### 7.1 第 1 题：fork 与变量复制

文件：`01_fork_x.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    int x = 100;
    printf("before fork: x = %d\n", x);

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        exit(1);
    } else if (pid == 0) {
        printf("child : x = %d\n", x);
        x = 200;
        printf("child after x = 200: x = %d\n", x);
    } else {
        printf("parent: x = %d\n", x);
        x = 300;
        printf("parent after x = 300: x = %d\n", x);
    }

    return 0;
}
```

**观察**

- 子进程继承父进程的 `x` 值（100）
- 父子进程各自修改 `x` 互不影响
- 说明 `fork()` 后父子有独立的地址空间副本

---

### 7.2 第 2 题：open 后 fork

文件：`02_fork_file.c`

```c
int fd = open("fork_output.txt", O_WRONLY | O_CREAT | O_APPEND, 0644);
fork();
// 父子进程都向 fd 写入
```

**观察**

- 子进程继承父进程打开的文件描述符
- `O_APPEND` 保证每次写入追加到文件末尾
- 父子进程共享同一个打开文件表项（包括文件偏移量）

---

### 7.3 第 3 题：确保子进程先打印

文件：`03_fork_hello_goodbye.c`

```c
if (pid == 0) {
    printf("hello\n");
    exit(0);
} else {
    wait(NULL);
    printf("goodbye\n");
}
```

**观察**

- 父进程调用 `wait(NULL)` 阻塞等待子进程结束
- 因此 `hello` 一定在 `goodbye` 之前输出

---

### 7.4 第 4 题：exec 变体

文件：`04_fork_exec_ls.c`

```c
void run_exec(const char *name, void (*exec_func)(void)) {
    printf("\n=== %s ===\n", name);
    fflush(stdout);
    if (fork() == 0) {
        exec_func();
        perror("exec failed");
        exit(1);
    }
    wait(NULL);
}
```

程序演示了 `execl`、`execle`、`execlp`、`execv`、`execvp`、`execvP` 六种变体。

**关键理解**

| 字母 | 含义 |
|---|---|
| `l` / `v` | 参数用列表还是数组传递 |
| `e` | 是否自定义环境变量 |
| `p` | 是否通过 `PATH` 搜索可执行文件 |

---

## 8. 总结

| 函数 | 作用 | 是否创建新进程 |
|---|---|---|
| `fork()` | 创建子进程 | 是 |
| `wait()` / `waitpid()` | 等待并回收子进程 | 否 |
| `exec()` 家族 | 用新程序替换当前进程 | 否 |

Unix 进程创建的哲学：

> 先 `fork()` 复制，再 `exec()` 换装，最后 `wait()` 收尾。

理解了这三个系统调用，就掌握了 Unix 进程控制的核心。

---

> **待完成**：补充 `fork()` 的写时复制（COW）机制图解、`pipe()` 与进程间通信、以及 xv6 中 `fork()` / `exec()` / `wait()` 的源码对照。
{: .notice--warning}
