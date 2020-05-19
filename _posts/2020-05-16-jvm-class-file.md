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

执行命令，将java文件编译class文件

````shell
javac SimpleUser.java
````

再执行命令，将二进制格式的class文件以十六进制打开。

````shell
hexdump SimpleUser.class
````

编译成对应的class文件的十六进制合适如下所示，java文件编异常class文件后，里面的内容就长这个样子,除了开头几个字节一眼能看明白是什么意思，其他的都得推算。

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/diego1109/diego1109.github.io/images/user_simple2.png">
</div>
使用javap反编译工具，可以直观的查看字节码中所描述的信息。

执行命令 `javap -verbose SimpleUser` 得到的结果在这里：[simple-user-file](https://github.com/diego1109/diego1109.github.io/blob/master/images/simple-user-byte-code)

## class文件中各个部分的作用

### 魔数

魔数（Magic Number）作为是class文件的标志，用来告诉java虚拟机，这是个class文件。魔数是一个4字节无符号整数，并且该值固定：0xCAFEBABE。如果一个class文件不以0xCAFEBABE，虚拟机在执行文件校验的时候会抛异常。在class文件中的位置：行：0000000，列：0~3。

### 版本号

版本号表示当前class文件是由哪个版本的编译器产生的，小版本号在前，大版本号在后，每个版本号占2个字节。（行：0000000，列：4~7）数字：00 00 00 34表示当前编译器的版本是1.8。高版本的虚拟机可以执行低版本编译器生成的class文件，但是低版本虚拟机无法执行高版本编译器生成的class文件。

### 常量池

一个很重要、又有点复杂的地方。 从 `simple-user-file` 的 `Constant pool` 部分可以看出，实例代码中有50个常量。这些常量主要分为两大类：**字面量** 和 **符号引用**：

`字面量` 就相当于java概念中的常量，大概有两种：

1. 文本字符串，比如：`String s = ”abc“`，”abc“会被放到常量池中。
2. 被声明为final的常量的值，比如：`final int TYPE = 1` ，”1“会被放到常量池中。

**有些博客中写道：八种基本类型的值，比如：`int length = 6`，这块的6也会被放到常量池中，我测试时在常量池中没找到6（用的是jdk1.8）。**

`符号引用` 是以一组符号来描述所引用的目标，符号可以是任意形式的字面量，只要使用时可以无歧义的定义到目标即可。 可以理解为：按”一定规则“对java文件中的类、字段、方法、接口等信息进行了另一种”描述“，目的是为了虚拟机在加载时能找到被该符号引用所描述的”对象“在哪。因为在java程序运行时虚拟机需要精确地知道某个类在内存中的地址是多少，但是在java编译成class文件时根本不能确定它的内需地址，所以此时用符号引用来描述这个类，只要虚拟机在使用这个类时能找到它就可以了。符号引用主要包含以下几类：

- 文件中包的名字；
- 类和接口的全限定名称；
- 成员变量（字段）的名称和描述符；
- 成员方法的名称和描述符；
- 方法句柄和方法类型；
- 动态调用点和动态常量；

等等，这些”常量“的名称和描述符都是按一定的规范写死在常量池中的，程序运行时不可能发生变化。当虚拟机在做类加载时，将会从常量池中获得对应的符号引用，再在类创建时或者运行时解析、翻译到具体的内存地址中（将符号引用变成直接引用）。（简单理解就是，开发者写的类、字段、方法都存放在了这个地方，虚拟机运行时自己会来找并且解析的）

**描述符** 的作用是用来描述字段的数据类型、方法的参数列表（包括数量、类型以及顺序）和返回值。

直到目前，总共有17中不同的类型的常量，每种常量都是一个独立的数据结构，（在常量池中的其他部分会存出现指向这些数据节结构的索引）。常量池类型如下图所示：



<div align="center">
    <img src="/Users/lyy/Documents/diego1109.github.io/images/constant-pool-kinds.png" style="zoom:80%;" />
</div>
[网上找的17种常量类型总表](https://blog.csdn.net/qq_39375211/article/details/79925127)，从这里可以看出，常量池中保存了，常量的长度、名称、字面量值、符号引用的索引(类中字段如果引用类型，则用索引指向该符号引用真正所在的地方)等信息。

### 访问标志

常量池之后紧跟着就是访问标志了，占1个字节。这个标志用来表示Class是接口还是类，以及它的修饰符是什么，是否被声明为final，等等。

### 当前类、父类和接口

访问标记结束后，会指定该类的类别、父类类别以及实现的接口。格式如下：

````c
u2	this_class;
u2	super_class;
u2  interfaces_count;
u2  interfaces[interfaces_count];
````

this_class 和 super_class都是2字节无符号整数，他们指向常量池中的CONSTANT_Class，以表示当前的类和父类 ，因为一个类可以实现多个接口，因此需要以数组的形式保存多个接口的索引。

### 类的字段

在类描述后，会有类的字段信息。因为一个有多个字段，所以需要先指明字段的个数。

````c
u2 fileds_count;
field_info	fileds[fields_count]
````

fileds_countb是2字节无符号整数，表示字段的数量。field_info表示字段的具体信息：

````C
field_info {
    u2	access_flags;
    u2	name_index;
    u2	descriptor_index;
    u2	attributes_count;
    attributes_info	attributes[attributes_count]
}
````

- access_flags: 访问标记，该字段是public还是private，或者.......。
- name_index: 字段的名字，不是直接存储名字，而是这块存放了一个指向常量池中的CONSTANT_Utf8结构的索引。

- descriptor_index：表示字段的类型，这块也是个索引，指向了常量池中的一个CONSTANT_Utf8结构数据。

类的字段部分所包含的固定数据项到descriptor_index为止就全部结束了。后面紧跟的attributes_count、attributes_info[attributes_count]是属性表集合，用来存储一些额外的信息，比如，某个int类型的属性有初始值，那么这个属性集合就会被用到了。

### 类的方法

class文件对于类的方法的描述和字段的描述用了几乎完全一致的方式。类的方法信息由两部分构成：

````C
u2 method_count;
method_info	methods[method_count]
````

method_info表示字段的具体信息：

````C
method_info {
    u2	access_flags;
    u2	name_index;
    u2	descriptor_index;
    u2	attributes_count;
    attributes_info	attributes[attributes_count]
}
````

方法表的结构如同字段表一样，依次包括访问标志、名称索引、描述符索引、属性表集合。方法的定义可以通过前面这四个信息来表达清楚，那方法里面的代码去哪里了呢？方法里的java代码，经过javac编译器译成字节码指令后，存放在方法属性表集合中一个名为”Code“的属性里面。

### 类的属性

属性表（attribute_info）会在多个场合出现，class文件、方法表、字段表都可以携带自己的属性表集合，以描述某些场景专有的信息。为了能正确解析Class文件，期初只定义了9项属性，知道Java SE 12版本中，预定以属性已经增加到了29项。这块只介绍属性表集合中Code属性以及跟它相关的属性，下表中只是罗列了29项属性中其中4项：

| 属性名称           | 使用位置 | 含义                                                         |
| ------------------ | -------- | ------------------------------------------------------------ |
| Code               | 方法表   | java代码编译成的字节码指令                                   |
| LineNumberTable    | Code属性 | Java源码的行号与字节码指令的对应关系                         |
| LocalVariableTable | Code属性 | 方法的局部变量描述                                           |
| StackMapTable      | Code属性 | JDK 6中新增的属性，供新的类型检查验证器检查和处理目标方法的局部变量和操作数栈所需要的类型是否匹配 |

上表的第二列”使用位置“意思是：class文件由大概11个基本结构组成，这个属性会在哪个基本结构中出现。上表的意思是：表中第一行显示Code属性会在方法表中出现，其余三个属性将会在Code属性中出现。

#### code属性

这块的内容参考[simple-user-file](https://github.com/diego1109/diego1109.github.io/blob/master/images/simple-user-byte-code)中的结果来看，会更直观些。

在类的方法那部分提到过，java程序方法体里面的代码经过javac编译器处理后，最终会变成字节码指令存储在Code属性内。Code属性出现在方法表的属性集合之中，但并非所有的方法表都必须存在这个属性，比如接口或者抽象类的方法中就不存在Code属性，如果方法表中有Code属性存在，那么它的结构将如下表所示：

| 类型           | 名称                   | 数量                   |
| -------------- | ---------------------- | ---------------------- |
| u2             | attribute_name_index   | 1                      |
| u4             | attribute_length       | 1                      |
| u2             | max_stack              | 1                      |
| u2             | max_locals             | 1                      |
| u4             | code_length            | 1                      |
| u1             | code                   | code_length            |
| u2             | exception_table_length | 1                      |
| exception_info | exception_table        | exception_table_length |
| u2             | attributes_count       | 1                      |
| attribute_info | attributes             | attributes_count       |

- **attribute_name_index** 是一项指向CONSTANT_Utf8_info型常量的索引，此常量值固定为”Code“，它代表了该属性的属性名称。

- attribute_length表示属性的长度。

- max_stack代表了操作数栈深度的最大值。在方法执行的任意时刻，操作数栈都不会超过这个最大深度。虚拟机在运行时需要根据这个值来分配栈帧中操作数栈深度。

- max_locals代表了局部变量表所需的存储空间。max_locals的单位是变量槽（Slot），变量槽是局部变量分配内存所使用的最小单位。对于byte、char、float、int、short、boolean和returnAddress等长度不大于32位的数据类型，每个局部变量占用一个槽，double和long这种64位长度的数据类型则需要两个槽来存放。

    java虚拟机会将局部变量表中的变量槽进行重用，当代码执行超出一个局部变量表的作用域时，这个局部变量表占用的变量槽可以被其他的局部变量表使用，javac编译器会根据变量做用户来分配变量槽给各个变量使用。

- code_length 代表字节码长度。

- code 用来存储java源程序编译后生成的字节码指令。

- exception_table_length和exception_table表示显式异常表，异常表对于Code来说并不是必须存在的（有些方法没有显式抛异常）。异常处理表告诉一个方法该如何处理字节码中可能抛出的异常。

到现在为止，Code属性的主体部分就介绍完了，但是Code属性中还包含更多的信息，这些信息都以属性的形式内嵌在Code属性中，除了类、方法、字段可以内嵌属性外，属性本身也可以内嵌属性。Code属性的中的attribute_info属性就里就包含着 **LocalVariableTable** 、**LineNumberTable** 和 **StackMapTable**。

- **LineNumberTable** 用来记录字节码偏移量和行号的对应关系，在软件调试时，该属性有至关重要的作用，若没有它，调试器无法定位到对应的源码。
-  **LocalVariableTable** 这是局部变量表，它记录了一个方法中所有的局部变量。（java程序运行时，这货在栈帧中躺着）。
- **StackMapTable** 在Class文件做类型校验时会用到该文件。



## 总结

- 梳理这块的只是感觉好枯燥，不过后面的类加载机制肯定会用到这些的，梳理下还是很有必要的。

- 常量池中存放了类、字段、方法的名字，如果字段是个字面量，那么它的值也被存放在这里了。

- 常量池里存放了方法、类的符号引用，如果字段是引用类型的，那么它的符号引用也被存放在这里了。

- 字段的描述性信息（修饰符、字段名索引、描述符、额外属性等）都存放在字段表集合中。

- 方法的描述性信息都存放在方法表集合中。

- 方法体中的java代码被编译成字节码后，存放在Code属性中，而Code属性是方法表中的attribute_info的一部分。（说白了，方法体编译成的字节码也存放在方法表中）。

    