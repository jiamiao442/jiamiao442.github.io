﻿---
title: docker-compose 安装fastdfs
date: 2023-10-24 15:04:45
tags: fastdfs
---

## 1, 配置docker-compose.yml

mkdir -p /opt/fastdfs
cd /opt/fastdfs
vi docker-compose.yml

```sh


version: '2'
services:
    fastdfs-tracker:
        hostname: fastdfs-tracker
        container_name: fastdfs-tracker
        image: season/fastdfs:1.2
        network_mode: "host"
        command: tracker
        volumes:
          - ./tracker_data:/fastdfs/tracker/data
    fastdfs-storage:
        hostname: fastdfs-storage
        container_name: fastdfs-storage
        image: season/fastdfs:1.2
        network_mode: "host"
        volumes:
          - ./storage_data:/fastdfs/storage/data
          - ./store_path:/fastdfs/store_path
        environment:
          - TRACKER_SERVER=10.33.5.109:22122
        command: storage
        depends_on:
          - fastdfs-tracker
    fastdfs-nginx:
        hostname: fastdfs-nginx
        container_name: fastdfs-nginx
        image: season/fastdfs:1.2
        network_mode: "host"
        volumes:
          - ./nginx.conf:/etc/nginx/conf/nginx.conf
          - ./store_path:/fastdfs/store_path
        environment:
          - TRACKER_SERVER=10.33.5.109:22122
        command: nginx

```

## 2, 编辑 nginx.conf

vi nginx.conf

```sh
#user  nobody;
worker_processes  1;
 
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;
 
#pid        logs/nginx.pid;
 
 
events {
    worker_connections  1024;
}
 
 
http {
    include       mime.types;
    default_type  application/octet-stream;
 
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';
 
    #access_log  logs/access.log  main;
 
    sendfile        on;
    #tcp_nopush     on;
 
    #keepalive_timeout  0;
    keepalive_timeout  65;
 
    #gzip  on;
 
    server {
        listen       7003;
        server_name  localhost;
 
        #charset koi8-r;
 
        #access_log  logs/host.access.log  main;
 
        location /group1/M00 {
            root /fastdfs/storage/data;
            ngx_fastdfs_module;
        }
 
        #error_page  404              /404.html;
 
        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
 }
}
```

##  3, 启动fastDFS

docker-compose up -d

查看容器状态

docker-compose ps

## 4, 客户端测试

配置客户端访问

docker run -tid --name fdfs_sh -p 13000:13000 season/fastdfs sh
docker cp fdfs_sh:/fdfs_conf/client.conf ./

配置客户端

vi client.conf 

```sh
base_path=/fastdfs
tracker_server=10.33.5.109:22122
```



重新配置客户端容器

docker rm -f fdfs_sh
docker run -tid -v /opt/fastdfs/client.conf:/fdfs_conf/client.conf --name fdfs_sh -p 13000:13000 season/fastdfs  sh

进入容器测试上传文件

docker exec -it fdfs_sh sh

生成文件并上传

echo hello>b.txt
fdfs_upload_file /fdfs_conf/client.conf /b.txt

返回的文件地址
group1/M00/00/00/CiEFbWO_x2WAU2l6AAAABncc3SA269.txt

浏览器访问地址
http://10.33.5.109:7003/group1/M00/00/00/CiEFbWO_x2WAU2l6AAAABncc3SA269.txt
