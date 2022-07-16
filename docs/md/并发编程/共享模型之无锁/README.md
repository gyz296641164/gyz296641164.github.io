<h1 align="center">共享模型之无锁</h1>

# 全文目录

- [全文目录](#全文目录)
- [1、问题提出](#1问题提出)
  - [1.1 使用`synchronized`保证线程安全](#11-使用synchronized保证线程安全)
  - [1.2 使用无锁来保证安全](#12-使用无锁来保证安全)
- [2、CAS 与 volatile](#2cas-与-volatile)
  - [2.1 CAS + 重试 原理](#21-cas--重试-原理)
  - [2.2 volatile的作用](#22-volatile的作用)
  - [2.3 为什么无锁效率高](#23-为什么无锁效率高)
  - [2.4 CAS 的特点](#24-cas-的特点)
- [3、CAS原理](#3cas原理)

---

# 1、问题提出

## 1.1 使用`synchronized`保证线程安全

有如下需求，保证`account.withdraw`取款方法的线程安全, 下面使用`synchronized`保证线程安全：

```java
package com.gyz.nonelock;

import java.util.ArrayList;
import java.util.List;

/**
 * @Description
 * @Author GongYuZhuo
 * @Date 2021/7/3 17:49
 * @Version 1.0.0
 */
public interface Account {

    //获取余额
    Integer getBalance();

    //转账
    void withdraw(Integer amount);

    /**
     * @param account :
     * @return void
     * @Description 启动1000个线程，初始金额10000，做1000次减操作，正常结果是0
     */
    static void demo(Account account) {

        List<Thread> list = new ArrayList<>();
        long start = System.nanoTime();

        for (int i = 0; i < 1000; i++) {
            list.add(new Thread(() -> {
                account.withdraw(10);
            }));
        }

        list.forEach(Thread::start);
        list.forEach(t -> {
            try {
                t.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        long end = System.nanoTime();
        System.out.println(account.getBalance() + " cost " + (end - start) / 1000_000 + "ms");
    }

}

```

```java
package com.gyz.nonelock;

public class AccountUnsafe implements Account {


    private Integer balance;

    public AccountUnsafe(Integer balance) {
        this.balance = balance;
    }

    @Override
    public Integer getBalance() {
        synchronized (this){
            return balance;

        }
    }

    @Override
    public  void withdraw(Integer amount) {
        synchronized (this){
            this.balance -= amount;

        }
    }

    public static void main(String[] args) {
        Account.demo(new AccountUnsafe(10000));
    }
}
```

**synchronized加锁操作太耗费资源 (因为底层使用了操作系统mutex指令, 造成内核态和用户态的切换)**



## 1.2 使用无锁来保证安全

```java
package com.gyz.nonelock;

import java.util.concurrent.atomic.AtomicInteger;

public class AccountCas implements Account {

    /** 使用原子整数，底层使用：CAS + 重试机制 */
    private AtomicInteger balance;


    public AccountCas(int balance) {
        this.balance = new AtomicInteger(balance);
    }

    @Override
    public Integer getBalance() {
        //获得原子整数的值
        return balance.get();
    }

    @Override
    public void withdraw(Integer amount) {
        while (true) {
            //获取取钱之前的值
            int pre = balance.get();
            //获取转账之后的余额
            int next = pre - amount;


            /**
             * @Description 此时的prev为共享变量的值, 如果prev被别的线程改了.也就是说: 自己读到的 共享变量的值 和 共享变量最新值 不匹配,
             * 				就继续where(true),如果匹配上了, 将next值设置给共享变量。
             *
             * 				AtomicInteger中value属性, 被volatile修饰, 就是为了确保线程之间共享变量的可见性.
             * @param amount :
             * @return void
             */
            if (balance.compareAndSet(pre, next)) {
                break;
            }
        }
    }

    public static void main(String[] args) {

        Account.demo(new AccountCas(10000));
    }
}
```



***

# 2、CAS 与 volatile  

使用原子操作来保证线程访问共享资源的安全性, CAS+重试的机制来确保(乐观锁思想)，相对于悲观锁思想的`synchronized`，`reentrantLock`来说，CAS的方式效率会更好！

## 2.1 CAS + 重试 原理

前面看到的`AtomicInteger`的解决方法，内部并`没有用锁`来保护`共享变量`的线程安全。那么它是如何实现的呢？

```java
 public void withdraw(Integer amount) {
        while (true) {
            //获取旧值
            int pre = balance.get();
            //获取转账之后的余额
            int next = pre - amount;
            
           /**
             * @Description
             *  compareAndSet 保证操作共享变量安全性的操作:
             *          ① 线程A首先获取balance.get(),拿到当前的balance值prev
             *          ② 根据这个prev值 - amount值 = 修改后的值next
             *          ③ 调用compareAndSet方法, 首先会判断当初拿到的prev值,是否和现在的
             *          	balance值相同;
             *          	    如果相同,表示其他线程没有修改balance的值, 此时就可以将next值设置给balance属性
             *          	    如果不相同,表示其他线程也修改了balance值, 此时就设置next值失败,
             *                 
             * 				然后一直重试, 重新获取balance.get()的值,计算出next值,
             * 				并判断本次的prev和balnce的值是否相同...重复上面操作
             * 
             * 
             */
             if (balance.compareAndSet(pre, next)) {
                break;
            }
        }
     
 }   
```

其中的关键是 `compareAndSwap（比较并设置值）`，它的简称就是 `CAS` （也有 Compare And Swap 的说法），它必须是`原子操作`。

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Concurrent/20220716075841.png" alt="1594776811158" style="zoom:67%;" />

**流程 :**

当一个线程要去修改`Account对象`中的值时，`先获取值prev（调用get方法）`，然后再将其设置为新的值`next`（调用cas方法）。在调用cas方法时，会将`prev`与`Account中的余额`进行比较。

- 如果两者**相等**，就说明该值还未被其他线程修改，此时便可以进行修改操作。
- 如果两者**不相等**，就不设置值，重新获取值prev（调用get方法），然后再将其设置为新的值next（调用cas方法），直到修改成功为止。

**注意 :**

- 其实 `CAS` 的底层是 **lock cmpxchg** 指令（X86 架构），在单核 CPU 和多核 CPU 下都能够保证【比较-交换】的 **原子性**。
- 在多核状态下，某个核执行到带 lock 的指令时，**CPU 会让总线锁住，当这个核把此指令执行完毕，再开启总线。这个过程中不会被线程的调度机制所打断，保证了多个线程对内存操作的准确性，是原子的。**



## 2.2 volatile的作用

- 在上面代码中的`AtomicInteger类`，**保存值的`value属性`使用了`volatile 修饰`**。获取共享变量时，为了`保证该变量的可见性`，需要使用 **volatile 修饰**。
- volatile可以用来修饰 **成员变量和静态成员变量**，他可以避免线程从自己的工作缓存中查找变量的值，必须到主存中获取它的值，线程操作 volatile 变量都是直接操作主存。**即一个线程对 volatile 变量的修改，对另一个线程可见。**
- **CAS 必须借助 volatile 才能读取到共享变量的最新值来实现【比较并交换】的效果**
- 注意:` volatile` 仅仅保证了共享变量的`可见性`，让其它线程能够看到最新值，但不能解决指令交错问题（**不能保证原子性**）

 

## 2.3 为什么无锁效率高  

- 使用CAS+重试---无锁情况下，即使`重试失败`，线程始终在高速运行，没有停歇，而 `synchronized`会让线程在没有获得锁的时候，`发生上下文切换，进入阻塞`。
  - 打个比喻：线程就好像高速跑道上的赛车，高速运行时，速度超快，一旦发生上下文切换，就好比赛车要减速、熄火，等被唤醒又得重新打火、启动、加速… 恢复到高速运行，代价比较大!
- 但无锁情况下，因为线程要保持运行，需要额外 CPU 的支持，CPU 在这里就好比高速跑道，没有额外的跑道，线程想高速运行也无从谈起，虽然不会进入阻塞，但由于没有分到时间片，仍然会进入可运行状态，还是会导致上下文切换。



## 2.4 CAS 的特点 

结合 CAS 和 volatile 可以实现无锁并发，适用于线程数少、多核 CPU 的场景下。  

- CAS 是基于乐观锁的思想：最乐观的估计，不怕别的线程来修改共享变量，就算改了也没关系，不断再重试呗。

- synchronized 是基于悲观锁的思想：最悲观的估计，得防着其它线程来修改共享变量，我上了锁你们都别想改，我改完了解开锁，你们才有机会。  

- CAS 体现的是无锁并发、无阻塞并发，请仔细体会这两句话的意思  ：

  - 因为没有使用 synchronized，所以线程不会陷入阻塞，这是效率提升的因素之一  
  - 但如果竞争激烈，可以想到重试必然频繁发生，反而效率会受影响  




***

# 3、CAS原理

**1、什么是CAS？**

- CAS：Compare and Swap，即比较再交换；
- jdk5增加了并发包`java.util.concurrent.*`，其下面的类使用CAS算法实现了区别于synchronouse同步锁的一种乐观锁。JDK 5之前Java语言是靠synchronized关键字保证同步的，这是一种独占锁，也是是悲观锁。

**2、CAS算法理解**

- 对CAS的理解，CAS是一种无锁算法，CAS有3个操作数：`内存值V`，`旧的预期值A`，`要修改的新值B`。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。

- CAS比较与交换的伪代码可以表示为：

  ```java
  do{
  
  备份旧数据；
  
  基于旧数据构造新数据；
  
  }while(!CAS( 内存地址，备份的旧数据，新数据 ))
  ```

  <img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Concurrent/20220716075842.png" alt="image-20210826151510174" style="zoom:50%;" />

`注：t1，t2线程是同时更新同一变量56的值`

- 因为t1和t2线程都同时去访问同一变量56，所以他们会把主内存的值完全拷贝一份到自己的工作内存空间，所以t1和t2线程的预期值都为56。

- 假设t1在与t2线程竞争中线程t1能去更新变量的值，而其他线程都失败。（失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次发起尝试）。t1线程去更新变量值改为57，然后写到内存中。此时对于t2来说，内存值变为了57，与预期值56不一致，就操作失败了（想改的值不再是原来的值）。

- 上图通俗的解释是：CPU去更新一个值，但如果想改的值不再是原来的值，操作就失败，因为很明显，有其它操作先改变了这个值。

- 就是指当两者进行比较时，如果相等，则证明共享数据没有被修改，替换成新值，然后继续往下运行；如果不相等，说明共享数据已经被修改，放弃已经所做的操作，然后重新执行刚才的操作。容易看出 CAS 操作是基于共享数据不会被修改的假设，采用了类似于数据库的commit-retry 的模式。当同步冲突出现的机会很少时，这种假设能带来较大的性能提升。

  

**3、CAS开销**

前面说过了，`CAS（比较并交换）`是CPU指令级的操作，只有一步原子操作，所以非常快。而且CAS避免了请求操作系统来裁定锁的问题，不用麻烦操作系统，直接在CPU内部就搞定了。但CAS就没有开销了吗？不！有`cache miss`情况。这个问题比较复杂，首先需要了解CPU的硬件体系结构。自行查阅！

**4、CAS 的问题**

- **CAS 容易造成 ABA 问题**

  一个线程将数值a改成了 b，接着又改成了 a，此时 CAS 认为是没有变化，其实是已经变化过了，而这个问题的解决方案可以使用版本号标识，每操作一次version 加 1。在 java5 中，已经提供了 `AtomicStampedReference` 来解决问题。 

- **不能保证代码块的原子性**

  CAS 机制所保证的只是一个变量的原子性操作，而不能保证整个代码块的原子性。比如需要保证 3 个变量共同进行原子性的更新，就不得不使用 synchronized 了。 

- **CAS 造成CPU 利用率增加**

  之前说过了 CAS 里面是一个循环判断的过程，如果线程一直没有获取到状态，cpu资源会一直被占用。



