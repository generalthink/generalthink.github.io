---
title: 好家伙，这些问题还不是手拿把掐
date: 2024-02-18 10:44:01
tags: [工具,Minio, Azure]
---

总结了下这段时间遇到的问题。


### 快速生成字典表数据

在前期开发的时候，BA总是给我好几张excel,让我生成字典表,写代码又耗时，而且不同的excel字段也不一样，不可能每次都要去改代码吧，总之我不干，好在我能借助excel函数完成这样的需求。

![函数](/images/excel.gif)

```
=CONCATENATE("insert into test_claims(`id`,`code`,`name`) values('", A1, "','",B1, "','",C1,"');")

这个函数的语法是 CONCATENATE(text1, [text2], ...)
1. text1（必需）：要联接的第一个项目。项目可以是文本值、数字或单元格引用；

2. Text2, ... （可选）：要联接的其他文本项目。最多可以有 255 个项目，总共最多支持 8,192 个字符
```

### Kubernetes Pod频繁重启

后台看到部署到kubernetes的pod一直在重启，但是看日志没有报错，但是一会儿它就自动重启了，最后通过describe命令看到是因为liveness接口的原因

```
kubelet  Liveness probe failed: Get "http://10.24.8.84:9202/actuator/health"
```

因为使用了springboot actuator接口,它会检测服务中使用到的其他服务是否能正常使用,从而判定当前服务是否存活，所以必然是因为这个接口返回的信息导致pod重启。

重启的这个服务主要用到了邮件以及Redis,但是不知道到底是哪个服务健康检查失败了。此时我们也无法进入到pod中访问health接口了。

所以我们先移除掉pod template的livenessProbe配置，然后重新部署服务

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health
    port: 9204
  initialDelaySeconds: 60
  periodSeconds: 20
  timeoutSeconds: 10
```

这个时候使用exec命令进入pod

```bash

kubectl exec -it <pod name> -n <namespace> -- /bin/bash
```

访问  `curl -i http://localhost:9202/actuator/health`  可以看到response status code是503,并且还可以看到具体失败的组件是哪一个。 

最终确定是redis访问超时了，于是调整了下 livenessProbe的timeoutSeconds,然后在重启就不报错了。

```
root@noti-844567c558-2mvj8:/home/pro# curl -i http://localhost:9202/actuator/health
HTTP/1.1 200 
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'; script-src 'self'; frame-ancestors 'self'; object-src 'none'
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Content-Type: application/vnd.spring-boot.actuator.v3+json;charset=utf-8
Transfer-Encoding: chunked
Date: Sun, 18 Feb 2024 07:36:22 GMT

{"status":"UP","components":{"discoveryComposite":{"description":"Discovery Client not initialized","status":"UNKNOWN","components":{"discoveryClient":{"description":"Discovery Client not initialized","status":"UNKNOWN"}}},"diskSpace":{"status":"UP","details":{"total":133003395072,"free":94000709632,"threshold":10485760,"exists":true}},"livenessState":{"status":"UP"},"mail":{"status":"UP","details":{"location":"xxmail:465"}},"ping":{"status":"UP"},"readinessState":{"status":"UP"},"redis":{"status":"UP","details":{"version":"6.0.14"}},"refreshScope":{"status":"UP"}},"groups":["liveness","readiness"]}

```

### minio数据迁移到Azure blob

项目中需要将minio中保存的文件迁移到azure的blob中,然后minio中保存文件的路径是这样的 `2023/11/07/20231107164256A579/Image3.jpg`  相当于文件夹中包含了时间戳等信息，虽然blob不支持目录，但是它的虚拟目录可以有相同的效果，这里我们使用azure提供的azcopy命令来进行数据迁移。

首先进入到minio所在的服务器,然后执行下面的命令

```

1.  sudo mkdir /home/azcopy 

2. cd /home/azcopy
 
3. sudo  wget -O azcopy_v10.tar.gz https://aka.ms/downloadazcopy-v10-linux &&

4. sudo tar -xf azcopy_v10.tar.gz --strip-components=1

5. sudo azcopy login  // 登录azure


6. 同步minio数据到blob,将sc-dev bucket的数据迁移到blob的container


	// 这里的sc-dev是minio的bucket, saoscdev是azure blob的account name, bvsc是container name
	a. sudo /home/azcopy/azcopy copy '/data/minio/data/sc-dev/*' 'https://saoscdev.blob.core.windows.net/bvsc' --recursive

	// 将增量数据同步到blob中
	b. sudo /home/azcopy/azcopy sync '/data/minio/data/sc-dev' 'https://saoscdev.blob.core.windows.net/bvsc' --recursive
```


### 总结

以上就是我最近遇到的问题，第一个字典表的那个当时第一反应是写代码，但是后来想到写代码的时候太长，成本太高还是需要借助工具，恰好提供给我的又是excel,于是就用excel顺带完成了这个功能，同时后面的其他字典数据如法炮制，也就变简单了。

第二个问题本来是定位重启的问题，但是看着看着就深入到了actuator的源码当中起了，这一点很不好，不过有失必有得，有顺带看了下actuator的源码,它其中的EntryPoint感觉很棒，后面针对这个写一篇。

第三个问题的话就是微软没有提供如何迁移的文档，不过好在它本生工具不少，多尝试也就成功了，这里我把迁移的步骤给出来，希望能帮助到同样有需求的人。



