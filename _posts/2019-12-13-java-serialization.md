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

设想一个这样的场景：我有一个对象序列化后存放在txt文本中，在文本中的其中一个属性的值给修改了但其他成员变量都保持不变，最后在反序列话回来，会不会报异常呢，如果不报异常那这个校验有什么用呢，如果报异常那很好奇啊，一个简单的`long`类型的UID（而不是一个类似于哈希码的东西）就能完成校验了，而且它校验的是什么呢。但仔细一想这个场景下反序列化应该是不会报异常的，要不然那得创建多少个类啊，而且校验应该不是校验属性值的。

再设想一个场景：有两个类`Customer`和`Student`，这两个类除了类名不一样外，所有成员变量都相同。先把Customer object序列化，再反序列化，不过把反序列化的结果类型转换成Student。显而易见这肯定是不成功的，实验结果是也报异常`ClassCastException`，不过这说明反序列化过程是成功的，只是Customer不能转成Student而已。

所以应该校验的场景是：对象序列化了以后，这个文本（暂且先这么叫吧）在“成员变量”（除值外）上发生了变化，也就是说文本被改动了。或者文本是以前序列化的，但是当下的Class变化了。对于前者来说更多的是文件内容损坏吧，因为存在文本中的东西人眼看起似懂非懂（就像下面一样）手动的精确修改应该很有难度，所以很少会人为修改，而且我尝试着删了一个可识别的字母，结果反序列化的时候报`IOException`。

```shell
¬í^@^Esr^@&com.yang.serialization.domain.Customer^@^@^@^@^@^@^@^A^B^@^CL^@^Gcompanyt^@^RLjava/lang/String;L^@^Bidq^@~^@^AL^@^Dnameq^@~^@^Axpt^@
```

那会猜后者的情况应该更常见些，但是当我给`Customer`类增加了一个属性`age`后，人家反序列化成功了，妥妥的，只是age值是null或者0（这个很好理解，本来就没有嘛）。我觉得自己要重新看看文档了，这本该很简单啊，但就是不按套路走啊，有些地方我肯定没有get到啊。最后我把Customer中的`serialVersionUID`的值给改了继续测试，报错信息如下：

```java
java.io.InvalidClassException: com.yang.serialization.domain.Customer; local class incompatible: stream classdesc serialVersionUID = 1, local class serialVersionUID = 2
```

看到`InvalidClassException`我有点明白了，反序列化前后会校验`serialVersionUID`的值，如果不一样那绝对报异常，也就是反序列化失败了。如果一样，反序列化后的结果会”对号入座“，多了或者少了有自己的处理方式，总之不报异常。之前看的stackoverflow上写着：**强烈建议自己显式声明serialVersionUID，serialVersionUID默认生成跟平台有关**。现在明白了，默认serialVersionUID反序列化前后有时会不一样，这样本该是正确的反序列化结果报InvalidClassException异常，这应该就是真正的原因了。

这是个很简单东西，也许现在又钻了一次牛角尖，有时候想着用的时候记着声明下就可以了，再加上现在大多时候用java jacksion，所以可以不用深究的，不过还是过不了心里这一关。