> - 先决条件
本教程假定已在本地[安装](https://www.rabbitmq.com/download.html)RabbitMQ且运行在端口`5672`上。 如果您使用不同的主机，端口或凭据，则需要调整连接设置。
> 
> - 哪里可以得到帮助
如果您在阅读本教程时遇到问题，可以通过邮件列表与我们[联系](https://groups.google.com/forum/#!forum/rabbitmq-users)。

# 介绍

RabbitMQ, and messaging in general, uses some jargon.

RabbitMQ是一个消息代理：它接受和转发消息。 可将其视为一个邮局：当您将要发布的邮件放在邮箱中时，邮递员会确保将邮件发送给您的收件人。 在这个类比中，RabbitMQ就是邮箱、邮局和邮递员。

RabbitMQ和邮局之间的主要区别在于它不处理纸张，而是接收、存储和转发二进制的消息数据。

RabbitMQ和消息传递通常会使用如下术语：

- `生产者`表示发送方，是发送消息的程序，用下图表示：  
![](1HelloWorld/img01.png)

- `队列`是在RabbitMQ中的邮箱角色。虽然消息流经RabbitMQ和您的应用程序，但它们只能存储在队列中。 队列只受主机内存和磁盘的限制，它本质上是一个大的消息缓冲区。 生产者可以将消息发送到队列，消费者可以从队列接收数据。用下图表示：  
![](1HelloWorld/img02.png)

- `消费者`是等待和接收消息的程序。用下图表示：  
![](1HelloWorld/img03.png)

注意：生产者、消费者和消息代理不一定在同一台主机上；大部分应用场景下，他们确实不在同一台主机。一个应用程序可以既是生产者，又是消费者。

# Hello World!
**(使用Go RabbitMQ 客户端)**

In the diagram below, "P" is our producer and "C" is our consumer. The box in the middle is a queue - a message buffer that RabbitMQ keeps on behalf of the consumer.

本章将使用Go编写两个小程序;：生产者发送单个消息，消费者接收并打印消息。 我们暂时先忽略[Go RabbitMQ](http://godoc.org/github.com/streadway/amqp) API中的一些细节，专注于如何启动消息传递的“Hello World”。

在下图中，“P”是生产者，“C”消费者。 中间的框是消息队列——一个消息缓冲区。  
![](1HelloWorld/img04.png)

> **Go RabbitMQ客户端库**
RabbitMQ支持多种协议。本教程使用AMQP 0-9-1，它是一种开放、通用的消息传递协议。 RabbitMQ有许多[不同语言的客户端](http://rabbitmq.com/devtools.html)。 此处我们将使用Go amqp。
> 
> 首先，使用go get安装amqp：
  ``` go
  go get github.com/streadway/amqp
  ```

安装完成amqp之后就可以开始写代码了。

## 发送端

![](1HelloWorld/img05.png)

接下来我们在`send.go`中发送消息，在`receive.go`中接收消息。消息发布方将连接到RabbitMQ，发送单条消息后退出。

需要先在`send.go`中导入库：
``` go
package main

import (
  "fmt"
  "log"

  "github.com/streadway/amqp"
)
```
添加一个helper函数来检查每个amqp函数调用的返回值：
``` go
func failOnError(err error, msg string) {
  if err != nil {
    log.Fatalf("%s: %s", msg, err)
  }
}
```
连接RabbitMQ服务器：
``` go
conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
failOnError(err, "Failed to connect to RabbitMQ")
defer conn.Close()
```

这一句封装了socket连接，我们只需要关注协议版本和身份验证等。 接下来，创建一个通道，这也是使用RabbitMQ的必要套路：
``` go
ch, err := conn.Channel()
failOnError(err, "Failed to open a channel")
defer ch.Close()
```

为了将消息发送出去，还需要声明一个队列用来承载即将发布的消息：
``` go
q, err := ch.QueueDeclare(
  "hello", // name
  false,   // durable
  false,   // delete when unused
  false,   // exclusive
  false,   // no-wait
  nil,     // arguments
)
failOnError(err, "Failed to declare a queue")

body := "Hello World!"
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
声明队列是个幂等操作（幂等操作的特点是：任意多次执行所产生的影响均与一次执行的影响相同）
[这里是send.go的完整代码。](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/send.go)

> **无法发送**
如果这是您第一次使用RabbitMQ并且没有看到“Sent”消息，您可能会为找不到错误原因而挠头。 比如磁盘空间不够（默认至少需要200 MB空闲）会导致拒绝接收消息。 检查日志文件会有助于定位错误，必要时可以据此减少限制。 [配置文件文档](http://www.rabbitmq.com/configure.html#config-items)描述了如何设置`disk_free_limit`。

## 接收端

以上是消息的生产者。 消费者将侦听来自RabbitMQ的消息，与发布单个消息的发布者不同，消费者将持续运行，以侦听和打印消息。  

![](1HelloWorld/img06.png)

`receive.go`代码同样需要import和helper函数：
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
```
设置部分与发送端相同：打开一个连接和一个通道，并声明将要使用的队列。 请注意，要与发送端的队列匹配。
``` go
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

请注意，此处也声明了队列。 因为我们可能在生产者之前启动消费者，所以我们希望在尝试使用消息之前确保队列存在。

我们即将告诉服务器从队列中传递消息。 因为它会异步地向我们发送消息，所以我们将在goroutine中读取来自通道（由`amqp :: Consume`返回）的消息。

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
  }
}()

log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
<-forever
```
[这里是receive.go的代码](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive.go)

## 组合在一起
接下来就可以跑两个端的代码了，在终端执行发送方：
``` shell
go run send.go
```
执行接收方：
``` shell
go run receive.go
```

接收方将打印由发送方通过RabbitMQ发布的消息。消费方将持续运行以等待消息到来（使用Ctrl-C终止运行），接下来可以在另一终端启动发送方。

使用`rabbitmqctl list_queues`命令查看队列内容。

接下来可以异步part 2来构造一个简单的工作队列了。