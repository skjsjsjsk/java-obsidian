## Spring面试模拟与AOP原理
**Time:** 2026-03-15 19:33
**Summary:** 用户针对 Spring AOP 的实现原理（JDK 动态代理与 CGLIB）寻求反馈，并进行了校招 Java 面试模拟。AI 提供了关于 IOC 与 DI 的详细解答，预测了如 Bean 生命周期等高频考点，并总结了 AOP 的核心实现机制。

## MySQL优化与Redis面试核心知识点
**Time:** 2026-03-15 21:10
**Summary:** 用户咨询了MySQL优化策略及RR隔离级别下MVCC与Next-Key Lock解决幻读的原理，确认了该说法的正确性。针对Redis面试笔记，AI指出了缓存穿透与击穿的术语错误，并建议补充过期策略、多线程及高可用架构等内容。此外，对话还解析了大型网站UV统计的HyperLogLog方案以及Redis主从、哨兵、集群的核心运作机制。

## Java类加载过程与双亲委派机制
**Time:** 2026-03-22 19:24
**Summary:** 用户询问如何在面试中回答JVM类加载机制的问题。AI详细讲解了类加载的三个阶段（加载、连接、初始化），优化了用户的回答话术，并深入解释了双亲委派模型的原理及其用于保证核心类安全性和避免重复加载的原因。

## MySQL InnoDB核心机制解析
**Time:** 2026-03-22 20:52
**Summary:** 用户探讨了MySQL中varchar长度对性能的影响、深分页优化方案及文件排序原理。对话深入解析了临键锁解决幻读的必要性及其锁定区间，并详细说明了MVCC Read View的可见性判断规则。最终验证并完善了用户关于锁机制触发条件和事务ID比对算法的笔记总结。

## Java校招面试题逐题点评
**Time:** 2026-04-22 16:33
**Summary:** 对话围绕 6 个 Java 校招高频基础题展开，包括多态、包装类、ArrayList/LinkedList、ArrayList 扩容与集合转换、hashCode/equals。用户逐题作答，AI 以资深面试官视角指出回答中的正确点与错误点，补充了更标准的面试回答、常见追问以及每题评分；当前已点评到第 5 题，并准备继续第 6 题 String、StringBuilder、StringBuffer 的区别。

## Java单例模式实现
**Time:** 2026-05-07 22:22
**Summary:** 用户询问 Java 中单例模式的实现方式，重点涉及饿汉式、懒汉式、双重检查锁、静态内部类和枚举单例。对话进一步解释了 `private static final Singleton INSTANCE = new Singleton();` 是创建类内部唯一实例，普通懒汉式因多线程同时判断并创建对象而线程不安全，以及 `private Singleton() {}` 是私有构造方法，用于禁止外部 new 对象。

## Java单例模式解析
**Time:** 2026-05-07 22:40
**Summary:** 用户围绕 Java 单例模式实现进行学习，重点询问了饿汉式、懒汉式、双重检查锁 DCL、静态内部类和枚举单例的代码含义与线程安全问题。对话详细解释了 `private static final Singleton INSTANCE = new Singleton()`、私有构造方法、懒汉式线程不安全原因，以及 DCL 中 `volatile` 防止指令重排和半初始化对象的作用。
