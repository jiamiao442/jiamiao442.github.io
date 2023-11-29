---
uuid: 12d52074-cb7d-9dc0-f2dc-03f216abce3b
title: window下kubectl访问k8s集群
date: 2023-11-29 15:47:45
tags: window, kubectl, k8s
---
# window下kubectl访问k8s集群


```sh
curl.exe -LO "https://dl.k8s.io/release/v1.28.4/bin/windows/amd64/kubectl.exe"
```
## git bash
当前我的k8s是v1.20.11 版本所以下载如下

```sh
curl -LO "https://dl.k8s.io/release/v1.20.11/bin/windows/amd64/kubectl.exe"
```
放入 gitbash 家目录的bin下
```sh
D:\git\Git\mingw64\bin
```

然后导入配置文件，就可以访问集群了

```
export kubeconfig=./config
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.11", GitCommit:"27522a29febbcc4badac257763044d0d90c11abd", GitTreeState:"clean", BuildDate:"2021-09-15T19:21:44Z", GoVersion:"go1.15.15", Compiler:"gc", Platform:"windows/amd64"}
Server Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.11", GitCommit:"27522a29febbcc4badac257763044d0d90c11abd", GitTreeState:"clean", BuildDate:"2021-09-15T19:16:25Z", GoVersion:"go1.15.15", Compiler:"gc", Platform:"linux/amd64"}

$ kubectl get nodes
NAME                  STATUS   ROLES                      AGE    VERSION
fastdfs02             Ready    worker                     35d    v1.20.11
k8s-master01          Ready    controlplane,etcd,worker   298d   v1.20.11
k8s-master02          Ready    controlplane,etcd,worker   298d   v1.20.11
k8s-master03          Ready    controlplane,etcd,worker   298d   v1.20.11
node-jenkins-master   Ready    worker                     287d   v1.20.11
node01                Ready    worker                     298d   v1.20.11
node02                Ready    worker                     298d   v1.20.11
node03                Ready    worker                     298d   v1.20.11
node04                Ready    worker                     298d   v1.20.11
node05                Ready    worker                     225d   v1.20.11
node6                 Ready    worker                     176d   v1.20.11
sky-es01              Ready    worker                     40d    v1.20.11
sky-es02              Ready    worker                     40d    v1.20.11
sky-es03              Ready    worker                     40d    v1.20.11
```

