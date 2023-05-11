---
title: 这次上线我花了12个小时
date: 2023-05-11 09:58:08
tags: [java]
---

### 迁移思路

这次上线比较复杂,线上有两个服务,A和B, A服务部署在生产环境(姑且叫做Prod-1), B服务部署到QA环境,当时客户催得急,被当做生产使用了.现在要做的就是将A,B两个服务迁移到C这个正式的生产环境(称作Prod-2).

服务主要依赖两个外部组件,一个是minio,一个是mysql数据库(部署到Azure云平台上), 经过讨论  Prod-2环境要使用的minio仍然沿用Prod-1使用的,mysql数据库则使用新的.

意味着我们需要将QA环境的minio数据迁移到Prod-2的minio(和Prod-1使用的同一个), 需要将QA环境的mysql数据和Prod-1环境的mysql数据迁移到Prod-2,但是由于QA和Prod-1是不同的两个服务,所用的表均不一样,所以sql没有冲突.

<!-- more -->


### minio迁移

minio数据是以文件方式存储到服务器上的,所以我们只需要将文件迁移过来,然后保证路径不变就行了(bucket name变动没有问题),因此最开始就调研到了两种方案,一种是使用scp进行文件迁移,一种是使用rclone,但是QA环境minio和prod-1环境所在minio网络不能直接通,让运维给开通,他说有问题(这里我当时没多想下,多想下就不得有这样问题了),所以就只能采用scp方式. 先将QA minio数据拷贝到跳板机,然后从跳板机拷贝到生产环境. 


#### SCP & CP

使用的SCP命令如下

```bash
1. 将文件从远程复制到本地
// 将qa环境中的minio数据复制到当前目录(在跳板机运行的命令), sc-qa指的是bucket名字
 sudo scp -r  scadmin@10.2.0.4:/data/minio/data/sc-qa .

// 将文件夹复制到prod环境所在目录(在跳板机运行命令),先放到一个目录中
sudo scp -r sc-qa/ scadmin@10.2.0.5:/home/scadmin/sc-qa

// 下面都在prod执行,先将现在的minio数据备份
sudo cp -r  /data/minio/data/sc-prod /data/minio/data/sc-prod-bak-20230311-zy

// 将sc-qa的数据递归开呗到sc-prod中,如果存在重复数据,会在文件名后添加一个 ~1~
sudo cp -frap --backup=number /home/scadmin/sc-qa/* /data/minio/data/sc-prod/

// 找出重复数据
find /data/minio/data/sc-prod -type f -regex ".*\.~1~"
```

找到重复数据后,这个时候我们只能重命名文件名,然后修改对应数据库记录,不过好在,未发现重复记录.

最开始使用SCP方式文件大小和拷贝前一致,但是在界面就是打不开,说文件已被损坏,后来通过对比发现,虽然文件大小一样,但是不知道为什么拷贝过来的文件中但是有些字节发生了改变,导致文件无法打开了.

于是考虑使用winscp来移动数据,但是文件比较大,网络又有问题,总是中断,在重试了2个小时后作罢.

开始怀疑scp有什么问题,后来让运维帮我把目录导过来,还是一样的问题,这个方案只能就此作罢,但是我在dev环境下使用scp是可以的,不知道为什么,如果有知道的老哥麻烦给我讲下.

此时时间来到了下午.

#### rclone

后面没办法只能采用rclone的方式,之前不是说两个网络不能通吗? 后来想想就让运维将qa环境开了一个外网,这样就可以了.于是开始使用rclone,首先需要安装好rclone, 我的rclone是安装在用户目录下的,所以编辑用户目录下的 `~/.config/rclone/rclone.conf`文件

配置如下

```
[oldminio]
type = s3
provider = Minio
env_auth = false
access_key_id = bvfs
secret_access_key = BvFs2022
region = cn-east-1
# qa环境的minio外网地址
endpoint = http://20.xx.xx.xx:9000
location_constraint =
server_side_encryption =

[newminio]
type = s3
provider = Minio
env_auth = false
access_key_id = minioadmin
secret_access_key = minioadmin
region = cn-east-1
// 生产环境minio地址
endpoint = http://10.1.1.5:9000
location_constraint =
server_side_encryption =
```

解释下access_key_id和secret_access_key这两个值是你项目中配置minio的accessKey和secretKey的值.

上面的配置文件是在prod minio服务器配置的.

然后使用 rclone copy命令进行文件复制,因为我们不能删除prod目录的文件.在运行这个命令之前,强烈建议使用 --dry-run 来测试下(它不会copy任何东西)

使用rclone后,minio文件同步成功,且能正确打开.


### Mysql同步

以为mysql同步总要好点,结果还是很坑. 因为我们的mysql也是不同的环境,而且使用navicate无法连接上,于是我只能使用idea自带的database进行连接,先将qa数据库数据导出,然后在导入. 

但是导入的时候发现太慢了(一个文件一千多行sql),经常一个文件执行就半个小时,可是好多个文件,就算把idea的事务换成手动提交还是很慢,这里又折腾了许久,发现自己还是不行.

只能把sql给运维,让他到mysql所在服务器执行去了,通过source命令, 它几分钟就搞定了.

但是进行验证的时候,发现有些数据序号错乱了,因为数据库有些表字段是自增的,在idea中导出的时候不会导出自增字段,这就导致了在源数据库的自增值和新的不一致,因为源数据库的数据可能会删除掉,然后自增值就不是连续的了,导入到新的库,这个值就变成连续的,就知道两边不一致,这个坑也把我整到了,所以又只能重新导数据.


### 总结

下班的时候已经是晚上十点钟了,宵夜都吃过一次了,回首这次上线感觉被坑了许多,复盘这次上线,最重要的是记得要在dev环境充分预演下上线流程,一定要使用相同方案!



