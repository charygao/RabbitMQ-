## Routing（路由）

In the previous tutorial we built a simple fanout exchange. We were able to broadcast messages to many receivers.

在上一个教程里，我们构建了一个简单的广播交换器。通过它我们能将消息广播到多个接收者。

In this tutorial we're going to add a feature to it - we're going to make it possible to subscribe only to a subset of the messages. For example, we will be able to direct only messages to the certain colors of interest \("orange", "black", "green"\), while still being able to print all of the messages on the console.

在本教程里，我们将往里添加一个功能——我们准备让接收者可以只订阅部分消息。例如，我们将只要某些我们感兴趣的颜色（“橙色”，“黑色”，“绿色”）的消息，但仍能在控制台里打印出所有的信息。

## Bindings（绑定）



