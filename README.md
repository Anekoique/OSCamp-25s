# OpenCamp-25s Record

前置基础 

Rust：学习了一周多的基础语法，完成官方版本的rustling

OS：OSTEP 和 MIT xv6的lab

## Toc

**Part1 Rust**

* [Day   1   (2025-03-20)](#0)
* [Day   2   (2025-03-21)](#1)

**Part2 rCore**

* [Day   3   (2025-03-23)](#2)
* [Day   4   (2025-03-25)](#3)
* [Day   5   (2025-03-26)](#4)
* [Day   6   (2025-03-27)](#5)
* [Day   7   (2025-03-28)](#6)

**Part3 ArceOS**

- [Day   8   (2025-03-30)](#7)
- [Day   9   (2025-03-31)](#8)
- [Day   10   (2025-04-01)](#9)
- [Day   11   (2025-04-02)](#10)
- [Day   12   (2025-04-04)](#11)
- [Day   13   (2025-04-05)](#12)
- [Day   14   (2025-04-07)](#13)
- [Day   15   (2025-04-08)](#14)
- [Day   16   (2025-04-09)](#15)
- [Day   17   (2025-04-10)](#16)
- [Day   18   (2025-04-11)](#17)
- [Day   19   (2025-04-14)](#18)
- [Day   20   (2025-04-15)](#19)
- [Day   21   (2025-04-16)](#20)

## Part 1 Rust

语言只是工具，在学完基础语法做完rustling后可以直接学习rcore，不用拘泥于语法细节，阅读rcore框架代码的过程也是学习rust的一种方式，而且相对于啃语法更高效

可供参考的rust学习资料：（ps：ai是最好的老师:dog:

[the rust programming language(cn)](https://rustwiki.org/zh-CN/book/)

[the rust programming language(en)](http://doc.rust-lang.org/stable/book/)

[the cargo book](https://doc.rust-lang.org/cargo/index.html)

[Rust_Cource](https://course.rs/about-book.html)

[Effective Rust](https://github.com/rustx-labs/effective-rust-cn)

<span id="0"></span>

### Day 1

完成rustling的basic部分

<span id="1"></span>

### Day 2

完成rustling的algorithm部分

## Part 2 rCore

没有阅读过太多rust代码，上手框架代码有一定难度，但是通过lab是不难的，相对与xv6的lab没有那么tricky。tutorial book内容较多，可以粗略阅读后直接阅读框架代码然后不懂的再找book和ai

<span id="2"></span>

### Day 3

阅读ch1，ch2，ch3，完成lab1

ch1：实现不依赖标准库的hello world

ch2：内核与应用解耦，实现批处理系统

ch3：实现并发执行与任务调度

<span id="3"></span>

### Day 4

阅读ch4，完成lab2

ch4：实现虚拟内存管理

<span id="4"></span>

### Day 5

阅读ch5，完成lab3

ch5：引入进程管理

<span id="5"></span>

### Day 6

阅读ch6，完成lab4

ch6：实现简单文件系统

<span id="6"></span>

### Day 7

阅读ch7，ch8，完成lab5

ch8：引入锁、信号量、条件变量，实现并发

## Part 3 ArceOS

<span id="7"></span>

### Day 8

阅读ArceOS book ch0 ch1 ch2

<span id="8"></span>

### Day 9

整理前两阶段完成过程，补充过程记录

记录ArceOS tutorial ch0 ch1学习过程，撰写文档

<span id="9"></span>

### Day 10

记录ArceOS tutorial ch2 ch3学习过程，撰写文档

<span id="10"></span>

### Day 11

记录ArceOS tutorial ch4 ch5学习过程，撰写文档

<span id="11"></span>

### Day 12

记录ArceOS tutorial ch6 ch7学习过程，撰写文档

<span id="12"></span>

### Day 13

完成exercises/print_with_color

<span id="13"></span>

### Day 14

完成exercises/support_hashmap

<span id="14"></span>

### Day 15

完成exercises/alt_alloc

学习主线arceos组件axalloc

<span id="15"></span>

### Day 16

学习主线arceos组件axtask

<span id="16"></span>

### Day 17

学习主线arceos组件axmm

<span id="17"></span>

### Day 18

学习主线arceos组件axdriver

<span id="18"></span>

### Day 19

完成exercises/ramfs_rename

学习主线arceos组件axfs

<span id="19"></span>

### Day 20

完成exercises/sys_mmap

了解异构内核，宏内核

<span id="20"></span>

### Day 21

完成exercises/simp_hv

了解hypervisor