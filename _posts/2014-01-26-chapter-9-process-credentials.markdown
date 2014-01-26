---
comments: true
date: 2014-01-26 10:33:00
layout: post
title: 进程凭据(chapter 9 linux programming interface)
summary: '进程相关的用户和组ID'
tags:
- linux
---

每一个进程，都有一组与之关联的数字的用户ID和组ID。有时候这些ID被成为进程凭据（process credentials），这些ID包含：

* 实际用户ID和实际组ID
* 有效用户ID和有效组ID
* 保存的设置用户ID和保存的设置组ID
* 文件系统用户ID和组ID（linux特定的）
* 补充组ID（supplementary group IDs）

## 实际用户ID和实际组ID

实际用户和组ID标记了进程属于哪个用户或者组ID。其值是登陆用户的id和组id。作为登陆进程的一部分，shell读取登陆口令中的用户ID和组ID。同时在shell执行的程序会继承这个实际用户ID和组ID。

## 有效用户ID和有效组ID

在大多数的Unix实现中，当进程执行这种操作时，有效用户id和有效组id加上补充的组id（supplementary group IDs）联合起来决定进程所具有的权限（linux稍微不同）。

* 如果进程有效用户ID是0（root的id），拥有所有超级用户权限。这种进程被称为特权进程，许多系统进程只能被特权进程调用
* 正常情况下，进程有效用户id和有效组id与实际用户id和实际组id是一样的。

有两种方式可以修改进程的有效用户（组）id：

* 通过系统调用。
* 执行set-user-ID和set-group-ID的程序。

## set-user-ID和set-group-ID的程序

通过把进程的有效用户id和有效组id设置为可执行文件的用户id和组id，可以使进程拥有正常情况下没有的权限。

一般文件，除了表示文件拥有关系的用户id和组id外，还有两个特殊的权限位：*set-user-ID* 和 *set-group-ID*。可以通过chmod来修改这两个位：

例子：

```
# ls -l prog
-rwxr-xr-x 1 root root 
# chmod u+s prog	打开set-user-ID权限位
# chmod g+s prog	打开set-group-ID权限位
# ls -l prog
-rwsr-sr-x 1 root root 302585 Jun 26 15:05 prog
```

修改密码程序passwd，mount和Unmount等都设置色set-user-id权限位。

## 保存的Set-User-ID和保存的Set-Group-ID

保存的Set-User-ID和保存的Set-Group-ID被设计成让设置了Set-User-ID和Set-Group-ID的程序用的。执行步骤：
1. 如果执行文件的Set-User-ID和Set-Group-ID权限位被设置了。则进程有效用户id和有效组id改成改文件的用户id和组id。如果这个权限位没有设置，则没有任何改变。
2. 保存的Set-Group-ID和保存的Set-User-ID值，拷贝自进程的有效用户id和有效组id。不管Set-User-ID和Set-Group-ID权限位设不设置，拷贝过程都会执行。

如果一个进程的实际用户id、有效用户id和保存的设置用户id都是1000.当它执行了一个root拥有的set-user-ID的程序，执行后，进程的实际、有效和保存的设置用户id分别是：

1000、0、0

许多系统调用允许set-user-ID的程序在实际用户id和保存用户id之间进行切换。这时，程序可以临时性的获取或者抛弃与可执行文件相关联的用户id的特殊权限。这是一种安全的编程实践。

## 文件系统用户id和文件系统组id

在linux系统上，操作文件的权限，不是根据进程的有效用户（组）id来确定的，而是根据文件系统的用户（组）id判断，其它操作，仍然用有效用户（组）id来进行权限判断。

正常情况下，文件系统用户（组）id与进程的有效用户（组）id是一样的。当进程的有效用户（组）id被修改时，不管是通过系统调用或者是通过执行set-user（group）-ID的程序，文件系统的用户（组）id也会跟着修改成同样的值。这意味着，linux与其它unix的实现，在行为上市一致的，除了在明确调用setfsuid() and setfsgid()两个系统调用的时候。

由于在表现上，文件系统用户（组）id与进程有效的用户id和组id是一样的，因此虽然在底层，操作文件时，linux是用文件系统的用户（组）id来判断权限的，我们只关心进程的有效用户（组）id即可。

## 补充的组ids（Supplementary Group IDs）

补充的组ids是一个额外的进程所属于的组的集合。

新进程会从父进程中继承这些组ids。一个登录shell进程从系统组文件（system group file）获取其补充组ids。这些ids与有效用户（组）id来检查进程的操作权限。





