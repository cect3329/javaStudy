[toc]



# 0.准备工作

1. maven 中 导入依赖 lombok
2. IDEA 中 Project 和 Modules 中的 Language level 为 8版本

![image-20201112161006172](多线程进阶JUC.assets/image-20201112161006172.png)![image-20201112161027920](多线程进阶JUC.assets/image-20201112161027920.png)![image-20201112161059940](多线程进阶JUC.assets/image-20201112161059940.png)

![image-20201112163533343](多线程进阶JUC.assets/image-20201112163533343.png)





# 1. 什么是JUC

- 源码+官方文档

![image-20201112163457441](多线程进阶JUC.assets/image-20201112163457441.png)



- 就是 java 工具包下的一些包和类 学习它的使用



# 2. 线程和进程

- java 默认有 2个线程， main线程和GC线程
- java无法开启线程，真正开启线程的是一个本地方法

```java
public synchronized void start() {
        /**
         * This method is not invoked for the main method thread or "system"
         * group threads created/set up by the VM. Any new functionality added
         * to this method in the future may have to also be added to the VM.
         *
         * A zero status value corresponds to state "NEW".
         */
        if (threadStatus != 0)
            throw new IllegalThreadStateException();

        /* Notify the group that this thread is about to be started
         * so that it can be added to the group's list of threads
         * and the group's unstarted count can be decremented. */
        group.add(this); //加入一个线程组

        boolean started = false;
        try {
            start0(); //调用一个start0()方法
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }

    private native void start0();  //start0()方法 native修饰方法 表示 该方法为一个本地方法，底层的C++去实现的
```

- 并发编程：为了充分利用cpu的资源

```java
//获取cpu的核数
package com.zdp;

public class Test01 {
    public static void main(String[] args) {
        //获取cpu的核数
        //cpu 密集型，IO密集型
        System.out.println(Runtime.getRuntime().availableProcessors());
    }
}
```



# 3.Lock锁（重点）

- 线程就是一个单独的资源类，他没有任何的附属操作（拿来即用）==>线程去操作资源类，类似线程不生产不拥有东西，线程只是去使用东西（写代码的思想模式）
- 并发：多个线程操作同一个资源，把资源丢入线程中

```java
package com.zdp;

//类似下面的操作，只是将要操作的资源类丢入线程中，线程去操作资源类，线程本身不拥有
//这里使用了传统的synchroinze锁
public class Test01 {
    public static void main(String[] args) {
        Tickets t = new Tickets();

        new Thread(()->{
            for (int i = 0; i < 20; i++) {
                t.sale();
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"小明").start();
        new Thread(()->{
            for (int i = 0; i < 20; i++) {
                t.sale();
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"小红").start();
        new Thread(()->{
            for (int i = 0; i < 20; i++) {
                t.sale();
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"小刚").start();

    }
}

class Tickets{
    int ticketsNum = 30;
    public synchronized void sale(){
        if(ticketsNum>0){
            System.out.println(Thread.currentThread().getName()+"卖出去了第"+(31-ticketsNum--)+"张票,还剩余"+ticketsNum+"张票");
        }
    }
}

```



## 3.1 Lock为接口

- 主要是加锁和解锁
  - 可以看到lock使用的步骤
    - new 一个 Lock锁对象 `Lock l = new ReentrantLock()`
    - `l.lock(); `加锁
    - try{}中 编写业务代码
    - finally 中` l.unlock();`解锁

![image-20201112173817152](多线程进阶JUC.assets/image-20201112173817152.png)

- 关于lock接口

![image-20201112173733005](多线程进阶JUC.assets/image-20201112173733005.png)

- 关于可重入锁 `ReentrantLock` （平常较常使用）

![image-20201112174109665](多线程进阶JUC.assets/image-20201112174109665.png)

- 公平锁：十分公平，可以先来后到 --->后面等待的线程竞争锁的机制是先来后到	
- 非公平锁：不公平，可以插队--->即后面等待的线程竞争锁是根据cpu调度来的（默认）

## 3.2 Lock 和 Synchronized 的区别

1. Synchronized 为内置的java关键字，Lock 是一个java类
2. Synchronized 无法判断获取锁的状态，Lock 可以判断是否获得了锁
3. Synchronized 会自动释放锁，Lock必须要手动释放锁，如果不释放锁，死锁
4. Synchronized 线程1（获得锁，阻塞）、线程2（等待，永远地等待）； Lock锁就不一定会等待下去==> `lock.tryLock()`等不到锁就结束了
5. Synchronized 可重入锁，不可以中断的，非公平锁； Lock，可重入锁，可以判断锁，可以自己设置公平/非公平锁
6. Synchronized 适合锁少量的代码同步问题 ； Lock适合锁大量的同步代码

# 4 生产者消费者问题（Lock实现)

- **面试：单例模式，八大排序算法，生产者消费者问题**

- 解决虚假唤醒问题：

  - 判断条件应该在while循环体内（不要使用if判断） ====>因为 wait()的线程被唤醒后，是进入对象的锁池中，等待cpu调度来竞争锁，而**if判断只有一次**，也就是说，可能**在线程wait()阻塞的期间，有其他线程已经改变了 对象的值，而被唤醒 被调度的线程 进入运行状态，是不需要再进行判断的（因为在wait()之前已经进行了判断）===>但为了防止 线程在wait()阻塞期间，其他线程改变了对象的值，所以线程在进入运行状态后，应该重新进行判断， 所以用判断条件应该放在循环体内**

  ![image-20201112181537467](多线程进阶JUC.assets/image-20201112181537467.png)



## 4.1 Condition 接口

![image-20201112181846781](多线程进阶JUC.assets/image-20201112181846781.png)

- 通过Lock 找到 Condition
- Condition 对应 同步监视器  Lock 对应 Synchronized
- 下图中使用两个 `Condition` ===>一个对象中 有两个等待池 ===>`Contidion`**使对象具有多个等待池，可能就是这样子来实现精确唤醒的？(没错！) 线程可以进入不同的等待池，但线程还是竞争同一个锁对象**

![image-20201112182109629](多线程进阶JUC.assets/image-20201112182109629.png)



## 4.2**Condition 可以精确地通知和唤醒线程**（优势所在）

- 利用 `Condition`来指定线程执行顺序

```java
package com.zdp;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

//创建 A B C三个线程 保证线程的执行顺序为 A->B->C
public class TestCondition {
    public static void main(String[] args) {
        //创建一个公共资源类
        Data1 data = new Data1();

        new Thread(()->{
            for (int i = 0; i < 5; i++) {
                data.printA();
            }
        },"A").start();

        new Thread(()->{
            for (int i = 0; i < 5; i++) {
                data.printB();
            }
        },"B").start();

        new Thread(()->{
            for (int i = 0; i < 5; i++) {
                data.printC();
            }
        },"C").start();

    }
}

class Data1{
    private Lock lock = new ReentrantLock();
    private Condition condition1 = lock.newCondition();
    private Condition condition2 = lock.newCondition();
    private Condition condition3 = lock.newCondition();
    private int Num=1;

    public void printA(){
        lock.lock();
        try{
            while(Num!=1){
                condition1.await();
            }
            System.out.println(Thread.currentThread().getName());
            Num = 2;
            condition2.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void printB(){
        lock.lock();
        try{
            while(Num!=2){
                condition2.await();
            }
            System.out.println(Thread.currentThread().getName());
            Num = 3;
            condition3.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void printC(){
        lock.lock();
        try{
            while(Num!=3){
                condition3.await();
            }
            System.out.println(Thread.currentThread().getName());
            Num = 1;
            condition1.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

}
```



# 5 八锁现象（关于锁的八个问题）

1. 线程执行 没有用Synchroinzed 修饰的方法/代码块（普通方法） ===>没有锁 ，就不用竞争锁了===>线程的执行就不受锁的影响
2. `synchronized` 锁的对象 是方法的调用者 ===>线程调用不同对象的同步方法，互不影响，因为竞争的不是同一把锁
3. 静态的同步方法  ====> 因为 `synchronized` 锁的对象是方法的调用者，而静态方法的调用者是类，所以 `synchronized`锁的**该类对应的Class对象**====>线程调用不同对象的静态同步方法，竞争的是同一把锁（**该类对应的Class对象的锁,类对应的Class对象全局唯一**）
4. 静态同步方法和普通同步方法，同一个对象===> 静态同步方法和普通同步方法锁的对象不同，静态同步方法锁的是Class对象，而普通同步方法锁的是方法的调用者



# 6 集合安全类

## 6.1 CopyOnWriteArrayList

- 并发修改 不安全的集合类会产生异常 java.util.ConcurrentModificationException

```java
package com.zdp;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

public class TestList {
    public static void main(String[] args) {

        List<String> list = new ArrayList<>();

        for (int i = 0; i < 10; i++) {
            new Thread(()->{
                list.add(UUID.randomUUID().toString().substring(0,5));
                System.out.println(list);
            },String.valueOf(i)).start();
        }
    }
}

```

- 运行结果图：

![image-20201112202249902](多线程进阶JUC.assets/image-20201112202249902.png)

- 解决方法：

  - 使用工具类下的安全的集合类	`List<String> list = Collections.synchronizedList(new ArrayList<>());`
  - 使用`java.util.concurrent`包下的类

  ![image-20201112201908814](多线程进阶JUC.assets/image-20201112201908814.png)

- 使用 `CopyOnWrite `类 ===>**写入时复制**  COW思想：计算基础程序设计领域的一种优化策略

  - `List<String> list = new CopyOnWriteArrayList<>();`
  - 多个线程调用的时候，list,读取的时候 是固定的 ； 写入的时候，复制一份对象中的数据，将复制的数据给调用者，调用者写入完成后，再将修改后的复制数据返还给对象，更新对象
  - 防止在多个线程同时写入的时候，因为数据覆盖而造成的数据问题

  ![image-20201112203333912](多线程进阶JUC.assets/image-20201112203333912.png)



## 6.2 CopyOnWriteArraySet

- 跟 ArrayList没什么区别 ===>遇到异常找替代的方法，同ArrayList 的解决方法差不多

  - `Set<String> set = Collections.synchronizedSet(new HashSet<>());`
  - `Set<String> set = new CopyOnWriteArraySet<>();`

- HashSet的底层

  - 本质上是一个HashMap

  ![image-20201112204122966](多线程进阶JUC.assets/image-20201112204122966.png)

  - add方法

  ![img](file:///C:\Users\zdp\AppData\Roaming\Tencent\Users\499799703\QQ\WinTemp\RichOle\RPDW1SDEQ$4[P%9IS$L3}OF.png)

  ![image-20201112204335622](多线程进阶JUC.assets/image-20201112204335622.png)



## 6.3 ConcurrentHashMap

- 没啥好写的，可以去深挖一下；









# 7.Callable

- Callable

![image-20201112211032855](多线程进阶JUC.assets/image-20201112211032855.png)

- `Thread()`只能实现 `Runnable `接口实现类 ====>如何让 `Thread() `来启动 Callable接口

  - 因为` Thread() `只能启动 `Runnable` 接口实现类，既然 `Thread()`无法启动 `Callable`接口，那么只能让`Callable `来与` Runnable`产生联系
  - 所以可以看一下` Runnable` 接口的实现类中是否有于` Callable`接口相关联的 ====>`FutureTask`  (Runnable接口实现类) 实现了Runnable接口的`run()`方法

  ![image-20201112211257644](多线程进阶JUC.assets/image-20201112211257644.png)

  ![image-20201112211305497](多线程进阶JUC.assets/image-20201112211305497.png)

  - `FutureTask` 为适配类

  ```java
  class MyCallable implements Callable<Integer>{
      
      public Integer call(){
          sout("call()");
          return 1024;
      }
  }
  
  main(){
      MyCallable mycallable = new Mycallable();
      FutureTask task = new FutureTask(mycallable);
      new Thread(task,"A").start();
      new Thread(task,"B").start(); //call()会被打印一次 ===>为什么？ 可以去查一下 
      Integer result =(Integer) task.get();  //获得Callable 的返回结果 ，会产生阻塞，一般放到最后一行或者使用异步通信来处理
  }
  ```

- 为什么call()只被打印一次？？

![image-20201112214215020](多线程进阶JUC.assets/image-20201112214215020.png)

![image-20201112214317588](多线程进阶JUC.assets/image-20201112214317588.png)

- 根据JDK文档介绍：

> Sets the result of this future to the given value unless this future has already been set or has been cancelled.
>
> This method is invoked internally by the [`RunnableFuture.run()`](RunnableFuture.html#run()) method upon successful completion of the computation.
>
> 意思大概是：将future的返回值设置为给定值，除非这个future的outcome已经被设置了。



# 8. 常用的辅助类（必会）

## 8.1 CountDownLatch

- 用来计数的（计数器，减法），是一个辅助工具类，结合代码理解？

```java
package com.zdp;

import java.util.concurrent.CountDownLatch;

public class TestCoutDownLatch {
    public static void main(String[] args) throws InterruptedException {
        //总数是6
        //在必须要先执行完某些任务才能往下执行时使用
        CountDownLatch countDownLatch = new CountDownLatch(6);
        //等待六个线程执行完毕
        for (int i = 1; i <= 6; i++) {
            new Thread(()->{
                System.out.println(Thread.currentThread().getName()+"Go out");
                countDownLatch.countDown();//数量-1
            },String.valueOf(i)).start();
        }

       //countDownLatch.await(); //等待计数器归零，计数器归零才向下执行
        System.out.println("close Door");
    }
}
```

- 运行结果图 :

![image-20201112215022647](多线程进阶JUC.assets/image-20201112215022647.png)

```java
package com.zdp;

import java.util.concurrent.CountDownLatch;

public class TestCoutDownLatch {
    public static void main(String[] args) throws InterruptedException {
        //总数是6
        //在必须要先执行完某些任务才能往下执行时使用
        CountDownLatch countDownLatch = new CountDownLatch(6);
        //等待六个线程执行完毕
        for (int i = 1; i <= 6; i++) {
            new Thread(()->{
                System.out.println(Thread.currentThread().getName()+"Go out");
                countDownLatch.countDown();//数量-1
            },String.valueOf(i)).start();
        }

       countDownLatch.await(); //等待计数器归零，计数器归零才向下执行
        System.out.println("close Door");
    }
}

```

- 运行结果图：

![image-20201112215119402](多线程进阶JUC.assets/image-20201112215119402.png)

- 原理
  - `countDownLatch.countDown();`
  - `countDownLatch.await();`
  - 每次有线程调用 `countDown()`数量-1，等计数器变为0，调用`countDownLatch.await()`的线程就会被唤醒，继续执行；



## 8.2 CyclicBarrier（加法计数器）

- 构造方法

![image-20201112220546268](多线程进阶JUC.assets/image-20201112220546268.png)

```java
package com.zdp;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class TestCyclicBarrier {
    public static void main(String[] args) {
        /*
        * 集齐七颗龙珠，召唤神龙
        * */
        //召唤神龙的线程
        CyclicBarrier cyclicBarrier = new CyclicBarrier(7,()->{
            System.out.println("召唤神龙成功");
        });
        for (int i = 0; i < 7; i++) {
            //lambda 如何操作i变量 ===》内部类的知识点
            final int temp = i;
            new Thread(()->{
                System.out.println("收集到了第"+(temp+1)+"颗龙珠");
                try {
                    cyclicBarrier.await();//等待这七个线程执行完毕，线程每执行完一个，parties+1，知道parties到7
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

![image-20201112220330415](多线程进阶JUC.assets/image-20201112220330415.png)

- 若将代码的执行计数器设置为8

  运行结果如下：（永远无法执行传入`CyclicBarrier`对象的`Runnable`线程）

  ![image-20201112220635900](多线程进阶JUC.assets/image-20201112220635900.png)



- 核心关键：`cyclicBarrier.await();`会进行计数，当条件达到，执行传入`CyclicBarrier`对象的`Runnable`接口实现类

## 8.3 Semaphore （操作系统中的信号量机制）（在并发中用的多）

- `Semaphore` ：代表可以使用的资源（可以使用的线程数）

- 主要方法：

  - `acquire()`：得到可使用的资源，信号量+1，该方法会造成阻塞（如果信号量<=0），直到资源被释放为止
  - `release()`：释放，信号量+1，唤醒等待的线程 ==>每个`acquire()`方法都要跟上一个`release()`
  - 构造方法：（boolean 决定公平锁和不公平锁）

  ![image-20201112221901317](多线程进阶JUC.assets/image-20201112221901317.png)

  ![image-20201112222006415](多线程进阶JUC.assets/image-20201112222006415.png)

- 作用：**多个共享资源互斥的使用！并发限流，控制最大的线程数！**（可以去看看操作系统的信号量机制，可以去复习一波操作系统了）

- 代码：

```java
package com.zdp;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class TestSemaphore {
    public static void main(String[] args) {
        //限流的时候可以使用
        //模拟一个抢车位操作，可供使用的车位为3个，共有6辆车（线程）
        Semaphore semaphore = new Semaphore(3);
        for (int i = 0; i < 6; i++) {
            new Thread(()->{
                //首先获取可用资源
                try {
                    semaphore.acquire();
                    //获取到资源后，执行操作
                    System.out.println(Thread.currentThread().getName()+"抢到车位");
                    //模拟延时
                    TimeUnit.SECONDS.sleep(1);
                    System.out.println(Thread.currentThread().getName()+"离开车位");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    //acuqire() 方法必须与 release()方法配套
                    semaphore.release();
                }
            },String.valueOf(i+1)).start();
        }
    }
}

```

- 运行结果图：

![image-20201112222715628](多线程进阶JUC.assets/image-20201112222715628.png)



# 9. ReadWriteLock (读写锁)

![image-20201113113751733](多线程进阶JUC.assets/image-20201113113751733.png)

> A `ReadWriteLock` maintains a pair of associated [`locks`](Lock.html), one for read-only operations and one for writing. The [read lock](#readLock()) may be held simultaneously by multiple reader threads, so long as there are no writers. The [write lock](#writeLock()) is exclusive.
>
> 大概的意思就是：ReadWriteLock 中有一对 关联锁（读锁和写锁），一个进行只读操作，一个进行写操作。读锁 可以被多个读线程同时占有，只要没有写线程。（读锁和写锁不可同时拥有） 写锁是独占的

![image-20201113114902849](多线程进阶JUC.assets/image-20201113114902849.png)

![image-20201113114938461](多线程进阶JUC.assets/image-20201113114938461.png)

- `ReadWriteLock` 
  - 读-读 可以共存
  - 读-写 不可共存
  - 写-写 不可共存
- 独占锁（写锁） 一次只能被一个线程占有
- 共享锁（读锁） 多个线程可以同时占有

```java
package com.zdp;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class TestReadWriteLock {
    public static void main(String[] args) {
        MyCache mycache = new MyCache();

        //写线程
        for (int i = 1; i < 6; i++) {
            final int temp = i;
            new Thread(()->{
                mycache.put(String.valueOf(temp),String.valueOf(temp));
            },String.valueOf(i)).start();
        }

        //读线程
        for (int i = 1; i < 6; i++) {
            final int temp = i;
            new Thread(()->{
                mycache.get(String.valueOf(temp));
            },String.valueOf(i)).start();
        }

    }
}

/*
* 自定义缓存
* */
class MyCache{
    private volatile Map<String,Object>map = new HashMap<>();
    //读写锁，更加细粒度的控制
    //写锁==>写操作只允许一个线程操作
    //读锁==>读操作可以允许多个线程操作
    private ReadWriteLock readWriteLock = new ReentrantReadWriteLock();

    //存操作，写入的时候，同时只能有一个线程操作
    //写与读不可以同时操作
    public void put(String key,String Object){
        //写锁加锁
        readWriteLock.writeLock().lock();
        try{
            System.out.println(Thread.currentThread().getName()+"写入"+key);
            map.put(key,Object);
            TimeUnit.SECONDS.sleep(1);
            System.out.println(Thread.currentThread().getName()+"写入"+key+"完毕");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            readWriteLock.writeLock().unlock();
        }
    }

    //读操作，读的时候可以同时多个线程读取
    public void get(String key){
        readWriteLock.readLock().lock();
        try{
            System.out.println(Thread.currentThread().getName()+"读取"+key);
            map.get(key);
            TimeUnit.SECONDS.sleep(1);
            System.out.println(Thread.currentThread().getName()+"读取"+key+"完毕");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally{
            readWriteLock.readLock().unlock();
        }
    }
}
```

- 运行结果：

![image-20201113114310078](多线程进阶JUC.assets/image-20201113114310078.png)



# 10 阻塞队列

![image-20201113120230642](多线程进阶JUC.assets/image-20201113120230642.png)

![image-20201113120509317](多线程进阶JUC.assets/image-20201113120509317.png)

- 写入：如果队列满了，就必须阻塞等待读取
- 读取：如果队列是空，必须阻塞等待生产
- `BlockingQueue` 为接口
- 什么时候使用 阻塞队列： 线程池，多线程等（典型场景：生产者消费者问题 )

## 10.1 阻塞队列使用方法

- 构造方法 ===>要传入队列大小的参数

![image-20201113123915553](多线程进阶JUC.assets/image-20201113123915553.png)

四组API

这些操作都好像有加锁，看**源码**

- 抛出异常  ===>添加时，队列已满，抛出异常  删除时，队列已空，抛出异常   获取队首时，队列为空，抛出异常
- 不会抛出异常 ===>offer()方法返回true/false, poll()方法返回被删除的值，如果队列为空，返回null, peek方法，队列为空，返回null
- 阻塞等待

![image-20201113124757745](多线程进阶JUC.assets/image-20201113124757745.png)

![image-20201113124823948](多线程进阶JUC.assets/image-20201113124823948.png)

- 超时等待

![image-20201113125031080](多线程进阶JUC.assets/image-20201113125031080.png)

![image-20201113125054904](多线程进阶JUC.assets/image-20201113125054904.png)

| 方式         | 抛出异常  | 有返回值，不抛出异常 | 阻塞等待 | 超时等待                  |
| ------------ | --------- | -------------------- | -------- | ------------------------- |
| 添加         | add(e)    | offer(e)             | put(e)   | offer(e,timeout,TimeUnit) |
| 删除         | remove()  | poll()               | take()   | poll(timeout,TimeUnit)    |
| 获得队首元素 | element() | peek()               |          |                           |

## 10.2  同步队列	SynchronizedQueue

- 容量为1，队列中放入一个元素，必须被取出来后，才可以再往里面放一个元素；否则会产生阻塞
- `put` `take`操作

```java
package com.zdp;

import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.TimeUnit;

public class TestSynchronizedQueue {
    public static void main(String[] args) throws InterruptedException {
        SynchronousQueue synchronousQueue = new SynchronousQueue();
        //一个线程放
        new Thread(()->{
            try {
                synchronousQueue.put(1);
                System.out.println(Thread.currentThread().getName()+"放入了1");
                synchronousQueue.put(2);
                System.out.println(Thread.currentThread().getName()+"放入了2");
                synchronousQueue.put(3);
                System.out.println(Thread.currentThread().getName()+"放入了3");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"放置线程").start();

        new Thread(()->{
            try {
                TimeUnit.SECONDS.sleep(1);
                System.out.println(Thread.currentThread().getName()+"===>"+synchronousQueue.take());
                TimeUnit.SECONDS.sleep(1);
                System.out.println(Thread.currentThread().getName()+"===>"+synchronousQueue.take());
                TimeUnit.SECONDS.sleep(1);
                System.out.println(Thread.currentThread().getName()+"===>"+synchronousQueue.take());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"读取线程").start();
        TimeUnit.SECONDS.sleep(6);
    }
}

```



# 11 线程池

**池化技术**

- 程序运行===>占用系统资源====>优化资源的使用====>池化技术

  线程池、连接池、内存池、对象池..... （创建和销毁十分浪费资源）

  池化技术：事先准备好一些资源，有人需要使用，就来池子中拿，使用完毕后，归还

- **线程池的好处**

  - 降低资源的消耗
  - 提高响应的速度
  - 方便管理。
  - 线程可以复用，可以控制最大并发数，管理线程

> **线程池：三大方法，七大参数，四种拒绝策略**

- 规范：

![image-20201113160705150](多线程进阶JUC.assets/image-20201113160705150.png)

- 三大方法

  ![image-20201113162119193](多线程进阶JUC.assets/image-20201113162119193.png)

  ```java
  package pool;
  
  import java.util.concurrent.ExecutorService;
  import java.util.concurrent.Executors;
  
  //使用线程池 拿出线程
  //三大方法 1)创建线程池
  //2)执行线程
  //3）关闭线程池
  public class TestExecutors {
      public static void main(String[] args) {
         // ExecutorService threadPool = Executors.newSingleThreadExecutor();//单个线程
        ExecutorService threadPool = Executors.newFixedThreadPool(5);//创建一个固定大小的线程池
       // ExecutorService threadPool = Executors.newCachedThreadPool();//该线程池可以根据需要创建线程， 也会复用以前创建的线程
  
          try{
              for (int i = 0; i < 5; i++) {
                  threadPool.execute(()->{
                      System.out.println(Thread.currentThread().getName()+"执行");
                  });
              }
  
          }finally{
              threadPool.shutdown();
          }
      }
  }
  
  ```

- 七大参数

```java
 public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1, // 核心线程数1，最大线程数1
                                    0L, TimeUnit.MILLISECONDS, //超时单位 毫秒
                                    new LinkedBlockingQueue<Runnable>()));
    }
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,//约为21亿
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads, // 核心线程数和最大线程数为传入的参数nThreads
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

```

可以看出来，三个创建线程池的方法，本质上都是调用 ThreadPoolExecutor()方法，只是传入的方法的参数有所不同而已

- `ThreadPoolExecutor()`
  - 核心线程池大小：===>一般情况下线程池内有多少个线程
  - 线程池最大大小：===>线程池内最多有几个线程
    - 最大线程大小如何定义？
    - 1. **CPU 密集型===>几核cpu 就定义为几，可以保证cpu的效率最高**
      2. **IO密集型==>判断程序中十分耗IO的线程有多少个，假设有15个大型任务，设置为30个（2倍）**
      3. 这两个概念去了解一下
  - `keepAliveTime` ===>超过一段时间没被使用的线程，将会被释放
  - 阻塞队列：==>线程池中的所有线程都在执行 Runnable，此时进来的Runnable 将进入阻塞队列，等待线程空闲
  - 拒绝策略：===>阻塞队列已满，还有Runnable 对象要进入线程池调用线程执行，线程池将拒绝这些Runnable

```java
public ThreadPoolExecutor(int corePoolSize, // 核心线程池大小
                              int maximumPoolSize, //线程池最大大小
                              long keepAliveTime, //保持存在的时间，如果超时还没有人调用，就会释放
                              TimeUnit unit,//超时时间单位
                              BlockingQueue<Runnable> workQueue,//阻塞队列
                              ThreadFactory threadFactory,//线程工厂，用于创建线程，一般不用修改
                              RejectedExecutionHandler handler // 拒绝策略) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

![image-20201113163150250](多线程进阶JUC.assets/image-20201113163150250.png)

- 手动创建线程池

```java
package pool;

import java.util.concurrent.*;

public class CreatPool {
    public static void main(String[] args) {
        ExecutorService threadPool = new ThreadPoolExecutor(2,
                5,
                3,
                TimeUnit.SECONDS,
                new LinkedBlockingDeque<Runnable>(3),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.AbortPolicy() //ThreadPool默认的拒绝策略，当阻塞队列已满，还有Runnable 对象要处理，不受理这个对象，并抛出异常
                );
    }
}

```

- 四大拒绝策略：

![image-20201113164613874](多线程进阶JUC.assets/image-20201113164613874.png)

RejectedExecutionHandler接口的四个实现类:（四大拒绝策略）

![image-20201113164659288](多线程进阶JUC.assets/image-20201113164659288.png)

- `AbortPolicy`
- `CallerRunsPolicy` :哪儿来的去哪儿，比如 main线程 要 使用一个线程来处理Runnable，但线程池满了，只能由main线程来处理这个Runnable
- `DiscardOldestPolicy` ：队列满了，尝试和最早的竞争，不会抛出异常
- `DiscardPolicy`：队列满了，丢掉Runnablb对象，不会抛出异常

















