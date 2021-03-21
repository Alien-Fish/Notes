### 安装docker
[root@hdss7-200 ~]# curl -fsSl https://get.docker.com | bash -s docker --mirror Aliyun

### 修改配置
[root@hdss7-200 docker]# vi /etc/docker/daemon.json  
```base
{
   "graph": "/data/docker",
   "storage-driver": "overlay2",
   "insecure-registries": ["registry.access.redhat.com"],
   "registry-mirrors": ["http://hub-mirror.c.163.com"],
   "bip": "172.7.200.1/24",
   "exec-opts": ["native.cgroupdriver=systemd"],
   "live-restore": ture
}
```
- graph：docker镜像存放路劲  
- insecure-registries：私服地址  
- registry-mirrors：镜像地址，使用国内的  
- live-restore：daemon进程关闭，容器进程独立pid

### 创建docker镜像存放路劲 
[root@hdss7-21 ~]# mkdir -p /data/docker

### 启动docker服务
[root@hdss7-200 docker]# systemctl start docker.service  

### 验证
[root@hdss7-200 docker]# docker version  
[root@hdss7-200 docker]# docker info  
[root@hdss7-200 docker]# docker images  

### 常用命令
1、设置docker开机启动

systemctl enable docker

2、设置容器自动重启
1）创建容器时设置

    docker run -d --restart=always --name 设置容器名 使用的镜像
    （上面命令  --name后面两个参数根据实际情况自行修改）
     --restart具体参数值详细信息：
           no　　　　　　　　容器退出时，不重启容器；
           on-failure　　  只有在非0状态退出时才重新启动容器；
           always　　　　　 无论退出状态是如何，都重启容器；

2）修改已有容器，使用update

    docker update --restart=always 容器ID(或者容器名)
    （容器ID或者容器名根据实际情况修改）
