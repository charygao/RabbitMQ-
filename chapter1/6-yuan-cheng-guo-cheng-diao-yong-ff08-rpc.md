## Remote procedure call \(RPC，远程过程调用\)

In the second tutorial we learned how to use _Work Queues_ to distribute time-consuming tasks among multiple workers.

在教程2里我们学习了如何使用工作队列在多个工作者之间分发耗时任务。

But what if we need to run a function on a remote computer and wait for the result? Well, that's a different story. This pattern is commonly known as _Remote Procedure Call or RPC_.

但如果我们需要在一个远程电脑上运行一个函数并且等待运行结果的话要怎么办呢？这就变成另一个问题了。这种模式通常被称为远程过程调用（Remote Procedure Call），或者简称RPC。

In this tutorial we're going to use RabbitMQ to build an RPC system: a client and a scalable RPC server. As we don't have any time-consuming tasks that are worth distributing, we're going to create a dummy RPC service that returns Fibonacci numbers.

在本节教程里，我们将用RabbitMQ来构建一个RPC系统，这个系统包括一个客户端和一个可伸缩的RPC服务端。由于我们没有什么耗时任务值得分发，所以我们准备创建一个假的RPC服务，这个服务返回斐波那契（Fibonacci）数值。

### Client interface（客户端接口）

To illustrate how an RPC service could be used we're going to change the names of our profiles from "Sender" and "Receiver” to "Client" and "Server". When we call the server we will get back the fibonacci of the argument we call with.

为了说明RPC服务可以如何被使用，我们准备修改我们的配置组，将名称从“Sender”和“Receiver”换成“Client”和“Server”。当我们调用服务端时，我们将会获得我们传入的参数所对应的斐波那契数值。

```java
Integer response = (Integer) template.convertSendAndReceive
    (exchange.getName(), "rpc", start++);
System.out.println(" [.] Got '" + response + "'");
```

> #### A note on RPC（RPC的注意点）
>
> Although RPC is a pretty common pattern in computing, it's often criticised. The problems arise when a programmer is not aware whether a function call is local or if it's a slow RPC. Confusions like that result in an unpredictable system and adds unnecessary complexity to debugging. Instead of simplifying software, misused RPC can result in unmaintainable spaghetti code.
>
> 虽然RPC在计算领域是很常见的模式，但它通常也是受争议的。但程序员不知道一个函数调用是本地的还是慢速的RPC时就会出现一些问题。像这样的混乱会导致不可预知的系统，而且会给调试增加不必要的复杂性。不恰当地使用RPC不仅不会简化程序，还会导致代码变得很难维护。
>
> Bearing that in mind, consider the following advice:
>
> 记住这一点，然后考虑一下几点建议：
>
> * Make sure it's obvious which function call is local and which is remote.（确保哪个函数调用是本地的，哪个是远程的。）
> * Document your system. Make the dependencies between components clear.（为你的系统做好文档。清晰化组件间的依赖。）
> * Handle error cases. How should the client react when the RPC server is down for a long time?（处理好会发生错误的场景。但RPC服务端长时间挂掉时，客户端应该做出什么反应？）
>
> When in doubt avoid RPC. If you can, you should use an asynchronous pipeline - instead of RPC-like blocking, results are asynchronously pushed to a next computation stage.
>
> 当你无法对这些问题无法做出明确回答时，就不要使用RPC。如果可以的话，你应该使用异步pipeline，而不是类似于阻塞的RPC。使用异步pipeline，计算结果可以异步推入到下一个计算阶段。

### Callback queue（回调队列）

In general doing RPC over RabbitMQ is easy. A client sends a request message and a server replies with a response message. In order to receive a response we need to send a 'callback' queue address with the request. Spring-amqp's RabbitTemplate handles the callback queue for us when we use the above 'convertSendAndReceive\(\)' method. There is no need to do any other setup when using the RabbitTemplate. For a thorough explanation please see [Request/Reply Message](http://docs.spring.io/spring-amqp/reference/htmlsingle/#request-reply).

一般情况下，在RabbitMQ上实现RPC挺简单的。客户端发送请求消息然后服务端返回一个响应消息。为了接收响应消息，我们必须传送一个用于处理请求的回调队列。在我们使用“convertSendAndReceive\(\)”方法时，Spring-amqp框架的RabbitTemplate类为我们做好了回调队列的处理工作。使用RabbitTemplate类时无需在做其它配置。若想看完整的文档，请参阅[请求/发送消息](https://legacy.gitbook.com/book/jiapengcai/rabbitmq/edit#)。

> #### Message properties（消息属性）
>
> The AMQP 0-9-1 protocol predefines a set of 14 properties that go with a message. Most of the properties are rarely used, with the exception of the following:
>
> AMQP 0-9-1协议预定义了14个消息属性。大部分的属性都很少用到，除了以下几个：
>
> * deliveryMode: Marks a message as persistent \(with a value of 2\) or transient \(any other value\). You may remember this property from the second tutorial.
> * deliveryMode：将消息标记为要持久化（此时属性值为2）或者瞬态（此时属性值为2以外的其它数字）。教程2里提到过这   
>   个属性，你应该还记得。
> * contentType: Used to describe the mime-type of the encoding. For example for the often used JSON encoding it is a good practice to set this property to: application/json.
> * contentType：用来描述编码的mime类型。例如，对于常用的JSON格式，最好将这个属性值设为application/json。
> * replyTo: Commonly used to name a callback queue.
> * replayTo：通常用来命名一个回调队列。
> * correlationId: Useful to correlate RPC responses with requests.
> * correlationId：该属性用来将RPC响应与请求进行关联。

### Correlation Id（关联Id）

Spring-amqp allows you to focus on the message style you're working with and hide the details of message plumbing required to support this style. For example, typically the native client would create a callback queue for every RPC request. That's pretty inefficient so an alternative is to create a single callback queue per client.

Spring-amqp能让你专注于正在处理的消息类型，并隐藏了支持该类型的消息所需的消息管道的实现细节。例如，通常情况下，本地客户端会为每个RPC请求都创建一个回调队列。这种做法效率很低，所以替换方案是每个客户端只创建一个回调队列。

That raises a new issue, having received a response in that queue it's not clear to which request the response belongs. That's when the correlationId property is used. Spring-amqp automatically sets a unique value for every request. In addition it handles the details of matching the response with the correct correlationId.

One reason that spring-amqp makes rpc style easier is that sometimes you may want to ignore unknown messages in the callback queue, rather than failing with an error. It's due to a possibility of a race condition on the server side. Although unlikely, it is possible that the RPC server will die just after sending us the answer, but before sending an acknowledgment message for the request. If that happens, the restarted RPC server will process the request again. The spring-amqp client handles the duplicate responses gracefully, and the RPC should ideally be idempotent.

### Summary

![](https://www.rabbitmq.com/img/tutorials/python-six.png)

Our RPC will work like this:

我们的RPC系统

1.The Tut6Config will setup a new DirectExchange and a client

在Tut6Config文件里将建立一个新的DirectExchange和一个客户端。

2.The client will leverage the convertSendAndReceive passing the exchange name, the routingKey, and the message.

客户端将使用convertSendAndReceive，并传入交换器名字，路由键和消息。

3.The request is sent to an rpc\_queue\("tut.rpc"\) queue.

请求被发送到用于rpc的队列里（“tut.rpc”）。

4.The RPC worker \(aka: server\) is waiting for requests on that queue. When a request appears, it performs the task and sends a message with the result back to the Client, using the queue from the replyTo field.

RPC工作者（也就是服务器）等待发送到队列里的请求。但一个请求出现时，它就执行任务，然后通过使用replyTo域里配置的队列将带有结果的消息发回给客户端。

5.The client waits for data on the callback queue. When a message appears, it checks the correlationId property. If it matches the value from the request it returns the response to the application. Again, this is done automagically via the RabbitTemplate.

客户端等待回调队列里的数据。当一条消息出现时，它会校验correlationId属性。如果属性值与请求匹配，它就将响应返回给应用。这个工作RabbitTemplate自动帮我们完成了。

## Putting it all together（代码整合）

The Fibonacci task is a @RabbitListener and is defined as:

计算斐波那契的任务用@RabbitListener进行标注，任务内容的定义如下：

```java
public int fib(int n) {
    return n == 0 ? 0 : n == 1 ? 1 : (fib(n - 1) + fib(n - 2));
}
```

We declare our fibonacci function. It assumes only valid positive integer input. \(Don't expect this one to work for big numbers, and it's probably the slowest recursive implementation possible\).

我们声明了斐波那契函数。它假定输入的参数是有效的正整数。（不要期望它能用于大数的场景，而且这种方式是最低效的递归实现）。

The code for our [Tut6Config](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/spring-amqp/src/main/java/org/springframework/amqp/tutorials/tut6/Tut6Config.java) looks like this:

[Tut6Config](https://legacy.gitbook.com/book/jiapengcai/rabbitmq/edit#)的代码看起来是如下这样子的：

```java
import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

@Profile({"tut6","rpc"})
@Configuration
public class Tut6Config {

    @Profile("client")
    private static class ClientConfig {

        @Bean
        public DirectExchange exchange() {
            return new DirectExchange("tut.rpc");
        }

        @Bean
        public Tut6Client client() {
            return new Tut6Client();
        }

    }

    @Profile("server")
    private static class ServerConfig {

        @Bean
        public Queue queue() {
            return new Queue("tut.rpc.requests");
        }

        @Bean
        public DirectExchange exchange() {
            return new DirectExchange("tut.rpc");
        }

        @Bean
        public Binding binding(DirectExchange exchange, 
            Queue queue) {
            return BindingBuilder.bind(queue)
                .to(exchange)
                .with("rpc");
        }

        @Bean
        public Tut6Server server() {
            return new Tut6Server();
        }

    }
}
```

It setups up our profiles as "tut6" or "rpc". It also setups a "client" profile with two beans; 1\) the DirectExchange we are using and 2\) the Tut6Client itself. We also configure the "server" profile with three beans, the "tut.rpc.requests" queue, the DirectExchange, which matches the client's exchange, and the binding from the queue to the exchange with the "rpc" routing-key.

它建立了我们的配置组，叫“tut6”或者“rpc”。同时，还建立了一个“client”配置组，这个组里配置了两个bean：一个是我们将要用到的DirectExchange类型的交换器，一个是Tut6Client本身。我们还建立了一个“server”配置组，这个组里配置了三个bean：一个名为“tut.rpc.requests”的队列，一个与客户端交换器相匹配的DirectExchange类型的交换器，以及用名为“rpc”的路由键将队列和交换器的绑定器。

The server code is rather straightforward:

服务端代码更直观点：

1.As usual we start annotating our receiver method with a @RabbitListener and defining the queue its listening on.

像之前那样，我们先用@RabbitListener来注解我们的接收者方法，然后定义它要监听的队列。

2.Our fibanacci method calls fib\(\) with the payload parameter and returns the result.

我们的斐波那契方法被命名为fib\(\)，接收有效参数并返回结果。

The code for our RPC server [Tut6Server.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/spring-amqp/src/main/java/org/springframework/amqp/tutorials/tut6/Tut6Server.java):

以下为我们的RPC服务端代码[Tut6Server.java](https://legacy.gitbook.com/book/jiapengcai/rabbitmq/edit#):

```java
package org.springframework.amqp.tutorials.tut6;

import org.springframework.amqp.rabbit.annotation.RabbitListener;

public class Tut6Server {

    @RabbitListener(queues = "tut.rpc.requests")
    // @SendTo("tut.rpc.replies") used when the 
    // client doesn't set replyTo.
    public int fibonacci(int n) {
        System.out.println(" [x] Received request for " + n);
        int result = fib(n);
        System.out.println(" [.] Returned " + result);
        return result;
    }

    public int fib(int n) {
        return n == 0 ? 0 : n == 1 ? 1 : (fib(n - 1) + fib(n - 2));
    }

}
```

The client code [Tut6Client](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/spring-amqp/src/main/java/org/springframework/amqp/tutorials/tut6/Tut6Client.java) is as easy as the server:

客户端代码[Tut6Client](https://legacy.gitbook.com/book/jiapengcai/rabbitmq/edit#)与服务端代码一样简单：

1.We autowire the RabbitTemplate and the DirectExchange bean as defined in the Tut6Config.

我们自动注入Tut6Config里定义的类型为RabbitTemplate和DirectExchange的bean。

2.We invoke template.convertSendAndReceive with the parameters exchange name, routing key and message.

我们调用template.convertSendAndReceive，传入的参数为交换器名字，路由键以及消息。

3.We print the result.

打印出结果。

Making the Client request is simply:

```java
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;

public class Tut6Client {

    @Autowired
    private RabbitTemplate template;

    @Autowired
    private DirectExchange exchange;

    int start = 0;

    @Scheduled(fixedDelay = 1000, initialDelay = 500)
    public void send() {
        System.out.println(" [x] Requesting fib(" + start + ")");
        Integer response = (Integer) template.convertSendAndReceive
            (exchange.getName(), "rpc", start++);
        System.out.println(" [.] Got '" + response + "'");
    }
}
```

Using the project setup as defined in \(see tutorial one\) with start.spring.io and SpringInitialzr the preparing the runtime is the same as the other tutorials:

```
mvn clean package
```

We can start the server with:

```
java -jar target/rabbit-tutorials-1.7.1.RELEASE.jar 
    --spring.profiles.active=rpc,server 
    --tutorial.client.duration=6000
```

To request a fibonacci number run the client:

```
java -jar target/rabbit-tutorials-1.7.1.RELEASE.jar
    --spring.profiles.active=rpc,server
java -jar target/rabbit-tutorials-1.7.1.RELEASE.jar 
    --spring.profiles.active=rpc,client
```

The design presented here is not the only possible implementation of a RPC service, but it has some important advantages:

* If the RPC server is too slow, you can scale up by just running another one. Try running a second
  RPCServer in a new console.
* On the client side, the RPC requires sending and receiving only one message with one method. No synchronous calls like
  queueDeclare are required. As a result the RPC client needs only one network round trip for a single RPC request.

Our code is still pretty simplistic and doesn't try to solve more complex \(but important\) problems, like:

* How should the client react if there are no servers running?
* Should a client have some kind of timeout for the RPC?
* If the server malfunctions and raises an exception, should it be forwarded to the client?
* Protecting against invalid incoming messages \(eg checking bounds, type\) before processing.

> If you want to experiment, you may find the [management UI](https://www.rabbitmq.com/management.html) useful for viewing the queues.

There is one other nice feature of RabbitMQ. It is featured as a supported tile on Pivotal Cloud Foundry \(PCF\) as a service.

