---
title: Synchronized锁优化源码解析
date: 2020-10-22 15:48:14
tags:
---

### 模板解释器

我们都知道Java之所以可以一次编译到处运行，完全是因为字节码的原因，字节码就相当于中间层屏蔽了底层细节。但是想要在机器执行，最终还是要翻译成机器指令。

而JVM是通过C/C++来编写的,Java程序编译后，会产生很多字节码指令，每一个字节码指令在JVM底层执行的时候又会编程一堆C代码，这一堆C代码在编译之后又会编程很多的机器指令，这样我们的java代码到最终执行的机器指令那一层，所产生的机器指令时指数级的，这也就导致了Java执行效率低下。

早期的JVM是因为解释执行慢而被人诟病，那么有没有办法优化这个问题呢？我们发现之所以慢是因为java和机器指令之间隔了一层C/C++,而GCC之类的编译器又不能做到绝对的智能编译，所产生的机器码效率就不是很高。因此我们只要跳过C/C++这个层次，直接将Java字节码和本地机器码进行一个对应就可以了。

因此HotSpot的工程师们废弃了早期的解释执行器，而采用了模板执行器。所谓的模板就是将一个 java 字节码通过人工手动的方式编写为固定模式的机器指令，这部分不在需要 GCC 的帮助，这样就可以大大减少最终需要执行的机器指令，所以才能提高效率。

<!--more-->

在OpenJdk12源码中，JVM所有的解释器都在`src/hotspot/share/interpreter `目录下，`templateInterpreter.cpp`就是模板解释器的代码位置。分析这里的initialize方法，我们可以在`templateTable.cpp`中找到和synchronized相关的两个指令(`monitorenter`,`monitorexit`)的实现方式，当然这里面还有其他我们熟悉的指令，比如`invokedynamic`,`newarray`等指令


```c++
def(Bytecodes::_monitorenter, ____|disp|clvm|____, atos, vtos, monitorenter, _);
def(Bytecodes::_monitorexit, ____|____|clvm|____, atos, vtos, monitorexit, _ );

```

### monitorenter执行逻辑

这里倒数第二个参数的`monitorenter`函数和`monitorexit`函数是对应字节码的机器码模板的位置，这里我们看下`monitorenter`的实现，因为机器码的实现和CPU相关的,这里我们看下x86的实现(`templateTable_x86.cpp`),当然也可以在`src/hotspot/cpu`下看到其他的实现，比如`ppc,arm,s390`等

```c++
void TemplateTable::monitorenter() {
  ...
    // 将要锁的对象指针放到BasicObjectLock的obj变量中
    __ movptr(Address(rmon, BasicObjectLock::obj_offset_in_bytes()), rax);
    // 跳转执行 lock_object 函数
    __ lock_object(rmon);
  ...
  }

void InterpreterMacroAssembler::lock_object(Register lock_reg) {

  // 如果使用重量级锁，则直接进入monitorenter()执行
  if (UseHeavyMonitors) {
    call_VM(noreg,
            CAST_FROM_FN_PTR(address, InterpreterRuntime::monitorenter),
            lock_reg);
  } else {
    ...

    // Load object pointer into obj_reg
    movptr(obj_reg, Address(lock_reg, obj_offset));

    // 关于偏向锁的处理
    if (UseBiasedLocking) {
      // lock_reg : 存储指向BasicObjectLock的指针
      // obj_reg : 存储锁对象的指针
      // slow_case : 标记，类似于goto,这里指的是InterpreterRuntime::monitorenter()
      // done: 标记，标志着获取锁成功。
      // slow_case 和 done 也被传入，这样在biased_locking_enter()中，就可以根据情况跳到这两处了。
      biased_locking_enter(lock_reg, obj_reg, swap_reg, tmp_reg, false, done, &slow_case);
    }
    ...

   // slow_case逻辑,需要进入InterpreterRuntime::monitorenter()中获取锁。
    bind(slow_case);
    // Call the runtime routine for slow case
    call_VM(noreg,
            CAST_FROM_FN_PTR(address, InterpreterRuntime::monitorenter),
            lock_reg);


    // 这里的done和上面传入到偏向锁的done是一样的。直接跳到这表明获取锁成功，接下来就会返回进行字节码的执行了。
    bind(done);
  }
}

```

从代码可以看出如果启用了重量级锁，那么就直接走重量级锁的逻辑(monitorenter)，不然会先处理偏向锁的逻辑，然后不满足会再回到monitorenter中
> 偏向锁: -XX:+UseBiasedLocking , JDK1.6之后默认启用
> 重量级锁: -XX:+UseHeavyMonitors

### 偏向锁,轻量级锁以及重量级锁

我们提到了重量级锁和偏向锁，这两个是什么意思呢？

我们都知道Java的线程是映射到操作系统的原生线程之上的，无论是是阻塞还是唤醒一个线程，都需要操作系统的帮助，这就需要从用户态转换到核心态中。而很多人说synchronized慢也正是由于这个原因。之前的文章也说过synchronized实际上是通过操作系统的互斥量来实现的，而这也被称为重量级锁。

相对于重量级锁，还有一个叫做轻量级锁。它的加锁不是通过操作系统来实现的，而是通过CAS配合Mark Word一起实现的，后面我会通过源码来展示它的实现方式。


而偏向锁相对于轻量级锁更加轻量，这里的偏向指的是偏向某一个线程。如果只有一个线程来获取锁，那么锁对象就会偏向这个线程，如果在接下来的执行过程中，该锁没有被其他的线程获取，则持有偏向锁的线程将永远不需要再进行同步。


接下来我们沿着源码从 偏向锁-->轻量级锁-->重量级锁这样来分析下JVM是如何进行优化的。


### 内存布局

在分析锁实现之前,你可能要先去看看上一篇文章,看看对象在内存中的布局，这里我贴一张图让你在重温下

![未开启指针压缩](/images/macos-jvm/mark-word-32-system.png)

### 锁状态转化及对象Mark Word的关系

实际上锁的优化逻辑,在JDK中的wiki中已经有一个提纲挈领的图了，这里我先贴出来。后面的代码分析也会跟着这张图走。

![锁优化](/images/macos-jvm/Synchronization.gif)


### 偏向锁

#### 偏向锁的启动

偏向锁会在虚拟机启动后的4秒之后才会生效，我们可以从`hotspot/share/runtime/biasedLocking.cpp`看到这样的设定

```c

void BiasedLocking::init() {
  
  if (UseBiasedLocking) {
    if (BiasedLockingStartupDelay > 0) {
      EnableBiasedLockingTask* task = new EnableBiasedLockingTask(BiasedLockingStartupDelay);
      task->enroll();
    } else {
      VM_EnableBiasedLocking op(false);
      VMThread::execute(&op);
    }
  }
}

// 上面的task最终会调用这个方法，将锁对象的类的mark word的后三位设置为101
static void enable_biased_locking(InstanceKlass* k) {
  k->set_prototype_header(markOopDesc::biased_locking_prototype());
}

```

`BiasedLockingStartupDelay`默认时间是4000毫秒，所以会在启动4s之后启动一个定时任务来设置开启偏向锁的设定。

我们可以通过`-XX:BiasedLockingStartupDelay=0`来设置马上启动偏向锁。这里也填了上一篇的一个坑。
> java -XX:+PrintFlagsFinal | grep BiasedLockingStartupDelay


定时任务会调用`enable_biased_locking`方法,将锁对象的类的Mark Word的后三个字节设置为101,**锁对象类的Mark Word被称为prototype_header**,记住这个下面分析偏向锁的时候会用到。

```java

MyObject obj = new MyObject();

synchronized(obj) {
  
  doSomething();
}

```
上面Java代码中锁对象是obj,其所属类型是MyObject(obj是MyObject的一个实例)。而`prototype header`实际上就是MyObject的Mark Word。

#### 偏向锁申请

`biased_locking_enter()`方法比较长，所以我们一段一段来分析。以下代码片段均来自于`hotspot/cpu/x86/macroAssembler_x86.cpp::biased_locking_enter`中。


1. 首先判断Mark Word中的后三位(是否偏向锁+锁标志位)的值是否为5,即是否为偏向锁状态，如果是则执行后面fast_enter的逻辑,如果不是则执行第2步

```c++

Address mark_addr (obj_reg, oopDesc::mark_offset_in_bytes());

... 省略部分代码 ...

Label cas_label;
int null_check_offset = -1;

// 如果swap_reg中没存mark_addr，那么就先将mark_addr存入swap_reg中。
if (!swap_reg_contains_mark) {
  null_check_offset = offset();
  movptr(swap_reg, mark_addr);
}
// 将对象的mark_addr，即markOop指针移入tmp_reg中
movptr(tmp_reg, swap_reg);

// 将tmp_reg和biased_lock_mask_in_place(111)进行与操作,取出markOop中后三位，即(是否偏向锁+锁标志位)
andptr(tmp_reg, markOopDesc::biased_lock_mask_in_place);

//查看Mark Word后三位是否为5(biased_lock_pattern为5,即101），如果不相等，则表明不为偏向锁状态,跳往cas_label。否则即为偏向锁状态。
cmpptr(tmp_reg, markOopDesc::biased_lock_pattern);
jcc(Assembler::notEqual, cas_label);

```

2. 判断锁对象Mark Word中是否包含当前线程地址,最后三位标志位是否相同，且epoch值和类的epoch值是否相等。如果都相同，那么当前线程持有该偏向锁，可以直接返回。不然执行第3步

```c++
// 将类的prototype_header(Mark Word)加载到tmp_reg中
load_prototype_header(tmp_reg, obj_reg);

//将当前线程地址和类的的prototype_header相或，这样得到的结果为(当前线程id + prototype_header中的(epoch + 分代年龄 + 偏向锁标志 + 锁标志位))
orptr(tmp_reg, r15_thread);

// 将上面计算得到的结果与锁对象的markOop进行异或,得到一个结果diff(相等的位被置为0)
xorptr(tmp_reg, swap_reg);
Register header_reg = tmp_reg;

// 将header_reg中除了分代年龄之外的其他位取了出来，即将上面异或得到的结果中分代年龄给忽略掉。
andptr(header_reg, ~((int) markOopDesc::age_mask_in_place));

// 如果除了分代年龄，对象的markOop和(当前线程地址+其他位)相等，那么上面与操作的结果应该为0，表明对象之前已经偏向当前线程,那么跳到done处，执行同步代码块中的代码
// 否则表明当前线程还不是偏向锁的持有者，会接着往下走。
jcc(Assembler::equal, done);

```

需要注意的是这里会得到一个异或结果header_reg，会在后面的步骤中使用到。

3. 判断类对象是否支持偏向锁，如果不支持，则跳转到第6步执行移除偏向锁的逻辑。 如果支持则跳转到第4步执行

```c++
testptr(header_reg, markOopDesc::biased_lock_mask_in_place);
jccb(Assembler::notZero, try_revoke_bias);
```

header_reg中存储的是(当前线程id + prototype_header中的(epoch + 分代年龄 + 偏向锁标志 + 锁标志位)) 和 锁对象 Mark Word异或的结果，我们要查看后三位(biased_lock_mask_in_place的值是111)的结果是否为0，如果不为0，表示之前异或时锁对象的Mark Word后三位和对象所属类的后三位不一致,所以对象所属类不再支持偏向锁，此时需要跳转到try_revoke_bias进行移除偏向锁操作。

> 这个testptr的实现实际上是获取第一个参数多少位的值。多少位是根据第二个参数的二进制长度来决定的。

4. 执行到这里表示锁对象以及类对象都支持偏向锁,但是并不是偏向的当前线程。所以接下来会判断异或结果中的epoch是否为0，如果为0，则跳转到第5步执行。如果不为0，则证明锁过期了,跳转到第7步执行重新偏向逻辑

```c++
// 测试锁对象的epoch值和锁对象类的epoch是否相等，如果不相等，则证明锁过期了，需要重新偏向
testptr(header_reg, markOopDesc::epoch_mask_in_place);
jccb(Assembler::notZero, try_rebias);

```

5. 表明锁对象还未偏向任何线程，则可以尝试去获取锁，使得对象偏向当前线程

```c++
// 取出对象Mark Word中除线程地址之外的其他位
andptr(swap_reg,
       markOopDesc::biased_lock_mask_in_place | markOopDesc::age_mask_in_place | markOopDesc::epoch_mask_in_place);

// 将其他位移动至 tmp_reg
movptr(tmp_reg, swap_reg);

// 将其他位和当前线程进行或，构造成一个新的完整的Mark Word,存入tmp_reg中。新的Mark Word因为保存了当前线程地址，所以会偏向当前线程。
orptr(tmp_reg, r15_thread);

// 尝试利用CAS操作将新构成的Mark Word存入锁对象的mark_addr(Mark Word)，如果设置成功，则获取偏向锁成功。
// cmpxchgptr操作会强制将rax寄存器（swap_reg）中内容作为老数据，与第二个参数，在这里即mark_addr处的内容进行比较，如果相等，则将第一个参数的内容，即tmp_reg中的新数据，存入mark_addr。
cmpxchgptr(tmp_reg, mark_addr); // compare tmp_reg and swap_reg

// 上面CAS操作失败的情况下，表明对象头中的markOop数据已经被篡改，即有其他线程已经获取到偏向锁，因为偏向锁不容许多个线程访问同一个锁对象，所以需要跳到slow_case(InterpreterRuntime::monitorenter)处，去撤销该对象的偏向锁，并进行锁升级。
if (slow_case != NULL) {
  jcc(Assembler::notZero, *slow_case);
}
//上面CAS成功的情况下，直接就跳往done处，回去执行方法的字节码了。
jmp(done);
```

6. try_revoke_bias使用CAS操作，重置mark word。撤销偏向锁后后续所有操作都走轻量级锁的加锁过程

  try_revoke_bias和try_rebias的代码定义也在biased_locking_enter中

```c++
bind(try_revoke_bias);

// 类的prototype_header中已经支持偏向锁的位了，即这个类的所有对象都不再支持偏向锁了，但是当前对象仍为偏向锁状态，所以我们需要重置下当前对象的markOop为无锁态。

// 将锁对象所属类的prototype_header送入tmp_reg。
load_prototype_header(tmp_reg, obj_reg);

// 尝试用CAS操作，使对象的markOop重置为无锁态。这里是否失败无所谓，即使失败了，也表明其他线程已经移除了对象的偏向锁标志。
cmpxchgptr(tmp_reg, mark_addr); 

```

7. try_rebias就是将使得锁对象重新偏向当前线程，如果失败则走slow_case(InterpreterRuntime::monitorenter)进行偏向锁撤销逻辑

```c++

bind(try_rebias);

// 将锁对象所属类的prototype_header送入tmp_reg。
load_prototype_header(tmp_reg, obj_reg);

//将当前线程地址和类的的prototype header(Mark Word)相或，这样得到的结果为(当前线程id + prototype_header中的(epoch + 分代年龄 + 偏向锁标志 + 锁标志位))
orptr(tmp_reg, r15_thread);

// 尝试用CAS操作，成功表示重偏向加锁成功
cmpxchgptr(tmp_reg, mark_addr); 

// 如果CAS失败，则表明Mark Word已经被其他线程更改，需要跳往slow_case进行撤销偏向锁，否则跳往done处，执行字节码。
if (slow_case != NULL) {
  jcc(Assembler::notZero, *slow_case);
}
jmp(done);

```

#### 偏向锁的撤销

`slow_case`(偏向锁的撤销)的逻辑是在`InterpreterRuntime::monitorente`r中

```c++
IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem))
  Handle h_obj(thread, elem->obj());
  if (UseBiasedLocking) {
    // Retry fast entry if bias is revoked to avoid unnecessary inflation
    ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
  } else {
    ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
  }
IRT_END


void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock,
                                    bool attempt_rebias, TRAPS) {
  if (UseBiasedLocking) {
    if (!SafepointSynchronize::is_at_safepoint()) {
      BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
      if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
        return;
      }
    } else {
      assert(!attempt_rebias, "can not rebias toward VM thread");
      // 移除偏向锁
      BiasedLocking::revoke_at_safepoint(obj);
    }
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
  }

  slow_enter(obj, lock, THREAD);
}
```

`BiasedLocking::revoke_and_rebias`也会再重试下看能否使用偏向锁，逻辑基本和上面分析的一致，你要是看了这里面的代码你还会发现如果你调用了`System.identityHashCode()`是会移除偏向锁的。

由于偏向锁的移除需要在全局安全点的时候执行,所以如果当有大量线程竞争同一个锁资源时，我们可以通过关闭偏向锁来调优系统性能。

接下来我们来看`revoke_at_safepoint`会做哪些事情

1. `update_heuristics()`方法会将类对象上revoke次数加1

```c++

// 增加revoke次数
if (revocation_count <= BiasedLockingBulkRevokeThreshold) {
  revocation_count = k->atomic_incr_biased_lock_revocation_count();
}

// 如果revoke次数等于BiasedLockingBulkRevokeThreshold(默认40)
if (revocation_count == BiasedLockingBulkRevokeThreshold) {
  return HR_BULK_REVOKE;
}

// 如果revoke次数等于BiasedLockingBulkRebiasThreshold(默认20)
if (revocation_count == BiasedLockingBulkRebiasThreshold) {
  return HR_BULK_REBIAS;
}

return HR_SINGLE_REVOKE;
```

2. 如果撤销次数等于`BiasedLockingBulkRebiasThreshold`(默认20)，则认为类对象还可以重偏向，因此要做以下操作(bulk rebias)

```c++
if (klass->prototype_header()->has_bias_pattern()) {
  int prev_epoch = klass->prototype_header()->bias_epoch();

  // 将类对象的prototype_header中的epoch值加1
  klass->set_prototype_header(klass->prototype_header()->incr_bias_epoch());
  int cur_epoch = klass->prototype_header()->bias_epoch();

 // 遍历所有线程的栈，找到所有类对象的实例，将它们Mark Word中的epoch值加1
  for (; JavaThread *thr = jtiwh.next(); ) {
    GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(thr);
    for (int i = 0; i < cached_monitor_info->length(); i++) {
      MonitorInfo* mon_info = cached_monitor_info->at(i);
      oop owner = mon_info->owner();
      markOop mark = owner->mark();
      if ((owner->klass() == k_o) && mark->has_bias_pattern()) {
        assert(mark->bias_epoch() == prev_epoch || mark->bias_epoch() == cur_epoch, "error in bias epoch adjustment");
        owner->set_mark(mark->set_bias_epoch(cur_epoch));
      }
    }
  }
}

// At this point we're done. All we have to do is potentially
// adjust the header of the given object to revoke its bias
revoke_bias(o, attempt_rebias_of_object && klass->prototype_header()->has_bias_pattern(), true, requesting_thread, NULL);
```

在bulk rebias过程中，首先会将类对象的epoch值加1，然后遍历所有线程的栈，找到所有该类对象的实例，将它们的epoch值加1,最后会移除掉锁对象的偏向信息。

如果你想查看`bulk revoke bias`的过程以及结果，你可以使用 这个回答(https://stackoverflow.com/questions/46312817/does-java-ever-rebias-an-individual-lock)中的代码

3. 如果类对象的撤销次数等于`BiasedLockingBulkRevokeThreshold`，则认为类对象不合适使用偏向锁，因此要做bulk revoke

代码和上面的类似，我就不贴出来了，主要做下面两件事情

+ 将类对象的prototype header设置为不可偏向状态
+ 遍历所有线程的栈,找到所有类的实例，修改mark word的状态位为001以及对应的lock record,并将偏向锁修改为轻量级锁


### 轻量级锁

轻量级锁的代码实现是在`slow_enter`方法里面

```c++

void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {

  // 省略其他代码。。。

  // 判断是否是无锁状态
  if (mark->is_neutral()) {

    // 直接把mark保存到BasicLock对象的_displaced_header字段
    lock->set_displaced_header(mark);
    
    // 通过CAS将mark word更新为指向BasicLock对象的指针，更新成功表示获得了轻量级锁
    if (mark == obj()->cas_set_mark((markOop) lock, mark)) {
      return;
    }
  }

  ...
}

class BasicLock {
  private volatile markOop _displaced_header;
}
```

首先判断Mark Word是否是中立的，即Mark Word的最后三个字节的值是否为1(001),如果是中立的，则表示此时处于未锁定，且不可偏向。

因此首先会将锁对象的mark word放入到lock对象(这就是我们常说的Lock Record)的`displaced_header`属性中，然后使用CAS将对象的Mark Word更新为指向Lock Record的指针，如果更新成功，表示这个线程就拥有了该对象的锁。并且Mark Word的锁标志位(Mark Word的最后2bit)将转变为00,即表示此对象处于轻量级锁定状态。

### 重量级锁

如果CAS更新失败，就会膨胀称为重量级锁了，锁标志的状态值也变成10，Mark Word中存储的就是指向重量级锁的指针，后面等待锁的线程也要进入阻塞状态。

```c++
ObjectSynchronizer::inflate(THREAD,
                          obj(),
                          inflate_cause_monitor_enter)->enter(THREAD);

```

inflate主要是一些状态的判断，看注释还是比较容易理解的，我们重点看下enter函数中的执行逻辑

```c++

void ObjectMonitor::EnterI(TRAPS) {

...

  // 已经是锁的持有者直接返回
  if (Self->is_lock_owned ((address)cur)) {
    assert(_recursions == 0, "internal state error");
    _recursions = 1;
    _owner = Self;
    return;
  }


  // 为了避免昂贵的线程阻塞、唤醒等操作，会在进入阻塞状态前先自适应自旋
  if (TrySpin(Self) > 0) {
    assert(_owner == Self, "invariant");
    assert(_recursions == 0, "invariant");
    assert(((oop)(object()))->mark() == markOopDesc::encode(this), "invariant");
    Self->_Stalled = 0;
    return;
  }

  ...
}

```

在重量级锁的判定中，不会马上去申请锁，而是会先自适应自旋几次看能否获取到锁，如果不能再去申请锁。
> 自适应的自旋锁,它会由前一次在同一个锁上的自旋时间以及锁的拥有者状态来决定,如果同一个锁上自旋刚获得,那么就认为这次也有很大几率获取到,就多自旋几次,如果对于某个锁说,自旋很少获取到,就认为没戏,就不自旋了,直接去挂起了。

```c++
for (;;) {
    if (TryLock(Self) > 0) break;
    ...
    if ((SyncFlags & 2) && _Responsible == NULL) {
      Atomic::replace_if_null(Self, &_Responsible);
    }
    // park self
    if (_Responsible == Self || (SyncFlags & 1)) {
      TEVENT(Inflated enter - park TIMED);
      Self->_ParkEvent->park((jlong) recheckInterval);
     
      recheckInterval *= 8;
      if (recheckInterval > MAX_RECHECK_INTERVAL) {
        recheckInterval = MAX_RECHECK_INTERVAL;
      }
    } else {
      TEVENT(Inflated enter - park UNTIMED);
      Self->_ParkEvent->park();
    }
    if (TryLock(Self) > 0) break;
    ...
}

```

然后再去申请锁之前还要自旋(贼心不死)，最后没成功才会park当前线程，而park的实现就是我们之前文章提到过的pthread的实现。

```
void os::PlatformEvent::park() {
  ...
  int status = pthread_mutex_lock(_mutex);
  ...
  status = pthread_cond_wait(_cond, _mutex);
  ...
  status = pthread_mutex_unlock(_mutex);
}

```
自旋状态还带来另外一个副作用，那便是不公平的锁机制。处于阻塞状态的线程，并没有办法立刻竞争被释放的锁。然而，处于自旋状态的线程，则很有可能优先获得这把锁。


当线程获取到重量级锁之后就可以执行方法了，但是即使锁被释放之后也不会被恢复到最初的那种无锁状态了

### 好消息和坏消息

我们可以看到偏向锁非常之负责，为了支持偏向锁整个代码复杂度大幅度提升，而许多受益于偏向锁的应用程序都是早期Java集合api，比如HashTable,Vector等。

所以好消息是在JDK15就把偏向锁禁用了，并在以后删除它。

坏消息是现在大部分应用使用的都是JDK8,并且还会使用很多年。


### 巨人的肩膀

1.《深入理解JVM虚拟机》
2.《深入拆解Java虚拟机》
3. https://stackoverflow.com/questions/46312817/does-java-ever-rebias-an-individual-lock
4. http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.94.8487&rep=rep1&type=pdf
5. https://createchance.github.io/post/java-并发之基石篇
6. https://zhuanlan.zhihu.com/p/34662715
7. https://www.zhihu.com/question/55075763