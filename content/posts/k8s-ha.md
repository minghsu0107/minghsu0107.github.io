---
title: "Learn Kubernetes HA Architecture with Hands-On Example"
date: 2021-04-27T02:27:23+08:00
draft: false
categories:
- Ops
- K8s
tags:
- K8s
- HA
- Keepalived
---
The following diagram shows a typical high-availibility Kubernetes cluster with embedded etcd: 

![](https://i.imgur.com/FwEzzWV.png)

However, HA architecture also brings about another problem - which K8s API server should the client connect to? How to detect a node failure while fowarding traffic in order to achieve high availibility?
<!--more-->

Each of K8s controller plane replicas will run the following components in the following mode:
- etcd instance: all instances will be clustered together using consensus;
- API server: each server will talk to local etcd - all API servers in the cluster will be available;
- controllers, scheduler, and cluster auto-scaler: will use lease mechanism - only one instance of each of them will be active in the cluster;
- add-on manager: each manager will work independently trying to keep add-ons in sync

But how to determine which K8s API server the client should connect to while ensuring healthiness of the backend servers? Simply adding mulitple DNS A records does not solve the problem since DNS server could not check server healthiness in real time. A reverse proxy with upstream health check is still not good enough because it would become a single-point-of-failure.

**The concept of virtual ip comes to the rescue**.

To achieve high availibility, we will use Keepalived, a LVS (Linux Virtual Server) solution based on VRRP protocol. A LVS service contains a master and multiple backup servers while exposing a virtual ip to outside as a whole. The virtual ip will be pointed to the master server. The master server will send heartbeats to backup servers. Once backup servers are not receiving heartbeats, one backup server will take over the virtual ip, ie. the virtual ip will "float" to that backup server.

To sum up, we have the following architecture:

![](https://i.imgur.com/FrwyxCQ.png)

1. A floating vip maps to three keepalived servers (one master and two backups). Each keepalived server runs on a K8s controller plane replica.
2. When keepalived master server fails, vip floats to another backup server automatically.
3. HAProxy load-balances traffic to three K8s API servers.

The following section is hands-on tutorial that implements the above architecture.
## Settings
- Three controller plane nodes in private network: `172.16.0.0/16`
- Node private IPs: `172.16.217.171`, `172.16.217.172`, `172.16.217.173`
- Virtual IP: `172.16.100.100`, `172.16.100.101`
## Usage
You can find full source code on [my Github](https://github.com/minghsu0107/k8s-ha).

HAProxy configuration for load balancing on K8s API servers (`haproxy.cfg`):
```bash
global
    log /dev/log  local0 warning
    maxconn     4000
    daemon
  
defaults
  log global
  option  httplog
  option  dontlognull
  timeout connect 5000
  timeout client 50000
  timeout server 50000


frontend kube-apiserver
  bind *:"$HAPROXY_PORT"
  mode tcp
  option tcplog
  default_backend kube-apiserver

backend kube-apiserver
  mode tcp
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  server kube-apiserver-1 172.16.217.171:6443 check # simple http health check
  server kube-apiserver-1 172.16.217.172:6443 check
  server kube-apiserver-1 172.16.217.173:6443 check
```

Keepalived configuration (`keepalived.conf`):
```bash
global_defs {
  default_interface {{ KEEPALIVED_INTERFACE }}
}

vrrp_script chk_haproxy {
    # check haproxy
    script "/bin/bash -c 'if [[ $(netstat -nlp | grep 7443) ]]; then exit 0; else exit 1; fi'"
    interval 2 # check every two seconds
    weight 11
}

vrrp_instance VI_1 {
  interface {{ KEEPALIVED_INTERFACE }}

  state {{ KEEPALIVED_STATE }}
  virtual_router_id {{ KEEPALIVED_ROUTER_ID }}
  priority {{ KEEPALIVED_PRIORITY }}
  advert_int 2

  unicast_peer {
    {{ KEEPALIVED_UNICAST_PEERS }}
  }

  virtual_ipaddress {
    {{ KEEPALIVED_VIRTUAL_IPS }}
  }

  authentication {
    auth_type PASS
    auth_pass {{ KEEPALIVED_PASSWORD }}
  }

  track_script {
    chk_haproxy
  }

  {{ KEEPALIVED_NOTIFY }}
}
```
This configure keepalived server to check if HAProxy is healthy every 2 seconds. Some important environment variables:
- `KEEPALIVED_INTERFACE`: Keepalived network interface. Defaults to eth0
- `KEEPALIVED_PASSWORD`: Keepalived password. Defaults to d0cker
- `KEEPALIVED_PRIORITY` Keepalived node priority. Defaults to 150
- `KEEPALIVED_ROUTER_ID` Keepalived virtual router ID. Defaults to 51
- `KEEPALIVED_UNICAST_PEERS` Keepalived unicast peers. Defaults to `192.168.1.10`, `192.168.1.11`
- `KEEPALIVED_VIRTUAL_IPS`: Keepalived virtual IPs. Defaults to `192.168.1.231`, `192.168.1.232`
- `KEEPALIVED_NOTIFY`: Script to execute when node state change. Defaults to `/container/service/keepalived/assets/notify.sh`
- `KEEPALIVED_STATE`: The starting state of keepalived; it can either be `MASTER` or `BACKUP`.

We will use docker to run HAProxy and on each K8s controller plane replicas.

On first node:
```bash
docker run -d --net=host --volume $(pwd)/config/haproxy:/usr/local/etc/haproxy \
    -e HAPROXY_PORT=7443 \
    haproxy:2.3-alpine
docker run -d --cap-add=NET_ADMIN --cap-add=NET_BROADCAST --cap-add=NET_RAW --net=host \
    --volume $(pwd)/config/keepalived/keepalived.conf:/container/service/keepalived/assets/keepalived.conf \
    -e KEEPALIVED_INTERFACE=eth0 \
    -e KEEPALIVED_PASSWORD=pass \
    -e KEEPALIVED_STATE=MASTER \
    -e KEEPALIVED_VIRTUAL_IPS="#PYTHON2BASH:['172.16.100.100', '172.16.100.101']" \
    -e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['172.16.217.171', '172.16.217.172', '172.16.217.173']" \
    osixia/keepalived:2.0.20 --copy-service
```
On second node:
```bash
docker run -d --net=host --volume $(pwd)/config/haproxy:/usr/local/etc/haproxy \
    -e HAPROXY_PORT=7443 \
    haproxy:2.3-alpine
    --volume $(pwd)/config/keepalived/keepalived.conf:/container/service/keepalived/assets/keepalived.conf \
docker run -d --cap-add=NET_ADMIN --cap-add=NET_BROADCAST --cap-add=NET_RAW --net=host \
    -e KEEPALIVED_INTERFACE=eth0 \
    -e KEEPALIVED_PASSWORD=pass \
    -e KEEPALIVED_STATE=BACKUP \
    -e KEEPALIVED_VIRTUAL_IPS="#PYTHON2BASH:['172.16.100.100', '172.16.100.101']" \
    -e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['172.16.217.171', '172.16.217.172', '172.16.217.173']" \
    osixia/keepalived:2.0.20 --copy-service
```
On third node:
```bash
docker run -d --net=host --volume $(pwd)/config/haproxy:/usr/local/etc/haproxy \
    -e HAPROXY_PORT=7443 \
    haproxy:2.3-alpine
docker run -d --cap-add=NET_ADMIN --cap-add=NET_BROADCAST --cap-add=NET_RAW --net=host \
    --volume $(pwd)/config/keepalived/keepalived.conf:/container/service/keepalived/assets/keepalived.conf \
    -e KEEPALIVED_INTERFACE=eth0 \
    -e KEEPALIVED_PASSWORD=pass \
    -e KEEPALIVED_STATE=BACKUP \
    -e KEEPALIVED_VIRTUAL_IPS="#PYTHON2BASH:['172.16.100.100', '172.16.100.101']" \
    -e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['172.16.217.171', '172.16.217.172', '172.16.217.173']" \
    osixia/keepalived:2.0.20 --copy-service
```
Finally, point your dns to either `172.16.100.100` or `172.16.100.101`. From now on, a request with url `<https://your.domain.com:7443>` will be resolved to `https://<172.16.100.100|172.16.100.101>:7443`, rounted to the current keepalived master, and eventually sent to one of the three K8s API servers by HAProxy.
## Reference
- https://kubernetes.io/docs/tasks/administer-cluster/highly-available-master/
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/
- https://www.kubernetes.org.cn/6964.html
- https://www.mdeditor.tw/pl/pRgX/zh-tw