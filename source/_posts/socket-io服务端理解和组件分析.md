---
title: socket.io服务端理解和组件分析
date: 2021-08-10 22:07:09
tags: Netty Socket.io
---



## 回顾Socket

Socket建立请求和处理数据的过程

### 一个客户端

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1614838003875-4ba0c7b8-51cb-448c-a826-f73ada176fa9.png)



整个过程服务端都是一个线程在做，客户端发来请求，服务端accpet()接受请求，read(), write()处理数据， 处理完数据关闭连接、释放线程。

### 多个客户端

如果多个客户端怎么办？

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1614838158237-31c32765-eaa7-462b-b864-7b21a0a0e90a.png?x-oss-process=image%2Fresize%2Cw_1500)

那就每一个客户端创建一个线程。新建一个线程专门用来监听，有新客户端就new一个新的线程。

我们还可以优化一下，**池化思想**。将new Thread这个过程变成， 从pool线程池里面获取，让很多线程不随着连接关闭，多次复用。

这就出现了NIO优化的地方



### NIO非阻塞思想

NIO 三个关键组件  Selector、Buffer、Channel



Selector有点像我们上面图的监听线程，他的作用就是从线程池里面找到空闲的线程来处理客户端的请求。

Selector充当中间分配任务的角色，所有客户端线程、服务端线程都注册到我这，我帮你们进行分配，进行多路复用。

线程池里面的线程会轮询去Selector中找感兴趣的事，不会绑定某个连接上，read()、write()事件处理完就继续找下一个连接要处理的事情，等到上一个连接有新的数据再返回处理。



Channel、Buffer改变传统流的方式，能够更好管理数据，提高效率。





## Netty基础概念和代码

Netty就是对NIO的使用进行了简化，又优化了很多地方。但是核心思想还是不变的，还是采用Channel和Buffer这种结构。

Netty将Channel中加入ChannelHandler这种组件，它负责调度应用程序的处理逻辑，并驱动数据和事件经过网络层。简单来说就是加入了带有方向的拦截器，用来处理数据。

Buffer也变成了ByteBuf。

为了简化nio使用，Netty使用了**BootStrap启动器**用来辅助进行建立**socket连接**、**创建Channel通道**等等这些操作，设计模式用的是构建者模式。



同时还有**EventLoop**用来管理所有的Channel，这里的**EventLoop**可以理解为上面图的**监控线程**、或者是**工作线程**这种线程。EventLoopGroup就是线程池，所以我们的工作线程要有线程池，这就是Netty代码里面的 NioEventLoopGroup **work** = new NioEventLoopGroup(); 同样，如果监控线程如果监控的是一个端口可以只用一个线程、如果是多个端口那就要用线程池了。所以在Netty中就出现了boss这种线程池（NioEventLoopGroup **boss** = new NioEventLoopGroup(); ）



Netty的学习首先要搞清楚它的架构、设计模式。

清楚他的各个组件的作用和关系，如图：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1614835289260-6477f02f-b166-43dc-9370-441899d5f036.png?x-oss-process=image%2Fresize%2Cw_1500)

如果你认为 ChannelPipeline 是一个拦截流经 Channel 的入站和出站事件的 Channel- Handler 实例链，是ChannelHandler的容器。



Netty在Channel里传输的数据是ByteBuf，所以我们要将客户端和服务端所接受的数据都转化为ByteBuf，这就涉及到了一个Netty的重要部分，编码和解码。



还有就是ChannelHandlerContext，是ChannelHandler上下文内容，ChannelHandler处理完的数据都是ChannelHandlerContext来传输下一个ChannelHandler。ChannelHandlerContext 代表了 ChannelHandler 和 ChannelPipeline 之间的关 联，每当有 ChannelHandler 添加到 ChannelPipeline 中时，都会创建 ChannelHandler- Context。ChannelHandlerContext 的主要功能是管理它所关联的 ChannelHandler 和在 同一个 ChannelPipeline 中的其他 ChannelHandler 之间的交互。



使用Netty大致的过程就是：

1. 创建Bootstrap启动器
2. 创建监控线程组boss和工作线程组work。（客户端不用boss)
3. 用Bootstrap建立Channel，添加ChannelHanlder拦截器
4. 客户端用Bootstrap连接服务端。服务端用Bootstrap绑定端口开启监听

### 服务端



```
public class NettyServer {

    public static void main(String[] args) throws InterruptedException {

        ServerBootstrap bootstrap = new ServerBootstrap();

        NioEventLoopGroup work = new NioEventLoopGroup();
        NioEventLoopGroup boss = new NioEventLoopGroup();

        bootstrap.group(boss, work)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new DecoderHandler());
                        ch.pipeline().addLast(new ServerInHandler());
                    }
                });

        ChannelFuture future = bootstrap.bind(1000).addListener((listener) -> {
            if (listener.isSuccess()) {
                System.out.println("服务器启动成功");
            }
        });
    }

}
```

### 客户端



```
# NettyClient.java
/**
 * Netty客户端
 */
public class NettyClient {

    Bootstrap bootstrap = new Bootstrap();

    EventLoopGroup work;

    public boolean connect(String inetHost, Integer inetPort) throws InterruptedException {
        work = new NioEventLoopGroup();
        bootstrap.group(work)
                .channel(NioSocketChannel.class) //
                .handler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new EncoderHandler());
                        ch.pipeline().addLast(new ClientInHandler());
                    }
                });
            bootstrap.connect(inetHost, inetPort).addListener((listener) -> {
                if (listener.isSuccess()) {
                    System.out.println("连接成功");
                    ChannelFuture future = (ChannelFuture) listener;
                    startConsoleThread(future.channel());  // 开启控制台线程
                }
            });
            return true;

    }
}
```



## socketio实例分析

分析的是github上socketio这个项目 https://github.com/mrniko/netty-socketio/

Socket.IO server implemented on Java. Realtime java framework

用netty实现的实时网络通信





### 模块分析

#### Namespace

在一个命名空间空间的所有客户端都是共享的

那共享的都有什么呢？一个一个分析

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1614920618711-1ada87d2-c22c-424a-a609-0c98624dbbfa.png?x-oss-process=image%2Fresize%2Cw_1500)





先看ScannerEngine类内容

```
# ScannerEngine.class
    
public void scan(Namespace namespace, Object object, Class<?> clazz)
            throws IllegalArgumentException {
        Method[] methods = clazz.getDeclaredMethods();

        // 类不是Object类型的
        if (!clazz.isAssignableFrom(object.getClass())) {
            for (Method method : methods) {
                for (AnnotationScanner annotationScanner : annotations) {
                    Annotation ann = method.getAnnotation(annotationScanner.getScanAnnotation());
                    if (ann != null) {
                        annotationScanner.validate(method, clazz);

                        Method m = findSimilarMethod(object.getClass(), method);
                        if (m != null) {
                            annotationScanner.addListener(namespace, object, m, ann);
                        } else {
                            log.warn("Method similar to " + method.getName() + " can't be found in " + object.getClass());
                        }
                    }
                }
            }
            // -----------------主要看这段--------------------------
        } else {
            for (Method method : methods) {
                for (AnnotationScanner annotationScanner : annotations) {
                    Annotation ann = method.getAnnotation(annotationScanner.getScanAnnotation());
                    if (ann != null) {
                        annotationScanner.validate(method, clazz);
                        makeAccessible(method);
                        annotationScanner.addListener(namespace, object, method, ann);
                    }
                }
            }
         // -----------------end--------------------------
            if (clazz.getSuperclass() != null) {
                scan(namespace, object, clazz.getSuperclass());
            } else if (clazz.isInterface()) {
                for (Class<?> superIfc : clazz.getInterfaces()) {
                    scan(namespace, object, superIfc);
                }
            }
        }

    }
```

先说这段代码干嘛的，去扫描类里面的方法，有没有带有这三个注解的方法，如果有的话，把相应的下图三个监听器添加到相应命名空间下的执行队列中。



![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1614921713851-7997ddad-d955-4573-b8d9-0967228615f4.png)![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1614921813582-a02626b3-2473-42f9-832a-0424577f9a92.png)

所以ScannerEngine类就是扫描注解，然后添加监听器。加个上面三个注解的方法都会添加到监听器中。

NameSpace这个对象拿着ScannerEngine引擎去查找带注解的方法，并添加到NameSpace中。



eventListeners、connectListeners、disconnectListeners

这三个就不要特别说明了，是三个不同类别的监听器，分别在不同时期进行执行。

pingListeners也是同理



allClients是在这个命名空间下所有的客户端， SocketIOClient是客户端实体

roomClients是房间内的客户端，就算是分组把

clientRooms是房间，也就是组。可以移除客户端也可以添加。





#### NameSpaceHub

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1614923575816-c1e9f0ca-b3e7-4941-84c3-85bbe741e7c1.png)

用来操作NameSpaceHub的，主要是创建、获取移除等等，同时可以看到所有的NameSpace。



#### Protocol

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1614923846574-f407d315-a478-4bcf-878d-5cbf561b1fb3.png)

因为在使用Netty过程中我们要自定义协议格式，这里Packet就是数据协议包，AuthPacket是一个认证的协议包。

剩下都是围绕Packet的一些工具。比如PacketDecoder或者PacketEncoder这种编码解码工具、Json序列化的工具等等。

重点说下Packet包

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1614923992993-0a7ce29c-e754-48ef-bf3c-ef7b1fae200b.png?x-oss-process=image%2Fresize%2Cw_1500)

data数据、nsp命名空间、ackId唯一标识，可能用于消息确认等其他用途、type、subType都是包的类型

包的类型又分为下面这些



***#PacketType.java***

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1614924404084-776ee465-968b-4e16-ab5c-0cd87dc67832.png?x-oss-process=image%2Fresize%2Cw_1500)



类型Connect、Disconnect或者是Event或者是Ackd等等



回到上图这个attachments我也没明白，应该是可以自定义的一些字段，附件，方便扩展。同时还需要进行初始化

***#Packet.java***

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1614924500498-1170a570-9d64-40cc-89fe-535e6b8bc16c.png)



Packet包里面的数据就是data，data经过***#PacketEncoder***编码后就变了ByteBuf，Netty就可以在Channel中传输了，下图是编码的过程

***#PacketEncoder.java***

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1614925102874-bfc4aba2-6291-45c2-b661-6c9f2ac75d16.png?x-oss-process=image%2Fresize%2Cw_1500)



#### Listener

.... 未完待续

#####  