---
title: 代码大赛DataService
date: 2021-10-08 20:24:11
tags:
---

# 代码大赛 Data Service



**服务器地址**：

**redis:**

域名： sunseason.xyz

端口： 50001

密码： 123456yun

**MySQL:**

域名： sunseason.xyz

端口： 3306

账号： admin

密码： admin

数据库名： data_server

---

**Go连接mysql, redis, rabbitMq测试代码：**

```go
package main

import (
    "database/sql"
    "fmt"
    _ "github.com/go-sql-driver/mysql"
    "github.com/garyburd/redigo/redis"
    "github.com/streadway/amqp"
)

func main() {

    db,_:=sql.Open("mysql","admin:admin@(sunseason.xyz:3306)/data_server")
    err :=db.Ping()
    if err != nil{
        fmt.Println("failed to connect mysql!!!")
    } else {
        fmt.Println("success to connect mysql!!!")
    }
    defer db.Close()

    c, err := redis.Dial("tcp", "sunseason.xyz:50001")
    if err != nil {
        fmt.Println("failed to connect redis!!!")
    } else {
        fmt.Println("success to connect redis!!!")
    }
    defer c.Close()
    
    //这里测试的是本地rabbitMq服务器，默认端口号为5672
    conn, err := amqp.Dial("amqp://admin:admin@localhost:5672/")
    if err != nil {
        fmt.Println("failed to connect rabbitMq!!!")
    } else {
        fmt.Println("success to connect rabbitMq!!!")
    }
}
```

---

**本地连接mysql, redis命令：**

```bash
#mysql
mysql -uadmin -padmin -h  sunseason.xyz -P 3306

#redis
redis-cli -h sunseason.xyz -p 50001 -a 123456yun
```

---

&ensp;

**查看端口占用情况：**

```bash
netstat anp | grep PORT
```



---

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

