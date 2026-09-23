# lr-grep

`grep` 大概是 Unix 世界里最常用的命令之一，它做的事情简单：**逐行读取输入，把包含指定内容的行打印出来**。

在这道题中，你将实现一个简化版的 `grep`，体验编写系统常用工具的乐趣。


## 先看看真正的 grep

在你的Linux环境上，准备一个样例文件：

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
$ grep zzz sample.log; echo "exit=$?"    # 没有匹配行
exit=1
$ grep error sample.log no-such-file; echo "exit=$?"   # 其中一个文件打不开
grep: no-such-file: No such file or directory
error: give up
exit=2
```

你可以使用man命令查看 `grep` 的手册页，了解它的更多选项和用法，但本题要求的实现不会涉及那样多的内容。


## 题目要求

### 语言与构建

- 语言：**C**（C99 及以上即可，推荐 `-std=gnu11`）；
- 构建方式不限：写 `Makefile` 也好，直接给编译命令也好，但要在 `README.md` 里写清楚；
- **读取文件必须使用系统调用 `open(2)` / `read(2)` / `close(2)`**（`<fcntl.h>`、`<unistd.h>`），**禁止**用 `fopen` / `fgets` / `getline` 之类的 stdio 封装来读文件内容，你需要手动管理文件描述符、缓冲区与 `errno`。输出用 `write` 或 `printf` 都可以，`<string.h>` 可以正常使用。
- **参数解析要求手写**：自己遍历 `argv` 完成选项解析，不要调用 `getopt(3)` / `getopt_long(3)`，具体规则见下文
- 不得调用系统 `grep`（`system`、`popen` 等）；本题是字面量匹配，也不需要引入任何正则引擎。


### 命令行接口

你的代码需要能交付一个标准 ELF 可执行文件 `lrg`，它的调用形式如下：

```text
lrg [-nic] PATTERN [FILE...]
```

| 选项 | 含义 |
| --- | --- |
| `-n` | 在每行前输出行号（从 1 开始，每个文件独立计数） |
| `-i` | 匹配时忽略 ASCII 字母的大小写 |
| `-c` | 只输出匹配行的数量，不输出行内容（与 `-n` 同时出现时忽略 `-n`） |

必须支持的调用形式：

```bash
lrg PATTERN FILE               # 基本匹配
lrg -n PATTERN FILE            # 单个文件
lrg -ni PATTERN FILE FILE2     # 选项可以合并，等价于 -n -i
lrg PATTERN                    # 省略 FILE 时读取标准输入
```

- `PATTERN` 是字面量字符串；空字符串 `""` 视为匹配所有行。
- 选项解析规则（自己遍历 `argv` 实现）：
    - 从左往右扫描，遇到第一个不以 `-` 开头的参数就停止，它就是 `PATTERN`，其后的参数全是 `FILE`；
    - 选项可以合并：`-ni` 等价于 `-n -i`；
    - 单独的 `-` 不是选项，按普通参数处理；其余无法识别的选项（如 `-x`、`-nix`）都是**非法选项**。
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

- 行内容原样输出；文件名按命令行里给出的原样输出。
- 只要命令行里给出的 `FILE` 多于一个，就带文件名前缀——即使其中某个文件打开失败，其他文件的输出也照样带前缀。
- 省略 `FILE` 时按“单个输入”处理，不加文件名前缀。
- 所有输出行都以 `\n` 结尾：若输入文件的最后一行没有换行符，匹配并输出它时要补一个。

### 行、缓冲与内存

- “一行”指以 `\n` 结尾的一段内容；文件最后一行可能没有 `\n`，它仍然是一行。
- **行长不受限制**：例如一整行有 1 MiB，也必须正确输出，不能截断、不能报错。
- **内存占用不随文件大小增长**：必须流式处理，不允许先把整个文件读进内存再逐行查找。
- 动态分配的内存必须全部 `free`；
- 本题不要求处理二进制文件：可以假定你的输入是纯文本文件或stdin。


### 错误处理与退出码

- 错误信息一律写到 **stderr**，必须包含 `lrg: ` 前缀、文件名与失败原因（用 `strerror(errno)` 取到原因文本），推荐格式：

  ```console
  $ ./lrg error sample.log no-such-file
  error: give up
  lrg: no-such-file: No such file or directory      # 这一行在 stderr
  ```

- 某个文件打开或读取失败：报告错误，**继续处理剩下的文件**，最终退出码为 2。
- 用法错误：向 stderr 输出用法提示并退出 2，不处理任何文件，推荐格式 `usage: lrg [-nic] PATTERN [FILE...]`。
- 所有打开的文件描述符都必须关闭，包括出错提前返回的路径；`read` 返回 `-1` 且 `errno == EINTR` 时应重试，而不是当作错误。

| 退出码 | 含义 |
| --- | --- |
| 0 | 至少有一行被选中，且没有发生任何错误 |
| 1 | 没有任何行被选中，且没有发生任何错误（`-c` 输出 `0` 时同样是 1） |
| 2 | 发生了任何错误（用法错误，或至少一个文件/标准输入打开、读取失败）——即使其他文件有匹配行，也要返回 2 |


## 示例会话

下面仍使用前面的 `sample.log`，另外准备一个 `notes.txt`：

```text
error handling rules
all good
```

lrg 的期望行为：

```console
$ ./lrg boot sample.log
boot ok
boot ok again

$ ./lrg -in error sample.log
3:ERROR: retry later
5:error: give up

$ ./lrg -ci error sample.log notes.txt
sample.log:2
notes.txt:1

$ ./lrg zzz sample.log; echo "exit=$?"
exit=1

$ cat sample.log | ./lrg -n boot
1:boot ok
4:boot ok again

$ ./lrg error sample.log no-such-file; echo "exit=$?"
lrg: no-such-file: No such file or directory      # stderr
error: give up
exit=2

$ ./lrg; echo "exit=$?"
usage: lrg [-nic] PATTERN [FILE...]               # stderr
exit=2
```


## 学习资料


- 手册页（最权威、也最该先查）：`man 1 grep`、`man 2 open`、`man 2 read`、`man 2 close`、`man 3 errno`、`man 1 make`
- [GNU make 官方手册](https://www.gnu.org/software/make/manual/make.html)，中文可看[《跟我一起写 Makefile》](https://seisman.github.io/how-to-write-makefile/)
- [POSIX 命令行工具的语法约定](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap12.html)：选项为什么能像 `-ni` 这样合并，规则就出自这里
- [CSAPP](https://hansimov.gitbook.io/csapp) 第 10 章系统级 I/O（文件描述符、`open` / `read` / `close`、短读）
- 内存自查工具：[Valgrind 快速入门](https://valgrind.org/docs/manual/quick-start.html)，编译器自带的 `-fsanitize=address,undefined` 也能发现大部分问题
- 想看别人怎么写：GNU grep 或 busybox 的源码（可以读，但提交必须是自己的实现）
- 本机没有 `man` 时可以用 [man7.org 的在线手册](https://man7.org/linux/man-pages/)

## 提交说明

**本题要求在 Linux 环境中完成**


- 在题解中说明你的实现思路、测试结果和遇到的难点，并随附你的 GitHub 仓库链接，保证仓库可访问。仓库建议结构如下：

  ```text
  lrg/
  ├── src/          # 源码，建议按模块拆分
  ├── tests/        # 测试脚本与各边界用例的输入文件
  └── README.md     # 交付文档
  ```

- `README.md` 要当成一份**交付文档**来写：结构清晰、有条理，别人不看你的代码，照着它就能把程序构建起来、跑起来，并看懂你的实现思路。
- 内容至少包含：构建方法（写清具体命令，照着能一次成功）、使用说明、至少 3 个带真实输出的运行示例、设计说明（模块划分、如何做到“行长不限 + 内存不随文件增长”）、**错误处理表**（列出你处理的所有错误情况、对应行为与退出码）、已知限制。
- 可以用 AI 辅助学习、查资料、审查代码，但我们拒绝一切直接由AI Agent生成的代码或文档。
- **参考资料与 AI 对话链接**：在文末列出实际阅读过的资料，以及与 AI 的对话链接，格式如下：

```markdown
## 参考资料

- [C 语言教程|菜鸟教程](https://www.runoob.com/cprogramming/c-tutorial.html)
- [为什么 C 语言字符串变量的长度总是要 +1](https://qianwen.my.cn/share/chat/8376c88ba3fc48809f26e51aad9ea3a7)
- [nixos 中的 home manager 推荐使用吗](https://qianwen.my.cn/share/chat/6ba2d99116674218be5bf294d72ad930)
```
