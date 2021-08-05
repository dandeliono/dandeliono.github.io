# ES 操作之批量写-BulkProcessor 原理浅析
-   [BulkProcessor](https://zhuanlan.zhihu.com/p/115096988/edit)

-   [创建流程](https://zhuanlan.zhihu.com/p/115096988/edit)

-   [内部逻辑实现]

```text
最近对线上业务进行重构，涉及到ES同步这一块，在重构过程中，为了ES 写入 性能考虑，大量的采取了 bulk的方式，来保证整体的一个同步速率，针对BulkProcessor 来深入一下，了解下 是如何实现，基于请求数，请求数据量大小 和 固定时间，刷新写入ES 的原理

针对ES 批量写入， 提供了3种方式，在 high-rest-client 中
分别是 bulk bulkAsync  bulkProcessor 3种方式。
本文主要针对 bulkProcessor 来进行一些讲述

```

## **BulkProcessor**

**官方介绍**

```text
BulkProcessor是一个线程安全的批量处理类,允许方便地设置 刷新 一个新的批量请求 
(基于数量的动作,根据大小,或时间),
容易控制并发批量的数量
请求允许并行执行。

```

### **创建流程**

**How To use ？**

**来看个 demo** `创建 BulkProcessor`

```text
@Bean(name = "bulkProcessor") // 可以封装为一个bean，非常方便其余地方来进行 写入 操作
  public BulkProcessor bulkProcessor(){

        BiConsumer<BulkRequest, ActionListener<BulkResponse>> bulkConsumer =
                (request, bulkListener) -> Es6XServiceImpl.getClient().bulkAsync(request, RequestOptions.DEFAULT, bulkListener);

        return BulkProcessor.builder(bulkConsumer, new BulkProcessor.Listener() {
            @Override
            public void beforeBulk(long executionId, BulkRequest request) {
              		// todo do something
                int i = request.numberOfActions();
                log.error("ES 同步数量{}",i);
            }

            @Override
            public void afterBulk(long executionId, BulkRequest request, BulkResponse response) {
						// todo do something
                Iterator<BulkItemResponse> iterator = response.iterator();
                while (iterator.hasNext()){
                    System.out.println(JSON.toJSONString(iterator.next()));
                }
            }

            @Override
            public void afterBulk(long executionId, BulkRequest request, Throwable failure) {
								// todo do something
                log.error("写入ES 重新消费");
            }
        }).setBulkActions(1000) //  达到刷新的条数
                .setBulkSize(new ByteSizeValue(1, ByteSizeUnit.MB)) // 达到 刷新的大小
                .setFlushInterval(TimeValue.timeValueSeconds(5)) // 固定刷新的时间频率
                .setConcurrentRequests(1) //并发线程数
                .setBackoffPolicy(BackoffPolicy.exponentialBackoff(TimeValue.timeValueMillis(100), 3)) // 重试补偿策略
                .build(); 

    }

```

`使用 BulkProcessor`

```text
bulkProcessor.add(xxxRequest)

```

**创建过程做了些什么？**

1.  创建一个 consumer 对象用来封装传递参数，和请求操作  
    BiConsumer&lt;BulkRequest, ActionListener<BulkResponse>> bulkConsumer =  
    (request, bulkListener) -> Es6XServiceImpl.getClient().bulkAsync(request, RequestOptions.DEFAULT, bulkListener);

    我们可以看到用了 java 8 的函数式编程接口 BiConsumer 关于 BiConsumer 的用法，可以自行百度，因为也是采取的 异步刷新策略， 所以，是一个返回结果的 Listener ActionListener
2.  构建并 BulkProcess  
    return BulkProcessor.builder(bulkConsumer, new BulkProcessor.Listener() {  
    \*\*\*\*

    }).setBulkActions(1000)  
    .setBulkSize(new ByteSizeValue(1, ByteSizeUnit.MB))  
    .setFlushInterval(TimeValue.timeValueSeconds(5))  
    .setConcurrentRequests(1)  
    .setBackoffPolicy(BackoffPolicy.exponentialBackoff(TimeValue.timeValueMillis(100), 3))  
    .build();

    }

    可以很清楚的看到，在 build 操作中，我们看到，在 build 中，除了 之前定义的 consumer，还实现了一个 Listener 接口 （稍后会具体讲到），用来做一些 在批量求情之前和请求之后的处理。

至此为止，BulkProcessor 创建，就 OK 啦～。

### **内部逻辑实现**

**先不说话，我们先上张类图**

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-5%2023-33-21/dec21ddc-4d0d-4ea0-9ca9-7296f534421f.jpeg?raw=true)

可以看到，在 BulkProcessor 中，有这样的一些类和接口

```text
Listener
Builder
BulkProcessor
Flush
===== 华丽的分界线
BulkProcessor 实现了 Closeable --> 继承自  AutoCloseable （关于AutoCloseable 本文不做过多说明，具体的可以百度，或者等待后续）

```

**那么先从构建开始，我们来看下 Builder**

```text
/**
* 简单的构建，可以看到，就是一个client 和 listener 这个不会做刷新策略，
*/
public static Builder builder(Client client, Listener listener) {
        Objects.requireNonNull(client, "client");
        Objects.requireNonNull(listener, "listener");
        return new Builder(client::bulk, listener, client.threadPool(), () -> {});
    }



/**
* 所有功能的builder 实现方法
* ScheduledThreadPoolExecutor 用来实现 按照时间频率，来进行 刷新，如 每5s 
*
*/
    public static Builder builder(BiConsumer<BulkRequest, ActionListener<BulkResponse>> consumer, Listener listener) {
        Objects.requireNonNull(consumer, "consumer");
        Objects.requireNonNull(listener, "listener");
        final ScheduledThreadPoolExecutor scheduledThreadPoolExecutor = Scheduler.initScheduler(Settings.EMPTY); // 接口静态方式，来实现 Executor 的初始化
        return new Builder(consumer, listener,
                (delay, executor, command) -> scheduledThreadPoolExecutor.schedule(command, delay.millis(), TimeUnit.MILLISECONDS), //
                () -> Scheduler.terminate(scheduledThreadPoolExecutor, 10, TimeUnit.SECONDS));
    }


/**
* 构造函数 
* @param consumer 前文定义的consumer request response action
* @param listener listener  BulkProcessor 内置监听器
* @param scheduler elastic 定时调度 类scheduler  
* @paran onClose 关闭时候的运行
*/

    private Builder(BiConsumer<BulkRequest, ActionListener<BulkResponse>> consumer, Listener listener,
                        Scheduler scheduler, Runnable onClose) {
            this.consumer = consumer;
            this.listener = listener;
            this.scheduler = scheduler;
            this.onClose = onClose;
        }

```

通过上述的代码片段，可以很明显的看到，关于初始化构建的一些关键点和要素

**看完 builder 接下来，我们看下 bulkprocessor 是如何工作的**

_先看下构造方法_

```text
   BulkProcessor(BiConsumer<BulkRequest, ActionListener<BulkResponse>> consumer, BackoffPolicy backoffPolicy, Listener listener,
                  int concurrentRequests, int bulkActions, ByteSizeValue bulkSize, @Nullable TimeValue flushInterval,
                  Scheduler scheduler, Runnable onClose) {
        this.bulkActions = bulkActions;
        this.bulkSize = bulkSize.getBytes();
        this.bulkRequest = new BulkRequest();
        this.scheduler = scheduler;
        this.bulkRequestHandler = new BulkRequestHandler(consumer, backoffPolicy, listener, scheduler, concurrentRequests); // BulkRequestHandler 批量执行 handler 操作
        // Start period flushing task after everything is setup
        this.cancellableFlushTask = startFlushTask(flushInterval, scheduler); //开始刷新任务
        this.onClose = onClose;
    }

```

**startFlushTask 如何进行工作**

```text
private Scheduler.Cancellable startFlushTask(TimeValue flushInterval, Scheduler scheduler) {
   
   // 如果 按照时间刷新 为空，则直接返回 任务为取消状态
   if (flushInterval == null) {
            return new Scheduler.Cancellable() {
                @Override
                public void cancel() {}

                @Override
                public boolean isCancelled() {
                    return true;
                }
            };
        }
        final Runnable flushRunnable = scheduler.preserveContext(new Flush());
        return scheduler.scheduleWithFixedDelay(flushRunnable, flushInterval, ThreadPool.Names.GENERIC);
    }

    private void executeIfNeeded() {
        ensureOpen();
        if (!isOverTheLimit()) {
            return;
        }
        execute();
    }


// 刷新线程
class Flush implements Runnable {

        @Override
        public void run() {
            synchronized (BulkProcessor.this) {
                if (closed) {
                    return;
                }
                if (bulkRequest.numberOfActions() == 0) {
                    return;
                }
                execute(); // 下面方法
            }
        }
    }



/**
*  刷新执行
*
*/

 // (currently) needs to be executed under a lock
    private void execute() {
        final BulkRequest bulkRequest = this.bulkRequest;
        final long executionId = executionIdGen.incrementAndGet();
				// 刷新 bulkRequest 为下一批做准备
        this.bulkRequest = new BulkRequest();
        this.bulkRequestHandler.execute(bulkRequest, executionId);
    }

```

**看到这里，关于时间的定时调度，我们其实是很清楚了，那么 关于数据量 和 大小的判断策略在哪儿？**

```text
/**
* 各种添加操作
*/
public BulkProcessor add(DocWriteRequest request, @Nullable Object payload) {
        internalAdd(request, payload);
        return this;
    }

	/**
	* 我们可以看到，在添加之后，会做一个操作
	* executeIfNeeded 如果需要，则进行执行
	*/
    private synchronized void internalAdd(DocWriteRequest request, @Nullable Object payload) {
        ensureOpen();
        bulkRequest.add(request, payload);
        executeIfNeeded();
    }


/**
* 如果超过限制，则执行刷新操作
*/
 private void executeIfNeeded() {
        ensureOpen();
        if (!isOverTheLimit()) {
            return;
        }
        execute();
    }


	/**
	* 这这儿，我们终于看到了 关于action 和 size 的判断操作，
	*
	*/
    private boolean isOverTheLimit() {
        if (bulkActions != -1 && bulkRequest.numberOfActions() >= bulkActions) {
            return true;
        }
        if (bulkSize != -1 && bulkRequest.estimatedSizeInBytes() >= bulkSize) {
            return true;
        }
        return false;
    }

```

**`通过上述的分析，关于按照时间，数据 size，大小来进行 flush 执行的入口我们都已经很清楚了`**

```text
针对数据大小的设置。在每次添加的时候，做判断是否 超过限制
针对 时间的频次控制，交由ScheduledThreadPoolExecutor 来去做监控

```

**下来，让我们看下具体的执行以及重试策略，和 返回值的处理**

```text
public void execute(BulkRequest bulkRequest, long executionId) {
        Runnable toRelease = () -> {};
        boolean bulkRequestSetupSuccessful = false;
        try {
          // listener 填充 request 和执行ID
            listener.beforeBulk(executionId, bulkRequest);
          	//通过信号量来进行资源的控制 来自于我们设置的 setConcurrentRequests
            semaphore.acquire();
            toRelease = semaphore::release;
            CountDownLatch latch = new CountDownLatch(1);
          // 进行执行并按照补偿重试策略如果失败
            retry.withBackoff(consumer, bulkRequest, new ActionListener<BulkResponse>() {
						 	//结果写入  ActionListener --> BulkProcessor.Listener 的转换
              @Override
                public void onResponse(BulkResponse response) {
                    try {
                        listener.afterBulk(executionId, bulkRequest, response);
                    } finally {
                        semaphore.release();
                        latch.countDown();
                    }
                }

                @Override
                public void onFailure(Exception e) {
                    try {
                        listener.afterBulk(executionId, bulkRequest, e);
                    } finally {
                        semaphore.release();
                        latch.countDown();
                    }
                }
            }, Settings.EMPTY);
            bulkRequestSetupSuccessful = true;
            if (concurrentRequests == 0) {
                latch.await();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            logger.info(() -> new ParameterizedMessage("Bulk request {} has been cancelled.", executionId), e);
            listener.afterBulk(executionId, bulkRequest, e);
        } catch (Exception e) {
            logger.warn(() -> new ParameterizedMessage("Failed to execute bulk request {}.", executionId), e);
            listener.afterBulk(executionId, bulkRequest, e);
        } finally {
            if (bulkRequestSetupSuccessful == false) {  // if we fail on client.bulk() release the semaphore
                toRelease.run();
            }
        }
    }

```

**最终的执行，在 RetryHandler 中，继续往下看**

```text
public void execute(BulkRequest bulkRequest) {
            this.currentBulkRequest = bulkRequest;
            consumer.accept(bulkRequest, this);
        }

```

对，没错，只有一个操作， consumer.accept(bulkRequest, this);

再一次展现了 java 8 函数式接口的功能强大之处 此处 consumer.accept(bulkRequest, this);

执行的操作即 Es6XServiceImpl.getClient().bulkAsync(request, RequestOptions.DEFAULT, bulkListener);

**如何设置重试策略，以及数据的筛选**

```text
@Override
        public void onResponse(BulkResponse bulkItemResponses) {
            if (!bulkItemResponses.hasFailures()) {
                // we're done here, include all responses
                addResponses(bulkItemResponses, (r -> true));
                finishHim();
            } else {
                if (canRetry(bulkItemResponses)) {
                    addResponses(bulkItemResponses, (r -> !r.isFailed()));
                    retry(createBulkRequestForRetry(bulkItemResponses));
                } else {
                    addResponses(bulkItemResponses, (r -> true));
                    finishHim();
                }
            }
        }


/**
* 只针对失败的请求，放入到重试策略中
*/
   private void addResponses(BulkResponse response, Predicate<BulkItemResponse> filter) {
            for (BulkItemResponse bulkItemResponse : response) {
                if (filter.test(bulkItemResponse)) {
                    // Use client-side lock here to avoid visibility issues. This method may be called multiple times
                    // (based on how many retries we have to issue) and relying that the response handling code will be
                    // scheduled on the same thread is fragile.
                    synchronized (responses) {
                        responses.add(bulkItemResponse);
                    }
                }
            }
        }

```

再一次展示了函数式接口的强大之处

**补充一张流程图**

![](https://pic4.zhimg.com/v2-2deb24392f9ed03d6a0d2f60b906559b_b.jpg)
