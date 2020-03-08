---
layout: post
title:  "让Plex使用上https更安全"
author: kevin
categories: network
tags: plex Multimedia security ssl
---
* content
{:toc}


> 在使用plex作为家里多媒体解决方案后，感受还是不错的。一体化多端的解决方案给人的体验一致，这比KODI等反复折腾好很多，而远程观看的体验也不错。暂时来看是目前较好的解决方案。然而默认走http使人感觉不放心呢，本文假设你已经有本地plex服务器了，只解决安全使用这个问题。


### 流程概览
整个流程可以分为如下三步：
1. 申请域名和SSL证书
1. 建立DDNS及端口映射
1. 生成plex可用的pkcs12证书并配置上



### 流程详述
#### 申请域名和SSL证书
在使用https前我们需要拥有自己的域名和用于加密的SSL证书。这个步骤就细不说了，我以在腾讯云上的操作为例简述。
* 在[腾讯云](https://console.cloud.tencent.com/domain)上花几十块钱购买一个域名。
![28cc06a38a94b22bda49d83264ee9ae0](/assets/202003/plex-https/54DAF6D1-F92C-4B4E-BEDC-AF7084A53901.png)

* 给域名申请免费的[SSL证书](https://console.cloud.tencent.com/ssl)。
![efafdceaf202c452e70ec66784bf91e7](/assets/202003/plex-https/6CA666FE-0366-4599-86D3-E005D1090CBF.png)

  申请证书的审核通过后，就可以下载证书了。可以下载好备后续使用。
  

#### 建立DDNS及端口映射
有了域名后，而我们的plex服务器一般是在自己家的设备中（如NAS上）。这时候我们要让这个域名能解析到自己家服务器，这就是DDNS（动态域名解析）的作用。
方法有很多，比如花生壳，还有openwrt上各种插件进行动态域名解析，这里以我目前使用的【爱快】路由来简述（不同的DDNS方法只要达成同样目标即可）

1. 创建写域名解析的token
因腾讯云和dnspod合作，可以比较方便的使用它来绑定我们前面申请的域名。而上图中的TokenID,TokenKey可以在[dnspod这里](https://www.dnspod.cn/console/user/security)创建,找到网页中的【API Token】查看来创建即可。
![3724696cbc1fcbc5c8932fcdec143ea8](/assets/202003/plex-https/BAB75F45-2035-4F85-9EAD-D9E2A3E5D563.png)

2. 在路由器上配置上对前面申请的域名的解析
* ![d11e43747d3b619dbbc56b155748c3af](/assets/202003/plex-https/ED024D48-8259-40B0-A5F2-712C94B99CCC.png)
* ![f51117b9e13ddd56282e77d0cadf0ac4](/assets/202003/plex-https/C566F6E8-9791-49F0-BB09-19CE81B8DB1A.png)

一会可以看到更新结果成功，这样我们的域名已经可以被解析到了。

3. 配置端口映射
从外部访问plex的默认端口32400，这一般是需要端口转发的，很简单在路由器上端口转发配置一行即可。
![74865eee15a3d6a483846028112d4245](/assets/202003/plex-https/4BAB4E91-6778-432B-A940-EEAFD5D0442C.png)


### 生成plex可用的pkcs12证书并配置上
1. 使用第1步下载的证书（腾讯云上下载了一个zip包）。
```
openssl pkcs12 -export -out ~/certificate.pfx \
-inkey ./3_[你的域名].key \
-in ./2_[你的域名].crt \
-certfile ./1_[根证书].crt
```
我以下载包中的Apache的三个证书，通过`openssl`将证书格式转换成了pkcs12格式。

2. 上传此证书到plex服务器，然后在plex的设置中配置
![f37b9bf4db28ea9d94c63b23b74c7ab3](/assets/202003/plex-https/8E335B93-7AF0-46F5-8AC8-F3D05C7CC5EC.png)
不用重启plex即可生效。现在打开浏览器访问的结果
![32319cd8c7bf6288fe685c8644abe873](/assets/202003/plex-https/FEB56996-C635-4DBC-B68B-DF371BB0513B.png)
