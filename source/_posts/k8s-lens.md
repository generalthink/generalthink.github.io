---
title: K8S你知道,K9S你可能也知道，那Lens呢？
date: 2021-01-29 15:15:49
tags: K8S
---

Kubernetes是用于自动部署、扩展和管理“容器化应用程序”的开源系统，而[K9S](https://github.com/derailed/k9s)则提供了一个终端让我们能方便的操作K8S集群，它在github上有10.8K的star

比如查看日志

![查看日志](/images/k8s/k9s_logs.png)


而[Lens](https://github.com/lensapp/lens)也是一个终端工具，它也能让我们更好的操控集群，它在github上也有12.6K star

![Lens](/images/k8s/lens.png)


Lens的使用比K9S更加方便也更加直观,无论是查看Deployment, Service, Pod。它还可以直接access work node,更是方便至极。

![Lens](/images/k8s/lens-info.png)

所以，我推荐大家使用Lens。