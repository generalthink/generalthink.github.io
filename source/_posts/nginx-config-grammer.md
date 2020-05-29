---
title: nginx语法入门
date: 2020-05-28 15:54:07
tags: Nginx
---

nginx应该是我们常用到的一个软件了，它的用法和语法也很简单，本文主要介绍nginx语法以及各个模块作用。

### Nginx配置目录

当我们安装好nginx之后，我们主要关注两个文件夹

1. /etc/nginx/conf.d/ 文件夹，是我们进行子配置的配置项存放处，/etc/nginx/nginx.conf 主配置文件会默认把这个文件夹中所有子配置项都引入
> windows下，是对应的安装目录下的conf目录。

2. /usr/share/nginx/html/ 文件夹，通常静态文件都放在这个文件夹，你也可以放到其他地方
> windows下,对应的目录是在安装目录下的html目录。

<!--more-->


### Nginx的常用命令

1. 查看Nginx版本号
```
nginx -V
```

2. nginx帮助命令
```
nginx -h
```

3. 验证配置语法是否正确
```
nginx -t
```

4. 配置文件修改重装载命令
```
nginx -s reload
```

5. 启动nginx
```
start nginx
```

6. 块速停止或关闭nginx
```
nginx -s stop
```

7. 正常停止或关闭(会等到worker处理完成请求后关闭)
```
nginx -s quit
```

>注意windows下需要将nginx.exe加入环境变量,然后才能执行上面的命令。不要双击启动，不然只能从任务列表中删除


### Nginx配置语法

1. 配置文件由指令与指令块构成
2. 每条指定以分号(`;`)结尾,指令与参数间以空格符号分割
3. 指令块以大括号(`{}`)将多条指令组织在一起
4. include语句允许组合多个配置文件以提升可维护性
5. 使用`#`符号添加注释
6. 使用`$`符号使用变量
7. 部分指令参数支持正则表达式


当我们打开nginx.conf文件你会看到和下面类似的结果:

```nginx

# nginx进程数，一般设置成和CPU个数一样
worker_processes  1;

events {
    # 每个进程允许最大并发数
    worker_connections  1024;
}

http {
    # 引入其他配置,mime.types文件存储的是文件扩展名与类型映射表
    include       mime.types;
    default_type  application/octet-stream;

    # 日志格式(使用了变量)
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                     '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    sendfile        on;
    keepalive_timeout  65;

    # 服务器配置
    server {
        # 监听端口
        listen       80;
        # 监听域名
        server_name  localhost;

        # 访问地址
        location / {
            root   html;
            index  index.html index.htm;
        }
    }
}
```

当nginx以上面的配置加载启动后，我们就可以访问 http://localhost这个地址了,然后默认会返回html目录下的index.html文件内容。

nginx的配置块嵌套关系如下:

```
main

events {...}

# 表示这里面的所有内容都由http模块来进行解析,比如mail这种是不起作用的
http {

  # 指定上游服务器
  upstream {...}

  # 用于A/B test
  split_clients {...}

  # 基于已有变量,创建新变量
  map {...}

  # 根据客户端地址创建新变量的geo模块
  geo {...}

  # 服务器
  server {
    if() {...}

    location {
      limit_except {...}
      if() {...}
    }

    location {...}
  }
  server {...}
}

```

### nginx指令

上面列出了一些常用的指令块，但是指令块中可以写哪些指令呢？指令那么多，我需要去背吗？我告诉你完全用不着，记不住的时候查文档就行了。

我们都知道nginx实际是由很多个模块组合到一起的，哪些模块提供了哪些功能一看便知。

首先打开nginx的官方文档(nginx.org/en/docs),从中我们可以看到nginx提供了哪些变量,哪些模块。

![nginx模块](/images/nginx/nginx-module-reference.png)


模块提供了各种功能,基本上看到名字也就明白了提供哪方面的功能。


当我们点开某一个module的时候，如果那个module没有build进去,那么它会告诉你如下信息

![nginx未build module提示](/images/nginx/nginx-not-built-module.png)

> nginx -V 可以查看nginx的配置参数，可以看到除了核心模块之外还添加了哪些模块。


在比如我们查看ngx_http_core_module看看这个模块提供了提供的root指令

![root指令](/images/nginx/nginx-root-order.png)

从上图中示例可以看出来，root指令写的位置是在location指令块中的。但是它还能写到http,server这两个指令块中。

这个指令的context指的是指令能够出现的位置。

>  如果块指令可以在括号内包含其他指令，则将其称为context(上下文,比如event,http,server,location)


```
Syntax: log_format name [escape=default|json|none] string ...;
Default:  
log_format combined "...";
Context:  http


Syntax: access_log path [format [buffer=size] [gzip[=level]] [flush=time] [if=condition]];
access_log off;
Default:  
access_log logs/access.log combined;
Context:  http, server, location, if in location, limit_except

```

比如log_format只能出现在http指令块中，而access_log则可以出现在http和server,location这些指令块中。

你是不是会疑惑，既然一个指令能出现在多个指令块中,那么到底哪个会生效呢？

**在nginx中存储值得指令继承规则是向上覆盖。当子配置存在时，直接覆盖父配置块,子配置不存在时，直接使用父配置块。**

> 存储值的指令指的是指令后面的数据是一个值。 比如 root html; root后面跟的就是一个值。