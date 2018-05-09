## Routing（路由）

In the previous tutorial we built a simple fanout exchange. We were able to broadcast messages to many receivers.

在上一个教程里，我们构建了一个简单的广播交换器。通过它我们能将消息广播到多个接收者。

In this tutorial we're going to add a feature to it - we're going to make it possible to subscribe only to a subset of the messages. For example, we will be able to direct only messages to the certain colors of interest \("orange", "black", "green"\), while still being able to print all of the messages on the console.

在本教程里，我们将往里添加一个功能——我们准备让接收者可以只订阅部分消息。例如，我们将只要某些我们感兴趣的颜色（“橙色”，“黑色”，“绿色”）的消息，但仍能在控制台里打印出所有的信息。

## Bindings（绑定）

In previous examples we were already creating bindings. You may recall code like this in our Tut3Config file:

在之前的例子当中，我们创建了绑定。通过下面的代码片段回顾下我们的Tut3Config配置文件：

```java
@Bean
public Binding binding1(FanoutExchange fanout, 
    Queue autoDeleteQueue1) {
    return BindingBuilder.bind(autoDeleteQueue1).to(fanout);
}
```

A binding is a relationship between an exchange and a queue. This can be simply read as: the queue is interested in messages from this exchange.

交换器和队列是通过绑定连结在一起的。这种关系可以读作：这个队列对这个交换器里的消息感兴趣。

Bindings can take an extra routingKey parameter. Spring-amqp uses a fluent API to make this relationship very clear. We pass in the exchange and queue into the BindingBuilder and simply bind the queue "to" the exchange "with a routing key" as follows:

可以再传一个路由键（routingKey）参数给绑定。在这点上，Spring-amqp使用了流式API，使得队列，交换器和路由键之间的关系变得很清晰。我们将某个交换器和某个队列传给BindingBuilder，将传入的队列用某个路由键绑定到传入的交换器上，如下所示：

```java
@Bean
public Binding binding1a(DirectExchange direct, 
    Queue autoDeleteQueue1) {
    return BindingBuilder.bind(autoDeleteQueue1)
        .to(direct)
        .with("orange");
}
```

The meaning of a binding key depends on the exchange type. The fanout exchanges, which we used previously, simply ignored its value.

绑定键的含义取决与交换器的类型。我们之前使用的广播交换器就忽略了这个值。

## Direct exchange（直接交换器）

Our messaging system from the previous tutorial broadcasts all messages to all consumers. We want to extend that to allow filtering messages based on their color type. For example, we may want a program which writes log messages to the disk to only receive critical errors, and not waste disk space on warning or info log messages.

在上一个教程里，我们的消息队列系统将所有消息广播给所有的消费者。现在我们需要让消息队列系统可以基于消息的颜色类型进行消息过滤。例如，对于一个将日志消息写入磁盘的程序，我们可能只想让它接收严重错误类型的日志消息，而不是警告或者信息级别的日志消息，从而不致于浪费磁盘空间。

We were using a fanout exchange, which doesn't give us much flexibility - it's only capable of mindless broadcasting.

在上一个教程里，我们用到的广播交换器不能给我们这个灵活性，因为它只会做机械广播。

We will use a direct exchange instead. The routing algorithm behind a direct exchange is simple - a message goes to the queues whose binding key exactly matches the routing key of the message.

我们将会使用直接交换器来替换它。直接交换器背后的路由算法很简单——当消息被推入到某个队列时，这个队列绑定的键要与消息的路由键完全匹配。

To illustrate that, consider the following setup:

为了说明这一点，考虑下面的情况：

![](https://www.rabbitmq.com/img/tutorials/direct-exchange.png)

In this setup, we can see the direct exchange X with two queues bound to it. The first queue is bound with binding key orange, and the second has two bindings, one with binding key black and the other one with green.

从图中我们可以看到，有两个队列绑定了直接交换器X。第一个队列绑定时用了orange键，第二个队列用了两个，一个是black键，另一个是green键。

In such a setup a message published to the exchange with a routing key orange will be routed to queue Q1. Messages with a routing key of black or green will go to Q2. All other messages will be discarded.

在这种情况下，带有orange路由键的消息被发布到交换器时，会被路由到队列Q1。

