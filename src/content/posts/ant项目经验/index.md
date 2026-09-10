---
title: Ant项目经验
category: CS
published: 2026-09-10
tags: []
draft: false
---

高考后CS复健啊

学学CS61A找找手感，没想到被Ant项目打趴下了

你别说，学了c++再学python是有点难度（

# python中的list与c++的数组
## 一
python中是先创建一个对象，把名字创建一个指向对象的指针

如
```python
t = [1, 2, 3]
```
实际上是先创建一个没有名字的list，再把t指向这个没有名字的list

而c++的数组是直接开辟了一块新的内存，连续的，数组直接存储元素

python中的list如果是存的是对象的话（eg list， 自定义的class），存的是引用

python中的等号是assign，也就是说，如果
```python
t = [1, 2, 3]
a = t
t.append(1)
```
a也是[1, 2, 3, 1]了

## 二
list也是每一个位置指向一个指针，而不是每一个位置创建一个内存空间

# 神秘的闭包
https://chat.deepseek.com/share/q6kavzxuhmd4j65eki