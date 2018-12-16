title: 从Eclipse到Idea,常用的快捷键
date: 2018-07-11 21:01:49
tags: [Idea,工具]
categories: 工具 
description: 从Eclipse转到Idea,多多少少会有些不习惯，虽然可以使用Eclipse的快捷键，但是开发中总是会在别人的机器上或者别人在你的机器上调测，所以掌握Idea的快捷键很很有必要的，但是一般来说我们掌握常用的即可，其他的即使记不住也可以搜索。
keywords: Idea常用快捷键
---


### 从Eclipse转到Idea ###
我用了七年的Eclipse，熟悉了Eclipse的快捷键，今年转战到Idea,还有有点不喜欢，虽然可以切换到Eclipse的快捷键，但是其他人的不会切换，以后你在别人电脑上调试或者别人帮你看问题的时候就会不喜欢，干脆就不换了，但是有些快捷键确实记不住，而且也很多，就记录下自己常用的快捷键。

```

1. 根据模糊的类名或者方法名查找Java文件： Ctrl+Shift+Alt+N

2. 查找文件或目录: Ctrl+Shift+N后，使用/然后输入目录名称即可

3. 查看所有快捷键: Ctrl+Shift+A(很常用)

4. 在当前窗口弹出某个类的定义: Ctrl+Shift+i

5. 实现接口方法： Ctrl+I

6. 复写或者实现父类的方法: Ctrl+O

7. 快速返回上次查看代码的位置: Ctrl+Alt+方向键

8. 一键格式化代碼: Ctrl+Alt+L

9. 去掉无效的import: Ctrl+Alt+O

9. 删除一行: Ctrl+X  或者Ctrl+Y

10. 查看类或接口的继承关系：Ctrl+H

11. 查找接口的实现类： ctrl + alt +B

12. 复制行: Ctrl+D

13. 快速打开类: Ctrl+N

14. 查看当前类的所有方法: Alt+7  或者Ctrl+F12(这个放方便)

15. 查看下一个方法: ctrl+下方向键

16. 查看上一个方法: ctrl+上方向键

15. 关闭Tab页 : Ctrl+F4

16. 查看方法被都被谁调用了： Ctrl+Alt+H  会显示出哪些类调用了它以及次数

17. 精确查看被谁调用了： Alt+F7

18. 寻找Controller中URL路径： Ctrl+Alt+Shift+H 或者 double shift(search everything)

19. 添加bookmark方法：
    1. 使用Ctrl+F12列出该类的所有方法，定位到你想要加入的方法
    2. 按下F11，将方法加入到bookmark
    3. 按下shift+F11,将bookmark列表弹出来
    4. 在上面可以进行对应的修改


20. 添加代码折叠块
    1. 将光标定位在左边大括号里面，然后使用Ctrl+Shift+.即可
    2. 按下Ctrl加上一个+即可让折叠快修改

21. 大括号匹配： Ctrl+] 或者 Ctrl+[来回定位即可

22. 高亮某个变量： Ctrl+Shift+F7  不会随着鼠标的移动而高亮消失

23. 跳转到父类接口： Ctrl+U

24. 回滚代码： Ctrl+Z  回滚回去： Ctrl+Shift+Z

25. 调试
    1. F9            resume programe 恢复程序
    2. Alt+F10       show execution point 显示执行断点
    3. F8            Step Over 相当于eclipse的f6      跳到下一步
    4. F7            Step Into 相当于eclipse的f5就是  进入到代码
    5. Alt+shift+F7  Force Step Into 这个是强制进入代码
    6. Shift+F8      Step Out  相当于eclipse的f8跳到下一个断点，也相当于eclipse的f7跳出函数
    7. Atl+F9        Run To Cursor 运行到光标处

26. 常用插件

    1. CheckStyle-IDEA   代码风格检测插件
    2. FindBugs-IDEA      寻找代码的bug插件
    3. Lombok plugin      支持lombok的插件
    4. 阿里巴巴规约插件
    5. Maven Helper     查看maven依赖关系插件
    6. CodeGlance   类似于SublimeText的Mini Map插件
    7. Grep Console     给工作台输出上色，根据不同的日志等级设置不同的前景色或者背景色，以及查找等功能
    8. JRebel for IntelliJ  热部署插件修改了代码无需重启
    9. lombok Plugin    使用lombok的都知道
    10.  rainbow brackets    彩虹括号,代码中括号层级太多的时候你就知道它的用处了

```
