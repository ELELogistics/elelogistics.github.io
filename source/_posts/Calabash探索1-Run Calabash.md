---
title: Calabash探索1-Run Calabash
date: 2017-3-20 16:20:58
author : 暴打小女孩

tags: 测试
---

## 前言

作为这个系列的第一篇，先介绍一下大纲：

- 《Run Calabash》简单介绍Calabash环境搭建及基本运行，然你给对Calabash有一个基础认识
- 《Calabash用法详解》认识Calabash后介绍Calabash的基础用法，从而可以快速上手
- 《Calabash进阶》介绍一些特殊或不常用到的技巧，对Calabash有更深的认识
- 《Calabash踩坑》为不影响文章阅读的连贯性，相关异常解决的内容单列一章

<!-- more -->

看了一些Calabash入门的文章，便准备开始实践。但在最开始时，真的好气啊。 
从来没人告诉我Mac安装Calabash需要xcode支持！
大部分的入门demo止步于一个简单的case，从来没人告诉我后续的case需要怎么继续执行！！！
一条怎么看怎么没毛病的语句，怎么就无脑报错呢？
...

其实这些都是开发中的一些意外情况，但就是这些意外情况，让好多人就此止步了。其实如果耐心一点，在官方文档深挖一下，把十几篇几十篇博客参照着一起看，再或者辛苦一点，多多试错，这些坑其实都能踩平。
但是这种方法代价太大了，作为一个教程博客，我想把我从最开始一直到最后经历所有,以初学者的角度记下来，让后来者可以更轻松一点。



OK，下面我们就从零Run起来！


<!-- more -->

## Calabash环境搭建

（之后环境搭建及使用均为Mac环境下，且基于Calabash-Android）

### 环境及依赖配置

这一部分有大量的入门文章进行了详细的讲解，这里就不在重复叙述了，我们直接引用官方说明：

*You need to have Ruby installed. Verify your installation by running ruby -v in a terminal - it should print “ruby 2.0.0” (or higher). We recommend using a managed version of Ruby like rbenv or rvm.*
*If you are on Windows you can get Ruby from RubyInstaller.org*
*You’ll also need to have the Java Development Kit (JDK) installed and available. Calabash will attempt to automatically find this from registry keys on windows, or monodroid config elsewhere, but you can also specify it explicitly by setting the JAVA_HOME environment variable to its location (e.g. C:\Program Files\Java\jdk1.8.0_20), or having the JDK binaries themselves (i.e. C:\Program Files\Java\jdk1.8.0_20\bin) in your path.*
*You should have the Android SDK installed. You can download it from here. Create an environment variable with the name : ANDROID_HOME and its value pointing to the location of the unzipped downloaded SDK.*
*To compile Calabash-Android from source, you will also need to have Ant installed and added to your path. It can be downloaded from here.*

简单来说，就是首先你需要保证你的电脑安装了Ruby，这个大部分Mac电脑都已经安装，可以在终端中使用命令``ruby -v``检查是否已成功安装。

然后你还需要保证电脑中安装了JDK，并且配置好java环境变量。
再然后你需要保证电脑中安装了Android SDK和 Ant，并且成功配置了环境变量。

如此，依照官网所说，你的Calabash的环境依赖配置基本已经完成。
（关于Ruby安装、JDK SDK ANT的安装及环境变量配置，网上有很多资料哦~）
### Calabash安装

依照官网步骤安装Calabash可能稍有繁琐，这里我们直接使用一行命令搞定：

执行命令：

	sudo gem install calabash-android
    
然后按照提示，输入开机密码即可。

如果你的网络没有翻墙，且出现很长时间都安装不了，参照别的博客给出的提示，可以切换到淘宝源，在终端下执行如下的三句命令即可：

	gem sources --remove https://rubygems.org/
	gem sources -a http://ruby.taobao.org/
	gem sources -l

然后重新执行命令：

	sudo gem install calabash-android
    
就此，如无意外，Calabash安装成功。也有可能你遇到了我曾遇到的问题： **坑1：Calabash安装时Ruby报错**（因为这次分享的特殊性，超链无法使用，关于踩坑内容在第四篇文章）

### 创建Cucumber项目结构

Calabash安装成功后，使用终端切换到你准备写测试case的目录执行命令：

	calabash-android gen
    
执行成功后，将会在该目录下生成如下文件：

```
features  
|_support  
| |_app_installation_hooks.rb  
| |_app_life_cycle_hooks.rb  
| |_env.rb  
| |_hooks.rb  
|_step_definitions  
| |_calabash_steps.rb  
|_my_first.feature 
```

#### features主文件夹
features 是主文件夹，一般情况，你的自动化脚本执行，都是在这个文件夹下编写操作的了。

#### .feature文件
**my_first.feature** 是默认生成的第一个测试脚本文件。通常每一个.feature文件中只能写一个Feature,所以每一个Feature文件都是一个测试用例集，在不指定具体的.feature文件夹的情况下，Calabash会按照随机顺序执行所有.feature文件。（.feature文件并不是必须在features根目录下，为了便于管理，你可以在根目录下随意创建文件夹存放.feature文件，Calabash会自动进行查找并执行）

#### step_definitions文件夹
step_definitions文件夹里存放着自定义的step,之前我们看到的大白话，就是在这里定义的，里面已经有一个默认生成的**calabash_steps.rb** 文件，它引用了Calabash预定义的Steps，以让我们直接使用：

	require 'calabash-android/calabash_steps'

我们也可以在这个文件夹下创建自己的.rb文件来自定义steps。这是后续我们会深入讲解的一个部分。[App自动化测试探索3-Calabash语法及策略详解](https://lizhaoxuan.github.io/2017/02/13/App自动化测试探索3-Calabash用法详解/)

#### support文件夹
support文件夹存放着一些依赖文件
先看**env.rb**:

	require 'calabash-android/cucumber'
    
所以为什么说cucumber是Calabash的核心呢~  [Cucumber-github](https://github.com/cucumber/cucumber)
上一篇我们看的脚本的几个关键字：Feature，Scenario，When，Then，在Calabash的官方文档并没有对他们着重进行介绍，因为它们其实是属于cucumber的部分，Calabash的资料稀少,所以很多资料我们需要去看cucumber。不过没关系，一些基础的，必然会用到的内容后续博客会讲到。

**app_installation_hooks.rb** 和 **app_life_cycle_hooks.rb**
两个文件所起到的作用分别是 对app安装hooks,和对app生命周期进行hooks。简单来说就是在app安装过程中和生命周期过程中的某些阶段做些什么事情。

所以到这里你也发现了，Calabash其实是一个很轻量级，透明的框架，这些文件你完全可以自己进行修改，添加自己想要的操作。

还有一个**hooks.rb**文件：
这是一个空文件，你可以在这里添加自己想要的hooks操作，我们在下一章会简单介绍一下。


上面说了很多可以自定义的文件夹，所有以.rb为文件后缀的文件，使用的语言都是Ruby，对Ruby陌生没关系，Ruby其实很简单的，这里安利一个[Ruby教程](http://www.runoob.com/ruby/ruby-tutorial.html)。

## 运行你的第一个自动化测试用例

环境配置结束，文件结构也介绍完了。按照我一直以来的习惯，不管三七二十一，我们先把demo跑起来，中间细节通通忽略，先看看是怎么回事，有了成就感我们再逐步深入！

修改**my_first.feature**

```
Feature: Welcome feature

  Scenario: 欢迎页面测试用例

	#左划
	Then I drag from 90:50 to 20:50 moving with 20 steps 
	Then I drag from 90:50 to 20:50 moving with 20 steps 
	Then I drag from 90:50 to 20:50 moving with 20 steps 
	Then I wait for 5 seconds  
	Then I take a screenshot 

```
（这里我已蜂鸟众包App为例进行测试，你也可以选择从应用市场下载蜂鸟众包进行同步测试，或修改脚本在自己的app进行测试，反正脚本很简单的啦）

简单解释一下
第一行Feature 表示功能测试的名称
第二行Scenario 表示应用场景
内容完全是给人看，所以写什么都无所谓，但这两个关键字可不简单，它们是你搭建自动化测试功能的重要策略。当然我们先卖个关子继续往后看。

第三到第五行 从坐标90:50 移动到坐标 20:50，做左划操作，20表示拖动速度，翻过三个页面后等待5秒然后截屏。

执行命令 

	calabash-android run test.apk
    

按照正常情况应用程序会安装到手机上或虚拟机上，启动并执行脚本命令。但你也可能会遇到问题：**坑2：Could not find an Android SDK please make it is install**
**坑3：App did not start 或 WARN:Did not find 'android.jar'...**
**坑4：\*\*.apk is not signed with any of the available keystores**

最终截屏图片保存在当前目录下，也可以自定义保存目录，这个我们后面的文章会讲到。

脚本执行过程中控制台会输出如下内容和结果：

```
Crowdsource-android git:(feature/testing) ✗ calabash-android run debug.apk
Feature: Welcome feature

  Scenario: 欢迎页面测试用例    
	#左划
	Then I drag from 90:50 to 20:50 moving with 20 steps 
	
    Then I drag from 90:50 to 20:50 moving with 20 steps 
	
    Then I drag from 90:50 to 20:50 moving with 20 steps 
	
    Then I wait for 5 seconds  
	
    Then I take a screenshot 
    
1 scenario (1 passed)
5 steps (5 passed)
```

每执行一个Step就会在控制台上输出，包括注释内容，所以这里你完全可以在脚本中添加一些说明性的注释或在自定义的Steps中使用ruby输出语句来增强提示。这些后面我们也会详细讲。

上一篇文章我们也提到过Calabash支持三种不同程度的自定义Steps，按照实际的业务需求，自定义一些更简洁的更符合自然语言的Steps。
比如，我们使用ruby重新自定义一个Step,将上面三个左划操作集合成一句命令：在step_definitions文件夹中创建一个drag_steps.rb文件，分别修改drag_steps.rb文件和my_first.feature文件：

*drag_steps.rb:*
```
Then /^I through welcomePages$/ do
	perform_action('drag', 90, 20, 50, 50, 20) 
	perform_action('drag', 90, 20, 50, 50, 20) 
	perform_action('drag', 90, 20, 50, 50, 20) 
end
```
*my_first.feature：*

```
Feature: Welcome feature

  Scenario: 欢迎页面测试用例

	Then I through welcomePages
	Then I wait for 5 seconds  
	Then I take a screenshot 
```

这就是自定义setp，但是！当然，这只是Demo，我们为了方便这样去写，在实际开发中，这个Steps开发应该更规范严谨，我们会在下一章详细对这个问题进行说明。

## 总结

总结下来，Calabash真的是一个非常轻量级，且脚本维护成本很低的一个自动化测试框架。后面几章我们将从以下几个方面对Calabash深入：

- Feature、Scenario等关键字使用策略。
- Calabash预定义Steps的使用。
- 自定义Steps
- 使用Ruby语法编写带逻辑判断的Steps。
- 自定义Steps的使用策略。
- Ruby Query的使用。
- view定位技巧。

进阶

- 在自定义的Steps中使用Query语句。
- 自定义Steps支持环境变量扩展。
- Hooks。
- Calabash源码修改与扩展。

恩，这么看下来，是不感觉Calabash虽然很轻量，但还是很强大的吧？


</br>
 
------

《Calabash探索1-Run Calabash》


[《Calabash探索2-Calabash用法详解》](http://lrd.ele.me/2017/03/18/Calabash%E6%8E%A2%E7%B4%A22-Calabash%E7%94%A8%E6%B3%95%E8%AF%A6%E8%A7%A3/)

[《Calabash探索3-Calabash进阶》](http://lrd.ele.me/2017/03/17/Calabash%E6%8E%A2%E7%B4%A23-Calabash%E8%BF%9B%E9%98%B6/)

[《Calabash探索4-Calabash踩坑总结》](https://lizhaoxuan.github.io/2017/03/16/Calabash探索4-Calabash踩坑总结/)















