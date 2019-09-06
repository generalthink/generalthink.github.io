---
title: 揭秘,门面日志如何自动发现日志组件
date: 2019-09-06 16:20:00
tags: log
---

### commons-logging

commons-logging是apache提供的一个通用的日志接口，是为了避免和具体的日志方案直接耦合的一种实现。通过commons-logging用户可以自己选择log4j或者jdk自带的logging作为具体实现。

使用commons-logging的代码如下

```java
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;


public class MyApplication {
  private Log log = LogFactory.getLog(this.getClass());
}

```

<!--more-->

Log是一个接口，LogFactory的内部会去装载具体的日志系统，并获得实现该Log接口的实现类，其具体流程如下

1. 通过系统属性org.apache.commons.logging.LogFactory加载LogFactory的实现
2. 如果未找到，则通过服务发现机制(SPI)在META-INF/services/org.apache.commons.logging.LogFactory寻找实现类进行加载
3. 如果未找到，则从classpath下寻找commons-logging.properties文件，根据文件中org.apache.commons.logging.LogFactory配置进行加载
4. 如果仍未找到,则使用默认配置：如果找到Log4j 则默认使用log4j 实现，如果仍没有则使用JDK14Logger 实现，再没有则使用commons-logging 内部提供的SimpleLog 实现

所以只要你引入了log4j的jar包以及对其进行了配置底层就会直接使用log4j来进行日志输出了,其实质就是在org.apache.commons.logging.impl.Log4JLogger(commons-logging包)的getLogger方法调用了log4j的Logger.getLogger来返回底层的Logger,当记录日志的时候就会通过这个Logger写日志。


### Slf4j

slf4j全称为Simple Logging Facade for JAVA，java简单日志门面。其使用方式如下

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MyApplication {
  private Logger logger = LoggerFactory.getLogger(this.getClass());
}

```
slf4j和commons-logging不一样的是，如果你没有具体的日志组件(logback,log4j等)，它是无法打印日志的，如果你在一个maven项目里面只引入slf4j的jar包，然后记录日志

```java
public class Main {

  public static Logger logger = LoggerFactory.getLogger(Main.class);

  public static void main(String[] args) {
      logger.info("slf4j");
      System.out.println("done");
  }

}


```

就会出现下面的信息
```
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
```

如果你此时引入log4j或者logback的jar包就会打印出日志。

通过LoggerFactory.getLogger()查看其实现原理，其加载Logger核心源码如下:

```java
static Set<URL> findPossibleStaticLoggerBinderPathSet() {
  
  Set<URL> staticLoggerBinderPathSet = new LinkedHashSet<URL>();
  try {
      ClassLoader loggerFactoryClassLoader = LoggerFactory.class.getClassLoader();
      Enumeration<URL> paths;
      if (loggerFactoryClassLoader == null) {
          paths = ClassLoader.getSystemResources("org/slf4j/impl/StaticLoggerBinder.class");
      } else {
          paths = loggerFactoryClassLoader.getResources("org/slf4j/impl/StaticLoggerBinder.class");
      }
      while (paths.hasMoreElements()) {
          URL path = paths.nextElement();
          staticLoggerBinderPathSet.add(path);
      }
  } catch (IOException ioe) {
      Util.report("Error getting resources from path", ioe);
  }
  return staticLoggerBinderPathSet;
}

```

其主要思想就是去classpath下找org/slf4j/impl/StaticLoggerBinder.class,即所有slf4j的实现,无论是log4j还是logback都有实现类，同学们可以看下自己项目中的jar包。

如果引入了多个实现，编译器会在编译的时候选择其中一个实现类进行绑定,也会将具体绑定的哪个日志框架告诉你

```java
private static void reportActualBinding(Set<URL> binderPathSet) {
  
    if (binderPathSet != null && binderPathSet.size() > 1) {
        Util.report("Actual binding is of type [" + StaticLoggerBinder.getSingleton().getLoggerFactoryClassStr() + "]");
    }
}

```
![具体实现](/images/java/logback-slf4j.png)