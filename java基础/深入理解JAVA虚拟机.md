## 二、JAVA内存区域与内存溢出异常
### 2.2 运行时数据区
- 方法区
- 堆区
- 程序计数器
- 虚拟机栈
- 本地方法栈
>#### 程序计数器
- Program Counter Register是一块较小的内存空间,它可以看作是当前线程锁执行字节码的行号指示器
```text
由于Java虚拟机的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现的，
在任何一个确定的时刻，一个处理器（对于多核处理器来说是一个内核）都只会执行一条线
程中的指令。因此，为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立
的程序计数器，各条线程之间计数器互不影响，独立存储，我们称这类内存区域为“线程私
有”的内存
```
>#### JAVA虚拟机栈
- JAVA虚拟机栈也是线程私有的,生命周期与线程相同.
- 每个方法在执行时会创建一个栈帧,栈帧中用于存储局部变量表、操作数栈、动态链接、方法出口等信息
```text
栈帧的大小在编译的时候确定
```
- [ ] 栈帧的组成
- 局部变量表:存放编译器已知的各种数据类型(boolean、byte、char ..)、对象引用和returnAddress类型
```text
对象引用 不等同于对象本身,可能是一个指向对象起始地址的引用指针,也可能是指向一个代表对象的句柄或其他与此对象相关的位置
```
>#### 本地方法栈
- 本地方法栈:Native Method Stack与虚拟机栈所发挥的作用是非常相似的，它们之间
  的区别不过是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则为虚
  拟机使用到的Native方法服务

>#### JAVA堆
- 对大多数应用来说,JAVA堆是JAVA虚拟机所管理的内存中最大的一块,所有的对象实例以及数组都要在堆上分配
- [ ] JAVA堆内存的组成
- 年轻代
```text
Eden: 创建对象
From Survivor: S1 存放年轻带对象
To Survivor: S2 存放年轻带对象

由于GC(复制清除)算法的限制,S1/S2只会使用其中一个
```
- 老年代
```text
存放不易被GC的对象(默认年龄>15)
大对象(Eden区放不下的年轻对象)
```
>#### 方法区
- 方法区（Method Area）与Java堆一样，是各个线程共享的内存区域，它用于存储已被虚
  拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。虽然Java虚拟机规
  范把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做Non-Heap（非堆），目的应
  该是与Java堆区分开来

```text
方法区并不等同于永久带,只能说永久带是实现方法区的方式之一,由于永久带的内存溢出问题
JDK 1.8已经取消永久带,使用META SPACE来代替
```
- 运行时常量池:Runtime Constant Pool是方法区的一部分。Class文件中除了有类的版
  本、字段、方法、接口等描述信息外，还有一项信息是常量池（Constant Pool Table),用于
  存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常
  量池中存放。

>#### 直接内存
- 直接内存（Direct Memory）并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域。但是这部分内存也被频繁地使用，而且也可能导致OutOfMemoryError异常出现，所以我们放到这里一起讲解。

```text
在JDK 1.4中新加入了NIO（New Input/Output）类，引入了一种基于通道（Channel）与缓
冲区（Buffer）的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储
在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。
```
### 2.3 HotSpot虚拟机对象探秘
>#### 对象的创建
- 1.执行new指令时,首先去检查这个指令的参数能否在常量池中定位到一个类的符号引用,并检查这个符号引用代表的类是否已经被加载、解析和初始化过,如果没有,那必须先执行相应的类加载过程
- 2.为新生对象分配内存
```text
在类加载时,对象的分配空间大小就可以被确定
```
- 3.内存分配完成后,虚拟机需要将分配到的内存空间都初始化为初始值
- 4.对对象进行必要的设置,GC分代年龄,哈希值,元数据信息等
- 5.执行<init>方法初始化对象

>#### 对象的内存分布
- [ ]对象头
- 第一部分用于存储对象自身的运行时数据，
  如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等，这部分数据的长度在32位和64位的虚拟机（未开启压缩指针）中分别为32bit和 64bit，官方称它为“Mark Word”
- 对象头的另外一部分是类型指针，即对象指向它的类元数据的指针(元数据在方法区内)，虚拟机通过这个指针来确定这个对象是哪个类的实例
- [ ] 实例数据
- [ ] 对齐填充
- 对齐填充并不一定存在,也没有特别的含义,它仅仅起到占位符的作用,由于VM的自动内存管理系统要求对象起始地址必须是8的整数倍
>#### 对象的访问定位
- JAVA栈的本地变量表->句柄池->对象实例数据(堆区)和对象类型数据(方法区)

## 三、垃圾收集器与内存分配策略
### 3.2对象已死?
>#### 可达性分析算法
- 通过"GC ROOTS"对象作为起始点,判断对象是否存活
```text
GC ROOTS:
虚拟机栈(本地变量表)中引用的对象
方法区中静态属性引用的对象
方法区中常量引用的对象
本地方法栈中JNI(一般说的Native方法)引用的对象
```
- 可达性分析本质是通过图算法来确定对象是否存在引用

>#### 四种引用
- 强引用:直接引用,通过可达性分析算法来判断是否可回收
- 软应用:SoftReference描述一些还有用但并非必需的对象,当内存空间不足的时候会被GC回收
- 弱引用:WeekReference描述一些还有用但并非必需的对象,但比软引用更弱一些,当GC回收的时候就会回收只被弱引用的对象
- 虚引用:一个对象是否有虚引 用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例,为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知

>#### 对象必须死?
- 即使在可达性分析算法中不可达的对象，也并非是“非死不可”的，这时候它们暂时处 于“缓刑”阶段，要真正宣告一个对象死亡，至少要经历两次标记过程：如果对象在进行可达 性分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记并且进行一次筛选， 筛选的条件是此对象是否有必要执行finalize（）方法。当对象没有覆盖finalize（）方法，或 者finalize（）方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”

>#### 回收方法区
- 方法区中也是有垃圾回收存在的,但实际并非如此
- 方法区中回收废弃的常量和无用的类,但是其成本较高而且回收率较低
### 3.3垃圾回收算法
>#### 基础算法
- 标记-清除算法:标记碎片和清除,效率不高,并且会产生大量的碎片
- 复制算法:将内存等分为两份,每次只使用其中的一块,优点是效率高不会产生碎片,缺点是内存利用率低
- 标记-整理算法:标记后清除并整理内存
>#### 分代收集算法
- Hot Spot在年轻代使用复制算法,老年代使用标记-整理算法

### 3.5垃圾收集器
- [ ] 常见的垃圾收集器
- Serial收集器
- ParNew收集器
- Parallel Scavenge
- Serial Old收集器
- Parallel Old
- [ ] 并发和并行
- 并行（Parallel）：指多条垃圾收集线程并行工作，但此时用户线程仍然处于等待状态。
- 并发（Concurrent）：指用户线程与垃圾收集线程同时执行（但不一定是并行的，可能会交替执行），用户程序在继续运行，而垃圾收集程序运行于另一个CPU上。
>#### Serial收集器
- Serial单线程”的意义并不仅仅说明它只会使用一个CPU或一条收集线程去完成垃圾收集工作，更重要的是在它进行垃圾收集时，必须暂停其他所有的工作线程，直到它收集结束
- 在垃圾收集的过程中,需要停止用户线程,等待垃圾收集完成
>#### ParNew收集器
- 是Serial的多线程版本,是一个并行处理器,它通过多个线程来进行GC
```
ParNew收集器在两个CPU下的性能可能还不及Serial收集器(由于控制的开销)
但是ParNew是唯一可以与CMS共同工作的收集器
    CMS一般用于老年代,是真正的并发收集器
    ParNew与其配合,用于年轻代的GC
```
>#### Parallel Scavenge收集器
- Parallel Scavenge收集器的关注点在于吞吐量而非停顿时间
```text
吞吐量 = 用户代码时间/(运行用户代码时间+垃圾收集时间)
```
- Parallel Scavenge适用于不需要与用户交互但是可以尽快完成运算任务的程序
```text
-XX:GCTimeRatio:参数的值应当是一个大于0且小于100的整数，也就是垃圾收集时间占总时间的比率，相当于是吞吐量的倒数
-XX:MaxGCPauseMillis:MaxGCPauseMillis参数允许的值是一个大于0的毫秒数，收集器将尽可能地保证内存回收花费的时间不超过设定值,GC停顿时间缩短是以牺牲吞吐量和新生代空间来换取
的
```
>#### Serial Old收集器
- Serial收集器的老年代版本,使用“标记-整理”算法
- 作用
```text
1.JDK 1.5之前与Parallel Scavenge收集器搭配使用
2.作为CMS收集器的后备预案

需要说明一下，Parallel Scavenge收集器架构中本身有PS MarkSweep收集器来进行老年代
收集，并非直接使用了Serial Old收集器，但是这个PS MarkSweep收集器与Serial Old的实现
非常接近，所以在官方的许多资料中都是直接以Serial Old代替PS
```
>#### Parallel Old收集器
- JDK1.6之后才开始提供,是Parallel Scavenge收集器的老年代版本,使用多线程和"标记-整理"算法
```text
在此之前，新生代的Parallel Scavenge收集器一直处于比较尴尬的状态。原因是，如果新生代选择了Parallel Scavenge收集器，老年代除了Serial Old（PS MarkSweep）收集器外别无选择
```
>#### CMS收集器(Concurrent Mark Sweep)
- CMS收集器是一种以获取最短回收停顿时间为目标的收集器
- [ ] CMS收集器的工作步骤
- 初始标记(CMS initial mark)
```text
标记GC ROOTS能否直接关联到对象,速度很快
```
- 并发标记(CMS concurrent mark)
```text
并发标记阶段就是进行GC RootsTracing的过程
```
- 重新标记(CMS remark)
```text
修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录
这个阶段停顿时间一般会比初始标记阶段稍长一些,但远比并发标记的时间短
```
- 并发清除(CMS concurrent sweep)
```text
并发清除数据,不需要停顿
```
- 注意: 初始标记和重新标记仍然需要stop the world
- [ ] 优缺点
- 优点:并发收集、低停顿
- 缺点:
```text
1.对CPU资源敏感:虽然不会停顿用户线程,但也会占用CPU时间片
    CMS默认启动线程数为(CPU+3)/4,也就是CPU4个以上时,并发回收时垃圾收集线程不少于25%的CPU资源
2.CMS无法清理浮动垃圾:在并发清除阶段,用户线程还会产生垃圾
3.CMS使用的是标记-清除算法,收集结束后会有大量碎片产生,空间碎片过多时,将会给大对象分配带来大麻烦,往往会有老年代还有很大剩余空间,但是不得不提前触发一次Full GC
    CMS收集器提供了一个-XX：+UseCMSCompactAtFullCollection开关参数（默认就是开启的）
    用于在CMS收集器顶不住要进行FullGC时开启内存碎片的合并整理过程
    内存整理的过程是无法并发的,空间碎片问题没有了但停顿时间不得不变长
    
    虚拟机设计者还提供了另外一个参数-XX：CMSFullGCsBeforeCompaction
    这个参数是用于设置执行多少次不压缩的Full GC后，跟着来一次带压缩的
```
>#### G1收集器

### 3.6理解GC日志
- 每一种收集器的日志形式都是由它们自身实现决定的
- [ ] GC日志例子
- 日志文本
```text
33.125: [GC[DefNew:3324K->152K(3712K),0.0025925 secs]3324K->152K(11904K),0.0031680 secs]
100.667:[Full GC[Tenured:0K->210K(10240K),0.0149142secs]4603K->210K(19456K),[Perm:2999K->2999K(21248K)],0.0150007secs]
        [Times:user=0.01 sys=0.00,real=0.02secs]
```
- 日志解读
```text
1.33.125/100.667 表示GC发生的时间,这个数字含义是从Java虚拟机启动以来经过的秒数
2.GC和Full GC说明了这次垃圾收集的停顿类型,而不是用来区分新生代还是老年代GC
    如果含有"Full"字样,说明这次GC发生了Stop-The-World
3.DefNew/Tenured/Perm表示GC发生的区域,这里的显示与GC收集器是密切相关的
    Serial: DefNew=>新生代
    ParNew: ParNew=>新生代
    Parallel Scavenge: PSYongGen=>新生代
4.容量
    区域:GC前该内存区域已使用的容量->GC后该内存区域已使用的容量(该区域总容量)
5.容量2
    方括号外面 GC前Java堆已使用容量->GC后Java堆已使用容量(Java堆总容量)
```
### 3.7垃圾收集相关的常用参数
- UseSerialGC:使用Serial+Serial Old收集器
- UseParNewGC:ParNew+Serial Old收集器
- UseConcMarkSweepGC: ParNew + CMS + Serial Old收集器
- UseParallelGC: Parallel Scanvage + Serial Old收集器
- UseParallelOldGC:Parallel Scanvage+ Parallel Old收集器
- SurvivorRatio:新生代中Eden区与Survivor区的容量比,默认为8,代表Eden:Survivor=8:1
- PretenureSizeThreshold:直接进入老年代的对象的大小
- MaxTenuringThreshold:晋升到老年代的对象年龄,每个对象经过一次MinorGC后,年龄就+1
- UseAdaptiveSizePolicy:动态调整Java堆中各个区域的大小以及进入老年代的年龄
- ParallelGCThreads: GC时进行内存回收的线程数
- GCTimeRatio: GC时间总占比,默认为99,即允许1%的GC时间,仅在Parallel Scavenge收集器时生效
- MaxGCPauseMillis: 设置GC最大停顿时间,仅在使用Parallel Scanvage收集器时生效
- CMSInitiatingOccupancyFraction: CMS收集器在老年代空间被使用多少后触发垃圾收集,默认值为68%,仅在CMS生效
- UseCMSCompactAtFullCollection: 设置CMS收集器在完成垃圾收集后是否进行一次内存碎片整理
- CMSFullGCCsBeforeCompaction: CMS收集器在进行若干次垃圾收集后在启动一次内存碎片整理

### 3.8内存分配与回收策略
- 虚拟机提供了-XX：+PrintGCDetails这个收集器日志参数，告诉虚拟机在发生垃圾收集行为时打印内存回收日志，并且在进程退出的时候输出当前的内存各区域分配情况。在实际应用中，内存回收日志一般是打印到文件后通过日志工具进行分析
>#### 对象优先在Eden分配
- 大多数情况下，对象在新生代Eden区中分配。当Eden区没有足够空间进行分配时，虚拟机将发起一次Minor GC
>#### 大对象直接进入老年代
- 所谓的大对象是指,需要大量连续内存的Java对象,最典型的大对象就是那种很长的字符串以及数组
- 大对象对于虚拟机来说是坏消息,比遇到大对象更加坏的消息是遇到一个短暂生命周期的大对象.
>#### 长期存活的对象进入老年代
- 每个对象在进行一次Minor GC后年龄会+1,当超过MaxTurningThreshold后会进入老年代
- 为了能更好地适应不同程序的内存状况，虚拟机并不是永远地要求对象的年龄必须达到了MaxTenuringThreshold才能晋升老年代，如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄。

## 四、虚拟机性能监控与故障处理
### 常见命令
- jps: JVM Process Status Tool 显示指定系统内所有的HotSpot虚拟机进程
- jstat: JVM Statistics Monitoring Tool用于收集HotSpot虚拟机各个方面的运行数据
- jinfo: Configuration Info for Java 显示虚拟机配置信息
- jmap: Memory Map for Java 生成虚拟机的内部存储快照(dump文件)
- jhat: JVM Heap Dump Browser 用于分析heapdump文件,他会建立一个Http/HTML服务器,让用户在浏览器上查看分析结果
- jstack: Stack Trace for Java 显示虚拟机的线程快照
>#### jps:虚拟机进程状况工具
- JPS主要参数:
```text
-q 只输出PID,省略主类名称
-m 输出虚拟机进程启动时传递给主类main()函数的参数
-l 输出主类的全名,如果是jar包,则输出jar路径
-v 输出虚拟机进程启动时的JVM参数
```
>### jstat:虚拟机统计信息监视工具(JVM Statistics Monitoring Tool)
- 用于监控虚拟机各种运行状态信息的命令行工具,可以显示本地或者远程虚拟机进程的类装载、内存、垃圾收集、JIT编译等运行数据,在没有GUI图形界面的服务器上,它将是运行期定位虚拟机性能问题的首选工具
- 命令行格式: jstat [option vmid[interval[s|ms][count]]]
```text
VMID|LVMID 如果是本地虚拟机进程,VMID与LVMID是一致的,如果是远程虚拟机进程,那VIMID格式应当是
    [protocol:][//]lvmid@[hostname[:port]/servername]
interval和count代表查询建个和次数,如果省略则只查一次
    假设250ms查询一次进程2764的垃圾收集状况,一共查询20次,命令为:
        jstat -gc 2764 250 20
option选项代表用户希望查询的虚拟机信息,主要分为3类:类装在、垃圾收集、运行期编译状况
    -class: 监视类装载、卸载数量、总空间及类装载所消耗的时间
    -gc: 见识Java堆状况,包括Eden,S1,S2,Old,Perm等容量,已用空间,GC时间合计等
    -gccapacity:与-gc基本相同,但主要关注Java堆各个区使用到的最大、最小空间
    -gcutil:与-gc基本相同,但主要关注已使用空间占总空间的百分比
    -gccause:与-gcutil一样,但会额外输出导致上次GC产生的原因
    -gcnew:监视新生代GC状况
    -gcnewcapacity:监视内容与-gcnew一致,输出主要关注使用到的最大、最小空间
    -gcold:监视老年代GC状况
    -gcoldcapacity:
    -gcpermcapacity:
    -compiler:输出JIT编译器编译过的方法、耗时等信息
    -printcompilation:输出已经被JIT编译的方法
```
- jstat样例:
```text
jstat -gcutil 2764
S0 S1 E O P YGC YGCT FGC FGCT GCT
0.00 0.00 6.20 41.42 47.20 16 0.105 3 0.472 0.577

表明S0 S1里面都是空的
Eden区使用6.2%,老年代使用了41.42%
Yong GC次数为16次,用时0.105s
FULL GC3次,用时0.472s
一共GC所用时间为0.577s
```

>#### jinfo:Java配置信息工具(Configuration Info For Java)
- 实时地查看和调整虚拟机各项参数

- [ ] 参数查看
- 使用jps命令的-v参数可以查看虚拟机启动时显式指定的参数列表，但如果想知道未被显式指定的参数的系统默认值,除了去找资料外,就只能使用jinfo的-flag选项进行查询了
- JDK1.6以上版本的话,使用-XX:PrintFlagsFinal查看默认值也是一个很好的选择
- jinfo命令
```text
jinfo [option] pid
jinfo -flag CMSInitiatingOccupancyFraction 14444
```

>#### jmap:JVM内存映像工具(Memory Map for Java)
- jmap命令用于生成或转储快照(dump文件)
- 除了使用jmap还有一些手段也可以获取dump文件:譬如-XX:+HeapDumpOnOutOfMemoryError参数可以让虚拟机在OOM异常出现后自动生成dump文件
- jmap命令
```text
jmap [option] vimd
-dump 生成dump文件,格式为: -dump:[live,]format=b,file=<filename>,其中live子参数说明是否只给出dump存活对象
-finalizerinfo:显示在F-QUEUE中等待Finalizer线程执行finalize方法的对象,只在linux/solaris平台生效
-heap 显示Java堆详细信息,如使用哪种回收期,参数配置,分代状况等,只在linux/solaris平台生效
-histo 显示堆中的对象统计信息,包括类、实例数量、合计容量
-permstat 以ClassLoader为口径显示永久代内存状态,只在linux/solaris平台生效
-F 当前虚拟机进程对-dump选项没有响应时,强制生成dump快照,只在linux/solaris平台生效
```
>#### jhat:虚拟机堆转储快照分析工具(JVM Heap Dump Browser)
>#### jstack:Java堆栈跟踪工具(Stack Trace For Java)
- 用于生成虚拟机当前时刻的线程快照（一般称为threaddump或者javacore文件),一般用于分析死锁、死循环、请求外部资源导致的长时间等待等线程停顿问题
```text
jstack [option] vmid
-F 当正常输出的请求不被响应时,强制输出线程堆栈
-l 除堆栈外,显示关于锁的附加信息
-m 如果调用到本地方法,可以显示C/C++的堆栈
```
### 可视化工具
- JConsole
- VisualVM

## 七、虚拟机类加载机制
### 7.2类加载时机
- 虚拟机没有强制规定类加载时机,但对初始化有严格的规定

```text
在未初始化的情况下
1.遇到new、getstatic、putstatic、invokestatic指令的时候
2.使用java.lang.reflect包的方法对类进行反射调用
3.初始化一个类时,如果发现父类还没有进行初始化,则需要先触发器父类的初始化
4.当虚拟机启动时,用户需要执行一个类的主类,虚拟机会先初始化这个主类
5.动态语言支持时,如果一个java.lang.invoke.MethodHandle实例的最后解析结果方法句柄对应的类没有初始化
```
### 7.3类加载过程
- 加载
- 验证(连接)
- 准备(连接)
- 解析(连接)
- 初始化
>#### 加载
- 加载阶段,虚拟机需要完成三件事情
```text
1.通过一个类的全限定名来获取定义此类的二进制流
2.将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构
3.在内存中生成一个代表这个类的Class对象,作为方法区这个类的各种数据的访问入口
```
>#### 连接
- [ ] 验证
- 为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求,并且不会危害虚拟机自身的安全
- [ ] 准备
- 准备阶段是正式为类变量分配内存并设置变量初始值的阶段
```text
1.这时候进行内存分配的仅包括类变量（被static修饰的变量),而不包括实例变量
2.准备阶段的初始值是该数据类型的0值,比如int的0,对象的null值
```
- [ ] 解析
- 虚拟机将常量池内的符号引用替换为直接引用的过程
```text
符号引用:以一组符号来描述锁引用的目标,符号引用可以是任何形式的字面量,只要使用时能无歧义的定位到目标即可
    符号引用于虚拟机实现的内存布局无关,引用的目标不一定已经加载到内存中.各种虚拟机实现内存布局可以不同,但其能够接受的符号引用都是一致的
    符号引用在Class文件中一般以CONSTANT_Class_info、CONSTANT_Fieldref_info、CONSTANT_Methodref_info等类型的常量出现
直接引用:直接引用可以使直接指向目标的指针、相对偏移量或者是一个能够间接定位到目标的句柄.
    直接引用是和虚拟机内存布局相关的,同一个符号引用在不同的虚拟机实例上翻译出来的直接引用一般不相同
```
>#### 初始化
- 初始化过程就是执行<init>方法的过程,将变量的初始值替换成程序所指定的值

### JNI??
