---
layout: post
title: VScode连接远程服务器上的jupyter notebook
date: 2019-12-18
tag: Tool
comments: true
---

**工欲善其事，必先利其器**，开发工具这个东西觉得折腾下还是有好处的。但常常感觉专门抽出时间搞这个浪费时间，更常见的现象是已经明显感觉到当前的开发工具用的很别扭，而且告诉自己等这个忙完了要搭一个更方便的工具，到最后却没下文了直到下次再次遇到这种感觉。我这会就是再次遇到了，想用VSCode连接服务器上的jupyter notebook运行tensorflow代码，这样在本地的VScode中直接写代码就方便了很多。整个过程很简单，我自诩记性也不错，但还是不如这白纸黑字来的保险，查资料也是很花时间的。

首先是本机与服务器之间配置ssh就不仔细描述了，要是忘了google一下“ssh远程登录服务器”大把都是资料而且大多数说的都是对的。但最好在~/.ssh/config中按照下面的样子再配置下，ssh用起来会更方便的。

```shell
Host remote_server
	HostName 119.254.92.61
	User xuser
	IdentityFile ~/.ssh/id_rsa
```

接下来是vscode这边要能远程连接到服务器上，记住不是在本地写代码然后再发送到服务器上，而是直接连接到了服务器的某个路径下，VScode对文件的增删改查就相当于是操作了服务器上这个路径下的对应文件（也许说的比较啰嗦，但是觉得概念还是要清楚的）。实现这个目的只需要3步：

1. 在`扩展（EXTENSIONS）`中搜插件`Remote - SSH`安装后再重新启动VScode。
2. 鼠标点击VScode左下角的齿轮选择`命令模式（command paletten）`,mac对应的快捷键是`shift+cmd+p`。
3. 在VScode顶部中间弹出的下拉菜单中输入`Remote - SSH`点击图片中选中的选项，接下来再点击你要连接的服务器的名字就行了，最后会弹出一个新的VSCode。

vscode现在就可以远程连接服务器了，如果想写python代码，直接创建文件就可以了。

<div align="center"><img src="/images/vscode-connet-server.png" width="50%"></div>

<div align="center"><img src="/images/server-vscode.png" width="50%"></div>

而服务器这边要能够创建jupyter noteboot，也就是安装些了，安装不难就是找起来有点麻烦。我喜欢用conda安装一个虚拟环境就是因为隔离了干净省心，真要是搞坏了直接删了重新建一个。服务器上的操作也只需要3步：

1. 安装虚拟环境：

   ```shell
   conda create --name notebook python=3.6
   ```

2. 激活虚拟环境并安装jupyter notebook：

   ```shell
   source activate notebook
   conda install -c conda-forge jupyter notebook
   ```

3. 创建一个notebook服务：

   ```shell
   sudo jupyter notebook --port=8889 --allow-root
   ```

   结果如下：最下面的两个URL就是刚才启动的服务的地址，我复制`http://localhost:8889/?token=aef9a514fa484b83aa4554371024ebc5b50bbed25c2521ab`，当然复制另一个也没问题。

   <div align="center"><img src="/images/create-notebook-on-server.png" width="100%"></div>

 最后在已经连接到服务器的VScode中进入命令模式，点击下图下拉菜单中被选中的选项（好绕口，理解就好）。意思也很明显：指定一个本地或者远程的jupyter服务连接。

<div align="center"><img src="/images/connet-special-notebook.png" width="100%"></div>

把刚才复制的URL粘贴进去，按回车。

<div align="center"><img src="/images/url.png" width="100%"></div>

<div >

创建一个jupyter文件测试下：



<div align="center"><img src="/images/test-notebook.png" width="100%"></div>



整个过程就这么简单而且内容也不多，但就是写了快两个小时吧，正好有今晚有时间就整理一下，以后就不需要google再去各种找了。后面几张大图看起来好丑，感觉以后要学一些有关排版设计的内容了，忽然想起自己曾今自学了一段时间PS，好久没用这会好像也忘差不多了。