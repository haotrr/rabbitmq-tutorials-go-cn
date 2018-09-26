# Topics

[教程五 原文地址: "Topics"](http://www.rabbitmq.com/tutorial-five-go.html)

在 [教程四](./routing.md) 中我们改善了日志系统，为了伪造出广播功能，我们使用 direct 交换机来替换 fanout 交换机，同时我们获得了选择性接收日志的能力。

尽管使用 direct 交换机提升了我们的系统，但它仍有限制 -- 无法基于多个条件来路由。

在我们的日志系统中，我们可能不仅仅想要基于日志严重级别来，而且同时可以几乎发送日志的来源来订阅。你此前可能了解过同时基于日志严重级别（info/warn/crit..）和设备（auth/cron/kern...）的 Unix 工具 syslog。

这样能够给我们提供更多的灵活性 -- 我们也许只想要监听那些来之 cron 严重的错误日志和来自 kern 的所有日志。

为了实现这样的日志系统，我们需要了解更加复杂的 topic 交换机。

## Topic 交换机

发送至 topic 交换机的消息不是使用任意字符的 `routing_key` -- 而是必须是一串使用点符号（.）分隔的单词。这些单词可以是任何字符串，但通常描述了消息的某种特性，合理的路由规则如：“stock.usd.nyse”，“nyse.vmw”，“quick.orange.rabbit” 等等。可以在路由规则中使用任意多个单词，但每个单词的长度上限为 255 字节。

绑定规则必须与路由规则相同，因为 topic 交换机背后的逻辑和 direct 交换机背后的逻辑一样 -- 发送给交换机的带有特定路由规则的消息会被投递到所有与绑定规则匹配的队列中。然而，对绑定规则有如下两个特殊情况：

+ *（star）可以替代一个单词
+ #（hash）可以替代零个或多个单词

最好通过实例来解释：

![5-1-overall](./pics/5-1-overall.png)

在上面的例子中，我们发送的消息都是用于描述动物的，这些待发送的消息的路由规则都包含三个单词（两个点），路由规则的第一个单词描述速度，第二个描述颜色，第三个描述物种：“<speed>.<colour>.<species>”。

同时创建三个绑定：Q1 使用路由规则 *.orange.* 绑定，Q2 使用路由规则 *.*.rabbit 和 lazy.#。

这些绑定可以概括为：

+ Q1 对所有橘色的动物的消息感兴趣
+ Q2 想要获取所有关于兔子的和所有跑得慢的动物的消息

一个路由规则为 quick.orange.rabbit 的消息会被同时投递到两个队列中，路由规则为 lazy.orange.elephant 的消息也是一样。另一方面，路由规则为 quick.orange.fox 的消息会被投递到第一个队列中，路由规则为 lazy.brown.fox 的消息只会被投递到第二个队列中。路由规则为 lazy.pink.rabbit 的消息只会被投递到第二个队列中一次，尽管它同时匹配两个绑定。路由规则为 quick.brown.fox 不匹配任何绑定，所以会被丢弃。

如果我们打破上面的规则，发送路由规则为有一个或四个单词的诸如 orange 和 quick.orange.male.rabbit 这样的消息会怎样呢？情况是，这些消息不会被任何绑定匹配从而被丢弃。

另外一方面，路由规则为诸如 lazy.orange.male.rabbit 这样的消息尽管有四个单词，它仍然会匹配最后一个绑定从而被投递到第二个队列。

## note

## 整体来看

我们将在日志系统中使用 topic 交换机，并一开始就假设日志消息的路由规则包含两个单词：<facility>.<severity>

代码几乎与上一篇教程一模一样。

`emit_log_topic.go` 文件中的代码如下：

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

`receive_logs_topic.go` 文件中的代码：

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
                false, // delete when usused
                true,  // exclusive
                false, // no-wait
                nil,   // arguments
        )
        failOnError(err, "Failed to declare a queue")

        if len(os.Args) < 2 {
                log.Printf("Usage: %s [binding_key]...", os.Args[0])
                os.Exit(0)
        }
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

接收所有日志：

```sh
go run receive_logs_topic.go "#"
```

从 kern 设备接收所有日志：

```sh
go run receive_logs_topic.go "kern.*"
```

或者只想要接收严重级别（critical）的日志：

```sh
go run receive_logs_topic.go "*.critical"
```

也可以创建多个绑定：

```sh
go run receive_logs_topic.go "kern.*" "*.critical"
```

发送一条如有规则为 kern.critical 的消息：

```sh
go run emit_log_topic.go "kern.critical" "A critical kernel error"
```

请尽情地探究这些程序的玩法。既然代码中并没有对路由规则或绑定规则做任何假设，你可以尝试使用超过两个以上的路由规则参数。

完整的代码文件参见 [emit_log_topic.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/emit_log_topic.go) 和 [receive_logs_topic.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive_logs_topic.go)

下一步，在[教程六](./rpc.md)中我们将探究发送作为远程调用过程的循环消息。