---
title: 自动化之旅--Appium
date: 2017-02-17 15:50:58
author : Mio4kon
tags: 自动化测试
---


为了避免每次上线前重复的人工回归测试,保证每次上线的版本不会引起核心业务的不稳定,所以急需自动化测试来保证业务的稳定性.经过调研我尝试使用Appium进行自动化测试,原因是功能强大,跨平台而且社区也很活跃.


# 主流框架对比

![-w682](http://mio4kon.qiniudn.com/14869609987714.jpg?imageView2/1/w/900/h/440|imageMogr2)


# Appium优点

* 开源
* 跨架构:Native App、Hybird App、Web App
* 跨设备:Android、iOS、Firefox OS
* 不依赖源码
* 使用任何 WebDriver 兼容的语言来编写测试用例。比如 Java， Objective-C， JavaScript with Node.js (in both callback and yield-based flavours)， PHP， Python， Ruby， C#， Clojure， 或者 Perl.
* 不需要重新编译APP

如果有不清楚WebDriver的小伙伴马上在Appium架构介绍中会有说明.

# Appium理念

1. 你无需为了自动化，而重新编译或者修改你的应用。
2. 你不必局限于某种语言或者框架来写和运行测试脚本。
3. 一个移动自动化的框架不应该在接口上重复造轮子。（移动自动化的接口应该统一）
4. 无论是精神上，还是名义上，都必须开源。

# Appium架构

![-w664](http://mio4kon.qiniudn.com/14869635664446.jpg?imageView2/1/w/900/h/440|imageMogr2)

**iOS:** 苹果的UIAutomation  
**Android 4.2+:** Google的UiAutomator  
**Android 2.3+:** Google's Instrumentation. (由单独的项目Selendroid提供支持 )   
 
**Appium 1.6版本以上增加了`UiAutomator2`**

为了满足上面跨平台,把这些三方框架封装成一套API —— WebDriver Api(客户端到服务端的协议)

事实上 WebDriver 已经成为 web 浏览器自动化的标准，也成了 W3C 的标准 —— [W3C Working Draft](https://w3c.github.io/webdriver/webdriver-spec.html),所以Appium在原有基础上扩充了移动自动化相关的API.

投资 WebDriver 意味着你可以押宝在一个已经成为标准的独立，自由和开放的协议。你不会被任何专利限制。

**核心架构**: Appium使用C/S架构,运行时候Service端会监听Client端发送的命令,接着在移动设备上执行这些命令，然后将执行结果放在 HTTP 响应中返还给客户端.

基于这架构可以做什么? 

* 可以用任何实现了该客户端的语言来写测试代码
* 可以把服务端放在不同的机器上
* 可以只写测试代码,然后利用类似 [saucelabs](https://saucelabs.com/) 云服务来解释命令.

下图解释了云服务的具体作用:


![](http://mio4kon.qiniudn.com/14872240688085.jpg)



# Appium 使用

## 服务端

* 安装Appium服务器 

		npm install -g appium
		npm install -g appium-doctor
		appium-doctor


其中`appium-doctor`用来检查电脑是否缺少相关依赖.当所有都是对勾表示Appium环境配置完毕,如下:

![](http://mio4kon.qiniudn.com/14872367178116.jpg?imageView2/1/w/900/h/440|imageMogr2)



* 开启appium服务器:

		appium --address 127.0.0.1 --port 4723 --log "/Users/mio4kon/Desktop/
		appium.log" --log-timestamp --local-timezone --session-override


## 客户端

再次强调 `Appium` 支持各种语言,这里我选择JAVA.如果觉得JAVA语法不够简洁或者不熟悉,可以使用你所熟悉的语言.

### 创建 MAVEN/Gradle 工程:

创建工程,并加入下面依赖:

```xml
        <dependency>
            <groupId>io.appium</groupId>
            <artifactId>java-client</artifactId>
            <version>5.0.0-BETA2</version>
            <exclusions>
                <exclusion>
                    <groupId>org.seleniumhq.selenium</groupId>
                    <artifactId>selenium-java</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-java</artifactId>
            <version>3.0.1</version>
        </dependency>
```

这样Appium 客户端的依赖就引入成功了.

### Capabiltiy配置:

每个Test的基类中定义`setUp`方法,并设置`Capabiltiy`以前其他的初始化操作:

```java
DesiredCapabilities capabilities = new DesiredCapabilities ();
capabilities.setCapability (MobileCapabilityType.DEVICE_NAME, deviceName);
capabilities.setCapability (MobileCapabilityType.PLATFORM_NAME, platformName);
capabilities.setCapability (MobileCapabilityType.PLATFORM_VERSION, platformVersion);
capabilities.setCapability (MobileCapabilityType.APP, apkPath);
capabilities.setCapability (AndroidMobileCapabilityType.APP_PACKAGE, appPackage);
capabilities.setCapability (AndroidMobileCapabilityType.APP_ACTIVITY, appActivity);
capabilities.setCapability (MobileCapabilityType.AUTOMATION_NAME, AutomationName.ANDROID_UIAUTOMATOR2);

```
这里我使用的是 [TestNG](http://testng.org/doc/index.html) 中的`Parameters`注释来配置参数.

![](http://mio4kon.qiniudn.com/14872464979722.jpg)

TestNG 又是什么鬼? 简单来说TestNG是java的一个测试框架,类似JUnit但功能更加强大,使用也方便.

### TestNG

利用TestNg的一些注释,做准备化和收尾操作.

![](http://mio4kon.qiniudn.com/14872471690847.jpg)

上面的`setUp`方法我就使用了`BeforeClass`和`Parameters`这两种注释.

```java
    @BeforeClass
    @Parameters({"driverName", "url", "deviceName", "platformName", "platformVersion", "apkPath", "appPackage", "appActivity"})
    public void setUp(String driverName, String url, String deviceName,
                      String platformName, String platformVersion, String apkPath,
                      String appPackage, String appActivity) throws Exception {
        log.i (TAG, "BeforeClass");
        driver = setRemoteDriver (driverName, url, deviceName, platformName, platformVersion, apkPath, appPackage, appActivity);
        actions = ElementActions.getInstance ().init (driver);
        assertActions = actions.getAssertActions ();
        Screenshot.getInstance ().init (driver);
        prepare ();
    }
```
同样在在`AfterClass`后,进行了`driver`的退出.

正常情况下我们写测试用例是下面这种情况:

一个`TestClass`中包含多个`TestMethod`.如果每个`TestMethod`都相互独立,需要重新运行APP显然非常耗时,所以这里我将每一个`TestClass`相互独立,而其中的`TestMethod`又相互依赖,执行顺序通过`TestNG`的XML来控制.如下:

```xml
   <test name="XX测试">
        <classes>
            <class name="XXTest">
                <methods>
                    <include name="testAAA"/>
                    <include name="testBBB"/>
                    <include name="testCCC"/>
                </methods>
            </class>
        </classes>
    </test>
```

但是有些情况如果需要`TestMethod`也相互独立的话,可以利用`AfterMethod`注释.

```java
    @AfterMethod
    public void afterMethod() {
        driver.resetApp ();
    }
```


# 编写用例

## 查找元素

### 定位方式

查找元素可以通过很多方法(可以通过`UIAutomatorViewer`获取页面的id,name等信息):

* id
* name
* className 
* xpath
* uiautomator

### 高级定位

* 用 `xpath`查找登录按钮: 

 		by.xpath ("//button[@name='login']")

[xpath实例教程](http://www.zvon.org/comp/r/tut-XPath_1.html#Pages~XPath_as_filesystem_addressing)

* 用`uiautomator`的API滚动查找:

```java
	String rule = "new UiScrollable(new UiSelector().scrollable(true)).scrollIntoView(new UiSelector().textContains(\"" + locator.value + "\"))"
	WebElement cl = driver.findElementByAndroidUIAutomator(rule));
```
 其实就是用如果用uiautomator来写:
 
```java
new UiScrollable(new UiSelector().scrollable(true)).scrollIntoView(new UiSelector().textContains(value));
```
[uiautomator API](https://developer.android.com/reference/android/support/test/uiautomator/package-summary.html)


		### 元素管理

为了让测试用例能够方便复用,简化写用例的时间,必要的封装是不可少的.
利用 **Page Object** 模式将页面上的元素进行封装.这样所有 Test 只需要简单的控制页面元素即可.

* 可以使用yaml文件进行管理页面元素:

![](http://mio4kon.qiniudn.com/14869709464700.jpg?imageView2/1/w/900/h/440|imageMogr2)

然后在BasePage中将yaml解析封装成`Locator`对象.并保存到集合.  
这样所有的PageObject都可以通过下面方法来定位元素.


```java
protected Locator getLocator(String locatorName) {
        checkNotNull (locatorName);
        Page page = getPage ();
        List<Locator> locators = page.locators;
        for(Locator locator : locators) {
            if (locatorName.equals (locator.name)) {
                return locator;
            }
        }
        return null;
    }
    
public Locator 请输入手机号 = getLocator ("请输入手机号");
public Locator 请输入验证码 = getLocator ("请输入验证码");
public Locator 发送验证码 = getLocator ("发送验证码");

```

~~TODO: 用工具生成对应page类~~  
之所这么做是可以在用的时候直接通过IDE提示有哪些控件,但是写上述代码缺都是无聊的操作,所有我用了 [freemarker](http://freemarker.org/) 自动解析yaml来生成对应Page类,如下:

```java
   		Map<String, Object> input = new HashMap<String, Object> ();
        input.put ("pageName", page.pageName);
        List<LocatorObject> genLocators = new ArrayList<> ();
        List<Locator> locators = page.locators;
        for(int i = 0; i < locators.size (); i++) {
            System.out.println ("生成Locator:" + locators.get (i).name);
            genLocators.add (new LocatorObject (locators.get (i).name, locators.get (i).name));
        }
        input.put ("locators", genLocators);

        Template template = cfg.getTemplate ("page-temp.ftl");

        Writer fileWriter = new FileWriter (new File (path, page.pageName + ".java"));
        try {
            template.process (input, fileWriter);
        } finally {
            fileWriter.close ();
        }
```

如果不会用 `freemarker`的可以搜索下相关资料.

## 元素交互

将交互事件,封装在`ElementActions`中.使用起来非常简单:

```java
actions.text (loginPage.请输入手机号, phone);
actions.click (loginPage.发送验证码);
actions.text (loginPage.请输入验证码, pwd);
actions.click (loginPage.登录); 
```

常用的交互事件:单击,连击,上下左右滑动,后退,输入文本等等.

上面元素定位其实只是封装Locator,并没有真正的查找元素,查找元素实际上是在`ElementActions`中处理的.由于应用经常处理网络请求,控件有时候需要过段时间才可见,所以在查找控件时候需要给予一定的时间.

```java
        WebElement element;
        try {
            element = (new WebDriverWait (mDriver, locator.timeOutInSeconds)).until (
                    new ExpectedCondition<WebElement> () {

                        @Override
                        public WebElement apply(WebDriver driver) {
                            List<WebElement> elements = getElement (locator);
                            if (elements.size () != 0) {
                                return elements.get (0);
                            }
                            return null;
                        }
                    });

        } catch (NoSuchElementException | TimeoutException e) {
            log.e (TAG, "超时[%4$d秒],找不到元素:[%1$s] , [By.%2$s : %3$s]", locator.name, locator.type, locator.value, locator.timeOutInSeconds);
            throw e;
        }
```

## 断言

同样为了使用方便讲常用的断言封装,如Toast验证等.

```java
    public void validatesToast(final String msg) {
        checkNotNull (msg);
        final WebDriverWait wait = new WebDriverWait (mDriver, 10);
        assertNotNull (wait.until (ExpectedConditions
                                           .presenceOfElementLocated (By.xpath (String.format ("//*[@text=\'%s\']", msg)))));
    }

```

## 测试报告

测试报告可以直观的展示测试的成功率,截图等信息.这里选择[extentreports](http://extentreports.com/)做为测试报告的框架.

最终效果:

![-w1083](http://mio4kon.qiniudn.com/14869768987854.jpg?imageView2/1/w/900/h/440|imageMogr2)



## 踩坑

* `findElementByName`无效.

> Searching by name was deprecated over a year ago and removed from 1.5. In general, searching by accessibility id is better for a variety of reasons.
	
如上`findElementByName`这个方法从`Appium 1.5`之后删除了,但是API不经能找到并且也没提示过时.这不坑爹嘛.后来使用下面的代码才解决用name查找元素的方法.

```java
 String query = "new UiSelector().textContains" + "(\"" + locator.value + "\")";
 webElements = mDriver.findElementsByAndroidUIAutomator (query);
```


* 据说`Appium 1.6.3`可以查找 Toast 的信息了.然后屁颠屁颠的跑去试了下网上的例子发现不好使啊.一度以为是Client版本的问题.搞了半天才发现需要加下面的代码:

```java
capabilities.setCapability (MobileCapabilityType.AUTOMATION_NAME, AutomationName.ANDROID_UIAUTOMATOR2);
```
(感觉现在的Appium文档不全而且有点乱

* 无意中发现测试的时候高德地图弹Toast报错.然后直接编译安装却不保存.猜测是不是在安装过程中Appium改了啥.看了下Service日志,竟然在安装的时候重新签名...

```
App not signed with debug cert.
2017-02-13 18:17:19:848 - info: [debug] [ADB] Resigning apk.
2017-02-13 18:17:23:938 - info: [debug] [ADB] Zip-aligning 'app-debug.apk'
2017-02-13 18:17:23:958 - info: [ADB] Checking whether zipalign is present
2017-02-13 18:17:23:964 - info: [ADB] Using zipalign from /Users/mio4kon/Library/Android/sdk/build-tools/25.0.2/zipalign
2017-02-13 18:17:23:968 - info: [debug] [ADB] Zip-aligning apk.
2017-02-13 18:17:24:104 - info: [AndroidDriver] Remote apk path is /data/local/tmp/463eb03788048b4a1dacfe28545ee76e.apk
```

解决方法:  

`capabilities.setCapability (AndroidMobileCapabilityType.NO_SIGN, true);`

