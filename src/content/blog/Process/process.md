---
title: '进程与线程'
author: '安'
description: '浅谈Linux内核中进程与线程长出的枝桠。'
publishDate: '2025-08-25'
updatedDate: '2025-08-25'
tags:
  - 内核
  - 进程
  - 线程
  - 系统调用
  - 操作系统
language: 'Chinese'
draft: false
heroImage: { src: './0.jpg', color: '#E43A3A' }
---

在早期操作系统 **Unix** 中，只有 **进程** 作为最小调度单位。  
但进程过重，资源开销大，于是引出了 **线程**。
而后进程作为资源分配的基本单位，线程则作为 CPU 调度的基本单位

线程更轻盈，可以称它为 *进程中的进程*，或者说 **轻量级进程**。

---
## 线程本质
在Linux中，线程就是共享内存的进程，通过 `clone()` 的 `flag` 区分。
<details> <summary>不妨看看精简源码</summary>


```c
// ========== 用户态 ==========
int pthread_create(...) {
    return clone(start_routine, stack,
                 CLONE_VM | CLONE_FS | CLONE_FILES |
                 CLONE_SIGHAND | CLONE_THREAD,
                 arg);
}

// ========== 内核态 ==========
SYSCALL_DEFINE5(clone, unsigned long flags, unsigned long newsp,
                int __user *parent_tidptr,
                unsigned long tls,
                int __user *child_tidptr)
{
    return _do_fork(flags, newsp, 0, parent_tidptr, child_tidptr, tls);
}

long _do_fork(unsigned long flags, unsigned long stack, ...) {
    struct task_struct *p;
    p = copy_process(flags, stack, ...); // 复制/共享资源
    wake_up_new_task(p);                 // 放入调度器
    return pid;
}

static struct task_struct *copy_process(unsigned long flags, ...) {
    struct task_struct *p = dup_task_struct(current);
    if (!(flags & CLONE_VM))
        p->mm = dup_mm(current->mm);   // 独立地址空间 => 进程
    else
        p->mm = current->mm;           // 共享地址空间 => 线程
    // 文件表、信号处理器等也可选择共享
    ...
    return p;
}

````

</details>

---

## 两棵树：关系与调度
现在，不论进程还是线程，我们将其称为 **Task**，由 `task_struct` 来表示。

Linux 内核中，进程与线程被组织在两棵树中：

1. **家族树**
    - 任务关系树，用于存储 `task_struct`，建立父子关系、兄弟关系。
    - 它是一棵普通树，主要为调度树提供遍历。

2. **调度树**
    - 也称为 **CFS 红黑树**，可以看作一个队列。
    - 为了公平调度，引入虚拟时间 `vruntime`：
      ```
      vruntime += 实际运行时间 × (NICE_0_LOAD / 任务权重)
      ```
    - 树的最左节点，总是当前 `vruntime` 最小的任务。

阻塞与唤醒：
- 如果任务空转，死循环自旋，会浪费 CPU。
- 解决办法：
    - **阻塞**：任务主动移出红黑树。
    - **唤醒**：任务重新加入红黑树。
    - 阻塞不是消极等待，而是积极地减轻调度树的负担。

---

## 同步与竞争：如何协调任务

当多个任务同时访问共享资源时，需要同步机制来避免冲突：

| 原语 | 行为 | 特点 |
|------|------|------|
| **mutex** | 抢到锁的任务执行；未抢到的被移出调度树 | 常见互斥锁 |
| **semaphore** | 控制并发数量，超过上限的任务被阻塞 | 适合控制资源池 |
| **spinlock** | 不离开调度树，自旋等待 | 用于内核关键路径 |
| **futex** | 用户态先尝试原子操作，失败才陷入内核 | 减少系统调用开销 |

---

## 创建与写时拷贝

任务的创建主要通过 `clone()` 和 `fork()` 完成。  
其中 `fork()` 会创建一个新进程，并与父进程共享以下资源：

- 相同的代码段
- 几乎一致的数据段、堆、栈
- 相同的文件描述符、信号处理器等

这种直接复制显然代价过大，因此引入 **写时拷贝（Copy-On-Write, COW）**。

### 写时拷贝机制
- fork 初始阶段，父子进程完全共享页表。
- 子进程页表映射到与父进程相同的物理地址，页表标记为 **只读**，并打上 **COW 标志**。
- 当父进程或线程尝试写入共享页时：
    1. CPU 检查权限，发现写只读页 → 触发 **Page Fault**
    2. 内核处理中断，检测到 **COW 标志**
    3. 分配新物理页，将原内容复制过去
    4. 修改页表项，映射新地址，并允许写入
    5. 继续执行写操作

总结：
- **第一次写入才会触发复制**，大幅节省内存，加快 `fork`。
- 若频繁写入，性能可能退化；且初次写入必然有额外开销。

### 调用链
```text
fork() -> exec() -> do_page_fault()
   -> { readOnly + Cow }
   -> alloc_page
   -> copy_user_page
   -> ptep_set_wrprotect()
   -> ptep_clear_flush
```

---
## 小结
总的来说，进程与线程在 Linux 内核中都以 task_struct 表示，区别只在资源共享的范围。它们通过调度器获得公平的 CPU 时间，通过同步原语避免冲突，通过 COW 等机制降低资源开销。进程与线程的博弈与协作，共同构成了现代操作系统运行的基石。