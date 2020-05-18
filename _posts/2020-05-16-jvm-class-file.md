---
layout: post
title: jvm-class文件
date: 2020-05-16
categories: 学习笔记
tags: Jvm
comments: true
---

开发者编写的java代码是先编译成了字节码存放在 `.class` 文件中（开发者编写的影响程序执行每一个单词、字母、数字都会编译成对应的字节码，在class文件中恰当的地方放着），之后java虚拟机加载class文件并执行其中的字节码，从而将程序运行起来。这篇文章主要介绍class文件的结构以及各部分的作用。

## class文件整体结构和格式

class文件中的基本结构如下图所示：

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/diego1109/diego1109.github.io/images/jvm-class-file.png">
</div>

在java虚拟机规范中，Class文件使用一种类似于C语言结构体的方式进行描述，并且使用统一的无符号整形数作为基本数据类型，由u1、u2、u4、u8分别表示1个字节、2个字节、4个字节、8个字节整数（1字节=8位）。class文格式：

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/diego1109/diego1109.github.io/images/class-file-format.png" width="50%" height="50%">
</div>

## 代码示例

文章中使用如下代码介绍class文件中的各个部分：

````java
public class SimpleUser {

  public static final int TYPE = 1;

  private int id;
  private String name;

  public int getId() {
    return id;
  }

  public void setId(int id) throws IllegalStateException {
    try {
      this.id = id;
    } catch (IllegalStateException e) {
      System.out.println(e.toString());
    }
  }

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }
}

````

编译成对应的class文件的十六进制合适如下所示，java文件编异常class文件后，里面的内容就长这个样子。

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/diego1109/diego1109.github.io/images/simple_user.png">
</div>

## class文件中各个部分的作用

### 魔数

魔数（Magic Number）作为是class文件的标志，用来告诉java虚拟机，这是个class文件。魔数是一个4字节无符号整数，并且该值固定：0xCAFEBABE。如果一个class文件不以0xCAFEBABE，虚拟机在执行文件校验的时候会抛异常。



