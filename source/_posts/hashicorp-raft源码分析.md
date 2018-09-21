---
title: hashicorp/raft源码分析
date: 2018-09-19 16:29:05
tags:
---

[https://github.com/hashicorp/raft](https://github.com/hashicorp/raft/tree/82694fb663be3ffa7769961ee9a65e4c39ebbf2c)

了解源码之前，
- 首先知道该项目是用来干什么的
- 然后，应该能够正常使用它
- 最后，才是了解源码，了解内部实现。

`hashicorp/raft` 是一个Go语言的库，用来管理日志复制。它可以和FSM一起管理复制状态机。`hashicorp/raft` 是实现一致性协议的库。
