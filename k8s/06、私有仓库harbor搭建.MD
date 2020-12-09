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
[root@hdss7-200 harbor]# yum install docker-compose -y  