---
title: 在git-ops中使用git-crypt保护敏感数据
date: 2020-3-01 09:21:11
tags: git-crypt
---

### 为什么要避免将敏感信息存储在git中？

不要在git仓库中存储任何敏感信息，并且要不惜一切代价这样做，即使仓库是私有的，也不应该将其视为存储敏感信息的安全场所，首先让我们了解为什么它存储敏感信息不安全。


git上如果你将你的仓库声明为public的,那么任何一个人都可以访问你仓库的内容，不仅如此，还可以游览仓库中的所有代码，甚至可以运行它。如果你将你的API秘钥存储在仓库中，那么任何人都可以拿到。

即使是存储在私有仓库，也会面临风险。当你与第三方程序集成的时候，你可能正在向第三方应用打开私有仓库。这些应用程序时可以访问你的私有仓库并阅读其中包含的信息。攻击者就有可能伪装成这些第三方应用来获取你的机密数据(API key,数据库密码等等)。

<!--more-->

### 使用git-crypt来加密你的敏感数据

git-crypt(https://github.com/AGWA/git-crypt)可以在git仓库中对数据进行透明的加解密。你选择保护的文件在提交时会加密，而在检出时会解密。git-crypt使开发者可以自由共享包含公共和私有内容混合的存储库。
git-crypt会正常降级，因此没有密钥的开发人员仍然可以克隆并提交到包含加密文件的存储库。这样一来，就可以将机密资料（例如密钥或密码）与代码存储在同一存储库中，而无需锁定整个仓库。


接下来我会演示如下如何使用git-crypt保护我的重要数据。

#### 安装git-crypt

mac和linux上安装git-crypt都比较简单，windows上比较麻烦，不过有人已经把安装包搞定了，大家可以去这里[下载 ](https://gitee.com/pharaoh/git-crypt-win),当然我后面的仓库中也有。下载下来的是一个exe文件，放到环境变量中即可。

#### 首先clone一个需要使用git-crypt的仓库,我的仓库中有2个文件,其中secret.properties是存放敏感数据的文件

```
$ tree
|-- DbInfo
|   `-- secret.properties
`-- index.html

```

仓库地址是: https://github.com/generalthink/git-crypt-test

DbInfo目录中secret.properties内容如下

```
$ cat DbInfo/secret.properties
mysql.ip=locahost
mysql.port=3306

```
#### 然后初始化你的存储库以便于使用git-crypt,它将生成一个密钥并使得当前的Git存储库可以使用git-crypt

```
git-crypt init
```
#### 检查文件状态

```
git-crypt status
```

![查看状态](/images/git-crypt/status.png)

你可以看到所有的文件都没有被加密

#### 通过创建gitattributes文件来指定要“加密”的文件。你要为每个要加密的文件分配“ filter = git-crypt diff = git-crypt”属性

```
<file-name-to-encrypt> filter=git-crypt diff=git-crypt
```

>需要注意的是，在执行第5步(即将.gitattributes添加到您的存储库)之前，请确保将“敏感文件”移出存储库并提交更改，然后添加.gitattributes文件，然后将“敏感文件”移回存储库。
就像下图这样，首先将敏感文件（在mycase中为secret.properties）移出存储库并提交更改，然后在仓库的根目录中添加.gitattributes，然后将我的“ secret.properties”文件再次添加至仓库。不然,你会收到git抛出的警告错误。

![加密](/images/git-crypt/encrypted.png)


**经过上面的步骤之后你的secret.properties已经被加密了,当然在你本地的git仓库你看到的是解密后的数据。 将它推送到git仓库就可以看到这是加密的数据了**

![github上的显示](/images/git-crypt/secret-in-github.png)


#### 导出秘钥和你的协作者共享
```
git-crypt export-key <key-to-unlock>
```

![导出秘钥](/images/git-crypt/export-key.png)

通过上面的命令，你就可以看到会生成一个git-crypt.key文件，将这个文件分发给你的协作者，当他们checkout这个仓库之后就可以用这个秘钥解密了。

> 别忘了在gitingore文件中忽略 git-crypt.key,以防止意外地将此文件提交到git repo。同时，从这里开始，无论克隆存储库的人是什么，除非他们有解锁它的钥匙，否则他们将无法看到“secret.properties”文件的内容！


### 如何解密仓库中的加密文件

现在，如果你尝试克隆仓库，将无法在仓库中看到加密文件。就像我克隆了测试仓库(我在我本地电脑的另一个目录中clone的)并尝试打开“ secret.properties（加密文件）”一样，这就是我所看到的！

![加密后](/images/git-crypt/after-encryption.png)


你还记得吗？解密前我们是可以看到真正内容的。

现在要解密此文件,你必须具有密钥！请要求管理员为你提供密钥。之后，你必须跳入克隆的仓库中执行下面的命令

```
git-crypt unlock /path/to/key
```

![解密](/images/git-crypt/unlock.png)


### 多说两句

看到这里可能有人会有疑问,开发的时候好使了，那打包的时候咋整呢？这个加密了，在服务器打包之后肯定跑不起来呀,其实解决办法很简单,只需要在打包之前解密就行了。

当然了关于对配置文件加密的方法，还是有很多种的，比如sealed-secrets或者公司内部的加解密服务。

为了便于大家测试我会在将我生成的key提交到git仓库中。
