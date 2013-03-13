---
layout: post
title: "深入理解daemon"
date: 2012-09-02 13:11
comments: true
categories:
---

{%img right http://0.gravatar.com/avatar/ce1e13bbf946c92e2abf740f8909bafa %}

用linux的各位巨巨应该都知道在系统里有种进程叫做`daemon`，一般理解为后台服务，它有一些特征，比如后台运行，不能直接在终端控制，用户退出登陆后也不会停止等等；有时候我们也想自己运行的脚本能够"后台运行"，往往使用的是`nohup`这个工具。那么daemon到底是什么呢?

(如果以下解释里有任何遗漏或者错误，也欢迎指出)

在许许多多的开源工具(例如[这里][1])里我们都能找到类似如下的代码，这2次fork被称作`unix magic 2 forks`

```python
def daemonize(self):
    """
    do the UNIX double-fork magic, see Stevens' "Advanced
    Programming in the UNIX Environment" for details (ISBN 0201563177)
    http://www.erlenstar.demon.co.uk/unix/faq_2.html#SEC16
    """
    try:
       	pid = os.fork()
        if pid > 0:
                # exit first parent
                sys.exit(0)
    except OSError, e:
        sys.stderr.write("fork #1 failed: %d (%s)\n" % (e.errno, e.strerror))
        sys.exit(1)

    # decouple from parent environment
    os.chdir("/")
    os.setsid()
    os.umask(0)

    # do second fork
    try:
        pid = os.fork()
        if pid > 0:
            # exit from second parent
                sys.exit(0)
    except OSError, e:
        sys.stderr.write("fork #2 failed: %d (%s)\n" % (e.errno, e.strerror))
        sys.exit(1)

    # redirect standard file descriptors
    sys.stdout.flush()
    sys.stderr.flush()
    si = file(self.stdin, 'r')
    so = file(self.stdout, 'a+')
    se = file(self.stderr, 'a+', 0)
    os.dup2(si.fileno(), sys.stdin.fileno())
    os.dup2(so.fileno(), sys.stdout.fileno())
    os.dup2(se.fileno(), sys.stderr.fileno())

    # write pidfile
    atexit.register(self.delpid)
    pid = str(os.getpid())
    file(self.pidfile,'w+').write("%s\n" % pid)
```

某始终没搞明白为什么需要fork 2次，所以带着疑问去找了找根源(网上很多解释都是一笔带过)。不过很遗憾的是文中写到的这个[来源][2]目前已经无法访问到了，幸运的是在『advanced programming in the unix enviroment』这本书里有提到，所以某去找了找(如今我已经买了一本来看XD)。

把关于daemon的一些特征、如何做到daemon以及稍稍探究了以下为何需要这么做，以下内容是某个人总结的。


## daemon 的特征和必要工作

* 总结引用自原书13.3 daemon process

> 避免不需要的交互，并且具有以下一些特征
>
> 1. 调用umask(0)，把创建文件的mask设置成0，以保证daemon本身创建的文件不会继承父进程的mask
> 2. 调用fork，并让其父进程退出，这里有3层含义
>    + 如果父进程是一个shell命令，则退出就使得shell认为该进程已经终止
>    + 子进程会继承父进程的group id，从而确保自己不是group leader (后面一步setsid的先决条件) [关于这个可以参考后面一段9.5 session]
>    + 子进程获得一个新的进程号(PID)
> 3. 调用setsid，创建一个新的session，这里有3个步骤含义
>    + 子进程自己成为新session的leader进程
>    + 子进程自己也会成为新session里唯一一个group的leader
>    + 子进程没有`controlling terminal` [关于这个可以参考后面9.6 controlling terminal]
> 4. 更改工作路径到`/`，unix系统传统上认为一个daemon是从system boot开始一直到system halt/reboot为止一直常驻的进程，如果把工作路径设置到某个挂载上来的设备上，那么系统halt/reboot的时候设备将无法卸载(因为有进程在使用)
> 5. 关于不需要的文件句柄(FD)，避免daemon自身打开从其父进程那里继承过来的任何FD
> 6. 有些daemon会把FD0、FD1、FD2(分别是标准输入、标准输出和标准错误)，重定向到`/dev/null`，确保各种引用lib的流程都无效化(即没有依赖)

典型的来说，daemon已经不再像unix系统所阐述的那样了，一般的可以认为没有交互终端控制访问进程的后台进程都可以认为是以后宗具有daemon性质的进程，所以以上步骤中第1、4步可以选择不做或者根据需要修改。

我们再来看看其中一些有疑点的地方(个人来说)

## 什么是会话(session)

* 总结引用自原书9.5 session

> 一般在进程建立新session的时候，会调用`setsid()`这个函数，它会有3个必要的步骤、特征
>
> 1. 如果调用者(以下称为caller)是一个进程组的leader，那么调用会出错，(我们可以通过fork来避免，父进程退出，子进程继续运行)
>     + 关于进程组的leader，指的就是一系列PIPE或者FORK的第一个进程，例如`cat foo | wc -l`，这里2个进程一起被称作一个group，而`cat`是这个group的leader
> 2. caller本身称为一个session的leader(道理和group类似)，也是这个新session里仅有的进程
>     + caller本身也称为一个group的leader，group id 就是caller的PID
> 3. caller本身不会有`controlling terminal`，如果在`setsid()`之前就有`controlling terminal`，那么调用就会失败
>     + 关于`controlliing terminal`，可以参考下面的9.6

## 什么是控制终端(controlling terminal)

* 总结引用自原书9.6 controlling terminal

> 关于`controlling terminal`，有以下这些特征
>
> 1. 一个session仅有一个终端，可以是真正的终端，也可以是伪终端(例如我们常在X11里用的终端模拟器)
> 2. 建立到`controlling terminal`的进程叫做`controlling process`，通常是`login shell`(常见的例如`/bin/bash --login`)
> 3. 在一个session里，进程组(group)，可以分为一个前台组(foreground group)和若干个后台组(background group) (具体的可以参考unix-like系统的进程控制方面的解说)
>    + 如果有conrolling terminal的情况下，也最多只有一个前台组，其他都是后台组，如果要和controlling terminal通信，需要打开例如`/dev/tty`这样的设备
> 4. 键盘随时按下中断键(通常是`ctrl+c`)，会给所有前台组里的进程发送`SIGINT`的信号
> 5. 键盘随时按下退出键(通常是`ctrl+bs`)，会给所有前台组里的进程发送`quit signal`的信号
> 6. 如果终端接口检测到了网络断开，则会给session leader(同时也是controlling process)发送一个hang-up的信号
>    + 这也就是为什么是session leader子进程的前台组里的程序在我们断开ssh连接后会立即终止的原因

## 但您还未解释为什么需要fork两次?

在前面解释daemon的地方，第2步fork只提到了1次，那么为什么网上那么多daemon的做法里都建议使用2次`fork`呢，原书里也做了一些解释。

* 总结引用自原书9.6 controlling terminal以及3.3 open

> 一些系统在使用daemon时建议调用2次fork，情况主要是这样的
>
> 1. 基于`System V`的Unix系统会为打开一个还未分配到任何session的终端设备，把它当作`controlling terminal`来使用
>    + `System V`在调用`open`时，如果没有指定`0_NOCTTY`这个flag就会这样
> 2. 基于`BSD`的Unix系统会在session leader调用`ioctl`时带上`TIOCSCTTY`参数，分配一个`controlling terminal`；而如果调用时该进程恰巧已经有一个`controlling terminal`，则调用会失败，所以一般来紧跟一个`setsid()`来保证调用的正确。然后在`POSIX.1`规范中，`BSD`系统并不会使用上面提到的`0_NOCTTY`这样的参数
>    + 示例调用代码在原书19.4节

这样一来就可以理解了，<u>**第二次调用`fork`，可以保证第二个子进程不是session leader，而之后的`open`调用，也不会分配到任何`controlling terminal`，这样就保证了daemon的必要条件。**</u>

## 那我们经常使用的nohup到底和daemon有什么区别?

首先，请认真`man nohup` (PIA死)

从[维基百科的页面][3]上，我们会发现解释可以和以上这些信息结合后概括成最终下面的结论

可以看到在session结束后，session leader会收到hang-up的信号，而`nohup`其实时给一个进程**忽略** `SIGHUP`信号，<u>并非真正的daemon</u>，它仅仅提供了一种想退出登陆又不愿意中断进程的选择。

对于类似的功能，我个人强烈推荐tmux，以及类似功能的screen。它们甚至可以把session从controlling terminal上detach，换一个地方再attach上来，<strike>**是不折腾会死星人的好伙伴XDDDDDDDDD**</strike>

## 方便的daemon工具?

而对于希望执行daemon的人来说，除了前面提到的`unix magic 2 forks`(大部分现代编程语言都提供了类似daemon这样的lib)，还可以使用debian等发行版提供的`/sbin/start-stop-daemon`

以上

\_\_END\_\_

[1]: http://www.jejik.com/articles/2007/02/a_simple_unix_linux_daemon_in_python/
[2]: http://www.erlenstar.demon.co.uk/unix/faq_2.html#SEC16
[3]: http://en.wikipedia.org/wiki/Nohup
