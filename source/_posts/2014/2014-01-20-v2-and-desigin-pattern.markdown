---
layout: post
title: "V2 中的软件设计"
date: 2014-05-13 11:30
comments: true
categories:
---

[{%img right https://avatars1.githubusercontent.com/u/863872?s=460 80 %}](https://github.com/kdlan)

这里介绍一下 [system](http://git.corp.anjuke.com/corp/v2-system) 和 [system-ext](http://git.corp.anjuke.com/site/system-ext) 里的一些设计原则。

本文需要对常见的设计模式有一个基本的了解，如果你对设计模式还一无所知，推荐阅读 [*Head First Design Pattern*](http://www.amazon.cn/Head-First%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F-%E5%BC%97%E9%87%8C%E6%9B%BC/dp/B0011FBU34)

**免责声明！！下面的内容都是我个人的总结，难免有错误的地方。主要是希望能抛砖引玉，大家能有自己的思考就好了**

## 1. 软件设计的一些原则

参考

* [一些软件设计的原则](http://coolshell.cn/articles/4535.html)
* [软件实体的设计原则](http://www.cnblogs.com/ldcsaa/archive/2012/02/26/2368959.html)
* [Head First Design Pattern](http://www.amazon.cn/Head-First%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F-%E5%BC%97%E9%87%8C%E6%9B%BC/dp/B0011FBU34)

这里偷个懒，给了一堆链接先参考一下，只提一些比较常见而且容易做到的原则。

### 1.1 职责单一原则
**Single Responsibility Principle (SRP)**

单一职责是说的一个类，一个方法只做一件事情。比如有用户注册，用户注册以后需要发邮件这样两个动作。那么遵守单一职责，需要将注册的逻辑放到一个类里，发送邮件的逻辑应该在另外一个类似，而不是合在同一段代码之中。

通过单一职责原则，可以比较容易的做到代码的高内聚低耦合。

### 1.2 开闭原则
**Open/Closed Principle (OCP)**

即对扩展开放，对修改关闭。V2 的核心代码对于这个原则有非常好的体现。比如我们要增加 URL 的处理逻辑，只需要写对应的 Controller 和 router 配置就行了，无需对 V2 进行修改。另外如果我们需要在 Router/Request/Response 对象上添加额外的行为，也只需要修改 V2 的配置，对于 V2 本身的代码是不需要修改的。

这样做的好处在于，对于核心的代码（对 V2 来说就是 APF），无论业务/需求如何变化，基本上不需要去修改这段代码（修 bug 除外），这样也可以比较容易的做到代码的高内聚。

### 1.3 最少知识原则
**Principle of Least Knowledge**

简单来说就是调用方对于被调用放无需知道太多的内容，只用了解最基本的 API 就行了。比如这段代码，

```java
final String outputDir = ctxt.getOptions()
        .getScratchDir()
        .getAbsolutePath();
```

这段代码本身作为调用方，需要了解 ctxt.getOptions() 返回了什么内容，还要知道 options.getScratchDir() 返回什么内容，更进一步才能拿到想要的 absolutePath。

如果这样写

```java
final String outputDir = ctxt.getScratchAbsolutePath()
```

相对来说就不太需要关心 ctxt 的里面到底是些什么东西。

同样的，最少知识原则也是在提高代码的内聚上有帮助的。

### 1.4 针对接口编程，而不是实现
**Program to an interface, not an implementation**

注意这里的接口是 interface 而不是 API。简单来说就是使用类或者对象的时候，使用其接口，而不是具体的实现类。

比如 V2 的 APF 里，对于各个 Controller 都是使用的 `APF_Controller` 这个接口，调用它的 `handler_request` 方法，而不是直接使用我们自己定义的那些 Controller。

比如 CGI，也是一种接口，Apache、Nginx 调用 CGI 的模块，按照 CGI 的协议向后端发送请求。至于后面是 C++、PHP、Python、Java 都是一样的使用。违反这个原则的做法就是直接用 apache/nginx 去调用我们业务的 C++、PHP、Python、Java 代码。

## 2. V2 的设计分析

### 2.1 [system](http://git.corp.anjuke.com/corp/v2-system)

我们从 [APF.php](http://git.corp.anjuke.com/corp/v2-system/browse/master/classes/APF.php) 开始，入口是 `APF::run()`

```php
<?php
public function run() {
    $this->prepare();
    if (!$this->dispatch()) {
        echo "Error";
    }
}

public function prepare(){
    apf_require_class($this->request_class);
    apf_require_class($this->response_class);

    $this->request = new $this->request_class();
    $this->response = new $this->response_class();

    apf_require_class($this->router_class);
    $router = new $this->router_class();
    $this->router = $router;

    return true;
}
```

`prepare()` 这段代码里实际体现到了如何做到 OCP 和面向接口的原则，通过参数定义的 request_class、response_class、router_class 来使用具体的实现类。这样只要我们自己定义的 Request、Response、Router 实现了具体的抽象接口，就可以来扩展我们自己的行为，而不用去修改 V2 本身的代码了。

接下来是核心的 `dispatch` 方法

```php
<?php
public function dispatch() {
    $class = $this->router->mapping();
    $controller = $this->get_controller($class);
    if (!$controller) {
        return false;
    }
    $this->current_controller = $controller;

    $interceptores = @$this->get_config($class, "interceptor");

    if ($interceptores) {
        $interceptor_classes = $this->get_interceptor_classes($class);
    } else {
        $basic_class = $controller->get_interceptor_index_name();
        $interceptor_classes = $this->get_interceptor_classes($basic_class);
    }

    $step = APF_Interceptor::STEP_CONTINUE;
    foreach ($interceptor_classes as $interceptor_class) {
        $interceptor = $this->load_interceptor($interceptor_class);
        if (!$interceptor) {
            continue;
        }
        $interceptors[] = $interceptor;
        $this->debug("interceptor::before(): " . get_class($interceptor));
        $step = $interceptor->before();
        if ($step != APF_Interceptor::STEP_CONTINUE) {
            break;
        }
    }

    if (!$this->is_debug_enabled()) {
        unset($this->debug_config);
        $this->trace_config = false;
    }

    if ($step != APF_Interceptor::STEP_EXIT) {
        while (true) {
            $result = $controller->handle_request();
            if ($result instanceof APF_Controller) {
                $controller = $result;
                continue;
            }
            break;
        }
        if (is_string($result)) {
            if (class_exists('APF_DB_Factory', false)) {
                APF_DB_Factory::get_instance()->close_pdo_all();
            }
            $this->page($result);
        }
    }

    $step = APF_Interceptor::STEP_CONTINUE;
    if (isset($interceptors)) {
        $interceptors = array_reverse($interceptors);
        foreach ($interceptors as $interceptor) {
            $step = $interceptor->after();
            $this->debug("interceptor::after(): " . get_class($interceptor));
            if ($step != APF_Interceptor::STEP_CONTINUE) {
                break;
            }
        }
    }

    return true;
}
```

这段代码比较长但是其实逻辑相对来说还是很清晰的

* 调用 router->mapping() 获得 controller 对象
* 获取该 controller 上的 interceptor 对象
* 调用 interceptor->before()
* 调用 controller->handle_request()
* 调用 this->page() 处理 Page 逻辑
* 调用 interceptor->after()

很明显，这里每一个对象都是通过接口的方式来使用的，这里的有 OCP 和面向接口两个原则体现。而为了实现这些原则，则是通过 router.php 和 interceptor.php 的配置文件来实现的

基本上 `APF->run()` 就是APF最核心的功能了，其余的都是 APF 提供的一些辅助方法（其实这些辅助方法应该放到别的地方去，因为违背了 **单一职责** 原则）。这里 APF 定义了最核心的处理逻辑，而一些具体的逻辑比如 URL mapping，比如具体每个 URL 的 handler_request 都交给了其他类来实现。同样 Router 只负责 URL mapping 一件事情，Controller 只负责 handle_request，Interceptor 只负责拦截 Controller 的处理，这里单一职责原则就体现出来了。~~如果你和我一样对代码有洁癖就请无视APF的那些辅助方法吧~~

再来看看 Interceptor ，这里其实有一点点 [事件驱动](https://zh.wikipedia.org/wiki/%E4%BA%8B%E4%BB%B6%E9%A9%85%E5%8B%95) 的味道。APF 定义了两个 Controller 的事件，*before handle_request* 和 *after handle_request* 两个事件，Interceptor 只不过是响应者两个事件的事件监听器而已。好处？最简单的，降低耦合度，提高代码重用。比如记录 Controller 的处理时间，如果没有 Interceptor，只能在每个 Controller 里加上时间记录的代码，如果要在每个 Controller 上都记录处理时间，则只能每个都增加代码，显然又违背了 **单一职责** 原则，而且有太多的重复代码，如果在 run 的代码里来做这件事情，又违背了 **OCP** 原则。

Router，在这里主要是负责做 `URL => controller` 的映射。直觉上，如果我们自己写一段这样的映射代码，可能就是大段的 if...else 来把各个 URL 分配到对应的 Controller 上（我真的见过几千行的 if...else 就是为了分发请求到对应的 Controller 上的）。大段的 if...else 首先来说很难看，而且很难维护，每次增加一段处理都需要修改 Router 的代码，显然是违背了 *OCP* 原则。所以 Router 这里的实现是通过一个 router.php 的配置定义 `正则 => Controller` 的映射。在 Router 里来处理这一堆的配置规则。这样，至少我们在增加 Controller 的时候，不用再去修改 Router 的代码了。

APF 其他的处理基本上也是按照 run() 里的方式来组织的（Page 和 Component 就这么被跳过了么...）。

### 2.2 [system-ext](http://git.corp.anjuke.com/site/system-ext)

抛开 DAO，关于 system-ext 相比 system 最大的不同在哪？我个人觉得是 [*Apf_Proxy_Proxy.php*](http://git.corp.anjuke.com/site/system-ext/browse/master/classes/apf/proxy/Proxy.php) 这个东西。这个类就是代理模式最直接的实现，当然我们也可以拿它来做装饰器模式的事情，只不过配置写起来麻烦点。

代理模式有什么用？如果你知道 AOP(Aspect Oriented Programming) 是个什么东西，如果里用过 Java，用过 SpringFramework，知道声明式事务是怎么回事，就知道 **代理** 这玩意有多么好用了

想想原来，我们要在一段代码上增加缓存是怎么做的？

```php
<?php
$data = $cache->get($key);
if (!$data){
	  $data = getFromDB();
	  $cache->put($key,$data,$time);
}
return $data;
```

每次增加一段缓存，我们都要写类似上面这样的代码。还有我们性能监控加的断点。

```php
<?php
Anjuke_Performance::get_instance()->benchmark_begin("formatPropData");
$prop = Props_Data::formatPropData($propInfo);
Anjuke_Performance::get_instance()->benchmark_end("formatPropData");
```

每个次加断点我们都要在代码里到处加上这种代码（真没人觉得难看么...）。好了，现在我们有了 proxy，要加个缓存怎么弄？完全不用修改原来的代码，只需要加一行配置。

```php
<?php
// servicecache.php
$config['allow_methodes'] = array(
    'Anjuke_Core_Service_DemoService::getCurrentTime' => array(
        'cachetime' => 10
    )
);
```

配置里定义的类上的方法就会自动的添加 cache 了，不用再写任何额外的代码。这样做的好处是，业务代码只需要专心处理业务就好了，缓存相关的逻辑是在其他的地方，所以可以比较容易的满足 SRP 原则。我们加缓存或者去掉缓存就只需要修改配置，而不用再去改我们的代码，所以 OCP 也满足了。

类似的，我们需要记录方法处理的时间，也只需要写一个对应的 Proxy, 然后在套在上面就可以处理。不用再满地的去些 benchmark_begin 和 benchmark_end 的代码了。需要去掉也很方便。

## 3. 一点点私货
主要是对代码的一点分类，完全是我个人总结，也有可能通篇都是错的，所以大家看看就好

个人觉得一个项目里的代码可以分为三种

* 控制代码
* 业务代码
* 组装代码

业务代码就是处理具体业务的代码；控制代码就是定义一段处理流程，将多个业务代码的逻辑组合起来；组装代码就是根据配置把业务代码和控制代码组合在一起的代码，比如各种 Factory/IoC 容器（Spring）。

我认为，好的代码设计，这三者的界限是非常清晰的。比如 APF，前面说的 `prepare()` 就是一段组装代码。run() 里面就是控制代码，定义了一套流程，业务代码就是 Router、Controller、Interceptor（Page 又被忽略了...） 。APF 里通过 `load_controller()`、`load_interceptor()` 的组装代码来组合业务和控制代码。通过这三者的结合一起来完成了一次请求的处理。

再进一步，实际上一段业务代码内部，也可以分为控制代码、业务代码、组装代码三个部分，这样反复的嵌套，就构成了我们整个项目。

做到这三部分代码的分离，可以比较清晰的写出比较容易维护，可扩展性高的代码。

不过绝大多数情况下，我们其实不需要一开始就分离的这么细，以免前期设计的时间过长，但是最后实际写代码的时候还是发现不能满足需求。也不需要把每一块代码都这么来分离，那样写得要累死了。

所以重构就在这里派上用场了。这里再推荐一本书，[重构:改善既有代码的设计](http://www.amazon.cn/%E9%87%8D%E6%9E%84-%E6%94%B9%E5%96%84%E6%97%A2%E6%9C%89%E4%BB%A3%E7%A0%81%E7%9A%84%E8%AE%BE%E8%AE%A1-%E7%A6%8F%E5%8B%92/dp/B003BY6PLK/)

后面我会再写一篇文档，结合我们实际的业务开发的例子，讲一下怎么来写比较容易维护，可扩展性高的代码。
