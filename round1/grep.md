# mini-grep

`grep` 大概是 Unix 世界里最常用的命令之一，它做的事情其实很朴素：**逐行读取输入，把包含指定内容的行打印出来**。

本题要求你用 C 从零实现一个 mini-grep。它**不需要支持正则表达式**——模式就是一段普通字符串（字面量），只要它出现在某一行里，这一行就算匹配。比如查找 `a.c` 只会匹配包含字面 `a.c` 的行，不会匹配 `abc`。

通过这个小项目，你将能掌握：

- **make**：会用 `gcc main.c -o main` 不等于会写 Makefile。目标、依赖、变量、增量构建、清理——一个工程是怎么被“一条命令”构建出来的。
- **文件操作**：`open` / `read` / `close` 是进程与文件系统打交道的原始接口。文件描述符是什么？`read` 为什么可能读不满？什么时候该关、怎么关、关的时候要注意什么？
- **参数解析**：`getopt` 是 C 世界命令行程序的事实标准。选项、合并选项、`--`、非法选项，这些约定要你亲手处理一遍。
- **错误处理**：文件不存在、没有权限、读到一半失败……“正常路径”之外的分支，才是真实程序的主要部分。

## 先看看真正的 grep

先别急着写代码。把系统自带的 `grep` 当作参考实现跑一跑，观察它的输出与退出码。准备一个样例文件：

```bash
cat > sample.log <<'EOF'
boot ok
disk full at 03:00
ERROR: retry later
boot ok again
error: give up
EOF
```

观察它的行为：

```console
$ grep error sample.log              # 大小写敏感
error: give up
$ grep -i error sample.log           # 忽略大小写
ERROR: retry later
error: give up
$ grep -n -i error sample.log        # 带行号
3:ERROR: retry later
5:error: give up
$ grep -c -i error sample.log        # 只计数
2
$ grep -v boot sample.log            # 反向匹配
disk full at 03:00
ERROR: retry later
error: give up
$ grep zzz sample.log; echo "exit=$?"    # 没有匹配行
exit=1
$ grep error sample.log no-such-file; echo "exit=$?"   # 其中一个文件打不开
grep: no-such-file: No such file or directory
error: give up
exit=2
```

如果你在 Linux 上，还可以用 `strace` 看看它到底调用了哪些系统调用（下面只是**示意**，具体缓冲区大小和调用次数因版本而异，请以你自己跑出来的为准）：

```console
$ strace -e trace=openat,read,close grep error sample.log 2>&1 | tail -n 4
openat(AT_FDCWD, "sample.log", O_RDONLY) = 3
read(3, "boot ok\ndisk full at 03:00\nERROR: "..., 98304) = 64
read(3, "", 98304) = 0
close(3) = 0
```

上面这些实验请亲自动手跑一遍，把你自己环境里的真实输出记进笔记。

## 题目要求

### 语言与构建

- 语言：C
- 仓库根目录必须提供 `Makefile`：执行 `make` 构建出可执行文件 `./mini-grep`，执行 `make clean` 清理所有构建产物后，`make` 仍能重新构建成功。
- **读取文件必须使用系统调用 `open(2)` / `read(2)` / `close(2)`（`<fcntl.h>`、`<unistd.h>`），**禁止**用 `fopen` / `fgets` / `getline` / `std::ifstream` / `std::filesystem` 之类的封装来读文件内容——本题就是要你亲手管理文件描述符、缓冲区与 `errno`。输出用 `write` 或 `printf` 都可以。
- 不得调用系统 `grep`（`system`、`popen`等），也不需要引入正则引擎（`std::regex`、PCRE、`regex` crate 等），本题考察的是你自己实现。
- 允许使用 string.h 库进行字符串操作

Makefile 的具体要求（面试时会当场检查）：

- `make`（默认目标）构建出 `./mini-grep`；在全新 `git clone` 出的目录里，一条 `make` 必须直接成功，不依赖任何本地路径或环境变量。
- 使用变量而不是写死命令：至少 `CC`、`CFLAGS`（或等价物），保证 `make CC=clang`、`make CFLAGS='-O2 -Wall -Wextra'` 这类调用都能生效。
- `make clean` 清掉全部构建产物（含拷贝出来的 `./mini-grep`），并声明为 `.PHONY`。构建产物不要提交到版本库，用 `.gitignore` 忽略。
- 增量构建必须正确：改了某个 `.c` 只重编它；**改了 `.h` 要重编所有依赖它的源文件**（`-MMD -MP` 自动依赖或手写依赖均可）。如果你只写了单个源文件这一条可以忽略，但我们**推荐**你把参数解析、文件读取、匹配、输出拆成独立模块。
- 构建失败时 `make` 必须返回非零（不要在命令前加 `-`、不要用 `; true` 之类掩盖失败）。
- 推荐（不强制）：提供 `make test` 运行你自己的测试脚本。

### 命令行接口

```text
mini-grep [-nic] PATTERN [FILE...]
```

| 选项 | 含义 |
| --- | --- |
| `-n` | 在每行前输出行号（从 1 开始，每个文件独立计数） |
| `-i` | 匹配时忽略 ASCII 字母的大小写 |
| `-c` | 只输出匹配行的数量，不输出行内容（与 `-n` 同时出现时忽略 `-n`） |

必须支持的调用形式：

```bash
mini-grep PATTERN FILE              # 基本匹配
mini-grep -n PATTERN FILE           # 单个选项
mini-grep -ni PATTERN FILE FILE2   # 选项可以合并，等价于 -n -i -v
```

- `PATTERN` 是字面量字符串，不是正则；空字符串 `""` 视为匹配所有行。
- 必须处理 `-n`、`-i`、`-v`、`-c` 之外的**非法选项**，例如 `mini-grep -x ...`。选项写在 PATTERN 之前即可（GNU `getopt` 默认会把选项重新排列，POSIX 行为则是遇到第一个非选项就停止，这个差异请你自己查清楚）。
- 缺少 `PATTERN`、出现非法选项都属于用法错误。

### 输出格式

| 情形 | 每行格式 |
| --- | --- |
| 单个文件 | `行内容` |
| 单个文件 + `-n` | `行号:行内容` |
| 多个文件 | `文件名:行内容` |
| 多个文件 + `-n` | `文件名:行号:行内容` |
| `-c`，单个输入 | `数量` |
| `-c`，多个文件 | `文件名:数量` |

- 行内容原样输出，不裁剪、不改写；文件名按命令行中给出的原样输出。
- 只要命令行里给出的 `FILE` 多于一个，就带文件名前缀——即使其中某个文件打开失败，其他文件的输出也照样带前缀。
- 所有输出行都以 `\n` 结尾：若输入文件的最后一行没有换行符，匹配并输出它时要补一个。

### 行、缓冲与内存

- “一行”指以 `\n` 结尾的一段内容；文件最后一行可能没有 `\n`，它仍然是一行。
- **行长不受限制**：例如一整行有 1 MiB，也必须正确输出，不能截断、不能报错（缓冲区策略由你决定，只要满足要求）。
- **内存占用不随文件大小增长**：必须流式处理，不允许先把整个文件读进内存再逐行查找（面试会问你为什么）。Rust 的 `BufRead` 系列接口（如 `read_until`）天然满足这一点，但你要能解释它为什么不会把整个文件读进内存。
- 不要求处理二进制文件：可以假定行内不含 `\0` 字节（想挑战的话见“进阶任务”）。

### 错误处理与退出码

- 错误信息一律写到 **stderr**，必须包含 `mini-grep: ` 前缀、文件名与失败原因（C/C++ 请用 `strerror(errno)` 的文本，Rust 用 `io::Error` 的显示信息），推荐格式：

  ```console
  $ ./mini-grep error sample.log no-such-file
  error: give up
  mini-grep: no-such-file: No such file or directory      # 这一行在 stderr
  ```

- 某个文件打开或读取失败：报告错误，**继续处理剩下的文件**，最终退出码为 2。
- 用法错误：向 stderr 输出用法提示并退出 2，不处理任何文件，推荐格式 `usage: mini-grep [-nivc] PATTERN [FILE...]`。
- 所有打开的文件描述符都必须关闭，包括出错提前返回的路径；`read` 返回 `-1` 且 `errno == EINTR` 时应当重试，而不是当作错误（Rust 对应 `ErrorKind::Interrupted`）。

| 退出码 | 含义 |
| --- | --- |
| 0 | 至少有一行被选中（应用完 `-v` 之后判断），且没有发生任何错误 |
| 1 | 没有任何行被选中，且没有发生任何错误（`-c` 输出 `0` 时同样是 1） |
| 2 | 发生了任何错误（用法错误，或至少一个文件/标准输入打开、读取失败）——即使其他文件有匹配行，也要返回 2 |


## 示例会话

下面仍使用前面的 `sample.log`，另外准备一个 `notes.txt`：

```text
error handling rules
all good
```

mini-grep 的期望行为：

```console
$ ./mini-grep boot sample.log
boot ok
boot ok again

$ ./mini-grep -in error sample.log
3:ERROR: retry later
5:error: give up

$ ./mini-grep -civ error sample.log notes.txt
sample.log:3
notes.txt:1

$ ./mini-grep zzz sample.log; echo "exit=$?"
exit=1

$ cat sample.log | ./mini-grep -n boot
1:boot ok
4:boot ok again

$ ./mini-grep error sample.log no-such-file; echo "exit=$?"
mini-grep: no-such-file: No such file or directory      # stderr
error: give up
exit=2

$ ./mini-grep; echo "exit=$?"
usage: mini-grep [-nivc] PATTERN [FILE...]              # stderr
exit=2
```


## 学习资料

这个列表只是起点，“遇到不懂的自己查”比“看完全部资料”更重要。

- 手册页（最权威、也最该先查）：`man 1 grep`、`man 2 open`、`man 2 read`、`man 2 close`、`man 3 getopt`、`man 3 errno`、`man 1 make`
- [GNU make 官方手册](https://www.gnu.org/software/make/manual/make.html)，中文可看[《跟我一起写 Makefile》](https://seisman.github.io/how-to-write-makefile/)
- [glibc 手册：Getopt](https://www.gnu.org/software/libc/manual/html_node/Getopt.html)，与 `man 3 getopt` 配合阅读，重点搞清楚 `optind` / `optarg` / `optopt` / `opterr`、`?` 的两种触发情况、`--` 的处理，以及 GNU 对操作数之后选项的“重排”行为
- [CSAPP](https://hansimov.gitbook.io/csapp) 第 10 章 系统级 I/O（文件描述符、`open` / `read` / `close`、短读）
- C 语言基础见 [Wiki 上的 C 语言教程](https://wiki.lingrui.studio/summer_camp/c/)；环境与 Shell 见[夏令营](https://wiki.lingrui.studio/summer_camp/)
- 想看别人怎么写：GNU grep 或 busybox 的源码（可以读，但提交必须是自己的实现）
- 本机没有 `man` 时可以用 [man7.org 的在线手册](https://man7.org/linux/man-pages/)

## 提交说明

**本题要求在 Linux 环境中完成**（WSL、虚拟机、物理机均可，与一轮整体要求一致）。

- 在一个 GitHub 仓库中提交你的工程，建议结构如下：

  ```text
  mini-grep/
  ├── Makefile
  ├── src/          # 源码，建议按模块拆分
  ├── tests/        # 测试脚本与各边界用例的输入文件
  └── README.md
  ```

- `README.md` 至少包含：构建方法（一条 `make` 即可）、使用说明、至少 3 个带真实输出的运行示例、设计说明（模块划分、如何做到“行长不限 + 内存不随文件增长”）、**错误处理表**（列出你处理的所有错误情况、对应行为与退出码）、已知限制。
- 题解里说明你的实现思路，测试结果和遇到的难点，在题解中随附 Github 仓库链接，保证仓库可访问。
- 可以用 AI 辅助学习、查资料、审查代码，但**不要**让 AI 直接替你写完再原样提交；面试会就你提交的代码追问。
