## Prerequisites

### Install `librocketmq`
rocketmq-client-go is a lightweight wrapper around [rocketmq-client-cpp](https://github.com/apache/rocketmq-client-cpp), so you need install 
`librocketmq` first.

#### include file
```bash
git clone https://github.com/apache/rocketmq-client-cpp

# By default, CFLAGS contains `/usr/local/include`
sudo mkdir -p /usr/local/include/rocketmq/
sudo cp rocketmq-client-cpp/include/* /usr/local/include/rocketmq
```

#### binary library
your could download directly or build manually:

- download specific release: [rocketmq-client-cpp](https://archive.apache.org/dist/rocketmq/rocketmq-client-cpp/1.2.4/rocketmq-client-cpp-1.2.4-bin-release.tar.gz)
 and unzip the package, please choose the right version according to your OS and unzip it, then copy the library files to to your `LDFLAGS` directory(as default, `/usr/local/lib/` works finely).
 Please make sure your `LDFLAGS` directory is under the `LD_LIBRARY_PATH`.
- build from source: [Build and Install](https://github.com/apache/rocketmq-client-cpp/tree/master#build-and-install)

### install
1. Go Version: 1.10 or later
2. `go get github.com/apache/rocketmq-client-go`

## How to use

- import package
    ```
    import rocketmq "github.com/lambdaxs/rocketmq-client-go/core"
    ```
- Send message
    ```go
    func SendMessagge(){
        producer := rocketmq.NewProducer(config)
        producer.Start()
        defer producer.Shutdown()
        fmt.Printf("Producer: %s started... \n", producer)
	    for i := 0; i < 100; i++ {
		    msg := fmt.Sprintf("%s-*d", *body, i)
            result, err := producer.SendMessageSync(&rocketmq.Message{Topic: "test", Body: msg})
            if err != nil {
                fmt.Println("Error:", err)
            }
		    fmt.Printf("send message: %s result: %s\n", msg, result)
        }
    }
    ```
- Send ordered message
    ```go
	func sendMessageOrderlyByShardingKey(config *rocketmq.ProducerConfig) {
		producer, err := rocketmq.NewProducer(config)
		if err != nil {
			fmt.Println("create Producer failed, error:", err)
			return
		}

		producer.Start()
		defer producer.Shutdown()
		for i := 0; i < 1000; i++ {
			msg := fmt.Sprintf("%s-%d", "Hello Lite Orderly Message", i)
			r, err := producer.SendMessageOrderlyByShardingKey(
				&rocketmq.Message{Topic: "YourOrderLyTopicXXXXXXXX", Body: msg}, "ShardingKey" /*orderID*/)
			if err != nil {
				println("Send Orderly Message Error:", err)
			}
			fmt.Printf("send orderly message result:%+v\n", r)
			time.Sleep(time.Duration(1) * time.Second)
		}

	}
    ```
- Push Consumer
    ```go
    func ConsumeWithPush(config *rocketmq.PushConsumerConfig) {

	    consumer, err := rocketmq.NewPushConsumer(config)
	    if err != nil {
		    println("create Consumer failed, error:", err)
		    return
	    }

	    ch := make(chan interface{})
	    var count = (int64)(*amount)
	    // MUST subscribe topic before consumer started.
	    consumer.Subscribe("test", "*", func(msg *rocketmq.MessageExt) rocketmq.ConsumeStatus {
		    fmt.Printf("A message received: \"%s\" \n", msg.Body)
		    if atomic.AddInt64(&count, -1) <= 0 {
			    ch <- "quit"
		    }
		    return rocketmq.ConsumeSuccess
	    })

	    err = consumer.Start()
	    if err != nil {
		    println("consumer start failed,", err)
		    return
	    }

	    fmt.Printf("consumer: %s started...\n", consumer)
	    <-ch
	    err = consumer.Shutdown()
	    if err != nil {
		    println("consumer shutdown failed")
		    return
	    }
	    println("consumer has shutdown.")
    }
    ```
- Pull Consumer
    ```go
    func ConsumeWithPull(config *rocketmq.PullConsumerConfig, topic string) {

	    consumer, err := rocketmq.NewPullConsumer(config)
	    if err != nil {
		    fmt.Printf("new pull consumer error:%s\n", err)
		    return
	    }

	    err = consumer.Start()
	    if err != nil {
		    fmt.Printf("start consumer error:%s\n", err)
		    return
	    }
	    defer consumer.Shutdown()

	    mqs := consumer.FetchSubscriptionMessageQueues(topic)
	    fmt.Printf("fetch subscription mqs:%+v\n", mqs)

	    total, offsets, now := 0, map[int]int64{}, time.Now()

    PULL:
	    for {
		    for _, mq := range mqs {
			    pr := consumer.Pull(mq, "*", offsets[mq.ID], 32)
			    total += len(pr.Messages)
			    fmt.Printf("pull %s, result:%+v\n", mq.String(), pr)

			    switch pr.Status {
			    case rocketmq.PullNoNewMsg:
				    break PULL
			    case rocketmq.PullFound:
				    fallthrough
			    case rocketmq.PullNoMatchedMsg:
				    fallthrough
			    case rocketmq.PullOffsetIllegal:
				    offsets[mq.ID] = pr.NextBeginOffset
			    case rocketmq.PullBrokerTimeout:
				    fmt.Println("broker timeout occur")
			    }
		    }
	    }

	    var timePerMessage time.Duration
	    if total > 0 {
		    timePerMessage = time.Since(now) / time.Duration(total)
	    }
	    fmt.Printf("total message:%d, per message time:%d\n", total, timePerMessage)
    }
    ```
- [Full example](../examples)
