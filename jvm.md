# JVM
JVM是可运行java代码的假想计算机，包括一套字节码指令集、一组寄存器、一个栈、一个垃圾回收、堆及一个存储方法域。JVM运行在操作系统上与硬件没有直接交互。
Java通过编译器产生.Class文件（字节码文件），而字节码文件通过Jaca虚拟机中的解释器编译成特定机器上的机器码。
我们运行一个Java程序就会实例化一个虚拟机，程序退出虚拟机实例消亡，虚拟机实例之间不能共享数据。实例化的虚拟机表现为一个进程该进程在内存中的结构如下：
![Image text](https://github.com/lizhanwei/design/blob/master/1574231740784.jpg?raw=true)
* 图中绿色部分是线程共享的数据区域，黄色部分为线程私有的数据区域。
* 程序计数器是唯一不会OOM的地方
* HotSpot VM把GC分代收集扩展到方法区，该区针对常量池的回收和类型的卸载（收益很小）
* Java8中,永久代被移除，被一个称为"元数据区"的区域所取代，该区域并不在虚拟机中，而是使用本地内存。类的元数据放入Native Memory，字符串池和类的静态
变量放入java堆中。这样加载多少类就有系统实际可用空间控制
* 服务器上只部署jit就可以，jdk并不必须

JVM 后台系统线程有下面几个：

* 虚拟机线程。这个线程等待 JVM 到达安全点操作出现。这些操作必须要在独立的线程里执行，因为当 堆修改无法进行时，线程都需要 JVM 位于安全点。这些操作的类型有:stop-the-world 垃圾回收、线程栈 dump、线程暂停、线程偏向锁(biased locking)解除。
* 周期性任务线程。这线程负责定时器事件(也就是中断)，用来调度周期性操作的执行。
* GC线程。这些线程支持 JVM 中不同的垃圾回收活动。
* 编译器线程。这些线程在运行时将字节码动态编译成本地平台相关的机器码。
* 信号分发线程。这个线程接收发送到 JVM 的信号并调用适当的 JVM 方法处理。

# 垃圾回收
## 如何确定垃圾
### 引用计数法
即一个对象如果没有任何与之关联的引用，也即他们的引用计数都不为0，则说明对象不太可能被用到。那么这个对象就是可回收对象。
* 引用计数算法是对象记录自己被多少程序引用，引用计数为零的对象将被清除。
* 计数器表示的是有多少程序引用了这个对象（被引用数）。计数器是无符号整数
程序在生成新对象的时候会调用 new_obj()函数
```java
func new_obj(size){
    obj = pickup_chunk(size, $free_list)
    
    if(obj == NULL)
        allocation_fail()
    else
        obj.ref_cnt = 1  // 新对象第一只被分配是引用数为1
        return obj
}
```
这里pickup_chunk放回null，分配就失败了；ref_cnt代表了obj的计数器。
举个例子

![Image text](https://github.com/lizhanwei/design/blob/master/1652dbec039c06f3.png)
上图A指向B，第二步A指向了C。可以看到通过更新，B 的计数器值变为了0，因此B被回收（连接到空闲链表），C的计数器值由1变成了2。
引用计数法不能解决循环引用的问题，例如：
```java
class Person{
    string name;
    Person friend;
}

Person li = new Person();
li.name = 'li';

Person yu = new Person();
yu.name = 'yu';

li.friend = yu;
yu.friend = li;
```
两个对象相互引用，所以各个对象的计数器都为1，且这些对象没有被其他对象引用。所以计数器最小值也为1，不可能为0。

### 可达性分析法

为了解决循环引用的问题，Java使用了可达性分析法。通过一系列的GC Roots对象作为起点搜索，如果GC Roots和一个对象没有可达路径则称该对象不可达。不可达对象
经过至少两次标记过程仍然是可回收对象，则面临回收。

可以作为GC Roots的对象包括以下几点

* 虚拟机栈（栈帧中的本地变量表）中引用的对象
* 方法区中的类静态属性引用的对象或者常量引用的对象
* 本地方法栈中JNI（就是native方法）引用的对象

可达性分析算法概要：

HotSpot首先需要枚举所有的GC Roots根节点，虚拟机栈的空间不大，遍历一次的时间或许可以接受，但是方法区的空间很可能就有数百兆，遍历一次需要很久。更加关键的是，当我们遍历所有GC Roots根节点时，我们需要暂停所有用户线程，因为我们需要一个此时此刻的”虚拟机快照”，找到此时此刻的可达性分析关系图。基于这种情况，HotSpot实现了一种叫做OopMap的数据结构，存储GC Roots对象，同时并不是每个指令都会对OopMap进行修改，这样OopMap很庞大，这里Hotspot引入了安全点，safePoint，只会在Safe Point处记录GC Roots信息。

如何让用户线程在接近SafePoint的时候完成停顿？

有两种方案，一种抢先式中断（Preemptive Suspension）和主动式中断（Voluntary Suspension），其中抢先式中断不需要线程的执行代码主动去配合，在GC发生时，首先把所有线程全部中断，如果发现有线程中断的地方不在安全点上，就恢复线程，让它“跑”到安全点上。这种方式比较粗暴，现在基本没有采用。现在主要采用的是主动式中断：当GC需要中断线程的时候，不直接对线程操作，仅仅简单地设置一个标志，各个线程执行时主动去轮询这个标志，发现中断标志为真时就自己中断挂起。

为什么要两次标记？

在对象被回收之前会调用对象的finalize()方法，如果当开发者重写了该方法将该对象重新加入关系网中。此时该对象仍然可达但是已经被标记了。因此虚拟机采用了两次标记，即第一次标记不在“关系网”中的对象。第二次的话就要先判断该对象有没有实现finalize()方法了，如果没有实现就直接判断该对象可回收；如果实现了就会先放在一个队列中，并由虚拟机建立的一个低优先级的线程去执行它，随后就会进行第二次的小规模标记，在这次被标记的对象就会真正的被回收了。

CMS两次标记时为什么要stop the world?

当虚拟机完成两次标记后，便确认了可以回收的对象。但是，垃圾回收并不会阻塞我们程序的线程，他是与当前程序并发执行的。所以问题就出在这里，当GC线程标记好了一个对象的时候，此时我们程序的线程又将该对象重新加入了“关系网”中，当执行二次标记的时候，该对象也没有重写finalize()方法，因此回收的时候就会回收这个不该回收的对象。 

四种对象引用方式

强引用
```java
Object obj = new Object();
```

软引用

软引用的生命周期比强引用短一些。软引用是通过SoftReference类实现的。
```java
Object obj = new Object();
SoftReference softObj = new SoftReference(obj);
obj = null； //去除强引用
```
这样就是一个简单的软引用使用方法。通过get()方法获取对象。当JVM认为内存空间不足时，就回去试图回收软引用指向的对象，也就是说在JVM抛出OutOfMemoryError之前，会去清理软引用对象。软引用可以与引用队列(ReferenceQueue)联合使用
```java
Object obj = new Object();
ReferenceQueue queue = new ReferenceQueue();
SoftReference softObj = new SoftReference(obj,queue);
obj = null； //去除强引用
```
软引用一般用来实现内存敏感的缓存，如果有空闲内存就可以保留缓存，当内存不足时就清理掉，这样就保证使用缓存的同时不会耗尽内存。例如图片缓存框架中缓存图片就是通过软引用的。

弱引用

弱引用是通过WeakReference类实现的，它的生命周期比软引用还要短,也是通过get()方法获取对象。
```java
 Object obj = new Object();
 WeakReference<Object> weakObj = new WeakReference<Object>(obj);
 obj = null； //去除强引用
```
在GC的时候，不管内存空间足不足都会回收这个对象，同样也可以配合ReferenceQueue使用，也同样适用于内存敏感的缓存。ThreadLocal中的key就用到了弱引用。

虚引用

通过PhantomReference类实现的。任何时候可能被GC回收,就像没有引用一样。
```java

Object obj = new Object();
ReferenceQueue queue = new ReferenceQueue();
PhantomReference<Object> phantomObj = new PhantomReference<Object>(obj , queue);
obj = null； //去除强引用

```
虚引用仅仅只是提供了一种确保对象被finalize以后来做某些事情的机制。比如说这个对象被回收之后发一个系统通知之类的。虚引用是必须配合ReferenceQueue 使用的，具体使用方法和上面提到软引用的一样。主要用来跟踪对象被垃圾回收的活动。

## 垃圾收集算法

### 复制算法
按内存容量将内存划分为等大小的两块。每次只使用其中一块，当这一块内存满后将尚存活的对象复制到另一块上去，把已使用的内存清掉。

### 标记清除算法（Mark-Sweep）
分为两个阶段，标注和清除。标记阶段标记出所有需要回收的对象，清除阶段回收被标记的对象所占用的空间。

### 标记整理算法
标记阶段和Mark-Sweep算法相同，标记后不是清理对象，而是将存活对象移向内存的一端。然后清除端边界外的对象

## GC算法（分代，分区收集算法）
当前主流 VM 垃圾收集都采用”分代收集”(Generational Collection)算法, 这种算法会根据 对象存活周期的不同将内存划分为几块, 如 JVM 中的 新生代、老年代、永久代，这样就可以根据 各年代特点分别采用最适当的 GC 算法。
### 分代收集算法
分代收集法是目前大部分 JVM 所采用的方法，其核心思想是根据对象存活的不同生命周期将内存 划分为不同的域，一般情况下将 GC 堆划分为老生代(Tenured/Old Generation)和新生代(Young Generation)。老生代的特点是每次垃圾回收时只有少量对象需要被回收，新生代的特点是每次垃 圾回收时都有大量垃圾需要被回收，因此可以根据不同区域选择不同的算法。

#### 新生代与复制算法
* 首先，把 Eden 和 ServivorFrom 区域中存活的对象复制到 ServicorTo 区域(如果有对象的年 龄以及达到了老年的标准，则赋值到老年代区)，同时把这些对象的年龄+1(如果 ServicorTo 不 够位置了就放到老年区);
* 然后，清空 Eden 和 ServicorFrom 中的对象
* 最后，ServicorTo 和 ServicorFrom 互换，原 ServicorTo 成为下一次 GC 时的 ServicorFrom区。

#### 在老年代-标记整理算法
因为对象存活率高、没有额外空间对它进行分配担保, 就必须采用“标记—清理”或“标 记—整理”算法来进行回收, 不必进行内存复制, 且直接腾出空闲内存

### 几种垃圾收集器
java虚拟中针对新生代和年老代分别提供了多种不同的垃圾收集器，JDK1.6 中 Sun HotSpot 虚拟机的垃圾收集器如下
![Image text](https://github.com/lizhanwei/design/blob/master/201911191.png)
#### Serial 垃圾收集器(单线程、复制算法)
#### ParNew垃圾收集器(Serial+多线程)
#### Parallel Scavenge 收集器(多线程复制算法、高效)
#### SerialOld收集器(单线程标记整理算法)
#### ParallelOld收集器(多线程标记整理算法)
#### CMS收集器(多线程标记清除算法)
#### G1收集器

### 分区收集算法
分区算法则将整个堆空间划分为连续的不同小区间, 每个小区间独立使用, 独立回收. 这样做的好处是可以控制一次回收多少个小区间。根据目标停顿时间, 每次合理地回收若干个小区间(而不是
整个堆), 从而减少一次 GC 所产生的停顿。



