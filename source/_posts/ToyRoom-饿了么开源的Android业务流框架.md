---
title: ToyRoom-饿了么开源的Android业务流框架
date: 2016-09-05 20:21:40
author : 进击的小羊
tags: Android
---

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如何优雅的解偶？这是所有开发者都面临的一个大问题，饿了么给大家提供了新的选择-[ToyRoom](https://github.com/eleme/ToyRoom)。

<!-- more -->
  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;大家在平时的业务开发过程中肯定遇到过这样的情况：在面向业务逻辑的编程中，一个特定的业务对象的改变可能会引起各个组件的变化，业务逻辑的复杂性也会增加模块之间的耦合。比如在订单列表页面，我们调用了一个异步网络请求去获取订单数据，当网络数据返回后，需要对数据进行处理，然后将数据保存到数据库，再通知多个tab页面刷新UI，之前我们使用EventBus来使用通知分发，但是当业务越来越多的时候，我们发现EventBus的onEvent方法维护变的不可控，因为我们不知道有多少个地方响应了这个Event，最后代码就变得难以维护，这时候我们就在想如何能把这些响应逻辑像流水线一样在一个地方铺开，先执行A，再执行B，最后执行C，一目了然，让世界重新回到我们的掌控中。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;基于这个需求，饿了么开发了一套面向业务逻辑的编程库-[ToyRoom](https://github.com/eleme/ToyRoom)，ToyRoom库提供了一种全新的编程模式，将业务对象的变化对各个模块的影响通过方法链表示出来。在方法链中，每个方法有一个action参数，这个action执行相应的操作改变特定的组件。方法串起来后就代表了一系列的对各个组件的操作。这样你就能从这一个文件中看到整个**世界**的变化。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;很多人会觉得这个库和RxJava很像，但其实只是API相似，仅仅是API相似！两个库针对的痛点、开发的初衷、用法、源码、方法论都是不同的。ToyRoom与RxJava完全不相干，绝对不是RxJava的替代品！希望大家搞清楚，并且重视这个区别。作者最担心的是曲高和寡，也担心很多人误以为这个库与RxJava类似。事实上，ToyRoom的开发难度非常大，很多源码的开发技术是原创的。具体区别请看[理论部分](https://github.com/eleme/ToyRoom/blob/master/doc-zh-cn/THEORY.md)的“与RxJava比较”一节。而且用过RxJava的同学会对它感觉格外亲切。诺，ToyRoom现在已经开源，欢迎大家品鉴。

## 特色

1. 为面向业务逻辑的编程提供一种全新的编程模式。

2. 无论业务逻辑怎么变化，使用ToyRoom库编写的代码都易于理解和维护。

3. 能够方便地发送网络请求和处理回调，尤其是发送并发请求和连续请求。

4. 能够方便地执行耗时任务和处理回调。

5. 提供强大丰富的数据流控制的API和线程调度的API。

## 预览

ToyRoom库基于[Shelly](https://github.com/Xiaofei-it/Shelly)库。[Shelly](https://github.com/Xiaofei-it/Shelly)是一个面向业务逻辑的编程库，它提供了一种全新的编程模式，将业务对象的变化对各个模块的影响通过方法链表示出来。

使用ToyRoom库时，你可以使用一个方法链创建一个名为“Domino”的对象。方法链中的每个方法都以一个action作为参数。创建的Domino一旦被调用，就会根据方法链中的action序列执行每个action。

在介绍之前，我们先看一个例子。

假设现在你想打印文件夹里所有文件的名字。使用ToyRoom库，你可以写如下代码：

```
Shelly.<String>createDomino("Print file names")
        .background()
        .flatMap((Function1) (input) -> {
                File[] files = new File(input).listFiles();
                List<String> result = new ArrayList<String>();
                for (File file : files) {
                    result.add(file.getName());
                }
                return result;
        })
        .perform((Action1) (input) -> {
                System.out.println(input);
        })
        .commit();
```

上面的代码用方法链打印文件夹中的文件名。文件夹的路径被传入，`Function1`获取此路径下的所有文件并将文件名传给`Action1`，`Action1`将文件名打印出来。

我们看一个稍微复杂的例子。假设现在你想使用Retrofit发送HTTP请求，然后

1. 如果服务端的响应成功，那么调用`MyActivity`和`SecondActivity`中的两个函数；

2. 如果服务端的响应失败，那么在屏幕上显示一个toast；

3. 如果在发请求的时候出现错误或者异常，那么打印错误信息。

使用ToyRoom库，你可以写下面的代码：

```
Shelly.<String>createDomino("Sending request")
        .background()
        .beginRetrofitTask((RetrofitTask) (s) -> {
                return netInterface.test(s);
        })
        .uiThread()
        .onSuccessResult(MainActivity.class, (TargetAction1) (mainActivity, input) -> {
                mainActivity.show(input.string());
        })
        .onSuccessResult(SecondActivity.class, (TargetAction1) (secondActivity, input) -> {
                secondActivity.show(input.string());
        })
        .onResponseFailure(MainActivity.class, (TargetAction1) (mainActivity, input) -> {
                Toast.makeText(
                    mainActivity.getApplicationContext(),
                    input.errorBody().string(),
                    Toast.LENGTH_SHORT
                ).show();
        })
        .onFailure((Action1) (input) -> {
                Log.e("Eric Zhao", "Error", input);
        })
        .endTask()
        .commit();
```

一个URL被传入，Retrofit发送HTTP请求，之后根据不同结果执行相应的操作。

代码中有一些线程调度相关的东西，比如`background()`和`uiThread()`。`background()`是说下面的操作在后台执行。`uiThread()`是说下面的操作在主线程（UI线程）执行。

上面的例子中，你可以看出发送HTTP请求后`MainActivity`和`SecondActivity`是如何变化的。我们在一个地方就可以看到整个世界的变化。


注意，如果不调用Domino，上面这段代码实际上并不会执行任何操作！这段代码做的只是提交并存储Domino，供以后使用。想要让Domino执行操作，必须调用它。只有调用Domino后，它才会执行操作。

这些只是简单的例子。实际上，ToyRoom库是非常强大的，将在后面几节介绍。


## Gradle

```
compile 'me.ele.android:toyroom:0.1.1'
```


## 用法

具体的用法请异步--https://github.com/eleme/ToyRoom  ，这里有详细的DEMO和测试用例。
