---
title: '佬们问题记录本'
author: '安'
description: '学长在群里问的问题及其回答，咱偷偷记下来'
publishDate: '2025-08-29'
updatedDate: '2025-08-29'
tags:
  - java
  - 反射
language: 'Chinese'
draft: false
heroImage: { src: './0.jpg', color: '#004D17' }
---
## Q
1. [mysql为啥要搞一个redologbuffer， 而不直接把redolog文件通过文件映射到用户态直接写？](#mysql-为什么要有-redo-log-buffer)
2. [询问各位佬一个问题，反射调方法和正常调方法有什么区别？](#java-反射调用和正常调用有什么区别)
3. [raft和zab的区别是什么](#raft和zab的区别)

---
## A

### MySQL 为什么要有 redo log buffer？
MySQL 之所以要有 redo log buffer，本质上是 **数据库不能把日志写盘这件事交给内核**。  
redo log 是小而频繁的顺序写，如果直接 mmap 文件，就会变成内核按脏页策略去刷，可能出现部分页写，顺序和完整性都不可控。  
而用 buffer，InnoDB 能自己控制：先把日志格式化、对齐，再在事务提交时统一 fsync，做 group commit。  
这就是数据库要的东西——既能保证崩溃恢复完整，又能批量控制 IO 成本。

---

### Java 反射调用和正常调用有什么区别？
反射和普通调用的差别，本质上是 **静态绑定和动态分派**。  
普通调用在编译期就能确定目标，JIT 可以内联，几乎就是一条直接指令；  
而反射 invoke 要在运行时查找方法、做参数封装，等于是绕了一层动态分派，所以性能差一个数量级。  
换句话说，反射是牺牲性能换来灵活性。

---
### raft和zab的区别
Raft 和 Zab 的核心区别在于 **Leader 选举和日志一致性处理时机**。Raft 在选举阶段就保证新 leader 的日志是最新的，候选人只有日志最完整才能当选，这样一旦 leader 产生，日志天然连续，只需要追加即可。而 Zab 则是先选出 leader，再由 leader 与大多数节点比对、修复日志，保证一致性。

所以本质上，Raft 是 **“选举先裁剪，日志后追加”**，保证 leader 永远是日志最新节点，协议更直观易理解；Zab 是 **“选举先定，日志后同步”**，保证广播线性化和强一致性，但恢复时需要更多同步步骤。这个差别也导致 Raft 更适合通用分布式系统，而 Zab 主要优化 ZooKeeper 的事务和元数据管理场景。

---
