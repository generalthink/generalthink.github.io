title: 使用hexo搭建github博客
date: 2015-09-20 22:09:46
tags: [hexo]
categories: learning
description: 使用hexo3搭建属于自己的github博客
keywords: github hexo blog 博客
---

1. 使用工具版本
```
git版本:git version 2.5.2.windows.2
npm版本:2.14.2
hexo: 3.1.1
```
2. github账号,同时新建一个仓库,仓库名称是固定的,格式为: **your_username.github.io**

3. 环境准备
[git](http://git-scm.com/)
[node.js](https://nodejs.org/en/)
**安装注意:**
    ![记住添加到path](/images/hexo-nodejs-add-to-path.png)

4. 打开刚刚安装好的git客户端,然后和github建立SSH连接
    * [建立连接](https://help.github.com/articles/generating-ssh-keys/)

5. hexo安装
```bash
$ npm install hexo-cli -g
$ cd f:  #可以是任何路径
$ hexo init blog
$ cd blog
$ npm install
#3.0版本和2.0版本的区别,3.0已经将发布的程序独立出来了，所以需要安装
$ npm install hexo-deployer-git --save
```

6. 发布
    + 修改blog根目录下的_config.yml文件,将deploy节点修改为如下内容(将**generalthink**替换成自己github的名称):
    ```
  	deploy: 
  	type: git  
  	repo: git@github.com:generalthink/generalthink.github.io.git
  	branch: master
    ```
    + hexo deploy
7. 访问自己的博客,博客地址为:http://your_username.github.io
8. _config.yml文件的配置均为key: value形式,值得一提的是value前面必须要有一个空格
9. 常见错误参考
    * ![deploy错误](/images/hexo-deploy-error.png)
    * 解决方案,这个是由于时间久了SSH连接过期导致的,此时重新建立连接即可,使用 
        ```bash
          $ eval $(ssh-agent -s)
          $ ssh-add ~/.ssh/id_rsa
        ```
	    或者参考<https://help.github.com/articles/generating-ssh-keys/>
10. 建议参考文档:
    1. <https://hexo.io/docs/> 
    2. <http://cnfeat.com/blog/2014/05/10/how-to-build-a-blog/>
    3. <http://ibruce.info/2013/11/22/hexo-your-blog/>