---
layout: post
title: 为什么要开发mybatis.flying
description: Mybatis是一个简约而强大的orm解决方案,而mybatis.flying是它的扩展插件。
category: blog
---

## mybatis.flying是什么

众所周知，Mybatis在设计之初是作为一个工作于单线程之下的中间件而存在，虽然它易于上手，但放到互联网环境下使用时，初学者不可避免的要面对诸如“一级缓存存在脏数据”、“需要写大量明文SQL语句”等问题。这些是用户最关心的“是否可用”级别问题，也是Mybatis-3系列版本中开发团队并没有完全解决的问题，或者说，相对于另一个竞争产品Hibernate，Mybatis的开发团队选择了一种更“谦逊”的方式，他们开放Mybatis接口，允许用户开发插件，按自己的方式来解决这些问题。于是，一切ORM领域相关的问题在Mybatis上通过插件都有了解决方案。而本文所要介绍的mybatis.flying，则集成了用户最需要的几个功能，同时提出一种新的调用数据方式，希望能起到抛砖引玉的作用。

## 如何获取mybatis.flying

因为目前mybatis.flying还不在maven中心库内（尚在apache上申请中），所以若要使用需要先在maven全局配置文件或POM文件中加入以下仓库： 

    <repository>
        <id>dpm-Release</id>
        <url>http://vpn.dpm.im:8081/nexus/content/repositories/releases/</url>
    </repository>
    
之后在POM文件中加入依赖：

    <groupId>indi.mybatis</groupId>
    <artifactId>mybatis.flying</artifactId>
    
即可以使用。当前最新版本为0.7.0。flying默认依赖Mybatis-3.2.5版本，如果您对实际项目中的Mybatis版本有特别需求（但建议使用Mybatis-3系列内的版本）或您使用的是个人定制版的Mybatis，您可以在依赖配置中加入exclusions，就像下面这样：

    <dependency>
        <groupId>indi.mybatis</groupId>
        <artifactId>mybatis.flying</artifactId>
        <version>0.7.0</version>
        <exclusions>
            <exclusion>
                <groupId>org.mybatis</groupId>
                <artifactId>mybatis</artifactId>
            </exclusion>
        </exclusions>
    </dependency>

在您做好以上配置之后，flying会自动下载到您的电脑本地。

## 如何使用mybatis.flying

使用本插件前您需要具备Java开发数据库应用的知识，和使用Mybatis基本功能的经验。
在Mybatis中，有一个核心的配置文件，称做Configuration.xml，我们的插件需要先在这里配置，才能在运行期被识别出来。

- Unix 系统里，每行结尾只有"<换行>"，即"\n"
- Windows 系统里面，每行结尾是"<回车><换行>"，即"\r\n"
- 在老的 Mac 系统里，每行结尾是"<回车>"，即"\r"

我们来验证一下，我在 Windows 下用记事本新建一个文本文件，它的二进制编码如下：

    <repository>
        <id>dpm-Release</id>
        <url>http://vpn.dpm.im:8081/nexus/content/repositories/releases/</url>
    </repository>

同样在 Mac 下用 Vim 新建一个：

    //源文件内容
    hello
    hello2

    //二进制内容
    0000000: 6865 6c6c 6f0a 6865 6c6c 6f32 0a         hello.hello2.

`0a`是 LF 的 ASCII 编码, `0d`是 CR 的 ASCII 编码。区别很明显了

- Mac 下的文本文件在 Windows 下打开会成为一行，因为 Windows 只认识`\r\n`，也就是`0d0a`
- Windows 下的文本文件在 Mac 下打开，Vim 中会在每行末尾显示一个 `^M`，(不是两个字符组成的)

## 文件末尾空行

[POSIX](https://zh.wikipedia.org/zh-sg/POSIX)对行的[定义](http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap03.html#tag_03_206)如下：

  > 3.206 Line

  > A sequence of zero or more non- <newline\> characters plus a terminating <newline\> character.

  > 行是由0个或者多个非 "换行" 符的字符组成，并且以 "换行" 符结尾。

这样做有什么好处呢，举个例子：

    //hello.c
    #include head.h
    print('hello')

    //world.c
    #include tail.h
    print('hello')

如果这两个文件都按 POSIX 规范来写， 在`cat *.c`之后，是没有问题的：

    //cat.c

    #include head.h
    print('hello')
    #include tail.h
    print('hello')

如果缺少最后一行的换行符（如 Windows 文件那样的定义），`cat`之后，就有问题了：

    //error.c

    #include head.h
    print('hello')#include tail.h
    print('hello')

所以，从这点去理解 POSIX 对行的定义，非常合理，对于任意文件的拼接，也各自保持了文件的完整性。

不遵守标准带来的则是：在一些编辑器下面，比如 Sublime，他把`\n`的当做了行之间的分隔符，于是文件最后一行的`\n`就看上去成了一个新的空行，这就是错误解读标准造成的，拼接文件时也会产生不必要的麻烦，比如上例。

## \ No new line at end of file

基于上面的原因，再去看 git diff 的`\ No new line at end of file`信息，就很好解释了。

各编辑器对于换行符的理解偏差，导致的文件确实发生了变化，多了或少了最后的`0a`，那么对于 diff 程序来说，这当然是不可忽略的，但因为`0a`是不可见字符，并且是长久以来的历史原因，所以 diff 程序有个专门的标记来说明这个变化，就是：

`\ No new line at end of file`

各编辑器也有相应的办法去解决这个问题，比如 Sublime，在`Default/Preferences.sublime-settings`中设置：

    // Set to true to ensure the last line of the file ends in a newline
    // character when saving
    "ensure_newline_at_eof_on_save": true,

所以，请遵守规范。

[BeiYuu]:    http://beiyuu.com  "BeiYuu"
