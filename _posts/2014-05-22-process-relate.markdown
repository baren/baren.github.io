---
comments: true
date: 2014-05-04 20:00:00
layout: post
title: apue(进程关系)
summary: 'apue chapter9'
tags:
- linux
- 笔记
---


# 终端登录

由终端登录至unix，这个过程是类似的，而与所使用的终端无关，终端可以是基于字符的终端、仿真简单的基于字符终端的图形终端，或者是运行窗口系统的图形终端。

终端登录步骤：

1. 系统自举时，创建进程ID为1的init进程。使系统进入多用户状态。
2. init进程读取文件/etc/ttys，对每一个允许登录的终端设备，init进程调用一次fork，fork后的子进程执行getty程序（exec getty程序）
3. getty程序会调用open函数，以读写方式打开终端，一旦打开设备，则文件描述符0、1、2设置到该设备，输出login:之类的信息
4. gettty调用login函数（```execle("/bin/login", "login", "-p", username, (char *) 0, envp);```）
5. login执行多个工作
	> 验证用户、密码正确性，连续几次不对，则退出；父进程init知道子进程终止后，会再次调用fork，执行getty，重复上述动作。登录成功后：
	> 将当前工作目录改为该用户的起始目录（home目录）
	> chown改变该终端的所有权，使登录用户成为它的所有者
	> 对该终端设备的访问权限改为用户读写
	> 调用setgid及initgroups设置进程的组id
	> 用login所得到的所有信息初始化环境
	> 调用该登录用户的登录shell

# 网络登录

