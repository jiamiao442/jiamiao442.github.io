---
uuid: 5362e7b6-05d1-9948-3204-1a7bf9ba9e82
title: 新版单机fastdfs安装
date: 2023-10-24 15:04:45
tags: fastdfs
---

##  1, 拉取最新镜像

```bash
docker pull ygqygq2/fastdfs-nginx:V6.9.4
docker tag ygqygq2/fastdfs-nginx:V6.9.4 192.168.0.79:5000/libray/fastdfs-nginx:V6.9.4
docker push 192.168.0.79:5000/libray/fastdfs-nginx:V6.9.4
```

## 2, 编写部署文件

```bash
mkdir -p /opt/fastdfs
cd /opt/fastdfs
vi docker-compose.yml
```

```
version: '3'
#networks:
#  fastdfs-net:
#    external: true
networks:
  fastdfs-net:
    driver: bridge
services:
  tracker:
    container_name: tracker
    image: 192.168.0.79:5000/library/fastdfs-nginx:V6.9.4 
    command: tracker
    #network_mode: host
    networks:
      - fastdfs-net
    volumes:   
      - /var/fdfs/tracker:/var/fdfs    
    ports:
      - 22122:22122
  storage0:
    container_name: storage0
    image: 192.168.0.79:5000/library/fastdfs-nginx:V6.9.4 
    command: storage
    #network_mode: host  
    networks:
      - fastdfs-net
    environment:
      - TRACKER_SERVER=tracker:22122
    volumes: 
      - /var/fdfs/storage0:/var/fdfs
    ports:
      - 28080:8080
    depends_on:
      - tracker
  storage1:
    container_name: storage1
    image: 192.168.0.79:5000/library/fastdfs-nginx:V6.9.4 
    command: storage
    #network_mode: host  
    networks:
      - fastdfs-net
    environment:
      - TRACKER_SERVER=tracker:22122
    volumes: 
      - /var/fdfs/storage1:/var/fdfs
    ports:
      - 28081:8080
    depends_on:
      - tracker
```

## 3, 启动应用并验证

```bash
docker compose up -d

docker exec -it tracker bash
[root@936b4a677e5b fdfs]# date >aaa.txt
[root@936b4a677e5b fdfs]# fdfs_upload_file /etc/fdfs/client.conf  aaa.txt
group1/M00/00/00/rBIAA2R-lRCAYKF3AAAAHVN3P3Y136.txt

打开另一终端访问
[root@fastdfs1 ~]# curl localhost:28080/group1/M00/00/00/rBIAA2R-lRCAYKF3AAAAHVN3P3Y136.txt
Tue Jun  6 02:07:29 UTC 2023
[root@fastdfs1 ~]# curl localhost:28081/group1/M00/00/00/rBIAA2R-lRCAYKF3AAAAHVN3P3Y136.txt
Tue Jun  6 02:07:29 UTC 2023


```

