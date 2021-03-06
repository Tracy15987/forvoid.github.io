---
title: Java无锁实现线程安全的方法（CAS）
date: 2017-6-23 20:50:49
tags:
- Java并发

---
## 理解CAS
基于锁的同步方式是`对数据进行加锁阻塞`的方式进行的，无论是使用信号量、重入锁、内部锁，都会造成线程之间的相互等待。

为了避免阻塞，就提出了`非阻塞同步`的方式，（比如ThreadLocal，每个线程都拥有各自独立的变量副本）在并行计算时，无需相互等待。

这里写一下我在书上学到的基于 比较并交换（Compare and Swap）算法的无锁并发控制.
