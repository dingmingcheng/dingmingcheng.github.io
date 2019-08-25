---
layout: post
title: "reactor单线程，多线程，主从的java代码实现"
date: 2018-01-23
description: "reactor实现"
author: "dingmc"
tags: 网络
---

## 前言

从最初，自己就对网络编程比较感兴趣，这段时间看了netty权威指南，收获很多。我觉得所有的知识都是要落地才能体现其价值，也为了加深自己对reactor模式的理解，自己也就敲了一遍，当然中间也遇到了很多困难，废话不多说，上代码。

以下代码使用java原生nio实现，未考虑半包，粘包等问题

### common类

主要包含客户端启动代码，之后的服务端使用同一个客户端代码，还有就是写字节函数。

``` java
//AsysWriter.java
//写字节函数，之后服务端会使用该函数
public class AsysWriter {
    public static void doWrite(SocketChannel channel, String response) throws IOException {
        byte[] bytes = response.getBytes();
        ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
        writeBuffer.put(bytes);
        writeBuffer.flip();
        channel.write(writeBuffer);
    }
}
```

``` java
//Client.java
package com.dmc.common;

import com.sun.org.apache.xpath.internal.operations.Bool;
import org.apache.ibatis.annotations.SelectKey;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.SocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

import static com.dmc.common.AsysWriter.doWrite;

public class Client implements Runnable{

    private SocketChannel clientChannel;

    private Selector selector;

    private volatile Boolean connected = false;

    private ReentrantLock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();
    public Client() {
        try {
            clientChannel = SocketChannel.open();
            selector = Selector.open();
            clientChannel.configureBlocking(false);
            clientChannel.register(selector, SelectionKey.OP_CONNECT);
            clientChannel.connect(new InetSocketAddress("127.0.0.1", 9090));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    public void run() {
        try {

            Set<SelectionKey> selectionKeys = null;
            while (true) {
                selector.select();
                selectionKeys = selector.selectedKeys();

                for (Iterator<SelectionKey> it = selectionKeys.iterator(); it.hasNext(); ) {
                    SelectionKey key = it.next();
                    it.remove();

                    SocketChannel sc = (SocketChannel) key.channel();
                    if (key.isConnectable()) {
                        lock.lock();
                        try {
                            System.out.println("client connect...");
                            if (sc.finishConnect()) ;
                            else System.exit(1);
                            sc.register(selector, SelectionKey.OP_READ);
                            this.connected = true;
                            System.out.println("this.connect is " + this.connected);
                            condition.signal();
                        } finally {
                            lock.unlock();
                        }
                    } else if (key.isReadable()) {
                        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                        int length = sc.read(byteBuffer);
                        if (length > 0) {
                            byteBuffer.flip();
                            byte[] ans = new byte[byteBuffer.remaining()];
                            byteBuffer.get(ans);
                            String response = new String(ans, "UTF-8");
                            System.out.println("get the result from server : " + response);
                        }
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void sendMsg(String sendString) throws Exception {
        lock.lock();
        try {
            if (!this.connected) {
                condition.await();
            }
            clientChannel.register(selector, SelectionKey.OP_READ);
            doWrite(clientChannel, sendString);
        } finally {
            lock.unlock();
        }
    }
}
```



``` java
//ClientStart.java
//客户端启动代码
package com.dmc.common;

import java.util.Scanner;

public class ClientStart {
    public static void main(String[] args) throws Exception {
        for (int i = 0; i < 1; i ++) {
            Client client1 = new Client();
            Thread thread = new Thread(client1);
            thread.start();

            Scanner in = new Scanner(System.in);
            while (in.hasNextLine()) {
                String str = in.nextLine();
                client1.sendMsg(str);
            }
        }
    }
}
```



### 单线程

![](/img/in-post/reactor/pic4.png)

``` java
//Server.java
package dmcOneThread;

import org.springframework.expression.spel.ast.Selection;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;

import static com.dmc.common.AsysWriter.doWrite;

public class Server {
    public static void main(String[] args){
        try {
            ServerSocketChannel serverSocketChannel;
            Selector selector;
            Selector selector1;
            Set<SelectionKey> selectionKeys = null;

            serverSocketChannel = ServerSocketChannel.open();
            selector = Selector.open();
            selector1 = Selector.open();

            serverSocketChannel.configureBlocking(false);
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
            serverSocketChannel.register(selector1, SelectionKey.OP_ACCEPT);
            serverSocketChannel.bind(new InetSocketAddress("127.0.0.1", 9090));

            System.out.println("server is start");
            while (true) {
                selector.select();
                selectionKeys = selector.selectedKeys();
                for (Iterator<SelectionKey> it = selectionKeys.iterator(); it.hasNext(); ) {
                    SelectionKey key = it.next();
                    it.remove();

                    if (key.isAcceptable()) {
                        ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
                        SocketChannel client = ssc.accept();
                        client.configureBlocking(false);
                        client.register(selector, SelectionKey.OP_READ);
                        System.out.println("connected...");
                    }
                    else if (key.isReadable()) {
                        SocketChannel sc = (SocketChannel) key.channel();
                        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                        int readBytes = sc.read(byteBuffer);
                        if (readBytes > 0) {
                            byteBuffer.flip();
                            byte[] bytes = new byte[byteBuffer.remaining()];
                            byteBuffer.get(bytes);
                            String response = new String(bytes, "UTF-8");
                            System.out.println("GET, the request is " + response + " it's from " + sc.getRemoteAddress());
                            String result = response + " 's result";
                            System.out.println("send result...");
                            doWrite(sc, result + "");
                        }
                        //链路已经关闭，释放资源
                        else if (readBytes < 0) {
                            key.cancel();
                            sc.close();
                        }
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

测试结果(之后就不放测试结果了，都是一样的)：

![](/img/in-post/reactor/pic1.png)

![](/img/in-post/reactor/pic2.png)

![](/img/in-post/reactor/pic3.png)



### 多线程

![](/img/in-post/reactor/pic5.png)

Acceptor类

``` java
package ThreadGroupTest;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;

public class Acceptor {

    public static void main(String[] args) throws IOException {
        /**
         * init
         */
        ServerSocketChannel serverSocketChannel;
        Selector selector;
        Set<SelectionKey> selectionKeys = null;

        serverSocketChannel = ServerSocketChannel.open();
        selector = Selector.open();

        serverSocketChannel.bind(new InetSocketAddress("127.0.0.1", 9090));
        serverSocketChannel.configureBlocking(false);
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);


        System.out.println("server is start...");
        ReactorThreadGroup group = new ReactorThreadGroup(10);
        while (true) {
            selector.select();
            selectionKeys = selector.selectedKeys();

            Iterator<SelectionKey> it = selectionKeys.iterator();
            for (; it.hasNext(); ) {
                SelectionKey key = it.next();
                it.remove();

                if (key.isAcceptable()) {
                    ServerSocketChannel sc = (ServerSocketChannel) key.channel();
                    SocketChannel client = sc.accept();
                    client.configureBlocking(false);
                    System.out.println("connected...");
                    group.dispatch(client);
                }
            }
        }
    }
}
```



ReactorThreadGroup类

``` java
package ThreadGroupTest;

import java.nio.channels.SocketChannel;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.Set;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.ReentrantLock;
import static com.dmc.common.AsysWriter.doWrite;

public class ReactorThreadGroup {

    private static final int DEFAULT_THREADPOOL_SIZE = 10;

    private Integer threadNum;

    private AtomicInteger currentNum = new AtomicInteger(0);

    private List<Handler> handlerWorkers = new ArrayList<Handler>();

    public ReactorThreadGroup() {
        this(DEFAULT_THREADPOOL_SIZE);
    }

    public ReactorThreadGroup(Integer threadNum) {
        this.threadNum = threadNum;
        for (int i = 0; i < threadNum; i ++) {
            Handler worker = new Handler();
            handlerWorkers.add(worker);
            worker.start();
        }
        System.out.println("threadpool create success");
    }

    public void dispatch(SocketChannel sc) {
        next().add(sc);
    }

    public Handler next() {
        return handlerWorkers.get((currentNum.getAndIncrement()) % threadNum);
    }
}

```



Handler类

``` java
package ThreadGroupTest;

import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.Set;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

import static com.dmc.common.AsysWriter.doWrite;

public class Handler extends Thread{

    private ReentrantLock lock = new ReentrantLock();

    private Condition condition = lock.newCondition();

    private volatile Boolean running = false;

    private List<SocketChannel> socketChannelList;

    private Selector selector;

    private Boolean isConnected = false;

    Handler() {
        try {
            socketChannelList = new ArrayList<>();
            selector = Selector.open();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public void add(SocketChannel socketChannel) {
        lock.lock();
        try {
            selector.wakeup();
            socketChannel.register(selector, SelectionKey.OP_READ);
            socketChannelList.add(socketChannel);
            condition.signal();
            isConnected = true;
        } catch (ClosedChannelException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    @Override
    public void run() {
        lock.lock();
        try {
            while (true) {
                if (!isConnected) {
                    condition.await();
                }
                selector.select();

                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> it = selectionKeys.iterator();

                for (; it.hasNext();) {
                    SelectionKey key = it.next();
                    it.remove();
                    if (key.isReadable()) {
                        SocketChannel sc = (SocketChannel) key.channel();

                        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                        int readBytes = sc.read(byteBuffer);
                        if (readBytes > 0) {
                            System.out.println("connected to thread " + Thread.currentThread().getName());
                            byteBuffer.flip();
                            byte[] bytes = new byte[byteBuffer.remaining()];
                            byteBuffer.get(bytes);
                            String response = new String(bytes, "UTF-8");
                            System.out.println("GET, the request is " + response + " it's from " + sc.getRemoteAddress());
                            String result = response + " 's result";
                            System.out.println("send result...");
                            doWrite(sc, result + "");
                        } else if (readBytes < 0) {
                            System.out.println(Thread.currentThread().getName() + " disconnected");
                            key.cancel();
                            sc.close();
                        }
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```



### 主从

![](/img/in-post/reactor/pic6.png)

ServerStart.java

``` java
package MasterSlaveReactor;

public class ServerStart {
    public static void main(String[] args) {
        MainReactor mainReactor = new MainReactor();
        new Thread(mainReactor).start();
    }
}
```

MainReactor.java

```Java
package MasterSlaveReactor;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;
import java.util.concurrent.Executor;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicInteger;

public class MainReactor implements Runnable, CallBack{
    private static final int DEFAULT_PORT = 9090;

    private static final String DEFAULT_HOST = "127.0.0.1";

    private int port;

    private String host;

    private AtomicInteger currentNum;

    private ServerSocketChannel serverSocketChannel;

    private Selector selector;

    private ExecutorService threadPool;

    private ExecutorService subThreadPool;

    private SubReactor[] subReactors = new SubReactor[10];

    MainReactor() {
        this(DEFAULT_HOST, DEFAULT_PORT);
    }

    MainReactor(String host, Integer port) {
        this.port = port;
        this.host = host;
        currentNum = new AtomicInteger(0);
        init();
        initSubPool();
    }

    public void initSubPool() {
        subThreadPool = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 10; i ++) {
            subReactors[i] = new SubReactor();
            subThreadPool.execute(subReactors[i]);
        }
    }

    public void init() {
        try {
            threadPool = Executors.newFixedThreadPool(10);

            serverSocketChannel = ServerSocketChannel.open();
            selector = Selector.open();

            serverSocketChannel.configureBlocking(false);
            serverSocketChannel.bind(new InetSocketAddress(host, port));
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
    @Override
    public void run() {
        while (true) {
            try {
                selector.select();

                Set<SelectionKey> selectionKeyset = selector.selectedKeys();
                Iterator<SelectionKey> it = selectionKeyset.iterator();
                for (;it.hasNext();) {
                    SelectionKey key = it.next();

                    SocketChannel sc = serverSocketChannel.accept();
                    sc.configureBlocking(false);
                    accept(sc);
                    it.remove();
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    public void accept(SocketChannel sc) {
        Acceptor acceptor = new Acceptor(sc);
        acceptor.setCallBack(this);
        threadPool.execute(acceptor);
    }

    @Override
    public void Dispatch(SocketChannel socketChannel) {
        SubReactor subReactor = next();
        subReactor.register(socketChannel);
    }

    public SubReactor next() {
        return subReactors[currentNum.getAndAdd(1) % 10];
    }
}

```

Acceptor.java

``` java
package MasterSlaveReactor;

import ThreadGroupTest.ReactorThreadGroup;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;

public class Acceptor implements Runnable{

    private SocketChannel socketChannel;

    private CallBack callBack;

    public CallBack getCallBack() {
        return callBack;
    }

    public void setCallBack(CallBack callBack) {
        this.callBack = callBack;
    }

    Acceptor(SocketChannel socketChannel) {
        this.socketChannel = socketChannel;
    }

    @Override
    public void run() {
        try {
            // TODO
            System.out.println( Thread.currentThread().getName() + " " + socketChannel.getRemoteAddress() +  " connected...");
            callBack.Dispatch(socketChannel);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```

SubReactor.java

``` java
package MasterSlaveReactor;

import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.ClosedChannelException;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.Set;

import static com.dmc.common.AsysWriter.doWrite;

public class SubReactor implements Runnable{

    private List<SocketChannel> socketChannels;

    private Selector selector;

    SubReactor() {
        init();
    }

    private void init() {
        try {
            selector = Selector.open();
            socketChannels = new ArrayList<>();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public void register(SocketChannel socketChannel) {
        try {
            socketChannel.register(selector, SelectionKey.OP_READ);
            socketChannels.add(socketChannel);
        } catch (ClosedChannelException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        while (true) {
            try {
                selector.select(1000);

                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> it = keys.iterator();
                for (; it.hasNext();) {
                    SelectionKey key = it.next();
                    if (key.isReadable()) {
                        SocketChannel sc = (SocketChannel) key.channel();

                        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                        int readBytes = sc.read(byteBuffer);
                        if (readBytes > 0) {
                            System.out.println("connected to thread " + Thread.currentThread().getName());
                            byteBuffer.flip();
                            byte[] bytes = new byte[byteBuffer.remaining()];
                            byteBuffer.get(bytes);
                            String response = new String(bytes, "UTF-8");
                            System.out.println("GET, the request is " + response + " it's from " + sc.getRemoteAddress());
                            String result = response + " 's result";
                            System.out.println("send result...");
                            doWrite(sc, result + "");
                        } else if (readBytes < 0) {
                            System.out.println(Thread.currentThread().getName() + " disconnected");
                            key.cancel();
                            sc.close();
                        }
                    }
                    it.remove();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

callback接口

``` java
package MasterSlaveReactor;

import java.nio.channels.SocketChannel;

public interface CallBack {
    void Dispatch(SocketChannel socketChannel);
}
```



## 结尾

在理解时过于注重类的关系与实现，导致卡了好久，当关注重点变为线程时，就豁然开朗了。

重在实践，理论知识的掌握是为了更好得落地。“实践是检验真理的唯一标准”。

模型图来自[这里](https://www.cnblogs.com/ivaneye/p/5731432.html)，十分直观的图，十分感谢