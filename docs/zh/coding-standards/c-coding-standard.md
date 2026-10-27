---
title: C 语言编码规范
date: 2026-10-27
description: 覆盖架构设计、头文件、函数、命名、排版、注释、内存与安全等内容的 C 语言编码规范 v2.0。
tags: [C语言, 编码规范, 编程规范]
---

# C语言编码规范

> **版本**：v2.0  
> **生效日期**：发布之日起  
> **适用范围**：公司所有C语言软件开发项目  
> **规范性质**：强制遵守（规则级）/ 推荐采纳（建议级）

---

## 0 规范制订说明

### 0.1 前言

**写法**：本规范以"原则→规则→建议"三级体系定义编码要求。原则是指导思想不可违背，规则是强制约束违反需审批，建议是推荐做法可酌情偏离。

**示例**：每一条规范条目均按以下结构组织：
```
### 规则/原则 X.Y 标题

**写法**：正确做法的描述和代码模式

**示例**：
```c
/* 错误 */   // 违反规范的写法
/* 正确 */   // 符合规范的写法
```

**警示**：⚠️ 不遵循此规范可能导致的后果和风险
```

**警示**：缺乏统一编码规范的组织，代码质量完全依赖个人水平，人员流动将导致代码无法维护。历史数据显示，不规范代码的缺陷密度是规范代码的 **3~5倍**，维护成本高出 **2~4倍**。

### 0.2 代码总体原则

**写法**：以下五项原则是编码的最高指导方针，所有规则和建议均在此基础上制定。代码审查时，违反原则的代码可以直接被驳回。

**示例**：

| 原则 | 核心要求 | 违背后果 |
|------|---------|---------|
| 清晰第一 | 代码可读性优先于技巧性 | 晦涩代码增加100%审查时间 |
| 简洁为美 | 删除无用代码，合并重复逻辑 | 冗余代码增加50%维护成本 |
| 风格一致 | 存量代码遵循原有风格 | 风格混搭降低可读性 |
| 防御性编程 | 假设所有输入都是恶意的 | 缺少校验导致安全漏洞 |
| 安全优先 | 功能/性能/安全冲突时选安全 | 安全是底线，不可妥协 |

**警示**：这些原则是长期实践总结的教训。例如"清晰第一"原则源自某项目因过度使用宏和内联导致代码无法调试，最终返工3个月的严重事故。

### 0.3 规范实施与解释

**写法**：本规范采用"渐进式推行"策略。存量代码在修改时逐步适配，新增代码自发布之日起强制遵守。

**示例**：
```c
/* 特殊情况下违反规则必须标注原因 */
/* RULE_OVERRIDE: 此处使用内联汇编是硬件访问的唯一方式，已通过架构评审 */
__asm__ volatile("wfi");
```

**警示**：
- ⚠️ `RULE_OVERRIDE` 标注必须经过正式评审，未经评审的覆盖标注等同于违规
- ⚠️ 每年至少评审一次规范，技术环境变化可能导致原有规则失效
- ⚠️ 新员工入职 **1个月内** 必须完成本规范的培训和考核

### 0.4 术语定义

**写法**：本规范中使用以下术语，所有条文的解释以本术语表为准。

**示例**：

| 术语 | 说明 | 典型场景 |
|------|------|---------|
| **原则** | 编程时必须坚持的指导思想，不可违背 | 架构设计、代码审查 |
| **规则** | 编程时强制必须遵守的约定，违反需审批 | 日常编码、CI检查 |
| **建议** | 编程时推荐采纳的约定，特殊情况可偏离 | 代码优化、重构 |
| **外部数据** | 来自文件、网络、用户输入、环境变量、IPC、函数参数等不可信数据 | 安全审查重点关注 |
| **安全函数** | 带长度/边界检查的函数替代版本（如 `strcpy_s`、`snprintf`） | 字符串操作、I/O操作 |
| **防御性编程** | 通过校验、断言、容错等手段预防和捕获编程错误的编码风格 | 输入校验、边界检查 |

**警示**：术语理解不一致是团队协作中常见的沟通障碍来源。例如"外部数据"的定义范围不清，可能导致开发者遗漏对某些数据源的校验，埋下安全隐患。

---

## 1 总体架构与设计原则

### 原则1.1 模块化设计

**写法**：每个模块应职责单一、接口清晰、内部实现隐藏。模块间通过头文件定义的接口通信，禁止直接访问其他模块的内部全局变量。模块划分应遵循"高内聚、低耦合"原则。

**示例**：
```c
/* 模块划分示例：网络模块 */
// network/tcp_client.h — 对外接口（公开）
int tcp_connect(const char *host, uint16_t port);
int tcp_send(int fd, const void *data, size_t len);
void tcp_close(int fd);

// network/tcp_client.c — 内部实现（隐藏）
static int s_connection_count = 0;          /* 内部全局变量，static隐藏 */
static void tcp_set_socket_timeout(int fd); /* 内部辅助函数，static隐藏 */

/* 错误：跨模块直接访问内部变量 */
// 在 http_client.c 中
extern int s_connection_count;  /* 错误：访问其他模块的static变量 */

/* 正确：通过接口访问 */
int count = tcp_get_connection_count();  /* 通过公开接口访问 */
```

**警示**：⚠️ 违反模块化设计将导致"意大利面条式代码"，修改一个模块可能影响多个无关模块。某项目因模块间全局变量直接耦合，导致一次安全升级需要修改 47 个文件，最终延期 2 个月。

### 原则1.2 层间依赖方向

**写法**：依赖方向应遵循：**产品代码 → 平台抽象层 → 标准库/操作系统**。禁止下层代码依赖上层代码，禁止循环依赖。

**示例**：
```c
/* 正确的依赖方向 */
// 产品代码层（上层）
#include "platform/network.h"    /* 依赖平台抽象层 */
int result = platform_send(data, len);

// 平台抽象层（中间层）
#include <sys/socket.h>          /* 依赖操作系统 */
int platform_send(const void *data, size_t len) {
    return send(sock_fd, data, len, 0);
}

/* 错误的依赖方向 */
// 平台抽象层反向依赖产品代码
#include "business/logic.h"      /* 错误：下层依赖上层 */
```

**警示**：⚠️ 循环依赖是架构腐化的最明显信号。一旦出现循环依赖，模块将无法独立测试、独立编译、独立发布。检测方法：在构建系统中如果出现 `A depends on B` 且 `B depends on A`，必须立即重构。

### 原则1.3 接口与实现分离

**写法**：头文件（`.h`）只放置接口声明，不放置实现。源文件（`.c`）包含实现，内部函数和变量用 `static` 隐藏。对外可见的符号应尽量少，遵循"最小暴露"原则。

**示例**：
```c
/* module.h — 只暴露必要的接口 */
#ifndef __MODULE_MODULE_H__
#define __MODULE_MODULE_H__

#include <stdint.h>

/* 公开接口声明 */
int module_init(void);
int module_process(const uint8_t *data, size_t len);
void module_deinit(void);

#endif /* __MODULE_MODULE_H__ */

/* module.c — 隐藏所有实现细节 */
#include "module.h"

/* 内部函数用static隐藏 */
static void validate_checksum(const uint8_t *data, size_t len);
static void reset_internal_state(void);

/* 内部全局变量用static隐藏 */
static uint32_t s_internal_counter = 0;

/* 公开接口实现 */
int module_init(void) {
    reset_internal_state();
    return 0;
}
```

**警示**：⚠️ 头文件暴露过多实现细节会导致：① 调用者可能绕开接口直接操作内部数据；② 修改内部实现需要同步修改所有调用者；③ 增加模块间的编译依赖，降低构建速度。一个常见的错误是在头文件中定义内部结构体的完整布局，而实际上调用者只需要一个不透明指针。

### 原则1.4 避免过度设计

**写法**：不为未来不确定的需求设计抽象层，不为单一使用场景设计通用框架。三次重复出现才考虑提取公共代码。遵循 **YAGNI（You Ain't Gonna Need It）** 原则。

**示例**：
```c
/* 错误：过度设计 — 为单一使用场景设计抽象工厂模式 */
typedef struct {
    void *(*create)(void);
    void (*destroy)(void *);
    int (*process)(void *);
} factory_t;

/* 当前只有一个数据源，却在设计抽象层 */
factory_t *get_factory(const char *type) { ... }

/* 正确：保持简单，真正需要时再重构 */
int read_sensor_data(int sensor_id, double *out_value);

/* 当第三次出现不同数据源时，再考虑提取抽象层 */
/* 第一次：读取温度传感器 */
/* 第二次：读取湿度传感器 */  
/* 第三次：读取压力传感器 → 此时提取抽象层 */
```

**警示**：⚠️ 过度设计比欠设计更危险，因为：
- 抽象层增加了代码复杂度，却没有带来实际收益
- 未来需求变化时，基于假设的抽象往往需要推倒重来
- 每增加一层抽象，调试难度增加一层，性能损耗增加一层

**经验法则**：如果不知道未来会怎样，选择最简单的方案。当同一种模式出现 **3次** 时，才是考虑抽象的正确时机。

---

## 2 头文件规范

### 规则2.1 每一个.c文件应有同名.h文件

**写法**：每个 `.c` 文件应有对应的 `.h` 文件，用于声明需要对外公开的接口。如果 `.c` 文件不需要对外暴露任何接口，则仍需保留同名 `.h` 文件（内容可仅为头文件保护宏）。

**示例**：
```c
// 文件结构
// module.c  →  module.h  （必须成对）
// main.c    →  无.h      （例外：main函数所在文件）
// test.c    →  无.h      （例外：测试文件）

// 即使模块不需要对外暴露接口，也保留头文件
// internal_helper.h
#ifndef __UTILS_INTERNAL_HELPER_H__
#define __UTILS_INTERNAL_HELPER_H__
/* 当前无对外接口，仅保留头文件保护宏，便于未来扩展 */
#endif /* __UTILS_INTERNAL_HELPER_H__ */
```

**警示**：⚠️ 缺少 `.h` 文件会导致：① 调用者无法获得类型检查，编译器无法发现参数类型不匹配的错误；② 接口变更时缺少编译时验证，运行时才能发现问题；③ 代码难以被其他模块复用。

### 规则2.2 禁止头文件循环依赖

**写法**：如果 A.h 包含 B.h，B.h 又直接或间接包含 A.h，则构成循环依赖。应通过前向声明、提取公共接口层等方式解除循环依赖。

**示例**：
```c
/* 错误：循环依赖 */
// a.h
#include "b.h"
typedef struct { int val; } a_t;
void process_a(a_t *a, b_t *b);

// b.h
#include "a.h"  
typedef struct { int val; } b_t;
void process_b(b_t *b, a_t *a);

/* 正确：使用前向声明打破循环 */
// a.h
typedef struct { int val; } a_t;
struct b_t;  /* 前向声明，不包含b.h */
void process_a(a_t *a, struct b_t *b);

// b.h  
typedef struct { int val; } b_t;
struct a_t;  /* 前向声明，不包含a.h */
void process_b(b_t *b, struct a_t *a);
```

**警示**：⚠️ 循环依赖的典型危害：
- 编译时：修改任何一个头文件都会触发所有依赖文件重编译，大型项目编译时间从分钟级变为小时级
- 链接时：循环依赖导致链接器无法确定初始化顺序，可能引发静态初始化顺序灾难
- 运行时：可能导致栈溢出（无限递归包含）
- 检测方法：在项目根目录执行 `cppcheck --check-config src/` 可检测循环依赖

### 规则2.3 头文件必须使用防止重复包含机制

**写法**：每个头文件必须使用 `#ifndef` / `#define` / `#endif` 三重保护宏。宏名称格式：`__<项目>_<模块>_<文件名>_H__`。

**示例**：
```c
/* 正确：标准格式 */
#ifndef __NETWORK_TCP_CLIENT_H__     /* 检查是否已定义 */
#define __NETWORK_TCP_CLIENT_H__     /* 定义保护宏 */

#include <stdint.h>

int tcp_connect(const char *host, uint16_t port);
void tcp_close(int fd);

#endif /* __NETWORK_TCP_CLIENT_H__ */  /* 结束保护 */

/* 错误：保护宏名称冲突 */
#define _H  /* 过于通用，容易与其他文件冲突 */
#define __HEADER__  /* 过于通用 */
```

**警示**：⚠️ 缺少重复包含保护会导致：① 头文件被多次包含时产生重复定义错误；② 大型项目中头文件嵌套包含可能导致编译失败。注意：`#pragma once` 虽然简洁但不是C标准，GCC和MSVC行为有细微差异，禁止使用。

### 规则2.4 禁止在头文件中定义变量

**写法**：头文件中只能使用 `extern` 声明变量，变量定义必须在 `.c` 文件中。这个规则的关键在于区分"声明"和"定义"。

**示例**：
```c
/* 错误：头文件中定义变量 */
// module.h
int g_global_counter = 0;         /* 定义：分配存储空间 */
void process(void);

/* 如果module.h被2个.c文件包含，链接时产生"重复定义"错误 */

/* 正确：头文件中声明，.c文件中定义 */
// module.h
extern int g_global_counter;      /* 声明：不分配存储空间 */
void process(void);

// module.c
int g_global_counter = 0;         /* 定义：只出现一次 */
```

**警示**：⚠️ 在头文件中定义变量是初学者最常见的错误之一。后果：
- 如果头文件被多个 `.c` 文件包含，链接器报 `multiple definition` 错误
- 即使使用 `static` 避免了链接错误，每个包含该头文件的 `.c` 文件都会有自己的独立副本，浪费内存
- 调试时难以定位，因为错误信息是链接阶段的，开发者在编译阶段不会发现

### 规则2.5 头文件包含顺序

**写法**：头文件包含应遵循以下顺序，每组之间用空行分隔。此顺序可尽早发现本模块头文件是否缺少必要的包含依赖。

**示例**：
```c
/* 正确：标准包含顺序 */
/* 1. 本模块头文件（检验头文件是否自包含） */
#include "module.h"

/* 2. 本项目其他模块头文件 */
#include "common/logger.h"
#include "network/tcp_client.h"

/* 3. 第三方库头文件 */
#include "lwip/tcp.h"

/* 4. 系统头文件 */
#include <stdio.h>
#include <string.h>

/* 错误：系统头文件放在最前面，本模块头文件缺失 */
#include <stdio.h>     /* 隐藏了本模块头文件是否自包含的问题 */
#include "module.h"    /* 如果module.h缺少#include <stdio.h>，这里不会报错 */
```

**警示**：⚠️ 不按此顺序包含头文件可能导致：① 本模块头文件缺少依赖但编译时因其他头文件提前引入而侥幸通过，换到其他编译器时出错；② 增加代码维护难度，使头文件依赖关系不清晰。此规则的核心价值在于：**让本模块头文件"裸编译"一次**，确保它的自包含性。

### 规则2.6 头文件职责单一

**写法**：一个头文件只对应一个模块或一个功能类，避免"万能头文件"。头文件包含的接口应是高内聚的。

**示例**：
```c
/* 错误：万能头文件，包罗万象 */
// common.h
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include "network.h"
#include "database.h"
#include "config.h"
#include "utils.h"

/* 正确：按模块拆分 */
// network/tcp_client.h    — TCP客户端接口
// network/udp_socket.h    — UDP套接字接口
// database/query.h        — 数据库查询接口
// database/connection.h   — 数据库连接接口
// config/app_config.h     — 应用配置接口
// utils/string_util.h     — 字符串工具接口
```

**警示**：⚠️ "万能头文件"的危害：① 修改任何模块接口都会触发所有包含该头文件的文件重编译，大型项目可能增加数十分钟编译时间；② 调用者不知道具体依赖哪些模块，导致隐式依赖；③ 代码审查时无法判断模块间的实际依赖关系。**一条经验法则**：如果头文件包含超过 10 个 `#include`，很可能已经违反本条规则。

### 规则2.7 使用前向声明减少头文件包含

**写法**：如果只需要指针类型，使用前向声明替代包含完整头文件，以减少编译依赖。

**示例**：
```c
/* 错误：为了使用指针类型而包含完整头文件 */
#include "other_module.h"  /* 实际只需要指针类型，不需要知道结构体细节 */
struct other_module_t *ptr;

/* 正确：使用前向声明 */
struct other_module_t;  /* 前向声明：告诉编译器这是一个类型名 */
void func(struct other_module_t *ptr);  /* 可以声明指针参数 */

/* 必须包含完整头文件的场景 */
void func(struct other_module_t val);  /* 值传递，需要知道结构体大小，必须包含头文件 */
```

**警示**：⚠️ 不必要的头文件包含会增加编译依赖链，导致：① 修改一个头文件会触发大量无关文件的重新编译；② 在大型项目中，每次 `git pull` 后可能需要等待数分钟到数十分钟的编译时间。**测试方法**：注释掉某个 `#include`，如果代码仍然能编译通过，说明该包含是不必要的。

### 建议2.1 头文件应自包含

**写法**：每个头文件都应能独立编译，即包含该头文件时不需要先包含其他头文件。可通过在 `.c` 文件中第一个包含本头文件来验证。

**示例**：
```c
/* 错误：头文件不自包含 */
// my_module.h
typedef struct { int x; int y; } point_t;  /* 假设使用了uint32_t */
/* 但忘记包含 <stdint.h>，依赖调用者提前包含 */

// 调用者代码
#include <stdint.h>     /* 必须在此处先包含 */
#include "my_module.h"  /* 才能编译通过 */

/* 正确：自包含的头文件 */
// my_module.h
#include <stdint.h>     /* 自包含：自己需要的头文件自己包含 */
typedef struct { uint32_t x; uint32_t y; } point_t;
```

**警示**：⚠️ 不自包含的头文件是最隐蔽的编译问题之一。它在某些编译器上可能编译通过（因为侥幸提前包含了所需头文件），但换到另一个编译器或调整包含顺序后就会失败。**验证方法**：在每个 `.c` 文件中，本模块头文件必须是第一个 `#include`。

### 建议2.2 头文件中的include guard命名

**写法**：Include guard 宏名称应遵循 `<项目>_<模块>_<文件名>_H_` 格式，确保全局唯一性。

**示例**：
```c
/* 正确：全局唯一的命名 */
#ifndef __MYPROJECT_NETWORK_TCP_CLIENT_H__
#define __MYPROJECT_NETWORK_TCP_CLIENT_H__

/* 错误：容易冲突的命名 */
#ifndef _H
#define _H

/* 错误：可能和系统头文件冲突 */
#ifndef _STRING_H
#define _STRING_H  /* 系统头文件也使用这个名称！ */
```

---

## 3 函数规范

### 原则3.1 一个函数仅完成一件功能

**写法**：遵循单一职责原则，函数应只做一件事，做好一件事。如果一个函数需要包含"并且"、"或者"等逻辑，应考虑拆分。函数名应能准确概括其全部行为。

**示例**：
```c
/* 错误：一个函数做了三件事 */
int process_user_data(const char *input) {
    /* 1. 解析输入 */
    parse_input(input, &parsed);
    /* 2. 校验数据 */
    validate_data(&parsed);
    /* 3. 保存到数据库 */
    save_to_database(&parsed);
    return 0;
}

/* 正确：拆分为三个函数，各司其职 */
int parse_user_input(const char *input, user_data_t *out);
int validate_user_data(const user_data_t *data);
int save_user_to_database(const user_data_t *data);
```

**警示**：⚠️ 违反单一职责的典型症状：① 函数名中含有"and"或"并且"；② 函数需要两个以上的缩进层次；③ 函数有多个不相关的错误码返回；④ 代码审查时需要用"第一件事...第二件事..."来描述函数功能。这种函数难以测试、难以复用、难以调试。

### 规则3.1 函数长度控制

**写法**：新增函数不超过 **50行**（非空非注释行）。超出时应考虑拆分为子函数。50行的限制针对函数体，不包括函数签名和注释。

**示例**：
```c
/* 错误：超过50行的函数 */
void process_message(const char *msg, size_t len) {
    /* 1. 校验消息头 — 20行 */
    /* 2. 解析消息体 — 25行 */
    /* 3. 处理业务逻辑 — 30行 */
    /* 4. 生成响应 — 15行 */
    /* 总计超过50行！ */
}

/* 正确：拆分为多个子函数 */
void process_message(const char *msg, size_t len) {
    msg_header_t header;
    if (!parse_header(msg, len, &header)) return;
    
    msg_body_t body;
    if (!parse_body(msg, len, &header, &body)) return;
    
    handle_business_logic(&body);
    
    generate_response(&body);
}
```

**警示**：⚠️ 长函数的危害：① 函数超过50行后，人脑很难在短时间内理解全部逻辑；② 长函数往往暗示违反了单一职责原则；③ 调试时难以定位问题所在区域；④ 代码覆盖率统计时，未覆盖的路径难以识别。**研究数据**：函数长度超过100行时，缺陷密度是50行以内函数的 **3倍**。

### 规则3.2 重复代码必须提炼成函数

**写法**：禁止复制粘贴代码。重复代码出现2次以上应提炼为函数，出现3次以上应纳入公共模块。

**示例**：
```c
/* 错误：重复代码 */
void save_user(const user_t *user) {
    char query[512];
    snprintf(query, sizeof(query), "INSERT INTO users VALUES (%u,'%s')", user->id, user->name);
    /* ... 执行SQL ... */
    log_info("User saved: %u", user->id);
}

void save_order(const order_t *order) {
    char query[512];
    snprintf(query, sizeof(query), "INSERT INTO orders VALUES (%u,'%s')", order->id, order->name);
    /* ... 执行SQL ... */
    log_info("Order saved: %u", order->id);  /* 重复了SQL构建和日志记录 */
}

/* 正确：提取公共函数 */
static int execute_insert(const char *table, const char *values) {
    char query[512];
    snprintf(query, sizeof(query), "INSERT INTO %s VALUES %s", table, values);
    int ret = db_execute(query);
    log_info("Inserted into %s: %s", table, values);
    return ret;
}

void save_user(const user_t *user) {
    char values[256];
    snprintf(values, sizeof(values), "(%u,'%s')", user->id, user->name);
    execute_insert("users", values);
}
```

**警示**：⚠️ 重复代码是代码质量的最大杀手之一，被称为"复制粘贴编程"：
- 修改一个逻辑时，需要修改所有重复位置，遗漏一个就产生bug
- 某项目因一个bug修复需要在12个重复位置做相同修改，遗漏了3个位置，导致线上故障
- 重复代码增加了代码审查的负担，审查者需要确认所有重复位置都已修改
- **检测工具**：使用 `cppcheck --enable=all` 或 `PMD CPD` 可自动检测重复代码

### 规则3.3 函数参数不超过5个

**写法**：参数超过5个时，应使用结构体打包。结构体方式使参数关系更清晰，且调用者可使用指定初始化器。

**示例**：
```c
/* 错误：参数超过5个 */
int create_task(const char *name, int priority, int stack_size,
                void *entry, void *arg, int cpu_affinity,
                int policy, int flags);

/* 正确：使用结构体打包 */
typedef struct {
    const char *name;       /* 任务名称 */
    int priority;           /* 优先级（0-99） */
    int stack_size;         /* 栈大小（字节） */
    void (*entry)(void *);  /* 入口函数 */
    void *arg;              /* 入口参数 */
    int cpu_affinity;       /* CPU亲和性（-1表示不绑定） */
    int policy;             /* 调度策略：SCHED_FIFO/SCHED_RR */
    int flags;              /* 创建标志位 */
} task_config_t;

int create_task(const task_config_t *config);
```

**警示**：⚠️ 参数过多的危害：① 调用者容易搞错参数顺序，导致难以发现的bug；② 每次调用都需要写大量参数，代码冗余；③ 增加或减少参数需要修改所有调用点；④ 使用结构体参数可以实现向后兼容（添加新字段不影响已有调用者）。

### 规则3.4 函数参数书写必须完整

**写法**：函数声明和定义中，每个参数必须同时有类型和参数名。缺少参数名使代码不可自注释。

**示例**：
```c
/* 错误：只写类型，不写参数名 */
int connect(char *, int);

/* 正确：参数名说明参数含义 */
int connect(const char *host, int port);

/* 正确：参数名让调用者一目了然 */
connect("192.168.1.1", 8080);  /* 调用者知道第一个参数是host，第二个是port */
```

**警示**：⚠️ 缺少参数名的坏处：① 调用者需要查阅文档才能理解参数含义；② IDE无法提供参数提示；③ 代码审查时审查者无法判断参数含义；④ 破坏了代码的可读性。

### 规则3.5 函数调用不应作为另一个函数的参数

**写法**：将函数调用结果赋给临时变量后，再作为参数传递。避免嵌套调用导致的可读性下降。

**示例**：
```c
/* 错误：嵌套调用，可读性差 */
result = process(get_data(ctx, KEY_NAME), validate(get_data(ctx, KEY_VAL)));

/* 正确：分解为临时变量 */
const char *name = get_data(ctx, KEY_NAME);
const char *value = get_data(ctx, KEY_VAL);
result = process(name, validate(value));

/* 特别错误：嵌套调用的执行顺序不确定 */
func(a++, b = a++);  /* 未定义行为！a的自增顺序不确定 */
```

**警示**：⚠️ 嵌套调用的主要问题是：① 函数参数的求值顺序在C标准中未明确定义，不同编译器可能产生不同结果；② 错误信息难以定位，不知道哪个函数调用出错；③ 调试时无法检查中间值；④ 代码可读性差，需要从内向外逐层阅读。

### 规则3.6 函数返回值必须校验

**写法**：所有函数返回值（特别是系统调用、库函数）必须检查。如果确实不需要检查，应使用 `(void)` 显式忽略。

**示例**：
```c
/* 错误：忽略返回值 */
fopen("config.ini", "r");          /* 可能打开失败 */
fclose(fp);                        /* 可能关闭失败 */
write(fd, buf, len);               /* 可能未写入全部数据 */
printf("message");                 /* 可能输出失败 */

/* 正确：检查返回值 */
FILE *fp = fopen("config.ini", "r");
if (fp == NULL) {
    log_error("Failed to open config file");
    return ERR_FILE_OPEN;
}

/* 显式忽略（需记录原因） */
(void)close(fd);  /* 只读操作，关闭失败不影响数据一致性 */
```

**警示**：⚠️ 不检查返回值是最常见的"看起来无害"的实践，但实际危害极大：
- 某项目因 `fclose()` 返回错误（磁盘空间不足）但未检查，导致数据丢失
- 某网络服务因 `write()` 返回-1但未检查，导致持续发送错误数据，最终服务崩溃
- **安全检查方法**：使用 `__attribute__((warn_unused_result))` 或编译器的 `-Wunused-result` 选项

### 规则3.7 函数嵌套调用不超过4层

**写法**：过深的嵌套导致代码难以理解和调试。如果嵌套深度超过4层，应重构代码。

**示例**：
```c
/* 错误：嵌套深度6层 */
void process(void) {
    for (...) {                  /* 第1层 */
        while (...) {            /* 第2层 */
            if (...) {           /* 第3层 */
                switch (v) {     /* 第4层 */
                    case A:
                        if (...) {  /* 第5层 （已超标） */
                            /* ... */
                        }
                }
            }
        }
    }
}

/* 正确：提取内部逻辑为函数，降低嵌套深度 */
void process(void) {
    for (...) {
        while (...) {
            handle_item(item);  /* 将内部逻辑提取为函数 */
        }
    }
}
```

**警示**：⚠️ 深度嵌套是复杂代码的典型特征，被称为"箭头代码"（因为代码形状像箭头）。深度嵌套超过4层时，代码的可读性急剧下降，且容易遗漏边界条件。**快速判断**：如果代码需要向右滚动才能看到完整内容，说明嵌套过深。

### 规则3.8 函数内部不应有未使用的参数

**写法**：对于回调函数等必须保留签名的未使用参数，使用 `(void)param;` 显式标记。

**示例**：
```c
/* 正确：显式标记未使用参数 */
void timer_callback(int timer_id, void *user_data)
{
    (void)user_data;  /* 回调函数签名要求，当前未使用 */
    log_info("Timer %d fired", timer_id);
    reset_timer(timer_id);
}

/* 错误：未使用参数没有标记 */
void timer_callback(int timer_id, void *user_data)
{
    /* user_data未使用，编译器会报警告 */
    log_info("Timer %d fired", timer_id);
}
```

**警示**：⚠️ 未使用的参数会导致编译器警告。如果项目开启了 `-Werror`（推荐），未使用的参数会导致编译失败。使用 `(void)param;` 的写法向代码阅读者和编译器明确传达了"我知道这个参数存在，但当前不需要使用它"的信息。

### 建议3.1 指针参数不修改内容时声明为const

**写法**：指针参数如果不修改其指向的内容，必须声明为 `const`。这是自文档化的最佳实践。

**示例**：
```c
/* 好的实践：声明const，明确表示不修改 */
void print_message(const char *message);  /* 不会修改message内容 */
void process_data(const data_t *data, data_t *out_result);  /* data只读，out_result可写 */

/* 不好的实践：不声明const */
void print_message(char *message);  /* 调用者不确定message是否会被修改 */

/* 错误：const指针被用于修改 */
void print_message(const char *message) {
    message[0] = 'A';  /* 编译错误！不能修改const内容 */
}
```

**警示**：⚠️ 不声明 `const` 的危害：① 调用者无法确定函数是否会修改参数；② 如果调用者传入的是 `const` 数据，需要强制类型转换去掉 const，容易导致未定义行为；③ 编译器无法帮助发现误修改的bug。

### 建议3.2 参数顺序：输入参数在前，输出参数在后

**写法**：函数参数按"输入参数 → 输出参数"的顺序排列，使调用者可以快速理解数据流向。

**示例**：
```c
/* 正确：输入参数在前，输出参数在后 */
int parse_config(const char *file_path, config_t *out_config);
int read_sensor(int sensor_id, double *out_value);
int find_user(const char *name, user_info_t *out_user);

/* 错误：输出参数在前 */
int parse_config(config_t *out_config, const char *file_path);  /* 违反直觉 */
```

**警示**：⚠️ 参数顺序混乱会增加调用者的认知负担。调用者需要查阅文档才能确定哪个参数是输入、哪个是输出。**一条经验法则**：函数签名应该像一个句子——主语（输入）在前，宾语（输出）在后。

### 建议3.3 函数名应描述行为

**写法**：函数名应使用"动词+名词"结构，清晰表达函数行为。函数名是代码自文档化的第一要素。

**示例**：
```c
/* 好的函数名 */
int calculate_checksum(const uint8_t *data, size_t len);
int open_serial_port(const char *device_name);
void close_serial_port(int fd);
bool is_valid_email(const char *email);
int find_user_by_id(uint32_t id, user_info_t *out);

/* 不好的函数名 */
int cs(const uint8_t *d, size_t l);      /* 缩写过度，不知所云 */
void port(int fd);                       /* 动词缺失，不知道是open还是close */
int data(const char *input);             /* 太泛，不知道做什么 */
void handle_stuff(void);                 /* "stuff"没有任何信息量 */
```

---

## 4 标识符命名规范

### 4.1 通用命名规则

#### 规则4.1 命名必须清晰明了

**写法**：标识符命名应自注释，使阅读者能通过名称理解其含义，而不需要查看代码逻辑或查阅文档。命名应使用完整的英文单词或广泛接受的缩写。

**示例**：
```c
/* 错误：单字母、无意义命名 */
int a, b, c;
int cnt;
void proc(void);

/* 正确：自注释的命名 */
int age, height, weight;
int item_count;
void process_user_login(void);
```

**警示**：⚠️ 命名不清晰是导致代码难以维护的首要原因。单字母变量只在极短的循环中允许（如 `for (int i = 0; i < n; i++)`）。研究表明，阅读代码时平均每遇到一个含义不清的命名，需要花费 **30秒** 去理解上下文。一个包含50个不清晰命名的函数，将浪费 **25分钟** 的阅读时间。

#### 规则4.2 禁止使用汉语拼音和随意缩写

**写法**：除广泛接受的通用缩写（如 `init`、`config`、`buf`、`len`、`max`、`min`、`cnt`、`tmp`）外，不得使用拼音或随意缩写。通用缩写列表应作为团队规范附录。

**示例**：
```c
/* 错误：拼音和随意缩写 */
int yonghu_count;
int szTemp;
void chl(void);
int tiaoMu(void);  /* 拼音：条目 */

/* 正确 */
int user_count;
int temp_buffer;
void clear_handle(void);
int get_item_count(void);

/* 可接受的通用缩写 */
int msg_id;       /* message缩写 */
int buf_size;     /* buffer缩写 */
const char *cfg;  /* config缩写 */
int cur_pos;      /* current position */
```

**警示**：⚠️ 拼音命名是中国开发团队的特有问题。其危害在于：① 非中文背景的开发者完全无法理解；② 拼音可能存在多音字歧义（如"行"对应hang/xing）；③ 拼音与英文混用导致命名风格不统一。**经验法则**：如果不知道一个英文单词，查字典，不要用拼音代替。

#### 规则4.3 命名长度适中

**写法**：命名应在清晰表达含义的前提下保持简洁。过长或过短的命名都应避免。

**示例**：
```c
/* 错误：过短 */
int n;           /* 不清晰 */
void h(void);    /* 不清晰 */

/* 错误：过长 */
int count_of_current_active_online_users;  /* 太啰嗦 */
int process_user_authentication_and_authorization(void);  /* 太长 */

/* 正确：适中 */
int active_user_count;
void authenticate_user(void);
```

**警示**：⚠️ 命名长度与可读性呈倒U型关系——过短不清晰，过长同样影响阅读。变量名通常 **2-3个单词** 足够，函数名 **3-5个单词** 足够。如果命名需要超过这个长度，往往是模块划分不合理导致的。

#### 规则4.4 使用正确的反义词命名

**写法**：成对出现的操作应使用标准的反义词组，保持对称性。使用一致的反义词组可以形成心理模式，减少阅读负担。

**示例**：

| 正 | 反 | 使用场景 |
|---|----|---------|
| begin / start | end / stop | 生命周期管理 |
| create | destroy | 资源创建/销毁 |
| open | close | 文件/设备操作 |
| lock | unlock | 互斥操作 |
| get | set / put | 属性访问 |
| read | write | I/O操作 |
| source | sink | 数据流 |
| push | push | 栈操作 |
| send | recv | 网络通信 |
| enable | disable | 功能开关 |
| insert | delete | 容器操作 |
| add | remove | 集合操作 |
| first | last | 范围/迭代 |
| min | max | 极值 |
| show | hide | 可见性控制 |

**警示**：⚠️ 反义词使用混乱是常见问题。例如同一个模块中同时使用 `open/close` 和 `start/stop` 表示相同的操作含义，会让调用者困惑。每个模块应统一使用一套反义词约定。

### 4.2 文件命名规则

#### 规则4.5 文件名使用小写字母+下划线

**写法**：文件名使用全小写字母和下划线，不使用大写字母、驼峰命名或连字符。这与C语言本身不区分大小写的文件系统（如Windows）兼容，也避免了大小写敏感的文件系统（如Linux）上的混淆。

**示例**：
```c
/* 正确 */
tcp_client.c
message_queue.h
crc32_checksum.c

/* 错误 */
TCPClient.c
MessageQueue.h
CRC32.c
```

**警示**：⚠️ Windows和Linux对文件名大小写的处理方式不同。在Windows上，`TCPClient.c` 和 `tcpclient.c` 指向同一个文件，但在Linux上是两个不同的文件。使用全小写+下划线可以避免跨平台时的文件名冲突问题。

### 4.3 变量命名规则

#### 规则4.6 全局变量加`g_`前缀

**写法**：全局变量（模块间可见的变量）使用 `g_` 前缀，使其在代码中一目了然。全局变量应尽量少用，优先通过接口函数访问。

**示例**：
```c
/* 正确：全局变量加g_前缀 */
int g_system_running = 0;
uint32_t g_uptime_seconds = 0;

/* 错误：全局变量没有前缀 */
int system_running = 0;  /* 无法区分是全局还是局部 */

/* 规范：使用全局变量前先问自己——真的需要全局变量吗？ */
```

**警示**：⚠️ 全局变量是万恶之源。`g_` 前缀不仅是一种命名约定，更是一种提醒——看到 `g_` 前缀就应该意识到这是一个需要谨慎使用的全局状态。过度使用全局变量是导致代码不可测试、不可并发的首要原因。

#### 规则4.7 静态变量加`s_`前缀

**写法**：文件作用域的 `static` 变量使用 `s_` 前缀，表示该变量是模块内部私有的。

**示例**：
```c
/* 正确：静态变量加s_前缀 */
static int s_instance_count = 0;
static const char *s_version_string = "v1.0.0";

/* 函数内部的静态变量也使用s_前缀 */
void get_next_id(void) {
    static int s_next_id = 0;  /* 注意：函数级静态变量也应加s_前缀 */
    return s_next_id++;
}
```

**警示**：⚠️ 静态变量虽然限制了作用域，但仍具有全局生命周期。使用 `s_` 前缀用于区分：① 与普通局部变量区分；② 与全局变量区分（`g_` vs `s_`）；③ 提醒开发者这个变量在函数调用之间保持状态，注意线程安全性。

#### 规则4.8 局部变量不加前缀

**写法**：函数内部的局部变量不加任何前缀，使用简洁明了的名称。

**示例**：
```c
void process_data(void)
{
    int count = 0;           /* 局部变量，无前缀 */
    char buffer[256];        /* 局部变量，无前缀 */
    const char *name = ...;  /* 局部变量，无前缀 */
}
```

**警示**：⚠️ 局部变量不要加 `g_` 或 `s_` 前缀。如果发现一个局部变量需要加前缀才能区分，说明函数太长或命名不够清晰。局部变量一旦离开函数体就不存在了，不需要作用域标记。

#### 规则4.9 线程局部变量加`t_`前缀（C11）

**写法**：C11标准的 `_Thread_local` 变量使用 `t_` 前缀，表示该变量是线程私有的，每个线程有独立的副本。

**示例**：
```c
/* 正确：线程局部变量加t_前缀 */
_Thread_local int t_errno = 0;
_Thread_local char t_errmsg[256] = {0};
```

**警示**：⚠️ 线程局部变量看似解决了并发问题，但如果使用不当（如存储指向共享数据的指针），仍可能导致竞态条件。`t_` 前缀提醒开发者这个变量是线程局部的，但**不意味着**可以随意访问共享资源。

### 4.4 函数命名规则

#### 规则4.10 函数名使用小写字母+下划线

**写法**：函数名使用全小写字母和下划线分隔，不使用驼峰命名。动词在前，名词在后，形成"操作+对象"的结构。

**示例**：
```c
/* 正确 */
int serial_port_open(const char *port_name);
void serial_port_close(int fd);
int get_user_count(void);
bool is_valid_email(const char *email);

/* 错误 */
int SerialPortOpen(const char *portName);
int GetUserCount(void);
bool isValidEmail(const char *email);
```

**警示**：⚠️ 函数命名风格的不一致是团队协作中最常见的争议点。本规范选择"小写+下划线"风格，因为这符合C语言标准库的命名传统（如 `printf`、`fopen`、`strcpy`），与C++的区分更清晰，且在Linux生态中广泛使用。

### 4.5 类型命名规则

#### 规则4.11 自定义类型加`_t`后缀

**写法**：所有通过 `typedef` 定义的类型（包括结构体、联合体、枚举、函数指针）都应加 `_t` 后缀，以区分类型名和变量名。

**示例**：
```c
/* 正确 */
typedef struct { uint32_t id; char name[64]; } user_info_t;
typedef enum { STATE_IDLE, STATE_RUNNING, STATE_ERROR } system_state_e;
typedef int (*compare_fn_t)(const void *, const void *);

/* 错误：不加_t后缀 */
typedef struct { uint32_t id; } UserInfo;  /* 风格也不一致 */

/* 使用 */
user_info_t user;
system_state_e state = STATE_IDLE;
compare_fn_t cmp = compare_by_id;
```

**警示**：⚠️ 注意：POSIX标准保留 `_t` 后缀，严格来说自定义类型加 `_t` 可能与POSIX冲突。但在嵌入式和非POSIX平台（如Windows、RTOS）上这通常不是问题。如果项目需要严格遵循POSIX，可以使用 `_s`（结构体）、`_e`（枚举）、`_fn`（函数指针）等替代后缀。

#### 规则4.12 结构体使用`_t`后缀，枚举使用`_e`后缀

**写法**：结构体类型使用 `_t` 后缀，枚举类型使用 `_e` 后缀，函数指针类型使用 `_fn_t` 后缀，便于区分不同类型。

**示例**：
```c
typedef struct { int x; int y; } point_t;        /* 结构体：_t */
typedef enum { COLOR_RED, COLOR_GREEN, COLOR_BLUE } color_e;  /* 枚举：_e */
typedef int (*sort_fn_t)(const void *, const void *);  /* 函数指针：_fn_t */

/* 联合体使用_t后缀 */
typedef union { uint32_t word; uint8_t bytes[4]; } word32_t;
```

**警示**：⚠️ 类型后缀的一致性对于代码可读性至关重要。在审查代码时，看到 `_t` 就知道这是一个类型（可以用于声明变量），看到 `_e` 就知道这是一个枚举（值是有限的常量集合）。如果不加后缀，阅读者无法区分 `point` 是一个类型还是一个变量。

### 4.6 宏命名规则

#### 规则4.13 宏名使用全大写+下划线

**写法**：宏名称使用全大写字母和下划线分隔。这是C语言中延续了几十年的传统，使宏在所有代码中脱颖而出。

**示例**：
```c
/* 正确 */
#define MAX_BUFFER_SIZE  4096
#define VERSION_MAJOR    2
#define VERSION_MINOR    1

/* 错误：使用小写或驼峰 */
#define maxBufferSize  4096
#define versionMajor   2
```

**警示**：⚠️ 宏名全大写是C语言中为数不多的"视觉约定"之一。违反这个约定可能导致严重后果：如果一个宏名与函数名或变量名冲突，预处理器会无声地替换掉标识符，导致难以调试的编译错误或运行时错误。**记住**：宏不遵循C语言的作用域规则，它们在任何地方都有效。

#### 规则4.14 函数式宏参数使用小写

**写法**：函数式宏的参数使用小写，与宏名的大写形成对比，清晰区分宏名和参数名。

**示例**：
```c
/* 正确：宏名大写，参数小写 */
#define MAX(a, b)  ((a) > (b) ? (a) : (b))
#define IS_VALID(n)  ((n) >= 0 && (n) < MAX_COUNT)
#define ABS(x)     ((x) < 0 ? -(x) : (x))

/* 警告：参数必须用括号包裹 */
#define SQUARE(x)  ((x) * (x))   /* 正确：外层括号 */
/* 错误：SQUARE(1+2) 扩展为 (1+2*1+2) = 5，不是9 */
```

**警示**：⚠️ 函数式宏容易产生隐蔽的错误：① 参数必须用括号包裹，防止运算符优先级问题；② 多次求值的参数可能导致错误（如 `MAX(++a, b)` 中 `a` 会被自增两次）；③ 对于复杂逻辑，优先使用内联函数而不是宏。

---

## 5 排版与格式规范

### 规则5.1 缩进使用4个空格

**写法**：禁止使用Tab字符缩进。编辑器应配置为将Tab自动转换为4个空格。缩进表示代码块的嵌套层次，每嵌套一层增加一个缩进级别。

**示例**：
```c
/* 正确：4空格缩进 */
void func(void)
{
····if (condition) {
········do_something();
····}
}

/* 注意：这里的"····"表示4个空格，不是Tab */
```

**警示**：⚠️ Tab在不同编辑器中的显示宽度不同（2个字符到8个字符不等），导致代码在不同环境下显示错乱。使用空格可以确保代码在所有编辑器中显示一致。**配置方法**：在 `.editorconfig` 文件中设置 `indent_style = space` 和 `indent_size = 4`。

### 规则5.2 程序块缩进风格

**写法**：函数定义使用 **Allman风格**（左大括号另起一行），控制语句使用 **K&R风格**（左大括号不另起一行）。这种混合风格既保持了函数定义的清晰性，又节省了控制语句的行数。

**示例**：
```c
/* 函数定义：左大括号另起一行（Allman风格） */
int calculate_total(int a, int b)
{
    return a + b;
}

/* 控制语句：左大括号不另起一行（K&R风格） */
if (condition) {
    do_something();
} else {
    do_other_thing();
}

while (condition) {
    process();
}

/* 错误：风格混搭 */
int calculate_total(int a, int b) {  /* 函数定义使用K&R风格 */
    return a + b;
}

if (condition)                         /* 控制语句使用Allman风格 */
{
    do_something();
}
```

**警示**：⚠️ 大括号放置风格是编程语言中最古老的"圣战"之一。本规范采用混合风格的理由：① 函数定义使用Allman风格，使函数签名与函数体清晰分离，便于快速定位函数边界；② 控制语句使用K&R风格，节省行数，减少代码滚动。**关键**：无论选择哪种风格，在整个项目中必须保持一致。

### 规则5.3 一行只写一条语句

**写法**：每行最多包含一条可执行语句。多个短语句放在同一行会降低可读性，也无法设置断点进行调试。

**示例**：
```c
/* 错误 */
a = 1; b = 2; c = 3;

/* 正确 */
a = 1;
b = 2;
c = 3;
```

**警示**：⚠️ 一行多条语句的主要问题：① 调试时无法在单条语句上设置断点；② 代码覆盖率工具无法精确统计单条语句的执行情况；③ 异常发生时，无法确定是哪个语句出错；④ 代码可读性差，增加了阅读负担。

### 规则5.4 if/for/while/do等控制语句独占一行

**写法**：控制语句（`if`、`for`、`while`、`do`）必须独占一行，即使只有一条语句也必须使用大括号。

**示例**：
```c
/* 错误：控制语句与执行语句在同一行 */
if (condition) do_something();

/* 错误：单条语句不加括号 */
if (condition)
    do_something();  /* 如果后续添加第二个语句，容易忘记加括号 */

/* 正确：控制语句独占一行，加括号 */
if (condition) {
    do_something();
}
```

**警示**：⚠️ 单条语句不加括号是著名的"Apple SSL bug"的根本原因（CVE-2014-1266）。当有人在代码审查时添加了第二个语句，但忘记加括号，导致安全认证被绕过。**永远不要**省略控制语句的大括号，即使只有一条语句。

### 规则5.5 运算符前后加空格

**写法**：二元运算符前后各加一个空格，一元运算符与操作数之间不加空格。

**示例**：
```c
/* 错误 */
int sum=a+b*c;
int result= a>0? a: -b;

/* 正确 */
int sum = a + b * c;
int result = a > 0 ? a : -b;

/* 一元运算符不加空格 */
int negative = -a;
int incremented = ++count;
```

**警示**：⚠️ 运算符前后不加空格会导致代码拥挤，难以快速识别表达式的结构。特别是在复杂表达式中，空格可以帮助读者快速识别运算符的优先级关系。但注意：一元运算符（如 `++`、`--`、`-`、`!`、`~`）与操作数之间不加空格。

### 规则5.6 逗号后加空格

**写法**：逗号后面加一个空格，逗号前面不加空格。这符合英文书写的标点符号规则。

**示例**：
```c
/* 错误 */
function(a,b,c);
int arr[] = {1,2,3,4,5};

/* 正确 */
function(a, b, c);
int arr[] = {1, 2, 3, 4, 5};
```

**警示**：⚠️ 逗号前后空格不一致是排版不专业的表现。虽然不会导致编译错误，但会影响代码审查的专业形象。**一条经验法则**：在C语言中，逗号的使用方式与英文书写一致——逗号后有一个空格，逗号前没有。

### 规则5.7 括号内侧不加空格

**写法**：左括号后和右括号前不加空格。括号内的内容应与括号紧密相连。

**示例**：
```c
/* 错误 */
if ( condition ) {
    function( a, b );
}

/* 正确 */
if (condition) {
    function(a, b);
}

/* 特别：函数调用时括号前不加空格 */
/* 正确 */
function(a, b);
/* 错误 */
function (a, b);  /* 括号前有空格，与函数名分开 */
```

**警示**：⚠️ 括号内侧加空格是一种"视觉噪音"，会分散阅读者的注意力。在函数调用时，函数名和左括号之间不应有空格（这有助于区分函数调用和关键字）。

### 规则5.8 行宽控制

**写法**：每行代码不超过 **120个字符**。超过时应在操作符后换行，并使用缩进对齐。

**示例**：
```c
/* 正确：在操作符处换行，对齐 */
int result = very_long_function_name(param1, param2, param3,
                                     param4, param5);

/* 正确：在函数调用参数处换行 */
if (very_long_condition(param1, param2, param3) &&
    another_condition(param4, param5)) {
    handle_condition();
}

/* 错误：超过120字符 */
int result = very_long_function_name(param1, param2, param3, param4, param5, param6);
```

**警示**：⚠️ 过长的行需要水平滚动才能阅读，极大地降低了代码的可读性。研究表明，最佳阅读宽度在 **80-120字符** 之间。如果一行代码超过120字符还无法自然地换行，说明代码过于复杂，应重构。

### 规则5.9 空行使用

**写法**：空行用于分隔不同逻辑块，提高代码的可读性。空行数量应适中，过多或过少都会影响阅读。

**示例**：
```c
/* 函数之间空2行 */
void func1(void)
{
    return;
}


void func2(void)
{
    return;
}

/* 同一文件中逻辑块之间空1行 */
void process(void)
{
    int result = 0;

    /* 初始化阶段 */
    init_module_a();
    init_module_b();

    /* 处理阶段 */
    result = process_data();
    if (result != 0) {
        log_error("Processing failed");
        return;
    }
}
```

**警示**：⚠️ 空行使用的常见问题：① 不空行：代码块之间没有分隔，读起来像一段"墙"；② 空行过多：代码像"填空"，有效信息密度太低；③ 空行随意：没有逻辑意义，让人无法通过空行理解代码结构。**经验法则**：空行应该像文章中的段落分隔——表示逻辑的切换。

### 规则5.10 switch-case缩进

**写法**：`case` 与 `switch` 同缩进级别，`case` 体内的代码再缩进一级。每个 `case` 必须以 `break` 或 `return` 结束。

**示例**：
```c
switch (value) {
    case OPTION_A:
        handle_option_a();
        break;
    case OPTION_B:
        handle_option_b();
        break;
    default:
        handle_default();
        break;
}

/* 错误：case缩进不正确 */
switch (value) {
    case OPTION_A:        /* 与switch相同缩进 */
        {
            /* ... */    /* case体内的代码多缩进一级 */
        }
        break;
}
```

**警示**：⚠️ `switch-case` 中的 `fall-through`（无 `break` 的case）是C语言中bug的高发区。如果确实需要 `fall-through`，必须在代码中显式注释 `/* fall-through */`。大多数编译器在 `-Wimplicit-fallthrough` 选项下会对此发出警告。

### 规则5.11 指针声明中`*`的位置

`*` 应靠近变量名，而非类型名：

```c
/* 正确 */
int *ptr;
int *ptr1, *ptr2;
void *data;

/* 错误 */
int* ptr;
int* ptr1, ptr2;  /* ptr2是int，不是int* */
```

---

## 6 注释规范

### 原则6.1 优秀的代码可以自我解释

**写法**：代码本身应清晰到不需要过多注释。注释应解释"为什么"，而非"是什么"。好的注释说明代码的意图、约束和背景，而不是重复代码本身。

**示例**：
```c
/* 坏的注释：解释"是什么"（代码本身已经表达了） */
/* 计数器加1 */
count++;

/* 好的注释：解释"为什么"（代码无法表达的背景信息） */
/* 跳过BOM头（UTF-8 BOM为0xEF 0xBB 0xBF），否则XML解析器会报错 */
if (buf[0] == 0xEF && buf[1] == 0xBB && buf[2] == 0xBF) {
    offset += 3;
}

/* 极其糟糕的注释 */
/* 2024-01-15 张三修改：修复了bug */
/* 这种修改日志应该由版本控制系统管理，而不是注释！ */
```

**警示**：⚠️ 注释的质量比数量更重要。**坏注释比没有注释更有害**，因为它会误导阅读者。以下情况不需要注释：① 代码本身已经清晰的（如 `i++` 不需要注释"自增"）；② 修改日志（由版本控制系统管理）；③ 显而易见的实现细节。**经验法则**：如果注释只说代码已经表达的内容，删掉它。

### 规则6.1 文件头注释

**写法**：每个源文件和头文件开头必须包含文件头注释，说明文件的基本信息。文件头注释使用 `/** */` Doxygen格式。

**示例**：
```c
/**
 * @file    tcp_client.c
 * @brief   TCP客户端模块实现
 * @details 提供TCP连接的创建、发送、接收和关闭功能。
 *          支持IPv4和IPv6，支持超时重连。
 * @author  张三
 * @date    2024-01-15
 * @version v1.2.0
 */
```

**警示**：⚠️ 文件头注释中的 `@date` 和 `@version` 应在每次修改文件时更新。但要注意：**不要记录详细的修改历史**——这些信息由版本控制系统（Git、SVN）管理。文件头注释只应包含当前版本的状态信息。

### 规则6.2 函数注释

**写法**：每个对外公开的函数必须有注释，内部函数建议添加注释。使用Doxygen格式，包含 `@brief`、`@param`、`@return`、`@retval` 等标签。

**示例**：
```c
/**
 * @brief 建立TCP连接
 * @param host      目标主机名或IP地址，不能为NULL
 * @param port      目标端口号（1-65535）
 * @param timeout   连接超时时间（毫秒，0表示使用默认值）
 * @return 成功返回文件描述符(>=0)，失败返回负值错误码
 * @retval 0         成功
 * @retval -EINVAL   参数无效（host为NULL或port超出范围）
 * @retval -ETIMEDOUT 连接超时（超过timeout毫秒未连接成功）
 * @retval -ECONNREFUSED 连接被目标拒绝
 * @note 本函数阻塞直到连接建立或超时，不建议在事件循环中直接调用
 * @see tcp_close(), tcp_send()
 */
int tcp_connect(const char *host, uint16_t port, uint32_t timeout);
```

**警示**：⚠️ 函数注释的几个常见问题：① `@param` 描述缺少约束条件（如"不能为NULL"）；② 缺少 `@retval` 说明特定错误码的含义；③ 注释与实现不一致（修改参数后忘记更新注释）。**检查方法**：使用Doxygen工具生成文档，检查是否有遗漏的 `@param` 或不匹配的签名。

### 规则6.3 全局变量注释

**写法**：全局变量必须有注释，说明其用途、初始值和使用约束（如线程安全性）。

**示例**：
```c
/** 系统运行状态标志，0=停止，1=运行。仅由主循环线程写入，其他线程只读 */
volatile int g_system_running = 0;

/** 全局配置实例，初始化后只读，无需加锁 */
config_t g_app_config;

/** 全局日志级别，可通过运行时配置接口修改，修改需加锁 */
log_level_t g_log_level = LOG_LEVEL_INFO;
```

**警示**：⚠️ 全局变量注释必须说明其**线程安全性**和**访问权限**。这是代码审查中经常被遗漏的关键信息。没有这些信息，后续维护者可能错误地访问全局变量，引入竞态条件。

### 规则6.4 修改代码时必须同时修改注释

**写法**：修改代码后，必须同步更新相关注释。过时的注释比没有注释更危险，因为它会误导阅读者。

**示例**：
```c
/* 错误：修改了代码但未更新注释 */
/* 函数返回：0=成功，1=失败 */
int process_data(void) {
    /* 后来重构为返回错误码，但注释没更新！ */
    return -EINVAL;  /* 实际返回负值错误码 */
}

/* 正确：同步更新注释 */
/* 函数返回：0=成功，负值=错误码（-EINVAL表示参数无效） */
int process_data(void) {
    return -EINVAL;
}
```

**警示**：⚠️ 过时的注释是"代码味道"的典型表现。**在代码审查中，如果发现注释与代码不一致，审阅者应该假设注释是错误的**。一个实际的案例：某项目因注释说"该函数线程安全"但实际不是，导致线上数据损坏。

### 规则6.5 注释格式统一

**写法**：单行注释使用 `/* ... */` 或 `// ...`，多行注释使用 `/** ... */`（Doxygen风格）。行尾注释只在变量声明等简单场景使用。

**示例**：
```c
/* 单行注释 */
int count = 0;  /* 行尾注释：仅用于简单说明 */

/**
 * 多行注释（Doxygen风格）
 * 用于函数、类型、文件等复杂场景
 */

// C++风格注释在C99+中也可使用，但项目中应统一使用一种风格
```

**警示**：⚠️ 在同一项目中混用多种注释风格会导致视觉混乱。建议使用 `/* */` 作为标准注释，`// ` 用于临时注释代码（因为可以嵌套宏条件编译）。

### 建议6.1 避免在注释中使用缩写

**写法**：注释中的缩写应限制在通用缩写范围内，避免使用团队内部缩写或项目特定缩写。

**示例**：
```c
/* 错误：使用内部缩写 */
/* 通过UART发送数据，TX完成时置位FLG */

/* 正确：使用完整单词 */
/* 通过串口发送数据，发送完成时置位完成标志 */

/* 如果必须使用缩写，首次出现时给出全称 */
/* 通过通用异步收发器（UART）发送数据 */
```

**警示**：⚠️ 注释中的缩写是新人入职理解代码的最大障碍之一。新成员可能需要花数周时间才能记住所有内部缩写。如果缩写必须使用，应当在项目文档中维护一个"缩写对照表"。

### 建议6.2 TODO/FIXME标记

**写法**：使用 `TODO`、`FIXME`、`HACK` 等标记注释尚未完成或需要修复的代码。这些标记应包含责任人和时间信息。

**示例**：
```c
/* TODO: 后续需要添加超时重传机制（张三，2024-Q2） */
/* FIXME: 此处存在内存泄漏，需要在错误路径释放buffer（李四，2024-01） */
/* HACK: 此处的临时方案仅用于兼容旧版协议，v3.0协议发布后需移除 */
/* XXX: 以下代码逻辑复杂，建议在下次重构时重新设计 */
```

**警示**：⚠️ `TODO` 和 `FIXME` 标记是技术债务的表现形式。应定期（如每个迭代结束前）清理这些标记——要么完成，要么创建正式的任务单跟踪。**不要让 TODO 变成 FOREVER**。

### 建议6.3 注释的Doxygen标签规范

**写法**：使用统一的Doxygen标签，确保文档生成的一致性。

**示例**：

| 标签 | 使用场景 | 说明 |
|------|---------|------|
| `@brief` | 函数/类型/宏 | 简短描述（必选） |
| `@details` | 函数/类型 | 详细描述（可选） |
| `@param` | 函数参数 | 参数名 + 描述，按声明顺序 |
| `@return` | 函数 | 返回值描述 |
| `@retval` | 函数 | 特定返回值的含义 |
| `@note` | 任何 | 需要注意的事项 |
| `@warning` | 任何 | 警告信息 |
| `@see` | 任何 | 参考链接 |
| `@deprecated` | 函数/类型 | 标记为废弃及替代方案 |

**警示**：⚠️ Doxygen标签的完整性和正确性可以通过CI流水线检查。建议在CI中配置Doxygen检查，如果 `@param` 与函数签名不匹配，视为构建警告。

---

## 7 变量规范

### 规则7.1 禁止使用未经初始化的变量

**写法**：所有变量在声明时必须初始化。未初始化的变量包含不确定的值（栈上的垃圾数据），使用它们会导致未定义行为。

**示例**：
```c
/* 错误 */
int count;
printf("%d", count);  /* 未定义行为：输出垃圾值 */

/* 正确 */
int count = 0;

/* 指针变量必须初始化 */
int *ptr = NULL;  /* 而不是 int *ptr; */

/* 数组初始化 */
int arr[10] = {0};  /* 所有元素初始化为0 */
```

**警示**：⚠️ 使用未初始化的变量是C语言中最常见、最隐蔽的错误之一。**在Debug模式下，某些编译器（如MSVC）会将未初始化的栈变量填充为特定模式（0xCC），使问题更容易发现。但在Release模式下，这些模式消失，可能表现为随机崩溃**。**静态检查工具**：`cppcheck`、`clang-tidy`、`Coverity` 都能检测未初始化变量。

### 规则7.2 防止局部变量与全局变量同名

**写法**：局部变量不应与全局变量同名。同名会导致局部变量"隐藏"全局变量，造成混淆和潜在的错误。

**示例**：
```c
int g_count = 0;  /* 全局变量 */

void function(void)
{
    int count = 10;  /* 隐藏了全局变量，容易混淆 */
    count++;  /* 修改的是局部变量，不是全局变量 */
    
    /* 正确：使用不同的局部变量名 */
    int local_count = 10;
    int item_count = 10;
}

/* 错误：不小心使用了全局变量同名 */
int g_system_running = 1;
void stop_system(void) {
    int g_system_running = 0;  /* 创建了一个局部变量，而不是修改全局变量！ */
    /* 实际上全局变量 g_system_running 仍然是 1 */
}
```

**警示**：⚠️ 局部变量隐藏全局变量是C语言中"静默错误"的典型例子。编译器可能会发出 `-Wshadow` 警告，但默认情况下这个警告是关闭的。**建议在编译选项中启用 `-Wshadow`（GCC/Clang）或 `/w44602`（MSVC）**，将同名隐藏视为警告。

### 规则7.3 严格控制全局变量使用

**写法**：全局变量增加模块间耦合，降低可测试性，应尽量少用。使用全局变量必须满足以下条件之一。

**示例**：
```c
/* 可接受的全局变量使用场景 */

/* 1. 系统全局状态 */
volatile int g_system_running = 0;  /* 系统运行/停止标志 */

/* 2. 全局唯一实例的配置 */
const char *g_app_name = "MyApp";   /* 只读全局配置 */

/* 3. 性能关键路径上的缓存（需评估后使用） */
static lookup_cache_t g_cache;      /* 注意：使用static限制作用域 */

/* 不可接受的全局变量 */
/* 错误：模块间通信通过全局变量 */
int g_user_login_flag;  /* 应该在模块内部管理，通过接口访问 */
```

**警示**：⚠️ 全局变量是"隐式耦合"的根源。使用全局变量时，需要考虑：① 多线程安全性（是否需要加锁）；② 初始化顺序（C语言不保证不同文件中全局变量的初始化顺序）；③ 可测试性（测试用例需要重置全局状态）。**全局变量每增加一个，代码的测试难度就增加一分**。

### 规则7.4 变量声明靠近使用处

**写法**：变量声明应尽可能靠近第一次使用的地方，而不是统一放在函数开头。这有助于读者理解变量的用途，并减少变量作用域。

**示例**：
```c
/* 错误：变量声明远离使用处 */
void process(void)
{
    int result;           /* 与使用处相隔50行 */
    /* ... 50行代码，可能不小心修改了result ... */
    result = compute();   /* 需要回看声明才能理解result的用途 */
}

/* 正确：变量声明靠近使用处 */
void process(void)
{
    /* ... 其他代码 ... */
    int result = compute();  /* 声明即使用，一目了然 */
}
```

**警示**：⚠️ 将变量统一放在函数开头是C89标准的要求（变量声明必须在语句之前），但C99及以后版本已经取消了这一限制。**如果项目使用C99+，应将变量声明放在使用处附近**。这遵循了"最小作用域"原则，减少变量被误修改的风险。

### 规则7.5 指针变量声明时必须初始化

**写法**：声明指针变量时必须初始化，要么赋值为有效的地址，要么赋值为 `NULL`。

**示例**：
```c
/* 错误：指针未初始化 */
int *ptr;
*ptr = 10;  /* 未定义行为：ptr指向随机地址 */

/* 正确：初始化为NULL */
int *ptr = NULL;

/* 正确：初始化为有效地址 */
int value = 10;
int *ptr = &value;

/* 正确：立即分配内存 */
int *ptr = (int *)malloc(sizeof(int) * 10);
if (ptr == NULL) {
    /* 处理内存分配失败 */
}
```

**警示**：⚠️ 未初始化的指针被称为"野指针"，是C语言中最危险的问题之一。使用野指针可能导致：① 段错误（程序崩溃）；② 内存数据损坏（静默写入错误地址）；③ 安全漏洞（指针指向敏感数据区）。**防御措施**：在释放指针后立即设置为 `NULL`，防止悬空指针。

### 规则7.6 局部变量不宜过大

**写法**：局部变量在栈上分配，单个函数局部变量总大小不应超过 **0x1000字节（4KB）**。大缓冲区应使用动态内存分配或静态数组。

**示例**：
```c
/* 错误：栈上分配过大的缓冲区 */
void process_file(void)
{
    char buffer[1024 * 1024];  /* 1MB栈空间！可能导致栈溢出 */
    /* ... */
}

/* 正确：使用动态内存分配 */
void process_file(void)
{
    char *buffer = (char *)malloc(1024 * 1024);
    if (buffer == NULL) {
        /* 处理内存分配失败 */
        return;
    }
    /* ... */
    free(buffer);
}

/* 正确：使用静态数组（注意线程安全性） */
static char s_buffer[1024 * 1024];
```

**警示**：⚠️ 栈溢出是嵌入式开发和资源受限环境中最常见的崩溃原因之一。默认栈大小因平台而异：Linux通常为8MB，Windows为1MB，嵌入式系统可能只有 **1KB-4KB**。**在嵌入式环境中，局部变量总大小不应超过栈大小的 1/4**。**检测方法**：使用编译器的 `-Wframe-larger-than=` 选项可以检测栈帧过大的函数。

### 规则7.7 布尔变量不应与TRUE/FALSE直接比较

**写法**：布尔变量（或用作布尔值的整数）应直接做真值判断，而不是与 `TRUE`、`FALSE`、`1`、`0` 等常量比较。

**示例**：
```c
/* 错误 */
if (flag == TRUE) { ... }
if (flag == FALSE) { ... }

/* 正确 */
if (flag) { ... }
if (!flag) { ... }
```

**警示**：⚠️ 在C语言中，任何非零值都被视为"真"。直接与 `TRUE` 比较的问题是：如果 `TRUE` 定义为 `1`，但某个函数返回 `2`（也代表真），则 `flag == TRUE` 的判断会失败。**在C99+中，`_Bool` 类型只能存储0或1，但与其他类型交互时仍需注意**。

### 规则7.8 避免对浮点数直接做相等比较

**写法**：由于浮点数的精度限制，直接做相等比较（`==`）通常不可靠。应使用绝对值差值与容差（epsilon）比较。

**示例**：
```c
/* 错误：直接比较浮点数 */
if (a == b) { ... }  /* 由于精度问题，即使数学上相等，a==b也可能为false */

/* 正确：使用容差比较 */
#define EPSILON 1e-6

if (fabs(a - b) < EPSILON) { ... }     /* 绝对误差比较 */
if (fabs(a - b) / fabs(a) < EPSILON) { ... }  /* 相对误差比较（推荐） */
```

**警示**：⚠️ 浮点数比较问题是计算机科学中的经典陷阱。根本原因在于：**很多十进制小数无法用二进制精确表示**（如 `0.1` 在二进制中是无限循环小数）。看似简单的 `0.1 + 0.2 == 0.3` 在浮点数计算中可能为 `false`。**经验法则**：永远不要使用 `==` 比较浮点数，除非你非常清楚自己在做什么。
/* 错误 */
if (f == 0.0) { ... }

/* 正确 */
#define EPSILON  1e-6
if (fabs(f) < EPSILON) { ... }
```

---

## 8 宏与常量规范

### 规则8.1 宏定义必须使用完备的括号

**写法**：宏定义中的参数和整个表达式都必须用括号包裹，确保展开后的运算优先级符合预期。

**示例**：
```c
/* 错误：缺少括号，运算符优先级错误 */
#define MUL(a, b)  a * b
/* MUL(1+2, 3+4) 被展开为 1+2*3+4 = 11，期望值21 */

/* 正确：参数和表达式都加括号 */
#define MUL(a, b)  ((a) * (b))
/* MUL(1+2, 3+4) 被展开为 ((1+2) * (3+4)) = 21 */
```

**警示**：⚠️ 宏是文本替换，不是函数调用。缺少括号的宏在复杂表达式中会产生难以发现的优先级错误。**经验法则**：宏定义中，每个参数都要加括号，整个表达式也要加括号。唯一例外是单个标识符或已经加括号的表达式。

### 规则8.2 多条语句的宏必须放在do-while中

**写法**：多条语句的宏必须使用 `do { ... } while (0)` 包裹，使宏像一个独立的语句块，在 `if`/`else` 等控制语句中也能正确工作。

**示例**：
```c
/* 错误 */
#define LOG_ERROR(msg) \
    printf("[ERROR] %s\n", msg); \
    log_to_file(msg);

/* 正确 */
#define LOG_ERROR(msg) \
    do { \
        printf("[ERROR] %s\n", msg); \
        log_to_file(msg); \
    } while (0)

/* 错误场景：if语句中 */
if (error) {
    LOG_ERROR("error");  /* 展开后只有 printf 在 if 中，log_to_file 不在 if 中 */
}
```

**警示**：⚠️ 不使用 `do { } while (0)` 包裹的多语句宏，在 `if`/`else` 中调用时会产生逻辑错误。`do { } while (0)` 不会产生任何运行时开销（编译器会优化掉），它只是一个语法技巧。

### 规则8.3 宏定义不能使用改变控制流的语句

**写法**：禁止在宏中使用 `return`、`break`、`continue`、`goto`，这些语句会使宏的调用者无法预期控制流的变化。

**示例**：
```c
/* 错误：宏中使用了return */
#define CHECK_NULL(ptr) \
    if ((ptr) == NULL) { \
        return ERR_NULL_PTR; \
    }

/* 使用场景：CHECK_NULL(ptr) 会从调用函数中返回，但代码上看不出来！ */

/* 正确：使用函数替代 */
int check_null(const void *ptr)
{
    if (ptr == NULL) {
        return ERR_NULL_PTR;
    }
    return 0;
}
```

**警示**：⚠️ 宏中隐藏控制流语句是最危险的宏设计错误之一。调用者从代码表面看不出这个宏会导致函数返回，违反了"最小惊讶原则"（Principle of Least Astonishment）。**最关键的是**：这种宏在代码审查中很难被发现，因为宏调用看起来像普通的函数调用。

### 规则8.4 宏参数不能发生变化

**写法**：宏参数在宏体中不能出现多次求值，否则带副作用的参数（如 `++x`、`func()`）会导致不可预期的结果。

**示例**：
```c
/* 错误：宏参数有副作用 */
#define MAX(a, b)  ((a) > (b) ? (a) : (b))
int x = 5;
int y = 10;
int z = MAX(x++, y++);  /* x被加两次，y被加一次 */

/* 展开结果：((x++) > (y++)) ? (x++) : (y++) */
/* x=6 (条件判断时自增1), y=11 (条件判断时自增1) */
/* 结果是 y++=11，同时 y 再自增到 12 */

/* 正确：使用内联函数 */
static inline int max(int a, int b)
{
    return (a > b) ? a : b;
}
```

**警示**：⚠️ 宏参数副作用是宏最隐蔽的错误来源。**经验法则**：如果宏参数可能在宏体中被多次使用，确保调用时传入的参数没有副作用。对于复杂的宏逻辑，优先使用内联函数。

### 规则8.5 宏定义中`#`和`##`运算符的使用

**写法**：`#` 将宏参数转为字符串字面量，`##` 连接两个符号。使用时需谨慎，确保结果可预期。

**示例**：
```c
#define STRINGIFY(x)  #x
#define CONCAT(a, b)  a ## b

int CONCAT(my, _variable) = 10;  /* 展开为 int my_variable = 10; */
const char *s = STRINGIFY(hello);  /* 展开为 const char *s = "hello"; */
```

**警示**：⚠️ `##` 运算符可能导致无法预料的符号命名冲突。如果连接后的符号与现有符号冲突，会产生难以调试的编译错误。**建议**：`##` 运算符只在必要时使用（如生成重复的模板代码），一般情况下不要滥用。

### 建议8.1 优先使用const替代宏定义常量

**写法**：对于常量值，优先使用 `const` 变量或枚举，而不是 `#define`。`const` 变量有类型检查和作用域，而宏是全局文本替换。

**示例**：
```c
/* 错误 */
#define MAX_BUFFER_SIZE  4096
#define PI  3.14159

/* 正确 */
static const size_t MAX_BUFFER_SIZE = 4096;
static const double PI = 3.14159;
```

**警示**：⚠️ 宏常量的主要问题：① 没有类型检查，编译器不会发现类型不匹配；② 没有作用域，在所有地方都有效；③ 调试时看不到宏名（预处理阶段已被替换为字面值）。**例外**：需要使用 `#if` 预处理指令判断的值必须使用 `#define`。

### 建议8.2 优先使用内联函数替代宏定义函数

**写法**：C99及以上标准支持 `inline` 关键字，对于简单的函数逻辑，应使用内联函数而非宏。

**示例**：
```c
/* 错误：宏定义函数 */
#define IS_VALID_ID(n)  ((n) > 0 && (n) < MAX_ID)

/* 正确：内联函数 */
static inline bool is_valid_id(int n)
{
    return (n > 0 && n < MAX_ID);
}
```

**警示**：⚠️ 内联函数可以替代绝大多数函数式宏的使用场景，且没有宏的副作用问题。但注意：`inline` 只是对编译器的建议，编译器可能忽略。对于必须内联的代码（如性能关键路径），仍需使用宏。

### 建议8.3 优先使用枚举替代宏定义一组整数常量

**写法**：对于一组相关的整数常量，使用枚举而不是多个 `#define`。枚举有类型检查、作用域和调试信息。

**示例**：
```c
/* 错误 */
#define STATUS_OK       0
#define STATUS_ERROR    1
#define STATUS_TIMEOUT  2

/* 正确 */
typedef enum {
    STATUS_OK = 0,
    STATUS_ERROR,
    STATUS_TIMEOUT
} status_e;
```

**警示**：⚠️ 枚举的优势：① 调试器能显示枚举名；② 编译器能对 `switch` 语句中的枚举值做警告检查（是否覆盖了所有枚举值）；③ 枚举有类型约束，不能随意赋值。

### 建议8.4 条件编译宏的命名规则

**写法**：不同类型的条件编译宏使用统一前缀，便于识别其用途。

**示例**：
```c
/* 平台宏使用统一前缀 */
#define PLATFORM_LINUX    1
#define PLATFORM_WINDOWS  2
#define PLATFORM_MACOS    3

/* 功能宏使用 HAVE_ 前缀 */
#define HAVE_PTHREADS     1
#define HAVE_OPENSSL      1

/* 调试宏使用 DEBUG_ 前缀 */
#define DEBUG_MEMORY      1
#define DEBUG_NETWORK     1
```

**警示**：⚠️ 条件编译宏的命名冲突是大型项目中的常见问题。两个不同的模块可能定义了同名的宏，导致预处理器互相干扰。**命名规范**：建议使用 `<项目>_<模块>_<名称>` 的格式，如 `MYPROJ_NETWORK_HAVE_TLS`。

---

## 9 预处理规范

### 规则9.1 #include 禁止使用非标准路径格式

**写法**：头文件包含路径使用正斜杠 `/`，禁止使用反斜杠 `\`。反斜杠在Windows上是路径分隔符，但在C语言中也是转义字符，可能导致未定义行为。

**示例**：
```c
/* 错误：反斜杠路径 */
#include "..\inc\header.h"

/* 正确：正斜杠路径 */
#include "../inc/header.h"

/* 更好：使用编译器的-I选项设置搜索路径 */
#include "module/header.h"
```

**警示**：⚠️ 反斜杠在字符串常量中会被解释为转义字符（如 `\i` 可能被解释为转义序列）。虽然在 `#include` 中大多数编译器能正确处理，但在不同平台上行为不一致，应始终使用正斜杠。

### 规则9.2 #ifdef / #ifndef 必须配合 #endif 注释

**写法**：条件编译块必须以 `#endif` 注释结束，注释标明 `#if` 对应的条件。

**示例**：
```c
/* 错误：难以匹配 */
#ifdef PLATFORM_LINUX
    /* ... 大量代码 ... */
#endif

/* 正确：注释标明结束匹配 */
#ifdef PLATFORM_LINUX
    /* ... 大量代码 ... */
#endif /* PLATFORM_LINUX */

/* 嵌套条件编译时，注释尤为重要 */
#if defined(PLATFORM_LINUX)
    #ifdef HAVE_PTHREADS
        /* ... */
    #endif /* HAVE_PTHREADS */
#elif defined(PLATFORM_WINDOWS)
    /* ... */
#endif /* PLATFORM_LINUX */
```

**警示**：⚠️ 嵌套条件编译是最容易出错的预处理场景之一。没有 `#endif` 注释，阅读者很难确定每个 `#endif` 对应哪个 `#if`。**经验法则**：如果条件编译块超过 10 行，或者有嵌套，`#endif` 注释是强制性的。

### 规则9.3 禁止使用 #pragma once 替代 include guard

**写法**：使用标准化的 `#ifndef`/`#define`/`#endif` 机制，不使用 `#pragma once`。

**示例**：
```c
/* 正确：标准include guard */
#ifndef __MODULE_HEADER_H__
#define __MODULE_HEADER_H__
/* ... */
#endif /* __MODULE_HEADER_H__ */

/* 错误：使用#pragma once，它不是C标准 */
#pragma once
```

**警示**：⚠️ `#pragma once` 虽然被大多数编译器支持，但不是C标准的一部分。GCC、Clang、MSVC 的行为有细微差异（如对符号链接的处理不同）。在跨平台项目中，应始终使用标准化的 `#ifndef` 机制。

### 规则9.4 条件编译块应尽量短小

**写法**：条件编译块应尽量短小，避免在条件编译中混杂大量逻辑代码。应将平台相关的代码封装为函数。

**示例**：
```c
/* 错误：条件编译中混杂大量代码 */
#ifdef PLATFORM_LINUX
    int result = linux_specific_function();
    if (result < 0) {
        /* ... 20行错误处理 ... */
    }
#else
    int result = windows_specific_function();
    if (result < 0) {
        /* ... 20行错误处理 ... */
    }
#endif

/* 正确：将平台相关代码封装为函数 */
#ifdef PLATFORM_LINUX
static int platform_function(void) { return linux_specific_function(); }
#else
static int platform_function(void) { return windows_specific_function(); }
#endif

int result = platform_function();
if (result < 0) {
    /* ... 统一的错误处理 ... */
}
```

**警示**：⚠️ 条件编译块是代码可读性的杀手。过多的 `#ifdef`/`#else`/`#endif` 使代码变得支离破碎，难以阅读和测试。**经验法则**：条件编译块不应超过 5 行。超过 5 行的平台相关代码，应提取为独立的函数。

### 规则9.5 禁止使用 #line 指令

**写法**：`#line` 指令用于改变编译器报告的行号和文件名，会干扰调试和错误定位，禁止使用。

**警示**：⚠️ `#line` 指令会改变编译器报告的行号，导致：① 调试器断点定位错误；② 编译器错误信息指向错误的行号；③ 代码覆盖率工具无法正确映射。唯一的合法使用场景是代码生成器（如 `yacc`/`lex` 生成的代码）。

### 规则9.6 使用 #error 进行编译时检查

**写法**：使用 `#error` 指令在编译时检测配置错误或不兼容的条件。

**示例**：
```c
#if defined(PLATFORM_LINUX) && defined(PLATFORM_WINDOWS)
    #error "Cannot define both PLATFORM_LINUX and PLATFORM_WINDOWS"
#endif

#if MAX_BUFFER_SIZE < 256
    #error "MAX_BUFFER_SIZE must be at least 256"
#endif
```

**警示**：⚠️ `#error` 在编译时立即终止编译并显示错误信息，比运行时检查更可靠。在配置管理、版本检查、平台兼容性检测中应优先使用 `#error` 而不是运行时检查。

### 规则9.7 使用 #warning 进行编译时提示

**写法**：使用 `#warning` 指令在编译时输出提示信息，用于标记已知问题或临时性配置。

**示例**：
```c
#if __STDC_VERSION__ < 199901L
    #warning "C99 or later is required for this module"
#endif
```

**警示**：⚠️ `#warning` 不是C标准指令（GCC扩展），但大多数编译器都支持。`#warning` 不会终止编译，仅输出警告信息。**注意**：不要将 `#warning` 用于永久性配置，它们最终应该被清理。

### 建议9.1 使用宏定义对版本号进行编译时检查

**写法**：使用标准宏 `__STDC_VERSION__` 检查C语言标准版本，实现条件编译。

**示例**：
```c
/* 检查C语言标准版本 */
#if __STDC_VERSION__ >= 201112L
    /* C11支持 */
    #define HAVE_C11_ATOMICS  1
#endif
```

**警示**：⚠️ `__STDC_VERSION__` 的值因编译器而异：C89为未定义，C99为 `199901L`，C11为 `201112L`，C17为 `201710L`，C23为 `202311L`。不是所有编译器都定义了此宏，在使用前可能需要检查 `__STDC__`。

---

## 10 结构体与联合体规范

### 规则10.1 结构体成员按逻辑分组

**写法**：结构体成员应按逻辑相关性分组，同组内的成员放在一起，组间用空行分隔。这有助于读者快速理解结构体的设计意图。

**示例**：
```c
typedef struct {
    /* 身份信息 */
    uint32_t id;
    char     name[64];

    /* 状态信息 */
    uint32_t state;
    uint64_t last_active_time;

    /* 配置信息 */
    uint32_t priority;
    uint32_t timeout_ms;
} task_info_t;
```

**警示**：⚠️ 结构体成员顺序反映了设计者的思维方式。杂乱无章的成员顺序表明结构体设计缺乏思考。**经验法则**：结构体成员应按"使用频率"或"逻辑关系"分组，而不是按声明顺序随意排列。

### 规则10.2 结构体初始化使用C99指定初始化器

**写法**：使用C99指定初始化器（designated initializer）初始化结构体，不依赖成员顺序。

**示例**：
```c
/* 错误：依赖成员顺序，可读性差 */
task_info_t task = {1, "test", 0, 0, 5, 1000};

/* 正确：使用指定初始化器 */
task_info_t task = {
    .id = 1,
    .name = "test",
    .priority = 5,
    .timeout_ms = 1000
};

/* 零初始化 */
task_info_t task = {0};
```

**警示**：⚠️ 依赖顺序的初始化在结构体成员顺序变化时（如添加新成员或重新排序）会静默失败。指定初始化器：① 不依赖成员顺序，顺序变化不影响；② 可读性更好，每个字段的含义一目了然；③ 未被初始化的字段自动置零。

### 规则10.3 结构体指针使用时必须判空

**写法**：所有结构体指针参数在使用前必须检查是否为 `NULL`。

**示例**：
```c
void print_task(const task_info_t *task)
{
    if (task == NULL) {
        log_error("NULL pointer to print_task");
        return;
    }
    printf("Task: %s (id=%u)\n", task->name, task->id);
}
```

**警示**：⚠️ 指针判空是防御性编程的基本要求。**注意**：即使函数内部没有解引用指针，也建议判空——因为后续修改可能添加解引用操作，而忘记添加判空检查。

### 规则10.4 结构体传参使用指针而非值传递

**写法**：结构体作为函数参数时，使用指针传递（加上 `const` 修饰）而非值传递，避免不必要的拷贝开销。

**示例**：
```c
/* 错误：值传递，拷贝整个结构体 */
void process_task(task_info_t task);

/* 正确：使用const指针 */
void process_task(const task_info_t *task);
```

**警示**：⚠️ 值传递在结构体较大时（> 32字节）会产生显著的性能开销。对于大型结构体，值传递可能导致栈溢出（特别是嵌入式环境）。**建议**：所有超过 4 个成员的结构体都应使用指针传递。

### 规则10.5 注意结构体对齐和填充

**写法**：结构体成员按大小降序排列（从大到小），可以减少内存填充，提高内存利用率。

**示例**：
```c
/* 次优：内存有空洞 */
typedef struct {
    uint8_t  flag;     /* 1字节 */
    /* 3字节填充 */
    uint32_t value;    /* 4字节 */
    uint16_t id;       /* 2字节 */
    /* 2字节填充 */
} bad_layout_t;        /* 共12字节，实际使用7字节 */

/* 优化：按大小降序排列，减少填充 */
typedef struct {
    uint32_t value;    /* 4字节 */
    uint16_t id;       /* 2字节 */
    uint8_t  flag;     /* 1字节 */
    /* 1字节填充 */
} good_layout_t;       /* 共8字节，实际使用7字节，节省33% */
```

**警示**：⚠️ 结构体对齐填充是C语言中常见的"隐形内存浪费"。在嵌入式系统中，这可能意味着数十KB的额外内存消耗。**检查方法**：使用 `sizeof()` 运算符检查结构体大小是否符合预期，或使用 `static_assert` 进行编译时验证。

### 规则10.6 禁止使用匿名结构体/联合体（C11前）

**写法**：C11之前匿名结构体/联合体不是标准特性，降低可移植性，应命名内部结构体/联合体。

**示例**：
```c
/* 错误（C99）：匿名结构体 */
typedef struct {
    int type;
    struct {        /* 匿名 */
        int x;
        int y;
    };
} message_t;

/* 正确：命名内部结构体 */
typedef struct {
    int x;
    int y;
} point_t;

typedef struct {
    int type;
    point_t point;
} message_t;
```

**警示**：⚠️ 匿名结构体/联合体是C11标准才引入的特性。使用非标准扩展（如GCC的 `-fms-extensions`）可能在不同编译器上行为不一致。**建议**：即使项目使用C11+，也建议命名内部结构体以提高可读性。

### 规则10.7 联合体使用时必须明确区分成员类型

**写法**：联合体必须配合类型标识字段使用，确保在读取时使用正确的成员。

**示例**：
```c
typedef struct {
    int type;  /* 类型标识字段 */
    union {
        int int_value;
        float float_value;
        char *string_value;
    } data;
} value_t;

void print_value(const value_t *v)
{
    switch (v->type) {
        case TYPE_INT:
            printf("%d", v->data.int_value);
            break;
        case TYPE_FLOAT:
            printf("%f", v->data.float_value);
            break;
        case TYPE_STRING:
            printf("%s", v->data.string_value);
            break;
        default:
            log_error("Unknown value type: %d", v->type);
            break;
    }
}
```

**警示**：⚠️ 联合体是C语言中类型安全性最差的特性之一。**从联合体中读取的成员类型与最后一次写入的成员类型不一致时，行为是未定义的**（C99起）。**安全建议**：① 始终使用类型标识字段；② 在 `switch` 中提供 `default` 分支处理未知类型；③ 考虑使用 `_Generic`（C11）提供类型安全。

### 规则10.8 结构体作为API接口参数时，应使用不透明指针

**写法**：API接口中，结构体应使用不透明指针（前向声明），隐藏内部实现细节。

**示例**：
```c
/* 头文件中：只声明，不定义 */
typedef struct context_t context_t;

/* API使用不透明指针 */
int context_create(context_t **out_ctx);
int context_process(context_t *ctx, const char *data);
void context_destroy(context_t *ctx);

/* 源文件中：定义结构体 */
struct context_t {
    int fd;
    char buffer[1024];
    size_t offset;
};
```

**警示**：⚠️ 不透明指针是C语言中实现封装性的核心技术。**如果不使用不透明指针**：① 调用者可以看到结构体内部，产生依赖；② 修改结构体内部布局需要重新编译所有调用者；③ 无法防止调用者绕过API直接操作内部成员。

### 建议10.1 使用静态断言检查结构体大小

**写法**：使用 `static_assert`（C11）在编译时检查结构体大小是否符合预期。

**示例**：
```c
#include <assert.h>

typedef struct {
    uint32_t id;
    char name[64];
    uint32_t state;
} user_info_t;

/* 编译时检查结构体大小是否符合预期 */
static_assert(sizeof(user_info_t) == 72, "Unexpected user_info_t size");
```

**警示**：⚠️ 静态断言是预防结构体布局意外的有效手段。在以下场景特别有用：① 结构体用于网络协议或文件格式时；② 结构体在多个模块间共享时；③ 使用 `#pragma pack` 等对齐指令时。

---

## 11 表达式与语句规范

### 规则11.1 表达式运算顺序使用括号明确

**写法**：在复杂表达式中，使用括号明确运算顺序，不依赖C语言运算符优先级规则。

**示例**：
```c
/* 错误：依赖运算符优先级记忆 */
int result = a + b * c >> 1 & 0xFF;

/* 正确：使用括号明确顺序 */
int result = ((a + (b * c)) >> 1) & 0xFF;
```

**警示**：⚠️ C语言有 **15个运算符优先级等级**，即使经验丰富的开发者也无法完全记住所有优先级。依赖运算符优先级不仅容易出错，还会降低代码可读性。**经验法则**：只要不是简单的四则运算，就使用括号明确优先级。

### 规则11.2 禁止将赋值语句放入表达式中

**写法**：不要在条件判断、循环条件等表达式中进行赋值操作，赋值和判断应分开写。

**示例**：
```c
/* 错误：赋值和判断混合 */
if ((fd = open(...)) == -1) {
    /* 赋值和判断混合 */
}

/* 正确：分开写 */
fd = open(...);
if (fd == -1) {
    /* 错误处理 */
}
```

**警示**：⚠️ 在表达式中进行赋值，虽然代码更"紧凑"，但降低了可读性：① 阅读者需要区分 `=`（赋值）和 `==`（等于）；② 调试时无法在赋值语句上设置条件断点；③ 代码审查时更容易被忽略。

### 规则11.3 赋值表达式不应出现在条件判断中

**写法**：`if`、`while`、`for` 等条件判断中不应使用赋值表达式，防止 `=` 与 `==` 混淆。

**示例**：
```c
/* 错误：易混淆为 a == b */
if (a = b) {  /* 实际上是赋值，且条件总是为真（除非b=0） */
    /* ... */
}

/* 正确 */
a = b;
if (a != 0) {
    /* ... */
}
```

**警示**：⚠️ `=` 与 `==` 混淆是C语言中最经典的bug之一。**编译器警告**：大多数编译器在 `-Wall` 选项下会发出 `assignment in conditional expression` 警告。如果确实需要在条件中赋值（如 `while ((c = getchar()) != EOF)`），应再加一层括号：`while ((c = getchar()) != EOF)`。

### 规则11.4 控制语句即使只有一条语句也应使用大括号

**写法**：`if`、`else`、`for`、`while`、`do` 等控制语句体必须使用大括号，即使只有一条语句。

**示例**：
```c
/* 错误：容易因后续添加语句而产生bug */
if (condition)
    do_something();

/* 正确 */
if (condition) {
    do_something();
}
```

**警示**：⚠️ 这是著名的"Apple SSL Bug"（CVE-2014-1266）的根本原因。在 `goto fail;` 语句前缺少大括号，导致安全认证被绕过，影响的用户数以亿计。**永远不要**省略控制语句的大括号。

### 规则11.5 switch语句必须有default分支

**写法**：每个 `switch` 语句必须有 `default` 分支，即使 `default` 分支什么也不做。

**示例**：
```c
switch (cmd) {
    case CMD_START:
        start();
        break;
    case CMD_STOP:
        stop();
        break;
    default:
        log_warning("Unknown command: %d", cmd);
        break;
}
```

**警示**：⚠️ `default` 分支不是可选的，它是防御性编程的基本要求。即使 `switch` 覆盖了所有已知枚举值，`default` 分支也应处理未知值（如未来扩展的枚举值）。**注意**：如果 `default` 分支确实不需要做任何事，应添加注释说明原因。

### 规则11.6 switch语句中case必须显式标注fall-through

**写法**：`case` 之间如果确实需要 `fall-through`（无 `break` 的穿透执行），必须使用 `/* fall-through */` 注释显式标注。

**示例**：
```c
switch (error_code) {
    case ERR_WARN_LOW_MEMORY:
        log_warning("Low memory");
        /* fall-through：继续执行降级处理 */
    case ERR_FATAL_NO_MEMORY:
        enter_degraded_mode();
        break;
    case ERR_FATAL_CORRUPTION:
        emergency_shutdown();
        break;
    default:
        log_error("Unknown error: %d", error_code);
        break;
}
```

**警示**：⚠️ `fall-through` 是 `switch` 语句中bug的高发区。大多数case后的 `break` 遗漏是bug而非有意为之。**编译检查**：使用 `-Wimplicit-fallthrough`（GCC/Clang）可以检测未标注的 `fall-through`。**建议**：如果项目有了C23标准，可以使用 `[[fallthrough]]` 属性。

### 规则11.7 避免使用逗号表达式

**写法**：逗号表达式降低代码可读性，应避免使用。`for` 循环的第三个表达式除外。

**示例**：
```c
/* 错误：逗号表达式降低可读性 */
for (i = 0, j = 10; i < 10; i++, j--) {
    /* ... */
}

/* 正确：分开处理 */
i = 0;
j = 10;
for (; i < 10; i++, j--) {
    /* ... */
}
```

**警示**：⚠️ 逗号表达式会按顺序执行所有表达式，但只返回最后一个表达式的值。这种特性容易导致代码难以理解，且几乎总是可以用多个语句替代。

### 规则11.8 禁止在表达式中使用sizeof计算指针指向的数组大小

**写法**：`sizeof` 对指针求值得到的是指针本身的大小，而不是它指向的数组的大小。数组大小应作为参数传递。

**示例**：
```c
/* 错误：sizeof(ptr)得到的是指针大小，而非数组大小 */
void process(int arr[])
{
    size_t size = sizeof(arr);  /* 错误：得到的是指针大小（4或8字节） */
}

/* 正确：将数组大小作为参数传递 */
void process(int arr[], size_t size)
{
    /* 使用size参数 */
}

/* 正确：在数组定义处使用sizeof */
int arr[100];
size_t size = sizeof(arr);  /* 正确：此时arr是数组，不是指针 */
```

**警示**：⚠️ 数组名作为函数参数时，会退化为指针（array decay）。这是C语言中最容易混淆的特性之一。**经验法则**：在函数参数中使用 `int arr[]` 时，必须同时传递数组大小。**宏辅助**：使用 `ARRAY_SIZE(arr)` 宏在数组定义处计算大小。

### 建议11.1 避免使用三元运算符代替简单if-else

**写法**：三元运算符（`?:`）适用于简单的赋值选择，嵌套的三元运算符应避免。

**示例**：
```c
/* 可读性差 */
result = (a > b) ? ((c > d) ? x : y) : z;

/* 更好 */
if (a > b) {
    result = (c > d) ? x : y;
} else {
    result = z;
}
```

**警示**：⚠️ 嵌套的三元运算符是"代码高尔夫"的典型表现。虽然代码行数更少，但可读性急剧下降。**经验法则**：三元运算符只用于简单的"二选一"场景，嵌套超过 1 层就应该使用 `if-else`。

### 建议11.2 避免使用goto语句

**写法**：`goto` 应谨慎使用，仅在多层嵌套的错误处理中用于统一资源释放是可接受的场景。

**示例**：
```c
/* 可接受的goto使用场景：统一错误处理 */
void *buffer1 = NULL;
void *buffer2 = NULL;

buffer1 = malloc(1024);
if (buffer1 == NULL) {
    goto cleanup;
}

buffer2 = malloc(2048);
if (buffer2 == NULL) {
    goto cleanup;
}

/* 正常处理逻辑 */
process(buffer1, buffer2);

cleanup:
    free(buffer1);
    free(buffer2);
```

**警示**：⚠️ `goto` 的主要问题是：① 使代码的执行路径变得难以追踪（"意大利面条代码"）；② 可能跳过变量初始化，导致未定义行为；③ 破坏了结构化编程的基本原则。**使用限制**：`goto` 只能向后跳转，不能跨函数跳转，且只能用于错误处理场景。

### 建议11.3 条件表达式中的常量应放在左侧

**写法**：在条件表达式中，将常量放在比较运算符的左侧，可以防止误将 `==` 写为 `=`。

**示例**：
```c
/* 建议：将常量放在左侧，避免误写为赋值 */
if (NULL == ptr) { ... }  /* 如果误写为 =，编译会报错 */
if (0 == error_code) { ... }

/* 但更推荐可读性优先的写法 */
if (ptr == NULL) { ... }
if (error_code == 0) { ... }
```

**警示**：⚠️ 将常量放在左侧（又称"尤达条件式"）是一种防错技巧。如果误写为 `=`，赋值给常量是非法操作，编译器会报错。但这种方式牺牲了可读性——英文中"if NULL equals ptr"不符合自然语言习惯。**建议**：结合编译器警告（`-Wparentheses`）使用，而不是依赖这种编码风格。

---

## 12 内存管理规范

### 规则12.1 内存申请前必须校验大小合法性

**写法**：在调用 `malloc`/`calloc` 前，必须检查申请的大小是否合法（大于0且不超过最大限制）。

**示例**：
```c
/* 错误 */
void *ptr = malloc(size);  /* size可能为0或过大 */

/* 正确 */
if (size == 0 || size > MAX_ALLOC_SIZE) {
    log_error("Invalid memory size: %zu", size);
    return NULL;
}
void *ptr = malloc(size);
```

**警示**：⚠️ 申请大小为 0 的内存时，C标准允许 `malloc(0)` 返回 `NULL` 或一个非 `NULL` 的不可解引用指针，这是未定义行为的重灾区。申请过大的内存可能导致拒绝服务（DoS）。**经验法则**：所有内存申请前必须做合法性校验。

### 规则12.2 内存分配后必须判断是否成功

**写法**：`malloc`/`calloc`/`realloc` 返回后必须立即检查是否为 `NULL`。

**示例**：
```c
/* 正确 */
int *buffer = malloc(sizeof(int) * count);
if (buffer == NULL) {
    log_error("Memory allocation failed");
    return ERR_NO_MEMORY;
}
```

**警示**：⚠️ 在嵌入式系统或内存受限环境中，`malloc` 失败的可能性更高。在大型系统中，内存碎片也可能导致 `malloc` 失败。即使是在"内存充裕"的服务器端，也应检查 `malloc` 返回值——这是防御性编程的基本要求。

### 规则12.3 禁止引用未初始化的内存

**写法**：使用 `malloc` 分配的内存后，在读取之前必须初始化。推荐使用 `calloc` 分配零初始化内存。

**示例**：
```c
/* 错误 */
int *buffer = malloc(sizeof(int) * 10);
printf("%d", buffer[0]);  /* 未初始化 */

/* 正确 */
int *buffer = calloc(10, sizeof(int));  /* 初始化为0 */
```

**警示**：⚠️ 读取未初始化的内存是未定义行为。从 `malloc` 返回的内存内容是不确定的，可能包含敏感数据（如密码、密钥）。**安全角度**：使用 `calloc` 不仅可以避免未初始化问题，还能防止内存内容泄露。

### 规则12.4 内存释放后指针必须置NULL

**写法**：`free(ptr)` 后立即将 `ptr` 置为 `NULL`，防止悬垂指针（dangling pointer）。

**示例**：
```c
free(ptr);
ptr = NULL;  /* 防止悬垂指针 */
```

**警示**：⚠️ 释放后不置 NULL 的指针称为"悬垂指针"。如果后续误用该指针，将导致 use-after-free（CWE-416），这是最严重的内存安全漏洞之一。**注意**：如果局部指针在函数末尾释放，且函数结束后指针不再使用，可以不必置 NULL，但建议统一置 NULL 以养成习惯。

### 规则12.5 禁止使用realloc()

**写法**：`realloc()` 语义复杂，容易出错，应使用 `malloc` + `memcpy` + `free` 替代。

**示例**：
```c
/* 错误：realloc语义复杂，容易导致内存泄漏 */
void *new_ptr = realloc(old_ptr, new_size);  /* 失败时old_ptr不变，但返回NULL */

/* 正确：使用malloc+memcpy+free */
void *new_ptr = malloc(new_size);
if (new_ptr == NULL) {
    log_error("Memory allocation failed");
    return NULL;  /* old_ptr仍然有效 */
}
memcpy(new_ptr, old_ptr, old_size < new_size ? old_size : new_size);
free(old_ptr);
```

**警示**：⚠️ `realloc` 的语义非常微妙：① 成功时返回新地址，原地址已释放；② 失败时返回 `NULL`，原地址不变。如果直接写 `ptr = realloc(ptr, new_size)`，当 `realloc` 失败时，`ptr` 被置为 `NULL`，原内存丢失，造成内存泄漏。**建议**：始终使用 `malloc` + `memcpy` + `free` 代替 `realloc`。

### 规则12.6 禁止使用alloca()

**写法**：`alloca()` 在栈上分配内存，大小不可控，容易导致栈溢出，应使用固定大小数组或 `malloc`。

**警示**：⚠️ `alloca()` 在栈上分配内存，且释放方式与 `free` 不同（函数返回时自动释放）。如果分配大小由外部数据控制，攻击者可以构造恶意输入导致栈溢出，覆盖返回地址。**此外**：`alloca()` 不是C标准函数，可移植性差。**替代方案**：使用固定大小数组（如 `char buffer[1024]`）或 `malloc`。

### 规则12.7 内存分配和释放应在同一层级

**写法**：谁分配谁释放，避免跨模块的内存管理。

**示例**：
```c
/* 错误：模块A分配，模块B释放 */
void *data = module_a_create();
module_b_destroy(data);

/* 正确：谁分配谁释放 */
void *data = module_a_create();
module_a_destroy(data);
```

**警示**：⚠️ 跨模块的内存管理是资源泄漏的主要来源之一。不同模块可能使用不同的内存管理器（如不同版本的 CRT），导致 `free` 崩溃。**规范**：如果确实需要跨模块释放，应提供专门的释放函数，如 `module_a_destroy()`。

### 规则12.8 资源获取即初始化（RAII风格）

**写法**：对于需要成对使用的资源操作，应确保所有路径都正确释放。

**示例**：
```c
/* 好的模式：函数内所有资源在退出前释放 */
int process_file(const char *path)
{
    FILE *fp = fopen(path, "r");
    if (fp == NULL) {
        return ERR_OPEN;
    }

    char *buffer = malloc(1024);
    if (buffer == NULL) {
        fclose(fp);
        return ERR_NO_MEMORY;
    }

    /* 处理逻辑 */
    int result = read_and_process(fp, buffer);

    /* 统一释放 */
    free(buffer);
    fclose(fp);
    return result;
}
```

**警示**：⚠️ RAII 的核心原则是"资源在初始化时获取，在析构时释放"。在C语言中没有构造/析构函数，因此需要手动维护。**常见错误**：在错误路径中忘记释放已分配的资源，导致内存泄漏。**建议**：使用 `goto cleanup` 模式统一管理资源释放。

### 规则12.9 使用malloc时，大小计算使用sizeof而非硬编码

**写法**：`malloc` 的大小计算使用 `sizeof` 运算符，避免硬编码，且使用类型推导提高可维护性。

**示例**：
```c
/* 错误：硬编码大小 */
int *arr = malloc(40);  /* 假设int为4字节，10个元素 */

/* 正确：使用sizeof */
int *arr = malloc(10 * sizeof(int));

/* 更好：使用类型推导 */
int *arr = malloc(10 * sizeof(*arr));  /* 即使类型改变也能自动适配 */
```

**警示**：⚠️ 硬编码大小在以下场景会出错：① 平台迁移（如从32位迁移到64位，`int` 从4字节变为8字节）；② 结构体字段变更（如 `uint32_t` 改为 `uint64_t`）。**经验法则**：始终使用 `sizeof(*ptr)` 而非 `sizeof(type)`，这样即使 `ptr` 的类型改变，分配的大小也会自动适配。

### 建议12.1 使用内存池管理频繁分配释放的场景

**写法**：对于频繁分配和释放小内存块的场景，建议使用内存池（Memory Pool）而非直接调用 `malloc`/`free`。

**警示**：⚠️ 频繁的 `malloc`/`free` 会导致内存碎片，降低系统性能，甚至导致分配失败。**内存池优势**：① 避免碎片；② 分配速度快（O(1)）；③ 可预测的内存使用。**适用场景**：网络包处理、消息队列、游戏引擎等。

### 规则12.10 封装安全释放宏，避免资源泄漏

**写法**：封装统一的资源释放宏，确保释放后指针置空、关闭后句柄置无效，减少遗漏的可能。

**示例**：

```c
/* 安全内存释放宏 */
#define SAFE_FREE(p)            \
    do {                        \
        free(p);                \
        (p) = NULL;             \
    } while (0)

/* 安全文件关闭宏 */
#define SAFE_FCLOSE(fp)         \
    do {                        \
        if ((fp) != NULL) {     \
            fclose(fp);         \
            (fp) = NULL;        \
        }                       \
    } while (0)

/* 安全文件描述符关闭宏 */
#define SAFE_CLOSE(fd)          \
    do {                        \
        if ((fd) >= 0) {        \
            close(fd);          \
            (fd) = -1;          \
        }                       \
    } while (0)

/* 使用示例 */
void process_file(const char *path)
{
    FILE *fp = fopen(path, "r");
    char *buf = malloc(BUF_SIZE);

    if (fp == NULL || buf == NULL) {
        log_error("Failed to open file or allocate buffer");
        SAFE_FCLOSE(fp);
        SAFE_FREE(buf);
        return;
    }

    /* ... 处理文件 ... */

    /* 统一释放 */
    SAFE_FCLOSE(fp);
    SAFE_FREE(buf);
}
```

**警示**：⚠️ `free(NULL)` 在C标准中是合法的，因此 `SAFE_FREE` 无需判空。但 `fclose(NULL)` 的行为是未定义的，因此 `SAFE_FCLOSE` 必须判空。**经验法则**：使用 `SAFE_*` 宏可以确保所有释放操作统一、完整，减少资源泄漏的可能。**注意**：这些宏使用了 `do { ... } while (0)` 包装，确保宏在 `if` 语句中也能正确工作。

---

## 13 字符串与缓冲区安全规范

### 规则13.1 使用安全函数替代不安全函数

**写法**：始终使用指定缓冲区大小的安全函数替代不安全函数，禁止使用 `strcpy`、`strcat`、`sprintf`、`gets` 等。

| 不安全函数 | 安全替代 | 说明 |
|-----------|---------|------|
| `strcpy` | `strncpy` / `strcpy_s` | 指定目标缓冲区大小 |
| `strcat` | `strncat` / `strcat_s` | 指定目标缓冲区剩余大小 |
| `sprintf` | `snprintf` / `sprintf_s` | 指定输出缓冲区大小 |
| `vsprintf` | `vsnprintf` / `vsprintf_s` | 指定输出缓冲区大小 |
| `gets` | `fgets` / `gets_s` | 指定输入缓冲区大小 |
| `scanf` | `fscanf` / `sscanf` | 指定最大读取长度 |
| `strlen` | `strnlen_s` | 指定最大长度，防止越界 |

**警示**：⚠️ 不安全字符串函数是缓冲区溢出漏洞（CWE-121）的主要来源。`strcpy` 不检查目标缓冲区大小，如果源字符串长度超过目标缓冲区，将导致缓冲区溢出。**Apple SSL Bug** 和 **Heartbleed** 等著名漏洞都与缓冲区操作相关。**经验法则**：永远不要使用不指定缓冲区大小的字符串函数。

### 规则13.2 确保字符串以'\0'结尾

**写法**：使用 `strncpy` 后必须手动确保字符串以 `'\0'` 结尾。

**示例**：
```c
char dest[16];
strncpy(dest, source, sizeof(dest) - 1);
dest[sizeof(dest) - 1] = '\0';  /* 确保以'\0'结尾 */
```

**警示**：⚠️ `strncpy` 在源字符串长度达到或超过缓冲区大小时，不会自动添加 `'\0'` 结尾。这会导致后续的 `strlen`、`printf("%s")` 等函数读取越界，造成信息泄露或缓冲区溢出。**注意**：`strncpy` 不是"安全版的 `strcpy`"，它只是"限制长度的 `strcpy`"——仍需手动确保 `'\0'` 结尾。

### 规则13.3 外部数据作为数组索引时必须校验范围

**写法**：来自外部输入的数据作为数组索引前，必须进行范围校验。

**示例**：
```c
/* 错误 */
int value = array[user_input];  /* 外部数据直接作为索引 */

/* 正确 */
if (user_input >= 0 && user_input < ARRAY_SIZE(array)) {
    int value = array[user_input];
} else {
    log_error("Array index out of bounds: %d", user_input);
}
```

**警示**：⚠️ 外部数据直接作为数组索引是严重的安全漏洞，允许攻击者读取或写入任意内存地址（CWE-787）。**攻击示例**：如果 `user_input` 为负数或极大值，可以越界访问数组前后的内存区域。

### 规则13.4 格式化字符串禁止由外部可控

**写法**：格式化字符串必须为常量字符串，禁止将用户输入直接作为 `printf` 的第一个参数。

**示例**：
```c
/* 错误：格式化字符串攻击 */
printf(user_input);

/* 正确 */
printf("%s", user_input);
```

**警示**：⚠️ 格式化字符串漏洞（CWE-134）允许攻击者：① 读取栈上任意数据（信息泄露）；② 写入任意内存地址（代码执行）。**攻击示例**：如果 `user_input` 为 `%x%x%x%x`，将泄露栈上的数据；如果为 `%n`，将写入内存。

### 规则13.5 格式化参数类型和个数必须匹配

**写法**：`printf` 系列函数的格式化字符串中参数类型和个数必须与实际参数匹配。

**示例**：
```c
/* 错误 */
printf("%s %d", "hello");          /* 缺少参数 */
printf("%s", 42);                  /* 类型不匹配 */

/* 正确 */
printf("%s %d", "hello", 42);
```

**警示**：⚠️ 格式化参数不匹配是未定义行为。常见的错误包括：① 参数个数少于格式化字符串中的占位符；② 参数类型不匹配（如 `%s` 传入整数，`%d` 传入指针）。**编译检查**：使用 `-Wformat`（GCC/Clang）可以检测大部分格式化参数不匹配的问题。

### 规则13.6 snprintf返回值必须检查

**写法**：`snprintf` 的返回值必须检查，判断是否发生了截断。

**示例**：
```c
char buffer[64];
int written = snprintf(buffer, sizeof(buffer), "value=%d", value);
if (written < 0 || (size_t)written >= sizeof(buffer)) {
    log_warning("Buffer truncated, expected %d chars", written);
}
```

**警示**：⚠️ `snprintf` 返回"如果缓冲区足够大，本应写入的字符数"（不包括 `'\0'`）。如果返回值 >= 缓冲区大小，说明输出被截断了。被截断的数据可能是不完整的，可能导致后续逻辑错误。**注意**：`snprintf` 总是以 `'\0'` 结尾（只要缓冲区大小 > 0）。

### 规则13.7 memcpy/memmove使用注意

**写法**：当源和目标内存区域可能重叠时，必须使用 `memmove` 而非 `memcpy`。

**示例**：
```c
/* memcpy要求源和目标不重叠，重叠时应使用memmove */
memmove(dest, src, len);  /* 安全：支持重叠 */
memcpy(dest, src, len);   /* 不安全：不支持重叠 */
```

**警示**：⚠️ `memcpy` 不保证处理重叠内存的正确性。当源和目标区域重叠时，`memcpy` 的行为是未定义的，而 `memmove` 保证正确处理重叠。**经验法则**：如果无法确定是否重叠，使用 `memmove`。

### 规则13.8 使用sizeof计算数组元素个数

**写法**：使用宏 `ARRAY_SIZE` 计算数组元素个数，避免硬编码。

**示例**：
```c
/* 推荐：使用宏 */
#define ARRAY_SIZE(arr)  (sizeof(arr) / sizeof((arr)[0]))

int buffer[256];
for (size_t i = 0; i < ARRAY_SIZE(buffer); i++) {
    buffer[i] = 0;
}
```

**警示**：⚠️ `ARRAY_SIZE` 宏在指针上使用时会产生错误结果。**常见错误**：将数组作为参数传递给函数后，在函数内部使用 `ARRAY_SIZE`——此时形参已退化为指针，`sizeof(arr)` 得到的是指针大小而非数组大小。**注意**：`ARRAY_SIZE` 宏中的 `(arr)` 带括号，确保即使传入表达式也能正确处理。

---

## 14 错误处理规范

### 规则14.1 使用统一的错误码定义

**写法**：项目中应定义统一的错误码体系，所有模块使用相同的错误码枚举。

**示例**：
```c
typedef enum {
    ERR_SUCCESS       = 0,      /* 成功 */
    ERR_GENERAL       = -1,     /* 通用错误 */
    ERR_NO_MEMORY     = -2,     /* 内存不足 */
    ERR_INVALID_PARAM = -3,     /* 参数无效 */
    ERR_TIMEOUT       = -4,     /* 超时 */
    ERR_NOT_FOUND     = -5,     /* 未找到 */
    ERR_BUSY          = -6,     /* 资源忙 */
    ERR_IO            = -7,     /* IO错误 */
    ERR_PERMISSION    = -8,     /* 权限不足 */
    ERR_ALREADY_EXIST = -9,     /* 已存在 */
    ERR_OVERFLOW      = -10,    /* 溢出 */
    ERR_NOT_SUPPORTED = -11,    /* 不支持 */
    ERR_CANCELLED     = -12,    /* 取消 */
    /* 各模块可从 -100 开始定义模块特有错误码 */
} errno_e;
```

**警示**：⚠️ 没有统一错误码体系的项目，常见问题：① 不同模块返回不同的错误码（如 `-1`、`0xFFFF`、`NULL`）；② 错误码语义冲突（如 `-1` 在不同模块中含义不同）；③ 调用者无法统一处理错误。**建议**：错误码一旦定义，不得修改其语义，只能新增。

### 规则14.2 函数返回值必须被检查或显式忽略

**写法**：函数返回值必须被检查，如果需要显式忽略，使用 `(void)` 转换并注明原因。

**示例**：
```c
/* 错误：忽略返回值 */
fclose(fp);

/* 正确：检查返回值 */
if (fclose(fp) != 0) {
    log_warning("Failed to close file properly");
}

/* 或显式忽略（需记录原因） */
(void)fclose(fp);  /* 只读操作，关闭失败不影响数据一致性 */
```

**警示**：⚠️ 忽略函数返回值是导致"非预期行为"的主要原因之一。**常见忽略场景**：`fclose`、`write`、`read`、`close` 等系统调用。**注意**：`fclose` 的失败可能意味着数据未完全写入磁盘，忽略它可能导致数据丢失。**编译检查**：使用 `-Wunused-result`（GCC）可以检测未使用的返回值。

### 规则14.3 错误处理代码与正常逻辑分离

**写法**：先检查错误条件，提前返回，正常逻辑放在后面，避免深层嵌套。

**示例**：
```c
/* 错误：错误处理散布在正常逻辑中 */
if (result == 0) {
    /* 正常处理100行 */
} else {
    /* 错误处理 */
}

/* 正确：先检查错误，提前返回 */
if (result != 0) {
    log_error("Operation failed: %d", result);
    return result;
}
/* 正常处理 */
```

**警示**：⚠️ 深层嵌套的 `if-else` 使代码难以阅读和维护。**"提前返回"模式**（Early Return）将错误处理前置，正常逻辑后置，使代码更清晰。**经验法则**：嵌套深度不应超过 3 层。

### 规则14.4 断言使用规范

**写法**：断言用于验证"不应该发生"的情况，仅在调试版有效。

**示例**：
```c
/* 断言必须使用宏封装 */
#ifdef DEBUG
#define ASSERT(cond) \
    do { \
        if (!(cond)) { \
            log_fatal("ASSERT failed: %s, file %s, line %d", \
                      #cond, __FILE__, __LINE__); \
            abort(); \
        } \
    } while (0)
#else
#define ASSERT(cond)  ((void)0)
#endif
```

```c
/* 断言使用场景：验证内部逻辑一致性 */
ASSERT(ptr != NULL);  /* 内部函数，调用者保证了ptr不为NULL */

/* 断言不能用于：处理外部输入 */
ASSERT(user_input > 0);  /* 错误：用户输入可能为负数 */
```

**警示**：⚠️ 断言在 `NDEBUG` 宏定义时被移除，因此断言不能用于：① 验证外部输入；② 执行有副作用的表达式；③ 检查运行时错误（这些应使用 `if` 检查）。**断言 vs 错误处理**：断言用于"程序内部错误"，错误处理用于"外部不可控条件"。

### 规则14.5 断言内不能有副作用

**写法**：断言表达式内不能有函数调用、赋值、自增等副作用操作。

**示例**：
```c
/* 错误：断言内有副作用 */
ASSERT(p = malloc(1024));  /* 调试版有效，发布版不执行 */

/* 正确 */
p = malloc(1024);
ASSERT(p != NULL);
```

**警示**：⚠️ 断言在发布版（`NDEBUG` 定义）中被完全移除。如果断言内有副作用，将导致调试版和发布版行为不一致，这是灾难性的。**经验法则**：断言表达式中只能包含比较操作和常量表达式。

### 规则14.6 错误码传递链

**写法**：函数应保持错误码的语义一致性，底层错误码不应被随意修改。

**示例**：
```c
int read_config(void)
{
    FILE *fp = fopen("config.ini", "r");
    if (fp == NULL) {
        /* 保留原始错误码的语义 */
        return ERR_IO;  /* 不要返回ERR_GENERAL而丢失语义 */
    }
    /* ... */
}
```

**警示**：⚠️ 错误码传递链中，每层都应保留或转化错误语义。将底层错误统一替换为 `ERR_GENERAL` 会使调用者无法区分错误类型。**建议**：如果确实需要转换错误码，应保证转换后的错误码语义不丢失。

### 建议14.1 公共API函数参数验证使用运行时检查

**写法**：公开API函数的参数验证应使用运行时检查（`if` 判断），而非断言。

**示例**：
```c
int tcp_connect(const char *host, uint16_t port)
{
    if (host == NULL || port == 0) {
        return ERR_INVALID_PARAM;  /* 运行时检查，对所有版本有效 */
    }
    /* ... */
}
```

**警示**：⚠️ API函数的参数验证必须使用运行时检查，因为调用者可能来自外部模块或第三方，无法保证其传递的参数合法性。断言仅用于内部函数的"不应发生"的假设验证。

### 建议14.2 错误日志应包含上下文信息

**写法**：错误日志应包含足够的上下文信息，便于定位问题。

**示例**：
```c
/* 好的错误日志：包含函数名、错误码、上下文 */
log_error("tcp_connect('%s', %u) failed: %d", host, port, errcode);

/* 不好的错误日志：缺乏上下文 */
log_error("connect failed");
```

**警示**：⚠️ 缺乏上下文的错误日志在生产环境中几乎毫无价值。当用户报告"连接失败"时，没有主机名、端口号、错误码的日志无法帮助定位问题。**经验法则**：错误日志应包含"发生了什么"、"在哪里发生"、"失败的原因是什么"。

### 原则14.1 代码质量保证优先原则

**写法**：以下原则按优先级排列，在编码和审查时以此顺序判断：正确性 > 简洁性 > 可维护性 > 可测试性 > 效率。质量保证的投入与代码质量正相关，应在编码阶段即开始质量保证活动。

**示例**：

**质量保证优先级矩阵**：
```
第一优先级：正确性 —— 代码必须正确实现功能
第二优先级：简洁性 —— 代码应清晰易懂，避免过度设计
第三优先级：可维护性 —— 代码应便于修改和扩展
第四优先级：可测试性 —— 代码应便于测试验证
第五优先级：效率 —— 在满足上述要求的前提下追求性能
```

**警示**：⚠️ 将效率置于正确性之上是"过早优化"的典型表现。在业界实践中，**可运行但慢的代码**远好于**快但错误的代码**。质量保证的投入在项目早期看起来"增加成本"，但在维护期会数倍回收获益。

### 规则14.7 运行时可能导致的错误严禁使用断言

**写法**：断言只用于验证"不应该发生"的内部逻辑错误（程序员的错误），不用于处理运行时可能发生的错误（如用户输入错误、文件不存在、网络超时等）。

**示例**：
```c
/* 正确：断言用于内部逻辑约束 */
typedef struct {
    int id;
    char name[64];
} user_t;

user_t *find_user_by_id(int id)
{
    user_t *user = hash_table_lookup(g_user_table, id);
    ASSERT(user != NULL);  /* 内部函数：调用者保证id存在，此处断言验证 */
    return user;
}

/* 正确：运行时错误使用if检查，不应使用断言 */
int load_config(const char *path)
{
    if (path == NULL) {
        return ERR_INVALID_PARAM;  /* 外部输入：运行时检查 */
    }
    FILE *fp = fopen(path, "r");
    if (fp == NULL) {
        return ERR_IO;  /* 运行时错误：文件可能不存在 */
    }
    /* ... */
}

/* 错误：用断言处理运行时错误 */
void process_request(const char *data)
{
    ASSERT(data != NULL);  /* 错误！data来自网络，在Release版中assert被移除 */
    /* 当data为NULL时，Release版将使用空指针，导致崩溃 */
}
```

**警示**：⚠️ 断言在 `NDEBUG` 宏定义时被完全移除（Release版）。如果断言用于验证外部输入：
- 调试版能正常工作（断言触发，abort）
- 发布版断言被移除，外部输入未经验证直接使用，可能导致安全问题
- **调试版和发布版行为不一致**，这是最危险的编程错误之一

**CWE相关**：CWE-617（断言中可触及的reachable assertion）、CWE-783（运算符优先级错误导致断言无效）

### 规则14.8 断言必须使用宏定义，禁止直接调用assert()

**写法**：禁止直接调用C标准库的 `assert()`，必须使用自定义的断言宏，以便统一控制断言行为（如日志记录、调试器中断、错误上报等）。

**示例**：
```c
/* 错误：直接使用assert() */
#include <assert.h>
assert(ptr != NULL);  /* 行为不可控，无法统一处理 */

/* 正确：自定义断言宏 */
#ifdef DEBUG
#define ASSERT(cond) \
    do { \
        if (!(cond)) { \
            log_fatal("[ASSERT] %s (file: %s, line: %d, func: %s)", \
                      #cond, __FILE__, __LINE__, __func__); \
            /* 触发调试器中断 */ \
            DEBUG_BREAK(); \
            abort(); \
        } \
    } while (0)
#else
#define ASSERT(cond)  ((void)0)
#endif

/* 使用自定义断言 */
ASSERT(ptr != NULL);
```

**警示**：⚠️ 自定义断言宏的优势：
1. **统一日志**：记录完整的断言失败上下文（条件、文件、行号、函数名）
2. **统一行为**：可以在调试版触发调试器断点，方便快速定位
3. **统一关闭**：Release版统一消除，不会遗漏
4. **可扩展**：可以添加统计、告警、崩溃上报等能力

### 规则14.9 函数参数合法性检查使用ASSERT（内部函数）或if（API函数）

**写法**：内部函数的参数合法性检查使用 `ASSERT`（调试版验证），公共API函数的参数合法性检查使用 `if` 运行时检查（所有版本有效）。禁止对公共API函数的参数进行 `ASSERT` 操作。

**示例**：
```c
/* 内部函数：使用ASSERT验证 */
static int process_internal(user_t *user)
{
    ASSERT(user != NULL);          /* 内部调用者保证 */
    ASSERT(user->id > 0);          /* 内部约束 */
    return do_process(user);
}

/* 公共API函数：使用if运行时检查 */
int api_process_user(user_t *user)
{
    if (user == NULL) {
        return ERR_INVALID_PARAM;  /* 运行时检查，对所有版本有效 */
    }
    if (user->id <= 0) {
        return ERR_INVALID_PARAM;
    }
    return process_internal(user);
}
```

**警示**：⚠️ **公共API函数的参数验证必须使用运行时检查（if）**，因为：
- 调用者可能来自外部模块或第三方，无法保证参数合法性
- 断言在Release版被移除，API函数的参数验证不能依赖断言
- 公共API函数的参数错误是"可能发生的"，而非"不应发生的"

### 规则14.10 代码质量保证检查清单

**写法**：每个函数提交前，应至少完成以下质量保证检查项的自我审查。

**示例**：

**函数提交前自检清单**：
```c
/* 
 * [ ] 函数是否只完成了一件事？
 * [ ] 函数是否超过50行？
 * [ ] 所有输入参数是否都进行了校验？
 * [ ] 所有返回值是否都进行了检查？
 * [ ] 所有分配的资源是否都正确释放？
 * [ ] 所有分支是否都有返回值？
 * [ ] 是否存在未初始化的变量？
 * [ ] 是否存在潜在的空指针解引用？
 * [ ] 是否存在潜在的缓冲区溢出？
 * [ ] 是否存在整数溢出的可能？
 * [ ] 断言是否没有副作用？
 * [ ] 断言是否只用于"不应发生"的情况？
 */
```

**警示**：⚠️ 质量保证的"最后一道防线"是代码审查（Code Review）。自检清单不能替代审查，但可以帮助开发者在提交前消除大部分常见问题。**经验法则**：自检清单的每一条都对应了实际项目中发生过的问题。

---

## 15 整数安全规范

### 规则15.1 整数运算必须检查溢出

**写法**：在整数运算前检查是否会导致溢出，避免未定义行为。

**示例**：
```c
/* 加法溢出检查 */
if (a > INT_MAX - b) {
    log_error("Integer overflow");
    return ERR_OVERFLOW;
}
int result = a + b;
```

```c
/* 乘法溢出检查 */
if (b != 0 && a > INT_MAX / b) {
    log_error("Integer overflow");
    return ERR_OVERFLOW;
}
int result = a * b;
```

```c
/* 无符号整数回绕（C标准定义的行为，但通常不期望） */
uint32_t a = UINT32_MAX;
uint32_t b = 1;
uint32_t c = a + b;  /* c = 0，回绕 */
```

**警示**：⚠️ 有符号整数溢出是未定义行为（CWE-190），编译器可能做出任何优化。**攻击示例**：如果 `a + b` 溢出，结果可能是负数，但后续代码将其作为正数使用，导致缓冲区大小计算错误，进而造成缓冲区溢出。**C标准**：有符号整数溢出是未定义行为，无符号整数溢出是定义的回绕（wrapping）。**推荐**：使用C11的 `ckd_add` 等安全运算函数，或使用编译器内置函数如 `__builtin_add_overflow`。

### 规则15.2 整数类型转换必须显式

**写法**：整数类型转换必须使用显式类型转换，避免隐式转换导致精度丢失或符号问题。

**示例**：
```c
/* 错误：隐式类型转换可能导致溢出 */
long result = a * b;  /* a和b是int，乘积累出int范围后才转换 */

/* 正确：在运算前转换 */
long result = (long)a * (long)b;
```

**警示**：⚠️ 隐式类型转换是C语言中最隐蔽的bug来源之一。**典型场景**：`int * int` 的乘积可能溢出 `int` 范围，然后再隐式转换为 `long`，此时结果已经是错误的值。**经验法则**：运算前将操作数转换为期望的类型，而不是依赖隐式转换。

### 规则15.3 禁止有符号整数进行位运算

**写法**：位运算必须使用无符号整数类型，有符号整数的位运算行为是实现定义的。

**示例**：
```c
/* 错误：有符号整数的位运算行为是实现定义的 */
int x = -1;
int y = x >> 1;  /* 算术右移还是逻辑右移？由实现定义 */

/* 正确：使用无符号类型 */
uint32_t x = 0xFFFFFFFF;
uint32_t y = x >> 1;  /* 确定：逻辑右移，结果为0x7FFFFFFF */
```

**警示**：⚠️ 有符号整数的位运算（特别是右移）行为由实现定义：大多数编译器实现算术右移（保留符号位），但C标准不保证这一点。**左移有符号负数**：左移有符号负数是未定义行为。**经验法则**：位运算全部使用无符号类型。

### 规则15.4 禁止整数与指针互相转化

**写法**：禁止将整数直接转换为指针，或指针直接转换为整数。必须转换时使用 `uintptr_t`。

**示例**：
```c
/* 错误 */
int addr = 0x1000;
int *ptr = (int *)addr;

/* 正确：需要访问固定地址时使用uintptr_t */
uintptr_t addr = 0x1000;
int *ptr = (int *)addr;  /* 嵌入式场景，需平台确认 */
```

**警示**：⚠️ 整数与指针的相互转换是C标准中未定义行为（指针大小可能不等于 `int` 大小）。**64位平台问题**：在64位系统中，指针是8字节，`int` 是4字节，将指针赋值给 `int` 会截断高位地址。**正确做法**：使用 `uintptr_t`（定义在 `<stdint.h>`），它是一个足以容纳指针大小的无符号整数类型。

### 规则15.5 循环次数受外部数据控制时必须校验

**写法**：循环次数如果来自外部输入，必须在循环前进行合法性校验。

**示例**：
```c
/* 错误 */
for (int i = 0; i < user_count; i++) {  /* user_count可能极大 */
    /* ... */
}

/* 正确 */
if (user_count < 0 || user_count > MAX_ITERATIONS) {
    log_error("Invalid iteration count: %d", user_count);
    return ERR_INVALID_PARAM;
}
for (int i = 0; i < user_count; i++) {
    /* ... */
}
```

**警示**：⚠️ 外部数据控制的循环次数可能导致拒绝服务（DoS）攻击。如果 `user_count` 极大（如 `INT_MAX`），循环将消耗大量CPU时间，甚至导致系统无响应。**安全影响**：在嵌入式系统中，这也可能导致看门狗超时触发复位。

### 规则15.6 位域使用无符号类型

**写法**：位域必须使用无符号整数类型，有符号位域的行为是实现定义的。

**示例**：
```c
/* 错误：有符号位域的行为是实现定义的 */
struct {
    int flag : 1;  /* 可能为0或-1，而非0或1 */
};

/* 正确：使用无符号位域 */
struct {
    uint32_t flag : 1;  /* 确定：0或1 */
};
```

**警示**：⚠️ 有符号位域的行为由实现定义。`int flag : 1` 可以存储的值是0和-1（使用补码的机器），而不是0和1。**位域布局**：位域在内存中的布局（从高位到低位还是从低位到高位）也是实现定义的，不应用于跨平台数据传输。

### 建议15.1 使用标准库中的安全整数运算函数

**写法**：C11标准提供了安全整数运算函数，用于检测和防止溢出。

**示例**：
```c
#include <stdckd.h>

int a = 1000000, b = 2000000, result;
if (ckd_add(&result, a, b)) {
    /* 溢出处理 */
}
```

**警示**：⚠️ 手动编写溢出检查容易出错（如忘掉边界条件）。使用标准库函数可以：① 提高代码可读性；② 保证正确性；③ 编译器可能将 `ckd_add` 优化为平台相关的溢出检测指令（如 `CF=1` 检测）。

### 规则15.7 禁止隐式截断转换

**写法**：将大类型赋值给小类型时，必须显式转换，并在转换前检查范围，防止数据丢失。

**示例**：
```c
/* 错误：隐式截断，可能丢失数据 */
size_t len = get_file_size();
int n = len;  /* 若len > INT_MAX，n将为负数或截断值 */

/* 正确：先检查范围，再显式转换 */
size_t len = get_file_size();
if (len > INT_MAX) {
    return -EOVERFLOW;
}
int n = (int)len;  /* 已确认在范围内，安全 */
```

**警示**：⚠️ 隐式截断转换（CWE-197）是C语言中常见的安全漏洞来源。**典型场景**：`size_t`（64位系统上8字节）赋值给 `int`（4字节），当文件大小超过2GB时，`n` 将变为负数或截断，后续使用 `n` 作为缓冲区大小参数可能导致缓冲区溢出。**编译器警告**：开启 `-Wconversion` 可以检测这类问题。

### 规则15.8 禁止有符号数与无符号数混用比较

**写法**：有符号数和无符号数在同一表达式中比较时，有符号数会被隐式转换为无符号数，可能导致逻辑错误。必须统一类型后再比较。

**示例**：
```c
/* 危险：有符号数被隐式转换为无符号数 */
int count = get_count();  /* 可能为负数 */
if (count < sizeof(buf)) {  /* sizeof返回size_t（无符号） */
    /* count = -1 时，被转换为一个极大的无符号数，比较结果为false */
    /* 这与预期行为相反！ */
}

/* 正确：统一类型后再比较 */
if (count < 0 || (size_t)count < sizeof(buf)) {
    return ERR_INVALID;
}

/* 或使用更清晰的方式 */
if (count < 0) {
    return ERR_INVALID;
}
if ((size_t)count >= sizeof(buf)) {
    return ERR_BUFFER_TOO_SMALL;
}
```

**警示**：⚠️ 有符号/无符号混用比较（CWE-682）是C语言中最隐蔽的bug之一。**C标准规则**：当有符号类型和无符号类型在同一个表达式中时，有符号类型会被隐式转换为无符号类型。**典型错误**：`strlen(s) > signed_len` 当 `signed_len` 为负数时，`strlen` 的返回值（`size_t` 无符号）使得比较结果总是 `true`。**编译器警告**：开启 `-Wsign-compare`（GCC/Clang）可以检测这类问题。

### 规则15.9 禁止对指针变量进行sizeof操作

**写法**：`sizeof` 作用于指针变量时，得到的是指针本身的大小（32位平台为4，64位平台为8），而不是指针指向的数组的大小。

**示例**：
```c
/* 错误：对指针变量使用sizeof */
void process_data(const uint8_t *data)
{
    size_t size = sizeof(data);  /* 错误！得到的是指针大小，而非数组大小 */
    /* ... */
}

/* 正确：数组作为参数时必须同时传递长度 */
void process_data(const uint8_t *data, size_t data_len)
{
    /* 使用data_len而非sizeof(data) */
    /* ... */
}

/* 正确：对数组使用sizeof（仅在数组定义的作用域内有效） */
void process_array(void)
{
    uint8_t buffer[256];
    size_t size = sizeof(buffer);  /* 正确：256 */
    memset(buffer, 0, sizeof(buffer));  /* 正确用法 */
}
```

**警示**：⚠️ 对指针使用 `sizeof`（CWE-467）是C语言中最常见的错误之一。**典型场景**：函数参数中声明为 `int arr[10]` 的数组，在函数参数中会被退化为指针，`sizeof(arr)` 得到的是 `sizeof(int*)` 而非 `sizeof(int) * 10`。**经验法则**：数组作为函数参数传递时，必须同时传递数组长度。

### 规则15.10 使用固定宽度类型确保跨平台一致性

**写法**：涉及协议、存储、网络、硬件的代码，必须使用 `<stdint.h>` 中的固定宽度类型（`uint8_t`、`uint32_t`、`int64_t` 等），避免使用 `int`、`long` 等宽度不确定的类型。

**示例**：
```c
/* 错误：使用宽度不确定的类型 */
long timestamp;     /* 32位系统=4字节，64位系统=8字节 */
int  packet_len;    /* 16位系统=2字节，32位系统=4字节 */

/* 正确：使用固定宽度类型 */
uint64_t timestamp;     /* 始终8字节，跨平台一致 */
uint32_t packet_len;    /* 始终4字节，跨平台一致 */

/* 网络协议定义必须使用固定宽度类型 */
typedef struct {
    uint32_t  magic;        /* 魔数，固定4字节 */
    uint16_t  type;         /* 类型，固定2字节 */
    uint8_t   flags;        /* 标志位，固定1字节 */
    uint8_t   reserved;     /* 保留字节 */
    uint32_t  payload_len;  /* 负载长度，固定4字节 */
} __attribute__((packed)) packet_header_t;
```

**警示**：⚠️ `int` 类型在16位系统上是2字节，32位系统上是4字节，64位系统上也是4字节（LP64模型中为4字节）。`long` 类型在32位系统上是4字节，64位Unix系统上是8字节，64位Windows系统上是4字节。**依赖 `int`/`long` 宽度的代码在跨平台时可能产生难以发现的bug**。**推荐**：在涉及数据驻留（存储、网络、文件）的代码中，始终使用 `<stdint.h>` 的固定宽度类型。

---

## 16 文件I/O安全规范

### 规则16.1 创建文件时必须指定合适的访问权限

**写法**：使用 `open` 创建文件时，必须显式指定权限掩码，避免使用默认权限。

**示例**：
```c
/* 错误：默认权限可能过于宽松 */
FILE *fp = fopen("data.txt", "w");

/* 正确：使用open显式指定权限 */
int fd = open("data.txt", O_WRONLY | O_CREAT, S_IRUSR | S_IWUSR);
/* 仅所有者可读写，组和其他用户无权限 */
```

**警示**：⚠️ `fopen` 创建的文件的默认权限取决于 `umask` 设置，在不同的环境中可能不同。如果文件包含敏感信息（如密码、密钥、配置中的数据库密码），默认权限可能导致信息泄露。**最佳实践**：始终显式指定最小权限。

### 规则16.2 文件路径必须规范化后再使用

**写法**：用户提供的文件路径必须经过规范化处理，防止路径遍历攻击。

**示例**：
```c
/* 错误：直接使用用户提供的路径 */
FILE *fp = fopen(user_path, "r");  /* 可能包含 ../ 等路径遍历 */

/* 正确：规范化路径 */
char normalized_path[PATH_MAX];
if (realpath(user_path, normalized_path) == NULL) {
    log_error("Invalid path: %s", user_path);
    return ERR_INVALID_PATH;
}
/* 检查是否在允许的目录内 */
if (strncmp(normalized_path, ALLOWED_DIR, strlen(ALLOWED_DIR)) != 0) {
    log_error("Path outside allowed directory: %s", normalized_path);
    return ERR_PERMISSION;
}
```

**警示**：⚠️ 路径遍历攻击（CWE-22）允许攻击者读取任意文件。**攻击示例**：`user_path = "../../etc/passwd"` 可能读取系统密码文件。**防御层次**：① 规范化路径（去除 `..` 和 `.`）；② 检查路径是否在允许的目录内；③ 检查路径是否包含特殊字符。

### 规则16.3 避免TOCTOU竞争条件

**写法**：TOCTOU（Time-of-Check-Time-of-Use）指在检查和使用之间，文件可能被篡改，应直接打开文件并检查。

**示例**：
```c
/* 错误：检查和使用之间存在时间窗 */
if (access("config.ini", R_OK) == 0) {
    /* 攻击者可能在此时替换文件 */
    FILE *fp = fopen("config.ini", "r");
}

/* 正确：直接打开文件并检查 */
FILE *fp = fopen("config.ini", "r");
if (fp == NULL) {
    log_error("Cannot open config file");
    return ERR_OPEN;
}
```

**警示**：⚠️ TOCTOU（CWE-367）是一种经典的竞争条件漏洞。**攻击场景**：攻击者监控文件系统，在 `access` 和 `fopen` 之间将文件替换为符号链接，指向一个高权限文件。**防御**：直接打开文件并检查错误，而不是先检查权限再打开。

### 规则16.4 临时文件使用安全

**写法**：创建临时文件必须使用安全的临时文件创建函数，避免可预测的文件名。

**示例**：
```c
/* 错误：使用可预测的临时文件名 */
char tmpname[256];
snprintf(tmpname, sizeof(tmpname), "/tmp/tmp_%d", getpid());
FILE *fp = fopen(tmpname, "w+");  /* 易受符号链接攻击 */

/* 正确：使用安全临时文件函数 */
#ifdef PLATFORM_LINUX
    int fd = mkstemp("/tmp/myapp_XXXXXX");
#elif defined(PLATFORM_WINDOWS)
    char tmpname[256];
    GetTempFileNameA(".", "myapp", 0, tmpname);
    int fd = open(tmpname, O_CREAT | O_EXCL | O_RDWR, S_IRUSR | S_IWUSR);
#endif
```

**警示**：⚠️ 可预测的临时文件名容易受到符号链接攻击。**攻击示例**：攻击者预先创建一个与预测临时文件名相同的符号链接，指向 `/etc/passwd`，程序写入临时文件时就会覆盖系统文件。**安全函数**：`mkstemp` 会创建文件并返回文件描述符，且保证文件名唯一。

### 规则16.5 文件读写操作后检查ferror

**写法**：文件读写操作后应检查 `ferror` 判断是否发生I/O错误。

**示例**：
```c
FILE *fp = fopen("data.bin", "rb");
if (fp == NULL) {
    return ERR_OPEN;
}

size_t n = fread(buffer, 1, sizeof(buffer), fp);
if (n != sizeof(buffer)) {
    if (ferror(fp)) {
        log_error("Read error");
        fclose(fp);
        return ERR_IO;
    }
    /* EOF是正常情况，部分读取 */
}
```

**警示**：⚠️ `fread`/`fwrite` 返回的值小于请求的字节数，不一定表示错误——也可能是EOF。必须使用 `ferror` 和 `feof` 区分是错误还是文件结束。**注意**：`ferror` 不会自动清除错误标志，需要调用 `clearerr` 清除。

---

## 17 安全编程规范

### 原则17.1 攻击者思维与零信任原则

**写法**：编程过程中应时刻保持攻击者思维，假设所有外部数据都是不可信的、攻击者时刻试图监听、篡改、破坏程序运行环境。这是安全编程的指导思想，也是防御性编程的根基。

**示例**：

**攻击者思维的核心假设**：
```
假设1：程序所处理的所有外部数据都是不可信的攻击数据
假设2：攻击者时刻试图监听、篡改、破坏程序运行环境、外部数据
```

**外部数据定义（均视为不可信）**：
| 数据来源 | 示例 | 安全风险 |
|---------|------|---------|
| 文件（含配置文件） | 配置文件、日志文件、数据文件 | 路径遍历、格式篡改 |
| 网络 | socket数据、HTTP请求、RPC | 缓冲区溢出、注入攻击 |
| 环境变量 | PATH、LD_PRELOAD、HOME | 环境变量注入、路径劫持 |
| 命令行参数 | argv传入的参数 | 参数注入、缓冲区溢出 |
| 用户输入 | 键盘输入、UI输入 | 所有类型的注入攻击 |
| 用户态数据（内核程序） | ioctl参数、系统调用参数 | 权限提升、内存破坏 |
| 进程间通信 | 管道、消息队列、共享内存、socket | 数据篡改、TOCTOU |
| 函数参数（API） | 公共接口的所有参数 | 参数校验绕过 |
| 全局变量 | 多线程共享变量 | 数据竞争、TOCTOU |

**安全编码基本思想**：
1. **程序在处理外部数据时必须经过严格的合法性校验** — 不能做出任何外部数据符合预期的假设
2. **尽量减少代码的攻击面** — 代码实现应尽量简单，避免与外部环境做多余的数据交互
3. **通过防御性编码策略弥补潜在的编码人员疏忽**，包括：
   - 变量声明赋予初值
   - 谨慎使用全局变量
   - 禁用功能复杂、易用错的函数
   - 禁用易用错的编译器/操作系统机制
   - 小心处理资源访问过程
   - 不改变操作系统运行环境
   - 严格的错误处理
   - 合理使用调试断言(ASSERT)

**警示**：⚠️ 缺乏攻击者思维是安全漏洞的主要根源。**典型错误案例**：
- 开发者认为"只有内部系统才会使用此API，所以不需要校验"，实际上内部系统也可能被攻陷
- 开发者认为"用户输入已经过前端验证"，实际上攻击者可以绕过前端直接发送请求
- 开发者认为"配置文件的格式是固定的，不会出错"，实际上配置文件可能被篡改

**CWE相关**：CWE-20（输入验证不当）、CWE-345（数据完整性验证不足）、CWE-923（不安全的通信通道）

### 原则17.2 所有外部数据不可信（零信任原则）

**写法**：程序处理的所有外部数据（文件、网络、用户输入、环境变量、命令行参数、IPC等）都视为不可信，必须经过严格的合法性校验。这是安全编程的基石原则。

**示例**：

```c
/* 安全的数据处理流程：校验 → 处理 → 输出 */
int process_user_data(const char *input, size_t input_len)
{
    /* 第一步：校验（长度、格式、范围、字符集） */
    if (input == NULL || input_len == 0 || input_len > MAX_INPUT_LEN) {
        return ERR_INVALID_PARAM;
    }
    if (!is_valid_charset(input, input_len)) {
        return ERR_INVALID_CHARSET;
    }

    /* 第二步：处理（安全处理，使用边界检查） */
    char safe_buf[MAX_INPUT_LEN] = {0};
    memcpy_s(safe_buf, sizeof(safe_buf), input, input_len);

    /* 第三步：输出 */
    return process_internal(safe_buf);
}
```

```c
/* 典型错误：信任外部数据 */
char config_path[256] = "/etc/myapp/";
strcat(config_path, user_input);                    /* 无校验拼接 */
FILE *fp = fopen(config_path, "r");                  /* 路径遍历！ */
```

```c
/* 正确做法：校验后再使用 */
char config_path[256] = "/etc/myapp/";
if (strchr(user_input, '/') != NULL || strchr(user_input, '.') != NULL) {
    log_error("Invalid path component: %s", user_input);
    return ERR_INVALID_PATH;
}
strcat_s(config_path, sizeof(config_path), user_input);
FILE *fp = fopen(config_path, "r");
```

**警示**：⚠️ "不可信数据"原则是安全编程的基石。**常见错误**：开发者认为"只有恶意用户才会输入恶意数据"——实际上，一个看似善意的输入也可能因为编码问题、长度问题等导致程序崩溃。**零信任原则**：永远不要信任任何外部数据。**CWE-20**：输入验证不当。

### 规则17.1 禁止将外部数据直接作为进程启动函数的参数

**写法**：禁止将用户输入直接传递给 `system`、`popen` 等函数，应使用固定命令或白名单校验。

**示例**：
```c
/* 错误：命令注入 */
system(user_input);
popen(user_input, "r");

/* 正确：使用固定命令 */
system("ls -l /tmp");

/* 或对用户输入进行严格校验 */
if (!is_valid_command(user_input)) {
    log_error("Invalid command input");
    return ERR_INVALID_PARAM;
}
```

**警示**：⚠️ 命令注入漏洞（CWE-78）允许攻击者执行任意系统命令。**攻击示例**：如果 `user_input = "echo hello; rm -rf /"`，则 `system(user_input)` 会执行 `rm -rf /`。**防御**：优先使用 `execve` 系列函数，避免使用 `system` 和 `popen`。

### 规则17.2 禁止将外部数据直接作为模块加载函数的参数

**写法**：禁止将用户输入直接传递给 `dlopen`、`LoadLibrary` 等模块加载函数，必须使用白名单。

**示例**：
```c
/* 错误：任意代码加载 */
void *handle = dlopen(user_input, RTLD_NOW);  /* 可能加载任意.so */

/* 正确：只允许加载指定白名单中的库 */
if (!is_allowed_library(user_input)) {
    log_error("Unauthorized library: %s", user_input);
    return ERR_PERMISSION;
}
void *handle = dlopen(user_input, RTLD_NOW);
```

**警示**：⚠️ 任意代码加载漏洞允许攻击者执行任意代码。**攻击示例**：攻击者上传恶意 `.so` 文件，然后通过 `dlopen` 加载它。**防御**：使用白名单机制，只允许加载已审核的库文件。

### 规则17.3 禁止使用外部数据拼接SQL命令

**写法**：禁止使用字符串拼接构造SQL语句，必须使用参数化查询或对输入进行转义。

**示例**：
```c
/* 错误：SQL注入 */
char query[256];
snprintf(query, sizeof(query), "SELECT * FROM users WHERE name='%s'", user_input);

/* 正确：使用参数化查询（通过数据库API） */
/* 若无参数化查询支持，必须对输入进行转义 */
char *escaped = escape_sql_string(user_input);
snprintf(query, sizeof(query), "SELECT * FROM users WHERE name='%s'", escaped);
```

**警示**：⚠️ SQL注入（CWE-89）是最常见的Web应用漏洞之一。**攻击示例**：`user_input = "' OR '1'='1"` 可以绕过身份验证查询所有用户数据。**防御**：始终使用参数化查询（Prepared Statement），这是唯一安全的防御方式。

### 规则17.4 禁止在信号处理例程中调用非异步安全函数

**写法**：信号处理函数中只能调用异步安全函数（如 `write`、`_exit`），禁止调用 `printf`、`malloc`、`free` 等。

**警示**：⚠️ 信号处理函数中调用的函数必须是可重入的（reentrant）和异步信号安全的（async-signal-safe）。`printf` 使用全局缓冲区，`malloc` 使用全局堆管理，都不是异步安全的。**后果**：在信号处理函数中调用非异步安全函数可能导致死锁、数据损坏或崩溃。

### 规则17.5 禁用setjmp/longjmp

**写法**：`setjmp`/`longjmp` 跳过栈展开，导致资源泄漏和未定义行为，应使用正常返回值和错误码传递机制。

**警示**：⚠️ `setjmp`/`longjmp` 的主要问题：① 跳过栈展开，导致资源泄漏（`malloc` 分配的内存不会自动释放）；② 未定义行为：如果 `longjmp` 跳过了 `setjmp` 函数的作用域，行为是未定义的；③ 可读性差：类似 `goto` 的"大跳转"。**替代方案**：使用返回值和错误码。

### 规则17.6 敏感信息及时清零

**写法**：密码、密钥等敏感信息使用完毕后，必须立即清零，且使用 `memset_s` 而非 `memset` 防止编译器优化。

**示例**：
```c
/* 处理完密码等敏感信息后立即清零 */
char password[256];
/* ... 使用密码 ... */
memset_s(password, sizeof(password), 0, sizeof(password));
/* 注意：使用memset_s而非memset，防止编译器优化掉清零操作 */
```

**警示**：⚠️ 编译器可能会优化掉 `memset` 调用（因为"后续不再使用该内存"），导致敏感信息残留在内存中。`memset_s`（C11）和 `SecureZeroMemory`（Windows）保证不会被优化掉。**CWE-14**：编译器优化移除内存清零操作。

### 规则17.7 禁用rand()用于安全场景

**写法**：`rand()` 可预测，禁止用于密码、密钥、令牌等安全场景，应使用安全的随机数生成器。

**示例**：
```c
/* 错误：rand()可预测 */
int token = rand();

/* 正确：使用安全的随机数生成器 */
/* Unix: arc4random() / getrandom() */
/* Windows: CryptGenRandom() */
```

**警示**：⚠️ `rand()` 使用线性同余生成器（LCG），其输出是完全可预测的。如果攻击者知道种子或观察到几个输出值，就可以预测后续所有输出。**CWE-338**：使用可预测的随机数生成器。

### 规则17.8 输入验证白名单优先

**写法**：输入验证时，白名单（只允许已知合法的值）优于黑名单（禁止已知不合法的值）。

**示例**：
```c
/* 错误：黑名单校验，容易被绕过 */
if (strstr(user_input, "..") != NULL) {
    return ERR_INVALID_PATH;
}

/* 正确：白名单校验，只允许明确合法的字符 */
if (!is_valid_path_chars(user_input)) {
    return ERR_INVALID_PATH;
}
```

**警示**：⚠️ 黑名单永远无法穷举所有恶意输入。攻击者可以找到绕过黑名单的方法（如URL编码、Unicode编码、大小写混写等）。**白名单优势**：明确规定了"什么是合法的"，所有不在白名单中的输入都被拒绝，安全性更高。

### 规则17.9 防止缓冲区溢出：使用固定格式字符串

**写法**：`printf` 系列函数的格式化字符串必须为常量，禁止将用户输入作为格式化字符串。

**示例**：
```c
/* 错误：format string可控 */
void log_message(const char *msg)
{
    printf(msg);  /* 如果msg包含%s，会导致栈内容泄露 */
}

/* 正确 */
void log_message(const char *msg)
{
    printf("%s", msg);
}
```

**警示**：⚠️ 格式化字符串漏洞（CWE-134）是严重的安全漏洞。**攻击示例**：如果 `msg = "%x%x%x%x%x%x%x%x"`，`printf(msg)` 会泄露栈上的数据。如果 `msg = "%s%s%s%s"`，可能读取非法内存地址导致崩溃。**防御**：始终使用 `printf("%s", msg)` 而非 `printf(msg)`。

### 规则17.10 防止路径穿越攻击

**写法**：对文件路径操作中涉及的外部输入，必须进行白名单校验，禁止包含 `../`、`..\\` 等路径遍历字符。使用真实路径（realpath）规范化后再进行访问控制检查。

**示例**：
```c
/* 错误：路径遍历 */
char file_path[PATH_MAX] = "/var/data/";
strcat(file_path, user_input);  /* user_input = "../../etc/passwd" */
int fd = open(file_path, O_RDONLY);  /* 读取了系统密码文件！ */

/* 正确：白名单校验 */
const char *allowed_chars = "abcdefghijklmnopqrstuvwxyz0123456789._-";
if (strspn(user_input, allowed_chars) != strlen(user_input)) {
    log_error("Invalid path characters: %s", user_input);
    return ERR_INVALID_PATH;
}

/* 正确：使用realpath规范化路径后检查前缀 */
char *real_path = realpath(user_input, NULL);
if (real_path == NULL) {
    return ERR_INVALID_PATH;
}
if (strncmp(real_path, ALLOWED_BASE_DIR, strlen(ALLOWED_BASE_DIR)) != 0) {
    free(real_path);
    return ERR_ACCESS_DENIED;
}
free(real_path);
```

**警示**：⚠️ **路径穿越攻击（CWE-22）**允许攻击者读取或写入任意文件。**攻击场景**：Web服务器接受用户请求的文件名，攻击者输入 `../../../etc/passwd` 读取系统密码文件。**CVE案例**：CVE-2019-9978（WordPress目录遍历）。**防御**：① 使用白名单校验路径字符；② 使用 `realpath` 规范化路径；③ 检查规范化后的路径是否在允许的基目录内。

### 规则17.11 防止DLL/模块劫持

**写法**：加载动态库时必须使用绝对路径，禁止使用相对路径。Windows上调用 `SetDllDirectory("")` 删除当前目录的搜索路径。

**示例**：
```c
/* Windows DLL劫持防护 */
#ifdef _WIN32
    /* 在程序启动时移除当前目录的DLL搜索路径 */
    SetDllDirectory(TEXT(""));
    
    /* 使用绝对路径加载DLL */
    HMODULE hmod = LoadLibrary(TEXT("C:\\Program Files\\MyApp\\plugins\\myplugin.dll"));
    if (hmod == NULL) {
        log_error("Failed to load plugin DLL");
    }
#else
    /* Linux下使用绝对路径加载共享库 */
    void *handle = dlopen("/usr/lib/myapp/libplugin.so", RTLD_NOW);
    if (handle == NULL) {
        log_error("Failed to load plugin: %s", dlerror());
    }
#endif
```

**警示**：⚠️ **DLL劫持（CWE-114）**是Windows平台常见的高危漏洞。**攻击场景**：攻击者在程序当前工作目录下放置恶意DLL，当程序使用相对路径调用 `LoadLibrary` 时，优先加载了恶意DLL。**CVE案例**：CVE-2018-0824（Microsoft Office DLL劫持）。**防御**：① 使用绝对路径加载；② 调用 `SetDllDirectory("")`；③ 使用 `LOAD_LIBRARY_SEARCH_*` 标志。

### 规则17.12 防止Off-By-One错误

**写法**：字符串操作中，必须确保为终止空字符 `\0` 预留空间。使用 `strncat` 等函数时，注意剩余空间的计算。

**示例**：
```c
/* 错误：Off-By-One */
char firstname[20];
char lastname[20];
char fullname[40] = {0};

strncat(fullname, firstname, 20);  /* 可能写入20个字符不补\0 */
strncat(fullname, lastname, 20);   /* 第二次追加时，起始位置可能已无\0 */
/* 结果：fullname可能没有终止\0，导致后续strlen等函数越界 */

/* 正确：为\0预留空间 */
char firstname[20] = {0};
char lastname[20] = {0};
char fullname[40] = {0};

strncat(fullname, firstname, sizeof(fullname) - strlen(fullname) - 1);
strncat(fullname, lastname, sizeof(fullname) - strlen(fullname) - 1);
/* 每次追加都保留1字节用于\0终止符 */
```

**警示**：⚠️ **Off-By-One（CWE-193）**指缓冲区边界计算错误，比正确值多1或少1。**典型场景**：循环 `for (i = 0; i <= n; i++)` 使用了 `<=` 而非 `<`，导致多访问一次。**CVE案例**：CVE-2019-13117（libSLIRP off-by-one导致堆溢出）。**编译器检查**：开启 `-Wstringop-overflow`（GCC 10+）可检测部分问题。

### 规则17.13 防止整数溢出导致的内存破坏

**写法**：内存分配大小计算中涉及整数运算时，必须检查溢出，防止整数溢出导致分配过小缓冲区，进而造成缓冲区溢出。

**示例**：
```c
/* 错误：整数溢出导致缓冲区过小 */
void copy_elements(const char *input, size_t count, size_t element_size)
{
    size_t total = count * element_size;  /* 如果count和element_size都很大，可能溢出！ */
    char *buf = malloc(total);            /* 分配了远小于预期的缓冲区 */
    if (buf == NULL) return;
    memcpy(buf, input, total);            /* 缓冲区溢出！ */
}

/* 正确：乘法溢出检查 */
void copy_elements(const char *input, size_t count, size_t element_size)
{
    if (count == 0 || element_size == 0) {
        return;
    }
    if (count > SIZE_MAX / element_size) {
        log_error("Integer overflow: count=%zu, elem_size=%zu", count, element_size);
        return;  /* 拒绝溢出 */
    }
    size_t total = count * element_size;
    char *buf = malloc(total);
    if (buf == NULL) return;
    memcpy(buf, input, total);
}

/* 更好：使用C11安全运算函数 */
#include <stdckd.h>

void copy_elements_safe(const char *input, size_t count, size_t element_size)
{
    size_t total;
    if (ckd_mul(&total, count, element_size)) {
        log_error("Integer overflow");
        return;
    }
    char *buf = malloc(total);
    if (buf == NULL) return;
    memcpy(buf, input, total);
}
```

**警示**：⚠️ **整数溢出导致缓冲区溢出（CWE-190 → CWE-122）**是经典的安全漏洞链。**攻击场景**：攻击者构造 `count = 0x1000000`，`element_size = 0x1000`，乘积为 `0x1000000000`，在32位系统中溢出为 `0`，`malloc(0)` 返回小缓冲区，后续 `memcpy` 写入大量数据导致堆溢出。**CVE案例**：CVE-2018-4407（iOS/macOS内核整数溢出导致任意代码执行）。

### 规则17.14 防止敏感信息泄露：禁止栈上未初始化变量

**写法**：栈上变量声明时必须初始化，防止未初始化的内存包含敏感数据（如之前函数调用留下的栈数据）。

**示例**：
```c
/* 错误：未初始化变量泄露栈数据 */
void process_user(void)
{
    char buf[256];  /* 未初始化，包含栈上残留数据 */
    /* ... 仅部分写入 */
    write(fd, buf, sizeof(buf));  /* 泄露了栈上残留数据（可能包含其他函数的密码等） */
}

/* 正确：声明时初始化 */
void process_user(void)
{
    char buf[256] = {0};  /* 全部初始化为0 */
    /* ... 仅部分写入 */
    write(fd, buf, sizeof(buf));  /* 未写入的部分为0，不会泄露数据 */
}

/* 正确：使用前清零 */
void process_user_safe(void)
{
    char buf[256];
    memset(buf, 0, sizeof(buf));
    /* ... 填充数据 ... */
    write(fd, buf, sizeof(buf));
}
```

**警示**：⚠️ **未初始化内存泄露（CWE-457）**可能导致敏感信息泄露。**攻击场景**：函数A中处理了密码（存储在栈上），函数A返回后，函数B在同一栈位置声明了未初始化的局部变量，通过函数B的输出泄露了密码。**CVE案例**：CVE-2019-13751（Chrome未初始化内存泄露）。**编译器检查**：开启 `-Wuninitialized`（GCC/Clang的 `-Wall` 中已包含）。

### 规则17.15 防止任意地址写

**写法**：写入内存时，必须确保写入的地址是合法且预期的。禁止将外部数据直接作为内存写入的目标地址。

**示例**：
```c
/* 错误：任意地址写 */
typedef struct {
    char *buf;
    size_t len;
} write_buf_t;

void write_payload(write_buf_t *dst, const char *payload, size_t len)
{
    memcpy(dst->buf, payload, len);  /* 如果dst->buf由外部控制，可写入任意地址！ */
}

int main(void)
{
    write_buf_t dst;
    dst.buf = (char *)user_controlled_addr;  /* 攻击者控制的地址 */
    /* 通过memcpy向任意地址写入数据 */
}
```

**警示**：⚠️ **任意地址写（CWE-123）**是最严重的内存破坏漏洞之一，通常可直接导致代码执行。**攻击场景**：攻击者通过控制结构体中的指针成员，利用 `memcpy` 等函数向任意地址写入任意数据。**防御**：结构体中的指针成员不应由外部数据直接控制，必须进行合法性验证。

---

## 18 并发编程规范

### 规则18.1 所有共享资源的访问必须加锁

**写法**：多线程环境下，所有共享变量的读写必须通过互斥锁保护，防止数据竞争（data race）。

**示例**：
```c
/* 错误：无锁访问共享变量 */
int g_counter = 0;

void increment(void)
{
    g_counter++;  /* 非原子操作，多线程下不安全 */
}

/* 正确：加锁保护 */
pthread_mutex_t g_mutex = PTHREAD_MUTEX_INITIALIZER;
int g_counter = 0;

void increment(void)
{
    pthread_mutex_lock(&g_mutex);
    g_counter++;
    pthread_mutex_unlock(&g_mutex);
}
```

**警示**：⚠️ 数据竞争（Data Race）是C标准中未定义行为。两个线程同时读写同一变量，即使只是简单的 `counter++`，也可能导致结果错误。`counter++` 在汇编层面是"读-改-写"三步，两个线程可能同时读取到相同的值，各加1后写回，导致最终结果少加1。**编译检查**：ThreadSanitizer（`-fsanitize=thread`）可以检测数据竞争。

### 规则18.2 避免死锁：固定锁获取顺序

**写法**：所有线程必须以相同的顺序获取锁，避免死锁。

**示例**：
```c
/* 错误：不同线程以不同顺序获取锁，可能死锁 */
/* 线程1 */
pthread_mutex_lock(&lock_a);
pthread_mutex_lock(&lock_b);

/* 线程2 */
pthread_mutex_lock(&lock_b);
pthread_mutex_lock(&lock_a);

/* 正确：所有线程以相同顺序获取锁 */
/* 线程1和线程2都先获取lock_a，再获取lock_b */
```

**警示**：⚠️ 死锁是并发编程中最常见的问题之一。**死锁四条件**：① 互斥；② 持有并等待；③ 不可剥夺；④ 循环等待。破坏"循环等待"最有效的方法就是固定锁获取顺序。**调试**：使用 `-fsanitize=thread` 或 `helgrind`（Valgrind工具）检测死锁。

### 规则18.3 谨慎使用不可重入函数

**写法**：多线程环境下应使用可重入版本的函数，避免使用不可重入函数。

**示例**：
```c
/* 常见不可重入函数及其可重入替代 */
/* strtok     → strtok_r */
/* rand       → rand_r */
/* gmtime     → gmtime_r */
/* localtime  → localtime_r */
/* asctime    → asctime_r */
/* ctime      → ctime_r */
```

**警示**：⚠️ 不可重入函数使用静态或全局缓冲区存储状态。多线程同时调用时，状态会被覆盖，导致数据损坏。**注意**：即使当前看似"只有一个线程调用"，后续代码修改时可能引入多线程调用，因此应始终使用可重入版本。

### 规则18.4 使用原子操作代替锁（适用场景）

**写法**：对于简单的计数器、标志位等场景，使用原子操作替代锁，减少锁竞争开销。

**示例**：
```c
#include <stdatomic.h>

atomic_int g_counter = 0;

void increment(void)
{
    atomic_fetch_add(&g_counter, 1);  /* 原子操作，无需锁 */
}
```

**警示**：⚠️ 原子操作适用于简单场景，但**不适用于复合操作**（如"先读后写"涉及多个字段的场景）。**内存序**：默认使用 `memory_order_seq_cst`（最严格的内存序），性能可能不如更宽松的内存序。**注意**：原子操作不是"万能替代锁"——复杂的数据结构仍需使用锁保护。

### 规则18.5 锁的作用域应最小化

**写法**：只对必要的共享资源访问加锁，不将耗时操作放在锁范围内。

**示例**：
```c
/* 错误：锁的粒度过大 */
void process_data(data_t *data)
{
    pthread_mutex_lock(&g_mutex);
    /* 复杂计算，可能耗时很长 */
    int result = heavy_computation(data);
    g_shared_state = result;
    pthread_mutex_unlock(&g_mutex);
}

/* 正确：只对共享资源访问加锁 */
void process_data(data_t *data)
{
    int result = heavy_computation(data);  /* 无锁计算 */
    pthread_mutex_lock(&g_mutex);
    g_shared_state = result;
    pthread_mutex_unlock(&g_mutex);
}
```

**警示**：⚠️ 锁的粒度过大会导致线程竞争激烈，降低并发性能。**锁的粒度原则**：锁的范围越小，并发性能越好。**但注意**：锁的粒度过细（每个字段一把锁）可能导致死锁风险和性能开销增加。

### 规则18.6 使用条件变量避免忙等待

**写法**：等待条件满足时，使用条件变量阻塞线程，避免CPU空转。

**示例**：
```c
/* 错误：忙等待，浪费CPU */
while (!g_data_ready) {
    /* 空转，浪费CPU */
}

/* 正确：使用条件变量 */
pthread_mutex_lock(&g_mutex);
while (!g_data_ready) {
    pthread_cond_wait(&g_cond, &g_mutex);
}
pthread_mutex_unlock(&g_mutex);
```

**警示**：⚠️ 忙等待（busy waiting）是并发编程中最常见的性能问题。**注意**：`pthread_cond_wait` 必须在 `while` 循环中调用，而非 `if`——因为条件变量可能被**虚假唤醒**（spurious wakeup）。**虚假唤醒**：即使没有线程调用 `pthread_cond_signal`，`pthread_cond_wait` 也可能返回。

### 规则18.7 线程退出前必须释放所有资源

**写法**：线程退出前必须释放所有持有的资源，包括堆内存、文件描述符、锁等。

**示例**：
```c
void *worker_thread(void *arg)
{
    thread_data_t *data = (thread_data_t *)arg;

    /* 工作循环 */
    while (!g_shutdown) {
        /* 处理任务 */
    }

    /* 退出前清理 */
    free(data->buffer);
    close(data->fd);
    return NULL;
}
```

**警示**：⚠️ 线程退出时未释放资源会导致资源泄漏。**注意**：① 如果线程被强制终止（如 `pthread_cancel`），可能无法执行清理代码；② 使用 `pthread_cleanup_push`/`pthread_pop` 注册清理函数，确保线程被取消时也能释放资源。

### 建议18.1 尽量减少锁的粒度

**写法**：使用细粒度锁替代粗粒度锁，提高并发性能。

**示例**：
```c
/* 粗粒度锁 */
pthread_mutex_lock(&big_lock);
/* 操作资源A */
/* 操作资源B */
/* 操作资源C */
pthread_mutex_unlock(&big_lock);

/* 细粒度锁（更好） */
pthread_mutex_lock(&lock_a);
/* 操作资源A */
pthread_mutex_unlock(&lock_a);

pthread_mutex_lock(&lock_b);
/* 操作资源B */
pthread_mutex_unlock(&lock_b);
```

**警示**：⚠️ 细粒度锁提高了并发性能，但增加了死锁风险。**经验法则**：不要过早优化锁粒度。先用粗粒度锁保证正确性，通过性能分析确认瓶颈后再优化。

---

## 19 Volatile与内存屏障规范

### 规则19.1 volatile用于以下场景

**写法**：`volatile` 用于以下四种场景：① 被中断处理程序修改的全局变量；② 被信号处理程序修改的全局变量；③ 内存映射寄存器（MMIO）；④ 多线程环境中的标志变量（仅限单次读取/写入的简单场景）。

**示例**：
```c
/* 中断处理程序修改的标志 */
volatile int g_timer_expired = 0;

void timer_isr(void)
{
    g_timer_expired = 1;  /* 中断中修改，主循环中读取 */
}

void main_loop(void)
{
    while (!g_timer_expired) {
        /* 等待定时器中断 */
    }
    process_timeout();
}
```

**警示**：⚠️ `volatile` 告诉编译器"不要优化对该变量的访问"，确保每次读取都从内存读取，而非从寄存器缓存。**但 volatile 不提供原子性**：如果中断和主循环同时修改同一个 volatile 变量，仍可能产生数据竞争。**注意**：`volatile` 不能替代原子操作或锁。

### 规则19.2 volatile不能替代原子操作或锁

**写法**：`volatile` 不能保证操作的原子性，也不提供任何内存序保证。

**示例**：
```c
/* 错误：volatile不能保证自增操作的原子性 */
volatile int g_counter = 0;
g_counter++;  /* 仍然是读-改-写三步，非原子 */

/* 正确：使用原子操作 */
atomic_int g_counter = 0;
atomic_fetch_add(&g_counter, 1);
```

**警示**：⚠️ 这是一个常见的误解。`volatile` 只保证"每次读取都从内存读取"，但 `g_counter++` 在汇编层面仍然是"读-改-写"三步，在多核环境下可能被其他线程打断。**C11 原子操作**：`atomic_int` 和 `atomic_fetch_add` 提供了原子性和内存序保证。

### 规则19.3 使用内存屏障确保内存访问顺序

**写法**：在多线程编程中，使用内存屏障（memory barrier）确保内存访问顺序的正确性。

**示例**：
```c
#include <stdatomic.h>

int g_data = 0;
atomic_int g_ready = 0;

/* 生产者线程 */
void producer(void)
{
    g_data = 42;
    atomic_store_explicit(&g_ready, 1, memory_order_release);
    /* release屏障：保证g_data的写入在g_ready之前完成 */
}

/* 消费者线程 */
void consumer(void)
{
    while (atomic_load_explicit(&g_ready, memory_order_acquire) == 0) {
        /* 等待 */
    }
    /* acquire屏障：保证g_ready读取后的g_data读取是最新的 */
    process(g_data);
}
```

**警示**：⚠️ 编译器优化和CPU乱序执行可能导致代码的实际执行顺序与源代码顺序不同。**Release-Acquire 语义**：`memory_order_release` 保证"本原子操作之前的所有写操作，对其他线程的 `memory_order_acquire` 读操作可见"。**注意**：错误的内存序可能导致难以调试的并发bug。

### 规则19.4 禁止使用volatile进行同步

**写法**：`volatile` 不提供任何内存序保证，不能用于线程间同步。

**警示**：⚠️ `volatile` 在不同线程间不提供任何同步保证。**常见误解**：认为 `volatile` 变量在多个线程间的读写是安全的。实际上，`volatile` 只防止编译器优化，不防止CPU乱序执行，也不提供原子性。**正确做法**：线程间同步必须使用原子操作或互斥锁。

---

## 20 中断处理规范

### 规则20.1 中断服务程序（ISR）应尽量短小

**写法**：ISR 中只做最必要的操作（如设置标志位），实际处理在主循环中完成。

**示例**：
```c
/* 错误：ISR中执行复杂操作 */
void timer_isr(void)
{
    complex_computation();  /* ISR中不应执行耗时操作 */
    log_to_file("timer fired");  /* 文件I/O可能阻塞 */
}

/* 正确：ISR中只设置标志，实际处理在主循环中 */
void timer_isr(void)
{
    g_timer_expired = 1;  /* 设置标志 */
}

void main_loop(void)
{
    if (g_timer_expired) {
        g_timer_expired = 0;
        complex_computation();  /* 主循环中处理 */
    }
}
```

**警示**：⚠️ ISR 应遵循"最短执行时间"原则。**原因**：① 长时间占用CPU会阻塞低优先级中断；② ISR 中执行阻塞操作（如文件I/O）可能触发系统崩溃；③ 在实时系统中，ISR 过长可能导致任务错过截止时间。**经验法则**：ISR 中只做"设置标志、清除中断、读/写寄存器"等操作。

### 规则20.2 ISR中访问的全局变量必须声明为volatile

**写法**：ISR 和主循环共享的全局变量必须声明为 `volatile`。

**示例**：
```c
/* 正确：ISR和主循环共享的变量 */
volatile uint32_t g_tick_count = 0;

void systick_isr(void)
{
    g_tick_count++;
}

uint32_t get_tick_count(void)
{
    return g_tick_count;
}
```

**警示**：⚠️ 没有 `volatile` 关键字，编译器可能优化掉对 `g_tick_count` 的读取，将其缓存到寄存器中，导致主循环无法感知 ISR 中的修改。**注意**：如果 `g_tick_count` 在 ISR 中多次修改（如 `++` 操作），`volatile` 不能保证原子性——在 32 位系统上，32 位变量的读取和写入通常是原子的，但 64 位变量不是。

### 规则20.3 ISR中禁止调用非可重入函数

**写法**：ISR 中只能调用可重入的、异步信号安全的函数。

**示例**：
```c
/* 错误：ISR中调用不可重入函数 */
void isr(void)
{
    printf("ISR fired\n");  /* printf可能不可重入 */
    char *token = strtok(NULL, ",");  /* strtok不可重入 */
}

/* 正确：ISR中只做最基本的操作 */
void isr(void)
{
    g_flag = 1;
}
```

**警示**：⚠️ 非可重入函数使用全局或静态缓冲区，ISR 中断了主循环的执行，如果主循环恰好也在调用同一个非可重入函数，会导致数据损坏。**安全函数列表**：`write`、`read`、`_exit`、`sig_atomic_t` 操作等是异步信号安全的。

### 规则20.4 嵌套中断应配置优先级

**写法**：嵌套中断应配置合适的优先级，避免优先级反转和栈溢出。

**警示**：⚠️ 嵌套中断的中断处理顺序：高优先级中断可以打断低优先级中断。**问题**：① 如果嵌套深度过大，可能导致栈溢出；② 优先级反转：低优先级中断持有高优先级中断需要的资源，导致高优先级中断被阻塞。**建议**：尽量减少嵌套中断的使用，保持中断优先级不超过 2 层。

---

## 21 日志与诊断规范

### 规则21.1 使用统一的日志等级

**写法**：项目中应使用统一的日志等级定义，不同等级对应不同的严重程度。

**示例**：
```c
typedef enum {
    LOG_LEVEL_FATAL   = 0,  /* 致命错误，系统不可用 */
    LOG_LEVEL_ERROR   = 1,  /* 错误，功能不可用 */
    LOG_LEVEL_WARNING = 2,  /* 警告，可能的问题 */
    LOG_LEVEL_INFO    = 3,  /* 重要信息 */
    LOG_LEVEL_DEBUG   = 4,  /* 调试信息 */
    LOG_LEVEL_TRACE   = 5,  /* 详细跟踪 */
} log_level_e;
```

**警示**：⚠️ 日志等级的合理使用是日志管理的核心。**常见问题**：① 所有日志都使用 `ERROR` 等级，导致真正的错误被淹没；② 生产环境开启 `DEBUG` 等级，导致日志量过大和性能下降；③ 日志等级不一致，不同模块使用不同的等级定义。

### 规则21.2 日志应包含时间戳和模块名

**写法**：日志应包含时间戳、模块名、文件名、行号等上下文信息。

**示例**：
```c
/* 日志格式：[时间戳] [等级] [模块] 消息 */
#define LOG_ERROR(mod, fmt, ...) \
    log_write(LOG_LEVEL_ERROR, mod, __FILE__, __LINE__, fmt, ##__VA_ARGS__)
```

**警示**：⚠️ 没有上下文信息的日志在生产环境中几乎无法定位问题。**最少信息**：每条日志至少应包含：① 时间戳（精确到毫秒）；② 日志等级；③ 模块名；④ 消息内容。**可选信息**：文件名、行号、函数名、线程ID、进程ID。

### 规则21.3 日志不应包含敏感信息

**写法**：日志中不得记录密码、密钥、令牌、个人身份信息（PII）等敏感数据。

**示例**：
```c
/* 错误：日志中记录密码 */
LOG_INFO("auth", "Login with password=%s", password);

/* 正确：密码等敏感信息不能记录到日志 */
LOG_INFO("auth", "Login attempt for user=%s", username);
```

**警示**：⚠️ 日志中记录敏感信息是严重的安全问题（CWE-532）。**后果**：① 日志文件泄露可能导致密码泄露；② 违反GDPR等数据保护法规。**审计**：代码审查时应特别关注日志中是否包含敏感信息。

### 规则21.4 调试日志在Release版本中应被移除

**写法**：调试日志（`DEBUG`、`TRACE` 等级）在 Release 版本中应通过宏控制移除。

**示例**：
```c
#ifdef DEBUG
    #define LOG_DEBUG(mod, fmt, ...) \
        log_write(LOG_LEVEL_DEBUG, mod, __FILE__, __LINE__, fmt, ##__VA_ARGS__)
#else
    #define LOG_DEBUG(mod, fmt, ...)  ((void)0)
#endif
```

**警示**：⚠️ 调试日志在 Release 版本中不应产生任何运行时开销。**注意**：`#define LOG_DEBUG(...) ((void)0)` 确保调试日志在 Release 版本中被完全移除，但参数中的表达式不会被求值。**另一种方法**：运行时通过日志级别控制，但性能开销略大。

### 规则21.5 日志宏的参数不能有副作用

**写法**：传递给日志宏的参数不能有副作用，因为调试日志在 Release 版本中可能被移除。

**示例**：
```c
/* 错误：宏参数有副作用 */
LOG_DEBUG("test", "value=%d", counter++);  /* DEBUG版执行，Release版不执行 */

/* 正确：日志参数不应有副作用 */
LOG_DEBUG("test", "value=%d", counter);
counter++;
```

**警示**：⚠️ 这是日志宏最隐蔽的bug来源。**后果**：调试版和发布版行为不一致，导致在调试版中发现的bug无法在发布版中复现。**经验法则**：日志宏的参数中永远不要包含赋值、自增、函数调用等副作用操作。

### 建议21.1 使用日志级别控制输出量

**写法**：运行时可通过配置动态调整日志级别，避免在生产环境中输出过多调试信息。

**警示**：⚠️ 生产环境建议使用 `WARNING` 或 `INFO` 级别。`DEBUG` 和 `TRACE` 级别在大型系统中每秒可能产生数百MB日志，导致磁盘填满和性能严重下降。**原则**：生产环境日志越少越好，但少到不能定位问题时又不行——需要权衡。

---

## 22 API设计与版本管理规范

### 规则22.1 API函数应使用统一的前缀

**写法**：API 函数名应使用模块前缀，格式为 `<模块>_<操作>`。

**示例**：
```c
/* 模块前缀：<模块>_<操作> */
int tcp_connect(const char *host, uint16_t port);
int tcp_send(tcp_handle_t handle, const void *data, size_t len);
int tcp_recv(tcp_handle_t handle, void *buffer, size_t len);
void tcp_close(tcp_handle_t handle);
```

**警示**：⚠️ 统一前缀是C语言中实现"命名空间"的主要方式。**原因**：C语言没有命名空间机制，所有全局符号共享同一个命名空间。没有前缀的函数名容易和其他模块冲突。**注意**：前缀也应用于类型名、宏、枚举常量等。

### 规则22.2 API函数应保持语义一致性

**写法**：API 中相同语义的操作应使用相同的动词，保持一致性。

**示例**：
```c
/* 好的API：语义一致 */
int user_create(const char *name, uint32_t *out_id);
int user_delete(uint32_t id);
int user_find(uint32_t id, user_info_t *info);

/* 不好的API：语义不一致 */
int create_user(const char *name, uint32_t *out_id);
int remove_user(uint32_t id);  /* 不一致：delete vs remove */
int get_user_info(uint32_t id, user_info_t *info);  /* 不一致：find vs get */
```

**警示**：⚠️ 语义不一致的 API 加重了使用者的记忆负担。**统一动词表**：项目中应定义统一的动词表，如 `create`/`delete`/`find`/`update`，所有 API 函数必须遵循。

### 规则22.3 API版本管理

**写法**：API 头文件中应声明版本宏，并提供版本查询函数。

**示例**：
```c
/* 头文件中声明API版本 */
#define USER_API_VERSION_MAJOR  1
#define USER_API_VERSION_MINOR  2
#define USER_API_VERSION_PATCH  0

/**
 * @brief 获取API版本号
 * @return 版本号编码：(major << 16) | (minor << 8) | patch
 */
uint32_t user_get_api_version(void);
```

**警示**：⚠️ API 版本管理是接口演进的基础。**语义化版本控制**：主版本号（Major）：不兼容的API修改；次版本号（Minor）：向后兼容的功能新增；修订号（Patch）：向后兼容的bug修复。**注意**：版本号应在头文件中声明，而非仅在代码注释中说明。

### 规则22.4 废弃API的处理

**写法**：废弃API必须使用 `@deprecated` 标记，并指明替代方案，至少保留 2 个主版本后方可移除。

**示例**：
```c
/**
 * @brief 旧版用户创建函数（已废弃）
 * @deprecated 请使用 user_create() 替代
 * @note 本函数将在 v3.0 中移除
 */
int create_user_legacy(const char *name, int len, uint32_t *out_id)
{
    /* 内部直接调用新接口 */
    return user_create(name, out_id);
}
```

**警示**：⚠️ 废弃API不能直接移除，必须给用户足够的迁移时间。**废弃策略**：① 标记为 `deprecated`；② 在日志中输出警告信息；③ 至少保留 2 个主版本；④ 在文档中明确说明移除时间。**注意**：废弃API的内部实现应直接调用新接口，避免维护两份代码。

### 规则22.5 API二进制兼容性

**写法**：修改已发布的API时，必须保持二进制兼容性（ABI兼容性）。

**示例**：
```c
/* 兼容的修改：添加新字段到结构体末尾 */
typedef struct {
    uint32_t id;
    char name[64];
    /* v2.0 新增字段 */
    uint32_t flags;     /* 新字段，必须初始化为0 */
    uint64_t timeout;   /* 新字段，必须初始化为0 */
} user_info_v2_t;

/* 不兼容的修改：修改已有字段大小或顺序 */
/* 错误：修改字段大小会破坏ABI */
typedef struct {
    uint64_t id;        /* 原为uint32_t，破坏ABI */
    char name[64];
} user_info_broken_t;
```

**警示**：⚠️ ABI 破坏是最隐蔽的 API 问题。**ABI兼容的修改**：① 在结构体末尾添加新字段；② 添加新函数；③ 放宽函数参数约束。**ABI不兼容的修改**：① 修改已有结构体字段大小；② 删除结构体字段；③ 修改结构体字段顺序；④ 修改函数参数类型。**注意**：ABI 破坏通常只在运行时才会暴露出来（如段错误）。

### 规则22.6 API错误码稳定性

**写法**：API 的错误码一经发布不得修改语义。新增错误码只能追加到枚举末尾。

**示例**：
```c
typedef enum {
    USER_ERR_SUCCESS        = 0,
    USER_ERR_NOT_FOUND      = -1,
    USER_ERR_DUPLICATE      = -2,
    USER_ERR_PERMISSION     = -3,
    /* v2.0 新增 */
    USER_ERR_BANNED         = -4,
    USER_ERR_RATE_LIMITED   = -5,
} user_err_e;
```

**警示**：⚠️ 错误码语义改变会导致调用者的错误处理逻辑出错。**重要规则**：① 错误码的值一旦发布，永久固定；② 新的错误码只能追加到枚举末尾；③ 禁止删除错误码（即使不再使用，也应保留定义）；④ 禁止修改错误码的语义含义。

### 建议22.1 提供API封装层

**写法**：对于可能变更的底层实现，应提供封装层，隔离外部依赖变化。

**示例**：
```c
/* 封装层接口 */
int storage_write(const char *key, const void *data, size_t len);
int storage_read(const char *key, void *buffer, size_t *len);

/* 内部实现可变：支持文件存储、数据库、KV存储等切换 */
```

**警示**：⚠️ 封装层是隔离外部依赖变更的关键设计。**好处**：① 底层实现变更时，调用者代码无需修改；② 便于测试（可以 Mock 封装层）；③ 降低模块耦合度。**注意**：封装层不应泄露底层实现细节（如文件名、数据库表名等）。

## 23 测试规范

### 原则23.1 测试代码与生产代码同等重要

**写法**：测试代码同样需要遵循编码规范、代码审查和维护。测试不是一次性工作，需要随代码同步更新。

**警示**：⚠️ 测试代码的质量直接影响产品质量。低质量的测试代码（如冗余、重复、难以理解的测试）会导致：① 测试维护成本高；② 开发人员不愿意修改测试；③ 测试失去可信度，最终被废弃。**原则**：测试代码的维护成本不应超过生产代码。

### 规则23.1 单元测试必须覆盖所有公开API

**写法**：每个对外公开的函数应至少有一个正向测试用例和一个负向测试用例。

**示例**：
```c
/* 正向测试：正常参数 */
void test_tcp_connect_normal(void)
{
    int fd = tcp_connect("127.0.0.1", 8080, 5000);
    TEST_ASSERT(fd >= 0);
    close(fd);
}

/* 负向测试：无效参数 */
void test_tcp_connect_invalid_host(void)
{
    int fd = tcp_connect(NULL, 8080, 5000);
    TEST_ASSERT(fd < 0);
}
```

**警示**：⚠️ 只测试正向路径的测试是"虚假安全感"。**测试类型**：① 正向测试（正常参数 → 期望成功）；② 负向测试（无效参数 → 期望失败）；③ 边界测试（边界值 → 期望正确行为）；④ 异常测试（异常条件 → 期望优雅处理）。**覆盖标准**：每个公开API函数必须至少有一个正向和一个负向测试。

### 规则23.2 测试代码与生产代码分离

**写法**：测试代码必须放在独立的目录中，与生产代码分离。

**示例**：
```
project/
├── src/            # 生产代码
│   ├── tcp_client.c
│   └── tcp_client.h
├── test/           # 测试代码
│   ├── test_tcp_client.c
│   └── test_main.c
```

**警示**：⚠️ 测试代码与生产代码混合会导致：① 生产代码的发布包中包含测试代码；② 编译配置混乱；③ 测试代码无法独立运行。**注意**：测试代码可以访问生产代码的内部接口（通过 `#include` 或链接），但反过来生产代码不能依赖测试代码。

### 规则23.3 测试用例命名规范

**写法**：测试用例函数的命名格式为 `test_<模块>_<功能>_<场景>`。

**示例**：
```c
/* 格式：test_<模块>_<功能>_<场景> */
void test_tcp_connect_normal(void);
void test_tcp_connect_timeout(void);
void test_tcp_connect_refused(void);
void test_user_create_normal(void);
void test_user_create_duplicate_name(void);
void test_user_create_null_name(void);
```

**警示**：⚠️ 测试用例命名不清晰时，测试失败后难以定位问题。**好的测试命名**应该描述：① 测试什么功能；② 在什么场景下；③ 期望的行为是什么。**反例**：`test1()`、`test_case_1()`、`test_function()`。

### 规则23.4 测试必须是可重复的

**写法**：测试不能依赖外部环境状态（如网络可达性、系统时间、文件存在性），应使用Mock/Stub隔离外部依赖。

**示例**：
```c
/* 错误：依赖外部网络 */
void test_http_request(void)
{
    int result = http_get("https://example.com/api");  /* 网络不可达则测试失败 */
    TEST_ASSERT(result == 0);
}

/* 正确：Mock网络层 */
void test_http_request(void)
{
    /* 替换网络层为Mock实现 */
    network_mock_set_response("{\"status\":\"ok\"}");
    int result = http_get("https://example.com/api");
    TEST_ASSERT(result == 0);
}
```

**警示**：⚠️ 不可重复的测试是"脆弱的测试"。**常见问题**：① 依赖网络：网络不可达时测试失败；② 依赖系统时间：测试在特定时间才通过；③ 依赖文件系统：文件不存在时测试失败；④ 依赖全局状态：测试执行顺序不同导致结果不同。**解决**：使用Mock/Stub隔离所有外部依赖。

### 规则23.5 测试必须独立

**写法**：每个测试用例执行前后必须清理状态，测试用例之间不能有依赖关系。

**示例**：
```c
/* 每个测试前执行 */
void setUp(void)
{
    g_user_count = 0;
    memset(g_users, 0, sizeof(g_users));
}

/* 每个测试后执行 */
void tearDown(void)
{
    /* 清理所有资源 */
}
```

**警示**：⚠️ 测试用例之间的依赖性是测试维护的噩梦。**问题**：测试 B 依赖测试 A 执行后的状态，如果测试 A 失败，测试 B 也会失败，但错误信息是误导性的。**原则**：测试用例应可独立运行，且可以任意顺序执行。

### 规则23.6 代码覆盖率要求

**写法**：项目必须设定代码覆盖率目标，并在CI流程中检查。

**示例**：
```c
/* 语句覆盖：≥90%（核心模块 ≥95%） */
/* 分支覆盖：≥80%（核心模块 ≥90%） */
/* 函数覆盖：100%（所有公开API函数） */

/* 分支覆盖示例：需要测试if和else两个分支 */
int parse_config(const char *path)
{
    if (path == NULL) {      /* 分支1：NULL路径 */
        return ERR_INVALID_PARAM;
    }
    /* 分支2：正常路径 */
    return do_parse(path);
}
```

**警示**：⚠️ 覆盖率只是指标，不是目标。**常见误区**：① 追求100%覆盖率而编写无意义的测试（如只调用函数但不验证结果）；② 覆盖率达标但测试质量差（没有验证边界条件）；③ 只关注语句覆盖而忽略分支覆盖和条件覆盖。**好的覆盖率策略**：语句覆盖 ≥ 90%，分支覆盖 ≥ 80%，函数覆盖 100%。

### 规则23.7 边界值测试

**写法**：所有输入参数必须进行边界值测试，包括最小值、最大值、空值、超长值等。

**示例**：
```c
/* 测试空字符串 */
void test_user_create_empty_name(void)
{
    int result = user_create("", &id);
    TEST_ASSERT(result == ERR_INVALID_PARAM);
}

/* 测试超长字符串 */
void test_user_create_overflow_name(void)
{
    char long_name[256];
    memset(long_name, 'A', sizeof(long_name) - 1);
    long_name[sizeof(long_name) - 1] = '\0';
    int result = user_create(long_name, &id);
    TEST_ASSERT(result == ERR_INVALID_PARAM);
}

/* 测试最大长度字符串 */
void test_user_create_max_name(void)
{
    char max_name[MAX_NAME_LEN + 1];
    memset(max_name, 'A', MAX_NAME_LEN);
    max_name[MAX_NAME_LEN] = '\0';
    int result = user_create(max_name, &id);
    TEST_ASSERT(result == 0);
}
```

**警示**：⚠️ 大多数Bug发生在边界值附近。**边界值测试原则**：① 测试最小值（0）；② 测试最大值（MAX）；③ 测试最小值-1；④ 测试最大值+1；⑤ 测试空值（NULL/空字符串）；⑥ 测试特殊值（负数、NaN、INF等）。**经验法则**：边界值测试能发现约 80% 的输入相关Bug。

### 建议23.1 使用测试框架

**写法**：推荐使用成熟的测试框架，禁止自建测试框架。

**示例**：
```
| 平台 | 推荐框架 | 说明 |
|------|---------|------|
| 通用 | Unity / CMock | 轻量级C语言测试框架 |
| 通用 | Check | 成熟的C语言测试框架 |
| Embedded | Ceedling | 嵌入式C项目测试框架 |
| Windows | WinUnit | Windows平台测试框架 |
```

**警示**：⚠️ 自建测试框架是常见的"非我发明"（NIH）综合征。**问题**：① 自建框架需要大量维护工作；② 功能不完善（缺少断言、Mock、报告生成等）；③ 团队成员需要学习两套框架。**建议**：使用成熟的测试框架，如有特殊需求，可以基于现有框架扩展。

### 建议23.2 测试驱动开发（TDD）

**写法**：对于核心逻辑模块，建议采用TDD方式：先编写测试用例，再实现功能代码。

**警示**：⚠️ TDD 并非适用于所有场景。**适用场景**：① 核心业务逻辑；② 算法类模块；③ 数据处理模块。**不适用场景**：① UI/界面代码；② 原型开发阶段；③ 功能需求不明确时。**TDD的好处**：① 迫使代码可测试；② 测试覆盖率高；③ 保证代码按预期工作。

### 规则23.8 可测性设计：函数纯净化

**写法**：核心逻辑函数应设计为纯函数——输入决定输出，无副作用，不依赖全局状态。纯函数天然易于测试。

**示例**：
```c
/* 错误：难以测试——依赖全局状态和外部调用 */
static int g_factor = 0;
int calculate(int input)
{
    return input * g_factor;  /* 依赖全局变量，测试前需要设置g_factor */
}

/* 正确：易于测试——纯函数，输入决定输出 */
int calculate(int input, int factor)
{
    return input * factor;  /* 纯函数，测试时只需传入不同参数 */
}

/* 测试时无需设置全局状态，直接调用 */
void test_calculate_normal(void)
{
    TEST_ASSERT(calculate(5, 2) == 10);
    TEST_ASSERT(calculate(0, 100) == 0);
    TEST_ASSERT(calculate(-5, 2) == -10);
}
```

**警示**：⚠️ 纯函数是可测试性最好的代码形式。**评判标准**：一个函数是否纯函数——相同输入是否总是产生相同输出？是否修改了外部状态？**可测性改造**：将依赖全局状态、文件系统、网络等的逻辑分离到外层，核心逻辑保持纯函数。

### 规则23.9 可测性设计：接口依赖注入

**写法**：对于必须依赖外部资源（文件、网络、数据库）的代码，使用接口抽象，使测试时可以通过Mock替换外部依赖。

**示例**：
```c
/* 可测性设计：定义接口抽象 */
typedef struct storage_ops {
    int (*read)(void *ctx, const char *key, char *buf, size_t len);
    int (*write)(void *ctx, const char *key, const char *data, size_t len);
} storage_ops_t;

/* 生产代码：依赖注入 */
int process_data(storage_ops_t *ops, void *ctx, const char *user_id)
{
    char user_data[1024] = {0};
    int ret = ops->read(ctx, user_id, user_data, sizeof(user_data));
    if (ret != 0) {
        return ret;
    }
    /* 处理数据 */
    return transform_user_data(user_data);
}

/* 测试：使用Mock实现 */
int mock_read(void *ctx, const char *key, char *buf, size_t len)
{
    const char *test_data = "{\"name\":\"test\"}";
    strncpy(buf, test_data, len - 1);
    buf[len - 1] = '\0';
    return 0;
}

/* 测试时注入Mock */
void test_process_data_normal(void)
{
    storage_ops_t mock_ops = { .read = mock_read, .write = NULL };
    int ret = process_data(&mock_ops, NULL, "user_001");
    TEST_ASSERT(ret == 0);
}
```

**警示**：⚠️ 依赖注入模式将依赖关系从编译时解耦到运行时，使得测试时可以用Mock替代真实依赖。**设计原则**：高层模块不应依赖低层模块，两者都应依赖抽象。**常见模式**：① 函数指针结构体（如上面的例子）；② 回调函数注册；③ 编译时接口替换（通过条件编译）。

### 规则23.10 可测性设计：避免静态初始化依赖

**写法**：避免使用静态初始化顺序依赖，这种依赖在测试中难以控制。使用显式初始化函数替代。

**示例**：
```c
/* 错误：静态初始化顺序依赖 */
static int g_config_loaded = 0;
static Config g_config;

/* 在其他模块中自动调用——初始化顺序不可控 */
__attribute__((constructor))
void init_config(void) {
    load_config("config.ini", &g_config);
    g_config_loaded = 1;
}

/* 正确：使用显式初始化函数 */
int g_config_loaded = 0;
Config g_config;

int config_init(const char *path)
{
    /* 可重复调用，且可在测试中重新初始化 */
    memset(&g_config, 0, sizeof(g_config));
    int ret = load_config(path, &g_config);
    if (ret == 0) {
        g_config_loaded = 1;
    }
    return ret;
}

void config_cleanup(void) {
    /* 清理资源，测试中可重新初始化 */
    free_config(&g_config);
    g_config_loaded = 0;
}
```

**警示**：⚠️ 静态初始化顺序是C++中著名的"静态初始化顺序灾难"问题，C语言中同样存在。**问题**：静态初始化函数的执行顺序在同一编译单元中可控，但跨编译单元不可控。**测试影响**：测试框架可能无法控制静态初始化顺序，导致测试在特定环境下失败。**建议**：避免使用 `__attribute__((constructor))`，使用显式 `init()`/`cleanup()` 函数。

---

## 24 构建与编译规范

### 规则24.1 编译器警告必须全部消除

**写法**：项目编译必须开启最高警告级别，并将警告视为错误。

**示例**：
```makefile
# GCC/Clang
CFLAGS += -Wall -Wextra -Wpedantic -Werror
CFLAGS += -Wshadow -Wformat=2 -Wstrict-overflow=5
CFLAGS += -Wstrict-prototypes -Wmissing-prototypes
CFLAGS += -Wconversion -Wsign-conversion

# MSVC
CFLAGS += /W4 /WX
CFLAGS += /wd4820  # 如需禁用特定警告，需注明原因
```

**警示**：⚠️ 编译器警告是编译器发现潜在Bug的最直接反馈。**常见错误态度**：① "这个警告不重要，先忽略"——累积的警告会掩盖真正的Bug；② "这个警告是误报"——即使误报也应通过类型转换或`#pragma`明确消除。**原则**：`-Werror` 确保零警告，任何警告都必须被消除或明确豁免。

### 规则24.2 禁止在编译输出中包含路径信息

**写法**：编译输出中不应包含绝对路径信息，防止泄露构建环境信息。

**示例**：
```makefile
# 错误：编译输出中包含绝对路径
CFLAGS += -g  # 默认包含绝对路径

# 正确：使用相对路径或映射到源路径
CFLAGS += -g -fdebug-prefix-map=$(PWD)=.
```

**警示**：⚠️ 编译输出中的绝对路径可能泄露：① 构建者的用户名；② 构建目录结构；③ 内部路径信息。**安全风险**：发布二进制文件中的路径信息可能被攻击者用于了解内部系统结构。

### 规则24.3 使用统一的构建系统

**写法**：项目必须使用统一的构建系统，禁止混合使用不同构建工具。

**示例**：
```makefile
# 推荐：CMake（跨平台）
cmake_minimum_required(VERSION 3.16)
project(myproject C)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

# 编译选项
add_compile_options(-Wall -Wextra -Wpedantic -Werror)
add_compile_options(-O2 -DNDEBUG)
```

**警示**：⚠️ 混合使用多个构建系统（如Makefile + CMake + 自定义脚本）会导致：① 构建配置不一致；② 新人难以理解构建流程；③ CI配置复杂化。**建议**：新项目统一使用 CMake，已有项目逐步迁移。

### 规则24.4 构建必须可重现

**写法**：同一份源码在相同环境下应产生完全相同的二进制输出。

**示例**：
```makefile
# GCC可重现构建
CFLAGS += -ffile-prefix-map=$(PWD)=.
CFLAGS += -frandom-seed=1
CFLAGS += -Wno-psabi  # 忽略ABI警告

# 链接选项
LDFLAGS += -Wl,--build-id=sha1  # 确定性build-id
```

**警示**：⚠️ 不可重现的构建导致：① 无法确认二进制是否确实由声称的源码产生；② 调试困难（无法在另一个环境复现问题）；③ 安全审计失败。**要求**：使用 `-frandom-seed=1` 和 `-ffile-prefix-map` 等选项确保可重现性。

### 规则24.5 禁止在构建过程中下载依赖

**写法**：所有依赖必须在构建前准备好，构建过程不应访问网络。

**示例**：
```makefile
# 错误：构建过程中下载依赖
wget https://example.com/libfoo.tar.gz
tar xzf libfoo.tar.gz

# 正确：使用vendored依赖或预先安装
# 将依赖的源码放入 third_party/ 目录
```

**警示**：⚠️ 构建过程中下载依赖导致：① 构建不可重现（远程文件可能变化）；② 构建失败（网络不可达）；③ 安全风险（下载的代码可能被篡改）。**解决方案**：将依赖的源码放入 `third_party/` 目录，或使用包管理器锁定版本。

### 规则24.6 区分Debug和Release构建

**写法**：Debug 和 Release 构建配置必须明确区分，使用不同的编译选项。

**示例**：
```makefile
# Debug构建
# CFLAGS: -O0 -g -DDEBUG
# 启用断言、调试日志、asan/lsan/ubsan

# Release构建
# CFLAGS: -O2 -DNDEBUG
# 禁用断言、调试日志，启用优化

# 安全要求的Release构建还需启用
CFLAGS += -fstack-protector-strong  # 栈保护
CFLAGS += -D_FORTIFY_SOURCE=2       # 缓冲区溢出检测
CFLAGS += -fPIE -pie                # 位置无关可执行
CFLAGS += -z relro -z now           # 只读重定位
```

**警示**：⚠️ Debug 和 Release 构建混淆导致：① Debug 版本发布到生产环境（性能差）；② Release 版本缺少调试信息（无法调试）。**安全开关**：Release 构建必须启用安全选项：`-fstack-protector-strong`、`-D_FORTIFY_SOURCE=2`、`-fPIE -pie`、`-z relro -z now`。

### 规则24.7 使用静态分析工具

**写法**：CI 流程中必须集成静态分析工具。

**示例**：
```makefile
# 推荐工具
# - Cppcheck: 通用静态分析
# - Clang Static Analyzer: 深度路径分析
# - Flawfinder: 安全漏洞扫描
# - Coverity: 企业级分析（可选）

# Makefile中集成cppcheck
static-check:
    cppcheck --enable=all --suppress=missingIncludeSystem \
             --std=c11 --error-exitcode=1 src/
```

**警示**：⚠️ 静态分析能发现编译器无法检测的深层问题。**静态分析能发现的问题**：① 空指针解引用；② 使用未初始化变量；③ 缓冲区溢出；④ 内存泄漏；⑤ 死代码。**CI集成**：静态分析结果必须作为 CI 的检查项，未通过则阻止合并。

### 规则24.8 使用地址消毒剂（AddressSanitizer）

**写法**：Debug 构建必须启用 AddressSanitizer 进行运行时检测。

**示例**：
```makefile
CFLAGS += -fsanitize=address -fsanitize=undefined
CFLAGS += -fsanitize=leak -fsanitize=null
LDFLAGS += -fsanitize=address -fsanitize=undefined
```

**警示**：⚠️ AddressSanitizer（ASan）是 Google 开发的运行时内存错误检测工具。**能检测的问题**：① 堆缓冲区溢出；② 栈缓冲区溢出；③ 全局缓冲区溢出；④ 使用已释放内存（use-after-free）；⑤ 内存泄漏。**注意**：ASan 会使程序运行变慢约 2x，但能发现大量内存错误，建议在 Debug 构建和测试中始终启用。

### 建议24.1 CI/CD流水线检查清单

**写法**：每次提交应通过一系列自动化检查。

**示例**：
```
1. 编译通过（无警告）
2. 单元测试全部通过
3. 静态分析无严重问题
4. 代码风格检查通过
5. 代码覆盖率不下降
6. 内存泄漏检测（Valgrind/ASan）
```

**警示**：⚠️ CI 流水线是代码质量的最后一道防线。**常见问题**：① CI 检查项过多导致提交频繁失败——应平衡检查项的数量和重要性；② CI 检查时间过长——应优化并行执行；③ CI 检查结果被忽略——应禁止跳过 CI 检查合并代码。

---

## 25 代码审查规范

### 原则25.1 所有代码变更必须经过审查

**写法**：任何代码提交（包括Bug修复、重构、测试代码）都必须经过至少一名审查者审核后方可合并。

**警示**：⚠️ 代码审查是发现Bug和保证代码质量的最有效手段之一。**研究发现**：代码审查能发现约 60% 的缺陷，远高于测试（约 30%）。**例外**：① 紧急安全修复（需事后补审）；② 自动生成的代码（如protobuf生成的代码）；③ 纯配置变更（如CI配置）。

### 规则25.1 审查者要求

**写法**：审查者必须具有被审查代码所在模块的领域知识，且不能是代码作者。

**示例**：
```
- 审查者必须具有被审查代码所在模块的领域知识
- 审查者必须是代码作者之外的人
- 核心模块必须由至少两名审查者审查
- 审查者应在 24小时内 响应审查请求
```

**警示**：⚠️ 审查者不具备领域知识时，审查效果大打折扣。**常见问题**：① 审查者只看格式不看逻辑；② 审查者不理解代码意图，只能给出"LGTM"（Looks Good To Me）的无效审查；③ 审查者回复太慢，导致代码停滞。

### 规则25.2 代码变更量控制

**写法**：单次代码审查的变更量应控制在合理范围内，超过 400 行应拆分为多个提交。

**示例**：
```c
/* 建议：单次提交 <= 400行变更 */
/* 超过400行：应拆分为多个逻辑独立的提交 */
/* 超过1000行：必须拆分为多个提交 */
```

**警示**：⚠️ 研究表明，一次审查超过 400 行代码时，缺陷发现率显著下降。**原因**：① 审查者注意力有限，超过 400 行后遗漏率升高；② 大变更难以理解上下文；③ 审查者容易产生"审查疲劳"而草率通过。

### 规则25.3 审查清单（Checklist）

**写法**：代码审查应使用标准化的审查清单，确保覆盖所有关键检查项。

**示例**：
#### 25.3.1 逻辑正确性
- [ ] 算法逻辑是否正确？
- [ ] 边界条件是否处理？
- [ ] 错误路径是否覆盖？
- [ ] 并发安全是否考虑？

#### 25.3.2 安全性
- [ ] 所有外部输入是否校验？
- [ ] 缓冲区大小是否安全？
- [ ] 是否存在命令注入/SQL注入风险？
- [ ] 敏感信息是否妥善处理？

#### 25.3.3 内存管理
- [ ] 所有malloc是否有对应的free？
- [ ] 错误路径是否释放已分配的内存？
- [ ] 是否存在内存泄漏或悬垂指针？
- [ ] 栈大小是否合理？

#### 25.3.4 可维护性
- [ ] 命名是否清晰自注释？
- [ ] 函数是否过长（超过50行）？
- [ ] 是否存在重复代码？
- [ ] 注释是否准确且必要？

#### 25.3.5 性能
- [ ] 是否存在不必要的拷贝？
- [ ] 锁的粒度是否合理？
- [ ] 是否存在性能退化风险？

**警示**：⚠️ 没有标准化审查清单的审查容易遗漏关键问题。**建议**：将审查清单集成到 PR 模板中，审查者逐项勾选确认。

### 规则25.4 审查回复规范

**写法**：审查意见应使用标准化的格式标记问题类型和严重程度。

**示例**：
```c
/* 审查意见格式：[类型] 位置: 描述 */

/* 必须修复（Must Fix） */
[MUST] tcp_client.c:123: 缺少空指针检查，ptr可能为NULL

/* 建议修复（Should Fix） */
[SHOULD] user_manager.c:456: 函数过长（82行），建议拆分为子函数

/* 建议（Suggestion） */
[SUGGEST] config.c:78: 可考虑使用const指针替代值传递

/* 问题（Question） */
[QUESTION] task_scheduler.c:234: 此处使用while(1)是否考虑添加退出条件？
```

**警示**：⚠️ 模糊的审查意见无法有效指导修改。**好的审查意见**：① 明确指出问题位置；② 说明问题原因；③ 给出修改建议。**反例**："这段代码不好"、"重构一下"、"这里有问题"。

### 规则25.5 审查通过标准

**写法**：代码合并必须满足明确的审查通过标准。

**示例**：
```
- 所有 [MUST] 类型问题已修复
- 至少一名审查者给予 Approved 评价
- 无未解决的 [QUESTION] 类型问题
```

**警示**：⚠️ 审查标准不明确导致：① 审查者在"是否通过"上意见分歧；② 代码带着未解决的问题合并；③ 审查者随意给出 Approved。

### 规则25.6 审查者责任

**写法**：审查者需对合并后的代码质量承担共同责任。

**示例**：
```
- 审查者需对合并后的代码质量承担共同责任
- 审查者应关注代码逻辑、安全性、可维护性，而非代码风格（代码风格应由自动化工具保证）
- 审查者发现无法确认的问题时，应标注并请求更多审查者参与
```

**警示**：⚠️ 审查者应避免"橡皮图章式审查"（rubber stamping）——不经仔细阅读就通过。**审查者的责任**：① 理解代码变更；② 思考潜在问题；③ 确认测试覆盖；④ 对合并后的代码质量负责。

### 建议25.1 自动化审查前置

**写法**：在人工审查之前，应通过自动化工具完成基础检查，减少人工审查负担。

**示例**：
```bash
# 1. 代码风格检查
clang-format --dry-run --Werror src/*.c

# 2. 静态分析
cppcheck --enable=all src/

# 3. 编译检查
make clean && make -j$(nproc)

# 4. 单元测试
make test
```

**警示**：⚠️ 人工审查的精力应该用在逻辑正确性、安全性、可维护性等"只有人才能判断"的问题上。**代码风格**、**编译警告**、**静态分析**等应由自动化工具完成，不应占用人工审查时间。

---

## 26 跨平台开发规范

### 原则26.1 平台差异必须通过抽象层隔离

**写法**：禁止在业务代码中直接使用平台相关的API，应通过平台抽象层统一封装。

**示例**：
```c
/* 错误：业务代码中直接使用平台API */
#ifdef _WIN32
    HANDLE fd = CreateFileA(path, GENERIC_READ, 0, NULL, OPEN_EXISTING, 0, NULL);
#else
    int fd = open(path, O_RDONLY);
#endif

/* 正确：通过平台抽象层 */
int fd = os_file_open(path, OS_FILE_READ);
```

**警示**：⚠️ 业务代码中直接使用平台API导致：① 代码可移植性差——移植到新平台需要修改大量代码；② 代码可读性差——`#ifdef` 散落在业务逻辑中；③ 测试困难——无法在非目标平台测试。**解决方案**：为每个平台差异点创建抽象层接口。

### 规则26.1 平台抽象层接口规范

**写法**：平台抽象层应定义平台无关的接口，并在不同平台分别实现。

**示例**：
```c
/* 平台抽象层：os_file.h */
#ifndef OS_FILE_H
#define OS_FILE_H

#include <stdint.h>
#include <stddef.h>

/* 文件句柄（平台无关） */
typedef struct os_file_t os_file_t;

/* 打开文件 */
os_file_t *os_file_open(const char *path, int flags);

/* 读取文件 */
int32_t os_file_read(os_file_t *file, void *buffer, size_t size);

/* 写入文件 */
int32_t os_file_write(os_file_t *file, const void *data, size_t size);

/* 关闭文件 */
void os_file_close(os_file_t *file);

/* 获取文件大小 */
int64_t os_file_get_size(os_file_t *file);

#endif /* OS_FILE_H */
```

```c
/* 平台抽象层实现：os_file_linux.c */
#ifdef __linux__
os_file_t *os_file_open(const char *path, int flags)
{
    int fd = open(path, flags, 0644);
    /* ... */
}
#endif

/* 平台抽象层实现：os_file_windows.c */
#ifdef _WIN32
os_file_t *os_file_open(const char *path, int flags)
{
    HANDLE h = CreateFileA(path, ...);
    /* ... */
}
#endif
```

**警示**：⚠️ 平台抽象层应保持接口最小化——只暴露必要的API，避免过度设计。**原则**：① 接口应使用平台无关的类型（如 `int32_t` 而非 `int`）；② 接口行为应在所有平台一致；③ 不同平台的实现文件应独立编译。

### 规则26.2 使用标准类型保证跨平台一致性

**写法**：使用 `stdint.h` 中定义的固定宽度整数类型，避免使用平台相关类型。

**示例**：
```c
/* 错误：使用平台相关类型 */
int fd;              /* 不同平台int大小不同 */
long offset;         /* 不同平台long大小不同 */
unsigned int addr;   /* 不同平台unsigned int大小不同 */

/* 正确：使用标准类型 */
int32_t fd;          /* 固定32位 */
int64_t offset;      /* 固定64位 */
uintptr_t addr;      /* 指针大小的无符号整数 */
```

**警示**：⚠️ 平台相关类型在不同平台的大小不同：`int` 在 16 位系统上是 16 位，在 32/64 位系统上是 32 位；`long` 在 32 位系统上是 32 位，在 64 位 Linux 上是 64 位，但在 64 位 Windows 上仍是 32 位。**使用 `int32_t`/`int64_t` 等固定宽度类型确保跨平台一致性**。

### 规则26.3 字节序处理

**写法**：网络传输和数据存储必须使用网络字节序（大端），通过 `htonl`/`ntohl` 等函数转换。

**示例**：
```c
#include <arpa/inet.h>  /* Linux */
/* 或 */
#include <winsock2.h>   /* Windows */

/* 发送端：主机字节序转为网络字节序 */
uint32_t value = htonl(host_value);
uint16_t port = htons(host_port);

/* 接收端：网络字节序转为主机字节序 */
uint32_t host_value = ntohl(net_value);
uint16_t host_port = ntohs(net_port);
```

```c
/* 自定义数据的字节序转换 */
typedef struct {
    uint32_t id;       /* 需要转换 */
    uint16_t length;   /* 需要转换 */
    uint8_t  flags;    /* 不需要转换 */
    char     data[256];/* 不需要转换 */
} net_packet_t;

void packet_to_net(net_packet_t *pkt)
{
    pkt->id = htonl(pkt->id);
    pkt->length = htons(pkt->length);
}

void packet_to_host(net_packet_t *pkt)
{
    pkt->id = ntohl(pkt->id);
    pkt->length = ntohs(pkt->length);
}
```

**警示**：⚠️ 字节序问题是跨平台开发中最隐蔽的Bug之一。**大端 vs 小端**：x86/ARM 是小端（低位字节在低地址），网络协议通常使用大端。**常见错误**：① 只在小端平台上测试，未发现字节序问题；② 忘记对结构体中的某些字段进行转换；③ 对单字节字段也进行转换。**最佳实践**：创建专门的序列化/反序列化函数处理字节序。

### 规则26.4 条件编译宏的使用规范

**写法**：平台检测宏统一在单个配置头文件中定义，业务代码只使用平台抽象宏。

**示例**：
```c
/* 平台检测宏统一在单个配置头文件中定义 */
/* platform.h */
#if defined(__linux__)
    #define PLATFORM_LINUX   1
#elif defined(_WIN32) || defined(_WIN64)
    #define PLATFORM_WINDOWS 1
#elif defined(__APPLE__)
    #define PLATFORM_MACOS   1
#else
    #error "Unsupported platform"
#endif
```

```c
/* 业务代码中只使用平台抽象宏，不直接使用编译器宏 */
/* 正确 */
#include "platform.h"
#if defined(PLATFORM_LINUX)
    /* Linux特定实现 */
#elif defined(PLATFORM_WINDOWS)
    /* Windows特定实现 */
#endif

/* 错误 */
#ifdef __linux__
    /* 直接使用编译器宏 */
#endif
```

**警示**：⚠️ 直接使用编译器宏（`__linux__`、`_WIN32` 等）导致：① 平台检测逻辑散落在各文件中，难以管理和修改；② 编译器宏可能因编译器版本不同而不同；③ 无法在非目标平台进行编译测试。**建议**：统一在 `platform.h` 中定义平台宏，业务代码只引用 `platform.h`。

### 规则26.5 文件路径处理

**写法**：使用统一的路径分隔符宏和路径处理函数，避免硬编码路径分隔符。

**示例**：
```c
/* 使用统一路径分隔符宏 */
#ifdef PLATFORM_WINDOWS
    #define PATH_SEPARATOR  '\\'
    #define PATH_SEPARATOR_STR  "\\"
#else
    #define PATH_SEPARATOR  '/'
    #define PATH_SEPARATOR_STR  "/"
#endif

/* 使用路径处理函数，而非字符串拼接 */
char full_path[PATH_MAX];
int ret = os_path_join(base_dir, file_name, full_path, sizeof(full_path));
```

**警示**：⚠️ Windows 使用反斜杠 `\` 作为路径分隔符，Linux/macOS 使用正斜杠 `/`。**常见问题**：① Windows 上硬编码 `/` 路径分隔符——某些Windows API可能不接受；② 路径长度限制——Windows 路径最大 260 字符（可启用长路径支持）；③ 驱动器和挂载点差异——Windows 使用 `C:\`，Linux 使用 `/`。**建议**：使用跨平台路径处理库（如 `std::filesystem` 或自定义封装）。

### 规则26.6 线程和同步原语抽象

**写法**：线程和同步原语应通过平台抽象层封装，提供统一的接口。

**示例**：
```c
/* 平台无关的线程抽象 */
typedef struct os_thread_t os_thread_t;

typedef void *(*os_thread_func_t)(void *arg);

int os_thread_create(os_thread_t **thread, os_thread_func_t func, void *arg);
int os_thread_join(os_thread_t *thread, void **retval);
void os_thread_detach(os_thread_t *thread);
void os_thread_sleep(uint32_t milliseconds);

/* 平台无关的互斥锁抽象 */
typedef struct os_mutex_t os_mutex_t;

int os_mutex_create(os_mutex_t **mutex);
int os_mutex_lock(os_mutex_t *mutex);
int os_mutex_unlock(os_mutex_t *mutex);
void os_mutex_destroy(os_mutex_t *mutex);
```

**警示**：⚠️ 不同平台的线程API差异很大。**主要差异**：① 线程创建参数（Linux 的 `pthread_create` vs Windows 的 `CreateThread`）；② 线程标识（`pthread_t` vs `DWORD`）；③ 同步原语（`pthread_mutex_t` vs `CRITICAL_SECTION`）。**建议**：封装后提供统一的接口，隐藏平台差异。

### 建议26.1 避免使用非标准C扩展

**写法**：避免使用编译器特定的非标准扩展，如需使用应通过宏封装。

**示例**：
```c
/* 避免使用的非标准扩展 */
asm("int 0x80");           /* 内联汇编，不可移植 */
__declspec(dllexport);     /* MSVC特定 */
__attribute__((packed));   /* GCC特定 */
__fastcall;                /* 调用约定，不可移植 */

/* 使用标准替代或宏封装 */
#ifdef PLATFORM_WINDOWS
    #define DLL_EXPORT  __declspec(dllexport)
#else
    #define DLL_EXPORT  __attribute__((visibility("default")))
#endif
```

**警示**：⚠️ 非标准C扩展在不同编译器间不兼容。**常见问题**：① GCC 的 `__attribute__` 在 MSVC 中不可用；② MSVC 的 `__declspec` 在 GCC 中不可用；③ 内联汇编语法完全不可移植。**建议**：优先使用标准C特性，非标准扩展通过宏封装并提供跨平台实现。

---

## 27 性能优化规范

### 原则27.1 先正确，再优化

**写法**：性能优化必须在代码正确性验证之后进行。未经测试的优化代码是最大的性能陷阱。

**警示**：⚠️ 优化代码往往比未优化的代码更复杂，更容易引入Bug。**常见错误**：① 在代码正确性未验证前就进行优化；② 优化引入了Bug，但调试成本远高于优化带来的收益；③ 优化后的代码无法通过测试。**基本流程**：先编写正确、清晰的代码 → 编写测试验证正确性 → 性能分析定位瓶颈 → 针对性优化 → 再次验证正确性。

### 原则27.2 先测量，再优化

**写法**：没有性能数据的优化是盲目优化。必须通过性能分析工具定位瓶颈后再优化。

**示例**：
```bash
# 推荐工具
# - perf (Linux): CPU热点分析
# - Valgrind/Callgrind: 调用图分析
# - gprof: 函数级性能分析
# - VTune (Intel): 高级性能分析
```

**警示**：⚠️ 编程中的"直觉"常常是错误的。**常见误区**：① 开发者认为的"瓶颈"往往不是实际瓶颈；② 优化了非热路径的代码，总性能提升不到 1%；③ 优化使代码可读性下降，但性能提升微乎其微。**原则**：90% 的执行时间花在 10% 的代码上——找到那 10% 再优化。

### 规则27.1 避免过早优化

**写法**：先写出清晰、正确的代码，编译器会处理大部分常规优化。

**示例**：
```c
/* 错误：过早优化，降低可读性 */
int *p = (int *)((char *)base + offset * sizeof(int) + 8);  /* 手动计算偏移 */

/* 正确：先写出清晰的代码 */
int *p = &base[offset + 2];  /* 编译器会优化 */
```

**警示**：⚠️ 过早优化是万恶之源（Donald Knuth）。**过早优化的问题**：① 增加了代码复杂度，降低可读性和可维护性；② 优化可能针对错误的方向；③ 编译器优化能力已经很强，很多手动优化是多余的。**建议**：先写出清晰代码，通过性能分析确定瓶颈后再优化。

### 规则27.2 减少不必要的内存拷贝

**写法**：避免不必要的数据拷贝，使用指针或引用传递大对象。

**示例**：
```c
/* 错误：不必要的拷贝 */
void process_data(const char *input)
{
    char buffer[1024];
    memcpy(buffer, input, strlen(input) + 1);  /* 不必要的拷贝 */
    /* 使用buffer处理 */
    parse(buffer);
}

/* 正确：直接使用原数据 */
void process_data(const char *input)
{
    parse(input);  /* 避免拷贝 */
}
```

**警示**：⚠️ 内存拷贝是性能杀手——尤其是大块数据拷贝。**常见场景**：① 函数间传递大结构体时使用值传递而非指针传递；② 在循环中重复拷贝数据；③ 使用 `memcpy` 复制仅需读取的数据。

### 规则27.3 优先使用栈内存而非堆内存

**写法**：小对象优先使用栈内存分配，避免堆分配的开销。

**示例**：
```c
/* 错误：小对象使用堆内存 */
void process(void)
{
    char *buffer = malloc(64);  /* 64字节，堆分配开销大 */
    /* 使用buffer */
    free(buffer);
}

/* 正确：使用栈内存 */
void process(void)
{
    char buffer[64];  /* 栈分配，几乎无开销 */
    /* 使用buffer */
}
```

**警示**：⚠️ 堆分配（`malloc`/`free`）的开销远大于栈分配。**经验法则**：① 小于 4KB 的临时缓冲区使用栈分配；② 栈分配几乎无开销（仅修改栈指针）；③ 但注意栈空间有限（通常 1-8MB），递归函数和大对象仍需使用堆。

### 规则27.4 循环体外提不变表达式

**写法**：循环体内不变的计算表达式应提至循环外，避免重复计算。

**示例**：
```c
/* 错误：循环内重复计算不变表达式 */
for (int i = 0; i < count; i++) {
    arr[i] = arr[i] * sqrt(2.0) / total_count;  /* sqrt(2.0)和total_count不变 */
}

/* 正确：将不变表达式提至循环外 */
const double factor = sqrt(2.0) / total_count;  /* 仅计算一次 */
for (int i = 0; i < count; i++) {
    arr[i] = arr[i] * factor;
}
```

**警示**：⚠️ 循环体不变表达式（Loop-Invariant Code Motion）是最常见的性能优化机会。**编译器优化**：大多数现代编译器会自动进行循环不变表达式外提优化，但前提是编译器能确定表达式的值在循环中不变（如不涉及指针别名、函数调用等）。

### 规则27.5 合理使用缓存行和局部性

**写法**：利用CPU缓存的局部性原理，优化数据访问模式。

**示例**：
```c
/* 空间局部性好：顺序访问 */
for (int i = 0; i < N; i++) {
    sum += arr[i];  /* 顺序访问，缓存友好 */
}

/* 空间局部性差：跳跃访问 */
for (int i = 0; i < N; i++) {
    sum += arr[scatter[i]];  /* 随机访问，缓存不友好 */
}
```

**警示**：⚠️ CPU 缓存是性能的关键。**经验法则**：① 顺序访问内存比随机访问快 10-100 倍；② 结构体数组（AoS）vs 数组结构体（SoA）的选择取决于访问模式；③ 避免在热路径中遍历链表（链表节点在内存中不连续）。**缓存行大小**：通常 64 字节，可容纳 16 个 `int` 或 8 个 `double`。

### 规则27.6 避免频繁调用昂贵的函数

**写法**：将昂贵的函数调用结果缓存，避免在循环中重复调用。

**示例**：
```c
/* 错误：每次循环都调用strlen */
for (int i = 0; i < strlen(str); i++) {  /* strlen每次遍历整个字符串 */
    str[i] = toupper(str[i]);
}

/* 正确：将strlen结果缓存 */
size_t len = strlen(str);
for (size_t i = 0; i < len; i++) {
    str[i] = toupper(str[i]);
}
```

**警示**：⚠️ 这个例子中，`strlen` 是 O(n) 的，`for` 循环条件中每次迭代都调用 `strlen`，导致总复杂度从 O(n) 变为 O(n²)。**常见场景**：① 循环条件中调用 `strlen`、`sizeof`（对数组有效）；② 循环中调用 `gettimeofday`、`clock_gettime` 等系统调用；③ 循环中打开/关闭文件。

### 规则27.7 选择合适的数据结构

**写法**：根据操作模式选择合适的数据结构，避免使用错误的数据结构导致性能问题。

**示例**：
```c
/* 频繁查找 → 使用哈希表或排序数组+二分查找 */
/* 频繁插入/删除 → 使用链表 */
/* 频繁按顺序访问 → 使用数组 */
/* 频繁按优先级访问 → 使用堆 */

/* 错误：使用数组进行频繁查找 */
int find_user(const user_t *users, int count, uint32_t id)
{
    for (int i = 0; i < count; i++) {  /* O(n) */
        if (users[i].id == id) return i;
    }
    return -1;
}

/* 正确：使用哈希表 */
int find_user(const hash_table_t *users, uint32_t id)
{
    return hash_table_get(users, id);  /* O(1) */
}
```

**警示**：⚠️ 数据结构的选择对性能影响最大。**时间复杂度对照**：① 数组：随机访问 O(1)，插入/删除 O(n)；② 链表：插入/删除 O(1)，随机访问 O(n)；③ 哈希表：查找/插入/删除 平均 O(1)；④ 平衡树：查找/插入/删除 O(log n)。**选择原则**：根据最频繁的操作选择数据结构。

### 规则27.8 减少锁竞争

**写法**：使用线程本地数据、读写锁、无锁数据结构等减少锁竞争。

**示例**：
```c
/* 错误：锁竞争严重 */
void process_request(int thread_id)
{
    pthread_mutex_lock(&g_global_lock);
    /* 所有线程互斥执行，实际可以并行 */
    do_independent_work(thread_id);
    pthread_mutex_unlock(&g_global_lock);
}

/* 正确：使用线程本地数据 */
_Thread_local int t_local_data = 0;

void process_request(int thread_id)
{
    do_independent_work(thread_id);  /* 无需加锁 */
    t_local_data = thread_id;
}
```

**警示**：⚠️ 锁竞争是并发程序性能下降的主要原因。**减少锁竞争的策略**：① 使用线程本地数据（Thread-Local Storage）；② 使用读写锁（读写分离）；③ 使用无锁数据结构（如无锁队列）；④ 减少锁的粒度；⑤ 使用原子操作替代锁。

### 建议27.1 使用编译器内置函数替代手写优化

**写法**：利用编译器内置函数（built-in functions）替代手写汇编或复杂算法。

**示例**：
```c
/* 使用编译器内置函数进行位操作 */
int leading_zeros = __builtin_clz(value);  /* GCC/Clang */
int popcount = __builtin_popcount(value);  /* GCC/Clang */

/* 使用编译器内置函数进行溢出检查 */
if (__builtin_add_overflow(a, b, &result)) {
    /* 溢出处理 */
}
```

**警示**：⚠️ 编译器内置函数通常被映射为单条CPU指令，比手写代码快得多。**常见内置函数**：① `__builtin_clz`/`__builtin_ctz`：前导/尾随零计数；② `__builtin_popcount`：位计数（汉明重量）；③ `__builtin_add_overflow`：带溢出检测的加法。**注意**：这些函数是编译器特定的，跨平台时需提供替代实现。

### 建议27.2 性能优化分级

**写法**：根据性能敏感度将代码分为三级，采用不同的优化策略。

**示例**：
```
| 级别 | 说明 | 优化策略 |
|------|------|---------|
| P0 | 热路径（每秒执行>10000次） | 极致优化，可牺牲部分可读性 |
| P1 | 普通路径（每秒执行100-10000次） | 合理优化，保持可读性 |
| P2 | 冷路径（每秒执行<100次） | 不优化，保持清晰可读 |
```

**警示**：⚠️ 并非所有代码都需要优化。**帕累托法则**：约 20% 的代码（热路径）消耗了 80% 的执行时间。**优化策略**：① P0 级代码：使用内联、缓存、手动循环展开等优化手段；② P1 级代码：保持合理性能，但优先保证可读性；③ P2 级代码：不优化，清晰性和正确性优先。

### 建议27.3 合理使用内联函数

**写法**：对于频繁调用且体积极小的函数（如 getter/setter、简单封装函数），使用 `inline` 关键字或 `__attribute__((always_inline))` 减少函数调用开销。

**示例**：
```c
/* 适合内联：小函数，频繁调用 */
static inline int is_valid_id(uint32_t id)
{
    return id > 0 && id < MAX_ID;
}

/* 不适合内联：大函数，调用频率低 */
int process_complex_data(data_t *data)
{
    /* 几百行代码... */
}
```

**警示**：⚠️ 内联是把双刃剑。**优点**：消除函数调用开销（参数压栈、跳转、返回）。**缺点**：① 增大代码体积（代码膨胀），可能导致指令缓存失效；② 修改内联函数体后，所有调用点都需要重新编译。**建议**：① 只对体积极小的函数（< 10 行）使用内联；② 只在热路径上使用内联；③ 让编译器自行决定是否内联（`inline` 只是建议，编译器可忽略）。

### 建议27.4 分支预测优化：将高概率路径放在前面

**写法**：在 `if`/`else` 分支中，将预期为真的条件放在前面，利用 `__builtin_expect`（GCC/Clang）提示编译器优化分支预测。

**示例**：
```c
/* 使用__builtin_expect提示分支预测 */
#define LIKELY(x)     __builtin_expect(!!(x), 1)
#define UNLIKELY(x)   __builtin_expect(!!(x), 0)

/* 错误：未使用分支预测提示 */
if (ptr == NULL) {  /* 正常情况下ptr不应该为NULL */
    return ERR_NULL_PTR;
}
/* 正常处理路径 */

/* 正确：使用UNLIKELY提示极低概率分支 */
if (UNLIKELY(ptr == NULL)) {
    return ERR_NULL_PTR;
}
/* 正常处理路径——编译器将其放在热路径上 */
```

**警示**：⚠️ 分支预测优化仅对CPU无法预测的分支有效。**原理**：现代CPU使用分支预测器，如果预测错误，会浪费约10-20个CPU周期。`__builtin_expect` 告诉编译器哪个分支更可能被执行，编译器将该分支放在"顺序执行"路径上。**适用场景**：① 错误处理分支（极低概率）；② 边界检查分支；③ 初始化检查分支。**注意**：滥用 `__builtin_expect` 可能导致性能下降，只在确认的热路径上使用。

### 建议27.5 注意内存对齐以提高访问效率

**写法**：结构体定义时，按照成员对齐要求排序字段，减少填充字节（padding）导致的浪费。

**示例**：
```c
/* 错误：字段顺序不当，浪费空间 */
struct bad_layout {
    char  c;       /* 1字节，偏移0 */
    /* 3字节填充 */
    int   i;       /* 4字节，偏移4 */
    short s;       /* 2字节，偏移8 */
    /* 2字节填充（对齐到4字节边界） */
};  /* sizeof = 12 */

/* 正确：按对齐要求降序排列 */
struct good_layout {
    int   i;       /* 4字节，偏移0，对齐要求4 */
    short s;       /* 2字节，偏移4，对齐要求2 */
    char  c;       /* 1字节，偏移6，对齐要求1 */
    /* 1字节填充（对齐到4字节边界） */
};  /* sizeof = 8，节省了33% */
```

**警示**：⚠️ 结构体填充是C语言中常见的空间浪费。**对齐规则**：① 结构体成员的偏移量必须是对齐要求的整数倍；② 结构体总大小必须是最宽成员对齐要求的整数倍。**优化策略**：将成员按对齐要求从大到小排序（int → short → char）。**注意**：在跨平台网络协议或硬件寄存器映射中，可能需要使用 `__attribute__((packed))` 强制紧凑布局，但访问未对齐数据会导致性能惩罚。

### 建议27.6 使用局部静态变量避免重复初始化

**写法**：对于函数内频繁使用的常量数据，使用 `static const` 局部变量避免每次调用时重新初始化。

**示例**：
```c
/* 错误：每次调用都重新初始化 */
void process(void)
{
    const char *prefix = "PREFIX_";  /* 每次调用都在栈上初始化 */
    const int max_retries = 3;       /* 每次调用都分配栈空间 */
    /* ... */
}

/* 正确：使用static const，仅初始化一次 */
void process(void)
{
    static const char prefix[] = "PREFIX_";  /* 仅初始化一次，数据段存储 */
    static const int max_retries = 3;        /* 编译期常量，无运行时开销 */
    /* ... */
}
```

**警示**：⚠️ `static const` 局部变量在数据段中分配，仅初始化一次，后续调用不再分配栈空间和执行初始化。**适合场景**：① 字符串常量；② 查找表；③ 配置常量。**不适合场景**：① 需要动态计算的值；② 每调用需要不同值的变量；③ 线程安全的场景（需要额外保护）。

---

## 附录A：安全函数对照表

### A.1 字符串操作函数

| 不安全函数 | 风险类型 | 风险等级 | CWE编号 | 安全替代方案 | 说明 |
|-----------|---------|---------|---------|-------------|------|
| `strcpy(dest, src)` | 缓冲区溢出 | 高危 | CWE-121 | `strncpy(dest, src, n)` / `strcpy_s(dest, n, src)` | n为目标缓冲区大小，注意strncpy在src长度>=n时不补`\0` |
| `strcat(dest, src)` | 缓冲区溢出 | 高危 | CWE-121 | `strncat(dest, src, n)` / `strcat_s(dest, n, src)` | n为目标缓冲区剩余大小 |
| `sprintf(buf, fmt, ...)` | 缓冲区溢出 | 高危 | CWE-121 | `snprintf(buf, n, fmt, ...)` / `sprintf_s(buf, n, fmt, ...)` | n为目标缓冲区大小，始终包含`\0`空间 |
| `vsprintf(buf, fmt, ap)` | 缓冲区溢出 | 高危 | CWE-121 | `vsnprintf(buf, n, fmt, ap)` / `vsprintf_s(buf, n, fmt, ap)` | n为目标缓冲区大小 |
| `gets(buf)` | 缓冲区溢出 | 高危 | CWE-121 | `fgets(buf, n, stdin)` / `gets_s(buf, n)` | 无长度限制，已从C11标准中移除 |
| `scanf(fmt, ...)` | 缓冲区溢出 | 高危 | CWE-120 | `fscanf(stream, "%Ns", buf)` / `sscanf_s(buf, "%Ns", ...)` | 使用`%Ns`限制字符串长度，N为最大长度 |
| `sscanf(str, fmt, ...)` | 缓冲区溢出 | 高危 | CWE-120 | `sscanf_s(str, buf, "%Ns", ...)` | 对`%s`/`%[`格式必须指定最大长度 |
| `wcscpy(dest, src)` | 缓冲区溢出 | 高危 | CWE-121 | `wcsncpy(dest, src, n)` / `wcscpy_s(dest, n, src)` | 宽字符版本，与strcpy同类型问题 |
| `strncpy(dest, src, n)` | 不补`\0` | 中危 | CWE-121 | `strcpy_s(dest, n, src)` 或手动补`\0` | n>=src长度时不补`\0`，需手动`dest[n-1]='\0'` |
| `_snprintf(buf, n, ...)`（MSVC） | 不补`\0` | 中危 | CWE-121 | `snprintf(buf, n, ...)`（C99标准） | MSVC中`_snprintf`在输出>=n时不补`\0` |

### A.2 格式化输出函数

| 不安全函数 | 风险类型 | 风险等级 | CWE编号 | 安全替代方案 | 说明 |
|-----------|---------|---------|---------|-------------|------|
| `printf(user_input)` | 格式化字符串攻击 | 高危 | CWE-134 | `printf("%s", user_input)` | 格式化字符串由外部可控时，可导致内存泄露和代码执行 |
| `fprintf(stderr, user_input)` | 格式化字符串攻击 | 高危 | CWE-134 | `fprintf(stderr, "%s", user_input)` | 同上 |
| `snprintf(buf, n, user_input, ...)` | 格式化字符串攻击 | 高危 | CWE-134 | `snprintf(buf, n, "%s", user_input, ...)` | 格式化字符串必须为常量或固定字符串 |
| `syslog(priority, user_input)` | 格式化字符串攻击 | 高危 | CWE-134 | `syslog(priority, "%s", user_input)` | syslog也支持格式化字符串 |

### A.3 进程执行函数

| 不安全函数 | 风险类型 | 风险等级 | CWE编号 | 安全替代方案 | 说明 |
|-----------|---------|---------|---------|-------------|------|
| `system(cmd)` | 命令注入 | 高危 | CWE-78 | 使用 `execve(argv, envp)` 或白名单校验 | 参数包含外部数据时，攻击者可执行任意命令 |
| `popen(cmd, type)` | 命令注入 | 高危 | CWE-78 | 使用白名单校验后调用 | 同上，命令执行后输出到管道 |
| `execlp(file, arg, ...)` | 命令注入 | 高危 | CWE-78 | 使用 `execve` 指定绝对路径 | 使用PATH搜索可执行文件，可能被劫持 |
| `execvp(file, argv)` | 命令注入 | 高危 | CWE-78 | 使用 `execve` 指定绝对路径 | 同上 |
| `WinExec(cmd, show)` | 命令注入 | 高危 | CWE-78 | `CreateProcess` 指定完整路径+参数 | Windows命令行执行，路径含空格时可能被劫持 |
| `ShellExecute(hwnd, op, file, ...)` | 命令注入 | 高危 | CWE-78 | 对文件路径加双引号 | 文件路径含空格可能导致执行非预期程序 |

### A.4 模块加载函数

| 不安全函数 | 风险类型 | 风险等级 | CWE编号 | 安全替代方案 | 说明 |
|-----------|---------|---------|---------|-------------|------|
| `dlopen(filename, flags)` | 任意代码加载 | 高危 | CWE-114 | 使用白名单机制，限制加载路径 | 外部数据控制filename时，可加载任意共享库 |
| `LoadLibrary(filename)` | DLL劫持 | 高危 | CWE-114 | 使用绝对路径 + `SetDllDirectory("")` | 相对路径搜索时，当前目录可能被植入恶意DLL |
| `LoadLibraryEx(filename, ...)` | DLL劫持 | 高危 | CWE-114 | 指定 `LOAD_LIBRARY_SEARCH_DLL_LOAD_DIR` 标志 | 使用安全搜索路径标志 |

### A.5 内存操作函数

| 不安全函数 | 风险类型 | 风险等级 | CWE编号 | 安全替代方案 | 说明 |
|-----------|---------|---------|---------|-------------|------|
| `alloca(size)` | 栈溢出 | 高危 | CWE-789 | 使用固定大小数组或 `malloc` | 栈上分配大小不可控，易导致栈溢出 |
| `realloc(ptr, size)` | 内存泄漏 | 中危 | CWE-401 | 使用 `malloc` + `memcpy` + `free` | 失败时返回NULL但原指针未释放 |
| `malloc(0)` | 未定义行为 | 中危 | CWE-687 | 确保size>0后再分配 | 大小为0时行为是实现定义的，可能返回NULL |
| `memset(ptr, 0, n)`（清零敏感信息） | 编译器优化 | 中危 | CWE-14 | 使用 `memset_s(ptr, n, 0, n)` 或 `SecureZeroMemory` | 编译器可能优化掉对"不再使用"内存的memset调用 |

### A.6 随机数生成函数

| 不安全函数 | 风险类型 | 风险等级 | CWE编号 | 安全替代方案 | 说明 |
|-----------|---------|---------|---------|-------------|------|
| `rand()` | 可预测随机数 | 高危 | CWE-338 | `arc4random()` / `getrandom()` / `RAND_bytes()` | LCG线性同余算法，输出完全可预测 |
| `random()` | 可预测随机数 | 中危 | CWE-338 | `getrandom()` / `RAND_bytes()` | 比rand稍好但依然不安全 |
| `srand(seed)` + `rand()` | 种子可预测 | 高危 | CWE-335 | 使用密码学安全随机数生成器 | `srand(time(0))` 种子在秒级精度内可预测 |

### A.7 不可重入函数

| 不安全函数 | 风险类型 | 风险等级 | CWE编号 | 安全替代方案 | 说明 |
|-----------|---------|---------|---------|-------------|------|
| `strtok(str, delim)` | 不可重入 | 中危 | CWE-676 | `strtok_r(str, delim, saveptr)` | 使用静态缓冲区，多线程不安全 |
| `asctime(tm)` / `ctime(t)` | 缓冲区溢出 | 中危 | CWE-121 | `asctime_r(tm, buf)` / `ctime_r(t, buf)` | 使用静态缓冲区，线程不安全 |
| `gmtime(t)` / `localtime(t)` | 不可重入 | 中危 | CWE-676 | `gmtime_r(t, result)` / `localtime_r(t, result)` | 返回静态结构体指针，线程不安全 |
| `rand()` / `srand()` | 不可重入 | 中危 | CWE-676 | 使用线程安全的随机数生成器 | 使用全局状态，多线程中行为未定义 |

### A.8 控制流函数

| 不安全函数 | 风险类型 | 风险等级 | CWE编号 | 安全替代方案 | 说明 |
|-----------|---------|---------|---------|-------------|------|
| `setjmp(env)` / `longjmp(env, val)` | 资源泄漏 | 高危 | CWE-562 | 使用正常返回值和错误码 | 跳过栈展开，导致资源泄漏和未定义行为 |
| `atexit(func)` | 注册不安全 | 中危 | CWE-364 | 在main函数返回前手动清理 | 注册的清理函数在exit时调用，顺序不可控 |
| `abort()` | 跳过清理 | 中危 | CWE-754 | 使用return或exit | 不执行任何清理，可能导致资源泄漏 |
| `pthread_exit(retval)` | 资源泄漏 | 中危 | CWE-676 | 从线程函数return | 跳过C++析构函数和栈展开 |

### A.9 文件I/O函数

| 不安全函数 | 风险类型 | 风险等级 | CWE编号 | 安全替代方案 | 说明 |
|-----------|---------|---------|---------|-------------|------|
| `fopen(path, mode)` | 路径遍历 | 中危 | CWE-22 | 白名单校验路径后再使用 | 路径包含`../`可能导致任意文件读写 |
| `creat(path, mode)` | 权限过宽 | 中危 | CWE-276 | 使用 `open(path, flags, S_IRUSR\|S_IWUSR)` | 默认权限可能过宽，必须显式指定 |
| `tmpfile()` / `mktemp()` | 竞争条件 | 中危 | CWE-377 | 使用 `mkstemp()` 或系统安全临时文件API | TOCTOU竞争条件，可能被符号链接攻击 |

---

## 附录B：代码审查清单

### B.1 通用检查项

| 编号 | 检查项 | 严重程度 | 检查方法 |
|------|-------|---------|---------|
| 1 | 所有外部输入是否经过校验？ | 严重 | 代码审查 |
| 2 | 缓冲区写入是否做了边界检查？ | 严重 | 代码审查 |
| 3 | 是否所有malloc都有对应的free？ | 严重 | 静态分析 |
| 4 | 是否所有fopen都有对应的fclose？ | 严重 | 静态分析 |
| 5 | 指针使用前是否判空？ | 严重 | 代码审查 |
| 6 | 函数返回值是否被检查？ | 重要 | 代码审查 |
| 7 | 是否存在未初始化的变量？ | 重要 | 编译警告 |
| 8 | 数组访问是否越界？ | 严重 | 静态分析 |
| 9 | 除零操作是否被避免？ | 严重 | 代码审查 |
| 10 | 整数溢出是否被检查？ | 重要 | 代码审查 |

### B.2 并发检查项

| 编号 | 检查项 | 严重程度 | 检查方法 |
|------|-------|---------|---------|
| 1 | 共享资源是否加锁保护？ | 严重 | 代码审查 |
| 2 | 锁的获取顺序是否一致？ | 严重 | 代码审查 |
| 3 | 锁的作用域是否最小化？ | 建议 | 代码审查 |
| 4 | 是否存在死锁风险？ | 严重 | 代码审查 |
| 5 | 条件变量是否配合while循环使用？ | 重要 | 代码审查 |
| 6 | 信号处理函数中是否调用了非异步安全函数？ | 严重 | 代码审查 |

### B.3 安全检查项

| 编号 | 检查项 | 严重程度 | CWE编号 |
|------|-------|---------|---------|
| 1 | 是否存在缓冲区溢出？ | 严重 | CWE-121 |
| 2 | 是否存在格式化字符串漏洞？ | 严重 | CWE-134 |
| 3 | 是否存在整数溢出？ | 重要 | CWE-190 |
| 4 | 是否存在命令注入？ | 严重 | CWE-78 |
| 5 | 是否存在路径遍历？ | 严重 | CWE-22 |
| 6 | 是否存在TOCTOU竞争条件？ | 重要 | CWE-367 |
| 7 | 是否存在内存泄漏？ | 重要 | CWE-401 |
| 8 | 是否存在使用已释放内存？ | 严重 | CWE-416 |
| 9 | 是否存在数组越界？ | 严重 | CWE-787 |
| 10 | 是否存在未定义行为？ | 严重 | CWE-758 |

### B.4 性能检查项

| 编号 | 检查项 | 严重程度 |
|------|-------|---------|
| 1 | 是否在热路径中进行了不必要的内存分配？ | 重要 |
| 2 | 循环内是否有不变表达式？ | 建议 |
| 3 | 是否进行了不必要的拷贝？ | 建议 |
| 4 | 锁的粒度是否合理？ | 重要 |
| 5 | 是否使用了合适的数据结构？ | 建议 |
| 6 | 是否存在栈上分配过大内存？ | 重要 |

---

## 附录C：参考文档与标准

### C.1 国际标准

| 标准 | 全称 | 说明 |
|------|------|------|
| **ISO/IEC 9899:2011** | Programming Languages - C | C11标准 |
| **ISO/IEC 9899:2018** | Programming Languages - C | C17标准 |
| **MISRA C:2012** | Motor Industry Software Reliability Association | 汽车行业C语言安全编码标准 |
| **MISRA C:2023** | Motor Industry Software Reliability Association | MISRA C最新版 |
| **CERT C Coding Standard** | SEI CERT C Coding Standard | 卡内基梅隆大学安全编码标准 |
| **CWE** | Common Weakness Enumeration | 通用弱点枚举 |
| **CVE** | Common Vulnerabilities and Exposures | 通用漏洞与暴露 |

### C.2 推荐工具

| 类别 | 工具 | 说明 |
|------|------|------|
| 编译器 | GCC / Clang / MSVC | 主流C语言编译器 |
| 静态分析 | Cppcheck / Clang Static Analyzer / Coverity | 代码静态分析 |
| 动态分析 | Valgrind / AddressSanitizer / UndefinedBehaviorSanitizer | 运行时检测 |
| 代码风格 | clang-format | 自动格式化 |
| 测试框架 | Unity / CMock / Ceedling / Check | 单元测试 |
| 构建系统 | CMake / Make / Meson | 项目构建 |
| 性能分析 | perf / gprof / Callgrind / VTune | 性能热点分析 |
| 文档生成 | Doxygen | 代码文档自动生成 |

### C.3 内部参考文档

| 文档 | 说明 |
|------|------|
| 《C语言结构体和联合体高级技巧》 | 结构体/联合体进阶用法 |
| 《C&C++语言安全编程规范》 | 安全编程补充规范 |
| 《代码审查操作指南》 | 审查流程详细说明 |
| 《构建系统配置手册》 | 构建系统详细配置 |

---

## 附录D：规范速查卡

### D.1 强制规则一览

| 章节 | 规则编号 | 简要描述 | 严重程度 |
|------|---------|---------|---------|
| 2 | 规则2.1-2.7 | 头文件规范 | 高 |
| 3 | 规则3.1-3.8 | 函数规范 | 高 |
| 4 | 规则4.1-4.14 | 命名规范 | 中 |
| 5 | 规则5.1-5.11 | 排版格式 | 中 |
| 6 | 规则6.1-6.5 | 注释规范 | 中 |
| 7 | 规则7.1-7.8 | 变量规范 | 高 |
| 8 | 规则8.1-8.5 | 宏规范 | 高 |
| 9 | 规则9.1-9.7 | 预处理规范 | 中 |
| 10 | 规则10.1-10.8 | 结构体/联合体 | 中 |
| 11 | 规则11.1-11.8 | 表达式/语句 | 高 |
| 12 | 规则12.1-12.9 | 内存管理 | 高 |
| 13 | 规则13.1-13.8 | 字符串安全 | 高 |
| 14 | 规则14.1-14.6 | 错误处理 | 高 |
| 15 | 规则15.1-15.6 | 整数安全 | 高 |
| 16 | 规则16.1-16.5 | 文件I/O安全 | 高 |
| 17 | 规则17.1-17.9 | 安全编程 | 高 |
| 18 | 规则18.1-18.7 | 并发编程 | 高 |
| 19 | 规则19.1-19.4 | Volatile/内存屏障 | 高 |
| 20 | 规则20.1-20.4 | 中断处理 | 高 |
| 21 | 规则21.1-21.5 | 日志/诊断 | 中 |
| 22 | 规则22.1-22.6 | API设计/版本管理 | 中 |
| 23 | 规则23.1-23.7 | 测试规范 | 中 |
| 24 | 规则24.1-24.8 | 构建/编译 | 中 |
| 25 | 规则25.1-25.6 | 代码审查 | 中 |
| 26 | 规则26.1-26.6 | 跨平台开发 | 中 |
| 27 | 规则27.1-27.8 | 性能优化 | 中 |

---

## 附录E：常见错误速查表

### E.1 编译期错误

| 错误信息 | 常见原因 | 示例 | 修复方法 |
|---------|---------|------|---------|
| `implicit declaration of function` | 未包含函数声明头文件 | `printf("hello");` 但未 `#include <stdio.h>` | 添加对应头文件，或前向声明函数 |
| `undefined reference to 'xxx'` | 链接阶段：函数未定义或未链接对应库 | 调用 `pthread_create` 但未链接 `-lpthread` | 检查函数实现，链接对应库 |
| `conflicting types for 'xxx'` | 函数声明和定义的类型不一致 | 声明 `int foo(void);` 但定义 `void foo(void) {}` | 统一声明和定义的类型 |
| `redefinition of 'xxx'` | 同一符号被重复定义 | 同一结构体在两个头文件中定义了两次 | 使用 `#ifndef` 头文件保护宏 |
| `expected ';' before '...'` | 缺少分号 | `int a = 5` 缺少分号 | 在语句末尾添加 `;` |
| `dereferencing pointer to incomplete type` | 使用不完整类型（前置声明但未定义） | 引用 `struct A` 的成员但 `struct A` 仅前置声明 | 包含定义该类型的头文件 |
| `assignment from incompatible pointer type` | 指针类型不匹配 | `int *p = &float_var;` | 确保指针类型与目标类型匹配 |
| `Multi-line comment` | 字符串或注释中未闭合的引号 | `printf("hello);` 缺少双引号 | 补全字符串引号 |

### E.2 运行时常见错误

| 错误类型 | 典型表现 | 常见原因 | 排查方法 |
|---------|---------|---------|---------|
| 段错误（Segmentation Fault） | 程序崩溃，信号SIGSEGV | ① 空指针解引用；② 访问已释放内存；③ 栈溢出；④ 数组越界 | 使用 GDB 查看 backtrace，使用 AddressSanitizer |
| 缓冲区溢出（Buffer Overflow） | 内存损坏、数据被覆盖 | ① `strcpy`/`sprintf` 未做长度检查；② 数组索引越界；③ 循环计算错误 | 使用 AddressSanitizer，开启栈保护 `-fstack-protector` |
| 内存泄漏（Memory Leak） | 程序内存持续增长 | ① `malloc` 后未 `free`；② 丢失指针引用；③ 异常路径中未释放 | 使用 Valgrind、LeakSanitizer，代码审查 |
| 使用已释放内存（Use-After-Free） | 段错误或数据损坏 | ① `free` 后继续使用指针；② 多个指针指向同一内存，部分被释放 | 使用 AddressSanitizer，释放后置 NULL |
| 双重释放（Double Free） | 堆损坏，崩溃 | ① `free` 同一指针两次；② 多个指针指向同一内存，分别释放 | 使用 AddressSanitizer，释放后置 NULL |
| 整数溢出（Integer Overflow） | 逻辑错误、缓冲区溢出 | ① 无符号整数回绕；② 有符号整数溢出（未定义行为）；③ 乘法溢出 | 使用 UndefinedBehaviorSanitizer，开启 `-Wstrict-overflow` |
| 死锁（Deadlock） | 程序卡死，无响应 | ① 锁顺序不一致；② 未释放锁；③ 递归加锁 | 使用 Helgrind、ThreadSanitizer，检查锁顺序 |
| 数据竞争（Data Race） | 非确定性行为，偶发崩溃 | ① 多线程无锁访问共享变量；② 未使用原子操作 | 使用 ThreadSanitizer，加锁保护共享变量 |
| 栈溢出（Stack Overflow） | 段错误，栈回溯被截断 | ① 无限递归；② 栈上分配过大数组（如 `char buf[1024*1024]`）；③ 深层递归 | 增大栈空间（`ulimit -s`），改用堆分配 |
| 格式化字符串漏洞 | 程序崩溃、信息泄露 | ① `printf(user_input)` 而非 `printf("%s", user_input)` | 使用固定格式化字符串，开启 `-Wformat-security` |

### E.3 逻辑常见错误

| 错误类型 | 典型表现 | 示例 | 修复方法 |
|---------|---------|------|---------|
| Off-By-One | 数组越界、字符串不完整 | `for (i = 0; i <= n; i++)` 多循环一次 | 使用 `i < n` 而非 `i <= n` |
| 赋值与比较混淆 | 条件始终为真 | `if (x = 5)` 而非 `if (x == 5)` | 使用 `if (5 == x)` 风格，开启 `-Wparentheses` |
| 有符号/无符号混用 | 比较结果与预期相反 | `if (signed < 0 || (size_t)signed < unsiged)` | 统一类型后再比较 |
| 短路求值误用 | 逻辑错误 | `if (p != NULL && p->value)` 顺序错误 | 确保先判空再解引用 |
| 变量作用域误用 | 变量值非预期 | 在 `if` 块中声明变量，块外使用 | 将变量声明在合适的作用域 |
| 位运算优先级错误 | 错误结果 | `if (x & MASK == 0)` 等价于 `if (x & (MASK == 0))` | 使用括号明确优先级：`if ((x & MASK) == 0)` |
| 浮点数相等比较 | 条件不成立 | `if (f1 == f2)` 由于精度问题可能永不成立 | 使用 `fabs(f1 - f2) < EPSILON` |
| 宏参数未加括号 | 运算符优先级问题 | `#define DOUBLE(x) x+x` 调用 `DOUBLE(1)*2` 得 3 而非 4 | 始终给宏参数加括号：`#define DOUBLE(x) (x)+(x)` |
| 结构体按值拷贝 | 浅拷贝问题 | 结构体包含指针时，按值拷贝导致两个指针指向同一内存 | 实现深拷贝，或使用引用计数 |
| 函数返回局部变量地址 | 使用已释放的栈内存 | `int *foo() { int x = 5; return &x; }` | 返回堆内存或静态内存，或通过参数返回 |

### E.4 安全漏洞速查

| 漏洞类型 | 风险等级 | 典型CWE | 常见触发条件 | 防御措施 |
|---------|---------|---------|-------------|---------|
| 缓冲区溢出 | 高危 | CWE-121/CWE-122 | 未检查输入长度直接拷贝到固定缓冲区 | 使用 `snprintf`/`strncpy`，限制输入长度 |
| 格式化字符串漏洞 | 高危 | CWE-134 | 外部数据作为格式化字符串参数 | 使用 `printf("%s", input)` 而非 `printf(input)` |
| 命令注入 | 高危 | CWE-78 | 外部数据拼接到命令字符串 | 使用白名单校验，避免使用 `system()` |
| SQL注入 | 高危 | CWE-89 | 外部数据拼接到SQL语句 | 使用参数化查询，转义特殊字符 |
| 路径遍历 | 高危 | CWE-22 | 外部数据拼接文件路径 | 白名单校验，使用 `realpath` 规范化 |
| 整数溢出 | 高危 | CWE-190 | 内存分配大小计算中整数溢出 | 使用 `ckd_mul`/`ckd_add`，检查溢出 |
| 内存泄漏 | 中危 | CWE-401 | 异常路径未释放资源 | 使用 `SAFE_FREE` 宏，goto cleanup 模式 |
| 使用已释放内存 | 高危 | CWE-416 | 释放后继续使用指针 | 释放后立即置 NULL |
| 双重释放 | 高危 | CWE-415 | 同一指针释放两次 | 释放后置 NULL，使用 `SAFE_FREE` |
| 数据竞争 | 高危 | CWE-362 | 多线程无锁访问共享变量 | 使用互斥锁或原子操作 |
| 死锁 | 中危 | CWE-833 | 锁顺序不一致 | 固定锁获取顺序，使用 `trylock` |
| 未初始化变量 | 中危 | CWE-457 | 局部变量声明时未初始化 | 声明时立即初始化 |
| 空指针解引用 | 中危 | CWE-476 | 未检查 malloc 返回值 | 每次 malloc 后检查返回值 |
| TOCTOU竞争条件 | 中危 | CWE-367 | 检查和操作之间存在时间差 | 使用原子操作，避免文件系统TOCTOU |

---

> **文档版本记录**：
>
> | 版本 | 日期 | 修订内容 | 审批人 |
> |------|------|---------|--------|
> | v1.0 | 2024-01-01 | 初始版本 | 技术委员会 |
> | v1.1 | 2024-06-01 | 新增安全编程章节 | 技术委员会 |
> | v2.0 | 2025-01-01 | 全面升级为企业级规范，新增预处理、结构体/联合体、Volatile、中断处理、日志、API版本管理、测试、构建、审查、跨平台、性能优化章节及附录 | 技术委员会 |
> | v2.1 | 2025-07-01 | 完善安全函数对照表，补充审查清单，新增附录B-D | 技术委员会 |
> | v2.2 | 2026-08-03 | 新增附录E常见错误速查表，增强类型安全规范（Ch15），增强安全编程攻击场景（Ch17），增强不安全函数替代表（附录A），新增可测性规范（Ch23），增强性能优化规范（Ch27），新增安全释放宏（Ch12） | 技术委员会 |