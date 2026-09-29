# 课题四：Shell Lab

你一定用过 shell：在终端里敲一行命令、按回车，程序就跑起来了；`Ctrl-C` 能终止运行中的程序，`Ctrl-Z` 能挂起进程，`fg` / `bg` 再把它调回来。可你有没有想过，shell 背后的实现原理是怎样的？

这一章你要自己写一个 shell——TSH(Tiny SHell)。它要能解析命令、`fork` 出子进程、`execve` 执行程序，还要处理前后台作业和 `SIGINT` / `SIGTSTP` / `SIGCHLD` 这些信号，并且复现真正的 shell 在作业控制上的行为。做完这道题，你对"进程"和"信号"的理解就不再是书上那两页纸了。

任务来自 CMU 的经典实验 **Shell Lab**。[在此](https://csapp.cs.cmu.edu/3e/labs.html)下载 README 和 Self-Study Handout 即可。

> 好好读官方 writeup。

## 学习要求

学习 CSAPP 第 8 章 **异常控制流**（异常、进程、系统调用、进程控制、信号、非本地跳转）并完成 Shell Lab。

## 提交要求

- 个人 Shell Lab 的 GitHub 仓库链接（注意 commit 前先 `make clean`）
- 实验结果：附上你的 tsh 在各条官方 trace 上与参考实现对比的结果（脚本自带 16 条 trace 和 `tshref`）
- 完成这道 lab 的经历：心路历程、阶段成果、踩过的坑……（不要知识点的堆砌）
- 参考资料
