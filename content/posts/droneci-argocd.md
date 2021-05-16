---
title: "[K8s] 使用 DroneCI 與 ArgoCD 實現 K8s 自動整合部署"
date: 2021-02-25T10:57:02+08:00
draft: false
categories:
- Ops
- K8s
tags:
- K8s
- DroneCI
- ArgoCD
- CI/CD
- GitOps
description: |-
  這個教學使用 DroneCI 與 ArgoCD 打造 cloud-native 的持續整合交付平台，讓我們在 push commit 或 merge PR 後即可自動跑完測試、打包 image 並部署到 K8s 叢集。必且藉由版控，我們也得以輕鬆 rollback 到之前的任一版本！
---

這個教學使用 DroneCI 與 ArgoCD 打造 cloud-native 的持續整合交付平台，讓我們在 push commit 或 merge PR 後即可自動跑完測試、打包 image 並部署到 K8s 叢集。必且藉由版控，我們也得以輕鬆 rollback 到之前的任一版本！

![](https://i.imgur.com/FygPEyK.png)
<!--more-->
[Source code](https://github.com/minghsu0107/cicd-demo)
## DroneCI 介紹
DroneCI 是一個 cloud-native 的 CI (Continuous Integration) 工具。它很好的整合了 Github、Gitlab 與 Bitbucket 等多種程式碼托管平台，讓我們可以直接同步 repository 到 Drone 上。如同 TravisCI 與其他的 CI 工具一樣，我們可以用一個 yaml 檔描述我們的 pipeline (比如 `.drone.yml`)，而 Drone 在偵測到程式碼異動後就會觸發 webhook 去執行它。

舉例來說，以下的 `.drone.yml` 描述了如何跑一個 Golang 應用的測試並發布到 Dockerhub 上：
```yaml
kind: pipeline

steps:
- name: test
  image: golang
  commands:
  - go test
  - go build

- name: publish
  image: plugins/docker
  settings:
    repo: octocat/hello-world
    tags: [ latest, 1.0, 1 ]
```

Drone 特別的是它的每個 pipeline step 都是一個 container，因此我們可以很大程度的客製化符合自身需求的 pipeline，或是可以很簡單的就啟動測試用的外部服務。而 Drone 官方也提供許多實用的 plugins 供我們使用，比如 [drone-docker](http://plugins.drone.io/drone-plugins/drone-docker/) 可以用來打包 image，而 [drone-slack](http://plugins.drone.io/drone-plugins/drone-slack/) 可以讓我們輕鬆結合 Slack notification。有興趣的話也可以貢獻自己的 plugins，比如這個 Drone 的 [Plugin Registry](http://plugins.drone.io) 彙整了眾多社群提供的 Plugins。
## ArgoCD 介紹
ArgoCD 幫助我們同步 Git 上的 manitests 與 K8s 叢集資源的狀態。也就是說，我們只需要在版控上維護系統的部署狀態 (如 image 的版本、資源限制的設定等)，ArgoCD 就會自動幫我們同步到機器上，並且確認服務是否健康。另外，我們也可以善用大家最熟悉的版控來管理服務，因此 rollback 到任一版本都是非常容易的。

![](https://argoproj.github.io/argo-cd/assets/argocd-ui.gif)

而這樣管理 K8s 叢集與應用程式交付的方式就叫做 **GitOps**。GitOps 讓我們可以維護服務部署狀態的 "source of truth"，進而提升團隊維護的效率與系統的可靠性。
## Overview
![](https://i.imgur.com/FygPEyK.png)

1. 使用者 push 程式碼或是 merge 新的 PR
2. 觸發 webhook，Drone 開始執行定義在 `.drone.yml` 的 CI pipeline
3. 若測試通過就發布新的 image 到 Dockerhub 上，並更新 manitests repository 上的 image 版本
4. ArgoCD 偵測到 manifests 的變動，因此通知 K8s 更新 image 並同步部署狀態
## 事先準備
Source code 可以看 [這裡](https://github.com/minghsu0107/cicd-demo)。

1. 一個 Drone server
    - [Github installation](https://docs.drone.io/server/provider/github/)
2. 一個測試用 K8s 叢集
    - [K3d](https://k3d.io)
    - [minikube](https://minikube.sigs.k8s.io/docs/start/)
    - [K0s](https://github.com/k0sproject/k0s)
3. 部署 ArgoCD 到叢集上
    - [All-in-one installation](https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd)
4. 一個 Github 帳號與一個 Dockerhub 帳號

## DroneCI
### Setup
當你成功的在 Drone 上連動 Github 帳號後，你可以在 Drone 的 dashboard 上看到所有的 repo。接著複製這個 repo、activate 它並前往 `Repositories -> cicd-demo -> settings` 新增以下 secret：

![](https://user-images.githubusercontent.com/50090692/111301242-f04c2e80-868c-11eb-9945-c2de30b0ee92.png)

- `docker_username`: 你的 Dockerhub 帳號
- `docker_password`: 你的 Dockerhub 密碼
- `ssh_key`: Github 的 SSH private key


最後修改 `.drone.yml`，把 `minghsu0107` 替換成你自己的 Github 與 Dockerhub 帳號。現在在 `main` branch 上的 push 或 pull request 都會觸發 Drone pipeline，有關 Drone 的 Github webhook 的設定可以前往  `your repo -> setting -> webhook` 查看。
### 本機開發
在本機開發時，我們會想要檢查 `.drone.yml` 是否撰寫正確但又不想每次都 push 到 repo。此時我們可以使用 [Drone CLI](https://docs.drone.io/cli/install/)。它可以讓我們在本機執行 pipeline，並且可以 include 或 exclude 某幾個 step，非常適合用在開發上。

使用 CLI 登入 Drone:
```bash
export DRONE_SERVER=<drone-server-url>
export DRONE_TOKEN=<drone-token> # check token under: dashboard -> user setting
drone info
```
舉例來說，我們可以僅執行 `test` 這個 step：
```bash
drone exec --include=<pipline-step-name>
```
## ArgoCD
開始前請先 clone 這個 [manifest repository](https://github.com/minghsu0107/cicd-demo-manifests)。這個 repo 負責所有有關部署應用的 manifests，之後會用來與 ArgoCD 同步。這邊使用 [Kustomize](https://github.com/kubernetes-sigs/kustomize) 這個模板工具維護 K8s 的 resources，而且 Kustomize 是被 ArgoCD 原生支援的，這裡就不細談 Kustomize 的使用方法了，只要知道它在這裡幫助我們區分 `dev` 與 `prod` 環境，並提高 manifests 的可讀性即可。


如果你的 repo 是 private 的，你必須要在 ArgoCD 上設定 credentials，否則就可以跳過這個步驟。

使用 Argo CD CLI:
```bash
argocd repo add <repo-url> --username=<username> --password=<password>
```
也可以使用 GUI：前往 `Settings/Repositories`、點擊 `Connect Repo using HTTPS` 並輸入 credentials：

![](https://i.imgur.com/UAyNkte.png)

你會看到如以下的畫面：

![](https://i.imgur.com/XaMezBA.png)

新增一個 app：

![](https://i.imgur.com/gOD9h1b.png)

![](https://i.imgur.com/8XlNtDL.png)

![](https://i.imgur.com/JK76lnT.png)

記得把 repo 替換成你自己的。

現在我們完成了所有的準備，前往 `/applications` 並點擊 `SYNC` 就可以看到 ArgoCD 自動同步了 cluster 的狀態！

![](https://i.imgur.com/RVH5QtL.png)

也可以點進去 app 看看一些詳細資訊：

![](https://i.imgur.com/pconXQR.png)
## 總結
DroneCI 與 ArgoCD 都是滿好上手的工具，而且功能也很強大。今後大家若要建置 CI/CD 平台，但又希望自己 host 伺服器而非使用第三方提供的服務 (可能有成本的考量等)，DroneCI 與 ArgoCD 的組合是一個值得考慮的選擇。
## Reference
- https://www.weave.works/technologies/gitops/
- https://argo-cd.readthedocs.io/en/stable/
- https://docs.drone.io
- https://hub.docker.com/r/line/kubectl-kustomize


