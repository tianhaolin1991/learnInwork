# Java_NIO 中文版

## 第一章、简介

### 1.1 I/O与CPU时间的比较

- I/O的操作时间大概是CPU的处理时间的几百倍,I/O是程序的瓶颈

### 1.4 I/O概念

> #### 缓冲区操作

- 进程执行 I/O 操作，归结起来，也就是向操作系统发出请求，让它要么把缓冲区里的数据排干(写)，要么用数据把缓冲区填满(读)

```text
读:
进程调用操作系统的read()指令,要求其缓冲区被填满.
内核随即向磁盘控制硬件发出命令,要求其从磁盘读取数据.磁盘控制器把数据写入内核内存缓冲区(由DMA)完成
一旦缓冲区被装满,内核即把数据从内核空间的临时缓冲区拷贝到read()调用时指定的缓冲区=>内核态到用户态
动作:
进程(读)->内核read()->磁盘读
数据:
磁盘->内核缓冲区->用户空间缓冲区
```

- 当进程请求I/O操作时,它执行一个系统调用(也称为陷阱)将控制权移交给内核.

```text
C程序员所熟知的底层函数open(),read()等无非就是建立和执行适当的系统调用
当内核以这种方式被调用,它随即采取任何必要措施,找到进程所需要的数据,并把数据传送到用户指定的缓冲区内
内核首先会试图从高速缓存中读取,如果没有数据,则将用户进程挂起,从硬盘读入数据到内存
```

> #### 虚拟内存

- 虚拟内存的好处

```text
1.一个以上的虚拟空间地址可以指向同一个物理地址
2.虚拟内存空间可以大于实际可用的硬件内存

由于设备不能直接访问用户空间,我们之前必须将数据从内核态复制到用户态
而虚拟内存的第一个好处表明我们可以将内核虚拟内存和用户虚拟内存映射到一个物理内存,从而实现零拷贝
```

- 内存页面调度

```text
为了实现第二个好处,就必须进行虚拟内存分页(经常称为交换)
依照方案,虚拟内存空间的页面能够继续存在外部磁盘存储,这样就为物理内存中的其他虚拟页面腾出了空间
```

> #### 文件I/O

- [ ] 概述
- 文件I/O属于系统范畴,与磁盘不同

```text
磁盘把数据放在扇区上(一般为512Byte),硬盘属于系统硬件,对何为文件一无所知
文件系统是更高层次的抽象,是安排、解释磁盘数据的一种独特方式.
```

- 文件I/O的转换

```text
所有 I/O 都是通过请求页面调度完成的,页面调度是非常底层的操作,仅发生于磁盘扇区与内存页之间的直接传输
文件系统把一连串大小一致的数据块组织到一起,当用户进程请求读取文件时,文件系统负责确定数据具体在磁盘的什么位置,然后把相关磁盘扇区读入内存.
```

- 内存映射文件

```text
传统的文件I/O是用户通过read()和write()来调用数据的
内存映射I/O利用虚拟内存,建立从用户空间直到文件系统页内存的映射
```

- [ ] 文件锁定
- 文件锁定机制允许一个进程阻止其他进程存取某个文件,或限制其存取方式
- 文件锁定不仅可以锁定整个文件,也可以更加细致,锁定区域往往可以细致到单个字节
- 文件锁定也可以分为共享锁和独占锁

> #### 流I/O

- 并非所有 I/O 都像前几节讲的是面向块的，也有流 I/O,其原理模仿了通道,I/O字节流必须顺序存取

```text
流的传输一般（也不必然如此）比块设备慢，经常用于间歇性输入
多数操作系统允许把流置于非块模式
这样,禁程可以查看流上是否有输入,即便当时没有也不影响它干别的.这样一种能力使得进程可以在有输入的时候进行处理,输入流闲置的时候执行其他功能。
```

## 第二章、缓冲区

- 一个Buffer对象是固定数量的数据的容器.其作用是一个存储器,或者分段运输区,在这里数据可被存储并在之后用于检索
- 对于每一个非Bollean类型的原始数据都有一个缓冲区类

```text
Buffer 超类
CharBuffer、ByteBuffer、IntBuffer、DoubleBuffer、ShortBuffer、FlotBuffer、LongBuffer
ByteBuffer还有一个子类:MappedByteBuffer
```

### 2.1缓冲区基础

> #### 属性

- 容量(capacity):缓冲区能够容纳数据元素的最大数量,在缓冲区创建时被设定,并且永远不能被改变
- 上界(limit): 缓冲区的第一个不能被读或者写的元素.或者说缓冲区中现存元素的计数
- 位置(position):下一个要被读写的元素的索引.位置会由get()、put()函数更新
- 标记(mark):一个备忘位置

```text
mark():设定mark=position
reset():设定position=mark
标记在设定前是未定义的
```

- 0<=mark<=position<=limit<=capacity,总是成立

> #### Buffer API

- int capacity(): 获取capacity值
- int position(): 获取position值
- int limit(): 获取limit值
- Buffer position(int newP):设置position
- Buffer limit(int newL):设置新limit
- Buffer mark(): 标记mark值
- Buffer reset(): 设置position为mark值
- Buffer clear(): 清空缓冲区,position=0,limit=capacity,mark=-1
- Buffer flip(): 反转缓冲区,limit=position,position=0,mark=-1

```text
如写入后读取写入的数据
假设缓冲区capacity为10
写入0,1,2三个数据后此时position=3,limit=10
调用flips后
position=0,limit=3
再读数据的话,依次为0,1,2读到2后将不能再向后读
```

- Buffer rewind(): position=0,remark=-1
- int remaining(): 剩余元素个数
- boolean hasRemaining(): 缓冲区是否已满
- boolean isReadOnly(): 是否只读

> #### 存取API(以ByteBuffer为例)

- byte get(): 获取当前position的byte并将position+1
- byte get(int index): 获取index位置的byte
- ByteBuffer put(byte b)
- ByteBuffer put(int index,byte b)

> #### 缓冲区使用示例

- [ ] 存放
- 例:将Hello放入一个容量为10的缓冲区

```text
buffer.put((byte)'H').put((byte)'e').put((byte)'l').put((byte)'l').put((byte)'o')
此时缓冲区的属性为:
mark = -1
position = 5
limit = 10
capacity = 10
```

- [ ] 翻转
- 我们已经写满了缓冲区,现在我们必须准备将其清空,在这之前我们先把这个缓冲区传递给一个通道,以使内容能被全部写出

```text
此时如果通道在缓冲区上执行get(),那么它会读取到未定义的数据或抛出异常,因为此时的position=5(接上例)
那么我们需要将position重新设置为0,并且设置一个limit标识缓冲区内的数据大小

buffer.flip()翻转缓冲区,此时
position=0
limit=5
capacity=10
mark=-1

这样当我们将Buffer传入通道,通道就可以从头读取数据了,直到缓冲区的上界
```

- [ ] 释放
- 当缓冲区中的数据被使用后,我们就可以释放缓冲区了

```text
假设我们已经读取完Hello,此时position=5,limit=5,capacity=10,mark=-1
buffer.clear()就会移动缓冲区的指针,使得缓冲区可以被重新写入数据
position=0
limit=10
capacity=10
mark=-1

注意:clear()不会清空缓冲区中的内容,只是移动指针,使得缓冲区可以重新被使用

如果我们执行如下的代码,hello就会被输出两次
public static void main(String[] args) {
    ByteBuffer buffer = ByteBuffer.allocate(10);
    buffer.put((byte)'H').put((byte)'e').put((byte)'l').put((byte)'l').put((byte)'o');
    buffer.flip();
    int remaining = buffer.remaining();
    for (int i = 0; i < remaining; i++) {
        System.out.print((char) buffer.get());
    }
    buffer.clear();
    System.out.print(System.lineSeparator());
    for (int i = 0; i < remaining; i++) {
        System.out.print((char) buffer.get());
    }
}
```

- [ ] 压缩
- 有时候我们只想从缓冲区释放一部分数据,而不是全部,然后重新填充,为了实现这一点,未读的数据需要下移使得第一个未读元素的索引为0.
- 尽管重复这样做会效率低下,但这有时非常必要
- compact()函数实现了这一逻辑,并且比使用get(),put()要快得多

```text
0 1 2 3 4 5 6 7 8 9 10
H e l l o N/A
假设此时position=2,limit=5

调用compact()会使得缓冲区的数据如下
0 1 2 3 4 5 6 7 8 9 10
l l o l o N/A
position=3,limit=10,这样我们再执行put()时,就会覆盖3的l
```

- [ ] 标记
- 为当前的position做标记,以便可以实现reset()操作

```text
reset()后,mark= -1 position回到mark的位置
```

- [ ] 比较
- 比较position和limit之间的数据

- [ ] 批量移动
- 缓冲区提供了批量移动的API

```text
CharBuffer get(char[] dst):从缓冲区中复制数组
CharBuffer put(char[] src):将src数组放入缓冲区
```

### 2.2创建缓冲区

- CharBuffer allocate(int capacity):隐式的分配了一个char型的数组来储存100个char变量
- CharBuffer wrap(char[] arr):使用自己创建的数组用作缓冲区的存储器

```text
wrap()创建了一个缓冲区对象,但是元素会存储在指定的数组中,如果我们对数组中的数据进行了操作,会影响到缓冲区
```

- CharBuffer wrap(CharSequence csq):这是CharBuffer特有的一个方法

```text
wrap()函数创建一个只读的备份存储区是CharSequence接口或者其实现的缓冲区对象
CharSequence描述了一个可读的字符流,三个标准的实现了CharSequence的类,String,StringBuffer,CharBuffer
CharSequence详见第五章
```

### 2.3复制缓冲区

- [ ] 视图缓冲区
- 缓冲区不限于管理数组中的外部数据,也能管理其他缓冲区中的外部数据.管理其他缓冲区所包含的数据元素的缓冲区被称为<font color='red'>视图缓冲区</font>
- 大多数的视图缓冲区都是ByteBuffer的视图
- [ ] 视图缓冲区的创建
- 视图缓冲区总是调用已经存在缓冲区实例的函数来撞见
- duplicate():创建一个与原始缓冲区相似的缓冲区

```text
两个缓冲区共享数据元素,拥有同样的容量.但每个缓冲区拥有各自的positoin,limit和mark属性
对一个缓冲区内的数据元素所作的改变会反映在另一个缓冲区上.
如果原始的缓冲区为"只读"或者"直接缓冲区",新的缓冲区将继承这些属性
"直接缓冲区会在2.4中讨论"
```

- asReadOnlyBuffer:生成一个只读缓冲区,与duplicate()一样,但是一定会将属性设置为"只读"
- slice():创建一个从原始缓冲区的当前位置开始的新缓冲区,容量是原始缓冲区的剩余容量(limit-position)

```text
slice()创建的缓冲区仍然与原始缓冲区共享数据
同样创建出来的缓冲区也继承"只读"等属性
```

### 2.4字节缓冲区

- 字节是I/O设备使用的基本数据类型,当JVM和操作系统间传递数据时,将其他数据类型拆分构成他们的字节是必要的
- 每个基本类型(除了Boolean)都是由组合在一起的几个字节组成的,这些数据一般都是以连续字节的形式存储

```text
布尔型代表两种值：true或false
一个字节可以包含256个唯一值,所以一个布尔值不能准确地对应到一个或多个字节上
字节是所有缓冲区的构造块,NIO架构决定了布尔缓冲区的执行是有疑问的,而且对这种缓冲区类型的需求不管怎么说都是颇具争议的
```

> #### 直接缓冲区

- 字节缓冲区跟其他缓冲区的最大不同,<font color='red'>是ByteBuffer可以成为通道所执行的I/O源头或者目标</font>

```text
操作系统的在内存区域中进行I/O操作,这些内存区域是相连的字节序列,所以只有字节缓冲区可以参与I/O操作
操作系统会直接存取进程(JVM)进程的内存空间,以传输数据
    这意味着I/O操作的目标内存区域必须是连续的字节.在JVM中,字节数组可能不会在内存中连续存储
    出于这一原因,引入了直接缓冲区的概念
```

- [ ] 非直接缓冲区
- 非直接缓冲区是使用传统的方式,需要将用户数据拷贝到内核态的缓存中,再和I/O交互
- 非直接的ByteBuffer不能直接成为I/O的直接对象,通常将非直接的ByteBuffer传入通道会隐式完成以下操作

```text
1.创建一个临时的直接ByteBuffer对象
2.将非直接缓冲区的内容复制到临时缓冲区
3.使用临时缓冲区执行I/O操作
4.临时缓冲区对象离开作用域,并最终被垃圾回收
```

- [ ] 直接缓冲区
- 直接缓冲区实现零拷贝

```
直接缓冲区利用虚拟内存
JVM尽最大努力将数据放入堆外内存,并直接操作虚拟内存与物理内存的映射
内核态也建立一个虚拟内存到物理内存的映射,从而I/O系统可以直接交互,避免了拷贝
```

- 直接缓冲区时I/O的最佳选择，但可能比创建非直接缓冲区要花费更高的成本

```text
直接缓冲区的内容可以驻留在垃圾回收堆之外(使用的是DirectMemory)
```

- [ ] 创建直接缓冲区
- ByteBuffer allocateDirect(int capacity)
- boolean isDirect()

```text
所有的缓冲区都有这个函数,虽然ByteBuffer是唯一可以被直接分配的类型
但是如果基础缓冲区是一个"直接"ByteBuffer,对于非字节视图缓冲区,isDirect()也可以为True
```

> #### 视图缓冲区

- 我们可以利用ByteBuffer类来创建直接视图缓冲区
- CharBuffer asCharBuffer()

```text
一个ByteBuffer的CharBuffer视图
backing storage[bytes]
0  1  2  3  4  5  6
ByteBuffer
position 0  1  2  3  4  5  6
映射      0  1  2  3  4  5  6
CharBuffer
position 0   1   2
映射      01  23  45
```

- ShortBuffer asShortBuffer()

> #### 内存映射缓冲区(MappedByteBuffer)

- MappedByteBuffer是java nio引入的文件内存映射方案，读写性能极高

```text
Java将磁盘中的文件映射到内存中,然后通过修改内存的数据,从而间接修改了磁盘中的文件
这样可以快速的堆磁盘中的文件进行修改,将原来的磁盘I/O操作转换成了内存I/O
```

- 创建

```text
FileChannel提供了一个map方法来创建MappedByteBuffer
MappedByteBuffer map(int mode，long position，long size)
    模式有3种:
        READ_ONLY
        READ_WRITE
        PRIVATE
```

## 第三章、通道

### 3.0预备

- 通道是java.nio的第二个主要创新.Channel用于字节缓冲区和位于通道另一侧的实体之间有效的传输数据
- 多数情况下通道和文件描述符(File Descriptor)和文件句柄(File Handle)

```text
linux中一切都可以看作是文件,不仅包括普通的文件和目录,还包括管道、FIFO、Socket、终端、设备等
```

> #### 文件描述符

- [ ] 文件描述符(file descriptor)
- 每一个文件描述符会与一个打开的文件相对应,不同的文件描述符也可以指向同一个文件

```text
系统为每一个进程都维护了一个文件描述符表,该表的值都是从0开始的,所以在不同的进程中你会看到相同的文件描述符
```

- 文件描述符是一个较小的非负整数,并且0,1,2这三个描述符总是默认分配给标准输入、标准输出和标准错误

```text
这就是nohup 命令中 nohup java -jar xxxx.jar > myLog.log 2 >&1 中的2和1的由来
```

- 文件描述符每个条目包括两个部分,一个是控制该描述符的标记(flags),二是指向<font color='red'>open file table</font>对应条目的指针 - [ ]系统级别的打开表 & 文件句柄
- 内核会维护系统内所有打开的文件及其相关信息,该结构被称为打开文件表(open file table)

```text
表中的每个条目维护以下信息:
文件偏移量,read()/write()/lseek()函数都会修改该值
打开文件时的状态和权限标记(通过open()函数的参数传入)
文件的访问模式(只读、读写、只写等)
指向其对应的inode对象的指针.(内核还会维护一个系统级别的inode表)
```

- 文件句柄:打开文件表中的一行称为一条文件描述(file description),也叫做文件句柄
- [ ] inode表
- inode主要存放一些文件信息,文件类型(如常规文件、Socket)、文件的各种属性(大小、不同类型操作相关的时间戳等)
- inode表中的一条记录代表一个文件,是文件的指针,所以文件句柄是指针的指针
- [ ] 文件描述符、文件句柄、I-Node索引节点的关系
- 每个进程维护一套文件描述符,不同的文件描述符可能会指向系统级打开文件表中的一条记录(文件句柄)
- 文件句柄指向I-Node索引表中的一条记录

```text
不同的文件句柄也会指向同一个索引表记录,也就是同一个文件
这种情况可能是不通进程对同一个文件发起open()导致的
也可能是同一个进程打开了两次同一个文件
```

- 文件描述符是进程级句柄,文件句柄是文件指针的指针,I-Node是文件的指针

### 3.1通道基础

- JAVA仅提供了SPI,不通的平台有不同的具体实现

> #### 打开通道

- I/O可以分为广义的两大类别:FILE I/O和Stream I/O,对应的有两种通道File通道和Socket通道

```text
JAVA提供了对应的4个类
FileChannel
ServerSocketChannel、SocketChannel和DatagramChannel
```

- [ ] Socket通道
- StreamChannel的打开

```java
import java.net.InetSocketAddress;
import java.nio.channels.DatagramChannel;
import java.nio.channels.ServerSocketChannel;

public class StreamChannelOpen {
    public static void main(String[] args) {
        SocketChannel sc = SocketChannel.open();
        sc.connect(new InetSocketAddress("somehost", someport));
        ServerSocketChannel ssc = ServerSocketChannel.open();
        ssc.socket().bind(new InetSocketAddress(somelocalport));
        DatagramChannel dc = DatagramChannel.open();
    }
}

java.net.Socket类中也有getChannel()方法,这些方法虽然能返回一个的SocketChannel对象,
        但是他们却并非新的通道来源,只有已经有通道存在的时候,它才返回与socket关联的通道.它们永远不会创建新的通道
```

- [ ] File通道
- FileChannel打开

```java
import java.io.RandomAccessFile;
import java.nio.channels.FileChannel;

public class FileChannelOpen {
    public static void main(String[] args) {
        RandomAccessFile raf = new RandomAccessFile("someFile", "r");
        FileChannel fc = raf.getChannel();
    }
}
```

> #### 使用通道

- ReadableByteChannel和WriteableByteChannel是上面通道要实现的接口
- SocketChannel是双向的很好理解,因为Socket的数据传输本来就是双向的.
- FileChannel声名上也是一个双向的通道,但是事实往往并非如此,比如FileInputStream()是只读流,那么要写入的话会抛出NonWriteableChannelException
- [ ] 数据读写
- int read (ByteBuffer dst):
  将Channel读取数据到Buffer,返回已传输的字节数,可能会比缓冲区的字节数少,甚至可能为0,缓冲区的位置也会发生已与传输字节相同数量的移动,如果仅进行了部分传输,缓冲区可以被重新提交给通道并从上次中断的地方继续传输,该过程可以重复进行直到缓冲区hasRemaining方法返回false

```java
import java.io.RandomAccessFile;
import java.nio.channels.FileChannel;

