# 工作队列

![](2WorkQueues/img01.png)

在第一篇教程中，我们编写了程序来发送和接收来自命名队列的消息。 本节将创建一个工作队列，用于向多个接收端分配耗时任务。

工作队列（又称：任务队列）背后的主要思想是避免立即执行资源密集型任务。该任务的执行将被安排在稍后完成，以避免长时间等待。 我们将任务封装为消息并将其发送到队列。 在后台运行的工作进程将取出任务并最终执行作业。 当运行多个工作程序时，它们之间将共享任务。

这个概念在Web应用程序中特别有用，这类应用中，在一个短HTTP请求窗口期间无法处理复杂任务。


# 准备
在本教程的前一部分中，我们发送了一条包含“Hello World！”的消息。 现在我们将发送代表复杂任务的字符串。 我们没有比如调整图像大小或渲染pdf文件这类的现实任务，可以通过使用`time.Sleep`函数来假装很忙。 我们将字符串中的点数作为其复杂性; 每打一个点表示“工作”了一秒钟。 例如，`Hello ...`表示任务花费了三秒钟。

我们稍微修改前面示例中的*send.go*代码，以允许从命令行发送任意消息。 该程序将任务安排到工作队列，所以将其命名为`new_task.go`：
``` go
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

旧的*receive.go*也需要进行一些更改：它需要为消息体中的每个点伪造一秒钟的工作。 它从队列中弹出消息并执行任务，所以称之为`worker.go`：
``` go
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
注意，我们用假任务模拟执行时间。

运行方法同教程一：
``` shell
# shell 1
go run worker.go
```
``` shell
# shell 2
go run new_task.go
```

# 循环调度
使用任务队列的一个优点是能够轻松地并行工作。 如果工作正在积压，我们可以添加更多工作者，这样就可以轻松扩展。

首先，让尝试同时运行两个`worker.go`。 他们都会从队列中获取消息，但具体是怎样的呢：

打开三个控制台。 两个将运行`worker.go`。这两个控制台就是两个消费者 —— C1和C2。
``` shell
# shell 1
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
```
``` shell
# shell 2
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
```

我们在第三个控制台发布新任务。 启动消费者后，就可以发布一些消息了：
``` shell
 shell 3
go run new_task.go First message.
go run new_task.go Second message..
go run new_task.go Third message...
go run new_task.go Fourth message....
go run new_task.go Fifth message.....
```
交给工作者的内容如下：
``` shell
# shell 1
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'First message.'
# => [x] Received 'Third message...'
# => [x] Received 'Fifth message.....'
```
``` shell
# shell 2
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'Second message..'
# => [x] Received 'Fourth message....'
```

默认情况下，RabbitMQ将按顺序将每条消息发送给下一个消费者。 平均而言，每个消费者将获得相同数量的消息。 这种分发消息的方式称为循环法。 您可以使用三个或更多工作者尝试一下。

# 消息确认
执行任务可能需要几秒钟。你可能想知道如果一个消费者开始一项长任务，但中途挂掉了会发生什么。在当前代码下，一旦RabbitMQ向消费者发送消息，它立即将该消息标记为删除。在这种情况下，如果kill掉一个工作者，不仅它正在处理的消息将丢失，还将丢失已经分发给这个工作者且尚未处理的所有消息。

但我们不想失去任何任务。如果某个工作者挂了，通常会希望将任务交给另一个工作者继续执行。

RabbitMQ通过引入[消息确认](https://www.rabbitmq.com/confirms.html)来确保消息永不丢失。消费者发回ack（nowledgement）告诉RabbitMQ已收到，收到该确认后，RabbitMQ才会删除掉对应的消息。

如果消费者挂了（其通道关闭，连接关闭或TCP连接丢失）而不发送确认，RabbitMQ会认为消息未被处理并将重新排队。 如果此时有其他消费者，则会迅速将消息重新发送给他们。 这样即使工作者偶尔会挂掉，也能确保消息不会丢失了。

没有任何消息超时; 当消费者挂掉时，RabbitMQ将重新发送消息。 即使处理消息需要非常长的时间，也没关系。

在本教程中，通过将“auto-ack”置为`false`来设定消息确认为手动模式。一旦完成任务，将使用`d.Ack（false）`向工作者发送确认（以确认本次任务传递成功）。
``` go
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

Acknowledgement must be sent on the same channel that received the delivery. Attempts to acknowledge using a different channel will result in a channel-level protocol exception. See the doc guide on confirmations to learn more.
该段代码可以确定即使在处理消息时使用CTRL + C kill掉一个工作者，也不会丢失任何内容。 该工作者挂掉后，所有未经确认的消息将被重新传输。

确认必须在收到交付的同一频道上发送。 尝试使用不同的通道进行确认将导致通道级协议异常。 有关确认的文档指南，请参阅[有关确认的文档](https://www.rabbitmq.com/confirms.html)更多信息。

> 被遗忘的确认
忘记发送`ack`是一类常见错误。 这个简单错误，却会引发严重的后果。 当您客户端退出时，消息将被重新传递（看起来可能像随机重新传递），而且因为无法释放未经确认的消息，RabbitMQ会占用越来越多的内存。
> 
> 可以使用`rabbitmqctl`打印`messages_unacknowledged`字段来调试这类错误：
``` shell
sudo rabbitmqctl list_queues name
  messages_ready messages_unacknowledged
```
> Windows下需要去掉sudo：
``` shell
rabbitmqctl.bat list_queues name messages_ready messages_unacknowledged
```

# 消息持久性
我们已经学会了如何确保即使消费者挂掉，任务也不会丢失。 但是如果RabbitMQ服务挂了，任务还是会丢失。

当RabbitMQ退出或崩溃时，它将丢失队列和消息，除非明确告诉它扔掉这些数据。为此我们需要做两件事：将队列和消息都标记为持久。

首先，确保RabbitMQ永远不会丢失队列。 为此，要声明它是持久的：
``` go
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

虽然这段代码本身是正确的，但是当前它还是不能正常工作。 这是因为之前已经定义了一个名为`hello`的非持久队列。 RabbitMQ不允许使用不同的参数定义两个同名队列，这会返回错误。只需修改一下队列的名称即可，例如`task_queue`：
``` go
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
This durable option change needs to be applied to both the producer and consumer code.

At this point we're sure that the task_queue queue won't be lost even if RabbitMQ restarts. Now we need to mark our messages as persistent - by using the amqp.Persistent option amqp.Publishing takes.
需要在生产者和消费者的代码中同时修改该`durable`选项。

这样即使RabbitMQ重新启动，`task_queue`队列也不会丢失。 接下来通过将需要将`amqp.Publishing`的`DeliveryMode`置为`amqp.Persistent`使
消息置为持久。
``` go
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

> 有关消息的持久性还需要注意
将消息标记为持久并不能完全保证消息不丢失。 虽然RabbitMQ会将消息保存到磁盘，依然存在RabbitMQ接收了消息但来不及保存的可能性。 此外，RabbitMQ不会为每条消息执行fsync（2）—— 可能只是保存到缓存而不是真正写入磁盘。 持久性的保证并不强，但对于简单任务队列来说已经足够了。 如果需要更强的保证，可以使用发布者确认。

# 公平分发
您可能已经注意到分发仍然无法完全按照我们的意愿运行。 例如，有两个工作者的情况下，当所有奇数的消息都很重，偶数消息很轻时，一个工作者将很忙，而另一个工作者始终很轻松。 而RabbitMQ对此一无所知，仍然会平均分发消息。

发生这种情况是因为RabbitMQ只是在消息进入队列时调度消息，而不会查看消费者未确认消息的数量。 它只是无脑地向第n个消费者发送第n个消息。
![](2WorkQueues/img02.png)

为了解决这个问题，我们可以将预取计数设置值为1。这会令RabbitMQ每次向工作者发送的消息不多于1。 换句话说，仅在工作者处理完消息并确之后，才向它发送新消息。 如果没有处理完，它会将消息发送给不忙的工作者。
``` go
err = ch.Qos(
  1,     // prefetch count
  0,     // prefetch size
  false, // global
)
failOnError(err, "Failed to set QoS")
```

> 关于队列大小需要注意
如果所有工作者都很忙，队列就会被填满。 您需要捕捉到这一点，并添加更多工作者，或者采取其他策略以避免这种情况的发生。

# 组装在一起
`new_task.go`的代码如下：
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
[(new_task.go)](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/new_task.go)

`worker.go`的代码如下：
``` go
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
[(worker.go)](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/worker.go)

Now we can move on to tutorial 3 and learn how to deliver the same message to many consumers.

使用消息确认和预取计数，可以搭建起持久化的工作队列。即使RabbitMQ重新启动，任务也不会丢失。

有关`amqp.Channel`方法和消息属性的更多信息，您可以浏览[amqp API参考](http://godoc.org/github.com/streadway/amqp)。

接下来可以进入[教程3](https://www.rabbitmq.com/tutorials/tutorial-three-go.html)，继续学习如何向许多消费者传递相同的消息了。
