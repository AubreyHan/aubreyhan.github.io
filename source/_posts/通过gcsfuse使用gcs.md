---
title: 通过gcsfuse使用gcs
tags:
  - GCP
  - 存储
categories:
  - 技术
  - 云计算
abbrlink: 7e702d3c
date: 2024-03-10 08:17:46
---

&emsp;&emsp;对象存储作为云计算infra的基础，有着广泛的用途，GCP上提供的对象存储——gcs也不例外。
&emsp;&emsp;但是，gcs的使用通常要通过sdk或者http client调用，增删改查等常规文件操作很不方便。gcsfuse作为GCP提供的小工具，可以将gcs挂载到本地，当作本地文件系统进行操作，大大方便了gcs的使用。
&emsp;&emsp;下边就简单聊聊gcsfuse的操作
<!-- more -->
# 基本安装使用
## 安装
&emsp;&emsp;gcsfuse的安装方法有很多，最直接简单的方式是直接从github网站下载对应平台及版本的安装文件，网址如下：
&emsp;&emsp;[gcsfuse release](https://github.com/GoogleCloudPlatform/gcsfuse/releases)
&emsp;&emsp;并安装（ubuntu为例）：
```
    sudo apt install gcsfuse***.deb
```

## 权限
&emsp;&emsp;如果其他gcs操作，通过gcsfuse挂载gcs，首先要具备对应权限。相关要求如下：
 * 如需装载仅用于读取的存储桶，服务账号必须具有该存储桶的 roles/storage.objectViewer 角色。
 * 如需装载用于读取和写入的存储桶，服务账号必须具有该存储桶的 roles/storage.objectAdmin 角色。

&emsp;&emsp;可以通过几种方式赋予权限：
### GCE 服务账号
&emsp;&emsp;如果是在GCP虚机（gce）挂载gcs，并且已在虚机启用服务账号，最简单的方法是直接对服务账号赋权。
#### GCE 默认服务账号
&emsp;&emsp;如果使用gce默认服务账号，默认配置对gcs只有只读权限：
![](https://home.aubreyhan.net:9000/blogpic/17100414080562.jpg)

&emsp;&emsp;所以如果使用gce默认服务账号，并需要读写权限，需要对其进行自定义配置：
![](https://home.aubreyhan.net:9000/blogpic/17100416084861.jpg)

&emsp;&emsp;另，在需要挂载的bucket对默认服务账号赋予权限：
![](https://home.aubreyhan.net:9000/blogpic/17100417821805.jpg)

#### 自定义服务账号
&emsp;&emsp;自定义服务账号更加灵活。
&emsp;&emsp;创建一个服务账号：
![](https://home.aubreyhan.net:9000/blogpic/17100422906570.jpg)

&emsp;&emsp;然后在gcs对该服务账号赋权：
![](https://home.aubreyhan.net:9000/blogpic/17101196088052.jpg)

&emsp;&emsp;再用做gce服务账号即可。
### 无服务账号
&emsp;&emsp;如果是非GCP虚机，或不希望通过gce服务账号赋权，可以通过如下方式配置权限：
#### 应用默认凭据
&emsp;&emsp;通过以下命令使用具备权限的用户登录
```
    gcloud auth application-default login
```
#### gcloud cli授权
```
    gcloud auth login
```
#### 服务账号凭据
&emsp;&emsp;为具备所需权限的服务账号创建key，并：
&emsp;&emsp;通过环境变量授权：
```
    export GOOGLE_APPLICATION_CREDENTIALS = key
```
&emsp;&emsp;或在挂载时直接通过参数 --key-file指定key文件

## 挂载方式
### 静态挂载
&emsp;&emsp;gcsfuse的基本使用方法如下：
```
    gcsfuse BUCKET_NAME MOUNT_POINT
```
&emsp;&emsp;注意，bucket名不要天街gs://等前缀。
&emsp;&emsp;如下是使用示例，~/mnt是提前创建的mount point，挂载前，里边为空，挂载后，可以看到对应bucket的内容：
![](https://home.aubreyhan.net:9000/blogpic/17101305569933.jpg)
&emsp;&emsp;卸载已挂载的gcs如同卸载普通文件系统，使用umount即可：
![](https://home.aubreyhan.net:9000/blogpic/17101351009637.jpg)
### 动态挂载
&emsp;&emsp;gcsfuse还提供了一种批量挂载的方式——动态挂载。
&emsp;&emsp;动态挂载时，无需指定bucket，会自动将有权限的bucket自动到mount point，每个bucket会自动载mount point下创建一个目录。
```
    gcsfuse MOUNT_POINT
```
&emsp;&emsp;如下创建3个bucket：
![](https://home.aubreyhan.net:9000/blogpic/17101354798834.jpg)

&emsp;&emsp;其中fuse001及fuse002赋予服务账号demosa@hy-20240308-001.iam.gserviceaccount.com存储读写权限，fuser没有赋权。
&emsp;&emsp;现将此服务账号的key作为环境变量，然后动态挂载：
![](https://home.aubreyhan.net:9000/blogpic/17101359814264.jpg)

&emsp;&emsp;可以看到fuse001和fuse002被正常挂载，但fuse003没有挂载。
&emsp;&emsp;同样，卸载时，只需卸载根mount point，所有bucket都将被卸载：
![](https://home.aubreyhan.net:9000/blogpic/17101360919781.jpg)
### 自动挂载
&emsp;&emsp;gscfuse会注入mount命令，所以可以通过修改fstab实现开机自动挂载。方式如下：
#### fstab修改
&emsp;&emsp;在fstab添加如下内容：
```
    my-bucket /mount/point gcsfuse rw,_netdev,allow_other,uid=1001,gid=1001
```
&emsp;&emsp;参数说明如下：
* mybucket：bucket名字（不要加gs://）
* /mount/point: 挂载点
* gcsfuse：文件系统类型
* rw：读写方式挂载
* _netdev：等待网络就绪后挂载
* allow_other：允许其他用户（非root）挂载
* uid=1001,gid=1001：挂载用户的uid和gid（可以用id user查看）
&emsp;&emsp;如下查找uid/gid并修改/etc/fstab：
![](https://home.aubreyhan.net:9000/blogpic/17104159923212.jpg)

#### 权限分配
&emsp;&emsp;如之前描述，挂载gcs时需要权限配置，通过fstab挂载也是一样。
&emsp;&emsp;当gce已经配置服务账号，并具备相关权限，虚机启动时会自动使用服务账号的权限并挂载gcs:
![](https://home.aubreyhan.net:9000/blogpic/17104167804143.jpg)
&emsp;&emsp;但是，如果gce没有配置服务账号/服务账号权限不够/非gcp虚机等无法使用服务账号时，如果获得默认凭据后重启虚机，却发现没有自动挂载：
![](https://home.aubreyhan.net:9000/blogpic/17104171325069.jpg)
![](https://home.aubreyhan.net:9000/blogpic/17104173915412.jpg)

&emsp;&emsp;产生这个问题的原因是，执行获取应用默认凭据命令时，获得的凭据保存在执行此命令的当前用户目录下。而虚机启动过程中，通过fstab自动挂载文件系统时，是root用户，所以是在root用户目录下查找凭据。
&emsp;&emsp;知道原因解决就简单了，使用root用户获取凭据，可以看到凭据保存在root用户目录下：
![](https://home.aubreyhan.net:9000/blogpic/17104177183955.jpg)
&emsp;&emsp;再次重启虚机检查，可以看到已正确挂载：
![](https://home.aubreyhan.net:9000/blogpic/17104178525507.jpg)

# 进阶使用
## 命令参数
&emsp;&emsp;gcsfuse支持参数设置，使用方式如下：
```
    gcsfuse GLOBAL_OPTIONS BUCKET_NAME MOUNT_POINT
```
&emsp;&emsp;通过GLOBAL_OPTIONS指定挂载参数，具体可参见：
&emsp;&emsp;[gcsfuse参数](https://cloud.google.com/storage/docs/gcsfuse-cli?hl=zh-cn)
&emsp;&emsp;通过参数可以精细化管理gcsfuse
## 目录
&emsp;&emsp;目录在对象存储中是一个虚拟的概念，作为对象文件名的前缀。所以使用gcsfuse挂载时会有不同.
&emsp;&emsp;首先直接在gcs里创建一个文件夹dir01并上传一个文件：
![](https://home.aubreyhan.net:9000/blogpic/17104607049046.jpg)

&emsp;&emsp;然后上传一个带文件的目录dir02到gcs（不提前在gcs创建目录）：
![](https://home.aubreyhan.net:9000/blogpic/17104609731371.jpg)

&emsp;&emsp;在gcs可以看到dir02:
![](https://home.aubreyhan.net:9000/blogpic/17104609955763.jpg)

&emsp;&emsp;回到虚机，去列目录及子目录：
![](https://home.aubreyhan.net:9000/blogpic/17104611530911.jpg)
&emsp;&emsp;只看到了dir01内容，却没看到dir02.
&emsp;&emsp;导致此情况的原因是，通过上传方式自动创建的目录，叫做隐式定义目录，gcsfuse挂载后，无法列出隐式定义目录的内容。
&emsp;&emsp;解决方法是，gcsfuse使用时加上--implicit-dirs参数：
![](https://home.aubreyhan.net:9000/blogpic/17104614299229.jpg)

&emsp;&emsp;这样即可正常显示隐式定义目录的内容。

## 缓存目录
&emsp;&emsp;gcsfuse使用缓存目录存放临时文件，默认情况下使用/tmp目录。也可以通过参数--temp-dir指定自定义目录。
&emsp;&emsp;此处创建一个100M分区并挂载到/temp：
![](https://home.aubreyhan.net:9000/blogpic/17104637348965.jpg)

&emsp;&emsp;将该目录作为gcsfuse的缓存目录：
![](https://home.aubreyhan.net:9000/blogpic/17104637748205.jpg)


&emsp;&emsp;现在创建一个500M的文件：
![](https://home.aubreyhan.net:9000/blogpic/17104627518314.jpg)

&emsp;&emsp;将该文件拷入gcs挂载目录：
![](https://home.aubreyhan.net:9000/blogpic/17104638399133.jpg)

&emsp;&emsp;出现空间不足错误。
&emsp;&emsp;这是因为gcsfuse的原理是读写文件时，先将文件放入本地缓存目录，到落地时间（--stat-cache-ttl参数设置，默认60s）再后台通过http client同步到gcs。所以如果文件大小超过缓存目录剩余空间，就会出现空间不足的错误。

## 多虚机挂载
&emsp;&emsp;同一个gcs可以挂载到多个虚机。
&emsp;&emsp;不同于传统DAS/NAS/SAN存储，多机挂载时gcs文件的管理机制不一样。
&emsp;&emsp;对于DAS/SAN，服务器/虚机识别为本地硬盘，每台服务器/虚机各自维护文件系统的映射表（分区表/superblock/inode等），所以多机挂载时，需要多台服务器/虚机之间通过仲裁机制（windows cluster/oracle RAC/AIX HACMP/Hadoop HDFS等）对每台服务器/虚机发出的操作指令进行排序同步等处理，保证不出现脑裂。
&emsp;&emsp;而对于NAS，服务器/虚机并不直接操作硬盘，而是发出指令由NAS系统处理。
&emsp;&emsp;而gcs存在本地缓存，造成本地和gcs之间异步文件处理，所以对同一个文件的操作，需要靠锁来管理。
&emsp;&emsp;再创建一台vm（vm02）并通过gcsfuse挂载fuse001,创建文件，两边可以同时看到:
![](https://home.aubreyhan.net:9000/blogpic/17104660006877.jpg)

&emsp;&emsp;现在vm01使用vi打开文件并随便写入内容保持打开状态：
![](https://home.aubreyhan.net:9000/blogpic/17104660911139.jpg)


&emsp;&emsp;再到vm02使用vi 打开并写入：
![](https://home.aubreyhan.net:9000/blogpic/17104662006461.jpg)

&emsp;&emsp;出现报错。
&emsp;&emsp;因为gcs作为对象存储，对同一个文件的多写没有仲裁机制，就会出现冲突。

# 小结
&emsp;&emsp;gcsfuse是一个方便易用的工具，实现了在虚机文件系统对gcs直接增删改查。并且现在GCP已整体提供对gcsfuse的技术支持。
&emsp;&emsp;使用过程中需注意如下事项：
* gcsfuse的读写性能不是最佳并且是异步传输，不适宜要求高带宽读写实时同步的场景
* 使用时注意配置缓存大小，避免缓存空间不足导致的问题
* 权限设置合理，避免权限不够
* 避免对同一个文件的多写造成的冲突
* 如无必要，尽量通过只读方式挂载，减少误写带来的错误

