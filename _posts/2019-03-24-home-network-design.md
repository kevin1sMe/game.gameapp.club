---
layout: post
title:  "重构家庭网络"
author: kevin
categories: network
tags: network;ap;ac
---
* content
{:toc}

### 起因
当NAS和投影一切准备就绪后，在卧室看高清因网口没通，有线的话得从书房拉起，而我一是懒，二是觉得不完美相当别扭。
而无线的话2.4G从书房到卧室，衰减得带宽过小，看一些高码率的电影就会各种缓冲。

我尝试忍受，直到有一天转变了想法，一个长久居住的地方，生活的便利性只能靠自己动手去解决，何必要忍受这种不必的痛苦呢？

### 方案
解决这个其实方案不只一种。

* 电力猫
可用电力猫解决无网口问题。但想到之前用过tplink的电子猫，不甚稳定，故放弃。虽然闲鱼看到有200块包邮的一套（子母），也考虑过华为的子母路由器，最终还是不放心。pass。

* 路由器Mesh
因所用的华硕路由器Ac1900p是支持它们家的AI Mesh的功能的。老家上下楼层用的就是这种方案。有自动快速漫游等，效果还是不错并且稳定的，但问题是，如果是通过无线中继的漫游，它的带宽如何保证呢？而有线的话，我之前测试过卧室的网口根本就没连接通。当然华硕的成本还好，买个68u也能组，估计600-700RMB左右。其实对于我家这面积和情况,可以考虑。不过我还是pass了。原因是下面。

* AC+AP
对于网络信号覆盖及带宽的追求。我希望方案更具扩展性。比如哪个房间网络不行，可以成本较低的覆盖,而不用再多购买路由器。比如小卧的信号要是不妥，可以花100-200搞定最好。于是AC+AP是可以考虑的。

并且由于目前2.4G各信道已经拥挤不堪了，经常在家玩个**火影忍者**都会卡顿。所以最好是支持5G(5.8G，以下简称5G)全覆盖，更多的频段，更高的带宽！有这下考虑下，那么似乎只有AC+AP能解决问题了。因为5G的衰减，你最好各房间都可以通过AP把5G信号覆盖全，然后通过AC的快速漫游使网络可以全程保持5G。那么就它了。

### 实施
我决定打开弱电箱，把家里的网络整体理一下。面对杂乱的网路，哪个网线对应哪个房间呢？ 这些网线是否能正常使用呢？ 其实挺忐忑。

有些工具虽不是必须的，但是有它会幸福。于是网上采购了一批：
* 网线检测仪
    * 因为我打算把弱电箱里所有房间预留而没有做的网线全联通，我需要制作不少水晶头。完事后不得测试一下联通信吗？
    * 题外话：为了保证内网全1000M网络，网线制作必须8根线路都是通的。以前百兆时只需要1,2,3,6通便可以，这里要注意。因为好久没做过了（网络工程专业的我汗颜啊），有一个水晶头制作了五六次才打通8根。你能体会每次满怀希望的接通检测仪时，又发现某个不亮时的淡淡忧伤吗？你能体会发现面板里预留的网线被越剪越短时，生怕用完了还没制作成功的恐惧吗？:D

* 寻线仪
    * 其实它不是必须的。因为收到它的时候，我已经把家里原4根没接通的网线搞清楚对应关系了。
    * 有了它后，从一堆网线中寻找确实会轻松很多。虽然没派上啥用场，而且老板说15天试用可以退，最后还是懒得退货了，留着没准以后兼职搞点网络改造吧（玩笑：P)

* 网线钳
    * 做水晶头必须的
    * 插曲是第一次拿到手上未注意到它锋利的刀片，瞬间血溅地板，毫无知觉下割的手指1cm长的划痕。所以如果你用它请特别注意下。

* TP-link TL-R479GP-AC全千兆有线路由器8口
    * 单独的AC较贵，TP的AC100要200块,并且AC100要v3硬件以上才支持快速漫游。
    * 名称中的G指千兆网口，为未来着想，为高清电影着想，这是必须的。
    * 名称中的P指PoE供电（利用网线可供电），这样我们的AP就不用再独立供电了。

* TP-link TL-AP1202GI-PoE 面板式AP 2个
    * 千兆AP
    * 有线网口也是千兆的
    * PoE供电
* TP-link TL-AP450I-PoE 面板式AP 1个
    * 这个误买了，以为450M够用，没想到有线网口是100M的，后面懒得退换了，某些不重要的活动区域将就用吧

* TP-link TL-SG1008M 8口千兆交换机
    * 后面退掉了,有个8口的R479GP足够了，不需要再用交换机

* 华硕Asus Ac1900p
    * 原来的主路由器，虽然R479GP是带路由功能的，但是，由于不可描述，我必须是让它继续连接光猫作为路由。

### 网络拓扑
上文提到，这里简要画一下。

```
___________            __________               _____________
| (弱电箱) |-----WAN-->| (书房) |               |  (弱电箱) | <-- POE --- > AP
| 光猫     |           |华硕路由|-LAN------WAN->| TP R479GP | <--- POE ---> AP
|__________|           |________|               |___________| <--- POE ---> AP
                         |   |
                         |   |
                         V   V
                        NAS  PC

```
* 因为仍然打算把路由器放书房，（利用一下它身上的几个千兆LAN口）这样可以同时接上Nas和Pc，所以从猫拨号到接通R479GP是需要两根网线的，万幸书房下面面板正有两根网线！天佑我呀：)


### 其它
* 设置一下R479GP的无线功能就可以了，它自己使用动态获取IP就可以上网。因为主路由出来到它WAN口的。
* 反正我把卧室床头的AP的功率降到最低了，有人告诉我高了后真的会得啥病吗？在线等：P
* 设置弱信号剔除。 70dbm, 75dbm为推荐值。

### 最后
* 推荐一个测试网络的工具。`WiFi魔盒`, 功能齐全，内网都能测。能测试漫游功能。
* 其实只发现这款AC+Poe一体化路由并不能完全替代AC,它的切换只能弱信号剔除，达不到“快速无缝漫游”的最理想效果。
* 后续如果感觉AP的切换延迟明显，再买个AC接到R479GP上，把它本身的AC功能关闭，应该就OK了。
* 其实这最终方案整体费用较高，近1400块。

---
> 后来，卧室看电影连接的是5G，再也没有卡顿过了。 我为它准备的有线千兆也没派上用场。

### 附部分工作图以为记录
---
![Alt text](./assets/201903/IMG_5328.JPG)
**工欲善其事，必先备其器**

---
![Alt text](./assets/201903/IMG_5329.JPG)
**TP全家桶啊**

---
![Alt text](./assets/201903/IMG_5303.JPG)
**开工**

---
![Alt text](./assets/201903/IMG_5310.JPG)
**验收网络，nas拷贝到主机千兆**

---
![Alt text](./assets/201903/IMG_5305.JPG)
**验收网络，无线带宽**