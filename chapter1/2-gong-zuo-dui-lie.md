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



