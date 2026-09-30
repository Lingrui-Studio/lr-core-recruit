# 课题一：Vector

> 预计耗时：1 天

## 任务介绍

在前面的一轮中，我们已经学习过了 C 语言内存管理的**理论基础**，现在就让我们来实践检验一下吧——用 C 语言实现一个 `std::vector`！

学习过 C 语言中的数组，你应该知道它有多么不好用：

- 容量固定，初始时给多少空间，以后就只能用多少空间
- 不包含 `size` 信息，经常需要用其他变量专门负责记录其中元素个数、容量大小等信息
- 传参时退化为指针

vector 是 C++ 标准模板库（STL）中最常用、最核心的容器之一。它解决了上述的几个问题，可以方便地在尾部添加新元素、获取元素数量和容量大小、超出容量自动扩容等。请前往 [vector 原理可视化页面](https://lingrui-studio.github.io/vector-playground/) 体验并了解 vector 的原理。

## 前置知识

- 一轮：【内存管理】
- 按文档 [C 语言开发环境](https://wiki.lingrui.studio/summer_camp/c/) 先配置好 C 语言开发环境

## 学习要求

- 了解 C++ 中 `std::vector` 的用处、用法和原理（**不需要你学习 C++ 或是 STL**）
- 简单了解 `make` 构建系统（Linux 中最经典和常用的构建系统）与时间复杂度
- 完成 [lr-core-vector](https://github.com/Lingrui-Studio/lr-core-vector) 练习仓库
  - Star 并 Fork 创建自己的个人 vector 仓库
  - 克隆到本地（Linux 环境）并细看 readme
  - 完成后将所有修改提交 commit 并 push 到 GitHub 上你的个人 vector 仓库
  - 按图中步骤 **在个人 vector 仓库中** 手动触发一次自动评分工作流 ![actions](https://recruitstatic.lingrui.studio/lingrui/images/2026/09/a7214e1802f2a83e2fa1d4e963c5cf23d90ca399a9ba3bcd32a887626a1dc4d6.png)

## 提交要求

- 个人 vector 仓库链接
- 学习总结与心得体会（不要长篇大论）
- 参考资料

> 有问题请联系出题人：@
