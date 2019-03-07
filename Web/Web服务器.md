# Web服务器

## Nginx

### 架构

#### 模块化/接口设计

* 所有的模块都遵循着同样的ngx_module_t接口设计规范。
  * ngx_module_t描述了整个模块的所有信息，为核心模块进行初始化和调用提供了接口。

#### 核心/常用模块

* **配置模块，配置模块是所有模块的基础，他实现了最基本的配置项的解析功能**
* **事件模块、HTTP模块、mail模块：其中事件模块是HTTP模块和mail模块的基础 **

### 进程模型

#### master进程

* 充当整个进程组与用户的交互接口，同时对进程进行监护
* 不需要处理网络事件，不负责业务的执行，只会通过管理worker进程来实现重启服务、平滑升级、更换日志文件、配置文件实时生效等功能。 
* **master进程中for(::)无限循环内有一个关键的sigsuspend()函数调用，该函数调用是的master进程的大部分时间都处于挂起状态，直到master进程收到信号为止。**

#### worker进程

* 处理基本的网络事件
* worker个数一般与cpu个数相同，

#### 处理过程

* master进程会先建立好需要listen的socket
* master进程fork生成子进程workers，worker会继承socket
  * 此时workers子进程们都继承了父进程master的所有属性，当然也包括已经建立好的socket，当然不是同一个socket，只是每个进程的这个socket会监控在同一个ip地址与端口，这个在网络协议里面是允许的
* 当一个连接进入，产生惊群现象
  * **当一个连接进来后，所有在accept在这个socket上面的进程，都会收到通知，但只有一个进程可以accept这个连接**
  * 当一个worker进程在accept这个连接之后，就开始读取请求，解析请求，处理请求，产生数据后，再返回给客户端，最后才断开连接，一个完整的请求。

### 事件处理机制

#### 异步非阻塞的事件处理机制

* 由进程循环处理多个准备好的事件，采用信号事务机制通知worker进行工作 ，从而实现高并发和轻量级。 
  * 以epoll为例，事件准备好了就去处理，未准备好在epoll等待。因此可以并发处理未完成的请求，实际上进程只有一个，只能同时处理一个请求，所以在请求间不断切换
    * 切换也是因为异步事件未准备好，而主动让出的
    * **这里的切换是没有任何代价，可以理解为循环处理多个准备好的事件**

### 对比Apache

#### 区别

* Apache，每个请求都会独占一个工作线程，当并发数到达几千时，就同时有几千的线程在处理请求了
  * 对于操作系统来说，占用的内存非常大，线程的上下文切换带来的cpu开销也很大，性能就难以上去，同时这些开销是完全没有意义的。 
* Nginx，一个进程只有一个主线程，通过异步非阻塞的事件处理机制，实现了循环处理多个准备好的事件，从而实现轻量级和高并发。

#### 适合场景

##### Nginx是事件驱动，异步非阻塞，适合IO密集型服务

* 适合反向代理，在客户端与WEB服务器之间起一个数据中转作用，纯粹是IO操作，自身并不涉及到复杂计算
* 处理静态资源，本身也是磁盘IO工作

##### Apache采用同步多进程/线程，适合CPU密集型服务

* 适合作为Web/应用服务器，跑具体的业务应用，处理动态请求/CPU密集型请求
  * 例如一个计算耗时2秒，那么这2秒就是完全阻塞的，什么event都没用。这个时候多进程或线程就体现出优势，每个进程各干各的事，互不阻塞和干扰



## Tomcat

### 简介

* 一个JSP/Servlet容器。其作为Servlet容器，有三种工作模式：独立的Servlet容器、进程内的Servlet容器和进程外的Servlet容器。
* **Tomcat依赖<CATALINA_HOME>/conf/server.xml这个配置文件启动server（一个Tomcat实例，核心就是启动容器Catalina**
* **Tomcat部署Webapp时，依赖context.xml和web.xml（<CATALINA_HOME>/conf/目录下的context.xml和web.xml在部署任何webapp时都会启动，他们定义一些默认行为，而具体每个webapp的  META-INF/context.xml  和  WEB-INF/web.xml  则定义了每个webapp特定的行为）两个配置文件部署web应用。**

### 目录

``` 
tomcat
　　|---bin：存放启动和关闭tomcat脚本

　　|---conf：存放不同的配置文件（server.xml和web.xml）；
　　|---doc：存放Tomcat文档；
　　|---lib/japser/common：存放Tomcat运行需要的库文件（JARS）；
　　|---logs：存放Tomcat执行时的LOG文件；
　　|---src：存放Tomcat的源代码；
　　|---webapps：Tomcat的主要Web发布目录（包括应用程序示例）；
　　|---work：存放jsp编译后产生的class文件；
```

### Tomcat启动过程

#### 1. **引导（Bootstrap）启动**

* *调用了org.apache.catalina.startup.Bootstrap.class中的main方法，开始启动Tomcat容器*

#### 2. **调用Bootstrap中的init（），创建了Catalina对象（核心容器）**

* 设置Tomcat实例需要的环境变量
* 实例化运行Tomcat实例的类加载器
* 初始化Tomcat实例

#### 3. ***调用Bootstrap中的load（）：***实际上是通过反射调用了catalina的load方法。 

* 解析Tomcat实例的配置文件(server.xml)，生成合适的组件对象(Server、Service等)

#### 4. 启动最上层的Server实例

* 同样是在Bootstrap中通过反射调用catalina对象的start方法
* 接着启动Server.start()、service.start()、一系列的container的start

#### 5. 设置shutdown hook

* 设置一系列JVM退出前的清理线程，最后执行的是*CatalinaShutdownHook*。

### 总体架构

![](https://www.ibm.com/developerworks/cn/java/j-lo-tomcat1/image001.gif)

#### Server

##### 1. 代表整个服务器实例，管理整个Tomcat的生命周期

* 定义在conf/server.xml

##### 2. 至少包含一个service

#### Service

##### 1. 对外提供具体服务，处理该服务下所有的的请求

##### 2. 包含多个Connector和一个Container

#### Connector

![](https://images2015.cnblogs.com/blog/1078737/201701/1078737-20170109110435088-1621430699.png)

##### 1.  用于接受请求/响应并将请求封装成Request和Response来具体处理

##### 2.  只侦听指定端口及处理指定网络协议

* 连接器类型：HTTP、SSL、AJP 1.3、Proxy

##### 3. 同时需要实现TCP/IP协议和HTTP协议

* 最底层使用的是Socket来进行连接的，Request和Response是按照HTTP协议来封装的

#### Container

##### 1. 封装和管理Servlet，以及具体处理request请求

### Connector架构

![](https://img-blog.csdn.net/20180108205854139?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 使用ProtocolHandler来处理请求

* 不同的ProtocolHandler代表不同的连接类型

#### 组成

##### Endpoint

###### 1. 处理底层Socket的网络连接，用来实现TCP/IP协议

###### 2. 抽象实现AbstractEndpoint里面的定义

* Acceptor：监听请求
* AsyncTimeout：检查异步Request超时
* Handler：处理接收到的Socket，在内部调用Processor处理

##### Processor

###### 1. 将Endpoint接收到的Socket封装成Request

###### 2. 实现HTTP协议

##### Adapter

###### 1. 将Request交给Container(Servlet容器)进行具体的处理

### Container架构

![](https://img-blog.csdn.net/20180108201104048?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### Engine引擎

##### 1. 管理多个站点Host

##### 2. 将请求转发到匹配的Host或默认Host 

##### 3. 一个Service只能有一个Engine

#### Host站点

##### 1. 又称虚拟主机，每个虚拟主机和某个网络域名Domain Name相匹配

##### 2. 部署一个或多个Webapp应用，每个对应一个Context

##### 3. 将请求转发给匹配的Context或默认Context处理

* 采用最长匹配方式

#### Context

##### 1. 代表一个Webapp应用，或者一个WEB-INF目录以及下面的web.xml文件

##### 2. 由一个或多个Servlet组成

* 创建的时候将根据配置文件`$CATALINA_HOME/conf/web.xml`和`$WEBAPP_HOME/WEB-INF/web.xml`载入Servlet类

##### 3. 将请求根据自己的映射表寻找匹配的Servlet类

#### Wrapper

##### 1. 每一Wrapper封装着一个Servlet

### Container 如何处理请求

####  使用Pipeline-Valve管道(责任链模式)

* 每个Pipeline都有特定的Valve，而且是在管道的最后一个执行，这个Valve叫做BaseValve，BaseValve是不可删除的；
  * 四个子容器对应的BaseValve分别在：StandardEngineValve、StandardHostValve、StandardContextValve、StandardWrapperValve。
* 在上层容器的管道的BaseValve中会调用下层容器的管道

#### 处理过程

![](https://img-blog.csdn.net/20180116093931129?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

##### 1. Connector收到请求后调用最顶层容器的Engine管道(EnginePipeline)

##### 2. 顺管道依次执行Valve，最后执行到StandardWrapperValve

##### 3. 在StandardWrapperValve中创建FilterChain，并调用其doFilter方法来处理请求

* 这个FilterChain包含着我们配置的与请求相匹配的Filter和Servlet，其doFilter方法会依次调用所有的Filter的doFilter方法和Servlet的service方法，这样请求就得到了处理

##### 4. 执行完所有的Pipline-Valve及请求，将结果交回Connector以Socket返回给客户端

###  示例：Tomcat结合Spring MVC处理一个HTTP请求

假设来自客户端的请求是:`http://localhost:8080/xxx/index.html`

参考：https://www.cnblogs.com/tengyunhao/p/7518481.html，https://www.cnblogs.com/crazylqy/p/4706223.html

1. 请求被发送到本地端口8080，被在那里侦听的Coyote HTTP/1.1 Connector获得
2. Connector将该请求交给它所在service的Engine处理，并等待来自Engine的回应
   * Connector的主要任务是负责接收浏览器的发过来的 tcp 连接请求，创建一个 Request 和 Response 对象分别用于和请求端交换数据，然后会产生一个线程来处理这个请求并把产生的 Request 和 Response 对象传给处理这个请求的线程
3. Engine获得请求`localhost/xxx/index.html`，匹配它所拥有的虚拟主机Host，找到名为localhohst的Host(即使匹配不到也把请求交给该Host处理，因为该Host被定义为该Engine的默认主机)
4. localhost Host获得请求`/xxx/index.html`，匹配它所有的Context，找到名为xxx的Context(如果匹配不到就把该请求交给路径名为”"的Context去处理)
5. xxx的Context获得请求`index.html`，在它的Mapping Table寻找对应的Servlet，匹配到Spring MVC的DispatcherServlet
6. 构造HttpServletRequest对象和HttpServletResponse对象，作为参数调用DispatcherServlet的doGet或doPost方法
   * doGet或doPost方法接连调用processRequest()、doService()、doDispatch()进行请求分发处理
   * DispatcherServlet 从容器中取出所有 HandlerMapping 实例（每个实例对应一个 HandlerMapping 接口的实现类）并遍历，每个 HandlerMapping 会根据请求信息，通过自己实现类中的方式去找到处理该请求的 Handler (执行程序，如Controller中的方法)，并且将这个 Handler 与一堆 HandlerInterceptor (拦截器) 封装成一个 HandlerExecutionChain 对象，一旦有一个 HandlerMapping 可以找到 Handler 则退出循环；
   * DispatcherServlet 取出 HandlerAdapter 组件，根据已经找到的 Handler，再从所有 HandlerAdapter 中找到可以处理该 Handler 的 HandlerAdapter 对象；
   * 执行 HandlerExecutionChain 中所有拦截器的 preHandler() 方法，然后再利用 HandlerAdapter 执行 Handler ，执行完成得到 ModelAndView，再依次调用拦截器的 postHandler() 方法；
   * 利用 ViewResolver 将 ModelAndView 或是 Exception（可解析成 ModelAndView）解析成 View，然后 View 会调用 render() 方法再根据 ModelAndView 中的数据渲染出页面；
   * 最后再依次调用拦截器的 afterCompletion() 方法，这一次请求就结束了。
7. Dispatcher处理后的请求，一层层返回，直到返回给客户端

### Tomcat调优

* 选择NIO或APR模式

  * 有三张运行模式，BIO、NIO及APR
    * APR安装起来最困难，但是从操作系统层面来解决异步IO问题，大幅度提升性能

* 使用线程池，Tomcat中每一个用户请求都是一个线程

* 禁用AJP连接器，如果用Nginx+Tomcat架构

  * AJPv13协议是面向包的，更简单，可加快响应速度。

  * WEB服务器和Servlet容器通过TCP连接来交互；为了节省SOCKET创建的昂贵代价，WEB服务器会尝试维护一个永久TCP连接到servlet容器，并且在多个请求和响应周期过程会重用连接。

    ![](http://img.blog.csdn.net/20160630220542161)