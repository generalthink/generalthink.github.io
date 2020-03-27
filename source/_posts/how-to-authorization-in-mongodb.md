---
title: MongoDB中如何管理用户权限?
date: 2020-03-26 10:41:34
tags: MongoDB
---

你还在用root账户来访问你程序的所有数据吗？午夜梦回,你是否有过担心,要是手误操作,一不小心就删库了，应该怎么办? 

如果你真有这样的担忧,那么现在开始就让你的用户权限小一点。


mongodb有下面四种认证方式

1. 用户名+密码 : 默认认证方式,用户信息存储于MongoDB本地数据库
2. 证书方式 : 采用X.509标准,服务端需要提供证书文件启动，客户端需要证书文件连接服务端,证书由内部或外部CA颁发
3. LDAP外部认证 : 企业版功能。连接到外部LDAP服务器认证
3. Kerberos外部认证 : 企业版功能, 连接到外部Kerberos服务器认证

而今天我们探究的是最简单的也是最常用的一种：用户名+密码。

<!--more-->

我们一般说到权限控制,都绕不过RBAC:基于角色的访问控制(Role-Based Access Control)，MongoDB中也不外如是。
> RBAC实际很简单,就是一个用户有哪些角色,而这些角色都有哪些资源。 这样就可以使得拥有对应角色的用户拥有对应的权限。

MongoDB中也存在这么三个概念,User,Role以及Action。

User,Role好理解,就是用户和角色，那么Action是啥玩意呢？Action实际上是用户能对数据库做什么操作，比如增删改查等
>更多Action可以查看：https://docs.mongodb.com/manual/reference/privilege-actions/


### 启用认证

默认情况下,mongodb安装完成后是不启用认证机制的,这个时候你没有用户名,密码也可以登录进去，并且你拥有操作数据库的任何权限(这个时候你的权限最大，也最危险)。

虽然启用认证后,不指定用户名，密码,也可以登录，但是此时只能创建用户，而不能做其他的操作。

启用认证有两种方式，

1. 在mongod.cfg中指定

```yaml
security:
  authorization: enabled
```
> https://docs.mongodb.com/manual/reference/configuration-options/#security-options

2. 通过命令行参数指定

```
mongod --auth --port 27017 --dbpath /data/db
```

此时如果你试图做其他操作，比如查询数据就会遇到下面的错误:

```
db.demo.find();
Error: error: {
  "ok" : 0,
  "errmsg" : "not authorized on test to execute command { find: \"demo\", filter: {}, $db: \"test\" }",
  "code" : 13,
  "codeName" : "Unauthorized"
}

```

此时我们可以选择创建用户，比如我们创建一个root用户

```
use admin;

db.createUser({
  user: "root", 
  pwd: "123456", 
  roles: [
   {
    role: "root", 
    db:"admin"
   }
  ]
 } 
)
```
上面的命令创建了一个root账户,role是root(对所有数据库都拥有最高权限，下面会讲到mongodb内置有哪些角色)。

此时使用下面的命令来登录mongodb

```
 mongo -u root -p 123456 --authenticationDatabase admin
```
设置以auth方式登陆之后，client端通过mongo登陆mongodb，是必须加上“--authenticationDatabase”选项的，“authenticationDatabase”指定了校验用户账户名和密码的数据库,我们一般存储在admin库中。

然后使用下面的命令可以查看对应的授权机制

```
db.runCommand({getParameter: 1, authenticationMechanisms: 1})
{
  "authenticationMechanisms" : [
          "MONGODB-X509",
          "SCRAM-SHA-1",
          "SCRAM-SHA-256"
  ],
  "ok" : 1
}
```
当使用Robo 3T工具连接的时候，对应用户的授权机制一定要是上面的其中一种。


### MongoDB内置角色以及权限继承关系

![内置角色](/images/mongodb/mongodb-role-extends.png)

可以看到root角色是处于顶层位置的。

当你想知道某个角色有哪些权限的时候可以使用getRole的命令
```
db.getRole('read', {showPrivileges: true});

```
getRole命令会给出角色对应的权限,继承下来的权限以及对应的Action

同样的，mongodb也是支持我们自建角色的。

```
// 创建 sampleRole角色
// 作用在sampledb中的sample Collection,只能拥有read,update操作
db.createRole(
{
  "role": "sampleRole",
  "privileges": [
    {
      "resource": {
        "db": "sampledb",
        "collection": "sample"
      },
      "actions": [
        "update"
      ]
    }
  ],
  "roles": [
    {
      "role": "read",
      "db": "sampledb"
    }
  ]
}
);

// 创建用户绑定角色
db.createUser(
{
  "user": "sampleUser",
  "pwd": "password",
  "roles": [
    {
      "role": "sampleRole",
      "db": "admin"
    }
  ]
}
)

```

当我们使用sampleUser登录的时候，就不能再向sample Collection中插入数据了。

### 创建应用用户

我们在实际使用的使用，可以根据不同的使用场景创建不同的用户分配不同的角色，这样可以使得权限控制更小，更加安全。

#### 创建只读用户

```
db.createUser({user: "reader", pwd: "abc123", roles: [{ role:"read", db: "mydb" }]})
```

#### 创建读写用户

```
db.createUser({user: "writer", pwd: "abc123", roles: [{ role:"readWrite", db: "mydb" }]})
```