---
title: vivo TWS 耳机应用框架技术文档
date: 2026-10-27
description: vivo TWS 耳机应用框架解析：QCC 线程机制、消息订阅发布、模块开发流程与开发规范。
tags: [TWS, 嵌入式, 应用框架, 蓝牙]
---

# vivo TWS 耳机应用框架技术文档

> 本文档基于 vivo TWS 内部培训材料整理，涵盖应用框架架构、QCC 线程机制、仓库管理、开发流程及开发规范等核心内容。

------

## 目录

1. TWS 应用框架与机制
   - 1.1 [耳机 TWS 应用架构分层](#vivo-tws-耳机应用框架技术文档-一tws-应用框架与机制-11-耳机-tws-应用架构分层)
   - 1.2 [QCC 线程机制：thread 划分](#vivo-tws-耳机应用框架技术文档-一tws-应用框架与机制-12-qcc-线程机制thread-划分)
   - 1.3 [QCC 线程机制：task 与 tasklist 区别](#vivo-tws-耳机应用框架技术文档-一tws-应用框架与机制-13-qcc-线程机制task-与-tasklist-区别)
   - 1.4 [仓库划分与平台切换控制](#vivo-tws-耳机应用框架技术文档-一tws-应用框架与机制-14-仓库划分与平台切换控制)
   - 1.5 [新应用框架重点改变](#vivo-tws-耳机应用框架技术文档-一tws-应用框架与机制-15-新应用框架重点改变)
   - 1.6 [消息订阅-发布过程](#vivo-tws-耳机应用框架技术文档-一tws-应用框架与机制-16-消息订阅-发布过程)
   - 1.7 [TWS 消息机制扩展：同步发送](#vivo-tws-耳机应用框架技术文档-一tws-应用框架与机制-17-tws-消息机制扩展同步发送)
   - 1.8 [模块开发两个基本原则与 ADK 管控](#vivo-tws-耳机应用框架技术文档-一tws-应用框架与机制-18-模块开发两个基本原则与-adk-管控)
   - 1.9 [TWS 各层级调用框图总览](#vivo-tws-耳机应用框架技术文档-一tws-应用框架与机制-19-tws-各层级调用框图总览)
2. 开发流程
   - 2.1 [自顶向下设计 + 自底向上实现](#vivo-tws-耳机应用框架技术文档-二开发流程-21-自顶向下设计-自底向上实现)
   - 2.2 [以机盒模块为例的开发过程](#vivo-tws-耳机应用框架技术文档-二开发流程-22-以机盒模块为例的开发过程)
   - 2.3 [实现 sdk_hal_uartcase.c 文件](#vivo-tws-耳机应用框架技术文档-二开发流程-23-实现-sdkhaluartcasec-文件)
   - 2.4 [实现 sdk_hal_5V_detect.c 文件](#vivo-tws-耳机应用框架技术文档-二开发流程-24-实现-sdkhal5vdetectc-文件)
   - 2.5 [实现 vivo_casebox_service.c 文件](#vivo-tws-耳机应用框架技术文档-二开发流程-25-实现-vivocaseboxservicec-文件)
   - 2.6 [实现 vivo_casebox_manager.c 文件](#vivo-tws-耳机应用框架技术文档-二开发流程-26-实现-vivocaseboxmanagerc-文件)
   - 2.7 [实现 ui_casebox.c 文件](#vivo-tws-耳机应用框架技术文档-二开发流程-27-实现-uicaseboxc-文件)
3. 开发规范
   - 3.1 [命名规范](#vivo-tws-耳机应用框架技术文档-三开发规范-31-命名规范)
   - 3.2 [消息 ID 规范](#vivo-tws-耳机应用框架技术文档-三开发规范-32-消息-id-规范)
   - 3.3 [修改权限规范](#vivo-tws-耳机应用框架技术文档-三开发规范-33-修改权限规范)
   - 3.4 [SDK 修改宏管控](#vivo-tws-耳机应用框架技术文档-三开发规范-34-sdk-修改宏管控)
   - 3.5 [消息发送 API 说明](#vivo-tws-耳机应用框架技术文档-三开发规范-35-消息发送-api-说明)
   - 3.6 [task 和 tasklist 声明规范](#vivo-tws-耳机应用框架技术文档-三开发规范-36-task-和-tasklist-声明规范)
   - 3.7 [其他开发规范](#vivo-tws-耳机应用框架技术文档-三开发规范-37-其他开发规范)

------

## 一、TWS 应用框架与机制

### 1.1 耳机 TWS 应用架构分层

vivo TWS 耳机应用框架采用分层架构设计，从上到下依次分为：**App 层（UI）**、**Framework 框架层**、**Platform Abstraction Layer（平台抽象层）** 和 **SDK 层**。

```
┌─────────────────────────────────────────────────────────────────────┐
│  App (UI)                                                            │
│  System app | Battery app | Connect app | Input app | Interact app   │
│  OTA app | ...app                                    | Exchange       │
└─────────────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────────────┐
│  Framework                                                            │
│  ┌──────────────┐  ┌──────────────────────────────────────────────┐ │
│  │ base         │  │ vivo Framework APIs (Manager)                │ │
│  │ Message      │  │  Battery | Connect | Input | Audio | DFX     │ │
│  │ Define       │  │  OTA | Factory | ...                         │ │
│  │ OS(system)   │  │ (Service层)                                   │ │
│  │ Common       │  └──────────────────────────────────────────────┘ │
│  └──────────────┘                                   Component/Sensor │
└─────────────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────────────┐
│  Platform Abstraction Layer                                           │
│  SDK OSAL | SDK HAL | SDK BT API | SDK Audio API | ...API            │
│  vivo SDK Adapt                                                       │
└─────────────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────────────┐
│  SDK                                                                  │
│  QCC Platform | BES Platform | Airoha Platform | Wuqi Platform | ... │
└─────────────────────────────────────────────────────────────────────┘
```

**各层职责说明：**

| 层级                       | 角色                                | 说明                                                       |
| -------------------------- | ----------------------------------- | ---------------------------------------------------------- |
| App（UI）                  | Message链/API接口的最终消费者       | 各功能 App，如系统、电量、连接、输入、交互、OTA 等         |
| Framework - Manager        | 将 Service 的接口封装，提供给 UI 层 | 管理某类功能，比如双连接管理                               |
| Framework - Service        | 提供某类功能的管理，例如双连接      | 内部实现功能逻辑，向上暴露接口                             |
| Platform Abstraction Layer | 平台抽象层（橙色标注）              | 兼容各平台，**向上提供统一的接口**，切换平台时只需修改此层 |
| SDK                        | Platform SDK 看成一个整体           | QCC / BES / Airoha / Wuqi 等各芯片平台 SDK                 |

**框架层目录树（与分层一致）：**

```
vivo_TWS_OS/
└── framework/
    ├── base/
    ├── components/
    ├── SDK_AL/
    ├── vivo_manager/
    ├── vivo_services/
    └── ui/
```

> **关键理解**：Manager 是对 Service 接口的封装，向上提供给 UI 层；Service 是实际功能实现者。平台抽象层屏蔽底层 SDK 差异，是跨平台移植的核心。

------

### 1.2 QCC 线程机制：thread 划分

QCC 平台将核心任务分为两个处理器核（P0 / P1），通过 IPC 机制通信。

```
┌─────────┐
│   P0    │   （主核，如蓝牙协议栈等）
└────┬────┘
     │  IPC 机制
┌────▼─────────────────────────────────────────────┐
│   P1                                              │
│  ┌─────────────────────────────────────────────┐ │
│  │              thread                          │ │
│  │  IPC task   ──►  BT task                    │ │
│  │  Sensor task ──► Audio task                 │ │
│  │  VM task    ──►  驱动 (uart/battery)         │ │
│  │  idle task  ──►  UI task                    │ │
│  │             ──►  other task                  │ │
│  └─────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

**task 的本质：各模块的函数指针**

task 相当于各模块注册到线程中的一个处理函数指针。VM_thread 的工作机制伪代码如下：

```c
VM_thread()
{
    MSG_T* msg;
    while(1) {
        OS_GET_MSG(msg);
        handle_t* handle = msg->task;
        if(handle) {
            handle(msg);
        }
        OS_FREE(msg);
    }
}
```

**核心原则：**

- 蓝牙、音频、驱动、UI 使用**相同的消息队列**，当 GET 到消息时，从消息中取出 task 的函数指针，然后执行。
- 注意：蓝牙、音频、驱动、UI 都是运行在**同一个 thread** 中，所以**不允许使用硬延时**，单次处理消息尽可能耗时短。

------

### 1.3 QCC 线程机制：task 与 tasklist 区别

**消息发送方式对比：**

| 发送方式             | API                                                          | 行为                              |
| -------------------- | ------------------------------------------------------------ | --------------------------------- |
| 单个发送（task）     | `MessageSend(task, id, message)`                             | 发送给单个 handle，插入到队列末尾 |
| 遍历发送（tasklist） | `TaskList_MessageSendWithSize(list, id, message, size_message)` | 遍历 task_list，向所有注册者广播  |

**VM_thread 消息处理流程（伪代码）：**

```c
VM_thread()
{
    while(1) {
        vm_message_next();
    }
}

static uint32 vm_message_next(void)
{
    AppMessage **p = &vm_message_queue;
    /* a = queue_first */
    if (ta->handler != NULL) {
        a->task->handler(a->task, id, message);  // 单个发送
    } else {
        while (*tptr != NULL) {
            (*tptr)->handler(*tptr, id, message); // 遍历发送
            tptr++;
        }
    }
    message_release(&message);
}
```

**task vs tasklist 区别说明：**

1. **task** 代表一个 handle，对 task 发送主要用于本地 task 中做**异步**延时处理，比如 `send_later`，或者明确知道通知其他模块的 task（比如通过 include "xxx.h" 从头文件中拿到其他模块 task，然后对这个模块发送）。不建议这么做，task 应该是对其他模块不可见的。
2. **tasklist** 是保存 task 的一个链表，一般用于开放接口供人 `add_task()`。当本模块有消息产生时，**广播通知所有注册的模块**。

```
task_list:
    handle_00;
    handle_10;
    handle_20;
    handle_30;
    ...
```

**tasklist 广播示意：**

```
                    ┌──────┐   ┌──────┐
                    │模块2 │   │模块3 │
                    └──────┘   └──────┘
         ┌──────┐              ┌──────┐
         │模块1 │◄── taskList ─►│模块4 │
         └──────┘              └──────┘
    task      ┌──────┐   ┌──────┐
              │模块6 │   │模块5 │
              └──────┘   └──────┘
```

------

### 1.4 仓库划分与平台切换控制

**仓库结构：仍然分为两个仓库——SDK 仓和 Earbud_Application 仓**

```
SDK 仓                          Earbud_Application 仓
┌──────────────┐               ┌─────────────────────────────┐
│ QCC SDK      │──────────────►│ QCC SDK 抽象层              │
│              │               │                             │
│ BES SDK      │──────────────►│ BES SDK 抽象层              │
│              │               │  SDK 抽象层公共头文件        │
│ Airoha SDK   │──────────────►│ Airoha SDK 抽象层           │
│              │               │  vivo Framework             │
│ Wuqi SDK     │──────────────►│ Wuqi SDK 抽象层             │
│              │               │  manager/service            │
│ ...SDK       │──────────────►│ other SDK 抽象层            │
└──────────────┘               │                             │
                               │ vivo框架头文件  vivo APP    │
                               └─────────────────────────────┘
```

**仓库划分原因：**

- 考虑到 SDK 仓会**开放给厂商**，SDK 独立有利于 SDK 更新、对齐。
- SDK 仓独立，有利于追溯 SDK 的改动。
- **平台切换：使用宏控制**，用宏修改 Makefile 路径。
- 必须确保 **SDK 抽象层的文件、函数名各平台统一**。如果某个平台没有，则要补齐。平台无法实现则函数内留空，但不影响上层使用。

------

### 1.5 新应用框架重点改变

新框架相比旧框架有两大核心改变：

#### 改变一：应用与平台交互——SDK 抽象层

|          | 旧 OS                                                        | 新 OS                                                        |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 方式     | 各个模块自行适配各自模块的平台接口，缺点：重复适配、代码风格不统一 | 平台接口统一抽象化，代码风格统一                             |
| SDK 接口 | 难以区分哪些是 SDK 应该提供的接口                            | SDK 接口集中到同一文件夹内，可清晰区分 SDK 接口，切换平台时只需修改抽象层的具体实现 |

**架构对比示意：**

```
旧架构：                          新架构：
应用层: app1 app2 app3 app4      应用层: app1 app2 app3 app4
适配层: Adapt1 Adapt2 Adapt3 Adapt4   新框架层: manager/service
platform（各自适配）              统一抽象层: Platform Abstraction Layer
                                  platform1 platform2 platform3 platform4
```

#### 改变二：模块与模块交互——OS 消息机制解耦

|      | 旧 OS                                                  | 新 OS                                                        |
| ---- | ------------------------------------------------------ | ------------------------------------------------------------ |
| 方式 | 各模块之间相互发送消息，或者直接调用接口，**耦合严重** | 采用**中介者模式**与**订阅-发布**思想                        |
| 核心 | 模块间直接调用                                         | 以 OS+MSG 为核心，任一模块只从 OS 接收/发送消息，不用关注谁产生的，也不用关注谁处理，只需处理好自己的部分 |

**架构对比示意：**

```
旧架构（网状，耦合重）：          新架构（OS 为中心，解耦）：
模块2 ─── 模块3                   模块2   模块3
  │  ╲   ╱  │                      │       │
模块1   ╳   模块4       ─►     模块1 ─── OS ─── 模块4
  │  ╱   ╲  │                      │       │
模块6 ─── 模块5                   模块6   模块5
```

------

### 1.6 消息订阅-发布过程

> 以下以**电压/电量 Msg** 为例，说明消息订阅-发布的完整流程（实际代码会根据响应速度决定是否走系统总线）。

**原则：**

- 任意一个 APP 或功能向 system OS 订阅消息。
- 当消息产生时，system OS 向指定的订阅者发送消息。
- `battery service` 不用关注谁产生的"电压 MSG"，也无需关注需要"电量 MSG"，只需处理好拿到电压后，经算法换算出正确的电量送出去（只做一件事，并把事情做正确）。
- 对于系统流程分析，可以以 MSG 为节点跟踪，就可以清晰地知道各个模块做了什么事情。

**电量消息流转图：**

```
┌─────────────────────────────────────────────────────────┐
│              SYSTEM  OS(system task_list)                │
│         ↑订阅电压MSG          ↑订阅电量MSG               │
│         │接收电压MSG          │接收电量MSG               │
│    ┌────┴──────┐         ┌────┴────────────────────┐   │
│    │发送电压MSG│         │发送完量MSG               │   │
│    │接收电压MSG│         │                          │   │
│    └────┬──────┘         └──┬──────────────────────┘   │
└─────────┼───────────────────┼──────────────────────────┘
          │                   │
    ┌─────┴────┐         ┌────┴──────────────────────────┐
    │ ADC 中断 │         │ Battery Service               │
    │          │         │ 电量换算算法                   │
    └──────────┘         └────────────────────────────────┘
                              │
                 ┌────────────┼────────────┐
                 │            │            │
          ┌──────┴──┐  ┌──────┴──┐  ┌─────┴───┐
          │电量APP   │  │蓝牙APP  │  │音效APP  │
          │判断是否  │  │上报手机 │  │更新显示 │
          │低电量天线│  │主从切换 │  │提示音   │
          └─────────┘  └─────────┘  └─────────┘
```

------

### 1.7 TWS 消息机制扩展：同步发送

**两种扩展场景：**

1. 考虑一些需要实时响应的场景，需要立即上报、执行某些消息 → **同步执行的消息机制（阻塞）**
2. 考虑到一些消息需要执行完成后，需要反馈给发送者 → **发送时需要添加 `done_callback` 函数指针（待实现）**

**同步发送 API（待实现）：**

```c
void sdk_MessageSend_DoneCb(Task task, MessageId id, void *message, func_t CB_func);
// 还在实现中，具体待定义
```

**同步发送过程**

以 UART 接收数据为例，数据到来后同步通知各订阅者：

```c
struct RX_MSG_T {
    uint16 rx_len;
    uint16 tx_len;
    uint8* rx_data;
    uint8* tx_data;
};

void uart_rx_handle(uint8* data, uint16 len)
{
    RX_MSG_T msg;
    msg->rx_len = len;
    msg->rx_data = data;
    msg->tx_len  = 0;

    sdk_TaskList_MessageExecuteImmediate(
        notify_task_list, MSG_RX_DATA, &msg);

    // 消息执行完，回到这里
    if(msg->tx_len > 0) {
        uart_send(msg->tx_len, msg->tx_data);
        free(msg->tx_data);
    }
}
```

**同步遍历执行的核心 API：**

```c
void sdk_TaskList_MessageExecuteImmediate(task_list_t *list, MessageId id, void *message)
{
    for (uint j = 0; j < taskListSize; j++) {
        task.handle(task, id, msg);
    }
}
```

> **关键**：同步执行的过程中，将 tasklist 用 for 循环遍历，确保所有订阅者都同步处理完消息后才继续后续逻辑。

------

### 1.8 模块开发两个基本原则与 ADK 管控

**两个原则：**

1. **框架内不允许直接调用 ADK 接口**（SDK AL 部分除外）。
2. **ADK 修改需要加宏。**

**模块开发架构图：**

```
模块开发者              大 SE（平台SE）             模块开发者
自身功能开发                                       自身功能开发

┌──────────────┐    ┌──────────────────┐    ┌──────────────┐
│  UI          │    │  UI exchange     │    │  UI          │
│  Service/    │    │  vivo base       │    │  Service/    │
│  Manager     │    │  SDK AL Common   │    │  Manager     │
│  SDK AL      │    │                  │    │  SDK AL      │
└──────────────┘    └──────────────────┘    └──────────────┘
    模块SE                                       模块SE
                    ┌──────────────────┐
连接+音频           │                  │
模块修改ADK ───────►│    ADK           │
                    │                  │
                    └──────────────────┘
```

**模块与模块交互部分：**

模块 SE 提出修改策略，并同步到大 SE，讨论可行性后再实现。

**SDK AL（SDK 抽象层）：**

需要 ADK 接口，应当由**模块开发者**提出接口需求，**模块 SE** 完成实现。如果需要修改 ADK，模块 SE 提出修改原因，然后与大 SE 讨论可行性后再修改。

------

### 1.9 TWS 各层级调用框图总览

最后一张图（image22）展示了 TWS **各层级调用框图**全貌，展示了从底层 SDK 抽象层到 Service、Manager，再到最终 APP 层的完整调用关系。各平台需要有**相同的抽象层接口**，这是跨平台一致性的核心保障。

------

## 二、开发流程

### 2.1 自顶向下设计 + 自底向上实现

开发流程分为**自顶向下设计**和**自底向上实现**两个阶段，针对 UI、Service/Manager、SDK AL 三层分别展开：

```
自顶向下设计（设计阶段）
─────────────────────────────────────────────────────────────────
考虑整体功能         ┌──────────────┐   消费产品（接收msg、调用API实现功能）
抽象化UI层需求  ──►  │      UI      │
（消费哪些msg、API）  └──────────────┘
                             ▼
进一步拆解UI需求      ┌──────────────┐   根据原材料制造产品（生产msg、API）
细化Service目标  ──► │Service/Manager│
（怎么生产/转发        └──────────────┘
  msg、API）                  ▼
                      ┌──────────────┐
拆解生产原料     ──►  │    SDK AL    │   提供生产原料（实现基础的SDK接口、MSG）
（提供哪些SDK            └──────────────┘
  接口、事件）

─────────────────────────────────────────────────────────────────
自底向上开发（实现阶段）
先实现SDK接口 → 再实现Service功能/接口 → 最后UI处理交互的部分
```

------

### 2.2 以机盒模块为例的开发过程

以**机盒（casebox）**模块为例，展示三层结构（SDK AL、Service/Manager、UI）的设计与实现细节：

#### 设计阶段（自顶向下）

**UI 层：**

- 功能：处理 in/out/open/closed 相关动作，处理机盒控制功能，比如配对功能
- 需求：in, out, open, closed（消息），接收到机盒指令、比如配对的消息包，peer 配对（接口）

**Service/Manager 层：**

- 功能：生产 in/out/open/closed 相关消息，发送消息；从串口数据包中，拿到并解析正确的机盒 packet
- 需求：5V 状态（接口消息），串口通讯（接口/消息）

**SDK AL 层：**

- 需求：5V 状态(get/notify)，串口通讯(open/close/rx/tx)，蓝牙配对接口（接口），peer 配对接口（接口）
- 输出：5V状态(get/notify) → `sdk_hal_5v_detect.h`；串口通讯(open/close/rx/tx) → `sdk_hal_uartcase.h`；蓝牙配对/peer 配对 → `sdk_hal_bt.h`

#### 实现阶段（自底向上）

1. **先实现 SDK 接口**（`sdk_hal_uartcase.c` / `sdk_hal_5v_detect.c`）
2. **再实现 Service 功能\接口**（`vivo_casebox_service.c`）
3. **最后 UI 处理交互的部分**（`ui_casebox.c`）

------

### 2.3 实现 sdk_hal_uartcase.c 文件

**第一步：先写好 SDK 抽象层的 .h 及相关接口**（可以后续再完善，不需要一次声明所有接口）

先创建 `sdk_hal_uartcase.h`、`sdk_hal_5v_detect.h`，在 .h 中声明我们待提供的接口，然后添加到 `sdk_common.h` 中。

**sdk_hal_uartcase.h 接口示例：**

```c
#ifndef _SDK_HAL_UART_CASE_H
#define _SDK_HAL_UART_CASE_H

typedef enum {
    UART_BAUDRATE_9K6   = 0x00,  /*!< 9600 baud */
    UART_BAUDRATE_19K2  = 0x01,  /*!< 19200 baud */
    UART_BAUDRATE_38K4  = 0x02,  /*!< 38400 baud */
    UART_BAUDRATE_57K6  = 0x03,  /*!< 57600 baud */
    UART_BAUDRATE_115K2 = 0x04,  /*!< 115200 baud */
    UART_BAUDRATE_230K4 = 0x05,  /*!< 230400 baud */
    UART_BAUDRATE_460K8 = 0x06,  /*!< 460800 baud */
    UART_BAUDRATE_921K6 = 0x07,  /*!< 921600 baud */
    UART_BAUDRATE_1M382 = 0x08,  /*!< 1382400 baud */
    UART_BAUDRATE_2M    = 0x09,  /*!< 2000000 baud */
    UART_BAUDRATE_3M    = 0x0A,  /*!< 3000000 baud */
    UART_BAUDRATE_1500K = 0x0B,  /*!< 1500000 baud */
} uart_baudrate_t;

void sdk_hal_uart_case_open(void);
void sdk_hal_uart_case_close(void);
bool sdk_hal_uart_case_is_enable(void);
void sdk_hal_uart_case_keep_uart_open(bool is_keep_open);
void sdk_hal_uart_case_switch_baudrate(uart_baudrate_t sw_baud);
int  sdk_hal_uart_case_send(uint8 *p_data, uint16 len);
void sdk_hal_uart_case_add_task(Task add_task);
void sdk_hal_uart_case_init(void);

#endif
```

**sdk_hal_5v_detect.h 接口示例：**

```c
#ifndef _SDK_HAL_5V_DETECT_H
#define _SDK_HAL_5V_DETECT_H

#include "sdk_osal.h"

void sdk_hal_5v_detect_init(void);
void sdk_hal_5v_detect_add_task(Task add_task);
bool sdk_hal_5v_detect_is_plugin(void);

#endif
```

**sdk_common.h 中添加头文件：**

```c
#ifndef __SDK_COMMON_H
#define __SDK_COMMON_H

/* base include */
#include "sdk_hal_log.h"
#include "sdk_hal_malloc.h"
#include "sdk_osal.h"

/* sw-function include */
#include "sdk_hal_system.h"
#include "sdk_hal_bt.h"
#include "sdk_hal_bt_addr.h"
#include "sdk_hal_peer.h"
#include "sdk_hal_ble_adv.h"
#include "sdk_hal_gaia.h"
#include "sdk_peer_sync.h"

/* hw-function include */
#include "sdk_hal_pio.h"
#include "sdk_hal_charger.h"
#include "sdk_hal_ntc.h"
#include "sdk_hal_5v_detect.h"   // ← 添加头文件到公共h
#include "sdk_hal_battery.h"

/* bus include */
#include "sdk_hal_iic.h"
#include "sdk_hal_spi.h"
#include "sdk_hal_uartcase.h"    // ← 添加头文件到公共h

#endif // __SDK_COMMON_H
```

**第二步：实现 sdk_hal_uartcase.c**

首先将三层结构搭建好（`sdk_xxx.c` 是可以引用 ADK 的头文件）：

```c
#include <stream.h>
#include "sdk_osal.h"
#include "sdk_malloc.h"
#include "sdk_hal_log.h"
#include "sdk_hal_uartcase.h"
#include "sdk_hal_uartcase_driver.h"  // 头文件包含
#include "vivo_message_define.h"

#undef LOG_TAG
#define LOG_TAG "sdk_uartcase"

/************* 前者名变量和结构体 **************/
static TaskData uartCaseTask;
static task_list_t* uartCaseTaskList = NULL;  // 用于控制消息，发送出去（发送过程）

static void sdk_hal_uart_case_msg_handler(Task task, MessageId id, Message msg)
{
    UNUSED(task);
    switch(id) {
        default:
            break;
    }
}
// 解析私有 task 和 msg handler 函数

/************* 供外部调用的接口 **************/
#void sdk_hal_uart_case_open(void) ...
#void sdk_hal_uart_case_close(void) ...
#bool sdk_hal_uart_case_is_enable(void) ...
#void sdk_hal_uart_case_keep_uart_open(bool is_keep_open, ...) ...
#void sdk_hal_uart_case_switch_baudrate(uart_baudrate_t sw_baud) ...
#int  sdk_hal_uart_case_send(uint8 *p_data, uint16 len) ...
#void sdk_hal_uart_case_add_task(Task add_task) ...
```

**注册并监听 ADK 的消息（在这里可以转成我们自己的消息 ID 再往上报）：**

```c
switch(id) {
    case MESSAGE_MORE_DATA:
        sdk_hal_uart_case_process_more_data(msg);
        break;
    ...
}
```

**Service 层只需要 Add Task 到这个 taskList 并监听这个消息，就可以拿到串口数据：**

```c
// 执行完成后回到这里，根据遍历结果后的 droplen 去弃数据
```

------

### 2.4 实现 sdk_hal_5V_detect.c 文件

```c
#include "vivo_message_define.h"
#include "sdk_hal_log.h"
#include "sdk_osal.h"
#include "charger_monitor.h"        // 头文件包含
#include "sdk_hal_5v_detect.h"

#undef LOG_TAG
#define LOG_TAG "sdk_5v_detect"

static TaskData sdk_5v_detect_task;
static task_list_t* notifyTaskList = NULL;
static bool isPlugin = FALSE;

/*
 * @brief 将task接到系统的os tasklist中，方便统一广播
 * @param[in] task: local task
 * @param[in] id: 发送的Message ID
 * @param[in] message: 消息数据，任意类型（收到消息后需要强制类型转换）
 */
static void sdk_5v_detect_msg_handler(Task task, MessageId id, Message msg)
{
    UNUSED(task);
    UNUSED(msg);
    switch(id) {
        case CHARGER_MESSAGE_ATTACHED:
            LOGI("VIVO_MSG_5V_PLUGIN");
            isPlugin = TRUE;
            sdk_TaskList_MessageSendId(notifyTaskList, VIVO_MSG_5V_PLUGIN); // ← 转成内部消息广播
            break;
        case CHARGER_MESSAGE_DETACHED:
            LOGI("VIVO_MSG_5V_PLUGOUT");
            isPlugin = FALSE;
            sdk_TaskList_MessageSendId(notifyTaskList, VIVO_MSG_5V_PLUGOUT);
            break;
        default:
            break;
    }
}
// 这里收到SDK的消息，然后转换成我们内部的消息，发送出去
// 因为考虑到接收者有限，所以创建了本地的 taskList
// 这里实际也可以改成：vivo_msg_broadcast(VIVO_MSG_5V_PLUGIN)

bool sdk_hal_5v_detect_is_plugin(void) ...
void sdk_hal_5v_detect_add_task(Task add_task) ...
void sdk_hal_5v_detect_init(void)
{
    if(notifyTaskList == NULL) {
        notifyTaskList = sdk_TaskList_Create();
    }
    sdk_5v_detect_task.handler = sdk_5v_detect_msg_handler;
    Charger_ClientRegister(&sdk_5v_detect_task);  // 注册到SDK的消息
}
```

------

### 2.5 实现 vivo_casebox_service.c 文件

同样将三层结构搭建好，拿到 Msg 并作对应的处理：

```c
#include "sdk_common.h"
#include "vivo_framework_common.h"
#include "vivo_casebox_protocol.h"

#undef LOG_TAG
#define LOG_TAG "vivo_casebox_service"

static TaskData vivo_casebox_service_task;   // 获取task
static task_list_t* vivo_casebox_service_parser_notify_tasklist = NULL;

/*
 * @brief task接到系统os tasklist中
 * @param[in] task: local task
 * @param[in] id: 发送的Message ID
 * @param[in] message: 消息数据，任意类型（收到消息后需要强制类型转换）
 */
static void vivo_casebox_service_msg_handler(Task task, MessageId id, Message msg)
{
    UNUSED(task);
    UNUSED(msg);
    switch(id) {
        default:
            break;
    }
}
```

**对外接口：**

```c
#void vivo_casebox_service_keep_uart_open(bool is_keep_open) ...
#void vivo_casebox_service_switch_baudrate(uint8 sw_baud) ...
#void vivo_casebox_service_get_parser_result_add_task(Task add_task) ...
vivo_box_status_t vivo_casebox_service_get_curr_box_state(void) ...
#void vivo_casebox_service_tx_rsp_state(uint8_t type, uint8 cmd, uint8 state) ...
#bool vivo_casebox_service_tx_rsp_data(uint8_t type, uint8 cmd,
                                        const uint8 *payload, uint16 size_payload) ...
```

**初始化：**

```c
void vivo_casebox_service_init(void)
{
    if(vivo_casebox_service_parser_notify_tasklist == NULL) {
        vivo_casebox_service_parser_notify_tasklist = sdk_TaskList_Create();
    }
    vivo_casebox_service_task.handler = vivo_casebox_service_msg_handler;
    vivo_os_system_add_task(&vivo_casebox_service_task);  // ← 向SDK AL注册task
    sdk_hal_uart_case_add_task(&vivo_casebox_service_task);
}
```

**关键消息处理——5V in/out 消息消抖：**

```c
// 对 5V in/out 进行消抖处理
// 当关关盖，消抖结束
```

------

### 2.6 实现 vivo_casebox_manager.c 文件

> **注意：manager 是不带 3 层结构的**，应该只是把 service 的接口封装成 manager。然后添加到 `vivo_framework_common.h`

```c
#include "vivo_casebox_service.h"
#include "vivo_casebox_manager.h"

void vivo_casebox_manager_keep_uart_open(bool is_keep_open)
{
    vivo_casebox_service_keep_uart_open(is_keep_open);
}

void vivo_casebox_manager_switch_baudrate(uint8 sw_baud)
{
    vivo_casebox_service_switch_baudrate(sw_baud);
}

void vivo_casebox_manager_get_parser_result_add_task(Task add_task)
{
    vivo_casebox_service_get_parser_result_add_task(add_task);
}

vivo_box_status_t vivo_casebox_manager_get_curr_box_state(void)
{
    return vivo_casebox_service_get_curr_box_state();
}

bool vivo_casebox_manager_tx_rsp_state(uint8_t type, uint8 cmd, uint8 state)
{
    return vivo_casebox_service_tx_rsp_state(type, cmd, state);
}

bool vivo_casebox_manager_tx_rsp_data(uint8_t type, uint8 cmd,
                                       const uint8 *payload, uint16 size_payload)
{
    return vivo_casebox_service_tx_rsp_data(type, cmd, payload, size_payload);
}
```

**vivo_framework_common.h（统一的框架公共头文件）：**

```c
#ifndef __VIVO_FRAMEWORK_COMMON_H
#define __VIVO_FRAMEWORK_COMMON_H

/* base include */
#include "vivo_base.h"
#include "vivo_message_define.h"
#include "vivo_message_type_define.h"
#include "vivo_os_message.h"
#include "vivo_framework_init.h"

/* base-manager include */
#include "vivo_battery_manager.h"
#include "vivo_casebox_manager.h"     // ← 添加 casebox manager
#include "vivo_ble_manager.h"
#include "vivo_gaia_manager.h"

/* base-service include */
#include "vivo_battery_service.h"
#include "vivo_casebox_service.h"     // ← 添加 casebox service
#include "vivo_ble_service.h"
#include "vivo_gaia_service.h"

/* components include */
/* ext module manager include */
/* ext module service include */

#endif
```

------

### 2.7 实现 ui_casebox.c 文件

同样先创建 3 层结构，然后监听 Msg，并做相关处理：

```c
#include "sdk_common.h"
#include "vivo_framework_common.h"
#include "ui_casebox.h"
#include "ui_casebox_cmd.h"

#undef LOG_TAG
#define LOG_TAG "vivo_ui_casebox"

/* 静态变量、函数 */
static TaskData vivo_ui_casebox_task;

static void vivo_ui_casebox_msg_handler(Task task, MessageId id, Message msg)
{
    UNUSED(task);
    switch(id) {
        case VIVO_MSG_UARTCASE_RX_MORE_PACKET:   // 监听串口机盒解析成功的数据包
            VIVO_MSG_UARTCASE_RX_MORE_PACKET_T* rx_packet =
                (VIVO_MSG_UARTCASE_RX_MORE_PACKET_T*) msg;
            caseBoxCmd_Handle(rx_packet->mode, rx_packet->cmd,
                              rx_packet->payload, rx_packet->payloadsize);
            break;
        default:
            break;
    }
}

/* 初始化ui casebox */
void vivo_ui_casebox_init(void)
{
    vivo_ui_casebox_task.handler = vivo_ui_casebox_msg_handler;
    vivo_os_system_add_task(&vivo_ui_casebox_task);  // 向system注册task
    vivo_casebox_manager_get_parser_result_add_task(&vivo_ui_casebox_task); // 注册串口接收task
}
```

**caseBoxCmd_Handle 中的命令处理示例：**

```c
void caseBoxCmd_Handle(uint8 mode, uint8 cmd, const uint8 *payload, uint16 payload_len)
{
    if(cmd == CASEBOX_CMD_GET_INIT_STAT && mode == CASEBOX_CMD_GET_NORMAL_START) {
        LOGD_DATA_INFO(payload, payload_len, "msg");
        return;
    }
    if(mode == 1) { // SET mode
        return;
    }
    switch(cmd) {
        case CASEBOX_CMD_GET_INIT_STATE:
            caseBoxCmd_HandleGetInitBandState(cmd, payload, payload_len);
            break;
        case CASEBOX_CMD_SET_RIGHT_STATE:
            caseBoxCmd_HandleGetInitBandState(cmd, payload, payload_len);
            break;
        case CASEBOX_CMD_PARING_STATE:
            caseBoxCmd_HandleInterfacePairingState(cmd, payload, payload_len);
            break;
        case CASEBOX_CMD_OPEN_LID:
            caseBoxCmd_HandleOpenLid(cmd, payload, payload_len);
            break;
        case CASEBOX_CMD_CLOSE_LID:
            caseBoxCmd_HandleCloseLid(cmd, payload, payload_len);
            break;
        case CASEBOX_CMD_IN_OUT_IN_CAS:
            caseBoxCmd_HandlePutInCase(cmd, payload, payload_len);
            break;
        case CASEBOX_CMD_SET_REBOOT:
            caseBoxCmd_HandleGetReboot(cmd, payload, payload_len);
            break;
        case CASEBOX_CMD_TWS_PAIRING:
            caseBoxCmd_HandleTwsPairing(cmd, payload, payload_len);
            break;
        case CASEBOX_CMD_TWS_SHUTDOWN:
            caseBoxCmd_HandleShutdown(cmd, payload, payload_len);
            break;
        case CASEBOX_CMD_PAIRING_RECORD:
            caseBoxCmd_HandlePairRecord(cmd, payload, payload_len);
            break;
        default:
            break;
    }
}
```

------

## 三、开发规范

### 3.1 命名规范

各层的命名规则如下：

| 层级       | 命名格式                                                 | 说明                                    |
| ---------- | -------------------------------------------------------- | --------------------------------------- |
| SDK 抽象层 | `sdk_xxx`                                                | 例：`sdk_hal_5v_detect_is_plugin`       |
| Service 层 | `vivo_XXX(模块)_svc_xxx` 或 `vivo_XXX(模块)_service_xxx` | svc 是 service 的缩写                   |
| Manager 层 | `vivo_XXX(模块)_mgr_xxx` 或 `vivo_XXX(模块)_manager_xxx` | mgr 是 manager 的缩写                   |
| UI 层      | `vivo_ui_XXX(模块)_xxx`                                  | 只在本 .c 文件内的，也可以直接 `ui_XXX` |

**命名示例：**

```c
bool sdk_hal_5v_detect_is_plugin(void);
void vivo_casebox_service_switch_baudrate(uint8 sw_baud);
void vivo_casebox_manager_switch_baudrate(uint8 sw_baud);
void vivo_ui_casebox_msg_handler(Task task, MessageId id, Message msg);
```

------

### 3.2 消息 ID 规范

消息统一在 `base/vivo_message_define.h` 中定义，并标注类型：

```c
enum {
    ...
    VIVO_MSG_MODULE_XXXX_BASE = 200,
    VIVO_MSG_XXXX,    // public
    VIVO_MSG_ZZZZ,    // limited
    VIVO_MSG_YYYY,    // private
    VIVO_MSG_KKKK,    // sub Msg, see to vivo_sub_message_kkkk_define.h
}
```

**消息类型说明：**

| 类型      | 说明                                                         |
| --------- | ------------------------------------------------------------ |
| `public`  | 表示全局消息，即可以广播中 case 的消息                       |
| `limited` | 表示受限的消息，监听这个消息需要一定条件（可能需要单独 add task 来监听） |
| `private` | 表示私有消息，仅在某个文件或者模块中传递                     |
| `sub Msg` | 表示需要扩展为子消息，可以参考 `vivo_sub_message_kkkk_define.h` 文件。message 内容中第一个结构体成员必须为 `sub_msg_id`（考虑到 Peer 同步、产测、自动化或者 OTA，有多个 ID） |

------

### 3.3 修改权限规范

**SDK 修改权限：**

- 只有**系统、音频、连接**有修改权限，其他模块不可以修改，需要修改可以提需求。

**SDK 抽象层：**

- 由 **SE 来添加**，系统复核。模块开发者仅提出需求，不参与修改。

**Service/Manager/UI：**

- 由**模块开发者**来完成，模块 SE 审核。

------

### 3.4 SDK 修改宏管控

所有的 SDK 修改，都需要加宏，并且以 `"VIVO_TWS_ADAPT_XXXX"` 开头。

**示例宏名：**

```
VIVO_TWS_ADAPT_OS
VIVO_TWS_ADAPT_BOX_OPEN_CLOSED
VIVO_TWS_ADAPT_BATTERY_CAPTURE
```

所有的宏，都必须记录到 `SDK_AL/SDK_ADAPT/QCC/MARCO_DEFINE.txt` 中（其他平台替换 QCC），用 `,` 隔开，然后描述大概的修改内容。

**记录格式示例：**

```
VIVO_TWS_ADAPT_OS, vivo框架初始化
VIVO_TWS_ADAPT_BOX_OPEN_CLOSED, 开关盖event的添加
VIVO_TWS_ADAPT_BATTERY_CAPTURE, 电池电压采集的修改
```

------

### 3.5 消息发送 API 说明

**指定给某个 Task 发送消息：**（一般用于本地延时处理，比如出入盒消抖）

```c
void vivo_msg_send(Task task, MessageId id, void *message);
void vivo_msg_send_later(Task task, MessageId id, void *message, uint32 delay);
uint16 vivo_msg_cancel_all(Task task, MessageId id);
```

**给 Service、UI 层发消息：**（默认 Service、UI 的 task 都有注册到 system os tasklist 中）

```c
void vivo_msg_broadcast(MessageId id, void *message);
void vivo_msg_broadcast_later(MessageId id, void *message, uint32 delay);
uint16 vivo_msg_broadcast_cancel(MessageId id);
```

**立即执行某个消息：**（代替大量注册 callback）

```c
void sdk_Task_MessageExecuteImmediate(Task task, MessageId id, void *message);
void sdk_TaskList_MessageExecuteImmediate(task_list_t *list, MessageId id, void *message);
```

------

### 3.6 task 和 tasklist 声明规范

**task、tasklist 需要单独声明定义，不要包含到结构体里面，更不要直接开放给外部。**

**❌ 不好的写法：**

```c
typedef struct
{
    TaskData task;          // 不好：task 放在结构体里
    bool battery_voltage_change_flag;
    bool its_complete_init;
    bool battery_full_can_power_off;
    bool enter_ship_mode_flag;
} BatteryManagerAdpTask;

extern BatteryManagerAdpTask battery_manager_adp_data;  // 不好：直接暴露外部
#define GetBatteryManagerData()       (&battery_manager_adp_data)
#define GetBatteryManagerTask()       (&battery_manager_adp_data.task)
#define GetBatteryManagerClientTask() ((task_list_flexible_t *)(&battery_manager_adp_data.client_tasks))
```

**✅ 比较好的写法：**

```c
static TaskData vivo_ui_battery_task;  // static 声明，仅本文件可见

void vivo_ui_battery_init(void)
{
    vivo_ui_battery_task.handler = vivo_ui_battery_msg_handler;
    vivo_os_system_add_task(&vivo_ui_battery_task);  // 在自己的 .c 文件，向自己的模块注册
}
```

**规则 7：任何一个 task，一定是收到某个 Msg 才触发执行。**

下层比如 SDK AL、Service 想要触发上层 task 执行，应采用 **Msg 通知**的方式，而不是直接调用函数。

**规则 8：实际编写代码时存在耦合，如何处理？**

可以以以下优先级评估：

```
SDK平台抽象性 > 代码理解难易度 > Service的功能复用性 > 框架代码耦合性
```

------

### 3.7 其他开发规范

- **APP 之间**，尽量采用 Message 交互。
- **APP 可以直接调用** Manager、SDK AL 的接口。
- **Service 不能调用** APP 层暴露的 API 接口。
- **APP 层暴露的 API 接口**，其他 APP 不允许直接 include。应当放到 `ui_exchange.h` 中，包含需要暴露的头文件，然后其他 APP 再 `include "ui_exchange.h"` 头文件。
- **如果存在耦合**，比如 ui OTA 要获取 peer 电量，peer 电量这一部分的处理就应该下放到 Service 层。
- **变量和函数一定明确作用域**，仅本地使用的一定要使用 `static` 声明。
- **禁止使用 `extern`**。
- **manager 是没有 task 的**，只是接口封装。
- **不是所有模块**在 vivo framework 中都需要加宏，基本的电量、出入盒、BT 连接、GAIA 等是不需要的。但是佩戴、汇顶 Sensor、耳温等扩展功能就需要加宏。

------

## 附录：关键术语对照表

| 术语                    | 含义                                                 |
| ----------------------- | ---------------------------------------------------- |
| TWS                     | True Wireless Stereo，真无线立体声耳机               |
| ADK                     | Audio Development Kit，QCC 平台的音频开发套件        |
| SDK AL                  | SDK Abstraction Layer，SDK 抽象层                    |
| task                    | 等效于各模块的函数指针，消息处理的最小单元           |
| tasklist                | 保存多个 task 的链表，用于广播消息给所有注册者       |
| IPC                     | Inter-Processor Communication，处理器间通信          |
| SE                      | System Engineer，系统工程师                          |
| vivo_msg_broadcast      | 广播消息给所有注册到 system os tasklist 的 task      |
| MARCO_DEFINE.txt        | SDK 修改宏的统一记录文件                             |
| P0/P1                   | QCC 平台的两个处理器核心                             |
| vivo_framework_common.h | 框架公共头文件，包含所有 manager 和 service 的头文件 |
| sdk_common.h            | SDK AL 公共头文件，包含所有 SDK 抽象接口             |