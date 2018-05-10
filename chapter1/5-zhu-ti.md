## Topics（主题）

In the previous tutorial we improved our messaging flexibility. Instead of using a fanout exchange only capable of dummy broadcasting, we used a direct one, and gained a possibility of selectively receiving the message based on the routing key.

在上一个教程里我们改善了我们的消息队列的灵活性。我们使用直接交换器来替代只会傻傻地广播消息的广播交换器，使得根据路由键来选择性接收消息成为了可能。

Although using the direct exchange improved our system, it still has limitations - it can't do routing based on multiple criteria.

虽然使用直接交换器改善了我们的系统，但它仍然有一些限制——它无法根据多个标准来进行路由。

In our messaging system we might want to subscribe to not only queues based on the routing key, but also based on the source which produced the message. You might know this concept from the syslog unix tool, which routes logs based on both severity \(info/warn/crit...\) and facility \(auth/cron/kern...\). Our example is a little simpler than this.

在我们的消息队列系统中，我们可能不仅仅想订阅基于路由键的队列，可能还想订阅基于消息源的队列。你可以通过unix工具syslog来理解这个概念，这个工具同时基于严重级别（info/warn/crit...）和组件类型（auth/cron/kern...）来路由日志。我们的例子有点类似于这个。

That example would give us a lot of flexibility - we may want to listen to just critical errors coming from 'cron' but also all logs from 'kern'.

这个例子将给予我们很大便利性—我们可能想级别为严重错误的日志，而这些日志则同时来自“cron”和“kern”。

To implement that flexibility in our logging system we need to learn about a more complex topic exchange.

为了在我们的日志系统中实现这个灵活性，我们需要了解更复杂的关于主题交换器的知识。

## Topic exchange（主题交换器）

Messages sent to a topic exchange can't have an arbitrary routing\_key - it must be a list of words, delimited by dots. The words can be anything, but usually they specify some features connected to the message. A few valid routing key examples: "stock.usd.nyse", "nyse.vmw", "quick.orange.rabbit". There can be as many words in the routing key as you like, up to the limit of 255 bytes.

发送到主题交换器的消息不能有任意的路由键——它必须是一个由点号隔开的单词组。这些单词可以是任意词语，但通常情况下它们能说明消息的特征。举几个有效的路由键的例子："stock.usd.nyse"，"nyse.vmw"，"quick.orange.rabbit"。路由键里的单词数你想要多少个都可以，但上限是255的字节。

The binding key must also be in the same form. The logic behind the topic exchange is similar to a direct one - a message sent with a particular routing key will be delivered to all the queues that are bound with a matching binding key. However there are two important special cases for binding keys:

绑定键也必须是相同的格式。主题交换器背后的逻辑类似于直接交换器——一条带着特定路由键的消息将会被发送所有绑定着匹配的绑定键的队列。不过，绑定键由两个重要的特殊情况：

\*\(star\) can substitute for exactly one word.

星号可以替代一个单词。

\#\(hash\) can substitute for zero or more words.

哈希号可以替代0个或多个单词。

It's easiest to explain this in an example:

用一个例子可以很容易地解释：

![](https://www.rabbitmq.com/img/tutorials/python-five.png)

In this example, we're going to send messages which all describe animals. The messages will be sent with a routing key that consists of three words \(two dots\). The first word in the routing key will describe speed, second a colour and third a species: "&lt;speed&gt;.&lt;colour&gt;.&lt;species&gt;".

在图例中，我们将发送所有描述动物的消息。每条消息将和包含着由三个单词组成（两个点号）的路由键一起被发送。路由键中的第一个单词将描述速度，第二个单词描述颜色，第三个单词描述种类，即格式为："&lt;speed&gt;.&lt;colour&gt;.&lt;species&gt;"。

We created three bindings: Q1 is bound with binding key "\*.orange.\*" and Q2 with "\*.\*.rabbit" and "lazy.\#".

我们建立了三个绑定：队列Q1使用绑定键“\*.orange.\*”，队列Q2使用“\*.\*.rabbit”和“lazy.\#”。

These bindings can be summarised as:

这些绑定可以总结描述为：

Q1 is interested in all the orange animals.

队列Q1对所有橙色的动物感兴趣。

Q2 wants to hear everything about rabbits, and everything about lazy animals.

队列Q2想监听所有的兔子，以及所有带有懒惰属性的动物。

A message with a routing key set to "quick.orange.rabbit" will be delivered to both queues. Message "lazy.orange.elephant" also will go to both of them. On the other hand "quick.orange.fox" will only go to the first queue, and "lazy.brown.fox" only to the second. "lazy.pink.rabbit" will be delivered to the second queue only once, even though it matches two bindings. "quick.brown.fox" doesn't match any binding so it will be discarded.

带有路由键为“quick.orange.rabbit”的消息将同时被发送给队列Q1和Q2。带有路由键为“lazy.orange.elephant”的消息也同样将被发送给这两条队列。另一方面，路由键为“quick.orange.fox”的消息将仅被发送给队列Q1，而路由键为“lazy.brown.fox”的消息将仅被发送给队列Q2。路由键为“lazy.pink.rabbit”的消息将只被发送给队列Q2一次，即使它匹配队列Q2的两个绑定。路由键“quick.brown.fox”的消息由于不匹配任何绑定，所以它将被丢弃。

What happens if we break our contract and send a message with one or four words, like "orange" or "quick.orange.male.rabbit"? Well, these messages won't match any bindings and will be lost.

如果我们打破了约定并发送了路由键为一个或四个单词的消息，如“orange”或者“quick.orange.male.rabbit”，会发生什么现象？没事，由于这些消息不能匹配任何绑定，所以它们也将被丢弃。

On the other hand "lazy.orange.male.rabbit", even though it has four words, will match the last binding and will be delivered to the second queue.

另一方面，虽然“lazy.orange.male.rabbit”包含了四个词，但它匹配了最后一条绑定规则，所以它将被发送给队列Q2。

> #### Topic exchange（主题交换器）
>
> Topic exchange is powerful and can behave like other exchanges.
>
> 主题交换器很强大，而且还能表现出与其它类型的交换器相同的行为。
>
> When a queue is bound with "\#" \(hash\) binding key - it will receive all the messages, regardless of the routing key - like in fanout exchange.
>
> 当一个队列与“\#”（哈希号）绑定键绑定时，它将接收所有的消息，而不管消息的路由键是什么，此时的队列看起来就像与广播交换器绑定了一样。
>
> When special characters "\*" \(star\) and "\#" \(hash\) aren't used in bindings, the topic exchange will behave just like a direct one.
>
> 当特殊符号“\*”（星号）和“\#”（哈希号）没有出现在绑定键时，主题交换器就表现得跟直接交换器一样。

## Putting it all together

We're going to use a topic exchange in our messaging system. We'll start off with a working assumption that the routing keys will take advantage of both wildcards and a hash tag.

我们将在我们的消息队列系统中使用主题交换器。开始之前，我们先假设路由键将会用到通配符和哈希标签。

The code is almost the same as in the previous tutorial.

代码几乎与上一个教程的代码一样。

First let's configure some profiles and beans in the Tut5Config.java of the tut5 package:

首先，我们在tut5包目录下新建Tut5Config.java，并在这个配置类里配置好一些配置组和bean：

```java
import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

@Profile({"tut5","topics"})
@Configuration
public class Tut5Config {

    @Bean
    public TopicExchange topic() {
        return new TopicExchange("tut.topic");
    }

    @Profile("receiver")
    private static class ReceiverConfig {

        @Bean
        public Tut5Receiver receiver() {
            return new Tut5Receiver();
        }

        @Bean
        public Queue autoDeleteQueue1() {
            return new AnonymousQueue();
        }

        @Bean
        public Queue autoDeleteQueue2() {
            return new AnonymousQueue();
        }

        @Bean
        public Binding binding1a(TopicExchange topic, 
            Queue autoDeleteQueue1) {
            return BindingBuilder.bind(autoDeleteQueue1)
                .to(topic)
                .with("*.orange.*");
        }

        @Bean
        public Binding binding1b(TopicExchange topic, 
            Queue autoDeleteQueue1) {
            return BindingBuilder.bind(autoDeleteQueue1)
                .to(topic)
                .with("*.*.rabbit");
        }

        @Bean
        public Binding binding2a(TopicExchange topic, 
            Queue autoDeleteQueue2) {
            return BindingBuilder.bind(autoDeleteQueue2)
                .to(topic)
                .with("lazy.#");
        }

    }

    @Profile("sender")
    @Bean
    public Tut5Sender sender() {
        return new Tut5Sender();
    }

}
```

We setup our profiles for executing the topics as the choice of "tut5" or "topics". We then created the bean for our TopicExchange. The "receiver" profile is the ReceiverConfig defining our receiver, two AnonymousQueues as in the previous tutorial and the bindings for the topics utilizing the topic syntax. We also create the "sender" profile as the creation of the Tut5Sender class.

我们将配置组的名字设置为“tut5”或“topics”，要运行主题时任选一个即可。然后我们创建了类型为TopicExchange的bean。接收者配置组为ReceiveConfig类，在其里面定义了我们的接收者，两个AnonymousQueue类型的队列（就像上一个教程那样），同时还通过主体语法为主体定义了一系列绑定。我们还创建了发送者配置组，用于创建Tut5Sender类的bean。

The Tut5Receiver again uses the @RabbitListener to receive messages from the respective topics.

Tut5Receiver类同样使用了@RabbitListener来接收相应主题的消息。

```java
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.util.StopWatch;

public class Tut5Receiver {

    @RabbitListener(queues = "#{autoDeleteQueue1.name}")
    public void receive1(String in) throws InterruptedException {
        receive(in, 1);
    }

    @RabbitListener(queues = "#{autoDeleteQueue2.name}")
    public void receive2(String in) throws InterruptedException {
        receive(in, 2);
    }

    public void receive(String in, int receiver) throws 
        InterruptedException {
        StopWatch watch = new StopWatch();
        watch.start();
        System.out.println("instance " + receiver + " [x] Received '" 
            + in + "'");
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

The code for Tut5Sender.java:

以下为Tut5Sender.java的代码：

```java
import org.springframework.amqp.core.TopicExchange;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;

public class Tut5Sender {

    @Autowired
    private RabbitTemplate template;

    @Autowired
    private TopicExchange topic;


    private int index;

    private int count;

    private final String[] keys = {"quick.orange.rabbit", 
            "lazy.orange.elephant", "quick.orange.fox",
            "lazy.brown.fox", "lazy.pink.rabbit", "quick.brown.fox"};

    @Scheduled(fixedDelay = 1000, initialDelay = 500)
    public void send() {
        StringBuilder builder = new StringBuilder("Hello to ");
        if (++this.index == keys.length) {
            this.index = 0;
        }
        String key = keys[this.index];
        builder.append(key).append(' ');
        builder.append(Integer.toString(++this.count));
        String message = builder.toString();
        template.convertAndSend(topic.getName(), key, message);
        System.out.println(" [x] Sent '" + message + "'");
    }

}
```

Compile and run the examples as described in Tutorial 1 . Or if you have been following along through the tutorials you only need to do the following:

像教程1描述的那样去编译并运行实例代码。或者如果你是一直跟着教程学习的，那么你只需接着跟着往下做：

To build the project:

先构建项目：

```
mvn clean package
```

To execute the sender and receiver with the correct profiles execute the jar with the correct parameters:

接着，分别使用正确的配置组来运行发送者和接收者，运行jar包时要使用正确的参数：

```
java -jar target/rabbit-tutorials-1.7.1.RELEASE.jar 
    --spring.profiles.active=topics,receiver 
    --tutorial.client.duration=60000
java -jar target/rabbit-tutorials-1.7.1.RELEASE.jar 
    --spring.profiles.active=topics,sender 
    --tutorial.client.duration=60000
```

The output from the sender will look something like:

发送者进程的输出看起来是类似于下面这样的：

```
Ready ... running for 60000ms
 [x] Sent 'Hello to lazy.orange.elephant 1'
 [x] Sent 'Hello to quick.orange.fox 2'
 [x] Sent 'Hello to lazy.brown.fox 3'
 [x] Sent 'Hello to lazy.pink.rabbit 4'
 [x] Sent 'Hello to quick.brown.fox 5'
 [x] Sent 'Hello to quick.orange.rabbit 6'
 [x] Sent 'Hello to lazy.orange.elephant 7'
 [x] Sent 'Hello to quick.orange.fox 8'
 [x] Sent 'Hello to lazy.brown.fox 9'
 [x] Sent 'Hello to lazy.pink.rabbit 10'
```

And the receiver will respond with the following output:

然后接收者进程的响应输出如下面这样：

```
instance 1 [x] Received 'Hello to lazy.orange.elephant 1'
instance 2 [x] Received 'Hello to lazy.orange.elephant 1'
instance 2 [x] Done in 2.005s
instance 1 [x] Done in 2.005s
instance 1 [x] Received 'Hello to quick.orange.fox 2'
instance 2 [x] Received 'Hello to lazy.brown.fox 3'
instance 1 [x] Done in 2.003s
instance 2 [x] Done in 2.003s
instance 1 [x] Received 'Hello to lazy.pink.rabbit 4'
instance 2 [x] Received 'Hello to lazy.pink.rabbit 4'
instance 1 [x] Done in 2.006s
instance 2 [x] Done in 2.006s
```

Have fun playing with these programs. Note that the code doesn't make any assumption about the routing or binding keys, you may want to play with more than two routing key parameters.

\(Full source code for [Tut5Receiver.java source](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/spring-amqp/src/main/java/org/springframework/amqp/tutorials/tut5/Tut5Receiver.java) and [Tut5Sender.java source](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/spring-amqp/src/main/java/org/springframework/amqp/tutorials/tut5/Tut5Sender.java). The configuration is in [Tut5Config.java source](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/spring-amqp/src/main/java/org/springframework/amqp/tutorials/tut5/Tut5Config.java). \)

（完整的代码请参阅[Tut5Receiver.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/spring-amqp/src/main/java/org/springframework/amqp/tutorials/tut5/Tut5Receiver.java)和[Tut5Sender.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/spring-amqp/src/main/java/org/springframework/amqp/tutorials/tut5/Tut5Sender.java) 。配置在[Tut5Config.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/spring-amqp/src/main/java/org/springframework/amqp/tutorials/tut5/Tut5Config.java)里）

Next, find out how to do a round trip message as a remote procedure call in tutorial 6.

接下来，我们将进入教程6，看看如何进行消息交互，即远程过程调用。

