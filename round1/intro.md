# 楔子

## 介绍

底层方向较详细的介绍请见 [招新方向 - Lingrui-Wiki](https://wiki.lingrui.studio/directions/)，不管你有没有看过都最好再读一遍。

## 引入

我们写程序时，经常会遇到一些与预期不符的“bug”，比如下面这个简单的例子：

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    for (int i = 0; i < 2; i++) {
        fork();
        printf("hello\n");
    }
}
```

这个程序做的事情很简单，就是循环`fork`自我复制并`printf`输出`hello`，运行结果如下：

```bash
~/proj/ctemplate/examples main* ❯ gcc fo.c -o fo
~/proj/ctemplate/examples main* ❯ ./fo
hello
```

可以看到输出了 6 个`hello`，但是我们若是通过`>`将输出重定向到文件中，结果却不是 6 个，而是 8 个：

```bash
~/proj/ctemplate/examples main* ❯ ./fo > fo.txt
~/proj/ctemplate/examples main* ❯ cat fo.txt
hello
```

在此例中，若你只知道`fork`、`printf`、`>`的基本用法，而不知道它们的底层原理，也不清楚他们组合使用会产生怎样奇妙的效果，那你可能永远也无法理解为什么会出现这种情况。

它就像光的波粒二象性一样，以不同方式观测同一个现象会得到不同的结果。

再看另一个例子，这里有两个程序，分别命名为`loop1.c`和`loop2.c`，他俩分别按行优先和列优先的方式为一个二维数组赋值（实际上就是对调了`i`和`j`）：

```c
// loop1.c
#include <stdio.h>
#define NUM 10000
long long matrix[NUM][NUM];

int main() {
    long long t = 0;
    for (int i = 0; i < NUM; i++)
        for (int j = 0; j < NUM; j++)
            matrix[i][j] = t++;
    printf("%lld\n", t);
}

```

```c
// loop2.c
#include <stdio.h>
#define NUM 10000
long long matrix[NUM][NUM];

int main() {
    long long t = 0;
    for (int j = 0; j < NUM; j++)
        for (int i = 0; i < NUM; i++)
            matrix[i][j] = t++;
    printf("%lld\n", t);
}

```

分别运行这两个程序并获取它们它们运行的时间：

```bash
~/proj/ctemplate/examples main* ❯ gcc loop1.c -o loop1
~/proj/ctemplate/examples main* ❯ time ./loop1
100000000
./loop1  0.11s user 0.68s system 92% cpu 0.857 total
~/proj/ctemplate/examples main* ❯ gcc loop2.c -o loop2
~/proj/ctemplate/examples main* ❯ time ./loop2
100000000
./loop2  0.45s user 0.88s system 92% cpu 1.438 total
```

诶？为啥`loop2`的耗时是`loop1`的 4 倍多？它们做的事情完全一样啊，为什么会有这么大的差异？

这就是性能优化当中最典型的一个例子，只关注算法的时间复杂度是不够的，我们还应当考虑计算机访问内存的方式。

## 前置知识

> 说是前置，其实就是一轮的一部分，只是不需要提交或在其他地方提交。

由于完成我们的招新题需要借助 GitHub，且有题目需要在 Linux 操作系统下完成，所以你必须学习 [夏令营 - Lingrui-Wiki](https://wiki.lingrui.studio/summer_camp/) 中的**所有**内容并完成“Basic”部分的题目，注册好自己的 GitHub 账号，并搭建好 Linux 环境（可以是 WSL、虚拟机等）。

Git 与 GitHub、Linux 与 Shell 不要求你能全部掌握，理解 60% 即可，遇到不懂的搜一下，后面你会越来越熟练的，因为我们后面的**所有题目**都要求**在 Linux 环境中完成**。

## 覆盖范围

本次招新一轮笔试将覆盖以下内容

- C 语言入门与简单的应用程序开发
- 基本的计算机组成原理、体系结构
- 基本的内存管理
- 数据的二进制表示

> 我们会在二轮接触到汇编、OS、ABI、并发、网络等更多有趣的内容。

## 学习资料

底层方向的招新将主要围绕《深入理解计算机系统（CSAPP）》展开，你可以将它作为主要参考书，招新群群文件里已上传电子版。

- [深入理解计算机系统（CSAPP）](https://hansimov.gitbook.io/csapp)： 个人整理的《深入理解计算机系统》中文电子版（原书第 3 版）与实验材料，但中间 2~6 章内容缺失
- [【CSAPP-深入理解计算机系统】1-1. 计算机系统漫游](https://www.bilibili.com/video/BV1cD4y1D7uR/)：CSAPP 中文视频教程
- [计算机系统漫游](https://fengmuzi2003.gitbook.io/csapp3e)：CSAPP 重点解读
- [C 语言编程入门 - Lingrui-Wiki](https://wiki.lingrui.studio/summer_camp/c/#c_2)：Wiki 上的 C 语言入门教程

需要注意的是，**寻找适合自己的学习资料**也是我们想考察的一项重要能力，所以若你觉得看书效率太低了或是不能很好地理解，都可以自己找其他的学习资料。优先推荐在浏览器中搜索关键词找比较系统性的书籍、技术博客、和知乎文章，当然你也可以问 AI 有没有合适的资料。

## 入围要求

我们只按能力划线，无人数限制。

入围一轮面试的要求：

- “Basic”部分得到至少 60% 的的原始分数
- “Core”部分得到至少 10000 原始分数

通过一轮面试进入二轮的条件：

- 诚实守信不作弊，遵守 AI 使用规范
- 你在一轮笔试中提交的东西至少掌握了 90%
- 通过了现场代码题（比如解释原理、推测结果等）

## 本题要求

好了，上面都是“底层方向”大的要求，下面讲讲在这道题。

**学习内容**

认真阅读 CSAPP 第 1 章 计算机系统漫游，对计算机系统有个大概的了解即可，第 1 章基本就是 CSAPP 全书的总览，读完后你就对后面一二轮的内容有更清晰的认识了。

**提交要求**

由于第 1 章只是总览，各方面都有所涉及却又不深入，我们干脆也不要求你花大力气写什么读书笔记了（当然你要是自己想写也可以，但不用提交）。只需要用群文件里 CSAPP 中文版中 **1.4.2 节 运行 hello 程序** 这一节的第一个**英语单词**替换`lr_studio{xxx}`中的`xxx`后作为本题答案提交即可。
