---
title: RabbitMq
date: 2021-10-12 21:44:06
tags:
---

# RabbitMq

> https://zhuanlan.zhihu.com/p/63700605
>
> https://www.cnblogs.com/zqllove/p/12858659.html
>
> http://tryrabbitmq.com/



一个**消息队列**

**消息队列**的使用场景：生产者-消费者

**消息队列**的好处：

1.能够解耦

2.能够提速

3.能够广播，可以有多个消费者来使用生产的数据

4.能够削峰

&ensp;

**消息队列**的坏处：

1.变得复杂

2.暂时的不一致性

&ensp;

使用**消息队列**的场景需要满足什么条件：

1.生产者不需要从消费者得到反馈

2.容许短暂的不一致性

&ensp;

**AMQP**:一个**消息队列协议**

rabbitMq就是实现了AMQP的消息队列服务



rabbitMq原理：

![img](https://pic3.zhimg.com/80/v2-cf2ff62088efcca10d15162142015e82_720w.jpg)

+ Producer：生产者
+ Exchange：**消息交换器**，用来指定消息路由规则，也就是 Routing Key 和 Exchange Type 及 Binding Key 来匹配消息并且分发到 Queue 中
+ Binding：？
+ Queue：队列，用于存储消息，消息最终在这里被消费者取走，一个 Message 可以同时拷贝到多个 Queue 中
+ Connection & Channel：RabbitMq 对外提供的 API 中最基本的对象，Connection 代表 TCP 连接，Channel 建立在 TCP 连接中，Channel 是我们与RabbitMq 打交道的最重要的一个接口，大部分业务操作实在 Channel 接口中完成的
+ Consumer：消费者
+ Broker：RabbitMq 的 Server
+ Virtual Host：多个用户使用同一个 RabbitMq Server 提供的服务时，可以划分出多个 VirtualHost，每个用户在自己的 Virtual Host 中创建 Exchange，Queue

&ensp;

**路由转发**：

![img](https://pic3.zhimg.com/80/v2-ae6966318989e41001ef11d9f3724f46_720w.jpg)

Producer生产完消息，发送消息时，需要指定一个Routing Key和Exchange，Exchange收到消息后先查看消息中指定的Routing Key，再根据当前Exchange的Exchange Type，按一定的规则将消息发送到Queue中。

&ensp;

**Exchange Type**

+ Direct:直接路由转发，当Routing Key与Binding Key相匹配时
+ Fanout:复制分发路由，拷贝多份转发给与自己绑定的消息队列，不需要Routing Key匹配
+ Topic:是Direct的通配符模式



**RabbitMq 模拟器：**

http://tryrabbitmq.com/

**Exchange Type为Direct时：**

下图如果Message指定的Routing Key为key1，那么消息将会被分发到Queue1中，最终被Consumer1消费；如果Message指定的Routing Key为key2，那么消息将会被分发到Queue2中，最终被Consumer2消费。

![image-20211008142047042](/home/ts/.config/Typora/typora-user-images/image-20211008142047042.png)

**Exchange Type为Fanout时：**

不需要Routing Key，消息将拷贝多份分发到所有消息队列中。

**Exchange Type为Topic时：**

如下图，生产者的Message中Routing Key为"key.w"，与Exchange绑定的两条Binding Key分别为"key. *"， "key. *"，此时消息将发送到 Queue1 和 Queue2 两个队列中。Server 提供的服务时，可以划分出多个Virtual Host，每个用户在自

![image-20211008142022647](/home/ts/.config/Typora/typora-user-images/image-20211008142022647.png)

&ensp;

## 安装

1.安装erlang

```bash
sudo apt-get install erlang-nox
```

2.安装RabbitMq

```bash
sudo apt-get update
sudo apt-get install rabbitmq-server
```

RabbitMq命令：

```bash
sudo rabbitmq-serer start # 启动
sudo rabbitmq-server stop # 停止
sudo rabbitmq-server restart # 重启
sudo rabbitmq-server status # 查看当前状态
```

添加admin并赋予administrator权限：

```bash
sudo rabbitmqctl add_user  admin  admin   # 添加admin用户，密码设置为admin
sudo rabbitmqctl set_user_tags admin administrator # 赋予权限
sudo rabbitmqctl  set_permissions -p / admin '.*' '.*' '.*' # 赋予virtual host中所有资源的配置、写、读权限以便管理其中的资源
```

**RabbitMQ GUID使用**

```bash
 sudo  rabbitmq-plugins enable rabbitmq_management
```

浏览器访问 localhost:15672

---



## 测试代码：

**生产者**测试代码：

```go
package main

import (
    "fmt"
    "github.com/streadway/amqp"
)

func handleError(err error, msg string) {
    if err != nil {
        fmt.Println(msg + " failed!")
    } else {
        fmt.Println(msg + " success!")
    }
}


func main() {
    conn, err := amqp.Dial("amqp://admin:admin@sunseason.xyz:5672/")
    handleError(err, "connect rabbitMq")
    defer conn.Close()

    ch, err := conn.Channel()
    handleError(err, "create channel")
    defer ch.Close()

    q, err := ch.QueueDeclare(
        "hello",
        false,
        false,
        false,
        false,
        nil,
    )
    handleError(err, "create queue")

    body := "hello world"
    err = ch.Publish(
        "",
        q.Name,
        false,
        false,
        amqp.Publishing{
            ContentType: "text/plain",
            Body: []byte(body),   // 这里必须转换成[]byte 类型，否则传输会报错 cannot use body (type string) as type []byte in field value，后续接收者也需将接收到的数据转换成string，否则显示的是ASCII码的数组。
        })
    handleError(err, "send message")
}

```

&ensp;

**消费者**测试代码：

```go
package main

import (
    "fmt"
    "github.com/streadway/amqp"
)

func handleError(err error, msg string) {
    if err != nil {
        fmt.Println(msg + " failed!")
    } else {
        fmt.Println(msg + " success!")
    }
}


func main() {
    conn, err := amqp.Dial("amqp://admin:admin@sunseason.xyz:5672/")
    handleError(err, "connect rabbitMq")
    defer conn.Close()

    ch, err := conn.Channel()
    handleError(err, "create channel")
    defer ch.Close()

    q, err := ch.QueueDeclare(
        "hello",
        false,
        false,
        false,
        false,
        nil,
    )
    handleError(err, "create queue")

    msgs, err := ch.Consume(
        q.Name,
        "",
        true,
        false,
        false,
        false,
        nil,
    )
    handleError(err, "register a consumer")

    forever := make(chan bool)
    // 开启了一个匿名func，使用go关键字，通过协程并发执行
    go func() {
        for d := range msgs {
            fmt.Println("Received a message: %s", string(d.Body)) // 这里记住将d.Body由ASCII码数组转换为string类型
        }
    }()

    <-forever // forever管道中取出数据，由于forever管道中没有数据，所以阻塞
}
```



**Cantor <--> Data Server**

1.要交换的数据结构，Cantor做消费者时需要的数据，Data Server做消费者时需要的数据 (无非就是确定这问题吧？)

2.
