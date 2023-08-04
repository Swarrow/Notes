# RabbitMQ Go语言客户端教程6——RPC

> https://www.liwenzhou.com/posts/Go/go_rabbitmq_tutorials_06/

本文翻译自[RabbitMQ官网的Go语言客户端系列教程](https://www.rabbitmq.com/getstarted.html)，共分为六篇，本文是第六篇——RPC。

这些教程涵盖了使用RabbitMQ创建消息传递应用程序的基础知识。 你需要安装RabbitMQ服务器才能完成这些教程，请参阅[安装指南](https://www.rabbitmq.com/download.html)或使用[Docker镜像](https://registry.hub.docker.com/_/rabbitmq/)。 这些教程的代码是[开源](https://github.com/rabbitmq/rabbitmq-tutorials)的，[官方网站](https://github.com/rabbitmq/rabbitmq-website)也是如此。

## 先决条件

本教程假设RabbitMQ已安装并运行在本机上的标准端口（5672）。如果你使用不同的主机、端口或凭据，则需要调整连接设置。

## 远程过程调用（RPC）

（使用Go RabbitMQ客户端）

在第二个教程中，我们学习了如何使用工作队列在多个worker之间分配耗时的任务。

但是，如果我们需要在远程计算机上运行函数并等待结果怎么办？好吧，那是一个不同的故事。这种模式通常称为*远程过程调用*或*RPC*。

在本教程中，我们将使用RabbitMQ构建一个RPC系统：客户端和可伸缩RPC服务器。由于我们没有值得分配的耗时任务，因此我们将创建一个虚拟RPC服务，该服务返回斐波那契数。

> 有关RPC的说明
>
> 尽管RPC是计算中非常常见的模式，但它经常受到批评。
>
> 当程序员不知道函数调用是本地的还是缓慢的RPC时，就会出现问题。这样的混乱会导致系统变幻莫测，并给调试增加了不必要的复杂性。滥用RPC可能会导致无法维护的意大利面条式代码而不是简化软件，
>
> 牢记这一点，请考虑以下建议：
>
> - 确定哪个函数调用是本地的，哪个是远程的。
> - 为你的系统编写文档。明确组件之间的依赖关系。
> - 处理错误情况。 当RPC服务器长时间关闭时，客户端应如何处理？

### 回调队列

通常，通过RabbitMQ进行RPC很容易。客户端发送请求消息，服务器发送响应消息。为了接收响应，我们需要发送带有“回调”队列地址的请求。我们可以使用默认队列。让我们尝试一下：

```go
q, err := ch.QueueDeclare(
  "",    // 不指定队列名，默认使用随机生成的队列名
  false, // durable
  false, // delete when unused
  true,  // exclusive
  false, // noWait
  nil,   // arguments
)

err = ch.Publish(
  "",          // exchange
  "rpc_queue", // routing key
  false,       // mandatory
  false,       // immediate
  amqp.Publishing{
    ContentType:   "text/plain",
    CorrelationId: corrId,
    ReplyTo:       q.Name,  // 在这里指定callback队列名，也是在这个队列等回复
    Body:          []byte(strconv.Itoa(n)),
})
```

> 消息属性
>
> AMQP 0-9-1协议预定义了消息附带的14个属性集。除以下属性外，大多数属性很少使用：
>
> - `persistent`：将消息标记为持久性（值为`true`）或瞬态（`false`）。你可能还记得第二个教程中的此属性。
> - `content_type`：用于描述编码的mime类型。例如，对于经常使用的JSON编码，将此属性设置为`application/ json`是一个好习惯。
> - `reply_to`：常用于命名回调队列
> - `correlation_id`：有助于将RPC响应与请求相关联

### 关联ID（Correlation Id）

在上面介绍的方法中，我们建议为每个RPC请求创建一个回调队列。这是相当低效的，但是幸运的是，有一种更好的方法——让我们为每个客户端创建一个回调队列。

这就引发了一个新问题，在该队列中收到响应后，尚不清楚响应属于哪个请求。这个时候就该使用`correlation_id`这个属性了。针对每个请求我们将为其设置一个唯一值。随后，当我们在回调队列中收到消息时，我们将查看该属性，并基于这个属性将响应与请求进行匹配。如果我们看到未知的`correlation_id`值，则可以放心地丢弃该消息——它不属于我们的请求。

你可能会问，为什么我们应该忽略回调队列中的未知消息，而不是报错而失败？这是由于服务器端可能出现竞争状况。尽管可能性不大，但RPC服务器可能会在向我们发送答案之后但在发送请求的确认消息之前死亡。如果发生这种情况，重新启动的RPC服务器将再次处理该请求。这就是为什么在客户端上我们必须妥善处理重复的响应，并且理想情况下RPC应该是幂等的。

### 总结

![img](https://www.liwenzhou.com/images/Go/rabbitmq/tutorials06/python-six.png)

我们的RPC工作流程如下：

- 客户端启动时，它将创建一个匿名排他回调队列。
- 对于RPC请求，客户端发送一条消息，该消息具有两个属性：`reply_to`（设置为回调队列）和`correlation_id`（设置为每个请求的唯一值）。
- 该请求被发送到`rpc_queue`队列。
- RPC工作程序（又名：服务器）正在等待该队列上的请求。当出现请求时，它会完成计算工作并把结果作为消息使用`replay_to`字段中的队列发回给客户端。
- 客户端等待回调队列上的数据。出现消息时，它将检查`correlation_id`属性。如果它与请求中的值匹配，则将响应返回给应用程序。

### 完整示例

斐波那契函数：

```go
func fib(n int) int {
        if n == 0 {
                return 0
        } else if n == 1 {
                return 1
        } else {
                return fib(n-1) + fib(n-2)
        }
}
```

声明我们的斐波那契函数。它仅假设有效的正整数输入。 （不要指望这种方法适用于大量用户，它可能是最慢的递归实现）。

我们的RPC服务器[rpc_server.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/rpc_server.go)的代码如下所示：

```go
package main

import (
        "log"
        "strconv"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
        }
}

func fib(n int) int {
        if n == 0 {
                return 0
        } else if n == 1 {
                return 1
        } else {
                return fib(n-1) + fib(n-2)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        q, err := ch.QueueDeclare(
                "rpc_queue", // name
                false,       // durable
                false,       // delete when unused
                false,       // exclusive
                false,       // no-wait
                nil,         // arguments
        )
        failOnError(err, "Failed to declare a queue")

        err = ch.Qos(
                1,     // prefetch count
                0,     // prefetch size
                false, // global
        )
        failOnError(err, "Failed to set QoS")

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                false,  // auto-ack
                false,  // exclusive
                false,  // no-local
                false,  // no-wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        forever := make(chan bool)

        go func() {
                for d := range msgs {
                        n, err := strconv.Atoi(string(d.Body))
                        failOnError(err, "Failed to convert body to integer")

                        log.Printf(" [.] fib(%d)", n)
                        response := fib(n)

                        err = ch.Publish(
                                "",        // exchange
                                d.ReplyTo, // routing key
                                false,     // mandatory
                                false,     // immediate
                                amqp.Publishing{
                                        ContentType:   "text/plain",
                                        CorrelationId: d.CorrelationId,
                                        Body:          []byte(strconv.Itoa(response)),
                                })
                        failOnError(err, "Failed to publish a message")

                        d.Ack(false)
                }
        }()

        log.Printf(" [*] Awaiting RPC requests")
        <-forever
}
```

服务器代码非常简单：

- 与往常一样，我们首先建立连接，通道并声明队列。
- 我们可能要运行多个服务器进程。为了将负载平均分配给多个服务器，我们需要在通道上设置`prefetch`设置。
- 我们使用`Channel.Consume`获取去队列，我们从队列中接收消息。然后，我们进入goroutine进行工作，并将响应发送回去。

我们的RPC客户端[rpc_client.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/rpc_client.go)的代码：

```go
package main

import (
        "log"
        "math/rand"
        "os"
        "strconv"
        "strings"
        "time"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
        }
}

func randomString(l int) string {
        bytes := make([]byte, l)
        for i := 0; i < l; i++ {
                bytes[i] = byte(randInt(65, 90))
        }
        return string(bytes)
}

func randInt(min int, max int) int {
        return min + rand.Intn(max-min)
}

func fibonacciRPC(n int) (res int, err error) {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        q, err := ch.QueueDeclare(
                "",    // name
                false, // durable
                false, // delete when unused
                true,  // exclusive
                false, // noWait
                nil,   // arguments
        )
        failOnError(err, "Failed to declare a queue")

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

        corrId := randomString(32)

        err = ch.Publish(
                "",          // exchange
                "rpc_queue", // routing key
                false,       // mandatory
                false,       // immediate
                amqp.Publishing{
                        ContentType:   "text/plain",
                        CorrelationId: corrId,
                        ReplyTo:       q.Name,
                        Body:          []byte(strconv.Itoa(n)),
                })
        failOnError(err, "Failed to publish a message")

        for d := range msgs {
                if corrId == d.CorrelationId {
                        res, err = strconv.Atoi(string(d.Body))
                        failOnError(err, "Failed to convert body to integer")
                        break
                }
        }

        return
}

func main() {
        rand.Seed(time.Now().UTC().UnixNano())

        n := bodyFrom(os.Args)

        log.Printf(" [x] Requesting fib(%d)", n)
        res, err := fibonacciRPC(n)
        failOnError(err, "Failed to handle RPC request")

        log.Printf(" [.] Got %d", res)
}

func bodyFrom(args []string) int {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "30"
        } else {
                s = strings.Join(args[1:], " ")
        }
        n, err := strconv.Atoi(s)
        failOnError(err, "Failed to convert arg to integer")
        return n
}
```

现在是时候看看[rpc_client.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/rpc_client.go)和[rpc_server.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/rpc_server.go)的完整示例源代码了。

我们的RPC服务现已准备就绪。我们可以启动服务器：

```bash
go run rpc_server.go
# => [x] Awaiting RPC requests
```

要请求斐波那契数，请运行客户端：

```bash
go run rpc_client.go 30
# => [x] Requesting fib(30)
```

这里介绍的设计不是RPC服务的唯一可能的实现，但是它具有一些重要的优点：

- 如果RPC服务器太慢，则可以通过运行另一台RPC服务器来进行扩展。尝试在新控制台中运行另一个`rpc_server.go`。
- 在客户端，RPC只需要发送和接收一条消息。结果，RPC客户端只需要一个网络往返就可以处理单个RPC请求。

我们的代码仍然非常简单，并且不会尝试解决更复杂（但很重要）的问题，例如：

- 如果没有服务器在运行，客户端应如何反应？
- 客户端是否应该为RPC设置某种超时时间？
- 如果服务器发生故障并引发异常，是否应该将其转发给客户端？
- 在处理之前防止无效的传入消息（例如检查边界，类型）。

> 如果要进行实验，可能会发现[管理后台界面](https://www.rabbitmq.com/management.html)对于查看队列很有用。