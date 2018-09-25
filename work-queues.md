# Work Queues

[教程二 原文地址: "Work Queues"](http://www.rabbitmq.com/tutorial-two-go.html)

在 [教程一](../hello-world.md) 中我们编写了一个从命名的队列中发送和读取消息的程序，在本教程中，我们将创建一个用于分发被多个工作者（worker）实时消费的工作队列（Work Queue）。

工作队列（又名任务队列）背后的主要思想是：避免立即处理一个资源集中型的任务并等到它完成为止，而是将其延后处理。我们将一个消息封装成一个任务（task）并将其发送到队列中，一个后台运行的工作线程（worker process）会从队列头部依次取出任务并执行相关操作。如果同时运行多个工作线程，则不同的任务会被分配到不同的工作线程中。

这个概念在那些通过短连接的 HTTP 请求处理复杂任务的 Web 应用程序中尤其有用。

## 准备

在本系列教程的教程一中，我们发送了一个包含"Hello World!"的消息，现在我们将发送代表复杂任务的字符串。本教程中，我们没有像图片大小调整或渲染 PDF 文件这样的真实任务，所以，我们通过使用 `time.Sleep` 函数伪造任务来假装很忙。我们使用字符串中点（dot）的个数来表示任务的复杂程度，每一个点将花费工作者一秒钟的时间，比如，一个由字符串 `Hello...` 伪造的任务将花费 3 秒钟的时间。

我们将稍稍改造前例中的 `send.go` 文件中的代码，从而让我们可以通过命令行来发送任意内容的消息。这个程序将任务调度到我们的工作队列中，我们将其命名为 `new_task.go` ：
```golang
body := bodyFrom(os.Args)
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

旧的 `receive.go` 文件同样需要调整：它需要伪造以秒为单位处理消息中每个点的工作任务，它需要从工作队列中依次取出消息并处理相应的工作，所以我们将其命名为 `worker.go` ：

```golang
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
    dot_count := bytes.Count(d.Body, []byte("."))
    t := time.Duration(dot_count)
    time.Sleep(t * time.Second)
    log.Printf("Done")
  }
}()

log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
<-forever
```
注意，我们的伪造任务模拟了执行时间。

运行操作如下：

```shell
# shell 1
go run worker.go
```

```shell
# shell 2
go run worker.go
```

## 轮询分发

任务队列的优势之一是它很容易地将任务平行分发。假设我们正在处理一个将积压的工作，我们可以简单地通过增加更多的工作者（worker）来解决它，扩展起来就是这么简单。

首先，我们同时运行两个 `work.go`，它们将同时从队列中获取消息，但具体如何获取呢？我们拭目以待。

你需要打开三个终端，其中两个运行 `worker.go`，这两个终端将作为我们的消费者 -- 用 C1 和 C2 表示。

```sh
# shell 1
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
```

```sh
# shell 2
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
```

在第三个终端中，我们发布新的任务。一旦你运行了消费者之后，你就可以发送一些消息了：

```sh
# shell 3
go run new_task.go First message.
go run new_task.go Second message..
go run new_task.go Third message...
go run new_task.go Fourth message....
go run new_task.go Fifth message.....
```

让我们来看看发送到消费者信息是什么：
```sh
# shell 1
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'First message.'
# => [x] Received 'Third message...'
# => [x] Received 'Fifth message.....'
```

```sh
# shell 2
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'Second message..'
# => [x] Received 'Fourth message....'
```

默认情况下，RabbitMQ 将顺序的把每个消息发送给下一个消费者。每个消费者将平均的获得相同数目的消息，这种分发消息的机制叫做轮询分发（round-robin）。可以尝试使用三个或更多的工作者来验证一下。

## 消息确认

处理任务需要花费几秒钟的时间，你或许对一个消费者处理一个长时间的任务但只处理了一部分时就终止的情况会怎样感到好奇。在我们目前的代码中，一旦 RabbitMQ 发送了一个消息给消费者，它就立即将这个消息标记为删除。这种情况下，如果终止相应的工作者的话，我们将丢失它正在处理的那条消息，同时，我们还会丢失所有分发给这个工作者的尚未处理的消息。

实际工作中，我们不想丢失任何任务，如果一个工作者被终止，我们希望任务将被分发到另一个工作者。

为了确保一条消息永不丢失，RabbitMQ 支持[消息确认](https://www.rabbitmq.com/confirms.html)。一个 ack（nowledgement) 是由接收、处理特定消息后消费者返回给 RabbitMQ 的，告诉 RabbitMQ 可以删除相应的消息。

如果消费者未返回一个 ack 而被终止（它的信道关闭，连接关闭或者 TCP 连接丢失），RabbitMQ 知道相关的消息没有被完整的处理并重新将其放入队列。如果此时有其它的消费者连接到队列上，RabbitMQ 将快速的将消息重新分发到另一个消费者。这样，就算有工作者意外终止，我们也可以保证没有消息丢失。

目前还没有考虑消息超时。RabbitMQ 会在当消费者意味终止是重新分发消息，甚至处理一个花费很长时间的消息也还算 OK。

本教程中，我们将通过设置 `auto-ack` 参数的值为 `false` 来让工作者手动返回消息确认，然后工作者在处理完任务时通过 `d.Ack(false)`（这个确认只针对于单个分发）将返回一个正确的确认。

```golang
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
    log.Printf("Received a message: %s", d.Body)
    dot_count := bytes.Count(d.Body, []byte("."))
    t := time.Duration(dot_count)
    time.Sleep(t * time.Second)
    log.Printf("Done")
    d.Ack(false)
  }
}()

log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
<-forever
```

使用上面的代码，就算通过 `CTRL+C` 终止一个正在处理消息的工作者，我们也可以保证没有消息丢失，一旦工作者被终止，所有的尚未确认的消息会被重新分发。

确认必须通过使用分发消息的同一个信道来返回，试图使用不同的信道来返回确认会导致信道级别的协议异常，访问 [doc guide on confirmations ](https://www.rabbitmq.com/confirms.html) 了解更多。

> Note:
> 
> **忘记确认**
> 
> 忘记 `ack` 是很常见的错误，犯错很简单，但后果很严重。当客户端终止时，消息将被（随机地）重新分发，但 RabbitMQ 将因为没有释放未应答的消息吃掉越来越多的内存。
> 
> 为了能够调试这类错误，可以使用 `rabbitmqctl` 来打印 `messages_unacknowledged` 字段：
> 
> ```sh
> sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
> ```
> 在 Windows 上：
> 
> ```sh
> rabbitmqctl.bat list_queues name messages_ready messages_unacknowledge
> ```

## 消息持久化

我们已经掌握了当消费者终止时怎样确保任务不会丢失，但当 RabbitMQ 服务停止时，任务还是会丢失。

当 RabbitMQ 关闭或崩溃时，它将丢失所有的队列和消息，除非你设置它不这样。为了确保消息不会丢失，需要做两件事情：我们需要标记队列和消息时持久化的。

首先，我们需要确保 RabbitMQ 永远不会丢失队列，为此，我们需要设置其为 `durable`：
```golang
q, err := ch.QueueDeclare(
  "hello",      // name
  true,         // durable
  false,        // delete when unused
  false,        // exclusive
  false,        // no-wait
  nil,          // arguments
)
failOnError(err, "Failed to declare a queue")
```

尽管上面的代码正确无误，但它不会按照我们的期望工作。这是因为我们已经声明了一个不是持久化的名为 `hello` 的队列了，RabbitMQ 不允许重新声明一个带有不同参数的同名队列，如果尝试这样声明，程序将会返回一个错误。但我们可以通过声明一个不同名字的队列来快速解决这个问题，比如 `task_queue`：

```golang
q, err := ch.QueueDeclare(
  "task_queue", // name
  true,         // durable
  false,        // delete when unused
  false,        // exclusive
  false,        // no-wait
  nil,          // arguments
)
failOnError(err, "Failed to declare a queue")
```

这里的 `durable` 选项需要在生成者和消费者代码中同时应用。

这一步中，我们确保了就算 RabbitMQ 重启 `task_queue` 队列也不会消失。现在，我们需要通过设置 `amqp.Publishing` 中的 `amqp.Persistent` 选项将消息标记为持久化的。

```golang
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
```

> Note:
> 
> **消息持久化**

> 将消息标记为持久化并不能完全保证消失不会丢失。尽管这样设置已经告知 RabbitMQ 将消息保存到磁盘中，但仍有当 RabbitMQ 接收了消息但没有将其保存到磁盘的时间窗口，并且 RabbitMQ 并不会对所有的消息执行 `fsync(2)` 函数 -- 这样有可能导致消息只保存到了缓存中而没有真正写入磁盘。持久化并未得到强保证，但是对于简单的任务队列已经足够。如果需要持久化更加强的保证，可以使用 [publisher confirms](https://www.rabbitmq.com/confirms.html)。

## 公平分发

你有可能注意到，有时消息并未按我们期望的那样的分发。设想这样一种情形：有两个工作者，奇数号的消息很重量而偶数号的消息很轻量，一个工作者会一直处于繁忙状态而另一个几乎没有什么工作。RabbitMQ 不会知晓并仍旧平等地分发消息。

导致这种情况是因为 RabbitMQ 当消息进入队列时仅仅分发消息，它不会考虑消费者的未确认消息的个数，它只是盲目地将第 n 号消息发送给第 n 号 消费者。

![2-1-prefetch-count](./pics/2-1-prefetch-count.png)

为了解决上述问题，我们可以将预取数设置为 1，这样我们告诉 RabbitMQ 不要同时分发给一个工作者超过 1 的消息。或者说，在工作者没有处理完任何并返回确认时不要给它分发新的消息，这样 RabbitMQ 将信息分发给了两外那个不忙的工作者。

```golang
err = ch.Qos(
  1,     // prefetch count
  0,     // prefetch size
  false, // global
)
failOnError(err, "Failed to set QoS")
```

> Note:
> 
> **队列容量**
> 
> 当所有的工作者都处于繁忙状态，队列有可能会被填满，这一点需要引起注意，可以通过增加更多的工作者或其它方法解决。

## 整体来看

最终的 `new_task.go` 代码：

```golang
package main

import (
        "fmt"
        "log"
        "os"
        "strings"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
                panic(fmt.Sprintf("%s: %s", msg, err))
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
                "task_queue", // name
                true,         // durable
                false,        // delete when unused
                false,        // exclusive
                false,        // no-wait
                nil,          // arguments
        )
        failOnError(err, "Failed to declare a queue")

        body := bodyFrom(os.Args)
        err = ch.Publish(
                "",           // exchange
                q.Name,       // routing key
                false,        // mandatory
                false,
                amqp.Publishing{
                        DeliveryMode: amqp.Persistent,
                        ContentType:  "text/plain",
                        Body:         []byte(body),
                })
        failOnError(err, "Failed to publish a message")
        log.Printf(" [x] Sent %s", body)
}

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

[new_task.go](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/new_task.go)

`worker.go` 代码：

```golang
package main

import (
        "bytes"
        "fmt"
        "github.com/streadway/amqp"
        "log"
        "time"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
                panic(fmt.Sprintf("%s: %s", msg, err))
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
                "task_queue", // name
                true,         // durable
                false,        // delete when unused
                false,        // exclusive
                false,        // no-wait
                nil,          // arguments
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
                        log.Printf("Received a message: %s", d.Body)
                        dot_count := bytes.Count(d.Body, []byte("."))
                        t := time.Duration(dot_count)
                        time.Sleep(t * time.Second)
                        log.Printf("Done")
                        d.Ack(false)
                }
        }()

        log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
        <-forever
}
```

[worker.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/worker.go)

通过消息确认和设置预取数，我们可以创建一个工作队列，而持久化选项可以在 RabbitMQ 重启的情况下保存我们的任务。

更多关于 `amqp.Channel` 方法和消息的属性，请参考 [amqp API reference](http://godoc.org/github.com/streadway/amqp)。

接下来，请移步至 [教程三](./work-queues.md) 来掌握怎样将同一消息发送给很多消费者。