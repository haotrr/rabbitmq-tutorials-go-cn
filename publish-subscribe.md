# Publish/Subscribe

[教程三 原文地址: "Publish/Subscribe"](http://www.rabbitmq.com/tutorial-three-go.html)

在[上一教程](./worker-queue.md)中我们创建了工作队列，在工作队列中，每个任务假设被分发给唯一的工作者，本教程将处理完全不同的情况 -- 将一个消息同时分发给不同的消费者，这种模式被成为“发布/订阅”模式。

为了演示“发布/订阅”模式，我们将创建一个简单的日志系统，包含两个程序 -- 一个发送日志消息，一个用于接收日志消息并将之打印。

在我们的日志系统中，每一份接收程序的拷贝都会待应接收到的消息，这样我们就能够在运行一个用于接收并将日志写入磁盘的程序的同时，我们还可以运行一个用于接收并将日志打印到屏幕的程序。

本质上，发布日志消息就是将其广播给所有接受者。

## 交换机

在系列教程的前面部分，我们使用队列来发送和接收消息，现在，我们将介绍 RabbitMQ 中完整的消息模型。

先让我们快速地复习一下之前学习过的内容：

+ 一个生产者是一个用于发送消息的应用程序。
+ 一个队列是一个用于存储消息的缓存。
+ 一个消费者是一个用于接收消息的应用程序。

RabbitMQ 中最核心的思想是：一个生成者永远不会向一个队列直接发送消息。实际上，一个消费者根本不需要知道一个消息是否将会投递到一个队列中。

相反，一个消费者只会将消息发送到交换机（exchange）上。交换机非常简单，一方面它从生成者那里接收消息，另一方面它将消息推送给队列。交换机必须明确地知道它将接收怎样的消息，消息将会被附加到特定的队列？消息将会被附加到很多队列？或者消息会被丢弃？这些规则将通过交换机类型（exchange type）定义。

![3-1-exchanges](3-1-exchanges.png)

RabbitMQ 提供了四种交换机：direct，topic，headers 和 fanout。我们主要关注最后一种 -- fanout。下面，让我们创建一个 fanout 类型的交换机，并命名为 `logs`：

```golang
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

fanout 类型的交换机非常简单，从名字就可推断出来，它将所有接收到的消息广播到它所知的所有队列上，这恰好就是我们日志系统所需要的。

> Note:
> 
> **列出所有交换机**
> 
> 列出服务器上的交换机可以使用命令 `rabbitmqctl`:
> ```sh
> sudo rabbitmqctl list_exchangs
> ```
> 

现在，我们可以将消息发布到我们命名的交换机上：

```golang
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

body := bodyFrom(os.Args)
err = ch.Publish(
  "logs", // exchange
  "",     // routing key
  false,  // mandatory
  false,  // immediate
  amqp.Publishing{
          ContentType: "text/plain",
          Body:        []byte(body),
  })
```

## 临时队列

你应该还记得我们之前使用的特定名字的队列（还记得 `hello` 和 `task_queue` 吗？），能够对队列进行命名相当重要 -- 我们需要将不同的工作者指向相同的队列。当在不同的生产者和消费者中共享同一个队列时给一个队列命名非常重要。

但对于我们的日志系统却并非如此，我们想要监控所有的日志消息，而不只是其中的部分，而且我们只对当前消息而不是旧的消息感兴趣，为了解决这个问题，我们需要做两件事情。

首先，每次我们连接到 RabbitMQ，我们都需要一个全新的、空的队列，我们可以创建一个随机命名的队列来实现，或者使用更好的方式 -- 让服务器来随机地为我们分配一个队列名字。

第二，一旦我们断开连接，队列自动删除。

在 [amqp](http://godoc.org/github.com/streadway/amqp) 客户端中，当我们将队列名字设为一个空字符串时，我们就创建了一个随机命名的非持久化队列。

```golang
q, err := ch.QueueDeclare(
  "",    // name
  false, // durable
  false, // delete when unused
  true,  // exclusive
  false, // no-wait
  nil,   // arguments
)
```

`QueueDeclare` 方法返回时，RabbitMQ 生成了一个随机命名的队列实例，随机的名字与 `amq.gen-JzTY20BRgKO-HjmUJj0wLg` 类似。

当连接断开时，队列会被删除，因为它被声明为独有的（exclusive）。

可以访问 [guide on queues](https://www.rabbitmq.com/queues.html) 来了解更多关于 `exclusive` 和其它的队列属性。

## 绑定

![3-2-bindings](./pics/3-2-bindings.png)

此前，我们已经创建了一个 fanout 类型的交换机和队列了，现在我们需要告诉交换机将消息发送给队列。交换机和队列的关系叫做绑定（binding）。

```golang
err = ch.QueueBind(
  q.Name, // queue name
  "",     // routing key
  "logs", // exchange
  false,
  nil
)
```

现在我们的 `log` 交换机将会把消息附加到我们声明的队列中。

## 整体运行

![3-3-overall](./pics/3-3-overall.png)

用于发送日志的生产者程序与教程二中的程序并没有太大区别，最主要的改变就是我们将消费发布到了我们自己命名的 `logs` 交换机而不是未命名的交换机。我们本需要在发送时设置一个 `routingKey`，但 `fanout` 类型的交换机会忽略它。下面时完整的 `emit_log.go` 文件内容：

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

        body := bodyFrom(os.Args)
        err = ch.Publish(
                "logs", // exchange
                "",     // routing key
                false,  // mandatory
                false,  // immediate
                amqp.Publishing{
                        ContentType: "text/plain",
                        Body:        []byte(body),
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

[emit_log.og](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/emit_log.go)

建立连接之后，我们声明了一个交换机，这一步是必须的，如果向一个不存在的交换机发布消息是被禁止的。

如果没有给交换机绑定队列，消息将会丢失，但对我们来说是 OK 的，如果没有消费者监听，我们可以简单地将消息丢弃。

`receive_logs.go` 的代码如下：

```golang
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
                false, // delete when usused
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
                nil)
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

[receive_logs.go](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive_logs.go)

如果想要将日志写入文件，运行以下命令：

```sh
go run receive_logs.go > logs_from_rabbit.log
```

想要在屏幕上查看日志，打开新的终端并运行：

```sh
go run receive_logs.go
```

当然，发送日志需要运行：

```sh
go run emit_log.go
```

通过运行 `rabbitmqctl list_bindings` 可以验证我们的代码确实创建了我们想要绑定和队列。通过运行两个 `receive_logs.go` 可以看到类似如下的输出内容：

```sh
sudo rabbitmqctl list_bindings
# => Listing bindings ...
# => logs    exchange        amq.gen-JzTY20BRgKO-HjmUJj0wLg  queue           []
# => logs    exchange        amq.gen-vso0PVvyiRIL2WoV3i48Yg  queue           []
# => ...done.
```

上述的打印信息非常明显：数据从 `logs` 交换机流向了有服务器命名的两个队列，这正是我们想要的。

探究如何监听部分消息，请移步至 [教程四](./routing.md)。