# 远程过程调用（RPC）

[教程六 原文地址: "Remote procedure call（RPC）"](http://www.rabbitmq.com/tutorial-six-go.html)

在 [教程二](./work-queues.md) 中我们已经掌握了如何使用工作队列在多个工作者中分发实时消费的任务。

但是如果我们需要运行一个在远程机器上的函数然后等待它的运行结果呢？这就需要不同的技术了，这种模式通常被称作远程过程调用（Remote Procedure Call）或者 RPC。

在本教程中，我们将使用 RabbitMQ 来构建一个 RPC 系统：包含一个客户端和一个可扩展的 RPC 服务器。目前我们手头上没有值得分发的实时消费的任务，所以我们创建一个伪造用于返回斐波纳契数字（Fibonacci Number）的 RPC 服务。

### 回调队列

一般来说通过 RabbitMQ 来实现 RPC 是很简单的，客户端发送一个请求消息然后服务器返回一个响应消息。为了接收响应消息，我们需要在请求中增加一个回调（callback）队列地址。可以使用默认队列，我们一起来试一试：

```golang
q, err := ch.QueueDeclare(
  "",    // name
  false, // durable
  false, // delete when usused
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
    ReplyTo:       q.Name,
    Body:          []byte(strconv.Itoa(n)),
})
```

### Message properties

### 关联 ID

在上面的方法中，我们对每一个 RPC 请求都创建了一个回调队列，这样做效率相对低下，幸好我们还有更好的方式 -- 对每一个客户只创建一个回调队列。

这样会造成一个新的问题，队列对接收到的响应消息属于哪一个请求并不清楚。这时就该轮到 correlation_id 派上用场了。对每个请求，我们都将关联 ID 设为唯一值，然后当我们从回调队列中接收一个消息时，我们将使用到这个属性，通过它，我们可以将响应消息匹配的请求。对于那些未知的关联 ID 值，我们只需简单地丢弃消息 -- 它们不属于我们发送的请求。

你也许会问，我们为什么应该忽略回调队列中那些未知的消息而不是抛出一个错误？这是因为在服务端有可能出现竟态条件（race condition）。尽管不太常见，RPC 服务在发送一个响应消息之后但在发送一个确认消息给求情之前有可能会死掉，如果这种情况发生，重启 RPC 服务会重新处理请求，这就是为什么我们序啊哟优雅地处理重复响应的原因，RPC 应该是完全幂等的。

### 总结

![6-1-overall](./pics/6-1-overall.png)

我们的 RPC 看起来应该是这样子的：
    客户机一旦启动，他就创建一个唯一匿名的回调队列。
    对每一个 RPC 请求，客户端发送的消息都包含这两个属性：reply_to 用于设置回调队列；correlation_id 用于对每一个请求设置唯一值。
    请求被发送到 rpc_queue 队列中。
    RPC 工作者（服务器）从 rpc_queue 队列中等待请求，请求一旦出现，它便处理响应的工作并通过 reply_to 字段提供的队列返回一条包含结果的消息给客户端。
    客户端通过回调队列等待接收数据。一旦响应消息出现，它便检查其 correlation_id 属性，如果匹配请求上的值便返回响应消息给对应的应用。

## 整合来看

斐波纳契函数：

```golang
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

我们声明了一个斐波纳契函数，并假设正整数为合法输入。（不要指望它对大的整数有效，它有可能是最慢的递归实现了）。

RPC 服务器的代码如下：

```golang
package main

import (
        "fmt"
        "log"
        "strconv"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
                panic(fmt.Sprintf("%s: %s", msg, err))
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
                false,       // delete when usused
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

服务端代码相当明了：
    像之前那样我们从建立连接，创建信道和声明队列开始。
    我们也许想要运行多个服务进程，为了均衡每个服务器的负载，我们需要在信道上设置 prefetch 属性。
    我们使用 Channel.Consume 来获取 Go 中从队列中接收消息的信道。然后我们进入处理具体工作和返回响应的 gorountine。

RPC 客户端的代码如下：

```golang
package main

import (
        "fmt"
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
                panic(fmt.Sprintf("%s: %s", msg, err))
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
                false, // delete when usused
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

现在是时候好好看一看整个例子的源代码了： [rpc_server.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/rpc_server.go) 和 [rpc_client.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/rpc_client.go) 。

RPC 服务现在已经就位，我们可以开始运行：

```sh
go run rpc_server.go
# => [x] Awaiting RPC requests
```

请求一个斐波纳契数字，运行客户端：

```sh
go run rpc_client.go 30
# => [x] Requesting fib(30)
```

上面展示的程序不是 RPC 服务的唯一实现，但其有几个很重要的优势：

+ 如果 RPC 服务器太慢，可以通过增加另外一个服务实例来扩展，请尝试在新的终端运行第二个 rpc_server.go。
+ 对于客户端，RPC 只需发送和接收一个消息，最终，RPC 客户端对每个 RPC 请求只需要一个网络循环。

我们的代码仍然相当简单，但是不要试图用它来解决更加复杂（但很重要）的问题，比如：

+ 如果没有服务程序在运行，客户端该怎样反应？
+ RPC 客户端是否需要某种超时机制？
+ 如果服务器失灵并且抛出异常，应该返回给客户端吗？
+ 在处理接消息之前对其进行保护（如检查边界，类型等）以免处理了非法的信息。

> 如果想要更加深入地探究，你会发现 [management UI](https://www.rabbitmq.com/management.html) 对查看队列非常有用。