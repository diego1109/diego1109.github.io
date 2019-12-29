---
layout: post
title: Tensorflow serving 学习（一）
date: 2019-12-29
categories: 学习笔记
tags: DeepLearning
comments: true 
---

`tensorflow serving`是tf官方推出的专为生产环节设计的机器学习模型服务系统（TensorFlow Serving is a flexible, high-performance serving system for machine learning models，designed for production environments），使用它可以很方便的将训练好的模型部署成web服务，并通过HTTP或GRPC的方式访问。

根据[官方资料](https://www.tensorflow.org/tfx/guide/serving)模仿出demo很容易，但往深走或者做些修改解决场景需求还是着实花了一些功夫的，我承认这跟我不熟悉tensorflow有些关系，但更多的时间花在了理解概念和做实验上了。顺便再吐槽下，官方的introduction和tutorial有点少呀。

tensroflow serving中很重要的概念是`SavedModel`，跟checkpoint中以`.ckpt`或`.h5`的格式存放训练好的模型一样，也是模型的一种存放格式（下图），saved_model是模型的存放路径，1表示模型的版本（version）每个版本都是一个单独的子目录，如若有其他版本，那么版本号一次递增。saved_model.pb是训练好的模型的图，variables是训练好的权重。 tensorflow serving识别这种格式并且通过load它就能把已训练好的模型部署起来。

```shell
saved_model
└── 1
    ├── saved_model.pb
    └── variables
        ├── variables.data-00000-of-00001
        └── variables.index
```

接下来的主要工作就是怎么生成这个SavedModel，官方资料中把这个过程叫做export model。官方资料的代码开起来太费劲了，我在此只想把这个过程讲清楚所以就弄了个简单点的代码。

```python

    export_path_base = "./savedModel"
    export_path = os.path.join(tf.compat.as_bytes(export_path_base),
															tf.compat.as_bytes(str(1)))
    print('Exporting trained model to', export_path)
    
    x = tf.placeholder('float', shape=[None, 3])
    y_ = tf.placeholder('float', shape=[None, 1])
    
    builder = tf.saved_model.builder.SavedModelBuilder(export_path)
    tensor_info_x = tf.saved_model.utils.build_tensor_info(x)
    tensor_info_y = tf.saved_model.utils.build_tensor_info(y)

    prediction_signature = (
        tf.saved_model.signature_def_utils.build_signature_def(
          inputs={'input': tensor_info_x},
          outputs={'output': tensor_info_y},
          method_name=tf.saved_model.signature_constants.PREDICT_METHOD_NAME))

    legacy_init_op = tf.group(tf.tables_initializer(), name='legacy_init_op')
    builder.add_meta_graph_and_variables(
        sess, [tf.saved_model.tag_constants.SERVING],
        signature_def_map={
          'prediction':
              prediction_signature,
      },
      legacy_init_op=legacy_init_op)
```

学习一个东西难点从不在它的语言上，即便当前见到的语言在之前从未遇到过。大多数人包括我在内感觉的是基础的语法都看不懂，那怎么看明白这块代码在做什么事情，更别提根据当前需求对它进行修改了。而现实也的确是这样的，我会先大体看先这段代码，会有那么一丢丢的印象，方法的参数或者它内部又调用了其他方法，接着是查case文档或者是API试图寻找文字性的说明，最后在理解和猜测中调试程序，通过调试再去理解和猜测，反复循环直到“感觉”明白了。从代码着手最后又通过代码检验自己是否学会，但背后的是我在反复的过程中理解它的`概念`和`使用方法`。因为`框架`、`包`、甚至是`语言`这些都是人为设计而非天生存在的，更通俗的说是人按照某个设计思路做出来的解决某个问题的`产品`，而我要做的是学会怎么去用这个产品解决当前场景的需求，仅此而已，所以**理解概念和设计思路（暂且先这么叫吧）就显得十分重要了，其地位远超语言本身。最后掌握的也是使用它的“套路”或者是其中的某几个关键环节。**

### 未写完........

