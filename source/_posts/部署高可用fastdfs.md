---
title: centos7部署高可用fastdfs
date: 2023-10-24 15:04:45
tags: fastdfs
---

## 1，环境介绍

由俩台服务器实现fastdfs的高可用

192.168.174.132     tracker，nginx，storage

192.168.174.150     tracker，nginx，storage

## 2，初始化centos7

### 2.1，关闭selinux

```bash
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

### 2.2,  关闭防火墙

```bash
systemctl stop firewalld && systemctl disable firewalld 
```

### 2.3,  配置时间同步

storage 同步文件依赖时间戳，所以要保证所有服务器时间一致。

```bash
yum install -y chrony \
&& systemctl start chronyd && systemctl enable chronyd \
&& timedatectl set-timezone Asia/Shanghai && timedatectl set-ntp yes &&timedatectl set-local-rtc 0
```

### 2.4，配置limit

```bash
vi /etc/security/limits.conf
root soft nofile 65535
root hard nofile 65535
 * soft nofile 65535
 * hard nofile 65535

```

注：只配置最后两行不就可以了吗，为啥还要单独为root用户配置呢？查了网上资料，说是*这样的通配符对root用户无效，所以root需要单独配置

## 3, 高可用安装fastdfs

### 3.1 ，生成安装路径

​    俩台服务器都创建目录

```
mkdir -p /data/docker/fastdfs/tracker/data   #data为tracker基础数据存储目录
mkdir -p /data/docker/fastdfs/storage/data     #data为storage基础数据
mkdir -p /data/docker/fastdfs/upload/path{0,1,2,3} #存储上传的文件,当有多块硬盘时，挂载到相应目录上即可 /data/fastdfs/upload/path0~n

```

### 3.2，上传配置文件并修改配置

https://github.com/jiamiao442/fastdfs-nginx.git

将配置文件fastdfs-conf中的文件上传到第一台服务器上/data/docker/fastdfs/ 目录中

```bash
[root@ip-192-168-174-132 fastdfs]# ls -F
conf/  nginx_conf/  nginx_conf.d/  setting_conf.sh*  storage/  tracker/  upload/
```

修改setting_conf.sh，主要修改以下几个参数，其他不变

```bash
# 1. tracker 主要参数，生产环境中建议更改一下端口
tracker_port=22122
# 实现互备，两台tracker就够了
tracker_server="tracker_server = 192.168.174.132:$tracker_port\ntracker_server = 192.168.174.150:$tracker_port"

# 格式：<id>  <group_name>  <ip_or_hostname
storage_ids="
100001   group1  192.168.174.132
100002   group1  192.168.174.150
"

# 设置tracker访问IP限制，避免谁都能上传文件，默认是allow_hosts = *
allow_hosts="allow_hosts = 192.168.174.0/24\n"

# 2. local storage 主要参数，生产环境中建议更改一下端口
storage_group_name="group1"
storage_server_port=23000
store_path_count=1   #文件存储目录的个数，存储目录约定为/data/fastdfs/upload/path0~n

```

运行脚本生成配置文件

```bash
[root@ip-192-168-174-132 fastdfs]# bash setting_conf.sh 
 请先设置好本脚本的tracker \ storage 的参数变量，然后再选择：
  [1] 配置 tracker
  
  [2] 配置 storage
please input number 1 to 2: 1
  配置文件设置完毕，建议人工复核一下
[root@ip-192-168-174-132 fastdfs]# bash setting_conf.sh
 请先设置好本脚本的tracker \ storage 的参数变量，然后再选择：
  [1] 配置 tracker
  
  [2] 配置 storage
please input number 1 to 2: 2
  配置文件设置完毕，建议人工复核一下
[root@ip-192-168-174-132 fastdfs]# 


```

检测没有问题后将配置文件复制到另一台服务器

```bash
scp -r /data/docker/fastdfs/conf root@192.168.174.150:/data/docker/fastdfs/
scp -r /data/docker/fastdfs/nginx_conf root@192.168.174.150:/data/docker/fastdfs/
scp -r /data/docker/fastdfs/nginx_conf.d root@192.168.174.150:/data/docker/fastdfs/
```

### 3.3, 启动tracker

  两台同时启动

```bash
docker run -d --net=host  --restart=always --name=tracker  \
-v /data/docker/fastdfs/tracker/data:/data/fastdfs_data \
-v /data/docker/fastdfs/conf:/etc/fdfs \
 jiamiao442/fastdfs-nginx:latest tracker
```

### 3.4 启动storage

 两台同时启动

```bash
docker run -d --net=host --restart always --name=storage0 \
--privileged=true \
-v /data/docker/fastdfs/storage/data:/data/fastdfs_data \
-v /data/docker/fastdfs/conf:/etc/fdfs \
-v /data/docker/fastdfs/upload:/data/fastdfs/upload \
-v /data/docker/fastdfs/nginx_conf/nginx.conf:/usr/local/nginx/conf/nginx.conf  \
-v /data/docker/fastdfs/nginx_conf.d:/usr/local/nginx/conf.d  \
 jiamiao442/fastdfs-nginx:latest storage
```

### 3.5 检查是否可用

```bash
docker exec -it storage0 sh

date >aaa.txt
fdfs_upload_file /etc/fdfs/client.conf aaa.txt 
group1/M00/00/00/oYYBAGR4VxSAdTkeAAAAHYhCyck239.txt 

浏览器访问
http://192.168.174.132:9088/group1/M00/00/00/oYYBAGR4VxSAdTkeAAAAHYhCyck239.txt  
http://192.168.174.150:9088/group1/M00/00/00/oYYBAGR4VxSAdTkeAAAAHYhCyck239.txt  
```



