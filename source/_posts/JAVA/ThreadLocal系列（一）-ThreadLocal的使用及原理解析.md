# ThreadLocal系列（一）-ThreadLocal的使用及原理解析
## 一、基本使用

先来看下基本用法：

````null

    private static ThreadLocal tl = new ThreadLocal<>();

    public static void main(String[] args) throws Exception {
        tl.set(1);
        System.out.println(String.format("当前线程名称: %s, main方法内获取线程内数据为: %s",
                Thread.currentThread().getName(), tl.get()));
        fc();
        new Thread(ThreadLocalTest::fc).start();
    }

    private static void fc() {
        System.out.println(String.format("当前线程名称: %s, fc方法内获取线程内数据为: %s",
                Thread.currentThread().getName(), tl.get()));
    }```

运行结果：

```null

当前线程名称: main, main方法内获取线程内数据为: 1
当前线程名称: main, fc方法内获取线程内数据为: 1
当前线程名称: Thread-0, fc方法内获取线程内数据为: null```

可以看到，main线程内任意地方都可以通过ThreadLocal获取到当前线程内被设置进去的值，而被异步出去的fc调用，却由于替换了执行线程，而拿不到任何数据值，那么我们现在再来改造下上述代码，在异步发生之前，给Thread-0线程也设置一个上下文数据：

```null

private static ThreadLocal tl = new ThreadLocal<>();

    public static void main(String[] args) throws Exception {
        tl.set(1);
        System.out.println(String.format("当前线程名称: %s, main方法内获取线程内数据为: %s",
                Thread.currentThread().getName(), tl.get()));
        fc();
        new Thread(()->{
            tl.set(2); 
            fc();
        }).start();
        Thread.sleep(1000L); 
        fc(); 
    }

    private static void fc() {
        System.out.println(String.format("当前线程名称: %s, fc方法内获取线程内数据为: %s",
                Thread.currentThread().getName(), tl.get()));
    }
````

运行结果为：

```null

当前线程名称: main, main方法内获取线程内数据为: 1
当前线程名称: main, fc方法内获取线程内数据为: 1
当前线程名称: Thread-0, fc方法内获取线程内数据为: 2
当前线程名称: main, fc方法内获取线程内数据为: 1
```

可以看到，主线程和子线程都可以获取到自己的那份上下文里的内容，而且互不影响。

## 二、原理分析

ok，上面通过一个简单的例子，我们可以了解到 ThreadLocal（以下简称 TL）具体的用法，这里先不讨论它实质上能给我们带来什么好处，先看看其实现原理，等这些差不多了解完了，我再通过我曾经做过的一个项目，去说明 TL 的作用以及在企业级项目里的用处。

我以前在不了解 TL 的时候，想着如果让自己实现一个这种功能的轮子，自己会怎么做，那时候的想法很单纯，觉得通过一个 Map 就可以解决，Map 的 key 设置为 Thread.currentThread()，value 设置为当前线程的本地变量即可，但后来想想就觉得不太现实了，实际项目中可能存在大量的异步线程，对于内存的开销是不可估量的，而且还有个严重的问题，线程是运行结束后就销毁的，如果按照上述的实现方案，map 内是一直持有这个线程的引用的，导致明明执行结束的线程对象不能被 jvm 回收，造成内存泄漏，时间久了，会直接 OOM。

所以，java 里的实现肯定不是这么简单的，下面，就来看看 java 里的具体实现吧。

先来了解下，TL 的基本实现，为了避免上述中出现的问题，TL 实际上是把我们设置进去的值以 k-v 的方式放到了每个 Thread 对象内（TL 对象做 k，设置的值做 v），也就是说，TL 对象仅仅起到一个标记、对 Thread 对象维护的 map 赋值的作用。

先从 set 方法看起：

````null

public void set(T value) {
        Thread t = Thread.currentThread(); 
        ThreadLocal.ThreadLocalMap map = getMap(t); 
        if (map != null)
            map.set(this, value); 
        else
            createMap(t, value); 
    }

    ThreadLocal.ThreadLocalMap getMap(Thread t) {
        return t.threadLocals; 
    }

    private void set(ThreadLocal<?> key, Object value) {

        ThreadLocal.ThreadLocalMap.Entry[] tab = table; 
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1); 

        for (ThreadLocal.ThreadLocalMap.Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) { 
            ThreadLocal<?> k = e.get(); 

            if (k == key) { 
                e.value = value;
                return;
            }

            if (k == null) { 
                replaceStaleEntry(key, value, i);
                return;
            }
        }

        
        tab[i] = new ThreadLocal.ThreadLocalMap.Entry(key, value);
        int sz = ++size;
        if (!cleanSomeSlots(i, sz) && sz >= threshold) 
            rehash(); 
    }


    
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocal.ThreadLocalMap(this, firstValue);
    }

    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new ThreadLocal.ThreadLocalMap.Entry[INITIAL_CAPACITY]; 
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1); 
        table[i] = new ThreadLocal.ThreadLocalMap.Entry(firstKey, firstValue); 
        size = 1;
        setThreshold(INITIAL_CAPACITY); 
    }```

通过上述代码，我们大致了解了TL在set值的时候发生的一些操作，结合之前说的，我们可以确定的是，TL其实对于线程来说，只是一个标识，而真正线程的**本地变量**被保存在每个线程对象的**ThreadLocalMap**里，这个map里维护着一个**Entry\[\]的数组（散列表）**，Entry是个k-v结构的对象（如图1-1），k为TL对象，v为对应TL保存在该线程内的本地变量值，值得注意的是，这里的k针对TL对象的引用是个**弱引用**，来看下源码：

```null

static class Entry extends WeakReference<ThreadLocal<?>> {
            
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
````

为什么这里需要弱引用呢？我们先来看一张图，结合上面的介绍和这张图，来了解 TL 和 Thread 间的关系：

![](https://img2018.cnblogs.com/blog/1569484/201902/1569484-20190219121530516-1149462899.png)

**图 1-1**

图中虚线表示弱引用，那么为什么要这么做呢？

简单来说，一个 TL 对象被创建出来，并且被一个线程放到自己的 ThreadLocalMap 里，假如 TL 对象失去原有的强引用，但是该线程还没有死亡，如果 k 不是弱引用，那么就意味着 TL 并不能被回收，现在 k 为弱引用，那么在 TL 失去强引用的时候，gc 可以直接回收掉它，弱引用失效，这就是上面代码里会进行检查，k=null 的清除释放内存的原因（这个可以参考下面**expungeStaleEntry**方法，而且 set、get、remove 都会调用该方法，这也是 TL 防止内存泄漏所做的处理）。

综上，简单来说这个**弱引用**就是用来解决由于使用 TL 不当导致的**内存泄漏**问题的，假如没有弱引用，那么你又用到了线程池（**池化后线程不会被销毁**），然后 TL 对象又是**局部**的，那么就会导致线程池内线程里的**ThreadLocalMap**存在大量的无意义的 TL 对象引用，造成过多无意义的 Entry 对象，因为即便调用了 set、get 等方法检查 k=null，也没有作用，这就导致了**内存泄漏**，长时间这样最终可能导致 OOM，所以 TL 的开发者为了解决这种问题，就将 ThreadLocalMap 里对 TL 对象的引用改为**弱引用**，一旦 TL 对象失去**强引用**，TL 对象就会被回收，那么这里的**弱引用**指向的值就为 null，结合上面说的，调用操作方法时会检查 k=null 的 Entry 进行回收，从而避免了内存泄漏的可能性。

因为 TL 解决了内存泄漏的问题，因此即便是局部变量的 TL 对象且启用线程池技术，也比较难造成内存泄漏的问题，而且我们经常使用的场景就像一开始的示例代码一样，会初始化一个**全局的 static 的 TL 对象**，这就意味着该对象在程序运行期间都不会存在强引用消失的情况，我们可以利用不同的 TL 对象给不同的 Thread 里的 ThreadLocalMap 赋值，通常会 set 值（覆盖原有值），因此在使用线程池的时候也不会造成问题，异步开始之前 set 值，用完以后 remove，TL 对象可以多次得到使用，启用线程池的情况下如果不这样做，很可能业务逻辑也会出问题（一个线程存在之前执行程序时遗留下来的本地变量，一旦这个线程被再次利用，get 时就会拿到之前的脏值）；

说完了 set，我们再来看下 get：

```null

public T get() {
        Thread t = Thread.currentThread();
        ThreadLocal.ThreadLocalMap map = getMap(t); 
        if (map != null) {
            ThreadLocal.ThreadLocalMap.Entry e = map.getEntry(this); 
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result; 
            }
        }
        return setInitialValue(); 
    }

    private ThreadLocal.ThreadLocalMap.Entry getEntry(ThreadLocal<?> key) {
        int i = key.threadLocalHashCode & (table.length - 1); 
        ThreadLocal.ThreadLocalMap.Entry e = table[i];
        if (e != null && e.get() == key) 
            return e;
        else
            return getEntryAfterMiss(key, i, e); 
    }

    private ThreadLocal.ThreadLocalMap.Entry getEntryAfterMiss(ThreadLocal<?> key, int i, ThreadLocal.ThreadLocalMap.Entry e) {
        ThreadLocal.ThreadLocalMap.Entry[] tab = table;
        int len = tab.length;

        while (e != null) {
            ThreadLocal<?> k = e.get(); 
            if (k == key) 
                return e;
            if (k == null) 
                expungeStaleEntry(i);
            else 
                i = nextIndex(i, len);
            e = tab[i];
        }
        return null; 
    }


    
    private static int nextIndex(int i, int len) {
        return ((i + 1 < len) ? i + 1 : 0);
    }
```

最后再来看看 remove 方法：

```null

public void remove() {
        ThreadLocal.ThreadLocalMap m = getMap(Thread.currentThread());
        if (m != null)
            m.remove(this); 
    }

    private void remove(ThreadLocal<?> key) {
        ThreadLocal.ThreadLocalMap.Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1); 
        for (ThreadLocal.ThreadLocalMap.Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            if (e.get() == key) { 
                e.clear(); 
                expungeStaleEntry(i); 
                return;
            }
        }
    }

    private int expungeStaleEntry(int staleSlot) { 
        ThreadLocal.ThreadLocalMap.Entry[] tab = table;
        int len = tab.length;

        tab[staleSlot].value = null; 
        tab[staleSlot] = null;
        size--;

        
        ThreadLocal.ThreadLocalMap.Entry e;
        int i;
        
        for (i = nextIndex(staleSlot, len);
             (e = tab[i]) != null;
             i = nextIndex(i, len)) {
            ThreadLocal<?> k = e.get();
            if (k == null) { 
                e.value = null;
                tab[i] = null;
                size--;
            } else {
                int h = k.threadLocalHashCode & (len - 1);
                if (h != i) {
                    tab[i] = null;

                    
                    
                    while (tab[h] != null)
                        h = nextIndex(h, len);
                    tab[h] = e;
                }
            }
        }
        return i;
    }
```

目前主要方法 set、get、remove 已经介绍完了，包含其内部存在的弱引用的作用，以及实际项目中建议的用法，以及为什么要这样用，也进行了简要的说明，下面一篇会进行介绍**InheritableThreadLocal**的用法以及其原理性分析。 
 [https://www.cnblogs.com/hama1993/p/10382523.html](https://www.cnblogs.com/hama1993/p/10382523.html)
