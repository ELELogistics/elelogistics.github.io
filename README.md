## PR提交

1. fork 项目
2. clone 项目
   ```
   git clone git@github.com:xxxxx/elelogistics.github.io.git
   ```
3. 切换到 blog 分支
   ```
   git checkout blog
   ```	
3. 进入文件夹
   ```
 	cd elelogistics.github.io/source/_posts/
   ```
4. 新建 博客标题.md 文件
5. 博客格式
 
 	```
    ---
    title: 这里是博客标题，例如:如何写好一篇博客
    date: 这里是博客时间,例如：2016-07-13 15:54:58
    author : 这里是博客作者，例如:james
    tags: 这里是技术类型，例如:JAVA
    ---
    正文 xxxxx
  	图片链接为:完整链接，可以把图片托管在七牛这样的第三方平台
    ```
6. 提交commit,将提交推送到blog分支
	```
    git add -A
    git commit -m "提交博客"
    git push
    ```
7. 发送PR
 	![](http://7o4zmy.com1.z0.glb.clouddn.com/QQ20161219-110638.png)
	切记，一定要发给主项目的blog分支
	![](http://7o4zmy.com1.z0.glb.clouddn.com/QQ20161219-110749.png)

8. OVER,等待合并
