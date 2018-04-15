---
title: rabbitmq-2-理解消息通讯及构造
date: 2018-01-15 15:32:22
tags: [rabbitMQ]
---

![jvm装载步骤](rabbitmq-2-理解消息通讯及构造/r2.png)

**在rabbitmq-1这篇博客里，是介绍如何安装erlang以及rabbitmq，和远程客户端的连接。但是我们还没理解什么是rabbitmq以及它其中的构造，这篇博客是我看完部分《RabbitMQ实战》这本书的理解。**


# 生产者和消费者模式

我们正常接触到的都是B/S或者C/S架构，都是一端发送请求，另一端处理数据。消息队列实现模式并不是这样的，举个例子来讲，我们都吃过自助餐，有一个自助餐桌，用来放厨师做出来的菜，厨师就相当于生产者，而消息就相当于放在自助餐桌里的菜，顾客就是消费者，消费者在自助餐桌上拿了一道菜，他甚至不知道这道菜是谁做的。这就起到了应用程序间解耦的作用。

# 消息

生产者创建消息，然后发布到代理服务器(rabbitmq)。消息分为两部分：有效载荷(payload)和标签(label)。有效载荷就是你想要发送的数据，它可以是任何内容。标签描述了有效载荷，并且RabbitMQ用来它来决定谁将获得信息的拷贝。举例来说，不同于TCP协议来说，当你明确了指定的发送方和接收方时，AMQP只会用标签来表述这条信息（一个交换器的名称和可选的主题标记），然后把消息交给rabbitmq。rabbit会根据标记把信息发送给感兴趣的接收方。这种通讯方式是一种"发后即忘"的单项模式。生产者会创建消息并且设置标签。

消费者连接到代理服务器上，并订阅到队列(queue)上。每当一个消息到达队列后，rabbitmq会将其发送给其中一个订阅的/监听的消费者。当消费者接收消息的时候，只会接收有效载荷。所以rabbitmq不会告诉你是谁生产了/发送了信息。就想我们刚才举得例子，顾客根本不知道这道菜是哪个厨师做的，如果想知道这道菜是谁做的，就是他在他的菜品上写了自己的名字。如果需要明确知道是谁生产的AMQP消息的话，就要看生产了是否把发送方的信息放到有效载荷中。

# 传输方式：信道

生产者生产消息，并且发送给rabbitmq；消费者从rabbitmq哪里接收消息。那么生产者是如何将信息发送给rabbitmq，消费者又怎样接收的呢，靠的是就是信道，英文名字为Channel，是不是觉得有点熟悉，NIO里面也有Channel，多个Channel被一个Selector包含。其实rabbit信道同样也是，多个channel对一个TCP连接。无论是发布消息、订阅队列或者接收消息，都是通过信道完成的。为什么不直接用TCP连接？因为对于操作系统来说，建立和销毁TCP会话是非常昂贵的开销，操作系统每秒也就能建立不多数量的TCP连接，这样如果每秒的请求有千万级的话，这种方式很快就成为性能瓶颈了。信道的好处，是建立一个TCP连接，然后里面建立成百上千的信道也不会影响操作系统，而且线程之间还能保持私密性。相当于每条电缆中有多条光纤束一样都可以传输，允许所有连接的线程通过多条光钎束同时进行传输和接收。

## java中创建信道

我们需要在连接工厂来创建一个连接，根据这个连接我们来获取信道。

``` java 
		ConnectionFactory factory=new ConnectionFactory();
        factory.setHost("192.168.14.128");
        factory.setUsername("test");
        factory.setPassword("test");
        factory.setPort(5672);
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
```


# 队列

首先，要确定的是，AMQP路由有三部分组成：交换器、队列和绑定。

## 队列的订阅方式

（1）通过AMQP的basic.consume命令订阅，这样会将信道设置为接收模式，直到取消对队列的订阅为止。订阅了消息之后，消费者在消费（或者拒绝）最近接收的消息后，就能从队列（可用的）自动接收下一条消息，如果消费者处理队列消息，并且/或者需要在信息一达到队列时就自动接受的话，你应该使用basic.consume。

（2）某些时候，你只想从队列获得单条消息而不是持续订阅。向队列请求单挑消息通过AMQP的basic.get命令实现的。这样做可以让消费者接收队列的下一条信息。如果要获得更多的信息的话，需要再次发送basic.get命令。不要将basic.get写进循环里代替basic.consume，那会影响rabbitmq的性能。

## 队列处理消息

如果至少有一个消费者订阅了队列的话，消息会立即发送给这些订阅的消费者。如果消息到达了一个无人订阅的队列，那么消息就会在队列里一直进行等待消费。

当消息有多个消费者的时候，队列收到的消息会以循环的方式发送给消费者。每条消息只会发送给一个消费者。
如果有消费者A，B同时订阅了seed队列的话，消息分发是这样的。

**消息one到达seed队列**

**rabbitmq将信息发送给消费者A**

**消费者确认接收到了消息one**

**rabbitmq把消息one从seed队列移除**

**消息two到达seed队列**

**rabbitmq把消息two发送给消费者B**

**消费者B确认接收到了消息two**

**rabbitmq把消息two从seed队列中移除**

我们看到消费者A和消费者B都对消息进行了确认。消费者接收到的每一条消息都必须进行确认。消费者必须通过AMQP的basic.ack命令显示地想RabbitMQ发送一个确认，或者在订阅到队列的时候就将auto_ack命令设置为true。当设置了auto_ack时，一旦消费者接收消息，RabbitMQ会自动视其确认了消息。需要记住的是，消费者对消息的确认和告诉生产者消息已经被接收了这两件事毫无关联。因此，消费者通过命令告诉RabbitMQ它已经正确地接收到了消息，同时RabbitMQ才能安全地把消息从队列中删除。

如果消费者收到一条消息，然后确认之前从Rabbit断开连接（或者从队列取消订阅），RabbitMQ会认为这条消息没有分发，然后重新分发给下一个订阅的消费者。如果你的应用程序崩溃了，这样做可以确保消息会被发送给另一个消费者进行处理。另一方面，如果应用程序有bug而忘记确认的话，RabbitMQ将不会给该消费者发送更多的信息了。这是因为在上一条消息确认之前，RabbitMQ会认为这个消费者没有准备好接收下一条消息。我们可以好好利用这一点，如果处理消息的内容非常耗时，则你得应用程序可以延迟确认该消息，知道消息处理完成。这样可以防止RabbitMQ持续不断的消息涌向你的应用而导致过载。

## java中创建队列

在java可以使用信道来创建队列。

``` java 
	/**
	* String queue:声明的队列名称
	* boolean durable：是否队列持久化
	* boolean exclusive：是否为私有
	* boolean autoDelete：最后一个消费者取消订阅的时候，对列是否自动移出
	* Map<String,Object> arguments
	**/
	channel.queueDeclare(QUEUE_NAME, false, false, false, null);
```

# 交换器和绑定

根据我们之前说的那样，消费者从队列中获取消息，可以消息是怎么样传到队列里的呢？是通过交换器和绑定。当你想将消息传递给队列的时候，需要通过消息发送给交换器来完成。然后，根据确定的规则，RabbitMQ会决定消息该投递给哪个队列。这个规则被称为路由键。队列通过路由器绑定到交换器。当你把消息发动给代理服务器时，消息将拥有一个路由键，即使是空的，RabbitMQ也会将其和绑定使用的路由器进行匹配。如果想匹配的话，那么消息就会进入队列中。如果路由的消息不匹配任何绑定模式的话，消息将进入黑洞。

## 交换器

服务器会根据路由键消息从交换器路由队列，但是它如何处理投递到多个队列的情况的呢？协议中定义的不同类型的交换器发挥了作用。一共有四种类型：direct、fanout、topic和headers。每一种类型实现了不同的路由算法。由于我是看《RabbitMQ实战》这本书进行学习，作者提到headers交换器允许你匹配AMQP消息的header而非路由键。除此之外，headers交换器和direct交换器完全一样，但是性能会差很多。因此它不太实用，并且现在几乎再也用不到了，那么我们就不介绍了。

### direct交换器

**如果路由键匹配的话，消息就被投递到对应的队列。**

direct交换器就是路由键匹配的话，消息就被投递到对应的队列。服务器必须实现direct类型的交换器，并且包含了一个空白字符串的默认交换器。当你声明一个队列的时候，他就自动绑定到这个默认的交换器上，并以队列名称作为路由键。

队列与交换器进行绑定，并且向发送消息

``` java
	/**
	* String exchange:交换器名称
	* String routingKey：路由键
	* AMQP.BasicProperties props：
	* byte[] body：发送的消息的内容
	**/
	basicPublish(String exchange, String routingKey, AMQP.BasicProperties props, byte[] body)

	
```

向默认的交换器,以及路由键发送消息，因为direct交换器中，路由键与队列是一对一的，路由键的名称就是队列的名称。所以在在这个发送消息的时候虽然没有体现队列，但是已经制定了你的数据传给了哪个队列，这是redict交换器中特有的，需要注意一下。

``` java 
channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
```

如果想更换交换器，我们可以设置自己的交换器。使用

``` java 
	/**
	* String exchange:交换器名称
	* BuiltinExchangeType type：交换器类型
	* 
	**/
exchangeDeclare(String exchange, BuiltinExchangeType type)
```

### fanout交换器

这个类型的交换器将收到的信息广播绑定到队列上。当你发送一条信息到fanout交换器的时候，他会把消息投递给所有附加在此交换器上的队列。

这个交换器我觉得应用场景会很多。假如说现实生活中，你有一张银行卡，你开通了手机银行，同时你的银行卡也绑定了支付宝、微信。当你使用这张卡完成支付后，那么需要手机银行给你发送一条短信，我支付了多少钱。同时支付宝、微信也会在自己的软件中进行通知。这就可以使用fanout交换器，让三个队列分别与fanout的交换器进行绑定，不用改生产者的任何代码，就可以解决这个问题。

### topic交换器

topic交换器允许你实现有趣的消息通讯场景，它使得来自不同源头的消息能够到达同一队列。如果拿Web应用程序作为示例。我们拥有不同的日志级别，error、info、warning等。于此同时，你的应用程序分为一下几个模块，users、product、org等。如果你在对用户模块进行操作的时候，你想报告一个error的话，则可以这样写。

``` java 
channel.basicpublic("logs-exchanges","error-users",null,message);
```

加入你声明了一个user-errors的队列，你可以将其绑定到交换器上来接收消息，如下所示。

``` java 
queueBind(String queue, String exchange, String routingKey)
```

``` java 
channel.queueBind("user-errors","logs-exchanges","error-users");
```

到目前为止，这看起来跟direct交换器很像。但是如果你想用一个队列监听users模块的所有错误级别呢，这就这需要使用topic交换器了。

``` java 
channel.queueBind("user-logs","logs-exchanges","*.-users");
```

user-logs这个队列可以接收所有从users模块接收模块发来的所有error,warning和info的日志。如果要是一个队列接收所有模块的所有的错误的信息。

``` java 
channel.queueBind("all-logs","logs-exchanges","#");
```

这样all-logs就可以接收所有传入logs-exchanges的消息了。这时候可能会问，那岂不是不是日志的消息也会放入all-logs吗？我们建立了一个logs-exchanges这个交换器，就是专门放日志的。所以对于交换器来说，不仅要选对类型，还要根据名字来来决定他的功能性。

# 虚拟主机vhost

每一个RabbitMQ都会创建一个虚拟消息服务器，我们称之为虚拟主机(vhost)。每一个vhost本质上是一个mini版的RabbitMQ服务器，拥有自己的队列、绑定、交换器。最重要的是，它拥有自己的权限控制。他们通过在各个实例间提供逻辑上分离，允许你为不同的应用程序安全保密地运行数据。这很有用，它既能将同一Rabbit的众多客户区分开来，又可以避免队列和交换器命名的冲突。负责你可能不得不运行多个Rabbit，并忍受随之而来头疼的管理问题。相反，你可以只运行一个Rabbit，然后按需启动和关闭vhost。

vhost是AMQP概念的基础，你必须在连接时指定。由于RabbitMQ包含了开箱即用的默认的vhost:"/"，因此使用起来非常简单。如果你不需要多个vhost的话，那么就使用默认的吧。通过使用缺省的guest用户名和密码guest就可以访问默认vhost。

当你在RabbitMQ里创建一个用户，用户通常会被制定给至少一个vhost，并且只能访问被指派的vhost内的队列、交换器和绑定。当你设计消息通讯架构时，记住vhost之间是绝对隔离的。你无法将vhost A上的交换器绑定到 vhost B上的队列中。这既保证了安全性，又保证了可移植性。当你在RabbitMQ集群上创建vhost时，整个集群上都会创建该vhost。vhost不仅消除了为基础架构中的每一层运行一个RabbitMQ服务器的需要，同样也避免了每一层创建不同集群。

vhost的创建和权限控制非常独特，他们是AMQP中无法通过AMQP协议创建的基元(不同于队列，交换器和绑定)。对于RabbitMQ来说，你需要通过RabbitMQ的安装路径下的./sbin/目录中的rabbitmqctl工具来创建。

新建一个名为test_vhost的vhost

``` shell
./rabbitmqctl add_vhost test_vhost
```

![你想输入的替代文字](rabbitmq-2-理解消息通讯及构造/vhost1.png)

查看当前vhost列表

``` shell
./rabbitmqctl list_vhosts
```

![你想输入的替代文字](rabbitmq-2-理解消息通讯及构造/vhost2.png)



























