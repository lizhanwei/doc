## jps
jps可以显示当前运用的java进程以及相关参数。
java程序在启动以后会在java.io.tmpdir指定的目录下生成一个类似于hsperdata_user的文件夹，这个文件夹下有几个文件，名字就是java进程的pid。jps命令只是列出该目录下的文件名而已。（jps只能列出当前用户的java进程，由于目录权限，文件删除等原因jps可能会失效[不能列出所有java进程]）

-m 输出传递给main 方法的参数
-l 输出应用程序main class的完整package名 或者 应用程序的jar文件完整路径名
-v 输出传递给JVM的参数

## jinfo
可看系统启动的参数
jinfo -flags 6864

## jstack
jstack用于生成java虚拟机当前时刻的线程快照。线程快照是当前java虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等.

前提：每个对象都有一个管程 (英语：Monitors，也称为监视器)用来同步状态

> 1：所有需要同步的对象进入EntrtSet(进入区)
2：通过synchronized来获取对象锁，获取到锁进入拥有区（the owner）没有获取到则去进入区
3：拥有区的对象需要等待其他资源调用wait方法进入等待区（WaitSet）释放锁，在等待区等待被唤醒

对应的jstack为
> locked <地址> 目标：使用synchronized申请对象锁成功,监视器的拥有者。
waiting to lock <地址> 目标：使用synchronized申请对象锁未成功,在迚入区等待。
waiting on <地址> 目标：使用synchronized申请对象锁成功后,释放锁幵在等待区等待。
parking to wait for <地址> 目标

常用命令
> -F当’jstack [-l] pid’没有相应的时候强制打印栈信息 
-l长列表. 打印关于锁的附加信息,例如属于java.util.concurrent的ownable synchronizers列表. 
-m打印java和native c/c++框架的所有栈信息. 
-h | -help打印帮助信息

```java
    static Object o1 = new Object();
    static Object o2 = new Object();

    class DeadLockclass implements Runnable {
        public boolean falg;// 控制线程

        DeadLockclass(boolean falg) {
            this.falg = falg;
        }

        @Override
        public void run() {
            /**
             * 如果falg的值为true则调用t1线程
             */
            if (falg) {
                while (true) {
                    synchronized (o1) {
                        System.out.println("o1 " + Thread.currentThread().getName());
                        synchronized (o2) {
                            System.out.println("o2 " + Thread.currentThread().getName());
                        }
                    }
                }
            }
            /**
             * 如果falg的值为false则调用t2线程
             */
            else {
                while (true) {
                    synchronized (o2) {
                        System.out.println("o2 " + Thread.currentThread().getName());
                        synchronized (o1) {
                            System.out.println("o1 " + Thread.currentThread().getName());
                        }
                    }
                }
            }
        }
    }

    public void Test() {
        Thread thread1 = new Thread(new DeadLockclass(true));
        Thread thread2 = new Thread(new DeadLockclass(false));
        thread1.start();
        thread2.start();
    }

    public static void main(String[] args) throws IOException {
        new TestDeadLock().Test();
        System.in.read();
    }
```
jstack
![Jstack 死锁](https://github.com/lizhanwei/design/blob/master/WeChatad9ab0b863c842778f6aab4039827bee.png?raw=true)

## jmap
打印指定Java进程(或核心文件、远程调试服务器)的共享对象内存映射或堆内存细节。该命令用于查内存泄漏。
![Jmap 内存](https://github.com/lizhanwei/design/blob/master/WeChat3a24483dec582ebcdbec31bd10a2dab0.png?raw=true)
>option 选项参数是互斥的(不可同时使用)。想要使用选项参数，直接跟在命令名称后即可。
pid 需要打印配置信息的进程ID。该进程必须是一个Java进程。想要获取运行的Java进程列表，你可以使用jps。
executable 产生核心dump的Java可执行文件。
core 需要打印配置信息的核心文件。
remote-hostname-or-IP 远程调试服务器的(请查看jsadebugd)主机名或IP地址。
server-id 可选的唯一id，如果相同的远程主机上运行了多台调试服务器，用此选项参数标识服务器。

><no option> 如果使用不带选项参数的jmap打印共享对象映射，将会打印目标虚拟机中加载的每个共享对象的起始地址、映射大小以及共享对象文件的路径全称。这与Solaris的pmap工具比较相似。
-dump:[live,]format=b,file=<filename> 以hprof二进制格式转储Java堆到指定filename的文件中。live子选项是可选的。如果指定了live子选项，堆中只有活动的对象会被转储。想要浏览heap dump，你可以使用jhat(Java堆分析工具)读取生成的文件。
-finalizerinfo 打印等待终结的对象信息。
-heap 打印一个堆的摘要信息，包括使用的GC算法、堆配置信息和generation wise heap usage。
-histo[:live] 打印堆的柱状图。其中包括每个Java类、对象数量、内存大小(单位：字节)、完全限定的类名。打印的虚拟机内部的类名称将会带有一个’*’前缀。如果指定了live子选项，则只计算活动的对象。
-permstat 打印Java堆内存的永久保存区域的类加载器的智能统计信息。对于每个类加载器而言，它的名称、活跃度、地址、父类加载器、它所加载的类的数量和大小都会被打印。此外，包含的字符串数量和大小也会被打印。
-F 强制模式。如果指定的pid没有响应，请使用jmap -dump或jmap -histo选项。此模式下，不支持live子选项。
-h 打印帮助信息。
-help 打印帮助信息。
-J<flag> 指定传递给运行jmap的JVM的参数。

## jmap -heap 31846
``` java
Attaching to process ID 31846, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 24.71-b01

using thread-local object allocation.
Parallel GC with 4 thread(s)//GC 方式

Heap Configuration: //堆内存初始化配置
   MinHeapFreeRatio = 0 //对应jvm启动参数-XX:MinHeapFreeRatio设置JVM堆最小空闲比率(default 40)
   MaxHeapFreeRatio = 100 //对应jvm启动参数 -XX:MaxHeapFreeRatio设置JVM堆最大空闲比率(default 70)
   MaxHeapSize      = 2082471936 (1986.0MB) //对应jvm启动参数-XX:MaxHeapSize=设置JVM堆的最大大小
   NewSize          = 1310720 (1.25MB)//对应jvm启动参数-XX:NewSize=设置JVM堆的‘新生代’的默认大小
   MaxNewSize       = 17592186044415 MB//对应jvm启动参数-XX:MaxNewSize=设置JVM堆的‘新生代’的最大大小
   OldSize          = 5439488 (5.1875MB)//对应jvm启动参数-XX:OldSize=<value>:设置JVM堆的‘老生代’的大小
   NewRatio         = 2 //对应jvm启动参数-XX:NewRatio=:‘新生代’和‘老生代’的大小比率
   SurvivorRatio    = 8 //对应jvm启动参数-XX:SurvivorRatio=设置年轻代中Eden区与Survivor区的大小比值 
   PermSize         = 21757952 (20.75MB)  //对应jvm启动参数-XX:PermSize=<value>:设置JVM堆的‘永生代’的初始大小
   MaxPermSize      = 85983232 (82.0MB)//对应jvm启动参数-XX:MaxPermSize=<value>:设置JVM堆的‘永生代’的最大大小
   G1HeapRegionSize = 0 (0.0MB)

Heap Usage://堆内存使用情况
PS Young Generation
Eden Space://Eden区内存分布
   capacity = 33030144 (31.5MB)//Eden区总容量
   used     = 1524040 (1.4534378051757812MB)  //Eden区已使用
   free     = 31506104 (30.04656219482422MB)  //Eden区剩余容量
   4.614088270399305% used //Eden区使用比率
From Space:  //其中一个Survivor区的内存分布
   capacity = 5242880 (5.0MB)
   used     = 0 (0.0MB)
   free     = 5242880 (5.0MB)
   0.0% used
To Space:  //另一个Survivor区的内存分布
   capacity = 5242880 (5.0MB)
   used     = 0 (0.0MB)
   free     = 5242880 (5.0MB)
   0.0% used
PS Old Generation //当前的Old区内存分布
   capacity = 86507520 (82.5MB)
   used     = 0 (0.0MB)
   free     = 86507520 (82.5MB)
   0.0% used
PS Perm Generation//当前的 “永生代” 内存分布
   capacity = 22020096 (21.0MB)
   used     = 2496528 (2.3808746337890625MB)
   free     = 19523568 (18.619125366210938MB)
   11.337498256138392% used

670 interned Strings occupying 43720 bytes.

```

## jmap -histo 3331
对象数量及大小

>1.如果程序内存不足或者频繁GC，很有可能存在内存泄露情况，这时候就要借助Java堆Dump查看对象的情况。
2.要制作堆Dump可以直接使用jvm自带的jmap命令
3.可以先使用jmap -heap命令查看堆的使用情况，看一下各个堆空间的占用情况。
4.使用jmap -histo:[live]查看堆内存中的对象的情况。如果有大量对象在持续被引用，并没有被释放掉，那就产生了内存泄露，就要结合代码，把不用的对象释放掉。
5.也可以使用 jmap -dump:format=b,file=<fileName>命令将堆信息保存到一个文件中，再借助jhat命令查看详细内容
6.在内存出现泄露、溢出或者其它前提条件下，建议多dump几次内存，把内存文件进行编号归档，便于后续内存整理分析。

## jsata
jstat位于java的bin目录下，主要利用JVM内建的指令对Java应用程序的资源和性能进行实时的命令行的监控，包括了对Heap size和垃圾回收状况的监控。
### jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
> Option — 选项，我们一般使用 -gcutil 查看gc情况
vmid — VM的进程号，即当前运行的java进程号
interval– 间隔时间，单位为秒或者毫秒
count — 打印次数，如果缺省则打印无数次

### jstat –class <pid> : 显示加载class的数量，及所占空间等信息
>Loaded 装载的类的数量 Bytes 装载类所占用的字节数 Unloaded 卸载类的数量 Bytes 卸载类的字节数 Time 装载和卸载类所花费的时间。

### jstat -compiler <pid>显示VM实时编译的数量等信息
>Compiled 编译任务执行数量 Failed 编译任务执行失败数量 Invalid 编译任务执行失效数量 Time 编译任务消耗时间 FailedType 最后一个编译失败任务的类型 FailedMethod 最后一个编译失败任务所在的类及方法

### jstat -gc <pid>: 可以显示gc的信息，查看gc的次数，及时间
>S0C 年轻代中第一个survivor（幸存区）的容量 (字节) S1C 年轻代中第二个survivor（幸存区）的容量 (字节) S0U 年轻代中第一个survivor（幸存区）目前已使用空间 (字节) S1U 年轻代中第二个survivor（幸存区）目前已使用空间 (字节) EC 年轻代中Eden（伊甸园）的容量 (字节) EU 年轻代中Eden（伊甸园）目前已使用空间 (字节) OC Old代的容量 (字节) OU Old代目前已使用空间 (字节) PC Perm(持久代)的容量 (字节) PU Perm(持久代)目前已使用空间 (字节) YGC 从应用程序启动到采样时年轻代中gc次数 YGCT 从应用程序启动到采样时年轻代中gc所用时间(s) FGC 从应用程序启动到采样时old代(全gc)gc次数 FGCT 从应用程序启动到采样时old代(全gc)gc所用时间(s) GCT 从应用程序启动到采样时gc用的总时间(s)

### jstat -gccapacity <pid>:可以显示，VM内存中三代（young,old,perm）对象的使用和占用大小
>NGCMN 年轻代(young)中初始化(最小)的大小(字节) NGCMX 年轻代(young)的最大容量 (字节) NGC 年轻代(young)中当前的容量 (字节) S0C 年轻代中第一个survivor（幸存区）的容量 (字节) S1C 年轻代中第二个survivor（幸存区）的容量 (字节) EC 年轻代中Eden（伊甸园）的容量 (字节) OGCMN old代中初始化(最小)的大小 (字节) OGCMX old代的最大容量(字节) OGC old代当前新生成的容量 (字节) OC Old代的容量 (字节) PGCMN perm代中初始化(最小)的大小 (字节) PGCMX perm代的最大容量 (字节)
PGC perm代当前新生成的容量 (字节) PC Perm(持久代)的容量 (字节) YGC 从应用程序启动到采样时年轻代中gc次数 FGC 从应用程序启动到采样时old代(全gc)gc次数

### jstat -gcutil <pid>:统计gc信息
>S0 年轻代中第一个survivor（幸存区）已使用的占当前容量百分比 S1 年轻代中第二个survivor（幸存区）已使用的占当前容量百分比 E 年轻代中Eden（伊甸园）已使用的占当前容量百分比 O old代已使用的占当前容量百分比 P perm代已使用的占当前容量百分比 YGC 从应用程序启动到采样时年轻代中gc次数 YGCT 从应用程序启动到采样时年轻代中gc所用时间(s) FGC 从应用程序启动到采样时old代(全gc)gc次数 FGCT 从应用程序启动到采样时old代(全gc)gc所用时间(s) GCT 从应用程序启动到采样时gc用的总时间(s)

### jstat -gcnew <pid>:年轻代对象的信息
>S0C 年轻代中第一个survivor（幸存区）的容量 (字节) S1C 年轻代中第二个survivor（幸存区）的容量 (字节) S0U 年轻代中第一个survivor（幸存区）目前已使用空间 (字节) S1U 年轻代中第二个survivor（幸存区）目前已使用空间 (字节) TT 持有次数限制 MTT 最大持有次数限制 EC 年轻代中Eden（伊甸园）的容量 (字节) EU 年轻代中Eden（伊甸园）目前已使用空间 (字节) YGC 从应用程序启动到采样时年轻代中gc次数 YGCT 从应用程序启动到采样时年轻代中gc所用时间(s)

### jstat -gcnewcapacity<pid>: 年轻代对象的信息及其占用量
>NGCMN 年轻代(young)中初始化(最小)的大小(字节) NGCMX 年轻代(young)的最大容量 (字节) NGC 年轻代(young)中当前的容量 (字节) S0CMX 年轻代中第一个survivor（幸存区）的最大容量 (字节) S0C 年轻代中第一个survivor（幸存区）的容量 (字节) S1CMX 年轻代中第二个survivor（幸存区）的最大容量 (字节) S1C 年轻代中第二个survivor（幸存区）的容量 (字节) ECMX 年轻代中Eden（伊甸园）的最大容量 (字节) EC 年轻代中Eden（伊甸园）的容量 (字节) YGC 从应用程序启动到采样时年轻代中gc次数 FGC 从应用程序启动到采样时old代(全gc)gc次数

### jstat -gcold <pid>：old代对象的信息
>PC Perm(持久代)的容量 (字节) PU Perm(持久代)目前已使用空间 (字节) OC Old代的容量 (字节) OU Old代目前已使用空间 (字节) YGC 从应用程序启动到采样时年轻代中gc次数 FGC 从应用程序启动到采样时old代(全gc)gc次数 FGCT 从应用程序启动到采样时old代(全gc)gc所用时间(s) GCT 从应用程序启动到采样时gc用的总时间(s)

### jstat -gcoldcapacity <pid>: old代对象的信息及其占用量
>OGCMN old代中初始化(最小)的大小 (字节) OGCMX old代的最大容量(字节) OGC old代当前新生成的容量 (字节) OC Old代的容量 (字节) YGC 从应用程序启动到采样时年轻代中gc次数 FGC 从应用程序启动到采样时old代(全gc)gc次数 FGCT 从应用程序启动到采样时old代(全gc)gc所用时间(s) GCT 从应用程序启动到采样时gc用的总时间(s)

### jstat -gcpermcapacity<pid>: perm对象的信息及其占用量
>PGCMN perm代中初始化(最小)的大小 (字节) PGCMX perm代的最大容量 (字节)
PGC perm代当前新生成的容量 (字节) PC Perm(持久代)的容量 (字节) YGC 从应用程序启动到采样时年轻代中gc次数 FGC 从应用程序启动到采样时old代(全gc)gc次数 FGCT 从应用程序启动到采样时old代(全gc)gc所用时间(s) GCT 从应用程序启动到采样时gc用的总时间(s)

### jstat -printcompilation <pid>：当前VM执行的信息
>Compiled 编译任务的数目 Size 方法生成的字节码的大小 Type 编译类型 Method 类名和方法名用来标识编译的方法。类名使用/做为一个命名空间分隔符。方法名是给定类中的方法。上述格式是由-XX:+PrintComplation选项进行设置的

## btrace
线上问题排查工具。地址：https://github.com/btraceio/btrace
```java
public int sum() {
    int result = 0;
    for (int i = 0; i < 100; i++) {
        result += i * i;
    }
    return result;
}
public static void main(String[] args) throws IOException, InterruptedException {
    while (true) {
        Thread.currentThread().setName("计算");
        TestJPS util = new TestJPS();
        int result = util.sum();
        System.out.println(result);
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
        }
    }
}

@BTrace
public class TestJps {
    @OnMethod(
            clazz="com.alo7.pb.voiceEvalution.TestJPS",
            method="sum",
            location=@Location(Kind.RETURN)
    )

    public static void func(@Return int result) {
        println("trace: =======================");
        println(strcat("result:", str(result)));
        jstack();
    }
}

```
btrace pid TestJps.java

## Jhat
jhat命令解析会Java堆dump并启动一个web服务器，然后就可以在浏览器中查看堆的dump文件了。

## Javap
javap还可以查看java编译器为我们生成的字节码。通过它，可以对照源代码和字节码，从而了解很多编译器内部的工作。

## VM options
-XX:+TraceClassLoading //类到底是从哪个文件加载进来的
-XX:+HeapDumpOnOutOfMemoryError 
-XX:HeapDumpPath=/home/admin/logs/java.hprof //应用挂了输出dump文件

