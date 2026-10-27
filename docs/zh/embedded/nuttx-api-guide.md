---
title: NuttX API 全量接口使用指南
date: 2026-10-27
description: NuttX 操作系统 12 大类约 130+ 个 C 接口的速查与使用说明。
tags: [NuttX, RTOS, API, 嵌入式]
---

# NuttX API 全量接口使用指南

> **来源文档**: NuttX Operating System User's Manual  
> **作者**: Gregory Nutt  
> **最后更新**: 2009年12月13日  
> **总页数**: 134页  
> **接口分组**: 12大类，覆盖约130+个C可调用接口  
> **目标读者**: 固件开发者

---

## 目录

- [1. 文档概述](#nuttx-api-全量接口使用指南-1-文档概述)
- [2. Task Control 接口（任务控制）](#nuttx-api-全量接口使用指南-2-task-control-接口任务控制)
- [3. Task Scheduling 接口（任务调度）](#nuttx-api-全量接口使用指南-3-task-scheduling-接口任务调度)
- [4. Task Switching 接口（任务切换）](#nuttx-api-全量接口使用指南-4-task-switching-接口任务切换)
- [5. Named Message Queue 接口（命名消息队列）](#nuttx-api-全量接口使用指南-5-named-message-queue-接口命名消息队列)
- [6. Counting Semaphore 接口（计数信号量）](#nuttx-api-全量接口使用指南-6-counting-semaphore-接口计数信号量)
- [7. Watchdog Timer 接口（看门狗定时器）](#nuttx-api-全量接口使用指南-7-watchdog-timer-接口看门狗定时器)
- [8. Clocks and Timers 接口（时钟与定时器）](#nuttx-api-全量接口使用指南-8-clocks-and-timers-接口时钟与定时器)
- [9. Signal 接口（信号）](#nuttx-api-全量接口使用指南-9-signal-接口信号)
- [10. Pthread 接口（POSIX线程）](#nuttx-api-全量接口使用指南-10-pthread-接口posix线程)
- [11. Environment Variables 接口（环境变量）](#nuttx-api-全量接口使用指南-11-environment-variables-接口环境变量)
- [12. File System 接口（文件系统）](#nuttx-api-全量接口使用指南-12-file-system-接口文件系统)
- [13. Network 接口（网络）](#nuttx-api-全量接口使用指南-13-network-接口网络)
- [14. OS Data Structures（OS数据结构）](#nuttx-api-全量接口使用指南-14-os-data-structuresos数据结构)
- [15. 关键风险与注意事项汇总](#nuttx-api-全量接口使用指南-15-关键风险与注意事项汇总)
- [16. POSIX 兼容性偏离汇总](#nuttx-api-全量接口使用指南-16-posix-兼容性偏离汇总)
- [17. 配置宏速查表](#nuttx-api-全量接口使用指南-17-配置宏速查表)

---

## 1. 文档概述

NuttX 是一个平坦地址空间（flat address space）的 RTOS，不支持像 Linux 那样的进程模型，只支持在同一地址空间内运行的简单线程。编程模型区分 **task**（具有独立性的线程）和 **pthread**（共享资源的线程）。

**核心区别**：
- **Task**：创建时复制前三个文件描述符（stdin/stdout/stderr），子任务可独立关闭或重定向，文件操作互不影响
- **Pthread**：与父线程共享文件描述符，文件操作会影响同源所有线程

**调度模型**：
- 默认严格优先级调度：高优先级任务独占CPU直到阻塞，同优先级FIFO
- 可选轮转调度（SCHED_RR）：同优先级任务通过时间片共享CPU，间隔由 `CONFIG_RR_INTERVAL` 设定

**文档约定**：每个接口按以下格式描述——函数原型、描述、输入参数、返回值、假设/限制、POSIX兼容性

---

## 2. Task Control 接口（任务控制）

### 2.1 task_create

```c
#include <sched.h>
int task_create(char *name, int priority, int stack_size, main_t entry,
                const char *argv[]);
```

**描述**：创建并激活一个新任务，指定优先级，返回系统分配的任务ID。入口函数在C环境建立后被调用，带四个参数。若入口函数返回，自动调用 `exit()`。参数通过 `strdup` 复制，不依赖调用者字符串生命周期。新任务不继承调度特性（使用默认优先级和SCHED_FIFO），但继承前三个文件描述符。

| 参数 | 类型 | 说明 |
|------|------|------|
| `name` | `char *` | 新任务名称 |
| `priority` | `int` | 新任务优先级 |
| `stack_size` | `int` | 栈大小（字节） |
| `entry` | `main_t` | 任务入口函数 |
| `argv` | `const char *[]` | 参数数组，最多 `CONFIG_MAX_TASK_ARG` 个，以NULL结尾；无参数时可传NULL |

**返回值**：非零任务ID（成功）或 `ERROR`（失败，内存不足或无法创建，errno不设置）

**POSIX兼容性**：**非POSIX接口**。类似 VxWorks `taskSpawn()`，差异：无options参数、支持可变数量参数（VxWorks仅10个）

---

### 2.2 task_init

```c
#include <sched.h>
int task_init(_TCB *tcb, char *name, int priority, uint32_t *stack,
              uint32_t stack_size, main_t entry, const char *argv[]);
```

**描述**：初始化任务控制块（TCB），为启动新线程做准备。执行 `task_create()` 的子集功能，但**不激活任务**（需调用 `task_activate()`）。**不推荐常规使用**，`task_create()` 是首选。

| 参数 | 类型 | 说明 |
|------|------|------|
| `tcb` | `_TCB *` | 新任务的TCB地址 |
| `name` | `char *` | 任务名称（未使用） |
| `priority` | `int` | 优先级 |
| `stack` | `uint32_t *` | 预分配栈起始地址 |
| `stack_size` | `uint32_t` | 栈大小（字节） |
| `entry` | `main_t` | 入口函数 |
| `argv` | `const char *[]` | 参数数组 |

**返回值**：`OK`（成功）或 `ERROR`（无法分配唯一任务ID，errno不设置）

**POSIX兼容性**：**非POSIX接口**。类似 VxWorks `taskInit()`

---

### 2.3 task_activate

```c
#include <sched.h>
int task_activate(_TCB *tcb);
```

**描述**：激活由 `task_init()` 创建的任务。未激活的任务不会被调度器执行。**不推荐常规使用**。

| 参数 | 类型 | 说明 |
|------|------|------|
| `tcb` | `_TCB *` | 任务的TCB（同 `task_init` 参数） |

**返回值**：`OK`（成功）或 `ERROR`（无法激活，errno不设置）

**POSIX兼容性**：**非POSIX接口**。类似 VxWorks `taskActivate()`

---

### 2.4 task_delete

```c
#include <sched.h>
int task_delete(pid_t pid);
```

**描述**：使指定任务终止存在——释放其栈和TCB。`task_create()` 的伴随函数。

| 参数 | 类型 | 说明 |
|------|------|------|
| `pid` | `pid_t` | 要删除的任务ID，0表示调用任务自身 |

**返回值**：`OK`（成功）或 `ERROR`（pid不对应任何任务，errno不设置）

⚠️ **风险提示**：`task_delete()` 必须谨慎使用——如果被删任务持有资源（已分配内存、信号量等），将导致资源泄漏

**POSIX兼容性**：**非POSIX接口**。类似 VxWorks `taskDelete()`，差异：不支持删除钩子（`taskDeleteHookAdd()` 不支持）、不支持自删除（用 `_exit()`）

---

### 2.5 exit / _exit

```c
#include <sched.h>
void exit(int code);
#include <nuttx/unistd.h>
void _exit(int code);
```

**描述**：使调用任务终止——释放栈和TCB。`exit()` 会刷新流、关闭文件描述符、执行 `atexit()` 注册的函数；`_exit()` 不执行这些清理。

| 参数 | 类型 | 说明 |
|------|------|------|
| `code` | `int` | 退出码（**被忽略**） |

**返回值**：无

**POSIX兼容性**：等价于 ANSI `exit()` 和 UNIX `_exit()`。差异：`code` 参数被忽略

---

### 2.6 task_restart

```c
#include <sched.h>
int task_restart(pid_t pid);
```

**描述**："重启"任务——先终止，再用相同ID、优先级、原始入口点、栈大小和参数重新初始化。

| 参数 | 类型 | 说明 |
|------|------|------|
| `pid` | `pid_t` | 要重启的任务ID，0表示调用任务 |

**返回值**：`OK`（成功）或 `ERROR`（任务ID无效或无法重启）

⚠️ **限制**：不支持重启当前运行任务（pid=0 或调用任务的pid 功能未实现）

**POSIX兼容性**：**非POSIX接口**。类似 VxWorks `taskRestart()`

---

### 2.7 getpid

```c
#include <unistd.h>
pid_t getpid(void);
```

**描述**：返回调用任务的任务ID。在中断级别调用返回的ID无效。

**参数**：无

**返回值**：调用任务的任务ID

**POSIX兼容性**：与同名POSIX接口兼容

---

## 3. Task Scheduling 接口（任务调度）

### 3.1 sched_setparam

```c
#include <sched.h>
int sched_setparam(pid_t pid, const struct sched_param *param);
```

**描述**：设置指定任务的优先级。注意：将优先级设为相同值有类似 `sched_yield()` 的效果——任务被移到同优先级队列末尾。

| 参数 | 类型 | 说明 |
|------|------|------|
| `pid` | `pid_t` | 任务ID，0表示调用任务自身 |
| `param` | `const struct sched_param *` | 包含 `sched_priority` 成员，有效范围 `SCHED_PRIORITY_MIN` ~ `SCHED_PRIORITY_MAX` |

**返回值**：`0 (OK)` 成功；`-1 (ERROR)` 失败，errno 设置为 `EINVAL`（参数无效）或 `EPERM`（权限不足）或 `ESRCH`（任务未找到）

**POSIX兼容性**：与同名POSIX接口可比。差异：优先级范围 0~255

---

### 3.2 sched_getparam

```c
#include <sched.h>
int sched_getparam(pid_t pid, struct sched_param *param);
```

**描述**：获取指定任务的调度优先级。

| 参数 | 类型 | 说明 |
|------|------|------|
| `pid` | `pid_t` | 任务ID，0表示调用任务 |
| `param` | `struct sched_param *` | 接收优先级的结构体 |

**返回值**：`0 (OK)` 成功；`-1 (ERROR)` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

## 6. Counting Semaphore 接口（计数信号量）

信号量是 NuttX 中同步和互斥的基础。优先使用信号量而非 `sched_lock()/sched_unlock()`，因为后者会阻止无关的高优先级任务运行。

**优先级反转**：低优先级任务持有信号量时被高优先级任务阻塞，中等优先级任务抢占低优先级任务，导致高优先级任务间接被低优先级任务阻塞。NuttX 通过 `CONFIG_PRIORITY_INHERITANCE` 支持优先级继承来缓解此问题。

**优先级继承配置项**：
- `CONFIG_SEM_PREALLOCHOLDERS`：预分配持有者结构池大小（信号量最大不同持有线程数-1）。仅用作互斥锁时可设为0
- `CONFIG_SEM_NNESTPRIO`：每个活动线程的嵌套优先级数组大小（等待某信号量的最大更高优先级线程数-1）。仅一个线程等待时可设为0

⚠️ **风险**：优先级继承实现复杂，边界情况（线程持锁退出、未调用 `sem_destroy()`、提升优先级后自调整）可能导致内存滞留

### 6.1 sem_init

```c
#include <semaphore.h>
int sem_init(sem_t *sem, int pshared, unsigned int value);
```

**描述**：初始化未命名信号量。成功后可用于 `sem_wait()`、`sem_post()`、`sem_trywait()`。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sem` | `sem_t *` | 要初始化的信号量 |
| `pshared` | `int` | 进程共享（**未使用**） |
| `value` | `unsigned int` | 信号量初始值 |

**返回值**：`0 (OK)` 或 `-1 (ERROR)`

**POSIX兼容性**：与同名POSIX接口可比。差异：`pshared` 未使用

---

### 6.2 sem_destroy

```c
#include <semaphore.h>
int sem_destroy(sem_t *sem);
```

**描述**：销毁未命名信号量。只能销毁由 `sem_init()` 创建的信号量。销毁其他任务正在阻塞等待的信号量行为未定义。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sem` | `sem_t *` | 要销毁的信号量 |

**返回值**：`0 (OK)` 或 `-1 (ERROR)`

**POSIX兼容性**：与同名POSIX接口可比

---

### 6.3 sem_open

```c
#include <semaphore.h>
sem_t *sem_open(const char *name, int oflag, ...);
```

**描述**：建立命名信号量与任务的连接。`O_CREAT` 时需提供 `mode`（未使用）和 `value`（初始值，不超过 `SEM_VALUE_MAX`）。

| 参数 | 类型 | 说明 |
|------|------|------|
| `name` | `const char *` | 信号量名称 |
| `oflag` | `int` | `0`（仅连接已存在的）、`O_CREAT`（不存在则创建）、`O_CREAT\|O_EXCL`（创建新的） |
| `...` | 可变 | `O_CREAT` 时：`mode`（mode_t，未使用）、`value`（unsigned int，初始值） |

**返回值**：`sem_t *` 指针（成功）或 `(sem_t *)(-1)` （失败）

**POSIX兼容性**：与同名POSIX接口可比。差异：链接/连接处理高度简化，只是计数信号量

---

### 6.4 sem_close

```c
#include <semaphore.h>
int sem_close(sem_t *sem);
```

**描述**：指示调用任务已完成使用指定命名信号量。释放为此命名信号量分配的资源。未 `sem_unlink()` 时不影响信号量本身。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sem` | `sem_t *` | 信号量描述符 |

**返回值**：`0 (OK)` 或 `-1 (ERROR)`

⚠️ 注意：不要删除另一个任务已锁定的信号量。不能用于未命名信号量。

**POSIX兼容性**：与同名POSIX接口可比

---

### 6.5 sem_unlink

```c
#include <semaphore.h>
int sem_unlink(const char *name);
```

**描述**：移除指定名称的信号量。若有任务仍打开，销毁延迟到所有引用关闭后。

| 参数 | 类型 | 说明 |
|------|------|------|
| `name` | `const char *` | 信号量名称 |

**返回值**：`0 (OK)` 或 `-1 (ERROR)`

**POSIX兼容性**：与同名POSIX接口可比。差异：链接/连接处理高度简化；`sem_unlink()` 后 `sem_open()` 可能引用同一信号量（POSIX要求创建新信号量）

---

### 6.6 sem_wait

```c
#include <semaphore.h>
int sem_wait(sem_t *sem);
```

**描述**：尝试锁定信号量。已被其他任务锁定时，调用任务阻塞直到获取锁或被信号中断。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sem` | `sem_t *` | 信号量描述符 |

**返回值**：`0 (OK)` 或 `-1 (ERROR)`，errno 设置为 `EINVAL`（无效信号量）或 `EINTR`（被信号中断，信号量未获取）

**POSIX兼容性**：与同名POSIX接口可比

---

### 6.7 sem_trywait

```c
#include <semaphore.h>
int sem_trywait(sem_t *sem);
```

**描述**：仅在信号量当前未锁定时锁定。任何情况下都不阻塞。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sem` | `sem_t *` | 信号量描述符 |

**返回值**：`0 (OK)` 或 `-1 (ERROR)`，errno 设置为 `EINVAL`（无效信号量）或 `EAGAIN`（未获取）

**POSIX兼容性**：与同名POSIX接口可比

---

### 6.8 sem_post

```c
#include <semaphore.h>
int sem_post(sem_t *sem);
```

**描述**：解锁信号量。若结果信号量值为正，无任务阻塞，值递增。若为零，一个阻塞任务被解除阻塞。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sem` | `sem_t *` | 信号量描述符 |

**返回值**：`0 (OK)` 或 `-1 (ERROR)`

⚠️ **文档矛盾**：描述中说"可从中断处理程序调用"，但 Assumptions 中说"不能从中断处理程序调用，假定当前执行任务就是执行解锁的任务"。实际使用时需核对源码

**POSIX兼容性**：与同名POSIX接口可比

---

### 6.9 sem_getvalue

```c
#include <semaphore.h>
int sem_getvalue(sem_t *sem, int *sval);
```

**描述**：获取信号量当前值，不影响信号量状态。信号量锁定时返回零或负数，负数的绝对值表示等待任务数。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sem` | `sem_t *` | 信号量描述符 |
| `sval` | `int *` | 值返回缓冲区 |

**返回值**：`0 (OK)` 或 `-1 (ERROR)`

**POSIX兼容性**：与同名POSIX接口可比

---

## 7. Watchdog Timer 接口（看门狗定时器）

NuttX 提供通用看门狗定时器设施。看门狗回调函数在**定时器中断处理程序上下文**中执行，因此受ISR限制。但可使用 `mq_send()`、`sigqueue()` 或 `kill()` 与任务通信。

### 7.1 wd_create

```c
#include <wdog.h>
WDOG_ID wd_create(void);
```

**描述**：分配资源创建一个看门狗。

**参数**：无

**返回值**：看门狗ID指针（成功）或 `NULL`（资源不足）

**POSIX兼容性**：**非POSIX接口**。类似 VxWorks `wdCreate()`。差异：可用看门狗数量固定（初始化时配置）

---

### 7.2 wd_delete

```c
#include <wdog.h>
int wd_delete(WDOG_ID wdog);
```

**描述**：释放看门狗资源。如果已启动，从定时器队列移除。

| 参数 | 类型 | 说明 |
|------|------|------|
| `wdog` | `WDOG_ID` | 要删除的看门狗ID |

**返回值**：`OK` 或 `ERROR`

⚠️ **风险**：不检查看门狗是否正在使用就释放（即从不返回ERROR）。调用者需自行确保看门狗不活跃

**POSIX兼容性**：**非POSIX接口**。类似 VxWorks `wdDelete()`

---

### 7.3 wd_start

```c
#include <wdog.h>
int wd_start(WDOG_ID wdog, int delay, wdentry_t wdentry,
             int argc, ...);
```

**描述**：将看门狗加入定时器队列。指定时钟节拍后从中断级调用回调函数。看门狗仅执行一次。再次调用 `wd_start()` 可替换超时延迟或回调函数。

| 参数 | 类型 | 说明 |
|------|------|------|
| `wdog` | `WDOG_ID` | 看门狗ID |
| `delay` | `int` | 延迟计数（时钟节拍） |
| `wdentry` | `wdentry_t` | 超时回调函数 |
| `argc` | `int` | 传给回调的 `uint32_t` 参数个数 |
| `...` | 可变 | `uint32_t` 类型参数 |

**返回值**：`OK` 或 `ERROR`

⚠️ **限制**：回调在定时器中断处理程序上下文执行，受所有ISR限制

**POSIX兼容性**：**非POSIX接口**。类似 VxWorks `wdStart()`。差异：支持多参数（VxWorks仅单参数）

---

### 7.4 wd_cancel

```c
#include <wdog.h>
int wd_cancel(WDOG_ID wdog);
```

**描述**：取消正在运行的看门狗定时器。可从中断级调用。

| 参数 | 类型 | 说明 |
|------|------|------|
| `wdog` | `WDOG_ID` | 要取消的看门狗ID |

**返回值**：`OK` 或 `ERROR`

**POSIX兼容性**：**非POSIX接口**。类似 VxWorks `wdCancel()`

---

### 7.5 wd_gettime

```c
#include <wdog.h>
int wd_gettime(WDOG_ID wdog);
```

**描述**：返回指定看门狗到期前的剩余时间。

| 参数 | 类型 | 说明 |
|------|------|------|
| `wdog` | `WDOG_ID` | 看门狗ID |

**返回值**：系统节拍数（剩余到期时间）。零表示wdog无效或已过期

**POSIX兼容性**：**非POSIX接口**

---

## 8. Clocks and Timers 接口（时钟与定时器）

### 8.1 clock_settime

```c
#include <time.h>
int clock_settime(clockid_t clockid, const struct timespec *tp);
```

**描述**：设置指定时钟的时间。

| 参数 | 类型 | 说明 |
|------|------|------|
| `clockid` | `clockid_t` | 时钟ID |
| `tp` | `const struct timespec *` | 要设置的时间 |

**返回值**：`0 (OK)` 成功；非零错误号表示失败

---

### 8.2 clock_gettime

```c
#include <time.h>
int clock_gettime(clockid_t clockid, struct timespec *tp);
```

**描述**：获取指定时钟的当前时间。

| 参数 | 类型 | 说明 |
|------|------|------|
| `clockid` | `clockid_t` | 时钟ID |
| `tp` | `struct timespec *` | 接收时间的结构体 |

**返回值**：`0 (OK)` 成功；非零错误号表示失败

---

### 8.3 clock_getres

```c
#include <time.h>
int clock_getres(clockid_t clockid, struct timespec *res);
```

**描述**：获取指定时钟的分辨率。

| 参数 | 类型 | 说明 |
|------|------|------|
| `clockid` | `clockid_t` | 时钟ID |
| `res` | `struct timespec *` | 接收分辨率的结构体 |

**返回值**：`0 (OK)` 成功；非零错误号表示失败

---

### 8.4 mktime

```c
#include <time.h>
time_t mktime(struct tm *tp);
```

**描述**：将 `struct tm` 分解时间转换为 `time_t` 日历时间。

| 参数 | 类型 | 说明 |
|------|------|------|
| `tp` | `struct tm *` | 分解时间结构体 |

**返回值**：成功返回日历时间值

---

### 8.5 gmtime

```c
#include <time.h>
struct tm *gmtime(const time_t *clock);
```

**描述**：将日历时间转换为UTC分解时间。返回静态分配的 `struct tm`。

| 参数 | 类型 | 说明 |
|------|------|------|
| `clock` | `const time_t *` | 日历时间（自1970-01-01 00:00:00 UTC起的秒数） |

**返回值**：成功返回指向静态 `struct tm` 的指针；失败返回 `NULL`

⚠️ **非线程安全**：返回静态缓冲区，多线程环境下应使用 `gmtime_r()`

---

### 8.6 localtime

```c
#include <time.h>
#define localtime(c) gmtime(c)
```

**描述**：在 NuttX 中，`localtime` 是 `gmtime` 的宏定义（无时区支持）。

---

### 8.7 gmtime_r

```c
#include <time.h>
struct tm *gmtime_r(const time_t *clock, struct tm *result);
```

**描述**：`gmtime()` 的可重入版本，使用调用者提供的缓冲区。

| 参数 | 类型 | 说明 |
|------|------|------|
| `clock` | `const time_t *` | 日历时间 |
| `result` | `struct tm *` | 调用者提供的缓冲区 |

**返回值**：成功返回 `result` 指针；失败返回 `NULL`

---

### 8.8 localtime_r

```c
#include <time.h>
#define localtime_r(c,r) gmtime_r(c,r)
```

**描述**：在 NuttX 中，`localtime_r` 是 `gmtime_r` 的宏定义。

---

### 8.9 timer_create

```c
#include <time.h>
int timer_create(clockid_t clockid, struct sigevent *evp, timer_t *timerid);
```

**描述**：创建每线程定时器，使用指定时钟作为时基。返回时定时器处于解除武装状态。`evp` 为NULL时使用默认信号通知。

| 参数 | 类型 | 说明 |
|------|------|------|
| `clockid` | `clockid_t` | 时钟ID，**必须为 `CLOCK_REALTIME`** |
| `evp` | `struct sigevent *` | 异步通知配置，可为NULL |
| `timerid` | `timer_t *` | 接收创建的定时器ID |

**返回值**：`0 (OK)` 成功；`-1 (ERROR)` 失败，errno 设置为 `EAGAIN`/`EINVAL`/`ENOTSUP`

**POSIX兼容性**：与同名POSIX接口可比。差异：仅支持 `CLOCK_REALTIME`

---

### 8.10 timer_delete

```c
#include <time.h>
int timer_delete(timer_t timerid);
```

**描述**：删除先前创建的定时器。如定时器已武装，自动解除武装。待处理信号的处理方式未指定。

| 参数 | 类型 | 说明 |
|------|------|------|
| `timerid` | `timer_t` | 要删除的定时器ID |

**返回值**：`0 (OK)` 成功；`-1 (ERROR)` 失败，errno 设置为 `EINVAL`

**POSIX兼容性**：与同名POSIX接口可比

---

### 8.11 timer_settime

```c
#include <time.h>
int timer_settime(timer_t timerid, int flags, const struct itimerspec *value,
                  struct itimerspec *ovalue);
```

**描述**：设置定时器的下次到期时间并武装。`it_value` 为零则解除武装。`TIMER_ABSTIME` 标志设为绝对时间。`it_interval` 非零则周期性触发。

| 参数 | 类型 | 说明 |
|------|------|------|
| `timerid` | `timer_t` | 定时器ID |
| `flags` | `int` | `TIMER_ABSTIME` 标志 |
| `value` | `const struct itimerspec *` | 定时器值 |
| `ovalue` | `struct itimerspec *` | 旧定时器值（**当前被忽略**） |

**返回值**：`0 (OK)` 成功；`-1 (ERROR)` 失败，errno 设置为 `EINVAL`

**POSIX兼容性**：与同名POSIX接口可比。差异：`ovalue` 参数被忽略

---

### 8.12 timer_gettime

```c
#include <time.h>
int timer_gettime(timer_t timerid, struct itimerspec *value);
```

**描述**：获取定时器的剩余时间和重载值。

| 参数 | 类型 | 说明 |
|------|------|------|
| `timerid` | `timer_t` | 定时器ID |
| `value` | `struct itimerspec *` | 接收定时器值 |

**返回值**：`0 (OK)` 成功；非零错误号表示失败（`EINVAL`）

**POSIX兼容性**：与同名POSIX接口可比

---

### 8.13 timer_getoverrun

```c
#include <time.h>
int timer_getoverrun(timer_t timerid);
```

**描述**：返回定时器超时计数。当定时器的信号仍待处理时再次到期，产生超限。

| 参数 | 类型 | 说明 |
|------|------|------|
| `timerid` | `timer_t` | 定时器ID |

**返回值**：成功返回超限计数；失败返回 `-1`，errno 设置为 `EINVAL`

⚠️ **重要**：此接口在 NuttX 中**当前未实现**

**POSIX兼容性**：与同名POSIX接口可比。差异：当前未实现

---

### 8.14 gettimeofday

```c
#include <sys/time.h>
int gettimeofday(struct timeval *tp, void *tzp);
```

**描述**：`clock_gettime(CLOCK_REALTIME)` 的薄包装，转换为 `struct timeval`。

| 参数 | 类型 | 说明 |
|------|------|------|
| `tp` | `struct timeval *` | 接收当前时间 |
| `tzp` | `void *` | 时区——**被忽略** |

**返回值**：同 `clock_gettime()`

---

## 9. Signal 接口（信号）

信号用于在任务上下文内或之间传递异步事件来改变控制流。任何任务或中断处理程序可向特定任务发送信号。

⚠️ **重要行为差异**：所有信号**默认行为是忽略**（非POSIX的终止行为），所有NuttX信号都是实时信号

### 9.1 sigemptyset

```c
#include <signal.h>
int sigemptyset(sigset_t *set);
```

**描述**：初始化信号集为空（排除所有信号）。

| 参数 | 类型 | 说明 |
|------|------|------|
| `set` | `sigset_t *` | 要初始化的信号集 |

**返回值**：`0 (OK)` 或 `-1 (ERROR)`

**POSIX兼容性**：与同名POSIX接口可比

---

### 9.2 sigfillset

```c
#include <signal.h>
int sigfillset(sigset_t *set);
```

**描述**：初始化信号集为满（包含所有信号）。

| 参数 | 类型 | 说明 |
|------|------|------|
| `set` | `sigset_t *` | 要初始化的信号集 |

**返回值**：`0 (OK)` 或 `-1 (ERROR)`

**POSIX兼容性**：与同名POSIX接口可比

---

### 9.3 sigaddset

```c
#include <signal.h>
int sigaddset(sigset_t *set, int signo);
```

**描述**：向信号集添加指定信号。

| 参数 | 类型 | 说明 |
|------|------|------|
| `set` | `sigset_t *` | 信号集 |
| `signo` | `int` | 要添加的信号 |

**返回值**：`0 (OK)` 或 `-1 (ERROR)`（信号编号无效）

**POSIX兼容性**：与同名POSIX接口可比

---

### 9.4 sigdelset

```c
#include <signal.h>
int sigdelset(sigset_t *set, int signo);
```

**描述**：从信号集删除指定信号。

| 参数 | 类型 | 说明 |
|------|------|------|
| `set` | `sigset_t *` | 信号集 |
| `signo` | `int` | 要删除的信号 |

**返回值**：`0 (OK)` 或 `-1 (ERROR)`（信号编号无效）

**POSIX兼容性**：与同名POSIX接口可比

---

### 9.5 sigismember

```c
#include <signal.h>
int sigismember(const sigset_t *set, int signo);
```

**描述**：测试指定信号是否在信号集中。

| 参数 | 类型 | 说明 |
|------|------|------|
| `set` | `const sigset_t *` | 信号集 |
| `signo` | `int` | 要测试的信号 |

**返回值**：`1 (TRUE)` 在集合中；`0 (FALSE)` 不在；`-1 (ERROR)` 信号编号无效

**POSIX兼容性**：与同名POSIX接口可比

---

### 9.6 sigaction

```c
#include <signal.h>
int sigaction(int signo, const struct sigaction *act, struct sigaction *oact);
```

**描述**：检查和/或设置与特定信号关联的动作。`act` 非NULL则设置新动作，`oact` 非NULL则保存旧动作。信号处理函数返回后恢复原始信号掩码。安装后持续有效直到再次调用 `sigaction()`。

| 参数 | 类型 | 说明 |
|------|------|------|
| `signo` | `int` | 信号编号 |
| `act` | `const struct sigaction *` | 新处理动作 |
| `oact` | `struct sigaction *` | 旧处理动作存储 |

**返回值**：`0 (OK)` 或 `-1 (ERROR)`（信号编号无效）

**POSIX兼容性**：与同名POSIX接口可比。差异：不处理 `SIG_DFL`/`SIG_IGN` 特殊值；所有 `sa_flags` 被忽略（都按 `SA_SIGINFO` 处理）

---

### 9.7 sigprocmask

```c
#include <signal.h>
int sigprocmask(int how, const sigset_t *set, sigset_t *oset);
```

**描述**：检查和/或修改调用任务的信号掩码。`how` 指定修改方式。调用后待处理的已解除阻塞信号会被递送。

| 参数 | 类型 | 说明 |
|------|------|------|
| `how` | `int` | `SIG_BLOCK`（并集）/`SIG_UNBLOCK`（交集补集）/`SIG_SETMASK`（直接设置） |
| `set` | `const sigset_t *` | 新信号掩码 |
| `oset` | `sigset_t *` | 旧信号掩码存储 |

**返回值**：`0 (OK)` 或 `-1 (ERROR)`（how 无效）

**POSIX兼容性**：与同名POSIX接口可比

---

### 9.8 sigpending

```c
#include <signal.h>
int sigpending(sigset_t *set);
```

**描述**：返回被阻塞且待处理的信号集。每个信号编号只保留一个待处理信号。

| 参数 | 类型 | 说明 |
|------|------|------|
| `set` | `sigset_t *` | 接收待处理信号集 |

**返回值**：`0 (OK)` 或 `-1 (ERROR)`

**POSIX兼容性**：与同名POSIX接口可比

---

### 9.9 sigsuspend

```c
#include <signal.h>
int sigsuspend(const sigset_t *set);
```

**描述**：用指定信号集替换信号掩码后挂起任务直到收到信号。若set解除阻塞了待处理信号则立即返回。返回后恢复原始掩码。

| 参数 | 类型 | 说明 |
|------|------|------|
| `set` | `const sigset_t *` | 挂起期间使用的信号掩码 |

**返回值**：始终返回 `-1 (ERROR)`

⚠️ 等待空信号集会停止任务而不释放资源（极不推荐）

**POSIX兼容性**：与同名POSIX接口可比。差异：POSIX未说明恢复原始掩码（NuttX会恢复）；即使信号被忽略也会恢复任务

---

### 9.10 sigwaitinfo

```c
#include <signal.h>
int sigwaitinfo(const sigset_t *set, struct siginfo *info);
```

**描述**：等价于 `sigtimedwait()` 的NULL超时参数版本。

| 参数 | 类型 | 说明 |
|------|------|------|
| `set` | `const sigset_t *` | 等待的信号集 |
| `info` | `struct siginfo *` | 返回的信号值 |

**返回值**：导致等待终止的信号编号；`-1 (ERROR)` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

### 9.11 sigtimedwait

```c
#include <signal.h>
int sigtimedwait(const sigset_t *set, struct siginfo *info,
                 const struct timespec *timeout);
```

**描述**：选择待处理信号集中编号最低的信号。无待处理信号时挂起直到信号到达或超时。`info` 非NULL时存储信号信息（`si_signo`、`si_code`、`si_value`）。

`si_code` 取值：`SI_USER`（kill/raise/abort）、`SI_QUEUE`（sigqueue）、`SI_TIMER`（定时器到期）、`SI_ASYNCIO`（异步IO完成）、`SI_MESGQ`（消息到达空队列）

| 参数 | 类型 | 说明 |
|------|------|------|
| `set` | `const sigset_t *` | 等待的信号集 |
| `info` | `struct siginfo *` | 返回的信号值 |
| `timeout` | `const struct timespec *` | 等待时间，NULL表示无限 |

**返回值**：导致等待终止的信号编号；`-1 (ERROR)` 失败

**POSIX兼容性**：与同名POSIX接口可比。差异：`si_code` 值不同；无返回ERROR原因的机制；不要求解除阻塞的信号被捕获

---

### 9.12 sigqueue

```c
#include <signal.h>
int sigqueue(int tid, int signo, union sigval value);
```

**描述**：向指定任务发送信号并附带参数值。每个信号编号只保留一个待处理信号。

| 参数 | 类型 | 说明 |
|------|------|------|
| `tid` | `int` | 接收信号的任务ID |
| `signo` | `int` | 信号编号 |
| `value` | `union sigval` | 传递给任务的值 |

**返回值**：`0 (OK)` 成功；`-1 (ERROR)` 失败，errno 设置为 `EAGAIN`/`EINVAL`/`EPERM`/`ESRCH`

**POSIX兼容性**：与同名POSIX接口可比。差异：默认动作是忽略；信号一次处理一个；无空信号（signo=0会发送）

---

### 9.13 kill

```c
#include <sys/types.h>
#include <signal.h>
int kill(pid_t pid, int sig);
```

**描述**：向任意任务发送任意信号。只支持正非零pid值（不支持进程组）。

| 参数 | 类型 | 说明 |
|------|------|------|
| `pid` | `pid_t` | 接收信号的任务ID（仅正非零值） |
| `sig` | `int` | 信号编号，为零时不发送但执行错误检查 |

**返回值**：`OK` 或 `ERROR`

**POSIX兼容性**：与同名POSIX接口可比。差异：默认动作忽略；信号一次处理一个；不支持进程组

---

## 10. Pthread 接口（POSIX线程）

NuttX 不支持进程模型，只支持同一地址空间内的简单线程。线程和任务主要区别在文件描述符继承等方面。线程初始化方式不同，共享更多资源。

NuttX 支持54个 pthread 接口，不支持约40个接口（包括读写锁、自旋锁、`pthread_setcanceltype` 等）。

### 线程属性接口

#### 10.1 pthread_attr_init

```c
#include <pthread.h>
int pthread_attr_init(pthread_attr_t *attr);
```

**描述**：用默认值初始化线程属性对象。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `pthread_attr_t *` | 要初始化的属性对象 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.2 pthread_attr_destroy

```c
#include <pthread.h>
int pthread_attr_destroy(pthread_attr_t *attr);
```

**描述**：销毁不再需要的属性对象。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `pthread_attr_t *` | 要销毁的属性对象 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.3 pthread_attr_setschedpolicy

```c
#include <pthread.h>
int pthread_attr_setschedpolicy(pthread_attr_t *attr, int policy);
```

**描述**：设置线程属性对象中的调度策略。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `pthread_attr_t *` | 属性对象 |
| `policy` | `int` | 调度策略 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.4 pthread_attr_getschedpolicy

```c
#include <pthread.h>
int pthread_attr_getschedpolicy(pthread_attr_t *attr, int *policy);
```

**描述**：获取线程属性对象中的调度策略。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `pthread_attr_t *` | 属性对象 |
| `policy` | `int *` | 接收调度策略 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.5 pthread_attr_setschedparam

```c
#include <pthread.h>
int pthread_attr_setschedparam(pthread_attr_t *attr,
                                 const struct sched_param *param);
```

**描述**：设置线程属性对象中的调度参数（优先级）。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `pthread_attr_t *` | 属性对象 |
| `param` | `const struct sched_param *` | 调度参数 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.6 pthread_attr_getschedparam

```c
#include <pthread.h>
int pthread_attr_getschedparam(pthread_attr_t *attr,
                                 struct sched_param *param);
```

**描述**：获取线程属性对象中的调度参数。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `pthread_attr_t *` | 属性对象 |
| `param` | `struct sched_param *` | 接收调度参数 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.7 pthread_attr_setinheritsched

```c
#include <pthread.h>
int pthread_attr_setinheritsched(pthread_attr_t *attr, int inheritsched);
```

**描述**：设置线程属性对象中的调度继承属性。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `pthread_attr_t *` | 属性对象 |
| `inheritsched` | `int` | 继承调度属性 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.8 pthread_attr_getinheritsched

```c
#include <pthread.h>
int pthread_attr_getinheritsched(const pthread_attr_t *attr,
                                   int *inheritsched);
```

**描述**：获取线程属性对象中的调度继承属性。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `const pthread_attr_t *` | 属性对象 |
| `inheritsched` | `int *` | 接收继承调度属性 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.9 pthread_attr_setstacksize

```c
#include <pthread.h>
int pthread_attr_setstacksize(pthread_attr_t *attr, long stacksize);
```

**描述**：设置线程属性对象中的栈大小。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `pthread_attr_t *` | 属性对象 |
| `stacksize` | `long` | 栈大小 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.10 pthread_attr_getstacksize

```c
#include <pthread.h>
int pthread_attr_getstacksize(pthread_attr_t *attr, long *stackaddr);
```

**描述**：获取线程属性对象中的栈大小。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `pthread_attr_t *` | 属性对象 |
| `stackaddr` | `long *` | 接收栈大小 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

### 线程管理接口

#### 10.11 pthread_create

```c
#include <pthread.h>
int pthread_create(pthread_t *thread, pthread_attr_t *attr,
                   pthread_startroutine_t startRoutine, pthread_addr_t arg);
```

**描述**：创建线程对象和可运行线程。指定起始例程和可选参数。

| 参数 | 类型 | 说明 |
|------|------|------|
| `thread` | `pthread_t *` | 接收线程ID |
| `attr` | `pthread_attr_t *` | 线程属性 |
| `startRoutine` | `pthread_startroutine_t` | 线程起始函数 |
| `arg` | `pthread_addr_t` | 传递给起始函数的参数 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.12 pthread_detach

```c
#include <pthread.h>
int pthread_detach(pthread_t thread);
```

**描述**：分离线程，指定不需要返回值和完成状态。

| 参数 | 类型 | 说明 |
|------|------|------|
| `thread` | `pthread_t` | 要分离的线程 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.13 pthread_exit

```c
#include <pthread.h>
void pthread_exit(pthread_addr_t pvValue);
```

**描述**：线程终止自身执行。

| 参数 | 类型 | 说明 |
|------|------|------|
| `pvValue` | `pthread_addr_t` | 返回值 |

**返回值**：无（void函数）

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.14 pthread_cancel

```c
#include <pthread.h>
int pthread_cancel(pthread_t thread);
```

**描述**：请求取消指定线程。目标线程的可取消状态决定取消何时生效。禁用取消时取消请求挂起；启用时立即执行。

| 参数 | 类型 | 说明 |
|------|------|------|
| `thread` | `pthread_t` | 要取消的线程 |

**返回值**：`0 (OK)` 成功；`ESRCH` 线程未找到

**POSIX兼容性**：与同名POSIX接口可比。差异：线程特定数据析构函数不支持；取消类型不支持；`pthread_testcancel()` 不支持；取消点不支持

---

#### 10.15 pthread_setcancelstate

```c
#include <pthread.h>
int pthread_setcancelstate(int state, int *oldstate);
```

**描述**：原子性地设置调用线程的可取消状态并返回旧状态。

| 参数 | 类型 | 说明 |
|------|------|------|
| `state` | `int` | `PTHREAD_CANCEL_ENABLE` 或 `PTHREAD_CANCEL_DISABLE` |
| `oldstate` | `int *` | 接收旧状态 |

**返回值**：`0 (OK)` 成功；`ESRCH` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.16 pthread_testcancelstate

```c
#include <pthread.h>
int pthread_setcancelstate(void);
```

**描述**：⚠️ **不支持（NOT SUPPORTED）**

**POSIX兼容性**：与同名POSIX接口可比（但未实现）

---

#### 10.17 pthread_join

```c
#include <pthread.h>
int pthread_join(pthread_t thread, pthread_addr_t *ppvValue);
```

**描述**：等待另一线程终止并获取返回值。

| 参数 | 类型 | 说明 |
|------|------|------|
| `thread` | `pthread_t` | 要等待的线程 |
| `ppvValue` | `pthread_addr_t *` | 接收线程返回值 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.18 pthread_yield

```c
#include <pthread.h>
void pthread_yield(void);
```

**描述**：告知调度器当前线程的处理器可被释放。

**参数**：无

**返回值**：无（void函数）

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.19 pthread_self

```c
#include <pthread.h>
pthread_t pthread_self(void);
```

**描述**：获取调用线程自身的线程句柄。

**参数**：无

**返回值**：调用线程的句柄

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.20 pthread_getschedparam

```c
#include <pthread.h>
int pthread_getschedparam(pthread_t thread, int *policy,
                           struct sched_param *param);
```

**描述**：获取线程的调度策略和参数。返回的优先级不受临时调整（如优先级继承）影响。

| 参数 | 类型 | 说明 |
|------|------|------|
| `thread` | `pthread_t` | 线程ID |
| `policy` | `int *` | 接收调度策略 |
| `param` | `struct sched_param *` | 接收调度参数 |

**返回值**：`0 (OK)` 成功；`ESRCH` 线程不存在

**POSIX兼容性**：与同名POSIX接口可比。仅支持 `SCHED_FIFO` 和 `SCHED_RR`

---

#### 10.21 pthread_setschedparam

```c
#include <pthread.h>
int pthread_setschedparam(pthread_t thread, int policy,
                           const struct sched_param *param);
```

**描述**：设置线程的调度策略和参数。

| 参数 | 类型 | 说明 |
|------|------|------|
| `thread` | `pthread_t` | 线程ID |
| `policy` | `int` | 新调度策略（`SCHED_FIFO` 或 `SCHED_RR`） |
| `param` | `const struct sched_param *` | 调度参数 |

**返回值**：`0 (OK)` 成功；errno 设置为 `EINVAL`/`ENOTSUP`/`EPERM`/`ESRCH`

**POSIX兼容性**：与同名POSIX接口可比

---

### 线程特定数据接口

#### 10.22 pthread_key_create

```c
#include <pthread.h>
int pthread_key_create(pthread_key_t *key, void (*destructor)(void*));
```

**描述**：创建对所有线程可见的线程特定数据键。创建时所有线程关联值为NULL。

| 参数 | 类型 | 说明 |
|------|------|------|
| `key` | `pthread_key_t *` | 接收创建的键 |
| `destructor` | `void (*)(void*)` | 可选析构函数（**当前实现被忽略**） |

**返回值**：`0 (OK)` 成功；`EAGAIN`/`ENOMEM` 失败

⚠️ **风险**：析构函数参数被忽略，线程退出时不会自动清理线程特定数据

**POSIX兼容性**：与同名POSIX接口可比。差异：析构函数被忽略

---

#### 10.23 pthread_setspecific

```c
#include <pthread.h>
int pthread_setspecific(pthread_key_t key, void *value);
```

**描述**：将线程特定值与键关联。

| 参数 | 类型 | 说明 |
|------|------|------|
| `key` | `pthread_key_t` | 数据键 |
| `value` | `void *` | 绑定的值 |

**返回值**：`0 (OK)` 成功；`ENOMEM`/`EINVAL` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.24 pthread_getspecific

```c
#include <pthread.h>
void *pthread_getspecific(pthread_key_t key);
```

**描述**：返回调用线程与指定键关联的值。无关联值返回NULL。

| 参数 | 类型 | 说明 |
|------|------|------|
| `key` | `pthread_key_t` | 数据键 |

**返回值**：线程特定数据指针；NULL表示无关联数据

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.25 pthread_key_delete

```c
#include <pthread.h>
int pthread_key_delete(pthread_key_t key);
```

**描述**：删除线程特定数据键。

| 参数 | 类型 | 说明 |
|------|------|------|
| `key` | `pthread_key_t` | 要删除的键 |

**返回值**：始终返回 `EINVAL`

⚠️ **重要**：此函数在当前实现中**实际什么都不做**，始终返回EINVAL

**POSIX兼容性**：与同名POSIX接口可比

---

### 互斥锁属性接口

#### 10.26 pthread_mutexattr_init

```c
#include <pthread.h>
int pthread_mutexattr_init(pthread_mutexattr_t *attr);
```

**描述**：初始化互斥锁属性对象。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `pthread_mutexattr_t *` | 属性对象 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.27 pthread_mutexattr_destroy

```c
#include <pthread.h>
int pthread_mutexattr_destroy(pthread_mutexattr_t *attr);
```

**描述**：销毁互斥锁属性对象。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `pthread_mutexattr_t *` | 属性对象 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.28 pthread_mutexattr_getpshared

```c
#include <pthread.h>
int pthread_mutexattr_getpshared(pthread_mutexattr_t *attr, int *pshared);
```

**描述**：获取互斥锁属性中的进程共享属性。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `pthread_mutexattr_t *` | 属性对象 |
| `pshared` | `int *` | 接收pshared值 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.29 pthread_mutexattr_setpshared

```c
#include <pthread.h>
int pthread_mutexattr_setpshared(pthread_mutexattr_t *attr, int pshared);
```

**描述**：设置互斥锁属性中的进程共享属性。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `pthread_mutexattr_t *` | 属性对象 |
| `pshared` | `int` | pshared值 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.30 pthread_mutexattr_gettype

```c
#include <pthread.h>
#ifdef CONFIG_MUTEX_TYPES
int pthread_mutexattr_gettype(const pthread_mutexattr_t *attr, int *type);
#endif
```

**描述**：获取互斥锁类型。需启用 `CONFIG_MUTEX_TYPES`。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `const pthread_mutexattr_t *` | 属性对象 |
| `type` | `int *` | 接收互斥锁类型 |

**返回值**：`0 (OK)` 成功；`EINVAL` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.31 pthread_mutexattr_settype

```c
#include <pthread.h>
#ifdef CONFIG_MUTEX_TYPES
int pthread_mutexattr_settype(pthread_mutexattr_t *attr, int type);
#endif
```

**描述**：设置互斥锁类型。需启用 `CONFIG_MUTEX_TYPES`。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `pthread_mutexattr_t *` | 属性对象 |
| `type` | `int` | 互斥锁类型 |

**互斥锁类型**：
- `PTHREAD_MUTEX_NORMAL`：不检测死锁，重锁会死锁
- `PTHREAD_MUTEX_ERRORCHECK`：提供错误检查，重锁/解锁他线程锁/解锁未锁返回错误
- `PTHREAD_MUTEX_RECURSIVE`：重锁成功，计数递增/递减
- `PTHREAD_MUTEX_DEFAULT`：默认类型（NORMAL）

⚠️ **重要**：NuttX 中 `PTHREAD_MUTEX_NORMAL` **未实现**，实际行为为 `PTHREAD_MUTEX_ERRORCHECK`

**返回值**：`0 (OK)` 成功；`EINVAL` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

### 互斥锁操作接口

#### 10.32 pthread_mutex_init

```c
#include <pthread.h>
int pthread_mutex_init(pthread_mutex_t *mutex, pthread_mutexattr_t *attr);
```

**描述**：初始化互斥锁。

| 参数 | 类型 | 说明 |
|------|------|------|
| `mutex` | `pthread_mutex_t *` | 互斥锁 |
| `attr` | `pthread_mutexattr_t *` | 属性对象 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.33 pthread_mutex_destroy

```c
#include <pthread.h>
int pthread_mutex_destroy(pthread_mutex_t *mutex);
```

**描述**：销毁互斥锁。

| 参数 | 类型 | 说明 |
|------|------|------|
| `mutex` | `pthread_mutex_t *` | 互斥锁 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.34 pthread_mutex_lock

```c
#include <pthread.h>
int pthread_mutex_lock(pthread_mutex_t *mutex);
```

**描述**：锁定互斥锁。已锁定时阻塞直到可用。NuttX中NORMAL类型行为实际为ERRORCHECK。信号传递不返回EINTR，信号处理函数返回后继续等待。

| 参数 | 类型 | 说明 |
|------|------|------|
| `mutex` | `pthread_mutex_t *` | 互斥锁 |

**返回值**：`0 (OK)` 成功；错误号表示失败（不返回 `EINTR`）

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.35 pthread_mutex_trylock

```c
#include <pthread.h>
int pthread_mutex_trylock(pthread_mutex_t *mutex);
```

**描述**：尝试锁定互斥锁，已锁定时立即返回 `EBUSY`。

| 参数 | 类型 | 说明 |
|------|------|------|
| `mutex` | `pthread_mutex_t *` | 互斥锁 |

**返回值**：`0 (OK)` 成功；错误号表示失败（不返回 `EINTR`）

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.36 pthread_mutex_unlock

```c
#include <pthread.h>
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

**描述**：释放互斥锁。释放方式取决于互斥锁类型属性。

| 参数 | 类型 | 说明 |
|------|------|------|
| `mutex` | `pthread_mutex_t *` | 互斥锁 |

**返回值**：`0 (OK)` 成功；错误号表示失败（不返回 `EINTR`）

**POSIX兼容性**：与同名POSIX接口可比

---

### 条件变量接口

#### 10.37 pthread_condattr_init

```c
#include <pthread.h>
int pthread_condattr_init(pthread_condattr_t *attr);
```

**描述**：初始化条件变量属性对象。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `pthread_condattr_t *` | 属性对象 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.38 pthread_condattr_destroy

```c
#include <pthread.h>
int pthread_condattr_destroy(pthread_condattr_t *attr);
```

**描述**：销毁条件变量属性对象。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `pthread_condattr_t *` | 属性对象 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.39 pthread_cond_init

```c
#include <pthread.h>
int pthread_cond_init(pthread_cond_t *cond, pthread_condattr_t *attr);
```

**描述**：初始化条件变量。

| 参数 | 类型 | 说明 |
|------|------|------|
| `cond` | `pthread_cond_t *` | 条件变量 |
| `attr` | `pthread_condattr_t *` | 属性对象 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.40 pthread_cond_destroy

```c
#include <pthread.h>
int pthread_cond_destroy(pthread_cond_t *cond);
```

**描述**：销毁条件变量。

| 参数 | 类型 | 说明 |
|------|------|------|
| `cond` | `pthread_cond_t *` | 条件变量 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.41 pthread_cond_broadcast

```c
#include <pthread.h>
int pthread_cond_broadcast(pthread_cond_t *cond);
```

**描述**：唤醒所有等待指定条件变量的线程。

| 参数 | 类型 | 说明 |
|------|------|------|
| `cond` | `pthread_cond_t *` | 条件变量 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.42 pthread_cond_signal

```c
#include <pthread.h>
int pthread_cond_signal(pthread_cond_t *cond);
```

**描述**：唤醒至少一个等待指定条件变量的线程。

| 参数 | 类型 | 说明 |
|------|------|------|
| `cond` | `pthread_cond_t *` | 条件变量 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.43 pthread_cond_wait

```c
#include <pthread.h>
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
```

**描述**：等待条件变量。自动释放互斥锁并阻塞，被唤醒时重新获取互斥锁。

| 参数 | 类型 | 说明 |
|------|------|------|
| `cond` | `pthread_cond_t *` | 条件变量 |
| `mutex` | `pthread_mutex_t *` | 互斥锁 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.44 pthread_cond_timedwait

```c
#include <pthread.h>
int pthread_cond_timedwait(pthread_cond_t *cond, pthread_mutex_t *mutex,
                            const struct timespec *abstime);
```

**描述**：带绝对超时的条件变量等待。

| 参数 | 类型 | 说明 |
|------|------|------|
| `cond` | `pthread_cond_t *` | 条件变量 |
| `mutex` | `pthread_mutex_t *` | 互斥锁 |
| `abstime` | `const struct timespec *` | 绝对超时时间 |

**返回值**：`0 (OK)` 成功；错误号表示失败

**POSIX兼容性**：与同名POSIX接口可比

---

### 屏障接口

#### 10.45 pthread_barrierattr_init

```c
#include <pthread.h>
int pthread_barrierattr_init(FAR pthread_barrierattr_t *attr);
```

**描述**：初始化屏障属性对象。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `FAR pthread_barrierattr_t *` | 属性对象 |

**返回值**：`0 (OK)` 成功；`EINVAL` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.46 pthread_barrierattr_destroy

```c
#include <pthread.h>
int pthread_barrierattr_destroy(FAR pthread_barrierattr_t *attr);
```

**描述**：销毁屏障属性对象。销毁后可重新初始化。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `FAR pthread_barrierattr_t *` | 属性对象 |

**返回值**：`0 (OK)` 成功；`EINVAL` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.47 pthread_barrierattr_setpshared

```c
#include <pthread.h>
int pthread_barrierattr_setpshared(FAR pthread_barrierattr_t *attr, int pshared);
```

**描述**：设置屏障的进程共享属性。默认 `PTHREAD_PROCESS_PRIVATE`。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `FAR pthread_barrierattr_t *` | 属性对象 |
| `pshared` | `int` | `PTHREAD_PROCESS_SHARED` 或 `PTHREAD_PROCESS_PRIVATE` |

**返回值**：`0 (OK)` 成功；`EINVAL` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.48 pthread_barrierattr_getpshared

```c
#include <pthread.h>
int pthread_barrierattr_getpshared(FAR const pthread_barrierattr_t *attr,
                                     FAR int *pshared);
```

**描述**：获取屏障的进程共享属性。

| 参数 | 类型 | 说明 |
|------|------|------|
| `attr` | `FAR const pthread_barrierattr_t *` | 属性对象 |
| `pshared` | `FAR int *` | 接收pshared值 |

**返回值**：`0 (OK)` 成功；`EINVAL` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.49 pthread_barrier_init

```c
#include <pthread.h>
int pthread_barrier_init(FAR pthread_barrier_t *barrier,
                          FAR const pthread_barrierattr_t *attr,
                          unsigned int count);
```

**描述**：初始化屏障。`count` 指定必须调用 `pthread_barrier_wait()` 的线程数。

| 参数 | 类型 | 说明 |
|------|------|------|
| `barrier` | `FAR pthread_barrier_t *` | 屏障 |
| `attr` | `FAR const pthread_barrierattr_t *` | 属性对象（NULL用默认值） |
| `count` | `unsigned int` | 线程数（必须大于零） |

**返回值**：`0 (OK)` 成功；`EAGAIN`/`EINVAL`/`ENOMEM`/`EBUSY` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.50 pthread_barrier_destroy

```c
#include <pthread.h>
int pthread_barrier_destroy(FAR pthread_barrier_t *barrier);
```

**描述**：销毁屏障，释放资源。

| 参数 | 类型 | 说明 |
|------|------|------|
| `barrier` | `FAR pthread_barrier_t *` | 屏障 |

**返回值**：`0 (OK)` 成功；`EBUSY`/`EINVAL` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.51 pthread_barrier_wait

```c
#include <pthread.h>
int pthread_barrier_wait(FAR pthread_barrier_t *barrier);
```

**描述**：在屏障处同步参与线程。阻塞直到所需数量的线程调用此函数。一个线程返回 `PTHREAD_BARRIER_SERIAL_THREAD`，其余返回0。

| 参数 | 类型 | 说明 |
|------|------|------|
| `barrier` | `FAR pthread_barrier_t *` | 屏障 |

**返回值**：`0 (OK)` 或 `PTHREAD_BARRIER_SERIAL_THREAD` 成功；`EINVAL` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

### 其他Pthread接口

#### 10.52 pthread_once

```c
#include <pthread.h>
int pthread_once(FAR pthread_once_t *once_control,
                 CODE void (*init_routine)(void));
```

**描述**：首次调用时执行初始化例程，后续调用无效果。`once_control` 应声明为 `PTHREAD_ONCE_INIT`。

| 参数 | 类型 | 说明 |
|------|------|------|
| `once_control` | `FAR pthread_once_t *` | 控制变量 |
| `init_routine` | `CODE void (*)(void)` | 初始化函数 |

**返回值**：`0 (OK)` 成功；`EINVAL` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.53 pthread_kill

```c
#include <signal.h>
#include <pthread.h>
int pthread_kill(pthread_t thread, int signo);
```

**描述**：向指定线程发送信号。`kill()` 的简单包装。

| 参数 | 类型 | 说明 |
|------|------|------|
| `thread` | `pthread_t` | 线程ID（仅正非零值） |
| `signo` | `int` | 信号编号，零时不发送但做错误检查 |

**返回值**：`0 (OK)` 成功；`EINVAL`/`EPERM`/`ESRCH`/`ENOSYS` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

#### 10.54 pthread_sigmask

```c
#include <signal.h>
#include <pthread.h>
int pthread_sigmask(int how, FAR const sigset_t *set, FAR sigset_t *oset);
```

**描述**：`sigprocmask()` 的简单包装。

| 参数 | 类型 | 说明 |
|------|------|------|
| `how` | `int` | `SIG_BLOCK`/`SIG_UNBLOCK`/`SIG_SETMASK` |
| `set` | `FAR const sigset_t *` | 新信号掩码 |
| `oset` | `FAR sigset_t *` | 旧信号掩码 |

**返回值**：`0 (OK)` 成功；`EINVAL` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

### 不支持的Pthread接口

以下接口在 NuttX 中**不支持**：

| 接口 | 说明 |
|------|------|
| `pthread_atfork` | 注册fork处理程序 |
| `pthread_attr_getdetachstate/setdetachstate` | 分离状态属性 |
| `pthread_attr_getguardsize/setguardsize` | 线程保护区大小 |
| `pthread_attr_getscope/setscope` | 竞争范围属性 |
| `pthread_attr_getstack/setstack` | 栈属性 |
| `pthread_attr_getstackaddr/setstackaddr` | 栈地址属性 |
| `pthread_cleanup_pop/push` | 取消处理程序 |
| `pthread_condattr_getclock/setclock` | 条件变量时钟属性 |
| `pthread_condattr_getpshared/setpshared` | 条件变量进程共享属性 |
| `pthread_getconcurrency/setconcurrency` | 并发级别 |
| `pthread_getcpuclockid` | 线程CPU时钟 |
| `pthread_mutex_getprioceiling/setprioceiling` | 互斥锁优先级上限 |
| `pthread_mutex_timedlock` | 带超时的互斥锁 |
| `pthread_mutexattr_getprioceiling/setprioceiling` | 优先级上限属性 |
| `pthread_mutexattr_getprotocol/setprotocol` | 协议属性 |
| `pthread_rwlock_*` | 读写锁（全部不支持） |
| `pthread_setcanceltype` | 取消类型 |
| `pthread_spin_*` | 自旋锁（全部不支持） |
| `pthread_testcancel` | 在取消点测试 |

---

## 11. Environment Variables 接口（环境变量）

NuttX 支持环境变量。**Task** 创建时继承父任务环境的独立副本（后续互不影响）；**Pthread** 创建时共享父任务环境（修改互相可见）。

⚠️ 可通过 `CONFIG_DISABLE_ENVIRON` 禁用全部环境变量支持

### 11.1 getenv

```c
#include <stdlib.h>
FAR char *getenv(const char *name);
```

**描述**：在环境列表中搜索匹配名称的变量。

| 参数 | 类型 | 说明 |
|------|------|------|
| `name` | `const char *` | 要查找的变量名 |

**返回值**：变量值（只读）或 `NULL`（未找到）

---

### 11.2 putenv

```c
#include <stdlib.h>
int putenv(char *string);
```

**描述**：添加或修改环境变量。参数格式为 `name=value`。

| 参数 | 类型 | 说明 |
|------|------|------|
| `string` | `char *` | `name=value` 格式的环境设置字符串 |

**返回值**：成功返回零

---

### 11.3 clearenv

```c
#include <stdlib.h>
int clearenv(void);
```

**描述**：清除所有名称-值对，将 `environ` 设为NULL。

**参数**：无

**返回值**：成功返回零

---

### 11.4 setenv

```c
#include <stdlib.h>
int setenv(const char *name, const char *value, int overwrite);
```

**描述**：添加或修改环境变量。变量存在时根据 `overwrite` 决定是否覆盖。

| 参数 | 类型 | 说明 |
|------|------|------|
| `name` | `const char *` | 变量名 |
| `value` | `const char *` | 新值 |
| `overwrite` | `int` | 非零时覆盖已有值 |

**返回值**：成功返回零

---

### 11.5 unsetenv

```c
#include <stdlib.h>
int unsetenv(const char *name);
```

**描述**：从环境中删除指定变量。

| 参数 | 类型 | 说明 |
|------|------|------|
| `name` | `const char *` | 要删除的变量名 |

**返回值**：成功返回零

---

## 12. File System 接口（文件系统）

### 12.1 文件系统概述

NuttX 包含可选的、可扩展的文件系统。文件系统可完全省略（NuttX不依赖文件系统）。

**伪根文件系统**：内存中的伪文件系统，通过 `CONFIG_NFILE_DESCRIPTORS`（设为非零值）启用。内容按需生成，类似Linux /proc。

**挂载文件系统**：可在伪文件系统中挂载块设备，绑定到真实文件系统。当前仅支持VFAT文件系统。

**与Linux对比**：NuttX根是伪文件系统，真实文件系统挂载在伪文件系统中；Linux根是真实文件系统，伪文件系统挂载在真实根中。

### 12.2 Driver Operations（驱动操作）

#### open

```c
#include <fcntl.h>
int open(const char *path, int oflag, ...);
```

#### close

```c
#include <unistd.h>
int close(int fd);
```

#### dup

```c
#include <unistd.h>
int dup(int fildes);
```

#### dup2

```c
#include <unistd.h>
int dup2(int fildes1, int fildes2);
```

#### lseek

```c
#include <unistd.h>
off_t lseek(int fd, off_t offset, int whence);
```

#### read

```c
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t nbytes);
```

#### write

```c
#include <unistd.h>
ssize_t write(int fd, const void *buf, size_t nbytes);
```

#### unlink

```c
#include <unistd.h>
int unlink(const char *path);
```

#### ioctl

```c
#include <sys/ioctl.h>
int ioctl(int fd, int req, unsigned long arg);
```

#### poll

```c
#include <poll.h>
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

**描述**：等待一组文件描述符中的一个变为I/O就绪。

| 参数 | 类型 | 说明 |
|------|------|------|
| `fds` | `struct pollfd *` | 监控文件描述符列表 |
| `nfds` | `nfds_t` | 列表中条目数 |
| `timeout` | `int` | 超时（毫秒），负值表示无限等待 |

**返回值**：成功返回revents非零的结构数；`0` 超时；`-1` 错误，errno 设置为 `EBADF`/`EFAULT`/`EINTR`/`EINVAL`/`ENOMEM`/`ENOSYS`

**配置要求**：`CONFIG_NFILE_DESCRIPTORS > 0` 且未定义 `CONFIG_DISABLE_POLL`

#### select

```c
#include <sys/select.h>
int select(int nfds, FAR fd_set *readfds, FAR fd_set *writefds,
           FAR fd_set *exceptfds, FAR struct timeval *timeout);
```

**描述**：监控多个文件描述符是否就绪。⚠️ `poll()` 是 NuttX 的基础API，`select()` 是 `poll()` 之上的兼容层，资源消耗更大。

| 参数 | 类型 | 说明 |
|------|------|------|
| `nfds` | `int` | 最大文件描述符编号+1 |
| `readfds` | `FAR fd_set *` | 读就绪监控集 |
| `writefds` | `FAR fd_set *` | 写就绪监控集 |
| `exceptfds` | `FAR fd_set *` | 异常监控集 |
| `timeout` | `FAR struct timeval *` | 超时时间 |

**返回值**：`0` 超时；`>0` 就绪描述符数；`-1` 错误

### 12.3 Directory Operations（目录操作）

```c
#include <dirent.h>
int closedir(DIR *dirp);
FAR DIR *opendir(const char *path);
FAR struct dirent *readdir(FAR DIR *dirp);
int readdir_r(FAR DIR *dirp, FAR struct dirent *entry, FAR struct dirent **result);
void rewinddir(FAR DIR *dirp);
void seekdir(FAR DIR *dirp, int loc);
int telldir(FAR DIR *dirp);
```

### 12.4 UNIX Standard Operations

```c
#include <unistd.h>
pid_t getpid(void);
void _exit(int status) noreturn_function;
unsigned int sleep(unsigned int seconds);
void usleep(unsigned long usec);
int close(int fd);
int dup(int fd);
int dup2(int fd1, int fd2);
int fsync(int fd);
off_t lseek(int fd, off_t offset, int whence);
ssize_t read(int fd, FAR void *buf, size_t nbytes);
ssize_t write(int fd, FAR const void *buf, size_t nbytes);
int pipe(int filedes[2]);
int chdir(FAR const char *path);
FAR char *getcwd(FAR char *buf, size_t size);
int unlink(FAR const char *pathname);
int rmdir(FAR const char *pathname);
int getopt(int argc, FAR char *const argv[], FAR const char *optstring);
```

### 12.5 Standard I/O

```c
#include <stdio.h>
int fclose(FILE *stream);
int fflush(FILE *stream);
FILE *fdopen(int fd, const char *type);
int feof(FILE *stream);              /* 已原型声明但未实现 */
int ferror(FILE *stream);            /* 已原型声明但未实现 */
int fileno(FAR FILE *stream);
int fgetc(FILE *stream);
int fgetpos(FILE *stream, fpos_t *pos);
char *fgets(char *s, int n, FILE *stream);
FILE *fopen(const char *path, const char *type);
int fprintf(FILE *stream, const char *format, ...);
int fputc(int c, FILE *stream);
int fputs(const char *s, FILE *stream);
size_t fread(void *ptr, size_t size, size_t n_items, FILE *stream);
int fseek(FILE *stream, long int offset, int whence);
int fsetpos(FILE *stream, fpos_t *pos);
long ftell(FILE *stream);
size_t fwrite(const void *ptr, size_t size, size_t n_items, FILE *stream);
char *gets(char *s);
int printf(const char *format, ...);
int puts(const char *s);
int rename(const char *source, const char *target);
int snprintf(FAR char *buf, size_t size, const char *format, ...);
int sprintf(char *dest, const char *format, ...);
int sscanf(const char *buf, const char *fmt, ...);
int ungetc(int c, FILE *stream);
int vprintf(const char *s, va_list ap);
int vfprintf(FILE *stream, const char *s, va_list ap);
int vsnprintf(FAR char *buf, size_t size, const char *format, va_list ap);
int vsscanf(char *buf, const char *s, va_list ap);
int vsprintf(char *buf, const char *s, va_list ap);
```

⚠️ `feof()` 和 `ferror()` 已原型声明但**未实现**

### 12.6 文件状态操作

```c
#include <sys/stat.h>
int mkdir(FAR const char *pathname, mode_t mode);
int mkfifo(FAR const char *pathname, mode_t mode);
int stat(const char *path, FAR struct stat *buf);
int fstat(int fd, FAR struct stat *buf);

#include <sys/statfs.h>
int statfs(const char *path, struct statfs *buf);
int fstatfs(int fd, struct statfs *buf);
```

### 12.7 Standard String Operations

```c
#include <string.h>
char *strchr(const char *s, int c);
FAR char *strdup(const char *s);
const char *strerror(int);
size_t strlen(const char *);
char *strcat(char *, const char *);
char *strncat(char *, const char *, size_t);
int strcmp(const char *, const char *);
int strncmp(const char *, const char *, size_t);
int strcasecmp(const char *, const char *);
int strncasecmp(const char *, const char *, size_t);
char *strcpy(char *dest, const char *src);
char *strncpy(char *, const char *, size_t);
char *strpbrk(const char *, const char *);
char *strrchr(const char *, int);
size_t strspn(const char *, const char *);
size_t strcspn(const char *, const char *);
char *strstr(const char *, const char *);
char *strtok(char *, const char *);
char *strtok_r(char *, const char *, char **);
void *memset(void *s, int c, size_t n);
void *memcpy(void *dest, const void *src, size_t n);
int memcmp(const void *s1, const void *s2, size_t n);
void *memmove(void *dest, const void *src, size_t count);
#define bzero(s,n) (void)memset(s,0,n)
```

### 12.8 Pipes and FIFOs

#### pipe

```c
#include <unistd.h>
int pipe(int filedes[2]);
```

**描述**：创建一对文件描述符，指向管道inode。`filedes[0]` 用于读取，`filedes[1]` 用于写入。

| 参数 | 类型 | 说明 |
|------|------|------|
| `filedes` | `int[2]` | 接收管道文件描述符的数组 |

**返回值**：`0` 成功；`-1` 失败

#### mkfifo

```c
#include <sys/stat.h>
int mkfifo(FAR const char *pathname, mode_t mode);
```

**描述**：创建FIFO设备驱动文件。与Linux不同，NuttX FIFO不是特殊文件类型而是设备驱动实例。必须同时被读写两端打开才能进行I/O。

| 参数 | 类型 | 说明 |
|------|------|------|
| `pathname` | `FAR const char *` | FIFO路径 |
| `mode` | `mode_t` | 权限（**被忽略**） |

**返回值**：`0` 成功；`-1` 失败

### 12.9 FAT File System Support

#### mkfatfs

```c
#include <nuttx/mkfatfs.h>
int mkfatfs(FAR const char *pathname, FAR struct fat_format_s *fmt);
```

**描述**：在指定块设备上格式化FAT文件系统。

⚠️ 调用者必须确保块驱动未挂载且未在使用中

| 参数 | 类型 | 说明 |
|------|------|------|
| `pathname` | `FAR const char *` | 块驱动路径 |
| `fmt` | `FAR struct fat_format_s *` | 格式化参数 |

`struct fat_format_s` 成员：
| 成员 | 类型 | 说明 |
|------|------|------|
| `ff_nfats` | `uint8_t` | FAT数量 |
| `ff_fattype` | `uint8_t` | FAT大小：0（自动）、12、16或32 |
| `ff_clustshift` | `uint8_t` | 每簇扇区数的log2：0-5、0xff（自动） |
| `ff_volumelabel[11]` | `uint8_t[]` | 卷标 |
| `ff_backupboot` | `uint16_t` | 备份引导扇区号 |
| `ff_rootdirentries` | `uint16_t` | 根目录条目数 |
| `ff_rsvdseccount` | `uint16_t` | 保留扇区数 |
| `ff_hidsec` | `uint32_t` | FAT前隐藏扇区数 |
| `ff_volumeid` | `uint32_t` | FAT卷ID |
| `ff_nsectors` | `uint32_t` | 使用扇区数：0表示全部 |

**返回值**：`0 (OK)` 成功；`-1 (ERROR)` 失败，errno 设置为 `EINVAL`/`ENOENT`/`ENOTBLK`/`EACCESS`

### 12.10 mmap() and eXecute In Place (XIP)

NuttX 在平坦地址空间运行，一般不需要 `mmap()`。唯一例外是支持直接访问随机访问媒体的XIP。

**XIP条件**：
1. 文件系统支持 `FIOC_MMAP` ioctl命令（目前仅ROMFS满足）
2. 底层块驱动支持 `BIOC_XIPBASE` ioctl命令（目前仅RAM/ROM磁盘驱动支持）

#### mmap

```c
#include <sys/mman.h>
FAR void *mmap(FAR void *start, size_t length, int prot, int flags,
                int fd, off_t offset);
```

**描述**：提供最小 `mmap()` 以支持XIP。

| 参数 | 类型 | 说明 |
|------|------|------|
| `start` | `FAR void *` | 映射地址提示——**被忽略** |
| `length` | `size_t` | 映射长度——**被忽略** |
| `prot` | `int` | 保护标志：`PROT_NONE`（错误）、`PROT_READ`/`PROT_WRITE`/`PROT_EXEC` |
| `flags` | `int` | `MAP_SHARED`（必须）、`MAP_PRIVATE`（错误）、`MAP_FIXED`（错误）等 |
| `fd` | `int` | 后备文件描述符——必须 |
| `offset` | `off_t` | 文件内偏移 |

**返回值**：成功返回映射区域指针；失败返回 `MAP_FAILED`，errno 设置为 `ENOSYS`/`EBADF`/`EINVAL`/`ENODEV`

---

## 13. Network 接口（网络）

NuttX 包含基于 uIP 的简单网络接口层，支持标准socket接口的子集。

### 13.1 socket

```c
#include <sys/socket.h>
int socket(int domain, int type, int protocol);
```

**描述**：创建通信端点并返回描述符。

| 参数 | 类型 | 说明 |
|------|------|------|
| `domain` | `int` | 地址族 |
| `type` | `int` | 套接字类型 |
| `protocol` | `int` | 协议 |

**返回值**：`0` 成功；`-1` 失败，errno 设置为 `EACCES`/`EAFNOSUPPORT`/`EINVAL`/`EMFILE`/`ENFILE`/`ENOBUFS`/`EPROTONOSUPPORT`

---

### 13.2 bind

```c
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

**描述**：为套接字分配本地地址。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sockfd` | `int` | 套接字描述符 |
| `addr` | `const struct sockaddr *` | 本地地址 |
| `addrlen` | `socklen_t` | 地址长度 |

**返回值**：`0` 成功；`-1` 失败，errno 设置为 `EACCES`/`EADDRINUSE`/`EBADF`/`EINVAL`/`ENOTSOCK`

---

### 13.3 connect

```c
#include <sys/socket.h>
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

**描述**：连接套接字到指定地址。`SOCK_DGRAM` 设定默认发送/接收地址；`SOCK_STREAM`/`SOCK_SEQPACKET` 尝试建立连接。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sockfd` | `int` | 套接字描述符 |
| `addr` | `const struct sockaddr *` | 服务器地址 |
| `addrlen` | `socklen_t` | 地址长度 |

**返回值**：`0` 成功；`-1` 失败，errno 设置为 `EACCES`/`EADDRINUSE`/`EAFNOSUPPORT`/`EAGAIN`/`EALREADY`/`EBADF`/`ECONNREFUSED`/`EFAULT`/`EINPROGRESS`/`EINTR`/`EISCONN`/`ENETUNREACH`/`ENOTSOCK`/`ETIMEDOUT`

---

### 13.4 listen

```c
#include <sys/socket.h>
int listen(int sockfd, int backlog);
```

**描述**：声明 willingness 接受连接，设置等待队列限制。仅适用于 `SOCK_STREAM` 或 `SOCK_SEQPACKET`。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sockfd` | `int` | 已绑定的套接字描述符 |
| `backlog` | `int` | 待处理连接队列最大长度 |

**返回值**：`0` 成功；`-1` 失败，errno 设置为 `EADDRINUSE`/`EBADF`/`ENOTSOCK`/`EOPNOTSUPP`

---

### 13.5 accept

```c
#include <sys/socket.h>
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

**描述**：从待处理连接队列提取第一个连接请求，创建新已连接套接字。原套接字不受影响。无待处理连接且非阻塞时阻塞（或返回 `EAGAIN`）。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sockfd` | `int` | 监听套接字描述符 |
| `addr` | `struct sockaddr *` | 接收连接客户端地址 |
| `addrlen` | `socklen_t *` | 输入：addr分配大小；返回：实际地址长度 |

**返回值**：成功返回新套接字描述符（非负整数）；`-1` 失败，errno 设置为 `EAGAIN`/`EBADF`/`ENOTSOCK`/`EOPNOTSUPP`/`EINTR`/`ECONNABORTED`/`EINVAL`/`EMFILE`/`ENFILE`/`EFAULT`/`ENOBUFS`/`EPROTO`

---

### 13.6 send

```c
#include <sys/socket.h>
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
```

**描述**：在已连接套接字上发送数据。`flags` 为零时等价于 `write()`。等价于 `sendto(s, buf, len, flags, NULL, 0)`。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sockfd` | `int` | 套接字描述符 |
| `buf` | `const void *` | 要发送的数据 |
| `len` | `size_t` | 数据长度 |
| `flags` | `int` | 发送标志 |

**返回值**：参见 `sendto()`

---

### 13.7 sendto

```c
#include <sys/socket.h>
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
               const struct sockaddr *to, socklen_t tolen);
```

**描述**：发送数据到指定地址。连接模式套接字忽略 `to`/`tolen`。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sockfd` | `int` | 套接字描述符 |
| `buf` | `const void *` | 要发送的数据 |
| `len` | `size_t` | 数据长度 |
| `flags` | `int` | 发送标志 |
| `to` | `const struct sockaddr *` | 接收者地址 |
| `tolen` | `socklen_t` | 地址结构长度 |

**返回值**：成功返回发送字符数；`-1` 失败，errno 设置为 `EAGAIN`/`EBADF`/`ECONNRESET`/`EDESTADDRREQ`/`EFAULT`/`EINTR`/`EINVAL`/`EISCONN`/`EMSGSIZE`/`ENOBUFS`/`ENOMEM`/`ENOTCONN`/`ENOTSOCK`/`EOPNOTSUPP`/`EPIPE`

---

### 13.8 recv

```c
#include <sys/socket.h>
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
```

**描述**：等价于 `recvfrom()` 的 `from` 为NULL版本。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sockfd` | `int` | 套接字描述符 |
| `buf` | `void *` | 接收缓冲区 |
| `len` | `size_t` | 缓冲区长度 |
| `flags` | `int` | 接收标志 |

**返回值**：参见 `recvfrom()`

---

### 13.9 recvfrom

```c
#include <sys/socket.h>
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                 struct sockaddr *from, socklen_t *fromlen);
```

**描述**：从套接字接收消息，可用于连接和非连接套接字。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sockfd` | `int` | 套接字描述符 |
| `buf` | `void *` | 接收缓冲区 |
| `len` | `size_t` | 缓冲区长度 |
| `flags` | `int` | 接收标志 |
| `from` | `struct sockaddr *` | 源地址（非NULL时填充） |
| `fromlen` | `socklen_t *` | 地址结构长度 |

**返回值**：成功返回接收字符数；`-1` 失败，errno 设置为 `EAGAIN`/`EBADF`/`ECONNREFUSED`/`EFAULT`/`EINTR`/`EINVAL`/`ENOMEM`/`ENOTCONN`/`ENOTSOCK`

---

### 13.10 setsockopt

```c
#include <sys/socket.h>
int setsockopt(int sockfd, int level, int option,
               const void *value, socklen_t value_len);
```

**描述**：设置套接字选项。`level` 为 `SOL_SOCKET` 设套接字级选项。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sockfd` | `int` | 套接字描述符 |
| `level` | `int` | 协议级别 |
| `option` | `int` | 选项标识 |
| `value` | `const void *` | 选项值 |
| `value_len` | `socklen_t` | 值长度 |

**返回值**：成功返回字符数；`-1` 失败，errno 设置为 `BADF`/`DOM`/`INVAL`/`ISCONN`/`NOPROTOOPT`/`NOTSOCK`/`NOMEM`/`NOBUFS`

---

### 13.11 getsockopt

```c
#include <sys/socket.h>
int getsockopt(int sockfd, int level, int option,
               void *value, socklen_t *value_len);
```

**描述**：获取套接字选项值。

| 参数 | 类型 | 说明 |
|------|------|------|
| `sockfd` | `int` | 套接字描述符 |
| `level` | `int` | 协议级别 |
| `option` | `int` | 选项标识 |
| `value` | `void *` | 接收选项值 |
| `value_len` | `socklen_t *` | 值长度（输入/输出） |

**返回值**：成功返回字符数；`-1` 失败，errno 设置为 `BADF`/`INVAL`/`NOPROTOOPT`/`NOTSOCK`/`NOBUFS`

---

## 14. OS Data Structures（OS数据结构）

### 14.1 Scalar Types（标量类型）

| 类型 | 说明 |
|------|------|
| `pid_t` | 任务/进程ID |
| `size_t` | 大小 |
| `sigset_t` | 信号集 |
| `time_t` | 日历时间 |

### 14.2 Hidden Interface Structures（隐藏接口结构）

以下结构对应用隐藏，应作为简单句柄引用OS资源：

| 类型 | 说明 |
|------|------|
| `_TCB` | 任务控制块 |
| `mqd_t` | 消息队列描述符 |
| `sem_t` | 信号量 |
| `WDOG_ID` | 看门狗ID |
| `pthread_key_t` | 线程特定数据键 |

⚠️ 应用不应引用这些结构的具体成员以保持可移植性

### 14.3 Access to the errno Variable

```c
#include <errno.h>
#define errno *get_errno_ptr()
int *get_errno_ptr(void);
```

**描述**：返回指向线程特定 errno 值的指针。`errno` 被定义为 `*get_errno_ptr()` 以支持常规访问方式。

⚠️ NuttX 中每个 pthread 都有自己的私有 errno 副本，不共享。这虽非标准但促进更好的线程独立性

### 14.4 User Interface Structures（用户接口结构）

#### main_t

```c
typedef int (*main_t)(int argc, char *argv[]);
```

定义任务入口点类型。

#### struct sched_param

```c
struct sched_param {
    int sched_priority;
};
```

用于传递调度优先级。

#### struct timespec

```c
struct timespec {
    time_t tv_sec;    /* 秒 */
    long   tv_nsec;   /* 纳秒 */
};
```

用于传递时间信息。

#### struct mq_attr

```c
struct mq_attr {
    size_t   mq_maxmsg;   /* 队列最大消息数 */
    size_t   mq_msgsize;  /* 最大消息大小 */
    unsigned mq_flags;    /* 队列标志 */
    size_t   mq_curmsgs;  /* 当前消息数 */
};
```

消息队列属性结构。

#### struct sigaction

```c
struct sigaction {
    union {
        void (*_sa_handler)(int);
        void (*_sa_sigaction)(int, siginfo_t *, void *);
    } sa_u;
    sigset_t sa_mask;
    int      sa_flags;
};
#define sa_handler    sa_u._sa_handler
#define sa_sigaction  sa_u._sa_sigaction
```

信号动作结构。

#### struct siginfo / siginfo_t

```c
typedef struct siginfo {
    int          si_signo;
    int          si_code;
    union sigval si_value;
} siginfo_t;
```

信号参数传递结构。

#### union sigval

```c
union sigval {
    int   sival_int;
    void *sival_ptr;
};
```

信号值联合体。

#### struct sigevent

```c
struct sigevent {
    int          sigev_signo;
    union sigval sigev_value;
    int          sigev_notify;
};
```

信号事件结构，用于消息队列通知。

#### Watchdog Data Types

```c
typedef void (*wdentry_t)(int argc, ...);

union wdparm_u {
    void     *pvarg;
    uint32_t *dwarg;
};
typedef union wdparm_u wdparm_t;
```

看门狗回调函数类型和参数联合体。对于 `sizeof(pointer) < sizeof(uint32_t)` 的系统（如SDCC MCS51），使用联合体保证指针对齐。

---

## 15. 关键风险与注意事项汇总

| 等级 | 风险项 | 来源 | 说明 |
|------|--------|------|------|
| 🔴 高 | `task_delete()` 资源泄漏 | §2.4 | 被删任务持有资源（内存/信号量）将丢失 |
| 🔴 高 | `pthread_key_delete()` 无操作 | §10.25 | 始终返回EINVAL，实际不做任何事 |
| 🔴 高 | `timer_getoverrun()` 未实现 | §8.13 | 接口存在但功能未实现 |
| 🔴 高 | `pthread_testcancelstate()` 不支持 | §10.16 | 接口存在但标记NOT SUPPORTED |
| 🔴 高 | `pthread_key_create()` 析构函数被忽略 | §10.22 | 线程退出时不会自动清理线程特定数据 |
| 🟡 中 | 消息队列最大消息22字节硬编码 | §5.1 | 超过22字节的消息无法发送 |
| 🟡 中 | 信号默认忽略（非终止） | §9 | 与POSIX标准不一致，异常处理逻辑可能偏差 |
| 🟡 中 | `wd_delete()` 不检查使用状态 | §7.2 | 从不返回ERROR，可能释放正在使用的看门狗 |
| 🟡 中 | `PTHREAD_MUTEX_NORMAL` 未实现 | §10.31 | 实际行为为ERRORCHECK，与预期可能不同 |
| 🟡 中 | `sem_post()` ISR调用矛盾 | §6.8 | 描述说可从中断调用但限制说不能 |
| 🟡 中 | `mq_notify()` 通知行为不标准 | §5.8 | 即使有任务在等待也会发通知 |
| 🟡 中 | `timer_settime()` ovalue被忽略 | §8.11 | 无法获取旧定时器值 |
| 🟢 注意 | `exit()` code参数被忽略 | §2.5 | 退出码不传递 |
| 🟢 注意 | `task_restart()` 不支持自重启 | §2.6 | pid=0功能未实现 |
| 🟢 注意 | `gmtime()` 非线程安全 | §8.5 | 返回静态缓冲区，多线程应用应使用 `gmtime_r()` |
| 🟢 注意 | `feof()`/`ferror()` 未实现 | §12.5 | 已原型声明但功能未实现 |
| 🟢 注意 | `localtime` 等于 `gmtime` | §8.6 | 无时区支持 |
| 🟢 注意 | `mkfifo()` mode被忽略 | §12.8 | FIFO权限不生效 |
| 🟢 注意 | `gettimeofday()` tzp被忽略 | §8.14 | 无时区支持 |
| 🟢 注意 | `mmap()` start/length被忽略 | §12.10 | 无MMU支持，地址固定 |

---

## 16. POSIX 兼容性偏离汇总

| 接口 | 偏离点 |
|------|--------|
| `task_create` | 非POSIX，类似VxWorks taskSpawn |
| `task_init` | 非POSIX，类似VxWorks taskInit |
| `task_activate` | 非POSIX，类似VxWorks taskActivate |
| `task_delete` | 非POSIX，不支持删除钩子和自删除 |
| `task_restart` | 非POSIX，不支持自重启 |
| `sched_setparam` | 优先级范围0~255 |
| `sched_getscheduler` | 不通过errno报告错误 |
| `sched_lock` | 非POSIX，类似VxWorks taskLock |
| `sched_unlock` | 非POSIX，类似VxWorks taskUnlock |
| `mq_open` | 最大消息大小22字节 |
| `mq_notify` | 即使有任务等待也发通知（与POSIX不一致） |
| `sem_init` | pshared未使用 |
| `sem_open` | 链接/连接处理高度简化 |
| `sem_unlink` | 后续sem_open可能引用同一信号量 |
| `wd_*` | 全部非POSIX，类似VxWorks |
| `timer_create` | 仅支持CLOCK_REALTIME |
| `timer_settime` | ovalue参数被忽略 |
| `timer_getoverrun` | 未实现 |
| `sigaction` | 不处理SIG_DFL/SIG_IGN；所有sa_flags被忽略 |
| `sigsuspend` | 恢复原始掩码（POSIX未说明）；不要求信号被捕获 |
| `sigtimedwait` | si_code值不同；无ERROR原因机制；不要求信号被捕获 |
| `sigqueue` | 默认忽略；无空信号（signo=0会发送） |
| `kill` | 默认忽略；不支持进程组 |
| `pthread_cancel` | 析构函数不支持；取消类型不支持；取消点不支持 |
| `pthread_key_create` | 析构函数被忽略 |
| `pthread_key_delete` | 实际无操作 |
| `pthread_mutexattr_settype` | NORMAL类型未实现，行为为ERRORCHECK |
| `pthread_mutex_lock` | NORMAL行为实际为ERRORCHECK |
| `localtime` | 等于gmtime（无时区） |
| `mmap` | 仅支持XIP场景；start/length被忽略 |
| `errno` | 每个pthread有独立副本（非共享） |

---

## 17. 配置宏速查表

| 配置宏 | 影响范围 | 说明 |
|--------|----------|------|
| `CONFIG_DEV_CONSOLE` | Task Control | 定义时为新任务复制stdin/stdout/stderr |
| `CONFIG_MAX_TASK_ARGS` | Task Control | 任务最大参数个数 |
| `CONFIG_RR_INTERVAL` | Task Scheduling | 轮转调度时间片间隔 |
| `CONFIG_PRIORITY_INHERITANCE` | Semaphore | 启用优先级继承 |
| `CONFIG_SEM_PREALLOCHOLDERS` | Semaphore | 信号量持有者结构池大小 |
| `CONFIG_SEM_NNESTPRIO` | Semaphore | 每线程嵌套优先级数组大小 |
| `CONFIG_MUTEX_TYPES` | Pthread | 启用互斥锁类型支持 |
| `CONFIG_DISABLE_ENVIRON` | Environment | 定义时禁用全部环境变量支持 |
| `CONFIG_NFILE_DESCRIPTORS` | File System | 文件描述符数量（>0启用文件系统） |
| `CONFIG_DISABLE_POLL` | File System | 定义时禁用poll支持 |
| `CONFIG_NET` | Network | 启用通用网络支持 |
| `CONFIG_NET_TCP` | Network | 启用TCP/IP支持 |
| `CONFIG_NSOCKET_DESCRIPTORS` | Network | 套接字描述符数量 |
| `CONFIG_NET_NTCP_READAHEAD_BUFFERS` | Network | TCP预读缓冲区数 |
| `CONFIG_NET_TCPBACKLOG` | Network | TCP连接等待队列 |

---

> **文档说明**：本文档基于 NuttX Operating System User's Manual（Gregory Nutt, 2009-12-13, 134页）完整提取编写，覆盖全部12大类、130+个C可调用接口的函数原型、参数、返回值、使用要点、POSIX兼容性差异和风险提示。文档中标注 ⚠️ 的内容需要特别关注。
---

### 3.3 sched_setscheduler

```c
#include <sched.h>
int sched_setscheduler(pid_t pid, int policy, const struct sched_param *param);
```

**描述**：同时设置指定任务的调度策略和优先级。

| 参数 | 类型 | 说明 |
|------|------|------|
| `pid` | `pid_t` | 任务ID，0表示调用任务 |
| `policy` | `int` | 调度策略：`SCHED_FIFO` 或 `SCHED_RR` |
| `param` | `const struct sched_param *` | 包含优先级 |

**返回值**：`OK (0)` 成功；`ERROR (-1)` 失败，errno 设置为 `EINVAL`（策略无效）或 `ESRCH`（任务未找到）

**POSIX兼容性**：与同名POSIX接口可比

---

### 3.4 sched_getscheduler

```c
#include <sched.h>
int sched_getscheduler(pid_t pid);
```

**描述**：返回指定任务当前的调度策略。

| 参数 | 类型 | 说明 |
|------|------|------|
| `pid` | `pid_t` | 任务ID，0表示调用任务 |

**返回值**：成功返回策略（`SCHED_FIFO` 或 `SCHED_RR`）；失败返回 `ERROR (-1)`

**POSIX兼容性**：与同名POSIX接口可比。差异：不通过errno报告错误

---

### 3.5 sched_yield

```c
#include <sched.h>
int sched_yield(void);
```

**描述**：强制调用任务让出CPU（仅让给同优先级任务）。

**参数**：无

**返回值**：`0 (OK)` 或 `-1 (ERROR)`

**POSIX兼容性**：与同名POSIX接口可比

---

### 3.6 sched_get_priority_max

```c
#include <sched.h>
int sched_get_priority_max(int policy);
```

**描述**：返回指定调度策略的最高可能优先级值。

| 参数 | 类型 | 说明 |
|------|------|------|
| `policy` | `int` | 调度策略 |

**返回值**：最大优先级值或 `-1 (ERROR)`

**POSIX兼容性**：与同名POSIX接口可比

---

### 3.7 sched_get_priority_min

```c
#include <sched.h>
int sched_get_priority_min(int policy);
```

**描述**：返回指定调度策略的最低可能优先级值。

| 参数 | 类型 | 说明 |
|------|------|------|
| `policy` | `int` | 调度策略 |

**返回值**：最小优先级值或 `-1 (ERROR)`

**POSIX兼容性**：与同名POSIX接口可比

---

### 3.8 sched_get_rr_interval

```c
#include <sched.h>
int sched_get_rr_interval(pid_t pid, struct timespec *interval);
```

**描述**：获取指定任务的时间片间隔（轮转调度）。任务应在SCHED_RR策略下运行。

| 参数 | 类型 | 说明 |
|------|------|------|
| `pid` | `pid_t` | 任务ID，0表示调用任务 |
| `interval` | `struct timespec *` | 返回时间片的结构体 |

**返回值**：`OK (0)` 成功；`ERROR (-1)` 失败，errno 设置为 `EFAULT`/`EINVAL`/`ENOSYS`/`ESRCH`

**POSIX兼容性**：与同名POSIX接口可比

---

## 4. Task Switching 接口（任务切换）

### 4.1 sched_lock

```c
#include <sched.h>
int sched_lock(void);
```

**描述**：禁止上下文切换——阻止新任务加入就绪队列。调用此函数的任务将是唯一允许运行的任务，直到调用 `sched_unlock()` 相应次数或自身阻塞。

**参数**：无

**返回值**：`OK` 或 `ERROR`

⚠️ **注意**：`sched_lock()` 会阻止所有更高优先级任务运行（不限于信号量管理的资源），可能影响系统响应时间。优先使用信号量而非 `sched_lock()`

**POSIX兼容性**：**非POSIX接口**。类似 VxWorks `taskLock()`

---

### 4.2 sched_unlock

```c
#include <sched.h>
int sched_unlock(void);
```

**描述**：递减抢占锁计数。与 `sched_lock()` 配对使用，结束临界区。锁计数减到零时，有资格抢占的任务才会执行。

**参数**：无

**返回值**：`OK` 或 `ERROR`

**POSIX兼容性**：**非POSIX接口**。类似 VxWorks `taskUnlock()`

---

### 4.3 sched_lockcount

```c
#include <sched.h>
int32_t sched_lockcount(void);
```

**描述**：返回当前锁计数值。零表示抢占已启用；非零表示 `sched_lock()` 被调用的次数。

**参数**：无

**返回值**：当前锁计数值

**POSIX兼容性**：无

---

## 5. Named Message Queue 接口（命名消息队列）

NuttX 支持POSIX命名消息队列用于任务间通信。任何任务可在命名消息队列上发送或接收消息。中断处理程序也可通过命名消息队列发送消息。

### 5.1 mq_open

```c
#include <mqueue.h>
mqd_t mq_open(const char *mqName, int oflags, ...);
```

**描述**：建立命名消息队列与调用任务之间的连接。成功后可使用返回的描述符引用该队列。

| 参数 | 类型 | 说明 |
|------|------|------|
| `mqName` | `const char *` | 队列名称 |
| `oflags` | `int` | 打开标志：`O_RDONLY`/`O_WRONLY`/`O_RDWR`/`O_CREAT`/`O_EXCL`/`O_NONBLOCK` |
| `...` | 可变 | 当 `O_CREAT` 指定时需提供：`mode`（mode_t类型，必须但未使用）、`attr`（mq_attr指针，初始化队列属性；NULL则使用默认值） |

**返回值**：消息队列描述符或 `-1 (ERROR)`

⚠️ **重要限制**：`mq_msgsize`（最大消息大小）当前实现限制为 **22字节**

**POSIX兼容性**：与同名POSIX接口可比。差异：最大消息大小限制为22字节

---

### 5.2 mq_close

```c
#include <mqueue.h>
int mq_close(mqd_t mqdes);
```

**描述**：指示调用任务已完成使用指定消息队列。释放系统为此任务分配的资源。如果已通过 `mq_notify()` 附加通知请求，将被移除。

| 参数 | 类型 | 说明 |
|------|------|------|
| `mqdes` | `mqd_t` | 消息队列描述符 |

**返回值**：`0 (OK)` 成功；`-1 (ERROR)` 失败

⚠️ **限制**：任务在 `mq_send()` 或 `mq_receive()` 上阻塞时调用 `mq_close()` 的行为未定义

**POSIX兼容性**：与同名POSIX接口可比

---

### 5.3 mq_unlink

```c
#include <mqueue.h>
int mq_unlink(const char *mqName);
```

**描述**：移除指定名称的消息队列。如果尚有任务打开该队列，移除延迟到所有引用关闭后。

| 参数 | 类型 | 说明 |
|------|------|------|
| `mqName` | `const char *` | 消息队列名称 |

**返回值**：无

**POSIX兼容性**：与同名POSIX接口可比

---

### 5.4 mq_send

```c
#include <mqueue.h>
int mq_send(mqd_t mqdes, const void *msg, size_t msglen, int prio);
```

**描述**：将消息添加到消息队列。按优先级插入（高优先级在低优先级之前）。队列满且未设 `O_NONBLOCK` 时阻塞；设了 `O_NONBLOCK` 则立即返回错误。

| 参数 | 类型 | 说明 |
|------|------|------|
| `mqdes` | `mqd_t` | 消息队列描述符 |
| `msg` | `const void *` | 要发送的消息 |
| `msglen` | `size_t` | 消息长度（字节），不超过 `mq_getattr()` 的最大消息长度 |
| `prio` | `int` | 消息优先级，不超过 `MQ_PRIO_MAX` |

**返回值**：`0 (OK)` 成功；`-1 (ERROR)` 失败，errno 设置为 `EAGAIN`/`EINVAL`/`EPERM`/`EMSGSIZE`/`EINTR`

**POSIX兼容性**：与同名POSIX接口可比

---

### 5.5 mq_timedsend

```c
#include <mqueue.h>
int mq_timedsend(mqd_t mqdes, const char *msg, size_t msglen, int prio,
                 const struct timespec *abstime);
```

**描述**：类似 `mq_send()`，但队列满且未设 `O_NONBLOCK` 时，只阻塞到 `abstime` 指定的绝对超时时间。超时已过期则立即返回。

| 参数 | 类型 | 说明 |
|------|------|------|
| `mqdes` | `mqd_t` | 消息队列描述符 |
| `msg` | `const char *` | 要发送的消息 |
| `msglen` | `size_t` | 消息长度 |
| `prio` | `int` | 消息优先级 |
| `abstime` | `const struct timespec *` | 绝对超时时间（自Epoch起的秒和纳秒） |

**返回值**：`0 (OK)` 成功；`-1 (ERROR)` 失败，errno 同 `mq_send()` 外加 `ETIMEDOUT`

**POSIX兼容性**：与同名POSIX接口可比

---

### 5.6 mq_receive

```c
#include <mqueue.h>
ssize_t mq_receive(mqd_t mqdes, void *msg, size_t msglen, int *prio);
```

**描述**：接收消息队列中最高优先级的最旧消息。如果缓冲区小于 `mq_msgsize`，返回错误。队列为空且未设 `O_NONBLOCK` 时阻塞。

| 参数 | 类型 | 说明 |
|------|------|------|
| `mqdes` | `mqd_t` | 消息队列描述符 |
| `msg` | `void *` | 接收消息的缓冲区 |
| `msglen` | `size_t` | 缓冲区大小（字节） |
| `prio` | `int *` | 非NULL时存储消息优先级 |

**返回值**：成功返回接收消息的长度（字节）；失败返回 `-1 (ERROR)`，errno 设置为 `EAGAIN`/`EPERM`/`EMSGSIZE`/`EINTR`/`EINVAL`

**POSIX兼容性**：与同名POSIX接口可比

---

### 5.7 mq_timedreceive

```c
#include <mqueue.h>
ssize_t mq_timedreceive(mqd_t mqdes, void *msg, size_t msglen,
                         int *prio, const struct timespec *abstime);
```

**描述**：类似 `mq_receive()`，但队列为空且未设 `O_NONBLOCK` 时，只阻塞到 `abstime` 指定的绝对超时。

| 参数 | 类型 | 说明 |
|------|------|------|
| `mqdes` | `mqd_t` | 消息队列描述符 |
| `msg` | `void *` | 接收消息的缓冲区 |
| `msglen` | `size_t` | 缓冲区大小 |
| `prio` | `int *` | 非NULL时存储消息优先级 |
| `abstime` | `const struct timespec *` | 绝对超时时间 |

**返回值**：成功返回消息长度；失败返回 `-1 (ERROR)`，errno 同 `mq_receive()` 外加 `ETIMEDOUT`

**POSIX兼容性**：与同名POSIX接口可比

---

### 5.8 mq_notify

```c
#include <mqueue.h>
int mq_notify(mqd_t mqdes, const struct sigevent *notification);
```

**描述**：当消息队列从空变为非空时，向注册任务发送指定信号。每个队列只能附加一个通知。通知发送后注册自动移除。`notification` 为NULL时取消注册。

| 参数 | 类型 | 说明 |
|------|------|------|
| `mqdes` | `mqd_t` | 消息队列描述符 |
| `notification` | `const struct sigevent *` | 通知配置：`sigev_notify`（应为SIGEV_SIGNAL，实际被忽略）、`sigev_signo`（信号编号）、`sigev_value`（信号关联值） |

**返回值**：无

⚠️ **POSIX偏离**：即使另一个任务正在等待消息队列变为非空，通知信号也会发送给注册任务——这与POSIX规范不一致

**POSIX兼容性**：与同名POSIX接口可比。差异：通知信号行为与POSIX规范不一致

---

### 5.9 mq_setattr

```c
#include <mqueue.h>
int mq_setattr(mqd_t mqdes, const struct mq_attr *mqStat,
               struct mq_attr *oldMqStat);
```

**描述**：设置消息队列属性。只有 `mq_flags` 中的 `O_NONBLOCK` 位可被修改。`oldMqStat` 非NULL时保存旧属性。

| 参数 | 类型 | 说明 |
|------|------|------|
| `mqdes` | `mqd_t` | 消息队列描述符 |
| `mqStat` | `const struct mq_attr *` | 新属性 |
| `oldMqStat` | `struct mq_attr *` | 旧属性存储位置（可NULL） |

**返回值**：`0 (OK)` 成功；`-1 (ERROR)` 失败

**POSIX兼容性**：与同名POSIX接口可比

---

### 5.10 mq_getattr

```c
#include <mqueue.h>
int mq_getattr(mqd_t mqdes, struct mq_attr *mqStat);
```

**描述**：获取消息队列的状态信息和属性。

| 参数 | 类型 | 说明 |
|------|------|------|
| `mqdes` | `mqd_t` | 消息队列描述符 |
| `mqStat` | `struct mq_attr *` | 接收属性的缓冲区 |

**返回属性**：`mq_maxmsg`（最大消息数）、`mq_msgsize`（最大消息大小）、`mq_flags`（队列标志）、`mq_curmsgs`（当前消息数）

**返回值**：`0 (OK)` 成功；`-1 (ERROR)` 失败

**POSIX兼容性**：与同名POSIX接口可比

