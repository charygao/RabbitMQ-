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



