# 2 工作队列（Work Queues）

![](http://www.rabbitmq.com/img/tutorials/python-two.png)

In the first tutorial we wrote programs to send and receive messages from a named queue. In this one we'll create a _Work_ _Queue_ that will be used to distribute time-consuming tasks among multiple workers.

在第一个教程中，我们编写了程序用于往一个命名了的队列发送消息，同时也从这个队列里接收消息。在本教程里，我们将创建一个工作队列，它将被用来在多个工作者之间分发耗时任务。

The main idea behind Work Queues \(aka: _Task Queues_\) is to avoid doing a resource-intensive task immediately and having to wait for it to complete. Instead we schedule the task to be done later. We encapsulate a _task_ as a message and send it to a queue. A worker process running in the background will pop the tasks and eventually execute the job. When you run many workers the tasks will be shared between them.



This concept is especially useful in web applications where it's impossible to handle a complex task during a short HTTP request window.

