# ThreadLocal系列（三）-TransmittableThreadLocal的使用及原理解析
## **一、基本使用**

首先，TTL 是用来解决 ITL 解决不了的问题而诞生的，所以 TTL 一定是支持父线程的本地变量传递给子线程这种基本操作的，ITL 也可以做到，但是前面有讲过，ITL 在线程池的模式下，就没办法再正确传递了，所以 TTL 做出的改进就是即便是在线程池模式下，也可以很好的将父线程本地变量传递下去，先来看个例子：

```null


    private static ExecutorService executorService = TtlExecutors.getTtlExecutorService(Executors.newFixedThreadPool(2));

    private static ThreadLocal tl = new TransmittableThreadLocal<>(); 

    public static void main(String[] args) {

        new Thread(() -> {

            String mainThreadName = "main_01";

            tl.set(1);

            executorService.execute(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之前(1), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            });

            executorService.execute(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之前(1), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            });

            executorService.execute(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之前(1), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            });

            sleep(1L); 
            tl.set(2); 

            executorService.execute(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之后(2), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            });

            executorService.execute(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之后(2), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            });

            executorService.execute(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之后(2), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            });

            System.out.println(String.format("线程名称-%s, 变量值=%s", Thread.currentThread().getName(), tl.get()));

        }).start();


        new Thread(() -> {

            String mainThreadName = "main_02";

            tl.set(3);

            executorService.execute(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之前(3), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            });

            executorService.execute(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之前(3), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            });

            executorService.execute(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之前(3), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            });

            sleep(1L); 
            tl.set(4); 

            executorService.execute(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之后(4), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            });

            executorService.execute(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之后(4), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            });

            executorService.execute(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之后(4), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            });

            System.out.println(String.format("线程名称-%s, 变量值=%s", Thread.currentThread().getName(), tl.get()));

        }).start();

    }

    private static void sleep(long time) {
        try {
            Thread.sleep(time);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

运行结果：

```null

线程名称-Thread-2, 变量值=4
本地变量改变之前(3), 父线程名称-main_02, 子线程名称-pool-1-thread-1, 变量值=3
线程名称-Thread-1, 变量值=2
本地变量改变之前(1), 父线程名称-main_01, 子线程名称-pool-1-thread-2, 变量值=1
本地变量改变之前(1), 父线程名称-main_01, 子线程名称-pool-1-thread-1, 变量值=1
本地变量改变之前(3), 父线程名称-main_02, 子线程名称-pool-1-thread-2, 变量值=3
本地变量改变之前(3), 父线程名称-main_02, 子线程名称-pool-1-thread-2, 变量值=3
本地变量改变之前(1), 父线程名称-main_01, 子线程名称-pool-1-thread-1, 变量值=1
本地变量改变之后(2), 父线程名称-main_01, 子线程名称-pool-1-thread-2, 变量值=2
本地变量改变之后(4), 父线程名称-main_02, 子线程名称-pool-1-thread-1, 变量值=4
本地变量改变之后(4), 父线程名称-main_02, 子线程名称-pool-1-thread-1, 变量值=4
本地变量改变之后(4), 父线程名称-main_02, 子线程名称-pool-1-thread-2, 变量值=4
本地变量改变之后(2), 父线程名称-main_01, 子线程名称-pool-1-thread-1, 变量值=2
本地变量改变之后(2), 父线程名称-main_01, 子线程名称-pool-1-thread-2, 变量值=2
```

程序有些啰嗦，为了说明问题，加了很多说明，但至少通过上面的例子，不难发现，两个主线程里都使用线程池异步，而且值在主线程里还发生过改变，测试结果展示一切正常，由此可以知道 TTL 在使用线程池的情况下，也可以很好的完成传递，而且不会发生错乱。

那么是不是对普通线程异步也有这么好的支撑呢？

改造下上面的测试代码：

```null

private static ThreadLocal tl = new TransmittableThreadLocal<>();

    public static void main(String[] args) {

        new Thread(() -> {

            String mainThreadName = "main_01";

            tl.set(1);

            new Thread(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之前(1), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            }).start();

            new Thread(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之前(1), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            }).start();

            new Thread(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之前(1), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            }).start();

            sleep(1L); 
            tl.set(2); 

            new Thread(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之后(2), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            }).start();

            new Thread(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之后(2), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            }).start();

            new Thread(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之后(2), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            }).start();

            System.out.println(String.format("线程名称-%s, 变量值=%s", Thread.currentThread().getName(), tl.get()));

        }).start();


        new Thread(() -> {

            String mainThreadName = "main_02";

            tl.set(3);

            new Thread(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之前(3), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            }).start();

            new Thread(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之前(3), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            }).start();

            new Thread(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之前(3), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            }).start();

            sleep(1L); 
            tl.set(4); 

            new Thread(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之后(4), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            }).start();

            new Thread(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之后(4), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            }).start();

            new Thread(() -> {
                sleep(1L);
                System.out.println(String.format("本地变量改变之后(4), 父线程名称-%s, 子线程名称-%s, 变量值=%s", mainThreadName, Thread.currentThread().getName(), tl.get()));
            }).start();

            System.out.println(String.format("线程名称-%s, 变量值=%s", Thread.currentThread().getName(), tl.get()));

        }).start();

    }
```

相比第一段测试代码，这一段的异步全都是普通异步，未采用线程池的方式进行异步，看下运行结果：

```null

本地变量改变之后(4), 父线程名称-main_02, 子线程名称-Thread-14, 变量值=4
本地变量改变之前(1), 父线程名称-main_01, 子线程名称-Thread-5, 变量值=1
线程名称-Thread-1, 变量值=2
本地变量改变之前(1), 父线程名称-main_01, 子线程名称-Thread-3, 变量值=1
本地变量改变之后(2), 父线程名称-main_01, 子线程名称-Thread-11, 变量值=2
本地变量改变之前(3), 父线程名称-main_02, 子线程名称-Thread-6, 变量值=3
本地变量改变之后(4), 父线程名称-main_02, 子线程名称-Thread-12, 变量值=4
本地变量改变之后(4), 父线程名称-main_02, 子线程名称-Thread-10, 变量值=4
本地变量改变之前(3), 父线程名称-main_02, 子线程名称-Thread-8, 变量值=3
本地变量改变之前(3), 父线程名称-main_02, 子线程名称-Thread-4, 变量值=3
本地变量改变之前(1), 父线程名称-main_01, 子线程名称-Thread-7, 变量值=1
线程名称-Thread-2, 变量值=4
本地变量改变之后(2), 父线程名称-main_01, 子线程名称-Thread-9, 变量值=2
本地变量改变之后(2), 父线程名称-main_01, 子线程名称-Thread-13, 变量值=2
```

ok，可以看到，达到了跟第一个测试一致的结果。

到这里，通过上述两个例子，TTL 的基本使用，以及其解决的问题，我们已经有了初步的了解，下面我们来解析一下其内部原理，看看 TTL 是怎么完成对 ITL 的优化的。

## **二、原理分析**

先来看 TTL 里面的几个重要属性及方法

TTL 定义：

```null

public class TransmittableThreadLocal extends InheritableThreadLocal
```

可以看到，TTL 继承了 ITL，意味着 TTL 首先具备 ITL 的功能。

再来看看一个重要属性 holder：

```null

   
    private static InheritableThreadLocal<Map<TransmittableThreadLocal<?>, ?>> holder =
            new InheritableThreadLocal<Map<TransmittableThreadLocal<?>, ?>>() {
                @Override
                protected Map<TransmittableThreadLocal<?>, ?> initialValue() {
                    return new WeakHashMap<TransmittableThreadLocal<?>, Object>();
                }

                @Override
                protected Map<TransmittableThreadLocal<?>, ?> childValue(Map<TransmittableThreadLocal<?>, ?> parentValue) {
                    return new WeakHashMap<TransmittableThreadLocal<?>, Object>(parentValue);
                }
            };
```

再来看下 set 和 get：

```null

@Override
    public final void set(T value) {
        super.set(value);
        if (null == value) removeValue();
        else addValue();
    }

    @Override
    public final T get() {
        T value = super.get();
        if (null != value) addValue();
        return value;
    }
    
    private void removeValue() {
        holder.get().remove(this); 
    }

    private void addValue() {
        if (!holder.get().containsKey(this)) {
            holder.get().put(this, null); 
        }
    }
```

TTL 里先了解上述的几个方法及对象，可以看出，单纯的使用 TTL 是达不到支持线程池本地变量的传递的，通过第一部分的例子，可以发现，除了要启用 TTL，还需要通过**TtlExecutors.getTtlExecutorService**包装一下线程池才可以，那么，下面就来看看在程序即将通过线程池异步的时候，TTL 帮我们做了哪些操作（这一部分是 TTL 支持线程池传递的核心部分）：

首先打开包装类，看下 execute 方法在执行时做了些什么。

```null

@Override
    public void execute(@Nonnull Runnable command) {
        executor.execute(TtlRunnable.get(command)); 
    }

    
    @Nullable
    public static TtlRunnable get(@Nullable Runnable runnable) {
        return get(runnable, false, false);
    }

    
    @Nullable
    public static TtlRunnable get(@Nullable Runnable runnable, boolean releaseTtlValueReferenceAfterRun, boolean idempotent) {
        if (null == runnable) return null;

        if (runnable instanceof TtlEnhanced) { 
            
            if (idempotent) return (TtlRunnable) runnable;
            else throw new IllegalStateException("Already TtlRunnable!");
        }
        return new TtlRunnable(runnable, releaseTtlValueReferenceAfterRun); 
    }

    
    private TtlRunnable(@Nonnull Runnable runnable, boolean releaseTtlValueReferenceAfterRun) {
        this.capturedRef = new AtomicReference<Object>(capture()); 
        this.runnable = runnable;
        this.releaseTtlValueReferenceAfterRun = releaseTtlValueReferenceAfterRun;
    }

    
    @Nonnull
    public static Object capture() {
        Map<TransmittableThreadLocal<?>, Object> captured = new HashMap<TransmittableThreadLocal<?>, Object>();
        for (TransmittableThreadLocal<?> threadLocal : holder.get().keySet()) { 
            captured.put(threadLocal, threadLocal.copyValue());
        }
        return captured; 
    }

    
    private T copyValue() {
        return copy(get());
    }
    protected T copy(T parentValue) {
        return parentValue;
    }
```

结合上述代码，大致知道了在线程池异步之前需要做的事情，其实就是把当前父线程里的本地变量取出来，然后赋值给 Rannable 包装类里的**capturedRef**属性，到此为止，下面会发生什么，我们大致上可以猜出来了，接下来大概率会在 run 方法里，将这些捕获到的值赋给子线程的 holder 赋对应的 TTL 值，那么我们继续往下看 Rannable 包装类里的 run 方法是怎么实现的：

```null

@Override
    public void run() {
        Object captured = capturedRef.get(); 
        if (captured == null || releaseTtlValueReferenceAfterRun && !capturedRef.compareAndSet(captured, null)) {
            throw new IllegalStateException("TTL value reference is released after run!");
        }

        
        Object backup = replay(captured);
        try {
            runnable.run(); 
        } finally {
            restore(backup); 
        }
    }
```

根据上述代码，我们看到了 TTL 在异步任务执行前，会先进行赋值操作（就是拿着异步发生时捕获到的父线程的本地变量，赋给自己），当任务执行完，就恢复原生的自己本身的线程变量值。

下面来具体看这俩方法：

```null

@Nonnull
    public static Object replay(@Nonnull Object captured) {
        @SuppressWarnings("unchecked")
        Map<TransmittableThreadLocal<?>, Object> capturedMap = (Map<TransmittableThreadLocal<?>, Object>) captured; 
        Map<TransmittableThreadLocal<?>, Object> backup = new HashMap<TransmittableThreadLocal<?>, Object>(); 

        
        for (Iterator<? extends Map.Entry<TransmittableThreadLocal<?>, ?>> iterator = holder.get().entrySet().iterator();
             iterator.hasNext(); ) {
            Map.Entry<TransmittableThreadLocal<?>, ?> next = iterator.next();
            TransmittableThreadLocal<?> threadLocal = next.getKey();

            backup.put(threadLocal, threadLocal.get()); 

            
            if (!capturedMap.containsKey(threadLocal)) {
                iterator.remove();
                threadLocal.superRemove();
            }
        }

        
        setTtlValuesTo(capturedMap);

        
        doExecuteCallback(true);

        return backup;
    }

    
    public static void restore(@Nonnull Object backup) {
        @SuppressWarnings("unchecked")
        Map<TransmittableThreadLocal<?>, Object> backupMap = (Map<TransmittableThreadLocal<?>, Object>) backup;
        
        doExecuteCallback(false);

        
        for (Iterator<? extends Map.Entry<TransmittableThreadLocal<?>, ?>> iterator = holder.get().entrySet().iterator();
             iterator.hasNext(); ) {
            Map.Entry<TransmittableThreadLocal<?>, ?> next = iterator.next();
            TransmittableThreadLocal<?> threadLocal = next.getKey();

            
            if (!backupMap.containsKey(threadLocal)) {
                iterator.remove();
                threadLocal.superRemove();
            }
        }

        
        setTtlValuesTo(backupMap);
    }

    
    private static void setTtlValuesTo(@Nonnull Map<TransmittableThreadLocal<?>, Object> ttlValues) {
        for (Map.Entry<TransmittableThreadLocal<?>, Object> entry : ttlValues.entrySet()) {
            @SuppressWarnings("unchecked")
            TransmittableThreadLocal<Object> threadLocal = (TransmittableThreadLocal<Object>) entry.getKey();
            threadLocal.set(entry.getValue()); 
        }
    }
```

ok，到这里基本上把 TTL 比较核心的代码看完了，下面整理下整个流程，这是官方给出的时序图：

![](https://img2018.cnblogs.com/blog/1569484/201902/1569484-20190226141722922-1064075768.png)

**图 2-1**

上图第一行指的是类名称，下面的流程指的是类所做的事情，根据上面罗列出来的源码，结合这个时序图，可以比较直观一些的理解整个流程。

## **三、TTL 中线程池子线程原生变量的产生**

这一节是为了验证上面 replay 和 restore，现在通过一个例子来验证下，先把源码 down 下来，在源码的 restore 和 replay 上分别加上输出语句，遍历 holder：

```null


public static Object replay(@Nonnull Object captured) {
            @SuppressWarnings("unchecked")
            Map<TransmittableThreadLocal<?>, Object> capturedMap = (Map<TransmittableThreadLocal<?>, Object>) captured;
            Map<TransmittableThreadLocal<?>, Object> backup = new HashMap<TransmittableThreadLocal<?>, Object>();
            System.out.println("--------------------replay前置，当前拿到的holder里的TTL列表");
            for (Iterator<? extends Map.Entry<TransmittableThreadLocal<?>, ?>> iterator = holder.get().entrySet().iterator();
                 iterator.hasNext(); ) {
                Map.Entry<TransmittableThreadLocal<?>, ?> next = iterator.next();
                TransmittableThreadLocal<?> threadLocal = next.getKey();
                System.out.println(String.format("replay前置里拿到原生的ttl_k=%s, ttl_value=%s", threadLocal.hashCode(), threadLocal.get()));
            }

            for...
            
            setTtlValuesTo(capturedMap);

            doExecuteCallback(true);

            System.out.println("--------------------reply后置，当前拿到的holder里的TTL列表");
            for (Iterator<? extends Map.Entry<TransmittableThreadLocal<?>, ?>> iterator = holder.get().entrySet().iterator();
                 iterator.hasNext(); ) {
                Map.Entry<TransmittableThreadLocal<?>, ?> next = iterator.next();
                TransmittableThreadLocal<?> threadLocal = next.getKey();
                System.out.println(String.format("replay后置里拿到原生的ttl_k=%s, ttl_value=%s", threadLocal.hashCode(), threadLocal.get()));
            }

            return backup;
        }


public static void restore(@Nonnull Object backup) {
            @SuppressWarnings("unchecked")
            Map<TransmittableThreadLocal<?>, Object> backupMap = (Map<TransmittableThreadLocal<?>, Object>) backup;
            
            doExecuteCallback(false);

            System.out.println("--------------------restore前置，当前拿到的holder里的TTL列表");
            for (Iterator<? extends Map.Entry<TransmittableThreadLocal<?>, ?>> iterator = holder.get().entrySet().iterator();
                 iterator.hasNext(); ) {
                Map.Entry<TransmittableThreadLocal<?>, ?> next = iterator.next();
                TransmittableThreadLocal<?> threadLocal = next.getKey();
                System.out.println(String.format("restore前置里拿到当前线程内变量，ttl_k=%s, ttl_value=%s", threadLocal.hashCode(), threadLocal.get()));
            }

            for...

            setTtlValuesTo(backupMap);

            System.out.println("--------------------restore后置，当前拿到的holder里的TTL列表");
            for (Iterator<? extends Map.Entry<TransmittableThreadLocal<?>, ?>> iterator = holder.get().entrySet().iterator();
                 iterator.hasNext(); ) {
                Map.Entry<TransmittableThreadLocal<?>, ?> next = iterator.next();
                TransmittableThreadLocal<?> threadLocal = next.getKey();
                System.out.println(String.format("restore后置里拿到当前线程内变量，ttl_k=%s, ttl_value=%s", threadLocal.hashCode(), threadLocal.get()));
            }
        }
```

代码这样做的目的，就是要说明线程池所谓的原生本地变量是怎么产生的，以及 replay 和 restore 是怎么设置和恢复的，下面来看个简单的例子：

```null

private static ExecutorService executorService = TtlExecutors.getTtlExecutorService(Executors.newFixedThreadPool(1));

    private static ThreadLocal tl = new TransmittableThreadLocal();
    private static ThreadLocal tl2 = new TransmittableThreadLocal();

    public static void main(String[] args) throws InterruptedException {

        tl.set(1);
        tl2.set(2);

        executorService.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000L);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
    }
```

运行结果如下：

```null

--------------------replay前置，当前拿到的holder里的TTL列表
replay前置里拿到原生的ttl_k=1259475182, ttl_value=2
replay前置里拿到原生的ttl_k=929338653, ttl_value=1
--------------------reply后置，当前拿到的holder里的TTL列表
replay后置里拿到原生的ttl_k=1259475182, ttl_value=2
replay后置里拿到原生的ttl_k=929338653, ttl_value=1
--------------------restore前置，当前拿到的holder里的TTL列表
restore前置里拿到当前线程内变量，ttl_k=1259475182, ttl_value=2
restore前置里拿到当前线程内变量，ttl_k=929338653, ttl_value=1
--------------------restore后置，当前拿到的holder里的TTL列表
restore后置里拿到当前线程内变量，ttl_k=1259475182, ttl_value=2
restore后置里拿到当前线程内变量，ttl_k=929338653, ttl_value=1
```

我们会发现，原生值产生了，从异步开始，就确定了线程池里的线程具备了 1 和 2 的值，那么，再来改动下上面的测试代码：

```null

public static void main(String[] args) throws InterruptedException {

        tl.set(1);

        executorService.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(100L);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        Thread.sleep(1000L);

        tl2.set(2);

        System.out.println("---------------------------------------------------------------------------------");
        executorService.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000L);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
    }
```

运行结果为：

```null

--------------------replay前置，当前拿到的holder里的TTL列表
replay前置里拿到原生的ttl_k=929338653, ttl_value=1
--------------------reply后置，当前拿到的holder里的TTL列表
replay后置里拿到原生的ttl_k=929338653, ttl_value=1
--------------------restore前置，当前拿到的holder里的TTL列表
restore前置里拿到当前线程内变量，ttl_k=929338653, ttl_value=1
--------------------restore后置，当前拿到的holder里的TTL列表
restore后置里拿到当前线程内变量，ttl_k=929338653, ttl_value=1
---------------------------------------------------------------------------------
--------------------replay前置，当前拿到的holder里的TTL列表
replay前置里拿到原生的ttl_k=929338653, ttl_value=1
--------------------reply后置，当前拿到的holder里的TTL列表
replay后置里拿到原生的ttl_k=1020371697, ttl_value=2
replay后置里拿到原生的ttl_k=929338653, ttl_value=1
--------------------restore前置，当前拿到的holder里的TTL列表
restore前置里拿到当前线程内变量，ttl_k=1020371697, ttl_value=2
restore前置里拿到当前线程内变量，ttl_k=929338653, ttl_value=1
--------------------restore后置，当前拿到的holder里的TTL列表
restore后置里拿到当前线程内变量，ttl_k=929338653, ttl_value=1
```

可以发现，第一次异步时，只有一个值被传递了下去，然后第二次异步，新加了一个 tl2 的值，但是看第二次异步的打印，会发现，restore 恢复后，仍然是第一次异步发生时放进去的那个 tl 的值。

通过上面的例子，基本可以确认，所谓线程池内线程的本地原生变量，其实是第一次使用线程时被传递进去的值，我们之前有说过 TTL 是继承至 ITL 的，之前的文章也说过，线程池第一次启用时是会触发 Thread 的 init 方法的，也就是说，在第一次异步时拿到的主线程的变量会被传递给子线程，作为子线程的原生本地变量保存起来，后续是**replay**操作和**restore**操作也是围绕着这个原生变量（即原生 holder 里的值）来进行**设置**、**恢复**的，设置的是当前父线程捕获到的本地变量，恢复的是子线程原生本地变量。

holder 里持有的可以理解就是当前线程内的**所有本地变量**，当子线程将异步任务**执行完毕后**，**会执行 restore 进行恢复原生本地变量**，具体参照上面的代码和测试代码。

## 四、总结

到这里基本上确认了 TTL 是如何进行线程池传值的，以及被包装的 run 方法执行异步任务之前，会使用**replay**进行设置父线程里的本地变量给当前子线程，任务执行完毕，会调用**restore**恢复该子线程原生的本地变量（目前原生本地变量的产生，就只碰到上述测试代码中的这一种情况，即线程第一次使用时通过 ITL 属性以及 Thread 的 init 方法传给子线程，还不太清楚有没有其他方式设置）。

其实，正常程序里想要完成线程池上下文传递，使用 TL 就足够了，我们可以效仿 TTL 包装线程池对象的原理，进行值传递，异步任务结束后，再 remove，以此类推来完成线程池值传递，不过这种方式过于单纯，且要求上下文为只读对象，否则子线程存在写操作，就会发生上下文污染。

TTL 项目地址（可以详细了解下它的其他特性和用法）：[https://github.com/alibaba/transmittable-thread-local](https://github.com/alibaba/transmittable-thread-local) 
 [https://www.cnblogs.com/hama1993/p/10409740.html](https://www.cnblogs.com/hama1993/p/10409740.html)
