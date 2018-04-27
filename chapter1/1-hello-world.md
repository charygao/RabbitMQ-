# 1 "Hello World!"

## Introduction

RabbitMQ is a message broker: it accepts and forwards messages. You can think about it as a post office: when you put the mail that you want posting in a post box, you can be sure that Mr. Postman will eventually deliver the mail to your recipient. In this analogy, RabbitMQ is a post box, a post office and a postman.

## 简介

RabbitMQ是一个消息代理：它接受并转发消息。你可以将它看成是一个邮局：当你将你想发送的邮件丢进邮箱时，你就可以确定邮递员先生最终会把这封邮件送到你的收件人手上。在这个类比当中，RabbitMQ既是邮箱，又是邮局，而且还是邮递员。

The major difference between RabbitMQ and the post office is that it doesn't deal with paper, instead it accepts, stores and forwards binary blobs of data ‒ _messages_.

RabbitMQ和邮局之间的主要不同点是，RabbitMQ不跟纸打交道，它只接收，存储并转发二进制的数据包—消息。

RabbitMQ, and messaging in general, uses some jargon.

RabbitMQ，一般称它为消息队列，使用了一些术语。

_Producing means nothing more than sending. A program that sends messages is a producer_:

消息生产其实就是消息发送。发送消息的程序就是一个生产者：

![](http://www.rabbitmq.com/img/tutorials/producer.png)

A _queue_ is the name for a post box which lives inside RabbitMQ. Although messages flow through RabbitMQ and your applications, they can only be stored inside a _queue_. A queue is only bound by the host's memory & disk limits, it's essentially a large message buffer. Many producers can send messages that go to one queue, and many consumers can try to receive data from one _queue_. This is how we represent a queue:

一个队列就是一个存在RabbitMQ内部的邮箱的名字。虽然消息在传递时会通过RabbitMQ以及你的应用，但它们只能被存储在一个队列里。队列大小只受限于主机内存和硬盘容量，它本质上就是个大的消息缓存。多个生产者可以往一个队列里发送消息，多个消费者可以从一个队列里获取数据。以下是我们表示一个队列的方式：

![](http://www.rabbitmq.com/img/tutorials/queue.png)

_Consuming_ has a similar meaning to receiving. A _consumer_ is a program that mostly waits to receive messages:

消息消费在意思上类似于消息接收。等待接收消息的程序就是一个消费者：

![](http://www.rabbitmq.com/img/tutorials/consumer.png)

Note that the producer, consumer, and broker do not have to reside on the same host; indeed in most applications they don't.

注意，生产者，消费者和代理不一定都在同一个主机里；实际上，在大多数应用中，这三者都不是在同一个主机里。

## "Hello World"

In this part of the tutorial we'll write two programs using the spring-amqp library; a producer that sends a single message, and a consumer that receives messages and prints them out. We'll gloss over some of the detail in the Spring-amqp API, concentrating on this very simple thing just to get started. It's a "Hello World" of messaging.

下面我们将写两个使用spring-amqp类库的程序；其中一个是发送单条消息的生产者，另一个是消费者，它接收消息并将它们打印出来。我们将掩盖Spring-amqp API的一些细节，专注于即将开始的东西。它就是“Hello World”消息队列。

In the diagram below, "P" is our producer and "C" is our consumer. The box in the middle is a queue - a message buffer that RabbitMQ keeps on behalf of the consumer.

在下面的图中，“P”是我们的生产者，“C”是我们的消费者。图中间的箱子是一个队列，也就是RabbitMQ保存的消息缓存，为了给消费者用：

![](http://www.rabbitmq.com/img/tutorials/python-one.png)

> #### The Spring AMQP Framework
>
> RabbitMQ speaks multiple protocols. This tutorial uses AMQP 0-9-1, which is an open, general-purpose protocol for messaging. There are a number of clients for RabbitMQ in many different languages.
>
> #### Spring AMQP框架
>
> RabbitMQ支持多种协议。本教程使用AMQP 0-9-1协议，它是一个开放，通用的消息队列协议。很多语言都提供了RabbitMQ客户端。

Spring AMQP leverages Spring Boot for configuration and dependency management. Spring supports maven or gradle but for this tutorial we'll select maven with Spring Boot 1.5.2. Open the [Spring Initializr](http://start.spring.io/) and provide: the group id \(e.g. org.springframework.amqp.tutorials\) the artifact id \(e.g. rabbitmq-amqp-tutorials\) Search for the amqp dependency and select the AMQP dependency.

Spring AMQP利用Spring Boot来进行配置和依赖管理。Spring同时支持maven或者gradle，但在本教程里我们选择用Spring Boot 1.5.2的maven。我们打开[Spring Initializr](http://start.spring.io/)并提供group id（如org.springframework.amqp.tutorials）和artifact id（如rabbitmq-amqp-tutorials）。查找amqp依赖并选择AMQP依赖。（译者注：应该是搜索rabbitmq）

Generate the project and unzip the generated project into the location of your choice. This can now be imported into your favorite IDE. Alternatively you can work on it from your favorite editor.

点击Generate Project生成项目，并将其解压到你想存放的目录。现在你可以在你喜欢的IDE里面导入这个项目。你也可以在你喜欢的编辑器上进行下一步编辑。

### Configuring the project

Spring Boot offers numerous features but we will only highlight a few here. First, Spring Boot applications have the option of providing their properties through either an application.properties or application.yml file \(there are many more options as well but this will get us going\). You'll find an application.properties file in the generated project with nothing in it. Rename application.properties to application.yml file with the following properties:

### 配置项目

Spring Boot提供了很多特性，但在这里我们只显示几个需要用到的。首先，Spring Boot应用的配置可以写在application.properties文件或者application.yml文件里（还有许多其它的方式，但对于我们，用这两种文件就足够了）。你将在生成的项目里找到一个空的application.properties文件。将这个文件重命名为application.yml，并写上这些属性：

```
spring:
  profiles:
    active: usage_message

logging:
  level:
    org: ERROR

tutorial:
  client:
    duration: 10000
```

Create a new directory \(package - tut1\) where we can put the tutorial code. We'll now create a JavaConfig file \(Tut1Config.java\) to describe our beans in the following manner:

创建一个新的目录（package - tut1）用来放我们的教程代码。现在我们将通过以下方式创建一个Java配置文件（Tut1Config.java）来描述我们的Bean：

```java
package org.springframework.amqp.tutorials.tut1;

import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

@Profile({"tut1","hello-world"})
@Configuration
public class Tut1Config {

    @Bean
    public Queue hello() {
        return new Queue("hello");
    }

    @Profile("receiver")
    @Bean
    public Tut1Receiver receiver() {
        return new Tut1Receiver();
    }

    @Profile("sender")
    @Bean
    public Tut1Sender sender() {
        return new Tut1Sender();
    }
}
```

Note that we've defined the 1st tutorial profile as either tut1, the package name, or hello-world. We use the @Configuration to let Spring know that this is a Java Configuration and in it we create the definition for our Queue \("hello"\) and define our Sender and Receiver beans.

注意，我们已经将教程的第一个配置文件定义为tu1，或者hello-world。我们用@Configuration注解来让Spring知道这是个Java配置，并且在配置里我们创建了Queue\("hello"\)的定义，而且也定义了我们的发送者和接收者。

We will run all of our tutorials through the Boot Application now by simply passing in which profiles we are using. To enable this we will modify the generated RabbitAmqpTutorialsApplication.java with the following:

现在我们将传入将要用的配置文件通过Boot Application来运行我们的教程。为了做到这一点，我们将修改生成的RabbitAmqpTutorialsApplication.java文件，如下所示：

```java
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Profile;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class RabbitAmqpTutorialsApplication {

    @Profile("usage_message")
    @Bean
    public CommandLineRunner usage() {
        return new CommandLineRunner() {

            @Override
            public void run(String... arg0) throws Exception {
                System.out.println("This app uses Spring Profiles to 
                    control its behavior.\n");
                System.out.println("Sample usage: java -jar 
                    rabbit-tutorials.jar 
                    --spring.profiles.active=hello-world,sender");
            }
        };
    }

    @Profile("!usage_message")
    @Bean
    public CommandLineRunner tutorial() {
        return new RabbitAmqpTutorialsRunner();
    }

    public static void main(String[] args) throws Exception {
        SpringApplication.run(RabbitAmqpTutorialsApplication.class, args);
    }
}
```

and add the RabbitAmqpTutorialsRunner.java code as follows:

然后像下面那样添加RabbitAmqpTutorialsRunner.java代码：

```java
package org.springframework.amqp.tutorials;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.ConfigurableApplicationContext;

public class RabbitAmqpTutorialsRunner implements CommandLineRunner {

    @Value("${tutorial.client.duration:0}")
    private int duration;

    @Autowired
    private ConfigurableApplicationContext ctx;

    @Override
    public void run(String... arg0) throws Exception {
        System.out.println("Ready ... running for " + duration + "ms");
        Thread.sleep(duration);
        ctx.close();
    }
}
```

### Sending（发送）

![](http://www.rabbitmq.com/img/tutorials/sending.png)

Now there is very little code that needs to go into the sender and receiver classes. Let's call them Tut1Receiver and Tut1Sender. The Sender leverages our config and the RabbitTemplate to send the message.

现在发送者类和接收者类需要写的代码很少。我们把接收者的类名叫做Tut1Receiver，把发送者的类名叫做Tut1Sender。发送者类利用我们的配置和RabbitTemplate类来发送消息。

```java
// Sender
package org.springframework.amqp.tutorials.tut1;

import org.springframework.amqp.core.Queue;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;

public class Tut1Sender {

    @Autowired
    private RabbitTemplate template;

    @Autowired
    private Queue queue;

    @Scheduled(fixedDelay = 1000, initialDelay = 500)
    public void send() {
        String message = "Hello World!";
        this.template.convertAndSend(queue.getName(), message);
        System.out.println(" [x] Sent '" + message + "'");
    }
}
```

You'll notice that spring-amqp removes the boiler plate code leaving you with only the logic of the messaging to be concerned about. We autowire in the queue that was configured in our bean definition in the Tut1Config class and like many spring connection abstractions, we wrap the boilerplate rabbitmq client classes with a RabbitTemplate that can be autowired into the sender. All that is left is to create a message and invoke the template's convertAndSend method passing in the queue name from the bean we defined and the message we just created.

> #### Sending doesn't work!
>
> If this is your first time using RabbitMQ and you don't see the "Sent" message then you may be left scratching your head wondering what could be wrong. Maybe the broker was started without enough free disk space \(by default it needs at least 200 MB free\) and is therefore refusing to accept messages. Check the broker logfile to confirm and reduce the limit if necessary. The [configuration file documentation](http://www.rabbitmq.com/tutorials/a%3E%20href=%22http://www.rabbitmq.com/configure.html#config-items%22%3C/a) will show you how to set disk\_free\_limit.

### Receiving（接收）

The receiver is equally simple. We annotate our Receiver class with @RabbitListener and pass in the name of the queue. We then annotate our receive method with @RabbitHandler passing in the payload that has been pushed to the queue.

```
package org.springframework.amqp.tutorials.tut1;

import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;

@RabbitListener(queues = "hello")
public class Tut1Receiver {

    @RabbitHandler
    public void receive(String in) {
        System.out.println(" [x] Received '" + in + "'");
    }
}
```

### Putting it all together

The app uses Spring Profiles to control what tutorial it's running, and whether it's a Sender or Receiver. Choose which tutorial to run by using the profile. For example:

```
- {tut1|hello-world},{sender|receiver}
- {tut2|work-queues},{sender|receiver}
- {tut3|pub-sub|publish-subscribe},{sender|receiver}
- {tut4|routing},{sender|receiver}
- {tut5|topics},{sender|receiver}
- {tut6|rpc},{client|server}
```

We'll come back to this list as we progress through the other five tutorials. After building with maven, run the app however you like to run boot apps \(e.g. from the ide, or command line\). We'll show how to run from the command line.

For example:

```
# publisher
java -jar rabbitmq-tutorials.jar --spring.profiles.active=hello-world,sender
```

```
# consumer
java -jar rabbitmq-tutorials.jar --spring.profiles.active=hello-world,receiver
```

> #### Listing queues
>
> You may wish to see what queues RabbitMQ has and how many messages are in them. You can do it \(as a privileged user\) using the rabbitmqctl tool:
>
> ```
> sudo rabbitmqctl list_queues
> ```
>
> On Windows, omit the sudo:
>
> ```
> rabbitmqctl.bat list_queues
> ```

Time to move on to part 2 and build a simple _work queue_

.

