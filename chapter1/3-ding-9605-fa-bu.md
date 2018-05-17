# Publish/Subscribe（发布/订阅）

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

A _producer_ is a user application that sends messages.

生产者是值发送消息的应用。

A _queue_ is a buffer that stores messages.

队列是指存储消息的缓存。

A _consumer_ is a user application that receives messages.

消费者是指接收消息的应用。

The core idea in the messaging model in RabbitMQ is that the producer never sends any messages directly to a queue. Actually, quite often the producer doesn't even know if a message will be delivered to any queue at all.

RabbitMQ的消息队列模型的核心概念是：生产者从不直接往队列里发送任何消息。实际上，多数情况下生产者甚至不知道消息是否会被发送到队列里。

Instead, the producer can only send messages to an _exchange_. An exchange is a very simple thing. On one side it receives messages from producers and the other side it pushes them to queues. The exchange must know exactly what to do with a message it receives. Should it be appended to a particular queue? Should it be appended to many queues? Or should it get discarded. The rules for that are defined by the _exchange type_.

与此相反，生产者只能将消息发送到一个交换器里。交换器做的事情很简单。一方面它接收生产者发送过来的消息，另一方面它将收到的消息推入队列里。交换器必须明确对于收到的消息它该怎么处理。这条消息是否应该附加到某个特定的队列后面？这条消息是否应该附加到多个队列后面？这条消息是否应该被丢弃？这些规则都由交换器类型（exchange type）来定义。

![](https://www.rabbitmq.com/img/tutorials/exchanges.png)

There are a few exchange types available: direct, topic, headers and fanout. We'll focus on the last one -- the fanout. Let's configure a bean to describe an exchange of this type, and call it tut.fanout:

有四种交换器类型可供我们选择：直连交换器（direct），主题交换器（topic），头部交换器（headers）和广播交换器（fanout）。我们将专注于最后一个——广播交换器。我们先配置一个bean来描述这种类型的交换器，并把这个交换器命名为tut.fanout：

```java
import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

@Profile({"tut3", "pub-sub", "publish-subscribe"})
@Configuration
public class Tut3Config {

    @Bean
    public FanoutExchange fanout() {
        return new FanoutExchange("tut.fanout");
    }

    @Profile("receiver")
    private static class ReceiverConfig {

        @Bean
        public Queue autoDeleteQueue1() {
            return new AnonymousQueue();
        }

        @Bean
        public Queue autoDeleteQueue2() {
            return new AnonymousQueue();
        }

        @Bean
        public Binding binding1(FanoutExchange fanout,
            Queue autoDeleteQueue1) {
            return BindingBuilder.bind(autoDeleteQueue1).to(fanout);
        }

        @Bean
        public Binding binding2(FanoutExchange fanout,
            Queue autoDeleteQueue2) {
            return BindingBuilder.bind(autoDeleteQueue2).to(fanout);
        }

        @Bean
        public Tut3Receiver receiver() {
            return new Tut3Receiver();
        }
    }

    @Profile("sender")
    @Bean
    public Tut3Sender sender() {
        return new Tut3Sender();
    }
}
```

We ollow the same approach as in the previous two tutorials. We create three profiles, the tutorial \("tut3", "pub-sub", or "publish-subscribe"\). They are all synonyms for running the fanout profile tutorial. Next we configure the FanoutExchange as a bean. Within the "receiver" \(Tut3Receiver\) file we define four beans: two autoDeleteQueues or AnonymousQueues and two bindings to bind those queues to the exchange.

我们采用了与前面两个教程相同的方式。我们创建了三个配置组，"tut3"，“pub-sub”，或者叫“publish-subscribe”。这三个配置在运行本教程时都是等效的。接下来我们会配置一个类型为FanoutExchange的bean。在“receiver”配置组里，我们定义了四个bean：两个AnonymousQueue类型的队列，即autoDeleteQueue1和autoDeleteQueue2，及两个将队列绑定到交换器的绑定器（binding）。

The fanout exchange is very simple. As you can probably guess from the name, it just broadcasts all the messages it receives to all the queues it knows. And that's exactly what we need for fanning out our messages.

广播交换器很简单。你大概可以从名字上看出，它只是将所有它接收到的消息广播给它所知道的所有队列。广播消息这一点正是我们需要的。

> #### Listing exchanges（列出所有的交换器）
>
> To list the exchanges on the server you can run the ever useful rabbitmqctl:
>
> 你可以通过运行强大的rabbitmqctl来列出服务器上所有的交换器：
>
> ```
> sudo rabbitmqctl list_exchanges
> ```
>
> In this list there will be some amq.\* exchanges and the default \(unnamed\) exchange. These are created by default, but it is unlikely you'll need to use them at the moment.
>
> 在这个列表里，会出现一些类似于amq.开头的交换器，以及默认的（未命名）交换器。这些交换器都默认被创建，但这时你不一定会用到它们。
>
> #### Nameless exchange（匿名交换器）
>
> In previous parts of the tutorial we knew nothing about exchanges, but still were able to send messages to queues. That was possible because we were using a default exchange, which we identify by the empty string \(""\).
>
> 在前面的教程里，我们虽然对交换器一无所知，但依旧能够将消息发送到队列里。之所以能这样是因为我们使用了一个默认的交换器，而这个默认的交换器则用空字符串（""）来标识。
>
> Recall how we published a message before:
>
> 回顾一下我们之前是如何发布消息的：
>
> ```
> template.convertAndSend(fanout.getName(), "", message);
> ```
>
> The first parameter is the the name of the exchange that was autowired into the sender. The empty string denotes the default or _nameless_ exchange: messages are routed to the queue with the name specified by routingKey, if it exists.
>
> 第一个参数是被自动注入到发送者类的交换器的名字。空字符串表示该交换器是默认或者匿名的：如果路由键存在的话，消息则通过这个路由键名被路由到某个队列里：

Now, we can publish to our named exchange instead:

现在，我们可以将信息发布到我们命名好的交换器里：

```java
@Autowired
private RabbitTemplate template;

@Autowired
private FanoutExchange fanout;   // configured in Tut3Config above

template.convertAndSend(fanout.getName(), "", message);
```

From now on the fanout exchange will append messages to our queue.

从现在开始，广播交换器将会把信息附加到我们的队列里。

## Temporary queues（临时队列）

As you may remember previously we were using queues which had a specified name \(remember hello\). Being able to name a queue was crucial for us -- we needed to point the workers to the same queue. Giving a queue a name is important when you want to share the queue between producers and consumers.

就如你记住的那样，之前我们都是使用具有指定名字的队列（前面的教程里用的是hello）。命名一个队列对于我们是至关重要的——我们需要将工作者指到相同的队列上去。当你想要在生产者和消费者之间共享队列时，为队列指定一个名字是很重要的。

But that's not the case for our fanout example. We want to hear about all messages, not just a subset of them. We're also interested only in currently flowing messages not in the old ones. To solve that we need two things.

但在我们在用广播交换器时则不用这么做。我们需要收到所有的消息，而不仅仅是部分。我们也只关心当前的消息，而不是旧的那一部分。为了解决这些需求，我们需要做两件事。

Firstly, whenever we connect to Rabbit we need a fresh, empty queue. To do this we could create a queue with a random name, or, even better - let the server choose a random queue name for us.

首先，无论什么时候连接RabbitMQ，我们都需要一个新的而且是空的队列。为了做到这点，我们可以创建一个名字随机的队列，或者更好的做法是，让服务器为我们选一个随机的队列。

Secondly, once we disconnect the consumer the queue should be automatically deleted. To do this with the spring-amqp client, we defined an _AnonymousQueue_, which creates a non-durable, exclusive, autodelete queue with a generated name:

然后，一旦我们断开了消费者，队列应该被自动删除。我们可以通过spring-amqp客户端来做到这点，在配置里我们定义了一个AnonymousQueue类型的队列，它的名字是由客户端生成的，而且是非持久的，独占的，自动删除的队列：

```java
@Bean
public Queue autoDeleteQueue1() {
    return new AnonymousQueue();
}

@Bean
public Queue autoDeleteQueue2() {
    return new AnonymousQueue();
}
```

At this point our queue names contain a random queue names. For example it may look like amq.gen-JzTY20BRgKO-HjmUJj0wLg.

此时我们的队列名字是随机的。例如，队列名字可能看起来是这样的：amq.gen-JzTY20BRgKO-HjmUJj0wLg。

## Bindings（绑定器）

![](https://www.rabbitmq.com/img/tutorials/bindings.png)

We've already created a fanout exchange and a queue. Now we need to tell the exchange to send messages to our queue. That relationship between exchange and a queue is called a _binding_. In the above Tut3Config you can see that we have two bindings, one for each AnonymousQueue.

我们已经创建了一个广播交换器和一个队列。现在我们需要让交换器将消息发送到我们的队列里。用于连接交换器和队列的对象被称为绑定器（binding）。在上面的Tut3Config里，你会发现我们配置了两个绑定器，分别对应一个AnonymousQueue。

```java
@Bean
public Binding binding1(FanoutExchange fanout, 
        Queue autoDeleteQueue1) {
    return BindingBuilder.bind(autoDeleteQueue1).to(fanout);
}
```

> #### Listing bindings（列出所有的绑定）
>
> You can list existing bindings using, you guessed it,
>
> 你可以通过使用某个命令来列出所有的绑定，猜是哪个，
>
> ```
> rabbitmqctl list_bindings
> ```

## Putting it all together（代码整合）

![](https://www.rabbitmq.com/img/tutorials/python-three-overall.png)

The producer program, which emits messages, doesn't look much different from the previous tutorial. The most important change is that we now want to publish messages to our fanout exchange instead of the nameless one. We need to supply a routingKey when sending, but its value is ignored for fanout exchanges. Here goes the code for tut3.Sender.java program:

本教程里用于生产消息的生产者程序看起来与前面教程里的生产者程序没什么区别。最大的变化是我们现在想把消息发布到广播交换器里去，而不是匿名交换器。发送消息时我们需要用到路由键（routingKey），但对于广播交换器，它的值是被忽略的。以下是本教程的发送者类代码：

```java
import org.springframework.amqp.core.FanoutExchange;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;
public class Tut3Sender {

    @Autowired
    private RabbitTemplate template;

    @Autowired
    private FanoutExchange fanout;

    int dots = 0;

    int count = 0;

    @Scheduled(fixedDelay = 1000, initialDelay = 500)
    public void send() {
        StringBuilder builder = new StringBuilder("Hello");
        if (dots++ == 3) {
            dots = 1;
        }
        for (int i = 0; i < dots; i++) {
            builder.append('.');
        }
        builder.append(Integer.toString(++count));
        String message = builder.toString();
        template.convertAndSend(fanout.getName(), "", message);
        System.out.println(" [x] Sent '" + message + "'");
    }
}
```

As you see, we leverage the beans from the Tut3Config file and autowire in the RabbitTemplate along with our configured FanoutExchange. This step is necessary as publishing to a non-existing exchange is forbidden.

就如你所看到的那样，我们利用Tut3Config文件里配置好的bean，并自动注入RabbitTemplate和FanoutExchange。这一步是很有必要的，因为发布消息到不存在的交换器是不允许的。

The messages will be lost if no queue is bound to the exchange yet, but that's okay for us; if no consumer is listening yet we can safely discard the message.

如果没有队列绑定到交换器，那么消息将会丢失，但这对于我们来说是可接受的；如果没有消费者在监听队列，那么即使消息丢失也是安全的。

The code forTut3Receiver.java:

以下是发送者类的代码：

```java
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.util.StopWatch;

public class Tut3Receiver {

    @RabbitListener(queues = "#{autoDeleteQueue1.name}")
    public void receive1(String in) throws InterruptedException {
        receive(in, 1);
    }

    @RabbitListener(queues = "#{autoDeleteQueue2.name}")
    public void receive2(String in) throws InterruptedException {
        receive(in, 2);
    }

    public void receive(String in, int receiver) throws InterruptedException {
        StopWatch watch = new StopWatch();
        watch.start();
        System.out.println("instance " + receiver + " [x] Received '" + in + "'");
        doWork(in);
        watch.stop();
        System.out.println("instance " + receiver + " [x] Done in " 
            + watch.getTotalTimeSeconds() + "s");
    }

    private void doWork(String in) throws InterruptedException {
        for (char ch : in.toCharArray()) {
            if (ch == '.') {
                Thread.sleep(1000);
            }
        }
    }

}
```

Compile as before and we're ready to execute the fanout sender and receiver.

像之前那样编译，我们已经准备好要运行基于广播的发送者程序和接收者程序了。

```
mvn clean package
```

And of course, to execute the tutorial do the following:

当然，若要运行示例代码，我们还要执行以下命令行语句：

```
java -jar target/rabbit-tutorials-1.7.1.RELEASE.jar --spring.profiles.active=pub-sub,receiver 
    --tutorial.client.duration=60000
java -jar target/rabbit-tutorials-1.7.1.RELEASE.jar --spring.profiles.active=pub-sub,sender 
    --tutorial.client.duration=60000
```

Using rabbitmqctl list\_bindings you can verify that the code actually creates bindings and queues as we want. With two ReceiveLogs.java programs running you should see something like:

使用rabbitmqctl list\_bindings语句你可以验证上述代码的确按照我们所想的创建了绑定和队列。执行语句后你会看到类似于如下的信息：

```
sudo rabbitmqctl list_bindings
tut.fanout  exchange    8b289c9c-a1eb-4a3a-b6a9-163c4fdcb6c2    queue       []
tut.fanout  exchange    d7e7d193-65b1-4128-a532-466a5256fd31    queue       []
```

The interpretation of the result is straightforward: data from exchange logs goes to two queues with server-assigned names. And that's exactly what we intended.

打印结果的意思很明显：从交换器过来的消息进入了两个由服务器命名的队列。这正是我们想要的结果。

To find out how to listen for a subset of messages, let's move on to tutorial 4.

接下来我们开始教程4，一起来看看如何监听部分消息。

