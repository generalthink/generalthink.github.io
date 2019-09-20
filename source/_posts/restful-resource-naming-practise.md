---
title: RESTful资源命名最佳实践
date: 2019-09-20 12:08:37
tags: RESTful
---

在Rest中，数据的呈现方式叫做资源(Resource)。拥有强大而一致的REST资源命名策略,是最好的设计决策。

一个资源可以是单个的也可以是一个集合。比如customers是一个集合，而customer是单个资源。我们可以定义customers这个集合的资源的URI是`/customers`,而单个customer资源的URI是`/customers/{customerId}`。

资源也可以包含子集合的资源。比如，使用`/customers/{customerId}/accounts` 来表示某个customer下的account集合资源。同样的，对于account集合资源下的单个account我们可以定义成这样:`/customers/{customerId}/accounts/{accountId}`

REST API使用统一资源标识符（URI）来寻址资源。REST API设计者应该创建URI，将REST API的资源模型传达给潜在的客户端开发人员。当资源命名良好时，API直观且易于使用。如果做得不好，那么相同的API会感觉难以使用和理解。

<!--more-->

### 使用名词来表示资源

RESTful URI应该引用作为事物（名词）的资源而不是引用动作（动词），因为名词具有动词不具有的属性 - 类似于具有属性的资源。资源的一些示例是：

1. 系统的用户
2. 用户账户(银行的场景): 
3. 网络设备

它们的资源URI可以被设计成下面这样:

```
http://api.example.com/device-management/managed-devices 
http://api.example.com/device-management/managed-devices/{device-id} 
http://api.example.com/user-management/users/
http://api.example.com/user-management/users/{id}

```

为了更好的说明我们把资源原型分为四个种类(document,collection,store 以及 controller),你应该总是把资源放到其中一个原型中，并且遵守它的统一命名。



#### document

文档资源是一种类似于对象实例或数据库记录的单一概念(比如mysql中的一行记录,Mongodb中的document),在REST中，你可以将其视为资源集合中的单个资源。文档的状态表示通常包括具有值的字段和指向其他相关资源的链接。

使用单数名称表示文档资源原型

```
http://api.example.com/device-management/managed-devices/{device-id}
http://api.example.com/user-management/users/{id}
http://api.example.com/user-management/users/admin

```

#### collection

集合资源是服务端管理的资源目录。客户可以建议将新资源添加到集合中。但是，要由集合选择是否创建新资源。集合资源选择它想要包含的内容，并决定每个包含的资源的URI。

使用复数名称表示集合资源原型

```
http://api.example.com/device-management/managed-devices
http://api.example.com/user-management/users
http://api.example.com/user-management/users/{id}/accounts

```

#### store

store是**客户端管理的资源库**,store资源允许API客户端放入资源，取出资源，并决定何时删除它们。store永远不会生成新的URI。相反，每个存储的资源都有一个客户端在最初放入存储时选择的URI。

使用复数名称表示store资源原型

```
http://api.example.com/cart-management/users/{id}/carts
http://api.example.com/song-management/users/{id}/playlists

```

#### controller

controller资源有点像程序的概念,controller资源就像可执行函数，带有参数和返回值;输入和输出。

使用动词表示controller原型

```
// 查看用户的信用卡
http://api.example.com/cart-management/users/{id}/cart/checkout

// 播放整个播放列表
http://api.example.com/song-management/users/{id}/playlist/play
```

这里的controller为什么要用动词呢?其实大家可以想象下Spring中Controller做了什么事情,它调用了service组合成各个业务逻辑,将数据组合起来之后进行返回.

### 一致性

使用一致的资源命名约定和URI格式，以最小化和最大可读性和可维护性。你可以实现以下设计提示以实现一致性：


#### 使用正斜杠（/）表示层次关系

正斜杠（/）字符用于URI的路径部分，以指示资源之间的层次关系

```
http://api.example.com/device-management
http://api.example.com/device-management/managed-devices
http://api.example.com/device-management/managed-devices/{id}
http://api.example.com/device-management/managed-devices/{id}/scripts
http://api.example.com/device-management/managed-devices/{id}/scripts/{id}
```

#### 不要在URI中使用尾部正斜杠（/）

作为URI路径中的最后一个字符，正斜杠（/）不会添加语义值，并可能导致混淆。最好完全放弃它们

```
http://api.example.com/device-management/managed-devices/

/*这个版本更好*/
http://api.example.com/device-management/managed-devices

```

#### 使用连字符（ - ）来提高URI的可读性

要使你的URI易于扫描和解释，请使用连字符（ - ）字符来提高长路径段中名称的可读性。

```
// 更好可读性
http://api.example.com/inventory-management/managed-entities/{id}/install-script-location

// 可读性不够高
http://api.example.com/inventory-management/managedEntities/{id}/installScriptLocation
```

#### 不用使用下滑线 _

可以使用下划线代替连字符作为分隔符 - 但是根据应用程序的字体，下划线 _ 字符可能会在某些浏览器或屏幕中被部分遮挡或完全隐藏。为避免这种混淆，请使用连字符 - 而不是下划线 _。

```
// 更具可读性
http://api.example.com/inventory-management/managed-entities/{id}/install-script-location

// 更容易出错
http://api.example.com/inventory_management/managed_entities/{id}/install_script_location

```

#### 在URI中使用小写字母

方便时，URI路径中应始终首选小写字母。

[RFC 3986](http://www.rfc-base.org/txt/rfc-3986.txt)将URI定义为区分大小写，但协议和host除外

```
// 1
http://api.example.org/my-folder/my-doc
// 2
HTTP://API.EXAMPLE.ORG/my-folder/my-doc
// 3
http://api.example.org/My-Folder/my-doc
```

在上面的例子中，1和2是相同的，但3不是,因为它使用大写字母的My-Folder。

#### 不要使用文件扩展名

文件扩展名看起来很糟糕，不会增加任何优势。删除它们也会减少URI的长度。没理由保留它们。除了上述原因，如果你想使用文件扩展突出显示API的媒体类型，那么你应该依赖于通过Content-Type标头传达的媒体类型来确定如何处理正文的内容。

```
// 不要这样用
http://api.example.com/device-management/managed-devices.xml

// 正确的URI
http://api.example.com/device-management/managed-devices

```

### 切勿在URI中使用CRUD函数名称

URI不应用于指示执行CRUD功能。URI应该用于唯一标识资源，而不是对它们的任何操作。应使用HTTP请求方法来指示执行哪个CRUD功能。

```
// 获取所有设备
HTTP GET http://api.example.com/device-management/managed-devices
// 创建新设备
HTTP POST http://api.example.com/device-management/managed-devices

// 根据给定id获取设备
HTTP GET http://api.example.com/device-management/managed-devices/{id}

// 根据给定id更新设备
HTTP PUT http://api.example.com/device-management/managed-devices/{id}

// 根据给定id删除设备
HTTP DELETE http://api.example.com/device-management/managed-devices/{id}

```

### 使用查询组件过滤URI集合

很多时候，你会遇到需要根据某些特定资源属性对需要排序，过滤或限制的资源集合的要求。为此，不要创建新的API  - 而是在资源集合API中启用排序，过滤和分页功能，并将输入参数作为查询参数传递

```
http://api.example.com/device-management/managed-devices
http://api.example.com/device-management/managed-devices?region=USA
http://api.example.com/device-management/managed-devices?region=USA&brand=XYZ
http://api.example.com/device-management/managed-devices?region=USA&brand=XYZ&sort=installation-date

```

