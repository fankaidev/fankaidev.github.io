---
layout: single
title:  "使用Maven Enforcer来约束DDD代码结构"
---

在领域驱动设计DDD中，非常重要的一点是让代码尽量能够体现领域逻辑或者说是业务设计。甚至有人希望让代码清晰到非开发的人员也能通过“阅读”代码来了解逻辑。虽然我觉得这一点过于理想化了，但是无疑我们需要保证代码有一个清晰的结构。

## 代码结构
对于DDD里代码分层架构的讨论已经非常多了，经典的比如六边形架构或者干净架构，都需要把领域层放在核心的位置，将其作为代码中最稳定的一部分，不依赖外层的内容。一个常见的代码结构可以参考[这篇文章](https://zhuanlan.zhihu.com/p/30877742)，这里引用一下文中Martin Fowler提炼出来的一个代码结构大概长这样。

<img src="https://pic4.zhimg.com/80/v2-5d36d7d11d02b8a3df2ccc7d96c3ee33_hd.jpg">

这样一个代码结构中，一般会有以下一些模块
```
- src/../server/
    domain
    service
    repository
    gateway
    infra
```

我们需要保证，其中的`domain`包里的内容不应该依赖任何其它对象（除了一些通用的基础设施）。而`gateway`和`infra`对接外部系统，不应该被任何其它的包所依赖。要使得`repository`不依赖`infra`，肯定需要用到依赖倒置。然而如果存储的接口和实现都放在一起，那么即使用了依赖注入也不能避免包之间的依赖。因此需要把接口都定义在`repository`包内，而`infra`中只包含具体的实现。

要强制这种依赖关系，一种方式是采用module，让各个部分成为独立的编译单元，就比较容易通过maven来控制依赖。但是实际项目中因为依赖情况非常复杂，这种做法可能比较麻烦甚至行不通。

更直观的做法就只是通过package来分隔，然后维护package之间的依赖。项目简单的时候这非常容易，可是当我们在一个复杂的项目中开发的时候就会碰到问题。由于现在的Java IDE太强大，各种自动补全和自动创建文件可能不知不觉就打破了上面提到的依赖规则，把类放在了错误的位置，毕竟谁会去关心各个类的具体路径是什么呢。当然我们可以通过人工检查来发现问题，但是在团队开发中很难强制所有人来执行。

## 包依赖自动检查
所以我们肯定需要引入自动化的检查工具来解决这个问题，本文介绍一下使用[Maven Enforcer Plugin](https://maven.apache.org/enforcer/maven-enforcer-plugin/)配合[restrict-imports-enforcer-rule](https://github.com/skuzzle/restrict-imports-enforcer-rule)的方式。`Maven Enforcer Plugin`本身功能非常强大，可以对代码做各种检查，比如说可以强制通过maven引用的依赖版本没有冲突，或者是不允许出现循环的maven依赖，具体可以参考文档。默认倒是没有对于package的检查规则，不过幸好这个插件提供了扩展的接口。`restrict-imports-enforcer-rule`就基于这个插件实现了对包依赖的检查规则，准确说是对于import语句的检查。因为Java里所有的包依赖总是会通过代码里的import语句来体现，因此我们实现了对于import语句的检查，就能保证包的依赖符合规则。

比如说我们要检查`domain`包不依赖于项目中自己之外的任何包，而`gateway`和`infra`不被任何其他包依赖，就可以这么定义
```
    <restrictImports implementation="de.skuzzle.enforcer.restrictimports.rule.RestrictImports">
        <basePackages>
            <basePackage>**.server.domain.**</basePackage>
        </basePackages>
        <bannedImport>**.server.**</bannedImport>
        <allowedImport>**.server.domain.**</allowedImport>
    </restrictImports>
     <restrictImports implementation="de.skuzzle.enforcer.restrictimports.rule.RestrictImports">
        <basePackages>
            <basePackage>**.server.service.**</basePackage>
            <basePackage>**.server.repository.**</basePackage>
        </basePackages>
        <bannedImports>
            <bannedImport>**.server.gateway.**</bannedImport>
            <bannedImport>**.server.infra.**</bannedImport>
        </bannedImports>
    </restrictImports>
```
上面定义的两条规则应该非常直白了，`basePackage`就是这条规则的适用范围，`bannedImport`就是禁止import的内容，`allowedImport`就是允许import的内容。如果禁止或允许的内容有多条，可以在外边套一个bannedImports或者allowedImports。

假设我们不小心把其中一个文件放错了位置打破了规则，就会出现编译错误：
```
[INFO] --- maven-enforcer-plugin:1.4.1:enforce (check-imports) @ ddd-sample ---
Banned imports detected:
        in file: net/fklj/dddsample/server/service/AccountService.java
                net.fklj.dddsample.server.infra.AccountRepository (Line: 3, Matched by: **.server.infra.**)

[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
```

这个插件还提供了其他一些灵活的配置，大家可以在它的github首页上查看用法，不过上面用到的基本配置也足够满足大部分需求了。由于可以使用通配符，理论上即使有非常多的服务，只要包结构大致类似，就可以定义出来通用的规则，所以还是非常方便使用的。

上面的配置样例可以在[这个项目](https://github.com/fklj/ddd-sample)中找到。