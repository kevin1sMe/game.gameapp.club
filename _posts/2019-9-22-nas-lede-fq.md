---
layout: post
title:  "二级路由LEDE助力全家出国游"
author: kevin
categories: nas
tags: nas;lede;openwrt;ss;ssr
---
* content
{:toc}


> 家里自从更换了AC+AP方案后，原华硕的AC1900p的WiFi功能已经被我关掉，只剩下因为刷梅林固件后的便利了，我颇感浪费，在了解到我家NAS(TS-453B Mini)支持虚拟化技术，本着物尽其用的态度，利用它搞了个二级路由，折腾蛮久终获成功，然后永久的关闭了华硕的AC1900p路由的电源。

### 原来的网络情况
之前有[文章](http://game.gameapp.club/2019/03/24/home-network-design/)记录过，我手绘一个大概是这样子的。

![Alt Text](/assets/201909/ori-family-net.jpg)

主要问题是Asus路由1900p有暴殄天物之嫌。而TL-R479GP-AC所带的能力（各种认证XX，防止火等）又使用不上，甚至拨号功能都没用上。（这货号称企业级路由器，但对它性能我还是持怀疑态度的）。


### 新网络方案
![Alt Text](/assets/201909/family-net.jpg)

原来从猫出来的网线接到AC1900p用于拨号的，现在可以接到POE路由器的WAN口由它来作为主路由，而从主路由出来连接到NAS中。在NAS中安装一个LEDE系统，可以设置复用原本NAS服务的网口（有独立于NAS的IP但是网口是同一个），我还没测试过是否会因此降低网速，不行的话咱NAS还有另一个网口呢（不过因NAS放在书房，而书房只有两个网络出口要留一个给PC使用，咱不想再放交换机啊！）

网络拓扑的变化大概就是如上，下面介绍一下怎么安装LEDE及一些路由上的配置。

### NAS安装LEDE
> 这个LEDE系统的作用就像是自己局域网中的一台电脑，它只需要接入我们局域网能被访问到。然后作为网关使用即可把流量导向它，然后安装一些不可描述的插件。达到全家出国旅游的目标。

网上有一些参考资料，可在文章最后附的链接中找到。我这里简要介绍一些关键点。（以我目前的QNAP的NAS为例，如果是群辉等应试类似）。


1. 确认自己的NAS支持虚拟化技术。
1. 在`网络与虚拟交换机`中创建虚拟交换机，高级模式，实体网络适配器中选择NAS原本使用的网口（复用），下一步，动态IP。一直默认下一步到结束。
1. 安装软件Virtualization Station。这个可以在应用商店找到。
1. 下载制作好的含有LEDE系统的[PE镜像](https://drive.google.com/file/d/1MTPGgpvoodV0LWM9aiZGvh82nG1N61MW/view)。（这个是由BigDongDong制作的，感谢）
1. 建立虚拟机，操作系统选择`Generic`，内存和CPU自己看情况设置，目前我感觉1核心1G够用了。选择选择上面下载好的PE镜像，并且设置一下虚拟机的镜像文件目录，大小可以改小一点，默认太大了。
![Alt Text](/assets/201909/nas-lede-02.png)
1. 安装过程参考上面，只是我们不是做主路由，不需要两个虚拟交换机，只需要一个LAN口。安装好LEDE后记得修改一下/etc/config/network中的IP，然后reboot重启。
![Alt Text](/assets/201909/nas-lede-03.png)

### LEDE及主路由配置
1. 如上图，我把LEDE的IP设置为了192.168.50.253。重启后我们就可以在浏览器输入这个地址进入设置界面，默认密码`koolshare`。 进入`网络-接口`界面，设置LAN的一些属性。关注一下网关和自定义的DNS服务器的设置，这个设置为你主路由的地址，我这里是192.168.50.1(你可能是192.168.1.1)。然后就可以愉快的安装插件了。酸酸乳插件需要离线安装，请自行搜索。
![Alt Text](/assets/201909/nas-lede-04.png)

1. 关闭LEDE中的DHCP服务器，一个局域网只需要一个DHCP，这事我们让主路由去分配。
![Alt Text](/assets/201909/nas-lede-05.png)

1. 打开主路由界面，设置DHCP的一些属性（拨号那些我就不讲啦）
![30c5961e67ff86cdc110a62b373772cb.png](evernotecid://1A26A2C9-D9C0-47AE-AB96-6C52D60D6EE6/appyinxiangcom/8166302/ENResource/p14997)
![Alt Text](/assets/201909/nas-lede-06.png)

以上设置完毕，所以连接上来的设备都会被DHCP分配并且把默认的网关及DNS设置为我们LEDE的地址。

> 可能你会问既然这样为何不使用这个LEDE做主路由，原因有二：第一是NAS放在书房要主路由需要有线回到交换机，浪费了端口。第二是总觉得不一定稳的话，咱还是让专业的路由做主路由吧。LEDE主要协助出国游。

如果你不需要全家出国游，那么可以不设置DHCP中的网关地址和DNS到LEDE，使用默认的这些流量就不会经过LEDE了。而对于需要的设备手动设置这两项为LEDE的IP即可。

最后附上正常运行的LEDE状态，负载还是相对较低的（听说ROS更是节省资源）
![Alt Text](/assets/201909/nas-lede-01.png)


### 参考资料
* [NAS摇身一变成路由丨QNAP虚拟机安装软路由系统LEDE](https://www.bilibili.com/video/av39483249?from=search&seid=16918862470062196380)
