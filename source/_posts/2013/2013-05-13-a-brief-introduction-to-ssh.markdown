---
layout: post
title: "A Brief Introduction to SSH"
date: 2013-05-13 10:53
comments: true
categories:
---

## 为什么会有这篇介绍？

不少人对ssh有疑问或者不明白，典型的例子是在gitcorp/github上添加了公钥后发现无法clone或者push仓库里的代码。

## 前言

ssh是个很大的话题，会发现牵涉到unix-like下很多东西，这里只谈谈我们日常工作中接触最多的那部分

哦，对了，这里的ssh不是Java里的SSH（Spring/Structs/Hibernate）。我也是前几天刚明白XDD

## 参考资料

+ [ssh - how it works](http://www.eng.cam.ac.uk/help/jpmg/ssh/ssh-detail.html)
+ [Secure Shell](http://en.wikipedia.org/wiki/Secure_Shell)
+ [An Illustrated Guide to SSH Agent Forwarding](http://www.unixwiz.net/techtips/ssh-agent-forwarding.html)

<!-- more -->

## 介绍

ssh是secure shell的缩写，它是一种网络协议。主要目的是为了提供和远程主机的交互式操作（典型的就是unix-like下的shell了），由服务器和客户端组成。同时它能提供安全加密的链路保障通信不被监听。

历史上它有版本1和版本2，我们最常见的一般都是版本2。以及最常见的实现是开源的openssh

## ssh操作的基本原则

ssh工具的主要目的是为了保障客户端和服务器间安全的通信操作，所以会有大致3步

1. 客户端建立到目标主机，而不是其他主机
2. 服务器接受客户端的连接，而不是其他客户端
3. 客户端需要让服务器信任为授权的用户

为了完成这几个工作，ssh在建立会话前需要完成几项工作

+ 主机识别 确保连接的是目标主机
+ 加密 建立一条端到端的加密通信链路
+ 授权 验证连接上来的用户是经过授权的

### 主机识别

比较常用的是RSA验证，主机上运行的ssh服务端会生成一段RSA私钥/公钥对，在客户端连接时提供身份验证。服务器把自己的公钥发送给客户端，客户端用它加密一段随机的session key并发回服务器端，服务器端利用自己的私钥解密这段session key用于之后的通信

比较常见的服务器检查，会看到客户端连接时会出现这样一段信息

```
Host key not found from the list of known hosts.
Are you sure you want to continue connecting (yes/no)?
```

客户端同意后会把该主机的key添加到自己的known hosts（`~/.ssh/known_hosts`）列表里，以后一旦连接该主机都会验证key是否一致，如果不一致会警告存在中间人攻击。

### 加密

客户端和服务器间会利用RSA交换不同的cipher确保会话通信都是加密的。窃听通信需要知道这些key来解密会话内容

### 授权

会话建立后客户端需要让服务端明白他是否有权限访问，典型的就是unix-like系统本身的用户，常见的授权方式有密码、RSA私钥/公钥对验证

这里就要稍微介绍一下私钥/公钥对验证了，原理上和前面主机验证的流程是一致的。

客户端生成一对私钥/公钥，然后把公钥上传到服务器上（比如给服务器的管理员），通常是在远端主机用户下的`~/.ssh/authorized_keys`里。

在授权的时候，服务端利用该私钥/公钥对发送一段加密的信息给客户端解密后返回服务端，如果验证通过即可表示授权通过。

**私钥即代表了一个用户身份，所以私钥要妥善保管不要被别人获取到。另外推荐定时更换自己的私钥。**

## openssh里提供的其他工具

上面介绍一下了ssh通信的一些基本原则，这里在介绍一下openssh带的其他工具

ssh-keygen

用于生成ssh私钥/公钥对的工具。为了保证加密的破解难度，推荐使用4096bit以上的RSA key。

另外，私钥本身是可以带passphrase的，这样保证了即使私钥泄露，不知道passphrase的人还是无法使用这个私钥。

ssh-agent

它是一个用于管理记忆ssh私钥的工具，在ssh客户端连接服务端进入授权阶段，openssh客户端会利用它访问已经添加过的私钥，而不需要每次都重新寻找。当它进程本身被kill后，私钥需要重新添加

ssh-add

用于往ssh-agent里添加私钥，被passphrase加密过的私钥此时需要提供passphrase。也可以通过`ssh-add -l`和`ssh-add -L`来查看已经添加了哪些私钥

## Git ssh hosting 里和ssh的关系

最简单的Git ssh hosting，就是原始的ssh访问目录，使用者把自己的公钥上传到服务端的`authorized_keys`里

而比较常见做用户权限管理的git管理是gitolite，比如Gitlab 5.0前和gitcorp目前都是在使用它。它利用在`authorized_keys`里可以为每个公钥指定运行的脚本，把授权处理的逻辑放到脚本里去判断（用户是否对某个路径有权限）

使用

```
ssh git@git.corp.anjuke.com
```

访问如果出现询问登录密码，则说明公钥没有添加成功

## 一些使用技巧和问题答疑

### 已经添加了公钥，但访问仍旧询问需要密码？

这个问题本身原因比较多，常见的是公钥没添加对。把自己的公钥写到服务端对应用户的`~/.ssh/authorized_keys`里

服务端sshd配置是否StrictMode了，是的话，`~/.ssh`目录以及`authroized_keys`文件需要是600权限

如果确保权限和公钥都添加正确还是不行。请检查一下sshd配置里MaxAuthRetries的值。Debian默认是6。也就是说如果agent里私钥比较多，正确的私钥在第7个，那么到第6次服务器就拒绝授权了。这种时候解决办法有几种，1. 把agent kill掉，换一下添加私钥的顺序。2. 给sshd配置里把retry次数加上去。 3. 改`~/.ssh/config` 给目标host指定`IdentityFile`。推荐第3种

客户端连接的时候用户是否正确？ ssh默认使用当前shell用户名，如果远端用户名不一样要显式指定，如`ssh username@remotehost`

### 非标准的ssh端口如何连接？

`ssh -p 2222 foo@bar`

rsync通信使用ssh呢？ssh不是标准端口呢？

`rsync -e ssh foo bar`以及`rsync -e 'ssh -p 2222' foo bar`

### 加密过的密钥，每次连接的时候提示需要密码？（sudo ssh连接就需要询问密码了？）

ssh-agent没利用好。其实ssh客户端连接授权的时候是看`$SSH_AGENT_PID`这个环境变量的。先检查这个环境变量的值和ssh-agent进程的PID是否一致？ 这个自己手动写脚本管理比较麻烦，推荐使用`keychain`工具

在`~/.bash_profile`里添加下面的内容

```
if [[ -n "$(command -v keychain)" ]]; then
  /path/to/keychain -q
  /path/to/keychain -q /path/to/your/private/key
  . ~/.keychain/$HOSTNAME-sh
fi
```

这样每次打开shell，keychain都会检查一下ssh-agent是否启动，没有的话会加载私钥，export那个环境变量

至于sudo ssh就没法使用公钥授权了，是因为sudo不会把这些环境变量带到sudo执行的命令里；使用`sudo -E`即可。

### 每次添加加密的密钥好麻烦，有工具可以自动输入密码吗？

请放狗搜索`expect ssh passphrase`。至于expect脚本，请妥善保管，因为里面写了passphrase...Good Luck!

### forward_agent是干啥的？

和ssh-agent同理，但可以把客户端的所有agent里的私钥信息带到服务器上。举例：

客户端A可以通过公钥授权访问主机B和主机C，但和主机C链路不通。主机B和主机C链路是通的。

那么`ssh -A hostB`登录后再`ssh hostC`就可以通过公钥授权了。线上堡垒机就是这么玩的

相对的，使用`ssh -a`来关闭forward功能

### 堡垒机需要连2次ssh才能登上线上服务器

<strike>可以上去开个2222端口什么的嘛，反正只禁止了22 XDDDDD</strike>

写个脚本加到自己$PATH里去，比如叫stargate

```
#!/bin/bash

ssh -A username@10.10.6.226 -t "ssh $1"
```

以后上线上机器，直接`stargate app10-045`

不过，有些环境变量可能会有小问题，比如xterm了（tmux下的配色哟...）

### ssh建立连接很慢

多数情况是因为服务端反向查询DNS造成的，sshd的配置里`UseDNS no`，重启

如果没用...恩... `ssh -v` 分析一下过程吧

### `sshd_config`, `ssh_config`和`~/.ssh/config`

一般sshd的配置文件在`/etc/ssh/sshd_config`里，定义了ssh服务端的配置

`/etc/ssh/ssh_config`，对于该系统全局的ssh客户端配置

比较常用的就是`~/.ssh/config`了，这里可以定义该用户自己ssh客户端的配置，想知道有多少配置？`man ssh_config`。这里举例一些比较有用的吧

```
Host * # 写*表示对全部主机使用
    ServerAliveInterval 60 # 和Server连接保持60s的心跳，防止broken pipe之类的会话断开
    GSSAPIAuthentication no # 不使用GSSAPI授权，用的比较少，但不少sshd上默认就开了，会多一次授权验证
    ControlMaster auto # 这和下面一句表示连接相同用户/主机/端口组合的时候利用同一条链路，减少网络开销
    ControlPath /tmp/ssh-%r@%h:%p
    Compression yes # ssh通信压缩，值越大压缩比越高
    CompressionLevel 7
    Cipher blowfish # 指定cipher的方式
```

看到很多人访问一些主机参数比较多的时候会`ssh -i ~/.ssh/gitcorp/id_rsa.pub -p 2222 git@git.corp.anjuke.com`，敲好多字... `~/.ssh/config`里可以自己定义ssh的主机名字，比如

```
host gitcorp
    hostname git.corp.anjuke.com
    user git
    port 2222
    IdentityFile ~/.ssh/gitcorp/id_rsa.pub
```

以后就可以直接使用`ssh gitcorp`或者`git clone gitcorp:foo/bar`了

### capistrano里使用ssh的注意点

ssh选项使用`ssh_options[:option_name] = value` 来添加。不是所有option都支持，具体要看`Net::SSH`的实现。

需要注意的地方

```
ssh_options[:forward_agent] = true # 开启forward
ssh_options[:compression] = false # capistrano 似乎不支持压缩通信，如果看到zlib报错之类的，基本都是这个原因
ssh_options[:port] = 2222 # 使用非标准端口
```

capistrano太重啦，用mina吧

### 利用ssh隧道建立socket代理（常用于梯子，**你懂得**）

```
ssh -D 7070 -qTfnN foo@bar
```

`ssh -L`还可以用于直接访问IDC DB的3306哦☆（ゝω・）

### X11 over ssh

远程桌面？VNC？弱暴了好不好！！！！（因为很重要，所以加了4个感叹号）

更重要的是，ssh本身是加密的...

服务端确保sshd的配置

```
AllowTcpForwarding yes
X11Forwarding yes
X11DisplayOffset 10
X11UseLocalhost yes
```

```
ssh -Y foo@bar

# 或者
ssh -X foo@bar
```

好了，打开你心爱的图形软件吧

X支持只开需要的软件的图形，比那啥好多了

```
gedit

# 上上网？
firefox
```

如果你对带宽有信心。。

```
startx
```

然后Xorg本身就是支持多个X实例的。。。所以完全可以在一台机子上跑多个X。别忘记unix-like的系统是多用户的。。

<strike>话说。。。有个shell再上个emacs/vim不就好干活了吗。。XDDDD </strike>

## 总结

更多技巧和知识，请认真`man ssh` `man sshd` `man ssh_config` `man sshd_config`。

\_\_END\_\_
