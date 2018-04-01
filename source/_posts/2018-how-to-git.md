---
title: how to git
date: 2017-03-01 13:51:08
tags: git
---
## __前言__
通常我们在完成项目的过程中，往往是在个人电脑上编写好代码，而后部署到生产服务器或者个人VPS中去运行。　　

本文使用Github的服务器进行项目代码托管，并以此实现PC与服务器端的源码同步。

以个人博客源码同步为例

## __配置SSH与Git__
### __生成SSH Key__
```bash
ssh-keygen -t rsa -C your_email@youremail.com
```
可在用户主目录 `~/` 下生成`.ssh`文件夹，存放SSH的密钥对，其中`id_rsa`是私钥，`is_rsa.pub`是公钥。

<!--more-->

## __导入SSH Key至Git服务器__

以Github为例，登录Github，右上角 头像 -> `Settings` —> `SSH nd GPG keys` —> `New SSH key` 。把公钥粘贴到key中，填好title并点击 `Add SSH key`

2.上传blog到git：此项建议先在blog进度最新的PC上进行，否则会有版本冲突，解决也比较麻烦。在PC上建立git ssh密钥连接和建立新库respo在此略过：    
* 编辑`.gitignore`文件：`.gitignore`文件作用是声明不被git记录的文件，blog根目录下的`.gitignore`是hexo初始化是创建的，可以直接编辑，建议`.gitignore`文件包括以下内容：      

```
.DS_Store      
Thumbs.db      
db.json      
*.log      
node_modules/      
public/      
.deploy*/
```
`public`内的文件可以根据`source`文件夹内容自动生成的，不需要备份。其他日志、压缩、数据库等文件也都是调试等使用，也不需要备份。

初始化仓库：
```
git init    
git remote add origin <server>
```
`server`是仓库的在线目录地址，可以从git上直接复制过来，`origin`是本地分支，`remote add`会将本地仓库映射到托管服务器的仓库上。

添加本地文件到仓库并同步到git上：
```
git add . #添加blog目录下所有文件，注意有个'.'(.gitignore里面声明的文件不在此内)    
git commit -m "hexo source first add" #添加更新说明    
git push -u origin master  #推送更新到git上
```

至此，git库上备份已完成。

3.将git的内容同步到另一台电脑：假设之前将公司电脑中的blog源码内容备份到了git上，现在家里电脑准备同步源码内容。**注意**，在同步前也要事先建好hexo的环境，不然同步后本地服务器运行时会出现无法运行错误。在建好的环境的主目录运行以下命令：
```
git init  #将目录添加到版本控制系统中    
git remote add origin <server>  #同上    
git fetch --all  #将git上所有文件拉取到本地    
git reset --hard origin/master  #强制将本地内容指向刚刚同步git云端内容
```
`reset`对所拉取的文件不做任何处理，此处不用`pull`是因为本地尚有许多文件，使用`pull`会有一些**版本冲突**，解决起来也麻烦，而本地的文件都是初始化生成的文件，较拉取的库里面的文件而言基本无用，所以直接丢弃。

4.家里电脑生成完文章并部署到服务器上后，此时需要将新的blog源码文件更新到git托管库上，不然公司电脑上无法获取最新的文章。在本地文件中运行以下命令：

```
git add . #将所有更新的本地文件添加到版本控制系统中
```
此时可以使用`git status`查看本地文件的状态。然后对更改添加说明更推送到git托管库上：

```
git commit -m '更新信息说明'  
git push
```
至此，家里电脑更新的备份完成。在公司电脑上使用时，只需先运行:
```
git pull
```