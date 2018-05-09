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



