---
title: Java Sandbox
date: 2024-03-21 17:54:54
categories: [Java]
tags: [IT, sandbox]
---

### 背景：

在项目开发中，我们经常会遇到一些需要动态执行脚本的场景，比如：java执行一段页面编辑好的js脚本，如果存在恶意脚本的话，会引发一些安全问题。
因此，一方面我们可以通过制定业务使用规则来限制脚本的内容； 而另一种则是通过沙箱来限制脚本的执行环境。


### 沙箱：
沙箱通常是一种受控的执行环境，应用程序在其中运行，但受到限制以防止其访问敏感资源或执行恶意操作。

有以下几个方面：

- 资源隔离: 将应用程序限制在其分配的资源范围内，例如内存、CPU、文件系统访问等，防止应用程序对系统资源的滥用。

- 权限控制: 沙箱机制可以限制应用程序的权限，确保其只能访问必要的资源和功能，防止其执行潜在危险的操作。

- 环境隔离: 沙箱可以在一个隔离的环境中运行应用程序，使其与系统的其他部分隔离开来。这有助于防止应用程序对系统的其他部分产生影响，即使应用程序本身被感染或受到攻击也能够最大程度地减少潜在的损害。

常见的沙箱技术有：虚拟机JVM、Docker、浏览器沙箱等。

<!--more-->


### Java 沙箱：

Java沙箱机制的主要包括以下几个方面：

- 字节码验证: 保字节码不包含任何违反Java语言规范的内容，以防止恶意代码通过修改字节码来执行恶意操作。

- 类加载器限制: Java应用程序中的类由类加载器加载。Java沙箱通过限制类加载器的行为来确保应用程序只能加载系统允许的类，防止恶意类的加载。

- 安全管理器: Java沙箱使用安全管理器（Security Manager）来控制应用程序对系统资源的访问。安全管理器定义了一组权限，指定了应用程序可以执行的操作类型和访问的资源范围。

- 操作系统访问限制: Java沙箱还通过限制应用程序对底层操作系统的直接访问来提高安全性。例如，Java应用程序不能直接访问底层文件系统或执行本地操作系统命令，必须通过Java API来进行操作。

### Java 开启沙箱：

以下通过一个简单例子来演示java沙箱：

1、编写一个读取文件的java程序，默认执行不开启沙箱，此时可以正常读取文件内容。
```java
String content = FileUtils.readFileToString(new File("/tmp/test.sh"), StandardCharsets.UTF_8);
System.out.println(content);
```

2、开启沙箱：`java -Djava.security.manager Test`, 读取文件会报错：
```java
Exception in thread "main" java.security.AccessControlException: access denied ("java.io.FilePermission" "/tmp/test.sh" "read")
```

3、添加策略policy文件`/tmp/custom.policy`，授予对应文件路径的读权限：
```java
grant {
    permission java.io.FilePermission "/tmp/test.sh", "read";
};
```
这里的permission可以参考：[Permissions in the Java Development Kit (JDK)](https://docs.oracle.com/javase/8/docs/technotes/guides/security/permissions.html)

4、再次启动：`java -Djava.security.manager -Djava.security.policy=/tmp/custom.policy Test`，可以正常读取文件内容。

### Nashorn - Java执行Js沙箱：
- https://github.com/javadelight/delight-nashorn-sandbox

### 参考：
- [The Security Manager](https://docs.oracle.com/javase/tutorial/essential/environment/security.html)
- [Permissions in the Java Development Kit (JDK)](https://docs.oracle.com/javase/8/docs/technotes/guides/security/permissions.html)
- [K：java中的安全模型(沙箱机制)](https://www.cnblogs.com/MyStringIsNotNull/p/8268351.html))