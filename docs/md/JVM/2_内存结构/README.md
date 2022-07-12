<h1 align="center" style="color:orange">内存结构</h1>

# 整体架构

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155788.png" alt="image-20210611162052821" style="zoom:50%;" />



***

# 1、程序计数器（PC Register）

**程序计数器（Program Counter Register）**

- 是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。在Java虚拟机的概念模型里，字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，它是程序控制流的指示器，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。（记住下一条JVM指令执行的地址）
- 特点：
  - **是线程私有的**
    1. CPU会为每个线程分配时间片，在当前线程时间片使用完以后，CPU就会执行另一线程的代码
    2. 程序计数器是每个线程私有的，当 另一个线程的时间片用完，又返回来执行当前线程的代码时，通过程序计数器就可以知道执行哪一条指令。
  - 唯一一个在《Java虚拟机规范》中没有规定任何OutOfMemoryError情况的区域。



***

# 2、虚拟机栈(Java Virtual Machine Stacks)



**设置栈内存大小**

我们可以使用参数 -Xss选项来设置线程的最大栈空间，栈的大小直接决定了函数调用的最大可达深度。

```
-Xss1m
-Xss1k
```



## 2.1 定义

- 每个线程运行所需要的内存空间，称为**虚拟机栈**
- 每个栈由多个**栈帧**组成，对应着每次调用方法时所占用的内存
- 每个线程只能有**一个活动栈帧**，对应着当前正在执行的方法

### 2.1.1 演示代码

```java
public  class Main { 
    public static void main(String[] args) { 
        method1(); 
    } 
    private static void method1() {            
        method2(1, 2); 
} 
    private static int method2(int a, int b) { 
        int c = a + b; 
        return c; 
    }
}
```

在控制台中可以看到，主类中的方法在进入虚拟机栈的时候，符合栈的特点。

![image-20210611163404136](https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155790.png)



### 2.1.2 问题辨析

**垃圾回收是否涉及栈内存？**

- **不需要**。因为虚拟机栈中是由一个个栈帧组成的，在方法执行完毕后，对应的栈帧就会被弹出栈。所以无需通过垃圾回收机制去回收内存。

**栈内存的分配越大越好吗？**

- 不是。因为**物理内存是一定的**，栈内存越大，可以支持更多的递归调用，但是可执行的线程数就会越少。（解释：假如物理内存500M，每个线程需要1M内存，即需要虚拟机栈为这么多，这时可以有500个线程。但是每个线程需要2M内存，这时可以有250个线程）。

**方法内的局部变量是否是线程安全的？**

- 如果方法内局部变量没有逃离方法的作用范围，则是**线程安全**的。
- 如果如果局部变量引用了对象，并逃离了方法的作用范围，则需要考虑线程安全问题。



### 2.1.3 线程安全问题

**示例代码**

```java
/**
 * 局部变量的线程安全问题
 */
public class Demo1_17 {
    public static void main(String[] args) {
        StringBuilder sb = new StringBuilder();
        sb.append(4);
        sb.append(5);
        sb.append(6);
        new Thread(()->{
            m2(sb);
        }).start();
    }

    public static void m1() {
        StringBuilder sb = new StringBuilder();
        sb.append(1);
        sb.append(2);
        sb.append(3);
        System.out.println(sb.toString());
    }

    public static void m2(StringBuilder sb) {
        sb.append(1);
        sb.append(2);
        sb.append(3);
        System.out.println(sb.toString());
    }

    public static StringBuilder m3() {
        StringBuilder sb = new StringBuilder();
        sb.append(1);
        sb.append(2);
        sb.append(3);
        return sb;
    }
}
```

1. m1()方法中sb为局部变量，线程私有，所以安全！
2. m2()方法的sb作为参数，有可能被其他线程访问，不安全。返回类型改为StringBuffer安全！
3. m3()方法作为return返回值，逃离了当前方法，被别的线程访问到了！不安全！



### 2.1.4 栈帧的内部结构

**每个栈帧中存储着：**

- 局部变量表（Local Variables）
- 操作数栈（operand Stack,或表达式栈）
- 动态链接（DynamicLinking，或指向运行时常量池的方法引用）
- 方法返回地址（Return Address，或方法正常退出或异常退出的定义）
- 一些附加信息

如图所示：

![image-20210611164350550](https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155791.png)

并行每个线程下的栈都是私有的，因此每个线程都有自己各自的栈，并且每个栈里面都有很多栈帧，栈帧的大小主要由`局部变量表`和`操作数栈`决定的。

**局部变量表**

- 局部变量表：Local Variables，被称之为`局部变量数组`或`本地变量表`。
- 定义为一个数字数组，主要用于`存储方法参数`和定义在方法体内的`局部变量`这些数据类型包括各类基本数据类型、对象引用（reference），以及returnAddress类型。
- 局部变量表中的变量**只在当前方法调用中有效**。在方法执行时，虚拟机通过使用局部变量表完成参数值到参数变量列表的传递过程。当方法调用结束后，随着方法栈帧的销毁，局部变量表也会随之销毁。
- 局部变量表所需的**容量大小**是在**编译期确定**下来的，在方法运行期间是不会改变局部变量表的大小的。

**操作数栈**

- 每一个独立的栈帧除了包含局部变量表以外，还包含一个后进先出（Last - In - First -Out）的 操作数栈，也可以称之为 表达式栈（Expression Stack）。
- 操作数栈，**主要用于保存计算过程的中间结果**，同时作为计算过程中变量临时的存储空间。
- 操作数栈就是JVM执行引擎的一个工作区，当一个方法刚开始执行的时候，一个新的栈帧也会随之被创建出来，这个方法的操作数栈是空的。



## 2.2 栈内存溢出

### 2.2.1 栈帧过多导致栈内存溢出

**示例代码：**

```java
/**
 * 演示栈内存溢出 java.lang.StackOverflowError
 * -Xss256k
 */
public class Demo1_2 {
    private static int count;

    public static void main(String[] args) {
        try {
            method1();
        } catch (Throwable e) {
            e.printStackTrace();
            System.out.println(count);
        }
    }

    private static void method1() {
        count++;
        method1();
    }
}
```



### 2.2.2 栈帧过大导致栈内存溢出

**示例代码**

```java

import com.fasterxml.jackson.annotation.JsonIgnore;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.util.Arrays;
import java.util.List;

/**
 * json 数据转换
 */
public class Demo1_19 {

    public static void main(String[] args) throws JsonProcessingException {
        Dept d = new Dept();
        d.setName("Market");

        Emp e1 = new Emp();
        e1.setName("zhang");
        e1.setDept(d);

        Emp e2 = new Emp();
        e2.setName("li");
        e2.setDept(d);

        d.setEmps(Arrays.asList(e1, e2));

        // { name: 'Market', emps: [{ name:'zhang', dept:{ name:'', emps: [ {}]} },] }
        ObjectMapper mapper = new ObjectMapper();
        System.out.println(mapper.writeValueAsString(d));
    }
}

class Emp {
    private String name;
    @JsonIgnore
    private Dept dept;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Dept getDept() {
        return dept;
    }

    public void setDept(Dept dept) {
        this.dept = dept;
    }
}
class Dept {
    private String name;
    private List<Emp> emps;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public List<Emp> getEmps() {
        return emps;
    }

    public void setEmps(List<Emp> emps) {
        this.emps = emps;
    }
}
```

加上`@JsonIgnore`注解解决部门员工之间的循环引用！



## 2.3 线程运行诊断

### 2.3.1 案例一：CPU占用过高

**Linux环境下运行某些程序的时候，可能导致CPU的占用过高，这时需要定位占用CPU过高的线程。**

**运行java代码：**

```
//com.gyz.jvm.t1.Demo1_16代码路径
nohup java com.gyz.jvm.t1.Demo1_16  
nohup: ignoring input and appending output to `nohup.out`	
```

```java
/**
 * 演示 cpu 占用过高
 */
public class Demo1_16 {

    public static void main(String[] args) {
        new Thread(null, () -> {
            System.out.println("1...");
            while(true) {

            }
        }, "thread1").start();


        new Thread(null, () -> {
            System.out.println("2...");
            try {
                Thread.sleep(1000000L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "thread2").start();

        new Thread(null, () -> {
            System.out.println("3...");
            try {
                Thread.sleep(1000000L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "thread3").start();
    }
}

```

- 用`top`命令定位哪个线程对cpu的占用过高。
- `ps H -eo pid,tid,%cpu | grep` （用ps命令进一步定位是哪个线程引起的cpu占用过高），-eo参数为规定输出哪些参数，H 打印线程信息。
- jstack进程id，通过查看进程中的线程的nid：
  - 可以根据线程id找到有问题的线程，进一步定位到问题代码的源码行号
  - 注意jstack查找出的线程id是**16进制的**，**需要转换**



### 2.3.2 案例二：程序运行很长时间没有结果（死锁）

**示例代码：**

```java
/**
 * 演示线程死锁
 */
class A{};
class B{};
public class Demo1_3 {
    static A a = new A();
    static B b = new B();


    public static void main(String[] args) throws InterruptedException {
        new Thread(()->{
            synchronized (a) {
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (b) {
                    System.out.println("我获得了 a 和 b");
                }
            }
        }).start();
        Thread.sleep(1000);
        new Thread(()->{
            synchronized (b) {
                synchronized (a) {
                    System.out.println("我获得了 a 和 b");
                }
            }
        }).start();
    }

}

```

**死锁产生四个条件：**

- 互斥条件：进程要求对所分配的资源（如打印机）进行排他性控制，即在一段时间内某 资源仅为一个进程所占有。此时若有其他进程请求该资源，则请求进程只能等待。
- 不剥夺条件：进程所获得的资源在未使用完毕之前，不能被其他进程强行夺走，即只能 由获得该资源的进程自己来释放（只能是主动释放)。
- 请求和保持条件：进程已经保持了至少一个资源，但又提出了新的资源请求，而该资源 已被其他进程占有，此时请求进程被阻塞，但对自己已获得的资源保持不放。
- 循环等待条件：存在一种进程资源的循环等待链，链中每一个进程已获得的资源同时被 链中下一个进程所请求。即存在一个处于等待状态的进程集合{Pl, P2, ..., pn}，其中Pi等 待的资源被P(i+1)占有（i=0, 1, ..., n-1)，Pn等待的资源被P0占有。



***

# 3、本地方法栈(Native Method Stacks)

一些带有**native关键字**的方法就是需要JAVA去调用本地的C或者C++方法，因为JAVA有时候没法直接和操作系统底层交互，所以需要用到本地方法，**本地方法栈，也是线程私有的**。

允许被实现成固定或者是可动态扩展的内存大小。（在内存溢出方面是相同的）

- 如果线程请求分配的栈容量超过本地方法栈允许的最大容量，Java虚拟机将会抛出一个stackoverflowError 异常。
- 如果本地方法栈可以动态扩展，并且在尝试扩展的时候无法申请到足够的内存，或者在创建新的线程时没有足够的内存去创建对应的本地方法栈，那么Java虚拟机将会抛出一个outofMemoryError异常。

当某个线程调用一个本地方法时，它就进入了一个全新的并且不再受虚拟机限制的世界。它和虚拟机拥有同样的权限。如下图所示。

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155792.png" alt="image-20210611171210708" style="zoom:67%;" />

- 本地方法可以通过本地方法接口来访问虚拟机内部的运行时数据区。
- 它甚至可以直接使用本地处理器中的寄存器。
- 直接从本地内存的堆中分配任意数量的内存。

并不是所有的JVM都支持本地方法。因为Java虚拟机规范并没有明确要求本地方法栈的使用语言、具体实现方式、数据结构等。**如果JVM产品不打算支持native方法，也可以无需实现本地方法栈**。



***

# 4、堆（Heap）

## 4.1 定义

**通过new关键字创建对象都会使用堆内存。特点：**

- 它是线程共享的，堆中对象都需要考虑线程安全的问题。
- Java堆区在JVM启动的时候即被创建，其空间大小也就确定了。是JVM管理的最大一块内存空间。

### 4.1.1 堆参数设置

```
-Xms //用来设置堆空间（年轻代+老年代）的初始内存大小
-X： //是jvm运行参数
ms： //memory start
-Xmx：//用来设置堆空间（年轻代+老年代）的最大内存大小
```

- 一旦堆区中的内存大小超过“-xmx"所指定的最大内存时，将会抛出outofMemoryError异常。
- 通常会将`-Xms`和`-Xmx`两个参数配置相同的值，其目的是**为了能够在ava垃圾回收机制清理完堆区后不需要重新分隔计算堆区的大小，从而提高性能**。
- 默认情况下:
  - 初始内存大小：物理电脑内存大小/64
  - 最大内存大小：物理电脑内存大小/4



### 4.1.2 堆内存细分

**堆空间内部结构，JDK1.8之前永久代，1.8之后永久代替换成元空间。如下图所示。**

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155793.png" alt="image-20210611175149235" style="zoom:67%;" />

**Java 7及之前堆内存逻辑上分为三部分：新生区+养老区+永久区**

- Young Generation Space 新生区 Young/New 又被划分为Eden区和Survivor区
- Tenure generation space 养老区 Old/Tenure
- Permanent Space永久区 Perm

**Java 8及之后堆内存逻辑上分为三部分：新生区+养老区+元空间**

- Young Generation Space：新生区 Young/New 又被划分为Eden区和Survivor区
- Tenure generation space： 养老区 Old/Tenure
- Meta Space： 元空间 Meta

`约定：新生区=新生代=年轻代；养老区=老年区=老年代；永久区=永久代。`

> **年轻代与老年代**

存储在JVM中的Java对象可以被划分为两类：

- 一类是生命周期较短的瞬时对象，这类对象的创建和消亡都非常迅速。生命周期短的，及时回收即可。
- 另外一类对象的生命周期却非常长，在某些极端的情况下还能够与JVM的生命周期保持一致。

Java堆区进一步细分的话，可以划分为`年轻代（YoungGen）`和`老年代（oldGen）`。

其中年轻代又可以划分为：`Eden(伊甸园区)`、`Survivor0空间`和`Survivor1空间`（有时也叫做from区、to区）。如下图所示。

![image-20210611175642041](https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155794.png)

下面这参数开发中一般不会调：

![image-20210611175730724](https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155795.png)

- Eden -> From ->To : **8:1:1**
- 新生代 -> 老年代：**1:2**

配置新生代与老年代在堆结构的占比：

- 默认`-XX:NewRatio=2`，表示新生代占1，老年代占2，新生代占整个堆的1/3 。
- 可以修改`-XX:NewRatio=4`，表示新生代占1，老年代占4，新生代占整个堆的1/5 。

`当发现在整个项目中，生命周期长的对象偏多，那么就可以通过调整老年代的大小，来进行调优。`

**几乎所有的Java对象都是在Eden区被new出来的**。绝大部分的Java对象的销毁都在新生代进行了。（有些大的对象在Eden区无法存储时候，将直接进入老年代）



### 4.1.3 图解对象分配过程

**概念**

为新对象分配内存是一件非常严谨和复杂的任务，JM的设计者们不仅需要考虑内存如何分配、在哪里分配等问题，并且由于内存分配算法与内存回收算法密切相关，所以还需要考虑GC执行完内存回收后是否会在内存空间中产生内存碎片。

- new的对象先放伊甸园区。此区有大小限制。
- 当伊甸园的空间填满时，程序又需要创建对象，JVM的垃圾回收器将对伊甸园区进行垃圾回收（**Minor GC**）,将伊甸园区中的不再被其它对象所引用的对象进行销毁。再加载新得对象放到伊甸园区。
- 然后将伊甸园区中的剩余对象移动到幸存者0区。
- 如果再次触发垃圾回收，此时上次幸存下来的放到幸存者0区的，如果没有回收，就会放到幸存者1区。
- 如果再次经历垃圾回收，此时会重新放回幸存者0区，接着再去幸存者1区。
- 啥时候能去养老区呢？可以设置次数：`-XX:MaxTenuringThreshold=N`，默认15次。
- 在养老区，相对悠闲。当养老区内存不足时，再次触发**Major GC**进行养老区的内存清理。
- 若养老区执行了Major GC/Full GC之后，发现依然无法进行对象的保存，就会产生OOM异常。

**图解过程**

我们创建的对象，一般都是存放在Eden区的，当我们Eden区满了后，就会触发GC操作，一般被称为 **YGC / Minor GC操作**。如下图所示。

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155796.png" alt="image-20210611181650259" style="zoom:67%;" />

当我们进行一次垃圾收集后，红色的将会被回收，而绿色的还会被占用着，存放在S0(Survivor From)区。同时我们给每个对象设置了一个年龄计数器，一次回收后就是1。

同时Eden区继续存放对象，当Eden区再次存满的时候，又会触发一个MinorGC操作，此时GC将会把 Eden和Survivor From中的对象 进行一次收集，把存活的对象放到 Survivor To区，同时让年龄 + 1。

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155797.png" alt="image-20210611181731676" style="zoom:67%;" />

我们继续不断的进行对象生成 和 垃圾回收，当Survivor中的对象的年龄达到15的时候，将会触发一次 Promotion晋升的操作，也就是将年轻代中的对象晋升到老年代中。

![image-20210611181748862](https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155798.png)



**思考：幸存区区满了后？**

**特别注意**：在Eden区满了的时候，才会触发Minor GC，而幸存者区满了后，不会触发Minor GC操作。

如果Survivor区满了后，将会触发一些特殊的规则，也就是可能直接晋升老年代。举例：

- 以当兵为例，正常人的晋升可能是 ： 新兵 -> 班长 -> 排长 -> 连长
- 但是也有可能有些人因为做了非常大的贡献，直接从 新兵 -> 排长



## 4.2 堆内存溢出

**java.lang.OutofMemoryError** ：java heap space. 堆内存溢出

示例代码：

```java
import java.util.ArrayList;
import java.util.List;

/**
 * 演示堆内存溢出 java.lang.OutOfMemoryError: Java heap space
 * -Xmx8m
 */
public class Demo1_5 {

    public static void main(String[] args) {
        int i = 0;
        try {
            List<String> list = new ArrayList<>();
            String a = "hello";
            while (true) {
                list.add(a); // hello, hellohello, hellohellohellohello ...
                a = a + a;  // hellohellohellohello
                i++;
            }
        } catch (Throwable e) {
            e.printStackTrace();
            System.out.println(i);
        }
    }
}

```



## 4.3 堆内存诊断

**常用工具**

- jps 工具
  查看当前系统中有哪些 java 进程
- jmap 工具
  查看堆内存占用情况 jmap - heap 进程id
- jconsole 工具
  图形界面的，多功能的监测工具，可以连续监测
- **jvisualvm（推荐）**

**在IDEA中操作流程为：**

- 在Terminal命令框中输入：jps -> jmap -heap 进程id 。
- 然后可以输入：jconsole或者jvisualvm（推荐）  可打开图形化工具 。

**示例代码**

```java
import java.util.ArrayList;
import java.util.List;

/**
 * 演示查看对象个数 堆转储 dump
 */
public class Demo1_13 {

    public static void main(String[] args) throws InterruptedException {
        List<Student> students = new ArrayList<>();
        for (int i = 0; i < 200; i++) {
            students.add(new Student());
		// Student student = new Student();
        }
        Thread.sleep(1000000000L);
    }
}
class Student {
    private byte[] big = new byte[1024*1024];
}

```



## 4.4 小结

堆空间的参数设置：

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155799.png" alt="image-20210818153240854" style="zoom:67%;" />

***

# 5、方法区（Method Area）

## 5.1 定义

《Java虚拟机规范》中明确说明：“尽管所有的方法区在逻辑上是属于堆的一部分，但一些简单的实现可能不会选择去进行垃圾收集或者进行压缩。”但对于HotSpotJVM而言，**方法区还有一个别名叫做Non-Heap（非堆）**，目的就是要和堆分开。

方法区主要存放的是 Class，而堆中主要存放的是实例化的对象。

- 方法区（Method Area）与Java堆一样，是各个线程共享的内存区域
- 方法区在JVM启动的时候被创建，并且它的实际的物理内存空间中和Java堆区一样都可以是不连续的
- 方法区的大小，跟堆空间一样，可以选择固定大小或者可扩展
- 方法区的大小决定了系统可以保存多少个类，如果系统定义了太多的类，导致方法区溢出，虚拟机同样会抛出内存溢出错误：java.lang.OutofMemoryError：PermGen space 或者 java.lang.OutOfMemoryError：Metaspace
- 加载大量的第三方的jar包
- Tomcat部署的工程过多（30~50个）
- 大量动态的生成反射类

关闭JVM就会释放这个区域的内存。



## 5.2 方法区组成

组成如图所示。

![image-20210611183029019](https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155800.png)

**注意：**

- 在jdk7及以前，习惯上把方法区，称为永久代。jdk8开始，使用元空间取代了永久代。
- jdk1.8把方法区的`StringTable`转移到了堆中。



## 5.3 方法区内存溢出

**设置方法区大小**

- **JDK7及以前**
  1. 通过`-XX:Permsize`来设置永久代初始分配空间。默认值是20.75M。
  2. `-XX:MaxPermsize`来设定永久代最大可分配空间。32位机器默认是64M，64位机器模式是82M。
  3. 当JVM加载的类信息容量超过了这个值，会报异常`OutofMemoryError:PermGen space`。

- **JDK8以后**

  元数据区大小可以使用以下两个参数指定

  ```
  -XX:MetaspaceSize
  -XX:MaxMetaspaceSize
  ```

  默认值依赖于平台。windows下，`-XX:MetaspaceSize`是21M，`-XX:MaxMetaspaceSize`的值是-1，即没有限制。

**方法区内存溢出示例代码**

```java
package cn.itcast.jvm.t1.metaspace;

import jdk.internal.org.objectweb.asm.ClassWriter;
import jdk.internal.org.objectweb.asm.Opcodes;

/**
 * 演示元空间内存溢出 java.lang.OutOfMemoryError: Metaspace
 * -XX:MaxMetaspaceSize=8m
 */
public class Demo1_8 extends ClassLoader { // 可以用来加载类的二进制字节码
    public static void main(String[] args) {
        int j = 0;
        try {
            Demo1_8 test = new Demo1_8();
            for (int i = 0; i < 10000; i++, j++) {
                // ClassWriter 作用是生成类的二进制字节码
                ClassWriter cw = new ClassWriter(0);
                // 版本号， public， 类名, 包名, 父类， 接口
                cw.visit(Opcodes.V1_8, Opcodes.ACC_PUBLIC, "Class" + i, null, "java/lang/Object", null);
                // 返回 byte[]
                byte[] code = cw.toByteArray();
                // 执行了类的加载
                test.defineClass("Class" + i, code, 0, code.length); // Class 对象
            }
        } finally {
            System.out.println(j);
        }
    }
}

```



**如何解决这些OOM**

- 要解决ooM异常或heap space的异常，一般的手段是首先通过内存映像分析工具（如jvirsualvm）对dump出来的堆转储快照进行分析，重点是确认内存中的对象是否是必要的，也就是要先分清楚到底是出现了内存泄漏（Memory Leak）还是内存溢出（Memory Overflow）。
  - 内存泄漏就是 有大量的引用指向某些对象，但是这些对象以后不会使用了，但是因为它们还和GC ROOT有关联，所以导致以后这些对象也不会被回收，这就是内存泄漏的问题。
- 如果是内存泄漏，可进一步通过工具查看泄漏对象到GC Roots的引用链。于是就能找到泄漏对象是通过怎样的路径与GCRoots相关联并导致垃圾收集器无法自动回收它们的。掌握了泄漏对象的类型信息，以及GCRoots引用链的信息，就可以比较准确地定位出泄漏代码的位置。
- 如果不存在内存泄漏，换句话说就是内存中的对象确实都还必须存活着，那就应当检查虚拟机的堆参数（-Xmx与-Xms），与机器物理内存对比看是否还可以调大，从代码上检查是否存在某些对象生命周期过长、持有状态时间过长的情况，尝试减少程序运行期的内存消耗。

**容易OOM的场景**

- Spring
- MyBatis

## 5.4 常量池

**常量池**：就是一张表，虚拟机指令根据这张常量表找到要执行的类名、方法名、参数类型、字面量等信息！

**通过反编译来查看类的信息**

- 获得对应的class文件

  - 在JDK对应的bin目录下，输入cmd，也可以在IDEA控制台输入命令！

    ![image-20210611184958178](https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155801.png)

  - 输入 **javac 对应类的绝对路径**

    ```
    F:\JAVA\JDK8.0\bin>javac F:\Thread_study\src\com\nyima\JVM\day01\Main.java
    ```

    输入完成后，对应的目录下就会出现类的.class文件。

    

- 在控制台输入 `javap -v`类的绝对路径，然后能在控制台看到反编译以后类的信息了。

  ```
  javap -v F:\Thread_study\src\com\nyima\JVM\day01\Main.class
  ```

   

- 类的基本信息

  <img src="https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155802.png" alt="image-20210611185155476" style="zoom:67%;" />



- 常量池

  <img src="https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155803.png" alt="image-20210611185211296" style="zoom:60%;" />

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155804.png" alt="image-20210611185216463" style="zoom:59%;" />



- 虚拟机中执行编译的方法（框内的是真正编译执行的内容，**#号的内容需要在常量池中查找**）



## 5.5 运行时常量池

**运行时常量池**：常量池是 *.class 文件中的，当该类被加载，它的常量池信息就会放入运行时常量池，并把里面的符号地址变为真实地址（即#1这种符号）！

- 运行时常量池是方法区的一部分
- 常量池表是class文件的一部分，用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后存放到方法区的运行时常量池中
- 在加载类和接口到虚拟机后，就会创建对应的运行时常量池
- JVM为每个已加载的类型都维护一个常量池，池中的数据像数组项一样，通过索引访问
- 运行时常量池，相对于class文件常量池的另一个重要特征是：具备动态性（例如：`String.intern`可以将字符串也放入运行时常量池）
- 当创建类或接口的运行时常量池，如果构造运行时常量池所需的内存空间超过了方法区所能提供的最大值。则JVM会抛出OOM异常
- 这里注意，常量池数量为N，则索引为1到N-1，



## 5.6 StringTable 特性

[文章引荐](https://zhuanlan.zhihu.com/p/260939453)

**特点：**

- 常量池中的字符串仅是符号，`只有在被用到时才会被转为对象`
- 利用串池的机制，来避免重复创建字符串对象
- `字符串变量`拼接的原理是StringBuilder
- `字符串常量`拼接的原理是编译器优化
- 可以使用intern方法，主动将常量池中没有的字符串对象放入池中
- `注意`：无论是串池还是堆里面的字符串，都是对象

**示例代码**

```java
// StringTable [ "a", "b" ,"ab" ]  hashtable 结构，不能扩容
public class Demo1_22 {
    // 常量池中的信息，都会被加载到运行时常量池中， 这时 a b ab 都是常量池中的符号，还没有变为 java 字符串对象
    // ldc #2 会把 a 符号变为 "a" 字符串对象
    // ldc #3 会把 b 符号变为 "b" 字符串对象
    // ldc #4 会把 ab 符号变为 "ab" 字符串对象

    public static void main(String[] args) {
        String s1 = "a"; // 懒惰的
        String s2 = "b";
        String s3 = "ab";
        String s4 = s1 + s2; // new StringBuilder().append("a").append("b").toString()  new String("ab")
        String s5 = "a" + "b";  // javac 在编译期间的优化，结果已经在编译期确定为ab

        System.out.println(s3 == s5);

    }
}

```

常量池中的信息，都会被加载到运行时常量池中 . 但这时a 、b 、ab 仅是常量池中的符号，**还没有成为java字符串**。

```
0: ldc #2  // String a  
2: astore_1 
3: ldc #3  // String b  
5: astore_2 
6: ldc #4  // String ab  
8: astore_3 
9: return
```

1. 执行到 ldc #2 时，会把符号 a 变为 “a” 字符串对象，**并放入串池中**（hashtable结构 不可扩容）
2. 当执行到 ldc #3 时，会把符号 b 变为 “b” 字符串对象，并放入串池中
3. 当执行到 ldc #4 时，会把符号 ab 变为 “ab” 字符串对象，并放入串池中
4. 最终**StringTable [“a”, “b”, “ab”]**

**注意**：字符串对象的创建都是**懒惰的**，只有当运行到那一行字符串且在串池中不存在的时候（如 ldc #2）时，该字符串才会被创建并放入串池中。

**字符串延迟加载示例代码**

```java
/**
  * 创建完的字符串不会再创建出来
  */
public class TestString(){
    public static void main(String[] args){
        int x = args.length;
        System.out.println();//字符串个数2275

        System.out.println("1");
        System.out.println("2");
        System.out.println("3");
        System.out.println("4");
        System.out.println("5");
        System.out.println("6");
        System.out.println("7");
        System.out.println("8");
        System.out.println("9");
        System.out.println("0");
        System.out.println("1");
        System.out.println("2");
        System.out.println("3");
        System.out.println("4");
        System.out.println("5");
        System.out.println("6");
        System.out.println("7");
        System.out.println("8");
        System.out.println("9");
        System.out.println("0");
        System.out.println(x);  //字符串个数2285
    }
}
```



## 5.7 StringTable 位置

**jdk1.6存在于常量池中，jdk1.8存在于堆中。**

**示例代码：**

```java
import java.util.ArrayList;
import java.util.List;

/**
 * 演示 StringTable 位置
 * 在jdk8下设置 -Xmx10m -XX:-UseGCOverheadLimit
 * 在jdk6下设置 -XX:MaxPermSize=10m
 */
public class Demo1_6 {

    public static void main(String[] args) throws InterruptedException {
        List<String> list = new ArrayList<String>();
        int i = 0;
        try {
            for (int j = 0; j < 260000; j++) {
                list.add(String.valueOf(j).intern());
                i++;
            }
        } catch (Throwable e) {
            e.printStackTrace();
        } finally {
            System.out.println(i);
        }
    }
}

```

参数说明:

```
-XX:MaxPermSize=10m //设置永久代最大值
-Xmx10m -XX:-UseGCOverheadLimit //关闭开关，演示堆内存溢出
```



## 5.8 StringTable垃圾回收调优

因为StringTable是由HashTable实现的，所以可以**适当增加HashTable桶的个数**，来减少字符串放入串池所需要的时间。

```
-XX:StringTableSize=xxxx
```

考虑是否需要将字符串对象入池, 可以通过**intern方法减少重复入池** 。



***

# 6、直接内存

## 6.1 定义

- **不是虚拟机运行时数据区的一部分**，也不是《Java虚拟机规范》中定义的内存区域
- 直接内存是在Java堆外的、直接向系统申请的内存区间
- 来源于NIO，通过存在堆中的DirectByteBuffer操作Native内存
- 直接内存大小可以通过`MaxDirectMemorySize`设置
- 如果不指定，默认与堆的最大值`-Xmx`参数值一致

通常，访问直接内存的速度会优于Java堆。即读写性能高：

1. 因此出于性能考虑，读写频繁的场合可能会考虑使用直接内存
2. java的NIO库允许Java程序使用直接内存，用于数据缓冲区
3. 也可能导致OOM异常
   - 直接内存在堆外，所以大小不受限于-Xmx指定的最大堆大小
   - 但是系统内存是有限的，Java堆和直接内存的总和依然受限于操作系统能给出的最大内存

使用下列代码，直接分配本地内存空间：

```
int BUFFER = 1024*1024*1024; // 1GB
ByteBuffer byteBuffer = ByteBuffer.allocateDirect(BUFFER);
```

**缺点**

- 分配回收成本较高
- 不受JVM内存回收管理



## 6.2 基本使用

**文件读写流程**

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155805.png" alt="image-20210612000941540" style="zoom: 47%;" />

**使用了DirectBuffer**

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/JVM/202207121155806.png" alt="image-20210612001323279" style="zoom:47%;" />

直接内存是操作系统和Java代码**都可以访问的一块区域**，无需将代码从系统内存复制到Java堆内存，从而提高了效率。



## 6.3 内存溢出

**示例代码**

```java
import java.nio.ByteBuffer;
import java.util.ArrayList;
import java.util.List;


/**
 * 演示直接内存溢出
 */
public class Demo1_10 {
    static int _100Mb = 1024 * 1024 * 100;

    public static void main(String[] args) {
        List<ByteBuffer> list = new ArrayList<>();
        int i = 0;
        try {
            while (true) {
                ByteBuffer byteBuffer = ByteBuffer.allocateDirect(_100Mb);
                list.add(byteBuffer);
                i++;
            }
        } finally {
            System.out.println(i);
        }
        // 方法区是jvm规范， jdk6 中对方法区的实现称为永久代
        //                  jdk8 对方法区的实现称为元空间
    }
}
```



## 6.4 释放原理

直接内存的回收不是通过JVM的垃圾回收来释放的，而是通过**unsafe.freeMemory**来手动释放。

- 通过ByteBuffer申请1M的直接内存：

  ```
  ByteBuffer byteBuffer = ByteBuffer.allocateDirect(_1M);
  ```

- 但JVM并不能回收直接内存中的内容，它是如何实现回收的呢？

  **allocateDirect的实现**

  ```java
  public static ByteBuffer allocateDirect(int capacity) {     
  	return  new DirectByteBuffer(capacity); 
  }
  ```

  **DirectByteBuffer类**

  ```java
  DirectByteBuffer(int cap) { // package-private  
      super(-1, 0, cap, cap); 
      boolean pa = VM.isDirectMemoryPageAligned(); 
      int ps = Bits.pageSize(); 
      long size = Math.max(1L, (long)cap + (pa ? ps : 0));     
      Bits.reserveMemory(size, cap); 
  
      long base = 0; 
      try { 
      	base = unsafe.allocateMemory(size); //申请内存 
      } catch (OutOfMemoryError x) { 
          Bits.unreserveMemory(size, cap); 
      	throw x; 
      }
      unsafe.setMemory(base, size, (byte) 0); 
      if (pa && (base % ps != 0)) { 
      	// Round up to page boundary address = base + ps - (base & (ps - 1)); 
      } else { 
      	address = base; 
      } 
      cleaner = Cleaner.create(this, new Deallocator(base, size, cap)); //通过虚引用，来实现直接内存的释放，this为虚引用的实际对象 
      att = null; 
  }
  ```

  这里调用了一个Cleaner的create方法，且后台线程还会对虚引用的对象监测，如果虚引用的实际对象（这里是DirectByteBuffer）被回收以后，就会调用Cleaner的clean方法，来清除直接内存中占用的内存。

  ```java
  public void clean() { 
      if (remove(this)) { 
          try { this.thunk.run(); //调用run方法 
      } catch (final Throwable var2) { 
        AccessController.doPrivileged(new PrivilegedAction<Void>() { 
      	public Void run() { 
       	 if (System.err != null) { 
       	   (new Error("Cleaner terminated abnormally", var2)).printStackTrace(); 
        } 
       	 System.exit(1); 
       	 return  null; 
         } 
       }); 
      }
  ```

  对应对象的run方法：

  ```java
  public void run() { 
      if (address == 0) { 
      // Paranoia  
      return; 
   }
   	unsafe.freeMemory(address); //释放直接内存中占用的内存    address = 0; 
  	Bits.unreserveMemory(size, capacity); 
  }
  
  
  ```

  

  **直接内存的回收机制总结**

  - 使用了Unsafe类来完成直接内存的分配回收，回收需要主动调用freeMemory方法。
  - ByteBuffer的实现内部使用了Cleaner（虚引用）来检测ByteBuffer。一旦ByteBuffer被垃圾回收，那么会SReferenceHandler来调用Cleaner的clean方法调用freeMemory来释放内存。

  

  

  

  

  

  

  
