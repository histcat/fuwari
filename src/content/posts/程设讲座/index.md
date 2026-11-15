---
title: 程设讲座
category: ''
published: '2026-11-15T05:58:45.631Z'
tags: []
draft: true
---

# 计算机基础导论 
编译型、解释型语言

编译器：IR intermediate representation
n中高级语言 -> IR
IR -> m 中cpu的汇编语言

c++(高级语言) -> 编译器 -> 汇编 -> 汇编器 -> 机器码 -> 链接器 -> 可执行文件 

程序是对计算过程的描述

main 栈空间
     堆空间
     全局区域

ALU 

cpu寄存器 eg. register int 寄存器变量 访问更快

c语言是如何发明的？
《论自然语言编程的愚蠢》（
自然语言难以翻译到机器语言
写形式化的语言

eg. lean

## 指令集架构 ISA
程序计数器
分支就是改一下程序计数器

两派：精简指令集架构RISC
     复杂指令系统 CISC

## 存储层次
寄存器 -> 缓存 Cache（介于之间 eg. L1, L2, L3） -> 主内存 RAM -> 固态硬盘
慢 便宜

DRAM SRAM

5级流水cpu
 IF instruction fetch
 ID instruction decode
 EX execute
 MA memory access
 WB write
第一条指令ID的时候 第二条就可以IF 了

摩尔定理 单位面积上的晶体管~18个月翻一番

OOM out of memory

访存能力

## 缓存的作用
时间局限性 同一个地址被经常访问
空间局限性 i i+1 i+2

链表比数组慢很多 因为链表没有空间局限性 没被放在缓存中
数组有预取 把接下来的都放入cache中

## ALU 算术逻辑答案

时序逻辑 和 组合逻辑
  寄存器      ALU

数字电子技术

如何实现时序逻辑 D 触发器 与 寄存器
如何实现组合逻辑 逻辑门 AND OR NOT NAND
eg. 加法 减法

半加器 
全加器 
in A B C in
out S C out

串行进位加法器

如何实现乘法 把所有的部分积求出来再加起来
二叉树优化乘法 
乱序执行 

线程 一核 多线程 

也有意义

上下文切换 切成好多时间片，时间共享机制


# c++程序与工程实践

## 编译器
gcc g++

gcc : gnu（开源软件协会） compiler collection

汇编 -> c子集的编译器 -> exe
再用 子集的exe -> 编译C

CPython Pipy

编译器： 源代码 -> 可执行文件

编译的过程： 
### 预处理
预处理器 （展开#xxx）  不能展开多次 写一个
\# pragma once 
也可以写 
\# IFNDEF XXX1
\# DEFINE XXX1
内容
\# ENDIF
防止多重展开


eg. 如何写 多文件c++
写头文件 **声明**写在头文件中
.cpp 实现文件放定义

### 编译

#### Frontend

Tokenizer 分词器
Parser 生成表达式树 具体语法树 检查 生成 抽象语法树
生成IR 

#### Backend

IR -> 汇编语言

### 后处理
Assembly 
Link Lib ：linker


### g++ 语法
```shell
g++ -Wall -std=c++17 -Ox（优化级别） -o 输出文件名称 输入文件.cpp
```
-Iinclude 建议去include文件夹去找include

## Shell
```shell
pwd
cd
ls
ls -a
cat ：输出文件
echo  原模原样打印出来/
which

mkdir
touch xx.txt 创建文件
编辑 vim / nano

cp 复制 cp A B 从A 复制到 B
mv 剪切
rm / rmdir
rm -r 递归地删除

输入输出重定向
./code > aa.txt
./code < in.txt
./code 2> err.txt 
./code 2> /dev/null 丢弃

管道运算符 |

把前一个输出送到输入

eg.

cat xxx.txt | grep PRINT -c (count)

// stdin stdout 标准输入输出
// stderr 走 cerr 没有缓冲区 


linux的权限

chmod +x xxx.sh
```

环境变量：打印没有前缀的命令会去$PATH下找
### 工作目录和相对路径
./ 从当前路径开始

## git

工作区 现在的内容
暂存区 
本地仓库 新增一个检查点

git add .
git commit -m "提交信息"

提交信息的格式：
fix: bugs in xxx
feat: 新增feature
docs: 新增文档


git reset -soft/hard 

## cmake & make
makefile 一个菜谱

输入 
```makefile
产物：依赖项

变量定义
CXX := g++
CPPFLAGS := -Iinclude

$(CXX) $(CPPFLAGS)
```

```cmake


```

```shell
make -jxx xx核上并行编译
```

make可以增量构建 

c没有包管理器

# C++ 语法

C++ Primer 
C++ Primer Plus

多写一些项目

## 程序的行为
Well-Defined
Ill-Defined
Implementation-Defined Behavior 不同的编译器的行为不同 eg. int的长度
Unspecified Behavior 未指定行为 文档没写
UB Undefined Behavior 没被明确定义 结果无法确定

间接修改const变量
```
const int a = 10;
int *p = const_cast<int *>(&a);
*p = 20;
```

非法解引用
```cpp
int *p = nullptr;
*p = 10;
```

## 程序编译流程
preparation后的源文件 叫做 翻译单元 Translation Unit

## 预处理指令
\# ifndef MACRO
\#  define MACRO
\# endif

宏函数 最外面一定加上括号

```cpp
#define SQUARE(x) x * x

SQUARE(2 + 3) 与预期不一样
```

不建议写宏函数

## Once Definition Rule
TODO:
inline
extern
static
是啥

## 声明和初始化
声明
定义
定义的时候可以Initialization
初始化不是赋值

## 基础类型
void *
### 浮点类型

## 常量和变量
包括 Literal Enumerator constexpr 编译时可以确定的const变量 常量表达式 eg. const int K = N * M + 1 (如果n，m是确定的)

变量： eg `int a = 2; const int d = a; int m[d]`; 报错！

## 表达式和语句
形式文法与产生柿
Formal Grammer 和 Production Rule
语句中不可分割的符号成为 终结符 Terminal Symbol

## 表达式
Expression 由 Operator运算符 和 Operand 操作数 组成

## 求值顺序
操作式 求值分为两部分：Value Computation 和 Side Effect

取余满足 (a / b) * b + (a % b) == a

## 位运算

a ^= b ^= a ^= b

## 逻辑表达式
短路求值

## 类型转换
c风格转换 (double)10
函数风格 double(10)

具名类型转换
static_cast 静态类型转换
const_cast 强行去掉const限定
dynamic_cast
reinterpret_cast重解释转换

## 优先级
只告诉如何添加括号
## 值类别
左值 右值
左值：能取地址的
右值：不能取地址的

右值包括 
盲值xva
lue 苟活一会 类型与右值引用的值 
和 
纯右值prvalue 临时对象

## 语句
switch定义变量建议一定要有作用域`{}`

do while()**;**


for的小括号的定义的语句也在同一个作用域里

## 基于范围的循环

:

## 数组
数组类型是完整类型 

数组本身也是一个类型

sizeof(T[N]) == sizeof(T) * sizeof (N)

数组在进行计算和传参的时候会发生退化，转换到指向首元素的指针

多维数组是 元素类型为数组的数组

## 指针
用于保存另一个变量的地址，从而实现间接访问
NULL 或者 nullptr 表示空指针
指针类型的长度为实现定义，与指向的类型无关，通常为32位/64位
取地址和解引用
const的引用
指针数组和数组指针

### 指针算数
#### 合法的指针
指向一个对象或者数组 或者 最后一个元素 或者空指针

p 指向 a[i], q 指向 a[j]

p + k 指向 i + k

p - q 表示 i - j 的值，如果没有指向

T t[N][M] 访问 t[0][M] == t[1][0] 但是是一个 UB

a[i] = *(a + i)

```cpp
a[1]
1[a]
是一个东西。。。
```

## 动态内存分配

new delete

### 二维数组的动态分配

如果多次分配会导致内存不连续

## 引用

引用必须初始化，并要绑定到一个左值
不能重新绑定

引用不合法的对像 为 Dangling Reference

指针一定存在

但是引用可以优化到宏

### 常值引用
const T &r1 = ;

可以绑定到右值和左值
eg 右值的话

```cpp
int foo();
const int &r3 = foo();
```

### 右值引用

### 引用折叠

只要有一个左值引用就是左值，否则就是右值

## 类型推导
decltype 推导 

# 函数

## 函数重载

形式参数 小对象
和
实际参数 大对象

Parameter 和 Argument

## 匿名函数

## 作用域和生命周期
###
在同一个作用域中，是实体的名字不能重复
### 生命周期
自动储存器：非静态局部变量和形参，版存在栈上，退出作用域小王
静态存储器：
动态存储器：new/delete手动管理

### 存储类别

自动
静态 全局 static（静态局部变量：相当于全局变量，但只能被当前看见；静态全区变量，只能被当前翻译单元看见）

### 链接性
无链接，内部链接性，外部链接性

## 复杂度分析
主定理

## 函数模板
