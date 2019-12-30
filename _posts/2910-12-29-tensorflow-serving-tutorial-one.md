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

tensroflow serving中很重要的概念是`SavedModel`，跟checkpoint中以`.ckpt`或`.h5`的格式存放训练好的模型一样，也是模型的一种[存放格式（下图）](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/python/saved_model/README.md)，saved_model是模型的存放路径，1表示模型的版本（version）每个版本都是一个单独的子目录，如若有其他版本，那么版本号一次递增。saved_model.pb是训练好的模型的图，variables是训练好的权重。 tensorflow serving识别这种格式并且通过load它就能把已训练好的模型部署起来。

```shell
saved_model
└── 1
    ├── saved_model.pb
    └── variables
        ├── variables.data-00000-of-00001
        └── variables.index
```

接下来的主要工作就是怎么生成这个SavedModel，官方资料中把这个过程叫做**export model**。官方资料的代码开起来太费劲了，我在此只想把这个过程讲清楚所以就弄了个简单点的代码（如下）。输出路径和输入输出维度、类型这个就不再啰嗦了。tensorflow采用`SavedModelBuilder`模块导出模型（当然还有别的export方法），它将与训练的模型的snapshort保存到硬盘上以便之后在的inference时直接load。先构建`builder`对象；再用`builder.add_meta_graph_and_variables(...)`方法将与训练模型的图和参数值添加到builder中；最后调用`builder.save()`将pretrain model保存成saved model。

```python
    # 输出路径
    export_path_base = "./savedModel"
    export_path = os.path.join(tf.compat.as_bytes(export_path_base),
															tf.compat.as_bytes(str(1)))
    print('Exporting trained model to', export_path)
    
    # 输入输出变量的维度和类型
    x = tf.placeholder('float', shape=[None, 3])
    y_ = tf.placeholder('float', shape=[None, 1])
    
    # 构建SavedModel
    builder = tf.saved_model.builder.SavedModelBuilder(export_path)
    tensor_info_x = tf.saved_model.utils.build_tensor_info(x)
    tensor_info_y = tf.saved_model.utils.build_tensor_info(y_)

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
    builder.save()
```

那剩下需要详细介绍的就是`add_meta_graph_and_variables(...)`方法了，貌似现在它变的很核心了（就使用而言它的确很核心），里面重要的是sess、tags、signature_def_map三个参数，[源码在这里](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/python/saved_model/builder_impl.py)。

```python
  def add_meta_graph_and_variables(self,sess,tags,signature_def_map=None,
       assets_list=None,clear_devices=False,init_op=None,train_op=None,
       strip_default_attrs=False,saver=None):
```

- `sess`里面包含有我们打算保存的训练的模型，这个在理解上不会有困惑的。

- `tags`是我们给即将生成的SavedModel中保存的图（meta graph）打的标签，tf规定每SavedModel中的每个meta graph都必须指定标签，用于表示meta graph的作用和使用场景，比如meta graph是用来做train还是serve，它需要用到CPU还是GPU。而且在load SavedModel时只有tags匹配，loader API才能加载成功，否则报错。[tf提供了四种常见的标签](https://www.tensorflow.org/versions/r1.15/api_docs/python/tf/saved_model/tag_constants)。因为我们创建的SavedModel是用做serve的，所以上面代码中的`tf.saved_model.tag_constants.SERVING`作为标签。

- `signature_def_map`**的作用是指定了导出模型的类型和指定当启动infenrece过程时输入和输出的tensor**，不可谓不重要。对这块的数据组织格式刚开始有点误解，后来有点想明白了，介绍性的文字说明不能吝啬的。接下来从头捋一下，看看signature_def_map是怎么来的。
  `x`、`y_`是我们定义输入、输出tensor，先要把`tensor`转换成`TensorInfo proto`结构的数据，为什么要转换，是因为tf人家的产品这么设计的（这个解释当然不具有说服性，但作为使用产品的用户，不管设计师的考量如何，还是理解和接受吧，主要精力还是应该放在基于它的应用开发上，出于兴趣自己可以研究研究），当然tf也同时提供了[转换工具](https://www.tensorflow.org/versions/r1.15/api_docs/python/tf/saved_model/utils)可供用户调用，所以不必纠结于为什么要转换，调两行代码轻松搞定继续后面的开发吧。

  ```python
      tensor_info_x = tf.saved_model.utils.build_tensor_info(x)
      tensor_info_y = tf.saved_model.utils.build_tensor_info(y_)
  ```

  `signature_def_utils.build_signature_def(...)`方法是SavedModel提供[构建signature的API](https://www.tensorflow.org/versions/r1.15/api_docs/python/tf/saved_model/signature_def_utils)之一。这个方法有下面三个参数：

  - `inputs={'images': tensor_info_x}`指定了输入的tensor info。
  - `outputs={'scores': tensor_info_y}`指定了输出的tensor info。
  - `method_name`指定了inference过程调用的方法，如果是预测请求，则method_name的值应该指定为`tensorflow/serving/predict`，[当前不要费时间问为什么，产品就是这么设计的，更详细的信息在这里。](https://www.tensorflow.org/versions/r1.15/api_docs/python/tf/saved_model/signature_constants) 

  下面在代码中我们prediction_signature中构建了一个signature，注意看下，prediction_signature是tuple类型的，当时很困惑为什么不是单独的signature呢？后来有点明白了，对于同一个inference过程，tf serving支持多种格式的数据输入，比如：inference是用来做猫狗分类的，图片也许是以tensor格式输入的，也有可能是以base64格式输入的，在里面再转换成tensor，只需要调用不同的method_name就可以了。

  ```python
  prediction_signature = (
          tf.saved_model.signature_def_utils.build_signature_def(
            inputs={'input': tensor_info_x},
            outputs={'output': tensor_info_y},
            method_name=tf.saved_model.signature_constants.PREDICT_METHOD_NAME))
  ```

  接下来只需要给prediction_signature配上key构建个map怼到对应的参数上去就行了。

  ```python
  builder.add_meta_graph_and_variables(...,
          signature_def_map={'prediction':prediction_signature,},
        legacy_init_op=legacy_init_op)
  ```

  

整个SavedModel的制作过程到现在算是介绍完了，不过还需要再加点东西这块才算比较全面些：

1. 当SavedModel的serve启动后，该怎么根据signature组织相应的HTTP POST RequestBody数据呢。
2. 怎么把已有的SavedModel 载入后稍加修改再保存成一个新SavedModel。
3. 怎么把已有的checkpoint（.pb、.ckpt、.h5）转化成SavedModel。
4. SavedModel支持自定义结构数据输入，比如：图片以base64格式输入。

**一些感触：**

学习一个东西难点从不在它的语言上，即便当前见到的语言在之前从未遇到过。大多数人包括我在内感觉的是基础的语法都看不懂，那怎么看明白这块代码在做什么事情，更别提根据当前需求对它进行修改了。而现实也的确是这样的，我会先大体看先这段代码，会有那么一丢丢的印象，方法的参数或者它内部又调用了其他方法，接着是查case文档或者是API试图寻找文字性的说明，最后在理解和猜测中调试程序，通过调试再去理解和猜测，反复循环直到“感觉”明白了。从代码着手最后又通过代码检验自己是否学会，但背后的是我在反复的过程中理解它的`概念`和`使用方法`。因为`框架`、`包`、甚至是`语言`这些都是人为设计而非天生存在的，更通俗的说是人按照某个设计思路做出来的解决某个问题的`产品`，而我要做的是学会怎么去用这个产品解决当前场景的需求，仅此而已，所以**理解概念和设计思路（暂且先这么叫吧）就显得十分重要了，其地位远超语言本身。最后掌握的也是使用它的“套路”或者是其中的某几个关键环节。**

