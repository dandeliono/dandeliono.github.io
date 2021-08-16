# ThreadLocal系列（二）-InheritableThreadLocal的使用及原理解析 
## 一、基本使用

我们继续来看之前写的例子：

```null


private static ThreadLocal tl = new ThreadLocal<>();

public static void main(String[] args) throws Exception {
        tl.set(1);
        System.out.println(String.format("当前线程名称: %s, main方法内获取线程内数据为: %s",
                Thread.currentThread().getName(), tl.get()));
        fc();
        new Thread(() -> {
            fc();
        }).start();
        Thread.sleep(1000L); 
        fc(); 
    }

    private static void fc() {
        System.out.println(String.format("当前线程名称: %s, fc方法内获取线程内数据为: %s",
                Thread.currentThread().getName(), tl.get()));
    }
```

输出为：

```null

当前线程名称: main, main方法内获取线程内数据为: 1
当前线程名称: main, fc方法内获取线程内数据为: 1
当前线程名称: Thread-0, fc方法内获取线程内数据为: null
当前线程名称: main, fc方法内获取线程内数据为: 1
```

我们会发现，父线程的本地变量是无法传递给子线程的，这当然是正常的，因为线程本地变量来就不应该相互有交集，但是有些时候，我们的确是需要子线程里仍然可以获取到父线程里的本地变量，现在就需要借助 TL 的一个子类：InheritableThreadLocal（下面简称 ITL），来完成上述要求 现在我们将例子里的

```null

private static ThreadLocal tl = new ThreadLocal<>();
```

改为：

```null

private static ThreadLocal tl = new InheritableThreadLocal<>();
```

然后我们再来运行下结果：

```null

当前线程名称: main, main方法内获取线程内数据为: 1
当前线程名称: main, fc方法内获取线程内数据为: 1
当前线程名称: Thread-0, fc方法内获取线程内数据为: 1
当前线程名称: main, fc方法内获取线程内数据为: 1
```

可以发现，子线程里已经可以获得父线程里的本地变量了。

结合之前讲的 TL 的实现，简单理解起来并不难，基本可以认定，是在创建子线程的时候，父线程的 ThreadLocalMap（下面简称 TLMap）里的值递给了子线程，子线程针对上述 tl 对象持有的 k-v 进行了 copy，其实这里不是真正意义上对象 copy，只是给 v 的值多了一条子线程 TLMap 的引用而已，v 的值在父子线程里指向的均是同一个对象，因此任意线程改了这个值，对其他线程是可见的，为了验证这一点，我们可以改造以上测试代码：

```null

private static ThreadLocal tl = new InheritableThreadLocal<>();
    private static ThreadLocal tl2 = new InheritableThreadLocal<>();

    public static void main(String[] args) throws Exception {
        tl.set(1);

        Hello hello = new Hello();
        hello.setName("init");
        tl2.set(hello);
        System.out.println(String.format("当前线程名称: %s, main方法内获取线程内数据为: tl = %s，tl2.name = %s",
                Thread.currentThread().getName(), tl.get(), tl2.get().getName()));
        fc();
        new Thread(() -> {
            Hello hello1 = tl2.get();
            hello1.setName("init2");
            fc();
        }).start();
        Thread.sleep(1000L); 
        fc(); 
    }

    private static void fc() {
        System.out.println(String.format("当前线程名称: %s, fc方法内获取线程内数据为: tl = %s，tl2.name = %s",
                Thread.currentThread().getName(), tl.get(), tl2.get().getName()));
    }
```

输出结果为：

```null

当前线程名称: main, main方法内获取线程内数据为: tl = 1，tl2.name = init
当前线程名称: main, fc方法内获取线程内数据为: tl = 1，tl2.name = init
当前线程名称: Thread-0, fc方法内获取线程内数据为: tl = 1，tl2.name = init2
当前线程名称: main, fc方法内获取线程内数据为: tl = 1，tl2.name = init2
```

可以确认，子线程里持有的本地变量跟父线程里那个是同一个对象。

## 二、原理分析

通过上述的测试代码，基本可以确定父线程的 TLMap 被传递到了下一级，那么我们基本可以确认 ITL 是 TL 派生出来专门解决线程本地变量父传子问题的，那么下面通过源码来分析一下 ITL 到底是怎么完成这个操作的。

先来了解下 Thread 类，上节说到，其实最终线程本地变量是通过 TLMap 存储在 Thread 对象内的，那么来看下 Thread 对象内关于 TLMap 的两个属性：

```null

 ThreadLocal.ThreadLocalMap threadLocals = null;
 ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

Thread 类里其实有两个 TLMap 属性，第一个就是普通 TL 对象为其赋值，第二个则由 ITL 对象为其赋值，来看下 TL 的 set 方法的实现，这次针对该方法介绍下 TL 子类的相关方法实现：

```null


    
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocal.ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    
    
    ThreadLocal.ThreadLocalMap getMap(Thread t) {
        return t.inheritableThreadLocals; 
    }
    
    
    void createMap(Thread t, T firstValue) {
        
        t.inheritableThreadLocals = new ThreadLocal.ThreadLocalMap(this, firstValue);
    }
```

而 inheritableThreadLocals 里的信息通过 Thread 的 init 方法是可以被传递下去的：

````null


    Thread parent = currentThread();
    if (parent.inheritableThreadLocals != null){ 
        
        this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    }

    
    static ThreadLocal.ThreadLocalMap createInheritedMap(ThreadLocal.ThreadLocalMap parentMap) {
        return new ThreadLocal.ThreadLocalMap(parentMap); 
    }

    
    private ThreadLocalMap(ThreadLocal.ThreadLocalMap parentMap) {
        ThreadLocal.ThreadLocalMap.Entry[] parentTable = parentMap.table; 
        int len = parentTable.length;
        setThreshold(len); 
        table = new ThreadLocal.ThreadLocalMap.Entry[len];

        for (int j = 0; j < len; j++) {
            ThreadLocal.ThreadLocalMap.Entry e = parentTable[j]; 
            if (e != null) {
                @SuppressWarnings("unchecked")
                ThreadLocal<Object> key = (ThreadLocal<Object>) e.get(); 
                if (key != null) {
                    Object value = key.childValue(e.value); 
                    ThreadLocal.ThreadLocalMap.Entry c = new ThreadLocal.ThreadLocalMap.Entry(key, value); 
                    int h = key.threadLocalHashCode & (len - 1); 
                    while (table[h] != null)
                        h = nextIndex(h, len); 
                    table[h] = c; 
                    size++;
                }
            }
        }
    }

    
    protected T childValue(T parentValue) {
        return parentValue; 
    }```

三、ITL所带来的的问题
------------

看过上述代码后，现在关于ITL的实现我们基本上有了清晰的认识了，根据其实现性质，可以总结出在使用ITL时可能存在的问题：

### 3.1：线程不安全

写在前面：这里讨论的线程不安全对象不包含Integer等类型，因为这种对象被重新赋值，变掉的是整个引用，这里说的是那种不改变对象引用，直接可以修改其内容的对象（典型的就是自定义对象的set方法）

如果说线程本地变量是只读变量不会受到影响，但是如果是可写的，那么任意子线程针对本地变量的修改都会影响到主线程的本地变量（本质上是同一个对象），参考上面的第三个例子，子线程写入后会覆盖掉主线程的变量，也是通过这个结果，我们确认了子线程TLMap里变量指向的对象和父线程是同一个。

### 3.2：线程池中可能失效

按照上述实现，在使用线程池的时候，ITL会完全失效，因为父线程的TLMap是通过init一个Thread的时候进行赋值给子线程的，而线程池在执行异步任务时可能不再需要创建新的线程了，因此也就不会再传递父线程的TLMap给子线程了。

针对上述2，我们来做个实验，来证明下猜想：

```null


    private static ExecutorService executorService = Executors.newFixedThreadPool(1);

    private static ThreadLocal tl = new InheritableThreadLocal<>();

    public static void main(String[] args) {

        tl.set(1);
        
        System.out.println(String.format("线程名称-%s, 变量值=%s", Thread.currentThread().getName(), tl.get()));

        executorService.execute(()->{
            System.out.println(String.format("线程名称-%s, 变量值=%s", Thread.currentThread().getName(), tl.get()));
        });

        executorService.execute(()->{
            System.out.println(String.format("线程名称-%s, 变量值=%s", Thread.currentThread().getName(), tl.get()));
        });

        System.out.println(String.format("线程名称-%s, 变量值=%s", Thread.currentThread().getName(), tl.get()));
    }
````

输出结果为：

```null

线程名称-main, 变量值=1
线程名称-pool-1-thread-1, 变量值=1
线程名称-main, 变量值=1
线程名称-pool-1-thread-1, 变量值=1
```

会发现，并没有什么问题，和我们预想的并不一样，原因是什么呢？因为线程池本身存在一个初始化的过程，第一次使用的时候发现里面的线程数（worker 数）少于核心线程数时，会进行创建线程，既然是创建线程，一定会执行 Thread 的 init 方法，参考上面提到的源码，在第一次启用线程池的时候，类似做了一次 new Thread 的操作，因此是没有什么问题的，父线程的 TLMap 依然可以传递下去。

现在我们改造下代码，把 tl.set(1) 改到第一次启用线程池的下面一行，然后再看看：

```null

public static void main(String[] args) throws Exception{

        System.out.println(String.format("线程名称-%s, 变量值=%s", Thread.currentThread().getName(), tl.get()));

        executorService.execute(()->{
            System.out.println(String.format("线程名称-%s, 变量值=%s", Thread.currentThread().getName(), tl.get()));
        });

        tl.set(1); 

        executorService.execute(()->{
            System.out.println(String.format("线程名称-%s, 变量值=%s", Thread.currentThread().getName(), tl.get()));
        });

        System.out.println(String.format("线程名称-%s, 变量值=%s", Thread.currentThread().getName(), tl.get()));
    }
```

输出结果为：

```null

线程名称-main, 变量值=null
线程名称-main, 变量值=1
线程名称-pool-1-thread-1, 变量值=null
线程名称-pool-1-thread-1, 变量值=null
```

很明显，第一次启用时没有递进去的值，在后续的子线程启动时就再也传递不进去了。

## 尾声

但是，在实际项目中我们大多数采用线程池进行做异步任务，假如真的需要传递主线程的本地变量，使用 ITL 的问题显然是很大的，因为是有极大可能性拿不到任何值的，显然在实际项目中，ITL 的位置实在是尴尬，所以在启用线程池的情况下，不建议使用 ITL 做值传递。为了解决这种问题，阿里做了[transmittable-thread-local](https://github.com/alibaba/transmittable-thread-local "TTL")（TTL）来解决线程池异步值传递问题，下一篇，我们将会分析 TTL 的用法及原理。 
 [https://www.cnblogs.com/hama1993/p/10400265.html](https://www.cnblogs.com/hama1993/p/10400265.html)
