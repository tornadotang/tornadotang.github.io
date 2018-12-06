---
title: (kubernetes) overview
category: k8s
draft: true
---

# overview

orchestration 王者

# 核心概念

## pod

在 kubernetes 中，能够被创建，调度和管理的最小单元是 pod，而非单个容器。一个 pod 是有若干个 Docker 容器构成的容器组，pod 里的容器共享 network namespace，并通过 volume 机制共享一部分存储


## repliction controller

它决定一个 pod 有多少同时运行到副本，并保证这些副本的期望状态与当前状态一致

# architecture

![](/asserts/k8s.png)


