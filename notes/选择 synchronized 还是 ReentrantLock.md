# 选择 synchronized 还是 ReentrantLock
`ReentrantLock` 与 `synchronized`相比有更多新的特性，并且性能也很高，我们是不是要使用它替换掉 `synchronized` 呢？答案是没必要。我们可以先看一下使用 `ReentrantLock` 会有哪些问题：  

- `ReentrantLock` 并没有进行类似 `synchronized` 的锁优化。
- 手动释放锁很容易被遗忘，这在测试时可能没有问题，但是在实际工作中就会出现死锁，并且很难找到原因。  
- 当 JVM 用 `synchronized` 管理锁的获取和释放时，它能够在生成 thread dump 时包含锁定的信息，这对调试非常有价值，因为它们能够标识死锁或者其他异常行为的来源。而 `Lock` 只是普通的类，JVM 不知道具体哪个线程拥有 Lock 对象（这个缺陷在 JDK 1.6 中得到了解决，它提供了一个管理和调试接口，锁可以使用这个接口进行注册，并通过其他管理和调试接口，从 thread dump 中得到 `ReentrantLock` 的加锁信息）。  

JDK 1.6 之所以对 `synchronized` 进行锁优化是因为在大部分场景中并不存在线程争用，因此我们可以先将高度争用放在一边。在无争用的情况下，通过锁粗化、适应性自旋、偏向锁等优化，`synchronized` 已经得到了很大的性能提升，甚至几乎与 `ReentrantLock` 相持平。  

基于以上原因，使用 `synchronized` 仍然是首要选择。除非我们对于 `Lock` 的高级特性有明确的要求，或者能够充分证明 `synchronized` 已经成为性能瓶颈。  

# 参考

> [java.util.concurrent ReentrantLock vs synchronized() - which should you use?](https://blogs.oracle.com/dave/javautilconcurrent-reentrantlock-vs-synchronized-which-should-you-use)

> [More flexible, scalable locking in JDK 5.0](https://www.ibm.com/developerworks/library/j-jtp10264/)