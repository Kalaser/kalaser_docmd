---
title: 嵌入式平台系统框架开发通用指南
date: 2026-10-27
description: 适用于智能手表、工控终端、IoT 网关等设备的嵌入式系统框架设计原则、四层架构与开发流程。
tags: [嵌入式, 架构设计, 框架]
---

# 嵌入式平台系统框架开发通用指南

> 本文提炼自 TWS 耳机应用框架的核心设计思想，去除 TWS 领域特异性，保留适用于**所有嵌入式设备开发**的通用原则、模式和流程。无论你是在做智能手表、工控终端、IoT 网关还是汽车仪表，这些原则同样适用。

---

## 目录

1. [核心设计原则](#嵌入式平台系统框架开发通用指南-一核心设计原则)
2. [推荐架构：四层模型](#嵌入式平台系统框架开发通用指南-二推荐架构四层模型)
3. [消息驱动通信](#嵌入式平台系统框架开发通用指南-三消息驱动通信)
4. [平台抽象层设计](#嵌入式平台系统框架开发通用指南-四平台抽象层设计)
5. [模块标准骨架](#嵌入式平台系统框架开发通用指南-五模块标准骨架)
6. [开发流程](#嵌入式平台系统框架开发通用指南-六开发流程)
7. [管控与治理](#嵌入式平台系统框架开发通用指南-七管控与治理)
8. [常见陷阱与纠正](#嵌入式平台系统框架开发通用指南-八常见陷阱与纠正)
9. [检查清单](#嵌入式平台系统框架开发通用指南-九检查清单)
10. [术语速查](#嵌入式平台系统框架开发通用指南-十术语速查)

---

## 一、核心设计原则

嵌入式系统框架设计，归根结底只解决一个问题：**在硬件平台多变、团队人员有限、维护周期漫长的约束下，如何让代码保持可理解和可修改。**

以下六条原则是后续所有模式和规范的根基：

### 原则 1：依赖方向永远向下

```
App（UI） → Framework（Manager/Service） → Platform Abstraction → SDK/Hardware
```

- **上层可以调用下层，下层绝不能调用上层**
- 违反这条规则会导致循环依赖——第一周还看不出来，第三个月开始每次改代码都要跳来跳去

### 原则 2：模块间通信走消息，不走函数调用

- 模块 A 不需要知道模块 B 的函数签名、头文件路径、调用时机
- 模块 A 只需要知道一个**消息 ID**
- 这是整个解耦体系的基石，后面会展开

### 原则 3：抽象不稳定的，保留稳定的

- **芯片 SDK API** 随平台变化 → 必须抽象
- **业务逻辑** 相对稳定 → 不需要过度抽象
- 抽象是有成本的（多一层间接调用），只在"变化点"上做抽象

### 原则 4：编译期强制优于文档约定

- `static` 变量、禁止 `extern`、单向 include
- 文档说"请不要直接调用"的力量远不如编译器报错 `undefined reference`

### 原则 5：单线程事件循环优先

- 除非有明确的并行计算需求，默认使用单线程事件循环
- 线程越多，竞态条件越多，调试难度指数上升
- 耗时任务拆成状态机 + 消息驱动

### 原则 6：命名即文档

- 函数名前缀表明它属于哪一层（`hal_`、`svc_`、`ui_`）
- 不需要看目录结构就能知道一个函数的层级归属
- 这在代码审查时能节省大量心智负担

---

## 二、推荐架构：四层模型

### 2.1 架构图

```
┌──────────────────────────────────────────────────────┐
│                 App / UI 层                           │
│  用户交互 · 界面展示 · 业务编排                        │
│  只依赖：Manager 接口 + 消息 ID                        │
└──────────────────────────────────────────────────────┘
                        ↓ 调用 Manager API / 订阅消息
┌──────────────────────────────────────────────────────┐
│               Framework 层                            │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │   Base      │  │   Manager    │  │  Service   │  │
│  │ OS抽象/消息  │  │ 接口封装(薄) │  │ 功能实现   │  │
│  │ 公共数据结构 │  │ 无线程/状态  │  │ 持有线程   │  │
│  └─────────────┘  └──────────────┘  └────────────┘  │
└──────────────────────────────────────────────────────┘
                        ↓ 调用 hal_ / osal_ 统一接口
┌──────────────────────────────────────────────────────┐
│          Platform Abstraction Layer（平台抽象层）     │
│  HAL（硬件抽象）· OSAL（OS抽象）· 协议栈抽象          │
│  头文件各平台完全相同，实现按平台替换                   │
└──────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────┐
│              芯片 SDK / BSP / 硬件                    │
│  厂商提供的 SDK，对框架而言是黑盒                      │
└──────────────────────────────────────────────────────┘
```

### 2.2 各层职责与约束

| 层 | 职责 | 可以调用 | 禁止调用 |
|---|---|---|---|
| **App / UI** | 用户交互，界面渲染，业务流程串联 | Manager 接口，消息订阅 API | 不能直接调 Service、不能直接调 HAL |
| **Manager** | 对 Service 接口的命名透传 | Service 内部函数 | 不允许持有线程，不允许包含业务逻辑，不允许直接调 HAL |
| **Service** | 核心业务逻辑，状态机，数据处理，持有线程/task | HAL 接口（通过统一头文件） | 不能调用 App 层任何接口 |
| **Platform AL** | 硬件/OS 接口统一封装 | 芯片 SDK / BSP | 不能包含业务逻辑 |
| **SDK / BSP** | 芯片厂商提供 | 硬件寄存器 | 视为只读黑盒 |

### 2.3 Manager 为什么必须是"空壳"

一个常见疑问：Manager 只是透传调用 Service，为什么不能省略？

**原因**：Manager 隔离了 App 层对 Service 实现的直接依赖。

```
无 Manager： App → include service.h → 直接调用 service_xxx()
有 Manager： App → include manager.h → 调用 manager_xxx() → Service 内部实现
```

当 Service 重构（拆分为两个模块、更换算法、改函数签名）时：
- 有 Manager：只改 Manager 内部的委托逻辑，App 不受影响
- 无 Manager：每个引用了 Service 的 App 都得改

**判断标准**：如果你的项目只有 1 个 App 模块且永远不会有第二个，可以省略 Manager。否则保留。

---

## 三、消息驱动通信

### 3.1 为什么不用直接函数调用

| 维度 | 直接函数调用 | 消息驱动 |
|---|---|---|
| 编译依赖 | 需要 include 对方头文件 | 只需共享消息 ID 枚举 |
| 调用时机 | 同步，调用者阻塞 | 异步，发送者立即返回 |
| 多对多通信 | 调用者持有所有被调用者引用 | 一条广播，所有订阅者自动接收 |
| 模块替换 | 新模块必须实现相同的函数签名 | 新模块只需发布相同的消息 ID |
| 错误隔离 | 被调用者崩溃 → 调用者也崩溃 | 被调用者崩溃 → 消息丢失，不影响发送者 |
| 可追踪性 | 需要抓调用栈 | grep 消息 ID 即可找到所有生产者和消费者 |
| 可测试性 | 需要 mock 被调用模块 | 直接向队列注入测试消息 |

### 3.2 核心机制：Task 与 TaskList

**Task** = 一个消息处理函数指针。对应"一个模块的入口"。

```c
// 每个模块定义自己的 task —— 必须是 static
static TaskData sensor_service_task;

static void sensor_service_handler(Task task, MessageId id, void *msg)
{
    switch (id) {
        case MSG_SENSOR_READ_REQUEST:
            // 读取传感器，发布结果
            break;
        default:
            break;
    }
}
```

**TaskList** = 一组 Task 的集合。用于"广播给所有订阅者"。

```c
// 模块 A 维护一个订阅者列表
static TaskList *notify_list;  // 谁想收我的消息，就把 task 加进来

// 对外暴露订阅接口
void module_a_add_notify(Task t) {
    TaskList_Add(notify_list, t);
}

// 有数据时广播
void on_data_ready(int value) {
    SensorMsg msg = { .id = MSG_SENSOR_DATA, .value = value };
    TaskList_Broadcast(notify_list, &msg);  // 所有订阅者都收到
}
```

### 3.3 消息流转完整示例

```
硬件中断（ADC 采集完成）
  │
  ↓ 发布 MSG_ADC_RAW_DATA
[Platform AL 的 notify_tasklist]
  │
  ↓ 分发给订阅者
Sensor Service
  │ 执行数据滤波、单位换算
  ↓ 发布 MSG_SENSOR_VALUE_READY
[Sensor Service 的 notify_tasklist]
  ├──→ Display App（更新屏幕数值）
  ├──→ Logger App（写入存储）
  └──→ Alarm App（检查是否超阈值）
```

**关键理解**：Display App 不知道 Sensor Service 的存在，它只知道"我订阅了 MSG_SENSOR_VALUE_READY"。Sensor Service 不知道谁来消费数据。两者之间唯一的耦合点是**消息 ID**。

### 3.4 异步 vs 同步消息

| | 异步广播 | 同步执行 |
|---|---|---|
| 使用场景 | 状态变更、事件通知、周期性数据 | 需要即时返回值、需要收集多方结果 |
| API | `TaskList_Broadcast(list, id, msg)` | `TaskList_ExecuteImmediate(list, id, msg)` |
| 执行时机 | 消息入队，下次调度时处理 | 立即遍历所有 handler，全部处理完才返回 |
| 典型用例 | 传感器数据上报、电量变化、连接状态 | UART 数据解析（需要确定谁来消费数据） |

**选择原则**：默认用异步。只有当你需要"我现在就要答案，没有答案我下一步做不了"时才用同步。

### 3.5 消息 ID 是模块间的"唯一接口"

这是整个架构中最核心的概念。传统模块间通信需要知道：
- 对方的头文件路径
- 函数名和参数列表
- 返回值含义
- 调用线程限制

消息驱动下只需要知道：**一个消息 ID 枚举值**。

这意味着：
- 模块内部实现可以任意重构——只要生产/消费的消息 ID 不变，其他模块完全不受影响
- 新人只需要知道"我订阅哪些消息"和"我发布哪些消息"就能开始写代码
- 系统行为可以用消息序列来描述和追踪

---

## 四、平台抽象层设计

### 4.1 要抽象什么

不是所有东西都需要抽象。抽象有成本（多一层调用、更长的代码路径），只在以下场景做：

| 变化源 | 抽象方式 | 示例 |
|---|---|---|
| 芯片外设（GPIO/I2C/SPI） | `hal_gpio_write(pin, level)` 代替直接操作寄存器 | 换芯片只改 HAL 实现 |
| RTOS API | `osal_mutex_create()` 代替 `xSemaphoreCreateMutex()` | 从 FreeRTOS 切到 Zephyr |
| 通信协议栈 | `bt_send_data()` 代替芯片 SDK 的蓝牙 API | 换蓝牙芯片 |
| 文件系统 | `fs_write()` 代替 `fatfs_write()` | 换存储方案 |

**不需要抽象的场景**：
- 标准 C 库函数（`memcpy`、`sprintf`）
- 算法逻辑（滤波、校验、编码）
- 业务状态机

### 4.2 抽象层的组织方式

```
platform_al/
├── pal_common.h          ← 统一入口头文件（上层只 include 这一个）
├── include/
│   ├── pal_gpio.h        ← 接口定义（各平台相同）
│   ├── pal_i2c.h
│   ├── pal_uart.h
│   └── pal_osal.h
├── nrf52840/             ← 平台 A 实现
│   ├── pal_gpio.c
│   ├── pal_i2c.c
│   └── pal_osal.c        ← 封装 FreeRTOS API
├── stm32h7/              ← 平台 B 实现
│   ├── pal_gpio.c
│   ├── pal_i2c.c
│   └── pal_osal.c        ← 封装 Zephyr API
└── MACRO_CHANGELOG.txt   ← 对 SDK 的修改记录
```

**关键规则**：
- 头文件名、函数名、参数列表各平台**完全相同**
- 某平台不支持某功能 → 函数体留空返回，**不删函数声明**（否则上层代码编译不过）
- Makefile/CMake 用宏选择编译哪个平台的 `.c` 文件

### 4.3 抽象层 API 命名约定

```c
// 统一前缀表明层级归属
pal_gpio_init(...)         // PAL = Platform Abstraction Layer
pal_gpio_write(...)
pal_uart_open(...)
pal_osal_mutex_create(...)
```

命名前缀本身就是文档：看到 `pal_` 就知道这是平台抽象层的接口，换平台时这里要改。

---

## 五、模块标准骨架

### 5.1 单个模块的标准结构

每个业务模块（无论是 Service 还是 UI）遵循同一套骨架：

```c
// ============================================================
//  模块名称：xxx_service
//  层级：Service
//  职责：xxx 数据采集与处理
// ============================================================

// ── 1. 私有变量（全部 static，禁止 extern） ──
static TaskData       xxx_task;               // 本模块的 task handler
static TaskList      *xxx_notify_list = NULL;  // 订阅者列表
static int            xxx_internal_state;      // 内部状态

// ── 2. 消息处理函数 ──
static void xxx_msg_handler(Task task, MessageId id, void *msg)
{
    switch (id) {
        case MSG_XXX_REQUEST:
            // 处理逻辑
            // ...
            // 必要时广播结果
            TaskList_Broadcast(xxx_notify_list, &result_msg);
            break;

        case MSG_XXX_TIMER:    // 定时消息（由 send_later 产生）
            // 周期性任务
            break;

        default:
            break;
    }
    // 注意：此函数中禁止 sleep / delay / 长时间循环
}

// ── 3. 对外接口 ──
void xxx_service_add_notify(Task t)
{
    TaskList_Add(xxx_notify_list, t);
}

void xxx_service_do_something(int param)
{
    // 同步接口（仅供 Manager 调用）
}

// ── 4. 初始化 ──
void xxx_service_init(void)
{
    xxx_notify_list = TaskList_Create();
    xxx_task.handler = xxx_msg_handler;
    System_AddTask(&xxx_task);           // 注册到系统消息总线
    pal_xxx_subscribe(&xxx_task);        // 订阅底层硬件消息
}
```

### 5.2 Manager 的标准骨架

```c
// ============================================================
//  模块名称：xxx_manager
//  层级：Manager
//  职责：对 xxx_service 的接口透传（无线程，无状态，无逻辑）
// ============================================================

// Manager 只做一件事：把调用委托给 Service
// 不需要 task，不需要 init（或者 init 只是调 service 的 init）

void xxx_manager_add_notify(Task t)
{
    xxx_service_add_notify(t);   // 纯透传
}

void xxx_manager_do_something(int param)
{
    xxx_service_do_something(param);   // 纯透传
}
```

### 5.3 App / UI 模块的标准骨架

```c
// ============================================================
//  模块名称：display_app
//  层级：App
//  职责：在屏幕上显示传感器数据
// ============================================================

static TaskData  display_task;
static MessageQueue display_queue;  // 接收消息的队列

static void display_msg_handler(Task task, MessageId id, void *msg)
{
    switch (id) {
        case MSG_SENSOR_VALUE_READY:
            SensorMsg *data = (SensorMsg *)msg;
            update_display(data->value);
            break;
        default:
            break;
    }
}

void display_app_init(void)
{
    display_queue = MessageQueue_Create(16, sizeof(SensorMsg));
    display_task.handler = display_msg_handler;
    System_AddTask(&display_task);
    sensor_manager_add_notify(&display_task);  // 通过 Manager 订阅
}
```

---

## 六、开发流程

### 6.1 设计阶段：自顶向下

**永远从 UI / 用户需求开始推导**，而不是从硬件开始。

| 步骤 | 做什么 | 输出物 |
|---|---|---|
| 1. 定义用户场景 | 用户看到什么、做什么操作 | UI 交互流程 |
| 2. 推导消息流 | UI 需要接收哪些消息？需要发送哪些消息？ | 消息 ID 列表 |
| 3. 推导 Service | 谁来生产这些消息？需要什么输入？ | Service 职责定义 |
| 4. 推导 Platform AL | Service 需要调用哪些硬件/OS 接口？ | PAL 接口定义 |

**示例**（以温度显示为例）：

```
用户场景：屏幕上显示当前温度，每 1 秒更新一次
  ↓
UI 需要什么消息：MSG_TEMPERATURE_DATA (value=250 表示 25.0°C)
  ↓
谁来生产：temperature_service（从温度传感器读取，转换单位，发布消息）
  ↓
需要什么硬件接口：pal_i2c_read(addr, reg, buf, len)
```

### 6.2 实现阶段：自底向上

设计的顺序是从上到下，但**实现的顺序是从下到上**。

1. **先实现 Platform AL**：硬件能跑通，接口验证正确
2. **再实现 Service**：在 PAL 基础上写业务逻辑，用单元测试或模拟数据验证
3. **再实现 Manager**：对 Service 接口做透传封装（通常只需几分钟）
4. **最后实现 UI**：订阅消息，渲染界面

这样每一步都有一个可以验证的基础，不会到最后才发现硬件驱动有问题。

### 6.3 注册到公共头文件

每新增一个模块，必须在以下位置注册：

```c
// 1. platform_al/pal_common.h — 新增的 PAL 头文件
#include "pal_temperature.h"

// 2. framework_common.h — 新增的 Manager + Service 头文件
#include "temperature_manager.h"
#include "temperature_service.h"

// 3. message_ids.h — 新增的消息 ID
MSG_TEMPERATURE_DATA,    // public  — 温度数据广播
MSG_TEMPERATURE_REQUEST, // private — 内部触发采集
```

---

## 七、管控与治理

### 7.1 可见性控制

| 手段 | 效果 |
|---|---|
| `static TaskData task` | 外部无法直接获得 task 指针，不能绕过消息系统直接发消息 |
| `static` 内部函数 | 模块内部实现细节不暴露，防止被意外依赖 |
| 禁止 `extern` 全局变量 | 切断模块间通过共享内存通信的捷径 |
| 统一入口头文件 | `pal_common.h` 是访问 PAL 的唯一合法入口；`framework_common.h` 是访问 Framework 的唯一入口 |
| App 间通信走中转 | 不直接 include 其他 App 的头文件，通过消息或 `app_exchange.h` 中转 |

### 7.2 命名规范

| 层级 | 前缀 | 示例 |
|---|---|---|
| 平台抽象层 | `pal_` | `pal_i2c_read()`、`pal_gpio_toggle()` |
| Service 层 | `xxx_svc_` 或 `xxx_service_` | `sensor_svc_read()`、`battery_service_get_level()` |
| Manager 层 | `xxx_mgr_` 或 `xxx_manager_` | `sensor_mgr_add_notify()` |
| UI / App 层 | `ui_xxx_` 或 `app_xxx_` | `ui_dashboard_init()` |
| 消息 ID | `MSG_` 前缀 | `MSG_TEMPERATURE_DATA` |

### 7.3 消息 ID 的可见性注释

```c
enum MessageId {
    MSG_BASE_SENSOR = 100,

    MSG_TEMPERATURE_DATA,   // public  — 任何模块可订阅
    MSG_HUMIDITY_DATA,      // public
    MSG_TEMP_THRESHOLD,     // limited — 仅 Alarm 模块可订阅
    MSG_SENSOR_POLL,        // private — 仅 Sensor Service 内部使用
};
```

### 7.4 SDK 修改记录

每一次对芯片 SDK 的修改（补丁、workaround、适配宏），必须：

1. 用宏包裹，宏名以 `ADAPT_` 开头
2. 记录到 `platform_al/MACRO_CHANGELOG.txt`

```
ADAPT_OS_INIT      — 系统初始化流程增加 PAL 层初始化
ADAPT_GPIO_WORKAROUND — GPIO 中断响应延迟 workaround
ADAPT_UART_FIFO    — 增大 UART FIFO 缓冲区
```

效果：当 SDK 升级时，grep `ADAPT_` 就能找到所有需要重新评估的修改点。

### 7.5 代码审查检查点

```
□ 新增的 task 是否声明为 static？
□ 是否在 Service 中直接调用了 App/UI 的函数？
□ 是否通过 extern 共享了全局变量？
□ Manager 是否持有线程或状态？
□ 消息处理函数中是否有 sleep/delay/长时间循环？
□ 新消息 ID 是否加了可见性注释？
□ 新增的 PAL 接口是否加入了 pal_common.h？
□ SDK 修改是否加了 ADAPT_ 宏并在 CHANGELOG 中记录？
```

---

## 八、常见陷阱与纠正

### 陷阱 1：在消息处理函数中 sleep

```c
// ❌ 错误
static void handler(Task t, MessageId id, void *msg) {
    start_adc_conversion();
    sleep(100);             // 整个系统卡住 100ms！
    read_adc_result();
}

// ✅ 正确：拆成状态机
static void handler(Task t, MessageId id, void *msg) {
    switch (id) {
        case MSG_START_SAMPLING:
            start_adc_conversion();
            send_later(t, MSG_READ_RESULT, NULL, 100);  // 100ms 后通知自己
            break;
        case MSG_READ_RESULT:
            read_adc_result();
            break;
    }
}
```

### 陷阱 2：下层调用上层

```c
// ❌ 错误：Service 直接调 UI
static void sensor_handler(...) {
    int temp = read_sensor();
    ui_display_update(temp);  // Service 依赖了 UI 层头文件
}

// ✅ 正确：Service 发布消息，UI 自己订阅
static void sensor_handler(...) {
    int temp = read_sensor();
    SensorMsg msg = { .id = MSG_TEMPERATURE_DATA, .value = temp };
    TaskList_Broadcast(notify_list, &msg);  // 不管谁来消费
}
```

### 陷阱 3：跳过 PAL 直接调 SDK

```c
// ❌ 错误：直接调芯片 SDK API
#include "nrf_gpio.h"
nrf_gpio_pin_set(LED_PIN);

// ✅ 正确：通过 PAL
#include "pal_common.h"
pal_gpio_write(LED_PIN, 1);
```

效果：换芯片时，❌ 需要改所有地方，✅ 只需要改 `pal_gpio.c` 一个文件。

### 陷阱 4：Manager 越权

```c
// ❌ 错误：Manager 包含逻辑
void sensor_manager_init(void) {
    k_thread_create(...);  // Manager 不应该持有线程
    threshold = 30;        // Manager 不应该持有状态
}

// ✅ 正确：Manager 只是透传
void sensor_manager_init(void) {
    sensor_service_init();  // 委托给 Service
}
```

### 陷阱 5：模块间用 extern 共享变量

```c
// ❌ 错误
// file_a.c:  int g_temperature = 0;
// file_b.c:  extern int g_temperature;  g_temperature = 25;

// ✅ 正确：通过消息传递
// file_a.c:  发布 MSG_TEMPERATURE_DATA，携带温度值
// file_b.c:  订阅 MSG_TEMPERATURE_DATA，从消息中获取
```

### 陷阱 6：修改 SDK 不记录

改了芯片 SDK 的一行代码，没加宏也没记录。三个月后 SDK 升级，这行被覆盖了，bug 重现，谁也记不起来改了什么。

**永远加上 `ADAPT_XXX` 宏 + CHANGELOG 一行**。这五分钟的投入在 SDK 升级时能节省五天。

---

## 九、检查清单

### 新项目启动时

- [ ] 确定需要支持的硬件平台数量（1 个 or 多个）
- [ ] 如果 ≥2 个平台，必须引入 Platform AL
- [ ] 确认 RTOS 选型及其抽象接口
- [ ] 建立目录结构（platform_al/、framework/、app/）
- [ ] 创建 `pal_common.h` 和 `framework_common.h` 两个统一入口
- [ ] 定义第一批消息 ID（从用户可见功能倒推）
- [ ] 确定命名规范并写入项目 README

### 新增模块时

- [ ] 自顶向下设计：从 UI 需求推导消息流
- [ ] 自底向上实现：PAL → Service → Manager → UI
- [ ] task 声明为 `static`
- [ ] Manager 不持有线程/状态
- [ ] 消息 ID 加可见性注释
- [ ] 新增头文件注册到 `pal_common.h` 或 `framework_common.h`

### 代码审查时

- [ ] 依赖方向是否单向向下
- [ ] 是否存在 `extern` 全局变量
- [ ] 消息处理函数是否无阻塞
- [ ] 是否直接调用了芯片 SDK（应走 PAL）
- [ ] SDK 修改是否加宏并记录

### SDK 升级时

- [ ] grep `ADAPT_` 找到所有修改点
- [ ] 逐一评估每个修改在新 SDK 中是否还需要
- [ ] 更新 `MACRO_CHANGELOG.txt`

---

## 十、术语速查

| 术语 | 含义 |
|---|---|
| **PAL** | Platform Abstraction Layer，平台抽象层。对芯片外设和 OS API 的统一封装 |
| **HAL** | Hardware Abstraction Layer，硬件抽象层。PAL 的子集，针对芯片外设 |
| **OSAL** | OS Abstraction Layer，操作系统抽象层。PAL 的子集，针对 RTOS API |
| **Task** | 消息处理函数指针。模块接收消息的入口。必须是 `static` |
| **TaskList** | 一组 Task 的集合。用于广播消息给所有订阅者 |
| **Manager** | Service 接口的命名透传层。无线程、无状态、无逻辑 |
| **Service** | 业务逻辑的实现层。持有线程/task，持有内部状态 |
| **App / UI** | 最终用户交互层。消费消息，渲染界面，处理用户输入 |
| **公共入口头文件** | `pal_common.h` 和 `framework_common.h`。上层代码只 include 这两个文件 |
| **消息 ID** | 模块间通信的唯一接口。一个枚举值 |
| **ADAPT 宏** | 对芯片 SDK 修改的标记宏。便于追踪和 diff |
| **自顶向下设计** | 从用户可见功能出发，倒推所需的消息和接口 |
| **自底向上实现** | 从硬件驱动开始，逐层向上构建，每层验证后再做上层 |
