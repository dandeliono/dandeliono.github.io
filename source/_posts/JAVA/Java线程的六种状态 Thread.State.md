# Java线程的六种状态 Thread.State
Thread.State 是 Thread 中的一个内部类，表示了 Thread 的六种状态，还有，这个类是一个枚举类。

## **NEW**

线程刚刚 new 出来，热乎的，还未启动

### 代码示例

    public class ThreadStateNewExample {
        private static Object waiter = new Object();
        
        public static void main(String[] args){
            Runnable waiting = () -> {
                try{
                    waiter.wait();
                }catch (InterruptedException e){
                    e.printStackTrace();
                }
            };
            Thread whoWillWait = new Thread(waiting);
            System.out.printf(whoWillWait.getState().toString());
        }
    }
    复制代码

结果

    NEW
    Process finished with exit code 0
    复制代码

## **RUNNABLE**

说明线程已经就绪，可能正在执行某个任务，**也可能在等待 CPU 资源**

### 代码示例

    public class ThreadStateRunnableExample {
        private static boolean flag = true;

        public static void main(String[] args){
            Runnable waiting = () -> {
                //让程序空转，保持线程是runnable状态
                    do { }while (flag);
            };
            Thread thread = new Thread(waiting);
            thread.start();
            try {
                //主线程先睡3秒，让子线程先跑起来，然后输出线程状态
                Thread.sleep(3000);
                System.out.printf(thread.getState().toString());
                //更改标志位，让子线程结束循环
                flag = false;
            }catch (InterruptedException e){
                e.printStackTrace();
            }
            System.exit(1);
        }
    }
    复制代码

### 运行结果

    RUNNABLE
    Process finished with exit code 1
    复制代码

## **BLOCKED**

线程正在阻塞，等待一个 monitor lock。例如等待获取一个文件的锁

### 代码示例

    public class ThreadStateBlockExample {

        private static boolean LOCK_FLAG = true;

        public static void main(String[] args){
            Runnable locker = ThreadStateBlockExample::locker;

            Thread whoWillLockOthers = new Thread(locker);
            /**
             * 启动whoWillLockOthers线程，主线程睡2秒让子线程先运行
             * 此时whoWillLockOthers获得锁，这时候其他线程需要等待
             */
            whoWillLockOthers.start();
            try {
                Thread.sleep(2000);
            }catch (InterruptedException e){
                e.printStackTrace();
            }
            Thread whoWillBeLocked = new Thread(locker);
            /**
             * 启动whoWillBeLocked线程，主线程睡2秒让子线程先运行
             * 因为locker方法是个死循环，所以whoWillBeLocked线程永远拿不到锁，就会进入BLOCKED状态
             */
            whoWillBeLocked.start();
            try {
                Thread.sleep(2000);
            }catch (InterruptedException e){
                e.printStackTrace();
            }
            System.out.printf("whoWillBeLocked 当前状态为="+whoWillBeLocked.getState().toString()+"");
            System.exit(1);
        }


        private static synchronized void locker(){
            do {

            }while (LOCK_FLAG);
        }
    }

    复制代码

### 执行结果

    whoWillBeLocked 当前状态为=BLOCKED
    Process finished with exit code -1
    复制代码

## **WAITING**

代表线程正在等待中。一个线程如果调用下列方法，会导致线程状态变为**WAITING**：

-   Object.wait with no timeout
-   Thread.join with no timeout
-   LockSupport.park

举个栗子：ThreadA 调用了 Object.wait() 方法，此时 ThreadA 状态为**WAITING**。ThreadA 会等待其他的线程调用 Object.notify() 或 Object.notifyAll 才会被唤醒，继续执行后面的逻辑

> 注意！！！**调用 wait() 和 notify() 此类方法必须先获得 Object 的锁，至于原理我们开新帖子去讲，本文主要介绍线程的状态和代码示例，有兴趣的同学也可以在掘金上搜搜其他作者关于锁和 Monitor 的文章。** 

### 代码示例

    public class ThreadStateWaitingExample {
        private static final Object LOCKER = new Object();

        public static void main(String[] args) {

            Runnable waiting = () -> {
                System.out.println("whoWillWait 开始等待 whoWillNotify");
                waiting();
                System.out.println("whoWillWait 等到了 whoWillNotify 的通知");
            };
            //创建一个线程调用waiter.wait()方法，让whoWillWait线程进入waiting状态
            Thread whoWillWait = new Thread(waiting);
            whoWillWait.start();
            //主线程先睡2秒，让whoWillWait先执行
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("whoWillWait当前的线程状态=" + whoWillWait.getState().toString());


            Runnable notify = () -> {
                System.out.println("whoWillNotify 准备通知 whoWillWait");
                notifying();
            };
            //创建一个线程调用waiter.notify()方法，唤醒whoWillWait
            Thread whoWillNotify = new Thread(notify);
            whoWillNotify.start();
            //主线程先睡2秒，让whoWillNotify先执行
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("唤醒后，whoWillWait当前的线程状态=" + whoWillWait.getState().toString());
            System.exit(1);
        }

        private static void waiting() {
            synchronized (LOCKER) {
                try {
                    LOCKER.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

        }

        private static void notifying() {
            synchronized (LOCKER) {
                LOCKER.notify();
                System.out.println("whoWillNotify 已经通知，即将离开同步代码块");
            }
        }
    }
    复制代码

### 执行结果

    whoWillWait 开始等待 whoWillNotify
    whoWillWait当前的线程状态=WAITING
    whoWillNotify 准备通知 whoWillWait
    whoWillNotify 已经通知，即将离开同步代码块
    whoWillWait 等到了 whoWillNotify 的通知
    唤醒后，whoWillWait当前的线程状态=TERMINATED

    Process finished with exit code 1
    复制代码

## TIMED_WAITING

线程正在等待其他线程的操作，直到超过指定的超时时间。线程在调用以下方法是会将状态改变为 TIMED_WAITING 状态:

-   Thread.sleep
-   Object.wait with timeout
-   Thread.join with timeout
-   LockSupport.parkNanos
-   LockSupport.parkUntil

### 代码示例

    public class ThreadStateTimedWaitingExample {
        private static final Object LOCKER = new Object();

        public static void main(String[] args) {

            Runnable waiting = () -> {
                System.out.println("whoWillWait 开始等待 2秒钟");
                waiting();
                System.out.println("whoWillWait 等待结束");
            };
            //创建一个线程调用waiter.wait()方法，让whoWillWait线程进入waiting状态
            Thread whoWillWait = new Thread(waiting);
            whoWillWait.start();
            //主线程先睡1秒，让whoWillWait先执行
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("whoWillWait当前的线程状态=" + whoWillWait.getState().toString());


            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("whoWillWait当前的线程状态=" + whoWillWait.getState().toString());
            System.exit(1);
        }

        private static void waiting() {
            synchronized (LOCKER) {
                try {
                    LOCKER.wait(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

        }
        
    }
    复制代码

### 执行结果

    whoWillWait 开始等待 2秒钟
    whoWillWait当前的线程状态=TIMED_WAITING
    whoWillWait 等待结束
    whoWillWait当前的线程状态=TERMINATED

    Process finished with exit code 1
    复制代码

## TERMINATED

线程已经结束执行，属于退出状态

### 代码示例

    public class ThreadStateTerminatedExample {

        public static void main(String[] args) {

            Runnable waiting = () -> {
                System.out.println("随便执行一下，然后线程就会变为Terminated");
            };
            Thread terminate = new Thread(waiting);
            terminate.start();
            //主线程先睡1秒，让terminate先执行，一秒钟足够terminate执行完毕，然后线程就结束了
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("terminate当前的线程状态=" + terminate.getState().toString());

            System.exit(1);
        }

    }
    复制代码

### 执行结果

    随便执行一下，然后线程就会变为Terminated
    terminate当前的线程状态=TERMINATED

    Process finished with exit code 1
    复制代码

* * *

> **线程在某个时间点只会拥有一种状态**，这些状态都是虚拟机的线程状态，并不是操作系统的线程状态，这是两个概念，不要混淆。 
>  [https://juejin.cn/post/6844903968200458247](https://juejin.cn/post/6844903968200458247)
