# RabbitMQ Go语言客户端教程5——topic

> https://www.liwenzhou.com/posts/Go/go_rabbitmq_tutorials_05/

本文翻译自[RabbitMQ官网的Go语言客户端系列教程](https://www.rabbitmq.com/getstarted.html)，共分为六篇，本文是第五篇——topic。

这些教程涵盖了使用RabbitMQ创建消息传递应用程序的基础知识。 你需要安装RabbitMQ服务器才能完成这些教程，请参阅[安装指南](https://www.rabbitmq.com/download.html)或使用[Docker镜像](https://registry.hub.docker.com/_/rabbitmq/)。 这些教程的代码是[开源](https://github.com/rabbitmq/rabbitmq-tutorials)的，[官方网站](https://github.com/rabbitmq/rabbitmq-website)也是如此。

## 先决条件

本教程假设RabbitMQ已安装并运行在本机上的标准端口（5672）。如果你使用不同的主机、端口或凭据，则需要调整连接设置。

### topic交换器（主题交换器）

发送到`topic`交换器的消息不能具有随意的`routing_key`——它必须是单词列表，以点分隔。这些词可以是任何东西，但通常它们指定与消息相关的某些功能。一些有效的`routing_key`示例：“`stock.usd.nyse`”，“`nyse.vmw`”，“`quick.orange.rabbit`”。`routing_key`中可以包含任意多个单词，最多255个字节。

绑定键也必须采用相同的形式。`topic`交换器背后的逻辑类似于`direct`交换器——用特定路由键发送的消息将传递到所有匹配绑定键绑定的队列。但是，绑定键有两个重要的特殊情况： - `*`（星号）可以代替一个单词。 - `＃`（井号）可以替代零个或多个单词。

通过下面这个示例可以很容易看明白这一点：

![img](https://www.liwenzhou.com/images/Go/rabbitmq/tutorials05/python-five.png)

在这个例子中，我们将发送一些都是描述动物的信息。将使用包含三个词（两个点）的路由密钥发送消息。路由键中的第一个单词将描述速度，第二个是颜色，第三个是种类：“`..`”。

我们创建了三个绑定关系：Q1与绑定键“`*.orange.*`”绑定，Q2与“`*.*.rabbit`”和“`lazy.＃`”绑定。

这些绑定可以总结为：

- Q1对所有橙色动物都感兴趣。
- Q2想接收有关兔子（rabbit）的一切消息，以及有关懒惰（lazy）动物的一切消息。

路由键设置为“`quick.orange.rabbit`”的消息将传递到两个队列。消息“`lazy.orange.elephant`”也将发送给他们两个。另一方面，“`quick.orange.fox`”将仅进入第一个队列，而“`lazy.brown.fox`”将仅进入第二个队列。即使“`lazy.pink.rabbit`”与两个绑定匹配（匹配Q2的两个绑定），也只会传递到第二个队列一次。 “`quick.brown.fox`”与任何绑定都不匹配，因此将被丢弃。

如果我们打破约定并发送一个或四个单词的消息，例如“`orange`”或“`quick.orange.male.rabbit`”，会发生什么？好吧，这些消息将不匹配任何绑定，并且将会丢失。

另外，“`lazy.orange.male.rabbit`”即使有四个单词，也将匹配最后一个绑定，并将其传送到第二个队列。

> topic交换器
>
> topic交换器功能强大，可以像其他交换器一样运行。
>
> 当队列用“`#`”（井号）绑定键绑定时，它将接收所有消息，而与路由键无关，就像在`fanout`交换器中一样。
>
> 当在绑定中不使用特殊字符“`*`”（星号）和“`#`”（井号）时，topic交换器的行为就像`direct`交换器一样。

### 完整示例

我们将在日志记录系统中使用`topic`交换器。我们将从一个可行的假设开始，即日志的路由键将包含两个词：“`.`”。

该代码与[上一教程](https://www.liwenzhou.com/posts/Go/go_rabbitmq_tutorials_04/)中的代码几乎相同。

`emit_log_topic.go`的代码：

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
                "logs_topic", // name
                "topic",      // type
                true,         // durable
                false,        // auto-deleted
                false,        // internal
                false,        // no-wait
                nil,          // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        body := bodyFrom(os.Args)
        err = ch.Publish(
                "logs_topic",          // exchange
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
                s = "anonymous.info"
        } else {
                s = os.Args[1]
        }
        return s
}
```

`receive_logs_topic.go`的代码：

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
                "logs_topic", // name
                "topic",      // type
                true,         // durable
                false,        // auto-deleted
                false,        // internal
                false,        // no-wait
                nil,          // arguments
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
                log.Printf("Usage: %s [binding_key]...", os.Args[0])
                os.Exit(0)
        }
  			// 绑定topic
        for _, s := range os.Args[1:] {
                log.Printf("Binding queue %s to exchange %s with routing key %s",
                        q.Name, "logs_topic", s)
                err = ch.QueueBind(
                        q.Name,       // queue name
                        s,            // routing key
                        "logs_topic", // exchange
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

想要接收所有的日志：

```bash
go run receive_logs_topic.go "#"
```

要从“`kern`”接收所有日志：

```bahs
go run receive_logs_topic.go "kern.*"
```

或者，如果你只想接收“`critical`”日志：

```bash
go run receive_logs_topic.go "*.critical"
```

你可以创建多个绑定：

```bash
go run receive_logs_topic.go "kern.*" "*.critical"
```

并发出带有路由键“`kern.critical`”的日志：

```bash
go run emit_log_topic.go "kern.critical" "A critical kernel error"
```

你可以自己尝试玩一下这个程序。请注意，代码没有对路由键或绑定键进行任何假设，你可能希望使用两个以上的路由键参数。

（关于[emit_log_topic.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/emit_log_topic.go)和[receive_logs_topic.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive_logs_topic.go)的完整源代码）

接下来，我们将在[教程6](https://www.liwenzhou.com/posts/Go/go_rabbitmq_tutorials_06/)中了解如何将往返消息用作远程过程调用。