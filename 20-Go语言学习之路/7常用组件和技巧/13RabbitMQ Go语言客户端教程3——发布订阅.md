# RabbitMQ Go语言客户端教程3——发布/订阅

> https://www.liwenzhou.com/posts/Go/go_rabbitmq_tutorials_03/

本文翻译自[RabbitMQ官网的Go语言客户端系列教程](https://www.rabbitmq.com/getstarted.html)，共分为六篇，本文是第三篇——发布/订阅。

这些教程涵盖了使用RabbitMQ创建消息传递应用程序的基础知识。 你需要安装RabbitMQ服务器才能完成这些教程，请参阅[安装指南](https://www.rabbitmq.com/download.html)或使用[Docker镜像](https://registry.hub.docker.com/_/rabbitmq/)。 这些教程的代码是[开源](https://github.com/rabbitmq/rabbitmq-tutorials)的，[官方网站](https://github.com/rabbitmq/rabbitmq-website)也是如此。

## 先决条件

本教程假设RabbitMQ已安装并运行在本机上的标准端口（5672）。如果你使用不同的主机、端口或凭据，则需要调整连接设置。

## 发布/订阅

在[上一个教程](https://www.liwenzhou.com/posts/Go/go_rabbitmq_tutorials_02/)中，我们创建了一个工作队列。工作队列背后的假设是每个任务只传递给一个工人。在这一部分中，我们将做一些完全不同的事情——我们将向多个消费者传递一个消息。这就是所谓的“订阅/发布模式”。

为了说明这种模式，我们将构建一个简单的日志系统。它将由两个程序组成——第一个程序将发出日志消息，第二个程序将接收并打印它们。

在我们的日志系统中，每一个运行的接收器程序副本都会收到消息。这样，我们就可以运行一个接收器并将日志定向到磁盘；同时，我们还可以运行另一个接收器并在屏幕上查看日志。

本质上，已发布的日志消息将被广播到所有接收者。

### Exchanges（交换器）

在本教程的前面部分中，我们向队列发送消息和从队列接收消息。现在是时候在Rabbit中引入完整的消息传递模型了。

让我们快速回顾一下先前教程中介绍的内容：

- *生产者*是发送消息的用户应用程序。
- *队列*是存储消息的缓冲区。
- *消费者*是接收消息的用户应用程序。

RabbitMQ消息传递模型中的核心思想是生产者从不将任何消息直接发送到队列。实际上，生产者经常甚至根本不知道是否将消息传递到任何队列。

相反，生产者只能将消息发送到交换器。交换器是非常简单的东西。一方面，它接收来自生产者的消息，另一方面，将它们推入队列。交换器必须确切知道如何处理接收到的消息。它应该被附加到特定的队列吗？还是应该将其附加到许多队列中？或者它应该被丢弃。这些规则由交换器的类型定义。

![img](https://www.liwenzhou.com/images/Go/rabbitmq/tutorials03/exchanges.png)

有几种交换器类型可用：`direct`, `topic`, `headers` 和 `fanout`。我们将集中讨论最后一个——`fanout`。让我们创建一个这种类型的交换器，并给它起个名字叫`logs`：

```go
err = ch.ExchangeDeclare(
  "logs",   // name
  "fanout", // type
  true,     // durable
  false,    // auto-deleted
  false,    // internal
  false,    // no-wait
  nil,      // arguments
)
```

`fanout`（扇出）交换器非常简单。正如你可能从名称中猜测的那样，它只是将接收到的所有消息广播到它知道的所有队列中。而这正是我们记录器所需要的。

> **交换器清单**
>
> 要列出服务器上的交换器，你可以执行有用的rabbitmqctl命令：
>
> ~~~bash
> > sudo rabbitmqctl list_exchanges
> > ```
> >
> > 在此列表中，将有一些`amq.*`交换器和一个默认的（未命名）交换器。这些是默认创建的，但是你现在不太可能需要使用它们。
> >
> > **默认交换器**
> >
> > 在本教程的前面部分中，我们还不知道交换器的存在，但仍然能够将消息发送到队列。之所以能这样做，是因为我们使用的是默认交换器，该交换器由空字符串（`""`）标识。
> >
> > 回想一下我们之前是怎么发布消息的：
> >
> > ```go
> > err = ch.Publish(
> >   "",     // exchange
> >   q.Name, // routing key
> >   false,  // mandatory
> >   false,  // immediate
> >   amqp.Publishing{
> >     ContentType: "text/plain",
> >     Body:        []byte(body),
> > })
> > ```
> >
> > 在这里，我们使用默认或无名称的交换器：消息将以`route_key`参数指定的名称路由到队列（如果存在）。
> 
> 现在，我们可以改为发布到我们的命名交换器：
> 
> ```go
> err = ch.ExchangeDeclare(
>   "logs",   // 使用命名的交换器
>   "fanout", // 交换器类型
>   true,     // durable
>   false,    // auto-deleted
>   false,    // internal
>   false,    // no-wait
>   nil,      // arguments
> )
> failOnError(err, "Failed to declare an exchange")
> 
> body := bodyFrom(os.Args)
> err = ch.Publish(
>   "logs", // exchange
>   "",     // routing key
>   false,  // mandatory
>   false,  // immediate
>   amqp.Publishing{
>           ContentType: "text/plain",
>           Body:        []byte(body),
>   })
> ~~~

### 临时队列

你可能还记得，先前我们使用的是具有特定名称的队列（还记得`hello`和`task_queue`吗？）能够命名队列对我们来说至关重要——我们需要将工作人员指向同一个队列。当你想在生产者和消费者之间共享队列时，给队列一个名称非常重要。

但对于我们的记录器来说，情况并非如此。我们希望收到所有日志消息，而不仅仅是它们的一部分。我们也只对当前正在发送的消息感兴趣，而对旧消息不感兴趣。为了解决这个问题，我们需要两件事。

首先，当我们连接到Rabbit时，我们需要一个新的、空的队列。为此，我们可以创建一个随机名称的队列，或者更好的方法是让服务器为我们选择一个随机队列名称。

其次，一旦我们断开消费者的连接，队列就会自动删除。

在[amqp](http://godoc.org/github.com/streadway/amqp)客户端中，当我们传递一个空字符串作为队列名称时，我们将使用随机生成的名称创建一个非持久队列：

```go
q, err := ch.QueueDeclare(
  "",    // 空字符串作为队列名称
  false, // 非持久队列
  false, // delete when unused
  true,  // 独占队列（当前声明队列的连接关闭后即被删除）
  false, // no-wait
  nil,   // arguments
)
```

上述方法返回时，生成的队列实例包含RabbitMQ生成的随机队列名称。例如，它可能看起来像`amq.gen-JzTY20BRgKO-HjmUJj0wLg`。

当声明它的连接关闭时，该队列将被删除，因为它被声明为独占。

你可以在[队列指南](https://www.rabbitmq.com/queues.html)中了解有关`exclusive`标志和其他队列属性的更多信息。

### 绑定

![img](https://www.liwenzhou.com/images/Go/rabbitmq/tutorials03/bindings.png)

我们已经创建了一个扇出交换器和一个队列。现在我们需要告诉交换器将消息发送到我们的队列。交换器和队列之间的关系称为*绑定*。

```go
err = ch.QueueBind(
  q.Name, // queue name
  "",     // routing key
  "logs", // exchange
  false,
  nil,
)
```

从现在开始，`logs`交换器将会把消息添加到我们的队列中。

> 列出绑定关系
>
> 你猜也猜到了，我们可以使用下面的命令列出绑定关系
>
> ~~~bash
> > rabbitmqctl list_bindings
> > ```
> 
> ### 完整示例
> 
> ![img](/images/Go/rabbitmq/tutorials03/python-three-overall.png)
> 
> 产生日志消息的生产程序与上一教程看起来没有太大不同。最重要的变化是我们现在希望将消息发布到`logs`交换器，而不是空的消息交换器。发送时，我们需要提供一个`routingKey`，但是对于`fanout`型交换器，它的值可以被忽略（传空字符串）。下面是`emit_log.go`脚本的代码：
> 
> ```go
> package main
> 
> import (
>         "log"
>         "os"
>         "strings"
> 
>         "github.com/streadway/amqp"
> )
> 
> func failOnError(err error, msg string) {
>         if err != nil {
>                 log.Fatalf("%s: %s", msg, err)
>         }
> }
> 
> func main() {
>         conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
>         failOnError(err, "Failed to connect to RabbitMQ")
>         defer conn.Close()
> 
>         ch, err := conn.Channel()
>         failOnError(err, "Failed to open a channel")
>         defer ch.Close()
> 
>         err = ch.ExchangeDeclare(
>                 "logs",   // name
>                 "fanout", // type
>                 true,     // durable
>                 false,    // auto-deleted
>                 false,    // internal
>                 false,    // no-wait
>                 nil,      // arguments
>         )
>         failOnError(err, "Failed to declare an exchange")
> 
>         body := bodyFrom(os.Args)
>         err = ch.Publish(
>                 "logs", // exchange
>                 "",     // routing key
>                 false,  // mandatory
>                 false,  // immediate
>                 amqp.Publishing{
>                         ContentType: "text/plain",
>                         Body:        []byte(body),
>                 })
>         failOnError(err, "Failed to publish a message")
> 
>         log.Printf(" [x] Sent %s", body)
> }
> 
> func bodyFrom(args []string) string {
>         var s string
>         if (len(args) < 2) || os.Args[1] == "" {
>                 s = "hello"
>         } else {
>                 s = strings.Join(args[1:], " ")
>         }
>         return s
> }
> ~~~

（[emit_logs.go源码](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/emit_log.go)）

如你所见，在建立连接之后，我们声明了交换器。此步骤是必需的，因为禁止发布到不存在的交换器。

如果没有队列绑定到交换器，那么消息将丢失，但这对我们来说是ok的。如果没有消费者在接收，我们可以安全地丢弃该消息。

`receive_logs.go`的代码：

```go
package main

import (
        "log"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        err = ch.ExchangeDeclare(
                "logs",   // name
                "fanout", // type
                true,     // durable
                false,    // auto-deleted
                false,    // internal
                false,    // no-wait
                nil,      // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        q, err := ch.QueueDeclare(
                "",    // name
                false, // durable
                false, // delete when unused
                true,  // exclusive
                false, // no-wait
                nil,   // arguments
        )
        failOnError(err, "Failed to declare a queue")

        err = ch.QueueBind(
                q.Name, // queue name
                "",     // routing key
                "logs", // exchange
                false,
                nil,
        )
        failOnError(err, "Failed to bind a queue")

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                true,   // auto-ack
                false,  // exclusive
                false,  // no-local
                false,  // no-wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        forever := make(chan bool)

        go func() {
                for d := range msgs {
                        log.Printf(" [x] %s", d.Body)
                }
        }()

        log.Printf(" [*] Waiting for logs. To exit press CTRL+C")
        <-forever
}
```

（[receive_logs.go源码](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive_logs.go)）

如果要将日志保存到文件，只需打开控制台并输入：

```bash
go run receive_logs.go > logs_from_rabbit.log
```

如果希望在屏幕上查看日志，请切换到一个新的终端并运行：

```bash
go run receive_logs.go
```

当然，要发出日志，请输入：

```bash
go run emit_log.go
```

使用`rabbitmqctl list_bindings`命令，你可以验证代码是否确实根据需要创建了绑定关系和队列。在运行两个`receive_logs.go`程序后，你应该看到类似以下内容：

```bash
sudo rabbitmqctl list_bindings
# => Listing bindings ...
# => logs    exchange        amq.gen-JzTY20BRgKO-HjmUJj0wLg  queue           []
# => logs    exchange        amq.gen-vso0PVvyiRIL2WoV3i48Yg  queue           []
# => ...done.
```

对结果的解释很简单：数据从`logs`交换器进入了两个由服务器分配名称的队列。这正是我们想要的。

要了解如何侦听消息的子集，让我们继续学习[教程4](https://www.liwenzhou.com/posts/Go/go_rabbitmq_tutorials_04/)。