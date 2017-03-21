---
title: Calabash探索3-Calabash进阶
date: 2017-3-17 16:22:58
author : 暴打小女孩

tags: 测试
---


## 前言

上一篇我们讲了Calabash的基本用法，有了上一篇的经验，已经可以写基本的测试脚本了，只不过一些特殊情况会写的不那么方便，这一篇我们讲一些Calabash的进阶用法：大概是这几个方向：

- 在自定义的Steps中使用Query语句。
- 自定义Steps支持环境变量扩展。
- Hooks。
- Calabash源码修改与扩展。

<!-- more -->

## 在自定义的Steps中使用Query语句

上一篇我们简单介绍了Query的用法和通过Query来帮助我们编写测试脚本。
但这个是建立在人工查询结果上，如果可以让脚本自己去查，是不是会更便捷？
Calabash提供的预定义Steps中，只有极个别几种View的Steps，类似ImageView等控件暂时还没有这种待遇，按照一般情况，我们只能选择通过ID，或者人工Query查询ImageView的index。那么如果可以只通过肉眼判断当前ImageView的index就可以编写脚本是不更好呢？

我们来自定义一个Step：

```
When /^I press imageView number (\d+)$/ do |index|
	$imageView = query("android.support.v7.widget.AppCompatImageView")
	touch($editText[index.to_i-1])
end
```

为了保证Steps的自然语义，这里我们也对index做了减1操作。
你也看到了，在Steps的定义体中，是可以使用Query语句的，不过这样的写法稍显繁琐了

```
When /^I press imageView number (\d+)$/ do |index|
	tap_when_element_exists("android.support.v7.widget.AppCompatImageView index:#{index.to_i-1}")
end
```

``tap_when_element_exists``本身也支持一定程度的查询功能。

你可能会问了，既然这些库方法已经支持这样的查询了，在内部写Query语句看起来也没啥用了啊。下面我们把Calabash预定义的Steps源码拿来瞅瞅你就知道用处在哪里了：

```
Then /^I select "([^\"]*)" from "([^\"]*)"$/ do |item_identifier, spinner_identifier|
  spinner = query("android.widget.Spinner marked:'#{spinner_identifier}'")

  if spinner.empty?
    tap_when_element_exists("android.widget.Spinner * marked:'#{spinner_identifier}'")
  else
    touch(spinner)
  end

  tap_when_element_exists("android.widget.PopupWindow$PopupViewContainer * marked:'#{item_identifier}'")
end
```

解释一下：首先查询是否有对应描述的下拉列表控件，查询的结果是一个数组，如果结果是空的，那么就调用点击事件尝试点击一下，不出意外会报错找不到这个控件。
如果不是空的，那就点击这个查询结果的第零个元素。
``touch``方法，如果传递的参数是一个数组，会默认点击第零个元素。
然后点击弹出的PopupWindow中的item。


## 自定义Steps支持环境变量扩展

有几个场景。
- 如果测试不同的系统或者不同分辨率的手机，测试用例内部可能需要细微改变，为了应对这些改变，复制多套脚本进行修改显然不是特别合适。
- 不同账号有不同的权限和数据，同一个测试用例可能需要不同的账号来测试，为了应对不同账号复制多套脚本进行修改显示不是特别合适。
- 初期想要让android和iOS无缝的使用一套脚本不太现实，两套脚本几乎必然，但如果遇到上面的现象，测试中需要使用一套数据进行支持（账号），这个时候两边都维护一套一模一样的数据就不太合适了。

以上场景的解决实质是，**如何在不修改脚本的情况下，可以改变其中的参数？**
其解决方案便是：引入环境变量。

新建一个新的自定义Step:

```
Then /^I enter \$([^\$]*) into input field number (\d+)$/ do |text_ev, index|
  text = ENV[text_ev]
  enter_text("android.widget.EditText index:#{index.to_i-1}", text)
end
```

有两处改变，方法名中的参数接收处：``\$([^\$]*)``和方法体中的参数获取：``text = ENV[text_ev]`` 这里基本照猫画虎就好了。

feature中这样调用：

```
Then I enter $env_account_1 into input field number 1
```

``$`` 后面的变量名是环境变量名。

在执行``calabash-android run **.apk``前先设置环境变量

```
set env_account_1 123456
calabash-android run **.apk
```

这样，在脚本执行过程中，会将123456当做账号填写到输入框中。

但这只是最基础的用法，当需要的环境变量非常多时，再使用这样的方式明显不太合适，这个时候可以将之放入ruby脚本中，也便于维护。

新建test_data1.rb
```
ENV["env_account_1"]="1111111111"
ENV["env_password_1"]="123456"
```

新建test_data2.rb
```
ENV["env_account_1"]="222222222"
ENV["env_password_1"]="123456"
```

终端中执行如下命令

```
Crowdsource-android git:(feature/testing) ✗ irb
irb(main):001:0> require './features/test_data1.rb'
=> true
irb(main):002:0>ENV["env_account_1"]
=> "11111111"
irb(main):003:0> exec('calabash-android run debug.apk ')
Feature: Login feature
...

#脚本执行完毕，切换另一套环境变量
irb(main):001:0> require './features/test_data1.rb'
=> true
irb(main):002:0>ENV["env_account_2"]
=> "2222222"
irb(main):003:0> exec('calabash-android run debug.apk ')
Feature: Login feature
...

```
这样就可以整套整套的替换环境变量进行测试，``ENV["env_account_2"]``命令用来查看环境变量的值。


## Hooks

关于Hooks，我们在前面第二章有简单提过一下。Hooks就是在监听程序运行的某个阶段，并做一些事情。在feature/support文件夹下有三个关于Hooks的文件：
``app_installation_hooks.rb , app_life_cycle_hooks.rb , hooks.rb``
前两个文件分别是对app安装的Hooks,和app生命周期的Hooks。hooks.rb文件夹是空的，由我们自己来编写。
这里暂时还没有做过什么特别的实践，我偷个懒，直接将 立成 [@richardcao](http://richardcao.me)博客中的关于这一部分的段落摘了过来，原文在这里：[http://richardcao.me/2016/10/31/客户端自动化测试小探索/](http://richardcao.me/2016/10/31/客户端自动化测试小探索/)。

```
我们看一个简单的app_life_cycle_hooks.rb理解理解：

require 'calabash-android/management/adb'
require 'calabash-android/operations'
Before do |scenario|
  start_test_server_in_background
end
After do |scenario|
  if scenario.failed?
    screenshot_embed
  end
  shutdown_test_server
end

这是默认就生成好的，从字面意思上看，就是app生命周期的hook，这里可以看到，在主要用到了Before和After关键字进行操作，这个显然很容易就看懂了，于是我自己写了一个在每个step执行之后都等待2秒的hook，下面的代码写在hooks.rb中（这个文件默认生成是空的）：

require 'calabash-android/calabash_steps'
AfterStep do |scenario|
	sleep 2
end

很容易对吧？关于这部分想多了解的可以看cucumber wiki中的Hooks部分。

```
[cucumber wiki中的Hooks部分](https://github.com/cucumber/cucumber/wiki/Hooks)。



## Calabash源码修改与扩展

前面我们提到了两种自定义Calabash Steps的方法，分别是自定义Steps和封装已有的Steps。还有第三种更为深入的定制化方案，那就是修改Calabash源码自定义Actions，这里的Actions指的便是我们之前经常会看到的，类似这样的代码: ``perform_action('swipe', 'right')  perform_action('tap_map_marker_by_title', marker_title, 60000)``

其中swipe,tap_map_marker_by_title实际是方法名，他们的核心，以Calabash-android为例，实际也是Android工程 java代码实现的。如果你有看过其他关于Calabash-android的博客，会有介绍修改其源码，创建自定义的Actions方法的实例。

基本原理是：Calabash目录``/calabash-android/ruby-gem/test-server/instrumentation-backend`` 是一个Android工程，在其包``/src/sh/calaba/instrumentationbackend/actions``中实现了我们用到的ruby库方法。
在其中创建我们自己的Action类，然后进行编译，即可创建我们自己的库方法并使用。

但这样的博客都写于2016年之前，目前最新的Calabash-android项目的源码已经删除了``/calabash-android/ruby-gem/test-server``目录下的文件夹，通过翻阅Git记录得知：Calabash-android在2015.12.12日删除了本地的test_server目录，将其移动到了新的项目：[calabash-android-server](https://github.com/calabash/calabash-android-server)中。如果你将Calabash-android项目切换到2016年之前，还可以找到test_server目录，查看里面的Android工程，修改并重新编译。但我并不建议你这样去做，使用这种方式扩展意味着要放弃 Calabash一年以上的更新进度。

[calabash-android-server](https://github.com/calabash/calabash-android-server)同样是开源项目，虽然暂时我还没搞明白如何在这个项目中修改源码并应用到Calabash-android中，但只是单纯研究其源码还是非常有价值的！

我挑了其中最简单的一个类，我们来研究一下：

../actions/gestures/ClickOnScreen.java

```
package sh.calaba.instrumentationbackend.actions.gestures;


import sh.calaba.instrumentationbackend.InstrumentationBackend;
import sh.calaba.instrumentationbackend.Result;
import sh.calaba.instrumentationbackend.actions.Action;
import android.view.Display;


public class ClickOnScreen implements Action {

    @Override
    public Result execute(String... args) {
        Display display = InstrumentationBackend.solo.getCurrentActivity().getWindowManager().getDefaultDisplay();
        
        float x = Float.parseFloat(args[0]);
        float y = Float.parseFloat(args[1]);
        
        int width = display.getWidth();
        int height = display.getHeight();
        
        InstrumentationBackend.solo.clickOnScreen((x/100)*width, (y/100)*height);
        return Result.successResult();
    }

    @Override
    public String key() {
        return "click_on_screen";
    }

}

```
先看下Steps的定义中是怎么用这个方法的：
```
Then /^I click on screen (\d+)% from the left and (\d+)% from the top$/ do |x, y|
  perform_action('click_on_screen', x, y)
end
```
``public String key()`` 方法便是定义Action方法的名称。``public Result execute(String... args)``便是实现。

Result返回值表示该动作是否执行成功。如果失败，会直接抛到终端中进行显示错误信息。
这里我们重点关注这个对象：``InstrumentationBackend.solo``
获取界面信息，以及真正实现点击操作的，都是通过solo对象来实现的。那么这个solo对象是个啥？

我们摘取``InstrumentationBackend``类的一段代码：

```
...
import com.jayway.android.robotium.solo.SoloEnhanced;
import sh.calaba.instrumentationbackend.automation.CalabashAutomation;
import sh.calaba.instrumentationbackend.query.ui.UIObject;

import java.util.*;

/*
    Utility class based on the current test-server life cycle.
 */
public class InstrumentationBackend {
    private static final String TAG = "InstrumentationBackend";

    public static List<Intent> intents = new ArrayList<Intent>();
    private static Map<ActivityIntentFilter, IntentHookWithCount> intentHooks =
            new HashMap<ActivityIntentFilter, IntentHookWithCount>();

    private static CalabashAutomation calabashAutomation;

    /* Instrumentation does not belong to this class. Here because of old architecture */
    public static Instrumentation instrumentation;

    public static SoloEnhanced solo;
    public static Actions actions;
    
...
```

solo实际是com.jayway.android.robotium.solo.SoloEnhanced类。看到这里应该明白了，Calabash-android对UI的操作核心其实借助以Robotium来做的。

*Robotium是一款国外的Android自动化测试框架，主要针对Android平台的应用进行黑盒自动化测试，它提供了模拟各种手势操作（点击、长按、滑动等）、查找和断言机制的API，能够对各种控件进行操作*

**哦，还要提一点，Calabash不支持跨进程的原有就在于Robotium不支持跨进程，所以让Calabash支持跨进程的契机就在这里，修改其源码，调用uiautomator的API即可，具体可行性和方式还待探索，如果你有兴趣，不妨我们一起研究呀**

## 总结

OK，大概就是这样了，Calabash进阶部分会随时更新，修改Calabash-android-server源码进行扩展的方法我也会继续研究下去，成功以后会随时更新博客。


</br>
 
------


[《Calabash探索1-Run Calabash》](http://lrd.ele.me/2017/03/20/Calabash%E6%8E%A2%E7%B4%A21-Run%20Calabash/)

[《Calabash探索2-Calabash用法详解》](http://lrd.ele.me/2017/03/18/Calabash%E6%8E%A2%E7%B4%A22-Calabash%E7%94%A8%E6%B3%95%E8%AF%A6%E8%A7%A3/)

《Calabash探索3-Calabash进阶》

[《Calabash探索4-Calabash踩坑总结》](https://lizhaoxuan.github.io/2017/03/16/Calabash探索4-Calabash踩坑总结/)












