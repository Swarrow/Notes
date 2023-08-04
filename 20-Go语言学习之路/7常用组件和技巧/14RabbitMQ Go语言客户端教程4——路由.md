# RabbitMQ Go语言客户端教程4——路由

> https://www.liwenzhou.com/posts/Go/go_rabbitmq_tutorials_04/

本文翻译自[RabbitMQ官网的Go语言客户端系列教程](https://www.rabbitmq.com/getstarted.html)，共分为六篇，本文是第四篇——路由。

这些教程涵盖了使用RabbitMQ创建消息传递应用程序的基础知识。 你需要安装RabbitMQ服务器才能完成这些教程，请参阅[安装指南](https://www.rabbitmq.com/download.html)或使用[Docker镜像](https://registry.hub.docker.com/_/rabbitmq/)。 这些教程的代码是[开源](https://github.com/rabbitmq/rabbitmq-tutorials)的，[官方网站](https://github.com/rabbitmq/rabbitmq-website)也是如此。

## 先决条件

本教程假设RabbitMQ已安装并运行在本机上的标准端口（5672）。如果你使用不同的主机、端口或凭据，则需要调整连接设置。

## 路由

**（使用Go RabbitMQ客户端）**

在[上一教程](https://www.liwenzhou.com/posts/Go/go_rabbitmq_tutorials_03/)中，我们构建了一个简单的日志记录系统。我们能够向许多接收者广播日志消息。

在本教程中，我们将向它添加一个特性-我们将使它能够只订阅消息的一个子集。例如，我们将只能将关键错误消息定向到日志文件（以节省磁盘空间），同时仍然能够在控制台上打印所有日志消息。

### 绑定

在前面的示例中，我们已经在创建绑定。你可能会想起以下代码：

```go
err = ch.QueueBind(
  q.Name, // queue name
  "",     // routing key
  "logs", // exchange
  false,
  nil)
```

绑定是交换器和队列之间的关系。这可以简单地理解为：队列对来自此交换器的消息感兴趣。

绑定可以采用额外的`routing_key`参数。为了避免与`Channel.Publish`参数混淆，我们将其称为`binding key`。这是我们如何使用键创建绑定的方法：

```go
err = ch.QueueBind(
  q.Name,    // queue name
  "black",   // routing key
  "logs",    // exchange
  false,
  nil)
```

绑定密钥的含义取决于交换器的类型。我们以前使用的`fanout`交换器只是忽略了这个值。

### 直连交换器

我们上一个教程中的日志系统将所有消息广播给所有消费者。我们希望扩展这一点，允许根据消息的严重性过滤消息。例如，我们可能希望将日志消息写入磁盘的脚本只接收严重错误，而不会在warning或info日志消息上浪费磁盘空间。

我们使用`fanout`交换器，这并没有给我们很大的灵活性——它只能进行无脑广播。

我们将使用`direct`交换器。`direct`交换器背后的路由算法很简单——消息进入其`binding key`与消息的`routing key`完全匹配的队列。

为了说明这一点，请考虑以下设置：

![direct-exchang](https://www.liwenzhou.com/images/Go/rabbitmq/tutorials04/direct-exchange.png)

在此设置中，我们可以看到绑定了两个队列的`direct`交换器`X`。第一个队列绑定键为`orange`，第二个队列绑定为两个，一个绑定键为`black`，另一个为`green`。

在这种设置中，使用`orange`路由键发布到交换器的消息将被路由到队列`Q1`。路由键为`black`或`green`的消息将转到`Q2`。所有其他消息将被丢弃。

### 多重绑定

![direct-exchange-multiple](https://www.liwenzhou.com/images/Go/rabbitmq/tutorials04/direct-exchange-multiple.png)

用相同的绑定键绑定多个队列是完全合法的。在我们的示例中，我们可以使用绑定键`black`在`X`和`Q1`之间添加绑定。在这种情况下，`direct`交换器的行为将类似`fanout`，并将消息广播到所有匹配的队列。带有`black`路由键的消息将同时传递给`Q1`和`Q2`。

### 发送日志

我们将在日志系统中使用这个模型。我们将发送消息到`direct`交换器，而不是`fanout`。我们将提供严重性（译注：通常我们使用日志级别划分日志信息的严重性）作为路由键。这样，接收脚本将能够选择其想要接收的日志级别。让我们首先关注发送日志。

与往常一样，我们需要首先创建一个交换器：

```go
err = ch.ExchangeDeclare(
  "logs_direct", // name
  "direct",      // type
  true,          // durable
  false,         // auto-deleted
  false,         // internal
  false,         // no-wait
  nil,           // arguments
)
```

我们已经准备好发送一条消息：

```go
err = ch.ExchangeDeclare(
  "logs_direct", // name
  "direct",      // type
  true,          // durable
  false,         // auto-deleted
  false,         // internal
  false,         // no-wait
  nil,           // arguments
)
failOnError(err, "Failed to declare an exchange")

body := bodyFrom(os.Args)
err = ch.Publish(
  "logs_direct",         // exchange
  severityFrom(os.Args), // routing key
  false, // mandatory
  false, // immediate
  amqp.Publishing{
    ContentType: "text/plain",
    Body:        []byte(body),
})
```

为了简化问题，我们假设“严重性”可以是“info”、“warning”、“error”之一。

### 订阅

接收消息的工作方式与上一教程一样，但有一个例外——我们将为感兴趣的每种严重性（日志级别）创建一个新的绑定。

```go
q, err := ch.QueueDeclare(
  "",    // name
  false, // durable
  false, // delete when unused
  true,  // exclusive
  false, // no-wait
  nil,   // arguments
)
failOnError(err, "Failed to declare a queue")

if len(os.Args) < 2 {
  log.Printf("Usage: %s [info] [warning] [error]", os.Args[0])
  os.Exit(0)
}
// 建立多个绑定关系
for _, s := range os.Args[1:] {
  log.Printf("Binding queue %s to exchange %s with routing key %s",
     q.Name, "logs_direct", s)
  err = ch.QueueBind(
    q.Name,        // queue name
    s,             // routing key
    "logs_direct", // exchange
    false,
    nil)
  failOnError(err, "Failed to bind a queue")
}
```

### 完整示例

![img](https://www.liwenzhou.com/images/Go/rabbitmq/tutorials04/python-four.png)

`emit_log_direct.go`脚本的代码：

```go
package main

import (
        "log"
        "os"
        "strings"

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
                "logs_direct", // name
                "direct",      // type
                true,          // durable
                false,         // auto-deleted
                false,         // internal
                false,         // no-wait
                nil,           // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        body := bodyFrom(os.Args)
        err = ch.Publish(
                "logs_direct",         // exchange
                severityFrom(os.Args), // routing key
                false, // mandatory
                false, // immediate
                amqp.Publishing{
                        ContentType: "text/plain",
                        Body:        []byte(body),
                })
        failOnError(err, "Failed to publish a message")

        log.Printf(" [x] Sent %s", body)
}

func bodyFrom(args []string) string {
        var s string
        if (len(args) < 3) || os.Args[2] == "" {
                s = "hello"
        } else {
                s = strings.Join(args[2:], " ")
        }
        return s
}

func severityFrom(args []string) string {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "info"
        } else {
                s = os.Args[1]
        }
        return s
}
```

`receive_logs_direct.go`的代码：

```go
package main

import (
        "log"
        "os"

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
                "logs_direct", // name
                "direct",      // type
                true,          // durable
                false,         // auto-deleted
                false,         // internal
                false,         // no-wait
                nil,           // arguments
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

        if len(os.Args) < 2 {
                log.Printf("Usage: %s [info] [warning] [error]", os.Args[0])
                os.Exit(0)
        }
        for _, s := range os.Args[1:] {
                log.Printf("Binding queue %s to exchange %s with routing key %s",
                        q.Name, "logs_direct", s)
                err = ch.QueueBind(
                        q.Name,        // queue name
                        s,             // routing key
                        "logs_direct", // exchange
                        false,
                        nil)
                failOnError(err, "Failed to bind a queue")
        }

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                true,   // auto ack
                false,  // exclusive
                false,  // no local
                false,  // no wait
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

如果你只想将“warning”和“err”（而不是“info”）级别的日志消息保存到文件中，只需打开控制台并输入：

```bash
go run receive_logs_direct.go warning error > logs_from_rabbit.log
```

如果你想在屏幕上查看所有日志消息，请打开一个新终端并执行以下操作：

```bash
go run receive_logs_direct.go info warning error
# => [*] Waiting for logs. To exit press CTRL+C
```

例如，要发出`error`日志消息，只需输入：

```bash
go run emit_log_direct.go error "Run. Run. Or it will explode."
# => [x] Sent 'error':'Run. Run. Or it will explode.'
```

（这里是（[emit_log_direct.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/emit_log_direct.go))和（[receive_logs_direct.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive_logs_direct.go)）的完整源码）

继续学习[教程5](https://www.liwenzhou.com/posts/Go/go_rabbitmq_tutorials_05/)，了解如何根据模式监听消息。