---
layout: post
title:  "bio、nio、aio"
categories: java-basics
tags:  io
author: 刚子
---

* content
{:toc}

总结bio、nio、aio的知识点











## 一、BIO

> 传统的阻塞IO（BIO）的服务器端利用一个Acceptor线程来接收客户端的连接，接收到请求之后为每个客户端创建一个线程来处理，处理完成之后通过输出流响应客户端。客户端并发访问量增大的时候，服务端的线程数也1:1的数量增大，系统性能会随着并发访问量增大会急剧下降。

主要用到的2个类：

* ServerSocket 服务端套接字，提供accept方法用来接收客户端连接请求，该方法会阻塞
* Socket 客户端套接字

示例：

```java
//服务端代码

import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;

public class TimeServer {
    private static int port = 8080;

    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = null;
        try {
            serverSocket = new ServerSocket(port);
            System.out.println("server starts in port: " + port);

            Socket socket = null;
            while (true) {
                // 监听来自客户端的连接，主线程阻塞在accept操作上
                socket = serverSocket.accept();
                // 创建一个新的线程处理socket链路
                new Thread(new TimeServerHandler(socket)).start();
            }
        } finally {
            if (serverSocket != null) {
                serverSocket.close();
            }
        }
    }
}


import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.Socket;
import java.util.Date;

public class TimeServerHandler implements Runnable {
    private Socket socket;

    public TimeServerHandler(Socket socket) {
        this.socket = socket;
    }

    @Override
    public void run() {
        BufferedReader in = null;
        PrintWriter out = null;

        try {
            in = new BufferedReader(new InputStreamReader(this.socket.getInputStream()));
            out = new PrintWriter(this.socket.getOutputStream(), true);

            String currTime = null;
            String body = null;

            while (true) {
                body = in.readLine();
                if (body == null) {
                    break;
                }

                System.out.println("time server receive: " + body);
                currTime = new Date(System.currentTimeMillis()).toString();
                out.println(currTime);
            }
        } catch (Exception e) {
            // ignore
        } finally {
            if (in != null) {
                try {
                    in.close();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }
            }
            if (out != null) {
                out.close();
            }
            if (this.socket != null) {
                try {
                    this.socket.close();
                } catch (IOException e2) {
                    e2.printStackTrace();
                }
            }
            this.socket = null;
        }
    }
}
```

```java
//客户端代码

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.Socket;

public class TimeClient {
    public static void main(String[] args) {
        int port = 8080;

        Socket socket = null;
        BufferedReader in = null;
        PrintWriter out = null;

        try {
            socket = new Socket("127.0.0.1", port);
            in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            out = new PrintWriter(socket.getOutputStream(), true);

            out.println("query time");
            System.out.println("send query time request");
            String rep = in.readLine();
            System.out.println("curr time is " + rep);
        } catch (Exception e) {
            // ignore
        } finally {
            if (in != null) {
                try {
                    in.close();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }
            }
            if (out != null) {
                out.close();
            }
            if (socket != null) {
                try {
                    socket.close();
                } catch (IOException e2) {
                    e2.printStackTrace();
                }
            }
        }
    }
}
```

为了减少线程创建销毁的开销，可以利用线程池来改进服务端代码，但是如果客户端并发访问量大，超过服务端最大线程数，或者服务端所有可用线程都被长时间占用，客户端连接请求会出现大量连接超时。

```java
public class TimeServer {
  private static int port = 8080;

  public static void main(String[] args) throws IOException {
    ServerSocket serverSocket = null;
    try {
      serverSocket = new ServerSocket(port);
      System.out.println("server starts in port: " + port);

      Socket socket = null;
      // 创建一个线程池处理socket链路
      TimeServerHandlerPool pool = new TimeServerHandlerPool(10, 1000);
      while (true) {
        // 监听来自客户端的连接，主线程阻塞在accept操作上
        socket = serverSocket.accept();
        pool.execute(new TimeServerHandler(socket));
      }
    } finally {
      if (serverSocket != null) {
        serverSocket.close();
      }
    }
  }
}

public class TimeServerHandlerPool {
  private ExecutorService executorService;

  public TimeServerHandlerPool(int poolSize, int queueSize) {
    executorService = new ThreadPoolExecutor(
        8,
        poolSize, 120,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<Runnable>(queueSize)
    );
  }

  public void execute(Runnable task) {
    executorService.execute(task);
  }
}
```

## 二、NIO

### 概念

> Java NIO（ New IO） 是从Java 1.4版本开始引入的一个新的IO API，可以替代标准的Java IO API。NIO与原来的IO有同样的作用和目的，但是使用的方式完全不同， NIO支持面向缓冲区的、基于通道的IO操作。 NIO将以更加高效的方式进行文件的读写操作。


NIO与普通IO的主要区别:

| IO                      | NIO                         |
| ----------------------- | --------------------------- |
| 面向流(Stream Oriented) | 面向缓冲区(Buffer Oriented) |
| 阻塞IO(Blocking IO)   | 非阻塞IO(Non Blocking IO) |
| (无)                   | 选择器(Selectors)        |
{: .table.table-bordered }

NIO的优点：

* 可以实现异步IO
* 基于Channel和Buffer进行操作，数据从Channel读取到Buffer中，或者从Buffer写入到Channel
* 引入Selector多路复用技术，单个线程利用Selector可以监听多个通道事件，减小系统开销

### 2.1 Channel

Channel种类：

* FileChannel

```java
  try(RandomAccessFile randomAccessFile = new RandomAccessFile("D:\\test.txt","rw");){
    FileChannel fileChannel = randomAccessFile.getChannel();

    //region 先写入数据
    String data = "New string to write..." + System.currentTimeMillis();
    ByteBuffer writeBuffer = ByteBuffer.allocate(1024);
    //进入写模式
    writeBuffer.clear();
    writeBuffer.put(data.getBytes());
    //进入读模式
    writeBuffer.flip();
    while(writeBuffer.hasRemaining()){
        fileChannel.write(writeBuffer);
    }
    //endregion

    //region 读取刚才写入的数据
    long position = fileChannel.position();
    long size = fileChannel.size();
    fileChannel.position(0);
    long newPosition = fileChannel.position();

    ByteBuffer readBuffer = ByteBuffer.allocate((int)size);
    //进入写模式
    readBuffer.clear();
    int bytesRead = fileChannel.read(readBuffer);
    byte[] dataBytes = new byte[bytesRead];
    //进入读模式
    readBuffer.flip();
    while(readBuffer.hasRemaining()){
        readBuffer.get(dataBytes);
    }
    String dataRead = new String(dataBytes);
    System.out.println(dataRead);
    //endregion

} catch (IOException e) {
    e.printStackTrace();
}
```

* SelectableChannel
  * DatagramChannel
  * SocketChannel
  * ServerSocketChannel
   > 可以监听新进来的TCP连接，对每一个新进来的连接都会创建一个SocketChannel

Channel之间可以通过以下方法来互相传输数据：

* transferFrom(position, count, fromChannel)
* transferTo(position, count, toChannel)

### 2.2 Buffer

只能容纳特定数据类型，Buffer种类：

* ByteBuffer
* CharBuffer
* DoubleBuffer
* FloatBuffer
* IntBuffer
* LongBuffer
* ShortBuffer
* MappedByteBuffer

Buffer提供的属性：
* capacity 容量
* limit 写模式下代表能写入多少数据(此时limit=capacity)，切换到读模式后默认limit=之前写模式的position
* position 写模式代表当前可写入数据的起始位置，读模式代表当前可读的起始位置

Buffer提供的方法：

* flip()  从写模式转换为读模式，position设为0，limit设为之前position的位置
* rewind() 不改变模式，将position设回0
* clear()  从读模式转换为写模式，但是不会真正清除数据

    ```java
    CharBuffer charBuffer = CharBuffer.allocate(8);
    System.out.println(charBuffer.capacity());  //8
    System.out.println(charBuffer.limit());     //8
    System.out.println(charBuffer.position());  //0

    charBuffer.put('a');
    charBuffer.put('b');
    charBuffer.put('c');
    System.out.println(charBuffer.position());  //3

    charBuffer.flip();//写模式转到读模式
    System.out.println(charBuffer.limit());     //3
    System.out.println(charBuffer.position());  //0

    System.out.println("取出第一个元素是："+charBuffer.get());  //a
    System.out.println("取完第一个元素之后，position的变化："+charBuffer.position());  //1

    charBuffer.clear();//取完第一个元素之后，执行clear方法，重新为写操作做准备
    System.out.println(charBuffer.position());  //0
    System.out.println(charBuffer.get());    //a 事实证明clear之后，之前的元素还在,并未被清空。当有新的元素进来时才会将其覆盖。
    System.out.println(charBuffer.get(1));   //b
    System.out.println(charBuffer.get(2));   //c
    System.out.println(charBuffer.limit());     //8

    System.out.println("---------------------");
    charBuffer.clear();
    charBuffer.put('d');
    charBuffer.put('e');
    System.out.println(charBuffer.position());  //2
    System.out.println(charBuffer.limit());     //8
    System.out.println(charBuffer.get(0));       //d
    System.out.println(charBuffer.get(1));      //e
    System.out.println(charBuffer.get(2));      //c 原先的c还在，并没有被清除掉
    ```

* compact()

    > * 一旦读完Buffer中的数据，需要让Buffer准备好再次被写入。可以通过clear()或compact()方法来完成。
    > * 如果调用的是clear()方法，position将被设回0，limit被设置成 capacity的值。换句话说，Buffer 被清空了。Buffer中的数据并未清除，只是这些标记告诉我们可以从哪里开始往Buffer里写数据。
    > * 如果Buffer中有一些未读的数据，调用clear()方法，数据将“被遗忘”，意味着不再有任何标记会告诉你哪些数据被读过，哪些还没有。
    > * 如果Buffer中仍有未读的数据，且后续还需要这些数据，但是此时想要先先写些数据，那么使用compact()方法。
    > * compact()方法将所有未读的数据拷贝到Buffer起始处。然后将position设到最后一个未读元素正后面。limit属性依然像clear()方法一样，设置成capacity。现在Buffer准备好写数据了，但是不会覆盖未读的数据。

* mark()
* reset()
  
   > 标记Buffer中的一个特定position。之后可以通过调用Buffer.reset()方法恢复到这个position。

### 2.3 Selector

> Selector多路复用器提供选择已就绪的任务的能力，Selector会不断轮询注册在其上的Channel，如果某个Channel发生读或写操作，这个Channel就会处于就绪状态，会被Selector轮询出来，通过SelectionKey可以获取就绪的Channel进行后续IO操作。JDK NIO使用epoll替代select/poll，基于事件驱动而不是轮询所有fd状态(时间复杂度O(1))，且没有最大连接句柄限制(select在32位机器上限制1024，64位机器上限制2048，poll基于链表存储也没有限制，但是poll跟select一样需要轮询channel，时间复杂度是O(n))，可以处理成千上万个客户端(1G内存可以处理10w)。

> select，poll，epoll都是IO多路复用的机制，I/O多路复用就是通过一种机制，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知应用程序进行相应的读写操作。但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。

![nio_selector模型.png](../images/io/nio_selector模型.png)

创建Selector：

```java
Selector selector = Selector.open();
```

注册Channel到selector上，监听Channel的4个事件：

```
SelectionKey.OP_CONNECT
SelectionKey.OP_ACCEPT
SelectionKey.OP_READ
SelectionKey.OP_WRITE

//可以同时监听多个事件
SelectionKey.OP_READ | SelectionKey.OP_WRITE
```

Selector方法：
* int select()  //阻塞到至少有一个通道在你注册的事件上就绪了，可以通过wakeUp()方法唤醒
* int select(long timeout)  //阻塞直到超时
* int selectNow()  //不阻塞，直接返回就绪通道数量

Selector示例：

```java
ServerSocketChannel serverChannel = ServerSocketChannel.open();// 打开一个未绑定的serversocketchannel   
Selector selector = Selector.open();// 创建一个Selector
serverChannel.configureBlocking(false);//设置非阻塞模式
//绑定端口，设置backlog为1024(客户端的连接请求FIFO队列的队列长度，超过则拒绝连接)
serverChannel.socket().bind(new InetSocketAddress(8888),1024);
serverChannel.register(selector, SelectionKey.OP_READ);//将ServerSocketChannel注册到Selector

while(true) {
  int readyChannels = selector.select();
  if(readyChannels == 0) continue;
  Set selectedKeys = selector.selectedKeys();
  Iterator keyIterator = selectedKeys.iterator();
  while(keyIterator.hasNext()) {
    SelectionKey key = keyIterator.next();
    if(key.isAcceptable()) {
        // a connection was accepted by a ServerSocketChannel.
    } else if (key.isConnectable()) {//连接就绪
        // a connection was established with a remote server.
    } else if (key.isReadable()) {//读就绪
        // a channel is ready for reading
    } else if (key.isWritable()) {//写就绪
        // a channel is ready for writing
    }
    keyIterator.remove();
  }
}
```

### 2.4 Pipe

线程之间的单向数据传输管道

![pipe](../images/io/pipe.bmp)

```java
public class PipeDemo {
    public static void main(String[] args) throws Exception {
        Pipe pipe = Pipe.open();
        new Thread(new PipTask(pipe)).start();
        Scanner scanner = new Scanner(System.in);
        try {
            while (true) {
                String input = scanner.next();
                pipe.sink().write(ByteBuffer.wrap(input.getBytes()));
            }
        } finally {
            scanner.close();
        }
    }
}

class PipTask implements Runnable {

    private Pipe pipe;

    public PipTask(Pipe pipe) {
        this.pipe = pipe;
    }

    @Override
    public void run() {
        try {
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            while (pipe.source().read(buffer) >= 0) {
                buffer.flip();
                byte[] bytes = new byte[buffer.limit()];
                for (int i = 0; buffer.hasRemaining(); i++) {
                    bytes[i] = buffer.get();
                }
                buffer.clear();
                System.out.println("Input : " + new String(bytes));
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 零拷贝，直接内存映射

[NIO效率高的原理之零拷贝与直接内存映射](https://mp.weixin.qq.com/s?__biz=MzUyNzgyNzAwNg==&mid=2247483933&idx=1&sn=d9776b9efe054b30523adbe60cb7524a&scene=21#wechat_redirect)

[关于零拷贝的一些理解](https://www.cnblogs.com/lhh-north/p/11031821.html)

* CPU copy
* DMA copy
* sendFile  java transferTo
* mmap  java MappedByteBuffer

1. 磁盘文件不需要操作直接发送出去使用transferTo

NIO的零拷贝由transferTo()方法实现。transferTo()方法将数据从FileChannel对象传送到可写的字节通道（如Socket Channel等）。在内部实现中，由native方法transferTo0()来实现，它依赖底层操作系统的支持。在UNIX和Linux系统中，调用这个方法将会引起sendfile()系统调用。

2. 需要操作文件的情况，使用直接内存mmap

NIO的直接内存是由MappedByteBuffer实现的。核心即是map()方法，该方法把文件映射到内存中，获得内存地址addr，然后通过这个addr构造MappedByteBuffer类，以暴露各种文件操作API。

由于MappedByteBuffer申请的是堆外内存，因此不受Minor GC控制，只能在发生Full GC时才能被回收。而DirectByteBuffer改善了这一情况，它是MappedByteBuffer类的子类，同时它实现了DirectBuffer接口，维护一个Cleaner对象来完成内存回收。因此它既可以通过Full GC来回收内存，也可以调用clean()方法来进行回收。

另外，直接内存的大小可通过jvm参数来设置：-XX:MaxDirectMemorySize。

NIO的MappedByteBuffer还有一个兄弟叫做HeapByteBuffer。顾名思义，它用来在堆中申请内存，本质是一个数组。由于它位于堆中，因此可受GC管控，易于回收。

## 三、AIO

> NIO 2.0引入了新的异步通道的概念，提供了异步文件通道（AsynchronousFileChannel）和异步套接字通道（AsynchronousServerSocketChannel）的实现。可以通过两种方式来获取异步操作的结果，Future或CompletionHandler。AIO是真正的异步非阻塞IO，不需要Selector来对注册通道进行轮询操作实现读写，简化了NIO编程模型。

异步文件通道示例：

```java
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousFileChannel;
import java.nio.channels.CompletionHandler;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.StandardOpenOption;
import java.util.concurrent.Future;

public class AsynchronousFileChannelDemo {
    public static void main(String[] args) throws IOException {
        //region 通过Future来读取文件
        /*Path path = Paths.get("D:\\test.txt");
        try(AsynchronousFileChannel fileChannel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)){
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            long position = 0;

            Future<Integer> operation = fileChannel.read(buffer, position);

            while(!operation.isDone());

            buffer.flip();
            byte[] data = new byte[buffer.limit()];
            buffer.get(data);
            System.out.println(new String(data));
            buffer.clear();
        }*/
        //endregion

        //region 通过CompletionHandler来读取文件
        /*Path path = Paths.get("D:\\test.txt");
        try(AsynchronousFileChannel fileChannel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)){
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            long position = 0;
            fileChannel.read(buffer, position, buffer, new CompletionHandler<Integer, ByteBuffer>() {
                @Override
                public void completed(Integer result, ByteBuffer attachment) {
                    System.out.println("result = " + result);

                    attachment.flip();
                    byte[] data = new byte[attachment.limit()];
                    attachment.get(data);
                    System.out.println(new String(data));
                    attachment.clear();
                }

                @Override
                public void failed(Throwable exc, ByteBuffer attachment) {
                    System.out.println("读取失败");
                }
            });
        }*/
        //endregion

        //region 通过future来写文件
        /*Path path = Paths.get("D:\\test.txt");
        try(AsynchronousFileChannel fileChannel = AsynchronousFileChannel.open(path, StandardOpenOption.WRITE)){
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            long position = 0;

            buffer.put("test data".getBytes());
            buffer.flip();

            Future<Integer> operation = fileChannel.write(buffer, position);
            buffer.clear();

            while(!operation.isDone());

            System.out.println("Write done");
        }*/
        //endregion

        //region 通过CompletionHandler来写文件
        Path path = Paths.get("D:\\test.txt");
        if(!Files.exists(path)){
            Files.createFile(path);
        }
        try(AsynchronousFileChannel fileChannel = AsynchronousFileChannel.open(path, StandardOpenOption.WRITE)){
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            long position = 0;

            buffer.put("test data".getBytes());
            buffer.flip();

            fileChannel.write(buffer, position, buffer, new CompletionHandler<Integer, ByteBuffer>() {

                @Override
                public void completed(Integer result, ByteBuffer attachment)  {
                    System.out.println("bytes written: " + result);
                }

                @Override
                public void failed(Throwable exc, ByteBuffer attachment) {
                    System.out.println("Write failed");
                    exc.printStackTrace();
                }
            });
        }
        //endregion
    }
}
```

异步套接字通道示例:

```java
//服务端代码
public class Server {
    private static int DEFAULT_PORT = 12345;
    private static AsyncServerHandler serverHandle;
    public volatile static long clientCount = 0;
    public static void start(){
        start(DEFAULT_PORT);
    }
    public static synchronized void start(int port){
        if(serverHandle!=null)
            return;
        serverHandle = new AsyncServerHandler(port);
        new Thread(serverHandle,"Server").start();
    }
    public static void main(String[] args){
        Server.start();
    }
}

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.util.concurrent.CountDownLatch;
public class AsyncServerHandler implements Runnable {
    public CountDownLatch latch;
    public AsynchronousServerSocketChannel channel;
    public AsyncServerHandler(int port) {
        try {
            //创建服务端通道
            channel = AsynchronousServerSocketChannel.open();
            //绑定端口
            channel.bind(new InetSocketAddress(port));
            System.out.println("服务器已启动，端口号：" + port);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    @Override
    public void run() {
        //CountDownLatch初始化
        //它的作用：在完成一组正在执行的操作之前，允许当前的线程一直阻塞
        //此处，让线程在此阻塞，防止服务端执行完成后退出
        //也可以使用while(true)+sleep
        //生产环境就不需要担心这个问题，因为服务端是不会退出的
        latch = new CountDownLatch(1);
        //用于接收客户端的连接
        channel.accept(this,new AcceptHandler());
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
//作为handler接收客户端连接
public class AcceptHandler implements CompletionHandler<AsynchronousSocketChannel, AsyncServerHandler> {
    @Override
    public void completed(AsynchronousSocketChannel channel,AsyncServerHandler serverHandler) {
        //继续接受其他客户端的请求
        Server.clientCount++;
        System.out.println("连接的客户端数：" + Server.clientCount);
        serverHandler.channel.accept(serverHandler, this);
        //创建新的Buffer
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        //异步读  第三个参数为接收消息回调的业务Handler
        channel.read(buffer, buffer, new ReadHandler(channel));
    }
    @Override
    public void failed(Throwable exc, AsyncServerHandler serverHandler) {
        exc.printStackTrace();
        serverHandler.latch.countDown();
    }
}

import java.io.IOException;
import java.io.UnsupportedEncodingException;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
public class ReadHandler implements CompletionHandler<Integer, ByteBuffer> {
    //用于读取半包消息和发送应答
    private AsynchronousSocketChannel channel;
    public ReadHandler(AsynchronousSocketChannel channel) {
        this.channel = channel;
    }
    //读取到消息后的处理
    @Override
    public void completed(Integer result, ByteBuffer attachment) {
        //flip操作
        attachment.flip();
        //根据
        byte[] message = new byte[attachment.remaining()];
        attachment.get(message);
        try {
            String expression = new String(message, "UTF-8");
            System.out.println("服务器收到消息: " + expression);
            String calrResult = null;
            try{
                calrResult = expression+"*****";
            }catch(Exception e){
                calrResult = "计算错误：" + e.getMessage();
            }
            //向客户端发送消息
            doWrite(calrResult);
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }

    //发送消息
    private void doWrite(String result) {
        byte[] bytes = result.getBytes();
        ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
        writeBuffer.put(bytes);
        writeBuffer.flip();
        //异步写数据 参数与前面的read一样
        channel.write(writeBuffer, writeBuffer,new CompletionHandler<Integer, ByteBuffer>() {
            @Override
            public void completed(Integer result, ByteBuffer buffer) {
                //如果没有发送完，就继续发送直到完成
                if (buffer.hasRemaining()) {
                    channel.write(buffer, buffer, this);
                } else{
                    //创建新的Buffer
                    ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                    //异步读  第三个参数为接收消息回调的业务Handler
                    channel.read(readBuffer, readBuffer, new ReadHandler(channel));
                }
            }
            @Override
            public void failed(Throwable exc, ByteBuffer attachment) {
                try {
                    channel.close();
                } catch (IOException e) {
                }
            }
        });
    }

    @Override
    public void failed(Throwable exc, ByteBuffer attachment) {
        try {
            this.channel.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

```java
//客户端代码
import java.util.Scanner;
public class Client {
    private static String DEFAULT_HOST = "127.0.0.1";
    private static int DEFAULT_PORT = 12345;
    private static AsyncClientHandler clientHandle;
    public static void start(){
        start(DEFAULT_HOST,DEFAULT_PORT);
    }
    public static synchronized void start(String ip,int port){
        if(clientHandle!=null)
            return;
        clientHandle = new AsyncClientHandler(ip,port);
        new Thread(clientHandle,"Client").start();
    }
    //向服务器发送消息
    public static boolean sendMsg(String msg) throws Exception{
        if(msg.equals("q")) return false;
        clientHandle.sendMsg(msg);
        return true;
    }
    @SuppressWarnings("resource")
    public static void main(String[] args) throws Exception{
        Client.start();
        System.out.println("请输入请求消息：");
        Scanner scanner = new Scanner(System.in);
        while(Client.sendMsg(scanner.nextLine()));
    }
}

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.util.concurrent.CountDownLatch;
public class AsyncClientHandler implements CompletionHandler<Void, AsyncClientHandler>, Runnable {
    private AsynchronousSocketChannel clientChannel;
    private String host;
    private int port;
    private CountDownLatch latch;
    public AsyncClientHandler(String host, int port) {
        this.host = host;
        this.port = port;
        try {
            //创建异步的客户端通道
            clientChannel = AsynchronousSocketChannel.open();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    @Override
    public void run() {
        //创建CountDownLatch等待
        latch = new CountDownLatch(1);
        //发起异步连接操作，回调参数就是这个类本身，如果连接成功会回调completed方法
        clientChannel.connect(new InetSocketAddress(host, port), this, this);
        try {
            latch.await();
        } catch (InterruptedException e1) {
            e1.printStackTrace();
        }
        try {
            clientChannel.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    //连接服务器成功
    //意味着TCP三次握手完成
    @Override
    public void completed(Void result, AsyncClientHandler attachment) {
        System.out.println("客户端成功连接到服务器...");
    }
    
    //连接服务器失败
    @Override
    public void failed(Throwable exc, AsyncClientHandler attachment) {
        System.err.println("连接服务器失败...");
        exc.printStackTrace();
        try {
            clientChannel.close();
            latch.countDown();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    //向服务器发送消息
    public void sendMsg(String msg){
        byte[] req = msg.getBytes();
        ByteBuffer writeBuffer = ByteBuffer.allocate(req.length);
        writeBuffer.put(req);
        writeBuffer.flip();
        //异步写
        clientChannel.write(writeBuffer, writeBuffer,new WriteHandler(clientChannel, latch));
    }
}

import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.util.concurrent.CountDownLatch;
public class WriteHandler implements CompletionHandler<Integer, ByteBuffer> {
    private AsynchronousSocketChannel clientChannel;
    private CountDownLatch latch;
    public WriteHandler(AsynchronousSocketChannel clientChannel,CountDownLatch latch) {
        this.clientChannel = clientChannel;
        this.latch = latch;
    }
    @Override
    public void completed(Integer result, ByteBuffer buffer) {
        //完成全部数据的写入
        if (buffer.hasRemaining()) {
            clientChannel.write(buffer, buffer, this);
        }
        else {
            //读取数据
            ByteBuffer readBuffer = ByteBuffer.allocate(1024);
            clientChannel.read(readBuffer,readBuffer,new ReadHandler(clientChannel, latch));
        }
    }

    @Override
    public void failed(Throwable exc, ByteBuffer attachment) {
        System.err.println("数据发送失败...");
        try {
            clientChannel.close();
            latch.countDown();
        } catch (IOException e) {
        }
    }
}

import java.io.IOException;
import java.io.UnsupportedEncodingException;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.util.concurrent.CountDownLatch;
public class ReadHandler implements CompletionHandler<Integer, ByteBuffer> {
    private AsynchronousSocketChannel clientChannel;
    private CountDownLatch latch;
    public ReadHandler(AsynchronousSocketChannel clientChannel,CountDownLatch latch) {
        this.clientChannel = clientChannel;
        this.latch = latch;
    }
    @Override
    public void completed(Integer result,ByteBuffer buffer) {
        buffer.flip();
        byte[] bytes = new byte[buffer.remaining()];
        buffer.get(bytes);
        String body;
        try {
            body = new String(bytes,"UTF-8");
            System.out.println("客户端收到结果:"+ body);
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }
    @Override
    public void failed(Throwable exc,ByteBuffer attachment) {
        System.err.println("数据读取失败...");
        try {
            clientChannel.close();
            latch.countDown();
        } catch (IOException e) {
        }
    }
}
```

## 四、IO模型的对比

|                   | 同步阻塞I/O | 带缓冲区的同步I/O | 非阻塞I/O                  | 异步I/O                          |
| ----------------- | ----------- | --------------------- | ----------------------------- | ---------------------------------- |
| 客户端数：I/O线程 | 1:1         | M:N（其中M可以大于N） | M:1（1个I/O线程处理多个连接） | M:0（无需额外的I/O线程，被动回调） |
| I/O类型（阻塞） | 阻塞      | 阻塞                | 非阻塞                     | 非阻塞                          |
| I/O类型（同步） | 同步      | 同步                | 同步（I/O多路复用）   | 异步                             |
| API使用难度   | 简单      | 简单                | 复杂                        | 复杂                             |
| 调试难度      | 简单      | 简单                | 复杂                        | 复杂                             |
| 可靠性         | 非常差   | 差                   | 高                           | 高                                |
| 吞吐量         | 低         | 中                   | 高                           | 高                                |
{: .table.table-bordered }

## 4.1 同步、异步/阻塞、非阻塞

### 针对IO而言

IO的操作分为两个阶段，①等待数据阶段、②将数据从内核复制到用户空间阶段，不同类型的IO操作在这两个阶段的区别如下图：

![5个IO模型比较.png](/images/io/5个IO模型比较.png)

* 阻塞IO：指用户进程在等待IO准备数据的过程中是需要Block住的(recvfrom的参数flags=0会默认保持等待)，例如BIO，应用程序在获取到指定字节的数据之前是Block住的。
![阻塞IO.png](/images/io/阻塞IO.png)
* 非阻塞IO：指用户进程在等待IO准备数据的过程中不是采取Block的方式，而是隔一段时间去检测IO缓冲区是否有数据，如果有则读取，如果没有也会立刻返回(通过recvfrom的flags参数控制，flags=MSG_DONTWAIT表示立即返回不等待)。但是在数据从内核缓冲区复制到用户空间的阶段跟阻塞IO一样，还是会Block。
![非阻塞IO.png](/images/io/非阻塞IO.png)
* 同步IO：用户进程和内核IO进程需要完全或者部分阶段保持步调一致，如图中前4种类型的IO，在IO第二阶段用户进程全部都要通过阻塞的方式来与内核IO操作保持步调一致，用户进程阻塞是保持同步的一种手段
* 异步IO：用户进程和内核IO进程的操作完全平行，内核在IO操作结束之后通过信号或回调函数的方式通知用户进程，如图中的第5种IO。

阻塞是实现同步的一种手段，如果以linux的socket而言(windows的socket函数非常类似，也有flags参数)，最终的实现方式是通过接收数据和发送数据函数实现的，flags参数可以控制是否需要则塞，默认是阻塞的：

```c
#include <sys/types.h>
#include <sys/socket.h>
ssize_t recv(int sockfd, void *buff, size_t nbytes, int flags);
ssize_t recvfrom(int sock, void *buf, size_t len, int flags, truct sockaddr *from, socklen_t *fromlen);
ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);

ssize_t send(int sockfd, const void *buff, size_t nbytes, int flags);
int sendto(int s, const void *msg, int len, unsigned int flags, const 
struct sockaddr *to , int tolen);
```

除去这4种IO，第一张图里面还有另外两个IO模型：

* IO复用模型(select/poll/epoll)：它也是一种同步非则塞模型(阻塞于select/poll函数，同时监听多个fs，但是没有阻塞于内核socket的recv函数)，
  ![IO复用.png](/images/io/IO复用.png)

  > * (1) select==>时间复杂度O(n)
  >
  > 它仅仅知道了，有I/O事件发生了，却并不知道是哪几个流（可能有一个，多个，甚至全部），我们只能无差别轮询所有流，找出能读出数据，或者写入数据的流，对他们进行操作。所以**select具有O(n)的无差别轮询复杂度**，同时处理的流越多，无差别轮询时间就越长。
  >
  > * (2) poll==>时间复杂度O(n)
  >
  > poll本质上和select没有区别，它将用户传入的数组拷贝到内核空间，然后查询每个fd对应的设备状态， **但是它没有最大连接数的限制**，原因是它是基于链表来存储的.
  >
  > * (3) epoll==>时间复杂度O(1)
  >
  > **epoll可以理解为event poll**，不同于忙轮询和无差别轮询，epoll会把哪个流发生了怎样的I/O事件通知我们。所以我们说epoll实际上是**事件驱动（每个事件关联上fd）**的，此时我们对这些流的操作都是有意义的。**（复杂度降低到了O(1)）**
  >

  >
  > select，poll，epoll都是IO多路复用的机制。I/O多路复用就通过一种机制，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。**但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的**，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。  
  >
  > epoll跟select都能提供多路I/O复用的解决方案。在现在的Linux内核里有都能够支持，其中**epoll是Linux所特有**，而select则应该是POSIX所规定，一般操作系统均有实现
  
* 信号驱动IO模型
![信号驱动IO.png](/images/io/信号驱动IO.png)

### 针对线程而言

* 线程同步：多个线程在访问临界资源的时候协调一致，同一时间只有一个线程获取临界资源，其它线程等待，目的是保证临界资源的安全，但效率较低

  > java中实现线程同步的方式有：
  >
  > * ThreadLocal
  > * synchronized
  > * wait()、notify()
  > * volatile

* 线程异步：多个线程可以同时访问临界资源，临界资源不安全

* 线程阻塞：某个线程在等待调用结果或者等待某个资源的时候挂起就是阻塞

* 线程非阻塞：与上面相反

### 针对远程调用而言

* 同步调用：客户端等待调用结果之后才返回
* 异步调用：客户端不等待调用结果直接返回，通过回调函数来接收返回信息

### 针对通信而言

### 使用场景



## 参考

[Java NIO 系列教程](http://ifeve.com/java-nio-all/)

[*****Java 网络IO编程总结（BIO、NIO、AIO均含完整实例代码）](https://blog.csdn.net/anxpp/article/details/51512200)

[Netty权威指南（第二版）](https://item.jd.com/11681556.html)

[Netty权威指南（第二版）源码](https://github.com/wuyinxian124/nettybook2)

[io-demo](https://gitee.com/qigangzhong/java-basics/tree/master/io-demo)

[彻底理解同步 异步 阻塞 非阻塞](https://www.cnblogs.com/loveer/p/11479249.html)

[*****简述同步IO、异步IO、阻塞IO、非阻塞IO之间的联系与区别](https://www.cnblogs.com/felixzh/p/10345929.html)

[*****select、poll、epoll之间的区别](https://www.cnblogs.com/aspirant/p/9166944.html)