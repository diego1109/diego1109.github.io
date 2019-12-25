---
layout: post
title: Tensorflow save and restore tutorial
date: 2019-12-24
categories: 学习笔记
tags: DeepLearning
comments: true 
---

记得刚开始上手tensorflow的时候被各种写法搞的晕头转向的，看到checkpoint总是莫名的会紧张些，它就像开发中的持久化一样，把刚才在内存中跑的结果给保存到磁盘上用于日后的使用。但他又比mysql更黑盒些，只能看到文件名以及文件的解释，对于里面到底是什么呢，一点也都不知道。至于存储的对错与否，不要意思，即使是restore了也不能精确的确认，除非出现大bug导致程序跑不通，那会才知道“哦哦，保存有错误”而已。不过现在看来这些担心和心里的紧张，大都只会出现在那些本身就不自信的小白身上，因为他们知道的却清楚，才月会对自己写的代码有信心。

[这篇文章的内容大多是翻译过来的，原文在这里。这个blog不知道是什么时候写的，但是看它最早的评论的时间是在三年前](https://cv-tricks.com/tensorflow-tutorial/save-restore-tensorflow-models-quick-complete-tutorial/)

这篇文章主要解释了四个问题：

- tensorflow模型的文件结构是怎么样的。
- tensorflow模型该怎么保存。
- 预测或者迁移学习的时候该怎么restore模型。
- 怎么使用导入的预训练模型进行再次训练（fine-turning）或者修改。

### 1. TF模型的文件结构

当我们训练完毕一个神经网络后，我们要把训练好的网络保存起来日后使用或者部署到生产环节。那么，什么是tensorflow（tf）模型（model）？tf model主要包括网络的设计（图，graph）和我们训练好的网络参数。因此，tf model有两个主要的文件：（1）`元图（meta graph）`它是个保存tf graph的协议缓存（protocol buffer）比如像变量、操作、集合，以及他们之间的前后关系，这个文件的后缀名是`.meta`。（2）`检查点文件（checkpoint file）`这是个二进制文件里面包含着网络中所有变量的值，这些值是用来对号入座填到元图的变量中的，文件扩展名是`.ckpt`。不过随着tensorflow版本的升级这个文件发生了改动，0.11版以后的ckpt文件有两个文件构成：

```python
# 0.11版以前的模型结构
inception_v1.meta
inception_v1.ckpt
checkpoint
```

```python
# 0.11版以后的模型结构
inception_v1.meta
inception_v1.index
inception_v1.data-00000-of-00001
checkpoint
```

`checkpoint`文件中只是做为一个记录，保存了着最新的checkpoint文件的名称。

### 2. TF模型的保存

在tensorflow中要想保存`图`和`参数`，就必须先创建出`tf.train.Saver()`的对象。而且只有当变量在session中时，它才值有值的，要不然只是一个”壳“而已。

