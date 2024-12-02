---
title: 在GCP虚机使用IPv6
tags:
  - GCP
  - 网络
categories:
  - 技术
  - 云计算
abbrlink: 6253a717
date: 2024-02-28 22:37:03
---

&emsp;&emsp;随着IPv6的普及，越来越多GCP的用户开始使用IPv6.在GCP，IPv6的配置和IPv4略有不同。本文简单讲解如何在GCP虚机配置IPv6。
<!-- more -->
&emsp;&emsp;如同IPv4，GCP的IPv6也分为内部地址和外部地址两种。启用IPv6的地方有两个：vpc和subnet，很容易造成疑惑。下边就具体介绍不同类型IPv6的创建方式
## 网络配置
&emsp;&emsp;创建vpc时，可以选择是否启用IPv6：
![](https://home.aubreyhan.net:9000/blogpic/17091775023408.jpg)

&emsp;&emsp;这里有几个点需要注意：
* IPv6是vpc固有属性，创建后无法修改（取消或启用）
* 在vpc启用的IPv6为内部地址
* vpc启用IPv6后，subnet必须使用cusome（自定义）模式
&emsp;&emsp;接下来创建subnet：
![](https://home.aubreyhan.net:9000/blogpic/17091778609032.jpg)

&emsp;&emsp;这里subnet可以选择单栈或双栈，同样，选择后无法修改，如要更换，只能删除subnet重建；另外，这里IPv6可以选择外网或内网，因为我们已经在vpc启用了IPv6。我们在此处选择内网，并创建vpc。
&emsp;&emsp;下边，在创建一个不启用IPv6的vpc，然后在subnet配置IPv6：
![](https://home.aubreyhan.net:9000/blogpic/17091805444989.jpg)

&emsp;&emsp;此时，subnet仍然可以配置双栈，但是IPv6只能配置外网。
&emsp;&emsp;简单总结一下，vpc和subnet配置IPv6的区别如下：
![](https://home.aubreyhan.net:9000/blogpic/17091811590095.jpg)

## 创建带有IPv6的虚机
### 创建带内部IPv6的虚机
&emsp;&emsp;创建过程如同普通虚机，只是在网络配置时，选择对应带内部IPv6的subnet，并选择IPv6分配方式：
![](https://home.aubreyhan.net:9000/blogpic/17091813860179.jpg)

&emsp;&emsp;创建完成，虚机已具备内部IPv6地址：
![](https://home.aubreyhan.net:9000/blogpic/17091814871148.jpg)

&emsp;&emsp;通过ssh进入虚机内部检查，也可看到网卡上已具备此地址，但是ping外部地址不通（DNS通过内网所以可以解析）：
![](https://home.aubreyhan.net:9000/blogpic/17091816301891.jpg)

### 创建带外部IPv6的虚机
&emsp;&emsp;创建方式相同，只是需要选择有外部IPv6的subnet：
![](https://home.aubreyhan.net:9000/blogpic/17091818389691.jpg)

&emsp;&emsp;创建完成，虚机具备外部IPv6地址：
![](https://home.aubreyhan.net:9000/blogpic/17091819314349.jpg)

&emsp;&emsp;进入虚机内部检查，可以在网卡上看到IPv6地址并且可以和外网相通：
![](https://home.aubreyhan.net:9000/blogpic/17091821470478.jpg)

## 总结
&emsp;&emsp;通过以上简单示例，可以总结如下几条：
* 如果需要给虚机配置IPv6，需要在网络启用，并且subnet必须选择自定义模式
* 如果vpc已启用IPv6，subnet可以选择内部或外部IPv6；如果vpc没有启用IPv6，subnet只能选择外部IPv6
* 对于同一个subnet，内部和外部IPv6只能二选一
* 创建虚机时，选择的subnet决定了其IPv6的模式，无需也不能在OS层面对网络进行配置
* 外部IPv6地址不同于外部IPv4的地址；外部IPv4地址不在虚机网卡，而是在一个隐形的LB上，所以在虚机的网卡上看不到外部IPv4地址；但外部IPv6地址是直接分配到subnet并且附加到虚机，所以在虚机的网卡上可以看到外部IPv6地址。当预留外部IPv6地址时，因为IPv6在subnet，所以在vpc的预留内部地址可以看到，但实际是外部地址，可能引起误会。
* 如果需要查询IPv6的网关，可以通过如下命令：
```
curl http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/gateway-ipv6 -H "Metadata-Flavor: Google"
```
&emsp;&emsp;
&emsp;&emsp;