---
title: "輕鬆「Go」建事件驅動應用"
date: 2021-02-25T23:57:02+08:00
draft: false
categories:
- Web
- Golang
tags:
- Event-driven
- Web
- Golang
- Watermill
---
這篇文章統整了我在 Golang Taipei #55 Meet Up 分享的內容。

Event-driven architecture 在近幾年越來越受關注，它不僅幫助我們解耦服務組件、反轉依賴，更可提高系統的 throughput，大幅提升了擴展性。

這次主題會講解 Event-driven 的核心概念，簡介幾種常見的分佈式消息系統，並展示如何輕鬆用 Golang 實作 event-driven application，幫助大家能更快理解。

![](https://i.imgur.com/RuF8Gbk.png)
<!--more-->
## 什麼是 Event
Event 可以是一個「改變系統狀態」的變化，也可以是陳述當前系統狀態的「事實」，如使用者的點擊、sensor 的資料流、一筆成立的訂單資訊等。而產生 event 的一方叫生產者 (producer)，接收 event 的一方叫消費者 (consumer)。

![](https://i.imgur.com/dxLNqaD.png)

如上圖，event publisher 產生了一個 event 後，數個對此事件有興趣的 consumer 都可以訂閱它。
## 事件驅動架構
主流的架構有兩種：

1. Pub/Sub model

消費者訂閱一到數個事件流，當一個事件產生(被發布)後會被送給有訂閱它的消費者

2. Event streaming

事件被 append 進 log 並存在 event store。不同於 Pub/Sub 模型，Consumer 可以參與任一事件流並從任何一個時間點開始 "replay"，產生一個 view，這個過程叫做 view generation。相關的應用有 Event sourcing、CQRS (讀寫分離) 等。


![](https://i.imgur.com/aPDXAsU.png)

上圖簡單的描述了一個 event sourcing 的架構。由 end users 產生的各種 event 會被存在 event store，而每個 consumer 各對應到一個 view generation，並寫道 read storage，而外部所有的 query 只會訪問 read storage。這樣讀寫分離的架構讓我們能更有彈性的根據不同場景選擇適合的 DB，比如需要全文檢索的時候就可以考慮使用 ElasticSearch 作為 read storage 等。
## Why 事件驅動
- 使用事件來溝通不同服務幫助我們明確建立 Domain event、並且維持程式邊界，進而維持服務自治性。
- 解耦系統組件、鬆散依賴。
- 事件的異步處理幫助提高系統的 throughput、提高整體架構的擴展性。
- 反轉依賴，讓系統更貼近真實業務邏輯關係。
- 幫助我們建構 [responsive system](https://www.reactivemanifesto.org)：

![](https://i.imgur.com/5pPEZwZ.png)

## 事件驅動與微服務
![](https://i.imgur.com/OiY3GoP.png)
- 圖源：[Microsoft Docs](https://docs.microsoft.com/zh-tw/dotnet/architecture/microservices/multi-container-microservice-net-applications/integration-event-based-microservice-communications)

從上圖可以看到，微服務之間藉由 event bus (message broker) 使用事件彼此溝通 (pub/sub)，而每個微服務都各自維護一個 database，並訂閱與自身服務相關的事件。由此可以發現，一個微服務可以是生產者、消費者、或是兩者都是。
## 注意事項
- 使用前需要評估系統對資料一致性的要求：當資料具最終一致性的場境較容易處理
- 要小心分散式交易時的資料一致性問題：在分散式的架構下，我們不再能使用 單個  DB transaction 確保交易的原子性。當一筆交易分佈在多個服務時確保交易一致性的方法：多階段提交、saga pattern。
- 額外的維運成本：相比單體架構，有更多的服務與外部系統要維護、監控

小結：**事件驅動並不是 silven bullet，還是要看應用場景選擇最適合的架構**。
## 用 Golang 實作事件驅動
### Watermill
- https://github.com/ThreeDotsLabs/watermill

Watermill 是一個幫助我們實作 message streaming 的 Golang library，它統一 publish/subscribe 介面，因此可以輕鬆換到不同底層 broker 而不需修改核心程式碼。

另外，它有著充滿彈性的 API ，讓我們可以掌握 broker-specific 的設定，比如使用 Kafka 時可以直接 override Sarama 的 config，完成更細部的客戶端設定。Watermill 同時也提供開箱即用的 middleware，讓我們不用手刻 Timeout、Retry、Recovery 等諸多功能。值得一提的是，除了 Kafka 或 RabbitMQ 等常見的 broker，watermill 也支持 HTTP 或是 MySQL binlog，因此實用性滿高的。

Watermill 的核心就是 Pub/Sub 的 interface。它將所有種類的 message broker 都封裝成 Publisher 與 Subscriber 的介面，讓我們的程式碼可以與底層的 client library 解耦：
```go
type Publisher interface {
    Publish(topic string, messages ...*Message) error
    Close() error
}
type Subscriber interface {
Subscribe(ctx context.Context, topic string) (<-chan *Message, error)
    Close() error
}
```
### Demo
完整程式碼請看[這裡](https://github.com/minghsu0107/golang-taipei-watermill-example)。

![](https://i.imgur.com/Yt6MIsA.png)

在這個 Demo 中，有一個 publisher 每三秒向 `incoming_topic` 發布一個新消息。同時， `helloHandler` 與 `incomingTopicHandler` 訂閱了 `incoming_topic` 這個主題，而 `outgoingTopicHandler` 則訂閱了 `outgoing_topic`。當 `helloHandler` 收到了一個新消息，它會再發布另一個 greeting message 到 `outgoing_topic`，因而讓訂閱了 `outgoing_topic` 的 outgoingTopicHandler 收到了這個 greeting message。

接著來看看程式碼實作。首先我們建立一個 router，router 負責管理所有的 pub/sub handler，並且可以在 router 上註冊 global 的 middleware：
```go
router.AddPlugin(plugin.SignalsHandler)
router.AddMiddleware(
    middleware.CorrelationID,
    middleware.Timeout(time.Second*10),
    middleware.NewThrottle(10, time.Second).Middleware,
    middleware.Retry{
        MaxRetries: 5,
        Logger:     logger,
    }.Middleware,
    middleware.Recoverer,
)
```
這邊展示了一些常用的 middleware，比如 Timeout、Throttle、Retry  與 Recovery 機制等。

由於我們使用 NATS Streaming 作為底層的 broker，因此我們可以寫一個 NATS Streaming client 的 factory，它會回傳前面所提到的 Publisher 或是 Subscriber：
```go
func NewNATSPublisher(logger watermill.LoggerAdapter, clusterID, natsURL string) (message.Publisher, error) {
    return nats.NewStreamingPublisher(
        nats.StreamingPublisherConfig{
            ClusterID: clusterID,
            ClientID:  watermill.NewShortUUID(),
            StanOptions: []stan.Option{
                stan.NatsURL(natsURL),
            },
            Marshaler: marshaler,
        },
        logger,
    )
}
func NewNATSSubscriber(logger watermill.LoggerAdapter, clusterID, clientID, natsURL string) (message.Subscriber, error) {
    return nats.NewStreamingSubscriber(
        nats.StreamingSubscriberConfig{
            ClusterID: clusterID,
            ClientID:  clientID,
            StanOptions: []stan.Option{
                stan.NatsURL("nats://nats-streaming:4222"),
            },
            Unmarshaler: marshaler,
        },
        logger,
    )
}
```
接著我們向 router 註冊 `helloHandler` 與只有作用在 `helloHandler` 的 middleware：
```go
handler := router.AddHandler(
    "hello_handler",
    incomingTopic,
    subscriber,
    outgoingTopic,
    publisher,
    helloHandler{}.Handler,
)

handler.AddMiddleware(func(h message.HandlerFunc) message.HandlerFunc {
    return func(message *message.Message) ([]*message.Message, error) {
        fmt.Printf("\nexecuting hello_handler specific middleware for %s", message.UUID)
        return h(message)
    }
})
```
註冊 `incomingTopicHandler` 與 `outgoingTopicHandler`：
```go
router.AddNoPublisherHandler(
    incomingTopic+"_handler",
    incomingTopic,
    subscriber,
    incomingTopicHandler{}.HandlerWithoutPublish,
)

router.AddNoPublisherHandler(
    outgoingTopic+"_handler",
    outgoingTopic,
    subscriber,
    outgoingTopicHandler{}.HandlerWithoutPublish,
)
```
最後在背景每三秒向 `incomingTopic` 發布訊息，同時啟動 router：
```go
go publishMessages(incomingTopic, publisher)

ctx := context.Background()
if err := router.Run(ctx); err != nil {
    log.Fatal(err)
}

func publishMessages(topic string, publisher message.Publisher) {
    for {
        msg := message.NewMessage(watermill.NewUUID(), []byte("Hello, watermill!"))
        middleware.SetCorrelationID(watermill.NewUUID(), msg)

        fmt.Printf("\n\n\nSending message %s, correlation id: %s\n", msg.UUID, middleware.MessageCorrelationID(msg))
        if err := publisher.Publish(topic, msg); err != nil {
            log.Fatal(err)
        }
        time.Sleep(3 * time.Second)
    }
}
```

啟動服務：
```bash
docker-compose up
```
若把 `WATERMILL_PUBSUB_TYPE` 環境變數設為空字串就可以將底層的 broker 換成 GoChannel Pub/Sub，有興趣可以實驗看看，兩者的 output 會是一模一樣的。
## Messaging Systems
前面談到 message broker 可以是各種不同的實作，比如 RabbitMQ 或 Kafka 等。但各種訊息系統有什麼不同呢？這邊會做一個簡單的比較與整理。

下圖來自我在簡報中做的 Message broker 比較與整理：

![](https://i.imgur.com/XeU2Bjr.png)

Kafka 是許多企業的 event streaming 平台首選，這是因爲他的高 throughput、擴展性與可靠性。而 RabbitMQ 實作了 AMQP 協定，它豐富的 routing 機制讓我們可以處理很複雜的資料流。而 NATS Streaming 是三者中最年輕的，它是以 Golang 實作的 CNCF 專案，十分輕快速，部署也相對容易，並且結合了前述兩者的優點。

舉例來說，NATS Streaming 有以下特性：

![](https://i.imgur.com/RdSrRl2.png)

由上圖可以看到，NATS Streaming 結合了 Kafka 的 ConsumerGroup 特點與 RabbitMQ 的路由 matching 機制，讓我們在開發上有更多選擇的彈性。
## 總結
事件驅動能夠以貼近真實業務邏輯的方式描述系統架構，並幫助我們解耦服務依賴，提高擴展性，而 Watermill 更使得這一切變的容易實現。

很高興這次能夠在 Golang Taipei 分享我的一些想法，這次的講者經驗也讓我體會到社群滿滿的熱情與活力！
## Reference
- https://www.rabbitmq.com
- http://kafka.apache.org
- https://nats.io
- https://www.redhat.com/en/topics/integration/what-is-event-driven-architecture
- https://arxiv.org/pdf/1912.03715.pdf
- https://github.com/ThreeDotsLabs/watermill
- https://watermill.io/docs/cqrs/#building-a-read-modelwith-the-event-handler

