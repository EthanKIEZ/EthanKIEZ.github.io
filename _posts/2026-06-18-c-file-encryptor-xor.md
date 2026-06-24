---
title: "用 C 语言写一个文件加密器：从命令行参数到 XOR 加密"
date: 2026-06-18 16:00:00 +0800
categories:
  - C语言
tags:
  - C
  - 文件操作
  - XOR
  - 命令行参数
---

> 适合人群：学过 C 语言基础（变量、循环、函数、指针），想动手写一个小项目的新手。

## 我们要做什么

写一个命令行程序，实现对任意文件的加密和解密：

```bash
./file_encryptor input.txt output.enc 42
```

运行后，`input.txt` 会被密钥 `42` 加密，结果保存到 `output.enc`。

想解密？用同样的命令，把加密后的文件当作输入：

```bash
./file_encryptor output.enc decrypted.txt 42
```

`decrypted.txt` 会和原来的 `input.txt` 一模一样。

## 核心思路

我们用的加密方法是 **XOR（按位异或）**，它有一个非常漂亮的性质：

```
A ^ B ^ B = A
```

也就是说，用同一个密钥加密两次，就还原了。所以加密和解密可以用完全相同的代码。

## 完整代码

```c
#include <stdio.h>
#include <stdlib.h>

// 函数声明
void encrypt_file(const char *input_file, const char *output_file, unsigned char key);
void print_usage(const char *program_name);

int main(int argc, char *argv[]) {
    // 程序需要 4 个参数：程序名、输入文件、输出文件、密钥
    if (argc != 4) {
        print_usage(argv[0]);
        return 1;
    }

    const char *input_file = argv[1];
    const char *output_file = argv[2];

    // 把密钥字符串转成数字，并检查是否在 0-255 范围内
    char *endptr;
    long key_input = strtol(argv[3], &endptr, 10);
    if (endptr == argv[3] || *endptr != '\0' || key_input < 0 || key_input > 255) {
        printf("错误：密钥必须是 0-255 之间的整数\n");
        return 1;
    }
    unsigned char key = (unsigned char) key_input;

    printf("输入文件：%s\n", input_file);
    printf("输出文件：%s\n", output_file);
    printf("密钥：%d\n", key);

    encrypt_file(input_file, output_file, key);

    printf("处理完成！\n");
    return 0;
}

void encrypt_file(const char *input_file, const char *output_file, unsigned char key) {
    // rb = read binary，以二进制方式读取
    FILE *in = fopen(input_file, "rb");
    if (in == NULL) {
        printf("错误：无法打开输入文件 %s\n", input_file);
        exit(1);
    }

    // wb = write binary，以二进制方式写入
    FILE *out = fopen(output_file, "wb");
    if (out == NULL) {
        printf("错误：无法创建输出文件 %s\n", output_file);
        fclose(in);
        exit(1);
    }

    int ch;
    // 逐字节读取，XOR 后写入
    while ((ch = fgetc(in)) != EOF) {
        unsigned char encrypted = (unsigned char) ch ^ key;
        fputc(encrypted, out);
    }

    fclose(in);
    fclose(out);
}

void print_usage(const char *program_name) {
    printf("用法：%s <输入文件> <输出文件> <密钥（0-255）>\n", program_name);
    printf("示例：%s secret.txt secret.enc 42\n", program_name);
}
```

## 逐段讲解

### 1. 命令行参数 `argc` 和 `argv`

```c
int main(int argc, char *argv[])
```

- `argc`：命令行参数的个数，**包含程序名本身**。
- `argv`：字符串数组，每个元素是一个参数。

例如运行：

```bash
./file_encryptor a.txt b.enc 42
```

则：

| 变量 | 值 |
|------|------|
| `argc` | `4` |
| `argv[0]` | `"./file_encryptor"` |
| `argv[1]` | `"a.txt"` |
| `argv[2]` | `"b.enc"` |
| `argv[3]` | `"42"` |

所以程序开头检查 `argc != 4`，参数不够就提示用法并退出。

### 2. 为什么用二进制模式打开文件

```c
FILE *in = fopen(input_file, "rb");
FILE *out = fopen(output_file, "wb");
```

- `"r"` 是文本模式，会把 `\r\n` 自动转成 `\n`（Windows），不适合加密后的二进制数据。
- `"rb"` 和 `"wb"` 是二进制模式，**原封不动**地读写每一个字节。

因为加密后的文件不再是可读的文本，必须用二进制模式。

### 3. 逐字节读写

```c
int ch;
while ((ch = fgetc(in)) != EOF) {
    unsigned char encrypted = (unsigned char) ch ^ key;
    fputc(encrypted, out);
}
```

- `fgetc(in)` 从输入文件读一个字节，返回 `int` 类型（读到末尾返回 `EOF`）。
- `ch ^ key` 做 XOR 运算。
- `fputc(encrypted, out)` 把结果写入输出文件。

注意 `ch` 用 `int` 而不是 `unsigned char`，因为要能正确判断 `EOF`。

### 4. XOR 加密为什么能解密

一个字节是 8 位二进制。XOR 的规则是：相同为 `0`，不同为 `1`。

```
明文:  01000001  ('A')
密钥:  00101010  (42)
密文:  01101011  ('k')
```

现在再用密钥 XOR 一次：

```
密文:  01101011  ('k')
密钥:  00101010  (42)
结果:  01000001  ('A')
```

所以加密和解密是同一个操作。

### 5. 用 `strtol` 而不是 `atoi`

`atoi("abc")` 会安静地返回 `0`，程序会误以为密钥是 `0`。

`strtol` 可以通过 `endptr` 判断输入是否合法：

```c
char *endptr;
long key_input = strtol(argv[3], &endptr, 10);

// endptr 指向解析停止的位置
// 如果 endptr 没动，说明一个数字都没解析出来
// 如果 *endptr != '\0'，说明字符串后面还有非法字符
```

这是更安全的做法。

## 编译和运行

### 在终端编译

```bash
gcc -std=c11 main.c -o file_encryptor
```

### 运行

```bash
# 加密
./file_encryptor input.txt output.enc 42

# 解密
./file_encryptor output.enc decrypted.txt 42
```

### 在 CLion 中运行

CLion 默认运行不带参数，会打印用法信息并退出。需要在运行配置里添加参数：

1. 顶部菜单 **Run → Edit Configurations...**
2. 选择 `file_encryptor`
3. 在 **Program arguments** 中填入：`input.txt output.txt 42`
4. 点击 **Apply → OK**

## 常见问题

**Q：密钥范围为什么是 0-255？**

因为 `unsigned char` 正好是一个字节，范围就是 `0` 到 `255`。

**Q：这种加密安全吗？**

不安全。XOR 单字节密钥非常容易被破解。这只是一个学习位运算和文件操作的练习，不要用来保护重要数据。

**Q：为什么解密和加密代码一样？**

因为 `data ^ key ^ key == data`， XOR 运算自己就是自己的逆运算。

## 拓展思考

你可以在这个基础上尝试：

1. 用更长的密钥（比如一个字符串），循环对每个字节 XOR。
2. 把加密后的文件内容用十六进制打印出来。
3. 加一个 `-d` 参数，让程序明确区分加密模式和解密模式（虽然代码可以一样）。

## 总结

通过这个小项目，我们学到了：

- `argc` / `argv` 接收命令行参数
- `fopen` 的二进制模式 `"rb"` / `"wb"`
- `fgetc` / `fputc` 逐字节读写文件
- XOR 运算的对称性：加密和解密同一个操作
- 用 `strtol` 做安全的字符串转数字

完整代码就在本文中，可以直接复制使用。也欢迎在此基础上修改和扩展。
