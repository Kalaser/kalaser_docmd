---
title: 高质量 C++/C 编程指南（精读笔记）
date: 2026-10-27
description: 《高质量 C++/C 编程指南》逐章精读笔记，涵盖文件结构、命名、函数、内存管理与类设计。
tags: [C++, C, 编程规范, 读书笔记]
---

# 《高质量C++/C编程指南》详细总结
> 整理：按原书章节结构分条梳理，保留核心规则、建议、示例与附录要点。
---

## 目录

## 目录

- [一、文档总览](#高质量cc编程指南详细总结-一文档总览)
- [二、前言](#高质量cc编程指南详细总结-二前言)
- [三、第1章 文件结构](#高质量cc编程指南详细总结-三第1章-文件结构)
- [四、第2章 程序的版式](#高质量cc编程指南详细总结-四第2章-程序的版式)
- [五、第3章 命名规则](#高质量cc编程指南详细总结-五第3章-命名规则)
- [六、第4章 表达式和基本语句](#高质量cc编程指南详细总结-六第4章-表达式和基本语句)
- [七、第5章 常量](#高质量cc编程指南详细总结-七第5章-常量)
- [八、第6章 函数设计](#高质量cc编程指南详细总结-八第6章-函数设计)
- [九、第7章 内存管理](#高质量cc编程指南详细总结-九第7章-内存管理)
- [十、第8章 C++函数的高级特性](#高质量cc编程指南详细总结-十第8章-c函数的高级特性)
- [十一、第9章 类的构造函数、析构函数与赋值函数](#高质量cc编程指南详细总结-十一第9章-类的构造函数析构函数与赋值函数)
- [十二、第10章 类的继承与组合](#高质量cc编程指南详细总结-十二第10章-类的继承与组合)
- [十三、第11章 其它编程经验](#高质量cc编程指南详细总结-十三第11章-其它编程经验)
- [十四、附录A：C++/C代码审查表](#高质量cc编程指南详细总结-十四附录acc代码审查表)
- [十五、附录B：C++/C试题](#高质量cc编程指南详细总结-十五附录bcc试题)
- [十六、附录C：试题答案与评分标准](#高质量cc编程指南详细总结-十六附录c试题答案与评分标准)

---
## 一、文档总览

1. **文件名称**：《高质量C++/C编程指南》
2. **作者/版本**：林锐博士，v1.0 正式文件，完成于 2001-7-24。
3. **版本历史**：v0.9 草稿由林锐起草；v1.0 由朱洪海审查，林锐修正。
4. **性质**：上海贝尔网络应用事业部规范化文件、培训教材，公开电子文档。
5. **核心主题**：C++/C 编程风格、命名、表达式、函数、内存管理、类设计、继承组合、代码审查与试题。
6. **总体思想**：高质量程序不仅要求功能正确，还要求可读、健壮、可靠、高效、可扩展、可复用、可移植；质量来自一致规范、细节习惯和防御性编程。

---
## 二、前言

1. **软件质量现状**  
   很多程序员嘴上重视质量，实际不放在心上。初读本书会觉得“原来我的程序有这么多毛病”。

2. **编程老手与高手的误区**  
   列举 Internet 上流传的“真正程序员”形象：不写进度表、不写说明书、不写注释、不画流程图、不看手册、不写文档、自认比用户懂、不接受团队开发、第一次运行不正确、昼夜颠倒工作等。作者认为这些是缺点被误当优点。

3. **作者经历与反省**  
   作者编写过数十万行 C++/C 代码，但被微软资深工程师面试 `strcpy` 时，从编程风格、出错处理、算法复杂度三方面考查，发现自身不足。到上海贝尔后测试近百名程序员，合格率约 10%，说明高质量编程教育缺失。

4. **定义**  
   - 编程老手：能长期稳定编写高质量程序的程序员。
   - 编程高手：能长期稳定编写高难度、高质量程序的程序员。
   作者自认既不是高手也算不上老手，强调知错就改、温故知新、坚持学习。

5. **本书导读**  
   建议先做附录 B 试题并严格打分：
   - 几十分：不要声张，质量差多因不良习惯，可改。
   - 及格：基础不错，继续学习。
   - 85 分以上：有资格为团队做培训。
   - 满分：作者愿拜其为师。
   第 1-6 章讲编程风格，难度不高但细节多；第 7-11 章为专题，难度较高，尤其第 7 章内存管理。

6. **版权声明**  
   可任意拷贝、修改，但不可篡改作者及单位；未经许可不得出版或大量印发；竞争对手员工请勿公开使用；预计 2002 年 7 月建立 CMMI3 级方案。

---
## 三、第1章 文件结构

1. **程序文件划分**  
   C++/C 程序通常分为头文件（声明，`.h`）和定义文件（实现，`.c` / `.cpp` / `.cc` / `.cxx`）。

2. **版权和版本声明**  
   位于头文件和定义文件开头，内容包含：版权信息、文件名称、标识符、摘要、当前版本号、作者/修改者、完成日期、版本历史信息。

3. **头文件结构**  
   三部分：版权版本声明、预处理块、函数和类结构声明。
   - 【规则1-2-1】用 `ifndef/define/endif` 防止头文件重复引用。
   - 【规则1-2-2】引用标准库用 `#include <filename.h>`，编译器从标准库目录搜索。
   - 【规则1-2-3】引用非标准库用 `#include "filename.h"`，编译器从用户工作目录搜索。
   - 【建议1-2-1】头文件中只存放“声明”，不存放“定义”。
   - 【建议1-2-2】不提倡全局变量，尽量不在头文件中写 `extern int value;` 之类声明。

4. **定义文件结构**  
   三部分：版权版本声明、引用头文件、程序实现体（数据和代码）。

5. **头文件的作用**  
   - 通过头文件调用库功能：用户只需接口声明和二进制库，不必关心实现。
   - 加强类型安全检查：接口实现或使用与声明不一致时，编译器报错。

6. **目录结构**  
   头文件多时，头文件放 `include` 目录，定义文件放 `source` 目录；私有头文件可与定义文件同目录，以加强信息隐藏。

---
## 四、第2章 程序的版式

1. **空行**
   - 【规则2-1-1】每个类声明之后、每个函数定义结束后加空行。
   - 【规则2-1-2】函数体内逻辑密切相关的语句之间不加空行，其它地方加空行分隔。

2. **代码行**
   - 【规则2-2-1】一行代码只做一件事，如只定义一个变量或只写一条语句。
   - 【规则2-2-2】`if`、`for`、`while`、`do` 等语句自占一行，执行语句不得紧跟其后；不论执行语句多少都要加 `{}`。
   - 【建议2-2-1】尽可能在定义变量时初始化，即“就近原则”。

3. **代码行内空格**
   - 【规则2-3-1】关键字后留空格，如 `const`、`virtual`、`inline`、`case`；`if`、`for`、`while` 后留空格再跟左括号。
   - 【规则2-3-2】函数名后不要留空格，紧跟左括号，以与关键字区别。
   - 【规则2-3-3】`(` 向后紧跟，`)`、`,`、`;` 向前紧跟，紧跟处不留空格。
   - 【规则2-3-4】`,` 之后留空格；若 `,` 不是行结束符，其后留空格。
   - 【规则2-3-5】赋值、比较、算术、逻辑、位域等二元操作符前后加空格。
   - 【规则2-3-6】一元操作符如 `!`、`~`、`++`、`--`、`&`（地址）前后不加空格。
   - 【规则2-3-7】`[]`、`.`、`->` 前后不加空格。
   - 【建议2-3-1】长 `for`、`if` 表达式可适当去空格。

4. **对齐**
   - 【规则2-4-1】程序的分界符 `{` 和 `}` 应独占一行且位于同一列，同时与引用它们的语句左对齐。
   - 【规则2-4-2】`{}` 之内的代码块在 `{` 右边数格处左对齐。

5. **长行拆分**
   - 【规则2-5-1】代码行最大长度宜控制在 70-80 字符以内。
   - 【规则2-5-2】长表达式在低优先级操作符处拆分成新行，操作符放新行之首，新行适当缩进。

6. **修饰符位置**
   - 【规则2-6-1】`*` 和 `&` 应紧靠变量名，如 `char *name; int *x, y;`，避免 `int* x, y;` 中 `y` 被误解为指针。

7. **注释**
   - 【规则2-7-1】注释是提示不是文档，不可喧宾夺主，花样要少。
   - 【规则2-7-2】代码本来清楚则不必加注释。
   - 【规则2-7-3】边写代码边注释，修改代码同时修改注释，删除无用注释。
   - 【规则2-7-4】注释准确易懂，防止二义性。
   - 【规则2-7-5】避免在注释中使用缩写，特别是不常用缩写。
   - 【规则2-7-6】注释位置应与被描述代码相邻，放在上方或右方，不可放下方。
   - 【规则2-7-8】长代码特别多重嵌套时，在段落结束处加注释。

8. **类的版式**
   - 以数据为中心：`private` 数据在前，`public` 函数在后。
   - 以行为为中心：`public` 函数在前，`private` 数据在后。
   - 作者建议“以行为为中心”，先考虑类应提供什么接口，方便阅读。

---
## 五、第3章 命名规则

1. **命名规则总体态度**  
   没有一种命名规则让所有人赞同，命名对软件产品不是“成败攸关”，应制定大多数成员满意的规则并贯彻。

2. **匈牙利命名法**  
   Microsoft 的“匈牙利”法用前缀表示类型，如 `ch` 表示字符，`p` 表示指针，`ppch` 表示指向字符指针的指针。缺点是烦琐。

```c
// 匈牙利命名法：前缀表示类型
char ch;               // ch 表示字符
char* pch;             // p 表示指针，pch 表示指向字符的指针
char** ppch;           // ppch 表示指向字符指针的指针
int iCount;            // i 表示 int
float fValue;          // f 表示 float

// 缺点：烦琐，现代 C++ 不推荐
int i, j, k;           // 普通 int 变量
float fX, fY, fZ;      // 前缀 f 表示 float
```



1. **共性规则**
   - 【规则3-1-1】标识符直观且可拼读，可望文知意，不必解码；用英文单词或组合，切忌汉语拼音；用词准确。
   - 【规则3-1-2】长度符合“min-length && max-information”原则；长名字能表达含义，但非越长越好；单字符如 `i,j,k,m,n,x,y,z` 可作局部变量。
   - 【规则3-1-3】命名规则尽量与操作系统或开发工具风格一致：Windows 用大小写混排如 `AddChild`；Unix 用小写加下划线如 `add_child`；别混用。
   - 【规则3-1-4】不要出现仅靠大小写区分的相似标识符。
   - 【规则3-1-5】不要出现完全相同的局部变量和全局变量。
   - 【规则3-1-6】变量名用“名词”或“形容词+名词”。
   - 【规则3-1-7】全局函数名用“动词”或“动词+名词”，类成员函数只用“动词”。
   - 【规则3-1-8】用正确反义词组命名互斥意义变量或相反动作函数。
   - 【建议3-1-1】尽量避免名字中出现数字编号，除非逻辑确需编号。

2. **简单 Windows 应用程序命名规则**
   - 【规则3-2-1】类名和函数名用大写字母开头的单词组合。
   - 【规则3-2-2】变量和参数用小写字母开头的单词组合。
   - 【规则3-2-3】常量全用大写字母，下划线分割单词。
   - 【规则3-2-4】静态变量加前缀 `s_`。
   - 【规则3-2-5】不得已需要全局变量时加前缀 `g_`。
   - 【规则3-2-6】类数据成员加前缀 `m_`，避免与成员函数参数同名。

   ### 正确命名

   ```c++
   // 文件名：Student.h
   #ifndef STUDENT_H
   #define STUDENT_H
   
   #include <string>
   
   // 【规则3-2-3】常量全用大写字母，下划线分割单词
   const int MAX_STUDENT_COUNT = 100;
   
   // 【规则3-2-5】全局变量加 g_ 前缀（不得已时）
   extern int g_totalStudentCount;
   
   // 【规则3-2-4】静态变量加 s_ 前缀
   static int s_instanceCount = 0;
   
   // 【规则3-2-1】类名用大写字母开头的单词组合
   class Student
   {
   public:
       Student();
       ~Student();
   
       // 【规则3-1-7】类成员函数只使用“动词”
       void SetName(const std::string& name);
       std::string GetName() const;
   
       void SetAge(int age);
       int GetAge() const;
   
       // 【规则3-1-8】用反义词组命名相反动作函数
       void IncreaseScore(int delta);
       void DecreaseScore(int delta);
   
       static int GetInstanceCount();
   
   private:
       // 【规则3-2-6】类数据成员加 m_ 前缀，避免与参数同名
       std::string m_name;
       int m_age;
       int m_score;
   
       // 静态数据成员
       static int s_instanceCount;
   };
   
   #endif
   ```

   ```c++
   // 文件名：Student.cpp
   #include "Student.h"
   #include <iostream>
   
   // 定义全局变量
   int g_totalStudentCount = 0;
   
   // 定义静态成员变量
   int Student::s_instanceCount = 0;
   
   Student::Student() : m_name(""), m_age(0), m_score(0)
   {
       ++s_instanceCount;
       ++g_totalStudentCount;
   }
   
   Student::~Student()
   {
       --s_instanceCount;
       --g_totalStudentCount;
   }
   
   void Student::SetName(const std::string& name)
   {
       m_name = name;
   }
   
   std::string Student::GetName() const
   {
       return m_name;
   }
   
   void Student::SetAge(int age)
   {
       m_age = age;
   }
   
   int Student::GetAge() const
   {
       return m_age;
   }
   
   void Student::IncreaseScore(int delta)
   {
       m_score += delta;
   }
   
   void Student::DecreaseScore(int delta)
   {
       m_score -= delta;
   }
   
   int Student::GetInstanceCount()
   {
       return s_instanceCount;
   }
   
   // 【规则3-1-7】全局函数名用“动词”或“动词+名词”
   void PrintStudent(const Student& student)
   {
       std::cout << "Name: " << student.GetName()
                 << ", Age: " << student.GetAge() << std::endl;
   }
   
   int main()
   {
       // 【规则3-2-2】变量和参数用小写字母开头的单词组合
       Student student;
       student.SetName("Alice");
       student.SetAge(20);
   
       PrintStudent(student);
   
       // 【规则3-1-2】单字符局部变量可用 i, j, k
       for (int i = 0; i < 3; ++i)
       {
           Student s;
           s.SetName("Bob");
           s.SetAge(18 + i);
           PrintStudent(s);
       }
   
       return 0;
   }
   ```

   ### 错误命名

   ```c
   // 【规则3-1-1】不要用汉语拼音
   int xingming;          // 坏：拼音，不可望文知意
   std::string name;      // 好：英文名词
   
   // 【规则3-1-4】不要仅靠大小写区分
   int x;
   int X;                 // 坏：容易混淆
   int value;
   int anotherValue;      // 好：含义不同
   
   // 【规则3-1-5】不要局部变量和全局变量同名
   int g_count = 0;
   
   void Func()
   {
       int count = 0;      // 坏：与全局变量同名，易误解
       int localCount = 0; // 好：名字区分
   }
   
   // 【规则3-1-6】变量名用“名词”或“形容词+名词”
   float oldValue;        // 好
   float newValue;        // 好
   float value1, value2;  // 坏：数字编号，含义不清（违反建议3-1-1）
   
   // 【规则3-1-7】全局函数用动词，成员函数只用动词
   void DrawBox();        // 好：全局函数，动词+名词
   void Box::Draw();      // 好：成员函数，只用动词
   void boxDraw();        // 坏：不清晰
   
   // 【规则3-1-8】用正确反义词组
   int minValue;
   int maxValue;          // 好
   void SetValue(int v);
   int GetValue() const;  // 好
   
   // 【建议3-1-1】避免数字编号
   int student1, student2; // 坏：偷懒命名
   int firstStudent;
   int secondStudent;      // 好：有含义
   ```

3. **Unix 命名规则**  
   Unix 常用小写加下划线风格，如 `add_child`。

```c++
// 文件名：student.h
#ifndef STUDENT_H
#define STUDENT_H

#include <string>

// Unix 风格：常量常用小写加下划线，或全大写也可
const int max_student_count = 100;

// 全局变量：g_ 前缀，Unix 常用小写加下划线
int g_total_student_count = 0;

// 类名：Unix 风格常用小写加下划线
class student
{
public:
    student();
    ~student();

    // 成员函数：小写加下划线
    void set_name(const std::string& name);
    std::string get_name() const;

    void set_age(int age);
    int get_age() const;

private:
    std::string m_name;   // 成员变量仍可用 m_ 前缀，或直接 name_
    int m_age;
};

#endif
```



---
## 六、第4章 表达式和基本语句

1. **运算符优先级**
   - 【规则4-1-1】代码行中运算符多时，用括号确定操作顺序，避免默认优先级。
   - 优先级从高到低大致：`()`、`[]`、`->`、`.`；`!`、`~`、`++`、`--`、`(类型)`、`sizeof`、一元 `+ - * &`；`* / %`；`+ -`；`<< >>`；`< <= > >=`；`== !=`；`&`；`^`；`|`；`&&`；`||`；`?:`；赋值类；逗号。

   ```c
   // 好：用括号明确顺序
   word = (high << 8) | low;
   
   // 坏：依赖默认优先级，可读性差
   word = high << 8 | low;
   ```

2. **复合表达式**

   - 【规则4-2-1】不要编写太复杂复合表达式。
   - 【规则4-2-2】不要有多用途复合表达式。
   - 【规则4-2-3】不要把程序复合表达式与数学表达式混淆。

   ```c
   // 坏：复合表达式过于复杂
   i = a >= b && c < d && c + f <= g + h;
   
   // 坏：多用途复合表达式，既求 a 又求 d
   d = (a = b + c) + r;
   // 应拆分为：
   a = b + c;
   d = a + r;
   
   // 坏：数学表达式与程序表达式混淆
   if (a < b < c)          // 不表示 if ((a < b) && (b < c))
                           // 实际是 if ((a < b) < c)
   {
       // ...
   }
   
   // 好：写成明确逻辑
   if ((a < b) && (b < c))
   {
       // ...
   }
   ```

3. **if 语句与零值比较**
   - 4.3.1 布尔变量：
     【规则4-3-1】不可将布尔变量直接与 `TRUE`、`FALSE` 或 `1`、`0` 比较。
     标准：`if (flag)` 表示真，`if (!flag)` 表示假。

   ```c
   // 好
   if (flag)      { /* flag 为真 */ }
   if (!flag)     { /* flag 为假 */ }
   
   // 坏：不良风格
   if (flag == TRUE)   { }
   if (flag == 1)      { }
   if (flag == FALSE)  { }
   if (flag == 0)      { }
   ```

   - 4.3.2 整型变量：
     【规则4-3-2】用 `==` 或 `!=` 直接与 0 比较。

   ```c
   // 好
   if (value == 0) { }
   if (value != 0) { }
   
   // 坏：会让人误解 value 是布尔变量
   if (value)  { }
   if (!value) { }
   ```

   - 4.3.3 浮点变量：
     【规则4-3-3】不可用 `==` 或 `!=` 与任何数字比较。
     应转化为区间比较：
     `const float EPSINON = 0.00001;`
     `if ((x >= -EPSINON) && (x <= EPSINON))`

   ```c
   // 坏：隐含错误的比较
   if (x == 0.0) { }
   
   // 好：用允许误差 EPSINON
   const float EPSINON = 0.00001;
   if ((x >= -EPSINON) && (x <= EPSINON))
   {
       // x 被认为等于 0
   }
   ```

   - 4.3.4 指针变量：
     【规则4-3-4】用 `==` 或 `!=` 与 `NULL` 比较。

   ```c
   // 好
   if (p == NULL) { }
   if (p != NULL) { }
   
   // 坏：容易让人误解 p 是整型或布尔变量
   if (p == 0) { }
   if (p != 0) { }
   if (p)      { }
   if (!p)     { }
   ```

   - 4.3.5 补充：
     可用 `if (NULL == p)` 防止误写 `if (p = NULL)`。
     `if/else/return` 应写成 `if (condition) { return x; } else { return y; }` 或 `return (condition ? x : y);`。

   ```c
   // 坏：不良风格
   if (condition)
       return x;
   return y;
   
   // 好：改写为
   if (condition)
   {
       return x;
   }
   else
   {
       return y;
   }
   
   // 更好：简练写法
   return (condition ? x : y);
   ```

4. **循环语句效率**
   - 【建议4-4-1】多重循环中，最长循环放最内层，最短循环放最外层，减少 CPU 跨切循环层次数。

   ```c
   // 低效率：长循环在最外层
   for (row = 0; row < 100; row++)
   {
       for (col = 0; col < 5; col++)
       {
           sum = sum + a[row][col];
       }
   }
   
   // 高效率：长循环在最内层
   for (col = 0; col < 5; col++)
   {
       for (row = 0; row < 100; row++)
       {
           sum = sum + a[row][col];
       }
   }
   ```

   - 【建议4-4-2】循环体内有逻辑判断且循环次数很大时，宜将逻辑判断移到循环体外。

   ```c
   // 第一种：循环体内判断，多执行 N-1 次逻辑判断
   for (i = 0; i < N; i++)
   {
       if (condition)
           DoSomething();
       else
           DoOtherthing();
   }
   
   // 第二种：N 很大时效率更高
   if (condition)
   {
       for (i = 0; i < N; i++)
           DoSomething();
   }
   else
   {
       for (i = 0; i < N; i++)
           DoOtherthing();
   }
   ```

   

5. **for 语句循环控制变量**  
   建议循环变量在 `for` 内控制，避免在循环体内修改。

```c
// 好：循环变量在 for 内控制
for (int i = 0; i < N; i++)
{
    // 不在循环体内修改 i
}

// 坏：在循环体内修改循环变量，容易出错
for (int i = 0; i < N; i++)
{
    if (someCondition)
        i++;   // 可能导致跳过元素或越界
}
```

1. **switch 语句**
   - 【规则4-6-1】每个 `case` 结尾不要忘加 `break`，否则多个分支重叠，除非有意。
   - 【规则4-6-2】不要忘记最后的 `default` 分支，即使不需要处理，也应保留 `default: break;`。

   ```c
   switch (value)
   {
       case 1:
           DoSomething1();
           break;          // 不要漏掉 break
   
       case 2:
           DoSomething2();
           break;
   
       default:
           break;          // 不要漏掉 default
   }
   ```

2. **goto 语句**
   - `goto` 可灵活跳转，不加限制会破坏结构化设计，可能跳过对象构造、变量初始化、重要计算等，留下隐患。
   - 但 `goto` 可从多重循环一下子跳到外面，避免写很多 `break`。
   - 主张少用、慎用 `goto`，而非禁用。

   ```c
   // 坏：goto 跳过变量初始化
   goto state;
   String s1, s2;   // 被 goto 跳过
   int sum = 0;     // 被 goto 跳过
   // ...
   state:
   // ...
   
   // 合理使用：从多重循环中跳出
   for (...)
   {
       for (...)
       {
           if (error)
               goto error;
       }
   }
   error:
   // 统一错误处理
   ```

   

---
## 七、第5章 常量

1. **为什么需要常量**  
   直接写数字或字符串会导致可读性差、书写错误、修改麻烦。
   【规则5-1-1】尽量使用含义直观的常量表示多次出现的数字或字符串。

2. **const 与 #define 比较**
   - `const` 常量有数据类型，编译器可做类型安全检查；`#define` 宏常量只做字符替换，无类型安全检查，可能产生边际效应。
   - 集成调试工具可调试 `const` 常量，不能调试宏常量。
   - 【规则5-2-1】在 C++ 程序中只使用 `const` 常量而不使用宏常量，即 `const` 完全取代宏常量。

3. **常量定义规则**
   - 【规则5-3-1】对外公开的常量放头文件，不公开的放定义文件头部；不同模块常量可集中放公共头文件。
   - 【规则5-3-2】常量间密切相关时，应在定义中包含关系，而非孤立值。

4. **类中的常量**
   - `const` 数据成员只在某个对象生存期内是常量，对整个类而言可变，不同对象可有不同值。
   - 不能在类声明中初始化 `const` 数据成员。
   - `const` 数据成员初始化只能在构造函数初始化表中进行。
   - 若要整个类中恒定，用类中枚举常量：
     `enum { SIZE1 = 100, SIZE2 = 200 };`
   - 枚举常量不占用对象存储空间，编译时求值；缺点是隐含整数类型，最大值有限，不能表示浮点数。

---
## 八、第6章 函数设计

1. **函数接口要素**  
   参数和返回值。传递方式：值传递、指针传递；C++ 增加引用传递。引用传递性质像指针传递，使用方式像值传递。

2. **参数规则**
   - 【规则6-1-1】参数书写完整，不要只写类型省略名字；无参数用 `void`。
   - 【规则6-1-2】参数命名恰当，顺序合理。
   - 【规则6-1-3】指针参数仅作输入时，类型前加 `const` 防止意外修改。
   - 【规则6-1-4】输入参数以值传递方式传递对象时，宜改用 `const &`，省去临时对象构造析构，提高效率。
   - 【建议6-1-1】参数个数尽量控制在 5 个以内。
   - 【建议6-1-2】尽量不要使用类型和数目不确定的参数，如 `printf`，会丧失严格类型安全检查。

   ### 参数规则示例
   ### 数书写完整、命名恰当、顺序合理
   
   ```c
   // 好：参数完整，名字有意义，目的参数在前，源参数在后
   void StringCopy(char *strDestination, const char *strSource);
   
   // 坏：省略参数名
   void StringCopy(char *, const char *);
   
   // 无参数用 void
   float GetValue(void);
   
   // 坏：无参数省略 void
   float GetValue();
   ```
   
   ### 指针参数仅作输入时加 const
   
   ```c
   // strSource 是输入参数，加 const 防止函数体内意外修改
   void StringCopy(char *strDestination, const char *strSource);
   ```
   
   ### 输入参数用 const & 代替值传递
   
   ```c++
   class A
   {
   public:
       A();                // 无参数构造函数
       A(const A &other);  // 拷贝构造函数
       ~A();
       A &operator=(const A &other);
   };
   
   // 坏：值传递，会产生临时对象，调用拷贝构造和析构
   void Func1(A a);
   
   // 好：const 引用传递，不产生临时对象，且防止修改
   void Func2(const A &a);
   ```
   
   ###  参数个数控制在 5 个以内
   
   ```c
   // 坏：参数太多，容易搞错顺序和类型
   void SetInfo(int id, const char *name, int age, const char *address,
                const char *phone, const char *email, int department);
   
   // 好：封装成结构体或类
   struct PersonInfo
   {
       int id;
       const char *name;
       int age;
       const char *address;
       const char *phone;
       const char *email;
       int department;
   };
   
   void SetInfo(const PersonInfo &info);
   ```
   
   ### 不使用类型和数目不确定的参数
   
   ```c
   // 坏：printf 风格，编译时失去严格类型检查
   int printf(const char *format, ...);
   
   // 好：参数类型明确
   void PrintInt(int value);
   void PrintString(const char *str);
   ```


3. **返回值规则**
   - 【规则6-2-1】不要省略返回值类型。C++/C 函数都必须有类型，无返回值声明为 `void`。
   - 【规则6-2-2】函数名字与返回值类型在语义上不可冲突。
   - 【规则6-2-3】不要将正常值和错误标志混在一起返回。正常值用输出参数获得，错误标志用 `return` 返回。
   - 【建议6-2-1】有时函数不需要返回值，但为支持链式表达可附加返回值。
   - 【建议6-2-2】返回对象时，有些场合用“引用传递”替换“值传递”提高效率；有些场合只能用值传递。

   ### 返回值规则示例
   
   ### 不要省略返回值类型
   
   ```c
   // 坏：C 语言中默认返回 int，C++ 中不允许
   Func();
   
   // 好：无返回值写 void
   void Func();
   
   // 好：有返回值写明确类型
   int GetValue();
   ```
   
   ### 函数名字与返回值类型语义一致
   
   ```c
   // 坏：getchar 名字像返回 char，实际返回 int
   int getchar(void);
   
   // 好：正常值用输出参数，错误标志用 return
   BOOL GetChar(char *c);
   ```
   
   ### 正常值和错误标志分开返回
   
   ```c
   // 坏：正常字符和 EOF 混在一起返回
   char c = getchar();
   if (c == EOF)   // 如果 char 是有符号，EOF 可能被截断，比较失败
   {
       // ...
   }
   
   // 好：正常值通过输出参数获得，错误标志用返回值
   BOOL GetChar(char *c)
   {
       // 成功：*c = 字符，返回 TRUE
       // 失败：返回 FALSE
       return TRUE;
   }
   ```
   
   ### 支持链式表达，可附加返回值
   
   ```c
   // 返回 strDest，支持链式调用
   char *strcpy(char *strDest, const char *strSrc);
   
   // 使用
   char str[20];
   int length = strlen(strcpy(str, "Hello World"));
   ```
   
   ### 返回对象：引用 vs 值
   
   ```c++
   class String
   {
   public:
       // 赋值函数：返回引用，支持 a = b = c，且避免拷贝
       String &operator=(const String &other);
   
       // 相加函数：必须返回值，不能返回引用
       friend String operator+(const String &s1, const String &s2);
   
   private:
       char *m_data;
   };
   
   // 赋值函数返回引用
   String &String::operator=(const String &other)
   {
       if (this == &other)
           return *this;
   
       delete [] m_data;
       int length = strlen(other.m_data);
       m_data = new char[length + 1];
       strcpy(m_data, other.m_data);
   
       return *this;   // 返回 *this 的引用
   }
   
   // 相加函数返回值
   String operator+(const String &s1, const String &s2)
   {
       String temp;
       // ... 拼接 s1 和 s2 到 temp
       return temp;    // 返回值，不能返回引用
   }
   ```
   
4. **函数内部实现规则**
   - 【规则6-3-1】入口处检查参数有效性，可用 `assert` 防止非法参数。
   - 【规则6-3-2】出口处检查 `return` 正确性和效率。
     注意：
     （1）`return` 不可返回指向“栈内存”的指针或引用。
     （2）搞清楚返回的是值、指针还是引用。
     （3）返回对象时考虑效率。

   ### 函数内部实现规则示例
   
   ### 入口处用 assert 检查参数
   
   ```c
   #include <cassert>
   
   void *MemCopy(void *pvTo, const void *pvFrom, size_t size)
   {
       // 入口检查：指针不能为空
       assert(pvTo != NULL);
       assert(pvFrom != NULL);
   
       // 防止改变 pvTo、pvFrom 的地址
       unsigned char *pbTo = (unsigned char *)pvTo;
       const unsigned char *pbFrom = (const unsigned char *)pvFrom;
   
       while (size-- > 0)
       {
           *pbTo++ = *pbFrom++;
       }
   
       return pvTo;
   }
   ```
   
   ### 出口处检查 return：不要返回栈内存
   
   ```c
   // 错误：返回指向栈内存的指针
   char *GetString(void)
   {
       char p[] = "hello world";   // p 在栈上
       return p;                   // 函数结束后 p 被销毁，返回野指针
   }
   
   // 正确：返回静态存储区字符串
   const char *GetString1(void)
   {
       return "hello world";       // 字符串字面量在静态存储区
   }
   
   // 正确：动态分配，调用者负责 free
   char *GetString2(void)
   {
       char *p = (char *)malloc(12);
       if (p != NULL)
       {
           strcpy(p, "hello world");
       }
       return p;
   }
   ```
   
   ### 返回对象时考虑效率
   
   ```c
   // 好：创建临时对象并返回，编译器可能直接在外部存储单元构造，省去拷贝
   String Func1()
   {
       return String("hello");
   }
   
   // 较差：先创建局部对象 temp，再拷贝返回，最后销毁 temp
   String Func2()
   {
       String temp("hello");
       return temp;
   }
   ```
   
5. **其它建议**
   - 【建议6-4-1】函数功能单一，不要设计多用途函数。
   - 【建议6-4-2】函数体规模小，尽量控制在 50 行代码内。
   - 【建议6-4-3】尽量避免函数带“记忆”功能，相同输入应产生相同输出；少用 `static` 局部变量。
   - 【建议6-4-4】不仅检查输入参数，还检查全局变量、文件句柄等进入函数体的变量有效性。
   - 【建议6-4-5】出错处理返回值要清楚，让使用者不易忽视或误解。

   ### 函数功能单一
   
   ```c++
   // 坏：一个函数既读文件又解析又打印
   void ProcessFile(const char *filename);
   
   // 好：拆分成多个单一功能函数
   void ReadFile(const char *filename, std::string &content);
   void ParseContent(const std::string &content, std::vector<Item> &items);
   void PrintItems(const std::vector<Item> &items);
   ```
   
   ### 函数体控制在 50 行以内
   
   ````c
   // 坏：函数体过长，难以阅读和维护
   void BigFunction()
   {
       // 几百行代码
   }
   
   // 好：拆分成小函数
   void Step1();
   void Step2();
   void Step3();
   
   void SmallFunction()
   {
       Step1();
       Step2();
       Step3();
   }
   ````
   
   ### 避免函数带“记忆”功能
   
   ```c
   // 坏：相同输入可能因 static 变量产生不同输出
   int Count()
   {
       static int n = 0;
       return ++n;
   }
   
   // 好：相同输入产生相同输出
   int Add(int a, int b)
   {
       return a + b;
   }
   ```
   
   ###  引用与指针比较示例
   
   ```c++
   #include <iostream>
   using namespace std;
   
   // 值传递：不能改变外部变量
   void Func1(int x)
   {
       x = x + 10;
   }
   
   // 指针传递：可以改变外部变量
   void Func2(int *x)
   {
       *x = *x + 10;
   }
   
   // 引用传递：书写像值传递，性质像指针
   void Func3(int &x)
   {
       x = x + 10;
   }
   
   int main()
   {
       int n = 0;
   
       Func1(n);
       cout << "n = " << n << endl;   // n = 0
   
       Func2(&n);
       cout << "n = " << n << endl;   // n = 10
   
       n = 0;
       Func3(n);
       cout << "n = " << n << endl;   // n = 10
   
       return 0;
   }
   ```
   
   
   
6. **使用断言**
   - `assert` 是仅在 Debug 版本起作用的宏，用于检查“不应该”发生的情况。参数为假则程序中止。
   - `assert` 不应有副作用，不是函数而是宏，避免 Debug 和 Release 版本差异。
   - 应加清晰注释说明断言检查什么。

7. **引用与指针比较**
   - 值传递、指针传递、引用传递示例对比。
   - 引用性质像指针，书写像值传递。指针功能强大但危险；若只需借用对象别名，用引用，不用指针，以免意外。



---
## 九、第7章 内存管理

1. **内存分配方式**
   - 静态存储区域分配：编译时分配，整个运行期存在，如全局变量、`static` 变量。
   - 栈上创建：函数内局部变量，函数结束自动释放，效率高，容量有限。
   - 堆上分配/动态内存：运行期用 `malloc` 或 `new` 申请，程序员负责 `free` 或 `delete`，灵活但问题最多。

   #### 三种内存分配方式

   ````c
   #include <cstdlib>
   
   int g_var;                  // 静态存储区：全局变量
   static int s_var;           // 静态存储区：静态变量
   
   void Func()
   {
       int local = 0;          // 栈上创建：局部变量
       int *p = (int *)malloc(sizeof(int) * 10);  // 堆上分配：动态内存
   
       if (p != NULL)
       {
           // 使用 p
           free(p);
           p = NULL;
       }
   }
   ````

2. **常见内存错误及对策**
   - 内存分配未成功却使用：检查指针是否为 `NULL`；函数参数入口用 `assert(p != NULL)`；`malloc`/`new` 后用 `if (p == NULL)` 防错。

   ```c
   // 坏：没有检查 malloc 是否成功
   char *p = (char *)malloc(100);
   strcpy(p, "hello");   // 如果 malloc 失败，p 为 NULL，程序崩溃
   
   // 好：立即检查
   char *p = (char *)malloc(100);
   if (p == NULL)
   {
       // 错误处理
       return;
   }
   strcpy(p, "hello");
   free(p);
   p = NULL;
   ```

   - 分配成功但未初始化就引用：内存缺省初值无统一标准，数组和动态内存都要赋初值。

   ```c
   // 坏：没有检查 malloc 是否成功
   char *p = (char *)malloc(100);
   strcpy(p, "hello");   // 如果 malloc 失败，p 为 NULL，程序崩溃
   
   // 好：立即检查
   char *p = (char *)malloc(100);
   if (p == NULL)
   {
       // 错误处理
       return;
   }
   strcpy(p, "hello");
   free(p);
   p = NULL;
   ```

   ```c
   // 坏：未初始化就使用
   int *arr = (int *)malloc(sizeof(int) * 10);
   for (int i = 0; i < 10; i++)
   {
       cout << arr[i] << endl;   // 内容是垃圾
   }
   
   // 好：赋初值
   int *arr = (int *)malloc(sizeof(int) * 10);
   if (arr != NULL)
   {
       for (int i = 0; i < 10; i++)
       {
           arr[i] = 0;
       }
   }
   ```

   - 操作越过内存边界：数组下标“多1”或“少1”，循环次数易错。

   ```c
   // 坏：越界写入
   int a[10];
   for (int i = 0; i <= 10; i++)   // i == 10 时越界
   {
       a[i] = 0;
   }
   
   // 好：正确边界
   for (int i = 0; i < 10; i++)
   {
       a[i] = 0;
   }
   ```

   - 忘记释放内存，造成内存泄漏：`malloc/free`、`new/delete` 必须配对。

   ```c
   // 坏：每次调用泄漏 100 字节
   void Func()
   {
       char *p = (char *)malloc(100);
       // 没有 free(p)
   }
   
   // 好：申请与释放配对
   void Func()
   {
       char *p = (char *)malloc(100);
       if (p != NULL)
       {
           // 使用 p
           free(p);
           p = NULL;
       }
   }
   ```

   - 释放后继续使用：对象关系复杂、`return` 返回栈内存、`free/delete` 后未置 `NULL` 产生野指针。

   ```c
   // 坏：free 后继续使用，p 成为野指针
   char *p = (char *)malloc(100);
   strcpy(p, "hello");
   free(p);
   strcpy(p, "world");   // 错误：p 指向已释放内存
   
   // 好：free 后立即置 NULL
   free(p);
   p = NULL;
   ```

   - 【规则7-2-1】`malloc` 或 `new` 后立即检查指针是否为 `NULL`。
   - 【规则7-2-2】不要忘记为数组和动态内存赋初值。
   - 【规则7-2-3】避免数组或指针下标越界，当心“多1”“少1”。
   - 【规则7-2-4】动态内存申请与释放必须配对，防止内存泄漏。
   - 【规则7-2-5】`free` 或 `delete` 后立即将指针置 `NULL`，防止野指针。

3. **指针与数组对比**

   - 数组在静态存储区或栈上创建，数组名对应一块内存，地址与容量在生命期内不变，内容可变。
   - 指针可随时指向任意类型内存块，特征是“可变”，常用于动态内存，灵活但危险。
   - 修改内容、内容复制与比较、计算内存容量等差异。

   ```c
   #include <iostream>
   #include <cstring>
   using namespace std;
   
   int main()
   {
       // 数组：内容可修改
       char a[] = "hello";
       a[0] = 'X';
       cout << a << endl;        // Xello
   
       // 指针指向常量字符串：内容不可修改
       const char *p = "world";
       // p[0] = 'X';            // 错误：修改常量字符串，运行错误
   
       // 数组复制必须用 strcpy
       char b[10];
       strcpy(b, a);             // 不能用 b = a;
       if (strcmp(b, a) == 0)    // 不能用 if (b == a)
       {
           cout << "b == a" << endl;
       }
   
       // sizeof 数组和指针不同
       char arr[] = "hello world";
       const char *ptr = arr;
       cout << sizeof(arr) << endl;   // 12
       cout << sizeof(ptr) << endl;   // 4（32 位系统）
   
       return 0;
   }
   ```

4. **指针参数如何传递内存**

   - 错误示例：`void GetMemory(char *p, int num) { p = (char *)malloc(sizeof(char) * num); }`，`Test` 中 `str` 仍为 `NULL`。
   - 正确方式一：指向指针的指针。
   - 正确方式二：用函数返回值传递动态内存。
   - 错误示例：返回栈内存。

   ```c
   void GetMemory(char *p, int num)
   {
       p = (char *)malloc(sizeof(char) * num);
   }
   
   void Test()
   {
       char *str = NULL;
       GetMemory(str, 100);
       strcpy(str, "hello");   // 错误：str 仍为 NULL，程序崩溃
   }
   
   ```

   ```c
   void GetMemory2(char **p, int num)
   {
       *p = (char *)malloc(sizeof(char) * num);
   }
   
   void Test2()
   {
       char *str = NULL;
       GetMemory2(&str, 100);
       strcpy(str, "hello");
       cout << str << endl;
   
       free(str);
       str = NULL;
   }         //正确方式一
   ```

   ```c
   char *GetMemory3(int num)
   {
       char *p = (char *)malloc(sizeof(char) * num);
       return p;
   }
   
   void Test3()
   {
       char *str = NULL;
       str = GetMemory3(100);
       strcpy(str, "hello");
       cout << str << endl;
   
       free(str);
       str = NULL;
   }   //正确方式二
   ```

   ```c
   char *GetString()
   {
       char p[] = "hello world";   // 栈上数组
       return p;                   // 错误：返回栈内存指针
   }
   
   void Test4()
   {
       char *str = NULL;
       str = GetString();
       cout << str << endl;        // 未定义行为：可能乱码
   }
   ```

5. **free 和 delete 把指针怎么啦**

   - `free` 和 `delete` 只是释放指针所指内存，并未把指针本身干掉。
   - `free(p)` 后 `p` 地址仍不变（非 `NULL`），成为野指针。
   - `if (p != NULL)` 不能防错，因为 `p` 不是 `NULL` 但指向非法内存。\

6. **动态内存会被自动释放吗**
   - 函数体内局部指针变量在函数结束时消亡，但不会自动释放其所指动态内存。
   - 两个“似是而非”特征：指针消亡了，不表示所指内存自动释放；内存被释放了，不表示指针消亡或成 `NULL`。

7. **杜绝“野指针”**
   - 野指针不是 `NULL` 指针，而是指向“垃圾”内存的指针。
   - 成因：
     （1）指针变量未初始化。
     （2）指针 `free` 或 `delete` 后未置 `NULL`。
     （3）指针操作超越变量作用范围。

   ```c++
   #include <cstdlib>
   #include <cstring>
   
   void Test()
   {
       char *p = (char *)malloc(100);
       strcpy(p, "hello");
       free(p);            // p 所指内存被释放，但 p 本身仍保存原地址
   
       if (p != NULL)      // 这个判断没有防错作用
       {
           strcpy(p, "world");  // 错误：p 是野指针
       }
   
       // 正确做法
       p = NULL;
   }
   ```

   

8. **有了 malloc/free 为什么还要 new/delete**
   - `malloc/free` 是 C++/C 标准库函数，`new/delete` 是 C++ 运算符。
   - 非内部数据类型对象创建时要自动执行构造函数，消亡前自动执行析构函数；`malloc/free` 不在编译器控制权限内，不能执行构造/析构。
   - `new` 完成动态内存分配和初始化，`delete` 完成清理与释放。
   - `new/delete` 必须配对，`malloc/free` 也必须配对。

   ```c++
   #include <iostream>
   using namespace std;
   
   class Obj
   {
   public:
       Obj()  { cout << "Obj 构造" << endl; }
       ~Obj() { cout << "Obj 析构" << endl; }
   };
   
   int main()
   {
       // malloc/free 不会调用构造和析构
       Obj *a = (Obj *)malloc(sizeof(Obj));
       // a->Obj();   // 不能这样调用构造函数
       free(a);       // 不会调用析构函数
   
       // new/delete 会调用构造和析构
       Obj *b = new Obj;   // 分配内存并调用构造函数
       delete b;           // 调用析构函数并释放内存
   
       return 0;
   }
   ```

   

9. **内存耗尽怎么办**
   - `malloc` 和 `new` 找不到足够内存时返回 `NULL`。
   - 处理方式：
     （1）判断 `NULL` 后用 `return` 终止本函数。
     （2）判断 `NULL` 后用 `exit(1)` 终止整个程序。
     （3）为 `new` 和 `malloc` 设置异常处理函数。
   - 32 位以上程序因虚存几乎不会“内存耗尽”，但不加错误处理质量很差。

   ```c++
   #include <iostream>
   #include <cstdlib>
   using namespace std;
   
   class A
   {
   public:
       A()  { cout << "A 构造" << endl; }
       ~A() { cout << "A 析构" << endl; }
   };
   
   void Func()
   {
       A *a = new A;
       if (a == NULL)
       {
           cout << "Memory Exhausted" << endl;
           exit(1);   // 终止整个程序
       }
   
       // 使用 a
   
       delete a;
   }
   ```

10. **malloc/free 使用要点**
    - `void *malloc(size_t size);` 返回 `void *`，需显式类型转换。
    - `malloc` 只关心总字节数，不识别类型。
    - 用 `sizeof` 计算字节数。
    - `free` 原型 `void free(void *memblock);`；`p` 为 `NULL` 时 `free` 多次无问题；非 `NULL` 时连续 `free` 两次会运行错误。

    ```c
    #include <cstdlib>
    #include <iostream>
    using namespace std;
    
    int main()
    {
        int length = 10;
    
        // 正确：显式类型转换，用 sizeof 计算字节数
        int *p = (int *)malloc(sizeof(int) * length);
        if (p == NULL)
        {
            return -1;
        }
    
        for (int i = 0; i < length; i++)
        {
            p[i] = i;
        }
    
        // free 后置 NULL
        free(p);
        p = NULL;
    
        // free(NULL) 是安全的，可以多次调用
        free(NULL);
        free(NULL);
    
        return 0;
    }
    ```

11. **new/delete 使用要点**
    - `new` 内置 `sizeof`、类型转换和类型安全检查，比 `malloc` 简单。
    - 非内部类型对象，`new` 创建时完成初始化，可有多种构造函数形式。
    - 用 `new` 创建对象数组只能用无参数构造函数。
    - 释放对象数组不要丢 `[]`。

    ```c
    #include <iostream>
    using namespace std;
    
    class Obj
    {
    public:
        Obj()          { cout << "Obj()" << endl; }
        Obj(int x)     { cout << "Obj(int): " << x << endl; }
        ~Obj()         { cout << "~Obj()" << endl; }
    };
    
    int main()
    {
        // 单个对象
        Obj *a = new Obj;       // 调用无参数构造函数
        Obj *b = new Obj(1);    // 调用带参数构造函数
    
        delete a;
        delete b;
    
        // 对象数组：只能使用无参数构造函数
        Obj *arr = new Obj[3];  // 调用 3 次无参数构造函数
        // Obj *bad = new Obj[3](1);  // 错误：不能给对象数组传构造参数
    
        delete [] arr;          // 正确：释放对象数组必须加 []
        // delete arr;          // 错误：相当于只释放 arr[0]，其余对象泄漏
    
        return 0;
    }
    ```

    

12. **心得体会**
    - 作者曾因怕指针而全用数组，代码笨重；后来改写，代码量缩小到一半。
    - 经验：越怕指针越要用指针；必须养成用调试器逐步跟踪程序的习惯。

---
## 十、第8章 C++函数的高级特性

1. **函数重载概念**
   - 重载起源：自然语言中一词多义；C++ 中语义、功能相似的函数可用同一名字。
   - 类构造函数需要重载。
   - 重载实现：同名函数参数不同可区分；仅返回值类型不同有时不能区分。
   - C++ 调用 C 函数需 `extern "C"`。
   - 全局函数和类成员函数同名不算重载。
   - 隐式类型转换可导致重载二义性。

2. **成员函数的重载、覆盖与隐藏**
   - 重载特征：同一类中，函数名相同，参数不同，`virtual` 可有可无。
   - 覆盖特征：派生类与基类中，函数名相同，参数相同，基类函数必须有 `virtual`。
   - 隐藏规则：
     （1）派生类函数与基类函数同名但参数不同，不论有无 `virtual`，基类函数被隐藏。
     （2）派生类函数与基类函数同名且参数相同，但基类函数没有 `virtual`，基类函数被隐藏。
   - 隐藏导致行为依赖指针类型。
   - 摆脱隐藏：在派生类中显式调用基类函数。

3. **参数缺省值**
   - 【规则8-3-1】参数缺省值只能出现在函数声明中，不能出现在定义体中。
   - 【规则8-3-2】多个参数时，参数只能从后向前挨个儿缺省。
   - 缺省值不赋予新功能，只使书写简洁，可能降低可理解性。
   - 不合理使用缺省值会导致重载二义性。

4. **运算符重载**
   - 用 `operator` 加运算符表示函数。
   - 重载为全局函数时，一元运算符一个参数，二元运算符两个参数；重载为成员函数时，一元无参数，二元只有一个右侧参数。
   - 规则表：
     - 所有一元运算符建议重载为成员函数。
     - `=`、`()`、`[]`、`->` 只能重载为成员函数。
     - `+=`、`-=`、`/=`、`*=`、`&=`、`|=`、`~=`、`%=`、`>=`、`<=` 建议重载为成员函数。
     - 其它运算符建议重载为全局函数。
   - 不能重载：不能改变内部类型运算符；不能重载 `.`、`::`、`?:` 等；不能重载 C++ 运算符集合中没有的符号；不能改变优先级规则。

5. **函数内联**
   - 用内联取代宏代码。
   - C++ 内联函数：编译器在符号表放入声明，调用时检查类型，正确则直接替换函数调用。
   - 内联可操作类私有数据成员，宏不行。
   - 应使用内联函数取代所有宏代码，`assert` 是唯一例外。
   - 编程风格：`inline` 必须与函数定义体放在一起才有效。
   - 慎用内联：函数体代码长、函数体内有循环时不宜内联。

6. **心得体会**  
   C++ 的重载、内联、缺省参数、隐式转换等机制有优点也有隐患，应辨证、恰如其分地使用。

---
## 十一、第9章 类的构造函数、析构函数与赋值函数

1. **基本概念**
   每个类只有一个析构函数和一个赋值函数，但可有多个构造函数。
   若未编写，编译器自动生成四个缺省函数：
   - `A(void);`
   - `A(const A &a);`
   - `~A(void);`
   - `A &operator=(const A &a);`
   缺省拷贝构造和赋值采用“位拷贝”而非“值拷贝”，若类含指针变量，将出错。

2. **构造函数与析构函数起源**
   C++ 将对象初始化放构造函数，清除放析构函数。
   构造函数、析构函数与类同名，析构函数名前加 `~`；没有返回值类型。

3. **构造函数初始化表**
   - 初始化表位于函数参数表之后、函数体 `{}` 之前。
   - 派生类必须在其初始化表里调用基类构造函数。
   - 类的 `const` 常量只能在初始化表中初始化。
   - 非内部数据类型成员对象应采用初始化表，以获取更高效率。
   - 内部数据类型成员两种方式效率几乎无区别。

4. **构造和析构次序**
   - 构造从类层次最根处开始，每层先调用基类构造函数，再调用成员对象构造函数。
   - 析构按与构造相反次序执行。
   - 成员对象初始化次序完全不受初始化表次序影响，只由成员对象在类中声明次序决定。

5. **示例：类 String 的构造函数与析构函数**
   - 普通构造函数。
   - 析构函数。

6. **不要轻视拷贝构造函数与赋值函数**
   - 若类含指针，缺省位拷贝会错误。
   - 拷贝构造函数在对象被创建时调用，赋值函数只能被已存在对象调用。

7. **示例：类 String 的拷贝构造函数与赋值函数**
   - 拷贝构造函数。
   - 赋值函数四步：
     （1）检查自赋值。
     （2）释放原有内存资源。
     （3）分配新内存并复制内容。
     （4）返回本对象引用。

8. **偷懒处理拷贝构造与赋值函数**
   若不想编写，又不允许别人使用编译器缺省函数，可将其声明为私有，不写代码。

9. **派生类中实现类的基本函数**
   - 基类构造函数、析构函数、赋值函数都不能被派生类继承。
   - 派生类构造函数应在初始化表里调用基类构造函数。
   - 基类与派生类析构函数应为虚（加 `virtual`）。
   - 派生类赋值函数不要忘记对基类数据成员重新赋值。

10. **心得体会**
    构造函数、析构函数和赋值函数是类的“Big-Three”，任何类最重要函数，不容轻视。

---
## 十二、第10章 类的继承与组合

1. **继承概念**
   若 A 是基类，B 是 A 的派生类，B 继承 A 的数据和函数。
   继承提高可复用性，但太容易用，要防止乱用。

2. **继承规则**
   - 【规则10-1-1】若类 A 和类 B 毫不相关，不可以为了使 B 功能更多而让 B 继承 A。
   - 【规则10-1-2】若逻辑上 B 是 A 的“一种”（a kind of），则允许 B 继承 A 的功能和属性。
   - 更严格规则：若逻辑上 B 是 A 的“一种”，并且 A 的所有功能和属性对 B 都有意义，才允许 B 继承 A。
   - 反例：鸵鸟是鸟，但鸵鸟不能飞；圆是特殊椭圆，但圆继承椭圆长轴短轴画蛇添足。

3. **组合规则**
   - 【规则10-2-1】若逻辑上 A 是 B 的“一部分”（a part of），则不允许 B 从 A 派生，而是用 A 和其它东西组合出 B。
   - 示例：眼、鼻、口、耳是头的一部分，`Head` 应由 `Eye`、`Nose`、`Mouth`、`Ear` 组合而成，不是派生。
   - “运行正确”的程序不见得是高质量程序。

---
## 十三、第11章 其它编程经验

1. **使用 const 提高函数健壮性**
   - `const` 可修饰函数参数、返回值，甚至函数定义体。
   - 修饰参数：
     - 输出参数不能加 `const`。
     - 输入参数用“指针传递”时加 `const` 防止意外改动。
     - 输入参数用“值传递”时不要加 `const`。
     - 非内部数据类型参数，应改为 `const &`。
     - 内部数据类型参数，不要改为 `const int &x`。
   - 修饰返回值：
     - 指针传递返回值加 `const`，返回值内容不能修改。
     - 值传递返回值加 `const` 无价值。
     - 引用传递返回值多用于类赋值函数，实现链式表达。
   - const 成员函数：
     - 任何不修改数据成员的函数都应声明为 `const`。
     - `const` 关键字只能放函数声明尾部。

2. **提高程序效率**
   - 【规则11-2-1】不要一味追求效率，应在满足正确性、可靠性、健壮性、可读性等质量因素前提下提高效率。
   - 【规则11-2-2】以提高全局效率为主，局部效率为辅。
   - 【规则11-2-3】优化效率时先找限制效率的“瓶颈”。
   - 【规则11-2-4】先优化数据结构和算法，再优化执行代码。
   - 【规则11-2-5】时间效率和空间效率可能对立，应适当折衷。
   - 【规则11-2-6】不要追求紧凑代码。

3. **有益建议**
   - 【建议11-3-1】当心视觉上不易分辨的操作符书写错误。
   - 【建议11-3-2】变量创建后及时初始化。
   - 【建议11-3-3】当心变量初值、缺省值错误或精度不够。
   - 【建议11-3-4】当心数据类型转换错误，尽量显式转换。
   - 【建议11-3-5】当心变量上溢/下溢、数组下标越界。
   - 【建议11-3-6】当心忘记编写错误处理程序。
   - 【建议11-3-7】当心文件 I/O 有错误。
   - 【建议11-3-8】避免编写技巧性很高的代码。
   - 【建议11-3-9】不要设计面面俱到、非常灵活的数据结构。
   - 【建议11-3-10】原有代码质量好则尽量复用；不要修补很差劲的代码。
   - 【建议11-3-11】尽量使用标准库函数。
   - 【建议11-3-12】尽量不要使用与具体硬件或软件环境关系密切的变量。
   - 【建议11-3-13】把编译器选择项设置为最严格状态。
   - 【建议11-3-14】可能时使用 PC-Lint、LogiScope 等工具进行代码审查。

---
## 十四、附录A：C++/C代码审查表

1. **文件结构**
   - 头文件和定义文件名称是否合理？
   - 目录结构是否合理？
   - 版权和版本声明是否完整？
   - 头文件是否使用 `ifndef/define/endif`？
   - 头文件是否只存放声明而不存放定义？

2. **程序的版式**
   - 空行、代码行内空格、长行拆分是否得体？
   - `{` 和 `}` 是否各占一行且对齐同一列？
   - 一行代码是否只做一件事？
   - `if`、`for`、`while`、`do` 是否自占一行并加 `{}`？
   - 定义变量或参数时 `*` 和 `&` 是否紧靠变量名？
   - 注释是否清晰必要？
   - 类结构 `public`、`protected`、`private` 顺序是否一致？

3. **命名规则**
   - 命名规则是否与操作系统或开发工具风格一致？
   - 标识符是否直观可拼读？
   - 长度是否符合“min-length & max-information”？
   - 是否出现相同局部变量和全局变量？
   - 类名、函数名、变量和参数、常量书写格式是否遵循规则？
   - 静态变量、全局变量、类成员变量是否加前缀？

4. **表达式与基本语句**
   - 运算符多时是否用括号确定顺序？
   - 是否编写太复杂或多用途复合表达式？
   - 是否将复合表达式与数学表达式混淆？
   - 是否用隐含错误方式写 `if`？
   - 循环体内逻辑判断且循环次数大时，是否移到循环体外？
   - `case` 是否忘加 `break`？是否忘写 `default`？
   - 使用 `goto` 是否留下隐患？

5. **常量**
   - 是否使用含义直观常量表示多次出现的数字或字符串？
   - C++ 中是否用 `const` 常量取代宏常量？
   - 常量间密切相关时是否在定义中包含关系？
   - 是否误解类中的 `const` 数据成员？

6. **函数设计**
   - 参数书写是否完整？参数命名、顺序是否合理？参数个数是否太多？
   - 是否省略返回值类型？函数名与返回值类型在语义上是否冲突？
   - 是否将正常值和错误标志混在一起返回？
   - 入口处是否用 `assert` 检查参数有效性？
   - `return` 是否返回指向栈内存的指针或引用？
   - 是否使用 `const` 提高函数健壮性？

7. **内存管理**
   - `malloc`/`new` 后是否立即检查 `NULL`？
   - 是否忘记为数组和动态内存赋初值？
   - 数组或指针下标是否越界？
   - 动态内存申请与释放是否配对？
   - 是否有效处理“内存耗尽”？
   - 是否修改“指向常量的指针”的内容？
   - 是否出现野指针？
   - 是否混淆 `malloc/free` 和 `new/delete`？
   - `malloc` 字节数、类型转换是否正确？
   - 创建与释放动态对象数组时 `new/delete` 语句是否正确？

8. **C++函数高级特性**
   - 重载函数是否有二义性？
   - 是否混淆成员函数的重载、覆盖与隐藏？
   - 运算符重载是否符合规范？
   - 是否滥用内联函数？
   - 是否用内联函数取代宏代码？

9. **类的构造函数、析构函数和赋值函数**
   - 是否违背规范让编译器自动产生四个缺省函数？
   - 构造函数是否遗漏初始化工作？是否正确使用初始化表？
   - 析构函数是否遗漏清除工作？是否错写、错用拷贝构造和赋值函数？
   - 赋值函数是否遗漏四步？
   - 是否正确编写派生类构造函数、析构函数、赋值函数？

10. **类的高级特性**
    - 是否违背继承和组合规则？

11. **其它常见问题**
    - 数据类型问题等。

---
## 十五、附录B：C++/C试题

1. **一、零值比较 if 语句（10分）**  
   写出 `BOOL flag`、`float x`、`char *p` 与“零值”比较的 `if` 语句。

2. **二、sizeof 计算（10分）**  
   Windows NT 下 32 位 C++ 程序计算 `sizeof` 值。

3. **三、简答题（25分）**
   - 头文件中 `ifndef/define/endif` 干什么用？
   - `#include <filename.h>` 和 `#include "filename.h"` 有什么区别？
   - `const` 有什么用途？至少说明两种。
   - C++ 中调用被 C 编译器编译后的函数，为什么要加 `extern "C"` 声明？
   - 简述两个 `for` 循环的优缺点。

4. **四、有关内存的思考题（20分）**  
   四段程序，问运行结果。

5. **五、编写 strcpy 函数（10分）**  
   原型：`char *strcpy(char *strDest, const char *strSrc);`

6. **六、编写类 String 的构造函数、析构函数和赋值函数（25分）**

---

## 十六、附录C：试题答案与评分标准

1. **一、零值比较答案**
   - `BOOL flag`：标准 `if (flag)`、`if (!flag)`。
   - `float x`：标准示例：
     `const float EPSINON = 0.00001;`
     `if ((x >= -EPSINON) && (x <= EPSINON))`
   - `char *p`：标准 `if (p == NULL)`、`if (p != NULL)`。

2. **二、sizeof 答案**
   - `sizeof(str) = 6`
   - `sizeof(p) = 4`
   - `sizeof(n) = 4`
   - `void Func(char str[100])` 中 `sizeof(str) = 4`
   - `void *p = malloc(100);` 中 `sizeof(p) = 4`

3. **三、简答题答案**
   - `ifndef/define/endif`：防止头文件被重复引用。
   - `#include <filename.h>`：从标准库路径搜索；`#include "filename.h"`：从用户工作路径搜索。
   - `const` 用途：定义常量；修饰函数参数、返回值，提高健壮性。
   - `extern "C"`：解决 C++ 函数重载导致的名字匹配问题。
   - 两个 for 循环：第一个简洁但多执行 N-1 次逻辑判断，降低效率；第二个效率高但程序不简洁。

4. **四、内存思考题答案**
   - 第一题：程序崩溃。
   - 第二题：可能乱码。
   - 第三题：能够输出 `hello`，但内存泄漏。
   - 第四题：`free` 后未置 `NULL`，`str` 成野指针，继续 `strcpy` 是错误行为。

5. **五、strcpy 答案**
   ```c
   char *strcpy(char *strDest, const char *strSrc)
   {
       assert((strDest != NULL) && (strSrc != NULL));
       char *address = strDest;
       while ((*strDest++ = *strSrc++) != '\0')
           NULL;
       return address;
   }