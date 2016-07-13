---
title: HermesEventBus-饿了么开源的Android跨进程事件分发框架
date: 2016-07-13 15:54:58
tags:
---
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;由于Android不同进程之前不能相互通信，所以当开发过程中遇到跨进程通信的时候,常用的方案就是AIDL(Android Interface Definition Language)通过它我们可以定义进程间的通信接口,但是当应用中出现大量跨进程通信的时候，比如你想体验一下插件化开发或者特殊需求在单应用中需要开多个进程，那么写过AIDL的同学都会有痛不欲生的感觉。现在福利来了，可以试试饿了么开源了一款进程间事件分发的库---[HermesEventBus](https://github.com/eleme/HermesEventBus)。

<!-- more -->

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在介绍HermesEventBus之前先简单介绍一下它底层依赖的库Hermes----同样是由饿了么Android资深工程师赵立飞操刀的一套新颖巧妙易用的Android进程间通信IPC框架,开发Hermes的初衷是为了解决插件化框架[DroidPlugin](https://github.com/DroidPluginTeam/DroidPlugin)的主从进程通信困难的问题,最后实现的效果是将进程间通信变的像调用本地函数一样方便简单，并且支持进程间函数回调和垃圾回收。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;想了解更多，请移步飞神的[Hermes](https://github.com/Xiaofei-it/Hermes)，下面开始正式介绍HermesEventBus。

# HermesEventBus

Hermes-EventBus是一个基于EventBus的、能在进程间发送和接收event的库，在IPC或者插件开发中非常有用。它底层基于
EventBus，并且和EventBus有相同API。

# 原理

事件收发是基于EventBus，IPC通信是基于Hermes。Hermes是一个简单易用的Android IPC库。

![事件流图](http://7o4zmy.com1.z0.glb.clouddn.com/figure.png)

首先选一个进程作为主进程，将其他进程作为子进程。

每次一个event被发送都会经过以下四步：

1、使用Hermes库将event传递给主进程。

2、主进程使用EventBus在主进程内部发送event。

3、主进程使用Hermes库将event传递给所有的子进程。

4、每个子进程使用EventBus在子进程内部发送event。

# 用法

能在app内实现多进程event收发，也可以跨app实现event收发。

## 单一app内的用法

如果你在单一app内进行多进程开发，那么只需要做以下三步：

### Step 1

在gradle文件中加入下面的依赖:

```
dependencies {
    compile 'xiaofei.library:hermes-eventbus:0.1.1'
}
```


### Step 2

在Application的onCreate中加上以下语句进行初始化：

```
HermesEventBus.getDefault().init(this);
```

### Step 3

每次使用EventBus的时候，用HermesEventBus代替EventBus。

```
HermesEventBus.getDefault().register(this);

HermesEventBus.getDefault().post(new Event());
```

HermesEventBus也能够在一个进程间传递event，所以如果你已经使用了HermesEventBus，那么就不要再使用EventBus了。

## 多个app间的用法（使用DroidPlugin的时候就是这种情况）

如果你想在多个app间收发event，那么就做如下几步：

### Step 1

在每个app的gradle文件中加入依赖：

```
dependencies {
    compile 'xiaofei.library:hermes-eventbus:0.1.1'
}
```


### Step 2

选择一个app作为主app。你可以选择任意app作为主app，但最好选择那个存活时间最长的app。

在使用DroidPlugin的时候，你可以把宿主app作为主app。

在主app的AndroidManifest.xml中加入下面的service：

```
<service android:name="xiaofei.library.hermes.HermesService$HermesService0"/>
```

你可以加上一些属性。

### Step 3

在app间收发的事件类必须有相同的包名、相同的类名和相同的方法。

务必记住在代码混淆的时候将这些类keep！！！

### Step 4

在主app的application类的onCreate方法中加入：

```
HermesEventBus.getDefault().init(this);
```

在其他app的Application类的onCreate方法中加入：

```
HermesEventBus.getDefault().connectApp(this, packageName);
```

“packageName”指的是主app的包名。

### Step 5

每次使用EventBus的时候，用HermesEventBus代替EventBus。

```
HermesEventBus.getDefault().register(this);

HermesEventBus.getDefault().post(new Event());
```

HermesEventBus也能够在一个进程间传递event，所以如果你已经使用了HermesEventBus，那么就不要再使用EventBus了。



**[HermesEventBus](https://github.com/eleme/HermesEventBus) 现已开源，欢迎大家前去提PR。**







