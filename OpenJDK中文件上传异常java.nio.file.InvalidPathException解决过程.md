# 概述
在使用[openjdk编译后出现的中文乱码问题及解决过程](http://wangzhenhua.rocks/open-jdk.html) 提到的解决方案之后，出现了另一个问题，
上传带有中文字符的文件时会报出如下的异常信息
```java
java.nio.file.InvalidPathException: Malformed input or input contains unmappable characters: 中文字符.jpg
        at sun.nio.fs.UnixPath.encode(UnixPath.java:147)
        at sun.nio.fs.UnixPath.<init>(UnixPath.java:71)
        at sun.nio.fs.UnixFileSystem.getPath(UnixFileSystem.java:281)
        at sun.nio.fs.AbstractPath.resolve(AbstractPath.java:53)
        at com.shlf.kernel.service.storage.FileSystemStorage.loadAsResource(FileSystemStorage.java:107)
        at com.shlf.biz.api.FileController.view(FileController.java:74)
        at com.shlf.biz.api.FileController$$FastClassBySpringCGLIB$$e0f8d0b9.invoke(<generated>)
        at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:204)
        at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:721)
        at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:157)
        at org.springframework.aop.aspectj.MethodInvocationProceedingJoinPoint.proceed(MethodInvocationProceedingJoinPoint.java:85)
        at com.shlf.aspect.LogAspect.around(LogAspect.java:89)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
```
这是由于在openjdk中存在两个设置编码的参数，分别是
1. -Dfile.encoding
2. -Djnu.encoding

在这个问题中，我们需要将第2个参数添加到jvm的启动参数中，即可解决当前的问题，但是在jvm中为何会存在两个设置编码的参数，请看下面的具体的分析过程。

# 问题发生时的环境信息
## [部署的服务器信息](http://wangzhenhua.rocks/open-jdk.html#sec-2-3)
## [部署的服务器的编码信息](http://wangzhenhua.rocks/open-jdk.html#sec-2-4)
## [部署的服务器上Java的编码信息](http://wangzhenhua.rocks/open-jdk.html#sec-2-5)

# 问题分析
这个问题是由于没有使用UTF-8的编码解析用户上传的文件名导致，检查系统环境的编码参数，都是UTF-8的编码，
所以根据这个现象，再联系到上一个问题[openjdk编译后出现的中文乱码问题及解决过程](http://wangzhenhua.rocks/open-jdk.html) 猜测是应用在运行时没有获取到
系统编码编码设置，而我们在启动时参数设置中，已经手动指定了编码参数-Dfile.encoding=UTF-8,所以理论上不会出现这个问题,
根据堆栈查看当前UnixPath.encode的源码发现
```java
    CharsetEncoder ce = ref != null ? (CharsetEncoder)ref.get() : null;
    if (ce == null) {
      ce = Util.jnuEncoding().newEncoder().onMalformedInput(CodingErrorAction.REPORT).onUnmappableCharacter(CodingErrorAction.REPORT);
      encoder.set(new SoftReference(ce));
    }

    char[] ca = fs.normalizeNativePath(input.toCharArray());
```
根据源码来看，问题出在Util.jnuEncoding()方法上，再深入分析该方法，它是通过如下方式获取编码参数

```java
 Charset.forName((String)AccessController.doPrivileged(new GetPropertyAction("sun.jnu.encoding")));
```

通过-Dfile.encoding设置的编码没有起作用的原因是jvm是根据sun.jnu.encoding参数来获取编码，于是尝试在启动时指定参数：
-Dsun.jnu.encoding=UTF-8 ，再重启应用，上传带有中文字符的文件，没有出现异常，问题解决。

# 结论
为什么JVM启动的时后需要通过不同的参数设定编码？这二者有什么关系？
在OpenJDK的maillist中看到有人有类似问题 [file.encoding vs. sun.jnu.encoding(?) on OS X](http://mail.openjdk.java.net/pipermail/jdk8-dev/2012-November/001610.html)
```java
There should be no direct relationship between path name and
file.encoding after we introduced in
the property sun.jnu.encoding (implementation detail, not a public
interface) couple release ago. The
charset specified by sun.jnu.encoding should be used for
decoding/encoding the "java" file path and
native file path, not the one from Charset.defaultCharset(), which is
from file.encoding.
Alan has a webrev for this one at

http://cr.openjdk.java.net/~alanb/7050570/webrev/

I just realized that this one is not in yet. Alan, any reason this one
is still not pushed in? I think I have
reviewed it already.

-Sherman
```
由于sun.jun.encoding 是在之前的几个版本引入，sun.jun.encoding和file.encoding并没有什么关系，
file.encoding是在
```java
 Charset.defaultCharset()
```
使用的，sun.jnu.encoding是被用来专门解析file path和native file path，但是该问题也是在打包从
Oracle JDK切换到Open JDK打包出现，所以根本原因还是需要继续分析JDK版本之间的差异

# 参考资料
[file.encoding vs. sun.jnu.encoding(?) on OS X](http://mail.openjdk.java.net/pipermail/jdk8-dev/2012-November/001610.html)
