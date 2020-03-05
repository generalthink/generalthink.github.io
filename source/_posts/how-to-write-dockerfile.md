---
title: Dockerfile的使用以及实践
date: 2020-02-14 16:20:22
tags: docker
---

Dockerfile是Docker用来构建镜像的文本文件,包括自定义的指令和格式。可以通过docker build命令从Dockerfile中构建镜像。用户可以通过统一的语法命令来根据需求进行配置，通过这份统一的配置文件，在不同的文件上进行分发，需要使用时就可以根据配置文件进行自动化构建，这解决了开发人员构建镜像的复杂过程。

<!--more-->
### Dockerfile的使用

Dockerfile描述了组装对象的步骤，其中每条指令都是单独运行的。除了FROM指令，其他每条命令都会在上一条指令所生成镜像的基础上执行，执行完后会生成一个新的镜像层，新的镜像层覆盖在原来的镜像之上从而形成了新的镜像。Dockerfile所生成的最终镜像就是在基础镜像上面叠加一层层的镜像层组建的。

### Dockerfile指令

Dockerfile的基本格式如下:

```
# Comment
INSTRUCTION arguments
```

在Dockerfile中,指令(INSTRUCTION)不区分大小写，但是为了与参数区分，推荐大写。
Docker会顺序执行Dockerfile中的指令，第一条指令必须是FROM指令，它用于指定构建镜像的基础镜像。在Dockerfile中以#开头的行是注释，而在其他位置出现的#会被当成参数。

Dockerfile中的指令有FROM、MAINTAINER、RUN、CMD、EXPOSE、ENV、ADD、COPY、ENTRYPOING、VOLUME、USER、WORKDIR、ONBUILD,错误的指令会被忽略。下面将详细讲解一些重要的Docker指令。

#### FROM

格式: `FROM <image> 或者 FROM <image>:<tag>`

FROM指令的功能是为后面的指令提供基础镜像,因此Dockerfile必须以FROM指令作为第一条非注释指令。从公共镜像库中拉取镜像很容易,基础镜像可以选择任何有效的镜像。
在一个Dockerfile中FROM指令可以出现多次,这样会构建多个镜像。tag的默认值是latest,如果参数image或者tag指定的镜像不存在，则返回错误。

#### ENV

格式: `ENV <key> <value> 或者 ENV <key>=<value> ...`

ENV指令可以为镜像创建出来的容器声明环境变量。并且在Dockerfile中，ENV指令声明的环境变量会被后面的特定指令(即ENV、ADD、COPY、WORKDIR、EXPOSE、VOLUME、USER)解释使用。

其他指令使用环境变量时，使用格式为`$variable_name`或者`${variable_name}`。如果在变量面前添加斜杠\可以转义。如`\$foo`或者`\${foo}`将会被转换为`$foo`和`${foo}`,而不是环境变量所保存的值。另外，ONBUILD指令不支持环境替换。

#### COPY

格式: `COPY <src> <dest>`

COPY指令复制<src>所指向的文件或目录,将它添加到新镜像中,复制的文件或目录在镜像中的路径是`<dest>`。`<src>`所指定的源可以有多个,但必须是上下文根目录中的相对路径。
不能只用形如 ` COPY ../something /something`这样的指令。此外,`<src>`可以使用通配符指向所有匹配通配符的文件或目录，例如，COPY home* /mydir/ 表示添加所有以"hom"开头的文件到目录/mydir/中。

`<dest>`可以是文件或目录，但必须是目标镜像中的绝对路径或者相对于WORKDIR的相对路径(WORKDIR即Dockerfile中WORKDIR指令指定的路径,用来为其他指令设置工作目录)。
若`<dest>`以反斜杠/结尾则其指向的是目录；否则指向文件。`<src>`同理。若`<dest>`是一个文件，则`<src>`的内容会被写到`<dest>`中；否则`<src>`指向的文件或目录中的内容会被复制添加到`<dest>`目录中。
当`<src>`指定多个源时，`<dest>`必须是目录。如果`<dest>`不存在，则路径中不存在的目录会被创建。

#### ADD

格式：`ADD <src> <dest>`

ADD与COPY指令在功能上很相似，都支持复制本地文件到镜像的功能，但ADD指令还支持其他功能。`<src>`可以是指向网络文件的URL,此时若`<dest>`指向一个目录，则URL必须是完全路径，这样可以获得网络文件的文件名filename，该文件会被复制添加到`<dest>/<filename>`。
比如 ADD http://example.com/config.property / 会创建文件/config.property。

`<src>`还可以指向一个本地压缩归档文件，该文件会在复制到容器时会被解压提取，如ADD sxample.tar.xz /。但是若URL中的文件为归档文件则不会被解压提取。

ADD 和 COPY指令虽然功能相似，但一般推荐使用COPY,因为COPY只支持本地文件，相比ADD而言，它更加透明。


#### EXPOSE
格式: `EXPOSE <port> [<port>/<protocol>...]`

EXPOSE指令通知Docker该容器在运行时侦听指定的网络端口。可以指定端口是侦听TCP还是UDP，如果未指定协议，则默认值为TCP。
这个指令仅仅是声明容器打算使用什么端口而已，并不会自动在宿主机进行端口映射,可以在运行的时候通过docker -p指定。

```
EXPOSE 80/tcp
EXPOSE 80/udp
```

#### USER

格式: `USER <user>[:<group] 或者 USER <UID>[:<GID>]`

USER指令设置了user name和user group(可选)。在它之后的RUN,CMD以及ENTRYPOINT指令都会以设置的user来执行。

#### WORKDIR

格式: `WORKDIR /path/to/workdir`

WORKDIR指令设置工作目录，它之后的RUN、CMD、ENTRYPOINT、COPY以及ADD指令都会在这个工作目录下运行。如果这个工作目录不存在，则会自动创建一个。
WORKDIR指令可在Dockerfile中多次使用。如果提供了相对路径，则它将相对于上一个WORKDIR指令的路径。例如

```
WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd
```
输出结果是 /a/b/c



#### RUN

格式1： `RUN <command>` (shell格式)
格式2： `RUN ["executable", "param1", "param2"]` (exec格式，推荐使用)

RUN指令会在前一条命令创建出的镜像的基础上创建一个容器，并在容器中运行命令，在命令结束运行后提交容器为新镜像，新镜像被Dockerfile中的下一条指令使用。

RUN指令的两种格式表示命令在容器中的两种运行方式。当使用shell格式时，命令通过/bin/sh -c运行。
当使用exec格式时，命令是直接运行的，容器不调用shell程序，即容器中没有shell程序。
exec格式中的参数会被当成JSON数组被Docker解析，故必须使用双引号而不能使用单引号。因为exec格式不会在shell中执行，所以环境变量的参数不会被替换。

比如执行`RUN ["echo", "$HOME"]`指令时，$HOME不会做变量替换。如果希望运行shell程序，指令可以写成 `RUN ["/bin/bash", "-c", "echo", "$HOME"]`。

#### CMD

CMD指令有3种格式。

格式1：`CMD <command>` (shell格式)
格式2：`CMD ["executable", "param1", "param2"]` (exec格式，推荐使用)
格式3：`CMD ["param1", "param2"]` (为ENTRYPOINT指令提供参数)

CMD指令提供容器运行时的默认值，这些默认值可以是一条指令，也可以是一些参数。一个Dockerfile中可以有多条CMD指令，但只有最后一条CMD指令有效。 
CMD ["param1", "param2"]格式是在CMD指令和ENTRYPOINT指令配合时使用的，CMD指令中的参数会添加到ENTRYPOING指令中.使用shell和exec格式时，命令在容器中的运行方式与RUN指令相同。

不同之处在于，RUN指令在构建镜像时执行命令，并生成新的镜像；CMD指令在构建镜像时并不执行任何命令，而是在容器启动时默认将CMD指令作为第一条执行的命令。如果用户在命令行界面运行docker run命令时指定了命令参数，则会覆盖CMD指令中的命令。

#### ENTRYPOINT

ENTRYPOINT指令有两种格式。

格式1：`ENTRYPOINT <command>` (shell格式)
格式2：`ENTRYPOINT ["executable", "param1", "param2"]` (exec格式，推荐格式)

ENTRYPOINT指令和CMD指令类似，都可以让容器在每次启动时执行相同的命令，但它们之间又有不同。一个Dockerfile中可以有多条ENTRYPOINT指令，但只有最后一条ENTRYPOINT指令有效。

当使用Shell格式时，ENTRYPOINT指令会忽略任何CMD指令和docker run命令的参数，并且会运行在bin/sh -c中。这意味着ENTRYPOINT指令进程为bin/sh -c的子进程,进程在容器中的PID将不是1，且不能接受Unix信号。即当使用`docker stop <container>`命令时，命令进程接收不到SIGTERM信号。

推荐使用exec格式，使用此格式时，docker run传入的命令参数会覆盖CMD指令的内容并且附加到ENTRYPOINT指令的参数中。从ENTRYPOINT的使用中可以看出，CMD可以是参数，也可以是指令，而ENTRYPOINT只能是命令；另外，docker run命令提供的运行命令参数可以覆盖CMD,但不能覆盖ENTRYPOINT。

### Dockerfile实践心得

#### 使用标签

给镜像打上标签，有利于帮助了解进镜像功能

#### 谨慎选择基础镜像

选择基础镜像时，尽量选择当前官方镜像库的肩宽，不同镜像的大小不同，目前Linux镜像大小由如下关系:

`busybox < debian < centos < ubuntu`

同时在构建自己的Docker镜像时,只安装和更新必须使用的包。此外相比Ubuntu镜像，更推荐使用Debian镜像，因为它非常轻量级(目前其大小是在100MB以下),并且仍然是一个完整的发布版本。

#### 充分利用缓存
Docker daemon会顺序执行Dockerfile中的指令，而且一旦缓存失效，后续命令将不能使用缓存。为了有效地利用缓存，需要保证指令的连续性，尽量将所有Dockerfile文件相同的部分都放在前面，而将不同的部分放到后面。

#### 正确使用ADD与COPY命令

当在Dockerfile中的不同部分需要用到不同的文件时，不要一次性地将这些文件都添加到镜像中去，而是在需要时添加，这样也有利于重复利用docker缓存。
另外考虑到镜像大小问题，使用ADD指令去获取远程URL中的压缩包不是推荐的做法。应该使用RUN wget或RUN curl代替。这样可以删除解压后不在需要的文件，并且不需要在镜像中在添加一层。

错误做法:

```
ADD http://example.com/big.tar.xz /usr/src/things/
RUN tar -xJf /usr/src/things/big.tar.xz -C /usr/src/things
RUN make -C /usr/src/things all
```

正确的做法:

```
RUN mkdir -p /usr/src/things \
    && curl -SL http://example.com/big.tar.xz \
    | tar -xJC /usr/src/things \
    && make -C /usr/src/things all
```

#### RUN指令

在使用较长的RUN指令时可以使用反斜杠\分隔多行。大部分使用RUN指令的常见是运行apt-wget命令，在该场景下请注意以下几点。

1. 不要在一行中单独使用指令RUN apt-get update。当软件源更新后，这样做会引起缓存问题，导致RUN apt-get install指令运行失败。所以,RUN apt-get update和RUN apt-get install应该写在同一行。比如 RUN apt-get update && apt-get install -y package-1 package-2 package-3

2. 避免使用指令RUN apt-get upgrade 和 RUN apt-get dist-upgrade。因为在一个无特权的容器中，一些必要的包会更新失败。如果需要更新一个包(如package-1)，直接使用命令RUN apt-get install -y package-1。

#### CMD和ENTRYPOINT命令

CMD和ENTRYPOINT命令指定是了容器运行的默认命令，推荐二者结合使用。使用exec格式的ENTRYPOINT指令设置固定的默认命令和参数，然后使用CMD指令设置可变的参数。

比如下面这个例子:

```
FROM busybox
WORKDIR /app
COPY run.sh /app
RUN chmod +x run.sh
ENTRYPOINT ["/app/run.sh"]
CMD ["param1"]
```

run.sh内容如下:
```
#!/bin/sh
echo "$@"
```

运行后输出结果为param1, Dockerfile中CMD和ENTRYPOINT的顺序不重要(CMD写在ENTRYPOINT前后都可以)。

当在windows系统下build dockerfile你可能会遇到这个问题

```
standard_init_linux.go:207: exec user process caused "no such file or directory"
```
这是因为sh文件的fileformat是dos,这里需要修改为unix,不需要下载额外的工具，一般我们机器上安装了git会自带git bash,进入git bash,使用vi 编辑，在命令行模式下修改(:set ff=unix)。

#### 不要再Dockerfile中做端口映射

使用Dockerfile的EXPOSE指令，虽然可以将容器端口映射在主机端口上，但会破坏Docker的可移植性，且这样的镜像在一台主机上只能启动一个容器。所以端口映射应在docker run命令中用-p 参数指定。

```
# 不要再Dockerfile中做如下映射
EXPOSE 80:8080

# 仅暴露80端口,需要另做映射
EXPOSE 80

```

### 实践Dockerfile的写法

#### Java 服务的DockerFile

```bash
FROM openjdk:8-jre-alpine
ENV spring_profiles_active=dev
ENV env_java_debug_enabled=false
EXPOSE 8080
WORKDIR /app
ADD target/smcp-web.jar /app/target/smcp-web.jar
ADD run.sh /app
ENTRYPOINT ./run.sh
```
可以看到基础镜像是openjdk,然后设置了两个环境变量,服务访问端口是9090(意味着springboot应用中指定了server.port=8080),设置了工作目录是/app。通过ENTRYPOINT设定了启动镜像时要启动的命令(./run.sh)。这个脚本中的内容如下:

```
#!/bin/sh
# Set debug options if required
if [ x"${env_java_debug_enabled}" != x ] && [ "${env_java_debug_enabled}" != "false" ]; then
    java_debug_args="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005"
fi

# ex: env_jvm_flags="-Xmx1200m -XX:MaxRAM=1500m" for production
java $java_debug_args $env_jvm_flags -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -jar target/smcp-web.jar
```
如果我们要指定jvm的一些参数,可以通过在环境变量中设置env_jvm_flags来指定。

#### Maven Dockerfile

maven的Dockerfile也写的很好，这里我发上来也给大家参考下

```
FROM openjdk:8-jdk

ARG MAVEN_VERSION=3.6.3
ARG USER_HOME_DIR="/root"
ARG SHA=c35a1803a6e70a126e80b2b3ae33eed961f83ed74d18fcd16909b2d44d7dada3203f1ffe726c17ef8dcca2dcaa9fca676987befeadc9b9f759967a8cb77181c0
ARG BASE_URL=https://apache.osuosl.org/maven/maven-3/${MAVEN_VERSION}/binaries

RUN mkdir -p /usr/share/maven /usr/share/maven/ref \
  && curl -fsSL -o /tmp/apache-maven.tar.gz ${BASE_URL}/apache-maven-${MAVEN_VERSION}-bin.tar.gz \
  && echo "${SHA}  /tmp/apache-maven.tar.gz" | sha512sum -c - \
  && tar -xzf /tmp/apache-maven.tar.gz -C /usr/share/maven --strip-components=1 \
  && rm -f /tmp/apache-maven.tar.gz \
  && ln -s /usr/share/maven/bin/mvn /usr/bin/mvn

ENV MAVEN_HOME /usr/share/maven
ENV MAVEN_CONFIG "$USER_HOME_DIR/.m2"

COPY mvn-entrypoint.sh /usr/local/bin/mvn-entrypoint.sh
COPY settings-docker.xml /usr/share/maven/ref/

ENTRYPOINT ["/usr/local/bin/mvn-entrypoint.sh"]
CMD ["mvn"]
```
可以看到它是基于openjdk这个基础镜像来创建的，先去下载maven的包，然后进行了安装。 然后又设置了MAVEN_HOME和MAVEN_CONFIG这两个环境变量，最后通过mvn-entrypoing.sh来进行了启动。

#### 前端服务的两阶段构建

我有一个前端服务，目录结构如下：

```
$ ls frontend/
myaccount/  resources/  third_party/
```

myaccount目录下是放置的js,vue等，resources放置的是css,images等。third_party放的是第三方应用。

这里采用了两阶段构建，即采用上一阶段的构建结果作为下一阶段的构建数据

```
FROM node:alpine as builder
WORKDIR '/build'
COPY myaccount ./myaccount
COPY resources ./resources
COPY third_party ./third_party

WORKDIR '/build/myaccount'

RUN npm install
RUN npm rebuild node-sass
RUN npm run build

RUN ls /build/myaccount/dist

FROM nginx
EXPOSE 80
COPY --from=builder /build/myaccount/dist /usr/share/nginx/html
```

需要注意结尾的 `--from=builder`这里和开头是遥相呼应的。

### 总结

我相信看完dockerfile指令,你看任何一个dockerfile应该都没有太大问题,不记得的命令回来翻一下就行了。如果你觉得还可以，关注下哟。 公众号: think123