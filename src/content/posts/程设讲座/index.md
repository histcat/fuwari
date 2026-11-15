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

## cmake
