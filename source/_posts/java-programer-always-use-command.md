title: Java开发者常用的线上命令
date: 2018-04-27 20:52:16
tags: 
	- Java开发常用命令
	- Linux命令
categories: Java开发命令
description: 在开发过程中，遇到一些问题常常需要在Linux下进行定位分析，查看日志、分析堆栈情况、分析网络状况等，但是Linux系统下命令不胜繁多，这里总结了写自己在开发过程中常用的命令。
keywords: Java开发命令、Linux常用命令、Java问题排查常用命令
---


### 1. Java命令的使用
jps、jconsole、jvisualvm等工具的数据来源是这个文件(/tmp/hsperfdata_userName/pid)。所以当该文件不存在或是无法读取时就会出现jps无法查看该进程号,jconsole无法监控等问题,如果该文件不存在或者没有权限均会导致这些命令无法使用,java启动时提供了参数(-Djava.io.tmpdir),可以对这个文件的位置进行设置,而jps、jconsole都只会从/tmp目录读取,而无法从设置后的目录读取信息。

___

1. 显示当前所有java进程

```
./jdk/bin/jps -v
```
2. 查看当前时刻java线程状态

```
./jdk/bin/jstack pid
jstack -F pid   强制性获取线程状态
```


**查看产生的线程文件，主要有以下几种线程修饰**

```
locked <地址> 目标：使用synchronized申请对象锁成功,监视器的拥有者。

waiting to lock <地址> 目标：使用synchronized申请对象锁未成功,在进入区等待。

waiting on <地址> 目标：使用synchronized申请对象锁成功后,释放锁幵在等待区等待。
```

**开发过程中，经常会遇到线程状态,线程状态有如下几种**
```
runnable:状态一般为RUNNABLE。

in Object.wait():等待区等待,状态为WAITING或TIMED_WAITING。

waiting for monitor entry:进入区等待,状态为BLOCKED。

waiting on condition:等待区等待、被park。

sleeping:休眠的线程,调用了Thread.sleep()。
```

**需要注意的几点:**

 + IO操作是可以以RUNNABLE状态达成阻塞。例如:数据库死锁、网络读写。 格外注意对IO线程的真实状态的分析。 一般来说,被捕捉到RUNNABLE的IO调用,都是有问题的。
 + wait on monitor entry： 被阻塞的,肯定有问题
 + in Object.wait()： 注意非线程池等待

#### 2. 查看java堆内存的细节
1. 查看对内存使用情况

```
jmap -heap pid
```
2. 查看堆内存对象的数量以及大小

```
jmap -histo pid
jmap -histo:live pid   查看存活的对象的数量以及大小,会先进行一次full gc
```
3. 将内存的统计信息输出到文件进行分析

```
jmap -dump:format=b,file=heapDump  将内存使用的详细情况输出到文件，后期可以使用mat分析
```
4. 监控java运行状态信息

```
jstat -gcutil 250 5   统计gc信息，每250ms统计一次,总共统计5次
```

### 2. Linux命令的使用
大多数的Java应用都运行在Linux下,所以在很多时候分析Java问题的时候免不了使用Linux帮助辅助分析，这里给出几个常用的Linux命令介绍

#### 1. vi命令
经常我们需要修改配置文件，这个时候vi就派的上用场了，但是vi命令这个参数太多了，也记不住，我一直都是需要用的时候再查，但是经常使用下来发现其实常用的也就那么几种。

 1. vi基本分为一般模式、编辑模式与命令行模式

    + 一般打开一个文件就是一般模式
    + 在一般模式中,按下“i,I,o,O,A,a,r,R"等任何一个字母进入编辑模式,按下ESC退出到编辑模式进入一般模式
    + 在一般模式中按下“:、/、?"三个中的任何一个按钮,就可以将光标移动到最下面那一行,进入命令行模式

2. 一般模式下常用按键
    + G : 移动到这个文件的最后一行
    + nG : n为数字,移动到这个文件的第n行。例如20G则会移动到第20行(可以配合:set nu)
    + /word : 向下寻找一个名称为word的字符串,例如你要在文件中查找config这个单词就输入/config
    + ?word : 向上寻找一个名称为Word的字符串
    + n : 这个n是英文按键,代表重复前一个查找的操作
    + N : 这个N是英文按钮,代表反向进行前一个查找的操作
    + :n1,n2s/word1/word2/g : n1与n2为数字,代表在n1与n2行之间查找word1这个串,然后替换成n2这个字符串
    + dd : 删除光标所在的那一整行
    + yy : 复制光标所在的那一行
    + p,P : p为将以复制的数据在光标下一行粘贴,P则为在上一行粘贴

3. 一般模式切换到命令行模式的可用按钮

    + :q! : 修改过不保存直接离开,!表示强制
    + :wq : 保存后离开,也可以用:wq!
    + ZZ : 大写的Z,如果没有改动则不保存离开,若已经改动过,则保存后离开
    + :w[filename] : 将编辑的数据保存成另一个文件
    + :set nu : 显示行号
    + : set nonu : 取消行号

#### 2. 文件操作相关命令

```
1. ls -a : 全部的文件,连同隐藏文件(开头为.的文件)
2. cp -a source destination 或者   cp [参数] source1 source2 source3 ... directory    说明: -a的意思是和复制之前的文件属性一模一样
3. rm -f ： 强制删除文件，忽略不存在的文件
4. rm -r : 递归删除,常用于目录的删除,非常危险参数
5. mv [-fiu] source destination 或者      mv [参数] source1 source2 source3 ... directory
	-f:表示强制，如果存在则直接覆盖
	-i:如果已经存在就会询问
	-u:如果已经存在，且source比较新,才会更新

6. cat -n fileName : 显示文件内容，并显示行号
7. tail [-n number] fileName : 显示文件结尾n行内容，比如tail -20 mylog.log
8. tial -f core.log    显示文件最新10行内容，会自动更新
9. find /usr -name fileName   查找usr目录下文件名为fileName的文件，fileName支持正则,比如*.js
10. ls -l | grep file.txt   列出file.txt的文件信息，grep后可以跟震泽表达式
11. tar -czf test.tar.gz /test1 /test2   压缩文件
12. tar -xvzf test.tar.gz  解压文件
```

#### 3. 系统状态相关命令

```
1. ps -aux | grep java 查看java进程
2. top
	按下数字1可以查看每个逻辑CPU的状态
	top -d 2 -n 3  每2s更新一次状态，总共更新3次
	top -p 574   显示指定的进程信息
	%wa 表示I/Owait占用的CPU时间百分比,通常系统变慢都是I/O产生的问题比较大
	swap表示虚拟内存，如果被大量使用表示系统的物理内存实在不足
	S 进程状态。D=不可中断的睡眠状态 R=运行 S=睡眠 T=跟踪/停止 Z=僵尸进程

3. vmstat  如果其中的cs(上下文切换次数)值过高，会导致系统负载过高，排查问题的时候需要用到
4. netstat -anp | grep 8080   查看端口占用情况
```

#### 4. 其他Linux命令

```
1. echo $LANG   查看当前系统编码
2. lsof app.js  查看文件被哪个进程打开了
3. lsof -i :8080 查看哪个端口属于哪个程序
4. lsof -p pid  查看某个进程打开了哪些文件
5. tcpdump -s 0 -i ge1 -A port 8084     监听8084端口收到的数据
6. lsof | wc -l     查看进程打开个多少个文件
7. lsof -p pid |wc -l    查看某个进程打开的文件数
```


### 参考文档

[jstack的使用](http://www.hollischuang.com/archives/110)