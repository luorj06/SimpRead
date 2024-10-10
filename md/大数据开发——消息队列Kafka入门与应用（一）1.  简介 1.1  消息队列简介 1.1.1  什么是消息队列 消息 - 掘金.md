> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7059317355263819784)

1.  简介
------

### 1.1  消息队列简介

#### 1.1.1  什么是消息队列

消息队列，英文名：Message Queue，经常缩写为MQ。从字面上来理解，消息队列是一种用来存储消息的队列。来看一下下面的代码：

```
arduino 代码解读复制代码`// 1. 创建一个保存字符串的队列
Queue<String> stringQueue = new LinkedList<String>();

// 2. 往消息队列中放入消息
stringQueue.offer("hello");

// 3. 从消息队列中取出消息并打印
System.out.println(stringQueue.poll());` 
```

上述代码，创建了一个队列，先往队列中添加了一个消息，然后又从队列中取出了一个消息。这说明了队列是可以用来存取消息的。

我们可以简单理解消息队列就是**将需要传输的数据存放在队列中**。

#### 1.1.2  消息队列中间件

消息队列中间件就是用来存储消息的软件（组件）。举个例子来理解，为了分析网站的用户行为，我们需要记录用户的访问日志。这些一条条的日志，可以看成是一条条的消息，我们可以将它们保存到消息队列中。将来有一些应用程序需要处理这些日志，就可以随时将这些消息取出来处理。

目前市面上的消息队列有很多，例如：Kafka、RabbitMQ、ActiveMQ、RocketMQ、ZeroMQ等。

##### 1.1.2.1  为什么叫Kafka呢

Kafka的架构师jay kreps非常喜欢franz kafka（弗兰兹·卡夫卡）,并且觉得kafka这个名字很酷，因此取了个和消息传递系统完全不相干的名称kafka，该名字并没有特别的含义。

#### 1.1.3  消息队列的应用场景

##### 1.1.3.1  异步处理

电商网站中，新的用户注册时，需要将用户的信息保存到数据库中，同时还需要额外发送注册的邮件通知、以及短信注册码给用户。但因为发送邮件、发送注册短信需要连接外部的服务器，需要额外等待一段时间，此时，就可以使用消息队列来进行异步处理，从而实现快速响应。

![图片.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ad9af7b9b6414b4a8b270406a5ae426e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

##### 1.1.3.2  系统解耦

![图片.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c2ffc85247494824b250697d46e00301~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

##### 1.1.3.3  流量削峰

![图片.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce438a8b14d94adc8df9f3620ef90b6c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

##### 1.1.3.4  日志处理（大数据领域常见）

大型电商网站（淘宝、京东、国美、苏宁...）、App（抖音、美团、滴滴等）等需要分析用户行为，要根据用户的访问行为来发现用户的喜好以及活跃情况，需要在页面上收集大量的用户访问信息。

![图片.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5cf1fa8f27274d9f8c761d6302f4636e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

#### 1.1.4  生产者、消费者模型

我们之前学习过Java的服务器开发，Java服务器端开发的交互模型是这样的：

![图片.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d162ab4421654aa6b840cefb3ded9e8a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

我们之前也学习过使用Java JDBC来访问操作MySQL数据库，它的交互模型是这样的：

  ![图片.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/25f55b5e96354869a41551dcd4d59935~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

它也是一种请求响应模型，只不过它不再是基于http协议，而是基于MySQL数据库的通信协议。

而如果我们基于消息队列来编程，此时的交互模式成为：生产者、消费者模型。

![图片.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a087b127e0fe440c9ca779836794272c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)