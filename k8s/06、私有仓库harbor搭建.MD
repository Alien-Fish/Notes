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