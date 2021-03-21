## Kubelet组件
Kubelet组件运行在Node节点上，维持运行中的Pods以及提供kuberntes运行时环境，主要完成以下使命：   
- 监视分配给该Node节点的pods   
- 挂载pod所需要的volumes   
- 下载pod的secret   
- 通过docker/rkt来运行pod中的容器   
- 周期的执行pod中为容器定义的liveness探针   
- 上报pod的状态给系统的其他组件   
- 上报Node的状态  

## pause容器 
kubelet安装在node节点，我们的node跟apiserver在一起，这里安装在hdss7-21,22上：  
首先，kubernetes依赖pause，我们先将此服务上传到们自己的docker私有仓库：  
kubernetes中的pause容器主要为每个业务容器提供以下功能：  
- 在pod中担任Linux命名空间共享的基础；  
- 启用pid命名空间，开启init进程。  

### pause安装
[root@hdss7-200 ~]# docker pull kubernetes/pause  
[root@hdss7-200 harbor]# docker tag f9d5de079539 harbor.lean.com/public/pause:latest  
[root@hdss7-200 harbor]# docker push harbor.lean.com/public/pause:latest  
### 签发证书
[root@hdss7-200 ssl]# vi /opt/ssl/certs/kubelet-csr.json  
```base
{
    "CN": "k8s-kubelet",
    "hosts": [
    "127.0.0.1",
    "10.4.7.10",
    "10.4.7.21",
    "10.4.7.22",
    "10.4.7.23",
    "10.4.7.24",
    "10.4.7.25",
    "10.4.7.26",
    "10.4.7.27",
    "10.4.7.28"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "od",
            "OU": "ops"
        }
    ]
}
```
[root@hdss7-200 certs]# cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json -profile=server kubelet-csr.json | cfssljson -bare kubelet  

### 下发证书到21、22
[root@hdss7-21 ~]# cd /opt/kubernetes/server/bin/certs/  
[root@hdss7-21 certs]# scp hdss7-200.host.com:/opt/ssl/certs/kubelet.pem ./  
[root@hdss7-21 certs]# scp hdss7-200.host.com:/opt/ssl/certs/kubelet-key.pem ./  

### 创建配置
[root@hdss7-21 certs]# cd /opt/kubernetes/conf  
[root@hdss7-21 conf]# kubectl config set-cluster myk8s \  
  --certificate-authority=/opt/kubernetes/server/bin/certs/ca.pem \  
  --embed-certs=true \  
  --server=https://10.4.7.10:7443 \  
  --kubeconfig=kubelet.kubeconfig  

[root@hdss7-21 conf]# kubectl config set-credentials k8s-node \  
  --client-certificate=/opt/kubernetes/server/bin/certs/client.pem \  
  --client-key=/opt/kubernetes/server/bin/certs/client-key.pem \  
  --embed-certs=true \  
  --kubeconfig=kubelet.kubeconfig   

[root@hdss7-21 conf]# kubectl config set-context myk8s-context \  
  --cluster=myk8s \  
  --user=k8s-node \  
  --kubeconfig=kubelet.kubeconfig  

[root@hdss7-21 conf]# kubectl config use-context myk8s-context --kubeconfig=kubelet.kubeconfig  

[root@hdss7-21 conf]# vi k8s-node.yaml  
```base
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: k8s-node
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:node
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: k8s-node
```
[root@hdss7-21 conf]# kubectl create -f k8s-node.yaml  

### 检查k8s-node资源创建状态
[root@hdss7-21 conf]# kubectl get clusterrolebinding k8s-node -o yaml  
```base
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: "2021-03-14T14:20:45Z"
  name: k8s-node
  resourceVersion: "9224"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/k8s-node
  uid: 8ead5682-908e-410b-bf05-646ccea6cfe8
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:node
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: k8s-node
```

## kubelet启动脚本
[root@hdss7-21 bin]# vi /opt/kubernetes/server/bin/kubelet.sh  
```base
#!/bin/sh
./kubelet \
  --anonymous-auth=false \
  --cgroup-driver systemd \
  --cluster-dns 192.168.0.2 \
  --cluster-domain cluster.local \
  --runtime-cgroups=/systemd/system.slice \
  --kubelet-cgroups=/systemd/system.slice \
  --fail-swap-on="false" \
  --client-ca-file ./certs/ca.pem \
  --tls-cert-file ./certs/kubelet.pem \
  --tls-private-key-file ./certs/kubelet-key.pem \
  --hostname-override hdss7-21.host.com \
  --image-gc-high-threshold 20 \
  --image-gc-low-threshold 10 \
  --kubeconfig ../../conf/kubelet.kubeconfig \
  --log-dir /data/logs/kubernetes/kube-kubelet \
  --pod-infra-container-image harbor.lean.com/public/pause:latest \
  --root-dir /data/kubelet
```
[root@hdss7-21 bin]# chmod u+x /opt/kubernetes/server/bin/kubelet.sh  
[root@hdss7-21 bin]# mkdir -p /data/logs/kubernetes/kube-kubelet  


## supervisord启动
[root@hdss7-21 bin]# vi /etc/supervisord.d/kube-kubelet.ini  
```base
[program:kube-kubelet-7-21]
command=/opt/kubernetes/server/bin/kubelet.sh     ; the program (relative uses PATH, can take args)
numprocs=1                                        ; number of processes copies to start (def 1)
directory=/opt/kubernetes/server/bin              ; directory to cwd to before exec (def no cwd)
autostart=true                                    ; start at supervisord start (default: true)
autorestart=true                                ; retstart at unexpected quit (default: true)
startsecs=30                                      ; number of secs prog must stay running (def. 1)
startretries=3                                    ; max # of serial start failures (default 3)
exitcodes=0,2                                     ; 'expected' exit codes for process (default 0,2)
stopsignal=QUIT                                   ; signal used to kill process (default TERM)
stopwaitsecs=10                                   ; max num secs to wait b4 SIGKILL (default 10)
user=root                                         ; setuid to this UNIX account to run the program
redirect_stderr=true                              ; redirect proc stderr to stdout (default false)
stdout_logfile=/data/logs/kubernetes/kube-kubelet/kubelet.stdout.log   ; stderr log path, NONE for none; default AUTO
stdout_logfile_maxbytes=64MB                      ; max # logfile bytes b4 rotation (default 50MB)
stdout_logfile_backups=4                          ; # of stdout logfile backups (default 10)
stdout_capture_maxbytes=1MB                       ; number of bytes in 'capturemode' (default 0)
stdout_events_enabled=false                       ; emit events on stdout writes (default false)
```
[root@hdss7-21 bin] supervisorctl update  
[root@hdss7-21 bin] supervisorctl status  

注意：docker必须启动

## 查看集群状态
[root@hdss7-21 ~]# kubectl get nodes   
NAME                STATUS   ROLES    AGE     VERSION   
hdss7-21.host.com   Ready    <none>   4m50s   v1.15.2   
hdss7-22.host.com   Ready    <none>   89s     v1.15.2  

## 启动问题
### 问题一
error: failed to run Kubelet: failed to create kubelet: misconfiguration: kubelet cgroup driver: "cgroupfs" is different from docker cgroup driver: "systemd"​  
(1).docker修改方法：  
cat > /etc/docker/daemon.json <<EOF  
{    
  "exec-opts": ["native.cgroupdriver=systemd"]    
}    
EOF  
重启docker：  
systemctl restart docker  

(2).k8s修改方法：    
在 Master 节点上配置 kubelet 所需的 cgroup 驱动  
使用 Docker 时，kubeadm 会自动为其检测 cgroup 驱动在运行时对 /var/lib/kubelet/kubeadm-flags.env 文件进行配置。
如果您使用了不同的 CRI， 您得把 /etc/default/kubelet 文件中的 cgroup-driver 位置改为对应的值，像这样：   
KUBELET_EXTRA_ARGS=--cgroup-driver=<value>   
这个文件将会被 kubeadm init 和 kubeadm join 用于为 kubelet 获取 额外的用户参数。   

需要重启 kubelet：  
systemctl daemon-reload  
systemctl restart kubelet  
