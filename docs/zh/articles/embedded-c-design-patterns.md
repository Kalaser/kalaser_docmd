---
title: 嵌入式 C 语言常用设计模式
date: 2026-10-27
description: 用结构体与函数指针在 C 语言中实现状态机、观察者、策略等 11 种常用设计模式。
tags: [C语言, 设计模式, 嵌入式]
---

# 嵌入式C语言常用设计模式总结

> 核心思想：C语言没有class/虚函数，但**结构体 + 函数指针**这两样东西组合起来，就能把面向对象语言里的大部分设计模式在C里原样实现出来。本文档收录嵌入式开发中最常用的11种模式，每种给出代码示例、适用场景、优点，以及容易被忽略的**坑**。

使用设计模式的目的不是"显得高级"，而是在需求变化、团队协作、长期维护时，减少改动范围、降低出错概率。**如果一个函数超过200行，或者main.c超过500行，就该考虑引入合适的模式做拆分了。**

---

## 一、状态机模式（State Machine）

**核心思想**：把系统状态用枚举表示，通过`switch`管理状态转换，避免复杂的`if-else`嵌套。

**适用场景**：充电管理、通信连接状态、按键状态（短按/长按/双击）等多状态切换。

```c
typedef enum {
    CHARGE_UNPLUGGED,
    CHARGE_CHARGING,
    CHARGE_FULL,
    CHARGE_FAULT
} ChargeState_t;

typedef struct {
    ChargeState_t state;
    uint16_t      battery_mv;
    uint8_t       fault_code;
} ChargeSM_t;

void ChargeSM_Update(ChargeSM_t *sm)
{
    switch (sm->state) {
        case CHARGE_UNPLUGGED:
            if (Is_Charger_Plugged()) {
                sm->state = CHARGE_CHARGING;
                LED_Set(LED_YELLOW);
            }
            break;

        case CHARGE_CHARGING:
            if (sm->battery_mv >= BATTERY_FULL_MV) {
                sm->state = CHARGE_FULL;
                LED_Set(LED_GREEN);
            } else if (Is_Fault_Detected()) {
                sm->state = CHARGE_FAULT;
                sm->fault_code = Get_Fault_Code();
                LED_Set(LED_RED);
            }
            break;

        case CHARGE_FULL:
            if (!Is_Charger_Plugged()) {
                sm->state = CHARGE_UNPLUGGED;
                LED_Set(LED_OFF);
            }
            break;

        case CHARGE_FAULT:
            if (!Is_Charger_Plugged()) {
                sm->state = CHARGE_UNPLUGGED;
                sm->fault_code = 0;
            }
            break;

        default:
            /* 未知状态兜底：记录异常并回到安全状态，而不是静默忽略 */
            LOG_ERROR("Unknown charge state: %d", sm->state);
            sm->state = CHARGE_FAULT;
            break;
    }
}
```

**优点**：每个状态逻辑独立，新增状态不影响其他状态，问题定位清晰。

**坑：**

- 原始写法漏了`default`分支——`switch`语句没有`default`兜底，一旦`state`因为内存踩踏、枚举扩展遗漏等原因出现非法值，函数会静默不做任何事，问题很难定位。补上`default`是硬性要求（`coding_standard.md`规则11.5：switch语句必须有default分支）。
- 状态越多，单个`switch`函数越长，超过一定规模（比如状态数>8、每个case逻辑>10行）时，考虑`references/loop_branch_techniques.md`里"while+switch状态机"那一节介绍的"每个状态一个独立处理函数、返回下一状态"的写法，把状态处理逻辑拆到独立函数里，`ChargeSM_Update`本身只做分发。
- 状态转换点建议加日志（`LOG_INFO("state: %d -> %d", old, new)`），量产设备排查"为什么卡在某状态"全靠这行日志。
- 多任务环境下，如果`sm->state`会被多个任务/中断读写，要考虑加锁或者用原子操作，而不是假设单线程访问（参考`coding_standard.md`第18章并发编程规范）。

---

## 二、观察者模式（Observer）

**核心思想**：事件产生者与处理者解耦，通过注册回调函数实现多个模块响应同一事件。

**适用场景**：按键事件、传感器报警等需要多个模块同时响应的场景。

```c
typedef enum {
    EVENT_BTN_PRESS,
    EVENT_SENSOR_ALARM,
    EVENT_COMM_CONNECTED,
    EVENT_MAX
} EventType_t;

typedef void (*EventCallback_t)(void *arg);

#define MAX_OBSERVERS 4
static EventCallback_t observers[EVENT_MAX][MAX_OBSERVERS];

/* 返回值：0成功，-1表示观察者槽位已满（调用者必须检查！） */
int Event_Register(EventType_t event, EventCallback_t cb)
{
    if (event >= EVENT_MAX || cb == NULL) {
        return -1;
    }
    for (int i = 0; i < MAX_OBSERVERS; i++) {
        if (observers[event][i] == NULL) {
            observers[event][i] = cb;
            return 0;
        }
    }
    LOG_ERROR("Event %d observer slots full", event);
    return -1;   /* 满了，注册失败——调用者必须检查这个返回值 */
}

void Event_Notify(EventType_t event, void *arg)
{
    if (event >= EVENT_MAX) {
        return;
    }
    for (int i = 0; i < MAX_OBSERVERS; i++) {
        if (observers[event][i]) {
            observers[event][i](arg);
        }
    }
}

/* 各模块独立注册，注意检查返回值 */
if (Event_Register(EVENT_BTN_PRESS, Display_OnButtonPress) != 0) {
    LOG_ERROR("Register Display_OnButtonPress failed");
}
Event_Register(EVENT_BTN_PRESS, Buzzer_OnButtonPress);
Event_Register(EVENT_SENSOR_ALARM, Comm_SendAlarm);

/* 事件源只负责发通知 */
void Button_Task(void)
{
    if (Is_Button_Pressed()) {
        Event_Notify(EVENT_BTN_PRESS, NULL);
    }
}
```

**优点**：新增响应模块只需注册回调，无需修改现有代码。

**坑：**

- 原始写法的`Event_Register`在槽位满时**静默返回**，调用者完全不知道注册失败——这违反了`coding_standard.md`规则3.6"函数返回值必须校验"。改成返回`int`错误码后，调用方必须检查，否则某个模块的回调"注册了但其实没生效"这种bug会非常难排查。
- 没有提供`Event_Unregister`：模块销毁/去初始化时如果不注销回调，函数指针可能变成悬空指针，事件触发时调用到已经无效的代码——如果观察者的生命周期短于事件系统本身，必须补一个注销接口。
- `Event_Notify`如果在中断上下文（ISR）里调用，要确保所有注册的回调函数本身都满足"中断安全"（不可重入函数不能调、执行时间要短），这属于`coding_standard.md`第20章中断处理规范的范畴，观察者模式本身不会自动保证这一点。
- 多任务下`Event_Register`和`Event_Notify`如果可能并发调用（比如一个任务注册的同时另一个任务在触发通知），需要加锁保护`observers`数组，避免遍历时数组被修改。

---

## 三、命令模式（Command）

**核心思想**：将操作封装为命令对象，放入队列顺序执行，分离"做什么"和"什么时候做"。

**适用场景**：串口指令队列、需要按顺序执行的操作序列等。

```c
typedef void (*CommandFunc_t)(void);

typedef struct {
    CommandFunc_t execute;
    const char   *name;   /* 调试用 */
} Command_t;

#define CMD_QUEUE_SIZE 16
static Command_t cmd_queue[CMD_QUEUE_SIZE];
static uint8_t   cmd_head = 0, cmd_tail = 0;

/* 返回值：0成功，-1表示队列已满 */
int CmdQueue_Push(CommandFunc_t func, const char *name)
{
    uint8_t next_tail = (cmd_tail + 1) % CMD_QUEUE_SIZE;
    if (next_tail == cmd_head) {
        LOG_ERROR("Command queue full, drop: %s", name);
        return -1;   /* 队列满：必须检测，否则会覆盖还没执行的命令 */
    }
    cmd_queue[cmd_tail].execute = func;
    cmd_queue[cmd_tail].name    = name;
    cmd_tail = next_tail;
    return 0;
}

void CmdQueue_Process(void)
{
    if (cmd_head != cmd_tail) {
        cmd_queue[cmd_head].execute();
        cmd_head = (cmd_head + 1) % CMD_QUEUE_SIZE;
    }
}

/* 使用示例 */
void Cmd_SetLedOn(void)   { LED_Set(LED_ON); }
void Cmd_SendStatus(void) { Comm_Send(Get_Device_Status()); }

CmdQueue_Push(Cmd_SetLedOn,   "LED ON");
CmdQueue_Push(Cmd_SendStatus, "SEND STATUS");
```

**优点**：解耦请求者与执行者，支持排队、延迟执行等。

**坑：**

- 原始写法的环形缓冲区**没有满队列检测**：`cmd_tail`绕回追上`cmd_head`时会直接覆盖还没被执行的命令，且`Push`永远返回成功假象。生产代码里环形缓冲区必须判断"下一个tail是否等于head"，满了就拒绝写入并报错，这是环形队列最经典的坑之一。
- 如果`CmdQueue_Push`可能被中断和主循环同时调用（比如串口中断里push命令，主循环里process），`cmd_head`/`cmd_tail`的读写要考虑原子性，普通的`uint8_t`自增在大部分MCU上是原子的，但如果索引类型变成`uint32_t`或者操作变成"读-改-写"多步，就需要关中断或加锁保护。
- `execute`函数指针不带参数，如果命令需要携带数据（比如"设置LED为某个颜色"），要么扩展`Command_t`加一个`void *arg`字段，要么每个命令单独用闭包变量（C没有闭包，通常用一个静态/全局的"命令参数暂存区"，注意多命令并发时的数据覆盖风险）。

---

## 四、工厂模式（Factory）

**核心思想**：集中管理对象创建，根据条件返回不同实现，业务代码依赖统一接口而非具体实现。

**适用场景**：不同硬件版本、多种传感器驱动选择等。

```c
/* 传感器统一接口 */
typedef struct {
    void    (*Init)(void);
    float   (*ReadTemp)(void);
    float   (*ReadHumi)(void);
    uint8_t (*IsReady)(void);
} SensorDriver_t;

/* 两种传感器实现，用const修饰——运行时不应被篡改，且编译器可放入只读段 */
static const SensorDriver_t sht30_driver = {
    .Init     = SHT30_Init,
    .ReadTemp = SHT30_ReadTemp,
    .ReadHumi = SHT30_ReadHumi,
    .IsReady  = SHT30_IsReady,
};

static const SensorDriver_t aht21_driver = {
    .Init     = AHT21_Init,
    .ReadTemp = AHT21_ReadTemp,
    .ReadHumi = AHT21_ReadHumi,
    .IsReady  = AHT21_IsReady,
};

/* 工厂函数 */
const SensorDriver_t *Sensor_GetDriver(void)
{
    uint8_t hw_ver = Read_HW_Version();
    if (hw_ver >= 3) {
        return &sht30_driver;
    }
    return &aht21_driver;
}

/* 业务代码不关心具体型号 */
const SensorDriver_t *sensor = Sensor_GetDriver();
sensor->Init();
float temp = sensor->ReadTemp();
```

**优点**：切换实现只需修改工厂函数，业务代码无感。

**坑：**

- 函数指针表（vtable）一旦确定运行时不会修改，务必加`const`——除了防止被意外篡改，链接器还能把它放进`.rodata`只读段，Flash型MCU上这块内存甚至不占RAM。这跟`references/struct_union_techniques.md`第八节"vtable模式"是同一套技巧，可以对照看。
- `Sensor_GetDriver`每次调用都会重新判断硬件版本，如果`Read_HW_Version()`开销较大（比如要读一次GPIO或I2C），且硬件版本运行时不会变化，可以在系统初始化时判断一次并缓存结果，避免重复查询。
- 新增第三种传感器时，要同时改`Sensor_GetDriver`里的判断逻辑——如果分支变多（比如按硬件版本区分4种以上传感器），考虑改成"版本号→驱动表"的查表方式（结合`coding_standard.md`第4章命名规范里提到的思路，用数组下标或`switch`跳转表代替长`if-else`链）。

---

## 五、单例模式（Singleton）

**核心思想**：保证全局唯一实例，并且只初始化一次。

**适用场景**：串口、Flash管理器、日志系统等全局资源。

```c
typedef struct {
    uint8_t  initialized;
    uint32_t write_count;
    uint32_t current_sector;
} FlashManager_t;

FlashManager_t *FlashMgr_GetInstance(void)
{
    static FlashManager_t instance = {0};

    if (!instance.initialized) {
        instance.current_sector = FLASH_DATA_START_SECTOR;
        instance.write_count    = 0;
        instance.initialized    = 1;
        Flash_HW_Init();
    }

    return &instance;
}

/* 任意模块获取的都是同一个实例 */
FlashManager_t *mgr = FlashMgr_GetInstance();
```

**优点**：避免重复初始化，确保资源访问一致性。

**坑（这是本模式在嵌入式RTOS环境下最容易被忽视的问题）：**

- 上面这种"首次调用时惰性初始化"（lazy initialization）的写法，在**多任务环境下有竞态条件**：如果两个任务几乎同时第一次调用`FlashMgr_GetInstance()`，都可能读到`initialized == 0`，都执行一遍初始化逻辑（`Flash_HW_Init()`被调用两次，甚至`write_count`被重复清零）。这是经典的"双重检查锁"问题，C11以下没有标准库的`pthread_once`等价物时，简单的判断无法保证原子性。
- 嵌入式场景下更推荐的做法是**避免运行时惰性初始化，改为在系统启动阶段（`main`函数早期、调度器启动前）显式调用一次`FlashMgr_Init()`**，之后`FlashMgr_GetInstance()`只负责返回指针，不做任何初始化判断——这样从根本上消除了竞态窗口，也符合`coding_standard.md`规则23.10"可测性设计：避免静态初始化依赖"的精神（初始化时机应该显式可控，而不是隐藏在第一次调用里）。
- 如果确实需要保留惰性初始化（比如某些资源只有部分任务会用到，不想在启动时都初始化一遍），必须用互斥锁或者关中断的方式保护"检查+初始化"这段临界区，NuttX下可以用`pthread_mutex`或`sched_lock()`（可在`references/nuttx_api.md`的Pthread接口章节查具体用法）。

---

## 六、装饰器模式（Decorator）

**核心思想**：在不修改原函数的基础上，通过包装函数动态添加功能（如重试、超时、日志）。

**适用场景**：传感器读取加重试、超时保护、性能统计等。

```c
typedef SensorStatus_t (*SensorReadFunc_t)(float *value);

/* 原始函数 */
SensorStatus_t Read_RawSensor(float *value)
{
    return I2C_Read_Sensor(value);
}

/* 装饰器：重试逻辑 */
SensorStatus_t Read_WithRetry(SensorReadFunc_t read_fn,
                               float *value,
                               uint8_t max_retry)
{
    for (uint8_t i = 0; i < max_retry; i++) {
        SensorStatus_t ret = read_fn(value);
        if (ret == SENSOR_OK) {
            return SENSOR_OK;
        }
        LOG_WARN("Read failed, retry %d/%d", i + 1, max_retry);
        HAL_Delay(10);
    }
    return SENSOR_ERR_TIMEOUT;
}

/* 装饰器：超时保护 */
SensorStatus_t Read_WithTimeout(SensorReadFunc_t read_fn,
                                 float *value,
                                 uint32_t timeout_ms)
{
    uint32_t start = HAL_GetTick();
    while (HAL_GetTick() - start < timeout_ms) {
        if (read_fn(value) == SENSOR_OK) {
            return SENSOR_OK;
        }
    }
    return SENSOR_ERR_TIMEOUT;
}

/* 动态组合功能 */
float temp;
Read_WithRetry(Read_RawSensor, &temp, 3);
```

**优点**：原函数保持简单，功能可灵活组合。

**坑：**

- `Read_WithTimeout`里的`while`循环没有任何延时，会疯狂轮询`read_fn`——如果`read_fn`本身是I2C/SPI读取，短时间内反复触发可能超出外设的访问频率限制，或者占满CPU导致其他任务饿死。建议循环体内加一个小延时（比如1ms），或者改用带超时参数的阻塞式I/O。
- `Read_WithTimeout`使用`HAL_GetTick() - start`判断超时，如果tick计数器发生**溢出回绕**（比如32位tick跑满后归零），这种减法在无符号整数运算下大多数情况正确（回绕后减法结果依然正确，这是无符号整数运算的一个有用特性），但如果误用有符号类型来存tick就会出问题——这类"卷绕比较"最好按`coding_standard.md`规则15.3、15.8关于有符号/无符号处理的要求来检查。
- 两个装饰器（重试+超时）如果要同时用，需要手写一个新函数把两者结合，不能简单嵌套调用（`Read_WithTimeout(Read_WithRetry(...))`这种写法在C里行不通，因为`Read_WithRetry`不是`SensorReadFunc_t`类型）。如果确实需要多层装饰组合，通常要么写一个统一的"读取选项结构体"（把retry、timeout都作为参数传给同一个函数），要么接受"每种组合手写一个函数"的现实——这是C语言装饰器模式相比面向对象语言的天然局限。

---

## 七、代理模式（Proxy）

**核心思想**：在业务逻辑与底层硬件之间增加代理层，控制访问并添加额外处理（权限、日志、缓存等）。

**适用场景**：分层架构中上层不直接操作硬件。

```c
/* 代理层结构 */
typedef struct {
    void    (*Set)(uint8_t state);
    uint8_t (*Get)(void);
    uint8_t locked;   /* 权限控制 */
} LEDProxy_t;

static void LED_ProxySet(uint8_t state)
{
    LOG_DEBUG("LED state -> %d", state);
    HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin,
                       state ? GPIO_PIN_SET : GPIO_PIN_RESET);
}

static LEDProxy_t led_proxy = {
    .Set    = LED_ProxySet,
    .locked = 0,
};

/* 业务代码只与代理交互 */
void Business_Logic(void)
{
    if (!led_proxy.locked) {
        led_proxy.Set(1);
        HAL_Delay(100);
        led_proxy.Set(0);
    }
}
```

**优点**：硬件平台变更只需修改代理层，业务代码不动；可方便添加控制。

**坑：**

- `led_proxy`是非`const`的全局可写结构体，`locked`字段任何模块都能直接改（`led_proxy.locked = 1;`），失去了"代理控制访问权限"的本意。如果权限控制要真正生效，应该把`locked`字段隐藏（放到`.c`文件的`static`变量里，只通过`LEDProxy_Lock()`/`LEDProxy_Unlock()`接口修改），而不是暴露在公开结构体里任人修改。
- 这个模式本质上和第四节的"工厂+统一接口"结构很像，区别在于**代理模式关注的是"控制访问"（权限、限流、缓存），工厂模式关注的是"选择实现"**——如果你的场景既要选择不同硬件实现、又要控制访问权限，两者可以叠加使用（工厂返回具体驱动，代理包一层控制逻辑）。

---

## 八、模板方法模式（Template Method）

**核心思想**：定义固定流程骨架（模板函数），各步骤的具体实现由子模块提供，实现代码复用。

**适用场景**：多传感器初始化流程固定但各步骤细节不同等场景。

```c
/* 步骤接口 */
typedef struct {
    void     (*PowerOn)(void);
    uint32_t (*GetWarmupMs)(void);
    uint8_t  (*SendConfig)(void);
    uint8_t  (*Verify)(void);
} SensorInitOps_t;

/* 模板函数：流程固定 */
SensorStatus_t Sensor_Init_Template(const SensorInitOps_t *ops)
{
    if (ops == NULL || ops->PowerOn == NULL || ops->GetWarmupMs == NULL ||
        ops->SendConfig == NULL || ops->Verify == NULL) {
        LOG_ERROR("Sensor init ops incomplete");
        return SENSOR_ERR_INIT;
    }

    ops->PowerOn();
    HAL_Delay(ops->GetWarmupMs());

    if (!ops->SendConfig()) {
        LOG_ERROR("Config failed");
        return SENSOR_ERR_INIT;
    }

    if (!ops->Verify()) {
        LOG_ERROR("Verify failed");
        return SENSOR_ERR_INIT;
    }

    LOG_INFO("Sensor init OK");
    return SENSOR_OK;
}

/* 不同传感器提供各自的步骤实现 */
static const SensorInitOps_t sht30_init_ops = {
    .PowerOn     = SHT30_PowerOn,
    .GetWarmupMs = SHT30_GetWarmupMs,
    .SendConfig  = SHT30_SendConfig,
    .Verify      = SHT30_Verify,
};

static const SensorInitOps_t bme280_init_ops = {
    .PowerOn     = BME280_PowerOn,
    .GetWarmupMs = BME280_GetWarmupMs,
    .SendConfig  = BME280_SendConfig,
    .Verify      = BME280_Verify,
};

/* 调用 */
Sensor_Init_Template(&sht30_init_ops);
Sensor_Init_Template(&bme280_init_ops);
```

**优点**：新增实现只需提供步骤函数，模板流程无需修改。

**坑：**

- 原始写法直接调用`ops->PowerOn()`等，如果某个具体传感器的ops表漏填了某个字段（函数指针为`NULL`），调用时会直接跳转到地址0附近，大概率触发HardFault——补上开头的空指针校验（`coding_standard.md`规则10.3"结构体指针使用时必须判空"的延伸，这里是"结构体里的函数指针成员使用前判空"）能在开发阶段尽早暴露"忘记实现某个步骤"的问题，而不是量产后才在特定传感器型号上crash。
- 模板方法的流程一旦固定，如果某个传感器的初始化流程本身有差异（比如需要多一步"校准"），要么在`SensorInitOps_t`里加可选字段（允许为`NULL`表示跳过），要么这个传感器就不适合套模板方法、需要单独写初始化函数——不要为了"复用模板"硬塞进不匹配的流程。

---

## 九、策略模式（Strategy）

**核心思想**：把一组可互相替换的算法/策略各自实现成独立函数，通过统一函数指针类型在运行时切换，调用方不关心具体用的是哪种策略。

**适用场景**：滤波算法切换（滑动平均/卡尔曼/中值滤波）、加密算法切换、不同精度等级的定位算法等。

**和工厂模式的区别**：工厂模式关心"根据什么条件创建/选择哪个对象"（通常对应硬件差异），策略模式关心"同一个问题有多种算法，运行时按需切换"（通常是算法/业务逻辑差异），两者经常配合使用（工厂造出的对象内部可能又用策略模式选算法）。

```c
typedef float (*FilterFunc_t)(float *history, uint8_t len, float new_sample);

float Filter_MovingAverage(float *history, uint8_t len, float new_sample)
{
    /* 滑动平均实现 */
    float sum = new_sample;
    for (uint8_t i = 0; i < len - 1; i++) {
        history[i] = history[i + 1];
        sum += history[i];
    }
    history[len - 1] = new_sample;
    return sum / len;
}

float Filter_Median(float *history, uint8_t len, float new_sample)
{
    /* 中值滤波实现（示意，实际需要排序逻辑） */
    /* ... */
    return new_sample;
}

typedef struct {
    FilterFunc_t filter;
    float        history[8];
    uint8_t      len;
} SensorFilter_t;

void SensorFilter_SetStrategy(SensorFilter_t *sf, FilterFunc_t strategy)
{
    sf->filter = strategy;
}

float SensorFilter_Process(SensorFilter_t *sf, float new_sample)
{
    return sf->filter(sf->history, sf->len, new_sample);
}

/* 运行时按配置切换滤波策略 */
SensorFilter_t sf = { .len = 8 };
SensorFilter_SetStrategy(&sf, Filter_MovingAverage);
float out = SensorFilter_Process(&sf, raw_sample);
```

**优点**：新增一种算法只需新写一个符合签名的函数，不用改调用方代码；便于A/B对比不同算法效果。

**坑：**

- 策略函数的签名（参数、返回值）要在设计初期就定好，中途改签名意味着所有实现该策略的函数都要跟着改——这类"统一接口"设计建议参考`coding_standard.md`第22章API设计与版本管理规范，提前想清楚未来可能的扩展参数，避免后期签名频繁变动。
- 如果各策略内部状态（比如`history`缓冲区）大小不同，`SensorFilter_t`里固定大小的`history[8]`可能不够用或浪费——需要更大灵活性时，可以结合本技能`references/struct_union_techniques.md`里的柔性数组成员或者按最大需求分配。

---

## 十、适配器模式（Adapter）

**核心思想**：把一个接口不兼容的底层实现，包一层转换成上层期望的统一接口，专门解决"新旧接口不匹配"或"多家硬件厂商接口不统一"的问题。

**适用场景**：更换Flash/传感器供应商但引脚定义和寄存器操作方式不同、迁移HAL库版本、对接第三方SDK但接口风格和自己项目不一致。

**和代理模式的区别**：代理模式的上下游接口是**相同的**，代理只是在中间加控制逻辑；适配器模式的上下游接口是**不同的**，适配器的核心工作是"转换"而不是"控制"。

```c
/* 项目统一的Flash接口（上层期望的样子） */
typedef struct {
    int (*read)(uint32_t addr, uint8_t *buf, uint32_t len);
    int (*write)(uint32_t addr, const uint8_t *buf, uint32_t len);
    int (*erase_sector)(uint32_t addr);
} FlashOps_t;

/* 第三方SDK提供的接口，参数顺序、返回值含义都不一样 */
extern int8_t VendorFlash_ReadBytes(uint8_t *dst, uint32_t src_addr, uint16_t n);
extern int8_t VendorFlash_WriteBytes(uint32_t dst_addr, uint8_t *src, uint16_t n);
extern int8_t VendorFlash_EraseSector(uint16_t sector_id);

/* 适配器：把厂商接口"翻译"成项目统一接口 */
static int Adapter_FlashRead(uint32_t addr, uint8_t *buf, uint32_t len)
{
    /* 参数顺序不同、长度类型不同(uint16_t vs uint32_t)，这里做转换和边界检查 */
    if (len > UINT16_MAX) {
        return -1;
    }
    return (VendorFlash_ReadBytes(buf, addr, (uint16_t)len) == 0) ? 0 : -1;
}

static int Adapter_FlashWrite(uint32_t addr, const uint8_t *buf, uint32_t len)
{
    if (len > UINT16_MAX) {
        return -1;
    }
    return (VendorFlash_WriteBytes(addr, (uint8_t *)buf, (uint16_t)len) == 0) ? 0 : -1;
}

static int Adapter_FlashEraseSector(uint32_t addr)
{
    uint16_t sector_id = (uint16_t)(addr / FLASH_SECTOR_SIZE);
    return (VendorFlash_EraseSector(sector_id) == 0) ? 0 : -1;
}

const FlashOps_t vendor_flash_adapter = {
    .read         = Adapter_FlashRead,
    .write        = Adapter_FlashWrite,
    .erase_sector = Adapter_FlashEraseSector,
};

/* 业务代码只认识 FlashOps_t，完全不知道背后是哪家厂商的SDK */
```

**优点**：更换供应商/SDK版本时只需重写适配器层，业务代码完全不用动——这正是`coding_standard.md`原则1.2"层间依赖方向"和第26章"跨平台开发规范"里"平台差异必须通过抽象层隔离"的具体落地。

**坑：**

- 适配器层经常涉及**类型收窄转换**（如上面`uint32_t len`转`uint16_t`），一定要做范围检查后再强转，直接截断会导致长度参数错误却不报错，属于`coding_standard.md`规则15.7"禁止隐式截断转换"重点提防的场景。
- 如果厂商SDK的错误码含义跟项目自己的错误码体系不一样（比如厂商用"0=成功"，而项目里"0"可能表示别的含义），适配器层必须做错误码的显式映射，而不是简单地透传或者假设两边语义相同。
- 适配器函数应该保持"薄"——只做接口转换和必要的边界检查，不要在适配器里夹带业务逻辑，否则出问题时很难判断到底是适配层的bug还是业务层的bug。

---

## 十一、对象池模式（Object Pool）

**核心思想**：预先分配一组固定数量的对象放入"池子"，需要时从池子里取，用完归还，而不是每次都动态`malloc`/`free`。

**适用场景**：嵌入式系统内存紧张、禁止或限制动态内存分配的场景（如安全关键系统、RTOS任务栈受限环境）；协议解析中频繁创建/销毁的消息缓冲区、事件对象等。

```c
#define POOL_SIZE 8

typedef struct {
    uint8_t data[64];
    uint8_t len;
} MsgBuf_t;

typedef struct {
    MsgBuf_t pool[POOL_SIZE];
    uint8_t  used[POOL_SIZE];   /* 0=空闲，1=占用 */
} MsgBufPool_t;

static MsgBufPool_t g_msg_pool;

/* 从池子里取一个空闲对象，取不到返回NULL（调用者必须检查！） */
MsgBuf_t *MsgBufPool_Acquire(void)
{
    for (uint8_t i = 0; i < POOL_SIZE; i++) {
        if (!g_msg_pool.used[i]) {
            g_msg_pool.used[i] = 1;
            memset(&g_msg_pool.pool[i], 0, sizeof(MsgBuf_t));
            return &g_msg_pool.pool[i];
        }
    }
    LOG_WARN("MsgBufPool exhausted");
    return NULL;   /* 池子耗尽，绝不能返回野指针或者复用正在使用的对象 */
}

/* 用完必须归还，否则池子会被"泄漏"耗尽 */
void MsgBufPool_Release(MsgBuf_t *buf)
{
    for (uint8_t i = 0; i < POOL_SIZE; i++) {
        if (&g_msg_pool.pool[i] == buf) {
            g_msg_pool.used[i] = 0;
            return;
        }
    }
    LOG_ERROR("MsgBufPool_Release: invalid pointer");
}

/* 使用示例 */
MsgBuf_t *msg = MsgBufPool_Acquire();
if (msg == NULL) {
    return;   /* 池子满了，必须处理，不能假设一定能拿到 */
}
/* ... 使用msg ... */
MsgBufPool_Release(msg);
```

**优点**：

- 完全避免运行时`malloc`/`free`，杜绝内存碎片，也符合很多安全关键嵌入式规范（如MISRA C）"禁止动态内存分配"的要求
- 对象数量和内存占用在编译期就确定，便于做内存预算和最坏情况分析（WCET/WCMU分析）
- 比每次手写static数组管理更结构化，池子逻辑可以复用到多种对象类型

**坑：**

- **"忘记归还"是这个模式最大的坑**——如果某条错误处理路径提前`return`，忘记调用`Release`，池子会像内存泄漏一样被逐渐耗尽，最终`Acquire`永远返回`NULL`。写这类代码时要格外注意`coding_standard.md`规则12.8"资源获取即初始化（RAII风格）"的思路：每一条可能的退出路径都要确保资源被释放，必要时用`do { ... } while(0)` + `break`的统一出口写法（见`references/loop_branch_techniques.md`第三节）来避免遗漏。
- `Acquire`返回`NULL`时调用方必须处理，绝不能假设"池子肯定有空闲对象"就跳过判空直接使用——这是`coding_standard.md`规则3.6"函数返回值必须校验"的具体场景。
- 多任务/中断环境下，`Acquire`/`Release`对`used`数组的读写如果存在并发访问，必须加锁或关中断保护，否则两个任务可能同时"抢到"同一个对象槽位。
- 池子大小（`POOL_SIZE`）定死后，业务突发流量超过池子容量时只能拒绝服务（返回`NULL`）——设计容量时要按最坏情况（而不是平均情况）预估，或者在池子耗尽时有合理的降级/丢弃策略。

---

## 十二、如何选择：常见模式对照表

| 模式 | 解决的问题 | 关键结构 | 最相关的坑 | 关联章节 |
|------|-----------|---------|-----------|---------|
| 状态机 | 多状态切换逻辑混乱 | enum + switch | 忘记default兜底 | `loop_branch_techniques.md`一 |
| 观察者 | 一个事件多方响应 | 回调数组 | 注册满/未注销悬空指针 | `coding_standard.md`第3章返回值校验 |
| 命令 | 操作排队延迟执行 | 函数指针+环形队列 | 队列满未检测导致覆盖 | — |
| 工厂 | 按条件选择实现 | 函数指针表+选择函数 | vtable应加const | `struct_union_techniques.md`八 |
| 单例 | 全局唯一实例 | static局部变量 | 惰性初始化的竞态条件 | `coding_standard.md`23.10 |
| 装饰器 | 给函数动态加功能 | 高阶函数包装 | 多层装饰无法简单嵌套 | — |
| 代理 | 控制/限制访问 | 结构体+权限字段 | 权限字段未真正隐藏 | — |
| 模板方法 | 固定流程+可变步骤 | ops结构体+统一调用函数 | 忘记给函数指针成员判空 | `coding_standard.md`10.3 |
| 策略 | 同一问题多种算法切换 | 统一函数指针类型 | 签名设计不周全后期难改 | `coding_standard.md`第22章 |
| 适配器 | 新旧/多方接口不兼容 | 转换函数+统一接口 | 类型收窄转换未做范围检查 | `coding_standard.md`15.7 |
| 对象池 | 避免动态内存分配 | 固定数组+占用标记 | 忘记归还导致池子耗尽 | `coding_standard.md`12.8 |

**选型建议**：不要为了用模式而用模式。先看当前代码的痛点是什么——"状态切换乱"用状态机，"要不要动态内存"用对象池，"接口不兼容"用适配器——痛点对上了再套模式，痛点不明显时，一个直白的函数往往比强行套模式更好维护（呼应`coding_standard.md`原则1.4"避免过度设计"：三次重复出现才考虑抽象，不为单一场景设计框架）。

**文档结束**
