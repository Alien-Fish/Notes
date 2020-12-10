## 安装
### harbor的github官网地址
https://github.com/goharbor/harbor/tags

### 下载离线安装包
[root@hdss7-200 ~]# cd /opt/  
[root@hdss7-200 opt]# mkdir src  
[root@hdss7-200 src]# wget https://github.com/goharbor/harbor/releases/download/v1.10.6/harbor-offline-installer-v1.10.6.tgz  

### 解压安装
[root@hdss7-200 src]# tar xf harbor-offline-installer-v1.10.6.tgz -C /opt/  
[root@hdss7-200 src]# cd /opt/  
[root@hdss7-200 opt]# mv harbor/ harbor-v1.10.6  
[root@hdss7-200 opt]# ln -s /opt/harbor-v1.10.6/ /opt/harbor  
[root@hdss7-200 harbor]# vi harbor.yml  
```base
hostname: harbor.lean.com
# http related config
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 180

# https related config
#https:
#  # https port for harbor, default is 443
#  port: 443
#  # The path of cert and key files for nginx
#  certificate: /your/certificate/path
#  private_key: /your/private/key/path

# The default data volume
data_volume: /datai/harbor 

# Log configurations
log:
  # options are debug, info, warning, error, fatal
  level: info
  # configs for logs in local storage
  local:
    # Log files are rotated log_rotate_count times before being removed. If count is 0, old versions are removed rather than rotated.
    rotate_count: 50
    # Log files are rotated only if they grow bigger than log_rotate_size bytes. If size is followed by k, the size is assumed to be in kilobytes.
    # If the M is used, the size is in megabytes, and if G is used, the size is in gigabytes. So size 100, size 100k, size 100M and size 100G
    # are all valid.
    rotate_size: 200M
    # The directory on your host that store log
    location: /data/harbor/logs
```  

[root@hdss7-200 harbor]# mkdir -p /data/harbor/logs  

### 安装docker-compose
[root@hdss7-200 harbor]# yum install docker-compose -y  
[root@hdss7-200 harbor]# rpm -qa docker-compose  
docker-compose-1.18.0-4.el7.noarch  

### 安装harbor
[root@hdss7-200 harbor]# ./install.sh  
....
[Step 5]: starting Harbor ...  
Creating network "harbor-v1106_harbor" with the default driver  
Creating harbor-log ... done  
Creating registry      ... done  
Creating redis         ... done  
Creating harbor-db     ... done  
Creating registryctl   ... done  
Creating harbor-portal ... done  
Creating harbor-core   ... done  
Creating harbor-jobservice ... done  
Creating nginx             ... done  
✔ ----Harbor has been installed and started successfully.----  

[root@hdss7-200 harbor]# docker-compose ps -a  
[root@hdss7-200 harbor]# yum install nginx -y  
[root@hdss7-200 harbor]# vi /etc/nginx/conf.d/harbor.lean.com.conf  
```base
server {
    listen             80;
    server_name        harbor.lean.com;

    client_max_body_size         1000m;

    location / {
         proxy_pass http://127.0.0.1:180;
    }
}
```
[root@hdss7-200 conf.d]# systemctl start nginx   
[root@hdss7-200 conf.d]# systemctl enable nginx  

### 配置DNS
[root@hdss7-200 conf.d]# curl harbor.lean.com   
curl: (6) Could not resolve host: harbor.lean.com; Unknown error    

[root@hdss7-11 ~]# vi /var/named/lean.com.zone  
```base
$ORIGIN lean.com.
$TTL 600        ;  10 minutes
@               IN SOA  dns.lean.com. dnsadmin.lean.com. (
                                2020112802 ; serial
                                10800      ; refresh (3 hours)
                                900        ; retry (1 week)
                                604800     ; expire (1 week)
                                86400      ; minimun (1 day)
                                )
                         NS dns.lean.com.
$TTL 60 ; 1minute
dns               A      10.4.7.11
harbor            A      10.4.7.200
```
备注：2020112802 ; serial，  加一个版本  2020112801->2020112802   
harbor            A      10.4.7.200，加一条dns解析记录  

[root@hdss7-11 ~]# systemctl restart named  
[root@hdss7-11 ~]# dig -t A harbor.lean.com +short  
10.4.7.200  

[root@hdss7-200 conf.d]# firewall-cmd --zone=public --permanent --add-service=http  
success  
[root@hdss7-200 conf.d]# firewall-cmd --zone=public --add-port=80/tcp  
success  

[root@hdss7-200 conf.d]# vi /etc/nginx/nginx.conf  
将第一行修改为 user root   
[root@hdss7-200 conf.d]# setsebool -P httpd_can_network_connect 1  

[root@hdss7-200 conf.d]# curl harbor.lean.com  
```base
<!doctype html>
<html>

<head>
    <meta charset="utf-8">
    <title>Harbor</title>
    <base href="/">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link rel="icon" type="image/x-icon" href="favicon.ico?v=2">
<link rel="stylesheet" href="styles.701dc5ee3007bd83bfa4.css"></head>
...
```

成功，在浏览器访问harbor.lean.com  
账号：admin  
密码：Harbor12345  
登录harbor，创建public项目  

### 启动/关闭
[root@hdss7-200 harbor]# pwd  
/opt/harbor  
[root@hdss7-200 harbor]# docker-compose up -d  
harbor-log is up-to-date  
Starting registry ...   
registryctl is up-to-date  
Starting registry      ... done  
Starting harbor-portal ... done  
Starting redis         ... done  
Starting harbor-core   ... done  
Starting harbor-jobservice ... done  
Starting nginx             ... done  
[root@hdss7-200 harbor]# docker-compose stop    

### 配置docker私服
[root@hdss7-200 harbor]# vi /etc/docker/daemon.json   
{   
   "graph": "/data/docker",  
   "storage-driver": "overlay2",  
   "insecure-registries": ["registry.access.redhat.com", "harbor.lean.com"],  
   "registry-mirrors": ["http://hub-mirror.c.163.com"],  
   "bip": "172.7.200.1/24"  
}  
更新"insecure-registries": ["registry.access.redhat.com", "harbor.lean.com"], 加入 harbor.lean.com

## 制作镜像上传到harbor
### 登录harbor
[root@hdss7-200 harbor]# docker login harbor.lean.com  
Username: admin  
Password: Harbor12345  

### 制作镜像
[root@hdss7-200 harbor]# docker pull nginx:1.7.9  
[root@hdss7-200 harbor]# docker images|grep 1.7.9  
nginx                           1.7.9               84581e99d807        5 years ago         91.7MB  

### 上传镜像
[root@hdss7-200 harbor]# docker tag 84581e99d807 harbor.lean.com/public/nginx:v1.7.9    
[root@hdss7-200 harbor]# docker push harbor.lean.com/public/nginx:v1.7.9  
The push refers to repository [harbor.lean.com/public/nginx]  
5f70bf18a086: Pushed   
4b26ab29a475: Pushed   
ccb1d68e3fb7: Pushed    
e387107e2065: Pushed   
63bf84221cce: Pushed   
e02dce553481: Pushed   
dea2e4984e29: Pushed   
v1.7.9: digest: sha256:b1f5935eb2e9e2ae89c0b3e2e148c19068d91ca502e857052f14db230443e4c2 size: 3012  
