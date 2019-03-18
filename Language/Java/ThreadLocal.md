# ThreadLocal

[TOC]

## 定义

> This class provides thread-local variables. These variables differ from their normal counterparts in that each thread that accesses one (via its get or set method) has its own, independently initialized copy of the variable. ThreadLocal instances are typically private static fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID).
> Each thread holds an implicit reference to its copy of a thread-local variable as long as the thread is alive and the ThreadLocal instance is accessible; after a thread goes away, all of its copies of thread-local instances are subject to garbage collection (unless other references to these copies exist).

> ThreadLocal 提供了线程本地的实例。它与普通变量的区别在于，每个使用该变量的线程都会初始化一个完全独立的实例副本。ThreadLocal 变量通常被`private static`修饰。当一个线程结束时，它所使用的所有 ThreadLocal 相对的实例副本都可被回收。

总的来说，**ThreadLocal的作用是提供线程内的局部变量，这种变量在线程的生命周期内起作用，减少同一个线程内多个函数或者组件之间一些公共变量的传递的复杂度。即，提供线程内部的局部变量，在本线程内随时可取，隔离其他线程**

**ThreadLocal 适用于每个线程需要自己独立的实例且该实例需要在多个方法中被使用，也即变量在线程间隔离而在方法或类间共享的场景。**

### 不恰当的理解

> ThreadLocal为解决多线程程序的并发问题提供了一种新的思路
>
> ThreadLocal的目的是为了解决多线程访问资源时的共享问题
>
> ThreadLocal 与 synchronize 的异同。

但是，**ThreadLocal 并不解决多线程 共享 变量的问题。既然变量不共享，那就更谈不上同步的问题。**

### 合理的理解

ThreadLoal 变量，它的基本原理是，同一个 ThreadLocal 所包含的对象（对ThreadLocal< String >而言即为 String 类型变量），在不同的 Thread 中有不同的副本（实际是不同的实例）

* 因为每个 Thread 内有自己的实例副本，且该副本只能由当前 Thread 使用。这是也是 ThreadLocal 命名的由来
* 既然每个 Thread 有自己的实例副本，且其它 Thread 不可访问，那就不存在多线程间共享的问题
* 既无共享，何来同步问题，又何来解决同步问题一说？



## 原理

目的：每个访问 ThreadLocal 变量的线程都有自己的一个“本地”实例副本

### JDK早期设计

#### 方案

* 一个全局的ThreadLocal，每个线程都可以访问
* ThreadLocal 维护一个 Map，键是 Thread，值是它在该 Thread 内的实例
* 线程通过该 ThreadLocal 的 get() 方案获取实例时，只需要以线程为键，从 Map 中找出对应的实例即可
* 每个新线程访问该 ThreadLocal 时，需要向 Map 中添加一个映射，而每个线程结束时，应该清除该映射。

### 问题

* 增加线程与减少线程均需要写 Map，故需保证该 Map 线程安全(主要原因)
* 线程结束时，需要保证它所访问的所有 ThreadLocal 中对应的映射均删除，否则可能会引起内存泄漏。
* 每个`Map`存储的`Entry`数量比较大

### JDK8以后的设计

#### 方案

* 每个线程维护一个ThreadLocalMap哈希表，这个哈希表的`key`是`ThreadLocal`实例本身，`value`才是真正要存储的值`Object`。
  * 与 HashMap 不同的是，ThreadLocalMap 的每个 Entry 都是一个对 **键** 的弱引用
  * 每个 Entry 都包含了一个对 **值** 的强引用
* 使用弱引用的原因在于，当没有强引用指向 ThreadLocal 变量时，它可被回收，避免ThreadLocal 不能被回收而造成的内存泄漏的问题。



#### 常用操作

##### 读取实例get()

``` java
public T get() {
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null) {
    ThreadLocalMap.Entry e = map.getEntry(this);
    if (e != null) {
      @SuppressWarnings("unchecked")
      T result = (T)e.value;
      return result;
    }
  }
  return setInitialValue();
}
ThreadLocalMap getMap(Thread t) {
  return t.threadLocals;
}
```

* 读取当前线程，并获得当前线程内的ThreadLocalMap
* 获取该 ThreadLocal 在当前线程的 ThreadLocalMap 中对应的 Entry
* 如果获取到的 Entry 不为 null，从 Entry 中取出值即为所需访问的本线程对应的实例。
* 如果获取到的 Entry 为 null，则通过`setInitialValue()`方法设置该 ThreadLocal 变量在该线程中对应的具体实例的初始值。

##### 设置初始值setInitialValue()

``` java
private T setInitialValue() {
  T value = initialValue();
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null)
    map.set(this, value);
  else
    createMap(t, value);
  return value;
}
```

* 通过`initialValue()`方法获取初始值。该方法为 public 方法，且默认返回 null。所以典型用法中常常重载该方法。
* 拿到该线程对应的 ThreadLocalMap 对象，若该对象不为 null，则直接将该 ThreadLocal 对象与对应实例初始值的映射添加进该线程的 ThreadLocalMap中。若为 null，则先创建该 ThreadLocalMap 对象再将映射添加其中。(不用考虑线程安全问题)

##### 设置实例set()

``` java
public void set(T value) {
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null)
    map.set(this, value);
  else
    createMap(t, value);
}	
```

* 获取该线程的 ThreadLocalMap 对象，
* 直接将 ThreadLocal 对象（即代码中的 this）与目标实例的映射添加进 ThreadLocalMap 中。
* 如果映射已经存在，就直接覆盖。另外，如果获取到的 ThreadLocalMap 为 null，则先创建该 ThreadLocalMap 对象。

#### 问题

ThreadLocalMap 维护 ThreadLocal 变量与具体实例的映射，当 ThreadLocal 变量被回收后，该映射的键变为 null，该 Entry 无法被移除。从而使得实例被该 Entry 引用而无法被回收造成内存泄漏。

* Entry虽然是弱引用，但它是 ThreadLocal 类型的弱引用（也即上文所述它是对 **键** 的弱引用），而非具体实例的的弱引用，所以无法避免具体实例相关的内存泄漏。

##### 解决方案

* ThreadLocalMap 的 set 方法中，通过 replaceStaleEntry 方法将所有键为 null 的 Entry 的值设置为 null，从而使得该值可被回收。另外，会在 rehash 方法中通过 expungeStaleEntry 方法将键和值为 null 的 Entry 设置为 null 从而使得该 Entry 可被回收。通过这种方式，ThreadLocal 可防止内存泄漏。

* 在`ThreadLocal`的`get()`,`set()`,`remove()`的时候都会清除线程`ThreadLocalMap`里所有`key`为`null`的`value`。**但是这些被动的预防措施并不能保证不会内存泄漏**：
  - 使用`static`的`ThreadLocal`，延长了`ThreadLocal`的生命周期，可能导致的内存泄漏（参考[ThreadLocal 内存泄露的实例分析](https://link.juejin.im?target=http%3A%2F%2Fblog.xiaohansong.com%2F2016%2F08%2F09%2FThreadLocal-leak-analyze%2F)）。
  - 分配使用了`ThreadLocal`又不再调用`get()`,`set()`,`remove()`方法，那么就会导致内存泄漏
* 每次使用完`ThreadLocal`，都调用它的`remove()`方法，清除数据。
* JDK建议将ThreadLocal变量定义成private static的，这样的话ThreadLocal的生命周期就更长，由于一直存在ThreadLocal的强引用，所以ThreadLocal也就不会被回收，也就能保证任何时候都能根据ThreadLocal的弱引用访问到Entry的value值，然后remove它，防止内存泄露



## 使用场景

#### 每个线程需要有自己单独的实例

每个线程拥有自己实例，实现它的方式很多。例如可以在线程内部构建一个单独的实例。ThreadLocal 可以以非常方便的形式满足该需求。

#### 实例需要在多个方法中共享，但不希望被多线程共享

可以在满足第一点（每个线程有自己的实例）的条件下，通过方法间引用传递的形式实现。ThreadLocal 使得代码耦合度更低，且实现更优雅。



## 案例

* 对于 Java Web 应用而言，Session 保存了很多信息。很多时候需要通过 Session 获取信息，有些时候又需要修改 Session 的信息。一方面，需要保证每个线程有自己单独的 Session 实例。另一方面，由于很多地方都需要操作 Session，存在多方法共享 Session 的需求。如果不使用 ThreadLocal，可以在每个线程内构建一个 Session实例，并将该实例在多个方法间传递

  * 优化前

    ``` java
    public class SessionHandler {
    
      @Data
      public static class Session {
        private String id;
        private String user;
        private String status;
      }
    
      public Session createSession() {
        return new Session();
      }
    
      public String getUser(Session session) {
        return session.getUser();
      }
    
      public String getStatus(Session session) {
        return session.getStatus();
      }
    
      public void setStatus(Session session, String status) {
        session.setStatus(status);
      }
    
      public static void main(String[] args) {
        new Thread(() -> {
          SessionHandler handler = new SessionHandler();
          Session session = handler.createSession();
          handler.getStatus(session);
          handler.getUser(session);
          handler.setStatus(session, "close");
          handler.getStatus(session);
        }).start();
      }
    }
    ```

  * 优化后

    ``` java
    public class SessionHandler {
    
      public static ThreadLocal<Session> session = new ThreadLocal<Session>();
    
      @Data
      public static class Session {
        private String id;
        private String user;
        private String status;
      }
    
      public void createSession() {
        session.set(new Session());
      }
    
      public String getUser() {
        return session.get().getUser();
      }
    
      public String getStatus() {
        return session.get().getStatus();
      }
    
      public void setStatus(String status) {
        session.get().setStatus(status);
      }
    
      public static void main(String[] args) {
        new Thread(() -> {
          SessionHandler handler = new SessionHandler();
          handler.getStatus();
          handler.getUser();
          handler.setStatus("close");
          handler.getStatus();
        }).start();
      }
    }
    ```

    

* 在`Spring`中，绝大部分`Bean`都可以声明为`singleton`作用域。就是因为`Spring`对一些`Bean`（如`RequestContextHolder`、`TransactionSynchronizationManager`、`LocaleContextHolder`等）中非线程安全状态采用`ThreadLocal`进行处理，让它们也成为线程安全的状态，因为有状态的`Bean`就可以在多线程中共享了。

* 一般的`Web`应用划分为展现层、服务层和持久层三个层次，在不同的层中编写对应的逻辑，下层通过接口向上层开放功能调用。在一般情况下，从接收请求到返回响应所经过的所有程序调用都同属于一个线程`ThreadLocal`是解决线程安全问题一个很好的思路，它通过为每个线程提供一个独立的变量副本解决了变量并发访问的冲突问题。在很多情况下，`ThreadLocal`比直接使用`synchronized`同步机制解决线程安全问题更简单，更方便，且结果程序拥有更高的并发性。



## 参考

[Java进阶（七）正确理解Thread Local的原理与适用场景](http://www.jasongj.com/java/threadlocal/)

[JAVA并发-自问自答学ThreadLocal](https://juejin.im/post/5a0e985df265da430e4ebb92)

[ThreadLocal和synchronized的区别?](https://link.juejin.im/?target=https%3A%2F%2Fwww.zhihu.com%2Fquestion%2F23089780)

