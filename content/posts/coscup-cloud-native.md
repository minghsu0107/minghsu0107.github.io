---
title: "COSCUP X Cloud Native"
date: 2021-08-08T12:12:23+08:00
draft: false
categories:
- Conference
tags:
- COSCUP
- Cloud-Native
- Golang
description: |-
  與大家分享我在 COSCUP 2021 擔任 Cloud Native 議程講者的一些內容簡介與感想。雖然這次是線上議程，但大家在各平台上的積極發問與留言實在令人感到社群滿滿的活力。

  這次我分享的主題是 "Empower Your Kubernetes with Service Mesh + Distributed Tracing"，主要介紹了 service mesh 與 distributed tracing 的基礎概念，並帶領大家以 kubernetes-native 的方式實際部署一個範例應用，將 distributed tracing 與 service mesh 無縫整合。
---
與大家分享我在 COSCUP 2021 擔任 Cloud Native 議程講者的一些內容簡介與感想。雖然這次是線上議程，但大家在各平台上的積極發問與留言實在令人感到社群滿滿的活力。

這次我分享的主題是 "Empower Your Kubernetes with Service Mesh + Distributed Tracing"，主要介紹了 service mesh 與 distributed tracing 的基礎概念，並帶領大家以 kubernetes-native 的方式實際部署一個範例應用，將 distributed tracing 與 service mesh 無縫整合。

{{< embed slide_page="3" data_id="15d69d4ce9204cb2aa56855ece74ae02" >}}

<!--more-->

Service mesh 是個提供應用程式安全性、可靠性、監控功能、並且快速的平台基礎設施。它接管了微服務之間複雜的網路溝通，並且提供開箱即用的負載均衡、憑證管理、流量控制、與監控追蹤等功能，而應用程式不會意識到它的存在。


{{< embed slide_page="8" data_id="15d69d4ce9204cb2aa56855ece74ae02" >}}


Service mesh 為我們節省了團隊開發上的溝通成本，它將與應用本身無關的一切網路相關實作轉移到 Kubernetes 上，並且透明可調控。除此之外，service mesh 甚至可以橫跨多個叢集，這對於使用混合雲等注重資料安全的企業無疑是個的好選擇。


{{< embed slide_page="9" data_id="15d69d4ce9204cb2aa56855ece74ae02" >}}


這邊介紹一個很棒的 service mesh 開源方案 -- Linkerd。相較於其他的 service mesh，比如當前很流行的 Istio，Linkerd 更輕量、好上手與易維護，而且他的功能齊全、運行快速且使用很少的資源。 


{{< embed slide_page="10" data_id="15d69d4ce9204cb2aa56855ece74ae02" >}}


Linkerd 為我們的應用提供了三大 golden metrics，分別是 success rate、rps、與 latency。這三大項指標幫助我們更能了解應用程式運行時可能發生的瓶頸，而這些 metrics 都可以輕鬆的在 Linkerd 的 dashboard 上觀測。


{{< embed slide_page="13" data_id="15d69d4ce9204cb2aa56855ece74ae02" >}}


另一方面，distributed tracing 則是在分散式系統中觀察請求與流量的一系列技術。它讓微服務之間的流量變得透明而易追蹤，在系統效能評測與除錯近來已被廣泛使用。藉由 distributed tracing，我們得以跨越 service boundary 追蹤所有被請求的服務 (End-to-End Visibility)，並由 latencies 分析與了解系統瓶頸。除此之外，我們更可以輕鬆得到所有服務的依賴關係圖。


{{< embed slide_page="16" data_id="15d69d4ce9204cb2aa56855ece74ae02" >}}


在收集 log 與 tracing span 方面，推薦大家使用 Grafana + Loki + Jaeger 這個 techstack。Grafana 是個非常彈性又功能強大的 dashboard，他可以幫我們彙整所有的 metrics 並繪製成各種圖表，同時也可設定各種警報機制以在問題發生時第一時間通知維運人員。Loki 則是一款輕量的 log 收集系統，他在每個叢集節點上運行一個 daemon (Daemonset)、收集 container 的 log，最後推送到存儲系統。而 Jaeger 是一套功能完善的分布式追蹤系統，他幫助開發者輕鬆的為應用程式加上追蹤的功能。


{{< embed slide_page="18" data_id="15d69d4ce9204cb2aa56855ece74ae02" >}}


Service mesh 與 distributed tracing 兩者皆可分析 service health、latency 與 topology 。然而 Service mesh 除了 observability 外也提供安全性與流量管控，且其為無狀態的設計，因此若要永久保存 trace 還是需要用到 Jaeger 等分布式追蹤方案，我們該如何結合兩者呢？

話不多說，我們實際部署一個範例應用來直接學習如何結合 service mesh 與 distributed tracing，這是一個簡單的表情符號投票系統，包含了以下四個微服務。
- voting - 負責表情符號的投票
- emoji - 負責列出所有的表情符號
- web - UI 介面
- vote-bot - 一個不斷投票以產生流量的程式

其中 vote-bot 以 REST API 訪問 web，接著 web 會以 gRPC API 訪問 voting 與 emoji 微服務。


{{< embed slide_page="27" data_id="15d69d4ce9204cb2aa56855ece74ae02" >}}


我把所有程式碼都放在[這裡](https://github.com/minghsu0107/emojivoto-kustomization)，這個範例包含了以下的內容：
1. 在 Kubernetes 上運行 Linkerd 與 Jaeger backend
2. 啟用 Linkerd 的 tracing 功能並連接 Jaeger backend
3. 在 Linkerd 上部署微服務並注入 Linkerd proxy
4. 使用 Opencensus Collector 串接 application 與 Linkerd proxy 的 span


總體來說，今年的與會者眾多，大家非常踴躍的發問與樂於分享技術，而各攤位的 short talk 也都非常有深度，讓人收穫良多。最後特別感謝 LINE 的贊助，讓我有機會站上這樣的舞台與大家分享技術！