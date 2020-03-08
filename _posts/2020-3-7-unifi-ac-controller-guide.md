---
layout: post
title:  "Unifi AC Controller容器运行指南"
author: kevin
categories: network
tags: unifi ssl https container docker
---
* content
{:toc}


> 家里搞了一套Unifi后（US8-switch+UAP-AC-pro+UAP-AC-IW)，思量着原来放在物理机macbookpro上的AC控制器为了稳定和长久起见，应该挪到NAS中去了，并且也可长期在线。有一些玩法（如访客认证等）有时间也可以搞起来。这里记录一下几个折腾的事项。



### 基本目标
* 容器化，不想安装乱七八糟的应用。
* 数据不能丢，Unifi中有比较好的各种记录和统计，蛮有意思的，必须在容器重建依赖存在。
* 为了更好配合远程控制，容器必须分配固定IP方便作端口映射。


### 步骤
#### 选择容器镜像
首选这个高star的镜像[jacobalberty/unifi](https://hub.docker.com/r/jacobalberty/unifi)

关注其说明：
![c8feffb6eaf2ea4c4b45ba44065874e9](/assets/202003/unifi-ac-ctrl/unifi-container-1.png)

为了容器中的配置等关键数据持久化，我们只需要将host机的某个目录以/unifi挂载。

#### 在主机中创建Unifi目录
在主机创建unifi目录而不是使用容器默认的挂载，是为了容器多次起来数据不丢失。
命令： `mkdir Unifi`

并且可以看到证书的配置方式（此步骤可选，当时配置上SSL证书，则可以使用https访问控制器）：
![e22509ee50f50f93c9d577f95baa6086](/assets/202003/unifi-ac-ctrl/unifi-controller-2.png)

我们只需要在上面创建的Unifi目录下，创建cert目录，并且将SSL证书按名字改好放过去即可。如果不改名则需要在环境变量中设置各证书实际名称。

#### 创建静态ip的network网桥
为了使在外部能方便访问，容器最好是有独立IP，不光如此，使用DHCP会使我们端口映射很难固定。所以这里需要让容器起来时能够获取到静态IP（我们指定的IP)。这里以我在QNAP的这个nas中操作为例：
* 创建静态network
docker network create -d qnet --ipam-driver=qnet --ipam-opt=iface=eth1 \
      --subnet=192.168.50.0/24 --gateway=192.168.50.251 qnet-static-eth1
 
这样我们一会在启动容器时可以使用这个网桥。

#### 启动容器
```
docker run \
-d \
--rm --init \
--name unifi-static-ip \
--net=qnet-static-eth1 \
--ip=192.168.50.60 \ # 这里你希望给unifi-ac-controller分配的ip
-e TZ="Asia/Shanghai" \
-h kv-unifi-controller \
-v /share/Unifi:/unifi \ # 这里/share/Unifi是你在主机创建的目录
jacobalberty/unifi:stable
```

#### 在路由中对AC-Controller作端口映射
这个很简单，步骤略。注意端口是8443即可。

### 成果
现在就可以远程去访问我们的AC-controller啦。即便涉及敏感的个人数据，我们也有https作保障啦。
![38691624d80160c9da49f652933d547e](/assets/202003/unifi-ac-ctrl/unifi-controller-3.png)


### 后话
* 家用的一些网站等，能走https都走，因为http实在不安全。
* 借助于远程查看unifi-ac-controller的能力，我想到以后要是手机落家里，大概在什么位置我都能定位到了。看它就近连接了哪个AP :)


### 参考
* [jacobalberty/unifi](https://hub.docker.com/r/jacobalberty/unifi)
* [QNAP中给容器分配静态ip](https://qnap-dev.github.io/container-station-api/qnet.html)
