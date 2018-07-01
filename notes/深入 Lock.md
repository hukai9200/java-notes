- [深入 Lock](#深入-lock)
    - [synchronized 的缺陷](#synchronized-的缺陷)
    - [Lock 接口](#lock-接口)
        - [lock](#lock)
        - [lockInterruptibly](#lockinterruptibly)
        - [tryLock](#trylock)
        - [unlock](#unlock)
        - [newCondition](#newcondition)
    - [LockSupport（JDK 1.8）](#locksupportjdk-18)
        - [几个关键属性](#几个关键属性)
        - [许可](#许可)
        - [几个重要的方法](#几个重要的方法)
            - [setBlocker() 和 getBlocker()](#setblocker-和-getblocker)
            - [park()](#park)
            - [unpark()](#unpark)
            - [LockSupport 的特性](#locksupport-的特性)

# 深入 Lock
除了基于对象监视器锁实现的 `synchronized`，还有基于 `AbstractQueuedSynchronizer` 实现的锁 `Lock`。  

# synchronized 的缺陷

- **不可中断**  
不能中断正在等待获取锁的线程。  
- **不可超时**  
在请求锁失败后，会无限等待（自旋超时后会进入阻塞状态）。  
- **自动释放锁**  
一般情况下，自动释放锁算是一个优点，相比手动释放锁代码要更优雅，并且不用担心因为忘记手动释放锁而造成死锁。但是如果一个获取到锁的线程因为等待 IO 或者其他原因而阻塞了，但是因为线程方法没有结束，也没有发生异常，所以对象锁是没有释放的，这时其他线程只能干等。因此有时手动释放锁更具灵活性。  
- **不可伸缩**  
无法细粒度控制锁，没有读锁和写锁的分类。比如多个线程读写文件时，读操作和写操作、写操作和写操作会发生冲突，而读操作和读操作不会发生冲突，而使用内部锁时，一个线程在做读操作，其他同样做读操作的线程只能等待。  
- **性能问题**  
在没有线程竞争的情况下，由于 JDK1.6 优化的原因，内部锁的性能与 `ReentrantLock` 的性能基本持平。一旦有线程竞争时，内部锁最终会膨胀为重量级锁，性能严重下降。  

# Lock 接口

![Lock接口](https://github.com/nekolr/java-notes/blob/master/images/Java%20并发/Jay.png)

## lock
获取锁的方法。  

```java
/**
* 获取锁。
*
* 如果锁不可用（被占用），则当前线程会休眠，直到获取到锁。
*
* 该方法的实现可能能够检测到锁的错误使用，比如死锁，并且可能会在这种情况下抛出
* 未检查的异常。此时该实现需要用文档注明这种情况以及其可能出现的异常。
*/ 
void lock();
```

## lockInterruptibly
可中断的锁获取操作，可以在锁获取的过程中断当前获取锁的线程。  

```java
/**
* 可中断地获取锁，即在获取锁的过程中可以中断当前线程。
*
* 如果锁可用会立即返回；
* 如果锁不可用，则当前线程会处于休眠状态，直到发生下面两种情况之一会返回：
* 1 锁被当前线程获取；
* 2 其他线程中断当前线程。
* 
* 在一些实现中中断获取锁是不可能的，即使可能的话也可能是昂贵的操作。
*/
void lockInterruptibly() throws InterruptedException;
```

## tryLock
`tryLock()` 方法提供可定时和可轮询的锁获取方式。可定时是指在指定的时间内获取锁，在超时时间结束立即返回。可轮询是指程序可以通过循环配合 `tryLock()` 来不断尝试获取锁。  

```java
/**
* 获取锁，如果锁可用立即返回 true；如果锁不可用，立即返回 false。
* 
* 下面这种用法可以确保解锁发生在获取锁后，如果没有获取到锁，则不会尝试解锁。
* Lock lock = ...;
* if (lock.tryLock()) {
*   try {
*     // manipulate protected state
*   } finally {
*     lock.unlock();
*   }
* } else {
*   // perform alternative actions
* }}
* 
*/
boolean tryLock();
```

```java
/**
* 如果在给定的时间内空闲并且当前线程未被中断，则获取锁。
*
* 如果锁可用，则立即返回 true；
* 如果锁不可用，则当前线程会处于休眠状态，直到发生下面三种情况之一会返回：
* 1 当前线程在超时时间内获取到锁，返回 true；
* 2 其他线程中断当前线程；
* 3 指定的时间已过，如果获取到锁，则返回 true；如果还是没有获取到锁，返回 false。
*
*/
boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
```

## unlock
释放锁的方法。  

```java
/**
* Releases the lock.
* 释放锁
*/
void unlock();
```

## newCondition
`Condition` 可以实现更为灵活的锁获取与释放的条件控制。  

```java
/**
* Returns a new {@link Condition} instance that is bound to this
* {@code Lock} instance.
*
* 返回一个绑定到 Lock 实例的新的 Condition 实例。
* 在调用等待方法之前，当前线程必须持有锁。调用 Condition.await() 将使线程释放锁，
* 并在等待返回之前重新获取锁。
*/
Condition newCondition();
```

## Lock 接口的使用方式
加锁需要与解锁成对出现。  

```java
Lock lock = ...;
lock.lock();
try {
    // do somethings
} finally {
    lock.unlock();
}
```

# LockSupport（JDK 1.8）
`LockSupport` 是一个线程阻塞工具类，能够在线程内任意位置让线程阻塞和释放，底层使用 `Unsafe` 实现，我们通常不会直接使用它，而是在锁实现中作为阻塞工具使用。  

这个类与每个使用它的线程通过 permit 相关联，这个 permit 从某种意义上可以理解为 Semaphore 类。  

## 几个关键属性

```java
// Hotspot implementation via intrinsics API
private static final sun.misc.Unsafe UNSAFE;
private static final long parkBlockerOffset;
private static final long SEED;
private static final long PROBE;
private static final long SECONDARY;
static {
    try {
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> tk = Thread.class;
        // 这里是获取 Thread 类中 parkBlocker 字段的偏移量
        parkBlockerOffset = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("parkBlocker"));
        SEED = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomSeed"));
        PROBE = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomProbe"));
        SECONDARY = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomSecondarySeed"));
    } catch (Exception ex) { throw new Error(ex); }
}
```

`LockSupport` 中有个重要的属性 `parkBlockerOffset`，它在类加载时通过 Unsafe 类的 `objectFieldOffset()` 方法获取到 Thread 类中 parkBlocker 字段的偏移量，拿到偏移量后，可以通过 `UNSAFE.putObject()` 给该字段赋值。  

> 注：parkBlocker 是阻塞对象，即表示线程被谁阻塞。  

## 许可
`LockSupport` 与每个使用它的线程通过 permit 关联，这个 permit 从某种意义上可以认为是 Semaphore 类，但是区别是：permit 的值最多为 1，即重复调用 `unpark()` 也不会累加。  

当调用 `unpark()` 时，permit 加 1；调用 `park()` 方法，permit 被消费掉，此时 permit = 0，再次调用 `park()`，该线程会被阻塞，因为没有许可可以使用。  

## 几个重要的方法
包括 `setBlocker()`、`getBlocker()`、`park()` 和 `unpark()` 方法。  

### setBlocker() 和 getBlocker()

```java
private static void setBlocker(Thread t, Object arg) {
    // Even though volatile, hotspot doesn't need a write barrier here.
    UNSAFE.putObject(t, parkBlockerOffset, arg);
}

public static Object getBlocker(Thread t) {
    if (t == null)
        throw new NullPointerException();
    return UNSAFE.getObjectVolatile(t, parkBlockerOffset);
}
```

`setBlocker()` 方法使用 `UNSAFE.putObject(t, parkBlockerOffset, arg)`，表示通过 Unsafe 类的 `putObject()` 方法根据偏移量找到将线程 t 的 parkBlocker 字段，并赋给值 arg。  

`getBlocker()` 方法返回最近一次调用 `park()` 方法并且尚未解除阻塞的线程的 parkBlocker 对象，如果线程没有被阻塞则返回 null。  

### park()
该方法用于等待许可，在第一次使用时默认是没有许可的，也就是在使用该方法之前需要先使用 `unpark()` 方法。  

在调用 `park()` 时，当许可可用时，立即返回并消费掉该许可；当许可不用时，当前线程被阻塞，直到发生下面三种情况之一：  
- 其他线程调用 `unpark()` 方法给当前线程提供许可。
- 其他线程中断当前线程。
- 虚假调用（无理由）返回。  

```java
public static void park(Object blocker) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 设置阻塞对象
    setBlocker(t, blocker);
    // 阻塞当前线程
    UNSAFE.park(false, 0L);
    // 清空阻塞对象值
    setBlocker(t, null);
}

public static void parkNanos(Object blocker, long nanos) {
    if (nanos > 0) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        // 阻塞 nanos 纳秒后返回
        UNSAFE.park(false, nanos);
        setBlocker(t, null);
    }
}

public static void parkUntil(Object blocker, long deadline) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    // 如果想实现从现在开始 3000 毫秒后返回，则 deadline 的值为
    // System.currentTimeMillis() + 3000
    UNSAFE.park(true, deadline);
    setBlocker(t, null);
}
```

## unpark()
该方法在线程尚没有得到许可时，为该线程提供许可。如果线程被 `park()` 方法阻塞，那么该方法将解除阻塞。  

```java
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```

## LockSupport 的特性
`LockSupport` 不支持可重入，可以响应中断，但是不会抛出中断异常。  

```java
public static void main(String[] args) {
    Thread currentThread = Thread.currentThread();
    // 提供许可 permit + 1 = 1
    LockSupport.unpark(currentThread);
    // permit 的值不变
    LockSupport.unpark(currentThread);
    // 第一次调用能够执行，permit - 1 = 0
    LockSupport.park();
    System.out.println("执行第一次 park");
    // 线程会一直阻塞在该方法，因为此时 permit = 0
    LockSupport.park();
    System.out.println("执行第二次 park");
}
```

```java
public static void main(String[] args) {
    Thread thread = new Thread(() -> LockSupport.park(), "thread-1");
    thread.start();
    thread.interrupt();
}
```
