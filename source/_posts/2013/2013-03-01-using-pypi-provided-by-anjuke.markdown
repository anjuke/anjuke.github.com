---
layout: post
title: "使用安居客提供的PyPI镜像"
date: 2013-03-01 10:52
comments: true
categories:
---

主要提示：安居客现在拥有了自己的PyPI镜像，直接使用http://pypi.corp.anjuke.com/simple ，就可以节省重复去公网下载模块的流量与时间。

## Welcome to the Python World!

这篇文章的目标读者是还刚接触Python以及安居客将要使用Python的各位开发者，强烈建议把文中提到的延伸信息都阅读一下，百利无害。

使用Python开发程序是一件轻松惬意的事情，它的第三方模块分发机制让开发者能够很方便快速的发布自己的代码、安装部署使用其他开发者做的模块。（详细请参考[安装Python模块][1]，[发布Python模块][2]）

关于Python打包的方式以及开源软加架构里打包的背景信息，可以阅读[这篇内容][3]了解更多。

## 如何安装第三方的Python模块

标准的Python模块源码包里都会带上一个`setup.py`，它是Python的模块分发机制中的一部分，其中会定义模块的基本信息以及 **依赖** ，后者就是我们重点需要关注的问题。

通常我们只需要执行

```
python setup.py install
```

即可完成安装。

## Python模块的分发

传统模块的分发如果采用原始的人肉方式那就显得不够好了；所以Python社区有了PyPI（Python Package Index），官方的地址是[pypi.python.org][4]，全世界各地还有几个[镜像][5]；以及配套使用的客户端程序，`setuptools`和`pip`

**我们建议使用`pip`**

前者一般在Python安装的时候随同一起分发，后者需要手工安装（`pythonbrew`里则是会自动一起安装）

对于想要使用的模块依赖，使用

```
easy_install foobar

#或者
pip install foobar
```

它的过程就像是ubuntu下的`apt-get`，自动从配置的镜像（默认是官方）自动查找目标模块，下载，（编译）安装，整个过程轻松简单。

## 为什么我们要提供安居客的镜像

出于安居客未来架构发展的需要，我们在开发、生产环境的部署将会更多的采用自动化；项目的依赖是其中一个不可忽视的环节。对于Python来说，快速高效地安装依赖能提高效率，减少错误。

还有一点是，每次安装都从官方下载会造成流量和时间的浪费；更多的，因为某些大家都懂的原因造成一个项目依赖无法安装，很令人恼火。做镜像提高速度，减少问题也是让工程师快乐的一个体现。

## 如何使用安居客的PyPI镜像

**更详细的使用方法请参考`easy_install`和`pip`的文档**

### 安装 (install)

+ easy_install

直接使用easy_install的命令，带上参数指定

```
easy_install -i http://pypi.corp.anjuke.com/simple FOOBAR
```

或者写配置文件`~/.pydistutils.cfg`，内容如下

```
[easy_install]
index_url = http://pypi.corp.anjuke.com/simple
```

+ pip

直接使用pip命令，带上参数

```
pip -i http://pypi.corp.anjuke.com/simple install FOOBAR
```

或者写配置文件`~/.pip/pip.conf`，内容如下

```
[global]
index-url = http://pypi.corp.anjuke.com/simple
```

### 搜索 (search)

**暂未支持**

## 最后

我们希望为工程师、开发者提供更好的环境来支持开发工作。

如果大家在使用中有各种问题，意见和建议，欢迎联系我们。

Happy Hacking ;)

[1]: http://docs.python.org/2/install/index.html
[2]: http://docs.python.org/2/distutils/index.html
[3]: http://www.ituring.com.cn/article/19090
[4]: https://pypi.python.org
[5]: http://www.pypi-mirrors.org


