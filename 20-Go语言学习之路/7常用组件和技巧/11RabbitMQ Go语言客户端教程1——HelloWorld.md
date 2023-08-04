# RabbitMQ Go语言客户端教程1——HelloWorld

> https://www.liwenzhou.com/posts/Go/go_rabbitmq_tutorials_01/

本文翻译自[RabbitMQ官网的Go语言客户端系列教程](https://www.rabbitmq.com/getstarted.html)，共分为六篇，本文是第一篇——HelloWorld。

这些教程涵盖了使用RabbitMQ创建消息传递应用程序的基础知识。 你需要安装RabbitMQ服务器才能完成这些教程，请参阅[安装指南](https://www.rabbitmq.com/download.html)或使用[Docker镜像](https://registry.hub.docker.com/_/rabbitmq/)。 这些教程的代码是[开源](https://github.com/rabbitmq/rabbitmq-tutorials)的，[官方网站](https://github.com/rabbitmq/rabbitmq-website)也是如此。

## 先决条件

本教程假设RabbitMQ已安装并运行在本机上的标准端口（5672）。如果你使用不同的主机、端口或凭据，则需要调整连接设置。

## RabbitMQ Go语言客户端教程（一）

### 介绍

RabbitMQ是一个消息代理：它接受并转发消息。你可以把它想象成一个邮局：当你把你想要邮寄的邮件放进一个邮箱时，你可以确定邮差先生或女士最终会把邮件送到你的收件人那里。在这个比喻中，RabbitMQ是一个邮箱、一个邮局和一个邮递员。

RabbitMQ和邮局的主要区别在于它不处理纸张，而是接受、存储和转发二进制数据块——消息。

RabbitMQ和一般的消息传递都使用一些术语。

- 生产仅意味着发送。发送消息的程序是生产者：

![img](https://www.liwenzhou.com/images/Go/rabbitmq/tutorials01/producer.png)

- 队列是位于RabbitMQ内部的邮箱的名称。尽管消息通过RabbitMQ和你的应用程序流动，但它们只能存储在队列中。队列只受主机内存和磁盘限制的限制，实际上它是一个大的消息缓冲区。许多生产者可以向一个队列发送消息，而许多消费者可以尝试从一个队列接收数据。以下是我们表示队列的方式：

![img](https://www.liwenzhou.com/images/Go/rabbitmq/tutorials01/queue.png)

- 消费与接收具有相似的含义。消费者是一个主要等待接收消息的程序：

![img](https://www.liwenzhou.com/images/Go/rabbitmq/tutorials01/consumer.png)

请注意，生产者，消费者和代理（broker）不必位于同一主机上。实际上，在大多数应用程序中它们不是。一个应用程序既可以是生产者，也可以是消费者。

### “Hello World”

**(使用Go RabbitMQ客户端）**

在本教程的这一部分中，我们将在Go中编写两个小程序：发送单个消息的生产者和接收消息并将其打印出来的消费者。我们将忽略[Go-RabbitMQ](http://godoc.org/github.com/streadway/amqp) API中的一些细节，只关注非常简单的事情，以便开始教程。这是一个消息传递版的“Hello World”。

在下图中，“ P”是我们的生产者，“ C”是我们的消费者。中间的框是一个队列——RabbitMQ代表消费者保存的消息缓冲区。

![(P) -> [|||] -> (C)](https://www.liwenzhou.com/images/Go/rabbitmq/tutorials01/python-one.png)

> Go RabbitMQ客户端库
>
> RabbitMQ讲多种协议。本教程使用amqp0-9-1，这是一个开放的、通用的消息传递协议。RabbitMQ有[许多不同语言的客户端](http://rabbitmq.com/devtools.html)。在本教程中，我们将使用Go amqp客户端。

首先，使用`go get`安装amqp

```bash
go get github.com/streadway/amqp
```

现在安装好amqp之后，我们就可以编写一些代码。

### 发送

![(P) -> [|||]](https://www.liwenzhou.com/images/Go/rabbitmq/tutorials01/sending.png)

我们将消息发布者（发送者）称为 `send.go`，将消息消费者（接收者）称为`receive.go`。发布者将连接到RabbitMQ，发送一条消息，然后退出。

在`send.go`中，我们需要首先导入库：

```go
package main

import (
  "log"

  "github.com/streadway/amqp"
)
```

我们还需要一个辅助函数来检查每个amqp调用的返回值：

```go
func failOnError(err error, msg string) {
  if err != nil {
    log.Fatalf("%s: %s", msg, err)
  }
}
```

然后连接到RabbitMQ服务器

```go
// 1. 尝试连接RabbitMQ，建立连接
// 该连接抽象了套接字连接，并为我们处理协议版本协商和认证等。
conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
failOnError(err, "Failed to connect to RabbitMQ")
defer conn.Close()
```

连接抽象了socket连接，并为我们处理协议版本协商和认证等。接下来，我们创建一个通道，这是大多数用于完成任务的API所在的位置：

```go
// 2. 接下来，我们创建一个通道，大多数API都是用过该通道操作的。
ch, err := conn.Channel()
failOnError(err, "Failed to open a channel")
defer ch.Close()
```

要发送，我们必须声明要发送到的队列。然后我们可以将消息发布到队列：

```go
// 3. 声明消息要发送到的队列
q, err := ch.QueueDeclare(
  "hello", // name
  false,   // durable
  false,   // delete when unused
  false,   // exclusive
  false,   // no-wait
  nil,     // arguments
)
failOnError(err, "Failed to declare a queue")

body := "Hello World!"
// 4.将消息发布到声明的队列
err = ch.Publish(
  "",     // exchange
  q.Name, // routing key
  false,  // mandatory
  false,  // immediate
  amqp.Publishing {
    ContentType: "text/plain",
    Body:        []byte(body),
  })
failOnError(err, "Failed to publish a message")
```

声明队列是幂等的——仅当队列不存在时才创建。消息内容是一个字节数组，因此你可以在此处编码任何内容。

[点击查看完整的send.go文件](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/send.go)

### 接收

上面是我们的发布者。我们的消费者监听来自RabbitMQ的消息，因此与发布单个消息的发布者不同，我们将使消费者保持运行状态以监听消息并打印出来。

![[|||] -> (C)](https://www.liwenzhou.com/images/Go/rabbitmq/tutorials01/receiving.png)

该代码（在`receive.go`中）具有与`send`相同的导入和帮助功能：

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
```

设置与发布者相同；我们打开一个连接和一个通道，并声明要消耗的队列。请注意，这与`send`发布到的队列匹配。

```go
// 建立连接
conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
failOnError(err, "Failed to connect to RabbitMQ")
defer conn.Close()

// 获取channel
ch, err := conn.Channel()
failOnError(err, "Failed to open a channel")
defer ch.Close()

// 声明队列
q, err := ch.QueueDeclare(
  "hello", // name
  false,   // durable
  false,   // delete when unused
  false,   // exclusive
  false,   // no-wait
  nil,     // arguments
)
failOnError(err, "Failed to declare a queue")
```

请注意，我们也在这里声明队列。因为我们可能在发布者之前启动使用者，所以我们希望在尝试使用队列中的消息之前确保队列存在。

我们将告诉服务器将队列中的消息传递给我们。由于它将异步地向我们发送消息，因此我们将在goroutine中从通道（由`amqp::Consume`返回）中读取消息。

```go
// 获取接收消息的Delivery通道
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
    log.Printf("Received a message: %s", d.Body)
  }
}()

log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
<-forever
```

[点击完整的receive.go脚本](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive.go)

### 完整示例

现在我们可以运行两个脚本。在一个终端窗口，运行发布者：

```bash
go run send.go
```

然后，运行使用者：

```go
go run receive.go
```

消费者将打印通过RabbitMQ从发布者那里得到的消息。使用者将持续运行，等待消息（使用Ctrl-C停止它），因此请尝试从另一个终端运行发布者。

如果要检查队列，请尝试使用`rabbitmqctl list_queues`命令。

接下来该继续教程的[第二部分](https://www.liwenzhou.com/posts/Go/go_rabbitmq_tutorials_02/)并建立一个简单的任务队列。