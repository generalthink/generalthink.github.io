---
title: 拥抱Kubernetes,再见了,SpringBoot cronjob
date: 2020-10-27 16:56:52
tags: Kubernetes
---

项目开发中总是需要执行一些定时任务，比如定时处理数据之后发送邮件，定时更新缓存等等。


### Java定时任务

1. 基于 java.util.Timer 定时器，实现类似闹钟的定时任务
2. 使用 Quartz、elastic-job、xxl-job 等开源第三方定时任务框架，适合分布式项目应用
3. 使用 Spring 提供的一个注解： @Scheduled


项目框架使用的是SpringBoot,所以之前定时任务使用的是SpringBoot中的@Scheduled。可是这种方式并不适合我们现在的cloud环境，为了更加cloud native一点,我删除了使用SpringBoot写的37个定时任务，改为使用Kubernetes cronjob的方式。

<!--more-->

### 定时任务代码编写

```java
public interface Command {
    /**
     * 遵循Unix约定，如果命令执行正常，则返回0；否则为非0。
     */
    int execute(String... args);
}

```
首先定义一个接口，所有具体的定时任务都必须实现该接口。接下来是具体的某一个定时任务

```java
@Component
@Slf4j
public class ProjectCommandLineRunner implements CommandLineRunner {

  Map<String, Command> commandMap = new HashMap<>();

  @Autowired
  private SendEmailCommand sendEmailCommand;

  @PostConstruct
  private void init() {
    commandMap.put("sendEmail", sendEmailCommand);
  }

  @Override
  public void run(String... args) throws Exception {

    if (args.length == 0) {
        return;
    }

    if (!commandMap.containsKey(args[0])) {
        log.error("'{}' command not found", args[0]);
        System.exit(-1);
    }

    Command command = commandMap.get(args[0]);

    String[] arguments = Arrays.copyOfRange(args, 1, args.length);

    System.exit(command.execute(arguments));
  }
}

@Component
@Slf4j
public class SendEmailCommand implements Command {

  @Override
  public int execute(String... args) {

    try {
      // 省略业务逻辑代码

      log.info("send email success");

      return 0;

    } catch (Exception e) {
      log.error("send email error", e);
      return -1;
    }
  }
}

```

上面的代码我们采用了策略模式，后面即使新增其他定时任务也只是会改动很少的代码。


### 本地调试

cronjob不用打包成单独的镜像，它直接和我们的web应用公用同一个镜像,本地调试的时候也是极为方便的,只需要我们启动SpringBoot Application时指定参数即可

![调试定时任务](/images/java/debug-cronjob-in-idea.png)

对应的定时任务执行完成之后就会，application就会退出。


### cronjob yaml

一个基本的cronjob yaml如下所示

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: send-email-job
spec:
  failedJobsHistoryLimit: 3
  successfulJobsHistoryLimit: 1
  startingDeadlineSeconds: 180
  concurrencyPolicy: Forbid
  schedule: "0 4 * * 1-5"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: send-email-job
              image: harbor.xxx.com/think123/project
              imagePullPolicy: Always
              command: ["java"]
              args: ["-jar","/app/target/think123-task.jar","sendEmail"]
              envFrom:
                - configMapRef:
                    name: smcp-config
                - secretRef:
                    name: smcp-service-secret
              resources:
                requests:
                  cpu: "250m"
                  memory: 1024Mi
                limits:
                  cpu: "500m"
                  memory: 1024Mi
          restartPolicy: Never

```

在定时任务中,可能某个job还没有执行完，另外一个job就产生了。这个时候我们可以通过spec.concurrencyPolicy字段来定义具体的处理策略。
1. concurrencyPolicy=Allow，这也是默认情况，这意味着这些Job可以同时存在；
2. concurrencyPolicy=Forbid，这意味着不会创建新的Pod，该创建周期被跳过；
3. concurrencyPolicy=Replace，这意味着新产生的Job会替换旧的、没有执行完的Job


几个关键参数解释如下:

1. schedule : Unix Cron格式的表达式，cron表达式中的五个部分分别代表：分钟、小时、日、月、星期。
2. startingDeadlineSeconds ： 表示在过去的多少秒(这里设置的180)里，如果job创建失败的数据达到了100次，那么这个job就不会被创建执行了。
3. restartPolicy: 重启策略(有Never和OnFailure两个选项)。当job正常结束之后是否需要重启
>restartPolicy在Job对象里只允许被设置为Never和OnFailure；而在Deployment对象里，restartPolicy则只允许被设置为Always。

实际上在`jobTemplate.spec.template`中可以像pod中那样，指定volume,指定nodeSelector，都是可以的。这个template实际上就是指的pod的template


### 实际使用


上面的yaml虽然可以直接使用，但是我们用不着每个job都去写一份同样的模板，实际中我们会使用Kustomize控制模板来生成job。比如我们有一个新的任务，是计算热点文章并更新redis

对于这个任务而言，变化的主要有两个地方，第一个是定时任务的时间不同，第二个是指定的参数不同。所以我们的每个任务只需要更新这两个参数就行了

关于kustomize的使用可以参考我之前[的kustomize的介绍](https://juejin.im/post/6844904039017086990),打包的话可以看看[springboot build的文章](https://juejin.im/post/6844903981966360583)


对于模板设定，我们形成了下面的目录结构


```
$ tree
.
|-- base
|   |-- cronjob.yaml
|   `-- kustomization.yaml
`-- overlay
    `-- beta
        |-- kustomization.yaml
        `-- send-email-patch-args.yaml

```

cronjob.yaml作为所有job的模板,而send-emial-patch-args.yaml则是针对具体的job的一个替换。涉及到的yaml文件内容如下:

```yaml

# base/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- cronjob.yaml


# base/cronjob.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: think123-
spec:
  failedJobsHistoryLimit: 3
  successfulJobsHistoryLimit: 1
  startingDeadlineSeconds: 180
  concurrencyPolicy: Forbid
  schedule: "0 0 1 * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cron-job
              image: harbor.xxx.com/think123/my-task
              imagePullPolicy: Always
              args:
                - "help"
              envFrom:
                - configMapRef:
                    name: smcp-config
                - secretRef:
                    name: smcp-service-secret
              resources:
                requests:
                  cpu: "250m"
                  memory: 1024Mi
                limits:
                  cpu: "500m"
                  memory: 1024Mi
          restartPolicy: Never


# overlay/beta/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

nameSuffix: send-email-job

resources:
- ../../base/

patchesStrategicMerge:
  - send-email-patch-args.yaml


# overlay/beta/send-email-patch-args.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: think123-
spec:
  schedule: "0 4 * * 1-5"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: send-email-job
              args: ["sendEmail"]

```

你可以使用`kustomize build beta > send-email-cron-job.yaml`命令,然后查看send-email-cron-job.yaml文件，就可以看到生成的具体的cronjob的详细。

kustomize的文档可以参考: https://kubernetes-sigs.github.io/kustomize/api-reference/

### 为什么要用Kubernetes Cron Job

使用SpringBoot的定时任务不香吗？为什么要还要引入新的东西。再想这个问题的时候，想想为什么你在SpringBoot中不写Servlet,不是一样可以吗？

其实想想还是有原因的，首先我们的服务是分布式的，我们的定时任务应该只需要运行一次，而不是每个实例都运行一次，如果用SpringBoot的task那么我们需要用代码来保证这个行为。

如果引入分布式任务框架，又是引入了一堆其他新的东西，比如注册中心等等，而且还要去学习一项新的技术。

而我们的服务由于是通过Kubernetes部署的，我们的job再使用Kubernetes来，更是相得益彰。


