# Java异步编程 
随着`RxJava`、`Reactor`等异步框架的流行，异步编程受到了越来越多的关注，尤其是在 IO 密集型的业务场景中，相比传统的同步开发模式，异步编程的优势越来越明显。

那到底什么是异步编程？异步化真正的好处又是什么？如何选择适合自己团队的异步技术？在实施异步框架落地的过程中有哪些需要注意的地方？

本文从以下几个方面结合真实项目异步改造经验对异步编程进行分析，希望能给大家一些客观认识：

1.  使用 RxJava 异步改造后的效果
2.  什么是异步编程？异步实现原理
3.  异步技术选型参考
4.  异步化真正的好处是什么？
5.  异步化落地的难点及解决方案
6.  扩展: 异步其他解决方案 - 协程

## 使用 RxJava 异步改造后的效果

下图是我们后端 java 项目使用 RxJava 改造成异步前后的 RT(响应时长) 效果对比：

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/6e4b11b5-1432-43e5-9d97-e64900558643.webp?raw=true)

image

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/72098355-3d8d-44df-a152-867979b45701.webp?raw=true)

image

统计数据基于 App 端的 gateway，以 75 线为准，还有 80、85、90、99 线，从图中可以看出改成异步后接口整体的平均响应时长降低了 **40%**左右。

(响应时间是以发送请求到收到后端接口响应数据的时长，上图改造的这个后端 java 接口内部流程比较复杂，因为公司都是微服务架构，该接口内部又调用了 6 个其他服务的接口，最后把这些接口的数据汇总在一起返回给前端)

这张图是同步接口和改造成异步接口前后的 CPU 负载情况对比

改造前 cpu load : 35.46

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/3176afd2-90e0-4b24-97a7-f9c2f66cdc00.webp?raw=true)

image

改造后 cpu load : 14.25

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/1a6269a5-84b2-4947-a97d-46769e609425.webp?raw=true)

image

改成异步后 CPU 的负载情况也有明显下降，但 CPU 使用率并无影响 (一般情况下异步化后 cpu 的利用率会有所提高，但要看具体的业务场景)

CPU LoadAverage 是指：一段时间内处于可运行状态和不可中断状态的进程平均数量。(可运行分为正在运行进程和正在等待 CPU 的进程；**不可中断则是它正在做某些工作不能被中断比如等待磁盘 IO、网络 IO 等**)

而我们的服务业务场景大部分都是 IO 密集型业务，功能实现很多需要依赖底层接口，会进行频繁的 IO 操作。

下图是 2019 年在全球架构师峰会上**阿里**分享的异步化改造后的 RT 和 QPS 效果：

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/57834315-4fe8-4330-8113-f30618d51257.webp?raw=true)

image

（图片来源：淘宝应用架构升级——反应式架构的探索与实践）

## 什么是异步编程？

### 响应式编程 + NIO

### 1. 异步和同步的区别：

我们先从 **I/O** 的角度看下同步模式下接口 A 调用接口 B 的交互流程:

下图是传统的同步模式下 io 线程的交互流程，可以看出 io 是阻塞的，即 bio 的运行模式

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/5a061793-d06c-4f58-92d7-762a7939ebe3.webp?raw=true)

image

接口 A 发起调用接口 B 后，这段时间什么事情也不能做，主线程阻塞一直等到接口 B 数据返回，然后才能进行其他操作，可想而知如果接口 A 调用的接口不止 B 的话 (A->B->C->D->E。。。)，那么等待的时间也是递增的，而且**这期间 CPU 也要一直占用着**，白白浪费资源，也就是上图看到的 cpu load 高的原因。

而且还有一个隐患就是如果调用的其他服务中的接口比如 C 超时，或接口 C 挂掉了，那么对调用方服务 A 来说，剩余的接口比如 D、E 都会无限等待下去。。。

其实大部分情况下我们收到数据后内部的处理逻辑耗时都很短，这个可以通过埋点执行时间统计，**大部分时间都浪费在了 IO 等待上**。

下面这个视频演示了同步模式下我们线上环境真实的接口调用情况，即接口调用的线程执行和变化情况，(使用的工具是 JDK 自带的 jvisual 来监控线程变化情况)

这里先交代下大致背景：服务端 api 接口 A 内部一共调用了 6 个其他服务的接口，大致交互是这样的：

A 接口（B -> C -> D -> E -> F -> G）返回聚合数据

背景：使用 Jemter 测试工具压测 100 个线程并发请求接口，以观察线程的运行情况（可以全屏观看）：

`http-nio-8080-exec*`开头的是 tomcat 线程池中的线程，即前端请求我们后端接口时要通过 tomcat 服务器接收和转发的线程，因为我们后端 api 接口内部又调用了其他服务的 6 个接口（B、C、D、E、F、G），同步模式下需要等待上一个接口返回数据才能继续调用下一个接口，所以可以从视频中看出，大部分的 http 线程耗时都在 8 秒以上 (绿色线条代表线程是 "运行中" 状态，8 秒包括等待接口返回的时间和我们内部逻辑处理的总时间，因为是本地环境测试，受机器和网络影响较大)

然后我们再看下异步模式的交互流程，即 nio 方式：

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/bd6a9b48-9175-4d1d-b8e7-45871144f058.webp?raw=true)

image

大致流程就是接口 A 发起调用接口 B 的请求后就立即返回，而不用阻塞等待接口 B 响应，这样的好处是`http-nio-8080-exec*`线程可以**马上得到复用，接着处理下一个前端请求的任务**，如果接口 B 处理完返回数据后，会有一个回调线程池处理真正的响应，即这种模式下我们的业务流程是 **http 线程只处理请求，回调线程处理接口响应**。

这个视频演示了异步模式下接口 A 的线程执行情况，同样也是使用 Jemter 测试工具压测 100 个线程并发请求接口，以观察线程的运行情况（可以全屏观看）：

模拟的条件和同步模式一样，同样是 100 个线程并发请求接口，但这次`http-nio-8080-exec*`开头的线程只处理请求任务，而不再等待全部的接口返回，所以 http 的线程运行时间普遍都很短 (大部分在 1.8 秒左右完成)，`AsfThread-executor-*`是我们系统封装的回调线程池，处理底层接口的真正响应数据。

演示视频中的`AsfThread-executor-*`的回调线程只创建了 30 多个，而请求的 http 线程有 100 个，也就是说这 30 多个回调线程处理了接口 B 的 100 次响应 (其实应该是 600 次，因为接口 B 内部又调用了 6 个其他接口，这 6 次也都是在异步线程里处理响应的)，因为每个接口返回的时间不一样，加上网络传输的时间，所以可以利用这个时间差充分复用线程即 cpu 资源，视频中回调线程`AsfThread-executor-*`的绿色运行状态是多段的，表示复用了多次，也就是少量回调线程处理了全部 (600 次) 的响应，这正是 **IO 多路复用**的机制。

nio 模式下虽然`http-nio-8080-exec*`线程和回调线程`AsfThread-executor-*`的运行时间都很短，但是从 http 线程开始到 asf 回调处理完返回给前端结果的时间和 bio 即同步模式下的时间差异不大（在相同的逻辑流程下），并不是 nio 模式下服务响应的整体时间就会缩短，而是**会提升 CPU 的利用率**，因为 CPU 不再会阻塞等待（不可中断状态减少），这样 **CPU 就能有更多的资源来处理其他的请求任务**，相同单位时间内能处理更多的任务，所以 nio 模式带来的好处是：

-   **提升 QPS（用更少的线程资源实现更高的并发能力）**
-   **降低 CPU 负荷, 提高利用率**

### 2. Nio 原理

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/431e83f4-a737-43e7-b545-33acce716610.webp?raw=true)

image

结合上面的接口交互图可知，接口 B 通过网络返回数据给调用方 (接口 A) 这一过程，对应底层实现就是网卡接收到返回数据后，通过自身的 DMA（直接内存访问）将数据拷贝到内核缓冲区，这一步不需要 CPU 参与操作，也就是把原先 CPU 等待的事情交给了底层网卡去处理，这样 **CPU 就可以专注于我们的应用程序即接口内部的逻辑运算**。

### 3. Nio In Java

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/8f4e922a-f1d3-40f2-8498-ff584ac415e8.webp?raw=true)

image

nio 在 java 里的实现主要是上图中的几个核心组件：`channel`、`buffer`、`selector`，这些组件组合起来即实现了上面所讲的**多路复用机制**，如下图所示：

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/dc73c54b-1f5d-4efc-b43c-ebbc8e9481f9.webp?raw=true)

image

## 响应式编程

### 1. 什么是响应式编程？它和传统的编程方式有什么区别？

响应式可以简单的理解为收到某个事件或通知后采取的一系列动作，如上文中所说的响应操作系统的网络数据通知，然后以**回调的方式**处理数据。

传统的命令式编程主要由：顺序、分支、循环 等控制流来完成不同的行为

响应式编程的特点是：

-   **以逻辑为中心转换为以数据为中心**
-   **从命令式到声明式的转换**

### 2. Java.Util.Concurrent.Future

在 Java 使用 nio 后无法立即拿到真实的数据，而且先得到一个 "`future`"，可以理解为邮戳或快递单，为了获悉真正的数据我们需要不停的通过快递单号查询快递进度，所以 **J.U.C 中的 Future 是 Java 对异步编程的第一个解决方案**，通常和线程池结合使用，伪代码形式如下：

    ExecutorService executor = Executors.newCachedThreadPool(); 

复制代码

`Future`的缺点很明显：

-   无法方便得知任务何时完成
-   无法方便获得任务结果
-   在主线程获得任务结果会导致主线程阻塞

### 3. ListenableFuture

Google 并发包下的`listenableFuture`对 Java 原生的 future 做了扩展，顾名思义就是使用监听器模式实现的**回调机制**，所以叫可监听的 future。

    Futures.addCallback(listenableFuture, new FutureCallback<String>() {

复制代码

回调机制的最大问题是：**Callback Hell（回调地狱）**

试想如果调用的接口多了，而且接口之间有依赖的话，最终写出来的代码可能就是下面这个样子：

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/23406713-7df9-4de2-aa31-ad7e6641208b.webp?raw=true)

image

-   代码的字面形式和其所表达的业务含义不匹配
-   业务的先后关系在代码层面变成了包含和被包含的关系
-   大量使用 Callback 机制，使应该是先后的业务逻辑在代码形式上表现为层层嵌套, 这会导致代码难以理解和维护。

那么如何解决 Callback Hell 问题呢？

**响应式编程**

其实主要是以下两种解决方式：

-   事件驱动机制
-   链式调用 (Lambda)

### 4. CompletableFuture

Java8 里的`CompletableFuture`和 Java9 的`Flow Api`勉强算是上面问题的解决方案：

    CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() ->

复制代码

但`CompletableFuture`处理简单的任务可以使用，但并不是一个完整的反应式编程解决方案，在服务调用复杂的情况下，存在服务编排、上下文传递、柔性限流 (背压) 方面的不足

如果使用`CompletableFuture`面对这些问题可能需要自己额外造一些轮子，Java9 的`Flow`虽然是基于 **Reactive Streams** 规范实现的，但没有 RxJava、Project Reactor 这些异步框架丰富和强大和完整的解决方案。

当然如果接口逻辑比较简单，完全可以使用`listenableFuture`或`CompletableFuture`，关于他们的详细用法可参考之前的一篇文章：[Java 异步编程指南](http://javakk.com/225.html)  

### 5. Reactive Streams

在网飞推出 RxJava1.0 并在 Android 端普及流行开后，响应式编程的规范也呼之欲出：

[https://www.reactive-streams.org/](http://javakk.com/redirect/aHR0cHM6Ly93d3cucmVhY3RpdmUtc3RyZWFtcy5vcmcv)  

包括后来的 RxJava2.0、Project Reactor 都是基于 Reactive Streams 规范实现的。

关于他们和`listenableFuture`、 `CompletableFuture`的区别通过下面的例子大家应该就会清楚。

比如下面的基于回调的代码示例：获取用户的 5 个收藏列表功能

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/bdabcc12-2936-439b-8945-a0017676723e.webp?raw=true)

image

图中标注序号的步骤对应如下：

1.  根据 uid 调用用户收藏列表接口`userService.getFavorites`  
2.  成功的回调逻辑
3.  如果用户收藏列表为空
4.  调用推荐服务`suggestionService.getSuggestions`  
5.  推荐服务成功后的回调逻辑
6.  取前 5 条推荐并展示 (`Java8 Stream api`)
7.  推荐服务失败的回调, 展示错误信息
8.  如果用户收藏列表有数据返回
9.  取前 5 条循环调用详情接口`favoriteService.getDetails` 成功回调则展示详情, 失败回调则展示错误信息

可以看出主要逻辑都是在回调函数（`onSuccess()`、`onError()`）中处理的，在可读性和后期维护成本上比较大。

基于 Reactive Streams 规范实现的响应式编程解决方案如下：

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/9c1bdcf0-5dd7-4f98-a57e-e32717024977.webp?raw=true)

image

1.  调用用户收藏列表接口
2.  压平数据流调用详情接口
3.  如果收藏列表为空调用推荐接口
4.  取前 5 条
5.  切换成异步线程处理上述声明接口返回结果)
6.  成功则展示正常数据, 错误展示错误信息

可以看出因为这些异步框架提供了丰富的 api，所以我们可以把主要精力**放在数据的流转上，而不是原来的逻辑控制上。这也是异步编程带来的思想上的转变。** 

下图是 RxJava 的`operator api`：

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/c4c81e50-ab46-404a-aa92-a4063cb54a01.webp?raw=true)

image

（如果这些操作符满足不了你的需求，你也可以自定义操作符）

所以说**异步最吸引人的地方在于资源的充分利用，不把资源浪费在等待的时间上 (nio)，代价是增加了程序的复杂度，而 Reactive Program 封装了这些复杂性，使其变得简单。** 

所以我们无论使用哪种异步框架，尽量使用框架提供的 api，而不是像上图那种基于回调业务的代码，把业务逻辑都写在 onSuccess、onError 等回调方法里，这样无法发挥异步框架的真正作用：

> Codes Like Sync，Works Like Async

即以**同步的方式编码，达到异步的效果与性能, 兼顾可维护性与可伸缩性**。

## 异步框架技术选型

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/c240222c-296b-434b-9f76-e2e7a46caeed.webp?raw=true)

（图片来源：淘宝应用架构升级——反应式架构的探索与实践）

上面这张图也是阿里在 2019 年的深圳全球架构师峰会上分享的 PPT 截图（文章末尾有链接），供大家参考，选型标准主要是基于稳定性、普及性、成本这 3 点考虑

如果是我个人更愿意选择 Project Reactor 作为首选异步框架，（具体差异网上很多分析，大家可以自行百度谷歌），还有一点是因为 Netflix 的尿性，推出的开源产品渐渐都不维护了，而且 Project Reactor 提供了`reactor-adapter`组件，可以方便的和 RxJava 的 api 转换。

其实还有 **Vert.x** 也算异步框架 (底层使用 netty 实现 nio, 最新版已支持 reactive stream 规范)

## 异步化真正的好处

### Scalability

伸缩性主要体现在以下两个方面：

-   **elastic 弹性**
-   **resilient 容错性**

（异步化在平时**不会明显降低 RT、提高 QPS**，文章开头的数据也是在大促这种流量高峰下的体现出的异步效果）

从架构和应用等更高纬度看待异步带来的好处则会提升系统的两大能力：**弹性** 和 **容错性**

前者反映了系统应对压力的表现，后者反映了系统应对故障的表现

#### 1. 容错性

像 RxJava，Reactor 这些异步框架处理回调数据时一般会切换线程上下文，其实就是使用不同的线程池来隔离不同的数据流处理逻辑，下图说明了这一特性的好处：

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/23a9bc85-939b-4eec-a07e-cf52ff9cebd4.webp?raw=true)

image

即利用异步框架支持线程池切换的特性实现**服务 / 接口隔离**，进而提高系统的**高可用**。

#### 2. 弹性

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/e2318819-f23b-491a-a40b-7c1272fb47b5.webp?raw=true)

image

back-pressure 是一种重要的反馈机制，相比于传统的熔断限流等方式，是一种更加**柔性的自适应限流**。使得系统得以优雅地响应负载，而不是在负载下崩溃。

## 异步化落地的难点及解决方案

还是先看下淘宝总结的异步改造中难点问题：

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-8%2023-35-25/c2a2686e-b3f1-4179-b2c0-955882d127a0.webp?raw=true)

image

（图片来源：淘宝应用架构升级——反应式架构的探索与实践）

中间件全异步牵涉到到公司中台化战略或框架部门的支持，包括公司内部常用的中间件比如 MQ、redis、dal 等，超出了本文讨论的范围，感兴趣的可以看下文章末尾的参考资料。

线程模型统一的背景在上一节异步化好处时有提到过，其实主要还是对线程池的管理，做好服务隔离，线程池设置和注意事项可以参考之前的两篇文章：[Java 踩坑记系列之线程池](http://javakk.com/188.html) 、[线程池 ForkJoinPool 简介](http://javakk.com/215.html)  

这里主要说下上下文传递和阻塞检测的问题：

### 1. 上下文传递

改造成异步服务后，不能再使用`ThreadLocal`传递上下文 context，因为异步框架比如 RxJava 一般在收到通知后会先调用`observeOn()`方法切换成另外一个线程处理回调，比如我们在请求接口时在`ThreadLocal`的 context 里设置了一个值，在回调线程里从 context 里取不到这个值的，因为此时已经不是同一个`ThreadLocal`了，所以需要我们手动在切换上下文的时候传递 context 从一个线程到另一个线程环境，伪代码如下：

    Context context = ThreadLocalUtils.get(); 

复制代码

在`observeOn()`方法切换成另外一个线程后调用`doOnEvent`方法将原来的 context 赋给新的线程`ThreadLocal`  

**注意**：这里的代码只是提供一种解决思路，实际在使用前和使用后还要考虑清空`ThreadLocal`，因为线程有可能会回收到线程池下次复用，而不是立即清理，**这样就会污染上下文环境**。

可以将传递上下文的方法封装成公共方法，不需要每次都手动切换。

### 2. 阻塞检测

阻塞检测主要是要能及时发现我们某个异步任务长时间阻塞的发生，比如异步线程执行时间过长进而影响整个接口的响应，原来同步场景下我们的日志都是串行记录到 ES 或 Cat 上的，现在改成异步后，每次处理接口数据的逻辑可能在不同的线程中完成，这样记录的日志就需要我们主动去合并（依据具体的业务场景而定），如果日志无法关联起来，对我们排查问题会增加很多难度。所幸的是随着异步的流行，现在很多日志和监控系统都已支持异步了。

Project Reactor 自己也有阻塞检测功能，可以参考这篇文章：[BlockHound](http://javakk.com/redirect/aHR0cHM6Ly9naXRodWIuY29tL3JlYWN0b3IvQmxvY2tIb3VuZA==)  

### 3. 其他问题

除了上面提到的两个问题外，还有一些比如 RxJava2.0 之后不支持返回 null，如果我们原来的代码或编程习惯所致返回结果有 null 的情况，可以考虑使用 java8 的`Optional.ofNullable()`包装一下，然后返回的 RxJava 类型是这样的：`Single<Optional>`，其他异步框架如果有类似的问题同理。

## 异步其他解决方案：纤程 / 协程

-   Quasar
-   Kilim
-   Kotlin
-   Open JDK Loom
-   AJDK wisp2

协程并不是什么新技术，它在很多语言中都有实现，比如 `Python`、`Lua`、`Go` 都支持协程。

协程与线程不同之处在于，**线程由内核调度，而协程的调度是进程自身完成的**。这样就可以不受操作系统对线程数量的限制，一个线程内部可以创建成千上万个协程。因为上文讲到的异步技术都是基于线程的操作和封装，Java 中的线程概念对应的就是操作系统的线程。

### 1. Quasar、Kilim

开源的 Java 轻量级线程（协程）框架，通过利用`Java instrument`技术对字节码进行修改，使方法挂起前后可以保存和恢复 JVM 栈帧，方法内部已执行到的字节码位置也通过增加状态机的方式记录，在下次恢复执行可直接跳转至最新位置。

### 2. Kotlin

Kotlin Coroutine 协程库，因为 Kotlin 的运行依赖于 JVM，不能对 JVM 进行修改，因此 Kotlin 不能在底层支持协程。同时 Kotlin 是一门编程语言，需要在语言层面支持协程，所以 Kotlin 对协程支持最核心的部分是在编译器中完成，这一点其实和 Quasar、Kilim 实现原理类似，都是在**编译期通过修改字节码**的方式实现协程

### 3. Project Loom

Project Loom 发起的原因是因为长期以来 Java 的线程是与操作系统的线程一一对应的，这限制了 Java 平台并发能力提升，Project Loom 是**从 JVM 层面对多线程技术进行彻底的改变**。

OpenJDK 在 2018 年创建了 Loom 项目，目标是在 JVM 上实现轻量级的线程，并解除 JVM 线程与内核线程的映射。其实 Loom 项目的核心开发人员正是从 Quasar 项目过来的，目的也很明确，就是要将这项技术集成到底层 JVM 里，所以 Quasar 项目目前已经不维护了。。。

### 4. AJDK Wisp2

Alibaba Dragonwell 是阿里巴巴的 Open JDK 发行版，提供长期支持。dragonwell8 已开源协程功能（之前的版本是不支持的），开启 jvm 命令：`-XX:+UseWisp2` 即支持协程。

## 总结

-   Future 在异步方面支持有限
-   Callback 在编排能力方面有 Callback Hell 的短板
-   Project Loom 最新支持的 Open JDK 版本是 16，目前还在测试中
-   AJDK wisp2 需要换掉整个 JVM，需要考虑改动成本和收益比
