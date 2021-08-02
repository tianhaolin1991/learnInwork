### 3.5 Socket通道

- 新的socket通道类可以运行非阻塞模式并且是可选择的.这两个性能可以极大激活大程序巨大的可伸缩性和灵活性
- 再也没有必要为每个socket连接使用一个线程的必要了,也避免了管理大量线程所需要的上下文交换总开销.

```text
借助NIO,一个或几个线程就可以管理成百上千的活动socket连接并且只有很少甚至可能没有性能损失
```

- 类图
  https://www.processon.com/diagraming/6104acc10e3e740416e3e43d

- 类图中显示DatagramChannel和SocketChannel实现定义读写功能接口而ServerSocketChannel不实现,它只负责监听传入的连接和创建新的SocketChannel对象

> #### Socket和SocketChannel

- Socket是系统套接字的抽象
- SocketChannel类在被实例化时都会创建一个对等的Socket对象.
- SocketChannel的socket()方法可以获得Socket对象,而这些对象也都有getChannel()方法

```text
如果使用传统方法创建一个socket(),它的getChannel()方法总是返回null,而不是创建一个新的Channel
```

- SocketChannel委派协议操作给Socket对象

> #### 非阻塞模式

- 要把Socket通道置于非阻塞模式,我们要依靠SelectableChannel.
- 设置非阻塞

```text
SocketChannel sc = SocketChannel.open();
sc.configureBlocking(false);
...
if(!sc.isBlocking()) {
    doSomething(cs);
}

服务器端的使用经常会考虑到非阻塞socket通道，因为它们使同时管理很多socket通道变得更容易
但是,在客户端使用一个或几个非阻塞模式的socket通道也是有益处的,例如,借助非阻塞socket通道
GUI程序可以专注于用户请求并且同时维护与一个或多个服务器的会话.在很多程序上，非阻塞模式都是有用的
```

> #### ServerSocketChannel

- ServerSocketChannel是一个基于通道的Socket监听器.它同我们熟悉的ServerSocket执行相同的基本任务,但是它可以在非阻塞模式下运行
- 创建: ServerSocketChannel.open()
- 绑定: 除了我们画的类图之外,ServerSocketChannel还实现了NetworkChannel接口,我们通过bind(SocketAddress address)来绑定监听的端口

```text
使用serverSocket.socket()获取到ServerSocket对象,再调用socket对象的bind()方法也是等价的
```

- 获取Socket对象: accept()

```text
如果在ServerSocket()上调用accept(),那么就会同任何其他的ServerSocket表现一样的行为:总是阻塞并返回一个Socket对象
如果在ServerSocketChannel()上调用accept(),则会返回一个SocketChannel()类型的对象,返回的对象可以在非阻塞模式下运行
假设系统已经有一个安全管理器(security manager),两种形式的方法调用都执行相同的安全检查

如果以非阻塞模式被调用,当没有传入连接在等待时,
ServerSocketChannel.accept( )会立即返回null
正是这种检查连接而不阻塞的能力实现了可伸缩性并降低了复杂性
```

- ServerSocketChannel例:

```java
```

> #### SocketChannel

- Socket和SocketChannel类封装点对点、有序的网络点解.
- SocketChannel扮演客户端发起同一个监听服务器的连接.直到连接成功,它才能收到收据并且只会从连接到的地址接收数据
- 连接: connect(SocketChannel socketChannel)

```text
刚创建的SocketChannel是未连接的状态,通过connect方法向监听器(ServerSocket)发起连接
如果使用Socket.connect()方法,那么传统的连接语义将适用于此,线程在连接建立好或者超时之前都将保持阻塞
如果在SocketChannel()上直接调用connect()方法来建立连接并且通道处于阻塞模式,那么连接过程实际上是一样的
如果是非阻塞的SocketChannel(),需要调用finishConnect()才可以完成连接,在connect()->finishConnect()期间可以做其他事情
```

- 等待连接检测:isConnectionPending()

```text
是否一个connection操作正在进行
当且仅当一个连接动作已经启动但尚未完成(未调用finishConnect()方法)时会返回true
```

- finishConnect():完成连接

```text
对于非阻塞的连接,连接操作会被设置为非阻塞模式,并通过connect()方法初始化
1.一旦连接已经建立或者失败,SocketChannel就变成"connectable"状态,此时执行finishConnect()方法就可以完成连接序列(Connect Sequence)
如果连接阶段失败,该方法会抛出IOException
2.如果连接已经准备完成,会立刻返回true
3.如果Channel是非阻塞的并且连接还未准备好,就会返回false,否则返回true
4.如果Channel是阻塞的,会阻塞到连接建立完成并返回true
5.该方法可以随时调用,如果一个读或写操作同时被执行,读写操作会被block,直到finishConnect()操作完成
总结:当前仅当Channel的Socket已经连接的情况下才会返回true
```

- SocketChannel是线程安全的,不过任何时候都<font color='red'>只有一个读或写操作在进行中</font>

```text
socket是面向流而非面向包的,它可以保证发送字节会按照顺序到达,但是无法承诺维持字节数组.
某个发送器可能给一个socket写入了20个字节而接收器使用read()只接收到了3个字节,剩下的17个还在传输中.
由于这个原因,让多个互不配合的线程共享某个流socket的同一侧并非一个好的设计选择
```  

- SocketChannel示例
```java
public class SocketChannelUnBlock {
    public static void main(String[] args) throws IOException {
        SocketChannel sc = SocketChannel.open();
        sc.configureBlocking(false);
        sc.connect(new InetSocketAddress("127.0.0.1", 9091));
        while (!sc.finishConnect()) {
            doSomethingUseful();
        }
        System.out.println("connection established");
        transferFile(sc);
        WritableByteChannel writableByteChannel = Channels.newChannel(System.out);
        ByteBuffer buffer = ByteBuffer.allocateDirect(1024);
        while (true) {
            if (sc.read(buffer) != 0) {
                buffer.flip();
                writableByteChannel.write(buffer);
                break;
            }
        }
        sc.close();
    }
}
```
>#### DatagramChannel
- 同上DatagramChannel对应DatagramSocket对象,它使用UDP传输
- DatagramChannel的创建和其他Channel一样,DatagramChannel既可以当Server又可以当cli
- DatagramChannel是无连接的,每个数据包都是一个自包含的实体,拥有它自己的目的地址及不依赖其他数据包的数据.
```text
DatagramChannel可以发送单独的数据报给不同的目的地址
DatagramChannel对象也可以接收来自任意地址的数据包,每个到达的数据报都含有关于它来自何处的信息(源地址)

一个未绑定的DatagramChannel仍能接收数据包
当一个底层socket被创建时,一个动态生成的端口号就会分配给它.绑定行为要求通道关联的端口被设置为一个特定的值
不论通道是否绑定,所有发送的包都含有DatagramChannel的源地址(带端口号)
未绑定的DatagramChannel可以接收发送给它的端口的包,通常是来回应该通道之前发出的一个包
已绑定的通道接收发送给它们所绑定的熟知端口(wellknown port)的包
数据的实际发送或接收是通过send( )和receive( )方法来实现的：
```
- DatagramChannel的connect()方法可以使得它忽略除了目标以外的所有数据
```text
这是一个安全控制
```
### 3.6管道(Pipe)

## 四、选择器
- 就绪选择和多元执行使得单线程能够有效率地同时管理多个I/O通道
```text
C/C++代码的工具箱中,许多年前就已经有select()和poll()两个系统调用可以使用了
JAVA 1.4才成为可行的方案
```
### 4.1选择器基础
>#### 选择器(Selector),可选择通道和选择键
- [ ] 选择器
- 选择器管理着一个被注册的通道集合的信息和它们的就绪状态,通道是和选择器一起被注册的,并且使用选择器来更新通道的就绪状态
- 当这么做的时候,可以选择将被激发的线程挂起,直到有就绪的通道
- [ ] 可选择的通道(SelectableChannel)
- 提供了注册到选择器上的方法,一个通道可以注册到多个选择器上,但对每个选择器而言只能被注册一次
- [ ] 选择键
- 选择键封装了特定的通道与特定的选择器的注册关系.
- SelectableChannel.register()返回并提供一个表示这种注册关系的标记.

### 4.2使用选择键
- [ ] 选择键中的操作bit
- public static final int OP_READ = 1 << 0
- public static final int OP_WRITE = 1 << 2
- public static final int OP_CONNECT = 1 << 3
- public static final int OP_ACCEPT = 1 << 4
- SelectionKey中保存了两个以整数形式进行编码的比特掩码,一个用于指示key中channel和selector所关心的操作(interest集合),另一个通道准好要执行的操作(ready集合)
```text
Interest集合:可以通过调用键对象的interestOps()来获取
    最初这个值是注册时传递的值,可以用OP_READ|OP_WRITE这样的或操作来赋值
    interestOps(int ops),可以改变这个值
Ready集合
    通过readyOps()获取相关通道的已经就绪的操作.例如下面的代码测试了是否就绪,如果已就绪则将数据读入缓冲区
    
    if((key.readOps() & SelectionKey.OP_READ) != 0) {
        myBuffer.clear();
        key.channel().read(myBuffer);
        doSomethingWithBuffer(myBuffer.flip());
    }
```
- [ ] 选择键API
- SelectableChannel channel(): 返回创建选择键的通道
- Selector selector(): 返回创建选择键的Selector
- boolean isValid(): 选择键是否有效
```text
以下三种情况会使得选择键无效
1.调用了cancel()
2.通道被关闭
3.选择器被关闭
```
- void cancel(): 取消注册
```text
调用cancel()之后,key将会被放入selector的cancelled-key set
在下一次selector的select操作后,key将会被从key set中移除
cancelled key set和key set请见4.3
```
- interestOps(): 关心的操作集合
- isWriteable(): 通道的写操作是否已准备好
```text
return (readOps&OP_READ) != 0
```
- isReadable(): 通道的读操作是否已准备好
- isConnectable(): 通道是否完成connection
- isAcceptable(): Test whether this key's channel is ready to accept a new socket connection
- Object attach(Object obj): 设置键上的附件
- Object attachment(): 获取键上的附件
```text
这是一种允许将任意对象与键关联的方法.
这将允许遍历与选择器相关的键,使用附加在上面的对象句柄作为引用来获取相关的上下文
```
### 4.3使用选择器
>#### 选择过程
- 每个Selector维护三个键集合
```text
Registered key set: 与选择器关联的已注册的键的集合,有效无效的键都在这个集合中.通过keys() API返回
Selected key set: 已注册的键集合的子集合.这个集合的每一个成员都是相关的通道被选择器判断已经准备好的,并且包含于键的interest集合中的操作.通过selectedKeys()方法返回
Cancelled key set: 已注册的键集合的子集合.这个集合包含了cancel()方法被调用过的键(这个键已经被无效化),但他们还没有被注销
```
- 选择过程:当select()三种形式中的任意一种被调用,将执行如下操作
```text
1.检查Cancelled key set,如果它是非空的,每个已取消的键集合中的键将从另外两个集合中移除.并且相关的cancel在选择器中将会注销
2.Registered key set中的键的interest集合将会被检查.在这个步骤之后,对interest集合的改动不影响此次的select()操作
    一旦检查标准确定,底层操作系统将会进行查询,以确定每个通道所关心的操作真是就绪状态
    依赖于select()方法调用,如果没有通道已经准备好,线程可能在这里阻塞.通常都会设置一个超时值
    直到系统调用完成为止,这个过程可能会使得调用线程睡眠一段时间.
        a.如果通道的键还没有处于已选择的键的集合中，那么键的ready集合将被清空，然后表示操作系统发现的当前通道已经准备好的操作的比特掩码将被设置
        b.否则,也就是键在已选择的键的集合中.键的ready集合将被表示操作系统发现的当前已经准备好的操作的比特掩码更新.
            所有的比特位都不会被清理,由操作系统决定的ready集合是与之前的ready集合按位分离的,一旦键被放置于选择器的已选择的键的集合中,它的ready集合将是累积的.比特位只会被设置，不会被清理.
3. 步骤2可能会花费较长的时间,所以在步骤2结束时,步骤1将会被重新执行
4. select操作返回的值是ready集合在步骤2中被修改的键的数量,而不是已选择的键的集合中的通道总数.
    之前调用中准备就绪的,并且在本次调用中仍然就绪的通道不会被计入,而那些之前就绪现在不再处于就绪的通道也不会被计入
```
>#### 停止选择过程
- wakeup():使得选择器上的第一个还没有返回的选择操作理解返回.如果当前没有select()方法被调用,下一次对select()方法的调用将立即返回
- close(): 如果选择器的close()方法被调用,那么任何一个在选择操作中阻塞的线程都会被唤醒.就像wakeup()被调用了一样
- interrupt(): 如果睡眠中的线程interrupt()方法被调用,它的状态将被设置
>#### 管理选择键
- 选择键是积累的,<font color='red'>一旦一个选择器将一个键添加到它的已选择的键的集合中,它就不会移除这个键.并且一个键的ready集合将只会被设置不会被清理</font>
```text
操作键提供给了极大的灵活性,但把合理地管理键以确保他们表示的状态信息不会过时的任务交给了程序员
```
- 清理一个SelectedKey的ready集合的方式是将这个键从已选择的键集合中移除

- 使用select()为多个通道提供服务
```java
public class UseSelector {
  public static int PORT_NUMBER = 1234;
  private ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

  public static void main(String[] args) throws Exception {
    new UseSelector().go();
  }

  private void go() throws Exception {
    int port = PORT_NUMBER;
    // Allocate an unbound server socket channel
    ServerSocketChannel serverChannel = ServerSocketChannel.open();
    // Set the port the server channel will listen to
    serverChannel.bind(new InetSocketAddress(port));
    // Set nonblocking mode for the listening socket
    serverChannel.configureBlocking(false);

    // Create a new Selector for use below
    Selector selector = Selector.open();
    // Register the ServerSocketChannel with the Selector
    serverChannel.register(selector, SelectionKey.OP_ACCEPT);

    while (true) {
      int n = selector.select();
      if (n == 0) {
        continue;
      }
      Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
      while (iterator.hasNext()) {
        SelectionKey key = iterator.next();
        // Is a new connection coming in?
        if (key.isAcceptable()) {
          ServerSocketChannel server = (ServerSocketChannel) key.channel();
          SocketChannel channel = server.accept();
          registerChannel(selector, channel, SelectionKey.OP_READ);
          sayHello(channel);
        }

        // Is there data to read on this channel?
        if (key.isReadable()) {
          readDataFromSocket(key);
        }
        // Remove key from selected set; it's been handled
        iterator.remove();
      }
    }
  }

  private void readDataFromSocket(SelectionKey key) throws IOException {
    SocketChannel socketChannel = (SocketChannel) key.channel();
    int count;
    buffer.clear();
    // Empty buffer // Loop while data is available; channel is nonblocking
    while ((count = socketChannel.read(buffer)) > 0) {
      buffer.flip(); // Make buffer readable
      // Send the data; don't assume it goes all at once
      while (buffer.hasRemaining()) {
        socketChannel.write(buffer);
      }
      // WARNING: the above loop is evil.Because
      // it's writing back to the same nonblocking
      // channel it read the data from, this code can
      // potentially spin in a busy loop. In real life
      // you'd do something more useful than this.
      buffer.clear(); // Empty buffer
    }
    if (count < 0) {
      // Close channel on EOF, invalidates the key
      socketChannel.close();
    }
  }

  private void sayHello(SocketChannel channel) throws Exception {
    buffer.clear();
    buffer.put("Hi there!\r\n".getBytes());
    buffer.flip();
    channel.write(buffer);
  }

  private void registerChannel(Selector selector, SocketChannel channel, int ops) throws IOException {
    if (channel == null) {
      return;
    }
    channel.configureBlocking(false);
    channel.register(selector, ops);
  }
}
```
>#### 并发性
- 选择器是线程安全的.但是键集合不是
