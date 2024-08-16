
1、首先安装Xcode（这个Xcode我是直接使用mac的App store安装的）

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/10fb0ceaf20117e08398feaee8b66488.png)

2、安装freetype与ccache
安装命令如下：
brew install freetype ccache

如果没有安装brew（一个包管理工具，类似Linux的yum，apt-get），请自行安装

3、安装mercurial（分布式版本管理工具）
安装命令如下：
brew install mercurial

4、拉取openJDK10（最好有vpn，否则基本上下不成功）
hg clone http://hg.openjdk.java.net/jdk10/master openJdk10

最后面的openJdk10表示下载下来后的目录名

5、配置参数

bash configure --with-debug-level=slowdebug --enable-dtrace --with-jvm-variants=server --with-target-bits=64 --enable-ccache --with-num-cores=8 --with-memory-size=8000 --disable-warnings-as-errors --with-freetype=/usr/local/Cellar/freetype/2.10.1


以下是相关选项说明：

--with-debug-level=slowdebug 启用slowdebug级别调试
--enable-dtrace 启用dtrace
--with-jvm-variants=server 编译server类型JVM
--with-target-bits=64 指定JVM为64位
--enable-ccache 启用ccache，加快编译
--with-num-cores=8 编译使用CPU核心数
--with-memory-size=8000 编译使用内存
--disable-warnings-as-errors 忽略警告, mac 使用 xcode 编译, 官方要求加上这个参数.
--with-freetype 设置freetype的路径, 直接在cmd键入cd /usr/local/Cellar/freetype/ 然后把对应的版本号填写正确就行

如果配置参数的时候出现以下提示

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/94ba83e163dd5b218d1a3cd778ef2be7.png)

那么按照上面的提示，安装JDK9或者JDK10
本人试过JDK10，但是还是会出现错误，会提示javah找不到的错误，后来卸载了JDK10，安装了JDK9作为Boot JDK才成功

参数配置成功的界面

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/27f12c03ddba86058ce066596b97767f.png)

7、编译

命令：make images
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/c254bd9464e3d1f579b8165fbe866cb8.png)
构建的过程需要一段时间，耐心等待
最后出现类似
Finished building target 'images' in configuration 'macosx-x86_64-normal-server-slowdebug
的语句基本上就妥了

8、测试

构建完后，在我们拉取代码的根目录下，也就是openJdk10下的build/macosx-x86_64-normal-server-slowdebug/jdk/bin可以找到java这个执行文件

然后java -version（就像我们安装了Oracle的jdk一样）就可以看到openJdk的版本的信息

9、配置环境

下载[clion](https://www.jetbrains.com/clion/)

File -> open -> 选择刚才编译好的目录，我们选到 src目录，也就是openJdk10/src

然后点击Add Configuration...(如下图)

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/5c5f48e1692789dd566920144cee5088.png)

点击弹框中的加号，选择Custom build applition
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/667c04bb9a8a1256871e0003a60d81e7.png)

默认情况下Target下拉啥都没有，这个时候可以点击Configure Custom Build Targets,如下图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4964108da8f0ec338ef165309c965ead.png)

进入Configure Custom Build Targets的界面如下
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/50d13f0853e048e854767ff9177591cf.png)

然后随便输入一个名字即可，然后回到Custom build applition的界面，配置Executable，这个Executable就是我们之前测试使用的那个java可执行文件，找到它配置上去即可（注意在before launch那一栏的Build一定要去掉，否则不能执行）
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4253a4d1c4e6a8d478de99f6de5ed480.png)
10、打断点

command + shift + O 找到thread.cpp文件，在Threads::create_vm函数里打一个断点，然后debug
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/e1c2b9550558c965acba1ac7450e5a28.png)

参考文档：
https://hunterzhao.io/post/2018/01/29/compile-openjdk10-source-code-on-mac/



