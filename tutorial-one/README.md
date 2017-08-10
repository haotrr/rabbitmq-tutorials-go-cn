# "Hello World!"
[教程一 原文地址: "Hello World!"](http://www.rabbitmq.com/tutorial-one-go.html)

## 介绍
RabbitMQ 是一个消息代理：接收然后投递消息。你可以把它看做一个邮局：你把你想要及的邮件投递到邮箱里，显然邮递员最终会将你的邮件投递到收件人手上。类似地，RabbitMQ 就像是一个邮箱，一个邮局或者一个邮递员。

RabbitMQ 与邮局的最大的区别在于，RabbitMQ 不处理信件而是接收、储存和传递二进制数据 -- 消息。

RabbitMQ，或者更一般的消息，使用了一些专业术语。
> `生产`代指发送，一个发送消息的应用程序则被称为`生产者`。

![1-1-producer.png](../pics/1-1-producer.png)
> 一个`队列`像是 RabbitMQ 里面的一个邮箱。尽管消息通过 RabbitMQ 在你的应用间相互传递，但消息只能保存在`队列`中。一个`队列`大小只受限于主机的内存和磁盘大小，它本质上是一个大的消息缓存。不同的`生产者`可以将消息发送到同一个队列，不同的`消费者`也可以从同一队列获取数据。以下图示代表了一个队列。

![1-2-queue.png](../pics/1-2-queue.png)
> `消费`代指接收，`消费者`指的是一个等待接收消息的应用程序。

![1-3-consumer.png](../pics/1-3-consumer.png)

注意生产者、消费者和消息代理不一定需要存在于同一台主机上，且对大部分应用程序来说，这才是常态。

## “Hello World”
本教程中，我们使用 Go 创建两个小程序：一个生产者用于发送一条简单的消息，一个消费者接收信息本将其打印。作为开始，我们将（gloss over）详细阐述几个 [Go RabbitMQ](https://godoc.org/github.com/streadway/amqp) API, 将主要集中精力在简单的事情上：发送一条 “Hello World” 的消息。

在下面的图示中，“P” 代表生产者，“C” 代表消费者，中间的盒子代表一个队列 -- RabbitMQ 中为消费者保存消息的消息缓存。

![1-4-p-q-c-model.png](../pics/1-4-p-q-c-model.png)

> Note:
> 
> **Go RabbitMQ 客户端库**
> 
> RabbitMQ 提供多种协议。本教程中使用开源、general-purpose 消息通信协议 AMQP 0-9-1。RabbitMQ 在页面 [RabbitMQ - Clients & Developer Tools](http://rabbitmq.com/devtools.html) 提供了一系列语言的客户端库。本教程中，我们使用 Go AMQP 客户端库。
>
> 使用 `go get` 安装 AMQP Go 客户端库：
> ```go
> go get github.com/streadway/amqp
> ```

既然已经安装了 AMQP 客户端库，那么接下来我们便可以写代码了。

## 发送
![1-5-sending.png](../pics/1-5-sending.png)

我们将消息的发布者（发送者）命名为 `seng.go`，将消息的消费者（接收者）命名为 `receive.go`。发布者会连接到 RabbitMQ，发送一条消息，然后退出。

在 `send.go` 中，我们首先导入相关包：
```go
package main

import (
  "fmt"
  "log"

  "github.com/streadway/amqp"
)
```
接着，我们声明一个用于检查每一次 AMQP 库函数调用返回值的帮助函数：
```go
func failOnError(err error, msg string) {
  if err != nil {
    log.Fatalf("%s: %s", msg, err)
    panic(fmt.Sprintf("%s: %s", msg, err))
  }
}
```
然后，连接 RabbitMQ 服务器：
```go
conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
failOnError(err, "Failed to connect to RabbitMQ")
defer conn.Close()
```
上面代码中的连接 abstracts 了 Socket 连接，将诸如协议握手、身份认证等等抽象化了。接下来，我们创建一个信道，我们所有的 API 调用都将基于信道：
```go
ch, err := conn.Channel()
failOnError(err, "Failed to open a channel")
defer ch.Close()
```
为了发送消息，我们还必须声明一个消息将要发往的队列；然后我们才能将消息发送到队列中：
```go
q, err := ch.QueueDeclare(
  "hello", // name
  false,   // durable
  false,   // delete when unused
  false,   // exclusive
  false,   // no-wait
  nil,     // arguments
)
failOnError(err, "Failed to declare a queue")

body := "hello"
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
声明一个队列时幂等的 -- 只有当队列不存在的时候，才会去创建这个队列。消息内容是一个字节数组（byte array），这样我们就能将任何想要的数据编码在里面了。

这里是完整的 [send.go](../tutorail-one/send.go) 文件。

> Note:
>
> **发送并没有成功！**
>
> 如果这是你第一次使用 RabbitMQ 并且没有看到“发送”出去的消息，你可能会想是不是哪里出现了错误。可能的原因是消息代理启动的时候没有分配到做够的磁盘空间（默认情况下至少需要 200 MB ）从而无法接收消息。检查消息代理的日志确认或者必要时增大限制值。页面 [RabbitMQ - RabbitMQ Configuration](https://www.rabbitmq.com/configure.html#config-items) 展示了如何修改 `disk_free_limit` 的值。
>

## 接收
以上就是一个完整的发布者。而消费者将从 RabbitMQ 拉取消息，所以不想发布者那样发送一条消息，消费者将一直监听消息并将其打印出来。

![1-6-receiving.png](../pics/1-6-receiving.png)

[receive.go](../tutorial-one/receive.go) 代码中与 send 中有相同的导入和帮助函数：
```go
package main

import (
  "fmt"
  "log"

  "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
  if err != nil {
    log.Fatalf("%s: %s", msg, err)
    panic(fmt.Sprintf("%s: %s", msg, err))
  }
}
```
与生产者的设置一样，我们依然要建立连接和打开信道，然后在声明一个要消费的队列。注意我们需要声明一个与 send 中发布消息匹配的队列：
```go
conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
failOnError(err, "Failed to connect to RabbitMQ")
defer conn.Close()

ch, err := conn.Channel()
failOnError(err, "Failed to open a channel")
defer ch.Close()

q, err := ch.QueueDeclare(
  "hello", // name
  false,   // durable
  false,   // delete when usused
  false,   // exclusive
  false,   // no-wait
  nil,     // arguments
)
failOnError(err, "Failed to declare a queue")
```
注意，我们再一次声明了一个队列。因为消费者有可能比生产者先运行，为了想要确保该队列在我们消费消息前就已经存在，我们必须这样做。

接下来，我们告诉服务器从队列中传递给我们需要消费的消息。鉴于消息将会以异步的方式发送过来，我们在（Go 语言中）一个 goroutine 中的使用管道来接收消息：
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
  }
}()

log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
<-forever
```

这里是完整的 [receive.go](../tutorail-one/receive.go) 文件。

## 作为一个整体
现在我们同时运行两个程序。在一个终端中，运行发布者：
```sh
go run send.go
```
在另一个终端运行消费者：
```sh
go run receive.go
```

消费者将打印发布者通过 RabbitMQ 发送过来的消息。消费者会一直运行等待消息（使用 Ctrl-C 来停止），所以可以尝试从不同的终端来运行发布者。

如果想要查看队列信息，运行命令 `rabbitmqctl list_queues`。

接下来，请移步至 [教程二](../turorial-two/README.md) 创建一个简单的 *工作队列*。