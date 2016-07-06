title: 借助Packet Capture 实现无Root抓请求
date: 2016-05-21 10:49:29
tags: Android
---
>  背景是这样的，有一天运营的同事反馈达达可以抓到我们的订单,简单查看了一下流程，发现达达先会下载一个插件，安装后，会提示信任一个vpn证书，然后就可以获取到我们的订单。

   上一次达达商家版通过Android的Accessibility api监听我们应用的布局变化，获取了每个view中的数据，拼接成了他们自己的订单，但是这个api随着napos的升级被堵死了。这次他们换了一个思路，通过建立虚拟vpn的方式充当中间人，攻击者与通讯的两端分别创建独立的联系，并交换其所收到的数据，使通讯的两端认为他们正在通过一个私密的连接与对方直接对话，但事实上整个会话都被攻击者完全控制。在中间人攻击中，攻击者可以拦截通讯双方的通话。大概理解了这次抓单的思路，我们就开始顺着packet capture的线索开始摸索达达的整个方案。


首先我们查到了packet capture这个软件，　
	Packet Capture免root抓包安卓版是一款数据包捕获/SSL网络流量嗅探的应用程序。
			特点：捕获网络数据包，并记录它们使用中间人技术对SSL解密,无须root权限
	这个软件使用了Android提供的VpnService api,实现了中间人攻击，完全符合达达的需求，那么我们就以这个为突破口，下载了最新的达达，发现在打开抓取饿了么和美团的订单的时候，他需要下载http://7xl8yn.dl1.z0.glb.clouddn.com/pkg-new.signed.apk 这个软件，下载反编译之，发现这个软件和packet capture 代码完全一样，但是我们在使用这个软件的时候，发现界面是不一样的，再比对，发现达达在这个软件外面包了一层壳，加了com.dada.provider 这个包。
    
![这里写图片描述](http://img.blog.csdn.net/20160519214735173)
![这里写图片描述](http://img.blog.csdn.net/20160519214752794)
反复查看这几个类的作用，发现
![这里写图片描述](http://img.blog.csdn.net/20160519214825362)

达达只是通过ContentProvider和intent 来实现了控制vpn开始关闭等功能，而vpn本身的实现完全依赖packet capture这个软件，看到这里我好像明白了什么。达达并没有自己实现高打上了的中间人攻击，他的刀，是别人的刀，那就是packet capture。但是达达是如何从packet capture 中获取数据的尼？要解开这个谜团还需要操开达达的包，哎呦，capture，还有mockhttp，日了明目张胆的出现了Elehttp和HttpDecoders$EleDecoders.java，当然美团也躺枪了。
![这里写图片描述](http://img.blog.csdn.net/20160519214935113)
首先我们找到了设置页面的代码，验证了达达在packet capture中的代码只是为了控制vpn的开关，方便他控制抓单。
![这里写图片描述](http://img.blog.csdn.net/20160519214959638)
![这里写图片描述](http://img.blog.csdn.net/20160519215019067)
![这里写图片描述](http://img.blog.csdn.net/20160519215034623)
通过代码可以清晰的看到，在打开监控饿了么美团的开关之后，达达通过contentProvide来检查socket capture 有没有打开，通过com.dada.walkthrough的start,stop 来开关vpn。
很好，现在我们确定了达达的刷单思路，那下面就开始找他如何从socket capture软件中获取所有请求的数据。很快我们从HttpDecoders$EleDecoders中找到了
![这里写图片描述](http://img.blog.csdn.net/20160519215102396)
很明显这里已经获取到了订单数据，正在解析。那么继续往上找，看什么地方调用了这个方法，顺藤摸瓜，找到了这个a方法，传入的参数是一个File，这让我们联想到达达是从文件中读取的数据，还与napos.order.getProcessedOrders 比对了method。
![这里写图片描述](http://img.blog.csdn.net/20160519215151272)
很快我们又发现了很重要的一个类HttpCaptureTool,这里面在使用contentprovide访问一个数据库，而app.greyshirts.sslcapture.captureprovider 正是packet capture软件的包名。 
![这里写图片描述](http://img.blog.csdn.net/20160519215531995)
到了这里，我立马root了手机，查看了packet capture应用data->databases 中的数据库文件，哈哈，果然在这里。
![这里写图片描述](http://img.blog.csdn.net/20160519215619939)
这和达达代码中的查询字段完全吻合
![这里写图片描述](http://img.blog.csdn.net/20160519215654715)
这个数据库是socket capture存储请求的，里面有
capture_time:抓取时间
capture_file_name:文件存放地址（/data/data/app.greyshirts.sslcapture/files/capture-1727863624.dat）
capture_server_ip:ip地址
capture_app_name_main:应用名称
capture_pkg_name_all：应用包名（达达就是通过包名来指定抓竞品的数据的）
接下来没有悬念，files文件夹下面的dat文件就是每个请求的数据包
![这里写图片描述](http://img.blog.csdn.net/20160519215731013)
到此为止，我们已经探明了达达的抓单思路，他通过在packet capture的外面包一层壳来灵活的开关使用packet capture作为中间人截获手机的流量，因为我们的https没有校验证书合法性，所以很轻易的就被解密为明文，然后充分利用packet capture每次抓到请求都存到文件中的特性，去files中查找对应包名的文件，对应的sql语句如下：

```
Cursor localCursor = d.getContentResolver().query(localUri, null, null, null, "captureset_start_time DESC limit 1");
Cursor localCursor = localContentResolver2.query(localUri, null, "capture_pkg_name_all=? and capture_server_port=? and capture_set_id=?", arrayOfString2, " _id desc limit 30");
```
在没有理清达达这次的刷单思路之前，我们没有想出好的应对方式，现在明白了整个过程，那么就可以兵来将挡，水来土掩。

	建议一：武功再高，也怕菜刀，达达手拿菜刀，有点吓人，但是回过神来，我们首先可以断他的刀。那么我们首先要从socket capture 的中间人攻击入手，防止中间人攻击网上的思路很多，这里不作重复。
	
	建议二：抢过他的菜刀，达达在socket capture上开的接口，我们可以调用，他能开，我们也能关，监听达达的start action，你开我就关，好不热闹。
	
	张小龙说：对产品人来说,善良比聪明更重要。达达这样不择手段的获取竞品app信息已经丧失了应有的理性，让我想起了去年美团商家版kill竞品进程的事情,达达和美团也就半斤八两。