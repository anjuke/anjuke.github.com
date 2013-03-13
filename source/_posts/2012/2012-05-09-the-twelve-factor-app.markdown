---
layout: post
title: "The Twelve-Factor App"
author: erning
date: 2012-05-09 17:48
comments: true
categories: [guide, arch]
toc: true
---

{%img right http://0.gravatar.com/avatar/7cdf5b1c46308979e3bf81390b0c8639 %}

中文翻译：[梁山](https://github.com/liangshan/)
英文原文：[Adam Wiggins](http://www.12factor.net/)

_[翻译问题反馈](https://github.com/anjuke/12factor/issues)_

#### 简介

如今，软件通常会作为一种服务来交付，它们被称为“互联网应用程序”（web apps），或“软件即服务”（SaaS）。这篇“___互联网应用的十二要素___”为构建如下的互联网应用程序提供了指导方法：

* 使用**标准化**流程自动配置，从而使新的开发者花费最少的学习成本加入这个项目；
* 和操作系统之间尽可能的**划清界限**，在各个系统中提供**最大的可移植性**；
* 适合**部署**在现代的**云计算平台**，从而在服务器和系统管理方面节省资源；
* 将开发环境和生产环境的**差异降至最低**，并使用**持续交付**实施敏捷开发；
* 可以在工具、架构和开发流程不发生明显变化的前提下实现**扩展**；

这套理论适用于任意语言和后端服务（数据库、消息队列、缓存等）开发的应用程序。

#### 背景

本文的贡献者参与过数以百计的应用程序的开发和部署，并通过[Heroku][1]平台间接见证了数十万应用程序的开发，运作以及扩展的过程。

本文综合了我们关于SaaS应用几乎所有的经验和智慧，是开发此类应用的理想实践标准，并特别关注于应用程序如何保持良性成长，开发者之间如何进行有效的代码协作，以及如何[避免软件污染][2]。

我们的初衷是分享在现代软件开发过程中发现的一些系统性问题，并加深对这些问题的认识。我们提供了讨论这些问题时所需的共享词汇，同时使用相关术语给出一套针对这些问题的广义解决方案。本文格式的灵感来自于Martin Fowler的书籍：[*Patterns of Enterprise Application Architecture*][3]，[*Refactoring*][4]。

[1]: http://www.heroku.com/
[2]: http://blog.heroku.com/archives/2011/6/28/the_new_heroku_4_erosion_resistance_explicit_contracts/
[3]: http://books.google.com/books/about/Patterns_of_enterprise_application_archi.html?id=FyWZt5DdvFkC
[4]: http://books.google.com/books/about/Refactoring.html?id=1MsETFPD3I0C


#### 读者应该是哪些人？

任何SaaS应用的开发人员；部署和管理此类应用的运维工程师。

<!-- more -->

## I. 基准代码
___一份基准代码（*Codebase*)，多份部署(*deploy*)___

12-Factor App(译者注：应该是说一个使用本文概念来设计的应用，下同)通常会使用版本控制系统加以管理，如[Git][5], [Mercurial][6], [Subversion][7]。一份用来跟踪代码所有修订版本的数据库被称作 *代码库* （code repository, code repo, repo）。

在类似SVN这样的集中式版本控制系统中， *基准代码* 就是指控制系统中的这一份代码库；而在Git那样的分布式版本控制系统中， *基准代码* 则是指最上游的那份代码库。

![一份代码库对应多份部署](/medias/20120509/codebase-deploys.png)

基准代码和应用之间总是保持一一对应的关系：

* 一旦有多个基准代码，就不能称为一个应用，而是一个分布式系统。分布式系统中的每一个组件都是一个应用，每一个应用可以分别使用12-Factor进行开发。
* 多个应用共享一份基准代码是有悖于12-Factor原则的。解决方案是将共享的代码拆分为独立的类库，然后使用[依赖管理](#dependencies)策略去加载它们。

尽管每个应用只对应一份基准代码，但可以同时存在多份部署。每份 *部署* 相当于运行了一个应用的实例。通常会有一个生产环境，一个或多个预发布环境。此外，每个开发人员都会在自己本地环境运行一个应用实例，这些都相当于一份部署。

所有部署的基准代码相同，但每份部署可以使用其不同的版本。比如，开发人员可能有一些提交还没有同步至预发布环境；预发布环境也有一些提交没有同步至生产环境。但它们都共享一份基准代码，我们就认为它们只是相同应用的不同部署而已。

[5]: http://git-scm.com/
[6]: http://mercurial.selenic.com/
[7]: http://subversion.apache.org/


## II. 依赖
___显式声明依赖关系(*dependency*)___

大多数编程语言都会提供一个打包系统，用来为各个类库提供打包服务，就像Perl的[CPAN][8]或是Ruby的[Rubygems][9]。通过打包系统安装的类库可以是系统级的（称之为"site packages"），或仅供某个应用程序使用，部署在相应的目录中（称之为"vendoring"或"bunding"）。

**12-Factor规则下的应用程序不会隐式依赖系统级的类库。** 它一定通过 *依赖清单* ，确切地声明所有依赖项。此外，在运行过程中通过 *依赖隔离* 工具来确保程序不会调用系统中存在但清单中未声明的依赖项。这一做法会统一应用到生产和开发环境。

例如，Ruby的[Gem Bundler][10]使用`Gemfile`作为依赖项声明清单，使用`bundle exec`来进行依赖隔离。Python中则可分别使用两种工具 -- [Pip][11]用作依赖声明，[Virtualenv][12]用作依赖隔离。甚至C语言也有类似工具，[Autoconf][13]用作依赖声明，静态链接库用作依赖隔离。无论用什么工具，依赖声明和依赖隔离必须一起使用，否则无法满足12-Factor规范。

显式声明依赖的优点之一是为新进开发者简化了环境配置流程。新进开发者可以检出应用程序的基准代码，安装编程语言环境和它对应的依赖管理工具，只需通过一个 *构建命令* 来安装所有的依赖项，即可开始工作。例如，Ruby/Bundler下使用`bundle install`，而Clojure/[Leiningen][14]则是`lein deps`。

12-Factor应用同样不会隐式依赖某些系统工具，如ImageMagick或是`curl`。即使这些工具存在于几乎所有系统，但终究无法保证所有未来的系统都能支持应用顺利运行，或是能够和应用兼容。如果应用必须使用到某些系统工具，那么这些工具应该被包含在应用之中。

[8]: http://www.cpan.org/
[9]: http://rubygems.org/
[10]: http://gembundler.com/
[11]: http://www.pip-installer.org/en/latest/
[12]: http://www.virtualenv.org/en/latest/
[13]: http://www.gnu.org/s/autoconf/
[14]: https://github.com/technomancy/leiningen#readme


## III. 配置
___在环境中存储配置___

通常，应用的*配置*在不同[部署](#codebase) (预发布、生产环境、开发环境等等)间会有很大差异。这其中包括：

* 数据库，Memcached，以及其他[后端服务](#backing-services)的配置
* 第三方服务的证书，如Amazon S3、Twitter等
* 每份部署特有的配置，如域名等

有些应用在代码中使用常量保存配置，这与12-factor所要求的**代码和配置严格分离**显然大相径庭。配置文件在各部署间存在大幅差异，代码却完全一致。

判断一个应用是否正确地将配置排除在代码之外，一个简单的方法是看该应用的基准代码是否可以立刻开源，而不用担心会暴露任何敏感的信息。

需要指出的是，这里定义的"配置"并**不**包括应用的内部配置，比如Rails的`config/routes.rb`，或是使用[Spring][15]时[代码模块间的依赖注入关系][16]。这类配置在不同部署间不存在差异，所以应该写入代码。

另外一个解决方法是使用配置文件，但不把它们纳入版本控制系统，就像Rails的`config/database.yml` 。这相对于在代码中使用常量已经是长足进步，但仍然有缺点：总是会不小心将配置文件签入了代码库；配置文件的可能会分散在不同的目录，并有着不同的格式，这让找出一个地方来统一管理所有配置变的不太现实。更糟的是，这些格式通常是语言或框架特定的。

**12-Factor推荐将应用的配置存储于*环境变量*中** (*env vars*, *env*) 。环境变量可以非常方便地在不同的部署间做修改，却不动一行代码；与配置文件不同，不小心把它们签入代码库的概率微乎其微；与一些传统的解决配置问题的机制（比如Java的属性配置文件）相比，环境变量与语言和系统无关。

配置管理的另一个方面是分组。有时应用会将配置按照特定部署进行分组（或叫做“环境”），例如Rails中的`development`、`test`和`production`环境。这种方法无法轻易扩展：更多部署意味着更多新的环境，例如`staging`或`qa`。随着项目的不断深入，开发人员可能还会添加他们自己的环境，比如`joes-staging`，这将导致各种配置组合的激增，从而给管理部署增加了很多不确定因素。

12-Factor应用中，环境变量的粒度要足够小，且相对独立。它们永远也不会组合成一个所谓的“环境”，而是独立存在于每个部署之中。当应用程序不断扩展，需要更多种类的部署时，这种配置管理方式能够做到平滑过渡。

[15]: http://www.springsource.org/
[16]: http://static.springsource.org/spring/docs/2.5.x/reference/beans.html


## IV. 后端服务
___把后端服务(*backing services*)当作附加资源___

*后端服务* 是指程序运行所需要的通过网络调用的各种服务，如数据库([MySQL][17]，[CouchDB][18])，消息/队列系统([RabbitMQ][19]，[Beanstalkd][20])，SMTP邮件发送服务([Postfix][21])，以及缓存系统([Memcached][22])。

类似数据库的后端服务，通常由部署应用程序的系统管理员一起管理。除了本地服务之外，应用程序有可能使用了第三方发布和管理的服务。示例包括SMTP(例如 [Postmark](http://postmarkapp.com/))，数据收集服务（例如 [New Relic](http://newrelic.com/) 或 [Loggly](http://www.loggly.com/)），数据存储服务（如[Amazon S3](http://http://aws.amazon.com/s3/)），以及使用API访问的服务(例如 [Twitter](http://dev.twitter.com/), [Google Maps](http://code.google.com/apis/maps/index.html), [Last.fm](http://www.last.fm/api))。

**12-Factor应用不会区别对待本地或第三方服务。** 对应用程序而言，两种都是附加资源，通过一个url或是其他存储在[配置](#config)中的服务定位/服务证书来获取数据。12-Factor应用的任意[部署](#codebase) ，都应该可以在不进行任何代码改动的情况下，将本地MySQL数据库换成第三方服务(例如 [Amazon RDS][23])。类似的，本地SMTP服务应该也可以和第三方SMTP服务(例如Postmark)互换。上述2个例子中，仅需修改配置中的资源地址。

每个不同的后端服务是一份*资源*。例如，一个MySQL数据库是一个资源，两个MySQL数据库(用来数据分区)就被当作是2个不同的资源。12-factor应用将这些数据库都视作*附加资源*，并且与这些附加资源保持松耦合。

![一种部署附加4个后端服务](/medias/20120509/attached-resources.png)

部署可以按需加载或卸载资源。例如，如果应用的数据库服务由于硬件问题出现异常，管理员可以从最近的备份中恢复一个数据库，卸载当前的数据库，然后加载新的数据库 -- 整个过程都不需要修改代码。

[17]: http://dev.mysql.com/
[18]: http://couchdb.apache.org/
[19]: http://www.rabbitmq.com/
[20]: http://kr.github.com/beanstalkd/
[21]: http://www.postfix.org/
[22]: http://memcached.org/
[23]: http://aws.amazon.com/rds/


## V. 构建，发布，运行
___严格分离构建和运行___

[基准代码](#codebase) 转化为一份部署(非开发环境)需要以下三个阶段：

* *构建阶段*是指将代码仓库转化为可执行包的过程。构建时会使用指定版本的代码，获取和打包[依赖项](#dependencies)，编译成二进制文件和资源文件。
* *发布阶段*会将构建的结果和当前部署所需[配置](#config)相结合，并能够立刻在运行环境中投入使用。
* *运行阶段*（或者说“运行时”）是指针对选定的发布版本，在执行环境中启动一系列应用程序[进程](#processes)。

![代码被构建，然后和配置结合成为发布版本](/medias/20120509/release.png)

**12-facfor应用严格区分构建、发布、运行这三个步骤。** 举例来说，直接修改处于运行状态的代码是非常不可取的做法，因为这些修改很难再同步回构建步骤。

部署工具通常都提供了发布管理工具，最引人注目的功能是退回至较旧的发布版本。比如，[Capistrano][24]将所有发布版本都存储在一个叫`releases`的子目录中，当前的在线版本只需映射至对应的目录即可。该工具的`rollback`命令可以很容易地实现回退版本的功能。

每一个发布版本必须对应一个唯一的发布ID，例如可以使用发布时的时间戳(`2011-04-06-20:32:17`)，亦或是一个增长的数字(`v100`) 。发布的版本就像一本只能追加的账本，一旦发布就不可修改，任何的变动都应该产生一个新的发布版本。

新的代码在部署之前，需要开发人员触发构建操作。但是，运行阶段不一定需要人为触发，而是可以自动进行。如服务器重启，或是进程管理器重启了一个崩溃的进程。因此，运行阶段应该保持尽可能少的模块，这样假设半夜发生系统故障而开发人员又捉襟见肘也不会引起太大问题。构建阶段是可以相对复杂一些的，因为错误信息能够立刻展示在开发人员面前，从而得到妥善处理。

[24]: https://github.com/capistrano/capistrano/wiki


## VI. 进程
___以一个或多个无状态进程运行应用___

运行环境中，应用程序通常是以一个和多个*进程*运行的。

最简单的场景中，代码是一个独立的脚本，运行环境是开发人员自己的笔记本电脑，进程由一条命令行(例如`python my_script.py`)。另外一个极端情况是，复杂的应用可能会使用很多[进程类型](#concurrency)，也就是零个或多个进程实例。

**12-factor应用的进程必须无状态且[无共享](http://en.wikipedia.org/wiki/Shared_nothing_architecture) 。** 任何需要持久化的数据都要存储在[后端服务](#backing-services)内，比如数据库。

内存区域或磁盘空间可以作为进程在做某种事务型操作时的缓存，例如下载一个很大的文件，对其操作并将结果写入数据库的过程。12-Factor应用根本不用考虑这些缓存的内容是不是可以保留给之后的请求来使用，这是因为应用启动了多种类型的进程，将来的请求多半会由其他进程来服务。即使在只有一个进程的情形下，先前保存的数据（内存或文件系统中）也会因为重启（如代码部署、配置更改、或运行环境将进程调度至另一个物理区域执行）而丢失。

源文件打包工具([Jammit][25], [django-assetpackager][26]) 使用文件系统来缓存编译过的源文件。12-factor应用更倾向于在[构建步骤](#build-release-run) 做此动作——正如[Rails资源管道][27]，而不是在运行阶段。

一些互联网系统依赖于[“粘性session”][28]，这是指将用户session中的数据缓存至某进程的内存中，并将同一用户的后续请求路由到同一个进程。粘性Session是twelve-factor极力反对的。Session中的数据应该保存在诸如[Memcached][29]或[Redis][30]这样的带有过期时间的缓存中。

[25]: http://documentcloud.github.com/jammit/
[26]: http://code.google.com/p/django-assetpackager/
[27]: http://ryanbigg.com/guides/asset_pipeline.html
[28]: http://en.wikipedia.org/wiki/Load_balancing_%28computing%29#Persistence
[29]: http://memcached.org/
[30]: http://redis.io/


## VII. 端口绑定
___通过端口绑定(*Port binding*)来提供服务___

互联网应用有时会运行于服务器的容器之中。例如PHP经常作为[Apache HTTPD][31]的一个模块来运行，正如Java运行于[Tomcat][32] 。

**12-factor应用完全自我加载**而不依赖于任何网络服务器就可以创建一个面向网络的服务。互联网应用**通过端口绑定来提供服务**，并监听发送至该端口的请求。

本地环境中，开发人员通过类似`http://localhost:5000/`的地址来访问服务。在线上环境中，请求统一发送至公共域名而后路由至绑定了端口的网络进程。

通常的实现思路是，将网络服务器类库通过[依赖声明](#dependencies) 载入应用。例如，Python的[Tornado][33]、Ruby的[Thin][34]、Java以及其他基于JVM语言的[Jetty][35]。完全由*用户端*，确切的说应该是应用的代码，发起请求。和运行环境约定好绑定的端口即可处理这些请求。

HTTP并不是唯一一个可以由端口绑定提供的服务。其实几乎所有服务器软件都可以通过进程绑定端口来等待请求。例如，使用[XMPP][36]的[ejabberd][37]，以及使用[Redis协议][38]的[Redis][39]。

还要指出的是，端口绑定这种方式也意味着一个应用可以成为另外一个应用的[后端服务](#backing-services)，调用方将服务方提供的相应URL当作资源存入[配置](#config) 以备将来调用。

[31]: http://httpd.apache.org/
[32]: http://tomcat.apache.org/
[33]: http://www.tornadoweb.org/
[34]: http://code.macournoyer.com/thin/
[35]: http://jetty.codehaus.org/jetty/
[36]: http://xmpp.org/
[37]: http://www.ejabberd.im/
[38]: http://redis.io/topics/protocol
[39]: http://redis.io/


## VIII. 并发
___通过进程模型进行扩展___

任何计算机程序，一旦启动，就会生成一个或多个进程。互联网应用采用多种进程运行方式。例如，PHP进程作为Apache的子进程存在，随请求按需启动。Java进程则采取了相反的方式，在程序启动之初JVM就提供了一个超级进程储备了大量的系统资源(CPU和内存)，并通过多线程实现内部的并发管理。上述2个例子中，进程是开发人员可以操作的最小单位。

![扩展表现为运行中的进程，工作多样性表现为进程类型。](/medias/20120509/process-types.png)

**在12-factor应用中，进程是一等公民。** 12-factor应用的进程主要借鉴于[unix进程模型][40]。开发人员可以运用这个模型去设计应用架构，将不同的工作分配给不同的*进程类型*。例如，HTTP请求可以交给web进程来处理，而常驻的后台工作则交由worker进程负责。

这并不表示应用不能通过单个进程来处理并发，如使用VM运行时的线程机制，或是由[EventMachine][41]、[Twisted][42]、[Node.js][43]等工具提供的异步/事件驱动模型。但是，单个VM的垂直扩展能力是有限的，所以应用必须能够扩展到多台物理机器上运行。

在需要对系统进行扩展时，进程模型的作用会大放异彩。[12-factor应用的进程所具备的无共享，水平分区的特性](#processes)意味着增加并发处理能力会是一项简单而稳妥的操作。这些进程的类型以及每个类型中进程的数量就被称作*进程构成* 。

12-factor应用 [不需要作为守护进程启动][44]或是写入PID文件。相反的，应该借助操作系统的进程管理器(例如[Upstart][45]，分布式的进程管理云平台，或在开发环境中使用类似[Foreman][46]的工具)，来管理[输出流](#logs)，应对进程崩溃，以及处理用户触发的重启和关闭操作。

[40]: http://adam.heroku.com/past/2011/5/9/applying_the_unix_process_model_to_web_apps/
[41]: http://rubyeventmachine.com/
[42]: http://twistedmatrix.com/trac/
[43]: http://nodejs.org/
[44]: http://dustin.github.com/2010/02/28/running-processes.html
[45]: http://upstart.ubuntu.com/
[46]: http://blog.daviddollar.org/2011/05/06/introducing-foreman.html


## IX. 易处理
___快速启动和优雅终止可最大化健壮性___

**12-factor应用的[进程](#processes)是*可支配*的，意思是说它们可以瞬间开启或停止。** 这有利于快速、弹性的伸缩应用，迅速部署变化的[代码](#codebase)或[配置](#config) ，稳健的部署应用。

进程应当追求**最小启动时间**。理想状态下，进程从敲下命令到真正启动并等待请求的时间应该只需很短的时间。更少的启动时间提供了更敏捷的[发布](#build-release-run) 以及扩展过程，此外还增加了健壮性，因为进程管理器可以在授权情形下容易的将进程搬到新的物理机器上。

进程**一旦接收[终止信号(`SIGTERM`)][47]就会优雅的终止**。就网络进程而言，优雅终止是指停止监听服务的端口，即拒绝所有新的请求，并继续执行当前已接收的请求，然后退出。此类型的进程所隐含的要求是HTTP请求大多都很短(不会超过几秒钟)，而在长时间轮询中，客户端在丢失连接后应该马上尝试重连。

对于worker进程来说，优雅终止是指将当前任务退回队列。例如，[RabbitMQ][48]中，worker可以发送一个[`NACK`][49]信号。[Beanstalkd][50]中，任务终止并退回队列会在worker断开时自动触发。有锁机制的系统诸如[Delayed Job][51]则需要确定释放了系统资源。此类型的进程所隐含的要求是，任务都应该 [可重复执行][52]，这主要由将结果包装进事务或是使重复操作[幂等][53]来实现。

进程还应当**在面对突然死亡时保持健壮**，例如底层硬件故障。虽然这种情况比起优雅终止来说少之又少，但终究有可能发生。一种推荐的方式是使用一个健壮的后端队列，例如[Beanstalkd][50]，它可以在客户端断开或超时后自动退回任务。无论如何，12-factor应用都应该可以设计能够应对意外的、不优雅的终结。[Crash-only design][54]将这种概念转化为[合乎逻辑的理论][55]。

[47]: http://en.wikipedia.org/wiki/SIGTERM
[48]: http://www.rabbitmq.com/
[49]: http://www.rabbitmq.com/amqp-0-9-1-quickref.html#basic.nack
[50]: http://kr.github.com/beanstalkd/
[51]: https://github.com/collectiveidea/delayed_job#readme
[52]: http://en.wikipedia.org/wiki/Reentrant_%28subroutine%29
[53]: http://en.wikipedia.org/wiki/Idempotence
[54]: http://lwn.net/Articles/191059/
[55]: http://couchdb.apache.org/docs/overview.html


## X. 开发环境与线上环境等价
___尽可能的保持开发，预发布，线上环境相同___

从以往经验来看，开发环境（即开发人员的本地[部署](#codebase)）和线上环境（外部用户访问的真实部署）之间存在着很多差异。这些差异表现在以下三个方面：

* **时间差异：** 开发人员正在编写的代码可能需要几天，几周，甚至几个月才会上线。
* **人员差异：** 开发人员编写代码，运维人员部署代码。
* **工具差异：** 开发人员或许使用Nginx，SQLite，OS X，而线上环境使用Apache，MySQL以及Linux。

**12-factor应用想要做到[持续部署][56]就必须缩小本地与线上差异。** 再回头看上面所描述的三个差异:

* 缩小时间差异：开发人员可以几小时，甚至几分钟就部署代码。
* 缩小人员差异：开发人员不只要编写代码，更应该密切参与部署过程以及代码在线上的表现。
* 缩小工具差异：尽量保证开发环境以及线上环境的一致性。

将上述总结变为一个表格如下：

<table>
  <tr>
    <th></th>
    <th>传统应用</th>
    <th>12-factor应用</th>
  </tr>
  <tr>
    <th>每次部署间隔</th>
    <td>数周</td>
    <td>几小时</td>
  </tr>
  <tr>
    <th>开发人员 vs 运维人员</th>
    <td>不同的人</td>
    <td>相同的人</td>
  </tr>
  <tr>
    <th>开发环境 vs 线上环境</th>
    <td>不同</td>
    <td>尽量接近</td>
  </tr>
</table>

[后端服务](#backing-services) 是保持开发与线上等价的重要部分，例如数据库，队列系统，以及缓存。许多语言都提供了简化获取后端服务的类库，例如不同类型服务的*适配器*。下列表格提供了一些例子。

<table>
  <tr>
    <th>类型</th>
    <th>语言</th>
    <th>类库</th>
    <th>适配器</th>
  </tr>
  <tr>
    <td>数据库</td>
    <td>Ruby/Rails</td>
    <td>ActiveRecord</td>
    <td>MySQL, PostgreSQL, SQLite</td>
  </tr>
  <tr>
    <td>队列</td>
    <td>Python/Django</td>
    <td>Celery</td>
    <td>RabbitMQ, Beanstalkd, Redis</td>
  </tr>
  <tr>
    <td>缓存</td>
    <td>Ruby/Rails</td>
    <td>ActiveSupport::Cache</td>
    <td>Memory, filesystem, Memcached</td>
  </tr>
</table>

开发人员有时会觉得在本地环境中使用轻量的后端服务具有很强的吸引力，而那些更重量级的健壮的后端服务应该使用在生产环境。例如，本地使用SQLite线上使用PostgreSQL；又如本地缓存在进程内存中而线上存入Memcached。

**12-factor应用的开发人员应该反对在不同环境间使用不同的后端服务**，即使适配器已经可以几乎消除使用上的差异。这是因为，不同的后端服务意味着会突然出现的不兼容，从而导致测试、预发布都正常的代码在线上出现问题。这些错误会给持续部署带来阻力。从应用程序的生命周期来看，消除这种阻力需要花费很大的代价。

与此同时，轻量的本地服务也不像以前那样引人注目。借助于[Homebrew][57]，[apt-get][58]等现代的打包系统，诸如Memcached、PostgreSQL、RabbitMQ等后端服务的安装与运行也并不复杂。此外，使用类似[Chef][59]和[Puppet][60]的声明式配置工具，结合像[Vagrant][61]这样轻量的虚拟环境就可以使得开发人员的本地环境与线上环境无限接近。与同步环境和持续部署所带来的益处相比，安装这些系统显然是值得的。

不同后端服务的适配器仍然是有用的，因为它们可以使移植后端服务变得简单。但应用的所有部署，这其中包括开发、预发布以及线上环境，都应该使用同一个后端服务的相同版本。

[56]: http://www.avc.com/a_vc/2011/02/continuous-deployment.html
[57]: http://mxcl.github.com/homebrew/
[58]: https://help.ubuntu.com/community/AptGet/Howto
[59]: http://www.opscode.com/chef/
[60]: http://docs.puppetlabs.com/
[61]: http://vagrantup.com/


## XI. 日志
___把日志当作事件流___

*日志*使得应用程序运行的动作变得透明。在基于服务器的环境中，日志通常被写在硬盘的一个文件里，但这只是一种输出格式。

日志应该是[事件流][62]的汇总，将所有运行中进程和后端服务的输出流按照时间顺序收集起来。日志的原始形式通常是文本文件，一行一个事件（程序异常产生的跟踪信息会跨越多行）。日志没有确定开始和结束，但随着应用在运行会持续的增加。

**12-factor应用本身从不考虑存储自己的输出流。** 不应该试图去写或者管理日志文件。相反，每一个运行的进程都会直接的标准输出(`stdout`)事件流。开发环境中，开发人员可以通过这些数据流，实时在终端看到应用的活动。

在预发布或线上部署中，每个进程的输出流由运行环境截获，并将其他输出流整理在一起，然后一并发送给一个或多个最终的处理程序，用于查看或是长期存档。这些存档路径对于应用来说不可见也不可配置，而是完全交给程序的运行环境管理。类似[Logplex][63]和[Fluent][64]的开源工具可以达到这个目的。

这些事件流可以输出至文件，或者在终端实时观察。最重要的，输出流可以发送到[Splunk][65]这样的日志索引及分析系统，或[Hadoop/Hive][66]这样的通用数据存储系统。这些系统为查看应用的历史活动提供了强大而灵活的功能，包括：

* 找出过去一段时间特殊的事件。
* 图形化一个大规模的趋势，比如每分钟的请求量。
* 根据用户定义的条件实时触发警报，比如每分钟的报错超过某个警戒线。

[62]: http://adam.heroku.com/past/2011/4/1/logs_are_streams_not_files/
[63]: https://github.com/heroku/logplex
[64]: https://github.com/fluent/fluentd
[65]: http://www.splunk.com/
[66]: http://hive.apache.org/


## XII. 管理进程
___后台管理任务当作一次性进程运行___

[进程构成](#concurrency) 是指用来处理应用的常规业务(比如处理web请求)的一组进程。与此不同，开发人员经常希望执行一些管理或维护应用的一次性任务，例如：

* 运行数据移植（Django中的`manage.py syncdb`, Rails中的`rake db:migrate`）。
* 运行一个控制台（也被称为[REPL][67] shell），来执行一些代码或是针对线上数据库做一些检查。大多数语言都通过解释器提供了一个REPL工具(`python`或`erl`) ，或是其他命令（Ruby使用 `irb`, Rails使用 `rails console` ）。
* 运行一些提交到代码仓库的一次性脚本。

一次性管理进程应该和正常的[常驻进程](#processes)使用同样的环境。这些管理进程和任何其他的进程一样使用相同的[代码](#codebase)和[配置](#config)，基于某个[发布版本](#build-release-run)运行。后台管理代码应该随其他应用程序代码一起发布，从而避免同步问题。

所有进程类型应该使用同样的[依赖隔离](#dependencies)技术。例如，如果Ruby的web进程使用了命令`bundle exec thin start`，那么数据库移植应使用`bundle exec rake db:migrate`。同样的，如果一个Python程序使用了Virtualenv，则需要在运行Tornado Web服务器和任何`manage.py`管理进程时引入‵bin/python‵ 。

12-factor尤其青睐那些提供了REPL shell的语言，因为那会让运行一次性脚本变得简单。在本地部署中，开发人员直接在命令行使用shell命令调用一次性管理进程。在线上部署中，开发人员依旧可以使用ssh或是运行环境提供的其他机制来运行这样的进程。

[67]: http://en.wikipedia.org/wiki/Read-eval-print_loop
