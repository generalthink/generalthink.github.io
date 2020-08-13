---
title: Kubernetes常用命令
date: 2019-12-19 17:58:24
tags:
- Kubernetes
- K8S
---


#### 获取所有的namespace

```
kubectl get namespace
```
> 默认使用的是 ~/.kube/config这个配置文件，如果文件不再这个目录下可以通过 --kubeconfig configFilePath>指定
> https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#set-the-kubeconfig-environment-variable

<!--more-->

#### 生成config文件
```
kubectl config --kubeconfig=config set-cluster <cluste name> --server=<kubernetes server address>

kubectl config --kubeconfig=config set-credentials <username> --token=<user token>

kubectl config --kubeconfig=config set-context <context name> --cluster=<cluster name> --user=<username>

kubectl config --kubeconfig=config use-context <current context>

```

通过上面的命令生成config文件，通过这个文件就可以使用kubectl访问kubernetes集群

#### 生成证书
```
kubectl create secret tls <cert name> --cert <hostname.crt> --key <hostname.key> -n <namespace>
```

生成证书之后就可以在ingress配置spec.tls.hosts.secretName中配置命令中指定的<cert name>

#### 生成secret

```
# <secret file>这里常用的就是property文件，内容是key=value
kubectl create secret generic <secret name> --from-env-file=<secret file> -n <namespace>

# 删除secret
kubectl delete secret <secret name> -n <namespace>

# 查看有哪些secret:
kubectl get secrets -n <namespace>

# 查看secret详细信息
kubectl describe secrets <secret name> -n <namespace>

```

secret数据字段存储的是源字符串使用base64编码的。你可以使用下面的命令对其进行base64编码和解码

```
# 编码
echo -n '1f2d1e2e67df' | base64

# 解码
echo 'MWYyZDFlMmU2N2Rm' | base64 --decode

```

#### 查看有哪些pod

```
kubectl get pods -n <namespace> -o wide
```

#### 查看deployment

```
kubectl describe deploy/<deployment name> -n <namespace>
```

#### 查看某个pod日志

```
kubectl logs <pod name> -n <namespace>
```

#### 进入到容器内部

```
winpty kubectl exec <pod name> -n <namespace> -it  -- bin/bash
```

windows下执行才需要加上winpty,mac是不需要的。 如果bin/bash没有，可以指定bin/sh。

如果你的pod中部署了多个container,你想进入到某一个中，可以使用下面命令

```
winpty kubectl exec <pod name> -c <container name> -n <name space> -it -- bin/sh
```

#### 水平扩展/收缩

```
kubectl scale deployment <deployment name> --replicas=4
```

#### 查看滚动更新结果

```
# 查看deployment
kubectl describe deployment <deployment name> -n <namespace>

# 或者查看rs
kubectl get rs -n <namespace>

```

#### 查看历史版本

```
kubectl rollout history deployment/<deployment name> -n <namespace>
```

#### 查看某个历史版本的细节

```
# 这里的versionNum是通过rollout history查看到的对应的数字
kubectl rollout history deployment/<deployment name> --revision=<versionNum>
```

#### 回滚到某个特定版本

```
kubectl rollout undo deployment/smcp-service --to-revision=<versionNum>
```

#### 删除rs

```
kubectl delete rs -l app=<label name> -n <namespace>

```
每次更新都会产生一个新的replica set,如果要删除那些旧的，可以使用这个命令，并通过-l参数指定label来进行删除

### 将服务器上的服务暴露到本地

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

当我们访问`http://localhost:8080`的时候，实际上就是访问的kubernetes集群中argocd-server的443端口，同理你可以暴露redis,mongodb等。

#### 总结

其实kubectl语法都是存在特定格式的

```
kubectl [command] [TYPE] [NAME] [flags]
```

写命令忘记的时候，可以使用kubectl -h来查看支持的命令，关于这些命令可以参考 [官方文档](https://kubernetes.io/docs/reference/kubectl/overview/)