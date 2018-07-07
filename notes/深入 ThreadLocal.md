- [深入 ThreadLocal](#深入-threadlocal)
    - [实现思路简单分析](#实现思路简单分析)
    - [ThreadLocalMap 的结构](#threadlocalmap-的结构)
    - [如何计算索引](#如何计算索引)
    - [从使用角度分析](#从使用角度分析)
        - [set 方法](#set-方法)
        - [get 方法](#get-方法)
        - [remove 方法](#remove-方法)
    - [为什么要使用弱引用](#为什么要使用弱引用)
    - [内存泄露问题](#内存泄露问题)
    - [InheritableThreadLocal](#inheritablethreadlocal)

# 深入 ThreadLocal

`ThreadLocal` 可以理解成线程局部变量，即每个线程都可以有一个或多个 `ThreadLocal`作为私有变量使用，每个线程向 `ThreadLocal` 中读写都是线程隔离的，`ThreadLocal` 提供了一种将共享数据通过每个线程独立的副本来实现线程封闭的机制。  

## 实现思路简单分析

Thread 类有一个类型为 `ThreadLocal.ThreadLocalMap` 的实例变量 `threadLocals`，也就是说每个线程都有一个自己的 `ThreadLocalMap`。`ThreadLocalMap` 是 `ThreadLocal` 的静态内部类，内部结构有点像 Map，它的 key 可以简单视作 `ThreadLocal`（实际上 key 不是 ThreadLocal 本身，而是它的一个弱引用），value 则为通过代码放入的值。每个线程在往某个 `ThreadLocal` 中放入值的时候，其实都会向自己的 `ThreadLocalMap` 中里存；读取值是通过也是通过某个 `ThreadLocal` 为 key，在自己的 `threadLocals` 里找对应的 value。  

```java
public class Thread implements Runnable {
    
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

## ThreadLocalMap 的结构
`ThreadLocalMap` 的结构与 HashMap 的结构有一部分相似。  

```java
static class ThreadLocalMap {

    /**
    * talbe 中存放的元素的类型
    */
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        // ThreadLocal 实例作为 key，外界提供的值作为 value
        Entry(ThreadLocal<?> k, Object v) {
            // 实际将 ThreadLocal 引用作为参数构造了一个弱引用
            super(k);
            value = v;
        }
    }

    /**
    * The initial capacity -- MUST be a power of two.
    * 初始容量，必须是 2 的幂，原因参考 HashMap
    */
    private static final int INITIAL_CAPACITY = 16;

    /**
    * HashMap 中也有一个 table，用来存放 Node 节点，这里的
    * table 存放的是 Entry 类型
    */
    private Entry[] table;

    /**
    * The number of entries in the table.
    * table 中元素的个数
    */
    private int size = 0;

    /**
    * The next size value at which to resize.
    * 扩容的阈值
    */
    private int threshold; // Default to 0

}
```

## 如何计算索引
既然是 `ThreadLocalMap` 是类 Map 的结构，那么肯定需要通过 key 来计算索引位置。它通过以下代码计算索引：  

```java
int i = key.threadLocalHashCode & (table.length - 1);
```

其中，`threadLocalHashCode` 的值通过如下代码获取。  

```java
public class ThreadLocal<T> {

    private final int threadLocalHashCode = nextHashCode();

   
    private static AtomicInteger nextHashCode =
        new AtomicInteger();

   
    private static final int HASH_INCREMENT = 0x61c88647;

    /**
    * 每次构造 ThreadLocal 就会生成一个 threadLocalHashCode，
    * 它的值通过将上一次的 threadLocalHashCode 值加上一个魔数
    * 0x61c88647 生成
    */
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }

}
```

`ThreadLocal` 类中有一个 `final` 修饰的成员变量 `threadLocalHashCode`，它在 `ThreadLocal` 被构造时就会生成，它的值是在上一个 `ThreadLocal` 的 `threadLocalHashCode` 值的基础上加上一个魔数 0x61c88647 得到的。这个魔数的选取与斐波那契散列有关，0x61c88647 对应的十进制为 1640531527。斐波那契散列的乘数可以用 (long) ((1L << 31) * (Math.sqrt(5) - 1)) 得到值 2654435769，如果把这个值给转为带符号的整型则是 -1640531527。换句话说，(1L << 32) - (long) ((1L << 31) * (Math.sqrt(5) - 1)) 得到的结果就是 1640531527 也就是 0x61c88647。通过理论与实践（不是我），当我们用 0x61c88647 作为魔数累加为每个 `ThreadLocal` 分配各自的 `threadLocalHashCode`，然后再与 2 的幂取模，得到的结果分布很均匀。  

对于 2 的幂作为模数取模，可以用 &(2^n - 1) 来替代 %2^n，位运算比取模效率高很多。  

> 这里为什么说 `threadLocalHashCode` 是在上一个 `ThreadLocal` 的 `threadLocalHashCode` 值的基础上加上一个魔数 0x61c88647 得到的呢？因为 `nextHashCode` 是类变量。  

## 从使用角度分析

```java
ThreadLocal<Integer> threadLocal = new ThreadLocal<>();
// 放入值
threadLocal.set(2333);
// 获取值
threadLocal.get();
// 删除值
threadLocal.remove();
```

### set 方法

```java
public void set(T value) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 拿到线程的 threadLocals
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // 放入值，this 为当前的 ThreadLocal 实例
        map.set(this, value);
    else
        // 创建一个 ThreadLocalMap
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

先查看它是如何创建一个 `ThreadLocalMap` 的。  

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    // 创建一个容量为 16 的 Entry 数组
    table = new Entry[INITIAL_CAPACITY];
    // 计算索引
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    // 放入元素
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    // 计算阈值
    setThreshold(INITIAL_CAPACITY);
}

private void setThreshold(int len) {
    threshold = len * 2 / 3;
}
```

下面查看它是如何实现放入元素的。  

```java
/**
* Set the value associated with key.
*
* @param key the thread local object
* @param value the value to be set
*/
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    // 计算索引
    int i = key.threadLocalHashCode & (len-1);

    // 只要 e 不为空，说明存在哈希冲突，会采用线性探测法计算下一个索引
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        // 通过 get 方法获取弱引用关联的 ThreadLocal
        ThreadLocal<?> k = e.get();
        // 是同一个 ThreadLocal，则覆盖值
        if (k == key) {
            e.value = value;
            return;
        }
        // ThreadLocal 为空，则说明已经无效，需要替换
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    // 放入 entry
    tab[i] = new Entry(key, value);
    int sz = ++size;
    // 只要清理了 slot 或者 当前元素个数大于扩展的阈值，就进行 rehash
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

```java
/**
* 替换无效的 Entry
* 
* 参数 key 和 value 是需要放入的新的 Entry 的 key 和 value
* 参数 staleSlot 是发现的无效 Entry 的索引
*/
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    int slotToExpunge = staleSlot;
    // 往前遍历，查找无效的 Entry
    for (int i = prevIndex(staleSlot, len);
            (e = tab[i]) != null;
            i = prevIndex(i, len))
        // 如果 ThreadLocal 为空，记录索引
        // 说明除了已经发现的那个，前面还有无效的 Entry
        if (e.get() == null)
            slotToExpunge = i;

    // 向后遍历
    for (int i = nextIndex(staleSlot, len);
            (e = tab[i]) != null;
            i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            // 如果 key 一样，则覆盖值
            e.value = value;
            // 将无效位置元素与当前位置元素交换，
            // 之前无效元素所在的位置上放置的是 set 方法新加入的元素
            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // 如果只有刚开始发现的那一个无效元素
            if (slotToExpunge == staleSlot)
                // 记录无效元素的新位置
                slotToExpunge = i;
            // 执行清理
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            // 返回
            return;
        }

        // 没有哪个 Entry 的 key 与新加入的元素相同，并且
        // 找到了刚开始发现的那个无效元素，则再次记录这个无效元素的索引
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // 执行到这里，说明之前的前序遍历找到了另一个无效的元素
    // 则将刚开始发现的那个位置的 value 置为空，并将新加入的
    // 元素赋给它
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // 这两个值不相等，说明至少有两个无效的元素
    if (slotToExpunge != staleSlot)
        // 清理后来的前序遍历发现的那个无效元素
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

```java
/**
* 计算上一个索引，如果上一个索引为数组中的第一个元素，
* 则会返回数组最后一个位置的索引
*/
private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}
```

```java
/**
* 计算下一个索引，如果上一个索引为数组中最后一个元素，
* 则会返回 0，因此这就组成了一个环
*/
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}
```

```java
/**
* 清理 slot（槽位），只要有 slot 被清理，就返回 true
* @return true if any stale entries have been removed.
*/
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        // 当前位置的元素肯定有效，所以获取下一个索引
        i = nextIndex(i, len);
        Entry e = tab[i];
        // 当弱引用不为空而它关联的强引用为空，说明此时的
        // ThreadLocal 已经没有强引用与之关联，已经被 GC 回收，
        // 此时可以清理它占用的槽位 slot
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            // 清理 slot
            i = expungeStaleEntry(i);
        }
        // 当 n 为 1 时停止，因为当前位置的元素肯定有效，n 为元素个数
    } while ( (n >>>= 1) != 0);
    return removed;
}

/**
* 全量清理
* @param staleSlot index of slot known to have null key
* @return the index of the next null slot after staleSlot
* (all between staleSlot and this slot will have been checked
* for expunging).
*/
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // 清理当前槽位
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // 下面执行全量清理

    Entry e;
    int i;
    // 通过不断寻找下一个索引来建立循环，如果索引上有元素，就
    // 通过判断决定是否进行清理
    for (i = nextIndex(staleSlot, len);
            (e = tab[i]) != null;
            i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 如果 ThreadLocal 为空，执行清理
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            // 如果不为空，则计算索引
            int h = k.threadLocalHashCode & (len - 1);
            // 如果通过 hash 计算出来的索引不是当前位置，
            // 则需要将元素移动到正确的位置
            if (h != i) {
                // 清空当前位置的元素
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                // 如果正确位置上有别的元素，则存在冲突
                while (tab[h] != null)
                    // 采用线性探测法寻找索引
                    h = nextIndex(h, len);
                // 在新位置设置该元素
                tab[h] = e;
            }
        }
    }
    return i;
}
```

```java
private void rehash() {
    // 先对每个元素执行全量清理
    expungeStaleEntries();

    // Use lower threshold for doubling to avoid hysteresis
    // 因为清理过一次，size 的值很可能变小，所以将阈值调小来判断是否
    // 需要扩容
    if (size >= threshold - threshold / 4)
        resize();
}
```

```java
/**
* Expunge all stale entries in the table.
*/
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            // 不用说了，执行全量清理
            expungeStaleEntry(j);
    }
}
```

```java
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    // 新容量为原来的 2 倍
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            // 如果 ThreadLocal 已经为空，将 value 置为空
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                // 计算新的索引
                int h = k.threadLocalHashCode & (newLen - 1);
                // 通过线性探测法设置值
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }
    // 重新设置阈值
    setThreshold(newLen);
    size = count;
    // 引用指向新的容器
    table = newTab;
}
```

<br>[⬆ Back to top](#深入-threadlocal)

### get 方法

```java
public T get() {
    // 当前线程
    Thread t = Thread.currentThread();
    // 获取 threadLocals
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 将当前 ThreadLocal 实例作为参数传入，获取 Entry
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            // 返回 Entry 的 value
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

```java
private Entry getEntry(ThreadLocal<?> key) {
    // 计算索引
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    // 如果找到了，就返回
    if (e != null && e.get() == key)
        return e;
    // 没有找到，使用线性探测法继续查找
    else
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        // 找到了，直接返回
        if (k == key)
            return e;
        // ThreadLocal 为空，则执行全量清理
        if (k == null)
            expungeStaleEntry(i);
        // 线性探测法计算下一个索引
        else
            i = nextIndex(i, len);
        // 获取元素
        e = tab[i];
    }
    return null;
}
```

```java
private T setInitialValue() {
    // 初始化值，需要子类实现
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // 设置值
        map.set(this, value);
    else
        // 创建 ThreadLocalMap
        createMap(t, value);
    return value;
}
```

<br>[⬆ Back to top](#深入-threadlocal)

### remove 方法

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}

private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    // 计算索引
    int i = key.threadLocalHashCode & (len-1);
    // 线性探测法找该元素
    for (Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {
        // 找到该元素
        if (e.get() == key) {
            e.clear();
            // 执行全量清理
            expungeStaleEntry(i);
            return;
        }
    }
}
```

<br>[⬆ Back to top](#深入-threadlocal)

## 为什么要使用弱引用

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

回顾这个静态内部类，会发现它继承了 `WeakReference`，并将作为 Entry 的 key，也就是 `ThreadLocal` 作为引用传递，构造出一个弱引用。  

那么，为什么要使用弱引用呢？首先我们先看一下弱引用的特点。  

```java
// 构造一个弱引用对象，传入需要关联的对象
WeakReference<Object> weakRef = new WeakReference<>(new Object());
// 获取弱引用关联的对象，不为 null
System.out.println(weakRef.get());
// 建议 GC
System.gc();
// 会返回 null
System.out.println(weakRef.get());
// 弱引用对象没有被 GC 回收，因此需要使用某种手段回收它们，可以使用 ReferenceQueue
System.out.println(weakRef);
```

```java
// 构造一个强引用对象
Object strongRef = new Object();
// 构造一个弱引用对象，传入需要关联的对象
WeakReference<Object> weakRef = new WeakReference<>(strongRef);
// 获取弱引用关联的对象，不为 null
System.out.println(weakRef.get());
// 建议 GC
System.gc();
// 由于弱引用关联的对象强可达，所以此处也不为 null
System.out.println(weakRef.get());
// 弱引用对象没有被 GC 回收，因此需要使用某种手段回收它们，可以使用 ReferenceQueue
System.out.println(weakRef);
```

```java
// 构造一个强引用对象
Object strongRef = new Object();
// 构造一个弱引用对象，传入需要关联的对象
WeakReference<Object> weakRef = new WeakReference<>(strongRef);
// 获取弱引用关联的对象，不为 null
System.out.println(weakRef.get());
// 将强引用设置为 null
strongRef = null;
// 建议 GC
System.gc();
// 返回 null
System.out.println(weakRef.get());
// 弱引用对象没有被 GC 回收，因此需要使用某种手段回收它们，可以使用 ReferenceQueue
System.out.println(weakRef);
```

上面的例子已经很明显的说明了弱引用的特性，即**当 JVM 进行垃圾回收时，无论内存是否足够，都会回收被弱引用关联的对象（这里所说的被弱引用关联的对象是指只有弱引用与之关联，如果存在强引用同时与之关联，则 GC 线程不会回收该对象）**。  

如果这里的 Entry 没有使用弱引用，而是通过 key-value 的形式来定义存储结构，则只要线程没有销毁，Entry 节点就不会销毁，这样线程就与节点强绑定了，在 GC 分析的时候节点也会一直处于强可达的状态，没有办法被回收，而程序本身没有办法判断节点是否可以被清理。  

使用了弱引用，当某个 `ThreadLocal` 没有强引用可达时，就会被下一次 GC 回收掉，这时当 Entry 通过 `get()` 方法来获取 `ThreadLocal` 对象时就会为空，这为 `ThreadLocalMap` 中自身的清理方法提供了便利。  

<br>[⬆ Back to top](#深入-threadlocal)

## 内存泄露问题
一般认为 `ThreadLocal` 会引起内存泄露是因为如果当前线程的一个 `ThreadLocal` 对象被回收了，但是对于**当前线程 -> 当前线程的 threadLocals -> Entry 数组 -> 某个 entry 的 value** 这样一条强引用链是可达的，所以 value 不会被回收。这种场景一般出现在有线程复用如线程池的场景中，一个线程会存活很长时间，如果对象很大，长期不被回收会影响系统运行效率与安全性。当然如果线程不复用，那么线程肯定不会存活很长时间，用完就销毁也就不存在 `ThreadLocal` 引发的内存泄露问题了。  

因此，在使用 `ThreadLocal` 的过程中，显式的使用 `remove()` 方法进行清理是必要的，虽然 `get()` 方法和 `set()` 方法有很高的几率会执行清理工作，断开 value 的连接。  

## InheritableThreadLocal
通过 `InheritableThreadLocal` 可以实现父子线程数据共享。  

Thread 类除了 `threadLocals` 外，还有一个 `inheritableThreadLocals`。  

```java
public class Thread implements Runnable {
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
}
```

在线程初始化的时候，会执行 `init()` 方法。  

```java
private void init(ThreadGroup g, Runnable target, String name,
                    long stackSize, AccessControlContext acc,
                    boolean inheritThreadLocals) {
    if (name == null) {
        throw new NullPointerException("name cannot be null");
    }

    this.name = name;

    Thread parent = currentThread();
    SecurityManager security = System.getSecurityManager();
    if (g == null) {
        if (security != null) {
            g = security.getThreadGroup();
        }
        if (g == null) {
            g = parent.getThreadGroup();
        }
    }
    g.checkAccess();

    if (security != null) {
        if (isCCLOverridden(getClass())) {
            security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
        }
    }

    g.addUnstarted();

    this.group = g;
    this.daemon = parent.isDaemon();
    this.priority = parent.getPriority();
    if (security == null || isCCLOverridden(parent.getClass()))
        this.contextClassLoader = parent.getContextClassLoader();
    else
        this.contextClassLoader = parent.contextClassLoader;
    this.inheritedAccessControlContext =
            acc != null ? acc : AccessController.getContext();
    this.target = target;
    setPriority(priority);
    // 当选择继承父线程的 inheritableThreadLocals，并且父线程的 inheritableThreadLocals 不为空时
    if (inheritThreadLocals && parent.inheritableThreadLocals != null)
        // 会将父线程的 inheritableThreadLocals 拷贝到子线程中
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);

    this.stackSize = stackSize;

    tid = nextThreadID();
}
```

```java
/**
* Factory method to create map of inherited thread locals.
* Designed to be called only from Thread constructor.
*
* @param  parentMap the map associated with parent thread
* @return a map containing the parent's inheritable bindings
*/
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    return new ThreadLocalMap(parentMap);
}

/**
* Construct a new map including all Inheritable ThreadLocals
* from given parent map. Called only by createInheritedMap.
*
* @param parentMap the map associated with parent thread.
*/
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    // 设置阈值
    setThreshold(len);
    // 创建容量为父线程 inheritableThreadLocals 的 table 大小的数组
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                // 获取子线程的对应元素的值，需要子类实现，这里默认使用 InheritableThreadLocal 类
                // 的实现，直接返回父线程中对应元素的值
                Object value = key.childValue(e.value);
                // 创建新元素
                Entry c = new Entry(key, value);
                // 计算索引
                int h = key.threadLocalHashCode & (len - 1);
                // 线性探测法放入元素
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```

这里需要注意的一点就是一个线程的 `inheritableThreadLocals` 只在子线程创建的时候会去拷一份父线程的 `inheritableThreadLocals`。如果父线程在子线程创建后再向 `inheritableThreadLocals` 中 set 值，这对子线程来说是不可见的。  

<br>[⬆ Back to top](#深入-threadlocal)

# 参考

> [ThreadLocal源码解读](https://www.cnblogs.com/micrari/p/6790229.html)