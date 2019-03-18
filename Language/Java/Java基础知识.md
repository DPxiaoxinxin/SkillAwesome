# Java基础知识

[TOC]

## 集合

### HashMap

#### 内部实现

##### 存储结构

* Node：Key + Value
* 数组：长度必须是2的n次方。
  * 非常规设计，常规设计是为素数，如HashTable初始化大小为11，减少冲突概率
  * 扩容和取模时做优化，同时减少冲突
* 链表 or 红黑树：
  * JDK1.8后添加。当容量超过MIN_TREEIFY_CAPACITY(默认64)且链表长度超过TREEIFY_THRESHOLD(默认8)会自动转换为红黑树

##### 存储方式: 哈希表

- 链地址法(采用): 数组+链表结合，元素定位hash定位到数组，通过链表相连
- 开放地址法

##### 扩容机制

* 当Size(实际容量) > Threshold = Length(最大容量默认16) * Load Factor(负载因子0.75)时，最大容量扩大2倍
* JDK1.8后的优化
  - 使用的是2次幂的扩展(指长度扩为原来2倍)
  - 元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。
  - 不需要像JDK1.7的实现那样重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了
    - 0的话索引没变
    - 1的话索引变成“原索引+oldCap”

##### 确定哈希桶数组索引

* 核心方法：

  ``` java
  // 方法一：
  static final int hash(Object key) {   //jdk1.8 & jdk1.7
       int h;
       // h = key.hashCode() 为第一步 取hashCode值
       // h ^ (h >>> 16)  为第二步 高位参与运算
       return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
  }
  // 方法二：
  static int indexFor(int h, int length) {  //jdk1.7的源码，jdk1.8没有这个方法，但是实现原理一样的
       return h & (length-1);  //第三步 取模运算
  }
  ```

* 取key的hashcode值

  - 在JDK1.8的实现中，优化了高位运算的算法，通过hashCode()的高16位异或低16位实现的：(h = k.hashCode()) ^ (h >>> 16)，主要是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候，也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销。

* 高位运算

* 取模运算

  - 通过h & (table.length -1)来得到该对象的保存位
  - 而HashMap底层数组的长度总是2的n次方，这是HashMap在速度上的优化
  - 当length总是2的n次方时，h& (length-1)运算等价于对length取模，也就是h%length，但是&比%具有更高的效率。

#### 线程不安全

* 扩容resize会有一定概率出现Infinite Loop
  * JDK8后已经修复了这个问题，但依然有其他弊端

#### 使用建议

* 扩容是一个特别耗性能的操作，初始化的时候给一个大致的数值，避免map进行频繁的扩容

#### 线程安全用法

- 替换成Hashtable，Hashtable通过对整个表上锁实现线程安全，因此效率比较低
- 使用Collections类的synchronizedMap方法包装一下
- 使用ConcurrentHashMap，它使用分段锁来保证线程安全

### ArrayList

#### 扩容机制：动态扩容

- 无参初始化长度为0
- 添加元素前先确认长度是否足够
- 每次扩容长度=old + (old >> 1) == 1.5 * old，如果初始长度0则扩容为默认长度10

## 内部类

### Object

#### 概述

* Java中其他所有类的祖先；位于java.lang包，在编译时会自动导入
* 没有定义属性，一共有13个方法

#### 方法

##### 1. 类构造器public Object()

* Java中规定：在类定义过程中，对于未定义构造函数的类，默认会有一个无参数的构造函数，作为所有类的基类，Object类自然要反映出此特性

##### 2. private static native void registerNatives()

* 方法的具体实现体在dll文件中，对于不同平台，其具体实现应该有所不同。

``` java
private static native void registerNatives();
static {
    registerNatives();
}
```

##### 3. protected native Object clone() throws CloneNotSupportedException

* clone函数返回的是一个引用，指向的是新的clone出来的对象，此对象与原对象分别占用不同的堆空间。
* 需要实现Cloneable接口，如果没有实现Cloneable接口，并且子类直接调用Object类的clone()方法，则会抛出CloneNotSupportedException异常
  * 仅是一个表示接口，接口本身不包含任何方法，用来指示Object.clone()可以合法的被子类引用所调用。

##### 4. public final native Class<?> getClass();

* 返回的是此Object对象的类对象/运行时类对象Class<?>。效果与Object.class相同。
  * 类对象：
    * 在Java中，类是是对具有一组相同特征或行为的实例的抽象并进行描述，对象则是此类所描述的特征或行为的具体实例
    * 作为概念层次的类，其本身也具有某些共同的特性，如都具有类名称、由类加载器去加载，都具有包，具有父类，属性和方法等
    * Java中有专门定义了一个类，Class，去描述其他类所具有的这些特性，因此，从此角度去看，类本身也都是属于Class类的对象

##### 5.public boolean equals(Object obj);

* 判断两个对象是否相等。必要时需重写，重写需重写hashCode()

  ``` java
  public boolean equals(Object obj) {
          return (this == obj);
  }
  ```

* ==判断两个变量值是否相等（对于基础类型，地址中存储的是值，引用类型则存储指向实际对象的地址）

##### 6. public native int hashCode();

* 返回一个整形数值，表示该对象的哈希码值
* **两个对象相等 <=>  equals()相等  => hashCode()相等**
* **hasCode()不相等 => equals（）不相等 <=> 两个对象不相等。**

##### 7. public String toString();

* toString()是由对象的类型和其哈希码唯一确定，同一类型但不相等的两个对象分别调用toString()方法返回的结果可能相同。

  ``` java
  public String toString() {
          return getClass().getName() + "@" + Integer.toHexString(hashCode());
  }
  ```

##### 8 - 12. wait(...) / notify() / notifyAll()edException;

* 方法

  * wait：
    * 调用此方法所在的当前线程等待，直到在其他线程上调用此方法的主调（某一对象）的notify()/notifyAll()方法。
    * 调用后当前线程将立即阻塞，且适当其所持有的同步代码块中的锁，直到被唤醒或超时或打断后且重新获取到锁后才能继续执行；
  * notify()/notifyAll():
    * 唤醒在此对象监视器上等待的单个线程/所有线程。notify()非公平，为随机唤醒。
    * 调用后，其所在线程不会立即释放所持有的锁，直到其所在同步代码块中的代码执行完毕，此时释放锁，因此，如果其同步代码块后还有代码，其执行则依赖于JVM的线程调度。

  ```java
  // 
  public final native void wait(long timeout) throws InterruptedException;
  public final void wait(long timeout, int nanos) throws InterruptedException{}
  public final void wait() throws InterruptedException{wait(0)}
  // 
  public final native void notify();
  public final native void notifyAll();
  ```

* 示例：

``` java
public class ThreadTest {

    /**
     * @param args
     */
    public static void main(String[] args) {
        // TODO Auto-generated method stub
        MyRunnable r = new MyRunnable();
        Thread t = new Thread(r);
        t.start();
        synchronized (r) {
            try {
                System.out.println("main thread 等待t线程执行完");
                r.wait();
                System.out.println("被notity唤醒，得以继续执行");
            } catch (InterruptedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
                System.out.println("main thread 本想等待，但被意外打断了");
            }
            System.out.println("线程t执行相加结果" + r.getTotal());
        }
    }
}

class MyRunnable implements Runnable {
    private int total;

    @Override
    public void run() {
        // TODO Auto-generated method stub
        synchronized (this) {
            System.out.println("Thread name is:" + Thread.currentThread().getName());
            for (int i = 0; i < 10; i++) {
                total += i;
            }
            notify();
            System.out.println("执行notif后同步代码块中依然可以继续执行直至完毕");
        }
        System.out.println("执行notif后且同步代码块外的代码执行时机取决于线程调度");
    }

    public int getTotal() {
        return total;
    }
}

/** 运行结果：
main thread 等待t线程执行完
Thread name is:Thread-0
执行notif后同步代码块中依然可以继续执行直至完毕
执行notif后且同步代码块外的代码执行时机取决于线程调度  //此行输出位置有具体的JVM线程调度决定，有可能最后执行
被notity唤醒，得以继续执行
线程t执行相加结果45
**/
```

##### 13. protected void finalize() throws Throwable { }

* 与Java垃圾回收机制有关。
  * 具体调用时机在：JVM准备对此对形象所占用的内存空间进行垃圾回收前，将被调用。
  * 此方法并不是由我们主动去调用的（虽然可以主动去调用，此时与其他自定义方法无异）。
* JDK9后已弃用
  * 可能会导致性能问题、死锁、资源泄漏和阻塞。



## IO模型

###  BIO(Blocking I/O)

#### 概述

* 同步阻塞I/O模式，数据的读取写入必须阻塞在一个线程内等待其完成。

#### 特点

- 面向Stream
  - 意味着从流中一次可以读取一个或多个字节，拿到读取的这些做什么你说了算，这里没有任何缓存
    - 这里指的是使用流没有任何缓存，接收或者发送的数据是缓存到操作系统中的，流就像一根水管从操作系统的缓存中读取数据
  - 只能顺序从流中读取数据，如果需要跳过一些字节或者再读取已经读过的字节，你必须将从流中读取的数据先缓存起来。
- 阻塞IO

#### 示例

* 传统的客户端-服务端模型：server收到请求创建新线程处理，即典型的典型的 **一请求一应答通信模型**
* 伪异步IO的客户端-服务端模型：通过采用线程池和任务队列对传统模型进行改进

### NIO

#### 概述

- 同步非阻塞的I/O模型，在Java 1.4 中引入了NIO框架
- NIO可让您只使用一个（或几个）单线程管理多个通道（网络连接或文件），但付出的代价是解析数据可能会比从一个阻塞流中读取数据更复杂。

#### 特点

- 面向Buffer
  - 数据是先被 读/写到buffer中的，根据需要你可以控制读取什么位置的数据。
  - 需要额外做的工作是检查你需要的数据是否已经全部到了buffer中
  - 需要保证当有更多的数据进入buffer中时，buffer中未处理的数据不会被覆盖
- 非阻塞IO
  - NIO的非阻塞模式允许一条线程从channel中读取数据，通过返回值来判断buffer中是否有数据，如果没有数据，NIO不会阻塞，因为不阻塞这条线程就可以去做其他的事情，过一段时间再回来判断一下有没有数据
- Selectors
  - selectors允许一条线程去监控多个channels的输入，你可以向一个selector上注册多个channel，然后调用selector的select()方法判断是否有新的连接进来或者已经在selector上注册时channel是否有数据进入

#### 数据类型

- Buffer：包含要写入或者刚读出的数据
- Charset：提供Unicode字符串影射到字节序列以及逆影射的操作
- Channels：
  - 可以通过它读取和写入数据
  - 数据的源头或者数据的目的地，用于向buffer提供数据或者读取buffer数据，并且对I/O提供异步支持。
- Selector：将多元异步I/O操作集中到一个或多个线程中。

### AIO (Asynchronous I/O)

#### 概述

* AIO 也就是 NIO 2，在 Java 7 中引入了 NIO 的改进版 NIO 2
* 异步非阻塞的IO模型
  * 异步 IO 是基于事件和回调机制实现的，也就是应用操作之后会直接返回，不会堵塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续的操作。
* AIO 是异步IO的缩写，虽然 NIO 在网络操作中，提供了非阻塞的方法，但是 NIO 的 IO 行为还是同步的。
  * 对于 NIO 来说，我们的业务线程是在 IO 操作准备好时，得到通知，接着就由这个线程自行进行 IO 操作，IO操作本身是同步的
* 目前来说 AIO 的应用还不是很广泛，Netty 之前也尝试使用过 AIO，不过又放弃了。





## 线程

### 线程池

` ExecutorService pool = Executors.newCachedThreadPool();`

#### 特点

* 解决处理器单元内多个线程的执行问题
* 减少处理器单元闲置时间
* 增加了处理器单元工作时间内的吞吐能力（为什么这么说？我们减少了多个任务每次线程的创建和销毁浪费，提高了任务执行效率）

#### 组成

* 线程池管理器（ThreadPool）：负责创建、管理、销毁线程池，以及添加任务
* 工作线程（PoolWorker）：无任务则等待，可循环、重复执行任务
* 任务接口（Task）：每个任务必须实现接口，工作线程负责调度任务的执行，规定了任务的入口，以及任务完成后的收尾工作以及任务执行状态等等
* 任务队列（TaskQueue）：存放没有处理的任务，提供任务缓冲机制

#### ExecutorService方法

``` java
public interface ExecutorService extends Executor {

   
    void shutdown();//顺次地关闭ExecutorService,停止接收新的任务，等待所有已经提交的任务执行完毕之后，关闭ExecutorService


    List shutdownNow();//阻止等待任务启动并试图停止当前正在执行的任务，停止接收新的任务，返回处于等待的任务列表


    boolean isShutdown();//判断线程池是否已经关闭

    boolean isTerminated();//如果关闭后所有任务都已完成，则返回 true。注意，除非首先调用 shutdown 或 shutdownNow，否则 isTerminated 永不为 true。

    
    boolean awaitTermination(long timeout, TimeUnit unit)//等待（阻塞）直到关闭或最长等待时间或发生中断,timeout - 最长等待时间 ,unit - timeout 参数的时间单位  如果此执行程序终止，则返回 true；如果终止前超时期满，则返回 false 

 
     Future submit(Callable task);//提交一个返回值的任务用于执行，返回一个表示任务的未决结果的 Future。该 Future 的 get 方法在成功完成时将会返回该任务的结果。


     Future submit(Runnable task, T result);//提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future。该 Future 的 get 方法在成功完成时将会返回给定的结果。

 
    Future submit(Runnable task);//提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future。该 Future 的 get 方法在成功 完成时将会返回 null


     List> invokeAll(Collection> tasks)//执行给定的任务，当所有任务完成时，返回保持任务状态和结果的 Future 列表。返回列表的所有元素的 Future.isDone() 为 true。
        throws InterruptedException;


     List> invokeAll(Collection> tasks,
                                  long timeout, TimeUnit unit)//执行给定的任务，当所有任务完成时，返回保持任务状态和结果的 Future 列表。返回列表的所有元素的 Future.isDone() 为 true。
        throws InterruptedException;


     T invokeAny(Collection> tasks)//执行给定的任务，如果在给定的超时期满前某个任务已成功完成（也就是未抛出异常），则返回其结果。一旦正常或异常返回后，则取消尚未完成的任务。
        throws InterruptedException, ExecutionException;


     T invokeAny(Collection> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

public interface Executor {

    void execute(Runnable command);//执行已提交的 Runnable 任务对象。此接口提供一种将任务提交与每个任务将如何运行的机制（包括线程使用的细节、调度等）分离开来的方法
}
```



#### 分类

##### newFixedThreadPool(int)：固定线程池

##### newWorkStealingPool()：处理器核心数的并行线程池

##### newSingleThreadExecutor()：一个线程的单独线程池

##### newCachedThreadPool()：缓存线程池

##### newSingleThreadScheduledExecutor() ：单独线程定时线程池

##### newScheduledThreadPool(int)：定时线程池

#### 实现类ThreadPoolExecutor

##### 核心参数

- `corePoolSize`为线程池的基本大小。
  - 默认情况下，即使是核心线程（core threads）也只有在新的任务到达时，才会开始创建和启动。但是，可以重置`prestartCoreThread()`和`prestartAllCoreThreads()`来动态改变。
- `maximumPoolSize`为线程池最大线程大小。
- `keepAliveTime`和 `unit`则是线程空闲后的存活时间。
  - 如果当前线程池中的线程数量超过`corePoolSize`，如果多余的线程已经处于空闲状态（idle）的时间大于`keepAliveTime`，那么这些多余的线程将被终止。当线程池不活跃时，这个策略可以降低资源消耗。可以使用`setKeepAliveTime(long, TimeUnit)`来动态的设置`keepAliveTime`。
  - 默认情况下，当且仅当线程池中的线程数量超过`corePoolSize`时，keep-alive策略才会生效，但是可以使用`allowCoreThreadTimeOut(boolean)`使得没有超过`corePoolSize`的线程在超过`keepAliveTime`时也会生效。
- `workQueue`用于存放任务的阻塞队列。
  - 入队策略
    - Direct handoffs：`SynchronousQueue`是handoff的默认的一个好例子，此队列传递任务而不是持有他们。当处理一些内部有依赖关系的任务时，此策略能够避免加锁。Direct handoffs通常需要无界的`maximumPoolSize`来避免发生拒绝提交任务请求的情况。如果任务处理速度小于任务提交的速度，那么可能会导致线程无界增长，引发资源问题。
    - Unbounded queues：当`corePoolSize`中的线程都忙（busy）时，使用无界队列（例如`LinkedBlockingQueue`）将导致新的任务入队等待。因此，创建线程的数量不可能大于`corePoolSize`，并且`maximumPoolSize`也不起任何作用。这种策略适用于各个提交的任务之间不会相互影响的场景，例如web page server。
    - Bounded queues：通过设置有限的maximumPoolSize，有界队列可以避免资源耗尽，但是在调优上有一定困难。队列大小和线程池最大值在以下两种场景中需要权衡利弊：
      1. 大队列小线程池，能够降低CPU使用率、系统资源和上下文切换开销，但是也会导致吞吐量严重降低。
      2. 小队列大线程池，能够提高CPU使用率，但是会导致不可接受的调度开销，因此也会降低吞吐量。
- `handler`当队列和最大线程池都满了之后的饱和策略。当`ThreadPoolExecutor`关闭、队列和线程池饱和时，会拒绝新提交的任务，同时调用`RejectedExecutionHandler.rejectedExecution(Runnable, ThreadPoolExecutor)`方法。
  - ThreadPoolExecutor.AbortPolicy，默认策略，在拒绝任务时，会抛出`RejectedExecutionException`。
  - `ThreadPoolExecutor.CallerRunsPolicy`，由提交的线程自己来执行（execute）当前提交的任务。这种策略提供了简单的反馈控制机制，能够降低新的任务提交的速率。
  - `ThreadPoolExecutor.DiscardPolicy`，简单粗暴的抛弃不能执行的任务。
  - `ThreadPoolExecutor.DiscardOldestPolicy`，如果`ThreadPoolExecutor`没有被关闭，那么删除队列头部的任务，并且再次尝试提交任务，如果仍然被拒绝，那么再删除队列头部任务，如此反复。
- `ThreadFactory` 创建线程的线程工厂
  - 如果没有特别指定，那么默认使用`Executors.defaultThreadFactory()`来创建线程，这些创建的线程属于同一个`ThreadGroup`、有同样的`NORM_PRIORITY`优先级和非守护线程状态（non-demon status）。通过使用不同的`ThreadFactory`，可以指定thread name、thread group、priority。

##### 线程池大小调整策略

1. 当用`execute`提交任务时，如果运行的线程数量少于`corePoolSize`，即使当前有线程处于空闲状态（idle），那么仍然会创建一个新的线程来处理提交的任务。
2. 如果线程的数量大于`corePoolSize`并且小于`maximumPoolSize`，当且仅当队列满了的时候，才会创建一个新的线程来处理提交的任务。
3. 如果线程的数量大于`maximumPoolSize`，那么就会执行相应的拒绝策略。



 

 

 

 