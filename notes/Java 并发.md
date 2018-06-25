- [线程与进程](#线程与进程)
    - [程序](#程序)
    - [进程](#进程)
    - [线程](#线程)
    - [并发](#并发)
    - [并行](#并行)
    - [程序运行](#程序运行)
- [线程的实现](#线程的实现)
    - [使用内核线程实现](#使用内核线程实现)
    - [使用用户线程实现](#使用用户线程实现)
    - [使用用户线程和轻量级进程混合实现](#使用用户线程和轻量级进程混合实现)
    - [Java 线程的实现](#Java-线程的实现)
    - [线程调度](#线程调度)
- [线程的状态](#线程的状态)
    - [新建（New）](#新建new)
    - [就绪及运行（Runnable）](#就绪及运行runnable)
    - [阻塞（Blocked）](#阻塞blocked)
    - [终止（Terminated）](#终止terminated)
    - [Java 线程状态转换](#Java-线程状态转换)
- [线程控制与线程通信基础](#线程控制与线程通信基础)
    - [线程控制](#线程控制)
        - [开启线程](#开启线程)
        - [暂停线程](#暂停线程)
        - [中断线程](#中断线程)
        - [设置守护线程](#设置守护线程)
    - [线程通信基础](#线程通信基础)
        - [wait()](#wait)
        - [wait(long timeout)](#waitlong-timeout)
        - [notify()](#notify)
        - [notifyAll()](#notifyall)
- [线程安全与线程安全的实现](#线程安全与线程安全的实现)
    - [Java中的线程安全](#Java中的线程安全)
        - [不可变](#不可变)
        - [绝对线程安全](#绝对线程安全)
        - [相对线程安全](#相对线程安全)
        - [线程兼容](#线程兼容)
        - [线程对立](#线程对立)
    - [线程安全的实现方法](#线程安全的实现方法)
        - [互斥同步](#互斥同步)
        - [非阻塞同步](#非阻塞同步)
        - [无同步](#无同步)

- [参考](#参考)

# 线程与进程

以经典的进程、线程模型为例，用一句简单的话概括就是：进程是操作系统进行资源分配的基本单位，线程是程序运行和调度的基本单位。  

## 程序

程序只是指令、数据及其组织形式的描述，是一个静态的概念。  

## 进程

进程是程序的真正运行实例，是一个动态的概念。每一个进程都有它自己的地址空间，一般情况下，包括文本区域（text region）、数据区域（data region）和堆栈（stack region）。在面向进程设计的系统（早期的 UNIX、Linux 2.4 及更早的版本）中，进程是程序的基本执行实体；在面向线程设计的系统（现代操作系统、Linux2.6 及以后的版本）中，进程不再是基本的运行单位，而是线程的容器。  

程序运行就会产生进程。同一程序可产生多个进程以允许同时有多位用户运行同一程序，却不会互相冲突。  

- **独立性**  
进程与进程之间相互独立，一个进程不能访问另一个进程的资源，但是由于进程都是抢占的系统资源，所以进程与进程也存在竞争关系。  
- **并发性**  
进程能够并发运行。从宏观上看，进程似乎是同时运行的，但是实际上，同一时刻，单个核心只能执行一个进程，只不过 CPU 处理速度很快，通过进程切换使人感觉多个进程在同时运行。  

## 线程
在之前的操作系统中，拥有资源，能够独立运行的基本单位是进程，但是由于进程创建、撤消与切换存在较大的时空开销，同时由于 SMP 的出现，多个进程并行开销较大，因此急需一种轻型进程，也就是后续的线程。  

线程是操作系统能够进行运算调度的最小单位。它被包含在进程之中，是进程中的实际执行单位。  

- **轻量实体**  
线程自己基本上不拥有系统资源，只拥有一点在运行中必不可少的资源，如程序计数器、一组寄存器和栈。  
- **独立运行**  
线程是能独立运行的基本单位，由于线程很“轻”，线程的切换非常迅速且开销小（在同一进程中的）。  
- **并发执行**  
线程可以并发执行。
- **共享资源**  
在同一个进程下的各个线程共享进程资源，同时线程之间的通信要比进程通信更加容易。  

## 并发
当有多个线程在执行时，如果系统只有一个 CPU，则它根本不可能真正同时进行一个以上的线程，它只能把 CPU 运行时间划分成若干个时间段，再将时间段分配给各个线程执行，某个时间段的某个线程代码运行时，其它线程处于挂起状态。  

## 并行
当系统有一个以上 CPU 时，一个 CPU 执行一个线程，另一个 CPU 可以执行另一个线程，两个线程互不抢占 CPU 资源，可以同时进行。  

## 程序运行
一个程序的运行，如果程序中的代码没有创建多个线程，则为单线程程序，程序由这条单线程执行，而这个程序的运行就是一个进程，系统为该进程分配资源，因此进程是高于线程的存在，进程在程序加载到内存中时为这个进程的开始；如果程序中的代码创建了多个线程（由 Java 多线程可知，多线程是在 main 主线程中创建的），那么多个线程共享系统为进程分配的资源。  

Java 程序的运行，由 Java 命令启动 JVM，JVM 的启动就相当于创建出了一个进程，由该进程创建出一个主线程去调用 java 代码中的 main 方法（JVM 的启动，最低启动了两个线程：主线程和垃圾回收线程）。  

# 线程的实现

实现线程主要有三种方式：使用内核线程的实现、使用用户线程的实现和使用用户线程 + 轻量级进程混合实现。  

## 使用内核线程实现

内核线程（Kernel-Level Thread，KLT）就是直接由操作系统内核支持的线程，这种线程由内核完成线程切换，内核通过操纵调度器（Scheduler）对线程进行调度，并负责将线程的任务映射到各个处理器上。  

程序一般不会直接使用内核线程，而是使用内核线程的一种高级接口——轻量级进程（Light Weight Process，LWP），这种轻量级进程与内核线程之间是 1:1 的关系。  

![内核线程](https://github.com/nekolr/java-notes/blob/master/images/Java%20并发/kxy.png)

- **优点**  
由于内核线程的支持，每个轻量级进程都成为一个独立的调度单元，即使有一个轻量级进程在调用时阻塞了，也不会影响整个进程继续工作。  

- **缺点**  
由于是基于内核线程实现的，所以各种线程操作，如创建、析构及同步，都需要进行内核的调用，而内核调用的代价相对较高，需要在用户态（User Mode）和内核态（Kernel Mode）间来回切换。并且由于每个轻量级进程都需要一个内核线程支持，所以一个系统支持的轻量级进程的数量是有限的。  

## 使用用户线程实现

广义上讲，一个线程只要不是内核线程就可以认为是用户线程（User Thread，UT），因此广义上轻量级进程也属于用户线程。狭义上的用户线程指的是完全建立在用户空间的线程，系统内核无法感知到用户线程的存在。用户线程的操作，如创建、同步、销毁和调度完全在用户态中完成，不需要内核帮助。  

![用户线程](https://github.com/nekolr/java-notes/blob/master/images/Java%20并发/8rQ.png)

- **优点**  
由于不需要在用户态和内核态间来回切换，因此操作非常快速且消耗低，并且支持的线程数量也相对较大。  

- **缺点**  
由于没有内核的帮助，线程所有的操作都需要用户程序自己处理，并且由于操作系统只把资源分配给进程，诸如“阻塞如何处理”、“多处理器系统中如何将线程任务映射到各个处理器”等这类问题处理比较困难。这些问题导致了现在单纯使用用户线程的程序越来越少了。  

## 使用用户线程和轻量级进程混合实现

在这种实现中，用户线程还是完全建立在用户空间，因此线程的创建、析构、切换等操作依然快速且消耗低，并且支持的线程数量较大。操作系统提供轻量级进程作为用户线程和内核线程的桥梁，这样可以使用内核提供的线程调度以及处理器映射，用户线程的调度也交由轻量级进程处理，这大大降低了进程阻塞的风险。在这种混合模式中，用户线程与轻量级进程的数量比是不定的，即为 N:M 的关系。许多 `UNIX` 系统，如 `Solaris`、`HP-UX` 等都提供了 N:M 的线程模型。  

![混合](https://github.com/nekolr/java-notes/blob/master/images/Java%20并发/NXw.png)

## Java 线程的实现

Java的线程在 `JDK 1.2` 之前是基于称为“绿色线程”（Green Thread）的用户线程实现的，在 `JDK 1.2` 线程模型替换成了基于操作系统原生线程模型来实现。因此，在目前的 JDK 版本中，不同的操作系统下，可能实现的方式不同。对于 `Sun JDK` 来说，在 `Windows` 和 `Linux` 系统中都是使用一对一的线程模型实现的，即一条 Java 线程映射到一个轻量级进程中。在 `Solaris` 平台中，由于操作系统支持一对一以及多对多的线程模型，所以在 `Solaris` 版的 JDK 提供了两个平台专有的虚拟机参数：`-XX:+UseLWPSynchronization` 和 `-XX:+UseBoundThreads` 来切换使用的线程模型。  

## 线程调度

线程调度是系统为线程分配处理器使用权的过程，主要的调度方式有两种，协同式线程调度和抢占式线程调度。  

### 协同式线程调度

使用协同式线程调度的多线程系统，线程的执行时间由线程本身控制。线程将自己的工作做完后，要主动通知系统切换到另一个线程。  

- **优点**  
实现简单，由于需要线程做完工作后主动通知系统进行线程切换，所以没有线程同步问题。  

- **缺点**  
线程执行时间不可控，如果一个线程编写有问题，一直不告知操作系统进行线程切换，那么程序将会阻塞，严重的可能会造成系统崩溃。  

### 抢占式线程调度

使用抢占式线程调度的多线程系统，每个线程由操作系统分配执行时间，线程切换不由线程来决定（Java 中，可以通过 `Thread.yield()` 让出执行时间，当然不一定管用）。  

- **优点**  
线程执行时间是确定的、可控的，因此不会有一个线程导致整个程序阻塞的问题。  

- **缺点**  
需要考虑线程同步问题。  

### Java 的线程调度

Java 的线程调度使用的是抢占式。  

虽然 Java 的线程调度是由操作系统自动完成的，但是我们还是可以“建议”操作系统给某些线程多一点执行时间，另外一些线程少一点执行时间，这个操作可以通过设置线程的优先级来完成。Java 提供了 10 个级别的线程优先级，在两个线程同时处于 `Ready` 状态时，优先级越高的越容易先被执行。不过，由于 Java 的线程是映射到原生线程上来实现的，所以最终的线程调度还是取决于操作系统，并且操作系统的线程优先级的数量可能和 Java 的不能一一对应（如 `Solaris` 中有 2^32 种优先级，而 `Windows` 中只有 7 种）。  

# 线程的状态

操作系统中的线程状态大致可以分为五种：新建、就绪、运行、阻塞和终止。由于 Java 的线程是映射到系统内核线程的，所以 Java 的线程状态与之类似。  

## Java 线程状态

```java
/**
*   Thread类中定义的枚举
**/
public enum State {
    NEW,

    RUNNABLE,

    BLOCKED,

    WAITING,

    TIMED_WAITING,

    TERMINATED;
}
```

### 新建（New）
新创建一个线程对象，表现为：`Thread thread = new Thread();`。  

### 就绪及运行（Runnable）
线程创建后，调用 `start()` 方法，该线程会处于就绪状态，等待被线程调度器选中，获取到 CPU 时间片。当线程获取到执行权，线程处于运行状态，执行 `run()` 方法。  

### 阻塞（Blocked）
线程由于某些原因放弃 CPU 执行权，让出时间片，暂时停止运行，此时线程处于阻塞状态，直到线程重新进入就绪态，才有机会再次获取 CPU 时间片运行。在 Java 线程中阻塞可以细分为三种：  

#### 同步阻塞
处在运行状态的线程在试图进入临界区时需要获取对象的互斥锁，当该锁已经被其他线程占用，操作系统会将程序的运行由用户态切换到内核态调用线程阻塞指令阻塞该线程，同时 JVM 会将该线程放入同步队列中等待。  

#### 无限期等待（Waiting）
处于这种状态的线程不会被分配 CPU 执行时间，需要等待其他线程显式唤醒后进入就绪态。 

- Object.wait() 方法
- Thread.join() 方法
- LockSupport.park() 方法

#### 超时等待（Timed Waiting）
处于这种状态的线程也不会被分配 CPU 执行时间，即使其他线程不显式唤醒，系统也会在超时时间结束时自动唤醒它们。  

- Thread.sleep(long millis) 方法
- Object.wait(long timeout) 方法
- Thread.join(long millis) 方法
- LockSupport.parkNanos() 方法
- LockSupport.parkUntil() 方法

### 终止（Terminated）
`run()` 方法结束或出现未捕获的异常，线程终止。  

## Java 线程状态转换

![thread_state](https://github.com/nekolr/java-notes/blob/master/images/Java%20并发/G3d.png)

# 线程控制与线程通信基础

## 线程控制
线程控制包括线程的开启、暂停、中断，以及设置守护线程。  

### 开启线程
开始一个线程使用线程的 `start()` 方法。  

### 暂停线程
暂停线程意味着线程还可以重新恢复，Java 提供了几种暂停的方法：`sleep()`、`join()`、`yield()`，以及 `wait()` 和 `notify()` 组合。  

#### sleep(long millis)
`public static native void sleep(long millis) throws InterruptedException`  

使当前正在执行的线程睡眠，不释放锁权限（monitor），睡眠结束后进入就绪态。虽然没有释放锁，但在醒来时其他线程可能正在运行，此时不会立马进入运行状态。  

#### join()、join(long millis)
`public final synchronized void join(long millis) throws InterruptedException`  

官方的解释是 **Waits for this thread to die.** 即等待这个线程死亡。  
举个例子会更好理解：比如在主线程 main 中调用某个线程 t 的 join() 的方法，主线程会一直等待线程 t 结束才开始执行。  

```java
public static void main(String[] args) throws InterruptedException {
    Thread t = new Thread(() -> {
        for (int i = 0; i < 1000; i++){
            System.out.println(Thread.currentThread().getName() + " i=" + i);
        }
    }, "thread-1");

    t.start();
    t.join();

    System.out.println(Thread.currentThread().getName());
}
```

```java
public final synchronized void join(long millis)
throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (millis == 0) {
        while (isAlive()) {
            wait(0);
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);
            now = System.currentTimeMillis() - base;
        }
    }
}
```

就拿刚才的例子结合源代码来理解。join() 方法是一个同步方法，因此需要获取锁对象权限，在 main 线程中调用线程 t 的 join() 方法，即 main 线程获取到了 t 线程对象的内在锁（同步方法的锁对象即调用者，此处即 t 线程对象），在一个 while 循环中反复判断 t 线程是否存活，如果 t 线程还没结束，则调用 `wait()` 方法（相当于调用 t.wait() 方法），使持有 t 线程对象锁的 main 线程处于等待状态，直到 t 线程结束，此时线程 t 会自动调用 `notifyAll()`（相当于调用 t.notifyAll() 方法）来唤醒主线程。  

#### yield()

`public static native void yield()`  

暂停当前线程。对调度程序暗示当前线程可以暂时停止让出 CPU 占用，但是要取决于 CPU 的调度，CPU 可能会忽略这个暗示。  

### 中断线程
由于 Java 无法立即停止一个线程，而终止操作很重要，所以 Java 提供了一种用于停止线程的机制，即中断机制。  

- **中断状态**  
每个线程维护一个中断状态位，用来表明该线程是否被中断，默认为非中断 false。  
- **中断方法**  
中断方法仅仅只能设置和检测中断状态。  
- **中断过程**  
JDK只负责检测和更新中断状态，因此中断的处理过程需要开发者自己实现，包括中断捕获和处理。  

#### 中断方法

- `public static boolean interrupted()`  
判断当前线程是否中断，静态方法，会清除中断状态（设置为 false）。  
- `public boolean isInterrupted()`  
判断线程是否中断，对象方法，不会清除中断状态。  
- `public void interrupt()`  
中断操作，对象方法，会将线程的中断状态设置为 true，仅此而已。  

#### InterruptedException
一个方法声明中抛出 `InterruptedException` 异常，说明该方法执行可能需要花费一些时间等待，该方法可以使用 Thread 实例的 `interrupt()` 方法来中断。  
		
能够抛出 `InterruptedException` 异常的方法：  
- `java.lang.Object` 的 wait() 方法，需要其他线程通过 notify() 唤醒，可能会一直等待。
- `java.lang.Thread` 的 sleep(millis) 方法，等待设定的时间。
- `java.lang.Thread` 的 join() 方法，等待指定线程结束。  

只有在线程执行到这些能够抛出 `InterruptedException` 异常的方法时，线程调用 interrupt() 方法，这些方法才会中断并执行 catch 块中代码。  

#### 中断处理

几种常用的中断处理方式。  

- 异常法  
```java
Thread thread = new Thread(() -> {
    try {
        if(Thread.currentThread().isInterrupted()) {
            throw new InterruptedException();
        }
    } catch (InterruptedException e) {
        System.out.println("成功捕获中断异常");
        e.printStackTrace();
    }
});
thread.start();
thread.interrupt();
```

- 阻塞法  
有时候抛出 `InterruptedException` 并不合适，例如当由 `Runnable` 定义的任务调用一个可中断的方法时，就是如此。在这种情况下，不能重新抛出 `InterruptedException`，但是你也不想什么都不做。当一个阻塞方法检测到中断并抛出 `InterruptedException` 时，**它会清除中断状态**。如果捕捉到 `InterruptedException` 但是不能重新抛出它，那么应该保留中断发生的证据，以便调用栈中更高层的代码能知道中断，并对中断作出响应。  
```java
Thread thread = new Thread(() -> {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        System.out.println("成功捕获中断异常");
        // 重新设置中断状态
        Thread.currentThread().interrupt();
    }
});
thread.start();
thread.interrupt();
```

- return法  
```java
Thread thread = new Thread(() -> {
    while (true) {
        if (Thread.currentThread().isInterrupted()) {
            System.out.println("线程停止");
            return;
        }
        System.out.println(System.currentTimeMillis());
    }
});
thread.start();
Thread.sleep(1000);
thread.interrupt();
```

### 设置守护线程

`public final void setDaemon(boolean on)`  
		
设置某个线程为守护线程，守护线程（后台线程）需要等待所有前台线程结束后才会结束（所有前台线程结束后，无论后台线程是否处在运行中，都会被强制结束）。例如：操作系统会为 JVM 创建一个进程，JVM 会启动一个前台线程来运行 main 方法和一个后台线程来运行 GC。  
		
在使用时，只要在创建线程后调用该方法，传入 true 即可将该线程设置为守护线程。  

```java
Thread thread = new Thread(() -> {
    for (int i = 0; i < 1000; i++)
        System.out.println("i=" + i);
});
thread.setDaemon(true);
thread.start();

System.out.println("main");
```

循环打印可能等不到全部打印就结束了，因为 main 线程只打印了一句，很快执行完后线程结束，此时守护线程也会被强制结束。  

## 线程通信基础

`wait()`、`notify()` 和 `notifyAll()` 这些方法常用来进行线程间通信，它们必须在同步代码块或同步方法中使用。  

### wait()
`public final native void wait() throws InterruptedException`  

调用 wait() 方法后，持有对象锁的线程释放对象锁权限，释放 CPU 执行时间并进入等待状态，等待其他线程唤醒。  

### wait(long timeout)
`public final native void wait(long timeout) throws InterruptedException`  

在超时时间内，可以被其他线程使用 notify() 或 notifyAll() 来唤醒，或者在超时后会由系统自动唤醒。如果超时时间为 0，等同于 wait() 方法。  

### notify()
`public final native void notify()`  

持有对象锁的线程释放对象锁权限，通知 jvm 唤醒某个竞争该对象锁的线程。其他竞争线程继续等待（即使该同步结束，释放对象锁，其他竞争线程仍然等待，直至有新的 notify() 或 notifyAll() 被调用）。  

### notifyAll()
`public final native void notifyAll()`  

持有对象锁的线程释放对象锁权限，通知 jvm 唤醒所有竞争该对象锁的线程，jvm 通过算法将对象锁交给某个线程，所有被唤醒的线程不再等待，而是一起竞争该对象锁权限。  

# 线程安全与线程安全的实现
重点讨论线程安全问题以及如何解决线程安全问题。  

## Java中的线程安全
线程安全问题出现的原因是多个线程访问共享数据造成的。  

按照线程安全的“安全程度”由强到弱，可以将 Java 中的线程安全分为五类：不可变、绝对线程安全、相对线程安全、线程兼容和线程对立。  

### 不可变

在 Java 中，不可变对象一定是线程安全的，无论是对象的方法还是方法的调用者，都不需要再采取任何保障线程安全的措施。如果共享数据是基本数据类型，那么只要在定义时使用 `final` 关键字修饰就可以保证它是不可变的。如果共享数据是一个对象，那就需要保证对象的行为（方法）不会改变它的状态（属性）。  

典型的不可变类型，比如 `java.lang.String`。  

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];

    ...

    public String substring(int beginIndex) {
        if (beginIndex < 0) {
            throw new StringIndexOutOfBoundsException(beginIndex);
        }
        int subLen = value.length - beginIndex;
        if (subLen < 0) {
            throw new StringIndexOutOfBoundsException(subLen);
        }
        return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
    }
}
```

String 类使用 `final` 修饰，意味着该类不可继承，这也就防止了子类在继承后重写方法改变其状态。String 的值是存放在一个名为 `value` 的数组中，数组用 `final` 修饰，但是这只能保证 `value` 的引用地址值不可变，引用指向的数组却可以发生变化，如：  

```java
final int value[] = {1, 2, 3};
int another[] = {4, 5, 6};
value = another; // 编译报错，final 修饰，引用地址值不可变
value[1] = 2333; // 直接修改数组的元素值
```

所以真正保证 String 不可变靠的是它的 `replace()`、`substring()`、`concat()`、`trim()` 等方法都不会修改它原来的值，而是返回一个新构造的字符串对象。在 Java 中不可变的类型，除了 String 外，还有基本类型对应的包装类型，以及枚举类型、`BigInteger` 和 `BigDecimal` 类型等。  

### 绝对线程安全

虽然在文中将线程安全按“安全程度”划分了五类，而实际上不可变可以看作绝对线程安全的一种。除了不可变类型，其他类要达到“不管运行环境如何，调用者都不需要任何额外的保证线程安全的措施”通常是不太可能的。Java 中通常所说的线程安全的类，大多不是绝对线程安全的，比如某些集合类：`java.util.Vector`、`java.util.HashTable` 等等。  

### 相对线程安全

相对线程安全就是我们通常意义上讲的线程安全，它保证对象的方法都是线程安全的，但是在调用者调用时，某些特殊的调用操作需要在调用端使用额外的保证线程安全的措施。比如：`java.util.Vector`、`java.util.HashTable`、`Collections.synchronizedCollection()` 方法包装的集合等。  

### 线程兼容

线程兼容是指对象本身不是线程安全的，但是可以通过在调用端使用同步手段来保证对象在并发环境中安全使用。我们通常所说的一个类不是线程安全的，绝大多数是属于这种情况，比如：`java.util.ArrayList`、`java.util.HashMap` 等。  

### 线程对立

线程对立是指无论调用端是否使用了同步手段，都无法保证在多线程环境下正确运行。这种情况是很少的，并且通常是有害的，应当尽量避免。常见的线程对立的操作有：`System.setIn()`、`System.setOut()` 等。  
## 线程安全的实现方法
除了不可变外，实现线程安全可以通过互斥同步、非阻塞同步以及无同步的可重入代码和线程本地变量来实现。  

### 互斥同步

互斥同步是一种常见的并发正确性保障手段。并发的核心矛盾是“竞态条件”，即多个线程同时访问共享变量，这个共享变量也可以叫做“竞态资源”，而涉及访问竞态资源的代码片段称为“临界区”。  

保障竞态资源安全的一个思路就是让临界区的代码“互斥”，即同一时刻最多只能有一个线程进入临界区，而保证同一时刻只有一个线程进入临界区的锁被称为互斥量（Mutex）。有的时候，临界区可以允许 N 个线程进入，这个时候可以把互斥量的概念推广，引入新的锁概念：信号量（Semaphore）。  

#### synchronized
在 Java 中，最基本的互斥同步手段就是 `synchronized` 关键字。`synchronized` 被编译后，会在同步代码块的前后分别形成 `monitorenter` 和 `monitorexit` 两个字节码指令，这两个字节码指令都需要一个引用类型的参数来指明要锁定和解锁的对象。如果代码中没有明确指定锁对象，那么将会根据 `synchronized` 修饰的是实例方法还是类方法，取对应的实例对象或Class对象作为锁对象。  

> 注：同步方法编译后会在常量池中存储 `ACC_SYNCHRONIZED` 标记，方法调用时，调用指令检查方法的 `ACC_SYNCHRONIZED` 是否被设置来实现同步。  

使用 `javap` 分析查看同步方法的字节码：  

```java
public class Demo {

    public synchronized void doSomething() {
        System.out.println("doSomething");
    }
}
```

![同步方法](https://github.com/nekolr/java-notes/blob/master/images/Java%20并发/lrv.png)

根据虚拟机规范的要求，在执行 `monitorenter` 指令时，首先尝试获取对象的锁，如果对象的锁没被占用，或者当前线程已经拥有了对象的锁，则把锁的计数器加 1，此时其他线程试图进入临界区时，操作系统会将程序的运行由用户态切换到内核态来执行内核指令（阻塞其他线程的指令，由于 Java 的线程是映射到系统原生线程上的，所以阻塞和唤醒一个线程都需要内核帮助）。在执行 `monitorexit` 指令时锁计数器减一，当计数器为 0 时，锁就被释放。  

> 注：`synchronized` 同步块对于同一个线程来说是可重入的，即同一个线程，在拿到了对象的锁权限后，可以多次进入临界区，每进入一次锁计数器加 1。与之相对的，在线程离开临界区时，需要释放对应次数的锁权限。  

#### ReentrantLock
除了 `synchronized` 之外，还可以使用 `java.util.concurrent.ReentrantLock` 来实现同步。`ReentrantLock` 是独占互斥可重入锁，与 `synchronized` 类似，但两者的实现不同。`synchronized` 依赖的是监视器 `Monitor`，而 `ReentrantLock` 依赖的是 `AbstractQueuedSynchronizer`。并且，`ReentrantLock` 还增加了一些高级功能，如支持响应中断、可实现公平锁、锁可以绑定多个条件等。在后续的 JDK 版本中，加入了很多针对锁的优化措施，在 `JDK 1.6` 发布之后，`synchronized` 与 `ReentrantLock` 的性能基本持平了。  

### 非阻塞同步
互斥同步最主要的问题就是进行线程阻塞和唤醒所带来的性能问题，因此这种同步也被称为阻塞同步，互斥同步属于一种悲观的并发策略，随着硬件指令集的发展，我们有了另外一个选择：基于冲突检测的乐观并发策略。通俗的讲，就是先进行操作，如果没有其他线程争用共享数据，则操作成功；如果共享数据有争用，产生了冲突，再采用其他的补偿措施（常见的就是不断的重试，直到成功）。由于这种乐观的并发策略不需要将线程挂起，所以这种同步操作被称为非阻塞同步。  

乐观并发策略需要硬件指令集的发展来支持的原因是，我们需要操作和冲突检测这两个步骤具备原子性，这里不能再使用互斥同步，只能靠硬件来完成，由硬件保证一个从语义上看起来需要多次操作的行为只通过一条处理器指令就能完成，常用的有：  

- 测试并设置（Test and Set）
- 获取并增加（Fetch and Increment）
- 交换（Swap）
- 比较并交换（Compare and Swap，CAS）
- 加载链接/条件存储（Load Linked/Store Conditional，LL/SC）

其中，前三个在很早之前就已经存在于大多数指令集中，后面的两条是现代处理器新增的，这两条指令的目的和功能类似。`CAS` 指令需要3个操作数，分别是内存位置（在 Java 中可以简单理解为变量的内存地址，用 V 表示）、旧的预期值（用 A 表示）和新值（用 B 表示）。`CAS` 指令执行时，当且仅当 V 符合预期值 A 时，处理器才会用新值 B 去更新 V 的值，否则不更新。无论更新与否，返回 V 的旧值。这整个操作是一个原子操作。  

下面演示用一个例子来演示非阻塞同步的使用。  

我们知道 `volatile` 关键字解决了变量的可见性问题，但是无法保证变量的原子性，如：  

```java
public class VolatileTest {

    private static volatile int race = 0;

    private static final int THREADS_COUNT = 20;

    public static void increase() {
        race++;
    }

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[THREADS_COUNT];

        for (int i = 0; i < THREADS_COUNT; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    increase();
                }
            });
            threads[i].start();
        }

        for (int j = 0; j < THREADS_COUNT; j++)
            threads[j].join();

        System.out.println(race);
    }
}
```

这段代码，理想的结果应该是打印出 20000，然而实际结果经常小于 20000。反汇编查看：  

![反汇编代码](https://github.com/nekolr/java-notes/blob/master/images/Java%20并发/dQ8.png)

`increase()` 方法中的 `race++` 并不是原子操作。  
1. `getstatic` 指令获取类的静态变量，并将其压入栈顶。  
2. `iconst_1` 指令将一个值为 1 的 int 型推至栈顶。  
3. `iadd` 指令将栈顶的两个 int 型数值相加并将结果压入栈顶。  
4. `putstatic` 指令将为指定的类的静态变量赋值。  
5. `return` 指令从当前方法返回 void。  

在某个线程中，当 `getstatic` 指令将 `race` 的值压入栈顶时，`volatile` 关键字保证了此时的 `race` 值是正确的，但是在执行 `iconst_1` 和 `iadd` 指令时，其他线程可能已经把 `race` 的值增加了，此时当前线程栈顶的值就变成了过期的值，所以 `putstatic` 指令执行后就有可能将较小的 `race` 值同步回了主内存中。  

那么怎么修改这段代码呢？当然用 `synchronized` 修饰 `increase()` 方法是可行的，但是这里演示如何通过 `CAS` 来处理。  

```java
public class VolatileTest {

    private static AtomicInteger race = new AtomicInteger(0);

    private static final int THREADS_COUNT = 20;

    public static void increase() {
        race.incrementAndGet();
    }

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[THREADS_COUNT];

        for (int i = 0; i < THREADS_COUNT; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    increase();
                }
            });
            threads[i].start();
        }

        for (int j = 0; j < THREADS_COUNT; j++)
            threads[j].join();

        System.out.println(race);
    }
}
```

使用 `AtomicInteger` 的 `incrementAndGet()` 方法能够实现原子操作的自增。查看该方法的实现：  

```java
private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();
private static final long VALUE = U.objectFieldOffset(AtomicInteger.class, "value");

public final int incrementAndGet() {
    return U.getAndAddInt(this, VALUE, 1) + 1;
}
```

```java
@HotSpotIntrinsicCandidate
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);
    } while (!weakCompareAndSetInt(o, offset, v, v + delta));
    return v;
}
```

这里分析的是 `JDK 10` 的实现，`JDK 8` 的实现与之类似，唯一的差别就是在 `JDK 8` 中使用的是 `compareAndSetInt`。  

通过 `U.objectFieldOffset(AtomicInteger.class, "value")` 方法获取的是 `AtomicInteger` 对象中的 `value` 字段相对于对象的起始内存地址的字节偏移量。  

> 注：一个 Java 对象可以看成一段内存，每个字段按照一定的顺序放入这段内存中——对象的布局。考虑到内存对齐的要求，可能这些字段不是连续放置的，使用 `objectFieldOffset()` 方法能够计算出某个字段相对于对象的起始内存地址的字节偏移量，同时 `Unsafe` 类还提供了 `getInt()`、`getLong()`、`getObject()` 等方法使用偏移量来访问某个字段。  

`getIntVolatile()` 方法通过偏移量获取 `AtomicInteger` 对象的 `value` 值，与 `getInt()` 不同，它支持 `volatile load` 语义。然后循环执行 `weakCompareAndSetInt()` 方法试图将新值赋给 `value`，直到更新成功为止，最终返回旧值 + 1。  

### 无同步
无同步即不使用同步控制，一种方式是设计成可重入的代码，另一种就是使用 JDK 提供的线程本地变量。  

#### 可重入代码
线程同步保证了共享数据争用时的正确性，如果一个方法本来就不涉及共享数据，也就无需同步去保证线程安全，因此有一些天生就是线程安全的代码，这种代码有一个很重要的特征：方法返回的结果可以预测，即只要输入参数相同，返回结果必然相同。  

#### 线程本地存储
在 Java 中，能够进行线程本地存储的为 `java.lang.ThreadLocal`，通常称它为线程局部变量，使用它能够将本来由线程共享的变量在每个线程中分别存放一份副本，这样每个线程操作的就是自己的那个副本，从而达到线程隔离的目的。  

# 参考

> 《深入理解Java虚拟机:JVM高级特性与最佳实践》  
[Java 理论与实践 : 处理 InterruptedException](https://www.ibm.com/developerworks/cn/java/j-jtp05236.html)  
[阮一峰：进程与线程的一个简单解释](http://www.ruanyifeng.com/blog/2013/04/processes_and_threads.html)