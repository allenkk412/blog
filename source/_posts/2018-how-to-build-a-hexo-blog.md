---
title: 搭建我的Hexo个人博客
date: 2017-03-01 14:03:52
categories: blog
tags: [hexo,nginx]
---
## __前言__
本文用于记录个人博客配置,记录基本步骤及所到的问题

个人电脑端：CentOS 7

腾讯云服务器：CentOS 7

## __什么是Hexo__
Hexo是基于Node.js的一个博客框架，Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。

## __安装准备__
### __安装Node.js__
[Hexo官方文档](https://hexo.io/zh-cn/docs/index.html)使用cURL和Wget获取nvm（Node.js管理器）来安装Node.js，这里使用yum进行安装。
```bash
# yum install -y nodejs
```

### __安装Git__
```bash
# yum install git-core
```
### __安装Hexo__
使用npm安装Hexo：
```bash
# npm install -g hexo-cli
```

## __搭建Hexo博客__
### __创建站点文件夹__
```bash
$ cd ~
$ mkdir Codes
$ cd Codes
```
### __初始化Hexo__
```bash
$ mkdir blog
$ hexo init blog
```

### __生成静态文件__
```bash
$ cd blog
$ hexo generate
```
***
至此，Hexo博客初始化配置基本完成，在指定的文件夹里新生成目录结构如下：
```
.
├── _config.yml               //网站的配置信息
├── package.json              //应用程序的信息
├── scaffolds                 //模版文件夹。当您新建文章时，Hexo会根据scaffold来建立文件
├── source                    //资源文件夹是存放用户资源的地方
|   ├── _drafts               //草稿文件夹
|   └── _posts                //文章文件夹
└── themes                    //主题文件夹    
```

## __架设Hexo博客__
### __安装Nginx__
```bash
# yum install -y nginx
```
### __配置server__
```bash
# vim /etc/nginx/nginx.conf
```
`http` 块的 `server` 部分 `root` 后值修改为 `/home/username/Codes/blog/public`

### __启动Nginx__
```bash
# service nginx start
```
### __设置权限__
由于权限问题，此时访问域名会返回403错误——服务器上文件或目录拒绝访问。

修改博客根目录为755（rwxr-xr-x）

```bash
# chmod -R 755 /home/username
```

## __本地启动__
在本地预览效果
```bash
hexo g # 等同于hexo generate，生成静态文件
hexo s # 等同于hexo server，在本地服务器运行
```

之后打开浏览器并输入IP地址 `http://localhost:4000/` 查看

## __Hexo常用指令__
```bash
$ hexo init [folder]         #新建一个网站。如果没有设置 folder ，Hexo 默认在目前的文件夹建立网站。
$ hexo new [layout] <title>  #新建一篇文章。如果没有设置 layout 的话，默认使用 _config.yml 中的 default_layout 参数代替。
$ hexo generate              #生成静态文件。
$ hexo server                #启动服务器。默认情况下，访问网址为： http://localhost:4000/。
$ hexo clean                 #清除缓存文件 (db.json) 和已生成的静态文件 (public)。
```
