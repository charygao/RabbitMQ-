# 2 工作队列（Work Queues）

![](http://www.rabbitmq.com/img/tutorials/python-two.png)

In the first tutorial we wrote programs to send and receive messages from a named queue. In this one we'll create a _Work_ _Queue_ that will be used to distribute time-consuming tasks among multiple workers.

在第一个教程中，我们编写了程序用于往一个命名了的队列发送消息，同时也从这个队列里接收消息。在本教程里，我们将创建一个工作队列，它将被用来在多个工作者之间分发耗时任务。

The main idea behind Work Queues \(aka: _Task Queues_\) is to avoid doing a resource-intensive task immediately and having to wait for it to complete. Instead we schedule the task to be done later. We encapsulate a _task_ as a message and send it to a queue. A worker process running in the background will pop the tasks and eventually execute the job. When you run many workers the tasks will be shared between them.

工作队列（也叫做任务队列）背后的主要目的是为了避免立即执行资源密集型的任务以及避免必须等待任务完成，而是计划着让这些任务可以留到后面去执行。我们将任务分装成消息，并将它发送到队列里。运行在后台的工作进程将取出任务并最终执行它。如果你运行了多个工作进程，那么任务将被它们共享。

This concept is especially useful in web applications where it's impossible to handle a complex task during a short HTTP request window.

对于web应用，这个概念特别有用，因为它使得web应用在短短的HTTP请求窗口内处理复杂的任务成为了可能。

### Preparation（准备工作）

In the previous part of this tutorial we sent a message containing "Hello World!". Now we'll be sending strings that stand for complex tasks. We don't have a real-world task, like images to be resized or pdf files to be rendered, so let's fake it by just pretending we're busy - by using the Thread.sleep\(\) function. We'll take the number of dots in the string as its complexity; every dot will account for one second of "work". For example, a fake task described by Hello... will take three seconds.

在本教程的前面部分，我们实现了发送包含“Hello World!”的消息。现在，我们将发送表示复杂任务的字符串。由于我们没有现实世界的任务，如调整图片大小或者渲染pdf文件，所以我们通过使用Thread.sleep\(\)函数来让任务显得很忙，以此达到耗时任务的效果。我们将在字符串中用点号的个数来象征复杂度，每个点号将耗费整个任务执行的一秒。例如，Hello...代表任务将执行三秒。

Please see the setup in first tutorial if you have not setup the project. We will follow the same pattern as in the first tutorial: 1\) create a package \(tut2\) and create a Tut2Config, Tut2Receiver, and Tut2Sender. Start by creating a new package \(tut2\) where we'll place our three classes. In the configuration class we setup two profiles, the label for the tutorial \("tut2"\) and the name of the pattern \("work-queues"\). We leverage spring to expose the queue as a bean. We setup the receiver as a profile and define two beans to correspond to the workers in our diagram above： receiver1 and receiver2. Finally, we define a profile for the sender and define the sender bean. The configuration is now done.

如果你还未配置好项目，请见第一个教程的配置过程。我们将采用与第一个教程相同的配置，新建一个包目录（tut2）并创建一个Tut2Config的配置类，一个Tut2Receiver的信息接收类，以及一个Tut2Sender的信息发送类。首先新建好新的包目录（tut2），我们将在这个包下面放刚说到的那三个类。在配置类里，我们将配置两个配置文件，一个作为当前教程的标签（“tut2”），一个作为当前模式的名字（“work-queue”）。我们利用Spring框架将队列暴露为一个bean。我们设置一个接受者配置文件，并定义两个bean来对应于上面图中的两个消费者：receiver1和receiver2。最后，我们会定义一个发送者配置文件，并定义作为发送者的bean。这样配置就结束了。

```java
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

@Profile({"tut2", "work-queues"})
@Configuration
public class Tut2Config {

    @Bean
    public Queue hello() {
        return new Queue("hello");
    }

    @Profile("receiver")
    private static class ReceiverConfig {

        @Bean
        public Tut2Receiver receiver1() {
            return new Tut2Receiver(1);
        }

        @Bean
        public Tut2Receiver receiver2() {
            return new Tut2Receiver(2);
        }
    }

    @Profile("sender")
    @Bean
    public Tut2Sender sender() {
        return new Tut2Sender();
    }
}
```

### Sender（发送者）

We will modify the sender to provide a means for identifying whether its a longer running task by appending a dot to the message in a very contrived fashion using the same method on the RabbitTemplate to publish the message, convertAndSend. The documentation defines this as, "Convert a Java object to an Amqp Message and send it to a default exchange with a default routing key."

我们将对发送者类进行修改，在发送方法中，通过人为地在消息后面添加点号来识别当前任务是否为耗时的，并依旧使用RabbitTemplate的convertAndSend方法来发布消息。文档把convertAndSend方法定义为，“将一个Java对象转换成一个Amqp消息，并用一个默认路由键（routing key）将其发送到一个默认的exchange里。”

```java
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;

public class Tut2Sender {

    @Autowired
    private RabbitTemplate template;

    @Autowired
    private Queue queue;

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
        template.convertAndSend(queue.getName(), message);
        System.out.println(" [x] Sent '" + message + "'");
    }
}
```

### Receiver（接收者）

Our receiver, Tut2Receiver, simulates an arbitary length for a fake task in the doWork\(\) method where the number of dots translates into the number of seconds the work will take. Again, we leverage a @RabbitListener on the "hello" queue and a @RabbitHandler to receive the message. The instance that is consuming the message is added to our monitor to show which instance, the message and the length of time to process the message.

我们的接收者类，Tut2Receiver，在doWork\(\)方法里根据消息所带点号的个数，将其转换成任务所需消耗的秒数，从而模拟了任务的任意长度。同样地，我们对“hello”队列使用了@RabbitListener注解，并使用@RabbitHandler来接收消息。同时，用一个编号来标识正在消费消息的实例，并将它假如我们的监控中，最终打印出实例编号，消息内容以及处理消息所耗费的时长。

```java
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.util.StopWatch;

@RabbitListener(queues = "hello")
public class Tut2Receiver {

    private final int instance;

    public Tut2Receiver(int i) {
        this.instance = i;
    }

    @RabbitHandler
    public void receive(String in) throws InterruptedException {
        StopWatch watch = new StopWatch();
        watch.start();
        System.out.println("instance " + this.instance +
            " [x] Received '" + in + "'");
        doWork(in);
        watch.stop();
        System.out.println("instance " + this.instance +
            " [x] Done in " + watch.getTotalTimeSeconds() + "s");
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

### Putting it all together

Compile them using mvn package and run with the following options

使用mvn package来编译上述代码，并在运行时添加下面的命令行参数

```
mvn clean package

java -jar target/rabbitmq-amqp-tutorials-0.0.1-SNAPSHOT.jar --spring.profiles.active=work-queues,receiver
java -jar target/rabbitmq-amqp-tutorials-0.0.1-SNAPSHOT.jar --spring.profiles.active=work-queues,sender
```

The output of the sender should look something like:

发送者类的输出看起来应该是类似于这样的：

```
Ready ... running for 10000ms
 [x] Sent 'Hello.1'
 [x] Sent 'Hello..2'
 [x] Sent 'Hello...3'
 [x] Sent 'Hello.4'
 [x] Sent 'Hello..5'
 [x] Sent 'Hello...6'
 [x] Sent 'Hello.7'
 [x] Sent 'Hello..8'
 [x] Sent 'Hello...9'
 [x] Sent 'Hello.10'
```

And the output from the workers should look something like:

而工作者的输出看起来应该是类似于这样的：

```
Ready ... running for 10000ms
instance 1 [x] Received 'Hello.1'
instance 2 [x] Received 'Hello..2'
instance 1 [x] Done in 1.001s
instance 1 [x] Received 'Hello...3'
instance 2 [x] Done in 2.004s
instance 2 [x] Received 'Hello.4'
instance 2 [x] Done in 1.0s
instance 2 [x] Received 'Hello..5'
```

### Message acknowledgment（消息确认）

Doing a task can take a few seconds. You may wonder what happens if one of the consumers starts a long task and dies with it only partly done. Spring AMQP by default takes a conservative approach to message acknowledgement. If the listener throws an exception the container calls:

完成一个任务需要耗费几秒。对于一个开始了耗时长的任务的消费者，你可能会想知道当它只完成了部分任务就挂掉时发生了什么。Spring AMQP默认采用保守的方式来进行消息确认。如果监听器抛出了一个异常，那么容器会调用：

```java
channel.basicReject(deliveryTag, requeue)
```

Requeue is true by default unless you explicitly set:

requeue（重入队列）默认为true，除非你显示地将其设置为false：

```
defaultRequeueRejected=false
```

or the listener throws an AmqpRejectAndDontRequeueException. This is typically the bahavior you want from your listener. In this mode there is no need to worry about a forgotten acknowledgement. After processing the message the listener calls:

否则监听器抛出AmqpRejectAndDontRequeueException异常。这通常会是你想要监听器的行为。在这种模式下，无需担心一个被忘记了的确认。在处理完消息后，监听器会调用：

```java
channel.basicAck()
```

Acknowledgement must be sent on the same channel the delivery it is for was received on. Attempts to acknowledge using a different channel will result in a channel-level protocol exception. See the doc guide on confirmations to learn more. Spring AMQP generally takes care of this but when used in combination with code that uses RabbitMQ Java client directly, this is something to keep in mind.

消息确认必须在与接收信息相同的channel上进行发送。试图使用不同的channel来进行确认将导致channel级别的协议异常。若想更详细了解，可以参阅文档指南。Spring AMQP一般都会处理好这种问题，除非与直接使用了RabbitMQ的Java客户端的代码结合使用，这点要小心。

> #### Forgotten acknowledgment（被遗忘的确认）
>
> It's a common mistake to miss the basicAck and spring-amqp helps to avoid this through its default configuraiton. The consequences are serious. Messages will be redelivered when your client quits \(which may look like random redelivery\), but RabbitMQ will eat more and more memory as it won't be able to release any unacked messages.
>
> 忘记调用basciAck方法是常见的一个错误，spring-amqp通过它的默认配置来避免它。这个错误的后果是很严重的。当你的客户端退出后，消息会被重复发送（看起来就像是随机重发），但RabbitMQ将吃掉越来越多的内存，因为它无法释放任何未确认的消息。
>
> In order to debug this kind of mistake you can use rabbitmqctl to print the messages\_unacknowledged field:
>
> 为了调试这种错误，你可以使用rabbitmqctl来打印messages\_unacknowledged域：
>
> ```
> sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
> ```
>
> On Windows, drop the sudo:
>
> 在Windows环境下，去掉sudo：
>
> ```
> rabbitmqctl.bat list_queues name messages_ready messages_unacknowledged
> ```

### Message durability（消息持久性）

With spring-amqp there are reasonable default values in the MessageProperties that account for message durability. In particular you can check the table for common properties. You'll see two relevant to our discussion here on durability:

对于spring-amqp，消息属性配置里有很多合理的默认值，这些默认值共同决定了消息的持久性。你可以查阅常用属性表。你将会看到与我们正在讨论的消息持久性相关的两个属性：

| Property | default | Description |
| :--- | :--- | :--- |
| durable | true | When declareExchange is true the durable flag is set to this value |
| deliveryMode | PERSISTENT | PERSISTENT or NON\_PERSISTENT to determine whether or not RabbitMQ should persist the messages |

> #### Note on message persistence（关于消息持久化要注意的地方）
>
> Marking messages as persistent doesn't fully guarantee that a message won't be lost. Although it tells RabbitMQ to save the message to disk, there is still a short time window when RabbitMQ has accepted a message and hasn't saved it yet. Also, RabbitMQ doesn't do fsync\(2\) for every message -- it may be just saved to cache and not really written to the disk. The persistence guarantees aren't strong, but it's more than enough for our simple task queue. If you need a stronger guarantee then you can use publisher confirms.
>
> 将消息标记为持久化并不能完全保证消息将不会丢失。虽然它告诉RabbitMQ要将消息保存到磁盘，但从RabbitMQ接收消息到将这条消息发送出去仍有一个小的时间窗口。而且，RabbitMQ不会为每条消息都进行fsync操作——它可能仅仅只是将其缓存起来，但并没有真的将消息写入磁盘。这么做虽然无法完全保证持久化，但对于我们简单的任务队列来说，这已经很足够了。如果你需要完全保证持久化，那么你可以使用发布者确认。

### Fair dispatch vs Round-robin dispatching（公平调度vs循环调度）

By default, RabbitMQ will send each message to the next consumer, in sequence. On average every consumer will get the same number of messages. This way of distributing messages is called round-robin. In this mode dispatching doesn't necessarily work exactly as we want. For example in a situation with two workers, when all odd messages are heavy and even messages are light, one worker will be constantly busy and the other one will do hardly any work. Well, RabbitMQ doesn't know anything about that and will still dispatch messages evenly.

默认情况下，RabbitMQ将把每一条消息按顺序逐一发送给下一个消费者。每个消费者都将被平均分到相同的个数的消息。这种消息分派方式被称为循环调度。这种调度模式有时无法完全满足我们的需求。例如，假设有两个工作者，第奇数条消息是重任务，第偶数条消息是轻任务，那么其中一个工作者将会总是很繁忙，而另一个工作者则有可能没什么事做。然而，RabbitMQ并不知道这个状况，并且仍然会这么均匀地分发消息。

This happens because RabbitMQ just dispatches a message when the message enters the queue. It doesn't look at the number of unacknowledged messages for a consumer. It just blindly dispatches every n-th message to the n-th consumer.

会发生这个情况是因为，当消息进入队列时，RabbitMQ仅仅是将消息分派出去。它并考虑某个消费者未确认的消息的数量。它只是盲目地将消息均匀分派给各个消费者。

However, "Fair dispatch" is the default configuration for spring-amqp. The SimpleMessageListenerContainer defines the value for DEFAULT\_PREFETCH\_COUNT to be 1. If the DEFAULT\_PREFECTH\_COUNT were set to 0 the behavior would be round robin messaging as described above.

然而，“公平调度”是spring-amqp的默认配置。SimpleMessageListenerContainer类将DEFAULT\_PREFETCH\_COUNT的值定义为1。如果DEFAULT\_PREFETCH\_COUNT的值被设置为0，那么调度方式将变成循环调用。

![](http://www.rabbitmq.com/img/tutorials/prefetch-count.png)

However, with the prefetchCount set to 1 by default, this tells RabbitMQ not to give more than one message to a worker at a time. Or, in other words, don't dispatch a new message to a worker until it has processed and acknowledged the previous one. Instead, it will dispatch it to the next worker that is not still busy.

> #### Note about queue size
>
> If all the workers are busy, your queue can fill up. You will want to keep an eye on that, and maybe add more workers, or have some other strategy.

By using spring-amqp you get reasonable values configured for message acknowledgments and fair dispatching. The default durability for queues and persistence for messages provided by spring-amqp allow let the messages to survive even if RabbitMQ is restarted.

For more information on Channel methods and MessageProperties, you can browse the javadocs online For understanding the underlying foundation for spring-amqp you can find the rabbitmq-java-client.

Now we can move on to tutorial 3 and learn how to deliver the same message to many consumers.

