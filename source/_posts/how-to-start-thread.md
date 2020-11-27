---
title: 为什么start方法才能启动线程,而run不行？
date: 2020-08-03 09:47:28
tags: java
---

我们都知道，一个线程直接对应了一个Thread对象,在刚开始学习线程的时候我们也知道启动线程是通过start()方法,而并非run()方法。

那这是为什么呢？

如果你熟悉Thread的代码的话,你应该知道在这个类加载的时候会注册一些native方法

```java
public
class Thread implements Runnable {
  /* Make sure registerNatives is the first thing <clinit> does. */
  private static native void registerNatives();
  static {
      registerNatives();
  }
}

```

<!--more-->

一看到native我就想起了JNI,registerNatives()实际上就是java方法和C/C++的函数对应。在首次加载的时候就会注册这些native方法。Thread中有很多native方法，大家有兴趣的可以去看看。

> 关于JNI方法的命名,我们可以这样测试，我们用java声明一个native方法，然后先使用javac编译源文件(比如javac main.java),然后在使用javah即可生成头文件(javah main),打开这个头文件你就知道方法命名是如何的了 

我们在JVM源码中搜索Java_java_lang_Thread_registerNatives可以看到registerNatives方法的具体实现

```c
static JNINativeMethod methods[] = {
    {"start0",           "()V",        (void *)&JVM_StartThread},
    {"stop0",            "(" OBJ ")V", (void *)&JVM_StopThread},
    {"isAlive",          "()Z",        (void *)&JVM_IsThreadAlive},
    {"suspend0",         "()V",        (void *)&JVM_SuspendThread},
    {"resume0",          "()V",        (void *)&JVM_ResumeThread},
    {"setPriority0",     "(I)V",       (void *)&JVM_SetThreadPriority},
    {"yield",            "()V",        (void *)&JVM_Yield},
    {"sleep",            "(J)V",       (void *)&JVM_Sleep},
    {"currentThread",    "()" THD,     (void *)&JVM_CurrentThread},
    {"countStackFrames", "()I",        (void *)&JVM_CountStackFrames},
    {"interrupt0",       "()V",        (void *)&JVM_Interrupt},
    {"isInterrupted",    "(Z)Z",       (void *)&JVM_IsInterrupted},
    {"holdsLock",        "(" OBJ ")Z", (void *)&JVM_HoldsLock},
    {"getThreads",        "()[" THD,   (void *)&JVM_GetAllThreads},
    {"dumpThreads",      "([" THD ")[[" STE, (void *)&JVM_DumpThreads},
    {"setNativeName",    "(" STR ")V", (void *)&JVM_SetNativeThreadName},
};

JNIEXPORT void JNICALL
Java_java_lang_Thread_registerNatives(JNIEnv *env, jclass cls)
{
    (*env)->RegisterNatives(env, cls, methods, ARRAY_LENGTH(methods));
}
```
可以看到，在registerNatives函数中，注册了很多的native方法比如这里的start0()方法。

所有对JNI函数的调用都使用了env指针,该指针是对每一个本地方法的第一个参数。env指针是函数指针表的指针。我们可以在<https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/jniTOC.html>中找到JNI API


在Thread.start()方法中，实际就是通过调用start0()方法来启动线程的。


```java
public synchronized void start() {
       
  if (threadStatus != 0)
      throw new IllegalThreadStateException();


  group.add(this);

  boolean started = false;
  try {
      // 主要调用了start0()这个native方法来启动线程
      start0();
      started = true;
  } finally {
      try {
          if (!started) {
              group.threadStartFailed(this);
          }
      } catch (Throwable ignore) {
          /* do nothing. If start0 threw a Throwable then
            it will be passed up the call stack */
      }
  }
}


private native void start0();

```

而JNINativeMethod这个数据结构定义如下:

```c

typedef struct {
  char *name;
  char *signature;
  void *fnPtr;
}
```

因此start0()这个方法对应的本地函数是JVM_StartThread

```c
 {"start0", "()V", (void *)&JVM_StartThread}
```

我们接下来看JVM_StartThread的方法实现

```c
JVM_ENTRY(void, JVM_StartThread(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_StartThread");
  JavaThread *native_thread = NULL;

  bool throw_illegal_thread_state = false;

 
  {
    // Ensure that the C++ Thread and OSThread structures aren't freed before
    // we operate.
    MutexLocker mu(Threads_lock);

    // 从JDK5开始，使用java.lang.Thread threadStatus来防止重新启动一个已经启动的线程
    if (java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread)) != NULL) {
      throw_illegal_thread_state = true;
    } else {
      
      jlong size =
             java_lang_Thread::stackSize(JNIHandles::resolve_non_null(jthread));
      
      NOT_LP64(if (size > SIZE_MAX) size = SIZE_MAX;)
      size_t sz = size > 0 ? (size_t) size : 0;

      // 创建Java线程
      native_thread = new JavaThread(&thread_entry, sz);

     
      if (native_thread->osthread() != NULL) {
        native_thread->prepare(jthread);
      }
    }
  }

  if (throw_illegal_thread_state) {
    THROW(vmSymbols::java_lang_IllegalThreadStateException());
  }

  // 省略了部分代码


  // 将线程状态设置为Runnable,表示可以被运行
  Thread::start(native_thread);

JVM_END
```

上面代码主要做了三件事情

1. 判断当前线程状态是否合法，不合法抛出IllegalThreadStateException

2. 创建一个Java线程(我们需要重点关注的)

3. 将线程状态设置为Runnable

如果面试官以后再问你两次调用start()方法会怎样,你就大胆而坚定的回复说抛出IllegalThreadStateException。

在JavaThread构造函数中实际调用的是os::create_thread方法

```c
bool os::create_thread(Thread* thread, ThreadType thr_type,
                       size_t req_stack_size) {

  
  OSThread* osthread = new OSThread(NULL, NULL);
  if (osthread == NULL) {
    return false;
  }

  osthread->set_thread_type(thr_type);
  osthread->set_state(ALLOCATED);

  thread->set_osthread(osthread);

  // init thread attributes
  pthread_attr_t attr;
  pthread_attr_init(&attr);
  pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

  
  size_t stack_size = os::Posix::get_initial_stack_size(thr_type, req_stack_size);
  int status = pthread_attr_setstacksize(&attr, stack_size);

  ThreadState state;

  {
    pthread_t tid;

    // 创建线程
    int ret = pthread_create(&tid, &attr, (void* (*)(void*)) thread_native_entry, thread);

    
    // 省略其他代码...

  }
  return true;
}

```
pthread_create函数作用是创建一个线程,它的第三个参数是线程运行函数的起始地址,第四个参数是运行函数参数。
> IEEE标准1003.1c中定义了线程的标准,它定义的线程包叫做Pthread,大部分UNIX系统都支持这个标准。

而pthread_create实际上是调用了clone()完成系统调用创建线程的,所以目前 Java 在 Linux 操作系统下采用的是用户线程加
轻量级线程，一个用户线程映射到一个内核线程.

我们的thread_native_entry实际传入的是JavaThread这个对象,所以最终会调用JavaThread::run()(thread.cpp中)

```c
void JavaThread::run() {
  
  ...

  thread_main_inner();
  ...
}

void JavaThread::thread_main_inner() {
  ...
  this->entry_point()(this,this);
  ...
}

```

thread_main_inner函数中entry_point的返回值实际上是我们在创建JavaThread的时候传入的第一个参数thread_entry。而thread_entry指针指向的函数如下:

```c
static void thread_entry(JavaThread* thread, TRAPS) {
  HandleMark hm(THREAD);
  Handle obj(THREAD, thread->threadObj());
  JavaValue result(T_VOID);
  JavaCalls::call_virtual(&result,
    obj,
    SystemDictionary::Thread_klass(),
    // run方法名称run
    vmSymbols::run_method_name(),
    // 方法签名()V
    vmSymbols::void_method_signature(),
    THREAD);
}

```

这样我们就最终通过JavaCalls调用了run方法。


### 总结

`new Thread`只是创建了一个普通的Java对象,只有在调用了start()方法之后才会创建一个真正的线程,在JVM内部会在创建线程之后调用run()方法,执行相应的业务逻辑。

由于Java中线程最终还是和操作系统线程挂钩了的,所以线程资源是一个很重要的资源,为了复用我们一般是通过线程池的方式来使用线程。