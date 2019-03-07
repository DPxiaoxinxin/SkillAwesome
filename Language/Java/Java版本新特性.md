# Java版本新特性

* 本文大部分来自[**codecraft**](https://segmentfault.com/u/codecraft)的Java语言特性系列[Java11的新特性](https://segmentfault.com/a/1190000016527932)

[TOC]



## Java5

### 泛型

### 枚举

### 装箱拆箱

### 变长参数

### 注解

### foreach循环

### 静态导入

### 格式化

### 线程框架/数据结构

### Arrays工具类/StringBuilder/instrument

## Java6

### SR223脚本引擎

### JSR199--Java Compiler API

### JSR269--Pluggable Annotation Processing API

### 支持JDBC4.0规范

### JAX-WS 2.0规范

## Java7

### suppress异常(`新语法`)

### 捕获多个异常(新语法)

### try-with-resources(新语法)

### JSR341-Expression Language Specification(新规范)

### JSR203-More New I/O APIs for the Java Platform(新规范)

### JSR292与InvokeDynamic

### 支持JDBC4.1规范

### Path接口、DirectoryStream、Files、WatchService

### jcmd

### fork/join framework

### Java Mission Control

## Java8

### lamda表达式(`重磅`)

### 集合的stream操作

### 提升HashMaps的性能

### Date-Time Package

### java.lang and java.util Packages

### Concurrency

## Java9

java9大刀阔斧，重磅引入了模块化系统，自身jdk的类库也首当其冲模块化。新引入的jlink可以精简化jdk的大小，外加Alpine Linux的docker镜像，可以大大减少java应用的docker镜像大小，同时也支持了Docker的cpu和memory限制(`Java SE 8u131及以上版本开始支持`)，非常值得使用。

完整的特性详见[JDK 9 features](http://openjdk.java.net/projects/jdk9/)

### 进程操作改进（JEP 102: Process API Updates）

### 竞争锁的性能优化（JEP 143: Improve Contended Locking）

### 代码执行效率改善（JEP 197: Segmented Code Cache）

### Java 模块化（JEP 261: Module System）

### 交互式命令行（JEP 222: jshell: The Java Shell）

### ResourceBundle 支持 UTF-8 编码（JEP 226: UTF-8 Property Resource Bundles）

### G1 成为默认的垃圾收集器（JEP 248: Make G1 the Default Garbage Collector）

### 优化字符串占用空间（JEP 254: Compact Strings）

### HTTP/2 Client



## Java10

### **局部变量的类型推断**[286: Local-Variable Type Inference](http://openjdk.java.net/jeps/286)(重磅)

> 相关解读: [java10系列(二)Local-Variable Type Inference](https://segmentfault.com/a/1190000014025792)

### **合并 JDK 多个代码仓库到一个单独的储存库中**[296: Consolidate the JDK Forest into a Single Repository](http://openjdk.java.net/jeps/296)

### **垃圾回收器接口**[304: Garbage-Collector Interface](http://openjdk.java.net/jeps/304)

### **并行全垃圾回收器 G1**[307: Parallel Full GC for G1](http://openjdk.java.net/jeps/307)

### **应用类数据共享(CDS)** [310: Application Class-Data Sharing](http://openjdk.java.net/jeps/310)

### **线程-局部变量管控**[312: Thread-Local Handshakes](http://openjdk.java.net/jeps/312)

### **移除 Native-Header 自动生成工具**[313: Remove the Native-Header Generation Tool (javah)](http://openjdk.java.net/jeps/313)

### **额外的 Unicode 语言标签扩展**[314: Additional Unicode Language-Tag Extensions](http://openjdk.java.net/jeps/314)

### **在备用存储装置上的堆分配**[316: Heap Allocation on Alternative Memory Devices](http://openjdk.java.net/jeps/316)

### **试验性的基于 Java 的 JIT 编译器**[317: Experimental Java-Based JIT Compiler](http://openjdk.java.net/jeps/317)(重磅)

> 相关解读: [Java10来了，来看看它一同发布的全新JIT编译器](https://mp.weixin.qq.com/s/NyDANTzK_uv6hwTXjknDLw)

**根证书**[319: Root Certificates](http://openjdk.java.net/jeps/319)

> 相关解读: [OpenJDK 10 Now Includes Root CA Certificates](https://dzone.com/articles/openjdk-10-now-includes-root-ca-certificates)

### **基于时间的版本控制**[322: Time-Based Release Versioning](http://openjdk.java.net/jeps/322)

> 相关解读: [java10系列(一)Time-Based Release Versioning](https://segmentfault.com/a/1190000013885784)





## Java11

### [181: Nest-Based Access Control](https://openjdk.java.net/jeps/181)

> 相关解读[Java Nestmate稳步推进](http://www.infoq.com/cn/news/2018/03/Nestmates)，[Specification for JEP 181: Nest-based Access Control](http://cr.openjdk.java.net/~dlsmith/nestmates.html)
> 简单的理解就是Class类新增了getNestHost，getNestMembers方法

### [309: Dynamic Class-File Constants](https://openjdk.java.net/jeps/309)

> 相关解读[Specification for JEP 309: Dynamic Class-File Constants (JROSE EDITS)](http://cr.openjdk.java.net/~jrose/jvm/constant-dynamic-jrose.html)
> jvm规范里头对Constant pool新增一类CONSTANT_Dynamic

### [315: Improve Aarch64 Intrinsics](https://openjdk.java.net/jeps/315)

> 对于AArch64处理器改进现有的string、array相关函数，并新实现java.lang.Math的sin、cos、log方法

### [318: Epsilon: A No-Op Garbage Collector](https://openjdk.java.net/jeps/318)

> 引入名为Epsilon的垃圾收集器，该收集器不做任何垃圾回收，可用于性能测试、短生命周期的任务等，使用-XX:+UseEpsilonGC开启

### [320: Remove the Java EE and CORBA Modules](https://openjdk.java.net/jeps/320)(`重磅`)

> 将java9标记废弃的Java EE及CORBA模块移除掉，具体如下：（1）xml相关的，java.xml.ws, java.xml.bind，java.xml.ws，java.xml.ws.annotation，jdk.xml.bind，jdk.xml.ws被移除，只剩下java.xml，java.xml.crypto,jdk.xml.dom这几个模块；（2）java.corba，java.se.ee，java.activation，java.transaction被移除，但是java11新增一个java.transaction.xa模块

### [321: HTTP Client (Standard)](https://openjdk.java.net/jeps/321)(`重磅`)

> 相关解读[java9系列(六)HTTP/2 Client (Incubator)](https://segmentfault.com/a/1190000013518969)，[HTTP Client Examples and Recipes](https://openjdk.java.net/groups/net/httpclient/recipes.html)，在java9及10被标记incubator的模块jdk.incubator.httpclient，在java11被标记为正式，改为java.net.http模块。

### [323: Local-Variable Syntax for Lambda Parameters](https://openjdk.java.net/jeps/323)

> 相关解读[New Java 11 Language Feature: Local-Variable Type Inference (var) extended to Lambda Expression Parameters](https://medium.com/the-java-report/java-11-sneak-peek-local-variable-type-inference-var-extended-to-lambda-expression-parameters-e31e3338f1fe)
> 允许lambda表达式使用var变量，比如(var x, var y) -> x.process(y)，如果仅仅是这样写，倒是无法看出写var有什么优势而且反而觉得有点多此一举，但是如果要给lambda表达式变量标注注解的话，那么这个时候var的作用就突显出来了(@Nonnull var x, @Nullable var y) -> x.process(y)

[324: Key Agreement with Curve25519 and Curve448](https://openjdk.java.net/jeps/324)

> 使用RFC 7748中描述的Curve25519和Curve448实现key agreement

### [327: Unicode 10](https://openjdk.java.net/jeps/327)

> 升级现有的API，支持Unicode10.0.0

### [328: Flight Recorder](https://openjdk.java.net/jeps/328)

> 相关解读[Java 11 Features: Java Flight Recorder](https://dzone.com/articles/java-11-features-java-flight-recorder)
> Flight Recorder以前是商业版的特性，在java11当中开源出来，它可以导出事件到文件中，之后可以用Java Mission Control来分析。可以在应用启动时配置java -XX:StartFlightRecording，或者在应用启动之后，使用jcmd来录制，比如

```
$ jcmd <pid> JFR.start
$ jcmd <pid> JFR.dump filename=recording.jfr
$ jcmd <pid> JFR.stop
```

### [329: ChaCha20 and Poly1305 Cryptographic Algorithms](https://openjdk.java.net/jeps/329)

> 实现 RFC 7539的ChaCha20 and ChaCha20-Poly1305加密算法

### [330: Launch Single-File Source-Code Programs](https://openjdk.java.net/jeps/330)(`重磅`)

> 相关解读[Launch Single-File Source-Code Programs in JDK 11](https://dzone.com/articles/launch-single-file-source-code-programs-in-jdk-11)
> 有了这个特性，可以直接java HelloWorld.java来执行java文件了，无需先javac编译为class文件然后再java执行class文件，两步合成一步

### [331: Low-Overhead Heap Profiling](https://openjdk.java.net/jeps/331)

> 通过JVMTI的SampledObjectAlloc回调提供了一个开销低的heap分析方式

### [332: Transport Layer Security (TLS) 1.3](https://openjdk.java.net/jeps/332)(`重磅`)

> 支持RFC 8446中的TLS 1.3版本

### [333: ZGC: A Scalable Low-Latency Garbage Collector(Experimental)](https://openjdk.java.net/jeps/333)(`重磅`)

> 相关解读[JDK11的ZGC小试牛刀](https://segmentfault.com/a/1190000015725327)，[一文读懂Java 11的ZGC为何如此高效](https://mp.weixin.qq.com/s/nAjPKSj6rqB_eaqWtoJsgw)

### [335: Deprecate the Nashorn JavaScript Engine](https://openjdk.java.net/jeps/335)

> 相关解读[Oracle弃用Nashorn JavaScript引擎](http://www.infoq.com/cn/news/2018/06/deprecate-nashorn)，[Oracle GraalVM announces support for Nashorn migration](https://medium.com/graalvm/oracle-graalvm-announces-support-for-nashorn-migration-c04810d75c1f)
> 废除Nashorn javascript引擎，在后续版本准备移除掉，有需要的可以考虑使用GraalVM

### [336: Deprecate the Pack200 Tools and API](https://openjdk.java.net/jeps/336)

> 废除了pack200以及unpack200工具以及java.util.jar中的Pack200 API。Pack200主要是用来压缩jar包的工具，不过由于网络下载速度的提升以及java9引入模块化系统之后不再依赖Pack200，因此这个版本将其移除掉。