# 1 "Hello World!"

## Introduction（简介）

RabbitMQ is a message broker: it accepts and forwards messages. You can think about it as a post office: when you put the mail that you want posting in a post box, you can be sure that Mr. Postman will eventually deliver the mail to your recipient. In this analogy, RabbitMQ is a post box, a post office and a postman.

RabbitMQ是一个消息代理：它接受并转发消息。你可以将它看成是一个邮局：当你将想发送的邮件丢进邮箱时，你就可以确定邮差最终会把这封邮件送到收件人手上。通过这个类比来看，RabbitMQ既是邮箱，又是邮局，而且还是邮差。

The major difference between RabbitMQ and the post office is that it doesn't deal with paper, instead it accepts, stores and forwards binary blobs of data ‒ _messages_.

RabbitMQ和邮局之间的主要不同点是，RabbitMQ不跟纸打交道，它只接收，存储并转发二进制的数据包—消息。

RabbitMQ, and messaging in general, uses some jargon.

RabbitMQ，一般称它为消息队列，它使用了一些术语。

_Producing means nothing more than sending. A program that sends messages is a producer_:

消息生产其实就是消息发送。发送消息的程序就是生产者：

![](http://www.rabbitmq.com/img/tutorials/producer.png)

A _queue_ is the name for a post box which lives inside RabbitMQ. Although messages flow through RabbitMQ and your applications, they can only be stored inside a _queue_. A queue is only bound by the host's memory & disk limits, it's essentially a large message buffer. Many producers can send messages that go to one queue, and many consumers can try to receive data from one _queue_. This is how we represent a queue:

队列就是相当于RabbitMQ内部的邮箱。虽然消息在传递时是流经RabbitMQ和你的应用，但它们只能被存储在某个队列里。队列大小只受限于主机内存和硬盘容量，它本质上就是个大的消息缓存。多个生产者可以往一个队列里发送消息，多个消费者可以从一个队列里获取数据。以下是我们表示一个队列的方式：

![](http://www.rabbitmq.com/img/tutorials/queue.png)

_Consuming_ has a similar meaning to receiving. A _consumer_ is a program that mostly waits to receive messages:

消息消费其实就是消息接收。等待接收消息的程序就是一个消费者：

![](http://www.rabbitmq.com/img/tutorials/consumer.png)

Note that the producer, consumer, and broker do not have to reside on the same host; indeed in most applications they don't.

注意，生产者，消费者和代理不一定都在同一个主机里；实际上，在大多数应用中，这三者都不是在同一个主机里。

## "Hello World"

In this part of the tutorial we'll write two programs in Java; a producer that sends a single message, and a consumer that receives messages and prints them out. We'll gloss over some of the detail in the Java API, concentrating on this very simple thing just to get started. It's a "Hello World" of messaging.

在本教程里，我们将写两个Java程序，其中一个是发送单条消息的生产者，另一个是消费者，它接收消息并将它们打印出来。我们将省略Java API的一些细节，专注于即将开始的东西。它是消息队列版本的“Hello World”程序。

In the diagram below, "P" is our producer and "C" is our consumer. The box in the middle is a queue - a message buffer that RabbitMQ keeps on behalf of the consumer.

在下面的图中，“P”是我们的生产者，“C”是我们的消费者。图中间的箱子是一个队列，也就是RabbitMQ给消费者用的的消息缓存：

![](https://www.rabbitmq.com/img/tutorials/python-one.png "\(P\) -&amp;gt; \[\|\|\|\] -&amp;gt; \(C\)")

> #### The Java client library（Java客户端类库）
>
> RabbitMQ speaks multiple protocols. This tutorial uses AMQP 0-9-1, which is an open, general-purpose protocol for messaging. There are a number of clients for RabbitMQ in [many different languages](http://rabbitmq.com/devtools.html). We'll use the Java client provided by RabbitMQ.
>
> RabbitMQ支持多种协议。本教程使用AMQP 0-9-1协议，它是一个开放，通用的消息队列协议。[很多编程语言](http://rabbitmq.com/devtools.html)都提供了RabbitMQ客户端。
>
> Download the [client library](http://central.maven.org/maven2/com/rabbitmq/amqp-client/4.0.2/amqp-client-4.0.2.jar) and its dependencies \([SLF4J API](http://central.maven.org/maven2/org/slf4j/slf4j-api/1.7.21/slf4j-api-1.7.21.jar) and [SLF4J Simple](http://central.maven.org/maven2/org/slf4j/slf4j-simple/1.7.22/slf4j-simple-1.7.22.jar)\). Copy those files in your working directory, along the tutorials Java files.
>
> 下载[客户端类库](http://central.maven.org/maven2/com/rabbitmq/amqp-client/4.0.2/amqp-client-4.0.2.jar)以及它的依赖（[SLF4J API](http://central.maven.org/maven2/org/slf4j/slf4j-api/1.7.21/slf4j-api-1.7.21.jar)和[SLF4J Simple](http://central.maven.org/maven2/org/slf4j/slf4j-simple/1.7.22/slf4j-simple-1.7.22.jar)）。拷贝这些文件到你的工作目录，与教程Java文件放一起。
>
> Please note SLF4J Simple is enough for tutorials but you should use a full-blown logging library like [Logback](https://logback.qos.ch/) in production.
>
> 请注意，SLF4J Simple对于本教程来说是足够的了，但在生产环境下，你应该使用像Logback这样的完整日志类库。
>
> \(The RabbitMQ Java client is also in the central Maven repository, with the groupId com.rabbitmqand the artifactId amqp-client.\)
>
> （RabbitMQ的Java客户端也在中央maven仓库里，它的groupId为com.rabbitmq，它的artifactId为amqp-client。）

Now we have the Java client and its dependencies, we can write some code.

现在我们有了Java客户端以及它的依赖包，我们可以写代码了。

### Sending（发送）

![](https://www.rabbitmq.com/img/tutorials/sending.png "\(P\) -&amp;gt; \[\|\|\|\]")

We'll call our message publisher \(sender\)Send and our message consumer \(receiver\) Recv. The publisher will connect to RabbitMQ, send a single message, then exit.

我们将把我们的消息发布者（也就是发送者）命名为Send，然后消息消费者（也就是接收者）命名为Recv。发布者将会连接到RabbitMQ，发送一条消息，然后就退出。

In [Send.java](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Send.java), we need some classes imported:

在[Send.java](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Send.java)代码里，我们需要导入一些类：

```java
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
```

Set up the class and name the queue:

编写这个类并对队列进行命名：

```java
public class Send {
  private final static String QUEUE_NAME = "hello";

  public static void main(String[] argv)
      throws java.io.IOException {
      ...
  }
}
```

then we can create a connection to the server:

然后我们创建服务器连接：

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setHost("localhost");
Connection connection = factory.newConnection();
Channel channel = connection.createChannel();
```

The connection abstracts the socket connection, and takes care of protocol version negotiation and authentication and so on for us. Here we connect to a broker on the local machine - hence the_localhost_. If we wanted to connect to a broker on a different machine we'd simply specify its name or IP address here.

Connection类对socket连接进行了抽象封装，

Next we create a channel, which is where most of the API for getting things done resides.

To send, we must declare a queue for us to send to; then we can publish a message to the queue:

```java
channel.queueDeclare(QUEUE_NAME, false, false, false, null);
String message = "Hello World!";
channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
System.out.println(" [x] Sent '" + message + "'");
```

Declaring a queue is idempotent - it will only be created if it doesn't exist already. The message content is a byte array, so you can encode whatever you like there.

Lastly, we close the channel and the connection;

```java
channel.close();
connection.close();
```

[Here's the whole Send.java class](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Send.java).

[Send.java类的完整代码在这里。](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Send.java)

> #### Sending doesn't work!（无法发送！）
>
> If this is your first time using RabbitMQ and you don't see the "Sent" message then you may be left scratching your head wondering what could be wrong. Maybe the broker was started without enough free disk space \(by default it needs at least 200 MB free\) and is therefore refusing to accept messages. Check the broker logfile to confirm and reduce the limit if necessary. The [configuration file documentation](http://www.rabbitmq.com/configure.html#config-items) will show you how to set disk\_free\_limit.
>
> 如果这是你第一次使用RabbitMQ并且你看不到打印出来的“Sent”消息，你可能会在那里苦恼着哪里出错了。也许消息代理在启动时不够磁盘空间（默认它需要200MB的空间），由此导致拒绝接收信息。如有必要，检查代理的日志文件来确认并减少所需最小磁盘空间的限制。[配置文件文档](http://www.rabbitmq.com/configure.html#config-items)里会告诉你如何设置disk\_free\_limit（最小所需磁盘空间）参数

### Receiving

That's it for our publisher. Our consumer is pushed messages from RabbitMQ, so unlike the publisher which publishes a single message, we'll keep it running to listen for messages and print them out.

![](https://www.rabbitmq.com/img/tutorials/receiving.png "\[\|\|\|\] -&amp;gt; \(C\)")

The code \(in [Recv.java](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Recv.java)\) has almost the same imports as Send:

```java
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
```

The extraDefaultConsumeris a class implementing theConsumerinterface we'll use to buffer the messages pushed to us by the server.

Setting up is the same as the publisher; we open a connection and a channel, and declare the queue from which we're going to consume. Note this matches up with the queue thatsendpublishes to.

```java
public class Recv {
  private final static String QUEUE_NAME = "hello";

  public static void main(String[] argv)
      throws java.io.IOException,
             java.lang.InterruptedException {

    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.queueDeclare(QUEUE_NAME, false, false, false, null);
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");
    ...
    }
}
```

Note that we declare the queue here, as well. Because we might start the consumer before the publisher, we want to make sure the queue exists before we try to consume messages from it.

We're about to tell the server to deliver us the messages from the queue. Since it will push us messages asynchronously, we provide a callback in the form of an object that will buffer the messages until we're ready to use them. That is what aDefaultConsumersubclass does.

```java
Consumer consumer = new DefaultConsumer(channel) {
  @Override
  public void handleDelivery(String consumerTag, Envelope envelope,
                             AMQP.BasicProperties properties, byte[] body)
      throws IOException {
    String message = new String(body, "UTF-8");
    System.out.println(" [x] Received '" + message + "'");
  }
};
channel.basicConsume(QUEUE_NAME, true, consumer);
```

[Here's the whole Recv.java class](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Recv.java).

### Putting it all together

You can compile both of these with just the RabbitMQ java client on the classpath:

```
javac -cp amqp-client-4.0.2.jar Send.java Recv.java
```

To run them, you'll need rabbitmq-client.jar and its dependencies on the classpath. In a terminal, run the consumer \(receiver\):

```
java -cp .:amqp-client-4.0.2.jar:slf4j-api-1.7.21.jar:slf4j-simple-1.7.22.jar Recv
```

then, run the publisher \(sender\):

```
java -cp .:amqp-client-4.0.2.jar:slf4j-api-1.7.21.jar:slf4j-simple-1.7.22.jar Send
```

On Windows, use a semicolon instead of a colon to separate items in the classpath.

The consumer will print the message it gets from the publisher via RabbitMQ. The consumer will keep running, waiting for messages \(Use Ctrl-C to stop it\), so try running the publisher from another terminal.

> #### Listing queues（列出队列）
>
> You may wish to see what queues RabbitMQ has and how many messages are in them. You can do it \(as a privileged user\) using the rabbitmqctl tool:
>
> 你可能希望看一下RabbitMQ有哪些队列，这些队列里有多少消息。你可以通过使用rabbitmqctl工具来查看（但你必须是个授权用户）：
>
> ```
> sudo rabbitmqctl list_queues
> ```
>
> On Windows, omit the sudo:
>
> 在Windows系统上，输入命令时要去掉sudo：
>
> ```
> rabbitmqctl.bat list_queues
> ```

Time to move on to part 2 and build a simple _work queue_.

> #### Hint
>
> To save typing, you can set an environment variable for the classpath e.g.
>
> ```
> export CP=.:amqp-client-4.0.2.jar:slf4j-api-1.7.21.jar:slf4j-simple-1.7.22.jar
> java -cp $CP Send
> ```
>
> or on Windows:
>
> ```
> set CP=.;amqp-client-4.0.2.jar;slf4j-api-1.7.21.jar;slf4j-simple-1.7.22.jar
> java -cp %CP% Send
> ```



