- [深入 ThreadLocal](#深入-threadlocal)

# 深入 ThreadLocal

`ThreadLocal` 可以理解成线程局部变量，即每个线程都可以有一个或多个 `ThreadLocal`作为私有变量使用，每个线程向 `ThreadLocal` 中读写都是线程隔离的，`ThreadLocal` 提供了一种将共享数据通过每个线程独立的副本来实现线程封闭。  
