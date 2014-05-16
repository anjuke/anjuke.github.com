---
layout: post
title: "编写可维护的代码"
date: 2014-05-15 11:30
comments: true
categories:
---

{%img right https://avatars1.githubusercontent.com/u/863872?s=460 80 %}

文：[蕫菲](https://github.com/kdlan) kdlan

接着 [上次的内容](/blog/2014/01/10/v2-and-desigin-pattern/)

### 从最简单用户注册开始

产品需求，需要用户通过 form 提交邮箱和注册密码。

```php
<?php
class UserRegisterController {
    //...
}

class UserRegisterService {
    public function register($data) {
        $userInfo = ...
        $this->userDao.save($userInfo);
    }
}
```

过了两天产品过来说，我们需要向注册的用户发送欢迎邮件。

```php
<?php
class User_RegisterController {
    //...
}

class User_RegisterService {
    public function register($data) {
        $userInfo = ...
        $this->userDao.save($userInfo);
        $this->sendWelcomeEmail($userInfo);
    }

    private function sendWelcomeEmail($userInfo) {
        //...
    }
}
```

过了两天，产品又来，说我们要支持手机注册，手机注册的用户需要验证手机号，并且注册成功以后要发送成功的短信。邮件的注册流程保持不变。

```php
<?php
class User_RegisterService {
    public function register($data) {
        $userInfo = ...
        if ($userInfo->type == EMAIL_USER) {
            //do nothing
        } else if ($userInfo->type == MOBILE_USER) {
            $this->checkPhone($userInfo);
        }

        $this->userDao.save($userInfo);
        if ($userInfo->type == EMAIL_USER)  {
            $this->sendWelcomeEmail($userInfo);
        } else if ($userInfo->type == MOBILE_USER) {
            $this->sendWelcomeSMS($userInfo);
        }
    }

    private function sendWelcomeEmail($userInfo) {
        //...
    }

    private function sendWelcomeSMS($userInfo) {
        //...
    }

    private function checkPhone($userInfo) {
        //...
    }
}
```

过了两天产品说用户注册成功以后要发送站内信

```php
<?php
class User_RegisterService {
    public function register($data) {
        $userInfo = ...
        if ($userInfo->type == EMAIL_USER) {
            //do nothing
        } else if ($userInfo->type == MOBILE_USER) {
            $this->checkPhone($userInfo);
        }

        $this->userDao.save($userInfo);
        if ($userInfo->type == EMAIL_USER) {
            $this->sendWelcomeEmail($userInfo);
        } else if ($userInfo->type == MOBILE_USER) {
            $this->sendWelcomeSMS($userInfo);
        }

        $this->sendUserMessage($userInfo);
    }

    private function sendWelcomeEmail($userInfo) {
        //...
    }

    private function sendWelcomeSMS($userInfo) {
        //...
    }

    private function checkPhone($userInfo) {
        //...
    }

    private function sendUserMessage($userInfo) {
        //...
    }
}
```

又过了两天，产品说用户注册以后，原来匿名状态下的数据要同步到登陆以后的数据中（比如收藏订阅）。

```php
<?php
class User_RegisterService {
    public function register($data) {
        $userInfo = ...
        if ($userInfo->type == EMAIL_USER) {
            //do nothing
        } else if ($userInfo->type == MOBILE_USER) {
            $this->checkPhone($userInfo);
        }

        $this->userDao.save($userInfo);
        if ($userInfo->type == EMAIL_USER) {
            $this->sendWelcomeEmail($userInfo);
        } else if ($userInfo->type == MOBILE_USER) {
            $this->sendWelcomeSMS($userInfo);
        }

        $this->sendUserMessage($userInfo);
        $this->syncUserFavorateData($userInfo);

    }

    private function sendWelcomeEmail($userInfo) {
        //...
    }

    private function sendWelcomeSMS($userInfo) {
        //...
    }

    private function checkPhone($userInfo) {
        //...
    }

    private function sendUserMessage($userInfo) {
        //...
    }
}
```

又过两天，产品说我们要支持微博注册，QQ注册，豆瓣注册。

按照现在的代码，一定会越来越混乱，功能越来越多，性能越来越差，代码难以维护。

### 模板方法模式
*Template Method Pattern*

#### 什么是模板方法

模板方法模式定义了一个算法的步骤，并允许次类别为一个或多个步骤提供其实践方式。让次类别在不改变算法架构的情况下，重新定义算法中的某些步骤。

#### 用法

模板方法模式多用在：

- 某些类别的算法中，实做了相同的方法，造成程式码的重复。
- 控制子类别必须遵守的一些事项。

简单来说就是在父类里面定义好业务逻辑的处理流程，将可变化的部分提取，变为抽象类。具体业务继承父类，实现父类的抽象方法来提供不同的行为。

#### 用模板方法模式重构我们的代码

在完成了邮件注册，手机注册，注册后发送站内信的几个功能后，我们的代码是这样的，

```php
<?php
class User_RegisterService {
    public function register($data){
        $userInfo = ...
        if ($userInfo->type == EMAIL_USER) {
            //do nothing
        } else if ($userInfo->type == MOBILE_USER) {
            $this->checkPhone($userInfo);
        }

        $this->userDao.save($userInfo);
        if($userInfo->type == EMAIL_USER) {
            $this->sendWelcomeEmail($userInfo);
        } else if ($userInfo->type == MOBILE_USER) {
            $this->sendWelcomeSMS($userInfo);
        }

        $this->sendUserMessage($userInfo);
    }

    private function sendWelcomeEmail($userInfo) {
        //...
    }

    private function sendWelcomeSMS($userInfo) {
        //...
    }

    private function checkPhone($userInfo) {
        //...
    }

    private function sendUserMessage($userInfo) {
        //...
    }
}
```

当产品提出要支持微博、QQ 注册的时候，我们可以预测到，以后还有类似的第三方账号注册，比如豆瓣、百度… 这时我们可以这么写，

```php
<?php
abstract class User_AbstractRegisterService {
    public function register($data) {
        $userInfo = ...
        $this->checkUserInfo($userInfo);
        $this->userDao.save($userInfo);
        $this->afterUserRegister($userInfo);
    }

    abstract function checkUserInfo($userInfo);

    abstract function afterUserRegister($userInfo);
}

class User_EmailRegisterService extends User_AbstractRegisterService {
    function checkUserInfo($userInfo){
        //donothing
    }

    function afterUserRegister($userInfo){
        //send welcome email
    }
}

class User_MobileRegisterService extends User_AbstractRegisterService {
    function checkUserInfo($userInfo){
        //check phone number
    }

    function afterUserRegister($userInfo) {
        //send welcome sms
    }
}

class User_WeiboRegisterService extends User_AbstractRegisterService {
    function checkUserInfo($userInfo) {
        //get user info from Weibo api
    }

    function afterUserRegister($userInfo) {
        //send welcome weibo
    }
}

class User_RegisterController {
    function handleRequest() {
        if (registerType == Email_USER) {
            $service=new User_EmailRegisterService();
        } else if (registerType == Mobile_USER) {
            $service=new User_MobileRegisterService();
        } else if (registerType == WEIBO_USER) {
            $service=new User_WeiboRegisterService();
        }

        $service->register($data);
    }
}
```

之后产品提出要新增微博注册，新增 QQ 注册，只需要新增一个子类，在Controller里面增加一个 else if 即可。

对于原来匿名状态下的数据要同步到登陆以后的数据中这样的需求，也只需要在父类中修改来进行统一处理即可。

```php
<?php
abstract class User_AbstractRegisterService {
    public function register($data) {
        $userInfo = ...
        $this->checkUserInfo($userInfo);
        $this->userDao.save($userInfo);
        $this->afterUserRegister($userInfo);
        $this->sendUserMessage($userInfo);
        $this->syncUserData($userInfo);
    }

    private function sendUserMessage($userInfo) {
        //...
    }

    private function syncUserData($userInfo) {
        //...
    }

    abstract function checkUserInfo($userInfo);

    abstract function afterUserRegister($userInfo);
}
```

对比第一个版本代码

- 少一次 if 判断，只需要在 Controller 里做一次处理即可，原来在 `saveUser` 之前和之后都要进行判断，减少了出错的可能。
- 各个子业务的代码分割在不同的子类中，`User_RegisterService` 不需要再承载过多的职责。缺点是类的数量变多了。
- 每新增一个用户注册后操作的需求（比如同步数据，比如发送api监控实时的用户注册量，以及发送站内消息），我们都需要去修改 `User_AbstractRegisterService` 来增加一部分逻辑。
- 整个处理流程里的做事情并没有变化，性能仍然是问题。

#### 需求又来了

TeamLeader 过来跟我们说单个页面性能不能超过 200 ms。

加了断点以后，发现同步数据这块的速度很慢，于是决定把这块内容发送消息出去，后台起个 job 来单独做数据同步。

```php
<?php
abstract class User_AbstractRegisterService {
    public function register($data){
        $userInfo = ...
        $this->checkUserInfo($userInfo);
        $this->userDao.save($userInfo);
        $this->afterUserRegister($userInfo);
        $this->sendUserMessage($userInfo);
        $this->sendUserRegisterMQ($userInfo);
    }

    public function sendUserMessage($userInfo) {
        //...
    }

    private function sendUserRegisterMQ($userInfo) {
    }

    abstract function checkUserInfo($userInfo);

    abstract function afterUserRegister($userInfo);
}

class User_RegisterSyncDataJob {
    public function run() {
        $data=getUserRegisterMessageFromMQ();
        $this->syncUserData($data);
    }

    private function syncUserData($userInfo){
        //...
    }
}
```

`syncUserData` 因为是在原来的抽象类中实现的，不太好复用。

虽然性能问题解决了，但是如果继续有需求，比如需要把调用 Weibo API 发送微博或者手机发送欢迎短信的操作也放在 job 中异步执行，这段代码仍然不太好修改。

### 继续改进

#### 重新梳理业务

把节点往回推到完成了邮件注册，手机注册，注册后发送站内信的几个功能后。

```php
<?php
class User_RegisterService {
    public function register($data) {
        $userInfo = ...
        if ($userInfo->type == EMAIL_USER) {
            //do nothing
        } else if ($userInfo->type == MOBILE_USER){
            $this->checkPhone($userInfo);
        }

        $this->userDao.save($userInfo);
        if ($userInfo->type == EMAIL_USER) {
            $this->sendWelcomeEmail($userInfo);
        } else if ($userInfo->type == MOBILE_USER) {
            $this->sendWelcomeSMS($userInfo);
        }

        $this->sendUserMessage($userInfo);
    }

    private function sendWelcomeEmail($userInfo) {
        //...
    }

    private function sendWelcomeSMS($userInfo) {
        //...
    }

    private function checkPhone($userInfo) {
        //...
    }

    private function sendUserMessage($userInfo) {
        //...
    }
}
```

这是，产品给我们提了支持微博注册的需求，我们就可以总结出来，用户注册业务的流程

- 对注册数据进行事前检查（短信验证码）
- 获取用户数据（表单、Weibo API）
- 保存注册数据
- 注册后特殊处理（发送欢迎邮件、欢迎短信）
- 注册后通用处理（发送站内信）
- 注册后特殊处理和通用处理本身其实没有本质上的区别，只是特殊处理需要去判断用户的类型来确定应该做哪些事情，而通用处理则不用去管这些

#### 重新设计

按照这样的流程，我们可以重新来设计 `User_RegisterService`的代码。

```php
<?php
class User_RegisterService {
    public function register($data) {
        $this->beforeUserRegister($data);
        $userInfo = $this->getUserInfo($data);
        $this->userDao.save($userInfo);
        $this->afterUserRegisterWithType($userInfo);
        $this->afterUserRegisterWithAll($userInfo);
    }

    private function beforeUserRegister($data) {
        //TODO
    }

    private function getUserInfo($data) {
        //TODO
    }

     private function afterUserRegisterWithType($userInfo) {
        //TODO
    }

    private function getAllTypeAfterRegisterHandler($userInfo) {
        //TODO
    }

}
```

我们有注册前检查和注册后处理两类处理逻辑，所以这里我们定义三个接口，分别用来处理 `beforeUserRegister`，`getUserInfo`，`afterUserRegister`。

但是考虑到每个类型的用户，对应的 `beforeUserRegister`，`getUserInfo`，`afterUserRegister` 之间都有很强的相关性，因此我们的接口这样来设计，

```php
<?php
interface AfterUserRegisterHandler
    public function afterUserRegister($userInfo);
}

interface UserRegisterHandler extends AfterUserRegisterHandler {
    public function beforeUserRegister($data);
    public function getUserInfo($data);
}
```

数据同步、发送站内信的处理可以单独实现 `AfterUserRegisterHandler` 接口，邮箱注册、微博注册的逻辑，就需要实现 `UserRegisterHandler` 接口。

接口定义好以后我们来实现 `User_RegisterService` 里的各个方法。

```php
<?php
class User_RegisterService {
     public function register($data) {
        $this->beforeUserRegister($data);
        $userInfo = $this->getUserInfo($data);
        $this->userDao.save($userInfo);
        $this->afterUserRegisterWithType($userInfo);
        $this->afterUserRegisterWithAll($userInfo);
    }

    private function beforeUserRegister($data) {
        $handler = getRegisterHandlerByType($data->type);
        return $handler->beforeUserRegister($data);
    }

    private function getUserInfo($data) {
        $handler = getRegisterHandlerByType($data->type);
        return $handler->getUserInfo($data);
    }

    private function afterUserRegisterWithType($userInfo) {
        $handler = getAfterRegisterHandlerByType($data->type);
        return $handler->afterUserRegister($data);
    }

    private function afterUserRegisterWithAll($userInfo) {
        $allHandler = getAllTypeRegisterHandler();
        forearch ($allHandler as $handler) {
            $handler->afterUserRegister($data);
        }
    }
    private function getRegisterHandlerByType($type) {
        //TODO
    }

    private function getAfterRegisterHandlerByType($type) {
        //TODO
    }

    private function getAllTypeAfterRegisterHandler() {
        //TODO
    }
}
```

`getRegisterHandlerByType` 的几个方法里，简单的可以直接 if else 返回对应的 handler，也可以通过映射的方式来实现。

```php
<?php
    // ...
    private $registerHandlerMapping = array(
        EMAIL_USER => new EmailUserRegisterHandler(),
        MOBILE_USER => new MobileUserRegisterHandler(),
        WEIBO_USER => new MobileUserRegisterHandler(),
        //...
    );

    private $allTypeAfterRegisterHandler = array(
        new SendUserMessageAfterRegisterHandler(),
        //new SyncAnonymousUserDataAfterRegisterHandler(),
        //...
    );

    private function getRegisterHandlerByType($type) {
        return $registerHandlerMapping[$type];
    }

    private function getAfterRegisterHandlerByType($type) {
        return getRegisterHandlerByType($type);
    }

    private function getAllTypeAfterRegisterHandler() {
        return $allTypeAfterRegisterHandler;
    }
```

下一次，当我们的产品又提出了新需求以后，只要产品没有重新定义用户注册的流程，我们就只需要去实现 `UserRegisterHandler` 或者 `AfterUserRegisterHandler`，添加到 `registerHandlerMapping` 或 `allTypeAfterRegisterHandler` 里面，就可以支持新的功能，`User_RegisterService` 其他地方的代码都不用修改。

#### 发送消息

为了提高性能，降低耦合度，需要再把用户同步的操作分离，发送消息到 MQ，单独起 job 来处理用户数据的同步。

```php
<?php
     private $registerHandlerMapping = array (
        EMAIL_USER => new EmailUserRegisterHandler(),
        MOBILE_USER => new MobileUserRegisterHandler(),
        WEIBO_USER => new MobileUserRegisterHandler(),
        //...
     );

    private $allTypeAfterRegisterHandler = array(
        new SendUserMessageAfterRegisterHandler(),
        //new SyncAnonymousUserDataAfterRegisterHandler(),
        new SendUserRegisterMQAfterRegisterHandler(),
        //...
    );

     private function getRegisterHandlerByType($type) {
        return $registerHandlerMapping[$type];
     }

     private function getAfterRegisterHandlerByType($type) {
        return getRegisterHandlerByType($type);
     }

     private function getAllTypeAfterRegisterHandler() {
        return $allTypeAfterRegisterHandler;
     }
```

```php
<?php
class AfterUserRegisterJob {
    public function run(){
        $handlerClass = getHandlerClassFromArgv();
        $data = getUserRegisterMessageFromMQ();
        $this->process($handlerClass, $data);
    }

    private function process($handlerClass, $data) {
        $handler = new $handlerClass();
        $handler->afterUserRegister($data);
    }
}
```

Usage：

    $ php laucher.php AfterUserRegisterJob SyncAnonymousUserDataAfterRegisterHandler

因为之前已经把后续处理的逻辑都分装在 `AfterRegisterHandler` 里面，`job` 这里调用的时候只需要写一些胶水代码，读取数据，调用具体的实现类即可。

之前 `User_RegisterService` 里的逻辑代码也不用修改，只需要修改 `allTypeAfterRegisterHandler` 这一处，就能做到发送消息到 MQ，启动 job 处理消息了。

下一次，当我们要把所有的用户注册后的行为全部搬迁到 job 的时候，只需要这么修改，

```php
<?php
     private $registerHandlerMapping = array(
        EMAIL_USER => new EmailUserRegisterHandler(),
        MOBILE_USER => new MobileUserRegisterHandler(),
        WEIBO_USER => new MobileUserRegisterHandler(),
        //...
     );

    private $allTypeAfterRegisterHandler = array (
        //new SendUserMessageAfterRegisterHandler(),
        //new SyncAnonymousUserDataAfterRegisterHandler(),
        new SendUserRegisterMQAfterRegisterHandler(),
    );

    private function getRegisterHandlerByType($type) {
        return $registerHandlerMapping[$type];
    }

    private function getAfterRegisterHandlerByType($type) {
        return new DoNothingAfterRegisterHandler();
    }

    private function getAllTypeAfterRegisterHandler() {
        return $allTypeAfterRegisterHandler;
    }
```

然后再启动对应的发送微博等任务的 job，就可以非常容易的做到这些需求了。

### 软件设计原则

前面我们看到了一个小业务模块的 3 种写法

- 第一种写法，每次来新需求，我们都需要对代码的多处来进行修改。
- 第二种写法做了一定程度的抽象，但是当有一些之前没有考虑到的需求（例如拆分处理，通过消息队列传输数据），仍然会有多处的代码修改点。
- 而第三种写法，只要产品对业务流程本身没有大的调整，新需求到来的时候，我们只需要编写新需求的处理类，通过修改或者新增胶水代码，就可以完成需求，对原来代码的修改点降到最低。

[上次](/blog/2014/01/10/v2-and-desigin-pattern/) 我们提到了一些软件设计的原则，我们来看看第三种的写法是如何做到只需要一两个修改点，就能完成新需求的。

#### Single Responsibility Principle (SRP) – 职责单一原则

简单来说，就是一段代码只做一件事情。

`User_RegisterService` 的 `register` 方法只负责定义了用户注册的流程，所有的具体业务逻辑处理全部交给了其他的对象来负责。

`UserRegisterHandler` 的实现类只负责某个具体注册类型的处理。

`AfterUserRegisterHandler` 的实现类，只负责用户注册后需要进行的处理。

#### Program to an interface, not an implementation - 针对接口编程，而不是实现

`User_RegisterService` 的 `register` 方法里所有操作的业务逻辑对象，全部都是针对 `UserRegisterHandler` 和 `AfterUserRegisterHandler` 这两个接口，`register` 方法并不关心具体操作的是哪个实现类，这样把 `register` 方法和某个特定实现的逻辑类（比如微博，手机）隔离开，做到解耦。

这样当我们把同步数据，发送微博这些处理独立到 job 中去以后，我们就只用实现一个简单的 job，负责读取数据调用实现类就行了，原来的代码都不用做修改。

#### Open/Closed Principle (OCP) – 开闭原则

对扩展开放，对修改关闭。

`User_RegisterService` 的 `register` 方法里通过定义流程，操作接口，来确定了核心的处理流程，通过替换不同的 `UserRegisterHandler`，`AfterUserRegisterHandler` 实现类，实现了对 `register` 的扩展。我们可以给 `register` 方法提供不同的实现类，来完成我们特定的业务逻辑。

#### 控制代码，业务代码，组装代码

`User_RegisterService` 的 `register` 方法就是控制代码，它负责定义我们的一段处理流程。

`UserRegisterHandler` 和 `AfterUserRegisterHandler` 的实现类，就是业务代码，定义了我们某个特殊的业务逻辑的实现。

`getRegisterHandlerByType`，`getAfterRegisterHandlerByType`，`getAllTypeAfterRegisterHandler` 几个方法就是组装代码，用来把控制代码和业务代码组合起来，来完成我们的业务。

### 如何操作

#### 不要一开始就设计

在第二个例子和第三个例子，都选取了在完成了邮件注册，手机注册，注册后发送站内信的几个功能后，产品提出新的需求，这个时间节点上来进行重构。

原因就是在这个节点之前，我们的功能其实是相对比较简单的，没必要做那么多设计，快速时间功能就行了。

这个节点之后，我们的业务需求就已经比较清晰，能够比较清楚的预测到后面一段时间的需求，因此选取了这样一个时间点来做。

#### 梳理需求，定义流程

重新设计开始之前，我们要对需求进行分析，定义出我们业务处理的流程，比如前面的列子，梳理出来的流程就是

- 邮箱注册
  - 获取表单数据
  - 保存用户数据
  - 发送欢迎邮件
  - 发送站内信
- 手机注册
  - 获取表单数据
  - 手机验证码
  - 保存数据
  - 发送欢迎短信
  - 发送站内信
- 微博注册
  - 调用API获取用户数据
  - 保存数据
  - 发送微博
  - 发送站内信

再将这几个流程合并，我们就能抽象出这个业务的基本流程

- 对注册数据进行事前检查（短信验证码）
- 获取用户数据（表单、Weibo API）
- 保存注册数据
- 注册后特殊处理（发送欢迎邮件、欢迎短信）
- 注册后通用处理（发送站内信）

我们只需要按照这个流程来编写我们核心的流程代码即可。

#### 设计接口，实现代码

流程出来以后，我们就可以针对每个流程来设计具体的业务接口。遵循单一职责的原则，接口的设计尽量保持只做一件事情。

### 如何学习

面向对象和设计模式都只是工具，不能帮助我们解决所有的问题，而且很多模式都是有试用场景的，生搬硬套只会让代码更加难以维护，可以参考 [设计模式有何不妥，所谓的荼毒体现在哪](http://www.zhihu.com/question/23757237)。只有平常多写，多看，多思考才能不断的提升我们写代码的技巧。
