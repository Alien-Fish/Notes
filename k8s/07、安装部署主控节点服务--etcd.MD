### 部署机器
分别在三台虚拟机上安装etcd：  
10.4.7.12   
10.4.7.21   
10.4.7.22  

## 10.4.7.200上签发证书 
### 制作ca证书，不懂回顾04节
在10.4.7.200上签发证书   
[root@hdss7-200 ~]# cd /opt/ssl/  
[root@hdss7-200 ssl]# ls  
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem  
[root@hdss7-200 ssl]# cat ca-config.json 
```base
{
    "signing": {
        "default": {
            "expiry": "43800h"
        },
        "profiles": {
            "server": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "peer": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
```

创建存放证书的目录certs  
[root@hdss7-200 ssl]# mkdir certs  
[root@hdss7-200 ssl]# cd certs/
[root@hdss7-200 certs]# vi etcd-peer-csr.json  
```base
{
    "CN": "k8s-etcd",
    "hosts": [
        "10.4.7.11",
        "10.4.7.12",
        "10.4.7.21",
        "10.4.7.22"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH"
        }
    ]
}
```
签发peer通信证书  
[root@hdss7-200 certs]# cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json  -profile=peer etcd-peer-csr.json | cfssljson -bare etcd-peer  
```base
2020/12/12 11:39:02 [INFO] generate received request
2020/12/12 11:39:02 [INFO] received CSR
2020/12/12 11:39:02 [INFO] generating key: rsa-2048
2020/12/12 11:39:02 [INFO] encoded CSR
2020/12/12 11:39:02 [INFO] signed certificate with serial number 463806500198977965983434355005550851087911549333
2020/12/12 11:39:02 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

## 10.4.7.12安装etcd
### 创建etcd用户  
[root@hdss7-12 ~]# useradd -s /sbin/nologin -M etcd  
### 下载etcd安装包
下载地址：https://github.com/etcd-io/etcd/tags  
[root@hdss7-12 opt]# cd /usr/src/  
[root@hdss7-12 src]# wget https://github.com/etcd-io/etcd/releases/download/v3.4.14/etcd-v3.4.14-linux-amd64.tar.gz  

### 解压安装etcd
[root@hdss7-12 src]# tar xfv etcd-v3.4.14-linux-amd64.tar.gz -C /opt/  
[root@hdss7-12 src]# cd /opt/  
[root@hdss7-12 opt]# mv etcd-v3.4.14-linux-amd64/ etcd-v3.4.14  
[root@hdss7-12 opt]# ln -s /opt/etcd-v3.4.14/ /opt/etcd  
[root@hdss7-12 opt]# cd etcd  


### 传输证书
[root@hdss7-12 ~]# cd /opt/  
[root@hdss7-12 opt]# mkdir -p /opt/etcd/certs /data/etcd /data/logs/etcd-server  
[root@hdss7-12 opt]# cd certs  
[root@hdss7-12 certs]# scp hdss7-200.host.com:/opt/ssl/certs/ca.pem .  
[root@hdss7-12 certs]# scp hdss7-200.host.com:/opt/ssl/certs/etcd-peer.pem .    
[root@hdss7-12 certs]# scp hdss7-200.host.com:/opt/ssl/certs/etcd-peer-key.pem .  
[root@hdss7-12 certs]# ll  
total 12  
-rw-r--r--. 1 root root 1318 Dec 12 12:05 ca.pem  
-rw-------. 1 root root 1679 Dec 12 12:09 etcd-peer-key.pem  
-rw-r--r--. 1 root root 1363 Dec 12 12:08 etcd-peer.pem  

### 设置scp传输
[root@hdss7-200 certs]# echo 1|passwd root --stdin  
Changing password for user root.  
passwd: all authentication tokens updated successfully.  
[root@hdss7-12 certs]# scp hdss7-200.host.com:/opt/ssl/certs/ca.pem .  
[root@hdss7-12 certs]# scp hdss7-200.host.com:/opt/ssl/certs/etcd-peer.pem .  
[root@hdss7-12 certs]# scp hdss7-200.host.com:/opt/ssl/certs/etcd-peer-key.pem .  

### 制作etcd启动脚本
[root@hdss7-12 etcd]# vi etcd-server-startup.sh  
```base
#!/bin/sh
./etcd --name etcd-server-7-12 \
       --data-dir /data/etcd/etcd-server \
       --listen-peer-urls https://10.4.7.12:2380 \
       --listen-client-urls https://10.4.7.12:2379,http://127.0.0.1:2379 \
       --quota-backend-bytes 8000000000 \
       --initial-advertise-peer-urls https://10.4.7.12:2380 \
       --advertise-client-urls https://10.4.7.12:2379,http://127.0.0.1:2379 \
       --initial-cluster  etcd-server-7-12=https://10.4.7.12:2380,etcd-server-7-21=https://10.4.7.21:2380,etcd-server-7-22=https://10.4.7.22:2380 \
       --cert-file ./certs/etcd-peer.pem \
       --key-file ./certs/etcd-peer-key.pem \
       --client-cert-auth \
       --trusted-ca-file ./certs/ca.pem \
       --peer-cert-file ./certs/etcd-peer.pem \
       --peer-key-file ./certs/etcd-peer-key.pem \
       --peer-client-cert-auth \
       --peer-trusted-ca-file ./certs/ca.pem \
       --log-outputs stdout
```
[root@hdss7-12 etcd]# chmod +x etcd-server-startup.sh  
#### 更新etcd目录属主
[root@hdss7-12 etcd]# chown -R etcd.etcd /opt/etcd-v3.4.14/  
[root@hdss7-12 etcd]# chown -R etcd.etcd /data/etcd/  
[root@hdss7-12 etcd]# chown -R etcd.etcd /data/logs/etcd-server/  

#### 安装管理后端进程软件
[root@hdss7-12 etcd]# yum install supervisor -y  
[root@hdss7-12 etcd]# systemctl start supervisord  
[root@hdss7-12 etcd]# systemctl enable supervisord  


[root@hdss7-12 etcd]# vi /etc/supervisord.d/etcd-server.ini  
```base
[program:etcd-server-7-12]
command=/opt/etcd/etcd-server-startup.sh
numprocs=1
directory=/opt/etcd
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=etcd
redirect_stderr=true
stdout_logfile=/data/logs/etcd-server/etcd.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
```
[root@hdss7-12 etcd]# supervisorctl update
etcd-server-7-12: added process group  
[root@hdss7-12 etcd]# supervisorctl start etcd-server-7-12  
[root@hdss7-12 etcd]# supervisorctl status  
etcd-server-7-12                 RUNNING   pid 1829, uptime 0:01:45  
[root@hdss7-12 etcd]# tail -fn 200 /data/logs/etcd-server/etcd.stdout.log   
[root@hdss7-12 etcd]# netstat -luntp|grep etcd  
tcp        0      0 10.4.7.12:2379          0.0.0.0:*               LISTEN      1830/./etcd           
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      1830/./etcd            
tcp        0      0 10.4.7.12:2380          0.0.0.0:*               LISTEN      1830/./etcd  
