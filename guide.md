# 详细教程

## 基本思路

> 在本地生成静态文件，把静态文件部署到VPS上，用Nginx直接做Web服务，由于hexo支持git的部署方式，从而可以实现从本地更新博客，方便快捷。

## 本地机操作流程(以windows 10为例)

### 安装node.js

直接官网下载，一路默认安装即可

### 安装git

* 官网下载安装，需要配置一下环境变量

* 设置一下用户名及邮箱

  ```
  git config --global user.email "email@example.com"
  git config --global user.name "username"
  ```

* 生成ssh密钥，主要用来本地的git可以连接到vps的git库

  ```
  ssh-keygen -t rsa -C "email@example.com"#一路回车生成公钥和密钥，一会要用到公钥id_rsa.pub
  ```

* 本地创建一个文件夹，用来放置网站的内容

  ```
  在你电脑的任意位置创建一个文件夹（例如E:\hexo，下文以此代替），作为网站目录。
  ```

* 打开cmd,输入如下命令

  ```
  npm install -g hexo-cli
  ```

  但是由于外网的原因，下载很慢，大家可以换成taobao的源，操作如下

  `npm install -g cnpm --registry=https://registry.npm.taobao.org`
  然后等着装完即可，之后的用法和npm一样，无非是把`npm install`改成`cnpm install`。但据我实际操作，下载还是很慢，所以最好还是自备梯子吧。

  下载完后，在cmd中，切入到hexo目录并输入以下命令

  ```
  hexo init
  npm install
  hexo d -fg
  hexo serve
  ```

  打开[http://localhost:4000](http://localhost:4000/) 即可看到你的站点（当然还没有发布到网络）。
  你可以看见`hexo`文件夹下有一个themes文件夹，这是可以自定义的，从而改变网站的呈现形式，[官网](https://hexo.io/themes/)也提供了一些可供选择的主题。

## vps操作流程

### ssh到vps

### 安装git

```
yum update && apt-get upgrade -y #更新内核
yum install git-core
```

### 安装nginx

```
yum install nginx -y 
```

网上的直接这个操作是有问题的，yum里并没有nginx这个包。

所以得把nginx添加到yum里,操作如下

```
rpm -ivh http://nginx.org/packages/centos/6/noarch/RPMS/nginx-release-centos-6-0.el6.ngx.noarch.rpm
```

但是这一解决方案我也失败了，我的做法是这样的

第一步，在/etc/yum.repos.d/目录下创建一个源配置文件nginx.repo：

```
cd /etc/yum.repos.d/
vi nginx.repo
```

并填写以下内容

```
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/6(此处填写操作系统的版本)/$basearch/
gpgcheck=0
enabled=1
```

接着执行命令

```
yum install nginx -y 
```

### 新建git用户添加sudo权限

```
adduser git
chmod 740 /etc/sudoers
vim /etc/sudoers

```

在vi编辑中找到如下内容：

```
## Allow root to run any commands anywhere
root    ALL=(ALL)     ALL

```

在下面添加一行

```
git   ALL=(ALL)     ALL

```

保存并退出后执行

```
chmod 440 /etc/sudoers

```

### 创建git仓库，并配置ssh登录

```
su git
cd ~
mkdir .ssh && cd .ssh
touch authorized_keys
vi authorized_keys//在这个文件中粘贴进刚刚申请的key（在id_rsa.pub文件中）
cd ~ 
mkdir hexo.git && cd hexo.git
git init --bare

```

测试一下，如果在`git bash`中输入`ssh git@VPS的IP地址`,能够远程登录的话，则表示设置成功了。

### 配置git hooks

```
su git
cd /home/git/hexo.git/hooks
vi post-receive
```

输入如下内容后保存退出，

```
#!/bin/bash
GIT_REPO=/home/git/hexo.git #git仓库
TMP_GIT_CLONE=/tmp/hexo
PUBLIC_WWW=/var/www/hexo #网站目录
rm -rf ${TMP_GIT_CLONE}
git clone $GIT_REPO $TMP_GIT_CLONE
rm -rf ${PUBLIC_WWW}/*
cp -rf ${TMP_GIT_CLONE}/* ${PUBLIC_WWW}

```

然后赋予脚本的执行权限

```
chmod +x post-receive

```

### 配置Nginx

```
vim /etc/nginx/conf.d/hexo.conf

插入如下代码：
server {
    listen         80 ;
    root /var/www/hexo;//这里可以改成你的网站目录地址，我将网站放在/var/www/hexo
    server_name example.com www.example.com;//这里输入你的域名或IP地址
    access_log  /var/log/nginx/hexo_access.log;
    error_log   /var/log/nginx/hexo_error.log;
    location ~* ^.+\.(ico|gif|jpg|jpeg|png)$ {
            root /var/www/hexo;
            access_log   off;
            expires      1d;
    }
    location ~* ^.+\.(css|js|txt|xml|swf|wav)$ {
        root /var/www/hexo;
        access_log   off;
        expires      10m;
    }
    location / {
        root /var/www/hexo;//这里可以改成你的网站目录地址，我将网站放在/var/www/hexo
        if (-f $request_filename) {
            rewrite ^/(.*)$  /$1 break;
        }
    }
}

```

重启Nginx

```
service nginx restart
```

## 本机的最后配置

### 配置hexo配置文件

位于`hexo`文件夹下，`_config.yml`,修改`deploy`选项

```
deploy:
  type: git
  message: update
  repo:
    s1: git@VPS的ip地址或域名:git仓库地址,master
```

***这里几个语句格式一定要对齐！！***

接着在`hexo`文件夹内，按住`shift`右击，选择`在此处打开命令窗口`（当然你也可以用`cd`命令），运行`hexo g hexo d`，如果一切正常，静态文件已经被成功的push到了blog的仓库里，如果出现`appears not to be a git repo`的错误，删除`hexo`目录下的`.deploy`后再次`hexo g hexo d`就可以了。

以上，博客已经完全建好了。

## 更新博客

使用一款 MarkDown 编辑器写 Blog 。写完后将文件以 *.md 的格式保存在本地`[网站目录]\source\_posts`中。文件编码必须为 UTF-8，这一点仅 Windows 用户需注意。

每篇 Blog 都有固定的参数必须填写，参数如下，注意每个参数的 : 后都有一个空格。

```
title: title
date: yyyy-mm-dd
categories: category  
tags: tag
#多标签请这样写：  
#tags: [tag1,tag2,tag3]
#或者这样写： 
#tags:
#- tag1 
#- tag2  
#- tag3  
---  
正文  

```

编写完后，只需要在hexo文件夹下执行`hexo g && hexo d`，博客即可更新。

## 错误解决

1.`ssh git@ip`，被拒绝，是远程端口默认为22端口，而不是我VPS的ssh真正端口

解决方法：在.ssh文件夹下（也就是生成公钥的文件夹）创建config文件，输入如下内容：

```
 # alex's git server
Host VPS的IP
HostName VPS的IP
User git
Port SSH端口
IdentityFile ~/.ssh/id_rsa
```

