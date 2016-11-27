# 记录一次安装OpenGrop的心酸历程

> 欢迎转载，转载请注明出处！

> Html版本: [recording\_opengork\_install](http://kevins.pro/blog/recording_opengrok_install/)

> MarkDown版本: [recording\_opengork\_install.md](http://github.com/KevinsBobo/KevinsBobo.github.io/blob/master/article/recording_opengrok_install.md)

> [Follow me on GitHub ^_^](http://github.com/KevinsBobo/)

## 写在前面

了解到OpenGrok这个辅助阅读GitHub上源码的神器后就迫不及待的想要安装，结果走了一个很心酸的安装过程。特此记录，顺便提醒其他想要尝试的朋友这里面隐藏的坑。

另外，由于我不懂Java的运行环境什么的，所以才会踩这么多坑。

没时间看我啰嗦的朋友我直接告诉重点吧：**安装 Latest release 版！**

最终环境配置：`Tomcat Version 7.0.73, JDK Oracl jdk 8, OS Ubuntu 16.04 amd64, Kernel 4.4.0-47`

## 安装过程参考：

http://opengrok.github.io/OpenGrok/ - （官方主页）

https://github.com/OpenGrok/OpenGrok/releases - （下载地址）

https://github.com/OpenGrok/OpenGrok/wiki/How-to-install-OpenGrok - （官方英文安装指导手册）

https://github.com/crazygit/temp/blob/master/setup\_opengrok.md - （中文安装方法）

http://blog.csdn.net/weihan1314/article/details/8944291 - （中文安装方法）

注：建立一个opengrok用户的这个选项不是必需的，跟据个人情况选择；Tomcat和OpenGrok安装位置不限，只要当前用户有读写权限就行。

---

## 心酸经历：

按照上面的指导下载opengrok-0.13-rc4（Pre-release版），安装好了openJDK-8，Tomcat7，并将OpenGrok中的/lib/source.war文件移到Tomcat的webapps目录下，然后执行启动，浏览器页面报错（当时的错误提升没有记录，不过这个提示不是关键）。

跟据提示的错误找到了Stack Overflow上的一个回答，说将JDK更换成Oracl的可能会好，于是尝试更换。

```bash
# 添加ppa源
$ sudo add-apt-repository ppa:webupd8team/java
$ sudo apt-get update
$ sudo apt-get install oracle-java8-installer
# 设置系统jdk
$ sudo update-java-alternatives -s java-8-oracle
```
然后再尝试，还是错误，于是查看错误日志，发现了提示`Tomcat webapps/source/WEB-INF/web.xml`中的`configuration.xml`文件找不到，可是按照上面的安装步骤这个文件的地址要到下一步才配置，而且这一步的时候OpenGrok已经可以运行了。

然而并没有其他可用的错误提示，无奈只能硬着头皮往下进行。配置完`configuration.xml`地址并创建索引后，重新尝试，还是错误。不过这次错误日志提供了下一条有用的信息`'<>' operator is not allowed for source level below 1.7`，这个说这个项目是要用jdk-7运行的，可是官方安装手册中明确要求的环境要是jdk8呀！无奈找不到更多的信息，只能更换jdk

```bash
sudo update-java-alternatives -s java-7-oracle
sudo update-java-alternatives -s java-7-oracle
```

在安装jdk-7的时候由于GreatWall的原因，下载速度极慢，于是从Oracle官网手动下载jdk包放入`/var/cache/orcale-jdk7-installer/`文件夹下，顺利完成安装。

修改~/.bashrc，在文件尾加入以下配置
```bash
JAVA_HOME=/home/weblogic/jdk1.7.0_72
JRE_HOME=/home/weblogic/jdk1.7.0_72
PATH=$JAVA_HOME/bin:$PATH
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME
export PATH
export CLASSPATH
```

这次之后再进行启动，发现`/source`即OpenGrok根本无法启动！
这次是错误提示是`Context [/source]startup failed due to previous errors`

看到其他博客说引起这个错误的原因很多，需要让Tomcat的log信息更精确些，于是在Tomcat安装目录下创建`common/classes/log4j.properties`文件，文件内容配置如下：
```
log4j.rootLogger=info,Console,R
log4j.appender.Console=org.apache.log4j.ConsoleAppender
log4j.appender.Console.layout=org.apache.log4j.PatternLayout
#log4j.appender.Console.layout.ConversionPattern=%d [%t] %-5p %c - %m%n
log4j.appender.Console.layout.ConversionPattern=%d{yy-MM-dd HH:mm:ss} %5p %c{1}:%L - %m%n

log4j.appender.R=org.apache.log4j.DailyRollingFileAppender
log4j.appender.R.File=${catalina.home}/logs/tomcat.log
log4j.appender.R.layout=org.apache.log4j.PatternLayout
log4j.appender.R.layout.ConversionPattern=%d{yyyy.MM.dd HH:mm:ss} %5p %c{1}(%L):? %m%n

log4j.logger.org.apache=info,R
log4j.logger.org.apache.catalina.core.ContainerBase.[Catalina].[localhost]=DEBUG, R
log4j.logger.org.apache.catalina.core=info,R
log4j.logger.org.apache.catalina.session=info,R
```

提示：此时会产生很多的记录日志，对硬盘空间造成影响，如无必要，解决问题后移除上面的文件，并重启Tomcat。

这时候在错误日志里发现了问题所在`Unsupported major.minor version 52.0`，意思是`major.minor`这个包不支持，搜索发现，这个包的52.0版本是用jkd8编译的，51.0才是用jdk7编译的！

What?!上面告诉我整个项目是用jdk7编译的，现在这里告诉我这个项目里引入的一个包却是用jdk8编译的！这怎么运行啊！

此时已经凌晨01：30了，宿舍断电，笔记本已经没电了，只能无奈地暂停。几分钟之后想到了有可能是
**我下载的安装包有问题！**
于是就有了两种方案，一种是自己编译（对我来说难度比较大），另一种是找早期的安装包进行尝试（适合我）！

于是早上进入官方下载目录后发现下面有个`Latest release 0.12.1.6 (stable)`版！惊喜！

然后将电脑配置更换回jdk8，安装0.12.1.6版，启动。此时我以不想用言语表达我喜悦并悲伤的心情了，一切尽在下图中：

<img src="http://kevins.pro/blog/recording_opengrok_install/opengrok_install_success.png" width="100%"/>

虽然折腾大，踩了好多坑，但是收获了OpenGrok这个神器，一切都值！
