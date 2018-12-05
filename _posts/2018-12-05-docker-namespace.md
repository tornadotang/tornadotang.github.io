---
title: (docker) namespace
category: docker
draft: true
---

# namespace 资源隔离

namespace | 系统调用参数 | 隔离内容
:---: | :--: | :-:
UTS | CLONE_NEWUTS | 主机名与域名
IPC | CLONE_NEWIPC | 信号量，消息队列和共享内存
PID | CLONE_NEWPID | 进程编号
Network | CLONE_NEWNET | 网络设备，网络栈，端口等
Mount | CLONE_NEWNS | 挂载点（文件系统）
User | CLONE_NEWUSER | 用户和用户组

# 进行 namespace API 操作等4种方式

- 通过 clone() 在创建新进程等同时创建 namespace
+ 通过 setns() 加入一个已经存在的 namespace
* 通过 unshare() 在原先进程上进行 namespace 隔离
* fork() 系统调用

# UTS namespace

^_^

[^_^]:
    commentted-out contents
    should be shift to right by four spaces (`>>`).

[^_^]:
	some comments

[^_^]:
    some comments

[//]: <> (some com)
[//]: # (This may be the most platform independent comment)
[comment]: <> (This is a comment, it will not be included)
[comment]: <> (in  the output file unless you use it in)
[comment]: <> (a reference style link.)

^_^

# IPC namespace

# PID namespace

# network namespace

# mount namespace

# user namespace
