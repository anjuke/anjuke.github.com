---
layout: post
title: "使用Thrift API监控Storm集群"
date: 2012-10-31 18:00
author: jizhang
comments: true
categories: [python]
---

{%img right http://www.gravatar.com/avatar/b881be65f3c4f45dd68bcf0fbe6ba82b.png %}

Storm UI提供了基本的监控界面，可以查看当前时点集群内脚本的运行情况，其中比较重要的是消息吞吐量（Transferred）和处理延迟（Process latency）。不足的是，这套系统没有记录时序数据，因此想看一段时间内的趋势图，或是做脚本上下线的负载监控，Storm UI就无能为力了。

不过，Storm Nimbus开放了一套Thrift API，可以使用他获取各类信息。下面就介绍一下如何使用Python编写监控程序，定时获取脚本运行状态（吞吐量和延迟），并在监控系统中出图。

### 安装Thrift API

[Thrift](http://thrift.apache.org/)是Apache基金会下的一个项目，定义了一套面向服务程序的通信协议。它使用Thrift的[IDL](http://thrift.apache.org/docs/idl/)定义接口，通过一个工具转换成不同语言的客户端，供其他程序调用。

可以[点此下载Thrift源码包](http://archive.apache.org/dist/thrift/0.7.0/)，因为Storm使用的Thrift版本是0.7.0，所以建议安装该版本。

安装过程即 ./configure && make && sudo make install，安装完成后PATH会多出thrift这个命令。

### 生成Storm Thrift API代码文件

Storm Thrift API的定义文件在源码中提供，可以从Github中克隆相应的分支，并执行thrift命令：

```
$ git clone -b 0.7.0 git://github.com/nathanmarz/storm.git
$ cd storm/src
$ thrift --gen py storm.thrift
$ cd gen-py
```

源码中似乎已经提供了py这个文件夹，也可以直接使用。

### 获取某个Topology的吞吐量和延迟

首先在Python项目中添加如下依赖：

```
setup(install_requires=['thrift==0.7.0'])
```

以下代码会找到集群中名为access\_log的脚本，记录filter（Bolt）的吞吐量和延迟数。由于Storm Thrift API返回值的组织方式比较复杂，所以需要多多参考刚才生成的gen-py包中的内容。

```python
#连接到Storm Nimbus Thrift Server：

socket = TSocket.TSocket(self.args.nimbus_host, self.args.nimbus_port)
transport = TTransport.TFramedTransport(socket)
protocol = TBinaryProtocol.TBinaryProtocol(transport)
nimbus = Nimbus.Client(protocol)
transport.open()

#获取集群内脚本信息，得到脚本ID：

cluster_info = nimbus.getClusterInfo()
topology_id = None
for topology in cluster_info.topologies:
if topology.name == 'access_log':
    topology_id = topology.id

# 获取脚本信息，计算吞吐量和延迟毫秒数

topology_info = nimbus.getTopologyInfo(topology_id)

transferred = 0
filter_latencies = []
for executor_info in topology_info.executors:
    # 600表示10分钟内的均值
    for k, v in executor_info.stats.transferred['600'].iteritems():
        transferred += v

    if executor_info.component_id == 'filter':
        latencies = []
        for k, v in executor_info.stats.specific.bolt.process_ms_avg['600'].iteritems():
            latencies.append(v)
        filter_latencies.append(sum(latencies) / len(latencies))
filter_latency = sum(filter_latencies) / len(filter_latencies)
```

然后就可以将计算得到的transferred和filter\_latency输出到监控系统中绘图了。

### 注意事项

在调试的过程中，有时会尝试用telnet查看Storm Thrift Server是否存活，一旦这样操作就会导致Nimbus挂起，无法自动退出。和作者确认过，这是Thrift自身的Bug，目前还没有解决方法。

