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

一个队列就是一个存在RabbitMQ内部的邮箱的名字。虽然消息在传递时会通过RabbitMQ以及你的应用，但它们只能被存储在一个队列里。

