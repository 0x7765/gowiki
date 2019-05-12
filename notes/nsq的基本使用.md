# 本地安装nsq

```shell
brew install nsq
brew services start nsq
```

# producer

```go
package main

import (
	"bufio"
	"fmt"
	"github.com/nsqio/go-nsq"
	"os"
)

var (
	Producer *nsq.Producer
	host     = "localhost:4150"
)

func init() {
	producer, err := nsq.NewProducer(host, nsq.NewConfig())
	if err != nil {
		fmt.Println(err)
		os.Exit(-1)
	}
	Producer = producer
}

// 同步的方式发送
func Send(topic string, message []byte) (err error) {
	err = Producer.Publish(topic, message)
	return
}

func SendAsyn(topic string, message []byte, doneChan chan *nsq.ProducerTransaction) (err error) {
	err = Producer.PublishAsync(topic, message, doneChan)
	return
}

func main() {
	running := true

	reader := bufio.NewReader(os.Stdin)

	for running {
		line, _, _ := reader.ReadLine()
		if string(line) == "stop" || string(line) == "quit" {
			running = false
		}
		Send("hello", []byte(line))
	}
}
```

# consumer

**消费者需要实现handler接口**

```go
package main

import (
	"fmt"
	"github.com/nsqio/go-nsq"
	"os"
	"time"
)

var (
	Consumer *nsq.Consumer
)

type MessageHandler struct {
}

func (*MessageHandler) HandleMessage(message *nsq.Message) error {
	fmt.Println("reveiced :", message.NSQDAddress, "message :", string(message.Body))
	return nil
}

func init() {
	nsqConsumer, err := nsq.NewConsumer("hello", "hello-test", nsq.NewConfig())
	if err != nil {
		os.Exit(-1)
	}
	Consumer = nsqConsumer
	Consumer.AddHandler(&MessageHandler{})
	nsqConsumer.ConnectToNSQD("localhost:4150")
}

func main() {
	time.Sleep(time.Minute)
}
```
