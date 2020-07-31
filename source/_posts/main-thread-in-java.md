---
title: Java中的main线程是如何被创建的？
date: 2020-07-27 15:51:09
tags: [java,jvm]
---

当我们运行Java程序main方法的时候，我们都知道当前线程是main线程

```java
Thread.currentThread().getName()
```

那么这个main线程是被谁启动,又是在什么时候被启动的呢?我们通过源码一探究竟。

<!--more-->

jvm的启动入口是main.c,由于我之前可以在mac上调试jvm了，所以我通过下面的参数进行启动

```
java -Xss512K -XX:+UseConcMarkSweepGC -Xms512M Main arg1=think123 arg2=666
```

```java
public class Main {

  public static void main(String[] args) {

      for(String arg: args) {
          System.out.println("input arg : " + arg);
      }

      System.out.println("main thread name : " + Thread.currentThread().getName());
  }
}
```

main.c中首先会通过启动器来创建启动jvm

```c++
 return JLI_Launch(margc, margv,
   jargc, (const char**) jargv,
   0, NULL,
   VERSION_STRING,
   DOT_VERSION,
   (const_progname != NULL) ? const_progname : *margv,
   (const_launcher != NULL) ? const_launcher : *margv,
   jargc > 0,
   const_cpwildcard, const_javaw, 0);
```

JLI_Launch的实现在java.c文件中,它的主要流程是

1. 创建执行环境，主要是确定jrepath/jvmpath

2. 加载jvm

3. 解析参数

4. 初始化jvm,执行main方法

```c
JNIEXPORT int JNICALL
JLI_Launch(int argc, char ** argv,              /* main argc, argv */
  int jargc, const char** jargv,          /* java args */
  int appclassc, const char** appclassv,  /* app classpath */
  const char* fullversion,                /* full version defined */
  const char* dotversion,                 /* UNUSED dot version defined */
  const char* pname,                      /* program name */
  const char* lname,                      /* launcher name */
  jboolean javaargs,                      /* JAVA_ARGS */
  jboolean cpwildcard,                    /* classpath wildcard*/
  jboolean javaw,                         /* windows-only javaw */
  jint ergo                               /* unused */
)
{
  char jvmpath[MAXPATHLEN];
  char jrepath[MAXPATHLEN];
  char jvmcfg[MAXPATHLEN];

  
  // 创建执行环境
  CreateExecutionEnvironment(&argc, &argv,
                             jrepath, sizeof(jrepath),
                             jvmpath, sizeof(jvmpath),
                             jvmcfg,  sizeof(jvmcfg));

  if (!IsJavaArgs()) {
      SetJvmEnvironment(argc,argv);
  }

  ifn.CreateJavaVM = 0;
  ifn.GetDefaultJavaVMInitArgs = 0;

  // 加载JVM
  if (!LoadJavaVM(jvmpath, &ifn)) {
      return(6);
  }

  // 解析参数
  if (!ParseArguments(&argc, &argv, &mode, &what, &ret, jrepath)) {
      return(ret);
  }

  // 初始化JVM
  return JVMInit(&ifn, threadStackSize, argc, argv, mode, what, ret);
}
```
上面的代码我保留了主体流程,将其他代码省略掉了。

### 创建执行环境

```c
void CreateExecutionEnvironment(int *pargc, char ***pargv,
  char jrepath[], jint so_jrepath,
  char jvmpath[], jint so_jvmpath,
  char jvmcfg[],  jint so_jvmcfg) {

    jboolean jvmpathExists;

    // 设置可执行文件的path,这里的path是java这个可执行程序的绝对路径，比如我这里是
    // /Users/xxx/jvm/jdk12-06222165c35f/build/macosx-x86_64-server-slowdebug/jdk/bin/java
    // 后面会根据这个路径来计算JREPath以及JDKPath
    SetExecname(*pargv);

    char * jvmtype    = NULL;
    int  argc         = *pargc;
    char **argv       = *pargv;

    // 找到jre path
    if (!GetJREPath(jrepath, so_jrepath, JNI_FALSE) ) {
        JLI_ReportErrorMessage(JRE_ERROR1);
        exit(2);
    }

   // 省略部分代码

    // 找到jvm path
    if (!GetJVMPath(jrepath, jvmtype, jvmpath, so_jvmpath)) {
        JLI_ReportErrorMessage(CFG_ERROR8, jvmtype, jvmpath);
        exit(4);
    }

    // mac os独有操作    
    MacOSXStartup(argc, argv);

   
    return;
}
```
> 需要注意的是jvmpath/jrepath的长度不能超过1024字节,所以我们安装java的时候一定要注意文件夹层次不能太深

执行完上面的代码之后,jvmpath/jrepath的值如下

![执行环境](/images/macos-jvm/execution-env.png)

着重注意下这里的jvmpath的值是libjvm.dylib,这个就是我们要使用的JVM动态链接库(windows中是jvm.dll,linux中是libjvm.so)


### 加载JVM

接下来加载JVM，实际上是加载libjvm.dylib这个动态链接库。

```c
boolean LoadJavaVM(const char *jvmpath, InvocationFunctions *ifn)
{
    Dl_info dlinfo;
    void *libjvm;


#ifndef STATIC_BUILD
  // 通过dlopen加载动态库文件(libjvm.dylib),并返回一个句柄
  libjvm = dlopen(jvmpath, RTLD_NOW + RTLD_GLOBAL);
#else
  libjvm = dlopen(NULL, RTLD_FIRST);
#endif
  if (libjvm == NULL) {
      JLI_ReportErrorMessage(DLL_ERROR1, __LINE__);
      JLI_ReportErrorMessage(DLL_ERROR2, jvmpath, dlerror());
      return JNI_FALSE;
  }

  // 通过dlsym函数将libjvm中JNI_CreateJavaVM函数地址绑定到ifn的CreateJavaVM属性
  ifn->CreateJavaVM = (CreateJavaVM_t)
      dlsym(libjvm, "JNI_CreateJavaVM");

  if (ifn->CreateJavaVM == NULL) {
      JLI_ReportErrorMessage(DLL_ERROR2, jvmpath, dlerror());
      return JNI_FALSE;
  }
  
  // 通过dlsym函数将libjvm中JNI_GetDefaultJavaVMInitArgs函数地址绑定到ifn的GetDefaultJavaVMInitArgs属性
  ifn->GetDefaultJavaVMInitArgs = (GetDefaultJavaVMInitArgs_t)
      dlsym(libjvm, "JNI_GetDefaultJavaVMInitArgs");

  if (ifn->GetDefaultJavaVMInitArgs == NULL) {
      JLI_ReportErrorMessage(DLL_ERROR2, jvmpath, dlerror());
      return JNI_FALSE;
  }

  // 同上将libjvm中的GetCreatedJavaVMs函数地址绑定到ifn的GetCreatedJavaVMs属性
  ifn->GetCreatedJavaVMs = (GetCreatedJavaVMs_t)
  dlsym(libjvm, "JNI_GetCreatedJavaVMs");


  if (ifn->GetCreatedJavaVMs == NULL) {
      JLI_ReportErrorMessage(DLL_ERROR2, jvmpath, dlerror());
      return JNI_FALSE;
  }

  return JNI_TRUE;
}

```

LoadJVM主要做了以下2件事

1. 通过dlopen加载libjvm.dylib动态链接库
2. 绑定动态链接库中的函数到InvocationFunctions这个结构体的属性中

>dlopen和dlsym系统提供的函数,dlsym一般和dlopen配合使用

### 解析命令行参数

在ParseArguments函数中主要解析的是命令行参数比如-classpath,-version,-help等,但是这里最重要的是解析-Xss,-Xmx,-Xms这三个参数,因为这三个参数格式和其他不一样。都是参数名称后面跟上具体大小


![参数解析](/images/macos-jvm/parse-arg.png)

> 单位只能是T(t),G(g),M(m),K(k)这8个中的一个

其他参数解析和判定会在初始化JVM的时候完成

### 初始化VM


JVMInit方法最终会调用java_md_macosx.m中的ContinueInNewThread0方法

```c

ContinueInNewThread0(int (JNICALL *continuation)(void *), jlong stack_size, void * args) {
  int rslt;
  pthread_t tid;
  pthread_attr_t attr;
  pthread_attr_init(&attr);
  pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE);

  // 设置stack_size(-Xss参数解析出来的值)
  if (stack_size > 0) {
    pthread_attr_setstacksize(&attr, stack_size);
  }
  pthread_attr_setguardsize(&attr, 0); // no pthread guard page on java threads

  // 第一个参数是线程提示符指针,第二个参数是线程属性
  // 第三个参数是线程运行函数的起始地址,第四个参数是运行函数参数
  if (pthread_create(&tid, &attr, (void *(*)(void*))continuation, (void*)args) == 0) {
    void * tmp;
    pthread_join(tid, &tmp);
    rslt = (int)(intptr_t)tmp;
  } else {
   
    rslt = continuation(args);
  }

  pthread_attr_destroy(&attr);
  return rslt;
}
```

pthread_create函数作用是创建一个线程(**我们的main线程就这样被创建出出来了**),pthread_create的第三个参数continuation传递进来的函数是JavaMain,这就相当于java线程中run方法。它位于java.c中,由于函数过长，我只保留了比较重要的部分
> IEEE标准1003.1c中定义了线程的标准,它定义的线程包叫做Pthread,大部分UNIX系统都支持这个标准。
```c

int JNICALL JavaMain(void * _args)
{
   
 ... 省略代码
  
  // 通过CreateJavaVM方法初始化JVM,这里逻辑比较复杂，暂时不做展开
  // 初始化jvm中的时候会解析和检查其他参数,比如-XX:+UseConcMarkSweepGC
  if (!InitializeJVM(&vm, &env, &ifn)) {
    JLI_ReportErrorMessage(JVM_ERROR1);
    exit(1);
  }

  ret = 1;

  // 加载我们要运行的class
  mainClass = LoadMainClass(env, mode, what);
  CHECK_EXCEPTION_NULL_LEAVE(mainClass);
 
  // 获取main方法id(main方法入口地址)
  mainID = (*env)->GetStaticMethodID(env, mainClass, "main",
                                     "([Ljava/lang/String;)V");
  CHECK_EXCEPTION_NULL_LEAVE(mainID);

  // 调用main方法
  (*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs);

  // 等到所有非守护进程结束后,销毁VM
  LEAVE();
}

```

调用main方法则是通过jni.cpp中的jni_invoke_static方法,而该方法最终是通过JavaCalls::call(javaCalls.cpp)完成的。

> javaCalls::call方法只有java线程才能调用该方法


至此我们的main线程就被启动起来了。

