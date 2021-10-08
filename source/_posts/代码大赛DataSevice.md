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

**RabbitMq生产者测试代码：**

```go
package main

import  (
    "fmt"
    "github.com/streadway/amqp"
)

func main() {
    conn, err := amqp.Dial("amqp://admin:admin@localhost:5672/")
    if err != nil {
        fmt.Println("failed to connect rabbitMq")
    } else {
        fmt.Println("success to connect rabbitMq")
    }
    defer conn.close()
    
    ch, err := conn.Channel()
    if err != nil {
        fmt.Println("failed to create channel")
    } else {
        fmt.Println("success to create channel")
    }
    defer ch.Close()
    
    q, err := ch.QueueDeclare(
    	"hello",
        false,
        false
        false,
        false,
        nil,
    )
    if err != nil {
        fmt.Println("failed to create queue")
    } else {
        fmt.Println("success to create queue")
    }
    
    body := "hello world"
    err = ch.Publish(
    	"",
        q.Name,
        false,
        false,
        amqp.Publishing{
            ContentType: "text/plain",
            Body: []byte(body),
        })
    if err != nil {
        fmt.Println("send message failed")
    } else {
        fmt.Println("send message success")
    }
}
```

