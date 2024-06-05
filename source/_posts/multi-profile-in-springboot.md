---
title: SpringBoot中使用profile的几种方式
date: 2024-06-05 09:58:23
tags: [maven, SpringBoot, profile]
---

最近项目中进行仓库拆分了之后,因为引入了公共包,所以就存在可能有snapshot版本以及release版本问题,比如我想要在dev环境的时候import snapshot版本,prod环境的时候又使用release版本，为了不频繁修改pom.xml文件，因此决定使用POM的profile来解决这个问题。 

当然由于maven默认是不下载snapshot包的，因此我们要配置让它下载，这里分为全局配置和项目级别配置

### 项目配置

在pom文件中添加如下内容

```xml
<repositories>
    <repository>
        <id>nexus</id>
        <!--修改成自己私服地址-->
        <url>http://localhost:18081/repository/maven-public/</url>
        <releases>
            <enabled>true</enabled>
            <updatePolicy>always</updatePolicy>
        </releases>
        <snapshots>
        	<!--主要是这里-->
            <enabled>true</enabled>
            <updatePolicy>always</updatePolicy>
        </snapshots>
    </repository>
 </repositories>

```

<!--more-->

### 全局配置

在maven的settings.xml中新增配置,如果不配置profile，只配置mirror,是下载不了snapshot包的

```xml

<profiles>
<profile>
  <id>bv-profile</id>
   <repositories>
    <repository>
        <id>nexus</id>
        <url>http://localhost:18081/repository/maven-public/</url>
        <releases>
            <enabled>true</enabled>
            <updatePolicy>always</updatePolicy>
        </releases>
        <snapshots>
            <enabled>true</enabled>
            <updatePolicy>always</updatePolicy>
        </snapshots>
    </repository>
  </repositories>
</profile>
</profiles>

```

回到正题，如何使用maven的profile呢？

### maven profile

同样的首先在pom.xml中新增配置

```xml
 <profiles>
    <profile>
        <id>dev</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <spring.profiles.active>dev</spring.profiles.active>
            <bvpro.api.version>2.0.0-SNAPSHOT</bvpro.api.version>
        </properties>
    </profile>
    <profile>
        <id>prod</id>
        <properties>
            <spring.profiles.active>prod</spring.profiles.active>
            <bvpro.api.version>1.9.0</bvpro.api.version>
        </properties>
        </properties>
    </profile>
</profiles>

<resources>
    <resource>
        <!-- 指定配置文件所在的resource目录 -->
        <directory>src/main/resources</directory>
        <includes>
            <include>db/**</include>
            <include>static/**</include>
            <include>mapper/**</include>
            <!-- 选择要打包的配置文件和日志，当然也可以不加打包所有配置文件，然后由具体服务器来选择使用哪个 -->
            <include>bootstrap.yml</include>
            <include>bootstrap-${spring.profiles.active}.yml</include>
            <include>logback.xml</include>
        </includes>
        <filtering>true</filtering>
    </resource>
</resources>
```

比如这里我就设定了两个profile(dev, prod), 然后他们引用某个包的版本不一样，然后应用包的时候这样使用就行了

```xml
<dependencies>
	<dependency>
        <groupId>com.bvpro</groupId>
        <artifactId>bvpro-common-api</artifactId>
        <version>${bvpro.api.version}</version>
    </dependency>
</dependencies>
```
然后项目中的bootstrap.yml文件中修改如下

```yml
# Spring
spring: 
  profiles:
    # 环境配置
    active: @spring.profiles.active@

```

然后当我们打包的时候 `mvn clean package -P dev` 就可以打包dev环境的配置，同时 @spring.profiles.active@ 会被替换成dev

当然如果你使用idea的时候就更加简单了，只需要勾选profile=dev就行了。


### docker中使用

我们知道在使用dockerfile的时候也可以指定环境变量,然后启动的时候就可以根据环境变量来启动了,比如这样子

```sh
FROM openjdk:8-jre-alpine
ENV env_java_debug_enabled=false
EXPOSE 8080
WORKDIR /app
ADD target/smcp-web.jar /app/target/smcp-web.jar
ADD run.sh /app
ENTRYPOINT ["java","-Dspring.profiles.active=test", "-jar","target/smcp-web.jar"]

```

然后run.sh这样写:


可以看到我们在dockerfile中使用了ENV来指定我们要启动什么环境，但是这种方式太僵硬了, 因为相当于把启动什么环境给写死了，后面想要切换到其他环境的时候还要去修改dockerfile,这种方式看上去不太行呀


### SPRING_PROFILES_ACTIVE

好在springboot项目启动的时候会去读这样的一个特殊环境变量 SPRING_PROFILES_ACTIVE， 如果环境变量中配置了这个值，那么会根据这个值找到对应的profile, 然后启动对应的环境，所以我么可以在启动的时候指定这个环境变量就行了， 所以可以这样子

```sh
FROM openjdk:8-jre-alpine
ENV spring_profiles_active=dev
ENV env_java_debug_enabled=false
EXPOSE 8080
WORKDIR /app
ADD target/smcp-web.jar /app/target/smcp-web.jar
ADD run.sh /app
ENTRYPOINT ["java", "-jar","target/smcp-web.jar"]
```

这里预设了spring_profiles_active的值是dev,当启动的时候你可以更改这个值。


如果你使用docker-compose来启动,那么也可以在docker-compose文件中配置environment,比如

```yml
environment:
   - SPRING_PROFILES_ACTIVE=prod
```

这就是指定profile的各种方式了，使用这种方式我们可以更加精确的控制我们需要启动的环境，并且还是最少的修改。