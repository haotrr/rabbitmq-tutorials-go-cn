# Routing

[教程四 原文地址: "Routing"](http://www.rabbitmq.com/tutorial-four-go.html)

在 [教程三](./publish-subscribe.md) 中我们构建了一个简单的日志系统，我们可以将日志信息广播给很多接受者。

本教程中，我们将向这个日志系统增加一些新的功能，使接受者能够只订阅部分的日志信息。比如，我们可以只将错误日志信息保存到文件（磁盘空间），同时将所有的日志信息打印到终端。

## 绑定

上一教程中，我们已经创建了绑定，让我们先来回顾相关代码：

```golang
err = ch.QueueBind(
  q.Name, // queue name
  "",     // routing key
  "logs", // exchange
  false,
  nil)
```

一个绑定是一个交换机和一个队列的相互关联，简单说就是：一个队列对某个交换机上的信息感兴趣。

绑定有一个名为 `routing_key` 的参数，为了避免与 `Channel.Publish` 中的参数相混淆，我们将其成为绑定规则（`binding key`），下面的代码演示了怎样使用绑定规则创建一个绑定。

```golang
err = ch.QueueBind(
  q.Name,    // queue name
  "black",   // routing key
  "logs",    // exchange
  false,
  nil)
```

绑定规则的意义取决于交换机类型。在之前的 fanout 交换机中，我们直接忽略了这个参数值。

## Direct 交换机

在上一篇教程中，我们的日志系统将所有消息广播给所有消费者，现在，我们通过扩展上例来允许我们根据日志的严重程度来筛选日志信息。比如，我们可能想要将日志使写入磁盘的程序只接收严格错误日志，从而不会浪费空间来储存警告或通知级别的日志信息。

我们之前使用的是 fanout 交换机，但其灵活性不高 -- 它只会做无脑的广播操作。

我们将使用 direct 交换机来替代。Direct 交换机背后的路由算法很简单 -- 消息将会进入与其 `routing key` 匹配到的 `binding key` 的队列。

为了演示上述算法，请看下面的图示：

![4-1-direct-exchange](./pics/4-1-direct-exchange.png)

图示中，我们可以看到有两个队列绑定到了 direct 交换机 X 上，第一个队列使用路由规则 orange 绑定；第二个队列则有连个绑定，一个绑定规则为 black，另一个为 green。

图示中，发布到交换机的消息中，带有路由规则 orange 被路由到了队列 Q1，带有路由规则 black 或 green 的则被路由到队列 Q2，其它的消息则会被丢弃。

## 多重绑定

![4-2-direct-exchange-multiple](./pics/4-2-direct-exchange-multiple.png)

使用相同的路由规则绑定多个队列完全是合法的，上面的图示中，我们使用路由规则 black 往 X 上增加了一个与 Q1 的绑定，这样，direct 交换机就像 fanout 交换机一样回报消息广播给所有匹配的队列：一个带有路由规则 black 的消息会被同时投递到 Q1 和 Q2 上。

## 发送日志

我们使用上述的模型来重构日志系统，使用 direct 交换机来替换原先的 fanout 交换机来发送消息。我们始于哦能够日志严重级别来作为路由规则，这样我们的接收程序就可以选择它想要接收的日志。我们先来关注日志的发送。

像之前一样，我们首先需要创建一个交换机：

```golang
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

然后我们就可以发送消息了：

```golang
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

为了简化操作我们假设日志严重级别分为：info、warning 和 error。

## 订阅

接收消息的操作与上一教程中相似，除了一点 -- 我们要为每一个我们在意的日志严重级别创建一个绑定。

```golang
q, err := ch.QueueDeclare(
  "",    // name
  false, // durable
  false, // delete when usused
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
```

## 整体来看

![4-3-overall](./pics/4-3-overall.png)

`emit_log_direct.go` 文件内容：

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

`receive_log_direct.go` 文件内容：

```golang
package main

import (
        "fmt"
        "log"
        "os"

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
                false, // delete when usused
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

如果只想要将 warning 和 error（而不是 info）级别的日志消息保存至文件中，只需要打开终端并键入：

```sh
go run receive_logs_direct.go warning error > logs_from_rabbit.log
```

如果想要在屏幕中显示所有的日志消息，打开终端并输入：

```golang
go run receive_logs_direct.go info warning error
# => [*] Waiting for logs. To exit press CTRL+C
```

如果只想要发送 error 级别的日志，只需要输入：

```golang
go run emit_log_direct.go error "Run. Run. Or it will explode."
# => [x] Sent 'error':'Run. Run. Or it will explode.'
```

完整的源代码参见：[emit_log_direct.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/emit_log_direct.go) 和 [receive_log_direct.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive_log_direct.go)

请移步 [教程五](./topics.md) 探究如何基于模式来监听消息。