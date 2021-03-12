---
title: WebSocket服务端实现
date: 2021-08-10 
tags: WebSocket
---



# WebSocket服务端实现



## 1. 建立连接

当客户端通过一系列的配置字段（主机（host）、端口（port）、资源名称（resource name）和安全标记（secure））以及一个可被使用的协议（protocol）和扩展（extensions）列表来建立一个WebSocket连接



## 2. 关闭WebSocket连接

### 关闭原理

关闭tcp连接和tls会话。 tcp关闭才算***彻底***

用一个状态码 `code` （第 7.4 节）和一个可选的关闭原因 `reason` （第 7.1.6 节）来`开始 WebSocket 关闭握手`，

`WebSocket 关闭状态码`被默认为1005。

### 服务端如何关闭

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1613888491004-b3205912-ae69-4bb2-82c2-9410fdb212b5.png?x-oss-process=image%2Fresize%2Cw_1500)

如何管理所有的session 

通过map





## 3. WebSocket 代码实现

### 配置文件

```
@Configuration
@EnableWebSocket
public class WebSocketConfig{
    // 
    @Bean
    public ReverseWebSocketEndpoint reverseWebSocketEndpoint() {
        return new ReverseWebSocketEndpoint();
    }
 
    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
 
}
```

## 客户端建立WebSocket连接测试

前端建立WebSocket连接

在浏览器控制台输入下面标红的语法即可

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1613885589624-6db65e08-8609-4ab2-9353-661849c64ef1.png?x-oss-process=image%2Fresize%2Cw_1500)

⚠️：同源问题

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1613885652515-f90275c9-0050-46ec-84c3-a977abfc7101.png?x-oss-process=image%2Fresize%2Cw_1500)











## 代码分析



**ServerEndpointExporter类代码分析**

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1613891579226-535abd17-59a1-467f-9828-dbfa87e0d87b.png?x-oss-process=image%2Fresize%2Cw_1500)



annotatedEndpointClasses是记录带有@*ServerEndpoint的类的一个集合*

这里涉及到**InitializingBean**接口、**WebApplicationObjectSupport**类的使用



**WebApplicationObjectSupport类，**继承此类可以获取ApplicationContext对象，调用getWebApplicationContext()获取WebApplicationContext

```
注：spring在代码中获取bean的几种办法
方法一：在初始化时保存ApplicationContext对象 
方法二：通过Spring提供的utils类获取ApplicationContext对象 
方法三：继承自抽象类ApplicationObjectSupport 
方法四：继承自抽象类WebApplicationObjectSupport 
方法五：实现接口ApplicationContextAware 
方法六：通过Spring提供的ContextLoader
```

注：spring在代码中获取bean的几种办法

方法一：在初始化时保存ApplicationContext对象 方法二：通过Spring提供的utils类获取ApplicationContext对象 方法三：继承自抽象类ApplicationObjectSupport 方法四：继承自抽象类WebApplicationObjectSupport 方法五：实现接口ApplicationContextAware 方法六：通过Spring提供的ContextLoader



**InitializingBean**接口为bean提供了初始化方法的方式，它只包括**afterPropertiesSet方法**，凡是继承该接口的类，在初始化bean的时候会执行该方法。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2597675/1613895411231-35c1eeab-3b5d-458d-bbdf-0b629cc699c3.png?x-oss-process=image%2Fresize%2Cw_1500)





工具

http://coolaf.com/tool/chattest 在线websocket

参考：

https://www.cnblogs.com/kiwifly/p/11729304.html  session共享问题