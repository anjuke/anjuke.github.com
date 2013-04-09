---
layout: post
title: "Homebrew is just like gentoo portage"
date: 2013-01-20 14:28
comments: true
categories:
---

{%img right http://0.gravatar.com/avatar/ce1e13bbf946c92e2abf740f8909bafa %}

## 一句话提示：

>
> 利用`brew versions`和`brew switch`命令也可以实现gentoo portage里eselect类似的版本切换功能!
>

老规矩，喜欢听故事的继续看下去XDDD

<!-- more -->

## 您使用何种包管理？

MacOS下安装软件是件很惬意的事情，因为DMG或者PKG一路NEXT魔法即可(<strike>这么说Windows其实也是XDD</strike>)，卸载也不是什么难事。Linux下安装软件同样...也不能算是一件难事，因为我们有apt、pacman、emerge<strike>甚至是yum</strike>这样的包管理器，一条命令随叫随到。

那么有在Mac下像Linux那样安装软件包的工具吗？如今大家可能第一想到的是[Homebrew][homebrew]（以下简称brew），没错，在他之前就有大名鼎鼎的[Macports][macports]，号称和BSD的ports一样自动下载源代码编译安装的工具。那么为什么Homebrew如今却广受欢迎呢，我想了想

+ 支持最新的MacOS
+ 包含着常用的大多数软件包
+ 更新很快
+ 够Cool! (使用Git方式发布，项目托管在Github上)
+ <strike>是用ruby写的XDDD</strike>

brew里的软件是通过Formula方式写成一个打包文件，git同步后，让brew系统来读取这个Formula进行常见的下载、编译、安装过程。看起来和gentoo portage或者BSD的ports如出一辙(如果使用archlinux，可以理解成ABS)。

平时在使用系统的时候比较大的一项烦恼是，我有同一个软件，有不同的版本想要同时使用？如果软件包本身不妨碍多个版本共存那自然是皆大欢喜，不幸的是多数软件都不支持。进一步讲，我是一个开发者，需要引用不同的第三方库的时候这个问题怎么解决呢？

大家也许想到了开发语言自带的包管理器和版本控制工具，这方面的例子，rvm与gem/bundler首当其冲。假使您还不了解其中的重要性，推荐阅读一下[『The Twelve-Factor App』里关于项目依赖的部分][12app-dependency]。其他的例子，还有python的setuptools/pip等等。自然的，[这个方面做的比较差的也有][PEAR]。

更进一步讲，要是我使用的是系统的库呢，除了像[CocoaPods][CocoaPods]这种吸收了gem的新兴工具外，似乎走进了一个死胡同。其实不然，我们可以选择带有这样功能的包管理器或者系统！

## 这就是为什么我喜爱gentoo与homebrew!

[gentoo portage里有一个叫做slot的功能][gentoo portage slot]，简单地解释一下，在portage的版本树里一个软件包可以有多个版本的ebuild文件，并把他们划分成不同地slot——例如python——大家都知道python目前有2.x和3.x的大版本区别，在portage里python就有几个slot（见下面的portage记录，版本之前括号里的就是slot），其中有2.7和3.2，他们属于同一个包下的多个slot版本；portage允许同一个包里多个slot并存，一般还会附赠一个`eselect-foobar`的包来提供版本的选择

```
[I] dev-lang/python
     Available versions:
	(2.2)	~2.2-r8[1]
	(2.5)	2.5.4-r4 ~2.5.4-r5
	(2.6)	2.6.8 ~2.6.8-r1
	(2.7)	2.7.3-r2 ~2.7.3-r3
	(3.1)	3.1.5 ~3.1.5-r1
	(3.2)	3.2.3 ~3.2.3-r1 ~3.2.3-r2
	(3.3)	**3.3.0 **3.3.0-r1
	{{(-)berkdb bootstrap build doc elibc_uclibc examples gdbm ipv6 +ncurses (+)readline sqlite +ssl tcltk +threads tk +wide-unicode wininst +xml}}
     Installed versions:  2.7.3-r2(2.7)(03:02:44 PM 01/19/2013)(gdbm ipv6 ncurses readline sqlite ssl threads wide-unicode xml -berkdb -build -doc -elibc_uclibc -examples -tk -wininst) 3.2.3(3.2)(03:04:49 PM 01/19/2013)(gdbm ipv6 ncurses readline sqlite ssl threads wide-unicode xml -build -doc -elibc_uclibc -examples -tk -wininst)
     Homepage:            http://www.python.org
     Description:         A really great language

[I] app-admin/eselect-python
     Available versions:  20091230 20100321 ~20111108 **99999999
     Installed versions:  20100321(02:18:04 PM 11/07/2012)
     Homepage:            http://www.gentoo.org
     Description:         Eselect module for management of multiple Python versions
```

对于gentoo portage来说，不同的包版本安装的路径是不同的，例如2.7和3.2分别安装在`/usr/lib/python2.7`与`/usr/lib/python3.2`下，利用特殊的机制（eselect和软链等，python在gentoo下使用了python-wrapper）来实现多版本并存的功能。

不过可以留意到的是，在portage里同一个slot下也会有多个版本，他们之间是无法并存的XDD

### 那么故事讲了这么多Homebrew到底有什么能耐呢？

其实我前几天在自己的Mac姬上需要测试最新版本的zeromq 3.x，而同时另外有一个项目的zeromq是要求2.1.x版本的；按照常规做法，要么我自己手工编译两个lib分开放（意味着编译链接的FLAG都要自己设置，不开玩笑...），要么在开发项目A时重新安装3.x，开发项目B时重新安装2.1.x，岂不蛋疼？

让我们回想一下Homebrew的目录结构：官方建议安装在/usr/local下（因为MacOS不会使用这个目录）

```
├── CONTRIBUTING.md
├── Cellar
├── Library
├── README.md
├── bin
├── etc
├── include
├── lib
├── opt
├── sbin
├── share
├── tmp
└── var
```

bin和lib下是常见的直接使用的文件，实际上是做了软链接到Cellar下具体某个包、某个版本下，例如

```
ls -lah lib/libzmq.a
lrwxr-xr-x  1 aleiphoenix  wheel    36B Jan 16 15:54 lib/libzmq.a -> ../Cellar/zeromq/2.1.11/lib/libzmq.a
```

这下明白了吧，由于brew的实现已经是同一个包不同版本的分开放置，所以要支持多版本切换，只是写一个切换软链接的工具而已，而brew里已经自带了这个功能；并且这个功能比gentoo portage的slot更自由一些，因为任意版本之间都是可以并存的XDDD

+ 查看已经安装的版本 `brew info`

```
brew info zeromq
zeromq: stable 3.2.2, HEAD
http://www.zeromq.org/
Depends on: pkg-config
/usr/local/Cellar/zeromq/2.1.11 (41 files, 1.6M) *
/usr/local/Cellar/zeromq/2.2.0 (41 files, 1.6M)
/usr/local/Cellar/zeromq/3.2.2 (54 files, 2.2M)
```

+ 查看所有版本 `brew versions`

这里有个技巧，由于brew的目录是用git管理的，所以对于Formula来说始终是一个文件（名），利用git命令把需要的版本checkout出来，就可以使用了XDDD

```
brew versions zeromq
3.2.2    git checkout ab8de4b Library/Formula/zeromq.rb
2.2.0    git checkout 6a2e6ef Library/Formula/zeromq.rb
2.1.11   git checkout 497b13a Library/Formula/zeromq.rb
2.1.10   git checkout 4c8ed3a Library/Formula/zeromq.rb
2.1.9    git checkout 381c97f Library/Formula/zeromq.rb
2.1.7    git checkout ed41f79 Library/Formula/zeromq.rb
2.1.8    git checkout 8e045d5 Library/Formula/zeromq.rb
2.1.6    git checkout 460a168 Library/Formula/zeromq.rb
2.1.4    git checkout 83ed494 Library/Formula/zeromq.rb
2.1.3    git checkout f4a925d Library/Formula/zeromq.rb
2.1.2    git checkout 3017b39 Library/Formula/zeromq.rb
2.1.1    git checkout 0476235 Library/Formula/zeromq.rb
2.0.10   git checkout 00e1ae3 Library/Formula/zeromq.rb
2.0.9    git checkout 0527b6f Library/Formula/zeromq.rb
2.0.8    git checkout 1f252bf Library/Formula/zeromq.rb
2.0.7    git checkout ce8d2f5 Library/Formula/zeromq.rb
```

+ 切换至目标版本 `brew switch`

```
brew switch zeromq 3.2.2
Cleaning /usr/local/Cellar/zeromq/2.1.11
Cleaning /usr/local/Cellar/zeromq/2.2.0
Cleaning /usr/local/Cellar/zeromq/3.2.2
49 links created for /usr/local/Cellar/zeromq/3.2.2
```

当然，这个并不表示brew一点问题也没有，比如某个libA引用的是特定版本的libB，在libB切换版本后libA可能就出错了。

> "It's up to you, Mason. It's all up to you."

所以XDDDD

总体而言，brew这个功能给开发者（<strike>不折腾会死星人</strike>）提供了莫大的便利。那天发现这个功能时，不禁感叹，『这不就是gentoo portage么！』 XDDDDDDDDDD

\_\_END\_\_

[homebrew]: https://github.com/mxcl/homebrew
[macports]: http://www.macports.org/
[12app-dependency]: http://www.12factor.net/dependencies
[PEAR]: http://blog.astrumfutura.com/2007/10/to-pear-or-not-to-pear-and-how-to-pear-anyway/
[CocoaPods]: https://github.com/CocoaPods/CocoaPods
[gentoo portage slot]: http://www.gentoo.org/doc/en/handbook/handbook-x86.xml?part=2&chap=1#doc_chap5
