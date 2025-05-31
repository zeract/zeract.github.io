# Virtual Thread

## 什么是Virtual Thread
Virtual Thread 在 JDK 21中引入，相比于Thread(Platform Thread)，Virtual Thread是运行在Platform Thread上的线程，Platfrom Thread 与 OS Thread 的关系是 1:1 对应，而 Virtual Thread 与 Platform Thread 的关系是 N:1 对应，多个 Virtual Thread 对应一个 Platform Thread， Virtual Thread 运行时挂载到 Platform Thread 上，阻塞时则从 Platform Thread 上解除挂载将 Platfrom Thread 释放给其他的 Virtual Thread 进行挂载使用，因此在高 I/O 情况下使用 Virtual Thread 的性能会比使用 Platform Thread 的性能高出很多。 

## 如何使用Virtual Thread

### 通过 Thread.startVirtualThread()创建

通过`Thread.startVirtualThread()`方法创建并启动一个虚拟线程，代码如下：
```java
public class VirtualThreadTest { 

  public static void main(String[] args) { 
    CustomThread customThread = new CustomThread();
    // 创建并且启动虚拟线程
    Thread.startVirtualThread(customThread); 
  }
}

class CustomThread implements Runnable { 
  @Override 
  public void run() { 
    System.out.println("CustomThread run"); 
  } 
}
```

### 通过Thread.ofVirtual()创建

通过`Thread.ofVirtual().unstarted()`方法创建一个未启动的虚拟线程，然后通过`Thread.start()`来启动线程，代码如下：
```java
public class VirtualThreadTest {  
  public static void main(String[] args) { 
    CustomThread customThread = new CustomThread();
    // 创建并且不启动虚拟线程，然后 unStarted.start()方法启动虚拟线程
    Thread unStarted = Thread.ofVirtual().unstarted(customThread);
    unStarted.start(); 
    // 等同于
    Thread.ofVirtual().start(customThread); 
  }
}
class CustomThread implements Runnable { 
  @Override
  public void run() { 
    System.out.println("CustomThread run"); 
  }
}
```

### 通过ThreadFactory创建

通过 `ThreadFactory.newThread()`方式就能创建一个虚拟线程，然后通过 `Thread.start()`来启动线程，代码如下：

```java
public class VirtualThreadTest { 
  public static void main(String[] args) { 
    CustomThread customThread = new CustomThread();
    // 获取线程工厂类
    ThreadFactory factory = Thread.ofVirtual().factory();
    // 创建虚拟线程
    Thread thread = factory.newThread(customThread);
    // 启动线程
    thread.start(); 
  }
}

class CustomThread implements Runnable {
  @Override
  public void run() {
    System.out.println("CustomThread run");
  }
}
```


### 通过Executors.newVirtualThreadPerTaskExecutor()创建

通过 JDK 自带的 `Executors` 工具类方式创建一个虚拟线程，然后通过 `executor.submit()`来启动线程，代码如下：

```java
public class VirtualThreadTest {
  public static void main(String[] args) {
    CustomThread customThread = new CustomThread();
    ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
    executor.submit(customThread);
  }
}
class CustomThread implements Runnable {
  @Override
  public void run() {
    System.out.println("CustomThread run");
  } 
}
```

## Virtual Thread 存在什么问题

如果 Virtual Thread 进入 `Synchronized` 代码块内部之后**被阻塞**，那么就会出现 Virtual Thread Pinning 的问题，即 Virutal Thread 与 Platform Thread 被锁定，此时 Virtual Thread 就无法从 Platform Thread 上解除挂载，如果所有的 Platform Thread 都被锁定，那么新的 Virtual Thread 就无法挂载到任何一个 Platform Thread 上，使得任务无法继续执行，这样就会大大降低 Virtual Thread 的性能。

## 如何解决 Virtual Thread Pinning问题

1. 使用 `ReentrantLock` 来 替换 `Synchronized`，这样就可以避免 Virtual Thread 与 Platform Thread 的绑定，因为 `Synchronized` 的底层是在 OS 级别进行上锁，所以 `JVM` 无法干预这个锁，而`ReentrantLock` 是由 `JVM` 进行管理，所以即使在上锁后，也可以将 Virtual Thread 从 Platform Thread 上解除挂载， 从而避免 **Virtual Thread Pinning** 的情况。
2. 使用 `JDK 24` ，在 `JDK 24` 中， 解决了 **Virtual Thread Pinning** 的问题，具体的说明可以查看`JEP 491`。


参考链接：
1. [Virtual Threads: Fast, Furious, and Sometimes Stuck](https://medium.com/@Games24x7Tech/virtual-threads-fast-furious-and-sometimes-stuck-654cde6c0c8c)
2. [Java 21 Virtual Threads - Dude, Where’s My Lock?](https://netflixtechblog.com/java-21-virtual-threads-dude-wheres-my-lock-3052540e231d)
3. [Concurrent programming in Java with virtual threads
](https://github.com/aliakh/demo-java-virtual-threads)
4. [The Idea Behind Java Virtual Threads - It's not about speed
](https://thebackendguy.com/posts/idea-behind-virtual-threads/)
5. [JEP 491: Synchronize Virtual Threads without Pinning
](https://openjdk.org/jeps/491)

