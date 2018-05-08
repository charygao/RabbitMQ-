## Publish/Subscribe（发布/订阅）

In the first tutorial we showed how to use start.spring.io to leverage Spring Initializr to create a project with the RabbitMQ starter dependency for create spring-amqp applications.

在第一个教程中，我们展示了如何通过[start.spring.io](https://start.spring.io/)上的Spring初始化手脚架来创建一个包含了RabbitMQ starter依赖的项目，并以此创建基于spring-amqp的应用。

In the previous tutorial we created a new package \(tut2\) to place our config, sender and receiver and created a work queue with two consumers. The assumption behind a work queue is that each task is delivered to exactly one worker.

在上一个教程当中，我们创建了一个新的包（tut2）来放置我们的配置类，发送者类和接收者类，并创建了一个对应着两个消费者的队列。工作队列背后的原理假设是，每个任务都发送给某个恰当的工作者。

In this part we'll implement the fanout pattern to deliver a message to multiple consumers. This pattern is known as "publish/subscribe" and is implementing by configuring a number of beans in our Tut3Config file.

在这部分教程中，我们将实现广播模式（fanout pattern），从而将一条消息发送给多个消费者。这个模式被称为“发布/订阅”，我们将在Tut3Config文件里配置一系列bean来实现这个模式。

Essentially, published messages are going to be broadcast to all the receivers.

本质上，发布的消息将被广播给所有的接收者。

## Exchanges（交换器）

In previous parts of the tutorial we sent and received messages to and from a queue. Now it's time to introduce the full messaging model in Rabbit.

在前面的教程里，我们通过一个队列来发送消息，并从这个队列里接收消息。接下来我们将介绍RabbitMQ完整的消息队列模型。

Let's quickly go over what we covered in the previous tutorials:

我们先来快速过一遍前面的教程里涉及到的内容：

A _producer _is a user application that sends messages.

生产者是值发送消息的应用。

A _queue _is a buffer that stores messages.

队列是指存储消息的缓存。

A _consumer _is a user application that receives messages.

消费者是指接收消息的应用。

The core idea in the messaging model in RabbitMQ is that the producer never sends any messages directly to a queue. Actually, quite often the producer doesn't even know if a message will be delivered to any queue at all.

RabbitMQ的消息队列模型的核心概念是：生产者从不直接往队列里发送任何消息。实际上，多数情况下生产者甚至不知道消息是否会被发送到队列里。

Instead, the producer can only send messages to an _exchange_. An exchange is a very simple thing. On one side it receives messages from producers and the other side it pushes them to queues. The exchange must know exactly what to do with a message it receives. Should it be appended to a particular queue? Should it be appended to many queues? Or should it get discarded. The rules for that are defined by the _exchange type_.





