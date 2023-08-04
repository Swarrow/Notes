# RabbitMQ Go语言客户端教程2——工作队列

> https://www.liwenzhou.com/posts/Go/go_rabbitmq_tutorials_02/

本文翻译自[RabbitMQ官网的Go语言客户端系列教程](https://www.rabbitmq.com/getstarted.html)，共分为六篇，本文是第二篇——任务队列。

这些教程涵盖了使用RabbitMQ创建消息传递应用程序的基础知识。 你需要安装RabbitMQ服务器才能完成这些教程，请参阅[安装指南](https://www.rabbitmq.com/download.html)或使用[Docker镜像](https://registry.hub.docker.com/_/rabbitmq/)。 这些教程的代码是[开源](https://github.com/rabbitmq/rabbitmq-tutorials)的，[官方网站](https://github.com/rabbitmq/rabbitmq-website)也是如此。

## 先决条件

本教程假设RabbitMQ已安装并运行在本机上的标准端口（5672）。如果你使用不同的主机、端口或凭据，则需要调整连接设置。

## 任务队列/工作队列

**（使用Go RabbitMQ客户端）**

![img](https://www.liwenzhou.com/images/Go/rabbitmq/tutorials02/python-two.png)

在[第一个教程](https://www.liwenzhou.com/posts/Go/go_rabbitmq_tutorials_01/)中，我们编写程序从命名的队列发送和接收消息。在这一节中，我们将创建一个工作队列，该队列将用于在多个工人之间分配耗时的任务。

工作队列（又称任务队列）的主要思想是避免立即执行某些资源密集型任务并且不得不等待这些任务完成。相反，我们安排任务异步地同时或在当前任务之后完成。我们将任务封装为消息并将其发送到队列，在后台运行的工作进程将取出消息并最终执行任务。当你运行多个工作进程时，任务将在他们之间共享。

这个概念在Web应用中特别有用，因为在Web应用中不可能在较短的HTTP请求窗口内处理复杂的任务，（译注：例如注册时发送邮件或短信验证码等场景）。

### 准备工作

在本教程的上一部分，我们发送了一条包含“ Hello World！”的消息。现在，我们将发送代表复杂任务的字符串。我们没有实际的任务，例如调整图像大小或渲染pdf文件，所以我们通过借助`time.Sleep`函数模拟一些比较耗时的任务。我们会将一些包含`.`的字符串封装为消息发送到队列中，其中每有一个`.`就表示需要耗费1秒钟的工作，例如，`hello...`表示一个将花费三秒钟的假任务。

我们将稍微修改上一个示例中的`send.go`代码，以允许从命令行发送任意消息。该程序会将任务安排到我们的工作队列中，因此我们将其命名为`new_task.go`

```go
body := bodyFrom(os.Args)  // 从参数中获取要发送的消息正文
err = ch.Publish(
  "",           // exchange
  q.Name,       // routing key
  false,        // mandatory
  false,
  amqp.Publishing {
    DeliveryMode: amqp.Persistent,
    ContentType:  "text/plain",
    Body:         []byte(body),
  })
failOnError(err, "Failed to publish a message")
log.Printf(" [x] Sent %s", body)
```

下面是`bodyFrom`函数：

```go
func bodyFrom(args []string) string {
    var s string
    if (len(args) < 2) || os.Args[1] == "" {
        s = "hello"
    } else {
        s = strings.Join(args[1:], " ")
    }
    return s
}
```

我们以前的`receive.go`程序也需要进行一些更改：它需要为消息正文中出现的每个`.`伪造一秒钟的工作。它将从队列中弹出消息并执行任务，因此我们将其称为`worker.go`：

```go
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
    dot_count := bytes.Count(d.Body, []byte("."))  // 数一下有几个.
    t := time.Duration(dot_count)
    time.Sleep(t * time.Second)  // 模拟耗时的任务
    log.Printf("Done")
  }
}()

log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
<-forever
```

请注意，我们的假任务模拟执行时间。

然后，我们就可以打开两个终端，分别执行`new_task.go`和`worker.go`了。

```bash
# shell 1
go run worker.go
# shell 2
go run new_task.go
```

### 循环调度

使用任务队列的优点之一是能够轻松并行化工作。如果我们的工作正在积压，我们可以增加更多的工人，这样就可以轻松扩展。

首先，让我们尝试同时运行两个`worker.go`脚本。它们都将从队列中获取消息，但是究竟是怎样呢？让我们来看看。

你需要打开三个控制台。其中两个将运行`worker.go`脚本。这些控制台将成为我们的两个消费者——C1和C2。

```bash
# shell 1
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
# shell 2
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
```

在第三个控制台中，我们将发布新任务。启动消费者之后，你可以发布一些消息：

```bash
# shell 3
go run new_task.go msg1.
go run new_task.go msg2..
go run new_task.go msg3...
go run new_task.go msg4....
go run new_task.go msg5.....
```

然后我们在`shell1`和 `shell2` 两个窗口看到如下输出结果了：

```bash
# shell 1
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received a message: msg1.
# => [x] Received a message: msg3...
# => [x] Received a message: msg5.....
# shell 2
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received a message: msg2..
# => [x] Received a message: msg4....
```

默认情况下，RabbitMQ将按顺序将每个消息发送给下一个消费者。平均而言，每个消费者都会收到相同数量的消息。这种分发消息的方式称为轮询。使用三个或者更多`worker`试一下。

### 消息确认

work 完成任务可能需要耗费几秒钟，如果一个`worker`在任务执行过程中宕机了该怎么办呢？我们当前的代码中，RabbitMQ一旦向消费者传递了一条消息，便立即将其标记为删除。在这种情况下，如果你终止一个`worker`那么你就可能会丢失这个任务，我们还将丢失所有已经交付给这个`worker`的尚未处理的消息。

我们不想丢失任何任务，如果一个`worker`意外宕机了，那么我们希望将任务交付给其他`worker`来处理。

为了确保消息永不丢失，RabbitMQ支持 [消息*确认*](https://www.rabbitmq.com/confirms.html)。消费者发送回一个确认（acknowledgement），以告知RabbitMQ已经接收，处理了特定的消息，并且RabbitMQ可以自由删除它。

如果使用者在不发送确认的情况下死亡（其通道已关闭，连接已关闭或TCP连接丢失），RabbitMQ将了解消息未完全处理，并将对其重新排队。如果同时有其他消费者在线，它将很快将其重新分发给另一个消费者。这样，您可以确保即使工人偶尔死亡也不会丢失任何消息。

没有任何消息超时；RabbitMQ将在消费者死亡时重新传递消息。即使处理一条消息需要很长时间也没关系。

在本教程中，我们将使用手动消息确认，方法是为“auto-ack”参数传递一个`false`，然后在完成任务后，使用`d.Ack(false)`从`worker`发送一个正确的确认（这将确认一次传递）。

```go
msgs, err := ch.Consume(
	q.Name, // queue
	"",     // consumer
	false,  // 注意这里传false,关闭自动消息确认
	false,  // exclusive
	false,  // no-local
	false,  // no-wait
	nil,    // args
)
if err != nil {
	fmt.Printf("ch.Consume failed, err:%v\n", err)
	return
}

// 开启循环不断地消费消息
forever := make(chan bool)
go func() {
	for d := range msgs {
		log.Printf("Received a message: %s", d.Body)
		dotCount := bytes.Count(d.Body, []byte("."))
		t := time.Duration(dotCount)
		time.Sleep(t * time.Second)
		log.Printf("Done")
		d.Ack(false) // 手动传递消息确认
	}
}()
```

使用这段代码，我们可以确保即使你在处理消息时使用`CTRL+C`杀死一个`worker`，也不会丢失任何内容。在`worker`死后不久，所有未确认的消息都将被重新发送。

消息确认必须在接收消息的同一通道（Channel）上发送。尝试使用不同的通道（Channel）进行消息确认将导致通道级协议异常。有关更多信息，请参阅[确认的文档指南](https://www.rabbitmq.com/confirms.html)。

> 忘记确认
>
> 忘记确认是一个常见的错误。这是一个简单的错误，但后果是严重的。当你的客户机退出时，消息将被重新传递（这看起来像随机重新传递），但是RabbitMQ将消耗越来越多的内存，因为它无法释放任何未确认的消息。
>
> 为了调试这种错误，可以使用rabbitmqctl打印messages_unacknowledged字段：
>
> ~~~bash
> > sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
> > ```
> >
> > 在Windows平台，去掉sudo
> >
> > ```bash
> > rabbitmqctl.bat list_queues name messages_ready messages_unacknowledged
> > ```
> 
> ### 消息持久化
> 
> 我们已经学会了如何确保即使消费者死亡，任务也不会丢失。但是如果RabbitMQ服务器停止运行，我们的任务仍然会丢失。
> 
> 当RabbitMQ退出或崩溃时，它将忘记队列和消息，除非您告诉它不要这样做。要确保消息不会丢失，需要做两件事：我们需要将队列和消息都标记为持久的。
> 
> 首先，我们需要确保队列能够在RabbitMQ节点重新启动后继续运行。为此，我们需要声明它是持久的：
> 
> ```go
> q, err := ch.QueueDeclare(
> 	"hello", // name
> 	true,    // 声明为持久队列
> 	false,   // delete when unused
> 	false,   // exclusive
> 	false,   // no-wait
> 	nil,     // arguments
> )
> ~~~

虽然这个命令本身是正确的，但它在我们当前的设置中不起作用。这是因为我们已经定义了一个名为`hello`的队列，它不是持久的。RabbitMQ不允许你使用不同的参数重新定义现有队列，并将向任何尝试重新定义的程序返回错误。但是有一个快速的解决方法——让我们声明一个具有不同名称的队列，例如`task_queue`：

```go
q, err := ch.QueueDeclare(
	"task_queue", // name
	true,         // 声明为持久队列
	false,        // delete when unused
	false,        // exclusive
	false,        // no-wait
	nil,          // arguments
)
```

这种持久的选项更改需要同时应用于生产者代码和消费者代码。

在这一点上，我们确信即使RabbitMQ重新启动，任务队列队列也不会丢失。现在我们需要将消息标记为持久的——通过使用`amqp.Publishing`中的持久性选项`amqp.Persistent`。

```go
err = ch.Publish(
	"",     // exchange
	q.Name, // routing key
	false,  // 立即
	false,  // 强制
	amqp.Publishing{
		DeliveryMode: amqp.Persistent, // 持久（交付模式：瞬态/持久）
		ContentType:  "text/plain",
		Body:         []byte(body),
	})
```

> 有关消息持久性的说明
>
> 将消息标记为持久性并不能完全保证消息不会丢失。尽管它告诉RabbitMQ将消息保存到磁盘上，但是RabbitMQ接受了一条消息并且还没有保存它时，仍然有一个很短的时间窗口。而且，RabbitMQ并不是对每个消息都执行`fsync(2)`——它可能只是保存到缓存中，而不是真正写入磁盘。持久性保证不是很强，但是对于我们的简单任务队列来说已经足够了。如果您需要更强有力的担保，那么您可以使用[publisher confirms](https://www.rabbitmq.com/confirms.html)。

### 公平分发

你可能已经注意到调度仍然不能完全按照我们的要求工作。例如，在一个有两个`worker`的情况下，当所有的奇数消息都是重消息而偶数消息都是轻消息时，一个`worker`将持续忙碌，而另一个`worker`几乎不做任何工作。嗯，RabbitMQ对此一无所知，仍然会均匀地发送消息。

这是因为RabbitMQ只是在消息进入队列时发送消息。它不考虑消费者未确认消息的数量。只是盲目地向消费者发送信息。

![img](https://www.liwenzhou.com/images/Go/rabbitmq/tutorials02/prefetch-count.png)

为了避免这种情况，我们可以将预取计数设置为`1`。这告诉RabbitMQ不要一次向一个`worker`发出多个消息。或者，换句话说，在处理并确认前一条消息之前，不要向`worker`发送新消息。相反，它将把它发送给下一个不忙的`worker`。

```go
err = ch.Qos(
  1,     // prefetch count
  0,     // prefetch size
  false, // global
)
```

> 关于队列大小的说明
>
> 如果所有的`worker`都很忙，你的`queue`随时可能会满。你会想继续关注这一点，也许需要增加更多的`worker`，或者有一些其他的策略。

### 完整的代码示例

我们的`new_task.go`的最终代码代入如下：

```go
package main

import (
	"fmt"
	"log"
	"os"
	"strings"

	"github.com/streadway/amqp"
)

func main() {
	// 1. 尝试连接RabbitMQ，建立连接
	// 该连接抽象了套接字连接，并为我们处理协议版本协商和认证等。
	conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
	if err != nil {
		fmt.Printf("connect to RabbitMQ failed, err:%v\n", err)
		return
	}
	defer conn.Close()

	// 2. 接下来，我们创建一个通道，大多数API都是用过该通道操作的。
	ch, err := conn.Channel()
	if err != nil {
		fmt.Printf("open a channel failed, err:%v\n", err)
		return
	}
	defer ch.Close()

	// 3. 要发送，我们必须声明要发送到的队列。
	q, err := ch.QueueDeclare(
		"task_queue", // name
		true,         // 持久的
		false,        // delete when unused
		false,        // 独有的
		false,        // no-wait
		nil,          // arguments
	)
	if err != nil {
		fmt.Printf("declare a queue failed, err:%v\n", err)
		return
	}

	// 4. 然后我们可以将消息发布到声明的队列
	body := bodyFrom(os.Args)
	err = ch.Publish(
		"",     // exchange
		q.Name, // routing key
		false,  // 立即
		false,  // 强制
		amqp.Publishing{
			DeliveryMode: amqp.Persistent, // 持久
			ContentType:  "text/plain",
			Body:         []byte(body),
		})
	if err != nil {
		fmt.Printf("publish a message failed, err:%v\n", err)
		return
	}
	log.Printf(" [x] Sent %s", body)
}

// bodyFrom 从命令行获取将要发送的消息内容
func bodyFrom(args []string) string {
	var s string
	if (len(args) < 2) || os.Args[1] == "" {
		s = "hello"
	} else {
		s = strings.Join(args[1:], " ")
	}
	return s
}
```

`work.go`内容如下：

```go
package main

import (
	"bytes"
	"fmt"
	"log"
	"time"

	"github.com/streadway/amqp"
)

func main() {
	conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
	if err != nil {
		fmt.Printf("connect to RabbitMQ failed, err:%v\n", err)
		return
	}
	defer conn.Close()

	ch, err := conn.Channel()
	if err != nil {
		fmt.Printf("open a channel failed, err:%v\n", err)
		return
	}
	defer ch.Close()

	// 声明一个queue
	q, err := ch.QueueDeclare(
		"task_queue", // name
		true,         // 声明为持久队列
		false,        // delete when unused
		false,        // exclusive
		false,        // no-wait
		nil,          // arguments
	)
	err = ch.Qos(
		1,     // prefetch count
		0,     // prefetch size
		false, // global
	)
	if err != nil {
		fmt.Printf("ch.Qos() failed, err:%v\n", err)
		return
	}

	// 立即返回一个Delivery的通道
	msgs, err := ch.Consume(
		q.Name, // queue
		"",     // consumer
		false,  // 注意这里传false,关闭自动消息确认
		false,  // exclusive
		false,  // no-local
		false,  // no-wait
		nil,    // args
	)
	if err != nil {
		fmt.Printf("ch.Consume failed, err:%v\n", err)
		return
	}

	// 开启循环不断地消费消息
	forever := make(chan bool)
	go func() {
		for d := range msgs {
			log.Printf("Received a message: %s", d.Body)
			dotCount := bytes.Count(d.Body, []byte("."))
			t := time.Duration(dotCount)
			time.Sleep(t * time.Second)
			log.Printf("Done")
			d.Ack(false) // 手动传递消息确认
		}
	}()

	log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
	<-forever
}
```

使用消息确认和预取计数，可以设置工作队列（work queue）。即使RabbitMQ重新启动，持久性选项也可以让任务继续存在。

有关`amqp.Channel`方法和消息属性的内容，可以浏览[amqp API文档](http://godoc.org/github.com/streadway/amqp)。