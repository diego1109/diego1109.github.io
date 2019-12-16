---
layout: post
title: 记一个序列化的困惑
date: 2019-12-16
categories: 学习笔记
tags: Java
comments: true 
---

`序列化（Serialization）`指的是将对象转化为字节流的过程，`反序列化（Deserialization）`顾名思义就是将字节流又重新转化成对象，并且保证转化前后对象的”状态“不变。序列化的目的就是为了将内存中的对象存在硬盘上或者进行网络传输。

序列化是个实例独立的过程，比如对象在一个平台上实例化后可以再其他平台上进行反序列化。但要求这些对象必须实现`Serializable接口`。对象实例化需要注意的地方主要两个，一是：对象的成员变量如果是基础类型，那是可以直接实例化的，如果是应用类型，那么这个成员变量也必须实现Serializable接口。二是：待序列化的对象**强烈建议**显式声明 `final static serialVersionUID`。关于这个 `final static serialVersionUID`一直很疑惑的，源码中给出的解释是这个UID是反序列化时做校验用的，当反序列化后的对象中的serialVersionUID与序列化对象中serialVersionUID不一样时报异常`InvalidClassException`。

设想一个这样的场景：我有一个对象序列化后存放在txt文本中，在文本中的其中一个属性的值给修改了但其他成员变量都保持不变，最后在反序列话回来，会不会报异常呢，如果不报异常那这个校验有什么用呢，如果报异常那很好奇啊，一个简单的String类型的UID（而不是哈希码）就能完成校验了。也许这是个很简单东西，只是现在的我把它给想偏了或者又钻了牛角尖。