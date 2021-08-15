# Java内存访问重排序的研究
## 什么是重排序

请先看这样一段代码：

```null
public class PossibleReordering {
static int x = 0, y = 0;
static int a = 0, b = 0;

public static void main(String[] args) throws InterruptedException {
    Thread one = new Thread(new Runnable() {
        public void run() {
            a = 1;
            x = b;
        }
    });
    
    Thread other = new Thread(new Runnable() {
        public void run() {
            b = 1;
            y = a;
        }
    });
    one.start();other.start();
    one.join();other.join();
    System.out.println(“(” + x + “,” + y + “)”);
}
```

很容易想到这段代码的运行结果可能为 (1,0)、(0,1) 或(1,1)，因为线程 one 可以在线程 two 开始之前就执行完了，也有可能反之，甚至有可能二者的指令是同时或交替执行的。

然而，这段代码的执行结果也可能是 (0,0). 因为，在实际运行时，代码指令可能并不是严格按照代码语句顺序执行的。得到(0,0) 结果的语句执行过程，如下图所示。值得注意的是，a=1 和 x=b 这两个语句的赋值操作的顺序被颠倒了，或者说，发生了指令“重排序”(reordering)。（事实上，输出了这一结果，并不代表一定发生了指令重排序，内存可见性问题也会导致这样的输出，详见后文）

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-15%2023-35-59/466e2565-c5ec-4adf-ad91-65373b578156.png?raw=true)

重排序图解

对重排序现象不太了解的开发者可能会对这种现象感到吃惊，但是，笔者开发环境下做的一个小实验证实了这一结果 2。

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-15%2023-35-59/e45bd6c1-b0a0-48e1-b3bb-04d6b8dd5cc3.png?raw=true)

重排序实验

实验代码是构造一个循环，反复执行上面的实例代码，直到出现 a=0 且 b=0 的输出为止。实验结果说明，循环执行到第 13830 次时输出了 (0,0).

大多数现代微处理器都会采用将指令乱序执行（out-of-order execution，简称 OoOE 或 OOE）的方法，在条件允许的情况下，直接运行当前有能力立即执行的后续指令，避开获取下一条指令所需数据时造成的等待 3。通过乱序执行的技术，处理器可以大大提高执行效率。

除了处理器，常见的 Java 运行时环境的 JIT 编译器也会做指令重排序操作 4，即生成的机器指令与字节码指令顺序不一致。

## as-if-serial 语义

As-if-serial 语义的意思是，所有的动作 (Action)5 都可以为了优化而被重排序，但是必须保证它们重排序后的结果和程序代码本身的应有结果是一致的。Java 编译器、运行时和处理器都会保证单线程下的 as-if-serial 语义。 比如，为了保证这一语义，重排序不会发生在有数据依赖的操作之中。

-   int a = 1;
-   int b = 2;
-   int c = a + b;

将上面的代码编译成 Java 字节码或生成机器指令，可视为展开成了以下几步动作（实际可能会省略或添加某些步骤）。

1.  对 a 赋值 1
2.  对 b 赋值 2
3.  取 a 的值
4.  取 b 的值
5.  将取到两个值相加后存入 c

在上面 5 个动作中，动作 1 可能会和动作 2、4 重排序，动作 2 可能会和动作 1、3 重排序，动作 3 可能会和动作 2、4 重排序，动作 4 可能会和 1、3 重排序。但动作 1 和动作 3、5 不能重排序。动作 2 和动作 4、5 不能重排序。因为它们之间存在数据依赖关系，一旦重排，as-if-serial 语义便无法保证。

为保证 as-if-serial 语义，Java 异常处理机制也会为重排序做一些特殊处理。例如在下面的代码中，y = 0 / 0 可能会被重排序在 x = 2 之前执行，为了保证最终不致于输出 x = 1 的错误结果，JIT 在重排序时会在 catch 语句中插入错误代偿代码，将 x 赋值为 2，将程序恢复到发生异常时应有的状态。这种做法的确将异常捕捉的逻辑变得复杂了，但是 JIT 的优化的原则是，尽力优化正常运行下的代码逻辑，哪怕以 catch 块逻辑变得复杂为代价，毕竟，进入 catch 块内是一种 “异常” 情况的表现。6

```null
public class Reordering {
    public static void main(String[] args) {
        int x, y;
        x = 1;
        try {
            x = 2;
            y = 0 / 0;    
        } catch (Exception e) {
        } finally {
            System.out.println("x = " + x);
        }
    }
}
```

## 内存访问重排序与内存可见性

计算机系统中，为了尽可能地避免处理器访问主内存的时间开销，处理器大多会利用缓存 (cache) 以提高性能。其模型如下图所示：

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-15%2023-35-59/e3ef66bc-506b-4c95-836e-9ec5d68d2854.png?raw=true)

处理器 Cache 模型

在这种模型下会存在一个现象，即缓存中的数据与主内存的数据并不是实时同步的，各 CPU（或 CPU 核心）间缓存的数据也不是实时同步的。这导致在同一个时间点，各 CPU 所看到同一内存地址的数据的值可能是不一致的。从程序的视角来看，就是在同一个时间点，各个线程所看到的共享变量的值可能是不一致的。

有的观点会将这种现象也视为重排序的一种，命名为 “内存系统重排序”。因为这种内存可见性问题造成的结果就好像是内存访问指令发生了重排序一样。

这种内存可见性问题也会导致章节一中示例代码即便在没有发生指令重排序的情况下的执行结果也还是 (0, 0)。

## 内存访问重排序与 Java 内存模型

Java 的目标是成为一门平台无关性的语言，即 Write once, run anywhere. 但是不同硬件环境下指令重排序的规则不尽相同。例如，x86 下运行正常的 Java 程序在 IA64 下就可能得到非预期的运行结果。为此，JSR-1337 制定了 Java 内存模型 (Java Memory Model, JMM)，旨在提供一个统一的可参考的规范，屏蔽平台差异性。从 Java 5 开始，Java 内存模型成为 Java 语言规范的一部分。

根据 Java 内存模型中的规定，可以总结出以下几条 happens-before 规则 8。Happens-before 的前后两个操作不会被重排序且后者对前者的内存可见。

-   程序次序法则：线程中的每个动作 A 都 happens-before 于该线程中的每一个动作 B，其中，在程序中，所有的动作 B 都能出现在 A 之后。
-   监视器锁法则：对一个监视器锁的解锁 happens-before 于每一个后续对同一监视器锁的加锁。
-   volatile 变量法则：对 volatile 域的写入操作 happens-before 于每一个后续对同一个域的读写操作。
-   线程启动法则：在一个线程里，对 Thread.start 的调用会 happens-before 于每个启动线程的动作。
-   线程终结法则：线程中的任何动作都 happens-before 于其他线程检测到这个线程已经终结、或者从 Thread.join 调用中成功返回，或 Thread.isAlive 返回 false。
-   中断法则：一个线程调用另一个线程的 interrupt happens-before 于被中断的线程发现中断。
-   终结法则：一个对象的构造函数的结束 happens-before 于这个对象 finalizer 的开始。
-   传递性：如果 A happens-before 于 B，且 B happens-before 于 C，则 A happens-before 于 C

Happens-before 关系只是对 Java 内存模型的一种近似性的描述，它并不够严谨，但便于日常程序开发参考使用，关于更严谨的 Java 内存模型的定义和描述，请阅读 JSR-133 原文或 Java 语言规范章节 17.4。

除此之外，Java 内存模型对 volatile 和 final 的语义做了扩展。对 volatile 语义的扩展保证了 volatile 变量在一些情况下不会重排序，volatile 的 64 位变量 double 和 long 的读取和赋值操作都是原子的。对 final 语义的扩展保证一个对象的构建方法结束前，所有 final 成员变量都必须完成初始化（的前提是没有 this 引用溢出）。

Java 内存模型关于重排序的规定，总结后如下表所示：

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-15%2023-35-59/4604ce6e-57e9-483e-8241-31955421c567.png?raw=true)

重排序示意表

表中 “第二项操作” 的含义是指，第一项操作之后的所有指定操作。如，普通读不能与其之后的所有 volatile 写重排序。另外，JMM 也规定了上述 volatile 和同步块的规则尽适用于存在多线程访问的情景。例如，若编译器（这里的编译器也包括 JIT，下同）证明了一个 volatile 变量只能被单线程访问，那么就可能会把它做为普通变量来处理。

留白的单元格代表允许在不违反 Java 基本语义的情况下重排序。例如，编译器不会对对同一内存地址的读和写操作重排序，但是允许对不同地址的读和写操作重排序。

除此之外，为了保证 final 的新增语义。JSR-133 对于 final 变量的重排序也做了限制。

-   构建方法内部的 final 成员变量的存储，并且，假如 final 成员变量本身是一个引用的话，这个 final 成员变量可以引用到的一切存储操作，都不能与构建方法外的将当期构建对象赋值于多线程共享变量的存储操作重排序。例如对于如下语句：

> x.finalField = v; … ; 构建方法边界 sharedRef = x; v.afield = 1; x.finalField = v; … ; 构建方法边界 sharedRef = x;

这两条语句中，构建方法边界前后的指令都不能重排序。

-   初始读取共享对象与初始读取该共享对象的 final 成员变量之间不能重排序。例如对于如下语句：

> x = sharedRef; … ; i = x.finalField;

前后两句语句之间不会发生重排序。由于这两句语句有数据依赖关系，编译器本身就不会对它们重排序，但确实有一些处理器会对这种情况重排序，因此特别制定了这一规则。

## 内存屏障

内存屏障（Memory Barrier，或有时叫做内存栅栏，Memory Fence）是一种 CPU 指令，用于控制特定条件下的重排序和内存可见性问题。Java 编译器也会根据内存屏障的规则禁止重排序。

内存屏障可以被分为以下几种类型：

-   LoadLoad 屏障：对于这样的语句 Load1; LoadLoad; Load2，在 Load2 及后续读取操作要读取的数据被访问前，保证 Load1 要读取的数据被读取完毕。
-   StoreStore 屏障：对于这样的语句 Store1; StoreStore; Store2，在 Store2 及后续写入操作执行前，保证 Store1 的写入操作对其它处理器可见。
-   LoadStore 屏障：对于这样的语句 Load1; LoadStore; Store2，在 Store2 及后续写入操作被刷出前，保证 Load1 要读取的数据被读取完毕。
-   StoreLoad 屏障：对于这样的语句 Store1; StoreLoad; Load2，在 Load2 及后续所有读取操作执行前，保证 Store1 的写入对所有处理器可见。它的开销是四种屏障中最大的。在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能。

有的处理器的重排序规则较严，无需内存屏障也能很好的工作，Java 编译器会在这种情况下不放置内存屏障。 为了实现上一章中讨论的 JSR-133 的规定，Java 编译器会这样使用内存屏障。

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-15%2023-35-59/fe35b9d7-a674-4d69-9db1-218d879efab3.png?raw=true)

内存屏障示意表

为了保证 final 字段的特殊语义，也会在下面的语句加入内存屏障。

> x.finalField = v; StoreStore; sharedRef = x;

## Intel 64/IA-32 架构下的内存访问重排序

Intel 64 和 IA-32 是我们较常用的硬件环境，相对于其它处理器而言，它们拥有一种较严格的重排序规则。Pentium 4 以后的 Intel 64 或 IA-32 处理的重排序规则如下。9

在单 CPU 系统中：

-   读操作不与其它读操作重排序。
-   写操作不与其之前的写操作重排序。
-   写内存操作不与其它写操作重排序，但有以下几种例外
-   CLFLUSH 的写操作
-   带有 non-temporal move 指令 (MOVNTI, MOVNTQ, MOVNTDQ, MOVNTPS, and MOVNTPD) 的 streaming 写入。
-   字符串操作
-   读操作可能会与其之前的写不同位置的写操作重排序，但不与其之前的写相同位置的写操作重排序。
-   读和写操作不与 I/O 指令，带锁的指令或序列化指令重排序。
-   读操作不能重排序到 LFENCE 和 MFENCE 之前。
-   写操作不能重排序到 LFENCE、SFENCE 和 MFENCE 之前。
-   LFENCE 不能重排序到读操作之前。
-   SFENCE 不能重排序到写之前。
-   MFENCE 不能重排序到读或写操作之前。

在多处理器系统中：

-   各自处理器内部遵循单处理器的重排序规则。
-   单处理器的写操作对所有处理器可见是同时的。
-   各自处理器的写操作不会重排序。
-   内存重排序遵守因果性 (causality)（内存重排序遵守传递可见性）。
-   任何写操作对于执行这些写操作的处理器之外的处理器来看都是一致的。
-   带锁指令是顺序执行的。

值得注意的是，对于 Java 编译器而言，Intel 64/IA-32 架构下处理器不需要 LoadLoad、LoadStore、StoreStore 屏障，因为不会发生需要这三种屏障的重排序。

## 一例 Intel 64/IA-32 架构下的代码性能优化

现在有这样一个场景，一个容器可以放一个东西，容器支持 create 方法来创建一个新的东西并放到容器里，支持 get 方法取到这个容器里的东西。我们可以较容易地写出下面的代码：

```null
public class Container {
    public static class SomeThing {
        private int status;

        public SomeThing() {
            status = 1;
        }

        public int getStatus() {
            return status;
        }
    }

    private SomeThing object;

    public void create() {
        object = new SomeThing();
    }

    public SomeThing get() {
        while (object == null) {
            Thread.yield(); 
        }
        return object;
    }
}
```

在单线程场景下，这段代码执行起来是没有问题的。但是在多线程并发场景下，由不同的线程 create 和 get 东西，这段代码是有问题的。问题的原因与普通的双重检查锁定单例模式 (Double Checked Locking,DCL)10 类似，即 SomeThing 的构建与将指向构建中的 SomeThing 引用赋值到 object 变量这两者可能会发生重排序。导致 get 中返回一个正被构建中的不完整的 SomeThing 对象实例。为了解决这一问题，通常的办法是使用 volatile 修饰 object 字段。这种方法避免了重排序，保证了内存可见性，摒弃比使用同步块导致的性能损失更小。但是，假如使用场景对 object 的内存可见性并不敏感的话（不要求一个线程写入了 object，object 的新值立即对下一个读取的线程可见），在 Intel 64/IA-32 环境下，有更好的解决方案。

根据上一章的内容，我们知道 Intel 64/IA-32 下写操作之间不会发生重排序，即在处理器中，构建 SomeThing 对象与赋值到 object 这两个操作之间的顺序性是可以保证的。这样看起来，仅仅使用 volatile 来避免重排序是多此一举的。但是，Java 编译器却可能生成重排序后的指令。但令人高兴的是，Oracle 的 JDK 中提供了 Unsafe. putOrderedObject，Unsafe. putOrderedInt，Unsafe. putOrderedLong 这三个方法，JDK 会在执行这三个方法时插入 StoreStore 内存屏障，避免发生写操作重排序。而在 Intel 64/IA-32 架构下，StoreStore 屏障并不需要，Java 编译器会将 StoreStore 屏障去除。比起写入 volatile 变量之后执行 StoreLoad 屏障的巨大开销，采用这种方法除了避免重排序而带来的性能损失以外，不会带来其它的性能开销。

我们将做一个小实验来比较二者的性能差异。一种是使用 volatile 修饰 object 成员变量。

```null
public class Container {
    public static class SomeThing {
        private int status;

        public SomeThing() {
            status = 1;
        }

        public int getStatus() {
            return status;
        }
    }

    private volatile  SomeThing object;

    public void create() {
        object = new SomeThing();
    }

    public SomeThing get() {
        while (object == null) {
            Thread.yield(); 
        }
        return object;
    }
}

```

一种是利用 Unsafe. putOrderedObject 在避免在适当的位置发生重排序。

```null
public class Container {
    public static class SomeThing {
        private int status;

        public SomeThing() {
            status = 1;
        }

        public int getStatus() {
            return status;
        }
    }

    private SomeThing object;

    private Object value;
    private static final Unsafe unsafe = getUnsafe();
    private static final long valueOffset;
    static {
        try {
            valueOffset = unsafe.objectFieldOffset(Container.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    public void create() {
        SomeThing temp = new SomeThing();
        unsafe.putOrderedObject(this, valueOffset, null);	
        object = temp;
    }

    public SomeThing get() {
        while (object == null) {
            Thread.yield();
        }
        return object;
    }


    public static Unsafe getUnsafe() {
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            return (Unsafe)f.get(null);
        } catch (Exception e) {
        }
        return null;
    }
}
```

由于直接调用 Unsafe.getUnsafe() 需要配置 JRE 获取较高权限，我们利用反射获取 Unsafe 中的 theUnsafe 来取得 Unsafe 的可用实例。  
unsafe.putOrderedObject(this, valueOffset, null) 这句仅仅是为了借用这句话功能的防止写重排序，除此之外无其它作用。

利用下面的代码分别测试两种方案的实际运行时间。在运行时开启 - server 和 -XX:CompileThreshold=1 以模拟生产环境下长时间运行后的 JIT 优化效果。

```null
public static void main(String[] args) throws InterruptedException {
    final int THREADS_COUNT = 20;
    final int LOOP_COUNT = 100000;

    long sum = 0;
    long min = Integer.MAX_VALUE;
    long max = 0;
    for(int n = 0;n <= 100;n++) {
        final Container basket = new Container();
        List<Thread> putThreads = new ArrayList<Thread>();
        List<Thread> takeThreads = new ArrayList<Thread>();
        for (int i = 0; i < THREADS_COUNT; i++) {
            putThreads.add(new Thread() {
                @Override
                public void run() {
                    for (int j = 0; j < LOOP_COUNT; j++) {
                        basket.create();
                    }
                }
            });
            takeThreads.add(new Thread() {
                @Override
                public void run() {
                    for (int j = 0; j < LOOP_COUNT; j++) {
                        basket.get().getStatus();
                    }
                }
            });
        }
        long start = System.nanoTime();
        for (int i = 0; i < THREADS_COUNT; i++) {
            takeThreads.get(i).start();
            putThreads.get(i).start();
        }
        for (int i = 0; i < THREADS_COUNT; i++) {
            takeThreads.get(i).join();
            putThreads.get(i).join();
        }
        long end = System.nanoTime();
        long period = end - start;
        if(n == 0) {
            continue;	
        }
        sum += (period);
        System.out.println(period);
        if(period < min) {
            min = period;
        }
        if(period > max) {
            max = period;
        }
    }
    System.out.println("Average : " + sum / 100);
    System.out.println("Max : " + max);
    System.out.println("Min : " + min);
}
```

在笔者的计算机上运行测试，采用 volatile 方案的运行结果如下：

```null
Average : 62535770
Max : 82515000
Min : 45161000
```

采用 unsafe.putOrderedObject 方案的运行结果如下：

```null
Average : 50746230
Max : 68999000
Min : 38038000
```

从结果看出，unsafe.putOrderedObject 方案比 volatile 方案平均耗时减少 18.9%，最大耗时减少 16.4%，最小耗时减少 15.8%. 另外，即使在其它会发生写写重排序的处理器中，由于 StoreStore 屏障的性能损耗小于 StoreLoad 屏障，采用这一方法也是一种可行的方案。但值得再次注意的是，这一方案不是对 volatile 语义的等价替换，而是在特定场景下做的特殊优化，它仅避免了写写重排序，但不保证内存可见性。

## 序

1.  样例选自《Java 并发编程实践》章节 16.1
2.  实验代码见附 1
3.  [http://en.wikipedia.org/wiki/Out-of-order_execution](http://en.wikipedia.org/wiki/Out-of-order_execution)
4.  Oracle Java Hotspot [https://wikis.oracle.com/display/HotSpotInternals/PerformanceTacticIndex](https://wikis.oracle.com/display/HotSpotInternals/PerformanceTacticIndex) IBM JVM [http://publib.boulder.ibm.com/infocenter/javasdk/v1r4m2/index.jsp?topic=%2Fcom.ibm.java.doc.diagnostics.142j9%2Fhtml%2Fhowjitopt.html](http://publib.boulder.ibm.com/infocenter/javasdk/v1r4m2/index.jsp?topic=%2Fcom.ibm.java.doc.diagnostics.142j9%2Fhtml%2Fhowjitopt.html)
5.  Java 语言规范中对 “动作” 这个词有一个明确而具体的定义，详见[http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.2。](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.2%E3%80%82)
6.  [https://community.oracle.com/thread/1544959](https://community.oracle.com/thread/1544959)
7.  [http://www.cs.umd.edu/~pugh/java/memoryModel/jsr133.pdf](http://www.cs.umd.edu/~pugh/java/memoryModel/jsr133.pdf)
8.  参见《Java 并发编程实践》章节 16.1
9.  Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 3 (3A, 3B & 3C): System Programming Guide 章节 8.2
10. [http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html](http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html)

### 附 1 复现重排序现象实验代码

```null
public class Test {
    private static int x = 0, y = 0;
    private static int a = 0, b =0;

    public static void main(String[] args) throws InterruptedException {
        int i = 0;
        for(;;) {
            i++;
            x = 0; y = 0;
            a = 0; b = 0;
            Thread one = new Thread(new Runnable() {
                public void run() {
                    
                    shortWait(100000);
                    a = 1;
                    x = b;
                }
            });

            Thread other = new Thread(new Runnable() {
                public void run() {
                    b = 1;
                    y = a;
                }
            });
            one.start();other.start();
            one.join();other.join();
            String result = "第" + i + "次 (" + x + "," + y + "）";
            if(x == 0 && y == 0) {
                System.err.println(result);
                break;
            } else {
                System.out.println(result);
            }
        }
    }


    public static void shortWait(long interval){
        long start = System.nanoTime();
        long end;
        do{
            end = System.nanoTime();
        }while(start + interval >= end);
    }
}

```
