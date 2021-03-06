---
title: 探究liunx中jdk多版本使用配置
tags:
- jdk
categories:
- jdk
---

在Java开发中我们都知道我们的程序运行环境都需要配置JDK开发环境，在程序运行中需要JRE环境。但是有时候我们的应用是会要求使用不同的JDK版本的，这样的话对于在同一台机器上进行不同版本的JDK JRE环境进行怎样处理实现就变成了一个问题。
<!-- more -->
# 提出问题
在我的开发机中一般是使用JDK8的版本。但是有时候我又想学习编写一些JDK11版本的Java代码，这样的话要在开发机上配置JDK版本就是一个比较困难的点。
## 方式思考
- 1、在不同的开发机中配置不同的JDK版本，不同版本的程序部署到不同的开发机
- 2、在同一台开发机中配置不同的JDK版本，在需要某一个版本的时候，在系统中进行环境变量的切换
- 3、再同一台开发机中配置不同版本的JDK，在服务生命周期管理程序中，进行环境变量的指定

这里我们主要是介绍第二种方式，我的思考和实现

# JDK配置在linux系统生效的知识基础

为了更好的理解多版本的配置，我这里总结我的一些linux 变量的认识
## shell变量
变量的定义：让某个特定字符串代表不固定的内容。在linux中有很多的变量在服务器启动，或者打开某一个bash连接的时候都会有变量的产生写入到我们的会话进程中，

变量的设置
```shell
# variable=testVariable
```

变量的获取，在sh或者命令行中在变量前加$
```shell
# echo $variable
```

变量的取消
```shell
# unset variable
```

> 在设置变量时，使用""中的特殊符号会保存自己的特性，使用''中的特殊符号不会被视作特殊符号

## 变量的分类

一般变量分为`自定义变量` `环境变量`（这个是我总结的有可能不是很标准）

两者的区别在于：子进程只会继承父进程的环境变量，而不会继承父进程的自定义变量。

所有当使用当前进程创建新的进程的时候自定义变量是不会被子进程访问的。Java的进程就是由父进程产生的

## 如何查看shell变量

```shell
# env 
## 或者
# export
```
通过env可以查看当前shell环境下的所有环境变量和内容

```shell
# set
```
查看所有当前shell的变量（包括环境变量和自定义变量）

## 将自定义变量变为环境变量

使用export命令可以将自定义变量变为环境变量
```shell
# export variable
```
这样子进程就可以查询到设置的变量了

# Jdk使用需要到的环境变量解释

为什么我们需要配置JDK在系统中的环境变量呢？这里我大概总结了一下
## PATH环境变量

PATH环境变量是去寻找命令行的，，比如我们的`mv` `cp` `cd`等都是在PATH定义好路径的。所有我们需要执行`java` `javac` `jstack` 都需要定义好路径，否则只能每次启动的时候完全指定到这些命令的路径才能找到命令了

## CLASSPATH环境变量

CLASSPATH环境变量 的作用是指定类的搜索路径，一般在启用java程序的时候我们都会去找对应需要加载的类，在程序执行中也需要去寻找对应加载的类，这些类也是在CLASSPATH中指定的。
所以一般`CLASSPATH`变量都包含:

`.` 这表示当前目录（调用CLASSPATH的工作目录) 
`${JAVA_HOME}/lib` jdk下面的基础类，有可能会用到
`${JAVA_HOME}/jre/lib` jre提供的基础类  jdk11没有jre包了

在jdk5以上的版本可以不用设置CLASSPATH程序也能正常的运行

## JAVA_HOME环境变量

指向JDK的安装目录，各种IDE就是通过JAVA_HOME的环境变量来找到本机安装好的JDK版本。这个很重要因为 有些IDE就是通过java环境来启动运行的。找不到的话很容易启动失败

# 在linux服务器进行多版本部署和切换

我的机器上将`jdk11`和`jdk8`分别安装在路径`/usr/local/java/jdk-11`和`/usr/local/java/jdk-8`中

## 编写设置环境变量脚本
我们需要编写设置环境变量脚本，来达到我们linux jdk版本环境变化的目的，下图是我的jdk8环境变量的配置。 jdk11类似就是java_home路径不同而已

![JDK-8环境变量脚本 jdk_env_1.8.sh](https://raw.githubusercontent.com/forvoid/imageHosting/master/blog/jdk_env_config_1.8.png)

我们知道，在设置一个PATH为`/usr/local/java/jdk-11`和`/usr/local/java/jdk-8`后需要对他们进行删除在配置新的才有效，不然一直都要设置 重复的`/usr/local/java/jdk-8`PATH路径造成 PATH重复很多的无用变量。但是如果不设置，又会使用最前面的作为路径，导致无法切换JDK11 和JDK8.

在我的脚本中我对已有的PATH数据进行了分解，将含有`java`字眼变量路径进行了忽略，然后将当前的`$JAVA_HOME`进行了写入。可以使PATH中只有当前`$JAVA_HOME`的路径。

这样只需要执行
```shell
# source jdk_env_1.8.sh
``` 

就可以生效脚本中的JDK环境变量。然后在linux服务器中生效当前的jdk版本，新创建的子进程也将采用新的环境变量进行启动，不影响已启动的不同版本的java服务。

## 默认选择生效的版本

一般情况下我们对jdk的版本很少更改，所有为了不要每次重启或者启动都设置一个jdk版本。我们需要在启动时的环境变量中配置执行我们的jdk脚本，例如将我们常用的JDK8的脚本配置到 系统的环境变量配置启动中

```shell
# vi /etc/profile
```

打开文件在最下面加一行

```vi
source /home/work/env/jdk_env_1.8.sh # 我的脚本路径
```

这样就可以了。当我要使用jdk11版本的是有启用命令行

```shell
# source /home/work/env/jdk_env_11.sh # 我的jdk11脚本路径
```

这样就可以使用jdk11启动程序了

这里需要注意的是修改了之后，如果还想在用jdk8的话，就只能重新source jdk8的环境变量设置脚本了或者重启服务器了。

# 总结

这里只介绍了一种在同一台服务器启用不同jdk版本的方式，也是我自己摸索 实践的方式，希望能够保存下来以后多看看。

在没有加入小米之前我，一直认为一个服务器只能使用一个jdk版本，在小米工作中发现。可以通过指定ruby的god工具指定jdk在jobs的环境变量指定到不同的路径来做一个服务器使用不同的jdk版本的方式。启发了我去思考，为什么要配置JDK的环境变量。配置这些变量有什么用。

后面我将使用god或者其他的服务管理工具进行在同一个服务器上的使用不同的jdk版本进行服务的部署。

