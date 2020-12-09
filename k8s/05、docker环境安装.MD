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
