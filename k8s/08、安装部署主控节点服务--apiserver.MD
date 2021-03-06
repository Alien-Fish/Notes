## 部署安装说明
### 部署机器
10.4.7.21   
10.4.7.22  这里省略安装

### 下载安装
地址：https://dl.k8s.io/v1.20.4/kubernetes-server-linux-amd64.tar.gz  
试了v1.20.4启动需要配置ServiceAccount，改回1.15版本。  
地址：https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.15.md#downloads-for-v1152  
以下自行切换版本    

[root@hdss7-21 ~]# cd /opt/src/  
[root@hdss7-21 ~]# wget https://dl.k8s.io/v1.20.4/kubernetes-server-linux-amd64.tar.gz
[root@hdss7-21 src]# du -sh kubernetes-server-linux-amd64.tar.gz   
303M	kubernetes-server-linux-amd64.tar.gz  
[root@hdss7-21 src]# tar xf kubernetes-server-linux-amd64.tar.gz -C /opt  
[root@hdss7-21 opt]# mv kubernetes/ kubernetes-v1.20  
[root@hdss7-21 opt]# ln -s /opt/kubernetes-v1.20/ /opt/kubernetes  
[root@hdss7-21 kubernetes]# cd ./server/bin/  
[root@hdss7-21 bin]# ll  
```base
total 986836
-rwxr-xr-x. 1 root root  46710784 Feb 19 00:26 apiextensions-apiserver
-rwxr-xr-x. 1 root root  39251968 Feb 19 00:26 kubeadm
-rwxr-xr-x. 1 root root  44699648 Feb 19 00:26 kube-aggregator
-rwxr-xr-x. 1 root root 118218752 Feb 19 00:26 kube-apiserver
-rw-r--r--. 1 root root         8 Feb 19 00:24 kube-apiserver.docker_tag
-rw-------. 1 root root 123035136 Feb 19 00:24 kube-apiserver.tar
-rwxr-xr-x. 1 root root 112758784 Feb 19 00:26 kube-controller-manager
-rw-r--r--. 1 root root         8 Feb 19 00:24 kube-controller-manager.docker_tag
-rw-------. 1 root root 117575168 Feb 19 00:24 kube-controller-manager.tar
-rwxr-xr-x. 1 root root  40263680 Feb 19 00:26 kubectl
-rwxr-xr-x. 1 root root 114113512 Feb 19 00:26 kubelet
-rwxr-xr-x. 1 root root  39518208 Feb 19 00:26 kube-proxy
-rw-r--r--. 1 root root         8 Feb 19 00:24 kube-proxy.docker_tag
-rw-------. 1 root root 120411648 Feb 19 00:24 kube-proxy.tar
-rwxr-xr-x. 1 root root  43745280 Feb 19 00:26 kube-scheduler
-rw-r--r--. 1 root root         8 Feb 19 00:24 kube-scheduler.docker_tag
-rw-------. 1 root root  48561664 Feb 19 00:24 kube-scheduler.tar
-rwxr-xr-x. 1 root root   1634304 Feb 19 00:26 mounter
```

## 10.4.7.200上签发证书 
### 签发客户端证书
[root@hdss7-200 ssl]# cd /opt/ssl/certs/  
[root@hdss7-200 certs]# cp etcd-peer-csr.json ./etcd-client-csr.json   
[root@hdss7-200 certs]# vi etcd-client-csr.json  
```base
{
    "CN": "k8s-node",
    "hosts": [
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH",
            "O": "lean",
            "OU": "yla"
        }
    ]
}
```
[root@hdss7-200 certs]# cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json -profile=client client-csr.json | cfssljson -bare client    
[root@hdss7-200 certs]# ll  
```base
total 32
-rw-r--r--. 1 root root  980 Feb 20 21:08 client.csr
-rw-r--r--. 1 root root  272 Feb 20 21:08 client-csr.json
-rw-------. 1 root root 1675 Feb 20 21:08 client-key.pem
-rw-r--r--. 1 root root 1338 Feb 20 21:08 client.pem
-rw-r--r--. 1 root root 1013 Dec 12 11:39 etcd-peer.csr
-rw-r--r--. 1 root root  305 Dec 12 11:39 etcd-peer-csr.json
-rw-------. 1 root root 1679 Dec 12 11:39 etcd-peer-key.pem
-rw-r--r--. 1 root root 1363 Dec 12 11:39 etcd-peer.pem

```

### 签发服务端证书
[root@hdss7-200 certs]# vi apiserver-csr.json  
```base
{
    "CN": "k8s-apiserver",
    "hosts": [
      "127.0.0.1",
      "192.168.0.1",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local",
      "10.4.7.10",
      "10.4.7.21",
      "10.4.7.22",
      "10.4.7.23"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH",
            "O": "lean",
            "OU": "yla"
        }
    ]
}
```
[root@hdss7-200 certs]# cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json -profile=server apiserver-csr.json | cfssljson -bare apiserver   
[root@hdss7-200 certs]# ll  
```base
total 48
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
```

## 10.4.7.21 
### 下发证书
[root@hdss7-21 kubernetes]# cd /opt/kubernetes/server/bin   
[root@hdss7-21 bin]# mkdir certs/  
[root@hdss7-21 certs]# scp hdss7-200.host.com:/opt/ssl/certs/apiserver-key.pem .  
[root@hdss7-21 certs]# scp hdss7-200.host.com:/opt/ssl/certs/apiserver.pem .    
[root@hdss7-21 certs]# scp hdss7-200.host.com:/opt/ssl/ca.pem .  
[root@hdss7-21 certs]# scp hdss7-200.host.com:/opt/ssl/ca-key.pem .  
[root@hdss7-21 certs]# scp hdss7-200.host.com:/opt/ssl/certs/client-key.pem .  
[root@hdss7-21 certs]# scp hdss7-200.host.com:/opt/ssl/certs/client.pem .  
[root@hdss7-21 certs]# ll  
```base
total 24
-rw-------. 1 root root 1679 Feb 20 21:57 apiserver-key.pem
-rw-r--r--. 1 root root 1574 Feb 20 21:58 apiserver.pem
-rw-------. 1 root root 1679 Feb 20 21:58 ca-key.pem
-rw-r--r--. 1 root root 1318 Feb 20 21:58 ca.pem
-rw-------. 1 root root 1675 Feb 20 21:59 client-key.pem
-rw-r--r--. 1 root root 1338 Feb 20 21:59 client.pem
```
### 配置apiserver日志审计
[root@hdss7-21 bin]# mkdir /opt/kubernetes/conf
[root@hdss7-21 bin]# cd /opt/kubernetes/conf
[root@hdss7-21 conf]# vi audit.yaml  
```base
apiVersion: audit.k8s.io/v1beta1 # This is required.
kind: Policy
# Don't generate audit events for all requests in RequestReceived stage.
omitStages:
  - "RequestReceived"
rules:
  # Log pod changes at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      # Resource "pods" doesn't match requests to any subresource of pods,
      # which is consistent with the RBAC policy.
      resources: ["pods"]
  # Log "pods/log", "pods/status" at Metadata level
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]

  # Don't log requests to a configmap called "controller-leader"
  - level: None
    resources:
    - group: ""
      resources: ["configmaps"]
      resourceNames: ["controller-leader"]

  # Don't log watch requests by the "system:kube-proxy" on endpoints or services
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: "" # core API group
      resources: ["endpoints", "services"]

  # Don't log authenticated requests to certain non-resource URL paths.
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*" # Wildcard matching.
    - "/version"

  # Log the request body of configmap changes in kube-system.
  - level: Request
    resources:
    - group: "" # core API group
      resources: ["configmaps"]
    # This rule only applies to resources in the "kube-system" namespace.
    # The empty string "" can be used to select non-namespaced resources.
    namespaces: ["kube-system"]

  # Log configmap and secret changes in all other namespaces at the Metadata level.
  - level: Metadata
    resources:
    - group: "" # core API group
      resources: ["secrets", "configmaps"]

  # Log all other resources in core and extensions at the Request level.
  - level: Request
    resources:
    - group: "" # core API group
    - group: "extensions" # Version of group should NOT be included.

  # A catch-all rule to log all other requests at the Metadata level.
  - level: Metadata
    # Long-running requests like watches that fall under this rule will not
    # generate an audit event in RequestReceived.
    omitStages:
      - "RequestReceived"
```

### 制作启动脚本
#### 配置apiserver启动配置
[root@hdss7-21 bin]# vi kube-apiserver-startup.sh  
```base
#!/bin/bash
./kube-apiserver \
    --apiserver-count 2 \
    --audit-log-path /data/logs/kubernetes/kube-apiserver/audit-log \
    --audit-policy-file ../../conf/audit.yaml \
    --authorization-mode RBAC \
    --client-ca-file ./certs/ca.pem \
    --requestheader-client-ca-file ./certs/ca.pem \
    --enable-admission-plugins NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota \
    --etcd-cafile ./certs/ca.pem \
    --etcd-certfile ./certs/client.pem \
    --etcd-keyfile ./certs/client-key.pem \
    --etcd-servers https://10.4.7.12:2379,https://10.4.7.21:2379,https://10.4.7.22:2379 \
    --service-account-key-file ./certs/ca-key.pem \
    --service-cluster-ip-range 192.168.0.0/16 \
    --service-node-port-range 3000-29999 \
    --target-ram-mb=1024 \
    --kubelet-client-certificate ./certs/client.pem \
    --kubelet-client-key ./certs/client-key.pem \
    --log-dir  /data/logs/kubernetes/kube-apiserver \
    --tls-cert-file ./certs/apiserver.pem \
    --tls-private-key-file ./certs/apiserver-key.pem \
    --v 2
```
#### 配置supervisor启动配置
[root@hdss7-21 bin]# vi /etc/supervisord.d/kube-apiserver.ini 
```base
#!/bin/bash

[program:kube-apiserver-7-21]
command=/opt/kubernetes/server/bin/kube-apiserver-startup.sh
numprocs=1
directory=/opt/kubernetes/server/bin/
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-apiserver/apiserver.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=5
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
```
[root@hdss7-21 bin]# mkdir -p /data/logs/kubernetes/kube-apiserver  
[root@hdss7-21 bin]# chmod +x /opt/kubernetes/server/bin/kube-apiserver-startup.sh  

[root@hdss7-21 bin]# supervisorctl update  
[root@hdss7-21 bin]# supervisorctl status 
[root@hdss7-21 bin]# tail -200f /data/logs/kubernetes/kube-apiserver/apiserver.stdout.log  

#### 启停apiserver
[root@hdss7-12 ~]# supervisorctl start kube-apiserver-7-21  
[root@hdss7-12 ~]# supervisorctl stop kube-apiserver-7-21  
[root@hdss7-12 ~]# supervisorctl restart kube-apiserver-7-21  
[root@hdss7-12 ~]# supervisorctl status kube-apiserver-7-21  

#### 查看进程
[root@hdss7-21 bin]# netstat -lntp|grep api  
[root@hdss7-21 bin]# ps uax|grep kube-apiserver|grep -v grep  
