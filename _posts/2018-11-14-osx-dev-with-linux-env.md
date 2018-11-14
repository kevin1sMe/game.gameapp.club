---
layout: post
title:  "Mac OSX下基于VirtualBox搭建linux开发环境"
author: kevin
categories: tools
tags: osx env develop docker virtualbox
---
* content
{:toc}

>最近在构建基于docker容器化的微服务框架，但mac osx上不方便随时编译测试某些功能（linux相关依赖)。于是从零搭建了个linux开发环境。和mac osx共享目录，代码可以直接在osx下写，编译部署等则可以在linux环境内进行（当然容器化服务也可以在osx下部署）。

#### 基础功能说明
* 基础开发环境
	* C/C++
		* gcc/g++4.8.5
	* python/pip环境
	* docker、docker-compose
* 可类似云服务器''远程"ssh开发
* 可共享本地host主机目录

#### why not docker?
为什么不直接基于docker来构建开发环境呢？我主要考虑：
1. docker相较于虚拟机的稳定可靠性较差
2. docker的环境需要commit才可持久化，烦琐不易维护

#### 前置安装
* 安装VirtualBox最新版本
* 在VirtualBox中安装linux发行版本（例:CentOS7，我下载了Minimal ISO版本 )

#### 开发工具及环境
* gcc、g++
`sudo yum install -y gcc-4.8.5-28.el7.x86_64 gcc-c++`
* 工具链
`sudo yum install -y cmake3 git net-tools`
* python因自带，仅额外安装pip
>1、首先安装epel扩展源：
[root@localhost ~]#  yum -y install epel-release
如果没有安装epel扩展源而直接安装python-pip时，会出现找到该软件包的错误。
这是因为像centos这类衍生出来的发行版，他们的源有时候内容更新的比较滞后，或者说有时候一些扩展的源根本就没有。
2、安装pip
[root@localhost ~]#  yum -y install  python-pip
3、更新pip
[root@localhost ~]#  pip install --upgrade pip
4、安装后清除cache
[root@localhost ~]#  yum clean all

* docker/docker-compose
	* 安装docker
		* https://docs.docker.com/install/linux/docker-ce/centos/#install-docker-ce-1
	* 免sudo, 将用户加到docker组
		* sudo usermod -aG docker kevin
	* 安装docker-compose
		* https://www.jianshu.com/p/dbc0fb6e9149
		* pip install docker-compose
#### 目录共享
* Virtualbox需要安装增强功能包
  安装增强功能包VBoxLinuxAdditions
   * 可参考： https://blog.csdn.net/buyueliuying/article/details/51645649
   * 注：功能包一般随Virtualbox安装目录已带，不用额外下载。
   * 注意： 光盘不一定是/dev/cdroom，有时被映射为/dev/sr0等。使用fdisk -l查看。
*   共享文件夹
	* 在VB中选择虚拟机->设置->共享文件夹
	* 注意：共享文件夹的名称千万不要和挂载点的名称相同。
		比如，上面的挂载点是/mnt/shared，如果共享文件夹的名字也是shared的话，在挂载的时候就会出现如下的错误信息(看http://www.virtualbox.org/ticket/2265)：
 /sbin/mount.vboxsf: mounting failed with the error: Protocol error

* 在linux内挂载共享目录
	  `sudo mount -t vboxsf repo/home/kevin/Source`  无读写权限
	  `sudo mount -t vboxsf repo -o rw,gid=1000,uid=1000 /home/kevin/Source` 带权限

* 开机自动挂载
	注：试用了几种方式没成功，略。（修改/etc/fstab或/etc/rc.local等）


#### 环境使用
你肯定不想每次要去打开VirtualBox的app再去选择点击某个虚拟机再启动吧，也不想在虚拟机里去敲命令的话。那么继续看下面。

* 操作虚拟机(命令行)
	* http://kodango.com/use-cli-to-start-vm
	* 查看虚拟机列表：
		*  VBoxManage list vms
		*  显示已经安装好的可用虚拟机名
	*  启动虚拟机（无gui）：
		*  VBoxManage startvm "centos7" --type headless
			*  centos7使用自己实际虚拟机名替换，可不用加引号
			*  headless可使系统不再弹出虚拟机UI
		* 注：启动还是需要一定时间（几秒？），刚启动不一定能马上连接上，保持耐心和微笑：P

* ssh去远程开发
  虽然是本机，你不一定喜欢非原生的虚拟机内部的操作感受，能不能利用osx的本地终端(如iTerm2)连接到虚拟机内开发呢？当然能。
  * 在VirtualBox的虚拟机设置中->网络->点开高级"->端口转发，配置一下本机和虚拟机的ip及端口就OK。
 >名称：ssh
 >协议：TCP
 >主机IP: 127.0.0.1
 >主机端口：36000
 >子系统IP: 10.0.2.15 #这个查看虚拟机内实际IP。若要使用静态IP的话，另寻资料
 >子系统端口: 22

#### 版本备注
* Host系统: Mac OSX 10.13.6
* VirtualBox 5.2.22
* CentOS 7.5.1804

#### 参考
* 以上操作上参考了网上一些文章，有些给了链接，但不一定完整列出，感谢Ta们
* 安装增强功能包VBoxLinuxAdditions
	* https://blog.csdn.net/buyueliuying/article/details/51645649
* VirtualBox命令行
	* http://kodango.com/use-cli-to-start-vm

