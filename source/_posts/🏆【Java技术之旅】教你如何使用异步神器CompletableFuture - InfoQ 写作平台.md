# 🏆【Java技术之旅】教你如何使用异步神器CompletableFuture - InfoQ 写作平台
### 前提概要

> 在 java8 以前，我们使用 java 的多线程编程, 一般是通过 Runnable 中的 run 方法来完成, 这种方式，有个很明显的缺点, 就是，没有返回值。这时候，大家可能会去尝试使用 Callable 中的 call 方法，然后用 Future 返回结果，如下:

    public static void main(String[] args) throws Exception {

复制代码

-   通过观察控制台, 我们发现先打印 main thread , 一秒后打印 async thread，似乎能满足我们的需求。但仔细想我们发现一个问题，当调用 future 的 get() 方法时，当前主线程是堵塞的，这好像并不是我们想看到的。
-   另一种获取返回结果的方式是先轮询, 可以调用 isDone，等完成再获取，但这也不能让我们满意.

1.  很多个异步线程执行时间可能不一致, 我的主线程业务不能一直等着, 这时候我可能会想要只等最快的线程执行完或者最重要的那个任务执行完, 亦或者我只等 1 秒钟, 至于没返回结果的线程我就用默认值代替.
2.  我两个异步任务之间执行独立, 但是第二个依赖第一个的执行结果.

java8 的 CompletableFuture, 就在这混乱且不完美的多线程江湖中闪亮登场了. CompletableFuture 让 Future 的功能和使用场景得到极大的完善和扩展, 提供了函数式编程能力, 使代码更加美观优雅, 而且可以通过回调的方式计算处理结果, 对异常处理也有了更好的处理手段.

CompletableFuture 源码中有四个静态方法用来执行异步任务:

### 创建任务

    public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier){..}

复制代码

-   如果有多线程的基础知识，我们很容易看出，run 开头的两个方法，用于执行没有返回值的任务，因为它的入参是 Runnable 对象。
-   而 supply 开头的方法显然是执行有返回值的任务了，至于方法的入参，如果没有传入 Executor 对象将会使用 ForkJoinPool.commonPool() 作为它的线程池执行异步代码. 在实际使用中, 一般我们使用自己创建的线程池对象来作为参数传入使用，这样速度会快些.

执行异步任务的方式也很简单, 只需要使用上述方法就可以了:

    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {

复制代码

接下来看一下获取执行结果的几个方法。

    V get();

复制代码

-   **上面两个方法是 Future 中的实现方式，get() 会堵塞当前的线程，这就造成了一个问题, 如果执行线程迟迟没有返回数据, get() 会一直等待下去, 因此, 第二个 get() 方法可以设置等待的时间.**
-   **getNow() 方法比较有意思，表示当有了返回结果时会返回结果，如果异步线程抛了异常会返回自己设置的默认值.**

接下来以一些场景的实例来介绍一下 CompletableFuture 中其他一些常用的方法

    thenAccept()

复制代码

-   功能：当前任务正常完成以后执行，当前任务的执行结果可以作为下一任务的输入参数，无返回值.
-   场景：执行任务 A, 同时异步执行任务 B，待任务 B 正常返回后，B 的返回值执行任务 C，任务 C 无返回值


    CompletableFuture<String> futureA = CompletableFuture.supplyAsync(() -> "任务A");

复制代码

-   功能：对不关心上一步的计算结果，执行下一个操作
-   场景：执行任务 A, 任务 A 执行完以后, 执行任务 B, 任务 B 不接受任务 A 的返回值 (不管 A 有没有返回值), 也无返回值


    CompletableFuture<String> futureA = CompletableFuture.supplyAsync(() -> "任务A");

复制代码

-   功能：当前任务正常完成以后执行，当前任务的执行的结果会作为下一任务的输入参数, 有返回值
-   场景：多个任务串联执行, 下一个任务的执行依赖上一个任务的结果, 每个任务都有输入和输出

> **异步执行任务 A，当任务 A 完成时使用 A 的返回结果 resultA 作为入参进行任务 B 的处理，可实现任意多个任务的串联执行**

    CompletableFuture<String> futureA = CompletableFuture.supplyAsync(() -> "hello");

复制代码

上面的代码，我们当然可以先调用 future.join() 先得到任务 A 的返回值, 然后再拿返回值做入参去执行任务 B, 而 thenApply 的存在就在于帮我简化了这一步, 我们不必因为等待一个计算完成而一直阻塞着调用线程，而是告诉 CompletableFuture 你啥时候执行完就啥时候进行下一步. 就把多个任务串联起来了.

    thenCombine(..)  thenAcceptBoth(..)  runAfterBoth(..)

复制代码

-   功能：结合两个 CompletionStage 的结果，进行转化后返回
-   场景：需要根据商品 id 查询商品的当前价格, 分两步, 查询商品的原始价格和折扣, 这两个查询相互独立, 当都查出来的时候用原始价格乘折扣, 算出当前价格. 使用方法: thenCombine(..)


     CompletableFuture<Double> futurePrice = CompletableFuture.supplyAsync(() -> 100d);

复制代码

-   thenCombine(..) 是结合两个任务的返回值进行转化后再返回, 那如果不需要返回呢, 那就需要
-   thenAcceptBoth(..), 同理, 如果连两个任务的返回值也不关心呢, 那就需要 runAfterBoth 了, 如果理解了上面三个方法, thenApply,thenAccept,thenRun, 这里就不需要单独再提这两个方法了, 只在这里提一下.


    thenCompose(..)

复制代码

功能：这个方法接收的输入是当前的 CompletableFuture 的计算值，返回结果将是一个新的 CompletableFuture

这个方法和 thenApply 非常像, 都是接受上一个任务的结果作为入参, 执行自己的操作, 然后返回. 那具体有什么区别呢?

-   thenApply(): 它的功能相当于将 CompletableFuture<T>转换成 CompletableFuture<U>, 改变的是同一个 CompletableFuture 中的泛型类型
-   thenCompose(): 用来连接两个 CompletableFuture，返回值是一个新的 CompletableFuture


    CompletableFuture<String> futureA = CompletableFuture.supplyAsync(() -> "hello");

复制代码

    applyToEither(..)  acceptEither(..)  runAfterEither(..)

复制代码

功能: 执行两个 CompletionStage 的结果, 那个先执行完了, 就是用哪个的返回值进行下一步操作场景: 假设查询商品 a, 有两种方式, A 和 B, 但是 A 和 B 的执行速度不一样, 我们希望哪个先返回就用那个的返回值.

    CompletableFuture<String> futureA = CompletableFuture.supplyAsync(() -> {

复制代码

> **同样的道理, applyToEither 的兄弟方法还有 acceptEither(),runAfterEither(), 我想不需要我解释你也知道该怎么用了.**

    exceptionally(..)

复制代码

-   功能: 当运行出现异常时, 调用该方法可进行一些补偿操作, 如设置默认值.
-   场景: 异步执行任务 A 获取结果, 如果任务 A 执行过程中抛出异常, 则使用默认值 100 返回.


    CompletableFuture<String> futureA = CompletableFuture.

复制代码

上面代码展示了正常流程和出现异常的情况, 可以理解成 catch, 根据返回值可以体会下.

    whenComplete(..)

复制代码

> **功能: 当 CompletableFuture 的计算结果完成，或者抛出异常的时候，都可以进入 whenComplete 方法执行, 举个栗子**

    CompletableFuture<String> futureA = CompletableFuture.

复制代码

根据控制台, 我们可以看出执行流程是这样, supplyAsync->whenComplete->exceptionally, 可以看出并没有进入 thenApply 执行, 原因也显而易见, 在 supplyAsync 中出现了异常, thenApply 只有当正常返回时才会去执行. 而 whenComplete 不管是否正常执行, 还要注意一点, whenComplete 是没有返回值的.

上面代码我们使用了函数式的编程风格并且先调用 whenComplete 再调用 exceptionally, 如果我们先调用 exceptionally, 再调用 whenComplete 会发生什么呢, 我们看一下:

复制代码 CompletableFuture<String> futureA = CompletableFuture.supplyAsync(() -> "执行结果:" + (100 / 0)).thenApply(s -> "apply result:" + s).exceptionally(e -> {System.out.println("ex:"+e.getMessage()); //ex:java.lang.ArithmeticException: / by zeroreturn "futureA result: 100";}).whenComplete((s, e) -> {if (e == null) {System.out.println(s);//futureA result: 100} else {System.out.println(e.getMessage());// 未执行}});System.out.println(futureA.join());//futureA result: 100

代码先执行了 exceptionally 后执行 whenComplete, 可以发现, 由于在 exceptionally 中对异常进行了处理, 并返回了默认值, whenComplete 中接收到的结果是一个正常的结果, 被 exceptionally 美化过的结果, 这一点需要留意一下.

    handle(..)

复制代码

功能: 当 CompletableFuture 的计算结果完成，或者抛出异常的时候，可以通过 handle 方法对结果进行处理

     CompletableFuture<String> futureA = CompletableFuture.

复制代码

通过控制台, 我们可以看出, 最后打印的是 handle result:futureA result: 100, 执行 exceptionally 后对异常进行了 "美化", 返回了默认值, 那么 handle 得到的就是一个正常的返回, 我们再试下, 先调用 handle 再调用 exceptionally 的情况.

     CompletableFuture<String> futureA = CompletableFuture.

复制代码

根据控制台输出, 可以看到先执行 handle, 打印了异常信息, 并对接过设置了默认值 500,exceptionally 并没有执行, 因为它得到的是 handle 返回给它的值, 由此我们大概推测 handle 和 whenComplete 的区别

1.  都是对结果进行处理, handle 有返回值, whenComplete 没有返回值
2.  由于 1 的存在, 使得 handle 多了一个特性, 可在 handle 里实现 exceptionally 的功能


    allOf(..)  anyOf(..)

复制代码

-   allOf: 当所有的 CompletableFuture 都执行完后执行计算
-   anyOf: 最快的那个 CompletableFuture 执行完之后执行计算

场景二: 查询一个商品详情, 需要分别去查商品信息, 卖家信息, 库存信息, 订单信息等, 这些查询相互独立, 在不同的服务上, 假设每个查询都需要一到两秒钟, 要求总体查询时间小于 2 秒.

    public static void main(String[] args) throws Exception {

复制代码

发布于: 2021 年 08 月 01 日阅读数: 956 
 [https://xie.infoq.cn/article/dc37b55efda4e2cc3b966fa38](https://xie.infoq.cn/article/dc37b55efda4e2cc3b966fa38)
