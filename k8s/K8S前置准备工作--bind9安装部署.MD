## 1.基础依赖
### 关闭防火墙
systemctl stop firewalld

### 安装基础依赖工具
yum install wget net-tools telnet tree nmap sysstat lrzsz dosunix bind-utils -y

## 2.bind安装和配置
### 10.4.7.11上安装DNS解析工具bind
yum install bind -y

### bind配置
vi /etc/named.conf
```base
options {
        listen-on port 53 { 10.4.7.11; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { any; };
        forwarders      {10.4.7.254; };

        /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable
           recursion.
         - If your recursive DNS server has a public IP address, you MUST enable access
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface
        */
        recursion yes;

        dnssec-enable no;
        dnssec-validation no;

        /* Path to ISC DLV key */
        bindkeys-file "/etc/named.root.key";

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};
```
更新内容:  
##监听地址  
listen-on port 53 { 10.4.7.11; };  
##白名单  
allow-query     { any; };  
##网关  
forwarders      {10.4.7.254; };  
##关闭DNS  
dnssec-enable no;  
dnssec-validation no;  

### named-checkconf
执行检查命令named-checkconf，无报错则正常

### bind区域配置
vi /etc/named.rfc1912.zones
```base
zone "host.com" IN {
        type master;
        file "host.com.zone";
        allow-update { 10.4.7.11; };
};

zone "lean.com" IN {
        type master;
        file "lean.com.zone";
        allow-update { 10.4.7.11; };
};
```
新增主机域host.com和业务域lean.com

### bind区域数据文件配置
#### 主机域配置
vi /var/named/host.com.zone
```base
$ORIGIN host.com.
$TTL 600        ;  10 minutes
@       IN SOA  dns.host.com. dnsadmin.host.com. (
                                2020112801 ; serial
                                10800      ; refresh (3 hours)
                                900        ; retry (1 week)
                                604800     ; expire (1 week) 
                                86400      ; minimun (1 day)
                                )
                         NS dns.host.com.
$TTL 60 ; 1minute
dns               A      10.4.7.11
HDSS7-11          A      10.4.7.11
HDSS7-12          A      10.4.7.12
HDSS7-21          A      10.4.7.21
HDSS7-22          A      10.4.7.22
HDSS7-200         A      10.4.7.200
```
#### 业务域配置
vi /var/named/lean.com.zone
```base
$ORIGIN lean.com.
$TTL 600        ;  10 minutes
@               IN SOA  dns.lean.com. dnsadmin.lean.com. (
                                2020112801 ; serial
                                10800      ; refresh (3 hours)
                                900        ; retry (1 week)
                                604800     ; expire (1 week)
                                86400      ; minimun (1 day)
                                )
                         NS dns.lean.com.
$TTL 60 ; 1minute
dns               A      10.4.7.11
```

### named-checkconf
执行检查命令named-checkconf，无报错则正常

### 启动named
[root@hdss7-11 ~]# systemctl start named  
[root@hdss7-11 ~]# netstat -luntp|grep 53  
tcp        0      0 127.0.0.1:953           0.0.0.0:*               LISTEN      1829/named          
tcp        0      0 10.4.7.11:53            0.0.0.0:*               LISTEN      1829/named          
tcp6       0      0 ::1:953                 :::*                    LISTEN      1829/named          
udp        0      0 10.4.7.11:53            0.0.0.0:*                           1829/named 