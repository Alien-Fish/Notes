### 部署节点
10.4.7.21   
10.4.7.22  

### 安装说明
运行在每个节点上，监听 API Server 中服务对象的变化，再通过管理 IPtables 来实现网络的转发
Kube-Proxy 目前支持三种模式：
- UserSpace
k8s v1.2 后就已经淘汰
- IPtables
目前默认方式
- IPVS--推荐，支持7层
需要安装ipvsadm、ipset 工具包和加载 ip_vs 内核模块

### 安装ipset，ipvsadm
[root@hdss7-21 ~]# yum install ipset  -y  
[root@hdss7-21 ~]# yum -y install ipvsadm  

### 签发证书
[root@hdss7-200 certs]# vi /opt/ssl/certs/kube-proxy-csr.json  
```base
{
    "CN": "system:kube-proxy",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "SH",
            "L": "SH",
            "O": "lean",
            "OU": "yla"
        }
    ]
}
```
[root@hdss7-200 certs]# cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json -profile=client kube-proxy-csr.json |cfssljson -bare kube-proxy-client  
[root@hdss7-200 certs]# ll  
total 80  
-rw-r--r--. 1 root root 1240 Feb 20 21:32 apiserver.csr  
-rw-r--r--. 1 root root  538 Feb 20 21:30 apiserver-csr.json  
-rw-------. 1 root root 1679 Feb 20 21:32 apiserver-key.pem  
-rw-r--r--. 1 root root 1574 Feb 20 21:32 apiserver.pem  
-rw-r--r--. 1 root root  980 Feb 20 21:08 client.csr  
-rw-r--r--. 1 root root  272 Feb 20 21:08 client-csr.json  
-rw-------. 1 root root 1675 Feb 20 21:08 client-key.pem  
-rw-r--r--. 1 root root 1338 Feb 20 21:08 client.pem  
-rw-r--r--. 1 root root 1013 Dec 12 11:39 etcd-peer.csr  
-rw-r--r--. 1 root root  305 Dec 12 11:39 etcd-peer-csr.json  
-rw-------. 1 root root 1679 Dec 12 11:39 etcd-peer-key.pem  
-rw-r--r--. 1 root root 1363 Dec 12 11:39 etcd-peer.pem  
-rw-r--r--. 1 root root 1115 Mar 14 22:07 kubelet.csr  
-rw-r--r--. 1 root root  452 Mar 14 22:04 kubelet-csr.json  
-rw-------. 1 root root 1675 Mar 14 22:07 kubelet-key.pem  
-rw-r--r--. 1 root root 1452 Mar 14 22:07 kubelet.pem  
-rw-r--r--. 1 root root  993 Mar 21 12:57 kube-proxy-client.csr  
-rw-------. 1 root root 1679 Mar 21 12:57 kube-proxy-client-key.pem  
-rw-r--r--. 1 root root 1350 Mar 21 12:57 kube-proxy-client.pem  
-rw-r--r--. 1 root root  259 Mar 21 12:54 kube-proxy-csr.json  

### 拷贝证书到21，22
[root@hdss7-21 certs]# scp hdss7-200.host.com:/opt/ssl/certs/kube-proxy-client-key.pem .  
[root@hdss7-21 certs]# scp hdss7-200.host.com:/opt/ssl/certs/kube-proxy-client.pem .
[root@hdss7-21 conf]# cd /opt/kubernetes/conf  
[root@hdss7-21 conf]# kubectl config set-cluster myk8s \
 --certificate-authority=/opt/kubernetes/server/bin/certs/ca.pem \
 --embed-certs=true \
 --server=https://10.4.7.10:7443 \
 --kubeconfig=kube-proxy.kubeconfig

### 配置
[root@hdss7-21 conf]# kubectl config set-credentials kube-proxy \
  --client-certificate=/opt/kubernetes/server/bin/certs/kube-proxy-client.pem \
  --client-key=/opt/kubernetes/server/bin/certs/kube-proxy-client-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

[root@hdss7-21 conf]# kubectl config set-context myk8s-context \
  --cluster=myk8s \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

[root@hdss7-21 conf]# kubectl config use-context myk8s-context --kubeconfig=kube-proxy.kubeconfig

### ipvs内核的脚本
[root@hdss7-21 conf]# vi /root/ipvs.sh
```base
#!/bin/bash
ipvs_mods_dir="/usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs"
for i in $(ls $ipvs_mods_dir|grep -o "^[^.]*")
do
  /sbin/modinfo -F filename $i &>/dev/null
  if [ $? -eq 0 ];then
    /sbin/modprobe $i
  fi
done
```
[root@hdss7-21 conf]# chmod u+x /root/ipvs.sh  
[root@hdss7-21 conf]# sh /root/ipvs.sh  

### kube-proxy启动脚本
[root@hdss7-21 conf]# vi /opt/kubernetes/server/bin/kube-proxy.sh
```base
#!/bin/sh
./kube-proxy \
  --cluster-cidr 172.7.0.0/16 \
  --hostname-override hdss7-21.host.com \
  --proxy-mode=ipvs \
  --ipvs-scheduler=nq \
  --kubeconfig /opt/kubernetes/conf/kube-proxy.kubeconfig
```
[root@hdss7-21 conf]# chmod +x /opt/kubernetes/server/bin/kube-proxy.sh  
[root@hdss7-21 conf]# mkdir -p /data/logs/kubernetes/kube-proxy  
[root@hdss7-21 conf]# vi /etc/supervisord.d/kube-proxy.ini
```base
[program:kube-proxy-7-21]
command=/opt/kubernetes/server/bin/kube-proxy.sh                     ; the program (relative uses PATH, can take args)
numprocs=1                                                           ; number of processes copies to start (def 1)
directory=/opt/kubernetes/server/bin                                 ; directory to cwd to before exec (def no cwd)
autostart=true                                                       ; start at supervisord start (default: true)
autorestart=true                                                     ; retstart at unexpected quit (default: true)
startsecs=30                                                         ; number of secs prog must stay running (def. 1)
startretries=3                                                       ; max # of serial start failures (default 3)
exitcodes=0,2                                                        ; 'expected' exit codes for process (default 0,2)
stopsignal=QUIT                                                      ; signal used to kill process (default TERM)
stopwaitsecs=10                                                      ; max num secs to wait b4 SIGKILL (default 10)
user=root                                                            ; setuid to this UNIX account to run the program
redirect_stderr=true                                                 ; redirect proc stderr to stdout (default false)
stdout_logfile=/data/logs/kubernetes/kube-proxy/proxy.stdout.log     ; stderr log path, NONE for none; default AUTO
stdout_logfile_maxbytes=64MB                                         ; max # logfile bytes b4 rotation (default 50MB)
stdout_logfile_backups=4                                             ; # of stdout logfile backups (default 10)
stdout_capture_maxbytes=1MB                                          ; number of bytes in 'capturemode' (default 0)
stdout_events_enabled=false                                          ; emit events on stdout writes (default false)
```
[root@hdss7-21 conf]# supervisorctl update  
kube-proxy-7-21: added process group  
[root@hdss7-21 conf]# supervisorctl status  

### 创建nginx-ds pod (21上执行)
[root@hdss7-21 conf]# vi /root/nginx-ds.yaml  
```base
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx-ds
spec:
  template:
    metadata:
      labels:
        app: nginx-ds
    spec:
      containers:
      - name: my-nginx
        image: harbor.lean.com/public/nginx:v1.7.9
        ports:
        - containerPort: 80
```
[root@hdss7-21 conf]# cd /root/  
[root@hdss7-21 ~]# kubectl create -f nginx-ds.yaml  
daemonset.extensions/nginx-ds created  
[root@hdss7-21 ~]# kubectl get pods   
NAME             READY   STATUS              RESTARTS   AGE  
nginx-ds-7c9v9   0/1     ContainerCreating   0          23s  
nginx-ds-fwmr8   0/1     ContainerCreating   0          23s  

### 验证集群状态
#### etcd controller-manager scheduler状态 
[root@hdss7-21 ~]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}     
etcd-1               Healthy   {"health":"true"}     
etcd-2               Healthy   {"health":"true"}  

#### nodes状态
[root@hdss7-21 ~]# kubectl get nodes  
NAME                STATUS   ROLES    AGE    VERSION  
hdss7-21.host.com   Ready    <none>   5d2h   v1.15.2  
hdss7-22.host.com   Ready    <none>   5d2h   v1.15.2  

#### pods状态
[root@hdss7-21 ~]# kubectl get pods   
NAME             READY   STATUS              RESTARTS   AGE   
nginx-ds-7c9v9   0/1     ContainerCreating   0          2m50s   
nginx-ds-fwmr8   0/1     ContainerCreating   0          2m50s   

