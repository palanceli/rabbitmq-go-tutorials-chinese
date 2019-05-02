# 路由
在[前面章节](https://www.rabbitmq.com/tutorials/tutorial-three-go.html)中，我们构建了一个简单的日志系统 能够向许多接收者广播日志消息。

本节将为其添加一个功能——只订阅一部分消息。 例如，只将关键错误消息定向到日志文件（以节省磁盘空间），在控制台上打印所有日志消息。

# 绑定
在前面的例子中，我们已经创建了绑定。 回顾以下代码：
``` go
err = ch.QueueBind(
  q.Name, // queue name
  "",     // routing key
  "logs", // exchange
  false,
  nil)
```
绑定是在交易所和队列之间建立关系。 可以简单理解为：队列对来自此交易所的消息感兴趣。

绑定可以采用额外的`routing_key`参数。 为了避免与`Channel.Publish`参数混淆，我们将其称为`关键字绑定binding key`。 以下代码演示了如何使用关键字创建绑定：
``` go
err = ch.QueueBind(
  q.Name,    // queue name
  "black",   // routing key
  "logs",    // exchange
  false,
  nil)
```
关键字绑定的含义取决于交换类型。 我们之前使用的`fanout`交易所忽略了该值。

# 直接交易所
上一章中的日志记录系统向所有消费者广播所有消息。 我们做一个扩展，根据消息的严重性过滤消息。 例如，将日志写入磁盘的脚本仅接收严重错误，而不在警告或信息日志消息上浪费磁盘空间。

我们使用的是`fanout`交易所，它没有给我们太大的灵活性——它只能进行无脑广播。

接下来我们将使用`direct`交易所。 `direct`交易所背后的路由算法很简单——`binding key`与消息的`routing key`完全匹配时，消息才能进入队列。

为了说明这一点，请参见下图：
![](4Routing/img01.png)
在此设置中，我们可以看到`direct`交易所`X`绑定到两个队列。 第一个队列的绑定关键字为`orange`，第二个队列有两个绑定关键字，一个是`black`，另一个是`green`。

在这样的设置中，使用路由关键字`orange`发布到交易所的消息将被路由到队列Q1。 路由关键字为`black`或`green`的消息将转到Q2。 所有其他消息将被丢弃。

# 多重绑定
![](4Routing/img02.png)
使用相同的绑定关键字绑定多个队列是完全合法的。 在我们的示例中，我们可以在X和Q1之间添加绑定关键字为`black`的绑定。 在这种情况下，`direct`交换机将表现得像`fanout`一样，将消息广播到所有匹配的队列。 路由关键字为`black`的消息将传送到Q1和Q2。

# 发送日志
我们将此模型用于我们的日志系统。 我们会将消息发送给`direct`交换机，而不是`fanout`。 我们将提供日志严重性作为`路由关键字routing key`。 这样接收脚本将能够选择它想要的严重级别。 首先关注发送日志。

和之前一样，需要先创建一个交易所：
``` go
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
准备发送消息：
``` go
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
为简化起见，假设“严重级别”是“info”，“warning”，“error”之一。

# 订阅
消息的接收方将像上一个教程一样，仅有一点不同——我们将为感兴趣的每个严重级别创建一个新的绑定。
``` go
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
# 组装在一起
`emit_log_direct.go`的代码：
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
`receive_logs_direct.go`的代码：
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
如果你只想把“warning”和“error”的日志保存到文件，打开控制台，运行：
``` shell
go run receive_logs_direct.go warning error > logs_from_rabbit.log
```
如果你希望在屏幕上看到所有日志，打开新的终端，运行：
``` shell
go run receive_logs_direct.go info warning error
# => [*] Waiting for logs. To exit press CTRL+C
```
要发送`error`级日志，运行：
``` shell
go run emit_log_direct.go error "Run. Run. Or it will explode."
# => [x] Sent 'error':'Run. Run. Or it will explode.'
```
（全部代码在[emit_log_direct.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/emit_log_direct.go)和[receive_logs_direct.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive_logs_direct.go)）

[第五章](https://www.rabbitmq.com/tutorials/tutorial-five-go.html)将继续介绍怎样通过模式匹配来筛选监听消息