# 远程过程调用（RPC）

在[第二章](https://www.rabbitmq.com/tutorials/tutorial-two-go.html)中，我们学习了如何使用工作队列在多个工作人员之间分配耗时的任务。

但是如果我们需要在远程计算机上运行一个函数并等待结果呢？ 嗯，这就是另一个故事了，这种模式通常称为远程过程调用（RPC）。

在章将使用RabbitMQ构建RPC系统：一个客户端和一个可伸缩的RPC服务器。 我们将创建一个返回Fibonacci数字的RPC服务来模拟耗时任务。

> 关于RPC的说明
> 尽管RPC在计算中是一种非常常见的模式，但它经常受到批评。 当程序员不知道函数调用是本地的还是耗时的RPC时，会出现问题。 这样的混淆导致系统不可预测，并增加了调试的不必要的复杂性。 错误使用RPC可能让代码变成一团乱麻，而不是简化。
> 
> 鉴于以上，请考虑以下建议：
> - 确保明显哪个函数调用是本地的，哪个是远程的。
> - 做好系统的文档化工作。 使组件之间的依赖关系变得清晰。
> - 做好错误处理。 当RPC服务器长时间无响应时，客户端应该如何反应？
>
> 如果不确定RPC带来的好处，请避免使用它。 您应该尽量使用异步管道——而不是容易造成阻塞的RPC，将结果异步推送到下一个计算阶段。

# 回调队列
通常通过RabbitMQ进行RPC很容易。 客户端发送请求消息，服务器回复响应消息。 为了接收响应，我们需要在发送的请求中加入“回调”队列地址。 可以使用默认队列。 如下：
``` go
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
> **消息属性**
> AMQP 0-9-1协议为消息预定义了14个属性。 大多数属性很少使用，但以下情况除外：
> - `persistent`：`true`表示消息为持久，`false`表示瞬态。 你可能还记得[第二章](https://www.rabbitmq.com/tutorials/tutorial-two-go.html)中的这个属性吧。
> - `content_type`：用于描述编码的mime类型。 例如，对于常用的JSON编码，要将此属性设置为：application / json。
> - `reply_to`：通常用于命名回调队列。
> - `correlation_id`：用于将RPC响应与请求相关联。

# 相关ID
在上面介绍的方法中，我们为每个RPC请求创建了一个回调队列。这很低效，更好的方法是为每个客户端创建一个回调队列。

但这会引发新问题，在该队列中收到响应后，不清楚响应属于哪个请求。在使用`correlation_id`属性的时候。我们将为每个请求将其设置为唯一值。之后在回调队列中收到消息时，根据该属性，就能做出与请求相匹配的响应了。当收到未知的`correlation_id`值，可以安全地丢弃该消息，因为不属于我们的请求。

您可能会问，为什么忽略回调队列中的未知消息，而不是失败报错呢？这是由于服务器端存在竞争条件的可能性。尽管概率很低，在向我们发送应答之后，以及确认消息之前，RPC服务器是有可能挂掉的。如果发生这种情况，重新启动的RPC服务器将再次处理请求。这就是为什么在客户端上我们必须优雅地处理重复的响应，理想情况下RPC应该是幂等的。

# 摘要
![](6RPC/img01.png)
我们的RPC将这样工作：

- 当客户端启动时，它会创建一个匿名的独占回调队列。
- 对于RPC请求，客户端发送带有两个属性的消息：`reply_to`，设置为回调队列，`correlation_id`，设置为每个请求的唯一值。
- 请求被发送到`rpc_queue`队列。
- RPC worker（aka：server）等待该队列上的请求。 当有请求时，它会执行作业并使用`reply_to`字段中的队列将结果返回给客户端。
- 客户端等待回调队列上的数据。 当有消息时，它会检查`correlation_id`属性。 如果它与请求中的值匹配，则将响应返回给应用程序。

# 组装在一起
斐波那契函数：
``` go
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
声明斐波那契函数，它假定只有有效的正整数输入。 （不要指望这个适用于大数字，它可能是最慢的递归实现）。

RPC服务端代码[rpc_server.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/rpc_server.go)如下：
``` go
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
服务器代码非常简单：
- 首先建立连接，通道和声明队列。
- 我们可能希望运行多个服务器进程。 为了在多个服务器上平均分配负载，我们需要在通道上设置`prefetch`值。
- 使用`Channel.Consume`来获取我们从队列接收消息的go channel。 然后进入goroutine完成工作并发回响应。

RPC客户端代码[rpc_client.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/rpc_client.go)如下：
``` go
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
接下来可以完整的看一遍我们的示例代码了，[rpc_client.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/rpc_client.go)和[rpc_server.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/rpc_server.go)

启动server：
``` shell
go run rpc_server.go
# => [x] Awaiting RPC requests
```
运行客户端，请求斐波那契数列：
``` shell
go run rpc_client.go 30
# => [x] Requesting fib(30)
```
此处介绍的设计并不是RPC服务唯一的实现方法，但它具有一些重要优势：

- 如果RPC服务器太慢，您可以通过运行另一个服务器来扩展。 可以尝试一下在新控制台中运行第二个`rpc_server.go`。
- 在客户端，RPC只需要发送和接收一条消息。 因此，对于单个RPC请求，RPC客户端只需要一次网络往返。

代码仍然相当简单，我们并没有试图解决更复杂（但重要）的问题，例如：

- 如果没有运行服务器，客户应该如何反应？
- 客户端是否应该为RPC设置某种超时？
- 如果服务器出现故障并引发异常，是否应将其转发给客户端？
- 在处理之前防止无效的传入消息（例如检查边界，类型）。

如果你希望尝试，你会发现[management UI](https://www.rabbitmq.com/management.html)用来观察队列非常有用。