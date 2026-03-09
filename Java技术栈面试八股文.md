# Java 技术栈面试八股文大全

> 涵盖 Java 基础、JVM、Spring Boot、微服务架构、AOP、事务六大核心主题，适用于中高级 Java 开发工程师面试准备。

---

## 一、Java 基础与核心知识

### 1.1 Java 语言基础

**Q：Java 中 == 和 equals() 的区别是什么？**

`==` 对于基本数据类型比较的是值，对于引用类型比较的是内存地址（即是否为同一个对象）。`equals()` 是 Object 类的方法，默认实现也是比较内存地址，但很多类（如 String、Integer）重写了该方法，使其比较的是对象的内容。需要注意的是，重写 `equals()` 时必须同时重写 `hashCode()`，否则在使用 HashMap、HashSet 等基于哈希的集合时会出现逻辑错误，因为这些集合先通过 `hashCode()` 定位桶，再通过 `equals()` 判断是否相等。

**Q：String、StringBuilder 和 StringBuffer 的区别？**

String 是不可变类（immutable），底层使用 `final char[]`（JDK 9 之后改为 `final byte[]`），每次修改都会创建新对象。StringBuilder 和 StringBuffer 都是可变的，区别在于 StringBuffer 的方法使用了 `synchronized` 关键字，是线程安全的，但性能较低；StringBuilder 非线程安全，但性能更高。在单线程环境下优先使用 StringBuilder，多线程共享场景使用 StringBuffer。实际开发中，由于字符串拼接场景很少涉及多线程共享，StringBuilder 使用更为广泛。

**Q：Java 中的基本数据类型有哪些？自动装箱和拆箱是什么？**

Java 有 8 种基本数据类型：byte（1字节）、short（2字节）、int（4字节）、long（8字节）、float（4字节）、double（8字节）、char（2字节）、boolean。自动装箱（Autoboxing）是编译器自动将基本类型转换为对应的包装类型（如 int → Integer），自动拆箱（Unboxing）则相反。需要注意 Integer 缓存池问题：`Integer.valueOf()` 对 -128 到 127 范围内的值会使用缓存，因此 `Integer a = 127; Integer b = 127; a == b` 为 true，但 128 就不是了。

**Q：final、finally、finalize 的区别？**

`final` 是修饰符：修饰类表示不可继承，修饰方法表示不可重写，修饰变量表示不可重新赋值（但引用类型的对象内容仍可修改）。`finally` 是异常处理的一部分，无论是否发生异常都会执行（除非 JVM 退出或线程被中断）。`finalize()` 是 Object 类的方法，在对象被 GC 回收前调用，但不推荐使用，因为执行时机不确定且可能导致性能问题，Java 9 已将其标记为 @Deprecated。

**Q：接口和抽象类的区别？**

抽象类使用 `abstract class` 声明，可以包含抽象方法和具体方法、成员变量、构造方法，支持单继承。接口使用 `interface` 声明，Java 8 之前只能有抽象方法和常量，Java 8 引入了 `default` 方法和 `static` 方法，Java 9 引入了 `private` 方法。一个类可以实现多个接口但只能继承一个抽象类。设计层面上，抽象类体现的是 "is-a" 关系，接口体现的是 "has-a" 或 "can-do" 的能力约定。

### 1.2 集合框架

**Q：HashMap 的底层实现原理？**

JDK 1.8 中 HashMap 底层采用 **数组 + 链表 + 红黑树** 的结构。数组是 `Node<K,V>[]`，默认初始容量为 16，负载因子为 0.75。put 操作时，先对 key 的 hashCode 进行扰动处理（高16位异或低16位），然后通过 `(n-1) & hash` 计算数组下标。如果发生哈希冲突，以链表形式存储；当链表长度超过 8 且数组长度大于等于 64 时，链表转化为红黑树（时间复杂度从 O(n) 降为 O(log n)）；当红黑树节点数小于 6 时退化为链表。扩容时容量翻倍，所有元素重新计算位置（rehash），JDK 1.8 优化为只需判断新增的一个 bit 是 0 还是 1 来决定元素在原位置还是原位置+旧容量的位置。

**Q：ConcurrentHashMap 的实现原理？JDK 1.7 和 1.8 有什么区别？**

JDK 1.7 采用 **Segment 分段锁** 机制，每个 Segment 继承 ReentrantLock，默认 16 个 Segment，最多支持 16 个线程并发写。JDK 1.8 摒弃了 Segment，采用 **CAS + synchronized** 的方式，锁粒度细化到每个桶（Node），并发度大幅提升。put 操作时，如果桶为空则通过 CAS 写入；如果不为空则对头节点加 synchronized 锁后进行链表/红黑树操作。size() 方法使用 baseCount + CounterCell 数组的方式（类似 LongAdder），避免了全局锁。

**Q：ArrayList 和 LinkedList 的区别？**

ArrayList 底层是动态数组（`Object[]`），支持随机访问（O(1)），尾部插入均摊 O(1)，中间插入/删除需要移动元素（O(n)），扩容时容量增长为原来的 1.5 倍。LinkedList 底层是双向链表，不支持随机访问（O(n)），头尾插入/删除 O(1)，但每个节点需要额外存储前后指针，内存开销更大。实际场景中，由于 CPU 缓存局部性原理，ArrayList 在大多数场景下性能优于 LinkedList。

**Q：HashSet 的实现原理？**

HashSet 底层基于 HashMap 实现，元素作为 HashMap 的 key 存储，value 统一使用一个静态的 `PRESENT` 对象（`new Object()`）。因此 HashSet 的去重依赖于元素的 `hashCode()` 和 `equals()` 方法。

### 1.3 多线程与并发

**Q：线程的创建方式有哪些？**

主要有四种方式：继承 Thread 类重写 `run()` 方法；实现 Runnable 接口；实现 Callable 接口（可以有返回值，配合 FutureTask 使用）；通过线程池（ExecutorService）提交任务。实际开发中推荐使用线程池，避免频繁创建和销毁线程的开销。

**Q：线程池的核心参数有哪些？工作流程是怎样的？**

ThreadPoolExecutor 的 7 个核心参数：corePoolSize（核心线程数）、maximumPoolSize（最大线程数）、keepAliveTime（非核心线程空闲存活时间）、unit（时间单位）、workQueue（任务队列）、threadFactory（线程工厂）、handler（拒绝策略）。

工作流程：提交任务时，如果当前线程数 < corePoolSize，创建核心线程执行；如果 >= corePoolSize，任务放入 workQueue；如果队列已满且线程数 < maximumPoolSize，创建非核心线程执行；如果队列已满且线程数 >= maximumPoolSize，执行拒绝策略。

四种拒绝策略：AbortPolicy（抛异常，默认）、CallerRunsPolicy（调用者线程执行）、DiscardPolicy（静默丢弃）、DiscardOldestPolicy（丢弃队列最老的任务）。

**Q：synchronized 和 ReentrantLock 的区别？**

synchronized 是 JVM 层面的关键字，基于 Monitor 机制实现，JDK 1.6 后引入了偏向锁、轻量级锁、重量级锁的锁升级机制。ReentrantLock 是 JDK 层面的 API，基于 AQS（AbstractQueuedSynchronizer）实现。区别在于：ReentrantLock 支持公平锁/非公平锁、可中断锁、超时获取锁、多条件变量（Condition）；synchronized 使用更简单，不需要手动释放锁，JVM 会自动释放。性能方面，JDK 1.6 优化后两者差距不大，优先使用 synchronized，需要高级功能时使用 ReentrantLock。

**Q：volatile 关键字的作用和原理？**

volatile 有两个作用：保证可见性和禁止指令重排序。可见性方面，volatile 变量的写操作会立即刷新到主内存，读操作会从主内存重新加载，底层通过 **内存屏障（Memory Barrier）** 实现。禁止重排序方面，编译器和处理器不会对 volatile 变量的读写操作与其前后的操作进行重排序。典型应用场景是 DCL（Double-Check Locking）单例模式中防止指令重排导致获取到未初始化完成的对象。注意 volatile 不保证原子性，`i++` 这样的复合操作仍然不是线程安全的。

**Q：AQS（AbstractQueuedSynchronizer）的原理？**

AQS 是 Java 并发包的基础框架，核心思想是：维护一个 volatile int state 表示同步状态，以及一个 FIFO 的 CLH 双向队列管理等待线程。获取锁时，通过 CAS 修改 state，成功则获取锁，失败则将线程封装为 Node 加入队列尾部并阻塞（park）。释放锁时，修改 state 并唤醒（unpark）队列中的后继节点。ReentrantLock、CountDownLatch、Semaphore、ReentrantReadWriteLock 等都是基于 AQS 实现的。

**Q：ThreadLocal 的原理？会有什么问题？**

每个 Thread 内部维护一个 `ThreadLocalMap`，key 是 ThreadLocal 对象的弱引用，value 是存储的值。调用 `threadLocal.set(value)` 时，实际是往当前线程的 ThreadLocalMap 中存入数据。内存泄漏问题：由于 key 是弱引用，GC 后 key 变为 null，但 value 是强引用不会被回收，如果线程长期存活（如线程池中的线程），这些 value 就无法被回收。解决方案是使用完后调用 `threadLocal.remove()`。

**Q：CAS 是什么？有什么问题？**

CAS（Compare And Swap）是一种无锁的原子操作，包含三个操作数：内存地址 V、期望值 A、新值 B。只有当 V 的值等于 A 时，才将 V 更新为 B。底层依赖 CPU 的 `cmpxchg` 指令。问题包括：ABA 问题（值从 A 变为 B 再变回 A，CAS 检测不到变化，可用 AtomicStampedReference 解决）；自旋开销（长时间 CAS 失败会持续占用 CPU）；只能保证单个变量的原子性。

### 1.4 JDK 新特性

**Q：Java 8 的核心新特性有哪些？**

Lambda 表达式（函数式编程支持）、Stream API（集合的声明式处理，支持 filter/map/reduce/collect 等操作，支持并行流）、Optional 类（优雅处理 null）、接口的 default 方法和 static 方法、新的日期时间 API（LocalDate/LocalTime/LocalDateTime，线程安全）、方法引用（`Class::method`）、CompletableFuture（异步编程增强）。

**Q：Stream 的中间操作和终端操作有什么区别？**

中间操作（如 filter、map、sorted、distinct、flatMap）是惰性求值的，只有在终端操作触发时才会执行。终端操作（如 forEach、collect、reduce、count、findFirst）会触发实际的计算。Stream 只能被消费一次，消费后不能再使用。并行流（parallelStream）底层使用 ForkJoinPool，适合 CPU 密集型且数据量大的场景，IO 密集型场景不建议使用。

---

## 二、JVM（Java 虚拟机）

### 2.1 内存模型与内存区域

**Q：JVM 的内存区域是如何划分的？**

JVM 运行时数据区分为线程私有和线程共享两大类。线程私有区域包括：程序计数器（记录当前线程执行的字节码行号，唯一不会 OOM 的区域）、虚拟机栈（每个方法调用创建一个栈帧，包含局部变量表、操作数栈、动态链接、方法返回地址，栈深度超限抛 StackOverflowError）、本地方法栈（为 Native 方法服务）。线程共享区域包括：堆（对象实例和数组的分配区域，GC 的主要区域，分为新生代和老年代）、方法区（存储类信息、常量、静态变量、JIT 编译后的代码，JDK 8 之前用永久代实现，JDK 8 改为元空间 Metaspace，使用本地内存）。此外还有直接内存（NIO 的 DirectByteBuffer 使用，不受 JVM 堆大小限制）。

**Q：Java 内存模型（JMM）是什么？**

JMM（Java Memory Model）定义了多线程环境下变量的访问规则。核心概念是：每个线程有自己的工作内存（CPU 缓存的抽象），所有变量存储在主内存中。线程对变量的操作必须在工作内存中进行，不能直接操作主内存。JMM 通过 happens-before 原则来保证内存可见性和有序性，主要规则包括：程序顺序规则、volatile 变量规则、锁规则、线程启动/终止规则、传递性规则等。

**Q：对象的创建过程是怎样的？**

类加载检查 → 分配内存（指针碰撞或空闲列表，通过 CAS 或 TLAB 解决并发问题）→ 内存空间初始化为零值 → 设置对象头（Mark Word 包含哈希码、GC 分代年龄、锁状态标志、线程持有的锁等；类型指针指向类元数据）→ 执行 `<init>` 方法（构造函数）。

**Q：对象在内存中的布局是怎样的？**

对象在堆中的存储布局分为三部分：对象头（Header）、实例数据（Instance Data）、对齐填充（Padding）。对象头包含 Mark Word（32/64 位，存储哈希码、GC 年龄、锁标志等）和类型指针（指向方法区中的 Class 元数据）。如果是数组对象，还有一个记录数组长度的字段。对齐填充确保对象大小是 8 字节的整数倍。

### 2.2 垃圾回收（GC）

**Q：如何判断对象是否可以被回收？**

两种方法：引用计数法（每个对象维护一个引用计数器，为 0 时回收，但无法解决循环引用问题，Java 未采用）和 **可达性分析法**（从 GC Roots 出发，沿引用链遍历，不可达的对象即为可回收对象）。GC Roots 包括：虚拟机栈中引用的对象、方法区中静态属性引用的对象、方法区中常量引用的对象、本地方法栈中 JNI 引用的对象、被 synchronized 持有的对象等。

**Q：Java 中的四种引用类型是什么？**

强引用（Strong Reference）：最常见的引用，只要强引用存在，对象不会被回收。软引用（Soft Reference）：内存不足时才会被回收，适合做缓存。弱引用（Weak Reference）：下一次 GC 时就会被回收，ThreadLocalMap 的 key 就是弱引用。虚引用（Phantom Reference）：不影响对象生命周期，仅用于在对象被回收时收到通知，必须配合 ReferenceQueue 使用，NIO 中 DirectByteBuffer 的回收就用到了虚引用。

**Q：常见的垃圾回收算法有哪些？**

标记-清除（Mark-Sweep）：先标记可回收对象，再统一清除，缺点是产生内存碎片。标记-复制（Copying）：将内存分为两块，每次只使用一块，GC 时将存活对象复制到另一块，缺点是内存利用率只有 50%（新生代的 Eden:S0:S1 = 8:1:1 优化了这个问题）。标记-整理（Mark-Compact）：标记后将存活对象向一端移动，然后清理边界外的内存，没有碎片但移动对象开销大。分代收集：根据对象存活周期将堆分为新生代（使用复制算法）和老年代（使用标记-清除或标记-整理）。

**Q：常见的垃圾收集器有哪些？各自的特点？**

Serial：单线程收集器，新生代用复制算法，老年代用标记-整理，适合客户端模式。ParNew：Serial 的多线程版本，常与 CMS 配合使用。Parallel Scavenge：关注吞吐量的多线程收集器，JDK 8 默认。

CMS（Concurrent Mark Sweep）：以最短停顿时间为目标的老年代收集器，采用标记-清除算法。四个阶段：初始标记（STW，标记 GC Roots 直接关联的对象）→ 并发标记（与用户线程并发，遍历对象图）→ 重新标记（STW，修正并发标记期间变动的对象，使用增量更新）→ 并发清除。缺点：产生内存碎片、对 CPU 资源敏感、无法处理浮动垃圾。

G1（Garbage First）：JDK 9 默认收集器，将堆划分为多个大小相等的 Region（1-32MB），每个 Region 可以是 Eden、Survivor、Old 或 Humongous。通过维护优先级列表，优先回收价值最大的 Region。支持设置期望停顿时间（-XX:MaxGCPauseMillis）。四个阶段：初始标记 → 并发标记 → 最终标记 → 筛选回收。使用 SATB（Snapshot At The Beginning）解决并发标记的漏标问题。

ZGC：JDK 11 引入的低延迟收集器，停顿时间不超过 10ms（JDK 16 后不超过 1ms），支持 TB 级堆内存。核心技术是染色指针（Colored Pointers）和读屏障（Load Barrier），几乎所有阶段都是并发的。

**Q：Minor GC、Major GC、Full GC 的区别？**

Minor GC（Young GC）：发生在新生代，当 Eden 区满时触发，速度快、频率高。Major GC：发生在老年代，通常伴随 Minor GC。Full GC：对整个堆（新生代 + 老年代）和方法区进行回收，速度慢，应尽量避免。触发 Full GC 的条件包括：老年代空间不足、方法区空间不足、调用 `System.gc()`（建议，不保证）、CMS 并发收集失败（Concurrent Mode Failure）、晋升失败等。

**Q：对象什么时候会进入老年代？**

四种情况：对象年龄达到阈值（默认 15，每经历一次 Minor GC 且存活，年龄 +1，通过 -XX:MaxTenuringThreshold 设置）；大对象直接进入老年代（-XX:PretenureSizeThreshold）；动态年龄判断（Survivor 区中相同年龄的对象大小之和超过 Survivor 空间的一半，则大于等于该年龄的对象直接进入老年代）；Minor GC 后 Survivor 区放不下的对象。

### 2.3 类加载机制

**Q：类加载的过程是怎样的？**

类的生命周期包括：加载（Loading）→ 验证（Verification）→ 准备（Preparation）→ 解析（Resolution）→ 初始化（Initialization）→ 使用（Using）→ 卸载（Unloading）。其中验证、准备、解析统称为连接（Linking）。

加载：通过类的全限定名获取二进制字节流，将其转化为方法区的运行时数据结构，在堆中生成 Class 对象。验证：确保字节流符合 JVM 规范（文件格式验证、元数据验证、字节码验证、符号引用验证）。准备：为类的静态变量分配内存并设置零值（final static 常量直接赋值）。解析：将常量池中的符号引用替换为直接引用。初始化：执行类构造器 `<clinit>()` 方法（编译器收集所有静态变量赋值和 static 块合并而成），JVM 保证 `<clinit>()` 的线程安全。

**Q：什么是双亲委派模型？为什么要用它？**

类加载器收到加载请求时，先委派给父加载器处理，父加载器无法完成时才自己加载。加载器层次：Bootstrap ClassLoader（加载 rt.jar 等核心类库）→ Extension ClassLoader（加载 ext 目录）→ Application ClassLoader（加载 classpath）→ 自定义 ClassLoader。

双亲委派的好处：避免类的重复加载；保证核心 API 不被篡改（如自定义 java.lang.String 不会被加载）。打破双亲委派的场景：SPI 机制（如 JDBC，使用线程上下文类加载器）、OSGi 模块化、Tomcat 的 WebAppClassLoader（每个 Web 应用有独立的类加载器，优先加载自己的类）、热部署。

### 2.4 JVM 调优

**Q：常用的 JVM 参数有哪些？**

堆内存：`-Xms`（初始堆大小）、`-Xmx`（最大堆大小，建议与 -Xms 相同避免动态扩容）、`-Xmn`（新生代大小）、`-XX:NewRatio`（老年代/新生代比例）、`-XX:SurvivorRatio`（Eden/Survivor 比例）。

元空间：`-XX:MetaspaceSize`（初始大小）、`-XX:MaxMetaspaceSize`（最大大小）。

GC 相关：`-XX:+UseG1GC`（使用 G1）、`-XX:MaxGCPauseMillis`（G1 期望停顿时间）、`-XX:+PrintGCDetails`（打印 GC 日志）、`-Xlog:gc*`（JDK 9+ 统一日志）。

OOM 排查：`-XX:+HeapDumpOnOutOfMemoryError`（OOM 时自动 dump）、`-XX:HeapDumpPath`（dump 文件路径）。

**Q：如何排查 OOM 问题？**

首先确认 OOM 类型：Java heap space（堆内存不足）、Metaspace（元空间不足）、GC overhead limit exceeded（GC 时间过长）、Direct buffer memory（直接内存不足）。排查步骤：通过 `-XX:+HeapDumpOnOutOfMemoryError` 获取 heap dump 文件 → 使用 MAT（Memory Analyzer Tool）或 VisualVM 分析 → 找到占用内存最大的对象 → 结合代码定位内存泄漏点。常见原因包括：集合类持有大量对象未释放、静态集合不断增长、未关闭的资源（连接、流）、ThreadLocal 未 remove 等。

---

## 三、Spring Boot 框架

### 3.1 核心原理

**Q：Spring Boot 的自动配置原理是什么？**

Spring Boot 自动配置的核心是 `@SpringBootApplication` 注解，它是一个组合注解，包含 `@SpringBootConfiguration`（标识配置类）、`@EnableAutoConfiguration`（开启自动配置）、`@ComponentScan`（组件扫描）。

`@EnableAutoConfiguration` 通过 `@Import(AutoConfigurationImportSelector.class)` 导入自动配置类。`AutoConfigurationImportSelector` 会读取 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（Spring Boot 3.x）或 `META-INF/spring.factories`（Spring Boot 2.x）文件中配置的所有自动配置类的全限定名。这些自动配置类通过 `@Conditional` 系列注解（如 `@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@ConditionalOnProperty`）进行条件判断，只有满足条件时才会生效。用户自定义的 Bean 优先级高于自动配置的 Bean（`@ConditionalOnMissingBean` 保证）。

**Q：Spring Boot 的启动流程是怎样的？**

`SpringApplication.run()` 方法的核心流程：创建 SpringApplication 对象（推断应用类型 Servlet/Reactive/None、加载 ApplicationContextInitializer 和 ApplicationListener）→ 获取并启动 SpringApplicationRunListeners → 准备环境（Environment）→ 创建 ApplicationContext → 准备上下文（执行 Initializer、加载 Bean 定义）→ 刷新上下文（`refresh()`，这是 Spring 容器的核心，完成 Bean 的创建和依赖注入）→ 执行 Runner（CommandLineRunner / ApplicationRunner）→ 发布 ApplicationReadyEvent。

**Q：Spring Boot 中的 Starter 是什么？如何自定义 Starter？**

Starter 是一组依赖的集合，引入一个 Starter 就能获得该功能所需的所有依赖和自动配置。如 `spring-boot-starter-web` 包含了 Spring MVC、Tomcat、Jackson 等。

自定义 Starter 的步骤：创建 `xxx-spring-boot-autoconfigure` 模块（包含自动配置类和 `@ConfigurationProperties` 配置属性类）→ 创建 `xxx-spring-boot-starter` 模块（仅包含 pom 依赖，引入 autoconfigure 模块和其他必要依赖）→ 在 autoconfigure 模块的 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 中注册自动配置类 → 使用 `@Conditional` 注解控制配置生效条件。

### 3.2 Spring 核心

**Q：Spring IoC 的原理是什么？**

IoC（Inversion of Control，控制反转）是一种设计思想，将对象的创建和依赖关系的管理交给 Spring 容器，而不是由程序员手动 new 对象。DI（Dependency Injection，依赖注入）是 IoC 的具体实现方式，包括构造器注入、Setter 注入、字段注入（@Autowired）。

Spring 容器的核心是 BeanFactory（懒加载）和 ApplicationContext（预加载，功能更丰富，支持国际化、事件发布等）。Bean 的生命周期：实例化（通过反射创建对象）→ 属性填充（依赖注入）→ Aware 接口回调（BeanNameAware、BeanFactoryAware、ApplicationContextAware）→ BeanPostProcessor 的 `postProcessBeforeInitialization` → InitializingBean 的 `afterPropertiesSet` / 自定义 init-method → BeanPostProcessor 的 `postProcessAfterInitialization`（AOP 代理在此创建）→ 使用 → DisposableBean 的 `destroy` / 自定义 destroy-method。

**Q：Spring 中 Bean 的作用域有哪些？**

singleton（默认，容器中只有一个实例）、prototype（每次获取都创建新实例）、request（每个 HTTP 请求一个实例）、session（每个 HTTP Session 一个实例）、application（每个 ServletContext 一个实例）、websocket（每个 WebSocket 会话一个实例）。

**Q：Spring 如何解决循环依赖？**

Spring 通过 **三级缓存** 解决 singleton 作用域的 setter/字段注入的循环依赖。一级缓存 `singletonObjects`：存放完全初始化好的 Bean。二级缓存 `earlySingletonObjects`：存放提前暴露的 Bean（已实例化但未完成属性填充）。三级缓存 `singletonFactories`：存放 Bean 的 ObjectFactory（用于创建早期引用，支持 AOP 代理）。

流程（A 依赖 B，B 依赖 A）：创建 A → A 实例化后将 ObjectFactory 放入三级缓存 → A 填充属性发现依赖 B → 创建 B → B 实例化后将 ObjectFactory 放入三级缓存 → B 填充属性发现依赖 A → 从三级缓存获取 A 的 ObjectFactory 创建早期引用放入二级缓存 → B 完成初始化放入一级缓存 → A 获取到 B 完成初始化放入一级缓存。

注意：构造器注入的循环依赖无法解决（可用 @Lazy 延迟加载）；prototype 作用域的循环依赖无法解决；Spring Boot 2.6+ 默认禁止循环依赖。

**Q：@Autowired 和 @Resource 的区别？**

`@Autowired` 是 Spring 提供的注解，默认按类型（byType）注入，如果存在多个同类型 Bean，再按名称匹配，可配合 `@Qualifier` 指定 Bean 名称。`@Resource` 是 JSR-250 标准注解，默认按名称（byName）注入，找不到再按类型匹配，可通过 `name` 和 `type` 属性指定。推荐使用构造器注入，因为它能保证依赖不可变且不为 null。

### 3.3 Spring MVC

**Q：Spring MVC 的请求处理流程是怎样的？**

客户端发送请求 → DispatcherServlet 接收（前端控制器）→ 调用 HandlerMapping 查找对应的 Handler（Controller 方法）→ 通过 HandlerAdapter 执行 Handler → Handler 返回 ModelAndView → ViewResolver 解析视图 → 渲染视图并返回响应。在前后端分离架构中，Controller 方法通常使用 `@ResponseBody`（或 `@RestController`），通过 HttpMessageConverter（如 MappingJackson2HttpMessageConverter）将返回值直接序列化为 JSON，不经过视图解析。

**Q：Spring MVC 中的拦截器和过滤器有什么区别？**

过滤器（Filter）是 Servlet 规范的一部分，在 DispatcherServlet 之前执行，基于函数回调，可以拦截所有请求（包括静态资源）。拦截器（Interceptor）是 Spring MVC 的组件，在 Handler 执行前后执行，基于 AOP 思想，只拦截 Controller 的请求，可以访问 Spring 容器中的 Bean。执行顺序：Filter → DispatcherServlet → Interceptor.preHandle → Handler → Interceptor.postHandle → 视图渲染 → Interceptor.afterCompletion → Filter。

---

## 四、微服务架构

### 4.1 微服务基础

**Q：什么是微服务？微服务和单体架构的区别？**

微服务是一种架构风格，将一个大型应用拆分为一组小型、自治的服务，每个服务围绕特定业务能力构建，独立部署、独立扩展，通过轻量级通信机制（通常是 HTTP/REST 或 RPC）进行交互。

与单体架构相比，微服务的优势在于：技术栈灵活（不同服务可以使用不同语言和框架）、独立部署（修改一个服务不需要重新部署整个应用）、独立扩展（可以针对高负载服务单独扩容）、故障隔离（一个服务故障不会导致整个系统崩溃）。劣势在于：分布式系统的复杂性（网络延迟、数据一致性、分布式事务）、运维成本高（需要容器化、服务编排、监控告警等基础设施）、服务间通信开销、测试复杂度增加。

**Q：微服务架构中有哪些核心组件？**

服务注册与发现（Nacos、Eureka、Consul、Zookeeper）、配置中心（Nacos、Apollo、Spring Cloud Config）、API 网关（Spring Cloud Gateway、Zuul）、负载均衡（Ribbon、Spring Cloud LoadBalancer）、服务调用（OpenFeign、Dubbo）、熔断降级（Sentinel、Hystrix、Resilience4j）、分布式链路追踪（SkyWalking、Zipkin、Jaeger）、消息队列（Kafka、RocketMQ、RabbitMQ）。

### 4.2 服务注册与发现

**Q：Nacos 的服务注册与发现原理？**

服务提供者启动时向 Nacos Server 注册自己的地址信息（IP、端口、服务名等），并通过心跳机制保持注册状态（临时实例默认 5 秒发送心跳，15 秒未收到标记为不健康，30 秒未收到剔除）。服务消费者从 Nacos Server 获取服务提供者列表并缓存到本地，通过订阅机制（推拉结合）感知服务变化。Nacos 支持 AP 和 CP 两种模式：临时实例使用 Distro 协议（AP，最终一致性），持久实例使用 Raft 协议（CP，强一致性）。

**Q：Nacos 和 Eureka 的区别？**

Nacos 同时支持 AP 和 CP 模式，Eureka 只支持 AP。Nacos 支持服务端主动推送变更，Eureka 只能客户端定时拉取。Nacos 集成了配置中心功能，Eureka 没有。Nacos 支持临时实例和持久实例，Eureka 只有临时实例。Nacos 有管理控制台，功能更丰富。Eureka 2.x 已停止维护。

### 4.3 服务调用与负载均衡

**Q：OpenFeign 的工作原理？**

OpenFeign 是一个声明式的 HTTP 客户端，通过接口 + 注解的方式定义远程调用。原理：启动时扫描 `@FeignClient` 注解的接口 → 为每个接口创建 JDK 动态代理 → 调用方法时，代理对象根据注解信息（URL、请求方法、参数等）构建 HTTP 请求 → 通过负载均衡器选择服务实例 → 发送 HTTP 请求并解析响应。OpenFeign 集成了 Ribbon/Spring Cloud LoadBalancer 实现客户端负载均衡，支持请求/响应压缩、日志级别配置、超时配置等。

**Q：常见的负载均衡策略有哪些？**

客户端负载均衡（如 Ribbon）的策略：轮询（RoundRobin，默认）、随机（Random）、加权轮询/随机（根据服务器性能分配权重）、最少连接数（选择当前连接数最少的服务器）、一致性哈希（相同请求路由到相同服务器）、区域感知（优先选择同区域的服务器）。服务端负载均衡（如 Nginx）的策略类似，但在服务端进行分发。

### 4.4 熔断降级与限流

**Q：什么是服务熔断？什么是服务降级？**

服务熔断是一种保护机制，当某个服务的错误率或响应时间超过阈值时，熔断器打开，后续请求直接返回失败（快速失败），不再调用下游服务，防止故障扩散（雪崩效应）。熔断器有三种状态：关闭（正常调用）→ 打开（直接失败）→ 半开（允许少量请求探测，成功则关闭，失败则继续打开）。

服务降级是在系统压力过大或部分服务不可用时，暂时关闭一些非核心功能，保证核心功能可用。降级可以是自动的（熔断触发后的 fallback）也可以是手动的（运维开关）。

**Q：Sentinel 的核心原理？**

Sentinel 以资源（Resource）为核心，通过定义规则（Rule）来保护资源。核心架构：所有请求通过 Sentinel 的责任链（ProcessorSlotChain）处理，每个 Slot 负责不同功能：NodeSelectorSlot（构建调用树）→ ClusterBuilderSlot（构建集群统计节点）→ StatisticSlot（实时统计，基于滑动窗口）→ FlowSlot（流量控制）→ DegradeSlot（熔断降级）→ AuthoritySlot（授权控制）→ SystemSlot（系统保护）。

流量控制支持 QPS 和并发线程数两种模式，支持直接拒绝、Warm Up（预热）、匀速排队三种效果。熔断降级支持慢调用比例、异常比例、异常数三种策略。

### 4.5 API 网关

**Q：Spring Cloud Gateway 的工作原理？**

Spring Cloud Gateway 基于 Spring WebFlux（Reactor 模型，非阻塞），核心概念有三个：Route（路由，包含 ID、目标 URI、Predicate 集合、Filter 集合）、Predicate（断言，匹配请求条件如路径、Header、参数等）、Filter（过滤器，在请求前后进行处理）。

请求处理流程：客户端请求 → Gateway Handler Mapping 根据 Predicate 匹配路由 → Gateway Web Handler 通过 Filter Chain 处理请求 → 代理到下游服务 → 响应经过 Filter Chain 返回客户端。Filter 分为 GatewayFilter（针对单个路由）和 GlobalFilter（针对所有路由），常用于鉴权、限流、日志、请求/响应修改等。

### 4.6 分布式相关

**Q：CAP 理论和 BASE 理论是什么？**

CAP 理论：分布式系统不可能同时满足一致性（Consistency，所有节点数据一致）、可用性（Availability，每个请求都能得到响应）、分区容错性（Partition tolerance，网络分区时系统仍能运行）三个特性，最多只能满足其中两个。由于网络分区不可避免，实际上是在 CP 和 AP 之间选择。

BASE 理论是对 CAP 中 AP 的延伸：Basically Available（基本可用，允许响应时间增加或功能降级）、Soft state（软状态，允许中间状态存在）、Eventually consistent（最终一致性，经过一段时间后数据达到一致）。BASE 是大规模互联网系统的实践指导。

**Q：分布式 ID 的生成方案有哪些？**

UUID（简单但无序、太长，不适合做数据库主键）、数据库自增（简单但有单点瓶颈）、数据库号段模式（如美团 Leaf-segment，批量获取 ID 段缓存在内存中）、Redis INCR（性能好但依赖 Redis）、雪花算法 Snowflake（64 位：1 位符号位 + 41 位时间戳 + 10 位机器 ID + 12 位序列号，有序且高性能，但依赖时钟，存在时钟回拨问题，美团 Leaf-snowflake 通过 Zookeeper 解决了 workerId 分配问题）。

**Q：分布式锁的实现方案有哪些？**

基于 Redis：使用 `SET key value NX EX` 命令获取锁，通过 Lua 脚本保证释放锁的原子性（判断 value 是否匹配再删除）。Redisson 框架提供了更完善的实现，支持可重入锁、看门狗自动续期、RedLock 算法（多节点加锁，解决单点故障）。

基于 Zookeeper：利用临时顺序节点实现，创建节点成功且序号最小的获取锁，其他节点监听前一个节点的删除事件。优点是可靠性高，缺点是性能不如 Redis。

基于数据库：悲观锁（`SELECT ... FOR UPDATE`）或乐观锁（版本号机制），实现简单但性能差，不推荐高并发场景使用。

---

## 五、AOP（面向切面编程）

### 5.1 AOP 基础

**Q：什么是 AOP？核心概念有哪些？**

AOP（Aspect-Oriented Programming）是一种编程范式，通过将横切关注点（如日志、事务、权限、缓存等）从业务逻辑中分离出来，实现代码的模块化和复用。

核心概念：Aspect（切面，横切关注点的模块化，如日志切面）、Join Point（连接点，程序执行的某个点，如方法调用、异常抛出，Spring AOP 只支持方法级别的连接点）、Pointcut（切入点，匹配连接点的表达式，定义在哪些方法上织入增强）、Advice（通知/增强，在连接点上执行的动作）、Target（目标对象，被代理的对象）、Proxy（代理对象）、Weaving（织入，将切面应用到目标对象的过程）。

**Q：Spring AOP 中有哪些通知类型？执行顺序是怎样的？**

五种通知类型：`@Before`（前置通知，方法执行前）、`@After`（后置通知，方法执行后，无论是否异常都执行，类似 finally）、`@AfterReturning`（返回通知，方法正常返回后）、`@AfterThrowing`（异常通知，方法抛出异常后）、`@Around`（环绕通知，包裹目标方法，可以控制是否执行目标方法以及修改返回值）。

正常执行顺序（Spring 5.2.7+）：Around（前）→ Before → 目标方法 → AfterReturning → After → Around（后）。异常执行顺序：Around（前）→ Before → 目标方法抛异常 → AfterThrowing → After。多个切面的执行顺序通过 `@Order` 注解控制，值越小优先级越高，进入时优先级高的先执行，退出时优先级高的后执行（类似洋葱模型）。

### 5.2 AOP 实现原理

**Q：Spring AOP 和 AspectJ 的区别？**

Spring AOP 是运行时通过动态代理实现的，只支持方法级别的连接点，性能开销在运行时。AspectJ 是编译时/类加载时通过字节码织入实现的，支持方法、构造器、字段等多种连接点，功能更强大但使用更复杂。Spring AOP 默认使用 AspectJ 的注解语法（`@Aspect`、`@Pointcut` 等），但底层实现是动态代理而非 AspectJ 的织入机制。

**Q：JDK 动态代理和 CGLIB 代理的区别？**

JDK 动态代理：基于接口，通过 `java.lang.reflect.Proxy` 和 `InvocationHandler` 实现，要求目标类必须实现接口。运行时通过反射生成代理类（`$Proxy0`），代理类实现了目标接口。

CGLIB 代理：基于继承，通过 ASM 字节码框架在运行时生成目标类的子类作为代理类，通过 `MethodInterceptor` 拦截方法调用。不需要目标类实现接口，但无法代理 final 类和 final 方法。

Spring AOP 的选择策略：如果目标类实现了接口，默认使用 JDK 动态代理（Spring Boot 2.x 默认改为 CGLIB）；如果没有实现接口，使用 CGLIB。可以通过 `@EnableAspectJAutoProxy(proxyTargetClass = true)` 强制使用 CGLIB。Spring Boot 2.x 默认 `spring.aop.proxy-target-class=true`，即默认使用 CGLIB。

**Q：为什么同一个类中方法调用 AOP 不生效？如何解决？**

因为 Spring AOP 基于代理实现，只有通过代理对象调用的方法才会被拦截。同一个类中 `this.method()` 调用的是目标对象本身而非代理对象，所以 AOP 不生效。

解决方案：注入自身（`@Autowired` 注入自己，通过注入的引用调用）；从 ApplicationContext 获取代理对象；使用 `AopContext.currentProxy()` 获取当前代理对象（需要 `@EnableAspectJAutoProxy(exposeProxy = true)`）；将方法抽取到另一个 Bean 中；使用 AspectJ 的编译时织入。

### 5.3 AOP 应用场景

**Q：AOP 在实际项目中有哪些应用场景？**

日志记录（记录方法入参、出参、执行时间）、权限校验（在方法执行前检查用户权限）、事务管理（Spring 的 `@Transactional` 就是基于 AOP 实现的）、缓存处理（`@Cacheable`）、异常处理（统一异常捕获和处理）、性能监控（方法执行耗时统计）、接口限流（通过自定义注解 + AOP 实现）、数据脱敏、审计日志、分布式锁（通过注解 + AOP 自动加锁释放锁）。

**Q：如何自定义一个 AOP 注解？**

步骤：定义自定义注解（`@Target(ElementType.METHOD) @Retention(RetentionPolicy.RUNTIME)`）→ 编写切面类（`@Aspect @Component`）→ 定义切入点（`@Pointcut("@annotation(com.example.MyAnnotation)")`）→ 编写通知方法（如 `@Around`）→ 在通知方法中通过 `ProceedingJoinPoint` 获取方法信息、注解参数等，执行增强逻辑。

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface OperationLog {
    String value() default "";
}

@Aspect
@Component
public class OperationLogAspect {
    @Around("@annotation(operationLog)")
    public Object around(ProceedingJoinPoint joinPoint, OperationLog operationLog) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            Object result = joinPoint.proceed();
            // 记录成功日志
            return result;
        } catch (Throwable e) {
            // 记录异常日志
            throw e;
        } finally {
            long cost = System.currentTimeMillis() - start;
            // 记录耗时
        }
    }
}
```

---

## 六、事务

### 6.1 事务基础

**Q：什么是事务？事务的 ACID 特性是什么？**

事务是一组不可分割的操作序列，要么全部成功，要么全部失败回滚。ACID 特性：原子性（Atomicity，事务中的操作要么全部完成要么全部不完成，通过 undo log 实现回滚）、一致性（Consistency，事务前后数据库从一个一致状态转换到另一个一致状态）、隔离性（Isolation，并发事务之间互不干扰，通过锁和 MVCC 实现）、持久性（Durability，事务提交后数据永久保存，通过 redo log 实现）。其中一致性是目的，原子性、隔离性、持久性是手段。

**Q：并发事务会带来哪些问题？**

脏读（Dirty Read）：事务 A 读取了事务 B 未提交的数据，如果 B 回滚，A 读到的就是无效数据。不可重复读（Non-Repeatable Read）：事务 A 两次读取同一数据，期间事务 B 修改并提交了该数据，导致 A 两次读取结果不同（针对 update）。幻读（Phantom Read）：事务 A 两次查询同一范围的数据，期间事务 B 插入了新数据并提交，导致 A 第二次查询多出了数据（针对 insert/delete）。

**Q：事务的隔离级别有哪些？**

读未提交（Read Uncommitted）：最低级别，可能出现脏读、不可重复读、幻读。读已提交（Read Committed）：只能读取已提交的数据，解决脏读，Oracle 默认级别。可重复读（Repeatable Read）：同一事务中多次读取结果一致，解决脏读和不可重复读，MySQL InnoDB 默认级别（通过 MVCC + Next-Key Lock 在很大程度上解决了幻读）。串行化（Serializable）：最高级别，完全串行执行，解决所有问题但性能最差。

**Q：MySQL InnoDB 的 MVCC 是如何实现的？**

MVCC（Multi-Version Concurrency Control，多版本并发控制）通过为每行数据维护多个版本来实现非锁定读。InnoDB 中每行数据有两个隐藏列：`DB_TRX_ID`（最近修改该行的事务 ID）和 `DB_ROLL_PTR`（指向 undo log 中该行的上一个版本）。

核心机制是 **ReadView**（读视图）：在 RC 级别下每次 SELECT 都创建新的 ReadView，在 RR 级别下只在事务第一次 SELECT 时创建。ReadView 包含：`m_ids`（当前活跃的事务 ID 列表）、`min_trx_id`（最小活跃事务 ID）、`max_trx_id`（下一个将分配的事务 ID）、`creator_trx_id`（创建该 ReadView 的事务 ID）。

可见性判断规则：如果行的 `DB_TRX_ID` < `min_trx_id`，说明该版本在 ReadView 创建前已提交，可见；如果 >= `max_trx_id`，说明是之后开启的事务修改的，不可见；如果在 `min_trx_id` 和 `max_trx_id` 之间，检查是否在 `m_ids` 中，在则不可见（未提交），不在则可见（已提交）。不可见时沿 undo log 版本链向前查找，直到找到可见的版本。

### 6.2 Spring 事务管理

**Q：Spring 事务的实现原理？**

Spring 事务管理基于 AOP 实现。使用 `@Transactional` 注解时，Spring 通过 `TransactionInterceptor`（事务拦截器）在方法执行前后进行事务管理。核心流程：方法执行前，根据事务属性（传播行为、隔离级别等）通过 `PlatformTransactionManager` 开启事务 → 执行目标方法 → 如果正常返回则提交事务 → 如果抛出异常则根据 rollbackFor 配置决定是否回滚。

底层实现：通过 `DataSourceTransactionManager` 获取数据库连接，设置 `autoCommit = false`，将连接绑定到 `ThreadLocal`（通过 `TransactionSynchronizationManager`），保证同一事务中的多次数据库操作使用同一个连接。

**Q：Spring 事务的传播行为有哪些？**

七种传播行为：

`REQUIRED`（默认）：如果当前存在事务则加入，不存在则创建新事务。最常用的传播行为。

`REQUIRES_NEW`：无论当前是否存在事务，都创建新事务。如果当前存在事务，则挂起当前事务。适用于不希望受外部事务影响的场景（如操作日志记录，即使业务回滚日志也要保存）。

`NESTED`：如果当前存在事务，则在嵌套事务中执行（通过 Savepoint 实现），嵌套事务可以独立回滚而不影响外部事务；如果不存在事务，则等同于 REQUIRED。

`SUPPORTS`：如果当前存在事务则加入，不存在则以非事务方式执行。

`NOT_SUPPORTED`：以非事务方式执行，如果当前存在事务则挂起。

`MANDATORY`：必须在事务中执行，如果当前没有事务则抛出异常。

`NEVER`：必须在非事务环境下执行，如果当前存在事务则抛出异常。

**Q：REQUIRED、REQUIRES_NEW 和 NESTED 的区别？**

REQUIRED：加入外部事务，内外是同一个事务，任何一方回滚都会导致整个事务回滚。

REQUIRES_NEW：创建独立的新事务，内外是两个独立事务，内部事务回滚不影响外部事务，外部事务回滚也不影响已提交的内部事务。

NESTED：嵌套事务，内部事务是外部事务的子事务，内部事务回滚不影响外部事务（回滚到 Savepoint），但外部事务回滚会导致内部事务也回滚。

### 6.3 事务失效场景

**Q：@Transactional 在哪些情况下会失效？**

这是高频面试题，常见的失效场景包括：

**方法访问权限问题**：`@Transactional` 只能应用于 public 方法。Spring 的 `TransactionInterceptor` 在执行前会检查方法的修饰符，非 public 方法事务不生效（使用 CGLIB 代理时，protected 和 package-private 方法也不会被代理）。

**自调用问题**：同一个类中，非事务方法调用事务方法（`this.transactionalMethod()`），由于绕过了代理对象，事务不生效。解决方案见 AOP 章节。

**异常被捕获**：方法内部 try-catch 捕获了异常但没有重新抛出，Spring 感知不到异常，不会触发回滚。

**异常类型不匹配**：`@Transactional` 默认只对 RuntimeException 和 Error 回滚，对 checked Exception（如 IOException）不回滚。解决方案：`@Transactional(rollbackFor = Exception.class)`。

**数据库引擎不支持事务**：如 MySQL 的 MyISAM 引擎不支持事务，需要使用 InnoDB。

**Bean 未被 Spring 管理**：类没有被 Spring 容器管理（没有 `@Service`、`@Component` 等注解），AOP 代理不会生效。

**传播行为设置不当**：如设置为 `NOT_SUPPORTED` 或 `NEVER`，方法不会在事务中执行。

**多线程调用**：事务基于 ThreadLocal 绑定数据库连接，新线程中的操作不在同一个事务中。

### 6.4 分布式事务

**Q：什么是分布式事务？有哪些解决方案？**

分布式事务是指跨多个数据库、多个服务的事务，需要保证这些操作要么全部成功要么全部失败。

**2PC（两阶段提交）**：协调者先向所有参与者发送 Prepare 请求，参与者执行事务但不提交，返回 Yes/No；如果所有参与者都返回 Yes，协调者发送 Commit 请求，否则发送 Rollback。缺点：同步阻塞、单点故障、数据不一致风险。

**3PC（三阶段提交）**：在 2PC 基础上增加了 CanCommit 阶段和超时机制，降低了阻塞问题，但仍无法完全解决数据不一致。

**TCC（Try-Confirm-Cancel）**：业务层面的两阶段提交。Try 阶段预留资源（如冻结库存）；Confirm 阶段确认执行（如扣减库存）；Cancel 阶段取消执行（如释放冻结的库存）。需要业务代码实现三个接口，侵入性强，但灵活性高。需要注意幂等性、空回滚、悬挂等问题。

**SAGA**：将长事务拆分为多个本地事务，每个本地事务有对应的补偿事务。正向执行所有本地事务，如果某个失败则逆序执行已完成事务的补偿操作。适合长流程业务，但不保证隔离性。

**本地消息表**：在本地事务中将消息写入消息表，通过定时任务扫描消息表发送消息到 MQ，消费者消费消息执行本地事务。通过重试和幂等保证最终一致性。

**可靠消息最终一致性**：基于 RocketMQ 的事务消息。发送半消息（Half Message）→ 执行本地事务 → 根据本地事务结果提交或回滚消息 → 消费者消费消息。RocketMQ 提供回查机制，如果长时间未收到确认，会主动回查本地事务状态。

**Seata**：阿里开源的分布式事务框架，支持 AT（自动补偿，基于 undo log，对业务无侵入）、TCC、SAGA、XA 四种模式。AT 模式原理：一阶段拦截 SQL 解析，保存修改前后的数据快照（undo log），执行 SQL 并提交本地事务；二阶段如果全局提交则删除 undo log，如果全局回滚则根据 undo log 反向补偿。

**Q：如何选择分布式事务方案？**

强一致性要求高、参与者少：2PC/XA（如 Seata XA 模式）。对业务无侵入、性能要求适中：Seata AT 模式。高性能、可接受最终一致性：TCC 或可靠消息。长流程业务：SAGA。简单场景、可接受最终一致性：本地消息表。实际项目中，大多数互联网场景选择最终一致性方案（可靠消息或 Seata AT），因为强一致性方案的性能开销太大。

---

## 七、补充高频题

### 7.1 设计模式

**Q：Spring 中用到了哪些设计模式？**

工厂模式（BeanFactory、FactoryBean）、单例模式（Bean 默认 singleton）、代理模式（AOP 的 JDK 动态代理和 CGLIB）、模板方法模式（JdbcTemplate、RestTemplate）、观察者模式（ApplicationEvent 和 ApplicationListener）、适配器模式（HandlerAdapter）、策略模式（Resource 接口的不同实现）、责任链模式（Filter Chain、Interceptor Chain）、装饰器模式（BeanWrapper）。

### 7.2 MySQL 相关（与事务紧密相关）

**Q：InnoDB 中有哪些锁？**

按粒度分：行锁（Record Lock，锁定单行记录）、间隙锁（Gap Lock，锁定索引记录之间的间隙，防止幻读）、临键锁（Next-Key Lock = Record Lock + Gap Lock，InnoDB 默认的行锁算法）、表锁。

按类型分：共享锁（S 锁，读锁，`SELECT ... LOCK IN SHARE MODE`）、排他锁（X 锁，写锁，`SELECT ... FOR UPDATE`、INSERT/UPDATE/DELETE）、意向锁（IS/IX，表级锁，用于快速判断表中是否有行锁）。

按思想分：乐观锁（通过版本号或时间戳实现，适合读多写少）、悲观锁（通过数据库锁机制实现，适合写多读少）。

**Q：什么是死锁？如何避免？**

死锁是两个或多个事务互相等待对方持有的锁，导致所有事务都无法继续执行。InnoDB 通过等待图（wait-for graph）检测死锁，发现后回滚代价最小的事务。

避免死锁的方法：按固定顺序访问表和行；尽量缩短事务执行时间；降低隔离级别；为表添加合理的索引（避免全表扫描导致锁范围过大）；使用 `LOCK TABLES` 一次性锁定所需资源。

---

> **使用建议**：本文档覆盖了 Java 后端面试中最高频的知识点，建议结合实际项目经验理解每个知识点，而不是死记硬背。面试时能够结合场景说出自己的理解和实践经验，会比单纯背诵答案更有说服力。重点关注 JVM 调优实战、Spring Boot 自动配置原理、微服务治理方案选型、事务失效场景排查等实战性强的问题。
