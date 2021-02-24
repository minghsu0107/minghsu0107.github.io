---
title: "JCConf 2020 心得"
date: 2021-02-24T16:27:23+08:00
draft: false
categories:
- Conference
tags:
- JCConf
- Java
---

以下與大家分享一些這次 JCConf 2020 中我認為值得討論的主題。

整體來說，滿多議程在推廣 Kotlin 、serverless 與微服務的使用，而大家滿心期待的 Java virtual thread 也已如火如荼的開發中！

![](https://i.imgur.com/T6Ib4ws.png)
<!--more-->

## 用開源的 MySQL 叢集解決大型網路應用的擴充性問題
現今後端應用的 bottleneck 常常是出現在資料庫上。因此當資料量大到一定程度時，解決資料庫的吞吐瓶頸就會成為一大挑戰。

常見提高資料庫吞吐量的方式有垂直擴充與水平擴充。垂直擴充是以增加 cpu 與 RAM 等硬體資源為機器升級以提高讀寫效能，而水平擴充則是增加機器並運用分布式架構擴充資料規模，比較常見的方式有主從式讀寫分離架構或資料分片 (sharding) 等。然而對於傳統關聯式資料庫來說，水平擴展比起 NoSQL 困難許多，因為其涉及到了資料的切分或一致性等複雜的問題。

### MySQL 叢集

為了解決上述的痛點，MySQL 提出了最新的高可用方案，包含了 MySQL Replication、MySQL Group Replication、Shared Disk Based Active/Passive 與 MySQL Cluster：

![](https://i.imgur.com/djvJIV2.png)

舉例來說，使用 MySQL Cluster 和 MySQL Group Replication多主模式的話，便可用 Connector/J 這個 Java SQL driver 達成負載平衡。而 MySQL Replicatino 與 MySQL Group Replication 的單主模式下則可用 Connector/J 的複製/讀寫分離模式 (只需將 connection object 設為 read-only)。另外，只需一行便可在 Connector/J 的 JDBC 上完成故障移轉。

因此，當我們的環境有以下的擴充需求時就可以考慮使用 MySQL 叢集：
- 高可用不停機
- 寫的高吞吐
- 高併發連線
- 高擴展
- 使用簡單和具有彈性

![](https://i.imgur.com/7m08DGY.png)
### MySQL & NoSQL
Cluster/J 是個另一個 MySQL Cluster 的 driver。它不經 SQL 節點，在 Java程式中透過 JNI 直接呼叫以 C 所開發的NDB API，目的是將 MySQL 以表為主的資料對應到 Java 程式的物件。使用 Cluster/J 便可以 NoSQL 的方式直接取用 MySQL Cluster 上的資料。Cluster/J 以 ClusterJPA 抽象化，支援JPA 而增加可攜性並提供以下特性：
- Persistent classes
- Relationships
- Joins in queries
- Lazy loading
- Table and index creation from object model

![](https://i.imgur.com/yi24JJM.png)

這是在官網上拿來的例子，假設我們現在要新增一位 employee：
```java
Employee newEmployee = session.newInstance(Employee.class);
```
`session` 物件對應一個 cluster/J 與 MySQL 的 connection.
下面的程式碼對應到 `insert` 操作，有些 ORM 的味道：
```java
emp.setId(988);

newEmployee.setFirstName("John");
newEmployee.setLastName("Jones");

newEmployee.setStarted(new Date());
```
而這段查詢的程式對應到 `SELECT * FROM employee WHERE id = 988`：
```java
Employee theEmployee = session.find(Employee.class, 988);
```

Cluster/J的增刪改的性能非常好，幾乎和 native NDB API 差不多，查詢為一般JDBC的兩倍快：

![](https://i.imgur.com/nv0XTbe.png)
### 何時使用
由於 Cluster/J 直接訪問 Data Node 可達2億 QPS，而 Connector/J 對 SQL Node 則可到每秒250萬個 SQL 指令。因此推薦**複雜的查詢可以透過 SQL 指令查詢，而 Key-Value 的操作則以 Cluster/J 加快操作速度**。而這些都是建立在 MySQL 叢集上的，因此兩種方式都可保有高可用與擴充性。
## RSocket
RSocket 是個使用 byte stream 的 binary protocol。它支援四種 interaction models：
1. request/response (stream of 1)
2. request/stream (finite stream of many)
3. fire-and-forget (no response)
4. channel (bi-directional streams)



由於另一位也有參加會議的 Allen 已詳細解說了與 Spring Boot 整合的使用方式。我在此補充並分享一些 RSocket 的特點還有與 gRPC (HTTP/2) 的比較，讓大家更清楚為何一些大公司如阿里巴巴與 Facebook 會選擇在 production 中使用 RSocket 作為通訊基礎。

- **輕量**：RSocket 的四個 interactive model 被規範在 RSocket 協定內，而 gRPC 則是藉由 HTTP/2 Stream 傳輸這些 RPC 的資訊，因此 gRPC 傳輸的 overhead 會比較大。
- **彈性**：RSocket 在資料的傳輸上更具有彈性，它所需要的只是一個 duplex connection，不像 gRPC 需要事先定義好 protobuf 並且只能使用 protobuf 有定義好的 procudure。另外，gRPC 需要 code generation，但 RSocket 提供開發者更多選擇：喜歡 RPC style 的人可以使用 RSocket-RPC，希望與 Spring 整合的話也可以使用 Spring Messaging，當然直接操作 RSocket 也完全沒問題。
- **Session 複用**：RSocket 可以在多次連線之間恢復 long-lived streams。這使的 RSocket 特別適合 mobile 與 server 端的溝通場景。
- **Back-pressure**：RSocket 實作了 reactive streams 並支援異步 back-pressure (消費者需要多少，生產者就生產多少)，這樣的 application-level flow control 是基於 HTTP/2 的 gRPC 無法做到的。
- **既是 client 也是 server**：RSocket 在連線建立後 server 與 client 都能成為 requester 與 responder。換句話說 RSocket 是 fully duplex 的，client 與 server 都能在同個連線下發出和接受請求。相較之下 gRPC 是標準的 client-server 模型，因此 server 端無法向 client 發出請求。
- **瀏覽器支援**：gRPC 在瀏覽器中沒有直接支援，需要額外的函式庫且提高了維護的複雜度。而 RSocket 可以藉由 Websocket 在瀏覽器直接運作，我們只需要開啟一個接受 Websocket 連線的 RSocket instance 即可。

總結來說，RSocket 為 reactive programming 與 event driven 架構打下了可靠的基礎。它為我們帶來嶄新的服務通訊方式，而其輕量的特性也使我們能更輕鬆擴展 real-time 的 application。
## 其他議程
- **街口支付的唯服務之路**：分享了街口支付如何將高耦合的單體架構重構成領域服務邊界明確的微服務：
    1. 定義業務領域 (Business Domain) - 找出並定義所有商業行為所包含的 domain knowledge，與該領域的專業人討論再進行業務切分。
    2. 明確領域服務邊界 - 強調 Conway's Law，系統架構即是公司組織的反映 。
    3. 重構與持續優化
- **快來一起認識 Helm 吧！**：介紹了 Helm 這個 Kubernetes 的套件管理工具，並現場使用 [k0s](https://github.com/k0sproject/k0s) + Docker-Compose 實作。詳細內容請看 [Github](https://github.com/CookieTsai/jcconf-helm-demo)。
## 總結
第一次參加 JCConf 這個 Java 年度盛會覺得收穫很多。看到了許多很有料的講者，也深刻的體會到了 Java 社群源源不絕的熱忱與活力。中場休息時間在攤位上也聽到許多新鮮有趣的 sharing。熱愛 Java 跟收集貼紙的人絕對不能錯過！
## Reference
- [MySQL Cluster Slides, JCConf 2020](https://drive.google.com/file/d/1CBiHgP2aGYG45gvYADBzHCtOUapMqan2/view?fbclid=IwAR0BAo5Q3N-aid1xcWYzuJYo-yLe_s7uk6fZ9dzKkF2T3KL5NM9Jrl4xlcY)
- [Differences between gRPC and RSocket - by Robert B Roeser
](https://medium.com/netifi/differences-between-grpc-and-rsocket-e736c954e60)
- [k0s](https://github.com/k0sproject/k0s)
- [Helm Demo, JCConf 2020](https://github.com/CookieTsai/jcconf-helm-demo)

