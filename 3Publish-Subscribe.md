# 发布/订阅

> - 先决条件
本教程假定已在本地[安装](https://www.rabbitmq.com/download.html)RabbitMQ且运行在端口`5672`上。 如果您使用不同的主机，端口或凭据，则需要调整连接设置。
> 
> - 哪里可以得到帮助
如果您在阅读本教程时遇到问题，可以通过邮件列表与我们[联系](https://groups.google.com/forum/#!forum/rabbitmq-users)。

[前一章](https://www.rabbitmq.com/tutorials/tutorial-two-go.html)，我们创建了工作队列。 工作队列背后的假设是每个任务都交付给一个工作者。 在本教程中将做一些完全不同的事情——向多个消费者传递信息。 此模式称为“发布/订阅”。

我们构建一个简单的日志记录系统来说明这种模式。 该系统包含两个程序——一个发出日志消息，一个接收和打印它们。

在该日志记录系统中，接收程序的每个运行副本都将获取到消息。 这样我们就可以运行一个接收器把日志定向到磁盘; 运行另一个把日志打印到屏幕。

基本上，发布的日志消息将被广播给所有接收者。

## 交易所
在本教程的前几部分，我们向队列发送消息并从队列接收消息。接下来让我们在Rabbit中引入完整的消息传递模型。

先快速回顾下前面教程中介绍的内容：

- 生产者是发送消息的应用程序。
- 队列是存储消息的缓冲区。
- 消费者是接收消息的应用程序。

RabbitMQ中消息传递模型的核心思想是生产者永远不会将任何消息直接发送到队列。实际上，生产者通常甚至不知道消息是否会被传递到任何队列。

相反，生产者只能向*交易所*发送消息。交易所非常简单。一方面，它接收来自生产者的消息，另一方面将它们推送到队列。交易所必须确切知道如何处理它收到的消息——应该追加到特定队列？还是应该追加到许多队列？或者应该丢弃……其规则由交换类型定义。
![](3Publish-Subscribe/img01.png)

可供选择的交换类型有：`direct`，`topic`，`headers`和`fanout`。 我们将专注于最后一个——`fanout`。 创建一个这种类型的交易所，明明为`logs`：
``` go
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
fanout交易所非常简单。 正如名称所揭示的，它只是将收到的所有消息广播到它知道的所有队列中。 这正是logger所需要的。

> **列出所有交易所**
> 运行命令`rabbitmqctl`可以列出server上运行的所有交易所：
> ``` shell
> sudo rabbitmqctl list_exchanges
> ```
> 此列表中有一些名称为`amq.*`的交易所和默认（未命名）交易所。 这些都是默认创建的，目前还用不到它们。

> **默认交易所**
> 本教程的前几部分中，我们对交易所一无所知，但仍然可以向队列发送消息。 其实我们使用了通过空字符串（`""`）来标识的默认交易所。
> 回顾一下之前我们是怎么发布消息的：
> ``` go
> err = ch.Publish(
>  "",     // exchange
>  q.Name, // routing key
>  false,  // mandatory
>  false,  // immediate
>  amqp.Publishing{
>    ContentType: "text/plain",
>    Body:        []byte(body),
> })
> ```
> `exchange`参数是交易所的名称。 空字符串表示默认或无名交换：消息通过`routing_key`指定的名称路由到队列。

现在我们可以把消息发布到命名的交易所了：
``` go
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
以前我们曾使用过具有特定名称的队列（还记得`hello`和`task_queue`吗？）。 能够命名队列至关重要——我们需要将工作者指向同一个队列。 在生产者和消费者之间共享队列时，要求队列必须有命名。

但本节的logger并非如此。 我们希望了解所有日志消息，而不仅仅是子集。 我们也只对当前流动的消息感兴趣，而不是旧消息。 要解决这个问题，我们需要两件事。

首先，每当连接到Rabbit时，都需要一个新的空队列。 要做到这一点，我们可以创建一个随机名称的队列，或者更好的做法是让服务器为我们选择一个随机队列名称。 

其次，消费者连接一旦断开，队列将自动删除。

在[amqp](http://godoc.org/github.com/streadway/amqp)客户端中，将空字串作为队列名称时，我们创建一个有命名的非持久队列：
``` go
q, err := ch.QueueDeclare(
  "",    // name
  false, // durable
  false, // delete when usused
  true,  // exclusive
  false, // no-wait
  nil,   // arguments
)
```
该方法返回时，该队列实例包含RabbitMQ生成的随机队列名称。 看起来可能类似于：`amq.gen-JzTY20BRgKO-HjmUJj0wLg`。

当声明它的连接断开时，队列将被删除，因为它被声明为`exclusive`。

可以在[队列指南](https://www.rabbitmq.com/queues.html)中了解有关`exclusive`参数和其他队列属性的更多信息。

## 绑定
![](3Publish-Subscribe/img02.png)
我们已经创建了一个fanout交易所和一个队列。 现在我们需要告诉交易所将消息发送到我们的队列。 在交易所和队列之间建立关系称为绑定。
``` go
err = ch.QueueBind(
  q.Name, // queue name
  "",     // routing key
  "logs", // exchange
  false,
  nil,
)
```
从现在开始，`logs`交易所将与我们的队列绑定。

> **列出所有绑定关系**
> 可以通过如下命令列出所有绑定关系：
> ``` shell
> rabbitmqctl list_bindings
> ```

## 组装在一起
![](3Publish-Subscribe/img03.png)
生成日志消息的生产者与之前章节没有什么不同。 最重要的变化是现在要将消息发布到`logs`交易所而不再是无名交易所。 我们需要在发送时提供`routing_key`，但是对于`fanout`交易所，它的值会被忽略。下面是`emit_log.go`的代码：
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
[emit_log.go的源码](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/emit_log.go)

如你所见，在建立连接后，我们声明了交易所。 此步骤是必要的，因为发布到不存在的交易所是不允许的。

如果没有队列绑定到交易所，消息将会丢失，但这对我们没有问题; 如果没有消费者在监听，该消息可以被安全丢弃。

下面是receive_logs.go的代码：
``` go
package main

import (
        "fmt"
        "log"

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
                nil,
        )
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
[receive_log.go的源码](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive_logs.go)
如果你希望吧日至保存到文件，只需要打开控制台，输入命令：
``` shell
go run receive_logs.go > logs_from_rabbit.log
```
如果你希望在屏幕中看到日志，再打开一个终端，并运行：
``` shell
go run receive_logs.go
```
发送日志可运行：
``` shell
go run emit_log.go
```
使用`rabbitmqctl list_bindings`，您可以验证代码是否实际创建了我们想要的绑定和队列。 运行两个`receive_logs.go`程序时，你应该可以看到类似的内容：
``` shell
sudo rabbitmqctl list_bindings
# => Listing bindings ...
# => logs    exchange        amq.gen-JzTY20BRgKO-HjmUJj0wLg  queue           []
# => logs    exchange        amq.gen-vso0PVvyiRIL2WoV3i48Yg  queue           []
# => ...done.
```
输出直接明了：来自交易所`logs`的数据被转到两个命名队列中。 这正是我们想要的。

[第四章](https://www.rabbitmq.com/tutorials/tutorial-four-go.html)我们将继续介绍如何监听消息的子集。