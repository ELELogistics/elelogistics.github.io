
---
title: Calabash探索2-Calabash用法详解
date: 2017-3-18 16:21:58
author : 暴打小女孩

tags: 测试
---


## 前言

上一篇Calabash Run起来以后，如果你立刻在自己的项目上进行尝试，我相信你一定会像我之前一样，一头雾水，那么从这篇开始，我们来对Calabash深挖。 大概从这几个方向开始：

- Feature、Scenario等关键字使用策略。
- Calabash预定义Steps的使用。
- 自定义Steps。
- 使用Ruby语法编写带逻辑判断的Steps。
- 自定义Steps的使用策略。
- Ruby Query的使用。

<!-- more -->

## Feature、Scenario等关键字使用策略

主要的关键字如下Feature、Scenario、Given、When、Then。前面我们也说了，其实这几个关键字实际是Cucumber体系下的，如果你想更进一步的了解，请查询[Cucumber相关资料](https://github.com/cucumber/cucumber)。
我们重点讲Feature、Scenario两个。

一个.feature文件只能含有一个Feature，以此作为一个测试用例集，一个测试用例集（Feature）可以包含多个场景（Scenario）。

在不指定具体的.feature文件及其顺序的情况下，Calabash会遍历features根目录下的所有Feature按随机顺序执行。
每执行一个Feature都会卸载重新安装App，而执行Scenario则不会重新卸载安装App，而是重新启动。

举例我有以下两个文件：

*login.feature* ： 测试登录和注册两个业务

```
Feature: Login 测试

	Scenario: 使用无效手机号注册

		Then I...
    	...
    
	Scenario: 使用正确手机号、无效验证码注册
  
		Then I...
		...
    
	Scenario: 使用正确手机号、正确验证码注册
   
        Then I...
        ...
	
    Scenario: 使用错误手机号登录
   
        Then I...
        ...
    Scenario: 使用正确手机号登录
   
        Then I...
        ...
	
```

*health.feature* ： 测试上传健康证业务

```
Feature: Health 测试

	Scenario: 登录

		Then I...
    	...
    
	Scenario: 使用错误健康证号码上传
  
		Then I...
		...
    
	Scenario: 使用正确健康证号码上传
   
        Then I...
        ...
	
```

**切记：在设计时，每个Feature都应该是完全独立，他们之间不能有任何耦合（App都卸载了，你互相依赖还有啥用）**
因为每个Feature都是独立的，所以完全不必在意他们的执行顺序。

使用命令 ``calabash-android run test.apk`` 执行时，Calabash会遍历features主目录下所有的.feature文件进行执行。
当你只想执行某一个或某几个Feature时，直接在该命令后加Feature文件路径即可

```
calabash-android run test.apk ./login.feature ./health.feature

```
值的一提的是，当你指定执行哪些Feature时,Calabash会按照你指定的顺序进行执行。
    
那么当我执行``calabash-android run test.apk`` 会发生什么呢？

```
1.安装App
2.执行Feature: Login 测试
3.启动App
4.执行Scenario: 使用无效手机号注册
5.关闭App,重启App
6.执行Scenario: 使用正确手机号、无效验证码注册
7.关闭App,重启App
8.执行Scenario: 使用正确手机号、正确验证码注册
9.关闭App,重启App
10.执行Scenario: 使用错误手机号登录
11.关闭App,重启App
12.执行Scenario: 使用正确手机号登录
13.卸载并重新安装App
14.执行Feature: Health 测试
15.启动App
16.执行Scenario: 登录
17.关闭App,重启App
18.执行Scenario: 使用错误健康证号码上传
19.关闭App,重启App
20.执行Scenario: 使用正确健康证号码上传
21.关闭App
```

基本上Calabash大的框框就是这样，所以我们在设计编写测试用例时，要按照这个框框来做。考虑什么情况分Feature,什么情况分Scenario，什么情况不能分，需要整体一大串的往下写。

然后是Given、When、Then三个关键字，其实我并没有太搞明白他们之间的区别，似乎只是在概念上做了区分，实际的使用中并没有什么明显的区别，以我暂时的理解就是：在写法上，你用Then和用When都可以执行。只是理解上有区别。

下面是Cucumber给出的定义：

*  Feature（功能）— test suite （测试用例集）
*  Scenario（情景） — test case （测试用例）
*  Given（给定）— setup（创建测试所需环境）
*  When（当）— test（触发被测事件）
*  Then（则）— assert(断言，验证结果)

虽然Then是用来验证结果的，但当When无法执行时，也会报错。我有怀疑过他们的超时时间不同，但经测试发现，他们超时时间也是一样的。这个还待深究……
虽然暂时认为他们在使用上是一样的，但我还是建议你在编写测试用例时，**按照上面的区分进行编写，毕竟测试脚本还是需要有很强可读性的，你可以用对应关键字来区分哪些步骤是需要测试验证的重点。**


## Calabash 预定义的Steps使用

我只能说，Calabash最全的文档，除了安装步骤以外就是这套预定义的Steps了，虽然这个最全的指令也写的不那么友好。 [轻戳跳转到Calabash 预定义Steps](https://github.com/calabash/calabash-android/blob/master/ruby-gem/lib/calabash-android/canned_steps.md)

简单解释一下，看懂一两条基本上其他照着下面的说明看，用起来就没问题了。

	Then /^I enter "([^\"]*)" into input field number (\d+)$/ do |text, index|

以这个Step为例。
我忍不住要吐槽了，首先要承认我正则表达式接触的很少，为了这个我还去恶补了一下，因为第一次看到上面这个我都不知道怎么用，/^是啥？要不要写上去？  那个 do 要不要加上？ 后面|text, index|看着应该是参数名，要不要也上去？ 好烦好烦~

好了吐槽结束，开讲：

正确的使用方法是： `` Then I enter "你好世界" into input field number 1``
其含义是在第一个输入框里输入 你好世界。
这里的1是指当前界面第一个输入框，至于为什么不像是数组一样，是从0开始的问题，我们在下一节讲**自定义Steps**时，会分析这条Steps的源码，到时候你就明白了。

文档给出的写法实际是Step的定义写法，前面Then是关键字，我们不管它，所以 /^ $/  do |text, index| 通通都是定义语法，在使用中都是不用写上去的。这个等我们看到后面自定义Steps时就能理解了。
中间"([^\"]*)"  (\d+) 这两个就很简单了，就是两个正则表达式参数，正则所修饰的就是该参数所能接收的类型。

## 自定义Steps

这一章我们暂只聊Calabash三种自定义Steps中的两种，第三种放到下一章进阶中讲。

### 自定义Steps

Calabash提供的预定义Steps很明显不够用的嘛，那么就需要我们自己进行扩展。怎么扩展呢？

在step_definitions文件夹下新建一个文件：first_steps.rb,然后进行修改：（这里我偷个懒，直接拿Calabash源码用来讲自定义方法）

```
Then /^I enter "([^\"]*)" into input field number (\d+)$/ do |text, index|
  enter_text("android.widget.EditText index:#{index.to_i-1}", text)
end
```

Then是前面的修饰符，这个没啥好说的
/^ $/ 成对出现，用来表示Steps方法名的开始和结尾
中间内容理解为方法名，随便写。可以在任意位置插入参数，参数用正在表达式表示
$/一个空格后 加关键字 do 开始定义方法体
如果有参数便按照|text, index| 格式添加参数，参数数目要和前面对应上，否则会报错
换行后就是真正的方法体，``enter_text()`` 是Calabash基于Ruby编写的库方法。关于这些方法的文档我还没有找到，初期，你可以对照Calabash预定义的Steps源码找到可用的ruby库方法。
最后以end结尾。
steps定义结束。

哦，还要解释一下上一节我们提到的：**为什么这条指令的这里的1是指当前界面第一个输入框，而不像是数组一样，是从0开始的问题。**

看下面的定义 ``enter_text("android.widget.EditText index:#{index.to_i-1}", text)``
**{index.to_i-1}** 这里传参中，做了一个减1的操作，calabash在实际的处理上，还是从0开始的，但是Calabash为了最大程度保证语义的自然程度，做了这样的修改，（在自然语言中，我们从头数的话，说的都是第一个，而不是第零个）这也是我们在自定义指令时可以借鉴的一个地方。

为什么说Calabash学起来很简单呢？因为他的语法简单，你完全可以照猫画虎的学。你可以在这个目录下
`` ./Library/Ruby/Gems/2.0.0/gems/calabash-android-0.9.0/lib/calabash-android/steps``

找到Calabash预定义的Steps源码，通过对比源码来学习Steps的使用和自定义，同时寻找可用的ruby库方法。

可以先看一下enter_text_steps.rb文件下的内容：

```
Then /^I enter "([^\"]*)" as "([^\"]*)"$/ do |text, content_description|
  enter_text("android.widget.EditText {contentDescription LIKE[c] '#{content_description}'}", text)
end

Then /^I enter "([^\"]*)" into "([^\"]*)"$/ do |text, content_description|
  enter_text("android.widget.EditText {contentDescription LIKE[c] '#{content_description}'}", text)
end

Then /^I enter "([^\"]*)" into input field number (\d+)$/ do |text, index|
  enter_text("android.widget.EditText index:#{index.to_i-1}", text)
end

Then /^I enter text "([^\"]*)" into field with id "([^\"]*)"$/ do |text, id|
  enter_text("android.widget.EditText id:'#{id}'", text)
end

Then /^I clear "([^\"]*)"$/ do |identifier|
  clear_text_in("android.widget.EditText marked:'#{identifier}'}")
end

Then /^I clear input field number (\d+)$/ do |index|
  clear_text_in("android.widget.EditText index:#{index.to_i-1}")
end

Then /^I clear input field with id "([^\"]*)"$/ do |id|
  clear_text_in("android.widget.EditText id:'#{id}'")
end
```

大概就长这样，稍微有点编程基础的，照猫画虎没啥问题。

### 封装以定义的Steps

以上面我们看到的health.feature为例，我只写了两个场景，因为细分的边界case会很多，所以实际过程中我可能要写很多个场景，那么每一个场景 进入到健康证上传页面这个步骤都是一模一样的，难道我要复制粘贴这么多下么？ 不不~，一个有追求的程序猿是百分百拒绝复制粘贴的！那么我就要把进入健康证上传页面这些个步骤封装起来变成一个命令，用起来省事，修改也简单！

需要说一下的是，如果你用上面自定义Steps的方法直接在方法体中写自定义Steps的话，是不可行的，就像这样：

```
Then /^I through welcomePages$/ do
	Then I drag from 90:50 to 20:50 moving with 20 steps 
    Then I drag from 90:50 to 20:50 moving with 20 steps 
    Then I drag from 90:50 to 20:50 moving with 20 steps 
end
```

上面的代码是无法执行的。想要封装已定义的Steps你需要这样做：

```
Then /^I through welcomePages with (\d+) steps$/ do |steps|
	steps %{
    	Then I drag from 90:50 to 20:50 moving with #{steps} steps
        Then I drag from 90:50 to 20:50 moving with #{steps} steps
        Then I drag from 90:50 to 20:50 moving with #{steps} steps
	}
end
```

``%{}`` 是ruby中表示多行字符串的格式，一对大括号之间的所有换行符和空格符都会原原本本的输出。

如果需要再``%{}`` 内部使用参数，直接写参数名是不会被识别的，需要使用``#{}``包裹。

在自定义Steps时，你可能会遇到**坑5：Calabash自定义的Steps,执行过程中提示未定义**

## 使用Ruby语法编写带逻辑判断的Steps

上面只是介绍了Steps的自定义方法，简单的对原有库方法或命令进行封装，修改方面名或参数值。扩展的能力有限，Calabash当然不会如此的初级，.rb是Ruby文件，所以这里使用的都是Ruby语法，你也当然可以通过Ruby语法为你的Steps添加各种逻辑。

**Steps的定义不只是已有库方法和以后Steps的封装，他的内部定义同样可以非常丰富。**

例如上一节我们封装的三行左划命令，如果我想增加一个扩展，通过传入的参数来控制左划的次数：

```
Then /^I drag toLeft (\d+) times$/ do |times|
	$num = times.to_i
	while $num > 0 do
		perform_action('drag', 90, 20, 50, 50, 20) 
		$num = $num-1
	end
    puts "循环结束"
end
```
即使你没有接触过Ruby，但是这个代码也应该可以大致看懂。
``$num`` 是Ruby定义变量的方法。
``times.to_i`` 是Ruby提供的类型转换函数

```
while conditional [do]
   code
end
```
是Ruby的循环语法。

前面一章我们还提到除了feature中编写的注释可以在执行中输出以外，你还可以使用ruby的输出语句``puts ""``来输出一些提示性语句。

Ruby的语法真的非常简单，这里再次推荐[Ruby的学习网站](http://www.runoob.com/ruby/ruby-loop.html)。

## 自定义Steps的使用策略

自定义Steps是Calabash测试脚本的基石，就像砖块一样，只有齐整的砖块盖起房子来才会更容易，所以为了保证后期脚本维护的方便，以及让测试脚本的编写变得越来越轻松容易，接纳更多的人员参与，应该建立一套Steps规范，并建立一个Steps文档。

下面是我的建议：

### 与业务完全解耦的自定义Steps：
该种类型应该像Calabash预定义的Steps一样，完全与业务解耦，只是不同类型的动作及检查，开放有更多的参数，有很好的灵活性，复用性很强。同时**每一个新建的Steps都要加入Steps库中，一直累加，禁止或尽量避免对原有Steps进行修改。对于这样的库我们称为：Steps库**
这类型自定义Steps更多的是使用Ruby函数库定义的Step,因为使用Ruby函数库定义的Step需要设计到很多Ruby语言，增加了学习树，所以这类型Steps应该保持更高的稳定性。

**未来的展望：** 中期Calabash测试脚本的开发将分为两个梯队，一个是Steps的开发，一个是测试用例开发，测试用例开发不需要对Calabash有过深入的了解，只需要对照着Steps文档即可编写测试脚本。如果需要新的Steps,就像Steps开发提需求即可。
到后期Steps库逐步完善，可能很久才会有新的Steps需求，那么这个时候Calabash自动化测试脚本就真的是谁都可以编写的了，BDD将不再是乌邦托。

此类型的Steps建议以 `` *_steps.rb`` 格式建立文件。

### 与业务耦合的自定义Steps:
为了保证Steps复用，不写大量重复代码，一些经常被使用、稳定性较高的操作应该封装起来作为一个Steps语句进行使用。
例如每次重新安装App后，都需要左划三次跳过欢迎界面，那么这样的操作，我们应该封装起来作为一个Step。
我们将这样的Steps库称为：Enca库

毕竟与业务耦合，所以变动的可能性非常大，管理难度同样会跟着上升。
同一个Enca Steps可能在很多个场景中使用，如果因某个场景变更，而需要修改这个Steps的话，很容易导致其他场景出错，但像Steps库一样有变更就单纯累加Steps的方式显然也不明智，会导致后期Enca库存在大量以废弃但又不敢删除的Steps。
所以Enca库中Steps的设计原则应该高度耦合于固定的界面，每一个Step都应对应且只对应一个页面的一个操作，即使两个页面的某个操作高度类似，也应该编写两个Steps，保证在某个页面逻辑修改时不会因为这样的耦合关系而影响其他页面。

### 合适的语法糖

自然语言很强的可读性副带的缺点便是记忆与编写速度的下降，过于松散的语法结构将会对脚本编写带来很大的困扰，所以你需要自定义一套符合大多数人习惯的语法糖，从而在保证可读性的同时固化大部分指令格式，提高Steps的记忆和脚本编写速度。可能说的不太明白，举个例子吧

	#我跳过欢迎界面
	Then I skip welcome page
    #我完成登录操作
	Then I have finished the login

上面是两个滥用自然语言特性的自定义操作。如果你的所有测试脚本都是这样写的，虽然读起来没有问题，但写起来就完全离不开文档了。下面看一下较为建议的写法：


	#我跳过欢迎界面
	Then I pass "welcome" scenario
    #我完成登录操作
	Then I pass "login" scenario

两个Steps的实际含义其实仅只是直接通过某种操作，如果我们规范的定义适合自己的语法糖，非常杂乱的脚本将变得很有规律可循，保证可读性的同时你只需记住几个特定的指令，从而提高脚本编写速度。

如果你一直抱怨现在正在使用的某种语言的某个语法多么的没人性，使用体验糟糕透顶！现在好了，做好开发一套属于自己的语言的准备了嘛？

### 严格的文档管理

无论是Stepks库还是Enca库，一定要建立完善的API文档，每一个入库的Steps一定要在文档中记录。添加新的Steps时需要现在Steps文档中查看是否已存在同样的Steps。

## Ruby Query的使用。

编写Calabash测试脚本，最大的难题在于定位到对应的元素。虽然预定的Steps中提供了通过文字、ID、Index的方式定位元素，但真实情况往往更加复杂。单纯通过肉眼的话，文字匹配对于图标无能为力，很多控件没有ID，也有很多控件你并不能很好的判断其类型，和其Index。

元素定位很困难，但我们可以通过Ruby Query等命令帮助我们编写测试用例。

首先第一步在终端中执行命令：``calabash-android console **.apk  ``

接下来会进入calabash命令行，提示符变为了 irb(main):001:0>

如果app未安装，先执行 ``reinstall_apps``
已安装直接执行 ``start_test_server_in_background ``

下面，先执行一个最简单的Query命令 ``query("*")  `` 返回该页面所有view元素

```
irb(main):021:0> query("*")
[
    [ 0] {
                     "class" => "com.android.internal.policy.impl.PhoneWindow$DecorView",
                       "tag" => nil,
               "description" => "com.android.internal.policy.impl.PhoneWindow$DecorView{ebb9e78 V.E..... R....... 0,0-1080,1920}",
                        "id" => nil,
                   "visible" => true,
                      "rect" => {
              "height" => 1920,
               "width" => 1080,
                   "y" => 0,
                   "x" => 0,
            "center_x" => 540,
            "center_y" => 960
        },
                   "enabled" => true,
        "contentDescription" => nil
    },
    [ 1] {
                     "class" => "android.widget.LinearLayout",
                       "tag" => nil,
               "description" => "android.widget.LinearLayout{27f6d6c3 V.E..... ........ 0,0-1080,1920}",
                        "id" => nil,
                   "visible" => true,
                      "rect" => {
              "height" => 1920,
               "width" => 1080,
                   "y" => 0,
                   "x" => 0,
            "center_x" => 540,
            "center_y" => 960
        },
                   "enabled" => true,
        "contentDescription" => nil
    },
    [ 2] {
           
    
    ...
    
    [38] {
                     "class" => "me.ele.crowdsource.components.RedPacketView",
                       "tag" => nil,
               "description" => "me.ele.crowdsource.components.RedPacketView{1df1f92a V.ED..C. ........ 880,1531-1080,1710 #7f0d02e1 app:id/red_packet_view}",
                        "id" => "red_packet_view",
                   "visible" => true,
                      "rect" => {
              "height" => 179,
               "width" => 200,
                   "y" => 1531,
                   "x" => 880,
            "center_x" => 980,
            "center_y" => 1620
        },
                   "enabled" => true,
        "contentDescription" => nil
    }
]
```

返回数据是数组格式。里面的index,class，id都是编写测试用例的重要依据。

查找某一类型的控件。 ``query("android.support.v7.widget.AppCompatEditText")`` 我们查到2个EditText。
```
irb(main):025:0> query("android.support.v7.widget.AppCompatEditText")
[
    [0] {
                     "class" => "android.support.v7.widget.AppCompatEditText",
                       "tag" => nil,
               "description" => "android.support.v7.widget.AppCompatEditText{1006278d VFED..CL ........ 45,0-720,153 #7f0d02cf app:id/phone_verify_sheet_phone_number}",
                        "id" => "phone_verify_sheet_phone_number",
                      "text" => "",
                   "visible" => true,
                      "rect" => {
              "height" => 153,
               "width" => 675,
                   "y" => 666,
                   "x" => 90,
            "center_x" => 427,
            "center_y" => 742
        },
                   "enabled" => true,
        "contentDescription" => nil
    },
    [1] {
                     "class" => "android.support.v7.widget.AppCompatEditText",
                       "tag" => nil,
               "description" => "android.support.v7.widget.AppCompatEditText{1042aa90 VFED..CL ........ 0,155-990,308 #7f0d02d2 app:id/phone_verify_sheet_verifying_code}",
                        "id" => "phone_verify_sheet_verifying_code",
                      "text" => "",
                   "visible" => true,
                      "rect" => {
              "height" => 153,
               "width" => 990,
                   "y" => 821,
                   "x" => 45,
            "center_x" => 540,
            "center_y" => 897
        },
                   "enabled" => true,
        "contentDescription" => nil
    }
]
```

假设这两个输入框都没有id，也没有文字可以匹配，你可以使用它们的下标进行定位。比如我要对下标为1的输入框进行输入，使用上面我们两次提到的Step: ``Then I enter 输入密码 into input field number 2`` 
下标为1的输入框在自然语言中是指第二个输入框，上面我们也分析了这条Step的源码，对传入的参数做了减1操作，所以这里传入2。

当然我们同样可以执行这样的命令

```
查询id等于phone_verify_sheet_verifying_code的ImageView
irb(main):018:0> query("android.widget.ImageView id:'img1'")  

其他一些属性均可
query("* visible:true")  
```

同时你还可以指定返回的结果``query("*", :id)``
```
irb(main):031:0> query("*", :id)
[
    [ 0] nil,
    [ 1] nil,
    [ 2] nil,
    [ 3] "action_bar_root",
    [ 4] "content",
    [ 5] nil,
    [ 6] "login_login_text",
    [ 7] "login_logo",
    [ 8] "login_sheet",
    [ 9] nil,
    [10] "phone_verify_sheet_phone_number",
    [11] "phone_verify_sheet_send_code",
    [12] "phone_verify_sheet_divider",
    [13] "phone_verify_sheet_verifying_code",
    [14] "login_login",
    [15] "login_audio",
    [16] "login_register",
    [17] "statusBarBackground"
]
```

上面我们简单的使用了Quewy查询，更详细的介绍可以参考下面两篇文章：

[https://github.com/calabash/calabash-android/wiki/05-Query-Syntax](https://github.com/calabash/calabash-android/wiki/05-Query-Syntax)
[http://blog.csdn.net/bigconvience/article/details/39182161](http://blog.csdn.net/bigconvience/article/details/39182161)


## View定位技巧

Calabash定位的主要方式有三种：文字、id、index。

文字是最简单的一种定位方式，但也有缺点，那就是文字的非唯一性。
以Step:``Then /^I press "([^\"]*)"$/ do |identifier|`` 为例，当一个界面同时存在两个以上的目标文字时，按照这条命令的定义，Calabash会默认点击第一个符合条件的view。如果你想要可以选择下标的Step，就需要自定义了。
同时，如果有些纯图标的View，文字定位就无用了。

Id虽然是唯一性的，但同样缺陷明显：1.有很多View没有ID,这个我们在上一节Query语句的结果中就可以看到了，这种情况不止存在于根布局，很多需要操作的View都存在这样的问题！
2.如果id的定义并不人性化，那么同样会导致脚本的可读性下降。
3.debug包和混淆过的release包ID往往不同，因为ID名也被混淆了，所以Debug包和relase包无法同用一套脚本。

View下标是指：把当前页面的所有View当做一个数组，每一个View都有一个下标。通常这种排列都有迹可循，比如从上到下，从左到右，从深到浅。如果布局非常复杂，可以借助Query语句查询。

基本原理知道了，从这几个方面找方法就很容易了。

### 肉眼

Calabash最大的魅力在于其自然语言一样的脚本，如果可以，我还是建议尽量使用肉眼识别文字与下标的方式进行编写Calabash脚本，点击登录按钮，在第2个输入框中输入XXX，通过这样的方式保持测试脚本的自然性。毕竟奇奇怪怪的ID和奇奇怪怪的下标，尤其混淆以后的ID，和下标上升到十几位数以后，比如下面这种尴尬：

```
When I press view with id "c2"
Then I enter "1" into input field number 13
```
c2是什么鬼，第13个输入框？？？

但毕竟人力有穷尽，很多view的定位并不是很容易。这只是建议。

### Android uiautomatorviewer

uiautomatorviewer工具位于Android SDK目录下,在终端中切换到Android SDK的目录下，在tools目录下可以看到uiautomatorviewer工具，运行./uiautomatorviewer就可以打开uiautomatorviewer了。

连接手机，打开要查看的页面，点击uiautomatorviewer左上角第二个按钮（Device Screenshot(uiautomator dump)），将会在屏幕上出现该页面，并在右上角的窗口中显示该页面的层级结构，点击页面上的View元素，会在右下角Node Detail窗口中出现该View的基本信息，包括ID，文字等等信息。

![uiautomatorviewer](http://img.blog.csdn.net/20170310132226312?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMDI1NTEyNw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

uiautomatorviewer的优点在于可以在不查看代码的情况下知道View的ID。对比Query查询，其优点在于更加直观，且不需要任何编程基础。毕竟Query是以数组形式返回，如果一个页面View特别多的话，将数组中的View与页面的View做对应是很麻烦的。

**需要提醒的是：这里的index和我们用Query查到的数组下标并不是一回事，所以这里的index不可用**

### Query

uiautomatorviewer虽然可以看到View的ID，但没有办法看到View的下标。Query的查询更加精确一点但不够直观。

所以最好的办法应该是肉眼、uiautomatorviewer、Query三者结合。当然前期可能会显得比较繁琐，但熟练了之后就会很简单了。

### 不定位了，找不到，直接点击屏幕吧

正常来说，所有的View都是可以定位到的，只是难易程度罢了，如果你要操作的View真的特别特别难以定位，直接点击屏幕也是办法。

调用下面这条预定义Step即可

```
Then /^I click on screen (\d+)% from the left and (\d+)% from the top$/ do |x, y|     

use:
Then I click on screen 23% from the left and 34% from the top     
```
这里非常非常要注意的是，这条命令后面的``%``不可以省略，他是按照当前屏幕的百分比位置来点击的。

但无论通过uiautomatorviewer还是Query命令，你都只能查到View的像素坐标，暂时还没有发现好的工具来查询百分比坐标。不过你可以使用像素坐标除以当前手机分辨率来计算百分比坐标。
看到这里，你也许会想，我根本不需要手动计算，直接把计算写在脚本里不就好了嘛：

```
Then /^I click login button $/ do 
  %{
  	Then I click on screen 1323 from the left and 568 from the top for 1920x1080
  }
end

Then /^I click on screen (\d+) from the left and (\d+) from the top for 1920x1080$/ do |x, y|
  perform_action('click_on_screen', x/1080, y/1920)
end
```

但我并不建议你这样去做，上面的自定义Step只适用于1920x1080的手机，在实际测试中，一定会涉及到替换不同手机跑case的，换一个手机就修改一下脚本明显不适用，尤其发展到云测阶段。这也是为什么预定义的Steps只提供了百分比坐标的点击事件。


## 总结

基本上了解上面的内容，就已经可以开始编写Calabash测试脚本进行测试了。当然Calabash还不止这些，我们将在下一章进阶中继续为你介绍Calabash使用技巧：

- 在自定义的Steps中使用Query语句。
- 自定义Steps支持环境变量扩展。
- Hooks。
- Calabash源码修改与扩展。

</br>
 
------

[《Calabash探索1-Run Calabash》](http://lrd.ele.me/2017/03/20/Calabash%E6%8E%A2%E7%B4%A21-Run%20Calabash/)

[《Calabash探索2-Calabash用法详解》]

[《Calabash探索3-Calabash进阶》](http://lrd.ele.me/2017/03/17/Calabash%E6%8E%A2%E7%B4%A23-Calabash%E8%BF%9B%E9%98%B6/)

[《Calabash探索4-Calabash踩坑总结》](https://lizhaoxuan.github.io/2017/03/16/Calabash探索4-Calabash踩坑总结/)










