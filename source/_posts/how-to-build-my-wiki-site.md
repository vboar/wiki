---
title: 如何构建我的Wiki站点
date: 2018-11-05 13:13:06
categories:
  - 技术记录
tags:
  - Linux
  - 步骤向
  - Travis CI
  - Hexo
---

> 书山有路勤为径，学海无涯苦作舟。

从大学到现在，都不知道搭建了多少个博客了，然而每次都是以断更而告终。这些日子一直在思考，从实习到现在已经工作了两年了，自己究竟在技术上提升了多少。提升是有的，但是远远不够。于是便决定了，第一步就是要建立自己的wiki系统，把平时阅读、学习、思考、编码的精髓记录下来，在过程中进一步理解熟悉，后续还可以用于查阅。

参考了一些开源的wiki系统后，还是决定采用和之前博客一样的hexo来搭建wiki站点，结合Travis CI实现持续集成和部署。

下面记录一下构建wiki站点的整个过程，方便后续出问题了要重新搭建……

<!-- more -->

## 1. 环境准备


### 云服务器（腾讯云）

正常可以通过用户名和密码登录，如果需要SSH登录则需要到控制台创建SSH密钥，然后将私钥下载到本地保存（或者在本地生成密钥，将公钥添加到控制台），然后就可以用xshell等SSH登录到服务器了。

腾讯云的服务器还有安全组策略，注意要去看一下，开通相应的端口访问。

### nginx

[nignx](http://nginx.org/)

```
// 基于APT源安装（方便）
sudo apt-get update
sudo apt-get install nginx
```

nginx文件和目录：
- /usr/sbin/nginx：执行程序
- /etc/nginx/：配置文件目录
- /var/log/nginx/：日志目录
- /var/www/html/：默认静态文件目录

nginx配置在：/etc/nginx/sites-available/default，修改后需要重新加载配置生效。

```
// 重新加载配置|重启|停止|退出
sudo nginx -s reload|reopen|stop|quit
// 测试配置是否有语法错误
sudo nginx -t
// 启动
sudo nginx
```

### 域名与HTTPS

安装完nginx之后就可以通过IP来访问到nginx的默认页面了，如果要通过域名访问，并且要上HTTPS，其实不需要手动更改default文件中太多的配置，利用[Let's Encrypt](https://letsencrypt.org/)可以轻松完成。

教程具体参考[How To Secure Nginx with Let's Encrypt on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-14-04)。

1. 安装certbot
  ```
  sudo add-apt-repository ppa:certbot/certbot
  sudo apt-get update
  sudo apt-get install python-certbot-nginx
  ```

2. 配置nginx
  ```
  sudo vi etc/nginx/sites-available/default

  server_name kasswang.cn www.kasswang.cn

  sudo nginx -t
  sudo nginx -s reload
  ```

3. 获得SSL证书
  ```
  sudo certbot --nginx -d kasswang.cn -d www.kasswang.cn
  ```
  这里可以选择HTTP是否强制重定向到HTTPS。

4. 验证证书自动续期
  ```
  sudo certbot renew --dry-run
  ```
  证书自动续期一般不用我们操作，cerbot每两天检查一次。

## 2. 使用Hexo

[Hexo]()是一个流行的静态博客系统，支持Markdown语法。一般的写作流程是这样的：使用Markdown语法完成写作后，运行Hexo命令生成静态页面，然后将静态页面部署到服务器上。

先在Github上创建一个wiki仓库，如[vboar/wiki](https://github.com/vboar/wiki)，用于存储wiki系统的source，拉到本地。

确保node环境正常，全局安装hexo-cli，然后初始化项目：
```
cnpm install hexo-cli -g
hexo init wiki
cd wiki
cnpm install
hexo server
```

安装[Next](https://github.com/theme-next/hexo-theme-next)主题：
```
git clone https://github.com/theme-next/hexo-theme-next themes/next

# 主题作为Git子模块引用，以便主题更新
git submodule add https://github.com/theme-next/hexo-theme-next themes/next
```

配置文件：
- 项目的配置在：./_config.yml
- 主题的配置在：./source/_data/next.yml

配置好.gitignore，只提交源代码，不提交build的产物和其他不必要的文件。

然后就可以愉快地开始写文章了。

## 使用Travis CI

写完文章后，我们就可以提交到Github上，然后怎么部署到我们的服务器上呢？在本地构建好拷贝到服务器？利用Github的Webhook自己实现拉取代码到服务器然后构建？不，使用[Travis CI](https://www.travis-ci.org/)可以快速地帮助我们实现持续集成和部署，每提交一次代码，就自动构建一次，然后服务器自动拉取构建产物，就实现自动部署了。

首先需要到Travis CI网站上，开启该项目的持续集成。

.travis.yml如下：
```yml
language: node_js
node_js:
- stable
branches:
  only:
  - master
before_install:
- git config --global user.name "vboar"
- git config --global user.email "vboar@live.com"
- openssl aes-256-cbc -K $encrypted_ffa94ce55f12_key -iv $encrypted_ffa94ce55f12_iv -in .travis/id_rsa.enc -out ~/.ssh/id_rsa -d
- chmod 600 ~/.ssh/id_rsa
- eval $(ssh-agent)
- ssh-add ~/.ssh/id_rsa
- cp .travis/ssh_config ~/.ssh/config
- git clone https://github.com/theme-next/hexo-theme-next themes/next
install:
- npm install hexo-cli -g
- npm install 
script:
- hexo clean
- hexo generate && rsync -az -vv --delete -e 'ssh' public/ ubuntu@kasswang.cn:/var/www/wiki
addons:
  ssh_known_hosts: kasswang.cn
```