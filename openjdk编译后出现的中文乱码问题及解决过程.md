# 概述

项目的打包是通过Jenkins进行的，
由于对Jenkins进行了一次升级和数据迁移之后，
凡是在Java类中有中文字符输出的地方，都变为乱码。
乱码问题是发生在升级迁移之后，
后来通过在jvm的启动参数追加-Dfile.encoding解决该问题，下面是问题的详细分析过程：

# 发生问题之前的打包环境
```
[root@server ~]# java -version
java -version
java version "1.8.0_152"
Java(TM) SE Runtime Environment (build 1.8.0_152-b16)
Java HotSpot(TM) 64-Bit Server VM (build 25.152-b16, mixed mode)
```

Jenkins版本为2.53，启动方式为
```
/etc/init.d/jenkins start
```

# 发生问题时的打包环境

```
jenkins@84d299b30e0d:/$ java -version
java -version
openjdk version "1.8.0_171"
OpenJDK Runtime Environment (build 1.8.0_171-8u171-b11-1~deb9u1-b11)
OpenJDK 64-Bit Server VM (build 25.171-b11, mixed mode)
```

Jenkins版本2.107.3，启动方式为
```
docker run -d --name myjenkins -p 80:8080 -p 50000:50000 -v /some/path/jenkins_data:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock jenkins
```

## 部署的服务器信息

``` 
[sshuser@server ~]$ rpm -q centos-release
centos-release-7-5.1804.el7.centos.x86_64
```

## 部署的服务器的编码信息
```java
[sshuser@server ~]$ locale
LANG=en_US.UTF-8
LC_CTYPE="en_US.UTF-8"
LC_NUMERIC=zh_CN.UTF-8
LC_TIME=zh_CN.UTF-8
LC_COLLATE="en_US.UTF-8"
LC_MONETARY=zh_CN.UTF-8
LC_MESSAGES="en_US.UTF-8"
LC_PAPER=zh_CN.UTF-8
LC_NAME="en_US.UTF-8"
LC_ADDRESS="en_US.UTF-8"
LC_TELEPHONE="en_US.UTF-8"
LC_MEASUREMENT=zh_CN.UTF-8
LC_IDENTIFICATION="en_US.UTF-8"
LC_ALL=
```

## 部署的服务器上Java的编码信息
```
[root@srv ~]# java -XshowSettings:all -version
VM settings:
    Max. Heap Size (Estimated): 1.70G
    Ergonomics Machine Class: server
    Using VM: OpenJDK 64-Bit Server VM
Property settings:
    awt.toolkit = sun.awt.X11.XToolkit
    file.encoding = UTF-8
    file.encoding.pkg = sun.io
    file.separator = /
    java.awt.graphicsenv = sun.awt.X11GraphicsEnvironment
    java.awt.printerjob = sun.print.PSPrinterJob
    java.class.path = .
    java.class.version = 52.0
    user.country = CN
    user.dir = /root
    user.home = /root
    user.language = zh
    user.name = root
    user.timezone =
```

# 问题分析和解决过程

Java文件中的中文输出到日志变为乱码，
猜测可能和Javac编译时没有指定UTF-8有关系，
项目是用Maven构建的，检查Maven的编码配置信息如下
```
       <maven.compiler.source>1.8</maven.compiler.source>
       <maven.compiler.target>1.8</maven.compiler.target>
       <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
```
所以排除和Maven的关系。

从服务器上的Java编码情况来看
```
    sun.jnu.encoding = UTF-8
    file.encoding = UTF-8
```
运行时默认的编码设置都是UTF-8。
那么编译时和运行时的编码设置都是UTF-8，理论上不应该出
Java文件中的中文输出为乱码，所以怀疑是应用运行时没有获取到Java的默认编码设置，
于是尝试在启动的脚本中增加时指定参数： -Dfile.encoding=UTF-8，再观察应用的中文日志输出，没有出现乱码。

# 结论
根据解决的过程分析应该是应用在运行时，没有获取到系统的编码，通过手动指定jvm启动
编码参数解决了该问题，虽然这个问题解决了，但是还需要深入调查，确认是由于Jenkins版本的差异导致的，
还是由于编译时JDK的差异导致，根据Jenkins编译时也是依赖系统jdk的java命令启动jvm，因此推测问题
的根本原因是Open JDK和Oracle JDK差异导致的问题，可能是Open JDK没有正确的获取到编UTF-8编码引起，
这个问题的本质原因在后续的文章中分析
