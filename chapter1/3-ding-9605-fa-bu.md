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

A _producer_ is a user application that sends messages.

生产者是值发送消息的应用。

A _queue_ is a buffer that stores messages.

队列是指存储消息的缓存。

A _consumer_ is a user application that receives messages.

消费者是指接收消息的应用。

The core idea in the messaging model in RabbitMQ is that the producer never sends any messages directly to a queue. Actually, quite often the producer doesn't even know if a message will be delivered to any queue at all.

RabbitMQ的消息队列模型的核心概念是：生产者从不直接往队列里发送任何消息。实际上，多数情况下生产者甚至不知道消息是否会被发送到队列里。

Instead, the producer can only send messages to an _exchange_. An exchange is a very simple thing. On one side it receives messages from producers and the other side it pushes them to queues. The exchange must know exactly what to do with a message it receives. Should it be appended to a particular queue? Should it be appended to many queues? Or should it get discarded. The rules for that are defined by the _exchange type_.

与此相反，生产者只能将消息发送到一个交换器里。交换器做的事情很简单。一方面它接收生产者发送过来的消息，另一方面它将收到的消息推入队列里。交换器必须明确对于收到的消息它该怎么处理。这条消息是否应该附加到某个特定的队列后面？这条消息是否应该附加到多个队列后面？这条消息是否应该被丢弃？这些规则都有交换器类型（exchange type）来定义。

![](https://www.rabbitmq.com/img/tutorials/exchanges.png)

There are a few exchange types available: direct, topic, headers and fanout. We'll focus on the last one -- the fanout. Let's configure a bean to describe an exchange of this type, and call it tut.fanout:

有四种交换器类型可供我们选择：直接交换（direct），主体（topic），头部交换（headers）和广播（fanout）。我们将专注于最后一个——广播。我们先配置一个bean来描述这种类型的交换器，并把这个交换器命名为tut.fanout：

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

我们采用了前面两个教程相同的方式。我们创建了三个配置，"tut3"，“pub-sub”，还有“publish-subscribe”。这三个配置在运行本教程时都是等效的。接下来我们会配置一个类型为FanoutExchange的bean。在“receiver”配置里，我们定义了四个bean：两个名字为autoDeleteQueue的AnonymousQueue以及两个将队列绑定到交换器binding。

The fanout exchange is very simple. As you can probably guess from the name, it just broadcasts all the messages it receives to all the queues it knows. And that's exactly what we need for fanning out our messages.

广播交换器很简单。你大概可以从名字上看出，

> #### Listing exchanges
>
> To list the exchanges on the server you can run the ever usefulrabbitmqctl:
>
> ```
> sudo rabbitmqctl list_exchanges
> ```
>
> In this list there will be someamq.\*exchanges and the default \(unnamed\) exchange. These are created by default, but it is unlikely you'll need to use them at the moment.
>
> #### Nameless exchange
>
> In previous parts of the tutorial we knew nothing about exchanges, but still were able to send messages to queues. That was possible because we were using a default exchange, which we identify by the empty string \(""\).
>
> Recall how we published a message before:
>
> ```
> template.convertAndSend(fanout.getName(), "", message);
> ```
>
> The first parameter is the the name of the exchange that was autowired into the sender. The empty string denotes the default or \_nameless \_exchange: messages are routed to the queue with the name specified by routingKey, if it exists.

Now, we can publish to our named exchange instead:

```java
@Autowired
private RabbitTemplate template;

@Autowired
private FanoutExchange fanout;   // configured in Tut3Config above

template.convertAndSend(fanout.getName(), "", message);
```

From now on the fanout exchange will append messages to our queue.

## Temporary queues

As you may remember previously we were using queues which had a specified name \(rememberhello\). Being able to name a queue was crucial for us -- we needed to point the workers to the same queue. Giving a queue a name is important when you want to share the queue between producers and consumers.

But that's not the case for our fanout example. We want to hear about all messages, not just a subset of them. We're also interested only in currently flowing messages not in the old ones. To solve that we need two things.

Firstly, whenever we connect to Rabbit we need a fresh, empty queue. To do this we could create a queue with a random name, or, even better - let the server choose a random queue name for us.

Secondly, once we disconnect the consumer the queue should be automatically deleted. To do this with the spring-amqp client, we defined and_AnonymousQueue_, which creates a non-durable, exclusive, autodelete queue with a generated name:

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

## Bindings

![](https://www.rabbitmq.com/img/tutorials/bindings.png)

We've already created a fanout exchange and a queue. Now we need to tell the exchange to send messages to our queue. That relationship between exchange and a queue is called a _binding_. In the above Tut3Config you can see that we have two bindings, one for each AnonymousQueue.

```java
@Bean
public Binding binding1(FanoutExchange fanout, 
        Queue autoDeleteQueue1) {
    return BindingBuilder.bind(autoDeleteQueue1).to(fanout);
}
```

> #### Listing bindings
>
> You can list existing bindings using, you guessed it,
>
> ```
> rabbitmqctl list_bindings
> ```

## Putting it all together

![](https://www.rabbitmq.com/img/tutorials/python-three-overall.png)

The producer program, which emits messages, doesn't look much different from the previous tutorial. The most important change is that we now want to publish messages to our fanout exchange instead of the nameless one. We need to supply a routingKey when sending, but its value is ignored for fanout exchanges. Here goes the code for tut3.Sender.java program:

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

As you see, we leverage the beans from the Tut3Config file and autowire in the RabbitTemplate along with our configured FanoutExchange This step is necessary as publishing to a non-existing exchange is forbidden.

The messages will be lost if no queue is bound to the exchange yet, but that's okay for us; if no consumer is listening yet we can safely discard the message.

The code forTut3Receiver.java:

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

```
mvn clean package
```

And of course, to execute the tutorial do the following:

```
java -jar target/rabbit-tutorials-1.7.1.RELEASE.jar --spring.profiles.active=pub-sub,receiver 
    --tutorial.client.duration=60000
java -jar target/rabbit-tutorials-1.7.1.RELEASE.jar --spring.profiles.active=pub-sub,sender 
    --tutorial.client.duration=60000
```

Using rabbitmqctl list\_bindings you can verify that the code actually creates bindings and queues as we want. With two ReceiveLogs.java programs running you should see something like:

```
sudo rabbitmqctl list_bindings
tut.fanout  exchange    8b289c9c-a1eb-4a3a-b6a9-163c4fdcb6c2    queue       []
tut.fanout  exchange    d7e7d193-65b1-4128-a532-466a5256fd31    queue       []
```

The interpretation of the result is straightforward: data from exchangelogsgoes to two queues with server-assigned names. And that's exactly what we intended.

To find out how to listen for a subset of messages, let's move on to tutorial 4

