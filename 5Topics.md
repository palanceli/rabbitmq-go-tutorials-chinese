# 主题
[上一章](https://www.rabbitmq.com/tutorials/tutorial-four-go.html)，我们改进了日志系统。 使用`direct`交易所，而不是只会无脑广播的`fanout`，有选择性地接收日志。

`direct`交易所虽然对我们的系统有所改进，但仍有局限性——它不能基于多个条件进行路由。

在我们的日志记录系统中，可能不仅要根据严重性，还要根据发出源来订阅日志。 可以从`syslog` 工具中了解这个概念，该工具可以根据严重性（info / warn / crit ...）和facility（auth / cron / kern ...）来路由日志。

这能带来很大的灵活性——我们可能想要监听来自'cron'的关键错误以及来自'kern'的所有日志。

要在日志记录系统中实现这一点，我们需要了解更复杂的`topic`交易所。

# 主题交易所
发送到`topic`交易所的消息不能具有任意的`routing_key`——它必须是由点分隔的单词列表。单词可以是任何内容，但通常它们指定与消息相关的一些特征。一些有效的路由关键字如：“`stock.usd.nyse`”，“`nyse.vmw`”，“`quick.orange.rabbit`”。routing key可以包含最多255字节的单词。

binding key也必须采用相同的形式。`topic`exchange背后的逻辑类似于`direct` exchange——使用特定routing key发送的消息将被传递到与之匹配的binding key绑定的所有队列。但是binding key有两个重要的特例：
- *（星号）可以替代一个单词。
- ＃（hash）可以替换零个或多个单词。

用下面的例子来解释：
![](5Topics/img01.png)
在这个例子中，我们将发送描述动物的消息。消息将与包含（用两个点分隔的）三个单词的routing key一起发送。routing key中的第一个单词描述速度，第二个是颜色，第三个是物种：“`<speed>.<color>.<species>`”。

我们创建了三个绑定：Q1绑定了binding key“`*.orange.*`”，Q2绑定了“`*.*.rabbit`”和“`lazy.＃`”。

这些绑定可以概括为：

- Q1对所有橙色动物感兴趣。
- Q2希望听到关于兔子的一切，以及关于懒惰动物的一切。

routing key设置为“`quick.orange.rabbit`”的消息将传递到两个队列。“`lazy.orange.elephant`”也将同时发送给他们。
“`quick.orange.fox`”只会转到Q1，“`lazy.brown.fox`”只会转到Q2。 
“`lazy.pink.rabbit`”将仅传递到Q2一次，即使它匹配两个绑定。 
“`quick.brown.fox`”与任何绑定都不匹配，因此它将被丢弃。

如果我们违反协议，发送带有一个或四个单词的消息，例如“`orange`”或“`quick.orange.male.rabbit`”，会发生什么？这些消息将不匹配任何绑定，因此将丢弃。

“`lazy.orange.male.rabbit`”，虽然有四个单词，但能匹配到最后一个绑定，因此将被传递到Q2。

> topic exchange
> topic exchange功能强大，可以像其他exchange一样运行。
> 当队列绑定到“`＃`”时 - 它将接收所有消息，而不管路由密钥 - 如`fanout`exchange。
> 如果绑定中没有使用特殊字符“`*`”和“`＃`”，`topic` exchange的行为就像`direct` exchange一样。

# 组装到一起
我们将在日志记录系统中使用主题交换。 首先假设日志的routing key有两个单词：“`<facility>.<severity>`”。

下面的代码与[上一章](https://www.rabbitmq.com/tutorials/tutorial-four-go.html)几乎一模一样。

`emit_log_topic.go`的代码：
``` go
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
``` go
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
接收所有日志，运行：
``` shell
go run receive_logs_topic.go "#"
```
接收所有源自“`kern`”的日志，运行：
``` shell
go run receive_logs_topic.go "kern.*"
```
只接收`critical`的日志，运行：
``` shell
go run receive_logs_topic.go "*.critical"
```
创建多重绑定，运行：
``` shell
go run receive_logs_topic.go "kern.*" "*.critical"
```
使用routing key“`kern.critical`”发送日志：
``` shell
go run emit_log_topic.go "kern.critical" "A critical kernel error"
```
玩这些程序很有意思。 请注意，代码不会对路由或绑定关键字做出任何假设，您可能希望使用两个以上的路由关键字做参数。

（[emit_log_topic.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/emit_log_topic.go)和[receive_logs_topic.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive_logs_topic.go)的完整源代码）

[第六章](https://www.rabbitmq.com/tutorials/tutorial-six-go.html)讲解如何将往返消息作为远程过程调用。