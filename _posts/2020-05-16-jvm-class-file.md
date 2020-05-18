---
layout: post
title: jvm-class文件
date: 2020-05-16
categories: 学习笔记
tags: Jvm
comments: true
---

开发者编写的java代码是先编译成了字节码存放在 `.class` 文件中（开发者编写的影响程序执行每一个单词、字母、数字都会编译成对应的字节码，在class文件中恰当的地方放着），之后java虚拟机加载class文件并执行其中的字节码，从而将程序运行起来。这篇文章主要介绍class文件的结构以及各部分的作用。

## class文件整体结构

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/diego1109/diego1109.github.io/images/jvm-class-file.png">
</div>

## class文件中各个部分的作用

