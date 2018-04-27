# 2 工作队列

![](http://www.rabbitmq.com/img/tutorials/python-two.png)

In the first tutorial we wrote programs to send and receive messages from a named queue. In this one we'll create a _Work_ _Queue_ that will be used to distribute time-consuming tasks among multiple workers.

The main idea behind Work Queues \(aka: _Task Queues_\) is to avoid doing a resource-intensive task immediately and having to wait for it to complete. Instead we schedule the task to be done later. We encapsulate a _task _as a message and send it to a queue. A worker process running in the background will pop the tasks and eventually execute the job. When you run many workers the tasks will be shared between them.

