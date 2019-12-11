---
title: github集成drone CI的那些事儿
date: 2019-12-10 10:56:00
tags:
- drone
- CI
---


基于Drone来做CI/CD,个人感觉真的很棒,相对于业界老大哥jekins,我更喜欢drone,相比较而言，我觉得它主要有以下优势

1. 插件不需要额外管理
2. 基于yaml文件，易编写,配置可以进行版本管理
3. 可以根据不同的条件进行构建
4. 更人性化的UI界面

对于我们后端的java项目而言，我们CI要做什么呢?

![CI流程](/images/drone/ci.png)

<!--more-->

>一般提交代码流程如下
>1. Clone 项目到本地，创建一个分支来完成新功能的开发, git checkout -b feature/sync-status。在这个分支修改一些代码
>2. git add .，书写符合规范的 Commit 并提交代码， git commit -m "sync article status”
>3. 将代码推送到代码库的对应分支， git push origin feature/sync-status
>4. 如果功能已经开发完毕，可以向 Develop(或者Master) 分支发起一个 Pull Request，并让项目的负责人 Code Review
>5. Review 通过后，项目负责人将分支合并入主干分支

从上图中可以看到当我们提交代码时,会执行整个CI流程，需要注意的有以下2点

1. 执行build或者unit test的时候，如果失败，会发送消息到Slack,这个时候开发人员就能注意到这个问题，当然也可以使用发送邮件或者微信的方式
2. 执行SonarQube check的时候,如果存在问题会将结果回写到github中,开发人员就会去看这个问题

先看下SonarQube回写到github中的的检查结果是怎样的

![SonarQube Feed Back](/images/drone/sonar-qube-check.png)


当sonar qube检测完成之后,会将检查结果通过oauth的方式发送给github,所以你需要在github中创建Personal access token(这个要记下来)
>当你激活你的代码仓库时，Drone会自动将Webhooks添加到版本控制系统中,例如GitHub,而无需手动配置

```yaml
kind: pipeline
name: default

steps:
# build for push and pull_request
- name: build-pr
  image: maven:latest
  pull: if-not-exists
  commands:
  - mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent install -Dmaven.test.skip=true -s settings.xml
  when:
    branch:
    - feature/*
    - issue/*
    - develop
    event:
    - push
    - pull_request

- name: unittest
  image: maven:latest
  pull: if-not-exists
  commands:
  - mvn test -s settings.xml
  when:
    branch:
    - develop
    event:
      include:
      - pull_request
      - push

# 这里我们使用命令来深度定制我们的扫描，而不是使用drone sonar plugin
- name: sonar-scan
  image: newtmitch/sonar-scanner:4.0.0-alpine
  environment:
    SONAR_TOKEN:
      from_secret: sonar_token
    GITHUB_ACCESS_TOKEN_FOR_SONARQUBE:
      from_secret: github_access_token_for_sonarqube
  commands:
  - >
    sonar-scanner
    -Dsonar.host.url=https://sonarqube.company-beta.com/
    -Dsonar.login=$$SONAR_TOKEN
    -Dsonar.projectKey=smcp-service-BE
    -Dsonar.projectName=smcp-service-BE
    -Dsonar.projectVersion=${DRONE_BUILD_NUMBER}
    -Dsonar.sources=src/main/java
    -Dsonar.tests=src/test/java
    -Dsonar.language=java
    -Dsonar.java.coveragePlugin=jacoco
    -Dsonar.modules=smcp-api,smcp-web
    -Dsonar.java.binaries=target
    -Dsonar.projectBaseDir=.
    -Dsonar.analysis.mode=preview
    -Dsonar.github.repository=Today_Group/SMCP-Service
    -Dsonar.github.oauth=$$GITHUB_ACCESS_TOKEN_FOR_SONARQUBE
    -Dsonar.github.pullRequest=${DRONE_PULL_REQUEST}
    -Dsonar.github.disableInlineComments=false
  when:
    event:
    - pull_request
    branch:
    - develop

# post sonarscan result back to git PR (not in preview mode)
- name: sonar-scan-feedback
  image: newtmitch/sonar-scanner:4.0.0-alpine
  environment:
    SONAR_TOKEN:
      from_secret: sonar_token
    GITHUB_ACCESS_TOKEN_FOR_SONARQUBE:
      from_secret: github_access_token_for_sonarqube
  commands:
    - >
      sonar-scanner
      -Dsonar.host.url=https://sonarqube.company-beta.com/
      -Dsonar.login=$$SONAR_TOKEN
      -Dsonar.projectKey=smcp-service-BE
      -Dsonar.projectName=smcp-service-BE
      -Dsonar.projectVersion=${DRONE_BUILD_NUMBER}
      -Dsonar.sources=src/main/java
      -Dsonar.tests=src/test/java
      -Dsonar.language=java
      -Dsonar.java.coveragePlugin=jacoco
      -Dsonar.modules=smcp-api,smcp-web
      -Dsonar.java.binaries=target
      -Dsonar.projectBaseDir=.
      -Dsonar.analysis.gitRepo=Today_Group/SMCP-Service
      -Dsonar.analysis.pullRequest=${DRONE_PULL_REQUEST}
  when:
    event:
      - pull_request
    branch:
      - develop

```

上面drone的配置就是整个CI的基本流程了，需要注意的有以下几点

1. 只有当分支名称以feature/,issue/,develop开头的才会触发上面的执行步骤,对于unit test而言,只有develop分支才生效(可以根据需要自行定制)
2. sonar配置中的sonar.projectKey,sonar.projectName一定要和你在sonar服务器(sonar.host.url指定的地址)中创建project时的名称一样
3. sonar_token的值是在sonar服务器上创建的，然后将这个值设置在了drone的secrets中(drone中点击某一个仓库,进入Settings可以进行设置)
4. github token和sonar_token是同样的方式，都需要在drone中预设置(好处就是你不会暴露你的密码在文件中，这样更加安全)
5. 由于所用Java项目是个多模块项目,所以可以在sonar.modules中指定多个模块名称
6. sonar scan feedback的内容到pr不要指定preview mode
7. build的时候使用了jacoco(分析单元测试覆盖率),所以需要在java项目中pom.xml引入这个plugin

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>${jacoco.version}</version>
  <executions>
    <execution>
      <id>prepare-agent</id>
      <goals>
          <goal>prepare-agent</goal>
      </goals>
    </execution>
    <execution>
      <id>default-report</id>
      <phase>test</phase>
      <goals>
          <goal>report</goal>
      </goals>
      <configuration>
          <dataFile>target/jacoco.exec</dataFile>
          <outputDirectory>target/jacoco</outputDirectory>
      </configuration>
    </execution>
  </executions>
</plugin>

```


**其他可能遇到的问题：**

1. ci执行完成之后，如何发送邮件或者消息到微信群

  答: drone提供了关于[邮件](http://plugins.drone.io/drillster/drone-email/)和[微信的插件](http://plugins.drone.io/clem109/drone-wechat/)

2. sonarqube能否集成阿里巴巴的p3c或者自定义checkstyle

  答: 没有p3c的插件，但是可以通过PMD来进行集成
> 集成p3c: https://www.jianshu.com/p/a3a58ac368be
> 自定义checkstyle: https://www.jianshu.com/p/a3a58ac368be

3. 我想根据build的信息(是否成功,时间等)自己做统计怎么办?

  答: drone提供了webhooke的plugin,你只需要编写自己统计的程序就可以了,可以根据模板设置需要发送的信息

4. 没有我想要的插件,怎么办?

  答: 可以自己写一个插件,官网有bash/go的示例,用你熟悉的语言也是可以的