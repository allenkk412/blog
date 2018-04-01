---
title: NetFPGA--SDN交换机板卡配置
date: 2017-03-11 18:59:53
tags: [SDN]
---
#### 配置环境

操作系统：CentOS 5.6 （已失去官方支持，软件源已过期）

NetFPGA 1G 

OpenFlow 1.0

#### 相关资料：

NetFPGA官网：https://netfpga.org/site/#/

NetFPGA Wiki：https://github.com/NetFPGA/netfpga/wiki/Guide

POX Wiki：https://openflow.stanford.edu/display/ONL/POX+Wiki

POX Sources Codes：http://nullege.com/codes/search/pox


```
#1、机器必须能上网。
#2、必须进入root权限进行操作。
#3、将文件夹“netfpga_temp”放置到/usr/local/文件夹下。
#4、板卡所在操作系统启动时的grub引导必须在kernel载入前输入uppermem 524288；在kernel传参数时，要输入参数vmalloc=256M。

# 进入jdk-6u6-linux-i586-rpm.bin所在目录，并执行命令增加文件权限。
cd /usr/local/netfpga_temp/
chmod 777 jdk-6u6-linux-i586-rpm.bin
chmod 777 rpmforge-release-0.3.6-1.el5.rf.i386.rpm
chmod 777 netfpga-repo-1-1_centos5.noarch.rpm

# 安装JDK并配置环境变量。命令如下：
./jdk-6u6-linux-i586-rpm.bin
export JAVA_HOME=/usr/java/jdk1.6.0_06
export LASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin

# 导入NetFPGA平台关联文件软件源。命令如下：
rpm --import http://jpackage.org/jpackage.asc
cd /etc/yum.repos.d
wget http://www.jpackage.org/jpackage17.repo


yum -y --enablerepo=jpackage-generic-nonfree install java-1.6.0-sun-compat.i586
/usr/sbin/alternatives --config java
#选择1.6.0-sun版本

# 安装rpmforge-release-0.3.6-1.el5.rf.i386.rpm
cd /usr/local/netfpga_temp/
rpm --import http://apt.sw.be/RPM-GPG-KEY.dag.txt
rpm -K rpmforge-release-0.3.6-1.el5.rf.i386.rpm
rpm -i rpmforge-release-0.3.6-1.el5.rf.i386.rpm
yum check-update
rpm -Uhv netfpga-repo-1-1_centos5.noarch.rpm

# 安装NetFPGA基础文件包并将其环境参数加入系统
yum -y install netfpga-base
/usr/local/netfpga/lib/scripts/user_account_setup/user_account_setup.pl

# 编译安装。命令如下：
cd /usr/local/netfpga  make
make install
lsmod | grep nf2


# 检查安装过程，若没出现Error字样，则说明安装顺利。若出现Error，则说明依赖包不完整或者没有在根权限下面运行。
# 初步检测。执行命令“lsmod | grep nf2”，若显示“nf2 23436 0”之类的字符，则说明驱动安装成功。

```