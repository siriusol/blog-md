---
title: "Go 并发编程 内存模型"
subtitle: ""
date: 2022-07-09T22:33:20+08:00
draft: true
author: ""
authorLink: ""
authorEmail: ""
description: ""
keywords: ""
license: ""
comment: false
weight: 0

tags:
- draft
categories:
- draft

hiddenFromHomePage: false
hiddenFromSearch: false

summary: ""
resources:
- name: featured-image
  src: featured-image.jpg
- name: featured-image-preview
  src: featured-image-preview.jpg

toc:
  enable: true
math:
  enable: false
lightgallery: false
seo:
  images: []

# See details front matter: /theme-documentation-content/#front-matter
---

<!--more-->

## 内存模型

本节的内存模型它并不是指 Go 对象的内存分配、内存回收和内存整理的规范，它描述的是并发环境中多 goroutine 读相同变量的时候，变量的可见性条件。具体点说，是指在什么条件下， goroutine 在读取一个变量的值的时候，能够看到其它 goroutine 对这个变量进行的写的结果。

由于 CPU 指令重排和多级 Cache 的存在，保证多核访问同一个变量这件事儿变得非常复 杂。毕竟，不同 CPU 架构（x86/amd64、ARM、Power 等）的处理方式也不一样，再加 上编译器的优化也可能对指令进行重排，所以编程语言需要一个规范，来明确多线程同时 访问同一个变量的可见性和顺序。在编程语言中，这个规 范被叫做内存模型。

除了 Go，Java、C++、C、C#、Rust 等编程语言也有内存模型。为什么这些编程语言都 要定义内存模型呢？主要是两个目的。 

* 向广大的程序员提供一种保证，以便他们在做设计和开发程序时，面对同一个数据同时 被多个 goroutine 访问的情况，可以做一些串行化访问的控制，比如使用 Channel 或 者 sync 包和 sync/atomic 包中的并发原语。
* 允许编译器和硬件对程序做一些优化。这一点其实主要是为编译器开发者提供的保证， 这样可以方便他们对 Go 的编译器做优化。

## 重排与可见性

由于指令重排，代码并不一定会按照书写的顺序执行。

举个例子，当两个 goroutine 同时对一个数据进行读写时，假设 goroutine g1 对这个变 量进行写操作 w，goroutine g2 同时对这个变量进行读操作 r，那么，如果 g2 在执行读 操作 r 的时候，已经看到了 g1 写操作 w 的结果，那么，也不意味着 g2 能看到在 w 之前 的其它的写操作。这是一个反直观的结果，不过的确可能会存在。

程序在运行的时候，两个操作的顺序可能不会得到保证，那该怎么办呢？接下 来，我要带你了解一下 Go 内存模型中很重要的一个概念：happens-before，这是用来描 述两个时间的顺序关系的。如果某些操作能提供 happens-before 关系，那么，我们就可 以 100% 保证它们之间的顺序。

## happens-before

在一个 goroutine 内部，程序的执行顺序和它们的代码指定的顺序是一样的，即使编译器 或者 CPU 重排了读写顺序，从行为上来看，也和代码指定的顺序一样。

但是，对于另一个 goroutine 来说，重排却会产生非常大的影响。因为 Go 只保证 goroutine 内部重排对读写的顺序没有影响，比如刚刚我们在讲“可见性”问题时提到的 三个例子，那该怎么办呢？这就要用到 happens-before 关系了。
