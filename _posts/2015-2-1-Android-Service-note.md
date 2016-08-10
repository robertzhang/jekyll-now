---
layout: post
title: Android Service笔记
subtitle:   "该文档是对android service简单的记录"
date:       2015-02-01
author:     "Robert Zhang"
header-img: "img/post-bg-android.jpg"
tags:
    - Android
    - 基础
---

Service是Android的四大控件之一。主要适用于需要常驻后台运行的进程，或者比较耗时的操作。

Android Service分为系统服务和应用程序服务。系统服务指的是开机后，由系统启动并维护的服务。这部分服务是为系统正常运行所必须的服务，并且也为应用程序提供了一个运行环境

在这里我主要讲解一下应用程序服务
* Local Service
* Remote Service
本地服务和远端服务的最大区别是Service运行的空间不同。本地服务和启动他的activity在同一个进程中，而远端服务和activity并不在同一个进程中，所以当主应用程序终止时，远端服务仍然会继续运行。

## Local Service（本地服务）

大多数情况下，我们使用的Service都是本地服务。activity通过bindService()方法来绑定和启动需要控制的Service。而LocalService通过onBind()方法将Service返回给activity供其使用。

## Remote Service（远端服务）

远端服务因为在单独的进程中，所以要和activity进行交互就必须要提到IPC机制。我们都知道IPC能使用，就需要AIDL的知识。AIDL就是Android Interface Definition Language，用于约束两个进程间的通信规则。我们在设计远端服务的时候，只需要编写好以.aidl为后缀的接口文件，Android SDK会帮我们自动在gen目录下生成对应的.java文件。这个部分的java文件不用我们过多的关注。 接下来我们需要实现一个抽象类IService.Stub，并定义一个binder。类似于：

~~~
private final IService.Stub binder = new IService.Stub(){
    .... //实现.aidl接口中定义的方法
}
~~~
我们再通过Service的onBind方法将上面定义binder返回给activity。这样剩下的处理逻辑就和本地服务差不多了。

To be continuing .....

后续持续添加


